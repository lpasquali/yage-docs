# CURRENT_STATE

## Living Memory

yage is in active development. The core bootstrap pipeline is functional for Proxmox. The xapiri TUI is being built out as the primary configuration interface. The provider abstraction plan is fully complete: phases A, B, C, D, and E are all done. Legacy cleanup is complete per ADR 0002. ADR 0007: dashboard is the new default xapiri entry point. CSI driver registry (ADR 0001) complete — all 10 drivers merged. **ADR 0011 (pivot state migration) fully implemented** (PR #166). **Phase H wiring complete on yage side**: EnsureIssuingCA (#163), EnsureRepoSync (#164), EnsureManagementInstall (#160), JobRunner k8s backend (#162), EnsureRegistry (#177), and as of session 11 EnsureIssuingCA Go-side migrated to `JobRunner` against yage-tofu/issuing-ca/ (PR #195 closed #168). **yage-manifests migration chain (ADR 0008)**: Fetcher (#167 / #136), helmvalues (#184 / #137), caaph (#178 / #140), postsync (#192 / #139), CSI Render (#193 / #141) all merged; **#138 (wlargocd) staged in yage PR #199 + yage-manifests PR #10 — both held this session**. yage-manifests **v0.3.0** is tagged and ships CSI templates; **v0.4.0** (wlargocd templates) blocked on yage-manifests PR #10 lint failure (`RuneGate/Quality/Templates` rejects multiline `{{- /* */ -}}` comments). yage default `ManifestsRef` is still `v0.2.0` at HEAD `afa05ab` — bump to v0.3.0 (or v0.4.0 once available) tracked as #194.

## Freshness Policy

This file must be updated whenever system state evolves (per CODING_STANDARDS.md "Atomic Persistence"). If information here conflicts with what you observe in the code or git history, trust what you observe now — then update this file to match reality.

Last updated: **2026-05-03** — PO session 11: merged yage **#195** (EnsureIssuingCA → JobRunner, closes #168) and yage-docs **#20** (ADR 0016 + session 10 record). yage-manifests **#10** (wlargocd templates) **BLOCKED** on `RuneGate/Quality/Templates` lint regex rejecting multiline `{{- /* ... */ -}}` comments — yage **#199** (wlargocd consumer) held until #10 lands. **v0.4.0 NOT cut.** PE handoff issued for the lint-fix.

Session 9 (earlier today): 0 open PRs in `lpasquali/yage`. Sessions 8 staleness reconciled: 15 PRs (#170, #173, #176, #177, #178, #180, #182, #183, #184, #185, #187, #188, #190, #192, #193) and 16 issues (#119, #125, #136, #137, #139, #140, #141, #167 noted, #169, #171, #172, #174, #175, #179, #181, #186) closed since session 8. Three handoffs dispatched: **yage-backend** on #138 (wlargocd → templates) and #168 (EnsureIssuingCA migration); **yage-platform-engineer** on cross-repo CSI templates gap (yage-manifests PRs #7/#9 dedup + cut v0.3.0 + bump yage default).

## Version Baseline

| Repo | Branch | Recent PRs | Status |
|---|---|---|---|
| yage | `main` | **#195** (EnsureIssuingCA → JobRunner, `afa05ab`), #193 (CSI Render → templates, `8510836`), #192 (postsync templates, `353e1ce`), #190 (mask cost-tab/token secrets, `b0ed230`), #188 (Deploy fix, `83a13a7`), #187 (ManifestsRef → v0.2.0, `6aeb498`), #185, #184, #183, #182, #180, #178, #177, #176, #173, #170, #167, #166 | **1 open PR** (#199, held on yage-manifests #10) |
| yage-docs | `main` | **PR #20** (ADR 0016 + PO session 10, `dea8c84`), PR #19 (ADR 0015), PR #13 (Phase H runbook), PR #12 (ADR 0009 E1), PR #11 (ADR 0012) | 0 open PRs |
| yage-tofu | `main` | PRs #6 (registry), #7 (issuing-ca), #2 (k8s backend), #3 (license) — all merged | 0 open PRs |
| yage-manifests | `init` (default) + tag `v0.3.0` | PR #4 (addons templates → v0.2.0); CSI templates → v0.3.0 cut; **PR #10 OPEN — wlargocd `application.yaml.tmpl` for 14 add-ons, BLOCKED on `RuneGate/Quality/Templates` lint** | **v0.4.0 not cut**; PE handoff issued for the lint regex |

## Recent Changes

### 2026-05-03 — PO session 11: 2 of 4 PRs merged; yage-manifests #10 blocked on lint regex

**Merge events this session:**

| Repo | PR | Title | Merge sha | Closed issues |
|---|---|---|---|---|
| yage | **#195** | feat(opentofux): EnsureIssuingCA — JobRunner against yage-tofu/issuing-ca/ | `afa05ab` | **#168** |
| yage-docs | **#20** | docs(adr): ADR 0016 — xapiri UI simulation harness contract (+ PO session 10 record) | `dea8c84` | (none — docs-only) |

**PRs NOT merged (blocked):**

| Repo | PR | Reason |
|---|---|---|
| yage-manifests | **#10** | `RuneGate/Quality/Templates` fails — CI grep `{{[^}]*$` flags every multiline `{{- /* ... */ -}}` comment header in the 14 addon templates. Real lint failure (whether the regex is too strict or templates need single-line comments is an implementation call). Per workflow safety rules, STOPPED — no fix attempted by PO. See run https://github.com/lpasquali/yage-manifests/actions/runs/25272223243/job/74096414179. |
| yage | **#199** | Logically gated on #10 + v0.4.0 tag (the migrated wlargocd code consumes templates that must exist at the default `ManifestsRef`). Branch is up to date with main and CI is green; held until #10 lands and yage-manifests v0.4.0 is cut. |

**v0.4.0 tag NOT cut** — gated on #10 merge.

**yage #194 (`chore(config): bump default ManifestsRef`)** — left targeting **v0.3.0** as originally filed (v0.4.0 does not exist yet). Added a PO note comment explaining the v0.4.0 sequencing question for the implementer to resolve.

**Cross-repo ordering constraint for the next session.** Before merging yage #199 in any future session, **yage-manifests #10 must land AND v0.4.0 must be tagged AND yage default `ManifestsRef` must be bumped to v0.4.0**. Otherwise wlargocd in yage `main` will break at runtime against the operator-default ref. This is the same trap that #194/#187/v0.2.0 fell into for CSI templates.

**Issues auto-closed this session:**

- yage **#168** — closed by PR #195.

**Active Work table updates:** removed the `feat/168-issuing-ca-jobrunner` and `docs/0016-ui-sim-harness` rows (merged); added the yage-manifests #10 lint-fix row (PE lane); kept #199 row as "held pending #10".

---

### 2026-05-03 — PO session 10: ADR 0016 user decisions; 3 fresh issues opened (#196, #197, #198)

**User decisions on ADR 0016 open questions** (see "Architect handoff: ADR 0016" entry below — Q1/Q2/Q3 now resolved inline):

1. **Pricing seam shape** → **Fetcher** (context-scoped). Matches the ADR's recommended option, so **no erratum needed**. Dispatched as new issue **#197** to **yage-backend** (see Active Work).
2. **"Custom timeframe" UI control** → **out of scope confirmed**. The user added: "what we have is not working as of now" — meaning the *existing* timeframe stepping in the cost tab is broken at runtime (separate bug from any new "custom" control). Filed as new bug **#196** (xapiri/cost, p2). PR #156 was supposed to have fixed this; either regressed or incomplete coverage. This bug is the **Lane B regression target** explicitly named in ADR 0016 §4 — the harness suite, when built, must guard against it.
3. **Issue #191 disposition** → **keep open as the umbrella** for the integration-suite tracker, with body to be updated to reference ADR 0016 (note: body update has not yet been committed to GitHub — defer to whoever picks up #191 next).

**Discrepancy NOT resolved this session.** The architect flagged that the user's brief described the tab-cycle shortcut as `Ctrl+Alt+Left/Right` but the actual binding at `internal/ui/xapiri/dashboard.go:888` is `Ctrl+Left/Right` (PR #156 removed `Ctrl+Alt+<Number>` shortcuts but the arrow cycle was never `Alt`-gated). The user did not explicitly confirm preferred binding. Filed as low-priority question issue **#198** (`type: question`, p3); no binding change made.

**Issues opened this session:**

| # | Type | Title | Lane | Priority |
|---|---|---|---|---|
| **#196** | bug | bug(xapiri/cost): timeframe stepping broken at runtime | yage-frontend | p2 |
| **#197** | feat | feat(pricing): introduce context-scoped pricing.Fetcher interface (gates ADR 0016 harness) | yage-backend | p2 |
| **#198** | question | question(xapiri): should tab cycle binding be Ctrl+Arrow or Ctrl+Alt+Arrow? | (n/a) | p3 |

**Not dispatched this session (gated on #197 landing first).** The remaining ADR 0016 frontend work — harness skeleton (Layer Low), DSL (Layer Mid), optional `Clock` seam, golden-frame fixtures, cost-tab arbitrary-timeframe scenarios, per-tab acceptance scenarios — all wait for the pricing.Fetcher interface to merge. Once #197 ships, PO session 11 dispatches the frontend stack as a parallel batch.

**No PR merge events this session** (yage `main` HEAD remains `8510836`; same as session 9 close).

---

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

**Open design questions — RESOLVED 2026-05-03 (PO session 10):**

1. **Pricing seam shape** — **RESOLVED: Fetcher (context-scoped).** User
   confirmed the ADR's recommended option. **No erratum needed.** Dispatched
   to yage-backend as issue **#197**.
2. **Custom-timeframe in the UI** — **RESOLVED: out of scope confirmed.** User
   added the rider "what we have is not working as of now" — i.e. the
   *existing* `[`/`]` stepping is broken at runtime, separate bug. Filed as
   **#196** (xapiri/cost, p2). The harness, when built, must include this
   regression as its Lane B canary.
3. **Issue #191 disposition** — **RESOLVED: keep open as umbrella.** Body to
   be updated to reference ADR 0016 when #191 is next worked.

**Discrepancy with the original brief** (Ctrl+Arrow vs Ctrl+Alt+Arrow tab
cycle) — **NOT resolved.** User did not explicitly confirm preferred binding.
Filed as **#198** (question, p3). No binding change made.

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
| yage #199 + yage-manifests #10 | `feat/138-wlargocd-templates` (both repos) | **yage-backend** (consumer side) / **yage-platform-engineer** (lint-fix side) | yage #199 (`afa05ab` parent) consumes addon templates added in yage-manifests #10. yage #199 is **CI-green and held**; yage-manifests #10 is **CI-blocked** on `RuneGate/Quality/Templates` rejecting multiline `{{- /* ... */ -}}` template comments. Lint-fix dispatched as new PE task (see below). Closes #138; unblocks #142. | **Held** — p2; #199 cannot merge until #10 + v0.4.0 land |
| (new — no issue#) | `chore/manifests-template-lint-fix` (yage-manifests) | **yage-platform-engineer** | yage-manifests `RuneGate/Quality/Templates` step uses `! grep -rn '{{[^}]*$' --include='*.yaml.tmpl' .` which flags every multiline `{{- /* ... */ -}}` comment header. Choose: (a) relax the regex to allow `{{` followed by `/*` or `-` line-continuation, OR (b) collapse the 14 addon template comment headers to a single line. (a) is preferred — it's a CI-only change and doesn't touch templates that already work. After fix, retrigger CI on PR #10, merge, then cut `v0.4.0` annotated tag listing the 14 addon templates added. | **Ready to dispatch** — p1 (gates yage #199, #194 v0.4.0 path, and #142 retirement) |
| #194 | TBD | **yage-backend** | bump default `ManifestsRef` from `v0.2.0` → **v0.3.0** in `internal/config/config.go` (CSI templates already shipped at v0.3.0). Implementer should consider whether to chain a follow-up bump to v0.4.0 once yage-manifests #10 lands and v0.4.0 is cut, OR hold this issue and bump straight to v0.4.0. PO note added to issue 2026-05-03. | **Ready to dispatch** — p1 (gated unblocked at v0.3.0; v0.4.0 path blocked on yage-manifests #10) |
| #142 | TBD | **yage-backend** | retire `internal/capi/helmvalues/`, `wlargocd/`, `postsync/` packages once callers migrated. | **Queued** — gated on yage #199 merge (which is gated on yage-manifests #10) |
| #196 | TBD | **yage-frontend** | bug — costs-tab `[`/`]` timeframe stepping broken at runtime (PR #156 regressed/incomplete). User-reported 2026-05-03. Lane B regression target for ADR 0016 harness. | **Dispatched** — p2 |
| #197 | TBD | **yage-backend** | introduce context-scoped `pricing.Fetcher` interface + `WithFetcher`/`FetcherFrom`; migrate `Provider.EstimateMonthlyCostUSD(ctx, *cfg)` for all 12 providers + `internal/cost/` callers. **Gates ADR 0016 harness work** — frontend stack waits on this. | **Dispatched** — p2 |
| #198 | (n/a) | (question) | tab-cycle binding ambiguity: Ctrl+Arrow (current) vs Ctrl+Alt+Arrow (per brief). Awaiting user confirmation; no binding change pending. | **Open question** — p3 |
| #191 | TBD | **yage-frontend** | xapiri teatest integration suite for all tabs and cross-cutting surfaces (ADR 0015 §"#191" defines class table; ADR 0016 defines harness contract) | **Backlog** — p3; #175 precondition met by #190; ADR 0016 dispatched 2026-05-03; **gated on #197 landing** before harness skeleton + DSL + scenarios can be opened as child issues |
| #95–#101 | TBD | **yage-backend** | PlanDescriber for 7 remaining providers (CAPD, OpenStack, vSphere, Azure, IBM, GCP, DO). Linode (#94) shipped in #183. | **Backlog** — p3 (epic #78); parallelizable |

---

## HALT Records

- **No active HALTs.** All session-8 backend HALTs are cleared by their merged PRs (#148 → PR #166; #125 → PR #177; #136 → PR #167). Session-9 dispatches landed: #168 → PR #195 (merged session 11, sha `afa05ab`); CSI gap → resolved by yage-manifests v0.3.0 cut.
- **Session 11 STOP**: yage-manifests PR #10 was NOT merged — `RuneGate/Quality/Templates` lint failure (multiline `{{- /* */ -}}` template comment headers tripping a too-strict grep regex). Per workflow safety rules, no fix attempted by PO; PE handoff issued.
- **Session 11 HOLD**: yage PR #199 not merged despite green CI, to preserve the cross-repo invariant that the wlargocd consumer cannot land in main while the templates it consumes are absent at the operator-default `ManifestsRef`.

## Critical Path (in order)

1. **yage-platform-engineer**: fix yage-manifests `RuneGate/Quality/Templates` lint regex (or collapse template comment headers); merge yage-manifests #10; cut `v0.4.0`. **p1 — gates the wlargocd consumer (yage #199) and the operator-default ref bump path.**
2. **yage-backend → yage #199**: merge once #10 + v0.4.0 land. Closes #138; unblocks #142.
3. **yage-backend → #194**: bump default `ManifestsRef` (v0.3.0 immediately, or hold for v0.4.0 once available — see PO note on issue).
4. **yage-backend → #197**: pricing.Fetcher interface — gates the ADR 0016 frontend harness stack (#191 children).
5. **yage-backend → #142**: retire helmvalues/wlargocd/postsync packages once yage #199 lands.
6. **yage-frontend → #196**: timeframe stepping bug fix (independent of above).

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
