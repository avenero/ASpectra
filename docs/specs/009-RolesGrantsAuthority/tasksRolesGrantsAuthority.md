# Tasks 009 — Roles, grants and the authority matrix

**Feature** The finite role model, the authority matrix, membership and tenant statuses, permission versions and revocation · **Tasks version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Derived from** `specRolesGrantsAuthority.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n; decisions Q1–Q7) and `planRolesGrantsAuthority.md` v0.1 (modules §2, trace §3, decisions D1–D16, tests §5, register items §6) · **Governed by** `constitution.md` v1.8.

This document is the DO. Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2 so each task can be checked against the test matrix. Tasks are sized at 20–30 minutes for one developer and listed in dependency order; "after" names the tasks that must be complete first when they are not simply the preceding ones. A later task may assume everything above it. Phase F (front end) depends only on phase A and may run alongside phases B–E.

Elements tagged **[001]**, **[002]**, **[003]**, **[006]** or **[007]** are owned by plans 001 (Login), 002 (Signup and verification), 003 (Enrollment), 006 (Sessions) and 007 (Logout); they are extended here, never duplicated. This feature **implements** the ports those plans declared as "published by `access`" — `MembershipProjectionPort` **[001]**, `RequirementReasonPort` **[003]**, `MembershipExistsPort` **[007]** — and publishes `AuthorityPort`, `MembershipCommands` and `GrantCommands` for plans 008, 010, 011 and 012, whose own tasks call them. If an owning plan is not built yet, the element is built here in the shape that plan gives it. Test-only tools (plan §5.1; none enters the product build): JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plans 001–007's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing.

---

## A · Foundations

- [ ] **T-01 · Closed enumerations and `GrantSet`** — `TenantRole` (`OWNER`, `ADMIN`, `MEMBER`), `ProjectRole` (`AUTHOR`, `REVIEWER`, `VIEWER`), `ActionClass` (the eighteen classes of plan §2.1), and the `GrantSet` value (subset of project roles + `canRun`) with `isEmpty()`, `includesRead()`, `eligibleToRun()` (Author or Reviewer **and** canRun). No other value exists. *(plan §2.1 `TenantRole`, `ProjectRole`, `GrantSet`, `ActionClass`, D2, D5; RF-1.1, RF-1.2, RF-1.3, RF-9.2, RF-9.3, RNF-9)*
  **Done when:** `roleModel.closedEnumerationsOnly` and `grantSet.anySubsetAccepted` pass, and `eligibleToRun()` is false for `{VIEWER, canRun}` and for `{AUTHOR}` and true for `{AUTHOR, canRun}` and `{REVIEWER, canRun}`.

- [ ] **T-02 · Migration: `tenant_membership`** — columns `tenant_id`, `id`, `identity_id`, `role`, `status`, `permission_version`, `created_at`, `updated_at`; primary key `(tenant_id, id)`; unique `(tenant_id, identity_id)`; `role` and `status` constrained to the enumerations; `identity_id` references the identity **[001]**. *(plan §2.1 Persistence, D1, D4; RF-1.4, RF-4.1, RF-6.6)* — after T-01
  **Done when:** the migration applies after plans 001–007's on an empty database and on one holding identities and tenants; a second membership for one identity in one tenant is rejected; `membership.uniquePerTenantIdentity` passes.

- [ ] **T-03 · Migration: `project_grant`** — columns `tenant_id`, `membership_id`, `project_id`, `author`, `reviewer`, `viewer`, `can_run`, `revoked_at`; primary key `(tenant_id, membership_id, project_id)`; composite foreign keys to `tenant_membership (tenant_id, id)` and `project (tenant_id, id)`, both restricting cross-tenant references; the project FK cascades on project delete into a revocation audit (T-31). *(plan §2.1 Persistence, D5; RF-1.4, RF-9.1, RF-9.4, LC-30)* — after T-02
  **Done when:** the migration applies; `grant.compositeKeysRejectCrossTenant` passes (a grant naming a project of another tenant is rejected by the key, not by application code).

- [ ] **T-04 · Row-level security policies** — `FORCE RLS` on both tables with `USING` and `WITH CHECK` on `tenant_id = current_setting('app.tenant_id')`; one additional policy on `tenant_membership` for `identity_id = current_setting('app.identity_id')` reachable only when that setting is present (D10, A-36); the runtime role non-owner, non-superuser, without `BYPASSRLS`. *(plan §2.1 Persistence, D10; RF-1.4, RF-4.3, RF-4.4, RNF-10)* — after T-03
  **Done when:** `rls.crossTenantReadWriteFkAttacksFail` passes under the runtime role, and a session without either setting reads zero rows.

- [ ] **T-05 · Problem-code catalogue** — register `access.denied-scope` (403, shared with plan 006's handler) and `access.forbidden-change` (403); each carries code, title and correlation id only — no reason class, no tenant or project existence, no content. *(plan §2.1 Problem codes, D14, D16; RF-3.8, RF-6.4, RF-11.2)* — after T-01
  **Done when:** a serialisation test renders both problems and finds only the permitted members; `deniedScope.codeOnlyWithCorrelationId` passes at the catalogue level.

- [ ] **T-06 · Shared matrix definition** — one machine-readable table (roles × grant sets × action classes × policy facts → permit or deny, plus the named missing grant per deny) kept in `contracts/access` and consumed by the BE `AuthorityMatrix` (T-10, T-11) and the FE mirror (T-35), so the two cannot drift. *(plan §2.2 `contracts/access`, D2, D12; RF-10.7, RNF-2)* — after T-01
  **Done when:** a build-time test shows the BE and FE artefacts are generated from the same table and a change to one row fails the parity test of T-38.

## B · Domain (pure, no Spring)

- [ ] **T-07 · `TenantMembership` — fields, invariants and state machine** — id, tenantId, identityId, `role`, `status` (`ACTIVE → SUSPENDED → ACTIVE`, `ACTIVE | SUSPENDED → REVOKED`, `REVOKED` terminal), `permissionVersion`, timestamps, its `ProjectGrant` children; the clock is always an argument. *(plan §2.1 `TenantMembership`, D6; RF-4.1, RF-4.2, RF-6.1, RF-6.6, LC-12)* — after T-01
  **Done when:** `membership.stateMachine` passes: every allowed transition succeeds, every other throws, and `REVOKED` accepts no transition.

- [ ] **T-08 · `TenantMembership` mutations** — `changeRole`, `suspend`, `reinstate`, `revoke` (revokes every grant in the same change, stamping `revokedAt`); each advances `permissionVersion` exactly once and returns the previous and new values for audit; `reinstate` leaves grants untouched. *(plan §2.1 `TenantMembership`, D4, D6; RF-4.3, RF-4.4, RF-4.5, RF-12.3, LC-12, LC-13)* — after T-07
  **Done when:** `version.advancesOncePerChange` (domain half) passes, `revoke` leaves no unrevoked grant, and `suspend` then `reinstate` leaves the grant set identical with the version advanced twice.

- [ ] **T-09 · `ProjectGrant` and `setGrants`** — child (tenantId, membershipId, projectId, `GrantSet`, `revokedAt`), unique per membership and project; `setGrants(projectId, GrantSet)` creates, replaces or — for an empty set — deletes the grant; refused when the membership is not `ACTIVE`; advances the version once. *(plan §2.1 `ProjectGrant`, D5, D6; RF-9.1, RF-9.4, RF-9.7, LC-9)* — after T-08
  **Done when:** unit tests show create, replace and delete-on-empty, that two grants for one project cannot coexist, and that `setGrants` on a `SUSPENDED` or `REVOKED` membership throws.

- [ ] **T-10 · `AuthorityMatrix` — tenant administration and content rows** — pure rule over `(tenantRole, grantSet, policy, actionClass, actionFacts)`: `TENANT_LIFECYCLE`, `TENANT_POLICY`, `TENANT_MEMBERS` for Owner / Admin / Member; `PROJECT_READ`, `AUTHOR`, `ACCEPT_MAPPING`, `UPLOAD_ASSET`, `AUTHOR_MACRO`, `REVIEW`, `PUBLISH_MACRO_OR_ASSET`, `SUBSCRIBE` from the grant set only; no default branch (an unlisted pair does not compile). *(plan §2.1 `AuthorityMatrix`, D2; RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-12.2, RNF-1, RNF-3)* — after T-06
  **Done when:** `matrix.ownerWithoutGrantReadsNothing`, `matrix.tenantRoleImpliesNoProjectAccess` and `matrix.noCrossScopeImplication` pass, and the rows of this task match the shared table of T-06.

- [ ] **T-11 · `AuthorityMatrix` — launch, lifecycle, trust, audit, export and approval rows** — `LAUNCH` = `eligibleToRun()` ∧ caller-supplied facts (valid approval, current policy, admission); `WHOLE_PROJECT_LIFECYCLE` = (Owner or Admin) ∧ any project role; `CONTENT_LIFECYCLE` = Author; `PUBLISH_TRUST` = Reviewer ∧ (Owner or Admin); `READ_AUDIT` and `EXPORT_PROJECT` = any project role on that project; `APPROVE` = Reviewer ∧ (¬self-authored ∨ ¬`requireDifferentApprover`); `APPROVE` and `LAUNCH` require a human membership context. *(plan §2.1 `AuthorityMatrix`, D2, D9; RF-2.8, RF-2.9, RF-2.10, RF-2.11, RF-2.12, RF-2.13, RF-2.14, RF-12.8, LC-6, LC-21, LC-22, LC-23, LC-24, LC-25)* — after T-10
  **Done when:** `matrix.canRunInertWithoutRole`, `matrix.reviewerOnlyOwnerLifecycle`, `matrix.memberNeverWholeProjectLifecycle`, `matrix.trustNeedsTwoAuthorities`, `matrix.auditFollowsProjectAuthority`, `matrix.exportPerProject`, `matrix.selfApprovalPerPolicy` and `matrix.approveAndLaunchRequireHumanMembership` pass.

- [ ] **T-12 · `AdministrationRule`** — pure rule `(actorRole, actorIsTarget, targetRole, change) → Permit | Deny`: Owner may set any role, create Owners and transfer; Admin may set `MEMBER` or `ADMIN` on non-Owner targets, never touch an Owner, never change own tenant role; nobody raises own tenant role; Owner lowers own only through transfer; Owner or Admin may set any grants including own; Member changes nothing. *(plan §2.1 `AdministrationRule`, D7; RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-12.1, LC-2, LC-3, LC-4, LC-5)* — after T-01
  **Done when:** `administrationRule.exhaustiveTable` passes over every (actor role × self/other × target role × change) combination.

- [ ] **T-13 · `TrustedActorTenantContext` and `DerivationOutcome`** — the immutable C10 value (identity, session, tenant, membership, role, membership status, tenant status, project scope with `GrantSet` or none, `permissionVersion`, policy revision, enrollment-gate outcome, correlation id); `DerivationOutcome` = `Derived(C10) | Denied(reasonClass)` with the eight reason classes; neither type is serialisable to a client (no public serialiser; `toString` redacted). *(plan §2.1 C10, `DerivationOutcome`, D3, D14; RF-3.1, RF-3.6, RF-6.2, RF-12.5)* — after T-01
  **Done when:** a reflection test finds no serialiser or accessor that exposes the whole context, and the reason-class enumeration has exactly the eight values of plan §2.1.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-27…T-32); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-14 · Port interfaces** — new: `MembershipRepository` (`findForUpdate`, `findForShare`, `findByTenantAndIdentity`, `listByIdentity`, `save`), `ProjectGrantRepository`, `TenantStatusPort`, `TenantPolicyPort` (`revision`, `requireDifferentApprover`, `requireMfaForMembers`), `ProjectExistsPort`, `SubscriptionRevocationPort`, `WorkRevocationPort` (`refuseQueued`, `requestBoundedCancel`), `AuthorityPort` (published), `MembershipCommands` and `GrantCommands` (published); implemented here: `MembershipProjectionPort` **[001]**, `MembershipExistsPort` **[007]**, `RequirementReasonPort` **[003]**. Reused: `SessionAssertionPort` **[006]**, `EnrollmentGate` and `FactorStatusPort` **[003]**, `AuditPort`, `Clock` **[001]**, outbox and idempotency outcome **[001 D1 / 002]**. No port reaches an approval, run, intent or content table. *(plan §2.1 Ports, D1; RF-1.6, RF-3.7, RNF-2)* — after T-07…T-13
  **Done when:** the application module compiles with no dependency on Spring or persistence classes, enforced by an ArchUnit test that also checks the dependency direction identity → access → project → execution and forbids any repository import across contexts.

- [ ] **T-15 · `DeriveAuthorityUseCase` — core derivation** — session facts from `SessionAssertionPort`; membership by (tenant, identity) with status and version; tenant status; grants for the requested project; route-versus-body ownership check; `Derived(C10)` or `Denied(NO_MEMBERSHIP | MEMBERSHIP_NOT_ACTIVE | TENANT_NOT_ACTIVE | SCOPE_MISMATCH | MISSING_GRANT)`; the requester learns only `access.denied-scope`. *(plan §2.1 `DeriveAuthorityUseCase`, D3, D14; RF-3.1, RF-3.2, RF-3.6, RF-5.2, RNF-4, LC-27, LC-28)* — after T-14
  **Done when:** `derive.fromCurrentFactsOnly`, `derive.absentStaleSuspendedRevokedDeletingDeny`, `derive.requestedScopeNeverAuthority`, `derive.routeBodyMismatchDenied` and `derive.unknownTenantUndisclosed` pass.

- [ ] **T-16 · Derivation — policy, enrollment gate and audit of denials** — read `TenantPolicyPort.revision` into C10; evaluate `EnrollmentGate` **[003]** with `FactorStatusPort` and deny `ENROLLMENT_GATE` when required; append `AUTHORITY_DENIED{reasonClass, actionClass}` for every `Denied`; the policy revision is checked by `APPROVE` and `LAUNCH` callers and never advances a membership version. *(plan §2.1 `DeriveAuthorityUseCase`, D9, D11, D14; RF-3.5, RF-3.8, RF-6.5, RF-11.2, LC-17, LC-26)* — after T-15
  **Done when:** `derive.enrollmentGateWithholdsAuthority`, `audit.deniedDerivationWithReasonClass`, `wire.carriesNoReason` and `policyChange.advancesPolicyRevisionNotMembershipVersion` pass.

- [ ] **T-17 · `AuthorityChecker` and `PermissionVersionGuard`** — `require(C10, actionClass, actionFacts)`: inside the caller's transaction, re-read the membership row `FOR SHARE`, compare status and `permissionVersion` with C10 (mismatch → `Denied(STALE_VERSION)`), evaluate `AuthorityMatrix`, throw `access.denied-scope` on `Deny`; async jobs call it with a fresh derivation for their own scope. *(plan §2.1 `AuthorityChecker`, `PermissionVersionGuard`, D3, D4; RF-3.3, RF-3.4, RF-6.3, RF-6.4, RF-12.4, RNF-6, LC-10)* — after T-15
  **Done when:** `checker.calledAtEveryEffectBoundary` (one stub use case per action class), `jobs.deriveAfreshPerScope` and `staleVersion.sensitiveWriteRefusedTransactionally` pass.

- [ ] **T-18 · `MembershipCommands` — `CreateMembership` and `ChangeTenantRole`** — each: derive the actor's C10 (or the bootstrap / acceptance context of plans 012 / 008), apply `AdministrationRule`, lock the target `FOR UPDATE`, apply the aggregate method, advance the version, append the audit entry, enqueue `AuthorityChanged`, record the idempotency outcome — one transaction. `CreateMembership` accepts `ADMIN` or `MEMBER` from an invitation (never `OWNER`) and `OWNER` only from the bootstrap; it replaces a `REVOKED` row (008 RF-6.2). `ChangeTenantRole` to or from `OWNER` is reachable only through plan 010's `TransferOwnership`. *(plan §2.1 `MembershipCommands`, D7, D15; RF-1.6, RF-8.1, RF-8.2, RF-8.3, RF-8.5, RF-8.6, RF-9.5, LC-19, LC-20)* — after T-17, T-12
  **Done when:** `membership.createdOnlyByInvitationOrBootstrap`, `createMembership.refusesOwnerFromInvitation`, `admin.cannotTouchOwnerOrOwnRole` and `nobody.raisesOwnTenantRole` pass.

- [ ] **T-19 · `MembershipCommands` — `SuspendMembership`, `ReinstateMembership`, `RevokeMembership`** — same transaction shape; `RevokeMembership` cascades through `revoke` (grants stamped `revokedAt`), the projection then omits the tenant; `SuspendMembership` keeps grants and emits `AuthorityChanged{SUSPENDED}`; neither touches content, identity or other tenants. *(plan §2.1 `MembershipCommands`, D6, D8; RF-4.3, RF-4.4, RF-4.5, RF-4.6, RF-7.5, RF-7.7, LC-12, LC-13)* — after T-18
  **Done when:** `suspend.noAuthorityGrantsRetained`, `reinstate.grantsUnchangedVersionAdvanced`, `revoke.cascadesGrantsAndEndsListing`, `revoke.leavesAuthoredContentAndProvenance` and `revoke.scopeProjectOrTenantOnly` pass.

- [ ] **T-20 · `GrantCommands.SetProjectGrants`** — derive the actor's C10; `AdministrationRule` (Owner or Admin; self allowed; Member refused with `access.forbidden-change`); require the target `ACTIVE` and the project in the tenant (`ProjectExistsPort`); lock; `setGrants` (empty → delete); version; audit `GRANTS_SET` / `GRANT_REMOVED`; `AuthorityChanged`; idempotency outcome. *(plan §2.1 `GrantCommands`, D5, D7, D15; RF-8.4, RF-9.1, RF-9.2, RF-9.3, RF-9.4, RF-9.6, RF-9.7, LC-4, LC-5, LC-6, LC-7, LC-8, LC-9)* — after T-18
  **Done when:** `grants.ownerSetsOwnAudited`, `grants.memberCannotChangeAny`, `grant.explicitRecordOnly`, `grant.emptySetDeletesRow`, `grant.canRunStoredEligibilityComputed`, `authorRemoved.canRunBecomesInert` and `grant.refusedForNonActiveMembership` pass.

- [ ] **T-21 · `AuthorityChangedHandler`** — consumes `AuthorityChanged{tenantId, membershipId, projectId?, kind}` and `TenantStatusChanged{tenantId, status}` from the outbox: `SubscriptionRevocationPort.close(scope)`; `WorkRevocationPort.refuseQueued(scope)` (NOT EXECUTED) and `requestBoundedCancel(scope)`; for a tenant status change, every membership of the tenant; never a rollback, never a verdict change. *(plan §2.1 `AuthorityChangedHandler`, D8; RF-5.3, RF-7.1, RF-7.2, RF-7.3, RF-7.4, RF-7.7, RNF-7, LC-11, LC-14)* — after T-19, T-20
  **Done when:** `revoke.deniedFromNextRequest`, `authorityChanged.closesSubscriptions`, `authorityChanged.refusesQueuedAsNotExecuted`, `authorityChanged.requestsBoundedCancelKeepsFacts` and `tenantStatusChanged.refreshesAndRevokesWork` pass with recording doubles.

- [ ] **T-22 · Tenant status in derivation** — `TenantStatusPort` read on every derivation; `SUSPENDED` or `DELETING` → `Denied(TENANT_NOT_ACTIVE)` for every membership; memberships and grants never written by a tenant transition; an `ACTIVE` tenant restores each membership's own facts. *(plan §2.1 `DeriveAuthorityUseCase`, D9; RF-5.1, RF-5.2, RF-5.4, LC-14, LC-15)* — after T-15
  **Done when:** `tenantStatus.suspendedDeniesAllKeepsMemberships` and `tenantActive.restoresExactMembershipFacts` pass.

- [ ] **T-23 · `MyAuthorityQuery`** — for the caller: tenant role, membership status, `permissionVersion` (for display and refresh detection only), `GrantSet` per permitted project, `requireDifferentApprover`, tenant status, enrollment-gate reason **[003]**; structurally a read; never another member's facts. *(plan §2.1 `MyAuthorityQuery`, D12; RF-3.8, RF-6.4, RF-10.4, RF-10.7, RF-10.8)* — after T-16, T-22
  **Done when:** a test shows the shape for an Owner without grants, an Author with inert `canRun` and a gated Admin, and that no field names another identity.

- [ ] **T-24 · Published queries** — `MembershipProjectionPort` **[001]** (identity-scoped listing through the narrow query that sets `app.identity_id`: tenant, tenant status, role, membership status, the person's grants; `SUSPENDED` shown without project names, `REVOKED` omitted), `MembershipExistsPort` **[007]**, `RequirementReasonPort` **[003]** (`OWNER_ADMIN`, `MEMBER_POLICY`, `NONE` with tenant names), `TenantMembersQuery` (010), `ProjectGrantsQuery` (011). *(plan §2.1 Published queries, D10; RF-4.3, RF-4.4, RF-10.1, RF-10.2, RF-10.4)* — after T-14
  **Done when:** plan 001's landing tests, plan 003's `SecurityMfaQuery` tests and plan 007's `MembershipExistsPort` tests pass against these implementations instead of their stubs, and the listing omits a `REVOKED` membership and hides project names of a `SUSPENDED` one.

- [ ] **T-25 · Ownership-transfer hook** — `ChangeTenantRole` exposes the two-step form plan 010's `TransferOwnership` needs (promote the target to `OWNER`, demote the actor) to be called inside 010's ownership-row lock with the last-active-Owner guard; no other caller may set or unset `OWNER`. *(plan §2.1 `MembershipCommands`, D7; RF-8.7, LC-16)* — after T-18
  **Done when:** `transferOwnership.guardOnEveryPath` passes with plan 010's stub under the ownership lock, and a direct `ChangeTenantRole(OWNER)` outside it is refused.

- [ ] **T-26 · Idempotency outcomes for commands** — every command of T-18…T-20 records its outcome under `(tenantId, operation, idempotencyKey, bodyHash)` in its transaction; a repeated key with the same body returns the recorded outcome; a different body conflicts. *(plan §2.1, D15; RF-8.6, RF-11.4, LC-18)* — after T-20
  **Done when:** `command.idempotentOutcome` passes for all six commands.

## D · Infrastructure adapters

- [ ] **T-27 · Persistence adapters** — implementations (access API per A-35) of `MembershipRepository` (`findForUpdate` as `SELECT … FOR UPDATE`, `findForShare` as `FOR SHARE`, the identity listing through the narrow query) and `ProjectGrantRepository`. *(plan §2.1 Persistence, D4, D10; RF-6.3, RF-6.6, RF-8.6, LC-18)* — after T-02…T-04, T-14
  **Done when:** repository tests against real PostgreSQL pass for every method; `version.monotonicUnderConcurrency` and `grants.twoAdministratorsSerialised` pass (the second writer waits and sees the advanced version).

- [ ] **T-28 · C10 request filter** — after the session filter **[006]**: resolves tenant and project from the route, runs `DeriveAuthorityUseCase`, stores C10 in the request context, and issues `SET LOCAL app.tenant_id` (and `app.identity_id` only for the narrow listing) inside every transaction of the request from C10, never from a client value; identity-scoped endpoints skip derivation; C10 never reaches the runner envelope. *(plan §2.1 C10 request filter, D3; RF-3.1, RF-3.2, RF-3.7, RF-12.5)* — after T-16, T-27
  **Done when:** `c10.neverSerialisedToClient`, `version.clientValueIgnored` and `runnerEnvelope.carriesNoRoleOrGrant` pass, and a transaction opened for a tenant request carries the tenant of C10 even when the body names another.

- [ ] **T-29 · Port adapters for tenant status, policy and project existence** — `TenantStatusPort` over the `tenant` context's published read; `TenantPolicyPort` over the policy record of F-SET-05 exposing only the tightening flags and the revision; `ProjectExistsPort` over the `project` context. *(plan §2.1 Ports, D9; RF-5.1, RF-6.5, RF-12.7, LC-17)* — after T-14
  **Done when:** `policyPort.exposesTighteningOnly` and `approve.selfApprovalRefusedUnderTightenedPolicy` (with an approval stub calling `AuthorityPort`) pass.

- [ ] **T-30 · Outbox event and consumer adapters** — `AuthorityChanged` and `TenantStatusChanged` through the existing outbox **[002]**; the `execution` adapter of `WorkRevocationPort` (refuse queued tickets of the scope as NOT EXECUTED, request bounded cancellation at the next checkpoint, never mark failed) and the realtime adapter of `SubscriptionRevocationPort`. *(plan §2.1 Outbox and event adapters, D8; RF-7.2, RF-7.3, RF-7.4, RNF-8)* — after T-21
  **Done when:** the handler tests of T-21 pass against the real outbox with the two adapters, and a cancelled attempt's recorded facts and verdict are unchanged in the execution double.

- [ ] **T-31 · Audit adapter** — map `MEMBERSHIP_CREATED{source}`, `TENANT_ROLE_CHANGED`, `MEMBERSHIP_SUSPENDED`, `MEMBERSHIP_REINSTATED`, `MEMBERSHIP_REVOKED`, `GRANTS_SET`, `GRANT_REMOVED`, `AUTHORITY_DENIED{reasonClass, actionClass}` to `AuditPort` with acting and affected membership and identity, tenant, project, previous value, new value, resulting version, correlation id and time; a serialisation guard rejects content and secrets; project deletion emits `GRANT_REMOVED` per grant. *(plan §2.1 Audit adapter; RF-11.1, RF-11.2, RF-11.3, LC-30)* — after T-14
  **Done when:** `audit.entryPerChange`, `audit.containsNoContentOrSecret` and `projectDeleted.grantsEndAudited` pass.

- [ ] **T-32 · Operator exclusion and configuration check** — a structural test that no membership or grant row can reference the platform operator principal (A-01 records live outside `access`), and a startup check that this feature binds no numeric property (checkpoint cadence and audit retention are owned elsewhere). *(plan §2.1, §1 P13; RF-1.5, RF-12.6, RNF-11)* — after T-14
  **Done when:** `operator.cannotHoldMembershipOrGrant` passes and the startup suite of this feature contains no property of its own.

## E · API

- [ ] **T-33 · `GET /api/tenants/{tenantId}/me/authority?project=`** — under the session filter and the C10 filter; 200 the `MyAuthorityQuery` shape; 403 `access.denied-scope`. The administration endpoints are plans 010 and 011's and call the published commands. *(plan §2.1 `AuthorityController`, D12; RF-3.8, RF-10.4, RF-10.7)* — after T-23, T-28
  **Done when:** `deniedScope.codeOnlyWithCorrelationId` passes at the API level and the 200 body matches the T-23 shapes.

- [ ] **T-34 · Contract and absence checks** — a test enumerating the surface: the two problem codes with correlation id; no answer carries C10, a reason class, a version as an input, another identity's facts or tenant existence; `AuthorityChecker` is wired into every protected use case of every context (one stub per action class). *(plan §3 RF-3, RF-11; RF-3.3, RF-6.2, RF-11.2, RF-12.5)* — after T-33
  **Done when:** `wire.carriesNoReason`, `c10.neverSerialisedToClient` and `checker.calledAtEveryEffectBoundary` pass across the API surface.

## F · Front end

- [ ] **T-35 · `contracts/access` and the generated mirror** — the closed enumerations, the `MyAuthority` shape, the two problem codes, and the FE mirror of the matrix generated from the shared table of T-06 (permit / deny and the named missing grant per action class). *(plan §2.2 `contracts/access`, D12; RF-1.1, RF-1.2, RF-1.3, RF-10.7)* — after T-06
  **Done when:** guard unit tests accept one fixture per documented answer and problem, and the generated mirror has one entry per (role × grant set × action class) of the shared table.

- [ ] **T-36 · i18n bundles `access.{en,es,fr}`** — every statement of RF-10.1–10.6 verbatim from R13, R17, N06, N03, N04 and the approval surface, the inert-`canRun` text (S-47), role and grant names, and the forbidden-change messages ("you cannot change this role", "you cannot change your own tenant role"). *(plan §2.2 i18n, D13, D16; RF-10.1, RF-10.2, RF-10.3, RF-10.4, RF-10.5, RF-10.6, RF-10.8, RNF-12)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value, and the EN values of RF-10.1–10.5 match the specification verbatim.

- [ ] **T-37 · `AuthorityStore`** — `shared/security`: the viewer's tenant role, membership status, `permissionVersion`, `GrantSet` per project, policy facts and enrollment reason from `GET me/authority`; loaded at step 4 of the guarded switch (016) and refreshed once by the denied-scope handler **[006]**; cleared with the scope epoch **[001]**; never persisted; never sent back as an input. *(plan §2.2 `AuthorityStore`, D12; RF-3.8, RF-6.4, RF-12.5)* — after T-35
  **Done when:** unit tests show the store cleared on epoch advance, no storage write (storage spies), and that no request built by the feature carries the version or the role.

- [ ] **T-38 · `useAuthority(actionClass, projectId?)`** — returns `{permitted, missingGrant}` from the store and the generated mirror; used only to hide or disable a control and to name the missing grant; the parity test compares every mirror entry with the shared table. *(plan §2.2 `useAuthority`, D12; RF-10.7, RF-10.8, RNF-2)* — after T-37
  **Done when:** `useAuthority.parityWithSharedTable` and `control.hiddenOrDisabledNamesMissingGrant` pass.

- [ ] **T-39 · `AuthorityStatements` component** — `shared/ui`: one component with the variants `members`, `grants`, `policy`, `chooser`, `creation`, `approval`, `inertCanRun`, rendering the bundle keys of T-36; consumed by the surfaces of plans 010, 011, 012, 013, settings and approval; identical text on every role variant. *(plan §2.2 `AuthorityStatements`, D13; RF-10.1, RF-10.2, RF-10.3, RF-10.4, RF-10.5, RF-10.6, RF-10.8, RF-10.9)* — after T-36
  **Done when:** `statements.variantsRenderSpecText` and `grantsEditor.inertCanRunStatement` pass, and a render with an Owner, an Admin and a Member store yields identical statement text.

- [ ] **T-40 · `DeniedScopeHandler` extension [006]** — `access.forbidden-change` renders the safe problem with the limit named from the bundle (never from the server) and leaves the session and the store untouched; `access.denied-scope` keeps plan 006's one-refresh rule, now refreshing through `GET me/authority`. *(plan §2.2 `shared/errors`, D16; RF-3.8, RF-6.4, RF-8.2, RF-8.3)* — after T-37
  **Done when:** `handler.refreshesOnceThenResumesOrDenies` passes against the authority endpoint double, and a forbidden-change answer renders the bundle text with no server-provided reason.

- [ ] **T-41 · Draft grant check on resume [006]** — the `ProtectedBoundary` grant check of plan 006 uses `useAuthority(AUTHOR, projectId)` from the refreshed store to decide whether a retained draft may be shown; a draft whose edit is no longer permitted is purged. *(plan §2.2, §3 RF-7.6; RF-7.6, LC-10)* — after T-38
  **Done when:** `drafts.purgedWhenGrantNoLongerPermits` passes.

- [ ] **T-42 · Reflow and accessibility of the statements** — the statements component reflows with its host surface at 360 px, 768 px and 1280 px; disabled controls expose the missing grant in their accessible description; status by text. *(plan §2.2, D13; RF-10.9, RNF-12)* — after T-39, T-38
  **Done when:** `statements.identicalAcrossRoleVariantsAndWidths` passes and a disabled control's accessible description names the grant.

## G · Verification suites

- [ ] **T-43 · Exhaustive matrix suite** — `authorityMatrix.exhaustiveTable`: every tenant role × every grant set (all sixteen subsets of three roles and `canRun`) × every action class × policy on/off, compared with the specification's RF-2 table; plus the named cases of T-10 and T-11. *(plan §5.2 RF-2 rows, D2; RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-2.8, RF-2.9, RF-2.10, RF-2.11, RF-2.12, RF-2.13, RF-2.14, RNF-1, RNF-3, LC-1, LC-6, LC-21, LC-22, LC-23, LC-24, LC-25)* — after T-11
  **Done when:** the exhaustive test passes with zero unlisted combinations and the expected table is reviewed against the specification by a second person.

- [ ] **T-44 · Administration suite** — `administrationRule.exhaustiveTable`, `admin.cannotTouchOwnerOrOwnRole`, `nobody.raisesOwnTenantRole`, `grants.ownerSetsOwnAudited`, `grants.memberCannotChangeAny`, `invitation.cannotExceedInviter` (008 stub). *(plan §5.2 RF-8 rows, D7; RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-8.5, RF-12.1, LC-2, LC-3, LC-4, LC-5, LC-19)* — after T-26
  **Done when:** all six tests pass.

- [ ] **T-45 · Concurrency suite** — `staleVersion.sensitiveWriteRefusedTransactionally`, `version.monotonicUnderConcurrency`, `grants.twoAdministratorsSerialised`, `command.lockVersionAuditOutboxAtomic`, `command.partialChangeNeverObservable`, revoke racing a launch, suspension racing a subscription. *(plan §5.1 Concurrency, D4, D15; RF-6.3, RF-6.6, RF-8.6, RNF-5, RNF-6, LC-10, LC-18)* — after T-27, T-30
  **Done when:** all seven tests pass ten times in a row with real PostgreSQL.

- [ ] **T-46 · Security suite** — `rls.crossTenantReadWriteFkAttacksFail`, `membership.uniquePerTenantIdentity`, `grant.compositeKeysRejectCrossTenant`, pooled-context contamination after error and cancellation, the identity policy reachable only through the narrow listing. *(plan §5.1 Security, D10; RF-1.4, RNF-10)* — after T-28
  **Done when:** every attack test fails to read or write another tenant's row under the runtime role, and a pooled connection reused after a rollback carries no scope.

- [ ] **T-47 · Status and revocation suite** — `membership.stateMachine`, `suspend.noAuthorityGrantsRetained`, `reinstate.grantsUnchangedVersionAdvanced`, `revoke.cascadesGrantsAndEndsListing`, `reinvite.createsNewMembershipNoRestore` (008 stub), `tenantStatus.suspendedDeniesAllKeepsMemberships`, `tenantStatusChanged.refreshesAndRevokesWork`, `tenantActive.restoresExactMembershipFacts`, `revocation.nextRequestSeesNewFacts`, `authorityChanged.requestsBoundedCancelKeepsFacts`. *(plan §5.2 RF-4, RF-5, RF-7 rows, D6, D8, D9; RF-4.1, RF-4.2, RF-4.3, RF-4.4, RF-4.5, RF-5.1, RF-5.2, RF-5.3, RF-5.4, RF-7.1, RF-7.2, RF-7.3, RF-7.4, RNF-7, RNF-8, LC-11, LC-12, LC-13, LC-14, LC-15)* — after T-30
  **Done when:** all ten tests pass.

- [ ] **T-48 · Boundary suite** — `bootstrap.membershipAndTickedGrantsOneTransaction` (012 stub), `projectCreation.impliesNoGrant`, `createMembership.refusesOwnerFromInvitation`, `transferOwnership.guardOnEveryPath` (010 stub), `projectDeleted.grantsEndAudited`, `identityRestricted.derivationYieldsNothing` (A-27 stub), `jobs.deriveAfreshPerScope`, `runnerEnvelope.carriesNoRoleOrGrant`. *(plan §5.3 boundaries, D3, D7; RF-3.4, RF-3.7, RF-8.5, RF-8.7, RF-9.5, RF-9.6, LC-16, LC-19, LC-20, LC-29, LC-30)* — after T-45
  **Done when:** all eight tests pass with the named stubs.

- [ ] **T-49 · FE e2e role variants and automated accessibility** — Playwright journeys for US-1…US-11 in EN, ES and FR at desktop, tablet and mobile widths on the surfaces of plans 010, 011, 012 and 013 with the BE doubles: Owner without grant, Admin, Member, Author, Reviewer with and without `requireDifferentApprover`, Viewer, inert `canRun`, revoked contractor, stale-version refusal, suspended member; axe on every statement variant (`a11y.authorityStatements.{en,es,fr}.{desktop,tablet,mobile}`). *(plan §5.1 FE e2e; RF-10.1, RF-10.2, RF-10.3, RF-10.4, RF-10.5, RF-10.6, RF-10.7, RF-10.8, RF-10.9, RNF-12, US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8, US-9, US-10, US-11)* — after T-42, T-34
  **Done when:** all journeys pass in the three languages and three widths with zero critical or serious axe violations.

- [ ] **T-50 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist: disabled controls announce the missing grant; statements read in order on the members, grants, policy and chooser surfaces; versions recorded in the release-support matrix. *(plan §5.1 Manual assistive; RNF-12)* — after T-49
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-51 · Adversarial suite (US-12)** — a tenant role reading a project; an Author approving; self-escalation by invitation, by own-role edit and by a Member editing own grants; a revived grant after revocation; a write under a stale version; an operator principal inside a tenant; a browser-supplied tenant id, project id or version; a non-human context requesting `APPROVE` or `LAUNCH`. *(plan §5.2 US-12 row, D2, D4, D6, D7; RF-12.1, RF-12.2, RF-12.3, RF-12.4, RF-12.5, RF-12.6, RF-12.7, RF-12.8, US-12)* — after T-43…T-48
  **Done when:** every attack test fails to achieve its goal and asserts the safe problem or the structural refusal.

## H · Closing

- [ ] **T-52 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-43…T-51
  **Done when:** the script reports 140 of 140 identifiers (86 RF, 12 RNF, 30 LC, 12 US) covered by at least one passing test.

- [ ] **T-53 · Register-dependency review** — walk plan §6 — A-53…A-58, S-47, S-48, S-49 as decided, and the open A-11, A-27, A-35, A-36, A-39, A-42, S-06 and S-14: for each, confirm the built assumption matches the current Figma register state or record the deferral. *(spec §7 item 13)* — after T-52
  **Done when:** each of the seventeen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-54 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run ids, the reviewed matrix table, a11y report, manual checklist, coverage report, register review). — after T-53
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-01, T-35 |
| RF-1.2 | T-01, T-35 |
| RF-1.3 | T-01, T-35 |
| RF-1.4 | T-02, T-03, T-04, T-46 |
| RF-1.5 | T-32 |
| RF-1.6 | T-14, T-18 |
| RF-2.1 | T-10, T-43 |
| RF-2.2 | T-10, T-43 |
| RF-2.3 | T-10, T-43 |
| RF-2.4 | T-10, T-43 |
| RF-2.5 | T-10, T-43 |
| RF-2.6 | T-10, T-43 |
| RF-2.7 | T-10, T-43 |
| RF-2.8 | T-11, T-43 |
| RF-2.9 | T-11, T-43 |
| RF-2.10 | T-11, T-43 |
| RF-2.11 | T-11, T-43 |
| RF-2.12 | T-11, T-43 |
| RF-2.13 | T-11, T-43 |
| RF-2.14 | T-11, T-43 |
| RF-3.1 | T-13, T-15, T-28 |
| RF-3.2 | T-15, T-28 |
| RF-3.3 | T-17, T-34 |
| RF-3.4 | T-17, T-48 |
| RF-3.5 | T-16 |
| RF-3.6 | T-13, T-15 |
| RF-3.7 | T-14, T-28, T-48 |
| RF-3.8 | T-05, T-16, T-23, T-33, T-37, T-40 |
| RF-4.1 | T-02, T-07, T-47 |
| RF-4.2 | T-07, T-47 |
| RF-4.3 | T-04, T-08, T-19, T-24, T-47 |
| RF-4.4 | T-04, T-08, T-19, T-24, T-47 |
| RF-4.5 | T-08, T-19, T-47 |
| RF-4.6 | T-19 |
| RF-5.1 | T-22, T-29, T-47 |
| RF-5.2 | T-15, T-22, T-47 |
| RF-5.3 | T-21, T-47 |
| RF-5.4 | T-22, T-47 |
| RF-6.1 | T-07 |
| RF-6.2 | T-13, T-34 |
| RF-6.3 | T-17, T-27, T-45 |
| RF-6.4 | T-05, T-17, T-23, T-37, T-40 |
| RF-6.5 | T-16, T-29 |
| RF-6.6 | T-02, T-07, T-27, T-45 |
| RF-7.1 | T-21, T-47 |
| RF-7.2 | T-21, T-30, T-47 |
| RF-7.3 | T-21, T-30, T-47 |
| RF-7.4 | T-21, T-30, T-47 |
| RF-7.5 | T-19 |
| RF-7.6 | T-41, T-41 |
| RF-7.7 | T-19, T-21 |
| RF-8.1 | T-12, T-18, T-44 |
| RF-8.2 | T-12, T-18, T-40, T-44 |
| RF-8.3 | T-12, T-18, T-40, T-44 |
| RF-8.4 | T-12, T-20, T-44 |
| RF-8.5 | T-18, T-44, T-48 |
| RF-8.6 | T-18, T-26, T-27, T-45 |
| RF-8.7 | T-25, T-48 |
| RF-9.1 | T-03, T-09, T-20 |
| RF-9.2 | T-01, T-20 |
| RF-9.3 | T-01, T-20 |
| RF-9.4 | T-03, T-09, T-20 |
| RF-9.5 | T-18, T-48 |
| RF-9.6 | T-20, T-48 |
| RF-9.7 | T-09, T-20 |
| RF-10.1 | T-24, T-36, T-39, T-49 |
| RF-10.2 | T-24, T-36, T-39, T-49 |
| RF-10.3 | T-36, T-39, T-49 |
| RF-10.4 | T-23, T-24, T-33, T-36, T-39, T-49 |
| RF-10.5 | T-36, T-39, T-49 |
| RF-10.6 | T-36, T-39, T-49 |
| RF-10.7 | T-06, T-23, T-33, T-35, T-38, T-49 |
| RF-10.8 | T-23, T-36, T-38, T-39, T-49 |
| RF-10.9 | T-39, T-42, T-49 |
| RF-11.1 | T-31 |
| RF-11.2 | T-05, T-16, T-31, T-34 |
| RF-11.3 | T-31 |
| RF-11.4 | T-26 |
| RF-12.1 | T-12, T-44, T-51 |
| RF-12.2 | T-10, T-51 |
| RF-12.3 | T-08, T-51 |
| RF-12.4 | T-17, T-51 |
| RF-12.5 | T-13, T-28, T-34, T-37, T-51 |
| RF-12.6 | T-32, T-51 |
| RF-12.7 | T-29, T-51 |
| RF-12.8 | T-11, T-51 |
| RNF-1 | T-10, T-43 |
| RNF-2 | T-06, T-14, T-38 |
| RNF-3 | T-10, T-43 |
| RNF-4 | T-15 |
| RNF-5 | T-45 |
| RNF-6 | T-17, T-45 |
| RNF-7 | T-21, T-47 |
| RNF-8 | T-30, T-47 |
| RNF-9 | T-01 |
| RNF-10 | T-04, T-46 |
| RNF-11 | T-32 |
| RNF-12 | T-36, T-42, T-49, T-50 |
| LC-1 | T-43 |
| LC-2 | T-12, T-44 |
| LC-3 | T-12, T-44 |
| LC-4 | T-12, T-20, T-44 |
| LC-5 | T-12, T-20, T-44 |
| LC-6 | T-11, T-20, T-43 |
| LC-7 | T-20 |
| LC-8 | T-20 |
| LC-9 | T-09, T-20 |
| LC-10 | T-17, T-41, T-45 |
| LC-11 | T-21, T-47 |
| LC-12 | T-07, T-08, T-19, T-47 |
| LC-13 | T-08, T-19, T-47 |
| LC-14 | T-21, T-22, T-47 |
| LC-15 | T-22, T-47 |
| LC-16 | T-25, T-48 |
| LC-17 | T-16, T-29 |
| LC-18 | T-26, T-27, T-45 |
| LC-19 | T-18, T-44, T-48 |
| LC-20 | T-18, T-48 |
| LC-21 | T-11, T-43 |
| LC-22 | T-11, T-43 |
| LC-23 | T-11, T-43 |
| LC-24 | T-11, T-43 |
| LC-25 | T-11, T-43 |
| LC-26 | T-16 |
| LC-27 | T-15 |
| LC-28 | T-15 |
| LC-29 | T-48 |
| LC-30 | T-03, T-31, T-48 |
| US-1 … US-11 | T-49 (journeys) |
| US-12 | T-51 (adversarial suite) |

Self-check: all 86 RF criteria, 12 RNF, 30 limit cases and 12 user stories of `specRolesGrantsAuthority.md` v0.1 appear above; every task of phases A–G names at least one requirement, and the closing tasks name the completion criterion they serve.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| A-53 grant combinations, inert canRun (decided, Q2) | One grant row with four flags; empty set deletes; eligibility computed by the matrix | T-01, T-03, T-09, T-11, T-20 |
| A-54 membership statuses (decided, Q3) | State machine in the aggregate; revoke cascades grants; suspend retains; projection shows suspended, omits revoked | T-07, T-08, T-19, T-24 |
| A-55 permissionVersion (decided, Q4) | Per-membership counter advanced under exclusive lock, checked under share lock; policy revision separate | T-08, T-17, T-27, T-29 |
| A-56 revocation and work in flight (decided, Q5) | Synchronous denial by derivation; `AuthorityChanged` outbox event to execution and realtime adapters | T-21, T-30 |
| A-57 administration authority (decided, Q6) | `AdministrationRule` applied by every command; Owner changes through 010's transfer only | T-12, T-18, T-20, T-25 |
| A-58 whole-project lifecycle grant (decided, Q7) | `WHOLE_PROJECT_LIFECYCLE` row of the matrix | T-11, T-43 |
| S-47 inert canRun statement (decided) | `inertCanRun` variant of `AuthorityStatements` | T-36, T-39 |
| S-48 suspended-membership chooser state (decided) | Projection marks `SUSPENDED` and hides project names; rendered by plan 013 | T-24 |
| S-49 Admin view of R13 (decided) | `useAuthority` hides actions on Owner rows and own role; rendered by plan 010 | T-38 |
| A-11 configuration values | No numeric value owned here | T-32 |
| A-27 identity statuses | Restricted or erasing identities yield no derivation (session assertion [006]) | T-48 |
| A-35 persistence tooling | Adapters, migrations and policies against the placeholder API and tool | T-02, T-03, T-04, T-27 |
| A-36 identity-scoped membership listing | Dedicated identity-predicate RLS policy reachable only through the narrow query | T-04, T-24, T-27, T-46 |
| A-39 revocation scope | Membership revocation never touches sessions; run-scoped authority is execution's | T-21, T-30 |
| A-42 enrollment gate timing | Gate evaluated at every derivation | T-16 |
| S-06 role variants | One statements component identical on every variant | T-39, T-42, T-49 |
| S-14 templates | No message sent by this feature | — |
