# Plan 004 — Password reset and forced password change

**Feature** Self-service password reset by emailed link, and the forced password change state · **Plan version** 0.2 · **Date** 26 October 2026 · **Status** Draft for review — aligned with specification v0.2 (decisions Q7–Q10) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/004-PasswordReset/specPasswordReset.md` v0.2 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-06 as the subject; F-IDN-01…F-IDN-05, F-IDN-07…F-IDN-09, F-IDN-16 and F-SET-03 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG) and FE board `46:7847` (feature-modular SPA DAG) · **Builds on** plan 001 (Login) and plan 002 (Signup and verification); elements they own are tagged **[001]** / **[002]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's ten decisions (Q1–Q10) are built as decided; Q7–Q10 closed the product doubts of its first draft (flood suppression window, newest-link-only, session-agnostic reset screen, no password-changed message after a compromise-forced change). Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-44, A-45, A-33, A-39, A-27, S-35, S-36, S-37, S-14, S-03, S-17; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | Neither flow reads, stores or replays any Run, approval or mapping intent. The credential-change service touches identity tables only. | RF-5.9, RF-8.11 |
| **P3** The browser expresses intent, never authority | The FE submits an address, a token or a password and renders the server's answer. Eligibility, link validity, the forced-change requirement, attempt counting and the post-change route are all decided server-side inside the login transaction of plan 001. | RF-2, RF-5, RF-7.6–7.10, RF-8.10, RNF-8 |
| **P6** Fail closed | An unknown, expired, consumed or superseded token, an absent policy, an absent requirement record or an absent configuration value refuses the effect; no default is assumed and no fallback exists. | RF-6, RF-3.6, RNF-6, RNF-7 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security `PasswordEncoder` and session support, the existing transactional outbox. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. Nothing is added: no cache store, no queue, no new identity provider. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: both flows live in the existing hexagonal bounded context `identity`; outer adapters depend inward; the session service, audit and shared support are reached only through published ports; mail goes through the transactional outbox. FE: the same feature module as Login; TanStack Query owns server facts; Zustand is not used (no unsaved fork). | §2 |
| **P12** Two independent tenancy gates | Reset links, forced-change requirements and credential metadata are **platform** rows of a global identity; they carry no `tenant_id` and are protected by application authority. No tenant, membership, grant or run table is read or written. | RF-2.7, RF-5.9, RF-8.11, LC-17, LC-18 |
| **P13** Configured values are never defaulted | Token expiry, maximum password age, throttle steps, restriction windows and the per-address mail cap are one validated configuration record; startup fails on any absent or unbounded value. | RF-3.6, RF-7.4, RNF-6 |
| **P14** Release bar | WCAG 2.2 AA on the four surfaces with automated and manual assistive evidence; EN / ES / FR for screens and both messages. | RF-9, RNF-12, RNF-13, RNF-14 |
| **F-X-01** C10 context | No C10 is derived anywhere in this feature: the reset screen is public and the forced change runs inside the limited identity state. Tenant entry after the change re-derives C10 exactly as plan 001 RF-8.5. | RF-4.7, RF-7.7, RF-8.10 |
| **F-X-02** Request integrity | Request, validate, complete and forced-change submissions are consequential writes with no generic auto-retry; the two completing submissions carry an idempotency key whose outcome is recorded and returned on re-submission; a lost answer leads to a lookup of that outcome, never a second change. | RF-5.1, RF-8.5, LC-31 |
| **F-X-03** Secrets and safe problems | Tokens exist only in the message and in the request that presents them; at rest only their hash; passwords only in the request body and the verifier; problem details carry safe codes, field paths and a correlation id; browser storage holds nothing from this feature. | RF-2.3, RF-10.3, RF-10.4, RNF-4 |
| **F-X-05** Audit log | Every event of RF-10.1 is appended through the `audit` context port, redacted, in the same transaction as the effect. | RF-10 |
| **F-X-06** Configuration validation and email port | `ResetPolicyProperties` is part of startup validation; both messages go through `TransactionalEmailPort` behind the outbox. | RF-2.4, RF-3.6, RNF-11 |
| **F-X-07** Responsive surfaces | M15 (`431:4243`), M16 (`431:4271`) and M17 (`431:4300`) are the drawn counterparts; tablet follows the D01 responsive policy with `SecurityStatements` shared across widths. | RF-9 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity` (shared with Login and Signup)

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 001 and 002. This plan adds the reset and forced-change use cases, two aggregates and one new outbound port. Contexts touched — `audit`, `sharedsupport`, and the session service of F-IDN-08 — are reached only through their published ports (allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `GlobalIdentity` **[001]** | Unchanged shape (email, display name, status, `emailVerifiedAt`, `securityVersion`). `isResetEligible()` = status `ACTIVE` and email verified (A-27, §6). `bumpSecurityVersion()` is called by every credential change, which voids every open login transaction at its next use (plan 001 D1) and is the version the session service checks (FR-IDN-061). | RF-2.2, RF-2.6, RF-5.5, RF-7.3 |
| | `Credential` **[001]** | Extended with `changedAt` and `policyVersion` (the version of the server policy the password satisfied when set). New method `replace(newEncoded, now, policyVersion)` returns the new credential; `matches(raw, verifier)` is reused for the current-password check and the same-as-current check. | RF-5.2, RF-5.4, RF-7.3, RF-7.4, RF-8.5, RF-8.8 |
| | `ResetLink` | Token hash, identity id, purpose `PASSWORD_RESET`, `issuedAt`, `expiresAt`, `supersededAt`, `usedAt`. Methods `isNewestAndLive(now)`, `markUsed(now)`, `supersede(now)`. Pure; mirrors `VerificationLink` **[002]** without sharing its table (different purpose, different lifetime). | RF-2.2, RF-2.5, RF-5.1, RF-5.3, RF-6.1 |
| | `ResetOutcome` | Result of validate or complete: `VALID`, `DONE`, `UNAVAILABLE(cause)` with causes `UNKNOWN`, `MALFORMED`, `EXPIRED`, `USED`, `SUPERSEDED`; `toResponse()` maps every cause to one value — the cause is for audit only. | RF-6.1–6.3, RF-10.1 |
| | `ForcedChangeRequirement` | Identity id, `source` (`COMPROMISE_RESPONSE`, `POLICY_CHANGE`, `PASSWORD_AGE`), `setAt`, `clearedAt`. At most one pending (uncleared) requirement per identity; a second trigger while one is pending records an audit event only. Methods `clear(now)`, `isPending()`. | RF-7.1–7.5, RF-5.6, RF-8.9 |
| | `ForcedChangeRule` | Pure function `(credential.changedAt, credential.policyVersion, currentPolicyVersion, maxPasswordAge, now) → Optional<source>`: `POLICY_CHANGE` when the credential's policy version is older than the current one, else `PASSWORD_AGE` when `now − changedAt > maxPasswordAge`, else none. Evaluated only after a correct first factor. | RF-7.3, RF-7.4, LC-20, LC-21 |
| | `LoginTransaction` **[001]** | Gains state `FORCED_CHANGE_REQUIRED` between `FIRST_FACTOR_PASSED` and the challenge: `requireForcedChange()`, `forcedChangeDone(now, newSecurityVersion)` (moves to `FIRST_FACTOR_PASSED` and re-stamps the stored `securityVersion` so the transaction survives its own credential change, LC-29 for the other one), existing `recordFailedAttempt`, `void`, `recordedOutcome(key)`. The idempotency-keyed outcome record covers the forced-change submission. | RF-7.6, RF-7.7, RF-8.5–8.7, RF-8.10, RF-8.12, LC-23, LC-24, LC-29, LC-31 |
| | `AbuseThrottlePolicy` **[001]** | Reused unchanged; dimension keys `RESET_ADDRESS` and `RESET_NETWORK` are added with their own configured steps and windows. The sign-in keys `ACCOUNT` / `NETWORK` are written by the forced-change current-password check (RF-8.6), never by a reset request. | RF-3.1–3.4, RF-6.5, RF-8.6 |
| | `ResetPolicy` | Immutable record: token expiry, maximum password age, reset throttle steps and windows for both keys, per-address message cap and its window, the public origin (shared configuration value with `SignupPolicy` **[002]**), notification send window. | RF-2.2, RF-3.5, RF-3.6, RF-7.4, RNF-6 |
| `identity.application` | `RequestPasswordResetUseCase` | Normalises the address; evaluates both reset throttles; always generates one token and its hash (discarded when ineligible); loads the identity by normalised email; if `isResetEligible()` and the address is under the message cap → `IssueResetLink`; otherwise nothing is written except throttle counters and the audit event `RESET_REQUESTED{matched, issued}`. Same response in every branch. Never touches status, sessions, factor or requirement. | RF-1.5, RF-2.1–2.8, RF-3, RF-10.1, LC-1–LC-4, LC-32, LC-33 |
| | `IssueResetLink` (internal service) | Supersedes every live reset link of the identity, stores the new hash with expiry from `ResetPolicy`, enqueues the `password-reset` message (locale, address, link on the public origin) in the outbox, audits `RESET_LINK_ISSUED`. | RF-2.2–2.5, RF-2.8 |
| | `ValidateResetLinkQuery` | Hashes the presented token, loads the link, returns `VALID` or `UNAVAILABLE(cause)` without changing anything; audits `RESET_LINK_REJECTED{cause}` on failure. Counts a failure on `RESET_NETWORK`. | RF-4.1, RF-5.3, RF-6.1, RF-6.2, RF-6.5, LC-5, LC-8 |
| | `CompletePasswordResetUseCase` | Looks up the recorded outcome for the idempotency key first (LC-31); validates confirmation and policy through `PasswordPolicyPort`; checks same-as-current with `Credential.matches`; then one transaction: conditional consume (`UPDATE … SET used_at WHERE token_hash = ? AND used_at IS NULL AND superseded_at IS NULL AND expires_at > now`) — zero rows → `UNAVAILABLE`; one row → `ChangeCredential(cause = RESET)`. Creates no session. | RF-4.5, RF-4.6, RF-5.1–5.9, RF-6.3, LC-6, LC-7, LC-9, LC-10, LC-13–LC-18 |
| | `ChangeCredential` (internal service, shared by both flows) | In the caller's transaction: `Credential.replace`, `bumpSecurityVersion`, supersede every live reset link, clear any pending requirement, `RevokeSessionsPort.revokeAll(identityId, except)` (`except` = none for a reset; none for a forced change either — the limited identity state is not a session), enqueue the `password-changed` notification through `SecurityNotificationPort` **unless** the cleared requirement's source is `COMPROMISE_RESPONSE` (decision Q10: the notice of RF-7.2 was the notification), in which case audit `NOTIFICATION_SKIPPED{reason = COMPROMISE_NOTIFIED}` instead; audit `CREDENTIAL_CHANGED{cause}` and `SESSIONS_REVOKED{cause}`. | RF-5.4–5.7, RF-8.9, RNF-10, LC-12, LC-13, LC-14, LC-34 |
| | `EvaluateForcedChangeStep` (internal, called by `SignInUseCase` **[001]**) | Runs only after a correct first factor on a verified identity: loads the pending requirement; if none, applies `ForcedChangeRule` and persists a new requirement when it yields a source (audit `FORCED_CHANGE_SET{source}`); if a requirement is pending, moves the new transaction to `FORCED_CHANGE_REQUIRED` and returns the `FORCED_CHANGE` response instead of the limited or completed one. On a failed first factor or an unverified identity it is never reached. | RF-7.3, RF-7.4, RF-7.6–7.10, LC-20–LC-22 |
| | `CompleteForcedChangeUseCase` | Locks the transaction row; checks state `FORCED_CHANGE_REQUIRED`, expiry and attempts; returns the recorded outcome for a repeated idempotency key; verifies the current password with `PasswordVerifier` — wrong → `recordFailedAttempt`, both sign-in throttle keys, audit `FORCED_CHANGE_ATTEMPT_FAILED`, void at zero; same-as-current → field rejection, not counted; policy and confirmation checks; then `ChangeCredential(cause = FORCED)` and `forcedChangeDone`; then branch: factor enrolled → state `FIRST_FACTOR_PASSED`, response `LIMITED` with the same transaction; no factor → `CompleteSignInUseCase` **[001]** → `COMPLETED` with landing. | RF-8.4–8.11, LC-23, LC-24, LC-27–LC-29, LC-31 |
| | `ApplyCompromiseResponseCommand` | Internal application command with no HTTP surface: sets a `COMPROMISE_RESPONSE` requirement, `bumpSecurityVersion`, `RevokeSessionsPort.revokeAll`, enqueues the `compromise-response-notice` message through `SecurityNotificationPort` (a change is required at the next sign-in, every session was signed out, when, what to do if unexpected — decision Q10), audits `FORCED_CHANGE_SET{source, actor}`, `SESSIONS_REVOKED` and `NOTIFICATION_ENQUEUED`. Idempotent per identity: a second application while a compromise requirement is pending revokes again but sends no second notice. Invoked only by the audited operational mechanism of A-44 (§6, D10). | RF-7.2, RF-7.5, LC-19, LC-35 |
| | `AbandonChallengeUseCase` **[001]** | Reused for "Sign out safely" and for leaving R08; voids the transaction, leaves the requirement pending. | RF-8.12, RF-8.13, LC-25, LC-26 |
| | `PasswordPolicyQuery` **[002]** | Reused for the rules shown on R07 and R08. | RF-4.2, RF-8.4 |
| | Ports | New: `ResetLinkRepository`, `ForcedChangeRequirementRepository`, `RevokeSessionsPort` (→ the existing session service, F-IDN-08 / spec 006; spec 003's replacement will reuse it). Reused: `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `PasswordVerifier`, `Clock`, `AuditPort` **[001]**; `TokenGenerator`, `TokenHasher`, `PasswordPolicyPort`, `TransactionalEmailPort` (behind the outbox), `SecurityNotificationPort` **[002]**. | all |
| `identity.infrastructure` | Persistence | `identity_credential` gains `changed_at` and `policy_version` (back-filled at migration with the migration time and the then-current policy version, so no identity is flagged by the migration itself). New `password_reset_link` (token hash unique, identity id, issued / expires / superseded / used timestamps; partial index on live links per identity) and `forced_change_requirement` (identity id, source, set_at, cleared_at; partial unique index on pending rows per identity). `signin_throttle` **[001]** gains the two reset dimension values. `login_transaction` **[001]** gains the `FORCED_CHANGE_REQUIRED` state value. Persistence access API and migration tool per A-35 (placeholders). | RF-2, RF-5, RF-7, RF-8, LC-6, LC-29 |
| | Outbox relay and email adapter **[002]** | Three new templates, `password-reset` (link, expiry statement, no-MFA-reset and no-auto-sign-in statements, "no action needed if you did not request this"), `password-changed` (time, "if this was not you" guidance) and `compromise-response-notice` (change required at next sign-in, sessions signed out, time, "if unexpected" guidance) in EN / ES / FR; content per S-14 (§6). Relay failures are retried by the relay, never surfaced to the request. | RF-2.4, RF-2.8, RF-5.7, RF-7.2, RNF-11, RNF-13, LC-32 |
| | `ResetPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding of every number into `ResetPolicy`; constraints: present, positive, bounded above by a configured ceiling; the per-address suppression window strictly shorter than the network window (RF-3.7, decision Q7); the public origin must be an absolute `https` origin. Any violation fails startup. | RF-3.6, RF-3.7, RF-7.4, RNF-6 |
| | Session revocation adapter | Implements `RevokeSessionsPort` against the existing session service: revokes every session and refresh family of the identity and relies on the bumped `securityVersion` for any token already issued. No change to JWT / refresh mechanics. | RF-5.5, RF-8.9, RNF-10 |
| | Compromise-response entry | The operational mechanism that invokes `ApplyCompromiseResponseCommand` is a placeholder until A-01 / A-44 decide the operator path: in this plan an administrative command runnable only from the deployment environment with an actor identifier, never an HTTP endpoint and never a tenant screen. | RF-7.2, RF-7.5 |
| | Audit adapter | Events: `RESET_REQUESTED{matched, issued}`, `RESET_LINK_ISSUED`, `RESET_LINK_CONSUMED`, `RESET_LINK_REJECTED{cause}`, `CREDENTIAL_CHANGED{cause}`, `FORCED_CHANGE_SET{source, actor?}`, `FORCED_CHANGE_COMPLETED`, `FORCED_CHANGE_ATTEMPT_FAILED`, `SESSIONS_REVOKED{cause}`, `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_SKIPPED{reason}` (RF-10.1: a notification deliberately not sent under RF-8.9), `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]** for the reset keys. Payload: identity id when matched, correlation id, network key, timestamp, outcome kind. Never a token, link, password, unmatched address or message body. | RF-10.1, RF-10.2, RF-10.5 |
| `identity.api` | `PasswordController` — `/api/auth` | `POST password-reset/request` {email} → 202 `{}` always, or 400 problem with field paths · `POST password-reset/validate` {token} → 200 `{}` or 410 problem `auth.reset-link.unavailable` (one code for every cause) · `POST password-reset/complete` {token, newPassword, confirmPassword, idempotencyKey} → 200 `{state: RESET_DONE}` or 400 problem with field paths (`auth.password.policy`, `auth.password.same-as-current`, `auth.password.mismatch`) or 410 `auth.reset-link.unavailable` · `POST challenge/forced-change` {transaction, idempotencyKey, currentPassword, newPassword, confirmPassword} → 200 `{state: LIMITED, transaction}` or 200 `{state: COMPLETED, landing}` or 400 `auth.forced-change.rejected` (no cause, no attempt count) or 400 field-path problems or 410 `auth.challenge.void`. `POST sign-in` **[001]** gains the response `200 {state: FORCED_CHANGE, transaction}`. `DELETE challenge` **[001]** serves "Sign out safely". `GET password-policy` **[002]** serves the rules. The UI locale travels in the request header on request and complete (for the message language). All problems are RFC 9457 problem details with a safe code, field paths and a correlation id. The server never redirects. | RF-1.4, RF-1.6, RF-2.1, RF-4.2, RF-4.5, RF-4.6, RF-5.8, RF-6, RF-7.6, RF-8 |

**Cross-context touch points.** `audit.AuditPort` — append-only. `sharedsupport` — `crypto` (token generation and hashing), `tx`, `contextIds` (correlation id), `outbox`. The session service of F-IDN-08 through `RevokeSessionsPort` only. Nothing reads or writes tenant, membership, grant, approval or run tables.

**Data model note (P12).** `password_reset_link`, `forced_change_requirement` and the credential columns are platform-scoped rows of a global identity and carry no tenant column; RLS does not apply to them.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login and Signup)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/forgot-password` (R03), `/reset-password` (R07; token in the query of the SPA route, read on mount — the consumption path is A-33, §6; the unavailable state is a state of this route), `/sign-in/update-password` (R08, reachable only with a transaction in state `FORCED_CHANGE`). | RF-1, RF-4, RF-6, RF-8 |
| | `useRequestReset()`, `useValidateResetLink()`, `useCompleteReset()`, `useForcedChange()`, `usePasswordPolicy()` **[002]** | Typed intent hooks; attach the UI locale header; return server state verbatim. | RF-1–RF-8 |
| | `contracts/password-reset` (in `src/contracts`) | Request / response / problem types and runtime guards for the four new responses and the `FORCED_CHANGE` sign-in state. | all |
| Views | `ForgotPasswordPage` | R03 in two states: initial (field, action, routes, statements) and received (the "Request received" banner above the same form, focus on the banner, action still available — S-36). | RF-1.1–1.6, RF-9.1 |
| | `ResetPasswordPage` | R07: validates the token on mount; `valid` state with new-password and confirm fields (show toggles), `PasswordRulesHint`, the factor statement, "Update password", "Return to sign in", Privacy / Terms; `unavailable` state as drawn in D02 ("This reset link can't be used", "Request a new reset link", "Return to sign in"); `updating` state disables resubmission. | RF-4.1–4.8, RF-6.1–6.4, RF-9.1 |
| | `ForceChangePasswordPage` | R08: "Required security update" marker, current / new / confirm fields with show toggles, `PasswordRulesHint`, the three statements, "Update & continue", "Sign out safely", Privacy / Terms, language switch; generic rejection on the current-password field. | RF-8.1–8.4, RF-8.6, RF-8.12, RF-9.1 |
| | `PasswordRulesHint` **[002]** | Rules and strength feedback from the server policy. | RF-4.2, RF-8.4 |
| | `SecurityStatements` **[001]** | Extended with the R03, R07, D02 and R08 statements so desktop and reflow cannot drift. | RF-1.2, RF-4.3, RF-6.2, RF-8.2, RF-9.1 |
| | `GenericFailureMessage` **[001]** | Reused for `auth.forced-change.rejected`. | RF-8.6 |
| | `ResetDoneNotice` | One-line confirmation rendered above the sign-in form when the in-memory flag is set (S-35). | RF-5.8 |
| State | TanStack Query | Mutations for request, validate, complete and forced-change (the two completing mutations generate their idempotency key once and reuse it on re-submission); query for the password policy. No retry on mutations (F-X-02). | RF-1–RF-8, LC-31 |
| | Transaction holder **[001]** | Extended with the `FORCED_CHANGE` state; cleared on completed, void, abandon and route leave; `DELETE challenge` fired on leave. | RF-7.7, RF-8.12, RF-8.13, LC-25, LC-26 |
| | Token holder | Module-private in-memory value read from the route on mount of `/reset-password`; never written to storage; dropped on route leave. | RF-10.4, RNF-4 |
| | Reset-done flag | Module-private in-memory boolean set by a successful complete and consumed once by `/sign-in`. | RF-5.8 |
| Routing | Handoffs | Sign-in `FORCED_CHANGE` → `/sign-in/update-password`; forced-change `LIMITED` → `/sign-in/challenge` **[001]**; `COMPLETED` → `LandingRouter` **[001]**; `auth.challenge.void` → `/sign-in` with no reason text; reset `RESET_DONE` → `/sign-in` with the flag; `unavailable` → "Request a new reset link" → `/forgot-password` (empty field); R02's "Forgot password" → `/forgot-password`; spec 002's security message → `/forgot-password`. | RF-1.3, RF-5.8, RF-6.1, RF-6.4, RF-7.6, RF-8.7, RF-8.10 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/errors` (field paths to fields), `shared/i18n` (bundle keys for every statement, state and message in EN / ES / FR), `shared/ui` (accessible `Field`, `PasswordField`, `Button`, `LiveRegion`, 44 px targets). | RF-1.4, RF-4.5, RF-4.6, RF-9.2, RNF-12, RNF-13 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | — | `ForgotPasswordPage` |
| RF-1.2 | — | `SecurityStatements` (generic response, expiry, no MFA reset, no auto sign-in) |
| RF-1.3 | — | Routes from `SignInPage` **[001]** and the spec 002 message target `/forgot-password` |
| RF-1.4 | `PasswordController` field-path problem (server-side validation too) | `Field` validation, associated message, focus |
| RF-1.5 | `RequestPasswordResetUseCase` uses `EmailAddress` **[002]** normalisation | — |
| RF-1.6 | 202 `{}` in every branch | received state: banner, focus, address kept, action available |
| RF-2.1 | `RequestPasswordResetUseCase` constant-work path; one response | — |
| RF-2.2 | `isResetEligible()` → `IssueResetLink`; `TokenGenerator`; expiry from `ResetPolicy` | — |
| RF-2.3 | `TokenHasher`; `password_reset_link.token_hash` only | — |
| RF-2.4 | Outbox `password-reset` to the verified address; link on the configured public origin | — |
| RF-2.5 | `IssueResetLink` supersedes live links first, whoever requested (decision Q8) | — |
| RF-2.6 | Ineligible branch writes throttle and audit only; the generated token is discarded | — |
| RF-2.7 | Use case touches no other row | — |
| RF-2.8 | Template `password-reset` content (S-14) | — |
| RF-3.1 | `AbuseThrottlePolicy` keys `RESET_ADDRESS`, `RESET_NETWORK`; `ThrottleStore` | — |
| RF-3.2 | Distinct keys from `ACCOUNT` / `NETWORK`; sign-in restriction never consulted here | — |
| RF-3.3 | `delayFor` / `restrictionFor` from `ResetPolicy` steps; restricted → no token, no message, same 202 | Same received state |
| RF-3.4 | Windows expire by time; no permanent flag; sign-in keys untouched | — |
| RF-3.5 | Per-address message cap in `ResetPolicy`, checked before `IssueResetLink` | — |
| RF-3.6 | `ResetPolicyProperties` validation at startup | — |
| RF-3.7 | `ResetPolicyProperties` cross-field constraint: address window < network window; no network exemption exists in `AbuseThrottlePolicy` | — |
| RF-4.1 | `ValidateResetLinkQuery` on mount → `VALID` | `ResetPasswordPage` valid state |
| RF-4.2 | `PasswordPolicyQuery` **[002]** | `PasswordRulesHint` |
| RF-4.3 | — | `SecurityStatements` (factor stays protected, return to sign-in, unavailable state exists) |
| RF-4.4 | — | `PasswordField` show toggle; value never persisted |
| RF-4.5 | `auth.password.mismatch` field problem | Client check before submit, associated message |
| RF-4.6 | `auth.password.policy` field problem via `PasswordPolicyPort` | Client check against the displayed rules |
| RF-4.7 | `validate` and `complete` responses carry no identity data | Page renders none |
| RF-4.8 | Endpoints require no session and ignore one if present (decision Q9) | Route is public; no session read |
| RF-5.1 | `CompletePasswordResetUseCase` conditional consume + `ChangeCredential` in one transaction | — |
| RF-5.2 | `Credential.matches` → `auth.password.same-as-current`; no consume | Field message |
| RF-5.3 | `validate` changes nothing; consume only in `complete` success path | Mount calls validate only |
| RF-5.4 | `Credential.replace` through `PasswordVerifier` encoding | — |
| RF-5.5 | `ChangeCredential`: `RevokeSessionsPort`, `bumpSecurityVersion`, supersede links | — |
| RF-5.6 | `ChangeCredential` clears the pending requirement | — |
| RF-5.7 | `SecurityNotificationPort` → outbox `password-changed` (every reset; forced changes except the compromise source, RF-8.9) | — |
| RF-5.8 | 200 `{state: RESET_DONE}`; no session issued | Flag set; route to `/sign-in`; `ResetDoneNotice` |
| RF-5.9 | `ChangeCredential` touches identity tables only; no factor, membership, status or run access | No pending intent stored |
| RF-5.10 | Nothing to build: plan 001 rules apply unchanged | `/sign-in` |
| RF-6.1 | `ResetOutcome.UNAVAILABLE` → 410 `auth.reset-link.unavailable` | `ResetPasswordPage` unavailable state with both routes |
| RF-6.2 | One problem code; cause in audit only | One state, no identity data, statements |
| RF-6.3 | Conditional consume affects zero rows → 410; submitted password discarded | Same unavailable state |
| RF-6.4 | `RequestPasswordResetUseCase` unchanged | `/forgot-password` initial state |
| RF-6.5 | `ValidateResetLinkQuery` and failed `complete` write `RESET_NETWORK` | — |
| RF-7.1 | `ForcedChangeRequirement` row with source and time | — |
| RF-7.2 | `ApplyCompromiseResponseCommand`: requirement, revocation, `compromise-response-notice` | — |
| RF-7.3 | `ForcedChangeRule` → `POLICY_CHANGE` in `EvaluateForcedChangeStep` | — |
| RF-7.4 | `ForcedChangeRule` → `PASSWORD_AGE` from `credential.changedAt` and `ResetPolicy.maxPasswordAge` | — |
| RF-7.5 | No port, endpoint or tenant use case can write `forced_change_requirement` | No control rendered |
| RF-7.6 | `SignInUseCase` **[001]** calls `EvaluateForcedChangeStep` before the limited / completed branch; `FORCED_CHANGE` response | Route to `/sign-in/update-password` |
| RF-7.7 | Transaction token authorises only `challenge/forced-change` and `DELETE challenge` while in `FORCED_CHANGE_REQUIRED` | Page renders no identity data |
| RF-7.8 | Step not reached on a failed first factor; generic problem **[001]** | `GenericFailureMessage` **[001]** |
| RF-7.9 | `SignInUseCase` verification branch **[001]** precedes the step | Verification handoff **[001]** |
| RF-7.10 | `VerifyTotpUseCase` / `VerifyRecoveryCodeUseCase` **[001]** refuse state `FORCED_CHANGE_REQUIRED`; `CompleteSignInUseCase` is reachable only after `forcedChangeDone` | No route bypass: challenge route refuses a `FORCED_CHANGE` holder |
| RF-8.1 | — | `ForceChangePasswordPage` |
| RF-8.2 | — | `SecurityStatements` (security condition, context preserved, no admin password, no Run replay) |
| RF-8.3 | Response and transaction carry no source | Nothing rendered about the source |
| RF-8.4 | `PasswordPolicyQuery` **[002]**; field problems | `PasswordRulesHint`; field checks |
| RF-8.5 | `CompleteForcedChangeUseCase` under row lock; `ChangeCredential` | Single submission with idempotency key |
| RF-8.6 | Wrong current → `recordFailedAttempt`, `ACCOUNT` + `NETWORK` keys, `auth.forced-change.rejected` without cause or count | `GenericFailureMessage` on the field; password fields cleared |
| RF-8.7 | Attempts zero or expiry → `void`; 410 `auth.challenge.void`; requirement untouched | Route to `/sign-in`, holder cleared, no reason text |
| RF-8.8 | `auth.password.same-as-current`; attempt not counted | Field message |
| RF-8.9 | `ChangeCredential(cause = FORCED)`; requirement cleared; notification skipped and audited when the source was `COMPROMISE_RESPONSE` (decision Q10) | — |
| RF-8.10 | `forcedChangeDone`; factor → `LIMITED` same transaction; none → `CompleteSignInUseCase` **[001]** | `LIMITED` → challenge route; `COMPLETED` → `LandingRouter` **[001]** |
| RF-8.11 | `ChangeCredential` scope as RF-5.9; `CompleteSignInUseCase` **[001]** replays nothing | No pending intent stored |
| RF-8.12 | `AbandonChallengeUseCase` **[001]**; requirement untouched | "Sign out safely" → `DELETE challenge`, holder cleared, `/sign-in` |
| RF-8.13 | Same | Route leave / back → abandon |
| RF-9.1 | — | Three pages reflow at mobile and tablet widths; `SecurityStatements` shared |
| RF-9.2 | — | `shared/ui` touch targets, labels, focus |
| RF-9.3 | — | Breakpoint layout replaceable per screen (S-03) |
| RF-10.1 | Audit adapter events in the effect's transaction, including `NOTIFICATION_SKIPPED` | — |
| RF-10.2 | Audit payload shape; unmatched address never stored | Correlation id header |
| RF-10.3 | Verifier inputs and tokens never logged; problem details exclude inputs | No secret in state or analytics |
| RF-10.4 | — | Token holder in memory only; nothing in storage |
| RF-10.5 | `RESET_REQUESTED{matched}` exists in audit only; response identical | — |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · Reset tokens are 256-bit random values stored only as a fast cryptographic hash; one live link per identity, enforced at issue time.**
*Why.* A random high-entropy token needs no work factor: a plain cryptographic hash makes lookup cheap and leaves nothing usable at rest (same reasoning as plan 002 D3). "Newest only" is implemented by superseding every live link when a new one is issued, so the invariant holds without comparing timestamps when the link is used, and a later request by anyone — the holder or a third party — makes earlier links dead (decisions Q4, Q8); the throttles of D5 make repeated invalidation impractical and the holder simply requests again.
*Discarded.* Encoding with the password encoder — needless cost, no gain. Multiple live links until expiry — rejected by Q4 (more windows for an intercepted link). Storing the token in clear — F-X-03.
*Covers.* RF-2.2, RF-2.3, RF-2.5, RNF-4, LC-4, LC-5.

**D2 · Consumption is a conditional update executed inside the credential-change transaction.**
*Why.* Atomic single use under concurrency (RF-5.1, LC-6) needs one statement that both tests and marks the link: `UPDATE … WHERE used_at IS NULL AND superseded_at IS NULL AND expires_at > now`. Exactly one of two concurrent submissions updates a row; the other sees zero rows and receives the unavailable state. Running the credential change in the same transaction means a crash between consume and change cannot leave a consumed link with an unchanged password.
*Discarded.* Read-then-write under optimistic versioning — a retry loop with the same guarantee and more code. A two-phase token exchange (validate issues a short-lived form token) — a second secret with its own lifetime and nothing the conditional update does not already give. Consumption on open — rejected by Q4 (a mail scanner would burn the link).
*Covers.* RF-5.1, RF-5.3, RF-6.3, RNF-3, LC-6, LC-7, LC-8.

**D3 · Opening the link validates the token without consuming it.**
*Why.* RF-4.1 shows the form only for a usable link and RF-6.1 shows the unavailable state otherwise; a non-consuming validation on mount gives the person the right screen immediately instead of after typing a password, costs one hash and one lookup, and changes nothing, so LC-8 (scanner opens first) holds trivially. Failures count on the network key (RF-6.5).
*Discarded.* No validation on open (form always shown, failure on submit) — the person types a new password into a dead link. A consuming open — Q4.
*Covers.* RF-4.1, RF-5.3, RF-6.1, RF-6.5, LC-5, LC-8.

**D4 · The request path performs the same work in every branch and sends mail through the transactional outbox.**
*Why.* RF-2.1 requires identical response, shape and comparable timing across active, unverified, restricted, erasing and unknown addresses. The use case always normalises, evaluates both throttles, generates a token and its hash, and performs one identity lookup; only the eligible branch writes the link and enqueues the message, which is a small bounded write. The outbox (P11; plan 002 D1) keeps provider latency and failure out of the request, so neither timing nor an error can reveal that a message was attempted (LC-32).
*Discarded.* Early return for an unknown address — timing leak. Synchronous send — timing and failure leaks, lost message on crash. A distinct "not eligible" response — Q3.
*Covers.* RF-2.1, RF-2.4, RF-2.6, RF-2.7, RNF-1, RNF-11, LC-1, LC-2, LC-3, LC-32.

**D5 · Reset throttling reuses `AbuseThrottlePolicy` [001] with its own dimension keys and adds a per-address message cap.**
*Why.* The policy is pure and already tested; separate keys (`RESET_ADDRESS`, `RESET_NETWORK`) make the throttle independent of sign-in in both directions (RF-3.2; plan 001 RF-4.7); unusable-link opens and failed submissions count on the network key so token guessing is slowed without ever touching the account's sign-in (RF-6.5). The message cap bounds mail per address per window independently of the delay steps (RF-3.5). Restriction means "no token, no message, same answer", never a different response (RF-3.3). A flood on one address suppresses that holder's message for the address window (decision Q7, accepted residual risk); the configuration validator requires the address window to be shorter than the network window, so the flood's source is cut off first and the holder's suppression ends sooner (RF-3.7); no network is exempt, because an exemption would reveal the holder's network.
*Discarded.* Sharing keys with sign-in — a reset flood would throttle sign-in, and vice versa. A second policy implementation — duplication. Edge rate limiting only — not designed (A-20) and blind to the address dimension. Exempting the holder's own network from the address window — rejected by Q7.
*Covers.* RF-3.1–3.5, RF-3.7, RF-6.5, RNF-2, RNF-5, LC-33.

**D6 · One internal `ChangeCredential` service serves both flows.**
*Why.* RF-5.4–5.7 and RF-8.9 list the same effects: replace the credential, bump the security version, supersede links, clear the requirement, revoke sessions, notify, audit. One service with a `cause` argument guarantees the two flows cannot drift (a later change to one effect reaches both), and makes the effects testable once.
*Discarded.* Two separate sequences — drift. A domain event with asynchronous handlers for revocation and notification — a window in which the old credential is gone but sessions still live, contrary to RNF-10.
*Covers.* RF-5.4–5.7, RF-8.9, RNF-10, LC-12, LC-13, LC-14.

**D7 · Session revocation goes through `RevokeSessionsPort` to the existing session service, and the identity's `securityVersion` is bumped as well.**
*Why.* Cookies, refresh families and rotation are F-IDN-08 (spec 006); the port keeps this feature from reimplementing them. The version bump is the second line: every open login transaction compares its stored version and voids itself (plan 001 D1), and any access credential already issued fails the server-side version check (FR-IDN-061) even before the session rows are read. Accepted Runs continue under run-scoped authority, which the session service never touches (FR-IDN-067; A-39).
*Discarded.* Deleting session rows from this context — a foreign repository. Relying on the version bump alone — refresh-family revocation is the session feature's contract and must be explicit.
*Covers.* RF-5.5, RF-7.2, RF-8.9, RNF-10, LC-12, LC-13, LC-19, LC-29.

**D8 · The forced change is a state of the existing `LoginTransaction` [001], not a new transaction and not a session interrupt.**
*Why.* Decision Q2 places R08 after the first factor and before the challenge; the transaction already carries identity binding, expiry, attempt limit, row locking, voiding and idempotency-keyed outcomes, so adding one state (`FORCED_CHANGE_REQUIRED`) gives every required bound for free (RF-8.6, RF-8.7) and lets the same token continue into the challenge (RF-8.10). The challenge use cases refuse the new state, so no path from first factor to landing skips the change (RF-7.10). After its own credential change the transaction re-stamps its stored `securityVersion`, so it survives while every other open transaction of the identity voids (LC-29).
*Discarded.* A separate forced-change transaction — two tokens, two expiries, a handoff to invent. Checking the flag at challenge completion — would let the challenge run before the change, contrary to Q2. Interrupting an active session in place — the alternative reading of FR-IDN-045, deferred to A-45.
*Covers.* RF-7.6, RF-7.7, RF-7.10, RF-8.5–8.7, RF-8.10, RF-8.12, RF-8.13, RNF-8, LC-23–LC-29.

**D9 · The requirement is its own row with a source; policy-change and age detection are a pure rule evaluated at the first factor.**
*Why.* The compromise response must persist across sign-ins, so a stored record is unavoidable; keeping it off the identity row keeps FR-IDN-001's "only email, display name and status" literal (same reasoning as plan 002's acknowledgement table). Policy-change and age need only two credential facts (`policyVersion`, `changedAt`) and two configured values, so a pure `ForcedChangeRule` evaluated after a correct first factor (RF-7.3, RF-7.4) is testable without a database and never runs for an unknown or wrong-password attempt (RF-7.8, LC-21). A partial unique index keeps one pending requirement per identity.
*Discarded.* Computing everything on the fly — the compromise source has no fact to compute from. A flag on the identity row — FR-IDN-001. A scheduled job that flags aged passwords — flags identities that never sign in and adds nothing the first-factor evaluation does not give.
*Covers.* RF-7.1, RF-7.3, RF-7.4, RF-5.6, RF-8.9, LC-20, LC-21.

**D10 · The compromise response is an internal, audited application command with no HTTP surface.**
*Why.* Scope §2 forbids a customer-facing operator surface and FR-IDN-049 forbids any tenant-side reset of another identity's credential; the only lawful caller is the platform's own audited mechanism (A-01 / A-44). The command exists so the behaviour (requirement set, sessions revoked, audit with actor) is implemented and tested now, while the invocation mechanism stays a placeholder that the operator path decides.
*Discarded.* An administrative REST endpoint — an operator surface in the product. A tenant Admin action — FR-IDN-049.
*Covers.* RF-7.2, RF-7.5, LC-19.

**D11 · A wrong current password on R08 follows plan 001 D2's generic path and is counted on both sign-in throttle keys and the transaction's attempts; same-as-current is not counted.**
*Why.* The current-password check is a first-factor verification in disguise; counting it on `ACCOUNT` and `NETWORK` and on the transaction's attempt limit (decision Q6) bounds guessing exactly as sign-in does and keeps one generic rejection without cause or count. A new password equal to the current one proves knowledge of the current password, so it is a validation error, not an attack signal (RF-8.8).
*Discarded.* Unbounded retries with throttling only — rejected by Q6. A separate counter — a second lockout model.
*Covers.* RF-8.6–8.8, RNF-2, LC-23, LC-24.

**D12 · Password rules come from `PasswordPolicyPort` [002] for display and enforcement; same-as-current is checked with `Credential.matches`.**
*Why.* FR-IDN-011: the client never hard-codes rules; the same port serves R07 and R08 and the server enforces it, so a policy tightened after the page loaded is still applied. The same-as-current check needs no history: the maintained verifier compares the candidate with the stored credential (decision Q5: no password history).
*Discarded.* Client-only validation — bypassable. A password-history table — not chosen in Q5.
*Covers.* RF-4.2, RF-4.6, RF-5.2, RF-8.4, RF-8.8, LC-9, LC-10.

**D13 · Three outbox templates — `password-reset`, `password-changed`, `compromise-response-notice` — in the locale of the triggering request; one notification per security event, so a compromise-forced change does not notify twice.**
*Why.* RF-2.8, RF-5.7 and RF-7.2 fix the statements each message must carry; the outbox gives atomicity with the effect (D4); the locale header (plan 002 D15) lets the BE pick a template language without rendering language itself (the compromise notice has no triggering request and therefore no locale header; it uses the configured default message language of `ResetPolicy` — a validated configuration value, not a code default, since the identity itself holds no language per FR-IDN-001). Decision Q10 makes the compromise response itself the notifying event: the holder learns at once that every session ended and a change is required, and the later forced change — which the holder performs knowingly — sends nothing. `ChangeCredential` reads the cleared requirement's source and skips the template for `COMPROMISE_RESPONSE`, writing `NOTIFICATION_SKIPPED` so the omission is auditable (RF-10.1).
*Discarded.* Synchronous send — D4. Notifying on every change including the compromise-forced one — rejected by Q10 (a duplicate message after the notice). No notice at compromise time — the holder would learn of the revocation only by being signed out.
*Covers.* RF-2.4, RF-2.8, RF-5.7, RF-7.2, RF-8.9, RF-10.1, RNF-11, RNF-13, LC-34, LC-35.

**D14 · Links are built from the configured public origin; the request's host is never used.**
*Why.* RF-2.4 and FR-IDN-041; a request-controlled host would let an attacker mint links to a look-alike domain. The value is the one `SignupPolicy` [002] already validates (plan 002 D13).
*Discarded.* Deriving the origin from `Host` or forwarded headers.
*Covers.* RF-2.4.

**D15 · The FE keeps the token and the reset-done confirmation in memory only.**
*Why.* RNF-4 and RF-10.4 forbid browser storage; the token is read from the route on mount, held in a module-private value for the validate and complete calls, and dropped on leave. The confirmation on `/sign-in` (S-35) is an in-memory flag consumed once, so a reload shows the plain form and nothing is left in history or storage.
*Discarded.* sessionStorage — readable by any script in the origin and survives navigation. A query flag on `/sign-in` — leaks into history and can be forged.
*Covers.* RF-5.8, RF-10.4, RNF-4.

**D16 · R08 is a route of the sign-in flow driven by the transaction holder [001]; "Sign out safely" is the abandon call.**
*Why.* Decision Q2 makes R08 part of sign-in; the holder already carries the transaction and clears on leave; `DELETE challenge` [001] voids it, leaving the requirement pending (RF-8.12). The challenge route refuses a holder in `FORCED_CHANGE`, so the browser cannot skip the step even by typing the URL (RF-7.10).
*Discarded.* A modal inside the app shell — the A-45 alternative, not chosen. A dedicated sign-out endpoint for R08 — nothing to sign out of in the limited state.
*Covers.* RF-7.7, RF-7.10, RF-8.12, RF-8.13, LC-25, LC-26.

**D17 · No dedicated mobile pages; the three screens reflow under the WCAG rules.**
*Why.* M15–M17 are the drawn counterparts (S-03); FR-X-055 requires AA reflow at every other width; `SecurityStatements` shared across widths keeps every statement present (RNF-14).
*Discarded.* Separate mobile components — drift.
*Covers.* RF-9.1–9.3, RNF-14.

**D18 · The two completing submissions carry an idempotency key and return a recorded outcome on re-submission.**
*Why.* F-X-02 and LC-31: a reload during "updating" must not change the credential twice or report failure for a change that happened. The transaction row (forced change) and a small outcome record keyed by the idempotency key (reset) hold the answer; the use case checks the key before any other work. That record is the table `password_reset_outcome` (`idempotency_key` unique, `identity_id`, `outcome`, `recorded_at`), created by the same migration as `password_reset_link`.
*Discarded.* Generic POST retry — forbidden. No key — a re-submission after a lost answer would meet a consumed link and show the unavailable state for a successful reset.
*Covers.* RF-5.1, RF-8.5, LC-31.

**D19 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id [001 D12].**
*Why.* F-X-03; field paths give RF-1.4, RF-4.5, RF-4.6 and RF-8.8 without parsing text; one code per class keeps RF-6.2 and RF-8.6 generic; the correlation id joins the audit record to the user's report.
*Discarded.* Ad-hoc error bodies.
*Covers.* RF-1.4, RF-4.5, RF-4.6, RF-6.2, RF-8.6, RF-8.8, RF-10.3.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure domain: `ResetLink` lifecycle, `ForcedChangeRule`, `ForcedChangeRequirement`, `LoginTransaction` new state, `AbuseThrottlePolicy` with the reset keys, `ResetOutcome` mapping. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: conditional consume, row locks, throttle upserts, outbox rows, audit rows, revocation port calls, configuration binding, migrations. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of identity, token or factor data in responses. |
| Timing comparability | JUnit 5 statistical harness (test-only) | N samples per request class and per unavailable cause; distributions overlap within a configured tolerance. |
| Concurrency | JUnit 5 with real PostgreSQL | Two-tab submission, parallel transactions for a flagged identity, request races. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page states, field validation, holders cleared on leave, handoffs, statements present at every width, nothing in storage. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Focus order, announcements, no focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, 1.2, 1.6 | FE unit, e2e | `forgotPasswordPage.rendersInitialAndReceivedStates` (desktop, tablet, mobile) | 1, 12 |
| RF-1.3 | FE e2e | `signInPage.forgotPasswordRoutesToRequest` | 1 |
| RF-1.4 | FE unit, API contract | `forgotPasswordPage.blocksEmptyOrMalformedEmail`; `passwordApi.requestFieldPathProblems` | 2 |
| RF-1.5 | BE integration | `resetRequest.normalisesAddress` | 2 |
| RF-2.1, 2.6, RNF-1 | API contract + timing | `resetRequest.responseIdenticalAcrossClasses` (active, unverified, restricted, erasing, unknown, throttled); `resetRequest.timingComparable` | 3 |
| RF-2.2, 2.3 | BE integration | `resetRequest.activeVerifiedGetsHashedTokenWithConfiguredExpiry` | 5 |
| RF-2.4, 2.8, RNF-11 | BE integration | `resetRequest.enqueuesMessageToVerifiedAddressOnPublicOrigin`; `template.passwordResetCarriesStatements` | 11 |
| RF-2.5, LC-4, LC-5 | BE integration | `resetRequest.newLinkSupersedesEarlier`; `validate.supersededIsUnavailable` | 5 |
| RF-2.7 | BE integration | `resetRequest.changesNothingElse` (status, sessions, factor, requirement unchanged) | 7 |
| RF-3.1, 3.3 | BE unit, integration | `throttlePolicy.resetKeysProgressiveSteps`; `resetRequest.restrictedIssuesNothingSameAnswer` | 4 |
| RF-3.2 | BE integration | `throttle.resetAndSignInIndependentBothWays` | 4 |
| RF-3.4, RNF-2, LC-33 | BE integration | `throttle.resetWindowLiftsAndSignInNeverRestricted` | 4 |
| RF-3.5, RNF-5 | BE integration | `resetRequest.perAddressMessageCapBoundsOutbox` | 4 |
| RF-3.6, RF-7.4 (config), RNF-6 | Startup | `resetPolicy.startupFailsOnAbsentOrUnboundedValue` (one case per property) | 9 |
| RF-3.7 | Startup, BE integration | `resetPolicy.startupFailsWhenAddressWindowNotShorterThanNetworkWindow`; `throttle.noNetworkExemptionOnAddressKey` | 4, 9 |
| RF-4.1, RF-5.3, LC-8 | BE integration, FE unit | `validate.doesNotConsume`; `resetPasswordPage.validatesOnMountThenShowsForm` | 5 |
| RF-4.2, RF-8.4 | FE unit | `passwordRulesHint.rendersServerPolicy` on R07 and R08 | 2 |
| RF-4.3, RF-8.2 | FE unit | `securityStatements.presentOnResetAndForcedChange` | 1 |
| RF-4.4 | FE unit | `passwordField.showToggleNeverPersists` | 2 |
| RF-4.5, RF-4.6, LC-10 | FE unit, API contract | `resetPasswordPage.blocksMismatchAndPolicyFailure`; `complete.fieldPathProblems` | 2 |
| RF-4.7, RNF-8 | API contract | `resetResponses.exposeNoIdentityData` | 7 |
| RF-4.8, LC-11, LC-12 | BE integration, FE unit | `complete.ignoresExistingSessionAndRevokesSameIdentity`; `resetPasswordRoute.isPublic` | 6 |
| RF-5.1, RNF-3, LC-6 | Concurrency | `complete.twoTabsOneSuccessOneUnavailable` | 5 |
| RF-5.2, LC-9 | BE integration | `complete.sameAsCurrentRejectedTokenNotConsumed` | 2 |
| RF-5.4 | BE integration | `complete.oldPasswordStopsWorkingImmediately` | 2 |
| RF-5.5, RNF-10, LC-13 | BE integration | `complete.revokesAllSessionsFamiliesAndVoidsTransactions` | 6 |
| RF-5.6, LC-14 | BE integration | `complete.clearsPendingRequirementOfAnySource` | 6, 8 |
| RF-5.7, RF-7.2 (notice) | BE integration | `complete.enqueuesPasswordChangedNotification`; `compromiseResponse.enqueuesNoticeOnce`; `templates.threeMessagesCarryStatements` | 11 |
| RF-5.8 | API contract, FE unit | `complete.issuesNoSession`; `resetDone.routesToSignInWithNoticeOnce` | 2 |
| RF-5.9, RF-8.11, LC-15–LC-18 | BE integration | `changeCredential.touchesOnlyIdentityTables` (factor, codes, memberships, suspension, status, runs unchanged) | 7 |
| RF-5.10, LC-15, LC-16 | FE e2e | `afterReset.nextSignInChallengesEnrolledFactor`; `afterReset.noFactorLandsByMembership` | 7 |
| RF-6.1, 6.2, LC-3-style causes | API contract + timing, FE unit | `unavailable.oneCodeForEveryCause`; `unavailable.timingComparable`; `resetPasswordPage.unavailableStateWithRoutes` | 3, 5 |
| RF-6.3, LC-7 | BE integration | `complete.afterExpiryIsUnavailableAndDiscardsPassword` | 5 |
| RF-6.4 | FE e2e | `unavailable.requestNewLinkOpensEmptyRequest` | 1 |
| RF-6.5 | BE integration | `unusableLink.countsOnNetworkKeyOnly` | 4 |
| RF-7.1, RF-7.2, LC-19 | BE integration | `compromiseResponse.setsRequirementRevokesSessionsLeavesRunAuthority` | 6, 8 |
| RF-7.3, LC-20 | BE unit, integration | `forcedChangeRule.policyChange`; `signIn.policyChangeFlagsAtFirstFactor` | 8 |
| RF-7.4, LC-21 | BE unit, integration | `forcedChangeRule.passwordAge`; `signIn.wrongPasswordNeverSetsRequirement` | 8 |
| RF-7.5 | API contract | `noTenantSurfaceWritesRequirement` (no endpoint, no use case reachable under C10) | 8 |
| RF-7.6, RF-8.10, LC-27, LC-28 | BE integration, FE e2e | `signIn.flaggedIdentityGetsForcedChangeBeforeChallenge`; `forcedChange.continuesToChallengeOrLanding` | 8 |
| RF-7.7, RF-7.10 | API contract | `forcedChangeState.onlyForcedChangeAndAbandonAllowed`; `challenge.refusesForcedChangeState` | 7, 8 |
| RF-7.8 | API contract | `signIn.failedFirstFactorRevealsNoRequirement` | 3 |
| RF-7.9, LC-22 | BE integration | `signIn.unverifiedRoutesToVerificationNotForcedChange` | 8 |
| RF-8.1, RF-8.3 | FE unit, e2e | `forceChangePage.rendersFieldsActionsStatementsNoSource` | 1, 12 |
| RF-8.5, RF-8.9 | BE integration | `forcedChange.successChangesCredentialClearsFlagRevokesOthersNotifies` (policy and age sources) | 6, 8 |
| RF-8.9, RF-10.1, LC-34 | BE integration | `forcedChange.afterCompromiseSendsNoMessageAndAuditsSkip` | 8, 10, 11 |
| LC-35 | BE integration | `compromiseResponse.whileSignedOutSendsNoticeAndRequirementMetAtNextSignIn` | 8 |
| RF-8.6 | API contract, BE integration | `forcedChange.wrongCurrentIsGenericAndCountedOnBothKeys` | 8 |
| RF-8.7, LC-23, LC-24 | BE integration | `forcedChange.lastAttemptVoidsRequirementStays`; `forcedChange.expiryVoids` | 8 |
| RF-8.8 | BE integration | `forcedChange.sameAsCurrentRejectedNotCounted` | 2 |
| RF-8.12, RF-8.13, LC-25, LC-26 | FE unit, BE integration | `forceChangePage.signOutSafelyAbandons`; `forceChangePage.leaveAbandons`; `abandon.leavesRequirementPending` | 8 |
| RF-9.1–9.3, RNF-12, RNF-14 | FE e2e + axe, manual assistive | `a11y.resetAndForcedChange.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 12 |
| RF-10.1, 10.2, 10.5 | BE integration | `audit.entryPerEvent`; `audit.containsNoSecretOrUnmatchedAddress`; `audit.matchedFlagNeverInResponse` | 10 |
| RF-10.3, 10.4, RNF-4 | BE integration, FE unit | `logs.containNoTokenOrPassword`; `storage.containsNothingFromFeature` | 5, 10 |
| RNF-7 | BE integration | `failClosed.absentExpiredConsumedSupersededOrUnconfiguredRefuses` | 2, 9 |
| RNF-9 | BE integration | `factor.untouchedByBothFlows` | 7 |
| RNF-13 | FE e2e | `languageSwitch.onEachScreenKeepsTokenAndTransaction` (LC-30) | 12 |
| LC-1, LC-2, LC-3 | BE integration | `resetRequest.unknownUnverifiedRestrictedErasingIssueNothing` | 3 |
| LC-29 | Concurrency | `parallelFlaggedSignIns.firstChangeVoidsOther` | 8 |
| LC-31 | FE unit, BE integration | `resetPasswordPage.reloadResubmitsSameKey`; `complete.resubmitReturnsRecordedOutcome`; `forcedChange.resubmitReturnsRecordedOutcome` | 2 |
| LC-32 | BE integration | `resetRequest.outboxFailureNotVisibleInResponse` | 3 |
| US-9 | all adversarial cases above | enumeration, lockout, link reuse, forgery, MFA bypass, admin backdoor suites | 1 |

### 5.3 Not tested here
Sign-in, challenge and landing mechanics (plan 001), signup and verification (plan 002), enrollment and replacement (003), controlled recovery (005), session issue, refresh and expiry (006), session listing (007), the operator mechanism that calls the compromise command (A-44). Their stubs return fixed answers; `RevokeSessionsPort` is verified by a recording stub and, in the integration suite, by the session service's own test double.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-44 forced-change mechanism | `forced_change_requirement` row with source and time; `identity_credential.changed_at` and `policy_version`; `ApplyCompromiseResponseCommand` invoked by an operational command with an actor id; max age and policy version in `ResetPolicyProperties`. | Rebind the command to the decided operator path; move the two values to the configuration catalogue of A-11; the use cases are unchanged. |
| A-45 reading of "workspace context" | Sign-in gate (D8, D16): R08 inside the login transaction, before the challenge. | Add an in-session interruption: an `access`-side interceptor raising `auth.forced-change.required`, an R08 modal in the app shell, and a re-verification step; D8 and D16 are replaced, the BE use case is reused. |
| A-33 link consumption | SPA route reads the token on mount, validates, then completes (D3, D15). | Back-end open with a 303 redirect to a token-less form bound to a short-lived reference; replace D3 and the FE token holder. |
| A-39 revocation scope | All other identity sessions and refresh families; run-scoped authority untouched (D7). | Extend `RevokeSessionsPort` with the decided scope. |
| A-27 identity statuses | `isResetEligible()` = `ACTIVE` and verified; `UNVERIFIED`, `RESTRICTED`, `ERASING` ineligible. | Change the predicate only. |
| A-35 persistence tooling | Adapters and migrations against the placeholder API and tool. | Rename base types and migration format; no use case changes. |
| S-35 success state | In-memory flag and `ResetDoneNotice` on `/sign-in`; form empty. | Add a confirmed state to `ResetPasswordPage` and drop the flag. |
| S-36 R03 initial state | Same layout without the banner; repeat submissions allowed and answered identically. | Adjust the page states only. |
| S-37 R08 copy | Labels and statements verbatim; "Sign out safely" = abandon. | Change i18n keys; the abandon call stays. |
| S-14 templates | The D04 copy (`437:4431`) carrying the statements of RF-2.8, RF-5.7 and RF-7.2 in EN / ES / FR. | None. |
| S-03 mobile counterparts | Reflow of R03, R07, R08 under the desktop rules. | Replace the breakpoint layout per screen. |
| S-17 "Need help?" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |

The product doubts of the specification's first draft are closed as decisions Q7–Q10 and are built, not assumed: the bounded address window shorter than the network window with no exemption (D5), newest-link-only whoever requested (D1), the session-agnostic reset screen (D3, D7), and the compromise-response notice in place of a password-changed message (D13). The S-14 template inventory now carries three messages for this feature.

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.6) | `ForgotPasswordPage`, `SecurityStatements`, `PasswordController` problems, `EmailAddress` | D4, D19 | RF-1 rows |
| RF-2 (2.1–2.8) | `RequestPasswordResetUseCase`, `IssueResetLink`, `ResetLink`, outbox template, public origin | D1, D4, D13, D14 | RF-2 rows |
| RF-3 (3.1–3.7) | `AbuseThrottlePolicy` reset keys, `ThrottleStore`, message cap, `ResetPolicyProperties` cross-field constraint | D5 | RF-3 rows |
| RF-4 (4.1–4.8) | `ValidateResetLinkQuery`, `ResetPasswordPage`, `PasswordRulesHint`, `PasswordPolicyQuery` | D3, D12, D19 | RF-4 rows |
| RF-5 (5.1–5.10) | `CompletePasswordResetUseCase`, `ChangeCredential`, `Credential.replace`, `RevokeSessionsPort`, `ResetDoneNotice` | D2, D6, D7, D12, D13, D15, D18 | RF-5 rows |
| RF-6 (6.1–6.5) | `ResetOutcome`, unavailable state, network key counting | D3, D5, D19 | RF-6 rows |
| RF-7 (7.1–7.10) | `ForcedChangeRequirement`, `ForcedChangeRule`, `EvaluateForcedChangeStep`, `ApplyCompromiseResponseCommand`, `LoginTransaction` state | D8, D9, D10 | RF-7 rows |
| RF-8 (8.1–8.13) | `ForceChangePasswordPage`, `CompleteForcedChangeUseCase`, `ChangeCredential`, `AbandonChallengeUseCase`, holder | D6, D8, D11, D12, D16, D18, D19 | RF-8 rows |
| RF-9 (9.1–9.3) | Responsive pages, `shared/ui` | D17 | RF-9 row |
| RF-10 (10.1–10.5) | Audit adapter, correlation id, token holder, redaction | D15, D19 | RF-10 rows |
| RNF-1 | D4 path, one unavailable code | D4, D3 | timing + contract |
| RNF-2 | reset keys, bounded windows, counted R08 attempts | D5, D11 | RF-3.4, RF-8.6 rows |
| RNF-3 | conditional consume | D2 | LC-6 |
| RNF-4 | hash only, token holder, redaction | D1, D15 | RF-2.2, RF-10 rows |
| RNF-5 | message cap, bounded verification | D5 | RF-3.5 |
| RNF-6 | `ResetPolicyProperties` | — | startup |
| RNF-7 | conditional consume, config validation, requirement checks | D2, D9 | `failClosed` |
| RNF-8 | no C10, limited state | D8 | RF-4.7, RF-7.7 rows |
| RNF-9 | `ChangeCredential` scope | D6 | `factor.untouchedByBothFlows` |
| RNF-10 | `RevokeSessionsPort` + version bump in the same transaction | D6, D7 | RF-5.5 row |
| RNF-11 | three outbox templates, email port | D4, D13 | RF-2.4, RF-5.7, LC-32 |
| RNF-12 | `shared/ui`, views | D17 | e2e + axe, manual |
| RNF-13 | i18n bundles, locale header | D13 | RF-9 row, `languageSwitch` |
| RNF-14 | one component per screen | D17 | RF-9 row |
| LC-1 … LC-35 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35) |

Self-check: every RF criterion (75), every RNF (14) and every limit case (35) of `specPasswordReset.md` v0.2 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
