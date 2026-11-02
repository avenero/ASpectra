# Plan 006 — Sessions, refresh and expiry

**Feature** Identity session lifetime, coordinated refresh, expiry and re-authentication · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/006-SessionsRefreshExpiry/specSessionsRefreshExpiry.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-08 as the subject; F-IDN-03, F-IDN-04, F-IDN-05, F-IDN-06, F-IDN-07, F-IDN-09, F-IDN-14, F-IDN-15, F-IDN-16, F-IDN-18, F-SET-03 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG; the `identity` context owns "sessions / refresh / recovery"), FE board `46:7847` (feature-modular SPA DAG; `shared/api`, `shared/realtime`, `shared/security`, `shared/errors`), board 3 `46:7847` ("One coordinated refresh, one eligible retry, no surprise Run") and the decision register `122:2` ("Session expiry, refresh, logout and accepted runs"; `SessionRegistry / RefreshFamily` among the proposed identity components) · **Builds on** plan 001 (Login), plan 002 (Signup and verification), plan 003 (Enrollment) and plan 004 (Password reset); elements they own are tagged **[001]** / **[002]** / **[003]** / **[004]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. Plans 001, 003 and 004 reach "the existing session service (F-IDN-08)" through three ports — `IssueSessionPort` **[001]**, `CurrentSessionPort` **[003]** and `RevokeSessionsPort` **[004]** — whose implementation they left to this feature; this plan builds that service and keeps those ports as its published surface, so nothing in the earlier plans changes. The specification's six decisions (Q1–Q6) and two derived rules (D-a, D-b) are built as decided; the register items it closed or raised (A-31, A-49, A-50, S-39…S-42) are therefore implemented, not assumed. Where the specification depends on items still open (A-11, A-27, A-30, A-35, A-39 port contract, A-45, S-02, S-07, S-14, S-42), the plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | No use case of this feature reads, stores or replays a mapping, approval, launch command or intent; a re-authentication issues a session and answers "resume", nothing else; the FE retry after refresh re-sends an existing request unchanged and never synthesises one. | RF-4.2, RF-8.5, RF-10.1, RF-10.2, RNF-10 |
| **P3** The browser expresses intent, never authority | A credential identifies a session; authority is derived by `access` on every request from the session's identity and current memberships. The preserved "return to" context is only the route the client was on: it re-requests, and the server derives afresh. The client may mark a request as *background* (lowering activity), never as *deliberate*. | RF-1.1, RF-2.2, RF-8.3, RNF-1 |
| **P4** Only the platform catalog is global | Sessions and families are identity-scoped platform rows; nothing per tenant is duplicated. | RF-1.1, RF-1.5 |
| **P6** Fail closed | An absent, ended, superseded, unknown or version-stale session or family refuses the request; an indeterminate refresh is reported as indeterminate and nothing is assumed; the forgery defence refuses without refreshing. | RF-2.6, RF-3.3, RF-5.5, RNF-6, RNF-7 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security JWT / refresh support, `PasswordEncoder`, the outbox and durable jobs. FE: React 18, TypeScript, Vite, TanStack Query, Zustand (scoped drafts only), i18next, Tailwind, axios, STOMP. No cache product, no new token library, no state-sync library. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the session service lives in the existing hexagonal bounded context `identity`; `access` consumes one published port to start C10 derivation; outer adapters (security filter, cookie, controller, persistence) depend inward; mail through the outbox; sweeps through durable jobs. FE: cross-cutting behaviour in `shared/api`, `shared/security`, `shared/realtime`, `shared/errors`; the dialog and re-authentication UI in the `identity-access` feature; TanStack Query owns server facts; Zustand holds only the deliberate scoped drafts the boundary protects. | §2 |
| **P12** Two independent tenancy gates | Session, family and credential rows are **platform** rows of a global identity with no tenant column; RLS does not apply to them. Every tenant request still derives C10 through `access` under RLS and runtime authorisation; a valid session never shortens that path. | RF-1.1, RF-1.5, LC-17 |
| **P13** Configured values are never defaulted | Idle deadline, absolute deadline, access-credential lifetime, save-notice threshold, safe-read retry bounds, ended-session and family retention, sweep cadence and batch size are one validated `SessionPolicyProperties` record; the client receives the values it needs from the server and holds no number of its own; startup fails on any absent or unbounded value. | RF-2.1, RF-9.1, RF-4.6, RNF-5 |
| **P14** Release bar | WCAG 2.2 AA on the dialog, the panel states, the notice and the retry status with automated and manual assistive evidence; EN / ES / FR for every state and for the reuse notification. | RF-12, RNF-12–14 |
| **F-X-01** C10 context | `access` derives C10 for every protected request from `SessionAssertionPort` (identity, session, version) plus memberships; no request is served on a credential alone; async jobs of this feature carry no customer context. | RF-1.1, RF-2.6 |
| **F-X-02** Request integrity | The FE retries an eligible operation at most once with the same axios request object (URL, body, headers, idempotency key, scope epoch); consequential writes are reconciled by operation-status lookup; no replay queue exists; refresh itself is serialised by a row lock so an ambiguous duplicate is reuse, never a second rotation. | RF-3.4, RF-4.2, RF-4.4, RNF-9 |
| **F-X-03** Secrets and safe problems | Refresh credentials are stored only as hashes; access credentials live in memory; problems carry safe codes (`auth.credential.expired`, `auth.session.invalid`, `auth.csrf.invalid`, `access.denied-scope`) and a correlation id; nothing of this feature touches browser storage. | RF-1.3, RF-11.2, RF-11.3, RNF-8 |
| **F-X-04** Accessibility | Dialog focus management, live region without focus theft, 44 px targets, text status. | RF-12.2 |
| **F-X-05** Audit log | Every event of RF-11.1 appended through `AuditPort`, redacted, in the same transaction as the effect. | RF-11.1 |
| **F-X-06** Configuration validation and email port | `SessionPolicyProperties` is part of startup validation; the reuse notification goes through `SecurityNotificationPort` **[002]** behind the outbox. | RF-11.4, RNF-5, RNF-9 |
| **F-X-07** Responsive surfaces | M23 (`432:4427`) is the drawn counterpart of R34; the new states follow the D01 responsive policy (S-02) with the statements shared across widths. | RF-12.1, RNF-14 |
| **F-IDN-09** boundary | Logout, listing and individual revocation (plan 007) use `EndSessionUseCase` and `SessionQuery` of this plan; they add no second session model. | RF-2.5, RF-2.6, RF-7.5 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity`, plus one consumer in `access`

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 001–004. This plan adds two aggregates, one policy, one classification rule, six use cases, one query, one durable job, one published port, and implements the three ports earlier plans declared. Contexts touched — `access` (C10 derivation and the denied-scope problem), `audit`, `sharedsupport`, `execution` (read-only: nothing; the run-scoped authority of accepted Runs is simply never touched) — are reached only through published ports.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `Session` | Aggregate of one browser session: id, identity id, `createdAt`, `lastActivityAt`, `idleDeadline`, `absoluteDeadline`, `securityVersionAtIssue`, `clientContext` (user-agent family, network key — the "context" of the 007 list), state `ACTIVE → ENDED(cause)` with causes `IDLE`, `ABSOLUTE`, `VERSION`, `REVOKED`, `LOGOUT`, `REUSE`, `IDENTITY_STATUS`. Methods: `isValid(now, currentVersion)`, `touch(now, policy)` (moves `idleDeadline` only; never the absolute one), `end(cause, now)`, `remaining(now)`. Pure; takes the clock as an argument. | RF-1.1, RF-1.5, RF-2.1–2.7, RF-3.2, LC-4, LC-21 |
| | `RefreshFamily` | One chain of refresh credentials per session: family id, session id, `generation`, the hash of the current credential, the hashes of retired generations with their retirement time, state `LIVE → ENDED(cause)`. `rotate(presentedHash, now)` returns `Rotated(nextGeneration)` when the hash is the current one, `Reuse` when it is a retired one, `Unknown` otherwise; `end(cause, now)`. The plaintext credential never enters the aggregate. | RF-3.1, RF-3.3–3.5, RNF-2, RNF-3, LC-1–LC-3 |
| | `SessionPolicy` | Immutable record: idle deadline, absolute deadline, access-credential lifetime, save-notice threshold, safe-read retry bounds (attempts, backoff ceiling), ended-session retention, family retention, sweep cadence and batch size. | RF-2.1, RF-4.6, RF-9.1, RNF-5 |
| | `ActivityClass` | Pure rule `(operationClass, backgroundMarker) → DELIBERATE \| BACKGROUND`: a consequential write or an explicit command is always `DELIBERATE`; a read is `DELIBERATE` unless the transport marked it background; the refresh operation, the session-status query and realtime traffic are structurally `BACKGROUND`. The marker can only lower a request to background, never raise it (D3). | RF-2.2, RF-2.3, RF-3.2, LC-21, LC-28 |
| | `LoginTransaction` **[001]** | Gains purpose `REAUTHENTICATION` and the id of the ended session it resumes; every other rule (identity binding, expiry, attempts, `securityVersion` check, idempotency outcomes, forced-change state **[004]**) is unchanged. | RF-8.1, RF-8.4, RF-8.6, LC-14, LC-16 |
| `access.domain` | C10 derivation **[001/003]** | Starts from the session facts of `SessionAssertionPort` (identity id, session id, version); unchanged otherwise. A request whose session is invalid never reaches derivation. A derivation that yields no authority for the requested tenant or project answers `access.denied-scope`. | RF-1.1, RF-5.1, RF-8.4, LC-8, LC-17 |
| `identity.application` | `IssueSessionUseCase` (implements `IssueSessionPort` **[001]**) | On a completed sign-in or re-authentication: creates `Session` with deadlines from `SessionPolicy` and the identity's current `securityVersion`, creates its `RefreshFamily` (generation 1), mints the access credential through `AccessCredentialCodec`, audits `SESSION_ESTABLISHED{cause: SIGN_IN \| REAUTH}`, returns `{accessCredential, refreshCredentialPlain, remaining}` once. | RF-1.1, RF-1.2, RF-8.3, RF-11.1 |
| | `AssertSessionUseCase` (implements `SessionAssertionPort`, published) | Per request: verifies the access credential (signature, lifetime) through the codec; an expired credential → `CREDENTIAL_EXPIRED` (eligible for refresh); otherwise loads the `Session` by id, checks state, deadlines, `securityVersionAtIssue` against the identity's current version and the identity status; any failure → `end(cause)` if not already ended and `SESSION_INVALID` (definitive); on success classifies the request (`ActivityClass`), touches the session when `DELIBERATE`, and returns the session facts plus `remaining`. | RF-1.1, RF-2.2–2.7, RF-3.2, RF-8.8, RNF-11, LC-18–LC-21, LC-27, LC-30 |
| | `RefreshSessionUseCase` | Locks the family row (`FOR UPDATE`); checks the session (`isValid`) and identity status; `rotate(presentedHash)`: `Rotated` → store the new hash, mint a new access credential, answer `{accessCredential, refreshCredentialPlain, remaining}`; `Reuse` → end family and session with `REUSE`, audit `REFRESH_REUSE_DETECTED`, enqueue `session-credential-reuse`, answer `SESSION_INVALID`; `Unknown` or invalid session → `SESSION_INVALID`. Never touches deadlines. The lock makes a concurrent duplicate wait and then see a retired hash — it is answered as reuse, which is the specification's rule (RF-3.5), while two tabs never reach here twice because the client coalesces (D5). | RF-3.1–3.6, RF-11.1, RF-11.4, RNF-2, RNF-3, LC-1–LC-3 |
| | `EndSessionUseCase` (implements `RevokeSessionsPort` **[004]** and serves plan 007) | `end(sessionId, cause)` and `revokeAll(identityId, exceptSessionId, cause)`: ends sessions and families, audits `SESSION_ENDED{cause, source}`. Version bumps (**[004]** `bumpSecurityVersion`) need no call: `AssertSessionUseCase` ends version-stale sessions lazily at their next use (RF-2.5 "at its next use"); `revokeAll` is the eager path for reset, replacement, compromise response, sign-out-everywhere and identity status changes. | RF-2.5, RF-2.6, RF-7.5, LC-18–LC-20, LC-27, LC-30 |
| | `CurrentSessionPort` **[003]** implementation | `currentSessionId()` from the request's session facts; `exists(sessionId)` = `Session.state == ACTIVE`. | RF-1.5 |
| | `StartReauthenticationUseCase` | Input: the ended session's id (sent by the dialog, opaque). Loads the ended session within its retention; resolves **its** identity; creates a `LoginTransaction(REAUTHENTICATION)` bound to that identity and session; the password and factor are then verified by `SignInUseCase` / the challenge use cases **[001]** under Login's throttles and forced-change state **[004]**; the identity's address is returned read-only for display. An unknown or out-of-retention session id → the generic sign-in failure. | RF-8.1, RF-8.2, RF-8.6, RF-8.8, LC-13–LC-16, LC-27 |
| | `CompleteSignInUseCase` **[001]** | For purpose `REAUTHENTICATION`: issues the session (`IssueSessionUseCase`, cause `REAUTH`), audits `REAUTHENTICATION_SUCCEEDED`, answers `{state: COMPLETED, resume: true}` and **does not** run `LandingRule` **[001]**; failures and voids audit `REAUTHENTICATION_FAILED \| VOIDED`. | RF-8.3, RF-8.5, RF-11.1 |
| | `SessionStatusQuery` | For the caller's session: `remaining`, both deadlines, and the policy values the client needs (save-notice threshold, safe-read retry bounds); structurally `BACKGROUND`. | RF-2.7, RF-4.6, RF-9.1 |
| | `ExpireSessionsJob` | Durable job (`sharedsupport/jobs` **[002]**): claims `ACTIVE` sessions past a deadline in bounded batches, ends them (`IDLE` / `ABSOLUTE`) with audit, so the 007 list and the audit log are accurate; prunes ended sessions and families past their retention. Check-on-access enforces expiry first; the job only cleans up. | RF-2.6, RF-11.1 |
| | Ports | New: `SessionRepository`, `RefreshFamilyRepository`, `AccessCredentialCodec` (→ existing Spring Security JWT support), `RefreshCredentialGenerator` / `RefreshCredentialHasher` (→ `TokenGenerator`, `TokenHasher` **[002]**), `SessionAssertionPort` (published to `access`), `ActivityMarkerPort` (reads the transport's background marker). Implemented here: `IssueSessionPort` **[001]**, `CurrentSessionPort` **[003]**, `RevokeSessionsPort` **[004]**. Reused: `IdentityRepository`, `LoginTransactionRepository`, `ThrottleStore`, `PasswordVerifier`, `TotpVerifier`, `Clock`, `AuditPort` **[001]**; `JobScheduler`, `SecurityNotificationPort` (behind the outbox) **[002]**. No port reaches a project, grant, approval, run or intent table. | all |
| `identity.infrastructure` | Persistence | `identity_session` (id, identity_id, created_at, last_activity_at, idle_deadline, absolute_deadline, security_version, client_context, state, end_cause, ended_at; index on `identity_id, state`), `refresh_family` (id, session_id unique, generation, state, end_cause, ended_at), `refresh_credential` (family_id, generation, hash unique, issued_at, retired_at; the current generation is the row with `retired_at IS NULL`). Access API and migration tool per A-35 (placeholders). | RF-1.5, RF-3.1, RF-3.5, RNF-2 |
| | Security filter | `SessionAuthenticationFilter` in the existing Spring Security chain: extracts the in-memory access credential from the authorisation header, calls `AssertSessionUseCase`, publishes the session facts to the security context, adds the `X-ASpectra-Session-Remaining` response header, maps `CREDENTIAL_EXPIRED` → 401 `auth.credential.expired` and `SESSION_INVALID` → 401 `auth.session.invalid`. The sign-in, challenge, re-authentication-start and refresh endpoints are outside the filter (RF-4.3). | RF-2.6, RF-2.7, RF-4.1, RF-4.3, RF-11.3 |
| | Cookie and origin configuration | The refresh credential travels only as a cookie: `Secure`, `HttpOnly`, `SameSite` per the configured origins, path restricted to the refresh and re-authentication endpoints, lifetime = absolute deadline. CSRF through Spring Security's configured SPA pattern on every state-changing endpoint, including refresh; CORS allow-list from configuration; a CSRF failure → 403 `auth.csrf.invalid` with no session change. | RF-1.2, RF-1.4, RF-5.5, RNF-8, LC-9 |
| | Credential adapters | `AccessCredentialCodec` over the existing JWT support: claims limited to session id, identity id, version, issue and expiry; signed with the existing key management; no tenant, role or personal data. `RefreshCredentialGenerator` = 256-bit CSPRNG; hasher = cryptographic hash (**[002]** `TokenHasher`). | RF-1.2, RF-11.2, RNF-8 |
| | Outbox and email adapter **[002]** | Template `session-credential-reuse` in EN / ES / FR: a session was ended because its credential was presented twice, when, what to do (change the password, review the sessions list); no credential, no link that grants access. | RF-11.4, RNF-9, LC-29 |
| | Audit adapter | Events: `SESSION_ESTABLISHED{cause}`, `SESSION_ENDED{cause, source}`, `REFRESH_REUSE_DETECTED`, `REAUTHENTICATION_SUCCEEDED`, `REAUTHENTICATION_FAILED`, `REAUTHENTICATION_VOIDED{reason}`, `NOTIFICATION_ENQUEUED{template}`. Payload: identity id, session id, correlation id, network key, timestamp. Never a credential, a hash, a password or a code. | RF-11.1, RF-11.2 |
| | `SessionPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding into `SessionPolicy`; every value present, positive, bounded by a configured ceiling; the idle deadline no greater than the absolute one; the access-credential lifetime shorter than the idle deadline. Any violation fails startup. | RF-2.1, RNF-5 |
| `identity.api` | `SessionController` — `/api/auth/session` (outside the session filter; cookie + CSRF protected) | `POST refresh` (cookie) → 200 `{accessCredential, expiresIn, remaining}` with the rotated cookie, or 401 `auth.session.invalid` (definitive) · `GET status` (session filter) → 200 `{remaining, idleDeadline, absoluteDeadline, saveNoticeThreshold, safeReadRetry}` · `POST reauthentication` {endedSession} → 200 `{state: LIMITED, transaction, address}` or 401 `auth.sign-in.failed` **[001]**; the password step and `challenge/*` steps are plan 001's endpoints with the transaction id, and completion answers `{state: COMPLETED, resume: true, accessCredential, expiresIn, remaining}` with the new cookie. Logout (`DELETE`) and the list are plan 007's. All problems are RFC 9457 with a safe code and a correlation id. | RF-2.7, RF-3, RF-4.3, RF-8, RF-11.3 |

**Cross-context touch points.** `access` — consumes `SessionAssertionPort` at the start of C10 derivation and emits `access.denied-scope`. `audit` — append-only. `sharedsupport` — `crypto` (CSPRNG, hashing, JWT key references), `jobs`, `tx`, `contextIds`, `outbox`. `execution` — nothing: run-scoped service authority is issued at admission (F-RUN) and never references a browser session, which is how RF-10 holds by construction. Nothing reads or writes project, grant, approval or run tables.

**Data model note (P12).** The three tables of this feature are platform-scoped rows of a global identity and carry no tenant column; RLS does not apply to them. Memberships are read by `access` only, under RLS, after the session is asserted.

### 2.2 Front end — `shared/*` modules plus the feature module `src/features/identity-access`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| `shared/security` | `AccessCredentialHolder` | The access credential and its expiry, in memory only; cleared on session end and on logout. | RF-1.2, RF-1.3, RNF-8 |
| | `SessionState` store | `{status: VALID \| ENDED \| UNKNOWN, remaining, saveNoticeThreshold, safeReadRetry, endedSessionId}` fed by the `X-ASpectra-Session-Remaining` header and `GET status`; never a local countdown — `remaining` is displayed, not decremented. | RF-2.7, RF-9.1, RNF-4 |
| | `RefreshCoordinator` | Single-flight: the first eligible failure starts `POST refresh`; every other eligible request awaits the same promise; outcome `REFRESHED \| INVALID \| INDETERMINATE`; the promise is cleared after settlement. | RF-4.1, RF-4.5, RF-7.8, LC-1, LC-7 |
| | `ProtectedBoundary` | Wraps every protected route: on `ENDED` masks (unmounts) restricted subtrees, calls the transient-secret registry to clear password and code fields, pauses the query client, disconnects realtime, and renders R34; on resume compares `{identityId, tenantId, projectId}` of the preserved context with the new session and the refreshed scope, then unmasks and refetches or discards drafts (D10). | RF-7.1, RF-7.6, RF-9.2, RF-9.3, LC-12, LC-17, LC-22 |
| | Scope epoch **[001]** | Captured on every request; a retry whose epoch differs from the current one is dropped (never retargeted). | RF-4.2, RF-9.6 |
| `shared/api` | axios transport **[001]** | Attaches the access credential and correlation id; sets the background marker on requests TanStack Query issues from interval refetch, focus refetch or reconnect refetch (query `meta.background`), and on `GET status`; response interceptor: `auth.credential.expired` → `RefreshCoordinator` then one retry of the **same** request object (same URL, body, headers, idempotency key) unless its epoch changed; `auth.session.invalid` → `SessionState.ENDED`; `access.denied-scope` → `DeniedScopeHandler`; `auth.csrf.invalid` → `CsrfFailureHandler`; network error / timeout → `TransportFailureStatus`. Safe reads retry within the server-supplied bounds; mutations never auto-retry. | RF-2.3, RF-4.1–4.6, RF-5.4, RF-5.5, RNF-9, LC-6, LC-9 |
| `shared/realtime` | STOMP client **[006]** | Connects with the current access credential; heartbeats carry no activity; on `ENDED` disconnects; reconnects after resume with the new credential; connection loss is reported to the monitor as stream loss, never as session end. | RF-2.3, RF-6.3, RF-7.1, LC-11, LC-22 |
| `shared/errors` | `DeniedScopeHandler` | On `access.denied-scope`: one permission refresh (`permissionVersion` and grants), then resume or render N28 (`ProjectAccessDeniedPanel`) with "Refresh permissions" and "Choose permitted project"; removes restricted data and subscriptions; never refreshes credentials or routes to sign-in. | RF-5.1–5.3, LC-8 |
| | `CsrfFailureHandler` | Safe problem "request could not be validated — reload and try again"; nothing else. | RF-5.5, LC-9 |
| | `TransportFailureStatus` | Accessible retry / offline status; when the indeterminate outcome was the refresh, offers "Retry session refresh" (one coordinated refresh per activation). | RF-5.4, RF-5.6, LC-7 |
| `identity-access` | `SessionExpiryDialog` (R34) | The designed copy; primary "Reauthenticate" → `ReauthenticationStep`; secondary "Retry session refresh" → one coordinated refresh, dialog unchanged on `INVALID`; footer routes: "local account recovery" → spec 005 public entry with context discarded; "sign out safely" → plan 007 logout with context discarded. Modal; no close. | RF-7.2–7.7, LC-13 |
| | `ReauthenticationStep` | Inside the dialog: address read-only, `PasswordField` **[001]** with Show, then `MfaChallenge` **[001]** when the server answers `LIMITED` with a challenge; `GenericFailureMessage` **[001]**; "verification in progress"; a voided transaction returns the dialog to its initial state (D-a). Reuses the sign-in and challenge hooks **[001]** with purpose `REAUTHENTICATION`. | RF-8.1, RF-8.2, RF-8.6, LC-14–LC-16 |
| | `useSessionStatus()`, `useReauthenticate()` | Typed hooks; `useReauthenticate` posts `{endedSession}` then delegates to the sign-in / challenge mutations **[001]**; on `COMPLETED, resume: true` stores the credential and signals `ProtectedBoundary.resume()`. | RF-8.3, RF-8.7 |
| | `SaveNowNotice` | Non-blocking banner shown while `status == VALID`, `remaining < saveNoticeThreshold` and the dirty-draft registry is non-empty; "Save now" invokes the screen's own save under the original scope; hidden when nothing is dirty. | RF-9.1, RF-9.4, LC-4, LC-28 |
| | `SecurityStatements` **[001]** | Extended with the R34 statements and the "Accepted runs continue" block so desktop and reflow cannot drift. | RF-7.2, RF-10.4, RF-12.1 |
| Drafts | Zustand scoped drafts **[board 3]** | Each draft carries `{identityId, tenantId, projectId, baseETag, editSeq}`; `ProtectedBoundary` keeps them in memory during the mask and purges any whose stamp differs from the resumed session and scope or whose edit the refreshed grants no longer permit; never persisted. | RF-9.2, RF-9.3, RF-9.5, LC-5, LC-17 |
| Run monitor | `execution` feature **[F-RUN]** | Renders C07 facts only; a target's "unauthorised" answer is a fact row; stream loss renders the D02 "Stream interrupted · facts stale" pattern; the console has no HTTP client for targets, so the wall is structural (D12). | RF-6.1–6.3, RF-10.2, LC-10, LC-11 |
| Shared used | `shared/i18n` (bundle keys for every statement, state and message), `shared/ui` (`Dialog` with focus trap, `LiveRegion`, `Banner`, 44 px targets). | RF-12.1–12.3 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `IssueSessionUseCase`; `access` derives from `SessionAssertionPort` | — |
| RF-1.2 | Codec (memory credential) + cookie adapter (refresh credential) | `AccessCredentialHolder` |
| RF-1.3 | — | No storage write anywhere in `shared/security`, drafts in memory only |
| RF-1.4 | CSRF and CORS configuration; `auth.csrf.invalid` | `CsrfFailureHandler` |
| RF-1.5 | `Session` aggregate and `identity_session` | — |
| RF-1.6 | No port of this feature is consulted by 003 / 004 / 005 proofs; `SessionAssertionPort` grants nothing | — |
| RF-2.1 | `SessionPolicyProperties` | — |
| RF-2.2 | `ActivityClass.DELIBERATE` → `Session.touch` | Requests from user actions carry no marker |
| RF-2.3 | `ActivityClass.BACKGROUND`; refresh, status and realtime structurally background | Marker on interval / focus / reconnect refetches; STOMP heartbeats are not HTTP |
| RF-2.4 | `Session.touch` never moves `absoluteDeadline`; `isValid` | — |
| RF-2.5 | `securityVersionAtIssue` check in `AssertSessionUseCase`; `EndSessionUseCase.revokeAll(except)` | — |
| RF-2.6 | `AssertSessionUseCase` → `SESSION_INVALID` after `end` | `auth.session.invalid` → `ENDED` |
| RF-2.7 | `X-ASpectra-Session-Remaining`; `GET status` | `SessionState.remaining` displayed, never decremented |
| RF-3.1 | `RefreshFamily.rotate` under `FOR UPDATE`; `refresh_credential` rows | — |
| RF-3.2 | `RefreshSessionUseCase` never calls `touch`; mints under current authority only | — |
| RF-3.3 | `isValid`, identity status, `Unknown` → `SESSION_INVALID` | — |
| RF-3.4 | Row lock serialises; the client coalesces (D5) | `RefreshCoordinator` |
| RF-3.5 | `Reuse` → end family + session, audit, template | Every tab receives `auth.session.invalid` next |
| RF-3.6 | Two answers only: 200 or `auth.session.invalid` | `INDETERMINATE` only when no answer arrived |
| RF-4.1 | `auth.credential.expired` is the only eligible code | Interceptor + `RefreshCoordinator` |
| RF-4.2 | — | Retry of the same request object; epoch check |
| RF-4.3 | Sign-in, challenge, reauthentication, refresh outside the filter | Interceptor ignores those endpoints and non-401 answers; no target client exists |
| RF-4.4 | Idempotency outcomes **[001 D1]** on every consequential endpoint | Same key and body on the retry; status lookup on lost acknowledgement |
| RF-4.5 | — | `INVALID` → `ENDED`; `INDETERMINATE` → `TransportFailureStatus` |
| RF-4.6 | `safeReadRetry` in `GET status` | Query retry bounds from the server |
| RF-5.1 | `access.denied-scope` from C10 derivation | `DeniedScopeHandler` (one permission refresh) |
| RF-5.2 | — | N28 copy and actions |
| RF-5.3 | — | Draft kept in memory; shown only if refreshed grants permit |
| RF-5.4 | — | `TransportFailureStatus` with "Retry session refresh" |
| RF-5.5 | `auth.csrf.invalid`, no session change | `CsrfFailureHandler` |
| RF-5.6 | Operation-status lookup endpoints **[001 D1]** | Lookup before re-offering a consequential submit |
| RF-6.1 | Runner reports target responses as C07 facts | Monitor renders facts; never the interceptor |
| RF-6.2 | Runner holds no ASpectra credential; target addresses from the environment snapshot only | No target HTTP client in the console |
| RF-6.3 | — | `shared/realtime` → monitor "Stream interrupted" |
| RF-7.1 | — | `ProtectedBoundary`: mask, secrets cleared, queries paused, STOMP disconnected |
| RF-7.2 | — | `SessionExpiryDialog` copy from the i18n bundle |
| RF-7.3 | `auth.session.invalid` carries no cause | Dialog shows none |
| RF-7.4 | — | Primary / secondary actions; one refresh per activation |
| RF-7.5 | Plan 007 logout; spec 005 entry | Context and drafts discarded before routing |
| RF-7.6 | — | Modal, no close, query client paused |
| RF-7.7 | — | D02 compact variant of the dialog component |
| RF-7.8 | Cookie is per browser context; new family on resume | Other tabs: `auth.credential.expired` → refresh with the new cookie succeeds |
| RF-8.1 | `StartReauthenticationUseCase` + `SignInUseCase` / challenge **[001]** | `ReauthenticationStep` |
| RF-8.2 | Identity resolved from the ended session; address returned read-only | Read-only address; "sign out safely" for another identity |
| RF-8.3 | `CompleteSignInUseCase(REAUTHENTICATION)` → `IssueSessionUseCase`; no `LandingRule` | `resume: true` → unmask and refetch |
| RF-8.4 | `FORCED_CHANGE_REQUIRED` **[004]** inside the transaction; `access` gate on re-entry | Denied-scope or chooser on changed authority |
| RF-8.5 | Completion touches identity tables only | No request replayed on resume |
| RF-8.6 | Transaction void rules **[001]** | Dialog back to initial state (D-a) |
| RF-8.7 | — | First request → `ENDED` over the shell; resume re-requests the route (D-b) |
| RF-8.8 | Identity status check in `SignInUseCase` **[001]** | Generic failure |
| RF-9.1 | `saveNoticeThreshold` from policy | `SaveNowNotice` |
| RF-9.2 | — | Drafts in memory under the mask |
| RF-9.3 | — | `ProtectedBoundary.resume` stamp comparison and grant check |
| RF-9.4 | — | No save call from boundary, dialog or resume |
| RF-9.5 | Idempotency outcomes | Lookup after resume |
| RF-9.6 | — | Dirty guard of 016 unchanged; original scope on save |
| RF-10.1 | Run-scoped authority never references a session | Nothing on resume touches runs |
| RF-10.2 | Launch status lookup **[F-RUN]** | Lookup, never resubmit |
| RF-10.3 | No call from session end to any run policy | — |
| RF-10.4 | — | Dialog block; R38 statement (plan 007) |
| RF-11.1 | Audit adapter | — |
| RF-11.2 | Payload shape; codec claims; hashed credentials | Nothing logged client-side |
| RF-11.3 | Problem catalogue | Mapped per code |
| RF-11.4 | `session-credential-reuse` via outbox | — |
| RF-11.5 | Row locks; one transaction per rotation | — |
| RF-12.1 | — | Reflow; `SecurityStatements` shared |
| RF-12.2 | — | `Dialog` focus trap, `LiveRegion` polite, 44 px |
| RF-12.3 | Template language from the locale header | i18n bundles |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · One `Session` aggregate per browser context, held on the server, checked on every request.**
*Why.* RF-2.6 and RNF-11 require that an ended session is refused from its next request on, whatever the lifetime of the access credential; only a server-side record consulted per request gives that. The record also carries what the 007 list needs and what the audit log records, so there is one session model. The per-request cost is one primary-key read inside the C10 derivation that `access` already performs.
*Discarded.* A purely stateless access credential trusted until it expires — an ended session would keep working for the credential's lifetime, contradicting RNF-11. A distributed cache of ended sessions — a new product outside P10 and a second source of truth.
*Covers.* RF-1.1, RF-1.5, RF-2.5, RF-2.6, RNF-11, LC-18–LC-20, LC-30.

**D2 · Two credentials: a short-lived signed access credential in memory, and an opaque rotating refresh credential in a restricted cookie, stored only as a hash.**
*Why.* FR-IDN-060 and RF-1.2: the access credential must be usable by the SPA without being stored, the refresh credential must be unreadable by scripts and unsendable cross-site. A signed access credential lets the filter reject a tampered or expired one before any read; an opaque refresh credential has nothing to forge and, hashed at rest, nothing to steal from the database. The cookie path is limited to the refresh and re-authentication endpoints so it never accompanies ordinary requests.
*Discarded.* Both credentials in cookies — the access credential would be sent on every request and need CSRF on every read. A refresh credential readable by the client — storable, exfiltratable. A refresh credential that is itself a signed token — replayable within its lifetime and not revocable without a store, which exists anyway.
*Covers.* RF-1.2, RF-1.3, RF-1.4, RF-11.2, RNF-8.

**D3 · Activity is classified on the server by operation class, with a transport-level marker that can only lower a request to background.**
*Why.* Decision Q5 (A-49) says what counts; the mechanism must keep P3: the server decides. Consequential writes and commands are deliberate by nature. Reads are ambiguous — a list fetched by navigation and the same list refetched on an interval are one endpoint — so the client's transport, not any feature, marks the refetches TanStack Query issues on interval, focus or reconnect. A client that omits the marker gains nothing it could not gain by actually using the product; a client cannot mark a request *deliberate*. The refresh, the status query and realtime traffic are background by construction.
*Discarded.* A client "activity ping" — the client would extend sessions without server-visible work (Q5's rejected option). A server allow-list of endpoints — cannot tell navigation from refetch. Counting every request — polling tabs would never expire (FR-IDN-061).
*Covers.* RF-2.2, RF-2.3, RF-3.2, LC-21, LC-28.

**D4 · Deadlines, remaining time and policy values come from the server on every answer; the client shows, never computes.**
*Why.* R34's statement and RNF-4: a local countdown drifts and can be manipulated. A response header on every authenticated answer costs nothing and keeps `SessionState` current without polling; `GET status` supplies the save-notice threshold and safe-read retry bounds so the client holds no number of its own (P13).
*Discarded.* A client timer seeded at sign-in — a local countdown. A polling status call — would itself be background traffic and still lag behind the real deadline.
*Covers.* RF-2.7, RF-4.6, RF-9.1, RNF-4, RNF-5.

**D5 · Atomic family rotation under a row lock; reuse ends the whole family; the client coalesces so legitimate concurrency never reaches the lock twice.**
*Why.* RF-3.1 and RF-3.4: the rotation must be indivisible and the server must not "accept an arbitrary recently consumed token" to tolerate concurrency. A `FOR UPDATE` lock serialises concurrent presentations: the first rotates, the second then sees a retired hash and is reuse — which is exactly the rule Q6 decided, and it is safe because the client's single-flight (`RefreshCoordinator`) means two tabs of one browser context present the cookie once. Ending the whole family on reuse (A-50) protects the legitimate holder from an attacker who raced first.
*Discarded.* A grace window in which the previous credential is still accepted — the "arbitrary recently consumed token" strategy FR-IDN-062 forbids. Ending only the presented credential — leaves an attacker's live branch. Optimistic rotation without a lock — two rotations from one credential.
*Covers.* RF-3.1, RF-3.3–3.5, RF-11.4, RNF-2, RNF-3, LC-1–LC-3.

**D6 · Two distinct 401 problem codes: `auth.credential.expired` (eligible for refresh) and `auth.session.invalid` (definitive).**
*Why.* RF-4.1 and RF-4.5 need the client to know whether a refresh can help; one code would force a refresh attempt on every ended session and blur "definitive". With two codes the interceptor refreshes only when the access credential is stale and goes straight to the dialog when the server has already said the session is gone; the refresh endpoint answers the same two ways.
*Discarded.* One generic 401 — every expiry costs a refresh round-trip and transport failures during it blur the answer. Carrying the cause in the problem — RF-7.3 forbids disclosing it; the two codes say only "refreshable" or "not".
*Covers.* RF-2.6, RF-3.6, RF-4.1, RF-4.5, RF-7.3, RF-11.3, RNF-7.

**D7 · Single-flight refresh and one retry of the same request object, dropped if the scope epoch moved.**
*Why.* RF-4.1 and RF-4.2 verbatim; retrying the identical axios request object keeps URL, body, headers and idempotency key frozen (FR-X-021); the epoch captured at request time makes a tenant switch during the refresh drop the retry rather than retarget it (FR-IDN-131). Mutations never auto-retry otherwise; safe reads retry within the server-supplied bounds.
*Discarded.* Rebuilding the request from current state — retargeting. A queue of failed mutations replayed after refresh — FR-X-023 forbids it. Per-request refresh — reuse detection on the client's own tabs.
*Covers.* RF-4.1–4.6, RF-9.6, RNF-9, LC-1, LC-6.

**D8 · Re-authentication reuses the login transaction with a new purpose, bound to the ended session's identity; completion issues a session and skips landing.**
*Why.* Decision Q2 (A-31): the proof is Login's proof, so Login's transaction, throttles, forced-change state and challenge are reused rather than copied; binding the transaction to the identity of the ended session (resolved server-side from the session id, kept within retention) makes "same identity only" a server rule, not a read-only field. Completion answers `resume: true` and does not run `LandingRule`; the client's preserved route is only a requested scope that the server derives authority for afresh (P3), so no server-side "return to" record is needed — plan 001's A-31 note considered one, and it would add state for nothing.
*Discarded.* A password-only re-authentication — Q2's rejected option; contradicts "no session grants a TOTP bypass". Redirecting to the sign-in screen — drops the preserved context R34 promises. A server-side resume record — duplicates what per-request derivation already guarantees.
*Covers.* RF-8.1–8.8, RF-1.6, LC-12–LC-17, LC-27.

**D9 · The version bump ends sessions lazily; `revokeAll(except)` is the eager path.**
*Why.* RF-2.5 says a session issued under an older version ends "at its next use", which `AssertSessionUseCase` enforces without any cross-feature call; reset, replacement, compromise response and sign-out-everywhere already call `RevokeSessionsPort` **[004]** so their sessions end eagerly and appear ended in the 007 list and the audit log at once. Both paths lead to the same `end(cause)`.
*Discarded.* Eager-only — every feature that bumps the version would have to enumerate sessions. Lazy-only — the sessions list would show live sessions that are dead.
*Covers.* RF-2.5, RF-2.6, LC-18–LC-20, LC-27.

**D10 · One protected boundary owns mask, secret clearing, draft retention and resume; drafts are stamped with identity and scope.**
*Why.* RF-7.1, RF-9.2 and RF-9.3 describe one transition with several effects; one component applied to every protected route cannot be forgotten by a feature. Zustand drafts already carry tenant, project, `baseETag` and `editSeq` (board 3); adding the identity id lets resume compare the stamp with the new session and the refreshed scope, and purge anything that does not match — the "different login must not inherit it" rule becomes a comparison, not a convention. Transient secrets (password and code fields) register with the boundary so they are cleared without a global search.
*Discarded.* Per-feature handling — drift. Persisting the draft to storage "just until resume" — FR-X-034. Keeping drafts unstamped and trusting the route — a different identity resuming the same route would see them.
*Covers.* RF-7.1, RF-7.6, RF-9.2–9.5, RF-5.3, RNF-8, LC-5, LC-12, LC-17, LC-22.

**D11 · The pre-expiry notice is driven by the server's remaining time and the dirty-draft registry, and it never extends the session.**
*Why.* Decision Q4 (S-40): the notice must come while authorisation holds and must not become a "keep me signed in" button. Reading `remaining` from the latest answer (D4) and the dirty registry from the drafts (D10) gives the condition without any timer; "Save now" invokes the screen's own save, which is a deliberate write and therefore counts as activity on its own merits (RF-2.2), not because of the notice.
*Discarded.* A countdown dialog with "Extend session" — a session extension without work on content (Q3's rejected option). A notice on every screen regardless of drafts — noise with no action.
*Covers.* RF-9.1, RF-9.4, LC-4, LC-28.

**D12 · The customer-target wall is structural: the console has no target client and the runner has no console credential.**
*Why.* RF-6.1 and RF-6.2 are easiest to guarantee by absence: targets are reached only by the runner from the approved environment snapshot (F-RUN), and the runner receives an opaque execution context, never a session or bearer credential (register §4 and §10). The monitor renders C07 facts, so a target's 401 is a fact row and cannot reach the interceptor; stream loss is a realtime event handled by `shared/realtime`, which never maps it to session end.
*Discarded.* Filtering target responses in the interceptor by URL — a rule that could be misconfigured; an absent path cannot be. Letting the console proxy a target for previews — a second path for ASpectra credentials to leak.
*Covers.* RF-6.1–6.3, RF-10.2, LC-10, LC-11.

**D13 · Denied scope, forgery failure and transport failure each have their own handler and problem code; none touches the session.**
*Why.* RF-5 forbids presenting any of the three as expiry and forbids loops. `DeniedScopeHandler` refreshes permissions exactly once per problem (a counter on the handler, not a retry loop), `CsrfFailureHandler` only reports, `TransportFailureStatus` only offers a retry; the three codes come from `access` and the security configuration, never from the session filter.
*Discarded.* Treating 403 as "maybe expired, try refresh" — FR-IDN-064's loop. Signing out on a CSRF failure — a forged request could sign the person out. Showing R34 on a network error — Q3's rejected option.
*Covers.* RF-5.1–5.6, RNF-7, LC-7–LC-9.

**D14 · Re-authentication binds to the ended session, which is retained for a configured time after it ends.**
*Why.* The dialog needs a server-side anchor for "the identity of the ended session" (RF-8.2) without the client naming an identity; the ended session row is that anchor, is already needed by the 007 list and the audit log, and its retention is a configured value (P13). After retention the start answers the generic sign-in failure and the person uses the sign-in screen.
*Discarded.* Sending the address from client memory — the client would choose the identity. Keeping ended sessions forever — unbounded personal data (register §3).
*Covers.* RF-8.2, RF-8.7, RF-8.8, LC-12.

**D15 · Expiry is enforced on access and swept by a durable job; retention is pruned by the same job.**
*Why.* Every request checks deadlines first (D1), so an expired session can never act; the existing durable-jobs facility **[002 D9]** marks expired rows ended with their cause so the list and the audit log are truthful, and prunes ended sessions and families past retention in bounded batches.
*Discarded.* Sweep only — an expired row could act between sweeps. Check-on-access only — the 007 list would show expired sessions as live and the audit log would lack the expiry event.
*Covers.* RF-2.6, RF-11.1, RNF-11.

**D16 · The reuse notification is an outbox template; the revocation never waits for it.**
*Why.* Decision Q6 (A-50): the revocation is the security effect and must commit with the detection; the message goes through `SecurityNotificationPort` **[002]** behind the outbox so a provider failure is retried by the relay and recorded, never surfaced to the refresh answer (RNF-9).
*Discarded.* Synchronous mail — couples the refusal to a provider. No notification — Q6's rejected option.
*Covers.* RF-11.4, RNF-9, LC-29.

**D17 · Every configured number is one validated `SessionPolicyProperties` record that fails startup, and the client's numbers come from it.**
*Why.* P13 and FR-X-046; the cross-field rules (idle ≤ absolute, credential lifetime < idle) are checked at bind time so a policy that could never expire or never refresh cannot start.
*Discarded.* Defaults in code or in the client — forbidden. Lazy validation — the first refresh would fail instead of startup.
*Covers.* RF-2.1, RF-4.6, RF-9.1, RNF-5.

**D18 · No dedicated mobile pages; every state reflows, and text comes from the feature-scoped i18n bundles [002 D16].**
*Why.* M23 is the drawn counterpart of R34 (S-02); FR-X-055 requires AA reflow at every other width; `SecurityStatements` shared across widths keeps every statement present (RNF-14). The server sends codes only; the locale header selects the template language.
*Discarded.* Separate mobile components — drift. Server-rendered messages — ties language to the request.
*Covers.* RF-12.1–12.3, RNF-12–14.

**D19 · Errors are RFC 9457 problem details with safe codes and a correlation id [001 D12].**
*Why.* F-X-03; the four codes of this feature say only what the client must do next; the correlation id joins the audit record to the person's report.
*Discarded.* Ad-hoc error bodies; a cause field on the session problem (RF-7.3).
*Covers.* RF-7.3, RF-11.2, RF-11.3.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**) | Pure domain: `Session` deadlines and `touch`, `RefreshFamily.rotate` outcomes, `ActivityClass` truth table, `SessionPolicy` cross-field rules, `LoginTransaction(REAUTHENTICATION)` with a fixed clock. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A; Testcontainers **test-only**) | Use cases through real adapters: row locks, rotation, reuse revocation, version-lazy ending, revokeAll(except), job sweep and pruning, outbox rows, audit rows, configuration binding. |
| Module boundary | ArchUnit (**test-only**) | The application module depends on no Spring or persistence class and imports no project, grant, approval, run or intent package. |
| API contract | Spring MockMvc (**test-only**) | Status codes, the four problem codes, correlation id, cookie attributes, CSRF and CORS behaviour, absence of any credential or cause in bodies, the remaining-time header on every authenticated answer. |
| Concurrency | JUnit 5 with real PostgreSQL | Two concurrent presentations of one refresh credential, duplicate retry, revokeAll racing a refresh, version bump racing a request. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent, unbounded or inconsistent value. |
| FE unit | Vitest + Testing Library (**test-only**) | Interceptor routing per code, single-flight coalescing, same-request retry and epoch drop, background marker on refetches, boundary mask / clear / resume rules, draft stamp purge, dialog states, notice condition, nothing in storage. |
| FE e2e + accessibility | Playwright + axe (**test-only**) | Journeys in EN / ES / FR at desktop, tablet and mobile widths against the BE test doubles; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS, iOS) — versions pinned in the release-support matrix | Dialog focus entry and trap, announcements without focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RNF-1 | BE integration | `session.grantsNoAuthorityWithoutDerivation`; `access.derivesFromSessionFactsOnly` | 2 |
| RF-1.2, RF-1.4 | API contract | `cookie.secureHttpOnlySameSitePathRestricted`; `csrf.refusedWithoutSessionChange`; `cors.allowListOnly` | 2, 6 |
| RF-1.3, RNF-8 | FE unit | `storage.containsNothingFromFeature` | 9 |
| RF-1.5 | BE integration | `session.rowHoldsDeadlinesVersionAndContext` | 2 |
| RF-1.6 | BE integration | `session.neverSubstitutesForStepUpRecoveryOrForcedChange` | 2 |
| RF-2.1, RF-4.6, RF-9.1, RNF-5 | Startup | `sessionPolicy.startupFailsOnAbsentUnboundedOrInconsistentValue` (one case per property plus the two cross-field rules) | 2 |
| RF-2.2, RF-2.3, LC-21, LC-28 | BE unit, integration, FE unit | `activityClass.truthTable`; `session.deliberateRequestMovesIdleOnly`; `session.pollingNeverMovesIdle`; `transport.marksRefetchesBackground` | 3 |
| RF-2.4 | BE integration | `session.absoluteDeadlineEndsDespiteActivity` | 3 |
| RF-2.5, LC-18, LC-19, LC-20, LC-27 | BE integration | `session.versionBumpEndsAtNextUse`; `revokeAll.exceptKeepsActingSession`; `session.statusChangeEndsSession` | 3 |
| RF-2.6, RNF-11, LC-30 | BE integration | `session.endedRefusedOnNextRequest`; `session.individuallyRevokedRefused` | 3 |
| RF-2.7, RNF-4 | API contract, FE unit | `remainingHeader.onEveryAuthenticatedAnswer`; `sessionState.displaysNeverCountsDown` | 3 |
| RF-3.1, RNF-2 | BE integration | `refresh.rotatesAtomically` | 4 |
| RF-3.2 | BE integration | `refresh.neverMovesDeadlinesOrGrantsAuthority` | 4 |
| RF-3.3 | BE integration | `refresh.refusedForEndedUnknownOrRestricted` | 4 |
| RF-3.4, LC-1 | Concurrency, FE unit | `refresh.concurrentPresentationsOneHonoured`; `coordinator.coalescesConcurrentTabs` | 4 |
| RF-3.5, RF-11.4, RNF-3, LC-2, LC-3 | BE integration | `refresh.reuseEndsFamilyAndSessionEverywhere`; `refresh.duplicateRetryIsReuse`; `notification.reuseTemplateWithoutSecrets` | 4 |
| RF-3.6, RNF-7 | FE unit | `coordinator.outcomeIsRefreshedInvalidOrIndeterminate` | 6 |
| RF-4.1, RF-4.3 | FE unit, API contract | `interceptor.refreshesOnlyOnCredentialExpired`; `interceptor.ignoresAuthEndpointsAndNon401`; `authEndpoints.outsideSessionFilter` | 5 |
| RF-4.2, RF-9.6, RNF-9 | FE unit | `interceptor.retriesSameRequestOnce`; `interceptor.dropsRetryWhenEpochMoved` | 5 |
| RF-4.4, RF-5.6, LC-6 | BE integration, FE unit | `retry.sameKeyReturnsRecordedOutcome`; `lostAck.looksUpNeverResubmits` | 5 |
| RF-4.5, LC-7 | FE unit | `interceptor.invalidEndsSessionIndeterminateShowsStatus` | 6 |
| RF-5.1, RF-5.2, LC-8 | FE unit, BE integration | `deniedScope.refreshesPermissionsOnceThenN28`; `access.deniedScopeNeverRefreshesCredentials` | 6 |
| RF-5.3 | FE unit | `deniedScope.draftShownOnlyIfRefreshedGrantPermits` | 6 |
| RF-5.4 | FE unit | `transportStatus.neverClaimsExpiryOffersRetryRefresh` | 6 |
| RF-5.5, LC-9 | API contract, FE unit | `csrf.ownProblemNoSessionChange`; `csrfHandler.reloadAdviceOnly` | 6 |
| RF-6.1, RF-6.2, LC-10 | FE unit, BE integration | `monitor.targetUnauthorisedIsFactRow`; `console.hasNoTargetClient`; `runner.carriesNoConsoleCredential` | 7 |
| RF-6.3, LC-11 | FE unit | `realtime.lossRendersStreamInterruptedNotExpiry` | 7 |
| RF-7.1, RF-7.6, LC-22 | FE unit | `boundary.masksClearsPausesDisconnects`; `boundary.noSessionRequestWhileMasked` | 8 |
| RF-7.2, RF-7.7 | FE unit | `dialog.rendersDesignedCopy`; `dialog.compactVariantSameRules` | 8 |
| RF-7.3 | API contract | `sessionInvalid.carriesNoCause` | 8 |
| RF-7.4 | FE unit | `dialog.retryIsSecondaryAndOncePerActivation` | 8 |
| RF-7.5, LC-13 | FE unit | `dialog.recoveryAndSignOutDiscardContext` | 8 |
| RF-7.8, LC-23 | FE e2e | `multiTab.oneReauthRecoversOthersOnRefresh` | 8 |
| RF-8.1, RF-8.2 | BE integration, FE unit | `reauth.fullProofBoundToEndedSessionIdentity`; `reauth.throttledAsSignIn`; `reauthStep.addressReadOnly` | 8 |
| RF-8.3 | BE integration, FE e2e | `reauth.issuesSessionWithoutLanding`; `reauth.returnsToInterruptedContext` | 8 |
| RF-8.4, LC-14, LC-15, LC-17 | BE integration | `reauth.forcedChangeInsideTransaction`; `reauth.enrollmentGateOnReentry`; `reauth.removedMembershipLeadsToChooser` | 8 |
| RF-8.5, RF-10.1, RNF-10, LC-24 | BE integration | `reauth.touchesNoRunTableAndReplaysNothing`; `run.continuesAcrossExpiryAndReauth` | 10 |
| RF-8.6, LC-16 | FE unit, BE integration | `reauth.terminalFailureVoidsAndKeepsContext` | 8 |
| RF-8.7, LC-12 | FE e2e | `reload.afterExpiryShowsDialogThenRequestedRoute` | 8 |
| RF-8.8, LC-27 | BE integration | `reauth.restrictedIdentityGenericFailure` | 8 |
| RF-9.1, LC-4 | FE unit | `saveNotice.shownUnderThresholdWithDirtyOnly`; `saveNotice.neverExtendsSession` | 9 |
| RF-9.2, RF-9.3, LC-5, LC-17 | FE unit | `drafts.keptInMemoryUnderMask`; `drafts.purgedOnDifferentIdentityScopeOrGrant` | 9 |
| RF-9.4 | FE unit | `boundary.neverSavesAutomatically` | 9 |
| RF-9.5 | FE unit | `inFlightSave.answerHonouredElseLookup` | 9 |
| RF-10.2, LC-25 | BE integration, FE unit | `launch.uncertainSubmissionLookedUpNeverResubmitted` | 10 |
| RF-10.3 | BE integration | `sessionEnd.triggersNoRunPolicy` | 10 |
| RF-10.4 | FE unit | `statements.acceptedRunsContinuePresent` | 10 |
| RF-11.1, RF-11.2 | BE integration | `audit.entryPerEvent`; `audit.containsNoCredential`; `logs.containNoCredential` | 11 |
| RF-11.3 | API contract | `problems.fourSafeCodesWithCorrelationId` | 11 |
| RF-11.5 | Concurrency | `rotation.allOrNothingUnderFailure` | 4 |
| RF-12.1, RF-12.2, RNF-12, RNF-14 | FE e2e + axe, manual | `a11y.sessionStates.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 12 |
| RF-12.3, RNF-13 | FE e2e | `languageSwitch.changesDisplayOnly` (LC-26) | 12 |
| RNF-6 | BE integration | `failClosed.absentEndedSupersededUnknownRefuses` | 3, 4 |
| RNF-9, LC-29 | BE integration | `notification.failureNeverRevertsRevocation` | 4 |
| RNF-15 | all of the above | Area A: concurrent refresh, reuse, multi-tab, cookie / CSRF / CORS, runs surviving expiry | 13 |
| US-12 | all adversarial cases above | replay, polling extension, SUT-401 wall, draft inheritance, Run relaunch, cause disclosure suites | 1 |

### 5.3 Not tested here
Sign-in, the challenge and landing mechanics (plan 001), enrollment and the gate rule (003), reset and forced change (004), recovery (005), logout and the sessions list (007), the tenant switch (016), admission and run-scoped authority (F-RUN). Their stubs return fixed answers; `IssueSessionPort`, `CurrentSessionPort` and `RevokeSessionsPort` are exercised here as this feature's own surface.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q2–Q6 were recorded in Figma on 27 October 2026 as **A-31** (closed), **A-49**, **A-50**, **S-39**, **S-40** and **S-41**; this plan implements each as decided (re-authentication D8, activity D3, reuse D5 / D16, dialog step D8, notice D11, retry status D13). The items below remain open.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-11 configuration values | `SessionPolicyProperties` holds idle and absolute deadlines, credential lifetime, save-notice threshold, safe-read retry bounds, retention values, sweep cadence and batch; values come from the catalogue. | Rename or regroup properties; no use case changes. |
| A-27 identity statuses | `AssertSessionUseCase` and the re-authentication accept `ACTIVE` only; `RESTRICTED` and `ERASING` end the session and fail generically. | Change the predicate only. |
| A-30 sign-in notifications | Only the reuse notification is sent by this feature (A-50); no "new session established" message. | Add a template and a call in `IssueSessionUseCase`. |
| A-35 persistence tooling | Adapters and migrations against the placeholder API and tool. | Rename base types and migration format; no use case changes. |
| A-39 revocation scope | `RevokeSessionsPort.revokeAll(identity, except, cause)` implemented here as decided for 003 and 004; run-scoped authority never referenced. | Extend the port's contract for other callers. |
| A-45 forced change reading | Reading (a): `FORCED_CHANGE_REQUIRED` **[004]** inside the re-authentication transaction. | Reading (b) would add an in-session interruption sequence to `ProtectedBoundary`. |
| S-02 responsive coverage | R34 has M23 MobileSessionExpirySheet (`432:4427`); the compact panel, the notice, the retry status and N28 follow the D01 responsive policy (`414:2645`). | None: the components are unchanged. |
| S-07 app-shell sign-out | "sign out safely" on R34 calls plan 007's logout; no shell menu is assumed. | Point the dialog's route at the designed menu action. |
| S-14 templates | `session-credential-reuse` with the D04 copy (`437:4431`) in EN / ES / FR. | None; the variables are the D04 ones. |
| S-42 forgery-defence state | `CsrfFailureHandler` renders a safe problem with "reload and try again". | Adjust the copy and placement; the handler is unchanged. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.6) | `Session`, `IssueSessionUseCase`, codec, cookie configuration, `AccessCredentialHolder` | D1, D2 | RF-1 rows |
| RF-2 (2.1–2.7) | `Session`, `ActivityClass`, `AssertSessionUseCase`, `EndSessionUseCase`, `SessionPolicyProperties`, remaining header | D1, D3, D4, D9, D15, D17 | RF-2 rows |
| RF-3 (3.1–3.6) | `RefreshFamily`, `RefreshSessionUseCase`, `refresh_credential` | D5, D6, D16 | RF-3 rows |
| RF-4 (4.1–4.6) | axios interceptor, `RefreshCoordinator`, scope epoch, idempotency outcomes | D6, D7 | RF-4 rows |
| RF-5 (5.1–5.6) | `DeniedScopeHandler`, `CsrfFailureHandler`, `TransportFailureStatus`, `access.denied-scope`, CSRF configuration | D13 | RF-5 rows |
| RF-6 (6.1–6.3) | structural wall, `shared/realtime`, monitor | D12 | RF-6 rows |
| RF-7 (7.1–7.8) | `ProtectedBoundary`, `SessionExpiryDialog`, problem codes | D6, D10 | RF-7 rows |
| RF-8 (8.1–8.8) | `StartReauthenticationUseCase`, `LoginTransaction(REAUTHENTICATION)`, `CompleteSignInUseCase`, `ReauthenticationStep` | D8, D14 | RF-8 rows |
| RF-9 (9.1–9.6) | `SaveNowNotice`, stamped drafts, `ProtectedBoundary.resume` | D10, D11 | RF-9 rows |
| RF-10 (10.1–10.4) | absence of any run touch point, launch status lookup, statements | D1, D12 | RF-10 rows |
| RF-11 (11.1–11.5) | audit adapter, codec claims, hashed credentials, outbox template, problem catalogue | D2, D5, D16, D19 | RF-11 rows |
| RF-12 (12.1–12.3) | reflow, `Dialog`, `LiveRegion`, i18n bundles | D18 | RF-12 rows |
| RNF-1 | `SessionAssertionPort` → `access` derivation | D1 | RF-1.1 row |
| RNF-2, RNF-3 | `RefreshFamily.rotate` under lock | D5 | RF-3.1, RF-3.5 rows |
| RNF-4 | remaining header, `SessionState` | D4 | RF-2.7 row |
| RNF-5 | `SessionPolicyProperties` | D17 | startup |
| RNF-6 | check-on-access, locks | D1, D5 | `failClosed` |
| RNF-7 | two problem codes, coordinator outcomes | D6, D13 | RF-3.6, RF-5.4 rows |
| RNF-8 | memory credential, hashed refresh, stamped drafts | D2, D10 | RF-1.3 row |
| RNF-9 | same-request retry, lookup | D7 | RF-4.2, RF-4.4 rows |
| RNF-10 | no run touch point | D1, D12 | RF-10.1 row |
| RNF-11 | per-request assertion, eager revokeAll | D1, D9 | RF-2.6 row |
| RNF-12, RNF-14 | `shared/ui`, reflow | D18 | RF-12 rows |
| RNF-13 | i18n bundles, locale header | D18 | `languageSwitch` |
| RNF-15 | concurrency and contract suites | D5, D7 | area A row |
| LC-1 … LC-30 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30) |

Self-check: every RF criterion (68), every RNF (15) and every limit case (30) of `specSessionsRefreshExpiry.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
