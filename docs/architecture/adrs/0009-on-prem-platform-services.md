# ADR 0009 — On-Prem Platform Services: Bootstrap Registry and Online Issuing CA (Phase H)

**Status:** Accepted (with errata — see end of document)
**Date:** 2026-04-30
**Owners:** Backend (EnsureRegistry, EnsureIssuingCA wiring), Architect (interface contract)

**Related ADRs:**
- ADR 0004 — OpenTofu identity bootstrap; Fetcher+Runner pattern Phase H extends
- ADR 0010 — In-cluster repo cache; registry VM provisioned post-kind so its IP is available before pivot
- ADR 0011 — Pivot yage-state migration; §1 supersedes the local `~/.yage/tofu/<module>/` state-storage wording in §1 and §3 below (see Errata)

---

## Context

yage targets on-premises Proxmox deployments that are often deployed in air-gapped or
partially-gapped environments. The existing airgap surface in the codebase already covers
several operator-supplied knobs:

| Config field | Env var | What it does |
|---|---|---|
| `Airgapped bool` | `YAGE_AIRGAPPED` | Disables every outbound internet call |
| `ImageRegistryMirror string` | `YAGE_IMAGE_REGISTRY_MIRROR` | Prefixes all CAPI/CNI container image pulls with an internal registry URL |
| `HelmRepoMirror string` | `YAGE_HELM_REPO_MIRROR` | Rewrites Helm chart repo URLs to an internal mirror |
| `InternalCABundle string` | `YAGE_INTERNAL_CA_BUNDLE` | Path to a PEM bundle; applied to `http.DefaultTransport` and `SSL_CERT_FILE` for child processes |

These fields are consumed by `internal/platform/airgap/` (`Apply`, `helmrepo.Rewrite`,
`HTTPClient`). cert-manager is already integrated as a platform add-on and is installed on
every workload cluster.

Two gaps remain that block a fully-automated air-gapped on-premises bootstrap:

**Gap 1 — No registry VM.** Before the workload cluster can be brought up, a container
registry must exist to serve CAPI provider images, CNI images, and Helm chart tarballs.
Today this registry must be pre-staged by the operator out-of-band. There is no yage phase
that provisions or seeds the registry. Operators must manually:

1. Provision a VM (or bare-metal host) running a container registry.
2. Push all required images into it.
3. Set `YAGE_IMAGE_REGISTRY_MIRROR` to the registry's URL.
4. Then run yage.

This creates a chicken-and-egg problem: yage can orchestrate infrastructure, but the
infrastructure registry it depends on is outside yage's lifecycle.

**Gap 2 — No issuing CA.** cert-manager's `ClusterIssuer` on the workload cluster has no
signing material until the operator manually creates the issuer Secret containing an
intermediate CA key and certificate. Today yage does not provision or wire this material;
operators must create the Secret before ArgoCD syncs the cert-manager configuration. For
on-prem deployments that own a private PKI, this requires coordinating cert-manager
installation timing with offline CA operations.

**Offline root CA:** The root CA is cold, operator-managed, and single-region. yage must not
automate the root CA lifecycle. The offline root CA is out of scope for automation now and
permanently. yage's existing `--internal-ca-bundle` consumption boundary is correct and does
not change. Phase H only introduces the intermediate/issuing CA layer, which is signed by
the offline root out-of-band by the operator before yage runs.

---

## Decision

### 1. Bootstrap registry — OpenTofu-provisioned Proxmox VM

The registry is provisioned by OpenTofu before the kind cluster phase using a new module
`yage-tofu/registry/` in the `lpasquali/yage-tofu` repository. This follows the same
Fetcher + Runner pattern established by ADR 0004 (Phase G):

- The `Fetcher` component in `internal/platform/opentofux` clones/updates `yage-tofu` at
  `YAGE_TOFU_REF` into `~/.yage/tofu-cache/` (same checkout used by Phase G providers).
- A `Runner` is constructed with `ModuleDir` pointing at `<tofu-cache>/registry/`.
- `tofu apply` provisions a Harbor VM on Proxmox using operator-supplied variables:
  `YAGE_REGISTRY_NODE` (Proxmox node), `YAGE_REGISTRY_VM_FLAVOR` (CPU/RAM template),
  `YAGE_REGISTRY_NETWORK` (bridge name), `YAGE_REGISTRY_STORAGE` (storage pool).
- `tofu output` reads the provisioned VM's IP address.
- yage automatically sets `cfg.ImageRegistryMirror` to `https://<registry-ip>` before
  the kind cluster phase begins. The operator does not need to set `YAGE_IMAGE_REGISTRY_MIRROR`
  manually when `YAGE_REGISTRY_NODE` is configured.

The registry VM runs [Harbor](https://goharbor.io/) (the default) or
[Zot](https://zotregistry.dev/) (opt-in via `YAGE_REGISTRY_FLAVOR=zot`). Harbor is the
default because it supports native multi-region replication (see §5 below). Zot is available
for operators who prefer a lighter footprint and do not need replication.

~~OpenTofu state for the registry lives at `~/.yage/tofu/registry/terraform.tfstate`.~~
**Superseded by ADR 0011 §1 — see Errata E1.** OpenTofu state for the registry is stored
via the `kubernetes` backend as a Secret `tfstate-default-registry` in namespace
`yage-system`, labeled `app.kubernetes.io/managed-by=yage` and
`app.kubernetes.io/component=tofu-state`. On `--purge`, yage runs `tofu destroy` before
deleting the state Secret, following the same destroy-before-cleanup ordering constraint
as all Phase G providers.

Image seeding (pushing CAPI provider images, CNI images, and Helm chart tarballs into the
registry) is an **operator step** for Phase H. A `--seed-registry` automation flag that
calls `crane copy` or equivalent for each required image is deferred to a follow-up issue.
The registry module outputs the push URL and credentials; the operator uses these to seed.

### 2. Bootstrap sequence

The full on-prem air-gapped bootstrap sequence for Phase H is:

```mermaid
sequenceDiagram
    participant Op as Operator
    participant yage
    participant Tofu as OpenTofu (yage-tofu)
    participant Reg as Registry VM (Proxmox)
    participant Kind as kind cluster
    participant WL as Workload cluster
    participant CM as cert-manager

    Op->>yage: yage bootstrap --airgapped
    yage->>Tofu: tofu apply yage-tofu/registry/ (YAGE_REGISTRY_NODE, …)
    Tofu->>Reg: Provision Harbor VM on Proxmox
    Reg-->>Tofu: VM IP
    Tofu-->>yage: registry_ip output
    yage->>yage: Set ImageRegistryMirror = https://<registry-ip>
    Op->>Reg: Seed CAPI/CNI/Helm images (manual, Phase H)
    yage->>Kind: kind create cluster (images from registry mirror)
    yage->>Tofu: tofu apply yage-tofu/issuing-ca/ (intermediate CA vars)
    Tofu-->>yage: ca.crt, tls.key (intermediate CA material)
    yage->>Kind: Create cert-manager issuer Secret in yage-system ns
    yage->>WL: clusterctl generate + apply workload manifests
    yage->>WL: Install cert-manager; apply ClusterIssuer backed by issuer Secret
    WL-->>CM: ClusterIssuer ready (signs workload TLS certs)
    yage->>WL: Optional: clusterctl pivot kind → permanent mgmt cluster
    yage->>WL: Install ArgoCD on workload cluster
```

Steps in order:

1. **kind cluster up** (existing phase — unchanged)
2. **OpenTofu**: `yage-tofu/registry/` provisions the Harbor registry VM on Proxmox;
   yage reads the VM IP and wires `ImageRegistryMirror` automatically.
3. **Seed registry**: operator pushes CAPI/CNI/Helm images into the registry
   (manual step for Phase H; `--seed-registry` is a follow-up).
4. **Bootstrap workload cluster** with `YAGE_IMAGE_REGISTRY_MIRROR` → registry VM IP;
   all image pulls route through the local registry.
5. **Optional pivot**: CAPI management plane moves from kind to a permanent management
   cluster if configured.
6. **cert-manager `ClusterIssuer`** wired to intermediate CA material on the workload
   cluster; signing of workload TLS certs is fully automated from this point.

### 3. Online issuing CA

The intermediate CA is provisioned by a second OpenTofu module `yage-tofu/issuing-ca/`,
also fetched via the same `YAGE_TOFU_REF` checkout. The module takes the offline root CA
material as input variables (`YAGE_ISSUING_CA_ROOT_CERT`, `YAGE_ISSUING_CA_ROOT_KEY`) and
generates an intermediate key pair signed by the root. Outputs are the intermediate CA
certificate (`ca.crt`) and private key (`tls.key`).

yage reads these outputs and creates a Kubernetes `Secret` of type `kubernetes.io/tls` in
the `cert-manager` namespace on the workload cluster before cert-manager is installed:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: yage-issuing-ca
  namespace: cert-manager
type: kubernetes.io/tls
data:
  tls.crt: <base64(intermediate-cert + root-cert chain)>
  tls.key: <base64(intermediate-key)>
```

cert-manager's `ClusterIssuer` manifest (applied by the add-on phase) references this
Secret:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: yage-internal-issuer
spec:
  ca:
    secretName: yage-issuing-ca
```

No Vault, no step-ca, no ACME for Phase H. The intermediate CA approach gives operators
full PKI control (they own the root CA) while automating the issuing CA lifecycle end-to-end
within yage's bootstrap run.

The `yage-tofu/issuing-ca/` module uses the [TLS Terraform provider](https://registry.terraform.io/providers/hashicorp/tls/latest)
to generate the intermediate key pair deterministically. ~~The OpenTofu state for the issuing
CA lives at `~/.yage/tofu/issuing-ca/terraform.tfstate`.~~ **Superseded by ADR 0011 §1 — see
Errata E1.** OpenTofu state for the issuing CA is stored via the `kubernetes` backend as a
Secret `tfstate-default-issuing-ca` in namespace `yage-system` (same label set as the
registry module). On `--purge`, `tofu destroy` is called before deletion of the state
Secret — this invalidates the intermediate key, which is the correct behavior.

### 4. Offline root CA boundary

**Explicit non-goal**: yage never automates the offline root CA. The offline root is cold,
operator-managed, and single-region. This boundary does not change with Phase H.

The operator's workflow with the offline root CA:

1. Generate the root CA key and self-signed certificate offline (air-gapped workstation or
   HSM) using any toolchain (cfssl, openssl, step-ca, EJBCA).
2. Supply the root cert as the `--internal-ca-bundle` PEM path (`YAGE_INTERNAL_CA_BUNDLE`).
   This is already implemented and consumed by `internal/platform/airgap/Apply`.
3. Supply the root cert + key as `YAGE_ISSUING_CA_ROOT_CERT` and `YAGE_ISSUING_CA_ROOT_KEY`
   for the `yage-tofu/issuing-ca/` module. These are passed as `-var` flags to `tofu apply`
   and are never persisted to disk by yage (they exist only as process env for the duration
   of the apply invocation).

The root CA material must never appear in yage's kind Secrets, CURRENT_STATE, or any
persistent store managed by yage. This is enforced by convention and code review.

### 5. Registry HA topology

**Single-region HA**: one registry VM on Proxmox with an active/passive PVC provided by
Proxmox CSI (default storage class). Harbor uses this PVC for its registry storage layer.
No second replica is required for single-region HA because Harbor's registry data is on
a replicated PVC (Longhorn or rook-ceph as opt-in storage backend) that survives VM restart.
The `yage-tofu/registry/` module provisions one VM; HA at the storage layer is handled by
the underlying Proxmox CSI driver.

**Multi-region stretched cluster**: Harbor native replication between two registry VMs, one
per region. yage provisions the primary registry (first region) via `yage-tofu/registry/`.
Operators configure replication targets via a future config field
`YAGE_REGISTRY_REPLICATION_TARGETS` (comma-separated list of secondary registry URLs). This
field is out of scope for Phase H and will be tracked in a follow-up issue. The primary
registry VM is fully functional as a standalone registry without replication configuration.

### 6. KubeVirt — explicit rejection

KubeVirt was evaluated as an alternative host for the bootstrap registry (running the
registry as a `VirtualMachineInstance` Pod on the kind management cluster instead of a
dedicated Proxmox VM). It was rejected for bootstrap use for the following reasons:

- `/dev/kvm` is not available inside Docker Desktop on macOS or Windows, which are the
  primary development environments for yage operators. KubeVirt without KVM falls back to
  QEMU software emulation, which is unacceptably slow for a registry workload.
- KubeVirt adds a nested-virtualization platform requirement that yage does not currently
  detect or enforce at preflight. Silently running in emulation mode would not surface the
  performance degradation clearly to the operator.
- A Proxmox VM provisioned via `yage-tofu/registry/` (the chosen approach) achieves the
  same registry-handoff semantics without KVM. It uses the same OpenTofu lifecycle that
  already exists for identity bootstrap (ADR 0004), and the VM runs at native hypervisor
  speed on the Proxmox host.

KubeVirt remains a valid Day-2 placement option for running workloads on an established
Proxmox cluster. It is not in scope for yage's bootstrap orchestration.

---

## Consequences

### Positive

- **Air-gapped on-premises deployments work end-to-end without manual pre-staging.** yage
  provisions the registry VM, wires the mirror URL, and configures the issuing CA in a
  single bootstrap run. The operator provides the offline root CA material; everything else
  is automated.

- **Operator mental model is consistent with Phase G.** The same pattern — one external
  repo (`yage-tofu`), one Fetcher+Runner lifecycle, one `YAGE_TOFU_REF` pin — covers
  identity bootstrap, registry provisioning, and CA material generation. Operators do not
  learn a new tool or lifecycle.

- **cert-manager `ClusterIssuer` is fully automated.** The issuing CA material is generated,
  applied as a Kubernetes Secret, and wired into cert-manager without any operator
  intervention after the root CA inputs are provided.

- **Offline root CA boundary is unchanged.** Operators retain full control of the root CA.
  The automation boundary is strictly the intermediate CA and below.

- **Registry HA at the storage layer.** Single-region HA is handled by Proxmox CSI
  (Longhorn/rook-ceph), not by running multiple registry replicas. This simplifies the
  `yage-tofu/registry/` module while still providing the durability operators need.

### Negative / mitigations

- **New config surface**: `YAGE_REGISTRY_NODE`, `YAGE_REGISTRY_VM_FLAVOR`,
  `YAGE_REGISTRY_NETWORK`, `YAGE_REGISTRY_STORAGE`, `YAGE_REGISTRY_FLAVOR`,
  `YAGE_ISSUING_CA_ROOT_CERT`, `YAGE_ISSUING_CA_ROOT_KEY` — seven new env vars. Mitigated
  by sensible defaults where possible (`YAGE_REGISTRY_FLAVOR=harbor`) and by the xapiri
  TUI surfacing them as a guided configuration flow.

- **Registry VM provisioning adds ~2–3 minutes to bootstrap time.** The `yage-tofu/registry/`
  apply step provisions a new VM on Proxmox before kind startup. This is a fixed cost on
  first bootstrap; subsequent runs that detect an existing registry (output already in
  OpenTofu state) skip re-provisioning.

- **Image seeding is an operator step for Phase H.** Pushing CAPI/CNI/Helm images into the
  registry is not automated in Phase H. A `--seed-registry` flag that drives `crane copy`
  (or equivalent) for the required image set is a planned follow-up and will be tracked in
  a child issue.

- **Multi-region replication config is deferred.** `YAGE_REGISTRY_REPLICATION_TARGETS` is
  out of scope for Phase H. Single-region deployments are unaffected; multi-region operators
  must configure Harbor native replication manually until the follow-up issue is resolved.

- **Root CA inputs must be supplied as env vars.** `YAGE_ISSUING_CA_ROOT_CERT` and
  `YAGE_ISSUING_CA_ROOT_KEY` hold sensitive key material. Operators running yage in CI must
  use a secrets manager or vault to inject these. This is the standard requirement for any
  credential input to yage's bootstrap phase and is not a new class of risk.

---

## References

- ADR 0004 (Phase G) — Universal OpenTofu identity bootstrap; Fetcher+Runner pattern that
  Phase H extends
- [abstraction-plan.md](abstraction-plan.md) — §21.2 and §21.4 context for the air-gapped
  topology design
- `internal/platform/airgap/` — existing airgap consumption layer
  (`Apply`, `helmrepo.Rewrite`, `HTTPClient`)
- `internal/platform/opentofux/` — Runner/Fetcher reference implementation
- `internal/config/config.go` — `Airgapped`, `ImageRegistryMirror`, `HelmRepoMirror`,
  `InternalCABundle` fields (already implemented)
- `lpasquali/yage-tofu` — OpenTofu module repository; Phase H adds `registry/` and
  `issuing-ca/` modules
- Epic #120 — on-prem platform services
- Issue #121 — this ADR
- Issue #118 — D1: orchestrator CSI wiring (same sprint; independent)

---

## Errata

### E1 — OpenTofu state backend (2026-05-01)

**Affects:** §1 last paragraph (registry module state); §3 last paragraph (issuing-ca
module state).

**Original wording (incorrect):** "OpenTofu state for the registry lives at
`~/.yage/tofu/registry/terraform.tfstate`" (and the analogous local-path wording for
`issuing-ca/`).

**Corrected wording:** OpenTofu state for both Phase H modules is stored via the built-in
`kubernetes` backend as Secrets in the `yage-system` namespace
(`tfstate-default-registry`, `tfstate-default-issuing-ca`), with labels
`app.kubernetes.io/managed-by=yage` and `app.kubernetes.io/component=tofu-state`. There is
no `~/.yage/tofu/<module>/` directory on the operator's workstation — Phase H state is
in-cluster only.

**Reason:** ADR 0011 §1 (dated 2026-05-01) migrates *all* `yage-tofu` modules to the
kubernetes backend so that state Secrets are automatically copied to the management cluster
by the label-based handoff (ADR 0011 §2) during pivot. The Phase H modules (`registry/`,
`issuing-ca/`) are in scope for this convention. yage-tofu PRs #6 (registry) and #7
(issuing-ca) implement the kubernetes backend per ADR 0011 §1; this errata aligns the
ADR 0009 wording with the actual implementation.

**Operational impact of the corrected wording:**

- `--purge` ordering is unchanged (destroy before cleanup); the cleanup target is the
  state Secret in `yage-system`, not a workstation directory.
- The "zero workstation residue" property described in ADR 0010 now holds for all Phase H
  state as well.
- Air-gapped deployments retain the same Phase H sequencing — the kind cluster (which
  hosts the tofu-state Secrets during bootstrap) exists by the time the registry module
  applies, so the kubernetes backend has somewhere to land.
