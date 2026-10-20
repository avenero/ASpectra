---
name: core-publisher
model: claude-sonnet-5-5[]
description: Local Core package publisher and BE synchronizer for the A·Spectra monorepo. Use after code-reviewer-kiu passes and before gitFlow, whenever qa-platformCore changed or whenever qa-platformBE/lib/core-m2 is empty or stale for the current coreVersion. Not called when Core did not change - there is nothing to publish. Publishes Core to the local Maven repository, syncs lib/core-m2, updates core.ref, and re-verifies the BE gate against the new artifacts. Serializes across sessions with the .core-publish.lock lock, and when several sessions finished Core work it publishes once for the whole batch instead of once per session. Refuses to publish a non-SNAPSHOT release, which belongs to the tag-driven publish workflow. Does NOT merge, push, or write feature code.
---

# core-publisher - Cursor adapter

The canonical definition of this agent is [`.claude/agents/core-publisher.md`](../../.claude/agents/core-publisher.md). Read it
in full before acting and follow it exactly: role, required inputs, rules, process, output format and the
"Not for this agent" list. This file only maps that definition to Cursor. It adds no rules; a change to the
agent's behavior is made in the canonical file, with explicit approval (working rule 2).

Shared rules the canonical file relies on: [platform.md](../../docs/architecture/platform.md),
[repository-rules.md](../../docs/architecture/repository-rules.md),
[engineering-rules.md](../../docs/development/engineering-rules.md),
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
