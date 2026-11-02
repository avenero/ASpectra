# ASpectra — Product scope

> **What this document is.** The complete scope of the ASpectra product — every capability it has, the authorities that own them, and the boundaries it never crosses. Functional detail lives in **`requirements.md`**.
>
> **Source.** Everything here is derived from the ASpectra design file (Figma `DpfPk8xDraLcDzBhMYwDTo`).

---

## 1. What ASpectra is

ASpectra is a **multi-tenant SaaS platform that turns requirements into executable, catalog-bound test scenarios and runs them deterministically, with humans owning every consequential decision.**

A customer brings requirements — user stories with acceptance criteria, API and product documentation, OpenAPI specs, Postman collections, existing `.feature` files, repository content — or simply writes test intent in their own words. ASpectra maps that intent onto a **published catalog of real, developer-authored executable steps**, shows the customer exactly what the mapping means, and only then compiles, seals and (after an explicit human approval and a separate explicit Run) executes it against the customer's own system under test. It returns immutable execution facts, evidence and a deterministic report, and can deliver them to the customer's own tools.

It is a QA and test-automation product, not an agent framework. The governing thesis:

> **AI proposes. Humans authorize. Core decides.**
> *Catalog-grounded test specifications, human-owned execution.*

### What makes it different from "an LLM that writes tests"

The design is built around one refusal: **AI never produces executable code, never approves, and never launches.** Scenarios can only be composed from steps that already exist as real Java implementations published in an immutable catalog. If a requirement cannot be expressed with existing steps, that is surfaced as a **visible capability gap** — never papered over with an invented step, a guessed assertion or a nearest-match binding. New executable capability requires a developer to write a Core handler, with tests, in a released version.

---

## 2. The authorities

Responsibility is split three ways, and the split is semantic, not deployment-based.

| Authority | Owns | Never does |
|---|---|---|
| **FE** — authoring, decisions, projections | Workspace UI, authoring, human review, explicit Run command, result projections | Decide authorization or verdict; seal executable hashes; touch Core runtime |
| **BE** — domain truth, policy, orchestration | Tenancy, permissions, canonical revisions, approval records, durable dispatch, admission, facts, reports, delivery, AI policy | Decide test semantics; execute browsers or devices in the main process |
| **Core** — semantics and execution | The step catalog, pure compilation, deterministic execution, the test verdict | Know about tenants, IAM, Jira, AI providers or customer business data |

The **runner** is a BE-owned out-of-process host that embeds pinned Core. It owns admission, resources, device leases, journaling and process supervision — deployment concerns — while Core alone owns test meaning. *"BE owns runner deployment, dispatch and lifecycle. Core is the pinned library inside that process and alone owns test semantics and verdict."*

Outside all three sits the **PlatformOperator**: a platform principal, not a tenant role, that allocates shared platform resources between customers. It publishes `ManagedNetworkProfile` (actual egress identity, execution region, capabilities), versions `ExecutionCapacityPolicy`, and issues `TenantAIEntitlement`. It can never accept a mapping, approve a bundle, launch a Run, mutate a revision or verdict, read customer content as a side effect of configuration, or grant itself tenant access. Its surface is an internal, audited, out-of-band configuration path that is never exposed to customers. Absent configuration is `UNCONFIGURED` and fails closed.

---

## 3. The workflow

```
C01 catalog (published by Core build)
        ↓
intent in  →  C11 visible mapping  →  human accepts meaning
(import / repository / structured / free-form)   ↓
                                   C02 immutable TestRevision
                                          ↓
                                   C03 pure compile + sealed bundle   (no target contact)
                                          ↓
                                   C04 human approval                 (approval never launches)
                                          ↓
                                   C05 explicit Run                   (human canRun, or a C05-S
                                          ↓                            service principal re-launching
                                          ↓                            an already-approved bundle)
                                   C06 runner admission → Core executes → C07 immutable facts
                                          ↓
                            base report  →  optional C08 analysis / C09 delivery
```

**Three distinct human decisions, none implying the next:**
1. **Accept the meaning** (C11) — "yes, that is what I meant" → creates a draft.
2. **Approve the artifact** (C04) — a Reviewer approves the exact sealed bytes, selection, environment and policy.
3. **Run it** (C05) — a separate explicit command requiring an independent `canRun` grant. A machine trigger may only re-launch what a human has already approved, under its own grant.

Every entry path converges on the *same* canonical model: imported artifacts, repository evidence, structured catalog building, and free-form natural-language authoring. AI is optional on all of them — with AI disabled, manual authoring, validation, approval, execution and reporting all still work.

---

## 4. The contract set

| Contract | What it is |
|---|---|
| **C01** CatalogManifest | Immutable published step catalog: stable stepIds, typed schemas, binding fingerprints, availability, artifact digests |
| **C02** TestRevision | BE-authoritative immutable canonical scenario (the single editable source of truth) |
| **C03** ValidationReport + CompiledBundle | Pure static compile; sealed executable bytes plus every dependency digest |
| **C04** ApprovalRecord | A human's decision on an exact bundle — with no launch side effect |
| **C05** LaunchCommand / RunnerSubmission | Explicit Run → durable, idempotent, fenced dispatch through fair admission |
| **C06** RunnerCapabilities | Profile/build/OS/browser/device/network constraints matched before any target contact |
| **C07** RunEvent / ResultEnvelope / ArtifactManifest | Immutable semantic verdict plus separate infrastructure, cleanup and evidence status |
| **C08** ProposalEnvelope | Optional AI/import/repository proposals — accepted into a draft only, never approval or run |
| **C09** DeliveryIntent | Policy-controlled external publication with dedupe, receipts and per-item reconciliation |
| **C10** TrustedActorTenantContext | Server-derived tenant/project/actor authority — never browser-claimed |
| **C11** AuthoredIntentMapping | Original wording preserved beside derived executable meaning, with gaps visible |
| **C12** AIPolicyAndUsage | Approved provider/model/egress/funding plus atomic budget reservation and reconciliation |

### Typed supporting contracts

These extend the contracts above without introducing a C13 or renumbering anything.

| Supporting type | Parent | What it adds |
|---|---|---|
| **C01-M** MacroRevision | C01 | Project-owned immutable macro: a namespace that can never shadow a Core stepId, typed inputs/outputs, ordered CoreRef/MacroRef AST, bounded cycles and depth, `expandedHash` + sourceMap. Author drafts, Reviewer publishes. Macro review is **not** C04. |
| **C01-A** AssetRevision | C01 | Governed project assets on two independent axes — intake (`UPLOADING → QUARANTINED → VALIDATING → READY / REJECTED`) and publication (`DRAFT → IN_REVIEW → PUBLISHED → DEPRECATED / REVOKED`). Kinds: **MobileAppRevision**, **FixtureRevision**, **SchemaRevision**, **LocatorRevision**, **PublicTrustMaterialRevision**. |
| **ExecutionCapacityPolicy** + **AdmissionTicket** | C05 | Operator-versioned tenant and global ceilings (`maxActiveRuns`, `maxScenarioWorkerSlots`, `maxQueuedTickets`, `maxDeviceLeasesByPool`, `maxQueueWait`) and a durable ticket carrying scope, pins, demand, FIFO sequence, deadline, holdReason, lease/attempt/fence. Independent of C12 AI budgets. |
| **C05-S** service principal | C05 | Machine-initiated launch: re-launches only an already human-approved exact `bundleDigest` and `envRevision` under its own `canRun`, idempotency key and concurrency ceiling. |
| **ManagedNetworkProfile** + **ReachabilityAssessment** | C06 | Operator-published profile with **actual egress identity**, execution region, network version and capabilities; plus a consented, bounded, time-expiring probe result. |
| **DeliveryProfileRevision** + **ItemReconciliation** | C09 | Per-destination entity/field mapping and frozen per-item receipts: `PENDING / CONFIRMED / UNMAPPED / RETRYABLE_FAILURE / PERMANENT_FAILURE / UNKNOWN`. |
| **TenantAIEntitlement** | C12 | Audited allocation issued by the **PlatformOperator** alone; default `DISABLED`; a tenant may lower limits or use approved BYOK but can never raise a funded cap. |

---

## 5. Product capabilities

Each block names the requirement area that specifies it. This section describes the whole product; `constitution.md` Part II names which of these capabilities form the implementation baseline, and its principle P10 names the technology that baseline is built with. Capabilities marked *(extension — outside the MVP baseline)* are specified in `requirements.md` area I or as lifecycle jobs, are listed in `constitution.md` III.4, and, where designed, live on the Figma Post-MVP page.

**Tenancy, identity and access** — `requirements.md` A, F; I.2, I.3
- One hosted multi-tenant SaaS product; one customer organization = one tenant. Shared infrastructure, never shared access. Personal, SMB and enterprise customers use the same product.
- Global identity → tenant membership → explicit project grants. Tenant roles **Owner / Admin / Member**; project roles **Author / Reviewer / Viewer**; **`canRun` is a separate fixed grant**. Tenant administration alone grants no project content access.
- Local verified identity hardened with Spring Security: JWT/refresh families with rotation and replay detection, TOTP mandatory for Owner/Admin, hashed single-use invitations and reset tokens, controlled recovery.
- **Federated identity (OIDC / SSO)** (extension — outside the MVP baseline) with explicit account linking bound to an invitation or tenant — never membership by email domain.
- **Phishing-resistant factors** (extension — outside the MVP baseline) (WebAuthn security keys, passkeys) under the same required-MFA policy; no trusted-device bypass.
- PostgreSQL FORCE RLS plus runtime authorization as two independent gates; composite tenant/project keys.
- Solo operation: a founding Owner ticks Author, Reviewer and canRun explicitly on the first project (the Owner role implies none of them); self-approval is allowed but visible and audited, and `requireDifferentApprover` can tighten it.
- Tenant selection, creation, suspension, leaving and the erasure request; the tenant switch as a guarded state transition. Tenant deletion and export run as durable jobs (extension — outside the MVP baseline).

**Projects, environments and trust** — `requirements.md` B
- Ownership tree `Tenant → Project → Suite → Feature → Scenario → TestRevision`; versioned selection snapshots.
- Customer target environments as immutable, versioned domain objects with typed target aliases, URL policy, write-only secret references and pinned assets.
- Public X.509 CA trust material as governed, revisioned project assets.
- **Client certificate identity** (extension — outside the MVP baseline) — client mTLS, private keys and app re-signing — under scoped secret custody, never a TLS bypass or global trust mutation.
- Tenant overview, first-run readiness and a project dashboard with explicit, linked-criteria denominators.

**Catalog, macros and assets** — `requirements.md` C; I.7
- The global, read-only C01 catalog produced by the Core build, with four distinct availability axes (implemented ≠ offered ≠ runner-compatible ≠ live ready) and EN/ES/FR display labels.
- Project macros (C01-M) and governed project assets (C01-A) — mobile app binaries, fixtures, schemas, locators and public trust material — uploaded into private quarantine, validated on actual bytes, reviewed, published immutably and pinned by digest. Intake never installs or executes anything.
- **Cross-project reuse** (extension — outside the MVP baseline) of macros and assets by copy or explicit reference to an authorized immutable version.

**Authoring, sources and review** — `requirements.md` D; I.1
- Entry paths converging on C02: imported sources, structured catalog building, free-form wording and project chat.
- Sources: user stories (HU/UH) + acceptance criteria, product/API/architecture documentation, OpenAPI/Swagger, Postman collections, existing `.feature` files. Sources are **evidence only** — nothing imported is ever executed during ingestion.
- **Repository-informed proposals** (extension — outside the MVP baseline): read-only, commit-pinned GitHub/Bitbucket evidence, with code excerpts as cited context only.
- Original wording preserved exactly (with explicit SecretRef redaction) beside the accepted executable meaning; revision diff; pure C03 compilation with sealed bundles.

**Approval** — `requirements.md` E
- Human approval of exact bytes, selection, environment and policy; any execution-affecting change invalidates it; self-approval is policy-controlled and audited.

**Execution** — `requirements.md` H; I.4, I.5
- Four protocols: **HTTP/API**, **Web** (Playwright Java), **Mobile** native/hybrid (Appium Java), **Database** (JDBC/Hikari).
- Managed ASpectra runners with explicitly offered browser/device/OS profiles; iOS on a provisioned macOS profile.
- **Customer-network execution** (extension — outside the MVP baseline) — private or customer-side runners and warm browser pools — under the same admission and pin invariants.
- **Machine-initiated execution** (extension — outside the MVP baseline) — CI/pipeline triggers, schedules and a launch API — through the `C05-S` service principal only.
- Fair admission as a persisted protocol (`QUEUED → ADMITTING → RUNNING → COMPLETED`; eligible-tenant round-robin per compatible pool, FIFO within a tenant; atomic reservation; `202 = accepted, not started`). Ceilings are configuration with no invented defaults.
- Explicit, consented reachability checks from a managed profile; save, validate, approve and compile never contact the target. Execution region is not data residency.
- Per-run/scenario isolation: isolated process and workspace, fresh browser contexts, exclusive device leases, scoped credentials, LIFO cleanup that survives partial initialization.

**Results, evidence and delivery** — `requirements.md` H, F.7; I.6
- Immutable facts, durable replay, a deterministic base report independent of AI and connectors; artifacts with identity, redaction and authorized retrieval.
- Live progress via snapshot + STOMP with watermark/cursor recovery — a projection, never the source of truth.
- Run history and lineage from author revision to attempt.
- Optional C08 failure analysis (proposal only).
- C09 controlled delivery with per-item reconciliation across provider families: **Jira Cloud issues**, **Jira Server / Data Center**, **Xray Cloud / Data Center**, and native **Slack / Teams** — each enabled by its own adapter contract and per-capability evidence; Jira Cloud issues is the baseline family, the others are extensions outside the MVP baseline. `PARTIAL` is never `DELIVERED`; an unacknowledged write is `UNKNOWN` and held, never blindly resent.

**AI** (optional throughout) — `requirements.md` D.8, F.8, G.5; I.9
- Project-scoped chat and semantic mapping through a bounded BE port under C12 tenant policy: approved provider/model/endpoint, egress rules, and explicit platform-managed **or** BYOK funding with atomic budget reservation.
- **Customer-hosted inference endpoints** (extension — outside the MVP baseline) under the same C12 governance.
- PostgreSQL structured and full-text retrieval, with **derived, replaceable vector indexes** that are never domain authority.
- **Python workers** (extension — outside the MVP baseline) as optional adapters behind existing Java BE ports.

**Platform operation and commercial** — `requirements.md` G; I.10
- Operator-published network profiles, capacity policies and AI entitlements as immutable, versioned, audited records with a read-only tenant projection.
- A usage ledger separate from charging; a **billing / subscription processor** that consumes the ledger without replacing it (extension — outside the MVP baseline).

**Accessibility, language and audit** — `requirements.md` §1.5, §1.6
- **WCAG 2.2 AA** on critical journeys, evidenced by automated axe + Playwright **plus** manual assistive testing on NVDA (Chrome/Firefox, Windows) and VoiceOver (Safari, macOS/iOS). Keyboard reorder with announced position replaces drag-only ordering.
- EN / ES / FR interface and catalog labels — strictly display; UI locale never rewrites approved feature bytes.
- Append-only, redacted audit with visibility that follows current authority.

**Lifecycle** — `requirements.md` B.5, A.9
- Archive / trash / restore with cascade provenance (purge by retention workers (extension — outside the MVP baseline)); tenant suspension; tombstones that block late-callback resurrection; tenant deletion jobs and exports that exclude secrets (extension — outside the MVP baseline).

**The product surface** — `requirements.md` all areas, §1.6
- A **public landing surface** with exactly two entry actions — create workspace and sign in.
- An authenticated **app shell** carrying tenant and project context, the permitted destinations, and the EN / ES / FR switch.
- **Twelve product sections**: foundations and coverage index · public landing · identity and account security · tenant workspace and governance · projects, environments and integrations · suites, scenarios and catalog · sources, mapping and human approval · run, evidence and delivery · contextual AI and usage policy · responsive product views · security, permission and recovery states · queue, reachability, governed-asset and delivery-reconciliation surfaces.
- The full **execution journey**: Run confirmation, execution queue, queued-ticket detail, admission refusal, reachability assessment, live monitoring, degraded retrieval, results, artifact detail, run history and lineage, unknown acceptance, delivery jobs and partial-delivery reconciliation.
- **Genuine responsive reflows** for mobile and tablet, not scaled desktop layouts.
- Handoff material: UI foundations (D01), the critical state and interaction atlas (D02), the screen coverage and journey index with the Screen decisions ledger (D03), and the messages, copy and share-metadata guide (D04).

---

## 6. Product boundaries

These are decisions about what ASpectra **is not**. They hold for every release.

- **No repository execution or writes.** Repository content is read-only evidence: no code execution, builds, hooks, dependency installation, branches, commits, pull requests, CI triggers or sync subsystem.
- **No AI authority.** AI never approves, launches, reaches a runner, writes executable code, or auto-reruns; repair is proposal-only and never weakens assertion intent.
- **No generated capability.** No user- or AI-created handlers, Cucumber glue or scripts; new capability is a developer-controlled Core release.
- **No conditional assertion weakening.** "If-present" conditional steps are intentionally excluded from the catalog.
- **No cross-customer anything.** No shared caches, model context, learning, training reuse or cross-tenant metrics.
- **No membership by domain.** Neither an email domain nor a successful federated sign-in ever grants tenant membership; SAML, SCIM and automatic enterprise provisioning are not part of the identity model.
- **A finite role model.** Owner / Admin / Member and Author / Reviewer / Viewer plus `canRun`; there is no custom-role builder or team hierarchy.
- **No customer-facing operator surface.** Platform operation is internal, audited and out-of-band.
- **No inferred infrastructure.** No invented address, region, quota, threshold or default; no next-profile fallback; execution region is never presented as data residency; no promise that every private network is reachable.
- **No trust bypass.** No TLS bypass, no global trust mutation.
- **No exactly-once promise** for external effects; unknown outcomes are reconciled, never assumed.
- **No new platform stack.** No new language, microservice split, mandatory broker, mandatory vector database, mandatory vault, cloud or auth migration (see §8).

---

## 7. Non-negotiable invariants

- No FE → Core driver access; no browser execution in the BE main process.
- No AI → runner connection, launch tool or auto-rerun; repair is always proposal-only and may never weaken assertion intent to reach green.
- No auto-run on save, proposal acceptance, approval, reconnect, reload or delivery retry. Machine-initiated launch exists only through `C05-S`, and only for an already-approved exact bundle.
- No invented steps, guessed aliases, nearest-match bindings or hidden assertion translation. Undefined or ambiguous is **never** a PASS.
- Unknown infrastructure outcome is indeterminate — never a fabricated PASS or a failed assertion.
- Approval cannot bless changed bytes: any execution-affecting change invalidates it.
- Cancel ≠ rollback. Request retry ≠ test rerun. Acceptance ≠ approval ≠ Run.
- No cross-tenant state; browser-supplied tenant IDs are selectors, never authority.
- Secrets never reach prompts, logs, indexes, artifacts or reports — only scoped references reach runners.
- Analysis and delivery can never rewrite a Core verdict, and their outage can never suppress the base report.
- A queued ticket cancelled, rejected or expired before execution is **NOT EXECUTED** — it has no Core verdict and is never a failed assertion.
- A lost ACK or expired lease is not proof that execution stopped: the outcome is **INDETERMINATE** and reconciles against the existing submission, never a blind re-execution.
- Macro publication and asset publication are never scenario approval, and neither is ever a Run.
- Missing operator configuration blocks admission (`UNCONFIGURED`); nothing is invented and there is no fallback.
- A tenant can lower its own AI limits but can never raise an operator-funded cap, and BYOK never bypasses platform service caps.

---

## 8. Technology baseline

| Layer | Stack |
|---|---|
| **FE** | React 18, TypeScript, Vite, TanStack Query, Zustand, i18next, Tailwind, axios, STOMP |
| **BE** | Java 21, Spring Boot 3.5, PostgreSQL, Spring Security (JWT / refresh / MFA); hexagonal modular monolith with durable jobs and outbox |
| **Core** | Java 21, Gradle, Cucumber; six modules — `common`, `http-core`, `web-core`, `mobile-core`, `database-core`, `mobile-agent` |
| **Protocols** | Playwright **Java** (Web), Appium **Java** (native/hybrid Mobile), HttpClient 5 / Playwright request (HTTP), JDBC/Hikari (DB) |
| **Stores** | PostgreSQL (with structured and full-text retrieval), a private object store, and the existing encrypted credential repository with external master-key injection |
| **Optional adapters** | Python workers and derived vector indexes, behind existing Java BE ports only |

No new mandatory broker, language, runtime, DI or form framework. No TypeScript Playwright Test inside Java, and no Playwright substitution for Appium on native mobile. This table is the product's technology baseline; `constitution.md` P10 fixes the subset the implementation baseline uses (the mobile modules and protocol are product scope that the baseline does not bind).

---

## 9. Related documents

- **`requirements.md`** — the functional requirements for every capability above, each carrying the board, section, screen or contract it came from.
- **`constitution.md`** — the implementation baseline: the governing principles and the features the design defines completely, each traced to its requirements.
- **`specs/`** — one folder per specification, each covering one or more baseline features and holding its specification, technical plan and task list (see `specs/README.md`).
- Figma spec `122:2` — *Final multi-tenant MVP decision register*, the governing design record.
- Register `122:2` §15 — *Platform operator authority and configuration publication* (formerly the spec `227:2`, merged on 2 November 2026).
- Boards 1–7 on the architecture page (BE `23:5587`, Core `46:7241`, FE `46:7847`, Alignment `46:8271`, Import `79:3652`, Authoring `79:4061`, Failure `3:2192`); board 8 (repository-informed proposals) and the federation and passkey reference frames on the Post-MVP page.
- The product-experience page — twelve numbered sections, including the D01–D04 handoff guides and the D03 Screen decisions ledger (`148:4`), the record of the screen decisions S-01 … S-63.
