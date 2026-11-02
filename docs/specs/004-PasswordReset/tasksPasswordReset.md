# Tasks 004 — Password reset and forced password change

**Feature** Self-service password reset by emailed link, and the forced password change state · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specPasswordReset.md` v0.2 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planPasswordReset.md` v0.2 (modules §2, decisions D1–D19, tests §5, register assumptions §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[002]** extend an element that plan 001 (Login) or plan 002 (Signup and verification) builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 001's and plan 002's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-01 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Reset configuration record** — `ResetPolicy` value object and `ResetPolicyProperties` binding: token expiry, maximum password age, throttle steps and restriction windows for `RESET_ADDRESS` and `RESET_NETWORK`, per-address message cap and its window, public origin (the same configuration value `SignupPolicy` **[002]** validates), notification send window, default message language for the compromise notice. Each value `@Validated` as present, positive and bounded above by a configured ceiling; cross-field constraint: the address suppression window is strictly shorter than the network window (decision Q7); the origin is an absolute `https` origin; the default language is one of EN / ES / FR. *(plan §2.1 `ResetPolicy`, `ResetPolicyProperties`, D5, D13, D14; RF-3.6, RF-3.7, RF-7.4, RNF-6, RNF-7)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed or set unbounded, when the address window equals or exceeds the network window, and when the origin is not `https`.

- [ ] **T-02 · Migration: credential metadata** — `identity_credential` **[001]** gains `changed_at` and `policy_version`, back-filled with the migration time and the then-current policy version so that no existing identity is flagged by the migration itself. *(plan §2.1 Persistence, D9; RF-7.3, RF-7.4)* — after T-01
  **Done when:** the migration applies on an empty database and on one with existing credentials, and every back-filled row carries the migration time and the current policy version (a sign-in by an existing identity with a compliant password sets no requirement).

- [ ] **T-03 · Migration: `password_reset_link` and the reset outcome record** — `password_reset_link` (`id`, `identity_id`, `token_hash` unique, `purpose`, `issued_at`, `expires_at`, `superseded_at`, `used_at`; partial index on live links per identity) and the small outcome record keyed by the idempotency key of a completing reset submission (`idempotency_key` unique, `identity_id`, `outcome`, `recorded_at`). *(plan §2.1 Persistence, D1, D2, D18; RF-2.2, RF-2.3, RF-5.1, RNF-3, RNF-4, LC-31)* — after T-02
  **Done when:** the migration applies; `token_hash` uniqueness and `idempotency_key` uniqueness are enforced; `identity_id` cascades on identity delete; the partial index exists and covers only rows with `superseded_at IS NULL AND used_at IS NULL`.

- [ ] **T-04 · Migration: `forced_change_requirement`, transaction state and throttle dimensions** — `forced_change_requirement` (`id`, `identity_id`, `source`, `set_at`, `cleared_at`; partial unique index on pending rows per identity); `login_transaction` **[001]** accepts the state value `FORCED_CHANGE_REQUIRED`; `signin_throttle` **[001]** keeps its shape and simply receives the new dimension values `RESET_ADDRESS` and `RESET_NETWORK`. *(plan §2.1 Persistence, D8, D9; RF-3.1, RF-7.1, RF-7.6)* — after T-03
  **Done when:** the migration applies; a second pending requirement for the same identity is rejected while a cleared row plus a new pending row is accepted; a transaction row in state `FORCED_CHANGE_REQUIRED` and a throttle row with dimension `RESET_ADDRESS` can be inserted and plan 001's rows are unaffected.

- [ ] **T-05 · Outbox templates registered** — register template ids `password-reset`, `password-changed` and `compromise-response-notice` with the outbox message type and the three locales, the D04 copy (`437:4431`) per S-14 carrying the statements of RF-2.8 (product name, link expires, no MFA reset, no automatic sign-in, no action needed and password unchanged if not requested), RF-5.7 (password changed, when, what to do if not you) and RF-7.2 (change required at next sign-in, every session signed out, when, what to do if unexpected). *(plan §2.1 Outbox relay and email adapter, D13; RF-2.8, RF-5.7, RF-7.2, RNF-11, RNF-13)* — after T-01
  **Done when:** `template.passwordResetCarriesStatements` and `templates.threeMessagesCarryStatements` pass in EN, ES and FR with sample variables, producing a non-empty subject and body for each of the nine renderings.

## B · Domain (pure, no Spring)

- [ ] **T-06 · `Credential` extension [001]** — add `changedAt` and `policyVersion`; new method `replace(newEncoded, now, policyVersion)` returning the new credential; `matches(raw, verifier)` reused unchanged for the current-password check and the same-as-current check; no accessor returns the stored hash. *(plan §2.1 `Credential`, D12; RF-5.2, RF-5.4, RF-7.3, RF-7.4, RF-8.5, RF-8.8)* — after T-01
  **Done when:** unit tests show `replace` yields a credential carrying the given time and policy version while the old one is unchanged, `matches` holds for the current password and fails for any other, and `toString` is redacted.

- [ ] **T-07 · `GlobalIdentity.isResetEligible()` [001]** — true only for status `ACTIVE` with a verified email (A-27 assumption); `bumpSecurityVersion()` reused as the hook every credential change calls. *(plan §2.1 `GlobalIdentity`, §6 A-27; RF-2.2, RF-2.6, LC-1, LC-2, LC-3)* — after T-06
  **Done when:** a table test over the four statuses × verified / unverified shows `true` for `ACTIVE` + verified only.

- [ ] **T-08 · `ResetLink` entity** — token hash, identity id, purpose `PASSWORD_RESET`, `issuedAt`, `expiresAt`, `supersededAt`, `usedAt`; `isNewestAndLive(now)`, `markUsed(now)`, `supersede(now)`; mirrors `VerificationLink` **[002]** without sharing its table. *(plan §2.1 `ResetLink`, D1; RF-2.2, RF-2.5, RF-5.1, RF-5.3, RF-6.1)* — after T-01
  **Done when:** unit tests cover live, expired, superseded and used states, and `markUsed` on an expired, superseded or already-used link throws.

- [ ] **T-09 · `ResetOutcome`** — `VALID`, `DONE`, `UNAVAILABLE(cause)` with causes `UNKNOWN`, `MALFORMED`, `EXPIRED`, `USED`, `SUPERSEDED`; one `toResponse()` mapping every cause to the same value; the cause is retained for audit only. *(plan §2.1 `ResetOutcome`, D3, D19; RF-6.1, RF-6.2, RF-6.3, RF-10.1, RNF-1)*
  **Done when:** a unit test shows all five causes produce one identical response value and that the cause is readable only through the audit-facing accessor.

- [ ] **T-10 · `ForcedChangeRequirement`** — identity id, `source` (`COMPROMISE_RESPONSE`, `POLICY_CHANGE`, `PASSWORD_AGE`), `setAt`, `clearedAt`; `clear(now)`, `isPending()`; at most one pending requirement per identity (a second trigger while one is pending is an audit event only, decided by the use case). *(plan §2.1 `ForcedChangeRequirement`, D9; RF-5.6, RF-7.1, RF-8.3, RF-8.9)*
  **Done when:** unit tests show pending → cleared, `clear` on a cleared requirement throws, and the source survives clearing (needed by RF-8.9).

- [ ] **T-11 · `ForcedChangeRule`** — pure function `(credential.changedAt, credential.policyVersion, currentPolicyVersion, maxPasswordAge, now) → Optional<source>`: `POLICY_CHANGE` when the credential's policy version is older than the current one, else `PASSWORD_AGE` when `now − changedAt > maxPasswordAge`, else none. *(plan §2.1 `ForcedChangeRule`, D9; RF-7.3, RF-7.4, LC-20, LC-21)* — after T-06
  **Done when:** `forcedChangeRule.policyChange` and `forcedChangeRule.passwordAge` pass, a credential that satisfies both yields none, and a credential failing both yields `POLICY_CHANGE`.

- [ ] **T-12 · `LoginTransaction` state `FORCED_CHANGE_REQUIRED` [001]** — `requireForcedChange()` (from `FIRST_FACTOR_PASSED` only), `forcedChangeDone(now, newSecurityVersion)` (back to `FIRST_FACTOR_PASSED`, re-stamping the stored `securityVersion` so the transaction survives its own credential change); `recordFailedAttempt`, `void`, `recordedOutcome(key)` apply in the new state; `complete` from the new state throws. *(plan §2.1 `LoginTransaction`, D8; RF-7.6, RF-7.7, RF-7.10, RF-8.5, RF-8.6, RF-8.7, RF-8.10, RF-8.12, LC-23, LC-24, LC-29, LC-31)*
  **Done when:** unit tests show the three transitions, a throw on `complete` or on a challenge-side transition from `FORCED_CHANGE_REQUIRED`, attempts exhausting to `VOIDED(ATTEMPTS_EXHAUSTED)` in the new state, and a recorded outcome returned for a repeated key.

- [ ] **T-13 · Reset throttle dimensions on `AbuseThrottlePolicy` [001]** — add dimension keys `RESET_ADDRESS` and `RESET_NETWORK` fed by the `ResetPolicy` steps and windows; no change to the policy arithmetic; no exemption parameter for any network. *(plan §2.1 `AbuseThrottlePolicy`, D5; RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.7, RF-6.5, RNF-2)* — after T-01
  **Done when:** `throttlePolicy.resetKeysProgressiveSteps` passes, sign-in dimension state does not affect reset decisions and vice versa, and the policy exposes no way to exempt a network key.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-27…T-31); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-14 · Port interfaces** — new: `ResetLinkRepository` (`findByTokenHash`, `supersedeAllLive`, `save`, `consumeIfLive` conditional update), `ForcedChangeRequirementRepository` (`findPending`, `save`, `clear`), `RevokeSessionsPort` (`revokeAll(identityId, except)` → session service of F-IDN-08), `ResetOutcomeStore` (recorded outcome by idempotency key). Reused: `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `PasswordVerifier`, `Clock`, `AuditPort` **[001]**; `TokenGenerator`, `TokenHasher`, `PasswordPolicyPort`, `TransactionalEmailPort`, `SecurityNotificationPort` **[002]**. No tenant, membership, grant, factor or run port exists in this feature. *(plan §2.1 Ports, D6, D7; enables RF-2 … RF-8; RF-2.7, RF-5.5, RF-5.9, RF-7.5, RNF-9)* — after T-06…T-13
  **Done when:** the application module compiles with no dependency on Spring or persistence classes (ArchUnit module-boundary test, test-only), and the boundary test lists no port that reaches a tenant, factor or run table.

- [ ] **T-15 · `IssueResetLink` service** — supersede every live reset link of the identity, generate a token and store only its hash with expiry from `ResetPolicy`, enqueue the `password-reset` message (locale, verified address, link built on `ResetPolicy.publicOrigin`) in the outbox, audit `RESET_LINK_ISSUED`. *(plan §2.1 `IssueResetLink`, D1, D13, D14; RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.8, RF-10.1, LC-4)* — after T-14
  **Done when:** `resetRequest.newLinkSupersedesEarlier` passes and the enqueued link starts with the configured public origin regardless of the request host.

- [ ] **T-16 · `RequestPasswordResetUseCase` — constant-work path and eligible branch** — normalise the address with `EmailAddress` **[002]**; evaluate both reset throttles; always generate one token and its hash (discarded when ineligible); one identity lookup; `isResetEligible()` and under the message cap → `IssueResetLink`; otherwise write only throttle counters and the audit event `RESET_REQUESTED{matched, issued}`; never touch status, sessions, factor or requirement; one response shape in every branch. *(plan §2.1 `RequestPasswordResetUseCase`, D4; RF-1.5, RF-2.1, RF-2.2, RF-2.6, RF-2.7, RF-10.1, RF-10.5, RNF-1, LC-1, LC-2, LC-3)* — after T-15
  **Done when:** `resetRequest.activeVerifiedGetsHashedTokenWithConfiguredExpiry`, `resetRequest.unknownUnverifiedRestrictedErasingIssueNothing`, `resetRequest.changesNothingElse` and `resetRequest.normalisesAddress` pass.

- [ ] **T-17 · `RequestPasswordResetUseCase` — throttle and message-cap branches** — progressive delay from `delayFor`; beyond the restriction threshold on either key no token and no message during the window, same response; per-address message cap checked before `IssueResetLink`; windows lift by time; audit `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]** for the reset keys. *(D5; RF-3.1, RF-3.3, RF-3.4, RF-3.5, RNF-2, RNF-5, LC-33)* — after T-16
  **Done when:** `resetRequest.restrictedIssuesNothingSameAnswer` and `resetRequest.perAddressMessageCapBoundsOutbox` pass, and a request after the window elapses (controllable clock) issues a link again.

- [ ] **T-18 · `ValidateResetLinkQuery`** — hash the presented token, load the link, return `VALID` or `UNAVAILABLE(cause)` without changing any row; audit `RESET_LINK_REJECTED{cause}` on failure; count a failure on `RESET_NETWORK`; the response carries no identity data. *(plan §2.1 `ValidateResetLinkQuery`, D3; RF-4.1, RF-4.7, RF-5.3, RF-6.1, RF-6.2, RF-6.5, LC-5, LC-8)* — after T-15
  **Done when:** `validate.doesNotConsume` and `validate.supersededIsUnavailable` pass, and a validated link re-validated a second time is still `VALID`.

- [ ] **T-19 · `ChangeCredential` internal service** — in the caller's transaction and in this order: `Credential.replace`, `bumpSecurityVersion`, supersede every live reset link, clear any pending requirement, `RevokeSessionsPort.revokeAll(identityId, except = none)`, enqueue `password-changed` through `SecurityNotificationPort` unless the cleared requirement's source is `COMPROMISE_RESPONSE` (then audit `NOTIFICATION_SKIPPED{reason = COMPROMISE_NOTIFIED}`), audit `CREDENTIAL_CHANGED{cause}` and `SESSIONS_REVOKED{cause}`; touches identity tables only. *(plan §2.1 `ChangeCredential`, D6, D7, D13; RF-5.4, RF-5.5, RF-5.6, RF-5.7, RF-5.9, RF-8.9, RF-8.11, RF-10.1, RNF-9, RNF-10, LC-14, LC-34)* — after T-14
  **Done when:** a unit test with recording doubles shows the effects in that order for cause `RESET` and for cause `FORCED` with sources `POLICY_CHANGE` and `PASSWORD_AGE` (notification enqueued), and for cause `FORCED` with source `COMPROMISE_RESPONSE` (no notification, `NOTIFICATION_SKIPPED` audited); no factor, membership, status or run port is invoked.

- [ ] **T-20 · `CompletePasswordResetUseCase` — validation and the success path** — look up the recorded outcome for the idempotency key first; confirmation match and policy through `PasswordPolicyPort`; same-as-current via `Credential.matches` → field rejection without consuming; then one transaction: `consumeIfLive` (one row) → `ChangeCredential(cause = RESET)`, record the outcome, return `RESET_DONE`; never a session. *(plan §2.1 `CompletePasswordResetUseCase`, D2, D12, D18; RF-4.5, RF-4.6, RF-5.1, RF-5.2, RF-5.4, RF-5.6, RF-5.8, RF-5.9, LC-9, LC-10, LC-14, LC-31)* — after T-19, T-18
  **Done when:** `complete.sameAsCurrentRejectedTokenNotConsumed`, `complete.oldPasswordStopsWorkingImmediately`, `complete.clearsPendingRequirementOfAnySource` and `complete.resubmitReturnsRecordedOutcome` pass.

- [ ] **T-21 · `CompletePasswordResetUseCase` — unavailable path** — `consumeIfLive` affecting zero rows → `UNAVAILABLE(cause)` with the submitted password discarded (never encoded, never stored); audit `RESET_LINK_REJECTED{cause}`; count on `RESET_NETWORK`; malformed or unknown tokens follow the same path. *(D2, D3; RF-6.3, RF-6.5, RF-10.1, RNF-7, LC-7)* — after T-20
  **Done when:** `complete.afterExpiryIsUnavailableAndDiscardsPassword` and `unusableLink.countsOnNetworkKeyOnly` pass, and the credential row is byte-identical before and after a rejected submission.

- [ ] **T-22 · `EvaluateForcedChangeStep` wired into `SignInUseCase` [001]** — runs only after a correct first factor on a verified identity, after the verification branch; loads the pending requirement; if none, applies `ForcedChangeRule` and persists a new requirement when it yields a source (audit `FORCED_CHANGE_SET{source}`); if pending, moves the new transaction to `FORCED_CHANGE_REQUIRED` and returns the `FORCED_CHANGE` response in place of `LIMITED` / `COMPLETED`; a second trigger while one is pending records an audit event only. *(plan §2.1 `EvaluateForcedChangeStep`, D8, D9; RF-7.3, RF-7.4, RF-7.6, RF-7.8, RF-7.9, RF-7.10, LC-20, LC-21, LC-22)* — after T-11, T-12, T-14
  **Done when:** `signIn.policyChangeFlagsAtFirstFactor`, `signIn.wrongPasswordNeverSetsRequirement`, `signIn.unverifiedRoutesToVerificationNotForcedChange` and `signIn.flaggedIdentityGetsForcedChangeBeforeChallenge` pass.

- [ ] **T-23 · `CompleteForcedChangeUseCase` — guards and the current-password check** — lock the transaction row; require state `FORCED_CHANGE_REQUIRED`, unexpired, attempts remaining; return the recorded outcome for a repeated idempotency key; verify the current password with `PasswordVerifier`: wrong → `recordFailedAttempt`, both sign-in throttle keys `ACCOUNT` and `NETWORK`, audit `FORCED_CHANGE_ATTEMPT_FAILED`, void at zero attempts (requirement untouched); same-as-current → field rejection not counted; expiry → void. *(plan §2.1 `CompleteForcedChangeUseCase`, D8, D11, D18; RF-8.5, RF-8.6, RF-8.7, RF-8.8, RF-10.1, RNF-2, LC-23, LC-24, LC-31)* — after T-22, T-19
  **Done when:** `forcedChange.wrongCurrentIsGenericAndCountedOnBothKeys`, `forcedChange.lastAttemptVoidsRequirementStays`, `forcedChange.expiryVoids` and `forcedChange.sameAsCurrentRejectedNotCounted` pass.

- [ ] **T-24 · `CompleteForcedChangeUseCase` — success and continuation** — policy and confirmation checks; `ChangeCredential(cause = FORCED)`; `forcedChangeDone`; audit `FORCED_CHANGE_COMPLETED`; branch: factor enrolled → state `FIRST_FACTOR_PASSED`, response `LIMITED` with the same transaction; no factor → `CompleteSignInUseCase` **[001]** → `COMPLETED` with landing; the response carries no source. *(D6, D8; RF-8.3, RF-8.5, RF-8.9, RF-8.10, RF-8.11, LC-27, LC-28, LC-29, LC-34)* — after T-23
  **Done when:** `forcedChange.successChangesCredentialClearsFlagRevokesOthersNotifies` (policy and age sources), `forcedChange.afterCompromiseSendsNoMessageAndAuditsSkip` and `forcedChange.resubmitReturnsRecordedOutcome` pass.

- [ ] **T-25 · Challenge refusal and abandonment [001]** — `VerifyTotpUseCase` and `VerifyRecoveryCodeUseCase` **[001]** refuse a transaction in state `FORCED_CHANGE_REQUIRED` (`auth.challenge.void` semantics are not used: the refusal is a state error that leaves the transaction intact); `AbandonChallengeUseCase` **[001]** voids a transaction in the new state and leaves the requirement pending. *(D8, D16; RF-7.10, RF-8.12, RF-8.13, LC-25, LC-26)* — after T-12, T-22
  **Done when:** `challenge.refusesForcedChangeState` and `abandon.leavesRequirementPending` pass.

- [ ] **T-26 · `ApplyCompromiseResponseCommand`** — internal command with no HTTP surface: set a `COMPROMISE_RESPONSE` requirement, `bumpSecurityVersion`, `RevokeSessionsPort.revokeAll`, enqueue `compromise-response-notice` through `SecurityNotificationPort` in the configured default language, audit `FORCED_CHANGE_SET{source, actor}`, `SESSIONS_REVOKED` and `NOTIFICATION_ENQUEUED`; idempotent per identity: a second application while a compromise requirement is pending revokes again but sends no second notice. *(plan §2.1 `ApplyCompromiseResponseCommand`, D10, D13; RF-7.1, RF-7.2, RF-7.5, RF-10.1, LC-19, LC-35)* — after T-19
  **Done when:** `compromiseResponse.setsRequirementRevokesSessionsLeavesRunAuthority`, `compromiseResponse.enqueuesNoticeOnce` and `compromiseResponse.whileSignedOutSendsNoticeAndRequirementMetAtNextSignIn` pass.

## D · Infrastructure adapters

- [ ] **T-27 · Persistence adapters** — implementations (persistence access API per A-35) of `ResetLinkRepository` with `consumeIfLive` as one statement (`UPDATE … SET used_at WHERE token_hash = ? AND used_at IS NULL AND superseded_at IS NULL AND expires_at > now`), `ForcedChangeRequirementRepository`, `ResetOutcomeStore`; reset dimensions on the existing `ThrottleStore` **[001]**; `changed_at` / `policy_version` on the credential mapping **[001]**; `FORCED_CHANGE_REQUIRED` on the transaction mapping **[001]**. *(plan §2.1 Persistence, D1, D2, D9; RF-2.3, RF-2.5, RF-5.1, RF-7.1, RNF-3, LC-6)* — after T-02…T-04, T-14
  **Done when:** repository tests against real PostgreSQL pass for every method, the partial unique index rejects a second pending requirement through the adapter, and `consumeIfLive` returns exactly one success when called twice concurrently on the same hash.

- [ ] **T-28 · Session revocation adapter** — `RevokeSessionsPort` against the existing session service: revokes every session and refresh family of the identity; relies on the bumped `securityVersion` for any credential already issued; no change to JWT / refresh mechanics; run-scoped authority untouched (A-39). *(plan §2.1 Session revocation adapter, D7, §6 A-39; RF-5.5, RF-7.2, RF-8.9, RNF-10, LC-11, LC-12, LC-13, LC-19)* — after T-19, T-14
  **Done when:** `complete.revokesAllSessionsFamiliesAndVoidsTransactions` and `complete.ignoresExistingSessionAndRevokesSameIdentity` pass against the session service's test double, and an accepted Run stub is not cancelled by a revocation.

- [ ] **T-29 · Email adapter and outbox wiring** — the three templates behind the existing outbox relay **[002]**; `password-reset` and `password-changed` rendered in the locale of the request header; `compromise-response-notice` in `ResetPolicy`'s default language; the link on the public origin only; relay failures retried by the relay, never surfaced. *(plan §2.1 Outbox relay and email adapter, D4, D13, D14; RF-2.4, RF-2.8, RF-5.7, RF-7.2, RNF-11, RNF-13, LC-32)* — after T-05, T-15
  **Done when:** `resetRequest.enqueuesMessageToVerifiedAddressOnPublicOrigin`, `resetRequest.outboxFailureNotVisibleInResponse` and `complete.enqueuesPasswordChangedNotification` pass, and the message language follows the header for request and complete while the compromise notice uses the configured default.

- [ ] **T-30 · Audit adapter** — map the events `RESET_REQUESTED{matched, issued}`, `RESET_LINK_ISSUED`, `RESET_LINK_CONSUMED`, `RESET_LINK_REJECTED{cause}`, `CREDENTIAL_CHANGED{cause}`, `FORCED_CHANGE_SET{source, actor?}`, `FORCED_CHANGE_COMPLETED`, `FORCED_CHANGE_ATTEMPT_FAILED`, `SESSIONS_REVOKED{cause}`, `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_SKIPPED{reason}`, `RESTRICTION_APPLIED`, `RESTRICTION_LIFTED` to `AuditPort` with identity id when matched, correlation id, network key, timestamp, outcome kind; a serialisation guard rejects any payload holding a token, link, password, unmatched address or message body; entries are written in the effect's transaction. *(plan §2.1 Audit adapter; RF-10.1, RF-10.2, RF-10.5)* — after T-14
  **Done when:** `audit.entryPerEvent` (thirteen events) and `audit.containsNoSecretOrUnmatchedAddress` pass.

- [ ] **T-31 · Logging and problem redaction** — request-body logging excludes `token`, `currentPassword`, `newPassword`, `confirmPassword`; problem details never echo them; traces and analytics carry only the correlation id. *(D19; RF-10.3, RNF-4)* — after T-30
  **Done when:** `logs.containNoTokenOrPassword` passes across request, validate, complete and forced-change with secrets present in every request.

- [ ] **T-32 · Compromise-response entry** — an administrative command runnable only from the deployment environment with an actor identifier that invokes `ApplyCompromiseResponseCommand`; no HTTP endpoint, no tenant screen; placeholder until A-01 / A-44 decide the operator path. *(plan §2.1 Compromise-response entry, D10, §6 A-44; RF-7.2, RF-7.5)* — after T-26
  **Done when:** an integration test runs the command with an actor id and observes the requirement, the revocation and the audit entry carrying the actor, and the API-surface enumeration shows no route maps to it.

## E · API

- [ ] **T-33 · `POST /api/auth/password-reset/request`** — `{email}` with the UI locale in a request header → 202 `{}` in every branch (active, unverified, restricted, erasing, unknown, throttled) or 400 problem with field paths for an empty or malformed address; server-side normalisation; no redirect. *(plan §2.1 `PasswordController`, D4, D19; RF-1.4, RF-1.5, RF-1.6, RF-2.1, RF-3.3, RNF-1)* — after T-17
  **Done when:** `passwordApi.requestFieldPathProblems` and `resetRequest.responseIdenticalAcrossClasses` pass (identical status, body and headers for the six classes).

- [ ] **T-34 · `POST /api/auth/password-reset/validate` and `/complete`** — validate `{token}` → 200 `{}` or 410 `auth.reset-link.unavailable`; complete `{token, newPassword, confirmPassword, idempotencyKey}` with the locale header → 200 `{state: RESET_DONE}`, 400 field-path problems (`auth.password.policy`, `auth.password.same-as-current`, `auth.password.mismatch`) or 410 `auth.reset-link.unavailable`; both endpoints require no session, ignore one if present, set no cookie and carry no identity data. *(D3, D19; RF-4.1, RF-4.5, RF-4.6, RF-4.7, RF-4.8, RF-5.2, RF-5.8, RF-6.1, RF-6.2, RF-6.3, RNF-8)* — after T-21
  **Done when:** `complete.fieldPathProblems`, `unavailable.oneCodeForEveryCause`, `resetResponses.exposeNoIdentityData` and `complete.issuesNoSession` pass.

- [ ] **T-35 · `POST /api/auth/challenge/forced-change` and the `FORCED_CHANGE` sign-in response [001]** — `{transaction, idempotencyKey, currentPassword, newPassword, confirmPassword}` → 200 `{state: LIMITED, transaction}`, 200 `{state: COMPLETED, landing}`, 400 `auth.forced-change.rejected` (no cause, no attempt count), 400 field-path problems or 410 `auth.challenge.void`; `POST sign-in` **[001]** gains 200 `{state: FORCED_CHANGE, transaction}`; while in `FORCED_CHANGE_REQUIRED` the transaction token authorises only this endpoint and `DELETE challenge` **[001]**; neither response nor transaction carries the source. *(D8, D16, D19; RF-7.6, RF-7.7, RF-7.8, RF-8.3, RF-8.6, RF-8.7, RF-8.8, RF-8.10, RF-10.5)* — after T-24, T-25
  **Done when:** `forcedChangeState.onlyForcedChangeAndAbandonAllowed`, `signIn.failedFirstFactorRevealsNoRequirement` and `audit.matchedFlagNeverInResponse` pass.

- [ ] **T-36 · Absence checks** — a test that enumerates the API surface and asserts no endpoint sets or clears a forced-change requirement, resets another identity's password, or reads a factor, recovery-code, membership or run table from this feature's use cases. *(D10; RF-5.9, RF-7.5, RF-8.11, RNF-9)* — after T-35, T-32
  **Done when:** `noTenantSurfaceWritesRequirement` passes and the enumeration lists no administrative reset endpoint.

## F · Front end

- [ ] **T-37 · `contracts/password-reset` types and guards** — request / response / problem types with runtime guards for the four new endpoints and the `FORCED_CHANGE` sign-in state; pure TypeScript; a guard rejects any response carrying a token, password, email, display name or source field. *(plan §2.2 `contracts/password-reset`; RF-4.7, RF-8.3, RF-10.3, RNF-8)* — after T-01
  **Done when:** guard unit tests accept every documented response and problem shape and reject a fixture with an extra `token`, `password`, `email` or `source` field.

- [ ] **T-38 · i18n bundle `identity-access.{en,es,fr}`: reset and forced-change keys** — every heading, label, action, status, validation message and statement of R03, R07, the D02 unavailable entry and R08 (RF-1.1, RF-1.2, RF-4.3, RF-6.2, RF-8.2), the "Request received" statement, the reset-done confirmation and the problem codes of T-33…T-35. *(D13, §6 S-37; RF-1.1, RF-1.2, RF-4.3, RF-6.2, RF-8.2, RF-9.1, RNF-13)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value.

- [ ] **T-39 · Hooks** — `useRequestReset`, `useValidateResetLink`, `useCompleteReset`, `useForcedChange` (mutations with retry disabled and the UI locale header attached; the two completing mutations generate their idempotency key once per mount and reuse it on re-submission), `usePasswordPolicy` **[002]** reused. *(plan §2.2 hooks and State, D15, D18; RF-1.6, RF-4.2, RNF-13, LC-31)* — after T-37
  **Done when:** unit tests show each mutation sends the current locale, has retry disabled, and the same idempotency key is sent on a second call of the same mounted hook.

- [ ] **T-40 · Holders** — token holder (module-private in-memory value read from the `/reset-password` route on mount, dropped on route leave); reset-done flag (in-memory boolean set by a successful complete, consumed once by `/sign-in`); transaction holder **[001]** extended with the `FORCED_CHANGE` state and cleared on completed, void, abandon and route leave with `DELETE challenge` fired on leave. *(D15, D16; RF-5.8, RF-7.7, RF-8.12, RF-8.13, RF-10.4, RNF-4)* — after T-39
  **Done when:** `storage.containsNothingFromFeature` passes (storage spies see no writes), the token is absent after route leave, and the flag reads true once then false.

- [ ] **T-41 · `ForgotPasswordPage`** — R03 in two states: initial (email field, "Request reset instructions", "Return to sign in", Privacy / Terms, language switch, the statements of RF-1.2) and received (the "Request received" banner above the same form, focus moved to the banner, address kept, action available — S-36); client validation of empty or malformed address with an associated message; trim before submit; "Need help?" opening N43 HelpShortcutsPanel (`430:4129`) (S-17). *(plan §2.2 `ForgotPasswordPage`, §6 S-36, S-17; RF-1.1, RF-1.2, RF-1.3, RF-1.4, RF-1.5, RF-1.6, RF-6.4)* — after T-38, T-39
  **Done when:** `forgotPasswordPage.rendersInitialAndReceivedStates` (desktop, tablet, mobile) and `forgotPasswordPage.blocksEmptyOrMalformedEmail` pass, and a second submission from the received state calls the mutation again.

- [ ] **T-42 · `ResetPasswordPage` — valid and updating states** — validate the token on mount (once, under strict mode); valid state with new-password and confirmation fields with show toggles, `PasswordRulesHint` **[002]** from the server policy, the statements of RF-4.3, "Update password", "Return to sign in", Privacy / Terms, language switch; client checks for mismatch, empty fields and displayed policy; updating state disables resubmission; on 200 set the reset-done flag and route to `/sign-in`; renders no identity data. *(plan §2.2 `ResetPasswordPage`, D3, D12, D15; RF-4.1, RF-4.2, RF-4.3, RF-4.4, RF-4.5, RF-4.6, RF-4.7, RF-5.3, RF-8.4, LC-10, LC-31)* — after T-40
  **Done when:** `resetPasswordPage.validatesOnMountThenShowsForm`, `resetPasswordPage.blocksMismatchAndPolicyFailure`, `resetPasswordPage.reloadResubmitsSameKey`, `passwordRulesHint.rendersServerPolicy` and `passwordField.showToggleNeverPersists` pass.

- [ ] **T-43 · `ResetPasswordPage` — unavailable state, and `ResetDoneNotice`** — unavailable state as drawn in D02 ("This reset link can't be used", the link is invalid, used or expired, no MFA reset and no automatic sign-in, "Request a new reset link" → `/forgot-password` with an empty field, "Return to sign in"), reached on a 410 at mount or on submit without stating the cause; `ResetDoneNotice` rendered once above the sign-in form **[001]** when the flag is set (S-35); the route is public and reads no session. *(plan §2.2, D15, §6 S-35; RF-4.8, RF-5.8, RF-6.1, RF-6.2, RF-6.3, RF-6.4)* — after T-42
  **Done when:** `resetPasswordPage.unavailableStateWithRoutes`, `resetDone.routesToSignInWithNoticeOnce` and `resetPasswordRoute.isPublic` pass.

- [ ] **T-44 · `ForceChangePasswordPage`** — R08: "Required security update" marker, current / new / confirmation fields with show toggles, `PasswordRulesHint`, the three statements of RF-8.2, "Update & continue", "Sign out safely" (→ `DELETE challenge`, holder cleared, `/sign-in`), Privacy / Terms, language switch; `GenericFailureMessage` **[001]** on the current-password field for `auth.forced-change.rejected` with the password fields cleared; same-as-current and mismatch as field messages; route leave or browser back = abandon; `auth.challenge.void` → `/sign-in` with no reason text; nothing rendered about the source. *(plan §2.2 `ForceChangePasswordPage`, D16, §6 S-37; RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-8.6, RF-8.7, RF-8.12, RF-8.13, LC-25, LC-26)* — after T-40
  **Done when:** `forceChangePage.rendersFieldsActionsStatementsNoSource`, `forceChangePage.signOutSafelyAbandons` and `forceChangePage.leaveAbandons` pass, and after a mocked rejection all three password fields are empty with focus on the current-password message.

- [ ] **T-45 · Routing handoffs** — sign-in `FORCED_CHANGE` → `/sign-in/update-password`; the challenge route **[001]** refuses a holder in `FORCED_CHANGE` (no bypass by URL); forced-change `LIMITED` → `/sign-in/challenge` **[001]**; `COMPLETED` → `LandingRouter` **[001]**; reset `RESET_DONE` → `/sign-in`; R02's "Forgot password" **[001]** → `/forgot-password`; the spec 002 security message target → `/forgot-password`; unavailable → `/forgot-password` with an empty field. *(plan §2.2 Routing, D16; RF-1.3, RF-5.8, RF-6.4, RF-7.6, RF-7.10, RF-8.7, RF-8.10, LC-27, LC-28)* — after T-43, T-44
  **Done when:** `signInPage.forgotPasswordRoutesToRequest` passes and route unit tests show each handoff above, including that `/sign-in/challenge` with a `FORCED_CHANGE` holder redirects to `/sign-in/update-password`.

- [ ] **T-46 · `SecurityStatements` extension [001]** — add the R03, R07, D02 and R08 statement blocks as shared components used by the three pages at every width. *(plan §2.2 `SecurityStatements`, D17; RF-1.2, RF-4.3, RF-6.2, RF-8.2, RF-9.1, RNF-14)* — after T-41, T-43, T-44
  **Done when:** `securityStatements.presentOnResetAndForcedChange` passes and a test renders each page at desktop and mobile widths and finds identical statement text in both.

- [ ] **T-47 · Reflow and touch targets** — Tailwind breakpoints for the three pages and the unavailable state; `shared/ui` field, password field, button and toggle at 44 px minimum on touch widths; visible labels and focus; breakpoint layout replaceable per screen (S-03). *(D17, §6 S-03; RF-9.1, RF-9.2, RF-9.3, RNF-12, RNF-14)* — after T-46
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow on the four surfaces and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-48 · Timing-comparability harness** — N samples per request class (active, unverified, restricted, erasing, unknown, throttled) and per unavailable cause (unknown, malformed, expired, consumed, superseded); distributions overlap within the configured tolerance. *(D3, D4; RF-2.1, RF-6.2, RNF-1)* — after T-33, T-34
  **Done when:** `resetRequest.timingComparable` and `unavailable.timingComparable` pass on the CI runner for three consecutive runs.

- [ ] **T-49 · Concurrency suite** — two-tab submission of one link; two parallel first-factor sign-ins for a flagged identity; request race on one address. *(D2, D8; RF-5.1, RNF-3, LC-6, LC-29)* — after T-27, T-24
  **Done when:** `complete.twoTabsOneSuccessOneUnavailable` and `parallelFlaggedSignIns.firstChangeVoidsOther` pass ten times in a row with real PostgreSQL, and the request race yields exactly one live link.

- [ ] **T-50 · Throttle suite** — independence from sign-in in both directions, window lift with a controllable clock, no network exemption on the address key, sign-in never restricted by a reset flood. *(D5; RF-3.2, RF-3.4, RF-3.7, RNF-2, LC-33)* — after T-17, T-27
  **Done when:** `throttle.resetAndSignInIndependentBothWays`, `throttle.resetWindowLiftsAndSignInNeverRestricted` and `throttle.noNetworkExemptionOnAddressKey` pass.

- [ ] **T-51 · Startup suite** — one case per property of T-01 plus the cross-field case. *(RF-3.6, RF-3.7, RF-7.4, RNF-6)* — after T-01
  **Done when:** `resetPolicy.startupFailsOnAbsentOrUnboundedValue` and `resetPolicy.startupFailsWhenAddressWindowNotShorterThanNetworkWindow` pass, each failing the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-52 · Fail-closed and scope suite** — absent, expired, consumed, superseded or malformed token, absent policy and absent configuration all refuse the effect; both flows touch only identity tables: factor, recovery codes, memberships, suspension, ownership, status and runs unchanged. *(D2, D6, D9; RF-5.9, RF-8.11, RNF-7, RNF-9, LC-15, LC-16, LC-17, LC-18)* — after T-24, T-27
  **Done when:** `failClosed.absentExpiredConsumedSupersededOrUnconfiguredRefuses`, `changeCredential.touchesOnlyIdentityTables` and `factor.untouchedByBothFlows` pass.

- [ ] **T-53 · FE e2e journeys** — Playwright journeys for US-1 … US-8 in EN, ES and FR at desktop, tablet and mobile widths with a fake email port exposing the link: request → received; open link → reset → sign-in with confirmation → first factor with the new password → challenge (factor enrolled) or landing (no factor); old, used and superseded link → unavailable → request a new link with an empty field; flagged sign-in → R08 → challenge or landing; R08 "Sign out safely" and browser back; language switch on each screen keeping token and transaction. *(RF-5.10, RF-6.4, RF-7.6, RF-8.10, RNF-13, LC-15, LC-16, LC-27, LC-28, LC-30; US-1 … US-8)* — after T-47, T-35
  **Done when:** `afterReset.nextSignInChallengesEnrolledFactor`, `afterReset.noFactorLandsByMembership`, `forcedChange.continuesToChallengeOrLanding`, `unavailable.requestNewLinkOpensEmptyRequest` and `languageSwitch.onEachScreenKeepsTokenAndTransaction` pass in the three languages and three widths.

- [ ] **T-54 · Automated accessibility** — axe on the request, reset, unavailable-link and forced-change surfaces at the three widths and three languages. *(RF-9.1, RF-9.2, RNF-12, RNF-13, RNF-14)* — after T-53
  **Done when:** `a11y.resetAndForcedChange.{en,es,fr}.{desktop,tablet,mobile}` reports zero critical or serious violations on all 36 combinations.

- [ ] **T-55 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on the four surfaces: focus order, field error association, "Request received" announced without focus theft, generic rejection announced on R08, labels, 44 px targets; versions recorded in the release-support matrix. *(RF-9.2, RNF-12; FR-X-040)* — after T-54
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-56 · Adversarial suite (US-9)** — enumeration across request, validate and complete; lockout attempts by reset flood and by wrong current passwords on R08 against the holder's sign-in; reuse of a used link, replay after a newer request, forged and extended tokens; MFA bypass through a reset or a forced change; search for an administrator reset surface. *(D1, D2, D5, D10; RF-2.1, RF-3.2, RF-5.1, RF-5.9, RF-6.2, RF-7.5, RF-8.11, RNF-1, RNF-2, RNF-3, RNF-9; US-9)* — after T-36, T-49, T-50
  **Done when:** every attack test fails to achieve its goal and asserts the generic response or the unavailable state.

## H · Closing

- [ ] **T-57 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-48 … T-56
  **Done when:** the script reports 124 of 124 identifiers (75 RF, 14 RNF, 35 LC) covered by at least one passing test.

- [ ] **T-58 · Register-dependency review** — walk §6 of the plan (A-44, A-45, A-33, A-39, A-27, A-35, S-35, S-36, S-37, S-14, S-03, S-17): for each, confirm the built assumption matches the current register state or record the deferral. *(spec §7 item 13)* — after T-57
  **Done when:** each of the twelve items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-59 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run id, timing reports, a11y report, manual checklist, coverage report, register review). — after T-58
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-38, T-41 |
| RF-1.2 | T-38, T-41, T-46 |
| RF-1.3 | T-41, T-45 |
| RF-1.4 | T-33, T-41 |
| RF-1.5 | T-16, T-33, T-41 |
| RF-1.6 | T-33, T-39, T-41 |
| RF-2.1 | T-16, T-33, T-48, T-56 |
| RF-2.2 | T-03, T-07, T-08, T-15, T-16 |
| RF-2.3 | T-03, T-15, T-27 |
| RF-2.4 | T-15, T-29 |
| RF-2.5 | T-08, T-15, T-27 |
| RF-2.6 | T-07, T-16 |
| RF-2.7 | T-14, T-16 |
| RF-2.8 | T-05, T-15, T-29 |
| RF-3.1 | T-04, T-13, T-17 |
| RF-3.2 | T-13, T-50, T-56 |
| RF-3.3 | T-13, T-17, T-33 |
| RF-3.4 | T-13, T-17, T-50 |
| RF-3.5 | T-17 |
| RF-3.6 | T-01, T-51 |
| RF-3.7 | T-01, T-13, T-50, T-51 |
| RF-4.1 | T-18, T-34, T-42 |
| RF-4.2 | T-39, T-42 |
| RF-4.3 | T-38, T-42, T-46 |
| RF-4.4 | T-42 |
| RF-4.5 | T-20, T-34, T-42 |
| RF-4.6 | T-20, T-34, T-42 |
| RF-4.7 | T-18, T-34, T-37, T-42 |
| RF-4.8 | T-34, T-43 |
| RF-5.1 | T-03, T-08, T-20, T-27, T-49, T-56 |
| RF-5.2 | T-06, T-20, T-34 |
| RF-5.3 | T-08, T-18, T-42 |
| RF-5.4 | T-06, T-19, T-20 |
| RF-5.5 | T-14, T-19, T-28 |
| RF-5.6 | T-10, T-19, T-20 |
| RF-5.7 | T-05, T-19, T-29 |
| RF-5.8 | T-20, T-34, T-40, T-43, T-45 |
| RF-5.9 | T-14, T-19, T-20, T-36, T-52, T-56 |
| RF-5.10 | T-53 |
| RF-6.1 | T-08, T-09, T-18, T-34, T-43 |
| RF-6.2 | T-09, T-18, T-34, T-38, T-43, T-46, T-48, T-56 |
| RF-6.3 | T-09, T-21, T-34, T-43 |
| RF-6.4 | T-41, T-43, T-45, T-53 |
| RF-6.5 | T-13, T-18, T-21 |
| RF-7.1 | T-04, T-10, T-26, T-27 |
| RF-7.2 | T-05, T-26, T-28, T-29, T-32 |
| RF-7.3 | T-02, T-06, T-11, T-22 |
| RF-7.4 | T-01, T-02, T-06, T-11, T-22, T-51 |
| RF-7.5 | T-14, T-26, T-32, T-36, T-56 |
| RF-7.6 | T-04, T-12, T-22, T-35, T-45, T-53 |
| RF-7.7 | T-12, T-35, T-40 |
| RF-7.8 | T-22, T-35 |
| RF-7.9 | T-22 |
| RF-7.10 | T-12, T-22, T-25, T-45 |
| RF-8.1 | T-44 |
| RF-8.2 | T-38, T-44, T-46 |
| RF-8.3 | T-10, T-24, T-35, T-37, T-44 |
| RF-8.4 | T-42, T-44 |
| RF-8.5 | T-06, T-12, T-23, T-24 |
| RF-8.6 | T-12, T-23, T-35, T-44 |
| RF-8.7 | T-12, T-23, T-35, T-44, T-45 |
| RF-8.8 | T-06, T-23, T-35 |
| RF-8.9 | T-10, T-19, T-24, T-28 |
| RF-8.10 | T-12, T-24, T-35, T-45, T-53 |
| RF-8.11 | T-19, T-24, T-36, T-52, T-56 |
| RF-8.12 | T-12, T-25, T-40, T-44 |
| RF-8.13 | T-25, T-40, T-44 |
| RF-9.1 | T-38, T-46, T-47, T-54 |
| RF-9.2 | T-47, T-54, T-55 |
| RF-9.3 | T-47 |
| RF-10.1 | T-09, T-15, T-16, T-19, T-21, T-23, T-26, T-30 |
| RF-10.2 | T-30 |
| RF-10.3 | T-31, T-37 |
| RF-10.4 | T-40 |
| RF-10.5 | T-16, T-30, T-35 |
| RNF-1 | T-09, T-16, T-33, T-48, T-56 |
| RNF-2 | T-13, T-17, T-23, T-50, T-56 |
| RNF-3 | T-03, T-27, T-49, T-56 |
| RNF-4 | T-03, T-31, T-40 |
| RNF-5 | T-17 |
| RNF-6 | T-01, T-51 |
| RNF-7 | T-01, T-21, T-52 |
| RNF-8 | T-34, T-37 |
| RNF-9 | T-14, T-19, T-52, T-56 |
| RNF-10 | T-19, T-28 |
| RNF-11 | T-05, T-29 |
| RNF-12 | T-47, T-54, T-55 |
| RNF-13 | T-05, T-29, T-38, T-39, T-53, T-54 |
| RNF-14 | T-46, T-47, T-54 |
| LC-1 | T-07, T-16 |
| LC-2 | T-07, T-16 |
| LC-3 | T-07, T-16 |
| LC-4 | T-15 |
| LC-5 | T-18 |
| LC-6 | T-27, T-49 |
| LC-7 | T-21 |
| LC-8 | T-18 |
| LC-9 | T-20 |
| LC-10 | T-20, T-42 |
| LC-11 | T-28 |
| LC-12 | T-28 |
| LC-13 | T-28 |
| LC-14 | T-19, T-20 |
| LC-15 | T-52, T-53 |
| LC-16 | T-52, T-53 |
| LC-17 | T-52 |
| LC-18 | T-52 |
| LC-19 | T-26, T-28 |
| LC-20 | T-11, T-22 |
| LC-21 | T-11, T-22 |
| LC-22 | T-22 |
| LC-23 | T-12, T-23 |
| LC-24 | T-12, T-23 |
| LC-25 | T-25, T-44 |
| LC-26 | T-25, T-44 |
| LC-27 | T-24, T-45, T-53 |
| LC-28 | T-24, T-45, T-53 |
| LC-29 | T-12, T-24, T-49 |
| LC-30 | T-53 |
| LC-31 | T-03, T-12, T-20, T-23, T-39, T-42 |
| LC-32 | T-29 |
| LC-33 | T-17, T-50 |
| LC-34 | T-19, T-24 |
| LC-35 | T-26 |
| US-1 … US-8 | T-53 (journeys) |
| US-9 | T-56 (adversarial suite) |

Self-check: all 75 RF criteria, 14 RNF and 35 limit cases of `specPasswordReset.md` v0.2 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.
