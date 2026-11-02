# Tasks 005 — Controlled account recovery

**Feature** Recovery of a lost authenticator: an unused recovery code, a verified recovery request, and the continuation of a recovery case · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specControlledAccountRecovery.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planControlledAccountRecovery.md` v0.1 (modules §2, decisions D1–D17, tests §5, register assumptions §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[002]** / **[003]** / **[004]** extend an element that plan 001 (Login), 002 (Signup and verification), 003 (Enrollment) or 004 (Password reset) builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plans 001–004's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-01 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Recovery configuration record** — `RecoveryPolicy` value object and `RecoveryPolicyProperties` binding: case lifetime, approval validity, one-time-code lifetime, length and attempt limit, recovery-transaction expiry and attempt limit, verified-window idle time, throttle steps and windows for `RECOVERY_ADDRESS`, `RECOVERY_REFERENCE` and `RECOVERY_NETWORK`, per-address caps and windows for re-sends, one-time codes and notifications, the public origin (the same configuration value `SignupPolicy` **[002]** validates), default message language. Each value `@Validated` as present, positive and bounded above by a configured ceiling; the code length at least a configured floor; the origin an absolute `https` origin; the language one of EN / ES / FR. *(plan §2.1 `RecoveryPolicy`, `RecoveryPolicyProperties`, D15; RF-8.6, RF-8.7, RNF-6, RNF-7)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed or set unbounded, for a code length below the floor, for a non-`https` origin and for a language outside EN / ES / FR.

- [ ] **T-02 · Migration: `recovery_case`** — columns `id`, `identity_id`, `state`, `reference_hash` (unique), `opened_at`, `expires_at`, `review_started_at`, `reviewed_by`, `approved_at`, `approved_by`, `approval_expires_at`, `closed_at`, `close_cause`, `request_locale`; **partial unique index** on `identity_id WHERE state IN ('RECEIVED','UNDER_REVIEW','APPROVED')`; `identity_id` cascades on identity delete. *(plan §2.1 Persistence, D5; RF-4.4, RF-4.8, RF-5.1, RNF-5, LC-14)* — after T-01
  **Done when:** the migration applies on an empty database and on one with existing identities; a second open case for the same identity is rejected while a closed row plus a new open row is accepted; `reference_hash` uniqueness is enforced.

- [ ] **T-03 · Migration: `recovery_continuation`** — columns `id`, `case_id` (nullable, for decoys), `code_hash`, `expires_at`, `attempts_remaining`, `state`, `verified_until`, `created_at`, the recorded outcome per idempotency key; index on `case_id`. *(plan §2.1 Persistence, D6, D12; RF-6.2, RF-6.4, RF-6.9, RNF-3)* — after T-02
  **Done when:** the migration applies; a row with `case_id` null is insertable; a row referencing a deleted case is removed with it; the outcome record enforces one row per key.

- [ ] **T-04 · Migration: transaction, pending-enrollment and throttle extensions** — `login_transaction` **[001]** accepts the purpose value `RECOVERY`, the state value `REPLACEMENT_PENDING` and gains `pending_enrollment_id`; `pending_enrollment` **[003]** replaces `owning_session_id` with `owner_kind` + `owner_id` (existing rows migrated to `SESSION`) and gains `started_by` + `case_id`; `signin_throttle` **[001]** keeps its shape and receives the three recovery dimension values. *(plan §2.1 Persistence, D3, D4, D9; RF-2.5, RF-2.6, RF-3.1, RF-8.1)* — after T-03
  **Done when:** the migration applies on a database holding plan 003 pending rows, which read back with `owner_kind = SESSION` and their former session id; a transaction row in state `REPLACEMENT_PENDING` with purpose `RECOVERY` and a throttle row with dimension `RECOVERY_REFERENCE` can be inserted; plan 001's and 003's rows are unaffected.

- [ ] **T-05 · Outbox templates registered** — register template ids `recovery-code-used`, `recovery-case-opened` (reference and a link to the recovery screen on the public origin), `recovery-one-time-code`, `recovery-case-approved` (validity, the continuation steps, nothing changed yet), `recovery-case-closed` (reason: rejected, expired, recovered by another route, no longer eligible) and the *through account recovery* variant of `authenticator-replaced` **[003]**, with the outbox message type and the three locales, the D04 copy (`437:4431`) per S-14 carrying the statements of RF-2.8, RF-3.4, RF-4.4, RF-5.3, RF-5.4, RF-5.6 and RF-6.2; none promises a completion time. *(plan §2.1 Outbox relay and email adapter, D11; RF-2.8, RF-3.4, RF-4.4, RF-5.3, RF-5.4, RF-5.5, RF-5.6, RF-6.2, RF-11.5, RNF-12, RNF-14)* — after T-01
  **Done when:** `templates.fiveMessagesAndReplacedVariantCarryStatementsNoTime` and `messages.promiseNoCompletionTime` pass in EN, ES and FR with sample variables, producing a non-empty subject and body for each of the eighteen renderings.

## B · Domain (pure, no Spring)

- [ ] **T-06 · `LoginTransaction` recovery purpose [001]** — purpose `RECOVERY`, state `REPLACEMENT_PENDING` with the owned pending enrollment id; `openForRecovery(identityId, pendingId, now, policy)`; `complete` allowed only when the owned enrollment is acknowledged; `void` with the reasons `ABANDONED`, `EXPIRED`, `REPLACEMENT_DISCARDED` beside `CREDENTIAL_CHANGED` and `FACTOR_CHANGED` **[001]**; completion re-stamps the stored `securityVersion` so the transaction survives its own factor swap. *(plan §2.1 `LoginTransaction`, D3; RF-2.5, RF-3.5, RF-3.7, RNF-8, LC-4, LC-29)*
  **Done when:** unit tests show the open, complete and void transitions, a throw on `complete` while the enrollment is unacknowledged, a throw on any challenge-side transition, and that a version bump caused by its own completion does not void it while an older version does.

- [ ] **T-07 · `PendingEnrollment` polymorphic owner [003]** — `owner = SESSION(sessionId) | RECOVERY_TRANSACTION(transactionId)`, `startedBy = STEP_UP | RECOVERY_CODE | RECOVERY_CASE(caseId)`; every ownership check compares the owner; state machine, steps, attempts, expiry and discard reasons unchanged. *(plan §2.1 `PendingEnrollment`, D4; RF-2.6, RF-3.1, RF-3.2, RF-3.7, LC-32)*
  **Done when:** unit tests show a call from a different session or a different recovery transaction refused for each owner kind, `startedBy` retained through the state machine, and plan 003's existing `PendingEnrollment` unit tests still pass unchanged.

- [ ] **T-08 · `RecoveryCase` aggregate** — identity id, state machine `RECEIVED → UNDER_REVIEW → APPROVED | REJECTED`, `RECEIVED` / `UNDER_REVIEW → EXPIRED | CLOSED`, `APPROVED → COMPLETED | EXPIRED | CLOSED`; reference hash, times, `reviewedBy`, `approvedBy`, `approvalExpiresAt`, `closeCause`, `requestLocale`; methods `moveToReview`, `approve(actor, validity, now)`, `reject`, `expire`, `close(cause, now)`, `complete`, `isOpen()`, `isApprovedAndValid(now)`. *(plan §2.1 `RecoveryCase`, D5; RF-4.4, RF-5.1, RF-5.3, RF-5.4, RF-5.6, RF-5.8, RF-6.10, LC-17, LC-21, LC-27)*
  **Done when:** `recoveryCase.stateMachineWithTimesAndActors` passes, every transition not in the machine throws, `isApprovedAndValid` turns false at the validity boundary, and the cause of every closure is retained.

- [ ] **T-09 · `ContinuationChallenge`** — `caseId` (absent for a decoy), one-time code hash, `expiresAt`, `attemptsRemaining`, state `OPEN → VERIFIED(verifiedUntil) | VOIDED`, recorded outcome per idempotency key; `verify(codeOk, now, idleWindow)`, `recordFailedAttempt`, `void`, `isVerified(now)`, `recordedOutcome(key)`. *(plan §2.1 `ContinuationChallenge`, D6, D12; RF-6.2, RF-6.3, RF-6.4, RF-6.9, LC-22, LC-23, LC-24, LC-30)*
  **Done when:** unit tests cover verify, failed attempts to zero, expiry, void, the verified window ending, a decoy that never verifies, and a recorded outcome returned for a repeated key.

- [ ] **T-10 · Recovery throttle dimensions on `AbuseThrottlePolicy` [001]** — keys `RECOVERY_ADDRESS`, `RECOVERY_REFERENCE`, `RECOVERY_NETWORK` fed by the `RecoveryPolicy` steps and windows; no change to the arithmetic; no exemption parameter. *(plan §2.1 `AbuseThrottlePolicy`, D9; RF-2.3, RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-8.5, RNF-2, LC-3, LC-16)* — after T-01
  **Done when:** `throttlePolicy.recoveryKeysProgressiveSteps` passes, and sign-in and reset dimension state does not affect a recovery decision and vice versa.

- [ ] **T-11 · `CaseView` value object** — the projection returned after a verified continuation: `state`, masked address, `requestedAt`, `outcome` for a case closed since the continuation started, `canContinue`; nothing else can be added to it by construction. *(plan §2.1 `VerifyContinuationCodeUseCase`, D6; RF-5.5, RF-6.5, RF-6.6, RF-6.7)* — after T-08
  **Done when:** a unit test shows the masking shape for short and long local parts and that serialising the object yields exactly the five fields.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-26…T-30); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-12 · Port interfaces** — new: `RecoveryCaseRepository` (`findOpenByIdentity`, `findOpenByReferenceHash`, `save`, `claimExpired`), `ContinuationChallengeRepository`, `RecoveryCaseClosurePort` (published). Reused: `IdentityRepository`, `LoginTransactionRepository`, the `identity_recovery_code` persistence behind `RecoveryCodeSet`, `ThrottleStore`, `PasswordVerifier`, `Clock`, `AuditPort`, `IssueSessionPort` **[001]**; `TokenGenerator`, `TokenHasher`, `TransactionalEmailPort`, `SecurityNotificationPort`, `JobScheduler` **[002]**; `PendingEnrollmentRepository`, `CurrentSessionPort` **[003]**; `RevokeSessionsPort` **[004]**. No tenant, membership, grant, approval or run port exists. *(plan §2.1 Ports, D5; enables RF-2 … RF-6; RF-3.8, RF-5.9, RF-9.1, RF-9.2)* — after T-06…T-11
  **Done when:** the application module compiles with no dependency on Spring or persistence classes (ArchUnit module-boundary test, test-only), and the boundary test lists no port that reaches a tenant, membership, grant, approval or run table.

- [ ] **T-13 · `CloseRecoveryCaseService` / `RecoveryCaseClosurePort`** — `close(identityId, cause)` moves any open case to `CLOSED` with the cause, enqueues `recovery-case-closed` in the case's stored locale and audits; `complete(identityId)` moves it to `COMPLETED` without a closure message; both idempotent when no case is open. *(plan §2.1 `CloseRecoveryCaseService`, D5, D11; RF-5.6, RF-5.7, RF-11.1)* — after T-12
  **Done when:** a unit test with recording doubles shows each cause recorded, the message enqueued in the stored locale for `close` and not for `complete`, and no write when no case is open.

- [ ] **T-14 · `VerifyRecoveryCodeForReplacementUseCase` — constant-work failure path** — normalise with `EmailAddress` **[002]**; evaluate `RECOVERY_ADDRESS` and `RECOVERY_NETWORK`; one password verification in every branch (real for an `isResetEligible()` **[004]** identity, `verifyAgainstDummy()` **[001]** otherwise); one code comparison bounded by the configured code count in every branch (the identity's unused codes padded, or a dummy set); any failure → `auth.recovery.rejected`, nothing consumed, recovery counters written, the sign-in keys `ACCOUNT` / `NETWORK` **[001]** written for a wrong password, audit `RECOVERY_CODE_REJECTED{matched}`. *(plan §2.1 `VerifyRecoveryCodeForReplacementUseCase`, D1, D9; RF-1.8, RF-2.2, RF-2.3, RF-2.7, RF-7.1, RF-7.2, RF-8.4, RNF-1, LC-1, LC-2, LC-3)* — after T-12
  **Done when:** `code.wrongPasswordConsumesNothing` and `recovery.normalisesAddress` pass, and a test shows the same number of encoder operations for an unknown address, a wrong password and a wrong code.

- [ ] **T-15 · `VerifyRecoveryCodeForReplacementUseCase` — success from a public entry** — in one transaction: lock the identity's code rows, `RecoveryCodeSet.consume` **[001]**, audit `RECOVERY_CODE_CONSUMED`, `RecoveryCaseClosurePort.close(identity, CODE_CONSUMED)`, enqueue `recovery-code-used`, create the `LoginTransaction(RECOVERY)`, `StartEnrollmentUseCase(REPLACEMENT)` **[003]** with `owner = RECOVERY_TRANSACTION` and `startedBy = RECOVERY_CODE`; return the setup material once with the transaction token; no session. *(D2, D3, D4; RF-2.4, RF-2.5, RF-2.8, RF-3.1, RF-5.6, LC-5, LC-9)* — after T-14, T-13
  **Done when:** `code.consumedAtVerificationNeverAcceptedAgain`, `code.publicEntryOpensRecoveryTransactionNoSession`, `notifications.codeUsedEnqueuedToVerifiedAddress` and `replacement.entersPlan003StepsWithRecoveryOwner` pass.

- [ ] **T-16 · `VerifyRecoveryCodeForReplacementUseCase` — session entry** — with a session, the identity comes from `CurrentSessionPort` **[003]**, a submitted address is refused with a field problem, the password is still required, and success starts `StartEnrollmentUseCase(REPLACEMENT)` with `owner = SESSION` and no `StepUpUseCase` **[003]**. *(D4; RF-1.4, RF-2.6, RF-9.5, LC-8)* — after T-15
  **Done when:** `code.sessionEntryStartsReplacementWithoutStepUp`, `recovery.inSessionUsesSessionIdentityAndRefusesAddress` and `recovery.inSessionStillRequiresPassword` pass.

- [ ] **T-17 · `RequestRecoveryCaseUseCase`** — same normalisation and throttles; acknowledgement must be `true`; one password verification in every branch; eligible with an open case → re-send `recovery-case-opened` with the same reference within the re-send cap; eligible without one → create the `RecoveryCase` in `RECEIVED` with a 256-bit `TokenGenerator` **[002]** reference stored as its `TokenHasher` **[002]** hash, record the request locale, enqueue the message; otherwise throttle counters and audit only; empty response in every branch; touches no other row. *(plan §2.1 `RequestRecoveryCaseUseCase`, D1, D5, D9, D11; RF-4.2, RF-4.3, RF-4.4, RF-4.5, RF-4.6, RF-4.7, RF-4.8, RF-5.1, RF-5.10, RF-8.6, LC-10, LC-11, LC-12, LC-13, LC-14, LC-15, LC-33)* — after T-12
  **Done when:** `request.eligibleCreatesCaseHashedReferenceAndMessage`, `request.identityWithoutFactorStillOpensCase`, `request.openCaseResendsSameReference` and `request.changesNothingElse` pass.

- [ ] **T-18 · `ContinueRecoveryCaseUseCase`** — evaluate `RECOVERY_REFERENCE` and `RECOVERY_NETWORK`; hash the reference; load an open case only; one password verification (real or dummy); match → `ContinuationChallenge` bound to the case with a one-time code of the configured length from `TokenGenerator`, stored as a maintained-encoder hash, and `recovery-one-time-code` enqueued; no match → a decoy challenge with no case and nothing sent; always the opaque continuation token; wrong password → sign-in keys too; audit `RECOVERY_CONTINUATION_STARTED{matched}`. *(plan §2.1 `ContinueRecoveryCaseUseCase`, D1, D6; RF-6.2, RF-6.3, RF-6.10, RF-7.2, RF-8.4, LC-22, LC-23)* — after T-12
  **Done when:** `continue.decoyRowForNonMatch` and `continue.closedExpiredOrCompletedReferenceIsDecoy` pass.

- [ ] **T-19 · `VerifyContinuationCodeUseCase`** — lock the challenge; check state, expiry and attempts; recorded outcome for a repeated key; verify the code with the maintained encoder (dummy for a decoy); wrong → `recordFailedAttempt`, `auth.recovery.rejected`, recovery keys; zero or expiry → `void`, `auth.recovery.continuation.void`; success → `VERIFIED` with `verifiedUntil`, audit, and the `CaseView`. *(plan §2.1 `VerifyContinuationCodeUseCase`, D6, D12; RF-6.4, RF-6.5, RF-6.6, RF-6.7, RF-6.9, RF-7.1, LC-24, LC-25, LC-27)* — after T-18, T-11
  **Done when:** `continuation.wrongOrExpiredCodeGenericThenVoid` and `continuation.verifiedWindowExpires` pass, and a re-submission with the same key returns the recorded outcome without counting an attempt.

- [ ] **T-20 · `StartReplacementFromCaseUseCase`** — lock the challenge; require `VERIFIED` within `verifiedUntil` and the case `APPROVED` with a valid approval; recorded outcome for a repeated key; public entry → `LoginTransaction(RECOVERY)` + `StartEnrollmentUseCase(REPLACEMENT)` with `owner = RECOVERY_TRANSACTION`, `startedBy = RECOVERY_CASE(caseId)`; session entry → `owner = SESSION`; the approval is not consumed; audit `RECOVERY_REPLACEMENT_STARTED{source = CASE}`. *(plan §2.1 `StartReplacementFromCaseUseCase`, D3, D4, D7, D12; RF-3.1, RF-5.8, RF-6.7, RF-6.8, LC-26, LC-28, LC-29)* — after T-19
  **Done when:** `approval.grantsReplacementStartOnly` and `approval.survivesAbandonedReplacementUntilValidity` pass.

- [ ] **T-21 · `RecoveryReplacementCompletion` hook in `AcknowledgeRevealUseCase(REPLACEMENT)` [003]** — after plan 003's atomic swap, when `startedBy` is `RECOVERY_CODE` or `RECOVERY_CASE`: `bumpSecurityVersion` **[004]**, `RevokeSessionsPort.revokeAll(identity, except = none | owningSessionId)` **[004]**, `RecoveryCaseClosurePort.complete(identity)`, enqueue the recovery variant of `authenticator-replaced`, audit `RECOVERY_REPLACEMENT_COMPLETED{source}` and `SESSIONS_REVOKED{cause = RECOVERY}`; for a recovery-transaction owner then `CompleteSignInUseCase` **[001]** on that transaction. *(plan §2.1 `RecoveryReplacementCompletion`, D4, D10, D11; RF-3.3, RF-3.4, RF-3.5, RF-3.6, RF-3.8, RF-5.6, RNF-10, RNF-11, LC-5, LC-6, LC-7, LC-8, LC-28)* — after T-15, T-20, T-13
  **Done when:** `completion.swapsRevokesClosesCaseNotifies` (code source and case source), `acknowledge.recoveryOwnedCompletesSignInAndLands` and `acknowledge.sessionOwnedKeepsCurrentRevokesOthers` pass.

- [ ] **T-22 · Closure callers in other features** — `VerifyRecoveryCodeUseCase` **[001]** calls `RecoveryCaseClosurePort.close(CODE_CONSUMED)` when a code completes the challenge; `AcknowledgeRevealUseCase(REPLACEMENT)` **[003]** started from R41 calls `complete`; the identity transitions to `RESTRICTED` / `ERASING` (A-27) call `close(IDENTITY_RESTRICTED | IDENTITY_ERASING)`; plan 004's `ChangeCredential` calls nothing. *(D5; RF-5.6, RF-5.7, LC-18, LC-19, LC-20, LC-21)* — after T-13
  **Done when:** `closure.onChallengeCodeUse`, `closure.onReplacementFromR41`, `closure.onRestrictedOrErasing` and `passwordReset.leavesCaseUntouched` pass.

- [ ] **T-23 · `ReviewRecoveryCaseCommand`** — internal command with no HTTP surface: `moveToReview(caseId, actor)`, `approve(caseId, actor)` with validity from `RecoveryPolicy`, `reject(caseId, actor)`; each locks the case, applies the transition, audits `RECOVERY_CASE_TRANSITION{state, actor}`, enqueues `recovery-case-approved` or `recovery-case-closed{REJECTED}` in the stored locale; no automatic approval exists anywhere. *(plan §2.1 `ReviewRecoveryCaseCommand`, D8, D11; RF-5.2, RF-5.3, RF-5.4, RF-5.5, RF-9.2, RNF-9)* — after T-12
  **Done when:** `review.approveRecordsActorValidityAndNotifies`, `review.rejectNotifiesClosed` and `review.noAutomaticApprovalExists` pass.

- [ ] **T-24 · `ExpireRecoveryCasesJob`** — durable job (`sharedsupport/jobs` **[002]**): cases past `expiresAt` → `EXPIRED{CASE_EXPIRED}`, approvals past `approvalExpiresAt` → `EXPIRED{APPROVAL_EXPIRED}`, each with `recovery-case-closed` and audit; continuation challenges and recovery transactions past expiry voided, the pending enrollment of a voided transaction discarded through `DiscardPendingEnrollmentUseCase` **[003]`; bounded batches. *(plan §2.1 `ExpireRecoveryCasesJob`, D8; RF-3.7, RF-5.1, RF-5.4, LC-17, LC-27)* — after T-12
  **Done when:** `expiryJob.expiresCasesAndApprovalsWithMessage` passes with a controllable clock, and a voided transaction's pending enrollment is discarded in the same run.

- [ ] **T-25 · `AbandonChallengeUseCase` for recovery transactions [001]** — voids the transaction (`ABANDONED`) and discards its pending enrollment; idempotent; the consumed code stays consumed and an approved case stays approved. *(D3; RF-3.7, LC-4, LC-26)* — after T-15
  **Done when:** `replacement.abandonedLeavesOldFactorCodeConsumedCaseApproved` passes.

## D · Infrastructure adapters

- [ ] **T-26 · Persistence adapters** — implementations (persistence access API per A-35) of `RecoveryCaseRepository`, `ContinuationChallengeRepository`; the recovery purpose, state and pending id on the transaction mapping **[001]**; the owner columns on the pending-enrollment mapping **[003]**; the recovery dimensions on `ThrottleStore` **[001]**; code consumption under `SELECT … FOR UPDATE` on the identity's `identity_recovery_code` **[001]** rows. *(plan §2.1 Persistence, D2, D5, D6; RF-2.4, RF-4.4, RF-4.8, RF-5.1, RF-6.2, RNF-3, LC-9, LC-14, LC-29)* — after T-02…T-04, T-12
  **Done when:** repository tests against real PostgreSQL pass for every method, `code.twoTabsOneSuccess` and `request.twoRequestsOneCase` pass, and the partial unique index rejects a second open case through the adapter.

- [ ] **T-27 · Session adapters and revocation scope** — `RevokeSessionsPort` **[004]** and `CurrentSessionPort` **[003]** against the existing session service; `except = none` for a recovery-transaction owner, the acting session for a session owner; run-scoped authority untouched (A-39). *(plan §2.1 Session adapters, D10; RF-1.4, RF-3.4, RF-3.6, RNF-11, LC-7)* — after T-21
  **Done when:** `completion.revokesOtherDeviceSessionLeavesRunAuthority` passes against the session service's test double, and an accepted Run stub is not cancelled.

- [ ] **T-28 · Email adapter and outbox wiring** — the six templates behind the existing outbox relay **[002]**; language from the request header for code-used, opened and one-time-code, from the case's stored locale for approved and closed, from the acknowledging request for the replacement variant; links on the public origin only; relay failures retried, never surfaced. *(plan §2.1 Outbox relay and email adapter, D11; RF-11.5, RNF-12, RNF-14, LC-33)* — after T-05, T-17
  **Done when:** `request.outboxFailureNotVisibleInResponse` and `notification.failureNeverRevertsEffect` pass, and a test shows each message rendered in the language its rule prescribes.

- [ ] **T-29 · Audit adapter** — map the events of plan §2.1 (`RECOVERY_CODE_REJECTED`, `RECOVERY_CODE_CONSUMED`, `RECOVERY_TRANSACTION{…}`, `RECOVERY_REPLACEMENT_STARTED`, `RECOVERY_REPLACEMENT_COMPLETED`, `RECOVERY_REQUEST{matched, action}`, `RECOVERY_CASE_TRANSITION{state, cause, actor?}`, `RECOVERY_CONTINUATION_STARTED | VERIFIED | FAILED | VOIDED`, `ONE_TIME_CODE_SENT`, `SESSIONS_REVOKED{RECOVERY}`, `NOTIFICATION_ENQUEUED`, `NOTIFICATION_SUPPRESSED`, `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]**) to `AuditPort` with identity id when matched, correlation id, network key, timestamp; a serialisation guard rejects any payload holding a password, code, one-time code, reference, unmatched address or message body; entries written in the effect's transaction. *(plan §2.1 Audit adapter, D17; RF-11.1, RF-11.2)* — after T-12
  **Done when:** `audit.entryPerEvent` and `audit.containsNoSecretOrUnmatchedAddress` pass.

- [ ] **T-30 · Logging and problem redaction** — request-body logging excludes `password`, `code`, `reference`; problem details never echo them; traces and analytics carry only the correlation id. *(D14; RF-11.3, RNF-4)* — after T-29
  **Done when:** `logs.containNoPasswordCodeOrReference` passes across the five endpoints with secrets present in every request.

- [ ] **T-31 · Review entry** — an administrative command runnable only from the deployment environment with an actor identifier and a case id that invokes `ReviewRecoveryCaseCommand`; no HTTP endpoint, no tenant screen; placeholder until A-47 decides the operator path. *(plan §2.1 Review entry, D8, §6 A-47; RF-5.2, RF-9.2)* — after T-23
  **Done when:** an integration test runs the three transitions through the command with an actor id and observes the audit entries carrying it, and the API-surface enumeration shows no route maps to it.

## E · API

- [ ] **T-32 · `POST /api/auth/recovery/code`** — `{email?, password, code}` with the UI locale in a request header → 200 `{state: REPLACEMENT_STARTED, transaction?, pendingId, purpose, setup, expiresAt}` or 400 `auth.recovery.rejected` or 400 field-path problems; no login transaction accepted; with a session the address is refused. *(plan §2.1 `RecoveryController`, D14; RF-1.3, RF-1.6, RF-1.7, RF-2.2, RF-2.3, RF-2.5, RF-7.1)* — after T-16
  **Done when:** `recoveryApi.fieldPathProblems` (code part), `recovery.acceptsNoLoginTransaction` and the contract half of `code.failureIsGenericAcrossClasses` pass (identical status, body and headers for the seven classes).

- [ ] **T-33 · `POST /api/auth/recovery/request`** — `{email?, password, acknowledged}` → 202 `{}` in every branch or 400 field-path problems; the acknowledgement refused when not `true`. *(D1, D14; RF-4.2, RF-4.3, RNF-1)* — after T-17
  **Done when:** `request.refusesUntickedAcknowledgement` and `request.responseIdenticalAcrossClasses` pass (identical status, body and headers for the seven classes).

- [ ] **T-34 · `POST /api/auth/recovery/case/continue`, `/case/verify`, `/case/replacement`** — continue `{reference, password}` → 202 `{continuation}` always; verify `{continuation, idempotencyKey, code}` → 200 `{case, canContinue}` or 400 `auth.recovery.rejected` or 410 `auth.recovery.continuation.void`; replacement `{continuation, idempotencyKey}` → 200 as `code` or 410; the case view carries exactly the five fields. *(D6, D12, D14; RF-6.2, RF-6.3, RF-6.4, RF-6.5, RF-6.8, RF-7.1)* — after T-20
  **Done when:** `continue.responseIdenticalAcrossClasses`, `caseView.fieldsAndNothingMore` and `rejected.oneCodeForEveryCause` pass.

- [ ] **T-35 · Enrollment endpoints under a recovery transaction [003] and `DELETE challenge` [001]** — `POST /api/account/mfa/enrollment/{id}/verify | reveal | acknowledge` and `DELETE …/{id}` accept a session **or** the recovery transaction that owns `{id}`; `acknowledge` on a recovery-transaction-owned enrollment answers 200 `{state: RECOVERED, landing}`; `DELETE /api/auth/challenge` voids a recovery transaction; the transaction token authorises nothing else. *(plan §2.1 `RecoveryController`, D3, D4; RF-2.5, RF-3.5, RF-3.7, RNF-8)* — after T-21, T-25
  **Done when:** `recoveryTransaction.authorisesOnlyItsEnrollmentAndAbandon` passes, and an acknowledge under a recovery transaction returns a landing while one under a session returns plan 003's `REPLACED` shape.

- [ ] **T-36 · Exposure and absence checks** — a test that enumerates the API surface and asserts no response carries identity, factor, count or case data before a verified continuation, no endpoint or tenant use case opens, moves, approves or closes a case, consumes a code, or touches another identity's factor, and that `recovery_case` is read by no tenant or `access` path. *(RF-1.6, RF-2.7, RF-5.2, RF-5.9, RF-7.3, RF-9.2, RNF-8, LC-34)* — after T-35, T-31
  **Done when:** `recoveryResponses.exposeNoIdentityFactorCountOrCase`, `noTenantSurfaceOrEndpointTouchesACase`, `noAdminFactorPathExists`, `case.invisibleToTenantSurfaces` and `tenantAdmin.hasNoPathToMemberMfa` pass.

## F · Front end

- [ ] **T-37 · `contracts/recovery` types and guards** — request / response / problem types with runtime guards for the five endpoints, the `RECOVERED` acknowledge response and the recovery-owned enrollment start; pure TypeScript; a guard rejects any response carrying a token, password, code, display name, tenant or count. *(plan §2.2 `contracts/recovery`; RF-1.6, RF-11.3, RNF-8)* — after T-01
  **Done when:** guard unit tests accept every documented response and problem shape and reject a fixture with an extra `code`, `password`, `displayName`, `tenant` or `count` field.

- [ ] **T-38 · i18n bundle `identity-access.{en,es,fr}`: recovery keys** — every heading, tab, label, action, status, validation message and statement of N05 and its D02 states (`423:3542`) (S-38): the received statement, the one-time-code statement, the case states, the generic failure, the in-session variant, and the problem codes of T-32…T-35. *(D16, §6 S-38; RF-1.1, RF-1.10, RF-10.1, RNF-14)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value.

- [ ] **T-39 · Hooks** — `useRecoveryCode`, `useRecoveryRequest`, `useContinueCase`, `useVerifyContinuation`, `useStartReplacementFromCase` (mutations with retry disabled and the UI locale header attached; the last two generate their idempotency key once per mount and reuse it on re-submission); the session presence read from the existing session query **[001]**. *(plan §2.2 hooks and State, D12, D13; RF-2.1, RF-4.1, RF-6.1, RNF-3)* — after T-37
  **Done when:** unit tests show each mutation sends the current locale, has retry disabled, and the same idempotency key is sent on a second call of the same mounted verify or replacement hook.

- [ ] **T-40 · Holders** — the transaction holder **[001]** extended with the purpose `RECOVERY` and the pending enrollment id, cleared on `RECOVERED`, void, abandon and route leave, `DELETE challenge` fired on leave and best-effort on `pagehide`; the continuation holder `{continuation, caseView, verifiedUntil}` cleared by its idle timer and on route leave; nothing in storage. *(plan §2.2 State, D13; RF-2.5, RF-3.7, RF-6.9, RF-11.4, RNF-4, LC-30, LC-32)* — after T-39
  **Done when:** `storage.containsNothingFromFeature` and `continuationHolder.clearsOnIdleAndLeaveNeverStorage` pass (storage spies see no writes), and the transaction holder is empty after route leave.

- [ ] **T-41 · `RecoveryPage` shell** — heading, subtitle, the keyboard-operable tabs "Recovery code" / "Lost all factors", the three card slots, the "No administrator backdoor" notice, "← Back to sign in", "Return to local sign in", Privacy / Terms (S-14), language switch, no "Help" (S-17); the in-session variant with the read-only address; no alternative channel offered. *(plan §2.2 `RecoveryPage`; RF-1.1, RF-1.4, RF-1.6, RF-1.9, RF-9.2, RF-9.4, LC-35)* — after T-38, T-40
  **Done when:** `recoveryPage.rendersTabsCardsNoticeAndRoutes` (desktop, tablet, mobile), `recoveryPage.rendersNoHelp`, `recoveryPage.sessionVariantReadOnlyAddress` and `recoveryPage.offersNoAlternativeChannel` pass.

- [ ] **T-42 · `RecoveryCodeCard`** — address (pre-filled from the voided challenge holder for display only), `PasswordField` with Show, `CodeField` **[001]** with paste and separator stripping, "Verify recovery code", the statement; client validation with associated messages; generic failure keeps the address and clears password and code, offering only try again or return; on `REPLACEMENT_STARTED` stores the transaction and opens the wizard; never a count. *(plan §2.2 `RecoveryCodeCard`; RF-1.3, RF-1.7, RF-2.1, RF-2.3, RF-2.7, RF-7.1)* — after T-41
  **Done when:** `recoveryCodeCard.fieldsPasteAndStatement`, `recoveryPage.prefillsAddressFromVoidedChallengeOnly`, `recoveryPage.blocksEmptyOrMalformedFields` and `genericFailure.tryAgainAndReturnOnly` pass.

- [ ] **T-43 · `RecoveryRequestCard`** — statements, "What remains protected", address, `PasswordField`, the never-pre-ticked acknowledgement `Checkbox`, "Start verified recovery request", the reference statement; received state: the generic statement above the same card, focus on it, address kept, password cleared, action available; no time promise. *(plan §2.2 `RecoveryRequestCard`; RF-4.1, RF-4.2, RF-4.3, RF-5.5)* — after T-41
  **Done when:** `requestCard.rendersStatementsCheckboxAndReceivedState` passes and the checkbox renders unticked on every mount.

- [ ] **T-44 · `ContinueCaseCard` and `CaseDetails`** — reference field, `PasswordField`, "Continue case verification", statement; the one-time-code step with its statement and code field; generic failure; void → back to the card; `CaseDetails` with state as text, masked address, request time, outcome; received / under review: statement and "Return to sign in" only; approved: "Continue to replace your authenticator"; cleared when the idle window ends; no time promise. *(plan §2.2 `ContinueCaseCard`, `CaseDetails`; RF-5.5, RF-6.1, RF-6.3, RF-6.4, RF-6.5, RF-6.6, RF-6.7, RF-6.9, LC-25)* — after T-41
  **Done when:** `continueCard.fieldsAndStatement`, `continueCard.voidReturnsToCard` and `caseDetails.underReviewOffersReturnOnly` pass.

- [ ] **T-45 · Wizard reuse and routing handoffs** — `EnrollmentWizard` **[003]** opened with a recovery owner at `/recover/replace` (public) or `/account/security/enroll` (session) showing the replacement heading and statement; "Leave setup" → abandon; `RECOVERED` → `LandingRouter` **[001]**; session completion → `/account/security`; `auth.recovery.continuation.void` → the continuation card; R02 **[001]**, R05 **[001]** (after `DELETE challenge`), R41 **[003]** and the R38 `RecoveryCodesCard` **[003]** routes → `/recover`; N20 carries no route. *(plan §2.2 Routing, D13; RF-1.2, RF-1.5, RF-3.1, RF-3.5, RF-3.6, RF-3.7, RF-6.8)* — after T-42, T-43, T-44
  **Done when:** `wizard.leaveDuringRecoveryAbandonsTransaction`, `entries.signInChallengeReplaceDialogAndSettingsRouteToRecover` and `tenantSuspended.hasNoRecoveryRoute` pass, and a route test shows `RECOVERED` handed to `LandingRouter`.

- [ ] **T-46 · `SecurityStatements` extension [001]** — add every statement of N05 and its D02 states (`423:3542`) as shared components used by the page and its cards at every width. *(plan §2.2 `SecurityStatements`, D16; RF-1.1, RF-2.1, RF-4.1, RF-6.1, RF-10.1, RNF-15)* — after T-45
  **Done when:** a test renders the page and each card state at desktop and mobile widths and finds identical statement text in both.

- [ ] **T-47 · Reflow and touch targets** — Tailwind breakpoints for the page and all its states; `shared/ui` field, password field, code field, checkbox, tabs and button at 44 px minimum on touch widths; visible labels and focus; breakpoint layout replaceable (S-03). *(D16, §6 S-03; RF-10.1, RF-10.2, RF-10.3, RNF-13, RNF-15)* — after T-46
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow on every state and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-48 · Timing-comparability harness** — N samples per input class for code verification (unknown address, wrong password, consumed code, no factor, restricted, erasing, throttled), request (eligible, wrong password, unverified, restricted, erasing, unknown, throttled) and continuation (open case, closed case, unknown reference, wrong password, throttled); distributions overlap within the configured tolerance. *(D1, D6; RF-2.2, RF-4.3, RF-6.3, RF-7.2, RNF-1)* — after T-32, T-33, T-34
  **Done when:** `code.timingComparable`, `request.timingComparable` and `continue.timingComparable` pass on the CI runner for three consecutive runs, and `code.failureIsGenericAcrossClasses` passes in full.

- [ ] **T-49 · Concurrency suite** — two continuations of one approved case both starting a replacement; an acknowledge racing a sign-in challenge of the same identity; two requests for one address; the same code in two tabs. *(D2, D5, D7; RF-5.8, RNF-3, LC-9, LC-14, LC-29)* — after T-26, T-21
  **Done when:** `approvedCase.twoReplacementsOneCompletes` passes ten times in a row with real PostgreSQL alongside `code.twoTabsOneSuccess` and `request.twoRequestsOneCase`, and the racing challenge is voided by the version bump.

- [ ] **T-50 · Throttle suite** — independence from sign-in and reset except the wrong-password rule, wrong-password counting on the sign-in keys, other failures on the recovery keys only, window lift with a controllable clock, restriction answering identically, per-address caps. *(D9; RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-8.5, RF-8.6, RNF-2, RNF-5, LC-3, LC-11, LC-15, LC-16, LC-23)* — after T-14, T-17, T-18, T-26
  **Done when:** `throttle.recoverySignInAndResetIndependentExceptPasswordRule`, `throttle.wrongPasswordCountsOnSignInKeys`, `throttle.wrongCodeOrReferenceCountsOnRecoveryKeysOnly`, `throttle.recoveryWindowLiftsAndSignInNeverRestricted`, `recovery.restrictedIssuesNothingSameAnswer` and `caps.resendsCodesAndNotificationsBoundedPerAddress` pass.

- [ ] **T-51 · Startup suite** — one case per property of T-01 plus the floor, origin and language cases. *(D15; RF-8.7, RNF-6)* — after T-01
  **Done when:** `recoveryPolicy.startupFailsOnAbsentOrUnboundedValue` passes, each case failing the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-52 · Fail-closed and scope suite** — unknown, consumed or malformed code, unknown, closed, expired or decoy reference, missing, expired or voided one-time code, expired approval, voided transaction, absent configuration all refuse the effect; both flows touch only identity tables and the outbox; the old factor stays required while a recovery replacement is pending; a recovered Owner of a suspended tenant lands on the chooser with nothing lifted; the case record holds no secret. *(D2, D6, D15; RF-3.2, RF-3.8, RF-5.10, RF-9.1, RF-9.3, RF-9.4, RNF-7, RNF-10, LC-6, LC-32)* — after T-21, T-26
  **Done when:** `failClosed.unknownConsumedExpiredDecoyVoidedOrUnconfiguredRefuses`, `recovery.touchesOnlyIdentityTablesAndOutbox`, `signIn.duringPendingRecoveryUsesOldFactor`, `recovery.ownerOfSuspendedTenantLandsOnChooserUnlifted` and `recoveryCase.holdsNoSecretOrTenantContent` pass.

- [ ] **T-53 · FE e2e journeys** — Playwright journeys for US-1 … US-9 in EN, ES and FR at desktop, tablet and mobile widths with a fake email port exposing the reference and the one-time code: code → replacement → landing; code from R41 → replacement in session; request → received → (review stub approves) → continue → one-time code → approved → replacement → landing; continue while under review; closed reference → generic; abandon at each step; language switch on each state; recovered Owner gated at tenant entry. *(RF-3.5, RF-3.9, RF-1.10, RNF-14, LC-5, LC-8, LC-25, LC-28, LC-31; US-1 … US-9)* — after T-47, T-36
  **Done when:** `recoveredOwner.gateDerivesFromNewFactorAtTenantEntry` and `languageSwitch.onRecoveryKeepsEverything` pass with the journeys in the three languages and three widths.

- [ ] **T-54 · Automated accessibility** — axe on the recovery screen and every state (initial, received, one-time-code step, case details per state, generic failure, in-session variant, wizard) at the three widths and three languages. *(RF-10.1, RF-10.2, RNF-13, RNF-14, RNF-15)* — after T-53
  **Done when:** `a11y.recovery.{en,es,fr}.{desktop,tablet,mobile}` reports zero critical or serious violations on every combination.

- [ ] **T-55 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist: tab operation, focus order, field error association, received and one-time-code statements announced without focus theft, generic rejection announced, labels, 44 px targets; versions recorded in the release-support matrix. *(RF-10.2, RNF-13; FR-X-040)* — after T-54
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-56 · Adversarial suite (US-10)** — enumeration across code, request and continuation; a code without the password, a reference without the mailbox, a mailbox without the password, a session without the password; reuse of a consumed code, a used one-time code and a completed approval; attempts to obtain a session, a membership or a lifted suspension from recovery; attempts to disable MFA; search for an administrator or operator surface. *(D1, D2, D6, D7, D8; RF-2.2, RF-5.8, RF-6.2, RF-9.1, RF-9.2, RF-9.3, RF-9.5, RNF-1, RNF-3, RNF-9, RNF-10; US-10)* — after T-36, T-49, T-50
  **Done when:** every attack test fails to achieve its goal and asserts the generic response or the refusal.

## H · Closing

- [ ] **T-57 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-48 … T-56
  **Done when:** the script reports 128 of 128 identifiers (78 RF, 15 RNF, 35 LC) covered by at least one passing test.

- [ ] **T-58 · Register-dependency review** — walk §6 of the plan (A-47, A-48, A-27, A-30, A-39, A-35, S-38, S-14, S-03, S-17): for each, confirm the built assumption matches the current register state or record the deferral. *(spec §7 item 14)* — after T-57
  **Done when:** each of the ten items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-59 · Completion criteria sign-off** — tick spec §7 items 1–14 with the evidence (test run id, timing reports, a11y report, manual checklist, coverage report, register review). — after T-58
  **Done when:** all fourteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-38, T-41, T-46 |
| RF-1.2 | T-45 |
| RF-1.3 | T-32, T-42 |
| RF-1.4 | T-16, T-27, T-41 |
| RF-1.5 | T-45 |
| RF-1.6 | T-32, T-36, T-37, T-41 |
| RF-1.7 | T-32, T-42 |
| RF-1.8 | T-14 |
| RF-1.9 | T-41 |
| RF-1.10 | T-38, T-53 |
| RF-2.1 | T-39, T-42, T-46 |
| RF-2.2 | T-14, T-32, T-48, T-56 |
| RF-2.3 | T-10, T-14, T-32, T-42 |
| RF-2.4 | T-15, T-26 |
| RF-2.5 | T-04, T-06, T-15, T-32, T-35, T-40 |
| RF-2.6 | T-04, T-07, T-16 |
| RF-2.7 | T-14, T-36, T-42 |
| RF-2.8 | T-05, T-15 |
| RF-3.1 | T-04, T-07, T-15, T-20, T-45 |
| RF-3.2 | T-07, T-52 |
| RF-3.3 | T-21 |
| RF-3.4 | T-05, T-21, T-27 |
| RF-3.5 | T-06, T-21, T-35, T-45, T-53 |
| RF-3.6 | T-21, T-27, T-45 |
| RF-3.7 | T-06, T-07, T-24, T-25, T-35, T-40, T-45 |
| RF-3.8 | T-12, T-21, T-52 |
| RF-3.9 | T-53 |
| RF-4.1 | T-39, T-43, T-46 |
| RF-4.2 | T-17, T-33, T-43 |
| RF-4.3 | T-17, T-33, T-43, T-48 |
| RF-4.4 | T-02, T-05, T-08, T-17, T-26 |
| RF-4.5 | T-17 |
| RF-4.6 | T-17 |
| RF-4.7 | T-17 |
| RF-4.8 | T-02, T-17, T-26 |
| RF-5.1 | T-02, T-08, T-17, T-24, T-26 |
| RF-5.2 | T-23, T-31, T-36 |
| RF-5.3 | T-05, T-08, T-23 |
| RF-5.4 | T-05, T-08, T-23, T-24 |
| RF-5.5 | T-05, T-11, T-23, T-43, T-44 |
| RF-5.6 | T-05, T-08, T-13, T-15, T-21, T-22 |
| RF-5.7 | T-13, T-22 |
| RF-5.8 | T-08, T-20, T-49, T-56 |
| RF-5.9 | T-12, T-36 |
| RF-5.10 | T-17, T-52 |
| RF-6.1 | T-39, T-44, T-46 |
| RF-6.2 | T-03, T-05, T-09, T-18, T-26, T-34, T-56 |
| RF-6.3 | T-09, T-18, T-34, T-44, T-48 |
| RF-6.4 | T-03, T-09, T-19, T-34, T-44 |
| RF-6.5 | T-11, T-19, T-34, T-44 |
| RF-6.6 | T-11, T-19, T-44 |
| RF-6.7 | T-11, T-19, T-20, T-44 |
| RF-6.8 | T-20, T-34, T-45 |
| RF-6.9 | T-03, T-09, T-19, T-40, T-44 |
| RF-6.10 | T-08, T-18 |
| RF-7.1 | T-14, T-19, T-32, T-34, T-42 |
| RF-7.2 | T-14, T-18, T-48 |
| RF-7.3 | T-36 |
| RF-8.1 | T-04, T-10, T-50 |
| RF-8.2 | T-10, T-50 |
| RF-8.3 | T-10, T-50 |
| RF-8.4 | T-10, T-14, T-18, T-50 |
| RF-8.5 | T-10, T-50 |
| RF-8.6 | T-01, T-17, T-50 |
| RF-8.7 | T-01, T-51 |
| RF-9.1 | T-12, T-52, T-56 |
| RF-9.2 | T-12, T-23, T-31, T-36, T-41, T-56 |
| RF-9.3 | T-52, T-56 |
| RF-9.4 | T-41, T-52 |
| RF-9.5 | T-16, T-56 |
| RF-10.1 | T-38, T-46, T-47, T-54 |
| RF-10.2 | T-47, T-54, T-55 |
| RF-10.3 | T-47 |
| RF-11.1 | T-13, T-29 |
| RF-11.2 | T-29 |
| RF-11.3 | T-30, T-37 |
| RF-11.4 | T-40 |
| RF-11.5 | T-05, T-28 |
| RNF-1 | T-14, T-33, T-48, T-56 |
| RNF-2 | T-10, T-50 |
| RNF-3 | T-03, T-26, T-39, T-49, T-56 |
| RNF-4 | T-30, T-40 |
| RNF-5 | T-02, T-50 |
| RNF-6 | T-01, T-51 |
| RNF-7 | T-01, T-52 |
| RNF-8 | T-06, T-35, T-36, T-37 |
| RNF-9 | T-23, T-56 |
| RNF-10 | T-21, T-52, T-56 |
| RNF-11 | T-21, T-27 |
| RNF-12 | T-05, T-28 |
| RNF-13 | T-47, T-54, T-55 |
| RNF-14 | T-05, T-28, T-38, T-53, T-54 |
| RNF-15 | T-46, T-47, T-54 |
| LC-1 | T-14 |
| LC-2 | T-14 |
| LC-3 | T-10, T-14, T-50 |
| LC-4 | T-06, T-25 |
| LC-5 | T-15, T-21, T-53 |
| LC-6 | T-21, T-52 |
| LC-7 | T-21, T-27 |
| LC-8 | T-16, T-21, T-53 |
| LC-9 | T-15, T-26, T-49 |
| LC-10 | T-17 |
| LC-11 | T-17, T-50 |
| LC-12 | T-17 |
| LC-13 | T-17 |
| LC-14 | T-02, T-17, T-26, T-49 |
| LC-15 | T-17, T-50 |
| LC-16 | T-10, T-50 |
| LC-17 | T-08, T-24 |
| LC-18 | T-22 |
| LC-19 | T-22 |
| LC-20 | T-22 |
| LC-21 | T-08, T-22 |
| LC-22 | T-09, T-18 |
| LC-23 | T-09, T-18, T-50 |
| LC-24 | T-09, T-19 |
| LC-25 | T-19, T-44, T-53 |
| LC-26 | T-20, T-25 |
| LC-27 | T-08, T-19, T-24 |
| LC-28 | T-20, T-21, T-53 |
| LC-29 | T-06, T-20, T-26, T-49 |
| LC-30 | T-09, T-40 |
| LC-31 | T-53 |
| LC-32 | T-07, T-40, T-52 |
| LC-33 | T-17, T-28 |
| LC-34 | T-36 |
| LC-35 | T-41 |
| US-1 … US-9 | T-53 (journeys) |
| US-10 | T-56 (adversarial suite) |

Self-check: all 78 RF criteria, 15 RNF and 35 limit cases of `specControlledAccountRecovery.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.

