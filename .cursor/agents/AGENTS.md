---
name: AGENTS
model: claude-sonnet-5-5[]
---

# AGENTS.md - Cursor adapter for the A·Spectra agent pipeline

Cursor reads this file as a subagent definition, so it does not load other files on its own. Before acting,
read the canonical documents below; they hold every shared rule, once. Do not copy their content here.

1. [docs/architecture/platform.md](../../docs/architecture/platform.md): product, release scope, repositories, end-to-end flow, cross-repository mechanics.
2. [docs/architecture/repository-rules.md](../../docs/architecture/repository-rules.md): cross-repository rules R1-R16 and precedence.
3. [docs/development/engineering-rules.md](../../docs/development/engineering-rules.md): how to work, AI limits, working rules, gates, guardrails G1-G36.
4. [docs/development/git-workflow.md](../../docs/development/git-workflow.md): worktrees, branch strategy, feature flow.
5. [docs/development/agent-pipeline.md](../../docs/development/agent-pipeline.md): task sizing, the ten agents, orchestration protocol.
6. The `AGENTS.md` of the repository you change.

Product scope is defined in [docs/alcance.md](../../docs/alcance.md).

## Cursor specifics

- The ten subagents live in `.cursor/agents/`; their canonical versions are in `.claude/agents/`. A change to
  an agent's behavior belongs in both.
- Call a subagent with the `Task` tool.
- GitHub work may use the GitHub MCP; library documentation may use the Context7 MCP; browser checks may use
  the cursor-ide-browser MCP with `.cursor/skills/agent-browser/SKILL.md`.
