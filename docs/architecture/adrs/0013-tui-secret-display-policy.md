# ADR 0013 — TUI Secret-Display Policy

**Status:** Accepted
**Date:** 2026-05-01
**Owners:** Architect (contract definition), Frontend (implementation, yage issue #167)

---

## Context

yage PR #161 (`d6e7d0a`) added Proxmox admin-token input to the xapiri TUI
with `EchoMode = textinput.EchoPassword` on the focused (typing) path.
However, the unfocused render path in `renderField` (dashboard.go line 3725)
calls `ti.Value()` directly, bypassing `EchoMode` entirely — the raw token
value is painted to the terminal whenever the field is not active. The same
defect affects the issuing-CA PEM fields (`tiIssuingCACert`,
`tiIssuingCAKey`), which have no `EchoPassword` at all.

Three concrete offending sites exist today:

1. **`dashboard.go:3725`** — `renderField` unfocused branch calls `ti.Value()`
   and interpolates the raw string. Affects every secret `textinput.Model`
   in the `textInputs` array when not focused.
2. **`dashboard.go:828–829`** — `tiIssuingCACert` / `tiIssuingCAKey` are seeded
   from `cfg.IssuingCARootCert` / `cfg.IssuingCARootKey` with no `EchoPassword`
   set. The raw PEM block (including the private key) is rendered in clear text.
3. **`dashboard.go:760`** — `tiProxmoxAdminToken` correctly sets `EchoPassword`
   for the focused path, but the unfocused `renderField` path at line 3725
   still calls `ti.Value()`, defeating the protection.

This ADR establishes the cross-cutting policy that closes these gaps and
prevents future regressions.

---

## Decision

### 1. Hard rule — secret values never displayed

Secret values **must never** be rendered in the TUI, written to logs, or
echoed in error messages. Violations are security bugs with p1 priority,
regardless of the code path that causes them.

### 2. Display contract — insertion-status indicator only

When a secret field is **not focused**, the TUI must render only an
insertion-status indicator. No value, no length, no first/last characters,
no preview of any kind.

The canonical form is:

| State | Rendered string |
|---|---|
| Value is set (non-empty) | `[✓] set` |
| Value is not set (empty) | `[ ] not set` |

These strings are literal. Implementations must not substitute synonyms,
abbreviations, or alternative glyphs.

When a secret field **is focused** (the user is actively typing), the input
must use `textinput.EchoPassword` so characters are masked during entry.

### 3. Editing contract — mandatory `EchoPassword`

Every text-input widget bound to a secret field must have:

```go
ti.EchoMode = textinput.EchoPassword
ti.EchoCharacter = '·'
```

set before the first call to `ti.View()`. Setting `EchoMode` after `SetValue`
is correct — the focused render path will mask the value. The unfocused
render path must **not** call `ti.Value()` for secret fields; it must emit
the insertion-status indicator instead.

### 4. Logging contract — explicit ban

The following must never appear in code that handles a secret field:

- `fmt.Printf` / `fmt.Fprintf` / `fmt.Sprintf` of a secret value
- `tea.Println` of a secret value
- `log.*` / `slog.*` of a secret value
- `logx.Log` / `logx.Warn` / `logx.Die` of a secret value

Error messages that reference a secret field must identify the field
**by name** (e.g. `"AdminToken is required"`), never by value.

### 5. Secret field registry

The following `config.Config` struct fields are classified as secrets and
are subject to all four rules above:

| Field path | Config struct field | Env var |
|---|---|---|
| `Providers.Proxmox.AdminToken` | `ProxmoxConfig.AdminToken` | `PROXMOX_ADMIN_TOKEN` |
| `Providers.Proxmox.CAPIToken` | `ProxmoxConfig.CAPIToken` | `PROXMOX_TOKEN` |
| `Providers.Proxmox.CAPISecret` | `ProxmoxConfig.CAPISecret` | `PROXMOX_SECRET` |
| `Providers.Proxmox.CSITokenID` | `ProxmoxConfig.CSITokenID` | `PROXMOX_CSI_TOKEN_ID` |
| `Providers.Proxmox.CSITokenSecret` | `ProxmoxConfig.CSITokenSecret` | `PROXMOX_CSI_TOKEN_SECRET` |
| `Providers.Vsphere.Password` | `VsphereConfig.Password` | `VSPHERE_PASSWORD` |
| `Providers.Hetzner.Token` | `HetznerConfig.Token` | `HCLOUD_TOKEN` |
| `Cost.Credentials.AWSSecretAccessKey` | `CostCredentials.AWSSecretAccessKey` | (no ambient fallback) |
| `Cost.Credentials.GCPAPIKey` | `CostCredentials.GCPAPIKey` | `YAGE_GCP_API_KEY` |
| `Cost.Credentials.HetznerToken` | `CostCredentials.HetznerToken` | `YAGE_HCLOUD_TOKEN` |
| `Cost.Credentials.DigitalOceanToken` | `CostCredentials.DigitalOceanToken` | `YAGE_DO_TOKEN` |
| `Cost.Credentials.LinodeToken` | `CostCredentials.LinodeToken` | `YAGE_LINODE_TOKEN` |
| `Cost.Credentials.IBMCloudAPIKey` | `CostCredentials.IBMCloudAPIKey` | `YAGE_IBMCLOUD_API_KEY` |
| `IssuingCARootCert` | `Config.IssuingCARootCert` | `YAGE_ISSUING_CA_ROOT_CERT` |
| `IssuingCARootKey` | `Config.IssuingCARootKey` | `YAGE_ISSUING_CA_ROOT_KEY` |
| `BootstrapKindBackupPassphrase` | `Config.BootstrapKindBackupPassphrase` | `BOOTSTRAP_KIND_BACKUP_PASSPHRASE` |

`AWSAccessKeyID` (a non-secret identifier) and `Vsphere.Username` (a
non-secret login name) are **not** classified as secrets and do not require
masking.

`IssuingCARootCert` is classified as a secret because it contains PEM
private-key material when the operator supplies the root CA certificate
together with its key. The cert itself is public, but given the field
co-locates with `IssuingCARootKey` in the TUI and both arrive as multi-line
PEM blobs, treating the cert field as secret too avoids accidental display
of the adjacent key.

### 6. Scope

This policy applies to:

- All tabs and panels in `internal/ui/xapiri/` (provision tab, costs tab,
  token re-prompt overlay, any future panel).
- All future xapiri features.
- Any future yage TUI surfaces outside xapiri.

It does **not** apply to non-interactive outputs (dry-run plan, `--help`,
log lines, Kubernetes Secret objects) — those are governed separately by
`CODING_STANDARDS.md §Security` and kindsync's snapshot exclusion rules.

---

## Consequences

### Positive

- Secret values can never leak through the terminal even if the operator
  shares a screen recording or screenshot.
- The insertion-status indicator is sufficient for the operator to confirm
  that a credential has been entered without revealing the value.
- A consistent visual language (`[✓] set` / `[ ] not set`) applies uniformly
  across all secret fields.

### Negative

- The unfocused `renderField` path must branch on whether a field is a
  secret. This couples the renderer to the field classification table.
  Mitigation: add a `secret bool` attribute to the `dashFieldMeta` struct
  so the renderer stays data-driven.

### Risks

- **Future field additions.** A new config field that carries a secret value
  but is not added to the registry here will not benefit from the protection
  until a PR explicitly updates both the field and this ADR. Mitigation:
  architecture review of any new `cfg.*` field that holds a token, key, or
  password must verify whether it belongs in §5 and, if so, update this
  document before landing the feature.

---

## References

- yage issue #167 — `ui: mask all secret fields in xapiri (security)`
- yage PR #161 (`d6e7d0a`) — introduced `tiProxmoxAdminToken` with partial protection
- `internal/ui/xapiri/dashboard.go` — `renderField` (line 3707), `tiProxmoxAdminToken` init (line 759), `tiIssuingCACert`/`tiIssuingCAKey` init (lines 828–829)
- `internal/config/config.go` — `ProxmoxConfig`, `VsphereConfig`, `HetznerConfig`, `CostCredentials`, `Config.IssuingCARootCert/Key`, `Config.BootstrapKindBackupPassphrase`
- `CODING_STANDARDS.md §Security` — secret handling at the shell and config layers
