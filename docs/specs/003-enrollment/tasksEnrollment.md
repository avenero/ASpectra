# Tasks 003 — TOTP enrollment, recovery codes and factor replacement

**Feature** Second-factor enrollment, recovery codes and authenticator replacement · **Tasks version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Derived from** `specEnrollment.md` v0.2 (requirements RF-x.y, RNF-n, LC-n, US-n; decisions Q1–Q20) and `planEnrollment.md` v0.2 (modules §2, trace §3, decisions D1–D19, tests §5, register items §6) · **Governed by** `constitution.md` v1.8.

This document is the DO. Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2 so each task can be checked against the test matrix. Tasks are sized at 20–30 minutes for one developer and listed in dependency order; "after" names the tasks that must be complete first when they are not simply the preceding ones. A later task may assume everything above it. Phase F (front end) depends only on phase A and may run alongside phases B–E.

Elements tagged **[001]**, **[002]** or **[004]** are owned by plans 001 (Login), 002 (Signup and verification) and 004 (Password reset); they are extended here, never duplicated. If the owning plan is not built yet, the element is built here in the shape that plan gives it. Test-only tools (plan §5.1; none enters the product build): JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 001's, 002's and 004's. The QR encoding library is ZXing `core` 3.5.3 (A-46, decided 2 November 2026); the `QrEncoder` port isolates it to one adapter (T-31).

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing.

---

## A · Foundations

- [ ] **T-01 · Enrollment configuration record** — `EnrollmentPolicy` immutable record and `EnrollmentPolicyProperties` (`@ConfigurationProperties` + `@Validated`): recovery-code count, code entropy in bits, pending-enrollment expiry, verification attempt limit, step-up expiry, step-up attempt limit, expiry-job cadence and batch size; each present, positive and bounded by a configured ceiling, entropy at or above a configured floor. The authenticator time tolerance is **not** duplicated: it is read from `LoginPolicy` **[001]** (A-32). *(plan §2.1 `EnrollmentPolicy`, `EnrollmentPolicyProperties`, D17; RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed, set non-positive or set above its ceiling (one failing case per property); the policy exposes the tolerance of `LoginPolicy` and holds no tolerance field of its own.

- [ ] **T-02 · Migration: factor and recovery-code extensions [001]** — `identity_totp_factor` gains `enrolled_at` and `seed_version`; `identity_recovery_code` gains `set_id` and `consumed_at`. Existing plan 001 rows get `seed_version = 1`, a `set_id` per identity and `enrolled_at` from their creation time. *(plan §2.1 Persistence, D4; RF-5.1, RF-7.3, RF-7.6, RF-9.6, LC-12)* — after T-01
  **Done when:** the migration applies after plan 001's on an empty database and on one holding factor and recovery-code rows, and plan 001's challenge tests still pass against the extended tables.

- [ ] **T-03 · Migration: `pending_enrollment`** — columns `id`, `identity_id`, `purpose`, `state`, `encrypted_seed`, `key_ref` (both nullable for `REGENERATE`), `replay_watermark`, `expires_at`, `attempts_remaining`, `owning_session_id`, `revealed_at`, `discard_reason`, `created_at`, plus an outcome table or column keyed by idempotency key; partial unique index on `identity_id WHERE state NOT IN ('ACKNOWLEDGED','DISCARDED')`. *(plan §2.1 Persistence, D1, D13; RF-6.1, RF-6.2, LC-8)* — after T-02
  **Done when:** the migration applies; a second non-terminal row for one identity is rejected by the index while any number of `ACKNOWLEDGED` or `DISCARDED` rows coexist; `identity_id` cascades on identity delete.

- [ ] **T-04 · Migration: `pending_recovery_code` and `step_up_challenge`** — `pending_recovery_code` (`pending_id` → cascade delete, `code_hash`); `step_up_challenge` (`id`, `identity_id`, `purpose`, `state`, `expires_at`, `attempts_remaining`, `consumed_by` nullable → `pending_enrollment.id`, `created_at`, idempotency outcomes). *(plan §2.1 Persistence, D3, D5; RF-4.8, RF-8.3, RF-9.2)* — after T-03
  **Done when:** the migration applies; deleting a pending enrollment deletes its pending codes; a `consumed_by` value that names no pending enrollment is rejected.

- [ ] **T-05 · Outbox templates registered** — template ids `enrollment-completed`, `recovery-codes-regenerated`, `authenticator-replaced` with the outbox message type **[002]** in EN, ES and FR; each states the event and its time and carries no code, seed, count by value or link that grants access (D04 copy `437:4431`). *(plan §2.1 Outbox relay and email adapter, D11; RF-5.5, RF-8.9, RF-9.7, RF-11.4, RNF-9)* — after T-01
  **Done when:** rendering each template in the three locales with sample variables yields a non-empty subject and body containing the event time, and a template-variable guard rejects any variable named like a code, seed, key or URL.

- [ ] **T-06 · Problem-code catalogue** — register the RFC 9457 codes `mfa.enrollment.already-enrolled` (409), `mfa.code.rejected` (400), `mfa.enrollment.restart` (410), `mfa.step-up.rejected` (400), `mfa.step-up.void` (410), the field problem for an unticked acknowledgement (400) and `auth.enrollment.required {reason}` (403, emitted by `access`); reuse `auth.verification-pending` **[001]**. Each carries code, title, field paths, correlation id and nothing else. *(plan §2.1 `MfaController`, D19; RF-3.6, RF-4.4, RF-8.3, RF-8.4, RF-11.2)* — after T-01
  **Done when:** a serialisation test renders one problem per code and finds only the five permitted members; `auth.enrollment.required` carries `reason ∈ {OWNER_ADMIN, MEMBER_POLICY}` and no tenant data.

## B · Domain (pure, no Spring)

- [ ] **T-07 · `TotpFactor` extension [001]** — add `enrolledAt`, `seedVersion` and `replaceWith(pendingSeedRef, now)`, which returns the successor factor (new seed reference, `seedVersion + 1`, `enrolledAt = now`, watermark reset) and marks the previous seed invalid in the same value change; no accessor returns the seed; `toString` redacted. *(plan §2.1 `TotpFactor`, D4, D8; RF-5.1, RF-7.3, RF-9.6, RF-11.3, RNF-4)* — after T-01
  **Done when:** unit tests show `replaceWith` yields a successor with the three fields set and the predecessor unusable, and a reflection test finds no public method returning the seed or key material.

- [ ] **T-08 · `RecoveryCodeSet` extension [001]** — add `setId`, `unusedCount()` and `invalidateAll(now)`; `consume` **[001]** is unchanged and stays the only reader of the hashes; a consumed or invalidated code is never accepted again. *(plan §2.1 `RecoveryCodeSet`, D3, D4; RF-4.8, RF-7.6, RF-8.6, RF-9.6, LC-12)* — after T-01
  **Done when:** unit tests show `unusedCount` falls by one per `consume`, reaches zero after the last code, and `invalidateAll` makes every remaining code refuse `consume`.

- [ ] **T-09 · `PendingEnrollment` — fields and state machine** — identity id, `purpose` (`INITIAL`, `REPLACEMENT`, `REGENERATE`), states `SET_UP → VERIFIED → REVEALED → ACKNOWLEDGED | DISCARDED`, encrypted seed and key reference (absent for `REGENERATE`), `replayWatermark`, `expiresAt`, `attemptsRemaining`, `owningSessionId`, `revealedAt`, pending hashes, `discardReason` (`SUPERSEDED`, `LEFT`, `SECOND_REVEAL`, `EXPIRED`, `ATTEMPTS`, `SESSION_ENDED`); factories `start(purpose, …)` in `SET_UP` and `forRegeneration(…)` directly in `VERIFIED`; the clock is always an argument. *(plan §2.1 `PendingEnrollment`, D1; RF-2.1, RF-3.2, RF-6.1, RF-6.3, RF-6.4, LC-1, LC-2, LC-3)* — after T-01
  **Done when:** a table test lists every allowed transition and every illegal one throws; `forRegeneration` holds no seed and starts in `VERIFIED`; a `PendingEnrollment` in any state answers `isActiveFactor() == false`.

- [ ] **T-10 · `PendingEnrollment.verify(codeOk, now)`** — refuses when not `SET_UP` or when `now ≥ expiresAt` (→ `discard(EXPIRED)`); on `codeOk` advances the watermark to the accepted time-step and moves to `VERIFIED`; otherwise decrements `attemptsRemaining` and at zero `discard(ATTEMPTS)`. A time-step at or below the watermark counts as not ok. *(plan §2.1, D1, D8; RF-3.2, RF-3.4, RF-3.5, RF-3.7, LC-5, LC-6, LC-7)* — after T-09
  **Done when:** unit tests with a fixed clock pass for success, wrong code, last-attempt discard and expiry discard, and `verify.rejectsReplayAtOrBelowPendingWatermark` (domain half) passes.

- [ ] **T-11 · `PendingEnrollment.reveal / acknowledge / discard / recordedOutcome`** — `reveal(hashes, now)` only from `VERIFIED` (stores the hashes, `REVEALED`, `revealedAt`); a second `reveal` on a `REVEALED` row discards with `SECOND_REVEAL`; `acknowledge(now)` only from `REVEALED` and clears the pending hashes; `discard(reason, now)` is idempotent and clears seed and hashes; `recordOutcome(key, outcome)` / `recordedOutcome(key)` per idempotency key. *(plan §2.1, D1, D3, D9, D13; RF-4.1, RF-4.6, RF-4.8, RF-6.3, RF-11.5, RNF-2, LC-3, LC-4, LC-11)* — after T-10
  **Done when:** unit tests show one reveal succeeds and the second discards, `acknowledge` from any state but `REVEALED` throws, a second `discard` is a no-op returning the first reason, and `recordedOutcome` returns exactly what was recorded for the key.

- [ ] **T-12 · `StepUpChallenge`** — identity id, `purpose` (`REGENERATE`, `REPLACE`), `expiresAt`, `attemptsRemaining`, states `OPEN → SUCCEEDED(consumedByPendingId) | VOIDED`, idempotency-keyed outcomes; `recordFailure(now)` voids at zero attempts; `succeed(pendingId, now)` once only; any call after expiry voids. *(plan §2.1 `StepUpChallenge`, D5; RF-8.3, RF-8.4, RF-9.2, RNF-5, LC-13, LC-15)* — after T-01
  **Done when:** unit tests show failures decrement and void at zero, expiry voids, `succeed` records the pending id, and a second `succeed` or a `succeed` after `VOIDED` throws.

- [ ] **T-13 · `EnrollmentGate` (`access.domain`)** — pure rule `(tenantRole, tenantPolicy.requireMfaForMembers, hasActiveFactor) → NOT_REQUIRED | REQUIRED(OWNER_ADMIN | MEMBER_POLICY)`: Owner / Admin without a factor is always `REQUIRED(OWNER_ADMIN)`; Member without a factor is `REQUIRED(MEMBER_POLICY)` only when the policy says so; any role with a factor is `NOT_REQUIRED`. *(plan §2.1 `EnrollmentGate`, D7; RF-1.3, RF-1.4, RF-1.5, RF-1.6, RF-10.2, LC-20, LC-21, LC-23)*
  **Done when:** `enrollmentGate.memberPolicyRequiresAtNextDerivation` and `enrollmentGate.policyNeverRelaxesOwnerAdmin` pass over the full truth table (three roles × policy on/off × factor present/absent).

- [ ] **T-14 · Code normalisation and recovery-code format** — one pure function stripping separators and surrounding whitespace from an authenticator code before verification (reuse the plan 001 normaliser **[001]** if it exists); one pure formatter producing a recovery code of `entropyBits` from random bytes in the grouped display form of R36, with the reverse normalisation used by `consume`. *(plan §2.1 `VerifyEnrollmentCodeUseCase`, `OpenRevealUseCase`; RF-3.1, RF-4.2)* — after T-01
  **Done when:** `verify.stripsSeparators` passes for spaces, hyphens and surrounding whitespace, and a generated code carries exactly `entropyBits` of input and round-trips through the normaliser.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-30…T-37); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-15 · Port interfaces** — new: `PendingEnrollmentRepository` (`findForUpdate`, `findOpenByIdentity`, `save`), `StepUpRepository`, `CurrentSessionPort` (`currentSessionId`, `exists`), `RequirementReasonPort` (published by `access`), `QrEncoder` (`svg(content)`), `FactorStatusPort` (published: `hasActiveFactor`, `enrolledAt`); extend `RevokeSessionsPort` **[004]** with `revokeAll(identityId, exceptSessionId)`; reuse `IdentityRepository`, `TotpVerifier`, `SeedCipher`, `PasswordVerifier`, `ThrottleStore`, `AuditPort`, `Clock` **[001]**, `TokenGenerator`, `JobScheduler`, `TransactionalEmailPort`, `SecurityNotificationPort` **[002]**. No port reaches a project, grant, approval, run or intent table. *(plan §2.1 Ports; RF-1.2, RF-6.4, RF-7.2, RF-9.7, RNF-13)* — after T-07…T-14
  **Done when:** the application module compiles with no dependency on Spring or persistence classes, enforced by an ArchUnit module-boundary test that also forbids any import from the run, approval or project packages.

- [ ] **T-16 · `StartEnrollmentUseCase`** — preconditions: identity `ACTIVE` and verified (`UNVERIFIED` → `auth.verification-pending` **[001]**, `RESTRICTED` / `ERASING` → refusal, A-27); `INITIAL` refused when a factor exists; discard any open pending enrollment as `SUPERSEDED` (audit `ENROLLMENT_ABANDONED{SUPERSEDED}`); generate the seed, encrypt under `SeedCipher`, create the `PendingEnrollment` bound to `CurrentSessionPort.currentSessionId()`, audit `ENROLLMENT_STARTED{purpose}`, return the setup material once (`uri`, `key`, `svg` from `QrEncoder`); `REPLACEMENT` callable only from `StepUpUseCase`. *(plan §2.1 `StartEnrollmentUseCase`, D1, D2, D9, D15; RF-1.6, RF-1.7, RF-1.8, RF-2.1, RF-2.2, RF-2.6, RF-6.1, RF-9.3, RF-11.3, LC-8, LC-24, LC-25)* — after T-15
  **Done when:** `start.refusesWhenFactorExists`, `start.unverifiedRoutesToVerification` and `start.restrictedOrErasingRefused` pass, and a second start for the same identity leaves exactly one open row and one `SUPERSEDED` row.

- [ ] **T-17 · `VerifyEnrollmentCodeUseCase`** — lock the row; require the owning session and `SET_UP`; return the recorded outcome for a repeated idempotency key; normalise the code (T-14); verify with `TotpVerifier` against the pending seed, the global tolerance and the pending watermark; delegate to `PendingEnrollment.verify`; success → `{state: VERIFIED}` and audit `ENROLLMENT_VERIFIED`; failure → `mfa.code.rejected`; discard (`ATTEMPTS` or `EXPIRED`, audit `ENROLLMENT_EXPIRED{reason}`) → `mfa.enrollment.restart`; never writes a factor; the response carries no material. *(plan §2.1 `VerifyEnrollmentCodeUseCase`, D1, D8, D13; RF-3.1, RF-3.2, RF-3.4, RF-3.5, RF-3.6, RF-3.7, RF-3.8, RF-6.2, RF-11.5, LC-2, LC-5, LC-6, LC-7, LC-8)* — after T-16
  **Done when:** `verify.validCodeAdvancesWithoutActivating`, `verify.boundToExpiryAndAttempts`, `verify.expiredRefusesAndRestarts`, `verify.lastWrongAttemptDiscardsAndRestarts` and `verify.resubmitReturnsRecordedOutcome` pass, and a code submitted from a different session is refused.

- [ ] **T-18 · `OpenRevealUseCase`** — lock the row; require the owning session and `VERIFIED`; generate `codeCount` codes of `entropyBits` (T-14) with `TokenGenerator`; store their maintained-encoder hashes on the pending enrollment; `REVEALED`; audit `REVEAL_OPENED`; return the plain codes, purpose and issue time once. A call on a `REVEALED` row discards (`SECOND_REVEAL`, audit `ENROLLMENT_ABANDONED{SECOND_REVEAL}`) and answers `mfa.enrollment.restart`. *(plan §2.1 `OpenRevealUseCase`, D3; RF-4.1, RF-4.2, RF-4.6, RF-4.8, RF-8.5, RNF-2, LC-3, LC-4)* — after T-17
  **Done when:** `reveal.returnsCodesExactlyOnce`, `reveal.secondOpenDiscards` and `pendingCodes.storedHashedAndDeletedOnDiscard` pass, and the stored hashes verify against the returned plain codes through the encoder.

- [ ] **T-19 · `AcknowledgeRevealUseCase` — shared rules and the `INITIAL` branch** — lock the row; require the owning session, `REVEALED` and `acknowledged == true` (else the field problem); return the recorded outcome for a repeated key; `INITIAL`: in one transaction create the `TotpFactor` from the pending seed with `enrolledAt`, create the `RecoveryCodeSet` from the pending hashes, `acknowledge`, delete the pending hashes, audit `REVEAL_ACKNOWLEDGED` and `ENROLLMENT_COMPLETED{INITIAL}`, enqueue `enrollment-completed` through `SecurityNotificationPort`, record the outcome; respond `{state: ACTIVE, enrolledAt, unusedCount}`. Creates no session, enters no tenant, touches no run table. *(plan §2.1 `AcknowledgeRevealUseCase`, D4, D11, D13; RF-4.4, RF-4.5, RF-5.1, RF-5.5, RF-5.6, RF-11.5, RNF-3, RNF-13, LC-10, LC-11)* — after T-18
  **Done when:** `acknowledge.initialActivatesFactorAndSetAtomically`, `acknowledge.refusesUnticked` (use-case level) and `acknowledge.touchesNoSessionTenantOrRunTable` pass, and a forced failure after the factor insert rolls back the set, the state and the outbox row together.

- [ ] **T-20 · `AcknowledgeRevealUseCase` — `REGENERATE` branch** — same guards; in one transaction `invalidateAll` on the current set, create the new set from the pending hashes, `acknowledge`, delete the pending hashes, audit `ENROLLMENT_COMPLETED{REGENERATE}`, enqueue `recovery-codes-regenerated`; respond `{state: REGENERATED, enrolledAt, unusedCount}`; the factor row, the session service and any requirement are untouched. *(plan §2.1, D4, D11; RF-8.6, RF-8.9, RNF-3)* — after T-19
  **Done when:** the acknowledge half of `regenerate.stepUpOpensRevealAndAcknowledgeSwapsSets` passes, every code of the old set refuses `consume` afterwards, and recording stubs for `RevokeSessionsPort` and the factor repository show no call.

- [ ] **T-21 · `AcknowledgeRevealUseCase` — `REPLACEMENT` branch** — same guards; in one transaction `TotpFactor.replaceWith`, `invalidateAll` on the current set, new set, `bumpSecurityVersion` **[004]**, `acknowledge`, delete the pending hashes, `RevokeSessionsPort.revokeAll(identityId, except = owningSessionId)`, audit `ENROLLMENT_COMPLETED{REPLACEMENT}` and `SESSIONS_REVOKED{cause=REPLACEMENT}`, enqueue `authenticator-replaced`; respond `{state: REPLACED, enrolledAt, unusedCount}`. *(plan §2.1, D4, D11; RF-9.6, RF-9.7, RF-9.8, RNF-3, RNF-4, LC-18, LC-19)* — after T-20
  **Done when:** `replace.acknowledgeSwapsFactorAndSetsAtomically` and `replace.revokesOthersKeepsCurrentLeavesRunAuthority` pass with a recording session double (the owning session id is the `except` argument; no run-scoped call is made).

- [ ] **T-22 · `DiscardPendingEnrollmentUseCase`** — `discard(LEFT)` for the caller's open pending enrollment; a missing, foreign-session or already-discarded row is still a success; seed and pending hashes destroyed; active factor and set untouched; audit `ENROLLMENT_ABANDONED{LEFT}` only when a row was discarded. *(plan §2.1 `DiscardPendingEnrollmentUseCase`, D9; RF-2.5, RF-4.6, RF-6.3, RF-8.7, RF-9.5, LC-1, LC-9, LC-13, LC-16)* — after T-19
  **Done when:** `discard.destroysSeedLeavesFactor`, `regenerate.abandonedRevealLeavesPreviousSetValid` and `replace.abandonedLeavesOldFactorNoRevocationNoMail` pass, and calling discard twice returns success both times with one audit entry.

- [ ] **T-23 · `StepUpUseCase` — challenge, verification and failure path** — create or continue the caller's `OPEN` challenge for the purpose; lock; check expiry and attempts; return the recorded outcome for a repeated key; verify the password with `PasswordVerifier` and the code with `TotpVerifier` against the **active** factor (watermark advanced on success); the code field is never offered to `RecoveryCodeSet.consume`; any failure → one `mfa.step-up.rejected`, counted on the challenge and on the sign-in throttle keys `ACCOUNT` and `NETWORK` **[001]**, audit `STEP_UP_FAILED{purpose}`; zero attempts or expiry → `VOIDED`, `mfa.step-up.void`, audit `STEP_UP_VOIDED`. *(plan §2.1 `StepUpUseCase`, D5, D6, D8; RF-8.2, RF-8.3, RF-8.4, RF-9.2, RNF-5, LC-14, LC-15)* — after T-15
  **Done when:** `stepUp.recoveryCodeRefusedAsWrongCode`, `stepUp.failureGenericForPasswordCodeOrBoth`, `stepUp.countsOnBothThrottleKeys` and `stepUp.limitOrExpiryVoidsNothingChanged` pass, and the active factor's watermark advances on a successful code.

- [ ] **T-24 · `StepUpUseCase` — success branches and consumption** — in the challenge's own transaction: `REGENERATE` → `PendingEnrollment.forRegeneration` in `VERIFIED` bound to the session; `REPLACE` → `StartEnrollmentUseCase(REPLACEMENT)` and its setup material in the response; the challenge records `consumedByPendingId` and `SUCCEEDED`; audit `STEP_UP_SUCCEEDED{purpose}`; record the outcome; respond `{pendingId, purpose, setup?}`. *(plan §2.1, D5; RF-8.5, RF-9.3, RF-11.5, RNF-5, LC-11)* — after T-23, T-16
  **Done when:** the step-up half of `regenerate.stepUpOpensRevealAndAcknowledgeSwapsSets`, `replace.stepUpStartsReplacementEnrollment`, `stepUp.cannotBeReusedForAnotherPurpose` and `stepUp.resubmitReturnsRecordedOutcome` pass.

- [ ] **T-25 · `SecurityMfaQuery`** — for the caller only: factor state and `enrolledAt`, display name and verified email, `unusedCount` (absent when no factor), requirement `{reason ∈ OWNER_ADMIN | MEMBER_POLICY | NONE, tenants[]}` from `RequirementReasonPort`; never the seed, a code, setup material or consumed codes by value. *(plan §2.1 `SecurityMfaQuery`, D14; RF-7.2, RF-7.3, RF-7.4, RF-7.6, RF-7.7, RF-7.8, LC-12, LC-30)* — after T-15
  **Done when:** `securityMfa.noFactorShape` and `securityMfa.unusedCountAfterLastCodeConsumed` pass, and the read model type has no field able to carry a code, seed or key.

- [ ] **T-26 · `FactorStatusPort` implementation** — `hasActiveFactor` and `enrolledAt` read `TotpFactor` rows only; pending enrollments of any state are invisible; plan 001's challenge keeps reading `TotpFactor` only, so a pending replacement is invisible to sign-in. *(plan §2.1 `FactorStatusPort`, D7; RF-1.6, RF-6.4, RF-9.4, LC-17)* — after T-15
  **Done when:** `factorStatus.ignoresPending` and `signIn.pendingEnrollmentIsNotChallenged` pass (the latter with plan 001's challenge stub), and `signIn.duringPendingReplacementUsesOldFactor` passes at the port level.

- [ ] **T-27 · `EnrollmentGate` wired into `access` C10 derivation** — on every tenant request the derivation calls `FactorStatusPort.hasActiveFactor` and reads the tenant policy at derivation time, evaluates `EnrollmentGate`; `REQUIRED(reason)` yields no privileged or customer authority and the request is answered with `auth.enrollment.required {reason}`; audit `ENROLLMENT_GATE_APPLIED{tenant, reason}`; `RequirementReasonPort` implemented over the same rule across the caller's memberships. A session, challenge or membership row is never consulted as a substitute. *(plan §2.1 `EnrollmentGate`, `RequirementReasonPort`, D7; RF-1.3, RF-1.4, RF-1.5, RF-1.6, RF-5.3, RF-10.2, RF-11.1, LC-20, LC-21, LC-23)* — after T-13, T-26
  **Done when:** `gate.unenrolledOwnerAdminDerivesNoAuthority`, `gate.sessionOrChallengeNeverSubstitutes`, `gate.singleFactorSatisfiesEveryTenant` and `gate.liftedOnNextDerivationWithoutNewSignIn` pass, and tightening the policy in the test makes the next derivation for an unenrolled Member answer `MEMBER_POLICY` without re-login.

- [ ] **T-28 · Workspace-creation precondition stub (spec 012)** — a precondition in the workspace-creation entry (stubbed command for this feature) that calls `FactorStatusPort.hasActiveFactor` before accepting any field and answers `auth.enrollment.required {OWNER_ADMIN}` when false; nothing is created. *(plan §2.1 `FactorStatusPort` consumers, D7; RF-1.2, LC-22)* — after T-26
  **Done when:** `workspaceCreation.blockedWithoutFactor` passes and no workspace, tenant or membership row exists after the refused call.

- [ ] **T-29 · `ExpirePendingEnrollmentsJob`** — durable job (`sharedsupport/jobs` **[002]**) claiming, in batches of the configured size, pending enrollments and step-up challenges past `expiresAt` or whose owning session no longer exists (`CurrentSessionPort.exists`); discards them (`EXPIRED` / `SESSION_ENDED`, `VOIDED`); audits `ENROLLMENT_EXPIRED{reason}`; live rows untouched. *(plan §2.1 `ExpirePendingEnrollmentsJob`, D10; RF-6.2, RF-6.3, RF-11.1, LC-5)* — after T-22, T-23
  **Done when:** a job test with a controllable clock and a session double removes exactly the expired and session-ended rows, destroys their seeds and hashes, writes one audit entry per row with the right reason, and leaves a live row with a live session unchanged.

## D · Infrastructure adapters

- [ ] **T-30 · Persistence adapters** — implementations (access API per A-35) of `PendingEnrollmentRepository` (`findForUpdate` as `SELECT … FOR UPDATE`), `StepUpRepository`, and the extended factor and recovery-code repositories **[001]** (`enrolledAt`, `seedVersion`, `setId`, `consumedAt`, `unusedCount` as a count query). *(plan §2.1 Persistence, D1, D4; RF-6.1, RF-11.5, RNF-3, RNF-7, LC-8, LC-11)* — after T-02…T-04, T-15
  **Done when:** repository tests against real PostgreSQL pass for every method, `start.supersedesPendingAndEarlierTabIsRefused` passes end to end, and two concurrent `findForUpdate` calls on one row are serialised (the second observes the first's committed state).

- [ ] **T-31 · QR adapter** — `QrEncoder.svg(content)` over ZXing `core` 3.5.3 (A-46; entered in the dependency inventory A-18 with version and license): error-correction level M, four-module quiet zone, the module matrix written as one SVG path, no `javase` module or AWT; unit test: the SVG module grid equals the encoder's matrix and the matrix decodes back to the provisioning content with the same library's reader; returning an inline SVG string with no external reference, script or raster. *(plan §2.1 QR adapter, D15; RF-2.2, RF-12.4)* — after T-15
  **Done when:** for a fixed provisioning URI the output equals a stored golden SVG, and a guard test finds no `href`, `url(`, `<script` or `<image` in the output of 100 random inputs.

- [ ] **T-32 · Session adapters** — `CurrentSessionPort` (caller's session id from the request's session, `exists` against the session service F-IDN-08) and `RevokeSessionsPort.revokeAll(identityId, exceptSessionId)` **[004]** revoking every other session and refresh family; no change to JWT / refresh mechanics; run-scoped authority never addressed. *(plan §2.1 Session adapters, D9, D11; RF-6.3, RF-9.7, RF-9.8, LC-17, LC-18)* — after T-15
  **Done when:** against the session service's test double, `revokeAll` with `except` leaves exactly that session and its refresh family valid, `exists` answers false for a revoked session, and no call touches any run-scoped authority.

- [ ] **T-33 · Email and outbox wiring** — the three templates of T-05 enqueued through `SecurityNotificationPort` **[002]** behind the outbox relay, addressed to the verified email, in the request's UI locale; relay failures retried by the relay and recorded, never surfaced to the request. *(plan §2.1 Outbox relay and email adapter, D11; RF-5.5, RF-8.9, RF-9.7, RF-11.4, RNF-9, LC-29)* — after T-05, T-15
  **Done when:** `notifications.threeTemplatesToVerifiedAddressWithoutSecrets` and `notification.failureNeverRevertsChange` pass with a fake provider that fails on first delivery.

- [ ] **T-34 · Audit adapter** — map the thirteen events of plan §2.1 (`ENROLLMENT_STARTED{purpose}`, `ENROLLMENT_VERIFIED`, `ENROLLMENT_COMPLETED{purpose}`, `ENROLLMENT_ABANDONED{reason}` for `SUPERSEDED`, `LEFT` and `SECOND_REVEAL`, `ENROLLMENT_EXPIRED{reason}` for `EXPIRED`, `ATTEMPTS` and `SESSION_ENDED`, `REVEAL_OPENED`, `REVEAL_ACKNOWLEDGED`, `STEP_UP_SUCCEEDED{purpose}`, `STEP_UP_FAILED{purpose}`, `STEP_UP_VOIDED`, `SESSIONS_REVOKED{cause}`, `NOTIFICATION_ENQUEUED{template}`, `ENROLLMENT_GATE_APPLIED{tenant, reason}`) to `AuditPort` with identity id, correlation id, network key and timestamp, appended in the effect's transaction; a serialisation guard rejects any payload holding a seed, key, code, recovery code, count by value or password. *(plan §2.1 Audit adapter; RF-11.1, RF-11.2, RNF-1)* — after T-15
  **Done when:** `audit.entryPerEvent` and `audit.containsNoSecret` pass.

- [ ] **T-35 · Logging and problem redaction** — request and response logging exclude `code`, `password`, `codes`, `setup`, `key`, `uri` and `svg`; problem details never echo a submitted value; a log-appender test asserts absence. *(plan §2.1, D19; RF-11.2, RNF-1)* — after T-34
  **Done when:** `logs.containNoSecret` passes with secrets present in every request and response of the seven endpoints.

- [ ] **T-36 · Job registration** — register `ExpirePendingEnrollmentsJob` with `sharedsupport/jobs` at the configured cadence and batch size of T-01. *(plan §2.1, D10; RF-6.2)* — after T-29
  **Done when:** the job is listed by the job registry at startup and runs once in an integration test when triggered.

- [ ] **T-37 · Crypto wiring** — the pending seed generated by `TokenGenerator` and encrypted under `SeedCipher` **[001]** before the first write; recovery-code hashes through the maintained `PasswordEncoder`; `pending_enrollment` and `pending_recovery_code` hold ciphertext and hashes only. *(plan §2.1, D2, D3; RF-4.8, RF-11.3, RNF-1)* — after T-30
  **Done when:** a database inspection test after start and reveal finds no plain seed, key or code in any column, and the stored hashes verify against the plain codes returned by the reveal.

## E · API

- [ ] **T-38 · `POST /api/account/mfa/enrollment`** — identity session required, no C10; 200 `{pendingId, purpose, setup: {uri, key, svg}, expiresAt}`; 409 `mfa.enrollment.already-enrolled`; 409 `auth.verification-pending` **[001]**; refusal for non-`ACTIVE`; the UI locale in a request header; never redirects. *(plan §2.1 `MfaController`, D2, D19; RF-1.6, RF-1.7, RF-1.8, RF-2.1, RF-2.2, RF-12.4)* — after T-16, T-31, T-37
  **Done when:** `start.returnsUriKeyAndSvgOnce` passes and the three refusal cases return their problem code with no setup material.

- [ ] **T-39 · `POST /api/account/mfa/enrollment/{id}/verify`** — `{code, idempotencyKey}` → 200 `{state: VERIFIED}`; 400 `mfa.code.rejected` with no cause and no count; 410 `mfa.enrollment.restart`; the response carries no material. *(plan §2.1 `MfaController`, D13, D19; RF-3.3, RF-3.4, RF-3.6, RF-3.7, RF-3.8)* — after T-17
  **Done when:** `codeRejected.carriesNoCauseOrCount` passes and the 200 and 410 bodies contain no `uri`, `key`, `svg` or `codes` member.

- [ ] **T-40 · `POST …/enrollment/{id}/reveal` and `POST …/enrollment/{id}/acknowledge`** — reveal → 200 `{codes[], purpose, issuedAt}` once or 410; acknowledge `{acknowledged: true, idempotencyKey}` → 200 `{state: ACTIVE | REGENERATED | REPLACED, enrolledAt, unusedCount}`, 400 field problem at `acknowledged` when not true, 410. No endpoint of the controller returns setup material or codes after these two responses. *(plan §2.1 `MfaController`, D3, D4, D13; RF-2.6, RF-3.8, RF-4.1, RF-4.4, RF-4.5, RF-11.3, RF-11.5)* — after T-18…T-21
  **Done when:** `acknowledge.refusesUnticked` (API level) and `noEndpointReturnsSetupAfterStart` pass, and a second reveal call returns 410 while the first returned the codes.

- [ ] **T-41 · `DELETE …/enrollment/{id}` and `POST /api/account/mfa/step-up`** — delete → 204 for an open, missing or already discarded row; step-up `{purpose, password, code, idempotencyKey}` → 200 `{pendingId, purpose, setup?}` (`setup` present only for `REPLACE`), 400 `mfa.step-up.rejected`, 410 `mfa.step-up.void`. *(plan §2.1 `MfaController`, D5, D9; RF-2.5, RF-8.3, RF-8.4, RF-8.5, RF-9.2, RF-9.3)* — after T-22, T-24
  **Done when:** contract tests show 204 in the three delete cases, the `REGENERATE` response has no `setup` member, the `REPLACE` response has one, and the 400 body is byte-identical for a wrong password, a wrong code and both.

- [ ] **T-42 · `GET /api/account/security/mfa`** — 200 `{factor: {state, enrolledAt?}, identity: {displayName, email}, recoveryCodes: {unusedCount?}, requirement: {reason, tenants[]}}` for the caller; the path is the server counterpart of the FE route `/account/security`; no identity parameter exists on this or any endpoint of the controller. *(plan §2.1 `MfaController`, D14; RF-7.2, RF-7.3, RF-7.6, RF-7.7, RF-7.8)* — after T-25
  **Done when:** `securityMfa.exposesMetadataOnly` and `securityMfa.noIdentityParameter` pass.

- [ ] **T-43 · Gate problem on tenant requests** — the `access` derivation outcome `REQUIRED(reason)` is rendered by the shared problem handler as 403 `auth.enrollment.required {reason}` on every tenant endpoint and on the workspace-creation entry; the FE heading is derived from `reason` only, never from a route value. *(plan §2.1 `MfaController` (tenant endpoints), D7, D16; RF-1.3, RF-1.4, RF-1.5, RF-1.9, LC-20, LC-23)* — after T-27, T-28
  **Done when:** a contract test on a stub tenant endpoint returns 403 with `reason = OWNER_ADMIN` for an unenrolled Admin and `MEMBER_POLICY` for a policy-bound Member, with no tenant data and no authority-bearing member in the body.

- [ ] **T-44 · Absence checks** — a test enumerating the API surface: no endpoint reads, resets, replaces, regenerates or disables another identity's factor or set; no tenant setting or administrator endpoint touches a factor; no device-trust or remembered-browser surface exists; no response after the start and reveal responses carries a seed, key, setup, code or count by value. *(plan §3 RF-10, RF-11; RF-10.1, RF-10.4, RF-11.3, RNF-8)* — after T-38…T-43
  **Done when:** `noAdminOrTenantPathTouchesAnotherFactor`, `auth.noDeviceTrustSurfaceExists` **[001]** and `responses.neverContainSeed` pass.

## F · Front end

- [ ] **T-45 · `contracts/mfa` types and guards** — request / response / problem types with runtime guards for the seven endpoints and for `auth.enrollment.required {reason}`; a guard rejects any response other than the start and reveal responses that carries `setup`, `key`, `svg`, `codes` or `seed`; the `reason` union is closed. *(plan §2.2 `contracts/mfa`; RF-1.9, RF-11.2, RNF-1)*
  **Done when:** guard unit tests accept one fixture per documented response and problem, and reject a verify response carrying `setup` and an acknowledge response carrying `codes`.

- [ ] **T-46 · i18n bundles `identity-access.{en,es,fr}`: enrollment keys** — every heading, subtitle (Owner / Admin, Member policy, voluntary, replacement — S-33), step name, statement, label, action, warning and message of R38 (TOTP & recovery area), R39, R41, R35 and R36; the D02 text "We couldn't verify the code" and "Setup must be restarted" (S-32, D02 `422:3452`); the zero-codes warning (S-30); the not-enrolled card texts (S-29); the download file heading (S-31). *(plan §2.2, D18, D19; RF-1.9, RF-3.6, RF-3.7, RF-12.5, RNF-11)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value, and the EN values for RF-1.9, RF-3.6 and RF-3.7 match the specification verbatim.

- [ ] **T-47 · Hooks** — `useStartEnrollment`, `useVerifyCode`, `useOpenReveal`, `useAcknowledge`, `useDiscard`, `useStepUp` (TanStack mutations, retry off, UI locale header; verify, acknowledge and step-up generate one idempotency key per submission and reuse it on re-submission) and `useSecurityMfa` (query, invalidated after every completion). *(plan §2.2 State, D13; RF-11.5, RF-12.5, LC-11)* — after T-45
  **Done when:** unit tests show each mutation sends the current locale, has retry disabled, and a re-submitted verify, acknowledge or step-up sends the same key as its first attempt.

- [ ] **T-48 · Enrollment, setup and reveal holders** — module-private in-memory records: enrollment `{pendingId, purpose, reason, returnTo}`, setup material (alive only while `SetupStep` is mounted), reveal codes (alive only while `RecoveryCodesReveal` is mounted); cleared on acknowledge, discard, restart, unmount and route leave; `DELETE mfa/enrollment/{id}` fired on route leave and, as a best-effort `keepalive` request, on `pagehide`; no read or write to `localStorage`, `sessionStorage`, cookies or IndexedDB; a reload therefore starts over. *(plan §2.2 Enrollment / Reveal / Setup holders, D9, D12, D16; RF-2.5, RF-2.6, RF-4.6, RF-4.7, RF-6.3, RNF-1, RNF-2, LC-1, LC-2, LC-3, LC-4)* — after T-47
  **Done when:** `storage.containsNothingFromFeature` passes with storage spies on every mechanism, the setup and reveal holders are empty after their component unmounts, leaving the wizard route calls `useDiscard` once, and a simulated `pagehide` issues exactly one `keepalive` DELETE for the open pending id.

- [ ] **T-49 · `EnrollmentGateHandler` (`shared/errors`)** — maps a 403 `auth.enrollment.required {reason}` from any tenant request, and from the workspace-creation entry, to the wizard route with `{reason, returnTo}` in the holder (`returnTo` = the interrupted tenant route or the creation entry); after activation navigates to `returnTo`, which re-requests and now derives authority; nothing of the tenant is fetched before that. *(plan §2.2 Routing, D16; RF-1.2, RF-1.3, RF-1.4, RF-5.3, LC-20, LC-22, LC-23)* — after T-48
  **Done when:** unit tests map each reason to the wizard with the matching holder, show `returnTo` taken from the current location and never from the problem body, and a mocked activation navigates to `returnTo`.

- [ ] **T-50 · `StepIndicator` and `SecurityStatements` extension [001]** — `StepIndicator` "1 · Set up · 2 · Verify code · 3 · Save recovery codes" with the current step named in text and `aria-current`; `SecurityStatements` extended with every statement of R38, R39, R41, R35 and R36 as shared blocks used by desktop and reflow alike. *(plan §2.2 `SecurityStatements`, `shared/ui`, D18; RF-2.3, RF-2.4, RF-7.2, RF-8.1, RF-9.1, RF-12.1, RF-12.3)* — after T-46
  **Done when:** `stepIndicator.currentStepInText` and `securityStatements.presentOnWizardDialogsAndArea` pass.

- [ ] **T-51 · `EnrollmentWizard` shell** — one component for the three purposes and four entries: heading and subtitle from the holder (`OWNER_ADMIN` → "Protect your Owner identity." + its subtitle; otherwise "Protect your identity." + the reason subtitle), `StepIndicator`, the statements of RF-2.4, "Leave setup" on every step calling `useDiscard` then navigating to the tenant chooser, the workspace-creation entry or `/account/security` per the entry (Q20); a restart resets to step 1 with a fresh start call. *(plan §2.2 `EnrollmentWizard`, D16; RF-1.9, RF-2.3, RF-2.4, RF-2.5, LC-1)* — after T-49, T-50
  **Done when:** `wizard.headingPerEntry` (owner, member policy, voluntary, replacement) and `wizard.leaveSetupDiscardsAndReturnsPerEntry` pass.

- [ ] **T-52 · `SetupStep`** — renders the inline SVG from the start response with a text alternative, the setup key as copyable text with a "Copy" control, "Continue" to step 2; the material lives only in the setup holder and is dropped on unmount; nothing is written anywhere. *(plan §2.2 `SetupStep`, D2, D15; RF-2.2, RF-2.6, RF-12.4)* — after T-51
  **Done when:** a unit test finds the SVG, the full key text and an accessible name on the graphic, the copy control places the key on the clipboard mock, and the holder is empty after unmount.

- [ ] **T-53 · `VerifyStep`** — `CodeField` **[001]** labelled "Code from your authenticator" accepting paste with separators and whitespace, "Verify code & continue", in-progress state with resubmission disabled, `GenericFailureMessage` **[001]** "We couldn't verify the code" on `mfa.code.rejected` staying on step 2, "Setup must be restarted" on `mfa.enrollment.restart` then a fresh start at step 1; no material rendered. *(plan §2.2 `VerifyStep`, D19; RF-3.1, RF-3.3, RF-3.6, RF-3.7, RF-3.8, LC-2, LC-6)* — after T-52
  **Done when:** `codeField.pasteWithSeparators`, `verifyStep.inProgressDisablesResubmit` and `wizard.restartShowsNewMaterial` pass, and the rendered step contains no `uri`, `key` or `svg` text.

- [ ] **T-54 · `RecoveryCodesReveal` (R36)** — every code in full, the statements (one use each; shown this one time; for regeneration and replacement, the previous codes stop being valid when this reveal is finished), heading per purpose ("Save your replacement codes." for regeneration), the acknowledgement "I have saved the new codes in a safe place" rendered unticked on every mount, finish action disabled until ticked; finish calls `useAcknowledge` once. *(plan §2.2 `RecoveryCodesReveal`, D3, D4; RF-4.1, RF-4.4, RF-4.5, RF-8.5, RNF-2, LC-10)* — after T-48, T-50
  **Done when:** `reveal.finishDisabledUntilTicked` and `reveal.finishWithoutDownloadCompletes` pass, and the checkbox is unticked after a remount even when it was ticked before.

- [ ] **T-55 · Download and copy** — "Download securely issued codes" builds a plain-text file on the client from the reveal response: one heading line naming the product and the issue date, then one code per line, nothing else (Q17); offered during the reveal only with a transient object reference revoked after the save; "Copy" places the codes on the clipboard; neither action ticks the acknowledgement or calls any endpoint. *(plan §2.2 `RecoveryCodesReveal`, D12; RF-4.3, RF-4.5, RF-4.7, LC-9)* — after T-54
  **Done when:** `download.contentPerQ17` and `download.doesNotAcknowledge` pass, the file contains no identity, tenant or seed data, and the object URL is revoked after the save.

- [ ] **T-56 · `RevealStep` wiring and completion navigation** — step 3 of the wizard mounts `RecoveryCodesReveal`; on a 200 from acknowledge the holders are cleared, `useSecurityMfa` is invalidated, and navigation goes to `returnTo` when present (forced entry) or to `/account/security` (voluntary or replacement), showing the enrolled state; no tenant is entered and no session is touched on the way. *(plan §2.2 Handoffs, D16; RF-5.3, RF-5.4, RF-5.6)* — after T-55, T-53
  **Done when:** unit tests show navigation to `returnTo` when set and to `/account/security` otherwise, with the query invalidated and both holders empty afterwards.

- [ ] **T-57 · `SecuritySettingsPage` and `MfaCard`** — heading "Your identity, protected.", subtitle "Global sign-in security. Tenant membership policies may require stronger verification.", the TOTP & recovery area beside the Sessions (slot, 007), Password (statement, 004) and Privacy slots; `MfaCard` enrolled state (identity, requirement line from `requirement.reason`, enrollment time, "Enrollment metadata only; seed never exposed", "Replace authenticator") and not-enrolled state in the same position ("No authenticator enrolled", requirement line, "Set up authenticator" → wizard with the voluntary entry; S-29). *(plan §2.2 `SecuritySettingsPage`, `MfaCard`, D14; RF-1.1, RF-7.1, RF-7.2, RF-7.3, RF-7.4, LC-30)* — after T-47, T-50
  **Done when:** `securitySettingsPage.rendersAreasAndEnrolledCards` and `mfaCard.notEnrolledOffersSetUpOnly` pass.

- [ ] **T-58 · `RecoveryCodesCard`** — statements (store safely; regeneration requires policy-approved re-authentication and invalidates the old set), unused count as text, inline zero-codes warning above "Regenerate recovery codes" (S-30) that blocks nothing, the "Controlled account recovery" route (spec 005); no action rendered when no factor; nothing by value beyond the count. *(plan §2.2 `RecoveryCodesCard`, D14; RF-7.4, RF-7.5, RF-7.6, RF-7.7, LC-12)* — after T-57
  **Done when:** `recoveryCodesCard.zeroWarningNeverBlocks` passes, the card at count zero still offers "Regenerate recovery codes", and the no-factor render contains no button.

- [ ] **T-59 · `RegenerateCodesDialog` (R35)** — "Replace your recovery codes?", the re-authentication statement, the warning "Existing codes will be invalidated" with its explanation, `PasswordField` with Show, `CodeField`, "Keep existing codes" (closes, calls nothing), "Verify & regenerate codes" (`useStepUp` with purpose `REGENERATE`), the statement "This does not disable TOTP or bypass required Owner / Admin MFA."; `mfa.step-up.rejected` → `GenericFailureMessage`; `mfa.step-up.void` → dialog closed, nothing changed; success → `useOpenReveal` and `RecoveryCodesReveal` with the replacement heading. *(plan §2.2 `RegenerateCodesDialog`, D5, D19; RF-8.1, RF-8.3, RF-8.4, RF-8.5, RF-8.8)* — after T-58, T-54
  **Done when:** `dialog.voidCloses` and the regenerate half of `dialogs.keepActionsCallNothing` pass, and a mocked successful step-up opens the reveal with "Save your replacement codes.".

- [ ] **T-60 · `ReplaceAuthenticatorDialog` (R41)** — "Replace, don't remove protection.", subtitle "Verify the current factor before enrolling its replacement.", warning "Required MFA remains active" with its explanation, `PasswordField` with Show, `CodeField`, "Keep current factor" (closes, calls nothing), "Verify & begin replacement" (`useStepUp` with purpose `REPLACE`), the route "I lost access · controlled recovery" (closes, calls nothing, navigates to the spec 005 entry), the statement "Lost-factor recovery is separately verified. Email password reset alone never replaces or resets MFA."; success → wizard at step 1 with the replacement entry and the setup material from the step-up response. *(plan §2.2 `ReplaceAuthenticatorDialog`, D5, D16; RF-9.1, RF-9.3, RF-9.9, RF-9.10, LC-27)* — after T-59
  **Done when:** `replaceDialog.lostAccessRoutesWithoutChange` and the replace half of `dialogs.keepActionsCallNothing` pass, and a mocked successful step-up mounts the wizard with the replacement heading and the returned setup material.

- [ ] **T-61 · Routes and handoffs** — `/account/security` (R38) and `/account/security/enroll` (R39) registered in the feature's public API; no purpose, reason or destination in the route; `auth.verification-pending` on start → the verification pending state **[002]**; `mfa.step-up.void` → dialog closed; the Sessions slot stays empty for 007. *(plan §2.2 routes, Handoffs, D16; RF-1.1, RF-1.7, RF-5.4, RF-8.4, LC-24)* — after T-56, T-60
  **Done when:** a route test renders both pages, shows the wizard with no query or path parameter carrying `reason` or `returnTo`, and a mocked `auth.verification-pending` lands on the pending state with no pending enrollment created.

- [ ] **T-62 · Reflow and touch targets** — the M13 / M14 layouts for the wizard and the reveal (`431:4190`, `431:4222`), Tailwind breakpoints for the three dialogs and the settings area; `shared/ui` fields, checkbox, buttons and the copy control at 44 px minimum on touch widths; visible focus; every statement, field, action, route and gate present at every width. *(plan §2.2 Shared used, D18; RF-12.1, RF-12.2, RNF-10, RNF-12)* — after T-61
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow, identical statement text at the three widths, and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-63 · Concurrency suite** — `acknowledge.doubleSubmitActivatesOnce`, `acknowledge.noObserverSeesFactorWithoutSet`, `start.supersedesPendingAndEarlierTabIsRefused`, `reveal.reloadNeverReshows`, two concurrent step-ups for one identity, an acknowledge racing a plan 001 sign-in challenge. *(plan §5.1 Concurrency, D1, D4, D13; RF-2.1, RF-4.6, RF-5.1, RF-6.1, RF-11.5, RNF-2, RNF-3, LC-8, LC-11)* — after T-30, T-40, T-41
  **Done when:** all six tests pass ten times in a row with real PostgreSQL.

- [ ] **T-64 · Never-unprotected and sign-in interplay suite** — `replacement.identityNeverWithoutFactor` (polling observer during the swap), `signIn.duringPendingReplacementUsesOldFactor`, `signIn.oldAuthenticatorRefusedAfterReplacement`, `signIn.pendingEnrollmentIsNotChallenged`, `afterEnrollment.nextSignInChallenges` (BE half with the plan 001 challenge), `codes.acceptedOnceOnly`. *(plan §5.2 RNF-4, RNF-5 rows, D4, D8; RF-5.2, RF-6.4, RF-9.4, RF-9.6, RNF-4, RNF-5, LC-17, LC-19)* — after T-63
  **Done when:** all six tests pass with plan 001's challenge code or its stub.

- [ ] **T-65 · Fail-closed and boundary suite** — `failClosed.absentExpiredSupersededUnverifiedRefuses` (every use case against every bad pending state), `passwordReset.leavesFactorAndCodesUntouched` (plan 004 `ChangeCredential`), `feature.neverLiftsSuspensionMembershipOrErasure`, `recovery.entersReplacementEnrollmentDirectly` (spec 005 stub re-entering `StartEnrollmentUseCase(REPLACEMENT)`). *(plan §5.2 RNF-7, RF-10, RF-9.9 rows, D1, D10; RF-9.9, RF-10.3, RF-10.5, RNF-7, RNF-8, LC-26, LC-28)* — after T-44
  **Done when:** all four tests pass; the fail-closed test has one case per (use case × bad state) pair and none is skipped.

- [ ] **T-66 · Startup suite** — `enrollmentPolicy.startupFailsOnAbsentOrUnboundedValue` with one case per property of T-01, plus the case where `LoginPolicy` **[001]** lacks the tolerance. *(plan §5.1 Startup, D17; RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6)* — after T-01
  **Done when:** every case fails the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-67 · FE e2e journeys** — Playwright journeys for US-1…US-11 in EN, ES and FR at desktop, tablet and mobile widths with the BE test doubles: forced entry from workspace creation and from tenant entry (Owner, Admin, Member policy) → wizard → activation → `returnTo`; voluntary enrollment → settings enrolled; "Leave setup" from each entry; regeneration; replacement with revocation observed from a second browser context; zero-codes notice; `wizard.returnsToInterruptedDestination`, `wizard.voluntaryReturnsToSettingsEnrolled`, `wizard.leaveDuringCreationCreatesNothing`, `afterEnrollment.nextSignInChallenges`, `languageSwitch.changesDisplayOnly`. *(plan §5.1 FE e2e, §5.2 journeys; RF-5.2, RF-5.3, RF-5.4, RF-12.5, RNF-11, LC-22, US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8, US-9, US-10, US-11)* — after T-62, T-44
  **Done when:** all journeys pass in the three languages and three widths, and the language switch mid-wizard changes only displayed text.

- [ ] **T-68 · Automated accessibility** — axe on the wizard (three steps), the three dialogs, the reveal and the settings area at the three widths and three languages (`a11y.enrollmentDialogsSettings.{en,es,fr}.{desktop,tablet,mobile}`), plus `a11y.setupKeyUsableWithoutGraphic` (the key is reachable and readable with the SVG hidden). *(plan §5.1 FE e2e + accessibility; RF-12.1, RF-12.2, RF-12.3, RF-12.4, RNF-10, RNF-12)* — after T-67
  **Done when:** zero critical or serious violations on every combination and the setup-key test passes.

- [ ] **T-69 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on the wizard, the dialogs, the reveal and the settings area: step indicator announced, focus order, errors associated with their field, live announcements without focus theft from the code field or the acknowledgement, labels, 44 px targets, the copyable key; versions recorded in the release-support matrix. *(plan §5.1 Manual assistive; RF-12.2, RF-12.3, RNF-10)* — after T-68
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-70 · Adversarial suite (US-12)** — seed exposure through every endpoint and log; code replay inside the wizard and at sign-in after a step-up; recovery-code reuse; double reveal through a second tab and a replayed request; activation without verification (acknowledge on a `SET_UP` row, reveal on a `SET_UP` row); a step-up succeeded for `REGENERATE` presented for `REPLACE`; another identity's pending id or session; any tenant or administrator path against a factor. *(plan §5.2 US-12 row, D2, D3, D5, D6, D8; RF-3.5, RF-4.6, RF-10.1, RF-11.3, RNF-2, RNF-5, RNF-8, US-12)* — after T-63, T-65
  **Done when:** every attack test fails to achieve its goal and asserts the generic or fail-closed response.

## H · Closing

- [ ] **T-71 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-63…T-70
  **Done when:** the script reports 138 of 138 identifiers (83 RF, 13 RNF, 30 LC, 12 US) covered by at least one passing test.

- [ ] **T-72 · Register-dependency review** — walk plan §6 — A-27 and A-32, the two items spec §7 item 13 names, and the plan-level A-35 and A-39 (port contract), plus confirmation that A-46 (decided) is entered in A-18 — and the fourteen decided items it implements (A-37…A-43, S-28…S-34): for each, confirm the built assumption matches the current Figma register state or record the deferral. *(spec §7 item 13)* — after T-71
  **Done when:** each of the nineteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-73 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run id, a11y report, manual checklist, coverage report, register review). — after T-72
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-57, T-61 |
| RF-1.2 | T-15, T-28, T-49 |
| RF-1.3 | T-13, T-27, T-43, T-49 |
| RF-1.4 | T-13, T-27, T-43, T-49 |
| RF-1.5 | T-13, T-27, T-43 |
| RF-1.6 | T-13, T-16, T-26, T-27, T-38 |
| RF-1.7 | T-16, T-38, T-61 |
| RF-1.8 | T-16, T-38 |
| RF-1.9 | T-43, T-45, T-46, T-51 |
| RF-2.1 | T-09, T-16, T-38, T-63 |
| RF-2.2 | T-16, T-31, T-38, T-52 |
| RF-2.3 | T-50, T-51 |
| RF-2.4 | T-50, T-51 |
| RF-2.5 | T-22, T-41, T-48, T-51 |
| RF-2.6 | T-16, T-40, T-48, T-52 |
| RF-3.1 | T-14, T-17, T-53 |
| RF-3.2 | T-01, T-09, T-10, T-17, T-66 |
| RF-3.3 | T-39, T-53 |
| RF-3.4 | T-10, T-17, T-39 |
| RF-3.5 | T-10, T-17, T-70 |
| RF-3.6 | T-06, T-17, T-39, T-46, T-53 |
| RF-3.7 | T-10, T-17, T-39, T-46, T-53 |
| RF-3.8 | T-17, T-39, T-40, T-53 |
| RF-4.1 | T-11, T-18, T-40, T-54 |
| RF-4.2 | T-01, T-14, T-18, T-66 |
| RF-4.3 | T-55 |
| RF-4.4 | T-06, T-19, T-40, T-54 |
| RF-4.5 | T-19, T-40, T-54, T-55 |
| RF-4.6 | T-11, T-18, T-22, T-48, T-63, T-70 |
| RF-4.7 | T-48, T-55 |
| RF-4.8 | T-04, T-08, T-11, T-18, T-37 |
| RF-5.1 | T-02, T-07, T-19, T-63 |
| RF-5.2 | T-64, T-67 |
| RF-5.3 | T-27, T-49, T-56, T-67 |
| RF-5.4 | T-56, T-61, T-67 |
| RF-5.5 | T-05, T-19, T-33 |
| RF-5.6 | T-19, T-56 |
| RF-6.1 | T-03, T-09, T-16, T-30, T-63 |
| RF-6.2 | T-01, T-03, T-17, T-29, T-36, T-66 |
| RF-6.3 | T-09, T-11, T-22, T-29, T-32, T-48 |
| RF-6.4 | T-09, T-15, T-26, T-64 |
| RF-7.1 | T-57 |
| RF-7.2 | T-15, T-25, T-42, T-50, T-57 |
| RF-7.3 | T-02, T-07, T-25, T-42, T-57 |
| RF-7.4 | T-25, T-57, T-58 |
| RF-7.5 | T-58 |
| RF-7.6 | T-02, T-08, T-25, T-42, T-58 |
| RF-7.7 | T-25, T-42, T-58 |
| RF-7.8 | T-25, T-42 |
| RF-8.1 | T-50, T-59 |
| RF-8.2 | T-23 |
| RF-8.3 | T-01, T-04, T-06, T-12, T-23, T-41, T-59, T-66 |
| RF-8.4 | T-06, T-12, T-23, T-41, T-59, T-61 |
| RF-8.5 | T-18, T-24, T-41, T-54, T-59 |
| RF-8.6 | T-08, T-20 |
| RF-8.7 | T-22 |
| RF-8.8 | T-59 |
| RF-8.9 | T-05, T-20, T-33 |
| RF-9.1 | T-50, T-60 |
| RF-9.2 | T-04, T-12, T-23, T-41 |
| RF-9.3 | T-16, T-24, T-41, T-60 |
| RF-9.4 | T-26, T-64 |
| RF-9.5 | T-22 |
| RF-9.6 | T-02, T-07, T-08, T-21, T-64 |
| RF-9.7 | T-05, T-15, T-21, T-32, T-33 |
| RF-9.8 | T-21, T-32 |
| RF-9.9 | T-60, T-65, T-65 |
| RF-9.10 | T-60 |
| RF-10.1 | T-44, T-70 |
| RF-10.2 | T-13, T-27 |
| RF-10.3 | T-65 |
| RF-10.4 | T-44 |
| RF-10.5 | T-65 |
| RF-11.1 | T-27, T-29, T-34 |
| RF-11.2 | T-06, T-34, T-35, T-45 |
| RF-11.3 | T-07, T-16, T-37, T-40, T-44, T-70 |
| RF-11.4 | T-05, T-33 |
| RF-11.5 | T-11, T-17, T-19, T-24, T-30, T-40, T-47, T-63 |
| RF-12.1 | T-50, T-62, T-68 |
| RF-12.2 | T-62, T-68, T-69 |
| RF-12.3 | T-50, T-68, T-69 |
| RF-12.4 | T-31, T-38, T-52, T-68 |
| RF-12.5 | T-46, T-47, T-67 |
| RNF-1 | T-34, T-35, T-37, T-45, T-48 |
| RNF-2 | T-11, T-18, T-48, T-54, T-63, T-70 |
| RNF-3 | T-19, T-20, T-21, T-30, T-63 |
| RNF-4 | T-07, T-21, T-64, T-64 |
| RNF-5 | T-12, T-23, T-24, T-64, T-64, T-70 |
| RNF-6 | T-01, T-66 |
| RNF-7 | T-30, T-65, T-65 |
| RNF-8 | T-44, T-65, T-70 |
| RNF-9 | T-05, T-33 |
| RNF-10 | T-62, T-68, T-69 |
| RNF-11 | T-46, T-67 |
| RNF-12 | T-62, T-68 |
| RNF-13 | T-15, T-19 |
| LC-1 | T-09, T-22, T-48, T-51 |
| LC-2 | T-09, T-17, T-48, T-53 |
| LC-3 | T-09, T-11, T-18, T-48 |
| LC-4 | T-11, T-18, T-48 |
| LC-5 | T-10, T-17, T-29 |
| LC-6 | T-10, T-17, T-53 |
| LC-7 | T-10, T-17 |
| LC-8 | T-03, T-16, T-17, T-30, T-63 |
| LC-9 | T-22, T-55 |
| LC-10 | T-19, T-54 |
| LC-11 | T-11, T-19, T-24, T-30, T-47, T-63 |
| LC-12 | T-02, T-08, T-25, T-58 |
| LC-13 | T-12, T-22 |
| LC-14 | T-23 |
| LC-15 | T-12, T-23 |
| LC-16 | T-22 |
| LC-17 | T-26, T-32, T-64 |
| LC-18 | T-21, T-32 |
| LC-19 | T-21, T-64 |
| LC-20 | T-13, T-27, T-43, T-49 |
| LC-21 | T-13, T-27 |
| LC-22 | T-28, T-49, T-67 |
| LC-23 | T-13, T-27, T-43, T-49 |
| LC-24 | T-16, T-61 |
| LC-25 | T-16 |
| LC-26 | T-65 |
| LC-27 | T-60 |
| LC-28 | T-65 |
| LC-29 | T-33 |
| LC-30 | T-25, T-57 |
| US-1 … US-11 | T-67 (journeys) |
| US-12 | T-70 (adversarial suite) |

Self-check: all 83 RF criteria, 13 RNF, 30 limit cases and 12 user stories of `specEnrollment.md` v0.2 appear above; every task of phases A–G names at least one requirement, and the closing tasks name the completion criterion they serve.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| A-27 identity statuses | `StartEnrollmentUseCase` accepts `ACTIVE` only; `UNVERIFIED` takes the verification handoff; `RESTRICTED` and `ERASING` are refused | T-16, T-38, T-61 |
| A-32 time tolerance | The single global value of `LoginPolicy` **[001]**, used for the pending seed and the active factor; no tolerance field in `EnrollmentPolicy` | T-01, T-17, T-23, T-66 |
| A-35 persistence tooling | Adapters and migrations against the placeholder access API and migration tool | T-02, T-03, T-04, T-30 |
| A-39 revocation scope | `RevokeSessionsPort.revokeAll(identity, except)` revokes every other session and refresh family; run-scoped authority untouched | T-15, T-21, T-32 |
| A-46 QR encoding library (decided 2 November 2026) | ZXing `core` 3.5.3 behind the `QrEncoder` port, one BE adapter rendering an inline SVG; recorded in P10 and A-18 | T-15, T-31 |
| A-37 … A-43, S-28 … S-34 (decided) | Implemented as decided: notifications (T-05, T-33), invalidation at acknowledgement (T-20), revocation scope (T-21), two configuration families (T-01), text key (T-16, T-52), next-derivation gate (T-27), session-bound pending enrollment (T-16, T-29), reflow (T-62), not-enrolled card / count / warning (T-57, T-58), download content (T-55), step-2 copy (T-46, T-53), headings (T-51), "Leave setup" destinations (T-51) | T-72 reviews all |
