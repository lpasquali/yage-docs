# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan (phases C → E → B → A → D) is partially complete — Phase C (config namespacing) is merged.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-04-30** — xapiri config/provision tab split (feat/xapiri-config-provision-split, in progress).

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #65 (config ns revamp), #64 (kindsync/pricing), #63 (network config), #62 (costs/UX) | Active development |
| yage-docs | `main` | — | Documentation in progress |

## Recent Changes

### 2026-04-30 — xapiri: config/provision tab split (feat/xapiri-config-provision-split, open)

Splitting the previous single "Config" tab in xapiri into two separate tabs:
- **Config** — profile selection (list of saved configs to pick from)
- **Provision** — edit form for the selected config

Work in progress on branch `feat/xapiri-config-provision-split`.

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
| E | Pivot generalization (kind → any-cloud mgmt) | Not started |
| B | Plan body delegation per provider | Not started |
| A | Inventory behind `Provider.Inventory()` | Not started |
| D | Generic kindsync + Purge | Not started |

---

## Active Work

| Branch / Issue | Summary | Status |
|---|---|---|
| `feat/xapiri-config-provision-split` | Split config tab → config (selection) + provision (edit form) | In progress |

---

## Known Issues

- xapiri is still a work-in-progress TUI; not all provider paths are fully wired.
- Pivot phase (Phase E) is Proxmox-only; generalization deferred.
- Cost estimation requires live Proxmox API; returns `ErrUnavailable` when unreachable.

## Next Steps

- Merge `feat/xapiri-config-provision-split` once config/provision split is stable.
- Continue xapiri walkthrough completeness (all provider fields reachable via TUI).
- Begin Phase E: pivot generalization.
