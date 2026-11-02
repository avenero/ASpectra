# Plan 001 — Login

**Feature** Sign-in (first factor, TOTP challenge and landing) · **Plan version** 0.1 · **Date** 26 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/001-login/specLogin.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; features F-IDN-03 and F-IDN-04 as the subject, F-IDN-01, F-IDN-02, F-IDN-05…F-IDN-09, F-IDN-14 and F-IDN-15 at the boundary, F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG) and FE board `46:7847` (feature-modular SPA DAG).

This plan is the HOW. It adds no behaviour the specification does not state. Where the specification left a question to the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (§8 of the spec: A-27, A-29, A-30, A-31, A-32, A-34, A-36, S-03, S-17, S-26; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P3** The browser expresses intent, never authority | The FE never decides an authentication outcome. Every state (failure, limited state, completed, void, landing data) is a BE response; the FE renders and routes. The landing rule is navigation, not authority: tenant entry re-derives C10. | RF-5, RF-8, RNF-9 |
| **P6** Fail closed | An absent, expired, superseded or voided login transaction, an unknown code, or an absent configuration value refuses the operation. No default is assumed anywhere. | RF-7, RF-4.8, RNF-4, RNF-8 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security JWT / refresh / MFA support. FE: React 18, TypeScript, Vite, TanStack Query, Zustand, i18next, Tailwind, axios. Nothing is added: no cache store, no new identity provider, no new form or DI framework. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: Login lives inside the existing hexagonal bounded context `identity`; outer adapters depend inward; other contexts are reached only through their published ports. FE: one feature module with a public API; TanStack Query owns server facts; Zustand is not used because Login has no unsaved fork. | §2 |
| **P12** Two independent tenancy gates | Identity, credential, factor, recovery-code, login-transaction and throttle rows are **platform** rows (an identity exists before any tenant). They carry no `tenant_id` and are protected by application authority. The one cross-tenant read in Login — the membership projection for landing — is an identity-scoped read published by the `access` context for the signed-in identity (the RLS policy shape is A-36); it carries no tenant authority. | RF-8, LC-15–LC-17 |
| **P13** Configured values are never defaulted | Thresholds, delays, restriction windows, transaction expiry, attempt limit and the authenticator time tolerance are one validated configuration record; startup fails if any value is absent or unbounded. | RF-4.8, RF-5.3, RF-6.5, RNF-4 |
| **P14** Release bar | WCAG 2.2 AA on sign-in, challenge and landing with automated and manual assistive evidence; EN / ES / FR. | RF-9, RNF-5, RNF-6, RNF-10 |
| **P1** Acceptance ≠ approval ≠ Run | Sign-in completion issues a session and nothing else; no Run intent is read, stored or replayed. | RF-8.6 |
| **F-X-01** C10 context | Tenant entry derives C10 server-side; the landing projection is an identity-scoped read (A-36); the limited identity state has no C10 at all. | RF-5.2, RF-8.5, RF-8.7 |
| **F-X-02** Request integrity | Sign-in and challenge submissions are consequential writes with no generic auto-retry; the FE submits once and waits. Each challenge submission carries an idempotency key; the transaction records the outcome per key, and a re-submission after a lost answer receives the recorded outcome (LC-24) — never a second evaluation and never a replay queue. | RF-6.3, LC-24 |
| **F-X-03** Secrets and safe problems | Passwords, codes and recovery codes exist only in the request body and the verifier; problem details carry safe codes and a correlation ID; browser storage holds nothing from Login. | RF-2.5, RF-10.3, RNF-7 |
| **F-X-05** Audit log | Every outcome in RF-10.1 is appended through the `audit` context port, redacted, in the same transaction as the attempt. | RF-10 |
| **F-X-06** Configuration validation | The Login policy record is part of startup validation. | RF-4.8, RNF-4 |
| **F-X-07** Responsive surfaces | M02 is implemented as the drawn reflow of R02; R05 reflows under the same rules. | RF-9 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity`

The Figma BE section already places "local identity / session" in the `identity` context, with the four standard layers `api / application / domain / infrastructure`. Login adds use cases to that context; it creates no new context. Contexts it touches — `access`, `audit`, `sharedsupport` — are reached only through their published ports (allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `GlobalIdentity` | Identity aggregate: id, email (normalised), display name, `IdentityStatus`, `emailVerifiedAt`. `IdentityStatus` is an enumeration with an explicit `permitsSignIn()` predicate; its values are a register dependency (A-27, §6). | RF-2.6, RF-2.7, RF-3.1 |
| | `Credential` | Non-reversible password credential; exposes only `matches(rawPassword, verifier)`. | RF-2.1 |
| | `TotpFactor` | Enrolled factor: the encrypted seed (ciphertext plus the server key reference under which it was encrypted), `replayWatermark` (last accepted time-step). Never exposes the seed. | RF-6.4, RF-6.5, RF-6.11 |
| | `RecoveryCodeSet` | Hashed single-use codes; `consume(rawCode, verifier)` returns the consumed code or nothing. | RF-6.7, RF-6.8, LC-7, LC-8 |
| | `LoginTransaction` | State machine `FIRST_FACTOR_PASSED → COMPLETED \| VOIDED`; holds identity id, purpose `SIGN_IN`, `expiresAt`, `attemptsRemaining`, `voidReason`, the identity's `securityVersion` at creation (compared at every verification; a mismatch voids with `CREDENTIAL_CHANGED` or `FACTOR_CHANGED`), and the recorded outcome of each submission by its idempotency key. Methods: `recordFailedAttempt`, `complete`, `void`, `recordedOutcome(key)`. Pure; takes the clock as an argument. | RF-5.1–5.4, RF-7.1–7.3, LC-4–LC-6, LC-9, LC-13, LC-14, LC-24 |
| | `AbuseThrottlePolicy` | Pure functions over the configured steps: `delayFor(failures)`, `restrictionFor(failures, now)`, `isRestricted(window, now)`. Two dimensions: `ACCOUNT` and `NETWORK`. | RF-4.1–4.4, RF-4.6 |
| | `LandingRule` | Pure function memberships → `CREATE_WORKSPACE \| ENTER_TENANT(id) \| TENANT_CHOOSER`. | RF-8.2–8.4, LC-15–LC-17 |
| | `LoginPolicy` | Immutable value object with every configured number: throttle steps, restriction windows, transaction expiry, attempt limit, time tolerance. | RF-4.8, RF-5.3, RNF-4 |
| `identity.application` | `SignInUseCase` | Normalises email; loads the identity; always runs exactly one credential verification — against the real credential for `ACTIVE` and `UNVERIFIED` identities, against the dummy hash (`PasswordVerifier.verifyAgainstDummy()`) when the identity is unknown or its status is `RESTRICTED` or `ERASING`; evaluates both throttle dimensions; branches: generic failure / verification-pending (carrying the pending reference of plan 002 D6) / limited state (factor enrolled) / completed (no factor). A restriction never short-circuits the verification; it forces the generic branch afterwards. Records throttle counters and the audit event in the same transaction. | RF-2, RF-3, RF-4, RF-5.1, RF-5.5 |
| | `VerifyTotpUseCase` | Locks the transaction row; checks state, expiry and attempts; verifies the code against the tolerance window and the replay watermark; on success advances the watermark and completes; on failure decrements attempts and voids at zero. | RF-6.2–6.6, RF-7.1, RF-7.2, RF-7.5 |
| | `VerifyRecoveryCodeUseCase` | Same transaction discipline; consumes one unused code; same failure path. | RF-6.7, RF-6.8 |
| | `AbandonChallengeUseCase` | Voids the transaction; idempotent. | RF-5.4, RF-6.9, LC-22 |
| | `CompleteSignInUseCase` | Shared tail of success: marks the transaction `COMPLETED`, calls `IssueSessionPort`, appends the audit event. Never reads or replays any Run intent. | RF-8.1, RF-8.6 |
| | `LandingQuery` | Reads the membership projection under C10 and applies `LandingRule`; returns the projection and the decision. | RF-8.2–8.5, RF-8.7 |
| | Ports (interfaces) | `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `Clock`, `PasswordVerifier` (adapter over the maintained Spring Security `PasswordEncoder`), `TotpVerifier` (adapter over the existing MFA support), `SeedCipher` (→ `sharedsupport/crypto`), `IssueSessionPort` (→ existing session service, F-IDN-08), `MembershipProjectionPort` (published by `access`), `AuditPort` (published by `audit`), `SecurityNotificationPort` (declared here; its first caller is plan 002 RF-3.2; sign-in events pending A-30, §6), `PendingReferenceStore` (owned by plan 002; used by the verification-pending branch to issue the handoff reference, A-34). | all |
| `identity.infrastructure` | Persistence adapters | PostgreSQL through the persistence access API and migration tool fixed by A-35 (placeholders until decided). Tables: `identity`, `identity_credential`, `identity_totp_factor`, `identity_recovery_code`, `login_transaction`, `signin_throttle`. `login_transaction` is read with `SELECT … FOR UPDATE` on every verification (LC-9). `signin_throttle` keyed by `(dimension, subject_key)` and updated with an upsert in the attempt's transaction. | RF-4, RF-5, RF-6, LC-9, LC-10 |
| | Spring Security integration | An authentication entry that delegates to `SignInUseCase`; the `PasswordVerifier` adapter supplies the fixed dummy hash (the use case decides when it is used); the session issue adapter behind `IssueSessionPort`. No change to the existing JWT / refresh mechanics. | RF-2.3, RF-8.1 |
| | `LoginPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding of every number into `LoginPolicy`; constraints: present, positive, bounded above by a configured ceiling. Any violation fails startup. | RF-4.8, RNF-4 |
| | Audit adapter | Maps RF-10.1 outcomes to `audit` events: `SIGN_IN_SUCCEEDED`, `SIGN_IN_FAILED`, `CHALLENGE_PASSED`, `CHALLENGE_FAILED`, `CHALLENGE_VOIDED`, `RECOVERY_CODE_USED`, `RESTRICTION_APPLIED{restrictedUntil}`, `RESTRICTION_LIFTED` (recorded by the first attempt observed after the window has elapsed). Payload: identity id when matched, correlation id, network key, outcome kind, timestamp. Never the password or code. | RF-10.1–10.3 |
| `identity.api` | `AuthController` — `/api/auth` | `POST sign-in` (email, password) → 200 `{state: LIMITED, transaction}` or 200 `{state: COMPLETED, landing}` or 400 problem with field paths or 401 problem `auth.sign-in.failed` or 409 problem `auth.verification-pending {pendingRef, email}` · `POST challenge/totp` (transaction, idempotencyKey, code) and `POST challenge/recovery-code` (transaction, idempotencyKey, code) → 200 `{state: COMPLETED, landing}` or 400 problem `auth.challenge.rejected` (no cause, no attempt count) or 410 problem `auth.challenge.void` (voided or already completed) · `DELETE challenge` (transaction) → 204 · `GET landing` → 200 `{decision, projection}`, used only when the landing route is reloaded (requires a session). `landing` in every completed response is `{decision, projection}`; the FE navigates by `decision` and never re-derives it. All problems are RFC 9457 problem details with a safe code, field paths for RF-1.6 and a correlation id. The server never redirects; routing is the FE's. | RF-1.6, RF-2.5, RF-3.1, RF-5–RF-8 |

**Cross-context touch points.** `access.MembershipProjectionPort` (memberships of the signed-in identity: tenant id, tenant status, role, membership status) — read under C10 derived from the new session. `audit.AuditPort` — append-only. `sharedsupport` — `crypto` (seed cipher, key references), `tx` (transaction helpers), `contextIds` (correlation id). No other context is touched; in particular Login never reads project, grant or run tables.

**Data model note (P12).** The six Login tables are platform-scoped and carry no tenant column; RLS does not apply to them. They are the identity half of the "identity 1 → N membership" ERD in the BE section; membership rows stay in `access` and under RLS.

### 2.2 Front end — feature module `src/features/identity-access`

The FE section's feature list (workspace, memberships, project, authoring, …) names no identity feature, so the module name is a plan decision (§4, D14). It follows the app-composed DAG: `app` → feature public APIs → pure `contracts` + generic `shared` adapters; no sibling-internal imports.

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/sign-in`, `/sign-in/challenge`; exported route objects and nothing else from views. | RF-1, RF-6 |
| | `useSignIn()`, `useChallenge()`, `useLanding()` | Typed intent hooks wrapping the mutations and query below; return server state verbatim. | RF-2–RF-8 |
| | `contracts/auth` (in `src/contracts`) | Request / response / problem types and runtime guards for the five endpoints. Pure TypeScript, no React. | all |
| Views | `SignInPage` | R02 at desktop, M02 at mobile widths by Tailwind breakpoints — one component. Email field, password field with show toggle, Sign in, Forgot password route, Create account route, lost-authenticator (controlled recovery) route, language switch, the three statement blocks. | RF-1.1–1.5, RF-1.7, RF-9.1–9.3 |
| | `MfaChallengePage` | R05 and its drawn mobile counterpart M12 (`431:4162`; S-03). Code field accepting paste, Verify & continue, Use a recovery code, controlled-recovery entry, "Need help?" opening N43, statements, "verification in progress" state. | RF-6.1–6.3, RF-6.9, RF-9.4 |
| | `GenericFailureMessage` | The single failure text, focus target, associated with the form. | RF-2.5, RNF-5 |
| | `SecurityStatements` | The statements of RF-1.2–1.4 and RF-6.1 as one reusable block so desktop and mobile cannot drift. | RF-1.2–1.4, RF-9.2 |
| State | TanStack Query | `useMutation` for sign-in, totp, recovery-code, abandon (each challenge mutation generates its idempotency key once and reuses it on re-submission); `useQuery` for `GET landing` only when the landing route is reloaded — a completed sign-in already carries `landing`. No retry on mutations (F-X-02). | RF-2–RF-8, LC-24 |
| | Transaction holder | Module-private in-memory store (not Zustand: there is no unsaved fork; not localStorage: F-X-03) holding the opaque transaction token; cleared on completed, void, abandon, route leave. | RF-5.4, RF-7.3, LC-22, RNF-7 |
| Routing | `LandingRouter` | Maps the server's `landing.decision` to a route and never evaluates the rule itself (P3): `CREATE_WORKSPACE` → workspace creation route; `ENTER_TENANT` → tenant route (which re-authorises); `TENANT_CHOOSER` → chooser route. | RF-8.2–8.5 |
| | Handoffs | `auth.verification-pending` → verification route (pending state, spec 002); `auth.challenge.void` → `/sign-in` with no reason text; "Use a recovery code" toggles the input mode in place; controlled recovery → its route (spec 005). | RF-3.1, RF-6.9, RF-7.3, RF-7.4 |
| | Scope epoch | On `COMPLETED`, `shared/security` advances the scope epoch and clears any scoped cache before landing fetches anything. | RF-8.7 |
| Shared used | `shared/api` | axios transport with correlation id header; no auto-retry on POST. | RF-10.2, F-X-02 |
| | `shared/errors` | Problem-details parsing into safe, typed problems; field-path to field mapping. | RF-1.6, RF-2.5 |
| | `shared/i18n` | Feature bundle `identity-access.{en,es,fr}.json` with a key for every statement and message. | RNF-6, LC-21 |
| | `shared/ui` | Accessible `Field`, `PasswordField`, `CodeField`, `Button`, `LiveRegion` (polite, non-focus-stealing), 44 px targets at touch widths. | RF-1.6, RF-9.3, RNF-5 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | — | `SignInPage` |
| RF-1.2 | — | `SecurityStatements` (TOTP, limited state, no bypass) |
| RF-1.3 | — | `SecurityStatements` (local identity only) |
| RF-1.4 | — | `SecurityStatements` + route to controlled recovery |
| RF-1.5 | — | `PasswordField` show toggle; value never persisted |
| RF-1.6 | `AuthController` field-path problems (server-side validation too) | `Field` validation, associated message, focus |
| RF-1.7 | — | `SignInPage` shows no password rules |
| RF-2.1 | `SignInUseCase` + `PasswordVerifier` | — |
| RF-2.2 | `SignInUseCase` single failure branch; `auth.sign-in.failed` | `GenericFailureMessage` |
| RF-2.3 | `SignInUseCase` with `PasswordVerifier.verifyAgainstDummy()` | — |
| RF-2.4 | `SignInUseCase`: no identity created and no password stored for an unknown email (throttle and audit rows are still written) | — |
| RF-2.5 | `auth.sign-in.failed` problem | `GenericFailureMessage`, keep email, clear password, focus |
| RF-2.6 | `GlobalIdentity` email normalisation; unique index on normalised email | — |
| RF-2.7 | `IdentityStatus`: `RESTRICTED` / `ERASING` → generic branch (`UNVERIFIED` → the RF-3 branch) | — |
| RF-3.1 | `SignInUseCase` verification-pending branch (no transaction, no session); `auth.verification-pending` | Handoff to verification route |
| RF-3.2 | `SignInUseCase` checks verification before factor | — |
| RF-3.3 | Generic branch precedes verification check | — |
| RF-4.1 | `AbuseThrottlePolicy` two dimensions; `ThrottleStore` | — |
| RF-4.2 | `delayFor` / `restrictionFor` from `LoginPolicy` steps | — |
| RF-4.3 | Throttled → full path then generic branch; no wait time in the problem | `GenericFailureMessage` (same text) |
| RF-4.4 | `isRestricted(window, now)`; windows expire by time; no permanent flag exists | — |
| RF-4.5 | `VerifyTotp` / `VerifyRecoveryCode` void path writes both counters | — |
| RF-4.6 | A restriction never short-circuits verification (full path, RF-4.3); the bounded worker pool in infrastructure caps verification work | — |
| RF-4.7 | `signin_throttle` dimension keys are distinct from the reset feature's store (spec 004) | — |
| RF-4.8 | `LoginPolicyProperties` validation at startup | — |
| RF-5.1 | `SignInUseCase` creates `LoginTransaction`; 200 LIMITED | Route to `/sign-in/challenge` |
| RF-5.2 | Transaction token authorises only the three challenge endpoints; no C10 derived | Challenge page renders no identity data |
| RF-5.3 | `LoginTransaction` fields from `LoginPolicy` | — |
| RF-5.4 | `AbandonChallengeUseCase`; `DELETE challenge` | Route leave / back calls abandon; holder cleared |
| RF-5.5 | `SignInUseCase` completed branch when no factor | `LandingRouter` |
| RF-6.1 | — | `MfaChallengePage` |
| RF-6.2 | Separator stripping before verification | `CodeField` paste handling |
| RF-6.3 | — | Pending state, disabled resubmit, then result |
| RF-6.4 | `VerifyTotpUseCase` success → `CompleteSignInUseCase` | `LandingRouter` |
| RF-6.5 | `TotpFactor.replayWatermark`; tolerance from `LoginPolicy` | — |
| RF-6.6 | `LoginTransaction.recordFailedAttempt`; `auth.challenge.rejected` without cause or attempt count | Generic "code not accepted" message |
| RF-6.7 | `VerifyRecoveryCodeUseCase`; `RecoveryCodeSet.consume` under row lock | Recovery-code input mode |
| RF-6.8 | Unknown / consumed code → same failure path | Same message |
| RF-6.9 | `AbandonChallengeUseCase` then handoff | Route to controlled recovery |
| RF-6.10 | No endpoint, flag or cookie exists for device trust | No control rendered |
| RF-6.11 | `TotpFactor` never serialises the seed; API records carry no factor data | — |
| RF-7.1 | Attempts reach zero → `void(ATTEMPTS_EXHAUSTED)`; `auth.challenge.void` | Route to `/sign-in` |
| RF-7.2 | Expiry checked under lock on every call → `void(EXPIRED)` | Same |
| RF-7.3 | Voided transaction refuses every challenge call; a new sign-in creates a new row | Holder cleared; form empty of transaction |
| RF-7.4 | `auth.challenge.void` carries no cause | No notice rendered |
| RF-7.5 | Void path updates both throttle counters | — |
| RF-8.1 | `CompleteSignInUseCase` → `IssueSessionPort`; the response carries `landing` | Epoch advance; navigation by the server decision |
| RF-8.2 | `LandingRule` → `CREATE_WORKSPACE` | `LandingRouter` maps the decision to the route |
| RF-8.3 | `LandingRule` → `ENTER_TENANT` | `LandingRouter` maps the decision to the route |
| RF-8.4 | `LandingRule` → `TENANT_CHOOSER` | `LandingRouter` maps the decision to the route |
| RF-8.5 | Tenant entry (other feature) re-derives C10 and the enrollment gate; Login passes nothing that could bypass it | Tenant route is a plain navigation |
| RF-8.6 | `CompleteSignInUseCase` touches no Run, approval or mapping table | No pending intent is stored anywhere in the FE |
| RF-8.7 | `LandingQuery` reads the signed-in identity's current memberships (identity-scoped read, A-36) | Scoped cache cleared before landing |
| RF-9.1 | — | `SignInPage` at M02 breakpoints |
| RF-9.2 | — | `SecurityStatements` shared by both layouts |
| RF-9.3 | — | `shared/ui` touch targets, labels, focus |
| RF-9.4 | — | `MfaChallengePage` reflow rules |
| RF-10.1 | Audit adapter events in the attempt's transaction | — |
| RF-10.2 | Audit payload shape (identity id, time, network key, correlation id) | Correlation id header |
| RF-10.3 | Verifier inputs never logged; problem details exclude inputs | No secret in state, URL or storage |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The limited identity state is a server-side `LoginTransaction` row plus an opaque client token.**
*Why.* Expiry, attempt limit, replay control and voiding must be enforced where the client cannot touch them; a row can be locked (`FOR UPDATE`) so two tabs cannot both succeed; a voided row stays voided across instances and restarts; every transition is auditable. The row also carries the identity's `securityVersion` at creation, so a password reset or factor change — which bump that version (FR-IDN-061) — voids every open transaction at its next use (LC-13, LC-14) without the other features having to know about Login. The outcome of each submission is recorded under the client's idempotency key, so a re-submission after a lost answer returns that outcome instead of evaluating the code again (F-X-02, LC-24).
*Discarded.* A stateless signed token carrying the attempt count — the client could replay an older token with more attempts, and a token cannot be voided. An HTTP session attribute — not safe across instances, not lockable, not auditable. A cross-feature "void my transactions" call from reset and factor replacement — couples three features to Login's table.
*Covers.* RF-5.1–5.4, RF-6.5, RF-7.1–7.3, LC-9, LC-13, LC-14, LC-24, RNF-8, RNF-9.

**D2 · One generic failure with a constant-cost path.**
*Why.* RF-2.2 requires identical content, shape and comparable timing. The use case runs the same steps for every failure class: load (or not), throttle evaluation, a real `PasswordEncoder` verification (against a fixed dummy hash when the identity is unknown or its status is `RESTRICTED` or `ERASING`; an `UNVERIFIED` identity is verified against its real credential so that RF-3 can detect a correct password), then the one failure branch. One problem code, one message key.
*Discarded.* Distinct problem codes with identical UI text — the network response leaks the class. Early return for unknown email — a timing leak. A separate "too many attempts" response — chosen against by the specification (RF-4.3).
*Covers.* RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.7, RF-4.3, RNF-1.

**D3 · Throttle counters live in PostgreSQL, in two dimensions, driven by a pure policy.**
*Why.* P10 permits no new store. A `signin_throttle` row per `(dimension, subject_key)` updated in the attempt's own transaction is consistent across instances and survives restarts; the pure `AbuseThrottlePolicy` keeps delay and restriction arithmetic testable without a database. Delay is applied by the use case before responding; restriction is a stored window that expires by time, so no permanent state can exist; its application is audited with `restrictedUntil`, and its lifting is audited by the first attempt observed after the window.
*Discarded.* Redis or an in-memory cache — a new mandatory store (P10) or inconsistent across instances. Edge / WAF rate limiting only — not designed (A-20) and cannot see the account dimension. A per-identity lock flag — a permanent lockout an attacker could trigger (RF-4.4).
*Covers.* RF-4.1–4.6, RNF-2, RNF-3, LC-10, LC-11, LC-12.

**D4 · TOTP verification uses the existing Spring Security MFA support with a per-factor replay watermark.**
*Why.* P10 keeps the existing MFA support; replay control needs only the last accepted time-step per factor: any code whose step is at or below the watermark is rejected. The tolerance window is a configured value (A-32).
*Discarded.* Storing every accepted code — unbounded growth, same guarantee. No replay control — violates RF-6.5.
*Covers.* RF-6.4, RF-6.5, LC-3, LC-5.

**D5 · Recovery codes are hashed with the maintained `PasswordEncoder` and consumed under the transaction row lock.**
*Why.* A recovery code is a password-equivalent secret; the maintained encoder gives the right work factor; consuming inside the locked transaction makes LC-7 and LC-9 deterministic.
*Discarded.* Reversible encryption (unnecessary: codes are only ever compared). A fast hash — brute-forceable at rest.
*Covers.* RF-6.7, RF-6.8, LC-7, LC-8.

**D6 · The TOTP seed is stored encrypted under a server key reference from `sharedsupport/crypto` and is never serialised.**
*Why.* Verification needs the seed, so it cannot be hashed; the key reference keeps the seed out of the database in clear; the domain object never exposes it and no API record carries factor data.
*Discarded.* Hashing — makes verification impossible. Plaintext column — violates F-X-03.
*Covers.* RF-6.11, RNF-7.

**D7 · Landing is computed from a server membership projection and routed by the FE.**
*Why.* The decision needs membership facts that only the server holds (an identity-scoped read, A-36); `LandingQuery` returns the decision with the facts behind it, and `LandingRouter` navigates by the decision without re-deriving it (P3: the browser never decides an outcome, however harmless). Routing is not authority: entering the tenant re-derives C10 and applies the enrollment gate (RF-8.5).
*Discarded.* Server HTTP redirects — wrong for an SPA and contrary to the FE owning navigation (P11). FE evaluating the rule from the projection — the browser would decide an outcome. FE fetching the tenant chooser payload — that is feature F-IDN-15 (screen N03), and it carries more than Login needs. Treating the landing result as an authorisation — P3.
*Covers.* RF-8.1–8.7, LC-15, LC-16, LC-17, LC-18.

**D8 · Session issue is delegated to the existing session service through one port.**
*Why.* Cookies, refresh families, rotation and CSRF are F-IDN-08 (spec 006). Login calls `IssueSessionPort.issue(identityId)` and nothing more, so Login cannot drift from the session rules.
*Discarded.* Issuing tokens inside the sign-in controller — duplicates and eventually contradicts the session feature.
*Covers.* RF-8.1.

**D9 · A correct password on an unverified identity returns `auth.verification-pending {pendingRef, email}` and the FE routes to the verification pending state.**
*Why.* The specification's decision: the password has proved control of the account, so the disclosure is acceptable and the dead end is avoided. No transaction and no session are created. The response carries the pending reference of plan 002 D6 — issued only because the password was just verified, authorising re-addressing of the pending identity and nothing else; it is neither a session nor a limited identity state (conditioned on A-34).
*Discarded.* Generic failure — the specification rejected it.
*Covers.* RF-3.1–3.3, LC-1.

**D10 · Every configured number is one validated `LoginPolicyProperties` record that fails startup.**
*Why.* P13 and FR-X-046. One record makes the full set visible and testable; validation at bind time means the product never runs with a missing or unbounded limit.
*Discarded.* Defaults in code — forbidden. Lazy validation on first use — the product would start and fail on the first sign-in.
*Covers.* RF-4.8, RF-5.3, RNF-4, RNF-8.

**D11 · Audit is appended through the `audit` context port in the attempt's transaction.**
*Why.* An audit entry that commits with the attempt can neither be lost nor describe something that did not happen. The payload carries only identifiers, outcome kind and the correlation id.
*Discarded.* The logging framework as audit — not append-only, not authority-scoped, prone to secret leakage. An asynchronous outbox — acceptable later for throughput, but it adds a window in which an outcome exists without its record.
*Covers.* RF-10.1–10.3, RNF-7.

**D12 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id.**
*Why.* F-X-03 requires safe problem codes; field paths give the FE RF-1.6 without parsing text; the correlation id joins the audit record to the user's report.
*Discarded.* Ad-hoc JSON error bodies — every feature would invent its own.
*Covers.* RF-1.6, RF-2.5, RF-6.6, RF-7.4, RF-10.2.

**D13 · The FE keeps the transaction token in memory only and voids it on leave.**
*Why.* RNF-7 forbids browser storage; RF-5.4 and LC-22 require that leaving the challenge voids the transaction. A module-private store cleared on completion, void, abandon and route leave, with `DELETE challenge` fired on leave, gives exactly that.
*Discarded.* sessionStorage — survives navigation and is readable by any script in the origin. A URL parameter — leaks into history and logs.
*Covers.* RF-5.4, RF-7.3, LC-22, RNF-7.

**D14 · One feature module `identity-access` with one responsive component per screen.**
*Why.* The FE DAG wants public APIs per feature and no sibling-internal imports; the designed mobile counterpart M02 is a reflow of R02, so one component with Tailwind breakpoints keeps the safety statements identical by construction (RNF-10); R05 reflows by the same rules (S-03). The module also hosts the identity screens of specifications 002–007, 012 and 013 later, which keeps the public identity surface in one place.
*Discarded.* Separate mobile routes and components — duplication and drift of statements. Placing Login in `app/` — it is a feature with its own views and state, not composition.
*Covers.* RF-9.1–9.4, RNF-10.

**D15 · The code field is a single input that accepts paste and strips separators; the server strips them too.**
*Why.* RF-6.2 requires a pasted complete code; a single labelled input is also the accessible choice. Server-side stripping makes the rule independent of the client.
*Discarded.* Six digit boxes — hostile to paste and screen readers.
*Covers.* RF-6.2, LC-23, RNF-5.

**D16 · Text comes from feature-scoped i18n bundles; the server sends codes only.**
*Why.* RNF-6 and FR-X-043: the server never renders language; every statement and message has a key in EN, ES and FR; a language switch re-renders without touching the transaction (LC-21).
*Discarded.* Server-rendered messages — ties language to the request and duplicates the catalogue.
*Covers.* RF-1.1–1.4, RF-6.1, RNF-6, LC-21.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure domain: `AbuseThrottlePolicy` arithmetic, `LoginTransaction` state machine, TOTP verifier with a fixed clock, `RecoveryCodeSet.consume`, `LandingRule`, email normalisation. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: locking, throttle upserts, audit rows, configuration binding. |
| API contract | Spring MockMvc (**test-only**, Spring Boot test support) | Status codes, problem shapes, safe codes, correlation id, absence of any secret or factor data in responses. |
| Timing comparability | JUnit 5 statistical harness (test-only) | N samples per failure class; distributions overlap within a configured tolerance. |
| Concurrency | JUnit 5 with real PostgreSQL | Two-tab, parallel transactions, throttle races. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | `LandingRouter` decision-to-route mapping, field validation, statements present in both layouts, holder clearing, separator stripping. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop and M02 widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Focus order, announcements, no focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core (the test product itself) and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, 1.2, 1.3, 1.4, 1.7 | FE unit, e2e | `signInPage.rendersFieldsRoutesAndStatements` (desktop, M02) | 1, 10 |
| RF-1.5 | FE unit | `passwordField.showToggleNeverPersists` | 2 |
| RF-1.6 | FE unit, API contract | `signInPage.blocksEmptyOrMalformedFields`; `authApi.fieldPathProblems` | 2 |
| RF-2.1, 2.6 | BE integration | `signIn.verifiesNormalisedEmailAndPassword` | 2 |
| RF-2.2, 2.3, 2.7, RF-4.3, RNF-1 | API contract + timing | `signIn.failureIsIdenticalAcrossClasses` (unknown, wrong, restricted, throttled); `signIn.timingComparable` | 3 |
| RF-2.4 | BE integration | `signIn.unknownEmailCreatesNoIdentityAndStoresNoPassword` | 2 |
| RF-2.5 | FE unit, e2e | `signInPage.genericFailureKeepsEmailClearsPasswordFocuses` | 1, 2 |
| RF-3.1, 3.2, LC-1 | BE integration, FE e2e | `signIn.unverifiedLandsOnPending` (no session, no challenge, reference carried) | 2 |
| RF-3.3 | API contract | `signIn.unverifiedWrongPasswordGeneric` | 3 |
| RF-4.1, 4.2 | BE unit, integration | `throttlePolicy.progressiveDelaySteps`; `signIn.appliesDelayPerDimension` | 4 |
| RF-4.4, RNF-2 | BE integration | `throttle.restrictionLiftsAfterWindow`; `throttle.noPermanentLockout` (long attacker sequence, then legitimate success after window) | 4 |
| RF-4.5, RF-7.5 | BE integration | `challengeVoid.countsOnBothDimensions` | 4, 5 |
| RF-4.6, RNF-3 | BE integration | `signIn.boundedVerificationWorkUnderFlood` | 4 |
| RF-4.7 | BE integration | `throttle.resetRequestsDoNotCountForSignIn` | 4 |
| RF-4.8, RF-5.3, RNF-4 | Startup | `loginPolicy.startupFailsOnAbsentOrUnboundedValue` (one case per property) | 8 |
| RF-5.1, 5.5 | BE integration | `signIn.enrolledFactorYieldsLimitedState`; `signIn.noFactorCompletes` | 2 |
| RF-5.2, RNF-9 | API contract | `limitedState.exposesNoIdentityTenantOrProjectData`; `transactionToken.cannotCallProtectedEndpoints` | 7 |
| RF-5.4, LC-22 | BE integration, FE unit | `abandon.voidsAndIsIdempotent`; `challengePage.leaveCallsAbandonAndClearsHolder` | 2 |
| RF-6.1, 6.3 | FE unit, e2e | `challengePage.rendersActionsStatementsAndPendingState` | 1, 10 |
| RF-6.2, LC-23 | FE unit, BE unit | `codeField.pasteWithSeparators`; `totpVerify.stripsSeparators` | 2 |
| RF-6.4, LC-5 | BE integration | `totpVerify.validCodeCompletes`; `totpVerify.validOnLastAttemptCompletes` | 5 |
| RF-6.5, LC-3 | BE unit, integration | `totpVerify.rejectsReplayAtOrBelowWatermark` | 5 |
| RF-6.6 | API contract | `challengeRejected.carriesNoCauseOrCount` | 5 |
| RF-6.7, 6.8, LC-7, LC-8 | BE integration | `recoveryCode.consumesOnce`; `recoveryCode.consumedTreatedAsWrong`; `recoveryCode.lastCodeCompletesLeavingZero` | 5 |
| RF-6.9 | FE e2e | `challengePage.controlledRecoveryVoidsAndRoutes` | 1 |
| RF-6.10 | API contract, FE unit | `auth.noDeviceTrustSurfaceExists` (no endpoint, no cookie, no control) | 2 |
| RF-6.11 | API contract | `responses.neverContainSeedOrFactorData` | 7 |
| RF-7.1, LC-6 | BE integration | `totpVerify.lastWrongAttemptVoids` | 5 |
| RF-7.2, LC-4 | BE integration | `totpVerify.expiredTransactionVoidsAndDoesNotConsumeCode` | 5 |
| RF-7.3, 7.4 | BE integration, FE e2e | `voidedTransaction.refusesAllCalls`; `signInPage.afterVoidShowsNoReason` | 5 |
| RF-8.1, 8.6 | BE integration | `complete.issuesSessionAndTouchesNoRunTables` | 2 |
| RF-8.2–8.4, LC-15–LC-17, LC-25 | BE unit, FE unit, integration | `landingRule.{none,oneActive,oneSuspended,oneRevoked,oneDeleting,many}`; `landingRouter.mapsEachDecision`; `landingQuery.usesCurrentMembershipsAfterRevocation` | 6 |
| RF-8.5, LC-18 | FE e2e (with tenant entry stub) | `landing.ownerWithoutFactorIsGatedOnTenantEntry` | 6 |
| RF-8.7 | FE unit | `completion.advancesEpochBeforeLanding` | 6 |
| RF-9.1–9.4, RNF-5, RNF-10 | FE e2e + axe, manual assistive | `a11y.signInChallengeLanding.{en,es,fr}.{desktop,m02}`; manual NVDA / VoiceOver checklist | 10 |
| RF-10.1, 10.2, RNF-7 | BE integration | `audit.entryPerOutcome`; `audit.containsNoSecret` | 9 |
| RF-10.3 | BE integration, FE unit | `logs.containNoCredentialOrCode`; `state.containsNoSecret` | 9 |
| RNF-6, LC-21 | FE e2e | `languageSwitch.onChallengeKeepsTransaction` | 10 |
| RNF-8 | BE integration | `failClosed.absentExpiredOrInvalidRefuses` (transaction, code, config) | 2, 8 |
| LC-2 | API contract | `signIn.restrictedIdentityIsGenericAndCounted` | 3 |
| LC-9 | Concurrency | `twoTabs.oneSuccessOtherRejected` | 5 |
| LC-10 | Concurrency | `parallelTransactions.independentAttemptsSharedThrottle` | 5 |
| LC-11, LC-12 | BE integration | `throttle.correctPasswordRefusedWhileRestricted`; `throttle.networkRestrictionCoversOtherAccounts` | 4 |
| LC-13, LC-14 | BE integration | `transaction.voidedByPasswordChange`; `transaction.voidedByFactorChange` | 5 |
| LC-19, LC-20 | BE unit, FE unit | `email.normalisation`; `signInPage.malformedEmailBlocked` | 2 |
| LC-24 | FE unit, BE integration | `challengePage.reloadDuringPendingResubmitsSameAttempt`; `challenge.resubmitReturnsRecordedOutcome` | 2 |
| US-11 | all adversarial cases above | enumeration, lockout, replay, hijack suites | 1 |

### 5.3 Not tested here
Session refresh and expiry (spec 006), TOTP enrollment (003), recovery beyond the entry link (005), chooser contents (013), workspace creation (012). Their stubs in Login tests return fixed answers.

### 5.4 Exit criteria
Specification §7 items 1–11 map to the "Proves criterion" column; item 11 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-27 identity statuses | `IdentityStatus` enumeration `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING`; `ACTIVE` permits sign-in; `UNVERIFIED` takes the RF-3 branch after a real credential check; `RESTRICTED` and `ERASING` take the generic branch. | Add or rename values; the status branch in `SignInUseCase` is the only place the rule lives. |
| A-29 after a recovery-code sign-in | No prompt; landing proceeds normally. | Add a step between `CompleteSignInUseCase` and landing; `LandingQuery` would return a `RECOVERY_FOLLOW_UP` decision. |
| A-30 sign-in notifications | `SecurityNotificationPort` declared here; first caller plan 002 (RF-3.2); no sign-in event sent. | `CompleteSignInUseCase` calls it for the decided events; templates per S-14. |
| A-31 re-authentication from R34 | Out of scope; R34 is spec 006's. | If R34 reuses Login, add a `returnTo` context carried server-side (never a client URL) and bypass `LandingRouter`. |
| A-32 time tolerance | One global value in `LoginPolicy`. | Per-identity value would move to `TotpFactor`. |
| S-03 mobile challenge | `MfaChallengePage` follows M12 (`431:4162`). | None. |
| S-17 "Need help?" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |
| S-26 already signed in | `/sign-in` shows the form; success issues a new session and the old is left to the session feature's rules. | Redirect by `LandingRouter` when a session exists. |
| A-34 pending reference | Issued by the verification-pending branch after the password is verified; authorises re-addressing only (plan 002 D6). | If password re-entry is chosen for re-addressing, the branch returns `{email}` only. |
| A-35 persistence tooling | Persistence adapters and migrations written against the API and tool named as placeholders. | Rename the adapters' base types and the migration format; no use case changes. |
| A-36 identity-scoped membership read | `MembershipProjectionPort` published by `access` for the signed-in identity. | Change the port's implementation to the decided policy; `LandingRule` is unchanged. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.7) | `SignInPage`, `SecurityStatements`, `shared/ui`, `AuthController` problems | D12, D16 | RF-1 rows |
| RF-2 (2.1–2.7) | `SignInUseCase`, `PasswordVerifier`, dummy path, `GlobalIdentity`, `GenericFailureMessage` | D2 | RF-2 rows |
| RF-3 (3.1–3.3) | `SignInUseCase` verification branch, handoff | D9 | RF-3 rows |
| RF-4 (4.1–4.8) | `AbuseThrottlePolicy`, `ThrottleStore`, `LoginPolicyProperties` | D3, D10 | RF-4 rows |
| RF-5 (5.1–5.5) | `LoginTransaction`, `AbandonChallengeUseCase`, transaction holder | D1, D13 | RF-5 rows |
| RF-6 (6.1–6.11) | `VerifyTotpUseCase`, `VerifyRecoveryCodeUseCase`, `TotpFactor`, `RecoveryCodeSet`, `MfaChallengePage`, `CodeField` | D4, D5, D6, D15 | RF-6 rows |
| RF-7 (7.1–7.5) | `LoginTransaction.void`, `auth.challenge.void`, holder clearing | D1, D12, D13 | RF-7 rows |
| RF-8 (8.1–8.7) | `CompleteSignInUseCase`, `LandingQuery`, `LandingRule`, `LandingRouter`, scope epoch | D7, D8 | RF-8 rows |
| RF-9 (9.1–9.4) | Responsive components, `shared/ui` | D14 | RF-9 row |
| RF-10 (10.1–10.3) | Audit adapter, correlation id, redaction | D11, D12 | RF-10 rows |
| RNF-1 | D2 path | D2 | timing + contract |
| RNF-2, RNF-3 | `AbuseThrottlePolicy`, bounded verification | D3 | RF-4.4, RF-4.6 rows |
| RNF-4 | `LoginPolicyProperties` | D10 | startup |
| RNF-5 | `shared/ui`, views | D15 | e2e + axe, manual |
| RNF-6 | i18n bundles | D16 | language rows |
| RNF-7 | holder, audit payload, problem details | D6, D11, D13 | RF-6.11, RF-10 rows |
| RNF-8 | transaction checks, config validation | D1, D10 | `failClosed` |
| RNF-9 | limited-state authorisation | D1 | RF-5.2 row |
| RNF-10 | one component per screen | D14 | RF-9 row |
| LC-1 … LC-25 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25) |

Self-check: every RF criterion (60), every RNF (10) and every limit case (25) of `specLogin.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
