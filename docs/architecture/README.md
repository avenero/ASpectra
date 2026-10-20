# Architecture

Status: Intended | Owner: TBD

This folder holds the cross-repository architecture of A·Spectra: what each repository owns,
the rules for changes, and the decisions behind them. The internal architecture of each
application lives in its own repository.

| Document | Purpose |
|---|---|
| [repository-boundaries.md](repository-boundaries.md) | What ASpectra and each application repository own, and how they depend on each other |
| [repository-rules.md](repository-rules.md) | Canonical cross-repository rules R1-R16 |
| [decisions/](decisions/README.md) | Architecture decision records (ADRs) |
| [../diagrams/](../diagrams/README.md) | Architecture diagrams (Unverified) |

## Where the product architecture is described today

The product definition, end-to-end flow, cross-repository mechanics, working rules, AI
limits and guardrails G1-G36 are currently maintained in [../../.claude/CLAUDE.md](../../.claude/CLAUDE.md).
That file stays authoritative for them until a separately approved change moves them here.
Their accuracy has not been re-verified (see [../README.md](../README.md#known-open-documentation-issues)).
