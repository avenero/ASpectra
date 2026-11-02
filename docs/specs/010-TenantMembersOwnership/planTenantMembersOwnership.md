# Plan 010 — Tenant members and ownership transfer

**Feature** The tenant member list, invitations, tenant role changes, membership management and ownership transfer · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/010-TenantMembersOwnership/specTenantMembersOwnership.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; features F-IDN-12 and F-SET-04 as the subject; F-IDN-10, F-IDN-11, F-IDN-13, F-IDN-15, F-IDN-16, F-IDN-17, F-SET-01, F-SET-05 and F-APR-04 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (the `access` context owns "memberships, invitations, project grants, permission versions"; ownership ERD; "lock the tenant ownership row during concurrent owner mutations"), decision register `122:2` §3 and §5, FE board `46:7847` (feature "workspace / memberships") · **Builds on** plan 009 (Roles, grants and the authority matrix: the `access` core and its published commands), plan 008 (Invitation acceptance: the invitation record), plan 003 (Enrollment: the step-up challenge and `FactorStatusPort`), plan 002 (Signup: token generation and hashing, the outbox), plan 001 (Login), plan 006 (Sessions: the session assertion and the denied-scope handling) and plan 007 (Logout); elements they own are tagged **[001]** … **[009]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's ten decisions (Q1–Q10) and four derived rules (D-a…D-d) are built as decided. Plan 009 D7 explicitly leaves "the last-Owner guard under the ownership row lock" and `TransferOwnership` to this plan, and declares `TenantMembersQuery` as the read model this plan implements; plan 008 leaves the issue, resend and revoke operations of its `Invitation` aggregate **[008 → 010]** to this plan. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (S-50, S-49, S-04, S-14, S-02; A-52, A-54…A-57, A-30, A-27, A-36; plus A-35 for the persistence tooling), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

**Cross-plan note.** Plans 008 and 009 share one project-grant record — one row per membership and project carrying four flags; plan 008 was aligned to plan 009 on 28 October 2026. Plan 009 is the declared owner of the `access` core and this plan follows it.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | No use case of this feature reads, stores or replays a mapping, approval, run or intent; membership changes reach queued and running work only through plan 009's `AuthorityChanged` event, which refuses or cancels at bounded checkpoints and never marks anything failed. | RF-7.4, RF-11.5, RNF-7 |
| **P3** The browser expresses intent, never authority | The client submits a target, a chosen role or action, a step-up proof, an address and scope for an invitation, and the version it saw; which actions a row offers, whether the actor may act, the last-Owner guard, eligibility, refusal reasons and the effects are all decided on the server. The FE mirrors plan 009's authority store for presentation only. | RF-1.3, RF-2.4, RF-5.2, RF-10.6, RNF-1 |
| **P6** Fail closed | A stale version, a failed recheck, an unknown target, a closed invitation, a missing step-up, an inactive tenant, an absent configuration value or an absent policy refuses the change; nothing is assumed and no fallback exists. | RF-10.4, RF-10.6, RNF-11, RNF-12 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL (row locks, partial unique indexes, `FORCE RLS`), the existing outbox and durable jobs, plan 003's step-up. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. Nothing is added. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the feature lives in the existing hexagonal bounded context `access` beside plan 009's core; `identity`, `tenant`, `project`, `audit` and `sharedsupport` are reached only through published ports; messages go through the outbox; expiry through a durable job. FE: a feature module `workspace-members` with a public API; TanStack Query owns server facts; Zustand is not used (no unsaved fork). | §2 |
| **P12** Two independent tenancy gates | `tenant_ownership`, `tenant_invitation` **[008]**, `tenant_membership` and `project_grant` **[009]** carry `tenant_id` in their keys under `FORCE RLS` with `USING` and `WITH CHECK`; every request derives C10 through plan 009's filter and sets the tenant context with `SET LOCAL`; runtime authorisation (`AuthorityMatrix` class `TENANT_MEMBERS`, `AdministrationRule`, the last-Owner guard) is the second gate. | RF-1.5, RF-11.6, RNF-8 |
| **P13** Configured values are never defaulted | Page size and search bounds (this plan), invitation expiry (`InvitationPolicy` **[008]**) and step-up attempts and expiry (`EnrollmentPolicy` **[003]**) are validated configuration; startup fails on any absent or unbounded value. | RNF-12 |
| **P14** Release bar | WCAG 2.2 AA on R13, its two tables, the dialogs and panels with automated and manual assistive evidence; EN / ES / FR for the surface and the eight messages. | RF-12, RNF-15–17 |
| **F-X-01** C10 context | Every endpoint of this feature derives C10 through plan 009's `DeriveAuthorityUseCase` for the requested tenant; every command re-asserts the actor's membership under lock inside its transaction through `AuthorityChecker` **[009]**; a change takes effect at the affected person's next derivation. | RF-1.3, RF-5.6, RF-7.9, RF-10.6, RNF-6 |
| **F-X-02** Request integrity | Transfer, role change, suspension, reinstatement, removal, invitation issue, resend and revoke are consequential writes with no generic auto-retry; each carries an idempotency key and the versions or states it was opened against; the outcome is recorded and returned on re-submission. | RF-5.7, RF-10.4, RNF-13 |
| **F-X-03** Secrets and safe problems | The step-up password and code exist only in the request and the verifiers; the invitation secret only in the message and as a hash at rest; problem details carry safe codes with administrator-level reasons, field paths and a correlation id, and never a fact about another identity's security; browser storage holds nothing. | RF-13.2, RF-13.3, RF-13.5, RNF-10 |
| **F-X-05** Audit log | Every event of RF-13.1 is appended through `AuditPort` in the effect's transaction with actor, target, previous and new value, version and correlation id; "Decision history" reads it under the matrix's `READ_AUDIT` class **[009]**. | RF-3.4, RF-13.1 |
| **F-X-06** Configuration validation and email port | `TenantMembersPolicyProperties` is part of startup validation; the eight messages go through `TransactionalEmailPort` behind the outbox, so a delivery failure never reverts a change. | RF-13.4, RNF-14 |
| **F-X-07** Responsive surfaces | R13 has no drawn counterpart (S-02); the page, its tables as records, the dialogs and the panels reflow under the AA rules with `AuthorityStatements` **[009]** shared across widths; mobile navigation is the M04 pattern of the app shell. | RF-12 |
| **F-SET-01** Settings write discipline | Every write confirms persistence before the FE shows success (the response is the committed state) and records actor, previous value, new value and the resulting version. | RF-13.6 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access` (shared with plans 008 and 009), plus one event in plan 008's use cases

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 008 and 009. This plan adds one lock aggregate, five pure rules, three read models, nine use cases and queries, one event handler, one durable job and one controller; it extends plan 008's `Invitation` with its issuing side and plan 003's step-up with one purpose. Contexts touched — `identity`, `tenant`, `project`, `audit`, `sharedsupport` — are reached only through published ports (allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `TenantOwnership` | One row per tenant, created with the tenant (plan 012's bootstrap writes it in the same transaction as the founding Owner): `tenantId`, `lockVersion`, `lastOwnerChangeAt`. It carries no business data; locking it `FOR UPDATE` is the serialisation point of every Owner-affecting change (FR-IDN-095). `activeOwnerCount()` is computed by the repository under that lock from `tenant_membership` joined with identity status (D-a), never cached on the row. | RF-5.2, RF-5.3, RF-10.2, RF-10.3, RNF-2, RNF-4, LC-4 |
| | `ActiveOwnerRule` | Pure predicate `(membership, identityStatus) → boolean`: role `OWNER` ∧ membership `ACTIVE` ∧ identity `ACTIVE` (A-27). The only definition of "active Owner" (D-a). | RF-10.1, LC-26 |
| | `LastOwnerGuard` | Pure rule `(activeOwnersBefore, change) → Permit \| Deny(LAST_OWNER)` where `change` is one of `DEMOTE_OWNER`, `SUSPEND_OWNER`, `REMOVE_OWNER`, `TRANSFER` (which adds one Owner before removing one, so it never denies). Evaluated only under the `TenantOwnership` lock; the count and the write belong to one transaction. | RF-5.3, RF-5.8, RF-6.3, RF-7.3, RF-10.1–10.3, LC-1–LC-4, LC-26 |
| | `OwnerAffectingRule` | Pure classification `(targetRole, newRole \| action) → ownerAffecting: boolean`: promote to Owner, demote an Owner, suspend or remove an Owner, transfer → `true`; everything else → `false`. Decides whether a step-up and the ownership lock are required (Q7). | RF-6.3, RF-6.4, RF-7.3, RNF-5, LC-14, LC-15 |
| | `SuccessorEligibilityRule` | Pure rule `(candidate membership, identity facts, hasActiveFactor, actorMembershipId) → eligible: boolean`: membership `ACTIVE` ∧ identity `ACTIVE` with verified email ∧ active factor ∧ candidate ≠ actor (Q3). Used by the picker query, by the transfer recheck and by promotion to Owner. | RF-4.2, RF-4.3, RF-5.2, RF-6.3, LC-5–LC-8, LC-29 |
| | `AdministrationRule` **[009]** | Reused unchanged: Owner may change any role and create Owners; Admin may set `MEMBER` or `ADMIN` on non-Owner targets, never touch an Owner, never change own role; Owner lowers own only through transfer. This plan applies it to the row actions, the role dialog, the membership dialog and the invitation's role bound. | RF-2.4, RF-4.4, RF-6.1, RF-6.2, RF-7.2, RF-8.2, RF-10.5, RF-11.3, LC-12–LC-14, LC-34 |
| | `Invitation` **[008 → 010]** | Extended with the issuing side plan 008 left open: `issue(tenantId, email, role, grantSet?, issuedBy, now, policy, secretHash)` (state `PENDING`), `resend(newSecretHash, now, policy)` (new hash, new `expiresAt`, `resentAt`; allowed only while `PENDING`), `revoke(by, now)` (→ `REVOKED`), `expire(now)` (→ `EXPIRED`). `tenantRole` stays `ADMIN \| MEMBER` by type; the grant set stays "at most one project" (A-52). | RF-8.1, RF-8.5, RF-9.3–9.5, RNF-9, LC-35–LC-37 |
| | `InvitationIssueRule` | Pure rule `(normalisedEmail, existingMembership?, pendingInvitation?, tenantStatus, actorRole, requestedRole, requestedGrants) → Permit \| Deny(reason)` with reasons `MEMBERSHIP_EXISTS` (active or suspended), `INVITATION_PENDING`, `TENANT_NOT_ACTIVE`, `ROLE_NOT_PERMITTED` (Owner, or beyond `AdministrationRule`), `SELF_ADDRESS` (the actor's own address). A `REVOKED` membership never denies (Q9). | RF-8.2, RF-8.4, RF-11.3, RF-11.7, LC-19, LC-31–LC-34 |
| | `MemberRow`, `InvitationRow`, `MembersFilter` | Read models: `MemberRow` = membership id, display name, verified address, `isYou`, role, status, `totpRequired` (role Owner or Admin, or the tenant policy's Member MFA flag from `TenantPolicyPort` **[009]**), `permissionVersion`, `actions` (the subset of `TRANSFER_OWNERSHIP`, `EDIT_ROLE`, `MANAGE_MEMBERSHIP` computed by `AdministrationRule` for the actor); `InvitationRow` = invitation id, address, role, project and grants or "membership only", issuer display name, `issuedAt`, `expiresAt`, `actions = {REVIEW}`; `MembersFilter` = search text (name or address, case-insensitive), role filter, state filter (`ACTIVE`, `SUSPENDED`, `INVITATION_PENDING`, `ALL`), page. Never a password, factor, count, other tenant or project grant. | RF-2.1–2.8, RF-9.1, D-d, LC-22, LC-28 |
| | `TenantMembersPolicy` | Immutable record: members page size, search minimum and maximum length. Invitation expiry stays in `InvitationPolicy` **[008]**; step-up expiry and attempts in `EnrollmentPolicy` **[003]**. | RNF-12 |
| `access.application` | `TenantMembersQuery` (declared by plan 009, implemented here) | Under the actor's C10 (`TENANT_MEMBERS` class **[009]**): memberships of the tenant in `ACTIVE` or `SUSPENDED` state, plus pending invitations, filtered and paged, each row with its actions for this actor. Reads identity display names and addresses through `IdentityFactsPort` **[008]**, factor status is **not** read here (never shown), the policy flag through `TenantPolicyPort` **[009]**. | RF-1.3, RF-1.5, RF-2.1–2.8, RF-9.1, LC-21, LC-22, LC-28 |
| | `SuccessorsQuery` | The picker of N23: active memberships of the tenant whose identity is `ACTIVE` and verified (`IdentityFactsPort`) and has an active factor (`FactorStatusPort` **[003]**), excluding the actor; returns display names only and the "only verified members with an enrolled authenticator" statement key; never the reason a member is absent. Owner-only. | RF-4.2–4.4, LC-7, LC-8, LC-29 |
| | `TransferOwnershipUseCase` | Owner-only. Steps, all in one transaction after the step-up (D3): (1) recorded outcome for the idempotency key; (2) `StepUpPort.consume(pendingId, actorIdentity, purpose = TENANT_OWNERSHIP)` — a challenge that is not `SUCCEEDED` for this actor and purpose, or already consumed, refuses; (3) lock `TenantOwnership` `FOR UPDATE`; (4) `AuthorityChecker.require(C10, TENANT_MEMBERS)` **[009]** re-reads the actor's membership: still `ACTIVE` and `OWNER`, version as submitted; (5) load the successor under lock and apply `SuccessorEligibilityRule`; `TenantStatusPort` active; any failure → `access.transfer.refused{reason}` with the stated reason and nothing changed; (6) `MembershipCommands.ChangeTenantRole(successor → OWNER)` then `ChangeTenantRole(actor → ADMIN)` **[009]** in this same transaction, so the active-Owner count never drops; (7) enqueue `ownership-received` (successor) and `ownership-transferred` (actor) through `SecurityNotificationPort` **[002]**; (8) audit `OWNERSHIP_TRANSFERRED{actor, successor, versions}`; (9) record the outcome. Returns the two new roles and versions. No grant, session, identity or other tenant is touched. | RF-4.4, RF-5.1–5.8, RNF-2–RNF-6, LC-1, LC-5, LC-6, LC-9–LC-12, LC-24 |
| | `EditTenantRoleUseCase` | Owner or Admin; target ≠ actor (`AdministrationRule`); `OwnerAffectingRule` decides: if Owner-affecting → `StepUpPort.consume(…, TENANT_OWNERSHIP)` and the `TenantOwnership` lock, `SuccessorEligibilityRule` for promotion to Owner, `LastOwnerGuard` for demotion of an Owner; otherwise a plain confirmation. Then `AuthorityChecker.require` (actor still permitted, versions as submitted), `MembershipCommands.ChangeTenantRole` **[009]**, `tenant-role-changed` notification, audit, outcome. Refusals: `access.forbidden-change` **[009]** (scope), `access.last-owner`, `access.transfer.refused{reason}` (eligibility), `access.denied-scope` **[006/009]** (stale version). | RF-6.1–6.7, RF-10.1, RF-10.4–10.6, LC-3, LC-4, LC-13–LC-15, LC-26, LC-27, LC-29 |
| | `ManageMembershipUseCase` | Owner or Admin; target ≠ actor; action `SUSPEND \| REINSTATE \| REMOVE`; when the target is an Owner → step-up, lock, `LastOwnerGuard`; then `AuthorityChecker.require`, the matching `MembershipCommands` operation **[009]** (`SuspendMembership`, `ReinstateMembership`, `RevokeMembership` — which cascades grants, advances the version and enqueues `AuthorityChanged` for streams and work), the matching notification, audit, outcome. Sessions are never touched (`RevokeSessionsPort` **[004]** is not a dependency of this feature). | RF-7.1–7.10, RF-11.5, LC-16–LC-19 |
| | `IssueInvitationUseCase` | Owner or Admin; normalises the address with `EmailAddress` **[002]**; loads the identity's membership in the tenant by address through `IdentityFactsPort.lookupByEmail` (an internal read whose result never reaches the client beyond the two refusal reasons) and the pending invitation by `(tenant, email)`; applies `AdministrationRule` to the requested role and grants and `InvitationIssueRule`; refuses with `access.invitation.conflict{MEMBERSHIP_EXISTS \| INVITATION_PENDING \| TENANT_NOT_ACTIVE \| ROLE_NOT_PERMITTED \| SELF_ADDRESS}`; otherwise in one transaction: `TokenGenerator` **[002]** 256-bit secret, `TokenHasher` **[002]**, `Invitation.issue`, `InvitationRepository.save` **[008]**, enqueue the `invitation` message (locale header, link on the public origin of `SignupPolicy` **[002]**), audit `INVITATION_ISSUED{scope}`, outcome. Creates no membership, grant or identity. | RF-8.1–8.7, RF-11.4, RF-11.7, RNF-9, RNF-10, LC-31–LC-34 |
| | `ReviewInvitationQuery` | Owner or Admin: the pending invitation's scope element by element, issuer display name, `issuedAt`, `expiresAt`; a non-pending invitation is not found. | RF-9.2 |
| | `ResendInvitationUseCase` | Owner or Admin; lock the invitation; `PENDING` required (D-d: expired rows are already `EXPIRED`); new secret and hash, `Invitation.resend`, enqueue the `invitation` message again, audit `INVITATION_RESENT`, outcome. The scope is unchanged by construction (no scope input). | RF-9.3, RF-9.7, LC-25, LC-35 |
| | `RevokeInvitationUseCase` | Owner or Admin; conditional `UPDATE … SET state = 'REVOKED', revoked_by, revoked_at WHERE state = 'PENDING'` — zero rows → `access.invitation.conflict{NOT_PENDING}`; audit `INVITATION_REVOKED`; no message; outcome. The same conditional update is what makes a revoke racing an acceptance yield one transition (plan 008 D4). | RF-9.4, RF-9.7, LC-36 |
| | `ExpireInvitationsJob` | Durable job (`sharedsupport/jobs` **[002]**): claims `PENDING` invitations past `expiresAt` in bounded batches under each tenant's context, `Invitation.expire`, audit `INVITATION_EXPIRED`, no message. Check-on-access in plan 008's `AcceptanceRule` enforces expiry first; the job only settles the rows so the lists are exact. | RF-9.5, LC-37 |
| | `InvitationSettledHandler` | Consumes the outbox event `InvitationSettled{invitationId, tenantId, outcome = ACCEPTED \| DECLINED, issuedBy}` — one line added to plan 008's `AcceptInvitationUseCase` and `DeclineInvitationUseCase`: enqueue the event in their transaction (§6 cross-plan item) — and enqueues the `invitation-settled` notification to the issuer's verified address (via `IdentityFactsPort`, regardless of the issuer's current membership), audits `NOTIFICATION_ENQUEUED`. Accepted rows leave the lists by state, with no action here. | RF-9.6, RF-13.4, LC-38, LC-39 |
| | `AuthorityChanged` **[009]** | Not re-implemented: plan 009's `MembershipCommands` enqueue it and `AuthorityChangedHandler` closes streams and refuses or cancels work; this plan relies on it for suspension and removal. | RF-7.4, RF-7.9, RF-11.5, LC-16 |
| | Ports | New: `TenantOwnershipRepository` (`lockForUpdate(tenantId)`, `activeOwnerCount(tenantId)` under the lock), `StepUpPort` (→ plan 003's `StepUpUseCase`: `consume(pendingId, identityId, purpose)` marks a `SUCCEEDED` challenge consumed by this operation, exactly as a pending enrollment consumes it there), `InvitationSettled` event (published by plan 008's use cases, consumed here). Reused: `MembershipCommands`, `AuthorityPort` / `AuthorityChecker`, `MembershipRepository`, `TenantStatusPort`, `TenantPolicyPort`, `ProjectExistsPort` **[009]**; `InvitationRepository`, `InvitationPolicy` **[008]**; `IdentityFactsPort` **[008]** extended here with `lookupByEmail(normalisedEmail) → identityId?` (implemented by plan 008's identity facts adapter; its result never reaches the client); `FactorStatusPort` **[003]**; `TokenGenerator`, `TokenHasher`, `TransactionalEmailPort` (behind the outbox), `SecurityNotificationPort`, `JobScheduler` **[002]**; `SessionAssertionPort` **[006]**; `AuditPort`, `Clock` **[001]**. No port reaches a session-revocation, credential, factor-mutation, approval, run or content table. | all |
| `access.infrastructure` | Persistence | New `tenant_ownership` (`tenant_id` primary key, `lock_version`, `last_owner_change_at`; one row per tenant, inserted by the bootstrap of plan 012 and backfilled for existing tenants by this migration). `tenant_invitation` **[008]** gains `resent_at`, `revoked_by`, `revoked_at`, `expired_at` and a **partial unique index** on `(tenant_id, invited_email_normalised) WHERE state = 'PENDING'` (one pending per address, Q9). `step_up_challenge` **[003]** accepts the purpose value `TENANT_OWNERSHIP` and gains `consumed_by_operation` beside `consumed_by_pending`. All rows tenant-scoped under `FORCE RLS` with `USING` and `WITH CHECK` on `tenant_id` **[009]**. Persistence access API and migration tool per A-35 (placeholders). | RF-5.3, RF-8.4, RF-8.5, RF-10.2, RNF-2, RNF-4, RNF-9, LC-4, LC-32 |
| | Step-up adapter | Implements `StepUpPort` against plan 003's `StepUpUseCase` and `StepUpRepository`: the FE runs the step-up through plan 003's endpoint with `purpose = TENANT_OWNERSHIP`; the succeeded challenge id travels in the confirmation and is consumed here under the ownership lock; a challenge of another purpose, another identity, or already consumed refuses. The step-up itself (password + TOTP code, generic failure, sign-in throttle keys, attempt limit, expiry, recovery code refused) is plan 003 D5 unchanged. | RF-5.1, RF-6.3, RF-6.5, RF-7.3, RNF-5, LC-9, LC-10 |
| | Outbox relay and email adapter **[002]** | Eight templates in EN / ES / FR — `invitation` (tenant, inviter display name, role, project and grants or "membership only", link on the public origin, expiry, "only the local identity holding this address can accept"), `invitation-settled` (tenant, accepted or declined, no link, no content), `ownership-received`, `ownership-transferred`, `tenant-role-changed`, `membership-suspended`, `membership-reinstated`, `membership-removed` — content per S-14 (§6); language from the request header; the settled notification uses the issue request's locale stored on the invitation. Relay failures are retried by the relay, never surfaced. | RF-5.5, RF-6.6, RF-7.8, RF-8.5, RF-9.3, RF-9.6, RF-13.4, RNF-14, LC-25 |
| | `TenantMembersPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding into `TenantMembersPolicy`; constraints: present, positive, bounded above by a configured ceiling. Any violation fails startup. | RNF-12 |
| | Audit adapter | Events: `OWNERSHIP_TRANSFERRED{actor, successor, previousRoles, newRoles, versions}`, `TENANT_ROLE_CHANGED` / `MEMBERSHIP_SUSPENDED` / `MEMBERSHIP_REINSTATED` / `MEMBERSHIP_REVOKED` (written by plan 009's commands, with this feature's correlation id), `CHANGE_REFUSED{reason}` (last-Owner, eligibility, stale, scope, invitation conflict), `STEP_UP_CONSUMED{purpose = TENANT_OWNERSHIP}` (the step-up's own success / failure / void events are plan 003's), `INVITATION_ISSUED{scope}`, `INVITATION_RESENT`, `INVITATION_REVOKED`, `INVITATION_EXPIRED`, `NOTIFICATION_ENQUEUED{template}`, `NOTIFICATION_FAILED{template}`. Payload: tenant, acting membership and identity, target membership, invitation or identity, previous and new value, resulting version, correlation id, time. Never a password, code, secret or project content. | RF-13.1, RF-13.2 |
| | Problem codes | `access.forbidden-change` (403) **[009]** for a change `AdministrationRule` refuses; `access.last-owner` (409) with the stated reason and the transfer route; `access.transfer.refused` (409) `{reason: SUCCESSOR_INELIGIBLE \| ACTOR_CHANGED \| TENANT_NOT_ACTIVE}`; `access.invitation.conflict` (409) `{reason}` as in `InvitationIssueRule` plus `NOT_PENDING`; `access.step-up.required` (428) when an Owner-affecting change arrives without a consumed challenge; `access.denied-scope` (403) **[006/009]** for a stale version or lost authority. All carry a correlation id; the reasons are the administrator-level reasons RF-13.3 permits and never name another identity's security facts. | RF-5.2, RF-6.5, RF-7.7, RF-8.4, RF-10.1, RF-10.4, RF-13.3 |
| `access.api` | `TenantMembersController` — `/api/tenants/{tenantId}/members` (C10 filter **[009]**; class `TENANT_MEMBERS`; a Member receives 403 `access.denied-scope`) | `GET ?search&role&state&page` → 200 `{rows: MemberRow[] ∪ InvitationRow[], total, page}` · `GET successors` → 200 `{candidates[], statementKey}` (Owner only) · `POST ownership/transfer` {successorMembershipId, stepUpPendingId, actorVersion, successorVersion, idempotencyKey} → 200 `{actor: {role, version}, successor: {role, version}}` or 409 / 428 / 403 · `POST {membershipId}/role` {newRole, stepUpPendingId?, targetVersion, actorVersion, idempotencyKey} → 200 `{role, version}` · `POST {membershipId}/suspend \| reinstate \| remove` {stepUpPendingId?, targetVersion, actorVersion, idempotencyKey} → 200 `{status, version}` · `GET invitations?page` → 200 `{rows: InvitationRow[], total, page}` · `POST invitations` {email, role, project?: {projectId, roles[], canRun}, idempotencyKey} → 201 `{invitationId, expiresAt}` or 400 field paths or 409 `access.invitation.conflict` · `GET invitations/{id}` → 200 the review shape · `POST invitations/{id}/resend` {idempotencyKey} → 200 `{expiresAt}` · `POST invitations/{id}/revoke` {idempotencyKey} → 200 `{state: REVOKED}`. The step-up runs on plan 003's `POST /api/account/mfa/step-up` with `purpose = TENANT_OWNERSHIP`, returning `{pendingId}` on success. The UI locale travels in the request header. All problems are RFC 9457 problem details with a safe code, field paths and a correlation id. The server never redirects. | RF-1.3, RF-2, RF-4–RF-9, RF-13.3 |

**Cross-context touch points.** `identity` — `IdentityFactsPort` **[008]** (display names, addresses, status, verification), `FactorStatusPort` **[003]**, `StepUpPort` (→ plan 003's step-up), `TokenGenerator` / `TokenHasher` **[002]**. `tenant` — `TenantStatusPort` **[009]**. `project` — `ProjectExistsPort` **[009]** for the invitation's project. `audit` — append-only. `sharedsupport` — `crypto`, `jobs`, `tx`, `contextIds`, `outbox`. Nothing reads or writes session, credential, factor-mutation, approval, run or content tables.

**Data model note (P12).** `tenant_ownership`, `tenant_invitation`, `tenant_membership` and `project_grant` are tenant rows under RLS; the lock row is the one row per tenant that every Owner-affecting change takes `FOR UPDATE`, which is what turns "count then write" into one protected operation.

### 2.2 Front end — feature module `src/features/workspace-members` (new; the FE section's "workspace / memberships" feature)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/t/{tenantId}/settings/people` (R13) with the tabs as nested states: `?tab=members` (default), `?tab=invitations`; "Workspace policy" is a route of the settings feature (F-SET-05). Reached from the app shell's Settings entry under the actor's current tenant. | RF-1.1, RF-1.2 |
| | `useTenantMembers()`, `useSuccessors()`, `useTransferOwnership()`, `useEditRole()`, `useManageMembership()`, `useIssueInvitation()`, `useReviewInvitation()`, `useResendInvitation()`, `useRevokeInvitation()`, `useStepUp()` **[003]** | Typed intent hooks; attach the UI locale header; return server state verbatim; the mutations generate their idempotency key once and reuse it on re-submission. | RF-2–RF-9 |
| | `contracts/members` (in `src/contracts`) | Request / response / problem types and runtime guards for the ten endpoints and the six problem codes; a guard rejects any row carrying a password, factor state, code count, other tenant or project grant. | all |
| Views | `PeoplePage` | R13: breadcrumb, heading "People in {tenant}", subtitle, "Invite member", the three tabs, search with the shortcut, role and state filters, the table of the active tab, the pagination line, the notice and card of RF-3, "Review project grants", "Decision history ↗", language switch; the Admin variant renders whatever actions the rows carry (S-49: none on Owner rows, none on the own row, no transfer). | RF-1.1, RF-1.2, RF-2.5–2.7, RF-3.1–3.5, S-49 |
| | `MembersTable` | Columns MEMBER (display name, "you"), TENANT ROLE, MEMBERSHIP (Active / Suspended / Invitation pending with the "TOTP required" marker), ADMINISTRATION (the actions the row carries, rendered as links; a cell with two actions renders both); records at narrow widths. | RF-2.1–2.4, RF-12.1, D-d |
| | `PendingInvitationsTable` | Address, role, project and grants or "membership only", issuer, issued, expires, "Review invitation". | RF-9.1 |
| | `TransferOwnershipDialog` | N23 verbatim: statement, authority line, `SuccessorPicker` (from `useSuccessors`, with the eligibility statement and the "no eligible member" state), `StepUpFields` (password with Show, code with paste), "Verify & transfer ownership" (runs the step-up, then the transfer with the consumed challenge id and the versions the dialog opened against), "Keep current Owner"; refusal reasons rendered from the problem code's i18n key. | RF-4.1–4.6, RF-5.1, RF-5.2, LC-5, LC-7–LC-9 |
| | `EditRoleDialog` | On N23's anatomy (S-50): member, current role, the new-role choice limited to what the server's row allows, the effect statements, `StepUpFields` shown only when `OwnerAffectingRule` (mirrored from the contract) says so, confirm / cancel. | RF-6.1–6.4, LC-14, LC-15 |
| | `ManageMembershipDialog` | On N23's anatomy: member, state, the available actions (suspend / reinstate / remove), the effect statement of the chosen action, `StepUpFields` when the target is an Owner, confirm / cancel. | RF-7.1–7.3, LC-16–LC-18 |
| | `InviteMemberDialog` | On N23's anatomy: address, role (Admin / Member), optional project choice (one project of the tenant from the project list of plan 011's query), Author / Reviewer / Viewer and canRun as independent unticked choices, the statements, "Send invitation" / "Cancel"; conflict reasons rendered from the problem code. | RF-8.1–8.4, LC-31–LC-34 |
| | `ReviewInvitationPanel` | Scope element by element, issuer, issued, expiry, "Resend invitation" (confirmation stating the earlier link stops working), "Revoke invitation" (confirmation stating it is final), "Close". | RF-9.2–9.4 |
| | `AuthorityStatements` **[009]** | The `members` variant renders RF-3.1 and RF-3.2 verbatim; the dialogs reuse its "membership is not project access" line. | RF-3.1, RF-3.2, RF-3.5, RF-6.1, RF-8.1 |
| | `StepUpFields` | New wrapper around `PasswordField` and `CodeField` **[001]** that reproduces the step-up block of plan 003's `RegenerateCodesDialog` and `ReplaceAuthenticatorDialog`, submits through `useStepUp()` **[003]** with the purpose `TENANT_OWNERSHIP`, renders the generic failure of `mfa.step-up.rejected` and handles the void: a voided step-up closes the dialog with nothing changed. | RF-5.1, LC-9, LC-10 |
| State | TanStack Query | Queries for the members list (keyed by tenant, tab, filter, page, scope epoch), successors, review; mutations for the eight writes (no retry, F-X-02); every mutation invalidates the list and plan 009's authority store when the actor's own row changed. | RF-2.8, RF-5.6, RNF-13 |
| | `useAuthority(TENANT_MEMBERS)` **[009]** | Gates the page: a Member or suspended member is routed to the denied-scope state by `DeniedScopeHandler` **[006]**; the row actions come from the server rows, not from this mirror. | RF-1.3, LC-21 |
| | Dialog state | Module-private in-memory records per open dialog (target id, versions the dialog opened against, chosen action, consumed challenge id); cleared on close, success, void and route leave; nothing in storage. | RF-10.4, RF-13.5, LC-23 |
| Routing | Handoffs | "Review project grants" → plan 011's grants surface for the current project; "Workspace policy" → the settings route; "Decision history ↗" → the audit trail (F-APR-04, N18) with the tenant membership filter; a stale-version refusal → `DeniedScopeHandler` **[006]** refreshes authority once and the list reloads; a suspended or deleting tenant → N20 (plan 014) by the app shell; after a transfer the actor's own authority store refreshes and the page re-renders the rows the server now offers. | RF-1.4, RF-3.3, RF-3.4, RF-5.6, RF-10.4, LC-11, LC-12, LC-20 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/errors` (field paths; `DeniedScopeHandler` **[006]**; problem-code to i18n reason mapping), `shared/i18n` (bundle keys for every statement, state, action, reason and message), `shared/ui` (accessible `Table` with record reflow, `Dialog`, `Field`, `PasswordField`, `CodeField` **[001]**, `Checkbox`, `Tabs`, `Pagination` **[007]**, `LiveRegion`, 44 px targets). | RF-1.6, RF-12.1–12.3, RNF-15, RNF-16 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `TenantMembersQuery` | `PeoplePage` with tabs, search, filters, tables, statements, routes, switch |
| RF-1.2 | — | "Workspace policy" tab routes to the settings feature |
| RF-1.3 | C10 filter **[009]** + `AuthorityChecker.require(TENANT_MEMBERS)`; 403 `access.denied-scope` otherwise | `useAuthority` gate; `DeniedScopeHandler` **[006]** |
| RF-1.4 | `TenantStatusPort` not `ACTIVE` → every command refuses `TENANT_NOT_ACTIVE`; derivation yields `TENANT_NOT_ACTIVE` **[009]** | App shell shows N20 (plan 014) |
| RF-1.5 | RLS tenant predicate; queries scoped by C10 tenant | No cross-tenant route |
| RF-1.6 | Nothing server-side depends on the locale except template language | i18n re-render; dialog state untouched |
| RF-1.7 | — | Help entry is the app shell's, opening N43 (S-17) |
| RF-2.1 | `MemberRow`, `InvitationRow` from `TenantMembersQuery` | `MembersTable` |
| RF-2.2 | Query predicate: membership `ACTIVE \| SUSPENDED`; invitation `PENDING` | — |
| RF-2.3 | `MemberRow` carries display name and address only; `FactorStatusPort` never read by the list | Contract guard rejects extra fields |
| RF-2.4 | `AdministrationRule` **[009]** computes `actions` per row for the actor | Cells render the server's actions |
| RF-2.5 | `MembersFilter`: case-insensitive name or address, role, state; `TenantMembersPolicy.pageSize`; total in the response | Search field, filters, pagination line |
| RF-2.6 | State filter enumeration excludes revoked and closed | Filter options |
| RF-2.7 | Empty page returns `total = 0` | Empty state with filters editable |
| RF-2.8 | Every command compares the submitted versions / state under lock | Dialog carries the versions it opened against |
| RF-3.1 | — | `AuthorityStatements` **[009]** `members` variant (last-Owner notice) |
| RF-3.2 | — | `AuthorityStatements` `members` variant (card, sub-statements, "Review project grants") |
| RF-3.3 | — | Route to plan 011's surface |
| RF-3.4 | Audit entries carry the tenant membership lineage; `READ_AUDIT` **[009]** scopes the read | Route to N18 with the membership filter |
| RF-3.5 | — | One component at every width and variant |
| RF-4.1 | `SuccessorsQuery`; step-up purpose `TENANT_OWNERSHIP` (new purpose of plan 003's `StepUpChallenge`) | `TransferOwnershipDialog` as N23 |
| RF-4.2 | `SuccessorEligibilityRule` with `IdentityFactsPort` and `FactorStatusPort`; names only | `SuccessorPicker` with the statement |
| RF-4.3 | Empty candidate list | "No eligible member" state; action unavailable |
| RF-4.4 | `SuccessorsQuery` and `TransferOwnershipUseCase` require role `OWNER` under C10; `AdministrationRule` | Own row offers transfer only for an Owner |
| RF-4.5 | No call before confirmation | "Keep current Owner" / close → nothing |
| RF-4.6 | Response carries display names only | Password and code never stored |
| RF-5.1 | `StepUpUseCase` **[003]** (password + TOTP, generic, throttled, bounded); `StepUpPort.consume` | `StepUpFields`; void closes the dialog |
| RF-5.2 | Lock → `AuthorityChecker` → `SuccessorEligibilityRule` → `TenantStatusPort`; `access.transfer.refused{reason}` | Reason rendered; nothing changed |
| RF-5.3 | Two `ChangeTenantRole` **[009]** calls in one transaction under the lock; audit `OWNERSHIP_TRANSFERRED` | — |
| RF-5.4 | No grant, session, identity or other-tenant write in the use case | — |
| RF-5.5 | `ownership-received`, `ownership-transferred` via `SecurityNotificationPort` | — |
| RF-5.6 | Next derivation reads the new roles (plan 009 D3) | Authority store refresh; rows re-rendered |
| RF-5.7 | Idempotency outcome recorded in the transaction | Key generated once |
| RF-5.8 | `LastOwnerGuard(TRANSFER)` never denies; order successor-first | — |
| RF-6.1 | `EditTenantRoleUseCase`; `AdministrationRule` bounds the offered roles | `EditRoleDialog` with effect statements |
| RF-6.2 | `AdministrationRule` denies `actorIsTarget` | Own row offers no edit |
| RF-6.3 | `OwnerAffectingRule` → step-up + lock; `SuccessorEligibilityRule` for Owner; `LastOwnerGuard(DEMOTE_OWNER)` | `StepUpFields` shown for Owner-affecting changes |
| RF-6.4 | No step-up for Member ↔ Admin | Confirmation only |
| RF-6.5 | Step-up failure per plan 003; rechecks with stated reasons | Reason rendered |
| RF-6.6 | `ChangeTenantRole` **[009]**; `tenant-role-changed`; audit | — |
| RF-6.7 | No factor check for Admin; `EnrollmentGate` **[003]** at tenant entry | — |
| RF-7.1 | `ManageMembershipUseCase` actions by current state | `ManageMembershipDialog` |
| RF-7.2 | `AdministrationRule` denies self | Own row offers no management |
| RF-7.3 | Owner target → step-up + lock + `LastOwnerGuard` | `StepUpFields` for an Owner target |
| RF-7.4 | `SuspendMembership` **[009]** → version, `AuthorityChanged` → streams closed, work refused / cancelled | Row reads Suspended |
| RF-7.5 | `ReinstateMembership` **[009]**: grants unchanged, new version | — |
| RF-7.6 | `RevokeMembership` **[009]**: cascade, version; row leaves the list | — |
| RF-7.7 | Rechecks under lock with stated reasons | Reason rendered |
| RF-7.8 | Audit by plan 009's command; `membership-*` notifications | — |
| RF-7.9 | No session port in this feature; next derivation denies | `DeniedScopeHandler` for the affected person's tabs **[006]** |
| RF-7.10 | No reinstate path for `REVOKED`; `IssueInvitationUseCase` is the only way | — |
| RF-8.1 | `IssueInvitationUseCase` input shape | `InviteMemberDialog` |
| RF-8.2 | `AdministrationRule` + `Invitation.tenantRole` type excludes Owner; grants bounded | Role choice limited |
| RF-8.3 | 400 field-path problems | Field validation |
| RF-8.4 | `InvitationIssueRule`; `access.invitation.conflict{reason}` | Reason rendered |
| RF-8.5 | One transaction: secret, hash, `issue`, save, outbox `invitation`, audit | — |
| RF-8.6 | No membership, grant or identity write; `IdentityFactsPort.lookupByEmail` result never on the wire | — |
| RF-8.7 | `TenantMembersQuery` unions pending invitations | Row in both tables |
| RF-9.1 | `TenantMembersQuery` invitations tab | `PendingInvitationsTable` |
| RF-9.2 | `ReviewInvitationQuery` | `ReviewInvitationPanel` |
| RF-9.3 | `ResendInvitationUseCase`: new secret, new expiry, message again | "Resend invitation" with confirmation |
| RF-9.4 | `RevokeInvitationUseCase` conditional update; no message | "Revoke invitation" with confirmation; row removed |
| RF-9.5 | `ExpireInvitationsJob`; `InvitationPolicyProperties` **[008]** | — |
| RF-9.6 | Plan 008's accept / decline move the state; `InvitationSettledHandler` notifies the issuer | List reload shows the membership |
| RF-9.7 | Owner or Admin under C10; conditional update (plan 008 D4) | — |
| RF-9.8 | No use case reads the issuer's membership to decide validity | — |
| RF-10.1 | `LastOwnerGuard` → `access.last-owner` with the transfer route | Reason and route rendered |
| RF-10.2 | `TenantOwnership` lock `FOR UPDATE` | — |
| RF-10.3 | `activeOwnerCount` evaluated under the lock in the writing transaction | — |
| RF-10.4 | `AuthorityChecker` / `PermissionVersionGuard` **[009]** compare the submitted versions; invitation state checked under lock | Versions carried; `DeniedScopeHandler` refresh |
| RF-10.5 | `AdministrationRule` (Admin never Owner) + `InvitationIssueRule.ROLE_NOT_PERMITTED` | Admin UI offers none |
| RF-10.6 | `AuthorityChecker.require` inside every command | — |
| RF-11.1 | No `GrantCommands` **[009]** call anywhere in this feature; invitation grants materialise only in plan 008's acceptance | — |
| RF-11.2 | No credential, factor, session, address or erasure port in this feature | Statements of RF-3.2 |
| RF-11.3 | `AdministrationRule` self rows; `InvitationIssueRule.SELF_ADDRESS` | — |
| RF-11.4 | `CreateMembership` **[009]** is never called here | — |
| RF-11.5 | Effects only through `AuthorityChanged` **[009]** | — |
| RF-11.6 | No tenant, policy or other-tenant write; RLS | — |
| RF-11.7 | No lookup result on the wire beyond the conflict reasons; no domain logic | — |
| RF-12.1 | — | `PeoplePage` reflow; tables as records; dialogs full-screen on narrow widths; M04 navigation (shell) |
| RF-12.2 | — | `shared/ui` targets, labels, focus, keyboard-reachable action cells |
| RF-12.3 | — | Breakpoint layout replaceable (S-02) |
| RF-13.1 | Audit adapter events in the effect's transaction | — |
| RF-13.2 | Payload shape; redaction; secret only hashed | No secret in state or analytics |
| RF-13.3 | Problem codes with administrator-level reasons; no identity security facts | Reasons from the i18n bundle |
| RF-13.4 | Eight templates via the outbox; locale rules | — |
| RF-13.5 | — | Dialog state in memory only |
| RF-13.6 | Responses carry the committed state | Rows re-rendered from the response or a reload |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The last-Owner guard is a count under an exclusive lock on a one-row-per-tenant `tenant_ownership` table, inside the writing transaction.**
*Why.* FR-IDN-095 forbids a count-then-write race and asks for the ownership row to be locked; a dedicated lock row per tenant gives every Owner-affecting change one serialisation point without locking the whole membership table, and counting active Owners inside the same transaction that writes makes the guard and the write one protected operation (RF-10.2, RF-10.3). Plan 009 D7 placed exactly this here.
*Discarded.* A count without a lock — the race the requirement names. Locking every membership row of the tenant — contention on every member change. A stored Owner count — must be kept in sync with status and identity changes it cannot see.
*Covers.* RF-5.3, RF-5.8, RF-6.3, RF-7.3, RF-10.1–10.3, RNF-2, RNF-4, LC-3, LC-4, LC-26.

**D2 · A transfer is two `ChangeTenantRole` calls of plan 009 in one transaction, successor first, under the ownership lock.**
*Why.* Decision Q2: step-down in one indivisible change. Reusing plan 009's command keeps the version advance, the audit record and the `AuthorityChanged` event in one place; ordering the successor's promotion before the actor's demotion makes `LastOwnerGuard(TRANSFER)` trivially satisfied and keeps RF-5.3's "never fewer active Owners" true at every statement. The lock serialises it against any concurrent demotion (LC-4).
*Discarded.* A dedicated "swap roles" command in `access` — a second path that writes roles. Transfer as promotion only (co-ownership) — Q2's rejected option.
*Covers.* RF-5.3, RF-5.4, RF-5.6, RF-5.8, RNF-3, LC-1, LC-11, LC-12.

**D3 · The step-up is plan 003's challenge with a new purpose, run by the FE first and consumed by the command under the lock.**
*Why.* Decision Q4 names the step-up of specification 003 RF-8; plan 003 D5 already provides a bounded, purpose-bound challenge row that is consumed by the operation it authorises, counted on the sign-in keys, with recovery codes refused (D6 there). Adding the purpose `TENANT_OWNERSHIP` and a `consumed_by_operation` column reuses all of it; consuming inside the command's transaction means a succeeded challenge authorises exactly one Owner-affecting change and can never be replayed for another (RNF-5).
*Discarded.* A second step-up implementation — drift from 003's rules. A session-level "re-authenticated until" flag — a bypass window usable for any purpose (rejected in plan 003 D5). Password only — Q4's rejected option.
*Covers.* RF-5.1, RF-6.3, RF-6.5, RF-7.3, RNF-5, LC-9, LC-10, LC-15.

**D4 · Successor eligibility is one pure rule reading identity facts and factor status through published ports, applied by the picker, the transfer and the promotion to Owner.**
*Why.* Decision Q3 fixes three facts (active membership, active verified identity, enrolled factor) plus "not the actor"; one rule shared by `SuccessorsQuery`, `TransferOwnershipUseCase` and `EditTenantRoleUseCase` means the picker and the recheck cannot disagree, and the recheck under the lock catches anything that changed since the dialog opened (LC-5). The picker returns names only and a general statement, so nobody learns who lacks a factor (RF-4.2).
*Discarded.* Eligibility decided by the picker response alone — P3. Exposing per-member reasons — discloses factor state.
*Covers.* RF-4.2, RF-4.3, RF-5.2, RF-6.3, LC-5–LC-8, LC-29.

**D5 · One classification rule decides which changes are Owner-affecting, and that classification alone switches on the step-up and the lock.**
*Why.* Decision Q7: promote to Owner, demote an Owner, suspend or remove an Owner, and transfer. Encoding it once as `OwnerAffectingRule` and letting both use cases and the FE dialogs consult it keeps "which changes need a fresh proof" in one place and makes a routine Member change cheap (confirmation only).
*Discarded.* Step-up on every change — Q7's rejected option. Step-up only on transfer — an Owner appointed or removed with no fresh proof.
*Covers.* RF-6.3, RF-6.4, RF-7.3, RNF-5, LC-14, LC-15.

**D6 · Issuing, resending and revoking extend plan 008's `Invitation` aggregate; one pending invitation per address is a partial unique index; a resend replaces the secret.**
*Why.* Decisions Q8 and Q9 within A-52: plan 008 built the record and left the issuing side explicitly to this plan, so the aggregate gains its remaining transitions rather than a sibling. A partial unique index on `(tenant, address) WHERE state = 'PENDING'` makes "one pending per address" a database fact that two concurrent issues cannot violate (LC-32). A resend generates a fresh secret and expiry on the same row, so the earlier link fails plan 008's hash lookup and the pending row, its scope and its issuer are unchanged (LC-35). Revocation is the same conditional update plan 008 races against (LC-36).
*Discarded.* A new record per resend — several pending rows per address, the option Q9 rejected. Re-sending the same secret — a link that already leaked stays valid. Superseding on re-issue — Q9's rejected option.
*Covers.* RF-8.4–8.7, RF-9.3–9.5, RF-9.7, RNF-9, LC-19, LC-31, LC-32, LC-35–LC-37.

**D7 · Invitation conflicts are refused with administrator-level reasons; the existence of an identity for an address is never one of them.**
*Why.* RF-13.3 permits the reasons an administrator needs (existing membership, pending invitation, inactive tenant, role not permitted, own address), all of which the administrator can already see in the lists; RF-11.7 and Q9 forbid revealing whether an identity exists for an address, so an unknown address and a known address with no membership are issued identically (LC-33). The lookup by address that finds a membership is an internal read whose result reaches the wire only as those reasons.
*Discarded.* A generic answer for every conflict — an administrator could not act on it. A "no such user" reason — enumeration.
*Covers.* RF-8.4, RF-8.6, RF-11.7, RF-13.3, RNF-10, LC-31–LC-33.

**D8 · The inviter notification is driven by an `InvitationSettled` outbox event enqueued by plan 008's accept and decline use cases.**
*Why.* Decision Q10 wants the inviter told of acceptance and decline; plan 008 Q7 sends nothing itself and leaves the inviter notification to this plan. The transitions happen in plan 008's transactions, so the only atomic hook is an event enqueued there; this plan's handler turns it into the message to the issuer's verified address, regardless of the issuer's current membership (LC-39). The event is the one line plan 008 must add, recorded in §6.
*Discarded.* Polling the invitation table for state changes — a timing window and a second source. Sending from plan 008's use cases — plan 008 decided to send nothing and does not own the template.
*Covers.* RF-9.6, RF-13.4, LC-38, LC-39.

**D9 · The member list is one server query over memberships and pending invitations with the row actions computed on the server for the actor.**
*Why.* RF-2.4 requires exactly the actions the actor's role permits per row and S-49 requires the Admin variant to show none on Owner rows; computing `actions` with `AdministrationRule` on the server means the FE renders what it receives and the same rule that will later refuse a submission decided what was offered (P3). Including pending invitations as rows (D-d) is what R13 draws; the union is paged and filtered once.
*Discarded.* Computing actions in the FE from the viewer's role — a mirror that can drift from the rule the server applies. Two independent lists — two paginations and two filters for one drawn table.
*Covers.* RF-2.1–2.8, RF-9.1, RF-10.5, RNF-1, LC-13, LC-21, LC-22, LC-28.

**D10 · Every confirmation carries the permission versions, or the invitation state, the dialog opened against, and the command checks them under lock.**
*Why.* RF-10.4 and plan 009 D4: the race between an administrator's dialog and another administrator's change is lost by the stale confirmation; `PermissionVersionGuard` **[009]** already refuses a sensitive write on a moved version, and the invitation's conditional update does the same for its state. The refusal reaches the FE as `access.denied-scope`, which `DeniedScopeHandler` **[006]** turns into one authority refresh and a reloaded list.
*Discarded.* Last-writer-wins — the administrator acts on stale rows. Optimistic retry — a second decision taken without the person.
*Covers.* RF-2.8, RF-10.4, RF-10.6, RNF-11, LC-27.

**D11 · Membership changes never touch sessions; authority changes propagate through plan 009's `AuthorityChanged` event and the next derivation.**
*Why.* Decision Q6 keeps sessions alive; plan 009 D3 and D8 give denial at the next request and bounded effects on streams and work; FR-IDN-067 keeps accepted Runs under their own authority. This feature therefore depends on neither `RevokeSessionsPort` **[004]** nor the session service, and a person suspended in one tenant keeps working in the others.
*Discarded.* Revoking the member's sessions — Q6's rejected option; it would end their work in other tenants.
*Covers.* RF-7.4, RF-7.9, RF-11.5, RNF-6, RNF-7, LC-16, LC-17.

**D12 · Eight outbox templates in the locale of the triggering request; review-independent messages use the locale stored on the invitation.**
*Why.* RF-13.4 fixes the statements; the outbox gives atomicity with the effect (plan 002 D1) and keeps provider latency and failure out of the request (LC-25). Messages caused by a request take its locale header (plan 002 D15); the settled notification has no request from the issuer, so the issue request's locale stored on the invitation is used, as plan 005 D11 stores it on the case.
*Discarded.* Synchronous send — timing and failure coupled to the change. A configured default language for the settled notification — the issuer already told us their language.
*Covers.* RF-5.5, RF-6.6, RF-7.8, RF-8.5, RF-9.3, RF-9.6, RF-13.4, RNF-14, LC-25, LC-38.

**D13 · Expiry of invitations is enforced on access by plan 008 and swept by a durable job that settles the rows.**
*Why.* Plan 008's `AcceptanceRule` already refuses an expired invitation; the lists of this feature must drop expired rows without waiting for someone to open the link, and audit must record the expiry; the existing durable-jobs facility (plan 002 D9) does that in bounded batches under each tenant's context.
*Discarded.* Filtering expired rows at read time only — the row never settles and never audits.
*Covers.* RF-9.5, LC-37.

**D14 · One FE feature module with dialogs on N23's anatomy and a single page component for every role variant.**
*Why.* The FE section names "workspace / memberships" as a feature; R13 is the only screen and S-50 asks the dialogs to reuse N23's anatomy; S-49 says the Admin variant differs only in the actions offered, which D9 already supplies from the server. One page and one dialog family keep every statement identical across variants and widths (RF-3.5, RF-12.1).
*Discarded.* Placing the surface in `identity-access` — tenant administration is not identity; the FE DAG separates them. An Admin-specific page — drift.
*Covers.* RF-1.1, RF-3.5, RF-4.1, RF-6.1, RF-7.1, RF-8.1, RF-9.2, RF-12.1–12.3, RNF-15–RNF-17.

**D15 · The eight writes carry an idempotency key with a recorded outcome [001 D1]; the invitation issue uses the key so a repeated send issues once.**
*Why.* F-X-02 and RF-5.7: a reload during "transferring" or "sending" must not transfer twice or issue twice; the outcome recorded in the transaction is returned on re-submission. The partial unique index of D6 is a second line for issues.
*Discarded.* Generic POST retry — forbidden. No key on issue — a lost answer would create two invitations or show a false conflict.
*Covers.* RF-5.7, RNF-13, LC-24.

**D16 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id [001 D12]; the codes of this feature carry administrator-level reasons.**
*Why.* F-X-03; RF-13.3 wants the reason an administrator needs — last-Owner, eligibility, stale, scope, conflict — on the wire, while plan 009 keeps derivation reasons off it; the two sets of codes serve the two audiences, and the FE maps every code to an i18n reason.
*Discarded.* Reusing `access.denied-scope` for every refusal — the administrator could not tell a stale row from a guard.
*Covers.* RF-5.2, RF-6.5, RF-7.7, RF-8.4, RF-10.1, RF-13.3.

**D17 · Every configured number is one validated `TenantMembersPolicyProperties` record that fails startup; the others stay with their owners.**
*Why.* P13 and FR-X-046; page size and search bounds are this feature's; invitation expiry (plan 008) and step-up bounds (plan 003) are validated where they live, so no value is duplicated.
*Discarded.* Defaults in code — forbidden. Copying the expiry here — two sources.
*Covers.* RNF-12.

**D18 · Audit is appended through the `audit` context port in the effect's transaction [001 D11], and the row changes themselves are audited by plan 009's commands.**
*Why.* RF-13.1 lists transfers, refusals, invitations and messages beside the role and status changes; the latter are already written by `MembershipCommands` with this feature's correlation id, so one entry per change exists and the transfer, refusal and invitation events are this plan's additions. "Decision history" reads them under the matrix's `READ_AUDIT` class.
*Discarded.* A second audit entry for every command — duplicates. The logging framework as audit — not append-only.
*Covers.* RF-3.4, RF-13.1, RF-13.2, RF-13.6.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure rules: `LastOwnerGuard`, `ActiveOwnerRule`, `OwnerAffectingRule`, `SuccessorEligibilityRule`, `InvitationIssueRule`, `Invitation` issue / resend / revoke / expire, `MembersFilter`, row-action computation with `AdministrationRule` **[009]**. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: the ownership lock, the partial unique index, plan 009's commands, step-up consumption, outbox and audit rows, the expiry job, migrations and RLS under the runtime role. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes and reasons, correlation id, absence of factor state, counts, other tenants and project grants in every response; action cells per role. |
| Concurrency | JUnit 5 with real PostgreSQL | Two Owners demoting each other; transfer racing a suspension of the successor; two issues for one address; revoke racing plan 008's accept; two administrators on one row. |
| Security | Spring Boot test with the runtime role, real PostgreSQL | Cross-tenant reads and writes on the four tables refused; the lock row unreachable from another tenant's context. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| Module boundary | ArchUnit (**test-only**) | `access.application` depends on no Spring or persistence class and imports no session, credential, approval, run or content package. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page and dialog states, server-driven action cells, version carrying, step-up fields shown only when Owner-affecting, dialog state cleared, nothing in storage, statements at every width. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths with BE doubles and a fake email port; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Table navigation, action cells in reading order, dialog focus, announcements without focus theft, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RF-1.2, RF-1.7 | FE unit, e2e | `peoplePage.rendersHeadingTabsFiltersTablesStatementsAndRoutes` (desktop, tablet, mobile); `peoplePage.tenantPolicyTabRoutesToSettings` | 1, 13 |
| RF-1.3, LC-21 | API contract, FE e2e | `members.ownerAndAdminOnly`; `members.memberGetsDeniedScopeNoRows` | 3 |
| RF-1.4, LC-20 | BE integration, FE e2e | `members.inactiveTenantRefusesEveryCommand`; `peoplePage.suspendedTenantShowsN20` | 8 |
| RF-1.5, RNF-8 | Security | `rls.membersInvitationsAndLockRowTenantScoped` | 8 |
| RF-1.6, LC-23 | FE e2e | `languageSwitch.onPeopleKeepsDialogState` | 13 |
| RF-2.1, RF-2.2, RF-2.3 | API contract, BE integration | `membersList.rowsShapeAndNoRevokedOrClosed`; `membersList.exposesNoFactorCountTenantOrGrant` | 3, 8 |
| RF-2.4, RF-10.5, LC-12, LC-13, LC-34 | API contract, BE unit | `rowActions.exactPerActorRole` (Owner, Admin, own row, Owner row, pending row); `admin.cannotReachOwnerAffectingActionOrOwnerInvitation` | 3 |
| RF-2.5, RF-2.6, RF-2.7, LC-28 | BE integration, FE unit | `membersList.searchFiltersAndPagination`; `membersList.emptyScopeKeepsFilters` | 1 |
| RF-2.8, RF-10.4, RF-10.6, LC-27 | Concurrency | `confirmation.staleVersionRefusedAndRefreshed`; `confirmation.actorChangedSinceOpenRefused` | 9 |
| RF-3.1, RF-3.2, RF-3.5 | FE unit | `authorityStatements.membersVariantVerbatimAtEveryWidthAndRole` | 1, 13 |
| RF-3.3, RF-3.4, LC-30 | FE e2e, BE integration | `routes.reviewGrantsAndDecisionHistory`; `decisionHistory.scopedByReadAudit` | 1 |
| RF-4.1, RF-4.5, RF-4.6 | FE unit, API contract | `transferDialog.rendersN23ContentAndKeepDoesNothing`; `transfer.responsesCarryNamesOnly` | 5 |
| RF-4.2, RF-4.3, LC-7, LC-8, LC-29 | BE unit, integration | `successorEligibility.rule`; `successors.listsEligibleOnlyWithStatementNoReasons`; `successors.emptyWhenNoneEligible` | 5 |
| RF-4.4 | API contract | `transfer.ownerOnlyAndOwnRowOnly` | 3, 5 |
| RF-5.1, LC-9, LC-10 | BE integration (plan 003 step-up) | `stepUp.tenantOwnershipPurposeGenericCountedBoundedRecoveryRefused`; `stepUp.voidClosesWithNothingChanged` | 5 |
| RF-5.2, LC-5, LC-6 | BE integration | `transfer.recheckRefusesWithStatedReason` (successor suspended, actor changed, tenant inactive); `transfer.successorFactorReplacedStillEligible` | 5 |
| RF-5.3, RF-5.8, LC-1 | BE integration | `transfer.oneTransactionSuccessorFirstBothVersions`; `transfer.lastOwnerAllowed` | 4, 5 |
| RF-5.4 | BE integration | `transfer.touchesNoGrantSessionIdentityOrOtherTenant` | 8 |
| RF-5.5 | BE integration | `notifications.ownershipReceivedAndTransferred` | 11 |
| RF-5.6, LC-11, LC-12 | FE e2e, BE integration | `transfer.successorDerivesOwnerAtNextRequest`; `transfer.formerOwnerDerivesAdminAtNextRequest` | 5 |
| RF-5.7, RNF-13, LC-24 | BE integration, FE unit | `transfer.resubmitReturnsRecordedOutcome`; `issue.resubmitIssuesOnce`; `dialog.reloadResubmitsSameKey` | 10 |
| RF-6.1, RF-6.2 | FE unit, API contract | `editRoleDialog.offersOnlyPermittedRoles`; `editRole.selfRefused` | 3, 6 |
| RF-6.3, RF-6.4, LC-14, LC-15 | BE integration | `editRole.ownerAffectingRequiresStepUpAndLock`; `editRole.memberAdminConfirmationOnly`; `editRole.promoteToOwnerRequiresEligibility` | 6 |
| RF-6.5 | BE integration | `editRole.recheckRefusesWithStatedReason` | 6 |
| RF-6.6, RF-7.8 | BE integration | `change.versionAuditAndNotification` (role, suspend, reinstate, remove) | 6, 11 |
| RF-6.7 | BE integration, FE e2e | `editRole.adminWithoutFactorAllowedThenGated` | 6 |
| RF-7.1, RF-7.2 | FE unit, API contract | `manageDialog.actionsByState`; `manage.selfRefused` | 3, 6 |
| RF-7.3 | BE integration | `manage.ownerTargetRequiresStepUpLockAndGuard` | 6 |
| RF-7.4, RF-11.5, LC-16 | BE integration | `suspend.deniesNextRequestKeepsGrantsRefusesQueuedCancelsRunningKeepsFacts` (plan 009 handler, execution double) | 6, 8 |
| RF-7.5, LC-17 | BE integration | `reinstate.grantsUnchangedNewVersionNothingResumed` | 6 |
| RF-7.6, LC-18 | BE integration | `remove.cascadesGrantsRetainsEvidenceIdentityUntouchedRowGone` | 6 |
| RF-7.7 | BE integration | `manage.recheckRefusesWithStatedReason` | 6 |
| RF-7.9, RNF-6 | BE integration | `change.sessionsNeverRevokedDeniedScopeAtNextRequest` | 6 |
| RF-7.10, LC-19 | BE integration (plan 008 accept double) | `removed.returnsOnlyByInvitationWithInvitedScope` | 6, 7 |
| RF-8.1, RF-8.3 | FE unit, API contract | `inviteDialog.fieldsStatementsAndChoices`; `issue.fieldPathProblems` | 7 |
| RF-8.2, LC-34 | API contract, BE integration | `issue.ownerRoleNeverOfferedNorAccepted`; `issue.boundedByAdministrationRule` | 7 |
| RF-8.4, RF-11.7, LC-31, LC-32, LC-33 | BE integration, API contract + timing | `issue.refusesExistingMembershipAndPendingWithReasons`; `issue.unknownAddressIssuedIdentically`; `issue.timingComparableKnownUnknownAddress` | 7 |
| RF-8.5, RF-8.6, RF-8.7 | BE integration | `issue.oneTransactionSecretHashSaveMessageAudit`; `issue.createsNoMembershipGrantOrIdentity`; `issue.rowAppearsInBothTables` | 7 |
| RF-9.1, RF-9.2 | API contract, FE unit | `invitations.listShape`; `reviewPanel.scopeIssuerExpiryAndActions` | 7 |
| RF-9.3, LC-35 | BE integration | `resend.newSecretNewExpiryOldLinkDeadScopeUnchanged` | 7 |
| RF-9.4, RF-9.7, LC-36 | Concurrency | `revoke.finalSilentRowGone`; `revoke.vsAcceptOneTransition` | 7 |
| RF-9.5, LC-37 | BE integration | `expiryJob.settlesExpiredSilently` | 7 |
| RF-9.6, LC-38, LC-39 | BE integration | `settled.inviterNotifiedOnAcceptAndDecline`; `settled.notifiedEvenIfInviterRemoved` | 7, 11 |
| RF-9.8 | BE integration | `invitation.survivesInviterLossOfAuthority` | 7 |
| RF-10.1, LC-2, LC-26 | BE integration | `lastOwner.refusedWithReasonAndRoute`; `lastOwner.restrictedOwnerDoesNotCount` | 4 |
| RF-10.2, RF-10.3, RNF-2, RNF-4, LC-3, LC-4 | Concurrency | `ownershipLock.serialisesOwnerAffectingChanges`; `lastOwner.twoOwnersDemoteEachOtherOneRefused`; `lastOwner.neverZeroActiveOwnersUnderConcurrency` | 4 |
| RF-11.1, RF-11.4 | BE integration | `feature.callsNoGrantOrCreateMembershipCommand` | 8 |
| RF-11.2, RF-11.3 | API contract | `noIdentityControlEndpointExists`; `self.cannotEditManageOrInvite` | 8 |
| RF-11.6 | BE integration | `change.touchesNoTenantPolicyOrOtherTenant` | 8 |
| RF-12.1, RF-12.2, RF-12.3, RNF-15, RNF-17 | FE e2e + axe, manual assistive | `a11y.people.{en,es,fr}.{desktop,tablet,mobile}` (page, both tables, every dialog and panel); manual NVDA / VoiceOver checklist | 13 |
| RF-13.1, RF-13.2 | BE integration | `audit.entryPerEvent`; `audit.containsNoPasswordCodeSecretOrContent` | 11 |
| RF-13.3 | API contract | `problems.reasonsForAdministratorsNoIdentitySecurityFacts` | 11 |
| RF-13.4, RNF-14, LC-25 | BE integration | `templates.eightMessagesCarryStatements`; `message.languageRules`; `notification.failureNeverRevertsChange` | 11 |
| RF-13.5 | FE unit | `storage.containsNothingFromFeature` | 11 |
| RF-13.6 | API contract | `responses.carryCommittedState` | 11 |
| RNF-1 | API contract, BE integration | `authority.serverDecidesEveryRowActionAndConfirmation` | 3 |
| RNF-3 | Concurrency | `change.partialStateNeverObservable` | 6 |
| RNF-5 | BE integration | `ownerAffecting.alwaysRequiresConsumedStepUp` | 6 |
| RNF-7 | BE integration | `change.identityCredentialFactorSessionsOtherTenantsUntouched` | 8 |
| RNF-9 | BE integration | `invitation.adminOrMemberOneProjectHashedOnePending` | 7 |
| RNF-10 | API contract | `responses.revealNoIdentityExistenceOrSecurityFacts` | 7 |
| RNF-11 | BE integration | `failClosed.staleUnknownClosedInactiveUnconfiguredRefuses` | 9 |
| RNF-12 | Startup | `tenantMembersPolicy.startupFailsOnAbsentOrUnboundedValue` | 12 |
| RNF-16 | FE e2e | `allStates.existInEnEsFr` | 13 |
| RNF-18 | Concurrency, Security | area A suites above | 4 |
| LC-22 | BE integration | `search.removedAddressNotListedAuditHasIt` | 8 |
| US-13 | all adversarial cases above | Admin-mints-Owner, Owner invitation, last-Owner race, self-change, address enumeration, identity controls, project read through role, unversioned change suites | 1 |

### 5.3 Not tested here
The `access` core (matrix, derivation, version guard, revocation effects — plan 009), invitation acceptance and decline (008), the step-up's internals and the enrollment gate (003), sessions and the denied-scope handler (006), the project grants surface (011), the bootstrap that creates the lock row (012), the chooser (013), the suspended-tenant screen (014), the audit trail surface (F-APR-04). Their stubs return fixed answers; `MembershipCommands`, `AuthorityChecker`, `StepUpPort`, `FactorStatusPort`, `IdentityFactsPort` and `TenantStatusPort` are verified by recording stubs and, in the integration suite, by the owning contexts' test doubles.

### 5.4 Exit criteria
Specification §7 items 1–14 map to the "Proves criterion" column; item 14 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's decisions Q1–Q10 are built, not assumed; the register items it cites are below with the assumption this plan builds. Two cross-plan items are added because this plan's design depends on them.

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| S-50 R13 dialogs and states (D02 `419:2645`) | `EditRoleDialog`, `ManageMembershipDialog`, `InviteMemberDialog`, `ReviewInvitationPanel`, the step-up variant, the empty picker, the empty list and the refused states, all on N23's anatomy. | Adjust the dialog layouts and copy only; the use cases are unchanged. |
| S-49 Admin view of R13 (decided) | Row actions computed on the server per actor (D9); the Admin sees none on Owner rows and none on their own row. | None here. |
| S-04 "Invite member" destination | `InviteMemberDialog` as specified; the invitation message template. | Replace the dialog layout. |
| S-14 templates | The D04 copy (`437:4431`) for the eight templates carrying the statements of RF-13.4 in EN / ES / FR. | None. |
| S-02 responsive coverage | Reflow of R13, its tables as records, N23 and the dialogs under the desktop rules. | Replace the breakpoint layouts. |
| A-52 invitation record (decided) | Plan 008's aggregate extended with issue, resend, revoke, expire; one pending per address by partial unique index; resend replaces the secret (D6). | If several pending per address were ever allowed, drop the index and add the first-acceptance-invalidates rule in plan 008. |
| A-54 … A-57 membership decisions (decided) | Plan 009's commands, states, version and administration rule are consumed as built. | None here. |
| A-30 notification set | The membership family of eight templates. | Add or remove templates; the outbox calls are the only place. |
| A-27 identity statuses | `ActiveOwnerRule` and `SuccessorEligibilityRule` require an `ACTIVE` identity. | Change the two predicates only. |
| A-36 identity-scoped read | Not used by this feature: every read is tenant-scoped under C10; the issuer's address for the settled notification is read by identity id. | None here. |
| A-35 persistence tooling | Adapters, migrations and the backfill of `tenant_ownership` against the placeholder API and tool. | Rename base types and migration format. |
| **Cross-plan · 008 must enqueue `InvitationSettled`** | Plan 008's `AcceptInvitationUseCase` and `DeclineInvitationUseCase` enqueue one outbox event in their transaction (D8). | If plan 008 refuses an outbox dependency, `InvitationSettledHandler` becomes a short-cadence job reading settled rows since its watermark; the notification is then delayed, not lost. |
| **Cross-plan · project-grant shape in plans 008 and 009** | Both plans use plan 009's one-row-per-membership-and-project record (plan 008 aligned on 28 October 2026). | None here: this feature never writes a grant. |
| **Cross-plan · 012 writes the lock row** | The workspace bootstrap inserts `tenant_ownership` in the same transaction as the founding Owner; this plan's migration backfills existing tenants. | If 012 cannot, the first Owner-affecting command of a tenant inserts the row with `INSERT … ON CONFLICT DO NOTHING` before locking it. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.7) | `PeoplePage`, C10 filter **[009]**, `TenantMembersQuery`, RLS, `TenantStatusPort` | D9, D14 | RF-1 rows |
| RF-2 (2.1–2.8) | `TenantMembersQuery`, `MemberRow`, `InvitationRow`, `MembersFilter`, `AdministrationRule` **[009]**, `MembersTable` | D9, D10 | RF-2 rows |
| RF-3 (3.1–3.5) | `AuthorityStatements` **[009]** `members` variant, routes, audit lineage | D14, D18 | RF-3 rows |
| RF-4 (4.1–4.6) | `SuccessorsQuery`, `SuccessorEligibilityRule`, `TransferOwnershipDialog` | D3, D4 | RF-4 rows |
| RF-5 (5.1–5.8) | `TransferOwnershipUseCase`, `StepUpPort`, `TenantOwnership`, `LastOwnerGuard`, `ChangeTenantRole` **[009]**, templates | D1, D2, D3, D12, D15 | RF-5 rows |
| RF-6 (6.1–6.7) | `EditTenantRoleUseCase`, `OwnerAffectingRule`, `EditRoleDialog` | D1, D3, D4, D5, D10 | RF-6 rows |
| RF-7 (7.1–7.10) | `ManageMembershipUseCase`, plan 009's commands and `AuthorityChanged`, `ManageMembershipDialog` | D1, D5, D11 | RF-7 rows |
| RF-8 (8.1–8.7) | `IssueInvitationUseCase`, `InvitationIssueRule`, `Invitation.issue` **[008]**, `InviteMemberDialog`, template `invitation` | D6, D7, D12, D15 | RF-8 rows |
| RF-9 (9.1–9.8) | `PendingInvitationsTable`, `ReviewInvitationQuery`, `ResendInvitationUseCase`, `RevokeInvitationUseCase`, `ExpireInvitationsJob`, `InvitationSettledHandler` | D6, D8, D13 | RF-9 rows |
| RF-10 (10.1–10.6) | `TenantOwnership`, `LastOwnerGuard`, `ActiveOwnerRule`, `PermissionVersionGuard` **[009]**, `AuthorityChecker` **[009]** | D1, D10 | RF-10 rows |
| RF-11 (11.1–11.7) | absence of grant, membership-creation, identity, session and cross-tenant paths; `InvitationIssueRule` | D7, D11 | RF-11 rows |
| RF-12 (12.1–12.3) | `PeoplePage` reflow, `shared/ui` | D14 | RF-12 row |
| RF-13 (13.1–13.6) | audit adapter, problem codes, templates, dialog state | D12, D16, D18 | RF-13 rows |
| RNF-1 | server-computed actions, `AuthorityChecker` | D9, D10 | `authority.serverDecides…` |
| RNF-2, RNF-4 | lock row and guard | D1 | concurrency rows |
| RNF-3 | one transaction per change | D2, D15 | `change.partialStateNeverObservable` |
| RNF-5 | step-up consumption | D3, D5 | `ownerAffecting.alwaysRequires…` |
| RNF-6, RNF-7 | next derivation; no session port | D11 | RF-7.9, RNF-7 rows |
| RNF-8 | RLS, C10 scope | — | security row |
| RNF-9, RNF-10 | invitation type, index, conflict reasons | D6, D7 | RF-8 rows |
| RNF-11 | rechecks, conditional updates, config | D10, D17 | `failClosed` |
| RNF-12 | `TenantMembersPolicyProperties` | D17 | startup |
| RNF-13 | idempotency outcomes | D15 | RF-5.7 row |
| RNF-14 | outbox templates | D12 | RF-13.4 row |
| RNF-15–17 | one page, `shared/ui`, i18n | D14 | a11y, language rows |
| RNF-18 | concurrency and security suites | D1, D10 | area A rows |
| LC-1 … LC-39 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39) |

Self-check: every RF criterion (88), every RNF (18) and every limit case (39) of `specTenantMembersOwnership.md` v0.1 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
