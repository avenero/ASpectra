# Tasks 006 — Sessions, refresh and expiry

**Feature** Identity session lifetime, coordinated refresh, expiry and re-authentication · **Tasks version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Derived from** `specSessionsRefreshExpiry.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n; decisions Q1–Q6; derived rules D-a, D-b) and `planSessionsRefreshExpiry.md` v0.1 (modules §2, trace §3, decisions D1–D19, tests §5, register items §6) · **Governed by** `constitution.md` v1.8.

This document is the DO. Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2 so each task can be checked against the test matrix. Tasks are sized at 20–30 minutes for one developer and listed in dependency order; "after" names the tasks that must be complete first when they are not simply the preceding ones. A later task may assume everything above it. Phase F (front end) depends only on phase A and may run alongside phases B–E.

Elements tagged **[001]**, **[002]**, **[003]** or **[004]** are owned by plans 001 (Login), 002 (Signup and verification), 003 (Enrollment) and 004 (Password reset); they are extended here, never duplicated. This feature **implements** the three ports those plans declared against "the existing session service" — `IssueSessionPort` **[001]**, `CurrentSessionPort` **[003]**, `RevokeSessionsPort` **[004]** — so their stubs are replaced by real adapters in phase C and nothing in those plans changes. If an owning plan is not built yet, the element is built here in the shape that plan gives it. Test-only tools (plan §5.1; none enters the product build): JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 001's, 002's, 003's and 004's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing.

---

## A · Foundations

- [ ] **T-01 · Session configuration record** — `SessionPolicy` immutable record and `SessionPolicyProperties` (`@ConfigurationProperties` + `@Validated`): idle deadline, absolute deadline, access-credential lifetime, save-notice threshold, safe-read retry bounds (attempts, backoff ceiling), ended-session retention, family retention, sweep cadence and batch size; each present, positive and bounded by a configured ceiling; cross-field rules idle ≤ absolute and credential lifetime < idle. The client holds no number of its own (D4, D17). *(plan §2.1 `SessionPolicy`, `SessionPolicyProperties`, D17; RF-2.1, RF-4.6, RF-9.1, RNF-5)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed, set non-positive or set above its ceiling, and for each of the two cross-field rules violated (one failing case each).

- [ ] **T-02 · Migration: `identity_session`** — columns `id`, `identity_id`, `created_at`, `last_activity_at`, `idle_deadline`, `absolute_deadline`, `security_version`, `client_context`, `state`, `end_cause`, `ended_at`; index on `(identity_id, state)`; `identity_id` cascades on identity delete. *(plan §2.1 Persistence, D1; RF-1.5, RF-2.5, RF-2.6)* — after T-01
  **Done when:** the migration applies after plans 001–004's on an empty database and on one holding identities; an `ACTIVE` row can be selected by identity through the index.

- [ ] **T-03 · Migration: `refresh_family` and `refresh_credential`** — `refresh_family` (`id`, `session_id` unique → cascade delete, `generation`, `state`, `end_cause`, `ended_at`); `refresh_credential` (`family_id` → cascade delete, `generation`, `hash` unique, `issued_at`, `retired_at`); the current generation is the row with `retired_at IS NULL`. *(plan §2.1 Persistence, D2, D5; RF-3.1, RF-3.5, RNF-2)* — after T-02
  **Done when:** the migration applies; two credentials with one hash are rejected; deleting a session deletes its family and credentials; a partial index or constraint allows at most one unretired credential per family.

- [ ] **T-04 · Problem-code catalogue** — register `auth.credential.expired` (401, eligible for refresh), `auth.session.invalid` (401, definitive, no cause member), `auth.csrf.invalid` (403), `access.denied-scope` (403, emitted by `access`); reuse `auth.sign-in.failed` **[001]** for the re-authentication. Each carries code, title, correlation id and nothing else. *(plan §2.1 Security filter, `SessionController`, D6, D19; RF-2.6, RF-4.1, RF-5.5, RF-7.3, RF-11.3)* — after T-01
  **Done when:** a serialisation test renders one problem per code and finds only the permitted members; `auth.session.invalid` has no member able to carry a cause.

- [ ] **T-05 · Outbox template registered** — template id `session-credential-reuse` with the outbox message type **[002]** in EN, ES and FR: a session was ended because its credential was presented twice, when, what to do if unexpected (change the password, review the sessions list); no credential, no link that grants access (D04 copy `437:4431`, S-14). *(plan §2.1 Outbox and email adapter, D16; RF-11.4, RNF-9)* — after T-01
  **Done when:** rendering the template in the three locales with sample variables yields a non-empty subject and body containing the event time, and the template-variable guard rejects any variable named like a credential, token or URL.

- [ ] **T-06 · `LoginTransaction` extension [001]** — add purpose `REAUTHENTICATION` and the field `resumesSessionId`; every other rule (identity binding, expiry, attempts, `securityVersion` check, idempotency outcomes, the `FORCED_CHANGE_REQUIRED` state **[004]**) is unchanged for the new purpose. *(plan §2.1 `LoginTransaction`, D8; RF-8.1, RF-8.4, RF-8.6)* — after T-01
  **Done when:** unit tests show a `REAUTHENTICATION` transaction voids on attempt exhaustion, expiry and version mismatch exactly as a `SIGN_IN` one, and carries the ended session id.

## B · Domain (pure, no Spring)

- [ ] **T-07 · `Session` aggregate — fields, `isValid`, `remaining`** — id, identity id, `createdAt`, `lastActivityAt`, `idleDeadline`, `absoluteDeadline`, `securityVersionAtIssue`, `clientContext`, state `ACTIVE → ENDED(cause)` with causes `IDLE`, `ABSOLUTE`, `VERSION`, `REVOKED`, `LOGOUT`, `REUSE`, `IDENTITY_STATUS`; `isValid(now, currentVersion)` false when ended, when either deadline has passed or when the version is stale; `remaining(now)` = time to the nearer deadline; the clock is always an argument. *(plan §2.1 `Session`, D1; RF-1.5, RF-2.1, RF-2.4, RF-2.6, RF-2.7, LC-27)* — after T-01
  **Done when:** unit tests with a fixed clock show `isValid` false for each of the three reasons and true otherwise, and `remaining` equals the smaller of the two distances.

- [ ] **T-08 · `Session.touch` and `Session.end`** — `touch(now, policy)` moves `lastActivityAt` and `idleDeadline` forward (bounded by the absolute deadline) and never moves `absoluteDeadline`; `end(cause, now)` is idempotent and keeps the first cause. *(plan §2.1 `Session`, D1, D3; RF-2.2, RF-2.4, RF-2.5, LC-4, LC-21)* — after T-07
  **Done when:** unit tests show `touch` advances only the idle deadline, a `touch` past the absolute deadline cannot extend beyond it, and a second `end` leaves the first cause and time.

- [ ] **T-09 · `RefreshFamily`** — family id, session id, `generation`, current hash, retired hashes with retirement time, state `LIVE → ENDED(cause)`; `rotate(presentedHash, now)` → `Rotated(nextGeneration)` for the current hash, `Reuse` for a retired one, `Unknown` otherwise; `end(cause, now)`; the plaintext credential never enters the aggregate. *(plan §2.1 `RefreshFamily`, D5; RF-3.1, RF-3.3, RF-3.5, RNF-2, RNF-3, LC-1, LC-2, LC-3)* — after T-01
  **Done when:** unit tests show the three outcomes, that the rotated hash is immediately retired, that `rotate` on an ended family answers `Unknown`, and a reflection test finds no field or accessor able to hold a plaintext credential.

- [ ] **T-10 · `ActivityClass` rule** — pure function `(operationClass, backgroundMarker) → DELIBERATE | BACKGROUND`: consequential writes and commands always `DELIBERATE`; reads `DELIBERATE` unless marked background; the refresh operation, the status query and realtime traffic structurally `BACKGROUND`; a marker can lower, never raise. *(plan §2.1 `ActivityClass`, D3; RF-2.2, RF-2.3, RF-3.2, LC-21, LC-28)*
  **Done when:** `activityClass.truthTable` passes over every (operation class × marker) pair, including the case where a "deliberate" marker on a structurally background operation is ignored.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-24…T-32); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-11 · Port interfaces** — new: `SessionRepository` (`findForUpdate`, `findById`, `findActiveByIdentity`, `save`), `RefreshFamilyRepository` (`findForUpdateBySession`, `findByCredentialHash`, `save`), `AccessCredentialCodec` (`mint`, `verify`), `RefreshCredentialGenerator`, `RefreshCredentialHasher`, `SessionAssertionPort` (published to `access`), `ActivityMarkerPort`; declared here as implemented: `IssueSessionPort` **[001]**, `CurrentSessionPort` **[003]**, `RevokeSessionsPort` **[004]** with `revokeAll(identityId, exceptSessionId, cause)`. Reused: `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `PasswordVerifier`, `TotpVerifier`, `Clock`, `AuditPort` **[001]**; `JobScheduler`, `SecurityNotificationPort`, `TokenGenerator`, `TokenHasher` **[002]**. No port reaches a project, grant, approval, run or intent table. *(plan §2.1 Ports; RF-1.1, RF-1.6, RF-10.1, RF-10.3, RNF-1, RNF-10)* — after T-06…T-10
  **Done when:** the application module compiles with no dependency on Spring or persistence classes, enforced by an ArchUnit module-boundary test that also forbids any import from the run, approval, project or grant packages.

- [ ] **T-12 · `IssueSessionUseCase`** — implements `IssueSessionPort`: creates the `Session` with deadlines from `SessionPolicy`, the identity's current `securityVersion` and the request's client context; creates its `RefreshFamily` at generation 1 with a fresh credential (hash stored, plaintext returned once); mints the access credential; audits `SESSION_ESTABLISHED{cause}`; returns `{accessCredential, refreshCredentialPlain, remaining}`. *(plan §2.1 `IssueSessionUseCase`, D1, D2; RF-1.1, RF-1.2, RF-8.3, RF-11.1)* — after T-11
  **Done when:** `session.rowHoldsDeadlinesVersionAndContext` passes and the returned access credential verifies through the codec with claims limited to session id, identity id, version, issue and expiry.

- [ ] **T-13 · `AssertSessionUseCase` — credential and session validity** — implements `SessionAssertionPort`: codec verify → an expired credential answers `CREDENTIAL_EXPIRED`; otherwise load the session, check state, deadlines, `securityVersionAtIssue` against the identity's current version and the identity status (A-27); any failure ends the session with its cause when still `ACTIVE`, audits `SESSION_ENDED{cause, source}` and answers `SESSION_INVALID`; success returns the session facts. *(plan §2.1 `AssertSessionUseCase`, D1, D6, D9; RF-1.1, RF-2.5, RF-2.6, RF-8.8, RNF-6, RNF-11, LC-18, LC-19, LC-20, LC-27, LC-30)* — after T-12
  **Done when:** `session.versionBumpEndsAtNextUse`, `session.endedRefusedOnNextRequest`, `session.absoluteDeadlineEndsDespiteActivity`, `session.statusChangeEndsSession` and `failClosed.absentEndedSupersededUnknownRefuses` pass.

- [ ] **T-14 · `AssertSessionUseCase` — activity and remaining time** — after a successful assertion, classify the request through `ActivityClass` and `ActivityMarkerPort`; `DELIBERATE` → `Session.touch` and save; `BACKGROUND` → no write; always return `remaining(now)`. *(plan §2.1 `AssertSessionUseCase`, D3, D4; RF-2.2, RF-2.3, RF-2.7, RF-3.2, LC-21, LC-28)* — after T-13
  **Done when:** `session.deliberateRequestMovesIdleOnly` and `session.pollingNeverMovesIdle` pass, and the returned `remaining` matches the aggregate's value after the touch.

- [ ] **T-15 · `RefreshSessionUseCase` — rotation path** — lock the family (`findForUpdateBySession` via the credential's family); check `Session.isValid` and the identity status; `rotate(presentedHash)`: `Rotated` → store the new hash with the retired one, mint a new access credential, answer `{accessCredential, refreshCredentialPlain, remaining}`; `Unknown` or invalid session → `SESSION_INVALID`; never calls `touch`. *(plan §2.1 `RefreshSessionUseCase`, D5, D6; RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.6, RF-11.5, LC-1)* — after T-13
  **Done when:** `refresh.rotatesAtomically`, `refresh.neverMovesDeadlinesOrGrantsAuthority` and `refresh.refusedForEndedUnknownOrRestricted` pass, and `rotation.allOrNothingUnderFailure` shows a forced failure after the hash insert leaves the previous generation current.

- [ ] **T-16 · `RefreshSessionUseCase` — reuse path** — `Reuse` → end the family and the session with `REUSE`, audit `REFRESH_REUSE_DETECTED` and `SESSION_ENDED{REUSE}`, enqueue `session-credential-reuse` through `SecurityNotificationPort`, answer `SESSION_INVALID`; all in the same transaction. *(plan §2.1 `RefreshSessionUseCase`, D5, D16; RF-3.5, RF-11.1, RF-11.4, RNF-3, LC-2, LC-3)* — after T-15
  **Done when:** `refresh.reuseEndsFamilyAndSessionEverywhere` and `refresh.duplicateRetryIsReuse` pass, and the outbox row, the audit rows and the ended states are rolled back together under a forced failure.

- [ ] **T-17 · `EndSessionUseCase`, `RevokeSessionsPort` and `CurrentSessionPort` implementations** — `end(sessionId, cause)` ends the session and its family with audit; `revokeAll(identityId, exceptSessionId, cause)` ends every active session of the identity but the exception; `CurrentSessionPort.currentSessionId()` from the asserted facts and `exists(sessionId)` = state `ACTIVE`. Plan 007's logout and individual revocation will call `end`; plans 003, 004 and 005 already call `revokeAll`. *(plan §2.1 `EndSessionUseCase`, `CurrentSessionPort`, D9; RF-1.5, RF-2.5, RF-2.6, RF-7.5, LC-18, LC-19, LC-20, LC-30)* — after T-13
  **Done when:** `revokeAll.exceptKeepsActingSession` and `session.individuallyRevokedRefused` pass, and plan 003's `replace.revokesOthersKeepsCurrentLeavesRunAuthority` and plan 004's revocation tests pass against the real adapter instead of their stubs.

- [ ] **T-18 · `StartReauthenticationUseCase`** — input: the ended session's id; load it within the ended-session retention; resolve its identity; refuse (generic `auth.sign-in.failed`) when unknown, out of retention, still active, or when the identity status permits no sign-in; create `LoginTransaction(REAUTHENTICATION, resumesSessionId)` bound to that identity; return `{state: LIMITED, transaction, address}` with the address read-only. The password and the challenge then run through `SignInUseCase` and the challenge use cases **[001]** under Login's throttles. *(plan §2.1 `StartReauthenticationUseCase`, D8, D14; RF-8.1, RF-8.2, RF-8.8, LC-13, LC-27)* — after T-12, T-06
  **Done when:** `reauth.fullProofBoundToEndedSessionIdentity`, `reauth.throttledAsSignIn` and `reauth.restrictedIdentityGenericFailure` pass, and a submitted address different from the ended session's identity is ignored.

- [ ] **T-19 · `CompleteSignInUseCase` [001] — `REAUTHENTICATION` branch** — on completion of a `REAUTHENTICATION` transaction: `IssueSessionUseCase` with cause `REAUTH`, audit `REAUTHENTICATION_SUCCEEDED`, answer `{state: COMPLETED, resume: true}` and skip `LandingRule` **[001]**; failures and voids audit `REAUTHENTICATION_FAILED` / `REAUTHENTICATION_VOIDED{reason}`; nothing else is read or replayed. *(plan §2.1 `CompleteSignInUseCase`, D8; RF-8.3, RF-8.5, RF-10.1, RF-11.1)* — after T-18
  **Done when:** `reauth.issuesSessionWithoutLanding` and `reauth.touchesNoRunTableAndReplaysNothing` pass.

- [ ] **T-20 · Re-authentication gates and terminal failure** — the `FORCED_CHANGE_REQUIRED` state **[004]** is entered inside a `REAUTHENTICATION` transaction exactly as in a sign-in; attempt exhaustion or expiry voids the transaction with its reason; the enrollment gate **[003]** is left to tenant re-entry (nothing to do here but a test). *(plan §2.1 `LoginTransaction`, D8; RF-8.4, RF-8.6, LC-14, LC-15, LC-16)* — after T-19
  **Done when:** `reauth.forcedChangeInsideTransaction`, `reauth.terminalFailureVoidsAndKeepsContext` (use-case half: the transaction is voided and a new start succeeds) and `reauth.enrollmentGateOnReentry` (with plan 003's gate stub) pass.

- [ ] **T-21 · `SessionStatusQuery`** — for the caller's session: `remaining`, both deadlines, `saveNoticeThreshold` and `safeReadRetry` from `SessionPolicy`; structurally `BACKGROUND`. *(plan §2.1 `SessionStatusQuery`, D4; RF-2.7, RF-4.6, RF-9.1)* — after T-14
  **Done when:** a test shows the query returns the policy values and does not move the idle deadline.

- [ ] **T-22 · `ExpireSessionsJob`** — durable job (`sharedsupport/jobs` **[002]**): claims `ACTIVE` sessions past a deadline in batches of the configured size, ends them (`IDLE` / `ABSOLUTE`) with audit; prunes ended sessions and families older than their retention. *(plan §2.1 `ExpireSessionsJob`, D15; RF-2.6, RF-11.1, RNF-11)* — after T-17
  **Done when:** a job test with a controllable clock ends exactly the expired sessions with the right cause and one audit entry each, prunes only rows past retention, and leaves live sessions untouched.

- [ ] **T-23 · `access` derivation from `SessionAssertionPort`** — the C10 derivation **[001/003]** starts from the asserted session facts; a request without valid facts never reaches derivation; a derivation that yields no authority for the requested tenant or project answers `access.denied-scope` and never a credential problem; membership removed since the session was issued leads to denied scope. *(plan §2.1 C10 derivation, D1, D13; RF-1.1, RF-5.1, RF-8.4, RNF-1, LC-8, LC-17)* — after T-13
  **Done when:** `session.grantsNoAuthorityWithoutDerivation`, `access.derivesFromSessionFactsOnly`, `access.deniedScopeNeverRefreshesCredentials` and `reauth.removedMembershipLeadsToChooser` pass.

## D · Infrastructure adapters

- [ ] **T-24 · Persistence adapters** — implementations (access API per A-35) of `SessionRepository` (`findForUpdate` as `SELECT … FOR UPDATE`) and `RefreshFamilyRepository` (lock by session; lookup by credential hash through `refresh_credential`; one unretired credential per family). *(plan §2.1 Persistence, D1, D5; RF-1.5, RF-3.1, RF-3.4, RNF-2)* — after T-02, T-03, T-11
  **Done when:** repository tests against real PostgreSQL pass for every method and `refresh.concurrentPresentationsOneHonoured` passes end to end (the second presentation waits on the lock and is answered as reuse).

- [ ] **T-25 · `AccessCredentialCodec`** — adapter over the existing Spring Security JWT support: claims limited to session id, identity id, version, issue and expiry; lifetime from `SessionPolicy`; signed with the existing key management; `verify` distinguishes expired from invalid. *(plan §2.1 Credential adapters, D2; RF-1.2, RF-11.2, RNF-8)* — after T-11, T-01
  **Done when:** a test decodes a minted credential and finds exactly the five claims, an expired one answers "expired", a tampered one "invalid", and no claim carries tenant, role or personal data.

- [ ] **T-26 · Refresh credential generator and hasher** — `RefreshCredentialGenerator` = 256-bit CSPRNG over `TokenGenerator` **[002]**; `RefreshCredentialHasher` over `TokenHasher` **[002]**; only the hash is ever persisted. *(plan §2.1 Credential adapters, D2; RF-1.2, RF-11.2, RNF-8)* — after T-11
  **Done when:** a test shows 256 bits of output, distinct values across 10 000 draws, and no plaintext in `refresh_credential` after an issue and a rotation.

- [ ] **T-27 · `SessionAuthenticationFilter`** — in the existing Spring Security chain: reads the access credential from the authorisation header, calls `AssertSessionUseCase`, publishes the facts to the security context, adds `X-ASpectra-Session-Remaining` to every authenticated answer, maps `CREDENTIAL_EXPIRED` → 401 `auth.credential.expired` and `SESSION_INVALID` → 401 `auth.session.invalid`; the sign-in, challenge, re-authentication and refresh endpoints are excluded from the filter. *(plan §2.1 Security filter, D4, D6; RF-2.6, RF-2.7, RF-4.1, RF-4.3, RF-11.3)* — after T-14, T-25
  **Done when:** `remainingHeader.onEveryAuthenticatedAnswer`, `authEndpoints.outsideSessionFilter` and `sessionInvalid.carriesNoCause` pass.

- [ ] **T-28 · Cookie, forgery-defence and origin configuration** — the refresh credential as a cookie with `Secure`, `HttpOnly`, `SameSite` per the configured origins, path restricted to the refresh and re-authentication endpoints, lifetime = absolute deadline; CSRF through Spring Security's configured SPA pattern on every state-changing endpoint including refresh; CORS allow-list from configuration; a CSRF failure → 403 `auth.csrf.invalid` with no session change and no refresh. *(plan §2.1 Cookie and origin configuration, D2, D13; RF-1.2, RF-1.4, RF-5.5, RNF-8, LC-9)* — after T-27
  **Done when:** `cookie.secureHttpOnlySameSitePathRestricted`, `csrf.refusedWithoutSessionChange`, `csrf.ownProblemNoSessionChange` and `cors.allowListOnly` pass.

- [ ] **T-29 · Outbox wiring of the reuse notification** — `session-credential-reuse` enqueued through `SecurityNotificationPort` **[002]** behind the outbox relay, addressed to the verified email in the request's locale; relay failures retried and recorded, never surfaced to the refresh answer. *(plan §2.1 Outbox and email adapter, D16; RF-11.4, RNF-9, LC-29)* — after T-05, T-16
  **Done when:** `notification.reuseTemplateWithoutSecrets` and `notification.failureNeverRevertsRevocation` pass with a fake provider that fails on first delivery.

- [ ] **T-30 · Audit adapter** — map `SESSION_ESTABLISHED{cause}`, `SESSION_ENDED{cause, source}`, `REFRESH_REUSE_DETECTED`, `REAUTHENTICATION_SUCCEEDED`, `REAUTHENTICATION_FAILED`, `REAUTHENTICATION_VOIDED{reason}`, `NOTIFICATION_ENQUEUED{template}` to `AuditPort` with identity id, session id, correlation id, network key and timestamp, appended in the effect's transaction; a serialisation guard rejects any payload holding a credential, a hash, a password or a code. *(plan §2.1 Audit adapter; RF-11.1, RF-11.2)* — after T-11
  **Done when:** `audit.entryPerEvent` and `audit.containsNoCredential` pass.

- [ ] **T-31 · Logging and problem redaction** — request and response logging exclude the authorisation header, the refresh cookie, `accessCredential`, `password` and `code`; problem details never echo a submitted value; a log-appender test asserts absence. *(plan §2.1, D19; RF-11.2)* — after T-30
  **Done when:** `logs.containNoCredential` passes with credentials present in every request and answer of the three endpoints and of a protected request.

- [ ] **T-32 · Job registration** — register `ExpireSessionsJob` with `sharedsupport/jobs` at the configured cadence and batch size of T-01. *(plan §2.1, D15; RF-2.6)* — after T-22
  **Done when:** the job is listed by the job registry at startup and runs once in an integration test when triggered.

## E · API

- [ ] **T-33 · `POST /api/auth/session/refresh`** — cookie-authenticated, CSRF-protected, outside the session filter; 200 `{accessCredential, expiresIn, remaining}` with the rotated cookie, or 401 `auth.session.invalid`; no other answer shape. *(plan §2.1 `SessionController`, D5, D6; RF-3.1, RF-3.5, RF-3.6, RF-4.3)* — after T-15, T-16, T-28
  **Done when:** contract tests show the two answers only, the rotated cookie attributes of T-28, no `cause` member on the 401, and a replayed cookie answering 401 while a following valid request of the same family also answers 401.

- [ ] **T-34 · `GET /api/auth/session/status`** — under the session filter, structurally background; 200 `{remaining, idleDeadline, absoluteDeadline, saveNoticeThreshold, safeReadRetry}`. *(plan §2.1 `SessionController`, D4; RF-2.7, RF-4.6, RF-9.1)* — after T-21, T-27
  **Done when:** a contract test shows the shape, that the call never moves the idle deadline, and that the values equal the configured policy.

- [ ] **T-35 · `POST /api/auth/session/reauthentication` and the completion shape** — `{endedSession}` → 200 `{state: LIMITED, transaction, address}` or 401 `auth.sign-in.failed`; the password and `challenge/*` steps are plan 001's endpoints with the transaction id; completion of a `REAUTHENTICATION` transaction answers `{state: COMPLETED, resume: true, accessCredential, expiresIn, remaining}` with the new cookie and never a `landing` member. *(plan §2.1 `SessionController`, D8; RF-8.1, RF-8.2, RF-8.3, RF-8.6)* — after T-18…T-20, T-28
  **Done when:** contract tests show the start shape, the generic 401 for an unknown or active session id, the completion shape without `landing`, and the generic failure body identical for a wrong password and a wrong code.

- [ ] **T-36 · Contract and absence checks** — a test enumerating the API surface: the four problem codes with correlation id; no answer of this feature carries a plaintext refresh credential, a hash, a cause or any tenant data; no endpoint of this feature reads or writes a run, approval, project or grant table; session end triggers no run policy. *(plan §3 RF-10, RF-11; RF-7.3, RF-10.1, RF-10.3, RF-11.3, RNF-10)* — after T-33…T-35
  **Done when:** `problems.fourSafeCodesWithCorrelationId`, `sessionEnd.triggersNoRunPolicy` and `session.neverSubstitutesForStepUpRecoveryOrForcedChange` pass.

## F · Front end

- [ ] **T-37 · `contracts/session` types and guards** — request / response / problem types with runtime guards for the three endpoints and the four problem codes; a guard rejects any answer carrying a refresh credential, a hash or a `cause`; the `resume: true` completion is a distinct union member from Login's `landing` completion. *(plan §2.2; RF-11.2, RF-11.3, RNF-8)*
  **Done when:** guard unit tests accept one fixture per documented answer and problem, and reject a completion carrying both `resume` and `landing` and a problem carrying `cause`.

- [ ] **T-38 · i18n bundles `identity-access.{en,es,fr}` and `shared/errors.{en,es,fr}`: session keys** — every string of R34 (marker, heading, statements, "Accepted runs continue" block, actions, footer), the D02 compact variant "Verify to continue protected work", the N28 copy and actions, the transport status, the forgery-defence problem, the "Save now" notice. *(plan §2.2 Shared used, D18; RF-7.2, RF-7.7, RF-12.3, RNF-13)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value, and the EN values of RF-7.2 and RF-5.2 match the specification verbatim.

- [ ] **T-39 · `AccessCredentialHolder` and `SessionState` store** — `shared/security`: the access credential and its expiry in memory only, cleared on session end and logout; `SessionState {status, remaining, saveNoticeThreshold, safeReadRetry, endedSessionId}` fed by the remaining-time header and `GET status`; `remaining` is displayed, never decremented. *(plan §2.2 `AccessCredentialHolder`, `SessionState`, D4; RF-1.2, RF-1.3, RF-2.7, RNF-4, RNF-8)* — after T-37
  **Done when:** `storage.containsNothingFromFeature` (storage spies on every mechanism) and `sessionState.displaysNeverCountsDown` pass.

- [ ] **T-40 · Background marker on transport** — `shared/api` sets the background marker on requests TanStack Query issues from interval, focus or reconnect refetch (query `meta.background`) and on `GET status`; requests from user actions carry none; no API exists to mark a request deliberate. *(plan §2.2 axios transport, D3; RF-2.3)* — after T-39
  **Done when:** `transport.marksRefetchesBackground` passes and a type-level test shows the marker option accepts only `background`.

- [ ] **T-41 · `RefreshCoordinator`** — single-flight: the first eligible failure starts `POST refresh`; every other eligible request awaits the same promise; outcome `REFRESHED | INVALID | INDETERMINATE`; the promise is cleared after settlement; an `INDETERMINATE` outcome is reported only when no server answer arrived. *(plan §2.2 `RefreshCoordinator`, D5, D7; RF-3.4, RF-3.6, RF-4.1, RF-4.5, LC-1, LC-7)* — after T-39
  **Done when:** `coordinator.coalescesConcurrentTabs` (one network call for N concurrent failures) and `coordinator.outcomeIsRefreshedInvalidOrIndeterminate` pass.

- [ ] **T-42 · Response interceptor** — `shared/api`: `auth.credential.expired` → coordinator, then one retry of the same request object (same URL, body, headers, idempotency key) unless its scope epoch **[001]** changed; `auth.session.invalid` → `SessionState.ENDED`; `access.denied-scope` → `DeniedScopeHandler`; `auth.csrf.invalid` → `CsrfFailureHandler`; network error or timeout → `TransportFailureStatus`; the sign-in, challenge, re-authentication and refresh endpoints and every non-401 answer are ignored; safe reads retry within `safeReadRetry`; mutations never auto-retry. *(plan §2.2 axios transport, D6, D7, D13; RF-4.1, RF-4.2, RF-4.3, RF-4.4, RF-4.5, RF-4.6, RF-9.6, RNF-9, LC-6)* — after T-41
  **Done when:** `interceptor.refreshesOnlyOnCredentialExpired`, `interceptor.ignoresAuthEndpointsAndNon401`, `interceptor.retriesSameRequestOnce`, `interceptor.dropsRetryWhenEpochMoved` and `interceptor.invalidEndsSessionIndeterminateShowsStatus` pass, and `retry.sameKeyReturnsRecordedOutcome` passes against the BE double.

- [ ] **T-43 · `DeniedScopeHandler` and N28** — `shared/errors`: on `access.denied-scope` one permission refresh (`permissionVersion` and grants) per problem, then resume or render `ProjectAccessDeniedPanel` with "This project isn't available", its statement, "Refresh permissions" and "Choose permitted project"; restricted data and subscriptions removed; a dirty draft kept and shown only if the refreshed scope permits; never a credential refresh or sign-in route. *(plan §2.2 `DeniedScopeHandler`, D13; RF-5.1, RF-5.2, RF-5.3, LC-8)* — after T-42
  **Done when:** `deniedScope.refreshesPermissionsOnceThenN28` and `deniedScope.draftShownOnlyIfRefreshedGrantPermits` pass.

- [ ] **T-44 · `CsrfFailureHandler` and `TransportFailureStatus`** — the forgery-defence problem renders "request could not be validated — reload and try again" and nothing else; the transport status is an accessible retry / offline state that names neither expiry nor sign-in, offers a retry and, when the indeterminate outcome was the coordinated refresh, "Retry session refresh" (one coordinated refresh per activation); a consequential operation is looked up before being re-offered. *(plan §2.2 `CsrfFailureHandler`, `TransportFailureStatus`, D13; RF-5.4, RF-5.5, RF-5.6, RNF-7, LC-7, LC-9)* — after T-42
  **Done when:** `csrfHandler.reloadAdviceOnly`, `transportStatus.neverClaimsExpiryOffersRetryRefresh` and `lostAck.looksUpNeverResubmits` pass.

- [ ] **T-45 · Realtime behaviour** — `shared/realtime` **[001]**: connects with the current access credential; heartbeats carry no activity; on `ENDED` disconnects; reconnects after resume with the new credential; connection loss is reported to the monitor as stream loss ("Stream interrupted · facts stale", "Refresh snapshot", "Leave monitor"), never as session end. *(plan §2.2 STOMP client, D12; RF-2.3, RF-6.3, RF-7.1, LC-11, LC-22)* — after T-39
  **Done when:** `realtime.lossRendersStreamInterruptedNotExpiry` passes and a unit test shows disconnect on `ENDED` and reconnect only after resume.

- [ ] **T-46 · Structural wall checks** — a lint rule and a test asserting the console has no HTTP client for customer targets (no module may call a target address; target addresses exist only in environment snapshots rendered as text) and that the run monitor renders a target's "unauthorised" answer as a C07 fact row. *(plan §2.2 Run monitor, D12; RF-6.1, RF-6.2, LC-10)* — after T-42
  **Done when:** `monitor.targetUnauthorisedIsFactRow` and `console.hasNoTargetClient` pass, and `runner.carriesNoConsoleCredential` passes on the BE side against the runner envelope double.

- [ ] **T-47 · Draft stamps and transient-secret registry** — Zustand scoped drafts carry `{identityId, tenantId, projectId, baseETag, editSeq}` and a dirty flag; a dirty-draft registry exposes "anything dirty in scope"; a transient-secret registry lets password and code fields register a `clear()`; nothing is persisted. *(plan §2.2 Drafts, D10; RF-9.2, RF-9.3, RF-9.5, LC-5, LC-17)* — after T-39
  **Done when:** `drafts.keptInMemoryUnderMask` passes (stamp present, storage spies see no write) and a registered field's `clear()` is invoked by the registry.

- [ ] **T-48 · `ProtectedBoundary` — mask** — wraps every protected route: on `ENDED` unmounts restricted subtrees, calls every registered `clear()`, pauses the query client, disconnects realtime, and renders the dialog modally; while masked no request needing the session is sent and no stream reopened. *(plan §2.2 `ProtectedBoundary`, D10; RF-7.1, RF-7.6, RF-9.2, LC-22)* — after T-47, T-45
  **Done when:** `boundary.masksClearsPausesDisconnects` and `boundary.noSessionRequestWhileMasked` pass.

- [ ] **T-49 · `ProtectedBoundary` — resume** — on `resume(newSession)`: compares `{identityId, tenantId, projectId}` of the preserved context and of each dirty draft with the new session and the refreshed scope; purges drafts whose stamp differs or whose edit the refreshed grants no longer permit; unmasks, resumes the query client and refetches; never saves anything itself; an in-flight save's answer, if it arrived, is honoured, otherwise the submission is looked up before "Save" is offered again. *(plan §2.2 `ProtectedBoundary`, D10; RF-8.3, RF-9.3, RF-9.4, RF-9.5, LC-5, LC-17)* — after T-48
  **Done when:** `drafts.purgedOnDifferentIdentityScopeOrGrant`, `boundary.neverSavesAutomatically` and `inFlightSave.answerHonouredElseLookup` pass.

- [ ] **T-50 · `SessionExpiryDialog` (R34), compact variant and `SecurityStatements` [001]** — the designed copy of RF-7.2 from the bundle; "Reauthenticate" primary after a definitive answer; "Retry session refresh" secondary, one coordinated refresh per activation, dialog unchanged on `INVALID`; the D02 compact variant for panels under the same rules; `SecurityStatements` extended with the R34 statements and the "Accepted runs continue" block; no cause rendered anywhere. *(plan §2.2 `SessionExpiryDialog`, `SecurityStatements`, D6, D18; RF-7.2, RF-7.3, RF-7.4, RF-7.7, RF-10.4, RF-12.1)* — after T-48, T-38
  **Done when:** `dialog.rendersDesignedCopy`, `dialog.retryIsSecondaryAndOncePerActivation`, `dialog.compactVariantSameRules` and `statements.acceptedRunsContinuePresent` pass.

- [ ] **T-51 · Dialog routes** — "Use local account recovery" discards the preserved context and drafts and routes to the public entry of specification 005; "sign out safely" discards them and calls plan 007's logout (stub until built), then routes to the sign-in screen. *(plan §2.2 `SessionExpiryDialog`; RF-7.5, LC-13)* — after T-50
  **Done when:** `dialog.recoveryAndSignOutDiscardContext` passes (holders and drafts empty before navigation; logout stub called once).

- [ ] **T-52 · `useReauthenticate` and `ReauthenticationStep`** — the hook posts `{endedSession}` and then delegates to the sign-in and challenge mutations **[001]** with purpose `REAUTHENTICATION`; the step renders the address read-only, `PasswordField` **[001]** with Show, `MfaChallenge` **[001]** when the answer is `LIMITED` with a challenge, the forced-change screen **[004]** when required, `GenericFailureMessage` **[001]**, "verification in progress"; a voided transaction returns the dialog to its initial state with the context preserved (D-a). *(plan §2.2 `ReauthenticationStep`, `useReauthenticate`, D8; RF-8.1, RF-8.2, RF-8.4, RF-8.6, LC-14, LC-16)* — after T-50
  **Done when:** `reauthStep.addressReadOnly` and the FE half of `reauth.terminalFailureVoidsAndKeepsContext` pass, and a mocked `LIMITED` answer mounts the challenge with the transaction id.

- [ ] **T-53 · Resume wiring, reload and multi-tab** — on `{state: COMPLETED, resume: true}` the hook stores the credential, updates `SessionState` and calls `ProtectedBoundary.resume()`; the preserved route is re-requested (authority derived by the server), the landing rule is never invoked; on a reload after expiry the first request sets `ENDED`, the dialog shows over the empty shell and resume requests the current route, falling back to the tenant chooser on denied scope (D-b); other tabs recover through the interceptor and the shared cookie without credentials. *(plan §2.2 `useReauthenticate`, `ProtectedBoundary`, D7, D8; RF-7.8, RF-8.3, RF-8.5, RF-8.7, LC-12, LC-23)* — after T-52, T-49
  **Done when:** unit tests show resume stores the credential, re-requests the route and never calls the landing hook, and a reload-without-draft case lands on the requested route; `multiTab.oneReauthRecoversOthersOnRefresh` is run in T-62.

- [ ] **T-54 · `SaveNowNotice`** — non-blocking banner while `status == VALID`, `remaining < saveNoticeThreshold` and the dirty registry is non-empty; "Save now" invokes the screen's own save under the original scope; hidden when nothing is dirty; never calls refresh or status to extend the session. *(plan §2.2 `SaveNowNotice`, D11; RF-9.1, RF-9.4, LC-4, LC-28)* — after T-47, T-39
  **Done when:** `saveNotice.shownUnderThresholdWithDirtyOnly` and `saveNotice.neverExtendsSession` pass.

- [ ] **T-55 · Run requests across expiry** — the launch feature **[F-RUN]** looks up an uncertain submission after resume by its idempotency key and never resubmits; nothing on resume touches the queue or replays an intent. *(plan §2.2 Run monitor, §3 RF-10; RF-10.2, RF-8.5, LC-24, LC-25)* — after T-49
  **Done when:** `launch.uncertainSubmissionLookedUpNeverResubmitted` passes (FE half) and a resume with a pending launch mutation issues one status lookup and zero launch calls.

- [ ] **T-56 · Reflow and accessibility primitives** — Tailwind breakpoints for the dialog, the compact panel, N28, the transport status and the notice; `shared/ui` `Dialog` with focus trap entering the heading, `LiveRegion` polite announcements that never move focus from the password or code field, 44 px targets, visible focus; every statement present at every width. *(plan §2.2 Shared used, D18; RF-12.1, RF-12.2, RNF-12, RNF-14)* — after T-50, T-43, T-44, T-54
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow, identical statement text at the three widths, focus on the dialog heading on open, and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-57 · Concurrency suite** — `refresh.concurrentPresentationsOneHonoured`, `refresh.duplicateRetryIsReuse`, `rotation.allOrNothingUnderFailure`, `revokeAll` racing a refresh, a version bump racing a protected request. *(plan §5.1 Concurrency, D5; RF-3.4, RF-3.5, RF-11.5, RNF-2, RNF-3, LC-1, LC-2, LC-3)* — after T-24, T-33
  **Done when:** all five tests pass ten times in a row with real PostgreSQL.

- [ ] **T-58 · Version, revocation and sign-in interplay suite** — `session.versionBumpEndsAtNextUse` driven by plan 004's reset and plan 003's replacement, `revokeAll.exceptKeepsActingSession` from a replacement, `session.statusChangeEndsSession`, `session.individuallyRevokedRefused`, `session.neverSubstitutesForStepUpRecoveryOrForcedChange`. *(plan §5.2 RF-2.5, RF-1.6 rows, D9; RF-1.6, RF-2.5, RF-2.6, LC-18, LC-19, LC-20, LC-27, LC-30)* — after T-57, T-36
  **Done when:** all five tests pass with plans 003 and 004 built or their recording stubs.

- [ ] **T-59 · Run continuity suite** — `run.continuesAcrossExpiryAndReauth` (an accepted Run under the F-RUN double keeps recording facts while the browser session expires and resumes), `launch.uncertainSubmissionLookedUpNeverResubmitted` (BE half), `sessionEnd.triggersNoRunPolicy`. *(plan §5.2 RF-10 rows, D12; RF-10.1, RF-10.2, RF-10.3, RNF-10, LC-24, LC-25)* — after T-36
  **Done when:** all three tests pass; the run-scoped authority double records zero calls from this feature.

- [ ] **T-60 · Fail-closed and wall suite** — `failClosed.absentEndedSupersededUnknownRefuses` (every use case × every bad session or family state), `console.hasNoTargetClient`, `runner.carriesNoConsoleCredential`, `monitor.targetUnauthorisedIsFactRow`, `realtime.lossRendersStreamInterruptedNotExpiry`. *(plan §5.2 RNF-6, RF-6 rows, D12; RF-6.1, RF-6.2, RF-6.3, RNF-6, LC-10, LC-11)* — after T-46, T-36
  **Done when:** all five tests pass; the fail-closed test has one case per (use case × bad state) pair and none is skipped.

- [ ] **T-61 · Startup suite** — `sessionPolicy.startupFailsOnAbsentUnboundedOrInconsistentValue` with one case per property of T-01 plus the two cross-field rules. *(plan §5.1 Startup, D17; RF-2.1, RNF-5)* — after T-01
  **Done when:** every case fails the context refresh with a message naming the property or rule, and the complete set starts.

- [ ] **T-62 · FE e2e journeys** — Playwright journeys for US-1…US-11 in EN, ES and FR at desktop, tablet and mobile widths against the BE test doubles: silent refresh while active; idle and absolute expiry; expiry mid-edit → dialog → re-authentication → return to context with the draft; the pre-expiry notice and save; two tabs (`multiTab.oneReauthRecoversOthersOnRefresh`); password reset on another device ending the session; denied scope after grant removal; network loss → retry status; accepted Run surviving expiry; target "unauthorised" rendered as a fact; `reauth.returnsToInterruptedContext`; `reload.afterExpiryShowsDialogThenRequestedRoute`; `languageSwitch.changesDisplayOnly`. *(plan §5.1 FE e2e, §5.2 journeys; RF-7.8, RF-8.3, RF-8.7, RF-12.3, RNF-13, LC-12, LC-23, LC-26, US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8, US-9, US-10, US-11)* — after T-56, T-36
  **Done when:** all journeys pass in the three languages and three widths, and a language switch while the dialog is open changes only displayed text.

- [ ] **T-63 · Automated accessibility** — axe on the dialog (initial, credentials, challenge, failure, in-progress), the compact panel, N28, the transport status and the notice at the three widths and three languages (`a11y.sessionStates.{en,es,fr}.{desktop,tablet,mobile}`). *(plan §5.1 FE e2e + accessibility; RF-12.1, RF-12.2, RNF-12, RNF-14)* — after T-62
  **Done when:** zero critical or serious violations on every combination.

- [ ] **T-64 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on the dialog, the panel states, the transport status and the notice: focus entry on the heading, focus trap, announcements without focus theft from the password or code field, labels, 44 px targets; versions recorded in the release-support matrix. *(plan §5.1 Manual assistive; RF-12.2, RNF-12)* — after T-63
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-65 · Adversarial suite (US-12)** — replay of a copied refresh cookie; a session kept alive by polling only; a forged "deliberate" marker; a target "unauthorised" answer injected into the monitor; a second identity resuming the same route with a draft in memory; a Run request re-sent after re-authentication; a 403 loop attempt; reading the cause of an ended session from any answer or log. *(plan §5.2 US-12 row, D2, D3, D5, D10, D12; RF-2.3, RF-3.5, RF-6.1, RF-7.3, RF-9.3, RF-10.2, RNF-3, RNF-8, US-12)* — after T-57…T-60
  **Done when:** every attack test fails to achieve its goal and asserts the generic, fail-closed or structural response.

- [ ] **T-66 · Area A evidence pack** — collect the passing runs of T-57, T-58, T-28 (cookie / CSRF / CORS), T-59 and T-62 into the release evidence for requirements A.12: concurrent refresh, reuse detection, multi-tab behaviour, origin configuration and active runs surviving browser expiry. *(plan §5.2 RNF-15 row; RNF-15)* — after T-57…T-65
  **Done when:** the evidence pack lists each required item with its test run id.

## H · Closing

- [ ] **T-67 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-57…T-66
  **Done when:** the script reports 125 of 125 identifiers (68 RF, 15 RNF, 30 LC, 12 US) covered by at least one passing test.

- [ ] **T-68 · Register-dependency review** — walk plan §6 — A-31, A-49, A-50, S-39, S-40, S-41 and S-42 as decided and drawn, and A-11, A-27, A-30, A-35, A-39 (port contract), A-45, S-02, S-07 and S-14: for each, confirm the built assumption matches the current Figma register state or record the deferral. *(spec §7 item 13)* — after T-67
  **Done when:** each of the sixteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-69 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run ids, a11y report, manual checklist, coverage report, evidence pack, register review). — after T-68
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-11, T-12, T-13, T-23 |
| RF-1.2 | T-12, T-25, T-26, T-28, T-39 |
| RF-1.3 | T-39 |
| RF-1.4 | T-28 |
| RF-1.5 | T-02, T-07, T-17, T-24 |
| RF-1.6 | T-11, T-58, T-58 |
| RF-2.1 | T-01, T-07, T-61 |
| RF-2.2 | T-08, T-10, T-14 |
| RF-2.3 | T-10, T-14, T-40, T-45, T-65 |
| RF-2.4 | T-07, T-08 |
| RF-2.5 | T-02, T-08, T-13, T-17, T-58, T-58 |
| RF-2.6 | T-02, T-04, T-07, T-13, T-17, T-22, T-27, T-32, T-58 |
| RF-2.7 | T-07, T-14, T-21, T-27, T-34, T-39 |
| RF-3.1 | T-03, T-09, T-15, T-24, T-33 |
| RF-3.2 | T-10, T-14, T-15 |
| RF-3.3 | T-09, T-15 |
| RF-3.4 | T-15, T-24, T-41, T-57 |
| RF-3.5 | T-03, T-09, T-16, T-33, T-57, T-65 |
| RF-3.6 | T-15, T-33, T-41 |
| RF-4.1 | T-04, T-27, T-41, T-42 |
| RF-4.2 | T-42 |
| RF-4.3 | T-27, T-33, T-42 |
| RF-4.4 | T-42 |
| RF-4.5 | T-41, T-42 |
| RF-4.6 | T-01, T-21, T-34, T-42 |
| RF-5.1 | T-23, T-43 |
| RF-5.2 | T-43 |
| RF-5.3 | T-43 |
| RF-5.4 | T-44 |
| RF-5.5 | T-04, T-28, T-44 |
| RF-5.6 | T-44 |
| RF-6.1 | T-46, T-60, T-65 |
| RF-6.2 | T-46, T-60 |
| RF-6.3 | T-45, T-60 |
| RF-7.1 | T-45, T-48 |
| RF-7.2 | T-38, T-50 |
| RF-7.3 | T-04, T-36, T-50, T-65 |
| RF-7.4 | T-50 |
| RF-7.5 | T-17, T-51 |
| RF-7.6 | T-48 |
| RF-7.7 | T-38, T-50 |
| RF-7.8 | T-53, T-62 |
| RF-8.1 | T-06, T-18, T-35, T-52 |
| RF-8.2 | T-18, T-35, T-52 |
| RF-8.3 | T-12, T-19, T-35, T-49, T-53, T-62 |
| RF-8.4 | T-06, T-20, T-23, T-52 |
| RF-8.5 | T-19, T-53, T-55 |
| RF-8.6 | T-06, T-20, T-35, T-52 |
| RF-8.7 | T-53, T-62 |
| RF-8.8 | T-13, T-18 |
| RF-9.1 | T-01, T-21, T-34, T-54 |
| RF-9.2 | T-47, T-48 |
| RF-9.3 | T-47, T-49, T-65 |
| RF-9.4 | T-49, T-54 |
| RF-9.5 | T-47, T-49 |
| RF-9.6 | T-42 |
| RF-10.1 | T-11, T-19, T-36, T-59 |
| RF-10.2 | T-55, T-59, T-65 |
| RF-10.3 | T-11, T-36, T-59 |
| RF-10.4 | T-50 |
| RF-11.1 | T-12, T-16, T-19, T-22, T-30 |
| RF-11.2 | T-25, T-26, T-30, T-31, T-37 |
| RF-11.3 | T-04, T-27, T-36, T-37 |
| RF-11.4 | T-05, T-16, T-29 |
| RF-11.5 | T-15, T-57 |
| RF-12.1 | T-50, T-56, T-63 |
| RF-12.2 | T-56, T-63, T-64 |
| RF-12.3 | T-38, T-62 |
| RNF-1 | T-11, T-23 |
| RNF-2 | T-03, T-09, T-24, T-57 |
| RNF-3 | T-09, T-16, T-57, T-65 |
| RNF-4 | T-39 |
| RNF-5 | T-01, T-61 |
| RNF-6 | T-13, T-60, T-60 |
| RNF-7 | T-44 |
| RNF-8 | T-25, T-26, T-28, T-37, T-39, T-65 |
| RNF-9 | T-05, T-29, T-42 |
| RNF-10 | T-11, T-36, T-59 |
| RNF-11 | T-13, T-22 |
| RNF-12 | T-56, T-63, T-64 |
| RNF-13 | T-38, T-62 |
| RNF-14 | T-56, T-63 |
| RNF-15 | T-66, T-66 |
| LC-1 | T-09, T-15, T-41, T-57 |
| LC-2 | T-09, T-16, T-57 |
| LC-3 | T-09, T-16, T-57 |
| LC-4 | T-08, T-54 |
| LC-5 | T-47, T-49 |
| LC-6 | T-42 |
| LC-7 | T-41, T-44 |
| LC-8 | T-23, T-43 |
| LC-9 | T-28, T-44 |
| LC-10 | T-46, T-60 |
| LC-11 | T-45, T-60 |
| LC-12 | T-53, T-62 |
| LC-13 | T-18, T-51 |
| LC-14 | T-20, T-52 |
| LC-15 | T-20 |
| LC-16 | T-20, T-52 |
| LC-17 | T-23, T-47, T-49 |
| LC-18 | T-13, T-17, T-58 |
| LC-19 | T-13, T-17, T-58 |
| LC-20 | T-13, T-17, T-58 |
| LC-21 | T-08, T-10, T-14 |
| LC-22 | T-45, T-48 |
| LC-23 | T-53, T-62 |
| LC-24 | T-55, T-59 |
| LC-25 | T-55, T-59 |
| LC-26 | T-62 |
| LC-27 | T-07, T-13, T-18, T-58 |
| LC-28 | T-10, T-14, T-54 |
| LC-29 | T-29 |
| LC-30 | T-13, T-17, T-58 |
| US-1 … US-11 | T-62 (journeys) |
| US-12 | T-65 (adversarial suite) |

Self-check: all 68 RF criteria, 15 RNF, 30 limit cases and 12 user stories of `specSessionsRefreshExpiry.md` v0.1 appear above; every task of phases A–G names at least one requirement, and the closing tasks name the completion criterion they serve.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| A-31 re-authentication from R34 (decided, Q2) | Full sign-in proof bound to the ended session's identity; completion issues a session and skips landing | T-06, T-18, T-19, T-35, T-52, T-53 |
| A-49 human activity (decided, Q5) | Server-side classification by operation class; transport marker can only lower a request to background | T-10, T-14, T-40 |
| A-50 reuse detection (decided, Q6) | Whole family and session ended; audit; `session-credential-reuse` notification through the outbox | T-05, T-09, T-16, T-29 |
| S-39 R34 re-authentication step (decided, Q2) | Read-only address, password with Show, challenge, generic failure, in-progress, voided → initial state | T-50, T-52 |
| S-40 pre-expiry notice (decided, Q4) | `SaveNowNotice` on server-supplied remaining time and the dirty registry | T-54 |
| S-41 retry / offline status (decided, Q3) | `TransportFailureStatus` with "Retry session refresh" | T-44 |
| A-11 configuration values | `SessionPolicyProperties` with the nine values and two cross-field rules; the client's numbers come from `GET status` | T-01, T-21, T-34, T-61 |
| A-27 identity statuses | `ACTIVE` only for assertion and re-authentication; `RESTRICTED` and `ERASING` end the session and fail generically | T-13, T-18 |
| A-30 sign-in notifications | Only the reuse notification is sent by this feature | T-05, T-29 |
| A-35 persistence tooling | Adapters and migrations against the placeholder access API and migration tool | T-02, T-03, T-24 |
| A-39 revocation scope | `RevokeSessionsPort.revokeAll(identity, except, cause)` as decided for 003 and 004; run-scoped authority never referenced | T-11, T-17, T-58 |
| A-45 forced-change reading | Reading (a): `FORCED_CHANGE_REQUIRED` inside the re-authentication transaction | T-06, T-20, T-52 |
| S-02 responsive coverage | M23 (`432:4427`) for R34; the compact panel, the notice, the status and N28 follow the D01 responsive policy | T-56, T-63 |
| S-07 app-shell sign-out | "sign out safely" on R34 calls plan 007's logout; no shell menu assumed | T-51 |
| S-14 templates | `session-credential-reuse` with the D04 copy (`437:4431`) in EN / ES / FR | T-05 |
| S-42 forgery-defence state | `CsrfFailureHandler` renders "reload and try again"; no sign-out, no refresh | T-28, T-44 |
