# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. Phase H (on-prem platform services via OpenTofu) is underway — ADRs 0009/0010/0011 accepted, most prereqs merged; PRs #163 (EnsureIssuingCA / #126) and #164 (EnsureRepoSync / #144) are CI-green and pending merge, after which #148/#125/#136 are fully unblocked. ADR 0011 (pivot state migration) is substantially implemented; only #148 (extended HandOff + VerifyParity) remains.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-01** — PO session 7: corrected CURRENT_STATE — PRs #161, #163, #164 are OPEN (not merged); #157–#160, #162, #165 confirmed merged; dispatched #148, #136, #125 to yage-backend.

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #162 (JobRunner k8s backend), #160 (EnsureManagementInstall), #159 (profile bugs), #158 (Phase H TUI), #157 (double prompt fix), #156 (timeframe+ctrl+alt fix), #150 (EnsureYageSystemOnCluster), #149 (YAGE_MANIFESTS_REF), #143 (Fetcher PVC resolver), #132 (Phase H config fields) | Active development; PRs #161/#163/#164 open, CI green |
| yage-docs | `main` | PR #9 (remove codeql.yml), PR #8 (ADRs 0010+0011+CURRENT_STATE) | Up to date |
| yage-tofu | `main` | PR #2 (kubernetes backend all 8 modules), PR #3 (license) | Up to date |
| yage-manifests | `init` | PR #3 (license) | Scaffolded, awaiting templates |

## Recent Changes

### 2026-05-01 — PO session 7: state correction; dispatch

**Correction:** Session 6 entry below originally listed PRs #161, #163, #164 as merged. GitHub shows all three are still **OPEN with CI green**. Issues #126, #144, #152 remain open until those PRs merge. Session 6 entry corrected; this entry records the dispatch.

**Per-agent dispatch (each agent merges its assigned PR, then starts the linked issue):**
- **yage-frontend** — merge PR #161 (closes #152, xapiri admin token TUI). Idle after.
- **yage-backend (A)** — merge PR #164 (closes #144), then issue #148 (p1) `HandOff` label-pass + `VerifyParity` extension.
- **yage-backend (B)** — merge PR #163 (closes #126), then issue #125 (p2) bootstrap registry VM via OpenTofu.
- **yage-backend (C)** — issue #136 (p2) `manifests.Fetcher` package; gate: confirm PR #164 is on `main` before Execute.
- **yage-architect / yage-platform-engineer** — standby; reactivate for #125 PR manifest review and Phase I ADR scope.

**Merge protocol** (every agent): rebase onto latest `main` immediately before merging to avoid a second CodeQL scan; use `gh pr merge <N> --squash`.

---

### 2026-05-01 — PO session 6: Phase H prereqs landed (partial); ADR 0011 substantially implemented; licenses; CodeQL removed

**yage PRs merged:**
- **PR #157** — `fix: skip pickBootstrapConfig before xapiri dashboard`. Closes #151. ✅
- **PR #158** — `feat(xapiri+plan): surface Phase H registry and issuing CA fields`. Closes #127. ✅
- **PR #159** — `fix(kindsync+xapiri): config profile system bug fixes`. Closes #153. ✅
- **PR #160** — `feat(csi): Driver.EnsureManagementInstall — CSI on management cluster (ADR 0011, #147)`. Closes #147. ✅
- **PR #162** — `feat(opentofux): switch JobRunner to OpenTofu kubernetes backend; eliminate tofu-state PVCs`. Closes #145 (yage side). ✅
- **PR #165** — `chore: remove codeql.yml`. CodeQL was not gating Merge Gate; removed to clean up check list. ✅

**yage PRs open — CI green, awaiting merge:**
- **PR #161** — `feat(xapiri): Proxmox admin token config in TUI; fix AdminToken kind Secret leak`. Closes #152. ⏳
- **PR #163** — `feat(opentofux): EnsureIssuingCA — generate intermediate CA + wire cert-manager ClusterIssuer`. Closes #126. ⏳
- **PR #164** — `feat(opentofux): implement EnsureRepoSync — yage-repos PVC + yage-repo-sync Job (ADR 0010, #144)`. Closes #144. ⏳

**Also merged earlier in session (from session summary):**
- **PR #156** — fix timeframe `[`/`]` keys on costs tab + remove ctrl+alt+Number tab shortcuts. Closes #154+#155. ✅

**yage-tofu PRs merged:**
- **PR #2** — `feat: switch all modules to OpenTofu kubernetes backend`. Closes #145 (yage-tofu side). All 8 modules now have `backend.tf`. ✅
- **PR #3** — `chore: add Apache 2.0 license`. ✅

**yage-manifests PRs merged:**
- **PR #3** — `chore: add Apache 2.0 license`. ✅

**yage-docs PRs merged:**
- **PR #9** — `chore: remove codeql.yml`. ✅

**Housekeeping:**
- Apache 2.0 LICENSE added to all repos that were missing it: `yage-tofu`, `yage-manifests`, `workload-smoketests`. (`yage`, `yage-docs`, `workload-app-of-apps` already had it.)
- CodeQL removed from `yage` (PR #165) and `yage-docs` (PR #9). Neither check contributed to Merge Gate; they added noise and transient cancellation failures.
- yage-tofu and yage-manifests rulesets cleaned up: removed `code_quality`, `code_scanning`, `required_status_checks` (yage-tofu has no CI; yage-manifests has no CI). Both keep `pull_request` + `deletion` + `non_fast_forward`.

**Issues closed this session:**
- #127 (Phase H TUI) — PR #158 ✅
- #145 (kubernetes backend) — PR #162 + yage-tofu PR #2 ✅
- #147 (EnsureManagementInstall) — PR #160 ✅
- #151 (double prompt) — PR #157 ✅
- #153 (profile bugs) — PR #159 ✅
- #154 (timeframe keys) + #155 (ctrl+alt shortcuts) — PR #156 ✅

**Issues pending closure (awaiting PR merge):**
- #126 (EnsureIssuingCA) — PR #163 open, CI green ⏳
- #144 (EnsureRepoSync) — PR #164 open, CI green ⏳
- #152 (admin token TUI) — PR #161 open, CI green ⏳

---

### 2026-05-01 — PO session 5 (earlier): ADR 0011; pivot state migration issues; manifests epic

*(Phase H prereq PRs merged: #132, #143, #149, #150; issues #144–#148, #133–#142, #151–#155 created; PR #128 merged.)*

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
| PR #161 | `worktree-agent-a2aa9a093ec134e40` | **yage-frontend** | rebase onto `main`, force-push, then `gh pr merge 161 --squash` — closes #152 (xapiri admin token TUI) | **CI green** — assigned for merge |
| PR #163 | `feat/126-issuing-ca` | **yage-backend (B)** | rebase onto `main`, force-push, then `gh pr merge 163 --squash` — closes #126 (EnsureIssuingCA); advances epic #120 | **CI green** — assigned for merge |
| PR #164 | `worktree-agent-af60c9cba9f0fffa6` | **yage-backend (A)** | rebase onto `main`, force-push, then `gh pr merge 164 --squash` — closes #144 (EnsureRepoSync); unblocks #148/#136/#125 | **CI green** — assigned for merge |
| #148 | new branch | **yage-backend (A)** | after merging PR #164, start: extend `HandOffBootstrapSecretsToManagement` (label-based yage-system copy) + extend `VerifyParity` (yage-system ns + labeled Secrets + yage-repos PVC) | **Dispatched** — p1 |
| #125 | new branch | **yage-backend (B)** | after merging PR #163 and confirming PR #164 merged, start: provision bootstrap registry VM via OpenTofu and auto-wire `ImageRegistryMirror` | **Dispatched** — p2 |
| #136 | new branch | **yage-backend (C)** | after PR #164 merge confirmed, start: `internal/platform/manifests` Fetcher package (MountRoot-based, no local clone); ADR 0008 step 2 | **Dispatched** — p2; prerequisite for #137–#142 |
| #137 | TBD | **yage-backend** | migrate `internal/capi/helmvalues/` to yage-manifests templates | **Blocked** on #136 |
| #138 | TBD | **yage-backend** | migrate `internal/capi/wlargocd/` renderers to yage-manifests templates | **Blocked** on #136 |
| #139 | TBD | **yage-backend** | migrate `internal/capi/postsync/` renderers to yage-manifests templates | **Blocked** on #136 |
| #140 | TBD | **yage-backend** | migrate `internal/capi/caaph/` string renderers to yage-manifests templates | **Blocked** on #136 |
| #141 | TBD | **yage-backend** | CSI `RenderValues` → `Render` + port all 14 drivers to yage-manifests (atomic) | **Blocked** on #136 |
| #142 | TBD | **yage-backend** | retire `helmvalues/`, `wlargocd/`, `postsync/` packages | **Blocked** on #137–#141 |
| #119 | TBD | **yage-backend** | D4: CAPD smoke E2E test for bootstrap pipeline | **Backlog** — p3 |
| #94–#101 | TBD | **yage-backend** | PlanDescriber for 8 providers (Linode, CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO) | **Backlog** — p3 (epic #78); parallelizable across multiple backend instances |

---

## Per-agent dispatch (session 7)

**Each agent: rebase the assigned PR branch onto latest `main`, force-push, then `gh pr merge --squash`. Wait for CI confirmation.** The rebase-immediately-before-merge step is mandatory — see CODING_STANDARDS / SOP step 8 to avoid the second CodeQL scan.

| Agent | Merge | Then start |
|---|---|---|
| **yage-frontend** | PR #161 → closes #152 | (idle) |
| **yage-backend (A)** | PR #164 → closes #144 | issue #148 (p1) — `HandOff` + `VerifyParity` |
| **yage-backend (B)** | PR #163 → closes #126 | issue #125 (p2) — registry VM via OpenTofu |
| **yage-backend (C)** | (none) | issue #136 (p2) — `manifests.Fetcher` package; **gate**: confirm PR #164 merged on `main` before SOP step 5 (Execute) |
| **yage-architect** | (none) | review ADR scope for Phase I (post yage-manifests migration); standby — no Execute work this session |
| **yage-platform-engineer** | (none) | manifest review for #125 PR (registry mirror wiring touches CAPI patching) when posted; standby |

**Order of operations within each backend agent**: merge first (the merge work is independent of the Execute gate, since the PR is already through Halt+Verify on a previous session), then SOP for the new issue. Halt before Execute on the new issue and report back.

**Coordination note**: agent C must verify `gh pr view 164 --json state` returns `MERGED` before SOP step 5. If not yet merged, hold at SOP step 4 (Halt) until agent A completes its merge.

---

## Known Issues

- xapiri is still a work-in-progress TUI; not all provider paths are fully wired.
- Cost estimation requires live Proxmox API; returns `ErrUnavailable` when unreachable.
- vSphere `Inventory()`: `Cores=0` — CPU expressed in MHz only.

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
