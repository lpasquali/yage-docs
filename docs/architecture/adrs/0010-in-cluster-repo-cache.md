# ADR 0010 — In-Cluster Repository Cache: Zero Workstation Residue

**Status:** Accepted
**Date:** 2026-05-01
**Owners:** Backend (Fetcher redesign, RepoSync Job, PVC lifecycle), Architect (interface contract)

**Supersedes (partial):**
- ADR 0004 §2 steps 1–4 (local `~/.yage/tofu-cache/` fetch) and §2 tofu state path
- ADR 0008 §3 (`manifests.Fetcher` `CacheDir` field and `~/.yage/manifests-cache/`)

---

## Context

ADR 0004 (Phase G) and ADR 0008 (yage-manifests) both design a fetch-and-cache pattern
where yage clones external Git repositories to the operator's workstation at bootstrap time:

- `yage-tofu` → `~/.yage/tofu-cache/`
- Per-provider tofu state → `~/.yage/tofu/<provider>/terraform.tfstate`
- `yage-manifests` → `~/.yage/manifests-cache/`

These paths leave residue on the operator's workstation that persists beyond the kind
cluster's lifetime. An operator who runs `kind delete cluster` expects a clean slate;
the cache dirs remain.

The architectural goal is **zero workstation residue**: when the kind cluster is deleted,
all yage-managed state disappears with it. The four external repositories yage depends on
during a bootstrap run are:

| Repository | Consumer | Already in-cluster? |
|---|---|---|
| `lpasquali/yage-tofu` | `opentofux.JobRunner` (tofu HCL modules) | No — cloned locally |
| `lpasquali/yage-manifests` | `manifests.Fetcher` (YAML templates) | No — cloned locally |
| operator's `workload-app-of-apps` | ArgoCD repo-server | **Yes** — ArgoCD fetches internally |
| operator's `workload-smoketests` | ArgoCD repo-server | **Yes** — ArgoCD fetches internally |

Only the first two require action. The ArgoCD-consumed repos are already in-cluster.

kind's default `StorageClass: standard` (rancher `local-path-provisioner`) stores PVC data
inside the kind node's Docker container filesystem, not on the host. A PVC created in the
`yage-system` namespace is gone the moment `kind delete cluster` runs. This is the correct
primitive.

---

## Decision

### 1. Single shared PVC: `yage-repos`

yage creates one PVC named `yage-repos` in the `yage-system` namespace immediately after
the kind cluster is up, before any repo-dependent phase runs.

```
Name:         yage-repos
Namespace:    yage-system
StorageClass: standard          # kind local-path-provisioner; in-Docker, not on host
Capacity:     500Mi             # covers yage-tofu + yage-manifests with headroom
AccessMode:   ReadWriteOnce     # kind is single-node; RWO is sufficient
```

Mount layout inside Jobs that consume the PVC:

```
/repos/
  yage-tofu/        ← lpasquali/yage-tofu at cfg.TofuRef
  yage-manifests/   ← lpasquali/yage-manifests at cfg.ManifestsRef
```

### 2. `yage-repo-sync` Job

A `batch/v1 Job` named `yage-repo-sync` in `yage-system` runs immediately after the PVC
is created. It has one init container per repo; the main container exits immediately after
the init containers succeed.

```yaml
initContainers:
- name: sync-yage-tofu
  image: alpine/git:2          # or cfg.ImageRegistryMirror/alpine/git:2 for airgap
  env:
  - name: REPO
    value: https://github.com/lpasquali/yage-tofu
  - name: REF
    value: "$(YAGE_TOFU_REF)"
  command:
  - sh
  - -c
  - |
    if [ -d /repos/yage-tofu/.git ]; then
      git -C /repos/yage-tofu fetch --depth=1 origin "$REF" && \
      git -C /repos/yage-tofu checkout FETCH_HEAD
    else
      git clone --depth=1 --branch "$REF" "$REPO" /repos/yage-tofu
    fi
  volumeMounts:
  - { name: repos, mountPath: /repos }

- name: sync-yage-manifests
  image: alpine/git:2
  # identical pattern, REPO=lpasquali/yage-manifests, REF=cfg.ManifestsRef
  ...

containers:
- name: done
  image: busybox:stable
  command: ["true"]

volumes:
- name: repos
  persistentVolumeClaim:
    claimName: yage-repos
```

yage runs the Job synchronously (waits for `Complete`; treats `Failed` as fatal). Log
streaming uses the pod log API — same pattern as `JobRunner` in `opentofux`. The Job is
deleted after it completes (TTL or explicit deletion) to keep `yage-system` clean.

### 3. Updated `opentofux.Fetcher`

The `Fetcher` in `internal/platform/opentofux/fetcher.go` is revised to replace the
local `CacheDir` clone with PVC-based path resolution. The struct no longer needs a
`CacheDir` field; the module path is derived from the known PVC mount at `/repos/yage-tofu/`.

```go
// Fetcher resolves yage-tofu module paths from the in-cluster PVC.
// The PVC must be mounted at /repos/yage-tofu/ in the calling Job pod.
// Use EnsureRepoSync to populate it before calling ModulePath.
type Fetcher struct {
    // MountRoot is the path at which the yage-repos PVC is mounted in the
    // current pod. Defaults to /repos when empty.
    MountRoot string
}

// ModulePath returns the absolute path to the named module directory.
// e.g. ModulePath("proxmox") → "/repos/yage-tofu/proxmox"
func (f *Fetcher) ModulePath(module string) string {
    root := f.MountRoot
    if root == "" {
        root = "/repos"
    }
    return filepath.Join(root, "yage-tofu", module)
}

// EnsureRepoSync creates the yage-repos PVC (if absent) and runs the
// yage-repo-sync Job to clone/fetch both yage-tofu and yage-manifests.
// Must be called once per bootstrap run, after kind is up.
func EnsureRepoSync(ctx context.Context, cli *k8sclient.Client, cfg *config.Config) error
```

The `JobRunner` already mounts the PVC for state (`tofu-state-<module>`). Its `Apply` and
`Destroy` implementations are updated to additionally mount `yage-repos` at `/repos/` so
the HCL source is available at `/repos/yage-tofu/<module>/` alongside the state PVC.

### 4. Updated `manifests.Fetcher`

The `internal/platform/manifests.Fetcher` (ADR 0008 §3) drops `CacheDir`. Template files
are read directly from the PVC mount:

```go
type Fetcher struct {
    // MountRoot is the path at which the yage-repos PVC is mounted.
    // Defaults to /repos when empty.
    MountRoot string
}

// Render reads the .yaml.tmpl at templatePath (relative to yage-manifests root
// on the PVC) and executes it with data via text/template.
// e.g. Render("csi/proxmox/values.yaml.tmpl", cfg)
func (f *Fetcher) Render(templatePath string, data any) (string, error) {
    fullPath := filepath.Join(f.mountRoot(), "yage-manifests", templatePath)
    raw, err := os.ReadFile(fullPath)
    ...
}
```

Callers that run as in-cluster Jobs receive the PVC mount automatically. Callers that run
locally (e.g. `--dry-run` before kind is up) must call `EnsureRepoSync` first or use a
local fixture path (test/dev mode).

### 5. Bootstrap sequence amendment

The `yage-system` namespace setup and PVC/Job creation are inserted immediately after the
kind cluster is ready:

```
kind cluster create
  ↓
EnsureNamespace(yage-system)         [existing]
EnsureNamespace(workload-namespace)  [existing]
CreatePVC(yage-repos)                [NEW — this ADR]
RunJob(yage-repo-sync)               [NEW — this ADR, synchronous, fatal on failure]
  ↓
EnsureIdentity (tofu Jobs mount yage-repos read-only for HCL source)
  ↓
manifests.Fetcher.Render (reads from yage-repos PVC via Job mount)
  ↓
... rest of bootstrap
```

### 6. OpenTofu provider plugin cache

ADR 0004 mentions `~/.terraform.d/plugin-cache` for warming the BPG provider. The
`JobRunner` handles this in-cluster: `tofu init` downloads providers into the Job
container's ephemeral layer. Providers are re-downloaded on each Job run unless a
dedicated `tofu-providers` PVC is added (deferred; not in scope for this ADR).

The `LocalRunner` (pre-kind exception, §7 below) continues to use
`~/.terraform.d/plugin-cache`. This is the only acceptable workstation write path; it is
documented as an exception and must be explicitly cleaned up on `--purge`.

### 7. Exception: phases that run before kind

ADR 0009 §1 places registry VM provisioning before the kind cluster phase. A `JobRunner`
requires a running management cluster; it cannot be used before kind exists.

**Resolution adopted by this ADR:**

Registry and issuing CA provisioning (ADR 0009 Phase H) are rescheduled to run **after**
the kind cluster is up and the `yage-repos` PVC is populated. The kind cluster uses
standard image pulls (internet or pre-seeded), not the yage-managed bootstrap registry,
for the kind phase itself. The registry VM is provisioned after kind, its IP is written to
`cfg.ImageRegistryMirror`, and subsequent CAPI workload cluster provisioning uses it.

This eliminates the pre-kind exception. `LocalRunner` remains in the codebase for:
- Development and test use (running tofu locally without a kind cluster).
- Manual operator invocations outside the normal bootstrap flow.

`LocalRunner.Apply` writes to `os.MkdirTemp` when used in the production pre-kind fallback
path, and the caller is responsible for `os.RemoveAll` after use. This path is explicitly
not the happy path.

### 8. Airgap support

For air-gapped environments:
- Mirror `lpasquali/yage-tofu` and `lpasquali/yage-manifests` to an internal Git server.
- Set `YAGE_TOFU_REF` and `YAGE_MANIFESTS_REF` to `https://internal.git/yage-tofu@<tag>`
  (the Fetcher accepts a `REPO_URL` override via env, analogous to `YAGE_TOFU_REPO` and
  `YAGE_MANIFESTS_REPO` — new config fields to be added alongside this ADR).
- Mirror `alpine/git:2` to the internal registry and set `cfg.ImageRegistryMirror`.

---

## Consequences

### Positive

- **Zero workstation residue.** `kind delete cluster` removes the PVC and all cloned
  content. The operator's `~/.yage/` directory is no longer written to during a normal
  bootstrap run.

- **Single source of truth for repo content.** Both `yage-tofu` Jobs and
  `manifests.Fetcher` render Jobs read from the same `yage-repos` PVC. The sync happens
  once at bootstrap; all subsequent reads are local (no repeated network calls).

- **Consistent with `JobRunner` PVC pattern.** The `tofu-state-<module>` PVC design in
  #123 already uses kind's local-path-provisioner. `yage-repos` follows the same pattern.

- **Registry provisioning resequencing simplifies the overall design.** The pre-kind
  exception is eliminated from the production path, making `JobRunner` the sole production
  execution model.

### Negative

- **`EnsureRepoSync` is a new required bootstrap phase.** A network failure during
  `yage-repo-sync` fails the entire bootstrap. This is the same failure mode as the
  previous local clone, but the failure surface moves from the operator's shell to a
  pod log.

- **`manifests.Fetcher.Render` requires a running cluster.** `--dry-run` before kind is
  up cannot render actual templates. Mitigation: `--dry-run` in `plan.go` uses local
  stubs or skips the Render calls entirely (same approach as other cluster-dependent
  phases in dry-run mode).

- **`alpine/git` image dependency.** The `yage-repo-sync` Job requires a container image.
  This is pinned by digest in production to prevent supply-chain drift.

### Risks

- **PVC ReadWriteOnce on multi-node kind.** kind is always single-node, so RWO is
  sufficient. If multi-node kind support is added in future, the PVC must change to RWX or
  each node must get its own PVC. This is explicitly deferred.

- **PVC capacity.** 500Mi is an estimate. If `yage-tofu` or `yage-manifests` grow
  significantly, the PVC capacity request must be updated. The `yage-repos` PVC creation
  code should accept a `YAGE_REPOS_PVC_SIZE` override (default `500Mi`).

---

## References

- ADR 0004 — Universal OpenTofu Identity Bootstrap (Phase G); §2 steps 1–4 and state path superseded by this ADR
- ADR 0008 — yage-manifests GitOps Template Repository; §3 `CacheDir` and `~/.yage/manifests-cache/` superseded by this ADR
- ADR 0009 — On-prem platform services; §1 registry provisioning sequence amended by §7 of this ADR
- Issue #123 — `opentofux.JobRunner` PVC design (`tofu-state-<module>`) — compatible with `yage-repos` PVC
- Issue #136 — `manifests.Fetcher` implementation (ADR 0008 step 2) — must follow this ADR's interface, not ADR 0008 §3
- Issue #125 — registry VM provisioning — must follow the post-kind sequencing decision in §7
