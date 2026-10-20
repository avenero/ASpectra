# Repositories

Status: Current-verified (from local Git remotes, 2026-10-20) | Owner: TBD

ASpectra references three independent application repositories. Each one owns its source code,
tests, build configuration, Git history, branches, tags and releases
([rule R1](docs/architecture/repository-rules.md#r1-ownership)). ASpectra never tracks their
files ([R3](docs/architecture/repository-rules.md#r3-root-isolation)).

The machine-readable mirror of this page is [config/repository-map.yaml](config/repository-map.yaml).
Change both in the same commit.

## Governance repository

| Field | Value |
|---|---|
| Name | ASpectra |
| URL | `https://github.com/<OWNER>/ASpectra` (TBD: not published yet) |
| Branch | `main` |
| Builds or deploys | Nothing |

## qa-platformCore

| Field | Value |
|---|---|
| URL | https://github.com/avenero/qa-platformCore |
| Local path | `qa-platformCore/` |
| Role | Deterministic test engine: Cucumber runtime, step libraries (HTTP, Web, Mobile, Database), evidence and reporting, mobile agent |
| Integration branch | `develop` |
| Default branch (origin/HEAD) | `master` |
| Its own instructions | `AGENTS.md` (tracked), `.claude/CLAUDE.md` (local, untracked) |

## qa-platformBE

| Field | Value |
|---|---|
| URL | https://github.com/avenero/qa-platformBE |
| Local path | `qa-platformBE/` |
| Role | Overlay backend: connectors, canonical model, review lifecycle, execution launch, evidence, result sync, all AI |
| Integration branch | `develop` |
| Default branch (origin/HEAD) | `master` |
| Its own instructions | `AGENTS.md` (tracked), `.claude/CLAUDE.md` (local, untracked) |

## qa-platformFE

| Field | Value |
|---|---|
| URL | https://github.com/avenero/qa-platformFE |
| Local path | `qa-platformFE/` |
| Role | Single-page web console: integrations settings and the execution overlay |
| Integration branch | `develop` |
| Default branch (origin/HEAD) | `master` |
| Its own instructions | `AGENTS.md` (tracked), `.claude/CLAUDE.md` (local, untracked) |

## Coordination

- Dependencies between the repositories: [docs/architecture/repository-boundaries.md](docs/architecture/repository-boundaries.md).
- Changes that span repositories: [docs/development/cross-repository-changes.md](docs/development/cross-repository-changes.md).
- Releases: [docs/deployment/release-coordination.md](docs/deployment/release-coordination.md).
- Owners per repository: TBD.
