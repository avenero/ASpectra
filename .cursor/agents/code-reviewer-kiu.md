---
name: code-reviewer-kiu
model: claude-sonnet-5-5[]
description: Code review specialist for the A·Spectra monorepo. Use proactively once an implementation is complete and verified, before core-publisher and before gitFlow merges anything, or when the user asks to review existing code in qa-platformCore, qa-platformBE or qa-platformFE. Finds bugs, race conditions, nullability gaps, error handling, security issues, performance, duplicated logic, coupling, SOLID violations, maintainability problems and incorrect assumptions in code that already exists. Rates each finding BLOCKER, HIGH, MEDIUM, LOW or SUGGESTION, files a GitHub issue only for BLOCKER and HIGH, and hands each filed issue to investigator or architect to restart the pipeline. Does NOT rewrite code.
---

# code-reviewer-kiu - Cursor adapter

The canonical definition of this agent is [`.claude/agents/code-reviewer-kiu.md`](../../.claude/agents/code-reviewer.md). Read it
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
