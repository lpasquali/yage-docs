# ADR 0004 — Universal OpenTofu Identity Bootstrap (Phase G)

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Backend (HCL templates + EnsureIdentity wiring), Architect (interface contract)

---

## Context

Proxmox is the only provider where yage currently mints credentials at bootstrap time
(`internal/platform/opentofux/` uses the BPG Terraform provider to create PVE users,
tokens, and ACLs). Every other provider returns `ErrNotApplicable` from `EnsureIdentity`
and expects the operator to supply credentials out-of-band.

Abstraction-plan §21.2 documents that seven providers have Terraform/OpenTofu providers
that yage could use to mint credentials programmatically:

| Provider | Credential minted | TF provider |
|---|---|---|
| AWS | IAM user + access key (or role for IRSA) | `hashicorp/aws` |
| Azure | Service Principal + client secret | `hashicorp/azurerm` |
| GCP | Service Account + JSON key | `hashicorp/google` |
| OpenStack | Application Credential (Keystone) | `terraform-provider-openstack` |
| OCI | API key pair | `oracle/oci` |
| IBMCloud | Service ID + API key | `ibm-cloud/ibm` |
| Linode | Personal Access Token (scoped) | `linode/linode` |

The pattern established by `opentofux` is:
1. Write a provider-specific `.tf` template to a temp dir.
2. Run `tofu init` + `tofu apply`.
3. Read outputs, write to kind bootstrap Secret via `kindsync`.
4. On `--purge`, run `tofu destroy`.

## Decision

### 1. Per-provider embedded HCL

Each provider gets a standalone `hcl.go` (or equivalent embedded file) inside its own package
(`internal/provider/<name>/hcl.go`), following exactly the pattern established by
`internal/platform/opentofux/hcl.go` for Proxmox. The HCL content is stored as a Go `const`
string and written to the state directory at runtime by a `WriteEmbeddedFiles` helper local to
the provider.

There is no shared mega-template. Each provider's HCL encodes the credential model, IAM
primitives, and TF provider source that are specific to that cloud. Sharing a single template
across providers would make the HCL harder to audit and version independently.

### 2. Shared `Runner` in `opentofux` (Phase G refactor target)

Today, `internal/platform/opentofux/` contains Proxmox-specific helpers (`runTofu`,
`applyVars`, `tofuEnv`, `GetOutput`, `StateRmAll`, `DestroyIdentity`) that are tightly coupled
to Proxmox config fields and the BPG provider. Phase G begins with **extracting a generic
`Runner` struct** from those helpers:

```
opentofux.Runner{
    StateDir   string          // ~/.yage/<provider>-identity-terraform/
    Env        []string        // provider auth env vars (passed by each provider)
    Vars       []string        // -var flags for tofu apply/destroy
    OutputKeys []string        // expected tofu output names
}
```

`Runner` owns the cross-cutting logic:

- `Init()` — `tofu init -upgrade`
- `Apply()` — `tofu apply -auto-approve` with the runner's vars
- `GetOutput(name)` — `tofu output -raw`
- `StateRmAll()` — walk `tofu state list` and remove each entry
- `Destroy()` — `tofu destroy -auto-approve` with the runner's vars

Each provider's `EnsureIdentity` constructs a `Runner` with its own HCL string, auth env, and
`-var` set. The Proxmox `ApplyIdentity` function is refactored to use this struct as its first
consumer, keeping backward compatibility while enabling the seven new providers to plug in
without duplicating shell-invocation logic.

### 3. State directory convention

Tofu state lives on the **operator's local filesystem**, under
`~/.yage/<provider>-identity-terraform/terraform.tfstate`. The Proxmox reference uses
`~/.yage/proxmox-identity-terraform/`. Phase G providers follow the same convention:
`~/.yage/aws-identity-terraform/`, `~/.yage/azure-identity-terraform/`, and so on.

This is an intentional choice: the state file is the source of truth for credential rotation
and destroy operations. `DestroyIdentity` (called by `Purge`) reads inputs back from the
state file before running `tofu destroy -auto-approve`; losing the state file orphans
cloud-side resources. Keeping state on the operator host (not in the kind cluster) means it
survives kind cluster teardown and remains recoverable across re-runs. The purge flow handles
cleanup in the correct order: `tofu destroy` first, then `os.RemoveAll(stateDir)`.

Kind Secrets hold the **outputs** (minted credentials), not the tofu state itself. After
`tofu apply`, each provider calls `kindsync.SyncBootstrapConfigToKind` to push the outputs
into the provider's bootstrap Secret, mirroring the Proxmox
`GenerateConfigsFromOutputs` → `SyncBootstrapConfigToKind` path.

### 4. `TofuManaged` flag per provider

Each non-Proxmox provider config struct gains a `TofuManaged bool` field:

```
YAGE_AWS_TOFU_MANAGED=true
YAGE_AZURE_TOFU_MANAGED=true
...
```

`EnsureIdentity` checks this flag as its first step. When `TofuManaged` is `false` (the
default), it returns `provider.ErrNotApplicable` immediately — the operator is expected to
supply credentials out-of-band via environment variables or a pre-existing kind Secret.

This flag is **not** added to Proxmox. Proxmox's `EnsureIdentity` remains always-on (the
existing behavior) for backward compatibility: every Proxmox bootstrap has always minted
credentials via OpenTofu, and operators expect it.

### 5. Opt-in for all non-Proxmox providers

All seven new providers listed in the Context table are **opt-in** by default
(`TofuManaged=false`). The operator explicitly enables Phase G for a given provider by
setting the corresponding env var. This avoids surprising cloud-side IAM mutations when
operators already have credential workflows in place.

## Consequences

### Positive

- **Uniform identity bootstrap across providers.** Every provider that activates `TofuManaged`
  gets idempotent, auditable, state-tracked credential minting with no per-provider shell
  scripts. The `tofu apply` / `tofu destroy` lifecycle is consistent and self-documenting via
  the HCL.

- **No code duplication in shell invocation.** Extracting the shared `Runner` from the current
  Proxmox-specific helpers means the eight providers (Proxmox + seven new ones) share a single
  implementation of `tofu init`, `tofu apply`, `tofu output`, and `tofu destroy` semantics.
  Only the HCL content and auth env differ.

- **Operator escape hatch is always available.** `TofuManaged=false` (the default for all new
  providers) means operators who already manage credentials externally experience no change.
  Phase G is purely additive.

- **State survives kind cluster teardown.** Because state is on the operator host
  (`~/.yage/<provider>-identity-terraform/`), a kind cluster rebuild does not orphan
  cloud-side resources. `Purge` runs `tofu destroy` before removing the state directory,
  preserving the correct cleanup ordering.

### Negative / Risks

- **Each provider requires its TF provider version to be pinned.** The HCL `required_providers`
  block must specify a tested version. Unpinned providers will drift on `tofu init -upgrade`,
  potentially breaking apply. Each provider's `hcl.go` must be reviewed when the upstream TF
  provider releases breaking changes.

- **State file on the operator host is a single point of failure for destroy.** If
  `~/.yage/<provider>-identity-terraform/terraform.tfstate` is lost before `--purge` runs,
  yage cannot `tofu destroy` the cloud-side resources and they must be cleaned up manually.
  Future mitigation: replicate state to a kind Secret as a backup, or document a recovery
  procedure using `tofu import`.

- **`tofu` binary must be present on the operator host.** Phase G providers inherit the same
  requirement Proxmox already imposes. The orchestrator's dependency-install phase must be
  extended to gate on `tofu` availability when any provider has `TofuManaged=true`.

- **Cloud IAM permissions required at bootstrap time.** For AWS/Azure/GCP/etc., the operator
  must supply an admin-level credential (IAM admin, Contributor, Owner) so OpenTofu can mint
  the restricted runtime credential. These admin creds must not be persisted beyond the
  bootstrap phase — the pattern is: pass via env, apply, outputs synced to kind, env cleared.

### Implementation sequencing

1. **This ADR accepted** — establishes the interface contract and `Runner` design.
2. **Backend: extract `Runner` from `opentofux`** — prerequisite for all new providers;
   Proxmox `ApplyIdentity` becomes `Runner`'s first consumer.
3. **Issue #80 (OpenStack `EnsureIdentity`)** — sequenced first among the seven providers
   because OpenStack already has a `clouds.yaml` templating spike; serves as the second
   `Runner` consumer and validates the generic approach.
4. **Issues #79 and follow-ons** — remaining six providers (AWS, Azure, GCP, OCI, IBMCloud,
   Linode) each get a tracking issue covering: `hcl.go`, `TofuManaged` flag in config,
   `EnsureIdentity` wiring, and `Purge` integration.
5. **Integration test** — a CAPD-backed integration test that stubs `tofu` with a fake binary
   validates the `Runner` contract end-to-end without requiring real cloud credentials.

## References

- abstraction-plan.md §21.2 — full design narrative
- `internal/platform/opentofux/` — Proxmox reference implementation
- Issue #80 — OpenStack `EnsureIdentity` clouds.yaml templating (sequenced before Phase G)
