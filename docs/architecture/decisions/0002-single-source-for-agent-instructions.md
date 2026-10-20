# 0002 Single source for agent instructions

Status: Accepted (partially implemented)
Date: 2026-10-20
Deciders: Abel (project owner)

## Context
Claude Code, Codex and Cursor are used in this workspace. They load instructions differently
([agent-workflow.md](../../development/agent-workflow.md)). The shared rule body was maintained by
hand in three copies (`.claude/CLAUDE.md`, `.claude/ORCHESTRATION.md`,
`.cursor/agents/AGENTS.md`) and had drifted. Agent definitions exist twice, in `.claude/agents/`
and `.cursor/agents/`, and have also drifted.

## Decision
- Cross-repository rules have one canonical file:
  [repository-rules.md](../repository-rules.md).
- Tool entry files (`AGENTS.md`, `.cursor/rules/*.mdc`) are short and only link to canonical files.
- `.claude/CLAUDE.md` stays in place as Claude Code's workspace memory and stays authoritative
  for the product rules and guardrails until a separately approved change extracts them.
- `.claude/agents/` becomes the canonical agent set; `.cursor/agents/` is kept unchanged until that
  separately approved change.
- ASpectra commits its own `.claude/` and `.cursor/` folders, except local settings. The rule in
  `.claude/CLAUDE.md` that `.claude/` is never committed applies to the three application
  repositories, not to ASpectra.

## Alternatives
- Keep the copies synchronized by hand: already failed (drift).
- Generate tool files from one source with a script: extra tooling without a demonstrated need.

## Consequences
- New files never copy a rule body.
- Pending, each needing approval: deduplicate `.claude/CLAUDE.md`, `.claude/ORCHESTRATION.md` and
  `.cursor/agents/AGENTS.md`; fix the child `AGENTS.md` links.
