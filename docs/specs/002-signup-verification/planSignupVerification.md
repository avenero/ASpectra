# Plan 002 — Signup and email verification

**Feature** Local identity creation and email verification · **Plan version** 0.1 · **Date** 26 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/002-signup-verification/specSignupVerification.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; features F-IDN-01 and F-IDN-02 as the subject, F-IDN-03, F-IDN-04 and F-IDN-07 at the boundary, F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG, transactional outbox) and FE board `46:7847` (feature-modular SPA DAG) · **Sibling plan** `specs/001-login/planLogin.md` (shared bounded context, shared feature module, shared throttle policy).

This plan is the HOW. It adds no behaviour the specification does not state. Where the specification left a question to the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (§8 of the spec: A-27, A-28, A-33, A-34, A-35, S-14, S-17, S-23, S-24, S-25, S-26, S-27), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); their FR identifiers are listed in the specification.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P3** Browser = intent | The FE never decides whether an address is new, verified or pending; every response to signup, resend, re-addressing and link opening is one server-defined state. | RF-3.4, RF-6.4, RF-8.6 |
| **P6** Fail closed | A token that is absent, malformed, expired, superseded or bound to a missing identity never verifies; a missing configuration value stops startup; a missing pending reference changes nothing. | RF-8.5, RF-10.2, RNF-9 |
| **P10** Fixed stack | Java 21, Spring Boot 3.5, PostgreSQL, the maintained Spring Security `PasswordEncoder`; React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. No queue, cache or scheduler product is added: asynchronous work uses the existing PostgreSQL durable jobs and transactional outbox named in P11. Test-only tools are flagged in §5; the persistence access API and the migration tool are placeholders until A-35 is decided. | whole plan |
| **P11** Architecture styles | BE: use cases inside the existing `identity` context; outbound email behind a port with a transactional outbox; jobs claimed from PostgreSQL. FE: the same `identity-access` feature module as Login; TanStack Query holds server facts; no Zustand (no unsaved fork). | §2 |
| **P12** Tenancy gates | Identity, verification-link, throttle and outbox rows for this feature are platform rows (no tenant exists yet). Nothing here reads a tenant table; no membership or grant is written (RF-3.5, RF-11.1). | RF-3.5, RF-11, RNF-10 |
| **P13** Configured values | Link expiry, unverified-identity window, resend thresholds and restriction windows, the current Terms and Privacy versions, and the public origin are one validated configuration record; startup fails if any is absent or unbounded. | RF-4.6, RF-10.2, RNF-4 |
| **P14** Release bar | WCAG 2.2 AA on signup and all verification states, EN / ES / FR, automated and manual evidence. | RF-13, RNF-6, RNF-7, RNF-11 |
| **F-X-02** Request integrity | Signup, resend, re-addressing and link opening are consequential writes with no client auto-retry. A re-submission after a lost answer is safe by construction: signup on a pending address replaces it (same end state), resend and re-addressing issue a new newest link (the previous one dies), and link opening is idempotent (RF-8.2). No replay queue exists. | RF-3.3, RF-4.4, RF-8.2, RNF-3 |
| **F-X-03** Secrets and safe problems | Passwords exist only in the request and the encoder; tokens exist only in the message and as a non-reversible form at rest; problem details carry safe codes; browser storage holds nothing. | RF-2.4, RF-4.1, RF-12, RNF-8 |
| **F-X-05** Audit | Every event of RF-12.1 goes through the `audit` context port in the same transaction. | RF-12 |
| **F-X-06** Email port and configuration | The transactional email adapter sits behind `TransactionalEmailPort`; the outbox decouples the request from delivery; no vendor is assumed. | RF-4.2, RNF-5 |
| **F-X-04 / F-X-07** Accessibility and responsive surfaces | R04 follows M10 (`431:4100`) and N01 follows M11 (`431:4135`); tablet follows the D01 responsive policy under the WCAG rules (FR-X-055, owned by F-X-04). | RF-13 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity` (shared with Login)

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `GlobalIdentity` (as in plan 001) | Adds `createUnverified(name, email, credential, acknowledgement)`, `replacePending(name, credential, acknowledgement)` (allowed only while `UNVERIFIED`), `readdress(newEmail)` (only while `UNVERIFIED`), `markVerified(now)` (moves `UNVERIFIED` to `ACTIVE` and records `emailVerifiedAt`; status is the single authority). `IdentityStatus` working values `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING` (A-27, §6); the signup branch is chosen by status: `UNVERIFIED` → replace, `ACTIVE` → notice, the others → no change. | RF-3.1, RF-3.3, RF-3.7, RF-7.2, RF-7.6, RF-8.1, RF-10.3 |
| | `EmailAddress` | Value object: trims, lower-cases, validates shape; equality on the normalised form. | RF-1.4, RF-1.6, LC-16 |
| | `DisplayName` | Value object: trimmed, non-blank; length and character rules from configuration (A-27). | RF-1.4, LC-17 |
| | `TermsAcknowledgement` | Terms version, Privacy version, acknowledged-at. | RF-9.1 |
| | `VerificationLink` | Token hash, identity id, purpose `EMAIL_VERIFICATION`, `issuedAt`, `expiresAt`, `supersededAt`, `usedAt`. Methods: `isNewestAndLive(now)`, `markUsed(now)`, `supersede(now)`. Pure. | RF-4.1, RF-4.4, RF-8.1, RF-8.2, RF-8.5 |
| | `VerificationOutcome` | Result of an open: `VERIFIED_NOW`, `ALREADY_VERIFIED`, `UNAVAILABLE(cause)` with causes `EXPIRED`, `SUPERSEDED` (covers a link invalidated by a verification through another link), `MALFORMED`, `NO_IDENTITY`; `toResponse()` maps every cause to one value — the cause is for audit only, never for the response. | RF-8.1, RF-8.2, RF-8.5, RF-8.6 |
| | `PendingReference` | Random capability stored hashed, bound to one identity, with `issuedAt`, `expiresAt`, `revokedAt`; valid only while unrevoked, unexpired and its identity is `UNVERIFIED`. Issued at signup, at the sign-in handoff (plan 001 D9), on replacement and on re-addressing; revoked on verification, replacement, re-addressing and expiry. Authorises re-addressing only. | RF-5.4, RF-7.4, RNF-10 |
| | `AbuseThrottlePolicy` (plan 001) | Reused; the dimension keys `RESEND_ADDRESS` and `RESEND_NETWORK` are added, with their own configured steps; the arithmetic is unchanged. | RF-6.3, RF-7.5 |
| | `SignupPolicy` | Immutable record: link expiry, unverified window, resend steps and windows, name constraints, current Terms and Privacy versions, public origin, notice send window, pending-reference lifetime, expiry-job cadence and batch size. | RF-4.6, RF-10.2, RNF-4 |
| `identity.application` | `SignUpUseCase` | Validates fields and password policy; normalises email; always runs one password encoding; branches on the existing identity's status: none → create + issue link + issue reference; `UNVERIFIED` → replace + supersede links + issue link + new reference (old revoked); `ACTIVE` → no change + enqueue the security notice (once per configured window); `RESTRICTED` / `ERASING` → no change, no message, audit `SIGNUP_ATTEMPT_IGNORED`. Every branch returns the same response shape `{pendingRef, email}` — the non-pending branches carry an unbound decoy reference — and appends its audit event in one transaction. | RF-1.4, RF-2.2–2.4, RF-3.1–3.7, RF-9.1 |
| | `IssueVerificationLink` (internal service) | Generates the token, stores only its non-reversible form, supersedes every live link of the identity, enqueues the verification message (locale, address, link to the public origin) in the outbox, and audits `LINK_ISSUED{cause}` for every cause (signup, replacement, resend, re-addressing). | RF-4.1–4.5, RF-12.1 |
| | `ResendVerificationUseCase` | Evaluates both resend throttles; if an `UNVERIFIED` identity exists for the address → `IssueVerificationLink`; otherwise nothing; always the same response. | RF-6.1–6.4 |
| | `ReaddressPendingIdentityUseCase` | Resolves the pending reference to an `UNVERIFIED` identity (RF-7.4 first: no identity → nothing, no notice); new address normalised; equal to the requester's own pending address → handled as a resend; if free → `readdress`, supersede links, issue to the new address, new reference; if owned by an `ACTIVE` identity → security notice to that identity; otherwise nothing. Counts as a resend for throttling. Same response always. | RF-7.1–7.6 |
| | `OpenVerificationLinkUseCase` | Hashes the presented token; loads the link; if newest and live and identity `UNVERIFIED` → atomic verify (`UPDATE … WHERE used_at IS NULL AND status = 'UNVERIFIED'`), supersede the rest, revoke pending references, audit `VERIFIED`; if used and still newest and unexpired → `ALREADY_VERIFIED`; else `UNAVAILABLE`. Creates no session. | RF-8.1, RF-8.2, RF-8.5–8.9 |
| | `PasswordPolicyQuery` | Returns the current rules for display. | RF-2.1 |
| | `ExpireUnverifiedIdentitiesJob` | Durable job (sharedsupport/jobs): claims `UNVERIFIED` identities whose `createdOrReplacedAt + window < now`, deletes them with their links, pending references, acknowledgement record and throttle rows, audits `UNVERIFIED_IDENTITY_REMOVED`. Never touches a verified identity. | RF-10.1–10.4 |
| | Ports | `IdentityRepository`, `AcknowledgementRepository`, `VerificationLinkRepository`, `PendingReferenceStore`, `ThrottleStore` (plan 001), `TokenGenerator` and `TokenHasher` (→ `sharedsupport/crypto`), `PasswordVerifier` (encoder), `PasswordPolicyPort`, `TransactionalEmailPort` (behind the outbox), `SecurityNotificationPort` (now with a caller: RF-3.2, RF-7.3), `AuditPort`, `Clock`, `JobScheduler` (→ `sharedsupport/jobs`). | all |
| `identity.infrastructure` | Persistence | Plan 001's migration creates `identity` (with `email_verified_at` and the unique index on the normalised email), `identity_credential` and `signin_throttle`; this plan only extends. `identity` gains `created_or_replaced_at`; new `identity_acknowledgement` (identity id, Terms version, Privacy version, acknowledged-at — kept off the identity row so FR-IDN-001's "only email, display name and status" stays literal); new `verification_link` and `pending_reference`; `signin_throttle` gains the resend dimension values; the existing `outbox_message`. Persistence access API and migration tool per A-35 (placeholders). | RF-3, RF-4, RF-6, RF-7, RF-9.1, RF-10 |
| | Outbox relay | Existing transactional-outbox relay publishes `outbox_message` rows to the email adapter; idempotent per message id; failures retried by the relay's policy, never surfaced to the signup request. | RF-4.2, RNF-1, RNF-5 |
| | Email adapter | Implements `TransactionalEmailPort` for whichever provider operations chooses (A-10/A-11 are outside this plan); templates `verification` and `signup-attempt-notice` in EN / ES / FR (S-23 for the second's content). | RF-4.2, RF-4.5, RNF-7 |
| | `SignupPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding into `SignupPolicy`; startup fails on any absent or unbounded value. | RF-4.6, RF-10.2, RNF-4 |
| | Audit adapter | Events: `IDENTITY_CREATED`, `IDENTITY_REPLACED`, `IDENTITY_READDRESSED`, `LINK_ISSUED{cause}`, `VERIFIED`, `LINK_OPEN_FAILED{cause}`, `SIGNUP_ATTEMPT_NOTICE_SENT`, `UNVERIFIED_IDENTITY_REMOVED`. Payload: identity id, correlation id, network key, timestamp. Never password, token or message body. | RF-12.1–12.3 |
| `identity.api` | `SignupController` — `/api/auth` | `GET password-policy` → 200 rules · `POST sign-up` {name, email, password, acknowledgement} → 202 `{pendingRef, email}` in every branch (same shape, status and timing; the reference value is random in every branch), or 400 problem with field paths · `POST verification/resend` {email} → 202 `{}` always · `POST verification/readdress` {pendingRef, newEmail} → 202 `{pendingRef, email}` always · the UI locale travels in a request header on all three (request metadata, not a collected field — FR-IDN-010) · `POST verification/open` {token} → 200 `{state: CONFIRMED, email}` or 410 problem `auth.verification-link.unavailable` (one code for every cause). No endpoint creates a session. | RF-1.4, RF-2.1, RF-3.4, RF-6.4, RF-7, RF-8 |

**Cross-context touch points.** `audit` (events), `sharedsupport` (`crypto` token generation and hashing, `jobs` durable job, `tx`, `contextIds`). The email adapter is an outer adapter of `identity`; no other context is touched. Nothing reads or writes tenant, membership, grant or invitation tables (RF-3.5, RF-11.1, RF-11.2).

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/sign-up`, `/verify-email` (pending), `/verify-email/open` (link target, token in the query of the SPA route; confirmed and unavailable are states of this route — the consumption path is A-33). | RF-1, RF-5, RF-8 |
| | `useSignUp()`, `useResend()`, `useReaddress()`, `useOpenLink()`, `usePasswordPolicy()` | Typed intent hooks; attach the UI locale header; return server state verbatim. | RF-2–RF-8 |
| | `contracts/signup` (in `src/contracts`) | Request / response / problem types and runtime guards for the five endpoints. | all |
| Views | `SignupPage` | R04: name, email, password with show toggle, policy rules from the server, acknowledgement checkbox (never pre-ticked), "Create account & verify email", "Already have an account? Sign in", language switch, the two statements, Privacy / Terms routes. | RF-1.1–1.7, RF-2.1, RF-9.2, RF-9.4 |
| | `VerifyEmailPage` | N01 with three rendered states — `pending` (address, generic text, Resend, Use a different email, Back to sign in, statements), `confirmed` (heading, status, address, statements, Continue to sign in), `unavailable` (heading, status, Request a new verification link, Back to sign in, statements); Privacy / Terms routes on all three states (external operator documents, S-14); "Help" link opening N43 HelpShortcutsPanel (`430:4129`) (S-17). | RF-5.1, RF-5.2, RF-8.3, RF-8.5, RF-9.4 |
| | `ReaddressEntry` | Where the new address is typed (S-25, §6: inline field under the pending state). | RF-7.1 |
| | `SecurityStatements` (plan 001) | Extended with the signup and verification statements so desktop and reflow cannot drift. | RF-1.2, RF-5.2, RF-8.3, RF-8.5, RF-13.1 |
| State | TanStack Query | Mutations for sign-up, resend, readdress, open; query for password policy; no retry on mutations. | RF-2–RF-8 |
| | Pending holder | Module-private in-memory store with `{pendingRef, email}` from the sign-up response (or from the sign-in handoff of plan 001 D9); cleared on route leave. Nothing in localStorage. On a reload the pending page offers resend by typed address only (re-addressing needs the reference). | RF-5.3, RF-5.4, RF-7.1, RNF-8 |
| | Link opener | `/verify-email/open` mounts, reads the token from the route, submits it once, renders `confirmed` or `unavailable`; never stores the token. | RF-8.1, RF-8.2, RF-8.5, RF-8.9 |
| Routing | Handoffs | Sign-in (plan 001) → `/verify-email` pending with `{pendingRef, email}`; `confirmed` → `/sign-in` (S-27: empty form); `unavailable` → "Request a new verification link" opens the pending state's resend with an address field; "Back to sign in" → `/sign-in`. | RF-5.3, RF-8.4, RF-8.7 |
| Shared used | `shared/api` (correlation id, no POST retry), `shared/errors` (field paths), `shared/i18n` (bundle keys for every statement, state and message; locale sent with each request so the message language matches), `shared/ui` (accessible fields, checkbox, live region, 44 px targets). | RF-1.4, RF-12.2, RF-13.2, RNF-6, RNF-7, LC-23 |

---

## 3. Behaviour-to-module trace

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `GET password-policy` | `SignupPage` |
| RF-1.2 | — | `SecurityStatements` (verify first, no queue, no org data) |
| RF-1.3 | `SignUpUseCase` accepts the four fields only; the API record has no other fields (the UI locale travels in a header) | Form has no other fields |
| RF-1.4 | Server-side validation → 400 field paths | Field validation, associated messages, no submit |
| RF-1.5 | — | `PasswordField` show toggle; never persisted |
| RF-1.6 | `EmailAddress`, `DisplayName` normalisation; unique index on normalised email | Trim before submit |
| RF-1.7 | — | "Already have an account? Sign in" route |
| RF-2.1 | `PasswordPolicyQuery` | `usePasswordPolicy()` rendered as "supplied by the current policy" |
| RF-2.2 | `SignUpUseCase` validates against `PasswordPolicyPort` regardless of client | — |
| RF-2.3 | 400 problem naming unmet rules by field path | Message on the password field |
| RF-2.4 | `PasswordVerifier` encoding; `identity_credential` holds the encoded form only | — |
| RF-3.1 | `SignUpUseCase` create branch; `IssueVerificationLink` | Route to pending |
| RF-3.2 | Verified-collision branch: no change; `SecurityNotificationPort` → outbox `signup-attempt-notice` | Same pending state |
| RF-3.3 | `GlobalIdentity.replacePending`; supersede links; issue | Same pending state |
| RF-3.4 | One response record, one status, one password encoding and one outbox or audit write per branch | One rendering |
| RF-3.5 | No tenant / membership / grant repository in the use case | — |
| RF-3.6 | No linking or domain logic exists; identity lookup is by exact normalised address only | — |
| RF-3.7 | `RESTRICTED` / `ERASING` → no change, no message, audit `SIGNUP_ATTEMPT_IGNORED` | Same pending state |
| RF-4.1 | `TokenGenerator` (256-bit random), `TokenHasher`; `verification_link.token_hash` | — |
| RF-4.2 | Outbox message addressed to the identity's address; `TransactionalEmailPort` | — |
| RF-4.3 | Link built from `SignupPolicy.publicOrigin`, never from request headers | — |
| RF-4.4 | `IssueVerificationLink` supersedes every live link first | — |
| RF-4.5 | `verification` template text in three languages | — |
| RF-4.6 | `SignupPolicyProperties` | — |
| RF-5.1 | — | `VerifyEmailPage` pending |
| RF-5.2 | — | `SecurityStatements` pending block |
| RF-5.3 | Sign-in (plan 001 D9) returns `pendingRef` for an unverified identity | Handoff into pending |
| RF-5.4 | No session endpoint is involved; the pending reference authorises re-addressing only | Page shows the address only |
| RF-6.1 | `ResendVerificationUseCase` → issue | Neutral "message requested" confirmation |
| RF-6.2 | No `UNVERIFIED` identity → nothing | Same confirmation |
| RF-6.3 | `AbuseThrottlePolicy` on `RESEND_ADDRESS` / `RESEND_NETWORK`; throttled → nothing | Same confirmation |
| RF-6.4 | One 202 response; the outbox makes sending asynchronous | — |
| RF-7.1 | `POST verification/readdress` | `ReaddressEntry` |
| RF-7.2 | `ReaddressPendingIdentityUseCase` free-address branch | Pending with new address |
| RF-7.3 | Collision branch: no change; notice when the owner is `ACTIVE`; the requester's own address → resend | Same |
| RF-7.4 | Reference resolves to no `UNVERIFIED` identity → nothing (evaluated before RF-7.3, so no notice) | Same |
| RF-7.5 | Readdress updates the resend throttle counters | — |
| RF-7.6 | `readdress` allowed only while `UNVERIFIED`; verified change belongs to personal settings | No entry on a verified identity |
| RF-8.1 | `OpenVerificationLinkUseCase` atomic verify | Link opener → `confirmed` |
| RF-8.2 | `ALREADY_VERIFIED` branch | `confirmed` |
| RF-8.3 | — | `confirmed` state content |
| RF-8.4 | No session issued by any endpoint here | "Continue to sign in" → `/sign-in` |
| RF-8.5 | `UNAVAILABLE` → 410 single code | `unavailable` state content |
| RF-8.6 | One code, one path cost (hash + lookup) for every cause | One rendering |
| RF-8.7 | — | "Request a new verification link" → pending resend with address field |
| RF-8.8 | Open touches `identity.status`, `email_verified_at`, links only | No other effect |
| RF-8.9 | Token is the only input; no device or session binding | Route works anywhere |
| RF-9.1 | `TermsAcknowledgement` persisted on create / replace from `SignupPolicy` versions | — |
| RF-9.2 | 400 if acknowledgement absent | Checkbox required, never pre-ticked |
| RF-9.3 | No re-acknowledgement logic exists | — |
| RF-9.4 | — | Privacy / Terms routes on all four surfaces |
| RF-10.1 | `ExpireUnverifiedIdentitiesJob` | — |
| RF-10.2 | `SignupPolicyProperties.unverifiedWindow` | — |
| RF-10.3 | Job predicate `status = 'UNVERIFIED'` | — |
| RF-10.4 | Only `replacePending` and `readdress` can touch the address before expiry | — |
| RF-11.1 | No tenant write anywhere in the context's signup path | — |
| RF-11.2 | No invitation repository touched | — |
| RF-11.3 | No factor repository touched | — |
| RF-11.4 | No administrative endpoint exists for identities | — |
| RF-12.1 | Audit adapter events | — |
| RF-12.2 | Payload shape; a collision branch (`ACTIVE`, `RESTRICTED`, `ERASING`) records no credential | — |
| RF-12.3 | Token only in the message and the SPA route on open; never logged | Token never stored; route left after open |
| RF-13.1 | — | Reflow of `SignupPage` and `VerifyEmailPage` |
| RF-13.2 | — | `shared/ui` targets, labels, focus |

---

## 4. Technical decisions

**D1 · Email is sent through the transactional outbox, never inside the request.**
*Why.* P11 names the outbox; it makes identity creation, link issue and message enqueue one atomic commit; the request's duration no longer depends on a provider, which is what keeps RNF-1 (comparable timing) achievable; provider failure is retried by the relay instead of being shown to the person, which would otherwise disclose whether a message was attempted.
*Discarded.* Synchronous send in the request — timing and failure leaks, lost message on crash after commit. Fire-and-forget thread — lost on restart, not idempotent.
*Covers.* RF-4.2, RF-6.4, RNF-1, RNF-5.

**D2 · Every signup branch performs the same work: one password encoding, one lookup, one write.**
*Why.* RF-3.4 requires identical response and comparable timing whether the address is new, pending, verified or non-registrable. The encoding is the dominant cost and is always executed (its result is discarded when nothing is stored); each branch runs the same sequence of steps with a bounded number of writes in one transaction and returns the same response shape — the reference value is random in every branch (a decoy where no pending identity exists), so responses are equal in shape, status and timing, not byte for byte.
*Discarded.* Early return on collision — timing leak. A distinct "already registered" problem — the specification rejected it.
*Covers.* RF-3.1–3.4, RF-3.7, RNF-1, LC-1, LC-2, LC-24.

**D3 · Verification tokens are 256-bit random values stored only as a fast cryptographic hash; one live link per identity.**
*Why.* A random high-entropy token needs no work factor (unlike a password): a plain cryptographic hash makes lookup cheap and leaves nothing usable at rest. "Newest only" is implemented by superseding every live link at issue time, so the invariant holds without comparing timestamps at open time.
*Discarded.* Encoding tokens with the password encoder — needless cost on every open and no security gain. Storing the token in clear — F-X-03. Multiple live links — violates RF-4.4.
*Covers.* RF-4.1, RF-4.4, RNF-2, LC-3, LC-4.

**D4 · Opening a link is an atomic, idempotent state change executed by the SPA route.**
*Why.* The link targets the SPA route `/verify-email/open`; on mount the route submits the token to `POST verification/open`. The use case verifies with a single conditional update (`WHERE used_at IS NULL AND status = 'UNVERIFIED'`): exactly one of two concurrent opens changes state (LC-6), and a later open of the same newest, unexpired link returns `ALREADY_VERIFIED` (RF-8.2). Whether an automated fetcher runs the application (and so verifies) or only loads the page shell, the person's later open within the link's lifetime shows the confirmed state (LC-5); the consumption path itself is register item A-33.
*Discarded.* A server-rendered GET endpoint that verifies on request — a state change on GET is exposed to every prefetcher and preview, and it bypasses the SPA's accessible, localised rendering. Requiring an explicit confirm action — reversed by the specification (Q3) because it needed a screen that was not designed at the time.
*Covers.* RF-8.1, RF-8.2, RF-8.8, RF-8.9, RNF-3, LC-5, LC-6, LC-22.

**D5 · One `unavailable` outcome with a single problem code and a constant-cost path.**
*Why.* RF-8.6 requires that neither the state nor its timing reveal the cause. The use case always hashes the token and performs one lookup; the cause (`EXPIRED`, `SUPERSEDED`, `MALFORMED`, `NO_IDENTITY`) goes to audit only.
*Discarded.* Distinct codes with identical text — leaks through the response.
*Covers.* RF-8.5, RF-8.6, RF-12.1, LC-3, LC-7, LC-8, LC-9.

**D6 · Re-addressing requires an opaque pending reference, not just the old address.**
*Why.* The specification lets a pending identity be re-addressed without a password (Q5) and requires that a stranger cannot take over a pending identity (US-10). A reference issued when the password was just set (signup) or just proven (the sign-in handoff), and renewed on replacement and re-addressing, binds the action to the person who created or proved the pending identity. A request without a valid reference gets the generic pending response and changes nothing, which is exactly RF-7.4's behaviour.
*Discarded.* Re-addressing by old address alone — anyone who knows a pending address could redirect its verification. Requiring the password again — not chosen in Q5 (A-34 may still choose it).
*Covers.* RF-7.1–7.4, RF-5.4, LC-13, LC-14, LC-15, US-10.

**D7 · Replacement of a pending identity is an in-place update under the unique email index.**
*Why.* The identity row is the owner of the address; replacing name, credential and acknowledgement in place keeps the address reserved without a delete-insert race, and the unique index on the normalised address guarantees that two concurrent signups for the same new address yield one identity.
*Discarded.* Delete and recreate — a window in which a third signup could take the address. Keeping the first credential — rejected by the specification (Q2).
*Covers.* RF-3.3, RF-10.4, LC-2, LC-16.

**D8 · Resend and re-addressing share Login's `AbuseThrottlePolicy` with their own dimension keys and configured steps.**
*Why.* The policy is pure and already tested; separate dimension keys (`RESEND_ADDRESS`, `RESEND_NETWORK`) make the throttle independent of sign-in and reset as RF-6.3 requires, while re-addressing counts as a resend (RF-7.5) by using the same keys.
*Discarded.* A second policy implementation — duplication. Sharing keys with sign-in — a resend flood would throttle sign-in.
*Covers.* RF-6.3, RF-7.5, LC-11.

**D9 · Unverified identities expire through a durable PostgreSQL job.**
*Why.* RF-10 requires removal after a configured window independent of traffic. The existing durable-jobs facility (P11) claims due identities in bounded batches, deletes them with their links and throttle rows, and audits each removal; verified identities are excluded by predicate, not by code path.
*Discarded.* Check-on-access only — an address nobody touches is never freed. A database-level TTL extension — not in the stack. An external scheduler — a new component.
*Covers.* RF-10.1–10.4, LC-9.

**D10 · Password rules come from one server policy port, displayed by the FE and enforced by the BE.**
*Why.* FR-IDN-011: the client never hard-codes the policy; the same port serves display and validation so the two cannot diverge, and a policy tightened after the page loaded is still enforced (LC-18).
*Discarded.* Rules bundled in the FE — forbidden. Validation only on the client — bypassable.
*Covers.* RF-2.1–2.3, LC-18.

**D11 · Terms and Privacy versions are configuration; the acknowledgement stores them with the identity.**
*Why.* RF-9.1 needs the version presented at the time; a validated configuration value is the only source that cannot be defaulted (P13) and changes only by deployment, which is what "no re-acknowledgement in this release" (RF-9.3) assumes.
*Discarded.* A versions table with admin editing — an operator surface that is not in the baseline. Storing only a boolean — rejected by the specification (Q6).
*Covers.* RF-9.1–9.3, RF-4.6.

**D12 · The security notice to an existing identity is a second outbox template, sent at most once per configured window per identity.**
*Why.* RF-3.2 and RF-7.3 require the notice; the outbox gives the same atomicity as D1; a per-identity send window stops a signup flood from becoming a mail flood to the victim. Content is S-23 (§6).
*Discarded.* Sending every time — a harassment vector. Not sending — rejected by the specification (Q1).
*Covers.* RF-3.2, RF-7.3, LC-1, LC-13.

**D13 · The verification link points at the configured public origin; the request's host is never used.**
*Why.* RF-4.3; a request-controlled host would let an attacker mint links to a look-alike domain.
*Discarded.* Deriving the origin from `Host` or forwarded headers.
*Covers.* RF-4.3.

**D14 · Pending state is carried in memory; the token is never stored by the FE.**
*Why.* RNF-8 forbids browser storage for anything from this feature; the pending reference and address live in the module store and die with the route; the open route submits the token once and leaves it in the URL history only as long as the browser keeps it (the token is single-effect and expiring, so this residue cannot verify anything after use beyond the idempotent confirmed view).
*Discarded.* sessionStorage / localStorage — F-X-03. Pending reference in the URL — leaks into logs and history.
*Covers.* RF-5.4, RF-12.3, RNF-8.

**D15 · The message language is the locale sent with the request that triggered it.**
*Why.* RNF-7 and LC-23: the BE never renders language itself but must pick a template language; the FE sends the current UI locale as a request header with sign-up, resend and re-addressing (metadata, not a collected field), and the outbox message carries it.
*Discarded.* Accept-Language sniffing — follows the browser, not the person's switch. One language — violates RNF-7.
*Covers.* RF-4.5, RNF-7, LC-23.

**D16 · No dedicated mobile pages; `SignupPage` and `VerifyEmailPage` reflow under the WCAG rules.**
*Why.* M10 and M11 are the drawn counterparts of R04 and N01; FR-X-055 requires AA reflow at every other width, and `SecurityStatements` shared across widths keeps every statement present (RNF-11).
*Discarded.* Separate mobile components — drift.
*Covers.* RF-13.1, RF-13.2, RNF-11.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, Spring Boot test support) | `EmailAddress`, `DisplayName`, `VerificationLink` state, `VerificationOutcome` mapping, `AbuseThrottlePolicy` with resend steps, job predicate. |
| BE integration | Spring Boot test + **real PostgreSQL** (Testcontainers, **test-only**) | Use cases through real adapters: unique index, conditional update, outbox rows, throttle upserts, audit rows, job runs with a controllable clock. |
| API contract | MockMvc (**test-only**, Spring Boot test support) | Status codes, single problem codes, identical response records across branches, no token or credential in any response. |
| Timing comparability | JUnit 5 statistical harness (**test-only**) | Signup, resend, readdress and open: N samples per address class; distributions overlap within tolerance. |
| Concurrency | JUnit 5 + real PostgreSQL | Two-tab open, two signups for one address, resend races. |
| Outbox / email | Integration with a fake `TransactionalEmailPort` | Message addressed, localised, carrying the public-origin link; notice send-window. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Field validation, checkbox rule, state rendering, holder clearing, opener submits once. |
| Module boundary | ArchUnit (**test-only**) | The application module depends on no Spring or persistence class. |
| FE e2e + accessibility | Playwright for the browser + axe (**test-only**) | Journeys EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS / iOS); versions in the release-support matrix | Focus order, announcements, labels, 44 px targets on all four surfaces. |

### 5.2 Test matrix

| Spec item | Level | Test case | Proves criterion |
|---|---|---|---|
| RF-1.1, 1.2, 1.3, 1.7, RF-9.4 | FE unit, e2e | `signupPage.rendersFieldsStatementsAndRoutes` | 1, 10 |
| RF-1.4, RF-9.2, LC-17, LC-19 | FE unit, API contract | `signupPage.blocksInvalidFields`; `signUpApi.fieldPathProblems` | 2 |
| RF-1.5 | FE unit | `passwordField.showToggleNeverPersists` | 2 |
| RF-1.6, LC-16 | BE unit, integration | `emailAddress.normalisation`; `signUp.sameAddressDifferentCaseIsOneIdentity` | 2 |
| RF-2.1 | FE unit, API contract | `passwordPolicy.displayedFromServer` | 2 |
| RF-2.2, 2.3, LC-18 | BE integration, API contract | `signUp.enforcesCurrentPolicyRegardlessOfClient`; `signUp.namesUnmetRules` | 2 |
| RF-2.4 | BE integration | `credential.storedNonReversibly` | 2 |
| RF-3.1 | BE integration | `signUp.newAddressCreatesUnverifiedAndIssuesLink` | 2 |
| RF-3.2, LC-1 | BE integration, outbox | `signUp.verifiedCollisionChangesNothingAndNotifies` | 5, 9 |
| RF-3.3, LC-2 | BE integration | `signUp.pendingCollisionReplacesAndSupersedesLinks` | 5 |
| RF-3.4, RNF-1 | API contract + timing | `signUp.responseIdenticalAcrossClasses`; `signUp.timingComparable` | 3 |
| RF-3.5, RF-11.1 | BE integration | `signUp.writesNoTenantMembershipOrGrant` | 2 |
| RF-3.6, LC-25 | BE integration | `signUp.noAutoLinkNoDomainJoin` | 2 |
| RF-3.7, LC-24 | BE integration, outbox | `signUp.nonRegistrableChangesNothingNoNotice` | 9 |
| RF-4.1, RNF-2 | BE unit, integration | `token.entropyAndHashOnlyAtRest` | 4 |
| RF-4.2, RF-4.5, RNF-5 | Outbox | `verificationMessage.addressedToIdentityWithStatements` | 2 |
| RF-4.3 | Outbox | `verificationLink.usesPublicOriginNotRequestHost` | 2 |
| RF-4.4, LC-3, LC-4 | BE integration | `issueLink.supersedesEarlierLinks`; `openLink.supersededIsUnavailable` | 4 |
| RF-4.6, RF-10.2, RNF-4 | Startup | `signupPolicy.startupFailsOnAbsentOrUnboundedValue` | 7 |
| RF-5.1, 5.2 | FE unit, e2e | `verifyEmailPage.pendingStateContent` | 1, 10 |
| RF-5.3, LC-20, LC-21 | FE e2e with plan 001 | `signIn.unverifiedLandsOnPending`; `signIn.unverifiedWrongPasswordGeneric` | 2 |
| RF-5.4, RNF-10 | API contract, FE unit | `pendingReference.authorisesReaddressOnly`; `noEndpoint.issuesSession`; `holder.clearsOnLeaveAndNeverTouchesStorage` | 2 |
| RF-6.1 | BE integration | `resend.issuesNewLinkForPending` | 2 |
| RF-6.2, LC-10 | BE integration, outbox | `resend.unknownOrVerifiedSendsNothingSameResponse` | 3 |
| RF-6.3, LC-11 | BE integration | `resend.throttledSendsNothingSameResponse`; `resend.independentOfSignInAndReset` | 3 |
| RF-6.4 | API contract + timing | `resend.timingComparable`; `readdress.timingComparable` | 3 |
| RF-7.1 | FE unit | `readdressEntry.rendersAndSubmits` | 1 |
| RF-7.2 | BE integration | `readdress.freeAddressReaddressesAndReissues` | 5 |
| RF-7.3, LC-13, LC-14 | BE integration, outbox | `readdress.collisionChangesNothing`; `readdress.verifiedOwnerNotified` | 5, 9 |
| RF-7.4, LC-15 | BE integration | `readdress.confirmedOrExpiredIdentityChangesNothing` | 5 |
| RF-7.5, LC-12 | BE integration | `readdress.countsAsResend`; `readdress.sameAddressIsResend` | 5 |
| RF-7.6 | API contract | `readdress.verifiedIdentityImpossible` | 5 |
| RF-8.1, RF-8.8 | BE integration | `openLink.verifiesOnceAndTouchesOnlyIdentityAndLinks` | 4 |
| RF-8.2, LC-6 | Concurrency | `openLink.twoTabsOneVerifiesBothConfirmed`; `openLink.reopenIsIdempotent` | 4 |
| RF-8.3, RF-8.4 | FE unit, e2e | `verifyEmailPage.confirmedStateContent`; `continueToSignIn.noSession` | 1, 10 |
| RF-8.5, RF-8.6, LC-7, LC-8, LC-9 | API contract + timing | `openLink.unavailableSingleCodeAllCauses`; `openLink.timingComparable` | 3, 4 |
| RF-8.7 | FE e2e | `unavailable.requestNewLinkOpensResend` | 1 |
| RF-8.9, LC-22 | FE e2e | `openLink.worksInFreshBrowser` | 4 |
| RF-9.1 | BE integration | `acknowledgement.versionsAndTimeStored` | 2 |
| RF-9.3 | BE integration | `termsChange.noEffectOnExistingIdentity` | 2 |
| RF-10.1, 10.3, 10.4, LC-9 | BE integration (job) | `expiryJob.removesDueUnverifiedOnly`; `expiryJob.freesAddressAndKillsLinks` | 6 |
| RF-11.2, 11.3, 11.4 | API contract, BE integration | `verification.touchesNoInvitationOrFactor`; `noAdminEndpointExists` | 2 |
| RF-12.1, 12.2, RNF-8 | BE integration | `audit.entryPerEvent`; `audit.containsNoSecretOrMessageBody` | 8 |
| RF-12.3 | BE integration, FE unit | `logs.containNoPasswordOrToken`; `opener.doesNotStoreToken` | 8 |
| RF-13.1, 13.2, RNF-6, RNF-11 | FE e2e + axe, manual | `a11y.signupAndVerification.{en,es,fr}.{desktop,tablet,mobile}`; manual checklist | 10 |
| RNF-3, LC-5 | BE integration | `openLink.firstOpenVerifiesLaterOpensConfirm` | 4 |
| RNF-7, LC-23 | Outbox | `message.languageFollowsRequestLocale` | 10 |
| RNF-9 | BE integration | `failClosed.absentExpiredSupersededMalformedNeverVerify` | 4 |
| US-10 | adversarial suites above | enumeration, takeover (pending reference), scanner, replay | 1 |

### 5.3 Not tested here
Sign-in behaviour beyond the two handoffs (plan 001), TOTP enrollment (003), invitation acceptance and its return route (008), workspace creation (012), personal-settings email change, the email provider itself (fake port; provider choice is A-10 / A-11).

### 5.4 Exit criteria
Specification §7 items 1–11 map to the "Proves criterion" column; item 11 is satisfied by §6 being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-27 statuses and name constraints | Working values `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING` (shared with plan 001): `UNVERIFIED` takes the replace branch, `ACTIVE` receives the notice, `RESTRICTED` and `ERASING` take the "no change" branch; `DisplayName` bounds come from configuration. | Change the status branch in `SignUpUseCase` and the configured bounds only. |
| A-28 invitation handoff (decided by specification 008 Q3) | A plain signup carries no invitation context. A signup opened from the invitation page sends the link secret with the form, and `SignUpUseCase` refuses any address other than the invited one through `InvitationAddressCheckPort` (plan 008 D11, T-20); the invitation is never consumed by signup. | Built by plan 008; this plan only exposes the hook in `SignUpUseCase`. |
| S-14 Privacy / Terms destinations | Routes open the operator-published documents outside the product with an external-link cue (017 Q3; `122:2` §15), rendered by the shared `PublicDocumentLinks` of plan 017 with the same validated addresses; no in-product page exists. | None. |
| S-17 "Need help?" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |
| S-23 notice content | Template `signup-attempt-notice` with the D04 copy (`437:4431`) offering sign-in and reset; send window configured. | None; keep the window. |
| S-24 return to invitation (decided by specification 008 Q3) | `confirmed` offers "Continue to sign in", plus "Return to your invitation" while the browser holds the invitation link (plan 008 T-39). | The two screens remain to be drawn (S-24); copy changes only. |
| S-25 re-address entry | Inline field under the pending state. | Swap for the designed dialog; the use case is unchanged. |
| S-26 already signed in | `/sign-up` shows the form; a success creates a separate pending identity and does not touch the session. | Redirect when a session exists. |
| S-27 pre-fill | "Continue to sign in" opens an empty form. | Pass the address in memory to the sign-in form. |
| A-33 link consumption | SPA-executed open (D4 as written); the recommended back-end open with a 303 redirect and a single-read outcome reference is the alternative. | Replace D4, the §2 endpoint table, the FE link opener and the RF-8 rows of §3 and §5. |
| A-34 re-addressing proof of control | Pending reference capability (D6). | If password re-entry is chosen, drop the reference and add a password field to the RF-7 entry. |
| A-35 persistence tooling | Persistence adapters and migrations written against the access API and migration tool named as placeholders. | Rename the adapters' base types and the migration format; no use case changes. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.7) | `SignupPage`, `EmailAddress`, `DisplayName`, field-path problems | D10, D16 | RF-1 rows |
| RF-2 (2.1–2.4) | `PasswordPolicyQuery`, `PasswordPolicyPort`, `PasswordVerifier` | D10 | RF-2 rows |
| RF-3 (3.1–3.7) | `SignUpUseCase`, `GlobalIdentity.replacePending`, `SecurityNotificationPort` | D2, D7, D12 | RF-3 rows |
| RF-4 (4.1–4.6) | `IssueVerificationLink`, `TokenGenerator`, outbox, `SignupPolicyProperties` | D1, D3, D13 | RF-4 rows |
| RF-5 (5.1–5.4) | `VerifyEmailPage` pending, pending holder, handoff from plan 001 | D6, D14 | RF-5 rows |
| RF-6 (6.1–6.4) | `ResendVerificationUseCase`, throttle dimensions | D1, D8 | RF-6 rows |
| RF-7 (7.1–7.6) | `ReaddressPendingIdentityUseCase`, `ReaddressEntry`, pending reference | D6, D8, D12 | RF-7 rows |
| RF-8 (8.1–8.9) | `OpenVerificationLinkUseCase`, link opener, `confirmed` / `unavailable` states | D4, D5 | RF-8 rows |
| RF-9 (9.1–9.4) | `TermsAcknowledgement`, checkbox, routes | D11 | RF-9 rows |
| RF-10 (10.1–10.4) | `ExpireUnverifiedIdentitiesJob`, `SignupPolicy.unverifiedWindow` | D9 | RF-10 row |
| RF-11 (11.1–11.4) | Absence of tenant / invitation / factor / admin paths | D2, D4 | RF-11 rows |
| RF-12 (12.1–12.3) | Audit adapter, payload shape, token handling | D5, D14 | RF-12 rows |
| RF-13 (13.1–13.2) | Reflow of both pages, `shared/ui` | D16 | RF-13 row |
| RNF-1 | equal-work branches | D1, D2, D5 | timing rows |
| RNF-2 | token generation and hashing | D3 | RF-4.1 row |
| RNF-3 | conditional update, `ALREADY_VERIFIED` | D4 | RNF-3 row |
| RNF-4 | `SignupPolicyProperties` | D11 | startup |
| RNF-5 | outbox and email port | D1 | outbox rows |
| RNF-6, RNF-11 | reflow, `shared/ui` | D16 | e2e + manual |
| RNF-7 | locale on request, templates | D15 | RNF-7 row |
| RNF-8 | holder, audit payload, logs | D14 | RF-12 rows |
| RNF-9 | `VerificationLink.isNewestAndLive`, single unavailable path | D5 | RNF-9 row |
| RNF-10 | no session endpoint, no tenant write, a reference that is not a session | D4, D6 | RF-5.4 row |
| LC-1 … LC-25 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25) |

Self-check: every RF criterion (64), every RNF (11) and every limit case (25) of `specSignupVerification.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
