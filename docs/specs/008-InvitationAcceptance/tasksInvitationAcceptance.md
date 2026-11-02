# Tasks 008 — Invitation acceptance

**Feature** Opening an invitation link, reviewing the exact scope it grants, accepting it as the invited identity or declining it · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specInvitationAcceptance.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planInvitationAcceptance.md` v0.1 (modules §2, decisions D1–D15, tests §5, register assumptions §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[002]** / **[003]** / **[006]** extend an element that plan 001 (Login), 002 (Signup and verification), 003 (Enrollment) or 006 (Sessions) builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Tasks marked **[009]** extend plan 009's membership and grant model (`TenantMembership`, `ProjectGrant`, `GrantSet`); tasks marked **[008 → 010]** build the invitation record whose issuing side plan 010 adds. Sign-out is specification 007's; where a task calls it, the 007 mutation is used or stubbed in the shape its specification states. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and migration tool are the A-35 placeholders.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-01 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Invitation configuration record** — `InvitationPolicy` value object and `InvitationPolicyProperties` binding: invitation expiry (stamped at issue by specification 010, validated here because acceptance depends on it), throttle steps and window for `INVITATION_NETWORK`. Each value `@Validated` as present, positive and bounded above by a configured ceiling; no default anywhere. *(plan §2.1 `InvitationPolicy`, `InvitationPolicyProperties`, D10; RF-5.7, RF-9.4, RNF-6)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed or set unbounded.

- [ ] **T-02 · `access` module skeleton and import DAG** — the bounded context `com.aspectra.access` with the four layers `api / application / domain / infrastructure`, the allowed-import rule (outer adapters → ports and use cases → pure domain), and an ArchUnit test (test-only) that forbids any import of `identity`, `tenant` or `audit` classes other than their published ports and forbids any repository of another context. The existing `access.domain` C10 derivation **[001/003/006]** is moved into this skeleton unchanged. *(plan §2.1 header, D1; RNF-9, RNF-10)* — after T-01
  **Done when:** the module compiles, the ArchUnit test passes, and plans 001, 003 and 006's tests that touch the C10 derivation still pass.

- [ ] **T-03 · Migration: `tenant_membership` [009]** — the table exactly as plan 009 defines it: primary key `(tenant_id, id)`, unique `(tenant_id, identity_id)`, `role` check ∈ {`OWNER`, `ADMIN`, `MEMBER`}, `status` check ∈ {`ACTIVE`, `SUSPENDED`, `REVOKED`}, `permission_version` (positive), timestamps; `FORCE RLS` with `USING` and `WITH CHECK` on `tenant_id`; the runtime role is non-privileged. Built here in plan 009's shape only when plan 009's migration is not yet applied; otherwise this task verifies it. *(plan §2.1 Persistence, D6; RF-6.1, RF-6.2, RF-6.6, RNF-9)* — after T-02
  **Done when:** the migration applies on an empty database; a second row for the same `(tenant_id, identity_id)` is rejected; an insert with no transaction-local tenant context is rejected by RLS; an insert under a matching `SET LOCAL` context succeeds.

- [ ] **T-04 · Migration: `project_grant` [009]** — the table exactly as plan 009 defines it: primary key `(tenant_id, membership_id, project_id)`, the four flags `author`, `reviewer`, `viewer`, `can_run`, `revoked_at`, composite foreign keys to the membership; `FORCE RLS` as T-03. Built here in plan 009's shape only when plan 009's migration is not yet applied. *(plan §2.1 Persistence, D6; RF-6.1, RF-6.2, RF-6.3)* — after T-03
  **Done when:** the migration applies; a second row for the same membership and project is rejected by the primary key while an update that clears `revoked_at` and sets new flags on the existing row succeeds; RLS rejects a write without context.

- [ ] **T-05 · Migration: `tenant_invitation`** — composite key `(tenant_id, id)`, `secret_hash` unique across tenants, `invited_email_normalised`, `tenant_role` check ∈ {`ADMIN`, `MEMBER`} (Owner excluded at the database), `project_id` nullable, `grant_roles`, `can_run`, `issued_by`, `issued_at`, `expires_at`, `state` check ∈ {`PENDING`, `ACCEPTED`, `DECLINED`, `REVOKED`, `EXPIRED`}, `accepted_by`, `accepted_at`, `declined_at`, `accept_idempotency_key`; partial index on pending rows per tenant; `FORCE RLS` as T-03. *(plan §2.1 Persistence, D2, D5, D7, D9; RF-1.1, RF-5.1, RF-5.2, RF-6.3, RNF-8)* — after T-04
  **Done when:** the migration applies; `secret_hash` uniqueness is enforced; a row with `tenant_role = 'OWNER'` or with a second project is rejected (`schema.tenantRoleRejectsOwner`); RLS rejects a read or write without context.

- [ ] **T-06 · Audited `SECURITY DEFINER` lookup** — function `invitation_tenant_by_hash(secret_hash) → tenant_id` owned by a definer role, returning the tenant id and nothing else (no state, no address, no role); registered in the audited list of definer functions that F-X-01 requires; `EXECUTE` granted to the runtime role only. *(plan §2.1 Persistence data-model note, D5; RF-1.1, RNF-9)* — after T-05
  **Done when:** `securityDefiner.returnsTenantIdOnly` passes: the function returns one column, returns nothing for an unknown hash, and the runtime role cannot read `tenant_invitation` through it or directly without context.

- [ ] **T-07 · Throttle dimension `INVITATION_NETWORK` [001]** — `signin_throttle` **[001]** receives the new dimension value; `AbuseThrottlePolicy` **[001]** gains the key fed by the `InvitationPolicy` steps and window; no per-address or per-invitation key; the sign-in keys are never written by this feature. *(plan §2.1 `AbuseThrottlePolicy`, D10; RF-9.1, RNF-7)* — after T-01
  **Done when:** a throttle row with dimension `INVITATION_NETWORK` can be inserted, the policy yields progressive delays and a bounded window for the key, and sign-in dimension state does not affect it.

## B · Domain (pure, no Spring)

- [ ] **T-08 · `ProjectGrantSet`** — value object `projectId`, `roles` ⊆ {`AUTHOR`, `REVIEWER`, `VIEWER`}, `canRun`; `toGrantSet()` yields the `GrantSet` **[009]** holding exactly the invited roles and `canRun`; an empty role set with `canRun` true is valid and yields a `GrantSet` that plan 009 treats as inert for launch. *(plan §2.1 `ProjectGrantSet`, D7; RF-2.2, RF-5.1, RF-6.1, RF-6.3, LC-20, LC-21)* — after T-02
  **Done when:** `projectGrantSet.atMostOneProject` passes and a table test over every role subset × `canRun` yields exactly the expected `GrantSet` flags.

- [ ] **T-09 · `Invitation` aggregate [008 → 010]** — fields per plan §2.1; `isPending(now)` = state `PENDING` and `expiresAt > now`; `accept(identityId, now, key)` and `decline(now)` from `PENDING` only; `tenantRole` typed so that Owner is not admissible; a test-only factory standing in for specification 010's constructor and `revoke`. *(plan §2.1 `Invitation`, D1, D2, D9; RF-5.1, RF-5.2, RF-6.1, RF-7.2, RF-7.3, RNF-4)* — after T-08
  **Done when:** `invitation.statesAndIsPending` passes: every non-pending state and an elapsed expiry refuse `accept` and `decline`, and `accept` records identity, time and key.

- [ ] **T-10 · `TenantMembership` operations [009]** — on plan 009's aggregate: `createFromInvitation(invitation, now)` (status `ACTIVE`, version 1, the invited role, then `setGrants` with the invited `GrantSet` when a project was invited); `reactivateFromInvitation(invitation, now)` allowed from `REVOKED` only (the same row: invited role, status `ACTIVE`, `permissionVersion + 1`, then `setGrants` for the invited project only; every other revoked grant row untouched); any other transition throws. *(plan §2.1 `TenantMembership`, D6; RF-6.1, RF-6.2, RF-6.6, LC-17)* — after T-09
  **Done when:** unit tests show creation at version 1 with the invited grant as the only child, reactivation from `REVOKED` bumping the version, replacing the role and reviving only the invited project's grant row, and a throw from `ACTIVE` or `SUSPENDED`.

- [ ] **T-11 · `AcceptanceRule`** — pure function `(invitation?, now, tenantStatus, identityFacts?, existingMembership?) → ACCEPTABLE(replace) | UNUSABLE(cause)` with the causes of plan §2.1 in the fixed evaluation order: unknown → invitation state and expiry → tenant `SUSPENDED` / `DELETING` → identity mismatch → identity ineligible (not `ACTIVE` or unverified, A-27) → membership `ACTIVE` / `SUSPENDED`; `REVOKED` membership yields `ACCEPTABLE(replace = true)`; `toProblem()` maps every cause to the single code `access.invitation.unavailable`. *(plan §2.1 `AcceptanceRule`, D3, D4; RF-5.3, RF-5.4, RF-5.5, RF-5.8, RF-9.2, RNF-2, RNF-5)* — after T-10
  **Done when:** `acceptanceRule.everyConditionAndOrder` passes: one test per cause, the order is proven by inputs failing several conditions at once, and all causes produce one identical problem code.

- [ ] **T-12 · `ResolutionOutcome` and `EmailMask`** — `PRE_MATCH(tenantName, maskedEmail)`, `REVIEW(scope)`, `WRONG_IDENTITY(tenantName, maskedEmail)`, `UNAVAILABLE(cause)`; `scope` = invited email, tenant name, tenant role, project name or none, granted roles, `canRun`, the signed-in display name; never inviter, members or secret. `EmailMask` keeps the first local character and the domain (`m•••@example.test`; a one-character local part becomes `•`). *(plan §2.1 `ResolutionOutcome`, `EmailMask`, D8; RF-1.2, RF-2.1, RF-2.5, RF-3.1, RF-9.3)* — after T-08
  **Done when:** unit tests show the four outcomes serialise only their listed fields and the mask holds for one-, two- and many-character local parts and for an address with a plus tag.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-23…T-26); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-13 · Port interfaces** — published by `access`: `InvitationRepository` (`findByHash`, `lockById`, `acceptIfPending` and `declineIfPending` as single conditional updates returning the row count, `findRecordedAcceptance(key)`), `MembershipRepository` **[009]** (`findForUpdate(tenantId, identityId)`, `save`), `ProjectGrantRepository` **[009]** (`insertAll`), `PermissionFactsPort` (`membershipChanged(tenantId, identityId, permissionVersion)`, read by the C10 derivation **[006]**), `InvitationAddressCheckPort` (consumed by `identity`), `InvitationTenantLookupPort` (over T-06). Consumed: `IdentityFactsPort` (published by `identity`: normalised email, status, verified, display name), `TenantStatusPort` (published by `tenant`: status, name), `SessionAssertionPort` **[006]**, `ThrottleStore`, `Clock`, `AuditPort` **[001]**, `TokenHasher` **[002]**. No credential, factor, session-write, approval or run port exists in this feature. *(plan §2.1 Ports, D1, D5; enables RF-1 … RF-7; RF-6.4, RNF-9)* — after T-08 … T-12
  **Done when:** the application module compiles with no dependency on Spring or persistence classes (ArchUnit), and the boundary test lists no port that reaches a credential, factor, approval or run table.

- [ ] **T-14 · Invitation-scoped context helper** — `InvitationScope.open(secretHash)`: resolves the tenant id through `InvitationTenantLookupPort`, then establishes the transaction-local tenant context (`SET LOCAL` through `sharedsupport.tx`) for the current transaction only — never session-level — with the C10 shape "actor = signed-in identity or none, tenant = the invitation's, authority = accept or decline this invitation"; unknown hash → no context and the `UNKNOWN` cause. *(plan §2.1 `ResolveInvitationQuery` / `AcceptInvitationUseCase`, D5; RF-1.1, RF-6.1, RNF-3, RNF-9)* — after T-13
  **Done when:** a unit test with a recording transaction double shows the context is set inside the transaction, cleared with it, and never set for an unknown hash.

- [ ] **T-15 · `ResolveInvitationQuery` — no session** — hash the secret (`TokenHasher` **[002]**), evaluate the network throttle, open the scope (T-14), load the invitation, tenant status and name, apply `AcceptanceRule` without identity facts: pending and tenant active → `PRE_MATCH{tenantName, maskedEmail}`; otherwise `UNAVAILABLE`; changes nothing; audits `INVITATION_RESOLVED{outcome, invitationId?}`. *(plan §2.1 `ResolveInvitationQuery`, D2, D3, D8; RF-1.1, RF-1.2, RF-1.5, RF-1.6, RF-9.3, LC-1, LC-6)* — after T-14
  **Done when:** `resolve.noSessionReturnsTenantNameAndMaskOnly`, `resolve.changesNothing` and `resolve.looksUpByHashOnlyAndStoresNoSecret` pass with in-memory doubles, and an unknown hash yields `UNAVAILABLE` with the same work performed (hash, lookup, audit).

- [ ] **T-16 · `ResolveInvitationQuery` — with a session** — additionally load `IdentityFactsPort.facts(identityId)` and the identity's own membership in that tenant (an identity-scoped read under the opened scope, A-36); mismatched normalised email → `WRONG_IDENTITY{tenantName, maskedEmail}`; matched and `ACCEPTABLE` → `REVIEW(scope)` with the signed-in display name; matched and `UNUSABLE` → `UNAVAILABLE`; the same work is performed in every branch (constant-work path). *(plan §2.1 `ResolveInvitationQuery`, D3, D8; RF-1.3, RF-1.4, RF-1.9, RF-2.1, RF-2.5, RF-3.1, RF-3.4, RF-5.4, RF-5.5, RF-9.2, LC-2 … LC-5, LC-13 … LC-16)* — after T-15
  **Done when:** `resolve.matchedIdentityGetsReview` (equal, case-different, whitespace), `emailAddress.normalisationUsedForInvitedAndSession`, `resolve.otherIdentityGetsWrongIdentityWithNoScope`, `accept.activeOrSuspendedMembershipUnusableAndUntouched` (resolve half) and `resolve.acceptedOrDeclinedIsUnavailable` pass with in-memory doubles.

- [ ] **T-17 · `AcceptInvitationUseCase` — guard, locks and consumption** — requires session facts; step 1 recorded-outcome lookup by idempotency key (same key by the same identity → the recorded `ACCEPTED` answer; different key on a non-pending row → `UNAVAILABLE`); step 2 `lockById` and `findForUpdate` of the membership; step 3 open the scope (T-14); step 4 `AcceptanceRule` with tenant status and identity facts — `UNUSABLE` → audit `INVITATION_ACCEPT_REFUSED{cause}`, nothing consumed; step 5 `acceptIfPending` — zero rows → `UNAVAILABLE`. A missing acknowledgement is refused before step 1 with the field problem. *(plan §2.1 `AcceptInvitationUseCase`, D3, D4, D9; RF-2.4, RF-3.4, RF-5.3, RF-5.4, RF-5.8, RF-6.8, RNF-1, RNF-5, LC-7, LC-8, LC-13, LC-14, LC-23)* — after T-16
  **Done when:** `accept.staleReviewAfterExpiryOrRevokeIsUnavailable`, `accept.recomparesIdentityInsideTransaction`, `accept.suspendedTenantLeavesPendingThenAcceptableWhenActive` and `accept.deletingTenantUnusable` pass with in-memory doubles, and a repeated key returns the recorded answer without a second consumption.

- [ ] **T-18 · `AcceptInvitationUseCase` — effect** — steps 6–9 in the same transaction: `TenantMembership.createFromInvitation` or `reactivateFromInvitation` when the rule returned `replace`; `setGrants(projectId, ProjectGrantSet.toGrantSet())` **[009]** for the invited project, when one was invited; `PermissionFactsPort.membershipChanged`; audit `INVITATION_ACCEPTED{invitationId, tenantId, role, grantKinds, membershipReplaced}`; enqueue the outbox event `InvitationSettled{invitationId, tenantId, outcome: ACCEPTED, issuedBy}` for plan 010; return `{tenantId, projectId?}`. Creates no session; touches no identity, factor, other membership or run row; sends no message itself. *(plan §2.1 `AcceptInvitationUseCase`, D1, D4, D6, D7, D12, D13; RF-6.1 … RF-6.4, RF-6.6, RF-6.9, RF-11.3, RNF-4, RNF-10, LC-17, LC-20 … LC-22)* — after T-17
  **Done when:** `accept.oneTransactionInvitationMembershipGrantsFactsAudit`, `accept.revokedMembershipReactivatedWithNewVersionOldGrantsStayRevoked`, `accept.scopeEqualsRecordElementByElement` and `accept.ignoresExtraFieldsAppliesRecord` pass with in-memory doubles, and the recording outbox double holds exactly one `InvitationSettled{ACCEPTED}` event.

- [ ] **T-19 · `DeclineInvitationUseCase`** — requires session facts; lock, open the scope, `AcceptanceRule` must yield `ACCEPTABLE` for the signed-in identity; `declineIfPending` — zero rows → `UNAVAILABLE`; audit `INVITATION_DECLINED{invitationId}`; enqueue `InvitationSettled{…, outcome: DECLINED}` for plan 010; creates nothing, sends nothing itself. *(plan §2.1 `DeclineInvitationUseCase`, D4, D9, D13; RF-7.1, RF-7.2, RF-7.3, LC-10, LC-12)* — after T-17
  **Done when:** `decline.finalAuditedNothingElse` passes with in-memory doubles, a wrong identity or an unusable invitation is refused, a second decline meets `DECLINED` and is answered as unavailable, and the outbox double holds exactly one `InvitationSettled{DECLINED}` event.

- [ ] **T-20 · `InvitationAddressCheck` and the signup hook [002]** — `InvitationAddressCheckPort.check(secret, typedAddress) → OK | MISMATCH | UNKNOWN`, comparing normalised addresses and revealing nothing; `SignUpUseCase` **[002]** consults it when a link secret accompanies the signup form and refuses `MISMATCH` with the field problem `auth.signup.invited-address-mismatch` (no address in the problem); `UNKNOWN` behaves as a plain signup. Signup, verification and sign-in never write `tenant_invitation`. *(plan §2.2 `SignupPage` row, D11; RF-4.1, RF-4.2, RF-4.3, LC-24, LC-25)* — after T-14
  **Done when:** `signup.withSecretRefusesDifferentAddress`, `signup.withSecretAcceptsInvitedAddress` and `signupVerifySignIn.leaveInvitationPending` pass with in-memory doubles.

- [ ] **T-21 · Throttle wiring** — the three use cases evaluate `INVITATION_NETWORK` before any lookup; a restricted dimension answers the throttled problem with no cause and performs no lookup, consumption or audit beyond `RESTRICTION_APPLIED`; windows lift by time; the sign-in keys are never touched. *(plan §2.1 `AbuseThrottlePolicy`, D10; RF-9.1, RF-9.4, RNF-7, LC-33)* — after T-15, T-17, T-19
  **Done when:** `throttle.invitationKeyProgressiveBoundedLifts` and `throttle.invitationFloodNeverRestrictsSignIn` pass with an in-memory throttle store and a controllable clock.

- [ ] **T-22 · Audit event catalogue** — the event types `INVITATION_RESOLVED`, `INVITATION_ACCEPTED`, `INVITATION_DECLINED`, `INVITATION_ACCEPT_REFUSED`, `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]** with their payload shapes (identity id when a session exists, correlation id, network key, timestamp, outcome or cause); a redaction test that no payload can carry the secret, an unmatched address or a message body. *(plan §2.1 Audit adapter, D13; RF-11.1, RF-11.2)* — after T-18, T-19
  **Done when:** `audit.entryPerEvent` passes with a recording `AuditPort` for every event of RF-11.1 and `audit.containsNoSecretOrUnmatchedAddress` passes.

## D · Infrastructure adapters

- [ ] **T-23 · Persistence adapters** — `InvitationRepository`, `MembershipRepository` and `ProjectGrantRepository` over the three tables under the A-35 placeholder API; `acceptIfPending` / `declineIfPending` as single conditional `UPDATE … WHERE state = 'PENDING' AND expires_at > now()` statements; `lockById` and `findForUpdate` as `SELECT … FOR UPDATE`; every statement runs under the transaction-local tenant context. *(plan §2.1 Persistence, D4, D5, D6; RF-6.1, RF-6.8, RNF-1, RNF-10)* — after T-05, T-13
  **Done when:** `accept.oneTransactionInvitationMembershipGrantsFactsAudit`, `accept.revokedMembershipReactivatedWithNewVersionOldGrantsStayRevoked` and `accept.rowsVisibleToPendingAndMembersQueries` (query shapes stubbed from specification 010) pass against real PostgreSQL, and `accept.crashBetweenStepsLeavesNothing` shows a rollback after step 5 leaves the invitation `PENDING`.

- [ ] **T-24 · Lookup, context and permission-facts adapters** — `InvitationTenantLookupPort` over the T-06 function; the `SET LOCAL` implementation of T-14 through `sharedsupport.tx`; `PermissionFactsPort` adapter writing what the C10 derivation **[006]** reads for `permissionVersion`. *(plan §2.1 Persistence, D5, D6; RF-6.1, RF-6.6, RNF-9)* — after T-23
  **Done when:** `rls.rejectsMembershipWriteWithoutContext` and `rls.acceptWritesOnlyUnderInvitationScopedContext` pass, and `accept.nextRequestDerivesNewPermissionVersion` shows the derivation **[006]** reading the new version on the next request.

- [ ] **T-25 · `IdentityFactsPort` and `TenantStatusPort` adapters** — in `identity.infrastructure`, `IdentityFactsPort` over `IdentityRepository` **[001]** returning normalised email, status (A-27 model), verification flag and display name, never a credential, factor or session; in `tenant.infrastructure`, `TenantStatusPort` reading the tenant row's status and name. *(plan §2.1 adapters, D1; RF-5.3, RF-5.4, LC-13, LC-14, LC-28)* — after T-13
  **Done when:** `restrictedOrErasingIdentityCannotReachReview` passes, and the identity adapter's response type exposes no credential, factor or session field.

- [ ] **T-26 · Audit adapter** — the T-22 catalogue through `audit.AuditPort`, appended in the effect's transaction; correlation id from `sharedsupport.contextIds`. *(plan §2.1 Audit adapter, D13; RF-11.1, RF-11.2, RF-11.4)* — after T-22, T-23
  **Done when:** `audit.entryPerEvent` passes against real PostgreSQL with the audit row committed or rolled back together with the effect.

- [ ] **T-27 · RLS and definer suite** — the policies of T-03 … T-05, the function of T-06 and the context of T-24 verified from the runtime role: reads and writes without context rejected, cross-tenant reads under another tenant's context empty, the definer function returning one column only. *(D5; RNF-9, P12)* — after T-24
  **Done when:** `rls.rejectsMembershipWriteWithoutContext`, `rls.acceptWritesOnlyUnderInvitationScopedContext` and `securityDefiner.returnsTenantIdOnly` pass in one suite run.

## E · API

- [ ] **T-28 · `POST /api/invitations/resolve`** — body `{secret}`; session optional (facts from `SessionAssertionPort` **[006]** when present); 200 `{state: PRE_MATCH, tenantName, maskedEmail}` · 200 `{state: REVIEW, scope}` · 200 `{state: WRONG_IDENTITY, tenantName, maskedEmail}` · 410 `access.invitation.unavailable` (one code) · 429 `access.throttled` (no cause); RFC 9457 problems with a correlation id; the secret is never echoed. *(plan §2.1 `InvitationController`, D8, D15; RF-1.2 … RF-1.5, RF-2.5, RF-5.6, RF-9.1, RF-9.2, RF-11.4)* — after T-16, T-21
  **Done when:** `resolve.unknownSecretIsUnavailableOneCode` and `unavailable.oneCodeNoTenantNoCause` pass under MockMvc, and the four 200 shapes match the contract.

- [ ] **T-29 · `POST /api/invitations/accept` and `decline`** — `accept` body `{secret, acknowledged, idempotencyKey}`: 401 without a session; 400 field problem `access.invitation.acknowledgement-required` when `acknowledged` is not true; 200 `{state: ACCEPTED, tenantId, projectId?}`; 410 one code. `decline` body `{secret}`: 401 without a session; 200 `{state: DECLINED}`; 410. No role, project or grant field is accepted in either body. *(plan §2.1 `InvitationController`, D7, D9, D15; RF-2.4, RF-6.1, RF-6.3, RF-6.5, RF-7.1, RF-7.2)* — after T-18, T-19
  **Done when:** `accept.requiresSession401`, `accept.acknowledgementFieldProblem`, `accept.bodyHasNoScopeFields` and `decline.requiresSessionAndInvitedIdentity` pass under MockMvc.

- [ ] **T-30 · API contract suite** — every response and problem of T-28 and T-29 contains no secret, inviter identity, member, other project, other tenant, factor state or cause; the correlation id is present on every problem; `linkAloneGrantsNothing` (no session → no accept; a forwarded link in another identity's session → `WRONG_IDENTITY`); no linking endpoint or tenant-side acceptance endpoint exists. *(D7, D8, D15; RF-2.5, RF-3.5, RF-9.3, RF-11.4, RNF-3, LC-35)* — after T-29
  **Done when:** `reviewResponse.exposesOnlyInvitedScope`, `responses.revealNoIdentityFactorTenantListOrOtherInvitation`, `linkAloneGrantsNothing`, `noLinkingEndpointOrControlExists`, `noTenantSurfaceAcceptsForAnother` and `problems.containNoSecret` pass.

## F · Front end

- [ ] **T-31 · Contracts and hooks** — `contracts/invitation` types and runtime guards for the four resolve states, the accept and decline responses and the three problem codes; `useResolveInvitation()` (safe read: one retry on transport failure, none on 410 or 429), `useAcceptInvitation()` (idempotency key generated once per page instance and reused on re-submission, no retry), `useDeclineInvitation()` (no retry); all attach the correlation id. *(plan §2.2 Public API, State; D9, D15; RF-1 … RF-7, LC-9)* — after T-01
  **Done when:** unit tests show the guards reject malformed responses, the accept mutation reuses its key across two submissions, and no mutation retries.

- [ ] **T-32 · `InvitationReturnHolder` and the `/invitation` route** — module-private in-memory holder (secret, masked address) set on mount from the route query, the secret removed from the visible URL by a history replace; dropped on acceptance, decline, the unavailable state and "Leave invitation"; kept across "Use another identity" / "Switch identity safely"; nothing written to storage; a new tab has no holder. *(plan §2.2 `InvitationReturnHolder`, routes, D11; RF-1.1, RF-3.2, RF-4.4, RF-11.5, RNF-8, LC-26)* — after T-31
  **Done when:** `storage.containsNothingFromFeature` and `url.secretReplacedInHistory` pass, and a unit test shows the holder's drop and keep rules for each exit.

- [ ] **T-33 · i18n bundle and `SecurityStatements` [001]** — keys in `identity-access.{en,es,fr}.json` for every statement, heading, action, field label and state of the four states, the invited signup hint and statement, and "Return to your invitation"; `SecurityStatements` **[001]** extended with the N02 / M03 / N29 / D02 statements (no automatic joining by domain; membership ≠ project grants, Admin reads no project, `canRun` independent; switching identity requires signing out and the invitation is not accepted for the wrong account; expired, revoked or used invitations cannot be accepted; MFA handoff; acceptance applies only to the invited local identity; no tenant content shown). *(plan §2.2 `SecurityStatements`, `shared/i18n`, D14; RF-2.3, RF-8.1, RF-8.2, RF-8.3, RNF-13)* — after T-31
  **Done when:** `securityStatements.presentOnInvitationStates` passes and every key resolves in EN, ES and FR with no fallback to the key name.

- [ ] **T-34 · `InvitationPage` — pre-match state** — heading, subtitle, tenant name, masked address, "Existing local account" → `/sign-in` keeping the holder, "Create local account" → `/sign-up` keeping the holder, the statement that only the invited local identity can accept and the MFA statement, "← Back to sign in", Privacy / Terms routes per S-14, no Help, language switch; resolve on mount; the language switch changes display only. *(plan §2.2 `InvitationPage`, D8, D14; RF-1.2, RF-1.7, RF-1.8, RF-8.3, LC-1, LC-32)* — after T-32, T-33
  **Done when:** `invitationPage.preMatchStateWithRoutes`, `invitationPage.rendersNoHelpAndDesignedLegalRoutes` and `preMatchAndSignup.carryMfaStatement` pass.

- [ ] **T-35 · `InvitationPage` — review state, desktop (N02)** — "Join <tenant>", "Invitation available", invited address, WORKSPACE, workspace membership role, PROJECT with explicit grants and "canRun: granted / not granted" or the "membership only" variant, the inviter statement, the identity-matches block, the acknowledgement checkbox never pre-ticked with an associated message on submit without it, "Accept invitation as <display name>", "Use another identity", "Decline invitation", the MFA handoff block, "Next destination", the switching / unusable statement; no editable scope control. *(plan §2.2 `InvitationPage`, D7, D14; RF-2.1 … RF-2.4, RF-7.1, RF-8.1, RF-8.2, LC-20, LC-23)* — after T-34
  **Done when:** `invitationPage.reviewState`, `reviewState.rendersScopeBlocksStatementsAndActions` (desktop, with and without project) and `reviewState.blocksWithoutAcknowledgement` pass.

- [ ] **T-36 · `InvitationPage` — review state, mobile (M03)** — the same component at mobile widths renders M03's layout and copy ("Your invitation, clearly scoped.", WORKSPACE / membership, PROJECT / content grants, "Independent execution grant", "Current identity matches", "Switch identity safely", MFA handoff, "Next destination", "Wrong identity or unusable invitation?", "Accept invitation", "Decline invitation"); every statement, gate and action of T-35 present; touch targets 44 px. *(plan §2.2 `InvitationPage`, D14; RF-10.1, RF-10.3, RNF-14)* — after T-35
  **Done when:** `reviewState.rendersScopeBlocksStatementsAndActions` passes at tablet and mobile widths and a unit test asserts the action set at mobile equals the action set at desktop.

- [ ] **T-37 · Wrong-identity and unavailable states** — N29: "Use the invited identity", the statement, tenant name and masked address, "No protected project content is revealed", "Switch identity safely", "Leave invitation". D02: "This invitation can't be accepted", the statement, "Return to memberships →" (signed in → the landing route; signed out → `/sign-in`); the 429 problem renders the same unavailable state; both states reflow. *(plan §2.2 `InvitationPage`, D3, D8, D14; RF-1.4, RF-1.5, RF-3.1, RF-5.6, RF-9.1, RF-10.2, LC-3, LC-5, LC-6, LC-11)* — after T-34
  **Done when:** `invitationPage.wrongIdentityState`, `wrongIdentityState.showsTenantNameMaskAndTwoActions` and `invitationPage.unavailableStateRoutes` pass.

- [ ] **T-38 · Handoffs** — `ACCEPTED` → holder dropped → the tenant route `LandingRouter` **[001]** uses for `ENTER_TENANT` with `projectId` preselected when present, and `EnrollmentGateHandler` **[003]** handles `auth.enrollment.required` from that entry with `returnTo` = the tenant route; `DECLINED` and "Leave invitation" → holder dropped → the chooser route `/workspaces` of plan 013, which always renders on explicit navigation; "Use another identity" / "Switch identity safely" → the sign-out mutation of specification 007 → `/sign-in` with the holder kept. *(plan §2.2 Routing, D12; RF-3.2, RF-3.3, RF-6.5, RF-6.7, RF-7.2, LC-18 … LC-20)* — after T-35, T-37
  **Done when:** `wrongIdentity.leaveDropsHolderAndLands`, `decline.dropsHolderAndLands` and a unit test of the `ACCEPTED` navigation (with and without project) pass with routing stubs, the first two asserting navigation to `/workspaces`.

- [ ] **T-39 · Return route [001] [002]** — `InvitationReturn` detour: after `COMPLETED` of sign-in **[001]** only (a re-authentication returns to the interrupted screen, 006 RF-8.3), if the holder is present navigate to `/invitation` instead of calling `LandingRouter`; "Return to your invitation" shown above the form of `SignInPage` **[001]** and in the confirmed state of `VerifyEmailPage` **[002]** while the holder is present. *(plan §2.2 `InvitationReturn`, `SignInPage`, `VerifyEmailPage`, D11; RF-4.4, RF-4.5, LC-24, LC-31)* — after T-32
  **Done when:** unit tests show the detour fires only with a holder, the two surfaces render the route only with a holder, and `/invitation` re-resolves on arrival.

- [ ] **T-40 · `SignupPage` invited variant [002]** — when the holder is present: the masked address as the email field's hint, the statement that the account must be created with the invited address and verified before returning, the secret sent with the form; the field problem `auth.signup.invited-address-mismatch` mapped to the email field; everything else unchanged. *(plan §2.2 `SignupPage`, D11; RF-4.1, RF-4.6, LC-25)* — after T-32, T-33
  **Done when:** `signupPage.invitedVariantHintAndStatement` passes and the mismatch problem renders on the email field without revealing an address.

- [ ] **T-41 · Width and language unit tests** — statements, gates and actions present at desktop, tablet and mobile widths for the four states; the language switch keeps the resolved outcome and the holder; 44 px targets, visible labels and focus on every interactive element. *(plan §2.2, D14; RF-1.8, RF-10.1 … RF-10.3, RNF-12, RNF-13, RNF-14, LC-32)* — after T-36, T-37, T-40
  **Done when:** `allStates.existInEnEsFr` passes and the width tests assert the same statement set at the three widths.

- [ ] **T-42 · FE hygiene suite** — nothing in `localStorage`, `sessionStorage` or IndexedDB from this feature after every state and exit; no secret in analytics or error reports; the holder dropped and kept per T-32; the URL secret replaced in history before the first render. *(D11; RF-11.5, RNF-8)* — after T-38, T-39
  **Done when:** `storage.containsNothingFromFeature` and `url.secretReplacedInHistory` pass after walking every state and exit.

## G · Verification suites

- [ ] **T-43 · Timing-comparability harness** — N samples of the resolve call per unusable cause (unknown, expired, revoked, used, declined, suspended, deleting, membership active, membership suspended) and of the `PRE_MATCH` / `REVIEW` / `WRONG_IDENTITY` outcomes; distributions overlap within the configured tolerance. *(D3; RF-9.2, RNF-2)* — after T-28
  **Done when:** `unavailable.timingComparableAcrossCauses` passes on the CI runner for three consecutive runs.

- [ ] **T-44 · Concurrency suite** — two tabs accepting one invitation; accept racing a decline; accept racing a revocation (010's `revoke` stubbed as a conditional update to `REVOKED`); accept racing expiry with a controllable clock. *(D4, D9; RF-6.8, RNF-1, LC-9, LC-10)* — after T-23, T-29
  **Done when:** `accept.twoTabsOneSuccess`, `accept.vsDeclineOneTransition`, `accept.vsRevokeOneTransition` and `accept.vsExpiryOneTransition` pass ten times in a row with real PostgreSQL, each yielding exactly one membership or none.

- [ ] **T-45 · Throttle suite** — progressive delays, bounded suppression, automatic lifting with a controllable clock, an invitation flood never restricting the invited identity's sign-in, and no per-address key existing. *(D10; RF-9.1, RNF-7, LC-33)* — after T-21, T-23
  **Done when:** `throttle.invitationKeyProgressiveBoundedLifts` and `throttle.invitationFloodNeverRestrictsSignIn` pass against real PostgreSQL.

- [ ] **T-46 · Startup suite** — one case per property of T-01. *(RF-5.7, RF-9.4, RNF-6)* — after T-01
  **Done when:** `invitationPolicy.startupFailsOnAbsentOrUnboundedValue` passes, each case failing the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-47 · Fail-closed and scope suite** — every unusable condition refuses; the membership and grants created equal the record element by element; `tenant_role` cannot be Owner; the use case touches only the three `access` tables and audit (session, credential, factor, other memberships, suspension, runs unchanged); a suspended tenant's invitation waits and a deleting tenant's is final; an existing active or suspended membership blocks and is untouched; a revoked one is reactivated. *(D3, D4, D6, D7; RF-5.4, RF-5.5, RF-6.2 … RF-6.4, RNF-4, RNF-5, LC-13 … LC-17, LC-21, LC-22, LC-27)* — after T-23, T-25
  **Done when:** `failClosed.everyUnusableConditionRefuses`, `accept.scopeEqualsRecordElementByElement`, `schema.tenantRoleRejectsOwner`, `accept.touchesOnlyAccessTables`, `accept.activeOrSuspendedMembershipUnusableAndUntouched`, `restrictedOrErasingIdentityCannotReachReview` and `newIdentity.linkExpiredDuringVerificationIsUnavailable` pass.

- [ ] **T-48 · Gate-at-entry and permission-version suite** — with plan 003's `EnrollmentGate` and `FactorStatusPort` (built or stubbed in their plan's shape): an invited Admin without a factor accepts and is gated on entering the tenant; a Member under a policy requiring MFA likewise; a Member without such a policy enters; the next request after acceptance derives the new permission version; no authority exists before the gate. *(D12; RF-4.7, RF-6.6, RF-6.7, RNF-11, LC-18, LC-19)* — after T-24, T-29
  **Done when:** `accept.withoutFactorSucceeds`, `tenantEntry.adminWithoutFactorIsGated`, `tenantEntry.policyMemberWithoutFactorIsGated`, `accept.nextRequestDerivesNewPermissionVersion` and `accept.grantsNoAuthorityBeforeGate` pass.

- [ ] **T-49 · Audit and secrecy suite** — an audit entry per event of RF-11.1 committed with its effect; no secret, unmatched address or message body in audit, logs, problems or traces; the outbox untouched by accept and decline; the email channel being down has no effect on acceptance. *(D13, D15; RF-11.1 … RF-11.4, RNF-8, RNF-15, LC-34)* — after T-26, T-30
  **Done when:** `audit.entryPerEvent`, `audit.containsNoSecretOrUnmatchedAddress`, `logs.containNoSecret`, `problems.containNoSecret`, `outbox.untouchedByAcceptAndDecline` and `emailChannelDown.noEffectOnAcceptance` pass.

- [ ] **T-50 · FE e2e journeys** — Playwright journeys for US-1 … US-8 in EN, ES and FR at desktop, tablet and mobile widths, with the 001, 002, 003 and 007 surfaces built or stubbed in their plans' shapes: signed-in invitee → review → accept → tenant with project; membership-only acceptance; new person → invited signup → verification → sign-in → "Return to your invitation" → review; the same in a new tab re-opening the link from the message; wrong identity → "Switch identity safely" → sign-in → return; forwarded link to a third party; invited Admin without factor → gate; decline → landing; language switch on each state; session expiry on the review state → re-authentication → re-resolve; back navigation after acceptance. *(RF-3.2, RF-4.4 … RF-4.6, RF-6.5, RF-6.7, RF-7.2, RNF-13, LC-2, LC-3, LC-18 … LC-20, LC-24 … LC-26, LC-30 … LC-32; US-1 … US-8)* — after T-42, T-30
  **Done when:** `wrongIdentity.switchSafelySignsOutKeepsHolderReturnsAfterSignIn`, `newIdentity.returnViaConfirmedStateAndSignIn`, `newIdentity.reopensLinkFromMessageInNewTab`, `newIdentity.otherAddressGetsWrongIdentity`, `accepted.entersTenantWithProject`, `accepted.membershipOnlyEntersTenantWithoutProject`, `languageSwitch.keepsOutcomeAndHolder` and `sessionExpiryOnReview.reauthThenReresolve` pass in the three languages and three widths.

- [ ] **T-51 · Automated accessibility** — axe on the four states at the three widths and three languages. *(RF-10.1 … RF-10.3, RNF-12, RNF-13, RNF-14)* — after T-50
  **Done when:** `a11y.invitation.{en,es,fr}.{desktop,tablet,mobile}` reports zero critical or serious violations on all 36 combinations.

- [ ] **T-52 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on the four states: focus order, acknowledgement error association, state changes announced without focus theft, labels, 44 px targets; versions recorded in the release-support matrix. *(RF-10.3, RNF-12; FR-X-040)* — after T-51
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-53 · Adversarial suite (US-9)** — enumeration of tenants, addresses and invitations across resolve, accept and decline; a forwarded link in another identity's session; acceptance for an identity the attacker does not control; role and grant escalation through the request body; an Owner by invitation; reuse of an accepted, declined, expired or revoked link; a lockout of the invitee by flooding the link; a search for a tenant-side surface that accepts on another person's behalf. *(D3, D4, D7, D10; RF-1.5, RF-3.4, RF-3.5, RF-6.3, RF-6.8, RF-9.1 … RF-9.3, RNF-1 … RNF-4, RNF-7; US-9)* — after T-30, T-44, T-45
  **Done when:** every attack test fails to achieve its goal and asserts the unavailable state, the wrong-identity state or the unchanged record.

## H · Closing

- [ ] **T-54 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-43 … T-53
  **Done when:** the script reports 113 of 113 identifiers (63 RF, 15 RNF, 35 LC) covered by at least one passing test.

- [ ] **T-55 · Register-dependency review** — walk §6 of the plan (A-52, A-28, A-27, A-36, A-35; S-45, S-46, S-24 with the D11 refinement, S-03, S-04, S-14, S-17): for each, confirm the built assumption matches the current register state or record the deferral. *(spec §7 item 14)* — after T-54
  **Done when:** each of the twelve items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-56 · Completion criteria sign-off** — tick spec §7 items 1–14 with the evidence (test run id, timing report, concurrency log, a11y report, manual checklist, coverage report, register review). — after T-55
  **Done when:** all fourteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-05, T-06, T-14, T-15, T-32 |
| RF-1.2 | T-12, T-15, T-28, T-34 |
| RF-1.3 | T-16, T-28 |
| RF-1.4 | T-16, T-28, T-37 |
| RF-1.5 | T-15, T-28, T-37, T-53 |
| RF-1.6 | T-15 |
| RF-1.7 | T-34 |
| RF-1.8 | T-34, T-41 |
| RF-1.9 | T-16 |
| RF-2.1 | T-12, T-16, T-35 |
| RF-2.2 | T-08, T-35 |
| RF-2.3 | T-33 |
| RF-2.4 | T-17, T-29, T-35 |
| RF-2.5 | T-12, T-16, T-28, T-30 |
| RF-2.6 | T-15 (resolve changes nothing), T-16 |
| RF-3.1 | T-12, T-16, T-37 |
| RF-3.2 | T-32, T-38, T-50 |
| RF-3.3 | T-38 |
| RF-3.4 | T-16, T-17, T-53 |
| RF-3.5 | T-30, T-53 |
| RF-4.1 | T-20, T-40 |
| RF-4.2 | T-20 |
| RF-4.3 | T-20 |
| RF-4.4 | T-32, T-39, T-50 |
| RF-4.5 | T-39, T-50 |
| RF-4.6 | T-40, T-50 |
| RF-4.7 | T-48 |
| RF-5.1 | T-05, T-08, T-09 |
| RF-5.2 | T-05, T-09 |
| RF-5.3 | T-11, T-17, T-25 |
| RF-5.4 | T-11, T-16, T-17, T-25, T-47 |
| RF-5.5 | T-11, T-16, T-47 |
| RF-5.6 | T-28, T-37 |
| RF-5.7 | T-01, T-46 |
| RF-5.8 | T-11, T-17 |
| RF-6.1 | T-03, T-04, T-08, T-09, T-10, T-14, T-18, T-23, T-24, T-29 |
| RF-6.2 | T-03, T-04, T-10, T-18, T-47 |
| RF-6.3 | T-04, T-05, T-08, T-18, T-29, T-47, T-53 |
| RF-6.4 | T-13, T-18, T-47 |
| RF-6.5 | T-29, T-38, T-50 |
| RF-6.6 | T-03, T-10, T-18, T-24, T-48 |
| RF-6.7 | T-38, T-48, T-50 |
| RF-6.8 | T-17, T-23, T-44, T-53 |
| RF-6.9 | T-18, T-23 |
| RF-7.1 | T-19, T-29, T-35 |
| RF-7.2 | T-09, T-19, T-29, T-38, T-50 |
| RF-7.3 | T-09, T-19 |
| RF-8.1 | T-33, T-35 |
| RF-8.2 | T-33, T-35 |
| RF-8.3 | T-33, T-34 |
| RF-9.1 | T-07, T-21, T-28, T-37, T-45, T-53 |
| RF-9.2 | T-11, T-16, T-28, T-43, T-53 |
| RF-9.3 | T-12, T-15, T-30, T-53 |
| RF-9.4 | T-01, T-21, T-46 |
| RF-10.1 | T-36, T-41, T-51 |
| RF-10.2 | T-37, T-41, T-51 |
| RF-10.3 | T-36, T-41, T-51, T-52 |
| RF-10.4 | T-36, T-37 (breakpoint layout replaceable per state, S-03) |
| RF-11.1 | T-22, T-26, T-49 |
| RF-11.2 | T-22, T-26, T-49 |
| RF-11.3 | T-18, T-49 |
| RF-11.4 | T-26, T-28, T-30, T-49 |
| RF-11.5 | T-32, T-42 |
| RNF-1 | T-17, T-23, T-44, T-53 |
| RNF-2 | T-11, T-43 |
| RNF-3 | T-14, T-30, T-53 |
| RNF-4 | T-09, T-18, T-47, T-53 |
| RNF-5 | T-11, T-17, T-47 |
| RNF-6 | T-01, T-46 |
| RNF-7 | T-07, T-21, T-45, T-53 |
| RNF-8 | T-05, T-32, T-42, T-49 |
| RNF-9 | T-02, T-03, T-06, T-13, T-14, T-24, T-27 |
| RNF-10 | T-02, T-18, T-23 |
| RNF-11 | T-48 |
| RNF-12 | T-41, T-51, T-52 |
| RNF-13 | T-33, T-41, T-50, T-51 |
| RNF-14 | T-36, T-41, T-51 |
| RNF-15 | T-49 |
| LC-1 | T-15, T-34 |
| LC-2 | T-16, T-50 |
| LC-3 | T-16, T-37, T-50 |
| LC-4 | T-16 |
| LC-5 | T-16, T-37 |
| LC-6 | T-15, T-37 |
| LC-7 | T-17 |
| LC-8 | T-17 |
| LC-9 | T-31, T-44 |
| LC-10 | T-19, T-44 |
| LC-11 | T-37 |
| LC-12 | T-19 |
| LC-13 | T-16, T-17, T-25, T-47 |
| LC-14 | T-16, T-17, T-25, T-47 |
| LC-15 | T-16, T-47 |
| LC-16 | T-16, T-47 |
| LC-17 | T-10, T-18, T-47 |
| LC-18 | T-38, T-48, T-50 |
| LC-19 | T-38, T-48, T-50 |
| LC-20 | T-08, T-18, T-35, T-38, T-50 |
| LC-21 | T-08, T-18, T-47 |
| LC-22 | T-18, T-47 |
| LC-23 | T-17, T-35 |
| LC-24 | T-20, T-39, T-50 |
| LC-25 | T-20, T-40, T-50 |
| LC-26 | T-32, T-50 |
| LC-27 | T-47 |
| LC-28 | T-25 |
| LC-29 | T-15 (resolve changes nothing) |
| LC-30 | T-50 |
| LC-31 | T-39, T-50 |
| LC-32 | T-34, T-41, T-50 |
| LC-33 | T-21, T-45 |
| LC-34 | T-49 |
| LC-35 | T-30 |
| US-1 … US-8 | T-50 (journeys) |
| US-9 | T-53 (adversarial suite) |

Self-check: all 63 RF criteria, 15 RNF and 35 limit cases of `specInvitationAcceptance.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.
