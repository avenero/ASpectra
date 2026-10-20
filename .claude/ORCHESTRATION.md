# ORCHESTRATION.md - Claude Code adapter for the agent pipeline

Imported into every session by `.claude/CLAUDE.md`. The pipeline itself (task sizing, the ten agents,
the orchestration protocol) is tool-neutral and lives in the canonical document imported below. This file
holds only what is specific to Claude Code.

## Claude Code specifics

- The ten subagents live flat in `.claude/agents/`; this folder is the canonical agent set. Cursor keeps a
  short adapter per agent in `.cursor/agents/` that points here and maps tool names; a change to an
  agent's behavior is made only here.
- Call a subagent with the `Agent` tool. Built-in tools: `Read`, `Grep`, `Glob`, `Bash`, `Edit`, `Write`,
  `NotebookEdit`, `WebFetch`, `WebSearch`, `TodoWrite`, `Agent`.
- GitHub work uses the `gh` CLI through `Bash`; no GitHub MCP server is configured here.
- Browser automation uses the `agent-browser` skill: read `.claude/skills/agent-browser/SKILL.md` before
  driving a browser. No Context7 or browser MCP is configured here.
- Backlog: GitHub issues per repository; open owner decisions are on
  [qa-platformBE #124](https://github.com/avenero/qa-platformBE/issues/124).

## Pipeline

@../docs/development/agent-pipeline.md
