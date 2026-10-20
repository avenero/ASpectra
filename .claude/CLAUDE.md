# CLAUDE.md - ASpectra workspace

Workspace memory for Claude Code at the ASpectra root. ASpectra is the governance repository; it holds
documentation, rules and agent configuration, and no application code. The three application repositories
are nested here and ignored by it. Each has its own guidance:

| Repository | Its guidance |
|---|---|
| `qa-platformCore` | [qa-platformCore/.claude/CLAUDE.md](../qa-platformCore/.claude/CLAUDE.md), [qa-platformCore/AGENTS.md](../qa-platformCore/AGENTS.md) |
| `qa-platformBE` | [qa-platformBE/.claude/CLAUDE.md](../qa-platformBE/.claude/CLAUDE.md), [qa-platformBE/AGENTS.md](../qa-platformBE/AGENTS.md) |
| `qa-platformFE` | [qa-platformFE/.claude/CLAUDE.md](../qa-platformFE/.claude/CLAUDE.md), [qa-platformFE/AGENTS.md](../qa-platformFE/AGENTS.md) |

Read this file plus the guidance of the repository you are changing, and only that one.

The shared rules live once, in the canonical documents imported below. Do not copy them back into this
file; change the canonical document instead, and only with explicit approval (working rule 2).

## Product and architecture

@../docs/architecture/platform.md

## Cross-repository rules

@../docs/architecture/repository-rules.md

## Engineering rules

@../docs/development/engineering-rules.md

## Git workflow

@../docs/development/git-workflow.md

## Subagent orchestration

@ORCHESTRATION.md
