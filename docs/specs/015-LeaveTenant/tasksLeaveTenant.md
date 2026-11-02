# Tasks 015 — Leave tenant and identity erasure request

**Feature** Leaving one tenant membership, and requesting the verified erasure of one's own global identity · **Tasks version** 0.1 · **Date** 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Derived from** `specLeaveTenant.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n, derived rules D-a…D-d) and `planLeaveTenant.md` v0.1 (modules §2, decisions D1–D15, tests §5, register assumptions and cross-plan items §6), governed by `constitution.md` v1.8. Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2 (plus the three 015-contract tests that plan 014 §5.2 already names). Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[003]** / **[005]** / **[006]** / **[007]** / **[008]** / **[009]** / **[010]** / **[013]** / **[014]** extend an element that plan builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Plan 016 (guarded switch) and the personal-settings plan (R12) are not written; where a task reaches them it builds against the fallback or the contract the plan names (plan D12, §6 items 7 and 8). Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API is the A-35 placeholder.

Phases: **A** foundations · **B** domain · **C** persistence and adapters · **D** `access` application (leave, checks, blocking tenants) · **E** `identity` application (erasure) · **F** fan-out and consumer contracts · **G** front end · **H** verification and closing. Phase G depends only on T-02 and T-04 (problem codes, endpoint shapes and the phrase key are fixed by the plan) and may run alongside phases B–F.

---

## A · Foundations

- [ ] **T-01 · Package placement and import rule** — erasure placed in `com.aspectra.identity` (`domain`, `application`, `api`, `infrastructure`) and leave / ownership checks / blocking tenants in `com.aspectra.access` (plan §2.1.1, §2.1.2); an ArchUnit test (test-only) asserts that `identity` reaches `access` only through `BlockingTenantsPort` and `access` reaches `identity` only through `IdentityErasureRequested`, `IdentityFactsPort` **[008]** and `StepUpPort` **[010]**. *(plan D1; RF-1.4, RF-9.2)*
  **Done when:** both modules compile and the ArchUnit rule passes, failing when a test class imports `identity.domain` from `access`.

- [ ] **T-02 · Problem codes** — RFC 9457 problem types with a correlation id and no free detail: `access.leave.not-listed` (404), `access.leave.changed` (409), `access.leave.tenant-suspended` (409), `identity.erasure.blocked{gates|tenantId}` (409), `identity.erasure.verification-failed` (400, generic), `identity.erasure.already-requested` (409), `identity.erasure.code-unsent` (503), `identity.erasure.resend-limit` (429); `access.last-owner`, `access.step-up.required` **[010]** and `mfa.step-up.*` **[003]** reused unchanged. *(plan §2.1.1 `identity.api`, §2.1.2 `access.api`, D10; RF-10.3)* — after T-01
  **Done when:** a unit test renders each problem with only type, title key, status, correlation id and its declared parameters, and a serialisation test shows no other field can be added.

- [ ] **T-03 · `ErasurePolicy` configuration** — validated record: code lifetime, code length, attempt limit, resend limit and resend window, ownership-checks page size, erasure process contact (channel and instructions text); every value required and bounded, no default; step-up limits stay in `EnrollmentPolicy` **[003]**. *(plan §2.1.1 `ErasurePolicy`, D5; RF-2.2, RF-6.3, RF-6.4, RF-8.3, RNF-9)* — after T-01
  **Done when:** `erasurePolicy.absentOrUnboundedFailsStartup` passes for each key absent, zero and unbounded, and a complete record starts the context.

- [ ] **T-04 · Typed-phrase key and `ErasurePhraseCatalog`** — message key `erasure.confirmPhrase` added to the EN / ES / FR bundles (EN "ERASE MY IDENTITY"); `ErasurePhraseCatalog` reads the same key per language at startup. *(plan §2.1.1 `ErasurePhraseCatalog`, D9; RF-5.2, RF-11.3, D-d)* — after T-01
  **Done when:** `phraseCatalog.missingLanguageFailsStartup` passes with one language removed, and the catalog returns the three phrases from the bundles.

- [ ] **T-05 · Step-up purposes [003]** — `StepUpChallenge.purpose` gains `LEAVE_TENANT` and `ERASURE`; limits from `EnrollmentPolicy` **[003]**; `StepUpPort.consume` **[010]** refuses a challenge of another purpose (plan §6 cross-plan item 3). *(plan §2.1.1 step-up purpose, D13; RF-4.3, RF-6.2, LC-7, LC-23)* — after T-01
  **Done when:** a unit test shows a `TENANT_OWNERSHIP` challenge refused for `ERASURE` and `LEAVE_TENANT` and vice versa, `stepUp.recoveryCodeRefused` passes for both new purposes, and plans 003 and 010 step-up tests still pass.

## B · Domain (pure, no Spring)

- [ ] **T-06 · `GlobalIdentity.beginErasure` [001]** — `ACTIVE → ERASING` only, any other source status refused (`NOT_ELIGIBLE`); calls `bumpSecurityVersion()` **[004]** in the same mutation; no method leaves `ERASING`. *(plan §2.1.1, D4; RF-7.2, RF-7.6, RF-7.7)* — after T-01
  **Done when:** a unit test shows the four source statuses (only `ACTIVE` accepted), the security version advanced exactly once, and `erasing.noProductExit` (domain part) finds no method whose result leaves `ERASING`.

- [ ] **T-07 · `ErasureGateRule`** — pure `(acknowledgements[2], typedPhrase, expectedPhrase, blockingTenants) → Open | Closed(gates[])` with gates `ACK_LEAVE_VS_ERASE`, `ACK_OWNERSHIP_AND_RETENTION`, `PHRASE`, `LAST_OWNER(tenantId, ACTIVE | SUSPENDED)`; phrase match exact, case-sensitive, after trimming leading and trailing white space. *(plan §2.1.1 `ErasureGateRule`, D9; RF-5.4, RF-5.5, RF-5.7, D-d, LC-16, LC-17, LC-21, LC-22)* — after T-04
  **Done when:** `gateRule.closedNamesGates` passes over every gate combination and `phrase.exactCaseTrimmedPerLanguage` (BE unit part) passes for case changes, typos, inner spaces (refused) and surrounding spaces (accepted) in each language.

- [ ] **T-08 · `ErasureCodeChallenge`** — identity, purpose fixed `ERASURE`, step-up challenge id, code hash, `expiresAt`, `attemptsRemaining`, `resendsRemaining`, states `OPEN → VERIFIED | VOIDED | SUPERSEDED`; `verify`, `recordFailedAttempt`, `void`, `supersede`; same shape as `ContinuationChallenge` **[005]**. *(plan §2.1.1, D3, D5; RF-6.4, RF-6.5, LC-24, LC-25)* — after T-03
  **Done when:** `codeChallenge.purposeErasureOnly`, `codeChallenge.resendSupersedes` and `codeChallenge.consumedOnVerify` pass, and exhausted or expired challenges verify nothing.

- [ ] **T-09 · `ErasureRequest` aggregate** — identity, `receivedAt`, proofs (`STEP_UP`, `EMAIL_CODE` with challenge ids, never a secret), handoff state `RECEIVED → HANDED_OFF`, correlation id; construction requires both proofs. *(plan §2.1.1 `ErasureRequest`, D3; RF-5.7, RF-7.2, RF-7.3, RF-10.2)* — after T-06
  **Done when:** a unit test shows construction refused with one proof missing, `HANDED_OFF` reachable only from `RECEIVED`, and no field able to hold a code or password.

- [ ] **T-10 · `LeaveRule`** — pure `(role, membershipState, tenantStatus, activeOwnersBefore) → Permit(stepUpRequired) | Deny(LAST_OWNER | TENANT_SUSPENDED_OWNER | TENANT_DELETING | NOT_LISTED)` using `ActiveOwnerRule` **[010]**. *(plan §2.1.2 `LeaveRule`, D2; RF-2.4, RF-3.2, RF-4.1, RF-4.3, D-a, D-b, LC-3, LC-9, LC-10, LC-11)* — after T-01
  **Done when:** `leaveRule.rowActionOrReason`, `leaveRule.ownerSuspendedDenied` and `leaveRule.deletingNoLeave` pass over every role × membership state × tenant status × owner-count combination.

- [ ] **T-11 · `LastOwnerGuard` change kind `LEAVE_OWNER` [010]** — new kind evaluated exactly like `REMOVE_OWNER` (plan §6 cross-plan item 2). *(plan §2.1.2, D6; RF-4.4, RNF-2)* — after T-10
  **Done when:** a unit test shows `LEAVE_OWNER` denied with one active Owner and permitted with two, and plan 010's guard tests still pass.

- [ ] **T-12 · `OwnershipCheckKind`** — pure: per listed membership → `LAST_OWNER_ACTIVE` | `LAST_OWNER_SUSPENDED` | `CO_OWNED` | `NO_OWNER_RESPONSIBILITY`, the designed check and next-step text keys, the row action from `LeaveRule`, and `TRANSFER` | `NO_ELIGIBLE_SUCCESSOR` for `LAST_OWNER_ACTIVE`. *(plan §2.1.2 `OwnershipCheckKind`, D8; RF-2.3, RF-2.4, RF-2.5, RF-3.2)* — after T-10
  **Done when:** `ownershipCheckKind.lastOwnerActiveSuspendedCoOwnedNone` passes, including a deleting tenant (state only) and a suspended membership (never last Owner).

## C · Persistence and adapters

- [ ] **T-13 · `identity_erasure_request` table and repository** — identity id unique, received_at, proofs, handoff_state, correlation id; no `tenant_id`; not granted to the tenant runtime role. *(plan §2.1.1 persistence, D10; RF-7.2, RF-7.3, RNF-8)* — after T-09
  **Done when:** an integration test on real PostgreSQL shows a second insert for one identity rejected by the unique key, and a select as the tenant runtime role is denied.

- [ ] **T-14 · `erasure_code_challenge` table and repository** — code hash, purpose, expires_at, attempts_remaining, resends_remaining, state, step_up_challenge_id. *(plan §2.1.1 persistence, D5; RF-6.4, RF-6.6, RNF-8)* — after T-08
  **Done when:** an integration test round-trips a challenge and shows only the hash stored (no column holds the clear code).

- [ ] **T-15 · Identity audit events** — `ERASURE_GATES_REFUSED{gates}`, `ERASURE_CODE_SENT`, `ERASURE_CODE_FAILED`, `ERASURE_CODE_VOIDED`, `ERASURE_BLOCKED_AT_RECORDING{tenantId}`, `ERASURE_REQUEST_RECORDED{proofs}`, `ERASURE_HANDED_OFF` through `AuditPort` **[001]**, each with identity, time, correlation id. *(plan §2.1.1 audit adapter; RF-10.2, RNF-11)* — after T-01
  **Done when:** a unit test serialises each event and a guard test rejects any payload containing a password, code or phrase field.

- [ ] **T-16 · Access audit events** — `MEMBERSHIP_LEFT{tenantId, role, previous, new}` and `LEAVE_REFUSED{reason}` through `AuditPort` **[001]**. *(plan §2.1.2 `access.infrastructure`; RF-10.1)* — after T-01
  **Done when:** a unit test serialises both events with tenant, membership, actor, time and correlation id.

- [ ] **T-17 · Message templates** — `erasure-one-time-code`, `erasure-request-received` (received, signed out everywhere, no time promised, minimised retention, process contact from `ErasurePolicy`, no acting link), `tenant-left`, `member-left` (display name and tenant, no link, no content) in EN / ES / FR, language from `IdentityLocalePort` **[014]**. *(plan §2.1.1 templates, §2.1.2; RF-6.3, RF-8.1, RF-8.2, RF-8.3, RF-8.5, RNF-6)* — after T-03
  **Done when:** `templates.noTimePromisedNoActingLink` passes for every template and language (no date or duration placeholder, no URL in the two erasure templates or `member-left`).

- [ ] **T-18 · `ErasureHandoffPort` placeholder adapter** — records the request for the platform and returns success; no HTTP surface. *(plan §2.1.1 ports, D7; RF-7.5, RF-7.7)* — after T-13
  **Done when:** a unit test shows the adapter records the request id and an ArchUnit rule finds no controller depending on the port.

## D · `access` application [009, 010]

- [ ] **T-19 · `MembershipCommands.LeaveOwnMembership` [009]** — identity-scoped command beside `RevokeMembership`: `TenantMembership.revoke` (grants revoked, version advanced once), audit `MEMBERSHIP_LEFT`, enqueue `AuthorityChanged{kind = MEMBERSHIP_REVOKED}` and `MembershipLeft{tenantId, membershipId, identityId}`; no `AdministrationRule` (plan §6 cross-plan item 1). *(plan §2.1.2, D2; RF-4.5, RF-4.6, RF-4.7)* — after T-11, T-16
  **Done when:** `leave.membershipRevokedGrantsEndedVersionOnce` passes on real PostgreSQL, and plan 009's revoke tests still pass.

- [ ] **T-20 · `LeaveTenantUseCase` — Member and Admin path** — steps 1–4, 6, 7 of plan §2.1.2: idempotency outcome **[001 D1]**, own membership by id and identity, `PermissionVersionGuard` **[009]**, tenant status `FOR SHARE` (`TenantStatusPort.statusOf` **[014]**), `LeaveRule`, then `LeaveOwnMembership`; no session call. *(plan §2.1.2, D2, D15; RF-4.1, RF-4.4, RF-4.6, RF-4.8, RNF-3, LC-1, LC-2)* — after T-19
  **Done when:** `leave.memberOrAdminRevokesOwnMembershipOnly`, `leave.identitySessionsOtherMembershipsUntouched` and `leave.noSessionEnded` pass.

- [ ] **T-21 · `LeaveTenantUseCase` — Owner path** — step 5: `StepUpPort.consume(challengeId, identityId, LEAVE_TENANT)` **[010]**, then `TenantOwnershipRepository.lockForUpdate` and `activeOwnerCount` **[010]**, then `LastOwnerGuard(LEAVE_OWNER)`. *(plan §2.1.2, D6, D13; RF-4.3, RF-4.4, RNF-2, LC-4, LC-6, LC-7)* — after T-20, T-05
  **Done when:** `leave.ownerRequiresStepUpLeaveTenant`, `leave.stepUpOtherPurposeRefused` and `leave.ownerBecameLastRefused` pass.

- [ ] **T-22 · Leave rechecks and the suspended tenant** — membership moved or version changed → `access.leave.changed`; a Member or Admin of a suspended tenant and a suspended membership may leave; every Owner of a suspended tenant refused (plan 014 D13 contracts). *(plan §2.1.2, D2; RF-4.1, RF-4.4, D-a, LC-8, LC-9, LC-11)* — after T-21
  **Done when:** `leave.recheckListedVersionStatusOwners`, `leave.suspendedTenantMemberPermitted`, `leave.suspendedMembershipPermitted` and plan 014's `leave.ownerRefusedTenantSuspended` pass.

- [ ] **T-23 · Leave idempotency and no-change paths** — same key returns the recorded outcome; an already revoked membership → `access.leave.not-listed`; a voided or failed step-up changes nothing. *(plan §2.1.2, D10; RF-4.10, RF-4.11, RNF-4, LC-12)* — after T-21
  **Done when:** `leave.sameKeyOneEffect`, `leave.alreadyRevokedNotListed` and `leave.cancelOrFailedStepUpChangesNothing` (BE part) pass.

- [ ] **T-24 · `MembershipLeftHandler`** — consumes `MembershipLeft`: `tenant-left` to the person; `member-left` to every recipient of `TenantAdministratorsQuery` **[014]** except the person, display name from `IdentityFactsPort` **[008]**. *(plan §2.1.2, D11; RF-8.1, RF-8.2)* — after T-19, T-17
  **Done when:** `leave.enqueuesTenantLeftToPerson` and `leave.enqueuesMemberLeftToOwnersAdminsExceptPerson` pass, and a relay failure leaves the membership revoked.

- [ ] **T-25 · `OwnershipChecksQuery`** — identity-scoped read through `MembershipProjectionPort` **[001/013]** (A-36 predicate): listed memberships (revoked excluded), tenant name, own role, membership state, tenant status, `OwnershipCheckKind`, row action, `SuccessorsQuery` **[010]** count; page size from `ErasurePolicy`; total count; current-tenant card data when a listed tenant is requested. *(plan §2.1.2, D8; RF-2.1, RF-2.2, RF-2.3, RF-2.5, RF-2.6, RF-2.8, RF-3.1, RF-3.4, RNF-7, LC-18, LC-34, LC-35)* — after T-12, T-03
  **Done when:** `ownershipChecks.onlyOwnListedMemberships`, `ownershipChecks.revokedExcluded`, `ownershipChecks.pagedByConfiguredSizeWithTotal` and `ownershipChecks.noEligibleSuccessorStatement` pass under RLS with a second identity's memberships present.

- [ ] **T-26 · `BlockingTenantsService`** — implements `BlockingTenantsPort`: `blockingTenants(identityId)` without lock; `lockAndCheck(identityId)` locks `TenantOwnership` of every tenant where the person holds an active Owner membership in ascending tenant-id order and recounts under the locks; deleting tenants, suspended memberships and non-Owner memberships never block. *(plan §2.1.2, D6, D8; RF-5.4, RF-5.5, RF-7.1, RNF-2, LC-16, LC-17, LC-35)* — after T-21
  **Done when:** `gates.suspendedSoleOwnerHeld` and plan 014's `erasure.heldWhileOwnedTenantSuspended` pass, and a test with memberships beyond one page still finds every blocking tenant.

- [ ] **T-27 · `access` endpoints** — `GET /api/me/ownership-checks?page&currentTenantId` and `POST /api/me/memberships/{membershipId}/leave` {expectedVersion, stepUpChallengeId?, idempotencyKey} with the responses of plan §2.1.2; identity-scoped (no tenant in the path as authority). *(plan §2.1.2 `access.api`; RF-1.1, RF-2.1, RF-4.9, RF-10.3, RNF-1)* — after T-23, T-25
  **Done when:** MockMvc tests show every status and problem of plan §2.1.2, `wasCurrentTenant` set only when the left tenant was the requested current one, and `problems.safeCodesCorrelationIdNoForeignData` (access part) passes.

## E · `identity` application (erasure)

- [ ] **T-28 · `EvaluateErasureGatesQuery` and `POST /api/account/erasure/gates`** — acknowledgements and phrase against `ErasurePhraseCatalog` for the request locale, plus `BlockingTenantsPort.blockingTenants`; answers `Open` or the named gates; requires an asserted session of an `ACTIVE` identity (`AssertSessionUseCase` **[006]**). *(plan §2.1.1, D8, D9; RF-1.1, RF-5.4, RF-5.5, RF-6.1, RNF-1)* — after T-07, T-26
  **Done when:** `gates.serverRefusesWithoutAcksOrPhrase` passes and the endpoint refuses a `RESTRICTED` or `ERASING` identity.

- [ ] **T-29 · `StartErasureVerificationUseCase` and `POST /api/account/erasure/verification`** — idempotency outcome, gates re-evaluated, `StepUpPort.consume(…, ERASURE)` **[010]**, earlier open code voided, new `ErasureCodeChallenge`, synchronous send through `TransactionalEmailPort` **[002]** committed only on accepted send; response with the masked verified address and expiry; audit `ERASURE_CODE_SENT`. *(plan §2.1.1, D3, D11; RF-6.1, RF-6.2, RF-6.3, RNF-12, LC-19, LC-26)* — after T-28, T-14, T-15, T-17
  **Done when:** `startVerification.reevaluatesGates`, `startVerification.consumesStepUpErasure`, `startVerification.codeSentBeforeCommit` and `emailUnavailable.retryableNothingRecorded` pass.

- [ ] **T-30 · `ResendErasureCodeUseCase` and its endpoint** — within `resendsRemaining` and the window: supersede, fresh code, same send rule; otherwise `identity.erasure.resend-limit`. *(plan §2.1.1; RF-6.3, RF-6.4, LC-25)* — after T-29
  **Done when:** an integration test shows the superseded code refused, the fresh one accepted, and the limit problem after the configured number of resends.

- [ ] **T-31 · `RecordErasureRequestUseCase` — verification and recording** — steps 1–5 of plan §2.1.1: idempotency outcome, `ErasureCodeChallenge.verify` (generic failure, `AbuseThrottlePolicy` **[001]** counted, voided when exhausted), `BlockingTenantsPort.lockAndCheck`, `beginErasure`, insert `ErasureRequest`, in one transaction. *(plan §2.1.1, D3, D4, D6, D10; RF-6.5, RF-7.1, RF-7.2, RF-7.3, RNF-3, RNF-4, D-c, LC-20, LC-24, LC-27)* — after T-29, T-13
  **Done when:** `record.wrongExpiredExhaustedGenericNothingRecorded`, `failure.countsOnThrottle`, `record.statusVersionRecordOneTransaction`, `record.failureLeavesNothing` and `record.oneRequestPerIdentity` pass.

- [ ] **T-32 · Recording — sessions, recovery case, outbox** — steps 6–9 in the same transaction: `RevokeSessionsPort.revokeAll(identityId, except = none, cause = IDENTITY_STATUS)` **[004/006]**, `RecoveryCaseClosurePort.close(identityId, IDENTITY_ERASING)` **[005]**, audit `ERASURE_REQUEST_RECORDED`, enqueue `IdentityErasureRequested` and `erasure-request-received`. *(plan §2.1.1, D4, D15; RF-7.4, RF-7.5, RNF-5, LC-28, LC-30)* — after T-31
  **Done when:** `record.allSessionsEndIncludingRequester` and `record.closesOpenRecoveryCase` pass, and a forced failure after step 6 rolls back status, record, sessions and case together.

- [ ] **T-33 · `ErasureHandoffRelay`** — consumes `IdentityErasureRequested`, calls `ErasureHandoffPort.handOff`, marks `HANDED_OFF`, retried by the relay, never changes identity status. *(plan §2.1.1, D7; RF-7.5, RF-7.7)* — after T-32, T-18
  **Done when:** `handoff.relayMarksHandedOff` passes, including a first failed attempt followed by success with one handoff recorded.

- [ ] **T-34 · `ErasureController` final endpoint and refusals** — `POST /api/account/erasure` {codeChallengeId, code, idempotencyKey} → 202 `{state: RECEIVED}` and the problems of plan §2.1.1; no withdraw, pause or expedite route exists; no path records a request without both proofs. *(plan §2.1.1 `identity.api`, D3; RF-5.7, RF-7.7, RF-10.3, LC-33)* — after T-32
  **Done when:** `noWithdrawEndpoint`, `noSingleActionPath` and `problems.safeCodesCorrelationIdNoForeignData` (identity part) pass.

- [ ] **T-35 · Erasure confirmation content** — `erasure-request-received` rendered with the process contact from `ErasurePolicy`, sent only to the verified address; no tenant, Owner, Admin or member recipient for any erasure event. *(plan §2.1.1 templates, D11; RF-8.3, RF-8.4, RF-8.5)* — after T-32
  **Done when:** `erasure.confirmationContentAndContact` and `erasure.noTenantRecipient` pass.

## F · Fan-out and consumer contracts

- [ ] **T-36 · `IdentityErasureHandler`** — consumes `IdentityErasureRequested`: one `AuthorityChanged{tenantId, membershipId, kind = IDENTITY_ERASING}` per non-revoked membership, handled by `AuthorityChangedHandler` **[009]**; no membership, grant or invitation row changed; idempotent per request id (plan §6 cross-plan item 1). *(plan §2.1.2, D4; RF-7.8, RF-7.9)* — after T-32
  **Done when:** `erasure.authorityChangedPerMembership` and `erasure.membershipsGrantsUnchanged` pass, and a redelivered event enqueues nothing more.

- [ ] **T-37 · Work revocation on leave (recording stubs)** — `AuthorityChanged{MEMBERSHIP_REVOKED}` from a leave reaches `WorkRevocationPort.refuseQueued` / `requestBoundedCancel` and `SubscriptionRevocationPort` **[009]** for that tenant only. *(plan §2.1.3; RF-4.7, LC-13)* — after T-19
  **Done when:** `leave.authorityChangedRevokesWorkInTenant` passes against recording stubs, with no call for another tenant.

- [ ] **T-38 · `ERASING` contracts with existing features** — recording stubs or the real features of plans 001, 003, 004, 005, 006, 008, 009, 010 and 012: sign-in generic, reset and recovery create and send nothing, enrollment and workspace creation not started, invitations to the address unusable while issued ones stay pending, derivation yields nothing, not an active Owner. *(plan §2.1.3; RF-7.6, RF-7.8, RF-7.9, LC-29, LC-31, LC-32)* — after T-32
  **Done when:** `erasing.signInResetRecoveryEnrollmentCreationAcceptanceGeneric`, `erasing.derivationYieldsNothing`, `erasing.notActiveOwner`, `erasure.invitationsToAddressUnusable` and `erasure.issuedInvitationsStayPending` pass.

- [ ] **T-39 · Cross-plan FE extensions [010, 007]** — `StepUpFields` **[010]** gains a `purpose` property (default `TENANT_OWNERSHIP` for existing callers); `SignOutCoordinator` **[007]** gains entry `N19` for `fromEndedSession()` (plan §6 cross-plan items 2 and 5). *(plan D13, D15; RF-4.3, RF-6.2, RF-7.4)* — after T-05
  **Done when:** Vitest shows `StepUpFields` posting the given purpose, plan 010's and 007's FE tests still pass, and `fromEndedSession` from `N19` lands on the signed-out state.

## G · Front end (`src/features/identity-access`, `src/features/workspace-members` [010] exports, `shared/*`)

- [ ] **T-40 · Contracts and data hooks** — `contracts/identity-erasure` and `contracts/access-leave` typed shapes; contract guard rejecting unexpected fields in the ownership-checks response; `useOwnershipChecks(page, currentTenantId)` (refetch on window focus and before "Continue"), `useLeaveTenant()`, erasure mutations (gates, verification, resend, record) on axios. *(plan §2.2, D8; RF-2.1, RF-2.6, RNF-7, LC-19)* — after T-02
  **Done when:** `contractGuard.rejectsExtraFields` passes and a hook test shows the checks refetched on focus.

- [ ] **T-41 · Route `/account/privacy` and `PrivacyStatements`** — `AccountPrivacyPage` (N19) with `currentTenantId` as requested scope only; session guard to sign-in; the designed statements verbatim via `shared/i18n`; focus to the heading on open; entry from the Privacy slot of `SecuritySettingsPage` **[003]**. *(plan §2.2; RF-1.1, RF-1.2, RF-1.5, RF-2.7, RF-9.3, RF-8.5, RNF-6)* — after T-40
  **Done when:** `privacy.noSessionToSignIn`, `n19.rendersDesignedStatementsVerbatim` and `templatesAndPage.noInstantErasureOrDuration` (page part) pass.

- [ ] **T-42 · `OwnershipChecksTable`** — columns YOUR MEMBERSHIP / OWNERSHIP CHECK / REQUIRED NEXT STEP plus the row action as the server states; "Previous / Next" and record count; "Review ownership transfer" opens `TransferOwnershipDialog` **[010]** for that row's tenant; no-successor statement; no-membership variant shows the erasure section only. *(plan §2.2, D8; RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.8, LC-18, LC-34)* — after T-41
  **Done when:** `row.reviewTransferOpensN23ForThatTenant` and `n19.noMembershipShowsErasureOnly` pass.

- [ ] **T-43 · `CurrentTenantLeaveCard`** — effect text; "Blocked · last Owner" with "Review ownership transfer"; or "Leave <tenant>"; hidden without a current tenant. *(plan §2.2; RF-3.1, RF-3.2, RF-3.3, RF-3.4, LC-3)* — after T-41
  **Done when:** `currentCard.leaveOrBlockedOrHidden` passes for the three variants.

- [ ] **T-44 · `LeaveTenantDialog`** — N23 anatomy **[010]**: heading "Leave <tenant>?", the five effect statements, "Leave tenant" / "Cancel"; for an Owner `StepUpFields` with `purpose = LEAVE_TENANT` before confirm; refusal reasons rendered and the row refreshed; resubmit with the same idempotency key. *(plan §2.2, D13; RF-4.2, RF-4.3, RF-4.4, RF-4.10, RF-4.11, LC-4, LC-6, LC-7)* — after T-42, T-39
  **Done when:** `leaveDialog.statesFiveEffects`, `leaveDialog.ownerStepUpVariant` and `leave.cancelOrFailedStepUpChangesNothing` (FE part) pass.

- [ ] **T-45 · Landing after a leave** — `useLeaveTenant` success with `wasCurrentTenant` → the FR-IDN-130 scope-epoch fallback in `shared/security` (abort requests, close streams, advance the epoch, clear the scoped cache) then `/workspaces` **[013]**; otherwise invalidate the checks and show the confirmation in place (plan D12, §6 cross-plan item 7). *(plan §2.2, D12; RF-4.8, RF-4.9, LC-1, LC-2)* — after T-44
  **Done when:** `leave.currentTenantGuardedSwitchToChooser` and `leave.otherTenantStaysOnN19` pass, and the scoped cache holds no entry of the left tenant afterwards.

- [ ] **T-46 · `ErasureCard` and `ErasureRequestPanel`** — effect, marker, Admin statement; own verified address; two unticked acknowledgements; localized phrase field; "Keep my account" clears; "Continue to identity verification" enabled only when the local match and `POST /gates` agree; blocked and held statements naming the gates; state in component state only, reset on mount and after any failure. *(plan §2.2, D9, D14; RF-5.1, RF-5.2, RF-5.3, RF-5.4, RF-5.5, RF-5.6, D-c, D-d, LC-16, LC-17, LC-21, LC-38)* — after T-41
  **Done when:** `erasureCard.statements`, `panel.ownAddressAcksPhraseLabelLocalized`, `panel.startsEmptyOnEveryMount`, `panel.nothingInStorage` and `panel.keepMyAccountClears` pass.

- [ ] **T-47 · `ErasureVerification` steps** — step 1 `StepUpFields` with `purpose = ERASURE`; step 2 code entry (masked address, expiry, "Resend code"); one `GenericFailureMessage` **[001]** for any failure then back to the cleared panel; step 3 request-received statements, then `SignOutCoordinator.fromEndedSession()` with entry `N19`. *(plan §2.2, D3, D14, D15; RF-6.1, RF-6.2, RF-6.3, RF-6.5, RF-6.6, RF-7.4, LC-24, LC-25)* — after T-46, T-39
  **Done when:** `failure.backToClearedPanel` and `received.thenSignedOut` pass, and a storage scan after each step finds nothing.

- [ ] **T-48 · ES and FR bundles** — every N19 statement, card, table text, dialog, step, problem title and the phrase in ES and FR. *(plan §2.2 `shared/i18n`; RF-11.3, D-d)* — after T-47
  **Done when:** `languages.enEsFrComplete` passes (no missing or untranslated key in the feature's namespaces).

- [ ] **T-49 · R12 entry contract** — in the shape plan §6 item 8 states (no owning plan yet): R12 "Your membership" statement and the "Account privacy" tab / "Account privacy options" linking to `/account/privacy`, built against the R12 shell if it exists or a minimal host page otherwise. *(plan §2.2, §6; RF-1.2, RF-1.3)* — after T-41
  **Done when:** `r12.yourMembershipStatement` and `privacy.opensFromSecuritySettingsAndR12WithOrWithoutTenant` pass.

## H · Verification and closing

- [ ] **T-50 · Leave journeys** — Playwright: Member leaves the current tenant, Admin leaves another, co-Owner leaves with step-up, Member leaves a suspended tenant, a second tab meets the denied scope, the leaver is invited again, a suspended-then-reinstated tenant no longer lists the leaver. *(plan §5.2; RF-4.1, RF-4.8, RF-4.9, LC-1, LC-2, LC-4, LC-8, LC-14, US-1, US-2, US-3, US-5)* — after T-45, T-27
  **Done when:** `otherTab.deniedScopeAfterLeave`, `leftThenReinvited.newMembershipNoGrantRestored` and plan 014's `leftThenReinstated.notAMember` pass, with the journeys above green.

- [ ] **T-51 · Erasure journeys** — Playwright: happy path to signed out everywhere; transfer done in another tab then proceed; a second browser ends at next use; sign-in afterwards generic; the page refused to a restricted identity. *(plan §5.2; RF-6.1, RF-7.4, RF-7.6, LC-15, LC-19, LC-28, LC-29, LC-36, US-4, US-6, US-7)* — after T-47, T-34
  **Done when:** `erasure.happyPath`, `transferElsewhere.thenProceeds`, `otherBrowser.endsAtNextUse` and `privacy.restrictedOrErasingRefused` pass.

- [ ] **T-52 · Concurrency on real PostgreSQL** — two co-Owners leave at once; recording while a co-Owner leaves; two multi-tenant operations locking overlapping tenants. *(plan §5.2, D6; RF-7.1, RNF-2, LC-5, LC-20)* — after T-31, T-21
  **Done when:** `coOwners.leaveTogetherOneRefused`, `record.coOwnerLeftMeanwhileRefusedNamesTenant` and `lockOrder.noDeadlockAcrossTenants` pass on 200 repetitions with no flaky outcome.

- [ ] **T-53 · Isolation, secrets, audit and untouched rows** — integration suites with a second identity and tenant present: no foreign data in checks, problems or messages; no secret in audit, logs or storage; every leave, refusal and erasure event audited; runs, approvals, revisions and evidence unchanged; decisions derived on the server per request; no instant-erasure or duration wording. *(plan §5.2; RF-2.6, RF-6.6, RF-9.2, RF-9.4, RF-10.1, RF-10.2, RNF-1, RNF-6, RNF-7, RNF-8, RNF-11, US-8, US-9)* — after T-35, T-36
  **Done when:** `secrets.neverInAuditLogsOrStorage`, `audit.leaveAndRefusals`, `audit.erasureEventsWithoutSecrets`, `operations.touchOnlyOwnRows`, `decisions.serverDerivedPerRequest` and `templatesAndPage.noInstantErasureOrDuration` pass.

- [ ] **T-54 · Architecture and entry-point suite** — ArchUnit: leave and erasure reachable only through `LeaveController` and `ErasureController`, no administrator path; Vitest: the chooser, the members surface and tenant screens offer neither. *(plan §5.2, D1, D2; RF-1.4, RF-9.1, LC-37)* — after T-34, T-27
  **Done when:** `arch.onlyTwoControllersLeaveOrErase` and `chooserMembersTenantScreens.offerNoLeaveOrErase` pass.

- [ ] **T-55 · Accessibility and reflow** — Playwright + axe on N19 (with and without current tenant, no-membership variant, blocked and held rows), the leave dialog (both variants) and every verification step at desktop, tablet and mobile widths in EN / ES / FR; manual NVDA and VoiceOver checklist: focus to heading, reading order, labelled fields and associated errors, state as text, 44 px targets. *(plan §5.2; RF-11.1, RF-11.2, RF-11.3, RNF-10, LC-39, US-10)* — after T-48
  **Done when:** `a11y.n19DialogStepsAA` and `n19.reflowTableAsRecordsAtMobileAndTablet` pass with zero axe violations, `phrase.exactCaseTrimmedPerLanguage` (e2e part) passes in the three languages, and the signed manual checklist is attached to the release evidence.

- [ ] **T-56 · Journeys and adversarial suite** — US-1 … US-10 walked end to end on desktop and in the reflows; US-11 adversarial cases: one-click or session-only erasure, a stolen session used after recording, acting for another person, last-Owner escape through leave, erasure or a race, foreign data through the checks, withdraw or replay of a request, sign-in while erasing. *(plan §5.2; US-1 … US-11, RF-5.7, RF-7.7, RF-9.1, RNF-5)* — after T-50 … T-55
  **Done when:** every journey passes and every adversarial case is refused with a safe problem or shows nothing, recorded in the release evidence.

- [ ] **T-57 · Closing** — the register items of plan §6 (A-64, A-27, A-48, A-30, A-56, A-11, A-36, A-35; S-63, S-14, S-02, S-04) are resolved or explicitly deferred by the product owner; the eight cross-plan items of plan §6 are recorded in plans 009, 010, 003, 005, 007, 014, 016 and the personal-settings plan; specification §7 items 1–9 checked off. *(plan §5.4, §6; spec §7)* — after T-56
  **Done when:** each register item shows "resolved" or "deferred by the product owner" with a date, each cross-plan item is present in its owning document, and the §7 checklist is complete.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-27, T-28, T-41 |
| RF-1.2 | T-41, T-49 |
| RF-1.3 | T-49 |
| RF-1.4 | T-01, T-54 |
| RF-1.5 | T-41 |
| RF-2.1 | T-25, T-27, T-40, T-42 |
| RF-2.2 | T-03, T-25, T-42 |
| RF-2.3 | T-12, T-25, T-42 |
| RF-2.4 | T-10, T-12, T-42 |
| RF-2.5 | T-12, T-25, T-42 |
| RF-2.6 | T-25, T-40, T-53 |
| RF-2.7 | T-41 |
| RF-2.8 | T-25, T-42 |
| RF-3.1 | T-25, T-43 |
| RF-3.2 | T-10, T-12, T-43 |
| RF-3.3 | T-43 |
| RF-3.4 | T-25, T-43 |
| RF-4.1 | T-10, T-20, T-22, T-50 |
| RF-4.2 | T-44 |
| RF-4.3 | T-05, T-10, T-21, T-39, T-44 |
| RF-4.4 | T-11, T-20, T-21, T-22, T-44 |
| RF-4.5 | T-19 |
| RF-4.6 | T-19, T-20 |
| RF-4.7 | T-19, T-37 |
| RF-4.8 | T-20, T-45, T-50 |
| RF-4.9 | T-27, T-45, T-50 |
| RF-4.10 | T-23, T-44 |
| RF-4.11 | T-23, T-44 |
| RF-5.1 | T-46 |
| RF-5.2 | T-04, T-46 |
| RF-5.3 | T-46 |
| RF-5.4 | T-07, T-26, T-28, T-46 |
| RF-5.5 | T-07, T-26, T-28, T-46 |
| RF-5.6 | T-46 |
| RF-5.7 | T-07, T-09, T-34, T-56 |
| RF-6.1 | T-28, T-29, T-47, T-51 |
| RF-6.2 | T-05, T-29, T-39, T-47 |
| RF-6.3 | T-03, T-17, T-29, T-30, T-47 |
| RF-6.4 | T-03, T-08, T-14, T-30 |
| RF-6.5 | T-08, T-31, T-47 |
| RF-6.6 | T-14, T-47, T-53 |
| RF-7.1 | T-26, T-31, T-52 |
| RF-7.2 | T-06, T-09, T-13, T-31 |
| RF-7.3 | T-09, T-13, T-31 |
| RF-7.4 | T-32, T-39, T-47, T-51 |
| RF-7.5 | T-18, T-32, T-33 |
| RF-7.6 | T-06, T-38, T-51 |
| RF-7.7 | T-06, T-18, T-33, T-34, T-56 |
| RF-7.8 | T-36, T-38 |
| RF-7.9 | T-36, T-38 |
| RF-8.1 | T-17, T-24 |
| RF-8.2 | T-17, T-24 |
| RF-8.3 | T-03, T-17, T-35 |
| RF-8.4 | T-35 |
| RF-8.5 | T-17, T-35, T-41 |
| RF-9.1 | T-54, T-56 |
| RF-9.2 | T-01, T-53 |
| RF-9.3 | T-41 |
| RF-9.4 | T-53 |
| RF-10.1 | T-16, T-53 |
| RF-10.2 | T-09, T-15, T-53 |
| RF-10.3 | T-02, T-27, T-34 |
| RF-11.1 | T-55 |
| RF-11.2 | T-55 |
| RF-11.3 | T-04, T-48, T-55 |
| RNF-1 | T-27, T-28, T-53 |
| RNF-2 | T-11, T-21, T-26, T-52 |
| RNF-3 | T-20, T-31 |
| RNF-4 | T-23, T-31 |
| RNF-5 | T-32, T-56 |
| RNF-6 | T-17, T-41, T-53 |
| RNF-7 | T-25, T-40, T-53 |
| RNF-8 | T-13, T-14, T-53 |
| RNF-9 | T-03 |
| RNF-10 | T-55 |
| RNF-11 | T-15, T-53 |
| RNF-12 | T-29 |
| LC-1 | T-20, T-45, T-50 |
| LC-2 | T-20, T-45, T-50 |
| LC-3 | T-10, T-43 |
| LC-4 | T-21, T-44, T-50 |
| LC-5 | T-52 |
| LC-6 | T-21, T-44 |
| LC-7 | T-05, T-21, T-44 |
| LC-8 | T-22, T-50 |
| LC-9 | T-10, T-22 |
| LC-10 | T-10 |
| LC-11 | T-10, T-22 |
| LC-12 | T-23 |
| LC-13 | T-37 |
| LC-14 | T-50 |
| LC-15 | T-51 |
| LC-16 | T-07, T-26, T-46 |
| LC-17 | T-07, T-26, T-46 |
| LC-18 | T-25, T-42 |
| LC-19 | T-29, T-40, T-51 |
| LC-20 | T-31, T-52 |
| LC-21 | T-07, T-46 |
| LC-22 | T-07 |
| LC-23 | T-05 |
| LC-24 | T-08, T-31, T-47 |
| LC-25 | T-08, T-30, T-47 |
| LC-26 | T-29 |
| LC-27 | T-31 |
| LC-28 | T-32, T-51 |
| LC-29 | T-38, T-51 |
| LC-30 | T-32 |
| LC-31 | T-38 |
| LC-32 | T-38 |
| LC-33 | T-34 |
| LC-34 | T-25, T-42 |
| LC-35 | T-25, T-26 |
| LC-36 | T-51 |
| LC-37 | T-54 |
| LC-38 | T-46 |
| LC-39 | T-55 |
| US-1 | T-50, T-56 |
| US-2 | T-50, T-56 |
| US-3 | T-50, T-56 |
| US-4 | T-51, T-56 |
| US-5 | T-50, T-56 |
| US-6 | T-51, T-56 |
| US-7 | T-51, T-56 |
| US-8 | T-53, T-56 |
| US-9 | T-53, T-56 |
| US-10 | T-55, T-56 |
| US-11 | T-56 |
| D-a | T-10, T-22 |
| D-b | T-10 |
| D-c | T-31, T-46 |
| D-d | T-04, T-07, T-46, T-48 |

Self-check: all 65 RF criteria, 12 RNF, 39 limit cases, US-1 … US-11 and the derived rules D-a … D-d of `specLeaveTenant.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line; every "after" points to an earlier task.
