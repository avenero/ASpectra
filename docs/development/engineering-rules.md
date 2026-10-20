# Engineering rules

Status: Intended (moved from `.claude/CLAUDE.md` and `.claude/ORCHESTRATION.md` on 2026-10-20) | Owner: TBD

Mandatory rules for every human and agent working in any A·Spectra repository. Cross-repository rules
R1-R16 are in [../architecture/repository-rules.md](../architecture/repository-rules.md); each
repository's `AGENTS.md` adds its own.

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

2. **Never edit the instruction and rule files without explicit approval.**  
   This covers every `CLAUDE.md`, `.claude/ORCHESTRATION.md`, every `AGENTS.md` and the canonical rule
   documents (`docs/architecture/platform.md`, `docs/architecture/repository-rules.md`,
   `docs/development/engineering-rules.md`, `docs/development/git-workflow.md`,
   `docs/development/agent-pipeline.md`). Report proposed changes, missing rules, or violations in chat.

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
    Update affected documentation when contracts change. Remove completed tasks and retain only unfinished work.

19. **Migrations are forward only.**  
    Never edit an applied migration. A schema change is a new forward `V*` migration with its `U*` undo.

20. **Keep issues current.**  
    After finishing a task, update or close its GitHub issue to reflect current status. A status claim needs
    a commit reference and green gates (G16).

## Gates (green means all of it)

GitHub Actions is enabled (confirmed by the owner on 2026-10-20) and runs each repository's
`ci.yml` on pushes and pull requests to `develop`, `master` and `release/**`. Run the same gate locally
before pushing; a red CI run is never fixed by skipping a gate.

| Repo | Gate |
|---|---|
| Core | `./gradlew build -Pstrict` with Docker running; `scripts/check-*.sh` (five guard scripts); `scripts/cli-smoke.sh` |
| BE | `env -u SPRING_PROFILES_ACTIVE ./gradlew clean check -Pstrict` |
| FE | `scripts/check-ci-gates.sh`, `scripts/check-self-contained.sh`, `npm run type-check`, `npm run lint`, `npm run test:coverage`, `npm run build`, `npm run test:e2e` |

- Testcontainers on this machine (Rancher Desktop) needs `export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock`.
- The BE gate is not concurrency-safe: never run a targeted Gradle test in the same checkout while `check` runs.
- If FE type-check fails locally but not in CI, run `npm ci` first (stale `node_modules`).

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

