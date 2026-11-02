# Tasks 001 — Login

**Feature** Sign-in (first factor, TOTP challenge and landing) · **Tasks version** 0.1 · **Date** 26 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/001-login/specLogin.md` v0.1 (the WHAT) and `specs/001-login/planLogin.md` v0.1 (the HOW) · **Governed by** `constitution.md` v1.8.

This document is the DO. It turns the plan into small, ordered, checkable tasks. It adds no behaviour: every task implements something the specification states and the plan has designed. Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n, US-n); plan references are its §2 element names, its decisions D1–D16 and its register assumptions (§6). Test names in "Done when" lines are taken verbatim from plan §5.2 so each task can be checked against the test matrix.

**How to read a task**

```
- [ ] T0nn [P] Imperative title (one sitting, 20–30 min)
      Covers: the specification items this task implements or proves · Plan: the element / decision it realises
      After: an explicit dependency, only when it is not simply the previous task
      Done when: one verifiable condition
```

`[P]` marks a task that can run in parallel with its neighbours (no shared file, dependencies met). Tasks are numbered in dependency order; a later task may assume everything above it. Phases 6–8 (front end) depend only on Phase 0 and may run alongside Phases 3–5 (back end).

**Assumption on the starting point.** The workspace holds no code yet. The platform pieces the plan names as existing — Spring Security JWT / refresh / MFA support, the session service (F-IDN-08), the `access`, `audit` and `sharedsupport` contexts — are expected in the product repository; where Login tests need them they are stubbed with fixed answers (plan §5.3). No task adds a dependency outside the fixed stack (P10) except the test-only ones the plan flags: JUnit 5, MockMvc, Testcontainers, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided.

---

## Phase 0 — Foundations (shared by both sides)

- [ ] T001 Create the back-end package skeleton `com.aspectra.identity.{api, application, domain, infrastructure}` with `package-info` files stating the allowed-import direction (api → application → domain; infrastructure → application/domain; no foreign repositories).
      Covers: P10, P11 (enabling; no specification criterion) · Plan: §2.1
      Done when: the module compiles with the four empty packages and a reviewer confirms no import points outward from `domain` or `application`.

- [ ] T002 [P] Create the front-end feature module skeleton `src/features/identity-access` with an `index.ts` public API (routes and hooks placeholders only) and an empty `src/contracts/auth.ts`.
      Covers: P10, P11 (enabling; no specification criterion) · Plan: §2.2, D14
      Done when: `tsc` and the lint rule forbidding sibling-internal imports pass with the empty module wired into `app`.

- [ ] T003 Write `contracts/auth`: request, response and problem types plus runtime guards for the five endpoints (`POST sign-in`, `POST challenge/totp`, `POST challenge/recovery-code`, `DELETE challenge`, `GET landing`), including the `LIMITED` / `COMPLETED` / `VERIFICATION_PENDING` outcome union (`VERIFICATION_PENDING` carries `{pendingRef, email}`; `COMPLETED` carries `landing = {decision, projection}`), the challenge requests with their idempotency key, and the landing decision union. No factor, seed, tenant or personal fields exist in the `LIMITED` shape.
      Covers: RF-1.6, RF-2.5, RF-5.1, RF-5.2, RF-5.5, RF-6.6, RF-6.11, RF-7.4, RF-8.1 · Plan: §2.2 `contracts/auth`, D12
      After: T002
      Done when: Vitest guard tests accept one fixture per response and problem shape and reject a `LIMITED` fixture carrying any extra field.

- [ ] T004 [P] Fix the problem codes (`auth.sign-in.failed`, `auth.verification-pending`, `auth.challenge.rejected`, `auth.challenge.void`) and write the EN bundle `identity-access.en.json` with one key for every statement and message of RF-1.2, RF-1.3, RF-1.4, RF-2.5, RF-6.1 and RF-6.6, the field-validation messages of RF-1.6 and the labels of every control on R02 / M02 / R05.
      Covers: RF-1.2, RF-1.3, RF-1.4, RF-2.5, RF-6.1, RF-6.6, RNF-6 · Plan: D12, D16
      Done when: the generic failure text of RF-2.5 is present verbatim, and a key-catalogue test lists every key referenced by this file as the single source for the ES / FR bundles of T058.

---

## Phase 1 — Back-end domain (pure, unit-tested with JUnit 5)

- [ ] T005 Implement `LoginPolicy`: an immutable value object holding every configured number (throttle thresholds and delay steps per dimension, restriction windows, transaction expiry, attempt limit, authenticator time tolerance). The constructor rejects any absent, non-positive or above-ceiling value.
      Covers: RF-4.8, RF-5.3, RNF-4 · Plan: §2.1 `LoginPolicy`, D10
      Done when: a unit test constructs a valid policy and one test per field proves that omitting or unbounding it throws.

- [ ] T006 [P] Implement `GlobalIdentity`, `IdentityStatus` (working values `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING`, pending A-27: `ACTIVE` permits sign-in, `UNVERIFIED` takes the verification branch, the others the generic branch), the identity's `securityVersion`, and the email normalisation function (trim, case-fold).
      Covers: RF-2.6, RF-2.7, LC-2, LC-19 · Plan: §2.1 `GlobalIdentity`, register A-27
      Done when: `email.normalisation` passes (mixed case and surrounding whitespace map to one value) and a table test shows the branch for each status: sign-in for `ACTIVE`, verification for `UNVERIFIED`, generic for the rest.

- [ ] T007 [P] Implement `Credential` exposing only `matches(rawPassword, verifier)` and declare the `PasswordVerifier` port. No accessor returns the stored hash; `toString` is redacted.
      Covers: RF-2.1, RF-10.3 · Plan: §2.1 `Credential`
      Done when: a unit test with a fake verifier proves `matches` delegates once, and a reflection test finds no public method returning the hash.

- [ ] T008 Implement the `LoginTransaction` state machine (`FIRST_FACTOR_PASSED → COMPLETED | VOIDED`) with identity id, purpose `SIGN_IN`, `expiresAt`, `attemptsRemaining`, `voidReason` (`ATTEMPTS_EXHAUSTED`, `EXPIRED`, `ABANDONED`, `CREDENTIAL_CHANGED`, `FACTOR_CHANGED`, `SUPERSEDED`), the identity's `securityVersion` captured at creation, the recorded outcome per submission idempotency key, and the methods `recordFailedAttempt`, `complete`, `void`, `recordedOutcome(key)`, each taking the clock as an argument.
      Covers: RF-5.3, RF-5.4, RF-7.1, RF-7.2, RF-7.3, LC-4, LC-5, LC-6, LC-13, LC-14, LC-24 · Plan: §2.1 `LoginTransaction`, D1
      After: T005
      Done when: unit tests prove: success on the last attempt completes (LC-5); failure on the last attempt voids with `ATTEMPTS_EXHAUSTED` (LC-6); any call after expiry voids with `EXPIRED` and never completes (LC-4); every method on a `COMPLETED` or `VOIDED` transaction is refused; a submission replayed with the same key returns the recorded outcome without a state change.

- [ ] T009 [P] Implement `AbuseThrottlePolicy` as pure functions over `LoginPolicy`: `delayFor(failures)`, `restrictionFor(failures, now)`, `isRestricted(window, now)`, for the two dimensions `ACCOUNT` and `NETWORK`.
      Covers: RF-4.1, RF-4.2, RF-4.4, RNF-2 · Plan: §2.1 `AbuseThrottlePolicy`, D3
      After: T005
      Done when: `throttlePolicy.progressiveDelaySteps` passes, `isRestricted` is false once the window has elapsed for every input, and a property test over long failure sequences never yields an unbounded window.

- [ ] T010 [P] Implement `TotpFactor` (encrypted seed reference, `replayWatermark` = last accepted time-step; never exposes the seed), declare the `TotpVerifier` port, and add the code-separator stripping function (spaces, hyphens, dots removed before verification).
      Covers: RF-6.2, RF-6.5, RF-6.11, LC-3, LC-23 · Plan: §2.1 `TotpFactor`, D4, D6
      Done when: `totpVerify.stripsSeparators` passes; `totpVerify.rejectsReplayAtOrBelowWatermark` passes at unit level with a fixed clock; serialising a `TotpFactor` yields no seed material.

- [ ] T011 [P] Implement `RecoveryCodeSet` with hashed single-use codes and `consume(rawCode, verifier)` returning the consumed code or nothing.
      Covers: RF-6.7, RF-6.8, LC-7, LC-8 · Plan: §2.1 `RecoveryCodeSet`, D5
      Done when: unit tests prove a code consumes exactly once, a second use of the same code returns nothing, and consuming the last code leaves a set of size zero.

- [ ] T012 [P] Implement `LandingRule`: memberships → `CREATE_WORKSPACE | ENTER_TENANT(id) | TENANT_CHOOSER`.
      Covers: RF-8.2, RF-8.3, RF-8.4, LC-15, LC-16, LC-17 · Plan: §2.1 `LandingRule`, D7
      Done when: `landingRule.{none,oneActive,oneSuspended,oneRevoked,oneDeleting,many}` pass (a single revoked membership → `TENANT_CHOOSER`, LC-15).

---

## Phase 2 — Back-end application (use cases over ports, unit-tested with in-memory doubles)

- [ ] T013 Declare the ports `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `Clock`, `SeedCipher`, `IssueSessionPort`, `MembershipProjectionPort`, `AuditPort`, `SecurityNotificationPort` (declared here; its first caller is plan 002), `PendingReferenceStore` (owned by plan 002; used by the verification-pending branch), the eight audit event kinds (`SIGN_IN_SUCCEEDED`, `SIGN_IN_FAILED`, `CHALLENGE_PASSED`, `CHALLENGE_FAILED`, `CHALLENGE_VOIDED`, `RECOVERY_CODE_USED`, `RESTRICTION_APPLIED{restrictedUntil}`, `RESTRICTION_LIFTED`) and one in-memory test double per port.
      Covers: RF-10.1 (event catalogue) · Plan: §2.1 ports, register A-30
      After: T012
      Done when: the application package compiles against the ports only and the doubles are used by at least one test in T014.

- [ ] T014 Implement `SignInUseCase` happy paths: normalise email, load identity, verify credential; no factor enrolled → delegate to `CompleteSignInUseCase` (stubbed until T018); factor enrolled → create `LoginTransaction` from `LoginPolicy` and answer `LIMITED` with an opaque token.
      Covers: RF-2.1, RF-5.1, RF-5.3, RF-5.5, LC-18 · Plan: §2.1 `SignInUseCase`, D1
      Done when: unit versions of `signIn.enrolledFactorYieldsLimitedState` and `signIn.noFactorCompletes` pass with the doubles.

- [ ] T015 Add the constant-cost generic failure to `SignInUseCase`: an unknown email, and a `RESTRICTED` or `ERASING` identity, run the verifier against the fixed dummy hash (`PasswordVerifier.verifyAgainstDummy()`); a wrong password and those statuses reach the same single failure branch; no identity is created and no password stored for an unknown email (throttle and audit writes still happen); the failure result carries no class information.
      Covers: RF-2.2, RF-2.3, RF-2.4, RF-2.7, LC-2, RNF-1 · Plan: D2
      Done when: a unit test shows the verifier invoked exactly once for each of unknown / wrong / restricted, the three results are `equals()`, and the identity repository double records no identity and no credential write for the unknown case.

- [ ] T016 Add the unverified-email branch to `SignInUseCase`: correct password on an `UNVERIFIED` identity (verified against its real credential) → `VERIFICATION_PENDING` carrying `{pendingRef, email}` issued through `PendingReferenceStore`, with no transaction and no session; wrong password and unverified → generic failure; the verification check runs before the factor check and after the generic branch.
      Covers: RF-3.1, RF-3.2, RF-3.3, LC-1 · Plan: D9, register A-34
      Done when: unit tests prove the three orderings: unverified + factor enrolled yields `VERIFICATION_PENDING` with a reference and no transaction (LC-1); unverified + wrong password yields the generic result of T015 and no reference.

- [ ] T017 Add throttle evaluation to `SignInUseCase`: both dimensions read from `ThrottleStore` after the dummy-cost step; `delayFor` applied before answering; an active restriction forces the generic branch even with a correct password but never short-circuits the verification (full path); failures upsert both counters; a newly applied restriction emits `RESTRICTION_APPLIED{restrictedUntil}`, and the first attempt observed after a window has elapsed emits `RESTRICTION_LIFTED`.
      Covers: RF-4.1, RF-4.2, RF-4.3, RF-4.4, RF-4.6, LC-11, LC-12 · Plan: D3
      Done when: unit tests prove a correct password under an account restriction returns the generic result (LC-11), a network restriction makes a different account fail (LC-12), and the delay double is called with the configured step for the failure count.

- [ ] T018 Implement `CompleteSignInUseCase`: mark the transaction `COMPLETED` (or create none for the no-factor path), call `IssueSessionPort.issue(identityId)`, append `SIGN_IN_SUCCEEDED`; call nothing else.
      Covers: RF-8.1, RF-8.6 · Plan: §2.1 `CompleteSignInUseCase`, D8
      Done when: a unit test asserts exactly one `issue` call, one audit append, and that no other port double was touched.

- [ ] T019 Implement `VerifyTotpUseCase`: lock the transaction row (port method); if the submission's idempotency key already has a recorded outcome, return it without re-evaluation; check state, expiry, attempts and the identity's current `securityVersion` (mismatch → void with `CREDENTIAL_CHANGED` or `FACTOR_CHANGED`); strip separators; verify within the tolerance and above the watermark; advance the watermark and complete on success; on failure `recordFailedAttempt`, void at zero with both throttle counters updated; record the outcome under the key; append the audit event.
      Covers: RF-4.5, RF-6.2, RF-6.4, RF-6.5, RF-6.6, RF-7.1, RF-7.2, RF-7.5, LC-3, LC-4, LC-5, LC-6, LC-13, LC-14, LC-24 · Plan: §2.1 `VerifyTotpUseCase`, D1, D4
      Done when: unit tests with doubles cover valid, replayed (LC-3), wrong, valid-on-last-attempt (LC-5), wrong-on-last-attempt (LC-6), expired (LC-4, watermark unchanged and throttle counters incremented on void), version mismatch (LC-13, LC-14) and a repeated key returning the recorded outcome (LC-24).

- [ ] T020 Implement `VerifyRecoveryCodeUseCase` with the same transaction discipline: consume one unused code under the lock; unknown or consumed code follows the failure path of T019; success appends `RECOVERY_CODE_USED` and completes.
      Covers: RF-6.7, RF-6.8, LC-7, LC-8 · Plan: §2.1 `VerifyRecoveryCodeUseCase`, D5
      Done when: unit tests prove consume-once, consumed-treated-as-wrong with an attempt decremented (LC-7), and last-code-completes (LC-8).

- [ ] T021 [P] Implement `AbandonChallengeUseCase`: void with `ABANDONED`, idempotent on an already voided or completed transaction, appends `CHALLENGE_VOIDED` once.
      Covers: RF-5.4, RF-6.9, LC-22 · Plan: §2.1 `AbandonChallengeUseCase`, D13
      Done when: a unit test calls abandon twice and sees one state change, one audit entry and the same result both times.

- [ ] T022 [P] Implement `LandingQuery`: read the membership projection through `MembershipProjectionPort` as an identity-scoped read for the signed-in identity (A-36), apply `LandingRule`, return `{decision, projection}`.
      Covers: RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-8.5, RF-8.7, LC-15 · Plan: §2.1 `LandingQuery`, D7
      Done when: a unit test shows the port called with the signed-in identity only and the decision matching `LandingRule` for a projection that changed after the first factor.

- [ ] T023 [P] Add `IdentityRepository.currentSecurityVersion(identityId)` and wire the version check of T019 into `VerifyRecoveryCodeUseCase` as well; a password reset or factor change bumps the identity's version (their own features), which voids every open transaction on its next use — no cross-feature call into Login's table.
      Covers: RF-7.3, LC-13, LC-14 · Plan: D1
      Done when: a unit test bumps the identity's version after two transactions were opened and both verification use cases then refuse them with `CREDENTIAL_CHANGED` / `FACTOR_CHANGED`.

---

## Phase 3 — Back-end infrastructure (PostgreSQL through the persistence access API and migration tool of A-35, placeholders until decided; integration tests on real PostgreSQL through Testcontainers, test-only)

- [ ] T024 Write the schema migration for the six platform tables `identity` (including `security_version`), `identity_credential`, `identity_totp_factor` (ciphertext column + key reference), `identity_recovery_code`, `login_transaction` (including `security_version` and a recorded-outcome child table keyed by idempotency key), `signin_throttle` (primary key `(dimension, subject_key)`), with a unique index on the normalised email and no `tenant_id` column anywhere. Plan 002 extends these same tables; it never re-creates them.
      Covers: RF-2.6, RF-5.3, RF-6.11, RNF-7 · Plan: §2.1 persistence, P12, D1, D6
      Done when: the migration applies on a fresh Testcontainers PostgreSQL and a schema test asserts the unique index, the composite key and the absence of `tenant_id` in all six tables.

- [ ] T025 Implement `LoginPolicyProperties` (`@ConfigurationProperties` + `@Validated`) binding every number into `LoginPolicy` with constraints present, positive, bounded above by a configured ceiling, and no default anywhere.
      Covers: RF-4.8, RF-5.3, RNF-4, RNF-8 · Plan: §2.1 `LoginPolicyProperties`, D10
      Done when: a Spring Boot test starts with a complete property set and fails context refresh when `transaction expiry` is removed (full matrix in T052).

- [ ] T026 [P] Implement the persistence adapters for `IdentityRepository`: identity, credential, factor and recovery-code rows, with lookup by normalised email.
      Covers: RF-2.1, RF-2.6, LC-19 · Plan: §2.1 persistence
      After: T024
      Done when: an integration test stores an identity and finds it by a mixed-case, space-padded email.

- [ ] T027 [P] Implement the `LoginTransactionRepository` adapter: `findForUpdate` using `SELECT … FOR UPDATE`, `save`, `saveRecordedOutcome`.
      Covers: RF-5.1, RF-7.3, LC-9, LC-13, LC-14 · Plan: §2.1 persistence, D1
      After: T024
      Done when: an integration test opens two transactions on the same row and observes the second `findForUpdate` blocking until the first commits.

- [ ] T028 [P] Implement the `ThrottleStore` adapter: upsert of the `(dimension, subject_key)` row inside the attempt's transaction; read of counter and restriction window; keys namespaced so the reset feature's store can never collide.
      Covers: RF-4.1, RF-4.4, RF-4.7, LC-10 · Plan: §2.1 persistence, D3
      After: T024
      Done when: an integration test runs ten parallel upserts on one key and reads a count of ten; a key written for a reset dimension is invisible to the sign-in dimensions.

- [ ] T029 Implement the `PasswordVerifier` adapter over the maintained Spring Security `PasswordEncoder`, a fixed dummy hash produced with the same parameters at startup, and a bounded worker pool through which every verification runs.
      Covers: RF-2.1, RF-2.3, RF-4.6, RNF-1, RNF-3 · Plan: §2.1 Spring Security integration, D2
      Done when: a test shows dummy and real verification within the same cost order, and a burst above the pool bound is queued or refused without unbounded threads.

- [ ] T030 [P] Implement the `TotpVerifier` adapter over the existing MFA support (tolerance from `LoginPolicy`) and the `SeedCipher` adapter over `sharedsupport/crypto`.
      Covers: RF-6.4, RF-6.5, RF-6.11, RNF-7 · Plan: D4, D6
      Done when: an integration test verifies a code generated from a known seed at a fixed clock, and the stored column for that factor contains only ciphertext.

- [ ] T031 [P] Implement the audit adapter over `audit.AuditPort`: map the eight outcomes to events with payload identity id (when matched), correlation id, network key, outcome kind, timestamp; append in the attempt's transaction.
      Covers: RF-10.1, RF-10.2, RF-10.3, RNF-7 · Plan: §2.1 audit adapter, D11
      Done when: `audit.entryPerOutcome` and `audit.containsNoSecret` pass on real PostgreSQL.

- [ ] T032 [P] Implement the `IssueSessionPort` adapter to the existing session service, the `MembershipProjectionPort` consumption from `access` as an identity-scoped read for the issued session's identity (A-36), and the `Clock` adapter.
      Covers: RF-8.1, RF-8.5, RF-8.7 · Plan: D7, D8, F-X-01, register A-36
      Done when: an integration test with a stubbed session service sees exactly one issue call and a projection read scoped to the new session's identity.

- [ ] T033 Implement the Spring Security integration: an authentication entry delegating to `SignInUseCase`; a transaction-token authority that grants only the three challenge endpoints and derives no C10; the session issue adapter behind `IssueSessionPort`; no change to JWT / refresh mechanics.
      Covers: RF-5.2, RF-8.1, RNF-9 · Plan: §2.1 Spring Security integration, D1
      After: T029, T032
      Done when: `transactionToken.cannotCallProtectedEndpoints` passes for every non-auth route in the application.

---

## Phase 4 — Back-end API (`AuthController`, `/api/auth`; contract tests with MockMvc)

- [ ] T034 Implement the problem-details support for `/api/auth`: RFC 9457 bodies with a safe code, field paths for validation, the correlation id from `sharedsupport/contextIds`, and no echo of any request value.
      Covers: RF-1.6, RF-2.5, RF-10.2, RF-10.3 · Plan: D12
      After: T033
      Done when: `authApi.fieldPathProblems` passes and a serialised problem for a malformed request contains none of the submitted values.

- [ ] T035 Implement `POST /api/auth/sign-in` with server-side validation: 200 `{state: LIMITED, transaction}`, 200 `{state: COMPLETED, landing: {decision, projection}}`, 400 field-path problem, 401 `auth.sign-in.failed`, 409 `auth.verification-pending {pendingRef, email}`.
      Covers: RF-1.6, RF-2.2, RF-2.5, RF-3.1, RF-4.3, RF-5.1, RF-5.5 · Plan: §2.1 `AuthController`
      Done when: `signIn.failureIsIdenticalAcrossClasses` passes at contract level (byte-identical 401 bodies for unknown, wrong, restricted and throttled) and the four other outcomes each have a green MockMvc case.

- [ ] T036 Implement `POST /api/auth/challenge/totp` and `POST /api/auth/challenge/recovery-code` (each request carries an idempotency key): 200 `{state: COMPLETED, landing}`, 400 `auth.challenge.rejected` with no cause and no attempt count, 410 `auth.challenge.void` with no cause for a voided or already completed transaction; a repeated key returns the recorded outcome.
      Covers: RF-6.4, RF-6.6, RF-6.7, RF-6.8, RF-7.1, RF-7.2, RF-7.4, LC-24 · Plan: §2.1 `AuthController`
      Done when: `challengeRejected.carriesNoCauseOrCount` passes, a 410 produced by exhaustion and one produced by expiry have identical bodies except the correlation id, and a second request with the same key returns the first response.

- [ ] T037 Implement `DELETE /api/auth/challenge` (204, idempotent) and `GET /api/auth/landing` (session required; `{decision, projection}` for a reload of the landing route; no redirect).
      Covers: RF-5.4, RF-8.1, RF-8.2, RF-8.3, RF-8.4 · Plan: §2.1 `AuthController`, D7
      Done when: MockMvc shows a second `DELETE` still answering 204, `GET landing` without a session answering 401, and with a session returning the `LandingRule` decision.

- [ ] T038 Write the redaction and no-device-trust contract tests over the whole `/api/auth` surface.
      Covers: RF-5.2, RF-6.10, RF-6.11, RNF-9 · Plan: D1, D6
      Done when: `limitedState.exposesNoIdentityTenantOrProjectData`, `responses.neverContainSeedOrFactorData` and the API half of `auth.noDeviceTrustSurfaceExists` (no endpoint, no cookie, no header) pass.

---

## Phase 5 — Back-end integration, timing, concurrency and startup evidence (real PostgreSQL)

- [ ] T039 Write the sign-in integration suite, part A (happy and silent paths).
      Covers: RF-2.1, RF-2.4, RF-2.6, RF-5.1, RF-5.5, LC-18, LC-19
      After: T038
      Done when: `signIn.verifiesNormalisedEmailAndPassword`, `signIn.unknownEmailCreatesNoIdentityAndStoresNoPassword`, `signIn.enrolledFactorYieldsLimitedState`, `signIn.noFactorCompletes` pass.

- [ ] T040 [P] Write the sign-in integration suite, part B (verification and status branches).
      Covers: RF-2.7, RF-3.1, RF-3.2, RF-3.3, LC-1, LC-2
      After: T038
      Done when: `signIn.unverifiedLandsOnPending`, `signIn.unverifiedWrongPasswordGeneric`, `signIn.restrictedIdentityIsGenericAndCounted` pass.

- [ ] T041 [P] Write the throttle integration suite, part A (delay and restriction behaviour).
      Covers: RF-4.1, RF-4.2, RF-4.3, RF-4.4, LC-11, LC-12
      After: T038
      Done when: `signIn.appliesDelayPerDimension`, `throttle.restrictionLiftsAfterWindow`, `throttle.correctPasswordRefusedWhileRestricted`, `throttle.networkRestrictionCoversOtherAccounts` pass.

- [ ] T042 [P] Write the throttle integration suite, part B (no lockout, independence from reset).
      Covers: RF-4.4, RF-4.7, RNF-2
      After: T038
      Done when: `throttle.noPermanentLockout` (long attacker sequence, then legitimate success after the window) and `throttle.resetRequestsDoNotCountForSignIn` pass.

- [ ] T043 [P] Write the challenge integration suite, part A (authenticator codes).
      Covers: RF-6.4, RF-6.5, RF-6.6, RF-7.1, RF-7.2, LC-3, LC-4, LC-5, LC-6, LC-24
      After: T038
      Done when: `totpVerify.validCodeCompletes`, `totpVerify.validOnLastAttemptCompletes`, `totpVerify.rejectsReplayAtOrBelowWatermark`, `totpVerify.lastWrongAttemptVoids`, `totpVerify.expiredTransactionVoidsAndDoesNotConsumeCode`, `challenge.resubmitReturnsRecordedOutcome` pass.

- [ ] T044 [P] Write the challenge integration suite, part B (recovery codes).
      Covers: RF-6.7, RF-6.8, LC-7, LC-8
      After: T038
      Done when: `recoveryCode.consumesOnce`, `recoveryCode.consumedTreatedAsWrong`, `recoveryCode.lastCodeCompletesLeavingZero` pass.

- [ ] T045 [P] Write the challenge integration suite, part C (void, abandon, refusal).
      Covers: RF-4.5, RF-5.4, RF-7.3, RF-7.5, LC-22
      After: T038
      Done when: `challengeVoid.countsOnBothDimensions`, `abandon.voidsAndIsIdempotent`, `voidedTransaction.refusesAllCalls` pass.

- [ ] T046 [P] Write the concurrency suite with parallel database transactions.
      Covers: LC-9, LC-10
      After: T038
      Done when: `twoTabs.oneSuccessOtherRejected` and `parallelTransactions.independentAttemptsSharedThrottle` pass ten consecutive runs.

- [ ] T047 [P] Write the transaction-voiding suite driven by bumping the identity's security version (as a password reset and a factor change do).
      Covers: LC-13, LC-14
      After: T038
      Done when: `transaction.voidedByPasswordChange` and `transaction.voidedByFactorChange` pass.

- [ ] T048 [P] Write the completion and landing suite.
      Covers: RF-8.1, RF-8.6, RF-8.7, LC-15
      After: T038
      Done when: `complete.issuesSessionAndTouchesNoRunTables` (query log shows no run, approval or mapping table) and `landingQuery.usesCurrentMembershipsAfterRevocation` pass.

- [ ] T049 [P] Build the timing-comparability harness: N samples per failure class (unknown, wrong, restricted, throttled) against the live endpoint, distributions compared within a configured tolerance.
      Covers: RF-2.2, RF-2.3, RNF-1
      After: T038
      Done when: `signIn.timingComparable` passes in CI with the tolerance recorded in the test configuration.

- [ ] T050 [P] Write the bounded-workload flood test.
      Covers: RF-4.6, RNF-3
      After: T038
      Done when: `signIn.boundedVerificationWorkUnderFlood` shows verifier executions per second capped at the pool bound while the flood runs.

- [ ] T051 [P] Write the fail-closed and log-redaction suite.
      Covers: RNF-7, RNF-8, RF-10.3
      After: T038
      Done when: `failClosed.absentExpiredOrInvalidRefuses` (missing transaction, malformed code, missing configuration value) and `logs.containNoCredentialOrCode` (captured log output searched for the submitted secrets) pass.

- [ ] T052 [P] Write the startup property matrix: one Spring Boot context test per `LoginPolicy` property, absent and unbounded.
      Covers: RF-4.8, RF-5.3, RNF-4
      After: T025
      Done when: `loginPolicy.startupFailsOnAbsentOrUnboundedValue` has one green case per property and the complete set starts.

---

## Phase 6 — Front-end shared adapters and primitives (Vitest + Testing Library)

- [ ] T053 Configure `shared/api` for Login: axios instance sending the correlation id header and performing no automatic retry on `POST` or `DELETE`.
      Covers: RF-10.2, LC-24 · Plan: §2.2 `shared/api`, F-X-02
      After: T003
      Done when: a unit test sees the header on every request and a failed `POST` issued exactly once.

- [ ] T054 [P] Implement `shared/errors` problem parsing: RFC 9457 body → typed safe problem; field paths mapped to form fields; unknown codes become a generic problem.
      Covers: RF-1.6, RF-2.5, RF-6.6, RF-7.4 · Plan: §2.2 `shared/errors`, D12
      After: T003
      Done when: unit tests parse the four auth codes, one field-path problem, and confirm that a 410 body with no cause yields a problem without a reason.

- [ ] T055 [P] Implement the `shared/ui` primitives `Field` (label, `aria-describedby` error association, `aria-invalid`), `Button` (pending state, disabled resubmit) and `LiveRegion` (polite, never focused), with 44 px targets at touch widths.
      Covers: RF-1.6, RF-6.3, RF-9.3, RNF-5 · Plan: §2.2 `shared/ui`
      After: T002
      Done when: Testing Library tests confirm the associations and the pending state, and axe reports no violation on a form built from the three primitives.

- [ ] T056 [P] Implement `PasswordField` with the show / hide toggle; the value lives only in component state.
      Covers: RF-1.5, RF-9.3 · Plan: §2.2 `shared/ui`
      After: T055
      Done when: `passwordField.showToggleNeverPersists` passes (spies on `localStorage`, `sessionStorage` and the URL see no write).

- [ ] T057 [P] Implement `CodeField`: a single labelled input accepting paste and stripping separators client-side.
      Covers: RF-6.2, LC-23, RNF-5 · Plan: §2.2 `shared/ui`, D15
      After: T055
      Done when: `codeField.pasteWithSeparators` passes (pasted `123 456`, `123-456` and `123.456` all submit `123456`).

- [ ] T058 [P] Write the ES and FR bundles `identity-access.{es,fr}.json` from the EN catalogue, register the feature namespace in `shared/i18n`, and add a key-parity test.
      Covers: RF-9.2, RNF-6 · Plan: §2.2 `shared/i18n`, D16
      After: T004
      Done when: the key-parity test proves the three bundles hold identical key sets and no value is empty.

---

## Phase 7 — Front-end feature state and hooks

- [ ] T059 Implement the module-private transaction holder: in-memory `set` / `get` / `clear`; nothing written to browser storage or the URL.
      Covers: RF-5.4, RF-7.3, LC-22, RNF-7 · Plan: §2.2 transaction holder, D13
      After: T053
      Done when: `state.containsNoSecret` passes (storage and URL spies see nothing after set / get / clear).

- [ ] T060 Implement `useSignIn()`: `useMutation` without retry; `LIMITED` → holder set; `COMPLETED` → scope epoch advanced; 409 → `VERIFICATION_PENDING` result; 401 → generic failure result; the result is returned verbatim, never interpreted.
      Covers: RF-2.5, RF-3.1, RF-4.3, RF-5.1, RF-5.5, RF-8.7 · Plan: §2.2 `useSignIn()`, P3
      After: T059
      Done when: unit tests with a mocked transport produce the four outcomes and the holder holds the token only after `LIMITED`.

- [ ] T061 Implement `useChallenge()`: mutations for totp, recovery-code and abandon (no retry; each attempt generates its idempotency key once and reuses it on re-submission); pending flag while in flight; 400 → rejected result; 410 → holder cleared and `VOID` result; a reload during pending leaves the holder and the attempt's key usable for a re-submission of the same attempt.
      Covers: RF-6.3, RF-6.6, RF-6.7, RF-6.8, RF-7.1, RF-7.2, RF-7.3, RF-7.4, LC-24 · Plan: §2.2 `useChallenge()`
      After: T059
      Done when: `challengePage.reloadDuringPendingResubmitsSameAttempt` passes (same key sent twice) and a 410 leaves the holder empty.

- [ ] T062 Implement `useLanding()`: takes `landing` from the completed response, or calls `GET landing` only when the landing route is reloaded; in both cases only after `shared/security` has advanced the scope epoch and cleared scoped caches.
      Covers: RF-8.1, RF-8.7 · Plan: §2.2 scope epoch
      After: T060
      Done when: `completion.advancesEpochBeforeLanding` passes (call order asserted).

- [ ] T063 Implement `LandingRouter` as a pure decision-to-route map that never evaluates the landing rule: `CREATE_WORKSPACE` → workspace creation route; `ENTER_TENANT(id)` → plain navigation to the tenant route; `TENANT_CHOOSER` → chooser route; no state, intent or authority is passed along.
      Covers: RF-8.2, RF-8.3, RF-8.4, RF-8.5, RF-8.6 · Plan: §2.2 `LandingRouter`, D7
      After: T062
      Done when: `landingRouter.mapsEachDecision` covers the three decisions and asserts the navigation call carries only the route.

---

## Phase 8 — Front-end views and routing

- [ ] T064 Implement `SecurityStatements` with two variants (sign-in: RF-1.2, RF-1.3, RF-1.4; challenge: RF-6.1), every string from the i18n bundle.
      Covers: RF-1.2, RF-1.3, RF-1.4, RF-6.1, RF-9.2, RNF-10 · Plan: §2.2 `SecurityStatements`, D14
      After: T058
      Done when: a unit test asserts every statement key of each variant is rendered, and the component is the only place those keys are used.

- [ ] T065 [P] Implement `GenericFailureMessage`: the single RF-2.5 text, focusable, associated with the form, receiving focus on mount.
      Covers: RF-2.5, RF-4.3, RNF-5 · Plan: §2.2 `GenericFailureMessage`
      After: T058
      Done when: a unit test confirms focus lands on the message when it appears and the text equals the RF-2.5 key.

- [ ] T066 Implement `SignInPage` at desktop (R02): email `Field`, `PasswordField`, "Sign in", password-reset route, account-creation route, lost-authenticator route, language switch, `SecurityStatements`; "Need help?" link opening N43 HelpShortcutsPanel (`430:4129`) (S-17); client validation for empty and malformed email; no password rules shown.
      Covers: RF-1.1, RF-1.2, RF-1.3, RF-1.4, RF-1.6, RF-1.7, LC-20 · Plan: §2.2 `SignInPage`, register S-17
      After: T060, T064, T065
      Done when: `signInPage.rendersFieldsRoutesAndStatements` (desktop), `signInPage.blocksEmptyOrMalformedFields` and `signInPage.malformedEmailBlocked` pass.

- [ ] T067 Add failure and handoff behaviour to `SignInPage`: generic failure keeps the email, clears the password, focuses the message; `VERIFICATION_PENDING` navigates to the verification route in pending state; arrival after a void shows the plain form with no reason text; an existing session is ignored (form shown).
      Covers: RF-2.5, RF-3.1, RF-7.3, RF-7.4 · Plan: D9, D13, register S-26
      Done when: `signInPage.genericFailureKeepsEmailClearsPasswordFocuses` and the unit form of `signInPage.afterVoidShowsNoReason` pass.

- [ ] T068 Add the M02 reflow to `SignInPage` with Tailwind breakpoints in the same component: identical fields, actions, routes and statements; 44 px targets; visible labels and focus.
      Covers: RF-9.1, RF-9.2, RF-9.3, RNF-10 · Plan: D14, F-X-07
      Done when: `signInPage.rendersFieldsRoutesAndStatements` passes at M02 width and a repository search finds no second sign-in component.

- [ ] T069 Implement `MfaChallengePage` (R05): `CodeField`, "Verify & continue", "Use a recovery code" toggling the input mode in place, controlled-recovery entry, "Need help?" link opening N43 HelpShortcutsPanel (`430:4129`), challenge `SecurityStatements`, pending state with resubmit disabled; no device-trust control; no identity, tenant or personal data rendered.
      Covers: RF-5.2, RF-6.1, RF-6.2, RF-6.3, RF-6.10, RF-6.11 · Plan: §2.2 `MfaChallengePage`
      After: T061, T064
      Done when: `challengePage.rendersActionsStatementsAndPendingState` and the FE half of `auth.noDeviceTrustSurfaceExists` pass.

- [ ] T070 Add leave and handoff behaviour to `MfaChallengePage`: route leave or browser back fires abandon and clears the holder; controlled recovery abandons then routes to its entry; a `VOID` result clears the holder and navigates to `/sign-in`; a rejected result shows the generic "code not accepted" message and keeps the page.
      Covers: RF-5.4, RF-6.6, RF-6.9, RF-7.1, RF-7.3, LC-22 · Plan: D13
      Done when: `challengePage.leaveCallsAbandonAndClearsHolder` passes and the void path leaves no transaction in the holder.

- [ ] T071 Implement the mobile layout of `MfaChallengePage` as M12 draws it (`431:4162`), by the same breakpoint rules as T068.
      Covers: RF-9.4, RNF-5 · Plan: register S-03
      Done when: the page at M02 width renders every action and statement and axe reports no violation.

- [ ] T072 Register the routes `/sign-in` and `/sign-in/challenge` through the module's public API in `app`; guard the challenge route so that an empty holder redirects to `/sign-in`.
      Covers: RF-1.1, RF-5.1, RF-7.3 · Plan: §2.2 routes, D14
      After: T066, T069
      Done when: route tests show the guard redirect and the lint rule confirms `app` imports only the module's `index.ts`.

---

## Phase 9 — Front-end end-to-end, accessibility and language evidence (Playwright + axe, API stubbed with fixed answers)

- [ ] T073 Write the e2e journey: sign-in without an enrolled factor → landing for each decision (`CREATE_WORKSPACE`, `ENTER_TENANT`, `TENANT_CHOOSER`); password-reset route reachable from the form.
      Covers: US-1, US-5, US-6, US-9, RF-5.5, RF-8.2, RF-8.3, RF-8.4
      After: T072
      Done when: three green Playwright cases, one per decision, each ending on the expected route.

- [ ] T074 [P] Write the e2e journey: sign-in → challenge → TOTP success; recovery-code success; rejected code stays on page; void returns to `/sign-in` with no reason; abandon on back; controlled recovery voids and routes.
      Covers: US-2, US-3, US-7, RF-6.3, RF-6.4, RF-6.6, RF-6.7, RF-6.9, RF-7.1, RF-7.4
      After: T072
      Done when: `challengePage.controlledRecoveryVoidsAndRoutes` and the e2e form of `signInPage.afterVoidShowsNoReason` pass alongside the success cases.

- [ ] T075 [P] Write the e2e journey: correct password on an unverified identity → verification pending with resend; generic failure keeps email and focuses the message; language switch on the challenge keeps the transaction.
      Covers: US-4, RF-2.5, RF-3.1, RNF-6, LC-21
      After: T072
      Done when: `languageSwitch.onChallengeKeepsTransaction` passes and the verification route is reached with no transaction in the holder.

- [ ] T076 [P] Write the e2e case with a tenant-entry stub: an Owner without a factor signs in without a challenge and is gated on tenant entry.
      Covers: RF-8.5, LC-18
      After: T072
      Done when: `landing.ownerWithoutFactorIsGatedOnTenantEntry` passes.

- [ ] T077 [P] Run axe over sign-in, challenge and landing in EN, ES and FR at desktop and M02 widths.
      Covers: US-8, US-10, RF-9.1, RF-9.2, RF-9.3, RF-9.4, RNF-5, RNF-6, RNF-10
      After: T072
      Done when: `a11y.signInChallengeLanding.{en,es,fr}.{desktop,m02}` reports zero WCAG 2.2 AA violations in CI.

- [ ] T078 Execute the manual assistive checklist on NVDA (Chrome, Firefox, Windows) and VoiceOver (Safari, macOS and iOS) with the versions pinned in the release-support matrix: focus order, announcements, no focus theft by the live region, labels, 44 px targets.
      Covers: RF-9.3, RNF-5
      After: T077
      Done when: the checklist document records a pass for every item on every pinned reader, with tester and date.

---

## Phase 10 — Exit

- [ ] T079 Run the traceability self-check: every RF criterion (60), RNF (10) and limit case (25) maps to at least one green test; every test named in plan §5.2 exists in the suites; adversarial suites (enumeration, lockout, replay, hijack) are green.
      Covers: specification §7 items 1–10, US-11
      After: T052, T078
      Done when: the generated traceability report lists no uncovered item and CI is green on the full back-end and front-end suites.

- [ ] T080 Close the register dependencies: A-27, A-29, A-30, A-31, A-32, A-34, A-35, A-36, S-03, S-17, S-26 are each resolved in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) or explicitly deferred by the product owner, and any resolution that differs from the plan's assumption is turned into a follow-up task.
      Covers: specification §7 item 11 · Plan: §6
      Done when: each of the eleven items has a recorded "resolved" or "deferred" state with the owner's name and date.

---

## Coverage table

Every specification item and the tasks that implement or prove it.

| Item | Tasks |
|---|---|
| RF-1.1 | T066, T072 |
| RF-1.2, RF-1.3, RF-1.4 | T004, T064, T066 |
| RF-1.5 | T056 |
| RF-1.6 | T003, T034, T035, T054, T055, T066 |
| RF-1.7 | T066 |
| RF-2.1 | T007, T014, T026, T029, T039 |
| RF-2.2 | T015, T035, T049 |
| RF-2.3 | T015, T029, T049 |
| RF-2.4 | T015, T039 |
| RF-2.5 | T003, T004, T034, T035, T054, T060, T065, T067, T075 |
| RF-2.6 | T006, T024, T026, T039 |
| RF-2.7 | T006, T015, T040 |
| RF-3.1 | T016, T035, T040, T060, T067, T075 |
| RF-3.2, RF-3.3 | T016, T040 |
| RF-4.1 | T009, T017, T028, T041 |
| RF-4.2 | T009, T017, T041 |
| RF-4.3 | T017, T035, T041, T060, T065 |
| RF-4.4 | T009, T017, T028, T041, T042 |
| RF-4.5 | T019, T045 |
| RF-4.6 | T017, T029, T050 |
| RF-4.7 | T028, T042 |
| RF-4.8 | T005, T025, T052 |
| RF-5.1 | T003, T014, T027, T035, T039, T060, T072 |
| RF-5.2 | T003, T033, T038, T069 |
| RF-5.3 | T005, T008, T014, T024, T025, T052 |
| RF-5.4 | T008, T021, T037, T045, T059, T070 |
| RF-5.5 | T003, T014, T035, T039, T060, T073 |
| RF-6.1 | T004, T064, T069 |
| RF-6.2 | T010, T019, T057, T069 |
| RF-6.3 | T055, T061, T069, T074 |
| RF-6.4 | T019, T030, T036, T043, T074 |
| RF-6.5 | T010, T019, T030, T043 |
| RF-6.6 | T003, T004, T019, T036, T043, T054, T061, T070, T074 |
| RF-6.7 | T011, T020, T036, T044, T061, T074 |
| RF-6.8 | T011, T020, T036, T044, T061 |
| RF-6.9 | T021, T070, T074 |
| RF-6.10 | T038, T069 |
| RF-6.11 | T003, T010, T024, T030, T038, T069 |
| RF-7.1 | T008, T019, T036, T043, T061, T070, T074 |
| RF-7.2 | T008, T019, T036, T043, T061 |
| RF-7.3 | T008, T023, T027, T045, T059, T061, T067, T070, T072 |
| RF-7.4 | T003, T036, T054, T061, T067, T074 |
| RF-7.5 | T019, T045 |
| RF-8.1 | T003, T018, T022, T032, T033, T037, T048, T062 |
| RF-8.2, RF-8.3, RF-8.4 | T012, T022, T037, T063, T073 |
| RF-8.5 | T022, T032, T063, T076 |
| RF-8.6 | T018, T048, T063 |
| RF-8.7 | T022, T032, T048, T060, T062 |
| RF-9.1 | T068, T077 |
| RF-9.2 | T058, T064, T068, T077 |
| RF-9.3 | T055, T056, T068, T077, T078 |
| RF-9.4 | T071, T077 |
| RF-10.1 | T013, T031 |
| RF-10.2 | T031, T034, T053 |
| RF-10.3 | T007, T031, T034, T051 |
| RNF-1 | T015, T029, T049 |
| RNF-2 | T009, T042 |
| RNF-3 | T029, T050 |
| RNF-4 | T005, T025, T052 |
| RNF-5 | T055, T057, T065, T071, T077, T078 |
| RNF-6 | T004, T058, T075, T077 |
| RNF-7 | T024, T030, T031, T051, T059 |
| RNF-8 | T025, T051 |
| RNF-9 | T033, T038 |
| RNF-10 | T064, T068, T077 |
| LC-1 | T016, T040 |
| LC-2 | T006, T015, T040 |
| LC-3 | T010, T019, T043 |
| LC-4, LC-5, LC-6 | T008, T019, T043 |
| LC-7, LC-8 | T011, T020, T044 |
| LC-9 | T027, T046 |
| LC-10 | T028, T046 |
| LC-11, LC-12 | T017, T041 |
| LC-13, LC-14 | T008, T019, T023, T047 |
| LC-15 | T012, T022, T048 |
| LC-16, LC-17 | T012 |
| LC-18 | T014, T039, T076 |
| LC-19 | T006, T026, T039 |
| LC-20 | T066 |
| LC-21 | T075 |
| LC-22 | T021, T045, T059, T070 |
| LC-23 | T010, T057 |
| LC-24 | T008, T019, T036, T043, T053, T061 |
| LC-25 | T012, T014, T063 |
| US-1 … US-10 | T073, T074, T075, T076, T077 (journeys); US-11 by T079 (adversarial suites) |

## Register assumptions built in

| Item | Built as (plan §6) | Task |
|---|---|---|
| A-27 identity statuses | `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING`; `ACTIVE` signs in, `UNVERIFIED` takes the verification branch, the others the generic branch | T006, T015, T016 |
| A-29 after recovery-code sign-in | No follow-up; landing proceeds normally | T020, T063 |
| A-30 sign-in notifications | `SecurityNotificationPort` declared here; first caller plan 002; no sign-in event | T013 |
| A-34 pending reference | Issued by the verification-pending branch after the password is verified | T016, T035 |
| A-35 persistence tooling | Access API and migration tool as placeholders | T024 |
| A-36 identity-scoped membership read | `MembershipProjectionPort` read for the signed-in identity | T022, T032 |
| A-31 re-authentication from R34 | Out of scope; nothing built | — |
| A-32 time tolerance | One global value in `LoginPolicy` | T005, T030 |
| S-03 mobile challenge | `MfaChallengePage` follows M12 | T071 |
| S-17 "Need help?" | Opens N43 HelpShortcutsPanel (`430:4129`) | T066 |
| S-26 already signed in | Form shown; new session issued on success | T067 |

**Totals.** 80 tasks: Phase 0 — 4 · Phase 1 — 8 · Phase 2 — 11 · Phase 3 — 10 · Phase 4 — 5 · Phase 5 — 14 · Phase 6 — 6 · Phase 7 — 5 · Phase 8 — 9 · Phase 9 — 6 · Phase 10 — 2.
