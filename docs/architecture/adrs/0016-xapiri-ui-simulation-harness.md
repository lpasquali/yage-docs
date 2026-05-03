# ADR 0016 — xapiri UI Simulation Harness (deterministic teatest contract)

**Status:** Proposed
**Date:** 2026-05-03
**Author:** Architect agent
**Refines:** ADR 0015 §"#191" (test-class enumeration). This ADR specifies the
  *harness contract*; ADR 0015 specifies the *coverage targets*. They are
  complementary and must be read together.
**Relates to:** ADR 0013 (TUI Secret-Display Policy), ADR 0014 (xapiri Dashboard
  Split), ADR 0003 (TUI Dispatch Keymap), yage issue #191, PR #176 (arrow nav),
  PR #156 (timeframe `[`/`]` keys + ctrl+alt+number removal),
  PR #185 (dashboard split), PR #190 (secret masking).

---

## Context

ADR 0015 defines *what* xapiri test classes must exist (the per-tab table at
§"#191") and the CI shape (`//go:build integration`, `make test-integration`).
It does **not** specify *how* tests synthesise input, capture output, pin
non-deterministic dependencies, or assert on the model surface. As a result,
yage issue #191 is "ready to execute" in scope but "ambiguous in execution":
two reasonable engineers would write structurally incompatible suites.

This ADR pins the harness contract so the suite is uniform across tabs and
new test classes can be added by following the same pattern.

The trigger for writing it now is the user's brief enumerating every input
mechanism that must be drivable from tests:

- checkbox toggles (Huh `Confirm` / boolean fields),
- arbitrary text edit on every field (insert, edit, backspace, masked secrets
  per ADR 0013),
- arrow up/down field navigation (regression-prone, see PR #176),
- tab cycling (the current binding is **`Ctrl+Left/Right`**, not
  `Ctrl+Alt+Left/Right` as the brief paraphrased; PR #156 removed
  `Ctrl+Alt+<Number>` shortcuts but left the arrow cycle untouched at
  `dashboard.go:888`),
- cost-tab timeframe estimation across arbitrary durations (1h, 6h, 1d, 1w,
  1M, 1y, …) — both via the keypress UX (`[` / `]`) and via direct math.

---

## Decision

### 1. Harness layering (three layers, all under `//go:build integration`)

| Layer | File (proposed) | Responsibility |
|---|---|---|
| **Low** | `internal/ui/xapiri/harness_test.go` | Wraps `github.com/charmbracelet/x/exp/teatest`. Exposes `New(t, opts...)` returning a `*Harness` with: `Send(tea.Msg)`, `Type(string)`, `Key(tea.KeyType)`, `Frame() string` (last rendered output, ANSI-stripped + raw variants), `Model() dashModel` (snapshot copy), `Quit()`. Pins `tea.WithoutSignals` and `teatest.WithInitialTermSize(120, 40)`. |
| **Mid** | `internal/ui/xapiri/harness_dsl_test.go` | User-story helpers built on Low: `Tick(toggleID)`, `Edit(textInputID, value)`, `Clear(textInputID)`, `NavDown(n)`, `NavUp(n)`, `NextTab()`, `PrevTab()`, `OpenTab(dashTab)`, `StepTimeframeForward()`, `StepTimeframeBack()`, `SetTimeframeIdx(int)`, `EstimateMonthlyToDuration(monthly float64, d time.Duration) float64` (pure math via `costForPeriod` against a synthetic `costWindowPreset{d: d}`). |
| **High** | `internal/ui/xapiri/scenarios_test.go` | End-to-end walkthroughs (`TestScenario_FullProvisionWalkthrough`, `TestScenario_CostsTimeframeAcrossAllPresets`, `TestScenario_TokenOverlayDoesNotEatArrows`, …). One scenario per per-tab class enumerated in ADR 0015 §"#191". |

All three files use the `//go:build integration` tag so the existing
`go test ./...` invocation under `quality-gates.yml` does not pay the
PTY/teatest startup cost. The `make test-integration` target defined in
ADR 0015 invokes the build-tagged set; this ADR does **not** introduce a
new make target.

### 2. Determinism seams

The harness must remove every uncontrolled input.

#### 2.1 Pricing fetcher — **interface + context plumbing**

`internal/pricing` currently exposes `SetCredentials(Credentials)` as a
process-global write (`pricing/credentials.go:63`). Provider
`EstimateMonthlyCostUSD(*config.Config)` then reads from that global. This
is the determinism blocker: parallel scenarios race the global, and a frozen
catalog cannot be installed without monkey-patching.

The decision: introduce a `pricing.Fetcher` interface and a
context-scoped helper.

```go
package pricing

type Fetcher interface {
    Fetch(ctx context.Context, vendor, region, sku string) (USDPerHour float64, err error)
    // Other catalog methods as needed; one per current vendor entry point.
}

func WithFetcher(ctx context.Context, f Fetcher) context.Context
func FetcherFrom(ctx context.Context) Fetcher  // returns DefaultFetcher when absent
```

Provider `EstimateMonthlyCostUSD` is migrated from
`(cfg *config.Config) (Estimate, error)` to
`(ctx context.Context, cfg *config.Config) (Estimate, error)` and reads
the fetcher from `ctx`. Tests pin a `pricing.StaticFetcher` (a map) on
the context they pass to `cost.StreamWithRegions`.

The existing `SetCredentials` global remains — it carries credentials, not
catalog data, and is orthogonal. Tests that need credential-free runs simply
do not call it.

This is a **breaking change to `Provider.EstimateMonthlyCostUSD`** and is
flagged as such in §Consequences.

#### 2.2 Clock seam — minimal, package-level

The cost-tab math in `dashboard_cost_helpers.go` does **not** call
`time.Now`; the period scaling is `monthly * w.d.Seconds() / costMonthSecs`
with `costMonthSecs = 30*24*3600` constant. No clock seam is needed for
cost math.

A `Clock` seam **is** needed for tabs that read wall clock (sysStatsTickCmd
heartbeat, log-ring timestamps). The minimal shape is a package-level
`var Clock func() time.Time = time.Now` in `dashboard.go`; tests
override it inside `harness_test.go`'s `New(t)` with `t.Cleanup` restore.
No interface is introduced — `time.Time` is sufficient.

#### 2.3 Other globals that must be addressed by the harness

- **`tea.Program` defaults** — pin via `teatest.WithInitialTermSize(120, 40)`
  so frame snapshots are reproducible.
- **`os.Getenv("SHELL")`** in the `Ctrl+T` PTY path — out of scope; the
  harness must not exercise the PTY path. `tab_costs`, `tab_provision`, etc.
  do not require it.
- **`globalLogRing`** — already a package-level `*logring.Ring`; tests reset
  it in `New(t)`'s setup.

### 3. Assertion surface

Tests assert at three levels:

1. **Frame golden compare** — `harness.Frame()` returns the last rendered
   string. Use `github.com/charmbracelet/x/exp/golden` (already in
   `go.sum` transitively per `go.sum:charmbracelet/x/exp/golden`) for
   `golden.RequireEqual(t, []byte(frame))`. Per-tab initial-render goldens
   live under `internal/ui/xapiri/testdata/golden/<tab>_initial.golden`.
2. **Model state inspection** — `harness.Model()` returns a copy of
   `dashModel`. Tests assert `m.activeTab == tabCosts`, `m.costPeriodIdx == 2`,
   `m.cfg.Cost.Credentials.AWSAccessKeyID == ""`, etc.
3. **Secret-masking compliance (ADR 0013 canary)** — every scenario that
   types into a secret field must also call
   `harness.AssertNoSecretLeak(t, secretValue)` which scans every captured
   frame for the literal value, the value's first/last 4 chars, and any
   substring of length ≥4 that matches the value. This is the structural
   enforcement of ADR 0013 across the suite, not just in the dedicated
   `TestRenderField_SecretFieldsNeverLeakValue` unit.

### 4. Cost-tab arbitrary-timeframe coverage — two lanes, both required

The user explicitly asks "estimate any timeframe (1h, 6h, 1d, 1w, 1M, 1y,
custom)". The harness must support this via two independent lanes so a
test can isolate UX bugs from arithmetic bugs.

**Lane A — math path (regression-detect arithmetic).**
DSL helper `EstimateMonthlyToDuration(monthly, d)` invokes the *exact same*
`costForPeriod` formula by constructing a transient
`costWindowPreset{d: d}` and calling the unexported scaling math directly
(or re-implementing the trivial `monthly * d.Seconds() / costMonthSecs`
inline in the DSL). Tests assert "monthly $730 → $1.00 for 1h ± epsilon".

**Lane B — keypress path (regression-detect keybindings).**
DSL helpers `StepTimeframeForward()` and `SetTimeframeIdx(i)` send `]`
and `[` key events through the harness, then assert
`harness.Model().costPeriodIdx == expectedIdx` and
`harness.Frame()` contains the expected suffix (`/hr`, `/day`, `/yr`).
This catches regressions in the dispatch order
(see PR #156 fix where `[` / `]` were swallowed by the credential form).

A "custom timeframe" entry in `costWindows` (open-ended duration input
through the UI) is **out of scope** for this ADR — it is a feature
request, not a harness requirement. Lane A already lets tests assert
"the math is correct for any duration"; Lane B will need extension only
if and when the feature ships.

### 5. CI integration

- The suite runs under the existing `make test-integration` target
  (defined by ADR 0015), invoked from a new `RuneGate/Quality/Integration`
  job in `quality-gates.yml`. The `merge-gate` aggregator depends on it.
- `make test` (default) does **not** run the integration suite; the
  `//go:build integration` tag excludes it. This keeps the developer-loop
  test time at its current ~5s and is consistent with ADR 0015.
- Adding `github.com/charmbracelet/x/exp/teatest` to `go.mod` **trips the
  `govulncheck` audit row in WORKFLOW.md** — the implementing PR must run
  `govulncheck ./...` and paste the summary in the Audit Checks table.

---

## Interface contracts (concrete targets for backend / frontend)

### Pricing

```go
package pricing

type Fetcher interface {
    USDPerHour(ctx context.Context, vendor, region, sku string) (float64, error)
}

type StaticFetcher map[string]float64 // key: vendor+"/"+region+"/"+sku

func (s StaticFetcher) USDPerHour(_ context.Context, v, r, sku string) (float64, error) { … }

type ctxKey struct{}

func WithFetcher(ctx context.Context, f Fetcher) context.Context
func FetcherFrom(ctx context.Context) Fetcher  // returns defaultFetcher if absent
```

The existing per-vendor live fetchers (`aws.go`, `azure.go`, …) implement
`Fetcher` as the default. `defaultFetcher` is the package-global initialised
on first use; production callers continue to work without context changes.

### Provider

```go
// EstimateMonthlyCostUSD signature change.
EstimateMonthlyCostUSD(ctx context.Context, cfg *config.Config) (Estimate, error)
```

All 12 providers update; `cost.Stream*` accept `ctx` and propagate.

### Dashboard

```go
package xapiri

// Package-level seam, defaults to time.Now. Tests override with t.Cleanup.
var Clock = time.Now
```

No other `dashModel` field changes are required by this ADR.

### Harness API (frontend deliverable)

```go
package xapiri

type Harness struct { /* unexported, wraps *teatest.TestModel */ }

func New(t *testing.T, opts ...Option) *Harness

func (h *Harness) Send(tea.Msg)
func (h *Harness) Key(tea.KeyType)
func (h *Harness) Type(string)
func (h *Harness) Frame() string
func (h *Harness) Model() dashModel
func (h *Harness) AssertNoSecretLeak(t *testing.T, secret string)
func (h *Harness) Quit()

// Mid-layer DSL (separate file, same package).
func (h *Harness) Tick(toggleID toiID)
func (h *Harness) Edit(field tiID, value string)
func (h *Harness) NavDown(n int)
func (h *Harness) NavUp(n int)
func (h *Harness) NextTab()
func (h *Harness) PrevTab()
func (h *Harness) OpenTab(t dashTab)
func (h *Harness) StepTimeframeForward()
func (h *Harness) StepTimeframeBack()
func (h *Harness) SetTimeframeIdx(i int)
func EstimateMonthlyToDuration(monthly float64, d time.Duration) float64
```

---

## Consequences

**Positive:**
- One harness shape per test; reviewers learn it once.
- Cost arithmetic and cost UX are independently testable; a broken
  keybinding does not mask a math regression and vice versa.
- ADR 0013 secret-masking compliance becomes a structural property of
  every scenario, not just one dedicated test.
- Pricing tests stop racing on the `pricing` package globals.
- Issue #191 becomes executable end-to-end: ADR 0015 defines *what*
  classes exist, ADR 0016 defines *how* each class is built.

**Negative / risks:**
- `Provider.EstimateMonthlyCostUSD` signature change is **breaking** for
  every provider implementation (12 packages) and every caller in
  `internal/cost/`. Mitigation: ship as a single PR; the change is
  mechanical (`ctx context.Context` parameter add).
- `pricing.Fetcher` interface adds a layer of indirection in a hot path.
  Default implementation must be zero-allocation when context carries no
  override (use a sentinel `defaultFetcher` singleton, return early in
  `FetcherFrom`).
- Adding `teatest` to `go.mod` triggers `govulncheck` per WORKFLOW.md;
  factor a clean run into the implementing PR.
- The `Clock = time.Now` package var pattern is mutable global state.
  Tests must use `t.Cleanup` to restore it; concurrent tests cannot
  override it independently. This is acceptable because the cost-tab
  scenarios do not need clock pinning at all and the wall-clock-using
  tabs are few.

---

## Alternatives considered

1. **Per-test `tea.NewProgram` wrapping without `teatest`.**
   Rejected: re-implements ~300 lines of teatest's frame capture and
   synchronous `Update` driving for no gain. teatest is the obvious
   industry-standard lever.

2. **Per-provider `SetPricingFetcher(Fetcher)` setter.**
   Smaller delta, but still process-global state that races between
   parallel scenarios. The context-scoped form is the only one that
   composes.

3. **Mock `cost.StreamWithRegions` instead of pinning the fetcher.**
   Rejected: the math under test (vendor pricing → monthly → period)
   lives below `Stream*`, so mocking at `Stream*` defeats the purpose.

4. **Keep the existing `SetCredentials` pattern and add a parallel
   `SetCatalog` global.**
   Rejected for the same race reason as alternative 2.

5. **Drive only golden-frame snapshots, no model inspection.**
   Rejected: a frame compare cannot distinguish "wrong frame because
   wrong state" from "wrong frame because rendering bug". Model
   inspection is required to localise failures.

---

## Acceptance criteria for the implementing PR(s)

A PR landing the harness skeleton (Layer Low + the pricing seam) is
**Done (Level 1)** when:

- [ ] `pricing.Fetcher` interface + `WithFetcher`/`FetcherFrom` shipped;
      all 12 providers' `EstimateMonthlyCostUSD` accept `ctx`.
- [ ] `internal/ui/xapiri/harness_test.go` compiles under
      `//go:build integration`; `New(t)` returns a usable `*Harness`.
- [ ] `make test-integration` target invoked in `quality-gates.yml` as
      `RuneGate/Quality/Integration`; `merge-gate` depends on it.
- [ ] One smoke-test scenario exercises both lanes of the cost-tab
      timeframe path against a `pricing.StaticFetcher`.
- [ ] `govulncheck ./...` clean; summary in Audit Checks table.
- [ ] PR body cites this ADR, ADR 0015, and issue #191.

Subsequent PRs (one per per-tab test class enumerated in ADR 0015 §"#191")
follow the same shape; each is **Done (Level 2)** when the corresponding
golden-frame fixture and scenario are committed and `make test-integration`
is green.
