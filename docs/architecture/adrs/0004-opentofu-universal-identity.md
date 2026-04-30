# ADR 0004 — Universal OpenTofu Identity Bootstrap (Phase G)

**Status:** Proposed
**Date:** 2026-04-30
**Owners:** Backend (HCL templates + EnsureIdentity wiring), Architect (interface contract)

---

## Context

Proxmox is the only provider where yage currently mints credentials at bootstrap time
(`internal/platform/opentofux/` uses the BPG Terraform provider to create PVE users,
tokens, and ACLs). Every other provider returns `ErrNotApplicable` from `EnsureIdentity`
and expects the operator to supply credentials out-of-band.

Abstraction-plan §21.2 documents that seven providers have Terraform/OpenTofu providers
that yage could use to mint credentials programmatically:

| Provider | Credential minted | TF provider |
|---|---|---|
| AWS | IAM user + access key (or role for IRSA) | `hashicorp/aws` |
| Azure | Service Principal + client secret | `hashicorp/azurerm` |
| GCP | Service Account + JSON key | `hashicorp/google` |
| OpenStack | Application Credential (Keystone) | `terraform-provider-openstack` |
| OCI | API key pair | `oracle/oci` |
| IBMCloud | Service ID + API key | `ibm-cloud/ibm` |
| Linode | Personal Access Token (scoped) | `linode/linode` |

The pattern established by `opentofux` is:
1. Write a provider-specific `.tf` template to a temp dir.
2. Run `tofu init` + `tofu apply`.
3. Read outputs, write to kind bootstrap Secret via `kindsync`.
4. On `--purge`, run `tofu destroy`.

## Decision

_Placeholder — to be completed by the Architect in a follow-up session._

Key questions to resolve:
- Should each provider get a standalone HCL file embedded in its package (like `proxmox/hcl.go`)?
- Is there a shared runner in `opentofux` that is parameterized, or do providers each call `tofu` directly?
- Where does state live? Kind Secret, same as the Proxmox path, or a per-provider subdirectory of `<kind-data-dir>/tofu-<provider>/`?
- How does the interface signal that Phase G is active (`cfg.Providers.<X>.TofuManaged bool`)?
- Which providers are opt-in (operator must set flag) vs. default-on when credentials are absent?

## Consequences

_To be filled after decision is made._

## References

- abstraction-plan.md §21.2 — full design narrative
- `internal/platform/opentofux/` — Proxmox reference implementation
- Issue #80 — OpenStack `EnsureIdentity` clouds.yaml templating (sequenced before Phase G)
