# ADR 0004 — Universal OpenTofu Identity Bootstrap (Phase G)

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Backend (EnsureIdentity wiring, Fetcher, Runner), Architect (interface contract)

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

### 1. `yage-tofu` public repository

All OpenTofu modules live in a separate public GitHub repository `lpasquali/yage-tofu`.
yage contains **no embedded HCL**. The repo structure is:

```
yage-tofu/
├── aws/main.tf          # IAM user + access key
├── azure/main.tf        # Service Principal + client secret
├── gcp/main.tf          # Service Account + JSON key
├── openstack/main.tf    # Application Credential (Keystone)
├── oci/main.tf          # API key pair
├── ibmcloud/main.tf     # Service ID + API key
├── linode/main.tf       # Personal Access Token (scoped)
└── proxmox/main.tf      # PVE user + token + ACLs (migrated from opentofux)
```

Each module takes provider credentials as input variables and outputs the runtime
credential(s) that CAPI needs. Modules are versioned and tagged independently of yage
releases. Operators can audit and fork `yage-tofu` without touching the yage binary.

### 2. yage runs tofu — it does not generate HCL

At bootstrap time, yage:

1. **Fetches** `yage-tofu` at a pinned tag/ref into a local cache
   (`~/.yage/tofu-cache/`) via the `Fetcher` component in `opentofux`
   (clone on first use, `git fetch` + checkout on subsequent runs).
2. Calls `tofu -chdir=<tofu-cache>/<provider>/ init`.
3. Calls `tofu -chdir=<tofu-cache>/<provider>/ apply -auto-approve`, passing
   credentials as `-var` flags or provider auth env vars.
4. Reads outputs via `tofu -chdir=<tofu-cache>/<provider>/ output -raw <key>`.
5. Syncs outputs to the provider's kind bootstrap Secret via `kindsync`.
6. On `--purge`, calls `tofu -chdir=<tofu-cache>/<provider>/ destroy -auto-approve`,
   then removes the per-provider state directory. The `tofu destroy` step must run
   **before** `os.RemoveAll(stateDir)` — the state file is the source of truth for
   destroy; losing it orphans cloud-side resources.

Per-provider tofu state lives under `~/.yage/tofu/<provider>/terraform.tfstate`.
This is separate from the source cache. State on the operator host survives kind
cluster teardown and remains recoverable across re-runs.

The pinned `yage-tofu` tag/ref is a config field (`YAGE_TOFU_REF`, default: latest
stable tag). This lets operators pin to a known-good version for reproducibility.

### 3. Shared `Runner` in `opentofux` (revised)

Phase G begins by extracting a generic `Runner` struct from the existing Proxmox-specific
`opentofux` helpers (`runTofu`, `applyVars`, `tofuEnv`, `GetOutput`, `StateRmAll`,
`DestroyIdentity`). The revised struct is:

```go
opentofux.Runner{
    ModuleDir  string   // absolute path to the provider's module within the yage-tofu checkout
    Env        []string // provider auth env vars
    Vars       []string // -var flags for tofu apply/destroy
    OutputKeys []string // expected tofu output names
}
```

`Runner` is responsible for `tofu -chdir=ModuleDir init/apply/output/destroy`. It owns
the cross-cutting logic:

- `Init()` — `tofu init -upgrade`
- `Apply()` — `tofu apply -auto-approve` with the runner's vars
- `GetOutput(name)` — `tofu output -raw`
- `StateRmAll()` — walk `tofu state list` and remove each entry
- `Destroy()` — `tofu destroy -auto-approve` with the runner's vars

The `yage-tofu` checkout itself is managed by the `Fetcher` component in `opentofux`
that clones/updates the repo to `~/.yage/tofu-cache/`. Each provider's `EnsureIdentity`
constructs a `Runner` with `ModuleDir` pointing at the appropriate subdirectory inside
the fetched checkout.

The Proxmox `ApplyIdentity` function is refactored to use `Runner` as its first consumer,
keeping backward compatibility while enabling the seven new providers to plug in without
duplicating shell-invocation logic.

### 4. `TofuManaged` flag, opt-in, Proxmox backward compat

Each non-Proxmox provider config struct gains a `TofuManaged bool` field (default `false`):

```
YAGE_AWS_TOFU_MANAGED=true
YAGE_AZURE_TOFU_MANAGED=true
...
```

`EnsureIdentity` checks this flag as its first step. When `TofuManaged=false`, it returns
`provider.ErrNotApplicable` immediately — the operator supplies credentials out-of-band via
environment variables or a pre-existing kind Secret. This is purely additive: operators who
already manage credentials externally experience no change.

`TofuManaged` is **not** added to Proxmox. Proxmox's `EnsureIdentity` remains always-on
for backward compatibility — every Proxmox bootstrap has always minted credentials via
OpenTofu. However, Proxmox's HCL is **migrated** from `internal/platform/opentofux/` to
`yage-tofu/proxmox/main.tf` as part of Phase G, and `opentofux` is updated to use the
shared `Runner` pointing at that module. This is a breaking change for operators running
`opentofux` directly; a migration guide is required.

## Consequences

### Positive

- **No HCL in the yage binary.** The binary stays clean; operators can audit and fork
  `yage-tofu` independently without touching yage itself.

- **All providers share the same tofu invocation path.** One `Runner` implementation
  serves all eight providers (Proxmox + seven new ones). Only `ModuleDir`, auth env,
  and `-var` set differ per provider.

- **`yage-tofu` modules are versioned independently of yage releases.** Operators can
  pin `YAGE_TOFU_REF` to a specific tag for reproducibility, and the tofu modules can
  be updated, audited, or forked without a yage release.

- **Operator escape hatch is always available.** `TofuManaged=false` (the default for
  all new providers) means the feature is purely opt-in and additive.

- **State survives kind cluster teardown.** Per-provider state at
  `~/.yage/tofu/<provider>/terraform.tfstate` is on the operator host, not in the kind
  cluster. `Purge` runs `tofu destroy` before removing the state directory, preserving
  correct cleanup ordering.

### Negative / Risks

- **Network dependency at bootstrap time.** yage must clone/fetch `yage-tofu` before
  any `EnsureIdentity` call can proceed. Mitigated by local cache at `~/.yage/tofu-cache/`;
  subsequent runs only require a `git fetch`.

- **If `yage-tofu` is unavailable** (GitHub outage, fork unreachable), identity bootstrap
  fails. Mitigated by pre-seeding the cache or using a mirror. Operators with air-gapped
  environments must mirror `lpasquali/yage-tofu` internally.

- **Proxmox migration is a breaking change.** Existing Proxmox operators running
  `opentofux` directly will need to migrate to `yage-tofu/proxmox/`. A migration guide
  documenting the state directory path and `YAGE_TOFU_REF` pin is required alongside
  the Phase G backend PR.

- **`tofu` binary must be present on the operator host.** Phase G providers inherit the
  same requirement Proxmox already imposes. The orchestrator's dependency-install phase
  must gate on `tofu` availability when any provider has `TofuManaged=true` (or for
  Proxmox unconditionally).

- **Cloud IAM permissions required at bootstrap time.** The operator must supply an
  admin-level credential so OpenTofu can mint the restricted runtime credential. These
  admin creds must not be persisted beyond the bootstrap phase — the pattern is: pass via
  env, apply, outputs synced to kind Secret, env cleared.

### Implementation sequencing

1. Create `lpasquali/yage-tofu` repo with modules for all 7 providers and Proxmox.
2. Backend: implement `Fetcher` in `opentofux` for `yage-tofu` clone/update; add
   `YAGE_TOFU_REF` config field.
3. Backend: extract `Runner` from `opentofux`; migrate Proxmox `ApplyIdentity` to use
   `yage-tofu/proxmox/` via `Runner` (publish migration guide).
4. Backend: wire remaining 7 providers' `EnsureIdentity` using `Runner` + their respective
   module dirs and `TofuManaged` guard.
5. Integration test: a CAPD-backed test that stubs `tofu` with a fake binary validates the
   `Runner` + `Fetcher` contract end-to-end without requiring real cloud credentials.

## References

- abstraction-plan.md §21.2 — full design narrative
- `lpasquali/yage-tofu` — public repository of OpenTofu modules (Phase G target)
- `internal/platform/opentofux/` — Proxmox reference implementation (to be refactored)
- Issue #80 — OpenStack `EnsureIdentity` clouds.yaml templating (sequenced before Phase G)
