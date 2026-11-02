# Tasks 007 — Logout and session management

**Feature** Sign-out, the sessions list, individual revocation and "Sign out everywhere" · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specLogoutSessionManagement.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planLogoutSessionManagement.md` v0.1 (modules §2, decisions D1–D16, tests §5, register assumptions §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** … **[006]** extend an element that an earlier plan builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. This feature consumes plan 006's session service (`Session`, `EndSessionUseCase`, `AssertSessionUseCase`, `SessionAuthenticationFilter`, `AccessCredentialHolder`, `ProtectedBoundary`, `TransportFailureStatus`) — those elements are prerequisites, not tasks of this file. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 006's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-04 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · List page size in the session policy [006]** — `SessionPolicy` gains `sessionListPageSize`; `SessionPolicyProperties` binds it `@Validated` as present, positive and bounded above by a configured ceiling; no default anywhere. *(plan §2.1 `SessionPolicy`, `SessionPolicyProperties`, D5; RF-6.7, RNF-9)*
  **Done when:** `sessionPolicy.startupFailsOnAbsentOrUnboundedPageSize` passes: the context starts with the value set and fails to start, naming the property, when it is absent, zero, negative or above the ceiling.

- [ ] **T-02 · Migration: `identity_session` columns and index [006]** — add `browser_family`, `platform_family` (enum text, not null, written at issue), `last_tenant_id`, `last_project_id`, `last_scope_at` (nullable); add the index `(identity_id, state, created_at desc)`; `end_cause` accepts the value `SIGN_OUT_EVERYWHERE`. Existing rows are back-filled with `OTHER / OTHER` and null scope. *(plan §2.1 Persistence, D3, D4, D7; RF-6.1, RF-6.2, RF-8.3)* — after T-01
  **Done when:** the migration applies on an empty database and on one holding plan 006 sessions; back-filled rows read `OTHER / OTHER` with null scope; a row with `end_cause = 'SIGN_OUT_EVERYWHERE'` inserts; the planner uses the new index for the list query's predicate and order.

- [ ] **T-03 · Migration: `sign_out_everywhere_outcome`** — table (`idempotency_key` unique, `identity_id`, `recorded_at`) for the recorded outcome of a repeated "Sign out everywhere" submission. *(plan §2.1 Persistence, D7; RF-8.6)* — after T-02
  **Done when:** the migration applies; a second row with the same key is rejected; `identity_id` cascades on identity delete.

- [ ] **T-04 · Problem-code catalogue** — register `auth.session.not-revocable` (404; one code for a foreign, unknown or current session) beside the reused `auth.session.invalid` (401) and `auth.csrf.invalid` (403) **[006]**; every problem is RFC 9457 with a correlation id and no cause, deadline, version or credential field. *(plan §2.1 `SessionController`, D6, D16; RF-7.2, RF-11.3)* — after T-01
  **Done when:** the catalogue test lists exactly the three codes for this feature with their statuses, and a sample problem of each serialises with `type`, `title`, `status`, `correlationId` and nothing else.

## B · Domain (pure, no Spring)

- [ ] **T-05 · `ClientContextClassifier`** — pure rule `(userAgentString) → ClientContext{browserFamily, platformFamily}` over the fixed vocabulary browser `CHROME, FIREFOX, SAFARI, EDGE, OTHER` and platform `WINDOWS, MACOS, LINUX, IOS, ANDROID, OTHER`, by substring precedence (Edge before Chrome, Chrome before Safari; iOS before macOS; Android before Linux); null, empty or unknown input yields `OTHER / OTHER`; the string is never retained. *(plan §2.1 `ClientContextClassifier`, D3; RF-6.2, RF-6.3, RF-11.2, RNF-12, LC-15)* — after T-01
  **Done when:** `classifier.vocabularyAndPrecedence` passes with a table of at least twelve agent strings (one per family plus the four precedence collisions and three unknowns) and the result type exposes no field holding the input.

- [ ] **T-06 · `Session` extension [006]** — `clientContext` narrowed to `{browserFamily, platformFamily}`; new `lastScope {tenantId, projectId, at}` (nullable); `recordScope(tenantId, projectId, now)` returns `false` and changes nothing when the scope is unchanged, otherwise sets the three values — never `lastActivityAt`, `idleDeadline` or `absoluteDeadline`; `endByHolder(cause, now)` delegates to `end` **[006]** and accepts only `LOGOUT`, `REVOKED`, `SIGN_OUT_EVERYWHERE` (the last is a new cause value). *(plan §2.1 `Session`, D1, D4; RF-2.1, RF-6.2, RF-7.1, RF-8.3, RNF-2)* — after T-05
  **Done when:** unit tests show `recordScope` is a no-op for an equal scope and leaves deadlines and activity untouched on a change, `endByHolder` with `IDLE` throws, and `end(SIGN_OUT_EVERYWHERE)` moves the state to `ENDED` with that cause.

- [ ] **T-07 · `SessionView` read model** — session id, `browserFamily`, `platformFamily`, `createdAt`, `context` = `Scope(tenantName, projectName) | IDENTITY_SESSION`, `expiresAt = min(idleDeadline, absoluteDeadline)`, `isCurrent`, `action = CURRENT | REVOKABLE` (`REVOKABLE` only when not current); no field for a deadline pair, version, family state, network key or agent string. *(plan §2.1 `SessionView`, D3, D5; RF-6.2, RF-6.3, RF-6.6, RF-6.10, RNF-8, RNF-12)* — after T-06
  **Done when:** `sessionView.cellsAndNoForbiddenField` passes: a reflective check lists exactly the eight fields above, `expiresAt` equals the nearer deadline in both orderings, and the current session maps to `CURRENT`.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-16…T-19); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-08 · Port interfaces** — new: `SessionScopePort` (published to `access`: `recordScope(sessionId, tenantId, projectId)`), `MembershipExistsPort` (published by `access`, consumed here: `activeMembership(identityId, tenantId) → Optional<Names>`), `ClientContextPort` (the transport's user-agent header, read once at issue), `SignOutEverywhereOutcomeStore` (recorded outcome by idempotency key); `LoginTransactionRepository` **[001]** gains `voidAllOpen(identityId)`; `SessionRepository` **[006]** gains `findActivePage(identityId, page, size)` and `selectOwnedForUpdate(sessionId, identityId, excludingSessionId)`. Reused: `EndSessionUseCase`, `CurrentSessionPort`, `RefreshFamilyRepository` **[006/003]**; `IdentityRepository`, `AuditPort`, `Clock` **[001]**. Not used: `SecurityNotificationPort`, `TransactionalEmailPort`. *(plan §2.1 Ports, D1, D5, D6, D13; RF-6.4, RF-6.5, RF-7.7, RF-11.5, RNF-5)* — after T-07
  **Done when:** the application module compiles with no dependency on Spring or persistence classes and the ArchUnit boundary test shows no import of a project, grant, approval, run or intent package and no reference to a notification or email port from this feature's use cases.

- [ ] **T-09 · `IssueSessionUseCase` [006] — client context at issue** — the use case reads the agent string from `ClientContextPort`, classifies it with `ClientContextClassifier`, stores the two enums on the new `Session`, and discards the string before any persistence, audit or log call. *(plan §2.1 `IssueSessionUseCase`, D3; RF-6.2, RF-6.3)* — after T-08
  **Done when:** a unit test with a recording audit double and a recording repository double shows the stored session carries the classified families and that no call received the agent string.

- [ ] **T-10 · `RecordSessionScope` and the `access` hook** — `RecordSessionScope` implements `SessionScopePort`: loads the session, calls `Session.recordScope`, persists only when it returned `true`; the C10 derivation of `access` **[001/003/006]** calls the port after a successful derivation for a tenant (and optional project) and never on a failed or denied one; the call is not a request and is classified nothing by `ActivityClass` **[006]**. *(plan §2.1 `RecordSessionScope`, `access.domain`, D4; RF-6.2, RF-6.5)* — after T-08
  **Done when:** unit tests show one persist for a changed scope, zero persists for ten identical derivations, zero calls after a denied derivation, and no change to `lastActivityAt`.

- [ ] **T-11 · `MembershipExistsPort` implementation in `access`** — `activeMembership(identityId, tenantId)` runs under RLS and runtime authorisation **[F-X-01]** and returns the tenant display name (and the project name when a project id is supplied) only for an `active` membership of that identity; a suspended or revoked membership, an unknown tenant or another identity's membership yields empty. *(plan §2.1 `access.domain`, D5; RF-6.5, RNF-5, LC-16)* — after T-08
  **Done when:** an integration test over active, suspended, revoked, foreign and unknown cases returns names for the active case only, with `SET LOCAL` context verified inside the transaction.

- [ ] **T-12 · `SignOutUseCase`** — input: the asserted session and the `entry` (`SHELL, N03, N20, M03, N29, R34`); `ACTIVE` → `endByHolder(LOGOUT)` and end the `RefreshFamily` **[006]** in one transaction, audit `SIGNED_OUT{entry}`; already ended → `ALREADY_ENDED`, no new audit; both outcomes are definitive. Reads no tenant, grant or run table; calls no notification port. *(plan §2.1 `SignOutUseCase`, D1, D13; RF-1.1, RF-1.3, RF-2.1, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-9.1, RF-9.2, RF-9.3, RF-9.4, RF-10.1, RF-11.1, RF-11.4, RNF-2, LC-5, LC-8, LC-30)* — after T-08
  **Done when:** `signOut.recordsEntryValue` passes for all six entry values; a unit test shows the session and family both `ENDED` with cause `LOGOUT` after one call, `ALREADY_ENDED` on a second call with no further audit entry, and the recording doubles for tenant, run and notification ports received zero calls.

- [ ] **T-13 · `RevokeSessionUseCase`** — input: acting identity and session, target id; `selectOwnedForUpdate(target, identity, excluding = acting)`: no row → `NOT_REVOCABLE` and audit `REVOCATION_REFUSED{target}`; `ACTIVE` row → `endByHolder(REVOKED)` + family end, audit `SESSION_REVOKED_BY_HOLDER{target}`; ended row → `DONE`, audit `SESSION_REVOKED_BY_HOLDER{target, alreadyEnded}`. No confirmation, no step-up, no notification. *(plan §2.1 `RevokeSessionUseCase`, D6, D13; RF-7.1, RF-7.2, RF-7.4, RF-7.5, RF-7.6, RF-7.7, RF-11.1, RNF-5, RNF-6, LC-17, LC-18, LC-19)* — after T-08
  **Done when:** unit tests show the four outcomes (own active, own ended, foreign, current) with the three audit shapes, and that foreign, unknown and current produce the same `NOT_REVOCABLE` value.

- [ ] **T-14 · `SignOutEverywhereUseCase`** — input: acting identity, idempotency key; a repeated key returns the recorded outcome; otherwise one transaction: `GlobalIdentity.bumpSecurityVersion()` **[004]**, `EndSessionUseCase.revokeAll(identityId, except = none, cause = SIGN_OUT_EVERYWHERE)` **[006]**, `LoginTransactionRepository.voidAllOpen(identityId)` **[001]**, audit `SIGNED_OUT_EVERYWHERE{count}`, record the outcome. No notification. *(plan §2.1 `SignOutEverywhereUseCase`, D7, D13; RF-8.3, RF-8.4, RF-8.6, RF-8.7, RF-8.8, RF-11.1, RF-11.4, RF-11.5, RNF-2, LC-23, LC-26)* — after T-12
  **Done when:** a unit test with three sessions, one open login transaction and a pending forced-change requirement shows: version advanced by one, all three sessions and families `ENDED(SIGN_OUT_EVERYWHERE)` including the acting one, the transaction voided, the requirement untouched, one audit entry with `count = 3`, and a second call with the same key returning the recorded outcome without a second bump.

- [ ] **T-15 · `ListSessionsQuery`** — input: acting identity and session, page; `findActivePage(identity, page, sessionListPageSize)` ordered `createdAt desc`; per row build `SessionView` with `isCurrent` from `CurrentSessionPort` **[003/006]**, `context` from `MembershipExistsPort` (empty or null scope → `IDENTITY_SESSION`), `expiresAt` the nearer deadline; a page beyond the last returns the last page with its number; the read is a deliberate navigation (spec D-e). *(plan §2.1 `ListSessionsQuery`, D5; RF-6.1, RF-6.2, RF-6.4, RF-6.5, RF-6.7, RF-6.8, RNF-5, LC-15, LC-16, LC-28)* — after T-11, T-07
  **Done when:** unit tests with five sessions across three tenants and a page size of two show three pages newest first, the current row flagged once, "Identity session" for a null scope and for a lost membership, and page 9 returning page 3.

## D · Infrastructure adapters

- [ ] **T-16 · Persistence adapters** — implementations (access API per A-35) of the new repository methods: `findActivePage` using the T-02 index, `selectOwnedForUpdate` as one statement `WHERE id = ? AND identity_id = ? AND id <> ? FOR UPDATE`, the scope column writes, and `SignOutEverywhereOutcomeStore` over T-03. *(plan §2.1 Persistence, D5, D6, D7; RF-6.1, RF-7.2, RF-8.6)* — after T-02, T-03, T-08
  **Done when:** integration tests with real PostgreSQL show the page query returns rows in `created_at desc` with the expected count per page, the ownership select returns no row for a foreign id, an unknown id and the acting id, and the outcome store rejects a duplicate key.

- [ ] **T-17 · Cookie clearing and forgery defence on the mutations** — the responses of `DELETE session` and `POST sign-out-everywhere` carry the refresh cookie **[006]** expired with the same name, path and attributes, written only after the transaction committed; CSRF applies to the three mutating endpoints; a CSRF failure answers `auth.csrf.invalid` and changes nothing. *(plan §2.1 Security filter and cookie adapter, D2; RF-2.2, RF-2.8, RF-8.3, RF-11.4, RNF-2, RNF-6, LC-9)* — after T-08
  **Done when:** `signOut.csrfRefusedNoChange` passes and a contract test shows the cleared cookie on both 204 responses with `Max-Age=0`, `Secure`, `HttpOnly` and the configured path, and no cleared cookie on any 4xx.

- [ ] **T-18 · Audit adapter [006]** — map `SIGNED_OUT{entry}`, `SESSION_REVOKED_BY_HOLDER{target, alreadyEnded}`, `REVOCATION_REFUSED{target}`, `SIGNED_OUT_EVERYWHERE{count}` and `SIGN_OUT_CSRF_REFUSED` to `AuditPort` entries with identity id, acting session id, target session id, correlation id, network key and timestamp, written in the effect's transaction; `SESSION_ENDED` **[006]** continues to be written by `EndSessionUseCase`. *(plan §2.1 Audit adapter, D13; RF-11.1, RF-11.2)* — after T-12, T-13, T-14
  **Done when:** an integration test performs one sign-out, one revocation, one refused revocation and one sign-out-everywhere and finds exactly the expected entries with the expected payload keys and no key named like a credential, hash or agent.

- [ ] **T-19 · Logging and problem redaction** — request and response logging for the four endpoints exclude the authorisation header, cookies and the user-agent header; problem details carry only the fields of T-04; the list serialiser emits the `SessionView` fields and nothing from the entity. *(plan §1 F-X-03, D3, D16; RF-6.3, RF-11.2, RF-11.3)* — after T-04, T-07
  **Done when:** a log-capture test over the four endpoints finds no access credential, cookie value or agent string, and the list response schema test lists exactly the `SessionView` fields.

## E · API

- [ ] **T-20 · `DELETE /api/auth/session`** — under the session filter **[006]**, CSRF-protected; body `{entry}`; 204 with the cleared cookie for an ended or already-ended session (RF-2.7); the filter's 401 `auth.session.invalid` precedes the controller and is definitive for the client; 403 `auth.csrf.invalid`. *(plan §2.1 `SessionController`, D1, D2; RF-2.1, RF-2.2, RF-2.7, RF-2.8, RF-11.3, LC-8)* — after T-12, T-17
  **Done when:** `signOut.204ClearsRefreshCookie` and `signOut.alreadyEndedAnswers204` pass; an invalid entry value answers 400 with a field path; a missing credential answers 401 `auth.session.invalid`.

- [ ] **T-21 · `GET /api/auth/sessions?page=n`** — under the session filter; 200 `{items: SessionView[], page, pageCount, total}`; `page` defaults to the first page only in the sense of the query parameter being optional, the size is server-side. *(plan §2.1 `SessionController`, D5; RF-6.1, RF-6.3, RF-6.7)* — after T-15, T-19
  **Done when:** a contract test shows the response shape, `pageCount` and `total` consistent with the configured size, and an out-of-range page answering the last page.

- [ ] **T-22 · `DELETE /api/auth/sessions/{id}`** — under the session filter, CSRF-protected; 204 for an own session (ended or already ended); 404 `auth.session.not-revocable` for a foreign, unknown or current id. *(plan §2.1 `SessionController`, D6; RF-7.1, RF-7.2, RF-7.3, RF-7.4, RF-7.6, RF-11.3)* — after T-13, T-17
  **Done when:** `revoke.currentSessionNotRevocable` and `revoke.foreignUnknownCurrentOneCode` pass: the three refusals are byte-identical apart from the correlation id, and a malformed id answers the same 404.

- [ ] **T-23 · `POST /api/auth/sessions/sign-out-everywhere`** — under the session filter, CSRF-protected; body `{idempotencyKey}`; 204 with the cleared cookie; a repeated key answers 204 again from the recorded outcome; the acting identity is the asserted one, no identity parameter exists. *(plan §2.1 `SessionController`, D7; RF-8.3, RF-8.6, RF-8.8)* — after T-14, T-17
  **Done when:** `signOutEverywhere.actsOnAssertedIdentityOnly` passes (a body naming another identity is ignored and answers 400 for the unknown field), and two submissions with one key produce one version bump.

- [ ] **T-24 · Contract and absence checks** — a test enumerating the API surface: the three problem codes with correlation id; no endpoint lists, revokes or signs out another identity's sessions; no tenant-scoped route reaches `RevokeSessionUseCase`; the list body carries no agent string, IP, location, credential, version or cause. *(plan §5.1 API contract, D6, D16; RF-6.3, RF-6.4, RF-7.7, RF-8.8, RF-11.3, RNF-5)* — after T-20…T-23
  **Done when:** `problems.threeSafeCodesWithCorrelationId`, `noAdminSessionEndpointExists`, `revoke.onlyTwoCallersNoTenantScopedEndpoint` and `list.bodyHasNoAgentIpLocationCredentialVersionOrCause` pass.

## F · Front end

- [ ] **T-25 · `contracts/sessions` types and guards** — request / response / problem types with runtime guards for `DELETE session` (entry enum), `GET sessions` (`SessionView`, page fields), `DELETE sessions/{id}` and `POST sign-out-everywhere`; the three problem codes. *(plan §2.2 `contracts/sessions`, D16; RF-2.7, RF-6.2, RF-6.3, RF-7.2, RF-8.6, RF-11.3)* — after T-04
  **Done when:** guard tests accept the documented shapes, reject an item with an extra field named `userAgent`, `ip` or `location`, and reject an unknown problem code.

- [ ] **T-26 · i18n bundles `identity-access.{en,es,fr}` and `shared/security.{en,es,fr}`: session keys** — keys for the shell action, the Sessions area heading and the three statements, the four column headings, browser and platform family labels, "Identity session", "Current session", "Revoke session", "Session revoked", pagination labels and unavailable state, the "Sign out everywhere" button, the confirmation statements and actions, the `signOutIncomplete` / `signOutEverywhereIncomplete` status texts and retry actions, the generic not-revocable problem. *(plan D14; RF-12.4, RNF-14)* — after T-25
  **Done when:** a bundle-parity test shows every key present in all three locales with no empty value, and a label exists for every enum value of T-05.

- [ ] **T-27 · `SessionState.SIGNED_OUT` [006] and transport routing** — `shared/security`: add `SIGNED_OUT`; `shared/api`: the three mutations never auto-retry; a 401 `auth.session.invalid` answered to `DELETE session` is delivered to the coordinator as a definitive answer instead of to `ProtectedBoundary`. *(plan §2.2 `SessionState`, axios transport; RF-2.7, RF-4.1, RF-4.4, RF-5.4, RNF-10)* — after T-25
  **Done when:** `coordinator.signingOutTabNeverShowsDialog` passes: a 401 on `DELETE session` leaves the boundary unmasked and sets `SIGNED_OUT`; a 401 on any other request still masks.

- [ ] **T-28 · `SignOutCoordinator` — core sequence** — `shared/security`: `signOut({entry})` for an entry without a guard: (2) abort in-flight requests and disconnect STOMP using the switch's step-2 primitives **[001/006]**; (3) `DELETE session`; (4) on 204 or 401 `auth.session.invalid`: `AccessCredentialHolder.clear()` **[006]**, purge every Zustand draft, reset the query client, advance the scope epoch, `SessionState = SIGNED_OUT`, route `/sign-in`. Nothing is written to browser storage at any step. *(plan §2.2 `SignOutCoordinator`, D2, D8; RF-2.2, RF-2.3, RF-3.6, RF-4.1, RF-4.4, RF-5.4, RNF-1, RNF-7, RNF-10, LC-5, LC-7)* — after T-27
  **Done when:** `coordinator.abortsAndDisconnectsBeforeRequest`, `coordinator.clearsHolderDraftsQueriesEpochAndRoutes`, `coordinator.treats401InvalidAsDefinitive` and `storage.containsNothingFromFeature` pass with an ordered call log.

- [ ] **T-29 · `SignOutCoordinator` — dirty guard branch** — step (1) for entry `SHELL` when the dirty-draft registry **[006]** is non-empty: await the guarded-switch dirty guard hook **[016]**; `Stay` → return with nothing touched; `Save` → the draft's own save under its stamped `{tenantId, projectId, baseETag}` awaited, a failed, refused or indeterminate save returns to the screen with the work intact and no sign-out (an indeterminate save is looked up per **[006]**); `Discard` → purge, then T-28's steps. The current header selector is never read; no API of the coordinator discards without the guard. *(plan §2.2 `SignOutCoordinator`, D8, D9; RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.7, RNF-10, LC-1, LC-2, LC-3, LC-4)* — after T-28
  **Done when:** `coordinator.runsGuardFirstStayReturns`, `coordinator.saveUnderDraftStampThenSignsOut`, `coordinator.failedRefusedOrIndeterminateSaveCancelsSignOut`, `coordinator.discardPurgesThenSignsOut` and `coordinator.neverRetargetsSaveOrDiscardsSilently` pass.

- [ ] **T-30 · `SignOutCoordinator` — indeterminate and forgery outcomes** — on no answer (network error, timeout, 5xx without body): render `TransportFailureStatus` **[006]** variant `signOutIncomplete` ("sign-out not complete", "Retry sign out" = one request per activation), clear nothing, keep `SessionState` unchanged, let streams reconnect on the next successful request, never retry on its own; a save completed under T-29 is not re-asked on retry; on `auth.csrf.invalid` → `CsrfFailureHandler` **[006]**, clear nothing. Same for `signOutEverywhereIncomplete`. *(plan §2.2 `SignOutCoordinator`, D10; RF-2.8, RF-4.1, RF-4.2, RF-4.3, RF-4.5, RNF-1, RNF-6, LC-6, LC-9)* — after T-29
  **Done when:** `coordinator.indeterminateShowsIncompleteStatusClearsNothing`, `coordinator.afterIndeterminateStreamsReconnectOnNextSuccess`, `coordinator.savedWorkNotReaskedAfterIndeterminate` and `coordinator.csrfClearsNothing` pass.

- [ ] **T-31 · `fromEndedSession()`, `signOutEverywhere(key)` and the R34 wiring [006]** — `fromEndedSession()`: discard the preserved context and drafts, fire `DELETE session {entry: R34}` without awaiting, route `/sign-in`; `SessionExpiryDialog` **[006]** "sign out safely" calls it. `signOutEverywhere(key)`: T-28 steps (2)–(4) with `POST sign-out-everywhere {idempotencyKey}`, the `signOutEverywhereIncomplete` variant on no answer, the same key on retry. *(plan §2.2 `SignOutCoordinator`, D7, D12; RF-3.5, RF-8.5, RF-8.6, RF-9.4, LC-13, LC-24)* — after T-30
  **Done when:** `coordinator.fromEndedSessionDiscardsWithoutGuardOrWait` and `coordinator.everywhereIndeterminateClearsNothing` pass, and the dialog's footer action invokes `fromEndedSession()` with the guard hook never called.

- [ ] **T-32 · `SignOutAction` in `app/shell`** — a labelled "Sign out" button in the identity block of the navigation footer on every desktop app-shell route and an item of the M04 menu on mobile **[F-X-07]**; rendered for every authenticated route regardless of grants, gate state or tenant state; visible focus, 44 px target; activation → `signOut({entry: 'SHELL'})`; the M04 destinations keep following server-derived grants. *(plan §2.2 `SignOutAction`, D15; RF-1.2, RF-1.3, RF-1.5, RF-12.2, RF-12.3)* — after T-28
  **Done when:** `shell.signOutOnEveryAuthenticatedRouteDesktopAndM04Menu` and `shell.signOutIgnoresGrants` pass: the action is found on every registered app-shell route at desktop and mobile widths, with a grant-less Member fixture and a gated Owner fixture.

- [ ] **T-33 · Designed entries: N03, N20, M03 / N29; R08 untouched** — N03 **[013]** and N20 **[014]** render `SignOutAction` with entries `N03` / `N20`; M03 / N29 **[008]** "Switch identity safely" calls `signOut({entry})` and touches no invitation state; none of them registers drafts, so no guard; R08 **[004]**, R05 **[001]** and N05 **[005]** keep calling `DELETE challenge` and never `DELETE session`. *(plan §2.2 Other entries; RF-1.1, RF-1.4, RF-3.5, RF-9.1, RF-9.2, RF-9.3, RF-9.5, LC-12)* — after T-28
  **Done when:** `publicScreens.noGuard` and `limitedStateScreens.callAbandonNeverSignOut` pass, and a unit test per screen shows the entry value sent.

- [ ] **T-34 · Hooks `useSessions`, `useRevokeSession`, `useSignOutEverywhere`, `useSignOut`** — typed hooks over `contracts/sessions`; `useSessions(page)` is a query with no interval and no focus refetch; the three mutations have no retry and return server state verbatim; `useSignOutEverywhere` delegates to `SignOutCoordinator.signOutEverywhere`. *(plan §2.2 hooks; RF-6.8, RF-7.1, RF-8.3, RNF-10)* — after T-31
  **Done when:** hook tests show zero automatic refetches over a simulated interval and focus change, zero retries on a failed mutation, and the coordinator called once per `useSignOutEverywhere` activation.

- [ ] **T-35 · `SessionsArea` and `SecurityStatements` [001]** — the Sessions tab of `SecuritySettingsPage` **[003]**: heading "Authenticated sessions", `SessionsTable`, record count, `Pagination`, `SignOutEverywhereAction`, `SecurityStatements` extended with "Sessions are not trusted devices. No session grants a TOTP bypass. Expiring a session does not cancel an accepted run."; fetch on open and refetch after each action of this feature; no interval. *(plan §2.2 `SessionsArea`, `SecurityStatements`, D5, D14; RF-6.1, RF-6.6, RF-6.8, RF-8.1, RF-10.3)* — after T-34, T-26
  **Done when:** `statements.threeSessionStatementsPresent` and `sessionsArea.fetchesOnOpenAndAfterActionsNeverInterval` pass.

- [ ] **T-36 · `SessionsTable` — cells and the current row** — columns Session (family labels + `createdAt`), Context (names or "Identity session"), Expiry (`expiresAt` formatted for display, never computed or counted down), Action ("Current session" text on `isCurrent`, no button); nothing else from the view is rendered; while the viewer's own session ends the next request is handled by `ProtectedBoundary` **[006]** and the table renders nothing from a stale response. *(plan §2.2 `SessionsTable`, D3, D5; RF-6.2, RF-6.3, RF-6.9, RF-6.10, RF-7.6, RNF-8)* — after T-35
  **Done when:** `table.currentRowHasNoRevoke` and `expiry.displayedFromServerNeverComputed` pass, and a snapshot shows the four cells only for a fixture with every family value.

- [ ] **T-37 · `SessionsTable` — revoke action states** — "Revoke session" button per non-current row with an accessible name including the session description; activation → `useRevokeSession(id)` at once (no dialog); 204 → the row shows "Session revoked" in text (live region, polite) until the next read; `auth.session.not-revocable` → one generic inline problem; no answer → `TransportFailureStatus` on the row, action repeatable, never "revoked" without an answer. *(plan §2.2 `SessionsTable`, D6, D10; RF-7.1, RF-7.3, RF-7.4, RF-12.3, LC-17, LC-18, LC-19, LC-20)* — after T-36
  **Done when:** `table.rowShowsSessionRevokedText` and `table.indeterminateShowsStatusRepeatable` pass, and the button's accessible name contains the family labels and sign-in time of its row.

- [ ] **T-38 · `Pagination`** — "Previous · Next" driven by `page` and `pageCount` from the server; an unavailable direction is disabled with its state conveyed in text; the record count is shown. *(plan §2.2 `Pagination`; RF-6.7, LC-28)* — after T-35
  **Done when:** `pagination.textStateWhenUnavailable` passes on the first, a middle and the last page.

- [ ] **T-39 · `SignOutEverywhereAction` and `ConfirmDialog`** — button beside the list (S-43, drawn `427:4040`); `shared/ui` `Dialog` with focus trap and return focus, carrying the statements "every session of your identity, including this one, will be signed out", "accepted runs continue", "you will have to sign in again", actions "Sign out everywhere" and "Cancel", no password or code field; an idempotency key generated once per open and reused on retry; confirm → `useSignOutEverywhere(key)`; the Settings route registers no draft. *(plan §2.2 `SignOutEverywhereAction`, `ConfirmDialog`, D7, D10, D14; RF-8.1, RF-8.2, RF-8.5, RF-8.6, RF-10.3, RF-12.3, LC-23, LC-24)* — after T-35
  **Done when:** `signOutEverywhere.buttonAndConfirmStatementsNoCredentialField`, `confirmDialog.statesRunsContinue`, `confirmDialog.reusesKeyOnRetry` and `settingsRoute.holdsNoDraft` pass.

- [ ] **T-40 · Reflow and accessibility primitives** — Tailwind breakpoints: the table becomes one record per session at mobile and tablet widths with the same four cells and the action; the confirmation and the status variants reflow; every action 44 px with visible focus; status conveyed by text; live announcements never move focus; the shell action reachable by keyboard in the M04 menu. *(plan D14, D15; RF-1.5, RF-12.1, RF-12.3, RNF-13, RNF-15, LC-14)* — after T-37, T-38, T-39, T-32
  **Done when:** Testing Library tests at three widths find every cell, statement and action, and no interactive element below 44 px.

## G · Verification suites

- [ ] **T-41 · Sign-out server-side suite** — `signOut.endsSessionAndFamilyAtomically`, `signOut.nextRequestRefused`, `signOut.leavesOtherSessionsOfIdentityUntouched`, `signOut.touchesOnlySessionTables`, `signOut.allowedForGatedOwnerGrantlessMemberSuspendedTenant`, `restrictedIdentity.signOutOfEndedSessionAnswersDone`, `feature.grantsNoAuthorityAndNoFactorBypass`, `feature.neverCallsNotificationPort`, `lostAck.lookedUpAfterNextSignIn`. *(plan §5.2 RF-1.3, RF-2, RF-3.6, RF-11.5, RNF-3, RNF-4 rows, D1; RF-1.3, RF-2.1, RF-2.4, RF-2.5, RF-3.6, RF-5.3, RF-11.5, RNF-2, RNF-3, RNF-4, LC-5, LC-30)* — after T-20, T-24
  **Done when:** all nine tests pass with real PostgreSQL and the recording doubles for notification, tenant and run ports report zero calls.

- [ ] **T-42 · Concurrency suite** — `revoke.racingExpiryOneOutcome`, `signOut.twoConcurrentOneEnd`, `everywhere.racingRefreshRefusesRefresh` (a refresh **[006]** presented during the sign-out-everywhere transaction is refused after it commits), `twoDevices.mutualRevokeBothEnd`. *(plan §5.1 Concurrency, D6, D7; RF-7.4, RF-8.3, RNF-2, RNF-6)* — after T-41
  **Done when:** all four tests pass ten times in a row with real PostgreSQL.

- [ ] **T-43 · Run continuity suite** — `run.continuesAcrossSignOutRevocationAndEverywhere` (an accepted Run under the F-RUN double keeps recording facts through all three actions), `pendingRun.lookedUpNeverResubmittedAfterSignOutEverywhere` (BE and FE halves). *(plan §5.2 RF-2.6, RF-10 rows, D1; RF-2.6, RF-10.1, RF-10.2, RNF-11, LC-22, LC-25)* — after T-41
  **Done when:** both tests pass; the run-scoped authority double records zero calls from this feature.

- [ ] **T-44 · List suite** — `list.allLiveSessionsAcrossTenantsNewestFirstPaged`, `list.identityScopedOnly`, `list.lostMembershipShowsIdentitySession`, `list.pageBeyondLastReturnsLast`. *(plan §5.2 RF-6 rows, D5; RF-6.1, RF-6.4, RF-6.5, RF-6.7, RNF-5, LC-15, LC-16, LC-28)* — after T-21, T-16
  **Done when:** all four tests pass with real PostgreSQL, the identity-scope test using two identities with sessions in the same tenant.

- [ ] **T-45 · Revocation and sign-out-everywhere suite** — `revoke.endsAtOnceNoDialogNoStepUp`, `revoke.foreignUnknownCurrentOneCode`, `revoke.refusalAudited`, `revoke.alreadyEndedAnswersDone`, `revoke.sendsNoNotification`, `signOutEverywhere.bumpsRevokesAllVoidsTransactionsOneTransaction`, `signOutEverywhere.recordedOutcomeForRepeatedKey`, `signOutEverywhere.leavesRequirementFactorCodesMembershipsCase`. *(plan §5.2 RF-7, RF-8 rows, D6, D7; RF-7.1, RF-7.2, RF-7.4, RF-7.5, RF-8.3, RF-8.6, RF-8.7, RNF-2, LC-17, LC-18, LC-19, LC-23, LC-26)* — after T-22, T-23, T-16
  **Done when:** all eight tests pass with real PostgreSQL and plans 003, 004 and 005 built or their recording stubs.

- [ ] **T-46 · Audit, redaction and startup suite** — `audit.entryPerEvent`, `audit.containsNoCredentialOrAgentString`, `logs.containNoCredential`, `sessionPolicy.startupFailsOnAbsentOrUnboundedPageSize`. *(plan §5.2 RF-11, RF-6.7 rows, D13; RF-6.7, RF-11.1, RF-11.2, RNF-9)* — after T-18, T-19, T-01
  **Done when:** all four tests pass; the audit test asserts one entry per event of RF-11.1 and none for a list read.

- [ ] **T-47 · FE e2e journeys** — Playwright journeys for US-1…US-10 in EN, ES and FR at desktop, tablet and mobile widths against the BE test doubles: `signOut.availableOnN03N20M03N29R34`, `afterSignOut.signInFormEmpty`, `coordinator.saveUnderDraftStampThenSignsOut` (journey half), `multiTab.signOutInOneOthersMeetDialogWithDrafts`, `multiTab.reauthRestoresPreservedContext`, `revokedDevice.meetsDialogDraftKept`, `threeDevices.allEndCurrentToSignInOthersMeetDialog`, `list.viewerSessionEndsShowsDialogNeverServed`, `languageSwitch.changesDisplayOnlyRecordsUnchanged`. *(plan §5.1 FE e2e; RF-1.1, RF-1.2, RF-2.3, RF-3.2, RF-5.1, RF-5.2, RF-5.3, RF-6.9, RF-7.5, RF-8.4, RF-12.4, RNF-14, LC-2, LC-10, LC-11, LC-12, LC-21, LC-23, LC-27, LC-29; US-1 … US-10)* — after T-40, T-24
  **Done when:** every journey passes in the nine locale × width combinations, and each US-1 … US-10 story is named by at least one passing journey.

- [ ] **T-48 · Automated accessibility** — axe on the shell action (footer and M04 menu), the Sessions area (table and record layouts, revoked row, indeterminate row), the confirmation dialog and both status variants, in EN, ES and FR at three widths. *(plan §5.1 FE e2e + accessibility; RF-1.5, RF-12.1, RF-12.3, RNF-13, RNF-15, LC-14)* — after T-47
  **Done when:** `a11y.signOutEntries.{en,es,fr}.{desktop,tablet,mobile}` and `a11y.sessionsArea.{en,es,fr}.{desktop,tablet,mobile}` report zero critical or serious violations.

- [ ] **T-49 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist, versions from the release-support matrix: row action names announce the session description, "Session revoked" is announced without focus moving, the confirmation traps focus and returns it to the button on close, the M04 menu item is reachable and labelled, the status variants are announced as text. *(plan §5.1 Manual assistive; RF-12.3, RNF-13)* — after T-48
  **Done when:** the checklist is complete for all four combinations with no open defect.

- [ ] **T-50 · Adversarial suite (US-11)** — a credential of a signed-out or revoked session presented to a protected endpoint; session-id enumeration through `DELETE sessions/{id}`; revocation and sign-out-everywhere attempted with another identity's id; a sign-out whose server call is dropped (nothing cleared); a Run cancellation attempted through sign-out; a crafted `DELETE sessions/{acting}`. *(spec US-11; RF-2.1, RF-4.2, RF-7.2, RF-7.6, RF-8.8, RF-10.1)* — after T-42, T-45, T-30
  **Done when:** every case is refused or has no effect, each with the expected safe code, and no response or log differs between a foreign and an unknown session id.

- [ ] **T-51 · Areas A and F evidence pack** — collect the passing runs of T-41 … T-50 with their database and browser versions into the release evidence folder. *(spec RNF-16; constitution P9)* — after T-50
  **Done when:** the pack lists each suite with run id, date and version, and spec §7 items 3–12 each reference at least one run.

## H · Closing

- [ ] **T-52 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-51
  **Done when:** the script reports 128 of 128 identifiers (71 RF, 16 RNF, 30 LC, 11 US) covered by at least one passing test.

- [ ] **T-53 · Register-dependency review** — walk plan §6 — A-51, S-43 and S-44 as decided, and the open A-11, A-27, A-31, A-35, A-39 (port contract), A-49, A-50, S-02 / S-28, S-07 and S-41: for each, confirm the built assumption matches the current Figma register state or record the deferral. *(spec §7 item 13)* — after T-52
  **Done when:** each of the thirteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-54 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run ids, a11y report, manual checklist, coverage report, evidence pack, register review). — after T-53
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-12, T-33, T-47 |
| RF-1.2 | T-32, T-47 |
| RF-1.3 | T-12, T-32, T-41 |
| RF-1.4 | T-33 |
| RF-1.5 | T-32, T-40, T-48 |
| RF-2.1 | T-06, T-12, T-20, T-41, T-50 |
| RF-2.2 | T-17, T-20, T-28 |
| RF-2.3 | T-28, T-47 |
| RF-2.4 | T-12, T-41 |
| RF-2.5 | T-12, T-41 |
| RF-2.6 | T-12, T-43 |
| RF-2.7 | T-12, T-20, T-27, T-28 |
| RF-2.8 | T-17, T-20, T-30 |
| RF-3.1 | T-29 |
| RF-3.2 | T-29, T-47 |
| RF-3.3 | T-29 |
| RF-3.4 | T-29 |
| RF-3.5 | T-31, T-33 |
| RF-3.6 | T-28, T-41 |
| RF-3.7 | T-29 |
| RF-4.1 | T-27, T-28, T-30 |
| RF-4.2 | T-30, T-50 |
| RF-4.3 | T-30 |
| RF-4.4 | T-27, T-28 |
| RF-4.5 | T-30 |
| RF-5.1 | T-47 |
| RF-5.2 | T-47 |
| RF-5.3 | T-12, T-41, T-47 |
| RF-5.4 | T-27, T-28 |
| RF-6.1 | T-02, T-15, T-16, T-21, T-35, T-44 |
| RF-6.2 | T-05, T-06, T-07, T-09, T-10, T-15, T-36 |
| RF-6.3 | T-05, T-07, T-19, T-21, T-24, T-36 |
| RF-6.4 | T-08, T-15, T-24, T-44 |
| RF-6.5 | T-10, T-11, T-15, T-44 |
| RF-6.6 | T-07, T-35 |
| RF-6.7 | T-01, T-15, T-21, T-38, T-46 |
| RF-6.8 | T-15, T-34, T-35 |
| RF-6.9 | T-36, T-47 |
| RF-6.10 | T-07, T-36 |
| RF-7.1 | T-06, T-13, T-22, T-34, T-37, T-45 |
| RF-7.2 | T-04, T-13, T-16, T-22, T-45, T-50 |
| RF-7.3 | T-22, T-37 |
| RF-7.4 | T-13, T-22, T-37, T-42, T-45 |
| RF-7.5 | T-13, T-45, T-47 |
| RF-7.6 | T-13, T-22, T-36, T-50 |
| RF-7.7 | T-08, T-13, T-24 |
| RF-8.1 | T-35, T-39 |
| RF-8.2 | T-39 |
| RF-8.3 | T-02, T-06, T-14, T-17, T-23, T-34, T-42, T-45 |
| RF-8.4 | T-14, T-47 |
| RF-8.5 | T-31, T-39 |
| RF-8.6 | T-03, T-14, T-16, T-23, T-30, T-31, T-39, T-45 |
| RF-8.7 | T-14, T-45 |
| RF-8.8 | T-14, T-23, T-24, T-50 |
| RF-9.1 | T-12, T-33 |
| RF-9.2 | T-12, T-33 |
| RF-9.3 | T-12, T-33 |
| RF-9.4 | T-12, T-31 |
| RF-9.5 | T-33 |
| RF-10.1 | T-12, T-43, T-50 |
| RF-10.2 | T-43 |
| RF-10.3 | T-35, T-39 |
| RF-11.1 | T-12, T-13, T-14, T-18, T-46 |
| RF-11.2 | T-05, T-18, T-19, T-46 |
| RF-11.3 | T-04, T-19, T-20, T-22, T-24 |
| RF-11.4 | T-12, T-14, T-17 |
| RF-11.5 | T-08, T-14, T-41 |
| RF-12.1 | T-40, T-48 |
| RF-12.2 | T-32 |
| RF-12.3 | T-32, T-37, T-39, T-40, T-48, T-49 |
| RF-12.4 | T-26, T-47 |
| RNF-1 | T-28, T-30 |
| RNF-2 | T-06, T-12, T-14, T-17, T-41, T-42, T-45 |
| RNF-3 | T-41 |
| RNF-4 | T-41 |
| RNF-5 | T-08, T-11, T-13, T-15, T-24, T-44 |
| RNF-6 | T-13, T-17, T-30, T-42 |
| RNF-7 | T-28 |
| RNF-8 | T-07, T-36 |
| RNF-9 | T-01, T-46 |
| RNF-10 | T-27, T-28, T-29, T-34 |
| RNF-11 | T-43 |
| RNF-12 | T-05, T-07 |
| RNF-13 | T-40, T-48, T-49 |
| RNF-14 | T-26, T-47 |
| RNF-15 | T-40, T-48 |
| RNF-16 | T-51 |
| LC-1 | T-29 |
| LC-2 | T-29, T-47 |
| LC-3 | T-29 |
| LC-4 | T-29 |
| LC-5 | T-12, T-28, T-41 |
| LC-6 | T-30 |
| LC-7 | T-28, T-30 |
| LC-8 | T-12, T-20 |
| LC-9 | T-17, T-30 |
| LC-10 | T-47 |
| LC-11 | T-47 |
| LC-12 | T-33, T-47 |
| LC-13 | T-31 |
| LC-14 | T-40, T-48 |
| LC-15 | T-05, T-15, T-44 |
| LC-16 | T-11, T-15, T-44 |
| LC-17 | T-13, T-37, T-45 |
| LC-18 | T-13, T-37, T-45 |
| LC-19 | T-13, T-37, T-45 |
| LC-20 | T-37 |
| LC-21 | T-47 |
| LC-22 | T-43 |
| LC-23 | T-14, T-39, T-45, T-47 |
| LC-24 | T-30, T-31, T-39 |
| LC-25 | T-43 |
| LC-26 | T-14, T-45 |
| LC-27 | T-47 |
| LC-28 | T-15, T-38, T-44 |
| LC-29 | T-47 |
| LC-30 | T-12, T-41 |
| US-1 … US-10 | T-47 (journeys) |
| US-11 | T-50 (adversarial suite) |

Self-check: all 71 RF criteria, 16 RNF and 30 limit cases of `specLogoutSessionManagement.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| A-51 session record fields and "Sign out everywhere" scope (decided, Q2 / Q7 / Q8) | Families classified at issue, last scope recorded by `access`, nearer deadline shown; every session ended including the current one, version bump, no notification | T-02, T-05, T-06, T-09, T-10, T-13, T-14, T-15 |
| S-43 "Sign out everywhere" action and confirmation (decided, Q1 / Q2 / Q8) | Button beside the list and `ConfirmDialog` with the drawn copy of D02 `422:3292`, no credential field | T-39 |
| S-44 content of the table rows (decided, Q7) | Four cells as decided, "Current session" text, "Session revoked" row state | T-36, T-37 |
| A-11 configuration values | `sessionListPageSize` in `SessionPolicyProperties` **[006]** | T-01, T-46 |
| A-27 identity statuses | A restricted or erasing identity's sessions end under **[006]**; sign-out of such a session answers done | T-12, T-41 |
| A-31 re-authentication (closed by 006) | Other tabs re-authenticate from R34 after a sign-out exactly as after expiry | T-47 |
| A-35 persistence tooling | Columns, index and outcome table against the placeholder access API and migration tool | T-02, T-03, T-16 |
| A-39 revocation scope | `revokeAll(identity, except = none, cause)` as decided for Q2; run-scoped authority never referenced | T-08, T-14, T-43 |
| A-49 human activity | The list read is a deliberate navigation; the scope hook counts nothing | T-10, T-15 |
| A-50 reuse detection | Its notification points at this list | T-35 |
| S-02 / S-28 responsive coverage | Sessions area, confirmation and status variants follow the D01 responsive policy with the D02 `422:3292` record form | T-40, T-48 |
| S-07 app-shell sign-out | `SignOutAction` in the footer identity block and the M04 menu | T-32 |
| S-41 retry / offline status | `TransportFailureStatus` **[006]** variants `signOutIncomplete` / `signOutEverywhereIncomplete` with the D02 copy (`424:3581`) | T-26, T-30, T-31 |
