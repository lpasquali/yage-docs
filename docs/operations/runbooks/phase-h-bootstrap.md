# Phase H bootstrap — registry and issuing CA runbook

Phase H provisions the two on-prem platform services described in
[ADR 0009](../../architecture/adrs/0009-on-prem-platform-services.md):

1. an **OCI bootstrap registry** (Harbor by default, Zot opt-in) running on a
   Proxmox VM, used as the air-gapped image mirror for CAPI / CNI / Helm
   pulls;
2. an **online intermediate (issuing) CA** signed by the operator's offline
   root, wired into a workload-cluster `cert-manager` `ClusterIssuer`.

This runbook covers what an operator must configure, what the cluster looks
like after bootstrap, how to verify each surface, the order in which to tear
the surface down on `--purge`, and which pieces of the design are not yet
implemented in the orchestrator.

!!! warning "Implementation gap — read this first"
    The two OpenTofu modules (`yage-tofu/registry/` and `yage-tofu/issuing-ca/`,
    [yage-tofu PR #6](https://github.com/lpasquali/yage-tofu/pull/6) and
    [PR #7](https://github.com/lpasquali/yage-tofu/pull/7)) are merged shapes,
    not active code paths from yage's point of view. As of commit
    [`7b9d19b`](https://github.com/lpasquali/yage/commit/7b9d19b) on
    `main`:

    - There is **no `EnsureRegistry` Go function**. Setting `YAGE_REGISTRY_*`
      configures the `--dry-run` plan output and surfaces the fields in the
      xapiri TUI ([yage commit `6504821`](https://github.com/lpasquali/yage/commit/6504821)),
      but yage does **not** apply the `yage-tofu/registry/` module and does
      **not** auto-populate `cfg.ImageRegistryMirror`. Operators must
      provision the registry VM separately (running `tofu apply` in
      `yage-tofu/registry/` by hand, or pre-staging out-of-band) and continue
      to set `YAGE_IMAGE_REGISTRY_MIRROR` explicitly.
    - `EnsureIssuingCA` exists ([yage commit `7b9d19b`](https://github.com/lpasquali/yage/commit/7b9d19b))
      but generates the intermediate CA inline using `crypto/x509`. It does
      **not** read outputs from the `yage-tofu/issuing-ca/` module. The
      module is dormant code waiting for the orchestrator migration tracked
      in [yage-tofu issue #4](https://github.com/lpasquali/yage-tofu/issues/4).

    Sections 2–4 below describe the state that is achievable today through
    `EnsureIssuingCA` plus a manually-applied registry. Section 5 calls out
    which `--purge` steps are wired and which the operator must do by hand.

---

## 1. Prerequisites — environment variables

The fields below are defined in
[`internal/config/config.go`](https://github.com/lpasquali/yage/blob/main/internal/config/config.go)
and surfaced in the `--dry-run` plan and the xapiri TUI.

### Registry VM (consumed by `yage-tofu/registry/`)

The variable wiring lives in
[`yage-tofu/registry/variables.tf`](https://github.com/lpasquali/yage-tofu/blob/registry-module/registry/variables.tf).
Setting these does **not** trigger automatic provisioning today — see the
warning above — but they are the canonical names to use when running the
module manually so that a future `EnsureRegistry` phase can take over without
reconfiguration.

| Env var | tofu variable | Required | Default | Description |
|---|---|---|---|---|
| `YAGE_REGISTRY_NODE` | `registry_node` | yes | — | Proxmox node where the registry VM is provisioned. Empty disables Phase H registry surface in `--dry-run`. |
| `YAGE_REGISTRY_VM_FLAVOR` | (mapped to `registry_vm_cores` / `registry_vm_memory_mb` / `registry_vm_disk_gb`) | no | `2` cores / `4096` MiB / `100` GiB | Operator-defined flavor name; the module exposes the three discrete inputs. Map flavor → triple before `tofu apply`. |
| `YAGE_REGISTRY_NETWORK` | `registry_network` | no | `vmbr0` | Proxmox bridge attached to the registry VM. |
| `YAGE_REGISTRY_STORAGE` | `registry_storage` | no | `local-lvm` | Proxmox storage pool for the registry root disk. |
| `YAGE_REGISTRY_FLAVOR` | `registry_flavor` | no | `harbor` | Registry implementation: `harbor` (replication-capable) or `zot` (lightweight). |
| (no env yet) | `registry_template_id` | yes | — | Proxmox VM template ID to clone (cloud-init enabled Ubuntu/Debian). Pass via `-var` when running `tofu apply` manually. |
| (no env yet) | `registry_hostname` | yes | — | DNS name used as cloud-init hostname and TLS SAN. |
| (no env yet) | `registry_tls_cert_pem` / `registry_tls_key_pem` / `registry_ca_bundle_pem` | no | empty | PEM material served by the registry. The module accepts empty values for dev mode; production must supply the chain. |
| (no env yet) | `registry_admin_password` | no | empty | Initial Harbor admin password seeded on first boot. |
| (no env yet) | `proxmox_url` / `proxmox_username` / `proxmox_password` / `proxmox_insecure` | yes | — | Proxmox API credentials. Reuse the same values as the yage Phase G identity bootstrap. |

The hostname, TLS material, admin password, and template ID are not yet
exposed as `YAGE_*` env vars in `internal/config/config.go`. When the
orchestrator migrates to consume the module these will be added; for the
manual workflow today, pass them as `-var` flags to `tofu apply`.

### Issuing CA (consumed by `yage-tofu/issuing-ca/` *and* by `EnsureIssuingCA`)

The variable wiring lives in
[`yage-tofu/issuing-ca/variables.tf`](https://github.com/lpasquali/yage-tofu/blob/feat/4-issuing-ca-module/issuing-ca/variables.tf)
and the [module README](https://github.com/lpasquali/yage-tofu/blob/feat/4-issuing-ca-module/issuing-ca/README.md).
The two PEM env vars are also read directly by
`EnsureIssuingCA` in
[`internal/platform/opentofux/issuing_ca.go`](https://github.com/lpasquali/yage/blob/main/internal/platform/opentofux/issuing_ca.go),
which is what runs today.

| Env var | tofu variable | Required | Default | Description |
|---|---|---|---|---|
| `YAGE_ISSUING_CA_ROOT_CERT` | `root_ca_cert` | yes (to enable the surface) | empty | PEM root CA cert. When empty, `EnsureIssuingCA` returns `ErrNotApplicable` and the orchestrator skips the phase silently. |
| `YAGE_ISSUING_CA_ROOT_KEY` | `root_ca_key` | yes (to enable the surface) | empty | PEM root CA key (RSA, ECDSA, or Ed25519). Sensitive — never persisted to kind Secrets (omitted from `Snapshot`). |
| (no env yet) | `cluster_name` | yes (in tofu) | — | Workload cluster name. yage's inline path uses `cfg.WorkloadClusterName` automatically. |
| (no env yet) | `validity_hours` | no | `8760` (365 d) | Intermediate validity. Mirrors `issuingCAValidityDays = 365` in `issuing_ca.go`. |
| (no env yet) | `early_renewal_hours` | no | `720` (30 d) | Renewal threshold. Not yet wired to the inline path. |

!!! danger "Root CA material handling"
    `YAGE_ISSUING_CA_ROOT_CERT` and `YAGE_ISSUING_CA_ROOT_KEY` carry
    operator-controlled offline-root material. Per ADR 0009 §4 these must
    never appear in kind Secrets, `CURRENT_STATE.md`, or any persistent
    store managed by yage. Inject from a vault / secrets manager for the
    duration of the bootstrap process, then unset them.

### Repo cache prerequisites (ADR 0010 / ADR 0011)

Both modules are cloned into the in-cluster `yage-repos` PVC by the
`yage-repo-sync` Job, created by `EnsureRepoSync` in
[`internal/platform/opentofux/fetcher.go`](https://github.com/lpasquali/yage/blob/main/internal/platform/opentofux/fetcher.go)
(yage commit
[`e7d116e`](https://github.com/lpasquali/yage/commit/e7d116e)). Operators
who plan to run the modules out-of-band can clone the repo locally instead;
the variables above apply identically.

| Env var | Default | Description |
|---|---|---|
| `YAGE_TOFU_REPO` | `https://github.com/lpasquali/yage-tofu` | Source for both modules. |
| `YAGE_TOFU_REF` | `main` | Pin to a tag for reproducibility. |
| `YAGE_REPOS_PVC_SIZE` | `500Mi` | Backing storage for the in-cluster cache. |

---

## 2. Expected post-bootstrap state in `yage-system`

After a bootstrap run with `YAGE_ISSUING_CA_ROOT_CERT` / `YAGE_ISSUING_CA_ROOT_KEY`
set (and `EnsureRepoSync` having succeeded), the management cluster's
`yage-system` namespace contains the following objects. All carry the
`app.kubernetes.io/managed-by=yage` label; see
[issue #148](https://github.com/lpasquali/yage/issues/148) for the
HandOff/VerifyParity contract that consumes that label.

| Kind | Name | Source | Notes |
|---|---|---|---|
| Namespace | `yage-system` | `EnsureYageSystemOnCluster` ([`kindsync/yagesystem.go`](https://github.com/lpasquali/yage/blob/main/internal/cluster/kindsync/yagesystem.go)) | Idempotent on every Ensure call. |
| ServiceAccount | `yage-job-runner` | `EnsureYageSystemOnCluster` | Used by `yage-repo-sync` and (future) tofu Jobs. |
| Role / RoleBinding | `yage-job-runner` | `EnsureYageSystemOnCluster` | Permits Job + Secret + ConfigMap RW within `yage-system`. |
| PersistentVolumeClaim | `yage-repos` | `EnsureRepoSync` (commit `e7d116e`) | `ReadWriteOnce`, default `500Mi`, storage class resolved by `repoSyncStorageClass()` (CSI default → Proxmox CSI → `standard`). Bound after the sync Job completes. |
| Secret | `yage-issuing-ca` | `EnsureIssuingCA` (commit `7b9d19b`) | Type `kubernetes.io/tls`. Keys: `tls.crt` (intermediate, ECDSA P-256), `tls.key`, `ca.crt` (operator-supplied root). Label `app.kubernetes.io/managed-by=yage`. |
| Secret | `tfstate-default-issuing-ca` | OpenTofu `kubernetes` backend (when the module is applied) | **Not produced by yage today** — see warning at top. Created only when an operator runs `tofu apply` against `yage-tofu/issuing-ca/` manually with the in-cluster backend. Labels `app.kubernetes.io/managed-by=yage`, `app.kubernetes.io/component=tofu-state` (per [`backend.tf`](https://github.com/lpasquali/yage-tofu/blob/feat/4-issuing-ca-module/issuing-ca/backend.tf)). |
| Secret | `tfstate-default-registry` | OpenTofu `kubernetes` backend (when the module is applied) | Same caveat. Same labels. |

!!! note "What the warning means in practice"
    Operators running yage today should expect **only** the first five rows
    on a clean run. The two `tfstate-default-*` Secrets appear only when an
    operator runs `tofu apply` against the modules out-of-band against a
    cluster that already has `yage-system` and the `yage-job-runner` RBAC
    in place. `yage --purge` does not delete those Secrets — see §5.

The workload cluster receives, in the `cert-manager` namespace:

| Kind | Name | Notes |
|---|---|---|
| Namespace | `cert-manager` | Created by `EnsureIssuingCA` even before cert-manager itself installs, so the Secret can land first. |
| Secret | `yage-issuing-ca` | Type `kubernetes.io/tls`. Keys: `tls.crt`, `tls.key` only — no `ca.crt`. Label `app.kubernetes.io/managed-by=yage`. |
| ClusterIssuer | `yage-ca-issuer` | `spec.ca.secretName: yage-issuing-ca`. Label `app.kubernetes.io/managed-by=yage`. |

---

## 3. cert-manager `ClusterIssuer` verification

Run the steps below from a shell that has both the management cluster
kubeconfig (`KUBECONFIG=$YAGE_MGMT_KCFG`) and the workload cluster
kubeconfig (`KUBECONFIG=$YAGE_WL_KCFG`) available.

### 3.1 Confirm the management-side Secret

```bash
kubectl --kubeconfig "$YAGE_MGMT_KCFG" \
  -n yage-system get secret yage-issuing-ca \
  -o jsonpath='{.metadata.labels.app\.kubernetes\.io/managed-by}{"\n"}'
# Expected: yage

kubectl --kubeconfig "$YAGE_MGMT_KCFG" \
  -n yage-system get secret yage-issuing-ca \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -dates
# Expected:
#   subject= CN = yage-issuing-ca-<workload-cluster-name>
#   issuer=  CN = <your-root-CA-CN>
#   notBefore=...  notAfter=... (≈ 365 d span)
```

### 3.2 Confirm the workload-side Secret + ClusterIssuer

```bash
kubectl --kubeconfig "$YAGE_WL_KCFG" \
  -n cert-manager get secret yage-issuing-ca \
  -o jsonpath='{.type}{"\n"}'
# Expected: kubernetes.io/tls

kubectl --kubeconfig "$YAGE_WL_KCFG" \
  get clusterissuer yage-ca-issuer -o yaml | grep -A2 'conditions:'
# Expected: type: Ready, status: "True", reason: KeyPairVerified
```

If `ClusterIssuer` is missing, cert-manager was not yet installed when
`EnsureIssuingCA` ran (the function logs a warning and continues). Re-run
the wiring with:

```bash
yage --workload-rollout
```

### 3.3 Issue a test certificate

```bash
cat <<'EOF' | kubectl --kubeconfig "$YAGE_WL_KCFG" apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: yage-issuer-smoketest
  namespace: default
spec:
  secretName: yage-issuer-smoketest-tls
  issuerRef:
    kind: ClusterIssuer
    name: yage-ca-issuer
  commonName: yage-issuer-smoketest.local
  dnsNames:
    - yage-issuer-smoketest.local
  duration: 24h
  renewBefore: 8h
EOF

kubectl --kubeconfig "$YAGE_WL_KCFG" \
  -n default wait --for=condition=Ready certificate/yage-issuer-smoketest --timeout=2m

kubectl --kubeconfig "$YAGE_WL_KCFG" \
  -n default get secret yage-issuer-smoketest-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer
# Expected: issuer= CN = yage-issuing-ca-<workload-cluster-name>
```

Clean up the smoke-test artifacts when done:

```bash
kubectl --kubeconfig "$YAGE_WL_KCFG" -n default delete certificate yage-issuer-smoketest
kubectl --kubeconfig "$YAGE_WL_KCFG" -n default delete secret yage-issuer-smoketest-tls
```

---

## 4. Registry verification

The verification below assumes a registry VM provisioned manually (today)
or by a future `EnsureRegistry` (when wired). The module outputs declared in
[`yage-tofu/registry/outputs.tf`](https://github.com/lpasquali/yage-tofu/blob/registry-module/registry/outputs.tf)
(`registry_ip`, `registry_host`, `registry_url`, `vm_id`) drive the checks
below.

### 4.1 VM reachability

```bash
REGISTRY_HOST=$(tofu -chdir=registry/ output -raw registry_host)   # if you ran the module
REGISTRY_IP=$(tofu   -chdir=registry/ output -raw registry_ip)

ping -c 2 "$REGISTRY_IP"
nc -zv "$REGISTRY_HOST" 443
```

If you provisioned by hand, substitute the operator-known hostname and IP.

### 4.2 TLS validation against the operator-supplied chain

```bash
openssl s_client -connect "$REGISTRY_HOST:443" -servername "$REGISTRY_HOST" \
  -CAfile /path/to/internal-ca-bundle.pem -showcerts </dev/null 2>/dev/null \
  | grep -E 'subject=|issuer=|Verify return code'
# Expected: Verify return code: 0 (ok)
```

The `internal-ca-bundle.pem` you pass here must be the same file referenced
by `YAGE_INTERNAL_CA_BUNDLE` (it must contain the offline root CA so the
intermediate served by the registry chains back to a trusted anchor).

### 4.3 Registry API (Harbor / Zot)

Harbor:

```bash
curl --cacert /path/to/internal-ca-bundle.pem \
  -u "admin:$YAGE_REGISTRY_ADMIN_PASSWORD" \
  "https://$REGISTRY_HOST/api/v2.0/health" | jq .
# Expected: components all "healthy"
```

Zot:

```bash
curl --cacert /path/to/internal-ca-bundle.pem \
  "https://$REGISTRY_HOST/v2/" -I
# Expected: HTTP/1.1 200 OK with Docker-Distribution-API-Version header
```

### 4.4 Confirm `cfg.ImageRegistryMirror` is in effect

Because there is no `EnsureRegistry` Go function today,
`cfg.ImageRegistryMirror` is **not** populated automatically from the tofu
output. Verify the value the orchestrator actually used by inspecting the
realized bootstrap-config Secret on the management cluster:

```bash
kubectl --kubeconfig "$YAGE_MGMT_KCFG" \
  -n yage-system get secret "${YAGE_CONFIG_NAME:-$YAGE_WORKLOAD_CLUSTER_NAME}-bootstrap-config" \
  -o jsonpath='{.data.config\.yaml}' | base64 -d \
  | grep -E '^(YAGE_IMAGE_REGISTRY_MIRROR|YAGE_HELM_REPO_MIRROR|YAGE_INTERNAL_CA_BUNDLE)='
# Expected: YAGE_IMAGE_REGISTRY_MIRROR=https://<your-registry-host>
```

The same value must match `cfg.ImageRegistryMirror` printed by `--dry-run`
under "Phase H — registry":

```bash
yage --dry-run | grep -A6 'Registry node'
# Expected: 'image registry mirror: https://...' line aligned to YAGE_IMAGE_REGISTRY_MIRROR.
```

If the line is empty, you forgot to set `YAGE_IMAGE_REGISTRY_MIRROR`. yage
does not derive it from `YAGE_REGISTRY_NODE` today.

### 4.5 Pull through the mirror

The fastest way to confirm the registry is fronting the mirror is to
re-pull a CAPI image on a workload node and check the pull origin:

```bash
ssh ubuntu@<worker-node-ip> -- 'sudo crictl pull $YAGE_IMAGE_REGISTRY_MIRROR/cluster-api/cluster-api-controller:v1.11.8'
ssh ubuntu@<worker-node-ip> -- 'sudo crictl images | grep cluster-api-controller'
# Expected: image listed with the registry-mirror prefix.
```

---

## 5. `--purge` teardown ordering

`yage --purge` drives `PurgeGeneratedArtifacts` in
[`internal/orchestrator/purge.go`](https://github.com/lpasquali/yage/blob/main/internal/orchestrator/purge.go),
which executes — in order — workload-cluster deletion, the CAPI manifest
Secret cleanup, the per-provider `Provider.Purge` (Proxmox runs the BPG
identity `tofu destroy`), then file/disk cleanup. **Phase H surfaces are
not yet integrated.** Operators must teardown manually in the order below
to avoid orphan VMs and orphan tofu state.

The two recently-introduced orchestrator commits anchor the surface:

- `EnsureRepoSync` ([`e7d116e`](https://github.com/lpasquali/yage/commit/e7d116e))
  creates the `yage-repos` PVC and the `yage-repo-sync` Job that clones
  `yage-tofu` and `yage-manifests` into it.
- `EnsureIssuingCA` ([`7b9d19b`](https://github.com/lpasquali/yage/commit/7b9d19b))
  creates the `yage-issuing-ca` Secret on the management cluster and the
  Secret + `ClusterIssuer` on the workload.

### Recommended teardown order

1. **Stop traffic** to the registry (set `Airgapped=false` and unset
   `YAGE_IMAGE_REGISTRY_MIRROR` / `YAGE_HELM_REPO_MIRROR` for any new
   workloads), or shut the workload cluster down first if you are tearing
   the whole environment.

2. **Workload cluster**: delete in this order so cert-manager doesn't
   re-issue against an about-to-be-revoked Secret:

    ```bash
    kubectl --kubeconfig "$YAGE_WL_KCFG" delete clusterissuer yage-ca-issuer --ignore-not-found
    kubectl --kubeconfig "$YAGE_WL_KCFG" -n cert-manager delete secret yage-issuing-ca --ignore-not-found
    ```

3. **Issuing CA tofu state** (only if you applied the module manually):

    ```bash
    cd /path/to/yage-tofu/issuing-ca/
    tofu destroy \
      -var="cluster_name=$YAGE_WORKLOAD_CLUSTER_NAME" \
      -var="root_ca_cert=$YAGE_ISSUING_CA_ROOT_CERT" \
      -var="root_ca_key=$YAGE_ISSUING_CA_ROOT_KEY"
    ```

    This deletes the `tfstate-default-issuing-ca` Secret in `yage-system`.
    The `ClusterIssuer` was already removed in step 2.

4. **Management-cluster issuing CA Secret**: removed automatically by
    `--purge` only when the management cluster is the `kind-<KindClusterName>`
    cluster being deleted. After a pivot, the post-pivot management cluster
    is **not** torn down by `--purge`, so the `yage-issuing-ca` Secret
    persists in `yage-system` until the operator deletes it:

    ```bash
    kubectl --kubeconfig "$YAGE_MGMT_KCFG" \
      -n yage-system delete secret yage-issuing-ca --ignore-not-found
    ```

5. **Run `yage --purge`** for the rest of the surface (workload cluster
   deletion, CAPI manifest cleanup, Proxmox identity `tofu destroy`, kind
   teardown):

    ```bash
    yage --purge
    ```

6. **Registry tofu state and VM** (only if you applied the module
   manually). Run **after** `yage --purge` so any remaining workload-node
   image pulls have completed:

    ```bash
    cd /path/to/yage-tofu/registry/
    tofu destroy \
      -var="proxmox_url=$PROXMOX_URL" \
      -var="proxmox_username=$PROXMOX_USERNAME" \
      -var="proxmox_password=$PROXMOX_PASSWORD" \
      -var="cluster_name=$YAGE_WORKLOAD_CLUSTER_NAME" \
      -var="registry_node=$YAGE_REGISTRY_NODE" \
      -var="registry_template_id=$YAGE_REGISTRY_TEMPLATE_ID" \
      -var="registry_hostname=<as-supplied>"
    ```

    !!! warning "Backend availability"
        The `kubernetes` backend stores tofu state in `yage-system` on the
        management cluster. After step 5 the kind cluster is gone — if you
        had not pivoted, your registry-module state is gone too. Either:

        - run step 6 **before** `yage --purge` (i.e. swap steps 5 and 6),
          accepting that workload pulls may fail mid-teardown; or
        - run the registry module with `-backend=false` from the start, so
          state lives in a local `terraform.tfstate` file you can keep
          across kind teardown.

7. **Verify no orphans on Proxmox**:

    ```bash
    pvesh get /cluster/resources --type vm --output-format json \
      | jq -r '.[] | select(.tags | contains("yage")) | "\(.vmid)\t\(.name)"'
    # Expected: empty
    ```

### Ordering rationale

- Workload `ClusterIssuer` first: avoids cert-manager attempting to
  renew certificates against a Secret that is about to vanish.
- Issuing-CA tofu destroy before `yage --purge`: the kubernetes backend
  Secret lives on the management cluster; once `yage --purge` deletes the
  kind management cluster, the state Secret is gone with it and `tofu
  destroy` will fail with a "no state" error.
- Registry tofu destroy last: the registry must remain reachable while
  the workload cluster is being torn down, in case CAPI controllers
  reach for any remaining images during cleanup.
- The Proxmox identity `tofu destroy` runs inside `yage --purge` via
  `Provider.Purge` — operators do not need to invoke it explicitly.

---

## 6. Known gaps

These are the deltas between ADR 0009 / the merged tofu modules and what
the orchestrator actually does today. Operators should plan around them.

### 6.1 `yage-tofu/issuing-ca/` is dormant

`EnsureIssuingCA` ([`internal/platform/opentofux/issuing_ca.go`](https://github.com/lpasquali/yage/blob/main/internal/platform/opentofux/issuing_ca.go),
commit [`7b9d19b`](https://github.com/lpasquali/yage/commit/7b9d19b))
generates the intermediate CA inline using `crypto/x509` and never invokes
the `yage-tofu/issuing-ca/` module. As a result:

- there is no `tfstate-default-issuing-ca` Secret in `yage-system` after
  a normal `yage` run;
- the intermediate CA is regenerated **on every run** when no existing
  `yage-issuing-ca` Secret is found — there is no early-renewal logic in
  the inline path;
- `validity_hours` and `early_renewal_hours` (declared in the module)
  have no effect today; `issuingCAValidityDays = 365` is hard-coded in
  Go.

The migration is tracked by
[yage-tofu issue #4](https://github.com/lpasquali/yage-tofu/issues/4).
Once it lands, `EnsureIssuingCA` will read the four module outputs
(`intermediate_cert_pem`, `intermediate_key_pem`, `ca_chain_pem`,
`root_ca_cert_pem`) instead of generating the material in-process, and
the tofu state Secret will appear after the first run.

### 6.2 `yage-tofu/registry/` has no Go consumer

There is no `EnsureRegistry` function in
`internal/platform/opentofux/`. Setting `YAGE_REGISTRY_*` configures
`--dry-run` output and the xapiri TUI ([yage commit
`6504821`](https://github.com/lpasquali/yage/commit/6504821)) but does
**not** apply the module. Consequences:

- `cfg.ImageRegistryMirror` is **not** auto-populated. Operators must
  continue to set `YAGE_IMAGE_REGISTRY_MIRROR` to the registry URL by
  hand.
- The `tfstate-default-registry` Secret never appears unless the operator
  runs `tofu apply` against the module manually.
- Registry image-seeding is fully manual (this is true even after
  `EnsureRegistry` lands — see ADR 0009 §1, Phase H scope explicitly
  excludes a `--seed-registry` flag).

The orchestrator integration is tracked by
[yage issue #125](https://github.com/lpasquali/yage/issues/125).

### 6.3 `--purge` does not destroy Phase H tofu state

Per §5, `PurgeGeneratedArtifacts` runs the Proxmox identity `tofu destroy`
through `Provider.Purge` but does not touch the registry or issuing-ca
modules. Operators must run those `tofu destroy` calls manually, in the
order documented above.

### 6.4 Hostname / TLS / template-ID env vars are not exposed

The registry module requires `registry_hostname`, `registry_template_id`,
and the three TLS PEM inputs, but no `YAGE_*` env vars exist for them in
`internal/config/config.go`. Operators must pass them as `-var` flags
when running `tofu apply` manually. These will be added when the
`EnsureRegistry` orchestrator integration lands.

### 6.5 ADR 0009 § state location is outdated

ADR 0009 §1 describes registry state at
`~/.yage/tofu/registry/terraform.tfstate`. That predates the kubernetes
backend migration ([yage PR `1c2365d`](https://github.com/lpasquali/yage/commit/1c2365d),
issue #145). Both modules' `backend.tf` now declare the kubernetes
backend; state lives in `yage-system` Secrets. Trust the
[`backend.tf`](https://github.com/lpasquali/yage-tofu/blob/feat/4-issuing-ca-module/issuing-ca/backend.tf)
files over the ADR text for the current location.

---

## References

- [ADR 0009 — On-Prem Platform Services (Phase H)](../../architecture/adrs/0009-on-prem-platform-services.md)
- [ADR 0010 — In-cluster repository cache](../../architecture/adrs/0010-in-cluster-repo-cache.md)
- [ADR 0011 — Pivot: yage state migration](../../architecture/adrs/0011-pivot-yage-state-migration.md)
- yage-tofu PR [#6](https://github.com/lpasquali/yage-tofu/pull/6) — registry module
- yage-tofu PR [#7](https://github.com/lpasquali/yage-tofu/pull/7) — issuing-ca module
- yage commit [`e7d116e`](https://github.com/lpasquali/yage/commit/e7d116e) — `EnsureRepoSync`
- yage commit [`7b9d19b`](https://github.com/lpasquali/yage/commit/7b9d19b) — `EnsureIssuingCA`
- yage commit [`6504821`](https://github.com/lpasquali/yage/commit/6504821) — xapiri + plan surfaces
- [`internal/platform/opentofux/issuing_ca.go`](https://github.com/lpasquali/yage/blob/main/internal/platform/opentofux/issuing_ca.go)
- [`internal/platform/opentofux/fetcher.go`](https://github.com/lpasquali/yage/blob/main/internal/platform/opentofux/fetcher.go)
- [`internal/orchestrator/purge.go`](https://github.com/lpasquali/yage/blob/main/internal/orchestrator/purge.go)
