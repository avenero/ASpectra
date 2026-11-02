# Plan 011 — Project grants editor

**Feature** The Project grants page of a project: listing who holds which grants, editing one member's Author / Reviewer / Viewer and `canRun`, adding a tenant member to the project and removing one from it · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/011-ProjectGrants/specProjectGrants.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-13 as the subject; F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-14, F-IDN-16, F-IDN-18, F-PRJ-02, F-PRJ-07, F-SET-04 and F-APR-04 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG; the `access` context owns memberships and grants), FE board `46:7847` (feature-modular SPA DAG; `shared/security`, `shared/ui`, `shared/errors`), board 1 §04 `23:5587` (ACCESS table) and the decision register `122:2` §3 ("new-project creator grants are explicit records") · **Builds on** plan 001 (Login), plan 002 (Signup and verification), plan 003 (Enrollment), plan 006 (Sessions, refresh and expiry), plan 007 (Logout and session management), plan 008 (Invitation acceptance) and plan 009 (Roles, grants and the authority matrix); elements they own are tagged **[001]** … **[009]** and are extended, never duplicated. Specification 010 (members page) has no plan yet; its elements are cited by specification as **[010]**.

This plan is the HOW. It adds no behaviour the specification does not state. Plan 009 built the `access` context — the grant record, the authority matrix, the administration rule, the permission version, revocation and its side effects — and published `GrantCommands.SetProjectGrants` and the declaration of `ProjectGrantsQuery` for this feature; this plan builds the surface over them and adds nothing to the grant model. The specification's seven decisions (Q1–Q7) and five derived rules (D-a…D-e) are built as decided; the register items it recorded (S-51, S-52) are therefore implemented, not assumed. Where the specification depends on items still open (S-04, S-06, S-13, S-02, S-17, A-11, A-35, A-36) or on earlier decisions (A-53, A-55, A-57, A-58, S-47, S-49), the plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The page writes grant records only; no use case of this feature reads, stores or replays a mapping, approval, launch command or intent; what a grant permits is decided by the matrix **[009]** at the moment of each later action. | RF-8.2, RNF-1 |
| **P3** The browser expresses intent, never authority | The client submits four flags, the version it saw and an idempotency key for one membership on one project, and renders the server's answer; whether the actor may open the page or save, whether the target is active, whether the project is editable and whether the state is stale are all decided on the server at the moment of the action. `useAuthority` **[009]** only hides or disables. | RF-1.2, RF-1.3, RF-4.2, RF-4.5, RF-4.6, RNF-2 |
| **P4** Only the platform catalog is global | Grants and memberships are tenant rows; display names are read through an identity-scoped port and never copied into tenant rows. | RF-2.2, RNF-6 |
| **P6** Fail closed | A non-active target, a non-editable project, a stale version, an actor who is not Owner or Admin, a missing context or an unknown project refuses the change; an empty set on a membership with no grant writes nothing. | RF-1.3, RF-4.5, RF-4.6, RF-5.5, RNF-5 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security and C10 filters **[006/009]**, the outbox and idempotency outcome. FE: React 18, TypeScript, Vite, TanStack Query, Zustand (the stamped draft of the open panel only), i18next, Tailwind, axios. Nothing is added. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: one rule, two queries and one controller in the existing hexagonal `access` context; `GrantCommands` **[009]** gains one parameter; `project` and `identity` are reached through two new published ports; outer adapters depend inward. FE: the page in the feature module `project-settings`; cross-cutting behaviour (authority store, statements, denied-scope handling, drafts registry) in `shared/*` **[006/009]**; TanStack Query owns the list and the candidates; Zustand holds only the dirty panel. | §2 |
| **P12** Two independent tenancy gates | Every read and write runs under C10 **[009]** with `SET LOCAL app.tenant_id` inside the transaction and `FORCE RLS` on `tenant_membership` and `project_grant` **[009]**; the runtime authorisation of `AuthorityChecker` and `AdministrationRule` applies on top; the project is resolved in the actor's tenant only. | RF-1.6, RF-8.4, RNF-6 |
| **P13** Configured values are never defaulted | The list and picker page sizes are one validated `GrantListPolicyProperties` record; startup fails on an absent or unbounded value; the client holds no number. | RF-2.1, RF-5.1, RNF-11 |
| **P14** Release bar | WCAG 2.2 AA on the page, the panel, the picker and the dialog with automated and manual assistive evidence; EN / ES / FR for every state. | RF-11, RNF-13–15 |
| **F-X-01** C10 context | The page's endpoints run behind the C10 request filter **[009]**; `AuthorityChecker.require(C10, TENANT_MEMBERS)` gates every read and write at its effect boundary; the target's membership is re-read under lock inside the write transaction. | RF-1.2, RF-1.3, RF-4.2, RNF-2 |
| **F-X-02** Request integrity | The save is idempotent on `(tenantId, projectId, operation, idempotencyKey, bodyHash)` **[009 D15]**; a repeated submission returns the recorded outcome; an unchanged save writes nothing; no mutation auto-retries. | RF-4.4, RF-4.7, RNF-10 |
| **F-X-03** Secrets and safe problems | Responses carry names, roles, flags, statuses and a version; never content, a credential or another tenant's row; six safe problem codes with a correlation id; browser storage holds nothing from this feature. | RF-1.6, RF-10.2, RF-10.3, RNF-9 |
| **F-X-04** Accessibility | Grouped checkboxes, text outcomes, focus entry and return on panel, picker and dialog, 44 px targets, live region without focus theft. | RF-11.2 |
| **F-X-05** Audit log | `GRANTS_SET` / `GRANT_REMOVED` **[009]** for every change and the new `GRANT_CHANGE_REFUSED{reasonClass}` for every refusal, appended through `AuditPort` in the same transaction. | RF-10.1 |
| **F-X-06** Configuration validation and email port | `GrantListPolicyProperties` is part of startup validation; the email port is not used (decision Q5). | RF-4.8, RNF-11, RNF-12 |
| **F-X-07** Responsive surfaces | R17, the panel, the picker and the dialog follow the D01 responsive policy (`414:2645`): one record per member, dialogs as sheets, statements shared across widths. | RF-11.1, RNF-15 |
| **F-IDN-11** boundary | No second grant model: the record, the valid combinations, inert `canRun`, the version advance, the audit record and the side effects of a change are plan 009's; this plan calls `SetProjectGrants` and reads `ProjectGrantsQuery`. | RF-4.1, RF-9, RNF-1 |
| **F-IDN-12** boundary | Nothing here creates, suspends, reinstates or revokes a membership or changes a tenant role; the members page **[010]** only routes here. | RF-5.7, RF-6.5, RF-8.1 |
| **F-IDN-18** boundary | The open panel registers a stamped draft so the dirty guard of the guarded switch and of sign-out **[007]** sees it; nothing of the guard is redefined. | RF-3.7 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access`, plus two published ports consumed from `project` and `identity`

The context, its four layers (`api / application / domain / infrastructure`), the grant aggregate, the matrix, the administration rule, the version guard, `GrantCommands` and the audit and problem adapters exist from plan 009. This plan adds one pure rule, one state enumeration, two read models, one query (and builds the one plan 009 declared), one configuration record, one controller, one audit event and four problem codes, and extends `SetProjectGrants` with the expected-version guard and the project-state check.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `GrantSet` **[009]** | Reused unchanged: the subset of Author / Reviewer / Viewer plus `canRun`; `isEmpty()`, `eligibleToRun()`. The inert flag shown by the page is `canRun ∧ ¬(Author ∨ Reviewer)`, computed, never stored (**[009 D5]**). | RF-3.3, RF-3.4, RF-2.6 |
| | `ProjectState` | Closed enumeration `ACTIVE \| ARCHIVED \| TRASHED \| DELETING` with `isEditable()` = `ACTIVE ∨ ARCHIVED`, as supplied by `ProjectStatePort`. | RF-1.5, RF-4.5, RF-5.1 |
| | `GrantEditRule` | Pure rule `(targetStatus, projectState, currentGrant?, requestedSet, expectedVersion, currentVersion) → Apply(SET \| REMOVE) \| Refuse(reasonClass)` evaluated after `AdministrationRule` **[009]**: `TARGET_NOT_ACTIVE` when the membership is not `ACTIVE`; `PROJECT_NOT_EDITABLE` when `¬isEditable()`; `STALE_STATE` when `expectedVersion ≠ currentVersion`; `NOTHING_CHOSEN` when the requested set is empty and no grant exists; `UNCHANGED` when the requested set equals the current grant; otherwise `Apply(REMOVE)` for an empty set on an existing grant and `Apply(SET)` for everything else. No default branch. | RF-4.1, RF-4.4, RF-4.5, RF-4.6, RF-5.4, RF-5.5, RF-6.2, LC-7–LC-12, LC-18, LC-19 |
| | `ProjectGrantRow` | Read model of one list row: `membershipId`, `displayName`, `tenantRole`, `membershipStatus` (`ACTIVE \| SUSPENDED`), `GrantSet`, `inertCanRun`, `isActor`, `permissionVersion` (the version the panel will carry), `editable` (= status `ACTIVE` ∧ project editable). Never a verified email on the wire (it is a search key only), never a revoked membership, never content. | RF-2.1–2.4, RF-2.6, RF-3.1, RF-4.6, RNF-9 |
| | `GrantCandidateRow` | Read model of one picker row: `membershipId`, `displayName`, `tenantRole`. | RF-5.1, RF-5.2 |
| | `GrantListPolicy` | Immutable record: list page size, picker page size. | RF-2.1, RF-5.1, RNF-11 |
| `access.application` | `GrantCommands.SetProjectGrants` **[009]** (extended) | Input gains `expectedVersion`. Sequence, one transaction: derive the actor's C10 and `AuthorityChecker.require(C10, TENANT_MEMBERS)`; return the recorded outcome for a repeated idempotency key; `AdministrationRule` **[009]** (Owner or Admin; self allowed); resolve the project in the actor's tenant through `ProjectExistsPort` **[009]** and read its state through `ProjectStatePort`; lock the target membership `FOR UPDATE`; `GrantEditRule`; on `Refuse` → audit `GRANT_CHANGE_REFUSED{reasonClass}` and answer the matching problem, nothing written; on `UNCHANGED` → answer the current row, nothing written; on `Apply` → `TenantMembership.setGrants` **[009]** (empty → delete), version advance, `GRANTS_SET` / `GRANT_REMOVED`, `AuthorityChanged` outbox event **[009]**, idempotency outcome. The bootstrap of plan 012 passes `expectedVersion = none`, which skips the stale check only. | RF-4.1–4.7, RF-4.9, RF-5.4–5.6, RF-6.2, RF-6.4, RF-10.1, RF-10.4, RNF-3, RNF-4, RNF-10, LC-4, LC-8–LC-12, LC-17–LC-19, LC-30 |
| | `ProjectGrantsQuery` **[009 declared]** | For the actor (gated by `TENANT_MEMBERS`): `project_grant` rows of the project joined to `tenant_membership` (status `ACTIVE` or `SUSPENDED` only), display names and verified emails through `IdentityNamesPort`; filter by status; search by display name or verified email; order by display name; page by the list page size; the envelope carries `projectState` and `editable`; a page beyond the last returns the last page. | RF-1.5, RF-2.1–2.7, RF-7.3, LC-1, LC-3, LC-13, LC-20–LC-22 |
| | `GrantCandidatesQuery` | For the actor (gated by `TENANT_MEMBERS`, project editable): `tenant_membership` rows of the tenant with status `ACTIVE` and no `project_grant` on the project (anti-join), the actor's included; display names through `IdentityNamesPort`; search by display name or verified email; order by display name; page by the picker page size. | RF-5.1, RF-5.2, LC-1, LC-21, LC-22 |
| | `AuthorityChangedHandler` **[009]** | Unchanged: the side effects of a grant change on the target (streams closed, queued tickets refused, running attempts asked to stop) are plan 009's; nothing is built here. | RF-9.1–9.4 |
| | Ports | New: `ProjectStatePort` (published by `project`: the `ProjectState` of a project of the tenant, or absent), `IdentityNamesPort` (published by `identity`: display name and verified email for a set of identity ids, nothing else). Reused: `AuthorityPort`, `MembershipRepository`, `ProjectGrantRepository`, `ProjectExistsPort`, `AuditPort`, outbox, idempotency outcome **[009]**; `Clock` **[001]**. Not used: `SecurityNotificationPort`, `TransactionalEmailPort` (decision Q5). No port reaches an approval, run, intent or content table. | all |
| `access.infrastructure` | Persistence **[009]** | `project_grant` gains the index `(tenant_id, project_id)` for the list; the anti-join of the candidates uses the existing `(tenant_id, membership_id, project_id)` key; the share and exclusive locks of `PermissionVersionGuard` **[009]** are reused. Access API and migration tool per A-35 (placeholders). | RF-2.1, RF-5.1, RNF-4 |
| | `GrantListPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding into `GrantListPolicy`: both sizes present, positive, bounded above by a configured ceiling; any violation fails startup. | RNF-11 |
| | Audit adapter **[009]** | New event `GRANT_CHANGE_REFUSED{reasonClass, actionClass}` with acting and target membership and identity, tenant, project, correlation id and time; `GRANTS_SET` / `GRANT_REMOVED` unchanged. Never a secret or content. | RF-10.1, RF-10.3 |
| | Problem codes | New: `access.grants.stale` (409), `access.grants.target-not-active` (409), `access.grants.project-not-editable` (409), `access.grants.nothing-chosen` (400). Reused: `access.denied-scope` (403) **[006/009]** for a Member, a suspended actor, a foreign or unknown project or a missing context; `access.forbidden-change` (403) **[009]** when `AdministrationRule` refuses. Each carries code, title, correlation id and, for the 400, the field path; never a reason beyond the code. | RF-1.3, RF-8.4, RF-10.2 |
| `access.api` | `ProjectGrantsController` — `/api/tenants/{tenantId}/projects/{projectId}/grants` (C10 filter; CSRF on the write) | `GET ?page&q&status` → 200 `{items: ProjectGrantRow[], page, pageCount, total, projectState, editable}` · `GET candidates?page&q` → 200 `{items: GrantCandidateRow[], page, pageCount, total}` or 409 `access.grants.project-not-editable` · `PUT {membershipId}` body `{author, reviewer, viewer, canRun, expectedVersion, idempotencyKey}` — all four flags mandatory, no partial update — → 200 the updated `ProjectGrantRow`, 204 when the grant was removed, 400 `access.grants.nothing-chosen` or a field-path problem, 403 `access.denied-scope` / `access.forbidden-change`, 409 `access.grants.stale` / `target-not-active` / `project-not-editable`. The route's tenant is the actor's current tenant; a `tenantId` the actor is not a member of answers 403 `access.denied-scope` from the C10 filter. All problems are RFC 9457. | RF-1.2, RF-1.3, RF-1.5, RF-2.1, RF-4, RF-5, RF-6.2, RF-8.4, RF-10.2 |

**Cross-context touch points.** `project` — `ProjectExistsPort` **[009]** and `ProjectStatePort` (read-only); project deletion ends grants through `GrantCommands` in its own transaction **[009 LC-30]**. `identity` — `IdentityNamesPort` (read-only, identity-scoped). `audit` — append-only. `execution`, `realtime` — reached only by plan 009's `AuthorityChangedHandler`. Nothing reads or writes approval, run, intent or content tables.

**Data model note (P12).** No new table. The index is on a tenant row under `FORCE RLS`. Display names are never stored on tenant rows; they are joined at read time through the identity port.

### 2.2 Front end — feature module `src/features/project-settings` plus `shared/*` [006/009]

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/t/:tenant/p/:project/settings/members` (R17, the "Project members" tab of project settings, beside Integrations, Environments and Configuration); reached from the project settings tabs and from "Review project grants" on the members page **[010]** with the current project. The route guard hides the tab for a viewer whose `AuthorityStore` **[009]** has no `TENANT_MEMBERS`; the server answers `access.denied-scope` regardless. | RF-1.1, RF-1.2, RF-1.3 |
| | `useProjectGrants(page, q, status)`, `useGrantCandidates(page, q)`, `useSetProjectGrants()` | Typed hooks over `contracts/access-grants`; queries with no interval and no focus refetch, invalidated after every mutation; the mutation never retries, carries the idempotency key and `expectedVersion`, and returns server state verbatim. | RF-2.7, RF-4, RF-5, RNF-10 |
| | `contracts/access-grants` (in `src/contracts`) | Request / response / problem types and runtime guards for the three endpoints, the `ProjectState` enum and the six problem codes; the grant flags reuse the `GrantSet` type of `contracts/access` **[009]**. | all |
| Views | `ProjectMembersTab` | R17: breadcrumb, heading "Project grants", the sub-statement, "Add project member" (rendered only while `editable`), the tabs, `SearchAndStatusFilter`, `GrantsTable`, record count, `Pagination`, the open `GrantEditPanel`, `AuthorityStatements` **[009]** variant `tenantAdminScope` ("Tenant administration is a different scope …"), `ReadOnlyNotice` when `projectState` is `TRASHED` or `DELETING`, and "Decision history ↗" routing to the audit trail **[F-APR-04]** with the project's grant-event scope. | RF-1.1, RF-1.5, RF-2.1, RF-2.8, RF-7.1, RF-7.3 |
| | `SearchAndStatusFilter` | "Find a project member" (⌘K) and "All statuses" (all / active / suspended); no "Current revisions" control is rendered. | RF-2.5 |
| | `GrantsTable` | Columns Member ("/ you" on `isActor`), Project grants (roles joined by "+"), canRun ("Granted" / "Not granted", with the inert note from `AuthorityStatements` variant `inertCanRun` **[009]** when `inertCanRun`), Tenant role; a `SUSPENDED` row shows the state, its grants read-only and the reinstatement statement, and cannot open the panel; an editable row's activation opens `GrantEditPanel` through the single-panel guard. Mobile and tablet: one record per member with the same cells. | RF-2.2–2.4, RF-2.6, RF-3.1, RF-11.1, LC-5, LC-20 |
| | `Pagination` **[007]** | "Previous · Next" from `page` / `pageCount`, unavailable state in text, record count. | RF-2.1, LC-28 |
| | `GrantEditPanel` | Title "Edit <name>'s grants" or "Add <name> to this project"; `CheckboxGroup` "Project roles" (Author, Reviewer, Viewer) and "Independent execution permission" (canRun), independent; the two drawn statements; the inert note while `canRun ∧ ¬(Author ∨ Reviewer)`; the removal notice while nothing is ticked on an existing grant; the tenant role beside the title; "Cancel changes" (purges the draft, closes) and "Save project grants". While dirty it holds a stamped draft `{identityId, tenantId, projectId, kind: 'grants', membershipId}` in the Zustand drafts registry **[006 D10]** so the dirty guards see it (D-b); at most one panel is open, and opening another row passes through the same guard. On save: an existing grant with nothing ticked → `RemoveFromProjectDialog`; a candidate with nothing ticked → inline "choose at least one role or canRun" without a request; otherwise `useSetProjectGrants` with the row's `permissionVersion` as `expectedVersion` and the key generated once per open. | RF-3.1–3.8, RF-4.3, RF-5.3, RF-5.5, RF-6.1, RF-6.3, LC-5–LC-8, LC-19, LC-25, LC-26 |
| | `RemoveFromProjectDialog` (S-52) | `shared/ui` `Dialog` with focus trap and return: the statements of RF-6.1, in the second person on the actor's own row; "Remove from project" → the mutation with all four flags false; "Cancel" → back to the panel unchanged. | RF-6.1, RF-6.3, RF-6.4, LC-7, LC-17 |
| | `AddProjectMemberPicker` (S-51) | `shared/ui` `Dialog` with a search field and the paged candidates (display name, tenant role); choosing one closes the picker and opens `GrantEditPanel` in "Add" mode with all boxes unticked. | RF-5.1–5.3, LC-1, LC-4, LC-22 |
| | Outcome handling | 200 → row updated, panel closed, "Grants saved" in the live region; 204 → row removed at the next read, "Removed from project"; `access.grants.stale` → "The grants changed since you opened this panel", row refetched, panel reopened on the current values with the new `expectedVersion`; `target-not-active` / `project-not-editable` → stated reason, row refetched, panel closed; `nothing-chosen` → inline message; `access.forbidden-change` / `access.denied-scope` → `DeniedScopeHandler` **[006/009]**; network failure → `TransportFailureStatus` **[006]** on the panel, action repeatable with the same key. | RF-4.6, RF-4.5, RF-4.7, RF-5.6, RF-10.2, LC-9–LC-12, LC-16, LC-18, LC-30 |
| State | TanStack Query | Keys per tenant, project, page, query and status; cleared with the scope epoch **[001/006]**; no interval. | RF-2.7 |
| | Zustand drafts registry **[006]** | Only the dirty panel; purged on Cancel, on a confirmed save, on epoch change and by the guards' Discard. | RF-3.7, RNF-9 |
| Shared used | `AuthorityStore` and `useAuthority(TENANT_MEMBERS)` **[009]** (hide the tab and the actions for a Member), `AuthorityStatements` **[009]** (variants `grants`, `inertCanRun`, `tenantAdminScope` added here with the R17 card text), `DeniedScopeHandler` and `TransportFailureStatus` **[006]**, `shared/i18n` (bundle keys for every statement, label, notice and problem), `shared/ui` (`Dialog`, `CheckboxGroup`, `LiveRegion`, `Table` / `RecordList`, 44 px targets). | RF-7.1–7.3, RF-11.2, RF-11.3 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | Endpoints under `/projects/{id}/grants` | `routes`; `ProjectMembersTab` heading and sub-statement; "Review project grants" **[010]** routes here |
| RF-1.2 | `AuthorityChecker.require(TENANT_MEMBERS)` ignores the actor's own grant | Tab shown for Owner and Admin |
| RF-1.3 | C10 derivation **[009]** → `access.denied-scope` for Member, suspended or no membership | `DeniedScopeHandler` **[006]**; tab hidden by `useAuthority` |
| RF-1.4 | Tenant status in derivation **[009 RF-5.2]** → denied | Suspended-tenant screen **[014]** replaces the shell |
| RF-1.5 | `projectState` and `editable` in the list envelope; candidates and writes refused while not editable | `ReadOnlyNotice`; no "Add project member"; rows not activatable |
| RF-1.6 | `ProjectGrantRow` carries names, roles, flags, status, version only | Table renders those cells only |
| RF-1.7 | Next derivation denies; a save in flight meets the actor's stale version at `AuthorityChecker` **[009 D4]** | `DeniedScopeHandler` |
| RF-2.1 | `ProjectGrantsQuery` ordered by display name, paged by `GrantListPolicy` | `GrantsTable`, record count, `Pagination` |
| RF-2.2 | `ProjectGrantRow.isActor`, roles, `canRun`, `tenantRole` | Cell rendering ("/ you", "+", Granted / Not granted) |
| RF-2.3 | Join restricted to `ACTIVE` and `SUSPENDED` memberships with a grant | — |
| RF-2.4 | `membershipStatus = SUSPENDED`, `editable = false` | Suspended row read-only with its statement |
| RF-2.5 | `q` matches display name or verified email; `status` filter | `SearchAndStatusFilter`; no "Current revisions" |
| RF-2.6 | `inertCanRun` computed in the row | `AuthorityStatements.inertCanRun` beside "Granted" |
| RF-2.7 | — | Query on open and invalidation after each mutation; no interval |
| RF-2.8 | Audit trail scope **[F-APR-04]** | "Decision history ↗" route |
| RF-3.1 | `ProjectGrantRow` supplies the current flags and version | `GrantEditPanel` opened from an editable row |
| RF-3.2 | — | Panel statements from the i18n bundle |
| RF-3.3 | `GrantSet` accepts any subset **[009 D5]** | Independent checkboxes |
| RF-3.4 | `SetProjectGrants` accepts canRun without a role **[009]** | Inert note while ticked so |
| RF-3.5 | — | Removal notice; save routes to the dialog |
| RF-3.6 | — | "Cancel changes" purges the draft |
| RF-3.7 | — | Stamped draft in the registry; single-panel guard |
| RF-3.8 | Body carries four flags only; no other field accepted | Tenant role shown beside the title; no other control |
| RF-4.1 | `SetProjectGrants` → `setGrants`, version, `GRANTS_SET`, outbox, outcome — one transaction | Row updated; "Grants saved" |
| RF-4.2 | `AuthorityChecker` + `AdministrationRule` (self allowed) at save time | Own row editable like any other |
| RF-4.3 | No step-up port consulted | No dialog for a non-empty save |
| RF-4.4 | `GrantEditRule.UNCHANGED` → nothing written, current row answered | "No changes to save" |
| RF-4.5 | `TARGET_NOT_ACTIVE` / `PROJECT_NOT_EDITABLE` → 409 | Stated reason; row refetched |
| RF-4.6 | `STALE_STATE` under the target's `FOR UPDATE` lock → 409 `access.grants.stale` | Refetch row, reopen panel with the new `expectedVersion` |
| RF-4.7 | Idempotency outcome **[009 D15]** | Same key on retry |
| RF-4.8 | No notification port used | — |
| RF-4.9 | Command writes `project_grant` and the membership version only | — |
| RF-5.1 | `GrantCandidatesQuery` (active, no grant, anti-join, search, paged); refused when not editable | `AddProjectMemberPicker` |
| RF-5.2 | Anti-join over `ACTIVE` memberships of the tenant; actor not excluded | — |
| RF-5.3 | — | Panel in "Add" mode, all unticked |
| RF-5.4 | `Apply(SET)` creates the row | New row at the next read |
| RF-5.5 | `NOTHING_CHOSEN` → 400 (server-side too) | Inline message; no request |
| RF-5.6 | Rule re-evaluated under lock: grant now exists → `STALE_STATE`; status / state refusals | Picker reloaded |
| RF-5.7 | Command touches no membership, invitation or role | No such control |
| RF-6.1 | — | `RemoveFromProjectDialog` statements |
| RF-6.2 | `Apply(REMOVE)` → delete, version, `GRANT_REMOVED`, outbox; same refusals | 204 → row leaves at the next read |
| RF-6.3 | — | Cancel returns to the panel |
| RF-6.4 | Self allowed by `AdministrationRule`; next action denied by derivation | Second-person copy on the actor's row |
| RF-6.5 | Membership untouched; a new grant is a new row | Removed member reappears in the picker |
| RF-7.1 | — | `AuthorityStatements.tenantAdminScope` |
| RF-7.2 | Same envelope for Owner and Admin | Same components on every variant and width |
| RF-7.3 | Problem codes | `ReadOnlyNotice`, suspended-row statement, `DeniedScopeHandler` |
| RF-8.1 | Only `SetProjectGrants` is called; no membership, role, policy, invitation, session or identity port | No such control |
| RF-8.2 | No approval, launch or content port | — |
| RF-8.3 | Body = four flags for one membership on the route's project | Four checkboxes |
| RF-8.4 | C10 filter and `ProjectExistsPort` in the actor's tenant; route-only project id | — |
| RF-8.5 | `AuthorityChecker(TENANT_MEMBERS)` denies Member whatever the grant | Tab hidden |
| RF-9.1 | Version advance **[009]**; next derivation under the new grant | — |
| RF-9.2 | `AuthorityChanged` → `AuthorityChangedHandler` **[009]** | — |
| RF-9.3 | No session port | — |
| RF-9.4 | Nothing but the grant row changes | — |
| RF-10.1 | `GRANTS_SET`, `GRANT_REMOVED`, `GRANT_CHANGE_REFUSED{reasonClass}` | Correlation id header |
| RF-10.2 | Six safe codes | Mapped per code |
| RF-10.3 | Payload shapes; no content port | Nothing logged client-side |
| RF-10.4 | One transaction; outcome recorded with the change | — |
| RF-11.1 | — | Record layout; dialogs reflow |
| RF-11.2 | — | `shared/ui` groups, focus, 44 px, `LiveRegion` |
| RF-11.3 | Role names as enum values | i18n bundles |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The page is a thin surface over `GrantCommands.SetProjectGrants` [009]; no second grant service exists.**
*Why.* Plan 009 owns the grant record, the administration rule, the version advance, the audit record and the side effects of a change; RF-4.1 and RF-9 require exactly those, and RNF-1 forbids any effect the derivation does not deliver. Calling the published command keeps one write path for this page, the bootstrap of plan 012 and project deletion.
*Discarded.* A grants service inside the page's feature — a second write path to drift. Writing `project_grant` rows from a controller — bypasses the lock, the version and the audit.
*Covers.* RF-4.1, RF-4.9, RF-5.4, RF-6.2, RF-9.1–9.4, RNF-1, RNF-3, RNF-7, RNF-8.

**D2 · The target membership's `permissionVersion` is the expected-version guard of a save, compared under the target's exclusive lock.**
*Why.* Decision Q4 (A-55 note): a save over a state that changed since the panel opened must be refused. The membership version already advances on every grant, role or status change of that membership (**[009 D4]**), so it is exactly the stamp the panel needs; comparing it under the `FOR UPDATE` lock the command already takes makes the check atomic with the write (LC-9). A project state change is checked beside it through `ProjectStatePort` (D7), since it is not a membership fact.
*Discarded.* A per-row ETag of the grant — a second version for the same row. Last write wins — Q4's rejected option. Comparing the four flags instead of a version — misses a status change and cannot tell "unchanged" from "changed back".
*Covers.* RF-4.6, RF-5.6, RF-6.2, RNF-4, LC-9–LC-12, LC-18.

**D3 · A save is a `PUT` carrying all four flags; removal is the explicit all-false body, confirmed on the client; an empty body on a membership without a grant is a 400.**
*Why.* Decision Q3 puts the confirmation on the surface; the server's contract must still make an accidental removal impossible: with four mandatory flags nothing can be removed by omission, and the dialog is the only client path that sends all-false for an existing grant. `GrantEditRule` names the empty-on-nothing case (`NOTHING_CHOSEN`) so the picker's rule (RF-5.5) is enforced server-side too.
*Discarded.* A `DELETE` endpoint for removal — a second operation with its own idempotency and audit for what is one grant edit. A `PATCH` with optional flags — an omitted flag could clear a grant. A server-side `confirmed` flag — a lie the client would always send.
*Covers.* RF-3.5, RF-4.1, RF-5.5, RF-6.1–6.3, RF-8.3, LC-7, LC-19.

**D4 · Page and write authority is the `TENANT_MEMBERS` action class of the matrix [009]; the actor's own grant on the project is irrelevant.**
*Why.* Decision Q2: an Owner or Admin administers grants on any project of the tenant, with or without a grant on it, and a Member never does. That is the tenant-administration row of the matrix, which plan 009 already decides under the class `TENANT_MEMBERS` and which `AdministrationRule` refines for self-changes. Reusing the class means the page cannot be opened by anyone the members page would refuse.
*Discarded.* A new action class `PROJECT_GRANTS` — the same row under another name. Requiring a grant on the project to administer it — contradicts Q2 and the solo-Owner story.
*Covers.* RF-1.2, RF-1.3, RF-4.2, RF-8.5, RNF-2, LC-1–LC-3.

**D5 · The list is a server-paged read model that joins display names through an identity-scoped port and never copies them.**
*Why.* RF-2.1 and RF-2.5 need ordering, search and paging over names and emails; names live on the global identity (FR-IDN-001) and must not be duplicated into tenant rows (P4). `IdentityNamesPort` answers for a set of identity ids with name and verified email only, so the join is explicit, narrow and auditable; paging on the server with a configured size keeps the client free of numbers (P13).
*Discarded.* Caching display names on `tenant_membership` — stale after a rename and a copy of identity data in a tenant row. Client-side search over a full list — a full list in memory and a client rule. A SQL join across contexts — a foreign table read.
*Covers.* RF-2.1–2.5, RF-1.6, RNF-6, RNF-9, RNF-11, LC-15, LC-20–LC-22, LC-28.

**D6 · Candidates are the anti-join of active memberships against the project's grants, actor included.**
*Why.* Decision Q1 and Q6: the picker shows exactly the memberships that could receive a grant — active, in the tenant, holding none — and the solo Owner must be able to pick themselves. An anti-join computed on the server cannot show a suspended or revoked membership or a member already on the project, and a race with a colleague's addition is caught by D2 at save time.
*Discarded.* Listing every tenant member with empty rows — Q6's rejected option. Excluding the actor — contradicts the solo-Owner story.
*Covers.* RF-5.1, RF-5.2, RF-5.6, LC-1, LC-4, LC-18, LC-21, LC-22.

**D7 · The project's lifecycle state is read through `ProjectStatePort` at read time and again at write time; `ACTIVE` and `ARCHIVED` are editable.**
*Why.* Decision Q7; the project state belongs to the `project` context and can change between a read and a write (LC-12). Reading it in the list envelope lets the page render read-only honestly; re-reading it inside the write transaction keeps P6 — a trashed project refuses the save whatever the page showed.
*Discarded.* Copying the state onto grant rows — stale. Checking only on the client — bypassable. Treating archived as non-editable — contradicts Q7 and the auditor case.
*Covers.* RF-1.5, RF-4.5, RF-5.1, RNF-5, LC-12–LC-14.

**D8 · The open panel registers a stamped draft in the Zustand drafts registry [006 D10], so the existing dirty guards see it.**
*Why.* Spec D-b: unsaved panel changes are unsaved work for the switch guard (F-IDN-18) and the sign-out guard (**[007 D9]**). Both guards already read the stamped registry; registering the panel's draft with its tenant and project stamp gives the guard, the "Save" branch (which calls the panel's own save under its stamped scope) and the epoch purge for free.
*Discarded.* A page-level "unsaved changes" prompt — a second guard with weaker options. No guard — a sign-out would silently discard a grant edit.
*Covers.* RF-3.6, RF-3.7, LC-25, LC-26.

**D9 · A stale or refused save refetches the row and reopens the panel on the current values; nothing merges automatically.**
*Why.* Decision Q4 says the actor re-applies; the current values and the new `expectedVersion` come from the refetched row, so the second attempt is made against what is true now. A refusal for a non-active target or a non-editable project closes the panel because there is nothing to re-apply.
*Discarded.* Merging the actor's ticks over the new state — hides the colleague's change. Retrying with the new version automatically — a write the actor did not confirm.
*Covers.* RF-4.5, RF-4.6, RF-5.6, LC-9–LC-12, LC-18.

**D10 · The picker and the removal dialog are `shared/ui` dialogs, not routes.**
*Why.* S-51 and S-52 are compact states on the N22 / N23 anatomy (D02 `420:2968`); a dialog keeps the list and the panel in place, traps focus and returns it (RF-11.2), and needs no URL that could be bookmarked into a half-finished edit.
*Discarded.* Separate pages — lose the context and add routes. Inline expansion of the table — no focus boundary for a confirmation.
*Covers.* RF-5.1, RF-5.3, RF-6.1, RF-6.3, RF-11.2, LC-7, LC-17.

**D11 · The side effects of a change on the affected member are plan 009's event handler; nothing is built here.**
*Why.* RF-9 restates what specification 009 enforces — denial from the next request, closed streams, refused tickets, bounded cancellation, retained evidence — and the `AuthorityChanged` event the command already emits drives all of it. Building anything here would be a second revocation path.
*Discarded.* Calling the realtime and execution ports from the page's use case — duplication and a chance to disagree with 009.
*Covers.* RF-9.1–9.4, RNF-7, RNF-8, LC-23, LC-24.

**D12 · No notification; every change and every refusal is audited with its reason class.**
*Why.* Decision Q5: the member learns at their next request and from the audit trail. `GRANTS_SET` and `GRANT_REMOVED` **[009]** already carry actor, target, previous and new value; the new `GRANT_CHANGE_REFUSED{reasonClass}` makes stale, inactive-target, non-editable-project and nothing-chosen refusals reviewable (RF-10.1) without disclosing the reason beyond the safe code on the wire.
*Discarded.* An email per change — Q5's rejected options. Auditing only successes — refusals are where races and misuse show.
*Covers.* RF-4.8, RF-10.1–10.3, RNF-12.

**D13 · Every configured number is one validated `GrantListPolicyProperties` record that fails startup.**
*Why.* P13 and FR-X-046: the two page sizes are the only numbers of this feature; binding them with bounds at startup means an unbounded page can never be served.
*Discarded.* Defaults in code or in the client — forbidden. Reusing the sessions list size **[007]** — a coupling between unrelated surfaces.
*Covers.* RF-2.1, RF-5.1, RNF-11.

**D14 · No dedicated mobile pages; the table becomes one record per member, and every text comes from the feature-scoped i18n bundles [002 D16].**
*Why.* R17 follows the D01 responsive policy (S-02); FR-X-055 requires AA reflow at every width; `AuthorityStatements` **[009 D13]** shared across widths keeps every statement present (RNF-15). The server sends enum values; the client labels them, so a language switch changes display only.
*Discarded.* Separate mobile components — drift. Server-rendered labels — ties language to the request.
*Covers.* RF-7.2, RF-11.1–11.3, RNF-13–15, LC-28, LC-29.

**D15 · Errors are RFC 9457 problem details with safe codes and a correlation id [001 D12].**
*Why.* F-X-03; the four new codes say only what the client must do next (reload the row, close the panel, choose something); the two reused codes keep their meaning; the correlation id joins the audit record to the actor's report.
*Discarded.* Ad-hoc error bodies; a free-text reason from the server (the i18n bundle carries the wording).
*Covers.* RF-7.3, RF-10.2, RF-10.3.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**) | Pure rules: `GrantEditRule` exhaustively over target status × project state × current grant × requested set × version match; `ProjectState.isEditable`; `ProjectGrantRow` mapping (inert flag, `isActor`, `editable`). |
| BE integration | Spring Boot test + **real PostgreSQL** (area A; Testcontainers **test-only**) | `SetProjectGrants` with the expected version under lock, the project-state check, audit rows and outbox rows; `ProjectGrantsQuery` and `GrantCandidatesQuery` through real adapters with RLS under the runtime role; configuration binding. |
| Module boundary | ArchUnit (**test-only**) | `access.application` depends on no Spring or persistence class; the page's use cases import no project-content, approval, run, intent, session or notification package; `project` and `identity` are reached only through the two ports. |
| API contract | Spring MockMvc (**test-only**) | The three endpoints' shapes, the six problem codes, the mandatory four flags, no verified email, content, reason class or foreign row on the wire. |
| Concurrency | JUnit 5 with real PostgreSQL | Two administrators saving one grant; a save racing a suspension; a save racing a project trash; a picker addition racing a colleague's. |
| Security | Spring Boot test with the runtime role, real PostgreSQL | Cross-tenant list, candidates and save attempts; route project of another tenant; Member and suspended actor attempts. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for an absent, zero or unbounded page size. |
| FE unit | Vitest + Testing Library (**test-only**) | Tab and row states, filters, panel behaviour (independent boxes, notices, Cancel, draft registration, single panel), dialog and picker flows, outcome handling per code, nothing in storage. |
| FE e2e + accessibility | Playwright + axe (**test-only**) | Journeys for US-1 … US-10 in EN / ES / FR at desktop, tablet and mobile widths against the BE doubles; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS, iOS) — versions pinned in the release-support matrix | Grouped checkboxes announced with their group, dialog focus entry and return, live outcomes without focus theft, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1 | FE e2e | `grantsPage.reachedFromSettingsTabAndReviewProjectGrants` | 3 |
| RF-1.2, LC-1, LC-3 | BE integration, FE e2e | `grantsPage.opensForOwnerAndAdminWithOrWithoutGrant`; `grantsPage.ownerWithoutGrantSeesListAndCanAddSelf` | 3 |
| RF-1.3, RF-8.5, LC-2 | API contract, FE unit | `grantsPage.memberAndSuspendedActorDeniedScope`; `tab.hiddenWithoutTenantMembersAuthority` | 3 |
| RF-1.4, LC-15 | BE integration | `grantsPage.unreachableWhileTenantSuspendedOrDeleting` | 3 |
| RF-1.5, LC-12, LC-13 | BE integration, FE unit | `grantsPage.readOnlyWhileTrashedOrDeleting`; `grantsPage.editableWhileArchived`; `readOnlyNotice.statesReason` | 3 |
| RF-1.6, RF-10.3, RNF-9 | API contract | `grants.wireCarriesNoContentEmailCredentialOrForeignRow` | 10, 11 |
| RF-1.7, LC-16 | Concurrency | `save.actorDemotedMidEditRefusedAsStale` | 3 |
| RF-2.1, RF-2.3, LC-21 | BE integration | `list.oneRowPerGrantOrderedByNamePaged`; `list.revokedMembershipNeverListed` | 4 |
| RF-2.2 | FE unit | `table.rendersFourCellsAndYouMarker` | 4 |
| RF-2.4, LC-20 | BE integration, FE unit | `list.suspendedRowReadOnly`; `table.suspendedRowShowsStateAndStatement` | 4 |
| RF-2.5, LC-22 | BE integration, FE unit | `list.searchByNameOrEmailAndStatusFilter`; `filters.currentRevisionsNotRendered` | 4 |
| RF-2.6, LC-5 | BE unit, FE unit | `row.inertCanRunComputed`; `table.inertNoteBesideGranted` | 4 |
| RF-2.7 | FE unit | `grants.fetchOnOpenAndAfterMutationsNeverInterval` | 4 |
| RF-2.8 | FE unit | `decisionHistory.routesToProjectGrantAuditScope` | 4 |
| RF-3.1, RF-3.8 | FE unit | `panel.opensWithCurrentFlagsAndTenantRoleOnly` | 5 |
| RF-3.2, RF-7.1, RF-7.2 | FE unit | `statements.panelCardAndSubStatementVerbatimOnEveryVariant` | 5, 9 |
| RF-3.3, LC-6 | FE unit, BE integration | `panel.checkboxesIndependent`; `save.viewerBesideAuthorRecorded` | 5 |
| RF-3.4 | FE unit | `panel.inertNoteWhileCanRunWithoutRole` | 5 |
| RF-3.5 | FE unit | `panel.removalNoticeWhenNothingTicked` | 5 |
| RF-3.6 | FE unit | `panel.cancelPurgesDraftAndCloses` | 5 |
| RF-3.7, LC-25, LC-26 | FE unit, e2e | `panel.registersStampedDraftAndSinglePanelGuard`; `signOut.guardSavesGrantUnderOriginalScope`; `switch.guardSavesGrantUnderOriginalProject` | 5 |
| RF-4.1, RNF-3 | BE integration | `save.atomicSetVersionAuditOutbox` | 6 |
| RF-4.2, LC-4 | BE integration | `save.ownerAndAdminMaySaveIncludingSelf` | 6, 7 |
| RF-4.3 | FE unit | `save.noStepUpNoDialogForNonEmptySave` | 6 |
| RF-4.4, LC-8 | BE integration | `save.unchangedWritesNothing` | 6 |
| RF-4.5, LC-10, LC-11 | BE integration | `save.refusedForNonActiveTargetOrNonEditableProject` | 6 |
| RF-4.6, RNF-4, LC-9 | Concurrency, FE unit | `save.staleVersionRefusedUnderLock`; `panel.staleReloadsRowAndReopens` | 6 |
| RF-4.7, RNF-10, LC-30 | BE integration, FE unit | `save.repeatedKeyReturnsRecordedOutcome`; `panel.retryReusesKey` | 6 |
| RF-4.8, RNF-12 | BE integration | `feature.neverCallsNotificationPort` | 6 |
| RF-4.9, RF-8.1, RF-8.2 | ArchUnit, BE integration | `feature.touchesOnlyGrantRowAndVersion`; `module.importsNoMembershipSessionApprovalRunOrContentPackage` | 10 |
| RF-5.1, RF-5.2, LC-1, LC-21, LC-22 | BE integration, FE unit | `candidates.activeWithoutGrantActorIncludedSuspendedRevokedExcluded`; `picker.searchAndPaging` | 7 |
| RF-5.3 | FE unit | `picker.choiceOpensAddPanelUnticked` | 7 |
| RF-5.4, LC-4 | BE integration | `save.fromPickerCreatesGrant` | 7 |
| RF-5.5, LC-19 | FE unit, API contract | `picker.nothingTickedRefusedInline`; `save.emptyOnNoGrantIs400` | 7 |
| RF-5.6, LC-18 | Concurrency | `candidates.additionRacingColleagueRefusedStale` | 7 |
| RF-5.7 | BE integration | `candidates.saveCreatesNoMembershipOrInvitation` | 7 |
| RF-6.1, LC-7 | FE unit | `removeDialog.statementsConfirmAndCancel` | 8 |
| RF-6.2 | BE integration | `save.allFalseRemovesGrantWithVersionAndAudit` | 8 |
| RF-6.3 | FE unit | `removeDialog.cancelReturnsToPanelUnchanged` | 8 |
| RF-6.4, LC-17 | FE unit, BE integration | `removeDialog.secondPersonOnOwnRow`; `save.selfRemovalDeniesOwnNextAction` | 8 |
| RF-6.5 | BE integration | `removal.leavesMembershipAndReappearsInCandidates` | 8 |
| RF-7.3 | FE unit | `unavailable.alwaysExplainedNeverSilent` | 9 |
| RF-8.3 | API contract | `save.bodyIsFourMandatoryFlagsOnly` | 10 |
| RF-8.4, RNF-6, LC-27 | Security | `grants.foreignTenantProjectDeniedSafely`; `grants.rlsBlocksCrossTenantRows` | 10 |
| RF-9.1, LC-23 | BE integration | `change.targetNextWriteRefusedStaleThenDeniedScope` | 10 |
| RF-9.2, LC-24 | BE integration | `change.authorityChangedEventDrivesStreamsAndWork` (recording doubles of **[009]** ports) | 10 |
| RF-9.3 | BE integration | `change.endsNoSession` | 10 |
| RF-9.4, RNF-8 | BE integration | `change.leavesAuthoredEvidenceAndVerdicts` | 10 |
| RF-10.1, RF-10.3 | BE integration | `audit.entryPerChangeAndRefusal`; `audit.containsNoSecretOrContent` | 11 |
| RF-10.2 | API contract | `problems.sixSafeCodesWithCorrelationId` | 11 |
| RF-10.4 | Concurrency | `save.allOrNothingUnderFailure` | 11 |
| RF-11.1, RF-11.2, RNF-13, RNF-15, LC-28 | FE e2e + axe, manual | `a11y.grantsPage.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 12 |
| RF-11.3, RNF-14, LC-29 | FE e2e | `languageSwitch.changesDisplayOnlyTicksUnchanged` | 12 |
| RNF-1, RNF-2 | BE integration | `feature.noGrantInferredNoClientAuthority` | 2 |
| RNF-5 | BE unit, integration | `grantEditRule.exhaustiveTruthTable`; `failClosed.everyBadFactRefuses` | 6 |
| RNF-7 | BE integration | `change.nextRequestDerivedUnderNewGrant` | 10 |
| RNF-11 | Startup | `grantListPolicy.startupFailsOnAbsentOrUnboundedSize` | 4 |
| RNF-16 | all of the above | Areas A and B: grant races, stale writes, revoked grants during jobs, cross-tenant attempts | 13 |
| LC-14 | BE integration | `projectDeleted.grantsEndAndPageUnreachable` (through **[009]** `GrantCommands`) | 3 |
| US-11 | all adversarial cases above | Member administration, project-role administration, foreign tenant, suspended / revoked target, stale commit, content disclosure, evidence erasure suites | 1 |

### 5.3 Not tested here
The matrix, the derivation, the administration rule, the version guard, revocation side effects and RLS policies (plan 009); the members page and its dialogs (010); invitation acceptance (008); the bootstrap (012); session, boundary and denied-scope mechanics (006); the switch guard (016); project lifecycle transitions (F-PRJ-07); the audit trail surface (F-APR-04). Their stubs return fixed answers; `SetProjectGrants`, `AuthorityChecker`, `AuthorityChanged`, `ProjectStatePort` and `IdentityNamesPort` are exercised through this feature's own calls or recording doubles.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q1 and Q3 were recorded in Figma on 27 October 2026 as **S-51** and **S-52**; Q2, Q4, Q5 and Q6 were recorded as notes on **A-57**, **A-55**, **S-06**, **S-13** and **S-04**; this plan implements each as decided (picker D6 / D10, removal dialog D3 / D10, authority D4, stale guard D2, no notification D12, list content D5). The items below remain open or are earlier decisions this plan builds on.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| S-51 picker (decided, drawn) | `AddProjectMemberPicker` as a `shared/ui` dialog with search and paged candidates, copy and layout of D02 `420:2968`. | None; the query is unchanged. |
| S-52 removal dialog (decided, drawn) | `RemoveFromProjectDialog` on the N23 anatomy with the statements of RF-6.1, copy of D02 `420:2968`. | None; the all-false save is unchanged. |
| S-47 inert canRun statement (decided, 009) | `AuthorityStatements.inertCanRun` **[009]** rendered in the row and the panel. | Adjust the text key only. |
| S-49 Admin view (decided, 009) | The Admin view of R17 is the Owner view; no variant component. | None here. |
| S-04 actions without destination | "Add project member" has the picker; "Edit … grants" is the drawn panel. | None here. |
| S-06 role variants | One component set rendered for Owner and Admin; a Member meets the denied-scope state. | Add a variant only if the design draws one. |
| S-13 copy defects | "Current revisions" not rendered; "All statuses" = membership status. | Render the control with the decided meaning. |
| S-02 responsive coverage | R17, the panel, the picker and the dialog follow the D01 responsive policy (`414:2645`; D14). | None: the components are unchanged. |
| S-17 Help & shortcuts | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |
| A-53 grant combinations (decided, 009) | Any subset accepted; inert flag computed (**[009 D5]**). | None here. |
| A-55 permission version (decided, 009; note from 011) | The target's version is the expected-version guard (D2). | If a separate row version were decided, `GrantEditRule` compares that instead; the FE contract is unchanged. |
| A-57 administration authority (decided, 009; note from 011) | `TENANT_MEMBERS` + `AdministrationRule` with self allowed (D4). | Change the rule in plan 009 only. |
| A-58 lifecycle grant (decided, 009) | Not consulted here; the page never performs lifecycle. | None here. |
| A-11 configuration values | `GrantListPolicyProperties` holds the two page sizes; values from the catalogue. | Rename or regroup; no use case changes. |
| A-35 persistence tooling | The index against the placeholder access API and migration tool. | Rename base types and migration format. |
| A-36 identity-scoped membership listing | Not used; the list and the candidates are tenant-scoped reads under RLS; names come through `IdentityNamesPort`. | None here. |

No new register item is raised by this plan: the two new ports stay inside P10 and P11, no library is added, and no notification is sent.

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.7) | `routes`, `AuthorityChecker(TENANT_MEMBERS)`, C10 filter, `ProjectStatePort`, `ReadOnlyNotice`, `DeniedScopeHandler` | D4, D7 | RF-1 rows |
| RF-2 (2.1–2.8) | `ProjectGrantsQuery`, `ProjectGrantRow`, `IdentityNamesPort`, `GrantsTable`, `SearchAndStatusFilter`, `Pagination`, audit route | D5, D13 | RF-2 rows |
| RF-3 (3.1–3.8) | `GrantEditPanel`, `CheckboxGroup`, drafts registry, `AuthorityStatements` | D3, D8, D14 | RF-3 rows |
| RF-4 (4.1–4.9) | `SetProjectGrants` (extended), `GrantEditRule`, idempotency outcome, outcome handling | D1, D2, D3, D9, D12 | RF-4 rows |
| RF-5 (5.1–5.7) | `GrantCandidatesQuery`, `GrantCandidateRow`, `AddProjectMemberPicker`, panel "Add" mode | D3, D6, D9, D10 | RF-5 rows |
| RF-6 (6.1–6.5) | `RemoveFromProjectDialog`, `Apply(REMOVE)` | D3, D10 | RF-6 rows |
| RF-7 (7.1–7.3) | `AuthorityStatements` variants, `ReadOnlyNotice`, problem codes | D14, D15 | RF-7 rows |
| RF-8 (8.1–8.5) | port set, C10 filter, `PUT` body, `TENANT_MEMBERS` | D1, D3, D4 | RF-8 rows |
| RF-9 (9.1–9.4) | `AuthorityChanged` → `AuthorityChangedHandler` **[009]** | D11 | RF-9 rows |
| RF-10 (10.1–10.4) | audit events, problem codes, one transaction | D12, D15 | RF-10 rows |
| RF-11 (11.1–11.3) | record layout, `shared/ui`, i18n bundles | D14 | RF-11 rows |
| RNF-1 | thin surface over **[009]** | D1 | `feature.noGrantInferredNoClientAuthority` |
| RNF-2 | server-side checks at save time | D4 | same |
| RNF-3 | one transaction | D1 | RF-4.1 row |
| RNF-4 | version under lock | D2 | RF-4.6 row |
| RNF-5 | `GrantEditRule`, project state | D2, D7 | `failClosed` |
| RNF-6 | C10 filter, RLS, tenant-scoped queries | D4, D5 | RF-8.4 row |
| RNF-7, RNF-8 | `AuthorityChangedHandler` **[009]** | D11 | RF-9 rows |
| RNF-9 | row shapes, no storage | D5 | RF-1.6 row |
| RNF-10 | idempotency outcome, `UNCHANGED` | D1, D3 | RF-4.7 row |
| RNF-11 | `GrantListPolicyProperties` | D13 | startup |
| RNF-12 | no notification port | D12 | RF-4.8 row |
| RNF-13, RNF-15 | `shared/ui`, record layout | D14 | RF-11 rows |
| RNF-14 | i18n bundles | D14 | `languageSwitch` |
| RNF-16 | concurrency and security suites | D2, D4 | areas A / B row |
| LC-1 … LC-30 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30) |

Self-check: every RF criterion (63), every RNF (16) and every limit case (30) of `specProjectGrants.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
