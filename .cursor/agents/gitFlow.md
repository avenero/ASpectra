---
name: gitFlow
model: claude-sonnet-5-5[]
description: Git flow enforcer for the A·Spectra monorepo. Use this agent after any feature or fix task is complete to handle compilation, test gates, the merge into develop, the remote push, and removal of the task's worktree and branch. Invoke proactively whenever work on a feature/session-idle/fix branch is done and the developer wants it merged into develop. Also use when resolving compilation errors or warnings before a merge. Runs after code-reviewer-kiu and after core-publisher; it never publishes or syncs Core itself. Stops at develop: it promotes develop to master only when the user asks for that literally.
---

# gitFlow - Cursor adapter

The canonical definition of this agent is [`.claude/agents/gitFlow.md`](../../.claude/agents/gitFlow.md). Read it
in full before acting and follow it exactly: role, required inputs, rules, process, output format and the
"Not for this agent" list. This file only maps that definition to Cursor. It adds no rules; a change to the
agent's behavior is made in the canonical file, with explicit approval (working rule 2).

Shared rules the canonical file relies on: [platform.md](../../docs/architecture/platform.md),
[repository-rules.md](../../docs/architecture/repository-rules.md),
[engineering-rules.md](../../.claude/rules/engineering-rules.md),
[git-workflow.md](../../docs/development/git-workflow.md) and
[agent-pipeline.md](../../docs/development/agent-pipeline.md), plus the `AGENTS.md` of the repository you work in.

## Cursor tool mapping

Where the canonical file names a Claude Code tool, use the Cursor equivalent:

| Canonical (Claude Code) | Cursor |
|---|---|
| `Read`, `Grep`, `Glob` | `Read`, `Grep`, `Glob` |
| `Edit` | `StrReplace` |
| `Write` | `Write` |
| `Bash` (background run, `BashOutput`) | `Shell` (background run, `AwaitShell`) |
| Lint or build check after an edit | `ReadLints`, then the repository gate in `Shell` |
| `TodoWrite` | `TodoWrite` |
| `Agent` (call another subagent) | `Task`, with the agent name from `.cursor/agents/` |
| `gh` CLI through `Bash` | GitHub MCP when configured, otherwise `gh` in `Shell` |
| `WebSearch` / `WebFetch` for current library APIs | Context7 MCP: `resolve-library-id`, then `query-docs` (skill `context7-mcp`) |
| `agent-browser` skill (`.claude/skills/agent-browser/SKILL.md`) | cursor-ide-browser MCP; read `.cursor/skills/agent-browser/SKILL.md` first |
| Built-in `security-review` skill (only when asked by name) | `bugbot` and `security-review` subagents (only when asked by name) |
| Removing a file with `Bash rm` (forbidden where the canonical file says so) | Delete (forbidden in the same cases) |
