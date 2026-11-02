# Plan 007 — Logout and session management

**Feature** Sign-out, the sessions list, individual revocation and "Sign out everywhere" · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/007-LogoutSessionManagement/specLogoutSessionManagement.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-09 as the subject; F-IDN-03, F-IDN-04, F-IDN-06, F-IDN-07, F-IDN-08, F-IDN-10, F-IDN-15, F-IDN-16, F-IDN-18 and F-SET-03 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG; the `identity` context owns "sessions / refresh / recovery"), FE board `46:7847` (feature-modular SPA DAG; `app/` composition, `shared/api`, `shared/security`, `shared/realtime`, `shared/errors`), board 3 `46:7847` (guarded switch order) and the decision register `122:2` ("Session expiry, refresh, logout and accepted runs": "Explicit logout revokes the session server-side; sign-out-everywhere revokes identity sessions") · **Builds on** plan 001 (Login), plan 002 (Signup and verification), plan 003 (Enrollment), plan 004 (Password reset), plan 005 (Controlled recovery) and plan 006 (Sessions, refresh and expiry); elements they own are tagged **[001]** … **[006]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. Plan 006 built the session service and left two things to this plan by name: the logout endpoint (`DELETE` on `SessionController`) and the sessions list; it also published `EndSessionUseCase`, `CurrentSessionPort` and the `Session` aggregate for this plan to use. The specification's eight decisions (Q1–Q8) and five derived rules (D-a…D-e) are built as decided; the register items it recorded (A-51, S-43, S-44) are therefore implemented, not assumed. Where the specification depends on items still open (A-11, A-27, A-35, A-39 port contract, A-49, A-50, A-31, S-02, S-07, S-28, S-41), the plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | No use case of this feature reads, stores or replays a mapping, approval, launch command or intent; signing out, revoking and "Sign out everywhere" touch identity tables only; a pending Run request is looked up by F-RUN's status endpoint after the next sign-in, never resubmitted. | RF-2.6, RF-10.1–10.3, RNF-11 |
| **P3** The browser expresses intent, never authority | The client asks to end *its* session, to end *one session id*, or to end *all*; the server resolves the acting identity from the asserted session and decides whether the named session is the identity's own. The list is an identity-scoped server read; the client renders it. | RF-2.1, RF-6.4, RF-7.2, RF-8.8, RNF-4, RNF-5 |
| **P4** Only the platform catalog is global | Sessions remain identity-scoped platform rows; the last-scope columns name a tenant and project but grant nothing and are read only to render the list. | RF-6.2, RF-6.5 |
| **P6** Fail closed | An ended, unknown or foreign session refuses revocation with one generic problem; an indeterminate sign-out clears nothing; a forgery-defence refusal changes nothing; an absent page size fails startup. | RF-2.7, RF-2.8, RF-4.1, RF-7.2, RF-7.4, RNF-6 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security session support **[006]**, the audit port. FE: React 18, TypeScript, Vite, TanStack Query, Zustand (scoped drafts **[006]** only), i18next, Tailwind, axios, STOMP. No user-agent library, no cross-tab messaging library, no notification. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: three use cases, one query and one write hook in the existing hexagonal `identity` context; `access` calls one published port after C10 derivation and publishes one membership port; outer adapters depend inward. FE: the sign-out sequence in `shared/security`, the shell action in `app/shell`, the Sessions area in the `identity-access` feature; TanStack Query owns server facts; Zustand holds only the drafts the boundary already protects. | §2 |
| **P12** Two independent tenancy gates | Session rows carry no tenant column and RLS does not apply to them; the membership check behind "Identity session" is performed by `access` under RLS and runtime authorisation; nothing of this feature reads a tenant table directly. | RF-6.4, RF-6.5, RNF-5 |
| **P13** Configured values are never defaulted | The list page size joins `SessionPolicyProperties` **[006]** and fails startup when absent or unbounded; no number lives in the client. | RF-6.7, RNF-9 |
| **P14** Release bar | WCAG 2.2 AA on the Sessions area, the confirmation, the dirty guard use and the retry status with automated and manual assistive evidence; EN / ES / FR for every state. | RF-12, RNF-13–15 |
| **F-X-01** C10 context | Every endpoint of this feature runs behind the session filter **[006]**; the acting identity comes from the asserted session; `RecordSessionScope` is invoked by `access` only after a successful derivation and never influences it. | RF-6.4, RF-7.2, RNF-4 |
| **F-X-02** Request integrity | Sign-out and revocation are idempotent by state (an ended session answers "done"); "Sign out everywhere" carries an idempotency key whose outcome is recorded; no mutation auto-retries; the person retries. | RF-2.7, RF-4.2, RF-7.4, RF-8.6, RNF-10 |
| **F-X-03** Secrets and safe problems | No credential leaves the server; the raw user-agent string is classified at issue and never stored or shown; problems carry safe codes (`auth.session.invalid` **[006]**, `auth.csrf.invalid` **[006]**, `auth.session.not-revocable`) and a correlation id; browser storage holds nothing from this feature. | RF-6.3, RF-11.2, RF-11.3, RNF-7, RNF-12 |
| **F-X-04** Accessibility | Text status for every outcome, accessible names on row actions, dialog focus management, 44 px targets. | RF-12.3 |
| **F-X-05** Audit log | Every event of RF-11.1 appended through `AuditPort`, redacted, in the same transaction as the effect. | RF-11.1 |
| **F-X-06** Configuration validation and email port | Page size in startup validation; the email port is not used (decisions Q2, Q8). | RF-11.5, RNF-9 |
| **F-X-07** Responsive surfaces | The shell action joins the M04 menu; the Sessions area, the confirmation and the retry variant reflow under the AA rules with the statements shared across widths. | RF-12.1, RF-12.2, RNF-15 |
| **F-IDN-08** boundary | No second session model: ending is `EndSessionUseCase` **[006]**; the definitive answer other tabs receive is `auth.session.invalid` **[006]**; the retry status is `TransportFailureStatus` **[006]**. | RF-2.1, RF-4.2, RF-5.1 |
| **F-IDN-18** boundary | The dirty guard is invoked through the published hook of the guarded switch; nothing of the guard, the epoch or the dialog N24 is redefined here. | RF-3.1–3.4, RF-3.7 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity`, plus one hook and one port in `access`

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 001–006. This plan extends the `Session` aggregate **[006]**, adds one pure classifier, one read model, three use cases, one query, one write hook and two ports, and completes `SessionController` **[006]** with the four endpoints plan 006 left to it. Contexts touched — `access` (membership existence; the scope hook), `audit`, `sharedsupport` — are reached only through published ports.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `Session` **[006]** | Extended with `clientContext` narrowed to `{browserFamily, platformFamily}` (enums; the raw user-agent string never enters the aggregate) and `lastScope` `{tenantId, projectId, at}` (nullable). New methods: `recordScope(tenantId, projectId, now)` (no-op when unchanged; never touches deadlines or `lastActivityAt`), `endByHolder(cause, now)` = `end` **[006]** restricted to the holder-initiated causes `LOGOUT`, `REVOKED`, `SIGN_OUT_EVERYWHERE` (the last is a new cause value). State machine unchanged. | RF-2.1, RF-6.2, RF-7.1, RF-8.3, RNF-2 |
| | `ClientContextClassifier` | Pure rule `(userAgentString) → ClientContext{browserFamily, platformFamily}` over a short fixed vocabulary — browser `CHROME, FIREFOX, SAFARI, EDGE, OTHER`; platform `WINDOWS, MACOS, LINUX, IOS, ANDROID, OTHER` — by substring precedence; unknown input yields `OTHER / OTHER`. Called once by `IssueSessionUseCase` **[006]**; the string is discarded afterwards (D3). | RF-6.2, RF-6.3, RF-11.2, RNF-12 |
| | `SessionView` | Read model of one row: session id, `browserFamily`, `platformFamily`, `createdAt`, `context` = `Scope(tenantName, projectName) \| IDENTITY_SESSION`, `expiresAt` = the nearer deadline, `isCurrent`, `action` = `CURRENT \| REVOKABLE`. Built by `ListSessionsQuery`; carries no deadline pair, version, family state, network key or raw agent. | RF-6.2, RF-6.3, RF-6.6, RF-6.10 |
| | `SessionPolicy` **[006]** | Gains `sessionListPageSize`. | RF-6.7, RNF-9 |
| | `LoginTransaction` **[001]** | Unchanged; its `void` is invoked by `SignOutEverywhereUseCase` through the repository for every open transaction of the identity. | RF-8.3 |
| `access.domain` | C10 derivation **[001/003/006]** | After a successful derivation for a tenant (and optionally a project), the derivation step calls `SessionScopePort.recordScope(sessionId, tenantId, projectId)`; a failed or denied derivation calls nothing. `access` also implements `MembershipExistsPort.activeMembership(identityId, tenantId) → Optional<Names>` under RLS, returning the tenant (and project) display names only for an active membership of that identity. | RF-6.2, RF-6.5 |
| `identity.application` | `SignOutUseCase` | Input: the asserted session (from the filter) and the `entry` (`SHELL, N03, N20, M03, N29, R34`). Loads the session; if `ACTIVE`, `endByHolder(LOGOUT)` and end its `RefreshFamily` **[006]** in one transaction, audit `SIGNED_OUT{entry}`; if already ended, audit nothing new and answer `ALREADY_ENDED`. Both answers are definitive; the controller maps both to the same client effect (RF-2.7). Reads no tenant, grant or run table. | RF-2.1, RF-2.4–2.7, RF-9.1–9.4, RF-10.1, RF-11.1, LC-5, LC-8, LC-12, LC-13 |
| | `RevokeSessionUseCase` | Input: acting identity and session (from the filter), target session id. One conditional statement selects the target `WHERE id = ? AND identity_id = ? AND id <> acting` for update; no row → `NOT_REVOCABLE` (foreign, unknown or current — one answer), audit `REVOCATION_REFUSED`; row `ACTIVE` → `endByHolder(REVOKED)` + family end, audit `SESSION_REVOKED_BY_HOLDER{target}`; row already ended → `DONE` (idempotent, audited as `SESSION_REVOKED_BY_HOLDER{target, alreadyEnded}`). No confirmation, no step-up (decision Q8). | RF-7.1–7.7, RF-11.1, LC-17–LC-20 |
| | `SignOutEverywhereUseCase` | Input: acting identity, idempotency key. Returns the recorded outcome for a repeated key. Otherwise one transaction: `GlobalIdentity.bumpSecurityVersion()` **[004]**; `EndSessionUseCase.revokeAll(identityId, except = none, cause = SIGN_OUT_EVERYWHERE)` **[006]** (ends every session and family including the acting one); `LoginTransactionRepository.voidAllOpen(identityId)` **[001]**; audit `SIGNED_OUT_EVERYWHERE{count}`; records the outcome under the key. Sends no notification (decision Q2). | RF-8.3, RF-8.4, RF-8.6–8.8, RF-11.1, RF-11.5, LC-23–LC-26 |
| | `ListSessionsQuery` | Input: acting identity and session, page number. Reads `ACTIVE` sessions of the identity ordered by `createdAt desc`, paged by `sessionListPageSize`; for each row builds `SessionView`: `isCurrent` from `CurrentSessionPort` **[003]**; `context` = `MembershipExistsPort.activeMembership(identity, lastScope.tenantId)` → names, else `IDENTITY_SESSION`; `expiresAt = min(idleDeadline, absoluteDeadline)`. A page beyond the last returns the last page. Structurally a deliberate read (navigation), so it moves the idle deadline as any navigation does (spec D-e). | RF-6.1, RF-6.2, RF-6.4, RF-6.5, RF-6.7, RF-6.8, LC-15, LC-16, LC-28 |
| | `RecordSessionScope` (implements `SessionScopePort`, published to `access`) | `recordScope(sessionId, tenantId, projectId)`: loads the session, calls `Session.recordScope`, persists only when changed (D4). Classified `BACKGROUND` for `ActivityClass` **[006]** by construction — it is not a request. | RF-6.2, RF-6.5 |
| | `IssueSessionUseCase` **[006]** | Extended: receives the user-agent string from `ClientContextPort`, classifies it with `ClientContextClassifier`, stores the two enums, discards the string. | RF-6.2, RF-6.3 |
| | Ports | New: `SessionScopePort` (published to `access`), `MembershipExistsPort` (published by `access`, consumed here), `ClientContextPort` (the transport's user-agent header, read once at issue). Reused: `SessionRepository`, `RefreshFamilyRepository`, `EndSessionUseCase`, `CurrentSessionPort` **[006/003]**; `IdentityRepository`, `LoginTransactionRepository`, `AuditPort`, `Clock` **[001]**; idempotency outcome record **[001 D1]**. Not used: `SecurityNotificationPort`, `TransactionalEmailPort` (decisions Q2, Q8). No port reaches a project, grant, approval, run or intent table. | all |
| `identity.infrastructure` | Persistence | `identity_session` **[006]** gains `browser_family`, `platform_family` (enum text), `last_tenant_id`, `last_project_id`, `last_scope_at` (nullable); index `(identity_id, state, created_at desc)` for the list; `end_cause` gains the value `SIGN_OUT_EVERYWHERE`. `sign_out_everywhere_outcome` (`idempotency_key` unique, `identity_id`, `recorded_at`). Access API and migration tool per A-35 (placeholders). | RF-6.1, RF-6.2, RF-8.3, RF-8.6 |
| | Security filter and cookie adapter **[006]** | The four endpoints of this feature run inside `SessionAuthenticationFilter` **[006]** (the acting session is the asserted one) with CSRF on the three mutating ones. The sign-out and sign-out-everywhere responses carry the refresh cookie cleared (expired, same name, path and attributes), so the browser drops it when — and only when — the server answered (D2). | RF-2.2, RF-2.8, RF-8.3, LC-9 |
| | `SessionPolicyProperties` **[006]** | Gains `sessionListPageSize`: present, positive, bounded above by a configured ceiling; any violation fails startup. | RF-6.7, RNF-9 |
| | Audit adapter **[006]** | New events: `SIGNED_OUT{entry}`, `SESSION_REVOKED_BY_HOLDER{target, alreadyEnded}`, `REVOCATION_REFUSED{target}`, `SIGNED_OUT_EVERYWHERE{count}`, `SIGN_OUT_CSRF_REFUSED`. `SESSION_ENDED{cause, source}` **[006]** is still written by `EndSessionUseCase` for each ended session. Payload: identity id, acting session id, target session id, correlation id, network key, timestamp. Never a credential, a hash, a raw user-agent string, a password or a code. | RF-11.1, RF-11.2 |
| `identity.api` | `SessionController` **[006]** — `/api/auth` (session filter; CSRF on mutations) | `DELETE session` {entry} → 204 with the cleared cookie (session ended, or already ended — RF-2.7), 403 `auth.csrf.invalid` **[006]**; a request whose access credential is missing or whose session is invalid answers 401 `auth.session.invalid` **[006]** before the controller, which the client treats as definitive (RF-2.7) · `GET sessions?page=n` → 200 `{items: SessionView[], page, pageCount, total}` · `DELETE sessions/{id}` → 204 (ended or already ended) or 404 `auth.session.not-revocable` (one code for foreign, unknown and current) · `POST sessions/sign-out-everywhere` {idempotencyKey} → 204 with the cleared cookie. All problems are RFC 9457 with a safe code and a correlation id; no body of this feature carries a credential, a deadline pair, a version or a cause. | RF-2.1, RF-2.2, RF-2.7, RF-2.8, RF-6.1, RF-7.1, RF-7.2, RF-8.3, RF-11.3 |

**Cross-context touch points.** `access` — consumes `SessionScopePort` after derivation; publishes `MembershipExistsPort`. `audit` — append-only. `sharedsupport` — `tx`, `contextIds`. `execution` — nothing: run-scoped authority never references a browser session **[006 D12]**, which is how RF-10 holds by construction. Nothing reads or writes project, grant, approval or run tables.

**Data model note (P12).** The new columns are attributes of a platform-scoped session row of a global identity; the tenant and project ids they hold are references for display, resolved to names only through `access` under RLS at read time, never joined directly.

### 2.2 Front end — `app/shell`, `shared/security` and the feature module `src/features/identity-access`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| `app/shell` | `SignOutAction` | A labelled "Sign out" action rendered in the identity block of the navigation footer on every desktop app-shell route and as an item of the M04 menu on mobile; visible focus, 44 px target; built as the S-07 assumption. Activation → `SignOutCoordinator.signOut({entry: 'SHELL'})`. Rendered by the shell for every authenticated route whatever the tenant state or grants (it follows no grant). | RF-1.2, RF-1.3, RF-1.5, RF-12.2, RF-12.3 |
| `shared/security` | `SignOutCoordinator` | One ordered sequence for every entry: (1) if the entry is `SHELL` and the dirty-draft registry **[006]** is non-empty, await the guarded-switch dirty guard hook (`Save \| Discard \| Stay`): `Stay` → return, nothing touched; `Save` → the screen's own save under the draft's stamped scope, awaited; a failed, refused or indeterminate save → return to the screen, no sign-out; `Discard` → purge the drafts; (2) abort in-flight requests and disconnect STOMP (the switch's step 2 primitives **[001/006]**); (3) `DELETE session` with the entry; (4) on a definitive answer (204, or 401 `auth.session.invalid`) → `AccessCredentialHolder.clear()` **[006]**, purge every Zustand draft, reset the query client, advance the scope epoch, set `SessionState.status = SIGNED_OUT`, route `/sign-in`; on `INDETERMINATE` → `TransportFailureStatus` **[006]** variant `signOutIncomplete` with "Retry sign out" (one request per activation), nothing cleared, `SessionState` unchanged, streams reconnect on the next successful request; on `auth.csrf.invalid` → `CsrfFailureHandler` **[006]**, nothing cleared. `fromEndedSession()` (R34 "sign out safely"): discards context and drafts, fires `DELETE session` without awaiting, routes `/sign-in` (D-d). `signOutEverywhere(idempotencyKey)`: steps (2)–(4) with `POST sessions/sign-out-everywhere` and the variant `signOutEverywhereIncomplete`. | RF-2.2, RF-2.3, RF-2.8, RF-3.1–3.7, RF-4.1–4.5, RF-5.4, RF-8.5, RF-8.6, RF-9.4, LC-1–LC-7, LC-13, LC-24 |
| | `SessionState` **[006]** | Gains `SIGNED_OUT` (the signing-out tab never renders R34 for its own sign-out). | RF-5.4 |
| | `ProtectedBoundary` **[006]** | Unchanged: other tabs of the browser context receive `auth.session.invalid` on their next eligible request and apply mask, secret clearing, stream closing and R34 with drafts kept (decision Q6). | RF-5.1, RF-5.2, LC-10, LC-11 |
| `shared/api` | axios transport **[001/006]** | `DELETE session`, `DELETE sessions/{id}` and `POST sign-out-everywhere` are mutations: never auto-retried; `auth.session.invalid` on `DELETE session` is routed to the coordinator as definitive rather than to `ProtectedBoundary`. | RF-2.7, RF-4.1, RNF-10 |
| `identity-access` | `SessionsArea` (R38 Sessions tab) | Hosted by `SecuritySettingsPage` **[003]**: heading "Authenticated sessions", `SessionsTable`, record count, `Pagination`, `SignOutEverywhereAction`, `SecurityStatements`. Reads with `useSessions(page)` on open and refetches after each action; no interval refetch (D-e). | RF-6.1, RF-6.6, RF-6.8, RF-8.1 |
| | `SessionsTable` | Columns Session (`browserFamily / platformFamily` labels + `createdAt`), Context (names or "Identity session"), Expiry (`expiresAt`), Action ("Current session" text on `isCurrent`, else "Revoke session" button whose accessible name includes the session description). On revoke: `useRevokeSession(id)`; success or `already ended` → the row shows "Session revoked" (text, live region polite) until the next read; `auth.session.not-revocable` → one generic inline problem; `INDETERMINATE` → `TransportFailureStatus` on the row, action repeatable. Mobile and tablet: one record per session with the same four cells and the action (D14). | RF-6.2, RF-6.3, RF-6.9, RF-6.10, RF-7.1, RF-7.3, RF-7.4, RF-7.6, RF-12.1, RF-12.3, LC-14, LC-17–LC-20 |
| | `Pagination` | "Previous · Next" with unavailable state conveyed in text; page from the server response. | RF-6.7, LC-28 |
| | `SignOutEverywhereAction` + `ConfirmDialog` | Button beside the list (S-43, drawn `427:4040`); opens `shared/ui` `Dialog` with the statements of RF-8.2 (every session including this one, accepted runs continue, sign in again), "Sign out everywhere" and "Cancel"; no password or code field; confirm → `SignOutCoordinator.signOutEverywhere(key)` with a key generated once per dialog open and reused on retry. | RF-8.1, RF-8.2, RF-8.5, RF-8.6, RF-10.3, RF-12.3, LC-23, LC-24 |
| | `SecurityStatements` **[001]** | Extended with "Sessions are not trusted devices. No session grants a TOTP bypass. Expiring a session does not cancel an accepted run." so desktop and reflow cannot drift. | RF-6.6, RF-10.3, RF-12.1 |
| | `useSessions()`, `useRevokeSession()`, `useSignOutEverywhere()`, `useSignOut()` | Typed hooks over `contracts/sessions`; return server state verbatim; mutations without retry. | RF-6, RF-7, RF-8 |
| | `contracts/sessions` (in `src/contracts`) | Request / response / problem types and runtime guards for the four endpoints. | all |
| Other entries | N03 **[013]**, N20 **[014]** | Render `SignOutAction` with entries `N03` / `N20` (no draft, no guard). | RF-1.1, RF-9.1, RF-9.2, LC-12 |
| | M03 / N29 **[008]** | "Switch identity safely" → `SignOutCoordinator.signOut({entry})`; the invitation is untouched; the return route is plan 008's. | RF-1.1, RF-9.3, LC-12 |
| | R34 **[006]** | "sign out safely" → `fromEndedSession()`. | RF-3.5, RF-9.4, LC-13 |
| | R08 **[004]** | "Sign out safely" = `AbandonChallengeUseCase` **[001]** through `DELETE challenge`; nothing of this feature. | RF-1.4, RF-9.5 |
| Shared used | `shared/i18n` (bundle keys for every statement, state and message), `shared/ui` (`Dialog` with focus trap and return focus, `LiveRegion`, `Table`/`RecordList`, 44 px targets), `shared/errors` (`TransportFailureStatus`, `CsrfFailureHandler` **[006]**). | RF-12.1–12.4 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `SignOutUseCase` accepts every entry value | `SignOutAction` on N03 / N20; M03 / N29 action; R34 footer; R08 is **[004]** |
| RF-1.2 | — | `SignOutAction` in `app/shell` footer and M04 menu on every authenticated route |
| RF-1.3 | `DELETE session` requires only an asserted session; no grant, enrollment or tenant-state check | `SignOutAction` rendered regardless of grants or gate state |
| RF-1.4 | `AbandonChallengeUseCase` **[001]** for R08 / R05 / N05 | Those screens call `DELETE challenge`, never `DELETE session` |
| RF-1.5 | — | `shared/ui` button: label, focus, 44 px; menu opens by activation |
| RF-2.1 | `SignOutUseCase`: `endByHolder(LOGOUT)` + family end in one transaction; `AssertSessionUseCase` **[006]** refuses afterwards | — |
| RF-2.2 | Cleared cookie on the 204 | Coordinator step (4): clear holder, drafts, query client, epoch; route `/sign-in` |
| RF-2.3 | — | `/sign-in` **[001]** opens empty; nothing carried over |
| RF-2.4 | `SignOutUseCase` ends the asserted session only | — |
| RF-2.5 | Use case touches `identity_session` / `refresh_family` only | No other state written |
| RF-2.6 | No run touch point (**[006 D12]**) | — |
| RF-2.7 | `ALREADY_ENDED` → 204; filter's 401 `auth.session.invalid` | Both definitive → step (4) |
| RF-2.8 | CSRF on `DELETE session`; 403 `auth.csrf.invalid`, no change | `CsrfFailureHandler`; nothing cleared |
| RF-3.1 | — | Coordinator step (1): dirty guard hook; `Stay` returns |
| RF-3.2 | Save endpoint of the screen with its idempotency key **[001 D1]** | `Save` → the draft's own save under its stamped scope, awaited |
| RF-3.3 | — | Failed / refused / indeterminate save → return, no sign-out; lookup per **[006 RF-9.5]** |
| RF-3.4 | — | `Discard` → purge, proceed |
| RF-3.5 | — | `fromEndedSession()` discards without guard; N03 / N20 / M03 / N29 have no registry entries |
| RF-3.6 | Idempotency outcomes **[001 D1]** | Step (2): abort + STOMP disconnect before `DELETE`; lookup after next sign-in |
| RF-3.7 | — | Save uses the draft's stamp, never the current selector; no feature calls the coordinator to discard |
| RF-4.1 | Two definitive answers only (204, 401 invalid) | `INDETERMINATE` only when no answer arrived |
| RF-4.2 | — | `TransportFailureStatus.signOutIncomplete` with "Retry sign out"; nothing cleared |
| RF-4.3 | Session untouched | Status dismissable; streams reconnect on next success |
| RF-4.4 | Retry meets 401 `auth.session.invalid` | Treated as definitive → step (4) |
| RF-4.5 | Recorded save outcome | Save not re-asked; registry already empty |
| RF-5.1 | `AssertSessionUseCase` **[006]** → `auth.session.invalid` for the ended session | `ProtectedBoundary` **[006]** in other tabs |
| RF-5.2 | Problem carries no cause **[006 D6]** | R34; re-authentication **[006]**; "sign out safely" → `fromEndedSession()` |
| RF-5.3 | Other browsers' sessions untouched by `SignOutUseCase`; revoked sessions refused at next use | — |
| RF-5.4 | — | `SessionState.SIGNED_OUT` in the signing-out tab; route, never R34 |
| RF-6.1 | `ListSessionsQuery`: identity-scoped, `ACTIVE`, `createdAt desc`, paged | `SessionsArea`, `SessionsTable`, count, `Pagination` |
| RF-6.2 | `SessionView` from families, `createdAt`, `lastScope` names, nearer deadline, `isCurrent` | Four cells rendered from the view |
| RF-6.3 | `SessionView` has no such field; classifier discards the string | Nothing else rendered |
| RF-6.4 | Query keyed by the asserted identity; no admin endpoint exists | — |
| RF-6.5 | `MembershipExistsPort` empty → `IDENTITY_SESSION` | "Identity session" label |
| RF-6.6 | — | `SecurityStatements` |
| RF-6.7 | `sessionListPageSize` validated; last page for overflow | `Pagination` text state |
| RF-6.8 | — | Fetch on open and after each mutation; no interval |
| RF-6.9 | `AssertSessionUseCase` **[006]** refuses the list for an ended viewer | `ProtectedBoundary` **[006]** |
| RF-6.10 | No endpoint extends or alters deadlines; current row `CURRENT` | No such control rendered |
| RF-7.1 | `RevokeSessionUseCase` conditional select + `endByHolder(REVOKED)` + family end | `useRevokeSession` on activation, no dialog |
| RF-7.2 | `WHERE id = ? AND identity_id = ? AND id <> acting`; 404 `auth.session.not-revocable` | One generic inline problem |
| RF-7.3 | 204 | Row "Session revoked" text; viewer unchanged |
| RF-7.4 | Ended target → 204 | `INDETERMINATE` → row status, repeatable; never "revoked" without an answer |
| RF-7.5 | `SESSION_ENDED` **[006]**; no notification call | Other device: `ProtectedBoundary` **[006]** |
| RF-7.6 | `id <> acting` in the predicate | "Current session" text, no button |
| RF-7.7 | Only the two endpoints call the use case; no tenant-scoped caller | — |
| RF-8.1 | — | `SignOutEverywhereAction` beside the list |
| RF-8.2 | — | `ConfirmDialog` statements; no credential field |
| RF-8.3 | `SignOutEverywhereUseCase`: bump + `revokeAll(except none)` + void transactions + audit, one transaction; cleared cookie | Confirm → coordinator steps (2)–(4) |
| RF-8.4 | `AssertSessionUseCase` **[006]** refuses the other sessions; no notification | Other devices: R34 **[006]** |
| RF-8.5 | — | Settings route holds no draft; other tabs per RF-5.1 |
| RF-8.6 | Recorded outcome by idempotency key | `signOutEverywhereIncomplete` variant; same key on retry; 401 invalid is definitive |
| RF-8.7 | Use case touches identity session, family, version and login transactions only | — |
| RF-8.8 | Acting identity from the asserted session; no admin endpoint | — |
| RF-9.1 | `SignOutUseCase{entry: N03}` | N03 `SignOutAction` **[013]** |
| RF-9.2 | `SignOutUseCase{entry: N20}` | N20 `SignOutAction` **[014]** |
| RF-9.3 | `SignOutUseCase{entry: M03 \| N29}`; invitation rows untouched | M03 / N29 action **[008]** |
| RF-9.4 | `DELETE session` on an ended session → 204 or 401, both ignored | `fromEndedSession()` without await |
| RF-9.5 | `AbandonChallengeUseCase` **[001]** | R08 **[004]** |
| RF-10.1 | No run touch point | Nothing on sign-out touches runs |
| RF-10.2 | Launch status lookup **[F-RUN]** | Lookup after next sign-in, never resubmit **[006]** |
| RF-10.3 | — | `SecurityStatements`; `ConfirmDialog` statement |
| RF-11.1 | Audit adapter events in the effect's transaction | — |
| RF-11.2 | Payload shape; classifier discards the agent string; no credential anywhere | Nothing logged client-side |
| RF-11.3 | `auth.session.invalid`, `auth.csrf.invalid` **[006]**, `auth.session.not-revocable` | Mapped per code |
| RF-11.4 | One transaction per use case; cookie cleared only on commit | — |
| RF-11.5 | No notification port used | — |
| RF-12.1 | — | `SessionsTable` record layout at mobile / tablet; `ConfirmDialog` and status reflow |
| RF-12.2 | — | `SignOutAction` in the M04 menu, authority unchanged |
| RF-12.3 | — | `shared/ui` labels, focus trap / return, accessible names, live region polite, 44 px |
| RF-12.4 | — | i18n bundles; `expiresAt` formatted for display only |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · Sign-out, revocation and "Sign out everywhere" are three thin use cases over `EndSessionUseCase` [006]; no second session model exists.**
*Why.* Plan 006 holds the session, its family, the `end(cause)` transition, the audit of `SESSION_ENDED` and the per-request assertion that refuses an ended session (RNF-3). Everything this feature adds is *who may end which session and what is recorded*: the holder's own session, one named session of the holder, or all of them. Three use cases that call the existing transition keep one state machine, one audit trail and one refusal path.
*Discarded.* A logout table or a token blacklist — a second source of truth beside the session row. Ending by deleting rows — the list, the audit log and the retention of ended sessions **[006 D14]** need the ended row.
*Covers.* RF-2.1, RF-2.4, RF-7.1, RF-8.3, RNF-1, RNF-2, RNF-3, LC-5, LC-17, LC-23.

**D2 · The server clears the refresh cookie in the sign-out response; the client clears its memory only on a definitive answer.**
*Why.* Decision Q5: nothing is cleared until the server confirms. The cookie is `HttpOnly` **[006 D2]**, so only the server can drop it, and it does so on the same response that confirms the session ended — the browser cannot be left with a usable refresh credential after a confirmed sign-out (RNF-2), and cannot lose its credential before one (RNF-1). A 401 `auth.session.invalid` is equally definitive: the session is gone, so clearing is honest (RF-2.7).
*Discarded.* Clearing the client first and ending the session best-effort — Q5's rejected option; a live session would hide behind a sign-in screen. A client-side cookie deletion — impossible for an `HttpOnly` cookie and pointless for the server-side session.
*Covers.* RF-2.2, RF-2.7, RF-4.1, RF-4.4, RNF-1, RNF-2, RNF-6, LC-6, LC-7, LC-8.

**D3 · The browser and platform family are classified at session issue by a pure in-house rule into two small enums; the raw user-agent string is never stored.**
*Why.* Decision Q7 (A-51, S-44): the list shows a family, never the raw string, and no address or location. A fixed vocabulary of five browsers and six platforms by substring precedence is enough to recognise a session and is trivially testable; classifying once at issue means the string is discarded and nothing of it reaches storage, audit or the client (RNF-12). A user-agent parsing library is outside P10 and would need a register item for a precision the specification does not ask for.
*Discarded.* Storing the raw user-agent and parsing on read — disclosure and a string in the database. A parsing library — a dependency outside the stack. Showing the network key `clientContext` **[006]** already holds — Q7 excludes network and location.
*Covers.* RF-6.2, RF-6.3, RF-11.2, RNF-12, LC-15.

**D4 · The session's last tenant and project are recorded by `access` after derivation, write-on-change, as background work.**
*Why.* The Context cell needs the scope the session last acted in (Q7); `access` is the only place that knows the derived scope of a request (P3), so a hook there is the single truthful source. Writing only when the scope differs from the stored one keeps the per-request cost at zero for the common case, and classifying the write as background keeps it from counting as activity — the request itself already did or did not (**[006 D3]**).
*Discarded.* Writing on every request — a row update per request. Reconstructing the context from the audit log — the audit log is append-only and authority-scoped, not a read model. Letting the client post its selected tenant — the browser would name a scope it may not hold.
*Covers.* RF-6.2, RF-6.5, LC-15, LC-16.

**D5 · The list is an identity-scoped read model paged on the server; "Identity session" is computed through `access` under RLS.**
*Why.* RF-6.1 and RF-6.4: every live session of the identity across tenants, and only of this identity — keying the query by the asserted identity makes the second rule structural. Names of the last tenant and project must be shown only while the identity still holds that membership (RF-6.5, P3): `MembershipExistsPort` answers from `access` under RLS and runtime authorisation, so a lost membership yields no name without this feature reading a tenant table (P12). Paging on the server with a configured size (P13) keeps the client free of numbers and gives the "last page" rule in one place.
*Discarded.* Client-side filtering of a full list — a full list in memory and a client rule. A join from the session row to tenant tables — a cross-context read outside RLS. Caching names on the session row — stale after a rename or a lost membership.
*Covers.* RF-6.1, RF-6.4, RF-6.5, RF-6.7, RF-6.8, RNF-5, RNF-9, LC-15, LC-16, LC-28.

**D6 · Individual revocation is one conditional statement keyed by session id *and* acting identity *and* "not the acting session"; every miss is one 404 code.**
*Why.* RF-7.2 and RF-7.6: a foreign, unknown or current session must be refused without revealing which; one predicate in one statement gives atomicity (no read-then-write race with expiry) and one outcome for all three misses. The current session is excluded in the predicate rather than in the client so that no crafted request can revoke the session that carries it (D-c). An already-ended own session is a hit that changes nothing and answers done (RF-7.4), so a stale list never produces an error.
*Discarded.* Distinct 403 (foreign) and 404 (unknown) — enumeration of session ids. Loading then checking in the use case — a race window and more code for the same guarantee. Allowing the current row — contradicts R38 and D-c.
*Covers.* RF-7.1, RF-7.2, RF-7.4, RF-7.6, RF-7.7, RNF-5, RNF-6, LC-18, LC-19.

**D7 · "Sign out everywhere" bumps the security version and eagerly ends every session in one transaction, the acting one included, under an idempotency key.**
*Why.* Decision Q2: every session ends, this one too. The version bump **[004]** is the lazy guarantee for any credential already issued (**[006 D9]**); `revokeAll(except = none)` is the eager path so the list and the audit log are truthful at once and the other devices meet R34 at their next request; voiding open login transactions closes the one remaining door (a half-finished sign-in elsewhere). One transaction makes the change all-or-nothing (RNF-2, RF-11.4). The idempotency key lets a retry after a lost answer return the recorded outcome instead of failing on an already-ended session, and the cleared cookie on the response gives the acting browser the same exit as a sign-out.
*Discarded.* Bump only, lazy ending — the list would show live sessions that are dead. `except = acting` — Q2's rejected option (the replacement scope of A-39, deliberately different here). A notification — Q2 rejected it; the person acted themselves.
*Covers.* RF-8.3, RF-8.4, RF-8.6, RF-8.7, RNF-2, LC-23, LC-24, LC-26.

**D8 · One `SignOutCoordinator` in `shared/security` owns the order guard → abort → request → clear for every entry.**
*Why.* RF-2.2, RF-3 and RF-4 describe one sequence with several effects that must happen in a fixed order (the guard before anything, the abort before the request, the clear only after a definitive answer). One coordinator used by the shell, the public screens, the invitation screens and R34 cannot be reordered by a feature, mirrors the guarded switch's fixed order (F-IDN-18) and reuses its step-2 primitives and the boundary's clear primitives **[006 D10]** instead of copying them.
*Discarded.* Per-entry sequences — drift (one screen clears before the answer, another forgets the guard). A sign-out as a special case of the tenant switch — the switch's steps 3–5 (fetch grants, load data) do not apply, and routing to the sign-in screen is not a scope change.
*Covers.* RF-2.2, RF-2.3, RF-3.1–3.7, RF-4.1–4.5, RF-9.1–9.4, RNF-10, LC-1–LC-7, LC-12, LC-13.

**D9 · The dirty guard is the guarded switch's own hook, invoked, not re-implemented; a guard save runs under the draft's stamped scope.**
*Why.* Decision Q4 names the guard of F-IDN-18; invoking its published hook keeps one dialog (N24) and one rule. The Zustand drafts carry `{identityId, tenantId, projectId, baseETag}` **[006 D10]**, so "saved under the context in which it was written" (RF-3.2, RF-3.7) is the draft's stamp, never the header selector — the same rule the switch applies (FR-IDN-131).
*Discarded.* A sign-out-specific confirmation ("You have unsaved changes — leave anyway?") — a second dialog with weaker options than the guard. Auto-saving before sign-out — RF-9.4 of 006 forbids saving without an explicit action.
*Covers.* RF-3.1–3.4, RF-3.7, RNF-10, LC-1–LC-4.

**D10 · An unconfirmed sign-out, revocation or "Sign out everywhere" renders the retry / offline status of 006 in a sign-out variant; nothing is cleared.**
*Why.* Decision Q5 (S-41): the outcome is indeterminate and the product must say so without claiming either state. `TransportFailureStatus` **[006 D13]** already exists for exactly this class of outcome; a variant with the statement "sign-out not complete" and the action "Retry sign out" (one request per activation) keeps one status component. Because step (2) disconnected streams before the request, the coordinator lets them reconnect on the next successful request under the still-valid session (RF-4.3).
*Discarded.* Clear locally and warn — Q5's rejected option. Automatic retry in the background — the person would walk away from a browser that is still trying.
*Covers.* RF-4.1–4.5, RF-7.4, RF-8.6, RNF-1, RNF-6, LC-6, LC-7, LC-20, LC-24.

**D11 · Other tabs learn of a sign-out through the server's definitive answer, exactly as for any ended session; no cross-tab messaging exists.**
*Why.* Decision Q6: other tabs treat it as an ended session — mask, R34, drafts kept in memory. That is precisely what `ProtectedBoundary` **[006 D10]** does on `auth.session.invalid`, so nothing new is needed and the behaviour is identical whether the session ended by deadline, revocation or sign-out (R34 never says why, **[006 RF-7.3]**). Only the signing-out tab knows it signed out, and it routes to the sign-in screen instead (`SessionState.SIGNED_OUT`).
*Discarded.* A `BroadcastChannel` message routing every tab to the sign-in screen — Q6's rejected option, and a client-side signal that could be forged by any script in the origin. Polling the session status from each tab — background traffic for nothing the next request does not tell.
*Covers.* RF-5.1–5.4, LC-10, LC-11.

**D12 · "Sign out safely" on R34 asks the server to end what remains and does not wait.**
*Why.* Spec D-d: the dialog opens only on a definitive answer that the session ended, so the client already knows there is nothing live to protect; the request is sent so an ended session's family row is closed if it was not, and both possible answers (204, 401) change nothing the client has not already done. Waiting would make a dead session's exit depend on the network.
*Discarded.* Treating it as an ordinary sign-out with Q5's wait — a person could be stuck on a dialog for a session that is already gone. Not calling the server at all — leaves a family row that only the sweep **[006 D15]** would close.
*Covers.* RF-3.5, RF-9.4, LC-13.

**D13 · No notification for any action of this feature; the audit log records each with its entry, target or count.**
*Why.* Decisions Q2 and Q8: the person acted themselves, and the sessions list is where they review. The five audit events name what the session-ended entries **[006]** cannot: which entry signed out, which session revoked which, how many ended at once, and each refused attempt (RF-11.1). They are appended in the effect's transaction so the record and the effect cannot disagree.
*Discarded.* A "signed out everywhere" email — rejected by Q2. Auditing only through `SESSION_ENDED` — loses entry, target and refusals.
*Covers.* RF-11.1, RF-11.5, LC-17, LC-19, LC-23.

**D14 · No dedicated mobile pages; the table becomes one record per session, and every text comes from the feature-scoped i18n bundles [002 D16].**
*Why.* The Sessions area follows the D01 responsive policy with the D02 `422:3292` record form (S-02, S-28); FR-X-055 requires AA reflow at every width; a table-to-records conversion keeps every cell and the action (RNF-15); `SecurityStatements` shared across widths keeps the three statements present. The server sends enums and timestamps; the client labels and formats them, so a language switch changes display only (RF-12.4).
*Discarded.* Separate mobile components — drift. Server-rendered labels — ties language to the request.
*Covers.* RF-12.1–12.4, RNF-13–15, LC-14, LC-29.

**D15 · The shell "Sign out" is a composition-level element in `app/shell`, rendered on every authenticated route and following no grant.**
*Why.* Decision Q3 (S-07): the action must exist on every app-shell screen, desktop and mobile; the shell is app-level composition (P11), the one place that wraps every authenticated route, and the M04 menu is its mobile form **[F-X-07]**. Sign-out is not a destination and follows no grant, so it is rendered unconditionally (RF-1.3) while the M04 destinations keep following server-derived grants (RF-12.2).
*Discarded.* Placing the action in each feature's page — forgotten on some. Waiting for the designed user menu — S-07's recommendation, but Q3 decided to build the action now; when the menu is drawn the action moves into it unchanged.
*Covers.* RF-1.2, RF-1.3, RF-1.5, RF-12.2.

**D16 · Errors are RFC 9457 problem details with safe codes and a correlation id [001 D12].**
*Why.* F-X-03; the one new code `auth.session.not-revocable` says only what the client must do (show one generic problem); the two reused codes **[006]** keep their meaning; the correlation id joins the audit record to the person's report.
*Discarded.* Ad-hoc error bodies; distinct codes per refusal cause (D6).
*Covers.* RF-7.2, RF-11.2, RF-11.3.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**) | Pure domain: `ClientContextClassifier` vocabulary and precedence, `Session.recordScope` no-op and change, `endByHolder` cause restriction, `SessionView` mapping (nearer deadline, `IDENTITY_SESSION`, no forbidden field). |
| BE integration | Spring Boot test + **real PostgreSQL** (area A; Testcontainers **test-only**) | Use cases through real adapters: conditional revocation predicate, sign-out idempotency, sign-out-everywhere transaction (bump, revokeAll, voided transactions, outcome record), list paging and ordering, scope hook write-on-change, audit rows, cookie clearing, configuration binding. |
| Module boundary | ArchUnit (**test-only**) | The application module imports no Spring or persistence class and no project, grant, approval, run or intent package; `access` is reached only through the two ports. |
| API contract | Spring MockMvc (**test-only**) | Status codes, the three problem codes, correlation id, cleared-cookie attributes on 204, CSRF refusal, absence of any credential, raw agent, deadline pair, version or cause in bodies; list endpoint shape. |
| Concurrency | JUnit 5 with real PostgreSQL | Revocation racing expiry, two concurrent sign-outs of one session, sign-out-everywhere racing a refresh **[006]**, two devices revoking each other. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for an absent, zero or unbounded page size. |
| FE unit | Vitest + Testing Library (**test-only**) | Coordinator order and outcomes per entry, guard branches, indeterminate variant with nothing cleared, `SIGNED_OUT` routing, table cells and actions, "Session revoked" text, pagination text state, confirm dialog statements and key reuse, nothing in storage. |
| FE e2e + accessibility | Playwright + axe (**test-only**) | Journeys in EN / ES / FR at desktop, tablet and mobile widths against the BE test doubles; multi-tab sign-out; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS, iOS) — versions pinned in the release-support matrix | Row action names, dialog focus entry and return, live announcements without focus theft, menu item labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RF-9.1–9.3, LC-12 | FE e2e, BE integration | `signOut.availableOnN03N20M03N29R34`; `signOut.recordsEntryValue` | 3 |
| RF-1.2, RF-12.2 | FE unit, e2e | `shell.signOutOnEveryAuthenticatedRouteDesktopAndM04Menu` | 3 |
| RF-1.3 | BE integration, FE unit | `signOut.allowedForGatedOwnerGrantlessMemberSuspendedTenant`; `shell.signOutIgnoresGrants` | 3 |
| RF-1.4, RF-9.5 | FE unit | `limitedStateScreens.callAbandonNeverSignOut` | 3 |
| RF-1.5, RF-12.3, RNF-13 | FE e2e + axe, manual | `a11y.signOutEntries.{en,es,fr}.{desktop,tablet,mobile}` | 12 |
| RF-2.1, RNF-2, RNF-3, LC-5 | BE integration | `signOut.endsSessionAndFamilyAtomically`; `signOut.nextRequestRefused` | 4 |
| RF-2.2, RF-11.4 | API contract, FE unit | `signOut.204ClearsRefreshCookie`; `coordinator.clearsHolderDraftsQueriesEpochAndRoutes` | 4 |
| RF-2.3 | FE e2e | `afterSignOut.signInFormEmpty` | 4 |
| RF-2.4, RF-5.3 | BE integration | `signOut.leavesOtherSessionsOfIdentityUntouched` | 4, 7 |
| RF-2.5, RF-8.7, LC-26 | BE integration | `signOut.touchesOnlySessionTables`; `signOutEverywhere.leavesRequirementFactorCodesMembershipsCase` | 4, 9 |
| RF-2.6, RF-10.1, RNF-11, LC-22 | BE integration | `run.continuesAcrossSignOutRevocationAndEverywhere` | 10 |
| RF-2.7, RF-4.4, LC-7, LC-8 | API contract, FE unit | `signOut.alreadyEndedAnswers204`; `coordinator.treats401InvalidAsDefinitive` | 4, 6 |
| RF-2.8, LC-9 | API contract, FE unit | `signOut.csrfRefusedNoChange`; `coordinator.csrfClearsNothing` | 6 |
| RF-3.1, LC-1 | FE unit | `coordinator.runsGuardFirstStayReturns` | 5 |
| RF-3.2, LC-2 | FE unit, e2e | `coordinator.saveUnderDraftStampThenSignsOut` | 5 |
| RF-3.3, LC-3, LC-4 | FE unit | `coordinator.failedRefusedOrIndeterminateSaveCancelsSignOut` | 5 |
| RF-3.4 | FE unit | `coordinator.discardPurgesThenSignsOut` | 5 |
| RF-3.5, RF-9.4, LC-13 | FE unit | `coordinator.fromEndedSessionDiscardsWithoutGuardOrWait`; `publicScreens.noGuard` | 5 |
| RF-3.6, RNF-10 | FE unit, BE integration | `coordinator.abortsAndDisconnectsBeforeRequest`; `lostAck.lookedUpAfterNextSignIn` | 4 |
| RF-3.7 | FE unit | `coordinator.neverRetargetsSaveOrDiscardsSilently` | 5 |
| RF-4.1, RF-4.2, RNF-1, RNF-6, LC-6 | FE unit | `coordinator.indeterminateShowsIncompleteStatusClearsNothing` | 6 |
| RF-4.3 | FE unit | `coordinator.afterIndeterminateStreamsReconnectOnNextSuccess` | 6 |
| RF-4.5 | FE unit | `coordinator.savedWorkNotReaskedAfterIndeterminate` | 5, 6 |
| RF-5.1, RF-5.2, LC-10, LC-11 | FE e2e | `multiTab.signOutInOneOthersMeetDialogWithDrafts`; `multiTab.reauthRestoresPreservedContext` | 7 |
| RF-5.4 | FE unit | `coordinator.signingOutTabNeverShowsDialog` | 7 |
| RF-6.1, LC-15 | BE integration | `list.allLiveSessionsAcrossTenantsNewestFirstPaged` | 8 |
| RF-6.2, RF-6.3, RNF-12 | BE unit, API contract | `classifier.vocabularyAndPrecedence`; `sessionView.cellsAndNoForbiddenField`; `list.bodyHasNoAgentIpLocationCredentialVersionOrCause` | 8 |
| RF-6.4, RNF-5 | BE integration, API contract | `list.identityScopedOnly`; `noAdminSessionEndpointExists` | 8 |
| RF-6.5, LC-16 | BE integration | `list.lostMembershipShowsIdentitySession` | 8 |
| RF-6.6, RF-10.3 | FE unit | `statements.threeSessionStatementsPresent`; `confirmDialog.statesRunsContinue` | 8, 10 |
| RF-6.7, RNF-9, LC-28 | Startup, BE integration, FE unit | `sessionPolicy.startupFailsOnAbsentOrUnboundedPageSize`; `list.pageBeyondLastReturnsLast`; `pagination.textStateWhenUnavailable` | 8 |
| RF-6.8 | FE unit | `sessionsArea.fetchesOnOpenAndAfterActionsNeverInterval` | 8 |
| RF-6.9, LC-27 | FE e2e | `list.viewerSessionEndsShowsDialogNeverServed` | 8 |
| RF-6.10, RF-7.6 | FE unit, API contract | `table.currentRowHasNoRevoke`; `revoke.currentSessionNotRevocable` | 8, 9 |
| RF-7.1, RF-7.3, LC-17 | BE integration, FE unit | `revoke.endsAtOnceNoDialogNoStepUp`; `table.rowShowsSessionRevokedText` | 9 |
| RF-7.2, LC-19 | API contract, BE integration | `revoke.foreignUnknownCurrentOneCode`; `revoke.refusalAudited` | 9 |
| RF-7.4, LC-18, LC-20 | BE integration, FE unit | `revoke.alreadyEndedAnswersDone`; `table.indeterminateShowsStatusRepeatable` | 9 |
| RF-7.5, LC-21 | FE e2e, BE integration | `revokedDevice.meetsDialogDraftKept`; `revoke.sendsNoNotification` | 7, 9 |
| RF-7.7 | ArchUnit, API contract | `revoke.onlyTwoCallersNoTenantScopedEndpoint` | 9 |
| RF-8.1, RF-8.2, RF-8.5 | FE unit | `signOutEverywhere.buttonAndConfirmStatementsNoCredentialField`; `settingsRoute.holdsNoDraft` | 9 |
| RF-8.3, RF-8.4, RNF-2, LC-23 | BE integration, FE e2e | `signOutEverywhere.bumpsRevokesAllVoidsTransactionsOneTransaction`; `threeDevices.allEndCurrentToSignInOthersMeetDialog` | 9 |
| RF-8.6, LC-24 | BE integration, FE unit | `signOutEverywhere.recordedOutcomeForRepeatedKey`; `confirmDialog.reusesKeyOnRetry`; `coordinator.everywhereIndeterminateClearsNothing` | 6, 9 |
| RF-8.8 | API contract | `signOutEverywhere.actsOnAssertedIdentityOnly` | 9 |
| RF-10.2, LC-25 | BE integration, FE unit | `pendingRun.lookedUpNeverResubmittedAfterSignOutEverywhere` | 10 |
| RF-11.1, RF-11.2, RF-11.3 | BE integration, API contract | `audit.entryPerEvent`; `audit.containsNoCredentialOrAgentString`; `logs.containNoCredential`; `problems.threeSafeCodesWithCorrelationId` | 11 |
| RF-11.5 | BE integration | `feature.neverCallsNotificationPort` | 11 |
| RF-12.1, RNF-15, LC-14 | FE e2e + axe, manual | `a11y.sessionsArea.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 12 |
| RF-12.4, RNF-14, LC-29 | FE e2e | `languageSwitch.changesDisplayOnlyRecordsUnchanged` | 12 |
| RNF-4 | BE integration | `feature.grantsNoAuthorityAndNoFactorBypass` | 2 |
| RNF-7 | FE unit | `storage.containsNothingFromFeature` | 2 |
| RNF-8 | FE unit | `expiry.displayedFromServerNeverComputed` | 8 |
| RNF-16 | all of the above | Areas A and F: server-side revocation, multi-tab, identity-scoped list, everywhere across devices, runs surviving | 13 |
| LC-30 | BE integration | `restrictedIdentity.signOutOfEndedSessionAnswersDone` | 4 |
| Concurrency | Concurrency | `revoke.racingExpiryOneOutcome`; `signOut.twoConcurrentOneEnd`; `everywhere.racingRefreshRefusesRefresh`; `twoDevices.mutualRevokeBothEnd` | 4, 9 |
| US-11 | all adversarial cases above | signed-out credential reuse, session enumeration, foreign revocation, foreign everywhere, silent failure, Run cancellation suites | 1 |

### 5.3 Not tested here
Sign-in, the challenge, landing and the limited-state abandon (plan 001), signup (002), enrollment and the Security Settings page shell (003), reset and forced change (004), recovery (005), session issue, assertion, refresh, R34, the boundary and the retry status component (006), invitation screens (008), the tenant chooser and suspended screens (013, 014), the guarded switch and its dirty guard dialog (016), admission and run-scoped authority (F-RUN). Their stubs return fixed answers; `EndSessionUseCase`, `CurrentSessionPort`, `AccessCredentialHolder`, `ProtectedBoundary` and `TransportFailureStatus` are exercised through this feature's own calls.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q1, Q2, Q7 and Q8 were recorded in Figma on 27 October 2026 as **A-51**, **S-43** and **S-44**; this plan implements each as decided (list content D3 / D4 / D5, revocation D6, sign-out everywhere D7, confirmation in `ConfirmDialog`). The items below remain open.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-11 configuration values | `sessionListPageSize` joins `SessionPolicyProperties` **[006]**; value from the catalogue. | Rename or regroup the property; no use case changes. |
| A-27 identity statuses | A restricted or erasing identity's sessions end under **[006]**; `SignOutUseCase` on such a session answers done (LC-30). | Change the predicate in **[006]** only. |
| A-31 re-authentication (closed by 006) | Other tabs re-authenticate from R34 after a sign-out exactly as after expiry (D11). | — |
| A-35 persistence tooling | Columns, index and outcome table against the placeholder API and migration tool. | Rename base types and migration format; no use case changes. |
| A-39 revocation scope | `revokeAll(identity, except = none, cause)` used here as decided for Q2; run-scoped authority never referenced. | Extend the port's contract; D7 keeps `except = none`. |
| A-49 human activity | `ListSessionsQuery` is a deliberate read; `RecordSessionScope` is not a request and counts nothing. | Reclassify the list read only. |
| A-50 reuse detection | Its notification names "review the sessions list"; the list of this plan is that surface. | Adjust the template's wording only. |
| S-02 / S-28 responsive coverage | Sessions area, confirmation and status follow the D01 responsive policy (`414:2645`) with the one-record-per-session form of D02 `422:3292` (D14). | None: the components are unchanged. |
| S-07 app-shell sign-out | `SignOutAction` in the footer identity block and the M04 menu (D15). | Move the action into the designed user menu; the coordinator is unchanged. |
| S-41 retry / offline status | `TransportFailureStatus` **[006]** variants `signOutIncomplete` / `signOutEverywhereIncomplete` with the D02 copy (`424:3581`). | None; keep the variants. |
| S-43 "Sign out everywhere" (decided, drawn) | Button beside the list (`427:4040`) and `ConfirmDialog` reusing `shared/ui` `Dialog` with the drawn copy (D02 `422:3292`). | None. |
| S-44 table rows (decided) | Cells as D3–D5; "Session revoked" row state as text. | Replace labels and layout when drawn. |

No new register item is raised by this plan: the user-agent classification is an in-house rule within P10 (D3), no cross-tab mechanism is introduced (D11), and no notification is sent (D13).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.5) | `SignOutAction`, N03 / N20 / M03 / N29 / R34 entries, `AbandonChallengeUseCase` **[001]** for R08 | D8, D15 | RF-1 rows |
| RF-2 (2.1–2.8) | `SignOutUseCase`, cookie adapter, `SignOutCoordinator`, CSRF configuration | D1, D2, D8, D16 | RF-2 rows |
| RF-3 (3.1–3.7) | `SignOutCoordinator` step (1)–(2), dirty guard hook, stamped drafts | D8, D9, D12 | RF-3 rows |
| RF-4 (4.1–4.5) | `SignOutCoordinator` step (4), `TransportFailureStatus` variants | D2, D10 | RF-4 rows |
| RF-5 (5.1–5.4) | `ProtectedBoundary` **[006]**, `SessionState.SIGNED_OUT` | D11 | RF-5 rows |
| RF-6 (6.1–6.10) | `ListSessionsQuery`, `SessionView`, `ClientContextClassifier`, `RecordSessionScope`, `MembershipExistsPort`, `SessionsArea`, `SessionsTable`, `Pagination`, `SecurityStatements` | D3, D4, D5 | RF-6 rows |
| RF-7 (7.1–7.7) | `RevokeSessionUseCase`, conditional predicate, `auth.session.not-revocable`, row states | D1, D6, D16 | RF-7 rows |
| RF-8 (8.1–8.8) | `SignOutEverywhereUseCase`, outcome record, `SignOutEverywhereAction`, `ConfirmDialog`, coordinator | D1, D7, D10 | RF-8 rows |
| RF-9 (9.1–9.5) | entry values of `SignOutUseCase`, `fromEndedSession()`, **[004]** abandon | D8, D12 | RF-9 rows |
| RF-10 (10.1–10.3) | absence of any run touch point, launch status lookup, statements | D1, D13 | RF-10 rows |
| RF-11 (11.1–11.5) | audit adapter events, classifier discard, problem catalogue, no notification port | D3, D13, D16 | RF-11 rows |
| RF-12 (12.1–12.4) | record layout, M04 menu item, `shared/ui`, i18n bundles | D14, D15 | RF-12 rows |
| RNF-1 | definitive-answer rule of the coordinator | D2, D10 | RF-4.1 row |
| RNF-2 | one transaction per use case; cleared cookie on commit | D1, D2, D7 | RF-2.1, RF-8.3 rows |
| RNF-3 | `AssertSessionUseCase` **[006]** after `end` | D1 | RF-2.1 row |
| RNF-4 | asserted identity; no bypass | — | `feature.grantsNoAuthorityAndNoFactorBypass` |
| RNF-5 | identity-keyed query and predicate | D5, D6 | RF-6.4, RF-7.2 rows |
| RNF-6 | conditional predicate; nothing cleared on indeterminate; CSRF no-change | D2, D6, D10 | RF-2.8, RF-4.1, RF-7.4 rows |
| RNF-7 | no storage write | — | `storage.containsNothingFromFeature` |
| RNF-8 | `expiresAt` from the server | D5 | `expiry.displayedFromServerNeverComputed` |
| RNF-9 | `sessionListPageSize` validation | D5 | startup |
| RNF-10 | guard, stamped save, lookup | D8, D9 | RF-3 rows |
| RNF-11 | no run touch point | D1 | RF-2.6 row |
| RNF-12 | classifier enums, `SessionView` | D3 | RF-6.2 row |
| RNF-13, RNF-15 | `shared/ui`, record layout | D14 | RF-1.5, RF-12.1 rows |
| RNF-14 | i18n bundles | D14 | `languageSwitch` |
| RNF-16 | integration, concurrency and e2e suites | D1, D7, D11 | areas A / F row |
| LC-1 … LC-30 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30) |

Self-check: every RF criterion (71), every RNF (16) and every limit case (30) of `specLogoutSessionManagement.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
