# ADR 0011 — Pivot: yage State Migration to the Management Cluster

**Status:** Accepted
**Date:** 2026-05-01
**Owners:** Backend (handoff extension, EnsureManagementStorageClass, EnsureRepoSync wiring), Architect (interface contract)

**Related ADRs:**
- ADR 0001 — CSI driver registry; management-cluster CSI installation path introduced by §4
- ADR 0004 — OpenTofu identity bootstrap; tofu-state storage model replaced by §1
- ADR 0010 — In-cluster repo cache; EnsureRepoSync called on mgmt cluster by §5

---

## Context

After a successful pivot, the kind bootstrap cluster is torn down and the CAPI management
cluster (provisioned on the active provider — Proxmox, AWS, vSphere, etc.) becomes the
permanent management plane. The current pivot implementation (`internal/capi/pivot/`) and
`kindsync.HandOffBootstrapSecretsToManagement` leave two categories of yage state behind:

**1. OpenTofu state PVCs (`tofu-state-<module>`).**
ADR 0004 §2 stores `terraform.tfstate` in a per-module PVC in the `yage-system` namespace.
These PVCs live inside the kind Docker container and are destroyed with it.
`HandOffBootstrapSecretsToManagement` does not copy PVC content.

**2. `yage-repos` PVC.**
ADR 0010 stores the `yage-tofu` and `yage-manifests` Git checkouts in a `yage-repos` PVC
in `yage-system`. Any post-pivot orchestrator phase that calls `manifests.Fetcher.Render`
or runs a tofu `JobRunner` requires this PVC on the management cluster.

Additionally, the management cluster is a freshly-provisioned CAPI workload cluster. Unlike
kind (which always has the `standard` StorageClass via local-path-provisioner), the management
cluster has no StorageClass unless a CSI driver is explicitly installed before PVC-dependent
operations run.

---

## Decision

### 1. OpenTofu `kubernetes` backend — state as Secrets, not PVCs

All `yage-tofu` modules are migrated from any local or PVC-based state backend to the
OpenTofu built-in `kubernetes` backend. State is stored as Kubernetes Secrets in the
`yage-system` namespace with a consistent label set, making them automatically copyable by
the label-based handoff described in §2.

Each module gets a `backend.tf` (or equivalent block in `main.tf`):

```hcl
terraform {
  backend "kubernetes" {
    secret_suffix = "<module>"   # e.g. "proxmox", "aws", "openstack"
    namespace     = "yage-system"
    labels = {
      "app.kubernetes.io/managed-by" = "yage"
      "app.kubernetes.io/component"  = "tofu-state"
    }
  }
}
```

The resulting Secret is named `tfstate-default-<module>`. With the
`app.kubernetes.io/managed-by=yage` label, it is automatically included in the extended
handoff (§2).

The `JobRunner` in `internal/platform/opentofux` is updated to pass the in-cluster
backend configuration to `tofu init`. The Job pod runs with the `yage-job-runner`
ServiceAccount, which has RBAC to manage Secrets in `yage-system` (see §3 below).

**Consequence:** the `tofu-state-<module>` PVCs are eliminated. The only PVC remaining in
`yage-system` is `yage-repos`.

### 2. Extended `HandOffBootstrapSecretsToManagement` — label-based yage-system copy

`kindsync.HandOffBootstrapSecretsToManagement` gains a second pass after the existing
named-Secret copy: a label-scoped list of all `yage-system` Secrets carrying
`app.kubernetes.io/managed-by=yage`, applied to the management cluster via server-side apply.

```go
// copyYageSystemSecrets lists all Secrets in namespace matching
// app.kubernetes.io/managed-by=yage on srcCli and applies them to dstCli.
func copyYageSystemSecrets(
    ctx context.Context,
    srcCli, dstCli *k8sclient.Client,
    namespace string,
) (int, error)
```

For each matched Secret, the function strips server-side metadata (same
`stripServerAnnotations` pattern as the existing code) and applies it with
`types.ApplyPatchType`, `Force=true`.

This covers:
- `tfstate-default-<module>` Secrets (tofu state, §1 above)
- Any future yage-managed Secrets that carry the label

The existing named-target list (`expectedHandoffTargets`) is retained for Proxmox-specific
Secrets in other namespaces (e.g. `capmox-system/capmox-manager-credentials`) that do not
carry the label.

`HandOffBootstrapSecretsToManagement` returns an augmented summary:

```go
type HandoffResult struct {
    NamedCopied int
    LabelCopied int
    FirstErr    error
}
```

Errors are warnings (consistent with existing behavior); the orchestrator logs both counts.

### 3. `yage-system` bootstrap on the management cluster (SA + RBAC)

The `yage-job-runner` ServiceAccount and its associated ClusterRole / ClusterRoleBinding
must exist on the management cluster before the first tofu Job runs there. They are not
Secrets and are not copied by the handoff mechanism (§2).

**Resolution:** the orchestrator calls a new helper `kindsync.EnsureYageSystemOnCluster`
immediately after the `yage-system` namespace is created on the management cluster (during
the handoff step §2). This helper re-creates all non-Secret yage-system primitives from
a known in-code spec — not by copying from kind — ensuring idempotency:

```go
// EnsureYageSystemOnCluster creates or updates the yage-system ServiceAccount,
// ClusterRole, and ClusterRoleBinding required for JobRunner operation.
// Idempotent; safe to call on both kind and the management cluster.
func EnsureYageSystemOnCluster(ctx context.Context, cli *k8sclient.Client) error
```

The ClusterRole grants `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` on
`secrets` scoped to `yage-system`. The ClusterRoleBinding binds it to the
`yage-job-runner` ServiceAccount in `yage-system`.

`EnsureYageSystemOnCluster` is also called during the initial kind bootstrap (before
`EnsureRepoSync`) so both clusters share the same setup path.

### 4. CSI driver installation on the management cluster

The management cluster needs a working StorageClass before `EnsureRepoSync` creates the
`yage-repos` PVC. On kind the `standard` (local-path-provisioner) StorageClass is always
present. On a real CAPI cluster it must be installed.

`internal/provider/provider.go` explicitly excludes CSI from the `Provider` interface
(see comment at line ~130: *"The Provider interface intentionally has no CSI hook"*). This
decision is preserved. Instead, the management-cluster CSI install is added to the
`internal/csi.Driver` interface, consistent with ADR 0001's registry pattern:

```go
// EnsureManagementInstall installs this CSI driver (chart + credentials Secret)
// on the management cluster identified by mgmtKubeconfig. Called after
// InstallCAPIOnManagement and before EnsureRepoSync. Idempotent.
// Returns ErrNotApplicable when this driver is not relevant for the
// management cluster (e.g. the provider does not pivot, or the management
// cluster uses a different CSI than the workload cluster).
EnsureManagementInstall(cfg *config.Config, mgmtKubeconfig string) error
```

A default `ErrNotApplicable` implementation is provided via the driver base type so
existing drivers require no change unless they implement a management path.

**Helm rationale:** the workload-cluster CSI is installed via CAAPH `HelmChartProxy`,
which is an ArgoCD-driven mechanism. The management cluster does not run ArgoCD at the
point `EnsureManagementInstall` is called (ArgoCD on the management cluster, if desired,
is a post-pivot operator concern). Therefore `EnsureManagementInstall` uses a direct
Helm invocation — `helm upgrade --install` via `shell.Run` — the same mechanism already
used for workload-cluster operations that precede ArgoCD readiness. The `helm` binary is
a documented dependency for `EnsureIdentity`-class operations and is available in the yage
execution environment.

**Proxmox implementation (`internal/csi/proxmoxcsi`):**

1. Call `proxmoxcsi.EnsureSecret` to apply the Proxmox API credential Secret to the
   management cluster (reuses the existing function, passing the management kubeconfig).
2. Run `helm upgrade --install proxmox-csi-plugin <chart> --namespace kube-system
   --create-namespace --set storageClass.name=<cfg.Providers.Proxmox.CSIStorageClassName>
   --kubeconfig <mgmtKubeconfig>` via `shell.Run`.

The storage class name defaults to `cfg.Providers.Proxmox.CSIStorageClassName` (same as
the workload cluster). Operators who need a different pool for the management cluster can
set a new optional field `PROXMOX_CSI_MGMT_STORAGE_CLASS` (defaults to workload value).

**Orchestrator lookup:** the orchestrator finds the active driver for the current provider
using the existing CSI registry (`csi.For(cfg)` or equivalent) and calls
`EnsureManagementInstall`. If the driver returns `ErrNotApplicable`, the orchestrator waits
up to `cfg.Pivot.StorageClassReadyTimeout` (default: 5 minutes) for any non-local-path
default StorageClass to appear — this handles cloud providers whose controller manager
injects a StorageClass automatically (e.g. AWS EBS SC created by the cloud controller
manager). If none appears within the timeout, the orchestrator logs a fatal error.

### 5. `EnsureRepoSync` on the management cluster

After the handoff (§2), RBAC setup (§3), and CSI install (§4), the orchestrator calls
`EnsureRepoSync` (ADR 0010 §2) targeting the management cluster:

```go
mgmtCli, err := k8sclient.ForKubeconfigFile(mgmtKubeconfig)
// ...
if err := opentofux.EnsureRepoSync(ctx, mgmtCli, cfg); err != nil {
    logx.Die("pivot: EnsureRepoSync on management cluster: %v", err)
}
```

This creates the `yage-repos` PVC on the management cluster (using the StorageClass
provided by §4) and runs the `yage-repo-sync` Job. The `alpine/git:2` image pull uses
`cfg.ImageRegistryMirror` if set. The registry VM (ADR 0009 §1, provisioned post-kind
per ADR 0010 §7) is available by the time pivot runs, so `cfg.ImageRegistryMirror` is
already populated in air-gapped environments.

`EnsureRepoSync` is idempotent: it checks for PVC existence before creating and re-uses
an existing `yage-repos` PVC if found. A re-run after a partial failure retries safely.

### 6. Extended `VerifyParity`

`pivot.VerifyParity` gains three additional checks after the existing CAPI inventory +
provider Secret checks:

**a. yage-system namespace present:** Assert that `yage-system` exists on the management
cluster (created by handoff §2).

**b. Minimum labeled Secrets present:** List Secrets in `yage-system` with
`app.kubernetes.io/managed-by=yage` on the management cluster. Zero is a warning (not
fatal) — a first-run with no tofu modules executed yet has no tfstate Secrets to copy.
One or more is the expected steady-state.

**c. `yage-repos` PVC in `Bound` phase:** Assert that the `yage-repos` PVC in `yage-system`
is `Bound`. A `Pending` PVC here indicates a StorageClass problem; surface it with a clear
diagnostic before the first post-pivot tofu Job runs.

### 7. Updated pivot bootstrap sequence

```
MoveCAPIState                              [existing — clusterctl move]
  ↓
HandOffBootstrapSecretsToManagement        [extended: §2 label-based yage-system copy]
  ↓
EnsureYageSystemOnCluster (on mgmt)        [NEW — §3: SA + RBAC from spec]
  ↓
csi.Driver.EnsureManagementInstall         [NEW — §4: CSI + StorageClass on mgmt]
  ↓
EnsureRepoSync (on mgmt)                   [NEW — §5: yage-repos PVC + repo-sync Job]
  ↓
VerifyParity                               [extended: §6 yage-system + PVC checks]
  ↓
rebindKindContextToMgmt                    [existing]
  ↓
[downstream phases: workload apply, ArgoCD on workload]
  ↓
TeardownKind                               [existing; deferred to post-workload]
```

---

## Consequences

### Positive

- **Complete yage state arrives on the management cluster.** Tofu state (as Secrets),
  repo content (via `yage-repos` PVC), and all yage-system bootstrap Secrets are present
  before any post-pivot phase runs.

- **Tofu-state PVCs are eliminated.** The switch to the `kubernetes` backend removes the
  `tofu-state-<module>` PVC lifecycle entirely.

- **Label-based handoff scales automatically.** New yage-managed Secrets that carry
  `app.kubernetes.io/managed-by=yage` are migrated without code changes to `handoff.go`.

- **ADR 0001 invariant preserved.** The `Provider` interface gains no CSI hook;
  management CSI install lives on `csi.Driver` where it belongs.

- **Consistent zero-workstation-residue (ADR 0010).** No tofu state, no repo content, and
  no yage Secrets live on the operator's workstation disk.

- **Post-pivot idempotency.** All new steps (`EnsureYageSystemOnCluster`,
  `EnsureManagementInstall`, `EnsureRepoSync`) are idempotent. A re-run after a partial
  failure re-creates what is missing.

### Negative

- **`csi.Driver` interface gains a new method.** All registered drivers receive a
  `EnsureManagementInstall` method that defaults to `ErrNotApplicable`. Only drivers used
  on providers that pivot need a real implementation.

- **Management cluster Helm dependency.** `EnsureManagementInstall` for Proxmox uses
  `helm` via `shell.Run`. The `helm` binary must be present in the yage execution
  environment — already a documented requirement for identity-bootstrap-class operations.

- **`kubernetes` backend requires RBAC on both clusters.** `EnsureYageSystemOnCluster` (§3)
  must run before any tofu Job on the management cluster. The orchestrator enforces this
  sequencing in §7.

- **`EnsureRepoSync` is fatal on mgmt.** A network or image-pull failure in
  `yage-repo-sync` on the management cluster blocks pivot completion. The Job is idempotent
  so a re-run of yage post-pivot retries safely.

### Risks

- **Air-gap: management-cluster image pull.** The `yage-repo-sync` Job on the management
  cluster needs `alpine/git:2`. `cfg.ImageRegistryMirror` must be set before pivot. The
  registry VM (ADR 0009, post-kind) is provisioned before pivot runs, so this is satisfied
  in the normal bootstrap flow.

- **Management cluster StorageClass name.** If the CSI driver creates a StorageClass the
  operator does not expect (e.g. `proxmox-data-xfs` when they wanted a different name),
  subsequent HCL modules referencing a specific StorageClass name may fail. Mitigated by
  `PROXMOX_CSI_MGMT_STORAGE_CLASS` config field defaulting to the workload value.

---

## References

- ADR 0001 — CSI driver registry; `Driver` interface extension in §4
- ADR 0004 — Universal OpenTofu Identity Bootstrap; state backend replaced by §1
- ADR 0009 — On-prem platform services; registry VM sequencing (post-kind) ensures `ImageRegistryMirror` is set before pivot
- ADR 0010 — In-cluster repo cache; `EnsureRepoSync` spec, `yage-repos` PVC; §5 of this ADR calls it on the management cluster
- `internal/cluster/kindsync/handoff.go` — `HandOffBootstrapSecretsToManagement` (§2 extension)
- `internal/capi/pivot/pivot.go` — `VerifyParity` (§6 extension)
- `internal/csi/driver.go` — `Driver` interface; `EnsureManagementInstall` method (§4)
- `internal/orchestrator/bootstrap.go` — pivot sequence (§7 updated)
- Issue #144 — `EnsureRepoSync` implementation; must target both kind and management clusters
