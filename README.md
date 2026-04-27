# yage Documentation Hub

Welcome to the central documentation for **yage** — a Go tool that bootstraps Kubernetes Cluster API (CAPI) environments on any of twelve registered infrastructure providers (Proxmox VE, AWS, Azure, GCP, Hetzner, OpenStack, vSphere, CAPD, DigitalOcean, Linode, OCI, IBM Cloud).

## Context (AI Memory)

Instructions, state, and rules for LLMs and agents.

- **[AGENT_SYSTEM_PROMPT](docs/context/AGENT_SYSTEM_PROMPT.md)** — Core identity, constraints, and guidance for AI agents working in the yage codebase.

## Architecture

How yage is designed internally.

- **[Architecture Overview](docs/architecture/ARCHITECTURE.md)** — Component diagrams, package layout, and bootstrap phase timeline.
- **[Architectures](docs/architecture/architectures.md)** — Multi-architecture build details.
- **[Abstraction Plan](docs/architecture/adrs/abstraction-plan.md)** — ADR: multi-cloud provider abstraction (Phases C, E, B, A, D).

## Operations

How yage is configured and run.

- **[Capacity Preflight](docs/operations/capacity-preflight.md)** — Host capacity planning and trichotomy verdicts (fits / tight / abort).
- **[Cost and Pricing](docs/operations/cost-and-pricing.md)** — Live FinOps cost estimation, currency conversion, and `--cost-compare`.

## Reference

- **[Providers](docs/reference/providers.md)** — All 12 infrastructure providers, their status, and configuration reference.

---

*For the yage source code, see [lpasquali/yage](https://github.com/lpasquali/yage).*
