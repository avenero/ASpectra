# Plan 015 — Leave tenant and identity erasure request

**Feature** Leaving one tenant membership, and requesting the verified erasure of one's own global identity · **Plan version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — aligned with specification v0.1 (decisions Q1–Q7, derived rules D-a…D-d) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/015-LeaveTenant/specLeaveTenant.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-17 as the subject; F-IDN-03, F-IDN-05, F-IDN-07, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-15, F-IDN-16, F-SET-02 and F-SET-03 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG; contexts `identity`, `access`, `tenant`, `execution`, `audit`), FE board `46:7847` (feature-modular SPA DAG) and decision register `122:2` (§3: "Tenant membership removal revokes that membership/project grants, not the global identity … Global account erasure is a separate verified workflow"; §5: the `access` context owns memberships, invitations, project grants and permission versions; §14: A-64, A-27, A-48, A-30, A-56, A-11) · **Builds on** plan 001 (Login: `GlobalIdentity`, `IdentityStatus`, `IdentityRepository`, `SignInUseCase` generic branch for `ERASING`, `MembershipProjectionPort`, `AuditPort`, `Clock`, `SecurityNotificationPort`, the idempotency outcome record of 001 D1, problem conventions of 001 D12, `GenericFailureMessage`, `PasswordField`, `CodeField`), plan 002 (outbox `outbox_message` and relay, `TransactionalEmailPort`, message language by request locale 002 D15), plan 003 (`StepUpChallenge`, `StepUpUseCase`, `StepUpRepository`, `EnrollmentPolicy`, the step-up endpoint, `SecuritySettingsPage` with its Privacy slot), plan 004 (`GlobalIdentity.bumpSecurityVersion()`, `RevokeSessionsPort`, `isResetEligible()`), plan 005 (`ContinuationChallenge` as the shape copied, `RecoveryCaseClosurePort.close(identityId, IDENTITY_ERASING)`), plan 006 (`EndSessionUseCase.revokeAll`, `AssertSessionUseCase`, `DeniedScopeHandler`, end cause `IDENTITY_STATUS`), plan 007 (`SignOutCoordinator`, the `SignOutEverywhereUseCase` sequence), plan 008 (`IdentityFactsPort`, `AcceptanceRule` → `UNUSABLE(IDENTITY_INELIGIBLE)`), plan 009 (`TenantMembership.revoke`, `MembershipRepository`, `MembershipCommands`, `AuthorityChanged` and `AuthorityChangedHandler`, `WorkRevocationPort`, `SubscriptionRevocationPort`, `PermissionVersionGuard`), plan 010 (`TenantOwnership` lock, `TenantOwnershipRepository`, `ActiveOwnerRule`, `LastOwnerGuard`, `StepUpPort`, `StepUpFields`, `SuccessorsQuery`, `TransferOwnershipDialog`, the N23 anatomy), plan 013 (`/workspaces`, `TenantChooserPage`, the identity-scoped `MembershipProjectionPort` extension) and plan 014 (`TenantStatusPort.statusOf` / `requireActiveForEffect`, `TenantAdministratorsQuery`, `IdentityLocalePort`, and the contracts of 014 D13 for leaving a suspended tenant and holding an erasure); elements they own are tagged **[001]** … **[014]** and are extended, never duplicated. Plan 016 (guarded switch) is not written; this plan reaches it only through the contract named in §6.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's seven decisions (Q1–Q7) and four derived rules (D-a…D-d) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-64, A-27, A-48, A-30, A-56, A-11, A-36, A-35; S-63, S-14, S-02, S-04), this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | Neither operation approves, accepts or launches anything; work of the person is only refused or asked to stop through plan 009's ports; approvals already given stay as evidence. | RF-4.7, RF-7.9, RF-9.4 |
| **P3** The browser expresses intent, never authority | The ownership checks, the leave permission and the erasure gates are server-built read models and server rules; the client sends a membership id or the gate inputs as intent. The typed phrase is matched on the server (D9); the client match only enables a button. | RF-2.3, RF-4.4, RF-5.4, RF-6.1, RNF-1 |
| **P6** Fail closed | An unknown tenant status, an unreadable owner count, a missing step-up, an unverifiable code or an unsent code refuse the operation; a gate that cannot be evaluated counts as failing; no record is written before the last proof verifies. | RF-4.4, RF-6.1, RF-6.5, RF-7.1, RNF-12 |
| **P8** No fabricated outcome | Work revocation ends queued tickets NOT EXECUTED and requests bounded cancellation, never "failed"; leaving and erasure write no row of a run, attempt, fact, verdict or approval. | RF-4.7, RF-7.9, RF-9.4 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security password and TOTP verification through plan 003, the existing outbox. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. Zustand is not used (nothing is a deliberate unsaved fork; D14). STOMP is only closed through plan 009's port. Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: erasure in the hexagonal context `identity` (the identity record's owner); leaving and the ownership checks in `access` (the membership owner); cross-context effects through published ports and outbox events only. FE: N19 in the feature module `identity-access`, dialogs reusing `workspace-members` components through its public exports; acyclic imports. | §2 |
| **P12** Two independent tenancy gates | Membership rows stay under `FORCE RLS` with `SET LOCAL`; the ownership checks and the leave use the identity-scoped predicate of A-36 (plan 009 D10), then the tenant predicate for the one tenant being changed. The erasure request record is an identity row, not a customer row (no `tenant_id`), unreadable by the tenant runtime role. Runtime authorisation is the second gate in every command. | RF-2.1, RF-2.6, RF-4.4, RNF-7 |
| **P13** Configured values are never defaulted | Page size, email-code lifetime, attempt limit, resend limit and resend window, and the erasure process contact are one validated configuration record `ErasurePolicy`; step-up limits stay in `EnrollmentPolicy` **[003]**. Absent or unbounded values fail startup. | RF-2.2, RF-6.3, RF-6.4, RF-8.3, RNF-9 |
| **P14** Release bar | WCAG 2.2 AA on N19, the leave dialog and every verification step with automated and manual assistive evidence; EN / ES / FR for every statement, the typed phrase and the three messages. | RF-11, RNF-10 |
| **F-X-01** C10 context | No tenant C10 is derived for leaving (plan 014 D13): the command is identity-scoped and authorised by the session (`AssertSessionUseCase` **[006]**) plus the person's own membership row; erasure is identity-scoped. Every effect re-reads tenant status inside its transaction (`requireActiveForEffect` **[014]** where the rule needs an active tenant). | RF-1.1, RF-4.1, RF-4.4 |
| **F-X-02** Request integrity | Each command carries an idempotency key recorded in the outcome record **[001 D1]**; the leave checks the membership's `permissionVersion` **[009]**; one erasure request per identity by a unique key. | RF-4.11, RF-7.3, RNF-4 |
| **F-X-03** Secrets and safe problems | Passwords and codes exist in clear only inside the verifying call; codes hashed at rest; problems carry safe codes (`access.leave.*`, `identity.erasure.*`, `mfa.step-up.*` **[003]**) with a correlation id; nothing in browser storage. | RF-6.6, RF-10.3, RNF-8 |
| **F-X-04** Accessibility and languages | Labels and associated errors on every field; state as text; focus to the heading on page, dialog and step; EN / ES / FR bundles. | RF-11.2, RF-11.3 |
| **F-X-05** Audit log | Every event of RF-10.1 and RF-10.2 is appended through `AuditPort` **[001]** in the effect's transaction, without secrets. | RF-10.1, RF-10.2, RNF-11 |
| **F-X-06** Email port | Three outbox templates and the erasure code template behind `SecurityNotificationPort` **[001/002]** and `TransactionalEmailPort` **[002]**; the code send is confirmed before the code step opens (D11). | RF-6.3, RF-8, RNF-12 |
| **F-X-07** Responsive surfaces | M22 (`432:4356`) is the drawn counterpart of N19 (S-63); one page component follows the D01 responsive policy at other widths, the table becoming records. | RF-11.1 |
| **F-IDN-17** Leave tenant and identity erasure request | Two different commands in two contexts (D1, D3); leave blocked for the last Owner with the transfer route; erasure gates, typed confirmation, last-Owner checks in every owned tenant, never one-click, no instant-erasure promise; only the person's own checks shown. | RF-1 … RF-9 |

---

## 2. Module structure

### 2.1 Back end

#### 2.1.1 Bounded context `com.aspectra.identity` — erasure request (owned here)

Plans 001–007 own the identity record and read `ERASING` in sign-in, reset, recovery, enrollment and session assertion; no plan owns the transition into it. This plan defines it, the erasure request, the verification chain and the handoff.

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `GlobalIdentity.beginErasure(now)` (extends **[001]**) | `ACTIVE → ERASING` only; any other source status refused (`NOT_ELIGIBLE`); calls `bumpSecurityVersion()` **[004]** in the same mutation. No outgoing transition from `ERASING` exists in the product. | RF-7.2, RF-7.6, RF-7.7 |
| | `ErasureRequest` | Aggregate: identity id, `receivedAt`, proofs passed (`STEP_UP`, `EMAIL_CODE` with their challenge ids — never a secret), handoff state `RECEIVED → HANDED_OFF`, correlation id. At most one per identity. | RF-7.2, RF-7.3, RF-10.2 |
| | `ErasureGateRule` | Pure: `(acknowledgements[2], typedPhrase, expectedPhrase, blockingTenants) → Open \| Closed(gates[])` with gates `ACK_LEAVE_VS_ERASE`, `ACK_OWNERSHIP_AND_RETENTION`, `PHRASE`, `LAST_OWNER(tenantId, ACTIVE \| SUSPENDED)`. Phrase match: exact, case-sensitive, after trimming leading and trailing white space (D-d). | RF-5.4, RF-5.5, RF-5.7, RF-6.1, D-d |
| | `ErasureCodeChallenge` | Same shape as `ContinuationChallenge` **[005]**: identity id, purpose fixed `ERASURE`, step-up challenge id consumed, code hash, `expiresAt`, `attemptsRemaining`, `resendsRemaining`, state `OPEN → VERIFIED \| VOIDED \| SUPERSEDED`; `verify`, `recordFailedAttempt`, `void`, `supersede` (on resend). Consumed when it verifies (D5). | RF-6.3, RF-6.4, RF-6.5 |
| | `ErasurePolicy` | Validated configuration: code lifetime, code length, attempt limit, resend limit and resend window, ownership-checks page size, erasure process contact (channel and instructions text). No default (P13). | RF-2.2, RF-6.3, RF-6.4, RF-8.3, RNF-9 |
| | `ErasurePhraseCatalog` | The typed phrase per interface language, loaded from the same message keys as the FE bundle (`erasure.confirmPhrase`); startup fails if a supported language lacks it. | RF-5.2, RF-11.3, D-d |
| `identity.application` | `EvaluateErasureGatesQuery` | Read-only: the person's acknowledgements and phrase against `ErasurePhraseCatalog` for the request locale, plus `BlockingTenantsPort.blockingTenants(identityId)` (→ `access`, no lock). Answers `Open` or the named gates. Used by "Continue to identity verification". | RF-5.4, RF-5.5, RF-6.1 |
| | `StartErasureVerificationUseCase` | After the step-up: (1) idempotency outcome **[001 D1]**; (2) re-evaluate the gates; (3) `StepUpPort.consume(challengeId, identityId, purpose = ERASURE)` **[010]**; (4) void any open `ErasureCodeChallenge` of the identity; create a new one; (5) send the code through `TransactionalEmailPort` **[002]** synchronously with the template `erasure-one-time-code` and commit only if the send is accepted (D11); audit `ERASURE_CODE_SENT`. | RF-6.1, RF-6.2, RF-6.3, RF-6.4, RNF-12 |
| | `ResendErasureCodeUseCase` | Within `resendsRemaining` and the resend window: `supersede` the open challenge, issue a fresh code, same send rule; otherwise `identity.erasure.resend-limit`. | RF-6.3, RF-6.4, LC-25 |
| | `RecordErasureRequestUseCase` | One transaction (D4): (1) idempotency outcome; (2) `ErasureCodeChallenge.verify` — wrong / expired / exhausted → one generic problem, counted on `AbuseThrottlePolicy` **[001]** keys, challenge voided when exhausted; (3) `BlockingTenantsPort.lockAndCheck(identityId)` (→ `access`: takes every relevant `TenantOwnership` lock in ascending tenant-id order, D6) — any blocking tenant → `identity.erasure.blocked{tenantId}`, nothing written; (4) `GlobalIdentity.beginErasure` (status `ERASING`, security version advanced); (5) insert `ErasureRequest` (unique identity id); (6) `RevokeSessionsPort.revokeAll(identityId, except = none, cause = IDENTITY_STATUS)` **[004/006]**; (7) `RecoveryCaseClosurePort.close(identityId, IDENTITY_ERASING)` **[005]**; (8) audit `ERASURE_REQUEST_RECORDED{proofs}`; (9) enqueue `IdentityErasureRequested{identityId, requestId}` and the `erasure-request-received` message. An identity already `ERASING` → the recorded outcome or `identity.erasure.already-requested`, nothing written. | RF-6.5, RF-7.1 … RF-7.5, RF-8.3, RNF-3, RNF-4, RNF-5 |
| | `ErasureHandoffRelay` | Consumes `IdentityErasureRequested` and calls `ErasureHandoffPort.handOff(request)`; on success marks `HANDED_OFF`; retried by the relay; never changes identity status. | RF-7.5, RF-7.7 |
| | Ports | Published: `IdentityErasureRequested` (outbox event), `ErasureStatusPort.isErasing(identityId)` (read per call). Consumed: `BlockingTenantsPort` (→ `access`, §2.1.2), `StepUpPort` **[010]**, `RevokeSessionsPort` **[004]**, `RecoveryCaseClosurePort` **[005]**, `ErasureHandoffPort` (placeholder adapter, D7), `TransactionalEmailPort` **[002]**, `SecurityNotificationPort` **[001]**, `AuditPort` **[001]**, `Clock` **[001]**, `IdentityRepository` **[001]**. | all of RF-6, RF-7 |
| `identity.api` | `ErasureController` | `POST /api/account/erasure/gates` {acknowledgements, typedPhrase} → 200 `{open: true}` \| 200 `{open: false, gates[]}`; `POST /api/account/erasure/verification` {acknowledgements, typedPhrase, stepUpChallengeId, idempotencyKey} → 201 `{codeChallengeId, sentTo (masked verified address), expiresAt}` \| 409 `identity.erasure.blocked{gates}` \| 428 `mfa.step-up.required` \| 503 `identity.erasure.code-unsent`; `POST /api/account/erasure/verification/{id}/resend`; `POST /api/account/erasure` {codeChallengeId, code, idempotencyKey} → 202 `{state: RECEIVED}` \| 400 `identity.erasure.verification-failed` (generic) \| 409 `identity.erasure.blocked{tenantId}` \| 409 `identity.erasure.already-requested`. Every endpoint requires an asserted session of an `ACTIVE` identity. | RF-1.1, RF-5.4, RF-6, RF-7, RF-10.3 |
| | Step-up purpose (extends **[003]**) | `StepUpChallenge.purpose` gains `ERASURE` and `LEAVE_TENANT`; limits from `EnrollmentPolicy` **[003]**; a recovery code is refused as for every purpose (003 D6). | RF-4.3, RF-6.2, LC-23 |
| `identity.infrastructure` | Persistence | Table `identity_erasure_request` (identity id unique, received_at, proofs, handoff_state, correlation id); table `erasure_code_challenge` (code hash, purpose, expires_at, attempts_remaining, resends_remaining, state, step_up_challenge_id). Identity rows, no `tenant_id`; not granted to the tenant runtime role. Persistence access API per A-35. | RF-7.2, RF-7.3, RF-6.4, RNF-8 |
| | Templates | `erasure-one-time-code`; `erasure-request-received` (received, signed out everywhere, no time promised, minimised retention, process contact from `ErasurePolicy`, no acting link). EN / ES / FR by `IdentityLocalePort` **[014]**. | RF-6.3, RF-8.3, RF-8.5 |
| | Audit adapter | `ERASURE_GATES_REFUSED{gates}`, `ERASURE_CODE_SENT`, `ERASURE_CODE_FAILED`, `ERASURE_CODE_VOIDED`, `ERASURE_BLOCKED_AT_RECORDING{tenantId}`, `ERASURE_REQUEST_RECORDED{proofs}`, `ERASURE_HANDED_OFF`; `STEP_UP_CONSUMED{purpose}` **[010]**. Each with identity id, time, correlation id; never a code, password or phrase. | RF-10.2, RNF-11 |

#### 2.1.2 Bounded context `com.aspectra.access` [009, 010] — self-service leave, ownership checks, erasure fan-out

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `access.domain` | `LeaveRule` | Pure: `(role, membershipState, tenantStatus, activeOwnersBefore) → Permit(stepUpRequired) \| Deny(LAST_OWNER \| TENANT_SUSPENDED_OWNER \| TENANT_DELETING \| NOT_LISTED)`: active tenant + not last active Owner → permit (step-up when Owner); suspended tenant + Member or Admin → permit; suspended tenant + Owner → deny (D-a); deleting → deny (D-b); revoked → `NOT_LISTED`. Uses `ActiveOwnerRule` **[010]**. | RF-2.4, RF-3.2, RF-4.1, RF-4.3, D-a, D-b |
| | `LastOwnerGuard` (extends **[010]**) | New change kind `LEAVE_OWNER`, evaluated under the lock exactly like `REMOVE_OWNER`. | RF-4.4, RNF-2 |
| | `OwnershipCheckKind` | Pure: per listed membership → `LAST_OWNER_ACTIVE` ("You are the last Owner · Transfer incomplete" / "Transfer to a verified eligible Owner"), `LAST_OWNER_SUSPENDED` ("… · Tenant suspended" / "Held until the tenant is reinstated or deleted"), `CO_OWNED` ("Another active Owner remains"), `NO_OWNER_RESPONSIBILITY`; plus the row action from `LeaveRule` and, for `LAST_OWNER_ACTIVE`, `TRANSFER` or `NO_ELIGIBLE_SUCCESSOR` (from `SuccessorsQuery` **[010]**, a count only). | RF-2.3, RF-2.4, RF-2.5, RF-3.2 |
| `access.application` | `OwnershipChecksQuery` | Identity-scoped read through `MembershipProjectionPort` **[001/013]** (A-36 predicate): listed memberships (active or suspended; revoked excluded) with tenant name, own role, membership state, tenant status (`TenantStatusPort.statusOf` **[014]**), active-owner count; page by `ErasurePolicy` page size; total count; `currentTenant` card data when a tenant id is requested and the membership is listed. Exposes nothing else (contract guard as plan 014). | RF-2.1, RF-2.2, RF-2.3, RF-2.6, RF-2.8, RF-3.1, RF-3.4 |
| | `LeaveTenantUseCase` | Identity-scoped (014 D13), one transaction: (1) idempotency outcome **[001 D1]**; (2) load the person's own membership by id and identity (`MembershipRepository` **[009]**) — absent / revoked → `access.leave.not-listed` (RF-4.11); (3) `PermissionVersionGuard` **[009]** against the version the dialog showed — moved → `access.leave.changed`; (4) read tenant status `FOR SHARE`; (5) if Owner: `StepUpPort.consume(challengeId, identityId, LEAVE_TENANT)` **[010]**, then `TenantOwnershipRepository.lockForUpdate` and `activeOwnerCount` **[010]**; (6) `LeaveRule` + `LastOwnerGuard(LEAVE_OWNER)`; deny → stated reason, nothing written; (7) `MembershipCommands.LeaveOwnMembership` (D2): `TenantMembership.revoke` **[009]** (grants revoked, version advanced once), audit `MEMBERSHIP_LEFT`, enqueue `AuthorityChanged{kind = MEMBERSHIP_REVOKED}` **[009]** and `MembershipLeft{tenantId, membershipId, identityId}`. Sessions untouched. | RF-4.1, RF-4.3 … RF-4.8, RF-4.10, RF-4.11, RNF-2, RNF-3, RNF-4 |
| | `MembershipLeftHandler` | Consumes `MembershipLeft`: `tenant-left` to the person; `member-left` to each recipient of `TenantAdministratorsQuery` **[014]** except the person (display name from `IdentityFactsPort` **[008]**, no link, no content). | RF-8.1, RF-8.2 |
| | `BlockingTenantsService` (implements `BlockingTenantsPort` for `identity`) | `blockingTenants(identityId)`: tenants (active or suspended) where the person holds the only active Owner membership — read without lock. `lockAndCheck(identityId)`: inside the caller's transaction, lock `TenantOwnership` of every tenant where the person holds an active Owner membership in ascending tenant-id order, recount under the locks, return the blocking set (D6). Deleting tenants, suspended memberships and non-Owner memberships are never in the set. | RF-5.4, RF-5.5, RF-7.1, RNF-2, LC-20 |
| | `IdentityErasureHandler` | Consumes `IdentityErasureRequested`: for every membership of the identity in any state but revoked, enqueue `AuthorityChanged{tenantId, membershipId, kind = IDENTITY_ERASING}` so `AuthorityChangedHandler` **[009]** closes subscriptions and calls `WorkRevocationPort.refuseQueued` / `requestBoundedCancel` per tenant; changes no membership, grant or invitation row (Q7). Idempotent per request id. | RF-7.8, RF-7.9 |
| | Ports | Published: `BlockingTenantsPort`, `MembershipLeft` (outbox event). Consumed: `IdentityErasureRequested` (from `identity`), `TenantStatusPort` **[014]**, `TenantAdministratorsQuery` **[014]**, `IdentityFactsPort` **[008]**, `StepUpPort` **[010]**, `SecurityNotificationPort` **[001]**, `AuditPort` **[001]**. | all of RF-2, RF-4, RF-8.1–8.2 |
| `access.api` | `LeaveController` and `OwnershipChecksController` | `GET /api/me/ownership-checks?page&currentTenantId` → the read model; `POST /api/me/memberships/{membershipId}/leave` {expectedVersion, stepUpChallengeId?, idempotencyKey} → 200 `{left: true, wasCurrentTenant}` \| 404 `access.leave.not-listed` \| 409 `access.last-owner` **[010]** \| 409 `access.leave.tenant-suspended` \| 409 `access.leave.changed` \| 428 `access.step-up.required` **[010]**. Identity-scoped: the path carries no tenant as authority (FR-X-002). | RF-2, RF-4, RF-10.3 |
| `access.infrastructure` | Persistence and audit | No new table; `tenant_membership` and `project_grant` **[009]**, `tenant_ownership` **[010]**. Audit `MEMBERSHIP_LEFT{tenantId, role, previous, new}`, `LEAVE_REFUSED{reason}`. Templates `tenant-left`, `member-left`. | RF-4.5, RF-8.1, RF-8.2, RF-10.1 |

#### 2.1.3 Existing elements this plan relies on unchanged

| Element | Owner | Behaviour relied on | Covers |
|---|---|---|---|
| `SignInUseCase` generic branch | **[001]** | `ERASING` → generic failure, throttle counted | RF-7.6, LC-29 |
| `isResetEligible()` | **[004]** | `ERASING` → nothing created or sent | RF-7.6 |
| Recovery request rule | **[005]** | `ERASING` → nothing created or sent | RF-7.6 |
| `AssertSessionUseCase` / `RefreshSessionUseCase` | **[006]** | old security version or `ERASING` → session ends at next use, cause `IDENTITY_STATUS` | RF-7.4, RNF-5, LC-28 |
| `StartEnrollmentUseCase`, workspace eligibility | **[003]**, **[012]** | `ERASING` → not started | RF-7.6 |
| `AcceptanceRule` | **[008]** | `ERASING` → `UNUSABLE(IDENTITY_INELIGIBLE)`; the invitation stays `PENDING`, so invitations the person issued are untouched | RF-7.9, LC-31, LC-32 |
| `AuthorityChangedHandler`, `WorkRevocationPort`, `SubscriptionRevocationPort` | **[009]** | NOT EXECUTED for queued tickets, bounded cancellation, streams closed, evidence kept | RF-4.7, RF-7.9, LC-13 |

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login, Signup, Enrollment, Security settings), reusing `src/features/workspace-members` **[010]** through its public exports, `shared/*`

| Element | Responsibility | Covers |
|---|---|---|
| Route `/account/privacy` → `AccountPrivacyPage` (N19) | Opens with or without a current tenant (`currentTenantId` passed as requested scope only); entries: the Privacy slot of `SecuritySettingsPage` **[003]** and the R12 "Account privacy" tab / "Account privacy options" (contract with the personal-settings owner, §6). Focus to the heading on open. | RF-1.1, RF-1.2, RF-1.5, RF-11.2 |
| `PrivacyStatements` | The heading, the two-operations statement, "Minimal history, not forever PII", "Recovery is separate", "No instant erasure …", the isolation statement — verbatim from N19 via `shared/i18n`. | RF-1.5, RF-2.7, RF-9.3, RF-8.5 |
| `useOwnershipChecks(page, currentTenantId)` | TanStack Query on `GET /api/me/ownership-checks`; refetched after a leave, a transfer elsewhere (window focus) and before "Continue". | RF-2.1, RF-2.2, LC-19 |
| `CurrentTenantLeaveCard` | Renders the server's current-tenant card: effect, "Blocked · last Owner" with "Review ownership transfer", or "Leave <tenant>"; hidden without a current tenant. | RF-3.1 … RF-3.4 |
| `OwnershipChecksTable` | Columns YOUR MEMBERSHIP / OWNERSHIP CHECK / REQUIRED NEXT STEP plus the row action, rendered as the server states; "Previous / Next" and the record count; records at narrow widths. "Review ownership transfer" opens `TransferOwnershipDialog` **[010]** for the row's tenant; `NO_ELIGIBLE_SUCCESSOR` shows the invite-and-enroll statement. | RF-2.1 … RF-2.5, RF-11.1 |
| `LeaveTenantDialog` | On the N23 anatomy **[010]**: heading, the five effect statements, "Leave tenant" / "Cancel"; for an Owner, `StepUpFields` **[010]** with `purpose = LEAVE_TENANT` (D13) before confirm; renders refusals with their reason; repeated submit uses the same idempotency key. | RF-4.2, RF-4.3, RF-4.10, RF-4.11, S-63 |
| `useLeaveTenant()` | Mutation; on success with `wasCurrentTenant` → the guarded switch to `/workspaces` (D12); otherwise invalidates `useOwnershipChecks` and shows the confirmation. | RF-4.9 |
| `ErasureCard` and `ErasureRequestPanel` | Effect, marker, the Admin statement; identity to verify (own address from the session); two unticked acknowledgements; phrase field with the localized label; "Keep my account" (clears); "Continue to identity verification" enabled only when the local match and the server gate (`POST /gates`) agree; blocked and held statements naming the gates. State lives in component state only and resets on mount and after any failure (D14). | RF-5.1 … RF-5.7, RF-2.8, D-c, D-d |
| `ErasureVerification` | Step 1: `StepUpFields` with `purpose = ERASURE`; step 2: code entry (sent-to masked address, expiry, "Resend code"); one `GenericFailureMessage` **[001]** for any failure, then back to the cleared panel; step 3: request-received state, then `SignOutCoordinator.fromEndedSession()` **[007]** with entry `N19`. | RF-6.1 … RF-6.5, RF-7.4 |
| `contracts/identity-erasure`, `contracts/access-leave` | Typed request / response shapes; a contract guard rejecting unexpected fields in the ownership-checks response. | RF-2.6, RNF-7 |
| `shared/i18n` bundles | EN / ES / FR for every statement, the dialog, the steps and `erasure.confirmPhrase` (the same key the BE catalog reads). | RF-11.3, D-d |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `AssertSessionUseCase` **[006]**; every endpoint refuses a non-`ACTIVE` identity | route guard → sign-in or 006 handling |
| RF-1.2 | — | `/account/privacy` from the `SecuritySettingsPage` Privacy slot **[003]** and the R12 tab (contract) |
| RF-1.3 | — | R12 "Your membership" statement (personal-settings owner, contract §6) |
| RF-1.4 | no leave or erasure endpoint outside `LeaveController` / `ErasureController` | no action in `TenantChooserPage` **[013]**, `workspace-members` **[010]** or tenant screens |
| RF-1.5 | — | `PrivacyStatements`, `CurrentTenantLeaveCard`, `ErasureCard` |
| RF-2.1 | `OwnershipChecksQuery` (A-36 predicate, revoked excluded) | `OwnershipChecksTable` |
| RF-2.2 | page size from `ErasurePolicy`; total count | Previous / Next |
| RF-2.3 | `OwnershipCheckKind` from status and owner count | renders the server's texts |
| RF-2.4 | `LeaveRule` → row action or reason | row action |
| RF-2.5 | `SuccessorsQuery` **[010]** count → `TRANSFER` \| `NO_ELIGIBLE_SUCCESSOR` | `TransferOwnershipDialog` **[010]** or statement |
| RF-2.6 | read model fields fixed; contract guard | contract guard |
| RF-2.7 | — | `PrivacyStatements` |
| RF-2.8 | empty listing | erasure section only |
| RF-3.1 | `currentTenant` card data | `CurrentTenantLeaveCard` |
| RF-3.2 | `LeaveRule` `LAST_OWNER` | blocked variant + transfer route |
| RF-3.3 | `LeaveRule` permit | "Leave <tenant>" → `LeaveTenantDialog` |
| RF-3.4 | no card without a requested current tenant | card hidden |
| RF-4.1 | `LeaveRule` | — |
| RF-4.2 | — | `LeaveTenantDialog` statements |
| RF-4.3 | `StepUpPort.consume(LEAVE_TENANT)` for Owners | `StepUpFields` purpose `LEAVE_TENANT` |
| RF-4.4 | `LeaveTenantUseCase` steps 2–6: listed, version, status `FOR SHARE`, ownership lock, `LastOwnerGuard(LEAVE_OWNER)` | refusal rendered, row refreshed |
| RF-4.5 | `TenantMembership.revoke` **[009]** via `LeaveOwnMembership` | — |
| RF-4.6 | no identity, session or other-membership port called | — |
| RF-4.7 | `AuthorityChanged{MEMBERSHIP_REVOKED}` → `AuthorityChangedHandler` **[009]** | — |
| RF-4.8 | no `RevokeSessionsPort` call | other tabs meet `DeniedScopeHandler` **[006]** |
| RF-4.9 | response `wasCurrentTenant` | `useLeaveTenant` → guarded switch → `/workspaces`, or confirmation in place |
| RF-4.10 | nothing written before step 7; step-up void → `mfa.step-up.void` **[003]** | Cancel / close change nothing |
| RF-4.11 | idempotency outcome; `access.leave.not-listed` | same key on resubmit |
| RF-5.1 | — | `ErasureCard` |
| RF-5.2 | `ErasurePhraseCatalog` | `ErasureRequestPanel` |
| RF-5.3 | nothing stored server-side before verification | component state reset on mount (D14) |
| RF-5.4 | `ErasureGateRule`, `EvaluateErasureGatesQuery` | Continue disabled; gates named |
| RF-5.5 | `LAST_OWNER(tenant, SUSPENDED)` gate | held statement |
| RF-5.6 | — | "Keep my account" clears |
| RF-5.7 | gates + both proofs required in `Start…` and `Record…` | no single-action path |
| RF-6.1 | `StartErasureVerificationUseCase` step 2 | gate refusal → panel |
| RF-6.2 | `StepUpPort.consume(ERASURE)` | `StepUpFields` purpose `ERASURE` |
| RF-6.3 | `ErasureCodeChallenge` created, code sent before commit; `ResendErasureCodeUseCase` | code step with address, expiry, resend |
| RF-6.4 | purpose fixed `ERASURE`, supersede on resend, consumed on verify | — |
| RF-6.5 | generic `identity.erasure.verification-failed`; throttle keys **[001]** | `GenericFailureMessage`, panel cleared |
| RF-6.6 | hashes only; no secret in audit | nothing in storage |
| RF-7.1 | `BlockingTenantsPort.lockAndCheck` | refusal names the tenant |
| RF-7.2 | `beginErasure` + `ErasureRequest` + security version in one transaction | — |
| RF-7.3 | unique identity id; already-`ERASING` branch | — |
| RF-7.4 | `RevokeSessionsPort.revokeAll(except = none)` **[004/006]** | received state, then `SignOutCoordinator.fromEndedSession()` |
| RF-7.5 | `RecoveryCaseClosurePort.close(IDENTITY_ERASING)` **[005]**; `ErasureHandoffRelay` | — |
| RF-7.6 | §2.1.3 rows for 001, 004, 005, 003, 012, 008 | — |
| RF-7.7 | no withdraw endpoint; `ERASING` has no product exit | no control |
| RF-7.8 | session assertion refuses → derivation yields nothing **[009]**; `ActiveOwnerRule` **[010]** excludes `ERASING` | — |
| RF-7.9 | `IdentityErasureHandler` → `AuthorityChangedHandler` **[009]**; `AcceptanceRule` **[008]** | — |
| RF-8.1 | `MembershipLeftHandler` → `tenant-left` | — |
| RF-8.2 | `MembershipLeftHandler` → `member-left` to `TenantAdministratorsQuery` **[014]** | — |
| RF-8.3 | `erasure-request-received` with `ErasurePolicy` contact | — |
| RF-8.4 | no tenant recipient for erasure events | — |
| RF-8.5 | template copy review; no time placeholder | — |
| RF-9.1 | leave only on the person's own membership by identity; erasure only for the session's identity; no admin endpoint | — |
| RF-9.2 | commands touch only the person's rows | — |
| RF-9.3 | — | `PrivacyStatements` |
| RF-9.4 | no run, approval, revision or evidence repository called | — |
| RF-10.1 | `MEMBERSHIP_LEFT`, `LEAVE_REFUSED` | — |
| RF-10.2 | identity audit events of §2.1.1 | — |
| RF-10.3 | safe problem codes + correlation id | problems rendered by code |
| RF-11.1 | — | one page component, table → records |
| RF-11.2 | — | labels, associated errors, text states |
| RF-11.3 | templates in three languages; phrase catalog | i18n bundles |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · Erasure lives in `identity`; leaving lives in `access`; they share no command.**
*Why.* FR-IDN-120 requires two operations, and P11 assigns each record to its owner: the identity status and security version belong to `identity` (plans 001, 004), memberships and the ownership lock to `access` (plans 009, 010). The only crossing is through published ports (`BlockingTenantsPort`, `IdentityErasureRequested`).
*Discarded.* One "account lifecycle" service over both — couples the two operations the spec separates and gives one context another's tables. Erasure implemented as "leave every tenant, then delete" — contradicts Q7 (memberships stay inert for the process) and would revoke memberships the process must settle.
*Covers.* RF-1.4, RF-1.5, RF-4.6, RF-7.8, RF-9.2.

**D2 · Leaving is a new self-service command `MembershipCommands.LeaveOwnMembership` beside plan 009's `RevokeMembership`, reusing the aggregate's `revoke`, not plan 010's `ManageMembershipUseCase`.**
*Why.* Plan 010 forbids acting on one's own row, and `RevokeMembership` derives the actor's tenant authority through `AdministrationRule`, under which a Member changes nothing; plan 014 D13 requires the leave to be identity-scoped and to work in a suspended tenant where no tenant derivation succeeds. The aggregate's `revoke` already gives the spec's effect (grants revoked, one version bump, evidence untouched), and the existing `AuthorityChanged` event already drives work revocation.
*Discarded.* Calling `ManageMembershipUseCase(REMOVE)` with actor = target — breaks 010's self-row rule and its Admin-scope rules. Adding a "self" exception to `AdministrationRule` — moves a personal right into the tenant administration matrix. A separate "left" membership state — FR-IDN-093 fixes the states; a left membership is revoked.
*Covers.* RF-4.1, RF-4.5, RF-4.6, RF-4.7, RF-9.1, D-a.

**D3 · Erasure verification is a server-held chain of two purpose-bound challenges; nothing durable is written until the second verifies.**
*Why.* Q1 requires the step-up and then the email code, both independent of the session. Each is already a proven pattern (`StepUpChallenge` **[003]**, `ContinuationChallenge` **[005]**); chaining them by consuming the step-up when the code is issued binds the code to a fresh proof, and writing nothing before the code verifies gives D-c ("nothing recorded") for every failure at any step.
*Discarded.* Step-up only — Q1. A confirmation link in the email — a mailbox alone would record the request and the link could act on the account (RF-8.3 forbids acting links). Recording a "pending" request at the step-up — leaves a record when the person abandons (D-c). A client-held token carrying the step-up result — the browser would carry authority (P3).
*Covers.* RF-5.7, RF-6.1 … RF-6.5, D-c, LC-23, LC-24.

**D4 · Recording is one transaction: the identity status, the security version, the request record, session revocation, recovery-case closure, audit and the outbox enqueue commit together; downstream effects are consumers of `IdentityErasureRequested`.**
*Why.* Q2 makes recording one indivisible change, and RNF-5 requires that no session acts afterwards. Advancing the security version inside the same transaction as the status means any session, already open or concurrent, ends at its next use (plan 006's lazy check); calling `revokeAll` and the case closure in the same transaction follows plan 007's "sign out everywhere" and plan 005's synchronous closure expectation. Work revocation across many tenants and the handoff are slow and retryable, so they run from the outbox, each consumer idempotent per request id.
*Discarded.* Everything synchronous, including work revocation in every tenant — one slow tenant would hold the identity locks and roll back the request. Everything asynchronous, including the status — a window in which the identity is still `ACTIVE` after "request received". A separate erasure flag beside the status — two sources of one truth; plans 001–008 already read `ERASING`.
*Covers.* RF-7.2, RF-7.4, RF-7.5, RF-7.9, RNF-3, RNF-5, LC-28, LC-30.

**D5 · The email code is a new `ErasureCodeChallenge` of the same shape as plan 005's `ContinuationChallenge`, with its own configuration, rather than a purpose added to the recovery record.**
*Why.* A-48 requires the same kind of code, and RF-6.4 requires it to be bound to the purpose "erasure". `ContinuationChallenge` is bound to a recovery case (and has a decoy variant with no case) and has no purpose field; reusing its table would put erasure codes in recovery's lifecycle and in its decoy rules. The same shape keeps the security properties (hashed, expiring, bounded attempts, consumed when it verifies) without coupling.
*Discarded.* A purpose column on `recovery_continuation` — changes plan 005's record and its case foreign key for another feature. A generic code service extracted now — a refactor of plan 005 not required by any criterion; listed as a possible later consolidation (§6, cross-plan item 3). Sharing `RecoveryPolicy` — the lifetimes and resend rules are separately configured (RNF-9).
*Covers.* RF-6.3, RF-6.4, RNF-9, LC-24, LC-25.

**D6 · The multi-tenant last-Owner recheck locks every tenant where the person holds an active Owner membership, in ascending tenant-id order, inside the recording transaction.**
*Why.* RF-7.1 and RNF-2 require that no interleaving of a transfer, a demotion, a co-Owner's leave or removal and the recording leaves a tenant without an active Owner. Plan 010 serialises every Owner-affecting change of one tenant under `TenantOwnership`; taking the same locks makes the recording one more Owner-affecting change. A single global order prevents deadlock with another multi-tenant operation, and single-tenant changes take only one lock.
*Discarded.* Checking without locks (count then write) — forbidden by FR-IDN-095 and plan 010 D1. `SERIALIZABLE` isolation — retries everywhere for a rare operation. Locking every membership's tenant, including non-Owner ones — needless contention; only Owner memberships can block.
*Covers.* RF-5.4, RF-7.1, RNF-2, LC-5, LC-20.

**D7 · The handoff to the erasure process is an outbound port with a placeholder adapter until A-64 decides the mechanism.**
*Why.* Q1 hands the request to an internal, audited, out-of-band process that is not designed; plan 014 D2 and plans 004/005 implement platform acts the same way. The port lets the request, its state and its audit be built and tested now; the adapter records the request for the platform and marks it `HANDED_OFF`.
*Discarded.* An administrative HTTP endpoint for the process — an operator surface in the product. Executing the erasure here — out of scope (spec §6) and outside the constitution's binding of FR-IDN-122. No handoff record — the request would be unobservable to the process.
*Covers.* RF-7.5, RF-7.7.

**D8 · The ownership checks are a server-built read model; the gates always consider every membership, not the visible page.**
*Why.* FR-IDN-123 and RNF-7: the page may show only the person's own rows, and the decision whether erasure is blocked must not depend on paging. The read model reuses the identity-scoped projection of A-36 and computes kind and action on the server; `BlockingTenantsService` runs over the full membership set.
*Discarded.* Sending raw memberships and computing checks in the client — the client would decide (P3). Gating erasure on the loaded page — LC-35. Including other members' names in "Another active Owner remains" — FR-IDN-123.
*Covers.* RF-2.1 … RF-2.6, RF-5.4, RNF-1, RNF-7, LC-35.

**D9 · The typed phrase is matched on the server against a catalog read from the same message key as the client bundle.**
*Why.* D-d: the phrase is the one displayed in the person's language, matched exactly after trimming. A server match prevents a scripted client from skipping the gate (RF-5.7); reading the same key keeps the two in step, and startup fails if a language lacks it.
*Discarded.* Client-only matching — P3. A language-independent phrase — D-d and P14 (the page is localized). Case-insensitive or fuzzy matching — D-d.
*Covers.* RF-5.2, RF-5.4, D-d, LC-21, LC-22.

**D10 · Idempotency through the outcome record of plan 001 D1 for every command, plus a unique identity key on the erasure request.**
*Why.* RF-4.11, RF-7.3 and RNF-4: a double click or a retry after a lost response must have one effect. The outcome record returns the first answer for the same key; the unique key makes a second request impossible even with a new key, and the `ERASING` status makes `beginErasure` refuse.
*Discarded.* Client-side de-duplication — not authority. A lock on the identity without a unique key — a retry after commit could still insert.
*Covers.* RF-4.11, RF-7.3, RNF-4, LC-12, LC-27.

**D11 · The erasure code is sent synchronously before its challenge commits; the leave and erasure confirmations go through the outbox.**
*Why.* RNF-12: a request is recorded only when the code could be sent, and the person must not be told "a code was sent" when it was not. The code is the gate itself, so its send is part of the step; the confirmations are notifications after a committed fact and follow plan 002's outbox rule (a relay failure is retried and never undoes the fact).
*Discarded.* Code through the outbox — "sent" could be shown for a message that never leaves. Confirmations sent synchronously — a mail outage would roll back a leave or a recorded request.
*Covers.* RF-6.3, RF-8.1 … RF-8.3, RNF-12, LC-26.

**D12 · After leaving the current tenant, the client lands on `/workspaces` through the guarded switch.**
*Why.* Q3 and specification 016: leaving the current tenant ends its scope; the switch's drain (abort requests, close streams, advance the epoch, clear the scoped cache) must run so nothing of the left tenant stays on screen. Plan 016 is not written; until it is, the client uses the fallback plan 012 already uses — the scope-epoch sequence of FR-IDN-130 in `shared/security` — and then navigates.
*Discarded.* A plain navigation — leaves cached tenant data and open streams. A full page reload — loses the session's in-memory credential handling of plan 006.
*Covers.* RF-4.9, RF-4.8.

**D13 · `StepUpFields` [010] is parameterised by purpose; the step-up gains `LEAVE_TENANT` and `ERASURE`.**
*Why.* Q1 and Q3 reuse the same step-up with their own purposes, so that a challenge succeeded for a transfer cannot be consumed by a leave or an erasure (and vice versa). Plan 010's component is hard-wired to `TENANT_OWNERSHIP`; a purpose property is the smallest change.
*Discarded.* A copy of the component per purpose — duplicate security UI. One shared purpose for every Owner-level act — a challenge could be replayed across operations.
*Covers.* RF-4.3, RF-6.2, LC-7, LC-23.

**D14 · The erasure panel's inputs live only in component state; nothing is persisted, prefetched or restored.**
*Why.* RF-5.3 and D-c: the acknowledgements and the phrase start empty every time and are cleared after any failure; FR-X-034 forbids storage. Zustand owns only deliberate unsaved editor forks (P11), which this is not.
*Discarded.* Zustand or browser storage to survive navigation — RF-5.3. Server-side drafts — the server would hold a half-made decision.
*Covers.* RF-5.3, RF-6.6, D-c, LC-38.

**D15 · A leave never ends a session; an erasure ends every session.**
*Why.* Q3 keeps the person's identity and other tenants intact, and plan 009 re-derives authority at the next request, so the left tenant's screens meet the denied-scope state of plan 006. Q2 ends every session, including the requesting one, through the security version and `revokeAll(except = none)`.
*Discarded.* Ending the session on leave — contradicts Q3 and would sign the person out of other tenants. Keeping the requesting session after erasure — RF-7.4.
*Covers.* RF-4.8, RF-7.4, RNF-5.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure rules: `LeaveRule`, `OwnershipCheckKind`, `ErasureGateRule` (phrase match), `LastOwnerGuard(LEAVE_OWNER)`, `ErasureCodeChallenge` state machine, `beginErasure` transitions, `ErasurePolicy` validation. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: one transaction for leave; one for recording (status, version, record, sessions, case, audit, outbox); RLS identity predicate on the ownership checks; the request table unreadable by the tenant runtime role. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes and codes, correlation id, generic failures, absence of other tenants' or members' data, masked address. |
| Concurrency | JUnit 5 with real PostgreSQL | Co-Owner leaves at once; leave vs demotion; recording vs transfer, co-Owner leave and removal; multi-tenant lock order without deadlock; double submit. |
| Contract (consumers) | JUnit 5 with recording stubs of plans 003, 005, 006, 008, 009, 010, 014 | Each reused port is called with the stated arguments (purposes, `except = none`, `IDENTITY_ERASING`, `AuthorityChanged` kinds) and nothing else. |
| Architecture | ArchUnit (**test-only**, as plan 010) | `identity` never imports `access` internals and vice versa; no leave or erasure endpoint outside the two controllers; no admin path to either. |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Page variants, card, table, dialog and step states, reset rules, no storage, contract guard. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Focus to heading on page, dialog and steps, reading order, labelled fields and errors, 44 px targets. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

"Proves criterion" refers to the numbered criteria of completion in specification §7.

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, LC-36 | API contract, FE e2e | `privacy.noSessionToSignIn`; `privacy.restrictedOrErasingRefused` | 2 |
| RF-1.2, RF-1.3 | FE e2e | `privacy.opensFromSecuritySettingsAndR12WithOrWithoutTenant`; `r12.yourMembershipStatement` | 2 |
| RF-1.4, RF-9.1, LC-37 | Architecture, FE unit | `arch.onlyTwoControllersLeaveOrErase`; `chooserMembersTenantScreens.offerNoLeaveOrErase` | 2 |
| RF-1.5, RF-2.7, RF-9.3 | FE unit | `n19.rendersDesignedStatementsVerbatim` | 2 |
| RF-2.1, RF-2.6, RNF-7 | BE integration, API contract | `ownershipChecks.onlyOwnListedMemberships`; `ownershipChecks.revokedExcluded`; `contractGuard.rejectsExtraFields` | 2, 7 |
| RF-2.2, LC-35 | BE integration | `ownershipChecks.pagedByConfiguredSizeWithTotal` | 2 |
| RF-2.3 | BE unit | `ownershipCheckKind.lastOwnerActiveSuspendedCoOwnedNone` | 2 |
| RF-2.4, D-a, D-b, LC-9, LC-10 | BE unit | `leaveRule.rowActionOrReason`; `leaveRule.ownerSuspendedDenied`; `leaveRule.deletingNoLeave` | 3 |
| RF-2.5, LC-18 | BE integration, FE e2e | `ownershipChecks.noEligibleSuccessorStatement`; `row.reviewTransferOpensN23ForThatTenant` | 2 |
| RF-2.8, LC-34 | FE unit | `n19.noMembershipShowsErasureOnly` | 2 |
| RF-3.1 … RF-3.4, LC-3 | FE unit | `currentCard.leaveOrBlockedOrHidden` | 3 |
| RF-4.1, LC-8, LC-11 | BE integration | `leave.memberOrAdminRevokesOwnMembershipOnly`; `leave.suspendedTenantMemberPermitted`; `leave.suspendedMembershipPermitted` | 3 |
| RF-4.2 | FE unit | `leaveDialog.statesFiveEffects` | 3 |
| RF-4.3, LC-4, LC-7 | BE integration, FE unit | `leave.ownerRequiresStepUpLeaveTenant`; `leave.stepUpOtherPurposeRefused`; `leaveDialog.ownerStepUpVariant` | 3 |
| RF-4.4, LC-6 | BE integration | `leave.recheckListedVersionStatusOwners`; `leave.ownerBecameLastRefused` | 3, 4 |
| RF-4.5, RF-4.6 | BE integration | `leave.membershipRevokedGrantsEndedVersionOnce`; `leave.identitySessionsOtherMembershipsUntouched` | 3 |
| RF-4.7, LC-13 | Contract | `leave.authorityChangedRevokesWorkInTenant` | 3 |
| RF-4.8 | BE integration, FE e2e | `leave.noSessionEnded`; `otherTab.deniedScopeAfterLeave` | 3 |
| RF-4.9, LC-1, LC-2 | FE e2e | `leave.currentTenantGuardedSwitchToChooser`; `leave.otherTenantStaysOnN19` | 3 |
| RF-4.10 | BE integration, FE unit | `leave.cancelOrFailedStepUpChangesNothing` | 3 |
| RF-4.11, LC-12 | BE integration | `leave.sameKeyOneEffect`; `leave.alreadyRevokedNotListed` | 3 |
| RF-5.1, RF-5.2 | FE unit | `erasureCard.statements`; `panel.ownAddressAcksPhraseLabelLocalized` | 5 |
| RF-5.3, D-c, LC-38 | FE unit | `panel.startsEmptyOnEveryMount`; `panel.nothingInStorage` | 5 |
| RF-5.4, RF-5.7, LC-16 | BE unit, API contract | `gateRule.closedNamesGates`; `gates.serverRefusesWithoutAcksOrPhrase`; `noSingleActionPath` | 5 |
| RF-5.5, LC-17 | BE integration | `gates.suspendedSoleOwnerHeld` | 5 |
| RF-5.6 | FE unit | `panel.keepMyAccountClears` | 5 |
| RF-6.1, LC-19 | BE integration | `startVerification.reevaluatesGates`; `transferElsewhere.thenProceeds` | 5 |
| RF-6.2, LC-23 | Contract, BE integration | `startVerification.consumesStepUpErasure`; `stepUp.recoveryCodeRefused` | 5 |
| RF-6.3, RNF-12, LC-26 | BE integration | `startVerification.codeSentBeforeCommit`; `emailUnavailable.retryableNothingRecorded` | 5 |
| RF-6.4, LC-24, LC-25 | BE unit | `codeChallenge.purposeErasureOnly`; `codeChallenge.resendSupersedes`; `codeChallenge.consumedOnVerify` | 5 |
| RF-6.5 | API contract, FE e2e | `record.wrongExpiredExhaustedGenericNothingRecorded`; `failure.backToClearedPanel`; `failure.countsOnThrottle` | 5 |
| RF-6.6, RNF-8 | BE integration, FE unit | `secrets.neverInAuditLogsOrStorage` | 7 |
| RF-7.1, LC-20 | Concurrency | `record.coOwnerLeftMeanwhileRefusedNamesTenant` | 4, 5 |
| RF-7.2, RNF-3 | BE integration | `record.statusVersionRecordOneTransaction`; `record.failureLeavesNothing` | 5 |
| RF-7.3, LC-27 | BE integration | `record.oneRequestPerIdentity` | 5 |
| RF-7.4, RNF-5, LC-28 | BE integration, FE e2e | `record.allSessionsEndIncludingRequester`; `otherBrowser.endsAtNextUse`; `received.thenSignedOut` | 5 |
| RF-7.5, LC-30 | Contract | `record.closesOpenRecoveryCase`; `handoff.relayMarksHandedOff` | 5 |
| RF-7.6, LC-29 | Contract | `erasing.signInResetRecoveryEnrollmentCreationAcceptanceGeneric` | 5 |
| RF-7.7, LC-33 | Architecture, API contract | `noWithdrawEndpoint`; `erasing.noProductExit` | 5 |
| RF-7.8 | Contract | `erasing.derivationYieldsNothing`; `erasing.notActiveOwner` | 5 |
| RF-7.9, LC-31, LC-32 | Contract, BE integration | `erasure.authorityChangedPerMembership`; `erasure.membershipsGrantsUnchanged`; `erasure.invitationsToAddressUnusable`; `erasure.issuedInvitationsStayPending` | 5 |
| RF-8.1, RF-8.2 | BE integration | `leave.enqueuesTenantLeftToPerson`; `leave.enqueuesMemberLeftToOwnersAdminsExceptPerson` | 6 |
| RF-8.3, RF-8.4, RF-8.5 | BE integration | `erasure.confirmationContentAndContact`; `erasure.noTenantRecipient`; `templates.noTimePromisedNoActingLink` | 6 |
| RF-9.2, RF-9.4 | BE integration | `operations.touchOnlyOwnRows` (row snapshots of runs, approvals, revisions, evidence before and after) | 7 |
| RF-10.1, RF-10.2, RNF-11 | BE integration | `audit.leaveAndRefusals`; `audit.erasureEventsWithoutSecrets` | 7 |
| RF-10.3 | API contract | `problems.safeCodesCorrelationIdNoForeignData` | 7 |
| RF-11.1, LC-39 | FE e2e | `n19.reflowTableAsRecordsAtMobileAndTablet` | 8 |
| RF-11.2, RNF-10 | FE e2e + axe, manual | `a11y.n19DialogStepsAA`; manual NVDA / VoiceOver script | 8 |
| RF-11.3, D-d, LC-21, LC-22 | BE unit, FE e2e | `phrase.exactCaseTrimmedPerLanguage`; `languages.enEsFrComplete` | 5, 8 |
| RNF-1 | BE integration | `decisions.serverDerivedPerRequest` | 2, 3 |
| RNF-2, LC-5 | Concurrency | `coOwners.leaveTogetherOneRefused`; `lockOrder.noDeadlockAcrossTenants` | 4 |
| RNF-4 | BE integration | covered by RF-4.11 and RF-7.3 cases | 3, 5 |
| RNF-6 | BE integration | `templatesAndPage.noInstantErasureOrDuration` | 6 |
| RNF-9 | Startup | `erasurePolicy.absentOrUnboundedFailsStartup`; `phraseCatalog.missingLanguageFailsStartup` | 5 |
| LC-14 | Contract (008 stub) | `leftThenReinvited.newMembershipNoGrantRestored` | 3 |
| LC-15 | FE e2e | `erasure.happyPath` | 5 |
| US-1 … US-10 | FE e2e | journeys above | 1 |
| US-11 | adversarial cases above | one-click or session-only erasure, acting for another person, last-Owner escape, foreign data through checks, withdraw or replay, sign-in while erasing | 1 |

### 5.3 Not tested here
Ownership transfer itself and the members surface (plan 010); the step-up internals (plan 003); session assertion and refresh (plan 006); sign-out (plan 007); invitation acceptance (plan 008); the derivation and work revocation internals (plan 009, execution plans); the chooser (plan 013); the suspended-tenant screen (plan 014); the guarded switch beyond its call (plan 016); the erasure process itself (A-64); R12 personal settings beyond its links (its own plan). Their stubs return fixed answers; the consumer contracts are verified by recording stubs here and by the owning plans' own tests.

### 5.4 Exit criteria
Specification §7 items 1–8 map to the "Proves criterion" column; item 9 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-64 erasure request record and handoff (decided in principle; process undesigned) | `identity_erasure_request` record, `beginErasure`, `ErasureHandoffPort` with a placeholder adapter (D4, D7); process contact in `ErasurePolicy`. | Rebind the adapter to the decided process; the record, gates and screens are unchanged. |
| A-27 identity statuses (working model) | `ACTIVE → ERASING` by `beginErasure` only; no product exit. | Adjust the transition table; the use case is unchanged. |
| A-48 one-time email code | `ErasureCodeChallenge` with the same shape, purpose `ERASURE` (D5). | Align the record; behaviour unchanged. |
| A-30 notification set | Adds `tenant-left`, `member-left`, `erasure-request-received` (and the `erasure-one-time-code` gate message). | Adjust the set. |
| A-56 work in flight (decided) | `AuthorityChanged` per membership through plan 009 (RF-4.7, RF-7.9). | None expected. |
| A-11 configuration catalogue | `ErasurePolicy` keys: code lifetime, length, attempts, resend limit and window, page size, process contact. | Rename keys. |
| A-36 identity-scoped listing | `OwnershipChecksQuery` through plan 009 D10's identity predicate. | Replace the policy; the read model is unchanged. |
| A-35 persistence tooling | Repositories against the placeholder API. | Rename base types. |
| S-63 N19 states | Dialog, step-up variant, verification steps, received state, row variants, no-membership variant, paging, refusals — built as described. | Replace layout and copy; read models unchanged. |
| S-14 email templates | Content of RF-8.1 … RF-8.3. | Copy only. |
| S-02 responsive coverage | Reflow, table to records (RF-11.1). | Replace with the designed counterpart. |
| S-04 R12 destination | R12 "Account privacy" → `/account/privacy`. | Re-point the link. |

**Cross-plan items raised by this plan.**
1. **Plan 009:** `MembershipCommands.LeaveOwnMembership` (identity-scoped, no `AdministrationRule`, same audit, event and idempotency as `RevokeMembership`) and the `AuthorityChanged` kind `IDENTITY_ERASING` (D2, D4).
2. **Plan 010:** `LastOwnerGuard` change kind `LEAVE_OWNER`; `StepUpFields` purpose property (D13); `TenantOwnershipRepository` used with several locks in ascending tenant-id order (D6).
3. **Plan 003:** `StepUpChallenge.purpose` gains `LEAVE_TENANT` and `ERASURE` (D13). Observed naming differences between plans, reported for alignment and not changed here: the step-up success shape is `{pendingId, purpose, setup?}` in plan 003 and `{challengeId}` in plan 010; the consumed column is `consumed_by` in plan 003 and `consumed_by_pending` / `consumed_by_operation` in plan 010. This plan uses plan 010's `StepUpPort.consume(challengeId, …)`.
4. **Plan 005:** `RecoveryCaseClosurePort.close(identityId, IDENTITY_ERASING)` is now called by `RecordErasureRequestUseCase`; a later consolidation of `ContinuationChallenge` and `ErasureCodeChallenge` into one code service is possible and not required (D5).
5. **Plan 007:** `SignOutCoordinator` entry `N19` (`fromEndedSession` after the received state).
6. **Plan 014:** the D13 contracts are built here — Member or Admin leave a suspended tenant, every Owner refused with `TENANT_SUSPENDED`, erasure held while a suspended tenant needs a transfer. Observed naming difference: `TenantStatusPort.status(tenantId)` in plan 008 and `statusOf(tenantId)` in plan 014; this plan uses `statusOf`.
7. **Plan 016:** the guarded switch to `/workspaces` after leaving the current tenant (D12); the FR-IDN-130 fallback is used until it exists.
8. **Personal settings plan (F-SET-02, R12):** no plan owns R12 yet; it must render the "Your membership" statement (RF-1.3) and link the "Account privacy" tab and "Account privacy options" to `/account/privacy`.

The specification's derived rules are built, not assumed: D-a in `LeaveRule`, D-b in `LeaveRule` and `BlockingTenantsService`, D-c in `RecordErasureRequestUseCase` (nothing written before the code verifies) and `ErasureRequestPanel` (reset), D-d in `ErasureGateRule` and `ErasurePhraseCatalog`.

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.5) | `ErasureController`/`LeaveController` session gate, `/account/privacy`, `PrivacyStatements` | D1 | RF-1 rows |
| RF-2 (2.1–2.8) | `OwnershipChecksQuery`, `OwnershipCheckKind`, `LeaveRule`, `OwnershipChecksTable` | D8 | RF-2 rows |
| RF-3 (3.1–3.4) | `currentTenant` card data, `CurrentTenantLeaveCard` | D8 | RF-3 row |
| RF-4 (4.1–4.11) | `LeaveRule`, `LeaveTenantUseCase`, `LeaveOwnMembership`, `LastOwnerGuard(LEAVE_OWNER)`, `LeaveTenantDialog`, `useLeaveTenant` | D2, D6, D10, D12, D13, D15 | RF-4 rows |
| RF-5 (5.1–5.7) | `ErasureGateRule`, `EvaluateErasureGatesQuery`, `ErasurePhraseCatalog`, `ErasureRequestPanel` | D3, D8, D9, D14 | RF-5 rows |
| RF-6 (6.1–6.6) | `StartErasureVerificationUseCase`, `ResendErasureCodeUseCase`, `ErasureCodeChallenge`, `ErasureVerification` | D3, D5, D11, D13, D14 | RF-6 rows |
| RF-7 (7.1–7.9) | `RecordErasureRequestUseCase`, `beginErasure`, `ErasureRequest`, `BlockingTenantsService`, `ErasureHandoffRelay`, `IdentityErasureHandler`, §2.1.3 | D4, D6, D7, D10, D15 | RF-7 rows |
| RF-8 (8.1–8.5) | `MembershipLeftHandler`, templates | D11 | RF-8 rows |
| RF-9 (9.1–9.4) | identity-scoped commands, `PrivacyStatements` | D1, D2 | RF-9 rows |
| RF-10 (10.1–10.3) | audit adapters, problem codes | D10 | RF-10 rows |
| RF-11 (11.1–11.3) | `AccountPrivacyPage` reflow, `shared/i18n`, phrase catalog | D9 | RF-11 rows |
| RNF-1 | server read models and rules | D8, D9 | `decisions.serverDerivedPerRequest` |
| RNF-2 | ownership locks, `LastOwnerGuard` | D6 | concurrency rows |
| RNF-3 | single transactions | D4 | `record.failureLeavesNothing` |
| RNF-4 | outcome record, unique key | D10 | RF-4.11 / RF-7.3 rows |
| RNF-5 | security version + `revokeAll` | D4, D15 | `record.allSessionsEndIncludingRequester` |
| RNF-6 | templates and statements | D7, D11 | `templatesAndPage.noInstantErasureOrDuration` |
| RNF-7 | identity-scoped read model, contract guard | D8 | `ownershipChecks.onlyOwnListedMemberships` |
| RNF-8 | hashes, no storage | D3, D14 | `secrets.neverInAuditLogsOrStorage` |
| RNF-9 | `ErasurePolicy`, `EnrollmentPolicy` **[003]** | D5 | `erasurePolicy.absentOrUnboundedFailsStartup` |
| RNF-10 | `shared/ui`, breakpoints | — | a11y rows |
| RNF-11 | audit adapters | — | audit rows |
| RNF-12 | synchronous code send | D11 | `emailUnavailable.retryableNothingRecorded` |
| LC-1 … LC-39 | per the §5.2 rows that name them | — | §5.2 |

Every EARS criterion RF-1.1 … RF-11.3 appears in §3; every RNF and LC appears in §5.2 or above. No element of §2 lacks a criterion, and no criterion lacks an element, a decision and a test.
