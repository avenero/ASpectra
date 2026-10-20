# Repository boundaries

Status: Current-verified (from repository files and Git remotes, 2026-10-20) | Owner: TBD

## Layout

```
ASpectra/              governance repository (this one)
├── qa-platformFE/     independent repository, ignored by ASpectra
├── qa-platformBE/     independent repository, ignored by ASpectra
└── qa-platformCore/   independent repository, ignored by ASpectra
```

The three application repositories are nested on disk only. Each has its own `.git`, remote,
branches and history. ASpectra ignores them by explicit path in [`.gitignore`](../../.gitignore),
so they can never be recorded as files or as gitlinks. Decision:
[ADR 0001](decisions/0001-governance-repo-without-submodules.md).

## Ownership

| Owns | ASpectra | qa-platformCore | qa-platformBE | qa-platformFE |
|---|---|---|---|---|
| Source code and tests | No | Yes | Yes | Yes |
| Build and test configuration | No | Yes | Yes | Yes |
| Git history, branches, tags, releases | Its own only | Yes | Yes | Yes |
| Deployment configuration | No | Artifact publishing | Production deploy workflow | Dockerfile only |
| Secrets | Never | Its own | Its own | Its own |
| Cross-repository rules and decisions | Yes | No | No | No |
| Agent orchestration across repositories | Yes | No | No | No |
| Repository-specific agent instructions | No | Yes | Yes | Yes |

ASpectra does not own a duplicated copy of any application documentation, and it is not a
runtime service.

## Dependencies

Read from repository files; whether the workflows run is not verified (see
[../deployment/README.md](../deployment/README.md)).

| From | To | Mechanism | Source |
|---|---|---|---|
| qa-platformBE | qa-platformCore | Maven artifacts `com.qa:{http,web,mobile,database}-core:${coreVersion}` | `qa-platformBE/gradle.properties` |
| qa-platformBE | qa-platformCore | Tested Core commit pinned in `core.ref` | `qa-platformBE/core.ref` |
| qa-platformBE | qa-platformFE | Deploy workflow builds the FE image at the commit pinned in `fe.ref` | `qa-platformBE/fe.ref`, `.github/workflows/deploy.yml` |
| qa-platformFE | qa-platformBE, qa-platformCore | E2E workflow checks out BE and Core refs | `qa-platformFE/.github/workflows/e2e.yml` |

`coreVersion` must match in `qa-platformCore/gradle.properties` and `qa-platformBE/gradle.properties`.
