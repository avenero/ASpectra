# Agent pipeline

Status: Intended (moved from `.claude/ORCHESTRATION.md` and `.cursor/agents/AGENTS.md` on 2026-10-20) | Owner: TBD

The subagent pipeline shared by Claude Code and Cursor: task sizing, the ten agents, and the orchestration
protocol. Tool-specific notes: [../../.claude/ORCHESTRATION.md](../../.claude/ORCHESTRATION.md) (Claude Code) and
[../../.cursor/agents/AGENTS.md](../../.cursor/agents/AGENTS.md) (Cursor). Git steps follow
[git-workflow.md](git-workflow.md).

## Agent collaboration

This workspace uses specialized subagents. Every implementation agent must act as
orchestrator: invoke the relevant subagent, wait for its result, resolve any errors
it could not fix on its own (using its own implementation context), and retry until
the pipeline is fully green.

### Task sizing — deciding how much of the pipeline runs

No subagent classifies this for you. Before calling any subagent, the orchestrating agent
reads the task and picks one of the three sizes below, then states which one it picked and
why. When in doubt, pick the larger size — the cost of an unnecessary `architect` call is
small; the cost of skipping design for something that needed it is not.

**Before sizing, check whether the task is even ready to be sized.** If it arrived as a raw,
natural-language requirement or feature request from a user or stakeholder — not an already
-detailed bug report and not an already fully-specified technical task — call
`requirements-analyst` first (Step -1 in the Orchestration protocol below). Sizing a vague
requirement is a guess; "allow users to execute test cases from Jira" can be one file or a new
bounded context, and you cannot tell which until the ambiguities are resolved.

| Size | Objective criteria | Pipeline |
|---|---|---|
| **Trivial** | All of: touches a single file (or a handful of clearly related lines in one change), adds no new public class or method, changes no contract/schema/config key, adds no new dependency, and the fix is fully described by the report itself (a typo, an off-by-one, a wrong log level, a wrong config default, a missing null check). Any doubt on any point moves it to Feature. | `implementer` directly — the orchestrator supplies the "architecture decision, implementation plan, acceptance criteria" implementer requires, since there is no `architect` proposal to draw them from. Then `code-reviewer-kiu` → `core-publisher` (only if Core changed) → `gitFlow`, same as every other size. |
| **Feature / refactor / non-trivial fix** | Anything that fails a Trivial criterion above: new behavior, a new public method or class, more than one file with related logic, a contract/schema/config change, cross-repository impact, or a refactor of existing structure. | Full pipeline: `architect` → `test-strategist` (Strategy) → `implementer` → `test-strategist` (Verification) → `code-reviewer-kiu` → `core-publisher` (only if Core changed) → `gitFlow`. |
| **Unknown-cause / complex** | A bug, flaky test, or behaviour whose root cause is not yet known — "it fails sometimes," an unexplained inconsistency, a race condition. | `investigator` first. If it names the cause with high confidence and the fix meets every Trivial criterion, continue as Trivial. Otherwise continue as Feature, with the investigator report as `architect`'s input. |

What never depends on size, per the Orchestration protocol below:
- The worktree comes first, Trivial included: every task that will produce a commit gets its own
  worktree and branch before any file is edited (Step 0a). A one-line fix in the shared checkout is
  exactly how two sessions collide.
- `code-reviewer-kiu` runs on every diff, Trivial included, and it runs **before** `gitFlow`
  merges anything. Reviewing after the merge would only tell you what you already shipped.
- `core-publisher` runs whenever `qa-platformCore` changed, Trivial included, and also before
  `gitFlow`. It is not called when Core did not change and `lib/core-m2` is already consistent -
  there is nothing to publish. When several sessions changed Core, only the last one to finish
  publishes, once, for the whole batch.
- `gitFlow` runs for anything that ships, Trivial included — compilation and the full test
  suite do not know how small a change looked. It also removes the worktree and branch at the end.
- `security-guardian`'s gate before a `develop`-to-`master` promotion (Step 7b) is mandatory
  regardless of size. It is additionally called proactively, outside that gate, whenever a
  diff of any size touches auth, session handling, secrets, connectors, or outbound calls.

### Available subagents

All ten live flat in the tool's agent folder: `.claude/agents/` for Claude Code (canonical) and
`.cursor/agents/` for Cursor. There is no per-repository variant of any of them. In the table below,
`agents/` means that folder.
A task in `qa-platformCore`, `qa-platformBE` or `qa-platformFE` uses the same ten agents; the
repository only changes which `AGENTS.md`/`CLAUDE.md` the agent reads for its constraints.

| Agent | File | When to invoke |
|-------|------|----------------|
| `requirements-analyst` | `agents/requirements-analyst.md` | Before `architect`, whenever a task arrives as a raw natural-language requirement or feature request rather than an already-detailed bug report or fully-specified technical task. Surfaces ambiguities, missing acceptance criteria, business rules, edge cases and technical implications as explicit Blocking/Non-blocking questions. Never fills a gap with an assumption and never proposes a design. Read only. |
| `investigator` | `agents/investigator.md` | When a problem is reported but its cause is unknown: intermittent failures, race conditions, data inconsistencies, unexplained errors, flaky tests. Traces the path across FE, API, service, database and external systems and returns hypotheses with evidence and confidence. No code changes. Call before `architect` when the problem itself is not yet understood. |
| `architect` | `agents/architect.md` | Before implementing any non-trivial feature, refactor, new bounded context, cross-repository change, infrastructure addition, or contract change. Produces a structured proposal only — no code changes. Call this agent first; implement only after the proposal is reviewed. |
| `test-strategist` | `agents/test-strategist.md` | Two moments. Before implementation: after `architect`, to choose which test level proves each risk and to produce the test plan and acceptance mapping. After implementation: to verify the tests actually prove the acceptance criteria. Read only; it specifies tests, `implementer` writes them. |
| `refactorer` | `agents/refactorer.md` | When code works correctly but its structure is a problem - duplication, oversized methods/classes, poor naming, coupling, dead code, technical debt. Improves structure only; behavior must be byte-for-byte identical, proven by the same tests passing before and after. Refuses to run without a green baseline; hands any bug it finds to `investigator` instead of fixing it. Requires no design decision - use `architect` instead if the fix needs a new abstraction, port, or contract change. |
| `implementer` | `agents/implementer.md` | Once an architecture decision, implementation plan and acceptance criteria exist. Writes the code, adds tests, preserves backward compatibility, runs the repository gates and commits. Does not design: if the plan is insufficient it stops and reports back instead of improvising. |
| `core-publisher` | `agents/core-publisher.md` | After `code-reviewer-kiu` passes and before `gitFlow`, whenever `qa-platformCore` changed, or whenever `qa-platformBE/lib/core-m2` is empty or has no folder for the current `coreVersion` (the fresh-clone case). Not called when Core did not change. Gates and publishes Core with `publishToMavenLocal`, syncs `lib/core-m2`, commits the new `core.ref` on this task's branch, and re-verifies the BE gate against the new artifacts. Holds `.core-publish.lock` so only one publish runs across all sessions; when several sessions finished Core work, the last one publishes once for the whole batch. Refuses a non-SNAPSHOT release, which belongs to the `v*` tag workflow. Writes only `core.ref`; never merges or pushes. |
| `gitFlow` | `agents/gitFlow.md` | After any feature, fix or refactor task is complete, reviewed, and published by `core-publisher`. The implementing agent calls gitFlow, waits for its result, fixes any compilation or test error gitFlow reports, and retries gitFlow until it exits green. Merges into `develop`, pushes, then removes this task's worktree and branch. Stops at `develop`; promotes to `master` only when the user asks for it literally, and only after `security-guardian` clears it. |
| `security-guardian` | `agents/security-guardian.md` | Exclusive scope: authentication, authorization, JWT/session, secrets, injection, XSS, CSRF, SSRF, insecure APIs, sensitive data exposure, dependency vulnerabilities, permissions. Mandatory and blocking before `gitFlow` promotes `develop` to `master`, or before any release. Also proactive after any change touching auth/security/connector/outbound code. Files a GitHub issue only for BLOCKER/HIGH and hands each to `investigator` or `architect`. Read only. Named `-guardian` to avoid colliding with the built-in `security-review` skill. |
| `code-reviewer-kiu` | `agents/code-reviewer-kiu.md` | After `test-strategist` verifies the implementation, and before `core-publisher` and `gitFlow` — so a BLOCKER is caught before any artifact is published and before anything is merged. Reviews the feature-branch diff for bugs, races, security, coupling and maintainability issues. Files a GitHub issue only for BLOCKER/HIGH findings and hands each one to `investigator` or `architect` to restart the pipeline. Read only; never fixes anything itself. Named `-kiu` to avoid colliding with the built-in `code-reviewer` subagent type. |

### Orchestration protocol

-1. If the task arrives as a raw natural-language requirement or feature request (not an
   already-detailed bug report, not an already fully-specified technical task), call
   `requirements-analyst` first and wait for its report.
   - If it has Blocking questions, stop and report them to the user. Do not proceed to Task
     sizing, `investigator`, or `architect` until the user answers them.
   - Once Blocking questions are answered (or there were none), continue with its Non-blocking
     assumptions as explicit input to whichever step comes next.
0. If the task is a bug or an unexplained behaviour whose cause is not yet known, call
   `investigator` first and wait for its report. Do not design or implement a fix until
   the root cause is established with evidence. If the report names the cause with high
   confidence and the fix is trivial and local, go straight to implementation; otherwise
   pass the report to `architect`.
0a. Before any agent edits a file, create this task's worktree and branch, and pass its absolute
   path to every agent you invoke afterwards. Read-only steps (`requirements-analyst`,
   `investigator`, `architect`, `test-strategist`, `security-guardian`) may run before this, in the
   main checkout, since they write nothing.
   - `git fetch origin`, then `git worktree list` to confirm no worktree already exists for this issue.
   - `git worktree add ../<repo>.worktrees/<issue>-<slug> -b feature/<issue>-<slug> origin/develop`
     in each repository the task will change.
   - If a worktree for this issue already exists, reuse it, or stop and ask the user. Never create a
     second one for the same branch, and never work in the main checkout instead.
   - Every write agent (`implementer`, `refactorer`, `core-publisher`) receives that path and works
     only inside it. An agent that was given no worktree path must stop and ask for one rather than
     editing the shared checkout.
1. For any non-trivial task, call `architect` and wait for its proposal.
   - Review the proposal (or pass it to the user if a decision is needed).
   - Only proceed to implementation after the proposal is accepted.
1b. Call `test-strategist` in Strategy mode with the accepted proposal and the acceptance
   criteria. Wait for its test strategy. If it finds a criterion that cannot be tested as
   written, resolve that with the user before implementation starts.
2. Call `implementer`, passing all three inputs it requires: the architecture decision,
   the implementation plan (including the test plan from `test-strategist`), and the
   acceptance criteria. Wait for its report.
   - If `implementer` stops because the plan is insufficient, resolve the gap — go back to
     `architect` or to the user for the decision — then call `implementer` again with the
     answer. Never tell it to decide for itself.
3. `implementer` commits its work on the feature branch and returns its report.
3b. Call `test-strategist` in Verification mode with the implementation report, the branch
   and the acceptance criteria. Anything rated "weakly proven" or "not proven" goes back to
   `implementer` with the missing cases, and verification repeats until every criterion is
   "proven". Only then continue to 3c.
3c. Call `code-reviewer-kiu` on the feature-branch diff and wait for its report. This runs
   before anything is published or merged, so a real problem is caught while the diff is
   still cheap to change.
   - For each BLOCKER or HIGH finding it files as a GitHub issue, it also hands the issue
     to `investigator` (cause not yet understood) or `architect` (fix needs a design) —
     this restarts the pipeline from step 0 or step 1 for that issue. Do not continue to
     3d or `gitFlow` with an open BLOCKER on this diff.
   - MEDIUM, LOW and SUGGESTION findings stay in its report only; do not file issues or
     restart the pipeline for them.
3d. If `qa-platformCore` changed in this task — or if `qa-platformBE/lib/core-m2` is empty or
   has no folder for the current `coreVersion` — call `core-publisher` and wait for its report.
   If Core did not change and `lib/core-m2` is consistent, skip this step and say so: there is
   nothing to publish, and calling it would only burn a Core gate and a BE gate.
   - Pass it the worktree paths for this task. It publishes from the Core worktree and syncs and
     commits `core.ref` inside the BE worktree, so the `core.ref` commit lands on this task's branch.
   - It gates and publishes Core locally, syncs `lib/core-m2`, commits the updated `core.ref`,
     and re-runs the BE gate against the new artifacts.
   - **One publish at a time across all sessions.** `~/.m2` and `lib/core-m2` live outside every
     worktree, so it takes `.core-publish.lock` at workspace root first. It never removes a lock it
     does not own, and a lock that looks stale is reported to the user, not deleted.
   - **Several sessions on Core publish once, not once each.** The session that finishes last takes
     the batch: `gitFlow` merges every finished Core branch into local `develop`, `core-publisher`
     publishes that merged state once and runs the BE gate against it, and only then does `gitFlow`
     push and delete the merged branches. A branch joins the batch only when it is pushed, its gate
     passed, its session reported finished, and no live worktree still holds it. Anything unconfirmed
     is left out and named in the report - its own session publishes it later. A session that finds
     another still working on Core releases the lock rather than waiting with it held.
   - If its BE re-verification fails, BE does not build against the new Core. Stop: that is a
     fix task of its own through `investigator` / `architect` / `implementer`, and the pipeline
     resumes at 3c afterwards. Never call `gitFlow` in this state. In a batch nothing was pushed,
     so `origin/develop` is untouched.
4. Orchestrating agent calls `gitFlow` through the subagent tool (Claude Code `Agent`, Cursor `Task`), passing: changed repos, branch name,
   the worktree path for each changed repo, and a summary of what was done.
5. Orchestrating agent waits synchronously for gitFlow's result.
6. If gitFlow reports a code error it could not resolve:
   - Read the error and the stack trace.
   - Apply the minimal fix, or hand it back to `implementer` with the failure output.
   - Call gitFlow again with "fix applied, please retry from Step 4".
7. If gitFlow reports a git or environment error (merge conflict, remote reject, dirty
   worktree): stop and report to the user — do not attempt to resolve git state blindly.
7b. **The pipeline ends at `develop`.** `gitFlow` merges into `develop`, pushes, removes this task's
   worktree and deletes its branch, and stops there. It does not promote to `master`, whatever the
   gates say.
   A `develop`-to-`master` promotion happens only when the user asks for it literally, as its own
   task. When that request comes, `gitFlow` must first call `security-guardian` on the full
   `develop`-to-`master` diff and wait for its verdict.
   - "Clear to promote": `gitFlow` proceeds to merge `develop` into `master`.
   - "Do not promote": `gitFlow` stops before the master merge and reports the blocking
     issue numbers, the same as a failing gate. The BLOCKER findings go through
     `investigator`/`architect`/`implementer` as their own task before promotion is retried.
   This gate cannot be skipped for a master promotion or a release, regardless of how the
   task started. Only one promotion runs at a time: `master` is shared by every session.
8. When gitFlow exits green, the work has shipped to `develop`. Do not review again here — the review
   already happened at 3c, on the same diff, before the merge. Confirm that the worktree and branch
   for this task are gone; a leftover worktree is a half-finished task, not a finished one.
9. Report the final summary to the user: the `code-reviewer-kiu` report and any issues it
   filed and handed off, what `core-publisher` published and synced (or that it was already
   consistent, and whether it waited on the lock), gitFlow's merge and push result, and the
   worktree and branch it removed.

Never bypass gitFlow to do git work directly at the end of a task — the gate and
merge steps belong to gitFlow so the rules are applied consistently.

### How to register a new subagent here

When a new agent file is created in `.claude/agents/` (and its Cursor counterpart), add one row to the table above with:
- **Agent**: the `name` field from the file's YAML frontmatter.
- **File**: path relative to workspace root.
- **When to invoke**: one sentence naming the trigger condition and which agent should call it.

Every agent file must also contain a "Tools and skills" section listing the tools and skills
it should use and the ones it must not use (for read-only agents: every write tool).

Then add a step to the Orchestration protocol if the new agent needs to be called in a
specific sequence relative to existing agents (for example, a `codeReview` agent that
must run before `gitFlow`).

There is deliberately no per-repository variant of any agent here (no `be-architect`,
`core-reviewer`, and so on). A repository-specific need is a constraint the existing agent
reads from that repository's own `AGENTS.md`/`CLAUDE.md`, not a reason for a second agent
file with the same job.

## Routing a code-reviewer-kiu MEDIUM/LOW finding to refactorer

`code-reviewer-kiu` reports MEDIUM/LOW findings (duplication, coupling, maintainability)
without filing an issue or restarting the pipeline for them (see its own agent file - this
is deliberate noise control). When the user wants one of those addressed, call `refactorer`
with the specific finding as its scope. This is on request, not automatic - do not queue
every MEDIUM/LOW finding as a refactoring task without being asked.
