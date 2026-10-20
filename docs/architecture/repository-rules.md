# Repository rules

Status: Intended | Owner: TBD

These are the canonical cross-repository engineering rules for A·Spectra. Every other
instruction file links here instead of copying them
([R7](#r7-single-source-of-truth)).

They extend, and never weaken, the engineering rules in
[../development/engineering-rules.md](../development/engineering-rules.md) (working rules 1-20, "What the
AI does, and never does", gates, guardrails G1-G36) and the git rules in
[../development/git-workflow.md](../development/git-workflow.md). Those remain authoritative. Repository-specific rules live in each repository's `AGENTS.md`.

## Precedence

1. The user's explicit instruction in the current session.
2. The owning repository's instructions, for files inside that repository.
3. These rules, the engineering rules and the git workflow.
4. Tool-specific files (`AGENTS.md`, `.cursor/rules/*.mdc`, agent and skill definitions).
   They may narrow behaviour but never contradict levels 2-3.

The tools do not enforce this order. When two levels disagree, stop and report the conflict
([R12](#r12-no-silent-override)).

## Rules

### R1 Ownership
Each of qa-platformFE, qa-platformBE and qa-platformCore owns its source code, tests, build
configuration, history, tags and releases. Extends G17 and working rule 9.

### R2 Change location
Every code change is committed and reviewed in its owning repository, through that
repository's git flow.

### R3 Root isolation
ASpectra tracks only ASpectra-owned files. The application directories are ignored by
explicit path and are never added as files or gitlinks. Stage by explicit path; never run
`git add .` or `git add -A` at the root. See
[ADR 0001](decisions/0001-governance-repo-without-submodules.md).

### R4 Impact statement
A cross-repository change names every affected repository and the dependency path between them
(for example Core, then BE, then FE). Extends working rule 12.

### R5 Contract coordination
A change to a shared contract (Core consumer surface, mobile-agent wire, REST and STOMP APIs,
Zod schemas, Cucumber step patterns) is coordinated with every consumer before it is merged.
Extends G34, G35 and working rules 12 and 15.

### R6 No silent breaking changes
No breaking change to the architecture or a contract without explicit review and approval.
Extends working rules 3, 5, 6 and 15.

### R7 Single source of truth
Each topic has one canonical document. Other documents link to it. The map of canonical
sources is in [../development/agent-workflow.md](../development/agent-workflow.md#canonical-sources).

### R8 Inspect first
Agents inspect repository boundaries, Git state and local instructions before proposing a
change. Extends working rule 3.

### R9 No fabrication
Agents never invent APIs, integrations, environments, configuration or repository
capabilities. Extends working rule 1.

### R10 No secrets in Git
Secrets, credentials and machine-specific configuration are never committed, in any
repository. Extends G1, G23 and "Never commit `.env`".

### R11 History is preserved
No history rewrite, no force-push and no re-initialization of an existing repository.

### R12 No silent override
Root governance never silently overrides repository instructions. Conflicts are reported and
resolved explicitly.

### R13 Local instructions apply
Agents follow the owning repository's `AGENTS.md` and `.claude/CLAUDE.md` in addition to these
rules.

### R14 Core boundaries
The Core execution contract and its boundaries are respected. Extends working rule 6, G11,
G18-G20 and G34.

### R15 Status labels
Every document states whether it describes Current-verified, Intended, Planned or Unverified
content. See [../README.md](../README.md#status-labels).

### R16 Explicit authorization
Destructive actions, remote changes, commits, pushes, promotions to `master` and deployments
require explicit authorization. Extends the branch strategy ("master only on literal request")
and the `security-guardian` gate.
