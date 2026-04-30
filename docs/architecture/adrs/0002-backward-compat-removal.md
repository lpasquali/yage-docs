# ADR 0002 — Backward Compatibility Removal

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Backend programmer, Platform Engineer (review)

---

## Decision

**yage has no production users.** No installed base. No scripts in the
wild depending on old Secret names, env var aliases, or JSON formats.
Backward-compatibility shims are dead weight that complicate every read
of the affected files.

**All backward-compatibility code is removed immediately — no
deprecation cycle, no dual-read period, no aliases.**

This supersedes the Phase D migration plan in abstraction-plan.md §R.3
and §3 (the two-release dual-read/dual-write schedule). That plan is
withdrawn.

---

## Scope

Every item below is a self-contained deletion. No design decision is
required. Each can ship in its own small PR.

### 1. `SyncProxmoxBootstrapLiteralCredentialsToKind` (kindsync)

**Why it exists:** pre-Phase-D function that wrote Proxmox credentials
to kind Secrets using the old Proxmox-specific Secret layout.

**Why it is dead:** `WriteBootstrapConfigSecret` + provider
`KindSyncFields` covers all providers generically and is already called
from the orchestrator. The old function does overlapping work using the
old schema.

**Action for Backend:**

1. Delete `SyncProxmoxBootstrapLiteralCredentialsToKind` from
   `internal/cluster/kindsync/kindsync.go` (lines 42–163 approx).
2. Delete all call sites:
   - `internal/orchestrator/bootstrap.go` lines 105, 200, 339, 426, 465, 491, 682
   - `internal/platform/opentofux/outputs.go` lines 43, 64
3. Verify each deletion point already has a `WriteBootstrapConfigSecret`
   or `KindSyncFields`-backed call nearby. If a gap exists, fill it
   before deleting.
4. Run `make test`. Fix any test that references the deleted function.

### 2. Pre-split Secret name fallback (proxmox provider)

**Why it exists:** the CAPI + CSI credentials used to live in a single
combined Secret (`proxmox-bootstrap-credentials`). They were split into
separate Secrets. The fallback reads the old combined name when the new
names are missing.

**Action for Backend:**

1. Delete the fallback branch in `internal/provider/proxmox/state.go`
   around line 326 (`// Fallback to the pre-split Secret name (legacy)`).
2. If the surrounding function returns early when the primary Secret is
   not found, ensure the error is surfaced (not silently swallowed).

### 3. OpenStack `OS_*` env-var aliases

**Why they exist:** the OpenStack CLI uses `OS_PROJECT_NAME`,
`OS_REGION_NAME`, etc. yage honoured these as aliases so operators
didn't need to set both. Since there are no operators yet, keep only
the canonical `OPENSTACK_*` names.

**Action for Backend:**

Delete from `internal/config/config.go` lines 1303–1308:

```go
// DELETE these fallback reads:
os.Getenv("OS_PROJECT_NAME"),   // line 1303 (second arg to firstNonEmpty)
os.Getenv("OS_REGION_NAME"),    // line 1308
```

Keep the primary `OPENSTACK_PROJECT_NAME` / `OPENSTACK_REGION` reads.
Update `internal/ui/cli/usage.txt` if it documents `OS_*`.

### 4. `YAGE_CURRENCY` env-var alias (pricing)

**Why it exists:** renamed to `YAGE_TALLER_CURRENCY`; old name kept as
alias during transition.

**Action for Backend:**

1. Delete the alias branch in `internal/pricing/taller.go` around
   line 144 (`if v, n, ok := pickOrFallback(os.Getenv("YAGE_CURRENCY"), ...)`).
2. Delete the comment on line 28 referencing `YAGE_CURRENCY`.
3. Update `usage.txt` if `YAGE_CURRENCY` appears there.

### 5. Legacy JSON format in config snapshot

**Why it exists:** a config field was stored in a different JSON
structure in an older build. A fallback unmarshal reads the old format.

**Action for Backend:**

1. Delete `internal/config/snapshot.go` lines 310–315 (the `legacy`
   struct and its `json.Unmarshal` branch).
2. If the surrounding code depends on the result, confirm the primary
   unmarshal path already covers it.

### 6. `proxmoxmachines` GVR hardcode in purge.go

**Why it exists:** purge deletes leftover ProxmoxMachine CRD objects
by querying the `proxmoxmachines` GVR directly from the orchestrator.
This is a Proxmox-specific concern leaking into a non-provider file.

**Action for Backend:**

1. Move the `proxmoxmachines` GVR query from
   `internal/orchestrator/purge.go:347` into the Proxmox provider's
   `Purge(cfg *config.Config) error` implementation
   (`internal/provider/proxmox/`).
2. The orchestrator at line 306 already calls `prov.Purge(cfg)` — the
   proxmox implementation should handle its own CRD cleanup there.
3. Delete the hardcoded GVR from the orchestrator file.

### 7. `if cfg.InfraProvider == "proxmox"` guards wrapping abstracted calls

**Why they exist:** added when the provider interface was first wired;
the guards prevented non-Proxmox runs from hitting Proxmox-only code.
Now that `ErrNotApplicable` from `MinStub` is the skip signal, the
outer guards are redundant for any method already behind the interface.

**Action for Backend:**

Audit each of the 19 `cfg.InfraProvider == "proxmox"` checks in
`internal/orchestrator/bootstrap.go`. For each:

- **If the body only calls a `Provider` method** (e.g. `prov.Inventory`,
  `prov.EnsureIdentity`, `prov.EnsureScope`, `prov.EnsureCSISecret`):
  delete the outer guard. Let `ErrNotApplicable` drive the skip.
- **If the body calls something with no provider method yet** (e.g. BPG
  provider install, CAPMOX image-tag resolution): keep the guard and
  open a follow-up issue describing the provider method needed.
- Do not refactor unrelated code in the same PR.

---

## No new backward-compatibility code

Going forward, no new backward-compatibility shims are introduced.
When a field, Secret name, or env var changes:

- Update in one commit.
- Delete the old path in the same commit.
- Done.

The Architect will reject PRs that add `// legacy`, `// deprecated`,
`// backcompat`, `// fallback to old`, or dual-read logic without a
filed user-impact justification.

---

## Handoff checklist

Backend programmer completes items 1–7 above, opens one small PR per
item. Platform Engineer reviews any change that touches Secret schemas
(items 1, 2, 6) to verify no k8s resource shape changes introduce
silent data loss.
