# ADR 0001 — CSI Driver Registry

**Status:** Accepted
**Date:** 2026-04-30 (retroactive — implementation landed in Wave 3, §20 of abstraction-plan.md)

---

## Context

The original provider abstraction (abstraction-plan.md §1–§13) handled
CSI storage via `Provider.EnsureCSISecret`, a single hook that assumed
one CSI driver per provider. This broke down as the provider matrix grew:

- Proxmox uses the Proxmox CSI plugin.
- AWS uses aws-ebs-csi-driver (+ optional others).
- GCP uses gcp-pd-csi-driver.
- Cross-provider installations may want Longhorn, Rook-Ceph, or
  OpenEBS independent of the underlying cloud.

The one-hook-per-provider model could not express multi-driver
installations or provider-agnostic drivers. `EnsureCSISecret` was
removed from the `Provider` interface.

## Decision

A separate `Driver` interface lives in `internal/csi/` and is
**orthogonal to `Provider`** — a provider can have zero or many
associated drivers. Drivers self-register via `init()` exactly like
providers.

### Interface

```go
package csi

// Driver is the interface every CSI add-on must satisfy.
// All methods are stateless; configuration comes from *config.Config.
type Driver interface {
    // Name returns the driver's registry key, e.g. "aws-ebs".
    Name() string

    // K8sCSIDriverName is the value in spec.provisioner of the
    // CSIDriver object, e.g. "ebs.csi.aws.com".
    K8sCSIDriverName() string

    // HelmChart returns the repo URL + chart name + pinned version.
    HelmChart() HelmChartRef

    // RenderValues returns the Helm values for this installation,
    // derived from cfg. May return nil (use chart defaults).
    RenderValues(cfg *config.Config) map[string]any

    // EnsureSecret creates or updates the cloud-credential Secret
    // the driver needs. Returns ErrNotApplicable when the driver
    // uses cloud-native identity (IRSA, Workload Identity, etc.)
    // rather than a credential Secret.
    EnsureSecret(cfg *config.Config, kubeconfigPath string) error

    // DefaultStorageClass returns the StorageClass name yage will
    // mark default. If multiple drivers are active, precedence is
    // cfg.CSI.DefaultDriver; if unset, first driver in the list wins.
    DefaultStorageClass() string

    // DescribeInstall emits dry-run plan bullets (Section + Bullet
    // calls on the plan.Writer). Called by the orchestrator's plan
    // path for each active driver.
    DescribeInstall(w plan.Writer, cfg *config.Config)
}
```

### Registration

```go
// In internal/csi/awsebs/awsebs.go
func init() { csi.Register(driver{}) }
```

### Relation to Provider interface

`Provider` no longer has an `EnsureCSISecret` method. CSI wiring is
driven entirely by `cfg.CSI.Drivers` (an ordered list of driver names)
plus `cfg.CSI.DefaultDriver`. The orchestrator iterates the list and
calls `driver.EnsureSecret` + `driver.HelmChart` + `driver.RenderValues`
for each.

Providers that previously called `capi/csi.ApplyConfigSecretToWorkload`
directly are now replaced by `orchestrator → csi.Get(name) → EnsureSecret`.

### Default driver selection

Each provider declares a preferred default in `internal/csi/defaults.go`:

```go
var ProviderDefaults = map[string][]string{
    "proxmox":      {"proxmox-csi"},
    "aws":          {"aws-ebs"},
    "azure":        {"azure-disk"},
    "gcp":          {"gcp-pd"},
    "hetzner":      {"hcloud-volumes"},
    "digitalocean": {"do-block-storage"},
    // …
}
```

If `cfg.CSI.Drivers` is empty the orchestrator falls back to
`ProviderDefaults[cfg.InfraProvider]`.

## Drivers (as of 2026-04-30)

| Driver key | K8s provisioner | Status |
|---|---|---|
| `proxmox-csi` | `csi.proxmox.sinextra.dev` | Landed |
| `aws-ebs` | `ebs.csi.aws.com` | Landed |
| `azure-disk` | `disk.csi.azure.com` | Landed |
| `gcp-pd` | `pd.csi.storage.gke.io` | Landed |
| `hcloud-volumes` | `csi.hetzner.cloud` | In flight (Agent B) |
| `do-block-storage` | `dobs.csi.digitalocean.com` | In flight (Agent B) |
| `linode-block-storage` | `linodebs.csi.linode.com` | In flight (Agent B) |
| `oci-block-storage` | `blockvolume.csi.oraclecloud.com` | In flight (Agent B) |
| `ibm-vpc-block` | `vpc.block.csi.ibm.io` | In flight (Agent B) |
| `openstack-cinder` | `cinder.csi.openstack.org` | In flight (Agent B) |
| `vsphere-csi` | `csi.vsphere.volume` | In flight (Agent B) |
| `longhorn` | `driver.longhorn.io` | In flight (Agent B) |
| `rook-ceph` | `rook-ceph.rbd.csi.ceph.com` | In flight (Agent B) |
| `openebs` | `openebs.io/local` | In flight (Agent B) |

## Consequences

**Good:**
- Multi-driver installations are first-class.
- Provider-agnostic drivers (Longhorn, Rook-Ceph, OpenEBS) work on
  any provider.
- Each driver ships its own Helm values; no central Helm-values file
  grows unboundedly.
- `DescribeInstall` integrates cleanly with the plan-output system
  from Phase B (same `plan.Writer`).

**Trade-offs:**
- `cfg.CSI.Drivers` is a new config surface users must understand when
  overriding defaults.
- Orchestrator must wire `csi.Get(name)` instead of calling
  `provider.EnsureCSISecret`. The wiring is in Agent D1's scope
  (§24.2); until it lands, the legacy path still runs.

## No backward compatibility

No legacy CSI paths (`capi/csi.ApplyConfigSecretToWorkload`,
`Provider.EnsureCSISecret`) will be preserved once Agent D1 integrates.
Delete them on merge.
