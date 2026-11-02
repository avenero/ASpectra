# Specification 015 — Leave tenant and identity erasure request

**Feature** Leaving one tenant membership, and requesting the verified erasure of one's own global identity · **Version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — open doubts closed 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-17 as the subject; F-IDN-03, F-IDN-05, F-IDN-07, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-15, F-IDN-16, F-SET-02 and F-SET-03 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P3, P6, P8, P12, P13, P14) → `requirements.md` (FR-IDN-020, 023, 028, 040, 046, 049, 061, 067, 068, 093, 094, 095, 099, 120, 121, 122, 123; FR-SET-010, 013; FR-X-001, 002, 004, 005, 020, 030, 033, 034, 040, 042, 043, 044, 045, 046, 051, 053, 055) → the design of record, screen N19 AccountDeletionPage `161:50881` (the "Leave Northstar Labs" card in its last-Owner blocked state, the "Erase my global identity" card, the "Ownership checks · all of your memberships" table, the "Verified erasure request" panel with its two acknowledgements, the typed confirmation "ERASE MY IDENTITY", "Keep my account" and "Continue to identity verification", and the statements "Minimal history, not forever PII" and "Recovery is separate"), the "Your membership" block and "Account privacy" tab of R12 PersonalSettingsPage `148:644`, the dialog N23 OwnershipTransferDialog `159:32066` inside the D02 atlas `159:32045`, and decision register `122:2` §3 ("Tenant membership removal revokes that membership/project grants, not the global identity … Global account erasure is a separate verified workflow with ownership transfer across owned tenants, credential/session revocation and minimized/pseudonymous attribution subject to lawful retention").

**Scope note.** Constitution v1.5 bound F-IDN-17 to FR-IDN-122 only in its erasure-request gates — acknowledgements, typed confirmation, last-Owner checks across all memberships, no one-click and no instant-erasure promise. The product owner decided on 28 October 2026 (decisions Q1, Q2) that this specification also covers the independent verification that precedes the request and what recording the request changes at once; the constitution was amended the same day (v1.6) to bind those parts. Carrying out the erasure — minimising or pseudonymising retained attribution, purging what may be purged, and ending the memberships — belongs to the platform's internal, audited erasure process and is not specified here.

**Related specifications:** 001 Login (an identity under erasure meets the generic sign-in failure, RF-2.7) · 003 TOTP enrollment, recovery codes and factor replacement (the step-up re-authentication reused here, RF-8.2 and RF-8.3 there, as specification 010 Q4 applies it) · 004 Password reset and 005 Controlled account recovery (neither works for an identity under erasure; an open recovery case closes, 005 RF-5.6; "Recovery is separate") · 006 Sessions, refresh and expiry (the security version advanced at erasure ends every session, RF-2.5; the denied-scope state a leaver's other tabs meet, RF-5.1) · 007 Logout and session management · 008 Invitation acceptance (invitations to an identity under erasure cannot be accepted, LC-28) · 009 Roles, grants and the authority matrix (revocation and work in flight, Q5 / RF-7; derivation for an identity under erasure, LC-29) · 010 Tenant members and ownership transfer (ownership transfer N23 and the last-Owner guard, RF-4 and RF-10; *active Owner*, D-a; nobody manages their own row, Q5) · 013 Tenant chooser (the landing after leaving the current tenant; the chooser offers neither leaving nor erasure) · 014 Tenant suspension state (leaving a suspended tenant, RF-5.3 and RF-5.4; an erasure held by a suspended owned tenant, RF-5.5) · 016 Guarded tenant and project switch (the switch to the chooser after leaving the current tenant).

This document states **what** leaving a tenant and requesting identity erasure do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global**: one person with a verified email, a password, an authenticator, recovery codes and sessions, who belongs to any number of tenants through **tenant memberships**. A membership carries a tenant role — Owner, Admin, Member — and a state — active, suspended, revoked — and nothing of the identity itself (specification 009). Two very different things can therefore be meant by "leaving":

- **Leaving a tenant** ends one membership. That tenant's access and grants go; the identity, its credentials, its sessions and its other memberships stay exactly as they were. It is reversible only in the sense that the person may be invited again.
- **Erasing the global identity** ends the person's presence in ASpectra as a whole. It is a high-impact, irreversible act on a principal shared across customers: it ends every session, closes every route back in, and hands the identity to a verified, internal process that removes what may be removed and minimises what lawful retention requires to be kept.

The design of record draws both on one page, N19, reached from personal settings, and puts the difference at the top of it: "Leaving a workspace is not erasing an identity." Both operations meet the same guard that protects every tenant: a tenant can never be left without an active Owner, so a person who is the last Owner anywhere must hand ownership on first.

### 1.2 Objective

Let a person leave any one of their tenants, or request the erasure of their own identity, safely and honestly, while:

- presenting and implementing the two as different operations, with their effects stated before anything is confirmed;
- never letting either leave a tenant without an active Owner, checked across every membership and rechecked on the server at the moment of commitment;
- making erasure deliberate — acknowledgements, a typed confirmation, a fresh password-and-authenticator proof and a code sent to the verified address — and never one-click;
- ending, at the moment the request is recorded, every session and every way back in, so that a stolen session cannot be used to undo or exploit the request;
- promising no instant erasure, no completion time and no retention duration, and stating that required records may be kept with minimised identity references;
- showing the person only their own memberships and ownership checks — no project content, no other member, no other customer's data;
- leaving work already accepted by the platform honest: stopped at a safe point where possible, never reported as a failed test, never rewritten.

### 1.3 Why it matters

Confusing the two operations is the costly mistake in both directions. A person who meant to leave one contractor workspace and erased the identity that holds their employer's tenants has lost everything; a person who meant to be forgotten and only left one tenant still exists everywhere else. A sole Owner who can leave or erase strands a whole organisation with nobody able to administer it. An erasure that is one click away from any open session turns a stolen laptop into a permanent lockout; one that promises "deleted in 30 days" makes a legal claim the platform cannot keep. And an administrator who can erase another person — or a page that reveals another tenant's projects while listing ownership checks — breaks the rule that a global identity belongs to nobody but its holder. The feature must say exactly what each act does, ask exactly for the proofs it needs, and promise nothing it cannot keep.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 28 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, written on 28 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14: **A-64** (the identity erasure request record, the transition to under erasure and the handoff to the internal erasure process) is new; notes appended to **A-27** (identity statuses), **A-48** (the one-time email verification code), **A-30** (notification set) and **A-56** (work in flight). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-63** (N19 states and dialogs, and the R12 Account privacy tab) is new; notes appended to **S-14** (email templates), **S-02** (responsive coverage) and **S-04** (destinations of the remaining buttons, now D02 `439:4460`).

| Q | Question | Decision |
|---|---|---|
| Q1 | What "Continue to identity verification" does, and where this specification stops | Two independent proofs, both required: the step-up of specification 003 (RF-8.2, RF-8.3) — current password and a fresh authenticator code, a recovery code refused — with the purpose "erasure"; then a one-time code sent to the verified address, of the same shape as specification 005's code (A-48) and bound to the purpose "erasure". Passing both records a durable erasure request and hands it to the platform's internal, audited, out-of-band erasure process. This specification ends at "request received"; carrying out the erasure is not specified here. |
| Q2 | What recording the request changes, and whether it can be withdrawn | In one indivisible change: the identity becomes *under erasure*; its security version advances, so every session and refresh family of the identity ends at its next use (specification 006 RF-2.5), including the one that made the request; any open recovery case closes (specification 005 RF-5.6); from then on sign-in, password reset and recovery answer generically (specifications 001, 004, 005). A confirmation email goes to the verified address stating that the request was received, that no completion time is promised, that required records may be retained with minimised identity references, and giving the erasure process contact. There is no withdrawal in the product; anything after recording, including any reversal the person asks for through that contact, belongs to the internal process. |
| Q3 | How leaving a tenant works | "Leave <tenant>" opens a compact confirmation dialog built on N23's anatomy, stating the effect: the membership and its project grants are revoked; authored revisions, approvals, runs and evidence remain with their provenance; queued work will not run and running work is asked to stop at a bounded checkpoint; the identity, its sessions and its other memberships are untouched; returning requires a new invitation. A Member or Admin confirms once. An Owner who is not the last active Owner also passes the step-up of Q1's first proof, and the system rechecks under the tenant ownership lock at confirmation (specification 010 Q7). Sessions continue. When the person left the tenant they are currently in, they land on the tenant chooser (specification 013); otherwise they stay on N19. |
| Q4 | Which memberships block the erasure request | The request cannot be submitted while the identity is the only active Owner of any tenant that is active or suspended. For an active tenant N19 shows "Transfer to a verified eligible Owner" with a route to ownership transfer (N23, specification 010); where no member is eligible it says to invite a successor and have them enroll first. For a suspended tenant it shows that the request is held until the tenant is reinstated or deleted (specification 014 RF-5.5), with no action. A deleting tenant, a tenant with another active Owner, an Admin or Member membership, and a suspended membership never block. Every check is re-run on the server at submission under each tenant's ownership lock. No tenant deletion is offered from this feature. |
| Q5 | Who is told | Leaving: the person receives a confirmation email naming the tenant and the effect; the tenant's active Owners and Admins receive a short notice naming the person's display name and the tenant, with no link and no project content. Erasure request: only the verified address receives the confirmation of Q2. No tenant is told that an identity is under erasure. |
| Q6 | How N19 is reached, and where each membership can be left | N19 opens from R12 — the "Account privacy" tab and the "Account privacy options" action — and from the security settings, with or without a tenant selected. The ownership-checks table lists every membership of the identity in the active or suspended state (revoked memberships are not listed), paged by a configured size; each row offers "Leave" or the reason it cannot be left, so a suspended tenant can be left without entering it. The top card addresses the current tenant when there is one. An identity with no listed membership sees the erasure section only. |
| Q7 | What happens to work in flight, invitations and memberships | Leaving applies, for that tenant, the revocation rule of specification 009 (Q5, RF-7; A-56): queued run tickets of the person not yet admitted end as NOT EXECUTED, running attempts receive a bounded cancellation request and keep their recorded facts, unconfirmed delivery items are held, approvals already given remain as evidence. The transition to under erasure applies the same rule across every tenant of the identity. Pending invitations addressed to the identity become unusable (specification 008 LC-28); invitations the person issued stay pending for their tenant. At erasure the memberships and grants are left as they are, yielding no authority (specification 009 LC-29), for the internal process to settle. |

Four rules below are derived from the design and from earlier specifications rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** An Owner cannot leave a suspended tenant, whatever the number of Owners, because every Owner-affecting change is frozen while the tenant is suspended (specification 014 RF-5.4); the row states that the tenant is suspended. A Member or Admin may leave it (specification 014 RF-5.3).
- **D-b** A membership of a *deleting* tenant offers no "Leave" — the tenant's deletion settles it — and never blocks an erasure request; the row shows the state only.
- **D-c** Any failed, expired or abandoned step of the erasure verification returns the person to N19 with nothing recorded and nothing changed; the two acknowledgements and the typed confirmation are cleared and must be given again.
- **D-d** The typed confirmation is the phrase displayed on the page in the person's interface language ("ERASE MY IDENTITY" in English); it matches only when the entry equals that phrase exactly, letter case included, after removing leading and trailing white space.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | Member of a tenant I no longer work with | to leave that one tenant from my personal settings, with the effect stated first | I lose access there and nothing else |
| US-2 | person who belongs to several tenants | to see all my memberships and, for each, whether I can leave it and what stands in the way | I know my next step before I try |
| US-3 | Owner who shares ownership with someone else | to leave after proving it is me, with the guarantee that another Owner remains | the tenant keeps an administrator and nobody can leave in my name |
| US-4 | sole Owner who wants to leave or erase my identity | to be told plainly that I must transfer ownership first, and to reach the transfer from the same page | the organisation is never stranded |
| US-5 | Member of a suspended tenant | to leave it without entering it | I am not tied to a paused workspace |
| US-6 | person who wants to be forgotten | to request the erasure of my identity through deliberate steps and real proofs | it cannot happen by accident or by someone else's hand |
| US-7 | person who has just requested erasure | to be signed out everywhere and told by email what happens next, without a promised date | a stolen session cannot undo or exploit my request, and I am not misled |
| US-8 | person whose runs were accepted before I left | those runs to stop at a safe point and never appear as failed tests | the evidence stays honest |
| US-9 | Owner or Admin of a tenant someone left | to be told who left | the member list does not change silently |
| US-10 | person on a phone or tablet | the page, the dialogs and the verification steps reflowed with every statement intact | I understand the consequences anywhere |
| US-11 | attacker with a stolen session, curious member or over-reaching administrator (negative story) | — | I cannot erase an identity with one click or with a session alone, erase or remove someone else, make a last Owner disappear, learn another tenant's projects or members through the ownership checks, withdraw or replay a request, or sign back in to an identity under erasure |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the person* means the signed-in identity acting on its own account; *the page* means N19; *listed membership* means a membership of the person in the active or suspended state; *active Owner* has the meaning of specification 010 D-a; *last active Owner* means the person holds the only active Owner membership of the tenant; *blocking tenant* means a tenant, active or suspended, of which the person is the last active Owner (decision Q4); *the step-up* means the re-authentication of specification 003 RF-8.2 and RF-8.3 (current password and a fresh authenticator code, a recovery code refused, bounded attempts and expiry, one generic failure) bound to the purpose named in each criterion; *the email code* means the one-time code of A-48 bound to the purpose "erasure"; *the erasure process* means the platform's internal, audited, out-of-band process that carries out an erasure request (A-64); *under erasure* means the identity status entered by recording the request (A-27).

### RF-1 · Reaching the page

- RF-1.1 The system shall present the page only to a signed-in identity whose status is active; a person without a session shall be sent to the sign-in screen, and a session that has ended shall be handled by specification 006. *(FR-IDN-024, FR-IDN-123; A-27)*
- RF-1.2 The system shall open the page from the "Account privacy" tab and the "Account privacy options" action of R12, and from the security settings, whether or not a tenant is selected. *(FR-SET-010, FR-SET-013; decision Q6; S-63, S-04)*
- RF-1.3 The "Your membership" block of R12 shall show the person's role in the current tenant, when there is one, with the statement "Leaving a workspace is different from deleting your global identity. Last Owners must transfer ownership first." *(FR-SET-013; R12)*
- RF-1.4 The tenant chooser, the tenant members surface and every tenant screen shall offer neither leaving nor erasure; the only route to either is the page. *(FR-IDN-120; specification 013; specification 010 Q5)*
- RF-1.5 The page shall present, as drawn, the heading "Leaving a workspace is not erasing an identity.", the statement "Personal account / verified global identity erasure. Understand ownership and retained history before continuing.", two separate cards — "Leave <tenant>" and "Erase my global identity" — each with its stated effect, the ownership-checks table, the erasure request panel, and the statements "Minimal history, not forever PII", "Recovery is separate" and "No instant erasure, invented retention duration or other-workspace data deletion is promised." *(FR-IDN-120, FR-IDN-122; N19)*

### RF-2 · Memberships and ownership checks

- RF-2.1 The ownership-checks table shall list one row per listed membership of the person and nothing else, with the columns YOUR MEMBERSHIP (tenant name and the person's tenant role, with "suspended tenant", "deleting tenant" or "suspended membership" where it applies), OWNERSHIP CHECK and REQUIRED NEXT STEP; a revoked membership shall not be listed. *(FR-IDN-123; decision Q6; N19)*
- RF-2.2 The table shall be paged by a configured page size with "Previous" and "Next", and shall state the number of records in the person's scope. *(FR-X-046; N19; S-63)*
- RF-2.3 For each row the system shall derive the check and the next step on the server from the current membership, role, tenant status and Owner count, as follows: last active Owner of an active tenant — "You are the last Owner · Transfer incomplete", next step "Transfer to a verified eligible Owner"; last active Owner of a suspended tenant — "You are the last Owner · Workspace suspended", next step "Held until the workspace is reinstated or deleted"; any other row — "No Owner responsibility" or "Another active Owner remains", next step "Handled through verified erasure process". *(FR-IDN-122, FR-IDN-095; decision Q4; specification 014 RF-5.5)*
- RF-2.4 Each row shall also offer "Leave" when RF-4.1 permits it, or state why the membership cannot be left: "Transfer ownership first" for the last active Owner of an active tenant, "Workspace suspended" for an Owner of a suspended tenant, and the state only for a deleting tenant. *(FR-IDN-121; decision Q6; D-a, D-b)*
- RF-2.5 Where the next step is a transfer, the row shall offer "Review ownership transfer", which opens N23 for that tenant under specification 010 RF-4; where the transfer dialog would find no eligible member (specification 010 RF-4.2), the row shall instead state that a successor must first be invited and complete authenticator enrollment, naming nobody. *(FR-IDN-095, FR-IDN-122; decision Q4)*
- RF-2.6 The table shall show for each row only the tenant name, the person's own role and state, and the check; it shall never show a project, a source, a run, another member, an invitation, a tenant's reason for suspension, or any other customer's data. *(FR-IDN-123; FR-IDN-116)*
- RF-2.7 The page shall state "Only your own membership checks are shown. No project content or other client data is exposed by identity erasure." *(FR-IDN-123; N19)*
- RF-2.8 When the person holds no listed membership, the page shall show neither the leave card nor the table, and shall show the erasure card and panel with the statement that no tenant ownership check applies. *(decision Q6; S-63)*

### RF-3 · The leave card for the current tenant

- RF-3.1 Where a tenant is currently selected, the "Leave <tenant>" card shall address that tenant, with the effect "Remove only this workspace membership and its grants. Your local identity and other memberships remain." *(FR-IDN-121; N19)*
- RF-3.2 While the person is the last active Owner of that tenant, the card shall show "Blocked · last Owner", the statement "Transfer ownership through the verified workspace process before leaving. Removing membership is not global user erasure." and the action "Review ownership transfer" (RF-2.5). *(FR-IDN-121, FR-SET-013; N19)*
- RF-3.3 Otherwise the card shall offer "Leave <tenant>", which opens the dialog of RF-4 for that tenant. *(decision Q3; S-63)*
- RF-3.4 Where no tenant is selected, the card shall not be shown; every listed membership remains leavable from its row (RF-2.4). *(decision Q6)*

### RF-4 · Leaving a tenant

- RF-4.1 The system shall permit the person to leave a listed membership when the tenant is active and the person is not its last active Owner, or when the tenant is suspended and the person's role is Member or Admin; it shall refuse leaving in every other case. *(FR-IDN-121; specification 014 RF-5.3, RF-5.4; D-a, D-b)*
- RF-4.2 When "Leave" or "Leave <tenant>" is activated, the system shall present a confirmation dialog, built on N23's anatomy, headed "Leave <tenant>?", stating that the membership and its project grants in that tenant will be revoked; that revisions, mappings, approvals, runs and evidence the person authored remain with their provenance; that their queued work in the tenant will not run and running work is asked to stop at a bounded checkpoint; that their identity, sign-in, sessions and other memberships are unaffected; and that returning requires a new invitation; with the actions "Leave workspace" and "Cancel". *(FR-IDN-121, FR-IDN-094; decision Q3; S-63)*
- RF-4.3 Where the person's role in the tenant is Owner, the dialog shall additionally require the step-up with the purpose "leave tenant" before "Leave workspace" can be confirmed; a Member or Admin confirms without a step-up. *(FR-IDN-095; decision Q3; specification 010 Q7)*
- RF-4.4 When "Leave workspace" is confirmed, the system shall recheck on the server, at that moment, that the membership is still listed and of the same role, that the tenant's status still permits leaving under RF-4.1, and, for an Owner, under the tenant ownership lock, that another active Owner remains; if any check fails it shall refuse with the stated reason, change nothing and refresh the row. *(FR-IDN-095, FR-X-004, FR-X-005; specification 010 RF-10.2, RF-10.3)*
- RF-4.5 When the rechecks pass, the system shall, as one change, set the membership to revoked, end every project grant of that membership, and advance its permission version. *(FR-IDN-121, FR-IDN-093, FR-IDN-099; specification 009 RF-6, RF-7)*
- RF-4.6 Leaving shall preserve the global identity, its credentials, authenticator, recovery codes, sessions and every other membership and grant of the person. *(FR-IDN-121, FR-IDN-120; specification 009 RF-7.7)*
- RF-4.7 Leaving shall apply to the person's work in that tenant the revocation rule of specification 009 RF-7.1 … RF-7.5. *(FR-IDN-094, FR-IDN-067; decision Q7; A-56)*
- RF-4.8 Leaving shall not end any session; the person's open screens in that tenant shall meet the denied-scope handling of specification 006 RF-5.1 at their next request. *(FR-IDN-064; decision Q3)*
- RF-4.9 When the person left the tenant they are currently in, the system shall take them to the tenant chooser (specification 013) through the guarded switch of specification 016; otherwise the page shall remain, with the row gone and a confirmation that the membership was left. *(decision Q3; S-63)*
- RF-4.10 "Cancel", closing the dialog, or a failed, expired or exhausted step-up shall change nothing. *(specification 003 RF-8.4; N23)*
- RF-4.11 A repeated confirmation of the same leave shall have the effect of one, and a confirmation for a membership already revoked shall change nothing and report that the membership is no longer listed. *(FR-X-020)*

### RF-5 · Erasure request gates

- RF-5.1 The "Erase my global identity" card shall state the effect "Begin a verified erasure process for your own global identity across its memberships.", the marker "High-impact · verification required", and "No Workspace Admin can erase another customer's data or reset another identity's MFA through this flow.", with the action "Review erasure requirements" that brings the person to the erasure request panel. *(FR-IDN-122, FR-IDN-049; N19)*
- RF-5.2 The erasure request panel shall show the identity to verify — the person's own verified address and no other —, the acknowledgements "I understand that leaving a workspace is different from erasing my global identity." and "I have reviewed ownership transfer and the data retained under applicable policy.", the typed confirmation field labelled "Confirmation · type ERASE MY IDENTITY" (in the person's interface language, D-d), and the actions "Keep my account" and "Continue to identity verification". *(FR-IDN-122; N19)*
- RF-5.3 The acknowledgements shall start unticked and the typed confirmation empty every time the panel is shown; nothing shall be pre-filled or remembered between visits. *(FR-IDN-122; FR-X-034; D-c)*
- RF-5.4 While any acknowledgement is unticked, the typed confirmation does not match (D-d), or any blocking tenant exists, "Continue to identity verification" shall be unavailable and the panel shall state "Blocked until all last-Owner transfers are verified and required acknowledgments are complete. This is not a one-click deletion.", naming which gates remain. *(FR-IDN-122; decision Q4; N19)*
- RF-5.5 While a blocking tenant is suspended, the panel shall additionally state that the request is held until that tenant is reinstated or deleted. *(FR-IDN-122; decision Q4; specification 014 RF-5.5)*
- RF-5.6 "Keep my account" shall clear the panel and change nothing. *(N19)*
- RF-5.7 The system shall never record an erasure request from a single action: the gates of RF-5.4 and both proofs of RF-6 shall be required every time. *(FR-IDN-122)*

### RF-6 · Identity verification

- RF-6.1 When "Continue to identity verification" is activated, the system shall re-evaluate every gate of RF-5.4 on the server and, if any fails, shall refuse with the gate named and return to the panel. *(FR-IDN-122, FR-X-004)*
- RF-6.2 When the gates hold, the system shall require the step-up with the purpose "erasure". *(FR-IDN-122; decision Q1; specification 003 RF-8.2, RF-8.3)*
- RF-6.3 When the step-up succeeds, the system shall send the email code to the person's verified address and shall ask for it, stating that a code was sent to that address and when it expires, and offering to resend it within the configured resend limit. *(FR-IDN-122, FR-X-045, FR-X-046; decision Q1; A-48; S-63)*
- RF-6.4 The email code shall be bound to the identity, the purpose "erasure", a configured lifetime and a configured attempt limit; a code sent for another purpose shall not be accepted, a resend shall invalidate the earlier code, and a code shall be consumed when it verifies. *(FR-IDN-122, FR-X-030; A-48)*
- RF-6.5 If the step-up fails, or the email code is wrong, expired or exhausted, or the person leaves the verification, then the system shall record nothing, show one generic failure without saying which proof failed, and return to the panel with the gates cleared (D-c); failures shall count on the sign-in throttles. *(FR-IDN-021; specification 003 RF-8.3, RF-8.4; D-c)*
- RF-6.6 The system shall never write the password, the authenticator code or the email code to browser storage, logs or audit. *(FR-X-030, FR-X-034)*

### RF-7 · Recording the request

- RF-7.1 When the email code verifies, the system shall recheck under each tenant's ownership lock that no blocking tenant exists and, if one appeared since the gates were evaluated, shall refuse with that tenant named, record nothing and return to the panel. *(FR-IDN-095, FR-IDN-122; decision Q4; specification 010 RF-10.2)*
- RF-7.2 When the recheck passes, the system shall, as one indivisible change, record a durable erasure request for the identity, set the identity status to under erasure, and advance the identity's security version. *(FR-IDN-122; decision Q2; A-64, A-27)*
- RF-7.3 The system shall hold at most one erasure request per identity; a second submission for an identity already under erasure shall record nothing further. *(FR-X-020; A-64)*
- RF-7.4 When the request is recorded, every session and refresh family of the identity shall end at its next use, including the session that made the request (specification 006 RF-2.5), and the page shall show the request-received state: that the request was received and handed to the verified erasure process, that a confirmation was sent to the verified address, that the person is now signed out everywhere, that no completion time is promised, and that required records may be retained with minimised identity references; followed by the signed-out state. *(FR-IDN-122, FR-IDN-061, FR-IDN-068; decision Q2; S-63)*
- RF-7.5 When the request is recorded, the system shall close any open recovery case of the identity (specification 005 RF-5.6) and shall hand the request to the erasure process. *(FR-IDN-046; decision Q2; A-48, A-64)*
- RF-7.6 While the identity is under erasure, sign-in shall fail generically (specification 001 RF-2.7), password reset and controlled recovery shall create and send nothing (specifications 004 RF-2.6, 005 RF-4.6), and no enrollment, workspace creation or invitation acceptance shall start. *(FR-IDN-020, FR-IDN-040; decision Q2; A-27)*
- RF-7.7 The product shall offer no way to withdraw, pause or expedite a recorded request; what happens after recording belongs to the erasure process. *(FR-IDN-122; decision Q2; A-64)*
- RF-7.8 While the identity is under erasure, every derivation of authority for any of its memberships shall yield nothing (specification 009 RF-3.6, LC-29), and the identity shall not count as an active Owner of any tenant (specification 010 D-a). *(FR-IDN-095; decision Q7)*
- RF-7.9 When the identity becomes under erasure, the system shall apply the revocation rule of specification 009 RF-7.1 … RF-7.5 to the person's work in every tenant, shall make every pending invitation addressed to the identity unusable (specification 008 LC-28), and shall leave the memberships, grants and the invitations the person issued as they are for the erasure process. *(FR-IDN-094, FR-IDN-067; decision Q7; A-56)*

### RF-8 · Messages

- RF-8.1 When a person leaves a tenant, the system shall send them an email naming the tenant and stating that their membership and project grants there were revoked, that their identity and other memberships are unaffected, and that returning requires a new invitation. *(FR-X-045; decision Q5; A-30, S-14)*
- RF-8.2 When a person leaves a tenant, the system shall send each active Owner and Admin of that tenant, other than the person, a notice naming the person's display name and the tenant, with no link and no project content. *(FR-X-045; decision Q5; A-30, S-14)*
- RF-8.3 When an erasure request is recorded, the system shall send to the verified address a confirmation stating that the request was received, that every session was ended, that no completion time is promised, that required audit and legal records may be retained with minimised identity references, and the erasure process contact; it shall contain no link that acts on the account. *(FR-IDN-122, FR-X-045; decision Q2, Q5; A-30, S-14)*
- RF-8.4 No message about an erasure request shall be sent to any tenant, Owner, Admin or member. *(FR-IDN-123; decision Q5)*
- RF-8.5 No message shall promise a completion date, an erasure of backups or a retention duration. *(FR-IDN-122, FR-IDN-125)*

### RF-9 · What nobody can do

- RF-9.1 The system shall offer no way to leave a tenant on another person's behalf, nor to request, record or withdraw the erasure of another identity, for any role, policy or administrator; tenant administrators remove memberships only through specification 010. *(FR-IDN-049; specification 010 RF-11.2)*
- RF-9.2 Neither leaving nor an erasure request shall change any other identity's password, authenticator, recovery codes, sessions or memberships, nor any tenant's role assignments other than the person's own membership. *(FR-IDN-049, FR-IDN-121)*
- RF-9.3 The page shall state "Erasure is not account recovery. Password reset and workspace administration do not disable MFA or bypass verification." *(FR-IDN-046, FR-IDN-049; N19)*
- RF-9.4 Neither operation shall delete, rewrite or detach authored revisions, mappings, approvals, runs, verdicts or evidence; leaving retains them with provenance, and erasure leaves their treatment to the erasure process under lawful retention. *(FR-IDN-121, FR-IDN-122, FR-PRJ-051)*

### RF-10 · Audit and secrecy

- RF-10.1 The system shall audit every leave with actor, tenant, membership, previous and new state and time, and every refused leave with its reason. *(FR-IDN-099, FR-X-044)*
- RF-10.2 The system shall audit every erasure gate refusal, every verification outcome (without any secret), and the recording of the request with the identity, time and the proofs that passed. *(FR-X-044, FR-X-030; A-64)*
- RF-10.3 Errors shown on the page and in the dialogs shall use safe problem codes with a correlation identifier and never disclose another tenant's or person's data. *(FR-X-033)*

### RF-11 · Responsive and accessible surface

- RF-11.1 Where no designed mobile or tablet counterpart exists for the page, the leave dialog or the verification steps, the system shall present them reflowed under the desktop rules, preserving every statement, gate and action, and converting the ownership-checks table to one record per membership with every cell intact. *(FR-X-051, FR-X-055; S-02)*
- RF-11.2 Every acknowledgement, the typed confirmation field and the code field shall carry a programmatic label and an associated error; every state — blocked, held, refused, received — shall be conveyed by text, not colour alone. *(FR-X-040, FR-X-042)*
- RF-11.3 The page, the dialog and the messages shall be available in EN, ES and FR. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement |
|---|---|
| RNF-1 | **Authority.** Every decision — which rows are listed, whether a membership may be left, whether a tenant blocks erasure — is derived on the server at the moment of the action; the client's view is never authority. *(FR-X-001, FR-X-002, FR-X-004)* |
| RNF-2 | **Last-Owner safety.** No interleaving of leaves, erasure requests, transfers, role edits or removals in one tenant can leave it without an active Owner; the guard and the write are one protected operation under the ownership lock. *(FR-IDN-095)* |
| RNF-3 | **Atomicity.** Leaving (RF-4.5) and recording an erasure request (RF-7.2) each take effect entirely or not at all; a failure leaves the state as it was. |
| RNF-4 | **Idempotence.** Repeated submissions of the same leave or the same erasure request have the effect of one. *(FR-X-020)* |
| RNF-5 | **Session end.** Once a request is recorded, no session of the identity performs any further action; every session ends at its next use. *(FR-IDN-061)* |
| RNF-6 | **Honesty.** No surface or message promises an instant erasure, a completion time, a backup erasure or a retention duration. *(FR-IDN-122, FR-IDN-125)* |
| RNF-7 | **Isolation.** The page and the messages disclose only the person's own memberships, roles, states and checks. *(FR-IDN-123)* |
| RNF-8 | **Secrets.** Passwords, authenticator codes and email codes are never logged, persisted in plain text or kept in browser storage. *(FR-X-030, FR-X-034)* |
| RNF-9 | **Configuration.** The table page size, the step-up expiry and attempt limit, and the email code lifetime, attempt limit and resend limit are validated configuration with no invented default. *(FR-X-046; A-11)* |
| RNF-10 | **Accessibility.** The page, the dialog and the verification steps meet WCAG 2.2 AA, including reflow and 44 px touch targets on responsive surfaces. *(FR-X-040, FR-X-053, FR-X-055)* |
| RNF-11 | **Auditability.** Every leave, every refusal and every erasure request is reconstructible from the audit trail without any secret or customer content. *(FR-X-044)* |
| RNF-12 | **Reliable email.** The leave and erasure messages require the transactional email capability; an erasure request is recorded only when the email code could be sent, and a failure to send is reported as a retryable state, never as a recorded request. *(FR-X-045)* |

---

## 5. Limit cases

| ID | Case | Expected behaviour |
|---|---|---|
| LC-1 | Member leaves the tenant they are currently in | Dialog, confirm; membership revoked; lands on the chooser (RF-4.2, RF-4.5, RF-4.9). |
| LC-2 | Admin leaves a tenant other than the current one | Row gone, confirmation shown, stays on the page (RF-4.9). |
| LC-3 | Sole Owner tries to leave | No "Leave"; "Blocked · last Owner" and "Review ownership transfer" (RF-3.2, RF-2.4). |
| LC-4 | Co-Owner leaves | Step-up, recheck under the lock, membership revoked; the other Owner remains (RF-4.3, RF-4.4). |
| LC-5 | Two co-Owners leave at the same moment | Serialised under the ownership lock; the first succeeds, the second is refused as last Owner (RF-4.4; RNF-2). |
| LC-6 | Co-Owner opens the leave dialog; meanwhile the other Owner is demoted | Confirmation refused with "you are now the last Owner"; nothing changes (RF-4.4). |
| LC-7 | Owner's step-up fails or expires | Dialog closes with nothing changed (RF-4.10). |
| LC-8 | Member of a suspended tenant leaves it from the page | Permitted; membership revoked; tenant leaves their chooser (RF-4.1; specification 014 RF-5.3). |
| LC-9 | Owner of a suspended tenant with a co-Owner tries to leave | Refused; the row states "Workspace suspended" (RF-2.4; D-a). |
| LC-10 | Membership in a deleting tenant | Listed with its state; no "Leave"; never blocks erasure (RF-2.4; D-b). |
| LC-11 | Person with a suspended membership in an active tenant leaves it | Permitted as a Member or Admin, and permitted for an Owner with the Owner step-up of RF-4.3 (the role is Owner even while suspended); the suspended Owner membership is not an active Owner and never blocks erasure (RF-4.1; decision Q4). |
| LC-12 | Leave confirmed twice (double click, retry after lost response) | One effect; the second reports the membership is no longer listed (RF-4.11). |
| LC-13 | Leaving while a run of the person is running in that tenant | Bounded cancellation requested; facts kept; never a failed test (RF-4.7). |
| LC-14 | Person who left is invited again | A new invitation creates a new membership with no grant restored (specification 008 Q5; RF-4.2). |
| LC-15 | Erasure with all gates satisfied and no owned tenant | Step-up, email code, request recorded, signed out everywhere, confirmation email (RF-6, RF-7). |
| LC-16 | Erasure while sole Owner of an active tenant | "Continue" unavailable; the gate named with the transfer route (RF-5.4, RF-2.5). |
| LC-17 | Erasure while sole Owner of a suspended tenant | Held; the panel states it; no action (RF-5.5; specification 014 LC-12). |
| LC-18 | Sole Owner whose tenant has no eligible successor | The row states that a successor must be invited and enroll first; no tenant deletion offered (RF-2.5; decision Q4). |
| LC-19 | Transfer completed in another tab while the panel is open | Gate re-evaluated at "Continue"; the request may proceed (RF-6.1). |
| LC-20 | A co-Owner leaves between the gate check and the email code verifying, making the person last Owner | Refused at recording with the tenant named; nothing recorded (RF-7.1). |
| LC-21 | Typed confirmation with different letter case or a typo | No match; "Continue" unavailable (RF-5.4; D-d). |
| LC-22 | Typed confirmation with surrounding spaces | Accepted after trimming (D-d). |
| LC-23 | Recovery code offered at the step-up | Refused; one generic failure (RF-6.2; specification 003 RF-8.2). |
| LC-24 | Email code wrong, expired, exhausted, or one sent for another purpose | Nothing recorded; generic failure; back to the panel with the gates cleared (RF-6.4, RF-6.5; D-c). |
| LC-25 | Code resent | Earlier code stops working (RF-6.4). |
| LC-26 | Email cannot be sent | Retryable state; nothing recorded (RNF-12). |
| LC-27 | Erasure request submitted twice | One request; the second records nothing (RF-7.3). |
| LC-28 | Another session of the identity open in another browser at recording | Ends at its next use (RF-7.4). |
| LC-29 | Identity under erasure tries to sign in, reset or recover | Generic answers; nothing created or sent (RF-7.6). |
| LC-30 | Open recovery case at recording | Closed with the case-closed message of specification 005 (RF-7.5). |
| LC-31 | Pending invitation to the identity's address at recording | Unusable (RF-7.9; specification 008 LC-28). |
| LC-32 | Invitation the person issued as Admin is pending at recording | Stays pending for the tenant (RF-7.9). |
| LC-33 | Person asks to cancel the request | Not possible in the product; the confirmation email gives the erasure process contact (RF-7.7, RF-8.3). |
| LC-34 | Identity with no listed membership | Erasure section only, no ownership check applies (RF-2.8). |
| LC-35 | Many memberships | Paged table; gates consider every membership, not only the visible page (RF-2.2, RF-5.4). |
| LC-36 | Restricted identity reaches the page's route | Not presented; sign-in is not possible for it (RF-1.1). |
| LC-37 | Administrator looks for a way to erase or "leave" a member | None exists; removal only through specification 010 (RF-9.1). |
| LC-38 | Session expires while the panel is filled | Specification 006 handles it; nothing is remembered, the gates start empty again (RF-5.3). |
| LC-39 | Page opened on a phone | Reflowed; table as records; every statement intact (RF-11.1). |

---

## 6. Out of scope

- **Carrying out the erasure** — minimising or pseudonymising retained attribution, purge order, the final identity record, ending the memberships the request leaves inert, and any reversal: the platform's internal erasure process (A-64); constitution v1.6 binds FR-IDN-122 only up to the recorded request.
- **Withdrawing a recorded request in the product** — decision Q2.
- **Data export** (FR-IDN-124) and **tenant deletion** (FR-IDN-125) — not in the baseline; no tenant deletion is offered to a sole Owner.
- **Ownership transfer itself** — specification 010 (N23); this feature only routes to it.
- **Removing another person's membership** — specification 010.
- **Changing the verified email** — a separate verification flow (FR-SET-011).
- **The tenant chooser and the guarded switch** — specifications 013 and 016; this feature only lands there.
- **Suspension and reinstatement** — specification 014.
- **Backup expiry and restore tombstones** — FR-IDN-125, not in the baseline.

---

## 7. Criteria of completion

1. Every criterion of §3 has at least one passing acceptance test, and every FR of F-IDN-17 — FR-IDN-120, FR-IDN-121, FR-IDN-122 (request gates, independent verification, recording and session revocation), FR-IDN-123 — is cited by at least one of them.
2. Page tests show the two cards, the statements, the entry points from R12 and the security settings, the table with only the person's listed memberships, paging, the no-membership variant, and no leaving or erasure entry anywhere else (RF-1, RF-2, RF-3).
3. Leave tests show the dialog's statements, single confirmation for Members and Admins, the step-up and lock recheck for Owners, refusal for the last Owner and for Owners of suspended tenants, the effects on membership, grants and work in flight, untouched sessions and other memberships, and the landing (RF-4, LC-1 … LC-14).
4. Concurrency tests show that no interleaving of leaves, erasure requests and specification 010 changes leaves a tenant without an active Owner (RNF-2, LC-5, LC-6, LC-20).
5. Erasure tests show the gates, the typed-confirmation rule, both proofs, generic failures, the recheck at recording, the single indivisible recording, the end of every session, the closed recovery case, the generic answers afterwards, the unusable invitations and the absence of withdrawal (RF-5 … RF-7, LC-15 … LC-33).
6. Message tests show the leaver email, the Owner and Admin notice, the erasure confirmation, no message to any tenant about erasure, and no promised time in any of them (RF-8).
7. Isolation, secrecy and audit tests show no project, member or other-customer data on the page or in messages, no secret in logs, storage or audit, and every leave, refusal and request in the audit trail (RF-2.6, RF-6.6, RF-10, RNF-7, RNF-8, RNF-11).
8. Accessibility evidence meets WCAG 2.2 AA on the page, the dialog and the verification steps, including reflow (RF-11, RNF-10).
9. The register items this feature relies on — A-64, A-27, A-48, A-30, A-56, A-11, S-63, S-14, S-02, S-04 — are resolved or their recorded assumption is the one built.
