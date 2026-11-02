# Specification 009 — Roles, grants and the authority matrix

**Feature** The finite role model, the authority matrix, membership and tenant statuses, permission versions and revocation · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-11 as the subject; F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-10, F-IDN-12, F-IDN-13, F-IDN-14, F-IDN-15, F-IDN-16, F-IDN-17, F-SET-05, F-X-01, F-X-05 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-06, F-X-07; principles P1, P2, P3, P4, P6, P12, P13, P14, P16) → `requirements.md` (FR-IDN-084, 090, 091, 092, 093, 094, 096, 098, 099, 110, 111, 113, 114, 115, 116, 122; FR-X-001, 002, 003, 004, 005, 006, 040, 042, 043, 044, 046; FR-PRJ-010, 012, 044; FR-APR-006, 020, 021, 022, 023; FR-CAT-082; FR-SET-033, 040, 044, 046, 047, 052; FR-RUN-067; FR-OPS-002; FR-EXT-023) → the design of record, the finite authority matrix and the three-scope summary of D03 `148:4`, the "Authority boundaries" and "Finite roles / Separate authority" blocks of N06 TenantPolicyPage `161:49504`, the statements of R13 TenantMembersPage `148:503`, R17 ProjectMembersPage `148:1754` and N03 TenantChooserPage `161:49326`, the ACCESS ERD and authority table of board 1 `23:5587`, and the BE board `23:5587`.

**Related specifications:** 001 Login (a completed sign-in establishes an identity session that carries no tenant or project authority; landing by memberships) · 003 TOTP enrollment (the enrollment gate that withholds Owner, Admin and policy-required Member authority until a factor exists; applied inside the derivation of RF-3) · 005 Controlled account recovery (recovery never restores a membership) · 006 Sessions, refresh and expiry (how a denied scope and a stale permission version reach the person: one permission refresh, then the denied-scope state) · 008 Invitation acceptance (an invitation creates a membership and grants bound to what the inviter was allowed to offer) · 010 Tenant members and ownership transfer (the members list, role edits, ownership transfer and the last-Owner guard) · 011 Project grants editor (the surface that writes the grant records of RF-9) · 012 Workspace creation (the founding Owner's bootstrap grants) · 013 Tenant chooser and 014 Tenant suspension state (where membership and tenant statuses are shown and what a suspended tenant still offers) · the settings specification of F-SET-05 (editing the tenant policy whose revision this feature checks) · the approval and execution specifications (the C04 decision and the C05 launch that consume the eligibility this feature derives).

This document states **what** roles, grants and the authority matrix are, how authority is derived from them, and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

ASpectra separates three things that other products collapse into one "admin" switch. A **global identity** is a person. A **tenant membership** ties that person to one customer workspace with exactly one tenant role — Owner, Admin or Member — and a status. A **project grant** ties a membership to one project of that tenant with any combination of the project roles Author, Reviewer and Viewer, plus the separate fixed grant `canRun`. Tenant administration never implies project content access: an Owner who holds no project grant reads no project, approves nothing and launches nothing. There is no custom role builder, no team hierarchy and no global customer administrator; the platform operator is a platform principal outside every tenant.

The **authority matrix** of FR-IDN-091 says exactly what each role and grant permits. It is not a description of screens; it is the rule from which the server derives, at every protected action, whether the acting membership may do what it asks. That derivation also reads the membership's status, the tenant's status, the membership's `permissionVersion`, the tenant policy revision and the enrollment gate of specification 003, and it is repeated at every effect boundary — never cached from sign-in, never taken from a browser-supplied tenant or project identifier.

**Revocation** is the other half of the rule: a membership or grant that ends denies every future command and subscription from the next request, stops queued work before it runs, asks running work to stop at a bounded checkpoint, and never pretends that effects already produced did not happen. Every change to a role, a status or a grant advances the membership's `permissionVersion`, so a write prepared under the old authority cannot commit, and every change is recorded with who made it, what it was and what it became.

### 1.2 Objective

Give every protected action in the product one exact, finite, auditable answer to "may this person do this here, now" — while:

- never letting a tenant role imply project content, approval or launch, and never letting any grant imply a tenant role;
- never deriving authority from anything the browser claims, from a session alone, or from a stale version of a membership;
- ending authority the moment a membership or grant ends, without fabricating outcomes for work already done;
- letting a founding Owner operate alone with explicit, visible, audited grants rather than implied ones;
- keeping the role model closed: no custom roles, no hierarchy, no escalation by invitation or self-assignment.

### 1.3 Why it matters

A test platform holds a customer's target addresses, credentials by reference, authored intent and execution evidence. The person who administers who belongs to the workspace is not necessarily the person entitled to read its projects, and the person who may author a scenario is not necessarily the person who may approve or run it against a live target. Collapsing those into one role is how a compromised administrator reaches every project, how an author approves their own work unnoticed, and how a revoked contractor keeps a run going. A matrix that is exact, derived on the server and rechecked at every effect is what makes the rest of the product's promises — acceptance ≠ approval ≠ Run, server-derived context, fail closed — true in practice.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Decisions that depend on an architecture or design decision are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row (architecture `122:2` §14 as A-nn, D03 Screen decisions ledger `148:4` as S-nn).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | The finite role model; the authority matrix as the rule every protected action is derived from; membership and tenant statuses with `permissionVersion`; revocation semantics; versioning and audit of changes; the statements every surface must carry. The surfaces that edit roles and grants stay elsewhere: invitations in 008, the members list and ownership transfer in 010, the project grants editor in 011, the workspace bootstrap in 012, the chooser in 013, the suspended-tenant state in 014, tenant policy editing in the settings specification. |
| Q2 | Valid grant combinations; `canRun` without a project role | Any subset of Author, Reviewer and Viewer may be held; Viewer adds nothing beside Author or Reviewer, since both include authorised read, but may still be recorded. `canRun` may be granted to any member as an explicit record and launches only when the holder also holds Author or Reviewer; otherwise it is inert and the surface says so. A record with no role and no `canRun` is no grant: the member is not a member of that project. Register A-53; D03 ledger S-47 (D02 `419:2645`, `420:2968`). |
| Q3 | What "suspended" and "revoked" mean | Suspended: the membership grants nothing in the tenant — no administration, no project content, no launch — its subscriptions are dropped, but the record and its project grants are retained so that reinstatement restores them unchanged; the tenant appears in the person's chooser with the state "suspended" and no project names. Revoked: the membership is ended; its project grants are revoked in the same change and kept only as audit history; the tenant no longer appears in the chooser; the person's authored revisions and evidence remain in the tenant with their provenance. Both transitions advance `permissionVersion`. Register A-54; D03 ledger S-48 (D02 `419:2645`, `420:2968`). |
| Q4 | What `permissionVersion` covers and what an advance does | One version per membership, advanced by any change to that membership's tenant role, status or any of its project grants, `canRun` included. A sensitive write carrying an older version is refused and the client refreshes permissions once (specification 006 RF-5.1). A tenant policy change advances the policy's own revision, which approval and launch check separately; it does not advance members' versions. The version is part of the server-derived context and is never supplied by a client as authority. Register A-55. |
| Q5 | Revocation and work in flight | New commands and subscriptions are denied from the next request. A queued ticket of the member not yet admitted is refused at its next checkpoint and ends as NOT EXECUTED, never as a failed test. A running attempt receives a bounded cancellation request; facts already recorded stay, the verdict is never rewritten, and no external effect is claimed undone. Open streams are closed. Unsaved client work follows specification 006 (kept only if the refreshed scope still permits the edit). Authored revisions and evidence remain with their provenance. Register A-56. |
| Q6 | Who may change which roles and grants, including their own | Tenant roles: an Owner may set any member's role, make another Owner, or hand over ownership (010); an Admin may set Member or Admin on non-Owner memberships, may neither create nor touch an Owner, and may not change their own tenant role; nobody may raise their own tenant role; an Owner lowers their own only through ownership transfer. Project grants: an Owner or Admin may set any member's grants, including their own, visibly and audited — this is how a solo Owner works; a Member may change nobody's grants, their own included. Every change is audited with actor, previous value and new value. Register A-57; D03 ledger S-49 (D02 `419:2645`, `420:2968`). |
| Q7 | Which grant qualifies an Owner or Admin for whole-project lifecycle | Any project role grant — Author, Reviewer or Viewer — on that project; `canRun` alone never; a Member never, whatever grants they hold. Content lifecycle inside a project (suite, feature, scenario) requires Author, whatever the tenant role. Register A-58. |

Register items this specification relies on: **A-53…A-58** and **S-47…S-49** (decided here), and the pre-existing **A-36** (identity-scoped membership listing, on which the chooser states of Q3 depend), **A-39** (revocation scope of sessions, consistent with Q5), **A-42** (Member MFA policy gate timing, applied in RF-3), **A-11** (configuration values), **S-06** (role variants of the screens, D02 `425:3736`; the statements of RF-10 are specified for every variant) and **S-14** (no notification is sent by this feature; none is added to the inventory).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | founding Owner working alone | to hold Author, Reviewer and `canRun` on my first project as explicit grants I can see and change | I can author, approve and run without an implied right I cannot audit |
| US-2 | tenant Owner | to administer memberships and policy without automatically reading every project | a compromised administrator account does not expose every customer project |
| US-3 | tenant Admin | to manage Members and other Admins within the limits an Owner delegates | day-to-day administration does not require the Owner |
| US-4 | project Author | to author and accept mappings in my project and to be told plainly that I cannot approve or run | the boundary between authoring and authorising is visible |
| US-5 | project Reviewer | to approve exact revisions, and to be prevented from approving my own when the tenant policy says so | approval is a deliberate human decision under the tenant's rules |
| US-6 | project Viewer | to read what I am authorised to read and nothing more | I can inspect evidence without being able to change or run anything |
| US-7 | member holding `canRun` | to launch only what a Reviewer approved and only while I also hold Author or Reviewer | a launch is never an accident of a leftover grant |
| US-8 | administrator revoking a contractor | their access to end from the next request and their queued work to stop before it runs, while what they authored stays | revocation is immediate and honest |
| US-9 | member whose grant changed while I was working | my next sensitive write to be refused and my permissions refreshed | I never commit under authority I no longer have |
| US-10 | person suspended in one tenant | my other tenants to be unaffected and my place in the suspended one kept for reinstatement | one tenant's decision does not reach my other work |
| US-11 | person reading any membership, grant or policy surface | the same statements everywhere about what membership is not, what each role grants and that permission is rechecked at each action | the product's promise is consistent |
| US-12 | attacker or over-reaching party (negative story) | — | I cannot read a project through a tenant role, approve or run through authoring, escalate myself by invitation or self-assignment, keep a revoked grant alive, commit under a stale version, make the operator a tenant role, or obtain authority from a browser-supplied identifier |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *membership* means one identity's tie to one tenant with its tenant role and status; *grant* means one membership's project record of project roles and `canRun`; *derivation* means the server's computation, for one request, of what the acting membership may do; *protected action* means any read, write, decision, launch, subscription, download, export or delivery that touches customer content.

### RF-1 · The finite role model

The set of roles is closed and small. *Why:* every additional role is a new path to audit and a new way to be wrong; the product sells a model a customer can hold in their head. *(Decision: Q1.)*

- RF-1.1 The system shall provide exactly three tenant roles — Owner, Admin, Member — and shall hold exactly one of them on each membership. *(FR-IDN-090)*
- RF-1.2 The system shall provide exactly three project roles — Author, Reviewer, Viewer — and one separate fixed project grant, `canRun`; a grant shall hold any subset of the three roles and the `canRun` flag. *(FR-IDN-090; decision Q2)*
- RF-1.3 The system shall provide no custom role, no team or group, no role hierarchy and no inheritance between tenant roles and project roles; a role shall mean exactly what RF-2 states and nothing more. *(FR-IDN-090; FR-SET-043)*
- RF-1.4 The system shall hold at most one membership per identity per tenant, and shall tie every grant to one membership and one project of the same tenant; no grant shall exist without its membership, and no membership shall reach another tenant's project. *(FR-IDN-090; P12)*
- RF-1.5 The platform operator shall be a platform principal outside every tenant membership and every project grant; it shall never be a tenant role, never be grantable, delegable or reachable from inside a tenant, and never read customer content as a side effect of configuration. *(FR-OPS-002)*
- RF-1.6 The system shall treat tenant membership and project grants as the only source of authority; an email domain, a federated sign-in, a verified address or a global identity attribute shall never grant membership or a grant. *(FR-EXT-023; scope "no membership by domain")*

### RF-2 · The authority matrix

Each role and grant permits exactly what this section states. *Why:* FR-IDN-091 is to be implemented exactly; the rows below are its contract, extended with the decisions that close its gaps. *(Decision: Q2, Q7.)*

- RF-2.1 An Owner shall be able to administer the tenant: its lifecycle, its policy, its memberships and ownership, subject to the last-active-Owner protection of specification 010. *(FR-IDN-091; FR-IDN-095 via 010)*
- RF-2.2 An Admin shall be able to administer the tenant within delegated scope as RF-8 states, and shall never be able to evade the last-active-Owner protection. *(FR-IDN-091; decision Q6)*
- RF-2.3 A Member shall hold no tenant administration authority. *(FR-IDN-091)*
- RF-2.4 A tenant role shall grant no project content access, no approval and no launch: an Owner, Admin or Member reads, authors, reviews, approves or runs in a project only through an explicit grant on that project. *(FR-IDN-091, FR-IDN-092; FR-SET-044)*
- RF-2.5 An Author shall be able, within the project, to read, draft and author canonical revisions, accept semantic mappings, import sources, upload assets and author macros; Author shall imply no tenant role, no approval and no launch. *(FR-IDN-091; FR-CAT-059 via its specification)*
- RF-2.6 A Reviewer shall be able, within the project, to read and review, to record the approval decision that publishes an exact revision, and to publish macro and asset revisions; Reviewer shall imply no tenant role and no launch. Where the tenant policy requires a different approver, a Reviewer shall not approve a revision they authored. *(FR-IDN-091; FR-APR-020, FR-APR-021; FR-CAT-034)*
- RF-2.7 A Viewer shall be able to read authorised project content only; a Viewer shall never create a revision, accept a mapping, approve, launch or delete. *(FR-IDN-091; FR-PRJ-012)*
- RF-2.8 `canRun` shall authorise a launch only when, at the moment of launch, the holder also holds Author or Reviewer on the project, a valid approval exists for the exact bundle, the current tenant policy permits it and the current admission prerequisites hold; `canRun` shall grant no author, review or approval access and shall be inert without Author or Reviewer. *(FR-IDN-091, FR-RUN-067, FR-APR-022; decision Q2)*
- RF-2.9 Whole-project lifecycle — archive, trash, restore, purge request and deletion of a project — shall require a tenant Owner or Admin who also holds Author, Reviewer or Viewer on that project; `canRun` alone shall never qualify, and a Member shall never act on whole-project lifecycle whatever grants they hold. *(FR-PRJ-044; decision Q7)*
- RF-2.10 Content lifecycle inside a project — suites, features, scenarios — shall require Author on that project, whatever the tenant role; Reviewer, Viewer or `canRun` alone shall never authorise deletion. *(FR-PRJ-044; decision Q7)*
- RF-2.11 Publishing public trust material shall require two authorities at once: a Reviewer grant on the project and a tenant Owner or Admin role. *(FR-CAT-082)*
- RF-2.12 Visibility of a project's audit payload shall follow the viewer's current project authority; a tenant Owner or Admin role alone shall never authorise reading it. *(FR-X-044)*
- RF-2.13 An export shall include only project content the acting membership is currently authorised to read; an organisation-wide export shall require that authority on every selected project, and tenant administration shall never bypass it. *(register §3; FR-X-044)*
- RF-2.14 Self-approval shall be permitted by default so that a solo Owner can operate, shall always be visible and audited, and shall be tightened to "a reviewer other than the author" when the tenant policy `requireDifferentApprover` says so; the policy shall change decision eligibility only and shall never remove the separate launch requirement. *(FR-APR-021, FR-APR-022, FR-SET-041)*

### RF-3 · Derivation and recheck

Authority is computed on the server for every protected action from current facts. *Why:* P3 and F-X-01 — the browser expresses intent; a role seen at sign-in is not a role held now. *(Decision: Q4.)*

- RF-3.1 When a protected action is requested, the system shall derive the acting membership's authority from, and only from: the identity of the session (specification 006), the membership of that identity in the requested tenant and its status, the tenant's status, the grants of that membership on the requested project, the membership's `permissionVersion`, the current tenant policy revision, and the enrollment gate of specification 003. *(FR-X-001; FR-IDN-093; specification 003 RF-1.3–1.5)*
- RF-3.2 A tenant or project identifier supplied by a client — in a route, a body or a credential — shall be a requested scope only; if the acting identity holds no active membership in the requested tenant, if the route and body disagree on ownership, or if context is missing, then the system shall deny safely without disclosing whether the tenant or project exists. *(FR-X-002; P3)*
- RF-3.3 The system shall repeat the derivation at every effect boundary — retrieval, provider submission, semantic acceptance, save, approval, launch, cancellation, download, delivery, export, subscription — never only at sign-in or tenant entry. *(FR-X-004)*
- RF-3.4 Every asynchronous job, retry and detached transaction acting for a tenant shall derive authority afresh for its own scope; an accepted Run shall continue under its own narrow run-scoped authority, which is not the launching member's membership. *(FR-X-003, FR-IDN-067)*
- RF-3.5 While the enrollment gate of specification 003 withholds a membership's authority, the derivation shall yield no privileged or customer authority for that membership, whatever its role and grants. *(FR-IDN-023, FR-IDN-026; specification 003 Q12, A-42)*
- RF-3.6 If any fact of RF-3.1 is absent, stale, suspended, revoked, deleting or unknown, then the derivation shall yield no authority for the action. *(P6)*
- RF-3.7 The execution kernel and the runner shall receive only an opaque execution context and never a role name, a membership, a grant or a user directory. *(FR-X-006; P11)*
- RF-3.8 When the derivation yields no authority for the requested tenant or project, the person shall reach the denied-scope handling of specification 006 (one permission refresh, then the denied-scope state) and never a sign-in loop. *(FR-IDN-064; specification 006 RF-5.1)*

### RF-4 · Membership status

A membership is active, suspended or revoked, and each state has one meaning. *Why:* FR-IDN-093 names the states; without one meaning each, a suspended member's residual access would be decided screen by screen. *(Decision: Q3.)*

- RF-4.1 The system shall hold on every membership exactly one status among `active`, `suspended`, `revoked`, and a `permissionVersion`. *(FR-IDN-093)*
- RF-4.2 While a membership is `active`, its tenant role and grants shall be derivable as RF-2 states. *(FR-IDN-093)*
- RF-4.3 While a membership is `suspended`, the system shall derive no authority from it — no tenant administration, no project content, no approval, no launch — shall close its subscriptions, and shall retain the membership record and all its grants unchanged for reinstatement; the tenant shall appear in the person's chooser with the state "suspended" and no project names. *(FR-IDN-093, FR-IDN-110, FR-IDN-111; decision Q3)*
- RF-4.4 When a membership is `revoked`, the system shall end it, shall revoke every grant of that membership in the same indivisible change, shall keep the grants only as audit history, and shall no longer list the tenant in the person's chooser; a later invitation to the same tenant, once accepted, shall return the same membership record to active with a new version, the invited tenant role and no former grant (RF-1.4; specification 008 Q5; specification 010 RF-7.10). *(FR-IDN-093, FR-IDN-094; decision Q3; register "old privileges do not return")*
- RF-4.5 When a membership is suspended, reinstated or revoked, the system shall advance its `permissionVersion` and record the change (RF-11). *(FR-IDN-093, FR-IDN-099; decision Q3, Q4)*
- RF-4.6 Revocation of a membership shall leave the person's authored revisions, accepted mappings, approvals, runs and evidence in the tenant with their provenance intact; it shall never erase the global identity or touch the person's other memberships. *(FR-IDN-094, FR-IDN-098, FR-IDN-120)*

### RF-5 · Tenant status

A tenant is active, suspended or deleting, and the derivation reads it before anything else. *Why:* the tenant's state bounds every membership in it.

- RF-5.1 The system shall hold on every tenant exactly one status among `active`, `suspended`, `deleting`. *(FR-IDN-093)*
- RF-5.2 While a tenant is `suspended` or `deleting`, the derivation shall yield no customer authority for any membership of that tenant — no project content, no approval, no launch, no administration of content — and the person shall be offered only what specification 014 states: the approved verified recovery process, personal account security or sign-out and, where the person holds an active membership in another active tenant, a route to the tenant chooser (specification 014 RF-3.4). *(FR-IDN-116; specification 014)*
- RF-5.3 When a tenant becomes `suspended` or `deleting`, the system shall refresh permissions and subscriptions for every member, freeze new effects and request bounded cancellation of active work; accepted runs shall not become failed tests and no verdict shall be rewritten. *(FR-IDN-116, FR-IDN-067)*
- RF-5.4 A tenant's status shall never change a membership's status or grants; when a tenant returns to `active`, each membership shall derive exactly what its own status and grants state. *(FR-IDN-093)*

### RF-6 · The permission version

Every change of authority is a new version, and a stale version cannot commit. *Why:* FR-X-005 — the race between an administrator's change and a member's in-flight write must be lost by the write. *(Decision: Q4.)*

- RF-6.1 The system shall hold one `permissionVersion` per membership and shall advance it whenever that membership's tenant role, status, or any of its project grants — `canRun` included — changes. *(FR-IDN-093; decision Q4)*
- RF-6.2 The version shall be part of the server-derived context of every protected action; a client shall never supply it as authority. *(FR-X-001, FR-X-002; P3)*
- RF-6.3 When a sensitive write is committed, the system shall verify, within the same transaction, that the membership's version is the one the authority was derived under; if it is not, then the write shall be refused as stale and nothing shall commit. *(FR-X-005; decision Q4)*
- RF-6.4 When a write is refused as stale, the system shall refresh the person's permissions once (specification 006 RF-5.1) and shall never retry the write on its own; the surface then either offers the action again under the refreshed authority, states the reason for the refusal as its own specification defines (specification 010 RF-10.4, specification 011 RF-4.6), or shows the denied-scope state when the scope is gone. *(specification 006 RF-5.1; decision Q4)*
- RF-6.5 A tenant policy change shall advance the policy's own revision, which approval and launch check at the moment of decision; it shall not advance any membership's `permissionVersion`. *(FR-APR-023, FR-SET-040; decision Q4)*
- RF-6.6 The version shall only ever advance; two changes to one membership shall never share a version. *(FR-IDN-099)*

### RF-7 · Revocation

Ending a membership or a grant ends authority now, stops work that has not started, asks running work to stop, and rewrites nothing. *Why:* FR-IDN-094 and P8 — a revocation that kills work mid-flight and marks it failed fabricates an outcome. *(Decision: Q5.)*

- RF-7.1 When a membership is revoked or suspended, or a grant is removed or narrowed, the system shall deny every later command and subscription that depended on it from the next request on. *(FR-IDN-094; decision Q5)*
- RF-7.2 When authority is revoked, the system shall close the affected person's open subscriptions and streams in the affected scope. *(FR-IDN-094, FR-IDN-116)*
- RF-7.3 If a queued run ticket of the affected membership has not been admitted when its authority ends, then the system shall refuse it at its next checkpoint and record it as NOT EXECUTED, never as a failed test. *(FR-IDN-094, FR-IDN-067; decision Q5)*
- RF-7.4 If an attempt of the affected membership is running when its authority ends, then the system shall request a bounded cancellation at the next checkpoint where possible; the facts already recorded shall remain, the verdict shall never be rewritten, and no external effect shall be presented as undone. *(FR-IDN-094, FR-IDN-067; P8; decision Q5)*
- RF-7.5 Revocation shall leave authored revisions, accepted mappings, approvals and evidence in place with their provenance; it blocks new work, it does not erase history. *(FR-IDN-098, FR-SET-052)*
- RF-7.6 Unsaved client work of the affected person shall follow specification 006: kept in memory and shown again only if the refreshed authority still permits the edit, otherwise discarded. *(specification 006 RF-5.3, RF-9.3)*
- RF-7.7 Revocation of a single grant shall affect only that project; revocation of a membership shall affect every project of that tenant; neither shall touch another tenant or the global identity. *(FR-IDN-094, FR-IDN-120)*

### RF-8 · Administration authority

Who may change roles and grants is itself a row of the matrix. *Why:* an administration model that lets an Admin mint an Owner, or anyone raise their own role, is an escalation path. *(Decision: Q6.)*

- RF-8.1 An Owner shall be able to set any member's tenant role, to make another member an Owner, and to transfer ownership under the rules of specification 010. *(FR-IDN-097; decision Q6)*
- RF-8.2 An Admin shall be able to set the tenant role of a non-Owner membership to Member or Admin, and shall never be able to create an Owner, change an Owner's role or status, or evade the last-active-Owner protection. *(FR-IDN-091; decision Q6)*
- RF-8.3 Nobody shall be able to raise their own tenant role; an Admin shall not change their own tenant role at all; an Owner shall lower their own only through ownership transfer. *(FR-IDN-084; decision Q6)*
- RF-8.4 An Owner or Admin shall be able to set the project grants of any member of the tenant, including their own; a Member shall be able to change nobody's grants, their own included. A self-granted project role or `canRun` shall be visible on the surfaces of specification 011 and recorded as RF-11 states. *(FR-IDN-098; decision Q6)*
- RF-8.5 An invitation shall carry a tenant role of Admin or Member only — never Owner, which is obtained only through ownership transfer — and at most one project with explicit grants, all within what the inviter is permitted to set under RF-8.1–8.4; acceptance shall never yield more than the invitation carried, and a revoked membership re-invited shall receive a new membership with only the invited scope. *(FR-IDN-084; specification 008 Q4, Q5)*
- RF-8.6 Every change of a tenant role, a membership status or a grant shall be a single indivisible change that advances the version and writes the audit record together; a partial change shall never be observable. *(FR-IDN-099, FR-X-005)*
- RF-8.7 The last-active-Owner protection and the re-check of current membership at ownership-transfer confirmation shall apply to every path that changes an Owner's role or status, including an Admin's or an Owner's own action. *(FR-IDN-095; specification 010)*

### RF-9 · Grant records

Grants are explicit records, never inferred rights. *Why:* FR-IDN-096 — even the founding Owner's authority is a record somebody can read and revoke. *(Decision: Q2.)*

- RF-9.1 The system shall hold a project grant as one record per membership and project carrying the subset of Author, Reviewer and Viewer held and whether `canRun` is granted; no right shall be inferred from a tenant role, from creating a project, from inviting, or from a previous membership. *(FR-IDN-096, FR-IDN-098; decision Q2)*
- RF-9.2 The system shall accept any subset of Author, Reviewer and Viewer on one grant; Viewer beside Author or Reviewer shall add nothing and may remain recorded. *(decision Q2)*
- RF-9.3 The system shall accept `canRun` on a grant whether or not Author or Reviewer is held; while neither is held, `canRun` shall be inert for RF-2.8 and the surface of specification 011 shall say so beside it. *(FR-IDN-091; decision Q2)*
- RF-9.4 If a change leaves a grant with no project role and no `canRun`, then the grant shall cease to exist and the member shall no longer be a member of that project. *(decision Q2)*
- RF-9.5 When a workspace is created with a first project, the founding Owner's Author, Reviewer and `canRun` shall be explicit records ticked by the person and written in the same transaction as the tenant and the membership (specification 012); the Owner role itself shall grant none of them. *(FR-IDN-096, FR-IDN-114, FR-IDN-115)*
- RF-9.6 When a project is created later, its creator shall hold only the grants explicitly recorded for them; creation shall imply no grant. *(register §3 "new-project creator grants are explicit records")*
- RF-9.7 A grant shall never be created for a membership that is not `active`, and a revoked membership shall never regain a grant except through a new membership. *(decision Q3)*

### RF-10 · What the surfaces say

Every surface that shows a role or a grant repeats the same promises. *Why:* FR-IDN-092 requires the membership and policy surfaces to say that administration is not content access; consistency across surfaces is what makes the statement believable.

- RF-10.1 The members surface (specification 010) shall state "Membership is not project access" with the explanation that Owner and Admin administer membership but do not implicitly read every project, that Author, Reviewer or Viewer are granted per project with `canRun` independent, and that removing a membership revokes this tenant's access while authored evidence is retained under policy. *(FR-IDN-092, FR-SET-033; R13)*
- RF-10.2 The project grants surface (specification 011) shall state that Author, Reviewer and Viewer are project roles and `canRun` an independent permission, that current permission is checked at each protected action, that revocation blocks new work while existing evidence and authored provenance remain, and that being Owner or Admin does not implicitly grant access to project content, approval or Run; it shall show each member's tenant role beside their grants for contrast. *(FR-IDN-098, FR-SET-052; R17)*
- RF-10.3 The tenant policy surface (settings specification) shall state the authority boundaries — who can configure, that only identities with an explicit project grant read project content, that Admin does not imply Author, Reviewer, Viewer or `canRun` — and the three-scope summary: tenant membership (Owner / Admin / Member, no implicit project read), project content (Author / Reviewer / Viewer, `canRun` independent), global identity (self-service and verified recovery, no global customer Admin). *(FR-SET-044, FR-SET-046; N06)*
- RF-10.4 The chooser (specification 013) shall state that tenant administration does not imply project content access and that project access is checked separately after a tenant is chosen, and shall show each membership's tenant role, state and the person's own project grants. *(FR-IDN-110; N03)*
- RF-10.5 Workspace creation (specification 012) shall state that the Owner role grants none of Author, Reviewer or `canRun` automatically. *(FR-IDN-114; N04)*
- RF-10.6 The approval surface shall show the independence of Author, Reviewer and `canRun`, the separateness of the tenant role and the self-approval policy in force. *(FR-APR-006)*
- RF-10.7 Where an action is unavailable for lack of a grant, the surface shall hide it or disable it with the missing grant named in its label or explanation; it shall never fail silently or after the fact. *(FR-PRJ-012)*
- RF-10.8 Where `canRun` is held without Author or Reviewer, the grants surface shall state that `canRun` has no effect until a project role is held. *(decision Q2; S-47)*
- RF-10.9 The statements of RF-10.1–10.8 shall be identical for every role variant of a surface and at every viewport width. *(FR-X-055; S-06)*

### RF-11 · Versioning and audit

Every change to authority is recorded. *Why:* FR-IDN-099; authority that cannot be reconstructed after the fact cannot be reviewed.

- RF-11.1 The system shall append to the audit log every change of a tenant role, a membership status, a project grant or `canRun`, and every ownership transfer, with: the acting membership and identity, the affected membership and identity, the tenant, the project where applicable, the previous value, the new value, the resulting `permissionVersion`, the time and the correlation id. *(FR-IDN-099, FR-X-044)*
- RF-11.2 The system shall append every derivation that yields no authority for a requested protected action with the requested scope and the reason class (no membership, membership not active, tenant not active, missing grant, stale version, policy, enrollment gate), without disclosing the reason to the requester beyond the safe problem of specification 006. *(FR-X-044, FR-X-033)*
- RF-11.3 Audit entries shall carry no credential, no secret and no project content; their visibility shall follow the reader's current authority as RF-2.12 states. *(FR-X-030, FR-X-044)*
- RF-11.4 Every write of this feature shall confirm persistence before reporting success and shall be idempotent, so that a repeated administration request yields one change and the same answer. *(FR-X-020, FR-X-022)*

### RF-12 · What nobody can do

The matrix is also a list of absences. *Why:* each absence is an escalation path the product promises not to have.

- RF-12.1 No role, grant, invitation, policy, tenant action or administrator action shall let a person raise their own tenant role or project grants beyond what RF-8 permits. *(FR-IDN-084; decision Q6)*
- RF-12.2 No tenant role shall read, author, approve or run project content without an explicit grant; no project role or `canRun` shall administer the tenant. *(FR-IDN-091, FR-IDN-092)*
- RF-12.3 No grant shall survive the revocation of its membership, and no revoked grant shall return except through a new explicit record. *(FR-IDN-094; decision Q3)*
- RF-12.4 No write shall commit under a `permissionVersion` older than the current one. *(FR-X-005)*
- RF-12.5 No session, challenge, credential or browser-supplied identifier shall be a source of authority; no cached derivation shall outlive its request. *(FR-X-002, FR-IDN-024; P3)*
- RF-12.6 No platform operator authority shall exist inside a tenant, and no tenant authority shall reach platform configuration. *(FR-OPS-002)*
- RF-12.7 No tenant policy shall weaken a platform boundary; it shall only tighten (require MFA for Members, require a different approver). *(FR-SET-047)*
- RF-12.8 No machine, AI path or scheduled job shall hold a tenant role or a project grant in this release; approval and launch are human decisions under RF-2. *(P2; register deferral of machine-initiated execution)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Exactness.** The matrix of RF-2 is implemented exactly; no action is permitted that it does not list, and none it lists is withheld from a membership that satisfies it. | FR-IDN-091 |
| RNF-2 | **Server-derived authority.** Every protected action is derived on the server from current facts at every effect boundary; nothing from the client, the session or a cache substitutes. | FR-X-001…004; P3 |
| RNF-3 | **No implicit access.** Tenant administration never implies project content, approval or launch; project roles never imply tenant administration. | FR-IDN-092 |
| RNF-4 | **Fail closed.** An absent, suspended, revoked, deleting, stale or unknown fact yields no authority. | P6 |
| RNF-5 | **Atomic change.** A role, status or grant change, its version advance and its audit record commit together or not at all. | FR-IDN-099, FR-X-005 |
| RNF-6 | **Monotonic version.** `permissionVersion` only advances; a stale version never commits a sensitive write. | FR-X-005; decision Q4 |
| RNF-7 | **Revocation latency.** From the moment a change commits, the next request of the affected membership is derived under the new facts. | FR-IDN-094 |
| RNF-8 | **No fabricated outcome.** Revocation never marks work failed, never rewrites a verdict, never claims an effect undone. | FR-IDN-094, FR-IDN-067; P8 |
| RNF-9 | **Closed model.** No custom role, hierarchy, team, operator-in-tenant or machine principal exists; the role sets are exactly those of RF-1. | FR-IDN-090, FR-OPS-002 |
| RNF-10 | **Verification.** Concurrent last-Owner and grant transitions, authorisation races, stale-version writes, revoked grants during jobs and before provider submission, and real cross-tenant RLS attacks are verified as areas A and B require. | requirements A.12, B.11; P9 |
| RNF-11 | **Configured limits.** Any bound this feature needs (checkpoint cadence for bounded cancellation, audit retention) is validated configuration with no default. | FR-X-046; P13 |
| RNF-12 | **Accessibility and language.** The statements of RF-10 are text, present at every width, in EN / ES / FR; the interface language never changes an authority. | FR-X-040, FR-X-043, FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Owner with no grant opens a project of their tenant | Denied scope; no content disclosed; the members and policy surfaces explain why (RF-2.4, RF-3.8). |
| LC-2 | Admin tries to change an Owner's role or status | Refused; the action is not offered (RF-8.2, RF-10.7). |
| LC-3 | Admin tries to change their own tenant role | Refused; not offered (RF-8.3). |
| LC-4 | Owner grants themselves Reviewer and `canRun` on a project | Permitted; visible on the grants surface; audited with actor = affected (RF-8.4, RF-11.1). |
| LC-5 | Member tries to edit their own grants | Refused; not offered (RF-8.4). |
| LC-6 | `canRun` granted to a Viewer-only member | Stored; inert; surface says so; launch refused (RF-2.8, RF-9.3, RF-10.8). |
| LC-7 | Author later loses Author but keeps `canRun` | `canRun` becomes inert; launch refused from the next request (RF-2.8, RF-7.1). |
| LC-8 | Viewer added to an Author | Recorded; no change of authority (RF-9.2). |
| LC-9 | All roles and `canRun` removed from a grant | Grant ceases to exist; the member leaves the project (RF-9.4). |
| LC-10 | Grant narrowed while the member edits a scenario | Next sensitive write refused as stale; one permission refresh; denied-scope state; draft shown only if still permitted (RF-6.3, RF-6.4, RF-7.6). |
| LC-11 | Membership revoked while the member has a queued and a running Run | Queued ticket refused at its checkpoint as NOT EXECUTED; running attempt gets a bounded cancellation request; facts kept; verdict untouched (RF-7.3, RF-7.4). |
| LC-12 | Membership suspended then reinstated | No authority while suspended; grants unchanged on return; two version advances; two audit entries (RF-4.3, RF-4.5). |
| LC-13 | Revoked member re-invited | The same membership record returns to active with the invited role and only the new invitation's grants; the old grants stay history (RF-4.4, RF-9.7). |
| LC-14 | Tenant suspended | No customer authority for anyone; memberships and grants untouched; subscriptions refreshed; accepted runs not failed (RF-5.2, RF-5.3). |
| LC-15 | Tenant returns to active | Each membership derives exactly its own status and grants (RF-5.4). |
| LC-16 | Last active Owner targeted by removal, demotion or suspension | Refused by the protection of specification 010 on every path (RF-8.7). |
| LC-17 | Policy `requireDifferentApprover` tightened while a Reviewer reviews their own revision | Decision refused at the moment of approval under the current policy revision; no membership version changes (RF-2.14, RF-6.5). |
| LC-18 | Two administrators change the same grant at once | One change commits first; the second is applied or refused under the advanced version; both audited in order (RF-8.6, RF-6.6). |
| LC-19 | An inviter tries to issue an invitation carrying the Owner role, or an Admin one carrying more than Member or Admin | Refused; an invitation never carries Owner and never more than the inviter may set (RF-8.5; specification 008 Q4). |
| LC-20 | Founding Owner creates the workspace with a first project | Membership, Owner role and the ticked grants are one transaction; the Owner role alone grants nothing (RF-9.5). |
| LC-21 | Reviewer-only Owner archives a project | Permitted: Owner plus a project role grant (RF-2.9). |
| LC-22 | Member holding Author, Reviewer and `canRun` tries to archive the project | Refused: whole-project lifecycle needs Owner or Admin (RF-2.9). |
| LC-23 | Admin without a grant reads a project's audit trail | Refused; tenant role alone never reads project audit (RF-2.12). |
| LC-24 | Reviewer alone publishes trust material | Refused; a tenant Owner or Admin role is also required (RF-2.11). |
| LC-25 | Admin exports the organisation | Only projects the Admin holds a grant on are included; selection of others refused (RF-2.13). |
| LC-26 | Unenrolled Owner enters the tenant | No authority until enrollment completes, whatever the grants (RF-3.5; specification 003). |
| LC-27 | Request with a tenant id the identity is not a member of | Denied safely; existence undisclosed; audited (RF-3.2, RF-11.2). |
| LC-28 | Route names project A, body names project B | Denied safely (RF-3.2). |
| LC-29 | Identity becomes restricted or under erasure | Every membership's derivation yields nothing; statuses and grants untouched until the erasure workflow acts (RF-3.6; specification 015). |
| LC-30 | Project deleted while a member holds grants on it | Grants end with the project; audited; the member's other grants untouched (RF-1.4, RF-11.1). |

---

## 6. Out of scope

- **The members list, role editing, invitations list and ownership transfer** — F-IDN-12, specification 010; the last-Owner protection is theirs, referenced here by RF-8.7.
- **The project grants editor** — F-IDN-13, specification 011; this feature defines the records it writes and the statements it carries.
- **Invitation issue and acceptance** — F-IDN-10, specification 008.
- **Workspace creation and the founding Owner's bootstrap transaction** — F-IDN-14, specification 012; this feature defines what the bootstrap grants mean.
- **The tenant chooser and the suspended-tenant screen** — F-IDN-15 and F-IDN-16, specifications 013 and 014.
- **Editing the tenant policy** (`requireDifferentApprover`, Member MFA) — F-SET-05; this feature checks the policy revision and defines its effect on eligibility.
- **The approval decision and the launch command themselves** — the approval and execution specifications; this feature defines who is eligible.
- **The audit trail surface** — F-X-05; this feature defines what is appended and who may read it.
- **Leaving a tenant and identity erasure** — F-IDN-17, specification 015.
- **Custom roles, teams, hierarchies, delegated administration beyond Admin, SSO or SCIM provisioning, machine principals with grants** — product boundary (FR-IDN-090, FR-SET-043, FR-EXT-041 deferred).
- **Platform operator tooling** — FR-OPS; outside every tenant by construction.

---

## 7. Criteria of completion

Roles, grants and the authority matrix are complete when all of the following hold:

1. Every user story US-1 … US-11 can be walked end to end through the surfaces of specifications 010, 011, 012 and 013 with no step outside this specification, and US-12 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-12 has at least one verification that fails before and passes after the feature exists.
3. Matrix tests show, for every tenant role × every grant combination × every protected action class of RF-2, exactly the outcome RF-2 states — including inert `canRun`, Reviewer-only Owner lifecycle, Member lifecycle refusal, trust material's two authorities, audit visibility and export scope (RF-2, RNF-1, RNF-3, LC-1, LC-6, LC-21 … LC-25).
4. Derivation tests show that authority is derived from the facts of RF-3.1 only, rechecked at every effect boundary and in every job, denied safely for requested scopes the identity does not hold, and withheld under the enrollment gate (RF-3, RNF-2, RNF-4, LC-26 … LC-28).
5. Status tests show the meaning of active, suspended and revoked memberships and of active, suspended and deleting tenants, including reinstatement with unchanged grants and re-invitation with new ones (RF-4, RF-5, LC-12 … LC-15).
6. Version tests show one version per membership advanced by every role, status or grant change, a stale sensitive write refused transactionally, and a policy change advancing only the policy revision (RF-6, RNF-5, RNF-6, LC-10, LC-17, LC-18).
7. Revocation tests show denial from the next request, closed subscriptions, queued tickets refused as NOT EXECUTED, running attempts requested to stop at a bounded checkpoint with facts and verdict intact, and authored content retained (RF-7, RNF-7, RNF-8, LC-7, LC-11).
8. Administration tests show the Owner and Admin limits, the self-change rules, invitation bounds and the atomicity of every change (RF-8, LC-2 … LC-5, LC-16, LC-19).
9. Grant-record tests show explicit records only, any subset accepted, inert `canRun`, record removal at empty, bootstrap and creator rules (RF-9, LC-6, LC-8, LC-9, LC-20).
10. The statements of RF-10 are present verbatim on every role variant of the members, grants, policy, chooser, creation and approval surfaces, at desktop, tablet and mobile widths, in EN, ES and FR, and unavailable actions name the missing grant (RF-10, RNF-12).
11. Audit entries exist for every change and every refused derivation with the fields of RF-11, contain no secret or content, and are readable only under current authority (RF-11).
12. The concurrency and isolation verifications of areas A and B — last-Owner races, grant races, stale-version writes, revoked grants during jobs, cross-tenant RLS attacks — pass on real infrastructure (RNF-10).
13. The register items this specification relies on — A-53, A-54, A-55, A-56, A-57, A-58 (decided here), A-36, A-39, A-42 and A-11 in the architecture register; S-47, S-48, S-49, S-06 and S-14 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
