# CODING_STANDARDS

## Formatting and linting

- **Go**: Use `gofmt` and `goimports` for formatting. Run `staticcheck ./...` before opening a PR.
- The Go binary may not be on PATH — use `make` targets or the full path `/usr/local/go/bin/go`.

## Testing

- Run tests with `make test` (`go test ./...`). Run a single package: `go test ./internal/<pkg>/... -run <TestName> -v`.
- Prefer **table-driven tests** for functions with multiple input/output cases.
- Mock at system boundaries (external HTTP, shell commands) — do not mock internal helpers.
- Do not write tests that only verify happy-path with no edge-case or error-path coverage.

## Logging

Use `logx.Log`, `logx.Warn`, `logx.Die` — never `fmt.Println` or `log.*` directly:

| Function | Prefix | Use |
|---|---|---|
| `logx.Log` | `✅ 🎉` | Normal progress |
| `logx.Warn` | `⚠️ 🙈` | Non-fatal warnings |
| `logx.Die` | `❌ 💩` | Fatal errors (calls os.Exit) |

`logx.Die` is for unrecoverable failures only. Functions below the orchestrator level must return `error`, not call `logx.Die`.

## Shell execution

Use `shell.Run`, `shell.Capture`, `shell.RunWithEnv`, `shell.Pipe` — never raw `os/exec`. The `shell` package handles `RUN_PRIVILEGED` (sudo elevation) and stderr routing. The handful of intentional `kubectl` shell-outs (kustomize apply, port-forward, streaming) are documented with rationale comments at each call site.

## YAML document handling

Multi-document YAML manifests (separated by `\n---\n`) are the primary workload manifest format.

- **Always split by `\n---\n` first**, then match within each document.
- Use **line-anchored regexes** (`^kind:\s*ProxmoxCluster\s*$`) for kind matching — never `strings.Contains`. The `infrastructureRef` field in Cluster objects contains `kind: ProxmoxCluster` as a nested value; `strings.Contains` produces false positives.

## Error handling

- Return `error` for recoverable failures; use `logx.Die` only in top-level orchestrator or CLI entry points.
- Use `ErrNotApplicable` (not `nil`, not `errors.New`, not panics) for provider phases that don't apply to a given provider.
- Retry loops (e.g., manifest apply): 3 attempts with 10s backoff is the established pattern.

## Comments

Default to **no comments**. Add one only when the WHY is non-obvious: a hidden constraint, a subtle invariant, a workaround for a specific external bug, behavior that would surprise a reader. Never describe what the code does — well-named identifiers do that. Never reference the current task, fix, or PR.

## Provider plugins

New providers self-register via `init()`:

```go
func init() { provider.Register("myprovider", func() provider.Provider { return &Provider{} }) }
```

Embed `provider.MinStub` for phases that don't apply. Return `ErrNotApplicable` — not errors or panics — for inapplicable phases. Reference implementation: `internal/provider/capd/capd.go`.

## Config and state

- All config comes from `internal/config/config.go` (env vars → CLI flags → kind Secret merge). Do not add hardcoded defaults outside this file.
- State lives in kind Secrets in `yage-system` — not local disk. `internal/cluster/kindsync` owns the round-trip.
- Do not add new config fields without a corresponding env-var name and CLI flag.

## Kubernetes clients

All cluster interactions go through `internal/platform/k8sclient` (typed + dynamic clients keyed by kubecontext/kubeconfig). Do not open new `client-go` clients outside this package.

## Security

- Do not commit secrets or tokens.
- Use environment variables for sensitive configuration.
- `kubectl port-forward`: use `--address 127.0.0.1` + `PORT:443` format — the `127.0.0.1:PORT:443` triple is not supported in modern kubectl.

## State Integrity (SSOT)

- **Single Source of Truth**: All project-specific WIP, active branches, known bugs, and inter-agent handoff state must reside in `yage-docs/docs/context/CURRENT_STATE.md`.
- **No External State**: Do not store project-level facts in an agent's global/personal memory.
- **Atomic Persistence**: Every task session that changes system state must end with an update to `CURRENT_STATE.md`.
