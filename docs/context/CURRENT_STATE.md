# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. Phase H (on-prem platform services via OpenTofu) is next — ADRs 0009/0010/0011 accepted. PR #128 (OpenStack EnsureIdentity) merged to main.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-01** — PO session 5: ADR 0011 committed; issues #144–#148 created; yage-manifests epic #133–#142 added; PR #128 merged to main; PR #132 MERGEABLE; PR #143 needs two changes before merge

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #128 (OpenStack EnsureIdentity), #131 (D1 csi.Selector), #130 (ADR 0002 item 7), #129 (vsphere PatchManifest) | Active development |
| yage-docs | `docs/po-session-3-state-update` | ADRs 0001–0011 written and accepted | PR #8 open, MERGEABLE |

## Recent Changes

### 2026-05-01 — PO session 5: ADR 0011; pivot state migration issues; manifests epic; PR #128 merged

**PR merged to main:**
- **PR #128** merged — `feat(openstack): EnsureIdentity — apply clouds.yaml Secret`. Closes #80. ✅

**yage-docs: ADRs 0010 and 0011 committed to `docs/po-session-3-state-update`:**

- **ADR 0010** — In-cluster repository cache (zero workstation residue): `yage-repos` PVC (500Mi, yage-system), `yage-repo-sync` Job (alpine/git init containers), updated Fetcher structs (MountRoot-based, no CacheDir), registry provisioning rescheduled post-kind. Supersedes local `~/.yage/tofu-cache/` and `~/.yage/manifests-cache/`.

- **ADR 0011** — Pivot: yage state migration to management cluster. Seven decisions:
  1. Switch `yage-tofu` modules to OpenTofu `kubernetes` backend — state as `tfstate-default-<module>` Secrets in `yage-system` (labeled `app.kubernetes.io/managed-by=yage`); eliminates `tofu-state-<module>` PVCs.
  2. Extend `HandOffBootstrapSecretsToManagement` with label-based (`app.kubernetes.io/managed-by=yage`) yage-system Secret copy.
  3. New `EnsureYageSystemOnCluster` — re-creates SA + Role + RoleBinding from in-code spec on any cluster (not copied); prerequisite for tofu kubernetes backend and EnsureRepoSync.
  4. `csi.Driver.EnsureManagementInstall` — management CSI install on Driver interface (Provider interface stays CSI-free per ADR 0001); Proxmox uses direct Helm (ArgoCD not running on mgmt at this stage).
  5. `EnsureRepoSync` called on management cluster post-handoff.
  6. Extended `VerifyParity`: yage-system namespace present + ≥0 labeled Secrets + `yage-repos` PVC Bound.
  7. Updated pivot sequence with explicit ordering of all new steps.

**New issues created:**

*yage-manifests epic (ADR 0008):*
- **#133** — epic: yage-manifests — migrate all inline YAML generation to external template repo
- **#134** — ops: scaffold lpasquali/yage-manifests repository structure
- **#135** — config: add `YAGE_MANIFESTS_REF` field
- **#136** — manifests: implement `internal/platform/manifests` Fetcher package
- **#137** — manifests: migrate `internal/capi/helmvalues/` to yage-manifests templates
- **#138** — manifests: migrate `internal/capi/wlargocd/` ArgoCD Application renderers
- **#139** — manifests: migrate `internal/capi/postsync/` renderers
- **#140** — manifests: migrate `internal/capi/caaph/` string renderers
- **#141** — csi: migrate `RenderValues` → `Render` + port all 14 drivers to yage-manifests (atomic)
- **#142** — manifests: retire `internal/capi/helmvalues/`, `wlargocd/`, `postsync/` packages

*Pivot state migration (ADR 0011):*
- **#144** — orchestrator: create `yage-repos` PVC and run `yage-repo-sync` Job — updated to require cluster-agnostic `EnsureRepoSync` callable on both kind and management cluster
- **#145** — opentofux: switch yage-tofu modules to OpenTofu kubernetes backend (eliminate `tofu-state-<module>` PVCs)
- **#146** — kindsync: `EnsureYageSystemOnCluster` — SA + RBAC bootstrap from spec on any cluster
- **#147** — csi: add `Driver.EnsureManagementInstall` + Proxmox implementation + `PROXMOX_CSI_MGMT_STORAGE_CLASS` field
- **#148** — kindsync+pivot: extend HandOff with label-based yage-system copy + extend VerifyParity

**PR status:**
- **PR #132** (`feat/124-phase-h-config-fields`): all CI green, MERGEABLE. **Programmer: merge now.** Closes #124.
- **PR #143** (`feat/123-opentofux-fetcher-runner`): MERGEABLE by CI but **review comment requires two changes before merge** — (1) `fetcher.go`: replace local clone with PVC path resolver per ADR 0010; (2) `opentofux.go`: delete, do not keep as stub per ADR 0002. Backend must address both, then Programmer merges.
- **PR #8** (yage-docs `docs/po-session-3-state-update`): MERGEABLE. Contains ADR 0010 + ADR 0011 + CURRENT_STATE. Programmer: merge when ready.

**Issue updates:**
- **#123**: Acceptance criteria updated — Fetcher is PVC path resolver (no local clone), `opentofux.go` deleted. Review comment left on PR #143 documenting required changes.
- **#125**: Now also blocked on #144 (EnsureRepoSync) in addition to #123 + #124.
- **#136**: Fetcher interface updated per ADR 0010 (MountRoot-based, no CacheDir/RepoURL/Ref).

---

### 2026-05-01 — PO session 4: PRs #129–#131 reconciled; Phase H issues #123–#127 added to Active Work; PR #128 ready to merge

**PRs merged to main (2026-04-30 late, not reflected in previous CURRENT_STATE update):**

- **PR #129** merged — `feat(vsphere): PatchManifest — honor VSphereMachineTemplate sizing fields`. Closes #79. Backend complete.
- **PR #130** merged — `refactor(orchestrator): remove redundant InfraProvider=="proxmox" guards (ADR 0002 item 7)`. Closes #71. ADR 0002 item 7 now done.
- **PR #131** merged — `feat(orchestrator): wire csi.Selector; delete internal/capi/csi`. Closes #118. D1 complete; `internal/capi/csi/` deleted from codebase.

**Phase H implementation issues created (all p2, children of epic #120):**
- **#123** — opentofux: introduce Fetcher+Runner abstraction (Backend, prerequisite for #125 + #126)
- **#124** — config: add `YAGE_REGISTRY_*` and `YAGE_ISSUING_CA_*` fields (Backend, prerequisite for all)
- **#125** — orchestrator: provision bootstrap registry VM via OpenTofu (Backend, blocked on #123 + #124 + #144)
- **#126** — orchestrator: provision online issuing CA + wire cert-manager ClusterIssuer (Backend, blocked on #123 + #124)
- **#127** — xapiri+plan: surface Phase H registry and issuing CA fields in TUI + `--dry-run` (Frontend/Backend, blocked on #124)

---

### 2026-04-30 — PO session 3: CSI wave complete; ADR docs merged; epics closed

*(See git log for full detail — ADRs 0001–0009 accepted, CSI epic #77 closed, ADR 0002 legacy cleanup complete, all 10 CSI drivers merged.)*

---

## Abstraction Plan Status

Tracked in [yage-docs ADR](https://lpasquali.github.io/yage-docs/architecture/adrs/abstraction-plan/).

| Phase | Goal | Status |
|---|---|---|
| C | Config namespacing (`cfg.Proxmox*` → `cfg.Providers.Proxmox.*`) | **Merged** |
| A | Inventory behind `Provider.Inventory()` | **Complete** |
| B | Plan body delegation per provider | **Complete** |
| D | Generic kindsync + Purge | **Substantially complete** |
| E | Pivot generalization (kind → any-cloud mgmt) | **Complete** |

---

## Active Work

| Issue / PR | Branch | Agent | Description | Status |
|---|---|---|---|---|
| PR #132 | `feat/124-phase-h-config-fields` | **Programmer** | Merge PR #132. Closes #124. All CI green. | **Ready** — MERGEABLE |
| PR #8 (yage-docs) | `docs/po-session-3-state-update` | **Programmer** | Merge PR #8 (ADR 0010 + ADR 0011 + CURRENT_STATE). | **Ready** — MERGEABLE |
| PR #143 | `feat/123-opentofux-fetcher-runner` | **yage-backend** | Two changes required before merge: (1) `fetcher.go` → PVC path resolver per ADR 0010; (2) delete `opentofux.go` per ADR 0002. Then Programmer merges. | **Needs Backend fix** |
| #146 | TBD | **yage-backend** | `EnsureYageSystemOnCluster` — SA + Role + RoleBinding from spec on any cluster | **Unblocked** — p1, prerequisite for #144 + #145 |
| #144 | TBD | **yage-backend** | `EnsureRepoSync` — create `yage-repos` PVC + `yage-repo-sync` Job; cluster-agnostic (kind + mgmt) | **Blocked** on #123 + #135 + #146 |
| #145 | TBD | **yage-backend** | OpenTofu `kubernetes` backend for yage-tofu modules; eliminates `tofu-state-<module>` PVCs | **Blocked** on #123 + #146 |
| #147 | TBD | **yage-backend** | `csi.Driver.EnsureManagementInstall` + Proxmox impl + `PROXMOX_CSI_MGMT_STORAGE_CLASS` config field | **Blocked** on #146 |
| #148 | TBD | **yage-backend** | Extend `HandOffBootstrapSecretsToManagement` (label pass) + extend `VerifyParity` (yage-system + PVC checks) | **Blocked** on #145 + #146 + #144 + #147 |
| #135 | TBD | **yage-backend** | config: add `YAGE_MANIFESTS_REF` field | **Unblocked** — p2 (can parallel with #123 fix) |
| #136 | TBD | **yage-backend** | `internal/platform/manifests` Fetcher package (MountRoot-based, no local clone) | **Blocked** on #135 + #144 |
| #125 | TBD | **yage-backend** | orchestrator: provision bootstrap registry VM via OpenTofu | **Blocked** on #123 + #124 + #144 |
| #126 | TBD | **yage-backend** | orchestrator: provision online issuing CA + wire cert-manager ClusterIssuer | **Blocked** on #123 + #124 |
| #127 | TBD | **yage-frontend** / **yage-backend** | xapiri+plan: surface Phase H fields in TUI and `--dry-run` | **Blocked** on #124 (merge PR #132) |
| #134 | TBD | **yage-backend** / **ops** | scaffold `lpasquali/yage-manifests` repo structure | **Unblocked** — p2 |
| #137 | TBD | **yage-backend** | migrate `internal/capi/helmvalues/` to yage-manifests templates | **Blocked** on #136 |
| #138 | TBD | **yage-backend** | migrate `internal/capi/wlargocd/` renderers to yage-manifests templates | **Blocked** on #136 |
| #139 | TBD | **yage-backend** | migrate `internal/capi/postsync/` renderers to yage-manifests templates | **Blocked** on #136 |
| #140 | TBD | **yage-backend** | migrate `internal/capi/caaph/` string renderers to yage-manifests templates | **Blocked** on #136 |
| #141 | TBD | **yage-backend** | CSI `RenderValues` → `Render` + port all 14 drivers to yage-manifests (atomic) | **Blocked** on #136 |
| #142 | TBD | **yage-backend** | retire `helmvalues/`, `wlargocd/`, `postsync/` packages | **Blocked** on #137–#141 |
| #119 | TBD | **yage-backend** | D4: CAPD smoke E2E test for bootstrap pipeline | **Backlog** — p3 |
| #94–#101 | TBD | **yage-backend** | PlanDescriber for 8 providers (Linode, CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO) | **Backlog** — p3 (epic #78) |

---

## Critical Path (unblocked work agents can start now)

1. **Programmer: merge PR #132** — closes #124; unblocks #127, #125, #126
2. **Programmer: merge yage-docs PR #8** — publishes ADR 0010 + ADR 0011
3. **yage-backend: fix PR #143** — address two ADR 0010 review comments, then Programmer merges; closes #123; unblocks #125, #126, #144, #145
4. **yage-backend: start #146** (`EnsureYageSystemOnCluster`) — unblocked; prerequisite for #144, #145, #147
5. **yage-backend: start #135** (`YAGE_MANIFESTS_REF` config field) — unblocked; can parallel with #146
6. **yage-backend: start #134** (scaffold yage-manifests repo) — unblocked; independent

---

## Known Issues

- xapiri is still a work-in-progress TUI; not all provider paths are fully wired.
- Cost estimation requires live Proxmox API; returns `ErrUnavailable` when unreachable.
- vSphere `Inventory()`: `Cores=0` — CPU expressed in MHz only.
- PR #143 (`feat/123-opentofux-fetcher-runner`) implements the old ADR 0008 §3 Fetcher (local clone to `~/.yage/tofu-cache/`). Must be updated to ADR 0010 PVC path resolver before merge.

## ADR Index

| ADR | Title | Status |
|---|---|---|
| 0001 | CSI driver registry | Accepted |
| 0002 | Backward compat removal | Accepted |
| 0003 | xapiri TUI dispatch keymap | Accepted |
| 0004 | Universal OpenTofu identity bootstrap (Phase G) | Accepted |
| 0005 | Pricing/cost subsystem | Accepted |
| 0006 | Capacity preflight architecture | Accepted |
| 0007 | xapiri dashboard as default entry | Accepted |
| 0008 | yage-manifests GitOps template repository | Accepted |
| 0009 | On-prem platform services (Phase H): registry + issuing CA | Accepted |
| 0010 | In-cluster repository cache (zero workstation residue) | Accepted |
| 0011 | Pivot: yage state migration to management cluster | Accepted |
