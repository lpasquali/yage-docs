# ADR 0012 — yage-manifests: Template Layout, File Naming, and Data Contract

**Status:** Accepted (amended — Errata E1 2026-05-01)
**Date:** 2026-05-01
**Owners:** Architect (template contract, directory layout), Backend (per-package migration #137–#141)

**Amends:** ADR 0008 §1 (directory layout — pinned, not changed), §2 (`text/template` data contract — added), §3 (Fetcher render call — clarified after ADR 0010 supersession)
**Depends on:** ADR 0010 (in-cluster repo cache; the Fetcher resolves under `MountRoot`)

---

## Context

ADR 0008 declared the move of all YAML templating out of yage into the public
`yage-manifests` repository, established the four top-level groups
(`cluster/`, `addons/`, `csi/`, `postsync/`), and chose Go `text/template` as the
engine. ADR 0010 then redirected the Fetcher to read from an in-cluster PVC
mount (`<MountRoot>/yage-manifests/<templatePath>`) instead of a workstation
cache. Issue #136 implements that Fetcher; issues #137–#141 are the per-package
migrations that fill `yage-manifests` with content.

What ADR 0008 did **not** pin, and what #137–#141 cannot land mechanically
without:

1. **File-naming convention** within each group. Today `addons/` could mean
   either "one directory per add-on with `values.yaml.tmpl` + `application.yaml.tmpl`"
   or "two flat directories `helmvalues/` + `wlargocd/`". The scaffold READMEs
   imply the former; nothing makes it normative.
2. **Data contract.** The `yage-manifests` README claims templates "receive a
   `*config.Config` struct". The actual call sites in
   `internal/capi/wlargocd/render.go` take 8+ ad-hoc parameters (name,
   namespace, repo URL, chart, version, sync-wave, values YAML, hook-short)
   that are *not* fields on `config.Config`. Without a pinned wrapper-struct
   convention, every migration PR will invent its own data shape and produce
   the same drift the inline strings have today.
3. **Missing-key policy.** `text/template`'s default behaviour for an
   undeclared `.Field` is to emit `<no value>`, which produces YAML the
   apiserver will accept (an empty `image:` value, an `Application` with no
   destination). This is silent breakage masquerading as a successful render.
4. **Multi-document handling.** `wlargocd` renderers emit single
   `---`-prefixed Application docs that callers concatenate into one
   kubectl-apply stream. The convention (one logical resource per file,
   callers concatenate) needs to be normative so authors don't bundle.
5. **Kustomize-fragment partials.** `internal/capi/postsync/postsync.go`
   produces two artefact kinds: standalone Job manifests and
   *kustomize-block fragments* that Argo Application templates embed inline.
   The fragments are not stand-alone YAML documents; they need a clear home
   so authors don't mix them with full manifests.

This addendum pins those five points and adds a complete migration mapping
from the current Go renderers to their target template paths so #137–#141
become mechanical PRs with no further design questions.

---

## Decision

### 1. Top-level layout — pinned, unchanged from ADR 0008 §1

The four top-level groups remain:

```
yage-manifests/
├── cluster/        # CAPI workload-cluster manifests, per provider
├── addons/         # Per-add-on directory: values.yaml.tmpl + application.yaml.tmpl + extras
├── csi/            # Per-driver Helm values (one directory per CSI driver)
└── postsync/       # ArgoCD PostSync hooks: standalone manifests + kustomize-block partials
```

Rejected alternative: a flat layout that mirrors the current Go packages
(`helmvalues/`, `wlargocd/`, `postsync/`, `caaph/`). That choice would split
a single add-on's templates across two trees (`helmvalues/kyverno/values.yaml.tmpl`
and `wlargocd/kyverno/application.yaml.tmpl`) and lose the per-add-on
fork affordance ADR 0008 was written to deliver. The scaffolded layout is
correct; this addendum only fills it in.

### 2. File-naming convention

| Group | Filename pattern | One file per |
|---|---|---|
| `addons/<addon>/` | `values.yaml.tmpl` | Helm values rendered from config |
| `addons/<addon>/` | `application.yaml.tmpl` | Workload ArgoCD `Application` resource |
| `addons/<addon>/` | `helmchartproxy.yaml.tmpl` | CAAPH `HelmChartProxy` (Cilium, argocd-apps) |
| `addons/<addon>/` | `<resource>.yaml.tmpl` | Any other single-resource manifest (e.g. `lb-ipam-pool.yaml.tmpl` for Cilium) |
| `csi/<driver>/` | `values.yaml.tmpl` | Helm values rendered from config |
| `cluster/<provider>/` | `cluster.yaml.tmpl` | Full multi-doc CAPI workload-cluster manifest |
| `cluster/<provider>/` | `k3s.yaml.tmpl` | K3s variant where applicable |
| `postsync/` | `<hookname>.yaml.tmpl` | Standalone manifest (PVC, Job, Secret) |
| `postsync/_partials/` | `<name>.kustomize.tmpl` | Kustomize-block fragment embedded by Application templates |

Rules that apply to every file:

- One logical resource per `.yaml.tmpl`. Callers concatenate with `\n---\n`
  when a multi-document stream is needed. The single exception is
  `cluster/<provider>/cluster.yaml.tmpl`, which is intrinsically a
  multi-document manifest produced by `clusterctl generate cluster`-style
  composition.
- Lowercase, hyphenated filenames; no underscores in user-facing paths.
  The `_partials/` subdirectory uses a leading underscore to mark
  template-fragment files that are not intended to be applied directly.
- Every file begins with a `# yage-manifests/<path>` header comment for
  grep-locatable provenance.

### 3. Data contract: per-group wrapper structs

Templates do **not** receive `*config.Config` directly. They receive a
purpose-built wrapper struct defined in a single Go package
(`internal/capi/templates/`) and instantiated by the caller of
`Fetcher.Render`. Wrapper-struct pattern justified by: wlargocd renderers
already take 8+ caller-supplied parameters that are not in `config.Config`
(release name, sync-wave, hook-short, indented values blob, etc.); these
must continue to vary per call site.

The five wrapper structs:

```go
package templates

// HelmValuesData is passed to addons/<addon>/values.yaml.tmpl and
// csi/<driver>/values.yaml.tmpl.
type HelmValuesData struct {
    Cfg *config.Config // entire workload config
}

// ArgoApplicationData is passed to addons/<addon>/application.yaml.tmpl.
type ArgoApplicationData struct {
    Cfg          *config.Config
    Name         string        // Application metadata.name
    DestNS       string        // workload destination namespace
    RepoURL      string
    Chart        string        // helm-repo / OCI chart name; "" for git/kustomize sources
    Path         string        // git source path; "" for helm sources
    Ref          string        // git ref / chart targetRevision; pre-shell-quote-escaped
    SyncWave     string
    ReleaseName  string        // optional; "" omits the field
    ValuesYAML   string        // pre-rendered values block (output of HelmValuesData → values.yaml.tmpl)
    Annotations  map[string]string // extra annotations (Kyverno needs ServerSideDiff)
    PostSync     PostSyncBlock // zero value when no hook
}

// PostSyncBlock is the second-source git+kustomize fields for an Argo
// Application. Zero value means "single-source Application; no hook".
type PostSyncBlock struct {
    URL  string
    Path string
    Ref  string
    KustomizePartial string // rendered output of postsync/_partials/<name>.kustomize.tmpl
}

// HelmChartProxyData is passed to addons/<addon>/helmchartproxy.yaml.tmpl.
type HelmChartProxyData struct {
    Cfg              *config.Config
    Name             string  // HelmChartProxy metadata.name
    Namespace        string  // HelmChartProxy metadata.namespace
    ClusterSelector  map[string]string
    ChartName        string
    RepoURL          string  // pre-rewritten via airgap.RewriteHelmRepo
    Version          string
    ChartNamespace   string  // namespace the chart installs into
    ValuesTemplate   string  // pre-built CAAPH-side template body (delimiters already escaped)
}

// PostSyncData is passed to postsync/<hookname>.yaml.tmpl.
type PostSyncData struct {
    Cfg            *config.Config
    Namespace      string
    StorageClass   string  // CSI smoke-test only
    KubectlImage   string
    K8sVersionTag  string  // for image tag derivation
}

// KustomizePartialData is passed to postsync/_partials/<name>.kustomize.tmpl.
type KustomizePartialData struct {
    Cfg          *config.Config
    Namespace    string
    JobName      string
    KubectlImage string
    Extra        map[string]string // env-var overrides etc.
}
```

A migration PR for issue #137–#141 chooses the wrapper struct that matches
the template group, populates the fields it needs, and calls
`fetcher.Render(<templatePath>, <data>)`. The struct lives next to the
caller in `internal/capi/templates/` so type drift surfaces at compile time.

For the CAAPH `HelmChartProxy` case (`HelmChartProxyData.ValuesTemplate`),
the inner `{{ }}` delimiters that CAAPH itself templates at apply-time must
be pre-escaped as `{{` "`{{`" `}}` / `{{` "`}}`" `}}` in the source
template per ADR 0008 §2. This addendum does not re-litigate that
constraint; it is unchanged.

### 4. Missing-key policy: hard error

The Fetcher executes every template with `Option("missingkey=error")`.
Justification: the `text/template` default ("missingkey=default") emits the
literal string `<no value>` for an undeclared field. In a YAML context this
silently produces invalid-but-syntactically-acceptable manifests — an
`Application` with `targetRevision: <no value>`, a `Job` with
`image: <no value>` — that the apiserver will accept and the workload will
hang on. Hard-erroring at render time forces every template to declare
every field it references against the wrapper struct, and surfaces drift
between yage and `yage-manifests` immediately.

The `missingkey=zero` mode is rejected because the zero value for
`*config.Config` is `nil`, which would panic on the next field access; the
zero value for a string is `""`, which produces unquoted empty fields and
the same silent-acceptance failure mode.

This pins the design pick raised by yage-backend (C) on issue #136 in the
affirmative.

### 4.1 Template FuncMap

**Errata E1 — 2026-05-01.** Amends §4 to allow a small, explicitly enumerated
`template.FuncMap` in the `Fetcher`. The `missingkey=error` policy is unchanged.

**Motivation.** Four config fields in `internal/capi/helmvalues/` carry
env-var-origin string values (`"true"`, `"false"`, `"1"`, `"yes"`, …). In Go
`text/template`, `{{ if "false" }}` is truthy (any non-empty string), so the
package-local `isTrue()` semantics cannot be expressed in a template without a
FuncMap. Migrating these fields to `bool` in `config.Config` expands scope
dramatically; pre-evaluating them in every wrapper struct pollutes the data
contract. A single named function is the minimal, auditable fix.

**Admitted function — `isTrue`.**

```
isTrue(string) bool
```

Semantics: identical to `internal/platform/sysinfo.IsTrue` — case-insensitive
match on `"1"`, `"t"`, `"true"`, `"y"`, `"yes"`, `"on"` after whitespace trim.
Admissibility justification: `isTrue` is a value-coercion helper that bridges
env-var-origin string fields to template boolean conditions. It does not
introduce new data, does not enable arbitrary computation, and does not bypass
the wrapper-struct contract.

**Extension rule.** Each future FuncMap addition requires its own ADR amendment
to this section. Admitting Sprig or any multi-function library wholesale is
explicitly rejected — Sprig's 100+ functions weaken the data contract without
providing auditable benefit.

**Implementation contract.** The `Fetcher` must expose a method:

```go
func (f *Fetcher) RegisterFunc(name string, fn any)
```

that registers `fn` under `name` in an internal `template.FuncMap` applied on
every `Render` call. `RegisterFunc` is the only permitted extension point; callers
may not supply arbitrary `FuncMap` values directly. Backend implements this
method per the amended ADR; the Architect does not touch Fetcher code.

**Template usage example.**

```
{{ if isTrue .Cfg.WorkloadMetricsServerInsecureTLS }}
args:
  - --kubelet-insecure-tls
{{ end }}
```

### 5. Multi-document and partial handling

- Every `.yaml.tmpl` outside `cluster/<provider>/cluster.yaml.tmpl` renders
  exactly one Kubernetes resource, with no leading or trailing `---`. The
  caller is responsible for concatenating multiple rendered strings with
  `\n---\n` between them. This matches the current behaviour of
  `wlargocd.Helm`, `wlargocd.HelmGit`, and friends, which emit one
  Application document per call.
- `postsync/_partials/*.kustomize.tmpl` files are *fragments*, not
  documents. They render an indented `kustomize:` block intended for
  embedding inside an `Application` template via the wrapper struct's
  `PostSync.KustomizePartial` field. The leading-underscore subdirectory
  marks them as partials; CI may add a check that no `_partials/` file is
  ever passed to `kubectl apply` directly.

---

## Migration mapping

The table below maps every renderer in the five Go source packages to its
target path in `yage-manifests`. After issues #137–#141 land, the listed
target files exist and the corresponding Go renderer becomes a one-line
`fetcher.Render(...)` call (or is retired entirely under #142).

### `internal/capi/helmvalues/` → `addons/<addon>/values.yaml.tmpl`

| Go function | Target template path | Wrapper struct |
|---|---|---|
| `MetricsServerValues` | `addons/metrics-server/values.yaml.tmpl` | `HelmValuesData` |
| `VictoriaMetricsValues` | `addons/observability/values-victoria-metrics.yaml.tmpl` | `HelmValuesData` |
| `OpenTelemetryValues` | `addons/opentelemetry/values.yaml.tmpl` | `HelmValuesData` |
| `GrafanaValues` | `addons/observability/values-grafana.yaml.tmpl` | `HelmValuesData` |
| `SPIRESubchartTolerations` | inlined in `addons/spire/values.yaml.tmpl` (conditional `{{ if }}` block) | `HelmValuesData` |
| `SPIREValues` | `addons/spire/values.yaml.tmpl` | `HelmValuesData` |
| `KeycloakValues` | `addons/keycloak/values.yaml.tmpl` | `HelmValuesData` |

The two observability values files share the `addons/observability/` directory
because they belong to the same logical add-on (VictoriaMetrics + Grafana
shipped as one stack); the filename suffix disambiguates.

Templates that reference env-var-origin string fields use the `isTrue` FuncMap
function (§4.1) rather than a bare `{{ if }}` expression:
`{{ if isTrue .Cfg.WorkloadMetricsServerInsecureTLS }}`. The same pattern applies
to any `helmvalues` field whose source is an env var that accepts
`"true"/"false"/"1"/"0"` variants.

### `internal/capi/wlargocd/` → `addons/<addon>/application.yaml.tmpl`

`wlargocd.HelmGit`, `wlargocd.Helm`, `wlargocd.HelmOCI`, `wlargocd.KustomizeGit`,
and `wlargocd.Kyverno` are not per-add-on functions — they are five
*Application-shape variants*. The target is therefore one
`application.yaml.tmpl` per add-on directory, and the template's body
selects the variant based on `ArgoApplicationData` field presence
(`{{ if .Chart }}…{{ else if .Path }}…{{ end }}`).

| Go function | Variant | Add-ons that use it (target dirs) |
|---|---|---|
| `wlargocd.HelmGit` | git source, `helm:` block | `addons/spire/`, `addons/cert-manager/` |
| `wlargocd.Helm` | http helm-repo source | `addons/metrics-server/`, `addons/observability/`, `addons/cnpg/`, `addons/external-secrets/`, others |
| `wlargocd.HelmOCI` | OCI helm source, multi-source for proxmox-csi smoke | `addons/proxmox-csi/` |
| `wlargocd.KustomizeGit` | git source, `kustomize:` block | `addons/argocd-apps/` (legacy direct path) |
| `wlargocd.Kyverno` | helm with ServerSideDiff annotation + ignoreDifferences | `addons/kyverno/` |
| `wlargocd.PostSyncBlock` (struct) | embedded in `ArgoApplicationData.PostSync` | n/a — Go-side only |

The `headerAnnot`, `syncPolicyTail`, `indent`, `indentValuesBoth`, and
`shellQuoteEscape` helpers stay in Go: they prepare the data passed into
the wrapper struct (e.g. `ArgoApplicationData.Ref` is shell-quote-escaped
before render) but are not template-shaped.

### `internal/capi/postsync/` → `postsync/`

| Go function | Target | Wrapper struct |
|---|---|---|
| `KustomizeBlockForJob` | `postsync/_partials/job-image-override.kustomize.tmpl` | `KustomizePartialData` |
| `SmokeRenderKustomizeBlock` | `postsync/_partials/proxmox-csi-smoke.kustomize.tmpl` | `KustomizePartialData` |
| `SmokeK8sVersionForImage` | stays in Go (parses an external manifest file) | n/a |
| `SmokeKubectlOCITag` | stays in Go (string normalisation) | n/a |
| `ResolveKubectlImage` | stays in Go (config plumbing) | n/a |
| `DiscoverURL` / `DiscoverRef` / `FullRelpath` | stay in Go (git/env discovery) | n/a |

There are no standalone-manifest functions in `postsync.go` today — the
smoke-test PVC + Job manifests live in the `workload-smoketests` repo,
which `postsync/_partials/proxmox-csi-smoke.kustomize.tmpl` patches via
the kustomize block. The `postsync/` top-level remains for future
standalone-hook manifests; the current migration only fills `_partials/`.

The `argocd-redis-secret.yaml.tmpl` file mentioned in the scaffold's
`postsync/README.md` is a placeholder for future work tracked separately;
it is not produced by any function in the current `postsync.go`.

### `internal/capi/caaph/` → `addons/<addon>/helmchartproxy.yaml.tmpl`

| Go function | Target | Wrapper struct |
|---|---|---|
| `CiliumHelmChartProxyYAML` | `addons/cilium/helmchartproxy.yaml.tmpl` | `HelmChartProxyData` |
| `ApplyWorkloadArgoHelmProxies` (HCP body inline) | `addons/argocd-apps/helmchartproxy.yaml.tmpl` | `HelmChartProxyData` |
| `buildArgoCDCR` | `addons/argocd/argocd-cr.yaml.tmpl` | `HelmValuesData` (uses `Cfg.ArgoCD` fields) |
| `PatchClusterCAAPHHelmLabels` | stays in Go (mutates an external file + live K8s object) | n/a |
| `ApplyWorkloadCiliumHelmChartProxy` | stays in Go (orchestrator-side wiring) | n/a |
| `ApplyWorkloadCiliumLBBToWorkload` | stays in Go (orchestrator-side wiring) | n/a |
| `ApplyWorkloadArgoCDOperatorAndCR` | stays in Go (orchestrator-side wiring; emits the CR doc via `buildArgoCDCR` → template) | n/a |

Both HelmChartProxy templates contain CAAPH `valuesTemplate:` blocks with
inner `{{ }}` delimiters that are escaped per ADR 0008 §2.

### `internal/capi/cilium/AppendLBIPAMPoolManifest` → `addons/cilium/lb-ipam-pool.yaml.tmpl`

The task brief did not list `cilium/` as a migration source, but
`AppendLBIPAMPoolManifest` in `internal/capi/cilium/cilium.go` emits a
`CiliumLoadBalancerIPPool` manifest inline via `fmt.Fprintf`. It belongs
in `addons/cilium/lb-ipam-pool.yaml.tmpl` with `HelmValuesData` (it reads
`Cfg.CiliumLBIPAMPoolName`, `Cfg.CiliumLBIPAMPoolCIDR`, etc.). Including it
here closes the loop on inline-YAML emitters and prevents a stray
`fmt.Fprintf` from surviving the migration.

### `internal/csi/<driver>/RenderValues()` → `csi/<driver>/values.yaml.tmpl`

Per ADR 0008 §6, the 14 driver packages migrate atomically together with
the `Driver.RenderValues` → `Driver.Render` interface rename. The
filename is fixed at `values.yaml.tmpl`; the directory matches the
driver's `Name()`:

| Driver `Name()` | Target |
|---|---|
| `proxmox-csi` | `csi/proxmox-csi/values.yaml.tmpl` |
| `aws-ebs` | `csi/aws-ebs/values.yaml.tmpl` |
| `azure-disk` | `csi/azure-disk/values.yaml.tmpl` |
| `gcp-pd` | `csi/gcp-pd/values.yaml.tmpl` |
| `hcloud` | `csi/hcloud/values.yaml.tmpl` |
| `do-block-storage` | `csi/do-block-storage/values.yaml.tmpl` |
| `oci-block-storage` | `csi/oci-block-storage/values.yaml.tmpl` |
| `linode-block-storage` | `csi/linode-block-storage/values.yaml.tmpl` |
| `openstack-cinder` | `csi/openstack-cinder/values.yaml.tmpl` |
| `vsphere-csi` | `csi/vsphere-csi/values.yaml.tmpl` |
| `ibm-vpc-block` | `csi/ibm-vpc-block/values.yaml.tmpl` |
| `longhorn` | `csi/longhorn/values.yaml.tmpl` |
| `rook-ceph` | `csi/rook-ceph/values.yaml.tmpl` |
| `openebs` | `csi/openebs/values.yaml.tmpl` |

All 14 use `HelmValuesData`.

---

## Consequences

### Positive

- Issues #137–#141 become mechanical PRs. Each migration consists of:
  port the renderer body into the table-listed `.yaml.tmpl`, define or
  reuse the wrapper struct, replace the Go renderer with a `Render` call.
  No design decisions remain.

- Type drift is compile-time. Wrapper structs in
  `internal/capi/templates/` mean adding a field to a template requires
  adding a field to the struct, which surfaces in every caller's build.

- Missing-key=error catches yage/yage-manifests version skew at render
  time, before any apply produces a half-formed manifest the cluster will
  hang on.

- Operators forking `addons/<addon>/` get the full add-on (values + Application
  + HelmChartProxy where applicable) in one directory. This was ADR 0008's
  core fork-affordance promise; pinning per-add-on grouping makes it real.

### Negative

- One new Go package (`internal/capi/templates/`) holds the wrapper-struct
  definitions. This is six structs and no behaviour, but it is a new
  surface that template authors and migration-PR reviewers must learn.

- The `application.yaml.tmpl` files become non-trivial Go-template logic
  (variant selection via `{{ if .Chart }}…{{ else if .Path }}…{{ end }}`).
  This is an authoring cost the inline-strings approach didn't have; it is
  the price of one file per add-on instead of five files per shape.

- Hard-error on missing keys means a typo in a template field name fails
  the orchestrator phase rather than rendering an empty value. Operators
  pinning a `YAGE_MANIFESTS_REF` ahead of yage will see this as a hard
  bootstrap failure. Mitigated by the changelog-required-min-yage-version
  policy in ADR 0008's risk section.

### Risks

- **Wrapper-struct churn during migration.** Each of #137–#141 will likely
  discover one or two fields the table missed. Mitigation: the migration
  agent extends the relevant struct in the same PR; reviewers check that
  field additions land in `internal/capi/templates/` and not in a local
  ad-hoc struct.

- **Partials directory leakage.** If a future template author drops a
  full-document manifest into `postsync/_partials/`, the leading-underscore
  convention won't catch it at apply time. Mitigation: a CI check in
  `yage-manifests` that asserts every `_partials/*` file lacks
  `apiVersion:` at the document root (template-fragment heuristic).

---

## References

- ADR 0008 — yage-manifests GitOps templates (this addendum amends §1, §2, §3)
- ADR 0010 — In-cluster repo cache (Fetcher MountRoot resolution)
- yage issue #136 — `internal/platform/manifests.Fetcher` implementation
- yage issues #137–#141 — per-package migrations unblocked by this addendum
- yage issue #142 — package retirement after #137–#141
- `internal/capi/helmvalues/`, `internal/capi/wlargocd/`,
  `internal/capi/postsync/`, `internal/capi/caaph/`,
  `internal/capi/cilium/cilium.go` — migration sources
- `internal/csi/driver.go` — `RenderValues` → `Render` interface change (ADR 0008 §6)
