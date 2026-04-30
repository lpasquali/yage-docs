# ADR 0005 — Pricing / Cost Subsystem Architecture

**Status:** Accepted
**Date:** 2026-04-30
**Owners:** Backend (implementation), Architect (interface contract)

---

## Context

Two packages handle cloud cost estimation:

- `internal/pricing/` — live FinOps pricing fetchers, one file per vendor
  (`aws.go`, `azure.go`, `gcp.go`, `hetzner.go`, `digitalocean.go`, `linode.go`,
  `oci.go`, `ibmcloud.go`). Also handles currency conversion (`taller.go`),
  geo-region mapping (`georegion.go`), and per-service Postgres add-on pricing
  (`aws_postgres.go`, `azure_postgres.go`, `gcp_postgres.go`, `oci_postgres.go`,
  `ibmcloud_postgres.go`).
- `internal/cost/` — multi-cloud cost comparator (`compare.go`), managed-add-on
  costs (`managed.go`, `addons.go`, `postgres.go`).

`Provider.EstimateMonthlyCostUSD` is the seam between the provider plugin system
and the pricing layer. Each provider's `cost.go` calls into `internal/pricing/`
for live prices and assembles a `provider.CostEstimate` breakdown. Cloud providers
wrap any `pricing.ErrUnavailable` as `provider.ErrNotApplicable` before returning.
On-prem providers (Proxmox, on-premises vSphere, private OpenStack) return
`provider.ErrNotApplicable` directly because they have no vendor pricing API.

The xapiri TUI surfaces live cost estimates on the Costs tab, streamed via
`kickRefreshCmd` which calls `cost.StreamWithRegions`. The `--cost-compare`
CLI flag calls `cost.PrintComparison` (which calls `CompareClouds`) directly.
The `--dry-run` plan calls `planMonthlyCost` + `planCostCompare` in
`internal/orchestrator/plan.go`.

---

## Decision

### 1. Caching strategy

Pricing data uses a two-layer cache: a 24-hour disk cache per SKU and a
24-hour disk cache for FX rates. There is no in-memory price cache between
the disk layer and the live fetcher. The taller currency selection is
in-process `sync.Once` per process.

**Disk SKU cache** (`~/.cache/yage/pricing/<vendor>.<sku>.<region>.json`):
`DefaultTTL = 24h` (constant in `pricing.go`). `Fetch()` reads the cache first;
only when the entry is missing or stale does it call the live fetcher and write
back. Cloud list prices change on month timescales; 24h is the correct TTL.

**Disk FX cache** (`~/.cache/yage/pricing/fx-USD.json`):
`tallerFXCacheTTL = 24h`. `ensureFXLoaded()` tries the disk cache before
hitting `open.er-api.com`. Same 24h rationale — FX rates for planning estimates
do not need sub-day freshness.

**Taller currency resolution** (`resolveTallerCurrency`): runs once per process
via `sync.Once`. The resolved code is stable for the process lifetime; back-to-back
`TallerCurrency()` calls are a pure in-memory read.

**Airgap / test kill-switch**: setting `YAGE_PRICING_DISABLED=true` bypasses all
fetching; `SetAirgapped(true)` (from `cfg.Airgapped`) has the same effect and
is the production override for offline environments. FX loading skips live fetch
when `disabled` is set; the local `{"USD": 1.0}` stub keeps conversion from
panicking.

**This layering is correct and stays as-is.** The only permitted extension is
narrowing the TTL per provider when a vendor publishes prices more frequently
than monthly — none currently qualify.

### 2. Error contract: ErrNotApplicable vs ErrUnavailable

The two sentinel errors have distinct semantics that must not be conflated:

- **`provider.ErrNotApplicable`** — this provider has no vendor pricing concept.
  Applies to: Proxmox (self-hosted TCO), on-prem vSphere, private OpenStack.
  The orchestrator and xapiri costs tab suppress the cost section entirely for
  this provider when this error is returned; no error message is rendered.

- **`pricing.ErrUnavailable`** — the vendor's live pricing API was reachable at
  one time but isn't now (network timeout, HTTP 5xx, stale cache that failed
  refresh). Applies to: any cloud provider (AWS, Azure, GCP, Hetzner, DO,
  Linode, OCI, IBM Cloud) when the API call fails.

**Current behaviour** (known issue): `provider/*/cost.go` wraps
`pricing.ErrUnavailable` as `provider.ErrNotApplicable`
(e.g. `fmt.Errorf("%w: ...", provider.ErrNotApplicable, err)`). The result is
that a transient API outage causes the cost row to disappear silently rather
than showing "(estimate unavailable)". The `cost.CompareClouds` table rendering
already has correct logic for non-nil `r.Err` — it prints the error string —
but it only triggers if the error is non-nil and not wrapped as
`ErrNotApplicable`.

**Accepted intent**: the wrap pattern in cloud `cost.go` files is wrong and must
be corrected as a cleanup task. The correct call-site rule is:

- Return bare `pricing.ErrUnavailable` (or wrap it with additional context) when
  a live API is unreachable. The cost table renders "unavailable"; onboarding
  hints may fire.
- Return `provider.ErrNotApplicable` only when the provider is structurally
  incapable of providing pricing (self-hosted, no billing API by design).

Until the cleanup lands, callers that display cost rows should treat any non-nil
error as a display-worthy failure, not a silent skip.

### 3. Package split: internal/cost/ vs internal/pricing/

The boundary is intentional and stays:

- `internal/pricing/` owns: per-vendor SKU catalog fetchers, disk cache, FX
  conversion, taller currency resolution, geo-region centroid ranking,
  onboarding hints, credential/currency process-globals (`credentials.go`).
  Everything in this package speaks "one vendor, one SKU, one price".

- `internal/cost/` owns: cross-vendor comparator (`CompareClouds`,
  `StreamWithFilter`, `StreamWithRegions`), managed-service matrix
  (`managed.go`), in-cluster substitute footprints (`addons.go`), block-storage
  retention math. Everything in this package speaks "compare N providers, pick
  the shape".

The comparator's parallel fan-out, sort, scope filtering, and retention
arithmetic belong in `cost/`. Merging the packages would create an import cycle
(`pricing` is imported by provider packages; `cost` imports `provider`;
providers cannot import `cost`).

### 4. onboarding.go and credentials.go

`credentials.go` exposes two process-globals (`creds Credentials`,
`prefs Currency`) and their setters (`SetCredentials`, `SetCurrency`).
`cmd/yage/main.go` calls both once at startup after `config.Load()`, populating
them from `cfg.Cost.Credentials` and `cfg.Cost.Currency`. Every pricing fetcher
reads from these globals; they never call `os.Getenv` directly. This keeps the
credential-source decision in one place and avoids ambient-shell-credential bleed
(notably: AWS ignores `AWS_ACCESS_KEY_ID` / `AWS_PROFILE` from the shell —
credentials must be explicitly set via the yage config).

`onboarding.go` provides `MaybePrintOnboarding(w, vendor)`, which writes a
one-time CLI/web setup snippet to `w` when: (a) the user has not seen it yet
(no sentinel file at `<cacheDir>/.onboarded-<vendor>`), and (b) credentials
for that vendor are not configured (`PricingCredsConfigured`), and (c) the
vendor has a hint (Azure, Linode, OCI have none — their APIs are anonymous).
`YAGE_NO_PRICING_ONBOARDING=1` suppresses; `YAGE_FORCE_PRICING_ONBOARDING=1`
force-replays.

`MaybePrintOnboarding` is called from two sites: `cost.PrintComparison` (after
the `--cost-compare` table is rendered) and `orchestrator/plan.go:planMonthlyCost`
(when the active provider's cost estimate fails). The xapiri TUI has its own
`costCredsMode` form for in-TUI credential entry; it does not call
`MaybePrintOnboarding` directly.

### 5. Currency / region flow: taller.go + georegion.go

Currency and region are resolved from the same root input
(`cfg.Cost.Currency.DataCenterLocation`, an ISO-3166 alpha-2 country code) but
through separate mechanisms:

**Currency (`taller.go`)**: `resolveTallerCurrency()` runs once per process.
Resolution order:
1. `prefs.DisplayCurrency` (explicit override via `YAGE_TALLER_CURRENCY` or
   `cfg.Cost.Currency.DisplayCurrency`)
2. `prefs.DataCenterLocation` → `CountryCurrency(cc)` → ISO-4217 code
3. Geo-IP detection (`ip-api.com/json/?fields=status,countryCode`) → country
   code → ISO-4217 code
4. `"USD"` fallback if all above fail

If the resolved code is not in `currencySymbol` (the supported-currency set),
the resolver falls back to USD and prints `UnsupportedCurrencyMessage`.

**Region (`georegion.go` + `xapiri/georegions.go`)**: `GeoRankedRegions(prov,
lat, lon, limit)` returns the nearest region slugs for a provider using
Haversine distance against embedded centroids. In `kickRefreshCmd`,
`cost.StreamWithRegions` receives up to 4 geo-ranked regions per provider. When
geo-IP is disabled and no `DataCenterLocation` is set, `regionsByProvider` is
nil and `StreamWithRegions` falls back to the region already set in the config
snapshot (single estimate per provider).

The country-to-centroid lookup (`pricing.CountryCentroid`, in `country_geo.go`)
is used by both the TUI's `ApplyDataCenterLocationDefaults` and the xapiri
`stampGeoRegions` path to pre-fill `cfg.Providers.*.Region` / `.Location`
fields before the first cost fetch.

---

## Consequences

- Operators on airgapped or offline hosts always see "estimate unavailable" for
  cloud providers — there is no stale-number substitution. This is the
  correct trade-off for a planning tool: a stale fabricated number is worse than
  a clear "(unavailable)" label.

- The 24h disk TTL means back-to-back `--dry-run` calls within a day share the
  same price snapshot. Operators who need truly live prices can `rm -rf
  ~/.cache/yage/pricing/` or set `YAGE_PRICING_CACHE=/dev/null` (which will
  force a live fetch on every call, at the cost of latency).

- The ErrUnavailable-wrapped-as-ErrNotApplicable bug in cloud `cost.go` files
  is tracked as a known cleanup. Until corrected, the `--cost-compare` table
  and xapiri Costs tab will silently drop cloud providers whose APIs are
  unreachable rather than showing "(unavailable)".

- The `credentials.go` process-globals mean pricing credentials cannot be
  changed mid-run. This is intentional — `cmd/yage/main.go` owns the one
  correct call site.

- Onboarding hints display at most once per vendor per cache directory. Running
  `yage --cost-compare` on a fresh install will print setup instructions for
  every vendor whose credentials are missing; subsequent runs are silent. This
  is the expected UX.

---

## References

- `internal/pricing/pricing.go` — `DefaultTTL`, `Fetch`, cache read/write
- `internal/pricing/taller.go` — `resolveTallerCurrency`, FX loading
- `internal/pricing/georegion.go` — `GeoRankedRegions`, centroid tables
- `internal/pricing/onboarding.go` — `MaybePrintOnboarding`, `PricingCredsConfigured`
- `internal/pricing/credentials.go` — `SetCredentials`, `SetCurrency`
- `internal/cost/compare.go` — `StreamWithRegions`, `StreamWithFilter`, `CompareWithFilter`
- `internal/cost/managed.go` — managed-service capability matrix
- `internal/cost/addons.go` — in-cluster substitute footprints
- `internal/provider/*/cost.go` — per-provider `EstimateMonthlyCostUSD` glue
- `internal/ui/xapiri/georegions.go` — `ApplyDataCenterLocationDefaults`, `stampGeoRegions`
- `internal/ui/xapiri/dashboard.go` — `kickRefreshCmd`, Costs tab rendering
- `internal/orchestrator/plan.go` — `planMonthlyCost`, `planCostCompare`
- ADR 0001 §DefaultsFor — CSI driver costs are not yet in the comparator
