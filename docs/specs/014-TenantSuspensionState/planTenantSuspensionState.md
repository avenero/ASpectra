# Plan 014 — Tenant suspension state

**Feature** The suspended state of a tenant: what suspension freezes, what every member sees, what remains possible, and reinstatement · **Plan version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — aligned with specification v0.1 (decisions Q1–Q6, derived rules D-a…D-d) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/014-TenantSuspensionState/specTenantSuspensionState.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-16 as the subject; F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-13, F-IDN-15, F-IDN-17, F-IDN-18, F-SET-03, F-RUN-03, F-RUN-07, F-RUN-08, F-RUN-13 and F-PRJ-07 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG; contexts `tenant`, `access`, `execution`, `integration`, `source`, `intelligence`, `audit`), FE board `46:7847` (feature-modular SPA DAG), Failure board `3:2192` ("Tenant suspended / membership revoked") and decision register `122:2` (§5: the `access` context owns memberships, invitations, project grants and permission versions; §14: A-62, A-56) · **Builds on** plan 001 (Login: `LandingRouter`, `AuditPort`, `CompleteSignInUseCase`), plan 002 (outbox, `SecurityNotificationPort`, `TransactionalEmailPort`), plan 003 (`SecuritySettingsPage`), plans 004 and 005 (the shape of a platform act: an environment-only audited command), plan 006 (Sessions: `SessionAssertionPort`, `DeniedScopeHandler`, `ProtectedBoundary`, scope epoch), plan 007 (Logout: `SignOutCoordinator` with the `N20` entry), plan 008 (`IdentityFactsPort`; an invitation to a non-active tenant cannot be accepted), plan 009 (Roles and grants: `TenantStatusPort`, `DeriveAuthorityUseCase`, `AuthorityChangedHandler`, `SubscriptionRevocationPort`, `WorkRevocationPort`, `MembershipRepository`, the identity-scoped listing), plan 010 (Members: invitation rules refuse a non-active tenant; the app shell sends a suspended tenant to N20), plan 012 (Workspace creation: `TenantCommands.createTenant`) and plan 013 (Tenant chooser: suspended-tenant and deleting-tenant cards, "View access status", `useEnterTenant`); elements they own are tagged **[001]** … **[013]** and are extended, never duplicated. Specifications 015 (leave and erasure) and 016 (guarded switch) are written without a plan yet, and the execution, delivery, import and AI specifications do not exist yet; this plan reaches them only through the contracts it names (§6), which match specification 015 RF-4.1, RF-2.4, RF-5.5 and D-a and specification 016 RF-7.4.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's six decisions (Q1–Q6) and four derived rules (D-a…D-d) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-62, A-56, A-54, A-39, A-36, A-30, A-01; S-57, S-55, S-48, S-14, S-02, S-17; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | Reinstatement resumes, replays and re-admits nothing: the `TenantStatusChanged{→ ACTIVE}` event has no consumer that starts work, tickets ended NOT EXECUTED stay so, held delivery items wait for a human retry, and a new Run needs a new human command under a valid approval. | RF-6.3, RNF-3 |
| **P3** The browser expresses intent, never authority | N20's content (state, role, membership state, whether an alternate entry exists) is a server-built read model rendered as given; the screen sends a tenant id as requested scope and nothing else. Whether a member may do anything in the tenant is decided by the C10 derivation **[009]**, never by the client. | RF-3.2, RF-3.4, RF-3.8, RNF-5 |
| **P6** Fail closed | A tenant status that is not `ACTIVE` — suspended, deleting or unknown — yields `Denied(TENANT_NOT_ACTIVE)` **[009]**; a job checkpoint that cannot read the status stops; `TenantFence` fences anything not `ACTIVE`; no status is cached. | RF-1.2, RF-1.6, RNF-1, RNF-9 |
| **P8** Evidence is never rewritten | Suspension writes no row of a run, attempt, fact, verdict or delivery receipt; tickets end NOT EXECUTED with cause `TENANT_SUSPENDED`, never failed; a cancellation is a bounded request and its acknowledgement is not rollback. | RF-1.4, RF-1.8, RF-8.3, RNF-2 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing outbox and durable-jobs facility. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. No STOMP subscription is opened by this feature (it only closes them); Zustand is not used (no draft exists on N20). Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the status lifecycle is added to the hexagonal bounded context `tenant`; the member-facing reads are two identity-scoped queries in `access` **[009]**; `execution`, `integration`, `source`, `intelligence` and the realtime adapter react through published ports and the outbox event, never through a foreign repository. FE: N20 and its panel in the feature module shared with the chooser; the denied-scope path extended in `shared/errors` **[006]**. | §2 |
| **P12** Two independent tenancy gates | Tenant rows stay under `FORCE RLS` with `SET LOCAL`; the N20 read is the identity-scoped pattern of A-36 (identity predicate first, then the tenant predicate for that one tenant). Transition records and the reason category are platform rows outside the tenant runtime role's reach. Runtime authorisation is the second gate: the derivation refuses every tenant action whatever the rows say. | RF-2.8, RF-10.2, RNF-8 |
| **P13** Configured values are never defaulted | This feature introduces no numeric value; the checkpoint cadence of bounded cancellation belongs to execution and audit retention to audit (RNF-10). No timer moves a suspended tenant. | RF-7.1, RF-7.2, RNF-10 |
| **P14** Release bar | WCAG 2.2 AA on N20, its variants and its panel with automated and manual assistive evidence; EN / ES / FR for every statement and both messages. | RF-9, RNF-12–RNF-14 |
| **F-X-01** C10 context | No tenant C10 is ever built for a suspended tenant: `DeriveAuthorityUseCase` **[009]** reads the status through `TenantStatusPort` at every derivation and denies. N20 and its panel derive nothing; they read the person's own membership through the session (`SessionAssertionPort` **[006]**). | RF-1.2, RF-2.1, RF-3.3 |
| **F-X-02** Request integrity | Every effectful transaction re-reads the tenant status under a share lock inside its own transaction, so a request racing a suspension is decided under the status at commit (D-c). The platform commands are idempotent per target state. | RF-1.1, RF-6.1, LC-26 |
| **F-X-03** Secrets and safe problems | Refusals carry code-only problems (`access.denied-scope` **[006/009]**, `access.status-unavailable`, `tenant.unavailable`) with a correlation id; the reason, actor and time never leave the platform audit; nothing is written to browser storage. | RF-8.4, RF-10.2, RF-10.3, RF-10.5 |
| **F-X-04** Accessibility and languages | Focus enters the heading on open of the screen and of the panel; state is text; EN / ES / FR bundles; the switch changes display only. | RF-3.9, RF-9.2 |
| **F-X-05** Audit log | Every event of RF-10.1 is appended through `AuditPort` **[001]**; the platform's own entries carry `visibility = PLATFORM` and never appear in the tenant trail. | RF-10.1, RF-10.2 |
| **F-X-06** Email port | The two messages are outbox templates behind `TransactionalEmailPort` **[002]**; a relay failure is recorded and retried by the relay, never reverting a transition. | RF-10.4, RNF-11 |
| **F-X-07** Responsive surfaces | M21 (`432:4317`) is the drawn counterpart of N20; one page component follows the D01 responsive policy at other widths. | RF-9 |
| **F-IDN-16** Tenant suspension state | Freeze before retrieval, acceptance, approval, launch, provider submission or delivery (derivation + checkpoint checks); bounded cancel preserving facts; the screen's statements, account security and sign-out. | RF-1, RF-3 |

---

## 2. Module structure

### 2.1 Back end

#### 2.1.1 Bounded context `com.aspectra.tenant` — status lifecycle (owned here)

Plan 012 declares `TenantCommands.createTenant` in this context and stubs it. No plan yet owns the tenant record's status; this plan defines it, its transitions, the recovery contact and the platform commands. The tenant record's other fields and its creation stay with the tenant-record owner (plan 012's declared port).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `tenant.domain` | `TenantStatus` | Enumeration `ACTIVE \| SUSPENDED \| DELETING` and the pure transition table: `ACTIVE → SUSPENDED`, `SUSPENDED → ACTIVE`, `SUSPENDED → DELETING`; every other pair is refused; `DELETING` has no outgoing transition here. | RF-1.1, RF-6.1, RF-7.1, RF-7.4 |
| | `Tenant` (status part) | Methods `suspend(actor, reasonCategory, now)`, `reinstate(actor, now)`, `beginDeletion(actor, now)` — each applies the table, advances `statusVersion` and returns a `TenantStatusTransition`; a request for the current state is a no-op (`ALREADY`). Touches no other field. | RF-1.1, RF-1.8, RF-6.1 |
| | `TenantStatusTransition` | Value: tenant id, from, to, `reasonCategory` (`BILLING \| ABUSE \| LEGAL \| OPERATOR_REQUEST`; required for `→ SUSPENDED`, absent otherwise), platform actor id, time, correlation id. A platform record (D7). | RF-1.1, RF-6.1, RF-10.1, RF-10.2 |
| | `TenantRecoveryContact` | Value: tenant id, `channelKind` (`EMAIL_ADDRESS \| PHONE \| POSTAL \| PLAIN_TEXT`), `channelValue`, `instructions` (plain text), version, configured-at. At most one per tenant. | RF-4.1, RF-4.3, LC-16 |
| | `RecoveryContactRule` | Pure validation: channel value matches its kind; neither field contains a URL scheme, a host-like link, a token-like or code-like string; no instruction to reset a password or factor (checked against the localized phrase list of the reset and recovery features). | RF-4.3 |
| | `TenantFence` | Pure rule `(tenantStatus) → ACCEPT \| FENCE`: anything but `ACTIVE` fences. Published for the ingestion adapters of `execution` and `integration`. | RF-1.6, RNF-8, RNF-9 |
| `tenant.application` | `SuspendTenantCommand` | Internal command with no HTTP surface: lock the tenant row `FOR UPDATE`; `suspend`; insert the transition record; append the platform audit entry; enqueue `TenantStatusChanged{tenantId, from, to, statusVersion}` and one `tenant-suspended` message per active Owner and Admin (`TenantAdministratorsQuery` **[009]**, D12) — one transaction. `ALREADY` → audit only, no event, no message. | RF-1.1, RF-1.3–1.5, RF-1.7, LC-29 |
| | `ReinstateTenantCommand` | Same shape: `reinstate`; transition record; platform audit; `TenantStatusChanged{→ ACTIVE}`; one `tenant-reinstated` message per active Owner and Admin. Restores nothing (the freeze was never written to rows). | RF-6.1, RF-6.2, RF-6.5 |
| | `ConfigureRecoveryContactCommand` | `CONFIGURE \| CHANGE \| REMOVE` of the tenant's recovery contact after `RecoveryContactRule`; platform audit with the previous and new values. Allowed in any status; read by members only while paused. | RF-4.3, LC-16 |
| | Entry | The operational mechanism that invokes the three commands is a placeholder until A-62 / A-01 decide the operator path: an administrative command runnable only from the deployment environment with an actor identifier, never an HTTP endpoint and never a tenant screen — the shape of plan 004's compromise response and plan 005's case review. `beginDeletion` has no entry here: the deletion feature invokes it. | RF-1.1, RF-6.1, RF-7.1, RF-8.5, RNF-6 |
| | Ports | Published: `TenantStatusPort` (implementing plan 009's declaration: `statusOf(tenantId)` read per call, never cached; new `requireActiveForEffect(tenantId)` that reads the row `FOR SHARE` inside the caller's transaction and answers `Denied(TENANT_NOT_ACTIVE)` otherwise — D3), `TenantFence`, `TenantStatusChanged` (outbox event), `RecoveryContactPort.contactOf(tenantId)`. Consumed: `TenantAdministratorsQuery` **[009]**, `IdentityFactsPort` **[008]**, `IdentityLocalePort` (→ `identity`, D12), `SecurityNotificationPort` **[002]**, `AuditPort` **[001]**, `Clock`, outbox **[002]**. | all |
| `tenant.infrastructure` | Persistence | Column `tenant.status` (check constraint over the three values) and `tenant.status_version`; table `tenant_recovery_contact` (tenant-scoped, `FORCE RLS`, primary key `tenant_id`); table `platform.tenant_status_transition` in the platform schema, not granted to the tenant runtime role (D7, A-62 assumption). Persistence access API per A-35. | RF-1.1, RF-4.3, RF-10.2 |
| | Audit adapter | Platform-visibility events: `TENANT_SUSPENDED{reasonCategory, actor}`, `TENANT_REINSTATED{actor}`, `TENANT_TRANSITION_REFUSED{from, requested}`, `RECOVERY_CONTACT_CONFIGURED \| CHANGED \| REMOVED{actor, previous, new}`. Tenant-visibility events: `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_FAILED{template}`. Each carries tenant id, time and correlation id. | RF-10.1, RF-10.2 |

#### 2.1.2 Bounded context `com.aspectra.access` [009] — member-facing reads and the freeze fan-out

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `AccessStatusKind` | Pure rule `(membershipStatus, tenantStatus) → SUSPENDED_TENANT \| DELETING_TENANT \| NOT_PAUSED \| UNAVAILABLE`: `REVOKED` or no membership → `UNAVAILABLE`; tenant `SUSPENDED` → `SUSPENDED_TENANT`; tenant `DELETING` → `DELETING_TENANT`; tenant `ACTIVE` → `NOT_PAUSED` (a suspended membership in an active tenant is never N20, D-b). | RF-2.9, RF-3.3, RF-3.10, LC-24, LC-25, LC-33 |
| | `TenantAccessStatus` (read model) | `kind`, `tenantId`, `tenantName`, own `tenantRole`, own `membershipStatus`, `alternateEntry` (true when the identity holds an `ACTIVE` membership in another `ACTIVE` tenant, D-a). Never a reason, actor, time, project, source, run, member, invitation or other tenant's name. | RF-3.1, RF-3.2, RF-3.4, RF-3.8, RNF-5 |
| | `RecoveryPanelView` (read model) | `contact` (`channelKind`, `channelValue`, `instructions`) or `none`; nothing else. Identical for every member. | RF-4.1, RF-4.2, RF-4.4 |
| `access.application` | `TenantAccessStatusQuery` | For the session identity (`SessionAssertionPort` **[006]**) and a requested tenant id: the identity-scoped membership row through plan 009's identity-predicate query (A-36); tenant name and status through `TenantStatusPort`; `alternateEntry` from the same identity-scoped listing; applies `AccessStatusKind`. `UNAVAILABLE` → `access.status-unavailable`. For a paused kind, audits `TENANT_STATUS_VIEWED`. Builds no C10; writes no row but the audit entry. | RF-2.3, RF-3.1–3.4, RF-3.8, RF-3.10, RF-10.1 |
| | `RecoveryContactQuery` | Same identity check; only for a paused kind (otherwise `access.status-unavailable`); reads `RecoveryContactPort` under the tenant predicate set for that one tenant; audits `RECOVERY_PANEL_VIEWED`. Stores nothing from the person. | RF-4.1, RF-4.2, RF-4.4, RF-4.5 |
| | `TenantAdministratorsQuery` (published) | Active memberships with role `OWNER` or `ADMIN` of a tenant, read in the caller's transaction; used by the two commands of 2.1.1. | RF-1.7, RF-6.5, RF-10.4 |
| | `AuthorityChangedHandler` **[009]** — tenant scope | On `TenantStatusChanged{→ SUSPENDED \| DELETING}`: one call `SubscriptionRevocationPort.closeTenant(tenantId)` and one call `WorkRevocationPort.revokeTenant(tenantId, cause = TENANT_SUSPENDED)` instead of a per-membership loop. On `→ ACTIVE`: nothing. | RF-1.3, RF-1.4, RF-6.3 |
| `access.api` | `TenantAccessStatusController` — `/api/me/tenants/{tenantId}` | `GET access-status` (session required) → 200 `TenantAccessStatus` · 404 `access.status-unavailable` (the same for no membership, a revoked membership and an unknown tenant) · 401 without a session. `GET recovery-contact` → 200 `RecoveryPanelView` · 404 `access.status-unavailable`. RFC 9457 problems with a correlation id; the UI locale header selects nothing but bundle keys. | RF-3, RF-4, RF-10.3 |

#### 2.1.3 Consumer contracts declared here (no plan yet; stubbed)

| Context | Contract | Covers |
|---|---|---|
| `execution` | Implements `WorkRevocationPort.revokeTenant` **[009]**: every queued, not-admitted ticket of the tenant is refused at its next checkpoint and recorded NOT EXECUTED with cause `TENANT_SUSPENDED`; every running attempt receives a bounded cancellation request at its next checkpoint; facts already recorded stay. Independently of the event, admission and every attempt checkpoint call `AuthorityPort` / `TenantStatusPort` and stop when not `ACTIVE` (D4). Runner submissions and late results pass `TenantFence`: `FENCE` → recorded as `RECEIVED_FENCED`, applied to nothing, answered `tenant.unavailable`. Run-scoped service authority is not revoked beyond the cancellation request (A-39). | RF-1.4, RF-1.6, RF-8.3, RF-8.6 |
| `integration` (delivery) | On the event, and at every checkpoint before a provider write, a delivery job of a non-`ACTIVE` tenant stops; unconfirmed and unknown items keep their reconciliation state, flagged `HELD_BY_SUSPENSION`, and are never retried by a job; provider callbacks and receipts pass `TenantFence` and are recorded fenced without changing reconciliation state. | RF-1.5, RF-1.6, RF-6.3 |
| `source` (imports), `intelligence` (AI) | Stop at the next checkpoint when the tenant is not `ACTIVE`; reserve or consume nothing further; a stopped job is never restarted by a status change. | RF-1.5, RF-6.3 |
| Realtime adapter | Implements `SubscriptionRevocationPort.closeTenant` **[009]**; every new subscription is authorised by derivation, so none reopens while paused. | RF-1.3, RF-6.4 |
| Retention and purge jobs | Run under the ordinary retention rules; suspension starts none and stops none. | RF-7.3 |

Each context records its own audit events (`TICKET_NOT_EXECUTED{cause}`, `CANCELLATION_REQUESTED`, `DELIVERY_STOPPED`, `DELIVERY_ITEM_HELD`, `IMPORT_STOPPED`, `AI_REQUEST_STOPPED`, `STREAM_CLOSED`, `LATE_INPUT_FENCED`) through `AuditPort` **[001]** with tenant visibility and no reason category (RF-10.1).

**Cross-context touch points.** `identity` — `SessionAssertionPort` **[006]**, `IdentityFactsPort` **[008]**, `IdentityLocalePort`. `access` — `TenantAdministratorsQuery`, `AuthorityChangedHandler`, the identity-scoped listing **[009]**. `execution`, `integration`, `source`, `intelligence`, realtime — the contracts above. `audit` — append-only, with the visibility attribute. `sharedsupport` — `tx`, `contextIds`, `outbox`. No membership, grant, policy, revision, approval, run, fact, artifact or receipt row is written by this feature.

**Data model note (P12, RF-1.8).** The freeze is the value of `tenant.status` read at every derivation; nothing else changes. `tenant_recovery_contact` is a tenant row under `FORCE RLS`. `platform.tenant_status_transition` is a platform row that names a tenant but is not readable by the tenant runtime role, so neither the reason category nor the actor can reach a tenant query, a tenant audit trail or a member.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login, Signup, Invitation and the chooser), `shared/*`

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/workspaces/:tenantId/access-status` (N20) under `ProtectedBoundary` **[006]**; reached from the chooser's "View access status" **[013]**, from `TenantStateResolver` and from the guarded switch (016). | RF-2.1, RF-2.3, RF-2.4 |
| | `useTenantAccessStatus(tenantId)` | TanStack query on `GET /api/me/tenants/{id}/access-status`: `staleTime` 0, refetch on mount, no interval, focus or reconnect refetch, background marker never set; one bounded retry on transport failure. | RF-3.10, RF-10.5 |
| | `useRecoveryContact(tenantId)` | Query enabled only when the panel opens; refetched on each opening; nothing cached beyond memory. | RF-4.1, RF-4.5, LC-16 |
| | `contracts/accessStatus` (in `src/contracts`) | Types and runtime guards for both read models and the two problem codes; the guard rejects any field beyond the contract. | RF-3.2, RF-4.4, RNF-5 |
| Views | `TenantSuspendedPage` | N20: product header with display name and language switch, marker, heading, tenant statement, "Your current membership" card (Tenant, "Your tenant role", "Access state — Suspended · content unavailable", or "Suspended" as membership state), the metadata statement, "Review verified workspace recovery", the "Accepted runs do not become failed tests" notice, "Safe next steps", the alternate-tenant entry or its absence statement, "Personal account security", "Sign out". Deleting variant chosen by `kind` (S-55 / S-57 wording). Focus enters the heading. | RF-3.1, RF-3.2, RF-3.4, RF-3.7, RF-3.8, RF-2.9 |
| | `RecoveryContactPanel` | D02 panel on N20 (S-57 assumption): verified-process statement; the contact rendered as plain text, never autolinked, with "use only this contact"; or the no-contact statement; "Close" only. Focus enters the panel heading and returns to the trigger on close. | RF-4.1, RF-4.2, RF-4.4, RF-9.2 |
| | `SecurityStatements` **[001]** | Extended with the N20 statements and the panel statements in EN / ES / FR. | RF-3.1, RF-4.1, RNF-13 |
| Routing | `TenantStateResolver` (in `shared/errors`, extending `DeniedScopeHandler` **[006]**) | When `DeniedScopeHandler`'s one permission refresh is itself refused at tenant level (the authority read **[009]** answers `access.denied-scope`), the resolver aborts in-flight requests, disconnects STOMP, advances the scope epoch, clears the scoped cache, then reads `access-status`: a paused kind → N20 route; `NOT_PAUSED` or `access.status-unavailable` → `/workspaces`. A project-level refusal keeps the N28 path of plan 006 unchanged. | RF-1.3, RF-2.1, RF-2.2, RF-5.6, LC-1, LC-9, LC-27 |
| | Handoffs | Alternate tenant entry → `/workspaces` **[013]**; "Personal account security" → `SecuritySettingsPage` **[003]** (TOTP & recovery and the Sessions tab **[007]**); "Sign out" → `SignOutCoordinator.signOut({entry: 'N20'})` **[007]**; `NOT_PAUSED` on mount → `useEnterTenant(tenantId)` **[013]** (entered → overview; refused → `/workspaces`); `access.status-unavailable` → `/workspaces`. | RF-3.4–3.6, RF-3.10, LC-19 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/security` (scope epoch, `AccessCredentialHolder` **[006]**), `shared/i18n`, `shared/ui` (`Dialog`/panel, `Card`, `Button`, 44 px targets, visible focus). | RF-3.9, RF-9.2 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `SuspendTenantCommand` via the environment-only entry; `TenantStatus` table; transition record with actor, time, reason category; no tenant-side caller exists | — |
| RF-1.2 | `DeriveAuthorityUseCase` **[009]** reads `TenantStatusPort` per derivation → `TENANT_NOT_ACTIVE` for every action class | — |
| RF-1.3 | Next derivation denies; `AuthorityChangedHandler` → `SubscriptionRevocationPort.closeTenant` | `TenantStateResolver` after the refused refresh; polling requests refused and resolved |
| RF-1.4 | `WorkRevocationPort.revokeTenant` (execution contract): NOT EXECUTED, bounded cancellation, facts kept; checkpoint checks | — |
| RF-1.5 | Delivery, import and AI contracts: stop before the next provider write / next checkpoint, items held, nothing reserved | — |
| RF-1.6 | `TenantFence` in execution and integration ingestion; `RECEIVED_FENCED`; `tenant.unavailable` | — |
| RF-1.7 | `SuspendTenantCommand`: platform audit + `tenant-suspended` messages for `TenantAdministratorsQuery` recipients | — |
| RF-1.8 | Only `tenant.status` / `status_version` change; no other repository is called | — |
| RF-2.1 | Every tenant endpoint derives → denied; no content served | `TenantStateResolver` → N20 route |
| RF-2.2 | Code-only `access.denied-scope` **[006/009]** | `DeniedScopeHandler` **[006]** + resolver |
| RF-2.3 | `TenantEntryUseCase` **[013]** refuses; chooser card **[013]**; `TenantAccessStatusQuery` derives nothing | "View access status" → N20 route; back → `/workspaces` refetches **[013]** |
| RF-2.4 | Guarded switch step (4) authority read denied (016 RF-7.4 contract) | Resolver → N20; switch to an active tenant unaffected |
| RF-2.5 | Plan 008 acceptance refuses a non-active tenant; plan 010 issue / resend / revoke derive → denied; expiry job unchanged | — |
| RF-2.6 | Export, retry, reconciliation, archive, restore, purge request and deletion endpoints derive → denied | — |
| RF-2.7 | No row written (RF-1.8); reads denied | — |
| RF-2.8 | Status is per tenant; `revokeTenant` / `closeTenant` scoped by tenant id; derivations for other tenants read their own status | Other tenants' routes unaffected |
| RF-2.9 | `AccessStatusKind` → `DELETING_TENANT`; derivation denies `DELETING` | Deleting variant of `TenantSuspendedPage` |
| RF-3.1 | `TenantAccessStatus` read model | `TenantSuspendedPage` content verbatim from bundles |
| RF-3.2 | Read model fields only; contract guard | Renders the read model only |
| RF-3.3 | `AccessStatusKind` → `UNAVAILABLE` for revoked / none → 404 | `access.status-unavailable` → `/workspaces` |
| RF-3.4 | `alternateEntry` from the identity-scoped listing | Entry → `/workspaces` or the absence statement |
| RF-3.5 | Identity-scoped settings unaffected **[003/007]** | Route to `SecuritySettingsPage` **[003]** |
| RF-3.6 | `SignOutUseCase{entry: N20}` **[007]** | `SignOutCoordinator.signOut({entry: 'N20'})` **[007]** |
| RF-3.7 | No endpoint accepts a write from N20 | No retry, request, form or tenant link rendered |
| RF-3.8 | `tenantRole`, `membershipStatus` in the read model | Same page; role and state text vary |
| RF-3.9 | Nothing server-side changes | i18n re-render; query data kept |
| RF-3.10 | `AccessStatusKind` on each read: `NOT_PAUSED` / `DELETING_TENANT` | On mount: `NOT_PAUSED` → `useEnterTenant` **[013]**; deleting → variant |
| RF-4.1 | `RecoveryContactQuery` | `RecoveryContactPanel` with contact and "Close" |
| RF-4.2 | `RecoveryPanelView.none` | No-contact statement |
| RF-4.3 | `ConfigureRecoveryContactCommand`, `RecoveryContactRule`, platform audit | Plain-text render, never autolinked |
| RF-4.4 | One read model per tenant, no per-member field | Same panel for every role |
| RF-4.5 | Query only; `RECOVERY_PANEL_VIEWED` | No input fields |
| RF-5.1 | Identity endpoints never consult tenant status | Routes of 003, 004, 005, 007, 012, 013 unaffected |
| RF-5.2 | Reset / recovery **[004/005]** untouched by status; `LandingRouter` **[001]** → chooser | — |
| RF-5.3 | Leave contract for plan 015 (identity-scoped, D13) | Plan 015's personal settings |
| RF-5.4 | Members commands **[010]** derive → denied; leave contract refuses an Owner with `TENANT_SUSPENDED` | Members route → resolver → N20 |
| RF-5.5 | Erasure contract for plan 015: held while an owned tenant is suspended | Plan 015's erasure surface statement |
| RF-5.6 | No session port in this feature | `DeniedScopeHandler` **[006]** + resolver |
| RF-6.1 | `ReinstateTenantCommand` via the environment-only entry | — |
| RF-6.2 | Derivation reads unchanged rows and `ACTIVE` | Next request proceeds |
| RF-6.3 | No consumer acts on `→ ACTIVE`; held, stopped, NOT EXECUTED stay | — |
| RF-6.4 | Subscriptions authorised again by derivation; chooser projection lists `ACTIVE` **[013]**; plan 008 acceptance permits again | Chooser enterable card **[013]** |
| RF-6.5 | `ReinstateTenantCommand`: platform audit + `tenant-reinstated` messages | — |
| RF-6.6 | No membership or grant written by either command | — |
| RF-7.1 | `TenantStatus` table permits `SUSPENDED → DELETING` only via `beginDeletion`, invoked by the deletion feature; no scheduler | — |
| RF-7.2 | Templates and audit payloads carry no duration or date field | Bundles carry no time statement |
| RF-7.3 | Retention jobs unchanged; no purge started | — |
| RF-7.4 | `DELETING` has no outgoing transition | — |
| RF-8.1 | No identity, session, credential, factor or address port | — |
| RF-8.2 | No membership or grant port is written; reinstatement restores nothing | — |
| RF-8.3 | Execution contract: NOT EXECUTED, never failed; no fact deleted | Notice on N20 |
| RF-8.4 | Read models and problems carry no reason, actor, time or content; platform records unreachable | Contract guards |
| RF-8.5 | No tenant-side caller of the commands; reset / recovery / re-auth never touch `tenant.status` | No reinstatement control |
| RF-8.6 | `revokeTenant` / `closeTenant` / `TenantFence` keyed by tenant id; run-scoped authority untouched beyond cancellation | — |
| RF-9.1 | — | Breakpoint reflow of page and panel |
| RF-9.2 | — | `shared/ui` targets, labels, focus to heading |
| RF-9.3 | — | Breakpoint layout replaceable (S-02) |
| RF-10.1 | Audit adapters of `tenant`, `access` and the consumer contexts | Correlation id header |
| RF-10.2 | `visibility = PLATFORM`; platform-schema transition table | — |
| RF-10.3 | `access.denied-scope`, `access.status-unavailable`, `tenant.unavailable` — code only | — |
| RF-10.4 | Two outbox templates, locale rule, failures recorded | — |
| RF-10.5 | — | Query cache in memory only; nothing in storage |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The status is one column on the tenant record; the freeze is a derivation outcome, never a rewrite of rows.**
*Why.* RF-1.8 and decision Q5: suspension must change no membership, grant, version, revision, run or fact, and reinstatement must restore exactly what was there. Plan 009 D9 already reads the status through `TenantStatusPort` at every derivation; one column read per request gives "next request" semantics (D-c) and makes reinstatement a status change with nothing to restore (RF-6.2, RF-6.6).
*Discarded.* Suspending every membership — rewrites membership state and versions, and reinstatement could not tell a membership suspended before from one suspended by the tenant (LC-22). Copying the status onto memberships — rejected by plan 009 D9. A separate "frozen" flag per context — many sources of one truth.
*Covers.* RF-1.2, RF-1.8, RF-2.7, RF-6.2, RF-6.6, RF-8.2, RNF-4, LC-22, LC-23.

**D2 · Suspension, reinstatement and recovery-contact changes are internal application commands invoked by an environment-only administrative entry with an actor identifier.**
*Why.* Decision Q1 and scope §2: only the platform's internal, audited, out-of-band mechanism moves the status; no customer-facing operator surface exists. Plans 004 D10 and 005 already implement platform acts this way; the commands exist so the behaviour (transitions, records, messages, audit) is built and tested now while the operator path stays A-62 / A-01's decision.
*Discarded.* An administrative REST endpoint — an operator surface in the product. An Owner or Admin action — Q1. A scheduled move to deleting after a configured time — Q5 and RF-7.1. A payment-event listener — RF-1.1.
*Covers.* RF-1.1, RF-4.3, RF-6.1, RF-7.1, RF-8.5, RNF-6, LC-31.

**D3 · A request racing a transition is decided under the status at commit: the transition holds the tenant row `FOR UPDATE`; every effectful transaction re-reads it `FOR SHARE` through `TenantStatusPort.requireActiveForEffect`.**
*Why.* D-c and LC-26: a write that started before the suspension must not commit after it. The derivation at request start is not enough on its own, because the transaction may commit after the transition. A share lock taken in the effect's own transaction serialises against the transition's exclusive lock and costs nothing between concurrent writes (share locks do not conflict); reads need no lock because they produce no effect.
*Discarded.* Checking only at request start — a racing write commits after the freeze. `SERIALIZABLE` isolation for every transaction — retries everywhere for one rare event. A cached status with a short lifetime — RNF-1 forbids authority outliving its request.
*Covers.* RF-1.2, RNF-1, RNF-9, LC-26.

**D4 · Freeze propagation is two independent mechanisms: every job, admission and ingestion checkpoint reads the status itself (correctness), and an outbox event `TenantStatusChanged` wakes the consumers early (latency).**
*Why.* RF-1.3–1.6 must hold even if an event is delayed or lost; the checkpoint check guarantees the freeze from the next checkpoint, as plan 009 does for membership revocation (A-56). The event lets streams close and queued tickets end without waiting for their own next poll. No push to clients is needed: the next request of every client is refused (D-c).
*Discarded.* The event alone — a lost or late event would let work run in a frozen tenant. A push message to every client — the client is not an authority (P3), and refusing the next request already reaches it.
*Covers.* RF-1.3–1.6, RNF-1, LC-2–LC-6, LC-27.

**D5 · Work in flight is revoked by tenant scope through plan 009's existing ports; reinstatement starts nothing.**
*Why.* Decision Q3 and A-56: the rule for a revoked membership is the rule for a suspended tenant, so `WorkRevocationPort` and `SubscriptionRevocationPort` **[009]** gain tenant-scope operations, called once instead of per membership. NOT EXECUTED carries the cause `TENANT_SUSPENDED` (never a reason category). Delivery items are held in their reconciliation state; no consumer listens to `→ ACTIVE`, so P1 holds by construction.
*Discarded.* Pausing queued tickets and resuming them on reinstatement — Q3 and P1. Marking running attempts failed — P8. A per-membership loop — N identical calls for one tenant fact.
*Covers.* RF-1.4, RF-1.5, RF-6.3, RF-8.3, RNF-2, RNF-3, LC-2–LC-4, LC-20, LC-21.

**D6 · Late inputs pass a published pure `TenantFence`; a fenced input is recorded and applied to nothing, and the sender gets one generic code.**
*Why.* RF-1.6 and the failure section: fencing rejects late submissions, results and delivery callbacks. A pure rule shared by every ingestion adapter keeps the decision identical across execution and delivery; recording the input keeps it reviewable; `tenant.unavailable` reveals nothing beyond the refusal (RF-10.3).
*Discarded.* Dropping silently — no evidence of what arrived. Buffering for application after reinstatement — a replay (RF-6.3). A code that names suspension — RF-10.3.
*Covers.* RF-1.6, RF-8.6, RF-10.3, RNF-8, LC-5, LC-6.

**D7 · The reason category, the actor and the contact history live only in platform records: the platform-schema transition table and audit entries with `visibility = PLATFORM`.**
*Why.* RF-10.2 and RNF-5: they must never be in the tenant's audit trail, on a surface or in a message. A table the tenant runtime role cannot read, and an audit visibility attribute the tenant trail query never selects, make the disclosure impossible rather than filtered. A-62 leaves the location open; this is the assumption built.
*Discarded.* The reason on the tenant row — readable by every tenant-scoped query. The tenant audit trail with the reason redacted — the redaction becomes the safeguard.
*Covers.* RF-1.1, RF-3.2, RF-8.4, RF-10.1, RF-10.2, RNF-5, LC-30.

**D8 · The recovery contact is a validated plain-text record, rendered as plain text.**
*Why.* Decision Q2 and RF-4.3: a channel and instructions, never a link, token or code that grants access, and no instruction to reset a password or factor. A validation rule at configuration and plain rendering on N20 (no autolinking) close both paths.
*Discarded.* Rich text or a link field — an access path from a paused screen. A contact form on N20 — Q2 ("nothing is submitted").
*Covers.* RF-4.1, RF-4.3, RF-4.5, LC-16, LC-17.

**D9 · N20 is fed by an identity-scoped read model and a separate panel read; neither builds a tenant C10.**
*Why.* RF-3.3 and plan 013 D11: N20 shows the person's own membership in a tenant that must stay closed. The A-36 pattern (identity predicate first) gives exactly the person's row; tenant name, status and contact are read for that one tenant only. A separate panel read is what makes "opening is audited as a view" (RF-4.5) a server fact, and lets a contact configured mid-suspension appear at the next opening (LC-16). One code for no membership, revoked and unknown keeps RF-3.3 from becoming an existence oracle.
*Discarded.* Rendering N20 from the chooser's data — stale, and the chooser carries no contact. Deriving a tenant C10 to read the metadata — a tenant context for a frozen tenant. Embedding the contact in the status read — the panel view could not be audited.
*Covers.* RF-2.3, RF-3.1–3.4, RF-3.8, RF-4.1–4.5, RNF-5, LC-7, LC-8, LC-16, LC-17, LC-33.

**D10 · Members reach N20 through the denied-scope path: the refusal stays code-only, and a `TenantStateResolver` beside plan 006's handler asks the identity-scoped status when a tenant-level refresh is refused.**
*Why.* RF-2.2 sends a member's refused request through the denied-scope handling of specification 006 to the screen. Plan 009 keeps the reason class on the server; a distinct "tenant suspended" code on every refusal would tell any caller — member or not — why (RF-10.3). The resolver asks only for the person's own row, so only a member learns the state. It runs the switch's step (2)–(3) primitives first so no late response or stream survives (F-IDN-18).
*Discarded.* A distinct problem code — discloses the cause to every caller. A server redirect — the server never redirects (plans 001, 013). Checking status before every navigation — a client gate (P3) and a request per click.
*Covers.* RF-1.3, RF-2.1, RF-2.2, RF-2.4, RF-5.4, RF-5.6, LC-1, LC-9, LC-10, LC-27.

**D11 · N20 is fresh "on open": refetch on mount, no polling, no subscription; a tenant found active is entered through plan 013's entry request.**
*Why.* RF-3.10 and RF-10.5: the state is a server fact re-read at every request; the screen opens no stream (none may exist for a frozen tenant) and polls nothing. When the read says `NOT_PAUSED`, `useEnterTenant` **[013]** runs the tenant-level derivation, so entering after reinstatement follows the ordinary rules and a suspended membership still ends on the chooser (D-b).
*Discarded.* Polling until reinstatement — background traffic for a state that may last months. A realtime notice — no subscription may exist in a frozen tenant. Navigating straight to the overview — bypasses the entry derivation.
*Covers.* RF-3.10, RF-6.4, RF-10.5, LC-19, LC-25.

**D12 · The two messages are outbox templates enqueued in the transition transaction for every active Owner and Admin, each in the recipient's last-used interface language.**
*Why.* Decision Q4 and RF-10.4. Enqueuing in the same transaction means a message exists exactly when the transition committed, and a relay failure (recorded `NOTIFICATION_FAILED`) never reverts it (RNF-11). There is no request from the recipients, so the language comes from `IdentityLocalePort`, which returns the UI locale recorded at the identity's latest completed sign-in (a fact recorded by `CompleteSignInUseCase` **[001]**, cross-plan item). Every identity has a recorded interface locale from signup onwards (plan 002), so no fallback is needed; a missing value is a defect to fix, not a trigger for a multilingual message.
*Discarded.* Sending after commit from the command — a crash between commit and send loses the message. Emailing Members — Q4. Defaulting to English — an assumed value. Storing a locale on the identity row — FR-IDN-001 (only email, display name, status).
*Covers.* RF-1.7, RF-6.5, RF-10.4, RNF-11, RNF-13, LC-29.

**D13 · Everything identity-level is reused untouched; the leave and erasure rules for a suspended tenant are contracts for plan 015.**
*Why.* Decision Q6 and RNF-7: no identity endpoint consults tenant status, so security settings, reset, recovery, sessions, sign-out, the chooser, workspace creation and other invitations work as usual by construction. Leaving cannot run under a tenant C10, because the derivation denies every action in a suspended tenant; plan 015's leave must therefore run on identity-scoped facts (the person's own membership), permit a Member or Admin, refuse an Owner with `TENANT_SUSPENDED`, and revoke through `MembershipCommands.RevokeMembership` **[009]** without the tenant derivation. Its erasure request must hold while a last-Owner transfer is needed in a suspended tenant.
*Discarded.* Opening a tenant C10 exception for leaving — a door in the freeze. Defining plan 015's surfaces here — another feature's plan.
*Covers.* RF-5.1–5.6, RF-8.1, RNF-7, LC-11–LC-14, LC-32.

**D14 · One page component with breakpoint layouts renders N20 and its panel at every width.**
*Why.* M21 is the drawn counterpart (S-02, S-57); FR-X-055 requires AA reflow at every other width; one component with shared statements keeps every statement, card, action and route present at every width (RNF-14).
*Discarded.* A separate mobile component — drift.
*Covers.* RF-9.1–9.3, RNF-12, RNF-14, LC-18.

**D15 · Errors are RFC 9457 problem details with safe codes and a correlation id [001 D12].**
*Why.* F-X-03 and RF-10.3. The feature's codes are `access.status-unavailable` and `tenant.unavailable`, plus `access.denied-scope` **[006/009]** on every tenant endpoint; the correlation id joins a person's report to the audit entry.
*Discarded.* Ad-hoc error bodies; a cause in the problem title.
*Covers.* RF-10.3, RNF-5.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure rules: `TenantStatus` transition table, `RecoveryContactRule`, `TenantFence`, `AccessStatusKind`, recipient selection, locale fallback. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Commands through real adapters: one transaction for status, record, audit, event and messages; the platform table unreadable by the tenant runtime role; the identity-scoped reads under RLS; derivation outcomes; audit visibility. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of reason, actor, time and content in every response. |
| Concurrency | JUnit 5 with real PostgreSQL | Writes racing a suspension (share lock vs exclusive lock), admission racing a suspension, late results and callbacks after the commit (areas A and H). |
| Contract (consumers) | JUnit 5 with recording stubs of `execution`, `integration`, `source`, `intelligence`, realtime | Each consumer contract of §2.1.3 is called with the tenant scope and that nothing is called on reinstatement; the owning plans re-run these as their own tests. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page variants, panel states, resolver branches, handoffs, no polling, nothing in storage. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Focus to heading on screen and panel, reading order, state as text, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RNF-6 | BE unit, integration | `tenantStatus.transitionTable`; `suspend.recordsActorTimeReasonCategory`; `tenantSide.noCallerOfStatusCommands` (ArchUnit-style dependency check) | 8 |
| RF-1.2, RNF-1, RNF-9 | BE integration | `derive.everyActionClassDeniedAfterSuspension`; `tenantStatusPort.neverCached` | 3 |
| RF-1.3, LC-27 | BE integration, FE e2e | `suspend.closesTenantSubscriptions`; `pollingTab.refusedThenN20NothingServed` | 3 |
| RF-1.4, RF-8.3, RNF-2, LC-2, LC-3 | Contract, BE integration | `revokeTenant.queuedNotExecutedCauseTenantSuspended`; `revokeTenant.runningBoundedCancelFactsKept`; `suspension.neverMarksFailedOrRewritesVerdict` | 4 |
| RF-1.5, LC-4 | Contract | `delivery.stopsBeforeProviderWriteItemsHeldNeverRetried`; `importAndAi.stopAtCheckpointReserveNothing` | 4 |
| RF-1.6, LC-5, LC-6 | BE unit, contract | `tenantFence.nonActiveFences`; `lateResultAndCallback.recordedFencedAppliedToNothing` | 4 |
| RF-1.7, RF-10.4, LC-29 | BE integration | `suspend.enqueuesMessagePerActiveOwnerAndAdmin`; `messageFailure.recordedTransitionStands` | 10 |
| RF-1.8, RF-2.7, RNF-4 | BE integration | `suspendAndReinstate.changeOnlyTenantStatus` (row snapshots of every tenant table before and after) | 7 |
| RF-2.1, LC-9 | FE e2e | `tenantRoute.typedUrlEndsOnN20NoContent` | 3 |
| RF-2.2 | API contract, FE unit | `tenantEndpoints.deniedScopeCodeOnly`; `resolver.tenantLevelRefusalReadsStatus` | 3 |
| RF-2.3, LC-7, LC-8 | FE e2e (013 stub) | `chooser.viewAccessStatusOpensN20WithoutEntry`; `onlySuspendedTenant.landsOnChooser` | 3 |
| RF-2.4 | FE e2e (016 stub) | `guardedSwitch.intoSuspendedEndsOnN20NoStream`; `guardedSwitch.outOfSuspendedToActiveProceeds` | 3 |
| RF-2.5, LC-15 | BE integration (008, 010 stubs) | `invitation.suspendedTenantUnusableStaysPending`; `invitation.issueResendRevokeDenied`; `invitation.expiryUnchanged` | 3 |
| RF-2.6 | API contract | `exportRetryReconcileArchiveRestorePurge.deniedWhileSuspended` | 3 |
| RF-2.8, RF-8.6, RNF-8 | BE integration | `suspension.otherTenantsUntouched`; `fence.neverCrossesTenants` | 9 |
| RF-2.9, RF-7.4, LC-24 | BE unit, FE unit | `accessStatusKind.deletingVariant`; `deleting.noOutgoingTransition`; `n20.deletingWording` | 8 |
| RF-3.1 | FE unit | `n20.rendersDesignedContentVerbatim` | 5 |
| RF-3.2, RF-8.4, RNF-5, LC-30 | API contract, FE unit | `accessStatus.exposesOnlyOwnMetadata`; `contractGuard.rejectsExtraFields`; `nowhere.namesReasonActorOrTime` | 5 |
| RF-3.3, LC-33 | BE integration | `accessStatus.revokedOrNoneSameUnavailable` | 5 |
| RF-3.4 | BE integration, FE unit | `alternateEntry.onlyWithAnotherActiveTenant`; `n20.alternateEntryOrAbsenceStatement` | 5 |
| RF-3.5 | FE e2e (003 stub) | `n20.personalAccountSecurityOpensSecuritySettings` | 5 |
| RF-3.6 | FE e2e (007 stub) | `n20.signOutEntryN20SuspensionUntouched` | 5 |
| RF-3.7, LC-31 | FE unit, API contract | `n20.noRetryRequestFormOrTenantLink`; `noEndpoint.acceptsWriteFromN20` | 8 |
| RF-3.8 | FE unit | `n20.sameForOwnerAdminMemberExceptRoleAndAlternate`; `n20.suspendedMembershipShowsSuspended` | 5 |
| RF-3.9, LC-28 | FE unit | `languageSwitch.displayOnly` | 11 |
| RF-3.10, LC-19 | FE e2e (013 stub) | `reinstatedWhileOpen.nextRequestEntersTenant` | 7 |
| RF-4.1, LC-16 | BE integration, FE unit | `recoveryContact.shownFromNextOpening`; `panel.contactPlainTextCloseOnly` | 5 |
| RF-4.2, LC-17 | FE unit | `panel.noContactStatement` | 5 |
| RF-4.3 | BE unit, integration | `recoveryContactRule.rejectsLinksTokensCodesResetInstructions`; `recoveryContact.changesAuditedPlatformOnly` | 5 |
| RF-4.4 | API contract | `panel.identicalForEveryMember` | 5 |
| RF-4.5 | BE integration | `panel.viewAuditedStoresNothing` | 10 |
| RF-5.1, RNF-7, LC-32 | FE e2e, BE integration | `identityActions.workWhileSuspended` (settings, reset, recovery, sessions, chooser, other tenants, creation, other invitations) | 6 |
| RF-5.2, LC-13, LC-14 | BE integration (004, 005 stubs) | `resetAndRecovery.leaveSuspensionLandOnChooser` | 6 |
| RF-5.3, LC-11, LC-23 | Contract (015 stub) | `leave.memberOrAdminRevokesOwnMembershipOnly`; `leftThenReinstated.notAMember` | 6 |
| RF-5.4, LC-10 | BE integration, contract | `members.transferRoleInviteDenied`; `leave.ownerRefusedTenantSuspended` | 6 |
| RF-5.5, LC-12 | Contract (015 stub) | `erasure.heldWhileOwnedTenantSuspended` | 6 |
| RF-5.6, RF-8.1, LC-1 | BE integration, FE e2e | `suspension.touchesNoSessionCredentialFactorAddress`; `openEditor.deniedScopeThenN20` | 6 |
| RF-6.1 | BE integration | `reinstate.recordsActorAndTime` | 8 |
| RF-6.2, RF-6.6, RF-8.2, LC-22 | BE integration | `reinstate.membershipsGrantsVersionsAsBefore`; `reinstate.restoresNoEarlierSuspendedOrRevoked` | 7 |
| RF-6.3, RNF-3, LC-20, LC-21 | Contract, BE integration | `reinstate.noConsumerStartsWork`; `notExecutedHeldStoppedStaySo` | 7 |
| RF-6.4 | BE integration (008, 013 stubs) | `reinstate.chooserActiveInvitationsAcceptableSubscriptionsAllowed` | 7 |
| RF-6.5 | BE integration | `reinstate.enqueuesRestoredMessageNothingResumed` | 7 |
| RF-7.1 | BE unit, startup | `suspendedToDeleting.onlyViaBeginDeletion`; `noSchedulerMovesStatus` | 8 |
| RF-7.2 | BE unit, FE unit | `templatesAndBundles.promiseNoDurationOrDate` | 8 |
| RF-7.3 | BE integration | `suspension.startsNoPurgeOrTombstone` | 7 |
| RF-8.5 | BE integration | `resetRecoveryReauth.neverLiftSuspension`; `noTenantSideTransition` | 8 |
| RF-9.1–9.3, RNF-12, RNF-14, LC-18 | FE e2e + axe, manual assistive | `a11y.n20AndPanel.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 11 |
| RF-10.1 | BE integration | `audit.everyEventOfRf10_1WithCorrelationId` | 10 |
| RF-10.2 | BE integration | `platformRecords.unreadableByTenantRole`; `tenantTrail.neverSelectsPlatformVisibility` | 10 |
| RF-10.3 | API contract | `problems.safeCodesOnlyNoCauseForNonMembers` | 10 |
| RF-10.4, RNF-13 | BE unit, integration | `messages.enEsFrRecipientLocaleTrilingualFallback`; `messages.noReasonNoAccessLink` | 10 |
| RF-10.5 | FE unit | `storage.containsNothingFromN20` | 5 |
| RNF-10 | Startup | `suspension.introducesNoConfigurationProperty` | 12 |
| RNF-11 | BE integration | `emailAbsent.operationalFailureTransitionStands` | 10 |
| RNF-15, LC-26 | Concurrency | `write.racingSuspensionDecidedAtCommit`; `admission.racingSuspensionRefused`; `lateCallback.afterCommitFenced` | 12 |
| LC-25 | BE unit, FE e2e | `suspendedMembershipActiveTenant.neverN20` | 3 |
| US-1 … US-9 | FE e2e | journeys above | 1 |
| US-10 | all adversarial cases above | reason discovery, content through the paused state, effects in a frozen tenant, cancelled shown as failed, resume by reinstatement, tenant-side transitions, reset or recovery lifting the state | 1 |

### 5.3 Not tested here
The derivation and its matrix (plan 009); the chooser cards and entry (013); the denied-scope panel N28 and session handling (006); sign-out (007); security settings (003); reset and recovery (004, 005); invitation acceptance and the members surface (008, 010); leave and erasure surfaces (015); the guarded switch beyond the resolver (016); the internals of admission, cancellation, delivery reconciliation, imports and AI (their own plans); the operator tooling (A-62 / A-01); tenant deletion (FR-IDN-125). Their stubs return fixed answers; the consumer contracts of §2.1.3 are verified by recording stubs here and by the owning contexts' own tests later.

### 5.4 Exit criteria
Specification §7 items 1–12 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-62 suspension mechanism and recovery-contact record (decided in principle; location open) | Environment-only commands (D2); platform-schema transition table and `visibility = PLATFORM` audit (D7); `tenant_recovery_contact` beside the tenant record (D8); status read at every derivation and checkpoint (D3, D4). | Rebind the commands to the decided operator path; move the records; the rules, reads and screen are unchanged. |
| A-01 PlatformOperator path | The administrative entry of D2 is a placeholder. | Same as A-62. |
| A-56 work in flight (decided) | Tenant-scope revocation through plan 009's ports (D5). | None expected. |
| A-54 suspended and revoked meanings (decided) | `AccessStatusKind`: revoked never reaches N20; a suspended membership in an active tenant is not N20 (D-b). | Rule table only. |
| A-39 run-scoped authority (decided) | Untouched beyond the cancellation request (RF-8.6). | None expected. |
| A-36 identity-scoped listing | N20 reads through plan 009 D10's identity predicate (D9). | Replace the policy; the read models are unchanged. |
| A-30 notification set | Adds `tenant-suspended` and `tenant-reinstated`. | Adjust the set; the commands are unchanged. |
| A-35 persistence tooling | Repositories against the placeholder API. | Rename base types. |
| S-57 N20 states | Panel in both states, alternate-entry variant, role variants, suspended-membership variant, deleting wording — built as described. | Replace layout and copy; the read models are unchanged. |
| S-55 deleting wording (decided) | `DELETING_TENANT` variant. | Copy only. |
| S-48 chooser suspended-membership card (decided) | Not this screen (D-b). | — |
| S-14 email templates | Two templates with the content of RF-10.4. | Copy only. |
| S-02 responsive coverage | Reflow (D14). | Replace with the designed counterpart. |
| S-17 Help | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |

**Cross-plan items raised by this plan.**
1. **Plan 009:** `WorkRevocationPort.revokeTenant(tenantId, cause)` and `SubscriptionRevocationPort.closeTenant(tenantId)` as tenant-scope operations used by `AuthorityChangedHandler` for `TenantStatusChanged`; the published `TenantAdministratorsQuery`; `TenantStatusPort.requireActiveForEffect` called inside every effectful command's transaction (D3).
2. **Plan 012:** `TenantCommands.createTenant` creates the tenant with status `ACTIVE` and `statusVersion` 0.
3. **Plan 006:** `DeniedScopeHandler` hands a tenant-level refusal of its permission refresh to `TenantStateResolver` (D10).
4. **Plan 001:** `CompleteSignInUseCase` records the request's UI locale for `IdentityLocalePort` (D12).
5. **Audit context:** the `visibility` attribute (`TENANT | PLATFORM`); the tenant audit trail (F-APR-04, N18) selects `TENANT` only.
6. **Plan 015:** an identity-scoped leave for a suspended tenant (Member or Admin permitted, every Owner refused with `TENANT_SUSPENDED` — specification 015 RF-4.1, RF-2.4, D-a), and the erasure hold (015 RF-2.3, RF-5.5) (D13).
7. **Plan 016:** when step (4) finds the target tenant suspended, the switch hands over to `TenantStateResolver`, which opens N20 (specification 016 RF-7.4); a revoked or suspended membership goes to the chooser (016 RF-7.3); a deleting tenant opens N20 in its deleting wording (`401:2714`; 016 RF-7.4, this specification D-d); for the chooser origin, 013 RF-6.3 refreshes the chooser instead (016 Q5).
8. **Execution, delivery, import, AI and realtime plans:** the contracts of §2.1.3.

The specification's derived rules are built, not assumed: alternate entry computed on the server (D-a, D9), N20 only for the tenant's state (D-b, `AccessStatusKind`), status read at every derivation and decided at commit (D-c, D3, D4), deleting treated as suspended for members (D-d, `AccessStatusKind`, `TenantFence`).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.8) | `TenantStatus`, commands, `TenantStatusPort`, `AuthorityChangedHandler` [009], consumer contracts, `TenantFence` | D1–D7, D12 | RF-1 rows |
| RF-2 (2.1–2.9) | derivation [009], `TenantStateResolver`, N20 route, `AccessStatusKind` | D1, D9, D10 | RF-2 rows |
| RF-3 (3.1–3.10) | `TenantAccessStatusQuery`, `TenantSuspendedPage`, handoffs | D9, D10, D11 | RF-3 rows |
| RF-4 (4.1–4.5) | `RecoveryContactQuery`, `RecoveryContactPanel`, `RecoveryContactRule`, `ConfigureRecoveryContactCommand` | D2, D8, D9 | RF-4 rows |
| RF-5 (5.1–5.6) | identity routes untouched, 015 contract, resolver | D10, D13 | RF-5 rows |
| RF-6 (6.1–6.6) | `ReinstateTenantCommand`, no `→ ACTIVE` consumer | D1, D2, D5, D11, D12 | RF-6 rows |
| RF-7 (7.1–7.4) | `TenantStatus` table, no scheduler, retention unchanged | D1, D2 | RF-7 rows |
| RF-8 (8.1–8.6) | no identity / membership port, platform records, tenant-keyed revocation | D1, D5–D7, D13 | RF-8 rows |
| RF-9 (9.1–9.3) | breakpoint reflow, `shared/ui` | D14 | RF-9 row |
| RF-10 (10.1–10.5) | audit adapters, visibility, problems, templates, no storage | D6, D7, D12, D15 | RF-10 rows |
| RNF-1 | per-derivation read, no cache | D3, D4 | `tenantStatusPort.neverCached` |
| RNF-2 | execution contract | D5 | `suspension.neverMarksFailedOrRewritesVerdict` |
| RNF-3 | no `→ ACTIVE` consumer | D5 | `reinstate.noConsumerStartsWork` |
| RNF-4 | status-only change | D1 | `suspendAndReinstate.changeOnlyTenantStatus` |
| RNF-5 | read models, platform records | D7, D9 | `accessStatus.exposesOnlyOwnMetadata` |
| RNF-6 | environment-only commands | D2 | `tenantSide.noCallerOfStatusCommands` |
| RNF-7 | identity routes untouched | D13 | `identityActions.workWhileSuspended` |
| RNF-8 | tenant-keyed scope, fence | D5, D6 | `fence.neverCrossesTenants` |
| RNF-9 | fail-closed status and fence | D3, D6 | `derive.everyActionClassDeniedAfterSuspension` |
| RNF-10 | no configuration | — | `suspension.introducesNoConfigurationProperty` |
| RNF-11 | outbox, failures recorded | D12 | `emailAbsent.operationalFailureTransitionStands` |
| RNF-12, RNF-13, RNF-14 | `shared/ui`, `shared/i18n`, breakpoints | D12, D14 | a11y and language rows |
| RNF-15 | locks, fence, checkpoints | D3, D4, D6 | concurrency rows |
| LC-1 … LC-33 | per the §5.2 rows that name them | — | §5.2 |

Every EARS criterion RF-1.1 … RF-10.5 appears in §3; every RNF and LC appears in §5.2 or above. No element of §2 lacks a criterion, and no criterion lacks an element, a decision and a test.
