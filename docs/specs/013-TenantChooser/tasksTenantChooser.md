# Tasks 013 — Tenant chooser

**Feature** The page where a signed-in identity sees the tenants it belongs to, with its role, its membership state and its own project grants in each, and chooses which one to enter · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specTenantChooser.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planTenantChooser.md` v0.1 (modules §2, decisions D1–D13, tests §5, register assumptions and cross-plan items §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** / **[003]** / **[006]** / **[007]** / **[008]** / **[009]** extend an element that plan 001 (Login), 003 (Enrollment), 006 (Sessions), 007 (Logout), 008 (Invitation acceptance) or 009 (Roles and grants) builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. Specifications 012, 014 and 016 have no plan yet; where a task reaches them it stubs their route in the shape the specification states. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API is the A-35 placeholder.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-01 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Read-side placement and import rule** — the chooser's elements are placed in the `access` context of plan 009 under `application` (two use cases) and `domain` (three pure rules and the read model); an ArchUnit test (test-only) asserts that this feature declares no repository, no write port, no table and no `@ConfigurationProperties`, and reaches `identity`, `tenant`, `project` and `audit` only through their published ports. *(plan §2.1 header, D10; RNF-9, RNF-10)*
  **Done when:** the module compiles, the ArchUnit test passes, and `chooser.introducesNoConfigurationProperty` passes (the Spring context starts with no property added for this feature).

- [ ] **T-02 · `ProjectNamesPort` published by `project`** — interface `namesOf(tenantId, projectIds) → Map<projectId, name>` in the `project` context's published ports, names only, under the tenant predicate; a recording double for the application tests of phase C (plan §6 cross-plan item 2). *(plan §2.1 Ports, D2; RF-4.2)* — after T-01
  **Done when:** the interface compiles in the `project` published package, the double returns names for requested ids only, and ArchUnit shows `access` importing nothing from `project` but this port.

- [ ] **T-03 · `MembershipProjectionPort` read model extension [009]** — the projection row of plan 009 gains `projectLines` (project id, project name, `GrantSet` **[009]**), populated only for rows whose membership is `ACTIVE` and whose tenant is `ACTIVE`, empty otherwise; the interface change is recorded as plan 009's cross-plan item and the existing consumers (`LandingQuery` **[001]**) are unaffected. *(plan §2.1 Ports, D2; RF-3.1, RF-4.2, RF-5.5)* — after T-02
  **Done when:** plan 001's and plan 009's tests that consume the port still pass with the extended row, and a unit test shows a non-enterable row carries an empty `projectLines`.

## B · Domain (pure, no Spring)

- [ ] **T-04 · `ChooserCardKind`** — pure rule `(membershipStatus, tenantStatus) → ENTERABLE | SUSPENDED_MEMBERSHIP | SUSPENDED_TENANT | DELETING_TENANT | NOT_LISTED`: tenant state first (`SUSPENDED` → suspended tenant, `DELETING` → deleting tenant), then membership (`SUSPENDED` → suspended membership, `REVOKED` → not listed), else enterable; each row classified independently. *(plan §2.1 `ChooserCardKind`, D1; RF-3.2, RF-3.4, RF-5.1, RF-5.2, RF-5.3, RF-5.6, LC-4, LC-5, LC-6, LC-7)* — after T-01
  **Done when:** `chooserCardKind.everyStatusPairTenantStateFirst` and `chooserCardKind.rowsClassifiedIndependently` pass over all nine status pairs.

- [ ] **T-05 · `ChooserOrder`** — pure comparator: kind rank (enterable, then suspended membership, then suspended and deleting tenants together), then tenant name under the collation of the request locale (EN, ES, FR), then tenant id as a stable tie-break; a second comparator orders project lines by project name under the same collation. *(plan §2.1 `ChooserOrder`, D9; RF-3.5, RF-4.2, LC-19, LC-20)* — after T-04
  **Done when:** `chooserOrder.kindRankThenLocaleCollationThenId` passes for the three locales, for accented names, and for two tenants with the same name.

- [ ] **T-06 · `MfaLineRule`** — pure rule `(tenantRole, policyRequiresMfaForMembers, hasActiveFactor) → NONE | OWNER | ADMIN | MEMBER_POLICY` plus `notYetEnrolled`; the inputs are those of `RequirementReasonPort` **[003]**, so the line and the enrollment gate cannot disagree. *(plan §2.1 `MfaLineRule`, D6; RF-4.4, RF-4.5, LC-12, LC-13, LC-14)* — after T-01
  **Done when:** `mfaLineRule.roleAndPolicyMatrix` passes over role × policy × factor, with `notYetEnrolled` true only when a line applies and no active factor exists.

- [ ] **T-07 · `ChooserCard` read model and project-line flags** — `kind`, `tenantId`, `tenantName`, `tenantRole`, `membershipStatus`, `tenantStatus`, `mfaLine`, `projectLines` (each `projectId`, `projectName`, `grants`, `canRunOnly` = `GrantSet` **[009]** with `canRun` and no role) and `noGrants`; non-enterable kinds carry an empty `projectLines`; no field exists for another identity, a member count, a source or a run. *(plan §2.1 `ChooserCard`, D1, D2; RF-2.6, RF-4.1, RF-4.3, RF-4.6, RF-5.5, RF-10.4, RNF-1, LC-15, LC-16)* — after T-04, T-06
  **Done when:** `projectLine.canRunOnlyFlag` passes, a serialisation test shows exactly the listed fields, and a non-enterable card cannot be constructed with project lines.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-13…T-15); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-08 · `TenantChooserQuery`** — for the session identity (`SessionAssertionPort` **[006]**): read the extended projection (T-03); for each enterable row read `TenantPolicyPort.requireMfaForMembers(tenantId)` **[009]**; read `FactorStatusPort.hasActiveFactor(identityId)` **[003]** once; read address and display name through `IdentityFactsPort` **[008]**; apply `ChooserCardKind`, `MfaLineRule` and `ChooserOrder`; return the ordered cards with address and display name. Writes nothing; audits nothing; builds no tenant context. *(plan §2.1 `TenantChooserQuery`, D1, D2, D6, D9; RF-1.6, RF-2.1, RF-3.1 … RF-3.7, RF-4.2 … RF-4.5, RF-5.5, RF-7.1, RF-10.3, LC-17)* — after T-05, T-07
  **Done when:** `listing.buildsNoTenantContext`, `listing.memberPolicyLineFromTenantPolicyPort` and `listing.notYetEnrolledSuffixFromFactorStatus` (a pending enrollment is not a factor) pass with in-memory doubles, and the returned order equals `ChooserOrder` for a mixed fixture.

- [ ] **T-09 · `TenantEntryUseCase`** — for a requested `tenantId`: run `DeriveAuthorityUseCase` **[009]** at tenant level for the session identity; `Derived` → `{tenantId, tenantName, tenantRole}` and nothing else; `Denied(NO_MEMBERSHIP | MEMBERSHIP_NOT_ACTIVE | TENANT_NOT_ACTIVE)` → audit `TENANT_ENTRY_REFUSED{cause}` and the single problem `access.entry-refused`; `Denied(ENROLLMENT_GATE)` → the derivation's own `auth.enrollment.required {reason}`; creates no session, changes no row. *(plan §2.1 `TenantEntryUseCase`, D3, D4, D5; RF-6.1, RF-6.3, RF-6.4, RF-6.5, RF-6.6, RF-10.1, LC-8, LC-9, LC-10, LC-12, LC-13)* — after T-01
  **Done when:** `entry.runsTenantLevelDerivationWithRequestedScopeOnly`, `entry.refusedAfterSuspensionRevocationOrTenantChangeAuditedOneCode`, `entry.gateRaisesEnrollmentRequired`, `entry.writesNothingReplaysNothing` and `entry.changesNoRow` pass with in-memory doubles.

- [ ] **T-10 · Audit event catalogue** — the event `TENANT_ENTRY_REFUSED{tenantId, cause}` with identity id, correlation id, calling context and timestamp; a redaction test that no payload can carry a card, a grant, a project name or another identity. *(plan §2.1 Audit adapter, D4; RF-10.1, RF-10.2)* — after T-09
  **Done when:** `audit.entryRefusedPerCauseNoListContent` passes with a recording `AuditPort` for each of the three refusal causes.

- [ ] **T-11 · Nothing-on-the-wire contract** — a unit contract over the two use cases' outputs: the listing carries only `ChooserCard` fields, address and display name; the entry result carries only `tenantId`, `tenantName`, `tenantRole`; neither carries a grant set for routing, a policy, a factor detail or another identity. *(D1, D3; RF-2.6, RF-6.5, RNF-1, RNF-2)* — after T-08, T-09
  **Done when:** `chooserGrantsNoAuthority` passes: the listing alone enters nothing and the entry result alone derives nothing.

## D · Infrastructure adapters

- [ ] **T-12 · Projection query extension** — the control-plane query of plan 009 D10 (identity predicate set, `tenant_membership` rows of the identity) extended with a join to `project_grant` restricted to rows whose membership is `ACTIVE` and whose tenant is `ACTIVE`, and a names lookup through `ProjectNamesPort`; a revoked membership never leaves the query; no new table or index. *(plan §2.1 Projection query, D2; RF-3.1, RF-3.2, RF-3.3, RF-4.2, RF-5.5, RF-10.3, RNF-1, RNF-5, RNF-6, LC-4, LC-34)* — after T-03
  **Done when:** `projection.identityPredicateReturnsOwnRowsOnly`, `projection.revokedNeverListedEvenWhenOnly`, `projection.noRowForTenantWithoutMembership`, `projection.projectLinesEqualLiveGrantsAlphabetical` and `projection.noProjectNameForNonEnterableRows` pass against real PostgreSQL.

- [ ] **T-13 · `ProjectNamesPort` adapter** — in `project.infrastructure`, a names-only read of the project table under the tenant predicate for the requested ids; unknown ids are omitted. *(plan §2.1 Ports, D2; RF-4.2)* — after T-02
  **Done when:** an integration test shows names for the identity's granted projects only and no row of another tenant.

- [ ] **T-14 · Audit adapter** — the T-10 event through `audit.AuditPort`, appended in the entry request's transaction with the correlation id from `sharedsupport.contextIds`. *(plan §2.1 Audit adapter, D4, D13; RF-10.1, RF-10.2)* — after T-10
  **Done when:** `audit.entryRefusedPerCauseNoListContent` passes against real PostgreSQL and `logs.noListOrGrant` shows no card content in the log output.

- [ ] **T-15 · Listing and entry integration** — `TenantChooserQuery` and `TenantEntryUseCase` wired to the real adapters (projection, `TenantPolicyPort` **[009]**, `FactorStatusPort` **[003]**, `IdentityFactsPort` **[008]**, `DeriveAuthorityUseCase` **[009]**). *(D2, D3; RF-3.1, RF-6.1)* — after T-12, T-13, T-14
  **Done when:** the T-08 and T-09 tests pass against real PostgreSQL with the owning contexts' test doubles, and `failClosed.everyNonEnterableStateRefusesEntry` passes.

## E · API

- [ ] **T-16 · `GET /api/me/memberships`** — session required (401 otherwise); 200 `{address, displayName, cards}` in `ChooserOrder` under the UI locale header; no path tenant, no query parameter; RFC 9457 problems with a correlation id. *(plan §2.1 `MembershipsController`, D1, D9, D13; RF-1.1, RF-2.1, RF-2.4, RF-3.5, RF-3.6)* — after T-15
  **Done when:** `memberships.requiresSession401` and `memberships.noPagingParameterFullList` (twenty rows) pass under MockMvc, and the order changes with the locale header for an accented fixture.

- [ ] **T-17 · `POST /api/tenants/{tenantId}/entry`** — session required; 200 `{state: ENTERED, tenantId, tenantName, tenantRole}` · 403 `access.entry-refused` (no cause) · 403 `auth.enrollment.required {reason}` **[003]** · 401; the server never redirects. *(plan §2.1 `TenantEntryController`, D3, D4, D5, D13; RF-6.1, RF-6.3, RF-6.4, RF-10.1)* — after T-15
  **Done when:** the three outcomes are produced under MockMvc with the exact problem codes, the 403 for a refusal carries no cause, and the response for `ENTERED` carries exactly the three fields.

- [ ] **T-18 · API contract suite** — the listing exposes only the person's own facts and the names of their tenants and eligible projects; non-enterable cards carry no project line and expose state and role only; no member, count, administrator, source, run or other identity appears in any response; problems carry safe codes and a correlation id. *(D1, D2, D13; RF-2.6, RF-4.6, RF-5.5, RF-10.2, RF-10.4, RNF-1, RNF-5, LC-35)* — after T-16, T-17
  **Done when:** `memberships.exposeOnlyOwnFactsAndNames`, `memberships.nonEnterableCardsCarryNoProjectLines`, `memberships.nonEnterableCardsExposeStateAndRoleOnly` and `problems.safeCodesOnly` pass.

## F · Front end

- [ ] **T-19 · Contracts and hooks** — `contracts/memberships` types and runtime guards (the guard rejects any field beyond the contract); `useMemberships()` with `staleTime` 0, refetch on mount, no interval, focus or reconnect refetch, the background marker of plan 006 never set, one bounded retry on transport failure; `useEnterTenant()` mutation with no retry. *(plan §2.2 Public API, D8, D13; RF-3.1, RF-3.7, RF-6.1, RF-7.1, RF-7.2, LC-11)* — after T-01
  **Done when:** `useMemberships.noIntervalFocusOrReconnectRefetch`, `useMemberships.backgroundMarkerNeverSet` and `useMemberships.refetchesOnEveryMount` pass, and the guard rejects a response with an extra field.

- [ ] **T-20 · Route `/workspaces`** — the chooser route under `ProtectedBoundary` **[006]**; `LandingRouter` **[001]** maps `TENANT_CHOOSER` here; no session → `/sign-in`; the route renders whatever the list holds with no redirect logic of its own. *(plan §2.2 routes, D3; RF-1.1, RF-1.2, RF-1.3, RF-7.4, LC-2, LC-3)* — after T-19
  **Done when:** `chooserRoute.noSessionGoesToSignIn`, `chooser.rendersWithOneCard` and `chooser.rendersEmptyBlockWithZeroCards` pass.

- [ ] **T-21 · i18n bundle and `SecurityStatements` [001]** — keys in `identity-access.{en,es,fr}.json` for every heading, marker, statement, state label, action and card line of the four variants; `SecurityStatements` **[001]** extended with the N03 statements (memberships for this address only and project access checked after choosing; administration ≠ project content; accepted runs keep their pins; invitation or new workspace as the only ways in; content not exposed while suspended / deleting / membership suspended). *(plan §2.2 `SecurityStatements`, `shared/i18n`, D12; RF-8.1 … RF-8.5, RNF-12)* — after T-19
  **Done when:** `statements.identicalAcrossVariantsAndWidths` and `allVariants.existInEnEsFr` pass with no fallback to a key name.

- [ ] **T-22 · `TenantChooserPage` skeleton** — product mark, display name, language switch, marker, heading "Choose your workspace.", the address statement, the card list as a live region in server order, the "A context change is not a new target" block, the "No membership for a workspace?" block with "Create workspace" (→ the creation route of specification 012 with no state), "Sign out" (→ `SignOutCoordinator.signOut({entry: 'N03'})` **[007]**); no Help, Privacy or Terms. *(plan §2.2 `TenantChooserPage`, D9, D10; RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-3.5, RF-3.6, RF-8.1, RF-8.3, RF-8.4, LC-22, LC-23, LC-24)* — after T-20, T-21
  **Done when:** `chooserPage.rendersHeadingStatementsBlocksAndActions` and `chooserPage.rendersHelpAndNoPrivacyOrTerms` pass, and the language switch keeps the query data.

- [ ] **T-23 · `MembershipCard` — enterable variant** — tenant name, "Active membership", the role, "Your project grants" with one line per project (grants verbatim; "<project> · canRun only · no project role yet" when `canRunOnly`), or "No project grants yet — workspace administration does not grant project content" when `noGrants`; the statement "Workspace administration does not grant project content."; the MFA line by `mfaLine` with the "Not yet set up — you will enroll when you enter." suffix when `notYetEnrolled`; "Enter <tenant> →" always present. *(plan §2.2 `MembershipCard`, D1, D6; RF-4.1 … RF-4.6, RF-8.2, LC-12 … LC-17)* — after T-22
  **Done when:** `membershipCard.enterableRendersAllFields`, `membershipCard.noGrantsLineWithEnter` and `membershipCard.canRunOnlyLine` pass, and the MFA line renders the four variants with and without the suffix.

- [ ] **T-24 · `MembershipCard` — suspended-tenant and deleting-tenant variants** — suspended tenant as drawn ("Workspace suspended", "Your current membership", role with "retained, protected access unavailable", the statement, "View access status"); deleting tenant ("Deleting · content unavailable", role, the statement, "View access status"); neither offers "Enter" nor any project line; "View access status" navigates to the route of specification 014 with `tenantId` as requested scope. *(plan §2.2 `MembershipCard`, handoffs, D11; RF-5.1, RF-5.3, RF-5.4, RF-5.5, LC-6, LC-7)* — after T-22
  **Done when:** `membershipCard.suspendedTenantAsDrawnWithViewAccessStatus` and `membershipCard.deletingTenantStateOnlyWithViewAccessStatus` pass, and the navigation carries the tenant id only.

- [ ] **T-25 · `MembershipCard` — suspended-membership variant** — tenant name, "Your membership is suspended · content unavailable", the role, "Project names and content are not exposed while your membership is suspended.", no action, no administrator named. *(plan §2.2 `MembershipCard`, D1; RF-5.2, RF-5.5, RF-5.6, LC-5)* — after T-22
  **Done when:** `membershipCard.suspendedMembershipNoActionNoAdmin` passes and a fixture with one suspended membership and two enterable ones renders three cards in server order.

- [ ] **T-26 · `EntryRefusedHandler`** — in `shared/errors`: on `access.entry-refused` from the entry mutation, invalidate and refetch `useMemberships`, keep the page, announce "Memberships updated" in the live region without moving focus; no toast, no dialog, no navigation. *(plan §2.2 `EntryRefusedHandler`, D4, D8; RF-6.3, RNF-4, LC-8, LC-9, LC-10)* — after T-23
  **Done when:** `entryRefusedHandler.refetchesAndRendersNewState` and `enter.neverUsesCachedListToDecide` pass.

- [ ] **T-27 · Entry handoffs** — on `ENTERED`: advance the scope epoch **[001/006]**, clear the scoped cache and navigate to the tenant overview route with `tenantId`; on `auth.enrollment.required {reason}`: `EnrollmentGateHandler` **[003]** with `returnTo` = the overview route, whose "Leave setup" returns to `/workspaces`. *(plan §2.2 Routing, D3, D5, D7; RF-6.2, RF-6.4, RF-6.5, LC-12, LC-13, LC-18)* — after T-26
  **Done when:** `enterTenant.advancesEpochClearsScopedCacheNavigatesToOverview` passes and a unit test shows the gate problem routed through `EnrollmentGateHandler` with the overview route as `returnTo`.

- [ ] **T-28 · `SwitchWorkspaceAction` in `app/shell` [007]** — a labelled "Switch workspace" action beside `SignOutAction` **[007]** in the footer identity block of every desktop app-shell route and as an item of the M04 menu; activation runs the shell dirty guard of plan 007 (N24 Save / Discard / Stay; a guard save under the current tenant, project and `baseETag`) and navigates to `/workspaces` on Save acknowledged or Discard, nothing on Stay; the current scope epoch and tenant context are kept until an entry succeeds. *(plan §2.2 `SwitchWorkspaceAction`, D7; RF-1.4, RF-1.5, RNF-8, LC-27, LC-28)* — after T-20
  **Done when:** `shell.switchWorkspaceBesideSignOutOnEveryAuthenticatedRouteAndM04Menu`, `switchWorkspace.dirtyGuardSaveDiscardStayBeforeChooser` and `switchWorkspace.guardSaveUnderOriginalScope` pass.

- [ ] **T-29 · Exits of other features wired to `/workspaces`** — "Leave setup" of a forced enrollment **[003]**, the re-authentication fallback **[006]**, and "Leave invitation", "Return to memberships" and a decline **[008]** navigate to `/workspaces`; each arrival mounts the page and refetches; the returns from the 014 route and from an abandoned creation route likewise. *(plan §2.2 routes, D8; RF-1.2, RF-7.1, LC-21, LC-25, LC-26, LC-29)* — after T-20
  **Done when:** a routing unit test shows each exit landing on `/workspaces` and `useMemberships.refetchesOnEveryMount` passes across them.

- [ ] **T-30 · Width, language and hygiene unit tests** — the four variants at desktop, tablet and mobile widths with the same cards, statements and actions; keyboard order equals server order; the language switch keeps cards and order; nothing in `localStorage`, `sessionStorage` or IndexedDB after every state and exit; a reload refetches. *(plan §2.2, D9, D10, D12; RF-2.4, RF-7.3, RF-9.1, RF-9.2, RF-9.3, RNF-10, RNF-13, LC-24, LC-31)* — after T-23, T-24, T-25, T-28
  **Done when:** `storage.containsNothingFromChooser`, `reload.refetches` and `languageSwitch.keepsCardsAndOrder` pass and the width tests assert the same statement set at the three widths.

## G · Verification suites

- [ ] **T-31 · Listing suite** — own rows only, revoked absent even when the only membership, no row for a tenant without a membership, every status pair classified with tenant state first, order under the three collations with ties, project lines equal to live grants alphabetical, no project name on non-enterable rows, the MFA line from policy and factor status. *(D1, D2, D6, D9; RF-3, RF-4.2 … RF-4.5, RF-5.5, RNF-1, RNF-5, RNF-6, LC-4, LC-5, LC-12 … LC-20, LC-34)* — after T-15
  **Done when:** every test named in T-04 … T-08 and T-12 passes in one suite run against real PostgreSQL.

- [ ] **T-32 · Entry suite** — a derived entry returns routing facts only; a refusal after a suspension, a revocation or a tenant state change is audited with its cause and answered with one code; the gate raises `auth.enrollment.required`; entry writes nothing and replays nothing; every non-enterable state refuses. *(D3, D4, D5; RF-6.1 … RF-6.6, RF-10.1, RNF-3, RNF-7, LC-8 … LC-10, LC-18)* — after T-17
  **Done when:** the tests named in T-09, T-14 and T-15 pass in one suite run, plus `entry.writesNothingReplaysNothing` with factor, credential, session, membership and run rows compared before and after.

- [ ] **T-33 · Concurrency suite** — two tabs entering two different tenants; an entry racing a suspension; an entry racing a revocation. *(D3; RF-6.1, LC-32)* — after T-17
  **Done when:** `entry.twoTabsTwoTenantsIndependentDerivations` passes ten times in a row with real PostgreSQL, and each race yields either a derived entry or an audited refusal, never both.

- [ ] **T-34 · Freshness suite** — refetch on every mount and after a refused entry; no interval, focus or reconnect refetch; background marker never set; nothing persisted; a change while idle appears only on the next open or Enter. *(D8, D10; RF-3.7, RF-7.1 … RF-7.3, RNF-4, RNF-10, LC-11, LC-31)* — after T-26, T-30
  **Done when:** the tests named in T-19, T-26 and T-30 pass in one run, and a membership reinstated while the page is idle appears only after a remount or a refused entry.

- [ ] **T-35 · No-authority and fail-closed suite** — the listing alone enters nothing; the entry result alone derives nothing; every non-enterable state refuses entry; an ended session shows R34 and re-authentication returns to the chooser with a fresh list. *(D1, D3; RF-1.6, RF-7.4, RNF-2, RNF-3, LC-30, LC-33)* — after T-32, T-34
  **Done when:** `chooserGrantsNoAuthority`, `failClosed.everyNonEnterableStateRefusesEntry` and `chooser.sessionEndedShowsR34ReauthReturnsFresh` (with the 006 stub) pass.

- [ ] **T-36 · FE e2e journeys** — Playwright journeys for US-1 … US-9 in EN, ES and FR at desktop, tablet and mobile widths, with the 001, 003, 006, 007, 008, 012 and 014 surfaces built or stubbed in their plans' shapes: landing with two memberships → chooser → enter; switch workspace with a dirty editor → guard → chooser → enter another tenant; gated Owner → wizard → "Leave setup" → chooser; suspended tenant → "View access status" → N20 → back; suspended membership beside enterable tenants; no membership → "Create workspace"; accepted invitation then "Switch workspace" lists the new tenant; decline and "Leave invitation" arrivals; "Sign out" from N03; sign-out from the shell untouched; language switch; after a grant revocation the overview omits the project; session expiry on the chooser. *(RF-1.2 … RF-1.5, RF-2.2, RF-2.3, RF-5.4, RF-6.2, RF-6.4, RF-6.5, RF-7.4, LC-1, LC-21 … LC-30; US-1 … US-9)* — after T-30, T-18
  **Done when:** `chooser.reachedFromLandingLeaveSetupInvitationExitsAndReauth`, `createWorkspace.offeredAlwaysAndCarriesNoTenant`, `chooser.signOutEntryN03NoGuard`, `viewAccessStatus.opensN20WithoutEntryAndReturnsRefetching`, `enter.gatedOwnerReachesWizardLeaveSetupReturnsToChooser`, `enter.afterGrantRevocationOverviewOmitsProject` and `acceptedInvitation.thenSwitchWorkspaceListsNewTenant` pass in the three languages and three widths.

- [ ] **T-37 · Automated accessibility** — axe on the page with each of the four card variants, with one card, with no card and after a refetch, at the three widths and three languages. *(RF-9.1, RF-9.2, RNF-11, RNF-12, RNF-13)* — after T-36
  **Done when:** `a11y.chooser.{en,es,fr}.{desktop,tablet,mobile}` reports zero critical or serious violations on all 36 combinations.

- [ ] **T-38 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist: card regions announced with their state, order as rendered, "Memberships updated" announced after a refused entry without focus theft, labels on "Enter", "View access status", "Create workspace", "Sign out" and "Switch workspace", 44 px targets; versions recorded in the release-support matrix. *(RF-9.2, RNF-11; FR-X-040)* — after T-37
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-39 · Adversarial suite (US-10)** — enumeration of tenants and members through the listing; a tenant the person does not belong to; project names of a suspended tenant or of a project without a grant; a stale entry after a membership change; a crafted entry request naming another tenant; a response field beyond the contract; any authority derived from the page alone. *(D1, D2, D3; RF-2.6, RF-3.3, RF-5.5, RF-6.3, RF-10.3, RF-10.4, RNF-1, RNF-2, RNF-3, RNF-5; US-10)* — after T-18, T-33, T-35
  **Done when:** every attack test fails to achieve its goal and asserts an empty result, a refusal or an unchanged state.

## H · Closing

- [ ] **T-40 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-31 … T-39
  **Done when:** the script reports 103 of 103 identifiers (55 RF, 13 RNF, 35 LC) covered by at least one passing test.

- [ ] **T-41 · Register-dependency and cross-plan review** — walk §6 of the plan (A-36, A-54, A-35; S-55, S-56, S-48, S-07, S-02, S-03, S-17): for each, confirm the built assumption matches the current register state or record the deferral; confirm the four cross-plan items are recorded in plan 009 (`MembershipProjectionPort` extension), in the first projects-area plan (`ProjectNamesPort`), in specification 001 (LC-15 wording, pending approval) and in specification 014 (tenant id as requested scope, deleting wording). *(spec §7 item 12)* — after T-40
  **Done when:** each of the ten register items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it, and each cross-plan item has its target document named.

- [ ] **T-42 · Completion criteria sign-off** — tick spec §7 items 1–12 with the evidence (test run id, concurrency log, a11y report, manual checklist, coverage report, register review). — after T-41
  **Done when:** all twelve items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-16, T-20 |
| RF-1.2 | T-20, T-29, T-36 |
| RF-1.3 | T-20 |
| RF-1.4 | T-28, T-36 |
| RF-1.5 | T-28, T-36 |
| RF-1.6 | T-08, T-35 |
| RF-2.1 | T-08, T-16, T-22 |
| RF-2.2 | T-22, T-36 |
| RF-2.3 | T-22, T-36 |
| RF-2.4 | T-16, T-22, T-30 |
| RF-2.5 | T-22 |
| RF-2.6 | T-07, T-11, T-18, T-39 |
| RF-3.1 | T-03, T-08, T-12, T-15, T-19 |
| RF-3.2 | T-04, T-08, T-12, T-31 |
| RF-3.3 | T-08, T-12, T-39 |
| RF-3.4 | T-04, T-08, T-31 |
| RF-3.5 | T-05, T-08, T-16, T-22, T-31 |
| RF-3.6 | T-08, T-16, T-22 |
| RF-3.7 | T-08, T-19, T-34 |
| RF-4.1 | T-07, T-23 |
| RF-4.2 | T-02, T-03, T-05, T-08, T-12, T-13, T-23, T-31 |
| RF-4.3 | T-07, T-23 |
| RF-4.4 | T-06, T-08, T-23, T-31 |
| RF-4.5 | T-06, T-08, T-23, T-31 |
| RF-4.6 | T-07, T-18 |
| RF-5.1 | T-04, T-24 |
| RF-5.2 | T-04, T-25 |
| RF-5.3 | T-04, T-24 |
| RF-5.4 | T-24, T-36 |
| RF-5.5 | T-03, T-07, T-08, T-12, T-18, T-24, T-25, T-31, T-39 |
| RF-5.6 | T-04, T-25 |
| RF-6.1 | T-09, T-15, T-17, T-19, T-32, T-33 |
| RF-6.2 | T-27, T-36 |
| RF-6.3 | T-09, T-17, T-26, T-32, T-39 |
| RF-6.4 | T-09, T-17, T-27, T-32, T-36 |
| RF-6.5 | T-09, T-11, T-27, T-32, T-36 |
| RF-6.6 | T-09, T-32 |
| RF-7.1 | T-08, T-19, T-29, T-34 |
| RF-7.2 | T-19, T-34 |
| RF-7.3 | T-30, T-34 |
| RF-7.4 | T-20, T-35, T-36 |
| RF-8.1 | T-21, T-22 |
| RF-8.2 | T-21, T-23 |
| RF-8.3 | T-21, T-22 |
| RF-8.4 | T-21, T-22 |
| RF-8.5 | T-21 |
| RF-9.1 | T-30, T-37 |
| RF-9.2 | T-30, T-37, T-38 |
| RF-9.3 | T-30 |
| RF-10.1 | T-09, T-10, T-14, T-17, T-32 |
| RF-10.2 | T-10, T-14, T-18 |
| RF-10.3 | T-08, T-12, T-39 |
| RF-10.4 | T-07, T-18, T-39 |
| RNF-1 | T-07, T-11, T-12, T-18, T-31, T-39 |
| RNF-2 | T-11, T-35, T-39 |
| RNF-3 | T-15, T-32, T-35, T-39 |
| RNF-4 | T-26, T-34 |
| RNF-5 | T-12, T-18, T-31, T-39 |
| RNF-6 | T-12, T-31 |
| RNF-7 | T-32 |
| RNF-8 | T-28 |
| RNF-9 | T-01 |
| RNF-10 | T-01, T-30, T-34 |
| RNF-11 | T-37, T-38 |
| RNF-12 | T-21, T-37 |
| RNF-13 | T-30, T-37 |
| LC-1 | T-36 |
| LC-2 | T-20 |
| LC-3 | T-20 |
| LC-4 | T-04, T-12, T-31 |
| LC-5 | T-04, T-25, T-31 |
| LC-6 | T-04, T-24 |
| LC-7 | T-04, T-24 |
| LC-8 | T-09, T-26, T-32 |
| LC-9 | T-09, T-26, T-32 |
| LC-10 | T-09, T-26, T-32 |
| LC-11 | T-19, T-34 |
| LC-12 | T-06, T-09, T-23, T-27, T-31 |
| LC-13 | T-06, T-09, T-23, T-27, T-31 |
| LC-14 | T-06, T-23, T-31 |
| LC-15 | T-07, T-23, T-31 |
| LC-16 | T-07, T-23, T-31 |
| LC-17 | T-08, T-23, T-31 |
| LC-18 | T-27, T-32, T-36 |
| LC-19 | T-05, T-31 |
| LC-20 | T-05, T-31 |
| LC-21 | T-29, T-36 |
| LC-22 | T-22, T-36 |
| LC-23 | T-22, T-36 |
| LC-24 | T-22, T-30, T-36 |
| LC-25 | T-29, T-36 |
| LC-26 | T-29, T-36 |
| LC-27 | T-28, T-36 |
| LC-28 | T-28, T-36 |
| LC-29 | T-29, T-36 |
| LC-30 | T-35, T-36 |
| LC-31 | T-30, T-34 |
| LC-32 | T-33 |
| LC-33 | T-35 |
| LC-34 | T-12, T-31 |
| LC-35 | T-18 |
| US-1 … US-9 | T-36 (journeys) |
| US-10 | T-39 (adversarial suite) |

Self-check: all 55 RF criteria, 13 RNF and 35 limit cases of `specTenantChooser.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.
