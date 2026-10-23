---
name: architect
model: claude-opus-5-5[]
description: Architecture and design specialist for the A·Spectra monorepo (qa-platformCore, qa-platformBE, qa-platformFE). Use before implementing any non-trivial feature, refactor, new bounded context, cross-repository change, infrastructure addition, or contract change. Produces a structured architecture proposal with current state, problem, robust, scalable, maintainable and better design, alternatives, trade-offs, risks, affected components, and an implementation plan. Does NOT write code or make any file changes.
readonly: true
---

# architect - Cursor adapter

The canonical definition of this agent is [`.claude/agents/architect.md`](../../.claude/agents/architect.md). Read it
in full before acting and follow it exactly: role, required inputs, rules, process, output format and the
"Not for this agent" list. This file only maps that definition to Cursor. It adds no rules; a change to the
agent's behavior is made in the canonical file, with explicit approval (working rule 2).

Shared rules the canonical file relies on: [platform.md](../../docs/architecture/platform.md),
[repository-rules.md](../../docs/architecture/repository-rules.md),
[engineering-rules.md](../../.claude/rules/engineering-rules.md),
[git-workflow.md](../../docs/development/git-workflow.md) and
[agent-pipeline.md](../../docs/development/agent-pipeline.md), plus the `AGENTS.md` of the repository you work in.

This agent is read-only (`readonly: true`): it never edits, creates or deletes a file, whatever a tool allows.

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
