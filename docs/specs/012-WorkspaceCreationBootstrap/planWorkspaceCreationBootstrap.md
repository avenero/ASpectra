# Plan 012 — Workspace creation and bootstrap

**Feature** Creating a tenant workspace as its founding Owner, with an optional first project and explicit grants, in one auditable bootstrap · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/012-WorkspaceCreationBootstrap/specWorkspaceCreationBootstrap.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-14 as the subject; F-IDN-01, F-IDN-02, F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-11, F-IDN-12, F-IDN-13, F-IDN-15, F-IDN-18, F-PRJ-02, F-PRJ-12, F-SET-05 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (contexts `tenant`, `access`, `project`; "Onboarding creates auditable memberships / project grants"), decision register `122:2` §3 ("Tenant / first-Owner / project-grant bootstrap is transactional and auditable") and §10 ("Use a bootstrap / composition package to wire published ports and onboarding workflows … Composition can orchestrate these ports without introducing callbacks that invert the graph"; allowed direction identity / tenant primitives → access → project), FE board `46:7847` (feature `workspace / memberships / project`), board 1 `23:5587` ("Verify email before creating tenant + first Owner / grant bootstrap in one transaction") · **Builds on** plan 001 (Login: `LandingRouter`, `ThrottleStore`, idempotency outcomes), plan 002 (`IdentityFactsPort` as declared by 008), plan 003 (Enrollment: `FactorStatusPort`, `EnrollmentGateHandler`, `returnTo`), plan 006 (Sessions: `SessionAssertionPort`), plan 007 (Logout: `SignOutUseCase`), plan 009 (Roles and grants: `MembershipCommands`, `GrantCommands`, `AdministrationRule`, `TenantPolicyPort`, `TenantStatusPort`, `ProjectExistsPort`), plan 010 (Members: `TenantOwnershipRepository`, `tenant_ownership`); elements they own are tagged **[nnn]** and are extended, never duplicated. The project-creation command of F-PRJ-02 and the policy record of F-SET-05 are consumed through ports declared here and stubbed until their plans exist.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's six decisions (Q1–Q6, registers A-59…A-61, S-53, S-54) and two derived rules (D-a, D-b) are built as decided. Where the specification depends on items still open (A-11, A-27, A-36, S-02, S-04, S-14, S-17, S-34), the plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The bootstrap writes identity, tenant, access and project rows only; it reads no mapping, approval, launch or intent table and emits no launch. | RF-4.5, RF-4.6, RF-5.4, RNF-6 |
| **P3** The browser expresses intent, never authority | The page sends a name, an optional project name, three booleans and an idempotency key; eligibility, the Owner role, the grant records, the policy values and the destination are decided on the server; the "origin" that selects the Back variant is held in memory and chooses a label, never an authority. | RF-1.8, RF-3.6, RF-5.1, RF-6.2 |
| **P4** Only the platform catalog is global | Every row the bootstrap writes carries the new `tenant_id`; the only cross-tenant read is the identity-scoped eligibility check (verified, enrolled, status) on identity rows. | RF-4.1, RNF-10 |
| **P6** Fail closed | Any failed precondition or write aborts the whole transaction; an absent policy revision cannot occur because r1 is written in the same transaction; absent configuration fails startup. | RF-4.3, RF-6.5, RNF-1, RNF-8 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL transactions and `SET LOCAL`, the existing throttle, outbox and idempotency mechanisms. FE: React 18, TypeScript, TanStack Query, i18next, Tailwind, axios. Nothing is added. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the bootstrap is a use case of the **composition package** that orchestrates the published commands of `tenant`, `access` and `project` through their ports in the allowed direction; no context calls back into another; the policy record is created through F-SET-05's port. FE: the N04 page in the `workspace` feature; entries through app-level routing; no feature decides authority. | §2 |
| **P12** Two independent tenancy gates | The new tenant's rows are written inside one transaction whose scope is set to the new `tenant_id` right after the tenant row exists, so `FORCE RLS` `WITH CHECK` applies to every bootstrap row; authority for the tenant is derived afresh by `access` on the first request after creation. | RF-4.1, RF-5.1, RNF-10 |
| **P13** Configured values are never defaulted | The name length bound and the two creation-throttle dimensions are in `WorkspaceCreationProperties`, validated at startup; the first policy revision's numeric values come by reference from the configured platform policy (A-11), never from this feature. | RF-2.4, RF-1.7, RF-4.2, RNF-7 |
| **P14** Release bar | WCAG 2.2 AA on N04 with automated and manual evidence; EN / ES / FR for every string. | RF-9, RNF-11, RNF-12 |
| **F-X-01** C10 context | Creation itself is identity-scoped (no C10 exists before the tenant); the bootstrap establishes the tenant scope inside its transaction; entering the tenant afterwards derives C10 through plan 009. | RF-1.2, RF-4.1, RF-5.1 |
| **F-X-02** Request integrity | One idempotency key per submission with a recorded outcome; same key + body → recorded outcome; same key + different body → conflict; a lost acknowledgement leads to a lookup. | RF-4.4, RF-6.6, RNF-4 |
| **F-X-03** Secrets and safe problems | Problems carry safe codes and a correlation id; nothing of the page is stored in the browser; no answer discloses another tenant or identity. | RF-7.4, RF-8.2, RF-8.3, RNF-9 |
| **F-X-05** Audit log | Every creation, refusal and failure appended through `AuditPort`, in the bootstrap transaction for successes. | RF-8.1 |
| **F-X-06** Configuration and email | `WorkspaceCreationProperties` validated at startup; no message is sent (Q5). | RNF-7, RF-4.6 |
| **F-X-07** Responsive surfaces | N04 reflows; statements shared across widths. | RF-9.1 |
| **F-IDN-04 / 11 / 12 / 13 / 18** boundary | Enrollment, grant records, the ownership row, the grants editor and the guarded switch are reused through their plans' elements, never redefined. | RF-1.5, RF-3, RF-4.1, RF-5.1 |

---

## 2. Module structure

### 2.1 Back end — composition package `com.aspectra.composition.onboarding`, with one query in `identity` and consumed ports of `tenant`, `access`, `project`

The register's §10 names a bootstrap / composition package that wires published ports and onboarding workflows. This feature is that workflow: it adds one eligibility query (identity-scoped), one bootstrap use case in the composition package, one configuration record, one controller, and declares the ports it consumes from `tenant` (tenant record, policy record), `access` **[009/010]** (membership, grants, ownership row), `project` (project creation, F-PRJ-02) and `audit`. It adds no aggregate of its own: every row it writes belongs to another context.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.application` | `WorkspaceCreationEligibilityQuery` | For the caller's identity: `verified` (email verified, status `ACTIVE` — A-27), `enrolled` (`FactorStatusPort.hasActiveFactor` **[003]**), `throttled` (`ThrottleStore` **[001]** on the creation dimensions). Returns `ELIGIBLE \| VERIFICATION_PENDING \| ENROLLMENT_REQUIRED \| NOT_PERMITTED \| THROTTLED`; used at entry and re-run inside the bootstrap transaction. | RF-1.2–1.4, RF-1.6, RF-1.7, RF-1.8, LC-4, LC-5, LC-18, LC-19 |
| `composition.onboarding` | `WorkspaceName`, `ProjectName` | Value objects: trimmed, non-empty, printable, within `WorkspaceCreationProperties.nameMaxLength`; no uniqueness, no normalisation beyond trimming. | RF-2.4, RF-3.1, LC-7, LC-8 |
| | `FirstProjectRequest` | Value: project name + three booleans; `toGrantSet()` → plan 009's `GrantSet` (any subset, `canRun` allowed alone). | RF-3.3, RF-3.5, RF-3.6, LC-10–LC-12 |
| | `CreateWorkspaceUseCase` | The bootstrap (D1, D2). In one transaction: (1) return the recorded outcome for a repeated idempotency key, conflict on a different body; (2) `WorkspaceCreationEligibilityQuery` again under the session's identity — any non-eligible answer aborts with its problem; (3) count the attempt on the creation throttles; (4) `TenantCommands.createTenant(name)` → tenant id, status `active`; (5) set the transaction scope to the new tenant id (`SET LOCAL`, D3); (6) `TenantOwnershipRepository.create(tenantId)` **[010]** (the ownership row the last-Owner guard locks); (7) `MembershipCommands.CreateMembership(tenantId, identityId, OWNER, source = BOOTSTRAP)` **[009]** → membership at version 1; (8) `TenantPolicyCommands.createInitialRevision(tenantId)` (F-SET-05 port) with the mandated values; (9) where a first project is requested: `ProjectCommands.create(tenantId, projectName, creator = membership)` (F-PRJ-02 port) → project id, then `GrantCommands.SetProjectGrants(actor = the new Owner membership, target = itself, projectId, grantSet)` **[009]** (the `AdministrationRule` permits an Owner's self-grant); an empty grant set writes no grant row; (10) audit entries; (11) record the outcome. Any exception rolls everything back and answers `workspace.create.failed`. | RF-1.8, RF-3.3, RF-3.7, RF-4.1–4.6, RF-5.4, RF-7.3, RF-7.5, RF-8.1, RNF-1, RNF-4, RNF-6, LC-6, LC-10–LC-15 |
| | `BootstrapAuditEvents` | `WORKSPACE_CREATED{tenantId}`, `FOUNDING_MEMBERSHIP_CREATED{membershipId, version}`, `POLICY_REVISION_CREATED{revision: 1}`, `FIRST_PROJECT_CREATED{projectId}`, `FIRST_GRANT_CREATED{grantSet}`, `WORKSPACE_CREATION_REFUSED{reasonClass}`, `WORKSPACE_CREATION_FAILED`; payload: identity, tenant, project, correlation id, time; no secret. | RF-8.1, RF-8.2 |
| | Ports consumed | `TenantCommands.createTenant` and `TenantPolicyCommands.createInitialRevision` (→ `tenant` context and the F-SET-05 policy record; declared here, stubbed until their plans), `ProjectCommands.create` (→ `project`, F-PRJ-02; declared, stubbed), `MembershipCommands`, `GrantCommands` **[009]**, `TenantOwnershipRepository` **[010]**, `FactorStatusPort` **[003]**, `IdentityFactsPort` **[008]**, `SessionAssertionPort` **[006]**, `ThrottleStore`, `Clock`, `AuditPort` **[001]**, idempotency outcome record **[001 D1]**, `sharedsupport/tx` (transaction and `SET LOCAL`). No port reaches an environment, source, delivery, approval, run or intent table; no email port. | RF-4.5, RF-4.6, RF-7.1, RF-7.5 |
| `composition.onboarding` (infrastructure) | `WorkspaceCreationProperties` | `@ConfigurationProperties` + `@Validated`: `nameMaxLength` (positive, bounded), throttle thresholds, delay steps and restriction windows for the two creation dimensions `WORKSPACE_CREATE_IDENTITY` and `WORKSPACE_CREATE_NETWORK` (shape of plan 001's `LoginPolicy`); any absent or unbounded value fails startup. | RF-1.7, RF-2.4, RNF-7 |
| | Transaction scope adapter | Opens the bootstrap transaction without tenant scope, and after the tenant row is inserted issues `SET LOCAL app.tenant_id = <new id>` so the ownership, membership, policy, project and grant rows pass `WITH CHECK` under `FORCE RLS` (D3); the identity rows read before that are identity-scoped. | RF-4.1, RNF-10 |
| | Problem codes | `workspace.name.invalid` (400, field path), `workspace.create.failed` (500, generic, nothing created), `workspace.create.throttled` (429, generic: no window, no count), `workspace.create.conflict` (409, idempotency body mismatch); reused: `auth.verification-pending` **[001]**, `auth.enrollment.required {reason: OWNER_ADMIN}` **[003]**, `auth.session.invalid` **[006]`. All RFC 9457 with a correlation id. | RF-1.3, RF-1.5, RF-2.4, RF-4.4, RF-6.4, RF-8.3, LC-19 |
| `composition.onboarding` (api) | `WorkspaceController` — `/api/workspaces` (identity session required; no C10) | `GET eligibility` → 200 `{state: ELIGIBLE \| VERIFICATION_PENDING \| ENROLLMENT_REQUIRED \| NOT_PERMITTED \| THROTTLED, identity: {displayName, email}, enrolled: boolean}` · `POST` `{name, firstProject?: {name, author, reviewer, canRun}, idempotencyKey}` → 201 `{tenantId, membershipId, projectId?, enter: {tenantId, projectId?}}` or 400 field problem or 409 `auth.verification-pending` / `auth.enrollment.required` / `workspace.create.conflict` or 429 `workspace.create.throttled` or 500 `workspace.create.failed`. The server never redirects; the destination is the client's guarded switch. | RF-1.2–1.8, RF-2.2, RF-2.4, RF-3, RF-4.4, RF-5.1, RF-6.4, RF-6.5 |

**Cross-context touch points.** `identity` — eligibility facts (`IdentityFactsPort` **[008]**, `FactorStatusPort` **[003]**), the session (`SessionAssertionPort` **[006]**), throttles. `tenant` — tenant record and first policy revision (through ports). `access` — `MembershipCommands`, `GrantCommands`, `TenantOwnershipRepository` **[009/010]**. `project` — `ProjectCommands.create` (F-PRJ-02). `audit` — append-only. `sharedsupport` — `tx`, `contextIds`, `problemcodes`. Nothing reads or writes environment, source, integration, scenario, approval, execution or intent tables.

**Data model note (P12).** This feature owns no table. Every row it causes carries the new `tenant_id` under the tables of `tenant`, `access` **[009/010]** and `project`, written inside the one transaction whose scope is the new tenant from the moment its row exists.

### 2.2 Front end — feature module `src/features/workspace` (register: `workspace / memberships / project`)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/workspaces/new` (N04). Reached from `LandingRouter` **[001]** on the `WORKSPACE_CREATION` decision, from the chooser's "Create workspace" (plan 013), and as the `returnTo` of the enrollment gate **[003]**. The origin (`landing \| chooser`) is held in memory by the entry, never in the route. | RF-1.1, RF-2.8, RF-6.1, RF-6.2 |
| | `useWorkspaceEligibility()`, `useCreateWorkspace()` | Query on entry (`GET eligibility`); mutation with one idempotency key generated per submission and reused on re-submission; no retry; locale header. | RF-1.8, RF-4.4, RF-6.6, LC-14 |
| | `contracts/workspace` (in `src/contracts`) | Request / response / problem types with runtime guards for the two endpoints; the first-project booleans are three independent fields. | RF-3.2, RF-8.3 |
| Entry | `WorkspaceEntry` | On entry: eligibility; `VERIFICATION_PENDING` → the pending state of plan 002; `ENROLLMENT_REQUIRED` → `EnrollmentGateHandler` **[003]** with `returnTo = /workspaces/new`; `NOT_PERMITTED` → the generic failure; `ELIGIBLE` → the page with step 2 marked complete (D-a). | RF-1.3–1.6, LC-2–LC-5 |
| Views | `CreateWorkspacePage` | N04 verbatim: marker, heading, subtitle, journey list, `StepIndicator` **[003]** ("1 · Email verified · 2 · Owner protected · 3 · Workspace"), the read-only "Workspace identity" block from the eligibility answer, the name field, the statements, `FirstProjectBlock`, "No connection required", the action, the enrollment statement, the Back action per origin, Privacy · Terms · Help (S-17), language switch. | RF-2.1–2.3, RF-2.5–2.8, RF-9 |
| | `FirstProjectBlock` | Unticked toggle; when ticked, the project name field and the three unticked `Checkbox` **[003]** controls with their statements; `GrantWarning` while fewer than three are ticked (names what is missing and where to change it); inert-`canRun` statement (plan 009's `AuthorityStatements.inertCanRun`). | RF-3.1–3.6, LC-9–LC-12 |
| | `CreationInProgress` | Disables the action and shows the in-progress state until the answer; success is shown only on 201. | RF-2.9 |
| State | TanStack Query | The eligibility query (invalidated after the enrollment return) and the creation mutation; no Zustand (no draft, Q6). | RF-6.3 |
| | Origin holder | Module-private in-memory `{origin}` set by the entry; cleared on leave; selects the Back label and destination. | RF-2.8, RF-6.1, RF-6.2 |
| Handoffs | After 201 | `useGuardedSwitch` (plan 016; until built, the scope-epoch sequence of FR-IDN-130 in `shared/security` **[001/006]**) into `enter.tenantId` with `enter.projectId` when present, then navigate to the readiness route of F-PRJ-12 (R09). | RF-5.1–5.3, LC-23, LC-24 |
| | Back | Origin `landing` → `useSignOut()` **[007]** then `/sign-in`; origin `chooser` → `/workspaces` (plan 013). | RF-6.1, RF-6.2, LC-21, LC-22 |
| | Failures | 400 → field paths; `workspace.create.failed` → `GenericFailureMessage` **[001]** "nothing was created", stay on the page, same key on resubmission; `workspace.create.throttled` → generic "try again later"; session end → plan 006's dialog, lookup by key after resume. | RF-6.4–6.6, LC-13, LC-17, LC-19 |
| Shared used | `shared/api` (correlation id, locale, no POST retry), `shared/errors` (`EnrollmentGateHandler` **[003]**, field paths), `shared/i18n` (bundle keys for every N04 string), `shared/ui` (`Field`, `Checkbox`, `StepIndicator`, `Banner`, `LiveRegion`, 44 px targets). | RF-9.2, RF-9.3, RNF-11, RNF-12 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | — | `LandingRouter` **[001]** decision, chooser entry (013), landing → signup (002) |
| RF-1.2 | Session filter **[006]**; no limited-state access | Route behind the authenticated shell |
| RF-1.3 | `WorkspaceCreationEligibilityQuery.VERIFICATION_PENDING` | Handoff to the pending state **[002]** |
| RF-1.4 | `NOT_PERMITTED` (A-27) | Generic failure |
| RF-1.5 | `ENROLLMENT_REQUIRED` from `FactorStatusPort` | `EnrollmentGateHandler` with `returnTo`; "Leave setup" lands on the entry (003 Q20) |
| RF-1.6 | `ELIGIBLE` with `enrolled = true` | Step 2 shown complete; no wizard |
| RF-1.7 | No per-identity count; `ThrottleStore` on the two creation dimensions | — |
| RF-1.8 | Eligibility re-run inside the transaction | Eligibility re-fetched on return from enrollment |
| RF-2.1 | — | `CreateWorkspacePage` copy, `StepIndicator` |
| RF-2.2 | `GET eligibility` identity facts | Read-only block |
| RF-2.3 | Request shape has one name and nothing else | Single field + statement |
| RF-2.4 | `WorkspaceName`, `workspace.name.invalid`; `nameMaxLength` from properties | Field error on 400 |
| RF-2.5 | `firstProject` optional in the request | `FirstProjectBlock` toggle |
| RF-2.6 | — | Statement block |
| RF-2.7 | — | Action and enrollment statement |
| RF-2.8 | — | Origin holder → Back variant; routes; language switch |
| RF-2.9 | 201 only after commit | `CreationInProgress` |
| RF-3.1 | `ProjectName` | Project name field |
| RF-3.2 | — | Three unticked checkboxes and statements |
| RF-3.3 | `FirstProjectRequest.toGrantSet()`; `SetProjectGrants` with any subset | — |
| RF-3.4 | — | `GrantWarning` |
| RF-3.5 | `GrantSet` **[009]** accepts `canRun` alone | Inert statement |
| RF-3.6 | No default in the request contract; no inferred grant | Nothing pre-ticked |
| RF-3.7 | At most one `ProjectCommands.create` call | One block |
| RF-4.1 | `CreateWorkspaceUseCase` steps 4–10 in one transaction | — |
| RF-4.2 | `TenantPolicyCommands.createInitialRevision` with the mandated values | — |
| RF-4.3 | Rollback on any exception; `workspace.create.failed` | Generic failure |
| RF-4.4 | Idempotency outcome **[001 D1]**; `workspace.create.conflict` | One key per submission |
| RF-4.5 | Exactly one `CreateMembership`; no invitation, environment, source or delivery port | — |
| RF-4.6 | No email, target, launch or outbox message port | — |
| RF-5.1 | 201 `enter` scope; derivation on the next request **[009]** | Guarded switch (016) |
| RF-5.2 | — | Navigate to R09 with `projectId` when present |
| RF-5.3 | `MembershipProjectionPort` **[009]** lists the new membership | Chooser (013) |
| RF-5.4 | Use case touches no session, credential, factor or other membership | No replay on resume |
| RF-6.1 | — | `useSignOut` **[007]** → `/sign-in` |
| RF-6.2 | — | Navigate to the chooser |
| RF-6.3 | — | No storage; no Zustand; query cache holds no form value |
| RF-6.4 | `workspace.create.failed` after rollback | `GenericFailureMessage`, resubmission with the same key |
| RF-6.5 | Eligibility problems from the transaction | Handoffs as RF-1.3–1.5 |
| RF-6.6 | Outcome lookup by key | Plan 006 resume → lookup before offering "Create" |
| RF-7.1 | Request contract has no such field | No such field |
| RF-7.2 | Eligibility and session gates | Entries only through the authenticated shell |
| RF-7.3 | One membership, grants from ticks only | — |
| RF-7.4 | Names never compared across tenants; problems carry no other tenant | — |
| RF-7.5 | No target, run, integration or message port | — |
| RF-8.1 | `BootstrapAuditEvents` | — |
| RF-8.2 | Audit payload shape; problem redaction **[001 D12]** | Nothing in storage |
| RF-8.3 | Problem catalogue | Mapped per code |
| RF-9.1 | — | Reflow at tablet and mobile widths |
| RF-9.2 | — | `shared/ui` targets, associations, polite `LiveRegion` |
| RF-9.3 | Locale header; nothing locale-dependent in the bootstrap | i18n bundles |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The bootstrap is one use case in the composition package that orchestrates the published commands of `tenant`, `access` and `project`.**
*Why.* The register (§10) reserves a composition package for onboarding workflows precisely because the bootstrap crosses three contexts in the allowed direction (tenant → access → project) and none of them may call another's repository. Reusing plan 009's `CreateMembership` and `SetProjectGrants` and plan 010's ownership row means the founder's membership and grants are the same records, with the same rules and audit, as any later one.
*Discarded.* A bootstrap inside the `tenant` context — it would import `access` and `project` commands upward, inverting the graph. A bootstrap inside `access` — it would create tenants and projects it does not own. Writing the rows directly — bypasses the rules and versions of plan 009.
*Covers.* RF-4.1, RF-4.5, RF-7.3, RNF-1, RNF-3.

**D2 · One database transaction wraps eligibility re-check, tenant, ownership row, membership, policy revision, project, grant and audit; any failure rolls all of it back.**
*Why.* FR-IDN-115 and RF-4.3: a tenant without an Owner, or an Owner without the policy revision approval will later look for, is a workspace nobody can repair. All the writes are rows in one PostgreSQL database, so one transaction gives atomicity by construction; the eligibility re-check inside it closes the window between entry and submission (RF-1.8).
*Discarded.* A saga with compensation across contexts — compensations for a half-created tenant are exactly the cleanup the specification forbids needing. Creating the tenant first and the project in a second request — a founder could end with a tenant and a lost project.
*Covers.* RF-1.8, RF-4.1–4.3, RF-6.4, RNF-1, RNF-8, LC-13, LC-16.

**D3 · The bootstrap transaction switches its row-level scope to the new tenant immediately after the tenant row exists.**
*Why.* P12: every customer row is written under `FORCE RLS` with `WITH CHECK`, and the runtime role cannot bypass it. The tenant row is created without a scope (it is the scope); from then on `SET LOCAL` names the new tenant so the ownership, membership, policy, project and grant rows are accepted by the same policies every later write obeys — the bootstrap is not an RLS exception.
*Discarded.* A privileged bootstrap role — a `BYPASSRLS` path in the application. Writing the rows and setting the scope afterwards — `WITH CHECK` would refuse them.
*Covers.* RF-4.1, RNF-10.

**D4 · The founder's grants are written by plan 009's `SetProjectGrants` with the founder's new membership as both actor and target.**
*Why.* Decision Q4 and plan 009 Q6: an Owner may set their own grants, visibly and audited; using the same command means the bootstrap grant is a record identical to one made later from the grants editor, with the same version advance, audit entry and inert-`canRun` semantics, and nothing is inferred (FR-IDN-096). An empty set writes nothing, which is the "no grant" of 009 Q2.
*Discarded.* A dedicated "bootstrap grant" insert — a second write path for the same record. Forcing all three ticks — Q4's rejected option.
*Covers.* RF-3.3, RF-3.5, RF-3.6, RF-7.3, RNF-3, LC-10–LC-12.

**D5 · The first policy revision is created through F-SET-05's port with the platform-mandated values; this feature holds no policy value.**
*Why.* Decision Q5 and plan 009 D9: approval and launch read "the current policy revision" and fail closed when it is absent, so the tenant must have one from birth; the values are the ones the requirements mandate (self-approval allowed, Member MFA off, limits by reference), and they belong to the policy record's owner, not to onboarding.
*Discarded.* No policy record until the first edit — the first approval in a new workspace would fail closed. Copying numeric limits into the revision — invented numbers (P13).
*Covers.* RF-4.2, LC-26.

**D6 · Eligibility is one identity-scoped query, run at entry for routing and again inside the transaction for truth.**
*Why.* RF-1.3–1.8: the page must route an unverified or unenrolled person before showing a field, and must still refuse at submission if anything changed; one query serving both keeps the two answers identical. `FactorStatusPort` **[003]** is the same fact the enrollment gate reads, so creation and tenant entry agree on "enrolled".
*Discarded.* Checking only at entry — a factor replaced or a status changed meanwhile would slip through. Checking only at submission — fields shown to an unenrolled founder, contrary to 003 RF-1.2.
*Covers.* RF-1.2–1.8, RF-6.5, LC-2–LC-5, LC-16, LC-18.

**D7 · Creation is counted on two throttle dimensions of the existing throttle store, with no per-identity cap.**
*Why.* Decision Q2 (A-59): no maximum, but a flood must be bounded; the throttle mechanism of plan 001 (thresholds, progressive delay, bounded restriction, generic answer) already exists and is configured, so two new dimension keys give FR-IDN-021 coverage without a cap or a new mechanism.
*Discarded.* A configured ceiling of tenants per identity — Q2's rejected option. No limit at all — unbounded tenant creation by one identity.
*Covers.* RF-1.7, RNF-7, LC-19.

**D8 · Names are value objects with a configured length bound and no uniqueness.**
*Why.* Decision Q3 (A-61): a label, not an identifier; validating length from configuration (P13) and refusing only blank or over-long input keeps enumeration impossible (RF-7.4) and the field simple.
*Discarded.* Global uniqueness — leaks existence. Uniqueness among the founder's tenants — a refusal state N04 does not draw, rejected by Q3.
*Covers.* RF-2.4, RF-3.1, RF-7.4, LC-7, LC-8.

**D9 · Every submission carries an idempotency key with a recorded outcome [001 D1]; a lost acknowledgement is a lookup.**
*Why.* F-X-02 and RF-4.4: a double click or a lost answer must yield one tenant; the outcome record returns the same 201 for the same key and body and a conflict for a different body.
*Discarded.* Natural idempotency by name — names are not unique (D8). No key — two tenants from one double click.
*Covers.* RF-4.4, RF-6.6, RNF-4, LC-14, LC-15, LC-17.

**D10 · The page holds nothing: no draft, no storage, an in-memory origin only.**
*Why.* Decision Q6: enrollment precedes any field, so there is nothing to preserve across the detour, and a name plus three ticks is not worth a draft; the origin that selects the Back variant is a presentation fact held in memory and cleared on leave.
*Discarded.* A draft across the enrollment detour — Q6's rejected option. The origin in the URL — forgeable and visible; it would also survive a reload, contrary to "starts empty".
*Covers.* RF-2.8, RF-6.1–6.3, RNF-9, LC-20–LC-22.

**D11 · The server answers the scope to enter; the client enters it through the guarded switch and lands on the readiness route.**
*Why.* Decision Q5 and P3: the server never redirects; entering a tenant is the guarded state transition of FR-IDN-130 (abort, epoch, fetch permission version, then load), after which `access` derives Owner authority for the enrolled founder on the first request; R09 is the D03 destination.
*Discarded.* Server-side redirect — a redirect is not a guarded switch. Landing on the chooser — Q5's rejected option.
*Covers.* RF-5.1–5.4, LC-23, LC-24.

**D12 · "Back" is two actions chosen by the origin: sign-out for a founder with no membership, the chooser otherwise.**
*Why.* Decision Q6 (S-54): a person with no membership has nowhere to go but the sign-in screen, and leaving them signed in on an empty shell is a dangling session; a person who came from the chooser keeps their session and returns to it. Sign-out is plan 007's use case, not a second one.
*Discarded.* Always sign out — Q6's rejected option. A plain navigation to R02 while signed in — S-26 territory, and an identity-less shell.
*Covers.* RF-6.1, RF-6.2, LC-21, LC-22.

**D13 · The first project is created through F-PRJ-02's command, consumed as a port and stubbed until that plan exists.**
*Why.* RF-3.7 and the register: the project record, its lifecycle and its defaults belong to the `project` context; the bootstrap only asks for one project with a name and a creator. A port keeps this plan buildable and testable before F-PRJ-02 is planned, with a stub that records the call.
*Discarded.* Inserting a project row here — a foreign write. Deferring the first project to R09 only — the designed toggle would disappear.
*Covers.* RF-3.1, RF-3.7, RF-4.1, LC-24.

**D14 · Errors are RFC 9457 problems with four new codes and three reused ones [001 D12]; the throttled and failed answers are generic.**
*Why.* F-X-03; `workspace.create.failed` says only "nothing was created"; `workspace.create.throttled` names no window or count (LC-19); the reused codes let plans 001, 003 and 006's handlers act unchanged.
*Discarded.* A cap message — no cap exists. A reason on the throttled answer — abuse feedback.
*Covers.* RF-6.4, RF-6.5, RF-8.3, LC-19.

**D15 · No dedicated mobile page; N04 reflows; text from the feature's i18n bundle [002 D16].**
*Why.* S-02; FR-X-055 requires AA reflow regardless; one bundle keeps the three languages in parity; the bootstrap reads nothing locale-dependent, so language cannot change what is created.
*Discarded.* Separate mobile components — drift.
*Covers.* RF-9.1–9.3, RNF-11, RNF-12, LC-25.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**) | `WorkspaceName` / `ProjectName` rules, `FirstProjectRequest.toGrantSet()`, eligibility outcome mapping, properties cross-field validation. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A; Testcontainers **test-only**) | The bootstrap through real adapters of `tenant`, `access` **[009/010]** and `audit` with the project and policy ports stubbed: atomicity under a forced failure after each step, RLS acceptance of every row under the runtime role, policy revision values, idempotency, throttles, audit rows. |
| Module boundary | ArchUnit (**test-only**) | The composition package imports only published ports; no context imports the composition package; the direction tenant → access → project holds. |
| API contract | Spring MockMvc (**test-only**) | The two endpoints, the problem codes, the request contract (no contact or integration field), 201 only after commit, no redirect. |
| Concurrency | JUnit 5 with real PostgreSQL | Double submit with one key; two submits with the same key and different bodies; eligibility changed between entry and submission. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| FE unit | Vitest + Testing Library (**test-only**) | Entry routing per eligibility state, page copy, toggle and checkboxes, `GrantWarning` rule, one key per submission, origin-based Back, nothing in storage, in-progress state. |
| FE e2e + accessibility | Playwright + axe (**test-only**) | Journeys in EN / ES / FR at desktop, tablet and mobile widths against the BE doubles; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS, iOS) — pinned versions | Step indicator, field associations, the warning announced politely, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, LC-1 | FE e2e | `entry.landingSignupSignInThenCreation`; `entry.chooserCreateWorkspace` | 3 |
| RF-1.2, RF-7.2 | API contract | `create.requiresCompleteSession`; `create.unreachableFromLimitedState` | 3, 8 |
| RF-1.3, LC-5 | BE integration, FE unit | `eligibility.unverifiedRoutesToPending` | 3 |
| RF-1.4, LC-18 | BE integration | `eligibility.restrictedOrErasingNotPermitted` | 3 |
| RF-1.5, LC-2, LC-3 | BE integration, FE unit | `eligibility.unenrolledRequiresEnrollment`; `entry.enrollmentReturnsToCreationEmpty`; `entry.leaveSetupLandsOnEntryNothingCreated` | 3 |
| RF-1.6, LC-4 | FE unit | `entry.enrolledOpensFormStepTwoComplete` | 3 |
| RF-1.7, LC-19 | BE integration | `create.noPerIdentityCap`; `create.throttledGenericOnBothDimensions` | 3, 11 |
| RF-1.8, RF-6.5, LC-16 | Concurrency | `create.preconditionRecheckedInsideTransaction` | 3 |
| RF-2.1, RF-2.2, RF-2.3, RF-2.5–2.8 | FE unit | `page.rendersN04CopyVerbatim`; `page.identityBlockReadOnly`; `page.singleNameFieldNoContactFields`; `page.backVariantPerOrigin` | 4 |
| RF-2.4, RF-3.1, LC-7, LC-8 | BE unit, API contract | `name.trimmedBoundedPrintableNotUnique`; `name.invalidFieldProblem` | 4 |
| RF-2.9 | FE unit, API contract | `page.inProgressUntilAnswer`; `create.201OnlyAfterCommit` | 4 |
| RF-3.2, RF-3.6 | FE unit | `grants.nothingPreTicked` | 5 |
| RF-3.3, LC-10, LC-12 | BE integration | `create.anySubsetBecomesOneGrantRecord`; `create.noTickNoGrantRow` | 5 |
| RF-3.4 | FE unit | `grantWarning.namesMissingAbilities`; `grantWarning.neverBlocks` | 5 |
| RF-3.5, LC-11 | BE integration, FE unit | `create.canRunAloneStoredInert`; `grants.inertStatementShown` | 5 |
| RF-3.7 | BE integration | `create.atMostOneProject` | 5 |
| RF-4.1, RNF-1, LC-13 | BE integration | `bootstrap.oneTransactionAllRows`; `bootstrap.rollbackAfterEachStep` (one case per step 4–10) | 6 |
| RF-4.2, LC-26 | BE integration | `bootstrap.policyRevisionOneMandatedValues` | 6 |
| RF-4.3, RF-6.4 | BE integration, FE unit | `bootstrap.failureCreatesNothingGenericProblem`; `page.failureKeepsFormSameKey` | 6 |
| RF-4.4, RNF-4, LC-14, LC-15 | Concurrency | `create.doubleSubmitOneTenant`; `create.sameKeyDifferentBodyConflict` | 6 |
| RF-4.5, RF-7.3 | BE integration | `bootstrap.exactlyOneOwnerMembershipNoInvitation` | 6, 8 |
| RF-4.6, RF-7.5, RNF-6 | BE integration | `bootstrap.noTargetRunMessageOrIntegrationPortCalled` | 8 |
| RF-5.1, RNF-10 | BE integration, FE unit | `afterCreate.ownerAuthorityDerivedOnFirstRequest`; `afterCreate.entersThroughGuardedSwitch` | 7, 12 |
| RF-5.2, LC-23, LC-24 | FE e2e | `afterCreate.landsOnReadinessWithProject`; `afterCreate.landsOnReadinessWithoutProject` | 7 |
| RF-5.3 | BE integration | `afterCreate.chooserListsNewOwnerMembership` | 7 |
| RF-5.4 | BE integration | `bootstrap.touchesNoSessionFactorOrOtherMembership` | 7 |
| RF-6.1, LC-21 | FE unit | `back.noMembershipSignsOut` (plan 007 stub called once) | 4 |
| RF-6.2, LC-22 | FE unit | `back.fromChooserReturnsSessionIntact` | 4 |
| RF-6.3, RNF-9, LC-20 | FE unit | `storage.containsNothingFromFeature`; `reload.formEmpty` | 8 |
| RF-6.6, LC-17 | FE unit | `sessionEnd.lookupBeforeOfferingCreateAgain` | 6 |
| RF-7.1 | API contract | `contract.noContactOrIntegrationField` | 8 |
| RF-7.4 | API contract | `problems.discloseNoOtherTenantOrIdentity` | 8 |
| RF-8.1 | BE integration | `audit.entryPerCreationRefusalAndFailure` | 9 |
| RF-8.2, RF-8.3 | API contract | `audit.containsNoSecret`; `problems.safeCodesWithCorrelationId` | 9 |
| RF-9.1, RF-9.2, RNF-11, RNF-12 | FE e2e + axe, manual | `a11y.createWorkspace.{en,es,fr}.{desktop,tablet,mobile}`; manual checklist | 10 |
| RF-9.3, LC-25 | FE e2e | `languageSwitch.changesDisplayOnly` | 10 |
| RNF-2 | BE integration | `bootstrap.neverWithoutVerifiedEnrolledOwner` | 3 |
| RNF-3 | BE integration | `bootstrap.grantsAreTickedRecordsOnly` | 5 |
| RNF-5 | API contract | `contract.noContactOrIntegrationField` | 8 |
| RNF-7 | Startup | `workspaceCreation.startupFailsOnAbsentOrUnboundedValue` | 11 |
| RNF-8 | BE integration | `failClosed.anyRefusedPreconditionCreatesNothing` | 8 |
| LC-6 | BE integration | `create.memberOfOtherTenantsFoundsThird` | 3 |
| LC-9 | BE integration | `create.toggleOffIgnoresProjectName` | 5 |
| US-11 | all adversarial cases above | unverified, unenrolled, connector, target contact, inferred grant, second Owner, double tenant, enumeration suites | 1 |

### 5.3 Not tested here
Signup and verification (002), enrollment (003), the session and its dialog (006), sign-out (007), the records and rules of memberships and grants (009), the ownership row and last-Owner guard (010), the chooser (013), the guarded switch (016), project creation and defaults (F-PRJ-02), the readiness checklist (F-PRJ-12), the policy record (F-SET-05). Their stubs return fixed answers or record calls; `TenantCommands`, `TenantPolicyCommands` and `ProjectCommands` are recording stubs until their plans exist.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q2–Q6 were recorded in Figma on 27 October 2026 as **A-59**, **A-60**, **A-61**, **S-53** and **S-54**; this plan implements each as decided (eligibility and throttles D6 / D7, bootstrap record set and destination D1 / D2 / D5 / D11, names D8, grant warning in `FirstProjectBlock`, Back variant D12). The items below remain open.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-11 configuration values | `WorkspaceCreationProperties`: `nameMaxLength` and the two creation-throttle dimensions; the first policy revision's limits by reference. | Rename or regroup; no use case changes. |
| A-27 identity statuses | Eligibility requires `ACTIVE` with a verified email; `UNVERIFIED` → pending; `RESTRICTED` / `ERASING` → not permitted. | Change the predicate only. |
| A-35 persistence tooling | Transaction and `SET LOCAL` through the placeholder access API. | Rename base types. |
| A-36 identity-scoped membership listing | The chooser's "Create workspace" entry and the landing rule use plan 009's `MembershipProjectionPort`. | None here. |
| F-PRJ-02 / F-SET-05 not yet planned | `ProjectCommands.create` and `TenantPolicyCommands.createInitialRevision` declared here as consumed ports with recording stubs. | Bind the ports to the real commands; the use case is unchanged. |
| S-02 responsive coverage | N04 follows the D01 responsive policy (`414:2645`). | None. |
| S-04 destinations (D02 `439:4460`) | N22 reached from R09 / R14 is F-PRJ-02's; N04's first project needs no dialog. | None here. |
| S-14 templates | No message is sent. | None here. |
| S-17 "Help" | The route opens N43 HelpShortcutsPanel (`430:4129`), as plans 001–011 do. | None: the destination is designed. |
| S-34 "Leave setup" destination | The enrollment return lands on `/workspaces/new` with the form empty. | None here. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.8) | `WorkspaceCreationEligibilityQuery`, `WorkspaceEntry`, `LandingRouter` **[001]**, `EnrollmentGateHandler` **[003]**, throttles | D6, D7 | RF-1 rows |
| RF-2 (2.1–2.9) | `CreateWorkspacePage`, `WorkspaceName`, properties, `CreationInProgress`, origin holder | D8, D10 | RF-2 rows |
| RF-3 (3.1–3.7) | `FirstProjectBlock`, `GrantWarning`, `FirstProjectRequest`, `SetProjectGrants` **[009]**, `ProjectCommands` | D4, D13 | RF-3 rows |
| RF-4 (4.1–4.6) | `CreateWorkspaceUseCase`, transaction scope adapter, `TenantPolicyCommands`, idempotency outcomes | D1, D2, D3, D5, D9 | RF-4 rows |
| RF-5 (5.1–5.4) | 201 `enter`, guarded switch, readiness route, `MembershipProjectionPort` **[009]** | D11 | RF-5 rows |
| RF-6 (6.1–6.6) | Back handoffs, `useSignOut` **[007]**, no-draft state, problem codes, lookup on resume | D9, D10, D12, D14 | RF-6 rows |
| RF-7 (7.1–7.5) | request contract, eligibility gates, port set, name rule | D1, D6, D8 | RF-7 rows |
| RF-8 (8.1–8.3) | `BootstrapAuditEvents`, problem catalogue | D14 | RF-8 rows |
| RF-9 (9.1–9.3) | reflow, `shared/ui`, i18n | D15 | RF-9 rows |
| RNF-1 | one transaction | D2 | `bootstrap.oneTransactionAllRows` |
| RNF-2 | eligibility in the transaction | D6 | `bootstrap.neverWithoutVerifiedEnrolledOwner` |
| RNF-3 | `SetProjectGrants` with ticked set only | D4 | `bootstrap.grantsAreTickedRecordsOnly` |
| RNF-4 | idempotency outcomes | D9 | concurrency rows |
| RNF-5 | request contract | D8 | `contract.noContactOrIntegrationField` |
| RNF-6 | port set | D1 | `bootstrap.noTargetRunMessageOrIntegrationPortCalled` |
| RNF-7 | `WorkspaceCreationProperties` | D7, D8 | startup |
| RNF-8 | rollback, eligibility problems | D2, D6 | `failClosed…` |
| RNF-9 | no draft | D10 | `storage.containsNothingFromFeature` |
| RNF-10 | transaction scope under RLS; derivation after | D3, D11 | RF-5.1 row |
| RNF-11, RNF-12 | `shared/ui`, reflow, i18n | D15 | a11y rows |
| LC-1 … LC-26 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26) |

Self-check: every RF criterion (51), every RNF (12) and every limit case (26) of `specWorkspaceCreationBootstrap.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
