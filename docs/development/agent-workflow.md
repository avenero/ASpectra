# Agent workflow

Status: Intended | Owner: TBD

How Claude Code, Codex and Cursor find their instructions in this nested layout. Items marked
**verify** come from each tool's public documentation and must be confirmed in the installed
version.

## Loading behaviour

| Tool | Loads automatically | Session at the ASpectra root | Session inside an application repository |
|---|---|---|---|
| Claude Code | `CLAUDE.md` or `.claude/CLAUDE.md` from the working directory upward, `@path` imports, subdirectory `CLAUDE.md` files on demand, agents and skills from `.claude/` | `.claude/CLAUDE.md` plus the imported `.claude/ORCHESTRATION.md` | The repository's `.claude/CLAUDE.md`; parent files upward (**verify** for the `.claude/` variant) |
| Codex | `AGENTS.md` files from the Git root down to the working directory | Root [`AGENTS.md`](../../AGENTS.md) | Only the repository's own `AGENTS.md` (it is its own Git root) |
| Cursor | `.cursor/rules/*.mdc`, `AGENTS.md` at the root and in subdirectories, subagents in `.cursor/agents/` (**verify**) | `workspace.mdc`, root `AGENTS.md`, `child-repositories.mdc` for files under a child | The repository's own rules and `AGENTS.md` when it is opened as its own project |

Consequence: when Codex runs inside an application repository, it reaches the shared rules
only through a link in that repository's `AGENTS.md`. Claude Code at the root receives the canonical
documents through the `@` imports in `.claude/CLAUDE.md`. Cursor subagents do not load files on their own;
`.cursor/agents/AGENTS.md` lists what they must read.

## Precedence

See [repository-rules.md](../architecture/repository-rules.md#precedence).

## Canonical sources

| Topic | Canonical source |
|---|---|
| Cross-repository rules | [repository-rules.md](../architecture/repository-rules.md) |
| Product scope | [../alcance.md](../alcance.md) |
| Product, repositories, end-to-end flow, cross-repository mechanics | [platform.md](../architecture/platform.md) |
| How to work, AI limits, working rules, gates, guardrails G1-G36 | [engineering-rules.md](engineering-rules.md) |
| Worktrees, branch strategy, feature flow | [git-workflow.md](git-workflow.md) |
| Agent pipeline and agent selection | [agent-pipeline.md](agent-pipeline.md) |
| Agent definitions | [.claude/agents/](../../.claude/agents/) (`.cursor/agents/` is a drifted copy, issue C5) |
| Repository URLs and roles | [REPOSITORIES.md](../../REPOSITORIES.md), mirrored in [config/repository-map.yaml](../../config/repository-map.yaml) |
| Ownership and dependencies | [repository-boundaries.md](../architecture/repository-boundaries.md) |
| Per-repository rules, build and test | Each repository's `AGENTS.md` and README |
| Deployment ownership and releases | [deployment/](../deployment/README.md) |

## Entry files

| File | Tool | Content rule |
|---|---|---|
| [AGENTS.md](../../AGENTS.md) | Codex, Cursor, other agents | Links only |
| [.cursor/rules/workspace.mdc](../../.cursor/rules/workspace.mdc) | Cursor, always applied | Links only |
| [.cursor/rules/child-repositories.mdc](../../.cursor/rules/child-repositories.mdc) | Cursor, files under `qa-platform*/` | Boundary reminder |
| [.claude/CLAUDE.md](../../.claude/CLAUDE.md) | Claude Code | `@` imports of the canonical documents |
| [.claude/ORCHESTRATION.md](../../.claude/ORCHESTRATION.md) | Claude Code | Claude-specific notes, `@` import of the pipeline |
| [.cursor/agents/AGENTS.md](../../.cursor/agents/AGENTS.md) | Cursor subagent | Cursor-specific notes and links |
