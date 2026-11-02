# Plan 016 — Guarded tenant and project switch

**Feature** The transition that moves a signed-in person from one working scope (a tenant and, within it, a project) to another, protecting unsaved work, carrying nothing across the boundary and loading the new scope only after its authority has been derived afresh · **Plan version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — aligned with specification v0.3 (decisions Q1–Q8, derived rules D-a…D-e) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/016-GuardedTenant/specGuardedTenant.md` v0.3 (the WHAT) · **Governed by** `constitution.md` (principles P1–P16; feature F-IDN-18 as the subject; F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-13, F-IDN-14, F-IDN-15 and F-IDN-16 at the boundary; F-X-01…F-X-05, F-X-07) · **Architecture source** Figma FE board `46:7847` (feature-modular SPA DAG: `app` owns providers, routes, compositions and the tenant shell; `shared` holds generic api / realtime / ui / i18n / errors / security, "security / scope epochs"; no shared → feature import), board 3 §03 `46:7847` ("switch request → dirty guard → drain old scope → reload trusted new scope", followed where the FE section is silent, register A-03), BE board `23:5587` and decision register `122:2` §5 (the `access` context owns memberships, grants and permission versions) · **Builds on** plan 001 (Login: the scope epoch and the axios transport), plan 003 (Enrollment: the gate handler), plan 006 (Sessions: `ProtectedBoundary`, the STOMP client of `shared/realtime`, the stamped Zustand drafts, `DeniedScopeHandler` and N28, `TransportFailureStatus`), plan 007 (Logout: `SignOutCoordinator`, which already awaits this feature's guard hook), plan 009 (Roles and grants: `DeriveAuthorityUseCase`, `GET me/authority` and `AuthorityStore`, loaded at step 4 of this switch), plan 011 (its edit panel registers a stamped draft), plan 012 (workspace creation calls `useGuardedSwitch`), plan 013 (Tenant chooser: `SwitchWorkspaceAction`, `useEnterTenant`, `TenantEntryUseCase` and `POST /api/tenants/{id}/entry`, `EntryRefusedHandler`, `ChooserOrder`, the `ProjectNamesPort` cross-plan item) and plan 014 (Tenant suspension: `TenantStateResolver`, `GET access-status`, the N20 route). Elements they own are tagged **[001]** … **[014]** and are extended, never duplicated. Plan 008 enters the tenant after acceptance and has no element of its own here.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's eight decisions (Q1–Q8) and five derived rules (D-a…D-e) are built as decided. Where the specification depends on Figma register items (A-63, A-03; S-58, S-59, S-07, S-02) this plan builds the stated assumption; one item is raised by this plan itself — **A-67**, the client routing and navigation-interception mechanism the fixed stack does not name (§4 D11, §6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n, US-n); the FR identifiers they implement are listed in the specification and repeated here only where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The switch sends no acceptance, approval, launch, cancel or replay. Aborting a pending command only stops the tab waiting for it; the command's operation identity is kept so its outcome can be looked up, never resubmitted. Accepted jobs run under their own pins, unseen by the switch. | RF-5.2–5.4, RNF-4 |
| **P3** The browser expresses intent, never authority | The client sends a *requested* scope. Whether it may be entered is decided on the server by `TenantEntryUseCase` **[013]** (extended to the project level) through `DeriveAuthorityUseCase` **[009]**; the PROJECT list is a server read model; client clearing and the guard decide nothing. | RF-3.7, RF-7.1, RF-8.2, RF-8.3, RF-9.4, RNF-5 |
| **P6** Fail closed | A step that cannot complete stops every later step; a failed or unanswered check loads nothing and shows the check-failed panel; the drain clears every query key not on an explicit identity allowlist; an unknown response epoch is dropped. | RF-2.4, RF-7.6, RF-8.1, RNF-2 |
| **P10** Fixed technology stack | FE: React 18, TypeScript, Vite, TanStack Query (server facts, cancellation, cache removal), Zustand (only the existing editor forks **[006]**), i18next, Tailwind, axios (abort signals, interceptors), STOMP (disconnect). BE: Java 21, Spring Boot 3.5, PostgreSQL, existing Spring Security. No routing library is added: navigation interception goes through a shell port over the standard browser History API and the leave-page event, and the choice of a routing mechanism is register item A-67. Test-only tools are flagged in §5. | whole plan |
| **P11** Architecture styles | FE: the coordinator, registries and port live in `shared/security`; the N24 dialog, the selectors and the navigation guard live in `app/shell`, which implements the `shared` ports — imports stay acyclic. TanStack Query owns server facts; Zustand owns only deliberate editor forks; other unsaved forms register callbacks, not state. BE: one use-case extension and one read query in the hexagonal `access` context **[009]**; `project` and `audit` reached only through ports. | §2 |
| **P12** Two independent tenancy gates | The entry and the PROJECT list derive C10 server-side, so RLS (`FORCE RLS`, `SET LOCAL` per transaction) and runtime authorisation both apply to everything the new scope loads; nothing the client cleared or kept is trusted. | RF-7.1, RF-8.2 |
| **P13** Configured values are never defaulted | The switch introduces no numeric value: no waiting period (D-a), no timeout, no retry count of its own; safe-read bounds are plan 006's. | RNF-6 |
| **P14** Release bar | WCAG 2.2 AA on N24 and its states, the switching state, the PROJECT list and the check-failed panel, with automated and manual assistive evidence; EN / ES / FR. | RNF-8–10 |
| **F-X-01** C10 context | Route and body identifiers are requested scope; step 4 derives tenant and project authority on the server; every later request re-derives. | RF-7.1, RF-8.2, RF-9.4 |
| **F-X-02** Request integrity | Each guard save carries its own operation identity and the stamp of the scope where the work was written; mutations are never auto-retried; a lost acknowledgement is looked up through the ledger's operation identity (FR-X-022); a request whose epoch moved is never retargeted. | RF-4.1, RF-4.6, RF-5.2, RF-5.3 |
| **F-X-03** Secrets and safe problems | Problems are RFC 9457 with safe codes and a correlation id; drafts, prompts and secrets never reach browser storage; object URLs of the old scope are revoked at the drain. | RF-6.2, RF-12.2, RF-12.3, RNF-7 |
| **F-X-04** Accessibility and languages | Modal guard with focus trap and focus return, announced switching state, keyboard-operable list, statements from i18next keys. | RNF-8, RNF-9 |
| **F-X-05** Audit log | Refused tenant entries are recorded by `TenantEntryUseCase` **[013]**, refused project entries by its project-level extension, once per refusal whatever the entry point. | RF-12.1 |
| **F-X-07** Responsive surfaces | M24 (`432:4464`) is the drawn counterpart of N24; the selectors appear in the M04 menu; other widths follow the D01 responsive policy (S-02). | RF-1.5, RNF-10 |
| **F-IDN-18** Guarded switch | Built in the exact order of FR-IDN-130 by one coordinator; FR-IDN-131 by stamped saves; FR-IDN-132 by epoch checks in transport and streams and by server derivation. | RF-2, RF-4, RF-8 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access` [009]

The switch is mostly a client transition; the back end contributes the entry decision for the new scope and the PROJECT list. No aggregate, table or configuration is added.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.application` | `TenantEntryUseCase` **[013]** — extended | Gains an optional `projectId`. Tenant level unchanged (013 D3): `DeriveAuthorityUseCase` **[009]** at tenant level → `ENTERED` / `Denied(NO_MEMBERSHIP \| MEMBERSHIP_NOT_ACTIVE \| TENANT_NOT_ACTIVE)` audited as `TENANT_ENTRY_REFUSED{cause}` with one problem `access.entry-refused` / `Denied(ENROLLMENT_GATE)` → `auth.enrollment.required {reason}` **[003]**. When `projectId` is present and the tenant level is granted: a project-level derivation — project exists, not deleted, at least one live grant — → `ENTERED{tenantId, projectId}`, or `Denied(NO_PROJECT_GRANT \| PROJECT_GONE)` audited as `PROJECT_ENTRY_REFUSED{tenantId, projectId, cause}` and answered with `access.denied-scope` **[006]** (the code N28 already handles). Writes nothing but the audit row; creates no session. | RF-7.1–7.5, RF-7.7, RF-12.1, LC-20–LC-24, LC-26, LC-28, LC-29 |
| | `MyProjectsQuery` (new) | For the session identity and a requested `tenantId`: tenant-level derivation first (a refusal answers `access.entry-refused` like the entry); then the identity's own live project grants in that tenant, with names through `ProjectNamesPort` **[013 cross-plan item]**, ordered by the project-line comparator of `ChooserOrder` **[013]** under the locale collation; each line `{projectId, projectName, grants, canRunOnly}`; `noGrants` when none. Never another project, count, source, run or member. Writes nothing. | RF-1.3, LC-4, LC-5 |
| | Ports | Reused only: `DeriveAuthorityUseCase`, `MembershipProjectionPort`, `TenantStatusPort`, `ProjectExistsPort` **[009]**, `ProjectNamesPort` **[013]**, `SessionAssertionPort` **[006]**, `AuditPort` **[001]**, `Clock`. | all |
| `access.infrastructure` | Audit adapter **[013]** — extended | Adds `PROJECT_ENTRY_REFUSED{tenantId, projectId, cause}` with identity id, correlation id, calling entry (`CHOOSER \| SELECTOR \| ROUTE \| INVITATION \| CREATION \| LEAVE`), timestamp; never a name, a grant or content. | RF-12.1 |
| `access.api` | `TenantEntryController` **[013]** — extended | `POST /api/tenants/{tenantId}/entry` body `{projectId?, entry}` → 200 `{state: ENTERED, tenantId, tenantName, tenantRole, projectId?, projectName?}` · 403 `access.entry-refused` · 403 `access.denied-scope` · 403 `auth.enrollment.required {reason}` · 401 without a session. | RF-7.1–7.5 |
| | `MyProjectsController` (new) — `/api/tenants/{tenantId}/me/projects` | `GET` (session required) → 200 `{lines, noGrants}` · 403 `access.entry-refused` · 401. No query parameter, no paging. | RF-1.3 |

**Cross-context touch points.** `project` — `ProjectExistsPort` **[009]**, `ProjectNamesPort` **[013]** (names only, under the tenant predicate). `identity` — `SessionAssertionPort` **[006]**. `audit` — append-only. No membership, grant, draft, approval or run row is written.

**Data model note (P12).** No table is added. Both endpoints run inside a transaction that sets the derived C10 with `SET LOCAL`; project names and grants are read only after the tenant level is granted, so a refused tenant contributes nothing to any response.

### 2.2 Front end — `shared/security/switch`, `shared/api`, `shared/realtime`, `shared/errors`, `app/shell`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| `shared/security/switch` | `GuardedSwitchCoordinator` | The single sequence for every entry point, per tab; for origin `CHOOSER` steps (2)–(3) run only after 013 RF-6.1 has granted the entry (the chooser opens with the old scope current), and for origin `LEAVE` steps (2)–(3) run and the chooser opens with no step (4). `switchTo(target{tenantId, projectId?}, origin)`: (0) if `target` equals the current scope → return (no guard, drain or fetch); if a switch is already running → ignore the request; (1) guard (`UnsavedWorkRegistry` non-empty → `DirtyGuardPresenter.ask(items)` → `STAY` → return, nothing touched; `DISCARD` → `discard()` every item; `SAVE` → `GuardSaveRunner`, any non-`SAVED` outcome → return with the switch cancelled); (2) `ScopeAbortRegistry.abortCurrent()`, `RealtimeScope.disconnect()`, pending commands moved to the `PendingCommandLedger` with the notice; (3) `ScopeEpoch.advance()` **[001]**, `ScopeDrain.run()`; (4) `ScopeEntryCheck.run(target)`; (5) only on `ENTERED` → publish the new scope, navigate, let screens load and streams connect. Exposes `switchState` (`IDLE \| GUARDING \| SAVING \| SWITCHING \| CHECK_FAILED`) for the shell. On `SessionState → ENDED` **[006]** during (1) the run is abandoned (D-d). | RF-1.1, RF-1.4, RF-2.1–2.4, RF-3.3, RF-11.1, LC-6, LC-33 |
| | `useGuardedSwitch(target, origin)` | The published hook other plans call (plan 012 already does; plan 013's `useEnterTenant`, plan 008's acceptance exit and the shell selectors are re-pointed at it). Returns `switchState` and the outcome. | RF-1.1, LC-35 |
| | `UnsavedWorkRegistry` | Per tab, in memory. Entries `{id, label, stamp{identityId, tenantId, projectId, baseETag, editSeq}, save(): Promise<SaveOutcome>, discard()}` with `SaveOutcome = SAVED \| CONFLICT \| INVALID \| INDETERMINATE \| SUPERSEDED_EDIT`. The Zustand editor forks **[006 D10]** register automatically when dirty; plan 011's panel already registers a stamped draft **[011 D8]**; settings and policy forms, dialogs with typed input and the chat / proposal composer register through `useUnsavedWork({label, isDirty, save, discard})`. Filters, sort, scroll and expansion never register. It is the "dirty guard hook" plan 007's `SignOutCoordinator` awaits. | RF-3.2, RF-3.3, RF-3.6, RF-11.2, LC-7, LC-15, LC-34 |
| | `DirtyGuardPresenter` (port) | `ask(items) → Promise<SAVE \| DISCARD \| STAY>`, `showSaving()`, `close()`; implemented by `app/shell` at bootstrap so `shared` never imports `app`. | RF-3.1, RF-3.4 |
| | `GuardSaveRunner` | Calls each item's `save()` — each submits under its own stamp and operation identity, never under `target` — sequentially in registration order, then reports. All `SAVED` → continue. Otherwise cancel: `CONFLICT` → the item opens N25 (its own resolution, authoring area); `INVALID` → the item shows field problems; `INDETERMINATE` → the item's lookup by operation identity, no resubmission; `SUPERSEDED_EDIT` (typing continued: only the captured `editSeq` acknowledged) → the item stays registered and the guard is shown again on the next attempt. Acknowledged items deregister; the rest stay. | RF-4.1–4.8, LC-3, LC-7–LC-12 |
| | `ScopeEpoch` **[001]** — extended | One counter per tab covering tenant + project (A-63); every scoped request and subscription captures it; never shared between tabs, never persisted. | RF-6.1, RF-10.1 |
| | `ScopeAbortRegistry` (new) | Holds one `AbortController` per epoch; `shared/api` attaches its signal to every scoped request; `abortCurrent()` aborts safe reads and pending mutations alike (aborting stops waiting, the server command continues) and opens a fresh controller. | RF-5.1, RF-5.2 |
| | `PendingCommandLedger` (new) | Per tab, in memory: for each mutation aborted at step 2, `{operation, operationId, tenantId, projectId, at}` — identifiers only, no body, no content. The owning feature's lookup (FR-X-022) reads it when its screen is next opened in that scope; an entry is removed when looked up. Drives the notice "A pending request remains bound to its original tenant and project." Never resubmits. | RF-5.2, RF-5.3, LC-16 |
| | `ScopeDrain` (new) | Step 3, in order: `queryClient.cancelQueries()` then `removeQueries` for every key **not** on the identity allowlist (the session status **[006]**, `me/memberships` **[013]**, identity facts, the language preference); `AuthorityStore.clear()` **[009]**; purge every Zustand draft **[006]** and every registry entry; revoke every object URL in `ScopedObjectUrls`. Identity-scoped facts and the FR-X-034 allowlist survive. | RF-6.2, RF-6.3, RF-12.2, LC-19, LC-38 |
| | `ScopedObjectUrls` (new) | Registry through which previews and downloads create object URLs, so the drain can revoke them. | RF-6.2 |
| | `ScopeEntryCheck` (new) | Step 4: `POST entry {projectId?, entry: origin}` (§2.1) then, on `ENTERED`, `GET me/authority` **[009]** into `AuthorityStore`. Outcome routing: `ENTERED` → step 5; `access.entry-refused` → origin `CHOOSER` → `EntryRefusedHandler` **[013]** (refetch, the card shows its state — 013 RF-6.3), any other origin → `TenantStateResolver` **[014]** (`access-status`: `SUSPENDED_TENANT` or `DELETING_TENANT` → N20 route with the deleting wording; `NOT_PAUSED` or `access.status-unavailable` → `/workspaces` re-derived); `access.denied-scope` → the tenant is entered without project content and N28 `ProjectAccessDeniedPanel` **[006]** renders in place of the project; `auth.enrollment.required` → `EnrollmentGateHandler` **[003]**; network error, timeout or 5xx → `CHECK_FAILED`. Never returns to the old scope. | RF-7.1–7.7, LC-20–LC-26 |
| `shared/api` | axios transport **[001/006]** — extended | Request interceptor: attach the current epoch and its abort signal to every request except the identity allowlist. Response interceptor (first in the chain): a response, error or problem whose captured epoch ≠ current → reject with the internal `ScopeSuperseded`, which TanStack Query and the error handlers ignore — never written to the cache, never shown, never retried. Mutations are never auto-retried (unchanged). | RF-8.1, RNF-2, LC-17, LC-18 |
| `shared/realtime` | STOMP client **[006]** — extended | `RealtimeScope.disconnect()` closes every subscription of the current scope at step 2; every frame handler is wrapped with the epoch captured at subscribe time and drops a frame whose epoch ≠ current; new subscriptions open only after step 5, each authorised by the server at subscribe time. | RF-5.1, RF-8.1, RF-8.2, LC-17 |
| `shared/errors` | `TenantStateResolver` **[014]**, `EntryRefusedHandler` **[013]**, `DeniedScopeHandler` **[006]**, `EnrollmentGateHandler` **[003]** | Reused unchanged as the destinations of `ScopeEntryCheck`; `ScopeSuperseded` is registered as a silent error. | RF-7.3–7.5, RF-7.7 |
| `app/shell` | `TenantSwitchDirtyDialog` (implements `DirtyGuardPresenter`) | N24: "Save before switching?", the statement naming the item(s) and "A pending request remains bound to its original tenant and project.", the line "AUTHORITY / … switch only after an explicit choice.", actions "Save & switch", "Stay here", "Discard changes & switch". S-59 states: list of several items by label; saving (actions disabled, announced); closes on any cancelling outcome. Modal with focus trap; Escape, close and backdrop = `STAY`; focus returns to the starting control. | RF-3.1, RF-3.2, RF-3.4, RF-3.5, RF-4.2, RNF-8, LC-2, LC-14 |
| | `SwitchingStatus` (new) | While `SWITCHING`: masks the content area, shows only "Switching to <target name>" in a polite live region; nothing of either scope is rendered. | RF-2.3, LC-36 |
| | `SwitchCheckFailedPanel` (new) | While `CHECK_FAILED`: a safe problem with the correlation id, "Retry" (re-runs step 4 only), "Choose workspace" (→ `/workspaces`) and "Sign out" (`SignOutCoordinator`, entry `SHELL` **[007]**, nothing to guard). Built as the S-59 assumption beside N28. | RF-7.6, LC-25 |
| | `PendingRequestNotice` (new) | Non-blocking notice shown with the switching state when the ledger received entries at step 2; text only, no action. | RF-5.2, LC-16 |
| | `TenantSelector` (WORKSPACE ⌄) | Context-header trigger showing the current tenant; activation runs exactly `SwitchWorkspaceAction` **[013]** (guard, then `/workspaces`, scope kept until an entry succeeds); also an item of the M04 menu. Built as the S-58 assumption. | RF-1.1, RF-1.2, RF-1.5, LC-1–LC-3 |
| | `ProjectSelector` (PROJECT ⌄) | Context-header trigger; opening it fetches `GET /me/projects` (no cache beyond the open list) and renders a keyboard-operable list: the lines with own grants, "canRun only · no project role yet", the current project marked, or "No project grants yet — workspace administration does not grant project content"; choosing a line → `useGuardedSwitch({tenantId, projectId}, SELECTOR)`; choosing the current line → nothing. Mobile: M04 menu item opening the same list. Built as the S-58 assumption. | RF-1.3–1.5, LC-4–LC-6, LC-37 |
| | `ScopeNavigationGuard` (implements nothing; consumes `NavigationInterceptPort`) | On every pending in-app navigation (link, Back, Forward, a deep link opened in the tab, a notification link) compares the route's tenant / project with the current scope: equal → release; different → hold, run `useGuardedSwitch(routeScope, ROUTE)`; `STAY` or a cancelled guard → restore the previous address; `ENTERED` → release to the route. While the registry is non-empty, asks the browser's leave-page confirmation. A full page load starts the tab with a fresh epoch and runs steps 4–5 only. | RF-9.1–9.4, LC-27–LC-30 |
| | `NavigationInterceptPort` (in `shared/security`) and its History-API adapter (in `app/shell`) | `onPendingNavigation(handler)`, `hold()`, `release()`, `restore(previous)`, `setLeavePrompt(on)`; the adapter wraps in-app link navigation, reacts to `popstate` by re-pushing the previous entry while the guard runs, and toggles the leave-page event. Any routing mechanism decided under A-67 must implement this port. | RF-9.1–9.3 |
| | `SignOutAction` **[007]**, `SwitchWorkspaceAction` **[013]** | Unchanged; both reach the same registry and presenter. | RF-1.1, RF-11.2 |
| Shared used | `ProtectedBoundary` **[006]** (session end during a switch, D-d), `SessionState` **[006]**, `AccessCredentialHolder` **[006]**, `shared/i18n` (bundle keys for every N24 state, the notice, the switching status, the list lines and the panel in EN / ES / FR), `shared/ui` (`Dialog` with focus trap, `LiveRegion`, `Listbox`, `Button`, 44 px targets). | RF-11.1, RNF-8, RNF-9 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | Entry endpoint takes every origin | `useGuardedSwitch` called by `TenantSelector`, `SwitchWorkspaceAction` **[013]**, `useEnterTenant` **[013]**, acceptance exit **[008]**, creation **[012]**, `ProjectSelector`, `ScopeNavigationGuard` |
| RF-1.2 | — | `TenantSelector` = `SwitchWorkspaceAction` **[013]**; scope kept until `ENTERED` |
| RF-1.3 | `MyProjectsQuery`, `MyProjectsController` | `ProjectSelector` list, fetched on open |
| RF-1.4 | — | Coordinator step (0) equality check |
| RF-1.5 | — | M04 menu items for both selectors |
| RF-2.1 | Entry derivation is step 4 | Coordinator steps (1)–(5) in order |
| RF-2.2 | Project-level derivation on every project switch | Same coordinator path for project targets; full `ScopeDrain` |
| RF-2.3 | — | `SwitchingStatus` |
| RF-2.4 | — | Coordinator stops at the failing step; no restore of the old scope |
| RF-3.1 | — | `TenantSwitchDirtyDialog` via `DirtyGuardPresenter.ask` |
| RF-3.2 | — | Dialog lists every registry entry |
| RF-3.3 | — | Empty registry → no guard |
| RF-3.4 | — | `STAY`, Escape, close → return; focus restored |
| RF-3.5 | — | `DISCARD` → `discard()` all → step 2 |
| RF-3.6 | — | Registration rules of `UnsavedWorkRegistry` |
| RF-3.7 | Server decides at step 4 and later | Presenter returns a choice only |
| RF-4.1 | Each save endpoint checks its own scope (owning features) | `GuardSaveRunner` uses each item's stamp and operation identity |
| RF-4.2 | — | Dialog saving state; old scope kept |
| RF-4.3 | — | All `SAVED` → step 2 |
| RF-4.4 | `412` / `409` from the owning save | `CONFLICT` → cancel; item opens N25 |
| RF-4.5 | Field-path problems (F-X-03) | `INVALID` → cancel; field problems |
| RF-4.6 | Operation-status lookup (FR-X-022) | `INDETERMINATE` → cancel; item lookup; no resubmission |
| RF-4.7 | — | Acknowledged items deregister; others stay; cancel |
| RF-4.8 | `editSeq` capture (FR-AUT-033) | `SUPERSEDED_EDIT` → cancel; guard again next time |
| RF-5.1 | — | `ScopeAbortRegistry.abortCurrent()`, `RealtimeScope.disconnect()` |
| RF-5.2 | Command continues server-side | Ledger entry, `PendingRequestNotice`; no delay |
| RF-5.3 | Operation-status lookup of the owning feature | `PendingCommandLedger` read by the owning screen |
| RF-5.4 | Accepted jobs untouched (run-scoped authority) | No run endpoint called by the switch |
| RF-6.1 | — | `ScopeEpoch.advance()` **[001]** |
| RF-6.2 | — | `ScopeDrain`: queries, `AuthorityStore` **[009]**, drafts, registry, object URLs |
| RF-6.3 | — | Identity allowlist kept |
| RF-7.1 | `TenantEntryUseCase` **[013]** with `projectId`; `GET me/authority` **[009]** | `ScopeEntryCheck` sends requested scope only |
| RF-7.2 | `ENTERED` | Step 5 |
| RF-7.3 | `access.entry-refused` (membership) | `EntryRefusedHandler` **[013]** from the chooser; `TenantStateResolver` **[014]** → `/workspaces` elsewhere |
| RF-7.4 | `access.entry-refused` (tenant not active); `access-status` **[014]** | `TenantStateResolver` → N20 (deleting wording for `DELETING_TENANT`); origin `CHOOSER` → `EntryRefusedHandler` **[013]**, the refreshed card (spec 016 RF-7.4 exception, 013 RF-6.3) |
| RF-7.5 | `access.denied-scope`, `PROJECT_ENTRY_REFUSED` | N28 **[006]** inside the tenant |
| RF-7.6 | — | `CHECK_FAILED` → `SwitchCheckFailedPanel` |
| RF-7.7 | `auth.enrollment.required` **[003]** | `EnrollmentGateHandler` **[003]** |
| RF-8.1 | — | Epoch check in axios interceptor and STOMP frame wrapper |
| RF-8.2 | C10 derivation on every request, subscription, preview and download | Requests carry requested scope only |
| RF-8.3 | Derivation independent of client state | Drain followed by server check; no cached item read before step 5 |
| RF-9.1 | — | `ScopeNavigationGuard` through `NavigationInterceptPort` |
| RF-9.2 | — | `restore(previous)` |
| RF-9.3 | — | Fresh epoch on load; leave-page prompt while registry non-empty |
| RF-9.4 | Route identifiers are requested scope | Same check as any switch |
| RF-10.1 | — | Per-tab epoch, registry, ledger and streams; no cross-tab channel |
| RF-10.2 | Refusal on the other tab's next request | That tab's `DeniedScopeHandler` **[006]** / `TenantStateResolver` **[014]** |
| RF-10.3 | — | Session sharing per plans 006 / 007 unchanged |
| RF-11.1 | Session end per plan 006 | `ProtectedBoundary` **[006]**; coordinator abandons the run |
| RF-11.2 | — | `SignOutCoordinator` **[007]** awaits `UnsavedWorkRegistry` and the same presenter |
| RF-12.1 | `TENANT_ENTRY_REFUSED` **[013]**, `PROJECT_ENTRY_REFUSED` | Origin field in the entry body |
| RF-12.2 | — | `ScopeDrain`; nothing written to storage |
| RF-12.3 | RFC 9457 safe codes **[001 D12]** | Panels show code and correlation id only |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · One `GuardedSwitchCoordinator` per tab runs the five steps for every entry point; everything else calls `useGuardedSwitch`.**
*Why.* FR-IDN-130 fixes one order and decision Q1 lists six entry points; one sequence makes "no entry point skips a step" a property of the code shape rather than of six reviews. Plans 007, 012 and 013 already call a guard hook or `useGuardedSwitch`; this plan provides it.
*Discarded.* Per-entry implementations (chooser, selectors, navigation each draining on their own) — six chances to reorder or skip a step. A global event bus — the order would depend on listener registration.
*Covers.* RF-1.1, RF-1.4, RF-2.1–2.4, RNF-1, LC-1, LC-4, LC-6.

**D2 · Unsaved work is an `UnsavedWorkRegistry` of save / discard callbacks with a scope stamp; Zustand keeps only the editor forks.**
*Why.* Decision Q3 counts forms, dialogs and the chat composer as unsaved work, but P11 limits Zustand to deliberate editor forks. A registry of callbacks lets any screen join the guard without moving its state, and the stamp `{identityId, tenantId, projectId, baseETag, editSeq}` already defined for drafts **[006 D10]** gives FR-IDN-131 for every item. The same registry is the hook plan 007's sign-out awaits, so sign-out and switch cannot disagree.
*Discarded.* Moving every form into Zustand — violates P11 and creates a tenant-global store (FR-AUT-031). Detecting dirtiness from the DOM or by comparing form values generically — cannot save, only guess. A second registry for sign-out — two guards.
*Covers.* RF-3.2, RF-3.3, RF-3.6, RF-4.1, RF-11.2, LC-7, LC-15, LC-34.

**D3 · The guard dialog lives in `app/shell` behind a `DirtyGuardPresenter` port declared in `shared/security`.**
*Why.* The FE DAG forbids `shared` importing `app` or features, yet the coordinator in `shared` must ask the question and N24's wording is shell copy. A port implemented at bootstrap keeps imports acyclic and lets tests drive the coordinator with a scripted presenter.
*Discarded.* The dialog in `shared/ui` — domain copy in the generic UI layer. The coordinator in `app/shell` — features (012, 013) could not call it without importing `app`.
*Covers.* RF-3.1, RF-3.4, RF-3.7.

**D4 · Guard saves run from each item's stamp, sequentially; any outcome other than `SAVED` cancels the switch; acknowledged items stay saved.**
*Why.* FR-IDN-131 and decision Q2. Sequential execution makes the outcome list deterministic and keeps saves serialised as FR-AUT-033 asks; reporting all outcomes lets RF-4.7 keep what was acknowledged; `SUPERSEDED_EDIT` reuses the editor's captured-`editSeq` rule so continued typing can never be lost by a switch (RF-4.8).
*Discarded.* Saving under the header's selected scope — the exact leak FR-IDN-131 forbids. Parallel saves — nondeterministic partial states. "Save what succeeds and switch anyway" — rejected by Q2. Auto-retry of a failed save — F-X-02.
*Covers.* RF-4.1–4.8, RNF-3, LC-3, LC-7–LC-12.

**D5 · Step 2 aborts every request through a per-epoch abort signal; aborted mutations go to an identifiers-only `PendingCommandLedger`; nothing is resubmitted and the switch does not wait.**
*Why.* Decision Q4 and D-a: aborting stops the tab waiting while the server command continues (F-X-02); the ledger keeps exactly what the lookup needs (operation and operation identity, original scope) and nothing a user could see, so it does not contradict the drain of RF-6.2; no wait means P13 needs no configured value.
*Discarded.* Blocking the switch until every write settles — a lost acknowledgement would hold it forever. Silent abort with no record — the outcome would become unknowable. A bounded wait — a configured value the specification does not define (D-a).
*Covers.* RF-5.1–5.3, RNF-3, RNF-6, LC-16.

**D6 · Late arrivals are dropped by an epoch check in the transport and in every stream handler, before any cache or view sees them.**
*Why.* FR-IDN-132: clearing the cache is not enough, because a response or a queued frame that resolves after the drain would repopulate it. Checking the epoch captured at request or subscribe time in the first interceptor and in the frame wrapper makes the drop structural. The silent `ScopeSuperseded` error keeps handlers from showing a problem for a response nobody is waiting for.
*Discarded.* Relying on abort alone — a response already in the event loop still resolves. Filtering in each feature — one forgotten screen leaks. Ignoring by tenant id in the payload — "never by payload ID" (FR-IDN-132).
*Covers.* RF-8.1, RNF-2, LC-17, LC-18, LC-38.

**D7 · The drain clears every query key except an explicit identity allowlist, on every switch, project switches included.**
*Why.* Decision Q8 drains tenant-level facts too. An allowlist (session status, own memberships, identity facts, language) fails closed: a feature that forgets a key convention still has its data cleared. `AuthorityStore` **[009]**, drafts **[006]** and object URLs are cleared in the same step so nothing of the old scope survives anywhere in memory.
*Discarded.* A `['scope', …]` key prefix with prefix removal — a key without the prefix would survive. Project-only drain — rejected by Q8. Full `queryClient.clear()` — would also drop identity facts the chooser and the session status need (RF-6.3).
*Covers.* RF-2.2, RF-6.2, RF-6.3, LC-19.

**D8 · Step 4 is a server entry decision on every switch — `POST entry` extended with `projectId`, then `GET me/authority` — and refusals are audited there.**
*Why.* P3 and FR-IDN-130 step 4: the client must not decide from cached grants that a project is reachable. Extending plan 013's `TenantEntryUseCase` gives one endpoint, one audit point (RF-12.1) and one problem vocabulary for every entry point; the project level reuses `access.denied-scope`, which N28 already renders (plan 006). The authority read then loads the store plan 009 defined for exactly this step.
*Discarded.* Deciding project reachability from `AuthorityStore` grants in the client — a client authority decision and no server audit. A separate project-entry endpoint — a second audit path. Skipping the entry call on project switches — Q8 requires the full fetch.
*Covers.* RF-2.2, RF-7.1, RF-7.2, RF-7.5, RF-8.3, RF-12.1, RNF-5, LC-23, LC-24, LC-28, LC-29.

**D9 · Refusal outcomes reuse the existing handlers: the chooser's own refresh when the switch began on the chooser, plan 014's resolver otherwise, N28 for a project, plan 003's gate.**
*Why.* Decision Q5 and the user's decision of 28 October 2026 (a deleting tenant → N20, as specification 014 D-d). `TenantStateResolver` **[014]** already maps `access-status` to N20 (suspended or deleting) or `/workspaces`; on the chooser, specification 013 Q4 already decides that the refreshed card is the answer ("no separate error"), and that card offers "View access status" to N20. Reusing them gives one wording per state.
*Discarded.* A switch-specific error page — rejected by Q5 ("honest state"). Returning to the old scope — rejected by Q5; the old scope is already drained. Routing a deleting tenant to the chooser — superseded by the 28 October decision.
*Covers.* RF-7.3–7.5, RF-7.7, LC-20–LC-24, LC-26.

**D10 · Every piece of switch state is per tab; there is no cross-tab coordination.**
*Why.* Decision Q6 (A-63). The epoch, registry, ledger and streams live in the tab's memory; the server re-derives authority at each request, so another tab learns about a membership change from its own next refusal (plan 006 / 014 handlers). Shared session behaviour stays with plans 006 and 007.
*Discarded.* A broadcast channel or storage events to make tabs follow — rejected by Q6, and storage events would put scope facts in browser storage (FR-X-034). Locking other tabs — rejected by Q6.
*Covers.* RF-10.1–10.3, LC-31, LC-32.

**D11 · Navigation interception goes through one `NavigationInterceptPort` implemented over the standard History API in the shell; the routing mechanism is register item A-67.**
*Why.* Decision Q7 needs every in-app navigation that names another scope to be held, guarded and either released or reverted, plus a leave-page prompt. P10 names no routing mechanism and the README forbids naming a technology outside it; the browser's History API and leave-page event are platform features, not dependencies. Isolating them behind a port lets a routing mechanism decided under A-67 replace the adapter without touching the coordinator.
*Discarded.* Adopting a routing library now — outside P10 without an amendment. Intercepting navigation inside each feature — inconsistent and easy to bypass. Treating route changes as plain filters — rejected by Q7.
*Covers.* RF-9.1–9.4, LC-27–LC-30.

**D12 · The PROJECT list is a server read model fetched when opened.**
*Why.* RF-1.3 requires a derivation on open and exact own grants; the chooser's project-line rules already live on the server (013 D1). A tenant-scoped read reusing `ChooserOrder` **[013]** keeps both lists identical and collation-correct.
*Discarded.* Filtering `AuthorityStore` grants in the client — client-built authority-shaped list and no names source. Reusing the chooser listing — identity-wide, carries every tenant.
*Covers.* RF-1.3, LC-4, LC-5.

**D13 · The switch introduces no configured value.**
*Why.* P13 and D-a: no waiting period, no timeout, no retry count; "Retry" on the check-failed panel is a deliberate action, not an automatic retry. Safe-read bounds remain plan 006's.
*Discarded.* A grace period before draining — a default the specification does not define.
*Covers.* RNF-6.

**D14 · Accessibility is built into the four surfaces: modal guard with focus trap and return, announced switching state without focus theft, listbox semantics for the PROJECT list, one component per surface across widths.**
*Why.* P14 and RNF-8 … RNF-10; one component with breakpoint layouts keeps every action present at every width (S-02), and all copy comes from i18next keys so the language switch changes display only (D-c).
*Discarded.* Separate mobile components — drift. A toast for the switching state — not announced reliably and dismissible mid-switch.
*Covers.* RF-1.5, RF-2.3, RF-3.4, RNF-8–10, LC-36, LC-37.

**D15 · Problems are RFC 9457 with safe codes and a correlation id [001 D12]; the switch adds no new problem code.**
*Why.* F-X-03 and RF-12.3; `access.entry-refused` **[013]**, `access.denied-scope` **[006]** and `auth.enrollment.required` **[003]** cover every refusal, and causes live only in audit.
*Discarded.* Cause-specific codes — the client would branch on authority facts.
*Covers.* RF-7.6, RF-12.1, RF-12.3, RNF-7.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Project-level entry rule over grant / project-state pairs; PROJECT list flags and order. |
| BE integration | Spring Boot test + **real PostgreSQL** (Testcontainers, **test-only**) | Extended `TenantEntryUseCase` and `MyProjectsQuery` through real adapters under `FORCE RLS`; audit rows; no row written but audit. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, absence of other projects, members, counts or content. |
| Concurrency | JUnit 5 with real PostgreSQL | Entry racing a grant removal, a membership revocation and a tenant suspension. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain), fake timers and a controllable transport double | Coordinator order, registry, presenter outcomes, guard-save outcomes, drain contents, epoch drops under scripted orderings of late responses and frames, ledger, navigation port behaviour. |
| FE e2e + accessibility | Playwright for the browser + axe (**test-only**) | Journeys from every entry point in EN / ES / FR at desktop, tablet and mobile widths; two-tab isolation; storage scan; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari, macOS and iOS), versions per the release-support matrix | Guard focus trap and return, announcements of saving and switching, list operation, panel actions. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

The last column is the completion criterion of specification §7 the test proves.

| Spec item | Level | Test case (name) | Criterion |
|---|---|---|---|
| RF-1.1, LC-35 | FE e2e | `switch.everyEntryPointRunsCoordinator` (WORKSPACE ⌄, Switch workspace, chooser Enter, invitation exit, creation, PROJECT ⌄, route) | 3 |
| RF-1.2, LC-1, LC-3 | FE e2e | `tenantSelector.guardThenChooserScopeKeptUntilEntered` | 3 |
| RF-1.3, LC-4, LC-5 | BE integration, API contract, FE unit | `myProjects.ownLiveGrantsOrderedWithFlags`; `myProjects.exposesNothingElse`; `projectSelector.rendersLinesCurrentMarkedNoGrantsLine` | 8 |
| RF-1.4, LC-6 | FE unit | `coordinator.sameScopeIsNoOp` | 3 |
| RF-1.5, LC-37 | FE e2e | `selectors.inM04MenuSameBehaviour` | 12 |
| RF-2.1, RNF-1 | FE unit | `coordinator.stepsRunInOrderNoLoadBeforeEntered` (spy order across guard, abort, disconnect, epoch, drain, entry, authority, load) | 3 |
| RF-2.2, LC-19 | FE unit, BE integration | `projectSwitch.fullDrainIncludingTenantFacts`; `entry.projectLevelDerivationOnEverySwitch` | 3 |
| RF-2.3, LC-36 | FE unit | `switchingStatus.showsTargetNameOnlyAnnouncedPolitely` | 3 |
| RF-2.4 | FE unit | `coordinator.failingStepStopsLaterStepsNoRestore` | 3 |
| RF-3.1 | FE unit | `guard.shownWhenRegistryNonEmptyWithDrawnCopy` | 4 |
| RF-3.2, LC-7 | FE unit | `guard.listsEveryItemOneDialog` | 4 |
| RF-3.3 | FE unit | `guard.notShownWhenNothingUnsaved` | 4 |
| RF-3.4, LC-2, LC-14 | FE unit, e2e | `guard.stayEscapeCloseChangeNothingFocusReturns` | 4 |
| RF-3.5, LC-13 | FE unit | `guard.discardPurgesAllThenDrains` | 4 |
| RF-3.6, LC-15 | FE unit | `registry.formsDialogsComposerRegisterViewStateDoesNot` | 4 |
| RF-3.7 | FE unit | `presenter.returnsChoiceOnlyNoAuthorityCall` | 4 |
| RF-4.1, RNF-3 | FE unit, e2e | `guardSave.submittedUnderItemStampNeverTarget` (request URL, headers and body checked) | 5 |
| RF-4.2 | FE unit | `guard.savingStateDisablesActionsKeepsScope` | 5 |
| RF-4.3 | FE unit | `guardSave.allSavedContinues` | 5 |
| RF-4.4, LC-8 | FE e2e (authoring stub) | `guardSave.conflictCancelsOpensN25DraftIntact` | 5 |
| RF-4.5, LC-9 | FE unit | `guardSave.invalidCancelsFieldProblems` | 5 |
| RF-4.6, LC-10 | FE unit | `guardSave.indeterminateCancelsLooksUpNeverResubmits` | 5 |
| RF-4.7, LC-11 | FE unit | `guardSave.partialFailureKeepsAcknowledgedCancels` | 5 |
| RF-4.8, LC-12 | FE unit | `guardSave.typingDuringSaveSupersedesGuardAgain` | 5 |
| RF-5.1 | FE unit | `drain.abortsAllScopedRequestsAndClosesStreams` | 6 |
| RF-5.2, LC-16 | FE unit, e2e | `drain.pendingMutationToLedgerNoticeNoDelay`; `launch.pendingAtSwitchNeverResent` | 6 |
| RF-5.3 | FE unit | `ledger.ownerLookupByOperationIdInOriginalScope` | 6 |
| RF-5.4, RNF-4 | FE e2e, BE integration | `switch.callsNoRunEndpoint`; `acceptedRun.keepsPinsAfterSwitch` | 6 |
| RF-6.1 | FE unit | `epoch.advancesPerTabCoversTenantAndProject` | 7 |
| RF-6.2, LC-38 | FE unit, e2e | `drain.clearsQueriesAuthorityDraftsRegistryObjectUrls`; `afterSwitch.noOldScopeInMemoryOrDom` | 7 |
| RF-6.3 | FE unit | `drain.keepsIdentityAllowlistOnly` | 7 |
| RF-7.1, RNF-5 | BE integration, FE unit | `entry.derivesFromCurrentFactsNotRequestBody`; `entryCheck.sendsRequestedScopeOnly` | 8 |
| RF-7.2 | FE unit | `entryCheck.enteredLoadsTarget` | 8 |
| RF-7.3, LC-20 | BE integration, FE e2e | `entry.revokedOrSuspendedMembershipRefused`; `refused.fromSelectorGoesToChooserRederived`; `refused.fromChooserRefreshesCard` | 8 |
| RF-7.4, LC-21, LC-22 | FE e2e (014 resolver) | `refused.suspendedTenantShowsN20`; `refused.deletingTenantShowsN20DeletingWording`; `refused.fromChooserRefreshesCard` (chooser origin) | 8 |
| RF-7.5, LC-23, LC-24 | BE integration, FE e2e | `entry.projectWithoutGrantOrDeletedDeniedScopeAudited`; `deniedProject.rendersN28InsideTenant` | 8 |
| RF-7.6, LC-25 | FE unit, e2e | `entryCheck.failureShowsPanelNothingOfEitherScope`; `panel.retryRerunsStep4Only` | 8 |
| RF-7.7, LC-26 | FE e2e (003 stub) | `entry.ownerWithoutFactorMeetsGateOnArrival` | 8 |
| RF-8.1, RNF-2, LC-17, LC-18 | FE unit (property-style, randomised orderings) | `lateResponse.droppedByEpochNeverCached`; `lateFrame.droppedByEpochNeverRendered` | 7 |
| RF-8.2, LC-29 | API contract, BE integration | `scopedEndpoints.ignorePayloadIdentifiersDeriveFromC10`; `download.authorisedAtRequestTime` | 8 |
| RF-8.3 | FE unit | `noCachedItemReadBeforeEntered` | 8 |
| RF-9.1, LC-27 | FE e2e | `backButton.toOtherTenantRunsGuardedSwitch` | 3 |
| RF-9.2 | FE e2e | `navigationStay.restoresPreviousAddress` | 4 |
| RF-9.3, LC-30 | FE e2e | `reload.withUnsavedWorkAsksLeavePrompt`; `reload.freshEpochStepsFourAndFive` | 3 |
| RF-9.4, LC-28 | FE e2e, API contract | `deepLink.foreignTenantRefusedNeverShown` | 8 |
| RF-10.1, LC-31 | FE e2e (two pages, one context) | `twoTabs.switchInOneLeavesOtherUntouched` | 9 |
| RF-10.2, LC-32 | FE e2e, concurrency | `twoTabs.revocationMetOnOtherTabsNextRequest` | 9 |
| RF-10.3 | FE e2e (006 / 007 stubs) | `twoTabs.sessionBehaviourUnchanged` | 9 |
| RF-11.1, LC-33 | FE e2e (006 stub) | `sessionEndsDuringGuard.abandonsSwitchReturnsOriginalScope` | 10 |
| RF-11.2, LC-34 | FE e2e (007) | `signOut.usesSameGuardAndRegistry` | 10 |
| RF-12.1 | BE integration | `audit.oneRowPerRefusalAnyOriginNoContent` | 11 |
| RF-12.2, RNF-7 | FE e2e | `storage.scanAfterSwitchAllowlistOnly` | 11 |
| RF-12.3 | API contract, FE unit | `problems.safeCodeAndCorrelationIdOnly` | 11 |
| RNF-6 | Startup | `switch.introducesNoConfigurationProperty` | — |
| RNF-8 | FE e2e + axe, manual | `a11y.guard.{en,es,fr}.{desktop,tablet,mobile}`; `a11y.projectList.*`; `a11y.checkFailedPanel.*`; NVDA / VoiceOver checklist | 12 |
| RNF-9 | FE e2e | `allSurfaces.existInEnEsFr`; `languageSwitch.duringSwitchDisplayOnly` | 12 |
| RNF-10 | FE e2e | `reflow.everyActionPresentAtEachWidth` | 12 |
| US-1 … US-9 | FE e2e | journeys above, one per story | 1 |
| US-10 | all adversarial cases above | old-scope read after switch, late events, cross-tenant save, address editing, stale page entry, resend or cancel by switch | 1 |
| Every RF | — | each row above fails before the feature and passes after | 2 |

### 5.3 Not tested here
The derivation and permission versions (plan 009); the chooser page and its cards (013); N20 and `access-status` (014); N28 and the denied-scope handler (006); session expiry, re-authentication and R34 (006); sign-out internals (007); the enrollment wizard (003); each feature's save, conflict resolution N25 and operation-status lookup (their own plans); accepted-run behaviour (execution area). This plan tests only that the switch calls them in the right order with the right scope and never bypasses them.

### 5.4 Exit criteria
Specification §7 items 1–12 map to the "Criterion" column; item 13 is satisfied when the register items of §6 are resolved or explicitly deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-63 scope epoch boundary (decided) | Per-tab epoch over tenant + project; route scope request-only; full drain on project switch (D6, D7, D10, D11). | None expected; it is the decided rule. |
| A-67 client routing and navigation interception (open, raised by this plan) | `NavigationInterceptPort` over the History API and the leave-page event (D11). | The chosen routing mechanism implements the port; the coordinator and the guard are unchanged. |
| A-03 board 3 / FE section duplication | The FE section's wording is followed; board 3 §03 used where it is silent. | Re-check the step list against the transplanted board. |
| S-58 header selectors (decided) | `TenantSelector` → chooser; `ProjectSelector` list with the 013 Q2 lines; M04 menu items. | Adjust layout and copy; behaviour unchanged. |
| S-59 N24 states (decided) | Several-items list, saving state, closing on cancel, pending notice, switching status, check-failed panel. | Adjust copy and layout. |
| S-07 app-shell identity area | Selectors in the context header; `SwitchWorkspaceAction` and `SignOutAction` stay in the footer identity block. | Move the actions into the designed user menu; the coordinator is unchanged. |
| S-02 responsive counterparts | One component per surface reflowed (D14). | Replace the breakpoint layout with the designed counterpart. |
| A-35 persistence tooling | Audit adapter and queries against the placeholder API. | Rename base types; no use case changes. |

**Cross-plan items raised by this plan** (reported, not applied; to be folded in when those plans are next revised):
1. Plan 013 — `TenantEntryUseCase` / `TenantEntryController` gain the optional `projectId`, the `entry` origin field and the `PROJECT_ENTRY_REFUSED` audit event; `useEnterTenant` becomes a caller of `useGuardedSwitch`.
2. Plan 001 — the scope epoch is per tab and covers tenant + project; the axios transport gains the abort signal and the first-position epoch check. Plan 006 — the STOMP client of `shared/realtime` gains `RealtimeScope.disconnect()` and the frame wrapper (plan 006 tags that client **[001]**, but plan 001 does not define it — a mistag to correct in plan 006).
3. Plan 009 — `AuthorityStore` is cleared by `ScopeDrain` and loaded by `ScopeEntryCheck` (as plan 009 already anticipates).
4. Plan 007 — `SignOutCoordinator` step (1) awaits `UnsavedWorkRegistry` and `DirtyGuardPresenter`, the "guarded-switch dirty guard hook" it names.
5. Plan 012 — `useGuardedSwitch` now exists; the interim "scope-epoch sequence" fallback can be removed.
6. Plan 008 — the acceptance exit calls `useGuardedSwitch({tenantId, projectId?}, INVITATION)`.
7. Previews and downloads (their owning plans) create object URLs through `ScopedObjectUrls`.

The specification's derived rules are built, not assumed: no waiting period (D-a → D5, D13), Escape and close = Stay (D-b → presenter), language display-only (D-c → D14), session end abandons the switch (D-d → coordinator and `ProtectedBoundary` **[006]**), enrollment gate on arrival (D-e → `ScopeEntryCheck`).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.5) | `useGuardedSwitch`, `TenantSelector`, `ProjectSelector`, `MyProjectsQuery`, coordinator step (0) | D1, D12, D14 | RF-1 rows |
| RF-2 (2.1–2.4) | `GuardedSwitchCoordinator`, `SwitchingStatus`, extended entry | D1, D7, D8 | RF-2 rows |
| RF-3 (3.1–3.7) | `UnsavedWorkRegistry`, `DirtyGuardPresenter`, `TenantSwitchDirtyDialog` | D2, D3 | RF-3 rows |
| RF-4 (4.1–4.8) | `GuardSaveRunner`, stamped items, dialog saving state | D4 | RF-4 rows |
| RF-5 (5.1–5.4) | `ScopeAbortRegistry`, `RealtimeScope`, `PendingCommandLedger`, `PendingRequestNotice` | D5 | RF-5 rows |
| RF-6 (6.1–6.3) | `ScopeEpoch` [001], `ScopeDrain`, `ScopedObjectUrls` | D7 | RF-6 rows |
| RF-7 (7.1–7.7) | `ScopeEntryCheck`, extended `TenantEntryUseCase` [013], handlers [003/006/013/014], `SwitchCheckFailedPanel` | D8, D9 | RF-7 rows |
| RF-8 (8.1–8.3) | axios epoch check, STOMP frame wrapper, server derivation | D6, D8 | RF-8 rows |
| RF-9 (9.1–9.4) | `ScopeNavigationGuard`, `NavigationInterceptPort` | D11 | RF-9 rows |
| RF-10 (10.1–10.3) | per-tab state, existing handlers | D10 | RF-10 rows |
| RF-11 (11.1–11.2) | `ProtectedBoundary` [006], `SignOutCoordinator` [007] | D2 | RF-11 rows |
| RF-12 (12.1–12.3) | audit adapter, `ScopeDrain`, safe problems | D8, D15 | RF-12 rows |
| RNF-1 | coordinator | D1 | `coordinator.stepsRunInOrderNoLoadBeforeEntered` |
| RNF-2 | epoch checks, drain | D6, D7 | `lateResponse.*`, `lateFrame.*` |
| RNF-3 | `GuardSaveRunner`, ledger | D4, D5 | `guardSave.submittedUnderItemStampNeverTarget` |
| RNF-4 | no run call | D5 | `switch.callsNoRunEndpoint` |
| RNF-5 | entry derivation | D8 | `entry.derivesFromCurrentFactsNotRequestBody` |
| RNF-6 | no configuration | D13 | startup |
| RNF-7 | drain, storage allowlist | D7, D15 | `storage.scanAfterSwitchAllowlistOnly` |
| RNF-8, RNF-9, RNF-10 | dialog, status, list, panel; i18n; reflow | D14 | a11y, language, reflow rows |
| LC-1 … LC-38 | per the §5.2 rows that name them | — | §5.2 |
| US-1 … US-10 | journeys and adversarial cases | — | §5.2 |

Every EARS criterion RF-1.1 … RF-12.3 appears in §3; every RNF, LC and US appears in §5.2; no element of §2 lacks a criterion, and no criterion lacks an element, a decision and a test.
