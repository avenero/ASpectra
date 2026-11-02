# Plan 013 — Tenant chooser

**Feature** The page where a signed-in identity sees the tenants it belongs to, with its role, its membership state and its own project grants in each, and chooses which one to enter · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — aligned with specification v0.1 (decisions Q1–Q7, derived rules D-a…D-e) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/013-TenantChooser/specTenantChooser.md` v0.1 (the WHAT) · **Governed by** `constitution.md` (principles P1–P16; feature F-IDN-15 as the subject; F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-14, F-IDN-16, F-IDN-17, F-IDN-18 and F-SET-04 at the boundary; F-X-01…F-X-05, F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG), FE board `46:7847` (feature-modular SPA DAG) and decision register `122:2` §5 (the `access` context owns memberships, invitations, project grants and permission versions) · **Builds on** plan 001 (Login: landing and the scope epoch), plan 003 (Enrollment: the gate and its handler), plan 006 (Sessions: session facts and the protected boundary), plan 007 (Logout: the shell identity area and the sign-out coordinator), plan 009 (Roles and grants: the `access` core, the C10 derivation and the identity-scoped membership listing) and plan 010 (Tenant members); elements they own are tagged **[001]** / **[003]** / **[006]** / **[007]** / **[009]** / **[010]** and are extended, never duplicated. Specifications 012 (workspace creation), 014 (suspended-tenant screen) and 016 (guarded switch) have no plan yet; this plan reaches them only by route and names the contract it expects.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's seven decisions (Q1–Q7) and five derived rules (D-a…D-e) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-36, A-54; S-55, S-56, S-48, S-07, S-02, S-03, S-17; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The chooser reads memberships and grants and requests a tenant entry; nothing on it accepts, approves, launches, replays or retargets a Run, and the entry request carries no Run, approval or mapping intent. | RF-6.5, RF-8.3, RNF-7 |
| **P3** The browser expresses intent, never authority | The list is a server-built read model: card kind, order, project lines and the MFA line are computed on the server and rendered as given. "Enter" sends a tenant identifier as requested scope; whether the entry is granted, refused or gated is decided by the C10 derivation of plan 009 at that moment. The client never sorts, classifies or gates. | RF-3, RF-4.4, RF-4.5, RF-6.1, RF-6.3, RNF-2 |
| **P6** Fail closed | A membership not active, a tenant not active, a revoked membership, an ended session or a derivation that yields nothing refuses the entry and shows no content; a listing failure renders no card and no stale list. | RF-6.3, RF-7.3, RNF-3, RNF-4 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security session support. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. No STOMP subscription is opened by this feature; Zustand is not used (no editor fork exists on the page). Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: two read-side use cases and three pure rules added to the hexagonal bounded context `access` **[009]**; `identity`, `tenant`, `project` and `audit` are reached only through published ports; no foreign repository. FE: the chooser page in the feature module shared with Login, Signup and Invitation; one shell action at app-level composition **[007]**; TanStack Query owns the server facts. | §2 |
| **P12** Two independent tenancy gates | Memberships and grants are tenant rows under `FORCE RLS`. The chooser's single cross-tenant read is the identity-scoped membership listing of register A-36, served by the identity-predicate policy plan 009 D10 defines and a narrow control-plane query; project names and grants are read only for rows whose membership and tenant are active. Runtime authorisation is the second gate: entry is a derivation, never a consequence of the list. | RF-3.1, RF-10.3, RNF-1, RNF-5 |
| **P13** Configured values are never defaulted | This feature introduces no numeric value; the bounds it meets belong to sessions (plan 006) and the guarded switch (specification 016). | RNF-9 |
| **P14** Release bar | WCAG 2.2 AA on the page and its four card variants with automated and manual assistive evidence; EN / ES / FR for every statement. | RF-9, RNF-11…13 |
| **F-X-01** C10 context | Listing and entry derive their facts from the session (plan 006 `SessionAssertionPort`); no tenant C10 exists while the chooser is shown. Entry is the tenant-level derivation of plan 009 (`DeriveAuthorityUseCase`), which rechecks membership, tenant state and the enrollment gate **[003]**; a refusal is audited here. | RF-1.6, RF-6.1–6.4, RF-10.1 |
| **F-X-02** Request integrity | The listing is a safe read with bounded retry on transport failure. The entry request writes nothing but an audit row on refusal, so it is naturally idempotent; no retry queue, no replay. | RF-6.1, RF-7.1 |
| **F-X-03** Secrets and safe problems | No secret exists on the page. Problems carry safe codes (`access.entry-refused`, `auth.enrollment.required`) with a correlation id; the refusal cause lives in audit only; browser storage holds nothing of the list. | RF-6.3, RF-10.2, RNF-10 |
| **F-X-04** Accessibility and languages | Cards are labelled regions in the order of RF-3.5; state is text, never colour alone; a refetch after a refused entry announces without stealing focus; collation and statements follow the UI locale header. | RF-9, RNF-11, RNF-12 |
| **F-X-05** Audit log | Each refused entry is appended through the `audit` port with identity, requested tenant, cause and calling context; a granted entry is recorded by the derivation itself. | RF-10.1 |
| **F-X-07** Responsive surfaces | M20 (`432:4257`) is the drawn counterpart of N03; one page component follows the D01 responsive policy at other widths with `SecurityStatements` shared across widths. | RF-9 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access` [009], read side

The context, its four layers and its published ports exist from plan 009 (memberships, grants, matrix, C10 derivation, the identity-scoped membership listing). Plans 008, 010 and 011 added their use cases. This plan adds two read-side use cases, three pure rules and one endpoint pair; it creates no aggregate and no table.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `TenantMembership`, `ProjectGrant`, `GrantSet`, `TenantRole` **[009]** | Reused unchanged as the facts the listing projects. `GrantSet.isEmpty()` and the role flags give the project line; `GrantSet` with `canRun` and no role is the "canRun only" case (plan 009 Q2). | RF-4.2, RF-4.3, RNF-6 |
| | `ChooserCardKind` | Pure rule `(membershipStatus, tenantStatus) → ENTERABLE \| SUSPENDED_MEMBERSHIP \| SUSPENDED_TENANT \| DELETING_TENANT \| NOT_LISTED`: tenant state first (`SUSPENDED` → suspended tenant, `DELETING` → deleting tenant), then membership (`SUSPENDED` → suspended membership, `REVOKED` → not listed), else enterable. | RF-3.2, RF-3.4, RF-5.1–5.3, LC-4–LC-7 |
| | `ChooserOrder` | Pure comparator: kind rank (enterable, suspended membership, suspended tenant and deleting tenant together), then tenant name under the collation of the request locale, then tenant id as a stable tie-break; project lines by project name under the same collation. | RF-3.5, LC-19, LC-20 |
| | `MfaLineRule` | Pure rule `(tenantRole, policyRequiresMfaForMembers, hasActiveFactor) → NONE \| OWNER \| ADMIN \| MEMBER_POLICY` plus `notYetEnrolled`; the same inputs `RequirementReasonPort` **[003]** gives the enrollment gate, so the line and the gate cannot disagree. | RF-4.4, RF-4.5, LC-12–LC-14 |
| | `ChooserCard` (read model) | `kind`, `tenantId`, `tenantName`, `tenantRole`, `membershipStatus`, `tenantStatus`, `mfaLine`, `projectLines` (each `projectId`, `projectName`, `grants`, `canRunOnly`) for an enterable card and an empty list otherwise, `noGrants` flag. Never another identity, a member count, a source or a run. | RF-2.6, RF-4.1, RF-4.6, RF-5.5, RNF-1 |
| `access.application` | `TenantChooserQuery` | For the signed-in identity (plan 006 `SessionAssertionPort`): reads the extended `MembershipProjectionPort` **[009]** rows (memberships `ACTIVE` or `SUSPENDED`, tenant name and status, role, and — for rows whose membership and tenant are both active — the live grants with project names); reads `TenantPolicyPort.requireMfaForMembers(tenantId)` **[009]** for each enterable row and `FactorStatusPort.hasActiveFactor(identityId)` **[003]** once; applies `ChooserCardKind`, `MfaLineRule` and `ChooserOrder`; returns the ordered `ChooserCard` list and the person's address and display name (`IdentityFactsPort` **[008]**). Writes nothing; audits nothing (a page view is not an event). | RF-1.6, RF-2.1, RF-3.1–3.7, RF-4, RF-5, RF-7.1, RF-10.3, RF-10.4 |
| | `TenantEntryUseCase` | For a requested `tenantId`: runs `DeriveAuthorityUseCase` **[009]** at tenant level for the session identity. `Derived(C10)` → returns `{tenantId, tenantName, tenantRole}` as routing facts and nothing else (the overview re-derives). `Denied(NO_MEMBERSHIP \| MEMBERSHIP_NOT_ACTIVE \| TENANT_NOT_ACTIVE)` → audits `TENANT_ENTRY_REFUSED{cause}` and answers the single problem `access.entry-refused`. `Denied(ENROLLMENT_GATE)` → the derivation's own `auth.enrollment.required {reason}`, exactly as any tenant request **[003]**. Creates no session, changes no row. | RF-6.1–6.4, RF-6.6, RF-10.1, LC-8–LC-13 |
| | Ports | Reused only: `MembershipProjectionPort` **[009]** (its read model is extended by this plan with `projectLines` for active-and-active rows — a cross-plan item for plan 009, §6), `TenantPolicyPort` **[009]**, `TenantStatusPort` **[009]**, `FactorStatusPort` **[003]**, `RequirementReasonPort` **[003]**, `SessionAssertionPort` **[006]**, `IdentityFactsPort` **[008]**, `AuditPort` **[001]**, `Clock`. One new port consumed from `project`: `ProjectNamesPort.namesOf(tenantId, projectIds)` (published by the `project` context; names only, under the tenant predicate). No write port exists in this feature. | all |
| `access.infrastructure` | Projection query | The control-plane query of plan 009 D10 (sets the identity predicate, reads `tenant_membership` for the identity) extended with a join to `project_grant` restricted to rows whose membership is `ACTIVE` and whose tenant is `ACTIVE`, and a names lookup through `ProjectNamesPort`; a revoked membership never leaves the query. No new table, no new index beyond plan 009's. Persistence access API per A-35. | RF-3.1, RF-3.2, RF-4.2, RF-5.5, RNF-1, RNF-5 |
| | Audit adapter | Event `TENANT_ENTRY_REFUSED{tenantId, cause}` with identity id, correlation id, calling context, timestamp; never a list, a grant or another identity. | RF-10.1, RF-10.2 |
| `access.api` | `MembershipsController` — `/api/me/memberships` | `GET` (session required, else 401) → 200 `{address, displayName, cards: [ChooserCard…]}` in the order of `ChooserOrder` under the UI locale header. No tenant path, no query parameter. | RF-2.1, RF-3, RF-7.1 |
| | `TenantEntryController` — `/api/tenants/{tenantId}/entry` | `POST` (session required) → 200 `{state: ENTERED, tenantId, tenantName, tenantRole}` · 403 `access.entry-refused` (one code, no cause) · 403 `auth.enrollment.required {reason}` **[003]** · 401 without a session. RFC 9457 problems with a correlation id; the server never redirects. | RF-6.1–6.4, RF-10.1 |

**Cross-context touch points.** `identity` — `SessionAssertionPort` **[006]**, `FactorStatusPort` **[003]**, `IdentityFactsPort` **[008]**. `tenant` — `TenantStatusPort` **[009]** (name and status). `project` — `ProjectNamesPort` (names only). `audit` — append-only. `sharedsupport` — `tx`, `contextIds` (correlation id), locale. No membership, grant, invitation, credential, session, approval or run row is written.

**Data model note (P12).** This plan adds no table. The identity-scoped listing is the one cross-tenant read of the product (A-36), served by plan 009's identity-predicate policy; project names and grants are joined only for enterable rows, so a suspended or deleting tenant and a suspended membership never contribute a project name to the result.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login, Signup and Invitation), `app/shell` [007] and `shared/*`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/workspaces` (the chooser). `LandingRouter` **[001]** maps `TENANT_CHOOSER` here; the exits of plans 003 ("Leave setup"), 006 (re-authentication fallback) and 008 ("Leave invitation", "Return to memberships", decline) navigate here. | RF-1.1, RF-1.2, RF-1.3 |
| | `useMemberships()` | TanStack query on `GET /api/me/memberships`: `staleTime` 0, refetch on mount, no interval, focus or reconnect refetch (the background marker of plan 006 is never set by this query); one bounded retry on transport failure (safe read); invalidated by `EntryRefusedHandler`. Returns the server read model verbatim. | RF-3.1, RF-3.7, RF-7.1–7.3 |
| | `useEnterTenant()` | Mutation on `POST /api/tenants/{id}/entry`; no retry; on `ENTERED` advances the scope epoch **[001/006]**, clears the scoped cache and navigates to the tenant overview route with `tenantId`. | RF-6.1, RF-6.2, RF-6.5 |
| | `contracts/memberships` (in `src/contracts`) | Request / response / problem types and runtime guards for the chooser read model, the entry response and the two problem codes; the guard rejects any field beyond the contract (no member, count or content can ever render). | all |
| Views | `TenantChooserPage` | N03: product mark, display name, language switch, marker, heading, the address statement, the card list, the "A context change is not a new target" block, the "No membership for a workspace?" block with "Create workspace", "Sign out". Renders whatever the list holds — several, one or no cards — with no routing of its own. Cards are labelled regions in server order; the list is a live region announcing "Memberships updated" after a refetch without moving focus. | RF-1.3, RF-2.1, RF-2.2, RF-2.5, RF-2.6, RF-3.5, RF-3.6 |
| | `MembershipCard` | One component with four variants by `kind`: *enterable* (tenant name, "Active membership", role, "Your project grants" lines or the "No project grants yet" line, the administration statement, the MFA line with its "Not yet set up" suffix, "Enter <tenant> →"); *suspended tenant* (as drawn, "View access status"); *suspended membership* (state, role, statement, no action); *deleting tenant* ("Deleting · content unavailable", role, statement, "View access status"). | RF-4.1–4.6, RF-5.1–5.5 |
| | `SecurityStatements` **[001]** | Extended with the N03 statements (memberships for this address only and project access checked after choosing; administration ≠ project content; accepted runs keep their pins; invitation or new workspace as the only ways in; content not exposed while suspended / deleting). | RF-8.1–8.5 |
| | `SignupPage`, `VerifyEmailPage` | Untouched. | — |
| Shell | `SwitchWorkspaceAction` (in `app/shell`) | A labelled "Switch workspace" action rendered beside `SignOutAction` **[007]** in the footer identity block on every desktop app-shell route and as an item of the M04 menu; built as the S-07 assumption. Activation → the shell dirty guard of plan 007 (N24 Save / Discard / Stay; a guard save under the current tenant, project and `baseETag`) → on Save acknowledged or Discard → navigate to `/workspaces`; on Stay → nothing. The current scope epoch and tenant context are kept until an entry succeeds. | RF-1.4, RF-1.5, RNF-8, LC-28 |
| Routing | `EntryRefusedHandler` (in `shared/errors`) | On `access.entry-refused` from the entry mutation: invalidate and refetch `useMemberships`, keep the page, announce the update, no toast, no dialog, no navigation. | RF-6.3, LC-8–LC-10 |
| | `EnrollmentGateHandler` **[003]** | Reused: `auth.enrollment.required {reason}` from the entry mutation → the wizard with `returnTo` = the tenant overview route; "Leave setup" there returns to `/workspaces` (plan 003 Q20). | RF-6.4, LC-12, LC-13 |
| | Handoffs | "View access status" → the suspended-tenant route of specification 014 with `tenantId` as requested scope (deleting wording decided by the screen from the tenant state); "Create workspace" → the creation route of specification 012 with no state; "Sign out" → `SignOutCoordinator.signOut({entry: 'N03'})` **[007]**; `ENTERED` → overview route; `ProtectedBoundary` **[006]** wraps the route (an ended session shows R34; re-authentication returns here and `useMemberships` refetches on mount). | RF-2.2, RF-2.3, RF-5.4, RF-6.2, RF-7.4 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry, background marker left unset), `shared/security` (scope epoch, `AccessCredentialHolder` **[006]**), `shared/errors` (handlers above), `shared/i18n` (bundle keys for every statement, label and state in EN / ES / FR), `shared/ui` (labelled `Card` regions, `Button`, `LiveRegion`, 44 px targets). | RF-2.4, RF-9.2, RF-10.2, RNF-11, RNF-12 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | Both endpoints require a session (`SessionAssertionPort`) → 401 otherwise | Route under `ProtectedBoundary` **[006]**; no session → `/sign-in` |
| RF-1.2 | — | `LandingRouter` **[001]** `TENANT_CHOOSER`; exits of 003, 006, 008 navigate to `/workspaces`; `SwitchWorkspaceAction` |
| RF-1.3 | `TenantChooserQuery` returns the list whatever its size; no routing decision in the response | `TenantChooserPage` renders any count; no redirect logic |
| RF-1.4 | — | `SwitchWorkspaceAction` beside `SignOutAction` **[007]**, footer and M04 menu |
| RF-1.5 | — | Shell dirty guard of plan 007 before navigation; scope kept until `ENTERED` |
| RF-1.6 | Listing derives from session facts only; no tenant C10 is built | Page holds no tenant scope; scoped cache untouched until entry |
| RF-2.1 | `GET /api/me/memberships` carries address, display name, cards | `TenantChooserPage` content |
| RF-2.2 | — | "Create workspace" → 012 route with no state |
| RF-2.3 | Sign-out use case **[007]** | `SignOutCoordinator.signOut({entry: 'N03'})` **[007]** |
| RF-2.4 | Collation follows the locale header; facts unchanged | Language switch re-renders; query data kept |
| RF-2.5 | — | "Help" opens N43; no Privacy / Terms rendered |
| RF-2.6 | `ChooserCard` shape; contract guard | Page renders only the read model |
| RF-3.1 | `TenantChooserQuery` over the extended `MembershipProjectionPort` **[009]** | `useMemberships` on mount |
| RF-3.2 | Projection query excludes `REVOKED`; `ChooserCardKind` → `NOT_LISTED` | Empty block when `cards` is empty |
| RF-3.3 | Identity-predicate policy (plan 009 D10) returns the identity's rows only | — |
| RF-3.4 | `ChooserCardKind` (tenant state first) | `MembershipCard` variant by `kind` |
| RF-3.5 | `ChooserOrder` under the locale collation | Cards rendered in server order |
| RF-3.6 | Full list, no paging parameter | No search, filter or paging control |
| RF-3.7 | — | No interval / focus / reconnect refetch; background marker unset |
| RF-4.1 | `ChooserCard` enterable fields | Enterable variant |
| RF-4.2 | Projection joins live grants with names for active-and-active rows; `ChooserOrder` for lines | Project lines verbatim |
| RF-4.3 | `canRunOnly` and `noGrants` flags | "canRun only · no project role yet"; "No project grants yet …"; Enter present |
| RF-4.4 | `MfaLineRule` with `TenantPolicyPort` **[009]** | MFA line text by `mfaLine` |
| RF-4.5 | `MfaLineRule.notYetEnrolled` from `FactorStatusPort` **[003]** | Suffix rendered; Enter present |
| RF-4.6 | Read model carries nothing else | Contract guard |
| RF-5.1 | `SUSPENDED_TENANT` card: no `projectLines` | Suspended-tenant variant with "View access status" |
| RF-5.2 | `SUSPENDED_MEMBERSHIP` card | Suspended-membership variant, no action |
| RF-5.3 | `DELETING_TENANT` card | Deleting-tenant variant with "View access status" |
| RF-5.4 | Specification 014's screen reads identity-scoped metadata; no derivation here | Navigation to the 014 route with `tenantId`; back → `/workspaces` refetches |
| RF-5.5 | Projection join restricted to active-and-active rows | Non-enterable variants render no project line |
| RF-5.6 | Each row classified independently | Each card independent |
| RF-6.1 | `TenantEntryUseCase` → `DeriveAuthorityUseCase` **[009]** at tenant level | `useEnterTenant` sends `tenantId` only |
| RF-6.2 | `ENTERED` routing facts | Scope epoch advance **[001/006]**, cache cleared, overview route |
| RF-6.3 | `Denied` → audit + `access.entry-refused` | `EntryRefusedHandler` refetches and re-renders |
| RF-6.4 | `Denied(ENROLLMENT_GATE)` → `auth.enrollment.required` **[003/009]** | `EnrollmentGateHandler` **[003]** → wizard with `returnTo` |
| RF-6.5 | Entry writes nothing; the overview derives current grants | No pending intent stored; no Run action exists |
| RF-6.6 | Entry changes no row | — |
| RF-7.1 | — | `staleTime` 0; refetch on mount; invalidation on refusal |
| RF-7.2 | No push, no subscription | No STOMP, no polling |
| RF-7.3 | — | Query cache in memory only; nothing persisted |
| RF-7.4 | Session handling **[006]** | `ProtectedBoundary` **[006]**; return → mount → refetch |
| RF-8.1 | Address in the response | `SecurityStatements` (address statement) |
| RF-8.2 | — | `SecurityStatements` on the enterable card |
| RF-8.3 | — | "A context change is not a new target" block |
| RF-8.4 | — | "No membership for a workspace?" block |
| RF-8.5 | One read model for every variant | One statement source for every variant and width |
| RF-9.1 | — | Breakpoint reflow of `TenantChooserPage` and `MembershipCard` |
| RF-9.2 | — | `shared/ui` targets, labels, focus; keyboard order = server order |
| RF-9.3 | — | Breakpoint layout replaceable (S-03) |
| RF-10.1 | Audit adapter `TENANT_ENTRY_REFUSED{cause}` in the entry transaction | Correlation id header |
| RF-10.2 | Problems carry safe codes only | Nothing in storage, logs or analytics |
| RF-10.3 | Identity-predicate policy and control-plane query (plan 009 D10) | — |
| RF-10.4 | Read model fields per kind | Variants render only their fields |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The chooser is a server-built read model — card kind, order, project lines and MFA facts — and the client renders it as given.**
*Why.* P3: the browser never decides an outcome, however harmless. Which card a membership becomes, whether a project line says "canRun only", whether the MFA line says "not yet set up", and the order of the cards all follow rules that depend on server facts (tenant policy, factor status, collation); computing them once on the server keeps the page identical for every client and makes RF-3.4, RF-3.5, RF-4.3 and RF-4.4 testable as pure rules (`ChooserCardKind`, `ChooserOrder`, `MfaLineRule`) without a browser.
*Discarded.* Returning raw memberships and classifying in the client — three rules duplicated in TypeScript and a page that can disagree with the gate. Returning per-kind lists — the order across kinds would be a client decision.
*Covers.* RF-3.4, RF-3.5, RF-4.3–4.5, RF-8.5, RNF-2, LC-5, LC-12–LC-17, LC-19.

**D2 · One identity-scoped listing serves the chooser, extending plan 009's `MembershipProjectionPort` with project names and grants for enterable rows only.**
*Why.* The listing is the single cross-tenant read of the product (A-36) and plan 009 D10 already defines its policy and its narrow query; the chooser needs exactly that read plus, for a membership that is active in an active tenant, the person's own grants with project names (RF-4.2). Joining grants and names only for those rows makes RF-5.5 and RNF-5 hold in the query itself: a suspended or deleting tenant and a suspended membership cannot contribute a project name because the join never reaches them. One read keeps the page one round trip and avoids N tenant-level derivations.
*Discarded.* Running `DeriveAuthorityUseCase` per tenant to collect grants — N derivations, each building a tenant context the person has not chosen. A materialised chooser table — a second source of truth for memberships. Reading project names for every row and hiding them in the client — the names would be on the wire.
*Covers.* RF-3.1–3.3, RF-4.2, RF-5.5, RF-10.3, RNF-1, RNF-5, LC-4, LC-18, LC-34.

**D3 · "Enter" is a dedicated entry request that runs plan 009's tenant-level derivation, audits a refusal and returns routing facts only.**
*Why.* RF-6.1 makes entry a request decided from current facts; plan 009's `DeriveAuthorityUseCase` is that decision and already covers membership state, tenant state and the enrollment gate. A dedicated endpoint lets the chooser learn the outcome before navigating: a refusal is audited with its cause (RF-10.1) and answered with one code, and the chooser refreshes (D4) instead of landing the person on the in-tenant denied-scope panel N28, which is for a scope lost *inside* a tenant (plan 006). The response carries only what routing needs; authority is re-derived by the overview on its first request (P3).
*Discarded.* Navigating straight to the overview and letting its first request fail — the refusal would surface as N28 ("Refresh permissions / Choose permitted project"), a wrong message for a tenant the person never entered. Checking the card's state in the client before navigating — a stale page would enter. Returning grants or policy in the entry response — authority on the wire.
*Covers.* RF-6.1, RF-6.2, RF-6.5, RF-6.6, RF-10.1, RNF-2, RNF-7, LC-8–LC-10, LC-18, LC-32.

**D4 · A refused entry has one problem code; the cause stays in audit; the client refetches the list and renders the new state with no separate error.**
*Why.* Decision Q4: the card in its new state *is* the explanation, and a message naming the cause would say the same thing twice. One code keeps the client logic to "refetch and render"; the audit row keeps the cause for the tenant and the platform.
*Discarded.* Distinct codes per cause — the client would branch on authority facts. A toast or dialog — rejected by Q4. The N28 panel — a tenant-internal state.
*Covers.* RF-6.3, RF-10.1, LC-8–LC-10.

**D5 · The enrollment gate is met on arrival through the same `auth.enrollment.required` every tenant request raises, handled by plan 003's handler; the chooser never gates.**
*Why.* Decision Q5 and plan 003 D7: the gate is a derivation outcome evaluated where C10 is built; the entry request is a tenant request, so the gate fires there with its reason, and `EnrollmentGateHandler` **[003]** routes to the wizard with `returnTo` = the overview route and "Leave setup" back to the chooser (003 Q20). The chooser's MFA line merely states what will happen.
*Discarded.* Disabling "Enter" when no factor exists — the chooser deciding authority (P3) and contradicting Q5. Routing to the wizard from the chooser — a second gate implementation.
*Covers.* RF-4.5, RF-6.4, RNF-2, LC-12, LC-13, LC-25.

**D6 · The MFA line's facts come from the server through `RequirementReasonPort` [003] and `FactorStatusPort` [003], the same sources the gate uses.**
*Why.* RF-4.4 and RF-4.5 must match the gate exactly: a line that promises no enrollment while the gate demands one would be a lie. `MfaLineRule` takes the same inputs — role, tenant policy flag, active factor — so the two cannot drift; factor status is read once per listing, and a pending enrollment is never reported as a factor (plan 003 RF-6.4).
*Discarded.* Inferring the line from the role in the client — ignores tenant policy and factor status. Omitting the "not yet set up" suffix — rejected by Q5.
*Covers.* RF-4.4, RF-4.5, LC-12–LC-14, LC-25.

**D7 · "Switch workspace" lives in `app/shell` beside plan 007's `SignOutAction`, runs the shell dirty guard first, and keeps the current scope until an entry succeeds; the scope epoch advances on `ENTERED`.**
*Why.* Decision Q1 and register S-07: the action must exist on every app-shell screen, desktop and M04 menu, and the shell is the one place that wraps every authenticated route (plan 007 D15). Running the N24 guard before leaving honours FR-IDN-130 step (1) and FR-IDN-131 (a guard save under the original scope); keeping the context until an entry succeeds lets "Stay" and a return from the chooser cost nothing and matches F-IDN-18's order, whose steps (2)–(5) run when the new tenant is entered — the epoch advance and cache clearing of plan 001 D8 are that transition's client half, pending specification 016's own plan.
*Discarded.* Ending the tenant context when the chooser opens — a person who returns to the same tenant would re-derive for nothing and a guard "Stay" would be meaningless. A header tenant switcher that bypasses the guard — the header WORKSPACE selector (`395:667`) is a route to this chooser (016 Q1; S-07), never a bypass. A modal chooser inside the shell — N03 is a page.
*Covers.* RF-1.4, RF-1.5, RF-6.2, RNF-8, LC-27, LC-28.

**D8 · Freshness is "on open and on Enter": the listing query refetches on mount and after a refused entry, never on interval, focus or reconnect, and opens no subscription.**
*Why.* Decision Q7. A membership change while the page is idle must not be pushed (no STOMP for an identity-scoped page), and TanStack Query's focus and reconnect refetches would silently refresh a page the person is reading, moving cards under their pointer; `staleTime` 0 with refetch on mount gives RF-7.1, and the handler of D4 gives RF-6.3. The background marker of plan 006 is never set, so no request of this feature is mistaken for human activity or for background noise.
*Discarded.* Polling — rejected by Q7. A realtime subscription — none exists before a tenant is entered. Focus refetch — unannounced movement of cards.
*Covers.* RF-3.7, RF-7.1–7.3, RNF-4, LC-11, LC-31.

**D9 · Ordering and collation happen on the server under the UI locale header.**
*Why.* RF-3.5 orders alphabetically "in the collation of the interface language"; browsers differ in their collation tables, and a client sort would also be a client decision about order (D1). The locale header plan 002 D15 already sends selects the collation; the tie-break on tenant id keeps the order stable for equal names (LC-20).
*Discarded.* Client-side sort — collation differences and a duplicated rule.
*Covers.* RF-3.5, RF-2.4, LC-19, LC-20, LC-24.

**D10 · Nothing of the chooser is persisted or configured: no table, no cache beyond the in-memory query, no numeric value.**
*Why.* RNF-9 and RNF-10; the page is a projection of facts other features own, and a persisted copy would be a stale authority source (FR-IDN-132). P13 is satisfied vacuously, and §6 names no configuration item.
*Discarded.* Caching the list in browser storage for a faster reopen — RF-7.3, F-X-03 allowlist.
*Covers.* RF-7.3, RF-10.2, RNF-9, RNF-10, LC-31.

**D11 · "View access status" is a navigation to specification 014's route with the tenant as requested scope; the screen reads identity-scoped membership metadata and derives no tenant authority.**
*Why.* Decision Q6 and FR-IDN-116: N20 shows the person's own membership in a suspended (or deleting, S-55) tenant and the recovery, security and sign-out routes; it is not an entry and must not build a tenant context. The chooser passes the tenant id only; what the screen shows is 014's contract, built on the same identity-scoped read (A-36). Returning to the chooser remounts the page and refetches (D8).
*Discarded.* Expanding the status inside the card — N20 is a drawn screen with its own routes. Entering the suspended tenant to show N20 — a tenant context for a tenant that must stay closed.
*Covers.* RF-5.4, RF-7.1, LC-6, LC-7, LC-21.

**D12 · One page component with breakpoint layouts renders the chooser at every width; statements come from `SecurityStatements` [001].**
*Why.* M20 is the drawn counterpart of N03 (S-02, S-03); FR-X-055 requires AA reflow at every other width; one component with shared statements keeps every card, statement and action present at every width by construction (RNF-13), and the labelled card regions in server order give the keyboard order of RF-9.2.
*Discarded.* A separate mobile component — drift. Reusing M04 — it is the tenant overview, a different screen with different facts.
*Covers.* RF-8.5, RF-9.1–9.3, RNF-11, RNF-13.

**D13 · Errors are RFC 9457 problem details with safe codes and a correlation id [001 D12].**
*Why.* F-X-03; `access.entry-refused` and `auth.enrollment.required {reason}` are the only problems this feature raises beyond 401; the correlation id joins the audit row of a refusal to the person's report.
*Discarded.* Ad-hoc error bodies; a cause in the problem title.
*Covers.* RF-6.3, RF-6.4, RF-10.1, RF-10.2.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure rules: `ChooserCardKind` over every status pair, `ChooserOrder` with locale collations and ties, `MfaLineRule` over role × policy × factor, project-line flags from `GrantSet`. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | The projection query under the identity-predicate policy, the join restricted to enterable rows, `TenantChooserQuery` and `TenantEntryUseCase` through real adapters, audit rows, derivation outcomes. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of other identities, members, counts or content in responses. |
| Concurrency | JUnit 5 with real PostgreSQL | Entry racing a suspension, a revocation and a tenant state change. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page and card variants, server order preserved, handlers, shell action and guard, no background refetch, nothing in storage. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Card regions, order, announcements on refetch without focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1 | API contract, FE unit | `memberships.requiresSession401`; `chooserRoute.noSessionGoesToSignIn` | 7 |
| RF-1.2, LC-1, LC-25, LC-26, LC-29 | FE e2e (001, 003, 006, 008 stubs) | `chooser.reachedFromLandingLeaveSetupInvitationExitsAndReauth` | 7 |
| RF-1.3, LC-2, LC-3 | FE unit, e2e | `chooser.rendersWithOneCard`; `chooser.rendersEmptyBlockWithZeroCards` (no redirect) | 7 |
| RF-1.4 | FE unit, e2e | `shell.switchWorkspaceBesideSignOutOnEveryAuthenticatedRouteAndM04Menu` | 7 |
| RF-1.5, RNF-8, LC-28 | FE e2e | `switchWorkspace.dirtyGuardSaveDiscardStayBeforeChooser`; `switchWorkspace.guardSaveUnderOriginalScope` | 7 |
| RF-1.6 | BE integration | `listing.buildsNoTenantContext` | 9 |
| RF-2.1, RF-8.1–8.4 | FE unit | `chooserPage.rendersHeadingStatementsBlocksAndActions` | 1 |
| RF-2.2, LC-22 | FE e2e | `createWorkspace.offeredAlwaysAndCarriesNoTenant` | 7 |
| RF-2.3, LC-23 | FE e2e (007 stub) | `chooser.signOutEntryN03NoGuard` | 7 |
| RF-2.4, LC-24 | FE e2e | `languageSwitch.keepsCardsAndOrder` | 11 |
| RF-2.5 | FE unit | `chooserPage.rendersHelpAndNoPrivacyOrTerms` | 1 |
| RF-2.6, RF-4.6, RNF-1, LC-35 | API contract | `memberships.exposeOnlyOwnFactsAndNames` (no member, count, admin, source, run) | 4 |
| RF-3.1, RF-10.3 | BE integration | `projection.identityPredicateReturnsOwnRowsOnly` | 3 |
| RF-3.2, LC-4, LC-34 | BE integration | `projection.revokedNeverListedEvenWhenOnly` | 3 |
| RF-3.3 | BE integration | `projection.noRowForTenantWithoutMembership` | 3 |
| RF-3.4, LC-5–LC-7 | BE unit | `chooserCardKind.everyStatusPairTenantStateFirst` | 3 |
| RF-3.5, LC-19, LC-20 | BE unit, integration | `chooserOrder.kindRankThenLocaleCollationThenId` (EN, ES, FR; duplicate names) | 3 |
| RF-3.6 | API contract | `memberships.noPagingParameterFullList` (twenty rows) | 3 |
| RF-3.7, RF-7.2, LC-11 | FE unit | `useMemberships.noIntervalFocusOrReconnectRefetch`; `useMemberships.backgroundMarkerNeverSet` | 8 |
| RF-4.1 | FE unit | `membershipCard.enterableRendersAllFields` | 1 |
| RF-4.2, RNF-6, LC-17 | BE integration | `projection.projectLinesEqualLiveGrantsAlphabetical` | 5 |
| RF-4.3, LC-15, LC-16 | BE unit, FE unit | `projectLine.canRunOnlyFlag`; `membershipCard.noGrantsLineWithEnter`; `membershipCard.canRunOnlyLine` | 5 |
| RF-4.4, LC-13, LC-14 | BE unit, integration | `mfaLineRule.roleAndPolicyMatrix`; `listing.memberPolicyLineFromTenantPolicyPort` | 1 |
| RF-4.5, LC-12 | BE integration | `listing.notYetEnrolledSuffixFromFactorStatus` (pending enrollment not a factor) | 1 |
| RF-5.1 | FE unit | `membershipCard.suspendedTenantAsDrawnWithViewAccessStatus` | 4 |
| RF-5.2 | FE unit | `membershipCard.suspendedMembershipNoActionNoAdmin` | 4 |
| RF-5.3 | FE unit | `membershipCard.deletingTenantStateOnlyWithViewAccessStatus` | 4 |
| RF-5.4, LC-21 | FE e2e (014 stub) | `viewAccessStatus.opensN20WithoutEntryAndReturnsRefetching` | 7 |
| RF-5.5, RNF-5 | BE integration, API contract | `projection.noProjectNameForNonEnterableRows`; `memberships.nonEnterableCardsCarryNoProjectLines` | 4 |
| RF-5.6 | BE unit | `chooserCardKind.rowsClassifiedIndependently` | 3 |
| RF-6.1 | BE integration | `entry.runsTenantLevelDerivationWithRequestedScopeOnly` | 6 |
| RF-6.2 | FE unit, e2e | `enterTenant.advancesEpochClearsScopedCacheNavigatesToOverview` | 6 |
| RF-6.3, LC-8, LC-9, LC-10 | BE integration, FE unit | `entry.refusedAfterSuspensionRevocationOrTenantChangeAuditedOneCode`; `entryRefusedHandler.refetchesAndRendersNewState` | 6 |
| RF-6.4, LC-12, LC-13 | BE integration (003 stub), FE e2e | `entry.gateRaisesEnrollmentRequired`; `enter.gatedOwnerReachesWizardLeaveSetupReturnsToChooser` | 6 |
| RF-6.5, RNF-7, LC-18 | BE integration, FE e2e | `entry.writesNothingReplaysNothing`; `enter.afterGrantRevocationOverviewOmitsProject` | 9 |
| RF-6.6 | BE integration | `entry.changesNoRow` | 9 |
| RF-7.1, LC-21, LC-25 | FE unit | `useMemberships.refetchesOnEveryMount` | 8 |
| RF-7.3, RNF-10, LC-31 | FE unit | `storage.containsNothingFromChooser`; `reload.refetches` | 8 |
| RF-7.4, LC-30, LC-33 | FE e2e (006 stub) | `chooser.sessionEndedShowsR34ReauthReturnsFresh` | 8 |
| RF-8.5 | FE unit | `statements.identicalAcrossVariantsAndWidths` | 1 |
| RF-9.1–9.3, RNF-11, RNF-13 | FE e2e + axe, manual assistive | `a11y.chooser.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 11 |
| RF-10.1 | BE integration | `audit.entryRefusedPerCauseNoListContent` | 10 |
| RF-10.2 | BE integration, FE unit | `problems.safeCodesOnly`; `logs.noListOrGrant` | 10 |
| RF-10.4 | API contract | `memberships.nonEnterableCardsExposeStateAndRoleOnly` | 4 |
| RNF-2 | API contract | `chooserGrantsNoAuthority` (list alone enters nothing; entry needs derivation) | 9 |
| RNF-3 | BE integration | `failClosed.everyNonEnterableStateRefusesEntry` | 6 |
| RNF-4 | FE unit | `enter.neverUsesCachedListToDecide` | 8 |
| RNF-9 | Startup | `chooser.introducesNoConfigurationProperty` | — |
| RNF-12 | FE e2e | `allVariants.existInEnEsFr` | 11 |
| LC-27 | FE e2e (008 stub) | `acceptedInvitation.thenSwitchWorkspaceListsNewTenant` | 7 |
| LC-32 | Concurrency | `entry.twoTabsTwoTenantsIndependentDerivations` | 6 |
| US-1 … US-9 | FE e2e | journeys above | 1 |
| US-10 | all adversarial cases above | enumeration of tenants and members, stale entry, content of non-enterable tenants, authority from the page | 1 |

### 5.3 Not tested here
The authority matrix, the derivation and the permission version (plan 009); landing and sign-in (001); the enrollment wizard and the gate's internals (003); session issue, expiry and R34 (006); sign-out and the shell guard's internals (007); invitation exits (008); the suspended-tenant screen (014); workspace creation (012); the guarded switch's steps beyond the client half plan 001 already performs (016); the tenant overview (projects area). Their stubs return fixed answers; `MembershipProjectionPort`, `TenantPolicyPort`, `FactorStatusPort`, `RequirementReasonPort` and `SessionAssertionPort` are verified by recording stubs and, in the integration suite, by the owning contexts' test doubles.

### 5.4 Exit criteria
Specification §7 items 1–11 map to the "Proves criterion" column; item 12 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-36 identity-scoped membership listing | Plan 009 D10's identity-predicate policy and control-plane query, extended here with project names and grants for active-and-active rows (D2). | Replace the policy or the query with the decided mechanism; `TenantChooserQuery` and the read model are unchanged. |
| A-54 suspended and revoked meanings (decided) | `ChooserCardKind`: suspended membership listed without projects, revoked never listed (D1, D2). | None expected; a change to 009 Q3 changes the rule's table only. |
| A-35 persistence tooling | The projection query against the placeholder API. | Rename base types; no use case changes. |
| S-55 deleting-tenant card and N20 deleting wording (decided) | `DELETING_TENANT` card variant; "View access status" passes the tenant id and 014's screen chooses its wording. | Adjust the variant's copy; the kind rule is unchanged. |
| S-56 N03 card variants (decided) | `MfaLineRule` variants, "canRun only" and "No project grants yet" lines, suspended-membership card content, order, no paging (D1, D6, D9). | Adjust copy and layout; if a search or paging control is designed, add a bounded query parameter (a configured page size under P13) and a control — the read model is unchanged. |
| S-48 suspended-membership card (decided) | Variant with state, role, statement, no action. | Copy change only. |
| S-07 app-shell identity area | `SwitchWorkspaceAction` beside `SignOutAction` **[007]** in the footer block and the M04 menu (D7). | Move both actions into the designed user menu; the guard path and the route are unchanged. |
| S-02, S-03 responsive counterparts | Reflow of N03 under the desktop rules (D12). | Replace the breakpoint layout with the designed counterpart. |
| S-17 "Help" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |

**Cross-plan items raised by this plan.** (1) Plan 009 `MembershipProjectionPort`: the read model gains `projectLines` (project id, name, `GrantSet`) for rows whose membership and tenant are active — to be added to plan 009 §2.1 and §6 when it is next revised. (2) The `project` context publishes `ProjectNamesPort.namesOf(tenantId, projectIds)` — a names-only read under the tenant predicate, to be recorded by the first projects-area plan. (3) Specification 014's route accepts a tenant id as requested scope and renders the deleting wording from the tenant state (S-55).

The specification's derived rules are built, not assumed: revoked never listed (D1, D2), own facts only (D2, contract guards), language switch display-only (D9), sign-out without guard through plan 007's coordinator (`entry: 'N03'`), "Create workspace" with no tenant context (handoff row).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.6) | `MembershipsController` session check, routes, `SwitchWorkspaceAction`, `ProtectedBoundary` [006], shell guard [007] | D3, D7 | RF-1 rows |
| RF-2 (2.1–2.6) | `TenantChooserPage`, `ChooserCard` shape, handoffs, `SecurityStatements` [001] | D1, D9, D10 | RF-2 rows |
| RF-3 (3.1–3.7) | `TenantChooserQuery`, projection query, `ChooserCardKind`, `ChooserOrder`, `useMemberships` | D1, D2, D8, D9 | RF-3 rows |
| RF-4 (4.1–4.6) | `MembershipCard` enterable, project lines, `MfaLineRule`, `FactorStatusPort` [003], `TenantPolicyPort` [009] | D1, D2, D6 | RF-4 rows |
| RF-5 (5.1–5.6) | `MembershipCard` non-enterable variants, projection join restriction, 014 handoff | D1, D2, D11 | RF-5 rows |
| RF-6 (6.1–6.6) | `TenantEntryUseCase`, `DeriveAuthorityUseCase` [009], `EntryRefusedHandler`, `EnrollmentGateHandler` [003], epoch advance [001/006] | D3, D4, D5, D7 | RF-6 rows |
| RF-7 (7.1–7.4) | `useMemberships` options, no persistence, `ProtectedBoundary` [006] | D8, D10 | RF-7 rows |
| RF-8 (8.1–8.5) | `SecurityStatements` [001], statement blocks | D12 | RF-8 rows |
| RF-9 (9.1–9.3) | Breakpoint reflow, `shared/ui` | D12 | RF-9 row |
| RF-10 (10.1–10.4) | Audit adapter, problem codes, identity-predicate policy, read-model fields | D2, D4, D13 | RF-10 rows |
| RNF-1 | projection, contract guards | D2 | `memberships.exposeOnlyOwnFactsAndNames` |
| RNF-2 | server read model, derivation on entry | D1, D3, D5 | `chooserGrantsNoAuthority` |
| RNF-3 | derivation outcomes, kind rule | D3, D4 | `failClosed.*` |
| RNF-4 | refetch on mount and refusal | D8 | `enter.neverUsesCachedListToDecide` |
| RNF-5 | join restriction | D2 | `projection.noProjectNameForNonEnterableRows` |
| RNF-6 | `GrantSet` [009] lines | D1 | `projection.projectLinesEqualLiveGrantsAlphabetical` |
| RNF-7 | entry writes nothing | D3 | `entry.writesNothingReplaysNothing` |
| RNF-8 | shell guard before the chooser | D7 | `switchWorkspace.*` |
| RNF-9 | no configuration | D10 | startup |
| RNF-10 | no persistence, safe codes | D10, D13 | `storage.*`, `problems.*` |
| RNF-11, RNF-12, RNF-13 | `shared/ui`, `shared/i18n`, breakpoints | D9, D12 | a11y, language rows |
| LC-1 … LC-35 | per the §5.2 rows that name them | — | §5.2 |

Every EARS criterion RF-1.1 … RF-10.4 appears in §3; every RNF and LC appears in §5.2 or above. No element of §2 lacks a criterion, and no criterion lacks an element, a decision and a test.
