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
| [alcance.md](alcance.md) | Product scope definition | TBD | Unverified |
| [requirements.md](requirements.md) | Screen inventory and requirements baseline | TBD | Unverified |
| [diagrams/](diagrams/README.md) | Architecture diagrams (General, BE, Core, FE) | TBD | Unverified |
| [architecture/README.md](architecture/README.md) | Architecture index | TBD | Intended |
| [architecture/repository-boundaries.md](architecture/repository-boundaries.md) | Ownership and dependencies | TBD | Current-verified (from files) |
| [architecture/repository-rules.md](architecture/repository-rules.md) | Cross-repository rules | TBD | Intended |
| [architecture/decisions/](architecture/decisions/README.md) | Architecture decision records | TBD | Per record |
| [development/workspace-setup.md](development/workspace-setup.md) | Workspace setup | TBD | Intended |
| [development/agent-workflow.md](development/agent-workflow.md) | How agents find instructions | TBD | Intended |
| [development/cross-repository-changes.md](development/cross-repository-changes.md) | Cross-repository change procedure | TBD | Intended |
| [deployment/README.md](deployment/README.md) | Deployment ownership | TBD | Current-verified (from files), partly TBD |
| [deployment/release-coordination.md](deployment/release-coordination.md) | Coordinated releases | TBD | Intended |
| [deployment/releases/](deployment/releases/TEMPLATE.md) | Release records | TBD | Per record |

## Known open documentation issues

These were found during the 2026-10-20 audit. They are recorded here, not fixed. Each needs a
separate, approved documentation update.

| ID | Issue | Location |
|---|---|---|
| C1 | "AI design" and "Technical design" sections describe a FastAPI/uv backend and a "legal document"; they contradict the Spring Boot/React stack described in the same file | `.claude/CLAUDE.md` |
| C2 | States that the workspace root is not a Git repository; no longer true | `.claude/CLAUDE.md`, `architect.md` and `gitFlow.md` in `.claude/agents/` and `.cursor/agents/` |
| C3 | States "GitHub Actions is off"; workflow files exist in all three repositories, including a production deploy | `.claude/CLAUDE.md`, `.claude/ORCHESTRATION.md` |
| C4 | The shared rule body exists in three hand-synced copies | `.claude/CLAUDE.md`, `.claude/ORCHESTRATION.md`, `.cursor/agents/AGENTS.md` |
| C5 | Agent definitions are duplicated and have drifted | `.claude/agents/`, `.cursor/agents/` |
| C6 | Child `AGENTS.md` files link to `../.cursor/AGENTS.md`, which does not exist | each child repository |
| C7 | Child `AGENTS.md` files name the child `.claude/CLAUDE.md` as source of truth, but it is gitignored | each child repository |
| C8 | `.cursor/agents/AGENTS.md` may be read by Cursor as a subagent definition; verify | `.cursor/agents/` |
| C9 | The SUMMARY section ends mid-sentence | `.claude/CLAUDE.md` |
| C10 | Technical content not validated against the code | `alcance.md`, `requirements.md`, `diagrams/` |
