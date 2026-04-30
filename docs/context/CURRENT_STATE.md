# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is largely complete: phases A, B, C, and E are done; D is substantially complete. Legacy cleanup is in flight per ADR 0002 (no-backward-compatibility policy).

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-04-30** — PRs #72, #73, #74, #75, #76 all merged; issues #66, #70, #69, #67, #82, #83 closed

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #76 (TUI keymap ADR 0003), #75 (OpenStack), #74 (env aliases), #73 (kindsync cleanup), #72 (vSphere) | Active development |
| yage-docs | `main` | ADRs 0003–0006 written | Documentation in progress |

## Recent Changes

### 2026-04-30 — PRs #72–#76 merged; issues #66, #67, #69, #70, #82, #83 closed

**Five PRs merged in sequence:**
- **#72** — `provider/vsphere`: EnsureGroup + Inventory via govmomi; Proxmox cleanup (ADR 0002 items 2+6). Closes #67.
- **#74** — `refactor(config,pricing)`: remove `OS_*`/`YAGE_CURRENCY` env aliases + legacy snapshot JSON (ADR 0002 items 3–5).
- **#76** — `feat(xapiri)`: TUI keymap normalization + dispatch architecture fixes (ADR 0003). Closes #69.
- **#75** — `provider/openstack`: PatchManifest flavor resolution + Inventory via gophercloud; fix flavor env keys. Closes #66.
- **#73** — `refactor(kindsync)`: remove `SyncProxmoxBootstrapLiteralCredentials` (ADR 0002 item 1). Closes #70 (partial).

**ADRs 0003–0006** written and accepted in `yage-docs`.

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

**ADR 0003** written (`yage-docs/docs/architecture/adrs/0003-xapiri-tui-dispatch-keymap.md`):
- Keymap normalization: remove j/k from 4 dashboard locations + huh picker `WithKeyMap`
- Promote `inTextField` to method; add `tabCosts+costCredsMode` case
- `preserveTransientState` helper replaces fragile manual copy in `cfgEntryLoadMsg`
- Call `detectFork` in `initFromConfig` when InfraProvider empty
- Inline "location unset" prompt on costs tab when GeoIP off and DataCenterLocation empty
- **Issue #69** opened for Frontend agent to implement

**Cleanup worktrees** — all three are one commit ahead of `550698e`, need rebase onto `02b1948`:
- `worktree-agent-a529a47e1fc8d2cf7` (83941d5): ADR 0002 item 1 — remove `SyncProxmoxBootstrapLiteralCredentials` (done); item 7 (`InfraProvider` guard removal) deferred — only TODO comments added, guards still present
- `worktree-agent-aceeb96a31d0f49b7` (e060830): ADR 0002 items 3+4+5 — remove `OS_*`/`YAGE_CURRENCY` aliases + legacy snapshot JSON
- `worktree-agent-afd01feec4a66c948` (97b5288): OpenStack #66 — also needs rebase before PR

---

### 2026-04-30 — Architect assessment: phase status corrected; ADR 0002 no-compat policy

Architect audited the codebase and produced §25 addendum to `abstraction-plan.md`:

- **Phase status corrected**: CURRENT_STATE previously showed A/B/D/E as "Not started"; ground truth is A/B/E complete, D substantially complete (see Abstraction Plan Status table below).
- **No-backward-compatibility policy adopted** (ADR 0002): yage has no production users; all legacy fallbacks (dual-read/write migration, env-var aliases, Secret-name fallbacks, JSON-format fallbacks) are dead weight and will be deleted without deprecation cycle.
- **Three ADR documents written**: §25 addendum to `abstraction-plan.md`; ADR 0001 (CSI driver registry); ADR 0002 (backward compat removal).
- **Four parallel cleanup agents spawned** to execute ADR 0002 on branches: `refactor/kindsync-cleanup`, `refactor/env-aliases`, `refactor/proxmox-purge-cleanup`, and one additional branch. Status unknown as of this writing — check before starting new work on overlapping files.

---

### 2026-04-30 — xapiri: config/provision tab split (merged to main)

Split the previous single "Config" tab in xapiri into two separate tabs:
- **Config** (tab 1) — profile selection list and new-config name input
- **Provision** (tab 2) — full interactive edit form for the selected config

Selecting a config from the list automatically switches to the provision tab. All other tabs (3–9) remain gated on `cfgSelected`. Keyboard shortcuts 1–8, ctrl+alt+1–8, and mouse click-to-focus remapped to new indices. Help tab updated. `main` is at `02b1948`; `feat/xapiri-config-provision-split` is 2 commits ahead (b4b09df + 761f900 — proxmox cleanup + vSphere) and awaits merge.

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
| D | Generic kindsync + Purge | **Substantially complete** (KindSyncFields + WriteBootstrapConfigSecret in use; legacy wrappers being removed — see ADR 0002) |
| E | Pivot generalization (kind → any-cloud mgmt) | **Complete** (PivotTarget called via interface in pivot.go) |

---

## Active Work

| Branch | Owner | Description | Status |
|---|---|---|---|
| *(none — all recent branches merged)* | — | — | — |

---

## Known Issues

- xapiri is still a work-in-progress TUI; not all provider paths are fully wired.
- Cost estimation requires live Proxmox API; returns `ErrUnavailable` when unreachable.
- xapiri TUI: `feat/xapiri-tui-dispatch-fixes` fixes j/k, inTextField guard, detectFork, and costs tab prompt (ADR 0003 / issue #69) — pending merge.
- vSphere `Inventory()`: `Cores=0` — CPU expressed in MHz only (cannot derive cores from ResourcePool quota alone without host-speed query).

## Next Steps

1. **Issue #68** (p1) — integrate background A/B/D1/D4 worktrees (Backend); ordering constraint: D1 after B; delete `internal/capi/csi/` post-D1 per ADR 0001.
2. **Issue #71** (p2) — ADR 0002 item 7: remove redundant `cfg.InfraProvider == "proxmox"` guards (Backend).
3. **Issue #81** (p2) — flesh out ADR 0004: OpenTofu Phase G universal identity bootstrap (Architect).
4. **Issues #84–#93** (p2) — 10 CSI driver implementations per ADR 0001 (Backend, epic #77).
5. **Issues #94–#101** (p2) — 8 PlanDescriber provider implementations (Backend, epic #78).
6. **Issues #79, #80** (p3) — vSphere PatchManifest sizing + OpenStack EnsureIdentity clouds.yaml (Backend).

---

### 2026-04-30 — Platform Engineer issue refinement

Three open issues refined with K8s/infra-level implementation specs:

- **#66** (OpenStack PatchManifest + Inventory): Flagged pre-existing bug — `OPENSTACK_NODE_MACHINE_FLAVOR` vs `OPENSTACK_WORKER_FLAVOR` env key mismatch in TemplateVars vs K3s template. gophercloud v2 not yet in go.mod — must be added. Nova `limits.Get` + Cinder quotasets API paths specified.
- **#67** (vSphere Inventory + EnsureScope): govmomi not yet in go.mod — must be added. ResourcePool property query path specified. Unlimited pool (`-1`) edge case: return `ErrNotApplicable`. TLS thumbprint field check required.
- **#68** (Background agent integration): D1 depends on B. Ordering constraint: proxmox-csi `EnsureSecret` must run before HelmChartProxy fires. Post-D1: delete `internal/capi/csi/` per ADR 0001.

**Handoff to Backend:** #66 and #67 are ready for implementation. #68 is integration ops — check worktree/branch status of A/B/D1/D4 before starting.
