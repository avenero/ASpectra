---
name: AGENTS
model: claude-sonnet-5-5[]
---

# AGENTS.md - A·Spectra workspace (Cursor and other agents)

This pipeline is mirrored for Claude Code: the agent files live flat in `.claude/agents/` and this
document's counterpart is `.claude/ORCHESTRATION.md`, imported into every Claude Code session by
`.claude/CLAUDE.md`. Same ten agents, same orchestration protocol, adapted to Claude Code's real
tool names - `gh` CLI instead of the GitHub MCP, the `agent-browser` skill instead of the Cursor
browser MCP, no Context7 since none is configured for Claude Code here. If you change an agent's
behavior, contract, or the orchestration protocol here, make the same change on the Claude side.

Rules for any agent working in this workspace. It carries the same rules Claude Code follows, from
`.claude/CLAUDE.md`, which remains the source of truth. When the two disagree, `.claude/CLAUDE.md` wins;
report the difference instead of guessing.

Paths below are relative to the workspace root (the folder that holds `.cursor/`). Read this file plus
the guidance of the repository you change, and only that one. Each repository has an `AGENTS.md` with its
own rules and gate; its `.claude/CLAUDE.md` is the source of truth behind it:

| Repository | Agent rules | Source of truth |
|---|---|---|
| `qa-platformCore` | `qa-platformCore/AGENTS.md` | `qa-platformCore/.claude/CLAUDE.md` |
| `qa-platformBE` | `qa-platformBE/AGENTS.md` | `qa-platformBE/.claude/CLAUDE.md` |
| `qa-platformFE` | `qa-platformFE/AGENTS.md` | `qa-platformFE/.claude/CLAUDE.md` |

Backlog: tracked directly in GitHub issues per repository; open owner decisions and pending
documentation corrections are on [qa-platformBE #124](https://github.com/avenero/qa-platformBE/issues/124).
Architecture views: `docs/diagrams/`.
Nothing else in the root is a plan. The root is **not a git repository**; each `qa-platform*` folder is its own.

## The product

An **execution overlay**. The customer's test management system stays the source of truth. The platform
imports selected cases, normalizes them to a canonical model, requires a **human** to review and approve
them, runs them deterministically through `qa-platformCore`, stores evidence, and writes results back
where the cases came from. The value claim: a machine never decided anything a human did not approve.

- Execution layers: HTTP/API, Web, Mobile, Database. Locally written `.feature` files can be imported too.
- **Release scope (1.0):** Jira + Xray Server/DC only, AI off, sign-up activated by
  an admin; mobile on Android emulators and iOS simulators through the mobile agent. TestRail has its own
  later release; Postman, Swagger and user-story imports come in 1.1; Jira + Xray Cloud stays hidden in
  1.0. Azure DevOps has **no import adapter**; never claim it.

| Repository | What it is | Stack |
|---|---|---|
| `qa-platformCore` | Deterministic test engine: Cucumber runtime, step libraries, evidence, `mobile-agent` | Java 21, Gradle multi-module |
| `qa-platformBE` | Overlay backend: connectors, canonical model, review, execution, evidence, sync, all AI | Spring Boot 3.5, Java 21, hexagonal, PostgreSQL/Flyway |
| `qa-platformFE` | Console SPA | React, TypeScript, Vite, TanStack Query, i18next (en/es/fr), Tailwind |

## How to work (mandatory)

- Be simple. Work incrementally in small steps, and validate each step before the next.
- Do not overengineer or program defensively. Handle exceptions only when needed. Use the current APIs of
  the libraries the repository already uses.
- **Prove before fixing.** Find the root cause and reproduce it (a failing test is best), then fix. One
  test at a time. No workarounds, no jumping to conclusions.
- Docstrings are clear and short; comments elsewhere are rare. **No emojis** anywhere: code, docs, logs,
  UI copy or commits.
- English for code, comments, logs, exceptions, API `detail` and prompts. User-facing text goes through
  FE i18n keys (en, es, fr) and backend error `code`s. Gherkin step patterns are frozen: never translate them.
- Keep each README concise and current.
- **Analyze before implementing.** Check architecture, contracts and consumers first. If you find a
  design gap or violation, stop, redesign (a two-pass design review: propose, challenge, refine), and report
  a summary.
- At the end of a task, list in chat any rule, standard or architecture violation you found.

## Working rules (mandatory)

1. **No hallucinations or undefined behavior.** Do not invent APIs, fields, migrations, connectors,
   config keys or product claims. Read the code; ask before implementing anything not specified.
2. **Never edit `CLAUDE.md` or `AGENTS.md` on your own initiative.** Propose changes in chat; edit only
   with explicit approval.
3. **Smallest solution.** No speculative abstractions, queues, caches, services or bounded contexts.
4. **Ask before deleting code or changing existing behavior** that the task does not explicitly require.
5. **Core stays deterministic and isolated.** Integrations and AI live in BE only. A Core contract change
   needs approval and must keep both consumers (BE and standalone) working.
6. **Three separate state machines:** canonical review status, execution status, sync-job status.
7. **Design for two backend instances** sharing one database. Cross-request or cross-instance state goes
   through the database (row claims, conditional updates), never through JVM memory or local files.
8. **Repositories are self-contained.** A cross-repo dependency is allowed only where marked `allow-cross-repo:`.
9. **Fail closed.** Deterministic over AI, strict schema validation over permissive parsing, durable jobs
   over fire-and-forget. A step that cannot work is marked unavailable, never silently passes.
10. **Security boundaries:** validate outbound destinations, authorize before loading project resources,
    keep secrets in the encrypted store, and prove every security control with an attack-path integration test.
11. **Contracts are complete and traced.** A field is honored end to end or removed. For any changed DTO,
    event, wire format or config key, list every touchpoint (BE, Core, FE types, Zod, mobile agent) with file:line.
12. **Frozen contracts:** Cucumber `@StepDef` patterns and step ids never change without an approved baseline change.
13. **Size limits for new code:** 400 lines per class, 80 per method, one implementation per concern.
14. **Migrations:** never edit an applied migration. A schema change is a new forward `V*` migration with its `U*` undo.
15. **Keep issues current.** After finishing a task, update or close its GitHub issue to reflect
    current status. A status claim needs a commit reference and green gates (G16).

## What AI does, and never does

AI lives only in BE (`intelligence` bounded context); Core and FE have none. `AI_ENABLED` is false by default.
AI never: decides pass/fail; produces counts, durations or statuses; writes a sync payload or renders
Gherkin; makes an authorization or outbound-policy decision; computes a source version hash; touches a
credential; **invents test data, credentials, URLs, selectors or expected results**; or alters a
`@StepDef` pattern. Nothing a model produces reaches domain state without a human decision. Every
capability is schema-validated, fail-closed, and has a deterministic fallback.

## Cross-repository facts that cost time

- **Core reaches BE only through `qa-platformBE/lib/core-m2`.** After any Core change, run
  `cd qa-platformCore && ./gradlew publishToMavenLocal`, then `cd qa-platformBE && ./gradlew syncCoreFromMavenLocal`.
  Skipping the sync silently runs stale Core. `coreVersion` must match in both `gradle.properties` files.
  `qa-platformBE/core.ref` must point at the Core commit BE was tested with. In the pipeline this is
  `core-publisher`'s job, and it runs before `gitFlow`; do it by hand only outside the pipeline.
  Note `syncCoreFromMavenLocal` deletes `lib/core-m2/com` before it copies, so never run it until the
  artifacts exist in `~/.m2` at `coreVersion`. `dependencyUpdates` is a dependency-update report, not a sync.
- When BE starts calling a new Core member, pin it in
  `qa-platformCore/http-core/src/test/java/com/qa/httpcore/arch/ConsumerSurfaceContractTest.java`.
- Core input is Gherkin text (file paths in process; `featureContents` over the mobile agent's wire v1).
  Wire changes are additive only.
- One outbound address policy: Core `OutboundHostPolicy`; BE `OutboundUrlPolicy` delegates to it.
  Private hosts are allowed only with `qa.outbound.allow-private-hosts=true` (true in `local`/`docker`, false in `prod`).
- `SPRING_PROFILES_ACTIVE` has no default (`local`, `docker`, `prod`). **Never commit `.env`** or put
  credentials in Compose files.

## Guardrails (each has an automated guard; keep both green)

G1 secrets only from the encrypted store. G2 outbound URLs checked on save and before each call, no
redirects. G3 authorize before loading; cross-project ids are 404. G4 no external I/O inside a DB
transaction; background work uses row claims. G5 the outbox is the only durable-event path; consumers are
idempotent. G6 terminal execution states are immutable; transitions are conditional. G7 contract fields are
honored end to end or removed. G8 every schema file is loaded and used. G9 every config key read is in
`application.yml`. G10 `META-INF/services` entries are valid. G11 no provider or LLM concepts in Core.
G12 FE eslint boundaries; pages use VMs or query hooks, and components never import `api/` modules.
G13 mocked Playwright specs fail on any unmocked `/api` call. G14 STOMP is authenticated and authorized.
G15 seed credentials need an opt-in Flyway placeholder. G16 status claims need a commit and green gates.
G17 repositories are self-contained. G18 step input cannot reach env, files or JVM properties.
G19 runtime state belongs to the execution or scenario, not the JVM. G20 execution is fail-closed.
G21 security controls have attack-path ITs. G22 project-scoped id resolution; external links use
`safeHref`; the SPA ships a CSP. G23 one redactor; INFO logs in prod; no secret fallbacks. G24 required
gates can fail and always run. G25 no emojis; size limits. G26 every connector passes the shared contract
suite against its simulator. G27 provider names stay inside their connector package. G28-G33 AI
governance (isolated providers, registered capabilities, schemas, evals, budgets, versioned prompts).
G34 frozen step patterns and ids. G35 full contract trace. G36 no unapproved `CLAUDE.md` edits.
Full statements: the repository `CLAUDE.md` files.

## Gates (GitHub Actions is off; every gate runs locally, and green means all of it)

| Repo | Gate |
|---|---|
| Core | `./gradlew build -Pstrict` with Docker running; `scripts/check-*.sh` (five guard scripts); `scripts/cli-smoke.sh` |
| BE | `env -u SPRING_PROFILES_ACTIVE ./gradlew clean check -Pstrict` |
| FE | `scripts/check-ci-gates.sh`, `scripts/check-self-contained.sh`, `npm run type-check`, `npm run lint`, `npm run test:coverage`, `npm run build`, `npm run test:e2e` |

- Testcontainers on this machine (Rancher Desktop) needs `export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock`.
- The BE gate is not concurrency-safe: never run a targeted Gradle test in the same checkout while `check` runs.
- If FE type-check fails locally but not in CI, run `npm ci` first (stale `node_modules`).

## Concurrent sessions: one session, one worktree, one branch

Several sessions work in this workspace at the same time. They never share a checkout. A session that
is about to work on a feature creates its own git worktree for that branch and works only there.

```
cd <repo> && git fetch origin && git worktree list
git worktree add ../<repo>.worktrees/<issue>-<kebab-slug> -b feature/<issue>-<kebab-slug> origin/develop
```

- The worktree lives at workspace root as a sibling of the three checkouts:
  `<repo>.worktrees/<issue>-<kebab-slug>`, the slug matching the branch, so branch-to-directory is mechanical.
- Always branch from `origin/develop` after a fetch, never from a stale local `develop`.
- Run `git worktree list` first; if one already exists for that issue, reuse it or stop and ask the user.
  Git refuses the same branch in two worktrees anyway.
- **Every agent that edits files works only inside its own worktree path.** Never edit the main checkout,
  never edit another session's worktree, even when `git worktree list` reveals it.
- The main checkout stays on `develop` and is left alone. If it is on someone else's branch, has
  uncommitted work, or holds `develop` with unpushed merges, stop and ask the user.
- Never use a bare `git stash` or `git stash pop`: **the stash is shared across every worktree of the
  same repository**, so a worktree does not isolate it. Use a WIP commit on your own branch.
- `gitFlow` owns cleanup, after the merge is confirmed on the remote: `git worktree remove <path>`
  (never `rm -rf`, which leaves stale metadata in `.git/worktrees/`) and `git branch -d <branch>`.
  It then removes the `<repo>.worktrees/` directory itself with `rmdir`, which succeeds only once that
  directory is empty. The directory is shared by every session of the repository, so a refusal means
  another session still has a worktree in it: it is reported and left alone, never emptied by force.

**What a worktree does not isolate** - single shared resources that serialize across all sessions:

| Resource | Rule |
|---|---|
| `~/.m2` and `qa-platformBE/lib/core-m2` | One Core publish at a time, serialized by `.core-publish.lock` at workspace root (never removed by a session that does not own it). When several sessions changed Core, the last one to finish publishes once for all of them. |
| `master` | One promotion at a time, and only on a literal request from the user. |
| Fixed ports (FE 3000, Compose, Testcontainers) | Two sessions running a stack at once need different ports. |

***
## Branch strategy (always applies for local and remote branches)

A feature/task branch is never merged directly into master.
The flow is always: feature/<branch> → develop → master.
develop is the integration branch: all feature branches are created from develop and merged back into develop.
**A task ends at develop.** Promotion of develop to master is a separate step that runs only when the user
asks for it literally. It is never part of a feature flow, never implied by a green gate, and never
triggered because develop looks stable.

***
Flow for "New Feature"

1. Create an issue (if it doesn't exist yet)
   gh issue create --title "Feature: <name>" --body "<description, acceptance criteria>"
   This keeps traceability and links the PR to the issue.

2. Create a dedicated worktree and branch from origin/develop
   Naming convention, used for both the branch and its worktree directory:
   feature/<issue-number>-<short-kebab-case-description>
   Example: feature/142-user-authentication -> qa-platformBE.worktrees/142-user-authentication

cd <repo>
git fetch origin
git worktree list
git worktree add ../<repo>.worktrees/142-user-authentication -b feature/142-user-authentication origin/develop
cd ../<repo>.worktrees/142-user-authentication

   Every later step runs inside that worktree. The main checkout stays on develop, untouched.

3. Iterative development with atomic commits
   Commits must follow Conventional Commits:
   feat: add OAuth login
   test: add authentication tests
   docs: update README with new feature
   Every commit must build and pass tests locally (never commit broken code)

4. Tests are mandatory before moving on
   Before considering the feature done, ensure that:
   Write unit tests for the new functionality
   Run the full test suite (not just the new tests)
   Confirm nothing existing was broken
   Compile locally to verify the feature works as expected and without error or warnings
   If any issues or warnings are found, fix them before proceeding to the next step.

5. Run core-publisher - only when Core changed
   Called when `qa-platformCore` changed, or to repair an empty or stale `lib/core-m2`. A task that did
   not touch Core does not call it. It runs before the merge, because its purpose is to prove BE compiles
   against the Core artifacts on disk before anything reaches develop. One publish at a time, serialized
   by `.core-publish.lock` at workspace root.
   When several sessions finished Core work, the last one publishes once for the whole batch: gitFlow
   merges every finished Core branch into local develop, core-publisher publishes that merged state and
   runs the BE gate, and only then is develop pushed. Unconfirmed branches are left out, never merged.

6. Merge the feature branch into develop (gitFlow does this)
   git fetch origin
   If develop moved while this task ran, merge origin/develop into the feature branch inside the
   worktree and re-run the full gate. A gate that was green against an older develop proves nothing.
   Then merge in the main checkout and push:
   git -C <main-checkout> checkout develop
   git -C <main-checkout> pull origin develop
   git -C <main-checkout> merge --no-ff feature/142-user-authentication
   git -C <main-checkout> push origin develop

7. Delete the worktree and the branch, after confirming the merge landed on the remote
   git fetch origin && git log origin/develop --oneline -1
   git -C <main-checkout> worktree remove ../<repo>.worktrees/142-user-authentication
   git -C <main-checkout> branch -d feature/142-user-authentication
   rmdir ../<repo>.worktrees        # only succeeds when empty; shared by every session
   Never `rm -rf` a worktree or its parent directory, and never delete either one before the push is
   confirmed. If `rmdir` refuses, another session is still working there - report it and leave it.
   For a batch, each auxiliary branch is deleted only after `git log --oneline <branch> --not develop`
   prints nothing, with `git branch -d` (never `-D`) as the second safety net. Remove only your own
   worktrees; another session's worktree is its own to clean up.

8. Stop at develop
   Do not promote to master. That is its own task, on an explicit request, and it runs the mandatory
   `security-guardian` gate first.

### Routing a code-reviewer-kiu MEDIUM/LOW finding to refactorer

`code-reviewer-kiu` reports MEDIUM/LOW findings (duplication, coupling, maintainability)
without filing an issue or restarting the pipeline for them (see its own agent file - this
is deliberate noise control). When the user wants one of those addressed, call `refactorer`
with the specific finding as its scope. This is on request, not automatic - do not queue
every MEDIUM/LOW finding as a refactoring task without being asked.

## Skills

- **Browser automation:** `.claude/skills/agent-browser/SKILL.md`. Read it before driving a browser
  (exploratory testing, screenshots, UI checks).
- **Design review:** when the rules above call for one, do two passes: propose the design, attack it
  against these rules and the code, then refine. Report the result before implementing.

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

| Agent | File | When to invoke |
|-------|------|----------------|
| `requirements-analyst` | `.cursor/agents/requirements-analyst.md` | Before `architect`, whenever a task arrives as a raw natural-language requirement or feature request rather than an already-detailed bug report or fully-specified technical task. Surfaces ambiguities, missing acceptance criteria, business rules, edge cases and technical implications as explicit Blocking/Non-blocking questions. Never fills a gap with an assumption and never proposes a design. Read only. |
| `investigator` | `.cursor/agents/investigator.md` | When a problem is reported but its cause is unknown: intermittent failures, race conditions, data inconsistencies, unexplained errors, flaky tests. Traces the path across FE, API, service, database and external systems and returns hypotheses with evidence and confidence. No code changes. Call before `architect` when the problem itself is not yet understood. |
| `architect` | `.cursor/agents/architect.md` | Before implementing any non-trivial feature, refactor, new bounded context, cross-repository change, infrastructure addition, or contract change. Produces a structured proposal only — no code changes. Call this agent first; implement only after the proposal is reviewed. |
| `test-strategist` | `.cursor/agents/test-strategist.md` | Two moments. Before implementation: after `architect`, to choose which test level proves each risk and to produce the test plan and acceptance mapping. After implementation: to verify the tests actually prove the acceptance criteria. Read only; it specifies tests, `implementer` writes them. |
| `refactorer` | `.cursor/agents/refactorer.md` | When code works correctly but its structure is a problem - duplication, oversized methods/classes, poor naming, coupling, dead code, technical debt. Improves structure only; behavior must be byte-for-byte identical, proven by the same tests passing before and after. Refuses to run without a green baseline; hands any bug it finds to `investigator` instead of fixing it. Requires no design decision - use `architect` instead if the fix needs a new abstraction, port, or contract change. |
| `implementer` | `.cursor/agents/implementer.md` | Once an architecture decision, implementation plan and acceptance criteria exist. Writes the code, adds tests, preserves backward compatibility, runs the repository gates and commits. Does not design: if the plan is insufficient it stops and reports back instead of improvising. |
| `core-publisher` | `.cursor/agents/core-publisher.md` | After `code-reviewer-kiu` passes and before `gitFlow`, whenever `qa-platformCore` changed, or whenever `qa-platformBE/lib/core-m2` is empty or has no folder for the current `coreVersion` (the fresh-clone case). Not called when Core did not change. Gates and publishes Core with `publishToMavenLocal`, syncs `lib/core-m2`, commits the new `core.ref` on this task's branch, and re-verifies the BE gate against the new artifacts. Holds `.core-publish.lock` so only one publish runs across all sessions; when several sessions finished Core work, the last one publishes once for the whole batch. Refuses a non-SNAPSHOT release, which belongs to the `v*` tag workflow. Writes only `core.ref`; never merges or pushes. |
| `gitFlow` | `.cursor/agents/gitFlow.md` | After any feature, fix or refactor task is complete, reviewed, and published by `core-publisher`. The implementing agent calls gitFlow, waits for its result, fixes any compilation or test error gitFlow reports, and retries gitFlow until it exits green. Merges into `develop`, pushes, then removes this task's worktree and branch. Stops at `develop`; promotes to `master` only when the user asks for it literally, and only after `security-guardian` clears it. |
| `security-guardian` | `.cursor/agents/security-guardian.md` | Exclusive scope: authentication, authorization, JWT/session, secrets, injection, XSS, CSRF, SSRF, insecure APIs, sensitive data exposure, dependency vulnerabilities, permissions. Mandatory and blocking before `gitFlow` promotes `develop` to `master`, or before any release. Also proactive after any change touching auth/security/connector/outbound code. Files a GitHub issue only for BLOCKER/HIGH and hands each to `investigator` or `architect`. Read only. Named `-guardian` to avoid colliding with the built-in `security-review` subagent type. |
| `code-reviewer-kiu` | `.cursor/agents/code-reviewer-kiu.md` | After `test-strategist` verifies the implementation, and before `core-publisher` and `gitFlow` — so a BLOCKER is caught before any artifact is published and before anything is merged. Reviews the feature-branch diff for bugs, races, security, coupling and maintainability issues. Files a GitHub issue only for BLOCKER/HIGH findings and hands each one to `investigator` or `architect` to restart the pipeline. Read only; never fixes anything itself. Named `-kiu` to avoid colliding with the built-in `code-reviewer` subagent type. |

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
4. Orchestrating agent calls `gitFlow` via Task, passing: changed repos, branch name,
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

When a new `.cursor/agents/<name>.md` is created, add one row to the table above with:
- **Agent**: the `name` field from the file's YAML frontmatter.
- **File**: path relative to workspace root.
- **When to invoke**: one sentence naming the trigger condition and which agent should call it.

Every agent file must also contain a "Tools and skills" section listing the tools and skills
it should use and the ones it must not use (for read-only agents: every write tool).

Then add a step to the Orchestration protocol if the new agent needs to be called in a
specific sequence relative to existing agents (for example, a `codeReview` agent that
must run before `gitFlow`).
