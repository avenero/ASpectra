# Specification 012 — Workspace creation and bootstrap

**Feature** Creating a tenant workspace as its founding Owner, with an optional first project and explicit grants, in one auditable bootstrap · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` v1.8 (feature F-IDN-14 as the subject; F-IDN-01, F-IDN-02, F-IDN-03, F-IDN-04, F-IDN-08, F-IDN-09, F-IDN-11, F-IDN-12, F-IDN-13, F-IDN-15, F-IDN-18, F-PRJ-02, F-PRJ-12, F-SET-05 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P4, P6, P12, P13, P14) → `requirements.md` (FR-IDN-012, 021, 023, 026, 067, 090, 091, 092, 093, 096, 099, 110, 113, 114, 115, 145; FR-PRJ-011, 070, 114, 115, 116; FR-SET-040, 041, 043, 047; FR-APR-021; FR-X-001, 002, 004, 005, 020, 021, 022, 030, 033, 034, 040, 042, 043, 044, 046, 053, 055) → the design of record, screen N04 CreateWorkspacePage `161:49374`, the D02 atlas `159:32045` entry N22 CreateProjectDialog `159:32052` ("Create a project"), the entry "Create workspace" on N03 TenantChooserPage `161:49326` and R01 LandingPage `148:341`, the readiness checklist R09 OnboardingPage `148:1193` as the destination, the D03 journey `148:4` ("N04 create workspace → R39 TOTP setup / R36 recovery acknowledgement → R09 readiness → N22 create project"), register `122:2` §3 and the Authoring board `79:4061`.

**Related specifications:** 001 Login (the landing rule that routes an identity with no membership to workspace creation, RF-8.2, decision Q4) · 002 Signup and email verification (the verified email every founder needs; "Create workspace" on the landing goes through signup first) · 003 TOTP enrollment (the enrollment forced before any workspace field, RF-1.2; the return to the creation entry after activation, RF-5.3, and from "Leave setup", decision Q20; LC-22) · 006 Sessions, refresh and expiry (the session the founder holds; the denied-scope handling if authority cannot be derived) · 007 Logout and session management (the sign-out that "Back to sign in" performs for a founder with no membership) · 009 Roles, grants and the authority matrix (what the bootstrap records mean: the Owner role, the explicit grant records with any subset of roles and an inert `canRun`, the policy revision; RF-9.5, RF-9.6, RF-10.5, decisions Q2 and Q7) · 010 Tenant members and ownership transfer and 011 Project grants editor (where the founder later changes what the bootstrap created) · 013 Tenant chooser (the "Create workspace" entry and the new membership's row) · 016 Guarded tenant and project switch (how the founder enters the new tenant) · the project specification of F-PRJ-02 (every later project creation, including N22 from R09) · the overview specification of F-PRJ-12 (the readiness checklist R09 the founder lands on) · the settings specification of F-SET-05 (the policy record the bootstrap creates at its first revision).

This document states **what** workspace creation does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

A **workspace** is a tenant: one customer organisation, holding its own memberships, projects, environments and evidence, isolated from every other. Nobody is born into a tenant. A person creates one, and in doing so becomes its **founding Owner** — the first and only membership of a new tenant, holding the Owner role, with no implicit access to any project, exactly as specification 009 states.

Creation asks for the least the product needs: a verified email, a protected Owner identity, and a workspace name. No address, no phone number, no connector. Because Owner authority is mandatory TOTP territory (FR-IDN-023), enrollment precedes creation: a founder who has no factor is taken through the enrollment wizard of specification 003 before any workspace field is accepted, and creation is **blocked, never bypassed**, until the factor is active and the recovery codes acknowledged. The designed page reads the journey back to the person — "1 · Email verified", "2 · Owner protected", "3 · Workspace".

Optionally the founder creates a **first project** in the same act, and ticks — explicitly, box by box — the Author, Reviewer and `canRun` grants they want on it. The page says why: "The workspace Owner role does not grant these automatically." Everything the act creates — the tenant, the Owner membership, the tenant's first policy revision, the project and each grant record — is written as **one auditable transaction**: either the whole workspace exists, or nothing does. Creating a workspace contacts no target, starts no run, and sends no message; it ends on the readiness checklist, where the founder sees what remains to be done.

### 1.2 Objective

Let a verified, protected identity create a workspace it owns, with an honest first project if it wants one — while:

- never creating or entering a tenant for an identity whose email is unverified or whose Owner factor is not enrolled;
- never inferring a right from the Owner role: every grant on the first project is a record the founder ticked and can later read and change;
- never leaving a half-created workspace: tenant, membership, policy, project and grants exist together or not at all;
- never touching a customer target, starting a run or granting anything beyond the new tenant;
- keeping every statement the page draws about identity, scope and authority.

### 1.3 Why it matters

The first minute decides whether the product's model is understood. A founder who gets Author, Reviewer and `canRun` silently will believe Owner means everything, and will later be surprised when an invited Admin cannot read a project. A founder who can create a workspace without a factor is an Owner a phishing message away from losing a company's test estate. And a creation that writes the tenant but fails on the project leaves a workspace nobody can clean up. Creation must be short, strict and atomic, and it must say what it does.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Decisions that depend on an architecture or design decision are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row (architecture `122:2` §14 as A-nn, D03 Screen decisions ledger `148:4` as S-nn).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | The three entries into N04 — the landing's "Create workspace" after signup and sign-in, the landing rule of specification 001 for an identity with no membership, and "Create workspace" on the tenant chooser; the preconditions; the form; the first project and its grants; the single bootstrap transaction; what follows creation; and the statements N04 carries. Later project creation (N22 from R14 or R09) belongs to F-PRJ-02; the readiness checklist content to F-PRJ-12; enrollment to 003; the chooser to 013; policy editing to F-SET-05. |
| Q2 | Who may create a workspace, and how many | Any identity whose email is verified and whose status permits sign-in, whether or not it already belongs to other tenants; each creation makes it the founding Owner of a new, independent tenant. There is no fixed maximum per identity in this release; creation is counted by the configured rate limits for authenticated work, so a flood is throttled without a cap that strands a legitimate founder. Enrollment is checked at entry and again at submission. Register A-59. |
| Q3 | Rules for the workspace and project names | A display label: required, trimmed, within a configured length, any printable text; not unique — the tenant's identity is its own, two founders may both call their workspace "Northstar Labs", and the chooser lists each membership with its role and state. The project name follows the same rule within the tenant. No reserved words, no derived identifier. Register A-61. |
| Q4 | What the first-project grant boxes may hold | Author, Reviewer and `canRun` start unticked and each must be ticked deliberately. Any subset is accepted, including none: the project is created and the founder holds exactly what was ticked. When fewer than three are ticked, an inline warning names what the founder will not be able to do in the project — author, approve, run — and that grants can be changed later from the project members surface. `canRun` without Author or Reviewer is accepted and stated inert (specification 009 Q2). D03 ledger S-53 (D02 `421:3038`). |
| Q5 | What a successful creation does and where it goes | One transaction creates: the tenant (active); the founder's Owner membership (active, at its first permission version); the tenant policy record at its first revision with the platform-mandated values — self-approval allowed, MFA for Members not required, retention and limits by reference to the configured platform policy, no number invented; the optional project and one explicit grant record holding the ticked grants; and the audit entries. The founder then enters the new tenant through the guarded switch with authority derived afresh — the enrolled Owner passes the gate — and lands on the readiness checklist with the first project selected when one was created. No run, no target contact, no message. Register A-60. |
| Q6 | Leaving N04; what survives an interruption | "← Back to sign in" ends the session and returns to the sign-in screen for a founder who holds no membership; for a person who came from the chooser it returns to the chooser with the session intact, under the label "Back to your workspaces" (D02 `421:3038`). Nothing typed on N04 is persisted anywhere: a leave, a reload, a session end or the enrollment detour starts the form empty. An interrupted creation leaves no tenant, membership, policy, project or grant. A double submit carries one idempotency key and yields exactly one tenant. D03 ledger S-54 (D02 `421:3038`). |

Two rules below are derived from these decisions rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** A founder who already holds an active factor is never asked to enroll again (specification 003 RF-1.6): step "2 · Owner protected" shows as complete and the form opens directly.
- **D-b** If the bootstrap cannot complete for a reason other than a refused precondition, the person sees one generic failure stating that nothing was created and may submit again; the same idempotency key then either returns the recorded outcome or performs the creation once.

Register items this specification relies on: **A-59**, **A-60**, **A-61**, **S-53** and **S-54** (decided here), and the pre-existing **A-11** (configuration values: the name length bound and the rate-limit family), **A-27** (identity statuses), **A-36** (identity-scoped membership listing, which the chooser entry and the landing rule use), **S-04** (N22's destination from R09 belongs to F-PRJ-02), **S-14** (no message is sent by this feature), **S-17** ("Help" on N04 opens N43 `430:4129`) and **S-34** (003's "Leave setup" from a forced entry returns to this feature's entry).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | person who just verified my email | to create my workspace with nothing more than a name | I reach a protected workspace without filling a company profile |
| US-2 | founder without an authenticator | to be taken through enrollment first, and to be told plainly that creation never bypasses it | my Owner identity is protected from the first day |
| US-3 | founder who already enrolled | to go straight to the form | I am not asked twice for what I already proved |
| US-4 | founder | to create my first project in the same act and tick exactly the grants I want on it | I start authoring immediately, with rights I chose and can read |
| US-5 | founder who ticks fewer than three grants | to be told what I will not be able to do, and where to change it later | I am not surprised by a project I cannot approve or run in |
| US-6 | founder | the workspace to exist completely or not at all | a failure never leaves me an Owner of a broken tenant |
| US-7 | founder | to land on a checklist that shows what is done and what is next | I know the product is not going to run anything by itself |
| US-8 | member of other workspaces | to create a new workspace from my chooser and come back to it if I change my mind | one identity can found several organisations |
| US-9 | founder leaving the page | my unfinished form to vanish and nothing to be created | I never leave a half-made workspace behind |
| US-10 | person on a phone or tablet | the creation page reflowed with every statement, field and gate intact | I can found a workspace anywhere |
| US-11 | attacker or over-reaching party (negative story) | — | I cannot create a tenant with an unverified address, without a factor, with a connector or a target contact, with inferred grants, with a second Owner, twice from one submission, or under a name that reveals another tenant's existence |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *founder* means the signed-in identity creating the workspace; *the page* means N04; *the bootstrap* means the single transaction of RF-4; *enrolled* means an active TOTP factor with acknowledged recovery codes (specification 003).

### RF-1 · Entries and preconditions

Creation is reachable from three places and open only to a verified, protected identity. *Why:* FR-IDN-012, FR-IDN-023 and FR-IDN-113 — the tenant must never exist before its Owner is both verified and protected. *(Decision: Q1, Q2.)*

- RF-1.1 The system shall reach the page from: "Create workspace" on the public landing, which routes through signup and sign-in first and never creates or enters a tenant directly; the landing rule of specification 001 when the signed-in identity holds no membership; and "Create workspace" on the tenant chooser for a signed-in identity that already holds memberships. *(FR-IDN-145, FR-IDN-110; specification 001 RF-8.2; R01, N03; decision Q1)*
- RF-1.2 The system shall require a complete identity session (specification 006) to open the page; the page shall never be reachable by a limited identity state, an unverified identity or a session-less visitor. *(FR-IDN-012, FR-IDN-024; specification 001 RF-5.2)*
- RF-1.3 If the founder's email is not verified, then the system shall not open the page and shall route to the verification pending state of specification 002. *(FR-IDN-012)*
- RF-1.4 If the founder's status permits no sign-in (restricted or under erasure), then the system shall not open the page. *(A-27)*
- RF-1.5 When a founder with no enrolled factor reaches the entry, the system shall route into the enrollment wizard of specification 003 before any workspace field is shown or accepted, with the return destination set to this entry; when activation completes, the founder returns to the page; when "Leave setup" is chosen, the founder returns to the entry with nothing created. *(FR-IDN-113, FR-IDN-023, FR-IDN-026; specification 003 RF-1.2, RF-5.3, Q20; S-34)*
- RF-1.6 When a founder who already holds an enrolled factor reaches the entry, the system shall open the form directly with step "2 · Owner protected" shown complete; it shall never ask for a second enrollment. *(specification 003 RF-1.6; D-a)*
- RF-1.7 The system shall allow any eligible identity to create any number of workspaces, each as the founding Owner of a new, independent tenant; it shall impose no fixed maximum and shall count creation under the configured rate limits for authenticated work. *(FR-IDN-021; decision Q2; A-59)*
- RF-1.8 The system shall re-check every precondition of RF-1.2 … RF-1.5 at submission, not only at entry; if any fails at submission, then nothing shall be created and the founder shall be routed as the failing precondition states. *(FR-IDN-113; P6; decision Q2)*

### RF-2 · The page

The page asks for the least and says the most. *Why:* FR-IDN-113 forbids an address, a phone number or a connector, and the design carries the promises of the product on the page.

- RF-2.1 The system shall present the page with the marker "LOCAL IDENTITY / HUMAN CONTROL", the heading "A clear path. A protected workspace.", the subtitle "Your identity is global. Your access is scoped to each workspace and project.", the journey "01 Verified email · 02 Local account · 03 Required TOTP for Owner / Admin", the step indicator "1 · Email verified · 2 · Owner protected · 3 · Workspace" with the current step named in text, and the headings "Make room for your first intent." and "Create a workspace with only the details it needs." *(FR-IDN-113, FR-X-042; N04)*
- RF-2.2 The system shall present the "Workspace identity" block read-only: "Verified Owner" with the founder's display name and verified email, and "Identity protection" with "TOTP enrolled · recovery codes acknowledged" once enrollment is complete; the founder shall not be able to change either on this page. *(FR-IDN-113, FR-IDN-027; N04)*
- RF-2.3 The system shall present a single required field "Workspace name" and the statement "Your identity becomes a Workspace Owner. No address, phone number or connector is required."; no other identifying, contact or integration field shall exist. *(FR-IDN-113; N04; decision Q3)*
- RF-2.4 The system shall validate the workspace name as a trimmed, non-empty label within the configured length bound; a name shall not be required to be unique anywhere; the length bound shall be validated configuration with no default. *(FR-X-046; P13; decision Q3; A-61)*
- RF-2.5 The system shall present the block "First project · optional" with the unticked toggle "Create my first project with this workspace"; while it is unticked, the project name and the grant boxes shall be absent or disabled and no project shall be created. *(FR-IDN-114; N04)*
- RF-2.6 The system shall present the block "No connection required" with the statement "Start with manual authoring. Set a project environment before compilation and Run; creating this workspace does not test a target." *(FR-PRJ-070, FR-PRJ-115; N04)*
- RF-2.7 The system shall present the single action "Create workspace and explicit grants" and, beneath it, the statement "If TOTP enrollment is incomplete, finish enrollment first. Workspace creation never bypasses mandatory Owner MFA." *(FR-IDN-113; N04)*
- RF-2.8 The system shall present "← Back to sign in" for a founder who holds no membership and "Back to your workspaces" for a founder who came from the chooser (RF-6.1, RF-6.2), the routes Privacy · Terms · Help, and the language switch EN · ES · FR. *(FR-X-043; N04; decision Q6; S-54; S-14, S-17)*
- RF-2.9 When the founder submits, the system shall show a creation-in-progress state, disable resubmission until the answer arrives, and never show success before the bootstrap has committed. *(FR-SET-001 by analogy; D02 "N04 / Field, progress, failure" `421:3038`)*

### RF-3 · The first project and its grants

A first project is optional, and every right on it is a tick. *Why:* FR-IDN-096 and FR-IDN-114 — the founder receives Author, Reviewer and `canRun` only as explicit records; the Owner role grants none of them. *(Decision: Q4.)*

- RF-3.1 When the toggle of RF-2.5 is ticked, the system shall present a required "Project name" field under the same rule as RF-2.4, applied within the new tenant. *(FR-IDN-114; N04; decision Q3)*
- RF-3.2 The system shall present, under "Grant project access explicitly to <founder's name>. The workspace Owner role does not grant these automatically.", three boxes — "Author", "Reviewer", "canRun · separate grant" — all unticked by default, with the statement "Run still requires valid approval and environment pins." *(FR-IDN-096, FR-IDN-114; N04; specification 009 RF-10.5)*
- RF-3.3 The system shall accept any subset of the three boxes, including none; the project shall be created and the founder shall hold exactly the ticked grants as one explicit grant record of specification 009; when nothing is ticked, no grant record exists (specification 009 RF-9.4) and the founder reaches the project later through the grants editor (specification 011). *(FR-IDN-096, FR-IDN-114; specification 009 RF-9.4, RF-9.5; decision Q4)*
- RF-3.4 While fewer than three boxes are ticked, the system shall show an inline warning naming what the founder will not be able to do in the project — author without Author, approve without Reviewer, run without `canRun` — and that grants can be changed later from the project members surface (specification 011); the warning shall never block submission. *(FR-IDN-114; decision Q4; S-53)*
- RF-3.5 Where `canRun` is ticked without Author or Reviewer, the system shall accept it and state beside it that `canRun` has no effect until a project role is held. *(specification 009 Q2, RF-9.3)*
- RF-3.6 The system shall never pre-tick a box, infer a grant from the Owner role, or create a grant that was not ticked. *(FR-IDN-096; specification 009 RF-9.1)*
- RF-3.7 The system shall create at most one project in the bootstrap; further projects are created later under F-PRJ-02 by an Owner or Admin with explicit initial grants. *(FR-PRJ-011; decision Q1)*

### RF-4 · The bootstrap

One act, one transaction. *Why:* FR-IDN-115 — a founder must never own a tenant that exists only in part. *(Decision: Q5.)*

- RF-4.1 When the action of RF-2.7 is activated and every precondition holds, the system shall, as one indivisible change: create the tenant in status `active` with the given name; create the founder's membership in that tenant with the Owner role, status `active` and its first permission version; create the tenant policy record at its first revision (RF-4.2); where the toggle was ticked, create the project with its name and, when at least one box is ticked, one explicit grant record for the founder holding the ticked grants; and append the audit entries of RF-8.1. *(FR-IDN-096, FR-IDN-115, FR-IDN-093, FR-IDN-099; specification 009 RF-4.1, RF-9.5; decision Q5; A-60)*
- RF-4.2 The first policy revision shall hold the platform-mandated values: self-approval allowed, as FR-APR-021 defaults it; MFA for Members not required; retention and limits by reference to the configured platform policy with no number invented; no SSO, SCIM or custom-role control. The founder shall be able to tighten it later under F-SET-05. *(FR-SET-040, FR-SET-041, FR-SET-043, FR-SET-047; decision Q5)*
- RF-4.3 If any part of RF-4.1 cannot be written, then nothing of it shall exist afterwards — no tenant, no membership, no policy, no project, no grant, no audit entry of a completed creation — and the founder shall see the failure of RF-6.4. *(FR-IDN-115; P6; D-b)*
- RF-4.4 Every submission shall carry an idempotency key; a repeated submission with the same key and body shall return the recorded outcome and create nothing further; the same key with a different body shall be refused as a conflict; a lost acknowledgement shall lead to a lookup, never to a second tenant. *(FR-X-020, FR-X-022; decision Q6)*
- RF-4.5 The bootstrap shall create exactly one membership — the founder's — and exactly one Owner; it shall create no invitation, no other membership, no environment, no source connection, no delivery profile, no scenario and no run intent. *(FR-IDN-096, FR-IDN-115; decision Q5)*
- RF-4.6 The bootstrap shall contact no customer target, start no run, send no message and enqueue nothing but the audit entries and the records of RF-4.1. *(FR-PRJ-070, FR-PRJ-115, FR-IDN-067; decision Q5; S-14)*

### RF-5 · After creation

The founder enters the new workspace as its enrolled Owner and sees what is next. *Why:* the D03 journey ends on the readiness checklist, and the enrollment the founder completed is what lets Owner authority be derived at once. *(Decision: Q5.)*

- RF-5.1 When the bootstrap commits, the system shall enter the new tenant through the guarded tenant switch of specification 016, with authority derived afresh for the new membership; the enrolled Owner shall pass the enrollment gate of specification 003 and hold Owner authority from the first request in the tenant. *(FR-IDN-130, FR-IDN-026, FR-X-001; specification 009 RF-3.1; decision Q5)*
- RF-5.2 When the tenant is entered, the system shall land on the readiness checklist of F-PRJ-12 with the first project selected when one was created, and with "Project" as the next step otherwise; the checklist content itself is that feature's. *(FR-PRJ-114, FR-PRJ-116; R09; decision Q5)*
- RF-5.3 The new membership shall appear in the founder's chooser with the role Owner and the state active from the next listing on. *(FR-IDN-110; specification 013)*
- RF-5.4 Creation shall not change the founder's session, password, factor, recovery codes or other memberships, and shall not replay any request that was pending before it. *(FR-IDN-067; P1)*

### RF-6 · Leaving and failing

Nothing typed survives, and nothing half-made exists. *Why:* the page holds a name and three ticks; retaining them buys nothing and a partial tenant costs everything. *(Decision: Q6.)*

- RF-6.1 When "← Back to sign in" is activated by a founder who holds no membership, the system shall end the session as specification 007 states and return to the sign-in screen; nothing shall be created. *(FR-IDN-068; specification 007; decision Q6)*
- RF-6.2 When "Back to your workspaces" is activated by a founder who came from the chooser, the system shall return to the chooser with the session intact; nothing shall be created. *(FR-IDN-110; decision Q6; S-54)*
- RF-6.3 The system shall never persist the workspace name, the project name or the ticks anywhere: a leave, a reload, a session end or the enrollment detour of RF-1.5 shall start the form empty. *(FR-X-034; decision Q6)*
- RF-6.4 If the bootstrap fails for a reason other than a refused precondition, then the system shall show one generic failure stating that nothing was created, keep the founder on the page, and allow resubmission under RF-4.4. *(FR-X-033; D-b)*
- RF-6.5 If a precondition fails at submission, then the system shall route as RF-1.3 … RF-1.5 state and shall show no partial success. *(decision Q2)*
- RF-6.6 If the session ends during creation, then the founder shall meet the expiry handling of specification 006; after re-authentication the submission shall be looked up by its idempotency key before the page offers to submit again. *(FR-X-022; specification 006 RF-9.5)*

### RF-7 · What creation never does

- RF-7.1 The system shall never require or accept an address, a phone number, a connector, a payment method or a target configuration to create a workspace. *(FR-IDN-113)*
- RF-7.2 The system shall never create or enter a tenant from the public landing directly, from an unverified identity, from a limited identity state, or from a founder whose enrollment is incomplete. *(FR-IDN-145, FR-IDN-012, FR-IDN-024, FR-IDN-113)*
- RF-7.3 The system shall never grant the founder Author, Reviewer, Viewer or `canRun` on any project without a ticked record, and shall never create a second Owner, a membership for anyone else or an invitation as part of creation. *(FR-IDN-096, FR-IDN-092; specification 009 RF-9.1)*
- RF-7.4 The system shall never disclose, through the name or any answer, whether another tenant or identity exists. *(FR-IDN-020 by analogy; decision Q3)*
- RF-7.5 The system shall never start a run, test a target, contact an integration or send a message as a consequence of creation. *(FR-PRJ-070, FR-PRJ-115; decision Q5)*

### RF-8 · Audit and secrecy

- RF-8.1 The system shall append to the audit log, in the bootstrap transaction: the tenant created; the founding membership created with the Owner role and its first permission version; the first policy revision created; where applicable, the project created and, when at least one box was ticked, the grant record created with the ticked grants; and, for each refused or failed attempt, the attempt with its reason class. Each entry identifies the founder, the time, the new tenant and the correlation id. *(FR-IDN-099, FR-IDN-115, FR-X-044)*
- RF-8.2 No audit entry, log line, problem detail or URL shall contain a credential, a code or any secret; the page shall write nothing to browser storage. *(FR-X-030, FR-X-033, FR-X-034)*
- RF-8.3 Every refusal shall be a safe problem with a correlation id that names the failing field or the generic failure, never another tenant or identity. *(FR-X-033)*

### RF-9 · Responsive and accessible surfaces

- RF-9.1 Where no designed mobile or tablet counterpart exists for the page, the system shall present it reflowed under the desktop rules, preserving every statement, field, action, route and gate. *(FR-X-055; S-02)*
- RF-9.2 Every interactive element shall have a touch target of at least 44 px, a visible label and visible focus; the step indicator, field errors and the grant warning shall be conveyed by text and programmatically associated with their field or region; announcements shall never steal focus from a field. *(FR-X-040, FR-X-042, FR-X-053)*
- RF-9.3 The page and its messages shall exist in EN / ES / FR; the interface language shall never change what is created. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Atomicity.** Tenant, Owner membership, first policy revision, optional project and grant record, and audit entries commit together or not at all. | FR-IDN-115 |
| RNF-2 | **Protected Owner.** No tenant exists whose founding Owner is unverified or unenrolled; the check holds at entry and at submission. | FR-IDN-012, FR-IDN-023, FR-IDN-113 |
| RNF-3 | **Explicit records.** Every grant the founder holds after creation is a ticked record; nothing is inferred from the Owner role. | FR-IDN-096 |
| RNF-4 | **Idempotent creation.** One submission, however repeated, yields one tenant. | FR-X-020, FR-X-022 |
| RNF-5 | **Minimal data.** Only a verified identity and a name are needed; no contact, integration or target data is collected. | FR-IDN-113 |
| RNF-6 | **No side effect.** Creation contacts no target, starts no run, sends no message, invites nobody. | FR-PRJ-070, FR-PRJ-115 |
| RNF-7 | **Configured limits.** The name length bound and the rate limits that count creation are validated configuration with no default. | FR-X-046; P13 |
| RNF-8 | **Fail closed.** A failed precondition or a failed write creates nothing. | P6 |
| RNF-9 | **No browser persistence.** Nothing typed on the page is stored anywhere. | FR-X-034 |
| RNF-10 | **Isolation.** The new tenant is a tenant like any other from its first row: composite ownership, row-level isolation, derived authority. | P12; FR-X-001 |
| RNF-11 | **Accessibility.** The page meets WCAG 2.2 AA with automated and manual assistive evidence. | FR-X-040, FR-X-042 |
| RNF-12 | **Language and reflow.** EN / ES / FR, display only; reflows preserve every statement, field, action and gate. | FR-X-043, FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | "Create workspace" on the landing by a visitor | Signup, verification and sign-in first; the landing rule then routes to the page; no tenant is created on the way (RF-1.1, RF-7.2). |
| LC-2 | Signed-in identity with no membership and no factor | Enrollment wizard before any field; activation returns to the page; "Leave setup" returns to the entry with nothing created (RF-1.5). |
| LC-3 | Founder closes the wizard during the enrollment detour | No workspace; the next attempt starts at enrollment again (RF-1.5; specification 003 LC-22). |
| LC-4 | Founder already enrolled | Form opens directly; step 2 shown complete (RF-1.6). |
| LC-5 | Email unverified identity reaches the entry | Verification pending state; page not opened (RF-1.3). |
| LC-6 | Member of two tenants uses "Create workspace" on the chooser | Page opens; a third tenant is created with its own Owner membership; the other memberships are untouched (RF-1.7, RF-5.4). |
| LC-7 | Two founders choose the same workspace name | Both created; each chooser row carries role and state (RF-2.4). |
| LC-8 | Name of only spaces, or longer than the bound | Field error; nothing created (RF-2.4). |
| LC-9 | Toggle unticked, project name typed earlier | No project created; the name is ignored (RF-2.5). |
| LC-10 | Toggle ticked, no box ticked | Project created; founder holds no grant on it; the warning said so; grants editable later (RF-3.3, RF-3.4). |
| LC-11 | Only `canRun` ticked | Grant record with `canRun` only, stated inert; launch impossible until a role is held (RF-3.5). |
| LC-12 | All three ticked | Grant record with Author, Reviewer and `canRun`; the founder can author, approve and run under the matrix (RF-3.3). |
| LC-13 | Bootstrap fails after the tenant row | Nothing exists; generic failure; resubmission with the same key creates once (RF-4.3, RF-4.4, RF-6.4). |
| LC-14 | Double click on the action | One idempotency key; one tenant; the second answer is the recorded outcome (RF-4.4). |
| LC-15 | Same key with a different name | Conflict; nothing further created (RF-4.4). |
| LC-16 | Factor replaced or sessions revoked between entry and submission | Precondition and session re-checked at submission; the founder meets the session handling of 006 or the refusal of RF-6.5 (RF-1.8, RF-6.6). |
| LC-17 | Session expires during creation | Expiry dialog of 006; after re-authentication the submission is looked up before "Create" is offered again (RF-6.6). |
| LC-18 | Identity becomes restricted before submission | Nothing created; the page is not reachable (RF-1.4, RF-1.8). |
| LC-19 | Creation flood from one identity or network | Throttled by the configured rate limits; no cap message; legitimate creation resumes when the window elapses (RF-1.7). |
| LC-20 | Founder reloads the page mid-form | Form empty; nothing created (RF-6.3). |
| LC-21 | "Back to sign in" by a founder without membership | Session ended; sign-in screen; nothing created (RF-6.1). |
| LC-22 | "Back to your workspaces" by a founder from the chooser | Chooser; session intact; nothing created (RF-6.2). |
| LC-23 | Founder lands on the checklist with a project | Project selected; "Environment" is the next step; no run has started (RF-5.2, RF-4.6). |
| LC-24 | Founder lands on the checklist without a project | "Project" is the next step; the later creation is F-PRJ-02 (RF-5.2, RF-3.7). |
| LC-25 | Language switched on the page | Display changes; the name and ticks are unaffected; what is created is identical (RF-9.3). |
| LC-26 | Policy tightened by the founder after creation | Done under F-SET-05 from revision 1; the bootstrap never writes a second revision (RF-4.2). |

---

## 6. Out of scope

- **Signup and email verification** — F-IDN-01, F-IDN-02, specification 002; the landing's "Create workspace" goes there first.
- **TOTP enrollment** — F-IDN-04, specification 003; this feature only forces it and returns from it.
- **Later project creation** (N22 from R14 or R09) and project configuration — F-PRJ-02, F-PRJ-04.
- **The readiness checklist content and the tenant overview** — F-PRJ-12.
- **The tenant chooser** — F-IDN-15, specification 013; only its "Create workspace" entry and the new row are referenced.
- **Editing the tenant policy** — F-SET-05; this feature creates its first revision only.
- **Inviting members, transferring ownership, editing grants** — specifications 010 and 011.
- **Tenant deletion, suspension, export** — F-IDN-16, F-IDN-17 and the tenant lifecycle features.
- **A maximum number of workspaces per identity, a workspace directory, name reservation, custom tenant identifiers, billing or plan selection** — product boundary (decision Q2, Q3).
- **Tablet- or mobile-specific designs** of N04 — none; the D01 responsive policy applies (S-02).

---

## 7. Criteria of completion

Workspace creation and bootstrap are complete when all of the following hold:

1. Every user story US-1 … US-10 can be walked end to end on the desktop design and in the reflows with no step outside this specification, and US-11 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-9 has at least one verification that fails before and passes after the feature exists.
3. Entry tests show the three entries, the verification handoff, the forced enrollment before any field with the return to the entry, the direct form for an enrolled founder, and the precondition re-check at submission (RF-1, LC-1 … LC-6, LC-16, LC-18).
4. Page tests show the designed copy verbatim, the single name field, the absent contact and integration fields, the unticked toggle and boxes, the in-progress state and the two "Back" variants (RF-2, RF-6.1, RF-6.2, LC-21, LC-22).
5. Grant tests show any subset accepted, the warning below three, inert `canRun`, nothing pre-ticked and nothing inferred (RF-3, LC-10 … LC-12).
6. Bootstrap tests show the one-transaction creation of tenant, Owner membership at version 1, policy revision 1 with the mandated values, optional project and grant record, with rollback of everything on any failure, idempotent repetition and conflict on a changed body (RF-4, RNF-1, RNF-4, LC-13 … LC-15).
7. After-creation tests show the guarded entry, Owner authority derived on the first request, the landing on the checklist with or without a project, and no change to session, factor or other memberships (RF-5, LC-23, LC-24).
8. Absence tests show no target contact, no run, no message, no invitation, no second Owner, no browser persistence, and no disclosure of other tenants (RF-7, RF-8.2, RNF-6, RNF-9).
9. Audit entries exist for every creation and every refused or failed attempt with the fields of RF-8.1 and contain no secret (RF-8).
10. The page passes WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR at desktop, tablet and mobile widths (RNF-11, RNF-12).
11. The rate-limit family and the name length bound are validated configuration; the product refuses to start without them (RNF-7).
12. The new tenant is verified isolated from its first row — composite ownership, row-level isolation and derived authority — by the suites of specification 009 (RNF-10).
13. The register items this specification relies on — A-59, A-60, A-61 (decided here), A-11, A-27 and A-36 in the architecture register; S-53, S-54 (decided here), S-02, S-04, S-14, S-17 and S-34 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
