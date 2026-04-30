# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is largely complete: phases A, B, C, and E are done; D is substantially complete. Legacy cleanup is complete per ADR 0002 (no-backward-compatibility policy). ADR 0007 adopted: dashboard is the new default xapiri entry point.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-04-30** — PO audit: PRs #117 + #116 merged (xapiri dashboard-only + vsphere-csi); CSI wave: 5 of 10 drivers merged (#107 hcloud, #108 openebs, #109 rook-ceph, #116 vsphere + earlier #88); 6 CSI PRs still in-flight (#110–#115); new issues #118–#121; yage-docs PR #5 open

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #117 (xapiri dashboard-only), #116 (vsphere-csi), #109 (rook-ceph), #108 (openebs), #107 (hcloud-csi) | Active development |
| yage-docs | `main` | ADRs 0001–0007 written; WORKFLOW added | Documentation in progress |

## Recent Changes

### 2026-04-30 — PO audit: xapiri #117 + vsphere-csi #116 merged; CSI wave ongoing; new issues #118–#121

**Frontend (ADR 0007 — xapiri dashboard-only):**
- **PR #117 merged** (`feat/xapiri-dashboard-only`) — closes #103 (dashboard default) and #105 (remove legacy walkthrough). Both issues auto-closed by GitHub.
- #104 (epic: ADR 0007) remains open as parent.

**Backend (CSI drivers — ADR 0001 epic #77):**
- **PR #116 merged** (vsphere-csi) — closes #91 (auto-closed).
- **PR #109 merged** (rook-ceph) — closes #92 (auto-closed).
- **PR #108 merged** (openebs) — closes #93 (auto-closed).
- **PR #107 merged** (hcloud-csi) — closes #84 (auto-closed).
- **6 CSI PRs still open and in-flight**: #110 (oci), #111 (do), #112 (longhorn), #113 (linode), #114 (ibm-vpc), #115 (openstack-cinder INI fix). Being merged by programmer agent; #110 has conflicts (DIRTY), others show UNKNOWN state (likely mid-merge).

**New issues created:**
- **#118** — D1: wire `csi.Selector` into orchestrator; delete `internal/capi/csi/` (p1, Backend, unblocked)
- **#119** — D4: CAPD smoke E2E test for bootstrap pipeline (p3, backlog)
- **#120** — Epic: on-prem platform services — bootstrap registry + online issuing CA via OpenTofu (airgap path)
- **#121** — ADR 0009: Phase H on-prem platform services: registry + issuing CA via OpenTofu (p2, Architect)

**Architect (#81 — ADR 0004 Phase G):**
- **yage-docs PR #5** open: https://github.com/lpasquali/yage-docs/pull/5 — accepts ADR 0004 universal OpenTofu identity bootstrap. Branch: `docs/81-adr-0004-phase-g`. Issue #81 assigned to lpasquali.

---

### 2026-04-30 — PO session: ADR 0007 epic opened; agent assignments issued

**New issues opened (ADR 0007 — xapiri dashboard as default entry):**
- **#104** — epic: ADR 0007 — xapiri dashboard as default entry; deprecate and remove legacy walkthrough
- **#103** — Phase 1 (Frontend, p1): make dashboard the default entry point; skip pre-dashboard config selection prompt. Key change: `internal/ui/xapiri/xapiri.go:useHuhTUI()` defaults to true when env var unset; `YAGE_XAPIRI_TUI=legacy` opts back for one sprint.
- **#105** — Phase 2 (Frontend, p3): remove legacy bufio walkthrough entirely. **Blocked**: must not start until one sprint after #103 merges.

**Agent assignments (this session):**
- **Frontend**: #103 (p1 — start now)
- **Backend**: #68 (p1 — integrate worktrees A/B/D1/D4), then #71, then #84–#93
- **Architect**: #81 (p2 — flesh out ADR 0004 OpenTofu Phase G)
- **Platform Engineer**: no new refinements pending; standby

**GitHub hygiene:** #68 assigned to lpasquali; #103–#105 added to project board.

---

### 2026-04-30 — PRs #72–#76 merged; issues #66, #67, #69, #70, #82, #83 closed

**Five PRs merged in sequence:**
- **#72** — `provider/vsphere`: EnsureGroup + Inventory via govmomi; Proxmox cleanup (ADR 0002 items 2+6). Closes #67.
- **#74** — `refactor(config,pricing)`: remove `OS_*`/`YAGE_CURRENCY` env aliases + legacy snapshot JSON (ADR 0002 items 3–5).
- **#76** — `feat(xapiri)`: TUI keymap normalization + dispatch architecture fixes (ADR 0003). Closes #69.
- **#75** — `provider/openstack`: PatchManifest flavor resolution + Inventory via gophercloud; fix flavor env keys. Closes #66.
- **#73** — `refactor(kindsync)`: remove `SyncProxmoxBootstrapLiteralCredentials` (ADR 0002 item 1). Closes #70 (partial).

**ADRs 0003–0007** written and accepted in `yage-docs`.

**Issues closed:** #66, #67, #69, #70, #82, #83.

---

### 2026-04-30 — Issue #69: xapiri TUI keymap normalization + dispatch fixes (ADR 0003)

**Frontend agent** completed on `feat/xapiri-tui-dispatch-fixes` (commit `e1b198d`):
- Removed `j/k` navigation from `updateCfgListScreen`, `updateCfgEditScreen`, `updateLogsTab`, `updateCostsTab`; up/down arrows only
- Removed `Tab`/`ShiftTab` as field-nav from `updateCfgEditScreen`; arrows only
- Promoted `inTextField` inline expression to `(m dashModel) inTextField() bool` method; added `tabCosts+costCredsMode` case
- Added `preserveTransientState(old, next dashModel) dashModel` helper; replaces manual copy in `cfgEntryLoadMsg` handler
- `initFromConfig` in `state.go` now calls `detectFork(cfg)` when `cfg.InfraProvider == ""`
- `renderCostsTab`: inline dim note when `CostCompareEnabled && !GeoIPEnabled && DataCenterLocation == ""`
- Help tab: updated j/k refs to show arrows; added `[ / ]` timeframe keys for Costs tab
- `bootstrap_config.go`: `WithKeyMap` override on all huh Select forms to remove j/k navigation
- All tests pass; `go vet` clean

---

### 2026-04-30 — Agent completions: vSphere (#67), OpenStack (#66); ADR 0003 written

**vSphere agent** completed on `feat/xapiri-config-provision-split` (commit `761f900`):
- New: `internal/provider/vsphere/connect.go` (shared `vsphereConnect` with TLS thumbprint pinning)
- New: `internal/provider/vsphere/ops.go` (`EnsureGroup` folder creation via govmomi, `Inventory` via ResourcePool)
- Modified: `internal/provider/vsphere/vsphere.go` (removed ErrNotApplicable stubs for EnsureGroup/Inventory)
- Added: `github.com/vmware/govmomi v0.53.1` to `go.mod`
- Note: `Inventory` returns `Cores=0` with CPU expressed in MHz (cannot derive cores from ResourcePool alone)

**OpenStack agent** completed on `worktree-agent-afd01feec4a66c948` (commit `97b5288`):
- New: `internal/provider/openstack/inventory.go` (`PatchManifest` flavor resolution via Nova, `Inventory` quota via Nova+Cinder)
- Fixed: `state.go` stale env keys (`OPENSTACK_WORKER_FLAVOR` → `OPENSTACK_NODE_MACHINE_FLAVOR`, `OPENSTACK_CONTROL_PLANE_FLAVOR` → `OPENSTACK_CONTROL_PLANE_MACHINE_FLAVOR`)
- Added: `github.com/gophercloud/gophercloud/v2 v2.12.0` to `go.mod`
- Added: smoke test `TestTemplateVarsKeysMatchTemplate` guarding the env key correctness

---

### 2026-04-30 — Architect assessment: phase status corrected; ADR 0002 no-compat policy

Architect audited the codebase and produced §25 addendum to `abstraction-plan.md`:

- **Phase status corrected**: CURRENT_STATE previously showed A/B/D/E as "Not started"; ground truth is A/B/E complete, D substantially complete (see Abstraction Plan Status table below).
- **No-backward-compatibility policy adopted** (ADR 0002): yage has no production users; all legacy fallbacks (dual-read/write migration, env-var aliases, Secret-name fallbacks, JSON-format fallbacks) are dead weight and will be deleted without deprecation cycle.
- **Three ADR documents written**: §25 addendum to `abstraction-plan.md`; ADR 0001 (CSI driver registry); ADR 0002 (backward compat removal).

---

### 2026-04-30 — xapiri: config/provision tab split (merged to main)

Split the previous single "Config" tab in xapiri into two separate tabs:
- **Config** (tab 1) — profile selection list and new-config name input
- **Provision** (tab 2) — full interactive edit form for the selected config

Selecting a config from the list automatically switches to the provision tab. All other tabs (3–9) remain gated on `cfgSelected`. Keyboard shortcuts 1–8, ctrl+alt+1–8, and mouse click-to-focus remapped to new indices. Help tab updated.

---

### 2026-04-29 — Config system revamp: per-ns layout, fork restore, dual-arch image check (#65)

Merged into `main`. Per-namespace config layout; fork restore logic; dual-arch image presence check at bootstrap.

---

### 2026-04-28 — xapiri + kindsync + pricing: per-config bootstrap Secrets, region-ranked costs, installer TUI API (#64)

Per-config bootstrap Secrets tied to `cfg.ConfigName`; region-ranked cost display in xapiri; installer toolchain exposed via TUI API.

---

### 2026-04-27 — xapiri: mode-filtered provider list + mgmt+workload network config (#63)

Provider list filtered by bootstrap mode; management and workload network config integrated into xapiri walk-through.

---

### 2026-04-26 — xapiri: streaming costs, full config expansion, mouse & log UX (#62)

Live cost streaming in xapiri; full config tree expansion; mouse support and improved log pane UX.

---

### Earlier xapiri milestones

| PR | Summary |
|---|---|
| #61 | Fix: editor never opens + wrong fallback editor + remove deploy cost preview |
| #60 | Fix: resolve editor via VISUAL→EDITOR→probe chain with OS-specific fallbacks |
| #59 | Embedded PTY terminal pane + UX navigation fixes |
| #58 | Phase-F: operator Secret config, CRD types, deploy tab, credential hardening |
| #54 | Phase-F spike: yage-operator cost runner + Prometheus metrics |
| #51 | Overcommit toggle in dashboard + capacity prompt |
| #50 | provider/ibmcloud: K3sTemplate, KindSyncFields, TemplateVars |
| #47 | xapiri: pivot/mgmt-cluster settings step (on-prem fork) |

---

## Abstraction Plan Status

Tracked in [yage-docs ADR](https://lpasquali.github.io/yage-docs/architecture/adrs/abstraction-plan/).

| Phase | Goal | Status |
|---|---|---|
| C | Config namespacing (`cfg.Proxmox*` → `cfg.Providers.Proxmox.*`) | **Merged** |
| A | Inventory behind `Provider.Inventory()` | **Complete** (interface + call sites wired; Proxmox implements; cleanup in progress) |
| B | Plan body delegation per provider | **Complete** (DescribeIdentity/DescribeWorkload/DescribePivot wired; Proxmox implements) |
| D | Generic kindsync + Purge | **Substantially complete** (KindSyncFields + WriteBootstrapConfigSecret in use; legacy wrappers removed per ADR 0002) |
| E | Pivot generalization (kind → any-cloud mgmt) | **Complete** (PivotTarget called via interface in pivot.go) |

---

## Active Work

| Issue | Branch | Agent | Description | Status |
|---|---|---|---|---|
| #84–#93 | Various worktree-agent-* | Backend | 10 CSI driver PRs (ADR 0001 epic #77) | Merging (#107–#116); 5 merged, 6 in-flight |
| #81 | docs/81-adr-0004-phase-g (yage-docs) | Architect | ADR 0004: Phase G universal OpenTofu identity | **PR #5 open** — https://github.com/lpasquali/yage-docs/pull/5 |
| #118 | TBD | Backend | D1: wire csi.Selector into orchestrator; delete internal/capi/csi/ | **Assigned** — unblocked |
| #121 | TBD | Architect | ADR 0009: Phase H on-prem platform services (registry + issuing CA) | **Assigned** |
| #120 | — | — | Epic: on-prem platform services via OpenTofu (airgap path) | Open |
| #104 | — | — | Epic: ADR 0007 (parent of #103, #105 — both now closed) | Open |
| #119 | — | Frontend | D4: CAPD smoke E2E test for bootstrap pipeline | Backlog |

---

## Known Issues

- xapiri is still a work-in-progress TUI; not all provider paths are fully wired.
- Cost estimation requires live Proxmox API; returns `ErrUnavailable` when unreachable.
- vSphere `Inventory()`: `Cores=0` — CPU expressed in MHz only (cannot derive cores from ResourcePool quota alone without host-speed query).
- **D1**: Orchestrator still calls `capi/csi.ApplyConfigSecretToWorkload` in `bootstrap.go` and `workloadapps.go`; `internal/capi/csi/` should be deleted per ADR 0001 once D1 (#118) merges.

## Next Steps

### Immediate (current sprint)

1. **Let programmer merge remaining CSI PRs #110/#111/#112/#113/#114/#115** (Backend) — each is one clean commit; rebase #110 (oci, has conflicts) first.
2. **Merge yage-docs PR #5** (Architect) — ADR 0004 + CURRENT_STATE on `docs/81-adr-0004-phase-g`.
3. **Start #118** (p1, Backend) — D1: wire `csi.Selector` into orchestrator; delete `internal/capi/csi/`. Already unblocked (openstack-cinder in main via #88).
4. **Start #121** (p2, Architect) — ADR 0009 Phase H: on-prem platform services spec.

### Planned (next sprint)

5. **Issue #71** (p2, Backend) — ADR 0002 item 7: remove redundant `cfg.InfraProvider == "proxmox"` guards.
6. **Issue #80** (p3, Backend) — OpenStack EnsureIdentity: template clouds.yaml. **Blocked** on ADR 0004 acceptance (yage-docs PR #5).
7. **Issue #79** (p3, Backend) — vSphere PatchManifest sizing fields.

### Backlog

8. **Issue #119** (p3, Backend) — D4: CAPD smoke E2E test for bootstrap pipeline.
9. **Issues #94–#101** (p3, Backend) — 8 PlanDescriber provider implementations (epic #78).

---

### 2026-04-30 — Platform Engineer issue refinement

Three open issues refined with K8s/infra-level implementation specs:

- **#66** (OpenStack PatchManifest + Inventory): Flagged pre-existing bug — `OPENSTACK_NODE_MACHINE_FLAVOR` vs `OPENSTACK_WORKER_FLAVOR` env key mismatch in TemplateVars vs K3s template. gophercloud v2 not yet in go.mod — must be added. Nova `limits.Get` + Cinder quotasets API paths specified.
- **#67** (vSphere Inventory + EnsureScope): govmomi not yet in go.mod — must be added. ResourcePool property query path specified. Unlimited pool (`-1`) edge case: return `ErrNotApplicable`. TLS thumbprint field check required.
- **#68** (Background agent integration): D1 depends on B. Ordering constraint: proxmox-csi `EnsureSecret` must run before HelmChartProxy fires. Post-D1: delete `internal/capi/csi/` per ADR 0001.

**Handoff to Backend:** #66 and #67 are resolved (merged in #75, #72). #68 is integration ops — check worktree/branch status of A/B/D1/D4 before starting.
