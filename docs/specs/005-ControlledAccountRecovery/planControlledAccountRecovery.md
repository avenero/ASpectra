# Plan 005 — Controlled account recovery

**Feature** Recovery of a lost authenticator: an unused recovery code, a verified recovery request, and the continuation of a recovery case · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/005-ControlledAccountRecovery/specControlledAccountRecovery.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-07 as the subject; F-IDN-03, F-IDN-04, F-IDN-05, F-IDN-06, F-IDN-08, F-IDN-09, F-IDN-15, F-IDN-16 and F-SET-03 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG) and FE board `46:7847` (feature-modular SPA DAG) · **Builds on** plan 001 (Login), plan 002 (Signup and verification), plan 003 (TOTP enrollment, recovery codes and factor replacement) and plan 004 (Password reset); elements they own are tagged **[001]** / **[002]** / **[003]** / **[004]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's seven decisions (Q1–Q7) and its two derived rules (D-a, D-b) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-47, A-48, A-27, A-30, A-39, S-38, S-14, S-03, S-17; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | No use case of this feature reads, stores or replays a mapping, approval, run or intent; every path ends in the identity tables, the outbox and — for a public entry — the session issue of plan 001. An accepted Run is untouched by the revocation (A-39). | RF-3.8, RF-9.1, RNF-10 |
| **P3** The browser expresses intent, never authority | The FE submits an address, a password, a code, a reference, a one-time code or an acknowledgement and renders the server's answer. Eligibility, code consumption, case state, approval validity, the verified window, revocation and the post-replacement landing are all decided server-side; the case view the FE shows is a server projection after a server-verified continuation. | RF-2.2, RF-5.8, RF-6.5, RF-7.3, RNF-8 |
| **P6** Fail closed | An unknown, consumed or malformed code, an unknown, closed, expired or decoy reference, a missing, expired or voided one-time code, an expired approval, a voided recovery transaction, an absent policy or an absent configuration value refuses the effect; no default is assumed and no fallback exists. | RF-6.4, RF-6.10, RF-8.7, RNF-6, RNF-7 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security `PasswordEncoder` and MFA support, the existing outbox and durable jobs. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. Nothing is added: no cache store, no queue, no new identity provider, no reviewer tooling. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the feature lives in the existing hexagonal bounded context `identity`; outer adapters depend inward; the session service, `audit` and `sharedsupport` are reached only through published ports; mail goes through the transactional outbox; expiry through durable jobs. FE: the same feature module as Login; TanStack Query owns server facts; Zustand is not used (no unsaved fork). | §2 |
| **P12** Two independent tenancy gates | Recovery cases, continuation challenges, recovery transactions and the pending enrollment they own are **platform** rows of a global identity; they carry no `tenant_id` and are protected by application authority. No tenant, membership, grant, approval or run table is read or written; a case is invisible to every tenant surface. | RF-4.7, RF-5.9, RF-9.1, LC-6 |
| **P13** Configured values are never defaulted | Case lifetime, approval validity, one-time-code lifetime, length and attempts, recovery-transaction expiry and attempts, the verified-window idle time, the three throttle families, the message caps and the default message language are one validated configuration record; startup fails on any absent or unbounded value. | RF-8.7, RNF-6 |
| **P14** Release bar | WCAG 2.2 AA on N05 and all its states with automated and manual assistive evidence; EN / ES / FR for the screen and the five messages. | RF-10, RNF-13, RNF-14, RNF-15 |
| **F-X-01** C10 context | No C10 is derived anywhere in this feature: the screen is public, the recovery transaction is a limited identity state, and the in-session variant is identity-scoped. Tenant entry after a recovery re-derives C10 and the enrollment gate exactly as plan 001 RF-8.5 and plan 003 D7. | RF-1.6, RF-3.9, RNF-8 |
| **F-X-02** Request integrity | Code verification, request, continuation, one-time-code entry and the start of the replacement are consequential writes with no generic auto-retry; the two state-changing submissions of the continuation carry an idempotency key whose outcome is recorded and returned on re-submission; the enrollment steps inherit plan 003 D13. A lost answer leads to a lookup, never a second consumption or a second start. | RF-6.4, RF-6.8, RNF-3 |
| **F-X-03** Secrets and safe problems | Passwords, codes and one-time codes exist only in the request body and the verifier; references and one-time codes at rest only as hashes; problem details carry safe codes, field paths and a correlation id; browser storage holds nothing from this feature. | RF-4.8, RF-11.3, RF-11.4, RNF-4 |
| **F-X-05** Audit log | Every event of RF-11.1 is appended through the `audit` context port, redacted, in the same transaction as the effect; review transitions carry the acting platform actor. | RF-11.1, RF-11.2 |
| **F-X-06** Configuration validation and email port | `RecoveryPolicyProperties` is part of startup validation; the five messages and the replacement notification go through `TransactionalEmailPort` behind the outbox, so a delivery failure never reverts an effect. | RF-8.7, RF-11.5, RNF-12 |
| **F-X-07** Responsive surfaces | M18 (`432:4185`) is the drawn counterpart of N05; tablet follows the D01 responsive policy with `SecurityStatements` shared across widths. | RF-10 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity` (shared with Login, Signup, Enrollment and Password reset)

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 001–004. This plan adds two aggregates, one challenge, five use cases, two internal services, one internal command, one durable job, one published port and one controller; it extends two aggregates the earlier plans own. Contexts touched — `audit`, `sharedsupport`, and the session service of F-IDN-08 — are reached only through published ports (allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `GlobalIdentity` **[001]** | Unchanged shape. `isResetEligible()` **[004]** (status `ACTIVE` and email verified, A-27) is reused as the single eligibility predicate of every recovery path; `bumpSecurityVersion()` **[004]** is called by the replacement completion. | RF-2.2, RF-4.4, RF-4.6, RF-3.4 |
| | `RecoveryCodeSet` **[001]** | `consume(rawCode, verifier)` reused unchanged as the only reader of the code hashes; consumption happens under a row lock on the identity's code rows (D2). | RF-2.2, RF-2.4, LC-2, LC-9 |
| | `LoginTransaction` **[001]** | Gains the purpose `RECOVERY` and the state `REPLACEMENT_PENDING`, holding the id of the pending enrollment it owns: `openForRecovery(identityId, pendingId, now, policy)`, `complete` (allowed only once the owned enrollment is acknowledged), `void(reason)` with the reasons `ABANDONED`, `EXPIRED`, `REPLACEMENT_DISCARDED`, `CREDENTIAL_CHANGED`, `FACTOR_CHANGED` **[001]**. The stored `securityVersion` is re-stamped by its own completion so the transaction survives the factor swap it caused (as plan 004 D8 does for the forced change). The opaque token authorises only the enrollment endpoints of its pending id and `DELETE challenge`. | RF-2.5, RF-3.5, RF-3.7, RF-6.8, RNF-8, LC-4, LC-29 |
| | `PendingEnrollment` **[003]** | The owner becomes polymorphic: `owner = SESSION(sessionId)` **[003]** or `RECOVERY_TRANSACTION(transactionId)`; `startedBy = STEP_UP` **[003]**, `RECOVERY_CODE` or `RECOVERY_CASE`, with the case id when started from a case. State machine, steps, attempts, expiry and discard reasons are unchanged; `owner` is compared on every call exactly as the session was. | RF-2.6, RF-3.1, RF-3.2, RF-3.7, LC-32 |
| | `RecoveryCase` | Aggregate of one case: identity id, `state` (`RECEIVED → UNDER_REVIEW → APPROVED \| REJECTED`; `RECEIVED` and `UNDER_REVIEW → EXPIRED \| CLOSED`; `APPROVED → COMPLETED \| EXPIRED \| CLOSED`), reference hash, `openedAt`, `expiresAt`, `reviewStartedAt` / `reviewedBy`, `approvedAt` / `approvedBy` / `approvalExpiresAt`, `closedAt`, `closeCause` (`REJECTED`, `CASE_EXPIRED`, `APPROVAL_EXPIRED`, `CODE_CONSUMED`, `REPLACEMENT_COMPLETED`, `IDENTITY_RESTRICTED`, `IDENTITY_ERASING`, `COMPLETED`), `requestLocale`. Methods: `moveToReview(actor, now)`, `approve(actor, validity, now)`, `reject(actor, now)`, `expire(now)`, `close(cause, now)`, `complete(now)`, `isOpen()`, `isApprovedAndValid(now)`. Pure; takes the clock as an argument. | RF-4.4, RF-5.1, RF-5.3, RF-5.4, RF-5.6, RF-5.8, RF-6.10, LC-17, LC-21, LC-27 |
| | `ContinuationChallenge` | One continuation attempt: `caseId` (absent for a decoy), one-time code hash, `expiresAt`, `attemptsRemaining`, state `OPEN → VERIFIED(verifiedUntil) \| VOIDED`, and the recorded outcome of each submission by its idempotency key. Methods: `verify(codeOk, now, idleWindow)`, `recordFailedAttempt`, `void`, `isVerified(now)`, `recordedOutcome(key)`. Mirrors `StepUpChallenge` **[003]**. | RF-6.2, RF-6.3, RF-6.4, RF-6.9, LC-22–LC-24, LC-30 |
| | `AbuseThrottlePolicy` **[001]** | Reused unchanged; dimension keys `RECOVERY_ADDRESS`, `RECOVERY_REFERENCE` and `RECOVERY_NETWORK` are added with their own configured steps and windows. A wrong password on any card is additionally written on the sign-in keys `ACCOUNT` / `NETWORK` **[001]** (D-a); a wrong code, reference or one-time code with a correct password is written on the recovery keys only. | RF-2.3, RF-8.1–8.5, LC-3, LC-16 |
| | `RecoveryPolicy` | Immutable record: case lifetime, approval validity, one-time-code lifetime, length and attempt limit, recovery-transaction expiry and attempt limit, verified-window idle time, throttle steps and windows for the three keys, per-address caps and windows for re-sends, one-time codes and notifications, the public origin (shared configuration value with `SignupPolicy` **[002]**, used for the link in the messages), default message language. | RF-8.6, RF-8.7, RNF-6 |
| `identity.application` | `VerifyRecoveryCodeForReplacementUseCase` | Path (a). Normalises the address with `EmailAddress` **[002]**; evaluates `RECOVERY_ADDRESS` and `RECOVERY_NETWORK`; loads the identity (or not); always runs exactly one password verification — against the real credential when the identity `isResetEligible()`, against the dummy hash (`PasswordVerifier.verifyAgainstDummy()` **[001]**) otherwise; always runs one code comparison bounded by the configured code count — against the identity's unused codes under `SELECT … FOR UPDATE` when the password matched, against a dummy set of the same size otherwise (D1); any failure → `auth.recovery.rejected`, nothing consumed, throttle counters written (sign-in keys too for a wrong password), audit `RECOVERY_CODE_REJECTED`. Success, in one transaction: `RecoveryCodeSet.consume`, audit `RECOVERY_CODE_CONSUMED`, `RecoveryCaseClosurePort.close(identity, CODE_CONSUMED)`, enqueue `recovery-code-used`; then **public entry**: create the `LoginTransaction(RECOVERY)` and `StartEnrollmentUseCase(REPLACEMENT)` **[003]** with `owner = RECOVERY_TRANSACTION`, `startedBy = RECOVERY_CODE`; **session entry** (identity taken from `CurrentSessionPort` **[003]**, submitted address refused): `StartEnrollmentUseCase(REPLACEMENT)` with `owner = SESSION`, no `StepUpUseCase`. Returns the setup material once with the transaction token (public) or without (session). Never a session, never a count. | RF-1.4, RF-1.8, RF-2.2–2.8, RF-3.1, RF-5.6, RF-8.4, RF-9.5, LC-1–LC-3, LC-8, LC-9, LC-18 |
| | `RequestRecoveryCaseUseCase` | Path (b). Same normalisation and throttles (`RECOVERY_ADDRESS`, `RECOVERY_NETWORK`); the acknowledgement must be `true` (field problem otherwise); one password verification (real or dummy) in every branch; then: eligible and matched with an open case → re-send `recovery-case-opened` with the same reference within the re-send cap, audit `RECOVERY_REQUEST{matched, action = RESENT}`; eligible and matched without an open case → create the `RecoveryCase` in `RECEIVED` with a 256-bit reference from `TokenGenerator` **[002]** stored as its `TokenHasher` **[002]** hash, record the request locale, enqueue `recovery-case-opened` (reference in clear only inside the message), audit `RECOVERY_REQUEST{matched, action = CREATED}` and `RECOVERY_CASE_TRANSITION{RECEIVED}`; otherwise nothing but throttle counters and audit `RECOVERY_REQUEST{matched, action = NONE}`. The response is empty in every branch. Touches no other row. | RF-4.2–4.8, RF-5.1, RF-5.10, RF-8.6, LC-10–LC-15, LC-33 |
| | `ContinueRecoveryCaseUseCase` | Path (c), first step. Evaluates `RECOVERY_REFERENCE` and `RECOVERY_NETWORK`; hashes the reference; loads a case only if it is open (`RECEIVED`, `UNDER_REVIEW`, `APPROVED`); one password verification against the case identity's credential, or the dummy hash when no open case matched; when both hold, creates a `ContinuationChallenge` bound to the case with a one-time code of the configured length from `TokenGenerator`, stored as a maintained-encoder hash, and enqueues `recovery-one-time-code` (locale from the request header); otherwise creates a **decoy** challenge with no case and sends nothing. Either way answers with the opaque continuation token, so response and work are the same (D6). Wrong password → sign-in keys too. Audit `RECOVERY_CONTINUATION_STARTED{matched}`. | RF-6.1–6.3, RF-6.10, RF-7.2, RF-8.4, LC-22, LC-23 |
| | `VerifyContinuationCodeUseCase` | Path (c), second step. Locks the challenge row; checks state `OPEN`, expiry and attempts; returns the recorded outcome for a repeated idempotency key; verifies the one-time code with the maintained encoder (against the dummy hash for a decoy); wrong → `recordFailedAttempt`, `auth.recovery.rejected`, recovery keys; zero or expiry → `void`, `auth.recovery.continuation.void`; success → `VERIFIED` with `verifiedUntil = now + idle`, audit `RECOVERY_CONTINUATION_VERIFIED`, and the case view: `state`, masked address, `requestedAt`, `outcome` for a case closed since the continuation started, `canContinue = isApprovedAndValid(now)`. Never a display name, tenant, membership, reviewer or time promise. | RF-6.4–6.7, RF-6.9, RF-7.1, LC-24, LC-25, LC-27 |
| | `StartReplacementFromCaseUseCase` | Path (c), the approved action. Locks the challenge; requires `VERIFIED` and unexpired `verifiedUntil`, and the case `APPROVED` with a valid approval; returns the recorded outcome for a repeated key; then — as the success branch of path (a) — public entry: `LoginTransaction(RECOVERY)` + `StartEnrollmentUseCase(REPLACEMENT)` with `owner = RECOVERY_TRANSACTION`, `startedBy = RECOVERY_CASE(caseId)`; session entry: owner `SESSION`. The approval is **not** consumed (D-b): the case stays `APPROVED` until the completion of RF-3.4 or its validity ends. Audit `RECOVERY_REPLACEMENT_STARTED{source = CASE}`. | RF-3.1, RF-5.8, RF-6.7, RF-6.8, LC-26, LC-28, LC-29 |
| | `RecoveryReplacementCompletion` (internal service) | Called by `AcknowledgeRevealUseCase(REPLACEMENT)` **[003]** in its own transaction when the pending enrollment's `startedBy` is `RECOVERY_CODE` or `RECOVERY_CASE`, after the atomic swap of plan 003 D4: `bumpSecurityVersion` **[004]**; `RevokeSessionsPort.revokeAll(identityId, except)` **[004]** with `except = none` for a recovery-transaction owner and `except = owningSessionId` for a session owner; `RecoveryCaseClosurePort.complete(identity)` (an open case moves to `COMPLETED`; the case the replacement started from, if any, is the one completed); enqueue `authenticator-replaced` **[003]** in its *through account recovery* variant; audit `RECOVERY_REPLACEMENT_COMPLETED{source}` and `SESSIONS_REVOKED{cause = RECOVERY}`; for a recovery-transaction owner, finally `CompleteSignInUseCase` **[001]** on that transaction, which issues the session and returns the landing. | RF-3.3–3.6, RF-3.8, RF-5.6, RNF-10, RNF-11, LC-5–LC-8, LC-28 |
| | `CloseRecoveryCaseService` — published as `RecoveryCaseClosurePort` | `close(identityId, cause)` and `complete(identityId)`: moves any open case of the identity to `CLOSED` (or `COMPLETED`), records the cause, enqueues `recovery-case-closed` in the case's stored locale (none for `COMPLETED`, which is covered by the replacement notification), audits `RECOVERY_CASE_TRANSITION`. Idempotent (no open case → no-op). Callers: path (a) above; `VerifyRecoveryCodeUseCase` **[001]** (a code consumed at the challenge, LC-18); `AcknowledgeRevealUseCase(REPLACEMENT)` **[003]** from R41 (LC-19); the identity status transitions to `RESTRICTED` / `ERASING` (A-27; LC-21). Plan 004's `ChangeCredential` never calls it (RF-5.7). | RF-5.6, RF-5.7, LC-18–LC-21 |
| | `ReviewRecoveryCaseCommand` (internal, no HTTP surface) | `moveToReview(caseId, actor)`, `approve(caseId, actor)` (validity from `RecoveryPolicy`), `reject(caseId, actor)`: each locks the case, applies the aggregate transition, audits `RECOVERY_CASE_TRANSITION{state, actor}`, and enqueues `recovery-case-approved` or `recovery-case-closed{REJECTED}` in the case's stored locale. Invoked only by the audited operational mechanism of A-47 (§6, D8); no tenant use case and no endpoint can reach it. | RF-5.2–5.5, RF-9.2, RNF-9 |
| | `ExpireRecoveryCasesJob` | Durable job (`sharedsupport/jobs` **[002]**): claims cases past `expiresAt` in `RECEIVED` / `UNDER_REVIEW` → `EXPIRED{CASE_EXPIRED}`, cases past `approvalExpiresAt` in `APPROVED` → `EXPIRED{APPROVAL_EXPIRED}`, each with `recovery-case-closed` and audit; voids continuation challenges and recovery transactions past expiry (`EXPIRED`) and discards the pending enrollment of a voided transaction through `DiscardPendingEnrollmentUseCase` **[003]**. Check-on-access in every use case enforces expiry first; the job only cleans up. | RF-5.1, RF-5.4, RF-3.7, LC-17, LC-27 |
| | `AbandonChallengeUseCase` **[001]** | Reused for a recovery transaction: voids it (`ABANDONED`) and discards its pending enrollment; idempotent; the consumed code stays consumed and an approved case stays approved. | RF-3.7, LC-4, LC-26 |
| | Ports | New: `RecoveryCaseRepository` (`findOpenByIdentity`, `findOpenByReferenceHash`, `save`, `claimExpired`), `ContinuationChallengeRepository`, `RecoveryCaseClosurePort` (published; consumers above). Reused: `IdentityRepository`, `LoginTransactionRepository`, the `identity_recovery_code` persistence behind `RecoveryCodeSet`, `ThrottleStore`, `PasswordVerifier`, `Clock`, `AuditPort`, `IssueSessionPort` (through `CompleteSignInUseCase`) **[001]**; `TokenGenerator`, `TokenHasher`, `TransactionalEmailPort` (behind the outbox), `SecurityNotificationPort`, `JobScheduler` **[002]**; `PendingEnrollmentRepository`, `CurrentSessionPort` **[003]**; `RevokeSessionsPort` **[004]**. No tenant, membership, grant, approval or run port exists in this feature. | all |
| `identity.infrastructure` | Persistence | New `recovery_case` (id, identity id, state, `reference_hash` unique, opened / expires / review_started / approved / approval_expires / closed timestamps, `reviewed_by`, `approved_by`, `close_cause`, `request_locale`; **partial unique index** on `identity_id WHERE state IN ('RECEIVED','UNDER_REVIEW','APPROVED')`) and `recovery_continuation` (id, case id nullable, code hash, expires_at, attempts_remaining, state, verified_until, created_at, recorded outcomes). `login_transaction` **[001]** gains the purpose value `RECOVERY`, the state value `REPLACEMENT_PENDING` and `pending_enrollment_id`. `pending_enrollment` **[003]** replaces `owning_session_id` with `owner_kind` + `owner_id` and gains `started_by` + `case_id`. `signin_throttle` **[001]** gains the three recovery dimension values. Every mutating use case reads its row `SELECT … FOR UPDATE`; code consumption locks the identity's `identity_recovery_code` **[001]** rows. Persistence access API and migration tool per A-35 (placeholders). | RF-2.4, RF-4.4, RF-4.8, RF-5.1, RF-6.2, RNF-3, LC-9, LC-14, LC-29 |
| | Outbox relay and email adapter **[002]** | Five new templates in EN / ES / FR — `recovery-code-used`, `recovery-case-opened` (carries the reference and a link to the recovery screen on the public origin), `recovery-one-time-code`, `recovery-case-approved` (validity, the continuation steps, nothing changed yet), `recovery-case-closed` (reason: rejected, expired, recovered by another route, no longer eligible) — plus the *through account recovery* variant of `authenticator-replaced` **[003]**; content per S-14 (§6); none promises a completion time. Language: the request header for code-used, opened and one-time-code; the locale stored on the case for approved and closed (review-originated messages have no request); the acknowledging request's header for the replacement notification. Relay failures are retried by the relay, never surfaced. | RF-2.8, RF-3.4, RF-4.4, RF-5.3, RF-5.4, RF-5.5, RF-11.5, RNF-12, RNF-14, LC-33 |
| | `RecoveryPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding of every value into `RecoveryPolicy`; constraints: present, positive, bounded above by a configured ceiling; one-time-code length at least a configured floor; the public origin an absolute `https` origin; the default language one of EN / ES / FR. Any violation fails startup. | RF-8.7, RNF-6 |
| | Review entry | The operational mechanism that invokes `ReviewRecoveryCaseCommand` is a placeholder until A-47 decides the operator path: in this plan an administrative command runnable only from the deployment environment with an actor identifier and a case id, never an HTTP endpoint and never a tenant screen (the same shape as plan 004's compromise-response entry). | RF-5.2, RF-9.2 |
| | Session adapters | `RevokeSessionsPort` **[004]** and `CurrentSessionPort` **[003]** against the existing session service; no change to JWT / refresh mechanics. | RF-3.4, RF-3.6, RF-1.4 |
| | Audit adapter | Events: `RECOVERY_CODE_REJECTED{matched}`, `RECOVERY_CODE_CONSUMED`, `RECOVERY_TRANSACTION{OPENED \| COMPLETED \| ABANDONED \| EXPIRED}`, `RECOVERY_REPLACEMENT_STARTED{source}`, `RECOVERY_REPLACEMENT_COMPLETED{source}`, `RECOVERY_REQUEST{matched, action}`, `RECOVERY_CASE_TRANSITION{state, cause, actor?}`, `RECOVERY_CONTINUATION_STARTED{matched}`, `RECOVERY_CONTINUATION_VERIFIED`, `RECOVERY_CONTINUATION_FAILED`, `RECOVERY_CONTINUATION_VOIDED`, `ONE_TIME_CODE_SENT`, `SESSIONS_REVOKED{cause = RECOVERY}`, `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_SUPPRESSED{template, reason = CAP}`, `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]** for the recovery keys. Payload: identity id when matched, correlation id, network key, timestamp, outcome kind. Never a password, a code, a one-time code, a reference, an unmatched address or a message body. | RF-11.1, RF-11.2 |
| `identity.api` | `RecoveryController` — `/api/auth/recovery` (public; a session, when present, supplies the identity and the address field is refused) | `POST code` {email?, password, code} → 200 `{state: REPLACEMENT_STARTED, transaction?, pendingId, purpose: REPLACEMENT, setup: {uri, key, svg}, expiresAt}` or 400 `auth.recovery.rejected` (no cause) or 400 field-path problems · `POST request` {email?, password, acknowledged} → 202 `{}` always, or 400 field-path problems · `POST case/continue` {reference, password} → 202 `{continuation}` always, or 400 field-path problems · `POST case/verify` {continuation, idempotencyKey, code} → 200 `{case: {state, maskedEmail, requestedAt, outcome?}, canContinue}` or 400 `auth.recovery.rejected` or 410 `auth.recovery.continuation.void` · `POST case/replacement` {continuation, idempotencyKey} → 200 as `POST code` or 410 `auth.recovery.continuation.void`. The enrollment steps reuse `POST /api/account/mfa/enrollment/{id}/verify \| reveal \| acknowledge` and `DELETE …/{id}` **[003]**, authorised by a session **or** by the recovery transaction that owns `{id}`; `acknowledge` on a recovery-transaction-owned enrollment answers 200 `{state: RECOVERED, landing}` (a session-owned one keeps plan 003's `{state: REPLACED, …}`). `DELETE /api/auth/challenge` **[001]** voids a recovery transaction. The UI locale travels in the request header. All problems are RFC 9457 problem details with a safe code, field paths and a correlation id. The server never redirects. | RF-1.3, RF-1.4, RF-1.6, RF-1.7, RF-2, RF-3.5, RF-3.7, RF-4.2, RF-4.3, RF-6, RF-7.1, RF-11.3 |

**Cross-context touch points.** `audit.AuditPort` — append-only. `sharedsupport` — `crypto` (token generation and hashing, the maintained encoder), `jobs`, `tx`, `contextIds` (correlation id), `outbox`. The session service of F-IDN-08 through `RevokeSessionsPort` and `CurrentSessionPort` only. Nothing reads or writes tenant, membership, grant, approval or run tables.

**Data model note (P12).** `recovery_case`, `recovery_continuation`, the recovery rows of `login_transaction` and `pending_enrollment` are platform-scoped rows of a global identity and carry no tenant column; RLS does not apply to them.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login, Signup, Enrollment and Password reset)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/recover` (N05; public; when a session exists the page renders its in-session variant with the address read-only). The replacement wizard is a nested state of this route for a public entry (`/recover/replace`, driven by the recovery transaction holder) and the existing `/account/security/enroll` **[003]** for a session entry. | RF-1.1, RF-1.4, RF-3.1 |
| | `useRecoveryCode()`, `useRecoveryRequest()`, `useContinueCase()`, `useVerifyContinuation()`, `useStartReplacementFromCase()` | Typed intent hooks; attach the UI locale header; return server state verbatim. | RF-2–RF-6 |
| | `contracts/recovery` (in `src/contracts`) | Request / response / problem types and runtime guards for the five endpoints, the `RECOVERED` acknowledge response and the recovery-owned enrollment start; a guard rejects any response carrying a token, password, code, display name, tenant or count. | all |
| Views | `RecoveryPage` | N05 heading, subtitle, the two tabs "Recovery code" / "Lost all factors" (keyboard-operable), the three cards, the "No administrator backdoor" notice, "← Back to sign in", "Return to local sign in", Privacy / Terms, language switch; in-session variant with read-only address. | RF-1.1, RF-1.4, RF-1.6, RF-1.9, RF-9.2 |
| | `RecoveryCodeCard` | Address (pre-filled from the voided challenge when present, editable), `PasswordField` with Show, `CodeField` **[001]** with paste and separator stripping, "Verify recovery code", the statement; generic failure keeps the address and clears password and code; on `REPLACEMENT_STARTED` stores the transaction and opens the wizard. | RF-1.3, RF-1.7, RF-2.1, RF-2.3, RF-2.7, RF-7.1 |
| | `RecoveryRequestCard` | Statements, "What remains protected", address, `PasswordField`, the never-pre-ticked acknowledgement `Checkbox`, "Start verified recovery request", the reference statement; received state: the generic statement above the same card, focus on it, address kept, password cleared, action available. | RF-4.1–4.3, RF-5.5 |
| | `ContinueCaseCard` | Reference field, `PasswordField`, "Continue case verification", statement; one-time-code step with its statement and code field; generic failure; void → back to the card; `CaseDetails` after verification. | RF-6.1, RF-6.3, RF-6.4, RF-7.1 |
| | `CaseDetails` | State as text, masked address, request time, outcome when closed; received / under review: statement and "Return to sign in" only; approved: "Continue to replace your authenticator"; never a time promise; cleared when the idle window ends. | RF-6.5–6.7, RF-6.9, RF-5.5 |
| | `EnrollmentWizard` **[003]** | Reused for the replacement with the replacement heading and the statement that the existing factor remains active; "Leave setup" → abandon; for a recovery-owned enrollment the acknowledge response `RECOVERED` is handed to `LandingRouter` **[001]**. | RF-3.1, RF-3.5, RF-3.7 |
| | `SecurityStatements` **[001]** | Extended with every statement of N05 and of its D02 states (`423:3542`; S-38) so desktop and reflow cannot drift. | RF-1.1, RF-2.1, RF-4.1, RF-6.1, RF-10.1 |
| | `GenericFailureMessage` **[001]** | Reused for `auth.recovery.rejected` ("We couldn't verify these details."). | RF-2.3, RF-6.4, RF-7.1 |
| State | TanStack Query | Mutations for the five endpoints (`case/verify` and `case/replacement` generate their idempotency key once and reuse it on re-submission); no retry on mutations (F-X-02); the session presence read from the existing session query **[001]**. | RF-2–RF-6, RNF-3 |
| | Recovery transaction holder | The transaction holder **[001]** extended with the purpose `RECOVERY` and the pending enrollment id; cleared on `RECOVERED`, void, abandon and route leave; `DELETE challenge` fired on leave and, best-effort, on `pagehide` (plan 003 D9). | RF-2.5, RF-3.7, RF-11.4, LC-4, LC-32 |
| | Continuation holder | Module-private in-memory `{continuation, caseView, verifiedUntil}`; the idle timer clears it; cleared on route leave; nothing in storage. | RF-6.9, RF-11.4, LC-30 |
| | Enrollment / setup / reveal holders **[003]** | Reused unchanged. | RF-3.1 |
| Routing | Handoffs | R02 lost-authenticator route **[001]**, R05 controlled-recovery entry **[001]** (after `DELETE challenge`), R41 "I lost access · controlled recovery" **[003]** and the R38 `RecoveryCodesCard` route **[003]** → `/recover`; `REPLACEMENT_STARTED` → wizard; `RECOVERED` → `LandingRouter` **[001]**; session-owned completion → `/account/security` **[003]**; `auth.recovery.continuation.void` → the continuation card, nothing else; abandon → `/sign-in` (public) or `/account/security` (session); "Back to sign in" / "Return to local sign in" → `/sign-in`. N20 (spec 014) carries no route here. | RF-1.2, RF-1.5, RF-3.5–3.7, RF-6.4, RF-6.8 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/errors` (field paths to fields), `shared/i18n` (bundle keys for every statement, state and message in EN / ES / FR), `shared/ui` (accessible `Field`, `PasswordField`, `CodeField`, `Checkbox`, `Tabs`, `LiveRegion`, 44 px targets). | RF-1.7, RF-1.10, RF-10.2, RNF-13, RNF-14 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | — | `RecoveryPage` with tabs, cards, notice, routes, switch |
| RF-1.2 | — | Routes from `SignInPage` **[001]**, `MfaChallengePage` **[001]**, `ReplaceAuthenticatorDialog` **[003]** and `RecoveryCodesCard` **[003]** target `/recover` |
| RF-1.3 | Recovery endpoints accept no login transaction; the challenge already called `AbandonChallengeUseCase` **[001]** | Address pre-filled from the voided holder for display only |
| RF-1.4 | With a session, `CurrentSessionPort` supplies the identity; a submitted address is refused with a field problem | In-session variant: read-only address; completion continues in session |
| RF-1.5 | — | No route from N20 (spec 014) |
| RF-1.6 | No response carries a display name, membership, factor state, count or case existence | Page renders none |
| RF-1.7 | `RecoveryController` field-path problems (server-side validation too) | `Field` validation, associated message, focus |
| RF-1.8 | `EmailAddress` **[002]** normalisation in every use case | Trim before submit |
| RF-1.9 | — | "Help" opens N43 (S-17); Privacy / Terms open the operator documents outside the product (S-14) |
| RF-1.10 | Nothing server-side depends on the locale except the template language | i18n re-render; holders untouched |
| RF-2.1 | — | `RecoveryCodeCard`: address, `PasswordField`, `CodeField` with paste, action, statement |
| RF-2.2 | `VerifyRecoveryCodeForReplacementUseCase`: eligibility, password, code as one evaluation | — |
| RF-2.3 | `auth.recovery.rejected` without cause; nothing consumed; counters per RF-8.4 | `GenericFailureMessage`; address kept, password and code cleared |
| RF-2.4 | `RecoveryCodeSet.consume` under the code-row lock, same transaction as the start | — |
| RF-2.5 | `LoginTransaction(RECOVERY)` in `REPLACEMENT_PENDING`; no `IssueSessionPort` call; token scoped to its pending id | Recovery transaction holder; wizard opens |
| RF-2.6 | Session entry → `StartEnrollmentUseCase(REPLACEMENT)` with `owner = SESSION`, no step-up | Wizard opens at `/account/security/enroll` |
| RF-2.7 | Responses carry no count | Nothing rendered |
| RF-2.8 | Outbox `recovery-code-used` via `SecurityNotificationPort` | — |
| RF-3.1 | `StartEnrollmentUseCase(REPLACEMENT)` **[003]** with `startedBy` | `EnrollmentWizard` **[003]** replacement heading and statement |
| RF-3.2 | Plan 003 RF-9.4 unchanged: `TotpFactor` untouched until acknowledge; `FactorStatusPort` ignores pending | — |
| RF-3.3 | `AcknowledgeRevealUseCase(REPLACEMENT)` **[003]** atomic swap | — |
| RF-3.4 | `RecoveryReplacementCompletion`: version bump, `RevokeSessionsPort`, `RecoveryCaseClosurePort.complete`, replaced-via-recovery template, audit | — |
| RF-3.5 | `CompleteSignInUseCase` **[001]** on the recovery transaction → session + landing | `RECOVERED` → `LandingRouter` **[001]** |
| RF-3.6 | Session owner → plan 003 RF-9.7 path (`except = owningSessionId`) | Back to `/account/security` |
| RF-3.7 | `DiscardPendingEnrollmentUseCase` **[003]**, `AbandonChallengeUseCase` **[001]**, the expiry job; code stays consumed; case untouched; no revocation, no replacement template | "Leave setup" → abandon → `/sign-in` or `/account/security` |
| RF-3.8 | Use cases write identity tables and the outbox only; no credential, address, membership, status or run access | No pending intent stored |
| RF-3.9 | `EnrollmentGate` **[003]** reads `hasActiveFactor` at tenant entry; nothing granted earlier | Tenant route is a plain navigation |
| RF-4.1 | — | `RecoveryRequestCard` |
| RF-4.2 | 400 field problem when `acknowledged != true` | Checkbox required, never pre-ticked |
| RF-4.3 | `RequestRecoveryCaseUseCase` constant-work path; 202 `{}` in every branch | Received state: statement, focus, address kept, password cleared, action available |
| RF-4.4 | Case created `RECEIVED`; `TokenGenerator` reference; `recovery-case-opened` | — |
| RF-4.5 | Open case → re-send within the cap; nothing created | Same received state |
| RF-4.6 | Ineligible or wrong-password branch writes throttle and audit only; dummy verification keeps the work equal | Same received state |
| RF-4.7 | Use case writes `recovery_case` and the outbox only | — |
| RF-4.8 | `TokenHasher`; `reference_hash` column only; reference in the message only | — |
| RF-5.1 | `RecoveryCase` state machine with times and actors | — |
| RF-5.2 | `ReviewRecoveryCaseCommand` internal; no endpoint; no tenant use case reaches `RecoveryCaseRepository` | — |
| RF-5.3 | `approve(actor, validity)`; `recovery-case-approved` in the stored locale | — |
| RF-5.4 | `reject` / `expire` → `recovery-case-closed{reason}` | — |
| RF-5.5 | No time field in any response, template or state | `CaseDetails` shows none |
| RF-5.6 | `RecoveryCaseClosurePort` called by 001 `VerifyRecoveryCodeUseCase`, path (a), 003 acknowledge, A-27 transitions; closure message | — |
| RF-5.7 | Plan 004 `ChangeCredential` does not call the closure port | — |
| RF-5.8 | `StartReplacementFromCaseUseCase` is the only reader of `APPROVED`; it grants a replacement start and nothing else | — |
| RF-5.9 | No tenant or `access` read of `recovery_case`; suspension, membership and factor untouched | Nothing rendered on tenant surfaces |
| RF-5.10 | `recovery_case` columns hold no secret and no tenant content | — |
| RF-6.1 | — | `ContinueCaseCard` |
| RF-6.2 | `ContinueRecoveryCaseUseCase`: open case + password → `ContinuationChallenge` + `recovery-one-time-code` | — |
| RF-6.3 | Decoy challenge when the checks fail; 202 `{continuation}` always | One-time-code step with the statement |
| RF-6.4 | `VerifyContinuationCodeUseCase`: generic failure, attempts, `void` | `GenericFailureMessage`; void → back to the card |
| RF-6.5 | Case view fields; no display name, tenant, membership, reviewer or time | `CaseDetails` |
| RF-6.6 | `canContinue = false` | Statement and "Return to sign in" only |
| RF-6.7 | `canContinue = isApprovedAndValid(now)` | "Continue to replace your authenticator" |
| RF-6.8 | `StartReplacementFromCaseUseCase`; approval not consumed | Wizard opens |
| RF-6.9 | `verifiedUntil` checked under lock; nothing persisted client-side | Continuation holder with idle timer; cleared on leave |
| RF-6.10 | `findOpenByReferenceHash` ignores closed cases → decoy | — |
| RF-7.1 | One code `auth.recovery.rejected` for code, one-time-code and continuation failures | `GenericFailureMessage` with try-again and return routes |
| RF-7.2 | Constant-work paths (D1, D6); timing harness | One rendering per state |
| RF-7.3 | No response discloses factor, count, case existence or state before a verified continuation | — |
| RF-8.1 | `AbuseThrottlePolicy` keys `RECOVERY_ADDRESS`, `RECOVERY_REFERENCE`, `RECOVERY_NETWORK`; `ThrottleStore` | — |
| RF-8.2 | Keys distinct from `ACCOUNT` / `NETWORK` **[001]** and from the reset keys **[004]**; no restriction consulted across families | — |
| RF-8.3 | Restricted → no case, no message, no code, no consumption; same 202 / generic answer | Same state |
| RF-8.4 | Wrong password → `ACCOUNT` + `NETWORK` **[001]** in addition; other failures → recovery keys only | — |
| RF-8.5 | Windows expire by time; no permanent flag exists | — |
| RF-8.6 | Per-address caps for re-sends, one-time codes and notifications, checked before enqueue; `NOTIFICATION_SUPPRESSED` audited | — |
| RF-8.7 | `RecoveryPolicyProperties` validation at startup | — |
| RF-9.1 | No membership, grant or tenant write; landing by plan 001 rules | `LandingRouter` **[001]** |
| RF-9.2 | No endpoint, command or tenant use case touches another identity's case, code or factor | "No administrator backdoor" notice |
| RF-9.3 | Only `AcknowledgeRevealUseCase` **[003]** changes the factor; no disable path exists | — |
| RF-9.4 | No credential or address write in any use case | No alternative channel offered |
| RF-9.5 | The password is required on every card even in session; `CurrentSessionPort` supplies the address and owner only | Session never skips a field |
| RF-10.1 | — | Reflow of `RecoveryPage` and all states; `SecurityStatements` shared |
| RF-10.2 | — | `shared/ui` targets, labels, focus |
| RF-10.3 | — | Breakpoint layout replaceable (S-03) |
| RF-11.1 | Audit adapter events in the effect's transaction | — |
| RF-11.2 | Audit payload shape; unmatched address never stored | Correlation id header |
| RF-11.3 | Verifier inputs, codes and references never logged; problem details exclude inputs | No secret in state or analytics |
| RF-11.4 | — | Holders in memory only; no storage write |
| RF-11.5 | Five templates plus the replacement variant through the outbox to the verified address; locale rules of §2.1 | — |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · One generic problem code and constant-work paths on every public entry.**
*Why.* RF-2.3, RF-4.3, RF-6.3 and RF-7 require that the answer, its shape and its timing not reveal whether an address has an account, a password was right, a code was valid or a case exists. Each use case therefore runs the same steps in every branch: normalisation, both throttles, one identity (or case) lookup, exactly one password verification — real for an eligible identity, against the dummy hash **[001]** otherwise — and, on path (a), one code comparison bounded by the configured code count (the identity's unused codes padded, or a dummy set of the same size); the continuation always creates a challenge row (D6). One problem code, `auth.recovery.rejected`, covers every failure class; the cause exists only in audit.
*Discarded.* Early return on an unknown address or a wrong password — a timing leak that tells an attacker when the password is right. Distinct problem codes with identical UI text — the network response leaks the class. A "no such case" answer — rejected by Q5.
*Covers.* RF-2.3, RF-4.3, RF-4.6, RF-6.3, RF-7.1–7.3, RNF-1, LC-1–LC-3, LC-10–LC-12, LC-22, LC-23.

**D2 · The recovery code is consumed under a row lock, in the same transaction that starts the replacement.**
*Why.* Q3: the code is spent the moment it verifies. Locking the identity's code rows (`FOR UPDATE`) before `RecoveryCodeSet.consume` makes two tabs with the same code yield exactly one success (LC-9); writing the consumption, the recovery transaction, the pending enrollment and the audit entry in one transaction means a crash cannot leave a consumed code with no replacement started, or a started replacement with an unconsumed code. The maintained encoder keeps the work factor of plan 001 D5.
*Discarded.* Consuming at replacement completion — rejected by Q3 (a code typed into a browser must not be usable again). Reserving the code — a second state and a window in which it is neither usable nor spent.
*Covers.* RF-2.2, RF-2.4, RF-2.5, RF-3.7, RNF-3, LC-2, LC-4, LC-9.

**D3 · The recovery transaction is a `LoginTransaction` [001] with the purpose `RECOVERY`, not a session; it completes through `CompleteSignInUseCase`.**
*Why.* RF-2.5 needs a limited identity state bound to identity, purpose, expiry and attempts, reachable only by its own steps; the login transaction already provides binding, expiry, locking, voiding, the `securityVersion` check and idempotency-keyed outcomes (plan 001 D1), so adding a purpose and one state gives every bound for free, as plan 004 D8 did for the forced change. Completing through `CompleteSignInUseCase` **[001]** makes the post-replacement landing exactly a sign-in (Q3, RF-3.5): the session is issued by the one port that issues sessions, and `LandingRouter` navigates by the server decision. Re-stamping the transaction's own `securityVersion` at completion lets it survive the version bump it caused while every other open transaction of the identity voids (LC-7).
*Discarded.* Issuing a session at code verification — a session before the new factor is proven, contrary to RNF-8. A dedicated recovery-session object — a second limited state to secure. Returning to the sign-in screen after the replacement — the alternative of Q3, not chosen.
*Covers.* RF-2.5, RF-3.5, RF-3.7, RF-3.8, RNF-8, LC-4–LC-7, LC-29.

**D4 · `PendingEnrollment` [003] gains a polymorphic owner and a `startedBy`; plan 003's three steps are reused unchanged, and completion delegates to a recovery hook.**
*Why.* RF-3.1 … RF-3.3 are plan 003's RF-9.3 … RF-9.6 verbatim; building them twice would drift. Plan 003 D9 binds a pending enrollment to the session that started it; a public recovery has no session, so the owner becomes "a session or a recovery transaction" and every ownership check and the cleanup job compare that owner instead. `startedBy` lets `AcknowledgeRevealUseCase` **[003]** call `RecoveryReplacementCompletion` after its atomic swap — revocation scope, case completion, the notification variant and, for a public entry, the sign-in completion — without plan 003 knowing the recovery rules.
*Discarded.* A separate recovery wizard and use cases — drift of the three-step rules and of "one reveal". Binding the pending enrollment to nothing for a public entry — anyone with the pending id could continue it. Making plan 003's acknowledge aware of cases and landing — the wrong owner for those rules.
*Covers.* RF-2.6, RF-3.1–3.7, RNF-10, LC-8, LC-28, LC-32.

**D5 · The case is its own aggregate with a hashed reference, one open case per identity by a partial unique index, and a published closure port.**
*Why.* RF-5.1 needs states, times and actors that persist across sessions and browsers; keeping them off the identity row keeps FR-IDN-001 literal (plan 002 D7's reasoning). The reference is a 256-bit random capability that grants nothing by itself, so a fast cryptographic hash suffices (plan 002 D3) and nothing usable is at rest. "At most one open case" (Q6) is a database fact through the partial unique index, so a race between two requests cannot open two cases (LC-14). The closure triggers of RF-5.6 live in three other features; a port published by `identity.application` and called synchronously in their transactions keeps "a consumed code or a completed replacement closes the case" atomic with the event that caused it.
*Discarded.* A flag on the identity row — FR-IDN-001. A domain event handled asynchronously — a window in which a case is open after the identity recovered. Several open cases — rejected by Q6.
*Covers.* RF-4.4, RF-4.5, RF-4.8, RF-5.1, RF-5.6, RF-5.7, RF-5.10, RNF-4, RNF-5, LC-14, LC-17–LC-21.

**D6 · A continuation is a decoy-capable challenge row with a one-time code hashed by the maintained encoder.**
*Why.* Q5 makes the one-time code the "separate verification" of FR-IDN-047; a short code must be bounded in lifetime and attempts and stored as a work-factored hash because it is guessable offline within its lifetime. Creating the row for every continuation — with no case when the reference and password did not match — makes RF-6.3 structural: the response, the write and the timing are identical, and a code submitted against a decoy fails exactly as a wrong code (D1). The `verifiedUntil` window on the same row gives RF-6.9 without a second state, and the idempotency outcomes give LC-31-style safety for the two state-changing submissions.
*Discarded.* Returning the continuation token only when the case matched — reveals the match. A stateless signed token — cannot be voided, cannot count attempts. Password only or mailbox only — the alternatives of Q5, not chosen.
*Covers.* RF-6.2–6.4, RF-6.9, RF-6.10, RF-7.2, RNF-1, RNF-3, LC-22–LC-24, LC-30.

**D7 · An approval is a case state with a validity, consumed only by the completed replacement; every entry needs a fresh continuation.**
*Why.* D-b: an abandoned replacement must not cost the person their approval, so `APPROVED` stays until RF-3.4 marks the case `COMPLETED` or the validity ends. Because each entry passes the full continuation (reference, password, one-time code), an approval that lives longer than one attempt is not a weaker proof. `StartReplacementFromCaseUseCase` is the only reader of `APPROVED`, which is what makes RF-5.8 ("grants exactly one thing") checkable.
*Discarded.* A single-use approval token delivered by email — a second secret and a lost approval after any abandoned step. Consuming the approval at the first start — rejected by D-b.
*Covers.* RF-5.8, RF-6.7, RF-6.8, LC-26–LC-29.

**D8 · The review is an internal command with an actor; expiry is a durable job.**
*Why.* Scope §2 forbids a customer-facing operator surface and FR-IDN-049 forbids any tenant-side action on another identity's factor; the only lawful caller is the platform's audited mechanism (A-47). The command exists so the behaviour — transitions, actor, validity, messages, audit — is implemented and tested now, while the invocation mechanism stays a placeholder that the operator path decides, exactly as plan 004 D10 did for the compromise response. Expiry of cases, approvals, continuations and recovery transactions is enforced on access and swept by the existing durable-jobs facility (plan 002 D9, plan 003 D10) so nothing lingers.
*Discarded.* An administrative REST endpoint — an operator surface in the product. Automatic approval after a delay — rejected by RNF-9. Sweep only — an expired approval could act between sweeps.
*Covers.* RF-5.1–5.5, RF-9.2, RNF-9, LC-17, LC-27.

**D9 · Recovery throttling reuses `AbuseThrottlePolicy` [001] with three keys, counts a wrong password on the sign-in keys as well (D-a), and caps messages per address.**
*Why.* The policy is pure and already tested; `RECOVERY_ADDRESS`, `RECOVERY_REFERENCE` and `RECOVERY_NETWORK` make recovery independent of sign-in and reset in both directions (RF-8.2); the reference key slows guessing of a specific case. A wrong password on a recovery card is a password guess, so it is written on `ACCOUNT` / `NETWORK` **[001]** exactly as the forced-change screen does (plan 004 D11), and nothing else of recovery touches those keys. Message caps bound re-sends, one-time codes and notifications per address so a flood cannot exhaust the channel (RF-8.6). Restriction means "no case, no message, no code, no consumption, same answer", never a different response.
*Discarded.* Sharing keys with sign-in — a recovery flood would throttle sign-in. Not counting the wrong password — unlimited password guessing through a public form. Edge rate limiting only — blind to the address and reference dimensions.
*Covers.* RF-8.1–8.6, RNF-2, RNF-5, LC-3, LC-11, LC-15, LC-16.

**D10 · Session revocation goes through `RevokeSessionsPort` [004] with `except = none` for a public entry and `except = owningSessionId` for a session entry, plus the `securityVersion` bump.**
*Why.* Q3 and A-39: from a public entry there is no session to keep, so every session and refresh family ends and the person lands with a fresh one; from R41 or R38 the acting session continues as plan 003 RF-9.7 states. The version bump (plan 004 D7) voids every other open login transaction at its next use and lets the session service refuse already-issued credentials before the session rows are read. Run-scoped authority is untouched (FR-IDN-067).
*Discarded.* Reimplementing revocation — a foreign repository. Keeping the recovery transaction's browser signed in through some other means — there is no session until `CompleteSignInUseCase` issues one.
*Covers.* RF-3.4, RF-3.6, RNF-10, RNF-11, LC-5, LC-7, LC-8.

**D11 · Five outbox templates plus the recovery variant of `authenticator-replaced`; review-originated messages use the locale stored on the case.**
*Why.* RF-2.8, RF-4.4, RF-5.3, RF-5.4, RF-5.6, RF-6.2 and RF-3.4 fix the statements each message carries; the outbox gives atomicity with the effect (plan 002 D1) and keeps provider latency out of the request (RNF-1, LC-33). Messages triggered by a request take its locale header (plan 002 D15); an approval or a closure has no request, so the case records the locale of the request that opened it — the person's own choice — rather than a configured default. The replacement notification states that the swap came through recovery so the holder can tell it from an R41 replacement.
*Discarded.* Synchronous send — timing and failure leaks. A configured default language for review-originated messages — the person already told us their language. A separate "case completed" message — the replacement notification covers it.
*Covers.* RF-2.8, RF-3.4, RF-4.4, RF-5.3–5.6, RF-6.2, RF-11.5, RNF-12, RNF-14, LC-33.

**D12 · The two state-changing continuation submissions carry an idempotency key with a recorded outcome [001 D1]; the enrollment steps inherit plan 003 D13.**
*Why.* F-X-02: a reload during the one-time-code step or at "Continue to replace your authenticator" must not count a second attempt or start a second replacement; the challenge row records the outcome per key and the use case returns it before any other work. The request and the first continuation step need no key: a repeated request re-sends the same reference (Q6) and a repeated continuation creates a new challenge, both harmless.
*Discarded.* Generic POST retry — forbidden. No key — a lost answer at the start of the replacement could open two pending enrollments.
*Covers.* RF-6.4, RF-6.8, RNF-3, LC-29.

**D13 · One public route with session detection; holders in memory only; the wizard of plan 003 reused.**
*Why.* N05 is one screen with two tabs (RF-1.1); the in-session variant differs only in the read-only address and the return destination (RF-1.4), so one page reading the existing session query **[001]** keeps every statement identical by construction. The transaction holder **[001]** already carries a limited identity state and clears on leave; extending it with the recovery purpose gives RF-2.5 and RF-3.7 the same discipline as the challenge. The continuation holder dies with the route and with its idle timer (RF-6.9). Nothing enters storage (RF-11.4).
*Discarded.* Separate public and in-session pages — drift. sessionStorage for the continuation — F-X-03. Passing the reference or the transaction in the URL — leaks into history and logs.
*Covers.* RF-1.1, RF-1.4, RF-2.5, RF-3.7, RF-6.9, RF-11.4, RNF-4, LC-30, LC-32.

**D14 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id [001 D12].**
*Why.* F-X-03; field paths give RF-1.7 and RF-4.2 without parsing text; one code per class keeps RF-7.1 generic; the void code of the continuation carries no cause; the correlation id joins the audit record to the user's report.
*Discarded.* Ad-hoc error bodies.
*Covers.* RF-1.7, RF-4.2, RF-6.4, RF-7.1, RF-11.3.

**D15 · Every configured number is one validated `RecoveryPolicyProperties` record that fails startup.**
*Why.* P13 and FR-X-046; one record makes the full set visible and testable; validation at bind time means the product never runs with a missing or unbounded lifetime, validity, attempt limit, threshold, window or cap.
*Discarded.* Defaults in code — forbidden. Lazy validation — the first request would fail instead of startup.
*Covers.* RF-8.7, RNF-6, RNF-7.

**D16 · No dedicated mobile page; N05 reflows, and text comes from feature-scoped i18n bundles [002 D15, D16].**
*Why.* M18 is the drawn counterpart (S-03); FR-X-055 requires AA reflow at every other width; `SecurityStatements` shared across widths keeps every statement present (RNF-15). The server sends codes only; the locale header and the stored case locale select template languages.
*Discarded.* Separate mobile components — drift. Server-rendered messages — ties language to the request.
*Covers.* RF-1.10, RF-10.1–10.3, RNF-13, RNF-14, RNF-15, LC-31.

**D17 · Audit is appended through the `audit` context port in the effect's transaction [001 D11].**
*Why.* An audit entry that commits with the effect can neither be lost nor describe something that did not happen; review transitions carry the actor so a case can be reconstructed end to end. The payload carries identifiers, outcome kinds and the correlation id only.
*Discarded.* The logging framework as audit — not append-only, prone to leakage. An asynchronous outbox for audit — a window in which an effect exists without its record.
*Covers.* RF-11.1, RF-11.2, RNF-4.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure domain: `RecoveryCase` state machine and causes, `ContinuationChallenge`, `LoginTransaction` recovery purpose, `PendingEnrollment` owner comparison, `AbuseThrottlePolicy` with the recovery keys, masked-address rendering. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: code-row locks, partial unique index, decoy rows, outbox rows, audit rows, revocation port calls, closure port calls from the three producers, job sweep, configuration binding, migrations. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of identity, factor, count, case or secret data in every response; session-or-transaction authorisation of the enrollment endpoints. |
| Timing comparability | JUnit 5 statistical harness (test-only) | N samples per input class for code verification, request and continuation; distributions overlap within a configured tolerance. |
| Concurrency | JUnit 5 with real PostgreSQL | Same code in two tabs, two requests for one address, two continuations of one approved case, acknowledge racing a sign-in challenge. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| Module boundary | ArchUnit (**test-only**) | The application module depends on no Spring or persistence class and imports no tenant, membership, grant, approval, run or intent package. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page states, tabs, field validation, holders cleared on leave and on the idle timer, handoffs, statements present at every width, nothing in storage. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths with a fake email port exposing the reference and the one-time code; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Tab operation, focus order, announcements without focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RF-1.9, RF-9.2 | FE unit, e2e | `recoveryPage.rendersTabsCardsNoticeAndRoutes` (desktop, tablet, mobile); `recoveryPage.rendersNoHelp` | 1, 13 |
| RF-1.2, RF-1.5 | FE e2e | `entries.signInChallengeReplaceDialogAndSettingsRouteToRecover`; `tenantSuspended.hasNoRecoveryRoute` | 1 |
| RF-1.3 | FE unit, BE integration | `recoveryPage.prefillsAddressFromVoidedChallengeOnly`; `recovery.acceptsNoLoginTransaction` | 5 |
| RF-1.4, RF-9.5, LC-8 | API contract, FE unit | `recovery.inSessionUsesSessionIdentityAndRefusesAddress`; `recovery.inSessionStillRequiresPassword`; `recoveryPage.sessionVariantReadOnlyAddress` | 5 |
| RF-1.6, RF-2.7, RF-7.3, RNF-8 | API contract | `recoveryResponses.exposeNoIdentityFactorCountOrCase` | 3, 9 |
| RF-1.7, RF-4.2 | FE unit, API contract | `recoveryPage.blocksEmptyOrMalformedFields`; `recoveryApi.fieldPathProblems`; `request.refusesUntickedAcknowledgement` | 2 |
| RF-1.8 | BE integration | `recovery.normalisesAddress` | 2 |
| RF-1.10, RNF-14, LC-31 | FE e2e | `languageSwitch.onRecoveryKeepsEverything` | 13 |
| RF-2.1 | FE unit | `recoveryCodeCard.fieldsPasteAndStatement` | 1 |
| RF-2.2, RF-2.3, RNF-1, LC-1–LC-3 | API contract + timing, BE integration | `code.failureIsGenericAcrossClasses` (unknown address, wrong password, consumed code, no factor, restricted, erasing, throttled); `code.timingComparable`; `code.wrongPasswordConsumesNothing` | 3, 5 |
| RF-2.4, RNF-3, LC-2, LC-9 | BE integration, Concurrency | `code.consumedAtVerificationNeverAcceptedAgain`; `code.twoTabsOneSuccess` | 6 |
| RF-2.5, RF-3.5, LC-5 | BE integration, API contract | `code.publicEntryOpensRecoveryTransactionNoSession`; `recoveryTransaction.authorisesOnlyItsEnrollmentAndAbandon`; `acknowledge.recoveryOwnedCompletesSignInAndLands` | 5, 7 |
| RF-2.6, RF-3.6, LC-8 | BE integration | `code.sessionEntryStartsReplacementWithoutStepUp`; `acknowledge.sessionOwnedKeepsCurrentRevokesOthers` | 7 |
| RF-2.8, RF-11.5 | BE integration | `notifications.codeUsedEnqueuedToVerifiedAddress`; `templates.fiveMessagesAndReplacedVariantCarryStatementsNoTime` | 12 |
| RF-3.1, RF-3.2, LC-32 | BE integration | `replacement.entersPlan003StepsWithRecoveryOwner`; `signIn.duringPendingRecoveryUsesOldFactor` **[003]** | 7 |
| RF-3.3, RF-3.4, RNF-10, RNF-11, LC-28 | BE integration | `completion.swapsRevokesClosesCaseNotifies` (code source and case source) | 7 |
| RF-3.7, LC-4, LC-26 | BE integration, FE unit | `replacement.abandonedLeavesOldFactorCodeConsumedCaseApproved`; `wizard.leaveDuringRecoveryAbandonsTransaction` | 7 |
| RF-3.8, RF-9.1, RF-9.3, RF-9.4, LC-6 | BE integration | `recovery.touchesOnlyIdentityTablesAndOutbox` (password, address, membership, suspension, status, runs unchanged); `recovery.ownerOfSuspendedTenantLandsOnChooserUnlifted` | 9 |
| RF-3.9 | FE e2e (tenant-entry stub) | `recoveredOwner.gateDerivesFromNewFactorAtTenantEntry` | 9 |
| RF-4.1, RF-4.3 | FE unit, e2e | `requestCard.rendersStatementsCheckboxAndReceivedState` | 1, 13 |
| RF-4.3, RF-4.6, RNF-1, LC-10–LC-12 | API contract + timing | `request.responseIdenticalAcrossClasses` (eligible, wrong password, unverified, restricted, erasing, unknown, throttled); `request.timingComparable` | 3 |
| RF-4.4, RF-4.8, LC-13 | BE integration | `request.eligibleCreatesCaseHashedReferenceAndMessage`; `request.identityWithoutFactorStillOpensCase` | 8 |
| RF-4.5, LC-14 | BE integration, Concurrency | `request.openCaseResendsSameReference`; `request.twoRequestsOneCase` | 8 |
| RF-4.7, RF-5.9 | BE integration | `request.changesNothingElse`; `case.invisibleToTenantSurfaces` | 8, 9 |
| RF-5.1, RF-5.10 | BE unit, integration | `recoveryCase.stateMachineWithTimesAndActors`; `recoveryCase.holdsNoSecretOrTenantContent` | 8 |
| RF-5.2, RF-9.2 | API contract | `noTenantSurfaceOrEndpointTouchesACase`; `noAdminFactorPathExists` | 8, 9 |
| RF-5.3, RF-5.4, RF-5.5 | BE integration | `review.approveRecordsActorValidityAndNotifies`; `review.rejectNotifiesClosed`; `messages.promiseNoCompletionTime` | 8, 12 |
| RF-5.6, LC-18, LC-19, LC-21 | BE integration | `closure.onChallengeCodeUse`; `closure.onReplacementFromR41`; `closure.onRestrictedOrErasing` | 8 |
| RF-5.7, LC-20 | BE integration | `passwordReset.leavesCaseUntouched` | 8 |
| RF-5.8, RF-6.7, RF-6.8, LC-26 | BE integration | `approval.grantsReplacementStartOnly`; `approval.survivesAbandonedReplacementUntilValidity` | 6, 8 |
| RF-6.1 | FE unit | `continueCard.fieldsAndStatement` | 1 |
| RF-6.2, RF-6.3, RNF-1, LC-22, LC-23 | API contract + timing, BE integration | `continue.responseIdenticalAcrossClasses` (open case, closed case, unknown reference, wrong password, throttled); `continue.timingComparable`; `continue.decoyRowForNonMatch` | 3 |
| RF-6.4, LC-24 | BE integration, FE unit | `continuation.wrongOrExpiredCodeGenericThenVoid`; `continueCard.voidReturnsToCard` | 2, 6 |
| RF-6.5, RF-6.6, LC-25 | API contract, FE unit | `caseView.fieldsAndNothingMore`; `caseDetails.underReviewOffersReturnOnly` | 5 |
| RF-6.9, LC-30 | FE unit, BE integration | `continuationHolder.clearsOnIdleAndLeaveNeverStorage`; `continuation.verifiedWindowExpires` | 5 |
| RF-6.10, LC-17, LC-27 | BE integration | `continue.closedExpiredOrCompletedReferenceIsDecoy`; `expiryJob.expiresCasesAndApprovalsWithMessage` | 6, 8 |
| RF-7.1 | API contract, FE unit | `rejected.oneCodeForEveryCause`; `genericFailure.tryAgainAndReturnOnly` | 3 |
| RF-7.2 | Timing | the three `timingComparable` harnesses above | 3 |
| RF-8.1, RF-8.3 | BE unit, integration | `throttlePolicy.recoveryKeysProgressiveSteps`; `recovery.restrictedIssuesNothingSameAnswer` | 4 |
| RF-8.2 | BE integration | `throttle.recoverySignInAndResetIndependentExceptPasswordRule` | 4 |
| RF-8.4, LC-3, LC-11, LC-23 | BE integration | `throttle.wrongPasswordCountsOnSignInKeys`; `throttle.wrongCodeOrReferenceCountsOnRecoveryKeysOnly` | 4 |
| RF-8.5, RNF-2, LC-16 | BE integration | `throttle.recoveryWindowLiftsAndSignInNeverRestricted` | 4 |
| RF-8.6, RNF-5, LC-15 | BE integration | `caps.resendsCodesAndNotificationsBoundedPerAddress` | 4 |
| RF-8.7, RNF-6 | Startup | `recoveryPolicy.startupFailsOnAbsentOrUnboundedValue` (one case per property) | 10 |
| RF-10.1–10.3, RNF-13, RNF-15 | FE e2e + axe, manual assistive | `a11y.recovery.{en,es,fr}.{desktop,tablet,mobile}` (all states); manual NVDA / VoiceOver checklist | 13 |
| RF-11.1, RF-11.2 | BE integration | `audit.entryPerEvent`; `audit.containsNoSecretOrUnmatchedAddress` | 11 |
| RF-11.3, RF-11.4, RNF-4 | BE integration, FE unit | `logs.containNoPasswordCodeOrReference`; `storage.containsNothingFromFeature` | 11 |
| RNF-7 | BE integration | `failClosed.unknownConsumedExpiredDecoyVoidedOrUnconfiguredRefuses` | 2, 10 |
| RNF-9 | BE integration | `review.noAutomaticApprovalExists` | 8 |
| RNF-12, LC-33 | BE integration | `request.outboxFailureNotVisibleInResponse`; `notification.failureNeverRevertsEffect` | 12 |
| LC-7 | BE integration | `completion.revokesOtherDeviceSessionLeavesRunAuthority` | 7 |
| LC-29 | Concurrency | `approvedCase.twoReplacementsOneCompletes` | 6 |
| LC-34 | API contract | `tenantAdmin.hasNoPathToMemberMfa` | 9 |
| LC-35 | FE unit | `recoveryPage.offersNoAlternativeChannel` | 9 |
| US-10 | all adversarial cases above | enumeration, code-alone / reference-alone / mailbox-alone / session-alone, reuse, backdoor suites | 1 |

### 5.3 Not tested here
Sign-in, the challenge and landing mechanics (plan 001), signup and verification (002), the enrollment steps, the reveal and the step-up (003), password reset and the forced change (004), session issue, refresh, expiry and listing (006, 007), the chooser contents (013), tenant suspension (014), the operator mechanism that calls the review command (A-47). Their stubs return fixed answers; `RevokeSessionsPort`, `CurrentSessionPort` and `IssueSessionPort` are verified by recording stubs and, in the integration suite, by the session service's own test double.

### 5.4 Exit criteria
Specification §7 items 1–14 map to the "Proves criterion" column; item 14 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-47 verified recovery process | `ReviewRecoveryCaseCommand` with three transitions, each audited with an actor; invoked by an environment-only administrative command with actor id and case id; approval validity and case lifetime in `RecoveryPolicyProperties`. | Rebind the command to the decided operator path and evidence procedure; move the two values to the configuration catalogue of A-11; the use cases are unchanged. |
| A-48 recovery case and one-time code records | `recovery_case` and `recovery_continuation` as in §2.1; closure triggers through `RecoveryCaseClosurePort` called synchronously by 001, 003, path (a) and the A-27 status transitions. | Rename or re-home the records; if closure becomes an event, keep the handler in the producer's transaction. |
| A-27 identity statuses | `isResetEligible()` **[004]** = `ACTIVE` and verified; `UNVERIFIED`, `RESTRICTED`, `ERASING` ineligible; a transition to `RESTRICTED` / `ERASING` calls the closure port. | Change the predicate and the transition hook only. |
| A-30 notification set | Four recovery messages added (code used, case opened, approved, closed) with templates per S-14. | Add or remove templates; the outbox calls are the only place. |
| A-39 revocation scope | `RevokeSessionsPort.revokeAll(identity, except)`: `except = none` for a public recovery, the acting session for an in-session one; run-scoped authority untouched. | Extend the port with the decided scope. |
| A-35 persistence tooling | Adapters and migrations against the placeholder API and tool. | Rename base types and migration format; no use case changes. |
| S-38 N05 fields and states (D02 `423:3542`) | Password fields on all three cards, received state, one-time-code step, case-details states, generic failure, in-session variant, tab split — built as the specification describes. | Adjust the page states and copy only. |
| S-14 templates | The D04 copy (`437:4431`) carrying the statements of RF-2.8, RF-4.4, RF-5.3, RF-5.4, RF-5.6, RF-6.2 and the recovery variant of RF-3.4 in EN / ES / FR. | None. |
| S-03 mobile counterparts | Reflow of N05 and all its states under the desktop rules. | Replace the breakpoint layout. |
| S-17 "Help" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |

No new register item is raised by this plan: every element is built with the fixed stack (P10), and the two questions this feature could not answer from the design (the review mechanism and the records) were already recorded by the specification as A-47 and A-48.

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.10) | `RecoveryPage`, handoffs, `RecoveryController` problems, `EmailAddress`, `SecurityStatements` | D1, D13, D14, D16 | RF-1 rows |
| RF-2 (2.1–2.8) | `VerifyRecoveryCodeForReplacementUseCase`, `RecoveryCodeSet`, `LoginTransaction(RECOVERY)`, `RecoveryCodeCard`, template `recovery-code-used` | D1, D2, D3, D11 | RF-2 rows |
| RF-3 (3.1–3.9) | `StartEnrollmentUseCase` / `AcknowledgeRevealUseCase` **[003]**, `PendingEnrollment` owner, `RecoveryReplacementCompletion`, `CompleteSignInUseCase` **[001]**, `RevokeSessionsPort`, `EnrollmentWizard` | D3, D4, D10, D11 | RF-3 rows |
| RF-4 (4.1–4.8) | `RequestRecoveryCaseUseCase`, `RecoveryCase`, `TokenGenerator` / `TokenHasher`, `RecoveryRequestCard`, template `recovery-case-opened` | D1, D5, D9, D11 | RF-4 rows |
| RF-5 (5.1–5.10) | `RecoveryCase`, `ReviewRecoveryCaseCommand`, `CloseRecoveryCaseService` / `RecoveryCaseClosurePort`, `ExpireRecoveryCasesJob`, review entry, templates | D5, D7, D8, D11 | RF-5 rows |
| RF-6 (6.1–6.10) | `ContinueRecoveryCaseUseCase`, `VerifyContinuationCodeUseCase`, `StartReplacementFromCaseUseCase`, `ContinuationChallenge`, `ContinueCaseCard`, `CaseDetails`, continuation holder | D1, D6, D7, D12, D13 | RF-6 rows |
| RF-7 (7.1–7.3) | one problem code, constant-work paths, `GenericFailureMessage` | D1, D6, D14 | RF-7 rows |
| RF-8 (8.1–8.7) | `AbuseThrottlePolicy` recovery keys, `ThrottleStore`, caps, `RecoveryPolicyProperties` | D9, D15 | RF-8 rows |
| RF-9 (9.1–9.5) | use-case scope, no endpoint or command for another identity, password required in session | D3, D4, D8, D10 | RF-9 rows |
| RF-10 (10.1–10.3) | responsive page, `shared/ui` | D16 | RF-10 row |
| RF-11 (11.1–11.5) | audit adapter, redaction, holders, templates | D11, D14, D17 | RF-11 rows |
| RNF-1 | D1 paths, decoy continuation, one code | D1, D6 | timing + contract |
| RNF-2 | recovery keys, bounded windows | D9 | RF-8.5 row |
| RNF-3 | code-row lock, challenge lock, idempotency outcomes | D2, D6, D12 | RF-2.4, LC-29 rows |
| RNF-4 | hashes only, holders, audit payload, redaction | D5, D6, D13, D17 | RF-4.8, RF-11 rows |
| RNF-5 | one open case, caps | D5, D9 | RF-8.6 row |
| RNF-6 | `RecoveryPolicyProperties` | D15 | startup |
| RNF-7 | check-on-access, locks, decoys, config validation | D2, D6, D15 | `failClosed` |
| RNF-8 | recovery transaction, no C10, responses | D3 | RF-1.6 row |
| RNF-9 | internal review command, no automatic approval | D8 | RNF-9 row |
| RNF-10 | 003 swap, completion hook | D4, D10 | RF-3.3 row |
| RNF-11 | `RevokeSessionsPort` + version bump in the same transaction | D10 | RF-3.3 row |
| RNF-12 | outbox templates, email port | D11 | RNF-12 row |
| RNF-13, RNF-15 | `shared/ui`, reflow | D16 | RF-10 row |
| RNF-14 | i18n bundles, locale rules | D11, D16 | RF-1.10 row |
| LC-1 … LC-35 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35) |

Self-check: every RF criterion (78), every RNF (15) and every limit case (35) of `specControlledAccountRecovery.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
