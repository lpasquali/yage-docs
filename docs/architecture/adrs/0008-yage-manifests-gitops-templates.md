# ADR 0008 — yage-manifests: GitOps YAML Template Repository

**Status:** Proposed
**Date:** 2026-04-30
**Owners:** Architect (interface contract, repository structure), Backend (Fetcher implementation, per-package migration)

---

## Context

yage generates all Kubernetes YAML manifests, Helm values, ArgoCD Application resources,
and CAPI add-on resources as Go string-building code. The current surface is approximately
3,100 lines spread across six packages:

| Package | Lines | Role |
|---|---|---|
| `internal/capi/manifest/` | 1,228 | Workload cluster YAML patching + K3s template generation |
| `internal/capi/caaph/` | 861 | CAAPH HelmChartProxy YAML + ArgoCD Operator install CR |
| `internal/capi/wlargocd/` | 409 | Workload ArgoCD Application YAML (one per add-on) |
| `internal/capi/postsync/` | 223 | ArgoCD PostSync-hook helpers + CSI smoke-test renderers |
| `internal/capi/cilium/` | 169 | Cilium CNI helper manifests |
| `internal/capi/helmvalues/` | 251 | Helm values YAML for metrics-server, observability stack |
| `internal/csi/*/RenderValues()` | ~14 pkgs | Per-driver Helm values YAML |

All of these use `strings.Builder` + `fmt.Fprintf` to assemble YAML inline. This approach
has several compounding costs:

- YAML output is not reviewable except by reading Go source or running yage.
- Operators cannot audit or fork the templates without forking the binary.
- Tests must compare string output against hardcoded literals, which makes template changes
  expensive and diff-noisy.
- Adding a new add-on requires modifying and rebuilding yage even when the logic is purely
  structural YAML.
- There is no mechanism to pin a manifest version independently of a yage release.

The project owner's architectural direction: move all YAML templating out of Go code into a
dedicated GitOps repository, following the same pattern established by ADR 0004 for
OpenTofu modules (`yage-tofu`).

---

## Decision

### 1. `yage-manifests` public repository

A new public GitHub repository `lpasquali/yage-manifests` holds all YAML templates and
Helm values files. yage contains **no inline YAML assembly**. The repository structure is:

```
yage-manifests/
├── cluster/            # CAPI workload cluster manifests per provider
│   ├── proxmox/
│   ├── aws/
│   └── ...
├── addons/             # Helm values + ArgoCD Application YAMLs per add-on
│   ├── cilium/
│   ├── argocd/
│   ├── metrics-server/
│   ├── observability/
│   └── ...
├── csi/                # Per-driver Helm values (replaces csi.RenderValues)
│   ├── hcloud/
│   ├── aws-ebs/
│   └── ...
└── postsync/           # ArgoCD PostSync hook manifests
```

Modules are versioned and tagged independently of yage releases. Operators can audit and
fork `yage-manifests` without touching the yage binary.

### 2. Templating engine

Template files use Go `text/template` syntax with the extension `.yaml.tmpl`. Rationale:
yage is a Go binary; `text/template` is stdlib, requires no external process or binary,
and template rendering is in-process with zero additional dependencies. No Helm,
no envsubst, no Kustomize — just Go template syntax.

One authoring constraint to note: the CAAPH `HelmChartProxy` resource embeds a
`valuesTemplate:` block that itself contains `{{ }}` CAAPH-side template expressions.
When these files move to `yage-manifests`, those inner `{{`/`}}` delimiters must be
escaped as `{{"{{"}}` / `{{"}}"}}` (or the template must use a custom delimiter pair).
This is documented in the per-template comment header and is the responsibility of the
Backend agent during migration.

### 3. Fetch and render mechanism

The `internal/platform/manifests` package introduces a `Fetcher` struct that mirrors the
pattern established in `internal/platform/opentofux` for the `yage-tofu` checkout. The two
Fetchers are independent implementations — the manifests Fetcher is a new package, not a
generic extraction of opentofux. Extracting a shared git-fetch base is deferred until the
duplication causes pain.

```go
// internal/platform/manifests/fetcher.go
type Fetcher struct {
    RepoURL  string  // https://github.com/lpasquali/yage-manifests
    Ref      string  // cfg.ManifestsRef (env: YAGE_MANIFESTS_REF, default: latest tag)
    CacheDir string  // ~/.yage/manifests-cache/
}

// Render fetches (clone on first use; git fetch + checkout on subsequent runs)
// the yage-manifests repo at the configured Ref, reads the .yaml.tmpl at
// templatePath (relative to repo root), executes it with data via text/template,
// and returns the rendered YAML string.
func (f *Fetcher) Render(templatePath string, data any) (string, error)
```

yage replaces each `strings.Builder`-based renderer with a `manifests.Fetcher.Render()`
call, passing the appropriate `templatePath` and the relevant section of `*config.Config`
(or a purpose-built data struct) as `data`.

### 4. Patch vs. template distinction

`internal/capi/manifest/patches.go` does **regex-based mutation of externally-supplied
YAML documents** — the inputs are manifests rendered by `clusterctl generate cluster`,
not content yage generates from scratch. This is a fundamentally different operation from
templating, and it stays in Go.

The four patch functions that remain in `internal/capi/manifest/`:

| Function | What it mutates |
|---|---|
| `ApplyRoleResourceOverrides` | Sizing fields in `ProxmoxMachineTemplate` blocks |
| `PatchProxmoxCSITopologyLabels` | CSI topology `node-labels` in `KubeadmConfig` |
| `PatchKubeadmSkipKubeProxyForCilium` | `skipPhases: - addon/kube-proxy` in `KubeadmControlPlane` |
| `PatchProxmoxMachineTemplateSpecRevisions` | PMT name revision hashes |

These functions receive clusterctl output and apply structural corrections. They are not
candidates for `yage-manifests` because their input is dynamic (operator-provided, not
yage-generated). Future work may introduce a cleaner patch API internally, but that is
out of scope for this ADR.

`caaph.PatchClusterCAAPHHelmLabels` likewise stays in Go — it mutates a live Cluster
object in the management cluster via a JSON merge-patch and also updates the on-disk
manifest in-place. The logic is imperative, not template-shaped.

### 5. `YAGE_MANIFESTS_REF` config field

A new config field `cfg.ManifestsRef string` (env: `YAGE_MANIFESTS_REF`, default: `v0.1.0`
initially, then tracking the latest stable tag). Analogous to `YAGE_TOFU_REF` from
ADR 0004.

The Fetcher uses this ref to check out the `yage-manifests` repository. Operators can pin
to a specific tag for reproducibility. Air-gapped environments must mirror
`lpasquali/yage-manifests` and set `YAGE_MANIFESTS_REF` to an internally reachable URL.

### 6. `csi.Driver.RenderValues()` migration

The current CSI driver interface declares:

```go
RenderValues(cfg *config.Config) (string, error)
```

After migration, this becomes:

```go
Render(fetcher *manifests.Fetcher, cfg *config.Config) (string, error)
```

Each driver's implementation becomes a one-line call:

```go
func (d driver) Render(f *manifests.Fetcher, cfg *config.Config) (string, error) {
    return f.Render("csi/"+d.Name()+"/values.yaml.tmpl", cfg)
}
```

This is a breaking interface change for all 14 registered driver packages. It is sequenced
as a single atomic PR that updates `internal/csi/driver.go` and all 14 implementations
together.

### 7. Migration sequencing

1. **Scaffold** — Create `lpasquali/yage-manifests` with the directory structure above.
   Populate README stubs per subdirectory; no templates yet.
2. **Fetcher** — Backend: implement `internal/platform/manifests.Fetcher` (clone/pull
   logic, `Render` method, cache at `~/.yage/manifests-cache/`, `YAGE_MANIFESTS_REF` config
   field). Unit-testable with a local file fixture (no network required in CI).
3. **Per-package migration** (can be parallelised per package):
   - Port template content from Go string-builder to `.yaml.tmpl` in `yage-manifests`.
   - Replace the Go renderer with a `Fetcher.Render()` call.
   - Update tests to use a local file fixture pointing at a `testdata/` copy of the
     template, so tests do not require network access.
4. **CSI interface change** — Update `internal/csi/driver.go` interface and all 14 driver
   packages in one PR. Retire the `RenderValues` name.
5. **Package retirement** — Once all callers are migrated, retire `internal/capi/helmvalues/`,
   `internal/capi/wlargocd/`, and `internal/capi/postsync/` as Go packages. Keep
   `internal/capi/caaph/` for the Kubernetes client logic; only its string-building
   helpers move out.

---

## Consequences

### Positive

- **YAML is auditable outside the binary.** Operators can read, diff, and fork
  `yage-manifests` without building or running yage.

- **Manifest version is independently pinnable.** `YAGE_MANIFESTS_REF` decouples
  template evolution from yage releases, giving operators a stable surface for controlled
  upgrades.

- **Go packages are smaller and focused.** `helmvalues/`, `wlargocd/`, and `postsync/`
  become no-op packages on the retirement path; `caaph/` is left with only Kubernetes
  client logic.

- **New add-ons require no Go change.** A new ArgoCD Application or Helm values block is
  a PR against `yage-manifests` only, not a yage rebuild.

- **Tests become structural.** Instead of comparing `strings.Builder` output against
  hardcoded literals, tests validate template rendering against known fixtures and verify
  that the Fetcher resolves the correct path.

### Negative

- **Network dependency at bootstrap.** yage must clone `yage-manifests` on first use.
  Mitigated by local cache; subsequent runs only require a `git fetch`. Air-gapped
  environments must mirror the repository.

- **Additional config surface.** `YAGE_MANIFESTS_REF` is a new env var / flag that
  operators must understand and optionally pin. Documentation and a sensible default
  (`v0.1.0`) reduce friction.

- **Migration work across ~15 packages.** All `strings.Builder` renderers must be ported.
  The migration is mechanical but large. Sequencing per-package migration (step 3) allows
  incremental merges and keeps PRs reviewable.

- **CSI interface break.** Renaming `RenderValues` to `Render` requires a coordinated
  change across all 14 driver packages. This is a one-time cost with no ongoing
  maintenance burden.

### Risks

- **Template drift.** If `yage-manifests` advances independently of yage, a `YAGE_MANIFESTS_REF`
  bump may introduce templates that reference config fields not yet present in the installed
  yage binary. Mitigation: templates should only reference fields present in `config.Config`;
  any new field required by a template must land in yage first. The `yage-manifests`
  changelog must document required minimum yage versions for each tag.

- **Double-template authoring trap.** CAAPH `HelmChartProxy` resources embed
  `valuesTemplate:` blocks that contain CAAPH's own `{{ }}` expressions. Template authors
  must escape these inner delimiters. Failing to do so silently produces broken YAML.
  A lint step in the `yage-manifests` CI (e.g. `go template validate` or a custom check)
  should catch unescaped inner delimiters before merge.

---

## References

- ADR 0004 — Universal OpenTofu Identity Bootstrap; establishes the `yage-tofu` / `Fetcher` / pinned-ref pattern this ADR mirrors
- `lpasquali/yage-manifests` — public repository of YAML templates (target)
- `internal/capi/manifest/patches.go` — patch functions that stay in Go
- `internal/capi/caaph/caaph.go` — CAAPH HelmChartProxy + ArgoCD CR renderers (migration target)
- `internal/capi/wlargocd/render.go` — ArgoCD Application renderers (migration target)
- `internal/capi/helmvalues/` — Helm values generators (migration target; retirement path)
- `internal/csi/driver.go` — CSI `Driver` interface (`RenderValues` → `Render`)
