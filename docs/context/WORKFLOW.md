# yage — Workflow: SOP, Definition of Done, Audit Triggers

Adapted from the rune-docs SOP. The canonical source for cross-project standards is
`~/Devel/rune-docs/docs/context/SYSTEM_PROMPT.md`; this file adapts those standards
for yage's Go/Kubernetes/CAPI stack.

---

## Anti-Rogue Constraint (non-negotiable)

Do **not** begin the Execute phase (writing or modifying code) until the chat confirms:

1. **SOP Step 1 (Assign)** — A GitHub issue exists for this work, assigned to `lpasquali`.
2. **SOP Step 2 (Isolate)** — You are on a dedicated feature branch for this task only.

**Halt and ask the user for permission** before Execute, even in autonomous mode.

---

## SOP: issue → merge

1. **Assign** — Ensure a GitHub issue exists. Assign to `lpasquali`. Add `claude_cli` label. Add to Project #1 and set **Status → In progress**.
2. **Isolate** — Create a feature branch from `main`. One branch per issue. Never modify another agent's branch.
3. **Research** — Read `AGENT_SYSTEM_PROMPT.md`, `CURRENT_STATE.md`, `CODING_STANDARDS.md`, relevant ADRs, and the affected code.
4. **Halt** — Confirm steps 1–2 are done in chat; get user OK to Execute.
5. **Execute** — Minimal scope. Strong tests. No features beyond the issue scope.
6. **Verify** — `make build && go test ./...` green. `go vet ./...` clean. Coverage not degraded. No new `govulncheck` findings.
7. **E2E** — For orchestrator or provider changes: manually walk through the affected bootstrap phase on a local kind cluster (or confirm the changed path is covered by the existing test suite). Document the verification in the PR body.
8. **PR** — Use the PR template. **Rebase on latest `main` before opening the PR and before each merge attempt** — if the branch goes BEHIND after another PR merges, rebase + force-push immediately (before attempting the merge), not after a failed merge attempt. This avoids triggering a second CodeQL scan. Wait for CI green (Quality Gates + CodeQL). PR body must include: `Closes #NNN`, one DoD level checked, Acceptance Criteria Evidence, Audit Checks.
9. **Persist** — Update `CURRENT_STATE.md` after merge. Close the issue. If the work spawned follow-up items, open child issues before closing the parent.

---

## Definition of Done (pre-PR)

Pick the **highest** applicable level. **CI green alone is not enough.**

### Level 1 — Full Validation
*Applies to: orchestrator phases, provider implementations, config changes, CSI drivers, kindsync, any code that runs against a real cluster.*

- [ ] `make build` passes
- [ ] `go test ./...` passes (all packages)
- [ ] `govulncheck ./...` — no new CVEs (if `go.mod` changed)
- [ ] Manually walked through the affected flow against a local kind cluster **or** the change is fully covered by the existing test suite (cite the test)
- [ ] Breaking-change audit: Secret schemas, config field renames, env var renames, CAPI CRD field changes — note in PR body

### Level 2 — Test / CI / Config Only
*Applies to: test changes, CI config, linter/vet config, coverage improvements — no runtime code changed.*

- [ ] Full test suite passes
- [ ] Coverage not degraded
- [ ] No unintended CI side effects

### Level 3 — Documentation Only
*Applies to: `yage-docs/` content, ADRs, runbooks, diagrams — no Go code changed.*

- [ ] `mkdocs build --strict` passes (in `yage-docs/`)
- [ ] No broken internal links

---

## Audit Triggers

When a change matches a row, run the check **before** opening the PR. `FAIL → no PR`.

| Change | Required check |
|---|---|
| `go.mod` / `go.sum` modified | `govulncheck ./...` — no new findings; paste summary in PR Audit Checks table |
| New external HTTP call / API integration | Review auth model; note in PR body |
| `.github/workflows/` modified | Review for supply-chain risk (pinned action SHAs, no `pull_request_target` without explicit scope) |
| Secret schema change (kindsync, provider state) | PE review required (per ADR 0002) |
| CAPI manifest YAML patching | PE review required (line-anchored regex, CAPI v1beta2 strict decoding) |
| New binary dependency in Makefile / installer | `govulncheck` + license check |

---

## PR Template compliance

Every PR body must:

- Reference `Closes #NNN` (the issue number)
- Check **exactly one** DoD level
- Fill the **Acceptance Criteria Evidence** section (CI logs count for automated checks; screenshots or command output for manual verification)
- Fill **Audit Checks** (`No triggers fired.` is a valid value if no trigger applies)
- Note **Breaking Changes** (`None.` if none)

See `.github/PULL_REQUEST_TEMPLATE.md` in the yage repo.

---

## GitHub Project Board

- Project: `#1` on `lpasquali/yage`
- Status transitions:
  - **Todo** → when item added
  - **In progress** → you set when Assign + Isolate complete
  - **Done** → auto on issue close / PR merge
- Agent lane is set by `*_cli` label (e.g. `claude_cli`). Only one `*_cli` label per issue.

---

## Issue closure rules

- Do **not** close an issue while any linked PR (including Draft) is open.
- For epics: close only when every child issue is closed and all linked PRs are merged.

---

## CURRENT_STATE.md update (after merge)

After every PR merge, update `yage-docs/docs/context/CURRENT_STATE.md`:

1. Add a dated entry under **Recent Changes** with PR number, summary, and follow-up items.
2. Update **Active Work** table — remove merged branches, add new ones.
3. Update **Next Steps** and **Known Issues** if anything changed.
