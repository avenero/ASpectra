# Specification 016 — Guarded tenant and project switch

**Feature** The transition that moves a signed-in person from one working context (a tenant and, within it, a project) to another, protecting their unsaved work, never carrying a request, a cached fact or a live event across the boundary, and loading the new context only after its authority has been derived afresh · **Version** 0.3 · **Date** 29 October 2026 · **Status** Draft for review — open doubts closed 28 October 2026; v0.2 routes a deleting target tenant to N20 (decision Q5); v0.3 keeps a switch started from the chooser on the refreshed chooser, as specification 013 Q4 (decision Q5) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-18 as the subject; F-IDN-03, F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-13, F-IDN-15, F-IDN-16 and F-IDN-04 (enrollment gate) where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-05, F-X-07; principles P1, P3, P6, P13, P14) → `requirements.md` (FR-IDN-024, 064, 066, 067, 068, 093, 110, 111, 112, 116, 130, 131, 132; FR-X-001, 002, 020, 021, 022, 023, 030, 033, 034, 040, 042, 043, 044, 051, 055; FR-AUT-031, 032, 033; FR-RUN-050) → the design of record: the compact state N24 TenantSwitchDirtyDialog ("Save before switching?", "Save & switch", "Stay here", "Discard changes & switch") and N25 StaleRevisionConflictDialog of the D02 interaction atlas `159:32045`; N28 ProjectAccessDeniedPanel `159:32104`; N03 TenantChooserPage `161:49326`; N20 TenantSuspendedPage `161:50995`; the context header "WORKSPACE ⌄ / PROJECT ⌄" drawn on the app-shell screens; board 3 §03 `46:7847` ("Tenant switching is a state transition, not a dropdown filter": switch request → dirty guard → drain old scope → reload trusted new scope) and the FE board `46:7847`, which is current where the two differ (register A-03).

**Related specifications:** 003 TOTP enrollment (the enrollment gate applied on arrival in a tenant, RF-1.3 and RF-1.4) · 006 Sessions, refresh and expiry (session expiry and re-authentication during a switch, RF-8.3 … RF-8.5; the session shared by tabs) · 007 Logout and session management (the same dirty guard run before sign-out, RF-3.1 … RF-3.7) · 008 Invitation acceptance (entry into the new tenant after acceptance, RF-6.5) · 009 Roles, grants and the authority matrix (what a membership and a grant allow) · 011 Project grants editor (its edit panel is unsaved work, RF-3.7) · 013 Tenant chooser (the page "Switch workspace" and WORKSPACE ⌄ open; its "Enter", RF-1.4, RF-1.5, RF-6.2) · 014 Tenant suspension state (N20 shown when the target tenant is suspended) · the authoring area (the editor fork, its saves and its conflict resolution, FR-AUT-031 … 033).

This document states **what** the guarded switch does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is global and may belong to several tenants, and within a tenant may hold grants on several projects (FR-IDN-002, FR-IDN-090, FR-IDN-093). Inside the application the person always works in one **scope** — a tenant and, usually, a project — shown in the context header as "WORKSPACE ⌄" and "PROJECT ⌄". Everything on the screen belongs to that scope: the data shown, the live event streams that update it, the requests under way and any work not yet saved.

Changing scope is therefore not a filter over the same data. It is a **transition** with five steps in a fixed order (FR-IDN-130): (1) a dirty-editor guard offering Save / Discard / Stay; (2) abort the old scope's in-flight requests and close its live event streams; (3) advance the **scope epoch** and clear the old scope's cached data and unsaved drafts; (4) fetch the person's current membership and grants for the new scope; (5) only then load data or open a stream. A save chosen in the guard goes to the scope the work was written in, never to the newly selected one (FR-IDN-131). Anything that arrives late from the old scope — a response, an event — is discarded by its epoch, and clearing the screen is never mistaken for an authorisation decision: the server authorises every request, subscription and download at the moment it is made (FR-IDN-132).

The switch starts from the "Switch workspace" action and the header WORKSPACE selector, both of which open the tenant chooser (specification 013); from the header PROJECT selector; from "Enter" on the chooser and the entry after an invitation is accepted (specification 008); and from any navigation that names another tenant or project.

### 1.2 Objective

Let a person move between workspaces and projects freely, while:

- never losing unsaved work without an explicit choice, and never saving it into the wrong tenant or project;
- never letting a request, a cached fact, a draft or a live event of the old scope reach the new one, or a late one be shown at all;
- never retargeting, resubmitting, cancelling or duplicating a command already sent, an accepted Run included;
- showing nothing of the new scope until its authority has been derived afresh on the server, and showing the honest state — chooser, suspended tenant, denied project, or a retryable problem — when it cannot be entered;
- applying one rule to every entry point, every tab and every width and language.

### 1.3 Why it matters

A consultant with three customer workspaces switches between them many times a day. A switch that keeps the old tenant's cache paints one customer's project names over another's header; a switch that lets a late event through shows a run result under the wrong tenant; a guard that saves "where the person is now" writes customer A's draft into customer B; a switch that cancels or resends a pending launch turns navigation into a Run effect. Each is a cross-tenant leak or an unintended execution, the two failures the product is built to make impossible. The switch is the single place where these leaks are cut, so it must be ordered, total and honest.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 28 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, all written on 28 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14: **A-63** (scope epoch boundary: per tab, tenant + project, route scope request-only) is new; a note is appended to **A-03** (board 3 and the FE section duplicate tenant switching; the FE section is followed). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-58** (open state of the header WORKSPACE / PROJECT selectors) and **S-59** (states of N24, D02 `421:3477`) are new; notes are appended to **S-07** (app-shell identity area and header selectors) and **S-02** (responsive counterparts).

| Q | Question | Decision |
|---|---|---|
| Q1 | Entry points, and what the header selectors do | "WORKSPACE ⌄" opens the tenant chooser of specification 013 — the same destination as "Switch workspace", so one tenant list exists. "PROJECT ⌄" opens a compact list of the current tenant's projects in which the person holds at least one live grant, with exactly their own grants and the lines of specification 013 Q2 ("canRun only · no project role yet"; "No project grants yet — workspace administration does not grant project content" when none). The other entry points are "Enter" on the chooser (013 RF-6.2), the entry after invitation acceptance (008 RF-6.5) and route navigation (Q7). Every entry point runs the same transition. |
| Q2 | A guard save that fails | The switch is cancelled and the person stays in the original scope with the work intact and still unsaved. A stale revision or workflow conflict opens the existing conflict resolution (N25: base / local / server); a validation problem is shown at its fields; a save with no answer is resolved by looking up its outcome, never by resubmitting it (F-X-02). The person then chooses again. |
| Q3 | What counts as unsaved work | Any deliberate unsaved fork: the authoring editor fork (FR-AUT-031), the project grants panel (011 RF-3.7), settings and policy forms, dialogs holding typed input, and a typed but unsent chat or proposal message. Read-only view state — filters, sort, scroll, expanded rows — never counts. One N24 covers all unsaved items; "Save & switch" saves each in its own original tenant, project and revision. |
| Q4 | Commands already sent but not yet acknowledged | Safe reads are aborted at once. A pending consequential command (a save, an approval, a launch) is never resubmitted, retargeted or cancelled by the switch; the switch shows "A pending request remains bound to its original workspace and project." and proceeds. Its outcome is found later by lookup in the original scope; an accepted Run keeps its pins (FR-RUN-050). This specification sets no waiting period (D-a). |
| Q5 | The new scope cannot be entered, or the check fails | The person is never silently returned to the old scope: for the selector, route, invitation and creation origins it is already drained when the check runs; for the chooser origin the check is the entry request of specification 013 RF-6.1, decided before anything is drained, so a refusal refreshes the chooser (RF-1.2, 013 RF-6.3). A tenant that can no longer be entered because of the membership (revoked or suspended) leads to the chooser, re-derived so the card shows its new state (013 Q4); a suspended tenant leads to N20 (specification 014), and a deleting tenant to N20 in its deleting wording, as specification 014 D-d treats deleting like suspended (decided 28 October 2026 while planning, replacing the earlier "deleting → chooser"). Exception (decided 29 October 2026): when the switch was started from the chooser's "Enter", the refreshed chooser is the answer for a suspended or deleting tenant too, as specification 013 Q4 and RF-6.3 decide — the card shows the tenant's state and offers the way to N20; no separate screen opens. A project without a live grant, or deleted, shows N28 inside the target tenant. When the membership and grant fetch fails, a safe problem with "Retry" is shown and nothing of either scope is displayed until it succeeds. |
| Q6 | Several browser tabs | Each tab owns its scope epoch, its unsaved work and its streams. A switch in one tab never changes, clears or guards another tab, which keeps working in its own scope; the server re-derives authority at every request, so a tab whose access was removed meets the refusal on its next request. The session is shared as specification 006 defines. (A-63) |
| Q7 | Navigation that names another scope | Any in-app navigation whose route names a different tenant or project — Back / Forward, a pasted or bookmarked deep link, a notification link — runs the same guarded transition; when unsaved work exists, N24 is shown and "Stay here" restores the previous address. A full page load or reload starts a fresh scope with nothing to guard; the browser's own leave-page prompt protects unsaved work in that case. The tenant and project in a route are a requested scope only (F-X-01). (A-63) |
| Q8 | How much a project switch drains | One epoch covers the tenant and the project together. A project switch within one tenant performs every step exactly like a tenant switch: every request aborted, every stream closed, all scoped cached data and drafts cleared — tenant-level facts included — and membership and grants fetched afresh. Choosing the current tenant or project again does nothing. (A-63) |

Five rules were derived from these decisions and from earlier specifications rather than asked; the product owner may overturn them:

- **D-a** Q4 was chosen as "wait briefly, then switch with notice" with no waiting period defined; this specification therefore sets none — the notice is shown and the switch proceeds without delay. A bounded wait, if ever wanted, would be a configured value (P13) added by amendment.
- **D-b** Escape, closing N24 and leaving it by any means other than its two switch actions equal "Stay here"; focus returns to the control that started the switch.
- **D-c** The language switch changes display only, also during a switch; it is never a scope change.
- **D-d** If the session ends while N24 is open or a guard save is pending, specification 006 governs: the dialog of 006 is shown, the switch is abandoned, and a successful re-authentication returns to the **original** scope with its work as 006 RF-8.3 defines; the switch is not resumed on its own.
- **D-e** The enrollment gate of specification 003 (RF-1.3, RF-1.4) applies on arrival in a tenant, after step 4; it is not part of the transition and the transition never bypasses it.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | consultant working for several customers | to move to another workspace from the header or "Switch workspace" without signing out | I can work for each customer in turn |
| US-2 | Author with an unsaved draft | to be asked whether to save, discard or stay before the context changes | I never lose edits by accident |
| US-3 | Author who chooses "Save & switch" | my draft saved to the tenant and project I wrote it in | it never lands in the workspace I am moving to |
| US-4 | Author whose guard save meets a newer revision | to stay where I am with my draft and resolve the conflict | a failed save never costs me my work or moves me |
| US-5 | person who just launched a Run and switches at once | the launch to stay bound to its original project and never be resent | navigation never becomes a second Run or a wrong target |
| US-6 | person moving between projects of one tenant | to pick a project I hold a grant on from the header | I change project without passing through the chooser |
| US-7 | person whose access to the target was removed a minute ago | to be told honestly where I can and cannot go | I am not shown content or left on a blank screen |
| US-8 | person with two tabs on two customers | each tab to keep its own workspace | switching in one never disturbs the other |
| US-9 | keyboard or screen-reader user, on any width and in any language | the guard and the selectors to be fully operable and announced | I can switch safely without a pointer or a large screen |
| US-10 | attacker, removed member or curious colleague (negative story) | — | I cannot read a previous scope's data after a switch, receive its late events, have a draft saved into another tenant, enter a scope by editing an address or reusing a page loaded before my access changed, or make a switch resend, cancel or retarget any command |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *scope* means the pair tenant + project in which a tab is working (the project may be absent, as on a tenant overview); *the switch* means the transition of this specification; *old scope* and *new scope* mean the scope before and after it; *scope epoch* means the marker that identifies the current scope of a tab and changes at every switch; *unsaved work* means an item of decision Q3; *pending command* means a consequential command sent but not yet definitively answered; *live stream* means any subscription that pushes events to the tab; *the guard* means N24.

### RF-1 · Entry points

Every way of changing scope is the same transition. *Why:* FR-IDN-130 — one rule, no entry point that skips a step. *(Decision: Q1, Q7.)*

- RF-1.1 The system shall start the switch from: "Switch workspace" and the header "WORKSPACE ⌄", which open the tenant chooser (specification 013 RF-1.4, RF-1.5) after the guard; "Enter" on the chooser (013 RF-6.2); the entry after an invitation is accepted (008 RF-6.5); a choice in the header "PROJECT ⌄" list (RF-1.3); any navigation of RF-9; and leaving the current tenant (specification 015 RF-4.9), which drains the scope and opens the chooser without an entry. *(FR-IDN-130; decision Q1; S-58)*
- RF-1.2 When "WORKSPACE ⌄" is activated, the system shall behave exactly as "Switch workspace": run the guard first, open the chooser only after it is satisfied, and end the old scope only when a tenant is entered, not when the chooser opens. *(FR-IDN-130; decision Q1; 013 RF-1.5; S-58)*
- RF-1.3 When "PROJECT ⌄" is activated, the system shall show the projects of the current tenant in which the person holds at least one live grant, each with exactly their own grants, the current project marked; a `canRun`-only project reads "canRun only · no project role yet"; with no grant the list reads "No project grants yet — workspace administration does not grant project content". The list shall show no other project, count, source, run or member, and shall be derived on the server when opened. *(FR-IDN-111, FR-IDN-130; decision Q1; 013 Q2; S-58)*
- RF-1.4 When the current tenant or project is chosen again, the system shall do nothing: no guard, no drain, no fetch. *(decision Q8)*
- RF-1.5 Where the app-shell screen is shown on a narrow width, the system shall reach both selectors from the mobile menu with the same behaviour. *(FR-X-051; S-02, S-58)*

### RF-2 · Order of the transition

*Why:* FR-IDN-130 fixes the order; each step depends on the one before. *(Decision: Q8.)*

- RF-2.1 The system shall perform the switch in this order and no other: (1) the guard (RF-3, RF-4); (2) abort in-flight requests and close live streams of the old scope (RF-5); (3) advance the scope epoch and clear the old scope's cached data and unsaved drafts (RF-6); (4) fetch the person's current membership and grants for the new scope (RF-7); (5) only then load data or open a stream for the new scope. *(FR-IDN-130)*
- RF-2.2 The system shall perform every step for a project switch within one tenant exactly as for a tenant switch, tenant-level facts included. *(FR-IDN-130; decision Q8; A-63)*
- RF-2.3 While steps 2 to 4 are running, the system shall show a switching state that displays nothing of the old or the new scope beyond the name of the target the person chose. *(FR-IDN-132; S-59)*
- RF-2.4 If a step cannot complete, then the system shall not perform any later step; steps already performed are never undone by restoring the old scope. *(FR-IDN-130; decision Q5)*

### RF-3 · The dirty guard

*Why:* unsaved work is lost by the drain of step 3; it may only be lost by the person's choice. *(Decision: Q3; D-b.)*

- RF-3.1 When the switch starts while the tab holds unsaved work, the system shall show the guard before any other step, with "Save before switching?", the statement "You have unsaved changes in <project>. A pending request remains bound to its original workspace and project." and the actions "Save & switch", "Stay here" and "Discard changes & switch". *(FR-IDN-130; N24)*
- RF-3.2 Where several items of unsaved work exist, the system shall list each by name in one guard and apply the chosen action to all of them. *(decision Q3; S-59)*
- RF-3.3 When the switch starts while the tab holds no unsaved work, the system shall not show the guard. *(FR-IDN-130)*
- RF-3.4 When "Stay here" is chosen, or the guard is closed or escaped, the system shall cancel the switch and leave the scope, the screen, the work and any address unchanged, and return focus to the control that started the switch. *(FR-IDN-130; D-b)*
- RF-3.5 When "Discard changes & switch" is chosen, the system shall discard every unsaved item and continue with the next step of the origin's sequence: for the chooser origin the chooser opens with the old scope still current (RF-1.2); for every other origin the pending-command step follows. *(FR-IDN-130)*
- RF-3.6 The system shall count as unsaved work exactly the items of decision Q3, and shall never count read-only view state. *(FR-AUT-031; decision Q3)*
- RF-3.7 The guard shall state what is protected and offer the choice; it shall never itself grant, deny or check authority. *(FR-IDN-132; P1)*

### RF-4 · The guard save

*Why:* FR-IDN-131 — the save belongs to the scope where the work was written. *(Decision: Q2, Q3.)*

- RF-4.1 When "Save & switch" is chosen, the system shall submit each item's save under the tenant, project and base revision in which that item was written, with its own operation identity, and never under the selected new scope. *(FR-IDN-131, FR-X-021; decision Q3)*
- RF-4.2 While guard saves are pending, the system shall keep the old scope, disable the guard's actions and state that saving is in progress. *(FR-IDN-131; S-59)*
- RF-4.3 When every guard save is acknowledged, the system shall continue with the next step of the origin's sequence, as RF-3.5 states. *(FR-IDN-130)*
- RF-4.4 If any guard save is refused for a stale revision or a workflow conflict, then the system shall cancel the switch, keep the person in the old scope with every unsaved item intact and still unsaved, and open the conflict resolution of that item (N25: base / local / server). *(FR-X-033, FR-AUT-032; decision Q2)*
- RF-4.5 If any guard save is refused for a validation problem, then the system shall cancel the switch, keep the work intact and show the problem at its fields. *(FR-X-033; decision Q2)*
- RF-4.6 If any guard save receives no definitive answer, then the system shall cancel the switch, keep the work intact, and resolve the outcome of that save by lookup; it shall never resubmit it. *(FR-X-022, FR-X-023; decision Q2)*
- RF-4.7 Where some guard saves succeed and one fails, the system shall keep the acknowledged ones as saved, keep the others unsaved, and cancel the switch as RF-4.4 … RF-4.6 state. *(decision Q2)*
- RF-4.8 When typing continues while a guard save is pending, the system shall treat only the captured edit as acknowledged and newer edits as still unsaved; the switch shall then not proceed, and the guard shall be shown again for the newer edits. *(FR-AUT-033)*

### RF-5 · Draining the old scope

*Why:* nothing of the old scope may still be on its way when the new one loads. *(Decision: Q4.)*

- RF-5.1 When step 2 begins, the system shall abort every in-flight safe read of the old scope and close every live stream of the old scope. *(FR-IDN-130)*
- RF-5.2 Where a pending command exists at step 2, the system shall stop waiting for it, never resubmit, retarget or cancel it, and show "A pending request remains bound to its original workspace and project."; the switch then proceeds without delay. *(FR-X-021, FR-X-022; decision Q4; D-a)*
- RF-5.3 The system shall make the outcome of a pending command of RF-5.2 findable later by lookup in its original scope, as the feature that owns the command defines. *(FR-X-022)*
- RF-5.4 The switch shall never cancel, duplicate, replay or retarget an accepted job; an accepted Run keeps its original tenant, project and environment pins. *(FR-RUN-050, FR-IDN-112; P1)*

### RF-6 · Epoch and clearing

*Why:* FR-IDN-130 step 3 and FR-IDN-132 — every fact on screen belongs to exactly one scope. *(Decision: Q8.)*

- RF-6.1 When step 3 begins, the system shall advance the tab's scope epoch so that every response, event and result of the old scope is recognisable as belonging to a past epoch. *(FR-IDN-130, FR-IDN-132; A-63)*
- RF-6.2 When step 3 begins, the system shall clear all cached data and all unsaved drafts held for the old scope, tenant-level facts included; no item of the old scope shall be shown, reused or offered as a starting value in the new one. *(FR-IDN-130, FR-AUT-031; decision Q8)*
- RF-6.3 The system shall keep across a switch only identity-scoped facts (the person's name, address, interface language and session) and the allowlisted harmless preferences of FR-X-034. *(FR-X-034)*

### RF-7 · Fresh authority for the new scope

*Why:* FR-IDN-130 step 4 — nothing loads on the strength of what the tab believed before. *(Decision: Q5.)*

- RF-7.1 When step 4 begins, the system shall fetch from the server the person's current membership in the target tenant and current grants and permission version for the target project, and shall use nothing cached, nothing from the route and nothing from the page that started the switch. *(FR-IDN-130, FR-X-001; A-63)*
- RF-7.2 When the fetch shows an enterable membership in an active tenant and, where a project is targeted, a live grant on an existing project, the system shall perform step 5 and open the target. *(FR-IDN-130)*
- RF-7.3 If the fetch shows that the target tenant cannot be entered because the membership is revoked or suspended, then the system shall open the tenant chooser re-derived, where the card shows its new state or is absent (specification 013 Q4), and shall not return to the old scope. *(FR-IDN-110, FR-IDN-116; decision Q5)*
- RF-7.4 If the fetch shows that the target tenant is suspended or deleting, then the system shall open the suspended-tenant screen N20 of specification 014 — in its deleting wording for a deleting tenant (specification 014 D-d, RF-3.10) — and show no project content — except when the switch was started from the chooser's "Enter", where specification 013 RF-6.3 applies: the chooser is derived again and the card shows the tenant's new state, with no separate error and no entry. *(FR-IDN-093, FR-IDN-116; decision Q5)*
- RF-7.5 If the tenant can be entered but the targeted project has no live grant for the person or no longer exists, then the system shall enter the tenant and show N28 in place of the project, with no project content. *(FR-IDN-064, FR-IDN-132; decision Q5)*
- RF-7.6 If the fetch fails or receives no definitive answer, then the system shall show a safe problem with a correlation identifier and "Retry", display nothing of either scope, and load nothing until a retried fetch succeeds; the person may also open the chooser or sign out. *(FR-X-033; decision Q5; S-59)*
- RF-7.7 When a tenant is entered, the system shall apply the enrollment gate of specification 003 on arrival. *(D-e)*

### RF-8 · Late arrivals and authorisation

*Why:* FR-IDN-132 — what the client clears is not what the server allows.

- RF-8.1 If a response, event or result arrives carrying a past epoch, then the system shall discard it without displaying, caching or acting on it. *(FR-IDN-132; A-63)*
- RF-8.2 The system shall authorise every request, subscription, preview and download of the new scope on the server at the moment it is made, by the person's current membership and grants, and never by an identifier carried in a payload, a route or a cached item. *(FR-IDN-132, FR-X-001)*
- RF-8.3 The system shall never treat the clearing of step 3 as an authorisation decision, and never treat the presence of a cached item as evidence of access. *(FR-IDN-132)*

### RF-9 · Navigation that names another scope

*Why:* the address bar and history are entry points too. *(Decision: Q7.)*

- RF-9.1 When an in-app navigation — Back, Forward, a deep link opened in the tab, a notification link — names a tenant or project different from the tab's scope, the system shall run the switch to that scope. *(FR-IDN-130; decision Q7; A-63)*
- RF-9.2 When "Stay here" is chosen on a guard raised by RF-9.1, the system shall restore the previous address and leave the scope unchanged. *(decision Q7)*
- RF-9.3 When a page is loaded or reloaded, the system shall start a fresh scope from the route with nothing to guard and perform steps 4 and 5; where unsaved work exists before a reload or closing the page, the browser's own leave-page confirmation shall be requested. *(FR-IDN-130; decision Q7)*
- RF-9.4 The system shall treat the tenant and project in any route as a requested scope only, derived and authorised by RF-7 and RF-8. *(FR-X-001; decision Q7)*

### RF-10 · Several tabs

*Why:* parallel work across customers must stay parallel. *(Decision: Q6.)*

- RF-10.1 The system shall give each tab its own scope epoch, unsaved work and live streams; a switch in one tab shall never change, clear or guard another tab. *(FR-IDN-130; decision Q6; A-63)*
- RF-10.2 If another tab's access changed because of a membership or grant change, then that tab shall meet the server's refusal on its next request and show the state of RF-7.3 … RF-7.5 for its own scope. *(FR-IDN-132; decision Q6)*
- RF-10.3 Session refresh, expiry and sign-out shared between tabs shall follow specifications 006 and 007; the switch adds no cross-tab behaviour. *(decision Q6)*

### RF-11 · Interaction with sessions and sign-out

*Why:* the switch must not open a side door around session handling. *(D-d.)*

- RF-11.1 If the session ends while the guard is open or a guard save is pending, then the system shall abandon the switch and apply specification 006; a successful re-authentication returns to the original scope and does not resume the switch. *(FR-IDN-066, FR-IDN-067; D-d)*
- RF-11.2 The dirty guard of sign-out (specification 007 RF-3.1 … RF-3.7) shall be the same guard and follow RF-3 and RF-4, with sign-out in place of the switch. *(FR-IDN-131)*

### RF-12 · Audit and secrecy

*Why:* FR-X-030, FR-X-034 — the switch moves between customers and must leave no trace of one in the other.

- RF-12.1 The system shall append to the audit log every entry refused at step 4 (RF-7.3 … RF-7.5) with the identity, the requested scope and the cause, and no content; an entry refused on the chooser's "Enter" is the record of specification 013 RF-10.1 and is not recorded twice. *(FR-X-044)*
- RF-12.2 The system shall keep no item of any scope in browser storage across a switch beyond the allowlist of FR-X-034, and shall write no draft, prompt or secret there. *(FR-X-034, FR-AUT-031)*
- RF-12.3 Problems shown by the switch shall carry safe codes and a correlation identifier only, never another scope's names or content. *(FR-X-033)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Fixed order.** No data load or stream opening of the new scope happens before its step 4 has succeeded; verified on every entry point. | FR-IDN-130 |
| RNF-2 | **Isolation.** After a switch, no cached fact, draft, response or event of the old scope is displayed or acted upon, under any timing of late arrivals. | FR-IDN-132 |
| RNF-3 | **No retargeting.** No save or command is ever submitted under a scope other than the one it was written in; no pending command is resubmitted or cancelled by a switch. | FR-IDN-131, FR-X-021, FR-X-022 |
| RNF-4 | **No Run effect.** No switch accepts, approves, launches, cancels, replays or retargets anything. | FR-RUN-050; P1 |
| RNF-5 | **Server authority.** Every check that decides access is made on the server at request time; client clearing and client guards are never authority. | FR-IDN-132, FR-X-001 |
| RNF-6 | **No configuration.** The switch introduces no numeric value of its own (no wait, no timeout); the bounds of retries and sessions belong to F-X-02 and specification 006. | P13; D-a |
| RNF-7 | **Secrecy.** Nothing of any scope reaches browser storage, logs, problems or analytics beyond safe codes and a correlation identifier. | FR-X-030, FR-X-034 |
| RNF-8 | **Accessibility.** The guard, the switching state, the PROJECT list and the failure panel meet WCAG 2.2 AA with automated and manual assistive-technology evidence: the guard is a labelled modal that traps focus and returns it on close, the switching state is announced, the list is keyboard-operable. | FR-X-040, FR-X-042 |
| RNF-9 | **Language.** Every statement and action exists in EN / ES / FR; the language switch changes display only. | FR-X-043; D-c |
| RNF-10 | **Reflow.** Mobile and tablet reflows keep every statement and action of the guard, the list and the panels. | FR-X-051, FR-X-055; S-02 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Header "WORKSPACE ⌄" with nothing unsaved | Chooser opens; old scope still current until a tenant is entered (RF-1.2). |
| LC-2 | "WORKSPACE ⌄" with an unsaved draft, then "Stay here" | Nothing changes; focus back on the selector (RF-3.4). |
| LC-3 | "WORKSPACE ⌄", "Save & switch", chooser opened, then the chooser is left without choosing | The draft is saved in the original scope; the old scope remains current (RF-1.2, RF-4.3). |
| LC-4 | "PROJECT ⌄" with grants on three projects | Three lines with own grants, current marked; choosing another runs the full switch (RF-1.3, RF-2.2). |
| LC-5 | "PROJECT ⌄" by an Admin with no project grant | "No project grants yet — workspace administration does not grant project content" (RF-1.3). |
| LC-6 | Current project chosen again | Nothing happens (RF-1.4). |
| LC-7 | Draft in the editor and unsaved grants panel; "Save & switch" | Both listed in one guard; each saved in its own scope (RF-3.2, RF-4.1). |
| LC-8 | Guard save meets a newer revision | Switch cancelled; draft intact; N25 opens (RF-4.4). |
| LC-9 | Guard save has a validation error | Switch cancelled; error at the field (RF-4.5). |
| LC-10 | Guard save gets no answer | Switch cancelled; outcome looked up; no resubmission (RF-4.6). |
| LC-11 | Two items; one saves, one conflicts | First stays saved; second unsaved; switch cancelled (RF-4.7). |
| LC-12 | Typing continues during the guard save | Captured edit acknowledged; newer edits unsaved; guard shown again (RF-4.8). |
| LC-13 | "Discard changes & switch" | Drafts discarded; switch continues (RF-3.5). |
| LC-14 | Guard escaped with the keyboard | Same as "Stay here" (RF-3.4; D-b). |
| LC-15 | Filter and sort changed but nothing edited | No guard (RF-3.3, RF-3.6). |
| LC-16 | Run launched, acknowledgement not yet received, switch started | Notice shown; switch proceeds; launch never resent; outcome found by lookup in the original project; the Run keeps its pins (RF-5.2 … RF-5.4). |
| LC-17 | Old project's run-status event arrives after the switch | Discarded by epoch; nothing shown (RF-8.1). |
| LC-18 | Old tenant's list response arrives after the new tenant loaded | Discarded by epoch (RF-8.1). |
| LC-19 | Project switch within one tenant | Tenant-level facts cleared too and fetched afresh (RF-2.2, RF-6.2). |
| LC-20 | Target membership revoked between choosing and step 4 | Chooser re-derived; tenant absent; no return to the old scope (RF-7.3). |
| LC-21 | Target tenant suspended at step 4 | N20; no project content (RF-7.4). Started from the chooser's "Enter": the refreshed chooser, the card reading "Workspace suspended" (RF-7.4, specification 013 RF-6.3). |
| LC-22 | Target tenant deleting at step 4 | N20 in its deleting wording; no project content (RF-7.4). Started from the chooser's "Enter": the refreshed chooser, the card showing the deleting state (RF-7.4, specification 013 RF-6.3). |
| LC-23 | Target project grant removed at step 4 | Tenant entered; N28 in place of the project (RF-7.5). |
| LC-24 | Target project deleted | Tenant entered; N28 (RF-7.5). |
| LC-25 | Fetch at step 4 fails (network) | Safe problem with "Retry"; nothing of either scope shown; chooser and sign-out reachable (RF-7.6). |
| LC-26 | Owner without a factor enters a tenant | Switch completes; enrollment gate on arrival (RF-7.7; D-e). |
| LC-27 | Back button to the previous tenant with a dirty draft | Guard; "Stay here" restores the address (RF-9.1, RF-9.2). |
| LC-28 | Deep link to a tenant the person never belonged to | Requested scope refused at step 4; chooser re-derived; tenant never shown (RF-9.4, RF-7.3). |
| LC-29 | Address edited to another tenant's project identifier | Same as LC-28 or LC-23 per the facts; payload identifiers never authorise (RF-8.2, RF-9.4). |
| LC-30 | Reload with unsaved work | Browser leave-page prompt; after reload, a fresh scope with no draft (RF-9.3). |
| LC-31 | Two tabs on tenant A; one switches to B | The other tab stays on A, its drafts and streams untouched (RF-10.1). |
| LC-32 | Two tabs; the person's membership in A revoked | Tab on A meets the refusal on its next request; chooser re-derived (RF-10.2). |
| LC-33 | Session expires while the guard is open | Specification 006 dialog; switch abandoned; re-authentication returns to the original scope (RF-11.1; D-d). |
| LC-34 | Sign-out with a dirty draft | The same guard, with sign-out in place of the switch (RF-11.2). |
| LC-35 | Entry after accepting an invitation | Full switch; no guard if nothing unsaved; the granted project offered (RF-1.1; 008 RF-6.5). |
| LC-36 | Language switched while the switching state is shown | Display only; the switch continues (RNF-9; D-c). |
| LC-37 | Narrow width | Selectors in the mobile menu; guard and panels reflowed with every action (RF-1.5, RNF-10). |
| LC-38 | Removed member keeps an old tab open and inspects it after switching | Nothing of the old scope remains on screen or in storage; late arrivals discarded (RF-6.2, RF-8.1, RF-12.2). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The tenant chooser page**, its cards and its "Enter" derivation — specification 013; the switch begins and ends around it.
- **The suspended-tenant screen** — specification 014; the switch only opens it.
- **The content of conflict resolution** (N25) and the authoring save rules — the authoring area (FR-AUT-031 … 033); the switch only opens it and respects its rules.
- **The denied-project panel N28** and its wording — owned by its feature; the switch only shows it.
- **Session expiry, refresh and re-authentication** — specification 006.
- **Sign-out and "Sign out everywhere"** — specification 007; it reuses the guard.
- **The enrollment gate** — specification 003; it applies on arrival.
- **Changing a membership, role or grant** — specifications 009, 010 and 011; the switch only observes their current state.
- **The tenant overview opened after entry** (R10 / M04) — the projects area.
- **Recent-scope lists**, favourite projects, search in the PROJECT list, and cross-tab synchronisation of scope — decided against (decision Q6). The user menu is the shell's (`395:668`; S-07), specified by 007 and 013.
- **Tablet-specific designs** of the guard and the selectors — none; M24 MobileGuardedSwitchSheet (`432:4464`) is the mobile counterpart and the D01 responsive policy applies elsewhere.

---

## 7. Criteria of completion

The guarded switch is complete when all of the following hold:

1. Every user story US-1 … US-9 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-10 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-12 has at least one verification that fails before and passes after the feature exists.
3. Order tests show, for every entry point of RF-1.1 and RF-9.1, that steps run in the order of RF-2.1, that nothing loads before step 4 succeeds, and that a project switch performs every step (RF-1, RF-2, RF-9, LC-1, LC-4, LC-19, LC-27).
4. Guard tests show the guard appears exactly when unsaved work of decision Q3 exists, that one guard covers several items, and that "Stay here", Escape and closing change nothing (RF-3, LC-2, LC-7, LC-13 … LC-15).
5. Guard-save tests show every save is submitted under its original scope, and that a conflict, a validation problem, an unanswered save, a partial failure and continued typing each cancel the switch with the work intact (RF-4, LC-3, LC-7 … LC-12).
6. Drain tests show that pending commands are never resubmitted, retargeted or cancelled, that their outcome is findable by lookup, and that accepted Runs keep their pins (RF-5, RNF-3, RNF-4, LC-16).
7. Isolation tests inject late responses and events of the old scope under every timing and show none is displayed, cached or acted upon, and that no old-scope item survives clearing (RF-6, RF-8.1, RNF-2, LC-17, LC-18, LC-38).
8. Authority tests show the new scope is derived on the server at step 4 and at every later request, that route and payload identifiers never authorise, and that each refusal outcome of RF-7.3 … RF-7.6 is shown without returning to the old scope (RF-7, RF-8.2, RF-9.4, LC-20 … LC-25, LC-28, LC-29).
9. Tab tests show a switch in one tab never affects another, and that a tab whose access changed meets the refusal on its next request (RF-10, LC-31, LC-32).
10. Session tests show that expiry during the guard follows specification 006 and returns to the original scope, and that sign-out uses the same guard (RF-11, LC-33, LC-34).
11. Audit entries exist for every refused entry of RF-12.1 and contain no content; nothing of any scope reaches browser storage, logs, problems or analytics beyond the allowlist (RF-12, RNF-7).
12. The guard, the switching state, the PROJECT list and the failure panel pass WCAG 2.2 AA with automated and manual assistive-technology evidence, in EN, ES and FR, at desktop, tablet and mobile widths (RNF-8 … RNF-10).
13. The register items this specification relies on — A-63 and A-03 in the architecture register; S-58, S-59, S-07 and S-02 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. The constitution's F-IDN-18 entry already reflects every decision of §1.4; no amendment is required.
