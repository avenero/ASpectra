# AGENTS.md - ASpectra workspace

Status: Intended | Owner: TBD

ASpectra is the governance repository of A·Spectra. It contains documentation, rules and agent
configuration, and no application code.

`qa-platformFE/`, `qa-platformBE/` and `qa-platformCore/` are independent Git repositories nested in
this folder and ignored by it. When you work inside one, read its own `AGENTS.md`, follow its git
flow and commit there, never at the ASpectra root.

## Read, in order

1. [docs/architecture/platform.md](docs/architecture/platform.md): product, release scope, repositories, end-to-end flow, cross-repository mechanics.
2. [docs/architecture/repository-rules.md](docs/architecture/repository-rules.md): cross-repository rules R1-R16 and precedence.
3. [docs/development/engineering-rules.md](docs/development/engineering-rules.md): how to work, AI limits, working rules, gates, guardrails G1-G36.
4. [docs/development/git-workflow.md](docs/development/git-workflow.md): worktrees, branch strategy, feature flow.
5. [docs/development/agent-pipeline.md](docs/development/agent-pipeline.md): task sizing, subagents, orchestration protocol.
6. [REPOSITORIES.md](REPOSITORIES.md): repository URLs, roles and their own instruction files.

Product scope is defined in [docs/alcance.md](docs/alcance.md). How each tool loads instructions:
[docs/development/agent-workflow.md](docs/development/agent-workflow.md).

## Precedence

The user's explicit instruction, then the owning repository's instructions, then the shared rules,
then tool-specific files. Report any conflict; never resolve one silently.

## Always

- Inspect Git state and repository boundaries before proposing a change.
- At the ASpectra root, stage by explicit path. Never `git add .` or `git add -A`.
- Commits, pushes, remote changes, deletions and deployments need explicit approval.
- Do not copy rules into this file. Link to the canonical document.
