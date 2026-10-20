# CLAUDE.md — workspace overview

## SUMMARY
Guidance for Claude Code across the whole A·Spectra workspace. This file holds only what spans more than one repository. Everything repository-specific lives in that repository's own file:

| Repository | Its guidance |
|---|---|
| `qa-platformCore` | [qa-platformCore/.claude/CLAUDE.md](../qa-platformCore/.claude/CLAUDE.md) |
| `qa-platformBE` | [qa-platformBE/.claude/CLAUDE.md](../qa-platformBE/.claude/CLAUDE.md) |
| `qa-platformFE` | [qa-platformFE/.claude/CLAUDE.md](../qa-platformFE/.claude/CLAUDE.md) |

Read this file plus the one for the repository you are changing and only if you


## Subagent orchestration

@ORCHESTRATION.md

The import above loads the full subagent pipeline automatically into every session: which of
the nine agents in `.claude/agents/` to call for a given task, in what order, the task-sizing
rule that decides how much of the pipeline runs, and the mandatory `security-guardian` gate
before any `develop`-to-`master` promotion. Nothing else needs to open that file on its own
initiative to discover this - it is already in context.

## The product in one sentence

A.spectra is an **execution overlay** for clients (companies, projects) who want to run their test cases
without building an automation framework. From the web console, `qa-platformFE`, they configure executions
(API, web, mobile and database). The customer's test management system (TMS) stays the source of truth:
the platform imports selected cases from it, normalizes them to the canonical model using the steps of
`qa-platformCore`, requires a human to review and approve them, runs them, stores the evidence, and
writes the results back to where the cases came from. Locally written `.feature` files can be imported
the same way.

**Release scope.** The product aims wider than one release. What ships when is decided below:
- **1.0:** Jira + Xray, Server/DC and Cloud, AI off, sign-up activated by an admin. Mobile runs on Android
  emulators and iOS simulators through the mobile agent. Imports from Postman collections, Swagger/OpenAPI
  specs and user stories (`qa-platformFE` `features/imports`) are also part of 1.0.
- **Later:** TestRail has its own release.
- Azure DevOps has no import adapter; never claim it.

That sentence constrains every architectural choice. The product's value claim is that a machine did not decide anything a human did not approve.

Execution coverage: HTTP/API, Web, Mobile, Database.

## Repository layout

The root is **not a git repo** and nothing there is built or run by CI; it holds planning documents
only. `qa-platformCore`, `qa-platformBE` and `qa-platformFE` are each their own git repository


| Repository | What it is | Stack |
|---|---|---|
| `qa-platformCore` | The deterministic test engine. Cucumber runtime, step libraries for HTTP, Web (Playwright), Mobile (Appium) and Database (JDBC), evidence and reporting. No knowledge of TMS tools or LLMs. Consumed both as Maven artifacts by BE and standalone. | Java 21, Gradle multi-module, ~53,800 lines main source, 6 modules |
| `qa-platformBE` | The overlay backend. Connectors, canonical model, review lifecycle, execution launch, evidence, result sync. Owns every integration and all AI. | Spring Boot 3.5, Java 21, hexagonal + bounded contexts, PostgreSQL/Flyway, ~51,700 lines main source |
| `qa-platformFE` | The single-page console. Integrations settings and the execution overlay. | React + TypeScript + Vite + TanStack Query + i18next + Tailwind, ~48,000 lines under `src/` |

Each repository is self-contained (guardrail G17): its own `.github/workflows/`, scripts, Docker context and env template.

## End-to-end flow, one line per stage

| # | Stage | Entry point | Owner |
|---|---|---|---|
| 1 | Connect | `PUT /api/projects/{id}/integrations`, `POST .../integrations/detect` | BE `project` BC + `testcase.infrastructure.connector.*`; FE `ProjectIntegrationsPage` |
| 2 | Import | `POST /api/projects/{p}/connectors/{c}/imports` (202 + durable job row) | BE `ConnectorImportApplicationService`, `ImportJobWorkerAdapter` |
| 3 | Canonical + review | `CanonicalTestCaseApplicationService`, `SourceCaseNormalizer`, `CatalogStepMatcher` | BE `testcase` BC |
| 4 | Plan + projection | `ExecutionPlanApplicationService` -> `JpaExecutableProjectionAdapter` | BE `testcase` -> legacy `model.project.TestScenario` |
| 5 | Run | `ExecutionApplicationService.launch` -> `ExecutionAsyncRunner` -> `FeatureGeneratorService` -> Core | BE `execution` BC + Core |
| 6 | Evidence | `BEStepReporter`, `ExecutionArtifactService`, `HtmlReportGeneratorService` | BE `execution` BC |
| 7 | Sync | outbox -> `IntegrationSyncJobApplicationService` -> `RunResultSetBuilder` -> adapter | BE `testcase` BC |
| 8 | Console | `ExecutionOverlayPage` + `useExecutionOverlayVM` | FE `features/overlay` |

**Stage 4 has no screen of its own, by design.** Plan has no dedicated FE screen: it is folded into the
environment-and-runtime-config step (environment choice, layer-specific runtime config such as whether a
WEB run uses Playwright) that every "Run"/"Execute" action collects immediately before launch, on
`ExecutionOverlayPage`'s `RunPanel` and on the Run/Execute modals of `SuiteDetailPage`, `FeatureDetailPage`,
`ScenarioBuilderPage` and `ScenarioSuggestionsPage`. This is the intended design, confirmed 2026-10-01 — do
not read it as a missing screen, and do not build one without a new product decision.

Two structural properties everything else depends on:

- **Every crossing of a process boundary is a durable job with a row claim**: insert a row, claim it with a conditional update, do the external I/O outside any transaction, record the outcome. There is no fire-and-forget external call anywhere in the overlay path.
- **Three state machines are deliberately independent** (working rule 4): canonical review status, execution status, sync job status. Nothing folds one into another.

## Cross-repository mechanics

**Core input is Gherkin text, not an object model.** BE hands Core `.feature` file paths (in process)
or `featureContents` strings (mobile agent `POST /v1/runs`, SSE events on `/v1/runs/{id}/events`,
agent port 8090, `AGENT_AUTH_TOKEN`). Wire v1 versioning is only the `/v1` path; new events such as
`STEP_ATTACHMENT` are additive.

**Frozen Core surface is larger than `com.qa.common.api..`.** BE also calls `@Internal`
`InProcessTransport`, `HttpAgentTransport`, `CucumberRuntimeEngine`, `StepDiscoveryService`,
`SSLContextFactory` and http-core reporting types. The exact members are pinned by
`qa-platformCore/http-core/src/test/java/com/qa/httpcore/arch/ConsumerSurfaceContractTest.java`.
**When BE starts calling a new Core member, add it there.**

**Outbound policy has one rule set.** Core `com.qa.common.api.security.OutboundHostPolicy` is the
single address rule set; BE `OutboundUrlPolicy` delegates to it. Link-local and cloud metadata
addresses are always denied; loopback, private, CGNAT and unique-local addresses are denied unless
`qa.outbound.allow-private-hosts=true` (env `QA_OUTBOUND_ALLOW_PRIVATE_HOSTS`; true in `local` and
`docker`, false by default and in `prod`).

**Artifact dependency, and the gotcha that costs the most time.** BE depends on
`com.qa:{http,web,mobile,database}-core:${coreVersion}`.

- `coreVersion` must match in `qa-platformCore/gradle.properties` and `qa-platformBE/gradle.properties`
  (local dev uses `-SNAPSHOT`; CI checks it).
- BE resolves `com.qa` artifacts **only** from `qa-platformBE/lib/core-m2` (plus GitHub Packages when
  credentials are set).
- After any Core change: `cd qa-platformCore && ./gradlew publishToMavenLocal`, then
  `cd qa-platformBE && ./gradlew syncCoreFromMavenLocal`. `bootRun`, tests and the BE `Dockerfile`
  read that folder — **skipping the sync silently runs stale Core.**
- `./gradlew printResolvedCoreModules` (BE) shows which Core artifacts actually resolved.
- `qa-platformBE/core.ref` records the Core commit BE CI checks out; `fe.ref` the FE commit deploys
  build. Both must point at commits that were tested together.


**Execution is still in-process.** A run is submitted into the instance's own JVM, so a rolling restart still loses in-flight runs. Making that lossless means out-of-process execution over the existing mobile-agent wire, and it is a separate decision, deliberately not part of OPS-2.

## Local stack
Requirements: Java 21, Node.js 22.12+ (`.nvmrc`), Docker with Compose v2, OpenSSL, `envsubst`.

- `SPRING_PROFILES_ACTIVE` is required (`local`, `docker`, `prod`; there is no default). Compose sets
  `docker`; set it yourself for `./gradlew bootRun`. Profile `dev-email` adds Mailhog.
- The FE container's nginx listens on 8080 and is published as 3000. There is no Selenium service;
  web runs use Playwright inside the backend.
- Bootstrap admin: V80 locks it outside `local`/`docker` unless `ADMIN_BOOTSTRAP_PASSWORD` is set.
  The demo account is inactive unless the `demoAccountEnabled` Flyway placeholder is true.
- **Never commit `.env`.** Connector credentials are entered in the Integrations settings
  (write-only, encrypted per credential kind); never put them in Compose files.

## VERY IMPORTANT
- Be simple. Approach tasks in a simple, incremental way.
- Work incrementally ALWAYS. Small, simple steps. Validate and check each increment before moving on.
- Use LATEST apis as of NOW.

## MANDATORY CODE STYLE
- Do not overengineer. Do not program defensively. Use exception manager only when needed.
- Identify root cause before fixing issues. Prove with evidence, then fix.
- Work incrementally with small steps. Validate each increment.
- Use LATEST librery APIs.
- Favor clear, concise docstring comments. Be sparing with comments outside docstrings.
- Never use emojis in code, documentations, loggings and statements.
- Keep always README.md concise and update.

## IMPORTANT - DEBUGGING AND FIXING
- When troubleshooting problems. ALWAYS identify root cause BEFORE fixing.
- PROVE THE PROBLEM FRIST - don't guess.
- Try oine test at a time. Be methodical.
- Don't jum to conclusions. Don't apply workarounds.


## What the AI does, and never does

AI lives **only in qa-platformBE**. Core is free of it by design (G11) and the frontend holds no AI
logic at all — no provider key in the browser, no prompt assembled client-side, no browser-side model.

Never, under any circumstance:

- decide pass/fail of a step, scenario or execution — Core alone decides (G20);
- produce any count, duration or status in a report, sync payload or dashboard — they are computed;
- produce the content of an outbound sync payload, or render Gherkin;
- make a permission, authorization or `OutboundHostPolicy` decision;
- produce a source version hash (re-import idempotency depends on byte stability);
- touch a credential — secrets never enter a prompt;
- **invent test data, credentials, URLs, selectors or expected outcomes.** A case whose expected
  result was written by a model asserts the model's guess and will pass. A model may say "an expected
  result is missing here"; it may never supply one;
- alter a Cucumber `@StepDef` pattern. Patterns are the executable identity of every stored scenario
  and a frozen contract.

Always: nothing an LLM produces reaches domain state without a human decision; every capability is schema-validated fail-closed with referential checks against the live catalog; every capability has a named deterministic fallback, so nothing becomes unavailable because AI is unavailable; deterministic results are never overwritten by a model. Today `AI_ENABLED` is false by default.


## Working rules for agents (mandatory)

1. **No hallucinations or undefined behavior.**  
   Do not invent APIs, fields, migrations, connectors, configuration, product claims, or behavior. Inspect the code/spec first; ask before implementing anything undefined.

2. **Never edit `CLAUDE.md` or `ORCHESTRATION.md` without explicit approval.**  
   Report proposed changes, missing rules, or violations in chat.

3. **Analyze before implementing.**  
   Verify architecture, contracts, dependencies, and affected consumers first. If a design gap or required change is found, stop and resolve the design before implementation; use `ralph-loop --max-iterations=2` when required by the task workflow.

4. **Prefer the smallest solution.**  
   No speculative abstractions, defensive code, queues, caches, services, bounded contexts, or other infrastructure without a demonstrated need or explicit request.

5. **Ask before deleting or changing existing behavior.**  
   Confirm before removing apparently unused/obsolete code or changing behavior that is not explicitly required.

6. **Keep Core deterministic and isolated.**  
   Core owns deterministic execution. Integrations and AI/LLM belong in BE. Core contract changes require approval and must preserve existing consumers.

7. **Keep state machines separate.**  
   Review status, execution state, and sync-job state must remain independent.

8. **Keep shared runtime state in the database.**  
   Design for two instances. Do not use JVM/local-file state for cross-request or cross-instance coordination. Use database claims/conditional updates; do not infer peer work is dead during startup.

9. **Keep repositories self-contained.**  
   Cross-repository dependencies are allowed only where explicitly documented with `allow-cross-repo:`.

10. **Fail closed and prefer deterministic behavior.**  
    Prefer deterministic mapping over AI, strict schema validation over permissive parsing, and durable jobs over fire-and-forget. Unavailable steps must never silently succeed.

11. **Protect security boundaries.**  
    Validate outbound destinations, authorize project resources before loading them, isolate secrets, authenticate/authorize STOMP, protect referenced IDs, and test security controls through attack-path integration tests.

12. **Contracts must be complete and traceable.**  
    A contract field must be implemented end-to-end or removed. Trace changed DTOs, events, wire formats, and config keys across BE, Core, FE, Zod, and mobile-agent.

13. **AI must remain governed and non-authoritative.**  
    AI providers are isolated behind registered capabilities. Every capability has a schema, redaction policy, deterministic fallback, evaluation, and governance controls. LLM output cannot mutate domain state without human approval or an explicit deterministic rule.

14. **Keep provider-specific code isolated.**  
    Provider names and concepts belong only inside their connector/integration packages.

15. **Preserve frozen contracts.**  
    Cucumber step patterns and IDs must not change without an explicit approved baseline change.

16. **Use English for engineering content.**  
    Code, comments, logs, exceptions, API `detail`, and prompts are English. User-facing text uses FE i18n and backend error codes. No emojis.

17. **Keep code simple and bounded.**  
    One implementation per concern. New classes ≤400 lines and new methods ≤80 lines.

18. **Keep documentation and plans current.**  
    Update affected documentation when contracts change.Remove completed tasks and retain only unfinished work.


## Shared workspace rules (several agents work here at once)

**One session, one worktree, one branch.** Concurrent sessions never share a checkout. A session that
is about to work on a feature creates its own git worktree for that branch and works only there:

```
cd <repo> && git fetch origin && git worktree list
git worktree add ../<repo>.worktrees/<issue>-<kebab-slug> -b feature/<issue>-<kebab-slug> origin/develop
```

- The worktree directory is a sibling of the three checkouts, at workspace root:
  `<repo>.worktrees/<issue>-<kebab-slug>`. The slug matches the branch name, so the mapping from
  branch to directory is mechanical and any session can find it.
- Branch from `origin/develop` after a `git fetch`, never from a stale local `develop`.
- Run `git worktree list` first. If a worktree for that issue already exists, reuse it or stop and ask
  the user - never create a second one for the same branch (git refuses the same branch in two
  worktrees anyway).
- Every agent that edits files works **only inside its own worktree path**. Never edit the main
  checkout, and never edit another session's worktree, even when `git worktree list` reveals it.
- The main checkout stays on `develop` and is left alone. If it is on someone else's branch, has
  uncommitted work, or holds `develop` with unpushed merges, stop and ask the user. Never route around it.
- Stage by explicit path, never `git add -A` or `git add .`: another agent's files may be in the tree.
- Never use a bare `git stash` or `git stash pop`; **the stash is shared across every worktree of the
  same repository**, so a worktree does not isolate it. Use a WIP commit on your own branch instead.
- Cleanup belongs to `gitFlow` and happens only after the merge is confirmed on the remote: it removes
  the worktree with `git worktree remove <path>` (never `rm -rf`, which leaves stale metadata in
  `.git/worktrees/`) and deletes the feature branch. It then removes the `<repo>.worktrees/` directory
  itself with `rmdir`, which succeeds only once that directory is empty. That directory is shared by
  every session of the repository, so a refusal means another session still has a worktree in it: it is
  reported and left alone, never emptied by force.

**What a worktree does not isolate.** These are single shared resources that live outside any
worktree, so they serialize across all sessions no matter how many worktrees exist:

- `~/.m2` and `qa-platformBE/lib/core-m2` - only one Core publish may run at a time, serialized by the
  `.core-publish.lock` lock at workspace root. When several sessions changed Core, the last one to finish
  publishes once for all of them instead of each publishing its own.
- `master` - one promotion at a time, and only when the user asks for it literally.
- Fixed ports (FE dev server 3000, Compose services, Testcontainers): two sessions running a stack at
  once need different ports.
- `.claude/` is never committed in any of the three repositories.
- GitHub Actions is off; every gate runs locally. Testcontainers on Rancher Desktop needs
  `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock`. Run the BE gate as
  `env -u SPRING_PROFILES_ACTIVE ./gradlew clean check -Pstrict`, and never run a targeted Gradle test in
  the same checkout while it runs.
- Cursor and other agents follow `.cursor/agents/AGENTS.md` (workspace) and the `AGENTS.md` of each
  repository. Claude Code's own equivalent is `.claude/ORCHESTRATION.md`, imported automatically
  above. The two are kept in sync by hand; a rule change in one belongs in both.

## Guardrail index

| # | Guardrail | Owner |
|---|---|---|
| G1 | Secrets only from encrypted store; input cannot select source | BE |
| G2 | Outbound URLs validated on save and before calls; no redirects | BE |
| G3 | Project authorization before resource loading; unknown/cross-project IDs return 404 | BE |
| G4 | No external I/O in DB transactions; background work uses row claims | BE |
| G5 | Outbox is the only durable-event publication path; consumers are idempotent | BE |
| G6 | Terminal execution states are immutable; transitions are conditional | BE |
| G7 | Contract fields are implemented end-to-end or removed | BE |
| G8 | All `resources/schemas` files are loaded and used | BE |
| G9 | All code-read config keys exist in `application.yml` | BE |
| G10 | `META-INF/services` entries reference existing implementations | Core |
| G11 | Core contains no provider/Jira/Xray/LLM concepts; generation is deterministic | Core |
| G12 | FE features respect ESLint boundaries; pages use VMs/query hooks | FE |
| G13 | Playwright mocks reject unmocked `/api`; overlay E2E runs in CI | FE |
| G14 | STOMP connections and subscriptions are authenticated/authorized | BE |
| G15 | Seed credentials require an opt-in Flyway placeholder | BE |
| G16 | Status claims require commit reference and green gates | Process |
| G17 | Repositories are self-contained | All |
| G18 | Step input cannot access env/files/JVM properties; managed runs validate destinations | Core |
| G19 | Runtime state belongs to execution/scenario, not JVM | Core |
| G20 | Execution is fail-closed; unavailable steps never silently no-op | Core |
| G21 | Security controls have attack-path integration tests | BE |
| G22 | IDs use project-scoped resolution; links use `safeHref`; SPA has CSP | BE + FE |
| G23 | One redactor for outbound/persisted text; INFO production logs; no secret fallbacks | Core + BE |
| G24 | Required gates can fail and run on every PR | All |
| G25 | No emojis; one implementation per concern; size limits enforced | All |
| G26 | Connector adapters pass the common contract suite against simulators | BE |
| G27 | Provider names remain inside connector packages | BE |
| G28 | AI providers callable only from `intelligence.infrastructure.inference` through registered capabilities | BE |
| G29 | AI capabilities declare schema, redaction, and deterministic fallback; schemas are loaded | BE |
| G30 | LLM output cannot mutate domain state without approved apply rules; `baseVersion` rechecked | BE |
| G31 | Every AI capability has a failing-capable golden evaluation in CI | BE |
| G32 | AI calls are rate-limited, budgeted, audited by hash, and correlated | BE |
| G33 | Prompts are versioned English files, contain no secrets/emojis, and record their hash | BE |
| G34 | Cucumber step patterns/IDs remain frozen against baseline | Core |
| G35 | Changed contracts have complete BE → Core → FE → Zod → mobile traceability | All |
| G36 | `CLAUDE.md`/`ORCHESTRATION.md` cannot be edited without explicit approval | All |


## AI design
When writing code to make calls to LLMs, use your Cerebras skill to use LiteLLM via OpenRouter to the ` OpenRouter/nvidia/nemotron-3-ultra-550b-a55b:free` model with Cerebras as the inference provider. You should use Structured Outputs so that you can interpret the results and populate fields in the legal document.

There is an OPENROUTER_API_KEY in the .env file in the project root.

## Technical design
The entire project should be packaged into a Docker container.
The backend should be in qa-platformBE and be a uv project, using FastAPI.
The frontend should be in qa-platformFE
The database should use Postgres

There should be scripts in scripts/ for:

# Mac
scripts/start-mac.sh    # Start
scripts/stop-mac.sh     # Stop

Backend available at http://localhost:8000


## Git flow rules and Classification Criteria 

This epigraph defines how Claude Code must handle GitHub work depending on whether the task is a new feature or a simple task/change.

***
Branch strategy (always applies for local and remote branches)

A feature/task branch is never merged directly into master.
The flow is always: feature/<branch> → develop → master.
develop is the integration branch: all feature branches are created from develop and merged back into develop.
**A task ends at develop.** Promotion of develop to master is a separate step that runs only when the
user asks for it literally. It is never part of a feature flow, never implied by a green gate, and never
triggered because develop looks stable.

Each session works in its own worktree on its own branch (see "Shared workspace rules" above), so
several sessions can run this flow at the same time without touching each other's state.

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
git worktree list                      # never create a second worktree for the same branch
git worktree add ../<repo>.worktrees/142-user-authentication -b feature/142-user-authentication origin/develop
cd ../<repo>.worktrees/142-user-authentication

Every later step in this flow runs inside that worktree. The main checkout is left on develop, untouched.

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
`core-publisher` is called when `qa-platformCore` changed in this task, or to repair an empty or stale
`lib/core-m2`. A task that did not touch Core has nothing to publish and does not call it at all.
When it is called, it runs **before** the merge: its purpose is to prove BE compiles against the Core
artifacts on disk before anything reaches develop.
Only one publish runs at a time across all sessions, because `~/.m2` and `lib/core-m2` live outside every
worktree: it takes the `.core-publish.lock` lock at workspace root.
**Several sessions working on Core publish once, not once each.** The session that finishes last takes the
batch: gitFlow merges every finished Core branch into local `develop`, core-publisher publishes that merged
state once, the BE gate runs against it, and only then is `develop` pushed. A branch joins the batch only if
it is pushed, its gate passed, its session reported finished, and no live worktree still holds it; anything
else is left out and named in the report. A session that finds another still working on Core releases the
lock and lets the last one publish - it never waits holding the lock.

6. Merge the feature branch into develop (gitFlow does this, not you by hand)
Re-fetch first: another session may have merged into develop while this task ran.
git fetch origin
If develop moved, merge origin/develop into the feature branch inside the worktree and re-run the full
gate before continuing. A gate that was green against an older develop proves nothing.
Then merge the feature branch into develop in the main checkout and push:
git -C <main-checkout> checkout develop
git -C <main-checkout> pull origin develop
git -C <main-checkout> merge --no-ff feature/142-user-authentication
git -C <main-checkout> push origin develop

7. Delete the worktree and the branch after the merge is confirmed on the remote
git fetch origin && git log origin/develop --oneline -1      # confirm the merge landed
git -C <main-checkout> worktree remove ../<repo>.worktrees/142-user-authentication
git -C <main-checkout> branch -d feature/142-user-authentication
Never `rm -rf` a worktree directory: that leaves stale metadata in `.git/worktrees/`. Never delete
either one before the push is confirmed.
For a batch merge, each auxiliary branch is deleted only after proving `develop` contains all of its
commits - `git log --oneline <branch> --not develop` must print nothing - and `git branch -d` (never `-D`)
is the second safety net. Remove only your own worktrees: another session's worktree is its own to clean up.

8. Stop here
Do not promote develop to master. That is its own task, on an explicit request from the user, and it
runs the mandatory `security-guardian` gate first.