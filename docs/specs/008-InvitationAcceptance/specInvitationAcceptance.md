# Specification 008 — Invitation acceptance

**Feature** Opening an invitation link, reviewing the exact scope it grants, accepting it as the invited identity or declining it · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-10 as the subject; F-IDN-01, F-IDN-02, F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-11, F-IDN-12, F-IDN-13, F-IDN-15, F-IDN-16, F-IDN-17, F-IDN-18 and F-SET-04 where a criterion touches their boundary; F-X-05, F-X-06, F-X-07; principles P3, P6, P12, P13, P14) → `requirements.md` (FR-IDN-001, 002, 012, 015, 020, 021, 023, 024, 026, 030, 067, 068, 080, 081, 082, 083, 084, 085, 090, 091, 092, 093, 097, 098, 099, 110, 111, 116, 121, 130; FR-SET-030, 031, 033; FR-X-030, 033, 034, 040, 042, 043, 044, 045, 046, 050, 051, 053, 055) → the design of record: screen N02 InvitationAcceptancePage `161:49265`, its mobile counterpart M03 MobileInvitationPage `161:51284`, the compact states N29 InvitationWrongIdentityPanel `159:32110` and "Expired or revoked invitation" `159:32148` inside the D02 interaction atlas `159:32045`, and the "Pending invitations" view and "Review invitation" row of R13 TenantMembersPage `148:503` as the issuing side this feature presumes.

**Related specifications:** 001 Login (the complete sign-in, with the TOTP challenge when a factor is enrolled, that an acceptance requires; its throttles RF-4; its landing rule RF-8 for a sign-in started from the pre-match state; "Leave invitation" and "Return to memberships" lead to the workspace chooser, specification 013) · 002 Signup and email verification (the signup opened from an invitation, RF-4 here, and the confirmed state's "Return to your invitation"; verification never accepts an invitation, 002 RF-11) · 003 TOTP enrollment, recovery codes and factor replacement (the enrollment gate applied at tenant entry to an invited Admin or policy-required Member, 003 RF-1.3 and RF-1.4; its LC-23 is this feature's RF-6.7) · 006 Sessions, refresh and expiry (the session continues through acceptance; the new membership is visible under its permission version) · 007 Logout and session management ("Switch identity safely" signs out server-side) · 009 Roles, grants and the authority matrix (what a membership, a project grant and `canRun` are) · 010 Tenant members and ownership transfer (issuing, listing, reviewing and revoking invitations; ownership, which an invitation never grants) · 013 Tenant chooser (where a declined or unusable invitation returns the person) · 014 Tenant suspension state (a suspended tenant's invitations wait) · 016 Guarded tenant and project switch (how the accepted tenant is entered).

This document states **what** invitation acceptance does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global** — one principal with a verified email, a display name and a status — and belongs to no tenant by itself. Joining an existing tenant has exactly one route: an **invitation** issued by a permitted Owner or Admin of that tenant (specification 010), bound to the invited address, the tenant, one tenant role and, optionally, one project with explicit grants. Nothing else joins an identity to a tenant: not a matching email domain, not a successful sign-in, not an administrator "adding" an address, not verification of the address (FR-IDN-012, FR-IDN-015; scope §2).

The invitation reaches the person as a link in a message to the invited address. Opening the link shows an invitation page with three possible situations:

- the person is **signed in as the invited identity** — the page shows exactly what the invitation grants (tenant, tenant role, project, explicit project grants, whether `canRun` is granted), asks for an acknowledgement that membership and project grants are separate, and offers to accept or to decline;
- the person is **signed in as a different identity** — the page refuses, explains that matching an email never links identities, and offers to sign out and switch, or to leave;
- the person is **not signed in** — the page shows only the tenant name and the masked invited address, and offers to sign in with an existing local account or to create one with the invited address, verify it and come back.

Acceptance is a single, indivisible act: the invitation is consumed, a membership with the invited role is created and the invited project grants are recorded as explicit grant records — exactly as issued, never more. The person then enters the tenant, where required MFA is gated as everywhere else (specification 003). Declining closes the invitation for good. Anything expired, revoked, used or declined leads to one generic "can't be accepted" state that names no cause and reveals no tenant content.

### 1.2 Objective

Let an invited person join a tenant with the exact scope an administrator chose, and nobody else join anything, while:

- showing the invitee, before the act, every element of what they are about to hold — tenant, tenant role, project, project grants, `canRun` — and that membership and project grants are two different things;
- accepting only for the identity whose verified address is the invited one, and only from a complete sign-in, so that a leaked link, a shared browser or a matching domain grants nothing;
- consuming every invitation exactly once, bounded in time, with the same generic answer for every unusable link;
- applying exactly the issued scope — never an Owner role, never a role or grant the invitee chose, never a merge into an existing membership;
- giving a person without an account a clear route — create it with the invited address, verify, sign in, return — that leaves the invitation untouched until they accept it themselves;
- disclosing nothing about tenants, members, projects or identities to anyone who has not proven to be the invited identity.

### 1.3 Why it matters

An invitation link travels through a mailbox, gets forwarded, sits in a shared inbox and is opened on the wrong device signed in as the wrong person. A product that accepts it for whoever opens it hands tenant membership to the mailbox rather than the person; one that lets the invitee pick their own role turns the inviter's decision into a suggestion; one that joins by email domain makes every employee of a lookalike domain a member. The opposite failure matters too: an invitation that cannot be accepted by a brand-new person, or that silently dies when the inviter changes role, leaves the founder's first colleague locked out with no one to ask. The feature must make the scope explicit, the identity exact, the act atomic and the dead ends generic.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, all written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14, summary table and entry paragraphs: **A-52** (invitation record, its lifecycle and the atomic acceptance transaction) is new; **A-28** (signup started from an invitation) is closed by decision Q3; a note is appended to **A-27** (identity statuses). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-45** (desktop N02 lacks "Decline invitation") and **S-46** (the pre-match state of N02) are new; **S-24** (return to the invitation after confirmation) is closed by decision Q3; notes are appended to **S-03** (mobile counterparts), **S-04** (destinations of the remaining buttons, now D02 `439:4460`), **S-14** (email templates) and **S-17** (Help link).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification and the meaning of "Decline" | Opening an invitation link, reviewing the scope, accepting and declining. Decline closes the invitation permanently — it behaves afterwards exactly like a used one — is audited, and returns the person to their memberships. Issuing an invitation (R13 "Invite member", whose dialog is drawn in D02 `419:2645`, S-04), the pending-invitations view, "Review invitation" and revoking a pending invitation are tenant administration (FR-IDN-097) and belong to specification 010; this specification states only what an issued invitation must carry, as a precondition. |
| Q2 | What a link holder sees before the signed-in identity matches | A person who is not signed in sees the tenant name, the invited address in masked form, and the routes "Existing local account" and "Create local account" — nothing else. The tenant role, the project, the grants and `canRun` appear only once the signed-in identity is the invited address, as N02 and M03 draw it. The wrong-identity panel N29 and the "can't be accepted" state show no tenant content beyond those two facts. |
| Q3 | How a new person returns to the invitation | The same link is the return route. Signup opened from the invitation page shows the invited address masked as the email field's hint and the server refuses any other address (RF-4.1; plan D11); the person verifies it, signs in and re-opens the link, which stays pending and bound to that address. When the browser still holds a reference to the link, the confirmed state of verification and the sign-in screen offer "Return to your invitation"; that reference carries no authority and the invitation is revalidated on return. Signup, verification and sign-in never consume or accept the invitation. This closes A-28 and S-24. |
| Q4 | What an invitation may carry and what is re-checked at acceptance | A tenant role of **Admin or Member** only — Owner is obtained only through ownership transfer (F-IDN-12) — and optionally **one** project with an explicit subset of Author / Reviewer / Viewer plus an explicit `canRun`. Acceptance applies exactly the scope bound at issue time; the invitee changes nothing. At the moment of acceptance the tenant must be active and the invitation pending — unexpired, unrevoked, unused, undeclined. The inviter's later loss of authority does not void an invitation; revocation is an explicit administration act (specification 010). |
| Q5 | The invitee already holds a membership record in that tenant | An **active or suspended** membership makes the invitation unusable (the generic state, routing to memberships) and nothing about the existing membership changes. A **revoked** membership is replaced by a new active membership carrying exactly the invited scope and a new permission version: no earlier grant is restored; authored evidence and provenance keep their attribution. |
| Q6 | When MFA is required and where acceptance lands | Accepting requires a complete identity session — the password and, whenever a factor is enrolled, the TOTP challenge of specification 001. Holding a factor is not a precondition: an invited Admin, or a Member under a tenant policy that requires MFA, accepts, the membership is created, and on entering the tenant authority is withheld until enrollment completes (specification 003 RF-1.3, RF-1.4). After acceptance the person enters the tenant directly through the guarded switch (F-IDN-18): the granted project is offered when there is one; otherwise the tenant view shows no project content. |
| Q7 | Messages on accept and decline | None from this feature. Acceptance and decline are audited, and the pending-invitations and members views reflect them; no message goes to the invitee (who just acted). Whether the inviter is told was left to specification 010, which decided (010 Q10, RF-9.6) that the inviter receives a short notification on acceptance and decline; that message is sent by 010's flow from the settled invitation, not by this feature. The invitation message itself belongs to issuing (S-14, specification 010). |

Five rules were derived from these decisions and from earlier specifications rather than asked; the product owner may overturn them:

- **D-a** An unknown, malformed, expired, revoked, used or declined link, and a pending invitation blocked by the tenant's state or by an existing membership, all render the single "This invitation can't be accepted" state of the D02 atlas, which never says which cause applies; link resolutions are throttled per calling network and risk signal like every other public entry (specification 001 RF-4; FR-IDN-021).
- **D-b** The acknowledgement "I understand the membership and separate project grants." is never pre-ticked and is required for acceptance, as the acknowledgement of specification 005 RF-4.1 is; the accept action names the identity that will hold the membership ("Accept invitation as …").
- **D-c** Single use is atomic: two concurrent acceptances, or an acceptance racing with a decline, a revocation or expiry, yield exactly one transition of the invitation; every other attempt sees the "can't be accepted" state (as specification 005 RNF-3 treats codes and approvals).
- **D-d** An invitation to a **suspended** tenant is not consumed: it stays pending and becomes acceptable again if the tenant returns to active before it expires; an invitation to a tenant that is **deleting** is unusable for good.
- **D-e** Acceptance changes nothing but the new membership and its grants: it creates no session, changes no password, factor, address or other membership, lifts no suspension, and replays no pending Run intent; the current session continues and sees the new membership at once.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | person with a verified local account who received an invitation | to open the link, see exactly which tenant, role, project and grants I am offered, and accept as myself | I join with the scope the inviter chose and no surprise later |
| US-2 | person without an account who received an invitation | to create a local account with the invited address, verify it, sign in and come back to the same invitation | I can join without asking anyone to re-invite me |
| US-3 | invited Admin who has no authenticator yet | to accept the invitation and then be guided into TOTP enrollment when I enter the tenant | the membership exists and my authority starts exactly when the factor does |
| US-4 | person signed in as a different identity than the invited one | to be told plainly that this identity cannot accept, and to sign out and switch safely | the invitation is never accepted for the wrong account |
| US-5 | former member whose membership was revoked and who is invited again | to accept and receive a fresh membership with the new scope only | I rejoin cleanly, without inheriting what I used to have |
| US-6 | person who does not want to join | to decline, and to know the invitation is closed | the inviter sees it is settled and the link cannot be used later |
| US-7 | person on a phone | the invitation page to carry every statement, the acknowledgement, accept and decline | I can join from anywhere without losing the gates |
| US-8 | tenant Owner or Admin | the invitee to receive exactly the role and grants I issued and nothing more, and the acceptance to appear in my members view | the scope I decided is the scope that exists |
| US-9 | attacker, forwarded-link recipient or over-reaching administrator (negative story) | — | I cannot learn tenant, project or grant details from a link; cannot accept for an identity I do not control; cannot join by email domain or by signing in; cannot raise, add or merge roles or grants; cannot become an Owner through an invitation; cannot reuse an accepted, declined, expired or revoked invitation; and cannot reach any surface that accepts on another person's behalf |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *invitation* means a record issued by a permitted Owner or Admin of a tenant (specification 010) that holds the invited address, the tenant, one tenant role, at most one project with its explicit grants, the issuing identity, the time of issue, a configured expiry and a state; *link* means the address contained in the invitation message, which carries the invitation's secret; *the invited identity* means the active identity whose verified address equals the invited address, compared case-insensitively and ignoring surrounding whitespace; *the review state* means the invitation page as N02 and M03 draw it for the invited identity; *the pre-match state* means the page shown to a person who is not signed in; *unusable* means any invitation that RF-5 does not allow to be accepted; *a complete identity session* means the identity session established by specification 001 RF-8.1 after the password and, when a factor is enrolled, the TOTP challenge.

### RF-1 · Opening a link

A link proves that someone holds the message; it proves nothing about who. *Why:* the page must orient the invited person and give everyone else nothing to work with. *(Decision: Q2; D-a.)*

- RF-1.1 When a link is opened, the system shall resolve its secret against the hashed form held at rest to at most one invitation; the secret shall never be stored in clear and shall never be written to browser storage beyond the current navigation. *(FR-IDN-080, FR-X-030, FR-X-034; A-52)*
- RF-1.2 When the link resolves to a pending invitation and the person is not signed in, the system shall present the pre-match state: the marker "LOCAL IDENTITY / HUMAN CONTROL", the heading "An invitation. An explicit scope.", the subtitle "Review the workspace membership and project grants before you join.", the tenant name, the invited address in masked form, the routes "Existing local account" (to the sign-in screen of specification 001, with the link reference kept for RF-4.4) and "Create local account" (RF-4.1), the statement that the invitation can be accepted only by the local identity holding the invited address, the route "← Back to sign in", the Privacy and Terms routes and the interface-language switch (EN / ES / FR). It shall show no tenant role, project, grant, `canRun` value, inviter or member. *(FR-IDN-082, FR-X-043; decision Q2; S-46)*
- RF-1.3 When the link resolves to a pending invitation and the signed-in identity is the invited identity, the system shall present the review state (RF-2). *(FR-IDN-081, FR-IDN-082; N02, M03)*
- RF-1.4 When the link resolves to a pending invitation and the signed-in identity is not the invited identity, the system shall present the wrong-identity panel (RF-3). *(FR-IDN-082; N29)*
- RF-1.5 If the link resolves to no invitation, or to an invitation that is unusable under RF-5, then the system shall present the "can't be accepted" state of RF-5.6 — the same state, with the same content, for every cause. *(FR-IDN-083; D-a)*
- RF-1.6 Opening a link shall change nothing: it shall accept nothing, create no membership or grant, consume no invitation, create no session and link no identity. *(FR-IDN-012, FR-IDN-015)*
- RF-1.7 The system shall render a "Help" action on the page opening N43 HelpShortcutsPanel (`430:4129`; S-17); Privacy and Terms open the documents the platform operator publishes outside the product, with an external-link cue (specification 017 Q3; register `122:2` §15) (S-14).
- RF-1.8 When the language switch is used, the system shall change the display only; nothing resolved, ticked or pending shall change. *(FR-X-043)*
- RF-1.9 The system shall treat the invited address and the signed-in address case-insensitively and ignore surrounding whitespace, as sign-in does (specification 001 RF-2.6), when deciding whether they are equal. *(FR-IDN-082)*

### RF-2 · The review state

The invitee sees the whole scope before the act, and acknowledges that two different things are being granted. *Why:* FR-IDN-081 — an Admin who believes tenant administration reads every project, or a Member who believes Author implies `canRun`, will act on a wrong model. *(Decision: Q1, Q2, Q4; D-b.)*

- RF-2.1 The review state shall present: the heading "Join <tenant>", the status "Invitation available", "Invited local identity" with the invited address in full, the block WORKSPACE with the tenant name, "Workspace membership" with the tenant role, the block PROJECT with the project name and "Explicit project grants" listing the granted project roles and, separately and explicitly, "canRun: granted" or "canRun: not granted" — or, where the invitation carries no project, the statement that the invitation grants tenant membership only and no project grants — the statement "Invited by a permitted <tenant> administrator. No automatic joining by email domain.", the block "Identity matches — Signed in as <address>. This is the identity on the invitation.", the acknowledgement "I understand the membership and separate project grants." as a checkbox that is never pre-ticked, the action "Accept invitation as <display name>", the action "Use another identity" (RF-3.2), the action "Decline invitation" (RF-7), the MFA handoff block (RF-8.1), the "Next destination" statement naming the tenant and, where granted, the project, and the statement "Switching identity requires signing out; the invitation is not accepted for the wrong account. Expired, revoked or already-used invitations cannot be accepted. Request a fresh invitation from the inviting workspace." *(FR-IDN-081, FR-IDN-082, FR-IDN-083, FR-IDN-090; N02, M03; decision Q4; S-45, S-46)*
- RF-2.2 The system shall show every element of the scope exactly as the invitation holds it and shall offer no control that changes the tenant role, the project, a project role or `canRun`. *(FR-IDN-084)*
- RF-2.3 The review state shall state that tenant membership and project grants are separate, that an Admin role reads no project content by itself, and that `canRun` is a grant independent of the project roles. *(FR-IDN-081, FR-IDN-092, FR-IDN-098)*
- RF-2.4 If the acknowledgement is not ticked when the accept action is activated, then the system shall identify it with a message associated with it and shall not submit. *(FR-X-042; D-b)*
- RF-2.5 The review state shall expose nothing of the tenant beyond its name, the project name and the invited grants: no member, no other project, no source, no run, no policy and no inviter identity. *(FR-IDN-111 by analogy; P3)*
- RF-2.6 While the review state is shown, the system shall hold the invitation unchanged; a review that is abandoned, reloaded or left idle shall consume nothing. *(FR-IDN-080)*

### RF-3 · The wrong identity

A signed-in identity that is not the invited one is refused, whatever its email looks like. *Why:* FR-IDN-082 and FR-IDN-015 — accepting for the wrong account, or linking accounts because their addresses match, are the two ways a mailbox becomes a membership. *(Decision: Q2.)*

- RF-3.1 The wrong-identity panel shall present: the heading "Use the invited identity", the statement "This signed-in identity cannot accept this invitation. Matching an email never automatically links identities.", the tenant name and the masked invited address as in the pre-match state and nothing more, the statement that no protected project content is revealed, and the actions "Switch identity safely" and "Leave invitation". *(FR-IDN-082, FR-IDN-015; N29; decision Q2)*
- RF-3.2 When "Switch identity safely" (or "Use another identity" on the review state) is activated, the system shall end the current session server-side (specification 007) and present the sign-in screen; the link reference may be kept by the browser so that sign-in can offer "Return to your invitation" (RF-4.4) and shall carry no authority. *(FR-IDN-068, FR-IDN-082; decision Q3)*
- RF-3.3 When "Leave invitation" is activated, the system shall route the current identity to the tenant chooser, which always renders on explicit navigation (specification 013 RF-1.2, RF-1.3), and shall leave the invitation pending. *(FR-IDN-110)*
- RF-3.4 The system shall decide whether the signed-in identity is the invited identity on the server at every resolution and again at acceptance (RF-5.8); the identity shown by the client shall never be the basis of the decision. *(FR-IDN-082; P3)*
- RF-3.5 The system shall never offer to link, merge or re-address identities because an address matches, and shall never accept an invitation into a session of another identity. *(FR-IDN-015)*

### RF-4 · The new identity

A person with no account creates one with the invited address and comes back through the same link. *Why:* FR-IDN-085 — the invitation must survive signup, verification and sign-in untouched, so that only the act on the review state joins anything. *(Decision: Q3.)*

- RF-4.1 When "Create local account" is activated, the system shall present the signup screen of specification 002 RF-1 with the masked invited address shown as the email field's hint, the statement that the account must be created with the invited address and verified before returning to the invitation, and a refusal — naming no address — of any other typed address, since the full invited address is never disclosed before the identity matches (decision Q2). *(FR-IDN-085, FR-IDN-010; decision Q3; A-28, S-24)*
- RF-4.2 Signup, email verification and sign-in shall not consume, accept, alter or extend the invitation and shall grant no membership or project grant; specification 002 RF-11 applies. *(FR-IDN-012; 002 RF-11.1, RF-11.2)*
- RF-4.3 The system shall keep the invitation pending, bound to the invited address, until it is accepted, declined, revoked or expired; a new identity created with that address becomes the invited identity by that fact alone and holds nothing until it accepts. *(FR-IDN-080, FR-IDN-085)*
- RF-4.4 Where the browser still holds the link reference, the confirmed state of verification (specification 002 RF-8.3) and the sign-in screen shall offer "Return to your invitation", which leads, after sign-in, to opening the same link under RF-1; the reference shall carry no authority and the invitation shall be revalidated at that moment. *(decision Q3; S-24)*
- RF-4.5 The system shall always accept the link itself as the return route: a person who re-opens it from the message after signing in reaches the same state as RF-4.4. *(FR-IDN-085)*
- RF-4.6 If the person creates or signs in to an identity with a different address, then RF-3 applies; the system shall not suggest re-addressing or linking. *(FR-IDN-015)*
- RF-4.7 The system shall not require TOTP enrollment before acceptance; required MFA is gated at tenant entry under RF-6.7. *(FR-IDN-085, FR-IDN-026; decision Q6)*

### RF-5 · Validity and the checks at acceptance

An invitation is acceptable in exactly one state, by exactly one identity, into exactly one tenant state. *Why:* FR-IDN-080 and FR-IDN-083 — anything else must fail closed and look the same. *(Decision: Q4, Q5; D-a, D-d.)*

- RF-5.1 The system shall hold, per invitation, the invited address, the tenant, a tenant role of **Admin or Member**, at most one project with an explicit subset of Author / Reviewer / Viewer and an explicit `canRun` value, the issuing identity, the time of issue, a configured expiry, and its state. No invitation shall carry the Owner role, more than one project, or an unstated grant. *(FR-IDN-080, FR-IDN-084, FR-IDN-090; decision Q4; A-52)*
- RF-5.2 The states shall be *pending* → *accepted* (RF-6) | *declined* (RF-7) | *revoked* (specification 010) | *expired* (configured expiry elapsed). Only *pending* is acceptable; every other state is final. *(FR-IDN-083; A-52)*
- RF-5.3 The system shall accept an invitation only when all of the following hold at the moment of acceptance: the invitation is *pending* and unexpired; the tenant is *active*; the signed-in identity is active, has a verified email and is the invited identity; the acceptance is made from a complete identity session; and the identity holds no *active* or *suspended* membership in that tenant. *(FR-IDN-012, FR-IDN-082, FR-IDN-093; decisions Q4, Q5, Q6; A-27)*
- RF-5.4 While the tenant is *suspended*, the invitation shall be unusable but shall not be consumed: it stays *pending* and becomes acceptable again if the tenant returns to *active* before the expiry. While the tenant is *deleting*, the invitation shall be unusable for good. *(FR-IDN-116; D-d)*
- RF-5.5 If the identity holds an *active* or *suspended* membership in the tenant, then the invitation shall be unusable and nothing about that membership shall change. *(decision Q5)*
- RF-5.6 The "can't be accepted" state shall present the heading "This invitation can't be accepted", the statement "Ask the permitted inviter for a new invitation. No workspace content or other identity details are shown.", and the route "Return to memberships →", which leads a signed-in identity to the tenant chooser (specification 013 RF-1.2, RF-1.3) and leads to the sign-in screen otherwise. It shall name neither the tenant, nor the cause, nor any identity. *(FR-IDN-083; D02 `159:32148`; D-a)*
- RF-5.7 Every expiry of this feature shall be validated configuration; the system shall refuse to start with an absent or unbounded value and shall never substitute a default. *(FR-X-046; P13)*
- RF-5.8 The system shall evaluate RF-5.3 on the server at the moment of acceptance, independently of what was shown: a review state that became stale — because the invitation was revoked, expired, declined or accepted elsewhere, the tenant changed state or a membership appeared — shall lead to the "can't be accepted" state when the accept action is activated. *(P3, P6; D-c)*

### RF-6 · Acceptance

One act, one transaction, exactly the issued scope. *Why:* FR-IDN-080 (atomically single-use), FR-IDN-084 (no self-escalation) and FR-IDN-099 (versioned, auditable) all hinge on the act being indivisible. *(Decision: Q4, Q5, Q6; D-c, D-e.)*

- RF-6.1 When the accept action is activated with the acknowledgement ticked and RF-5.3 holds, the system shall, as one indivisible change: move the invitation to *accepted*, so that it can never be accepted again anywhere; create a membership of the identity in the tenant in state *active* with the invited tenant role and a new permission version; create one explicit grant record per invited project role and the `canRun` grant where issued, exactly as the invitation holds them; and append the audit entry of RF-11.1. *(FR-IDN-080, FR-IDN-093, FR-IDN-096 by analogy, FR-IDN-099; decision Q4; D-c; A-52)*
- RF-6.2 When the identity holds a *revoked* membership in the tenant, the same change shall replace it with the new *active* membership carrying only the invited scope and a new permission version; no earlier tenant role or project grant shall be restored, and authored evidence and provenance shall keep their attribution. *(FR-IDN-093, FR-IDN-121, FR-IDN-098; decision Q5)*
- RF-6.3 The system shall never grant more than the invitation holds: no Owner role, no tenant role other than the invited one, no project beyond the invited one, no project role or `canRun` not issued; any value supplied by the client shall be ignored and the scope derived from the record alone. *(FR-IDN-084; P3)*
- RF-6.4 Acceptance shall create no session, change no password, factor, address or other membership, lift no suspension, regenerate nothing, and replay no Run intent pending before it. *(FR-IDN-067; D-e)*
- RF-6.5 When RF-6.1 completes, the system shall enter the tenant through the guarded switch of F-IDN-18 (specification 016): where a project was granted, that project is offered as the destination named in "Next destination"; otherwise the tenant view opens with no project content. *(FR-IDN-130; decision Q6; M03)*
- RF-6.6 The current session shall see the new membership and its grants from the moment of RF-6.1, under the new permission version, without re-authentication. *(FR-IDN-093; specification 006)*
- RF-6.7 When the accepted role is Admin, or Member in a tenant whose policy requires MFA, and the identity has no active factor, the system shall apply the enrollment gate of specification 003 RF-1.3 and RF-1.4 at tenant entry: the membership exists, its authority is withheld, and the enrollment wizard is the only way forward in that tenant. Acceptance shall never grant authority earlier and never bypass the gate. *(FR-IDN-023, FR-IDN-026, FR-IDN-030; decision Q6; 003 LC-23)*
- RF-6.8 If two acceptances of the same invitation, or an acceptance and a decline, revocation or expiry, are submitted concurrently, then exactly one transition shall take effect and every other attempt shall receive the "can't be accepted" state. *(FR-IDN-080; D-c)*
- RF-6.9 When the invitation is accepted, it shall leave the tenant's pending-invitations view and the new membership shall appear in the members view (specification 010; F-SET-04). *(FR-IDN-097, FR-SET-030, FR-SET-031)*

### RF-7 · Decline

Declining is final and tells the tenant the matter is settled. *Why:* a dismissed-but-pending invitation keeps a live secret in a mailbox for no purpose. *(Decision: Q1, Q7.)*

- RF-7.1 The system shall offer "Decline invitation" on the review state only, to the invited identity only; the pre-match state and the wrong-identity panel shall not offer it. *(FR-IDN-082; M03; S-45)*
- RF-7.2 When "Decline invitation" is activated, the system shall ask for confirmation in one compact dialog — "Decline this invitation?", stating that the decision is final, that the link stops working, that no message is sent and that nothing is created; actions "Decline invitation" and "Keep it open" (D02 `424:3689`) — and on confirmation shall move the invitation to *declined*, append the audit entry of RF-11.1, create nothing, change no membership, send no message of its own, and route the identity to the tenant chooser (specification 013 RF-1.2, RF-1.3). *(decision Q1, Q7)*
- RF-7.3 A declined invitation shall behave exactly like a used one: any later opening of its link shall show the "can't be accepted" state, and a new membership for that address requires a fresh invitation (specification 010). *(FR-IDN-083; decision Q1)*

### RF-8 · The MFA handoff statements

The page says where MFA happens so that nobody takes acceptance for authority. *Why:* FR-IDN-023, FR-IDN-026 and FR-IDN-030 — the membership exists before the factor does, and the person must know it.

- RF-8.1 The review state shall carry the block "MFA handoff — After acceptance, complete any required MFA before entering protected content. Owner and Admin roles always require TOTP. Member requirements follow the inviting workspace's policy. Email verification does not substitute for MFA." *(FR-IDN-023, FR-IDN-030; M03)*
- RF-8.2 The review state shall carry the statement "Acceptance applies only to the invited local identity." and the "Next destination" statement naming the tenant and, where granted, the project chooser. *(FR-IDN-082; M03)*
- RF-8.3 The pre-match state and the signup route of RF-4.1 shall state that any required MFA is completed before protected access, as the N02 "New here?" statement does. *(FR-IDN-085; N02)*

### RF-9 · Abuse protection and generic states

A link is guessable in principle and a page is a probe in practice. *Why:* FR-IDN-020 and FR-IDN-021 — the invitation page must not become an oracle for tenants, addresses or invitations. *(D-a.)*

- RF-9.1 The system shall throttle link resolutions, acceptances and declines per calling network and risk signal, with configured thresholds, progressive delays and bounded suppression windows; these throttles shall never restrict the invited identity's sign-in and shall lift automatically. *(FR-IDN-021; specification 001 RF-4.7)*
- RF-9.2 The "can't be accepted" state shall be identical in content and shape, with timing comparable within normal variance, for an unknown link, an expired, revoked, accepted or declined invitation, a suspended or deleting tenant and an existing membership. *(FR-IDN-020; D-a; RNF-2)*
- RF-9.3 No state of this feature shall reveal whether an address has an identity, whether an identity has a factor, which tenants exist, or whether any invitation other than the one opened exists; the pre-match state shows only what the holder of the message already holds — the tenant name and the masked address. *(FR-IDN-020; decision Q2)*
- RF-9.4 Every threshold, delay and window of this feature shall be validated configuration with no default. *(FR-X-046; P13)*

### RF-10 · Responsive surface

M03 is the designed counterpart of the review state; the other states reflow. *Why:* FR-X-050 lists M03 among the genuine reflows, and the invitation is the first thing many people do on a phone. *(FR-X-050, FR-X-051, FR-X-055.)*

- RF-10.1 The system shall present the review state on mobile as M03 draws it, with every statement, the scope blocks, the acknowledgement, "Accept invitation", "Decline invitation", "Switch identity safely" (RF-3.2), the MFA handoff block and the "Next destination" statement preserved in meaning. *(FR-X-050; M03)*
- RF-10.2 The system shall present the pre-match state, the wrong-identity panel and the "can't be accepted" state in a reflowed layout at mobile and tablet widths that meets WCAG 2.2 AA reflow, with every element of RF-1.2, RF-3.1 and RF-5.6 preserved; M19 MobileInvitationPreMatchPage (`432:4231`) and M03 are the drawn counterparts, tablet follows the D01 responsive policy (`414:2645`). *(FR-X-051, FR-X-055; S-03)*
- RF-10.3 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*
- RF-10.4 The pre-match state shall follow M19 MobileInvitationPreMatchPage (`432:4231`) at mobile widths; N29 and the "can't be accepted" state follow the D01 responsive policy. *(FR-X-050; S-03)*

### RF-11 · Audit, messages and secrecy

Every outcome is recorded; the secret never is. *Why:* FR-X-044 and FR-IDN-099 — a membership's origin must be reconstructible, and the link that created it must not be in the record. *(Decision: Q7.)*

- RF-11.1 The system shall append to the audit log: each link resolution with its outcome class (pending, unusable, unknown) and the invitation when one is matched; each acceptance with the identity, the tenant, the invitation, the tenant role, the project grants and `canRun` recorded, and whether a revoked membership was replaced; each decline; each acceptance refused under RF-5.8 with its internal cause; each throttle applied or lifted. *(FR-X-044, FR-IDN-099)*
- RF-11.2 The audit record shall identify the identity when one is signed in, the time and the calling context, and shall never contain the link's secret, an unmatched address or a message body. *(FR-X-030, FR-X-044)*
- RF-11.3 This feature shall send no message of its own on acceptance or decline; the invitation message belongs to issuing, and the inviter notification on acceptance or decline is sent by specification 010 (RF-9.6) from the settled invitation. *(decision Q7; specification 010 Q10)*
- RF-11.4 The system shall never include the link's secret in a log, an error, a trace, a client-visible problem or analytics; client-visible problems shall use safe codes with a correlation identifier. *(FR-X-030, FR-X-033)*
- RF-11.5 The system shall never write the secret, the scope shown or the acknowledgement to browser storage; a reference to the link kept for RF-4.4 shall hold no more than what is needed to re-open it and shall grant nothing by itself. *(FR-X-034; decision Q3)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Single use.** An invitation is accepted at most once; under concurrency two acceptances, or an acceptance and a decline, revocation or expiry, yield exactly one transition. | FR-IDN-080; D-c |
| RNF-2 | **Indistinguishability.** The "can't be accepted" state is identical in content and shape, with timing comparable within normal variance, for every cause of RF-9.2. | FR-IDN-020, FR-IDN-083 |
| RNF-3 | **No authority from a link.** Holding, opening or forwarding a link grants nothing; only a complete identity session of the invited identity can accept, and the decision is taken on the server. | FR-IDN-082, FR-IDN-024; P3 |
| RNF-4 | **Exact scope.** The membership and grants created equal the invitation's record element by element; no Owner role, no extra project, no unissued grant, no merge with an existing membership. | FR-IDN-084, FR-IDN-090; Q4, Q5 |
| RNF-5 | **Fail closed.** An unknown, expired, revoked, accepted or declined invitation, a suspended or deleting tenant, an existing active or suspended membership, an unverified or restricted identity, a mismatched address, an absent configuration value: each blocks acceptance; nothing is assumed or defaulted. | P6; FR-X-046 |
| RNF-6 | **Configured limits.** Invitation expiry and every throttle threshold, delay and window are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-7 | **No lockout.** No volume of link openings or acceptance attempts by a third party restricts the invited identity's sign-in, and no restriction of this feature outlasts its configured window. | FR-IDN-021 |
| RNF-8 | **Secrecy.** The link's secret is held only in hashed form, never logged, exported, persisted in clear or written to browser storage. | FR-X-030, FR-X-034 |
| RNF-9 | **Isolation.** The invitation page reveals nothing of the tenant beyond its name and the invited project and grants, and nothing of any other tenant or identity; every row the feature writes is tenant-scoped. | FR-IDN-111; P12 |
| RNF-10 | **Atomic effect.** The transition of the invitation, the membership, the grant records and the audit entry take effect together or not at all; the new permission version is visible to the current session immediately. | FR-IDN-080, FR-IDN-093, FR-IDN-099 |
| RNF-11 | **MFA integrity.** No path of this feature grants Owner / Admin authority, or policy-required Member authority, before the enrollment gate of specification 003 is satisfied; acceptance never weakens a tenant policy. | FR-IDN-023, FR-IDN-030 |
| RNF-12 | **Accessibility.** The four states meet WCAG 2.2 AA with automated and manual assistive-technology evidence: the acknowledgement error associated with its control, focus moved to messages and states, status conveyed by text, no focus stealing. | FR-X-040, FR-X-042 |
| RNF-13 | **Language.** The four states exist in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-14 | **Reflow.** M03 carries every statement, gate and action of the review state; the other states reflow without loss. | FR-X-050, FR-X-051, FR-X-055 |
| RNF-15 | **Email as prerequisite.** The invitation message that carries the link depends on the transactional email channel (specification 010); its absence is an operational failure and never alters the behaviour of this feature. | FR-X-045 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Link opened while not signed in | Pre-match state: tenant name, masked address, two routes; nothing consumed (RF-1.2, RF-1.6). |
| LC-2 | Link opened signed in as the invited identity | Review state with the full scope (RF-1.3, RF-2.1). |
| LC-3 | Link opened signed in as another identity whose address differs only in domain or spelling | Wrong-identity panel; no linking offered (RF-1.4, RF-3.1, RF-3.5). |
| LC-4 | Signed-in address equals the invited one but differs in case or surrounding whitespace | Treated as equal; review state (RF-1.9). |
| LC-5 | Link forwarded to a third party who signs in as themselves | Wrong-identity panel; tenant name and masked address only; nothing else revealed (RF-3.1, RF-9.3). |
| LC-6 | Unknown or malformed link | "Can't be accepted" state; nothing reveals that no invitation exists (RF-1.5, RF-9.2). |
| LC-7 | Invitation expired while the review state was open; accept activated | "Can't be accepted" state; nothing created (RF-5.8). |
| LC-8 | Invitation revoked by an administrator while the review state was open | Same as LC-7; the race yields one transition (RF-5.8, RF-6.8). |
| LC-9 | Same invitation accepted in two tabs at once | Exactly one acceptance; the other tab sees "can't be accepted" (RF-6.8, RNF-1). |
| LC-10 | Accept and decline submitted concurrently | Exactly one transition; the other attempt sees "can't be accepted" (RF-6.8). |
| LC-11 | Link opened after the invitation was accepted | "Can't be accepted" state, even for the invited identity (RF-5.2, RF-1.5). |
| LC-12 | Link opened after decline | Same as LC-11 (RF-7.3). |
| LC-13 | Tenant suspended when the link is opened | "Can't be accepted" state; the invitation stays pending; acceptable again once the tenant is active, if unexpired (RF-5.4). |
| LC-14 | Tenant deleting | "Can't be accepted" state; the invitation is unusable for good (RF-5.4). |
| LC-15 | Invited identity already an active member of the tenant | "Can't be accepted" state; the existing membership untouched (RF-5.5). |
| LC-16 | Invited identity's membership in the tenant is suspended | Same as LC-15 (RF-5.5). |
| LC-17 | Invited identity's membership in the tenant was revoked | Review state; acceptance replaces it with a fresh active membership carrying the invited scope only; new permission version (RF-6.2). |
| LC-18 | Invited as Admin, no factor enrolled | Acceptance succeeds; on tenant entry authority withheld and the enrollment wizard is the only way forward (RF-6.7). |
| LC-19 | Invited as Member in a tenant whose policy requires MFA, no factor | Same gate as LC-18 (RF-6.7). |
| LC-20 | Invited as Member, no project in the invitation | Review state states membership only; acceptance creates the membership and no grant; tenant view with no project content (RF-2.1, RF-6.5). |
| LC-21 | Invitation grants `canRun` without Author or Reviewer | Shown exactly so; grant records created exactly so (RF-2.2, RF-6.1). |
| LC-22 | Client submits a different role or an extra project with the acceptance | Ignored; scope derived from the record (RF-6.3). |
| LC-23 | Accept activated without the acknowledgement | Field message; no submission (RF-2.4). |
| LC-24 | New person creates the account with the invited address, verifies, signs in and re-opens the link | Review state; the invitation was untouched meanwhile (RF-4.2 … RF-4.5). |
| LC-25 | New person creates the account with a different address | Wrong-identity panel on return; no linking (RF-4.6). |
| LC-26 | Browser lost the link reference before verification completed | No "Return to your invitation"; the person re-opens the link from the message (RF-4.5). |
| LC-27 | Invitation expires while the new person is still verifying | "Can't be accepted" on return; a fresh invitation is needed (RF-5.2, RF-5.6). |
| LC-28 | Identity restricted or under erasure opens the link | Sign-in is not possible for such an identity (specification 001), so only the pre-match state is reachable; acceptance impossible (RF-5.3; A-27). |
| LC-29 | Review state reloaded or left idle | Invitation unchanged; the page re-resolves the link under RF-1 (RF-2.6). |
| LC-30 | Back navigation after acceptance | Re-opening the link shows "can't be accepted"; the membership exists (RF-5.2). |
| LC-31 | Session expires on the review state | The expiry handling of specification 006 applies; nothing consumed; after re-authentication the link is re-resolved (RF-2.6). |
| LC-32 | Language switched on any state | Display changes only (RF-1.8). |
| LC-33 | Third party floods link openings | Throttled per network; the invited identity's sign-in untouched; windows lift (RF-9.1, RNF-7). |
| LC-34 | Email channel down when the invitation was issued | The link never arrives; nothing in this feature changes; an operational event of specification 010 (RNF-15). |
| LC-35 | Tenant Admin looks for a way to accept on the invitee's behalf | None exists on any tenant surface; only the invited identity accepts (RF-7.1, RNF-3). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **Issuing an invitation, the pending-invitations view, "Review invitation" and revoking a pending invitation** — tenant administration, specification 010 (FR-IDN-097; the "Invite member" dialog is drawn in D02 `419:2645`, S-04).
- **The invitation message** that carries the link, and any notification to the inviter — specification 010 and the S-14 template inventory (decision Q7).
- **Ownership** — an invitation never grants Owner; ownership is obtained through transfer (F-IDN-12, specification 010).
- **Signup, verification and sign-in themselves** — specifications 002 and 001; this feature adds only the invited signup variant (RF-4.1) and the "Return to your invitation" routes (RF-4.4).
- **TOTP enrollment and the gate at tenant entry** — specification 003; this feature only states where the gate applies (RF-6.7).
- **Logout** — specification 007; "Switch identity safely" only invokes it.
- **The guarded tenant and project switch** — specification 016; acceptance only enters it (RF-6.5).
- **Editing a member's role or project grants after acceptance** — specifications 010 and 011.
- **Invitations carrying several projects, custom roles or team membership** — not part of the product (FR-IDN-090; scope §2).
- **Federated sign-in and account linking bound to an invitation** — FR-EXT-021, deferred; not in this release.
- **Tablet-specific designs** of the invitation states — none; the D01 responsive policy applies.
- **The content of the Help panel** — "Help" on the page opens N43 HelpShortcutsPanel (`430:4129`; S-17); the panel is the shell's, not this feature's.
- **The choice and configuration of the transactional email channel** — an operational prerequisite, not a feature.

---

## 7. Criteria of completion

Invitation acceptance is complete when all of the following hold:

1. Every user story US-1 … US-8 can be walked end to end on the desktop design, on M03 and in the reflows of the other states with no step outside this specification, and US-9 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-11 has at least one verification that fails before and passes after the feature exists.
3. Single-use tests show that an invitation is accepted at most once, that two concurrent acceptances, or an acceptance racing with a decline, a revocation or expiry, yield exactly one transition, and that an accepted or declined link never works again (RNF-1, LC-9 … LC-12).
4. Identity tests show that only a complete identity session of the invited identity can accept, that a mismatched identity, a signed-out holder and a forwarded link each get the designed refusal with no tenant content, and that no path links identities by matching email (RF-3, RNF-3, LC-3, LC-5).
5. Scope tests show that the membership and grant records created equal the invitation element by element, that client-supplied values are ignored, that no Owner role, extra project or unissued grant can result, and that a revoked membership is replaced without restoring anything (RNF-4, RF-6.2, RF-6.3, LC-17, LC-21, LC-22).
6. Validity tests show the "can't be accepted" state for every cause of RF-9.2, that a suspended tenant's invitation waits and a deleting tenant's is final, that an existing active or suspended membership blocks acceptance, and that the checks are re-evaluated at the moment of acceptance (RF-5, LC-7, LC-8, LC-13 … LC-16).
7. Enumeration tests show no observable difference — content, shape, timing within normal variance — between the causes of the "can't be accepted" state, and that no state reveals tenants, addresses, factors or other invitations (RNF-2, RF-9.3).
8. New-identity tests show that signup from the page refuses any address other than the invited one, that signup, verification and sign-in leave the invitation pending, and that both "Return to your invitation" and re-opening the link lead to the review state (RF-4, LC-24 … LC-27).
9. MFA tests show that an invited Admin, or a policy-required Member, without a factor accepts and is gated at tenant entry with authority withheld, and that no path grants authority earlier (RF-6.7, RNF-11, LC-18, LC-19).
10. Throttle tests show progressive delay, bounded suppression, automatic lifting and no effect on the invited identity's sign-in (RF-9.1, RNF-7, LC-33).
11. The product refuses to start when the invitation expiry or any threshold, delay or window of this feature is absent or unbounded (RNF-6).
12. Audit entries exist for every event of RF-11.1 and contain no link secret, unmatched address or message body (RF-11.2); no message is sent on acceptance or decline (RF-11.3).
13. The four states pass WCAG 2.2 AA with automated evidence and manual assistive-technology evidence, in EN, ES and FR, at desktop, tablet and mobile widths (RNF-12 … RNF-14).
14. The register items this specification relies on — A-52, A-28 and A-27 in the architecture register; S-45, S-46, S-24, S-03, S-04, S-14 and S-17 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. Constitution v1.8 (2 November 2026) aligns F-IDN-10 with decisions Q3 and Q6 (masked invited address; MFA gated at tenant entry).
