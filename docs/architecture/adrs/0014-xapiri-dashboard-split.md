# ADR 0014 — xapiri dashboard.go Split

**Status:** Proposed  
**Date:** 2026-05-01  
**Author:** Architect agent  
**Relates to:** ADR 0013 (TUI Secret-Display Policy), ADR 0003 (TUI Dispatch Keymap)

---

## Context

`internal/ui/xapiri/dashboard.go` is 4 513 lines long and holds, in one file:

- The `dashModel` struct and its constructor `newDashModel`
- All 79 focus ID constants (`focCount` enum), 67 text-input slots, 6 select slots, 16 toggle slots
- The full `fieldMeta` table (`dashFields`) and the single renderer `renderField`
- Focus-visibility logic (`isHidden`, `visibleFocusList`, `moveFocus`, `jumpFocus`)
- The Bubble Tea outer `Update` dispatch and all 9 per-tab update methods
- `buildSnapshotCfg` and `flushToCfg` (paired atomic state serialisers, ~380 lines together)
- Chrome renderers: tab bar, system widget, footer, bottom strip
- PTY / embedded terminal helpers (~210 lines)
- Per-tab renderers for all 9 tabs
- Cost-tab helpers, log-tab helpers, deploy, deps, help, about
- All `tea.Msg` type definitions
- `runDashboard`, `dashDefault`, `marshalConfigYAML`

Three production regressions have been traced directly to the monolith's size making
the cross-cutting invariants invisible:

1. **PR #173 — secret value leak**: `renderField`'s unfocused branch bypassed the
   `meta.secret` check because the `fieldMeta` table was too far away to notice.
2. **In-flight — broken arrow-key navigation**: the token-overlay `default:` branch
   consumed arrow keys before `updateCfgEditScreen` could handle them.
3. **Queued — Deploy button exits**: wrong `Update` branch reached before
   `updateDeployTab`.

The existing sibling files (`state.go`, `shared.go`, `validate.go`, `logring.go`,
`feasibility_shim.go`, `georegions.go`) demonstrate that the package already supports
this style of decomposition within a single Go package.

## Decision

Split `dashboard.go` into 18 files using a **hybrid** strategy:

- Per-tab files hold only that tab's `updateXxx` and `renderXxx` methods.
- Shared-concern files hold everything that spans multiple tabs.
- `dashboard.go` (kept) holds only the model struct, constructor, `Init`, the
  top-level `Update` outer dispatch, and `View`.

All files remain in `package xapiri`. No public API changes. No behaviour changes.

### File layout

| File | Approx. lines | Responsibility |
|---|---|---|
| `dashboard.go` (kept, shrunk) | ~250 | `dashModel` struct, `newDashModel`, `Init`, top-level `Update` outer dispatch, `View`, `runDashboard`, `preserveTransientState`, `inTextField` |
| `dashboard_fields.go` | ~430 | `tiCount`/`siCount`/`toiCount`/`focCount` enums, `fkind`, `fieldMeta`, `dashFields`, `renderField` |
| `dashboard_focus.go` | ~160 | `isCloud`, `onPremProviders`, `providerListForMode`, `rebuildProviderList`, `isHidden`, `visibleFocusList`, `focusAtConfigRow`, `jumpFocus`, `moveFocus` |
| `dashboard_snapshot.go` | ~360 | `buildSnapshotCfg`, `flushToCfg`, `markDirty` |
| `dashboard_msgs.go` | ~80 | All `tea.Msg` type definitions + `waitForCostRowCmd` |
| `dashboard_chrome.go` | ~250 | `renderTabBar`, `renderSysWidget`, `fmtBytes`, `fmtRate`, `tabAtX`, `renderBottomStrip`, `renderFooter` |
| `dashboard_term.go` | ~210 | `termPaneHDefault`, `termPaneHMin`, `watchPTYCmd`, `processTermBytes`, `keyMsgToBytes`, `stripNonSGR`, `termRawToLines`, `renderTermPane` |
| `dashboard_overlays.go` | ~30 | `renderTokenPromptOverlay`, token-prompt key-handling block |
| `dashboard_cost_helpers.go` | ~120 | `costWindowPreset`, `costWindows`, `costDefaultPeriodIdx`, `costMonthSecs`, `activeCostWindow`, `costForPeriod`, `formatCost`, `sortedCostRows`, `kickRefreshCmd` |
| `tab_config.go` | ~140 | `updateConfigTab`, `updateCfgListScreen`, `updateCfgNewNameScreen`, `renderConfigTab`, `renderCfgListScreen`, `renderCfgNewNameScreen` |
| `tab_provision.go` | ~80 | `updateCfgEditScreen`, `renderCfgEditScreen` |
| `tab_editor.go` | ~280 | `cfgReady`, `loadCfgListCmd`, `loadCfgEntryCmd`, `switchToEditorTab`, `loadEditorResourcesCmd`, `updateEditorTab`, `openKindResourceEditorCmd`, `secretToEditableYAML`, `configMapToEditableYAML`, `applyEditedResourceToKind`, `parseEditableYAML`, `renderEditorTab`, `openEditorCmd`, `resolveEditor`, `renderEditorPlaceholder` |
| `tab_costs.go` | ~280 | `updateCostsTab`, `updateCostsCredsForm`, `saveCostCredsCmd`, `renderCostsTab`, `renderCostsCredsForm`, `renderCostRow`, `renderVendorRow` |
| `tab_logs.go` | ~120 | `updateLogsTab`, `renderLogsTab` |
| `tab_deploy.go` | ~50 | `updateDeployTab`, `renderDeployTab` |
| `tab_deps.go` | ~120 | `updateDepsTab`, `renderDepsTab` |
| `tab_help.go` | ~55 | `renderHelpTab` |
| `tab_about.go` | ~35 | `renderAboutTab` |

### What does NOT move

- `state.go`, `shared.go`, `validate.go`, `logring.go`, `feasibility_shim.go`,
  `georegions.go`, `editor_unix.go`, `xapiri.go`, `xapiri_test.go` — untouched.
- The Lipgloss palette (`colAccent`, `colOK`, `colWarn`, `colBad`, `colMuted`,
  `colHdr`, style vars) moves to `dashboard_fields.go` because `renderField` is the
  primary consumer; it is a package-level `var` block visible to all files.

### What is explicitly forbidden during the refactor

1. Do not add, remove, or rename any exported or package-private symbol.
2. Do not reorder or restructure the `Update` outer dispatch; the token-overlay →
   ctrl+s → esc/q → ctrl+left/right → ctrl+t → ctrl+alt+up/down → termFocused PTY
   passthrough → number keys → arrow tab cycle → per-tab dispatch order is
   load-bearing (ADR 0003).
3. Do not extract `renderField` into a per-tab file. It must remain co-located with
   `dashFields` in `dashboard_fields.go`.
4. Do not split `isHidden` per-tab. It references toggles and selects across all tabs
   and must stay whole in `dashboard_focus.go`.

## Regression-risk traps

These are the three highest-risk invariants the frontend must protect during and after
the refactor.

### Trap 1 — `renderField` + secret display policy (ADR 0013 canary)

`renderField` is the single renderer for all field types. The unfocused path **must**
branch on `meta.secret` before emitting any value. Splitting this function into
per-tab helpers breaks the guarantee because each helper would need its own copy of
the guard. The canonical regression test is
`TestRenderField_SecretFieldsNeverLeakValue` in `xapiri_test.go`; it **must pass**
throughout the refactor, not only at the end.

**Rule:** `renderField` and `dashFields` live in the same file (`dashboard_fields.go`)
and that file is reviewed as a unit on every PR that touches either.

### Trap 2 — Global focus ID namespace must not be split

`focCount` (79 IDs) spans all tabs. `isHidden` cross-references `toiFork`,
`toiRegistryEnabled`, `siMode`, and `siProvider` when evaluating visibility of fields
in tabs it does not own. Splitting `isHidden` per-tab produces N copies that will
drift. Splitting the `focCount` enum across files risks re-using the same iota value
in two files (Go does not prevent this across files in the same package if iota is
reset).

**Rule:** `focCount`, `tiCount`, `siCount`, `toiCount`, and `isHidden` all live in
`dashboard_fields.go` or `dashboard_focus.go` — never in a `tab_*.go` file.

### Trap 3 — `Update` outer dispatch order is load-bearing

The outer `Update` in `dashboard.go` must handle the token-prompt overlay before
routing to per-tab dispatch. If any `tab_*.go` update method starts handling
`tea.KeyMsg` before the overlay has been checked, the token-overlay and navigation
regressions from PR #173 and the in-flight arrow-key fix will resurface.

**Rule:** `Update` stays entirely in `dashboard.go`. Per-tab `updateXxxTab` methods
handle only the cases that `Update` explicitly delegates to them; they must not read
`tea.KeyMsg` directly for global keys (tab switch, quit, save, overlay).

## Consequences

**Positive:**
- Each tab's update + render code becomes independently reviewable (~50–280 lines per file).
- `renderField`, the secret-policy guard, and the focus namespace are each in one
  small file where reviewers can see the whole invariant at a glance.
- `buildSnapshotCfg`/`flushToCfg` are isolated in `dashboard_snapshot.go`; a diff
  that touches state persistence is obviously scoped.
- `go test ./internal/ui/xapiri/...` continues to be the single gate; no new test
  infrastructure needed.

**Negative / risks:**
- The mechanical move is a large diff (~4 500 lines moved) with high merge-conflict
  risk against any in-flight branch that touches `dashboard.go`. Coordinate with all
  open `dashboard.go` PRs (currently: arrow-nav fix, deploy-button fix) before
  starting.
- iota-reset risk: if a developer creates a new `tiCount`/`focCount`/`toiCount`
  constant in a `tab_*.go` file and forgets that iota resets per `const()` block, the
  global index will silently collide. The rule above (keep all enums in
  `dashboard_fields.go`) prevents this.

## Acceptance criteria for the mechanical refactor PR

A PR implementing this split is **Done (Level 2)** when all of the following hold:

- [ ] `go build ./...` passes with no new warnings.
- [ ] `go test ./internal/ui/xapiri/... -v` passes, including
      `TestRenderField_SecretFieldsNeverLeakValue`, `TestArrowNav_ProvisionTabAdvancesFocus`,
      and `TestArrowNav_TokenOverlayDoesNotSwallowArrows`.
- [ ] `go vet ./internal/ui/xapiri/...` clean.
- [ ] No exported or package-private symbol is renamed or removed (verified by
      `grep` diff on the symbol list).
- [ ] `dashboard.go` is ≤ 300 lines after the split.
- [ ] No `tab_*.go` file contains a `const (` block that introduces new iota values
      for the `tiCount`/`siCount`/`toiCount`/`focCount` namespaces.
- [ ] PR description cites this ADR and links the three regression tests.
