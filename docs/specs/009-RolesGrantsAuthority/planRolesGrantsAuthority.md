# Plan 009 — Roles, grants and the authority matrix

**Feature** The finite role model, the authority matrix, membership and tenant statuses, permission versions and revocation · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/009-RolesGrantsAuthority/specRolesGrantsAuthority.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-11 as the subject; F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-10, F-IDN-12, F-IDN-13, F-IDN-14, F-IDN-15, F-IDN-16, F-IDN-17, F-SET-05, F-X-01, F-X-05 at the boundary; F-X-02, F-X-03, F-X-04, F-X-06, F-X-07) · **Architecture source** Figma BE board `23:5587` (the `access` context owns "memberships, invitations, project grants, permission versions"; ownership ERD; fixed role / grant table), board 1 `23:5587` §04 (ACCESS ERD, authority table, "RLS and runtime authorization are independent, mandatory gates"), decision register `122:2` §3 and §4, FE board `46:7847` · **Builds on** plan 001 (Login: `MembershipProjectionPort`, `LandingRule`), plan 003 (Enrollment: `EnrollmentGate`, `FactorStatusPort`, `RequirementReasonPort`), plan 006 (Sessions: `SessionAssertionPort`, `access.denied-scope`, the denied-scope handler) and plan 007 (Logout: `MembershipExistsPort`); elements they own are tagged **[001]** / **[003]** / **[006]** / **[007]** and are extended, never duplicated. Specifications 008, 010, 011 and 012 consume the commands this plan publishes.

This plan is the HOW. It adds no behaviour the specification does not state. Plans 001, 003, 006 and 007 declared ports "published by `access`" whose implementation they left to this feature; this plan builds the `access` context core — memberships, grants, the matrix, the C10 derivation, the permission version — and implements those ports. The specification's seven decisions (Q1–Q7, registers A-53…A-58, S-47…S-49) are built as decided. Where the specification depends on items still open (A-11, A-36, A-39, A-42, S-06, S-14), the plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The matrix separates authoring, approval and launch as three action classes with three different grants; no derivation, command or event of this feature accepts a mapping, approves a bundle, launches or replays a Run. | RF-2.5–2.8, RF-2.14, RF-12.8 |
| **P2** AI proposes, humans authorise | No machine principal holds a role or a grant; the action classes APPROVE and LAUNCH can only be derived for a human membership. | RF-12.8 |
| **P3** The browser expresses intent, never authority | Tenant and project identifiers in a request are requested scope; C10 is derived on the server from the session, the membership and the grants; the FE authority store mirrors server facts for presentation only and names the missing grant on a hidden or disabled control. | RF-3.1, RF-3.2, RF-6.2, RF-10.7, RF-12.5, RNF-2 |
| **P4** Only the platform catalog is global | Memberships and grants are tenant rows; the only cross-tenant read is the identity-scoped membership listing (A-36), a narrow control-plane query. | RF-1.4, RF-4.3, RF-4.4 |
| **P6** Fail closed | A missing, suspended, revoked, deleting, stale or unknown fact yields `Deny`; the matrix has no default row; the version guard refuses on mismatch. | RF-3.6, RF-6.3, RNF-4 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL (composite keys, `FORCE RLS`, `SET LOCAL`), the existing outbox and jobs. FE: React 18, TypeScript, TanStack Query, i18next, Tailwind, axios. No policy engine, rules DSL or authorisation product is added. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the feature is the core of the existing hexagonal bounded context `access`; `tenant`, `identity`, `project`, `execution`, `audit` and `sharedsupport` are reached only through published ports; revocation effects travel by outbox events. FE: authority facts in `shared/security`, statements in `shared/ui`, no feature decides authority. | §2 |
| **P12** Two independent tenancy gates | `tenant_membership` and `project_grant` carry `tenant_id` in composite keys under `FORCE RLS` with `USING` and `WITH CHECK`; the runtime authorisation of the matrix is applied on top; neither replaces the other. | RF-1.4, RF-3, RNF-2, RNF-10 |
| **P13** Configured values are never defaulted | This feature owns no numeric value; the cancellation checkpoint cadence belongs to `execution` and audit retention to `audit`, both in the catalogue of A-11. | RNF-11 |
| **P14** Release bar | The statements of RF-10 are shared components with AA text and EN / ES / FR keys; the surfaces that render them (010, 011, 012, 013, settings) carry the evidence. | RF-10, RNF-12 |
| **F-X-01** C10 context | `DeriveAuthorityUseCase` builds `TrustedActorTenantContext` once per request after the session assertion **[006]**, and `AuthorityChecker` re-reads the membership under lock inside every effect transaction; jobs derive afresh; the runner gets an opaque context. | RF-3.1–3.7 |
| **F-X-02** Request integrity | Every command is idempotent on `(tenantId, operation, idempotencyKey, bodyHash)` with a recorded outcome; the version guard is the transactional stale-authorisation check. | RF-6.3, RF-8.6, RF-11.4 |
| **F-X-03** Safe problems | Two codes — `access.denied-scope` **[006]** and `access.forbidden-change` — with a correlation id and no reason class on the wire; the reason lives in the audit entry. | RF-3.8, RF-11.2 |
| **F-X-05** Audit log | Every change and every denied derivation appended through `AuditPort` in the same transaction; audit visibility follows the matrix's READ_AUDIT class. | RF-2.12, RF-11 |
| **F-X-07** Responsive surfaces | The statements are text components reflowing with their host surfaces. | RF-10.9 |
| **F-IDN-10 / 12 / 13 / 14** boundary | Plans 008, 010, 011 and 012 call `MembershipCommands` and `GrantCommands` of this plan; they add no second membership or grant model. | RF-8, RF-9 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access`, plus one consumer each in `execution`, `realtime` and `audit`

The context, its four layers (`api / application / domain / infrastructure`) and three published ports exist as declarations from plans 001, 003 and 007. This plan adds two aggregates, three pure rules, the C10 value, one derivation use case, one checker, one version guard, two command families, three queries, one event handler and the persistence with its RLS policies, and implements the declared ports.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `TenantRole`, `ProjectRole`, `GrantSet` | Closed enumerations `OWNER \| ADMIN \| MEMBER` and `AUTHOR \| REVIEWER \| VIEWER`; `GrantSet` = subset of project roles + `canRun` flag with `isEmpty()`, `includesRead()` (any role), `eligibleToRun()` (Author or Reviewer **and** canRun). No other value exists. | RF-1.1–1.3, RF-9.2, RF-9.3, LC-6, LC-8 |
| | `TenantMembership` | Aggregate: id, tenantId, identityId, `role`, `status` (`ACTIVE \| SUSPENDED \| REVOKED`), `permissionVersion`, timestamps, and its `ProjectGrant` children. Methods `changeRole`, `suspend`, `reinstate`, `revoke` (revokes every grant in the same change), `setGrants(projectId, GrantSet)` (empty set deletes the grant); every mutation advances `permissionVersion` exactly once and records the previous and new values for audit. Invariant: one membership per (tenant, identity). Pure; takes the clock as an argument. | RF-1.4, RF-4.1–4.5, RF-6.1, RF-6.6, RF-8.6, RF-9.1, RF-9.4, RF-9.7 |
| | `ProjectGrant` | Child of the membership: (tenantId, membershipId, projectId, `GrantSet`, `revokedAt`); unique per membership and project; never exists for a non-active membership. | RF-1.4, RF-9.1, RF-9.7, LC-9, LC-30 |
| | `ActionClass` | Closed enumeration of what the matrix decides: `TENANT_LIFECYCLE`, `TENANT_POLICY`, `TENANT_MEMBERS`, `PROJECT_READ`, `AUTHOR`, `ACCEPT_MAPPING`, `UPLOAD_ASSET`, `AUTHOR_MACRO`, `REVIEW`, `APPROVE`, `PUBLISH_MACRO_OR_ASSET`, `PUBLISH_TRUST`, `LAUNCH`, `WHOLE_PROJECT_LIFECYCLE`, `CONTENT_LIFECYCLE`, `READ_AUDIT`, `EXPORT_PROJECT`, `SUBSCRIBE`. Every protected operation of every context names one class. | RF-2, RF-3.3 |
| | `AuthorityMatrix` | Pure rule `(tenantRole, grantSet, tenantPolicy, actionClass, actionFacts) → Permit \| Deny(reasonClass)`: the rows of RF-2 verbatim — Owner / Admin administer, nobody reads without a grant, Author / Reviewer / Viewer as stated, LAUNCH = `eligibleToRun()` ∧ valid approval ∧ policy ∧ admission facts supplied by the caller, WHOLE_PROJECT_LIFECYCLE = Owner or Admin ∧ any project role (Q7), CONTENT_LIFECYCLE = Author, PUBLISH_TRUST = Reviewer ∧ (Owner or Admin), READ_AUDIT = any project role, EXPORT_PROJECT = any project role per project, APPROVE = Reviewer ∧ (¬self-authored ∨ ¬policy.requireDifferentApprover). No default branch. | RF-2.1–2.14, RF-12.2, RNF-1, LC-1, LC-6, LC-21–LC-25 |
| | `AdministrationRule` | Pure rule `(actorRole, actorIsTarget, targetRole, change) → Permit \| Deny`: Owner may change any role and create Owners; Admin may set MEMBER or ADMIN on non-Owner targets, never touch an Owner, never change own tenant role; nobody raises own tenant role; Owner lowers own only through transfer (010); Owner or Admin may set any grants including own; Member changes nothing. | RF-8.1–8.4, RF-12.1, LC-2–LC-5 |
| | `TrustedActorTenantContext` (C10) | Value: identityId, sessionId, tenantId, membershipId, tenantRole, membershipStatus, tenantStatus, projectScope (projectId + `GrantSet` or none), `permissionVersion`, policyRevision, enrollment-gate outcome **[003]**, correlation id. Immutable per request; never serialised to a client. | RF-3.1, RF-6.2, RF-12.5 |
| | `DerivationOutcome` | `Derived(C10) \| Denied(reasonClass)` with reason classes `NO_MEMBERSHIP`, `MEMBERSHIP_NOT_ACTIVE`, `TENANT_NOT_ACTIVE`, `MISSING_GRANT`, `STALE_VERSION`, `POLICY`, `ENROLLMENT_GATE`, `SCOPE_MISMATCH`. The reason never leaves the server. | RF-3.6, RF-3.8, RF-11.2 |
| `access.application` | `DeriveAuthorityUseCase` (request-scoped resolver) | For a request naming a tenant (and optionally a project): session facts from `SessionAssertionPort` **[006]**; membership by (tenant, identity) with status and version; tenant status from `TenantStatusPort`; grants for the project; policy revision from `TenantPolicyPort`; `EnrollmentGate` **[003]** with `FactorStatusPort` **[003]**; route-versus-body ownership check; yields `Derived` or `Denied`, audits every `Denied` with its reason class, and answers `access.denied-scope` (code only). | RF-3.1, RF-3.2, RF-3.5, RF-3.6, RF-3.8, RF-5.2, RF-11.2, LC-26–LC-28 |
| | `AuthorityChecker` (implements `AuthorityPort`, published) | `require(C10, actionClass, actionFacts)` called by every protected use case of every context at its effect boundary: re-reads the membership row inside the caller's transaction (`FOR SHARE`), compares `permissionVersion` and status, evaluates `AuthorityMatrix`, throws the safe problem on `Deny`. Async jobs call it with a fresh derivation for their scope. | RF-3.3, RF-3.4, RF-3.6, RF-6.3, RF-12.4, RNF-2, RNF-6 |
| | `PermissionVersionGuard` | The transactional check of FR-X-005 used by `AuthorityChecker` for sensitive writes: the write's transaction holds a share lock on the membership row and refuses if the version moved since derivation; writers of this feature take an exclusive lock and advance it. | RF-6.3, RF-6.4, RF-6.6, LC-10, LC-18 |
| | `MembershipCommands` (published: `CreateMembership`, `ChangeTenantRole`, `SuspendMembership`, `ReinstateMembership`, `RevokeMembership`) | Each: derive the actor's C10, apply `AdministrationRule`, lock the target membership `FOR UPDATE`, apply the aggregate method, advance the version, append the audit entry, enqueue `AuthorityChanged` in the outbox, record the idempotency outcome — one transaction. `CreateMembership` serves invitation acceptance (008 RF-6.1, which also replaces a `REVOKED` row) and the founding-Owner bootstrap (012); `ChangeTenantRole` to OWNER and the last-Owner guard are plan 010's `TransferOwnership`, which calls these commands under the ownership row lock. | RF-4.4, RF-4.5, RF-8.1–8.3, RF-8.5–8.7, RF-9.7, RF-11.1, RF-11.4, LC-12, LC-13, LC-16, LC-19, LC-20 |
| | `GrantCommands` (published: `SetProjectGrants`) | Derive actor C10; `AdministrationRule` (Owner or Admin; self allowed); require the target membership `ACTIVE` and the project in the tenant; lock the membership; `setGrants` (empty → delete); advance the version; audit; `AuthorityChanged`; idempotency outcome. Serves plan 011 and the bootstrap of plan 012 (which calls it in the same transaction as `CreateMembership`). | RF-8.4, RF-8.6, RF-9.1–9.6, RF-11.1, LC-4–LC-9, LC-20 |
| | `AuthorityChangedHandler` | Consumes the outbox event `AuthorityChanged{tenantId, membershipId, projectId?, kind}`: calls `SubscriptionRevocationPort` (close the member's streams in the affected scope) and `WorkRevocationPort` published by `execution` (refuse queued, not-admitted tickets of that membership as NOT EXECUTED; request bounded cancellation of running attempts). Denial itself needs no event: the next derivation sees the new facts. Also consumes `TenantStatusChanged` from the `tenant` context for every membership (014). | RF-5.3, RF-7.1–7.4, RF-7.7, LC-11, LC-14 |
| | `MyAuthorityQuery` | For the caller: tenant role, membership status, `permissionVersion`, the `GrantSet` per permitted project, the policy facts the surfaces state (requireDifferentApprover), the tenant status, and the enrollment-gate reason **[003]**. Feeds the FE authority store on tenant entry (016 step 4), after a denied scope (006 RF-5.1) and for RF-10.7. | RF-3.8, RF-6.4, RF-10.4, RF-10.7, RF-10.8 |
| | Published queries | `MembershipProjectionPort` **[001]** (identity-scoped listing with role, status, tenant status and the person's grants — A-36), `MembershipExistsPort` **[007]**, `RequirementReasonPort` **[003]** (reasons from role and policy), `TenantMembersQuery` (010), `ProjectGrantsQuery` (011) — read models only; never a row of another tenant. | RF-4.3, RF-4.4, RF-10.1, RF-10.2, RF-10.4 |
| | Ports | New: `MembershipRepository`, `ProjectGrantRepository`, `TenantStatusPort` (→ `tenant` context), `TenantPolicyPort` (→ policy record of F-SET-05), `ProjectExistsPort` (→ `project`), `SubscriptionRevocationPort` (→ realtime), `WorkRevocationPort` (→ `execution`), `AuthorityPort` (published), `MembershipCommands` and `GrantCommands` (published). Reused: `SessionAssertionPort` **[006]**, `EnrollmentGate` and `FactorStatusPort` **[003]**, `AuditPort`, `Clock` **[001]**, outbox and idempotency outcome **[001 D1 / 002]**. No port reaches an approval, run, intent or content table. | all |
| `access.infrastructure` | Persistence | `tenant_membership` (`tenant_id`, `id`, `identity_id`, `role`, `status`, `permission_version`, `created_at`, `updated_at`; primary key `(tenant_id, id)`; unique `(tenant_id, identity_id)`); `project_grant` (`tenant_id`, `membership_id`, `project_id`, `author`, `reviewer`, `viewer`, `can_run`, `revoked_at`; primary key `(tenant_id, membership_id, project_id)`; composite FKs to `tenant_membership (tenant_id, id)` and `project (tenant_id, id)`); `FORCE RLS` on both with `USING` and `WITH CHECK` on `tenant_id = current_setting('app.tenant_id')`, plus the identity-scoped policy of D10 for the control-plane listing; the runtime role non-owner, non-superuser, no `BYPASSRLS`. Access API and migration tool per A-35 (placeholders). | RF-1.4, RF-4.1, RF-5.1, RF-6.6, RNF-10 |
| | C10 request filter | After the session filter **[006]**: resolves the requested tenant and project from the route, runs `DeriveAuthorityUseCase`, stores C10 in the request context, and — in every transaction opened for the request — issues `SET LOCAL app.tenant_id` and `app.identity_id` from C10, never from a client value; requests without a tenant (identity-scoped endpoints of 001–007) skip derivation. | RF-3.1, RF-3.2, RF-3.7 |
| | Outbox and event adapters | `AuthorityChanged` and `TenantStatusChanged` through the existing outbox **[002]**; the `execution` adapter of `WorkRevocationPort` and the realtime adapter of `SubscriptionRevocationPort` are consumers in their own contexts. | RF-7.2–7.4 |
| | Audit adapter | Events: `MEMBERSHIP_CREATED{source: INVITATION \| BOOTSTRAP}`, `TENANT_ROLE_CHANGED`, `MEMBERSHIP_SUSPENDED`, `MEMBERSHIP_REINSTATED`, `MEMBERSHIP_REVOKED`, `GRANTS_SET`, `GRANT_REMOVED`, `AUTHORITY_DENIED{reasonClass, actionClass}`; payload: acting and affected membership and identity, tenant, project, previous value, new value, resulting version, correlation id, time; never content or a secret. | RF-11.1–11.3 |
| | Problem codes | `access.denied-scope` (403) **[006]** for every `Denied` derivation and every stale version; `access.forbidden-change` (403) for an administration change `AdministrationRule` refuses; both carry code, title and correlation id only. | RF-3.8, RF-6.4, RF-11.2 |
| `access.api` | `AuthorityController` — `/api/tenants/{tenantId}/me` | `GET authority?project=` → 200 the `MyAuthorityQuery` shape; 403 `access.denied-scope`. The administration endpoints live in plans 010 (`/members`) and 011 (`/projects/{id}/grants`) and call the published commands; the identity-scoped listing lives in plan 001's landing and plan 013's chooser. | RF-10.4, RF-10.7, RF-3.8 |

**Cross-context touch points.** `identity` — `SessionAssertionPort` **[006]**, `FactorStatusPort` **[003]**. `tenant` — `TenantStatusPort`, `TenantStatusChanged`. `project` — `ProjectExistsPort`; project deletion revokes grants through `GrantCommands` in its own transaction (LC-30). `execution` — `WorkRevocationPort` (consumer side); the matrix's LAUNCH facts (valid approval, admission) are supplied by `execution` when it calls `AuthorityPort`. `scenario`, `catalog`, `source`, `intelligence`, `reporting`, `integration` — call `AuthorityPort` with their action class at every effect boundary. `audit` — append-only; READ_AUDIT is decided here. Nothing reads or writes approval, run, intent or content tables.

**Data model note (P12).** Both tables are tenant rows under RLS; the identity-scoped membership listing is the single cross-tenant read, served by a dedicated policy and a narrow query (D10, A-36).

### 2.2 Front end — `shared/security`, `shared/ui`, `contracts/access`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| `shared/security` | `AuthorityStore` | The viewer's tenant role, membership status, `permissionVersion`, `GrantSet` per project, policy facts and enrollment reason, from `GET me/authority`; loaded at step 4 of the guarded switch (016), refreshed once by the denied-scope handler **[006]**; cleared with the scope epoch; never persisted. | RF-3.8, RF-6.4, RF-10.7 |
| | `useAuthority(actionClass, projectId?)` | Returns `{permitted, missingGrant}` by mirroring the matrix for presentation (the same table, generated from one shared definition so it cannot drift); used only to hide or disable a control and to name the missing grant; the server decides on every request. | RF-10.7, RF-10.8, RNF-2 |
| `contracts/access` | Types and guards | `MyAuthority` shape, the two problem codes, the closed enumerations of roles, grant sets and action classes shared with the BE definition. | RF-1.1–1.3 |
| `shared/ui` | `AuthorityStatements` | The statement blocks of RF-10.1–10.6 and RF-10.8 as one component with variants (`members`, `grants`, `policy`, `chooser`, `creation`, `approval`, `inertCanRun`), consumed by the surfaces of plans 010, 011, 012, 013, settings and approval; identical text on every role variant and width. | RF-10.1–10.6, RF-10.8, RF-10.9, S-06 |
| `shared/errors` | `DeniedScopeHandler` **[006]** | Unchanged: one `MyAuthorityQuery` refresh, then resume or the N28 state; `access.forbidden-change` renders the safe problem with the named reason from the i18n bundle ("you cannot change this role"), never from the server. | RF-3.8, RF-6.4, RF-8 |
| i18n | `access.{en,es,fr}` | Keys for every statement, role and grant name, the "canRun has no effect until a project role is held" text and the forbidden-change messages. | RF-10, RNF-12 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `TenantRole`; one `role` column | enum in `contracts/access` |
| RF-1.2 | `ProjectRole`, `GrantSet` | enum and type |
| RF-1.3 | No other type; `AuthorityMatrix` is the only meaning | — |
| RF-1.4 | Unique (tenant, identity); composite FKs; RLS | — |
| RF-1.5 | No membership or grant row can name the operator; operator records live outside `access` (A-01) | — |
| RF-1.6 | `CreateMembership` is reachable only from invitation acceptance and bootstrap | — |
| RF-2.1 | `AuthorityMatrix` TENANT_* rows for OWNER | — |
| RF-2.2 | TENANT_MEMBERS for ADMIN + `AdministrationRule` | — |
| RF-2.3 | TENANT_* rows deny MEMBER | — |
| RF-2.4 | PROJECT_READ / APPROVE / LAUNCH never read `tenantRole` | `useAuthority` mirrors |
| RF-2.5 | AUTHOR, ACCEPT_MAPPING, UPLOAD_ASSET, AUTHOR_MACRO rows | — |
| RF-2.6 | REVIEW, APPROVE (with self-authored fact and policy), PUBLISH_MACRO_OR_ASSET rows | — |
| RF-2.7 | PROJECT_READ only for VIEWER; all write classes deny | — |
| RF-2.8 | LAUNCH = `eligibleToRun()` ∧ caller-supplied approval, policy and admission facts | inert statement |
| RF-2.9 | WHOLE_PROJECT_LIFECYCLE row (Q7) | — |
| RF-2.10 | CONTENT_LIFECYCLE row | — |
| RF-2.11 | PUBLISH_TRUST row | — |
| RF-2.12 | READ_AUDIT row consumed by `audit` | — |
| RF-2.13 | EXPORT_PROJECT row per selected project consumed by `reporting` | — |
| RF-2.14 | APPROVE row reads `TenantPolicyPort.requireDifferentApprover`; approval context audits self-approval | approval statements |
| RF-3.1 | `DeriveAuthorityUseCase` inputs | — |
| RF-3.2 | Route-versus-body check; `SCOPE_MISMATCH`; code only | — |
| RF-3.3 | `AuthorityChecker` at every effect boundary of every context | — |
| RF-3.4 | Jobs derive afresh; run-scoped authority is `execution`'s | — |
| RF-3.5 | `EnrollmentGate` **[003]** inside derivation | `EnrollmentGateHandler` **[003]** |
| RF-3.6 | `Denied(reasonClass)`; no default row | — |
| RF-3.7 | C10 never crosses into Core or the runner envelope | — |
| RF-3.8 | `access.denied-scope` | `DeniedScopeHandler` **[006]** |
| RF-4.1 | `status`, `permission_version` columns | — |
| RF-4.2 | Derivation for ACTIVE | — |
| RF-4.3 | SUSPENDED → `MEMBERSHIP_NOT_ACTIVE`; grants untouched; `AuthorityChanged{SUSPENDED}` closes streams; projection shows "suspended" | chooser (013) |
| RF-4.4 | `revoke` cascades grants (`revokedAt`); projection omits REVOKED; 008 reactivates a REVOKED row in place with the invited scope only | — |
| RF-4.5 | Version advance + audit in the aggregate methods | — |
| RF-4.6 | No command of this feature touches content or identity tables | — |
| RF-5.1 | `TenantStatusPort` values | — |
| RF-5.2 | Derivation → `TENANT_NOT_ACTIVE` | N20 (014) |
| RF-5.3 | `TenantStatusChanged` → handler for every membership | — |
| RF-5.4 | Tenant status is read, never written into memberships | — |
| RF-6.1 | Aggregate: every mutation advances once | — |
| RF-6.2 | Version in C10 only; the `MyAuthority` response carries it for display and for the FE to detect refresh need, never as an input | `AuthorityStore` |
| RF-6.3 | `PermissionVersionGuard` under `FOR SHARE` | — |
| RF-6.4 | `access.denied-scope` on `STALE_VERSION` | handler refresh once |
| RF-6.5 | `TenantPolicyPort.revision` read at APPROVE / LAUNCH; not in `permissionVersion` | — |
| RF-6.6 | Monotonic counter under exclusive lock | — |
| RF-7.1 | Next derivation sees the new facts | — |
| RF-7.2 | `SubscriptionRevocationPort` | realtime disconnect |
| RF-7.3 | `WorkRevocationPort.refuseQueued` | — |
| RF-7.4 | `WorkRevocationPort.requestBoundedCancel` | — |
| RF-7.5 | No content deletion anywhere in this feature | — |
| RF-7.6 | — | `ProtectedBoundary` **[006]** grant check on drafts |
| RF-7.7 | Event scope = project or tenant; identity untouched | — |
| RF-8.1 | `AdministrationRule` OWNER rows; `TransferOwnership` (010) | — |
| RF-8.2 | ADMIN rows | hidden actions (010, S-49) |
| RF-8.3 | `actorIsTarget` rows | — |
| RF-8.4 | `SetProjectGrants` with self allowed for OWNER / ADMIN | grants editor (011) |
| RF-8.5 | 008 invitation carries ADMIN \| MEMBER; `CreateMembership` refuses OWNER from invitation | — |
| RF-8.6 | One transaction: lock, mutate, version, audit, outbox, outcome | — |
| RF-8.7 | `TransferOwnership` (010) under the ownership row lock calls `ChangeTenantRole` | — |
| RF-9.1 | `ProjectGrant` rows; no inference anywhere | — |
| RF-9.2 | `GrantSet` accepts any subset | — |
| RF-9.3 | `can_run` stored; `eligibleToRun()` computed | inert statement |
| RF-9.4 | `setGrants(empty)` deletes | — |
| RF-9.5 | Bootstrap (012) calls `CreateMembership` + `SetProjectGrants` in one transaction | — |
| RF-9.6 | Project creation calls `SetProjectGrants` only for ticked grants | — |
| RF-9.7 | `setGrants` requires ACTIVE; 008 reactivates the membership first (`reactivateFromInvitation`), then sets the invited grant | — |
| RF-10.1–10.6 | — | `AuthorityStatements` variants in 010, 011, settings, 013, 012, approval |
| RF-10.7 | `MyAuthorityQuery` | `useAuthority` hide / disable with named grant |
| RF-10.8 | — | `inertCanRun` variant |
| RF-10.9 | — | one component, one bundle |
| RF-11.1 | Audit adapter events | — |
| RF-11.2 | `AUTHORITY_DENIED{reasonClass, actionClass}`; wire carries code only | — |
| RF-11.3 | Payload shape; READ_AUDIT row | — |
| RF-11.4 | Idempotency outcome per command | — |
| RF-12.1 | `AdministrationRule`; 008 scope bound | — |
| RF-12.2 | Matrix rows never cross tenant and project columns | — |
| RF-12.3 | `revoke` cascade; `setGrants` requires ACTIVE | — |
| RF-12.4 | `PermissionVersionGuard` | — |
| RF-12.5 | C10 from server facts; request-scoped; never serialised | store is presentation only |
| RF-12.6 | No operator principal in `access` | — |
| RF-12.7 | `TenantPolicyPort` exposes tightening flags only (settings spec) | — |
| RF-12.8 | APPROVE / LAUNCH require a human membership C10; no service principal in this release | — |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · One bounded context owns memberships, grants, versions and derivation; everyone else consumes published ports.**
*Why.* The BE section places memberships, invitations, project grants and permission versions in `access`; plans 001, 003, 006 and 007 already consume ports "published by `access`". One owner means one state machine, one version counter and one matrix; a second copy of any of them would drift.
*Discarded.* Authorisation checks inside each context's use cases with their own reads of membership tables — foreign repositories and divergent rules. A separate "authorisation service" deployment — a microservice per concern, contrary to P11.
*Covers.* RF-1.4, RF-3.3, RF-8.6, RNF-1, RNF-3.

**D2 · The matrix is one pure, total function over closed enumerations, and every protected operation names its action class.**
*Why.* FR-IDN-091 is to be implemented exactly; a table-shaped rule with enumerated action classes can be tested exhaustively (role × grant set × class) and reviewed against the specification row by row, which scattered conditionals cannot. Totality without a default branch is how P6 becomes a compile-time property: an unlisted combination does not exist.
*Discarded.* Permission strings per endpoint — unbounded, unreviewable. A rules DSL or policy engine — outside P10 and a second place for the truth. Deriving permissions from role names in the FE — P3.
*Covers.* RF-2.1–2.14, RF-12.2, RNF-1, RNF-9.

**D3 · C10 is derived once per request after the session assertion, and re-asserted inside every effect transaction by `AuthorityChecker` with a fresh read under lock.**
*Why.* FR-X-001 wants the context before any effect; FR-X-004 wants a recheck at every effect boundary. Deriving once gives every use case the same facts; re-reading the membership row under `FOR SHARE` inside the writer's transaction is what makes the recheck transactional (D4) rather than a second cached look-up. Jobs and detached transactions derive afresh for their own scope (FR-X-003).
*Discarded.* Derive once and trust it for the request — a role revoked mid-request still commits. Recheck by re-running the whole derivation per effect — needless reads of session and tenant; the membership row is the fact that changes.
*Covers.* RF-3.1, RF-3.3, RF-3.4, RF-3.6, RNF-2, RNF-4.

**D4 · `permissionVersion` is a per-membership counter advanced under an exclusive lock by writers and checked under a share lock by sensitive writes.**
*Why.* Decision Q4 (A-55): the version belongs to the membership; a share lock on the membership row during a sensitive write serialises it against any concurrent role, status or grant change (which takes the exclusive lock), so a stale write cannot commit — the race is decided by PostgreSQL, not by timing. The counter is monotonic by construction.
*Discarded.* A tenant-wide version — every administration action invalidates every member's writes (Q4's rejected option). Timestamps — not monotonic across clocks. Checking the version without a lock — the race window FR-IDN-095 forbids for owners applies here too.
*Covers.* RF-6.1–6.6, RF-12.4, RNF-5, RNF-6, LC-10, LC-18.

**D5 · A grant is one row per membership and project with four flags; an empty set deletes the row; inert `canRun` is computed, never stored.**
*Why.* Decision Q2 (A-53): any subset is valid and `canRun` is independent; four booleans with a composite primary key give uniqueness, make "no role and no canRun = no grant" a delete, and let `eligibleToRun()` decide launch eligibility in the matrix rather than by a stored derived flag that could go stale when a role is removed (LC-7).
*Discarded.* One row per role — four rows to keep consistent for one member. A stored "effective canRun" — stale after a role change. Refusing `canRun` without a role (Q2's rejected option) — cascade semantics on role removal.
*Covers.* RF-1.2, RF-9.1–9.4, RF-2.8, LC-6–LC-9.

**D6 · Membership status is a state machine in the aggregate; revocation cascades to grants in the same transaction; suspension retains them.**
*Why.* Decision Q3 (A-54): suspended keeps, revoked ends. Putting the cascade in `revoke` makes "no grant survives its membership" an invariant of the aggregate (RF-12.3), and keeping suspended grants untouched makes reinstatement a status change only (LC-12). A re-invitation reactivates the revoked row in place with a new `permissionVersion` and `setGrants` for the invited project only (008 RF-6.2, plan 008 D6); under the unique (tenant, identity) key no second row can exist, and its other revoked grant rows stay revoked.
*Discarded.* Deleting grants on suspension — reinstatement would need a copy. Leaving grants live on revocation — "old privileges return", which the register forbids.
*Covers.* RF-4.1–4.6, RF-9.7, RF-12.3, LC-12, LC-13.

**D7 · Administration authority is a second pure rule, applied by the commands; the last-Owner guard stays in plan 010 under the ownership row lock.**
*Why.* Decision Q6 (A-57): who may change whom is a matrix of its own (actor role, self or other, target role, change); a pure rule is testable like D2 and shared by the members list, the grants editor, invitation issue and the bootstrap. The last-active-Owner protection needs a count under a lock on the tenant ownership row (FR-IDN-095), which belongs with ownership transfer in 010; `ChangeTenantRole` to or from OWNER is reachable only through that command.
*Discarded.* Checks inside each surface's controller — four copies. Folding the last-Owner guard into `ChangeTenantRole` — the guard needs the ownership row lock that 010 owns, and would duplicate the transfer rules.
*Covers.* RF-8.1–8.7, RF-12.1, LC-2–LC-5, LC-16, LC-19.

**D8 · Revocation denies synchronously through derivation and propagates its side effects asynchronously through an outbox event.**
*Why.* Decision Q5 (A-56): denial from the next request needs nothing but the new facts (D3); closing streams and refusing or cancelling work are effects in other contexts with their own checkpoints, so they travel as `AuthorityChanged` through the existing outbox (plan 002 D1) and are applied by `execution` and realtime at their next bounded checkpoint — never as a rollback, never marking anything failed (P8).
*Discarded.* Synchronous calls into `execution` from the membership command — couples the administration transaction to runner state and violates the dependency direction (access → project → execution, never back). Killing attempts — a fabricated outcome.
*Covers.* RF-7.1–7.7, RF-5.3, RNF-7, RNF-8, LC-11, LC-14.

**D9 · Tenant status and policy revision are read through ports at derivation time, never copied into memberships.**
*Why.* RF-5.4 and RF-6.5: the tenant's state and the policy's revision are owned elsewhere; reading them per derivation gives "next request" semantics (specification 003 Q12 style) without synchronisation, and keeps `permissionVersion` about the membership only (Q4).
*Discarded.* Denormalising tenant status onto memberships — must be kept in sync on every tenant transition. Bumping all versions on a policy change — Q4's rejected option.
*Covers.* RF-5.1–5.4, RF-6.5, RF-2.14, LC-14, LC-15, LC-17.

**D10 · RLS carries a tenant predicate on both tables and one additional identity predicate for the control-plane membership listing (A-36 assumption).**
*Why.* P12: RLS and runtime authorisation are independent gates. The listing that the landing rule **[001]**, the chooser (013) and `MembershipExistsPort` **[007]** need crosses tenants for one identity; a dedicated policy `identity_id = current_setting('app.identity_id')` on `tenant_membership`, enabled only by the narrow query that sets that variable with `SET LOCAL`, serves it without an RLS exception for anything else.
*Discarded.* A `BYPASSRLS` role for the listing — a hole every other query could use. A denormalised per-identity table — a second copy of membership state.
*Covers.* RF-1.4, RF-4.3, RF-4.4, RNF-10, A-36.

**D11 · The enrollment gate is evaluated inside the derivation, before the matrix.**
*Why.* Specification 003 (A-42) places the gate in the C10 derivation; a membership under the gate yields no authority whatever its grants (RF-3.5), so the gate is a derivation fact, not a matrix row.
*Discarded.* A matrix column for enrollment — would need every row to repeat it.
*Covers.* RF-3.5, LC-26.

**D12 · The FE authority store mirrors the matrix for presentation from one shared definition and never decides.**
*Why.* RF-10.7 requires a hidden or disabled control to name the missing grant, which needs the viewer's role and grants client-side; generating the FE mirror and the BE matrix from one shared table (contracts) keeps them from drifting, and P3 is preserved because every request is derived on the server regardless of what the FE showed.
*Discarded.* Asking the server per control — chatty and still presentation. A hand-written FE copy of the rules — drift.
*Covers.* RF-10.7, RF-10.8, RF-12.5, RNF-2.

**D13 · The authority statements are one shared component with variants, keyed in one bundle.**
*Why.* RF-10.9 and S-06: the same promises must read identically on every surface, role variant and width; one component consumed by plans 010, 011, 012, 013, settings and approval cannot drift.
*Discarded.* Per-surface copy — drift.
*Covers.* RF-10.1–10.6, RF-10.8, RF-10.9, RNF-12.

**D14 · Every denied derivation is audited with its reason class; the wire carries the code only.**
*Why.* RF-11.2 and RF-3.2: the requester must not learn whether a tenant or project exists or why they were refused, while the operator must be able to reconstruct it; the reason class in the audit entry and a bare code on the wire give both. Volume is bounded by the edge rate limits of A-20.
*Discarded.* A reason on the wire — enumeration. Not auditing denials — blind to probing.
*Covers.* RF-3.2, RF-3.8, RF-11.2, LC-27, LC-28.

**D15 · Commands are idempotent on the tenant-scoped key with a recorded outcome, in the same transaction as the change.**
*Why.* F-X-02 and RF-11.4: a repeated "Save project grants" or role change must yield one change and the same answer; the outcome record of plan 001 D1 is reused.
*Discarded.* Natural idempotency by comparing states — a second change with the same body after an intervening change would be misread.
*Covers.* RF-8.6, RF-11.4, LC-18.

**D16 · Errors are RFC 9457 problem details with two safe codes and a correlation id [001 D12].**
*Why.* F-X-03; `access.denied-scope` is already the code plan 006's handler reacts to; `access.forbidden-change` distinguishes an administration refusal so the FE can show the named limit from its own bundle.
*Discarded.* Reason-specific codes — enumeration (D14).
*Covers.* RF-3.8, RF-6.4, RF-8.2, RF-8.3.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**) | Pure rules: `AuthorityMatrix` exhaustively over role × grant set × action class × policy; `AdministrationRule` over actor × target × change; `GrantSet`; `TenantMembership` state machine and version arithmetic with a fixed clock. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A; Testcontainers **test-only**) | Derivation through real adapters; commands with locks, cascades, versions, audit and outbox rows; RLS policies under a non-privileged role; the identity-scoped listing; event handling into recording consumers. |
| Module boundary | ArchUnit (**test-only**) | `access.application` depends on no Spring or persistence class; no context imports another's repository; dependency direction identity → access → project → execution holds. |
| API contract | Spring MockMvc (**test-only**) | `GET me/authority` shape; the two problem codes; no reason class, tenant existence or content on the wire. |
| Concurrency | JUnit 5 with real PostgreSQL | Grant change racing a sensitive write; two administrators on one grant; revoke racing a launch; suspension racing a subscription. |
| Security | Spring Boot test with the runtime role, real PostgreSQL | Cross-tenant read / write / FK attacks on both tables; pooled-context contamination; the identity policy reachable only through the narrow query. |
| FE unit | Vitest + Testing Library (**test-only**) | `useAuthority` mirror parity with the shared table; hide / disable with the named grant; statement component variants; store cleared with the epoch; nothing in storage. |
| FE e2e + accessibility | Playwright + axe (**test-only**) | Role-variant journeys on the surfaces of 010, 011, 012, 013 with the BE doubles; AA on the statements. |
| Manual assistive | NVDA (Chrome, Firefox), VoiceOver (Safari macOS, iOS) — pinned versions | Disabled controls announce the missing grant; statements readable in order. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1–1.3, RNF-9 | BE unit | `roleModel.closedEnumerationsOnly`; `grantSet.anySubsetAccepted` | 3 |
| RF-1.4, RNF-10 | Security | `membership.uniquePerTenantIdentity`; `grant.compositeKeysRejectCrossTenant`; `rls.crossTenantReadWriteFkAttacksFail` | 12 |
| RF-1.5, RF-12.6 | BE integration | `operator.cannotHoldMembershipOrGrant` | 3 |
| RF-1.6 | BE integration | `membership.createdOnlyByInvitationOrBootstrap` | 3 |
| RF-2.1–2.14, RNF-1, LC-1, LC-6, LC-21–LC-25 | BE unit | `authorityMatrix.exhaustiveTable` (every role × grant set × action class × policy, compared with the specification table); `matrix.ownerWithoutGrantReadsNothing`; `matrix.canRunInertWithoutRole`; `matrix.reviewerOnlyOwnerLifecycle`; `matrix.memberNeverWholeProjectLifecycle`; `matrix.trustNeedsTwoAuthorities`; `matrix.auditFollowsProjectAuthority`; `matrix.exportPerProject`; `matrix.selfApprovalPerPolicy` | 3 |
| RF-3.1, RF-3.6, RNF-2, RNF-4 | BE integration | `derive.fromCurrentFactsOnly`; `derive.absentStaleSuspendedRevokedDeletingDeny` | 4 |
| RF-3.2, LC-27, LC-28 | API contract, BE integration | `derive.requestedScopeNeverAuthority`; `derive.routeBodyMismatchDenied`; `derive.unknownTenantUndisclosed` | 4 |
| RF-3.3, RF-3.4 | BE integration | `checker.calledAtEveryEffectBoundary` (one case per action class through a stub use case); `jobs.deriveAfreshPerScope` | 4 |
| RF-3.5, LC-26 | BE integration | `derive.enrollmentGateWithholdsAuthority` (plan 003 stub) | 4 |
| RF-3.7 | BE integration | `runnerEnvelope.carriesNoRoleOrGrant` | 4 |
| RF-3.8, RF-6.4 | API contract, FE unit | `deniedScope.codeOnlyWithCorrelationId`; `handler.refreshesOnceThenResumesOrDenies` **[006]** | 4, 6 |
| RF-4.1–4.6, LC-12, LC-13 | BE unit, integration | `membership.stateMachine`; `suspend.noAuthorityGrantsRetained`; `reinstate.grantsUnchangedVersionAdvanced`; `revoke.cascadesGrantsAndEndsListing`; `reinvite.createsNewMembershipNoRestore` (with 008's acceptance stub) | 5 |
| RF-5.1–5.4, LC-14, LC-15 | BE integration | `tenantStatus.suspendedDeniesAllKeepsMemberships`; `tenantStatusChanged.refreshesAndRevokesWork`; `tenantActive.restoresExactMembershipFacts` | 5 |
| RF-6.1, RF-6.6 | BE unit, integration | `version.advancesOncePerChange`; `version.monotonicUnderConcurrency` | 6 |
| RF-6.2, RF-12.5 | API contract | `c10.neverSerialisedToClient`; `version.clientValueIgnored` | 4 |
| RF-6.3, RF-12.4, LC-10 | Concurrency | `staleVersion.sensitiveWriteRefusedTransactionally` | 6 |
| RF-6.5, LC-17 | BE integration | `policyChange.advancesPolicyRevisionNotMembershipVersion`; `approve.selfApprovalRefusedUnderTightenedPolicy` | 6 |
| RF-7.1, RF-7.7 | BE integration | `revoke.deniedFromNextRequest`; `revoke.scopeProjectOrTenantOnly` | 7 |
| RF-7.2 | BE integration | `authorityChanged.closesSubscriptions` (recording realtime double) | 7 |
| RF-7.3, RF-7.4, RNF-8, LC-11 | BE integration | `authorityChanged.refusesQueuedAsNotExecuted`; `authorityChanged.requestsBoundedCancelKeepsFacts` (recording execution double) | 7 |
| RF-7.5 | BE integration | `revoke.leavesAuthoredContentAndProvenance` | 7 |
| RF-7.6 | FE unit | `drafts.purgedWhenGrantNoLongerPermits` **[006]** | 7 |
| RF-8.1–8.3, RF-12.1, LC-2, LC-3 | BE unit, integration | `administrationRule.exhaustiveTable`; `admin.cannotTouchOwnerOrOwnRole`; `nobody.raisesOwnTenantRole` | 8 |
| RF-8.4, LC-4, LC-5 | BE integration | `grants.ownerSetsOwnAudited`; `grants.memberCannotChangeAny` | 8 |
| RF-8.5, LC-19 | BE integration | `createMembership.refusesOwnerFromInvitation`; `invitation.cannotExceedInviter` (008 stub) | 8 |
| RF-8.6, RF-11.4, LC-18 | Concurrency | `command.lockVersionAuditOutboxAtomic`; `command.idempotentOutcome`; `grants.twoAdministratorsSerialised` | 8 |
| RF-8.7, LC-16 | BE integration | `transferOwnership.guardOnEveryPath` (plan 010 stub under the ownership lock) | 8 |
| RF-9.1–9.4, LC-7–LC-9 | BE integration | `grant.explicitRecordOnly`; `grant.emptySetDeletesRow`; `grant.canRunStoredEligibilityComputed`; `authorRemoved.canRunBecomesInert` | 9 |
| RF-9.5, RF-9.6, LC-20 | BE integration | `bootstrap.membershipAndTickedGrantsOneTransaction` (012 stub); `projectCreation.impliesNoGrant` | 9 |
| RF-9.7 | BE integration | `grant.refusedForNonActiveMembership` | 9 |
| RF-10.1–10.6, RF-10.9 | FE unit, e2e + axe | `statements.variantsRenderSpecText`; `statements.identicalAcrossRoleVariantsAndWidths` | 10 |
| RF-10.7 | FE unit | `useAuthority.parityWithSharedTable`; `control.hiddenOrDisabledNamesMissingGrant` | 10 |
| RF-10.8, LC-6 | FE unit | `grantsEditor.inertCanRunStatement` | 10 |
| RF-11.1, RF-11.3 | BE integration | `audit.entryPerChange`; `audit.containsNoContentOrSecret`; `audit.readFollowsProjectAuthority` | 11 |
| RF-11.2 | BE integration | `audit.deniedDerivationWithReasonClass`; `wire.carriesNoReason` | 11 |
| RF-12.2, RF-12.3 | BE unit, integration | `matrix.noCrossScopeImplication`; `grant.neverSurvivesRevocation` | 3, 5 |
| RF-12.7 | BE integration | `policyPort.exposesTighteningOnly` | 3 |
| RF-12.8 | BE unit | `matrix.approveAndLaunchRequireHumanMembership` | 3 |
| RNF-3 | BE unit | `matrix.tenantRoleImpliesNoProjectAccess` | 3 |
| RNF-5, RNF-7 | Concurrency | `command.partialChangeNeverObservable`; `revocation.nextRequestSeesNewFacts` | 7, 8 |
| RNF-10 | Security, Concurrency | area A and B suites above | 12 |
| RNF-11 | Startup | no numeric value owned here; `execution` and `audit` startup suites cover theirs | — |
| RNF-12 | FE e2e + axe, manual | `a11y.authorityStatements.{en,es,fr}.{desktop,tablet,mobile}`; manual checklist | 10 |
| LC-29 | BE integration | `identityRestricted.derivationYieldsNothing` (A-27 stub) | 4 |
| LC-30 | BE integration | `projectDeleted.grantsEndAudited` | 9 |
| US-12 | all adversarial cases above | tenant-role read, author-approve, self-escalation, revived grant, stale write, operator-in-tenant, browser-supplied id suites | 1 |

### 5.3 Not tested here
Invitation acceptance (008), the members list and ownership transfer with the last-Owner lock (010), the grants editor surface (011), the bootstrap transaction (012), the chooser (013), the suspended-tenant screen (014), policy editing (settings), the approval decision and the launch (approval and execution specifications), the runner envelope beyond its absence of roles. Their stubs return fixed answers; `AuthorityPort`, `MembershipCommands`, `GrantCommands` and the published queries are exercised here as this feature's own surface.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q2–Q7 were recorded in Figma on 27 October 2026 as **A-53…A-58**, **S-47**, **S-48** and **S-49**; this plan implements each as decided (grant shape D5, membership states D6, version D4, revocation D8, administration D7, lifecycle row D2). The items below remain open.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-11 configuration values | This feature owns no numeric value; checkpoint cadence (`execution`) and audit retention (`audit`) are theirs. | None here. |
| A-36 identity-scoped membership listing | A dedicated identity-predicate RLS policy on `tenant_membership`, reachable only through the narrow control-plane query that sets `app.identity_id` (D10). | Replace the policy with the decided mechanism; the query's contract is unchanged. |
| A-39 revocation scope | Membership revocation never touches sessions; run-scoped authority is `execution`'s and is never the membership (D8). | None here. |
| A-42 enrollment gate timing | Gate evaluated at every derivation (D11), as decided for 003. | None here. |
| A-27 identity statuses | A restricted or erasing identity yields no derivation (`SessionAssertionPort` **[006]** already refuses). | Change the predicate only. |
| A-35 persistence tooling | Adapters, migrations and RLS policies against the placeholder API and tool. | Rename base types and migration format. |
| S-06 role variants | `AuthorityStatements` is one component; the surfaces render it on every variant. | None here. |
| S-14 templates | No message is sent by this feature. | None here. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.6) | `TenantRole`, `ProjectRole`, `GrantSet`, `TenantMembership`, composite keys, RLS | D1, D5, D10 | RF-1 rows |
| RF-2 (2.1–2.14) | `ActionClass`, `AuthorityMatrix`, `TenantPolicyPort` | D2, D9 | `authorityMatrix.exhaustiveTable` and its named cases |
| RF-3 (3.1–3.8) | `DeriveAuthorityUseCase`, `AuthorityChecker`, C10 filter, `EnrollmentGate` **[003]**, problem codes | D3, D11, D14, D16 | RF-3 rows |
| RF-4 (4.1–4.6) | `TenantMembership` state machine, `revoke` cascade, projections | D6 | RF-4 rows |
| RF-5 (5.1–5.4) | `TenantStatusPort`, `TenantStatusChanged` handler | D9 | RF-5 rows |
| RF-6 (6.1–6.6) | `permissionVersion`, `PermissionVersionGuard`, `TenantPolicyPort.revision` | D4, D9 | RF-6 rows |
| RF-7 (7.1–7.7) | `AuthorityChangedHandler`, `WorkRevocationPort`, `SubscriptionRevocationPort` | D8 | RF-7 rows |
| RF-8 (8.1–8.7) | `AdministrationRule`, `MembershipCommands`, `GrantCommands`, 010's transfer | D7, D15 | RF-8 rows |
| RF-9 (9.1–9.7) | `ProjectGrant`, `GrantSet`, `SetProjectGrants`, bootstrap calls | D5, D6 | RF-9 rows |
| RF-10 (10.1–10.9) | `AuthorityStatements`, `useAuthority`, `MyAuthorityQuery` | D12, D13 | RF-10 rows |
| RF-11 (11.1–11.4) | Audit adapter, `AUTHORITY_DENIED`, idempotency outcomes | D14, D15 | RF-11 rows |
| RF-12 (12.1–12.8) | rules, cascade, guard, C10 scope, no operator, policy port, human-only APPROVE / LAUNCH | D2, D4, D6, D7 | RF-12 rows |
| RNF-1 | `AuthorityMatrix` total table | D2 | exhaustive test |
| RNF-2 | derivation + checker | D3, D12 | RF-3 rows |
| RNF-3 | matrix columns | D2 | `matrix.tenantRoleImpliesNoProjectAccess` |
| RNF-4 | no default branch; `Denied` | D2, D3 | `derive.absent…Deny` |
| RNF-5, RNF-6 | one transaction; locked counter | D4, D15 | concurrency rows |
| RNF-7, RNF-8 | synchronous denial; bounded effects | D8 | RF-7 rows |
| RNF-9 | closed enumerations; no operator | D2 | RF-1 rows |
| RNF-10 | RLS, concurrency and security suites | D4, D10 | security rows |
| RNF-11 | no numeric value here | — | — |
| RNF-12 | shared statements, i18n | D13 | a11y row |
| LC-1 … LC-30 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30) |

Self-check: every RF criterion (86), every RNF (12) and every limit case (30) of `specRolesGrantsAuthority.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
