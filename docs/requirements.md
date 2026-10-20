# A·Spectra — Requirements by feature, module and screen

Status: Intended (derived from [alcance.md](alcance.md) as of its 2026-10-19 revision) | Owner: avenero

This document restates the scope decided in [alcance.md](alcance.md) as numbered, traceable
requirements, grouped three ways: by product feature (Part 1), by functional module (Part 2) and by
screen (Part 3). **`alcance.md` is the only source of truth.** Every requirement cites the section it
comes from; where this document and `alcance.md` disagree, `alcance.md` wins and this document is the
defect. Nothing here adds scope, and nothing here is a work order by itself: each requirement reaches
implementation through a GitHub issue with acceptance criteria (Part 6).

The previous version of this file, the 2026-10-01 FE screen baseline, is superseded. It is kept in Git
history (ASpectra commit `fcd912c`) as the historical reference that `alcance.md` §13 describes.

## How to read this document

| Column | Meaning |
|---|---|
| ID | `<MODULE>-<nn>`. IDs are stable: a withdrawn requirement keeps its ID and is marked OUT, never reused. |
| Requirement | What the product must do or must not do, in product language. |
| Scope | `IN` in general scope · `OUT` permanently or until reopened, with the reason in `alcance.md` · `LATER` deferred to the named release · `OPEN` waits on an owner decision. |
| Today | State on 2026-10-19 as `alcance.md` records it: `Built`, `Partial`, `Gated` (built, not offered), `Not built`, `Defect` (built in a way that contradicts the decision), `—` (not stated). |
| Source | Section of `alcance.md`. A dagger (†) marks §4.12 onward, defined by delegation and **awaiting the owner's confirmation**: not a work order until confirmed. |

`IN` does not mean "build now". The MVP 1.0 cut is a separate exercise (`alcance.md` header). Releases
named here: **1.0.0** (the third project grade with protected environments) and **MVP-1.0.1** (machine
access).

## The product in one sentence

**A·Spectra is the automation framework a team does not have to build** (§1.1). A QA whose test case is a
title and an ID writes the step-by-step in Gherkin against a ready-made catalog for API, Web, Mobile and
Database, runs it, and gets a result with evidence, reported back against the ID. It is one execution
runtime with one stored scenario model, reached through several entry doors: a test-management connector,
a `.feature` upload, a Postman/Swagger/user-story import, and native authoring (§1.2). Review is a
convention of the console flow, not an enforced property of the backend (§1.3, provisional until
MVP-1.0.1).

---

# Part 1 — Product features

The capabilities the job requires, tiered by what the job needs (§1.5). Each feature lists the module
requirements that deliver it.

## Tier 1 — Constitutive

| ID | Feature | Delivered by |
|---|---|---|
| F-01 | A place for the ID and title to enter: from the connector, typed by hand, or a bulk list | IMA-01, IMA-03, AUT-01 |
| F-02 | Authoring Gherkin against a discoverable, searchable step catalog | AUT-01, CAT-01 |
| F-03 | Validation while authoring: "this step does not exist" before run time | REV-05, IMC-03 |
| F-04 | Environments and credentials held outside the scenario text, so one scenario runs against several environments | ENV-01..ENV-04, REV-06 |
| F-05 | Execute and get pass/fail | EXE-01 |
| F-06 | A per-step result and the reason for a failure | EVD-03 |
| F-07 | The scenario is stored; it is the customer's asset | AUT-01, ORG-06 |

## Tier 2 — Viability

| ID | Feature | Delivered by |
|---|---|---|
| F-08 | A work queue: which cases still have no steps | IMC-03, DSH-01 |
| F-09 | Reuse: macros, shared step sequences, `Background` | MAC-01..MAC-04 |
| F-10 | Data-driven scenarios with `Examples` tables | AUT-01, SYN-01 |
| F-11 | Run one scenario while authoring it, fast (API/Database under 5 s, Web/Mobile under 20 s to the first step result) | EXE-11, NFR-01 |
| F-12 | Evidence at the failure: screenshot, HTTP request and response, SQL, mobile screen, logs | EVD-03 |
| F-13 | Parallel execution | EXE-06 |
| F-14 | Help obtaining selectors: the element inspector | WEB-06 |
| F-15 | Context attached to the case: linked story, bug, acceptance criteria, notes, attachments | IMA-01 |
| F-16 | Suites and grouping; run a selection | EXE-01 |
| F-17 | History per scenario: did it ever pass, and when did it start failing | EVD-05, DSH-01 |

## Tier 3 — Platform

| ID | Feature | Delivered by |
|---|---|---|
| F-18 | Failure triage: group failures by cause | DSH-01, AI-04 |
| F-19 | Flakiness detection | EXE-09, DSH-01 |
| F-20 | Impact analysis: a catalog step changes, which scenarios break | CAT-06, MAC-02 |
| F-21 | Bulk edit across scenarios | — (no module decision yet in `alcance.md`) |
| F-22 | Versioning, who changed what, and revert | EXE-10, AUD-02 |
| F-23 | An owner per scenario | — (no module decision yet in `alcance.md`) |
| F-24 | Scheduled or unattended runs | EXE-04 |
| F-25 | Notification on failure | NTF-01, NTF-02 |

## Tier 4 — Governance

| ID | Feature | Delivered by |
|---|---|---|
| F-26 | Roles: who authors, who runs against production | ORG-02, ORG-03 |
| F-27 | An audit trail | AUD-01..AUD-05 |
| F-28 | A gate on running against production | ENV-05 |

## Cross-cutting product principles

| ID | Principle | Source |
|---|---|---|
| P-01 | If there is no step, the capability is not claimed. The catalog is the capability statement. | §4.10, §6.1 |
| P-02 | A·Spectra never writes test-case content (steps, Gherkin, new cases, defects) into the customer's test management system. Results and evidence only. | §1.4, §4.14 |
| P-03 | A·Spectra is not a test management system, not a CI tool, and AI never decides pass/fail. | §1.4 |
| P-04 | Using A·Spectra with no test management system connected is fully supported. A missing integration never blocks a run. | §1.4 |
| P-05 | Built but not offered is never a silent state: a capability is offered, removed, or visibly gated with a reason. | §11 |
| P-06 | A scenario is strictly linear (no conditional steps) and does not span layers. | §6.1 |
| P-07 | The Gherkin text is portable; the step implementations and the engine are not distributed. | §1.1 |
| P-08 | The canonical product name and spelling is `A·Spectra`. | §1.4 |

---

# Part 2 — Requirements by module

## 2.1 Users, roles and tenancy (ORG)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| ORG-01 | Two kinds of people: account holders (QA author, QA lead, platform administrator) and stakeholders without an account, who receive results in their test management system. No separate auditor persona. | IN | — | §2 |
| ORG-02 | Platform `ADMIN` is a vendor support role, renamed so it is never mistaken for a customer administrator. No customer sees another customer's data. | IN | Defect: resolves `READ_WRITE` on every project; `GET /api/users` and `/api/users/organizations` span every customer | §2 |
| ORG-03 | A customer organisation administrator exists as a distinct role, scoped to one organisation, with no authority outside it. | IN | Not built | §2 |
| ORG-04 | Project grades are `READ`, `READ_WRITE` and, from release 1.0.0, a higher grade that guards launching against a protected environment and configuring a connector. Anything built against two grades must survive a third. `QA_LEAD` becomes real with that grade or is deleted. | IN (1.0.0) | Partial: two grades, Postgres enum | §1.3, §2 |
| ORG-05 | Write implies execute: there is no separate execute permission. | IN | Built | §2 |
| ORG-06 | Projects and scenarios are soft-deleted with a recovery window; executions and evidence may be hard-deleted under retention. A customer organisation administrator may delete a project; an author may delete their own scenarios. | IN | — | §2 |
| ORG-07 | Deleting a user never dissolves a project's tenant boundary and never erases who ran what. | IN | Defect: `owner_id ON DELETE SET NULL` | §2, §4.13 |
| ORG-08 | An organisation may require two-factor authentication of all members; it is mandatory for anyone who can configure a connector or change AI data egress; an administrator can see who has enrolled. | IN | Not built: self-service opt-in only | §2 |
| ORG-09 | Single sign-on is a paid enterprise capability. | IN | Partial: generic OIDC built; handoff held in single-instance memory | §2, §4.1 |
| ORG-10 | Impersonation ("view as") is not offered, because it would make the audit trail lie about who acted. | OUT | — | §2, §4.17 |
| ORG-11 | Permissions are granted per user. Group or team assignment is deferred. | LATER (MVP-1.0.1) | Built (per user) | §2 |
| ORG-12 | AI data egress is an organisation setting: off by default, changed only by the customer organisation administrator, audited. A project may opt into a capability the organisation allows, never one it forbids. | IN | Defect: held per project | §2, §8 |
| ORG-13 | Deployment model: self-hosted single-tenant first, vendor-hosted SaaS second, shipped as one artifact with configuration differences. A dedicated managed deployment is the self-hosted artifact sold as managed. | IN | Partial: self-hosted shippable | §3 |
| ORG-14 | Hosted isolation is the organisation, row-scoped in one shared database: an organisation column on project-owning data and tenant-scoped aggregate queries. It lands before or with the audit table (AUD-06). | IN | Not built: `projects` has no organisation column | §3, §12.6 |
| ORG-15 | The organisation is a first-class managed object: created, renamed, lists its members and projects, soft-deleted with a recovery window. Suspension is commercial (§10). | IN | Partial: one field (`name`), no update or delete | §3 |
| ORG-16 | A user belongs to exactly one organisation. Multi-organisation membership is not offered. | IN (OUT for multi-org) | Built | §3 |
| ORG-17 | On self-hosted the customer holds the keys and the vendor cannot read their data; support works from a diagnostics bundle the customer generates, redacted by the single redactor (G23). On hosted, the vendor support role is the access path. | IN | Not built (bundle) | §3 |
| ORG-18 | Running inside the customer's own network is supported through a customer-side runner that generalises the mobile-agent pattern (`/v1/runs`, token authentication). | IN | Partial: mobile agent only | §1.5, §3 |
| ORG-19 | Data residency is one region, the vendor's, disclosed. Customer-chosen regions are not offered. Which region: owner decision. | IN / OPEN | Not built | §3, §9.4 |
| ORG-20 | An execution's evidence (screenshots, transcripts, logs) can be downloaded as one archive. Bulk export of all historical evidence is not offered. | IN | Not built | §3 |

## 2.2 Identity and access (IAM)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| IAM-01 | Local password sign-in, TOTP two-factor with recovery codes, forced password rotation, password reset, per-account lockout, token rotation and generic OIDC federation (Google wired; any OIDC provider by configuration) stay built and owned. | IN | Built | §4.1 |
| IAM-02 | SAML is part of the paid single-sign-on tier; build or integrate is decided in §14.1. | IN | Not built | §4.1 |
| IAM-03 | LDAP and Active Directory binding are not offered. | OUT | — | §4.1 |
| IAM-04 | Account creation: hosted is self-service sign-up with administrative activation; self-hosted is administrative creation only. Sign-up records the version of the terms accepted. | IN | Built (sign-up, SSO and email off by default) | §4.1 |
| IAM-05 | WebAuthn and passkeys (hardware keys included) are second factors beside TOTP; the method picker, WebAuthn prompt and remove-one-factor confirmation follow the existing Figma frames. | IN | Not built (designed) | §4.1, §5.1 |
| IAM-06 | SMS as a second factor is not offered. | OUT | — | §4.1 |
| IAM-07 | Trusted devices ("remember this device") are not offered; the frame is removed from the design. | OUT | Designed, unbuilt | §4.1, §5.1 |
| IAM-08 | Session idle timeout and absolute lifetime are configurable, with defaults of 30 minutes idle and 12 hours absolute. | IN | — | §4.1 |
| IAM-09 | A forced global logout exists. | IN | — | §4.1 |
| IAM-10 | A concurrent-session limit is not imposed. | OUT | — | §4.1 |
| IAM-11 | API tokens, service accounts and CI credentials follow machine access (MCH). | LATER (MVP-1.0.1) | Not built | §4.1, §4.18 |
| IAM-12 | The OIDC sign-in handoff, the two-factor login challenge and the two-factor enrolment state are durable before a second backend instance runs. | IN | Defect: single-instance memory | §3, §12 |

## 2.3 Projects (PRJ)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| PRJ-01 | A project is one application under test and owns its connector, environments, members, suites and executions. | IN | Built | §4.2 |
| PRJ-02 | A project's name and description are editable after creation; its key is immutable. | IN | Not built | §4.2, §5.2 |
| PRJ-03 | Ordinary deletion is soft with a recovery window; an administrative hard purge exists for legally required erasure, and the audit record of the purge survives it. | IN | Partial: purge only as an operator script | §4.2, §4.17 |

## 2.4 Environments, secrets and fixtures (ENV)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| ENV-01 | Credentials are encrypted at rest with the owning row's identity bound into the ciphertext, are write-only, and use two independent keys without which the application refuses to start. | IN | Built | §4.3 |
| ENV-02 | The customer's own secret manager is a supported source (Vault and AWS Secrets Manager named), through the `SecretProviderKind` port (`LocalEncrypted` / `ExternalRef`). | IN | Not built (port designed) | §4.3 |
| ENV-03 | Credentials and signing keys can be rotated. | IN | Not built | §4.3, §9.3 |
| ENV-04 | Mutual TLS to the system under test is configured per environment, beside its base URL and SSL configuration. | IN | — | §4.3 |
| ENV-05 | An environment can be marked protected, per project; launching against it (production and pre-production only) requires the higher project grade. The control is available, not required. An organisation policy is deferred and, when it exists, wins. The mechanism (`type` vs `kind`, mobile environments) is designed before it is built. | IN (1.0.0) | Not built | §1.3, §4.3 |
| ENV-06 | A credential never appears in a report, log, evidence artifact or sync payload. This is not published to customers until step text and step warnings are redacted too. | IN | Partial: payloads, headers and errors redacted; step text and warnings stored verbatim | §4.3 |
| ENV-07 | "Test connection" promises reachability and authentication only, and the console says so in those words. | IN | — | §4.3, §7 |
| ENV-08 | Per-project fixture store: a scenario with an upload step offers an upload control, the file is stored against the project on the evidence storage, and the step holds a reference resolved to a real path at run time. The Gherkin pattern does not change (G34); the file type is traced end to end (G35). | IN | Not built | §4.3 |
| ENV-09 | Test-data management (seeding, data pools, cleanup) is not a product feature. Scenarios may still write to the system under test through their own steps. | OUT | — | §4.3 |

## 2.5 Integrations and connectors (INT)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| INT-01 | Jira + Xray Server/Data Center: import, and write-back of results and evidence. Priority 1. | IN | Built, offered | §4.4, §7† |
| INT-02 | Jira + Xray Cloud: import and write-back. Priority 2. | IN | Built, offered | §4.4, §7† |
| INT-03 | TestRail: import and write-back. Priority 3. Offering it opens three gates: `offered-providers`, the console's hard-coded list, and the landing copy. When is release planning. | IN | Gated: built and contract-tested, hidden | §4.4, §7† |
| INT-04 | Jira without Xray: a distinct adapter; write-back as a comment and attachments on the issue. Priority 4. | IN | Not built | §4.4, §7† |
| INT-05 | The `AZURE_DEVOPS` value is removed as dead scope; Azure DevOps is never claimed. | IN (removal) | Defect: enum value exists | §4.4 |
| INT-06 | One connector per project. A project drawing from several sources is deferred. | IN / LATER | Built | §4.4 |
| INT-07 | Customer- or partner-written connectors are not offered. | OUT | — | §4.4 |
| INT-08 | A connector is offered when it passes the shared contract suite against its provider's simulator (G26). | IN | Built | §4.4, §7† |
| INT-09 | Each connector accepts the credentials its provider issues: personal access token or password (Jira Server/DC), API token (Jira Cloud), client id and secret (Xray Cloud), API key (TestRail), all in the encrypted store. | IN | Built | §7† |
| INT-10 | The field mapping is the connector's contract. Customer-defined mappings are not offered; an instance-specific field that plays a fixed canonical role is confirmed once per integration by a person. | IN | — | §7† |
| INT-11 | A source field the canonical model has no home for is dropped, and the import proceeds. | IN | — | §7† |
| INT-12 | Credential health is checked on a schedule and a rejected credential is notified before an import fails. | IN | Not built | §7† |
| INT-13 | Outbound rate limits mirror each provider: 300/min Jira + Xray, 60/min Xray Cloud, 180/min TestRail, enforced by the shared database-backed limiter. No per-customer override on hosted; self-hosted sets its own. | IN | Built (limiter) | §7† |
| INT-14 | The customer can see the exact payload written to their system. Replay is only the retry of a failed sync job. | IN | Partial: payload stored with the sync job | §7† |
| INT-15 | Which connectors are offered is an organisation setting (SET-01), so TestRail can be offered to one customer and not another. | IN | Not built | §4.4, §4.17† |
| INT-16 | Distribution as a Jira Marketplace app. | OPEN | — | §7† |

## 2.6 Import from a test management system (IMA)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| IMA-01 | Import a test plan, a test set, a test execution or a single case by key. A typed ID is a single case by key; a bulk list is the same import repeated. The case arrives with its context. | IN | — | §1.5, §4.5 |
| IMA-02 | Importing a folder, or requirements and user stories as coverage objects, is not offered. | OUT | — | §4.5 |
| IMA-03 | Re-import performs a three-way merge that preserves reviewer edits, and is idempotent by content hash. | IN | Built | §4.5 |
| IMA-04 | When a source case changes after approval, the canonical case returns to review, the approved projection stays runnable, and the drift is reported in the console (never blocking the run). | IN | Partial: drift indicator not built | §1.3, §4.5 |
| IMA-05 | A case deleted at the source is marked orphaned and stops syncing; its scenario, history and evidence are kept; it is refused execution on the write-back path until a container lists it again. | IN | Built (refusal) | §4.5 |
| IMA-06 | Scheduled unattended re-import is allowed only if it reports which cases re-queued, which edits were preserved and which drifted. | IN | Gated: built, switched off | §4.5 |
| IMA-07 | One import job accepts up to 500 cases; a larger selection is split into several jobs. | IN | — | §4.5, §9.1† |
| IMA-08 | Import runs as a durable asynchronous job (202 plus a job row). | IN | Built | §15 |

## 2.7 Import from specifications and user stories (IMB)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| IMB-01 | This pipeline terminates in the canonical model; the separate `NEW`/`ACCEPTED`/`REJECTED` lifecycle goes away. | IN | Defect: separate bounded context writes executable rows | §1.2, §4.6 |
| IMB-02 | Accepting a draft promotes it to a canonical scenario that is then editable and runnable directly, with no second approval. | IN | Not built | §4.6 |
| IMB-03 | The AI-generated path is shown only where AI is enabled. | IN | Defect: produces empty drafts when AI is off | §4.6 |
| IMB-04 | Deterministic extraction for Swagger/OpenAPI and Postman: a request step with its documented status-code and schema assertions, working with AI off. User stories have no deterministic mode. | IN | Not built | §4.6 |
| IMB-05 | Extracted cases are runnable and labelled as derived from a specification; business expectations not in the specification stay empty for a person to write. | IN | Not built | §4.6, §8† |
| IMB-06 | The draft confidence score is not shown to users. | OUT | — | §4.6 |
| IMB-07 | Drafts are never pushed back into the customer's system as new cases. | IN (boundary) | — | §1.4, §4.6 |
| IMB-08 | Supported inputs: Swagger/OpenAPI 2.x and 3.x in JSON or YAML, Postman v2.1, each up to 5 MB. | IN | Built | §4.6 |
| IMB-09 | A specification enters only as a file the user uploads; reading one from a test-management attachment is not offered. | IN / OUT | — | §1.1 |

## 2.8 Import of `.feature` files (IMC)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| IMC-01 | A `.feature` file enters the canonical model with the seven-state review, versioning and re-import idempotency; its text, structure, tags and examples survive. | IN | Built | §4.7 |
| IMC-02 | Migrating an existing Cucumber suite is a promised use case, stated together with its limit: the customer's step definitions do not come along, and steps are re-expressed against the catalog. | IN | — | §4.7 |
| IMC-03 | Steps that do not match the catalog are imported and marked; the case cannot become `EXECUTABLE` until they resolve; the marked steps form the work queue. | IN | — | §4.7 |
| IMC-04 | A `.feature` case never writes results back, and the customer is told so. | IN | Built | §4.7, §4.14† |

## 2.9 Native authoring and Gherkin paste (AUT)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| AUT-01 | Native authoring is a first-class door: a scenario is written in the console against the catalog, with `Background` and `Examples`, and stored. | IN | Built | §1.2, §4.8 |
| AUT-02 | A natively authored or pasted scenario runs without review. | IN | Built | §1.3, §4.8 |
| AUT-03 | Gherkin paste goes through the same stored model as the scenario editor. | IN | Defect: different repository, bypasses the editor's guard | §1.2, §4.8 |
| AUT-04 | Natively authored cases are never pushed into the customer's test management system. | IN (boundary) | — | §1.4, §4.8 |
| AUT-05 | A scenario exports as Markdown documentation (English, Spanish, French) and as one standard `.feature` file. No per-tool exporters. | IN | Partial: Markdown built; `.feature` export not built | §1.1 |

## 2.10 Canonical review (REV)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| REV-01 | Seven states: `IMPORTED`, `NORMALIZED`, `DRAFTED`, `REVIEWED`, `APPROVED`, `EXECUTABLE`, `REJECTED`, with the meanings defined in §4.9. | IN | Built | §4.9 |
| REV-02 | `REVIEWED` (correctness) and `APPROVED` (readiness) stay distinct states. | IN | Built | §4.9 |
| REV-03 | Anyone with write access moves a case between states; there is no approver grade. | IN | Built | §2, §4.9 |
| REV-04 | Each transition is typed as human or system and audited with actor, reason and states under optimistic locking, and is also written to the audit trail, which survives the case's deletion. | IN | Partial: review trail written, never read, deleted with its case | §4.9, §4.17† |
| REV-05 | Executability rule, verbatim: a non-blank title; layer and BDD mode set; at least one step; every step resolves in the live catalog, is available, belongs to the case's layer and carries exactly its pattern's parameters; at least one THEN step; no blocking flag; no open ambiguity question. A step's expected text is optional. | IN | Built | §4.9 |
| REV-06 | Approval and environment availability are never part of executability. | IN | Built | §4.9 |
| REV-07 | Editing an approved case's steps returns it to `DRAFTED`; editing its title or description does not; the existing projection stays launchable. | IN | — | §4.9 |
| REV-08 | Bulk approval is available, and the audit records each case individually with actor, timestamp and version. | IN | — | §4.9 |
| REV-09 | The reviewer, timestamp and exact version approved are visible later, per case, through the audit trail. | IN | Not built | §4.9 |

## 2.11 Step catalog (CAT)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| CAT-01 | The catalog is the published, versioned capability statement: 515 steps (HTTP 141, Web 186, Mobile 136, Database 52) in 62 components, searchable, with display text in English, Spanish and French. | IN | Built | §4.10 |
| CAT-02 | A hidden step is never silently absent: it does not appear at all, or it appears with the reason it is unavailable. | IN | Defect: 14 hidden without a reason | §4.10 |
| CAT-03 | The six file-dependent steps are shown once the fixture store (ENV-08) exists. | IN | Not built | §4.10, §12.12† |
| CAT-04 | The eight conditional "if present" steps are excluded permanently as a determinism policy. | IN | Built | §4.10 |
| CAT-05 | Customers cannot add primitive steps; macros are the extension point. A step request is acknowledged, triaged and answered with a macro or a target release, with no promised turnaround. | IN | — | §1.5, §4.10 |
| CAT-06 | Deprecation is visible: a deprecated step names its replacement, keeps running, and the console says so. The notice period is an owner decision. | IN / OPEN | Partial: contract carries the flag and replacement id | §4.10 |
| CAT-07 | The engine's GraphQL and SOAP capability advertisement, and the test that locks it in, are removed. | IN | Defect | §4.10 |
| CAT-08 | Step patterns and ids are frozen (G34) and never translated. | IN | Built | §4.10, §12.11† |

## 2.12 Macros (MAC)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| MAC-01 | Macros are expanded before the test is generated, so the engine sees only catalog steps and Core stays free of the concept. | IN | Built | §4.11 |
| MAC-02 | Macros are versioned: a scenario references a version, an edit creates a new one, and a scenario moves to it deliberately. If versioning is staged, editing a body at least reports which scenarios it affects. | IN | Defect: a body edit silently re-expands everywhere | §4.11 |
| MAC-03 | Macros are shareable at organisation level, only after they are versioned. | IN | Not built | §4.11 |
| MAC-04 | When a user cannot find a step, the platform proposes a composed step (a macro of existing steps); its origin (by hand or accepted suggestion) is recorded, and nothing is applied without a person. | IN | Built (custom-step mechanism) | §1.5, §8† |

## 2.13 Execution (EXE)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| EXE-01 | Three launch paths (direct, execution plan, bulk execute of a suite or feature) converge on one launch service; the review gate stays only on the plan path. | IN | Built | §1.3, §4.12† |
| EXE-02 | Execution moves out of process onto the customer-side runner; in-process remains for tests, CI and a single host. | IN | Not built: runs in the backend JVM | §3, §4.12† |
| EXE-03 | Bulk execute carries the same runtime configuration as the other paths (layer, Web configuration and browser, reporting, HTTP engine). | IN | Defect | §4.12† |
| EXE-04 | A schedule is created in the console by a person who chooses what it runs; each scheduled run records that it was scheduled and who created the schedule; a schedule against a protected environment needs the higher grade at creation and at every firing. | IN | Not built: flag hardcoded false | §4.12† |
| EXE-05 | A trigger from an external system (public API, webhook, CLI). | LATER (MVP-1.0.1) | Not built | §1.4, §4.12† |
| EXE-06 | The scenarios of one run execute in parallel, Mobile bounded by its devices. The capacity number is a deployment setting, not published. | IN | Not built (parallel runs and parallel HTTP requests exist) | §4.12† |
| EXE-07 | A run's maximum duration is two hours, configurable; exceeding it ends the run as `ERROR`, never as a failure. | IN | Built | §4.12† |
| EXE-08 | Cancel always brings the run to a terminal state; remaining steps report as skipped; a thread that ignores the cancel is released after ten minutes and its late results are discarded. | IN | Built | §4.12† |
| EXE-09 | Retry inside a run is not offered. A rerun is a new execution and both are kept. Flakiness is detected and labelled; quarantine is manual and visible; automatic quarantine is not offered. | IN | Partial: flakiness index on the project dashboard | §4.12† |
| EXE-10 | The case version each scenario ran is recorded on the execution, so what ran can be re-launched; a byte-identical rerun is not promised. | IN | Not built | §4.12† |
| EXE-11 | A run launched from the editor while authoring is a draft run (never writes back); a run launched from the plan, a suite or a feature is a recorded run. | IN | Partial | §4.14† |

## 2.14 Evidence and reporting (EVD)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| EVD-01 | Evidence is tamper-evident: every artifact's SHA-256 is verified on read, the HTML report is hashed and versioned instead of overwritten, and each execution carries a manifest of its artifacts' hashes. | IN | Partial: hash stored, never checked | §4.13† |
| EVD-02 | The HTML report includes the evidence it summarises (screenshots, stack traces). | IN | Defect: no screenshots, empty trace slot | §4.13† |
| EVD-03 | Captured per step: screenshot, HTTP request and response, SQL, mobile screen, logs, plus a DOM snapshot on a Web failure. Video, HAR, traces and device logs are not captured. | IN | Partial: screenshots complete; HTTP as 500-character summaries; SQL and logs not captured | §1.5, §4.13† |
| EVD-04 | Evidence retention defaults to 30 days, set per organisation within a vendor range by the organisation administrator and audited; it covers a run's detail (step text, request and response summaries, errors, warnings) as well as files. The vendor range is an owner decision. | IN / OPEN | Defect: run detail never purged | §4.13† |
| EVD-05 | The verdict history of each scenario (what ran, when, outcome) outlives evidence retention. | IN | — | §4.13† |
| EVD-06 | Evidence survives the ordinary deletion of its project, case and user, but not a hard purge or retention expiry; a purge removes the files, not only the rows. | IN | Defect | §4.13† |
| EVD-07 | A JUnit XML report is produced beside the HTML report. PDF is not offered. | IN | Not built | §4.13† |
| EVD-08 | A legal-hold feature. | OUT (until a customer obligation) | — | §4.13† |
| EVD-09 | Evidence is stored on S3-compatible object storage readable by every instance. | IN | Defect: local disk, unreadable by a second instance | §3, §9.2† |

## 2.15 Result sync (SYN)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| SYN-01 | One aggregate result per source case, keyed by its id; the most severe outcome wins (`FAILED`, `ERROR`, `CANCELLED`, `SKIPPED`, `PASSED`); a comment with the failed step, the sanitised error, a link to the execution and the identifiers; evidence attachments within the connector's limits. | IN | — | §1.1, §4.14† |
| SYN-02 | A defect is never created; linking an existing defect is not offered until asked. | IN / OUT | — | §1.4, §4.14† |
| SYN-03 | Recorded runs of syncable cases write back, whichever screen launched them (plan, suite or feature); draft runs never do. | IN | Partial: only the plan path writes back | §4.14† |
| SYN-04 | An optional build or version under test is captured at launch, stored with the execution and included in the comment; it is supplied, never inferred or produced by a model. | IN | Not built | §4.14† |
| SYN-05 | The comment names the case version that ran, recorded at run time. | IN | Defect: names the version current at sync time | §4.14† |
| SYN-06 | A failed sync is an operational problem, never a failed run; it is visible and retryable. | IN | Built | §4.14† |
| SYN-07 | Locally authored and spec-derived cases never write back. | IN | Built | §4.14† |
| SYN-08 | Nothing is skipped silently: every excluded scenario is reported with its reason, and the console says so before the run. | IN | Defect: the exclusion note is discarded by the sync worker | §4.14† |
| SYN-09 | Write-back can be turned off per project (a read-only integration). | IN | Not built | §4.14† |

## 2.16 Dashboards and analytics (DSH)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| DSH-01 | The project dashboard serves the QA author and lead, is reached from the project and scoped to it, and answers: what is failing and why, what is flaky, what is left to implement, and whether a scenario ever passed. | IN | Partial: reachable only from the global dashboard | §4.15† |
| DSH-02 | The global dashboard and the component-deprecation surface are vendor-only operational surfaces, removed from the customer sidebar. | IN | Defect: offered to every user, cross-tenant | §4.15† |
| DSH-03 | Requirement coverage is removed completely: card, types, test, i18n keys, dashboard field, coverage and link-writing endpoints, service, and the `requirement_links` table (forward migration with undo, after confirming it holds no customer data). | IN (removal) | Defect: full vertical, never drawn | §4.15† |
| DSH-04 | Ranges of 7, 30 and 90 days are not capped by evidence retention; a figure older than the evidence window is shown as computed, not substantiated. | IN | — | §4.15† |

## 2.17 AI (AI)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| AI-01 | AI is off by default, permanently; the product works completely with it off; every capability has a deterministic fallback. | IN | Built | §8† |
| AI-02 | The switch belongs to the organisation (ORG-12). | IN | Defect: per project | §8† |
| AI-03 | Nine capabilities in scope: structure an imported free-text case into canonical steps; propose a catalog step for an unmapped step; explain reasons the platform computed; narrate a finished run as a separate artifact that never alters the report; propose environment settings found in the source; flag near-duplicate cases (never merge or delete); compose a macro from existing steps; name a step sequence a deterministic count found; draft canonical cases from parsed endpoints or criteria (hidden where AI is off). | IN | Built (registered) | §8† |
| AI-04 | Two experiment-only capabilities, opt-in and advisory only, not sold until each passes its evaluation (G31): agentic failure triage, and proposing which provider field carries which canonical role. Deterministic grouping of failures by cause does not depend on them. | IN (experiment) | Built (registered) | §8† |
| AI-05 | A self-hosted model is supported and required by nothing; an unmapped model tier is refused, never defaulted. | IN | Built | §8† |
| AI-06 | The "AI never" list is a customer commitment: a model never decides pass/fail; never produces a count, duration or status; never writes a sync payload or renders Gherkin; never makes an authorisation or outbound-policy decision; never computes a source version hash; never touches a credential; never invents test data, credentials, URLs, selectors or expected results; never alters a step pattern. | IN | Built (G28-G33) | §1.3, §8† |
| AI-07 | "A model may say an expected result is missing, but may never supply one" is published. | IN | — | §4.6, §8† |
| AI-08 | What a model touched stays visible: spec-derived labels, composed-step origin, human or system review transitions. | IN | Partial | §8† |
| AI-09 | Who pays for model usage, and the hosted provider, region and agreement. | OPEN | — | §8† |

## 2.18 Audit, notifications, settings and retention (AUD, NTF, SET, RET)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| AUD-01 | One append-only audit table: scoped by organisation with optional project; actor stored as a stable string (never a foreign key); a real target column; the kind of actor (person, service account, vendor support, system) from the first migration; one row per decided action with before and after, redacted (G23); the retention sweep runs under a separate database role. | IN | Not built: six incompatible audit-shaped stores | §4.17† |
| AUD-02 | Organisation-visible events: permission granted, changed or revoked; member added, removed or re-roled; two-factor requirement set and factor removed by an administrator; AI egress changed; connector configured; credential created, updated, read, deleted or rotated; protected flag set or cleared; launch against a protected environment; project soft-deleted, restored or purged; organisation renamed or soft-deleted; scenario or evidence exported; machine credential issued, revoked or first used. | IN | Not built: permission grants are not audited | §4.17† |
| AUD-03 | Project-visible events: review transitions; scenario created, edited or deleted; macro version published; execution launched with actor, environment and case versions; sync-job outcome. | IN | Partial | §4.17† |
| AUD-04 | Vendor-only events (sign-ins, refreshes, lockouts, password resets, vendor support access) stay in their trail and are never shown to customers. | IN | Built (login trail) | §4.17† |
| AUD-05 | A customer-visible audit screen for the organisation administrator, with export. | IN | Not built | §1.5, §4.17† |
| AUD-06 | The organisation column on project-owning data (ORG-14) lands before or with the audit table. | IN | — | §4.17† |
| NTF-01 | Notifications go by email and by one signed outbound webhook whose URL is validated on save and before every call, with no redirects (G2), proven by an attack-path test (G21) and redacted (G23). Slack and Teams arrive through the webhook; native adapters are not offered. | IN | Partial: email transport built, off by default | §4.17† |
| NTF-02 | Events that notify: run failed; run finished (by subscription); scheduled re-import with what it changed; failed sync job; connector credential rejected; macro version published with the number of affected scenarios. Subscriptions per project, user and endpoint. A digest is not offered. | IN | Not built: nothing is sent when a run finishes | §4.17† |
| NTF-03 | A notification is a durable job fed by an outbox consumer registry, never a fire-and-forget call. The console bell shows real deliveries or is removed. | IN | Defect: bell shows a roadmap of unbuilt notifications | §4.17† |
| SET-01 | An organisation settings screen, changeable at runtime, covers only: AI data egress, offered connectors, two-factor required, retention values within the vendor range, and notification subscriptions. Deployment switches stay environment variables. | IN | Not built | §4.17† |
| RET-01 | Retention in three classes: evidence (EVD-04); operational, vendor-set (outbox, import items, AI invocations, drafts, rate-limit windows); audit, with organisation- and project-visible rows kept long (number: owner decision) and vendor-only rows 90 days, under a configuration key. | IN / OPEN | Partial: fourteen separate mechanisms | §4.17† |
| RET-02 | Delete-on-request is an audited purge inside the product, replacing the operator script; one organisation-visible audit row survives it; erasure reaches backups only as the 14-day rotation passes, and that is stated. Whether erasure is a contractual obligation is an owner decision. | IN / OPEN | Partial: operator script only | §4.17† |

## 2.19 Machine access (MCH)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| MCH-01 | A service account is a first-class principal: it belongs to one organisation, receives grants through the same model as a person, never holds the vendor role and never owns a project. Authorisation takes a principal before the first token is issued. | LATER (MVP-1.0.1) | Not built | §4.18† |
| MCH-02 | A machine credential is revocable, expiring, attributable (last-used time) and scoped (projects, read or write); issuing, revoking and first use are audited. | LATER (MVP-1.0.1) | Not built | §4.18† |
| MCH-03 | The public API is declared, versioned and narrower than the console's: launch, poll, fetch result, fetch evidence, in the 202-and-poll shape, with its own OpenAPI document served in production and a compatibility guard. | LATER (MVP-1.0.1) | Not built | §4.18† |
| MCH-04 | A service account may never launch against a protected environment. | IN (invariant now) | — | §4.18† |
| MCH-05 | The CLI is a client of the public API only, plus one GitHub Action. Jenkins and GitLab plugins, and an inbound webhook as the first machine door, are not offered. | LATER (MVP-1.0.1) / OUT | Not built | §4.18† |
| MCH-06 | When machine access ships, §1.3 (review as console convention) is re-decided as a contract or an auditable property. | LATER (MVP-1.0.1) | — | §1.3, §4.18† |

## 2.20 Test layers (HTTP, WEB, MOB, DB)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| HTTP-01 | REST calls on two interchangeable engines with six authentication schemes (Bearer, Basic, API key in header or query, OAuth2, client credentials, JWT) over nine steps; assertions on status, headers, body and JSON Schema; seven concurrency steps; seven response-security steps; polling and retry. | IN | Built | §6.1†, §6.2† |
| HTTP-02 | NTLM, Digest, AWS SigV4, mutual TLS as a step, WebSocket, gRPC and HAR are not claimed; each can arrive only through a step request. | OUT | — | §6.2† |
| HTTP-03 | File upload through the fixture store (ENV-08). | IN | Gated: engine complete, steps hidden | §6.2† |
| WEB-01 | Playwright drives forms, frames, tabs and Shadow DOM (nested hosts included), validates 19 field formats, and filters every browser connection through the outbound proxy on managed runs. | IN | Built | §6.3† |
| WEB-02 | The user chooses Chromium, Firefox or WebKit for each Web execution; managed runs accept all three. | IN | Defect: managed runs accept Chromium only | §1.5, §6.3† |
| WEB-03 | No browser version or branded channel (Chrome, Edge) is offered or promised. | IN | — | §6.3† |
| WEB-04 | Selenium is not an engine, and its leftover configuration is removed. | IN (removal) | Defect: dead Grid configuration | §6.3† |
| WEB-05 | Applications behind a login, SSO pages included, can be tested; generating a second factor for the system under test is not claimed. | IN | Built | §6.3† |
| WEB-06 | Element inspector: the user supplies the application URL, A·Spectra opens it with Core's Playwright through the customer-side runner, the user picks the element, and the platform proposes the selector. | IN | Not built | §1.5 |
| WEB-07 | Visual regression, video, trace, HAR, network mocking and accessibility assertions are not offered. | OUT | — | §6.3†, §11† |
| MOB-01 | Native and hybrid apps on Android emulators, Android physical devices, iOS simulators and any Appium-compatible hub, through the agent the customer installs next to the devices. | IN | Built | §6.4† |
| MOB-02 | Physical iOS devices and a vendor device farm. | OPEN (owner spending decision) | Not built | §6.4† |
| MOB-03 | The app under test is stored against the project with its build identity and referenced by the scenario. | IN | Not built: hand-copied absolute path | §6.4† |
| MOB-04 | Mobile web in a device browser is not claimed. | OUT | — | §6.4† |
| MOB-05 | Devices are leased exclusively; a run asking for a busy device fails fast; no concurrency number is promised. | IN | Built | §6.4† |
| DB-01 | PostgreSQL, SQL Server and Oracle are promised. | IN | Built: 51 of 52 steps executed end to end on each | §4.4, §6.5† |
| DB-02 | MySQL and H2 stay out until executed end to end. | OUT | Gated | §4.4, §6.5† |
| DB-03 | Writes are allowed as steps; the one step with an inline password requires an environment reference for it. | IN | — | §6.5† |

## 2.21 Non-functional (NFR)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| NFR-01 | Authoring-loop speed per launch, when a run slot is free: API or Database under 5 s, Web or Mobile under 20 s, from the launch click to the first step result. | IN | Not built: no warm browser pool | §1.5, §9.1† |
| NFR-02 | When capacity is exhausted the product queues to a bounded depth, then refuses with a retry time computed from the real queue; slots in use are measured. | IN | Defect: `Retry-After` hardcoded to 5 s, nothing measured | §4.12†, §9.1† |
| NFR-03 | The warm browser pool lives in the customer-side runner, one per engine, never in Core. | IN | Not built | §9.1† |
| NFR-04 | Enforced limits are published: 500 cases per import job, the two-hour run timeout, and inbound request limits once shared across instances. | IN | — | §9.1† |
| NFR-05 | A deploy is zero-downtime and loses no in-flight run. | IN | Defect: runs longer than the 120 s drain are lost | §3, §9.2† |
| NFR-06 | High-availability work: evidence on object storage (EVD-09), durable sign-in and two-factor state (IAM-12), the inbound rate limiter on the database-backed pattern; sticky sessions are rejected. | IN | Not built | §3, §9.2† |
| NFR-07 | No recovery promise is published until a restore is proven; backups cover evidence. | IN | — | §9.2† |
| NFR-08 | Encryption at rest for the database and evidence, published once true. | IN | Partial: credentials only | §9.3† |
| NFR-09 | A static egress address exists and the console displays it. | IN | Built (empty until hosted) | §9.3† |
| NFR-10 | Interface languages are English, Spanish and French, held equal by a test; display text is localised, identity (step patterns) never is; provider messages pass through untranslated. | IN | Built | §9.5† |
| NFR-11 | Accessibility target WCAG 2.1 AA, as the automated gate enforces; the gate header claiming 2.2 is corrected; a manual checklist covers what automation cannot. | IN | Defect: header claims 2.2 | §9.5† |
| NFR-12 | The console supports current Chromium-based browsers, Firefox and Safari, and the end-to-end gate runs all three engines. | IN | Partial: gate runs Chromium only | §9.6† |
| NFR-13 | The console is desktop-first and usable down to 320 px. | IN | Built | §5.2†, §9.6† |
| NFR-14 | Customers see their own runs and jobs in one organisation-scoped view of durable jobs and failures; health and info endpoints are public; metrics need a dedicated credential; logs are structured with a correlation id. | IN | Partial | §9.7† |

## 2.22 Operations and lifecycle (OPS)

| ID | Requirement | Scope | Today | Source |
|---|---|---|---|---|
| OPS-01 | An upgrade never breaks stored data or approved scenarios; migrations are forward-only and a failed upgrade is recovered from backup. | IN | Built: forward migrations proven; rollback unproven | §10† |
| OPS-02 | Stored and approved scenarios stay backward compatible indefinitely. | IN | — | §10† |
| OPS-03 | Withdrawal of a connector, step or capability is announced and named in the product before it happens. | IN | — | §4.10, §10† |
| OPS-04 | Versioning is release-numbered and the catalog is versioned with it. | IN | — | §10† |
| OPS-05 | A runbook is a shipped deliverable maintained with the product. | IN | Defect: the deploy workflow points at a runbook section that does not exist | §9†, §10† |
| OPS-06 | A trial exists, with the read-only integration (SYN-09) as its low-friction path. | IN | Not built | §10† |
| OPS-07 | The landing page copy matches `alcance.md` and the code, in all three languages. | IN | Defect: contradicts itself on connectors | §1.4, §5.2† |
| OPS-08 | "Built but not offered" is never silent: the run-result email with no caller, the scheduled-run label with no writer, Selenium configuration, GraphQL/SOAP advertisement and other switches hiding finished work are each offered, removed or visibly gated. | IN | Defect | §11† |

---

# Part 3 — Requirements by screen

The 31 routed page components in `qa-platformFE` are the mechanical anchor (§5). Scope: **K**eep,
**R**edesign, **D**rop (§5.2†). **Bold** screens must be excellent: they are the path by which a title
and an ID become a running scenario with evidence. Every screen must also meet NFR-10 to NFR-13.

| # | Route | Screen | Scope | Requirements that apply |
|---|---|---|---|---|
| 1 | `/` | LandingPage | R | OPS-07, P-03, P-08 |
| 2 | `/login` | LoginPage | K | IAM-01, IAM-08 |
| 3 | `/login/mfa-challenge` | MfaChallengePage | K | IAM-01, IAM-05 |
| 4 | `/forgot-password` | ForgotPasswordPage | K | IAM-01 |
| 5 | `/signup` | SignupPage | K | IAM-04 |
| 6 | `/reset-password` | ResetPasswordPage | K | IAM-01 |
| 7 | `/oidc/callback` | OidcCallbackPage | K | ORG-09, IAM-01, IAM-12 |
| 8 | `/auth/force-change-password` | ForceChangePasswordPage | K | IAM-01 |
| 9 | `/onboarding` | OnboardingPage | K | — (reachable only by automatic redirect) |
| 10 | `/dashboard` | GlobalDashboardPage | R | DSH-02 |
| 11 | `/dashboard/projects/:id` | ProjectDashboardPage | R | DSH-01, DSH-03, DSH-04, EXE-09 |
| 12 | `/projects` | ProjectsPage | K | PRJ-01, PRJ-03 |
| 13 | `/projects/:id` | ProjectDetailPage | K | PRJ-01, DSH-01 (entry point) |
| 14 | `/projects/:id/settings/:tab` | ProjectSettingsPage | R | PRJ-02 (`general`: name and description); `execution-defaults` tab removed (environment and runtime are chosen at launch) |
| 15 | — | ProjectMembersTab | R | ORG-02, ORG-04, AUD-02 (shows the owner; the vendor role is not a member) |
| 16 | — | ProjectIntegrationsPage | R | INT-01..INT-04, INT-12, INT-15, SYN-09, ENV-07 |
| 17 | `/projects/:id/environments` | **EnvironmentsPage** | K | ENV-01..ENV-08, MOB-03 |
| 18 | `/projects/:id/ssl/certificates` | CertificateManagerPage | K | ENV-04 (the "trust all" option goes to security review) |
| 19 | `/projects/:id/suites` | SuitesPage | K | EXE-01 |
| 20 | `/projects/:id/suites/:id` | SuiteDetailPage | K | EXE-01, EXE-03, SYN-03, WEB-02 |
| 21 | `.../features/:id` | **FeatureDetailPage** | K | AUT-03, EXE-03, SYN-03 |
| 22 | `.../scenarios/new`, `/edit` | **ScenarioBuilderPage** | K | AUT-01, AUT-05, REV-05, EXE-11, WEB-06 (hosts the element inspector), ENV-08, MAC-04 |
| 23 | `/steps` | **StepsPage** | K | CAT-01..CAT-08, P-01 |
| 24 | `/projects/:id/custom-steps` | CustomStepsPage | R | MAC-01..MAC-04 |
| 25 | `/projects/:id/imports` | ImportsPage | R | IMB-01..IMB-09, IMC-01..IMC-04 |
| 26 | `/projects/:id/suggestions` | ScenarioSuggestionsPage | R | IMB-01..IMB-06 (redesigned to §4.6, not locked into tests first) |
| 27 | `/projects/:id/suggestions/:id` | ScenarioDraftBuilderPage | R | IMB-01, IMB-02, IMB-05 |
| 28 | `/projects/:id/execution-overlay` | **ExecutionOverlayPage** | K | IMA-01..IMA-08, REV-01..REV-09, EXE-01, SYN-01..SYN-08 (six panels: import, review queue, case detail, step mapper, sync jobs, feature import) |
| 29 | `/projects/:id/executions` | HistoryPage | K | EVD-05, EXE-09, EXE-10 |
| 30 | `/.../executions/:id` | **ExecutionLivePage** | K | EXE-07, EXE-08 |
| 31 | `/.../executions/:id/results` | **ExecutionResultsPage** | K | EVD-01..EVD-07, ORG-20 |
| 32 | `/settings` | SettingsPage | K | IAM-05, IAM-07 (no trusted-devices tab), IAM-09 |
| 33 | `/admin/users` | AdminUsersPage | R | ORG-03, ORG-08, ORG-15 (becomes the organisation administrator's member management, route-guarded) |
| 34 | `/components/*` | ForcedNotFoundPage | K | A deliberate 404 for links to a retired surface |
| 35 | `*` | RouteErrorFallback | K | "Report issue" appears only where a deployment sets its URL, never on a customer deployment (G22) |

## Screens and surfaces still to design or build

| ID | Requirement | Scope | Source |
|---|---|---|---|
| SCR-01 | The overlay's six internal panels are designed, not one frame for six panels. | IN | §5.1† |
| SCR-02 | The agent surface (onboarding, readiness checks, device selection) is designed as the onboarding of the customer-side runner, not as a mobile detail. | IN | §5.1† |
| SCR-03 | The AI assistant panels stay undesigned while AI is off. | IN (gap accepted) | §5.1† |
| SCR-04 | The organisation settings screen (SET-01) and the audit screen (AUD-05) are new surfaces for the organisation administrator. | IN | §4.17† |
| SCR-05 | Dropping a screen drops its endpoints and its data (DSH-03 is the worked example). | IN | §5.2† |

---

# Part 4 — Non-goals

The permanent non-goals, the "not yet" items and their deciding sections are listed once, in
[alcance.md §11](alcance.md#11-explicit-non-goals). Permanent non-goals may be stated publicly; "not yet"
items are roadmap and are not.

---

# Part 5 — Owner decisions still open

These block a requirement's final form. The authoritative list is the "Still open" block of each
`alcance.md` section; this table indexes them.

| Topic | Requirement | Source |
|---|---|---|
| Hosting region | ORG-19 | §3 |
| Step deprecation notice period | CAT-06 | §4.10 |
| Evidence retention vendor range | EVD-04 | §4.13† |
| Audit retention for organisation- and project-visible rows | RET-01 | §4.17† |
| Erasure on request as a contractual obligation, turnaround, backups | RET-02 | §4.17† |
| Physical iOS and a device farm | MOB-02 | §6.4† |
| Jira Marketplace listing | INT-16 | §7† |
| Who pays for model usage; hosted model provider, region, agreement | AI-09 | §8† |
| Target scale, 500-case import time, availability, recovery point and time, status page, security standard, penetration-test cadence and remediation window, static egress commitment, BYOK, jurisdictions, sub-processors, terms and IP, WCAG contractual claim, fallback language | NFR-01..NFR-14 | §9† |
| Commercial model, trial shape, hosted operator and hours, support model, onboarding, release cadence, backup promise, first customer | OPS-06, OPS-08 | §10† |

---

# Part 6 — Traceability

Every requirement reaches implementation through a GitHub issue in the owning repository, with
acceptance criteria, a test that proves each criterion, and a commit with green gates (G16); a screen is
named only where there is one (§13). A requirement without acceptance criteria cannot reach
implementation. Reference requirements in issues by ID (for example `Implements REV-08`). When
`alcance.md` changes, this document is updated in the same change set, and its status line records the
`alcance.md` revision it follows.
