# Tasks 012 — Workspace creation and bootstrap

**Feature** Creating a tenant workspace as its founding Owner, with an optional first project and explicit grants, in one auditable bootstrap · **Tasks version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Derived from** `specWorkspaceCreationBootstrap.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n; decisions Q1–Q6; derived rules D-a, D-b) and `planWorkspaceCreationBootstrap.md` v0.1 (modules §2, trace §3, decisions D1–D15, tests §5, register items §6) · **Governed by** `constitution.md` v1.8.

This document is the DO. Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2 so each task can be checked against the test matrix. Tasks are sized at 20–30 minutes for one developer and listed in dependency order; "after" names the tasks that must be complete first when they are not simply the preceding ones. A later task may assume everything above it. Phase F (front end) depends only on phase A and may run alongside phases B–E.

Elements tagged **[001]**, **[002]**, **[003]**, **[006]**, **[007]**, **[008]**, **[009]** or **[010]** are owned by those plans and are consumed or extended here, never duplicated. This feature owns no table and no aggregate: the bootstrap orchestrates the published commands of the `tenant`, `access` and `project` contexts from the composition package. The project-creation command (F-PRJ-02) and the policy-revision command (F-SET-05) are consumed through ports declared in T-03 with recording stubs until their plans exist. Test-only tools (plan §5.1; none enters the product build): JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing.

---

## A · Foundations

- [ ] **T-01 · Workspace-creation configuration record** — `WorkspaceCreationProperties` (`@ConfigurationProperties` + `@Validated`): `nameMaxLength` (positive, bounded by a configured ceiling) and, for the two creation-throttle dimensions `WORKSPACE_CREATE_IDENTITY` and `WORKSPACE_CREATE_NETWORK`, thresholds, delay steps and restriction windows in the shape of plan 001's `LoginPolicy`; every value present, positive and bounded. No policy value, no per-identity cap. *(plan §2.1 `WorkspaceCreationProperties`, D7, D8; RF-1.7, RF-2.4, RNF-7)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed, set non-positive or set above its ceiling (one failing case per property).

- [ ] **T-02 · Problem-code catalogue** — register `workspace.name.invalid` (400, field path), `workspace.create.failed` (500, generic "nothing was created"), `workspace.create.throttled` (429, generic: no window, no count), `workspace.create.conflict` (409); reuse `auth.verification-pending` **[001]**, `auth.enrollment.required {reason: OWNER_ADMIN}` **[003]**, `auth.session.invalid` **[006]**. Each carries code, title, field paths where applicable, correlation id and nothing else. *(plan §2.1 Problem codes, D14; RF-1.3, RF-1.5, RF-2.4, RF-4.4, RF-6.4, RF-8.3, LC-19)* — after T-01
  **Done when:** a serialisation test renders one problem per code and finds only the permitted members; the throttled and failed problems carry no number, window or tenant data.

- [ ] **T-03 · Consumed port declarations and recording stubs** — declare `TenantCommands.createTenant(name) → tenantId`, `TenantPolicyCommands.createInitialRevision(tenantId)` (F-SET-05) and `ProjectCommands.create(tenantId, name, creatorMembership) → projectId` (F-PRJ-02) as ports of the composition package with recording stubs; list the reused ports (`MembershipCommands`, `GrantCommands` **[009]**, `TenantOwnershipRepository` **[010]**, `FactorStatusPort` **[003]**, `IdentityFactsPort` **[008]**, `SessionAssertionPort` **[006]**, `ThrottleStore`, `Clock`, `AuditPort` **[001]**, idempotency outcome record **[001 D1]**, `sharedsupport/tx`). No port reaches an environment, source, integration, approval, run or intent table; no email port. *(plan §2.1 Ports consumed, D1, D5, D13; RF-3.7, RF-4.2, RF-4.5, RF-4.6, RF-7.5, RNF-6)* — after T-01
  **Done when:** an ArchUnit test shows the composition package imports only published ports, that no context imports the composition package, that the direction tenant → access → project holds, and that no email, environment, source, integration, approval or execution port is referenced.

- [ ] **T-04 · Creation throttle dimensions on the throttle store [001]** — register `WORKSPACE_CREATE_IDENTITY` and `WORKSPACE_CREATE_NETWORK` as dimension keys fed by the values of T-01; progressive delay then bounded restriction; the restriction lifts when its window elapses; independent of the sign-in and reset dimensions. *(plan §2.1 `WorkspaceCreationEligibilityQuery`, D7; RF-1.7, LC-19)* — after T-01
  **Done when:** a unit test shows delay and restriction computed from the creation values, automatic lift after the window, and that a creation flood leaves the sign-in dimensions untouched.

## B · Domain (pure, no Spring)

- [ ] **T-05 · `WorkspaceName` and `ProjectName`** — value objects: trimmed, non-empty, printable text, length ≤ `nameMaxLength`; no uniqueness, no normalisation beyond trimming, no reserved words; the same rule for both. *(plan §2.1 `WorkspaceName`, `ProjectName`, D8; RF-2.4, RF-3.1, RF-7.4, LC-7, LC-8)* — after T-01
  **Done when:** `name.trimmedBoundedPrintableNotUnique` passes: surrounding spaces are trimmed, a blank or over-long name is rejected, two equal names are both accepted, and no comparison against other tenants exists.

- [ ] **T-06 · `FirstProjectRequest`** — value: `ProjectName` plus three booleans; `toGrantSet()` returns plan 009's `GrantSet` with any subset, `canRun` allowed alone; `isEmpty()` when nothing is ticked. *(plan §2.1 `FirstProjectRequest`, D4; RF-3.3, RF-3.5, RF-3.6, LC-10, LC-11, LC-12)* — after T-05
  **Done when:** unit tests show the eight subsets map to the expected `GrantSet`, `{canRun}` alone yields a set that is not `eligibleToRun()`, and no default tick exists in the type.

- [ ] **T-07 · Eligibility outcome** — enumeration `ELIGIBLE | VERIFICATION_PENDING | ENROLLMENT_REQUIRED | NOT_PERMITTED | THROTTLED` with the pure rule `(identityFacts, enrolled, throttled) → outcome` in the order status → verification → enrollment → throttle (A-27: `ACTIVE` with a verified email; `UNVERIFIED` → pending; `RESTRICTED` / `ERASING` → not permitted). *(plan §2.1 `WorkspaceCreationEligibilityQuery`, D6; RF-1.3, RF-1.4, RF-1.6)* — after T-01
  **Done when:** a table test covers every status × verified × enrolled × throttled combination and the outcome order is the one stated.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-15…T-18); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-08 · `WorkspaceCreationEligibilityQuery`** — for the session's identity (`SessionAssertionPort` **[006]**): facts from `IdentityFactsPort` **[008]**, `FactorStatusPort.hasActiveFactor` **[003]**, the creation throttles **[001]** → the outcome of T-07 plus the display name and verified email for the read-only block; no tenant is read. *(plan §2.1 `WorkspaceCreationEligibilityQuery`, D6; RF-1.2, RF-1.3, RF-1.4, RF-1.6, RF-1.7, RF-2.2, LC-4, LC-5, LC-18)* — after T-07, T-04, T-03
  **Done when:** `eligibility.unverifiedRoutesToPending`, `eligibility.restrictedOrErasingNotPermitted`, `eligibility.unenrolledRequiresEnrollment` and `create.noPerIdentityCap` (the query is eligible for an identity holding memberships elsewhere) pass.

- [ ] **T-09 · `CreateWorkspaceUseCase` — steps 1–3: idempotency, recheck, throttle** — open the transaction; return the recorded outcome for a repeated idempotency key, `workspace.create.conflict` for the same key with a different body; re-run the eligibility query inside the transaction and abort with the matching problem on any non-eligible outcome; count the attempt on both creation dimensions. *(plan §2.1 `CreateWorkspaceUseCase`, D2, D6, D7, D9; RF-1.8, RF-4.4, RF-6.5, RNF-4, LC-14, LC-15, LC-16)* — after T-08
  **Done when:** `create.preconditionRecheckedInsideTransaction` (a factor removed or a status changed between entry and submission aborts with nothing created) and `create.sameKeyDifferentBodyConflict` pass at the use-case level.

- [ ] **T-10 · `CreateWorkspaceUseCase` — steps 4–7: tenant, scope, ownership row, Owner membership** — `TenantCommands.createTenant(name)`; set the transaction scope to the new tenant (T-15); `TenantOwnershipRepository.create(tenantId)` **[010]**; `MembershipCommands.CreateMembership(tenantId, identityId, OWNER, source = BOOTSTRAP)` **[009]** → membership at version 1; exactly one membership and one Owner. *(plan §2.1 `CreateWorkspaceUseCase`, D1, D3; RF-4.1, RF-4.5, RF-7.3, RNF-10)* — after T-09
  **Done when:** `bootstrap.exactlyOneOwnerMembershipNoInvitation` passes and the recording stubs show the four calls in this order inside one transaction.

- [ ] **T-11 · `CreateWorkspaceUseCase` — step 8: first policy revision** — `TenantPolicyCommands.createInitialRevision(tenantId)` with the mandated values: self-approval allowed, MFA for Members not required, retention and limits by reference to the configured platform policy; this feature passes no numeric value. *(plan §2.1 `CreateWorkspaceUseCase`, D5; RF-4.2, LC-26)* — after T-10
  **Done when:** `bootstrap.policyRevisionOneMandatedValues` passes and a static check shows the call carries no numeric literal.

- [ ] **T-12 · `CreateWorkspaceUseCase` — step 9: first project and grants** — where `firstProject` is present: `ProjectCommands.create(tenantId, projectName, creator = the new membership)`; then `GrantCommands.SetProjectGrants(actor = new Owner membership, target = itself, projectId, toGrantSet())` **[009]** unless the set is empty, in which case no grant row is written; at most one project; the toggle off ignores any project name. *(plan §2.1 `CreateWorkspaceUseCase`, D4, D13; RF-3.3, RF-3.5, RF-3.7, RF-7.3, LC-9, LC-10, LC-11, LC-12)* — after T-11
  **Done when:** `create.anySubsetBecomesOneGrantRecord`, `create.noTickNoGrantRow`, `create.canRunAloneStoredInert`, `create.atMostOneProject` and `create.toggleOffIgnoresProjectName` pass.

- [ ] **T-13 · `CreateWorkspaceUseCase` — steps 10–11: audit, outcome, rollback** — append `WORKSPACE_CREATED`, `FOUNDING_MEMBERSHIP_CREATED`, `POLICY_REVISION_CREATED`, `FIRST_PROJECT_CREATED` and `FIRST_GRANT_CREATED` as applicable; record the idempotency outcome; answer `{tenantId, membershipId, projectId?, enter}`; any exception rolls everything back, appends `WORKSPACE_CREATION_FAILED` outside the rolled-back transaction, and answers `workspace.create.failed`; a refused precondition appends `WORKSPACE_CREATION_REFUSED{reasonClass}`. *(plan §2.1 `CreateWorkspaceUseCase`, `BootstrapAuditEvents`, D2, D9; RF-4.3, RF-6.4, RF-8.1, RNF-1, RNF-8, LC-13)* — after T-12
  **Done when:** `bootstrap.failureCreatesNothingGenericProblem` and `failClosed.anyRefusedPreconditionCreatesNothing` pass with in-memory doubles, and the success answer carries the four ids and `enter`.

- [ ] **T-14 · Absence guarantees of the use case** — the use case holds no reference to an email, environment, source, integration, approval, execution or intent port; it never touches the session, credential, factor, recovery codes or other memberships of the founder; it sends nothing to the outbox but audit. *(plan §2.1 Ports consumed, D1; RF-4.6, RF-5.4, RF-7.5, RNF-6)* — after T-13
  **Done when:** `bootstrap.noTargetRunMessageOrIntegrationPortCalled` and `bootstrap.touchesNoSessionFactorOrOtherMembership` pass with recording doubles for every reused port.

## D · Infrastructure adapters

- [ ] **T-15 · Transaction scope adapter** — opens the bootstrap transaction without tenant scope for the identity-scoped reads, and after the tenant row exists issues `SET LOCAL app.tenant_id = <new id>` (through `sharedsupport/tx`) so the ownership, membership, policy, project and grant rows pass `WITH CHECK` under `FORCE RLS` with the non-privileged runtime role. *(plan §2.1 Transaction scope adapter, D3; RF-4.1, RNF-10)* — after T-03
  **Done when:** against real PostgreSQL under the runtime role, every bootstrap row is accepted after the scope switch and refused before it, and no `BYPASSRLS` role is involved.

- [ ] **T-16 · Properties binding and startup validation wiring** — bind `WorkspaceCreationProperties` into the eligibility query, the name value objects and the throttle dimensions; the context fails to start on any violation. *(plan §2.1 `WorkspaceCreationProperties`, D7, D8; RF-2.4, RNF-7)* — after T-01, T-08
  **Done when:** `workspaceCreation.startupFailsOnAbsentOrUnboundedValue` passes with one case per property and the complete set starts.

- [ ] **T-17 · Audit adapter** — map the seven events of `BootstrapAuditEvents` to `AuditPort` with identity, tenant, membership, project, revision, grant set, reason class, correlation id and time; successes appended in the bootstrap transaction; a serialisation guard rejects any secret or content. *(plan §2.1 `BootstrapAuditEvents`; RF-8.1, RF-8.2)* — after T-13
  **Done when:** `audit.entryPerCreationRefusalAndFailure` and `audit.containsNoSecret` pass.

- [ ] **T-18 · Logging and problem redaction** — request and response logging of the two endpoints exclude nothing secret by construction (no secret is sent) but never echo another tenant's data; problem details never name an existing tenant or identity; a log-appender test asserts the absence. *(plan §2.1 Problem codes, D14; RF-7.4, RF-8.2, RF-8.3)* — after T-17
  **Done when:** `problems.discloseNoOtherTenantOrIdentity` passes with a second tenant of the same name present.

## E · API

- [ ] **T-19 · `GET /api/workspaces/eligibility`** — identity session required, no C10; 200 `{state, identity: {displayName, email}, enrolled}`; unreachable without a complete session or from a limited identity state. *(plan §2.1 `WorkspaceController`, D6; RF-1.2, RF-1.3, RF-1.4, RF-1.5, RF-1.6, RF-2.2)* — after T-08, T-16
  **Done when:** `create.requiresCompleteSession` and `create.unreachableFromLimitedState` pass for this endpoint and the 200 body matches the five outcomes of T-07.

- [ ] **T-20 · `POST /api/workspaces`** — `{name, firstProject?: {name, author, reviewer, canRun}, idempotencyKey}` → 201 `{tenantId, membershipId, projectId?, enter: {tenantId, projectId?}}` only after commit; 400 `workspace.name.invalid` with the field path; 409 `auth.verification-pending` / `auth.enrollment.required` / `workspace.create.conflict`; 429 `workspace.create.throttled`; 500 `workspace.create.failed`; no redirect; no other field accepted. *(plan §2.1 `WorkspaceController`, D9, D11, D14; RF-2.3, RF-2.4, RF-2.9, RF-3.2, RF-4.4, RF-5.1, RF-6.4, RF-6.5, RF-7.1)* — after T-13, T-15, T-16
  **Done when:** `name.invalidFieldProblem`, `create.201OnlyAfterCommit` (a forced failure after the project step answers 500 and no row exists) and `contract.noContactOrIntegrationField` (an extra `address`, `phone` or `connector` member is rejected) pass.

- [ ] **T-21 · Contract and absence checks** — a test enumerating the surface: the problem codes with correlation id; the 201 body carries no policy value, no other identity and no tenant data beyond the new ids; a throttled answer is byte-identical across identities and networks; no endpoint of this feature is reachable without a complete session. *(plan §3 RF-7, RF-8; RF-1.2, RF-7.2, RF-7.4, RF-8.3, RNF-5)* — after T-19, T-20
  **Done when:** `problems.safeCodesWithCorrelationId` and `create.throttledGenericOnBothDimensions` pass across the API surface.

## F · Front end

- [ ] **T-22 · `contracts/workspace` types and guards** — request / response / problem types for the two endpoints with runtime guards; the first-project booleans are three independent fields with no default; a guard rejects a request carrying any field beyond name, first project and key. *(plan §2.2 `contracts/workspace`; RF-3.2, RF-7.1, RF-8.3)*
  **Done when:** guard unit tests accept one fixture per documented answer and problem, and reject a request with an extra field and a response with a `redirect` member.

- [ ] **T-23 · i18n bundle `workspace.{en,es,fr}`** — every string of N04 verbatim (marker, heading, subtitle, journey list, step names, block headings, statements, field labels, checkbox labels, action, enrollment statement, both Back labels, routes), the grant warning texts, the generic failure "nothing was created", the throttled text. *(plan §2.2 Shared used, D15; RF-2.1, RF-2.3, RF-2.6, RF-2.7, RF-2.8, RF-3.2, RF-3.4, RF-9.3, RNF-12)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value, and the EN values of RF-2.1, RF-2.3, RF-2.6, RF-2.7 and RF-3.2 match the specification verbatim.

- [ ] **T-24 · Hooks** — `useWorkspaceEligibility()` (query, refetched on return from enrollment) and `useCreateWorkspace()` (mutation, retry off, locale header, one idempotency key generated per submission and reused on re-submission; after a session end the outcome is looked up by key before "Create" is offered again). *(plan §2.2 hooks, D9; RF-1.8, RF-4.4, RF-6.6, LC-14, LC-17)* — after T-22
  **Done when:** unit tests show retry disabled, the same key on a re-submission after failure, a fresh key on a new submission, and `sessionEnd.lookupBeforeOfferingCreateAgain` passes.

- [ ] **T-25 · Routes and origin holder** — `/workspaces/new` registered in the `workspace` feature's public API; module-private in-memory `{origin: landing | chooser}` set by the entry (`LandingRouter` **[001]** decision `WORKSPACE_CREATION`, the chooser's "Create workspace", the enrollment `returnTo` **[003]**) and cleared on leave; no origin in the route; nothing in storage; no Zustand. *(plan §2.2 routes, Origin holder, D10; RF-1.1, RF-2.8, RF-6.3, RNF-9, LC-20)* — after T-24
  **Done when:** `storage.containsNothingFromFeature` (storage spies on every mechanism) and `reload.formEmpty` pass, and the route carries no query or path parameter.

- [ ] **T-26 · `WorkspaceEntry`** — on entry: eligibility; `VERIFICATION_PENDING` → the pending state of plan 002; `ENROLLMENT_REQUIRED` → `EnrollmentGateHandler` **[003]** with `returnTo = /workspaces/new`; `NOT_PERMITTED` → the generic failure; `THROTTLED` → the throttled text; `ELIGIBLE` → the page with step 2 marked complete (D-a); on return from enrollment the eligibility is refetched and the form is empty. *(plan §2.2 `WorkspaceEntry`, D6; RF-1.3, RF-1.4, RF-1.5, RF-1.6, LC-2, LC-3, LC-4, LC-5)* — after T-25
  **Done when:** `entry.enrollmentReturnsToCreationEmpty`, `entry.leaveSetupLandsOnEntryNothingCreated` (with plan 003's handler double) and `entry.enrolledOpensFormStepTwoComplete` pass.

- [ ] **T-27 · `CreateWorkspacePage` shell** — N04 copy from the bundle: marker, heading, subtitle, journey list, `StepIndicator` **[003]** with the current step in text, the read-only "Workspace identity" block from the eligibility answer, the "Workspace name" field with its statement, "No connection required", the action "Create workspace and explicit grants", the enrollment statement, Privacy · Terms · Help (S-17), language switch. *(plan §2.2 `CreateWorkspacePage`, D8, D15; RF-2.1, RF-2.2, RF-2.3, RF-2.6, RF-2.7)* — after T-23, T-26
  **Done when:** `page.rendersN04CopyVerbatim`, `page.identityBlockReadOnly` and `page.singleNameFieldNoContactFields` pass.

- [ ] **T-28 · `FirstProjectBlock` and `GrantWarning`** — unticked toggle "Create my first project with this workspace"; when ticked, the "Project name" field, the three unticked checkboxes with their statements and "Run still requires valid approval and environment pins."; `GrantWarning` while fewer than three are ticked, naming what the Owner will not be able to do and that grants can be changed later; the inert-`canRun` statement (plan 009's `AuthorityStatements.inertCanRun`); the warning never disables the action; toggle off clears and ignores the project fields. *(plan §2.2 `FirstProjectBlock`, D4; RF-2.5, RF-3.1, RF-3.2, RF-3.4, RF-3.5, RF-3.6, LC-9, LC-10, LC-11, LC-12)* — after T-27
  **Done when:** `grants.nothingPreTicked`, `grantWarning.namesMissingAbilities`, `grantWarning.neverBlocks` and `grants.inertStatementShown` pass.

- [ ] **T-29 · `CreationInProgress` and failure handling** — disable the action and show the in-progress state until the answer; success only on 201; 400 → field paths; `workspace.create.failed` → `GenericFailureMessage` **[001]** "nothing was created", stay on the page, same key on resubmission; `workspace.create.throttled` → the throttled text; eligibility problems → the handoffs of T-26; session end → plan 006's dialog then the lookup of T-24. *(plan §2.2 `CreationInProgress`, Failures, D9, D14; RF-2.9, RF-6.4, RF-6.5, RF-6.6, LC-13, LC-17, LC-19)* — after T-28
  **Done when:** `page.inProgressUntilAnswer` and `page.failureKeepsFormSameKey` pass, and a mocked 429 renders the throttled text without a number.

- [ ] **T-30 · Back actions** — origin `landing`: "← Back to sign in" → `useSignOut()` **[007]** then `/sign-in`; origin `chooser`: "Back to your workspaces" → the chooser route (plan 013); both clear the origin holder; nothing created or stored. *(plan §2.2 Back, D12; RF-2.8, RF-6.1, RF-6.2, LC-21, LC-22)* — after T-27
  **Done when:** `page.backVariantPerOrigin`, `back.noMembershipSignsOut` (plan 007 stub called once) and `back.fromChooserReturnsSessionIntact` pass.

- [ ] **T-31 · After 201: guarded switch and readiness landing** — on 201 clear the origin holder, run the guarded switch (plan 016; until built, the FR-IDN-130 sequence in `shared/security` **[001/006]**: abort, disconnect, advance epoch, clear scoped cache, fetch `MyAuthority` **[009]**) into `enter.tenantId` with `enter.projectId` when present, then navigate to the readiness route of F-PRJ-12 with the project selected or "Project" as the next step; nothing is replayed. *(plan §2.2 After 201, D11; RF-5.1, RF-5.2, RF-5.4, LC-23, LC-24)* — after T-29
  **Done when:** `afterCreate.entersThroughGuardedSwitch` passes (epoch advanced, authority fetched before any tenant data) and unit tests show the readiness route with and without `projectId`.

- [ ] **T-32 · Reflow and accessibility** — Tailwind breakpoints for N04; `shared/ui` field, checkboxes, toggle and buttons at 44 px minimum on touch widths; the step indicator, field errors and the grant warning as text associated with their field or region; polite `LiveRegion` that never moves focus; every statement present at every width. *(plan §2.2 Shared used, D15; RF-9.1, RF-9.2, RNF-11, RNF-12)* — after T-28, T-30
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow, identical statement text at the three widths, every interactive element ≥ 44 px on the two smaller widths, and the warning announced without a focus change.

## G · Verification suites

- [ ] **T-33 · Bootstrap atomicity suite** — `bootstrap.oneTransactionAllRows` (tenant, ownership row, membership at version 1, policy revision 1, project, grant, audit rows all present after one commit) and `bootstrap.rollbackAfterEachStep` (one case per step 4–10: a forced failure leaves no row of any kind and answers `workspace.create.failed`), against real PostgreSQL with the project and policy stubs. *(plan §5.2 RF-4 rows, D2; RF-4.1, RF-4.3, RNF-1, RNF-8, LC-13)* — after T-15, T-17, T-20
  **Done when:** both tests pass, the rollback test has seven cases and none is skipped.

- [ ] **T-34 · Concurrency and idempotency suite** — `create.doubleSubmitOneTenant` (two concurrent submissions with one key: one tenant, the second answer is the recorded 201), `create.sameKeyDifferentBodyConflict`, `create.preconditionRecheckedInsideTransaction` (factor removed, status changed, session ended between entry and submission). *(plan §5.1 Concurrency, D6, D9; RF-1.8, RF-4.4, RNF-4, LC-14, LC-15, LC-16)* — after T-33
  **Done when:** all three tests pass ten times in a row with real PostgreSQL.

- [ ] **T-35 · Isolation and authority-after suite** — the new tenant's rows are refused to any other tenant's scope and accepted under their own (`FORCE RLS`, runtime role); `afterCreate.ownerAuthorityDerivedOnFirstRequest` (plan 009's derivation yields Owner authority for the enrolled founder on the first tenant request); `afterCreate.chooserListsNewOwnerMembership`; `bootstrap.neverWithoutVerifiedEnrolledOwner`; `create.memberOfOtherTenantsFoundsThird`. *(plan §5.2 RF-5, RNF-2, RNF-10 rows, D3, D11; RF-5.1, RF-5.3, RNF-2, RNF-10, LC-6)* — after T-33
  **Done when:** all five tests pass; a cross-tenant read of any bootstrap row under another tenant's scope returns zero rows.

- [ ] **T-36 · Absence suite** — `bootstrap.exactlyOneOwnerMembershipNoInvitation`, `bootstrap.noTargetRunMessageOrIntegrationPortCalled`, `bootstrap.touchesNoSessionFactorOrOtherMembership`, `bootstrap.grantsAreTickedRecordsOnly`, `contract.noContactOrIntegrationField`, `problems.discloseNoOtherTenantOrIdentity`. *(plan §5.2 RF-7 rows, D1, D4; RF-4.5, RF-4.6, RF-5.4, RF-7.1, RF-7.2, RF-7.3, RF-7.4, RF-7.5, RNF-3, RNF-5, RNF-6)* — after T-21
  **Done when:** all six tests pass with recording doubles for every reused port.

- [ ] **T-37 · Startup suite** — `workspaceCreation.startupFailsOnAbsentOrUnboundedValue` with one case per property of T-01, plus the case where the platform policy reference the first revision needs is absent (the policy port's own startup check). *(plan §5.1 Startup, D7, D8; RNF-7)* — after T-16
  **Done when:** every case fails the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-38 · FE e2e journeys** — Playwright journeys for US-1…US-10 in EN, ES and FR at desktop, tablet and mobile widths against the BE doubles: landing → signup → sign-in → creation (`entry.landingSignupSignInThenCreation`); unenrolled founder through enrollment and back; enrolled founder straight to the form; chooser entry (`entry.chooserCreateWorkspace`) and "Back to your workspaces"; creation with and without a project (`afterCreate.landsOnReadinessWithProject`, `afterCreate.landsOnReadinessWithoutProject`); partial grants with the warning; failure and resubmission; `languageSwitch.changesDisplayOnly`. *(plan §5.1 FE e2e; RF-1.1, RF-5.2, RF-9.3, LC-1, LC-23, LC-24, LC-25, US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8, US-9, US-10)* — after T-32, T-21
  **Done when:** all journeys pass in the three languages and three widths, and a language switch mid-form changes only displayed text.

- [ ] **T-39 · Automated accessibility** — axe on N04 (eligible form, project block open, warning shown, in-progress, failure, throttled) at the three widths and three languages (`a11y.createWorkspace.{en,es,fr}.{desktop,tablet,mobile}`). *(plan §5.1 FE e2e + accessibility; RF-9.1, RF-9.2, RNF-11, RNF-12)* — after T-38
  **Done when:** zero critical or serious violations on every combination.

- [ ] **T-40 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on N04: step indicator announced, field errors associated, the grant warning announced politely without focus theft, checkbox labels, 44 px targets; versions recorded in the release-support matrix. *(plan §5.1 Manual assistive; RNF-11)* — after T-39
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-41 · Adversarial suite (US-11)** — creation by an unverified identity, by a limited identity state, by an unenrolled identity bypassing the entry, with a connector or address field, with a request naming a target, with a pre-ticked or inferred grant, with a second Owner or an invitation in the body, a double submit, and a name probe against an existing tenant. *(plan §5.2 US-11 row, D2, D4, D6, D8, D9; RF-7.2, RF-7.3, RF-7.4, RF-7.5, RNF-2, RNF-3, RNF-4, US-11)* — after T-34, T-36
  **Done when:** every attack test fails to achieve its goal and asserts the safe problem or the structural refusal.

## H · Closing

- [ ] **T-42 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-33…T-41
  **Done when:** the script reports 100 of 100 identifiers (51 RF, 12 RNF, 26 LC, 11 US) covered by at least one passing test.

- [ ] **T-43 · Register-dependency review** — walk plan §6 — A-59, A-60, A-61, S-53 and S-54 as decided, and the open A-11, A-27, A-35, A-36, S-02, S-04, S-14, S-17 and S-34, plus the two ports consumed from unplanned features (F-PRJ-02, F-SET-05): for each, confirm the built assumption matches the current Figma register state or the later plan, or record the deferral. *(spec §7 item 13)* — after T-42
  **Done when:** each of the sixteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-44 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run ids, a11y report, manual checklist, coverage report, register review). — after T-43
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-25, T-38 |
| RF-1.2 | T-08, T-19, T-21 |
| RF-1.3 | T-02, T-07, T-08, T-19, T-26 |
| RF-1.4 | T-07, T-08, T-19, T-26 |
| RF-1.5 | T-02, T-19, T-26 |
| RF-1.6 | T-07, T-08, T-19, T-26 |
| RF-1.7 | T-01, T-04, T-08 |
| RF-1.8 | T-09, T-24, T-34 |
| RF-2.1 | T-23, T-27 |
| RF-2.2 | T-08, T-19, T-27 |
| RF-2.3 | T-20, T-23, T-27 |
| RF-2.4 | T-01, T-02, T-05, T-16, T-20 |
| RF-2.5 | T-28 |
| RF-2.6 | T-23, T-27 |
| RF-2.7 | T-23, T-27 |
| RF-2.8 | T-23, T-25, T-30 |
| RF-2.9 | T-20, T-29 |
| RF-3.1 | T-05, T-28 |
| RF-3.2 | T-20, T-22, T-23, T-28 |
| RF-3.3 | T-06, T-12 |
| RF-3.4 | T-23, T-28 |
| RF-3.5 | T-06, T-12, T-28 |
| RF-3.6 | T-06, T-28 |
| RF-3.7 | T-03, T-12 |
| RF-4.1 | T-10, T-15, T-33 |
| RF-4.2 | T-03, T-11 |
| RF-4.3 | T-13, T-33 |
| RF-4.4 | T-02, T-09, T-20, T-24, T-34 |
| RF-4.5 | T-03, T-10, T-36 |
| RF-4.6 | T-03, T-14, T-36 |
| RF-5.1 | T-20, T-31, T-35 |
| RF-5.2 | T-31, T-38 |
| RF-5.3 | T-35 |
| RF-5.4 | T-14, T-31, T-36 |
| RF-6.1 | T-30 |
| RF-6.2 | T-30 |
| RF-6.3 | T-25 |
| RF-6.4 | T-02, T-13, T-20, T-29 |
| RF-6.5 | T-09, T-20, T-29 |
| RF-6.6 | T-24, T-29 |
| RF-7.1 | T-20, T-22, T-36 |
| RF-7.2 | T-21, T-36, T-41 |
| RF-7.3 | T-10, T-12, T-36, T-41 |
| RF-7.4 | T-05, T-18, T-21, T-36, T-41 |
| RF-7.5 | T-03, T-14, T-36, T-41 |
| RF-8.1 | T-13, T-17 |
| RF-8.2 | T-17, T-18 |
| RF-8.3 | T-02, T-18, T-21, T-22 |
| RF-9.1 | T-32, T-39 |
| RF-9.2 | T-32, T-39 |
| RF-9.3 | T-23, T-38 |
| RNF-1 | T-13, T-33 |
| RNF-2 | T-35, T-35, T-41 |
| RNF-3 | T-36, T-41 |
| RNF-4 | T-09, T-34, T-41 |
| RNF-5 | T-21, T-36 |
| RNF-6 | T-03, T-14, T-36 |
| RNF-7 | T-01, T-16, T-37 |
| RNF-8 | T-13, T-33 |
| RNF-9 | T-25 |
| RNF-10 | T-10, T-15, T-35, T-35 |
| RNF-11 | T-32, T-39, T-40 |
| RNF-12 | T-23, T-32, T-39 |
| LC-1 | T-38 |
| LC-2 | T-26 |
| LC-3 | T-26 |
| LC-4 | T-08, T-26 |
| LC-5 | T-08, T-26 |
| LC-6 | T-35 |
| LC-7 | T-05 |
| LC-8 | T-05 |
| LC-9 | T-12, T-28 |
| LC-10 | T-06, T-12, T-28 |
| LC-11 | T-06, T-12, T-28 |
| LC-12 | T-06, T-12, T-28 |
| LC-13 | T-13, T-29, T-33 |
| LC-14 | T-09, T-24, T-34 |
| LC-15 | T-09, T-34 |
| LC-16 | T-09, T-34 |
| LC-17 | T-24, T-29 |
| LC-18 | T-08 |
| LC-19 | T-02, T-04, T-29 |
| LC-20 | T-25 |
| LC-21 | T-30 |
| LC-22 | T-30 |
| LC-23 | T-31, T-38 |
| LC-24 | T-31, T-38 |
| LC-25 | T-38 |
| LC-26 | T-11 |
| US-1 … US-10 | T-38 (journeys) |
| US-11 | T-41 (adversarial suite) |

Self-check: all 51 RF criteria, 12 RNF, 26 limit cases and 11 user stories of `specWorkspaceCreationBootstrap.md` v0.1 appear above; every task of phases A–G names at least one requirement, and the closing tasks name the completion criterion they serve.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| A-59 creation eligibility and limits (decided, Q2) | Eligibility query for any verified, enrolled, active identity; two creation-throttle dimensions; no cap | T-01, T-04, T-07, T-08, T-09 |
| A-60 bootstrap record set and destination (decided, Q5) | One transaction: tenant, ownership row, Owner membership v1, policy revision 1, optional project and grant, audit; 201 `enter` then guarded switch to R09 | T-10, T-11, T-12, T-13, T-31 |
| A-61 name constraints (decided, Q3) | `WorkspaceName` / `ProjectName`: trimmed, bounded, printable, not unique | T-05, T-20 |
| S-53 partial-grants warning (decided, Q4) | `GrantWarning` in `FirstProjectBlock` | T-28 |
| S-54 N04 "Back" variant from the chooser (decided, Q6) | Origin holder selects label and destination; sign-out for a founder without membership | T-25, T-30 |
| A-11 configuration values | `WorkspaceCreationProperties`; policy limits by reference through F-SET-05's port | T-01, T-11, T-16, T-37 |
| A-27 identity statuses | `ACTIVE` with verified email eligible; `UNVERIFIED` → pending; `RESTRICTED` / `ERASING` → not permitted | T-07, T-08 |
| A-35 persistence tooling | Transaction and `SET LOCAL` through the placeholder access API | T-15 |
| A-36 identity-scoped membership listing | Chooser entry and landing rule use plan 009's projection | T-25 |
| F-PRJ-02 / F-SET-05 not yet planned | `ProjectCommands.create` and `TenantPolicyCommands.createInitialRevision` as consumed ports with recording stubs | T-03, T-11, T-12 |
| S-02 responsive coverage | N04 follows the D01 responsive policy | T-32, T-39 |
| S-04 destinations (D02 `439:4460`) | N22 from R09 / R14 is F-PRJ-02's; the first project needs no dialog | T-28 |
| S-14 templates | No message sent | T-14 |
| S-17 "Help" | Opens N43 HelpShortcutsPanel (`430:4129`) | T-27 |
| S-34 "Leave setup" destination | Enrollment return lands on `/workspaces/new` with the form empty | T-26 |
