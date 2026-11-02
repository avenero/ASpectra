# Plan 008 — Invitation acceptance

**Feature** Opening an invitation link, reviewing the exact scope it grants, accepting it as the invited identity or declining it · **Plan version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — aligned with specification v0.1 (decisions Q1–Q7, derived rules D-a…D-e) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/008-InvitationAcceptance/specInvitationAcceptance.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-10 as the subject; F-IDN-01, F-IDN-02, F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-11…F-IDN-18 and F-SET-04 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG), FE board `46:7847` (feature-modular SPA DAG) and decision register `122:2` §5 (domain and data ownership: the `access` context owns memberships, invitations, project grants and permission versions) · **Builds on** plan 001 (Login), plan 002 (Signup and verification), plan 003 (Enrollment), plan 006 (Sessions) and plan 009 (Roles and grants: the membership and grant model, `TenantMembership`, `ProjectGrant` and `GrantSet`); elements they own are tagged **[001]** / **[002]** / **[003]** / **[006]** / **[009]** and are extended, never duplicated. The invitation record is tagged **[008 → 010]**: plan 010 adds its issuing side. Specifications 007 (logout), 012 (workspace creation) and 016 (guarded switch) are reached only by route.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's seven decisions (Q1–Q7) and five derived rules (D-a…D-e) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-52, A-28, A-27; S-45, S-46, S-24, S-03, S-04, S-14, S-17; plus A-35 for the persistence tooling and A-36 for identity-scoped reads), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | Nothing in this feature reads, stores or replays a Run, approval or mapping intent; acceptance writes membership and grant rows only. The word "acceptance" here is the invitee's acceptance of a membership, never C11 semantic acceptance. | RF-6.4, RNF-10 |
| **P3** The browser expresses intent, never authority | The client submits a link secret, an acknowledgement flag and an idempotency key, and renders the server's answer. Whether the signed-in identity is the invited one, whether the invitation is usable, what scope is granted and where the person lands are all decided on the server from the invitation record and the session facts; the client never sends a role, a grant or a tenant identifier for this feature. | RF-1.3–1.5, RF-3.4, RF-5.8, RF-6.3, RNF-3 |
| **P6** Fail closed | An unknown, expired, revoked, accepted or declined invitation, a suspended or deleting tenant, an existing active or suspended membership, an unverified or restricted identity, a mismatched address, a missing acknowledgement or an absent configuration value refuses the effect; no default is assumed. | RF-5, RNF-5, RNF-6 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security session support (plan 006), the existing transactional outbox (unused by this feature: no message is sent). FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. Nothing is added: no cache store, no queue, no new identity provider. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the invitation record and the acceptance transaction live in the hexagonal bounded context `access`, which owns memberships, invitations, project grants and permission versions (register §5); `identity`, `tenant` and `audit` are reached only through published ports; no foreign repository is touched. FE: the same feature module as Login and Signup; TanStack Query owns server facts; Zustand is not used (no unsaved fork exists on these screens). | §2 |
| **P12** Two independent tenancy gates | `tenant_invitation`, `tenant_membership` and `project_grant` are tenant-scoped rows with `tenant_id` in their composite keys under `FORCE RLS`; the acceptance transaction establishes its tenant context with `SET LOCAL` from the invitation record before any write, so `WITH CHECK` passes without a bypass role. Runtime authorisation is the second gate: the use case rechecks identity, tenant state and membership itself. The resolve path reads the invitation through the same context. | RF-6.1, RNF-9, RNF-10 |
| **P13** Configured values are never defaulted | Invitation expiry and the throttle steps and windows form one validated configuration record; startup fails on any absent or unbounded value. | RF-5.7, RF-9.4, RNF-6 |
| **P14** Release bar | WCAG 2.2 AA on the four states with automated and manual assistive evidence; EN / ES / FR for every statement. | RF-10, RNF-12–14 |
| **F-X-01** C10 context | Resolve and decline derive an *identity-scoped* context from the session facts (plan 006 `SessionAssertionPort`); accept derives an *invitation-scoped* C10 — actor = the signed-in identity, tenant = the invitation's tenant, authority = "accept this invitation" — inside the transaction and rechecks `permissionVersion` facts transactionally. Tenant entry after acceptance re-derives ordinary C10 and runs the enrollment gate exactly as plan 003 D7. | RF-5.8, RF-6.1, RF-6.5, RF-6.7 |
| **F-X-02** Request integrity | Accept and decline are consequential writes with no generic auto-retry; accept carries an idempotency key whose outcome is recorded on the invitation row and returned on re-submission; a lost answer leads to that lookup, never a second write. The resolve call is a safe read. | RF-6.1, RF-6.8, LC-9, LC-29 |
| **F-X-03** Secrets and safe problems | The link secret exists only in the message, in the route the person opened and in the request that presents it; at rest only its hash; problem details carry safe codes, field paths and a correlation id; browser storage holds nothing from this feature. | RF-1.1, RF-11.4, RF-11.5, RNF-8 |
| **F-X-05** Audit log | Every event of RF-11.1 is appended through the `audit` context port, redacted, in the same transaction as the effect. | RF-11 |
| **F-X-06** Configuration validation and email port | `InvitationPolicyProperties` is part of startup validation; no message is sent by this feature (decision Q7), so the email port is not touched. | RF-5.7, RF-9.4, RF-11.3 |
| **F-X-07** Responsive surfaces | M03 is the designed counterpart of the review state and is built as drawn; the pre-match state follows M19 (`432:4231`); the wrong-identity and unavailable states follow the D01 responsive policy with `SecurityStatements` shared across widths. | RF-10 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.access` (new; shared later with specifications 009, 010, 011, 016)

This plan is the first to build the `access` context beyond the C10 derivation that plans 001, 003 and 006 placed in `access.domain`. It has the same four layers as `identity` (`api / application / domain / infrastructure`) and the same allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories. Contexts touched — `identity`, `tenant`, `audit`, `sharedsupport` — are reached only through their published ports. The `identity` context adds nothing but one port implementation.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `Invitation` **[008 → 010]** | Aggregate: id, `tenantId`, `secretHash`, `invitedEmail` (normalised `EmailAddress` **[002]**), `tenantRole` ∈ {`ADMIN`, `MEMBER`}, optional `ProjectGrantSet`, `issuedBy`, `issuedAt`, `expiresAt`, `state` ∈ {`PENDING`, `ACCEPTED`, `DECLINED`, `REVOKED`, `EXPIRED`}, `acceptedBy`, `acceptedAt`, `declinedAt`, `acceptIdempotencyKey`. Methods `isPending(now)` (state `PENDING` and `expiresAt > now`), `accept(identityId, now, key)`, `decline(now)`; `revoke` and the constructor used at issue belong to specification 010 and are stubbed here for tests only. Owner is never a permitted value of `tenantRole`; the type does not admit it. | RF-5.1, RF-5.2, RF-6.1, RF-7.2, RNF-4 |
| | `ProjectGrantSet` | Value object: `projectId`, `roles` ⊆ {`AUTHOR`, `REVIEWER`, `VIEWER`}, `canRun` boolean; at most one per invitation (decision Q4). `toGrantSet()` yields the `GrantSet` **[009]** (the invited roles and `canRun`) that `TenantMembership.setGrants(projectId, GrantSet)` **[009]** writes as the single `ProjectGrant` row of the invited project — exactly as issued, never more, never fewer. | RF-2.2, RF-5.1, RF-6.1, RF-6.3, LC-20, LC-21 |
| | `TenantMembership` **[009]** | The aggregate plan 009 owns (id, tenantId, identityId, `role`, `status`, `permissionVersion`, timestamps, `ProjectGrant` children; primary key `(tenant_id, id)`, unique `(tenant_id, identity_id)`; `setGrants`, `revoke` and the rest of its lifecycle). This plan adds two operations to it: `createFromInvitation(invitation, now)` (status `ACTIVE`, version 1, the invited role, then `setGrants` with the invited `GrantSet` when a project was invited) and `reactivateFromInvitation(invitation, now)` (allowed only from `REVOKED`; the **same row**: the invited role, status `ACTIVE`, `permissionVersion + 1`, then `setGrants` for the invited project only — restoring nothing else). | RF-6.1, RF-6.2, RF-6.6, LC-17 |
| | `ProjectGrant` **[009]** | The child record plan 009 owns: (tenantId, membershipId, projectId, `GrantSet`, `revokedAt`), one row per membership and project, keyed `(tenant_id, membership_id, project_id)`. This plan creates or revives it only through `setGrants` on the invited project; a row of a revoked membership keeps `revokedAt` until a re-invitation to the same project revives it with the invited set. | RF-6.1, RF-6.2, RF-6.3 |
| | `AcceptanceRule` | Pure function `(invitation?, now, tenantStatus, identityFacts?, existingMembership?) → ACCEPTABLE | UNUSABLE(cause)` with causes `UNKNOWN`, `EXPIRED`, `REVOKED`, `USED`, `DECLINED`, `TENANT_SUSPENDED`, `TENANT_DELETING`, `IDENTITY_MISMATCH`, `IDENTITY_INELIGIBLE`, `MEMBERSHIP_ACTIVE`, `MEMBERSHIP_SUSPENDED`; `MEMBERSHIP_REVOKED` is acceptable with `replace = true`. Evaluated at resolve (to choose the state shown) and again inside the accept transaction under row locks. The cause is for audit only; `toProblem()` maps every cause to the single code `access.invitation.unavailable`. Order of evaluation is fixed so that a suspended tenant is detected before anything is consumed (D-d). | RF-5.3–5.6, RF-5.8, RF-9.2, RNF-2, RNF-5 |
| | `ResolutionOutcome` | Result of a resolve: `PRE_MATCH(tenantName, maskedEmail)`, `REVIEW(scope)`, `WRONG_IDENTITY(tenantName, maskedEmail)`, `UNAVAILABLE(cause)`. `scope` = invited email, tenant name, tenant role, project name (or none), granted roles, `canRun`, display name of the signed-in identity (for "Accept invitation as …"). Never the inviter, never another member, never the secret. | RF-1.2–1.5, RF-2.1, RF-2.5, RF-3.1 |
| | `EmailMask` | Pure function: keeps the first character of the local part and the full domain (`m•••@example.test`); a one-character local part becomes `•`. Computed server-side only. | RF-1.2, RF-3.1, RF-9.3 |
| | `AbuseThrottlePolicy` **[001]** | Reused unchanged; the dimension key `INVITATION_NETWORK` is added with its own configured steps and window. No per-address key (D10). The sign-in keys are never written by this feature. | RF-9.1, RNF-7 |
| | `InvitationPolicy` | Immutable record: invitation expiry (consumed at issue by specification 010; validated here because acceptance depends on it), throttle steps and window for `INVITATION_NETWORK`. | RF-5.7, RF-9.4, RNF-6 |
| `access.application` | `ResolveInvitationQuery` | Hashes the presented secret (`TokenHasher` **[002]**); evaluates the network throttle; loads the invitation by hash under a platform read of the invitation table (the tenant is not yet known — see D5); loads `TenantStatusPort.statusOf(tenantId)` and the tenant name; if a session is present, loads `IdentityFactsPort.facts(identityId)` and the identity's own membership in that tenant (an identity-scoped read, A-36); applies `AcceptanceRule`; returns the `ResolutionOutcome`: no session → `PRE_MATCH`; session with mismatched address → `WRONG_IDENTITY`; session matched and acceptable → `REVIEW`; otherwise `UNAVAILABLE`. Changes nothing; audits `INVITATION_RESOLVED{outcome, invitationId?}`. | RF-1.1–1.6, RF-1.9, RF-2.6, RF-3.1, RF-3.4, RF-5.4–5.6, RF-9.2, RF-9.3, LC-1–LC-8, LC-11–LC-16 |
| | `AcceptInvitationUseCase` | Requires a session (plan 006 `SessionAssertionPort`). Steps, all in one transaction: (1) look up the recorded outcome for the idempotency key on the invitation row — same key → return the recorded `ACCEPTED` answer, different key on a non-pending row → `UNAVAILABLE` (F-X-02); (2) lock the invitation row (`SELECT … FOR UPDATE`) and the membership row of (tenant, identity) if any; (3) establish the invitation-scoped C10 with `SET LOCAL` tenant = `invitation.tenantId` (D5); (4) load tenant status and identity facts, apply `AcceptanceRule` — `UNUSABLE` → `INVITATION_ACCEPT_REFUSED{cause}` and 410, nothing consumed; (5) conditional `UPDATE tenant_invitation SET state = 'ACCEPTED', accepted_by, accepted_at, accept_idempotency_key WHERE id = ? AND state = 'PENDING' AND expires_at > now()` — zero rows → `UNAVAILABLE`; (6) `TenantMembership.createFromInvitation` or `reactivateFromInvitation` **[009]**; (7) `setGrants(projectId, ProjectGrantSet.toGrantSet())` **[009]** for the invited project, when one was invited — one `project_grant` row, created or revived; (8) `PermissionFactsPort.membershipChanged(tenantId, identityId, permissionVersion)` so the C10 derivation **[006]** sees the new facts on the next request; (9) audit `INVITATION_ACCEPTED{membershipReplaced}` and enqueue the outbox event `InvitationSettled{invitationId, tenantId, outcome: ACCEPTED, issuedBy}` that plan 010's `InvitationSettledHandler` turns into the inviter notification (specification 010 Q10). Returns `{tenantId, projectId?}` for routing only. Creates no session, sends no message itself, touches no identity, factor, other membership or run row. | RF-5.3, RF-5.8, RF-6.1–6.4, RF-6.6, RF-6.8, RF-6.9, RNF-1, RNF-4, RNF-10, LC-7–LC-10, LC-17, LC-20–LC-22 |
| | `DeclineInvitationUseCase` | Requires a session; locks the invitation; `AcceptanceRule` must yield `ACCEPTABLE` for the signed-in identity (only the invited identity may decline, RF-7.1 — a suspended-tenant or existing-membership cause also refuses, since the review state is not shown then); conditional `UPDATE … SET state = 'DECLINED', declined_at WHERE state = 'PENDING' AND expires_at > now()`; audits `INVITATION_DECLINED` and enqueues `InvitationSettled{…, outcome: DECLINED}` for plan 010. Creates nothing, sends nothing itself. | RF-7.1–7.3, LC-10, LC-12 |
| | `EnrollmentGate` **[003]** | Untouched. It runs inside the ordinary C10 derivation when the person enters the tenant after acceptance (plan 003 D7); the membership exists, its authority is withheld. | RF-4.7, RF-6.7, LC-18, LC-19 |
| | Ports | New, published by `access`: `InvitationRepository`, `MembershipRepository` **[009]**, `ProjectGrantRepository` **[009]**, `PermissionFactsPort` (written here, read by the C10 derivation **[006]**). New, consumed from other contexts: `IdentityFactsPort` (published by `identity`: `facts(identityId) → {normalisedEmail, status, emailVerified, displayName}`), `TenantStatusPort` (published by `tenant`: `status(tenantId) → ACTIVE | SUSPENDED | DELETING`, `name(tenantId)`), `SessionAssertionPort` **[006]**. Reused: `ThrottleStore`, `Clock`, `AuditPort` **[001]**; `TokenHasher` **[002]**. `TokenGenerator` **[002]** is used at issue by specification 010, not here. | all |
| `access.infrastructure` | Persistence | New tables, all tenant-scoped under `FORCE RLS` with `USING` and `WITH CHECK` on `tenant_id`: `tenant_invitation` (composite key `(tenant_id, id)`, `secret_hash` unique across tenants, invited_email_normalised, tenant_role, project_id nullable, grant_roles, can_run, issued_by, issued_at, expires_at, state, accepted_by, accepted_at, declined_at, accept_idempotency_key; partial index on pending rows per tenant), `tenant_membership` and `project_grant` exactly as plan 009 defines them — primary key `(tenant_id, id)` with unique `(tenant_id, identity_id)`; `project_grant` keyed `(tenant_id, membership_id, project_id)` with the four grant flags and `revoked_at` — to which this plan adds no column; when plan 009's migration is not yet applied, this plan creates both tables in plan 009's shape. One `SECURITY DEFINER` function, audited as F-X-01 requires, resolves `secret_hash → tenant_id` for the resolve path (D5); every other access runs under the tenant context. `signin_throttle` **[001]** gains the `INVITATION_NETWORK` dimension value. Persistence access API and migration tool per A-35 (placeholders). | RF-1.1, RF-5, RF-6, RNF-8, RNF-9 |
| | `InvitationPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding of every number into `InvitationPolicy`; constraints: present, positive, bounded above by a configured ceiling. Any violation fails startup. | RF-5.7, RF-9.4, RNF-6 |
| | Identity facts adapter (in `identity.infrastructure`) | Implements `IdentityFactsPort` over `IdentityRepository` **[001]**: returns the normalised email, status (A-27 model), verification flag and display name of one identity by id; never a credential, factor or session. | RF-5.3, LC-28 |
| | Tenant status adapter (in `tenant.infrastructure`) | Implements `TenantStatusPort`; a read of the tenant row's status and name. | RF-5.3, RF-5.4, LC-13, LC-14 |
| | Audit adapter | Events: `INVITATION_RESOLVED{outcome, invitationId?}`, `INVITATION_ACCEPTED{invitationId, tenantId, role, grantKinds, membershipReplaced}`, `INVITATION_DECLINED{invitationId}`, `INVITATION_ACCEPT_REFUSED{cause}`, `RESTRICTION_APPLIED` / `RESTRICTION_LIFTED` **[001]** for the invitation key. Payload: identity id when a session exists, correlation id, network key, timestamp. Never the secret, an unmatched address or a message body. | RF-11.1, RF-11.2, RF-11.4 |
| `access.api` | `InvitationController` — `/api/invitations` | `POST resolve` {secret} (session optional) → 200 `{state: PRE_MATCH, tenantName, maskedEmail}` · 200 `{state: REVIEW, scope}` · 200 `{state: WRONG_IDENTITY, tenantName, maskedEmail}` · 410 problem `access.invitation.unavailable` (one code for every cause) · 429 problem `access.throttled` with no cause. `POST accept` {secret, acknowledged, idempotencyKey} (session required, else 401) → 200 `{state: ACCEPTED, tenantId, projectId?}` · 400 field problem `access.invitation.acknowledgement-required` · 410 `access.invitation.unavailable`. `POST decline` {secret} (session required) → 200 `{state: DECLINED}` · 410. All problems are RFC 9457 problem details with a safe code, field paths and a correlation id. The server never redirects and never echoes the secret. | RF-1.2–1.5, RF-2.4, RF-5.6, RF-6.1, RF-7.2, RF-9.1, RF-11.4 |

**Cross-context touch points.** `identity.IdentityFactsPort` — one read per resolve with a session and per accept. `tenant.TenantStatusPort` — status and name. `audit.AuditPort` — append-only. `sharedsupport` — `crypto` (hashing), `tx`, `contextIds` (correlation id). The session service **[006]** through `SessionAssertionPort` only. Nothing reads or writes credential, factor, session, approval or run tables; no message is enqueued.

**Data model note (P12).** All three tables are tenant-scoped. The only read that precedes a tenant context is the lookup `secret_hash → tenant_id` on the resolve and accept paths, performed by an audited `SECURITY DEFINER` function that returns the tenant id and nothing else (D5); from that point every statement runs under `SET LOCAL` tenant context.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login and Signup)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/invitation` (the secret in the query of the SPA route, read on mount into the holder and removed from the visible URL by a history replace; the four states are states of this route). The invited signup variant is `/sign-up` **[002]** with the holder present. | RF-1, RF-4.1 |
| | `useResolveInvitation()`, `useAcceptInvitation()`, `useDeclineInvitation()` | Typed intent hooks; `useAcceptInvitation` generates its idempotency key once per page instance and reuses it on re-submission; all attach the correlation id; return server state verbatim. | RF-1–RF-7 |
| | `contracts/invitation` (in `src/contracts`) | Request / response / problem types and runtime guards for the four resolve states, the accept and decline responses and the three problem codes. | all |
| Views | `InvitationPage` | One component, four states: `pre-match` (heading, subtitle, tenant name, masked address, "Existing local account" → `/sign-in` keeping the holder, "Create local account" → `/sign-up` keeping the holder, the statements, "← Back to sign in", Privacy / Terms, language switch); `review` (N02 at desktop widths, M03 at mobile widths: scope blocks, "membership only" variant when no project, identity-matches block, acknowledgement checkbox never pre-ticked, "Accept invitation as <display name>", "Use another identity", "Decline invitation", MFA handoff block, "Next destination", the switching / unusable statement); `wrong-identity` (N29 panel: heading, statement, tenant name, masked address, "Switch identity safely", "Leave invitation"); `unavailable` (D02 "This invitation can't be accepted", statement, "Return to memberships →"). Resolves on mount; the language switch changes display only and keeps the resolved outcome. | RF-1.2–1.8, RF-2.1–2.5, RF-3.1, RF-5.6, RF-8.1–8.3, RF-10.1–10.2 |
| | `SecurityStatements` **[001]** | Extended with the N02 / M03 / N29 / D02 statements (no automatic joining by domain, membership ≠ project grants, switching identity requires signing out, MFA handoff, acceptance applies only to the invited identity, no tenant content shown) so desktop and reflow cannot drift. | RF-2.3, RF-8.1, RF-8.2, RF-10.1 |
| | `SignupPage` **[002]** | Gains the invited variant (D11): when the holder is present, the email field shows the masked invited address as its hint, the statement of RF-4.1 is shown, and the form is submitted together with the link secret; `SignUpUseCase` **[002]** compares the typed address with the invitation's invited address through `InvitationAddressCheckPort` (published by `access`) and refuses a different address with the field problem `auth.signup.invited-address-mismatch`, which names no address. The field is editable but must equal the invited address, because the server never discloses the full address before the identity matches (decision Q2). Everything else unchanged. | RF-4.1, RF-4.6, LC-24, LC-25 |
| | `VerifyEmailPage` **[002]** | Confirmed state offers "Return to your invitation" when the holder is present (route to `/sign-in` with the holder, then the detour). | RF-4.4 |
| | `SignInPage` **[001]** | Shows "Return to your invitation" above the form when the holder is present; on `COMPLETED` the `InvitationReturn` detour (below) runs before `LandingRouter`. | RF-4.4, RF-3.2 |
| | `GenericFailureMessage` **[001]** | Not used: this feature has no generic failure message; the unavailable state is a full state. | — |
| State | TanStack Query | Query for resolve (no retry on 410 or 429; one retry on transport failure as a safe read); mutations for accept and decline (no retry, F-X-02). | RF-1, RF-6, RF-7 |
| | `InvitationReturnHolder` | Module-private in-memory value holding the link secret and the masked address for the current tab; set on `/invitation` mount; read by the pre-match routes, `SignupPage`, `VerifyEmailPage` confirmed, `SignInPage` and the detour; dropped on acceptance, decline, the unavailable state, and "Leave invitation"; **kept** across "Use another identity" / "Switch identity safely" so that the next sign-in can return (RF-3.2). Never written to storage; a new tab has no holder (LC-26). | RF-3.2, RF-4.4, RF-11.5, RNF-8, LC-24, LC-26 |
| Routing | `InvitationReturn` detour | After a completed sign-in **[001]** — never after a re-authentication, which returns to the interrupted screen (006 RF-8.3) — if the holder is present, navigate to `/invitation` (which re-resolves under RF-1) instead of calling `LandingRouter`; the session is established either way, so the landing rule is merely deferred. | RF-4.4, RF-4.5 |
| | Handoffs | `ACCEPTED` → the tenant route `LandingRouter` **[001]** uses for `ENTER_TENANT`, with `projectId` preselected when present; that entry re-derives C10 and, when `auth.enrollment.required` comes back, `EnrollmentGateHandler` **[003]** routes to the wizard with `returnTo` = the tenant route (RF-6.7). `DECLINED`, "Leave invitation" and "Return to memberships →" (signed in) → holder dropped → the chooser route `/workspaces` of plan 013, which always renders on explicit navigation; "Return to memberships →" signed out → `/sign-in`. "Use another identity" / "Switch identity safely" → the sign-out mutation of specification 007 → `/sign-in` with the holder kept. "Existing local account" → `/sign-in` with the holder. "Create local account" → `/sign-up` with the holder. | RF-1.2, RF-3.2, RF-3.3, RF-4.1, RF-5.6, RF-6.5, RF-6.7, RF-7.2 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/errors` (field paths to fields; `EnrollmentGateHandler` **[003]**; `DeniedScopeHandler` **[006]**), `shared/i18n` (bundle keys for every statement and state in EN / ES / FR), `shared/ui` (accessible `Field`, `Checkbox`, `Button`, `LiveRegion`, 44 px targets), `shared/security` (`AccessCredentialHolder` **[006]** for the session). | RF-1.7, RF-1.8, RF-2.4, RF-10.3, RNF-12, RNF-13 |

**Note on the invited signup variant.** Specification 008 Q3 originally said "pre-filled and read-only"; decision Q2 forbids disclosing the full address before the identity matches, so the client cannot pre-fill it. D11 keeps RF-4.1's substance — the account cannot be created with another address through this route — by enforcing equality server-side and showing the masked address as the hint. Q3 and RF-4.1 state this rule since 2 November 2026.

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `ResolveInvitationQuery`: `TokenHasher`, lookup by `secret_hash` only; secret never persisted | `InvitationReturnHolder` in memory; URL secret replaced in history |
| RF-1.2 | `PRE_MATCH{tenantName, maskedEmail}` when no session; `EmailMask` | `InvitationPage` pre-match state with both routes and statements |
| RF-1.3 | `REVIEW(scope)` when session identity matches and `AcceptanceRule` is `ACCEPTABLE` | review state |
| RF-1.4 | `WRONG_IDENTITY` when the session's normalised email ≠ invited email | wrong-identity state |
| RF-1.5 | `UNAVAILABLE(cause)` → 410 one code | unavailable state |
| RF-1.6 | Resolve is read-only; audit only | Mount triggers resolve only |
| RF-1.7 | — | No Help rendered; Privacy / Terms routes per S-14 |
| RF-1.8 | — | Language switch re-renders; resolved outcome and holder unchanged |
| RF-1.9 | `EmailAddress` **[002]** normalisation of the invited address at issue and of the identity's email in `IdentityFactsPort` | — |
| RF-2.1 | `REVIEW.scope` carries every listed element; "membership only" when no `ProjectGrantSet` | review state renders N02 / M03 content |
| RF-2.2 | Scope comes from the record; accept body has no scope fields | No editable scope control |
| RF-2.3 | — | `SecurityStatements` (separate grants, Admin reads no project, `canRun` independent) |
| RF-2.4 | 400 `access.invitation.acknowledgement-required` (server-side too) | Checkbox validation, associated message, no submit |
| RF-2.5 | `ResolutionOutcome.scope` excludes members, other projects, inviter, policy | Page renders only the scope |
| RF-2.6 | Resolve writes nothing; review state is stateless server-side | Reload re-resolves; nothing consumed |
| RF-3.1 | `WRONG_IDENTITY{tenantName, maskedEmail}` | N29 panel with both actions |
| RF-3.2 | Sign-out use case of specification 007 | Sign-out mutation → `/sign-in`; holder kept |
| RF-3.3 | — | "Leave invitation" → holder dropped → the chooser route `/workspaces` (plan 013) |
| RF-3.4 | Identity comparison in `AcceptanceRule` at resolve and inside the accept transaction | Client never sends an identity |
| RF-3.5 | No linking use case exists; `IdentityFactsPort` is read-only | No linking control rendered |
| RF-4.1 | `InvitationAddressCheckPort` consumed by `SignUpUseCase` **[002]**; field problem on mismatch | `SignupPage` invited variant (hint, statement) |
| RF-4.2 | Signup, verification and sign-in use cases **[001/002]** never touch `tenant_invitation` | — |
| RF-4.3 | Invitation state unchanged by any identity use case; `AcceptanceRule` matches on the normalised address | — |
| RF-4.4 | — | `VerifyEmailPage` confirmed and `SignInPage` "Return to your invitation"; `InvitationReturn` detour |
| RF-4.5 | Resolve is idempotent | Re-opening the link mounts `/invitation` again |
| RF-4.6 | `WRONG_IDENTITY` | wrong-identity state; no linking |
| RF-4.7 | `AcceptanceRule` does not read factor status; gate at entry **[003]** | Accept never routes to the wizard; tenant entry does |
| RF-5.1 | `Invitation`, `ProjectGrantSet` types; `tenant_role` check constraint ∈ {ADMIN, MEMBER} | — |
| RF-5.2 | `state` enum; only `PENDING` passes `isPending(now)` | — |
| RF-5.3 | `AcceptanceRule` inputs: invitation, tenant status, identity facts (status `ACTIVE`, verified, email equal), membership status; session required by the controller | — |
| RF-5.4 | `AcceptanceRule` order: `TENANT_SUSPENDED` / `TENANT_DELETING` before the conditional update; suspended leaves `PENDING` | unavailable state |
| RF-5.5 | `MEMBERSHIP_ACTIVE` / `MEMBERSHIP_SUSPENDED` → unusable; membership row untouched | unavailable state |
| RF-5.6 | 410 `access.invitation.unavailable`; response carries no tenant or cause | unavailable state with "Return to memberships →" |
| RF-5.7 | `InvitationPolicyProperties` validation at startup | — |
| RF-5.8 | `AcceptInvitationUseCase` step 4 under row locks | Stale review → unavailable state on accept |
| RF-6.1 | Steps 5–9 in one transaction; `createFromInvitation`; `setGrants` **[009]**; audit; `InvitationSettled` event | `ACCEPTED` handoff |
| RF-6.2 | `reactivateFromInvitation` on the same row; `setGrants` creates or revives the invited project's grant row; other revoked rows untouched | — |
| RF-6.3 | Scope derived from the record; request body has no scope fields; `tenant_role` constraint | — |
| RF-6.4 | Use case touches only the three `access` tables and audit; no session issued | No pending intent stored |
| RF-6.5 | Response `{tenantId, projectId?}` for routing | Tenant route with project preselected (the `ENTER_TENANT` entry) |
| RF-6.6 | `PermissionFactsPort.membershipChanged` in the transaction; C10 derivation **[006]** reads current facts | Scoped cache empty before tenant entry |
| RF-6.7 | `EnrollmentGate` **[003]** in the C10 derivation of the tenant entry | `EnrollmentGateHandler` **[003]** → wizard with `returnTo` |
| RF-6.8 | Row locks + conditional update: one transition; losers → 410 | Loser renders the unavailable state |
| RF-6.9 | State `ACCEPTED` and the new membership row are what specification 010's views read | — |
| RF-7.1 | `DeclineInvitationUseCase` requires session + `ACCEPTABLE` | "Decline invitation" only in the review state |
| RF-7.2 | Conditional update to `DECLINED`; audit; `InvitationSettled{DECLINED}` enqueued; nothing else | Holder dropped → `/workspaces` (plan 013) |
| RF-7.3 | `DECLINED` is final; resolve → `UNAVAILABLE(DECLINED)` | unavailable state |
| RF-8.1 | — | MFA handoff block from `SecurityStatements` |
| RF-8.2 | `scope.projectName` for "Next destination" | "Acceptance applies only…" and "Next destination" statements |
| RF-8.3 | — | Pre-match and invited signup statements |
| RF-9.1 | `AbuseThrottlePolicy` key `INVITATION_NETWORK`; `ThrottleStore`; 429 without cause; sign-in keys untouched | 429 renders the unavailable state unchanged (one state, D-a) |
| RF-9.2 | One problem code; cause in audit only; constant-work resolve path (D3) | One state |
| RF-9.3 | Responses carry only the facts the holder already has | — |
| RF-9.4 | `InvitationPolicyProperties` | — |
| RF-10.1 | — | `InvitationPage` review state at mobile widths = M03 |
| RF-10.2 | — | Other three states reflow; statements shared |
| RF-10.3 | — | `shared/ui` targets, labels, focus |
| RF-10.4 | — | Breakpoint layout replaceable per state (S-03) |
| RF-11.1 | Audit adapter events in the effect's transaction | — |
| RF-11.2 | Audit payload shape; unmatched address never stored | Correlation id header |
| RF-11.3 | No message template and no email port in this feature; the only outbox write is the `InvitationSettled` event consumed by plan 010 | — |
| RF-11.4 | Secret excluded from logs, problems, traces; safe codes | No secret in analytics |
| RF-11.5 | — | Holder in memory only; nothing in storage; history replaced |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · The invitation record and the acceptance transaction live in the `access` bounded context; identity and tenant facts arrive through published ports.**
*Why.* Register `122:2` §5 assigns memberships, invitations, project grants and permission versions to `access`; acceptance creates a membership and grant records in the same transaction as the invitation's transition (RF-6.1), which is only possible without foreign repositories if all three aggregates share one context (P11). The identity's facts (normalised email, status, verification, display name) and the tenant's status are reads of other contexts, so they are published ports (`IdentityFactsPort`, `TenantStatusPort`) with no write surface. The BE section's remark that `identity.application` adds "invite/status use cases" is read as the identity-facing port and the signup check (D11), not as ownership of the record.
*Discarded.* Placing the use case in `identity` — it would write `access` tables through foreign repositories. Duplicating membership facts in `identity` — two sources of truth for `permissionVersion`. A separate `invitation` context — nothing distinguishes it from `access`, and the atomic transaction would cross a boundary.
*Covers.* RF-5.1, RF-5.2, RF-6.1, RF-6.2, RF-6.9, RNF-10.

**D2 · The link secret is a 256-bit random value stored only as a fast cryptographic hash; opening the link resolves it without consuming it.**
*Why.* A high-entropy secret needs no work factor, so a plain cryptographic hash (`TokenHasher` **[002]**, as plans 002, 004 and 006 use) keeps lookup cheap and leaves nothing usable at rest (RF-1.1, RNF-8). Resolving on mount without a write (RF-1.6, RF-2.6) gives the person the right state immediately, lets a mail scanner or a curious click change nothing (LC-29), and keeps the review state stateless on the server. The secret is generated at issue by specification 010 with `TokenGenerator` **[002]**; this plan fixes only the hash and the lookup.
*Discarded.* Consuming on open — a scanner would burn the invitation. Storing the secret in clear — F-X-03. Hashing with the password encoder — needless cost for a random value.
*Covers.* RF-1.1, RF-1.6, RF-2.6, RF-4.5, RNF-8, LC-29.

**D3 · One pure `AcceptanceRule` decides usability, evaluated at resolve and again inside the accept transaction; every cause maps to one external state.**
*Why.* RF-5.3 lists five conditions and RF-5.8 requires them re-evaluated at the moment of acceptance; a pure function with explicit inputs is testable without a database and guarantees that the state shown and the transaction's decision cannot disagree. The cause is kept for audit (RF-11.1) and mapped to the single code `access.invitation.unavailable` (RF-5.6, RF-9.2, D-a). The resolve path performs the same work in every branch — hash, lookup, tenant read, identity read when a session exists — so timing does not separate the causes (RNF-2).
*Discarded.* Distinct problem codes per cause — enumeration of tenants, addresses and invitation states. Client-side gating from the resolve response alone — P3, and a stale review would accept.
*Covers.* RF-5.3–5.6, RF-5.8, RF-9.2, RNF-2, RNF-5, LC-6–LC-8, LC-11–LC-16.

**D4 · Acceptance is one transaction with row locks on the invitation and the (tenant, identity) membership and a conditional update of the invitation; a suspended tenant is detected before the update so nothing is consumed.**
*Why.* Atomic single use under concurrency (RF-6.8, RNF-1, D-c) needs one statement that both tests and marks the invitation: `UPDATE … WHERE state = 'PENDING' AND expires_at > now()`. Exactly one of two concurrent acceptances, or of an acceptance and a revocation or decline, updates a row; the others see zero rows. Locking the membership row (or its absence through the invitation lock, since both run under the same tenant context) serialises the existing-membership check with the write. Evaluating `TENANT_SUSPENDED` before the update implements D-d: the invitation stays `PENDING` and becomes acceptable when the tenant returns to active. Membership, grants, permission facts and audit are written in the same transaction, so a crash can never leave an accepted invitation without its membership (RNF-10).
*Discarded.* Read-then-write under optimistic versioning — a retry loop with the same guarantee and more code. A two-phase token exchange (resolve issues a short-lived accept token) — a second secret with nothing the conditional update does not give. Consuming the invitation of a suspended tenant and asking for a fresh one — rejected by D-d.
*Covers.* RF-5.4, RF-5.8, RF-6.1, RF-6.8, RNF-1, RNF-10, LC-7–LC-10, LC-13.

**D5 · The accept and resolve paths establish an invitation-scoped tenant context with `SET LOCAL` derived from the record; only the `secret_hash → tenant_id` lookup precedes it, through one audited `SECURITY DEFINER` function.**
*Why.* All three tables are tenant-scoped under `FORCE RLS` (P12), but the person accepting is not yet a member and the request carries no tenant identifier (P3). The tenant is a fact of the invitation record, so the server derives it from the hash — the one lookup that cannot run under a tenant context — through a `SECURITY DEFINER` function that returns the tenant id and nothing else, audited as F-X-01 requires. From that point the transaction runs under `SET LOCAL` tenant context, so every read and `WITH CHECK` on the write passes without a bypass role, and pooled connections never inherit the scope (transaction-local only). The C10 for this operation is explicit and narrow: actor = signed-in identity, tenant = invitation's tenant, authority = "accept or decline this invitation"; it grants nothing else and is never reused.
*Discarded.* A platform role that bypasses RLS for the acceptance — P12 forbids a bypass of either gate. Writing membership rows before setting the context — fails `WITH CHECK` by design. Carrying the tenant id in the link — the client would be stating scope, and a tampered link would target another tenant's row (harmless under RLS but a needless surface). A global, non-RLS invitation table — the record is tenant data and must be erased with the tenant (FR-IDN-125 by analogy).
*Covers.* RF-1.1, RF-6.1, RNF-3, RNF-9, RNF-10.

**D6 · One membership row per (tenant, identity), as plan 009 keys it: a revoked row is reactivated in place with the invited role and `permissionVersion + 1`; the invited project's grant row is created or revived through `setGrants`, and every other revoked grant row stays revoked.**
*Why.* Decision Q5 wants a fresh membership carrying only the invited scope while authored evidence keeps its attribution. Plan 009's keys make this the only possible shape: `tenant_membership` is unique per (tenant, identity) and `project_grant` is one row per membership and project, so "a new membership" at the specification level is the same row reactivated with a new permission version, and "no earlier grant restored" means that `setGrants` is called for the invited project only — reviving that project's row with the invited `GrantSet` — while the other revoked rows keep their `revokedAt`. Every reference to the membership id (evidence, audit, provenance) stays valid. Bumping `permissionVersion` is what F-X-01 requires for sensitive writes and what the C10 derivation **[006]** checks.
*Discarded.* A second membership row per identity — impossible under plan 009's unique key, and it would break every reference to the membership. One grant row per role kind with a partial unique index (this plan's first draft) — plan 009 D5 chose one row per membership and project with four flags; two models of the same table cannot coexist. Restoring the old grants — rejected by Q5. Deleting the old grant rows — destroys attribution.
*Covers.* RF-6.1, RF-6.2, RF-6.6, RNF-4, LC-17.

**D7 · The granted scope is derived from the invitation record alone; the accept request carries no role, project or grant field.**
*Why.* RF-6.3 and FR-IDN-084: the client may not state what it receives. With no scope in the body there is nothing to validate, compare or reject; the review state is a display of the record and acceptance applies the record. The `tenant_role` column's check constraint excludes `OWNER` at the database as well, so no path — this feature's or specification 010's — can create an Owner by invitation.
*Discarded.* Echoing the reviewed scope back for confirmation (a body hash) — a comparison that can only fail when the record changed, which the conditional update already handles; it adds a client-stated scope to the wire for no gain.
*Covers.* RF-2.2, RF-6.3, RNF-4, LC-21, LC-22.

**D8 · Before the identity matches, the server discloses the tenant name and a domain-preserving mask of the invited address, computed server-side.**
*Why.* Decision Q2: a link holder learns what the message already told them and nothing more; the mask orients the invitee ("sign in with m•••@example.test") without giving a forwarded link the full address. The same two facts appear on the wrong-identity panel (RF-3.1). The full scope is returned only in the `REVIEW` outcome, which requires the session identity to equal the invited address (RF-1.3).
*Discarded.* Full scope to any holder — rejected by Q2. Nothing before sign-in — rejected by Q2 (the person could not tell which account to use). Client-side masking — the client would need the full address.
*Covers.* RF-1.2, RF-3.1, RF-9.3, LC-1, LC-5.

**D9 · Accept is idempotent on a client-generated key recorded on the invitation row; decline is idempotent by state.**
*Why.* F-X-02 and LC-9 / LC-29: a reload or a lost answer during the accept call must not create two memberships or show the unavailable state for an acceptance that happened. The invitation row is the natural outcome record: `accept_idempotency_key` and `accepted_by` are written in the same statement that consumes it, so a re-submission with the same key by the same identity returns the recorded `ACCEPTED` answer and its routing facts; a different key meets a non-pending row and receives the unavailable state. Decline needs no key: a second decline meets `DECLINED` and is answered as unavailable, which is the designed outcome.
*Discarded.* Generic POST retry — forbidden. A separate outcome table — the row already is the outcome.
*Covers.* RF-6.1, RF-6.8, RF-7.2, LC-9, LC-29, LC-30.

**D10 · Throttling reuses `AbuseThrottlePolicy` [001] with one new key, `INVITATION_NETWORK`; no per-address key exists.**
*Why.* RF-9.1 requires throttling of resolutions, acceptances and declines per calling network and risk signal, independent of the sign-in throttles, lifting automatically. The policy is pure and already tested; a new key gives independent steps and windows. There is deliberately no per-address or per-invitation key: the invited address is the holder's own, and counting on it would let a third party who holds the link block the invitee (RNF-7). Guessing a 256-bit secret is infeasible regardless; the network key bounds the probe rate and the audit volume.
*Discarded.* Sharing the sign-in keys — an invitation flood would throttle sign-in and vice versa. A per-invitation key — a denial of the invitee by anyone with the link. Edge rate limiting only — not designed (A-20).
*Covers.* RF-9.1, RF-9.4, RNF-7, LC-33.

**D11 · The FE keeps the link in a tab-scoped in-memory holder; "Return to your invitation" exists only while the holder exists; the invited signup is enforced server-side by an address check against the invitation.**
*Why.* Decision Q3 makes the link itself the return route and RF-11.5 forbids storing it; F-X-03 allows browser storage only for harmless preferences, which a secret is not. A module-private value survives SPA navigation within the tab (invitation → sign-in → verification pending → sign-in), which is exactly the case RF-4.4 describes, and vanishes in a new tab, which is LC-26 (the person re-opens the link from the message). For the invited signup, decision Q2 forbids disclosing the full address before the identity matches, so the client cannot pre-fill it from the resolve response; the server therefore enforces RF-4.1's intent — the account must be created with the invited address — through `InvitationAddressCheckPort`, which `SignUpUseCase` **[002]** consults when a link secret accompanies the form, refusing a different address with a field problem that reveals nothing. The field shows the masked address as its hint and the statement of RF-4.1.
*Discarded.* `sessionStorage` — readable by any script in the origin and outside the F-X-03 allowlist. `localStorage` — shared across tabs and persistent. A server-side handoff binding the invitation to the pending identity — rejected by Q3. Pre-filling the full address from the resolve response — contradicts Q2. A read-only field with the masked value — unusable.
*Covers.* RF-3.2, RF-4.1, RF-4.4–4.6, RF-11.5, RNF-8, LC-24–LC-26.

**D12 · After acceptance the FE navigates through the same tenant entry `LandingRouter` [001] uses for `ENTER_TENANT`, with the granted project preselected; authority and the enrollment gate come from that entry, never from the accept response.**
*Why.* Decision Q6 lands the person in the tenant; the tenant entry already re-derives C10 on every request (F-X-01) and raises `auth.enrollment.required` when `EnrollmentGate` **[003]** withholds authority, which `EnrollmentGateHandler` **[003]** turns into the wizard with a `returnTo` (RF-6.7). The accept response carries `tenantId` and `projectId` as routing facts only (P3); the guarded switch of specification 016 applies when written — from the invitation page there is no dirty editor and no previous scope, so its steps reduce to the epoch advance and the fresh membership fetch that plan 001 D8 already performs before landing.
*Discarded.* Injecting the new membership into the client cache from the response — the client would be asserting authority. Landing on the tenant chooser — rejected by Q6.
*Covers.* RF-6.5–6.7, RF-4.7, LC-18–LC-20.

**D13 · This feature sends no message; every outcome is audited through `AuditPort`, and the settled invitation is published as an outbox event for plan 010's inviter notification.**
*Why.* Decision Q7 and specification 010 Q10: the invitee performed the act in a complete session and needs no message; the inviter is told by specification 010, which owns the invitation message set. Publishing `InvitationSettled{ACCEPTED | DECLINED}` from the same transaction (plan 010 §2.1 `InvitationSettledHandler`) keeps the template, the locale and the delivery in 010 and keeps this feature free of any email adapter. The audit events of RF-11.1 give the tenant and the platform the reconstruction FR-IDN-099 requires.
*Discarded.* Sending the inviter notification from this feature — a second owner of 010's template inventory. A security notification to the invitee — the invitee performed the act in a complete session. No event at all — 010 would have to poll the invitation table.
*Covers.* RF-11.1–11.3, RNF-15.

**D14 · One `InvitationPage` component renders the four states at every width; the review state follows M03 at mobile widths and N02 at desktop widths; the other states reflow.**
*Why.* RF-10.1 names M03 as the designed counterpart and RF-10.2 reflows the rest; one component with breakpoint layouts and statements drawn from `SecurityStatements` **[001]** keeps every statement, gate and action present at every width by construction (RNF-14).
*Discarded.* A separate mobile component — drift between the two.
*Covers.* RF-10.1–10.4, RNF-14.

**D15 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id [001 D12].**
*Why.* F-X-03; the acknowledgement problem needs a field path (RF-2.4); one code for every unusable cause keeps RF-5.6 and RF-9.2 generic; the throttle answer carries no cause; the correlation id joins the audit record to the person's report.
*Discarded.* Ad-hoc error bodies; a cause in the problem title.
*Covers.* RF-2.4, RF-5.6, RF-9.1, RF-9.2, RF-11.4.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure domain: `AcceptanceRule` for every cause and order, `Invitation` state transitions, `ProjectGrantSet.toGrantSet`, `TenantMembership.createFromInvitation` / `reactivateFromInvitation`, `EmailMask`, `AbuseThrottlePolicy` with the invitation key, `ResolutionOutcome` mapping. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: RLS policies with and without context, the `SECURITY DEFINER` lookup, conditional update, row locks, permission facts, throttle upserts, audit rows, configuration binding, migrations. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of the secret, the inviter, members or other tenants in responses. |
| Timing comparability | JUnit 5 statistical harness (test-only) | N samples per unusable cause; distributions overlap within a configured tolerance. |
| Concurrency | JUnit 5 with real PostgreSQL | Two-tab accept, accept versus decline, accept versus revocation, accept versus expiry. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page states, checkbox validation, holder lifecycle, handoffs, statements present at every width, nothing in storage, URL secret replaced. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Focus order, announcements, no focus theft, labels, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RNF-8 | BE integration | `resolve.looksUpByHashOnlyAndStoresNoSecret`; `logs.containNoSecret` | 12 |
| RF-1.2, LC-1 | API contract, FE unit | `resolve.noSessionReturnsTenantNameAndMaskOnly`; `invitationPage.preMatchStateWithRoutes` | 4, 7 |
| RF-1.3, LC-2, LC-4 | BE integration, FE unit | `resolve.matchedIdentityGetsReview` (equal, case-different, whitespace); `invitationPage.reviewState` | 4 |
| RF-1.4, LC-3, LC-5 | BE integration, FE unit | `resolve.otherIdentityGetsWrongIdentityWithNoScope`; `invitationPage.wrongIdentityState` | 4, 7 |
| RF-1.5, LC-6 | API contract | `resolve.unknownSecretIsUnavailableOneCode` | 6, 7 |
| RF-1.6, RF-2.6, LC-29 | BE integration | `resolve.changesNothing` (state, membership, audit-only) | 3 |
| RF-1.7 | FE unit | `invitationPage.rendersNoHelpAndDesignedLegalRoutes` | 1 |
| RF-1.8, LC-32 | FE e2e | `languageSwitch.keepsOutcomeAndHolder` | 13 |
| RF-1.9 | BE unit | `emailAddress.normalisationUsedForInvitedAndSession` | 4 |
| RF-2.1, RF-8.1, RF-8.2 | FE unit, e2e | `reviewState.rendersScopeBlocksStatementsAndActions` (desktop, tablet, mobile; with and without project) | 1, 13 |
| RF-2.2, RF-6.3, LC-21, LC-22 | API contract, BE integration | `accept.bodyHasNoScopeFields`; `accept.ignoresExtraFieldsAppliesRecord`; `schema.tenantRoleRejectsOwner` | 5 |
| RF-2.3 | FE unit | `securityStatements.presentOnInvitationStates` | 1 |
| RF-2.4, LC-23 | FE unit, API contract | `reviewState.blocksWithoutAcknowledgement`; `accept.acknowledgementFieldProblem` | 2 |
| RF-2.5, RNF-9 | API contract | `reviewResponse.exposesOnlyInvitedScope` | 7 |
| RF-3.1 | FE unit | `wrongIdentityState.showsTenantNameMaskAndTwoActions` | 4 |
| RF-3.2, LC-3 | FE e2e (sign-out stub of 007) | `wrongIdentity.switchSafelySignsOutKeepsHolderReturnsAfterSignIn` | 4, 8 |
| RF-3.3 | FE e2e | `wrongIdentity.leaveDropsHolderAndLands` | 4 |
| RF-3.4 | BE integration | `accept.recomparesIdentityInsideTransaction` | 4 |
| RF-3.5 | API contract | `noLinkingEndpointOrControlExists` | 4 |
| RF-4.1, LC-25 | BE integration, FE unit | `signup.withSecretRefusesDifferentAddress`; `signup.withSecretAcceptsInvitedAddress`; `signupPage.invitedVariantHintAndStatement` | 8 |
| RF-4.2, RF-4.3 | BE integration | `signupVerifySignIn.leaveInvitationPending` | 8 |
| RF-4.4, LC-24 | FE e2e | `newIdentity.returnViaConfirmedStateAndSignIn` | 8 |
| RF-4.5, LC-26 | FE e2e | `newIdentity.reopensLinkFromMessageInNewTab` | 8 |
| RF-4.6 | FE e2e | `newIdentity.otherAddressGetsWrongIdentity` | 4, 8 |
| RF-4.7, RF-6.7, LC-18, LC-19 | BE integration (gate of 003), FE e2e | `accept.withoutFactorSucceeds`; `tenantEntry.adminWithoutFactorIsGated`; `tenantEntry.policyMemberWithoutFactorIsGated` | 9 |
| RF-5.1, RF-5.2 | BE unit | `invitation.statesAndIsPending`; `projectGrantSet.atMostOneProject` | 5 |
| RF-5.3 | BE unit, integration | `acceptanceRule.everyConditionAndOrder`; `accept.requiresSession401` | 6 |
| RF-5.4, LC-13, LC-14 | BE integration | `accept.suspendedTenantLeavesPendingThenAcceptableWhenActive`; `accept.deletingTenantUnusable` | 6 |
| RF-5.5, LC-15, LC-16 | BE integration | `accept.activeOrSuspendedMembershipUnusableAndUntouched` | 6 |
| RF-5.6 | API contract, FE unit | `unavailable.oneCodeNoTenantNoCause`; `invitationPage.unavailableStateRoutes` (signed in → chooser `/workspaces`, signed out → sign-in) | 6, 7 |
| RF-5.7, RF-9.4, RNF-6 | Startup | `invitationPolicy.startupFailsOnAbsentOrUnboundedValue` (one case per property) | 11 |
| RF-5.8, LC-7, LC-8 | BE integration | `accept.staleReviewAfterExpiryOrRevokeIsUnavailable` | 6 |
| RF-6.1, RNF-10 | BE integration | `accept.oneTransactionInvitationMembershipGrantsFactsAudit`; `accept.crashBetweenStepsLeavesNothing` | 5 |
| RF-6.2, LC-17 | BE integration | `accept.revokedMembershipReactivatedWithNewVersionOldGrantsStayRevoked` | 5 |
| RF-6.4 | BE integration | `accept.touchesOnlyAccessTables` (session, credential, factor, other memberships, runs unchanged) | 5 |
| RF-6.5, LC-20 | FE e2e | `accepted.entersTenantWithProject`; `accepted.membershipOnlyEntersTenantWithoutProject` | 9 |
| RF-6.6 | BE integration | `accept.nextRequestDerivesNewPermissionVersion` | 5 |
| RF-6.8, RNF-1, LC-9, LC-10 | Concurrency | `accept.twoTabsOneSuccess`; `accept.vsDeclineOneTransition`; `accept.vsRevokeOneTransition`; `accept.vsExpiryOneTransition` | 3 |
| RF-6.9 | BE integration | `accept.rowsVisibleToPendingAndMembersQueries` (query shapes stubbed from 010) | 5 |
| RF-7.1 | API contract | `decline.requiresSessionAndInvitedIdentity` | 3 |
| RF-7.2, LC-12 | BE integration, FE e2e | `decline.finalAuditedNothingElse`; `decline.dropsHolderAndLands` | 3 |
| RF-7.3, LC-11, LC-30 | BE integration | `resolve.acceptedOrDeclinedIsUnavailable` | 3 |
| RF-8.3 | FE unit | `preMatchAndSignup.carryMfaStatement` | 1 |
| RF-9.1, RNF-7, LC-33 | BE integration | `throttle.invitationKeyProgressiveBoundedLifts`; `throttle.invitationFloodNeverRestrictsSignIn` | 10 |
| RF-9.2, RNF-2 | API contract + timing | `unavailable.timingComparableAcrossCauses` (unknown, expired, revoked, used, declined, suspended, deleting, membership) | 7 |
| RF-9.3 | API contract | `responses.revealNoIdentityFactorTenantListOrOtherInvitation` | 7 |
| RF-10.1–10.3, RNF-12, RNF-14 | FE e2e + axe, manual assistive | `a11y.invitation.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 13 |
| RF-11.1, RF-11.2 | BE integration | `audit.entryPerEvent`; `audit.containsNoSecretOrUnmatchedAddress` | 12 |
| RF-11.3, RNF-15 | BE integration | `outbox.untouchedByAcceptAndDecline` | 12 |
| RF-11.4, RF-11.5, RNF-8 | BE integration, FE unit | `problems.containNoSecret`; `storage.containsNothingFromFeature`; `url.secretReplacedInHistory` | 12 |
| RNF-3 | API contract | `linkAloneGrantsNothing` (no session → no accept; forwarded link → wrong identity) | 4 |
| RNF-4 | BE integration | `accept.scopeEqualsRecordElementByElement` | 5 |
| RNF-5 | BE integration | `failClosed.everyUnusableConditionRefuses` | 6 |
| RNF-9, P12 | BE integration | `rls.rejectsMembershipWriteWithoutContext`; `rls.acceptWritesOnlyUnderInvitationScopedContext`; `securityDefiner.returnsTenantIdOnly` | 5 |
| RNF-11 | BE integration | `accept.grantsNoAuthorityBeforeGate` | 9 |
| RNF-13 | FE e2e | `allStates.existInEnEsFr` | 13 |
| LC-27 | BE integration | `newIdentity.linkExpiredDuringVerificationIsUnavailable` | 6 |
| LC-28 | BE integration | `restrictedOrErasingIdentityCannotReachReview` | 6 |
| LC-31 | FE e2e (expiry of 006 stubbed) | `sessionExpiryOnReview.reauthThenReresolve` | 3 |
| LC-34 | BE integration | `emailChannelDown.noEffectOnAcceptance` | 12 |
| LC-35 | API contract | `noTenantSurfaceAcceptsForAnother` | 4 |
| US-9 | all adversarial cases above | enumeration, forwarded link, wrong identity, escalation, Owner by invitation, reuse, admin backdoor suites | 1 |

### 5.3 Not tested here
Sign-in, challenge and landing mechanics (plan 001), signup and verification mechanics beyond the address check (plan 002), enrollment and the gate's internals (plan 003), session issue, refresh and expiry (plan 006), sign-out (specification 007), issuing, listing and revoking invitations (specification 010), the guarded switch's internals (specification 016). Their stubs return fixed answers; `IdentityFactsPort`, `TenantStatusPort` and `SessionAssertionPort` are verified by recording stubs and, in the integration suite, by the owning contexts' test doubles.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 14 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-52 invitation record and acceptance transaction (decided) | `Invitation` with the decided fields and states; one transaction with the conditional update; revoked membership reactivated (D1, D4, D6). | If specification 010 extends the record (several projects, a note, a resend), `ProjectGrantSet` becomes a list and `setGrants` is called once per invited project; `AcceptanceRule` is unchanged. |
| A-28 handoff (decided) | Tab-scoped in-memory holder; the link is the return route; the invited signup enforced server-side by `InvitationAddressCheckPort` (D11). | If a server-side binding of the invitation to the pending identity is wanted later, `SignUpUseCase` **[002]** stores the invitation id on the pending identity and `VerifyEmail` returns a return route; D11's holder is kept for the same-tab case. |
| A-27 identity statuses | `IdentityFactsPort.facts` reports the A-27 model; `AcceptanceRule` requires `ACTIVE` and verified. | Change the predicate only. |
| A-36 identity-scoped reads | The identity's own membership in the invitation's tenant is read under the invitation-scoped context (D5), not through the landing projection. | If a dedicated policy is decided, the read moves behind it; the rule is unchanged. |
| A-35 persistence tooling | Adapters and migrations against the placeholder API and tool. | Rename base types and migration format; no use case changes. |
| S-45 decline on desktop | "Decline invitation" on N02 beside "Use another identity"; one compact confirmation dialog (D02 `424:3689`); chooser after decline. | None: the dialog is drawn; `DeclineInvitationUseCase` is unchanged. |
| S-46 pre-match state | Tenant name, masked address, two routes and the statements as the specification lists them. | Adjust the page state and i18n keys only. |
| S-24 return route (decided) | "Return to your invitation" on the confirmed state and the sign-in screen while the holder exists; the invited signup shows the masked address as a hint with an editable field enforced server-side (D11, a refinement of the register's "pre-filled and locked" for the design team). | If the design wants a locked field, the resolve response for the signup handoff must carry the full address, which contradicts Q2 unless disclosure is tied to a proof; a register note is needed first. |
| S-03 mobile counterparts | M03 for the review state; M19 (`432:4231`) for the pre-match state; the D01 policy for the other two. | None. |
| S-04 issuing dialog | Only the record contract (A-52) is assumed; no acceptance behaviour depends on the dialog. | None. |
| S-14 templates | No message from this feature. | None unless Q7 is reopened. |
| S-17 "Help" | Opens N43 HelpShortcutsPanel (`430:4129`) as drawn. | None: the destination is designed. |

The specification's derived rules are built, not assumed: one unusable state and network-only throttling (D3, D10), acknowledgement required and never pre-ticked (RF-2.4 trace), atomic single use (D4, D9), the suspended-tenant hold (D4), and acceptance changing nothing but the membership (D4, D12).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.9) | `ResolveInvitationQuery`, `ResolutionOutcome`, `EmailMask`, `InvitationPage` pre-match, holder, `EmailAddress` | D2, D3, D5, D8, D11 | RF-1 rows |
| RF-2 (2.1–2.6) | `REVIEW.scope`, review state, `SecurityStatements`, acknowledgement problem | D7, D8, D14, D15 | RF-2 rows |
| RF-3 (3.1–3.5) | `WRONG_IDENTITY`, N29 state, sign-out handoff, `AcceptanceRule` identity check | D3, D8, D11 | RF-3 rows |
| RF-4 (4.1–4.7) | `InvitationAddressCheckPort`, `SignupPage` variant, `VerifyEmailPage` / `SignInPage` return, `InvitationReturn` detour, gate at entry | D11, D12 | RF-4 rows |
| RF-5 (5.1–5.8) | `Invitation`, `ProjectGrantSet`, `AcceptanceRule`, `TenantStatusPort`, `IdentityFactsPort`, `InvitationPolicyProperties`, unavailable state | D3, D4, D7 | RF-5 rows |
| RF-6 (6.1–6.9) | `AcceptInvitationUseCase`, `TenantMembership`, `ProjectGrant`, `PermissionFactsPort`, tenant entry handoff, `EnrollmentGate` [003] | D1, D4, D5, D6, D7, D9, D12 | RF-6 rows |
| RF-7 (7.1–7.3) | `DeclineInvitationUseCase`, decline action, landing | D4, D9 | RF-7 rows |
| RF-8 (8.1–8.3) | `SecurityStatements`, "Next destination" | D14 | RF-8 rows |
| RF-9 (9.1–9.4) | `AbuseThrottlePolicy` key, one problem code, constant-work resolve, `InvitationPolicyProperties` | D3, D10, D15 | RF-9 rows |
| RF-10 (10.1–10.4) | `InvitationPage` breakpoints, `shared/ui` | D14 | RF-10 row |
| RF-11 (11.1–11.5) | Audit adapter, correlation id, holder, redaction, no outbox | D9, D13, D15 | RF-11 rows |
| RNF-1 | conditional update, row locks | D4, D9 | concurrency |
| RNF-2 | one code, constant-work path | D3 | timing + contract |
| RNF-3 | session required, server comparison, no scope in body | D5, D7 | `linkAloneGrantsNothing` |
| RNF-4 | record-derived scope, `setGrants` with the invited `GrantSet` | D6, D7 | `scopeEqualsRecord` |
| RNF-5 | `AcceptanceRule`, config validation | D3 | `failClosed` |
| RNF-6 | `InvitationPolicyProperties` | — | startup |
| RNF-7 | network key only, bounded windows | D10 | RF-9.1 rows |
| RNF-8 | hash only, holder in memory, redaction | D2, D11 | RF-1.1, RF-11 rows |
| RNF-9 | RLS, invitation-scoped context, scope-only responses | D5 | `rls.*`, RF-2.5 |
| RNF-10 | one transaction with permission facts | D4, D6 | RF-6.1, RF-6.6 rows |
| RNF-11 | gate at entry, no authority in response | D12 | RF-4.7 rows |
| RNF-12, RNF-13, RNF-14 | `shared/ui`, `shared/i18n`, breakpoints | D14, D15 | a11y, language rows |
| RNF-15 | no outbox use | D13 | `outbox.untouched` |
| LC-1 … LC-35 | per the §5.2 rows that name them | — | §5.2 |

Every EARS criterion RF-1.1 … RF-11.5 appears in §3; every RNF and LC appears in §5.2 or above. No element of §2 lacks a criterion, and no criterion lacks an element, a decision and a test.
