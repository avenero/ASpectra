# 0002 Single source for agent instructions

Status: Accepted
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
- The shared rule body lives in five canonical documents: `docs/architecture/platform.md`,
  `docs/architecture/repository-rules.md`, `docs/development/engineering-rules.md`,
  `docs/development/git-workflow.md` and `docs/development/agent-pipeline.md` (moved 2026-10-20).
- `.claude/CLAUDE.md` stays in place as Claude Code's workspace memory and imports those documents with
  `@path`; `.claude/ORCHESTRATION.md` and `.cursor/agents/AGENTS.md` hold only tool-specific notes.
- `.claude/agents/` is the canonical agent set. Each file in `.cursor/agents/` is a short adapter that keeps
  Cursor's frontmatter (name, model, description, readonly), points to the canonical file and maps tool names.
- ASpectra commits its own `.claude/` and `.cursor/` folders, except local settings. The rule in
  `.claude/CLAUDE.md` that `.claude/` is never committed applies to the three application
  repositories, not to ASpectra.

## Alternatives
- Keep the copies synchronized by hand: already failed (drift).
- Generate tool files from one source with a script: extra tooling without a demonstrated need.

## Consequences
- New files never copy a rule body.
- Any rule change is made once, in the canonical document, with explicit approval (working rule 2).
- Agent behavior changes in one place. The adapters change only when Cursor's tool names change.
