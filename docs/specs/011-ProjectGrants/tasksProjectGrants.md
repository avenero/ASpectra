# Tasks 011 — Project grants editor

**Feature** The Project grants page of a project: listing who holds which grants, editing one member's Author / Reviewer / Viewer and `canRun`, adding a tenant member to the project and removing one from it · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specProjectGrants.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planProjectGrants.md` v0.1 (modules §2, decisions D1–D15, tests §5, register assumptions §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** … **[009]** extend an element that an earlier plan builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states. This feature consumes plan 009's `access` context (`GrantCommands.SetProjectGrants`, `AuthorityChecker`, `AdministrationRule`, `PermissionVersionGuard`, `AuthorityChanged`, `AuthorityStore`, `useAuthority`, `AuthorityStatements`) and plan 006's boundary (`DeniedScopeHandler`, `TransportFailureStatus`, the stamped drafts registry) — those elements are prerequisites, not tasks of this file. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plan 009's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-03 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Grant list configuration record** — `GrantListPolicy` immutable record (list page size, picker page size) and `GrantListPolicyProperties` binding, each value `@Validated` as present, positive and bounded above by a configured ceiling; no default anywhere. *(plan §2.1 `GrantListPolicy`, `GrantListPolicyProperties`, D13; RF-2.1, RF-5.1, RNF-11)*
  **Done when:** `grantListPolicy.startupFailsOnAbsentOrUnboundedSize` passes: the context starts with both values set and fails to start, naming the property, when either is absent, zero, negative or above the ceiling.

- [ ] **T-02 · Migration: list index on `project_grant` [009]** — add the index `(tenant_id, project_id)` on `project_grant`; no column change; RLS policies of plan 009 untouched. *(plan §2.1 Persistence, D5; RF-2.1, RF-5.1, RNF-4)* — after T-01
  **Done when:** the migration applies on an empty database and on one holding plan 009 rows, and the planner uses the new index for the list query's `tenant_id = ? AND project_id = ?` predicate.

- [ ] **T-03 · Problem-code catalogue** — register `access.grants.stale` (409), `access.grants.target-not-active` (409), `access.grants.project-not-editable` (409) and `access.grants.nothing-chosen` (400, with field path) beside the reused `access.denied-scope` (403) and `access.forbidden-change` (403) **[009]**; every problem is RFC 9457 with a correlation id and no reason class, content or tenant existence. *(plan §2.1 Problem codes, D15; RF-1.3, RF-8.4, RF-10.2)* — after T-01
  **Done when:** the catalogue test lists exactly the six codes for this feature with their statuses, and a sample problem of each serialises with `type`, `title`, `status`, `correlationId` (plus `fieldPath` for the 400) and nothing else.

- [ ] **T-04 · Ports `ProjectStatePort` and `IdentityNamesPort`** — declare in `access.application`: `ProjectStatePort.stateOf(tenantId, projectId) → Optional<ProjectState>` (published by `project`) and `IdentityNamesPort.namesOf(Set<identityId>) → Map<identityId, {displayName, verifiedEmail}>` (published by `identity`; nothing else of the identity); implement each in its owning context as a narrow read-only adapter — the project adapter reads the project's lifecycle state in the tenant under RLS, the identity adapter reads display name and verified email by id only. *(plan §2.1 Ports, D5, D7; RF-1.5, RF-2.2, RF-2.5, RF-4.5, RNF-6)* — after T-01
  **Done when:** integration tests show `stateOf` returning each of the four states and empty for a project of another tenant or an unknown id, and `namesOf` returning name and email for the given ids and no other attribute; an ArchUnit rule shows `access` reaches `project` and `identity` only through these two ports and `ProjectExistsPort` **[009]**.

## B · Domain (pure, no Spring)

- [ ] **T-05 · `ProjectState`** — closed enumeration `ACTIVE | ARCHIVED | TRASHED | DELETING` with `isEditable()` = `ACTIVE ∨ ARCHIVED`. *(plan §2.1 `ProjectState`, D7; RF-1.5, RF-4.5, RF-5.1)* — after T-01
  **Done when:** a unit test shows `isEditable()` true for exactly `ACTIVE` and `ARCHIVED`, and the enum has no fifth value.

- [ ] **T-06 · `GrantEditRule`** — pure rule `(targetStatus, projectState, currentGrant?, requestedSet, expectedVersion, currentVersion) → Apply(SET | REMOVE) | Refuse(reasonClass)`: `TARGET_NOT_ACTIVE` when the membership is not `ACTIVE`; `PROJECT_NOT_EDITABLE` when `¬isEditable()`; `STALE_STATE` when `expectedVersion` is present and differs from `currentVersion`; `NOTHING_CHOSEN` when the set is empty and no grant exists; `UNCHANGED` when the set equals the current grant; `Apply(REMOVE)` for an empty set on an existing grant; `Apply(SET)` otherwise; no default branch; evaluated in that order. *(plan §2.1 `GrantEditRule`, D2, D3; RF-4.1, RF-4.4, RF-4.5, RF-4.6, RF-5.4, RF-5.5, RF-6.2, RNF-5, LC-7, LC-8, LC-9, LC-10, LC-11, LC-12, LC-18, LC-19)* — after T-05
  **Done when:** `grantEditRule.exhaustiveTruthTable` passes over every combination of target status (3) × project state (4) × current grant (none / some) × requested set (empty / same / different) × version (match / mismatch / absent), with every expected outcome listed explicitly and no combination unspecified.

- [ ] **T-07 · `ProjectGrantRow` and `GrantCandidateRow`** — read models: `ProjectGrantRow` (`membershipId`, `displayName`, `tenantRole`, `membershipStatus` ∈ {`ACTIVE`, `SUSPENDED`}, `GrantSet` **[009]**, `inertCanRun` = `canRun ∧ ¬(Author ∨ Reviewer)`, `isActor`, `permissionVersion`, `editable` = status `ACTIVE` ∧ project editable); `GrantCandidateRow` (`membershipId`, `displayName`, `tenantRole`). Neither carries a verified email, a revoked membership, content or a reason class. *(plan §2.1 read models, D5; RF-1.6, RF-2.2, RF-2.3, RF-2.4, RF-2.6, RF-3.1, RF-4.6, RF-5.1, RF-5.2, RNF-9, RNF-12)* — after T-05
  **Done when:** `row.inertCanRunComputed` passes for all eight role × canRun combinations; a reflective check lists exactly the nine fields of `ProjectGrantRow` and the three of `GrantCandidateRow`; `editable` is false for a suspended membership and for a trashed project.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-12…T-14); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-08 · `GrantCommands.SetProjectGrants` [009] — expected version, project state and refusals** — input gains `expectedVersion` (optional; absent for plan 012's bootstrap). Sequence, one transaction: `AuthorityChecker.require(C10, TENANT_MEMBERS)`; recorded outcome for a repeated idempotency key; `AdministrationRule` **[009]** (Owner or Admin; self allowed); `ProjectExistsPort` **[009]** and `ProjectStatePort`; lock the target membership `FOR UPDATE`; `GrantEditRule`; `Refuse` → audit `GRANT_CHANGE_REFUSED{reasonClass}` and the matching problem, nothing written; `UNCHANGED` → current row, nothing written; `Apply` → `setGrants` **[009]**, version advance, `GRANTS_SET` / `GRANT_REMOVED`, `AuthorityChanged` **[009]**, outcome recorded. *(plan §2.1 `SetProjectGrants`, D1, D2, D3, D9, D12; RF-4.1, RF-4.2, RF-4.3, RF-4.4, RF-4.5, RF-4.6, RF-4.7, RF-4.9, RF-5.4, RF-5.5, RF-5.6, RF-6.2, RF-6.4, RF-10.1, RF-10.4, RNF-3, RNF-4, RNF-10, LC-4, LC-8, LC-9, LC-10, LC-11, LC-12, LC-17, LC-18, LC-19, LC-30)* — after T-06, T-07, T-04
  **Done when:** unit tests with recording doubles show, for each outcome of T-06, exactly the expected writes (none on `Refuse` / `UNCHANGED`; one grant write, one version advance, one audit entry, one outbox event and one outcome record on `Apply`), the refusal audit entry on every `Refuse`, the recorded outcome on a repeated key, and zero calls to any membership, session, notification or run port.

- [ ] **T-09 · `ProjectGrantsQuery` [009 declared]** — for the actor (gated by `TENANT_MEMBERS`): `project_grant` rows of the project joined to `tenant_membership` with status `ACTIVE` or `SUSPENDED`; names and emails through `IdentityNamesPort`; `status` filter (all / active / suspended); `q` matching display name or verified email; order by display name; page by the list size; envelope `{items, page, pageCount, total, projectState, editable}`; a page beyond the last returns the last page. *(plan §2.1 `ProjectGrantsQuery`, D5, D7; RF-1.5, RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-7.3, LC-1, LC-3, LC-13, LC-20, LC-21, LC-22)* — after T-07, T-04
  **Done when:** unit tests with five memberships (three active with grants, one suspended with a grant, one revoked with a historical grant) and a page size of two show two pages ordered by name, the suspended row `editable = false`, the revoked membership absent, the status filter and the email search narrowing correctly, `editable = false` in the envelope for a trashed project, and page 9 returning page 2.

- [ ] **T-10 · `GrantCandidatesQuery`** — for the actor (gated by `TENANT_MEMBERS`; refused with `PROJECT_NOT_EDITABLE` while the project is not editable): `tenant_membership` rows with status `ACTIVE` and no `project_grant` on the project (anti-join), the actor's included; names through `IdentityNamesPort`; `q` by display name or verified email; order by display name; page by the picker size. *(plan §2.1 `GrantCandidatesQuery`, D6, D7; RF-5.1, RF-5.2, LC-1, LC-21, LC-22)* — after T-09
  **Done when:** unit tests show the actor listed when they hold no grant, a member with a grant absent, suspended and revoked memberships absent, another tenant's memberships absent, and a refusal for a trashed project.

- [ ] **T-11 · Module boundary and port absence** — ArchUnit rules: the page's use cases import no project-content, approval, run, intent, session, membership-command or notification package; `access` reaches `project` and `identity` only through the three ports; no code path of this feature references `SecurityNotificationPort` or `TransactionalEmailPort`. *(plan §5.1 Module boundary, D1, D11, D12; RF-4.8, RF-4.9, RF-8.1, RF-8.2, RNF-12)* — after T-08, T-09, T-10
  **Done when:** `module.importsNoMembershipSessionApprovalRunOrContentPackage` and `feature.neverCallsNotificationPort` pass.

## D · Infrastructure adapters

- [ ] **T-12 · Persistence adapters** — implementations (access API per A-35) of the list join (using the T-02 index), the status filter and search predicates, the candidates anti-join over `tenant_membership` and `project_grant`, and paging; the exclusive lock of `SetProjectGrants` reuses `PermissionVersionGuard` **[009]**. *(plan §2.1 Persistence, D5, D6; RF-2.1, RF-2.5, RF-5.1, RNF-4)* — after T-02, T-09, T-10
  **Done when:** integration tests with real PostgreSQL under the runtime role show the list ordered and paged as T-09 states, the anti-join excluding members with a grant, and both queries returning nothing for a project id of another tenant.

- [ ] **T-13 · Audit adapter [009]** — map `GRANT_CHANGE_REFUSED{reasonClass, actionClass}` to `AuditPort` entries with acting and target membership and identity, tenant, project, correlation id and time, written in the command's transaction; `GRANTS_SET` and `GRANT_REMOVED` unchanged. *(plan §2.1 Audit adapter, D12; RF-10.1, RF-10.3)* — after T-08
  **Done when:** an integration test performs one set, one removal and one refusal of each reason class and finds exactly the expected entries with the expected payload keys and no key named like a secret, email or content.

- [ ] **T-14 · Problem mapping and redaction** — map each `Refuse` reason class and each `AuthorityChecker` / `AdministrationRule` refusal to its code of T-03; request and response logging for the three endpoints exclude the authorisation header and cookies; the list and candidate serialisers emit the read-model fields only. *(plan §2.1 Problem codes, D15; RF-1.6, RF-10.2, RF-10.3)* — after T-03, T-07
  **Done when:** a mapping test shows every reason class reaching exactly one code, and a log-capture test over the three endpoints finds no credential, email or project content.

## E · API

- [ ] **T-15 · `GET /api/tenants/{t}/projects/{p}/grants`** — C10 filter **[009]**; query `page`, `q`, `status`; 200 the envelope of T-09; 403 `access.denied-scope` for a Member, a suspended actor or a foreign project. *(plan §2.1 `ProjectGrantsController`, D4, D5; RF-1.2, RF-1.3, RF-1.5, RF-2.1, RF-2.5)* — after T-09, T-14
  **Done when:** `grantsPage.readOnlyWhileTrashedOrDeleting` and `grantsPage.editableWhileArchived` pass at the contract level (`editable` and `projectState` in the envelope for each of the four states), and an invalid `status` value answers 400 with a field path.

- [ ] **T-16 · `GET …/grants/candidates`** — C10 filter; query `page`, `q`; 200 `{items, page, pageCount, total}`; 409 `access.grants.project-not-editable` while trashed or deleting; 403 `access.denied-scope` as T-15. *(plan §2.1 `ProjectGrantsController`, D6; RF-5.1)* — after T-10, T-14
  **Done when:** a contract test shows the shape, the 409 for a trashed project and the 403 for a Member.

- [ ] **T-17 · `PUT …/grants/{membershipId}`** — C10 filter, CSRF-protected; body `{author, reviewer, viewer, canRun, expectedVersion, idempotencyKey}` with all four flags mandatory (a missing flag → 400 with its field path, never a default); 200 the updated row; 204 when removed; 400 `access.grants.nothing-chosen`; 403 `access.denied-scope` / `access.forbidden-change`; 409 `stale` / `target-not-active` / `project-not-editable`. *(plan §2.1 `ProjectGrantsController`, D3; RF-4.1, RF-4.5, RF-4.6, RF-4.7, RF-5.4, RF-5.5, RF-6.2, RF-8.3, RF-8.4)* — after T-08, T-14
  **Done when:** `save.emptyOnNoGrantIs400` passes, each of the seven answers above is produced by a contract case, and a body with three flags answers 400 naming the missing one.

- [ ] **T-18 · Contract and absence checks** — a test enumerating the API surface: the six problem codes with correlation id; the four mandatory flags; no verified email, project content, reason class, foreign-tenant row or tenant existence on the wire; no endpoint of this feature changes a membership, a role or a session. *(plan §5.1 API contract, D3, D15; RF-1.6, RF-8.1, RF-8.3, RF-8.4, RF-10.2, RNF-9)* — after T-15, T-16, T-17
  **Done when:** `problems.sixSafeCodesWithCorrelationId`, `save.bodyIsFourMandatoryFlagsOnly` and `grants.wireCarriesNoContentEmailCredentialOrForeignRow` pass.

## F · Front end

- [ ] **T-19 · `contracts/access-grants` types and guards** — request / response / problem types with runtime guards for the three endpoints, the `ProjectState` enum, the six problem codes; the grant flags reuse the `GrantSet` type of `contracts/access` **[009]**. *(plan §2.2 `contracts/access-grants`; RF-4, RF-5, RF-10.2)* — after T-03
  **Done when:** guard tests accept the documented shapes, reject a row with an extra field named `email` or `userAgent`, reject a body with a missing flag, and reject an unknown problem code.

- [ ] **T-20 · i18n bundles `project-settings.{en,es,fr}`: grant keys** — keys for the heading, the sub-statement, the tab names, the four column headings, "/ you", role names and "+", "Granted" / "Not granted", the suspended state and reinstatement statement, the inert note (reusing the `access` key **[009]**), the panel title variants, the two panel statements, the removal notice, "Cancel changes" / "Save project grants", "No changes to save", "Grants saved" / "Removed from project", the stale, inactive-target, not-editable and nothing-chosen messages, the picker title and search label, the removal dialog statements (second-person variant included) and actions, "Add project member", the read-only notice, "Decision history", "Find a project member", "All statuses" and its values. *(plan D14; RF-11.3, RNF-14)* — after T-19
  **Done when:** a bundle-parity test shows every key present in all three locales with no empty value.

- [ ] **T-21 · Hooks `useProjectGrants`, `useGrantCandidates`, `useSetProjectGrants`** — queries keyed per tenant, project, page, query and status, cleared with the scope epoch **[001/006]**, no interval and no focus refetch, invalidated after every mutation; the mutation never retries, carries `expectedVersion` and the idempotency key, and returns server state verbatim. *(plan §2.2 hooks, D9; RF-2.7, RF-4.7, RNF-10)* — after T-19
  **Done when:** `grants.fetchOnOpenAndAfterMutationsNeverInterval` passes and a failed mutation records zero automatic retries.

- [ ] **T-22 · Routes and tab guard** — `/t/:tenant/p/:project/settings/members` registered as the "Project members" tab of project settings; reached from the project settings tabs and from "Review project grants" on the members page **[010]** with the current project; the tab and its route hidden when `useAuthority(TENANT_MEMBERS)` **[009]** is not permitted; a direct request still answers the server's `access.denied-scope` through `DeniedScopeHandler` **[006]**. *(plan §2.2 `routes`, D4; RF-1.1, RF-1.2, RF-1.3, RF-8.5)* — after T-21
  **Done when:** `tab.hiddenWithoutTenantMembersAuthority` passes for a Member fixture and the tab renders for Owner and Admin fixtures with and without a grant on the project.

- [ ] **T-23 · `ProjectMembersTab` shell** — breadcrumb, heading "Project grants", the sub-statement, "Add project member" rendered only while `editable`, the tabs, `ReadOnlyNotice` naming the reason while `projectState` is `TRASHED` or `DELETING`, `AuthorityStatements` **[009]** variant `tenantAdminScope` (new variant with the R17 card text) and variant `grants`, record count placeholder, and "Decision history ↗" routing to the audit trail **[F-APR-04]** scoped to the project's grant events. *(plan §2.2 `ProjectMembersTab`, D14; RF-1.1, RF-1.5, RF-2.8, RF-7.1, RF-7.2, RF-7.3)* — after T-22, T-20
  **Done when:** `readOnlyNotice.statesReason`, `statements.panelCardAndSubStatementVerbatimOnEveryVariant` (tab half), `decisionHistory.routesToProjectGrantAuditScope` and `unavailable.alwaysExplainedNeverSilent` pass.

- [ ] **T-24 · `SearchAndStatusFilter` and `Pagination` [007]** — "Find a project member" (⌘K) bound to `q`; "All statuses" with all / active / suspended bound to `status`; no "Current revisions" control; `Pagination` from `page` / `pageCount` with the unavailable state in text and the record count. *(plan §2.2 `SearchAndStatusFilter`, `Pagination`; RF-2.1, RF-2.5, LC-22, LC-28)* — after T-23
  **Done when:** `filters.currentRevisionsNotRendered` passes, the search and the filter change the query key, and the pagination text state is correct on the first, a middle and the last page.

- [ ] **T-25 · `GrantsTable` — cells and row states** — columns Member ("/ you" on `isActor`), Project grants (roles joined by "+"), canRun ("Granted" / "Not granted" with the inert note from `AuthorityStatements.inertCanRun` **[009]** when `inertCanRun`), Tenant role; a `SUSPENDED` row shows the state and the reinstatement statement, its grants read-only, and cannot open the panel; an editable row's activation requests the panel through the single-panel guard; mobile and tablet render one record per member with the same cells. *(plan §2.2 `GrantsTable`, D5, D14; RF-2.2, RF-2.3, RF-2.4, RF-2.6, RF-3.1, RF-11.1, LC-5, LC-20)* — after T-24
  **Done when:** `table.rendersFourCellsAndYouMarker`, `table.suspendedRowShowsStateAndStatement` and `table.inertNoteBesideGranted` pass, and a snapshot at three widths shows the four cells for a fixture with every role combination.

- [ ] **T-26 · `GrantEditPanel` — structure and statements** — title "Edit <name>'s grants" (or "Add <name> to this project" in Add mode); the tenant role beside the title; `CheckboxGroup` "Project roles" (Author, Reviewer, Viewer) and `CheckboxGroup` "Independent execution permission" (canRun), each box independent and reflecting the row's current grant; the two drawn statements; the inert note while `canRun ∧ ¬(Author ∨ Reviewer)`; no other control. *(plan §2.2 `GrantEditPanel`, D3; RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.8, LC-5, LC-6)* — after T-25
  **Done when:** `panel.opensWithCurrentFlagsAndTenantRoleOnly`, `panel.checkboxesIndependent`, `panel.inertNoteWhileCanRunWithoutRole` and `statements.panelCardAndSubStatementVerbatimOnEveryVariant` (panel half) pass.

- [ ] **T-27 · `GrantEditPanel` — notices, Cancel, draft registration, single panel** — the removal notice while nothing is ticked on an existing grant; "Cancel changes" purges the draft and closes; while dirty the panel holds a stamped draft `{identityId, tenantId, projectId, kind: 'grants', membershipId}` in the Zustand drafts registry **[006 D10]** so the switch and sign-out guards see it, with the panel's own save as the guard's "Save" branch under the stamped scope; at most one panel open — activating another row with a dirty panel passes through the same guard. *(plan §2.2 `GrantEditPanel`, D8; RF-3.5, RF-3.6, RF-3.7, LC-25, LC-26)* — after T-26
  **Done when:** `panel.removalNoticeWhenNothingTicked`, `panel.cancelPurgesDraftAndCloses` and `panel.registersStampedDraftAndSinglePanelGuard` pass, and the registry is empty after Cancel, after a confirmed save and after an epoch change.

- [ ] **T-28 · Save flow and outcome handling** — "Save project grants": an existing grant with nothing ticked → `RemoveFromProjectDialog` (T-29); otherwise `useSetProjectGrants` with the row's `permissionVersion` as `expectedVersion` and a key generated once per open; no step-up and no dialog for a non-empty save. Outcomes: 200 → row updated, panel closed, "Grants saved" in the live region; 204 → "Removed from project", row gone at the next read; `access.grants.stale` → "The grants changed since you opened this panel", row refetched, panel reopened on the current values with the new version; `target-not-active` / `project-not-editable` → stated reason, row refetched, panel closed; `nothing-chosen` → inline message; `forbidden-change` / `denied-scope` → `DeniedScopeHandler` **[006]**; no answer → `TransportFailureStatus` **[006]** on the panel, repeatable with the same key. *(plan §2.2 Outcome handling, D9, D15; RF-4.3, RF-4.5, RF-4.6, RF-4.7, RF-5.6, RF-10.2, LC-9, LC-10, LC-11, LC-12, LC-16, LC-18, LC-30)* — after T-27, T-21
  **Done when:** `save.noStepUpNoDialogForNonEmptySave`, `panel.staleReloadsRowAndReopens` and `panel.retryReusesKey` pass, and one unit case per problem code shows the stated handling.

- [ ] **T-29 · `RemoveFromProjectDialog` (S-52)** — `shared/ui` `Dialog` with focus trap and return: the statements of RF-6.1 (loses access from the next action; queued work will not run and running work is asked to stop at a bounded checkpoint; authored revisions, mappings, approvals, runs and evidence remain with provenance; tenant membership unchanged), in the second person on the actor's own row; "Remove from project" → the mutation with all four flags false; "Cancel" → back to the panel with its ticked state unchanged. *(plan §2.2 `RemoveFromProjectDialog`, D3, D10; RF-6.1, RF-6.3, RF-6.4, RF-11.2, LC-7, LC-17)* — after T-28
  **Done when:** `removeDialog.statementsConfirmAndCancel`, `removeDialog.cancelReturnsToPanelUnchanged` and `removeDialog.secondPersonOnOwnRow` pass, and focus returns to "Save project grants" on cancel.

- [ ] **T-30 · `AddProjectMemberPicker` (S-51) and Add mode** — `shared/ui` `Dialog` with a search field and the paged candidates (display name, tenant role) from `useGrantCandidates`; choosing one closes the picker and opens `GrantEditPanel` in Add mode with all boxes unticked; saving with nothing ticked shows "choose at least one role or canRun" inline without a request; saving with something ticked creates the grant and the new row appears at the next read; "Add project member" is absent while not editable. *(plan §2.2 `AddProjectMemberPicker`, D6, D10; RF-5.1, RF-5.2, RF-5.3, RF-5.5, RF-11.2, LC-1, LC-4, LC-19, LC-22)* — after T-28
  **Done when:** `picker.searchAndPaging`, `picker.choiceOpensAddPanelUnticked` and `picker.nothingTickedRefusedInline` pass, and the button is absent for a trashed-project fixture.

- [ ] **T-31 · Reflow and accessibility primitives** — Tailwind breakpoints: the table becomes one record per member at mobile and tablet widths; the panel, the picker and the dialog reflow; every checkbox grouped under its group label; every action 44 px with visible focus; outcomes conveyed by text; focus entry on heading or first control and return on close for panel, picker and dialog; live announcements never move focus. *(plan D10, D14; RF-11.1, RF-11.2, RNF-13, RNF-15, LC-28)* — after T-29, T-30
  **Done when:** Testing Library tests at three widths find every cell, statement and action, no interactive element below 44 px, and the two checkbox groups exposed with their group names.

## G · Verification suites

- [ ] **T-32 · Access suite** — `grantsPage.opensForOwnerAndAdminWithOrWithoutGrant`, `grantsPage.memberAndSuspendedActorDeniedScope`, `grantsPage.unreachableWhileTenantSuspendedOrDeleting`, `grantsPage.readOnlyWhileTrashedOrDeleting`, `grantsPage.editableWhileArchived`. *(plan §5.2 RF-1 rows, D4, D7; RF-1.2, RF-1.3, RF-1.4, RF-1.5, RF-8.5, LC-1, LC-2, LC-3, LC-12, LC-13, LC-15)* — after T-15, T-18
  **Done when:** all five tests pass with real PostgreSQL and plan 009 built or its recording stubs.

- [ ] **T-33 · Save suite** — `save.atomicSetVersionAuditOutbox`, `save.ownerAndAdminMaySaveIncludingSelf`, `save.unchangedWritesNothing`, `save.refusedForNonActiveTargetOrNonEditableProject`, `save.repeatedKeyReturnsRecordedOutcome`, `save.viewerBesideAuthorRecorded`, `save.allFalseRemovesGrantWithVersionAndAudit`, `save.selfRemovalDeniesOwnNextAction`, `feature.touchesOnlyGrantRowAndVersion`, `feature.noGrantInferredNoClientAuthority`. *(plan §5.2 RF-4, RF-6 rows, D1, D2, D3; RF-4.1, RF-4.2, RF-4.4, RF-4.5, RF-4.7, RF-4.9, RF-6.2, RF-6.4, RNF-1, RNF-2, RNF-3, RNF-10, LC-4, LC-6, LC-8, LC-10, LC-11, LC-17, LC-30)* — after T-17, T-13
  **Done when:** all ten tests pass with real PostgreSQL; the recording doubles for membership commands, sessions, notifications and runs report zero calls.

- [ ] **T-34 · Concurrency suite** — `save.staleVersionRefusedUnderLock` (two administrators on one grant), `save.actorDemotedMidEditRefusedAsStale`, `candidates.additionRacingColleagueRefusedStale`, `save.allOrNothingUnderFailure`, plus a save racing a suspension and a save racing a project trash. *(plan §5.1 Concurrency, D2, D7; RF-1.7, RF-4.6, RF-5.6, RF-10.4, RNF-4, LC-9, LC-16, LC-18)* — after T-33
  **Done when:** all six tests pass ten times in a row with real PostgreSQL.

- [ ] **T-35 · List and candidates suite** — `list.oneRowPerGrantOrderedByNamePaged`, `list.revokedMembershipNeverListed`, `list.suspendedRowReadOnly`, `list.searchByNameOrEmailAndStatusFilter`, `candidates.activeWithoutGrantActorIncludedSuspendedRevokedExcluded`, `save.fromPickerCreatesGrant`, `candidates.saveCreatesNoMembershipOrInvitation`, `removal.leavesMembershipAndReappearsInCandidates`. *(plan §5.2 RF-2, RF-5, RF-6.5 rows, D5, D6; RF-2.1, RF-2.3, RF-2.4, RF-2.5, RF-5.1, RF-5.2, RF-5.4, RF-5.7, RF-6.5, LC-1, LC-13, LC-20, LC-21, LC-22)* — after T-15, T-16, T-12
  **Done when:** all eight tests pass with real PostgreSQL under the runtime role.

- [ ] **T-36 · Effects-on-the-member suite** — `change.targetNextWriteRefusedStaleThenDeniedScope`, `change.authorityChangedEventDrivesStreamsAndWork` (recording doubles of plan 009's realtime and execution ports), `change.endsNoSession`, `change.leavesAuthoredEvidenceAndVerdicts`, `change.nextRequestDerivedUnderNewGrant`, `projectDeleted.grantsEndAndPageUnreachable` (through plan 009's `GrantCommands` from the project context double). *(plan §5.2 RF-9 rows, D11; RF-9.1, RF-9.2, RF-9.3, RF-9.4, RNF-7, RNF-8, LC-14, LC-23, LC-24)* — after T-33
  **Done when:** all six tests pass; the run-scoped authority double and the session double record zero calls from this feature.

- [ ] **T-37 · Security and fail-closed suite** — `grants.foreignTenantProjectDeniedSafely`, `grants.rlsBlocksCrossTenantRows` (list, candidates and save under the runtime role against another tenant's rows), `failClosed.everyBadFactRefuses` (every use case × every bad fact: Member actor, suspended actor, suspended target, revoked target, trashed project, deleting project, unknown project, foreign project, stale version, missing context). *(plan §5.1 Security, D4, D7; RF-8.4, RNF-5, RNF-6, LC-12, LC-27)* — after T-34
  **Done when:** all three tests pass; the fail-closed test has one case per (use case × bad fact) pair and none is skipped.

- [ ] **T-38 · Audit, redaction and startup suite** — `audit.entryPerChangeAndRefusal`, `audit.containsNoSecretOrContent`, `grantListPolicy.startupFailsOnAbsentOrUnboundedSize`. *(plan §5.2 RF-10, RNF-11 rows, D12, D13; RF-10.1, RF-10.3, RNF-11)* — after T-13, T-14, T-01
  **Done when:** all three tests pass; the audit test asserts one entry per change and per refusal of each reason class and none for a list read.

- [ ] **T-39 · FE e2e journeys** — Playwright journeys for US-1 … US-10 in EN, ES and FR at desktop, tablet and mobile widths against the BE doubles: `grantsPage.reachedFromSettingsTabAndReviewProjectGrants`, `grantsPage.ownerWithoutGrantSeesListAndCanAddSelf`, `signOut.guardSavesGrantUnderOriginalScope`, `switch.guardSavesGrantUnderOriginalProject`, `languageSwitch.changesDisplayOnlyTicksUnchanged`. *(plan §5.1 FE e2e; RF-1.1, RF-1.2, RF-3.7, RF-5.1, RF-11.3, RNF-14, LC-1, LC-4, LC-25, LC-26, LC-29; US-1 … US-10)* — after T-31, T-18
  **Done when:** every journey passes in the nine locale × width combinations, and each US-1 … US-10 story is named by at least one passing journey.

- [ ] **T-40 · Automated accessibility** — axe on the tab (editable, read-only, empty), the table (desktop and record layouts, suspended row, inert row), the panel (edit and add modes, with each notice), the picker and the removal dialog, in EN, ES and FR at three widths. *(plan §5.1 FE e2e + accessibility; RF-11.1, RF-11.2, RNF-13, RNF-15, LC-28)* — after T-39
  **Done when:** `a11y.grantsPage.{en,es,fr}.{desktop,tablet,mobile}` report zero critical or serious violations.

- [ ] **T-41 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist, versions from the release-support matrix: checkbox groups announced with their group names, the row action and the "/ you" marker read, "Grants saved" and "Removed from project" announced without focus moving, the picker and the dialog trap focus and return it, the read-only notice readable in order. *(plan §5.1 Manual assistive; RF-11.2, RNF-13)* — after T-40
  **Done when:** the checklist is complete for all four combinations with no open defect.

- [ ] **T-42 · Adversarial suite (US-11)** — a Member and a project-role holder calling the three endpoints; a save naming a membership of another tenant or a project of another tenant; a save for a suspended and for a revoked target; a `PUT` with a stale version after a colleague's change; a `PATCH`-style body with three flags; a list request probing for content or emails; a removal followed by a check that authored evidence, approvals and verdicts are untouched. *(spec US-11; RF-1.3, RF-4.6, RF-8.3, RF-8.4, RF-8.5, RF-9.4, RNF-9)* — after T-37, T-33
  **Done when:** every case is refused or has no effect with the expected safe code, no response differs between a foreign and an unknown project, and the evidence check finds nothing changed.

- [ ] **T-43 · Areas A and B evidence pack** — collect the passing runs of T-32 … T-42 with their database and browser versions into the release evidence folder. *(spec RNF-16; constitution P9)* — after T-42
  **Done when:** the pack lists each suite with run id, date and version, and spec §7 items 3–12 each reference at least one run.

## H · Closing

- [ ] **T-44 · Requirement coverage check** — script listing every RF-x.y, RNF-n, LC-n and US-n of the specification against the passing test names of plan §5.2 and the "Done when" lines of this file. *(spec §7 item 2)* — after T-43
  **Done when:** the script reports 120 of 120 identifiers (63 RF, 16 RNF, 30 LC, 11 US) covered by at least one passing test.

- [ ] **T-45 · Register-dependency review** — walk plan §6 — S-51 and S-52 as decided; S-47, S-49, A-53, A-55, A-57 and A-58 as decided by 009 with 011's notes; and the open S-04, S-06, S-13, S-02, S-17, A-11, A-35 and A-36: for each, confirm the built assumption matches the current Figma register state or record the deferral. *(spec §7 item 13)* — after T-44
  **Done when:** each of the sixteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-46 · Completion criteria sign-off** — tick spec §7 items 1–13 with the evidence (test run ids, a11y report, manual checklist, coverage report, evidence pack, register review). — after T-45
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-22, T-23, T-39 |
| RF-1.2 | T-15, T-22, T-32, T-39 |
| RF-1.3 | T-03, T-15, T-22, T-32, T-42 |
| RF-1.4 | T-32 |
| RF-1.5 | T-04, T-05, T-09, T-15, T-23, T-32 |
| RF-1.6 | T-07, T-14, T-18 |
| RF-1.7 | T-34 |
| RF-2.1 | T-01, T-02, T-09, T-12, T-15, T-24, T-35 |
| RF-2.2 | T-04, T-07, T-09, T-25 |
| RF-2.3 | T-07, T-09, T-25, T-35 |
| RF-2.4 | T-07, T-09, T-25, T-35 |
| RF-2.5 | T-04, T-09, T-12, T-15, T-24, T-35 |
| RF-2.6 | T-07, T-09, T-25 |
| RF-2.7 | T-09, T-21 |
| RF-2.8 | T-23 |
| RF-3.1 | T-07, T-25, T-26 |
| RF-3.2 | T-26 |
| RF-3.3 | T-26 |
| RF-3.4 | T-26 |
| RF-3.5 | T-27 |
| RF-3.6 | T-27 |
| RF-3.7 | T-27, T-39 |
| RF-3.8 | T-26 |
| RF-4.1 | T-06, T-08, T-17, T-33 |
| RF-4.2 | T-08, T-33 |
| RF-4.3 | T-08, T-28 |
| RF-4.4 | T-06, T-08, T-33 |
| RF-4.5 | T-04, T-05, T-06, T-08, T-17, T-28, T-33 |
| RF-4.6 | T-06, T-07, T-08, T-17, T-28, T-34 |
| RF-4.7 | T-08, T-17, T-21, T-28, T-33 |
| RF-4.8 | T-11 |
| RF-4.9 | T-08, T-11, T-33 |
| RF-5.1 | T-01, T-02, T-05, T-07, T-10, T-12, T-16, T-30, T-35, T-39 |
| RF-5.2 | T-07, T-10, T-30, T-35 |
| RF-5.3 | T-30 |
| RF-5.4 | T-06, T-08, T-17, T-35 |
| RF-5.5 | T-06, T-08, T-17, T-30 |
| RF-5.6 | T-08, T-28, T-34 |
| RF-5.7 | T-35 |
| RF-6.1 | T-29 |
| RF-6.2 | T-06, T-08, T-17, T-33 |
| RF-6.3 | T-29 |
| RF-6.4 | T-08, T-29, T-33 |
| RF-6.5 | T-35 |
| RF-7.1 | T-23 |
| RF-7.2 | T-23, T-26 |
| RF-7.3 | T-09, T-23 |
| RF-8.1 | T-11, T-18 |
| RF-8.2 | T-11 |
| RF-8.3 | T-17, T-18, T-42 |
| RF-8.4 | T-03, T-17, T-18, T-37, T-42 |
| RF-8.5 | T-22, T-32, T-42 |
| RF-9.1 | T-36 |
| RF-9.2 | T-36 |
| RF-9.3 | T-36 |
| RF-9.4 | T-36, T-42 |
| RF-10.1 | T-08, T-13, T-38 |
| RF-10.2 | T-03, T-14, T-18, T-19, T-28 |
| RF-10.3 | T-13, T-14, T-38 |
| RF-10.4 | T-08, T-34 |
| RF-11.1 | T-25, T-31, T-40 |
| RF-11.2 | T-29, T-30, T-31, T-40, T-41 |
| RF-11.3 | T-20, T-39 |
| RNF-1 | T-33 |
| RNF-2 | T-33 |
| RNF-3 | T-08, T-33 |
| RNF-4 | T-02, T-08, T-12, T-34 |
| RNF-5 | T-06, T-37 |
| RNF-6 | T-04, T-37 |
| RNF-7 | T-36 |
| RNF-8 | T-36 |
| RNF-9 | T-07, T-18, T-42 |
| RNF-10 | T-08, T-21, T-33 |
| RNF-11 | T-01, T-38 |
| RNF-12 | T-07, T-11 |
| RNF-13 | T-31, T-40, T-41 |
| RNF-14 | T-20, T-39 |
| RNF-15 | T-31, T-40 |
| RNF-16 | T-43 |
| LC-1 | T-09, T-10, T-30, T-32, T-35, T-39 |
| LC-2 | T-32 |
| LC-3 | T-09, T-32 |
| LC-4 | T-08, T-30, T-33, T-39 |
| LC-5 | T-25, T-26 |
| LC-6 | T-26, T-33 |
| LC-7 | T-06, T-29 |
| LC-8 | T-06, T-08, T-33 |
| LC-9 | T-06, T-08, T-28, T-34 |
| LC-10 | T-06, T-08, T-28, T-33 |
| LC-11 | T-06, T-08, T-28, T-33 |
| LC-12 | T-06, T-08, T-28, T-32, T-37 |
| LC-13 | T-09, T-32 |
| LC-14 | T-36 |
| LC-15 | T-32 |
| LC-16 | T-28, T-34 |
| LC-17 | T-08, T-29, T-33 |
| LC-18 | T-06, T-08, T-28, T-34 |
| LC-19 | T-06, T-08, T-30 |
| LC-20 | T-09, T-25, T-35 |
| LC-21 | T-09, T-10, T-35 |
| LC-22 | T-09, T-10, T-24, T-30, T-35 |
| LC-23 | T-36 |
| LC-24 | T-36 |
| LC-25 | T-27, T-39 |
| LC-26 | T-27, T-39 |
| LC-27 | T-37 |
| LC-28 | T-24, T-31, T-40 |
| LC-29 | T-39 |
| LC-30 | T-08, T-28, T-33 |
| US-1 … US-10 | T-39 (journeys) |
| US-11 | T-42 (adversarial suite) |

Self-check: all 63 RF criteria, 16 RNF and 30 limit cases of `specProjectGrants.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.

## Register assumptions built in

| Item | Built as (plan §6) | Tasks |
|---|---|---|
| S-51 "Add project member" picker (decided, Q1) | `AddProjectMemberPicker` as a `shared/ui` dialog over `GrantCandidatesQuery`, copy of D02 `420:2968` | T-10, T-16, T-30 |
| S-52 removal confirmation (decided, Q3) | `RemoveFromProjectDialog` on the N23 anatomy; the all-false save | T-29, T-17 |
| S-47 inert canRun statement (decided, 009) | `AuthorityStatements.inertCanRun` in the row and the panel; `inertCanRun` computed in the row | T-07, T-25, T-26 |
| S-49 Admin view (decided, 009) | The Admin view of R17 is the Owner view | T-22, T-32 |
| S-04 actions without destination | "Add project member" → the picker; "Edit … grants" is the drawn panel | T-30, T-26 |
| S-06 role variants | One component set; a Member meets the denied-scope state | T-22, T-23 |
| S-13 copy defects | "Current revisions" not rendered; "All statuses" = membership status | T-24 |
| S-02 responsive coverage | R17, the panel, the picker and the dialog follow the D01 responsive policy | T-31, T-40 |
| S-17 Help & shortcuts | Opens N43 HelpShortcutsPanel (`430:4129`) | T-23 |
| A-53 grant combinations (decided, 009) | Any subset accepted; inert flag computed, never stored | T-07, T-26 |
| A-55 permission version (decided, 009; note from 011) | The target's version is the expected-version guard under lock | T-06, T-08, T-34 |
| A-57 administration authority (decided, 009; note from 011) | `TENANT_MEMBERS` + `AdministrationRule` with self allowed | T-08, T-22, T-32 |
| A-58 lifecycle grant (decided, 009) | Not consulted; the page performs no lifecycle | T-11 |
| A-11 configuration values | `GrantListPolicyProperties` with the two page sizes | T-01, T-38 |
| A-35 persistence tooling | The index and the adapters against the placeholder access API and migration tool | T-02, T-12 |
| A-36 identity-scoped membership listing | Not used; tenant-scoped reads under RLS; names through `IdentityNamesPort` | T-04, T-12 |
