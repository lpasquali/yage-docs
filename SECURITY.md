# Security Policy

We take the security of `yage` very seriously.

## Supported Versions

yage is currently in **pre-release development** and is not intended for production use.

| Version | Supported | Notes |
|---|---|---|
| `main` (latest) | Best-effort | Critical and high vulnerabilities (CVSS >= 7.0) addressed promptly |
| Any older build | Not supported | Use latest |

## Reporting a Vulnerability

If you discover a security vulnerability within `yage`, please do **not** open a public issue.

Instead, please send an e-mail to **[luca@bucaniere.us]**.

All security vulnerabilities will be promptly addressed. We will try to get back to you within 48 hours to acknowledge the report.

## Mandatory Merge Protection

The repository enforces branch protection on `main` — pull requests cannot be merged when required checks fail.

- `Merge Gate` is a required status check.
- SBOM is generated and scanned on every PR.
- If any fixable vulnerability has CVSS score > 8.8, CI fails and PR merge is blocked.
