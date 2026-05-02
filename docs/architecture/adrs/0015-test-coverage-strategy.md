# ADR 0015 — Test Coverage Strategy and Targets

**Status:** Proposed
**Date:** 2026-05-01
**Author:** Architect agent
**Relates to:** ADR 0013 (TUI Secret-Display Policy), ADR 0014 (xapiri Dashboard Split), yage issue #181 (go test missing from CI — closed), yage issue #191 (teatest integration suite), yage issue #175 (pre-existing xapiri failures)

---

## Context

Three UI regressions and a JobRunner bug shipped this session despite "CI green."
The root causes were structural:

1. `quality-gates.yml` did not run `go test` until PR #182 added it to the
   `RuneGate/Quality/Go` job.  Every unit test added before that PR ran only on
   developer workstations — CI passed on build + vet alone.

2. xapiri had no integration-level tests.  Two tests added to enforce ADR 0013 are
   currently failing (`TestRenderCostsCredsForm_SecretFieldsNeverLeakValue` and
   `TestRenderTokenPromptOverlay_NeverLeaksWhenUnfocused`), demonstrating that the
   test suite is already load-bearing for correctness invariants.  Both failures are
   tracked in yage issue #175.

3. Coverage is wildly uneven across the codebase (3.6% in `internal/pricing/` to
   100% in `internal/csi/` and `internal/platform/manifests/`) with no enforcement
   mechanism preventing regression in the already-covered packages.

Test coverage is now load-bearing.  This ADR establishes a tier-based coverage
strategy with realistic CI ramp targets and 100% as the aspirational ceiling.

---

## Coverage Audit (2026-05-01)

The following table was produced by running `go test -coverprofile` against every
package that has at least one test file.  Packages with `src=N, tests=0` are listed
separately.

### Packages with existing tests

| Package | Coverage | Tier |
|---|---|---|
| `internal/csi` (registry) | 100.0% | A |
| `internal/platform/manifests` | 100.0% | A |
| `internal/util/yamlx` | 100.0% | A |
| `internal/platform/airgap` | 97.5% | A |
| `internal/cluster/capacity` | 87.0% | A |
| `internal/config` | 85.4% | A |
| `internal/ui/plan` | 73.7% | A |
| `internal/csi/openebs` | 73.1% | B |
| `internal/csi/rookceph` | 70.4% | B |
| `internal/csi/longhorn` | 66.7% | B |
| `internal/feasibility` | 62.8% | B |
| `internal/capi/helmvalues` | 58.3% | A |
| `internal/csi/cindercsi` | 58.0% | B |
| `internal/obs` | 57.1% | B |
| `internal/operator/cost` | 55.6% | B |
| `internal/csi/azuredisk` | 54.8% | B |
| `internal/csi/ociblock` | 54.5% | B |
| `internal/csi/doblock` | 51.0% | B |
| `internal/csi/linodebs` | 50.0% | B |
| `internal/provider` (interface + registry) | 49.3% | A |
| `internal/csi/gcppd` | 48.7% | B |
| `internal/csi/hcloud` | 47.7% | B |
| `internal/capi/cilium` | 43.4% | A |
| `internal/csi/vspherecsi` | 59.7% | B |
| `internal/ui/promptx` | 35.0% | A |
| `internal/platform/secmem` | 34.5% | A |
| `internal/provider/openstack` | 27.7% | B |
| `internal/platform/keyring` | 25.0% | A |
| `internal/ui/xapiri` | 24.6% | C |
| `internal/provider/vsphere` | 24.7% | B |
| `internal/platform/opentofux` | 23.5% | B |
| `internal/cluster/kindsync` | 19.8% | B |
| `internal/platform/shell` | 19.4% | A |
| `internal/provider/linode` | 19.4% | B |
| `internal/csi/awsebs` | 17.1% | B |
| `internal/capi/caaph` | 17.6% | A |
| `internal/platform/k8sclient` | 12.2% | B |
| `internal/provider/proxmox` | 11.7% | B |
| `internal/cost` | 13.5% | B |
| `internal/orchestrator` | 8.2% | B |
| `internal/capi/pivot` | 5.4% | B |
| `internal/cluster/kind` | 3.9% | B |
| `internal/pricing` | 3.6% | B |
| `internal/csi/ibmvpcblock` | 38.3% | B |
| `internal/capi/templates` | [no statements] | D |

### Packages with zero tests (0%)

All of these have at least one source file and zero `*_test.go` files.

| Package | Source files | Tier |
|---|---|---|
| `internal/capi/argocd` | 1 | A |
| `internal/capi/manifest` | 3 | A |
| `internal/capi/postsync` | 1 | A |
| `internal/capi/wlargocd` | 1 | A |
| `internal/csi/proxmoxcsi` | 1 | B |
| `internal/platform/installer` | 2 | D |
| `internal/platform/kubectl` | 1 | B |
| `internal/platform/sysinfo` | 3 | A |
| `internal/provider/aws` | 5 | B |
| `internal/provider/azure` | 3 | B |
| `internal/provider/capd` | 1 | B |
| `internal/provider/digitalocean` | 3 | B |
| `internal/provider/gcp` | 3 | B |
| `internal/provider/hetzner` | 4 | B |
| `internal/provider/ibmcloud` | 3 | B |
| `internal/provider/oci` | 3 | B |
| `internal/provider/proxmox/api` | 2 | B |
| `internal/ui/cli` | 4 | A |
| `internal/ui/logx` | 1 | A |
| `internal/util/idgen` | 1 | A |
| `internal/util/versionx` | 1 | A |
| `api/v1alpha1` | 1 | D |
| `cmd/yage` | 1 | D |
| `cmd/yage-operator` | 1 | D |

---

## Decision

### Tier definitions

Coverage targets are per-package, not per-repo.  A repo-wide aggregate masks
Tier A regressions behind Tier C improvements.

| Tier | Target | Rationale |
|---|---|---|
| **A — Pure logic** | 100% (aspirational), CI gate ≥90% | Pure renderers, YAML generators, config parsers, utility packages, provider stubs.  These have no I/O, no OS calls, and no network calls; they are fully testable with table-driven and golden tests. |
| **B — Integration boundaries** | ≥80% | Packages that cross a system boundary (kubectl shell-out, k8s API, OpenTofu runner, pricing HTTP, kind cluster lifecycle).  Boundary crossings are mocked at the interface; non-mocked paths are acknowledged gaps. |
| **C — TUI surface** | ≥50% | `internal/ui/xapiri`.  Drive with `teatest` synthetic key sequences + golden frame snapshots.  Some Lipgloss render branches (terminal resize, palette variants) are legitimately hard to hit. |
| **D — Exempt** | N/A | Code that cannot be meaningfully unit-tested: `cmd/yage/main.go`, `cmd/yage-operator`, generated code (`*_generated.go`), `internal/capi/templates` (text/template data-only structs, validated by render tests in the caller package), `internal/platform/installer` (external binary downloads). |

### Tier A packages

`internal/capi/caaph`, `internal/capi/argocd`, `internal/capi/manifest`,
`internal/capi/postsync`, `internal/capi/wlargocd`, `internal/capi/helmvalues`,
`internal/capi/cilium`, `internal/config`, `internal/platform/manifests`,
`internal/platform/sysinfo`, `internal/platform/secmem`, `internal/platform/airgap`,
`internal/platform/shell`, `internal/platform/keyring`, `internal/provider` (interface),
`internal/cluster/capacity`, `internal/ui/cli`, `internal/ui/logx`, `internal/ui/plan`,
`internal/ui/promptx`, `internal/util/idgen`, `internal/util/versionx`,
`internal/util/yamlx`, `internal/csi` (registry).

### Tier B packages

`internal/orchestrator`, `internal/cluster/kind`, `internal/cluster/kindsync`,
`internal/platform/opentofux`, `internal/platform/kubectl`, `internal/platform/k8sclient`,
`internal/capi/pivot`, `internal/cost`, `internal/pricing`, `internal/feasibility`,
`internal/obs`, `internal/operator/cost`, all CSI driver packages (`awsebs`, `azuredisk`,
`cindercsi`, `doblock`, `gcppd`, `hcloud`, `ibmvpcblock`, `linodebs`, `longhorn`,
`ociblock`, `openebs`, `proxmoxcsi`, `rookceph`, `vspherecsi`), all provider packages
(`aws`, `azure`, `capd`, `digitalocean`, `gcp`, `hetzner`, `ibmcloud`, `linode`, `oci`,
`openstack`, `proxmox`, `proxmox/api`, `vsphere`).

### Tier C packages

`internal/ui/xapiri`.

---

## Top-5 coverage gaps per tier

### Tier A — top priorities (backend agent)

1. `internal/capi/postsync` — 0%, 1 source file.  PostSync-hook YAML renderers are
   pure string generation.  Table-driven tests with golden fixtures close this completely.
2. `internal/capi/wlargocd` — 0%, 1 source file.  Workload Argo Application renderers;
   same pattern as `postsync`.
3. `internal/capi/argocd` — 0%, 1 source file.  ArgoCD helpers (admin password,
   kubeconfig discovery).  The k8s-API calls can be mocked with `httptest`.
4. `internal/capi/manifest` — 0%, 3 source files.  YAML-patch workload manifests.
   These are the highest-risk pure-logic paths given the CAPI v1beta2 strict-decoding
   gotcha; table-driven tests with multi-doc YAML fixtures are required.
5. `internal/capi/caaph` — 17.6%.  CAAPH HelmChartProxy rendering; already has a test
   file; significant render branches remain uncovered.

### Tier B — top priorities (backend agent + platform-engineer agent)

1. `internal/pricing` — 3.6%, 5+ source files.  Live fetchers can be mocked behind
   `httptest`; the existing `defaulttransport_airgap_test.go` is the correct pattern.
2. `internal/capi/pivot` — 5.4%.  clusterctl move path; mock `shell.Capture` at the
   `shell` boundary.  High correctness risk.
3. `internal/cluster/kind` — 3.9%.  Kind lifecycle; mocking the kind CLI at the
   `shell` boundary enables most paths.
4. `internal/orchestrator` — 8.2%.  Top-level bootstrap driver.  `plan_golden_test.go`
   and `plan_snapshot_test.go` exist but cover only the plan path, not the execute path.
   Introducing a `--dry-run` coverage harness with a CAPD-backed integration test
   (yage issue #119) is the recommended approach rather than trying to unit-test
   `bootstrap.Run` directly.
5. `internal/platform/k8sclient` — 12.2%.  Client wrapper; `httptest`-based API server
   mocking closes most paths.

### Tier C — top priorities (frontend agent)

The current xapiri coverage is 24.9% (or an effective lower figure once the
`TestRenderCostsCredsForm_SecretFieldsNeverLeakValue` failure is considered).
Per ADR 0014, `dashboard.go` is being split into 18 files.  The following per-file
test classes are required:

1. `dashboard_fields.go` — `TestRenderField_*` suite: secret fields never leak value
   (ADR 0013 canary), all field kinds render correctly for all states.
2. `dashboard_focus.go` — `TestMoveFocus_*`, `TestIsHidden_*`: provider-mode switching
   shows/hides the correct fields; visibility invariants do not break across tabs.
3. `dashboard_snapshot.go` — `TestBuildSnapshotCfg_*`, `TestFlushToCfg_*`:
   round-trip fidelity for all config fields.
4. `tab_config.go`, `tab_provision.go` — `TestArrowNav_*` suite: advance/retreat focus
   with arrow keys does not cross tab boundaries.
5. `dashboard_overlays.go` — `TestTokenOverlay_*`: overlay captures arrow keys before
   the per-tab handler; dismissal restores normal routing (regression for the in-flight
   arrow-key fix).

### Tier D — no tests required

`cmd/yage`, `cmd/yage-operator`, `internal/capi/templates`, `internal/platform/installer`.
Any `*_generated.go` file.

---

## #191 — teatest UI integration suite (scope definition)

Issue #191 in the yage repository is queued but unscoped.  This ADR defines its scope.

The teatest suite must provide synthetic-input coverage for every interactive surface
in xapiri.  Using `github.com/charmbracelet/x/exp/teatest`, the suite must exercise:

**Per-tab actions (one `TestXxx_Teatest` class per ADR 0014 file):**

| File | Required coverage |
|---|---|
| `tab_config.go` | Load config list; create new profile; navigate entries |
| `tab_provision.go` | Enter all config fields; save; cancel; field validation errors |
| `tab_editor.go` | Open kind Secret in editor; apply change; cancel; error path |
| `tab_costs.go` | Enter cost credentials; save; cycle timeframe presets; `[`/`]` keys |
| `tab_logs.go` | Log ring renders; scroll; clear |
| `tab_deploy.go` | Deploy button triggers deploy command; exit does not leak on on-prem mode |
| `tab_deps.go` | Deps list renders; install action; all-green state |
| `tab_help.go` | Help text renders without panic |
| `tab_about.go` | About text renders without panic |

**Cross-cutting surfaces:**

| Surface | Required coverage |
|---|---|
| `dashboard_overlays.go` | Token-prompt overlay: show → type → dismiss; arrow keys not consumed by overlay when overlay is inactive |
| `dashboard_focus.go` | Tab-switch (Ctrl+Left/Right); provider-mode change triggers field hide/show |
| `dashboard_chrome.go` | Tab bar renders active tab highlight; footer renders key hints |
| `dashboard_term.go` | PTY pane renders; terminal input is passed through when terminal pane is focused |

**Golden frame snapshots** must be committed for the initial render of each tab so
regressions in Lipgloss output are caught by CI without manual review.

The teatest suite lives in `internal/ui/xapiri/` alongside `xapiri_test.go`.  It must
use the `//go:build integration` build tag so it does not run on every `go test ./...`
invocation (PTY and terminal init add ~2s per test); a dedicated `make test-integration`
target invokes it.

The `TestRenderCostsCredsForm_SecretFieldsNeverLeakValue` test failure present as of
2026-05-01 demonstrates that the test suite is already catching real regressions.
That failure must be resolved before the #191 PR opens (it is a pre-existing condition,
not a coverage-ADR concern).  Both failures are tracked in yage issue #175.

---

## Enforcement mechanism

### Per-package coverage gate script

A new `scripts/check-coverage.go` (invoked as `go run scripts/check-coverage.go`)
reads a `coverprofile` output file and a `scripts/coverage-tiers.json` configuration
that maps each package path to its tier and minimum threshold.  The script:

1. Parses `go tool cover -func` output per package.
2. Compares each package's coverage against its tier threshold.
3. Prints a table of PASS/FAIL per package.
4. Exits non-zero if any package fails its threshold.

The `coverage-tiers.json` is committed and updated when packages move tiers or
when new packages are added.  Adding a new package without an entry in
`coverage-tiers.json` is a CI error (fail-open: new packages default to their
tier's minimum threshold with a TODO comment).

### New `coverage` job in `quality-gates.yml`

```
coverage:
  name: RuneGate/Quality/Coverage
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout
    - uses: actions/setup-go (go-version-file: go.mod)
    - name: generate coverage profile
      run: go test -coverprofile=cover.out -covermode=atomic ./...
    - name: check per-package thresholds
      run: go run scripts/check-coverage.go cover.out
```

The `merge-gate` job's `needs` array is extended to include `coverage`.  This makes
coverage failures block merge without requiring a repository ruleset change.

### PR-level delta gate ("coverage cannot decrease")

Within the same `coverage` job, after the threshold check, the script compares the
new coverage figures against a baseline stored in `scripts/coverage-baseline.json`.
The baseline is committed to `main` and updated automatically by a post-merge workflow
step.  The delta check fails if any package's coverage decreases by more than 1
percentage point compared to the baseline.

This prevents the common "add code without tests" pattern from hiding behind a
package that was already above its tier threshold.

The 1 pp tolerance accommodates floating-point rounding in `go tool cover` output.
A zero-tolerance threshold would produce spurious failures on inconsequential
refactors.

---

## Milestone timeline

| Milestone | Target date | Criteria |
|---|---|---|
| **M1 — Gate in place** | ADR merge + 7 days | `coverage` job added to `quality-gates.yml`; `check-coverage.go` script implemented; `coverage-tiers.json` committed with current coverage values as baselines (no package can regress); `coverage-baseline.json` committed |
| **M2 — Tier A ≥90%** | ADR merge + 30 days | Every Tier A package passes the ≥90% threshold; the 7 currently-zero Tier A packages have initial test suites |
| **M3 — Tier A 100%, Tier B ≥80%** | ADR merge + 90 days | All Tier A packages at 100%; every Tier B package passes ≥80%; `make test-integration` passes for the first teatest classes |
| **M4 — Tier C ≥50%, teatest suite complete** | ADR merge + 180 days | xapiri at ≥50%; all per-tab and cross-cutting teatest classes from the #191 scope table above are committed; golden frames committed |

---

## Agent assignments

| Work item | Assigned agent |
|---|---|
| `scripts/check-coverage.go` + `coverage-tiers.json` + `coverage-baseline.json` + `quality-gates.yml` update | yage-backend |
| Tier A tests: `capi/postsync`, `capi/wlargocd`, `capi/manifest`, `capi/argocd`, `capi/caaph` | yage-backend |
| Tier A tests: `ui/cli`, `ui/logx`, `ui/promptx`, `util/idgen`, `util/versionx` | yage-backend |
| Tier B tests: `pricing`, `capi/pivot`, `cluster/kind`, `orchestrator` CAPD integration (#119) | yage-backend |
| Tier B tests: `platform/kubectl`, `platform/k8sclient`, `platform/opentofux` | yage-backend |
| Tier C / #191: teatest suite for all xapiri tabs + cross-cutting surfaces | yage-frontend |
| Tier B: provider stub packages (aws, azure, capd, gcp, do, hetzner, ibmcloud, oci) | yage-backend |
| Tier B: CSI driver packages below 50% (awsebs, gcppd, hcloud, ibmvpcblock) | yage-backend |

---

## Consequences

**Positive:**
- Coverage failures block merge through the existing `merge-gate` aggregator.
- Per-package thresholds prevent any single package from regressing without the
  author noticing before opening a PR.
- The delta gate surfaces the "add code without tests" anti-pattern on every PR.
- The teatest scope definition in §"#191" unblocks that issue from "queued but unscoped"
  to "ready to execute"; the frontend agent can link this ADR in the #191 issue body
  rather than re-deriving the scope.
- xapiri's per-tab file structure (ADR 0014) is the natural test boundary: one test
  class per `tab_*.go` file, one test class per shared-concern file.

**Negative / risks:**
- `scripts/check-coverage.go` is new infrastructure that must itself be reviewed and
  maintained.  If the script has a bug, it could produce false positives (failing PRs
  that should pass) or false negatives (passing PRs that should fail).
- The 30-day M2 milestone requires the backend agent to write tests for 7 currently-zero
  Tier A packages in addition to ongoing feature work.  This is a non-trivial load.
- `internal/orchestrator` at 8.2% is classified Tier B rather than Tier A because the
  `bootstrap.Run` driver orchestrates side-effecting phases that are not meaningfully
  unit-testable without a real cluster.  The recommended path is the CAPD-backed
  integration test (#119), not attempting to mock every phase.  If #119 slips beyond M3,
  `orchestrator` remains the single largest gap in the Tier B gate.
- Two `xapiri` failures (`TestRenderCostsCredsForm_SecretFieldsNeverLeakValue` and
  `TestRenderTokenPromptOverlay_NeverLeaksWhenUnfocused`, both present as of 2026-05-01)
  mean `xapiri` currently fails CI.  M1 cannot close until these pre-existing failures
  are resolved (tracked as yage issue #175).

---

## Acceptance criteria for this ADR

A PR implementing the enforcement mechanism (M1) is **Done (Level 2)** when:

- [ ] **`make test` passes** with no new failures (pre-existing `xapiri` failure
      resolved per #175 before M1 closes).
- [ ] **`scripts/check-coverage.go`** is committed and passes `go vet`.
- [ ] **`coverage-tiers.json`** contains an entry for every package currently
      returned by `go test ./...` (no package silently missing).
- [ ] **`coverage-baseline.json`** reflects the 2026-05-01 audit values; the delta
      gate does not fire on the M1 PR itself.
- [ ] **`quality-gates.yml`** has a `coverage` job; `merge-gate` depends on it.
- [ ] **No Tier A package regresses** below its pre-M1 coverage after M1 merges.
- [ ] **PR body cites this ADR** and links issue #175 and #191.
