# Architecture

Status: Intended | Owner: TBD

This folder holds the cross-repository architecture of A·Spectra: what each repository owns,
the rules for changes, and the decisions behind them. The internal architecture of each
application lives in its own repository.

| Document | Purpose |
|---|---|
| [repository-boundaries.md](repository-boundaries.md) | What ASpectra and each application repository own, and how they depend on each other |
| [platform.md](platform.md) | Product, repositories, end-to-end flow, cross-repository mechanics, local stack |
| [repository-rules.md](repository-rules.md) | Canonical cross-repository rules R1-R16 |
| [../diagrams/](../diagrams/README.md) | Architecture diagrams (Unverified) |

## Where the product architecture is described today

The product definition, repository layout, end-to-end flow and cross-repository mechanics are in
[platform.md](platform.md). Product scope is defined in [../alcance.md](../alcance.md), which wins on any
disagreement. Engineering rules, AI limits and guardrails G1-G36 are in
[../.claude/rules/engineering-rules.md](../../.claude/rules/engineering-rules.md).
