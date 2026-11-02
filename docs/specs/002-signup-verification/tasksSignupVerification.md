# Tasks 002 — Signup and email verification

**Feature** Local identity creation and email verification · **Tasks version** 0.1 · **Date** 26 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specSignupVerification.md` v0.1 (requirements RF-x.y, RNF-n, LC-n) and `planSignupVerification.md` v0.1 (modules §2, decisions D1–D16, tests §5). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first. Tasks marked **[001]** reuse an element that plan 001 (Login) builds; if 001 is not built yet, that element is built here in the same shape. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 001's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing.

---

## A · Foundations

- [ ] **T-01 · Signup configuration record** — `SignupPolicy` value object and `SignupPolicyProperties` binding (link expiry, unverified window, resend steps and restriction windows, name bounds, Terms version, Privacy version, public origin, notice send window, pending-reference lifetime, expiry-job cadence and batch size), each `@Validated` as present, positive and bounded. *(plan §2.1 `SignupPolicy`, D11; RF-4.6, RF-10.2, RNF-4)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed or set unbounded (one failing case per property).

- [ ] **T-02 · Migration: identity extension and acknowledgement table** — plan 001's migration already creates `identity` (with `email_verified_at` and the unique index on the normalised email), `identity_credential` and `signin_throttle`; this migration only adds `created_or_replaced_at` to `identity` and creates `identity_acknowledgement` (identity id, `terms_version`, `privacy_version`, `acknowledged_at`). *(plan §2.1 Persistence, D7, D12; RF-9.1, RF-10.1)* — after T-01
  **Done when:** the migration applies after plan 001's on an empty database and on one with existing rows; plan 001's unique index still rejects two identities with equal normalised email.

- [ ] **T-03 · Migration: `verification_link`** — columns `id`, `identity_id`, `token_hash` (unique), `purpose`, `issued_at`, `expires_at`, `superseded_at`, `used_at`; index on `identity_id`. *(plan §2.1 Persistence, D3; RF-4.1, RF-4.4)* — after T-02
  **Done when:** the migration applies; `token_hash` uniqueness is enforced; `identity_id` cascades on identity delete.

- [ ] **T-04 · Migration: `pending_reference`** — `pending_reference` (`ref_hash` unique, `identity_id`, `issued_at`, `expires_at`, `revoked_at`). The throttle table keeps plan 001's shape (primary key `(dimension, subject_key)`, no constraint on the dimension values); the resend dimensions are new key values, nothing else. *(plan §2.1 Persistence, D6, D8; RF-5.4, RF-6.3, RF-7.5)* — after T-03
  **Done when:** the migration applies; a throttle row with dimension `RESEND_ADDRESS` can be inserted and plan 001's sign-in rows are unaffected.

- [ ] **T-05 · Outbox templates registered** — register template ids `verification` and `signup-attempt-notice` with the outbox message type and the three locales, the D04 copy (`437:4431`) for the notice (S-23). *(plan §2.1 Email adapter, D1, D12; RF-4.5, RNF-7)* — after T-01
  **Done when:** rendering each template in EN, ES and FR with sample variables produces a non-empty subject and body, and the verification body contains the three statements of RF-4.5.

## B · Domain (pure, no Spring)

- [ ] **T-06 · `EmailAddress` value object** — trim, lower-case, shape validation, equality on the normalised form. *(plan §2.1; RF-1.4, RF-1.6, LC-16)*
  **Done when:** unit test `emailAddress.normalisation` passes for mixed case, surrounding spaces, and rejects malformed input.

- [ ] **T-07 · `DisplayName` and `TermsAcknowledgement` value objects** — trimmed non-blank name within configured bounds; acknowledgement of Terms version, Privacy version, time. *(plan §2.1, D11; RF-1.4, RF-9.1, LC-17)* — after T-01
  **Done when:** a whitespace-only name is rejected; a name at the configured bounds is accepted; the acknowledgement holds the versions given.

- [ ] **T-08 · `IdentityStatus` branches and `GlobalIdentity` factory** — `createUnverified(name, email, credential, now)` sets `UNVERIFIED` and `createdOrReplacedAt` (the acknowledgement is a separate record, T-07); the signup branch by status: `UNVERIFIED` → replace, `ACTIVE` → notice, `RESTRICTED` / `ERASING` → no change. *(plan §2.1, §6 A-27; RF-3.1, RF-3.7)* — after T-06, T-07
  **Done when:** a table test shows the branch for each status and the factory produces an `UNVERIFIED` identity with all fields set.

- [ ] **T-09 · `GlobalIdentity.replacePending` and `readdress`** — both allowed only while `UNVERIFIED`; replacement updates name, credential, acknowledgement and `createdOrReplacedAt`; re-addressing updates the email. *(plan §2.1, D7; RF-3.3, RF-7.2, RF-7.6, RF-10.4)* — after T-08
  **Done when:** unit tests pass for the `UNVERIFIED` case and both methods throw for a verified identity.

- [ ] **T-10 · `GlobalIdentity.markVerified`** — sets status `ACTIVE` and `emailVerifiedAt`; idempotence is not the domain's job (the use case decides), so a second call on a verified identity throws. *(plan §2.1; RF-8.1, RF-10.3)* — after T-08
  **Done when:** unit test shows the transition from `UNVERIFIED` and a throw from any other status.

- [ ] **T-11 · `VerificationLink` entity** — fields of T-03; `isNewestAndLive(now)`, `supersede(now)`, `markUsed(now)`. *(plan §2.1, D3; RF-4.1, RF-4.4, RF-8.1, RF-8.2, RF-8.5)* — after T-01
  **Done when:** unit tests cover live, expired, superseded and used states, and `markUsed` on a superseded link throws.

- [ ] **T-12 · `VerificationOutcome`** — `VERIFIED_NOW`, `ALREADY_VERIFIED`, `UNAVAILABLE(cause)` with causes `EXPIRED`, `SUPERSEDED`, `MALFORMED`, `NO_IDENTITY`; a single `toResponse()` that maps every `UNAVAILABLE` cause to the same value. *(plan §2.1, D5; RF-8.5, RF-8.6, RF-12.1)*
  **Done when:** unit test shows all four causes map to one identical response and the cause is retained only for audit.

- [ ] **T-13 · Resend throttle dimensions on `AbuseThrottlePolicy` [001]** — add dimension keys `RESEND_ADDRESS` and `RESEND_NETWORK` fed by the `SignupPolicy` steps; no change to the policy arithmetic. *(plan §2.1, D8; RF-6.3, RF-7.5, LC-11)* — after T-01
  **Done when:** unit test shows delay and restriction computed from the resend steps, and that sign-in dimension state does not affect resend decisions.

- [ ] **T-14 · `PendingReference` value and lifecycle rules** — random reference, hash, identity binding, `expiresAt`, `revoke()`; rule that a reference is valid only while unrevoked, unexpired and its identity is `UNVERIFIED`. *(plan D6, §6 A-34; RF-5.4, RF-7.4)* — after T-01
  **Done when:** unit tests cover valid, expired, revoked and identity-not-pending cases.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-25…T-28); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-15 · Port interfaces** — `IdentityRepository` (`findByNormalisedEmail`, `save`, `deleteDueUnverified`), `AcknowledgementRepository`, `VerificationLinkRepository` (`findByTokenHash`, `supersedeAllLive`, `save`, `verifyIfLive` conditional update), `PendingReferenceStore`, `TokenGenerator`, `TokenHasher`, `PasswordPolicyPort`, `TransactionalEmailPort`, `SecurityNotificationPort`, `AuditPort`, `Clock`, `JobScheduler`; `ThrottleStore` and `PasswordVerifier` reused **[001]**. *(plan §2.1 Ports; enables RF-3, RF-4, RF-6–RF-8, RF-10, RF-12; RF-3.5 — no tenant, membership or grant port exists)* — after T-06…T-14
  **Done when:** the application module compiles with no dependency on Spring or persistence classes (enforced by an ArchUnit module-boundary test, test-only).

- [ ] **T-16 · `IssueVerificationLink` service** — generate token, store hash, supersede all live links of the identity, enqueue the `verification` message with locale, address and a link built from `SignupPolicy.publicOrigin`, audit `LINK_ISSUED{cause}` for every cause (signup, replacement, resend, re-addressing). *(plan §2.1, D1, D3, D13; RF-4.1–4.5, RF-12.1, LC-4)* — after T-15
  **Done when:** integration test `issueLink.supersedesEarlierLinks` passes and the enqueued message's link starts with the configured public origin regardless of the request host.

- [ ] **T-17 · `SignUpUseCase` — validation and the new-address branch** — field validation (field-path errors), password policy check through `PasswordPolicyPort`, one password encoding, create identity and its acknowledgement record, issue link (which audits `LINK_ISSUED{SIGNUP}`), issue pending reference, audit `IDENTITY_CREATED`, return `{pendingRef, email}`. *(plan §2.1, D2, D10, D11; RF-1.3, RF-1.4, RF-2.2–2.4, RF-3.1, RF-3.5, RF-9.1, RF-9.2, RNF-10)* — after T-16
  **Done when:** integration tests `signUp.newAddressCreatesUnverifiedAndIssuesLink` and `credential.storedNonReversibly` pass; `signUp.writesNoTenantMembershipOrGrant` shows no row in any tenant, membership or grant table.

- [ ] **T-18 · `SignUpUseCase` — pending-collision branch** — `replacePending`, supersede links, issue link, new pending reference (old revoked), audit `IDENTITY_REPLACED`; same response. *(D2, D7; RF-3.3, RF-10.4, LC-2)* — after T-17
  **Done when:** integration test `signUp.pendingCollisionReplacesAndSupersedesLinks` passes and the pre-replacement link opens as unavailable.

- [ ] **T-19 · `SignUpUseCase` — `ACTIVE` collision and ignored branches** — `ACTIVE`: no change, `SecurityNotificationPort` enqueues `signup-attempt-notice` once per configured window, audit `SIGNUP_ATTEMPT_NOTICE_SENT`; `RESTRICTED` / `ERASING`: no change, no message, audit `SIGNUP_ATTEMPT_IGNORED`; both still run the password encoding and return the same response shape with an unbound decoy reference. *(D2, D12; RF-3.2, RF-3.4, RF-3.6, RF-3.7, LC-1, LC-24, LC-25)* — after T-18
  **Done when:** integration tests `signUp.verifiedCollisionChangesNothingAndNotifies`, `signUp.nonRegistrableChangesNothingNoNotice` and `signUp.noAutoLinkNoDomainJoin` pass, and a second verified collision inside the window enqueues no second notice.

- [ ] **T-20 · `ResendVerificationUseCase`** — evaluate `RESEND_ADDRESS` and `RESEND_NETWORK` throttles; if an `UNVERIFIED` identity exists → `IssueVerificationLink` + audit `LINK_ISSUED{RESEND}`; otherwise nothing; always the same empty response. *(D8; RF-6.1–6.4, LC-10, LC-11)* — after T-16, T-13
  **Done when:** integration tests `resend.issuesNewLinkForPending`, `resend.unknownOrVerifiedSendsNothingSameResponse` and `resend.throttledSendsNothingSameResponse` pass.

- [ ] **T-21 · `ReaddressPendingIdentityUseCase`** — resolve the pending reference (T-14 rules; no pending identity → nothing and no notice, RF-7.4 before RF-7.3); normalise the new address; equal to the requester's own pending address → handled as a resend; free → `readdress`, supersede links, issue to the new address, new reference, audit `IDENTITY_READDRESSED`; owned by an `ACTIVE` identity → notice to the owner; otherwise nothing; counts on the resend throttle; same response with the new address. *(D6, D8, D12; RF-7.1–7.6, LC-12–LC-15)* — after T-20
  **Done when:** integration tests `readdress.freeAddressReaddressesAndReissues`, `readdress.collisionChangesNothing`, `readdress.verifiedOwnerNotified`, `readdress.confirmedOrExpiredIdentityChangesNothing`, `readdress.countsAsResend` and `readdress.sameAddressIsResend` pass.

- [ ] **T-22 · `OpenVerificationLinkUseCase`** — hash the token; load; `verifyIfLive` conditional update (`used_at IS NULL AND status = 'UNVERIFIED'`); on success supersede the other links, revoke pending references, audit `VERIFIED`; used-and-newest-unexpired → `ALREADY_VERIFIED`; otherwise `UNAVAILABLE(cause)` with audit `LINK_OPEN_FAILED{cause}`; never a session. *(D4, D5; RF-8.1, RF-8.2, RF-8.5, RF-8.6, RF-8.8, RF-11.3, RNF-9, LC-3, LC-7, LC-8, LC-9)* — after T-16
  **Done when:** integration tests `openLink.verifiesOnceAndTouchesOnlyIdentityAndLinks`, `openLink.supersededIsUnavailable` and `failClosed.absentExpiredSupersededMalformedNeverVerify` pass.

- [ ] **T-23 · `PasswordPolicyQuery`** — returns the current rules in a display shape from `PasswordPolicyPort`. *(D10; RF-2.1)* — after T-15
  **Done when:** the query returns exactly the rule set the port enforces (one test comparing both against the same policy instance).

- [ ] **T-24 · `ExpireUnverifiedIdentitiesJob`** — durable job claiming `UNVERIFIED` identities with `createdOrReplacedAt + window < now` in bounded batches; delete with links, references and throttle rows; audit `UNVERIFIED_IDENTITY_REMOVED`; predicate excludes any other status. *(D9; RF-10.1, RF-10.3, RF-10.4, LC-9)* — after T-15, T-01
  **Done when:** integration tests `expiryJob.removesDueUnverifiedOnly` and `expiryJob.freesAddressAndKillsLinks` pass with a controllable clock, and a verified identity older than the window is untouched.

## D · Infrastructure adapters

- [ ] **T-25 · Persistence adapters** — implementations (persistence access API per A-35) of `IdentityRepository`, `AcknowledgementRepository`, `VerificationLinkRepository` (including the single-statement conditional update), `PendingReferenceStore`; resend dimensions on the existing `ThrottleStore` **[001]**. *(plan §2.1 Persistence, D4, D7; RF-1.6, RF-3.3, RF-4.4, RF-8.1, RF-8.2, LC-6, LC-16)* — after T-02…T-04, T-15
  **Done when:** repository tests against real PostgreSQL pass for every method, `signUp.sameAddressDifferentCaseIsOneIdentity` passes, and `verifyIfLive` returns exactly one success when called twice concurrently (`openLink.twoTabsOneVerifiesBothConfirmed`).

- [ ] **T-26 · Crypto adapters** — `TokenGenerator` (256-bit CSPRNG, URL-safe encoding) and `TokenHasher` (cryptographic hash) over `sharedsupport/crypto`; the same pair serves pending references. *(D3, D6; RF-4.1, RNF-2)* — after T-15
  **Done when:** test `token.entropyAndHashOnlyAtRest` shows 256 bits of output, distinct tokens across 10 000 draws, and no clear token in the link or reference tables.

- [ ] **T-27 · Email adapter and outbox wiring** — `TransactionalEmailPort` implementation behind the existing outbox relay; `SecurityNotificationPort` implemented as a second outbox message type; the relay's idempotency key is the message id. *(D1, D12; RF-4.2, RNF-5)* — after T-05, T-15
  **Done when:** tests `verificationMessage.addressedToIdentityWithStatements` and `verificationLink.usesPublicOriginNotRequestHost` pass with a fake provider, and a relay retry sends the same message id once.

- [ ] **T-28 · Audit adapter** — map the eight events of plan §2.1 to `AuditPort` with identity id, correlation id, network key, timestamp; a serialisation guard rejects any payload containing the password, token or message body fields. *(RF-12.1, RF-12.2, RNF-8)* — after T-15
  **Done when:** tests `audit.entryPerEvent` and `audit.containsNoSecretOrMessageBody` pass.

- [ ] **T-29 · Logging and problem redaction** — request-body logging excludes `password` and `token`; problem details never echo them; a log appender test asserts absence. *(RF-12.3, RNF-8)* — after T-28
  **Done when:** test `logs.containNoPasswordOrToken` passes across sign-up, resend, readdress and open with secrets present in the requests.

- [ ] **T-30 · Password policy port adapter** — adapter over the existing server password policy; one instance serves display and validation. *(D10; RF-2.1, RF-2.2, LC-18)* — after T-17, T-23
  **Done when:** test `signUp.enforcesCurrentPolicyRegardlessOfClient` passes: a password accepted under the displayed rules is rejected after the policy is tightened in the same test.

- [ ] **T-31 · Job registration** — register `ExpireUnverifiedIdentitiesJob` with `sharedsupport/jobs` at the configured cadence; batch size from configuration. *(D9; RF-10.1)* — after T-24
  **Done when:** the job is listed by the job registry at startup and runs once in an integration test when triggered.

## E · API

- [ ] **T-32 · `GET /api/auth/password-policy`** — returns the display rules. *(RF-2.1)* — after T-30
  **Done when:** contract test `passwordPolicy.displayedFromServer` passes with the rule set of the configured policy.

- [ ] **T-33 · `POST /api/auth/sign-up`** — request record `{name, email, password, acknowledgement}` with the UI locale in a request header; 400 problem with field paths; 202 `{pendingRef, email}` for every branch. *(D2, D12; RF-1.3, RF-1.4, RF-2.3, RF-3.4, RF-9.2, LC-17, LC-19)* — after T-19, T-30
  **Done when:** contract tests `signUpApi.fieldPathProblems`, `signUp.namesUnmetRules` and `signUp.responseIdenticalAcrossClasses` pass (identical shape, status and timing for new, pending, verified and non-registrable addresses; the reference value is random in every branch).

- [ ] **T-34 · `POST /api/auth/verification/resend` and `/readdress`** — resend `{email}` → 202 `{}`; readdress `{pendingRef, newEmail}` → 202 `{pendingRef, email}`; the UI locale in a request header on both; an invalid or absent reference yields the same 202. *(D6; RF-6.4, RF-7.1, RF-7.4, RF-7.6)* — after T-20, T-21
  **Done when:** contract tests `pendingReference.authorisesReaddressOnly` and `readdress.verifiedIdentityImpossible` pass, and a request with a forged reference returns the same body as a valid one while changing nothing.

- [ ] **T-35 · `POST /api/auth/verification/open`** — `{token}` → 200 `{state: CONFIRMED, email}` or 410 problem `auth.verification-link.unavailable`; no `Set-Cookie`, no session. *(D4, D5; RF-8.1, RF-8.2, RF-8.4, RF-8.5, RF-8.6, RF-8.8, RF-8.9, RNF-10)* — after T-22
  **Done when:** contract tests `openLink.unavailableSingleCodeAllCauses` and `noEndpoint.issuesSession` pass (identical 410 body for all four causes; no session or cookie on any response of this feature).

- [ ] **T-36 · Sign-in handoff: pending reference on `auth.verification-pending` [001]** — plan 001 D9's problem response carries `{pendingRef, email}` issued by `PendingReferenceStore` after the password is proven. *(D6; RF-5.3, LC-20, LC-21)* — after T-14, T-25
  **Done when:** tests `signIn.unverifiedLandsOnPending` and `signIn.unverifiedWrongPasswordGeneric` pass: the correct password yields a reference, the wrong password yields the generic failure.

- [ ] **T-37 · Absence checks** — a test that enumerates the API surface and asserts no endpoint creates, verifies or alters another person's identity, and that the signup path touches no invitation or factor table. *(RF-11.1–11.4)* — after T-35
  **Done when:** tests `verification.touchesNoInvitationOrFactor` and `noAdminEndpointExists` pass.

## F · Front end

- [ ] **T-38 · `contracts/signup` types and guards** — request / response / problem types for the five endpoints with runtime guards; pure TypeScript. *(plan §2.2; RF-12.3, RNF-8 — a guard rejects any response carrying a token or password)*
  **Done when:** guard unit tests accept every documented response and reject a response with an extra `token` or `password` field.

- [ ] **T-39 · i18n bundle `identity-access.{en,es,fr}`: signup and verification keys** — every statement, heading, status, action, message and validation text of RF-1, RF-5, RF-8 and RF-6.1. *(D15, D16; RNF-7, RF-9.4)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value.

- [ ] **T-40 · Hooks** — `useSignUp`, `useResend`, `useReaddress`, `useOpenLink` (mutations, no retry, UI locale header attached), `usePasswordPolicy` (query). *(plan §2.2 State, D15; RF-2.1, RF-6.4, RNF-7, LC-23)* — after T-38
  **Done when:** unit tests show each mutation sends the current locale and has retry disabled.

- [ ] **T-41 · Pending holder** — module-private in-memory store of `{pendingRef, email}`; set by the sign-up response or the sign-in handoff; cleared on route leave; never written to storage. *(D14; RF-5.4, RNF-8)* — after T-40
  **Done when:** unit test `holder.clearsOnLeaveAndNeverTouchesStorage` passes (storage spies see no writes).

- [ ] **T-42 · `SignupPage` — form and validation** — name, email, password with show toggle, acknowledgement checkbox (never pre-ticked), "Create account & verify email", field-level messages with focus, trim before submit; policy rules from `usePasswordPolicy`. *(D10, D16; RF-1.1, RF-1.3–1.6, RF-2.1, RF-9.2, LC-17, LC-19)* — after T-39, T-40
  **Done when:** unit tests `signupPage.blocksInvalidFields` and `passwordField.showToggleNeverPersists` pass, and the checkbox renders unticked on every mount.

- [ ] **T-43 · `SignupPage` — statements, routes and submit handling** — the three statements of RF-1.2, "Already have an account? Sign in", Privacy / Terms routes (external operator documents, S-14), language switch; "Need help?" link opening N43 HelpShortcutsPanel (`430:4129`) (S-17); on 202 store the holder and route to `/verify-email`; on 400 map field paths. *(RF-1.1, RF-1.2, RF-1.7, RF-2.3, RF-3.1, RF-9.4)* — after T-42, T-41
  **Done when:** unit test `signupPage.rendersFieldsStatementsAndRoutes` passes and a mocked 202 lands on the pending route with the address shown.

- [ ] **T-44 · `VerifyEmailPage` — pending state** — heading, status, address from the holder, generic text, "Resend verification email" (neutral confirmation on success, identical on every response), "Use a different email", "Back to sign in", the four statements of RF-5.2, the Privacy / Terms routes; after a reload with no holder, an address field for resend and no re-addressing (S-25). *(D6, D14; RF-5.1, RF-5.2, RF-6.1, RF-6.4, RF-9.4)* — after T-41, T-39
  **Done when:** unit test `verifyEmailPage.pendingStateContent` passes and the confirmation text after resend is identical for a 202 regardless of mocked server behaviour.

- [ ] **T-45 · `ReaddressEntry`** — inline field under the pending state (S-25); submits `{pendingRef, newEmail}` with the locale header; on 202 updates the holder's address and re-renders pending. *(D6; RF-7.1, RF-7.2)* — after T-44
  **Done when:** unit test `readdressEntry.rendersAndSubmits` passes and the pending state shows the new address after a mocked 202.

- [ ] **T-46 · `VerifyEmailPage` — confirmed and unavailable states** — confirmed: heading, status, address, three statements, "Continue to sign in" → `/sign-in` with an empty form (S-27); unavailable: heading, status, two statements, "Request a new verification link" → pending resend with an address field, "Back to sign in". *(RF-8.3, RF-8.4, RF-8.5, RF-8.7, RF-9.4)* — after T-39
  **Done when:** unit test `verifyEmailPage.confirmedStateContent` passes and both states contain every statement listed in RF-8.3 and RF-8.5.

- [ ] **T-47 · Link opener route `/verify-email/open`** — reads the token from the route, submits once on mount, renders confirmed on 200 and unavailable on 410, never stores the token, leaves the route after rendering. *(D4, D14; RF-8.1, RF-8.2, RF-8.5, RF-8.9, RF-12.3)* — after T-46, T-40
  **Done when:** unit test `opener.doesNotStoreToken` passes and the mutation is called exactly once per mount under React strict mode.

- [ ] **T-48 · `SecurityStatements` extension [001]** — add the signup, pending, confirmed and unavailable statement blocks as shared components used by both pages. *(D16; RF-1.2, RF-5.2, RF-8.3, RF-8.5, RF-13.1)* — after T-43, T-44, T-46
  **Done when:** a test renders each page at desktop and mobile widths and finds identical statement text in both.

- [ ] **T-49 · Reflow and touch targets** — Tailwind breakpoints for both pages; `shared/ui` field, checkbox and button at 44 px minimum on touch widths; visible focus. *(D16; RF-13.1, RF-13.2, RNF-11)* — after T-48
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-50 · Timing-comparability harness: signup** — N samples per address class (new, pending, verified, non-registrable); distributions overlap within the configured tolerance. *(D2; RF-3.4, RNF-1)* — after T-33
  **Done when:** test `signUp.timingComparable` passes on the CI runner for three consecutive runs.

- [ ] **T-51 · Timing-comparability harness: resend, readdress, open** — same harness for the three other operations. *(D1, D5; RF-6.4, RF-8.6, RNF-1)* — after T-34, T-35
  **Done when:** tests `resend.timingComparable`, `readdress.timingComparable` and `openLink.timingComparable` pass on three consecutive runs.

- [ ] **T-52 · Concurrency suite** — two-tab open (`openLink.twoTabsOneVerifiesBothConfirmed`), re-open idempotence (`openLink.reopenIsIdempotent`, `openLink.firstOpenVerifiesLaterOpensConfirm`), two signups for one address, resend race. *(D4, D7; RF-8.2, RNF-3, LC-5, LC-6, LC-16)* — after T-25, T-22
  **Done when:** all five tests pass ten times in a row with real PostgreSQL.

- [ ] **T-53 · Throttle independence and lift** — `resend.independentOfSignInAndReset` and a lift-after-window test with a controllable clock. *(D8; RF-6.3, LC-11)* — after T-20
  **Done when:** both tests pass and the sign-in throttle state is unchanged by a resend flood.

- [ ] **T-54 · Outbox and locale suite** — `message.languageFollowsRequestLocale` (switch between signup and resend), `signup-attempt-notice` send-window test. *(D12, D15; RF-4.5, RNF-7, LC-23, LC-1, LC-13)* — after T-27
  **Done when:** both tests pass with EN, ES and FR.

- [ ] **T-55 · Startup suite** — `signupPolicy.startupFailsOnAbsentOrUnboundedValue` with one case per property of T-01. *(RF-4.6, RF-10.2, RNF-4)* — after T-01
  **Done when:** every case fails the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-56 · Terms suite** — `acknowledgement.versionsAndTimeStored`, `termsChange.noEffectOnExistingIdentity`. *(D11; RF-9.1, RF-9.3)* — after T-17
  **Done when:** both tests pass; changing the configured versions and restarting alters nothing on an existing identity.

- [ ] **T-57 · FE e2e journeys** — Playwright journeys for US-1…US-9 in EN, ES and FR at desktop, tablet and mobile widths, with a fake email port exposing the link: signup → pending → resend → open → confirmed → sign-in; unavailable → request new link; re-address; sign-in with unverified identity → pending; fresh browser open (`openLink.worksInFreshBrowser`); `unavailable.requestNewLinkOpensResend`; `continueToSignIn.noSession`. *(RF-1–RF-9, RF-13, LC-20–LC-22)* — after T-49, T-36
  **Done when:** all journeys pass in the three languages and three widths.

- [ ] **T-58 · Automated accessibility** — axe on signup, pending, confirmed and unavailable at the three widths and three languages (`a11y.signupAndVerification.{en,es,fr}.{desktop,tablet,mobile}`). *(RF-13.1, RF-13.2, RNF-6, RNF-11)* — after T-57
  **Done when:** zero critical or serious violations on all 36 combinations.

- [ ] **T-59 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist on the four surfaces: focus order, field error association, live confirmation announced without focus theft, labels, 44 px targets; versions recorded in the release-support matrix. *(RNF-6; FR-X-040)* — after T-58
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-60 · Adversarial suite (US-10)** — enumeration across all four operations, pending-identity takeover with a forged or pre-replacement reference, scanner pre-open, replay of a used link after a newer one exists. *(D5, D6; RF-3.4, RF-6.4, RF-7.4, RF-8.6, US-10)* — after T-34, T-35, T-52
  **Done when:** every attack test fails to achieve its goal and asserts the generic response.

## H · Closing

- [ ] **T-61 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of §5.2 of the plan and this file. *(spec §7 item 2)* — after T-50…T-60
  **Done when:** the script reports 100 of 100 identifiers covered by at least one passing test.

- [ ] **T-62 · Register-dependency review** — walk §6 of the plan (A-27, A-28, A-33, A-34, A-35, S-14, S-17, S-23, S-24, S-25, S-26, S-27): for each, confirm the built assumption matches the current register state or record the deferral. *(spec §7 item 11)* — after T-61
  **Done when:** each of the twelve items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-63 · Completion criteria sign-off** — tick spec §7 items 1–11 with the evidence (test run id, a11y report, checklist, coverage report). — after T-62
  **Done when:** all eleven items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-42, T-43 |
| RF-1.2 | T-43, T-48 |
| RF-1.3 | T-17, T-33, T-42 |
| RF-1.4 | T-06, T-07, T-17, T-33, T-42 |
| RF-1.5 | T-42 |
| RF-1.6 | T-06, T-25, T-42 |
| RF-1.7 | T-43 |
| RF-2.1 | T-23, T-30, T-32, T-40, T-42 |
| RF-2.2 | T-17, T-30 |
| RF-2.3 | T-17, T-33, T-43 |
| RF-2.4 | T-17 |
| RF-3.1 | T-08, T-17, T-43 |
| RF-3.2 | T-19 |
| RF-3.3 | T-09, T-18, T-25 |
| RF-3.4 | T-19, T-33, T-50, T-60 |
| RF-3.5 | T-15, T-17 |
| RF-3.6 | T-19 |
| RF-3.7 | T-08, T-19 |
| RF-4.1 | T-03, T-11, T-16, T-26 |
| RF-4.2 | T-16, T-27 |
| RF-4.3 | T-16 |
| RF-4.4 | T-03, T-11, T-16, T-25 |
| RF-4.5 | T-05, T-16, T-54 |
| RF-4.6 | T-01, T-55 |
| RF-5.1 | T-44 |
| RF-5.2 | T-44, T-48 |
| RF-5.3 | T-36 |
| RF-5.4 | T-04, T-14, T-41 |
| RF-6.1 | T-20, T-44 |
| RF-6.2 | T-20 |
| RF-6.3 | T-04, T-13, T-20, T-53 |
| RF-6.4 | T-20, T-34, T-44, T-51, T-60 |
| RF-7.1 | T-21, T-34, T-45 |
| RF-7.2 | T-09, T-21, T-45 |
| RF-7.3 | T-21 |
| RF-7.4 | T-14, T-21, T-34, T-60 |
| RF-7.5 | T-04, T-13, T-21 |
| RF-7.6 | T-09, T-21, T-34 |
| RF-8.1 | T-10, T-11, T-22, T-25, T-35, T-47 |
| RF-8.2 | T-11, T-22, T-25, T-35, T-47, T-52 |
| RF-8.3 | T-46, T-48 |
| RF-8.4 | T-35, T-46 |
| RF-8.5 | T-11, T-12, T-22, T-35, T-46, T-47, T-48 |
| RF-8.6 | T-12, T-22, T-35, T-51, T-60 |
| RF-8.7 | T-46 |
| RF-8.8 | T-22, T-35 |
| RF-8.9 | T-35, T-47 |
| RF-9.1 | T-02, T-07, T-17, T-56 |
| RF-9.2 | T-17, T-33, T-42 |
| RF-9.3 | T-56 |
| RF-9.4 | T-39, T-43, T-44, T-46 |
| RF-10.1 | T-02, T-24, T-31 |
| RF-10.2 | T-01, T-55 |
| RF-10.3 | T-10, T-24 |
| RF-10.4 | T-09, T-18, T-24 |
| RF-11.1 | T-37 |
| RF-11.2 | T-37 |
| RF-11.3 | T-22, T-37 |
| RF-11.4 | T-37 |
| RF-12.1 | T-12, T-16, T-28 |
| RF-12.2 | T-28 |
| RF-12.3 | T-29, T-38, T-47 |
| RF-13.1 | T-48, T-49, T-58 |
| RF-13.2 | T-49, T-58 |
| RNF-1 | T-50, T-51 |
| RNF-2 | T-26 |
| RNF-3 | T-52 |
| RNF-4 | T-01, T-55 |
| RNF-5 | T-27 |
| RNF-6 | T-58, T-59 |
| RNF-7 | T-05, T-39, T-40, T-54 |
| RNF-8 | T-28, T-29, T-38, T-41 |
| RNF-9 | T-22 |
| RNF-10 | T-17, T-35 |
| RNF-11 | T-49, T-58 |
| LC-1 | T-19, T-54 |
| LC-2 | T-18 |
| LC-3 | T-22 |
| LC-4 | T-16 |
| LC-5 | T-52 |
| LC-6 | T-25, T-52 |
| LC-7 | T-22 |
| LC-8 | T-22 |
| LC-9 | T-22, T-24 |
| LC-10 | T-20 |
| LC-11 | T-13, T-20, T-53 |
| LC-12 | T-21 |
| LC-13 | T-21, T-54 |
| LC-14 | T-21 |
| LC-15 | T-21 |
| LC-16 | T-06, T-25, T-52 |
| LC-17 | T-07, T-33, T-42 |
| LC-18 | T-30 |
| LC-19 | T-33, T-42 |
| LC-20 | T-36, T-57 |
| LC-21 | T-36, T-57 |
| LC-22 | T-57 |
| LC-23 | T-40, T-54 |
| LC-24 | T-19 |
| LC-25 | T-19 |
| US-1 … US-9 | T-57 (journeys) |
| US-10 | T-60 (adversarial suite) |

Self-check: all 64 RF criteria, 11 RNF and 25 limit cases of the specification appear above; every task names at least one requirement.
