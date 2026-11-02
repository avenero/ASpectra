# Tasks 014 — Tenant suspension state

**Feature** The suspended state of a tenant: what suspension freezes, what every member sees, what remains possible, and reinstatement · **Tasks version** 0.1 · **Date** 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specTenantSuspensionState.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planTenantSuspensionState.md` v0.1 (modules §2, decisions D1–D15, tests §5, register assumptions and cross-plan items §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[003]** / **[006]** / **[007]** / **[009]** / **[012]** / **[013]** extend an element that plan 001 (Login), 003 (Enrollment), 006 (Sessions), 007 (Logout), 009 (Roles and grants), 012 (Workspace creation) or 013 (Tenant chooser) builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Specifications 015 and 016 have no plan yet, and the execution, delivery, import and AI features have no specification yet; where a task reaches them it builds the contract of plan §2.1.3 against a recording stub in the shape the plan states. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API is the A-35 placeholder.

Phases: **A** foundations · **B** domain · **C** persistence and adapters · **D** application (`tenant`) · **E** `access` reads and fan-out · **F** consumer contracts · **G** front end · **H** verification and closing. Phase G depends only on T-02 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–F.

---

## A · Foundations

- [ ] **T-01 · `tenant` context placement and import rule** — the status lifecycle is placed in `com.aspectra.tenant` under `domain`, `application` and `infrastructure` (plan §2.1.1); an ArchUnit test (test-only) asserts that no controller, no `access`, `identity` or other tenant-side class calls `SuspendTenantCommand`, `ReinstateTenantCommand` or `ConfigureRecoveryContactCommand`, that `tenant` reaches other contexts only through published ports, and that this feature declares no `@ConfigurationProperties`. *(plan §2.1.1, D2; RF-1.1, RF-8.5, RNF-6, RNF-10)*
  **Done when:** the module compiles, `tenantSide.noCallerOfStatusCommands` passes, and `suspension.introducesNoConfigurationProperty` passes (the Spring context starts with no property added for this feature).

- [ ] **T-02 · Problem codes** — `access.status-unavailable` and `tenant.unavailable` added to `sharedsupport/problemcodes` as RFC 9457 problem types with a correlation id and no detail field; `access.denied-scope` **[006/009]** reused unchanged. *(plan D15; RF-10.3)* — after T-01
  **Done when:** a unit test renders both problems with only type, title key, status and correlation id, and a serialisation test shows no cause, reason or tenant field can be added.

- [ ] **T-03 · Audit visibility attribute [001]** — `AuditPort` **[001]** entries gain `visibility` (`TENANT | PLATFORM`, default `TENANT` for every existing caller); the tenant audit-trail read (F-APR-04, N18) selects `TENANT` only (plan §6 cross-plan item 5). *(plan §2.1.1 Audit adapter, D7; RF-10.1, RF-10.2)* — after T-01
  **Done when:** existing audit tests of plans 001–013 still pass unchanged, and a unit test shows an entry written with `PLATFORM` is absent from the tenant-trail query's result.

- [ ] **T-04 · `IdentityLocalePort` and the sign-in locale record [001]** — `identity` publishes `IdentityLocalePort.lastUsed(identityId) → EN | ES | FR | none`; `CompleteSignInUseCase` **[001]** records the request's UI locale header in its own transaction, in a row beside (not on) the identity row (plan §6 cross-plan item 4). *(plan D12; RF-10.4, RNF-13)* — after T-01
  **Done when:** an integration test signs in under ES and then FR and reads FR from the port, and an identity that never completed a sign-in reads `none`; plan 001's sign-in tests still pass.

## B · Domain (pure, no Spring)

- [ ] **T-05 · `TenantStatus` transition table** — enumeration `ACTIVE | SUSPENDED | DELETING` and the pure table: `ACTIVE → SUSPENDED`, `SUSPENDED → ACTIVE`, `SUSPENDED → DELETING`; every other pair refused; `DELETING` has no outgoing transition. *(plan §2.1.1 `TenantStatus`, D1, D2; RF-1.1, RF-6.1, RF-7.1, RF-7.4, LC-24)* — after T-01
  **Done when:** `tenantStatus.transitionTable` and `deleting.noOutgoingTransition` pass over all nine pairs.

- [ ] **T-06 · `Tenant` status methods and `TenantStatusTransition`** — `suspend(actor, reasonCategory, now)`, `reinstate(actor, now)`, `beginDeletion(actor, now)` apply the table, advance `statusVersion` once and return a `TenantStatusTransition` (tenant, from, to, `reasonCategory` required only for `→ SUSPENDED`, actor, time, correlation id); a request for the current state returns `ALREADY` and changes nothing; no other field of the tenant is touched. *(plan §2.1.1 `Tenant`, `TenantStatusTransition`, D1; RF-1.1, RF-1.8, RF-6.1, RF-7.1)* — after T-05
  **Done when:** `suspendedToDeleting.onlyViaBeginDeletion` passes, a unit test shows `suspend` without a reason category is refused, and `ALREADY` leaves `statusVersion` unchanged.

- [ ] **T-07 · `TenantRecoveryContact` and `RecoveryContactRule`** — value (tenant, `channelKind` `EMAIL_ADDRESS | PHONE | POSTAL | PLAIN_TEXT`, `channelValue`, plain-text `instructions`, version, configured-at) and the pure rule: value matches its kind; neither field contains a URL scheme, a host-like link, a token-like or code-like string, or an instruction to reset a password or factor (phrase list of the reset and recovery bundles in EN / ES / FR). *(plan §2.1.1, D8; RF-4.3)* — after T-01
  **Done when:** `recoveryContactRule.rejectsLinksTokensCodesResetInstructions` passes for each rejection class in each language, and a valid contact of each kind is accepted.

- [ ] **T-08 · `TenantFence`** — pure rule `(tenantStatus) → ACCEPT | FENCE`: anything but `ACTIVE`, including an unknown value, fences. Published for ingestion adapters. *(plan §2.1.1 `TenantFence`, D6; RF-1.6, RNF-8, RNF-9, LC-5, LC-6)* — after T-05
  **Done when:** `tenantFence.nonActiveFences` passes for `SUSPENDED`, `DELETING` and an unknown status, and `ACTIVE` accepts.

- [ ] **T-09 · `AccessStatusKind`** — pure rule `(membershipStatus, tenantStatus) → SUSPENDED_TENANT | DELETING_TENANT | NOT_PAUSED | UNAVAILABLE`: `REVOKED` or no membership → `UNAVAILABLE`; tenant `SUSPENDED` → `SUSPENDED_TENANT`; tenant `DELETING` → `DELETING_TENANT`; tenant `ACTIVE` → `NOT_PAUSED`, whatever the membership state. *(plan §2.1.2, D9; RF-2.9, RF-3.3, RF-3.10, LC-24, LC-25, LC-33)* — after T-05
  **Done when:** `accessStatusKind.deletingVariant` and `suspendedMembershipActiveTenant.neverN20` (BE unit part) pass over every status pair.

- [ ] **T-10 · Read models `TenantAccessStatus` and `RecoveryPanelView`** — `TenantAccessStatus` = `kind`, `tenantId`, `tenantName`, own `tenantRole`, own `membershipStatus`, `alternateEntry`; `RecoveryPanelView` = `contact` (kind, value, instructions) or `none`. No field exists for a reason, actor, time, project, source, run, member, invitation or other tenant. *(plan §2.1.2, D9; RF-3.1, RF-3.2, RF-3.4, RF-3.8, RF-4.1, RF-4.2, RF-4.4, RF-8.4, RNF-5)* — after T-09
  **Done when:** a serialisation test shows exactly the listed fields for each model, and a reflection test fails the build if a field is added.

## C · Persistence and adapters

- [ ] **T-11 · Tenant status columns [012]** — migration adds `tenant.status` (check constraint over the three values, not null) and `tenant.status_version`; plan 012's `TenantCommands.createTenant` writes `ACTIVE` and `0` (plan §6 cross-plan item 2). Migration tool per A-35. *(plan §2.1.1 Persistence, D1; RF-1.1, RF-1.8)* — after T-06
  **Done when:** the migration applies on real PostgreSQL, an insert with any other status is refused by the constraint, and plan 012's bootstrap test reads `ACTIVE` on the new tenant.

- [ ] **T-12 · Platform transition table** — `platform.tenant_status_transition` (id, tenant id, from, to, reason category, actor id, time, correlation id) in the platform schema, granted to the platform role only and not to the tenant runtime role; repository in `tenant.infrastructure`. *(plan §2.1.1 Persistence, D7; RF-1.1, RF-6.1, RF-10.2, RNF-5)* — after T-11
  **Done when:** `platformRecords.unreadableByTenantRole` passes: a select under the tenant runtime role is refused, the platform role reads the row.

- [ ] **T-13 · Recovery-contact table** — `tenant_recovery_contact` (primary key `tenant_id`, kind, value, instructions, version, configured-at) under `FORCE RLS` with the tenant predicate of `SET LOCAL`; repository with optimistic version. *(plan §2.1.1 Persistence, D8; RF-4.1, RF-4.3, LC-16)* — after T-11
  **Done when:** an integration test shows the row is read only with the matching tenant predicate set, and a second concurrent write with a stale version is refused.

- [ ] **T-14 · `TenantStatusPort` adapter** — implements plan 009's declaration: `statusOf(tenantId)` reads the column on every call with no cache; new `requireActiveForEffect(tenantId)` reads the row `FOR SHARE` inside the caller's transaction and answers `Denied(TENANT_NOT_ACTIVE)` when not `ACTIVE` (plan §6 cross-plan item 1). *(plan §2.1.1 Ports, D3; RF-1.2, RNF-1, RNF-9)* — after T-11
  **Done when:** `tenantStatusPort.neverCached` passes (a status changed between two calls in the same thread is seen by the second), and a unit test shows `requireActiveForEffect` outside a transaction throws.

- [ ] **T-15 · `RecoveryContactPort` adapter** — `contactOf(tenantId) → contact | none` over T-13, setting the tenant predicate for that one tenant inside its read transaction. *(plan §2.1.1 Ports; RF-4.1, RF-4.2)* — after T-13
  **Done when:** an integration test returns the configured contact, `none` for a tenant without one, and nothing for another tenant's id under the first tenant's predicate.

- [ ] **T-16 · `TenantAdministratorsQuery` [009]** — published query in `access`: active memberships with role `OWNER` or `ADMIN` of a tenant, read in the caller's transaction (plan §6 cross-plan item 1). *(plan §2.1.2; RF-1.7, RF-6.5, RF-10.4)* — after T-11
  **Done when:** an integration test over a fixture of active, suspended and revoked Owners, Admins and Members returns exactly the active Owners and Admins.

- [ ] **T-17 · Templates `tenant-suspended` and `tenant-reinstated`** — outbox templates **[002]** in EN / ES / FR with the content of RF-10.4 (S-14 assumption): suspended = tenant name, access paused, accepted runs are not failed tests and nothing is rewritten, the recovery contact when configured, no time promised; reinstated = access restored, nothing resumed automatically; no reason, no link that grants access, no project content. Language from `IdentityLocalePort` (T-04); `none` → the three languages in sequence. *(plan §2.1.1, D12; RF-1.7, RF-6.5, RF-7.2, RF-10.4, RNF-13)* — after T-04
  **Done when:** `messages.enEsFrRecipientLocaleTrilingualFallback` and `messages.noReasonNoAccessLink` pass, and `templatesAndBundles.promiseNoDurationOrDate` (template half) finds no duration or date placeholder.

- [ ] **T-18 · `tenant` audit events** — platform visibility: `TENANT_SUSPENDED{reasonCategory, actor}`, `TENANT_REINSTATED{actor}`, `TENANT_TRANSITION_REFUSED{from, requested}`, `RECOVERY_CONTACT_CONFIGURED | CHANGED | REMOVED{actor, previous, new}`; tenant visibility: `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_FAILED{template}`; each with tenant id, time and correlation id. *(plan §2.1.1 Audit adapter, D7; RF-10.1, RF-10.2)* — after T-03
  **Done when:** a unit test with a recording `AuditPort` shows each event's visibility and payload, and no tenant-visibility event carries a reason category, actor or contact value.

## D · Application (`tenant`)

- [ ] **T-19 · `SuspendTenantCommand`** — one transaction: lock the tenant row `FOR UPDATE`; `suspend`; insert the transition record (T-12); platform audit; enqueue `TenantStatusChanged{tenantId, from, to, statusVersion}`; enqueue one `tenant-suspended` message per recipient of `TenantAdministratorsQuery` with `NOTIFICATION_ENQUEUED`. `ALREADY` → `TENANT_TRANSITION_REFUSED` only, no event, no message. *(plan §2.1.1, D1, D2, D4, D12; RF-1.1, RF-1.3, RF-1.4, RF-1.5, RF-1.7, RF-10.1, LC-29)* — after T-12, T-14, T-16, T-17, T-18
  **Done when:** `suspend.recordsActorTimeReasonCategory` and `suspend.enqueuesMessagePerActiveOwnerAndAdmin` pass on real PostgreSQL, and a forced failure after the event insert leaves status, record, audit and outbox all unchanged.

- [ ] **T-20 · `ReinstateTenantCommand`** — same shape: `reinstate`; transition record without reason; platform audit; `TenantStatusChanged{→ ACTIVE}`; one `tenant-reinstated` message per active Owner and Admin; restores nothing. *(plan §2.1.1, D1, D5, D12; RF-6.1, RF-6.2, RF-6.5)* — after T-19
  **Done when:** `reinstate.recordsActorAndTime` and `reinstate.enqueuesRestoredMessageNothingResumed` pass.

- [ ] **T-21 · `ConfigureRecoveryContactCommand`** — `CONFIGURE | CHANGE | REMOVE` after `RecoveryContactRule`; writes T-13 with its version; platform audit with previous and new values; allowed in any status. *(plan §2.1.1, D8; RF-4.3, LC-16)* — after T-07, T-13, T-18
  **Done when:** `recoveryContact.changesAuditedPlatformOnly` passes, and a contact the rule rejects writes nothing and audits a refusal.

- [ ] **T-22 · Environment-only administrative entry** — a command-line entry runnable only from the deployment environment (Spring `ApplicationRunner` profile, no web binding) taking an actor id, a tenant id and the operation (`suspend` with reason category, `reinstate`, `contact` with its fields); placeholder until A-62 / A-01 decide the operator path, the shape of plan 004 D10. No HTTP mapping exists. *(plan §2.1.1 Entry, D2; RF-1.1, RF-4.3, RF-6.1, RF-8.5, RNF-6, LC-31)* — after T-20, T-21
  **Done when:** an integration test runs each operation through the entry and finds its audit row, and a MockMvc scan of every mapped route finds none reaching the three commands.

- [ ] **T-23 · Status-only invariant** — a row-snapshot harness over every tenant table (memberships, grants, permission versions, policy, revisions, approvals, runs, attempts, facts, artifacts, receipts, snapshots, configuration) before and after suspend and reinstate; no job, timer or scheduler is registered by `tenant`. *(plan D1, D2; RF-1.8, RF-2.7, RF-7.1, RF-7.3, RNF-4)* — after T-20
  **Done when:** `suspendAndReinstate.changeOnlyTenantStatus`, `suspension.startsNoPurgeOrTombstone` and `noSchedulerMovesStatus` pass.

- [ ] **T-24 · Message-failure path** — a relay delivery failure of either template is recorded `NOTIFICATION_FAILED{template}` and retried by the relay **[002]**; the transition is never reverted; an absent email channel is an operational failure, never a skipped message. *(plan D12; RF-10.4, RNF-11, LC-29)* — after T-19
  **Done when:** `messageFailure.recordedTransitionStands` and `emailAbsent.operationalFailureTransitionStands` pass.

## E · `access` reads and fan-out [009]

- [ ] **T-25 · Tenant-scope revocation ports and handler [009]** — `WorkRevocationPort.revokeTenant(tenantId, cause)` and `SubscriptionRevocationPort.closeTenant(tenantId)` added to plan 009's ports; `AuthorityChangedHandler` **[009]** consumes `TenantStatusChanged{→ SUSPENDED | DELETING}` with one call to each, and does nothing for `→ ACTIVE` (plan §6 cross-plan item 1). *(plan §2.1.2, D4, D5; RF-1.3, RF-1.4, RF-6.3, RF-8.6)* — after T-19
  **Done when:** `suspend.closesTenantSubscriptions` passes with a recording realtime port, the recording execution port receives exactly one `revokeTenant(tenant, TENANT_SUSPENDED)`, and a `→ ACTIVE` event calls neither.

- [ ] **T-26 · Commit-time check in effect transactions [009]** — `AuthorityChecker.require` **[009]** calls `TenantStatusPort.requireActiveForEffect` inside every effectful command's transaction; reads keep the plain per-derivation `statusOf` (plan 009 D9). *(plan D3; RF-1.2, RF-2.6, RNF-1, RNF-9, LC-26)* — after T-14
  **Done when:** `derive.everyActionClassDeniedAfterSuspension` passes for every action class of plan 009's matrix, and plan 009's derivation tests still pass.

- [ ] **T-27 · `TenantAccessStatusQuery`** — for the session identity (`SessionAssertionPort` **[006]**) and a requested tenant id: the identity's own membership row through plan 009 D10's identity-predicate query (A-36); tenant name and status through `TenantStatusPort`; `alternateEntry` = an `ACTIVE` membership in another `ACTIVE` tenant from the same listing; `AccessStatusKind`; `UNAVAILABLE` → `access.status-unavailable`; a paused kind audits `TENANT_STATUS_VIEWED`. Builds no C10. *(plan §2.1.2, D9; RF-2.3, RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.8, RF-3.10, RF-10.1, LC-7, LC-8, LC-33)* — after T-09, T-10, T-14
  **Done when:** `accessStatus.revokedOrNoneSameUnavailable` and `alternateEntry.onlyWithAnotherActiveTenant` pass on real PostgreSQL, and a unit test shows no `DeriveAuthorityUseCase` call.

- [ ] **T-28 · `RecoveryContactQuery`** — same identity check; only for a paused kind, else `access.status-unavailable`; reads `RecoveryContactPort`; audits `RECOVERY_PANEL_VIEWED`; stores nothing from the person. *(plan §2.1.2, D9; RF-4.1, RF-4.2, RF-4.4, RF-4.5, LC-16, LC-17)* — after T-15, T-27
  **Done when:** `recoveryContact.shownFromNextOpening` and `panel.viewAuditedStoresNothing` pass.

- [ ] **T-29 · `TenantAccessStatusController`** — `/api/me/tenants/{tenantId}`: `GET access-status` → 200 `TenantAccessStatus` · 404 `access.status-unavailable` (no membership, revoked and unknown tenant alike) · 401 without a session; `GET recovery-contact` → 200 `RecoveryPanelView` · 404; no write mapping. *(plan §2.1.2 `access.api`, D9, D15; RF-3.2, RF-3.7, RF-4.4, RF-8.4, RF-10.3, RNF-5, LC-30, LC-31)* — after T-02, T-28
  **Done when:** `accessStatus.exposesOnlyOwnMetadata`, `panel.identicalForEveryMember`, `problems.safeCodesOnlyNoCauseForNonMembers`, `noEndpoint.acceptsWriteFromN20` and `tenantEndpoints.deniedScopeCodeOnly` pass under MockMvc.

## F · Consumer contracts (recording stubs; plan §2.1.3)

- [ ] **T-30 · Execution contract** — a contract test suite for `WorkRevocationPort.revokeTenant` and the checkpoint check: queued, not-admitted tickets → NOT EXECUTED with cause `TENANT_SUSPENDED`; running attempts → one bounded cancellation request, facts kept; admission and each checkpoint call `TenantStatusPort` and stop when not `ACTIVE` even without the event; run-scoped authority untouched beyond the request (A-39). Built against a recording stub of `execution`; the execution plan re-runs it. *(plan §2.1.3, D4, D5; RF-1.4, RF-8.3, RF-8.6, RNF-2, LC-2, LC-3)* — after T-25
  **Done when:** `revokeTenant.queuedNotExecutedCauseTenantSuspended`, `revokeTenant.runningBoundedCancelFactsKept` and `suspension.neverMarksFailedOrRewritesVerdict` pass against the stub.

- [ ] **T-31 · Fencing contract (execution and integration ingestion)** — runner submissions, late results and provider callbacks pass `TenantFence`; `FENCE` → recorded `RECEIVED_FENCED` with `LATE_INPUT_FENCED` audit, applied to nothing, answered `tenant.unavailable`; the fence key is the input's own tenant id. *(plan §2.1.3, D6; RF-1.6, RF-2.8, RF-8.6, RF-10.3, RNF-8, LC-5, LC-6)* — after T-08, T-02
  **Done when:** `lateResultAndCallback.recordedFencedAppliedToNothing` and `fence.neverCrossesTenants` pass against the stubs.

- [ ] **T-32 · Delivery, import and AI contracts** — delivery jobs stop before the next provider write, unconfirmed and unknown items flagged `HELD_BY_SUSPENSION` in their reconciliation state and never retried by a job; import and AI jobs stop at their next checkpoint and reserve nothing; each records its audit event (`DELIVERY_STOPPED`, `DELIVERY_ITEM_HELD`, `IMPORT_STOPPED`, `AI_REQUEST_STOPPED`). *(plan §2.1.3, D4, D5; RF-1.5, RF-10.1, LC-4)* — after T-25
  **Done when:** `delivery.stopsBeforeProviderWriteItemsHeldNeverRetried` and `importAndAi.stopAtCheckpointReserveNothing` pass against the stubs.

- [ ] **T-33 · Nothing resumed on reinstatement** — a `TenantStatusChanged{→ ACTIVE}` is delivered to every recording consumer of T-25, T-30 and T-32; none starts, re-admits or retries anything; NOT EXECUTED tickets, cancelled attempts, held items and stopped jobs keep their state. *(plan D5; RF-6.3, RNF-3, LC-20, LC-21)* — after T-30, T-32
  **Done when:** `reinstate.noConsumerStartsWork` and `notExecutedHeldStoppedStaySo` pass.

## G · Front end (`src/features/identity-access`, `shared/*`)

- [ ] **T-34 · `contracts/accessStatus` and the route** — types and runtime guards for `TenantAccessStatus`, `RecoveryPanelView`, `access.status-unavailable` and `tenant.unavailable`, rejecting any extra field; route `/workspaces/:tenantId/access-status` under `ProtectedBoundary` **[006]**, outside the tenant shell. *(plan §2.2, D9; RF-3.2, RF-4.4, RF-8.4, RNF-5)* — after T-02
  **Done when:** `contractGuard.rejectsExtraFields` passes, and a route test shows the page mounts with a tenant id and no session redirects to `/sign-in`.

- [ ] **T-35 · `useTenantAccessStatus` and `useRecoveryContact`** — TanStack queries: status with `staleTime` 0, refetch on mount, no interval, focus or reconnect refetch, background marker never set, one bounded retry on transport failure; contact enabled only when the panel opens, refetched on each opening; nothing persisted. *(plan §2.2, D11; RF-3.10, RF-4.1, RF-4.5, RF-10.5, LC-16)* — after T-34
  **Done when:** `storage.containsNothingFromN20` passes, and hook tests show no refetch on focus or reconnect and one contact request per opening.

- [ ] **T-36 · N20 statements and bundles** — `SecurityStatements` **[001]** extended with every N20 and panel statement, label and state of RF-3.1, RF-3.4, RF-4.1 and RF-4.2 (including the deleting wording, S-55 / S-57) in EN / ES / FR. *(plan §2.2, D14; RF-3.1, RF-4.1, RF-4.2, RF-7.2, RNF-13)* — after T-34
  **Done when:** `templatesAndBundles.promiseNoDurationOrDate` (bundle half) passes and a key-parity test shows every key in all three languages.

- [ ] **T-37 · `TenantSuspendedPage` (N20)** — product header with display name and language switch, marker, heading, tenant statement, "Your current membership" card, metadata statement, "Review verified workspace recovery", the accepted-runs notice, "Safe next steps", the alternate-tenant entry or its absence statement, "Personal account security", "Sign out"; renders the read model only; focus enters the heading on open; no retry, request, form or tenant link. *(plan §2.2, D9, D14; RF-3.1, RF-3.2, RF-3.4, RF-3.7, RF-3.8, RF-8.4, LC-30)* — after T-35, T-36
  **Done when:** `n20.rendersDesignedContentVerbatim`, `n20.sameForOwnerAdminMemberExceptRoleAndAlternate`, `n20.suspendedMembershipShowsSuspended`, `n20.alternateEntryOrAbsenceStatement`, `n20.noRetryRequestFormOrTenantLink` and `nowhere.namesReasonActorOrTime` pass.

- [ ] **T-38 · Deleting variant** — `kind = DELETING_TENANT` renders the deleting wording with the same metadata and routes. *(plan §2.2, D9; RF-2.9, LC-24)* — after T-37
  **Done when:** `n20.deletingWording` passes.

- [ ] **T-39 · `RecoveryContactPanel`** — D02 panel on N20 (S-57 assumption): verified-process statement; contact as plain text, never autolinked, with "use only this contact"; or the no-contact statement; "Close" only; focus to the panel heading, back to the trigger on close. *(plan §2.2, D8; RF-4.1, RF-4.2, RF-4.4, RF-4.5, RF-9.2, LC-16, LC-17)* — after T-37
  **Done when:** `panel.contactPlainTextCloseOnly` and `panel.noContactStatement` pass, and a test shows a contact containing text that looks like a link renders without an anchor.

- [ ] **T-40 · Language switch** — EN / ES / FR switch on N20 and its panel re-renders from bundles; query data and panel state are kept; no request is sent. *(plan §2.2; RF-3.9, LC-28)* — after T-39
  **Done when:** `languageSwitch.displayOnly` passes.

- [ ] **T-41 · `TenantStateResolver` [006]** — in `shared/errors`, beside `DeniedScopeHandler` **[006]**: when the handler's one permission refresh is itself refused at tenant level, abort in-flight requests, disconnect STOMP, advance the scope epoch, clear the scoped cache, then read `access-status`: paused kind → N20 route; `NOT_PAUSED` or `access.status-unavailable` → `/workspaces`; a project-level refusal keeps plan 006's N28 path (plan §6 cross-plan item 3). *(plan §2.2, D10; RF-1.3, RF-2.1, RF-2.2, RF-5.4, RF-5.6, LC-1, LC-9, LC-10, LC-27)* — after T-35
  **Done when:** `resolver.tenantLevelRefusalReadsStatus` passes for the three outcomes, and plan 006's `DeniedScopeHandler` tests still pass.

- [ ] **T-42 · Handoffs** — alternate entry → `/workspaces` **[013]**; "Personal account security" → `SecuritySettingsPage` **[003]**; "Sign out" → `SignOutCoordinator.signOut({entry: 'N20'})` **[007]**; on mount `NOT_PAUSED` → `useEnterTenant(tenantId)` **[013]** (entered → overview, refused → `/workspaces`); `access.status-unavailable` → `/workspaces`. *(plan §2.2, D11, D13; RF-3.4, RF-3.5, RF-3.6, RF-3.10, RF-6.4, LC-19)* — after T-37
  **Done when:** `n20.personalAccountSecurityOpensSecuritySettings`, `n20.signOutEntryN20SuspensionUntouched` and `reinstatedWhileOpen.nextRequestEntersTenant` pass with 003, 007 and 013 stubs.

## H · Verification suites and closing

- [ ] **T-43 · Freeze journeys** — Playwright journeys with a suspended fixture tenant: a typed tenant URL, an open polling tab, an open editor, the chooser's "View access status", and a sign-in whose only membership is the suspended tenant. *(plan §5.2; RF-1.3, RF-2.1, RF-2.3, RF-5.6, LC-1, LC-7, LC-8, LC-9, LC-25, LC-27)* — after T-29, T-41, T-42
  **Done when:** `tenantRoute.typedUrlEndsOnN20NoContent`, `pollingTab.refusedThenN20NothingServed`, `openEditor.deniedScopeThenN20`, `chooser.viewAccessStatusOpensN20WithoutEntry`, `onlySuspendedTenant.landsOnChooser` and `suspendedMembershipActiveTenant.neverN20` (e2e part) pass, and network capture shows no tenant content served.

- [ ] **T-44 · Guarded switch** — with a 016 stub of the switch's step (4) handing a suspended target to `TenantStateResolver` (plan §6 cross-plan item 7): into the suspended tenant ends on N20 with no stream opened; out of it into an active tenant proceeds. *(plan §5.2; RF-2.4)* — after T-43
  **Done when:** `guardedSwitch.intoSuspendedEndsOnN20NoStream` and `guardedSwitch.outOfSuspendedToActiveProceeds` pass.

- [ ] **T-45 · Denied surfaces, invitations and isolation** — integration and API suites with a suspended tenant and a second active tenant of the same members: export, delivery retry, reconciliation, archive, restore, purge request and deletion of content; plan 010's members commands; plan 008's acceptance and plan 010's issue, resend, revoke and expiry. *(plan §5.2; RF-2.5, RF-2.6, RF-2.8, RF-5.4, RF-8.6, RNF-8, LC-10, LC-15)* — after T-26
  **Done when:** `exportRetryReconcileArchiveRestorePurge.deniedWhileSuspended`, `members.transferRoleInviteDenied`, `invitation.suspendedTenantUnusableStaysPending`, `invitation.issueResendRevokeDenied`, `invitation.expiryUnchanged` and `suspension.otherTenantsUntouched` pass.

- [ ] **T-46 · Identity untouched** — suites showing security settings, password reset, controlled recovery, sessions, sign-out, the chooser, other tenants, workspace creation and other invitations work while suspended; no session, credential, factor or address changes; no reset, recovery or re-authentication lifts the status; no tenant-side route transitions it. *(plan §5.2, D13; RF-5.1, RF-5.2, RF-5.6, RF-8.1, RF-8.5, RNF-7, LC-13, LC-14, LC-32)* — after T-43
  **Done when:** `identityActions.workWhileSuspended`, `resetAndRecovery.leaveSuspensionLandOnChooser`, `suspension.touchesNoSessionCredentialFactorAddress`, `resetRecoveryReauth.neverLiftSuspension` and `noTenantSideTransition` pass.

- [ ] **T-47 · Leave and erasure contracts for plan 015** — contract suite against a 015 stub (plan §6 cross-plan item 6; specification 015 RF-4.1, RF-2.4, RF-5.5, D-a): an identity-scoped leave permits a Member or Admin of a suspended tenant and revokes only their membership and grants through `MembershipCommands.RevokeMembership` **[009]** without a tenant derivation; every Owner is refused with `TENANT_SUSPENDED`; the erasure request is held while a last-Owner transfer is needed in a suspended tenant; a person who left is not a member after reinstatement. *(plan D13; RF-5.3, RF-5.4, RF-5.5, LC-11, LC-12, LC-23)* — after T-26
  **Done when:** `leave.memberOrAdminRevokesOwnMembershipOnly`, `leftThenReinstated.notAMember`, `leave.ownerRefusedTenantSuspended` and `erasure.heldWhileOwnedTenantSuspended` pass against the stub.

- [ ] **T-48 · Reinstatement suite** — after suspend then reinstate: memberships, grants, permission versions and policy revision as before; a membership suspended or revoked before the suspension stays so; the chooser lists the tenant active, pending unexpired invitations are acceptable again, and new subscriptions are authorised. *(plan §5.2, D1, D11; RF-6.2, RF-6.4, RF-6.6, RF-8.2, LC-22)* — after T-20, T-25
  **Done when:** `reinstate.membershipsGrantsVersionsAsBefore`, `reinstate.restoresNoEarlierSuspendedOrRevoked` and `reinstate.chooserActiveInvitationsAcceptableSubscriptionsAllowed` pass.

- [ ] **T-49 · Concurrency on real PostgreSQL** — a write racing `SuspendTenantCommand` (share lock vs exclusive lock), an admission racing it, and a late callback after the commit. *(plan §5.2, D3, D6; RNF-15, LC-26)* — after T-26, T-31
  **Done when:** `write.racingSuspensionDecidedAtCommit`, `admission.racingSuspensionRefused` and `lateCallback.afterCommitFenced` pass on 200 repetitions with no flaky outcome.

- [ ] **T-50 · Audit suite** — every event of RF-10.1 across `tenant`, `access` and the consumer stubs, each with tenant, member where involved, time, calling context and correlation id; the tenant trail never returns a platform-visibility entry. *(plan §5.2, D7; RF-10.1, RF-10.2, LC-30)* — after T-24, T-29, T-33
  **Done when:** `audit.everyEventOfRf10_1WithCorrelationId` and `tenantTrail.neverSelectsPlatformVisibility` pass.

- [ ] **T-51 · Accessibility and reflow** — Playwright + axe on N20 (suspended and deleting variants, alternate entry present and absent) and the panel (contact and no contact) at desktop, tablet and mobile widths in EN / ES / FR; manual NVDA and VoiceOver checklist: focus to heading on screen and panel, reading order, state as text, 44 px targets, no focus stealing. *(plan §5.2, D14; RF-9.1, RF-9.2, RF-9.3, RNF-12, RNF-13, RNF-14, LC-18)* — after T-40, T-42
  **Done when:** `a11y.n20AndPanel.{en,es,fr}.{desktop,tablet,mobile}` passes with zero axe violations and the signed manual checklist is attached to the release evidence.

- [ ] **T-52 · Journeys and adversarial suite** — US-1 … US-9 walked end to end on desktop and in the reflows; US-10 adversarial cases: reason discovery through screen, chooser, problems, messages and audit; content through the paused state; launch, delivery, import or query in a frozen tenant; a cancelled attempt shown as failed; resume by reinstatement; tenant-side transitions; reset or recovery lifting the state. *(plan §5.2; US-1 … US-9, US-10, RF-8.4, RNF-5)* — after T-43 … T-51
  **Done when:** every journey passes and every adversarial case is refused with a safe problem or shows nothing, recorded in the release evidence.

- [ ] **T-53 · Closing** — the register items of plan §6 (A-62, A-56, A-54, A-39, A-36, A-35, A-30, A-01; S-57, S-55, S-48, S-14, S-02, S-17) are resolved or explicitly deferred by the product owner; the eight cross-plan items of plan §6 are recorded in plans 009, 012, 006, 001, the audit context, 015, 016 and the execution, delivery, import and AI plans; specification §7 items 1–13 checked off. *(plan §5.4, §6; spec §7)* — after T-52
  **Done when:** each register item shows "resolved" or "deferred by the product owner" with a date, each cross-plan item is present in its owning document, and the §7 checklist is complete.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-01, T-05, T-06, T-11, T-12, T-19, T-22 |
| RF-1.2 | T-14, T-26 |
| RF-1.3 | T-19, T-25, T-41, T-43 |
| RF-1.4 | T-19, T-25, T-30 |
| RF-1.5 | T-19, T-32 |
| RF-1.6 | T-08, T-31 |
| RF-1.7 | T-16, T-17, T-19 |
| RF-1.8 | T-06, T-11, T-23 |
| RF-2.1 | T-41, T-43 |
| RF-2.2 | T-41 |
| RF-2.3 | T-27, T-43 |
| RF-2.4 | T-44 |
| RF-2.5 | T-45 |
| RF-2.6 | T-26, T-45 |
| RF-2.7 | T-23 |
| RF-2.8 | T-31, T-45 |
| RF-2.9 | T-09, T-38 |
| RF-3.1 | T-10, T-27, T-36, T-37 |
| RF-3.2 | T-10, T-27, T-29, T-34, T-37 |
| RF-3.3 | T-09, T-27 |
| RF-3.4 | T-10, T-27, T-37, T-42 |
| RF-3.5 | T-42 |
| RF-3.6 | T-42 |
| RF-3.7 | T-29, T-37 |
| RF-3.8 | T-10, T-27, T-37 |
| RF-3.9 | T-40 |
| RF-3.10 | T-09, T-27, T-35, T-42 |
| RF-4.1 | T-10, T-13, T-15, T-28, T-35, T-36, T-39 |
| RF-4.2 | T-10, T-15, T-28, T-36, T-39 |
| RF-4.3 | T-07, T-13, T-21, T-22 |
| RF-4.4 | T-10, T-28, T-29, T-34, T-39 |
| RF-4.5 | T-28, T-35, T-39 |
| RF-5.1 | T-46 |
| RF-5.2 | T-46 |
| RF-5.3 | T-47 |
| RF-5.4 | T-41, T-45, T-47 |
| RF-5.5 | T-47 |
| RF-5.6 | T-41, T-43, T-46 |
| RF-6.1 | T-05, T-06, T-12, T-20, T-22 |
| RF-6.2 | T-20, T-48 |
| RF-6.3 | T-25, T-33 |
| RF-6.4 | T-42, T-48 |
| RF-6.5 | T-16, T-17, T-20 |
| RF-6.6 | T-48 |
| RF-7.1 | T-05, T-06, T-23 |
| RF-7.2 | T-17, T-36 |
| RF-7.3 | T-23 |
| RF-7.4 | T-05 |
| RF-8.1 | T-46 |
| RF-8.2 | T-48 |
| RF-8.3 | T-30 |
| RF-8.4 | T-10, T-29, T-34, T-37, T-52 |
| RF-8.5 | T-01, T-22, T-46 |
| RF-8.6 | T-25, T-30, T-31, T-45 |
| RF-9.1 | T-51 |
| RF-9.2 | T-39, T-51 |
| RF-9.3 | T-51 |
| RF-10.1 | T-03, T-18, T-19, T-27, T-32, T-50 |
| RF-10.2 | T-03, T-12, T-18, T-50 |
| RF-10.3 | T-02, T-29, T-31 |
| RF-10.4 | T-04, T-16, T-17, T-24 |
| RF-10.5 | T-35 |
| RNF-1 | T-14, T-26 |
| RNF-2 | T-30 |
| RNF-3 | T-33 |
| RNF-4 | T-23 |
| RNF-5 | T-10, T-12, T-29, T-34, T-52 |
| RNF-6 | T-01, T-22 |
| RNF-7 | T-46 |
| RNF-8 | T-08, T-31, T-45 |
| RNF-9 | T-08, T-14, T-26 |
| RNF-10 | T-01 |
| RNF-11 | T-24 |
| RNF-12 | T-51 |
| RNF-13 | T-04, T-17, T-36, T-51 |
| RNF-14 | T-51 |
| RNF-15 | T-49 |
| LC-1 | T-41, T-43 |
| LC-2 | T-30 |
| LC-3 | T-30 |
| LC-4 | T-32 |
| LC-5 | T-08, T-31 |
| LC-6 | T-08, T-31 |
| LC-7 | T-27, T-43 |
| LC-8 | T-27, T-43 |
| LC-9 | T-41, T-43 |
| LC-10 | T-41, T-45 |
| LC-11 | T-47 |
| LC-12 | T-47 |
| LC-13 | T-46 |
| LC-14 | T-46 |
| LC-15 | T-45 |
| LC-16 | T-13, T-21, T-28, T-35, T-39 |
| LC-17 | T-28, T-39 |
| LC-18 | T-51 |
| LC-19 | T-42 |
| LC-20 | T-33 |
| LC-21 | T-33 |
| LC-22 | T-48 |
| LC-23 | T-47 |
| LC-24 | T-05, T-09, T-38 |
| LC-25 | T-09, T-43 |
| LC-26 | T-26, T-49 |
| LC-27 | T-41, T-43 |
| LC-28 | T-40 |
| LC-29 | T-19, T-24 |
| LC-30 | T-29, T-37, T-50 |
| LC-31 | T-22, T-29 |
| LC-32 | T-46 |
| LC-33 | T-09, T-27 |
| US-1 | T-52 |
| US-2 | T-52 |
| US-3 | T-52 |
| US-4 | T-52 |
| US-5 | T-52 |
| US-6 | T-52 |
| US-7 | T-52 |
| US-8 | T-52 |
| US-9 | T-52 |
| US-10 | T-52 |

Self-check: all 62 RF criteria, 15 RNF, 33 limit cases and US-1 … US-10 of `specTenantSuspensionState.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line; every "after" points to an earlier task.
