# Contributing to yage-docs

Thanks for considering a contribution. This repository contains the central documentation for the **yage** project. For the source code, see [lpasquali/yage](https://github.com/lpasquali/yage).

## Before you open a PR

- Pick the DoD level from the PR template and tick its boxes.
- **Level 3 — Documentation**: `mkdocs build --strict` must pass.
- CI enforces all checks via the `Merge Gate` compliance job.

## Development setup

```bash
pip install -r requirements.txt
mkdocs serve       # live preview at http://127.0.0.1:8000
mkdocs build --strict   # full build with strict link checking
```

## Security disclosure

Please do **not** open public issues for security vulnerabilities. See [SECURITY.md](SECURITY.md).
