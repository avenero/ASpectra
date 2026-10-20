# Platform overview

Status: Intended (moved from `.claude/CLAUDE.md` on 2026-10-20; content not re-verified against the code) | Owner: TBD

Shared product and architecture facts for every repository and every agent. Product scope is defined in
[../alcance.md](../alcance.md). Engineering rules are in
[../development/engineering-rules.md](../development/engineering-rules.md).

## The product in one sentence

**A·Spectra is the automation framework a team does not have to build.** A QA whose test case is a title
and an ID writes the step-by-step in Gherkin against a ready-made step catalog for API, Web, Mobile and
Database, runs it, and gets a result with evidence, reported back against the ID. Product scope is defined
in [../alcance.md](../alcance.md), the only source of truth for it; requirements derived from it are in
[../requirements.md](../requirements.md).

- **One runtime, several entry doors** (alcance §1.2): a test-management connector, a `.feature` upload, a
  Postman/Swagger/user-story import, and native authoring. Every door converges on one stored scenario model.
- **Review is a convention of the console flow, not an invariant of the backend** (alcance §1.3,
  provisional until machine access ships at MVP-1.0.1). Say that the console asks for a review before a run;
  never say that a scenario cannot run without one. The one hard control is the protected environment.
- **The test management system is optional** (alcance §1.4). It is the source of truth only for cases that
  originated there; A·Spectra never writes steps, Gherkin, new cases or defects into it, only results and
  evidence. A missing integration never blocks a run.
- **AI is an accelerator the customer may switch on, never the product** (alcance §8). The AI rules in
  [../development/engineering-rules.md](../development/engineering-rules.md) hold in full.

**Scope as configuration** (alcance §13.1). The values that make the scope true in a running system:

| Scope item | Configuration (`qa-platformBE` `application.yml`) | Default |
|---|---|---|
| Offered test management systems | `integration.offered-providers` (`INTEGRATION_OFFERED_PROVIDERS`) | `JIRA_XRAY,JIRA_XRAY_CLOUD` |
| Offered database engines | `integration.offered-database-engines` (`INTEGRATION_OFFERED_DATABASE_ENGINES`) | `postgresql,sqlserver,oracle` |
| AI | `ai.enabled` (`AI_ENABLED`) | `false` |

Integration scope (alcance §4.4, §7): Jira + Xray Server/Data Center and Cloud are built and offered;
TestRail is built and in scope, hidden behind the gates above; Jira without Xray is in scope and not built;
Azure DevOps is never claimed. Machine access (public API, CLI, service accounts) is deferred to MVP-1.0.1.

Execution coverage: HTTP/API, Web, Mobile, Database.

## Repository layout

The workspace root is the ASpectra governance repository: documentation, rules and agent configuration, never
built or deployed ([ADR 0001](decisions/0001-governance-repo-without-submodules.md)). `qa-platformCore`,
`qa-platformBE` and `qa-platformFE` are each their own Git repository, nested in the root and ignored by it.


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
  read that folder — **skipping the sync silently runs stale Core.** `syncCoreFromMavenLocal` deletes
  `lib/core-m2/com` before it copies, so never run it until the artifacts exist in `~/.m2` at
  `coreVersion`. In the agent pipeline this is `core-publisher`'s job.
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

