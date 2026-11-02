# Tasks 010 — Tenant members and ownership transfer

**Feature** The tenant member list, invitations, tenant role changes, membership management and ownership transfer · **Tasks version** 0.1 · **Date** 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specTenantMembersOwnership.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n) and `planTenantMembersOwnership.md` v0.1 (modules §2, decisions D1–D18, tests §5, register and cross-plan items §6). Every task names the plan element it builds, the requirements it covers and one verifiable "Done when" line; test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order; "after" names the tasks that must be complete first (when it is not simply the previous task). Tasks marked **[001]** … **[009]** extend an element that an earlier plan builds; the element is extended, never duplicated, and if the owning feature is not built yet the element is built here in the shape its plan states — in particular plan 009's `access` core (`TenantMembership`, `AdministrationRule`, `MembershipCommands`, `AuthorityChecker`, `PermissionVersionGuard`, `AuthorityChanged`), plan 008's `Invitation` and `InvitationRepository`, and plan 003's step-up. Test-only tools: JUnit 5, MockMvc, Testcontainers, ArchUnit, Vitest, Testing Library, Playwright, axe. The persistence access API and the schema-migration tool are placeholders until A-35 is decided; "migration" below means a migration in that tool, applied after plans 001–009's.

Phases: **A** foundations · **B** domain · **C** application · **D** infrastructure · **E** API · **F** front end · **G** verification suites · **H** closing. Phase F depends only on T-01 (problem codes and endpoint shapes are fixed by the plan) and may run alongside phases B–E.

---

## A · Foundations

- [ ] **T-01 · Members configuration record** — `TenantMembersPolicy` value object and `TenantMembersPolicyProperties` binding: members page size, search minimum and maximum length; each `@Validated` as present, positive and bounded above by a configured ceiling. Invitation expiry stays in `InvitationPolicy` **[008]**, step-up expiry and attempts in `EnrollmentPolicy` **[003]**. *(plan §2.1 `TenantMembersPolicy`, D17; RNF-12)*
  **Done when:** the Spring context starts with a complete property set and fails to start, naming the property, for each value removed or set unbounded.

- [ ] **T-02 · Migration: `tenant_ownership`** — one row per tenant: `tenant_id` (primary key, composite FK to the tenant), `lock_version`, `last_owner_change_at`; backfilled for every existing tenant; `FORCE RLS` with `USING` and `WITH CHECK` on `tenant_id` **[009]**. *(plan §2.1 Persistence, D1; RF-10.2, RF-10.3, RNF-2, RNF-4)* — after T-01
  **Done when:** the migration applies on an empty database and on one with existing tenants, every existing tenant has exactly one row, a second row for a tenant is rejected, and a row is readable only under its own tenant context.

- [ ] **T-03 · Migration: `tenant_invitation` extensions [008]** — columns `resent_at`, `revoked_by`, `revoked_at`, `expired_at`, `request_locale`, and a **partial unique index** on `(tenant_id, invited_email_normalised) WHERE state = 'PENDING'`. *(plan §2.1 Persistence, D6; RF-8.4, RF-8.5, RF-9.3, RF-9.4, RF-9.5, RNF-9)* — after T-02
  **Done when:** the migration applies on a database holding plan 008 rows; a second `PENDING` invitation for the same tenant and address is rejected; a `REVOKED` row plus a new `PENDING` row for the same address is accepted.

- [ ] **T-04 · Migration: step-up purpose [003]** — `step_up_challenge` accepts the purpose value `TENANT_OWNERSHIP` and gains `consumed_by_operation` beside `consumed_by_pending`. *(plan §2.1 Persistence, D3; RF-5.1, RNF-5)* — after T-03
  **Done when:** the migration applies; a challenge row with purpose `TENANT_OWNERSHIP` can be inserted and marked consumed by an operation id; plan 003's rows are unaffected.

- [ ] **T-05 · Outbox templates registered** — register template ids `invitation`, `invitation-settled`, `ownership-received`, `ownership-transferred`, `tenant-role-changed`, `membership-suspended`, `membership-reinstated`, `membership-removed` with the outbox message type and the three locales, the D04 copy (`437:4431`) per S-14 carrying the statements of RF-13.4 (the invitation message with tenant, inviter, role, project and grants or "membership only", link, expiry and "only the local identity holding this address can accept"; the settled notification with tenant and outcome and no link). *(plan §2.1 Outbox relay and email adapter, D12; RF-13.4, RNF-14)* — after T-01
  **Done when:** `templates.eightMessagesCarryStatements` passes in EN, ES and FR with sample variables, producing a non-empty subject and body for each of the twenty-four renderings, and no template contains a completion-time statement.

## B · Domain (pure, no Spring)

- [ ] **T-06 · `ActiveOwnerRule` and `LastOwnerGuard`** — `ActiveOwnerRule(membership, identityStatus)` = role `OWNER` ∧ membership `ACTIVE` ∧ identity `ACTIVE` (D-a); `LastOwnerGuard(activeOwnersBefore, change)` with changes `DEMOTE_OWNER`, `SUSPEND_OWNER`, `REMOVE_OWNER`, `TRANSFER` → `Permit | Deny(LAST_OWNER)`; `TRANSFER` never denies. *(plan §2.1 `ActiveOwnerRule`, `LastOwnerGuard`, D1; RF-5.8, RF-10.1, RF-10.3, LC-2, LC-26)*
  **Done when:** a table test shows `Deny` for each of the three removing changes when exactly one active Owner exists, `Permit` when two exist, `Permit` for `TRANSFER` at any count, and that a suspended Owner or one whose identity is restricted or erasing is not counted.

- [ ] **T-07 · `OwnerAffectingRule`** — pure classification: promote to Owner, demote an Owner, suspend or remove an Owner, transfer → Owner-affecting; Member ↔ Admin and suspend / reinstate / remove of a Member or Admin → not. *(plan §2.1 `OwnerAffectingRule`, D5; RF-6.3, RF-6.4, RF-7.3, RNF-5)*
  **Done when:** a table test over every (target role × new role or action) pair yields exactly the classification of decision Q7.

- [ ] **T-08 · `SuccessorEligibilityRule`** — `(candidate membership, identity facts, hasActiveFactor, actorMembershipId) → eligible`: membership `ACTIVE` ∧ identity `ACTIVE` with verified email ∧ active factor ∧ candidate ≠ actor. *(plan §2.1 `SuccessorEligibilityRule`, D4; RF-4.2, RF-5.2, RF-6.3, LC-7, LC-8, LC-29)*
  **Done when:** `successorEligibility.rule` passes: each missing fact alone makes the candidate ineligible and the actor is never eligible.

- [ ] **T-09 · `Invitation` issuing side [008 → 010]** — `issue(tenantId, email, role, grantSet?, issuedBy, now, policy, secretHash, locale)` (state `PENDING`), `resend(newSecretHash, now, policy)` (new hash and expiry, `resentAt`; `PENDING` only), `revoke(by, now)`, `expire(now)`; `tenantRole` stays `ADMIN | MEMBER` by type; at most one project. *(plan §2.1 `Invitation`, D6; RF-8.1, RF-8.5, RF-9.3, RF-9.4, RF-9.5, RNF-9)* — after T-01
  **Done when:** unit tests show the four transitions, a throw for `resend`, `revoke` or `expire` on a non-pending invitation, that `OWNER` is not admitted by the type, and that plan 008's existing `Invitation` tests still pass unchanged.

- [ ] **T-10 · `InvitationIssueRule`** — `(normalisedEmail, existingMembership?, pendingInvitation?, tenantStatus, actorRole, requestedRole, requestedGrants) → Permit | Deny(reason)` with reasons `MEMBERSHIP_EXISTS`, `INVITATION_PENDING`, `TENANT_NOT_ACTIVE`, `ROLE_NOT_PERMITTED`, `SELF_ADDRESS`; a `REVOKED` membership never denies; role and grant bounds through `AdministrationRule` **[009]**. *(plan §2.1 `InvitationIssueRule`, D7; RF-8.2, RF-8.4, RF-11.3, RF-11.7, LC-19, LC-31, LC-32, LC-33, LC-34)*
  **Done when:** a table test yields each reason for its condition, `Permit` for an unknown address and for a revoked membership, and `ROLE_NOT_PERMITTED` for `OWNER` and for an Admin issuing beyond `AdministrationRule`.

- [ ] **T-11 · `TenantOwnership` aggregate and the read models** — `TenantOwnership` (tenant id, `lockVersion`, `lastOwnerChangeAt`; no business data); `MemberRow` (display name, address, `isYou`, role, status, `totpRequired`, `permissionVersion`, `actions` computed with `AdministrationRule` **[009]**), `InvitationRow` (address, role, project and grants or "membership only", issuer, issued, expires, `actions = {REVIEW}`), `MembersFilter` (search text normalised case-insensitively, role, state among `ACTIVE`, `SUSPENDED`, `INVITATION_PENDING`, `ALL`, page). *(plan §2.1 `TenantOwnership`, `MemberRow`, `InvitationRow`, `MembersFilter`, D9; RF-2.1, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-9.1; D-d)* — after T-06
  **Done when:** unit tests show the actions of RF-2.4 for every (actor role × row kind) combination, including none on an Owner row for an Admin and none on the own row except transfer for an Owner, and that serialising a `MemberRow` yields no factor, count, tenant or grant field.

## C · Application (use cases and ports)

The integration tests named in this phase run once the adapters of phase D exist (T-24…T-28); until then each use case is unit-tested with in-memory doubles.

- [ ] **T-12 · Port interfaces** — new: `TenantOwnershipRepository` (`lockForUpdate`, `activeOwnerCount` under the lock), `StepUpPort` (`consume(pendingId, identityId, purpose)` → plan 003's `StepUpUseCase`), the `InvitationSettled` outbox event (published by plan 008's use cases, consumed here); `IdentityFactsPort` **[008]** extended with `lookupByEmail(normalisedEmail) → identityId?`. Reused: `MembershipCommands`, `AuthorityPort`, `MembershipRepository`, `TenantStatusPort`, `TenantPolicyPort`, `ProjectExistsPort` **[009]**; `InvitationRepository`, `InvitationPolicy` **[008]**; `FactorStatusPort` **[003]**; `TokenGenerator`, `TokenHasher`, `TransactionalEmailPort`, `SecurityNotificationPort`, `JobScheduler` **[002]**; `SessionAssertionPort` **[006]**; `AuditPort`, `Clock` **[001]**. No session-revocation, credential, factor-mutation, approval, run or content port exists. *(plan §2.1 Ports, D1, D3, D8; enables RF-2 … RF-9; RF-11.2, RF-11.4, RF-11.6)* — after T-06…T-11
  **Done when:** the application module compiles with no dependency on Spring or persistence classes (ArchUnit module-boundary test, test-only), and the boundary test lists no port that reaches a session, credential, factor-mutation, approval, run or content table.

- [ ] **T-13 · `TenantMembersQuery`** — under the actor's C10 (`TENANT_MEMBERS` **[009]**): memberships `ACTIVE | SUSPENDED` plus pending invitations, filtered, paged, each row with its actions for this actor; display names and addresses through `IdentityFactsPort`; the Member-MFA flag through `TenantPolicyPort`; `FactorStatusPort` never read. *(plan §2.1 `TenantMembersQuery`, D9; RF-1.3, RF-1.5, RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-2.8, RF-9.1, LC-21, LC-22, LC-28)* — after T-12
  **Done when:** `membersList.rowsShapeAndNoRevokedOrClosed`, `membersList.searchFiltersAndPagination`, `membersList.emptyScopeKeepsFilters` and the back-end half of `rowActions.exactPerActorRole` pass.

- [ ] **T-14 · `SuccessorsQuery`** — Owner-only: active memberships whose identity is active and verified and has an active factor, excluding the actor; display names only plus the statement key; never the reason a member is absent. *(plan §2.1 `SuccessorsQuery`, D4; RF-4.2, RF-4.3, RF-4.4, LC-7, LC-8, LC-29)* — after T-12, T-08
  **Done when:** `successors.listsEligibleOnlyWithStatementNoReasons` and `successors.emptyWhenNoneEligible` pass.

- [ ] **T-15 · Step-up adapter and purpose consumption [003]** — `StepUpPort` implemented against plan 003's `StepUpUseCase` and `StepUpRepository`: a `SUCCEEDED` challenge of purpose `TENANT_OWNERSHIP` belonging to the actor is consumed by the operation id under the ownership lock; another purpose, another identity or an already consumed challenge refuses; the step-up itself (password + TOTP, generic failure, sign-in keys, attempts, expiry, recovery code refused) is plan 003 D5 unchanged. *(plan §2.1 Step-up adapter, D3; RF-5.1, RF-6.5, RNF-5, LC-9, LC-10)* — after T-04, T-12
  **Done when:** `stepUp.tenantOwnershipPurposeGenericCountedBoundedRecoveryRefused`, the back-end half of `stepUp.voidClosesWithNothingChanged` and `ownerAffecting.alwaysRequiresConsumedStepUp` pass.

- [ ] **T-16 · `TransferOwnershipUseCase`** — Owner-only; recorded outcome for the key; `StepUpPort.consume`; lock `TenantOwnership`; `AuthorityChecker.require` **[009]** (actor still `ACTIVE` Owner, version as submitted); successor under lock through `SuccessorEligibilityRule`; `TenantStatusPort` active; failures → `access.transfer.refused{reason}`; then `ChangeTenantRole(successor → OWNER)` and `ChangeTenantRole(actor → ADMIN)` **[009]** in one transaction; `ownership-received` and `ownership-transferred` enqueued; audit `OWNERSHIP_TRANSFERRED`; outcome recorded. *(plan §2.1 `TransferOwnershipUseCase`, D1, D2, D15; RF-4.4, RF-5.1, RF-5.2, RF-5.3, RF-5.4, RF-5.5, RF-5.6, RF-5.7, RF-5.8, RNF-2, RNF-3, RNF-6, LC-1, LC-5, LC-6, LC-11, LC-12, LC-24)* — after T-15, T-14
  **Done when:** `transfer.recheckRefusesWithStatedReason`, `transfer.successorFactorReplacedStillEligible`, `transfer.oneTransactionSuccessorFirstBothVersions`, `transfer.lastOwnerAllowed`, `transfer.touchesNoGrantSessionIdentityOrOtherTenant` and `transfer.resubmitReturnsRecordedOutcome` pass.

- [ ] **T-17 · `EditTenantRoleUseCase`** — Owner or Admin; target ≠ actor; `OwnerAffectingRule` → step-up consumption and ownership lock when Owner-affecting; `SuccessorEligibilityRule` for promotion to Owner; `LastOwnerGuard(DEMOTE_OWNER)`; `AuthorityChecker.require` with the submitted versions; `ChangeTenantRole` **[009]**; `tenant-role-changed`; audit; outcome; refusals `access.forbidden-change` **[009]**, `access.last-owner`, `access.transfer.refused{reason}`, `access.denied-scope` **[006/009]**. *(plan §2.1 `EditTenantRoleUseCase`, D5, D10; RF-6.1, RF-6.2, RF-6.3, RF-6.4, RF-6.5, RF-6.6, RF-6.7, RF-10.1, RF-10.4, RF-10.5, RF-10.6, LC-3, LC-13, LC-14, LC-15, LC-27, LC-29)* — after T-16, T-07
  **Done when:** `editRole.ownerAffectingRequiresStepUpAndLock`, `editRole.memberAdminConfirmationOnly`, `editRole.promoteToOwnerRequiresEligibility`, `editRole.recheckRefusesWithStatedReason`, `editRole.selfRefused` and the back-end half of `editRole.adminWithoutFactorAllowedThenGated` pass.

- [ ] **T-18 · `ManageMembershipUseCase`** — Owner or Admin; target ≠ actor; action `SUSPEND | REINSTATE | REMOVE`; Owner target → step-up, lock, `LastOwnerGuard`; `AuthorityChecker.require`; `SuspendMembership` / `ReinstateMembership` / `RevokeMembership` **[009]** (cascade, version, `AuthorityChanged`); the matching notification; audit; outcome; no session port. *(plan §2.1 `ManageMembershipUseCase`, D5, D11; RF-7.1, RF-7.2, RF-7.3, RF-7.4, RF-7.5, RF-7.6, RF-7.7, RF-7.9, RF-7.10, RF-11.5, RNF-6, RNF-7, LC-16, LC-17, LC-18)* — after T-17
  **Done when:** `manage.ownerTargetRequiresStepUpLockAndGuard`, `suspend.deniesNextRequestKeepsGrantsRefusesQueuedCancelsRunningKeepsFacts`, `reinstate.grantsUnchangedNewVersionNothingResumed`, `remove.cascadesGrantsRetainsEvidenceIdentityUntouchedRowGone`, `manage.recheckRefusesWithStatedReason`, `manage.selfRefused` and `change.sessionsNeverRevokedDeniedScopeAtNextRequest` pass.

- [ ] **T-19 · Notification wiring for role, membership and ownership changes** — the five membership templates and the two ownership templates enqueued through `SecurityNotificationPort` **[002]** in the change's transaction with the request locale; audit `NOTIFICATION_ENQUEUED`. *(plan §2.1 Outbox relay and email adapter, D12; RF-5.5, RF-6.6, RF-7.8, RF-13.4)* — after T-18, T-05
  **Done when:** `notifications.ownershipReceivedAndTransferred` and `change.versionAuditAndNotification` (role, suspend, reinstate, remove) pass.

- [ ] **T-20 · `IssueInvitationUseCase`** — Owner or Admin; `EmailAddress` **[002]** normalisation; `IdentityFactsPort.lookupByEmail` and the pending invitation by (tenant, address) as internal reads; `AdministrationRule` **[009]** on role and grants; `InvitationIssueRule`; refusals `access.invitation.conflict{reason}`; otherwise one transaction: `TokenGenerator` secret, `TokenHasher` hash, `Invitation.issue`, save, outbox `invitation` (link on the public origin of `SignupPolicy` **[002]**), audit `INVITATION_ISSUED{scope}`, outcome; no membership, grant or identity. *(plan §2.1 `IssueInvitationUseCase`, D6, D7, D15; RF-8.1, RF-8.2, RF-8.4, RF-8.5, RF-8.6, RF-8.7, RF-11.4, RF-11.7, RNF-9, RNF-10, RNF-13, LC-31, LC-32, LC-33, LC-34)* — after T-12, T-09, T-10
  **Done when:** `issue.refusesExistingMembershipAndPendingWithReasons`, `issue.unknownAddressIssuedIdentically`, `issue.oneTransactionSecretHashSaveMessageAudit`, `issue.createsNoMembershipGrantOrIdentity`, `issue.rowAppearsInBothTables`, `issue.boundedByAdministrationRule` and `issue.resubmitIssuesOnce` pass.

- [ ] **T-21 · `ReviewInvitationQuery`, `ResendInvitationUseCase`, `RevokeInvitationUseCase`** — review returns the scope element by element, issuer, issued and expiry for a pending invitation only; resend locks the row, requires `PENDING`, new secret and expiry, sends the message again, audits `INVITATION_RESENT`; revoke is the conditional `UPDATE … WHERE state = 'PENDING'`, audits `INVITATION_REVOKED`, sends nothing; the issuer's current authority is never consulted for validity. *(plan §2.1 `ReviewInvitationQuery`, `ResendInvitationUseCase`, `RevokeInvitationUseCase`, D6; RF-9.2, RF-9.3, RF-9.4, RF-9.7, RF-9.8, LC-35, LC-36)* — after T-20
  **Done when:** `resend.newSecretNewExpiryOldLinkDeadScopeUnchanged`, `revoke.finalSilentRowGone` and `invitation.survivesInviterLossOfAuthority` pass.

- [ ] **T-22 · `ExpireInvitationsJob`** — durable job (`sharedsupport/jobs` **[002]**) claiming `PENDING` invitations past `expiresAt` in bounded batches under each tenant's context; `Invitation.expire`; audit `INVITATION_EXPIRED`; no message. *(plan §2.1 `ExpireInvitationsJob`, D13; RF-9.5, LC-37)* — after T-20
  **Done when:** `expiryJob.settlesExpiredSilently` passes with a controllable clock, and an unexpired invitation is untouched.

- [ ] **T-23 · `InvitationSettled` event and `InvitationSettledHandler`** — plan 008's `AcceptInvitationUseCase` and `DeclineInvitationUseCase` **[008]** enqueue `InvitationSettled{invitationId, tenantId, outcome, issuedBy}` in their transaction (cross-plan item of plan §6); the handler enqueues `invitation-settled` to the issuer's verified address through `IdentityFactsPort`, regardless of the issuer's current membership, in the locale stored on the invitation, and audits. *(plan §2.1 `InvitationSettledHandler`, D8, D12; RF-9.6, RF-13.4, LC-38, LC-39)* — after T-19
  **Done when:** `settled.inviterNotifiedOnAcceptAndDecline` and `settled.notifiedEvenIfInviterRemoved` pass against plan 008's use cases.

## D · Infrastructure adapters

- [ ] **T-24 · Persistence adapters** — implementations (persistence access API per A-35) of `TenantOwnershipRepository` (`SELECT … FOR UPDATE` on the lock row; `activeOwnerCount` joining `tenant_membership` and identity status under that lock), the invitation repository extensions **[008]** (resend, revoke, expire, pending by (tenant, address), claim expired), and the step-up columns **[003]**. *(plan §2.1 Persistence, D1, D6; RF-10.2, RF-10.3, RNF-2, RNF-4, LC-4, LC-32)* — after T-02…T-04, T-12
  **Done when:** repository tests against real PostgreSQL pass for every method, `ownershipLock.serialisesOwnerAffectingChanges` passes, and the partial unique index rejects a second pending invitation through the adapter.

- [ ] **T-25 · Email adapter and outbox wiring** — the eight templates behind the existing outbox relay **[002]**; language from the request header for every message except `invitation-settled`, which uses the locale stored on the invitation; links on the public origin only; relay failures retried, never surfaced. *(plan §2.1 Outbox relay and email adapter, D12; RF-13.4, RNF-14, LC-25)* — after T-05, T-23
  **Done when:** `message.languageRules` and `notification.failureNeverRevertsChange` pass.

- [ ] **T-26 · Audit adapter** — map the events of plan §2.1 (`OWNERSHIP_TRANSFERRED`, the four membership events written by plan 009's commands with this feature's correlation id, `CHANGE_REFUSED{reason}`, `STEP_UP_CONSUMED`, `INVITATION_ISSUED | RESENT | REVOKED | EXPIRED`, `NOTIFICATION_ENQUEUED | FAILED`) to `AuditPort` with tenant, acting and target membership and identity, previous and new value, version, correlation id, time; a serialisation guard rejects any payload holding a password, code, secret or project content; entries written in the effect's transaction. *(plan §2.1 Audit adapter, D18; RF-3.4, RF-13.1, RF-13.2)* — after T-12
  **Done when:** `audit.entryPerEvent` and `audit.containsNoPasswordCodeSecretOrContent` pass.

- [ ] **T-27 · Problem codes and redaction** — `access.last-owner` (409, reason and transfer route), `access.transfer.refused` (409, reason), `access.invitation.conflict` (409, reason), `access.step-up.required` (428), beside `access.forbidden-change` **[009]** and `access.denied-scope` **[006/009]**; request-body logging excludes `password`, `code`, `secret`; problem details never echo them nor another identity's security facts. *(plan §2.1 Problem codes, D16; RF-13.3)* — after T-26
  **Done when:** `problems.reasonsForAdministratorsNoIdentitySecurityFacts` passes across every refusal of the feature with secrets present in the requests.

- [ ] **T-28 · RLS and tenant isolation** — `FORCE RLS` with `USING` and `WITH CHECK` on `tenant_ownership` and the extended `tenant_invitation`; the runtime role non-owner, non-superuser, no `BYPASSRLS`; every query and command of this feature runs under the C10 tenant context of plan 009's filter. *(plan §2.1 Persistence, Data model note; RF-1.5, RNF-8)* — after T-24
  **Done when:** `rls.membersInvitationsAndLockRowTenantScoped` passes: cross-tenant reads, writes and lock attempts on the four tables are refused under the runtime role.

## E · API

- [ ] **T-29 · `GET /api/tenants/{tenantId}/members` and `GET successors`** — the list with `?search&role&state&page` → 200 `{rows, total, page}`; successors → 200 `{candidates[], statementKey}` for an Owner only; a Member receives 403 `access.denied-scope`. *(plan §2.1 `TenantMembersController`; RF-1.3, RF-2.1, RF-2.3, RF-2.4, RF-2.5, RF-4.2, RF-4.3, RF-4.4, LC-13, LC-21)* — after T-13, T-14
  **Done when:** `members.ownerAndAdminOnly`, the back-end half of `members.memberGetsDeniedScopeNoRows`, `membersList.exposesNoFactorCountTenantOrGrant`, `transfer.ownerOnlyAndOwnRowOnly` and the contract half of `rowActions.exactPerActorRole` pass.

- [ ] **T-30 · `POST ownership/transfer`, `POST {membershipId}/role`, `POST {membershipId}/suspend | reinstate | remove`** — bodies carry the step-up challenge id (when Owner-affecting), the versions the dialog opened against and an idempotency key; 200 with the committed roles, statuses and versions; 409 / 428 / 403 problems as T-27; a missing challenge on an Owner-affecting change → 428 `access.step-up.required`. *(plan §2.1 `TenantMembersController`; RF-5.2, RF-5.6, RF-6.5, RF-7.7, RF-10.4, RF-13.3, RF-13.6)* — after T-18, T-27
  **Done when:** `transfer.responsesCarryNamesOnly`, the back-end half of `confirmation.staleVersionRefusedAndRefreshed` and `responses.carryCommittedState` pass, and an Owner-affecting change without a challenge id answers 428.

- [ ] **T-31 · Invitation endpoints** — `GET invitations?page`, `POST invitations` {email, role, project?, idempotencyKey} → 201 `{invitationId, expiresAt}` or 400 field paths or 409 `access.invitation.conflict`, `GET invitations/{id}`, `POST invitations/{id}/resend`, `POST invitations/{id}/revoke`; the UI locale in the request header. *(plan §2.1 `TenantMembersController`; RF-8.2, RF-8.3, RF-8.4, RF-9.1, RF-9.2, RF-9.3, RF-9.4)* — after T-21
  **Done when:** `issue.fieldPathProblems`, `issue.ownerRoleNeverOfferedNorAccepted` and `invitations.listShape` pass.

- [ ] **T-32 · Absence checks** — a test that enumerates the API surface and the application module and asserts: no call to `GrantCommands` or `CreateMembership` **[009]**; no endpoint over another identity's credential, factor, codes, sessions, address or erasure; no action on the actor's own row and no invitation to the actor's own address; no write to tenant state, policy or another tenant; no Owner-affecting action or Owner invitation reachable by an Admin. *(RF-10.5, RF-11.1, RF-11.2, RF-11.3, RF-11.4, RF-11.6, LC-34)* — after T-31
  **Done when:** `feature.callsNoGrantOrCreateMembershipCommand`, `noIdentityControlEndpointExists`, `self.cannotEditManageOrInvite`, `change.touchesNoTenantPolicyOrOtherTenant` and `admin.cannotReachOwnerAffectingActionOrOwnerInvitation` pass.

## F · Front end

- [ ] **T-33 · `contracts/members` types and guards** — request / response / problem types with runtime guards for the ten endpoints and the six problem codes; a guard rejects any row carrying a password, factor state, code count, other tenant or project grant. *(plan §2.2 `contracts/members`; RF-2.3, RF-13.3)* — after T-01
  **Done when:** guard unit tests accept every documented response and problem shape and reject a row fixture with an extra `factor`, `codeCount`, `tenant` or `grants` field.

- [ ] **T-34 · i18n bundle `workspace-members.{en,es,fr}`** — every heading, tab, column, label, action, statement, state, reason and validation text of R13, N23 and the dialogs and panels of D02 `419:2645` (S-50), including the six problem reasons and the refused states. *(D14, D16, §6 S-50; RF-3.5, RF-12.1, RNF-16)*
  **Done when:** a key-parity test shows the three files have identical key sets and no empty value.

- [ ] **T-35 · Hooks** — `useTenantMembers`, `useSuccessors`, `useTransferOwnership`, `useEditRole`, `useManageMembership`, `useIssueInvitation`, `useReviewInvitation`, `useResendInvitation`, `useRevokeInvitation` (mutations with retry disabled and the UI locale header; each generates its idempotency key once per mount and reuses it on re-submission) and `useStepUp()` **[003]** with the purpose `TENANT_OWNERSHIP`. *(plan §2.2 hooks and State, D15; RF-2.8, RNF-13)* — after T-33
  **Done when:** unit tests show each mutation sends the current locale, has retry disabled, and the same idempotency key is sent on a second call of the same mounted hook.

- [ ] **T-36 · Feature module, route and `PeoplePage` shell** — `src/features/workspace-members` with its public API; route `/t/{tenantId}/settings/people` with the tabs as nested states; breadcrumb, heading, subtitle, "Invite member", search with the shortcut, role and state filters, pagination line, the `AuthorityStatements` **[009]** `members` variant (last-Owner notice, card, sub-statements, "Review project grants"), "Decision history ↗", "Workspace policy" routing to the settings feature; the `useAuthority(TENANT_MEMBERS)` **[009]** gate with `DeniedScopeHandler` **[006]**; the Admin variant renders whatever actions the rows carry (S-49); no help surface is added (the app shell's entry follows S-17). *(plan §2.2 `routes`, `PeoplePage`, `AuthorityStatements`, D14; RF-1.1, RF-1.2, RF-1.3, RF-1.6, RF-1.7, RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-3.5, LC-21, LC-30)* — after T-34, T-35
  **Done when:** `peoplePage.rendersHeadingTabsFiltersTablesStatementsAndRoutes` (desktop, tablet, mobile), `peoplePage.tenantPolicyTabRoutesToSettings`, `authorityStatements.membersVariantVerbatimAtEveryWidthAndRole` and `routes.reviewGrantsAndDecisionHistory` pass, and the front-end half of `members.memberGetsDeniedScopeNoRows` shows the denied-scope state.

- [ ] **T-37 · `MembersTable` and `PendingInvitationsTable`** — the member columns with "you", the "TOTP required" marker, "Invitation pending" rows, action cells rendering exactly the server's actions (two actions in one cell when both apply); the invitations table with address, role, scope or "membership only", issuer, issued, expires, "Review invitation"; records at narrow widths; the empty state keeps the filters editable. *(plan §2.2 `MembersTable`, `PendingInvitationsTable`, D9; RF-2.1, RF-2.2, RF-2.4, RF-2.6, RF-2.7, RF-9.1, RF-12.2; D-d)* — after T-36
  **Done when:** the front-end half of `rowActions.exactPerActorRole` and of `membersList.emptyScopeKeepsFilters` pass, and the invitation rows render `invitations.listShape` fixtures without a membership state.

- [ ] **T-38 · `StepUpFields`** — wrapper around `PasswordField` and `CodeField` **[001]** reproducing the step-up block of plan 003's dialogs, submitting through `useStepUp()` with the purpose `TENANT_OWNERSHIP`, rendering the generic failure of `mfa.step-up.rejected`, and closing the host dialog with nothing changed on `mfa.step-up.void`. *(plan §2.2 `StepUpFields`, D3; RF-5.1, LC-9, LC-10)* — after T-35
  **Done when:** the front-end half of `stepUp.voidClosesWithNothingChanged` passes and a rejected step-up shows the generic message without naming the input.

- [ ] **T-39 · `TransferOwnershipDialog`** — N23 verbatim: statement, authority line, `SuccessorPicker` from `useSuccessors` with the eligibility statement and the "no eligible member" state, `StepUpFields`, "Verify & transfer ownership" (step-up first, then the transfer with the consumed challenge id and the versions the dialog opened against), "Keep current Owner"; refusal reasons from the problem code's i18n key; nothing on close. *(plan §2.2 `TransferOwnershipDialog`; RF-4.1, RF-4.2, RF-4.3, RF-4.5, RF-4.6, RF-5.2, LC-5, LC-7, LC-8)* — after T-38
  **Done when:** `transferDialog.rendersN23ContentAndKeepDoesNothing` passes, the picker renders only the server's candidates with the statement, and a mocked `access.transfer.refused` renders its reason.

- [ ] **T-40 · `EditRoleDialog`, `ManageMembershipDialog` and the dialog state** — both on N23's anatomy (S-50): the role dialog offers only the roles the server's row allows, the effect statements, `StepUpFields` only when `OwnerAffectingRule` (mirrored from the contract) says so; the membership dialog offers suspend / reinstate / remove by state with the effect statement of the chosen action and `StepUpFields` for an Owner target; module-private dialog state (target id, versions, action, challenge id) cleared on close, success, void and route leave; nothing in storage. *(plan §2.2 `EditRoleDialog`, `ManageMembershipDialog`, Dialog state, D5, D10, D14; RF-6.1, RF-6.2, RF-6.3, RF-6.4, RF-7.1, RF-7.2, RF-7.3, RF-10.4, RF-13.5, LC-14, LC-15, LC-16, LC-17, LC-18, LC-23, LC-24)* — after T-38
  **Done when:** `editRoleDialog.offersOnlyPermittedRoles`, `manageDialog.actionsByState`, `storage.containsNothingFromFeature` and `dialog.reloadResubmitsSameKey` pass, and a language switch with a dialog open keeps its state.

- [ ] **T-41 · `InviteMemberDialog` and `ReviewInvitationPanel`** — the invite dialog with address, Admin / Member choice, one optional project with Author / Reviewer / Viewer and canRun as independent unticked choices, the statements, "Send invitation" / "Cancel", conflict reasons from the problem code; the review panel with the scope element by element, issuer, issued, expiry, "Resend invitation" and "Revoke invitation" each behind a confirmation stating the effect, "Close". *(plan §2.2 `InviteMemberDialog`, `ReviewInvitationPanel`, D6, D7; RF-8.1, RF-8.2, RF-8.3, RF-8.4, RF-9.2, RF-9.3, RF-9.4, LC-31, LC-32, LC-33)* — after T-36
  **Done when:** `inviteDialog.fieldsStatementsAndChoices` and `reviewPanel.scopeIssuerExpiryAndActions` pass, and the Owner role is never present among the choices.

- [ ] **T-42 · Handoffs, stale handling and authority refresh** — a stale-version refusal → `DeniedScopeHandler` **[006]** refreshes authority once and the list reloads; after a transfer the actor's authority store **[009]** refreshes and the rows re-render; a suspended or deleting tenant → N20 by the app shell; "Review project grants" → plan 011's surface; "Decision history ↗" → N18 with the membership filter. *(plan §2.2 Routing, D10; RF-1.4, RF-3.3, RF-3.4, RF-5.6, RF-10.4, LC-11, LC-12, LC-20, LC-27)* — after T-39, T-40, T-41
  **Done when:** the front-end half of `confirmation.staleVersionRefusedAndRefreshed` and `peoplePage.suspendedTenantShowsN20` pass, and after a mocked transfer the own row shows no "Transfer ownership" action.

- [ ] **T-43 · Reflow and touch targets** — Tailwind breakpoints for the page, both tables as records, the dialogs and the panel; `shared/ui` field, password field, code field, checkbox, tabs, pagination and buttons at 44 px minimum on touch widths; visible labels and focus; action cells reachable by keyboard in reading order; breakpoint layout replaceable (S-02). *(D14, §6 S-02; RF-12.1, RF-12.2, RF-12.3, RNF-15, RNF-17)* — after T-42
  **Done when:** a layout test at 360 px, 768 px and 1280 px shows no horizontal overflow on every state and every interactive element ≥ 44 px on the two smaller widths.

## G · Verification suites

- [ ] **T-44 · Concurrency suite** — two Owners demoting each other; a transfer racing a suspension of the successor; the actor changed since the dialog opened; a revoke racing plan 008's acceptance; two administrators on one row; partial state never observable. *(D1, D2, D6, D10; RF-10.2, RF-10.3, RF-10.4, RF-10.6, RF-9.7, RNF-2, RNF-3, RNF-4, RNF-18, LC-3, LC-4, LC-27, LC-36)* — after T-24, T-21
  **Done when:** `lastOwner.twoOwnersDemoteEachOtherOneRefused`, `lastOwner.neverZeroActiveOwnersUnderConcurrency`, `confirmation.actorChangedSinceOpenRefused`, `revoke.vsAcceptOneTransition` and `change.partialStateNeverObservable` pass ten times in a row with real PostgreSQL.

- [ ] **T-45 · Last-Owner and guard suite** — the guard refuses with the stated reason and the transfer route on every path; a restricted or suspended Owner does not count; an inactive tenant refuses every command. *(D1; RF-1.4, RF-10.1, LC-2, LC-20, LC-26)* — after T-18
  **Done when:** `lastOwner.refusedWithReasonAndRoute`, `lastOwner.restrictedOwnerDoesNotCount` and `members.inactiveTenantRefusesEveryCommand` pass.

- [ ] **T-46 · Enumeration and invitation-shape suite** — issuing to a known and an unknown address compared in content, shape and timing; no response reveals identity existence or security facts; the invitation record holds Admin or Member, at most one project, a hashed secret and one pending per address. *(D6, D7; RF-8.4, RF-11.7, RNF-9, RNF-10, LC-33)* — after T-31
  **Done when:** `issue.timingComparableKnownUnknownAddress` passes on the CI runner for three consecutive runs, and `responses.revealNoIdentityExistenceOrSecurityFacts` and `invitation.adminOrMemberOneProjectHashedOnePending` pass.

- [ ] **T-47 · Fail-closed and scope suite** — stale, unknown, closed, inactive and unconfigured inputs all refuse; identity, credential, factor, sessions and other tenants untouched; every row action and confirmation decided on the server; a removed member returns only by invitation with the invited scope; a removed address is not listed but audited. *(D10, D11; RF-2.2, RF-7.10, RNF-1, RNF-7, RNF-11, LC-19, LC-22)* — after T-32
  **Done when:** `failClosed.staleUnknownClosedInactiveUnconfiguredRefuses`, `change.identityCredentialFactorSessionsOtherTenantsUntouched`, `authority.serverDecidesEveryRowActionAndConfirmation`, `removed.returnsOnlyByInvitationWithInvitedScope` and `search.removedAddressNotListedAuditHasIt` pass.

- [ ] **T-48 · Startup suite** — one case per property of T-01. *(D17; RNF-12)* — after T-01
  **Done when:** `tenantMembersPolicy.startupFailsOnAbsentOrUnboundedValue` passes, each case failing the context refresh with a message naming the property, and the complete set starts.

- [ ] **T-49 · FE e2e journeys** — Playwright journeys for US-1 … US-12 in EN, ES and FR at desktop, tablet and mobile widths with BE doubles and a fake email port: list, search and filter; transfer with step-up and landing of both parties at their next request; promote, demote, suspend, reinstate, remove with and without step-up; an Admin without a factor promoted then gated at tenant entry; invite, resend, revoke, expiry, acceptance and decline reflected in both tables with the inviter notified; the Member denied; the suspended tenant; "Decision history" scoped; the language switch on every state. *(RF-5.6, RF-6.7, RF-1.6, RF-1.3, RF-1.4, RF-3.4, RNF-16, LC-11, LC-12, LC-20, LC-21, LC-23, LC-30, LC-38; US-1 … US-12)* — after T-43, T-32
  **Done when:** `transfer.successorDerivesOwnerAtNextRequest`, `transfer.formerOwnerDerivesAdminAtNextRequest`, the end-to-end half of `editRole.adminWithoutFactorAllowedThenGated`, `languageSwitch.onPeopleKeepsDialogState`, `allStates.existInEnEsFr` and `decisionHistory.scopedByReadAudit` pass in the three languages and three widths.

- [ ] **T-50 · Automated accessibility** — axe on the page, both tables, every dialog and panel and every state at the three widths and three languages. *(RF-12.1, RF-12.2, RNF-15, RNF-16, RNF-17)* — after T-49
  **Done when:** `a11y.people.{en,es,fr}.{desktop,tablet,mobile}` reports zero critical or serious violations on every combination.

- [ ] **T-51 · Manual assistive evidence** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) checklist: table navigation, action cells in reading order, dialog focus, refusal reasons announced without focus theft, step-up fields labelled, 44 px targets; versions recorded in the release-support matrix. *(RF-12.2, RNF-15; FR-X-040)* — after T-50
  **Done when:** the checklist is filled with pass on every row and attached to the release-support matrix.

- [ ] **T-52 · Adversarial suite (US-13)** — an Admin appointing an Owner by role edit, by invitation or by a direct submission; an Owner invitation; the last-Owner race; a self-change of role or membership; address enumeration through issuing; identity controls on another person; a project read through a tenant role; an unversioned or stale change; a reused step-up challenge. *(D1, D3, D5, D7, D9, D10; RF-8.2, RF-10.5, RF-11.1, RF-11.2, RF-11.3, RF-11.7, RNF-1, RNF-2, RNF-5, RNF-10; US-13)* — after T-32, T-44, T-47
  **Done when:** every attack test fails to achieve its goal and asserts the refusal or the generic response.

## H · Closing

- [ ] **T-53 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-44 … T-52
  **Done when:** the script reports 145 of 145 identifiers (88 RF, 18 RNF, 39 LC) covered by at least one passing test.

- [ ] **T-54 · Register and cross-plan review** — walk §6 of the plan (S-50, S-49, S-04, S-14, S-02, A-52, A-54 … A-57, A-30, A-27, A-36, A-35, and the three cross-plan items: plan 008's `InvitationSettled` event, the project-grant shape of plans 008 and 009, plan 012's creation of the lock row): for each, confirm the built assumption matches the current state or record the deferral. *(spec §7 item 14)* — after T-53
  **Done when:** each of the fifteen items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it.

- [ ] **T-55 · Completion criteria sign-off** — tick spec §7 items 1–14 with the evidence (test run id, concurrency and timing reports, a11y report, manual checklist, coverage report, register review). — after T-54
  **Done when:** all fourteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-36 |
| RF-1.2 | T-36 |
| RF-1.3 | T-13, T-29, T-36, T-49 |
| RF-1.4 | T-42, T-45, T-49 |
| RF-1.5 | T-13, T-28 |
| RF-1.6 | T-36, T-49 |
| RF-1.7 | T-36 |
| RF-2.1 | T-11, T-13, T-29, T-37 |
| RF-2.2 | T-13, T-37, T-47 |
| RF-2.3 | T-11, T-13, T-29, T-33 |
| RF-2.4 | T-11, T-13, T-29, T-37 |
| RF-2.5 | T-11, T-13, T-29 |
| RF-2.6 | T-11, T-13, T-37 |
| RF-2.7 | T-13, T-37 |
| RF-2.8 | T-13, T-35 |
| RF-3.1 | T-36 |
| RF-3.2 | T-36 |
| RF-3.3 | T-36, T-42 |
| RF-3.4 | T-26, T-36, T-42, T-49 |
| RF-3.5 | T-34, T-36 |
| RF-4.1 | T-39 |
| RF-4.2 | T-08, T-14, T-29, T-39 |
| RF-4.3 | T-14, T-29, T-39 |
| RF-4.4 | T-14, T-16, T-29 |
| RF-4.5 | T-39 |
| RF-4.6 | T-39 |
| RF-5.1 | T-04, T-15, T-16, T-38 |
| RF-5.2 | T-08, T-16, T-30, T-39 |
| RF-5.3 | T-16 |
| RF-5.4 | T-16 |
| RF-5.5 | T-16, T-19 |
| RF-5.6 | T-16, T-30, T-42, T-49 |
| RF-5.7 | T-16 |
| RF-5.8 | T-06, T-16 |
| RF-6.1 | T-17, T-40 |
| RF-6.2 | T-17, T-40 |
| RF-6.3 | T-07, T-08, T-17, T-40 |
| RF-6.4 | T-07, T-17, T-40 |
| RF-6.5 | T-15, T-17, T-30 |
| RF-6.6 | T-17, T-19 |
| RF-6.7 | T-17, T-49 |
| RF-7.1 | T-18, T-40 |
| RF-7.2 | T-18, T-40 |
| RF-7.3 | T-07, T-18, T-40 |
| RF-7.4 | T-18 |
| RF-7.5 | T-18 |
| RF-7.6 | T-18 |
| RF-7.7 | T-18, T-30 |
| RF-7.8 | T-19 |
| RF-7.9 | T-18 |
| RF-7.10 | T-18, T-47 |
| RF-8.1 | T-09, T-20, T-41 |
| RF-8.2 | T-10, T-20, T-31, T-41, T-52 |
| RF-8.3 | T-31, T-41 |
| RF-8.4 | T-03, T-10, T-20, T-31, T-41, T-46 |
| RF-8.5 | T-03, T-09, T-20 |
| RF-8.6 | T-20 |
| RF-8.7 | T-20 |
| RF-9.1 | T-11, T-13, T-31, T-37 |
| RF-9.2 | T-21, T-31, T-41 |
| RF-9.3 | T-03, T-09, T-21, T-31, T-41 |
| RF-9.4 | T-03, T-09, T-21, T-31, T-41 |
| RF-9.5 | T-03, T-09, T-22 |
| RF-9.6 | T-23 |
| RF-9.7 | T-21, T-44 |
| RF-9.8 | T-21 |
| RF-10.1 | T-06, T-17, T-45 |
| RF-10.2 | T-02, T-24, T-44 |
| RF-10.3 | T-02, T-06, T-24, T-44 |
| RF-10.4 | T-17, T-30, T-40, T-42, T-44 |
| RF-10.5 | T-17, T-32, T-52 |
| RF-10.6 | T-17, T-44 |
| RF-11.1 | T-32, T-52 |
| RF-11.2 | T-12, T-32, T-52 |
| RF-11.3 | T-10, T-32, T-52 |
| RF-11.4 | T-12, T-20, T-32 |
| RF-11.5 | T-18 |
| RF-11.6 | T-12, T-32 |
| RF-11.7 | T-10, T-20, T-46, T-52 |
| RF-12.1 | T-34, T-43, T-50 |
| RF-12.2 | T-37, T-43, T-50, T-51 |
| RF-12.3 | T-43 |
| RF-13.1 | T-26 |
| RF-13.2 | T-26 |
| RF-13.3 | T-27, T-30, T-33 |
| RF-13.4 | T-05, T-19, T-23, T-25 |
| RF-13.5 | T-40 |
| RF-13.6 | T-30 |
| RNF-1 | T-47, T-52 |
| RNF-2 | T-02, T-16, T-24, T-44, T-52 |
| RNF-3 | T-16, T-44 |
| RNF-4 | T-02, T-24, T-44 |
| RNF-5 | T-04, T-07, T-15, T-52 |
| RNF-6 | T-16, T-18 |
| RNF-7 | T-18, T-47 |
| RNF-8 | T-28 |
| RNF-9 | T-03, T-09, T-20, T-46 |
| RNF-10 | T-20, T-46, T-52 |
| RNF-11 | T-47 |
| RNF-12 | T-01, T-48 |
| RNF-13 | T-20, T-35 |
| RNF-14 | T-05, T-25 |
| RNF-15 | T-43, T-50, T-51 |
| RNF-16 | T-34, T-49, T-50 |
| RNF-17 | T-43, T-50 |
| RNF-18 | T-44 |
| LC-1 | T-16 |
| LC-2 | T-06, T-45 |
| LC-3 | T-17, T-44 |
| LC-4 | T-24, T-44 |
| LC-5 | T-16, T-39 |
| LC-6 | T-16 |
| LC-7 | T-08, T-14, T-39 |
| LC-8 | T-08, T-14, T-39 |
| LC-9 | T-15, T-38 |
| LC-10 | T-15, T-38 |
| LC-11 | T-16, T-42, T-49 |
| LC-12 | T-16, T-42, T-49 |
| LC-13 | T-17, T-29 |
| LC-14 | T-17, T-40 |
| LC-15 | T-17, T-40 |
| LC-16 | T-18, T-40 |
| LC-17 | T-18, T-40 |
| LC-18 | T-18, T-40 |
| LC-19 | T-10, T-47 |
| LC-20 | T-42, T-45, T-49 |
| LC-21 | T-13, T-29, T-36, T-49 |
| LC-22 | T-13, T-47 |
| LC-23 | T-40, T-49 |
| LC-24 | T-16, T-40 |
| LC-25 | T-25 |
| LC-26 | T-06, T-45 |
| LC-27 | T-17, T-42, T-44 |
| LC-28 | T-13 |
| LC-29 | T-08, T-14, T-17 |
| LC-30 | T-36, T-49 |
| LC-31 | T-10, T-20, T-41 |
| LC-32 | T-10, T-20, T-24, T-41 |
| LC-33 | T-10, T-20, T-41, T-46 |
| LC-34 | T-10, T-20, T-32 |
| LC-35 | T-21 |
| LC-36 | T-21, T-44 |
| LC-37 | T-22 |
| LC-38 | T-23, T-49 |
| LC-39 | T-23 |
| US-1 … US-12 | T-49 (journeys) |
| US-13 | T-52 (adversarial suite) |

Self-check: all 88 RF criteria, 18 RNF and 39 limit cases of `specTenantMembersOwnership.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.

