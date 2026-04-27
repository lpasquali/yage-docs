# yage Documentation

Welcome to the central documentation for **yage** — a Go tool that bootstraps Kubernetes Cluster API (CAPI) environments on any of twelve registered infrastructure providers.

## Context (AI Memory)

- **[Agent System Prompt](context/AGENT_SYSTEM_PROMPT.md)** — Core identity, constraints, and guidance for AI agents working in the yage codebase.

## Architecture

- **[Overview](architecture/ARCHITECTURE.md)** — Component diagrams, package layout, and bootstrap phase timeline.
- **[Architectures](architecture/architectures.md)** — Multi-architecture build details.
- **[Abstraction Plan ADR](architecture/adrs/abstraction-plan.md)** — Multi-cloud provider abstraction phases (C, E, B, A, D).

## Operations

- **[Capacity Preflight](operations/capacity-preflight.md)** — Host capacity planning and trichotomy verdicts.
- **[Cost and Pricing](operations/cost-and-pricing.md)** — Live FinOps cost estimation, `--cost-compare`, currency.

## Reference

- **[Providers](reference/providers.md)** — All 12 infrastructure providers and their configuration.
