# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. **ADR 0011 (pivot state migration) fully implemented** (PR #166). **Phase H wiring complete on yage side**: EnsureIssuingCA (#163), EnsureRepoSync (#164), EnsureManagementInstall (#160), JobRunner k8s backend (#162), EnsureRegistry (#177). **yage-manifests migration chain (ADR 0008) substantially complete**: Fetcher (#167 / #136), helmvalues (#184 / #137), caaph (#178 / #140), postsync (#192 / #139), CSI Render (#193 / #141). **Only #138 (wlargocd) and #142 (retire packages) remain in the chain.** Phase H epic #120 closed in spirit by #125. Open follow-up: **#168** (migrate inline crypto/x509 in EnsureIssuingCA to JobRunner.Output of yage-tofu/issuing-ca/). **Cross-repo runtime gap discovered (2026-05-03)**: yage default `ManifestsRef=v0.2.0` ships addons templates only; CSI driver dirs at `v0.2.0` contain README.md only — runtime CSI deploys would fail until yage-manifests cuts a release with `csi/<driver>/values.yaml.tmpl` files. yage-manifests PRs #7 and #9 (duplicates of each other) on the `init` branch hold this content; one must be merged then a `v0.3.0` tag cut and yage default bumped.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-03** — PO session 9: 0 open PRs in `lpasquali/yage`. Sessions 8 staleness reconciled: 15 PRs (#170, #173, #176, #177, #178, #180, #182, #183, #184, #185, #187, #188, #190, #192, #193) and 16 issues (#119, #125, #136, #137, #139, #140, #141, #167 noted, #169, #171, #172, #174, #175, #179, #181, #186) closed since session 8. Three new handoffs dispatched: **yage-backend** on #138 (wlargocd → templates) and #168 (EnsureIssuingCA migration); **yage-platform-engineer** on cross-repo CSI templates gap (yage-manifests PRs #7/#9 dedup + cut v0.3.0 + bump yage default).

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | #193 (CSI Render → templates, `8510836`), #192 (postsync templates, `353e1ce`), #190 (mask cost-tab/token secrets, `b0ed230`), #188 (Deploy fix, `83a13a7`), #187 (ManifestsRef → v0.2.0, `6aeb498`), #185 (dashboard split, `d3f19e4`), #184 (helmvalues + caaph converge, `d97fb77`), #183 (Linode PlanDescriber, `f410d76`), #182 (CI go test, `cb398c4`), #180 (post-pivot phase-order test, `9f4273e`), #178 (caaph templates, `f5ae1cd`), #177 (EnsureRegistry, `f895c7b`), #176 (UI nav fix, `26729f9`), #173 (mask secrets, `7cbe73c`), #170 (wrapper structs, `ff9c766`), #167 (Fetcher, `ca3562f`), #166 (label-based handoff, `bb60c79`) | **0 open PRs**; #138 + #168 dispatched fresh this session |
| yage-docs | `main` | PR #13 (Phase H runbook), PR #12 (ADR 0009 E1), PR #11 (ADR 0012) | 0 open PRs |
| yage-tofu | `main` | PRs #6 (registry), #7 (issuing-ca), #2 (k8s backend), #3 (license) — all merged | 0 open PRs |
| yage-manifests | `init` (default) + tag `v0.2.0` | PR #4 (addons templates → v0.2.0 cut); **PRs #7 + #9 OPEN — duplicates** carrying CSI `values.yaml.tmpl` files for all 14 drivers | **Runtime gap**: `v0.2.0` lacks CSI templates → CSI deploys would fail; PE handoff issued |

## Recent Changes

### 2026-05-03 — PO session 9: 0 open PRs in yage; staleness reconciled; 3 fresh handoffs

**State at session start:** `gh pr list` (yage) returned `[]`. Sister repos: yage-docs 0 open, yage-tofu 0 open, yage-manifests **2 open** (#7, #9 — duplicates carrying CSI templates for `init` branch).

**Session 8 → 9 staleness reconcile.** The following 15 yage PRs merged between session 8 close and session 9 start; the table in session 8 still listed several of these issues as "in progress / blocked":

| PR | Issue closed | Title |
|---|---|---|
| #167 | (#136 follow-on landed earlier) | feat(manifests): implement internal/platform/manifests Fetcher (ADR 0008/0010/0012) |
| #170 | #169 | feat(templates): add ADR 0012 §3 wrapper structs |
| #173 | #171, #172 | fix(ui): mask all secret fields in xapiri (security) |
| #176 | #174 | fix(ui): restore arrow up/down field navigation |
| #177 | #125 | feat(opentofux): EnsureRegistry — bootstrap registry VM via yage-tofu module |
| #178 | #140 | feat(caaph): migrate to yage-manifests templates |
| #180 | #119 | test(orchestrator): add post-pivot phase-order regression test |
| #182 | #181 | ci: run go test -race -short ./... in quality-gates |
| #183 | #94 | feat(provider/linode): implement PlanDescriber |
| #184 | #137 | feat(manifests): add Fetcher.RegisterFunc + migrate helmvalues + converge caaph |
| #185 | #179 | refactor(xapiri): split dashboard.go per ADR 0014 |
| #187 | #189 | chore(config): bump default ManifestsRef to v0.2.0 |
| #188 | #186 | fix(xapiri): Deploy action no longer exits the app on on-prem mode |
| #190 | #175 | fix(xapiri/security): mask cost-tab + token-overlay secret length per ADR 0013 |
| #192 | #139 | feat(postsync): migrate to yage-manifests _partials templates |
| #193 | #141 | feat(csi): migrate Driver.RenderValues → Render + yage-manifests templates |

Net effect on the migration chain (issue #133 epic): **#136, #137, #139, #140, #141 all merged**. Only **#138** (wlargocd → templates) and **#142** (retire helmvalues/wlargocd/postsync packages, gated on #138) remain.

**Cross-repo runtime gap discovered.** PR #187 set the yage default `ManifestsRef` to `v0.2.0`. Verified at session 9 start:
- `yage-manifests v0.2.0` (annotated tag, sha `4607ee1`, msg "first release with real template content") ships **only addons templates**: `argocd-apps/`, `argocd/`, `cilium/`, `keycloak/`, `metrics-server/`, `observability/` (grafana + victoria-metrics), `opentelemetry/`, `spire/`.
- `yage-manifests v0.2.0` `csi/<driver>/` directories all contain **README.md only**, no `values.yaml.tmpl`.
- yage PR #193 (`8510836`) lands the `Render(template, data) (string, error)` migration but its tests use yage-side `internal/csi/<driver>/testdata/` fixtures, so green CI does not exercise the remote fetch path.
- Result: an operator running yage `main` with default `ManifestsRef=v0.2.0` and any non-Proxmox CSI driver will hit a Fetcher render error at runtime (template file not present in the materialised mount).
- yage-manifests has **two duplicate open PRs** (#7 on `feat/141-csi-templates` and #9 on `feat/141-csi-values-templates`) that both add the missing `csi/<driver>/values.yaml.tmpl` for all 14 drivers + rename `csi/hcloud/` → `csi/hcloud-csi/`. PR #9 is CI-green; PR #7 has `mergeable: UNKNOWN`. They contain identical content.

**Fresh handoffs dispatched this session:**

1. **yage-backend → #138** — port `internal/capi/wlargocd/render.go` (409 lines: `HelmGit`, `HelmRegistry`, PostSync conditional `sources:` array variants) to `addons/argocd/*.yaml.tmpl` in yage-manifests. Branch suggestion: `feat/138-wlargocd-templates`. Pattern reference: PR #184 (helmvalues) + PR #178 (caaph) — same migration shape. May need `template.FuncMap` for `shellQuoteEscape`. Closes #138; unblocks #142 retirement.

2. **yage-backend → #168** — replace inline `crypto/x509` `generateIntermediateCA` in `internal/platform/opentofux/issuing_ca.go` with `JobRunner` invocation against `yage-tofu/issuing-ca/` module (now merged on yage-tofu main). Read outputs `intermediate_cert_pem`, `intermediate_key_pem`, `ca_chain_pem` via `JobRunner.Output`. Module exists at yage-tofu (PR #7 merged). Issue body has a complete plan and acceptance criteria. Branch suggestion: `feat/168-issuing-ca-jobrunner`.

3. **yage-platform-engineer → cross-repo CSI templates gap** — (a) on `lpasquali/yage-manifests`, pick PR #9 (CI green) as canonical, close PR #7 as duplicate, ensure `csi/hcloud/` → `csi/hcloud-csi/` rename is in the merge (matches Driver `Name()`); (b) merge PR #9 to `init`; (c) cut tag `v0.3.0` with annotated message listing CSI driver coverage; (d) open follow-up yage issue to bump default `ManifestsRef` from `v0.2.0` → `v0.3.0` and hand the bump back to yage-backend. This is sequenced — must complete before any operator-facing CSI deploy from `main` will work.

**Not dispatched this session (left in backlog with rationale):**
- **#142** (retire helmvalues/wlargocd/postsync packages) — gated on #138; will dispatch once #138 merges.
- **#191** (xapiri teatest integration suite, p3) — gated on #175 per its body; #175 is closed by #190, so technically unblocked. Holding for explicit user dispatch since it's p3 and frontend lane has no in-flight item — flag for next session.
- **#95–#101** (7 PlanDescribers, p3 backlog, parallelizable) — only Linode (#94) shipped this session via PR #183. Hold until user signals priority.

---

### 2026-05-03 — Architect handoff: ADR 0016 (xapiri UI simulation harness)

**ADR drafted:** `docs/architecture/adrs/0016-xapiri-ui-simulation-harness.md`
on yage-docs branch `docs/0016-ui-sim-harness` (PR pending).

**Position vs ADR 0015 / issue #191.** ADR 0015 §"#191" already enumerates the
per-tab teatest classes and pins `//go:build integration` + `make test-integration`.
ADR 0016 **refines** that section: it specifies the harness *contract* (three-layer
file structure, deterministic seams, assertion surface, cost-tab two-lane testing).
ADR 0015 stays the source of truth for *which* test classes exist; ADR 0016 is the
source of truth for *how* each class is built. Issue #191 stays open as the
umbrella implementation issue and gains a "see ADR 0016 for harness contract"
note in its body.

**Implementation issues to open (PO action — architect does not open issues).**
Frontend/backend lanes; suggest opening as an epic (#191 stays as the umbrella)
with the children below. All gated on the pricing-fetcher seam landing first.

| Suggested # | Lane | Title | Notes |
|---|---|---|---|
| (new) | yage-backend | `pricing.Fetcher` interface + context plumbing; migrate `Provider.EstimateMonthlyCostUSD(ctx, *cfg)` for all 12 providers | **p2** — gates the harness; breaking signature change; mechanical across providers + `internal/cost/`. Triggers `govulncheck` audit row. |
| (new) | yage-frontend | `internal/ui/xapiri/harness_test.go` skeleton (Layer Low) — wraps teatest, `Send`/`Type`/`Key`/`Frame`/`Model`/`Quit`/`AssertNoSecretLeak` | **p2** — adds `github.com/charmbracelet/x/exp/teatest` to `go.mod` (+ `govulncheck`). Pin `WithInitialTermSize(120,40)`. |
| (new) | yage-frontend | `internal/ui/xapiri/harness_dsl_test.go` (Layer Mid) — `Tick`, `Edit`, `NavDown/Up`, `NextTab/PrevTab/OpenTab`, `StepTimeframeForward/Back`, `SetTimeframeIdx`, `EstimateMonthlyToDuration` | **p3** — depends on Layer Low. |
| (new) | yage-frontend | Optional `var Clock = time.Now` package seam in `dashboard.go` for log-ring + sysStatsTickCmd timestamp pinning | **p4** — only if scenarios need wall-clock pinning; cost math does not. |
| (new) | yage-frontend | Golden-frame fixtures for initial render of every tab (`testdata/golden/<tab>_initial.golden`) | **p3** — enabled by Layer Low; one fixture per tab from ADR 0015 §"#191" table. |
| (new) | yage-frontend | Cost-tab arbitrary-timeframe scenarios (Lane A math + Lane B keypress; both `[`/`]` and direct `costForPeriod`) | **p3** — addresses the user's "estimate any timeframe" requirement explicitly. |
| (new) | yage-frontend | Per-tab acceptance scenarios (one per ADR 0015 §"#191" table row) | **p3** — N issues, one per tab, parallelizable across the frontend lane. |
| #191 | yage-frontend | Umbrella: keep open as the integration-suite tracker; closed when all children land | **p3** — body to be updated to reference ADR 0016. |

**Open design questions to resolve before backend picks up the pricing seam:**

1. **Pricing seam shape — context-scoped `Fetcher` (recommended) vs per-provider
   `SetPricingFetcher` setter.** ADR 0016 picks context-scoped because it
   composes across parallel scenarios; if backend prefers the setter form for
   delta-minimisation reasons, ADR 0016 needs a one-line erratum before the
   first PR.
2. **Custom-timeframe in the UI (open-ended duration field).** ADR 0016 marks
   this **out of scope** — Lane A already enables math testing for any
   duration. If product wants this as a real feature, open a separate
   feature issue; the harness will not need extension until then.
3. **Issue #191 disposition.** ADR 0016 recommends keeping #191 open as the
   umbrella. Alternative is closing #191 as superseded and re-opening one
   per-tab issue. PO call.

**Discrepancy with the original brief (logged for the record).** The brief
described the tab-switch shortcut as `Ctrl+Alt+Left/Right`. The actual binding
in `internal/ui/xapiri/dashboard.go:888` is `Ctrl+Left/Right`. PR #156 removed
`Ctrl+Alt+<Number>` shortcuts but the arrow cycle was never gated on `Alt`.
ADR 0016 documents the actual binding so issue bodies opened from this handoff
do not propagate the wrong shortcut.

**No contradictions discovered with PRs #190 / #185 / #176 / #156.** PR #185
(dashboard split) gives ADR 0016 the per-file structure it relies on. PR #176
fixed the arrow-nav regression that ADR 0016 mandates be guarded by the
`AssertNoSecretLeak` + per-tab scenario class. PR #156's `[` / `]` dispatch
fix is the exact regression Lane B is designed to detect.

---

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
| #138 | `feat/138-wlargocd-templates` (yage) | **yage-backend** | migrate `internal/capi/wlargocd/render.go` (409 lines: `HelmGit`, `HelmRegistry`, PostSync `sources:` variants) to `addons/argocd/*.yaml.tmpl` in yage-manifests. Pattern: PR #184 (helmvalues) + PR #178 (caaph). May need `template.FuncMap` for `shellQuoteEscape`. Closes #138; unblocks #142. | **Dispatched** — p2 |
| #168 | `feat/168-issuing-ca-jobrunner` (yage) | **yage-backend** | replace inline `crypto/x509` in `internal/platform/opentofux/issuing_ca.go` with `JobRunner` against yage-tofu `issuing-ca/` module (now on main). Read `intermediate_cert_pem` / `intermediate_key_pem` / `ca_chain_pem` via `JobRunner.Output`. Plan + acceptance criteria in issue body. | **Dispatched** — p3 |
| yage-manifests PR #7, #9 + retag | `feat/141-csi-templates` + `feat/141-csi-values-templates` (yage-manifests) | **yage-platform-engineer** | (a) pick PR #9 as canonical (CI green), close #7 as duplicate; (b) merge to `init`; (c) cut tag `v0.3.0`; (d) open yage follow-up to bump default `ManifestsRef v0.2.0 → v0.3.0`. Closes runtime CSI gap. | **Dispatched** — p1 (operator-blocking) |
| #142 | TBD | **yage-backend** | retire `internal/capi/helmvalues/`, `wlargocd/`, `postsync/` packages once callers migrated. | **Queued** — gated on #138 |
| #191 | TBD | **yage-frontend** | xapiri teatest integration suite for all tabs and cross-cutting surfaces (ADR 0015 §"#191" defines class table; ADR 0016 defines harness contract) | **Backlog** — p3; #175 precondition met by #190; ADR 0016 dispatched 2026-05-03 — see "Architect handoff" entry below for the implementation-issue list to open |
| #95–#101 | TBD | **yage-backend** | PlanDescriber for 7 remaining providers (CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO). Linode (#94) shipped in #183. | **Backlog** — p3 (epic #78); parallelizable |

---

## HALT Records

- **No active HALTs.** All session-8 backend HALTs are cleared by their merged PRs (#148 → PR #166; #125 → PR #177; #136 → PR #167).
- **#138** and **#168** were dispatched without an Execute halt because each has a complete plan in its issue body and follows established migration patterns from already-merged PRs (#184 / #178 for #138; the EnsureIssuingCA Step 2/3 contract is unchanged for #168). The receiving agent should still pause at SOP Step 4 (Halt) for any deviation from the issue plan.
- **yage-manifests PR #7/#9 dedup + retag** was dispatched to platform-engineer with the explicit instruction to halt before destroying PR #7 and confirm with user that #9 is the chosen canonical.

## Critical Path (in order)

1. **yage-platform-engineer**: dedup yage-manifests PRs #7/#9, merge canonical, cut `v0.3.0`. **p1 operator gap** — runtime CSI deploys don't work until this lands.
2. **yage-backend → #138**: complete the ADR 0008 migration chain.
3. **yage-backend → #168**: clean up the EnsureIssuingCA stopgap; brings the Phase H Go-side fully aligned with ADR 0009 §3.
4. **yage-backend → #142**: retire packages once #138 lands.
5. After (1) lands: bump yage default `ManifestsRef` to `v0.3.0` (small follow-up issue PE will open).

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
| 0013 | TUI secret-display policy | Accepted |
| 0014 | xapiri dashboard.go split | Proposed (mechanical refactor landed via PR #185) |
| 0015 | Test coverage strategy and targets | Proposed |
| 0016 | xapiri UI simulation harness (deterministic teatest contract) | Proposed (yage-docs branch `docs/0016-ui-sim-harness`) |
