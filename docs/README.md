# Documentation index

Status: Current-verified (index, 2026-10-20) | Owner: TBD

This index lists every ASpectra document with its topic, owner and status. Documentation of a
single application repository lives in that repository and is linked, not copied.

## Status labels

Every document states one status on its second line
([rule R15](architecture/repository-rules.md#r15-status-labels)).

| Status | Meaning |
|---|---|
| Current-verified | Checked against the code or Git state; the line gives the date or commit |
| Intended | Agreed target design; not necessarily implemented |
| Planned | Future work; not decided in detail |
| Unverified | Existing content that has not been reviewed against the current code |

## Documents

| Path | Topic | Owner | Status |
|---|---|---|---|
| [../README.md](../README.md) | Entry point | TBD | Current-verified |
| [../REPOSITORIES.md](../REPOSITORIES.md) | Repository registry | TBD | Current-verified |
| [../AGENTS.md](../AGENTS.md) | Agent entry point | TBD | Intended |
| [alcance.md](alcance.md) | Product scope definition: **the only source of truth for scope** | avenero | Canonical (§4.12 onward awaits owner confirmation) |
| [requirements.md](requirements.md) | Requirements by feature, module and screen, derived from `alcance.md` | avenero | Intended |
| [diagrams/](diagrams/README.md) | Architecture design diagrams (platform, BE, Core, FE) and detailed views, today's state and decided target | TBD | Current-verified (2026-10-20) |
| [architecture/README.md](architecture/README.md) | Architecture index | TBD | Intended |
| [architecture/platform.md](architecture/platform.md) | Product, repositories, flow, cross-repository mechanics | TBD | Intended |
| [architecture/repository-boundaries.md](architecture/repository-boundaries.md) | Ownership and dependencies | TBD | Current-verified (from files) |
| [architecture/repository-rules.md](architecture/repository-rules.md) | Cross-repository rules | TBD | Intended |
| [architecture/decisions/](architecture/decisions/README.md) | Architecture decision records | TBD | Per record |
| [development/workspace-setup.md](development/workspace-setup.md) | Workspace setup | TBD | Intended |
| [development/engineering-rules.md](development/engineering-rules.md) | How to work, AI limits, working rules, gates, guardrails | TBD | Intended |
| [development/git-workflow.md](development/git-workflow.md) | Worktrees, branch strategy, feature flow | TBD | Intended |
| [development/agent-pipeline.md](development/agent-pipeline.md) | Subagent pipeline | TBD | Intended |
| [development/agent-workflow.md](development/agent-workflow.md) | How agents find instructions | TBD | Intended |
| [development/cross-repository-changes.md](development/cross-repository-changes.md) | Cross-repository change procedure | TBD | Intended |
| [deployment/README.md](deployment/README.md) | Deployment ownership | TBD | Current-verified (from files), partly TBD |
| [deployment/release-coordination.md](deployment/release-coordination.md) | Coordinated releases | TBD | Intended |
| [deployment/releases/](deployment/releases/TEMPLATE.md) | Release records | TBD | Per record |

## Known open documentation issues

Found during the 2026-10-20 audit and not yet resolved. C1-C5 and C8-C11 were resolved on 2026-10-20 (shared rules moved to the canonical
documents; `alcance.md` §0.1 and §13 updated to the ASpectra layout).

| ID | Issue | Location |
|---|---|---|
| C6 | Child `AGENTS.md` files link to `../.cursor/AGENTS.md`, which does not exist. Fixed on branch `feature/workspace-instructions` in each repository (pushed), pending merge into develop | each child repository |
| C7 | Child `.claude/CLAUDE.md` is gitignored. Versioned on branch `feature/workspace-instructions` in each repository (pushed), pending merge into develop | each child repository |
