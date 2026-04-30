# ADR 0006 — Capacity Preflight Architecture

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Backend (implementation), Architect (interface contract)

---

## Context

Capacity preflight is the pre-deploy gate that checks whether the target
infrastructure has enough CPU, memory, and storage for the planned cluster
before committing any API calls.

Phase A of the provider abstraction plan merged the older split between
`capacity.FetchHostCapacity` + `capacity.FetchExistingUsage` into a single
`Provider.Inventory()` call. `Inventory` returns a `provider.Inventory` struct
with `Total`, `Used`, `Available`, and `Hosts` fields. Flat-pool providers
(Proxmox, OpenStack per-project) implement it; cloud providers (AWS, Azure,
GCP, Hetzner) return `provider.ErrNotApplicable` per §13.4 #1 because their
per-family quota models do not compose with flat `Total − Used` subtraction.
vSphere (govmomi, PRs #67) and OpenStack (Nova + Cinder, PR #66) implementations
have already landed on `main`.

The verdict logic lives in `internal/cluster/capacity/` and produces one of
three outcomes: `VerdictFits`, `VerdictTight`, or `VerdictAbort`. The
`CheckCombined` function evaluates (existing + planned) demand against two
thresholds: a soft budget fraction and a hard overcommit ceiling. Two operator
dials — `AllowOvercommit` and `OvercommitTolerancePct` — are surfaced via
`cfg.Capacity` and the xapiri TUI.

Before this ADR no document captured:

- The threshold values and their rationale.
- The exact semantics of `VerdictTight` vs `VerdictAbort` and how the two
  overcommit dials interact with them.
- What the orchestrator does when `Inventory` returns `ErrNotApplicable`.
- The CPU-vs-memory asymmetry in the hard abort ceiling.
- How the `--plan` (`--dry-run`) output integrates with capacity checking.
- The allocation planning model (`AllocationsFor` / `WorkloadAllocations`).

---

## Decision

### 1. Verdict thresholds and their rationale

The verdict algorithm is implemented in `CheckCombined`
(`internal/cluster/capacity/capacity.go`).

**Soft budget** — `DefaultThreshold = 2.0/3.0` (66%):
`(existing + planned)` must not exceed 66% of total host CPU, memory, or
storage. The remaining ~33% is reserved for the host OS, the Proxmox
hypervisor, rollout slack, and workload headroom that grows after initial
bootstrap. This value is stable and should not be changed except via the
operator-tunable `cfg.Capacity.ResourceBudgetFraction` field.

**Hard overcommit ceiling** — `DefaultOvercommitTolerancePct = 15.0`:
`(existing + planned)` may not exceed `host × 1.15` on the memory or disk
dimension. Beyond that the orchestrator refuses to continue. 15% captures the
Proxmox ballooning + swap headroom that is safe under normal load while
preventing OOM conditions under spike.

**CPU asymmetry**: the hard ceiling applies only to `max(memFrac, diskFrac)`,
not to `cpuFrac`. CPU overcommit is normal on Proxmox — VMs share physical
cores and rarely all run at 100% simultaneously. Memory is the dangerous
dimension: overcommit beyond physical RAM causes OOM-kills under load.
This asymmetry is intentional and must not be removed.

**Verdict rules** (in order):

| Condition | Verdict | Orchestrator action |
|---|---|---|
| `max(cpuFrac, memFrac, diskFrac) ≤ threshold` | `VerdictFits` | Silent proceed |
| `max(memFrac, diskFrac) ≤ 1 + tolerancePct/100` | `VerdictTight` | Warn + proceed |
| otherwise | `VerdictAbort` | Error (unless `AllowOvercommit=true`) |

**Small-environment detection** (`SmallEnvCPUCores = 8`, `SmallEnvMemoryGiB = 16`):
when host totals fall below these thresholds the orchestrator emits a
suggestion to use `--bootstrap-mode k3s`. This is advisory only — no verdict
changes.

**K3s sizing constants** (`K3sCPCores=2`, `K3sCPMemMiB=1024`, `K3sCPDiskGB=20`,
`K3sWorkerMem=1024`, `K3sWorkerDisk=15`): these define the per-node footprint
that `PlanForK3s` / `WouldFitAsK3s` use to check whether the same machine
counts would fit if the operator switched to `--bootstrap-mode k3s`.

### 2. Overcommit contract: two separate dials

There are two orthogonal operator controls. Conflating them is wrong.

**`cfg.Capacity.AllowOvercommit` (`ALLOW_RESOURCE_OVERCOMMIT=true`)**:
does NOT change the `Verdict` computed by `CheckCombined`. It changes what
the orchestrator does *after* receiving `VerdictAbort`: instead of returning
an error it logs a warning and proceeds. `VerdictTight` is unaffected —
it already proceeds with a warning regardless of this flag.

Use `AllowOvercommit` when the operator consciously accepts the risk and wants
to proceed on a host that is beyond the hard ceiling. It is an escape hatch,
not the normal mode of operation.

**`cfg.Capacity.OvercommitTolerancePct` (`OVERCOMMIT_TOLERANCE_PCT`)**:
changes the Tight→Abort boundary itself. Raising this value (e.g. from 15 to
30) widens the `VerdictTight` band so that higher memory fractions still
produce a warning-proceed rather than an abort. This is the right dial to use
when the operator has validated that a higher overcommit ratio is safe for
their hardware.

The two dials are not equivalent: `OvercommitTolerancePct` changes the
arithmetic; `AllowOvercommit` overrides the outcome. The xapiri TUI surfaces
both controls; the `--plan` output describes the active values.

### 3. ErrNotApplicable: silent skip vs. annotated skip

**In `preflightCapacity` (real run, `bootstrap.go`)**: when
`prov.Inventory(cfg)` returns `provider.ErrNotApplicable`, the preflight is
skipped silently — no log line, no warning. Operators targeting AWS, Azure,
GCP, or Hetzner see no capacity output. This is correct: their quota models
(per-family service quotas, IAM-account limits) do not express as flat
`Total − Used`. Attempting to check them here would produce misleading numbers.

A non-`ErrNotApplicable` error (network failure, credential error) is degraded
to a `logx.Warn` + nil return — the run continues. Capacity acquisition must
never be a hard blocker when the underlying API is temporarily unreachable.

**In `planCapacity` (`--plan` / `--dry-run`, `plan.go`)**: when
`prov.Inventory(cfg)` returns `ErrNotApplicable`, the section prints a `skip`
line: `"provider %q has no flat-pool inventory model — preflight skipped
(§13.4 #1)"`. This makes the absence explicit to the operator reviewing a
dry-run rather than silently omitting the section. The planned-resource
breakdown (roles × replicas × sizing) is always printed regardless of whether
live inventory data is available.

**Verdict display in `planCapacity`**: `✅ fits`, `⚠️ tight`, or `❌ abort` with
the numeric breakdown. For `VerdictAbort` the plan output also shows the k3s
alternative sizing if `WouldFitAsK3s` returns true.

### 4. Cloud quota APIs: current scope and follow-up

The following providers currently return `ErrNotApplicable` from `Inventory`:
AWS, Azure, GCP, Hetzner. The reason (§13.4 #1) is that their quota models do
not compose with flat subtraction — AWS has per-instance-family service quotas,
GCP has per-region per-resource quota, Azure has per-subscription per-SKU
limits. None of these map to a single `Cores / MemoryMiB / StorageGiB` triple
without lossy aggregation that would mislead the operator.

Two providers have full `Inventory` implementations:
- **Proxmox** (`internal/provider/proxmox/inventory.go`): node query +
  storage query + VM list (running-only for CPU/RAM, all VMs for disk).
  Reference flat-pool implementation.
- **vSphere** (`internal/provider/vsphere/inventory.go`, PR #67): govmomi
  host aggregation; ESXi hosts listed in `Inventory.Hosts`.
- **OpenStack** (`internal/provider/openstack/inventory.go`, PR #66):
  Nova + Cinder project quota; per-project flat totals.

The decision on wiring vendor quota APIs (AWS `service-quotas`, GCP
`cloudquotas`, Azure `Microsoft.Quota`) behind `Inventory` for those three
providers is **deferred**. These APIs expose different shapes (count-based
limits, per-family caps, per-region per-SKU limits) and would require a
separate `QuotaInventory` response shape or a Notes-only advisory mode that
does not feed into `CheckCombined`. The current ADR records that:

- `ErrNotApplicable` is the correct return for AWS/GCP/Azure/Hetzner today.
- Any future cloud quota implementation must not feed fabricated flat-pool
  numbers into `CheckCombined` — it may contribute advisory `Notes` entries.
- A follow-up issue should define the contract before implementation starts.

### 5. `--plan` integration: planCapacity + planAllocations

Both `planCapacity` and `planAllocations` are called unconditionally from
`PrintPlanTo` in `plan.go`. They run after all provider-specific phases
(`describeIdentity`, `describeWorkload`, `describePivot`).

**`planCapacity`** always prints the planned-resource breakdown (roles × replicas
× sizing from config). It then attempts a live `Inventory` call. If that
fails or returns `ErrNotApplicable`, the verdict section is replaced by a
`skip` annotation. If live data is available, the full two-threshold verdict is
printed with numeric details.

**`planAllocations`** computes `capacity.AllocationsFor(cfg)` which divides
worker-node capacity into four buckets: `SystemApps` (fixed reserve,
default 2000 mCPU / 4096 MiB for yage's own add-ons), and three equal
thirds of the remainder for `Database`, `Observability`, and `Product`.
These are planning numbers only — no `ResourceQuota` objects are created.
The operator can override the system reserve via
`cfg.Capacity.SystemAppsCPUMillicores` / `cfg.Capacity.SystemAppsMemoryMiB`.
An over-reserve (system bucket alone exceeds total worker capacity) is surfaced
as a warning in the plan output; `WorkloadAllocations.IsOverReserved()` drives
that check.

The `xapiri` TUI does not currently have a dedicated capacity widget that
calls `CheckCombined` live. The Provision tab surfaces sizing fields; capacity
feedback is available only via `--plan`. This is a known gap, not a bug.

---

## Consequences

- Operators on AWS, GCP, Azure, or Hetzner see no capacity section in the real
  run and a "preflight skipped (§13.4 #1)" note in `--plan`. This is correct
  and expected. Operators who want quota visibility should use the vendor's own
  CLI/console tools until a cloud quota advisory mode is implemented.

- The CPU-asymmetry in the abort ceiling means a plan that overcommits CPU
  (e.g. 120% of physical cores) will produce `VerdictTight` or `VerdictFits`,
  not `VerdictAbort`. This is intentional for Proxmox workloads where CPU
  sharing is normal. Operators who want strict CPU enforcement can lower
  `ResourceBudgetFraction` to force CPU into the soft-budget check.

- `AllowOvercommit=true` does not suppress `VerdictTight` warnings — it only
  converts `VerdictAbort` errors into warnings. Operators who set it expect
  a warning-proceed, not a silent proceed. Documentation and TUI labels must
  reflect this.

- The three-bucket allocation model (database / observability / product) is
  purely advisory. It produces no Kubernetes objects. The intent is to give
  operators a sanity-check before they commit sizing — not to enforce namespace
  quotas at admission time.

- A future cloud quota advisory implementation must route through `Inventory.Notes`
  and must not feed the numbers into `CheckCombined`. Violating this constraint
  would produce misleading capacity verdicts for cloud providers.

---

## References

- `internal/cluster/capacity/capacity.go` — `DefaultThreshold`,
  `DefaultOvercommitTolerancePct`, `CheckCombined`, `Check`, `PlanFor`,
  `PlanForK3s`, `WouldFitAsK3s`, `Verdict`, `SmallEnvCPUCores`,
  `SmallEnvMemoryGiB`, `K3s*` constants
- `internal/cluster/capacity/allocations.go` — `AllocationsFor`,
  `WorkloadAllocations`, `IsOverReserved`
- `internal/provider/provider.go` — `Provider.Inventory`, `ErrNotApplicable`
- `internal/provider/inventory.go` — `Inventory`, `ResourceTotals`
- `internal/provider/proxmox/inventory.go` — reference flat-pool implementation
- `internal/provider/vsphere/inventory.go` — govmomi flat-pool implementation
- `internal/provider/openstack/inventory.go` — Nova+Cinder flat-pool implementation
- `internal/orchestrator/bootstrap.go` — `preflightCapacity`,
  `hostCapacityFromInventory`, `existingUsageFromInventory`
- `internal/orchestrator/plan.go` — `planCapacity`, `planAllocations`,
  `PrintPlanTo`
- abstraction-plan.md §A (Phase A narrative)
- abstraction-plan.md §13.4 #1 — why cloud providers return ErrNotApplicable
