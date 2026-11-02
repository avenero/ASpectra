# Specification 014 — Tenant suspension state

**Feature** The suspended state of a tenant: what suspension freezes, what every member sees, what remains possible, and reinstatement · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-16 as the subject; F-IDN-08, F-IDN-09, F-IDN-10, F-IDN-11, F-IDN-12, F-IDN-13, F-IDN-15, F-IDN-17, F-IDN-18, F-SET-03, F-RUN-03, F-RUN-07, F-RUN-08, F-RUN-13 and F-PRJ-07 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P8, P12, P13, P14) → `requirements.md` (FR-IDN-043, 048, 061, 064, 066, 067, 068, 093, 094, 110, 111, 116, 120, 121, 122, 125; FR-PRJ-043; FR-RUN-025, 050, 055, 067, 096; FR-X-001, 002, 003, 020, 022, 030, 033, 034, 040, 042, 043, 044, 045, 046, 052, 053, 055) → the design of record, screen N20 TenantSuspendedPage `161:50995`, the "Workspace suspended" card of N03 TenantChooserPage `161:49326`, and the Failure board `3:2192` ("Tenant suspended / membership revoked": freeze new effects and revoke pending authority before retrieval, acceptance, approval, launch, provider submission or delivery; bounded active cancel preserves known facts; acknowledgement is not rollback; fencing rejects late submissions, results and delivery callbacks).

**Related specifications:** 001 Login (landing by membership: a single membership in a suspended tenant lands on the chooser showing the state) · 004 Password reset and 005 Controlled account recovery (neither lifts a suspension; 005 Q7 keeps N20 out of identity recovery) · 006 Sessions, refresh and expiry (sessions are not ended by suspension; the denied-scope handling a member's open screens meet) · 007 Logout and session management ("Sign out" and "Personal account security" on N20) · 008 Invitation acceptance (an invitation to a suspended tenant waits; to a deleting tenant it is unusable) · 009 Roles, grants and the authority matrix (tenant status in every derivation, RF-5; the work-in-flight rule of its Q5 / A-56 applied here) · 010 Tenant members and ownership transfer and 011 Project grants editor (unreachable while suspended) · 012 Workspace creation (a founder may create another workspace while one is suspended) · 013 Tenant chooser (the suspended-tenant card RF-5.1 and "View access status" RF-5.4, which opens this screen) · 015 Leave tenant and identity erasure request (leaving a suspended tenant; an erasure held by a suspended owned tenant) · 016 Guarded tenant and project switch (entering a suspended tenant ends on N20) · the execution and delivery specifications (NOT EXECUTED tickets, bounded cancellation, held deliveries).

This document states **what** the tenant suspension state does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

A tenant is one customer organisation with its projects, sources, revisions, approvals, runs and evidence. Its status is one of **active**, **suspended** or **deleting** (FR-IDN-093), read by the server before anything else when it decides what a member may do (specification 009). **Suspension** is the platform's way of pausing a customer workspace without destroying anything: for billing, abuse, legal or operational reasons the platform freezes every new effect in the tenant, stops work in flight at a safe point, and keeps the workspace's data, memberships and grants exactly as they are until the platform reinstates it or begins its deletion.

For the people who belong to the tenant, suspension is a single, honest screen, N20: the workspace is paused, their identity is intact, their membership metadata is shown and nothing else, accepted runs are not failed tests and no verdict is rewritten, a verified workspace recovery process is the only route, and personal account security and sign-out remain theirs. Their other tenants are untouched; their sessions continue; their credentials, authenticator and recovery codes are not the tenant's and are not affected.

Suspension is not deletion and not a membership change. A **suspended membership** in an active tenant is a different thing (specification 009, S-48); a **deleting** tenant is a further, irreversible act with its own feature. This specification covers the suspended state and the transitions into and out of it.

### 1.2 Objective

Let the platform pause a customer workspace safely and transparently, while:

- freezing every new effect — retrieval, acceptance, approval, launch, provider submission, delivery, import, AI — from the next request of every member and every job, and stopping work in flight at bounded checkpoints without fabricating any outcome;
- showing every member one state that discloses the suspension and their own membership metadata and nothing else: no reason, no project name, no source, no run detail, no other member;
- keeping the person's identity, sessions, other memberships and other tenants fully usable, and offering account security and sign-out from the screen;
- changing nothing in the tenant's data while it is suspended, so that reinstatement restores exactly what was there, with nothing resumed or replayed that a human did not decide again;
- leaving suspension and reinstatement to the platform's internal, audited mechanism alone, with no tenant-side way to cause, request or lift either.

### 1.3 Why it matters

A suspension that leaks is a disclosure: a reason on screen tells every Member about a billing dispute, a project name tells a revoked contractor what still exists. A suspension that destroys is a breach of trust: a queued run executed days later when the tenant returns, a delivery resent twice, an attempt killed and shown as a failed test, a verdict rewritten. A suspension that traps is a lockout: a founder who cannot reach their other tenants, cannot secure their identity, cannot even sign out. The state must be a freeze — exact, silent about causes, reversible by construction — and the screen must be the calmest page in the product.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14: **A-62** (the suspension and reinstatement mechanism and the tenant recovery-contact record) is new; notes appended to **A-30** (notification set) and **A-56** (work in flight). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-57** (N20 states) is new; notes appended to **S-14** (email templates), **S-48** (chooser cards) and **S-02** (responsive coverage).

| Q | Question | Decision |
|---|---|---|
| Q1 | Who suspends and reinstates a tenant | Only the platform's internal, audited, out-of-band mechanism — the same shape as the compromise response of specification 004 and the case review of specification 005 — with an actor, a time and a recorded reason category (billing, abuse, legal, operator request) that is never shown to any member. No tenant Owner, Admin or Member can suspend, reinstate, or request either from inside the product. Deletion is a separate audited act of the platform, never an automatic consequence. The mechanism is register item A-62. |
| Q2 | What "Review verified workspace recovery" opens | A read-only panel on N20 showing the tenant recovery contact the platform configured for this suspension — a channel and instructions, never a link that grants access — and the statement that only that contact is to be used; when none is configured, the panel states that no contact is available and that the tenant's Owners will be notified when the state changes. Nothing is submitted from the product; the panel is the same for every member and exposes no reason, no project content and no other tenant. The panel is drawn in D02 `421:3131` (S-57). |
| Q3 | Work in flight at suspension, and reinstatement | At suspension: queued, not-yet-admitted run tickets end as NOT EXECUTED; running attempts receive a bounded cancellation request and their recorded facts stay; delivery jobs stop before their next provider write and their unconfirmed items are held, never retried blindly; import and AI jobs stop at their next checkpoint; realtime streams close; late callbacks, results and submissions are fenced. On reinstatement: memberships, grants, revisions, approvals, facts and evidence are exactly as they were; nothing queued, cancelled or held is resumed or replayed — a new Run needs a new human command, a held delivery a human retry, an import a new start. Runs already complete keep their verdicts. |
| Q4 | Who is told, and how | When a tenant is suspended, every active Owner and Admin receives an email naming the tenant, that access is paused, that accepted runs are not failed tests and nothing is rewritten, the configured recovery contact when there is one, and that no time is promised; when it is reinstated, the same people receive an email that access is restored and that nothing was resumed automatically. Members are not emailed: they meet N20 and the chooser card. No message names the reason. |
| Q5 | How long a suspension lasts | Until the platform reinstates the tenant or starts its deletion as a separate audited act. No automatic transition exists, no configured maximum moves the tenant on its own, and N20 and the messages promise no time. While suspended, every datum, membership, grant, revision, approval and fact of the tenant is untouched. |
| Q6 | What a person can still do | Personal account security, sign-out, the tenant chooser, entering other active tenants and accepting invitations to other tenants work as usual. A Member or Admin may leave the suspended tenant from personal settings (specification 015), since that touches only their own membership. An Owner cannot transfer ownership or leave while the tenant is suspended; an identity erasure request that needs a last-Owner transfer in a suspended tenant is held, and the erasure surface says so. |

Four rules below are derived from the design and from earlier specifications rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** N20 offers an alternate tenant entry — a route to the tenant chooser — only when the person holds an active membership in another active tenant; otherwise the screen states, as drawn, that no alternate tenant entry is offered.
- **D-b** A suspended *membership* in an *active* tenant never shows N20: that person meets the chooser card of specification 009 (S-48) and the denied-scope handling of specification 006. N20 belongs to the tenant's state alone.
- **D-c** The tenant's status is read at every derivation of authority (specification 009 RF-3.1, RF-5.2), so the freeze takes effect at the next request of every member, job and subscription without any push to clients; a request in flight when the status changes is decided under the status the server holds when it commits.
- **D-d** While a tenant is *deleting*, every member-facing rule of this specification applies as for *suspended* — the same screen, the same freeze, the same unreachability — with the heading and statements of the deleting variant as drawn on N20 / Deleting tenant (`401:2714`; this specification owns that screen, 013 owns the chooser card `401:2922`; S-55, S-57); what deletion additionally does (tombstones, purge, finality) is not this specification.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | member of a tenant that was just suspended | to see one clear page saying the workspace is paused, what that means and what I can still do | I am not left guessing from errors |
| US-2 | Owner of a suspended tenant | to be emailed that it is paused, with the recovery contact if there is one | I can act even if I am not signed in |
| US-3 | Member of a suspended tenant who also belongs to other tenants | to reach my other workspaces and my account security as usual | one customer's situation does not stop my other work |
| US-4 | person with a run accepted before the suspension | the platform to stop it at a safe point, keep every recorded fact and never mark it failed | the evidence stays honest |
| US-5 | Owner whose tenant is reinstated | to find my projects, members, grants, approvals and run history exactly as I left them, and to be told that nothing was resumed by itself | I decide what to run again |
| US-6 | Admin of a suspended tenant | to be unable to change members, grants or policy while paused, and to know why | nothing drifts during the freeze |
| US-7 | Member who wants out of a suspended tenant | to leave it from my personal settings | I am not tied to a paused workspace |
| US-8 | person on a phone or tablet | the suspended screen and its panel reflowed with every statement and action intact | I can understand my situation anywhere |
| US-9 | platform operator (internal) | every suspension and reinstatement to be recorded with who, when and why | the act is reviewable |
| US-10 | attacker, curious member or over-reaching administrator (negative story) | — | I cannot learn why a tenant is suspended, read a project name or a run through the paused state, launch, deliver, import or query anything in it, make a cancelled attempt look like a failure, resume frozen work by reinstatement, suspend or lift a suspension from inside the product, or lift it through a password reset or an identity recovery |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the tenant* means the tenant whose status is suspended; *member* means an identity holding a membership in the tenant in the active or suspended state; *the screen* means N20; *the platform mechanism* means the internal, audited, out-of-band act of Q1 (A-62); *the recovery contact* means the channel and instructions the platform mechanism configured for the suspension; *work in flight* means queued run tickets, running attempts, delivery jobs, import jobs and AI requests of the tenant.

### RF-1 · Entering suspension

Suspension is one platform act whose effects follow from the next request on. *Why:* FR-IDN-116 — freeze new effects, refresh permissions and subscriptions, request bounded cancellation; the failure section — freeze before retrieval, acceptance, approval, launch, provider submission or delivery. *(Decision: Q1, Q3, Q4; D-c.)*

- RF-1.1 The system shall move a tenant from active to suspended only through the platform mechanism, recording the acting platform actor, the time, the reason category and, when configured, the recovery contact; no tenant-side action, policy, membership change, invitation or payment event shall move the status. *(FR-IDN-093, FR-IDN-116; decision Q1; A-62; scope §2)*
- RF-1.2 When the status becomes suspended, the system shall yield no customer authority for any membership of the tenant from the next derivation on — no project content, no acceptance, no approval, no launch, no administration of members, grants or policy, no export, no delivery, no import, no AI request — whatever the membership's role, status and grants. *(FR-IDN-116, FR-X-001; specification 009 RF-5.2; D-c)*
- RF-1.3 When the status becomes suspended, the system shall refresh the permissions of every member's open session at its next request, close every realtime subscription and stream of the tenant, and stop every polling or background request of the tenant from obtaining anything. *(FR-IDN-116, FR-IDN-064; specification 009 RF-5.3)*
- RF-1.4 When the status becomes suspended, the system shall refuse every queued, not-yet-admitted run ticket of the tenant at its next checkpoint and record it as NOT EXECUTED, never as a failed test; shall request a bounded cancellation of every running attempt at its next checkpoint, keeping every fact already recorded and rewriting no verdict; and shall never present a cancellation as instant or as a rollback. *(FR-IDN-116, FR-IDN-067, FR-PRJ-043; decision Q3; specification 009 RF-7.3, RF-7.4; P8)*
- RF-1.5 When the status becomes suspended, the system shall stop every delivery job of the tenant before its next provider write, hold its unconfirmed items in their current reconciliation state, and never retry a held or unknown item on its own; shall stop every import job and every AI request of the tenant at its next checkpoint; and shall reserve or consume nothing further on the tenant's behalf. *(FR-IDN-116, FR-RUN-096; decision Q3)*
- RF-1.6 While the tenant is suspended, the system shall fence every late result, runner submission, provider callback or delivery receipt that names the tenant: it shall be recorded as received and fenced, applied to no revision, fact, verdict or delivery state, and answered without revealing the tenant's state beyond the refusal. *(failure section; FR-RUN-050; D-d)*
- RF-1.7 When the status becomes suspended, the system shall append the audit entry of RF-10.1 and send the suspension message of RF-10.4 to every active Owner and Admin of the tenant. *(FR-IDN-116, FR-X-044, FR-X-045; decision Q4)*
- RF-1.8 Suspension shall change no membership status, tenant role, project grant, permission version, policy revision, revision, approval, run, attempt, fact, artifact, delivery receipt, source snapshot or configuration of the tenant; the freeze is a derivation outcome of the tenant's status, not a rewrite of its rows. *(FR-IDN-093, FR-IDN-116; specification 009 RF-5.4; decision Q5)*

### RF-2 · Every surface while suspended

One status, read everywhere, with one outcome. *Why:* FR-IDN-116 — freeze; FR-IDN-111 — the state is shown, the content is not. *(Decision: Q1, Q3; D-b, D-c, D-d.)*

- RF-2.1 While the tenant is suspended, the system shall route every request of a member for any tenant surface — overview, projects, suites, sources, review, runs, queue, library, environments, chat, settings, members, policy — to the screen (RF-3) instead of the surface, and shall serve no tenant content on the way. *(FR-IDN-116, FR-IDN-111; specification 009 RF-5.2, specifications 010 D-b and 011 D-a)*
- RF-2.2 While the tenant is suspended, the system shall refuse every tenant-scoped request of a member that is not a request for the screen, its panel, sign-out or the chooser with the denied-scope problem of specification 006, carrying no reason and no tenant content; a client that receives it shall apply the denied-scope handling and reach the screen. *(FR-IDN-064, FR-X-033; specification 006 RF-5.1)*
- RF-2.3 While the tenant is suspended, the tenant chooser shall list the tenant as the suspended-tenant card of specification 013 RF-5.1 — the tenant name, the state "Workspace suspended", the person's role, no project name, source, run detail or grant — and its "View access status" shall open the screen (RF-3) for that tenant without deriving tenant authority and without counting as an entry (specification 013 RF-5.4); a return from the screen re-derives the chooser (specification 013 RF-7.1). *(FR-IDN-110, FR-IDN-111; N03; specification 013)*
- RF-2.4 While the tenant is suspended, the guarded switch into the tenant shall end on the screen without loading tenant data or opening a stream, and a switch from the suspended tenant to another active tenant shall proceed as usual. *(FR-IDN-130; specification 016)*
- RF-2.5 While the tenant is suspended, an invitation to it shall stay pending and unusable, as specification 008 RF-5.4 states; no invitation of the tenant shall be issued, resent or revoked, and none shall expire differently because of the suspension. *(FR-IDN-116; specification 008 D-d, specification 010)*
- RF-2.6 While the tenant is suspended, no export, delivery retry, reconciliation action, archive, restore, purge request or deletion of content shall be accepted for the tenant. *(FR-IDN-116, FR-IDN-125 by analogy, FR-RUN-096)*
- RF-2.7 While the tenant is suspended, every member's authored revisions, accepted mappings, approvals, run facts, evidence and provenance shall remain stored and attributed; nothing shall be read by a member and nothing shall be removed. *(FR-IDN-094, FR-IDN-116; decision Q5)*
- RF-2.8 The suspension of one tenant shall have no effect on any other tenant of any of its members: their memberships, grants, sessions, chooser entries, invitations and work elsewhere continue unchanged. *(FR-IDN-002, FR-IDN-120; P12; decision Q6)*
- RF-2.9 While the tenant is deleting, the member-facing rules of RF-2 and RF-3 shall apply as for suspended; the deletion feature owns what happens to the data. *(FR-IDN-093, FR-IDN-125; D-d)*

### RF-3 · The suspended-tenant screen

One page that says what is paused, what is protected, and what remains. *Why:* FR-IDN-116 — the screen states that accepted runs do not become failed tests and that verdicts are not rewritten, and offers only the approved verified recovery process and personal account security / sign-out. *(Decision: Q2, Q6; D-a.)*

- RF-3.1 The system shall present the screen under the product header with the person's display name and the interface-language switch (EN / ES / FR), the marker "Workspace suspended", the heading "Workspace access is paused.", the statement "{tenant} is currently suspended. Your identity remains protected; restricted projects, sources and run details are not exposed in this state.", the card "Your current membership" with Tenant, "Your workspace role" and "Access state — Suspended · content unavailable", the statement "Membership metadata is shown only for your identity. No global customer directory or other workspace content is available.", the action "Review verified workspace recovery", the notice "Accepted runs do not become failed tests — This protected state does not rewrite run verdicts or imply accepted jobs have stopped. Restored authorized access is needed to inspect their current lifecycle.", the card "Safe next steps" with "Use the approved workspace recovery process available to your verified identity. If a workspace recovery contact is configured, use only the contact shown through that verified process.", the alternate-tenant statement of RF-3.4, and the actions "Personal account security" and "Sign out". *(FR-IDN-116, FR-IDN-111, FR-X-043; N20)*
- RF-3.2 The screen shall show, of the tenant, only its name and the person's own tenant role and membership state; it shall never show the reason category, the actor, the time of suspension, any project, source, run, member, invitation, policy or other tenant. *(FR-IDN-111, FR-IDN-116; decision Q1; P3)*
- RF-3.3 The screen shall be shown only to an identity holding a membership in the tenant in the active or suspended state; an identity with a revoked membership, or none, shall never reach it and shall see the tenant nowhere. *(FR-IDN-110; D-b)*
- RF-3.4 When the person holds an active membership in another active tenant, the screen shall offer an alternate tenant entry leading to the tenant chooser; otherwise it shall state "No other active permitted workspace is available, so no alternate workspace entry is offered. Personal account security and sign-out remain available." *(FR-IDN-110; N20; D-a)*
- RF-3.5 When "Personal account security" is activated, the system shall route to the security settings of the person's identity (specification 003's TOTP & recovery area and specification 007's Sessions area), which are identity-scoped and unaffected by the suspension. *(FR-IDN-116, FR-SET-020; specification 007 RF-9.2)*
- RF-3.6 When "Sign out" is activated, the system shall perform the sign-out of specification 007 RF-2 and return to the sign-in screen; the suspension, the membership and the frozen work shall be untouched. *(FR-IDN-068; specification 007 RF-9.2)*
- RF-3.7 The screen shall offer no action that reads, changes or requests anything in the tenant: no retry, no reinstatement request, no contact form, no link into tenant content. *(FR-IDN-116; decision Q1, Q2)*
- RF-3.8 The screen shall be the same for an Owner, an Admin and a Member except for the role shown and the alternate-tenant statement; a suspended membership in the suspended tenant shall show "Suspended" as the role's membership state without changing anything else. *(FR-IDN-111; S-57)*
- RF-3.9 When the language switch is used, the system shall change the display only. *(FR-X-043)*
- RF-3.10 If the tenant is reinstated while the screen is open, then the person's next request shall reach the tenant as usual and the screen shall not be shown again; if a suspended tenant's status is read as deleting, then the deleting variant of the screen (D-d) shall be shown. *(FR-IDN-093; decision Q5)*

### RF-4 · The verified workspace recovery panel

The only route out is a contact the platform chose, shown read-only. *Why:* FR-IDN-116 offers "only the approved verified recovery process"; scope §2 forbids a customer-facing operator surface. *(Decision: Q2.)*

- RF-4.1 When "Review verified workspace recovery" is activated, the system shall present a panel on the screen stating that recovery of the workspace goes through the platform's verified process and, when a recovery contact is configured for this suspension, the contact's channel and instructions with the statement that only this contact is to be used; the panel shall offer only "Close". *(FR-IDN-116; decision Q2; S-57)*
- RF-4.2 If no recovery contact is configured, then the panel shall state that no contact is available for this suspension and that the workspace's Owners and Admins will be notified when the state changes (Q4), and shall offer nothing else. *(decision Q2; S-57)*
- RF-4.3 The recovery contact shall be configured, changed or removed only through the platform mechanism, with each change audited; it shall carry no link, token or code that grants access to anything, and no instruction to reset a password or a factor. *(decision Q1, Q2; A-62; FR-X-030)*
- RF-4.4 The panel shall be identical for every member of the tenant and shall expose no reason, no actor, no time, no project content and no other tenant. *(FR-IDN-111; decision Q2)*
- RF-4.5 The panel shall submit nothing and store nothing from the person; opening it shall be audited as a view only. *(decision Q2; FR-X-034)*

### RF-5 · What remains possible for the person

The identity is not paused; only the tenant is. *Why:* FR-IDN-116 keeps personal account security and sign-out; FR-IDN-120 and FR-IDN-121 keep the identity and its other memberships. *(Decision: Q6.)*

- RF-5.1 While the tenant is suspended, every identity-level action of the person shall work as usual: security settings, password reset, controlled account recovery, session management, sign-out, personal settings, the tenant chooser, entering other active tenants, workspace creation, and accepting an invitation to another tenant. *(FR-IDN-116, FR-IDN-002; decision Q6; specifications 003, 004, 005, 007, 012, 013)*
- RF-5.2 A password reset or a controlled account recovery completed by a member shall change nothing about the suspension and shall land, when the suspended tenant is their only one, on the chooser showing the state. *(FR-IDN-043, FR-IDN-048; specification 004 LC-17, specification 005 LC-6)*
- RF-5.3 While the tenant is suspended, a Member or Admin may leave the tenant from their personal settings (specification 015): the membership is revoked, their grants in the tenant are revoked, nothing else of the tenant changes, and the tenant disappears from their chooser. *(FR-IDN-120, FR-IDN-121; decision Q6)*
- RF-5.4 While the tenant is suspended, an Owner shall be unable to transfer ownership, edit any role, manage any membership, invite, or leave; the members surface is unreachable (specification 010) and the leave action of specification 015 shall state that the tenant is suspended. *(FR-IDN-095, FR-IDN-116; decision Q6; specification 010 D-b)*
- RF-5.5 If an identity erasure request requires a last-Owner transfer in a suspended tenant, then the erasure surface shall state that the request is held until the tenant is reinstated or deleted, and the request shall not proceed through that tenant meanwhile. *(FR-IDN-122; decision Q6; specification 015)*
- RF-5.6 The suspension shall never end, revoke or otherwise touch any session of any member; a member's open screens in the tenant meet the denied-scope handling and the screen at their next request. *(FR-IDN-061, FR-IDN-064; specification 006)*

### RF-6 · Reinstatement

Returning to active restores exactly what was there and resumes nothing. *Why:* P1 and P8 — nothing runs, delivers or is replayed without a current human decision; specification 009 RF-5.4 — a tenant's status never changes a membership's status or grants. *(Decision: Q3, Q4, Q5.)*

- RF-6.1 The system shall move a tenant from suspended to active only through the platform mechanism, recording the acting platform actor and the time. *(FR-IDN-093; decision Q1; A-62)*
- RF-6.2 When the status becomes active, every membership shall derive, from its next request on, exactly what its own status and grants state; the tenant's policy revision, project grants and permission versions shall be those recorded before the suspension, unchanged by it. *(FR-IDN-093; specification 009 RF-5.4)*
- RF-6.3 When the status becomes active, the system shall resume, replay or re-admit nothing: tickets ended as NOT EXECUTED stay so; cancelled attempts stay cancelled with their facts; held delivery items stay held until a human retries them under the rules of the delivery feature; stopped imports and AI requests stay stopped; a new Run requires a new human command under a valid approval and current prerequisites. *(FR-IDN-067, FR-RUN-067, FR-RUN-096; decision Q3; P1)*
- RF-6.4 When the status becomes active, realtime subscriptions may be reopened by members' next requests, the chooser shall list the tenant as active, invitations to the tenant that are still pending and unexpired shall be acceptable again (specification 008 RF-5.4), and the tenant surfaces shall be reachable under the ordinary rules. *(FR-IDN-110; specification 008 D-d)*
- RF-6.5 When the status becomes active, the system shall append the audit entry of RF-10.1 and send the reinstatement message of RF-10.4 to every active Owner and Admin of the tenant, stating that access is restored and that nothing was resumed automatically. *(FR-X-044, FR-X-045; decision Q4)*
- RF-6.6 A member whose membership was suspended or revoked, or whose grants changed, before the suspension shall return to exactly that state; the suspension shall neither reinstate nor restore anything of a membership. *(FR-IDN-093; specification 009 RF-5.4)*

### RF-7 · The boundary with deletion

Suspension is reversible and bounded by nothing but the platform's next act. *Why:* FR-IDN-125 makes deletion a durable scoped job of its own; the constitution forbids invented durations and instant-erasure promises. *(Decision: Q5.)*

- RF-7.1 The system shall move a suspended tenant to deleting only through the platform mechanism's separate, audited deletion act; no configured time, no absence of reinstatement and no member action shall move it. *(FR-IDN-093, FR-IDN-125; decision Q5; A-62)*
- RF-7.2 No surface, message or audit entry of this feature shall promise or imply a duration of the suspension, a date of deletion or a date of reinstatement. *(FR-IDN-116; decision Q5; scope "no inferred infrastructure")*
- RF-7.3 While suspended, the tenant's data shall be retained in full under the ordinary retention rules; suspension shall start no purge, no tombstone and no backup expiry. *(FR-IDN-125; decision Q5)*
- RF-7.4 A tenant in deleting state shall never return to suspended or active through this feature. *(FR-IDN-125; D-d)*

### RF-8 · What suspension never does

The boundaries that keep the freeze a freeze. *Why:* FR-IDN-116, FR-IDN-043, FR-IDN-048 and P8 are each a promise a suspension could silently break.

- RF-8.1 Suspension shall never end a session, change a password, touch an authenticator or recovery code, change a verified address, or restrict an identity. *(FR-IDN-001, FR-IDN-061; decision Q6)*
- RF-8.2 Suspension shall never change a membership's status, role, grants or permission version, and reinstatement shall never restore a membership or grant revoked earlier. *(FR-IDN-093; specification 009 RF-5.4)*
- RF-8.3 Suspension shall never mark a run, attempt or ticket failed, rewrite a verdict, delete a fact, artifact or piece of evidence, or present a cancellation as undone. *(FR-IDN-116, FR-IDN-067; P8)*
- RF-8.4 Suspension shall never disclose its reason, its actor or its time to a member, nor a project name, source, run detail, member or invitation through the screen, the chooser, a problem, a message or any refusal. *(FR-IDN-111, FR-X-033; decision Q1)*
- RF-8.5 No tenant-side surface, policy, role or action shall suspend, reinstate, request the suspension or reinstatement of, or delete a tenant; no password reset, controlled account recovery or re-authentication shall lift a suspension. *(FR-IDN-043, FR-IDN-048; decision Q1; scope §2)*
- RF-8.6 Suspension shall never touch another tenant, nor the platform operator's records, nor any run-scoped service authority already issued to an accepted attempt beyond the bounded cancellation request of RF-1.4. *(FR-IDN-067; P12; A-39)*

### RF-9 · Responsive surface

M21 MobileTenantSuspendedPage (`432:4317`) is the drawn counterpart; tablet follows the D01 responsive policy. *Why:* FR-X-055 — AA reflow on every critical-journey surface; the suspended screen is one a person may meet on any device.

- RF-9.1 The system shall present the screen and its recovery panel in a reflowed layout at mobile and tablet widths that meets WCAG 2.2 AA reflow, with every statement, card, action and route of RF-3 and RF-4 preserved in meaning. *(FR-X-051, FR-X-055; S-02)*
- RF-9.2 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus; when the screen or the panel opens, focus shall enter its heading and follow reading order. *(FR-X-040, FR-X-042, FR-X-053)*
- RF-9.3 At mobile widths the system shall follow M21 MobileTenantSuspendedPage (`432:4317`); at tablet widths the D01 responsive policy. *(FR-X-050; S-02)*

### RF-10 · Audit, notification and secrecy

Every transition is recorded with its cause; no member ever reads it. *Why:* FR-X-044 and FR-IDN-099 — reviewable platform acts; FR-IDN-111 — nothing disclosed. *(Decision: Q1, Q4.)*

- RF-10.1 The system shall append to the audit log: each suspension and each reinstatement with the platform actor, the time and the reason category; each configuration, change or removal of the recovery contact; each ticket refused as NOT EXECUTED, each bounded cancellation requested, each delivery job stopped and each item held, each import or AI job stopped, each stream closed, each late callback, result or submission fenced; each view of the screen and of the recovery panel; each message sent and each delivery failure. Each entry identifies the tenant, the member when one is involved, the time and the calling context with a correlation id. *(FR-X-044, FR-IDN-099)*
- RF-10.2 The reason category, the platform actor and the recovery contact's configuration history shall be visible in the platform's own audit only, never in the tenant's audit trail, never on a surface, never in a message. *(FR-X-044; decision Q1; P3)*
- RF-10.3 Every refusal of this feature shall be a safe problem with a correlation id that states, at most, that the workspace is not available; it shall never name suspension as the cause to a request that is not a member's, nor the reason to anyone. *(FR-X-033; specification 006 RF-11.3)*
- RF-10.4 The two messages of this feature — tenant suspended (tenant name, access paused, accepted runs are not failed tests and nothing is rewritten, the recovery contact when configured, no time promised) and tenant reinstated (access restored, nothing resumed automatically) — shall be sent to the verified address of every active Owner and Admin through the transactional email channel in EN, ES and FR, in each recipient's last-used interface language, with wording that is a design deliverable (S-14); neither shall name the reason, carry a link that grants access, or disclose project content; a delivery failure shall never revert the transition and shall be recorded. *(FR-X-045, FR-X-030; decision Q4; S-14)*
- RF-10.5 The screen and the panel shall write nothing to browser storage; the state of the tenant is a server fact re-read at every request. *(FR-X-034)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Freeze latency.** From the moment a suspension commits, the next derivation of authority for any membership, job, subscription or callback of the tenant yields no customer authority; no cached authority outlives its request. | FR-IDN-116, FR-X-001; D-c |
| RNF-2 | **Nothing fabricated.** No ticket, attempt, run or verdict of the tenant becomes failed, undone or rewritten because of a suspension; cancellation is bounded and its acknowledgement is not rollback. | FR-IDN-067, FR-IDN-116; P8 |
| RNF-3 | **Nothing resumed.** Reinstatement replays, re-admits or retries nothing; every new effect after it is a new human decision. | FR-IDN-067, FR-RUN-067; P1; decision Q3 |
| RNF-4 | **Data integrity.** No row of the tenant — memberships, grants, policy, revisions, approvals, runs, facts, evidence, deliveries, sources — is changed, removed or purged by suspension or reinstatement. | FR-IDN-093, FR-IDN-125; decision Q5 |
| RNF-5 | **Non-disclosure.** No member learns the reason, actor or time of the suspension, and no member reads a project name, source, run detail, member or invitation of the suspended tenant through any surface, problem or message. | FR-IDN-111, FR-X-033 |
| RNF-6 | **Platform-only transitions.** Suspension, recovery-contact configuration, reinstatement and deletion are acts of the platform's internal audited mechanism; no tenant-side path exists. | scope §2; decision Q1 |
| RNF-7 | **Identity untouched.** Sessions, credentials, factors, codes, addresses and other memberships of every member are unaffected; identity-level features work as usual. | FR-IDN-001, FR-IDN-002, FR-IDN-061; decision Q6 |
| RNF-8 | **Tenant isolation.** The freeze applies to exactly one tenant; every other tenant of every member is untouched, and the fencing of late callbacks never leaks across tenants. | P12; FR-X-010 |
| RNF-9 | **Fail closed.** An unknown, suspended or deleting tenant status, an absent policy or an absent configuration value yields no customer authority and no effect; nothing is assumed. | P6 |
| RNF-10 | **Configured limits.** The checkpoint cadence of bounded cancellation (execution) and audit retention (audit) are validated configuration owned by their features; this feature owns no numeric value and invents no duration. | FR-X-046; P13; decision Q5 |
| RNF-11 | **Email as prerequisite.** The two messages depend on the transactional email channel; its absence is an operational failure, never a reverted transition or a silent skip. | FR-X-045 |
| RNF-12 | **Accessibility.** The screen and its panel meet WCAG 2.2 AA with automated and manual assistive-technology evidence: focus enters the heading, status is conveyed by text, no focus stealing. | FR-X-040, FR-X-042 |
| RNF-13 | **Language.** The screen, the panel and the two messages exist in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-14 | **Reflow.** Mobile and tablet reflows preserve every statement, card, action and route of the desktop screen. | FR-X-051, FR-X-055 |
| RNF-15 | **Verification.** Revoked-tenant races, fencing of late results and callbacks, bounded cancellation with immutable facts, and deletion and late-callback safety are verified as areas A and H require. | requirements A.12, H verification; P9 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Tenant suspended while a member is editing a scenario | Next request meets the denied-scope handling, then the screen; the draft follows specification 006 RF-9 (kept in memory, shown again only if the refreshed authority permits, which it will not until reinstatement) (RF-2.1, RF-2.2, RF-5.6). |
| LC-2 | Tenant suspended with a run accepted and running | Bounded cancellation requested at the next checkpoint; facts recorded so far stay; verdict never written as failed; the screen says so (RF-1.4, RF-3.1). |
| LC-3 | Tenant suspended with tickets queued and not admitted | Each refused at its next checkpoint as NOT EXECUTED; never a failed test (RF-1.4). |
| LC-4 | Tenant suspended with a delivery job mid-way | Stops before the next provider write; confirmed items stay confirmed; unconfirmed and unknown items held; nothing retried (RF-1.5). |
| LC-5 | Late result from a runner arrives after suspension | Fenced and recorded; applied to nothing; no verdict (RF-1.6). |
| LC-6 | Provider callback (delivery receipt) arrives after suspension | Fenced and recorded; reconciliation state unchanged; a human reconciles after reinstatement (RF-1.6, RF-6.3). |
| LC-7 | Member signs in with the suspended tenant as their only membership | Landing on the chooser showing "Workspace suspended"; selecting it shows the screen (RF-2.3; specification 001 LC-16). |
| LC-8 | Member with another active tenant | Chooser shows both; the suspended card has no content; the screen offers the alternate tenant entry (RF-2.3, RF-3.4). |
| LC-9 | Member types a tenant route of the suspended tenant | The screen; no content served (RF-2.1). |
| LC-10 | Owner tries to transfer ownership, edit a role or invite while suspended | Members surface unreachable; the screen (RF-5.4; specification 010 D-b). |
| LC-11 | Member leaves the suspended tenant from personal settings | Membership revoked; grants revoked; tenant gone from their chooser; the tenant otherwise untouched (RF-5.3). |
| LC-12 | Sole Owner of a suspended tenant requests identity erasure | Held: the erasure surface states the tenant is suspended; nothing proceeds through that tenant (RF-5.5). |
| LC-13 | Owner resets their password while the tenant is suspended | Reset succeeds; suspension untouched; landing on the chooser (RF-5.2). |
| LC-14 | Owner completes controlled account recovery while suspended | Same as LC-13 (RF-5.2; specification 005 LC-6). |
| LC-15 | Invitation to the suspended tenant opened by the invitee | "Can't be accepted"; the invitation stays pending; acceptable again after reinstatement if unexpired (RF-2.5; specification 008 LC-13). |
| LC-16 | Recovery contact configured after the suspension began | The panel shows it from the next opening; the configuration is audited (RF-4.1, RF-4.3). |
| LC-17 | No recovery contact configured | The panel states that none is available and that Owners will be notified (RF-4.2). |
| LC-18 | Member opens the recovery panel on a phone | Reflowed panel with the full statement and "Close" (RF-9.1). |
| LC-19 | Tenant reinstated while a member is on the screen | Their next request reaches the tenant; the screen is not shown again (RF-3.10, RF-6.2). |
| LC-20 | Tenant reinstated with tickets that ended as NOT EXECUTED | They stay NOT EXECUTED; a new Run needs a new human command (RF-6.3). |
| LC-21 | Tenant reinstated with held delivery items | They stay held until a human retries under the delivery feature's rules (RF-6.3). |
| LC-22 | Member suspended in the tenant before the tenant's suspension | On reinstatement their membership is still suspended (RF-6.6). |
| LC-23 | Membership revoked by leaving during the suspension; tenant reinstated | The person is not a member; they return only by invitation (RF-5.3, RF-6.6). |
| LC-24 | Suspended tenant moved to deleting | The deleting variant of the screen; deletion's rules apply; never back to active through this feature (RF-2.9, RF-7.4). |
| LC-25 | Suspended membership in an active tenant | Not this feature: the chooser card of S-48 and the denied-scope handling (D-b). |
| LC-26 | A request in flight when the suspension commits | Decided under the status the server holds when it commits; if after, refused (D-c). |
| LC-27 | Polling tab open in the suspended tenant | Its next request is refused with denied scope; the tab reaches the screen; nothing is served (RF-1.3, RF-2.2). |
| LC-28 | Language switched on the screen or the panel | Display changes only (RF-3.9). |
| LC-29 | Suspension message cannot be delivered to an Owner | The suspension stands; the failure is recorded (RF-10.4). |
| LC-30 | Member looks for the reason on the screen, in the chooser, in a problem or in a message | Nothing names it (RF-3.2, RF-8.4, RF-10.2). |
| LC-31 | Admin looks for a way to reinstate or request reinstatement in the product | None exists; the only route is the recovery contact (RF-3.7, RF-8.5). |
| LC-32 | Founder of a suspended tenant creates a new workspace | Allowed; the new tenant is independent (RF-5.1; specification 012). |
| LC-33 | Identity with a revoked membership in the suspended tenant | Sees the tenant nowhere; never the screen (RF-3.3). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The platform mechanism itself** — how the operator suspends, configures the recovery contact, reinstates or deletes, its tooling and its surface: internal, audited, out-of-band (scope §2; A-62). Only its effects are specified here.
- **Tenant deletion** — FR-IDN-125: the durable job, tombstones, purge order, export; this specification states only that deletion is a separate act and that a deleting tenant shows the member-facing freeze (D-d).
- **A suspended membership in an active tenant** — specification 009 (A-54, S-48).
- **The tenant chooser** beyond the suspended card — specification 013.
- **Sessions, refresh, expiry and the denied-scope handling** — specification 006; **logout** — specification 007; this feature only routes to them.
- **Leaving a tenant and identity erasure** — specification 015; this feature states only which of them a suspended tenant permits.
- **The guarded tenant switch** — specification 016.
- **Execution, admission, bounded cancellation, delivery reconciliation, imports and AI jobs** — their own specifications; this feature states what suspension requires of them and nothing of how they do it.
- **Billing, subscription, abuse or legal processes** that lead to a suspension — not part of the product; the reason category is a platform record only.
- **Tablet-specific designs** of N20 and its panel — none; the D01 responsive policy applies.
- **"Help & shortcuts"** — the shell's N43 HelpShortcutsPanel (`430:4129`) (S-17); no help surface is added.
- **The choice and configuration of the transactional email channel** — an operational prerequisite, not a feature.

---

## 7. Criteria of completion

The tenant suspension state is complete when all of the following hold:

1. Every user story US-1 … US-9 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-10 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-10 has at least one verification that fails before and passes after the feature exists.
3. Freeze tests show that from the next request after a suspension no membership derives customer authority on any tenant surface, API, job, subscription or callback, that permissions and streams are refreshed and closed, and that every tenant route ends on the screen without serving content (RF-1.2, RF-1.3, RF-2, RNF-1, LC-1, LC-7 … LC-10, LC-26, LC-27).
4. Work tests show queued tickets ending as NOT EXECUTED, running attempts receiving a bounded cancellation with facts kept and no verdict rewritten, deliveries stopped with items held and never retried, imports and AI stopped, and late results and callbacks fenced (RF-1.4 … RF-1.6, RNF-2, LC-2 … LC-6).
5. Screen and panel tests show the designed content verbatim, the role and alternate-tenant variants, the recovery contact and the no-contact state, "Personal account security" and "Sign out" working, and no reason, project, source, run, member or invitation disclosed anywhere (RF-3, RF-4, RNF-5, LC-16, LC-17, LC-30).
6. Identity tests show that security settings, reset, recovery, sessions, other tenants, workspace creation and invitations to other tenants work as usual, that a Member or Admin may leave, that an Owner cannot transfer or leave, that a sole Owner's erasure is held, and that no session is ended (RF-5, RNF-7, LC-11 … LC-14, LC-32).
7. Reinstatement tests show every membership, grant, policy revision, revision, approval and fact exactly as before, nothing resumed, replayed, re-admitted or retried, pending invitations acceptable again, and the Owner and Admin messages sent (RF-6, RNF-3, RNF-4, LC-19 … LC-23).
8. Transition tests show that only the platform mechanism moves the status, that no tenant-side path and no reset, recovery or re-authentication lifts a suspension, that no configured time moves a tenant to deleting, and that a deleting tenant never returns through this feature (RF-1.1, RF-6.1, RF-7, RF-8.5, RNF-6, LC-24, LC-31).
9. Isolation tests show that a suspension touches no other tenant and that fenced callbacks never leak across tenants (RF-2.8, RF-8.6, RNF-8).
10. Audit entries exist for every event of RF-10.1 and contain no reason, actor or contact history readable by a member; the two messages exist in EN, ES and FR, reach only active Owners and Admins, carry no reason and no access-granting link, and a delivery failure never reverts a transition (RF-10, RNF-11, LC-29).
11. The screen and its panel pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RNF-12, RNF-13, RNF-14).
12. The concurrency and fencing verifications of areas A and H — revoked-tenant races, late results and callbacks, bounded cancellation with immutable facts — pass on real infrastructure (RNF-15).
13. The register items this specification relies on — A-62, A-56, A-54, A-39 and A-30 in the architecture register; S-57, S-48, S-14, S-02 and S-17 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. Constitution v1.8 (2 November 2026) binds FR-IDN-116 fully in F-IDN-16, covering the recovery panel (Q2) and the chooser route.
