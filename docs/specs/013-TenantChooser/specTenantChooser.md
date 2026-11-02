# Specification 013 — Tenant chooser

**Feature** The page where a signed-in identity sees the tenants it belongs to, with its role, its membership state and its own project grants in each, and chooses which one to enter · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-15 as the subject; F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-14, F-IDN-16, F-IDN-17, F-IDN-18 and F-SET-04 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-07; principles P1, P3, P6, P12, P13, P14) → `requirements.md` (FR-IDN-001, 002, 012, 023, 024, 026, 030, 061, 064, 066, 067, 068, 090, 091, 092, 093, 098, 110, 111, 112, 113, 116, 121, 130, 131, 132; FR-PRJ-110, 111, 112; FR-X-001, 030, 034, 040, 042, 043, 044, 050, 051, 053, 055) → the design of record: screen N03 TenantChooserPage `161:49326`, the suspended-tenant screen N20 TenantSuspendedPage `161:50995` it links to, and the compact state N24 TenantSwitchDirtyDialog of the D02 interaction atlas `159:32045` that guards a switch made from inside a workspace.

**Related specifications:** 001 Login (the landing rule RF-8 that enters a single active membership directly and otherwise routes here; the identity session that carries no tenant authority) · 003 TOTP enrollment, recovery codes and factor replacement (the enrollment gate applied when a tenant is entered, RF-1.3 and RF-1.4; "Leave setup" lands here, 003 Q20) · 006 Sessions, refresh and expiry (a re-authentication whose interrupted tenant is no longer derivable lands here, 006 RF-8.4 and RF-8.7) · 007 Logout and session management ("Sign out" on this page, 007 RF-9.1; the shell identity area where this specification adds "Switch workspace") · 008 Invitation acceptance ("Leave invitation", "Return to memberships" and a decline land here; a new membership appears here) · 009 Roles, grants and the authority matrix (the meaning of active, suspended and revoked memberships, 009 Q3; what each surface must state, 009 RF-10.4) · 010 Tenant members and ownership transfer (the administration that changes what this page shows) · 012 Workspace creation and bootstrap ("Create workspace") · 014 Tenant suspension state ("View access status") · 016 Guarded tenant and project switch (the transition a choice made from inside a workspace runs) · the projects area (the tenant overview R10 / M04 that a chosen tenant opens, FR-PRJ-110 … 112).

This document states **what** the tenant chooser does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global** and may belong to several tenants, each through its own membership with a tenant role (Owner, Admin, Member), a membership state (active, suspended, revoked) and its own explicit project grants (Author, Reviewer, Viewer, `canRun`) inside that tenant (FR-IDN-002, FR-IDN-090, FR-IDN-093). A signed-in identity holds an **identity session** that carries no tenant or project authority; authority exists only after a tenant is entered and is derived on the server at every request (FR-IDN-024; specification 001 RF-8.1).

The **tenant chooser** is the page between the two: it shows the signed-in identity the tenants it belongs to — and nothing else — and lets it choose one to enter. For each tenant it shows the person's own role, the state of their membership or of the tenant, and the projects where they hold a grant with exactly those grants, so that the person knows, before entering, what they will and will not be able to do. A suspended tenant, a suspended membership and a tenant being deleted appear with their state and no content; a revoked membership does not appear at all (specification 009 Q3). The page states the two boundaries the product is built on: tenant administration does not imply project content access, and changing workspace context never retargets a previously accepted Run.

The chooser is reached after sign-in when the landing rule does not enter a tenant directly, after a re-authentication whose interrupted tenant can no longer be derived, when a person leaves a forced enrollment, an invitation or a declined invitation, and — by this specification — from inside a workspace through a "Switch workspace" entry. Entering a tenant from it is the trigger of the guarded tenant switch (F-IDN-18) and of the enrollment gate (specification 003); the chooser itself grants nothing.

### 1.2 Objective

Let a signed-in person see exactly where they belong and choose where to work, while:

- listing only the signed-in identity's own memberships, with its own role, state and project grants, and never a tenant it does not belong to, another member, or content of a tenant it cannot enter;
- showing a suspended tenant, a suspended membership and a deleting tenant with their access state only, and a revoked membership not at all;
- making "Enter" a request whose authority is derived on the server from the current membership and tenant state, so that a stale page can never enter a tenant that is no longer enterable;
- stating, before any choice, that administration is not project access and that a change of context is not a new target for accepted Runs;
- offering, on every rendering, the two other ways forward — a valid invitation and workspace creation — and a way out — sign-out;
- keeping the page honest at every width and in every language without ever deciding authority itself.

### 1.3 Why it matters

The chooser is the first thing a multi-tenant user sees after signing in and the last thing between an identity and a customer's data. A chooser that lists a tenant the person was removed from, shows project names of a suspended tenant, or enters a tenant on the strength of a page loaded an hour ago turns a membership change into a leak. A chooser that hides the person's role and grants sends an Admin into a tenant expecting to read projects they were never granted, and a Member with `canRun` but no role into a project expecting to launch. And a chooser that cannot be reached from inside a workspace leaves a consultant with three customers signing out and in to move between them. The page must be exact about the person, silent about everyone else, and never a source of authority.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, all written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14: notes appended to **A-36** (identity-scoped membership listing) and **A-54** (meaning of the suspended and revoked membership statuses). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-55** (deleting-tenant card on N03, `401:2922`; the deleting wording of the suspended-tenant screen itself, N20 `401:2714`, is owned by specification 014) and **S-56** (N03 card variants: Member-policy and not-yet-enrolled MFA lines, `canRun`-only and no-grants project lines, suspended-membership card content) are new; notes are appended to **S-07** (app-shell identity area), **S-48** (suspended-membership card), **S-02** and **S-03** (responsive counterparts).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification and how the chooser is reached | The page, its cards, its statements and the "Enter" trigger. The chooser is reached from sign-in landing (001), from a re-authentication whose interrupted tenant is no longer derivable (006), from "Leave setup" (003), from "Leave invitation", "Return to memberships" and a decline (008), and — added here — from the **"Switch workspace"** entry of the shell user menu (`395:668`), beside the "Sign out" action of specification 007, and from the header WORKSPACE selector (`395:667`), which is the same guarded route (specification 016 Q1; S-07). A choice made from inside a workspace runs the guarded transition of F-IDN-18 (specification 016); a choice made from a fresh landing has nothing to guard. The suspended-tenant screen is specification 014, workspace creation 012, sign-out 007, and the tenant overview that opens after entry belongs to the projects area. |
| Q2 | What "Your project grants" lists on an enterable card | Every project of that tenant where the person holds at least one live grant — Author, Reviewer, Viewer or `canRun` — with exactly their own grants. A project where only `canRun` is held reads "canRun only · no project role yet" (specification 009 RF-10.8). A person with no grant in the tenant reads "No project grants yet — workspace administration does not grant project content" and may still enter. The card never shows other projects, counts, sources, runs, or another member's grants. |
| Q3 | When the chooser is shown, and the empty state | Reached by explicit navigation — "Switch workspace", "Leave setup", the exits of 008, a re-authentication fallback — the chooser always renders, with one membership or none; direct entry of a single active membership and routing to workspace creation when no membership exists are rules of sign-in landing only (001 RF-8). With no membership the page shows the "No membership for a workspace?" block alone. "Create workspace" is offered on every rendering; specification 012 enforces its own enrollment precondition. |
| Q4 | What "Enter" re-checks, and a tenant being deleted | "Enter" is a request whose authority is derived on the server from the current membership and tenant state; if either is no longer enterable, the list is refreshed and the card renders its new state — suspended membership, suspended or deleting tenant, or the tenant gone when the membership was revoked — with no separate error. A deleting tenant is listed like a suspended one: name, own role, "Deleting · content unavailable", no project names, no "Enter"; "View access status" opens the suspended-tenant screen of specification 014 in its deleting wording. Entering an enterable tenant applies the enrollment gate of specification 003 on arrival. |
| Q5 | The MFA line on a card | The requirement is stated per card from the role and the tenant policy: "TOTP is required for your Owner role." / "… Admin role." always; "TOTP is required by this workspace's policy." for a Member under a policy that requires MFA; no line for a Member otherwise. When the line applies and the person holds no active factor, it adds that enrollment will be required on entry. The chooser never blocks "Enter" for it and never offers enrollment itself; the gate of specification 003 applies on arrival. The person's own factor status is identity-scoped and may be shown; nothing about other members is. |
| Q6 | The non-enterable cards and "View access status" | Suspended tenant: as drawn, with "View access status" opening the suspended-tenant screen of specification 014, which shows the person's own membership metadata and its routes without deriving tenant authority or counting as an entry; the screen offers a route back to the chooser when the person holds an active membership in another active tenant (specification 014 RF-3.4); otherwise the chooser stays reachable by returning to its route, where it re-derives and shows the single card. Suspended membership in an active tenant: the tenant name, "Your membership is suspended · content unavailable", the person's role, no project names, no action; no administrator is named and no contact implied; the person's other tenants remain enterable. |
| Q7 | Order, volume and freshness | Cards are ordered: enterable memberships first, then suspended memberships, then suspended and deleting tenants; alphabetical by tenant name within each group; projects alphabetical within a card. Every membership is rendered; there is no search, filter or paging in this release. The list is derived on the server each time the page opens and again on "Enter"; there is no polling and no push — a change while the page is idle appears on the next open or "Enter". |

Five rules were derived from these decisions and from earlier specifications rather than asked; the product owner may overturn them:

- **D-a** A revoked membership is never listed, even when it was the person's only one: specification 009 Q3 (register A-54) governs, and the chooser then shows the empty block. Specification 001 LC-15 says the chooser "shows its state" for a single revoked membership; that wording predated 009 and was amended on 27 October 2026 to match.
- **D-b** The page shows only the signed-in identity's own facts — address, display name, tenant roles, membership states, project grants, factor status — and the names of the tenants and projects it may reach; it never shows another member, a member count, an administrator, a source, a run or any content.
- **D-c** The language switch changes display only; nothing listed or chosen changes.
- **D-d** "Sign out" on the chooser is specification 007 RF-9.1: it ends the session with no guard, because the page holds no unsaved work (007 RF-3.5).
- **D-e** "Create workspace" from the chooser carries no tenant context and copies nothing from any membership; it is the same entry the landing rule uses (specification 012).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | person who belongs to several tenants | to see each tenant with my role, my membership state and my own project grants, and enter the one I need | I know before entering what I can do there |
| US-2 | Owner or Admin with no project grants | to be told on the card that administration does not give me project content | I do not enter expecting to read projects |
| US-3 | Member whose tenant is suspended | to see the tenant with its access state and read my status, without seeing any content | I understand why I cannot work and what remains protected |
| US-4 | Member whose membership was suspended | to see the tenant marked as suspended for me, while my other tenants stay enterable | I am not locked out of everything because of one tenant |
| US-5 | person working inside a workspace | to switch to another workspace from the shell, with my unsaved work guarded | I do not sign out and in to move between customers |
| US-6 | Owner who has not enrolled an authenticator | to be told on the card that TOTP will be required when I enter | the enrollment step on arrival is not a surprise |
| US-7 | person with no membership | to be told that only a valid invitation or a new workspace can give me one, and to create one | I am not stranded on an empty page |
| US-8 | person with an accepted Run in another tenant | to read that choosing a workspace never retargets my accepted Runs | I switch without fear of moving a test to the wrong target |
| US-9 | person on a phone or tablet | the chooser to reflow and keep every card, statement and action | I can choose a workspace anywhere |
| US-10 | attacker, removed member or curious colleague (negative story) | — | I cannot see a tenant I do not belong to, cannot see project names of a suspended tenant or of a project I have no grant in, cannot enter a tenant from a stale page after my membership changed, cannot learn who else is a member, and cannot make the page grant any authority |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the person* means the signed-in identity of the current identity session; *membership* means the person's membership record in a tenant, with its state active, suspended or revoked (specification 009); *tenant state* means active, suspended or deleting; *enterable* means an active membership in an active tenant; *a card* means the block the chooser renders for one membership; *own grants* means the project grant records held by the person's membership in that tenant; *explicit navigation* means any route to the chooser other than sign-in landing.

### RF-1 · Reaching the chooser

The chooser is a page of the identity session, before any tenant authority exists. *Why:* FR-IDN-024 — the session is not a C10-authorised customer session; the chooser must work without one and grant none. *(Decision: Q1, Q3.)*

- RF-1.1 The system shall present the chooser only to a signed-in identity; a person without a session who reaches its route shall be sent to the sign-in screen, and a session that has ended shall be handled by specification 006. *(FR-IDN-024, FR-IDN-061)*
- RF-1.2 The system shall reach the chooser from: sign-in landing when the landing rule does not enter a tenant directly (specification 001 RF-8.4); a re-authentication whose interrupted tenant is no longer derivable (specification 006 RF-8.4, RF-8.7); "Leave setup" of a forced enrollment (specification 003 RF-2.5); "Leave invitation", "Return to memberships" and a decline (specification 008 RF-3.3, RF-5.6, RF-7.2); and the "Switch workspace" entry of RF-1.4. *(FR-IDN-110; decision Q1)*
- RF-1.3 When the chooser is reached by explicit navigation, the system shall render it whatever the number of memberships — several, one or none — and shall neither enter a tenant nor route to workspace creation on its own; those are rules of sign-in landing only. *(decision Q3; specification 001 RF-8.2, RF-8.3)*
- RF-1.4 The system shall present a "Switch workspace" action in the identity area of the application shell, beside the "Sign out" action of specification 007 RF-1.2, on every screen reached with a session inside a tenant, in the user menu on desktop (`395:668`) and in the mobile menu (`395:669`); the header WORKSPACE selector (`395:667`) opens the same route (specification 016 Q1). *(FR-IDN-130; decision Q1; S-07)*
- RF-1.5 When "Switch workspace" is activated inside a workspace, the system shall run the guarded transition of F-IDN-18 (specification 016) — the dirty-editor guard with Save / Discard / Stay first, a guard save under the original tenant and project — and shall open the chooser only after the guard is satisfied; the current tenant context ends when a different tenant is entered, not when the chooser opens. *(FR-IDN-130, FR-IDN-131; decision Q1)*
- RF-1.6 While the chooser is shown, the system shall hold no tenant or project authority for the person; every element on the page is derived from identity-scoped facts. *(FR-IDN-024, FR-X-001; A-36)*

### RF-2 · The page

One page, one identity, two boundaries stated. *Why:* FR-IDN-110 and FR-IDN-112 fix what the page must list and say. *(Decision: Q3.)*

- RF-2.1 The system shall present the chooser with: the product mark; the person's display name and the interface-language switch (EN / ES / FR); the marker "YOUR IDENTITY / WORKSPACE MEMBERSHIPS"; the heading "Choose your workspace."; the statement "Only memberships for <address> are shown. Project access is checked separately after you choose a workspace." with the person's verified address; one card per listed membership (RF-3 … RF-5); the block "A context change is not a new target" with "Previously accepted runs keep their original tenant, project and environment pins. Choosing another workspace never retargets them."; the block "No membership for a workspace?" with "Use a valid invitation for your verified identity or create a new workspace. Workspaces you do not belong to are never listed." and the action "Create workspace"; and the action "Sign out". *(FR-IDN-110, FR-IDN-112; N03)*
- RF-2.2 The system shall offer "Create workspace" on every rendering of the page, with or without memberships; the action shall lead to workspace creation (specification 012) carrying no tenant context. *(FR-IDN-113; decision Q3; D-e)*
- RF-2.3 When "Sign out" is activated, the system shall end the session as specification 007 RF-9.1 defines, with no guard; nothing about any membership changes. *(FR-IDN-068; D-d)*
- RF-2.4 When the language switch is used, the system shall change the display only; nothing listed, derived or chosen shall change. *(FR-X-043; D-c)*
- RF-2.5 The system shall render a "Help" action on the page opening N43 HelpShortcutsPanel (`430:4129`) (S-17), and shall render no Privacy or Terms route, none being drawn on N03.
- RF-2.6 The system shall show on the page nothing about any identity other than the person, nothing about any member of any tenant, no member count, no administrator, no source, no run and no content; the tenant names and project names of RF-3 and RF-4 are the only customer data on the page. *(FR-IDN-110, FR-IDN-111; D-b)*

### RF-3 · The listing rule

Only the person's memberships, in a fixed order, derived fresh. *Why:* FR-IDN-110 — tenants the identity does not belong to are never listed; specification 009 Q3 fixes what each state means. *(Decision: Q2, Q7; D-a.)*

- RF-3.1 The system shall derive the list on the server, for the signed-in identity only, each time the page opens: every membership of the person in state *active* or *suspended*, together with the tenant's name and state, the person's tenant role, and, for an enterable membership, the person's own live project grants with the project names. *(FR-IDN-110; A-36)*
- RF-3.2 The system shall not list a membership in state *revoked*, whatever other memberships exist; a person whose only membership was revoked sees the empty block of RF-2.1. *(specification 009 Q3; A-54; D-a)*
- RF-3.3 The system shall not list any tenant in which the person holds no membership record, and shall expose no fact that such a tenant exists. *(FR-IDN-110)*
- RF-3.4 The system shall render each listed membership as exactly one of four cards: *enterable* (RF-4), *suspended tenant* (RF-5.1), *suspended membership* (RF-5.2), *deleting tenant* (RF-5.3); the tenant state takes precedence over the membership state when both are not active. *(FR-IDN-093, FR-IDN-111; decisions Q4, Q6)*
- RF-3.5 The system shall order the cards: enterable first, then suspended memberships, then suspended and deleting tenants; alphabetically by tenant name within each group, in the collation of the interface language. *(decision Q7)*
- RF-3.6 The system shall render every listed membership; no search, filter or paging exists. *(decision Q7; S-56)*
- RF-3.7 The system shall not refresh the list on its own while the page is idle — no polling, no push; the list is derived again on the next open and on every "Enter" (RF-6.3). *(decision Q7)*

### RF-4 · The enterable card

What the person will hold, before they hold it. *Why:* FR-IDN-110 lists the role, the state and the eligible projects with their grants; specification 009 RF-10.4 adds the statements. *(Decision: Q2, Q5.)*

- RF-4.1 The enterable card shall present: the tenant name; the state "Active membership"; the label "Your role" with the person's tenant role (Owner, Admin or Member); the heading "Your project grants" with one line per project of RF-4.2; the statement "Workspace administration does not grant project content."; the MFA line of RF-4.4 where it applies; and the action "Enter <tenant> →". *(FR-IDN-110; N03; specification 009 RF-10.4)*
- RF-4.2 The system shall list under "Your project grants" every project of the tenant in which the person holds at least one live grant, each as the project name followed by the person's own grants verbatim — the roles held among Author, Reviewer and Viewer, and "canRun" when held — alphabetically by project name. *(FR-IDN-110, FR-IDN-098; decision Q2)*
- RF-4.3 Where a project line holds `canRun` and no project role, the system shall render it as "<project> · canRun only · no project role yet"; where the person holds no grant in the tenant, the system shall render in place of the project lines "No project grants yet — workspace administration does not grant project content" and shall still offer "Enter". *(specification 009 Q2, RF-10.8; decision Q2; S-56)*
- RF-4.4 The system shall render the MFA line from the role and the tenant policy: "TOTP is required for your Owner role." or "… Admin role." for an Owner or Admin; "TOTP is required by this workspace's policy." for a Member in a tenant whose policy requires MFA of Members; no line for a Member otherwise. *(FR-IDN-023, FR-IDN-030; decision Q5; S-56)*
- RF-4.5 Where the MFA line applies and the person holds no active factor, the line shall add "Not yet set up — you will enroll when you enter."; the card shall still offer "Enter". *(FR-IDN-026; decision Q5; S-56)*
- RF-4.6 The card shall show nothing of the tenant beyond its name, the person's role and the person's eligible projects with their own grants: no other project, no member, no policy detail beyond the MFA line, no source, run or count. *(FR-IDN-110; D-b)*

### RF-5 · The non-enterable cards

State, not content. *Why:* FR-IDN-111 — a suspended tenant appears with its access state and exposes no project names, sources or run details; specification 009 Q3 extends the rule to a suspended membership. *(Decision: Q4, Q6.)*

- RF-5.1 The *suspended tenant* card shall present: the tenant name; the state "Workspace suspended"; the heading "Your current membership" with the person's role; the statement "Project names and content are not exposed while this workspace is suspended."; and the action "View access status". It shall offer no "Enter". *(FR-IDN-111, FR-IDN-116; N03)*
- RF-5.2 The *suspended membership* card (the tenant active, the membership suspended) shall present: the tenant name; the state "Your membership is suspended · content unavailable"; the person's role; the statement "Project names and content are not exposed while your membership is suspended."; and no action. It shall name no administrator and imply no contact. *(FR-IDN-093, FR-IDN-111; specification 009 Q3; decision Q6; S-48, S-56)*
- RF-5.3 The *deleting tenant* card shall present: the tenant name; the state "Deleting · content unavailable"; the person's role; the statement "Project names and content are not exposed while this workspace is being deleted."; and the action "View access status". It shall offer no "Enter". *(FR-IDN-093; decision Q4; S-55)*
- RF-5.4 When "View access status" is activated, the system shall open the suspended-tenant screen of specification 014 for that tenant — in its deleting wording for a deleting tenant — showing only the person's own membership metadata and its routes; opening it shall derive no tenant authority and shall not count as entering the tenant; the screen offers a route back to the chooser only when the person holds an active membership in another active tenant (specification 014 RF-3.4), and the chooser's route stays reachable in every case, re-deriving on return. *(FR-IDN-111, FR-IDN-116; decisions Q4, Q6; S-55)*
- RF-5.5 The non-enterable cards shall show no project name, grant, source, run, member or count. *(FR-IDN-111; D-b)*
- RF-5.6 While one of the person's tenants is suspended or deleting, or one membership is suspended, the person's other enterable memberships shall remain enterable and unaffected. *(FR-IDN-002; decision Q6)*

### RF-6 · Entering a tenant

A choice is a request; the server decides. *Why:* P3 and FR-X-001 — the browser expresses intent; authority is derived at the effect boundary from current facts, so a stale page cannot enter a tenant. *(Decision: Q1, Q4.)*

- RF-6.1 When "Enter <tenant>" is activated, the system shall request entry into that tenant for the person; the request shall carry the tenant as requested scope only, and the server shall derive whether the person's membership is active, the tenant active and what authority the membership holds, at that moment. *(FR-IDN-130, FR-X-001; P3)*
- RF-6.2 When the request is granted, the system shall enter the tenant through the guarded transition of F-IDN-18 (specification 016): when the chooser was opened from inside another tenant, that tenant's context ends and the new one begins with a fresh membership and grant fetch; when the chooser was opened with no tenant context, the transition reduces to the fresh fetch. The destination shall be the tenant's overview as the projects area defines it. *(FR-IDN-130, FR-IDN-132; FR-PRJ-110; decision Q1)*
- RF-6.3 If the request is refused because the membership is no longer active or the tenant no longer active, then the system shall derive the list again and render the chooser with the card in its new state — suspended membership, suspended or deleting tenant, or absent when the membership was revoked — with no separate error message and no entry. *(FR-IDN-067, FR-IDN-093; decision Q4; P6)*
- RF-6.4 When the entered tenant requires a factor the person does not hold — Owner or Admin role, or Member under a policy requiring MFA — the system shall apply the enrollment gate of specification 003 RF-1.3 and RF-1.4 on arrival: the membership exists, its authority is withheld and the enrollment wizard is the only way forward, with "Leave setup" returning to the chooser. The chooser shall never block the entry for it and never offer enrollment itself. *(FR-IDN-023, FR-IDN-026, FR-IDN-030; decision Q5; specification 003 Q20)*
- RF-6.5 Entering a tenant shall grant visibility only of what the person's current membership and grants permit, derived on the server; it shall never restore access a change has since removed, never accept a mapping, approve a bundle, launch a Run or replay a Run intent, and never retarget an accepted Run. *(FR-IDN-067, FR-IDN-112; P1)*
- RF-6.6 Entering a tenant shall change nothing about the membership, the grants, the person's other memberships or the identity; it is a change of context, not of authority. *(FR-IDN-110)*

### RF-7 · Freshness

The page is as current as its last derivation, and says nothing it cannot stand behind. *Why:* FR-IDN-067 and FR-IDN-132 — stale state is never authority; the chooser must be cheap to render and impossible to trust beyond "Enter". *(Decision: Q7.)*

- RF-7.1 The system shall derive the list on every open of the page, including a return from the suspended-tenant screen, from workspace creation abandoned, from "Leave setup" and after a re-authentication. *(FR-IDN-067; decision Q7)*
- RF-7.2 While the page is open, a change of membership, grant, tenant state or factor shall not be pushed to it; the next open or "Enter" shall reflect it. *(decision Q7)*
- RF-7.3 The system shall not keep the derived list in browser storage; a reload derives it again. *(FR-X-034)*
- RF-7.4 When the identity's session ends while the chooser is open, the next request shall meet the handling of specification 006; a successful re-authentication shall return to the chooser with a fresh derivation. *(FR-IDN-061, FR-IDN-066; specification 006 RF-8.7)*

### RF-8 · Statements

The page carries the product's promises in the person's words. *Why:* FR-IDN-112 and specification 009 RF-10.4 require them here; FR-PRJ-111 repeats one of them on the overview that follows.

- RF-8.1 The system shall state on the page that only memberships for the person's address are shown and that project access is checked separately after a tenant is chosen. *(FR-IDN-110; N03; specification 009 RF-10.4)*
- RF-8.2 The system shall state on every enterable card that tenant administration does not imply project content access. *(FR-IDN-092; N03; specification 009 RF-10.4)*
- RF-8.3 The system shall state that previously accepted runs keep their original tenant, project and environment pins and that choosing another workspace never retargets them. *(FR-IDN-112; N03)*
- RF-8.4 The system shall state that a membership is obtained only through a valid invitation for the verified identity or by creating a new workspace, and that workspaces the person does not belong to are never listed. *(FR-IDN-015, FR-IDN-110; N03)*
- RF-8.5 The statements shall be identical for every role, state and grant variant of the page and at every viewport width. *(FR-X-055; specification 009 RF-10.9)*

### RF-9 · Responsive surface

M20 MobileTenantChooserPage (`432:4257`) is the drawn counterpart; tablet follows the D01 responsive policy. *Why:* FR-X-051 and FR-X-055 — AA reflow on every critical-journey surface. *(S-02, S-03.)*

- RF-9.1 The system shall present the chooser and all its card variants in a reflowed layout at mobile and tablet widths that meets WCAG 2.2 AA reflow, with every card, statement, action and block of RF-2 … RF-5 preserved in meaning; M20 MobileTenantChooserPage (`432:4257`) is the drawn counterpart, tablet follows the D01 responsive policy (`414:2645`). *(FR-X-051, FR-X-055; S-02, S-03)*
- RF-9.2 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus; cards shall be reachable and operable by keyboard in the order of RF-3.5. *(FR-X-042, FR-X-053)*
- RF-9.3 At mobile widths the system shall follow M20 MobileTenantChooserPage (`432:4257`); at tablet widths the D01 responsive policy. *(FR-X-050; S-03)*

### RF-10 · Audit, secrecy and disclosure

The page reveals the person to themselves and no one to anyone. *Why:* FR-IDN-110 and FR-IDN-111 are disclosure rules; FR-X-044 wants the refusals that matter on record.

- RF-10.1 The system shall append to the audit log each entry request refused under RF-6.3, with the identity, the requested tenant, the cause (membership state, tenant state, revoked) and the calling context; a granted entry is recorded by the tenant-entry derivation itself, not by this feature. *(FR-X-044)*
- RF-10.2 The system shall write nothing of the chooser — no list, no tenant name, no grant — to browser storage, logs, client-visible problems or analytics beyond safe codes with a correlation id. *(FR-X-030, FR-X-033, FR-X-034)*
- RF-10.3 Every fact rendered on the page shall be read for the signed-in identity only, through the identity-scoped listing of register A-36; the page shall never be the result of a tenant-scoped read made on the person's behalf before entry. *(FR-IDN-110; P12; A-36)*
- RF-10.4 The system shall expose, for a suspended or deleting tenant, no fact beyond its name, the person's role and the state; for a suspended membership, none beyond the tenant name, the role and the state. *(FR-IDN-111)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Own facts only.** Every element of the page derives from the signed-in identity's own memberships, grants and factor status; no element derives from another identity, another member or a tenant the person does not belong to. | FR-IDN-110; A-36 |
| RNF-2 | **No authority from the page.** Neither rendering the chooser nor choosing a card grants tenant or project authority; "Enter" is a request decided on the server from current facts. | FR-IDN-024, FR-X-001; P3 |
| RNF-3 | **Fail closed.** A membership not active, a tenant not active, a revoked membership, an ended session or an absent derivation blocks entry and never shows content; nothing is assumed. | P6 |
| RNF-4 | **Freshness bound.** The list is never older than the last open or "Enter"; no cached list is ever used to decide an entry. | FR-IDN-067, FR-IDN-132 |
| RNF-5 | **State-only exposure.** A suspended or deleting tenant and a suspended membership expose their state and the person's role only. | FR-IDN-111 |
| RNF-6 | **Exact grants.** Project lines equal the person's live grant records element by element, including the `canRun`-only case; no grant is inferred from a role. | FR-IDN-090, FR-IDN-098; 009 Q2 |
| RNF-7 | **No Run effect.** No action on the page accepts, approves, launches, replays or retargets anything. | FR-IDN-112, FR-IDN-067; P1 |
| RNF-8 | **Guarded switch.** A choice made from inside a workspace never loses unsaved work without the Save / Discard / Stay guard and never submits a guard save under the new tenant. | FR-IDN-130, FR-IDN-131 |
| RNF-9 | **No configuration.** The chooser introduces no numeric value of its own; the bounds of the guarded switch and of sessions belong to specifications 016 and 006. | P13 |
| RNF-10 | **Secrecy.** Nothing of the page reaches browser storage, logs, problems or analytics beyond safe codes and a correlation id. | FR-X-030, FR-X-034 |
| RNF-11 | **Accessibility.** The page and every card variant meet WCAG 2.2 AA with automated and manual assistive-technology evidence: cards as labelled regions, state conveyed by text, keyboard order per RF-3.5, no focus stealing on refresh. | FR-X-040, FR-X-042 |
| RNF-12 | **Language.** The page and every statement exist in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-13 | **Reflow.** Mobile and tablet reflows preserve every card, statement and action of the desktop page. | FR-X-051, FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Sign-in with two active memberships in active tenants | Landing routes here (001 RF-8.4); two enterable cards (RF-1.2, RF-4). |
| LC-2 | "Switch workspace" with a single enterable membership | The chooser renders with one card; no auto-entry (RF-1.3). |
| LC-3 | "Switch workspace" with no membership at all | The empty block with "Create workspace"; no auto-routing to creation (RF-1.3, RF-2.1). |
| LC-4 | Only membership revoked; chooser reached by any route | Not listed; empty block (RF-3.2; D-a). |
| LC-5 | One suspended membership and two enterable ones | Three cards: two enterable first, then the suspended-membership card with no action (RF-3.5, RF-5.2, RF-5.6). |
| LC-6 | One membership, tenant suspended | Suspended-tenant card with "View access status"; "Create workspace" and "Sign out" (RF-5.1, RF-2.1). |
| LC-7 | One membership, tenant deleting | Deleting-tenant card; "View access status" in deleting wording (RF-5.3, RF-5.4). |
| LC-8 | Tenant suspended while the chooser is open; "Enter" activated | Refused; list re-derived; the card now reads "Workspace suspended"; no error message (RF-6.3). |
| LC-9 | Membership revoked while the chooser is open; "Enter" activated | Refused; list re-derived; the card is gone (RF-6.3, RF-3.2). |
| LC-10 | Membership suspended while the chooser is open; "Enter" activated | Refused; the card now reads suspended membership (RF-6.3, RF-5.2). |
| LC-11 | Membership reinstated while the chooser is open | Nothing changes until the next open or "Enter"; then the card is enterable (RF-7.2). |
| LC-12 | Owner without a factor enters | Entry granted; enrollment gate on arrival; "Leave setup" returns here (RF-6.4, RF-4.5). |
| LC-13 | Member in a tenant whose policy requires MFA, no factor | MFA line with "Not yet set up"; same gate on entry (RF-4.4, RF-4.5, RF-6.4). |
| LC-14 | Member in a tenant without such a policy | No MFA line; entry without gate (RF-4.4). |
| LC-15 | Admin with no project grants | "No project grants yet …" line; "Enter" offered (RF-4.3). |
| LC-16 | Member with `canRun` only on one project | "<project> · canRun only · no project role yet" (RF-4.3). |
| LC-17 | Member with Author on one project and Viewer on another | Two project lines, alphabetical, grants verbatim (RF-4.2). |
| LC-18 | Project grant revoked while the chooser is open; "Enter" activated | Entry granted (the membership is active); the overview that opens derives current grants and no longer shows that project (RF-6.5). |
| LC-19 | Twenty memberships across every state | All rendered in the order of RF-3.5; no paging (RF-3.5, RF-3.6). |
| LC-20 | Two tenants with the same name | Both cards shown, ordered by the collation tie-break; each enters its own tenant (RF-3.5, RF-6.1). |
| LC-21 | "View access status" then return | Chooser re-derived on return; no tenant entered; the return route on N20 exists only when another active membership exists, otherwise the chooser is reached by its route (RF-5.4, RF-7.1; 014 RF-3.4). |
| LC-22 | "Create workspace" while holding memberships | Creation entry opens with no tenant context (RF-2.2; D-e). |
| LC-23 | "Sign out" on the chooser | Session ended; sign-in screen; nothing else changes (RF-2.3). |
| LC-24 | Language switched | Display only; same cards (RF-2.4). |
| LC-25 | Chooser reached by "Leave setup" from a forced enrollment | Rendered fresh; the gated tenant still enterable, its card still showing "Not yet set up" (RF-1.2, RF-7.1). |
| LC-26 | Chooser reached by a decline of an invitation | Rendered fresh; the declined tenant absent (RF-1.2). |
| LC-27 | Chooser reached after accepting an invitation and leaving the tenant through "Switch workspace" | The new membership listed as enterable (RF-1.2, RF-3.1). |
| LC-28 | "Switch workspace" with a dirty editor | Save / Discard / Stay before the chooser opens; a save under the original tenant (RF-1.5, RNF-8). |
| LC-29 | Re-authentication after the interrupted tenant's membership was removed | Lands here; the tenant absent; draft discarded (RF-1.2; 006 RF-8.4, LC-17). |
| LC-30 | Session expires while the chooser is open | Specification 006's handling; re-authentication returns here with a fresh list (RF-7.4). |
| LC-31 | Page reloaded | List derived again; nothing from storage (RF-7.3). |
| LC-32 | Two tabs, "Enter" different tenants in each | Each request derived on its own; each tab enters its tenant under its own context; the guarded switch of 016 governs any shared state (RF-6.1, RF-6.2). |
| LC-33 | Identity restricted while the chooser is open | The next request meets the session handling of 006; no entry (RF-1.1, RF-7.4). |
| LC-34 | Removed member opens the chooser route by its address | Only their remaining memberships; the removed tenant absent; no trace of it (RF-3.2, RF-3.3). |
| LC-35 | Colleague looks for a member list or tenant directory on the page | None exists; the page shows only the person's own facts (RF-2.6, RNF-1). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The landing rule itself** — which sign-ins enter a tenant directly and which arrive here — specification 001 RF-8.
- **The guarded tenant and project switch** — the order of its steps, the dirty guard dialog N24, epoch and cache rules — F-IDN-18, specification 016; the chooser is one of its entry points.
- **The tenant overview opened after entry** (R10 / M04) and the first-run checklist (R09) — FR-PRJ-110 … 116, the projects area.
- **The suspended-tenant screen and the tenant recovery process** — F-IDN-16, specification 014; the chooser only opens it.
- **Workspace creation and its enrollment precondition** — F-IDN-14, specification 012.
- **Sign-out, "Sign out everywhere" and the sessions list** — specification 007.
- **TOTP enrollment and the gate at tenant entry** — specification 003; the chooser only states the requirement.
- **Changing a role, a grant or a membership state** — specifications 009, 010 and 011; the chooser only reflects them.
- **Leaving a tenant and identity erasure** — F-IDN-17, specification 015; the chooser offers neither.
- **Search, filter or paging of memberships** — decided against (decision Q7). The user menu and the header tenant switcher are shell families (`395:668`, `395:667`; S-07) that route here; their anatomy is specification 016's.
- **Tablet-specific designs** of the chooser — none; the D01 responsive policy applies.
- **The content of the Help panel** — "Help" on the page opens N43 HelpShortcutsPanel (`430:4129`; S-17); the panel is the shell's, not this feature's.

---

## 7. Criteria of completion

The tenant chooser is complete when all of the following hold:

1. Every user story US-1 … US-9 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-10 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-10 has at least one verification that fails before and passes after the feature exists.
3. Listing tests show that only the signed-in identity's active and suspended memberships are rendered, that a revoked membership and a tenant without a membership never appear, and that the four card variants are chosen and ordered as RF-3.4 and RF-3.5 state (RF-3, LC-4, LC-5, LC-19, LC-34).
4. Exposure tests show that an enterable card shows only the person's own role and grants and the names of their eligible projects, and that a suspended tenant, a suspended membership and a deleting tenant show their state and the person's role only (RF-4.6, RF-5.5, RNF-1, RNF-5, LC-35).
5. Grant tests show that project lines equal the live grant records, including the `canRun`-only and no-grant variants (RF-4.2, RF-4.3, RNF-6, LC-15 … LC-17).
6. Entry tests show that "Enter" is derived on the server from current facts, that a stale page is refreshed and never enters after a suspension, a deletion or a revocation, and that a granted entry runs the guarded switch and the enrollment gate on arrival (RF-6, LC-8 … LC-13, LC-18).
7. Navigation tests show every route into the chooser of RF-1.2, the always-render rule of RF-1.3 with one and zero memberships, the "Switch workspace" entry with and without a dirty editor, and the exits "View access status", "Create workspace" and "Sign out" (RF-1, RF-2, RF-5.4, LC-2, LC-3, LC-21 … LC-29).
8. Freshness tests show derivation on every open and on every "Enter", no push while idle, nothing in storage, and the session handling of specification 006 on expiry (RF-7, LC-11, LC-30, LC-31).
9. Boundary tests show that no action on the page accepts, approves, launches, replays or retargets a Run, changes a membership or grant, or grants authority (RF-6.5, RF-6.6, RNF-2, RNF-7).
10. Audit entries exist for every refused entry of RF-10.1 and contain no list or grant content; nothing of the page reaches storage, logs, problems or analytics (RF-10.1, RF-10.2).
11. The page and every card variant pass WCAG 2.2 AA with automated evidence and manual assistive-technology evidence, in EN, ES and FR, at desktop, tablet and mobile widths (RNF-11 … RNF-13).
12. The register items this specification relies on — A-36 and A-54 in the architecture register; S-55, S-56, S-48, S-07, S-02, S-03 and S-17 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. The constitution's F-IDN-15 entry already reflects every decision of §1.4; no amendment is required.
