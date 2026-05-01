# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. **ADR 0011 (pivot state migration) is fully implemented** — yage PR #166 (`bb60c79`) merged today closes #148. Phase H (on-prem platform services via OpenTofu) orchestrator wiring is complete on the yage side (EnsureIssuingCA #163, EnsureRepoSync #164, EnsureManagementInstall #160, JobRunner k8s backend #162); only #125 (registry VM orchestrator wiring) remains, HELD pending architect review of the cross-repo modules. **Cross-repo gap is closing, not closed**: yage-tofu PR #6 (`registry/`, closes yage-tofu#5) and PR #7 (`issuing-ca/`, closes yage-tofu#4) are now open and need architect review (5 decisions + 2 contradictions flagged in PR bodies). yage-side wiring stays dormant (`ErrNotApplicable`-guarded) until they merge. ADR 0012 — yage-manifests template layout addendum to ADR 0008 — **merged** as yage-docs PR #11 (`1cdf251`); pins the `missingkey=error` policy + wrapper-struct contract that #136 needs. ADR 0009 erratum E1 (kubernetes backend supersedes local tofu state) **merged** as yage-docs PR #12 (`20cb4b9`). Phase H bootstrap runbook **merged** as yage-docs PR #13 (`c1930af`).

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-01** — PO session 8: PR #166 (`bb60c79`) merged — closes #148, fully implements ADR 0011, clears yage-backend (A) HALT. yage-tofu PR #6 (`registry/`, closes yage-tofu#5) and PR #7 (`issuing-ca/`, closes yage-tofu#4) opened for architect review. yage-docs PR #11 (ADR 0012) opened — closes #136 design pick #2. yage-backend (B) rotated from HELD #125 onto now-unblocked #136 and is **executing** on `feat/136-manifests-fetcher`. #125 remains HELD pending review of yage-tofu PRs #6/#7. #137–#141 remain gated on #136 landing.

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #166 (label-based yage-system handoff + extended VerifyParity, `bb60c79`), #164 (EnsureRepoSync, `e7d116e`), #163 (EnsureIssuingCA, `7b9d19b`), #161 (admin token TUI, `d6e7d0a`), #162 (JobRunner k8s backend), #160 (EnsureManagementInstall), #159, #158, #157, #156, #150, #149, #143, #132 | Active development; (A) HALT cleared by #166; (B) executing #136; #125 HELD |
| yage-docs | `main` | PR #13 (Phase H runbook, `c1930af`), PR #12 (ADR 0009 erratum E1, `20cb4b9`), PR #11 (ADR 0012 template layout, `1cdf251`), PR #9 (remove codeql.yml), PR #8 (ADRs 0010+0011+CURRENT_STATE) | PR #11/#12/#13 merged |
| yage-tofu | `main` + open PRs #6, #7 | PR #2 (kubernetes backend all 8 modules), PR #3 (license) | PRs #6 (registry) + #7 (issuing-ca) awaiting architect review |
| yage-manifests | `init` | PR #3 (license) | Scaffolded, awaiting templates (#137–#141 gated on #136) |

## Recent Changes

### 2026-05-01 — PO session 8: #148 merged; cross-repo PRs opened; ADR 0012 in review; (B) rotated onto #136

**yage merged:**
- **PR #166** (`bb60c79`, squash) — `feat(kindsync+pivot): label-based yage-system handoff + extended VerifyParity (#148)`. Closes #148. Fully implements ADR 0011. Clears yage-backend (A) HALT. ✅

**Issues closed (auto on merge):** #148.

**Cross-repo PRs opened (awaiting architect review):**

- **yage-tofu PR #6** — `feat: add registry/ module for Phase H bootstrap registry (#5)` on `registry-module`. Closes yage-tofu#5. Five interpretive decisions flagged for architect:
  1. **kubernetes backend** used despite ADR 0009 §1 wording `~/.yage/tofu/registry/terraform.tfstate` (local) — resolved by ADR 0009 erratum E1 (yage-docs PR #12, `20cb4b9`).
  2. `vm_flavor` split into discrete `registry_vm_cores` / `registry_vm_memory_mb` / `registry_vm_disk_gb` (defaults 2 / 4 GiB / 100 GiB) instead of single `YAGE_REGISTRY_VM_FLAVOR` knob.
  3. TLS material exposed as PEM strings (`registry_tls_cert_pem`, `registry_tls_key_pem`, `registry_ca_bundle_pem`); defaults `""` so plan succeeds without certs.
  4. Cloud-init bash bootstrap pulling Harbor 2.11.0 / Zot v2.1.0 on first boot; no baked image; image seeding remains an operator step per ADR 0009 §1.
  5. `registry_template_id` is a required input — operator must supply a cloud-init enabled template ID.

- **yage-tofu PR #7** — `feat: add issuing-ca/ module — online intermediate CA (#4)` on `feat/4-issuing-ca-module`. Closes yage-tofu#4. Two contradictions surfaced in the PR body:
  1. **Output names not yet consumed by yage.** `EnsureIssuingCA` at `7b9d19b` generates the intermediate inline using Go `crypto/x509` and never calls `tofu output`. Module names mirror the Go-side material so a follow-up Go-side migration to `JobRunner.Output` becomes a no-op contract change.
  2. **Root CA is an input, not generated.** yage-tofu#4 brief said "root + intermediate via `tls_self_signed_cert` / `tls_locally_signed_cert`" but yage code receives the root from operator config (`cfg.IssuingCARootCert` / `cfg.IssuingCARootKey`). Module follows Go-side semantics: offline-managed root is an input variable, not a TF resource — preserves the offline-root boundary per ADR 0009.

**yage-docs PRs merged (post-session-8):**
- **yage-docs PR #11** (`1cdf251`) — `docs(adr): ADR 0012 — yage-manifests template layout and data contract`. Addendum to ADR 0008 pinning: on-disk layout (`cluster/`, `addons/<addon>/`, `csi/<driver>/`, `postsync/`), `missingkey=error` rendering policy, six wrapper-struct data contracts, full migration mapping table (helmvalues + wlargocd + postsync + caaph + cilium LBIPAMPool + 14 CSI drivers). Closes the open design pick #2 on #136 in the affirmative.
- **yage-docs PR #12** (`20cb4b9`) — `docs(adr-0009): erratum E1 — kubernetes backend supersedes local tofu state`. Resolves decision #1 on yage-tofu PR #6.
- **yage-docs PR #13** (`c1930af`) — `docs(operations): add Phase H bootstrap runbook (registry + issuing CA)`.

**Unblocking moves this session:**

- **#148 merged** (PR #166) clears yage-backend (A) HALT — agent rolls off.
- **#136 unblocked** — its only deps were #135 (`d13e0eb`) and #144 (`e7d116e`), both merged in session 7. ADR 0012 (yage-docs PR #11) closes the last open design pick (`missingkey=error` confirmed). yage-backend (B) rotated from HELD #125 onto #136 and is now **executing** on `feat/136-manifests-fetcher`. The constructor-shape pick (zero-value `Fetcher` vs `NewFetcher()`) remains a small open decision but is not blocking.
- **#125 unblocked by code dependencies** (#123 ✅, #124 ✅, #144 ✅, #148 ✅ — the bootstrap.go phase reorder by #148 removes the merge-conflict risk on `bootstrap.go`) but **HELD** pending architect review of yage-tofu PR #6 (registry module). The 3 questions raised in session 7 are partly answered: the yage-tofu module now exists in PR #6, but the architect must accept the 5 flagged decisions before yage-side wiring lands.
- **#137–#141** (manifests migration chain) remain gated on #136 merging.

**Architect lane (loaded):** review yage-tofu PR #6 (5 decisions), yage-tofu PR #7 (2 contradictions). yage-docs PR #11 (ADR 0012), PR #12 (ADR 0009 erratum E1), PR #13 (Phase H runbook) all **merged** post-session.

**Next session moves:** (1) architect resolves yage-tofu PR #6/#7 reviews; (2) Programmer merges yage-tofu PRs #6/#7 once accepted (this unblocks #125 Execute); (3) backend (B) lands #136 → unblocks #137–#141 chain.

---

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

**yage PRs open at session 6 close — CI green, awaiting merge:**
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
| yage-tofu PR #6 | `registry-module` (yage-tofu) | **yage-architect** review | `registry/` OpenTofu module — Phase H bootstrap registry; closes yage-tofu#5; 5 decisions flagged in PR body | **Awaiting architect review** — p2 |
| yage-tofu PR #7 | `feat/4-issuing-ca-module` (yage-tofu) | **yage-architect** review | `issuing-ca/` OpenTofu module — online intermediate CA; closes yage-tofu#4; 2 contradictions surfaced in PR body | **Awaiting architect review** — p2 |
| yage-docs PR #11 | `docs/adr-0008-addendum-template-layout` (yage-docs) | merged | ADR 0012 — yage-manifests template layout + data contract; addendum to ADR 0008; closes #136 design pick #2 | **Merged** (`1cdf251`) |
| yage-docs PR #12 | `docs/adr-0009-erratum-kubernetes-backend` (yage-docs) | merged | ADR 0009 erratum E1 — kubernetes backend supersedes local tofu state | **Merged** (`20cb4b9`) |
| yage-docs PR #13 | `docs/operations-phase-h-runbook` (yage-docs) | merged | Phase H bootstrap runbook (registry + issuing CA) | **Merged** (`c1930af`) |
| #136 | `feat/136-manifests-fetcher` (yage) | **yage-backend (B)** | `internal/platform/manifests` Fetcher package (MountRoot-based, no local clone); ADR 0008 step 2 | **In progress** — p2; constructor-shape pick still open (not blocking) |
| #125 | `feat/125-registry-vm` (yage) | **yage-backend** (TBD) | provision bootstrap registry VM via OpenTofu; mirror `EnsureIssuingCA` shape; auto-wire `ImageRegistryMirror` | **HELD** — code unblocked, awaiting architect on yage-tofu PR #6 |
| #137 | TBD | **yage-backend** | migrate `internal/capi/helmvalues/` to yage-manifests templates | **Blocked** on #136 |
| #138 | TBD | **yage-backend** | migrate `internal/capi/wlargocd/` renderers to yage-manifests templates | **Blocked** on #136 |
| #139 | TBD | **yage-backend** | migrate `internal/capi/postsync/` renderers to yage-manifests templates | **Blocked** on #136 |
| #140 | TBD | **yage-backend** | migrate `internal/capi/caaph/` string renderers to yage-manifests templates | **Blocked** on #136 |
| #141 | TBD | **yage-backend** | CSI `RenderValues` → `Render` + port all 14 drivers to yage-manifests (atomic) | **Blocked** on #136 |
| #142 | TBD | **yage-backend** | retire `helmvalues/`, `wlargocd/`, `postsync/` packages | **Blocked** on #137–#141 |
| #119 | TBD | **yage-backend** | D4: CAPD smoke E2E test for bootstrap pipeline | **Backlog** — p3 |
| #94–#101 | TBD | **yage-backend** | PlanDescriber for 8 providers (Linode, CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO) | **Backlog** — p3 (epic #78); parallelizable across multiple backend instances |

---

## HALT Records

- **yage-backend (A) HALT — CLEARED** by PR #166 merge (`bb60c79`, 2026-05-01T11:12:11Z). Agent rolls off; #148 closed.
- **yage-backend (B) HALT — CLEARED**. (B) rotated from HELD #125 onto #136 (now unblocked); executing on `feat/136-manifests-fetcher`.
- **yage-backend (C) HALT — N/A** (rolled into the (B) #136 thread; only one agent on #136).

## Critical Path (in order)

1. **Architect**: review yage-tofu PR #6 (5 decisions — decision #1 already settled by ADR 0009 erratum E1 merged in yage-docs PR #12 `20cb4b9`), yage-tofu PR #7 (2 contradictions). yage-docs PR #11 (ADR 0012, `1cdf251`), PR #12 (`20cb4b9`), PR #13 (`c1930af`) all merged.
2. **Programmer**: merge yage-tofu PRs #6 and #7 once accepted — unblocks #125 Execute.
3. **yage-backend (B)**: land #136 (in progress) — unblocks the #137–#141 manifests migration chain.
4. **yage-backend** (TBD): start #125 once yage-tofu PR #6 is merged and architect direction is clear.
5. **Phase H epic #120** closes once #125 lands AND yage-tofu PRs #6/#7 ship.

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
| 0009 | On-prem platform services (Phase H): registry + issuing CA | Accepted (erratum E1 merged in yage-docs PR #12 `20cb4b9`: kubernetes backend supersedes local tofu state) |
| 0010 | In-cluster repository cache (zero workstation residue) | Accepted |
| 0011 | Pivot: yage state migration to management cluster | Accepted (fully implemented as of PR #166) |
| 0012 | yage-manifests template layout and data contract (addendum to 0008) | Accepted (yage-docs PR #11 merged `1cdf251`) |
