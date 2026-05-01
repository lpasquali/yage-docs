# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. Phase H (on-prem platform services via OpenTofu) is in late-stage implementation — ADRs 0009/0010/0011 accepted; orchestrator wiring for EnsureIssuingCA (#126/PR #163) and EnsureRepoSync (#144/PR #164) merged; only #125 (registry VM orchestrator wiring) remains. **Cross-repo gap**: yage-tofu currently has only the 8 identity modules (no `issuing-ca/` or `registry/` modules) — orchestrator wiring is dormant until those land in yage-tofu (tracked separately). ADR 0011 (pivot state migration) is substantially implemented; only #148 (extended HandOff + VerifyParity, plus a phase-order fix in bootstrap.go per ADR 0011 §7) remains.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-01** — PO session 7 (post-dispatch): PRs #161 ✅ `d6e7d0a`, #163 ✅ `7b9d19b`, #164 ✅ `e7d116e` all merged; #126/#144/#152 closed; backend agents A/B/C at Halt for #148/#125/#136; cross-repo gap surfaced (yage-tofu missing `issuing-ca/` and `registry/` modules).

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #164 (EnsureRepoSync, `e7d116e`), #163 (EnsureIssuingCA, `7b9d19b`), #161 (admin token TUI, `d6e7d0a`), #162 (JobRunner k8s backend), #160 (EnsureManagementInstall), #159 (profile bugs), #158 (Phase H TUI), #157 (double prompt fix), #156 (timeframe+ctrl+alt fix), #150, #149, #143, #132 | Active development; 3 backend agents at Halt for #148/#125/#136 |
| yage-docs | `main` | PR #9 (remove codeql.yml), PR #8 (ADRs 0010+0011+CURRENT_STATE) | Up to date |
| yage-tofu | `main` | PR #2 (kubernetes backend all 8 modules), PR #3 (license) | Up to date |
| yage-manifests | `init` | PR #3 (license) | Scaffolded, awaiting templates |

## Recent Changes

### 2026-05-01 — PO session 7: dispatch executed; 3 PRs merged; 3 backend halts

**Correction (session start):** Session 6 entry below originally listed PRs #161, #163, #164 as merged. They were OPEN at session 7 start. This entry records the actual merge events.

**yage PRs merged this session:**
- **PR #161** (`d6e7d0a`, squash) — `feat(xapiri): Proxmox admin token config in TUI; fix AdminToken kind Secret leak`. Closes #152. Merged by yage-frontend agent.
- **PR #163** (`7b9d19b`, squash) — `feat(opentofux): EnsureIssuingCA — generate intermediate CA + wire cert-manager ClusterIssuer`. Closes #126. Merged by yage-backend (B); needed two rebases (parallel #164 merge raced).
- **PR #164** (`e7d116e`, squash) — `feat(opentofux): implement EnsureRepoSync — yage-repos PVC + yage-repo-sync Job`. Closes #144. Merged by yage-backend (A).

**Issues closed this session (auto on merge):** #126, #144, #152.

**Backend agents at Halt — awaiting user OK to Execute:**

- **yage-backend (A)** on `feat/148-handoff-yage-system` — issue #148 plan: 3 changes (not 2). (1) `kindsync/handoff.go`: new `HandoffResult{NamedCopied, LabelCopied, FirstErr}` + `copyYageSystemSecrets` helper that SSA-applies labeled (`app.kubernetes.io/managed-by=yage`) Secrets in the `yage-system` namespace. (2) `pivot/pivot.go`: 3 verification helpers (ns-present, labeled-Secrets-present, yage-repos PVC bound) inside the existing poll loop. (3) **`bootstrap.go` phase reorder** — current order runs `VerifyParity` *before* `EnsureYageSystemOnCluster` and `EnsureRepoSync`, so the new checks would always fail; reorder per ADR 0011 §7 to: `MoveCAPIState → HandOff → EnsureYageSystemOnCluster → EnsureManagementInstall → EnsureRepoSync → VerifyParity → rebind`. Tests for both packages.

- **yage-backend (B)** on `feat/125-registry-vm` — issue #125 plan: new `internal/platform/opentofux/registry.go` exposing `EnsureRegistry(ctx, cli, cfg) error` (mirrors `EnsureIssuingCA` shape: `ErrNotApplicable` when `RegistryNode==""`; preserves operator-set `ImageRegistryMirror`); wire into `bootstrap.go` post-`EnsureRepoSync` + `EnsureIdentity`, before workload manifest apply; `purge.go` integration to destroy registry before kind teardown; unit tests for the no-op/preserve paths. **Three open questions for user**:
  1. `JobRunner.Output` is a known TODO referencing #125 explicitly; bundle its implementation (parse terminated-pod logs for `tofu output -json`) into this PR, or split?
  2. `yage-tofu/registry/` module **does not exist on `lpasquali/yage-tofu` main** — only the 8 identity modules. Ship orchestrator wiring + no-op tests now (runtime path dormant unless `RegistryNode` set), and open follow-up issue for the OpenTofu module? Or hold #125 until module lands?
  3. OK to add `NewJobRunnerWithClient(cfg, *k8sclient.Client)` constructor to mirror EnsureRepoSync wiring?

- **yage-backend (C)** on `feat/136-manifests-fetcher` — issue #136 plan: new `internal/platform/manifests/fetcher.go` with zero-value `Fetcher{MountRoot}` (default `/repos`), `Render(templatePath, data any) (string, error)`, path = `<root>/yage-manifests/<templatePath>`, `text/template` with `Option("missingkey=error")`. Plus `_test.go` and a testdata fixture. **Two design picks for user**:
  1. Zero-value `Fetcher` (mirrors opentofux pattern, agent recommends) **vs** `NewFetcher()` constructor (issue text suggests).
  2. Confirm `Option("missingkey=error")` so missing config fields fail loud instead of silently rendering `<no value>`.

**Cross-repo gap discovered + tracked:**
yage-tofu top-level has only `aws/`, `azure/`, `gcp/`, `ibmcloud/`, `linode/`, `oci/`, `openstack/`, `proxmox/` (the 8 identity modules). The `issuing-ca/` and `registry/` modules — referenced by the just-merged PR #163 (EnsureIssuingCA) and the in-flight #125 (EnsureRegistry) respectively — do not exist. The yage-side orchestrator wiring is dormant (guarded by `ErrNotApplicable` when config fields are unset) so this is not a runtime regression, but the operator-facing feature is non-functional until the modules ship. **Tracking issues opened**: [yage-tofu#4 (issuing-ca/)](https://github.com/lpasquali/yage-tofu/issues/4) and [yage-tofu#5 (registry/)](https://github.com/lpasquali/yage-tofu/issues/5).

**Architect / platform-engineer:** standby this session; no Execute work dispatched.

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
| #148 | `feat/148-handoff-yage-system` | **yage-backend (A)** | extend `HandOffBootstrapSecretsToManagement` (labeled yage-system copy) + extend `VerifyParity` (ns + labeled Secrets + yage-repos PVC) + **reorder pivot phases per ADR 0011 §7** | **HALT** — p1; awaiting user OK to Execute |
| #125 | `feat/125-registry-vm` | **yage-backend (B)** | provision bootstrap registry VM via OpenTofu; mirror `EnsureIssuingCA` shape; auto-wire `ImageRegistryMirror` | **HALT** — p2; 3 open questions for user (see session 7 entry) |
| #136 | `feat/136-manifests-fetcher` | **yage-backend (C)** | `internal/platform/manifests` Fetcher package (MountRoot-based, no local clone); ADR 0008 step 2 | **HALT** — p2; 2 design picks for user (constructor shape, `missingkey=error`) |
| yage-tofu#4 | TBD | **yage-backend** (cross-repo) | add `yage-tofu/issuing-ca/` OpenTofu module — companion to merged yage PR #163 | **Open** — p2 |
| yage-tofu#5 | TBD | **yage-backend** (cross-repo) | add `yage-tofu/registry/` OpenTofu module — companion to in-flight yage #125 | **Open** — p2 |
| #137 | TBD | **yage-backend** | migrate `internal/capi/helmvalues/` to yage-manifests templates | **Blocked** on #136 |
| #138 | TBD | **yage-backend** | migrate `internal/capi/wlargocd/` renderers to yage-manifests templates | **Blocked** on #136 |
| #139 | TBD | **yage-backend** | migrate `internal/capi/postsync/` renderers to yage-manifests templates | **Blocked** on #136 |
| #140 | TBD | **yage-backend** | migrate `internal/capi/caaph/` string renderers to yage-manifests templates | **Blocked** on #136 |
| #141 | TBD | **yage-backend** | CSI `RenderValues` → `Render` + port all 14 drivers to yage-manifests (atomic) | **Blocked** on #136 |
| #142 | TBD | **yage-backend** | retire `helmvalues/`, `wlargocd/`, `postsync/` packages | **Blocked** on #137–#141 |
| #119 | TBD | **yage-backend** | D4: CAPD smoke E2E test for bootstrap pipeline | **Backlog** — p3 |
| #94–#101 | TBD | **yage-backend** | PlanDescriber for 8 providers (Linode, CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO) | **Backlog** — p3 (epic #78); parallelizable across multiple backend instances |

---

## Critical Path (in order)

1. **User decides** on the 3 backend Halts — issue #148 (Execute OK?), issue #125 (3 questions), issue #136 (2 picks). Prefer batched OK to keep agents moving.
2. **Backend agents Execute** their dispatched issues; PRs land normally via Workflow SOP.
3. **PO opens cross-repo issues** in yage-tofu for `issuing-ca/` and `registry/` modules so the just-merged orchestrator wiring becomes operationally usable.
4. **Phase H epic #120** closes once #125 lands AND yage-tofu modules ship. Track both arms.
5. **Manifests migration chain** (#137–#142) unblocks once #136 ships.

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
