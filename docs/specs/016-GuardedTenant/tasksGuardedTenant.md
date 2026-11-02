# Tasks 016 — Guarded tenant and project switch

**Feature** The transition that moves a signed-in person from one working scope (a tenant and, within it, a project) to another, protecting unsaved work, carrying nothing across the boundary and loading the new scope only after its authority has been derived afresh · **Tasks version** 0.1 · **Date** 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specGuardedTenant.md` v0.3 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planGuardedTenant.md` v0.1 (modules §2, decisions D1–D15, tests §5, register assumptions and cross-plan items §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2, and where §5.2 names no test the line states a directly observable condition. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[003]** / **[006]** / **[007]** / **[008]** / **[009]** / **[011]** / **[012]** / **[013]** / **[014]** extend an element that plan builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. The STOMP client is tagged **[006]**, where it is defined (plan 006 tags it [001], a mistag reported in plan 016 §6). Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API is the A-35 placeholder; the routing mechanism is reached only through `NavigationInterceptPort` (A-67).

**Chooser-origin refusal (decided 29 October 2026).** For a switch started from the chooser's "Enter", spec 016 v0.3 RF-7.4 keeps the person on the refreshed chooser for a suspended or deleting tenant, as spec 013 Q4 / RF-6.3; every other origin goes to N20. Plan 016 D9 and T-25 build exactly that.

Phases: **A** foundations · **B** back end · **C** front-end primitives (`shared/*`) · **D** shell surfaces (`app/shell`) · **E** entry-point rewiring · **F** verification suites · **G** closing. Phase C depends only on T-02 and T-03 (contracts and copy keys are fixed by the plan) and may run alongside phase B; phase D needs the primitives it renders; phase E needs T-28.

---

## A · Foundations

- [ ] **T-01 · Placement and import rules** — the back-end additions live in the `access` context of plan 009 (`application`: the extended `TenantEntryUseCase` **[013]** and `MyProjectsQuery`; `domain`: the project-level entry rule); an ArchUnit test asserts that this feature adds no repository, no table, no write port other than the audit port, no `@ConfigurationProperties`, and reaches `project` and `audit` only through their published ports. A front-end import-boundary test (Vitest, scanning import statements) asserts that `shared/security/switch`, `shared/api` and `shared/realtime` never import from `app` or from any feature module. *(plan §2.1 header, §2.2, D3, D13; RNF-6)*
  **Done when:** both boundary tests pass and `switch.introducesNoConfigurationProperty` passes (the Spring context starts and the front-end build completes with no property or configured value added for this feature).

- [ ] **T-02 · Scope-entry contracts** — in `src/contracts/scopeEntry`: request `{projectId?, entry: CHOOSER | SELECTOR | ROUTE | INVITATION | CREATION}`, response `{state: ENTERED, tenantId, tenantName, tenantRole, projectId?, projectName?}`, my-projects response `{lines: [{projectId, projectName, grants, canRunOnly}], noGrants}`, and the three problem codes `access.entry-refused` **[013]**, `access.denied-scope` **[006]**, `auth.enrollment.required` **[003]** with correlation id; runtime guards reject any field beyond the contract. *(plan §2.1 API, D12, D15; RF-1.3, RF-7.1, RF-12.3)* — after T-01
  **Done when:** guard unit tests accept the exact shapes and reject a response carrying an extra field (a member, a count, another project), and the types compile against plan 013's existing entry contract.

- [ ] **T-03 · Copy keys in EN / ES / FR** — i18next bundle keys for N24 and its S-59 states ("Save before switching?", the item statement, "A pending request remains bound to its original tenant and project.", the authority line, "Save & switch", "Stay here", "Discard changes & switch", the several-items list, the saving state), the switching status ("Switching to <target>"), the pending-request notice, the PROJECT list lines ("canRun only · no project role yet", "No project grants yet — workspace administration does not grant project content"), the check-failed panel ("Retry", "Choose workspace", "Sign out") and the two selector labels. *(plan §2.2 Shared used, D14; RNF-9, RF-3.1, RF-5.2, RF-1.3)* — after T-01
  **Done when:** a key-parity test shows every key present and non-empty in EN, ES and FR, and no component of this feature contains a literal user-facing string.

## B · Back end

- [ ] **T-04 · Project-level entry rule** — pure rule `(projectExists, projectDeleted, GrantSet) → GRANTED | NO_PROJECT_GRANT | PROJECT_GONE`, using `GrantSet` **[009]** (any live grant — Author, Reviewer, Viewer or `canRun` — is a grant); no Spring. *(plan §2.1 `TenantEntryUseCase` extension, D8; RF-7.5, LC-23, LC-24)* — after T-01
  **Done when:** a JUnit unit test over every combination of project state and grant set returns the expected outcome, including `canRun`-only as `GRANTED`.

- [ ] **T-05 · `TenantEntryUseCase` [013] — project level and origin** — accept the optional `projectId` and the `entry` origin; run the tenant-level `DeriveAuthorityUseCase` **[009]** exactly as before; when granted and `projectId` is present, read `ProjectExistsPort` **[009]** and the identity's grants and apply T-04; return `ENTERED{tenantId, projectId?}` or `Denied(NO_PROJECT_GRANT | PROJECT_GONE)`; nothing read from the request beyond the two identifiers; writes nothing but the audit call (T-06). *(plan §2.1, D8; RF-2.2, RF-7.1, RF-7.2, RF-7.5, RF-7.7, RF-8.3, RNF-5, LC-19, LC-26)* — after T-04
  **Done when:** `entry.projectLevelDerivationOnEverySwitch` and `entry.derivesFromCurrentFactsNotRequestBody` pass with in-memory doubles, and plan 013's existing entry tests still pass unchanged.

- [ ] **T-06 · Audit adapter — `PROJECT_ENTRY_REFUSED` and the origin field [013]** — add the event `PROJECT_ENTRY_REFUSED{tenantId, projectId, cause}` and the `entry` origin on `TENANT_ENTRY_REFUSED`, each with identity id, correlation id and timestamp, appended in the entry transaction; never a name, a grant or content; one row per refusal whatever the origin. *(plan §2.1 audit adapter, D8, D15; RF-12.1, LC-20, LC-23)* — after T-05
  **Done when:** `audit.oneRowPerRefusalAnyOriginNoContent` passes on real PostgreSQL for a tenant refusal and a project refusal from each origin.

- [ ] **T-07 · `MyProjectsQuery`** — for the session identity (`SessionAssertionPort` **[006]**) and a requested tenant: tenant-level derivation first (refusal → `access.entry-refused`); then the identity's own live project grants with names through `ProjectNamesPort` **[013]**, ordered by the project-line comparator of `ChooserOrder` **[013]** under the locale collation, each line with `canRunOnly`, and `noGrants` when none; no other project, count, source, run or member. *(plan §2.1 `MyProjectsQuery`, D12; RF-1.3, LC-4, LC-5)* — after T-05
  **Done when:** `myProjects.ownLiveGrantsOrderedWithFlags` passes on real PostgreSQL with three granted projects, a `canRun`-only project and an Admin with no grant.

- [ ] **T-08 · `TenantEntryController` [013] — extended contract** — `POST /api/tenants/{tenantId}/entry` with body `{projectId?, entry}`; 200 with `projectId` / `projectName` when entered at project level; 403 `access.denied-scope` for a project refusal; the existing 403 `access.entry-refused`, 403 `auth.enrollment.required` and 401 unchanged; RFC 9457 problems with a correlation id; the server never redirects. *(plan §2.1 API, D8, D15; RF-7.1–7.5, RF-12.3)* — after T-06
  **Done when:** MockMvc contract tests show each status and problem shape, and `problems.safeCodeAndCorrelationIdOnly` passes for the entry endpoint (no cause, name or content in any problem).

- [ ] **T-09 · `MyProjectsController`** — `GET /api/tenants/{tenantId}/me/projects` (session required): 200 `{lines, noGrants}`, 403 `access.entry-refused`, 401 without a session; no query parameter, no paging. *(plan §2.1 API, D12; RF-1.3)* — after T-07
  **Done when:** `myProjects.exposesNothingElse` passes (MockMvc: no other project, member, count, source or run in the body) and the 403 and 401 cases answer the plan's problem shapes.

- [ ] **T-10 · Entry refusal suite on real PostgreSQL** — integration tests through the real adapters under `FORCE RLS`: revoked membership, suspended membership, suspended tenant, deleting tenant (each `access.entry-refused`), project without grant and deleted project (each `access.denied-scope` with its audit row), owner without a factor (`auth.enrollment.required`). *(plan §5.1 BE integration, D8; RF-7.3, RF-7.4, RF-7.5, RF-7.7, LC-20, LC-21, LC-22, LC-23, LC-24, LC-26)* — after T-08
  **Done when:** `entry.revokedOrSuspendedMembershipRefused` and `entry.projectWithoutGrantOrDeletedDeniedScopeAudited` pass, and the suspended, deleting and enrollment-gate cases answer their codes with no project name in the response.

- [ ] **T-11 · Entry concurrency** — JUnit with real PostgreSQL: an entry request racing a grant removal, a membership revocation and a tenant suspension; every interleaving either enters with facts current at the decision or refuses; none enters on facts already withdrawn when its transaction committed. *(plan §5.1 concurrency, D8; RF-7.1, RF-10.2, LC-20, LC-32)* — after T-10
  **Done when:** the concurrency test passes over repeated runs (each run's outcome matches the facts committed before the entry transaction) and no run returns `ENTERED` after the withdrawing transaction committed first.

- [ ] **T-12 · C10 contract checks for the new scope** — contract and integration tests that scoped endpoints derive tenant and project from the C10 context and ignore identifiers in a payload or route that disagree, and that a preview or download is authorised at request time, not by a previously issued identifier. *(plan §5.2, D8; RF-8.2, LC-29)* — after T-10
  **Done when:** `scopedEndpoints.ignorePayloadIdentifiersDeriveFromC10` and `download.authorisedAtRequestTime` pass.

## C · Front-end primitives (`shared/*`)

- [ ] **T-13 · `ScopeEpoch` [001] — per tab, tenant + project** — one counter per tab covering the pair tenant + project; `advance()` returns the new value; every scoped request and subscription captures it; never shared between tabs, never persisted. *(plan §2.2 `ScopeEpoch`, D6, D10; RF-6.1, RF-10.1)* — after T-02
  **Done when:** `epoch.advancesPerTabCoversTenantAndProject` passes (a project-only change advances it; two tab instances hold independent values; nothing is written to browser storage).

- [ ] **T-14 · `ScopeAbortRegistry`** — one `AbortController` per epoch; `currentSignal()` for the transport; `abortCurrent()` aborts safe reads and pending mutations alike and opens a fresh controller for the next epoch. *(plan §2.2, D5; RF-5.1)* — after T-13
  **Done when:** a Vitest unit test shows requests holding the old signal are aborted, requests issued afterwards hold a fresh, unaborted signal, and aborting never issues a request.

- [ ] **T-15 · axios transport [001/006] — epoch, signal and the silent `ScopeSuperseded`** — request interceptor attaches the current epoch and the registry signal to every request except the identity allowlist (session status **[006]**, `me/memberships` **[013]**, identity facts, language); a first-position response interceptor rejects any response, error or problem whose captured epoch differs from the current one with the internal `ScopeSuperseded`, registered in `shared/errors` as silent; mutations still never auto-retry. *(plan §2.2 axios transport, D6; RF-8.1, RNF-2, LC-17, LC-18)* — after T-14
  **Done when:** `lateResponse.droppedByEpochNeverCached` passes under scripted orderings (a response resolving after `advance()` never reaches the TanStack Query cache, never renders and raises no visible problem).

- [ ] **T-16 · STOMP client [006] — `RealtimeScope.disconnect()` and the epoch frame wrapper** — `disconnect()` closes every subscription of the current scope; every frame handler is wrapped with the epoch captured at subscribe time and drops a frame whose epoch differs; no subscription is opened while the coordinator is between steps 2 and 5. *(plan §2.2 `shared/realtime`, D6; RF-5.1, RF-8.1, RF-8.2, LC-17)* — after T-13
  **Done when:** `lateFrame.droppedByEpochNeverRendered` passes (a queued frame delivered after `advance()` reaches no handler and no view).

- [ ] **T-17 · `PendingCommandLedger`** — per tab, in memory, identifiers only: `{operation, operationId, tenantId, projectId, at}` for each mutation aborted at step 2; `record()`, `pendingFor(scope)`, `markLookedUp(operationId)`; exposes whether the last drain recorded entries (for the notice); never stores a body or content; never resubmits. *(plan §2.2, D5; RF-5.2, RF-5.3, RNF-3, LC-16)* — after T-14
  **Done when:** `ledger.ownerLookupByOperationIdInOriginalScope` passes (the owning screen reopened in the original scope finds the entry by operation id and looks it up; no request with the same operation is sent again).

- [ ] **T-18 · `ScopedObjectUrls`** — registry through which previews and downloads create object URLs; `revokeAll()` revokes every URL created in the current epoch. *(plan §2.2, D7; RF-6.2)* — after T-13
  **Done when:** a Vitest unit test shows every URL created through the registry is revoked by `revokeAll()` and no URL survives an epoch advance.

- [ ] **T-19 · `UnsavedWorkRegistry` and `useUnsavedWork`** — per tab, in memory: entries `{id, label, stamp{identityId, tenantId, projectId, baseETag, editSeq}, save(), discard()}` with `SaveOutcome = SAVED | CONFLICT | INVALID | INDETERMINATE | SUPERSEDED_EDIT`; the Zustand editor forks **[006 D10]** register automatically while dirty and deregister when clean; `useUnsavedWork({label, isDirty, save, discard})` registers forms, dialogs with typed input and the chat / proposal composer; filters, sort, scroll and expansion are never registered. *(plan §2.2, D2; RF-3.2, RF-3.3, RF-3.6, LC-7, LC-15)* — after T-13
  **Done when:** `registry.formsDialogsComposerRegisterViewStateDoesNot` passes (a dirty form, a typed dialog and a typed composer register; a changed filter and sort do not; a saved fork deregisters).

- [ ] **T-20 · `DirtyGuardPresenter` port and its scripted double** — port `ask(items) → Promise<SAVE | DISCARD | STAY>`, `showSaving()`, `close()` in `shared/security/switch`; a scripted test double for the coordinator's tests; the port carries labels only and calls no endpoint. *(plan §2.2, D3; RF-3.1, RF-3.4, RF-3.7)* — after T-19
  **Done when:** `presenter.returnsChoiceOnlyNoAuthorityCall` passes (driving the double triggers no network request and no authority read).

- [ ] **T-21 · `GuardSaveRunner` — submission under each item's stamp** — calls each item's `save()` sequentially in registration order; each item submits under its own stamp and operation identity, never under the switch target; all `SAVED` → continue; acknowledged items deregister while the others stay; any other outcome → cancel. *(plan §2.2, D4; RF-4.1, RF-4.3, RF-4.7, RNF-3, LC-3, LC-7, LC-11)* — after T-20
  **Done when:** `guardSave.submittedUnderItemStampNeverTarget`, `guardSave.allSavedContinues` and `guardSave.partialFailureKeepsAcknowledgedCancels` pass (URLs, headers and bodies of each save carry the item's tenant, project and `baseETag`).

- [ ] **T-22 · `GuardSaveRunner` — cancelling outcomes** — `CONFLICT` → cancel and hand the item to its own resolution (N25, owned by the authoring area); `INVALID` → cancel, item shows field problems; `INDETERMINATE` → cancel, item looks up by operation identity, never resubmits; `SUPERSEDED_EDIT` (only the captured `editSeq` acknowledged) → cancel, item stays registered and the guard is shown again on the next attempt; the work stays intact in every case. *(plan §2.2, D4; RF-4.4, RF-4.5, RF-4.6, RF-4.8, LC-8, LC-9, LC-10, LC-12)* — after T-21
  **Done when:** `guardSave.invalidCancelsFieldProblems`, `guardSave.indeterminateCancelsLooksUpNeverResubmits` and `guardSave.typingDuringSaveSupersedesGuardAgain` pass, and a unit test shows `CONFLICT` cancels with the item's resolution callback invoked once.

- [ ] **T-23 · `ScopeDrain`** — step 3 in order: `cancelQueries()`, then `removeQueries` for every key not on the identity allowlist; `AuthorityStore.clear()` **[009]**; purge every Zustand draft **[006]** and every registry entry; `ScopedObjectUrls.revokeAll()`; identity facts and the FR-X-034 storage allowlist survive. *(plan §2.2, D7; RF-6.2, RF-6.3, RF-12.2, RNF-2, RNF-7, LC-19, LC-38)* — after T-18, T-19
  **Done when:** `drain.clearsQueriesAuthorityDraftsRegistryObjectUrls` and `drain.keepsIdentityAllowlistOnly` pass (a query key added by a hypothetical feature without any convention is still removed).

- [ ] **T-24 · `ScopeEntryCheck` — the step-4 calls** — `POST entry {projectId?, entry: origin}` (T-08 contract) and, on `ENTERED`, `GET me/authority` **[009]** into `AuthorityStore`; sends the requested identifiers only; reads nothing from the cache or the page that started the switch. *(plan §2.2, D8; RF-7.1, RF-7.2, RF-8.3, RNF-5)* — after T-15, T-23
  **Done when:** `entryCheck.sendsRequestedScopeOnly` and `entryCheck.enteredLoadsTarget` pass against a transport double.

- [ ] **T-25 · `ScopeEntryCheck` — outcome routing** — `access.entry-refused` with origin `CHOOSER` → `EntryRefusedHandler` **[013]** (refetch; the card shows its state — spec 016 RF-7.4 exception, 013 RF-6.3); with any other origin → `TenantStateResolver` **[014]** (`SUSPENDED_TENANT` or `DELETING_TENANT` → the N20 route, deleting wording for deleting; `NOT_PAUSED` or `access.status-unavailable` → `/workspaces` re-derived); `access.denied-scope` → the tenant entered without project content and N28 `ProjectAccessDeniedPanel` **[006]**; `auth.enrollment.required` → `EnrollmentGateHandler` **[003]**; network error, timeout or 5xx → `CHECK_FAILED`; never a return to the old scope. *(plan §2.2, D9; RF-2.4, RF-7.3, RF-7.4, RF-7.5, RF-7.6, RF-7.7, LC-20, LC-21, LC-22, LC-23, LC-24, LC-25, LC-26)* — after T-24
  **Done when:** a Vitest table test maps each answer and origin to exactly one destination as listed, and no case navigates to the previous scope's route.

- [ ] **T-26 · `GuardedSwitchCoordinator` — steps 0 to 2** — `switchTo(target, origin)`: (0) target equal to the current scope → return; a switch already running → ignore; (1) empty registry → no guard; else `ask()`: `STAY` → return untouched, `DISCARD` → `discard()` all, `SAVE` → `GuardSaveRunner`, cancel on any non-`SAVED`; (2) `ScopeAbortRegistry.abortCurrent()`, `RealtimeScope.disconnect()`, aborted mutations into the ledger, notice flag set; no waiting period. *(plan §2.2 coordinator, D1, D5, D13; RF-1.4, RF-2.1, RF-3.3, RF-3.4, RF-3.5, RF-5.1, RF-5.2, LC-6, LC-13, LC-16)* — after T-16, T-17, T-22
  **Done when:** `coordinator.sameScopeIsNoOp`, `guard.notShownWhenNothingUnsaved`, `guard.discardPurgesAllThenDrains`, `drain.abortsAllScopedRequestsAndClosesStreams` and `drain.pendingMutationToLedgerNoticeNoDelay` pass with the scripted presenter.

- [ ] **T-27 · `GuardedSwitchCoordinator` — steps 3 to 5, state and abandonment** — (3) `ScopeEpoch.advance()` then `ScopeDrain.run()`; (4) `ScopeEntryCheck.run()`; (5) only on `ENTERED` publish the new scope and allow loads and subscriptions; `switchState` `IDLE | GUARDING | SAVING | SWITCHING | CHECK_FAILED`; a failing step stops every later step without restoring the old scope; a project target runs every step; on `SessionState → ENDED` **[006]** during the guard or a guard save the run is abandoned and `ProtectedBoundary` **[006]** takes over (D-d). *(plan §2.2 coordinator, D1, D7, D8; RF-2.1, RF-2.2, RF-2.4, RF-6.1, RF-8.3, RF-11.1, RNF-1, LC-19, LC-33)* — after T-25, T-26
  **Done when:** `coordinator.stepsRunInOrderNoLoadBeforeEntered`, `coordinator.failingStepStopsLaterStepsNoRestore`, `projectSwitch.fullDrainIncludingTenantFacts` and `noCachedItemReadBeforeEntered` pass, and a unit test shows a session end during the guard leaves scope, drafts and address unchanged.

- [ ] **T-28 · `useGuardedSwitch(target, origin)`** — the published hook over the coordinator: returns `switchState` and the outcome; the single entry for every caller (plan 012 already calls it). *(plan §2.2, D1; RF-1.1)* — after T-27
  **Done when:** a hook test shows state transitions for a guarded and an unguarded switch, and plan 012's call site compiles against it with no interim fallback.

## D · Shell surfaces (`app/shell`)

- [ ] **T-29 · `TenantSwitchDirtyDialog` — N24** — implements `DirtyGuardPresenter` at bootstrap: "Save before switching?", the statement naming the item and "A pending request remains bound to its original tenant and project.", the authority line, "Save & switch", "Stay here", "Discard changes & switch"; modal with focus trap; Escape, close and backdrop return `STAY`; focus returns to the control that started the switch. *(plan §2.2, D3, D14; RF-3.1, RF-3.4, RF-3.5, RNF-8, LC-2, LC-14)* — after T-20, T-03
  **Done when:** `guard.shownWhenRegistryNonEmptyWithDrawnCopy` and `guard.stayEscapeCloseChangeNothingFocusReturns` pass.

- [ ] **T-30 · N24 states of S-59** — the several-items list (one line per registry entry by label) and the saving state (actions disabled, "saving" announced in a polite live region); the dialog closes on any cancelling outcome of T-22. *(plan §2.2, D4, D14; RF-3.2, RF-4.2, LC-7)* — after T-29, T-22
  **Done when:** `guard.listsEveryItemOneDialog` and `guard.savingStateDisablesActionsKeepsScope` pass.

- [ ] **T-31 · `SwitchingStatus` and `PendingRequestNotice`** — while `SWITCHING` the content area is masked and only "Switching to <target name>" is shown in a polite live region, without moving focus; the notice appears with it when the ledger recorded entries at step 2, text only, no action. *(plan §2.2, D5, D14; RF-2.3, RF-5.2, LC-16, LC-36)* — after T-27, T-03
  **Done when:** `switchingStatus.showsTargetNameOnlyAnnouncedPolitely` passes and a unit test shows the notice rendered exactly when the ledger flag is set.

- [ ] **T-32 · `SwitchCheckFailedPanel`** — while `CHECK_FAILED`: the safe problem with its correlation id, "Retry" (re-runs step 4 only), "Choose workspace" (→ `/workspaces`) and "Sign out" (`SignOutCoordinator` **[007]**, entry `SHELL`); nothing of either scope rendered. *(plan §2.2, D9, D15; RF-7.6, RF-12.3, LC-25)* — after T-27, T-03
  **Done when:** `entryCheck.failureShowsPanelNothingOfEitherScope` and `panel.retryRerunsStep4Only` pass, and `problems.safeCodeAndCorrelationIdOnly` passes for the panel (code and correlation id only).

- [ ] **T-33 · `TenantSelector` — WORKSPACE ⌄** — context-header trigger showing the current tenant; activation runs exactly the path of `SwitchWorkspaceAction` **[013]** (guard first, then `/workspaces`, current scope kept until an entry succeeds). *(plan §2.2, D1; RF-1.1, RF-1.2, LC-1, LC-3)* — after T-28, T-29
  **Done when:** `tenantSelector.guardThenChooserScopeKeptUntilEntered` passes.

- [ ] **T-34 · `ProjectSelector` — PROJECT ⌄ list** — opening fetches `GET /me/projects` (T-09) with no cache beyond the open list; keyboard-operable list (listbox semantics): own-grant lines, "canRun only · no project role yet", the current project marked, or the "No project grants yet …" line; choosing a line → `useGuardedSwitch({tenantId, projectId}, SELECTOR)`; choosing the current line → nothing. *(plan §2.2, D12, D14; RF-1.3, RF-1.4, LC-4, LC-5, LC-6)* — after T-28, T-09
  **Done when:** `projectSelector.rendersLinesCurrentMarkedNoGrantsLine` passes and a unit test shows choosing the current line issues no request.

- [ ] **T-35 · Selectors in the M04 menu** — both selectors reachable as items of the mobile menu with the same behaviour and the same list; 44 px targets; every action present at mobile and tablet widths. *(plan §2.2, D14; RF-1.5, RNF-10, LC-37)* — after T-33, T-34
  **Done when:** `selectors.inM04MenuSameBehaviour` passes.

- [ ] **T-36 · `NavigationInterceptPort` and the History-API adapter** — port in `shared/security`: `onPendingNavigation(handler)`, `hold()`, `release()`, `restore(previous)`, `setLeavePrompt(on)`; adapter in `app/shell` over the standard History API: wraps in-app link navigation, reacts to `popstate` by re-pushing the previous entry while held, toggles the leave-page event; no routing library added (A-67). *(plan §2.2, D11; RF-9.1, RF-9.2, RF-9.3)* — after T-28
  **Done when:** Vitest unit tests show a held navigation does not change the rendered route, `restore` returns the address bar to the previous entry after a `popstate`, and the leave-page prompt is requested only while enabled.

- [ ] **T-37 · `ScopeNavigationGuard`** — on each pending in-app navigation compare the route's tenant / project with the current scope: equal → release; different → hold and `useGuardedSwitch(routeScope, ROUTE)`; `STAY` or a cancelled guard → `restore(previous)`; `ENTERED` → release; leave prompt on while the registry is non-empty; a full page load starts a fresh epoch and runs steps 4 and 5 only. *(plan §2.2, D11; RF-9.1, RF-9.2, RF-9.3, RF-9.4, LC-27, LC-28, LC-30)* — after T-36
  **Done when:** `backButton.toOtherTenantRunsGuardedSwitch`, `navigationStay.restoresPreviousAddress`, `reload.withUnsavedWorkAsksLeavePrompt` and `reload.freshEpochStepsFourAndFive` pass.

## E · Entry-point rewiring (cross-plan items of plan §6)

- [ ] **T-38 · Chooser "Enter" [013] through the coordinator** — `useEnterTenant` becomes a caller of `useGuardedSwitch(target, CHOOSER)`; its existing epoch advance and cache clear are removed (now steps 3 and 5); `EntryRefusedHandler` **[013]** remains its refusal destination. *(plan §6 item 1, D1, D9; RF-1.1, RF-7.3)* — after T-28, T-25
  **Done when:** `refused.fromChooserRefreshesCard` passes and plan 013's chooser tests still pass.

- [ ] **T-39 · Invitation exit [008] and workspace creation [012] through the coordinator** — the acceptance exit calls `useGuardedSwitch({tenantId, projectId?}, INVITATION)`; workspace creation calls it with `CREATION` and its interim scope-epoch fallback is removed. *(plan §6 items 5 and 6, D1; RF-1.1, LC-35)* — after T-38
  **Done when:** `switch.everyEntryPointRunsCoordinator` passes for all seven origins (WORKSPACE ⌄, Switch workspace, chooser Enter, invitation exit, creation, PROJECT ⌄, route).

- [ ] **T-40 · Sign-out [007] on the same guard** — `SignOutCoordinator` step (1) awaits `UnsavedWorkRegistry` and `DirtyGuardPresenter` with sign-out in place of the switch; Save / Discard / Stay and the cancelling outcomes behave as in T-21 and T-22. *(plan §6 item 4, D2; RF-11.2, LC-34)* — after T-28, T-30
  **Done when:** `signOut.usesSameGuardAndRegistry` passes and plan 007's sign-out tests still pass.

- [ ] **T-41 · Existing unsaved surfaces join the registry** — the project grants panel's stamped draft **[011 D8]** and the Zustand editor forks are seen by the registry; any settings or policy form, typed dialog or chat / proposal composer that already exists calls `useUnsavedWork`; for screens not built yet, `useUnsavedWork` is the contract their tasks must use. *(plan §2.2 `UnsavedWorkRegistry`, D2; RF-3.2, RF-3.6, LC-7)* — after T-19, T-30
  **Done when:** with the grants panel open and dirty, N24 lists it by label, and each existing registering screen appears in the list when dirty and disappears when saved.

- [ ] **T-42 · Previews and downloads through `ScopedObjectUrls`** — every existing call that creates an object URL for a preview or download goes through the registry; a lint-style Vitest scan finds no direct object-URL creation outside it. *(plan §6 item 7, D7; RF-6.2, RF-8.2)* — after T-18
  **Done when:** the scan finds no direct object-URL creation in feature code, and after a switch no object URL created before it can be opened.

## F · Verification suites

- [ ] **T-43 · Guard-save end to end** — Playwright with an authoring stub: a dirty draft, "Save & switch", the save answered `412`; the switch is cancelled, N25 opens, the draft and the scope are intact. *(plan §5.2; RF-4.4, LC-8, US-4)* — after T-30, T-39
  **Done when:** `guardSave.conflictCancelsOpensN25DraftIntact` passes.

- [ ] **T-44 · Refusal outcomes end to end** — Playwright through the real API: a switch from a selector to a tenant whose membership was revoked (→ chooser re-derived), to a suspended tenant (→ N20), to a deleting tenant (→ N20 deleting wording), to a project without grant (→ N28 inside the tenant), an Owner without a factor (→ enrollment gate on arrival), and a deep link to a tenant never belonged to. *(plan §5.2, D9; RF-7.3, RF-7.4, RF-7.5, RF-7.7, RF-9.4, LC-20, LC-21, LC-22, LC-23, LC-26, LC-28, US-7)* — after T-37, T-39, T-10
  **Done when:** `refused.fromSelectorGoesToChooserRederived`, `refused.suspendedTenantShowsN20`, `refused.deletingTenantShowsN20DeletingWording`, `deniedProject.rendersN28InsideTenant`, `entry.ownerWithoutFactorMeetsGateOnArrival` and `deepLink.foreignTenantRefusedNeverShown` pass.

- [ ] **T-45 · Run boundary** — Playwright plus a BE integration check: a switch calls no run endpoint; an accepted Run keeps its tenant, project and environment pins after a switch; a launch pending at the switch is never sent again. *(plan §5.2, D5; RF-5.2, RF-5.4, RNF-4, LC-16, US-5)* — after T-39
  **Done when:** `switch.callsNoRunEndpoint`, `acceptedRun.keepsPinsAfterSwitch` and `launch.pendingAtSwitchNeverResent` pass.

- [ ] **T-46 · Isolation after a switch** — Playwright: after a switch from tenant A to B, no A name, identifier or content remains in the DOM, the query cache or memory-held stores, and browser storage holds only the FR-X-034 allowlist. *(plan §5.2, D6, D7; RF-6.2, RF-12.2, RNF-2, RNF-7, LC-38)* — after T-39
  **Done when:** `afterSwitch.noOldScopeInMemoryOrDom` and `storage.scanAfterSwitchAllowlistOnly` pass.

- [ ] **T-47 · Two tabs** — Playwright with two pages in one browser context: a switch in one tab leaves the other on its scope with its drafts and streams; a revocation is met by the other tab on its next request; session refresh and sign-out behave as plans 006 and 007 define. *(plan §5.2, D10; RF-10.1, RF-10.2, RF-10.3, LC-31, LC-32, US-8)* — after T-39
  **Done when:** `twoTabs.switchInOneLeavesOtherUntouched`, `twoTabs.revocationMetOnOtherTabsNextRequest` and `twoTabs.sessionBehaviourUnchanged` pass.

- [ ] **T-48 · Session end during the guard** — Playwright with the session stub of plan 006: the session ends while N24 is open; R34 is shown, the switch is abandoned, and a successful re-authentication returns to the original scope with its work. *(plan §5.2, D-d; RF-11.1, LC-33)* — after T-40
  **Done when:** `sessionEndsDuringGuard.abandonsSwitchReturnsOriginalScope` passes.

- [ ] **T-49 · Automated accessibility** — axe in Playwright on N24 and its states, the switching status, the PROJECT list and the check-failed panel, in EN, ES and FR, at desktop, tablet and mobile widths. *(plan §5.2, D14; RNF-8)* — after T-35, T-32
  **Done when:** `a11y.guard.{en,es,fr}.{desktop,tablet,mobile}`, `a11y.projectList.*` and `a11y.checkFailedPanel.*` report no WCAG 2.2 AA violation.

- [ ] **T-50 · Language and reflow** — every surface of this feature exists in EN / ES / FR; switching the interface language during a switch changes display only; every action is present at each width. *(plan §5.2, D14; RNF-9, RNF-10, LC-36, LC-37, US-9)* — after T-49
  **Done when:** `allSurfaces.existInEnEsFr`, `languageSwitch.duringSwitchDisplayOnly` and `reflow.everyActionPresentAtEachWidth` pass.

- [ ] **T-51 · Manual assistive-technology evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari on macOS and iOS), versions per the release-support matrix: the guard's focus trap and return, announcement of saving and switching, operation of the PROJECT list, the check-failed panel's actions. *(plan §5.1 manual assistive, D14; RNF-8, US-9)* — after T-50
  **Done when:** the signed checklist for each screen reader and browser pair is attached to the feature's evidence with every item passed.

- [ ] **T-52 · User-story journeys and the adversarial suite** — one Playwright journey per story US-1 … US-9 on the desktop design and in the reflows; the US-10 suite: reading the old scope after a switch, late events, a cross-tenant save, address editing, entry from a stale page, and a switch that tries to resend or cancel a command. *(plan §5.2 US rows; US-1, US-2, US-3, US-4, US-5, US-6, US-7, US-8, US-9, US-10, RF-4.1, RF-8.1, RF-9.4, LC-29, LC-38)* — after T-43 … T-50
  **Done when:** every journey passes end to end with no step outside the specification, and every US-10 case is refused or shows nothing of the other scope.

## G · Closing

- [ ] **T-53 · Coverage self-check** — a script compares the RF, RNF, LC and US identifiers of `specGuardedTenant.md` with the coverage index below, checks that every index row names existing tasks, and that every test name of plan §5.2 appears in at least one "Done when" line. *(plan §7; all requirements)* — after T-52
  **Done when:** the script reports no missing identifier, no unknown task id and no unused §5.2 test name.

- [ ] **T-54 · Register items and cross-plan hand-off** — confirm with the product owner that A-63, A-67, A-03, S-58, S-59, S-07 and S-02 are resolved or explicitly deferred; hand the seven cross-plan items of plan §6 to the owning plans' maintainers. *(plan §6; spec §7 criterion 13)* — after T-53
  **Done when:** each register item shows "resolved" or "deferred by the product owner" in its Figma register, and each cross-plan item has an acknowledged owner.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-28, T-33, T-38, T-39 |
| RF-1.2 | T-33 |
| RF-1.3 | T-02, T-03, T-07, T-09, T-34 |
| RF-1.4 | T-26, T-34 |
| RF-1.5 | T-35 |
| RF-2.1 | T-26, T-27 |
| RF-2.2 | T-05, T-27 |
| RF-2.3 | T-31 |
| RF-2.4 | T-25, T-27 |
| RF-3.1 | T-03, T-20, T-29 |
| RF-3.2 | T-19, T-30, T-41 |
| RF-3.3 | T-19, T-26 |
| RF-3.4 | T-20, T-26, T-29 |
| RF-3.5 | T-26, T-29 |
| RF-3.6 | T-19, T-41 |
| RF-3.7 | T-20 |
| RF-4.1 | T-21, T-52 |
| RF-4.2 | T-30 |
| RF-4.3 | T-21 |
| RF-4.4 | T-22, T-43 |
| RF-4.5 | T-22 |
| RF-4.6 | T-22 |
| RF-4.7 | T-21 |
| RF-4.8 | T-22 |
| RF-5.1 | T-14, T-16, T-26 |
| RF-5.2 | T-03, T-17, T-26, T-31, T-45 |
| RF-5.3 | T-17 |
| RF-5.4 | T-45 |
| RF-6.1 | T-13, T-27 |
| RF-6.2 | T-18, T-23, T-42, T-46 |
| RF-6.3 | T-23 |
| RF-7.1 | T-02, T-05, T-08, T-11, T-24 |
| RF-7.2 | T-05, T-08, T-24 |
| RF-7.3 | T-08, T-10, T-25, T-38, T-44 |
| RF-7.4 | T-08, T-10, T-25, T-38, T-44 |
| RF-7.5 | T-04, T-05, T-08, T-10, T-25, T-44 |
| RF-7.6 | T-25, T-32 |
| RF-7.7 | T-05, T-10, T-25, T-44 |
| RF-8.1 | T-15, T-16, T-52 |
| RF-8.2 | T-12, T-16, T-42 |
| RF-8.3 | T-05, T-24, T-27 |
| RF-9.1 | T-36, T-37 |
| RF-9.2 | T-36, T-37 |
| RF-9.3 | T-36, T-37 |
| RF-9.4 | T-37, T-44, T-52 |
| RF-10.1 | T-13, T-47 |
| RF-10.2 | T-11, T-47 |
| RF-10.3 | T-47 |
| RF-11.1 | T-27, T-48 |
| RF-11.2 | T-40 |
| RF-12.1 | T-06 |
| RF-12.2 | T-23, T-46 |
| RF-12.3 | T-02, T-08, T-32 |
| RNF-1 | T-27 |
| RNF-2 | T-15, T-23, T-46 |
| RNF-3 | T-17, T-21 |
| RNF-4 | T-45 |
| RNF-5 | T-05, T-24 |
| RNF-6 | T-01 |
| RNF-7 | T-23, T-46 |
| RNF-8 | T-29, T-49, T-51 |
| RNF-9 | T-03, T-50 |
| RNF-10 | T-35, T-50 |
| LC-1 | T-33 |
| LC-2 | T-29 |
| LC-3 | T-21, T-33 |
| LC-4 | T-07, T-34 |
| LC-5 | T-07, T-34 |
| LC-6 | T-26, T-34 |
| LC-7 | T-19, T-21, T-30, T-41 |
| LC-8 | T-22, T-43 |
| LC-9 | T-22 |
| LC-10 | T-22 |
| LC-11 | T-21 |
| LC-12 | T-22 |
| LC-13 | T-26 |
| LC-14 | T-29 |
| LC-15 | T-19 |
| LC-16 | T-17, T-26, T-31, T-45 |
| LC-17 | T-15, T-16 |
| LC-18 | T-15 |
| LC-19 | T-05, T-23, T-27 |
| LC-20 | T-06, T-10, T-11, T-25, T-44 |
| LC-21 | T-10, T-25, T-38, T-44 |
| LC-22 | T-10, T-25, T-38, T-44 |
| LC-23 | T-04, T-06, T-10, T-25, T-44 |
| LC-24 | T-04, T-10, T-25 |
| LC-25 | T-25, T-32 |
| LC-26 | T-05, T-10, T-25, T-44 |
| LC-27 | T-37 |
| LC-28 | T-37, T-44 |
| LC-29 | T-12, T-52 |
| LC-30 | T-37 |
| LC-31 | T-47 |
| LC-32 | T-11, T-47 |
| LC-33 | T-27, T-48 |
| LC-34 | T-40 |
| LC-35 | T-39 |
| LC-36 | T-31, T-50 |
| LC-37 | T-35, T-50 |
| LC-38 | T-23, T-46, T-52 |
| US-1 | T-52 |
| US-2 | T-52 |
| US-3 | T-52 |
| US-4 | T-43, T-52 |
| US-5 | T-45, T-52 |
| US-6 | T-52 |
| US-7 | T-44, T-52 |
| US-8 | T-47, T-52 |
| US-9 | T-50, T-51, T-52 |
| US-10 | T-52 |

Self-check: all 53 RF criteria, 10 RNF, 38 limit cases and 10 user stories of `specGuardedTenant.md` v0.3 appear above; every task names at least one requirement (T-53 covers the whole index); every one of the 74 test names of plan §5.2 appears in at least one "Done when" line; no task depends on a later one.
