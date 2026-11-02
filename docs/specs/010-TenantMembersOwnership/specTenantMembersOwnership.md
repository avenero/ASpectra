# Specification 010 — Tenant members and ownership transfer

**Feature** The tenant member list, invitations, tenant role changes, membership management and ownership transfer · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (features F-IDN-12 and F-SET-04 as the subject; F-IDN-10, F-IDN-11, F-IDN-13, F-IDN-15, F-IDN-16, F-IDN-17, F-SET-01, F-SET-05 and F-APR-04 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P3, P6, P12, P13, P14) → `requirements.md` (FR-IDN-001, 002, 012, 015, 023, 030, 049, 067, 080, 081, 084, 090, 091, 092, 093, 094, 095, 097, 098, 099, 110, 116, 120, 121; FR-SET-001, 030, 031, 032, 033, 034; FR-X-001, 002, 020, 021, 022, 030, 033, 034, 040, 042, 043, 044, 045, 046, 052, 053, 055) → the design of record, screen R13 TenantMembersPage `148:503`, the dialog N23 OwnershipTransferDialog `159:32066` inside the D02 atlas `159:32045`, and the authority matrix of D03 `148:4`.

**Scope note.** Constitution v1.3 bound F-IDN-12 to FR-IDN-095 and to the part of FR-IDN-097 that lists members and transfers ownership, and F-SET-04 to the Members and Tenant policy views of FR-SET-030. The design of record draws on R13 the actions "Edit workspace role", "Manage membership", "Invite member" and "Review invitation" and the "Pending invitations" tab; specifications 008 and 009 (27 October 2026) delegate invitation issuance, listing and revocation to this feature; and the product owner decided on 27 October 2026 (decisions Q1, Q8) that this specification covers all of them. The constitution was amended the same day (v1.4): F-IDN-12 binds FR-IDN-097 in full and F-SET-04 binds FR-SET-030 in full. The dialogs the design does not draw are built on N23's anatomy and recorded as S-50.

**Related specifications:** 001 Login (landing by membership state; a suspended or revoked membership is a non-active membership for the landing rule) · 003 TOTP enrollment, recovery codes and factor replacement (the step-up re-authentication reused here, RF-8 there; the TOTP requirement and the enrollment gate for Owners and Admins) · 005 Controlled account recovery (no administrator backdoor on another identity's password or MFA) · 006 Sessions, refresh and expiry (the denied-scope state a member meets when their authority changed; permissions are refreshed, sessions are not revoked) · 007 Logout and session management (no change here ends a session) · 008 Invitation acceptance (what an issued invitation holds, A-52; acceptance and decline, which this feature only observes; the "can't be accepted" state a revoked or expired link shows) · 009 Roles, grants and the authority matrix (the roles, statuses, permission version, revocation semantics and administration authority this feature applies: its Q3–Q6, A-54 … A-57) · 011 Project grants editor (the "Review project grants" route; project grants are never edited here) · 013 Tenant chooser · 014 Tenant suspension state (a suspended or deleting tenant replaces every tenant surface, including this one) · 015 Leave tenant and identity erasure request (leaving one's own tenant; last-Owner transfer before erasure) · the tenant policy specification of F-SET-05 (the "Workspace policy" tab).

This document states **what** tenant member administration, invitations and ownership transfer do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

A tenant is one customer organisation. People belong to it through a **tenant membership** that carries exactly one of three **tenant roles** — Owner, Admin, Member — and a **membership state** — active, suspended, revoked — with a **permission version** that moves on every change (specification 009). Tenant membership is **not** project access: an Owner or Admin has no implicit read of any project; project access is a separate explicit grant per project, administered elsewhere (specification 011). The identity behind a membership is **global**: its password, its authenticator and its other memberships are never the tenant's to touch.

Owners and Admins administer the tenant's people on one screen, R13. They see who belongs, with which role, in which state and under which TOTP requirement; they **invite** a person by address with a role and, optionally, one project's grants; they review, resend or revoke a pending invitation; they change a tenant role; they suspend, reinstate or remove a membership; and an Owner hands ownership to another verified member through the dialog N23. Every one of these changes is a consequential, versioned, audited decision, and the ones that make or unmake an Owner change who controls the tenant and therefore require a fresh proof of the actor, exactly as replacing an authenticator does.

One rule stands above all of them: the **last active Owner is protected**. A tenant can never be left with nobody able to administer it, and no administrator — however delegated — can evade that guard, race it, or count-then-write around it.

### 1.2 Objective

Let the people who own and administer a tenant see and change its membership safely, while:

- keeping tenant administration and project content access two different things, and saying so on the screen;
- never letting a change remove, demote or suspend the last active Owner, even under concurrent attempts;
- requiring a fresh password-plus-authenticator proof for any change that creates or removes an Owner, and rechecking everything that could have changed at the moment of confirmation;
- confining an Admin to delegated scope — Members and Admins, never Owners — and nobody to their own row;
- issuing invitations that carry exactly the scope the inviter may give — never Owner, never more than one project — one pending per address, revocable and resendable, and nothing that joins a person by email domain;
- making every change versioned, audited and notified, effective at the affected person's next request, without ending their sessions or touching their identity, their other memberships or their work already accepted by the platform;
- exposing no global identity control: no reset of another person's password or MFA, no erasure.

### 1.3 Why it matters

Membership administration is where a tenant's authority is handed around. A screen that lets a delegated Admin appoint an Owner, that lets two Owners demote each other at once, that strips the last Owner, or that lets an invitation carry a role the inviter never held, converts a routine settings page into a takeover or a lockout. The opposite failure is quieter: an administrator who believes tenant role means project access, or who looks for a way to "fix" a member's MFA, has been taught the wrong model by the surface. The feature must be strict where authority moves, explicit about what it does and does not do, and boring everywhere else.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-50** (R13 dialogs and states) is new; notes appended to **S-04** (destinations of the remaining buttons, now D02 `439:4460`), **S-14** (email templates) and **S-02** (responsive coverage); **S-49** (the Admin view of R13, decided by specification 009) is applied. Page "ASpectra Architecture" (`0:1`) → spec block `122:2` → §14: notes appended to **A-30** (notification set) and **A-52** (invitation record); the pre-existing **A-27** (identity statuses), **A-36** (identity-scoped membership read) and **A-54** … **A-57** (specification 009's membership decisions) are cited where they apply.

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | The member list with search, filters and statements; ownership transfer (N23); editing a member's tenant role; managing a membership — suspend, reinstate, remove — through compact confirmation dialogs drawn in D02 `419:2645` on N23's dialog anatomy (S-50); and, by Q8, invitations. The "Workspace policy" tab routes to its own feature (F-SET-05). Constitution v1.4 records the amendment. |
| Q2 | What ownership transfer does | A transfer is a step-down: in one indivisible change the chosen member becomes Owner and the acting Owner becomes Admin; both memberships get a new permission version; both people are notified; no project grant changes. Adding a co-Owner without stepping down is "Edit tenant role → Owner" (Q5), not a transfer. |
| Q3 | Who is an eligible new Owner | A member whose membership is active, whose identity is active with a verified email, and who already has an enrolled authenticator; never the actor. The picker lists only such members and states the rule in general terms, naming nobody who is missing or why. The same eligibility applies to promoting a member to Owner. |
| Q4 | Re-authentication and recheck for a transfer | The step-up of specification 003 RF-8: the acting Owner's current password and a fresh authenticator code, always; bounded attempts and expiry; one generic failure counted on the sign-in throttles; a recovery code refused. At confirmation, under the tenant ownership lock, the system rechecks that the actor is still an active Owner, that the chosen member is still eligible, and that the result leaves at least one active Owner; any change since the dialog opened refuses the transfer with a stated reason and nothing changes. |
| Q5 | Who may do what | An Owner may transfer their ownership, edit any role — including promoting to Owner or demoting another Owner when at least one active Owner remains — and manage any membership. An Admin may change roles between Member and Admin and manage memberships of Members and Admins, but never touches an Owner's row and never assigns Owner. Nobody edits or manages their own row: an Owner's own row offers "Transfer ownership" only; leaving is specification 015. A Member cannot open the page and receives the denied-scope state. Identical to specification 009 Q6 (A-57) and its Admin view of R13 (S-49). |
| Q6 | "Manage membership" and the effect of each change | The dialog offers suspend (membership suspended, grants kept but inert, reversible), reinstate (back to active with the same grants) and remove (membership revoked, project grants revoked, authored evidence and provenance retained; identity, other memberships and credentials untouched). Every change and every role edit bumps the membership's permission version, is audited with actor, previous and new value, and notifies the affected member by email. The member's sessions are not revoked: their open screens meet the denied-scope state of specification 006 at the next request, and in-flight work is cancelled at bounded checkpoints. The last-Owner guard blocks suspending or removing the last active Owner. The meanings of suspended and revoked are those of specification 009 Q3 (A-54), the version rule its Q4 (A-55) and the in-flight rule its Q5 (A-56). |
| Q7 | When the step-up is required | For any change that creates or removes an Owner: promoting to Owner, demoting an Owner, suspending or removing an Owner, and the transfer itself; all share the step-up of Q4 and the recheck under the ownership lock. Changes between Member and Admin, and suspending, reinstating or removing a Member or Admin, need only an explicit confirmation dialog stating the effect. |
| Q8 | Invitations | Issuing an invitation ("Invite member"), the "Pending invitations" view, "Review invitation", resending and revoking a pending invitation, the invitation message and the inviter notification belong to this specification, as specifications 008 and 009 expect, within the record of A-52: the invited address, the tenant, one tenant role of Admin or Member, at most one project with an explicit subset of Author / Reviewer / Viewer and an explicit `canRun`, the issuing identity, the time of issue, a configured expiry, and the states pending → accepted / declined (specification 008) / revoked / expired. |
| Q9 | Issuing and reviewing | One pending invitation per address: issuing to an address that already holds an active or suspended membership of the tenant is refused with that reason; issuing while an invitation to that address is pending is refused and points to "Review invitation". A revoked membership does not block a new invitation. Review shows the issued scope, who issued it and when it expires, and offers "Resend invitation" (a fresh secret and a fresh expiry; the earlier link stops working) and "Revoke invitation" (final; the link stops working; the row leaves the list). Nothing about an address reveals whether an identity exists for it. |
| Q10 | Messages | Issuing and resending send the invitation message to the invited address: the tenant name, the inviter's display name, the role and, if any, the project and its grants, the link, the expiry, and that only the local identity holding this address can accept. When an invitation is accepted or declined, the inviter receives a short notification naming the tenant and the outcome, with no link and no project content. Revocation and expiry send nothing to anyone. |

Four rules below are derived from the constitution and earlier specifications rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** *Active Owner* means a membership with the Owner role in the active state whose identity is active (A-27). A suspended Owner, or an Owner whose identity is restricted or under erasure, does not count toward "at least one active Owner".
- **D-b** While the tenant is suspended or deleting, no surface of this feature is reachable and no change is accepted: the suspended-tenant screen of specification 014 replaces every tenant surface (FR-IDN-116). Pending invitations wait, as specification 008 D-d states. Administration resumes when the tenant is active again.
- **D-c** A removed membership is no longer listed; its history is visible only in the audit trail. A removed person returns only through a new invitation, with no grant restored (specification 008 Q5).
- **D-d** The row "Invitation pending" that R13 draws in the Members table is a pending invitation, not a membership: it appears in the Members view with the invited role and the action "Review invitation", and in the Pending invitations tab with its details (the separate views of FR-SET-030 exist; the Members view additionally lists the pending rows as R13 draws them); it never carries a membership state, never counts as a member for any guard, and leaves both views when accepted, declined, revoked or expired. Invitation expiry is validated configuration.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | tenant Owner | to see everyone in my tenant with their role, state and TOTP requirement, and search and filter them | I know who can do what before I change anything |
| US-2 | tenant Owner | to hand ownership to another verified member and step down to Admin in one confirmed, re-authenticated step | the tenant never has a moment without an Owner and nobody but me can do it |
| US-3 | tenant Owner | to promote a member to Admin or Owner, or demote an Admin, with the effect stated before I confirm | delegation is explicit and reversible |
| US-4 | tenant Admin | to manage Members and other Admins within my delegated scope, and to be told plainly that Owner rows are not mine to touch | I can run day-to-day administration without being able to seize the tenant |
| US-5 | Owner or Admin | to invite a colleague by address with a role and, if I choose, one project's grants, and to see the invitation pending until they act | joining is explicit and I keep sight of who is expected |
| US-6 | Owner or Admin | to resend an invitation that got lost or revoke one that should not stand | a stale link in a mailbox never becomes a surprise membership |
| US-7 | Owner or Admin | to suspend a member temporarily and reinstate them later with their grants intact | I can react to a concern without destroying their setup |
| US-8 | Owner or Admin | to remove a member and be told that their authored evidence stays and their identity is untouched | the tenant's history is intact and I am not deleting a person |
| US-9 | member whose role or membership changed | to be notified and to see my new scope at my next request, without being signed out of my other tenants | I understand what happened and lose nothing elsewhere |
| US-10 | member with an accepted Run in progress when my membership is suspended | the platform to stop my work at a safe point and keep the facts | nothing is presented as undone or as a failed test |
| US-11 | sole Owner | to be prevented from removing or demoting myself and pointed to ownership transfer | I cannot lock my own organisation out by accident |
| US-12 | Owner or Admin on a phone or tablet | the member list, the pending invitations and the dialogs reflowed with every statement, guard and action intact | I can administer anywhere without losing a safeguard |
| US-13 | attacker or over-reaching administrator (negative story) | — | I cannot appoint an Owner as an Admin, invite an Owner or a scope I may not give, remove or demote the last Owner by racing another change, change my own role, learn whether an address has an account, reach a member's password or MFA, read a project through tenant administration, or change a membership without a versioned, audited, notified record |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the page* means R13; *the actor* means the signed-in Owner or Admin acting on it; *the target* means the member or invitation whose row is acted on; *active Owner* has the meaning of D-a; *the step-up* means the re-authentication of specification 003 RF-8 (current password and a fresh authenticator code) with the purpose of tenant ownership; *Owner-affecting change* means a change that creates or removes an Owner (Q7); *the ownership lock* means the serialisation of every Owner-affecting change of one tenant required by FR-IDN-095; *invitation* means the record of A-52 as specification 008 defines it.

### RF-1 · The page, its entries and its authority

One screen for the tenant's people, reachable only by those who administer it. *Why:* tenant administration is the one place authority is handed around; it must be visible to exactly the roles that hold it. *(Decision: Q1, Q5; D-b.)*

- RF-1.1 The system shall present the page under the app shell with the breadcrumb "{workspace} / Settings", the heading "People in {tenant}", the subtitle "Manage membership and finite workspace roles, not global identity credentials.", the action "Invite member" (RF-8), the tabs "Members", "Pending invitations" and "Workspace policy", the search field "Search workspace members" with its keyboard shortcut, the role filter ("All roles" by default) and the state filter ("Active" by default), the table of RF-2, the pagination line, the statements and routes of RF-3, and the interface-language switch (EN / ES / FR). *(FR-SET-030, FR-SET-031, FR-IDN-097, FR-X-043; R13)*
- RF-1.2 When "Workspace policy" is activated, the system shall route to the tenant policy surface of its own feature; nothing of this feature changes. *(FR-SET-030; F-SET-05)*
- RF-1.3 The system shall open the page only for an actor whose membership in the current tenant is active with the role Owner or Admin, as derived on the server at every request; a Member, a suspended member or anyone else shall receive the denied-scope state of specification 006 RF-5.2 and shall see no row. *(FR-IDN-091, FR-X-001; decision Q5; P3)*
- RF-1.4 While the tenant is suspended or deleting, the system shall make the page unreachable and shall accept no change of this feature; the suspended-tenant screen applies and pending invitations wait. *(FR-IDN-116; D-b; specification 008 D-d)*
- RF-1.5 The system shall show only memberships and invitations of the current tenant; no other tenant's membership, and no project content, shall be reachable from the page. *(FR-IDN-110 by analogy, FR-IDN-092; P12)*
- RF-1.6 When the language switch is used, the system shall change the display only; nothing selected, typed, verified or pending shall change. *(FR-X-043)*
- RF-1.7 The "Help & shortcuts" entry of the app shell shall open N43 HelpShortcutsPanel (`430:4129`) (S-17); this feature adds no help surface.

### RF-2 · The member list

Every member and every pending invitation, with the facts an administrator needs and the one action they may take. *Why:* FR-SET-031 — member, tenant role, membership state including the TOTP requirement, and the administration action available. *(Decision: Q5, Q6; D-c, D-d.)*

- RF-2.1 The Members view shall list one row per membership of the tenant in the active or suspended state, with the columns MEMBER (display name, marked "you" on the actor's own row), TENANT ROLE (Owner, Admin or Member), MEMBERSHIP (Active or Suspended, with the marker "TOTP required" when the role is Owner or Admin or the tenant policy requires MFA of Members) and ADMINISTRATION (the actions of RF-2.4), and one row per pending invitation with the invited address, the invited role, the state "Invitation pending" and the action "Review invitation" (RF-9). *(FR-IDN-090, FR-IDN-093, FR-IDN-023, FR-SET-031; R13; D-d)*
- RF-2.2 The system shall not list revoked memberships, nor accepted, declined, revoked or expired invitations; their history exists in the audit trail only. *(decision Q6, Q9; D-c, D-d)*
- RF-2.3 The system shall show, per row, nothing about the identity beyond its display name and verified address: no password or factor state, no code count, no other tenant, no project grant. *(FR-IDN-001, FR-IDN-049; P3)*
- RF-2.4 The system shall offer in the ADMINISTRATION column exactly the actions the actor's role permits for the row, derived on the server: on the actor's own row, "Transfer ownership" for an Owner and nothing for an Admin; on another Owner's row, "Edit workspace role" and "Manage membership" for an Owner and nothing for an Admin; on an Admin's or Member's row, "Edit workspace role" and "Manage membership" for an Owner or an Admin; on a pending-invitation row, "Review invitation" for an Owner or an Admin. *(FR-IDN-091, FR-IDN-097; decision Q5; S-49, S-50)*
- RF-2.5 When the search field holds text, the system shall filter rows by display name or address, case-insensitively; when a role or state filter is chosen, by that role or state; the pagination line shall state the count of rows in the current scope and offer "Previous" and "Next". *(FR-SET-031; R13)*
- RF-2.6 The state filter shall offer Active, Suspended, Invitation pending and All; it shall never offer revoked memberships or closed invitations. *(decision Q6, Q9; D-c)*
- RF-2.7 If the current scope holds no row, then the system shall state that no member matches and keep the filters editable. *(FR-X-042; S-50)*
- RF-2.8 The list shall reflect the server's current memberships and invitations at every load; a row whose record changed since the page loaded shall be refused at the moment of any action on it (RF-10.4), never acted on from stale data. *(FR-IDN-095, FR-X-021)*

### RF-3 · Statements and routes

The page teaches the model it enforces. *Why:* FR-SET-032 to FR-SET-034 require the four statements; FR-IDN-092 requires the boundary to be stated on the surface; specification 009 RF-10.1 requires the same statement on this surface.

- RF-3.1 The system shall present the notice "Protect the last Owner — You cannot remove or demote the final verified Owner. Ownership transfer checks current membership again at confirmation." *(FR-SET-032, FR-IDN-095; R13)*
- RF-3.2 The system shall present the card "Membership is not project access" with "Owner and Admin can administer workspace membership. They do not implicitly read every project. Grant Author, Reviewer or Viewer per project, with canRun independent.", the sub-statement "Removing a membership — Revokes this workspace's access; retains authored evidence under policy", the sub-statement "Global identity controls — No reset of another person's password or MFA", and the action "Review project grants". *(FR-SET-033, FR-SET-034, FR-IDN-092, FR-IDN-049; R13; specification 009 RF-10.1)*
- RF-3.3 When "Review project grants" is activated, the system shall route to the project grants surface of specification 011 for the current project; nothing of this feature changes. *(FR-SET-033, FR-IDN-098)*
- RF-3.4 The system shall present the route "Decision history ↗" to the audit trail, scoped to the membership, invitation and ownership events of the tenant that the actor's current authority permits to read. *(FR-IDN-099, FR-X-044; F-APR-04; R13)*
- RF-3.5 The statements shall be present at every width, in every language and in every role variant of the page (S-49); the reflow may not drop them. *(FR-X-051, FR-X-043; specification 009 RF-10.9)*

### RF-4 · The ownership transfer dialog

An Owner chooses a verified successor and proves themselves. *Why:* N23 — ownership is granted to a member, not a global role and not project access, by a verified current Owner. *(Decision: Q2, Q3, Q4.)*

- RF-4.1 When "Transfer ownership" is activated on the actor's own row, the system shall present the dialog "Transfer workspace ownership" with the statement "Choose a verified eligible member. This grants workspace ownership, not a global identity role or implicit project access.", the authority line "Verified current Owner; final confirmation rechecks last-Owner and membership race guards.", the picker "New verified Owner", the re-authentication block "Current reauthentication — Password + factor required by policy" with a current-password field with "show" and an authenticator-code field that accepts a pasted code, the actions "Verify & transfer ownership" and "Keep current Owner". *(FR-IDN-095, FR-IDN-097; N23; decision Q4)*
- RF-4.2 The picker shall list only eligible members — active membership, active identity with a verified email, an enrolled authenticator, not the actor — by display name, and shall state that only verified members with an enrolled authenticator can receive ownership; it shall name nobody who is excluded nor the reason for any individual. *(FR-IDN-012, FR-IDN-023; decision Q3; S-50)*
- RF-4.3 If no member is eligible, then the picker shall say so and "Verify & transfer ownership" shall stay unavailable. *(decision Q3; S-50)*
- RF-4.4 The dialog shall be offered to an Owner only; an Admin's own row offers no transfer, and a transfer submitted by anyone but an active Owner of the tenant shall be refused on the server. *(FR-IDN-091; decision Q5; S-49)*
- RF-4.5 When "Keep current Owner" is activated, or the dialog is closed or left by any other means before confirmation, the system shall change nothing. *(N23)*
- RF-4.6 The dialog shall never show the successor's email verification date, factor details or other memberships, and shall never write the password or the code to browser storage. *(FR-IDN-027, FR-X-034)*

### RF-5 · Transfer confirmation, guards and effects

The step-up, the recheck under the lock, and one indivisible change. *Why:* FR-IDN-095 — recheck current membership at confirmation, lock the ownership row, never count-then-write. *(Decision: Q2, Q4.)*

- RF-5.1 When "Verify & transfer ownership" is activated, the system shall run the step-up for the actor: the password and the authenticator code shall both be verified against the actor's credential and active factor; a recovery code shall be refused as a wrong code; any failure shall be one generic rejection that does not say which input failed, counted on the sign-in throttles and against the step-up's attempt limit; reaching the limit or the step-up's expiry shall close the dialog with nothing changed. *(FR-IDN-020, FR-IDN-021, FR-IDN-025; decision Q4; specification 003 RF-8.2–8.4)*
- RF-5.2 When the step-up succeeds, the system shall, under the ownership lock, recheck that the actor's membership is still active with the role Owner, that the chosen member is still eligible under RF-4.2, and that the tenant is active; if any recheck fails, then the system shall refuse the transfer with a stated reason — the successor is no longer eligible, the actor's own membership changed, or the tenant is no longer active — and shall change nothing. *(FR-IDN-095, FR-IDN-116; decision Q4; N23)*
- RF-5.3 When every recheck passes, the system shall, as one indivisible change: set the chosen member's role to Owner, set the actor's role to Admin, advance the permission version of both memberships, record the actor, the previous and new values and the time, and append the audit entry; at no instant shall the tenant have fewer active Owners than before. *(FR-IDN-093, FR-IDN-095, FR-IDN-099; decision Q2; specification 009 RF-8.6)*
- RF-5.4 The transfer shall change no project grant of either person, create no session, enter no tenant on the successor's behalf, and touch no identity, credential, factor or membership of another tenant. *(FR-IDN-092, FR-IDN-098, FR-IDN-001; N23)*
- RF-5.5 When the transfer completes, the system shall send a notification to the verified address of the new Owner stating that they now own the tenant and who transferred it, and to the former Owner stating that the transfer completed and that they are now an Admin, each with the time and what to do if unexpected. *(FR-X-045; decision Q2; S-14)*
- RF-5.6 When the transfer completes, the actor's current session shall continue; their authority shall be derived as Admin at their next request, and the page shall show the new roles. The successor's open sessions, if any, shall derive Owner authority at their next request. *(FR-X-001, FR-IDN-061; P3)*
- RF-5.7 The transfer shall be a consequential write: a re-submission after a lost answer shall return the recorded outcome and never transfer twice; no transfer shall be queued for later. *(FR-X-020, FR-X-022)*
- RF-5.8 If the actor is the last active Owner, then the transfer shall still be permitted, the successor becoming Owner before the actor steps down within the same change. *(FR-IDN-095; decision Q2)*

### RF-6 · Edit tenant role

A role change is explicit, bounded by the actor's own role, and heavier when an Owner is made or unmade. *Why:* FR-IDN-091 — Admin is delegated administration that cannot evade the last-Owner guard; FR-IDN-090 — exactly three roles. *(Decision: Q1, Q5, Q7.)*

- RF-6.1 When "Edit workspace role" is activated on a row the actor may edit (RF-2.4), the system shall present a dialog, built on the anatomy of N23, showing the member, their current role, a choice of the new role among those the actor may assign — Owner, Admin or Member for an Owner; Admin or Member for an Admin — and a statement of the effect of the chosen role: that tenant role grants no project access, that an Owner or Admin must have an enrolled authenticator before privileged authority, and whether the change requires re-authentication. *(FR-IDN-090, FR-IDN-091, FR-IDN-092, FR-IDN-023; decision Q5; specification 009 RF-8.1, RF-8.2; S-50)*
- RF-6.2 The system shall never offer the actor's own row for a role edit; an Owner wishing to step down shall use the transfer (RF-4) and anyone wishing to leave shall use specification 015. *(decision Q5; specification 009 RF-8.3)*
- RF-6.3 When the new role is Owner, the system shall require that the member is eligible under RF-4.2 and shall require the step-up of RF-5.1 before confirming; when the current role is Owner and the new role is not, the system shall require the step-up and, under the ownership lock, that at least one other active Owner remains. *(FR-IDN-095, FR-IDN-023; decision Q3, Q7)*
- RF-6.4 When the change is between Member and Admin, the system shall require an explicit confirmation only. *(decision Q7)*
- RF-6.5 If the step-up fails or expires, then the dialog shall behave as RF-5.1; if a recheck at confirmation fails — the actor's own role or membership changed, the member's membership changed, the member is no longer eligible for Owner, or the change would leave no active Owner — then the system shall refuse with a stated reason and change nothing. *(FR-IDN-095; decision Q4)*
- RF-6.6 When the change is confirmed, the system shall set the new role, advance the membership's permission version, record actor, previous and new value and time, append the audit entry, and notify the member of their new role; no project grant shall change. *(FR-IDN-093, FR-IDN-098, FR-IDN-099; decision Q6; S-14)*
- RF-6.7 When a member becomes Admin without an enrolled authenticator, the system shall not fail the change for that reason; the enrollment gate of specification 003 shall withhold privileged authority until they enrol. For the Owner role, RF-6.3 requires the enrolled authenticator beforehand. *(FR-IDN-023, FR-IDN-026; specification 003 RF-1.3)*

### RF-7 · Manage membership

Suspend, reinstate or remove — never erase. *Why:* FR-IDN-093 and FR-IDN-094 define the states and the meaning of revocation; FR-IDN-121 keeps the identity and the other memberships intact. *(Decision: Q6, Q7.)*

- RF-7.1 When "Manage membership" is activated on a row the actor may manage (RF-2.4), the system shall present a dialog, built on the anatomy of N23, showing the member, their current state, the actions available — "Suspend membership" for an active membership, "Reinstate membership" for a suspended one, and "Remove from workspace" for either — and, for the chosen action, a statement of its effect before confirmation. *(FR-IDN-093, FR-IDN-097; decision Q6; S-50)*
- RF-7.2 The system shall never offer the actor's own row for management; leaving the tenant is specification 015. *(decision Q5)*
- RF-7.3 When the target is an Owner, the system shall require the step-up of RF-5.1 and, under the ownership lock, that at least one other active Owner remains after the change; otherwise an explicit confirmation only. *(FR-IDN-095; decision Q7)*
- RF-7.4 When a suspension is confirmed, the system shall set the membership to suspended and advance its permission version: the member's commands and subscriptions in this tenant shall be denied from their next request, their project grants shall be kept but inert, their queued work shall be refused at its next checkpoint as NOT EXECUTED and their running work cancelled at a bounded checkpoint where possible and never presented as undone, and the row shall read Suspended. *(FR-IDN-093, FR-IDN-094, FR-IDN-067; decision Q6; specification 009 RF-4.3, RF-7)*
- RF-7.5 When a reinstatement is confirmed, the system shall set the membership back to active with the same project grants and a new permission version; nothing cancelled while suspended shall be resumed or replayed. *(FR-IDN-093; decision Q6; P1)*
- RF-7.6 When a removal is confirmed, the system shall set the membership to revoked and revoke every project grant of that member in this tenant, in one indivisible change with a new permission version; the member's authored evidence, provenance, approvals and run facts shall be retained under policy; their identity, credentials, factor and memberships in other tenants shall be untouched; the row shall no longer be listed. *(FR-IDN-094, FR-IDN-121, FR-SET-034; decision Q6; specification 009 RF-4.4; D-c)*
- RF-7.7 If a recheck at confirmation fails — the actor's own role or membership changed, the target's state already changed, or the change would leave no active Owner — then the system shall refuse with a stated reason and change nothing. *(FR-IDN-095; decision Q4)*
- RF-7.8 When a suspension, reinstatement or removal is confirmed, the system shall record actor, previous and new value and time, append the audit entry, and notify the member by email of the change and of what it means for their access; the notification shall name the tenant and shall carry no project content. *(FR-IDN-099, FR-X-045; decision Q6; S-14)*
- RF-7.9 The system shall not end any session of the affected identity because of a suspension, reinstatement or removal; their open screens in this tenant shall meet the denied-scope state of specification 006 RF-5.1 at their next request, and their sessions in other tenants shall be unaffected. *(FR-IDN-061, FR-IDN-064; decision Q6)*
- RF-7.10 A removed membership shall not be restorable from this page; the person returns only through a new invitation, and no former grant is restored. *(FR-IDN-080; D-c; specification 008 Q5)*

### RF-8 · Inviting a member

An invitation is the only way into a tenant, and it carries exactly what the inviter may give. *Why:* FR-IDN-080 and FR-IDN-084 — bound to one address, one tenant, one role, no self-escalation; specification 008 presumes the record this feature issues. *(Decision: Q8, Q9, Q10.)*

- RF-8.1 When "Invite member" is activated, the system shall present a dialog, built on the anatomy of N23, with: an address field; a tenant-role choice among Admin and Member only; an optional project choice limited to one project of the tenant with the explicit grants Author, Reviewer, Viewer as independent choices and `canRun` as a separate choice, each unticked by default; the statement that membership is not project access and that `canRun` is independent of the project roles; the statement that the invitation can be accepted only by the local identity holding the invited address and expires after the configured time; and the actions "Send invitation" and "Cancel". *(FR-IDN-080, FR-IDN-081, FR-IDN-084, FR-IDN-090, FR-IDN-092; decision Q8; A-52; S-50)*
- RF-8.2 The dialog shall offer only what the actor may set: an Owner or an Admin may issue Admin or Member with any project grants of RF-8.1; the Owner role shall never be offered or accepted, and a project role or `canRun` shall never be inferred. *(FR-IDN-084; specification 009 RF-8.5)*
- RF-8.3 If the address is empty or malformed, or no role is chosen, then the system shall identify the field with a message and shall not submit. *(FR-X-042)*
- RF-8.4 When "Send invitation" is activated, the system shall normalise the address (case-insensitively, ignoring surrounding whitespace) and, under the tenant's current state, refuse with a stated reason when: the address already belongs to an active or suspended membership of this tenant; an invitation to this address is already pending, pointing to "Review invitation"; or the tenant is not active. A revoked membership of that address shall not block the invitation. *(FR-IDN-093; decision Q9; specification 008 Q5)*
- RF-8.5 When no refusal applies, the system shall, as one indivisible change, create the invitation in state *pending* with the normalised address, the tenant, the chosen role, the chosen project and grants (or none), the actor as issuer, the time of issue and the configured expiry; store its secret only in a non-reversible form; send the invitation message of RF-13.4 to the invited address; and append the audit entry. *(FR-IDN-080, FR-X-030; decision Q8, Q10; A-52)*
- RF-8.6 The invitation shall create no membership, no grant and no identity; it shall grant nothing until accepted by the invited identity under specification 008, and nothing about issuing shall disclose whether an identity exists for the address. *(FR-IDN-012, FR-IDN-015; decision Q9; P3)*
- RF-8.7 When the invitation is created, its row shall appear in the Members view as "Invitation pending" and in the Pending invitations tab. *(FR-SET-030; D-d)*

### RF-9 · Pending invitations — list, review, resend, revoke, expiry

A pending invitation is a live secret in someone's mailbox; the tenant keeps sight of it until it is settled. *Why:* FR-IDN-083 — expired, revoked or used invitations are not acceptable; FR-IDN-097 — pending invitations are listed and administered. *(Decision: Q9, Q10; D-d.)*

- RF-9.1 The Pending invitations tab shall list one row per invitation of the tenant in state *pending*: the invited address, the invited tenant role, the project and grants (or "membership only"), the issuer, the time of issue, the expiry, and the action "Review invitation". *(FR-SET-030, FR-IDN-097; R13; S-50)*
- RF-9.2 When "Review invitation" is activated, the system shall present a panel, built on the anatomy of N23, showing the issued scope element by element — address, role, project, project roles, `canRun` — the issuer, the time of issue and the expiry, and offering "Resend invitation", "Revoke invitation" and "Close"; it shall offer no control that changes the scope. *(FR-IDN-084; decision Q9; S-50)*
- RF-9.3 When "Resend invitation" is confirmed, the system shall issue a fresh secret and a fresh expiry for the same invitation, so that the earlier link stops working, send the invitation message again to the invited address, and append the audit entry; the scope shall be unchanged. *(FR-IDN-080, FR-IDN-083; decision Q9)*
- RF-9.4 When "Revoke invitation" is confirmed, the system shall move the invitation to *revoked*, so that its link shows the "can't be accepted" state of specification 008 from then on, append the audit entry, send no message, and remove the row from both views. *(FR-IDN-083; decision Q9, Q10; specification 008 RF-5.6)*
- RF-9.5 When the configured expiry elapses, the system shall move the invitation to *expired*, send no message, and remove the row from both views; the expiry shall be validated configuration with no default. *(FR-IDN-083, FR-X-046; decision Q10; D-d; P13)*
- RF-9.6 When an invitation is accepted or declined (specification 008), the system shall remove the row from both views, show the new membership in the Members view when accepted, and send the inviter the notification of RF-13.4. *(FR-IDN-097; decision Q10; specification 008 RF-6.9, RF-7.2)*
- RF-9.7 Resend and revoke shall be offered to an Owner or an Admin; a revocation racing with an acceptance shall yield exactly one transition, as specification 008 D-c states. *(decision Q5; specification 008 RF-6.8)*
- RF-9.8 The inviter's later loss of authority shall not void a pending invitation; only revocation, expiry, acceptance or decline settles it. *(specification 008 Q4)*

### RF-10 · Last-Owner protection and concurrency

No sequence of actions, by one person or several at once, leaves the tenant without an active Owner. *Why:* FR-IDN-095 — the lock, the guard and the ban on count-then-write. *(Decision: Q4, Q5; D-a.)*

- RF-10.1 The system shall refuse any change — role edit, suspension, removal, or a change caused elsewhere — that would leave the tenant with no active Owner, and shall say so with the reason "the final verified Owner cannot be removed, demoted or suspended" and the route to ownership transfer. *(FR-IDN-095, FR-SET-032; specification 009 RF-8.7)*
- RF-10.2 The system shall serialise every Owner-affecting change of one tenant under the ownership lock, so that two concurrent changes are evaluated one after the other against the state the first one produced; the second shall be refused if the guard no longer holds. *(FR-IDN-095)*
- RF-10.3 The system shall never decide the guard by counting Owners and then writing; the count and the write shall be one protected operation. *(FR-IDN-095)*
- RF-10.4 Every confirmation shall carry the permission version the dialog was opened against for the target and for the actor, or the invitation's state for an invitation; if either moved, then the confirmation shall be refused with a stated reason and the page refreshed. *(FR-IDN-093, FR-X-021; specification 009 RF-6.3)*
- RF-10.5 An Admin shall never be able to reach, by any route, an action that creates or removes an Owner, nor issue an invitation carrying Owner; the guard applies to Owners' actions and the scope rule to Admins', and neither replaces the other. *(FR-IDN-091; decision Q5; specification 009 RF-8.2)*
- RF-10.6 The actor's own role shall be derived on the server at every confirmation; an actor demoted, suspended or removed since the page loaded shall have every pending action refused. *(FR-X-001; P3)*

### RF-11 · What tenant administration never does

The boundaries the screen states are the boundaries the system enforces. *Why:* FR-IDN-092 and FR-IDN-049 are what make a global identity and a tenant safe together.

- RF-11.1 The system shall grant no project content access through a tenant role, a role change, an invitation's tenant role or a transfer; every project grant stays a separate explicit record administered by specification 011, and an invitation's project grants become records only at acceptance. *(FR-IDN-092, FR-IDN-098)*
- RF-11.2 The system shall offer no control over another person's password, authenticator, recovery codes, sessions, verified address or erasure from this feature, and the page shall say so. *(FR-IDN-049, FR-SET-034; specification 005 RF-9.2)*
- RF-11.3 The system shall let nobody assign, raise or remove their own tenant role, manage their own membership, or invite their own address from this feature. *(FR-IDN-084; decision Q5; specification 009 RF-8.3)*
- RF-11.4 The system shall create no membership here; only acceptance under specification 008 does. *(FR-IDN-080)*
- RF-11.5 The system shall never cancel, duplicate, replay or re-verdict an accepted Run because of a membership change; a suspension or removal may stop queued or in-flight work of that member at bounded checkpoints, and the facts stay as recorded. *(FR-IDN-067, FR-IDN-094; P1, P8)*
- RF-11.6 The system shall never change the tenant's own state, its policy, its other members' grants, or any other tenant's membership because of a change made here. *(FR-IDN-116; P12)*
- RF-11.7 The system shall never join an identity to the tenant by matching an email domain or by a sign-in, and never disclose, through issuing or reviewing, whether an identity exists for an address. *(FR-IDN-015, FR-IDN-020)*

### RF-12 · Responsive surface

R13 is an app-shell screen with no designed mobile counterpart. *Why:* FR-X-055 — AA reflow on every critical-journey surface; FR-X-052 — mobile navigation follows the M04 pattern.

- RF-12.1 The system shall present the page, its two tables as records at narrow widths, its dialogs and panels and its states reflowed at mobile and tablet widths that meet WCAG 2.2 AA reflow, with every statement, guard, action and route of RF-1 … RF-9 preserved in meaning; mobile navigation shall follow the M04 pattern with destinations that still follow current server-derived grants. *(FR-X-051, FR-X-052, FR-X-055; S-02)*
- RF-12.2 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus; the tables shall be operable by keyboard with the action cell reachable in reading order. *(FR-X-040, FR-X-042, FR-X-053)*
- RF-12.3 R13, its dialogs and panels shall follow the D01 responsive policy (`414:2645`): one record card per member row, dialogs as sheets at or below 768 px. *(FR-X-050; S-02)*

### RF-13 · Audit, messages and secrecy

Every change is reconstructable; nothing secret is ever written. *Why:* FR-IDN-099 — versioned and auditable; FR-X-044 — append-only redacted audit with actor and decision. *(Decision: Q10.)*

- RF-13.1 The system shall append to the audit log: each ownership transfer (actor, successor, previous and new roles, permission versions); each role change; each suspension, reinstatement and removal; each invitation issued, resent, revoked or expired, with its scope; each refused confirmation or refused issue with its reason; each step-up succeeded, failed or voided for this feature; each message sent and each delivery failure. Each entry identifies the tenant, the actor, the target, the time and the calling context with a correlation id. *(FR-IDN-099, FR-X-044; specification 009 RF-11.1)*
- RF-13.2 No audit entry, log line, problem detail, URL, message or analytics event shall contain a password, an authenticator code, an invitation secret or any project content. *(FR-X-030, FR-X-033)*
- RF-13.3 Every refusal of this feature shall be a safe problem with a correlation id that states the reason an administrator needs — stale version, guard, eligibility, scope, existing membership, pending invitation — and never a secret or another identity's security facts. *(FR-X-033)*
- RF-13.4 The messages of this feature shall go through the transactional email channel in EN, ES and FR, in the language of the request that caused them, with wording that is a design deliverable (S-14): the invitation message to the invited address (tenant name, the inviter's display name, the role, the project and grants when any, the link, the expiry, and that only the local identity holding this address can accept); the notification to the inviter when an invitation is accepted or declined (tenant and outcome, no link, no project content); ownership received and ownership transferred (RF-5.5); tenant role changed (RF-6.6); membership suspended or reinstated and membership removed (RF-7.8). Revocation and expiry of an invitation send nothing. A delivery failure shall never revert a change and shall be recorded. *(FR-X-045; decision Q10; S-14)*
- RF-13.5 The system shall write nothing of the dialogs — selections, addresses, passwords, codes — to browser storage. *(FR-X-034)*
- RF-13.6 Every change shall confirm persistence before the page reports success; the page shall show the committed state. *(FR-SET-001; F-SET-01)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Authority at every effect.** Who may open the page, which action each row offers and whether a confirmation or an issue is accepted are derived on the server at each request; the browser's rendering is never the decision. | FR-X-001, FR-IDN-091; P3 |
| RNF-2 | **Last-Owner invariant.** At no committed instant does an active tenant have zero active Owners, whatever the sequence or concurrency of changes. | FR-IDN-095 |
| RNF-3 | **Atomicity.** A transfer, a role change, a suspension, a reinstatement, a removal, an invitation issue, resend or revocation are each one indivisible change with its version or state and its audit entry; no observer sees a half-applied state. | FR-IDN-080, FR-IDN-093, FR-IDN-099 |
| RNF-4 | **Serialisation.** Owner-affecting changes of one tenant are serialised under the ownership lock; count-then-write never occurs. | FR-IDN-095 |
| RNF-5 | **Fresh proof for ownership.** Every Owner-affecting change requires the step-up (password and authenticator code) of the actor, bounded in attempts and time, with a generic failure. | FR-IDN-023, FR-IDN-095; decision Q4, Q7 |
| RNF-6 | **Permission latency.** A changed membership takes effect at the affected person's next request; no stale authority is honoured after the change commits. | FR-X-001, FR-IDN-061 |
| RNF-7 | **Identity integrity.** No change here touches an identity's credential, factor, codes, sessions, verified address or memberships in other tenants. | FR-IDN-001, FR-IDN-049, FR-IDN-121 |
| RNF-8 | **Tenant isolation.** The page reads and writes memberships and invitations of the current tenant only; no cross-tenant row is reachable. | FR-X-010, FR-IDN-110; P12 |
| RNF-9 | **Exact invitation scope.** An invitation holds Admin or Member, at most one project, and only grants the inviter may set; its secret exists at rest only in non-reversible form and only one invitation per address is pending. | FR-IDN-080, FR-IDN-084; decision Q8, Q9 |
| RNF-10 | **No disclosure.** Issuing, resending or reviewing an invitation never reveals whether an identity exists for an address; the member list reveals nothing of an identity beyond display name and address. | FR-IDN-020, FR-IDN-049 |
| RNF-11 | **Fail closed.** A stale version, a failed recheck, an unknown target, a closed invitation, an absent configuration value or an absent policy refuses the change; nothing is assumed. | P6 |
| RNF-12 | **Configured limits.** Step-up attempts and expiry (inherited from specification 003), invitation expiry, page size and search bounds are validated configuration with no default. | FR-X-046; P13 |
| RNF-13 | **Idempotency.** A re-submitted confirmation or issue returns the recorded outcome; no change is applied twice and no invitation is issued twice. | FR-X-020, FR-X-022 |
| RNF-14 | **Email as prerequisite.** The messages of RF-13.4 depend on the transactional email channel; its absence is an operational failure, never a reverted change. | FR-X-045 |
| RNF-15 | **Accessibility.** The page, tables, dialogs, panels and states meet WCAG 2.2 AA with automated and manual assistive-technology evidence. | FR-X-040, FR-X-042 |
| RNF-16 | **Language.** EN / ES / FR, display only. | FR-X-043 |
| RNF-17 | **Reflow.** Mobile and tablet reflows preserve every statement, guard, action and route. | FR-X-051, FR-X-055 |
| RNF-18 | **Verification.** Concurrent last-Owner and grant transitions, authorisation races and invitation concurrency are verified as area A requires. | requirements A.12; P9 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Sole Owner transfers to the only eligible member | Allowed: successor becomes Owner and the actor Admin in one change; both notified (RF-5.3, RF-5.8). |
| LC-2 | Sole Owner looks for a way to demote or remove themselves | Own row offers "Transfer ownership" only; the notice points to transfer (RF-6.2, RF-7.2, RF-10.1). |
| LC-3 | Two active Owners; one demotes the other to Admin | Step-up, lock, one active Owner remains → allowed (RF-6.3). |
| LC-4 | Two active Owners demote each other at the same instant | Serialised under the lock: the first succeeds, the second is refused because it would leave no active Owner (RF-10.2, RF-10.1). |
| LC-5 | Successor's membership is suspended between opening N23 and confirming | Recheck fails; refused with the reason; nothing changes (RF-5.2). |
| LC-6 | Successor replaces their authenticator while N23 is open | Still eligible (an enrolled factor exists); transfer proceeds (RF-4.2, RF-5.2). |
| LC-7 | Member without an enrolled authenticator | Absent from the picker; cannot be promoted to Owner; can be made Admin and is gated at tenant entry (RF-4.2, RF-6.3, RF-6.7). |
| LC-8 | Actor tries to choose themselves as successor | Not in the picker (RF-4.2). |
| LC-9 | Wrong authenticator code on the transfer, then attempts exhausted | Generic rejections counted on the sign-in throttles; at the limit the dialog closes with nothing changed (RF-5.1). |
| LC-10 | Recovery code typed as the step-up code | Refused as a wrong code (RF-5.1; specification 003 Q4). |
| LC-11 | New Owner is signed in elsewhere when the transfer completes | Their next request derives Owner authority; no new sign-in needed (RF-5.6). |
| LC-12 | Former Owner keeps using the page after stepping down | Their next request derives Admin authority; Owner rows show no action for them (RF-5.6, RF-2.4). |
| LC-13 | Admin opens an Owner's row | No action rendered; a direct submission is refused on the server (RF-2.4, RF-10.5; S-49). |
| LC-14 | Admin promotes a Member to Admin | Confirmation only; no step-up; member notified (RF-6.4, RF-6.6). |
| LC-15 | Owner promotes a Member to Owner | Eligibility required; step-up required; member notified (RF-6.3). |
| LC-16 | Member suspended while a Run they launched is in progress | Commands and subscriptions denied from the next request; the Run is stopped at a bounded checkpoint where possible and its facts retained; never a failed test (RF-7.4, RF-11.5). |
| LC-17 | Suspended member reinstated | Active again with the same grants and a new permission version; nothing cancelled is resumed (RF-7.5). |
| LC-18 | Removed member was the only Reviewer of a project | Removal allowed; grants revoked; approvals already recorded stay; the project simply has no Reviewer until one is granted (RF-7.6). |
| LC-19 | Removed member is invited again and accepts | A new membership with the invited scope only, no former grant (RF-7.10, RF-8.4). |
| LC-20 | Tenant suspended while the page is open | Next request meets the suspended-tenant screen; every pending confirmation is refused; pending invitations wait (RF-1.4). |
| LC-21 | Member types the page's address | Denied-scope state; no row (RF-1.3). |
| LC-22 | Search for a removed member's address | No row; the audit trail holds the removal (RF-2.2, RF-3.4). |
| LC-23 | Language switched with a dialog open | Display changes; selection, step-up and attempts untouched (RF-1.6). |
| LC-24 | Interface reloads during "transferring" or "sending" | The client re-submits the same confirmation; the server returns the recorded outcome; one transfer, one invitation (RF-5.7, RNF-13). |
| LC-25 | Notification or invitation message cannot be delivered | The change stands; the failure is recorded; an invitation can be resent (RF-13.4, RF-9.3). |
| LC-26 | The only other Owner's identity is restricted | They do not count as an active Owner (D-a): the actor is the last active Owner and cannot be demoted or removed; the restricted Owner may be demoted or removed (RF-10.1). |
| LC-27 | Actor's own membership suspended by another Owner while a dialog is open | Confirmation refused: the actor is no longer an active Owner or Admin (RF-10.6). |
| LC-28 | Role filter "Admin" with state "Suspended" | Only suspended Admins listed; the count reflects the scope (RF-2.5, RF-2.6). |
| LC-29 | Owner edits the role of a member whose identity is under erasure | The row is listed with its state; a role change is allowed but the person cannot be made Owner (identity not active, RF-4.2); their erasure proceeds under specification 015 (RF-6.3). |
| LC-30 | "Decision history" opened by an Admin | The audit trail shows the membership, invitation and ownership events the Admin's authority permits; no project payload (RF-3.4). |
| LC-31 | Invite an address that is already an active member | Refused with that reason; nothing issued (RF-8.4). |
| LC-32 | Invite an address with a pending invitation | Refused, pointing to "Review invitation"; nothing issued (RF-8.4). |
| LC-33 | Invite an address that has no ASpectra identity | Issued exactly as for any other address; nothing reveals the difference; the person creates the account from the link (RF-8.6; specification 008 RF-4). |
| LC-34 | Admin tries to issue an invitation carrying Owner | Not offered; a direct submission is refused (RF-8.2, RF-10.5). |
| LC-35 | Invitation resent after the invitee opened the first link but had not accepted | The first link shows "can't be accepted"; the new link is pending with a fresh expiry (RF-9.3; specification 008 RF-1.5). |
| LC-36 | Revoke racing with the invitee's acceptance | Exactly one transition; the loser sees the "can't be accepted" state or a stale-state refusal (RF-9.7; specification 008 D-c). |
| LC-37 | Invitation expires | Row leaves both views; no message; the link shows "can't be accepted" (RF-9.5). |
| LC-38 | Invitee declines | Row leaves both views; the inviter is notified of the decline (RF-9.6, RF-13.4). |
| LC-39 | Inviter is removed from the tenant before the invitee accepts | The invitation stays pending and acceptable; the notification goes to the issuer's verified address regardless of membership (RF-9.8, RF-13.4). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **Opening, reviewing, accepting and declining an invitation** — specification 008; this feature issues, lists, resends and revokes.
- **Tenant policy** — F-SET-05; the "Workspace policy" tab only routes there.
- **Project grants** — specification 011; "Review project grants" only routes there; no grant is edited here, and an invitation's grants become records only at acceptance (008).
- **Leaving a tenant and identity erasure** — specification 015, including the last-Owner transfer that erasure requires.
- **Tenant suspension, deletion and lifecycle** — specification 014 and the tenant lifecycle features; this feature is unreachable while the tenant is suspended or deleting.
- **The authority matrix itself, the meaning of statuses, the permission version and revocation semantics** — specification 009; this feature applies them.
- **Session listing and revocation** — specification 007; no change here ends a session.
- **The audit trail surface** — F-APR-04; this feature only links to it and appends entries.
- **Global identity controls** on another person — password reset (004), factor replacement (003), controlled recovery (005), erasure (015): no such capability exists for an administrator (FR-IDN-049).
- **Export of tenant data** — FR-IDN-124; tenant administration grants no export.
- **Invitations carrying several projects, custom roles or teams; bulk invitations; invitation by domain** — product boundary (FR-IDN-090, scope §2).
- **Tablet- or mobile-specific designs** of R13, its dialogs and panels — none; the D01 responsive policy applies (S-02).
- **"Help & shortcuts"** in the app shell — the shell's N43 HelpShortcutsPanel (`430:4129`) (S-17).
- **The choice and configuration of the transactional email channel** — an operational prerequisite, not a feature.

---

## 7. Criteria of completion

Tenant members, invitations and ownership transfer are complete when all of the following hold:

1. Every user story US-1 … US-12 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-13 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-13 has at least one verification that fails before and passes after the feature exists.
3. Authority tests show that the page opens for active Owners and Admins only, that each row offers exactly the actions of RF-2.4 for the actor's role, that an Admin can reach no Owner-affecting action or Owner invitation by any route, and that nobody can act on their own row (RF-1.3, RF-2.4, RF-6.2, RF-7.2, RF-10.5, RNF-1).
4. Last-Owner tests show that no sequence of role edits, suspensions, removals or transfers — including concurrent ones — leaves the tenant without an active Owner, that the lock serialises concurrent Owner-affecting changes, and that a restricted or suspended Owner does not count (RF-10, RNF-2, RNF-4, LC-3, LC-4, LC-26).
5. Transfer tests show the picker's eligibility rule, the step-up with its generic failure, attempt limit and recovery-code refusal, the recheck under the lock with its stated refusal reasons, the single indivisible change with both permission versions, both notifications, and authority derived at the next request without a new sign-in (RF-4, RF-5, LC-1, LC-5 … LC-12).
6. Role-edit and membership tests show the step-up required exactly for Owner-affecting changes, confirmation only otherwise, the effects of suspension, reinstatement and removal on commands, subscriptions, grants, evidence and in-flight work, and that sessions are never revoked (RF-6, RF-7, LC-14 … LC-19).
7. Invitation tests show the exact scope rule, one pending per address, the refusals for existing memberships and pending invitations with no disclosure for unknown addresses, resend with a fresh secret that kills the earlier link, revocation that is final, expiry that is silent, the rows leaving both views on every settlement, and the inviter notified on acceptance and decline (RF-8, RF-9, RNF-9, RNF-10, LC-31 … LC-39).
8. Boundary tests show that no change grants project access, touches an identity's credential, factor, codes, sessions, address or other memberships, cancels or replays an accepted Run, or affects the tenant's state, policy or other tenants (RF-11, RNF-7, RNF-8).
9. Stale-version tests show that every confirmation carries the versions or states it was opened against and is refused when either moved (RF-10.4, RF-10.6, LC-27).
10. Idempotency tests show that a re-submitted confirmation or issue returns the recorded outcome and applies nothing twice (RF-5.7, RNF-13, LC-24).
11. Audit entries exist for every event of RF-13.1 and contain no password, code, invitation secret or project content; the messages of RF-13.4 exist in EN, ES and FR, reach only the person concerned, and a delivery failure never reverts a change (RF-13, LC-25).
12. The product refuses to start when a configured value of this feature is absent or unbounded (RNF-12).
13. The page, its tables, its dialogs, its panels and its states pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RNF-15, RNF-16, RNF-17).
14. The register items this specification relies on — S-50, S-49, S-04, S-14 and S-02 in the D03 Screen decisions ledger (`148:4`); A-52, A-54, A-55, A-56, A-57, A-30, A-27 and A-36 in the architecture register — are resolved, or explicitly deferred by the product owner, before the feature is declared done. The constitution's F-IDN-12 and F-SET-04 entries (v1.4) already reflect decisions Q1 and Q8.
