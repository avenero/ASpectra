# ASpectra

Status: Current-verified (repository layout, 2026-10-20) | Owner: TBD

ASpectra is the governance repository of the A·Spectra platform. It holds project-level
documentation, cross-repository rules, architecture decisions, AI-agent orchestration and the
references to the three application repositories. It contains no application source code and
is never built or deployed.

ASpectra is **not a monorepo** and uses **no Git submodules**. The application repositories are
independent Git repositories. They are cloned inside this workspace and ignored by it.

## Repositories

| Repository | Role | Details |
|---|---|---|
| `qa-platformCore` | Deterministic test engine and step libraries | [REPOSITORIES.md](docs/architecture/REPOSITORIES.md#qa-platformcore) |
| `qa-platformBE` | Overlay backend, integrations, AI | [REPOSITORIES.md](docs/architecture/REPOSITORIES.md#qa-platformbe) |
| `qa-platformFE` | Web console | [REPOSITORIES.md](docs/architecture/REPOSITORIES.md#qa-platformfe) |

## Start here

- People: [docs/README.md](docs/README.md) (documentation index), then
  [docs/development/workspace-setup.md](docs/development/workspace-setup.md).
- AI agents: [AGENTS.md](.cursor/AGENTS.md). Claude Code also loads [.claude/CLAUDE.md](.claude/CLAUDE.md).
- Rules for any change that touches more than one repository:
  [docs/architecture/repository-rules.md](docs/architecture/repository-rules.md).

## Documentation status labels

Every document declares one status. See [docs/README.md](docs/README.md#status-labels).
