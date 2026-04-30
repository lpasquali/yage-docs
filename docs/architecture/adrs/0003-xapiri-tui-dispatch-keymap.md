# ADR 0003 — xapiri TUI Dispatch Architecture and Keymap Normalization

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Frontend programmer, Platform Engineer (review of huh picker change)

---

## Context

xapiri uses two distinct UI surfaces that each have their own keymap:

1. **Huh config picker** (`internal/cluster/kindsync/bootstrap_config.go`) — a
   `github.com/charmbracelet/huh` form that runs before the dashboard to select or
   create a named config. Huh's default keymap binds `j`/`k` for list navigation and
   `Tab`/`Shift-Tab` for field cycling.

2. **Bubble Tea dashboard** (`internal/ui/xapiri/dashboard.go`) — the main TUI.
   j/k bindings were copied into the dashboard in four separate locations during
   iterative development.

Both surfaces are presented to the same user in the same session. The j/k bindings in
the huh picker create a muscle-memory expectation that breaks in the dashboard, and
`Tab` in the edit form works as field-navigation in some contexts but as tab-switch in
others. Users must mentally switch keymaps between surfaces and sometimes within the
same screen.

### Root causes identified in `dashboard.go`

**RC-1 — Four separate j/k binding sites.**
`updateCfgListScreen`, `updateCfgEditScreen`, `updateLogsTab`, and `updateCostsTab`
each independently bind `keyStr == "j"` / `keyStr == "k"`. Every new tab that adds
list navigation copy-pastes the same pattern. There is no central "list navigation"
handler.

**RC-2 — `inTextField` guard is incomplete.**
The guard at the top of `Update()` is used to decide whether global shortcuts (number
keys, `[/]` timeframe, etc.) should fire or yield to a focused text input. It currently
covers:

```
tabProvision + fkText fields
tabConfig + cfgScreenNewName input
tabLogs + log-filter input
```

It does NOT cover `tabCosts + costCredsMode`. When the cost-credentials form is open
(text inputs focused), pressing a number key fires the global tab-switch handler instead
of typing into the form. Any future tab that contains a text input must explicitly update
this guard or it will silently steal keys.

**RC-3 — `cfgEntryLoadMsg` state-copy list is fragile.**
When a config is loaded, `newDashModel` is called with the new config and a fixed list
of fields is manually copied back from the old model (costPeriodIdx, logLines, logSub,
sysSampler, sysStats, width, height). Any new field added to `dashModel` is silently
wiped on config reload unless someone remembers to add it to this list.

**RC-4 — `detectFork` is not called during dashboard init.**
`initFromConfig` in `state.go` sets `s.fork` only from `cfg.InfraProvider`. In a fresh
dashboard session where InfraProvider is not yet set, `s.fork` stays `forkUnknown`.
`detectFork` in `shared.go` (which checks env vars: `PROXMOX_URL`,
`AWS_ACCESS_KEY_ID`, etc.) is only called from `step0_modePick` in the huh walkthrough
path, never from `initFromConfig`. A user who opens xapiri with `PROXMOX_URL` set will
see the dashboard default to "cloud" mode rather than "on-prem".

**RC-5 — Cost comparison runs blind when GeoIP is off and DataCenterLocation is unset.**
`cfg.GeoIPEnabled` defaults to `false`. `kickRefreshCmd` is gated on GeoIPEnabled to
perform the geo lookup, and falls back to `cfg.DataCenterLocation` only if the field is
set. When both are absent `regionsByProvider = nil` and cost comparison returns no
results, with no explanation shown to the user. This affects currency selection too —
the region context is what drives the currency default.

---

## Decision

### 1. Keymap: remove j/k, keep up/down arrows everywhere

`j` and `k` are removed from all list-navigation handlers in `dashboard.go`. Only
`tea.KeyUp` and `tea.KeyDown` are used. This eliminates the mismatch between the huh
picker (where j/k are the *default*) and the dashboard.

`Tab` is removed from the form-field cycling handler in `updateCfgEditScreen`. Field
focus moves with `up`/`down` arrows only. `Tab` is reserved for its global role
(future use, e.g. surface switching).

Left/right arrow keys (`tea.KeyLeft` / `tea.KeyRight`) are kept for `fkSelect` fields —
they cycle option values within a field, which is distinct from row navigation.

The `[` and `]` keys for timeframe cycling on the costs tab are **kept**. Left/right
arrows on the costs tab already have a conflicting binding (global tab cycling),
so reassigning them would create a worse conflict. `[/]` are documented in the help tab.

### 2. Huh picker: override default keymap

In `SelectBootstrapConfigForXapiri` (and any other huh form in the codebase), the huh
keymap is overridden via `WithKeyMap` to remove j/k navigation before the form is
presented:

```go
km := huh.NewDefaultKeyMap()
km.Select.Up.SetKeys("up")
km.Select.Down.SetKeys("down")
km.Select.Filter.SetKeys()          // disable filter shortcut if present
// keep tab for huh-internal use within the form
huh.NewForm(...).WithKeyMap(km)
```

This makes the huh picker's keymap consistent with the dashboard before users reach it.

### 3. Fix `inTextField` guard (RC-2)

The `inTextField` guard is promoted from an inline boolean expression to a function:

```go
func (m dashModel) inTextField() bool {
    switch m.activeTab {
    case tabProvision:
        return m.focusedField < len(m.fields) && m.fields[m.focusedField].kind == fkText
    case tabConfig:
        return m.cfgScreen == cfgScreenNewName
    case tabLogs:
        return m.logFiltering
    case tabCosts:
        return m.costCredsMode
    default:
        return false
    }
}
```

Any future tab that contains a text input adds a case here. The build remains the gate.

### 4. Fix `cfgEntryLoadMsg` state-copy (RC-3)

The manual field-copy list in the `cfgEntryLoadMsg` handler is replaced by a
`preserveTransientState(old, new dashModel) dashModel` helper that lives next to
`newDashModel`. The helper holds the authoritative list of fields that survive config
reload. Every new persistent-display field is added here (not scattered in the handler).

### 5. Call `detectFork` in `initFromConfig` (RC-4)

`initFromConfig` in `state.go` calls `detectFork(cfg)` when `cfg.InfraProvider` is
empty to populate `s.fork` from env vars before the dashboard is displayed. This
ensures that a user opening xapiri with `PROXMOX_URL` set sees "on-prem" mode
immediately, not only after completing the walkthrough.

### 6. DataCenterLocation/currency prompt when GeoIP is off (RC-5)

When `cfg.CostCompareEnabled == true` and `cfg.GeoIPEnabled == false` and
`cfg.DataCenterLocation == ""`, the dashboard shows a one-line inline prompt at the
top of the costs tab:

```
DataCenter location unset — cost comparison unavailable. Set YAGE_DATA_CENTER_LOCATION
or enable GeoIP to activate region-aware pricing.
```

No blocking gate is added; the message is informational. The prompt disappears once
`DataCenterLocation` is populated. This surfaces the cause of "costs tab shows nothing"
without requiring the user to dig through env var docs.

---

## Implementation scope

All items are self-contained changes to existing files. No new files required.

| Item | File(s) | Effort |
|---|---|---|
| Remove j/k from 4 sites | `dashboard.go` | Small |
| Remove Tab from field-nav | `dashboard.go` | Small |
| Huh keymap override | `kindsync/bootstrap_config.go` | Small |
| Promote `inTextField` to method | `dashboard.go` | Small |
| `preserveTransientState` helper | `dashboard.go` | Small |
| `detectFork` in `initFromConfig` | `state.go` | Small |
| DataCenterLocation inline prompt | `dashboard.go` (costs tab view) | Small |
| Document `[/]` in help tab | `dashboard.go` (help tab view) | Trivial |

All items can ship in a single PR. They are logically related (all address the same
keymap/dispatch/init gap) and each change is small enough to review at once.

---

## What is NOT changed

- The `[/]` timeframe keys on the costs tab — kept as-is, documented in help.
- Left/right arrows for `fkSelect` option cycling — kept.
- Global tab-switch bindings (number keys, ctrl+alt+N) — kept.
- The `YAGE_XAPIRI_TUI` flag and `useHuhTUI()` function — kept (pending a separate
  naming cleanup if desired; the confusion is in the name, not the behavior).

---

## Handoff checklist

Frontend programmer:

- [ ] Remove `keyStr == "j"` / `keyStr == "k"` from `updateCfgListScreen`,
      `updateCfgEditScreen`, `updateLogsTab`, `updateCostsTab`
- [ ] Remove `key == tea.KeyTab` branch from `updateCfgEditScreen` field navigation
- [ ] Promote `inTextField` inline expression to `(m dashModel) inTextField() bool`
      method; update all call sites
- [ ] Add `case tabCosts: return m.costCredsMode` to `inTextField`
- [ ] Introduce `preserveTransientState(old, new dashModel) dashModel` and replace
      manual copy block in `cfgEntryLoadMsg` handler
- [ ] In `state.go` `initFromConfig`: call `detectFork(cfg)` when `cfg.InfraProvider == ""`
- [ ] In costs-tab `View()`: render inline "location unset" message when
      `CostCompareEnabled && !GeoIPEnabled && DataCenterLocation == ""`
- [ ] In `kindsync/bootstrap_config.go`: apply `WithKeyMap` override to all huh forms
      to disable j/k navigation
- [ ] Update help tab to document `[` / `]` timeframe keys

Platform Engineer reviews huh picker change (no k8s resource impact; review is for
keymap correctness and that huh's `WithKeyMap` call compiles against the installed
version).
