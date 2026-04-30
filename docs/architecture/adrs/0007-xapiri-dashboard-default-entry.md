# ADR 0007 — xapiri: Dashboard as Default Entry Point

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Frontend (implementation), Architect (interface contract)

---

## Context

`yage --xapiri` has two execution paths in `internal/ui/xapiri/xapiri.go`:

1. **Legacy bufio walkthrough** (default): a step-driven terminal flow built on
   `huh` prompts and `bufio` output. Steps run sequentially:
   `stepKindClusterName` → `stepConfigSelection` → `MergeBootstrapSecretsFromKind`
   → `step0_modePick` → provider-specific fork.

2. **Dashboard** (opt-in, `YAGE_XAPIRI_TUI=huh`): the full-screen bubbletea/lipgloss
   TUI. `runHuhBranch` brings the kind cluster up, then launches `runDashboard`.
   The dashboard has a dedicated **Config tab** (`tabConfig`) that lists all
   bootstrap-config Secrets in the kind cluster, lets the user pick one or create
   a new draft, and calls `loadCfgEntryCmd` to merge the selected config.

The Config tab in the dashboard is a strict superset of `stepConfigSelection`:

| Capability | `stepConfigSelection` | Config tab |
|---|---|---|
| List existing configs with status | ✅ | ✅ |
| Create new draft with custom name | ✅ | ✅ |
| Non-TTY fallback (auto-select first) | ✅ | N/A (dashboard requires TTY) |
| Reload list after external change | ❌ | ✅ (`r` key) |
| Switch config mid-session | ❌ | ✅ |
| Reflects `realized` vs `draft` status | ✅ | ✅ |

The `YAGE_XAPIRI_TUI=huh` environment gate was introduced as a feature-flag
for the dashboard while it was incomplete. The dashboard has since received
all features required to replace the legacy flow: config selection, provider
config, network config, cost estimation, logs, capacity feedback, and deployment
dispatch.

Keeping the legacy flow as the default creates user confusion: operators who
run `yage --xapiri` without setting the env var land in a step-by-step terminal
flow that includes a config selection prompt, then provider prompts — when the
dashboard already handles all of this in a single, navigable interface.

---

## Decision

### 1. Make the dashboard the default

`useHuhTUI()` currently returns true only when `YAGE_XAPIRI_TUI=huh`. The
function body is changed to return `true` unconditionally. The env var
override is kept for a single additional cycle (one sprint) to allow operators
who may have scripted `YAGE_XAPIRI_TUI=` (empty/false) to opt out, then
removed.

```go
// Before:
func useHuhTUI() bool {
    return strings.EqualFold(strings.TrimSpace(os.Getenv("YAGE_XAPIRI_TUI")), "huh")
}

// After:
func useHuhTUI() bool {
    v := strings.TrimSpace(os.Getenv("YAGE_XAPIRI_TUI"))
    if v != "" {
        return strings.EqualFold(v, "huh")
    }
    return true // dashboard is default
}
```

### 2. Config selection is handled exclusively by the Config tab

`stepConfigSelection` is **not called** from `runHuhBranch`. This is already
the correct behaviour — the Config tab calls `loadCfgEntryCmd` which merges
the selected bootstrap-config Secret when the user picks one.

`stepConfigSelection` (and `kindsync.SelectBootstrapConfigForXapiri`) remain
in the codebase for the legacy bufio path. They will be deleted when the
legacy path is removed (tracked separately).

### 3. `stepKindClusterName` is replaced by a config default

The legacy path calls `stepKindClusterName` to ask the operator for the kind
cluster name. `runHuhBranch` already defaults to `"yage-mgmt"` when
`cfg.KindClusterName` is empty. A future CLI flag (`--kind-cluster`) can
override this explicitly; in the interim the default is correct for almost
all single-machine installs.

### 4. Legacy bufio path is deprecated but not yet removed

The `if !useHuhTUI()` branch in `Run()` is kept and marked deprecated. It will
be removed in a follow-up issue once any remaining functional gaps between the
dashboard and the legacy flow are confirmed closed. This is explicitly **not** a
backward-compat obligation — there are no external callers.

---

## Implementation checklist (Frontend agent)

- [ ] Change `useHuhTUI()` to default `true` when `YAGE_XAPIRI_TUI` is unset
  (keep explicit `=huh` as opt-in, explicit `=legacy` or `=bufio` as opt-out)
- [ ] Add a `// Deprecated` comment to the legacy branch in `Run()` and to
  `stepConfigSelection()` and `stepKindClusterName()`
- [ ] Verify `runHuhBranch` does NOT call `stepConfigSelection` (already correct —
  confirm with a quick read; no code change needed)
- [ ] Update `usage.txt` and any `--help` output: remove mention of
  `YAGE_XAPIRI_TUI=huh` as required; note it is now opt-out
- [ ] `make build && go test ./...` green

---

## Consequences

- Operators who run `yage --xapiri` without any env var now land in the
  dashboard directly. No intermediate config selection prompt. The Config tab
  is the first screen and prompts for selection.

- `YAGE_XAPIRI_TUI=huh` continues to work (it selects the dashboard, which
  is now the default anyway). Scripts that set it remain functional.

- `YAGE_XAPIRI_TUI=legacy` (or `=bufio`) opts back to the old flow as an
  escape hatch for one sprint.

- The initial "choose a config file" prompt that appeared before the
  walkthrough in the legacy flow is gone. Config selection is now part of the
  dashboard's Config tab, which is richer (live reload, mid-session switch,
  explicit status column).

---

## References

- `internal/ui/xapiri/xapiri.go` — `Run`, `useHuhTUI`, `runHuhBranch`,
  `stepConfigSelection`, `stepKindClusterName`
- `internal/ui/xapiri/dashboard.go` — `runDashboard`, `loadCfgListCmd`,
  `loadCfgEntryCmd`, `renderCfgListScreen`, `tabConfig`
- `internal/cluster/kindsync/bootstrap_config.go` — `SelectBootstrapConfigForXapiri`,
  `ListBootstrapCandidates`
- ADR 0003 — xapiri TUI dispatch and keymap normalization
