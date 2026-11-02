# Specification 011 — Project grants editor

**Feature** The Project grants page of a project: listing who holds which grants, editing one member's Author / Reviewer / Viewer and `canRun`, adding a tenant member to the project and removing one from it · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-13 as the subject; F-IDN-11, F-IDN-10, F-IDN-12, F-IDN-14, F-IDN-16, F-IDN-18, F-PRJ-02, F-PRJ-07, F-SET-04 and F-APR-04 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P12, P13, P14) → `requirements.md` (FR-IDN-049, 067, 084, 090, 091, 092, 093, 094, 096, 098, 099, 116, 130; FR-SET-033, 052; FR-PRJ-010, 011, 012, 014, 040, 044; FR-APR-006; FR-X-001, 002, 004, 005, 020, 022, 030, 033, 034, 040, 042, 043, 044, 046, 050, 052, 053, 055) → the design of record, screen R17 ProjectMembersPage `148:1754` (the "Edit grants" panel `159:30526` and "Save project grants" `159:30539`), N22 CreateProjectDialog in the D02 atlas `159:32045` (the checkbox anatomy of initial grants), R13 TenantMembersPage `148:503` ("Review project grants"), the authority matrix of D03 `148:4`, and the ACCESS table of board 1 `23:5587`.

**Related specifications:** 009 Roles, grants and the authority matrix (the grant record this page writes, RF-9; the valid combinations and inert `canRun`, Q2; who may change whose grants, Q6 and RF-8.4; the permission version, RF-6; revocation and work in flight, RF-7; the statements every grants surface carries, RF-10.2 and RF-10.8; the audit record, RF-11) · 010 Tenant members and ownership transfer (the members page that routes here through "Review project grants"; membership statuses and their administration, which this page never changes; the confirmation-dialog pattern it builds on N23) · 008 Invitation acceptance (the other way a grant is created: an invitation carrying one project) · 012 Workspace creation (the founding Owner's bootstrap grants) · 006 Sessions, refresh and expiry (how an affected member meets their new authority: one permission refresh, then the denied-scope state) · 007 Logout and session management (the dirty guard that an open panel joins at sign-out) · 014 Tenant suspension state (no grant is edited while the tenant is suspended) · 016 Guarded tenant and project switch (the dirty guard and the project selector) · the project lifecycle specification of F-PRJ-07 (archived, trashed and deleting projects) · the approval specification (where a project member sees review authority, FR-APR-006) · the audit trail of F-APR-04 ("Decision history").

This document states **what** the project grants editor does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

A **project grant** (specification 009) is one explicit record per membership and project: the subset of the project roles Author, Reviewer and Viewer the member holds, and whether the independent permission `canRun` is granted. It is never inferred from a tenant role, from creating a project, from inviting, or from an earlier membership; a founding Owner's own grants are records the Owner ticked. Specification 009 fixed what each role permits, which combinations are valid, that `canRun` is inert without Author or Reviewer, that a grant with nothing left ceases to exist, that only an Owner or Admin may set grants — anyone's, their own included — and that every change advances the membership's permission version and is audited.

This specification is the **surface** that writes those records: the Project grants page of a project (R17), reached from the project's settings and from the members page's "Review project grants". It lists every member who holds a grant on the project with their grants, `canRun` and tenant role side by side; it lets an Owner or Admin edit one member's grants in an inline panel and save them; by decision of this specification it lets them add a tenant member who holds no grant yet, and it asks a confirmation before a save that removes a member from the project. It carries the statements the product makes everywhere about authority: that project roles and `canRun` are independent, that permission is checked at each protected action, that revocation blocks new work while evidence and provenance remain, and that being Owner or Admin grants no access to project content, approval or Run.

A change made here takes effect at the affected member's next request. Their open screens meet the denied-scope handling of specification 006; their queued and running work follows specification 009; their sessions are not ended; nothing of the membership, the tenant role, the project's content or any accepted Run changes.

### 1.2 Objective

Let a tenant Owner or Admin see and set, for one project, exactly who may author, review, read and run — while:

- never granting, removing or implying anything through a tenant role, a membership change or the act of opening the page;
- never writing a grant for a membership that is not active, on a project that is being deleted, or over a state that changed since the actor looked at it;
- never removing a member from a project by accident, and never pretending a removal erases what they authored;
- never disclosing project content, credentials or another tenant's data on a page that shows names and roles;
- never letting a Member, a project role or `canRun` administer grants, and never reaching another tenant's project.

### 1.3 Why it matters

The grants page is where the authority matrix stops being a table in a document and becomes somebody's decision. If it lets an Author tick their own Reviewer box, approval stops being independent; if it hides who holds `canRun`, a launch against a live target becomes untraceable; if a removal fails silently or applies over a colleague's change, authority drifts from what the audit log says. The page has to make every grant visible, every change deliberate and every outcome exact, for an administrator who may hold no grant of their own on the project they administer.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Decisions that depend on an architecture or design decision are also recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row. Exact location, all written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-51** ("Add project member" picker) and **S-52** (removal confirmation dialog) are new; notes are appended to **S-04** (destinations of the remaining buttons, now D02 `439:4460`), **S-13** (the "Current revisions" filter), **S-47** (the inert `canRun` statement is built here), **S-06** (role variants) and **S-02** (responsive coverage). Page "ASpectra Architecture" (`0:1`) → spec block `122:2` → §14: notes appended to **A-55** (the permission version as the stale-save guard) and **A-57** (administration authority applied by this surface).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope: "Add project member" | **Included.** R17 draws the button (S-04). "Add project member" opens a picker listing the tenant's active memberships that hold no grant on this project — the actor included, so a solo Owner can add themselves — choosing one opens the same edit panel titled for that member, and saving creates the grant. Without it a member invited without a project, or whose grant was removed, could never be given one. Drawn in D02 `420:2968`; D03 ledger S-51. |
| Q2 | Who may open the page | A tenant **Owner or Admin only**, for any project of the tenant, whether or not they hold a grant on it (specification 009 Q6: that is how a solo Owner obtains grants). A Member meets the denied-scope state, as 010 decided for the members page. Project members learn who holds Reviewer or `canRun` from the approval surface (FR-APR-006), not from this page. |
| Q3 | Confirmation and step-up on "Save project grants" | No step-up. A save that changes or adds roles or `canRun` applies at once from the panel. A save that would leave the member with **no role and no `canRun`** — removing them from the project — opens a compact **confirmation dialog** (built on the N23 anatomy, drawn in D02 `420:2968`; D03 ledger S-52) stating the effect; confirming applies the removal, cancelling returns to the panel. The actor's own row is marked "you" and is edited under the same rules. |
| Q4 | Concurrent changes between opening the panel and saving | **A stale save is refused.** The save carries the grant state the panel was opened on; if the member's grants, their membership status or the project's state changed meanwhile, the save is refused with a stated reason, nothing changes, the row is reloaded and the panel reopens on the current values for the actor to re-apply. Consistent with 009 LC-18. |
| Q5 | Telling the affected member | **No email.** The change is audited and takes effect at the member's next request (009 RF-6.4, RF-7); the projects list and the chooser show the current grants. No template is added to S-14. |
| Q6 | What the list holds and what the filters mean | One row per **membership holding a grant on this project**; a revoked membership never appears. A suspended membership's row is shown with the state "suspended" and its grants read-only until it is reinstated (009 RF-9.7). "All statuses" filters by membership status; the search matches a member's display name or verified email; **"Current revisions" is a copy defect and is not rendered** (note on S-13). The page size is a configured value. |
| Q7 | Project lifecycle state | Grants may be added, changed and removed while the project is **active or archived** (a Viewer grant still gives read access to archived content). While the project is **trashed or deleting** the page is read-only with the reason stated and without "Add project member"; grants end with the project on deletion (009 LC-30). |

Five rules below are derived from these decisions and from earlier specifications rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** While the tenant is suspended or deleting, no surface of this feature is reachable and no change is accepted: the suspended-tenant screen of specification 014 replaces every tenant surface (FR-IDN-116; specification 010 D-b).
- **D-b** An open edit panel with unsaved changes is unsaved work: it joins the dirty guard of the guarded switch (F-IDN-18) and of sign-out (specification 007 Q4) as Save / Discard / Stay; "Cancel changes" discards them explicitly.
- **D-c** "Decision history ↗" routes to the audit trail (F-APR-04) scoped to the grant events of this project that the actor's current authority permits them to read (as specification 010 RF-3.4 scopes it for memberships).
- **D-d** The page size of the list and of the picker is a configured value with no default; the search matches display name and verified email and never a project, source or run.
- **D-e** An actor who loses the Owner or Admin role, or whose membership is suspended, while on the page meets the denied-scope handling of specification 006 at their next request; a save in flight is refused as stale under the advanced version.

Register items this specification relies on: **S-51** and **S-52** (decided here), the pre-existing decided **A-53** (grant combinations and inert `canRun`), **A-55** (permission version; note on its use as the stale-save guard), **A-57** (administration authority; note that this surface applies it), **A-58** (lifecycle grant), **S-47** (the inert `canRun` statement, built here), **S-49** (the Admin view of R13, the same pattern applied to R17), and **S-04** (destinations of the remaining buttons, D02 `439:4460`; "Add project member" is decided here), **S-06** (role variants, D02 `425:3736`: R17's Admin view is identical and the Member view is the denied-scope state), **S-13** (terminology and copy; the "Current revisions" filter is removed from R17), **S-02** (responsive coverage; R17, the picker and the dialog follow the D01 responsive policy), **S-17** ("Help & shortcuts" opens N43 `430:4129`), **A-11** (configuration values) and **A-36** (identity-scoped membership listing, on which the picker's membership facts depend).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | tenant Owner | to see, for one project, every member with their grants, `canRun` and tenant role side by side | I know exactly who may author, review, read and run here |
| US-2 | tenant Owner or Admin | to change one member's roles and `canRun` independently and save them in one action | authority matches what the team actually needs today |
| US-3 | founding Owner working alone | to add myself to a project and tick my own Author, Reviewer and `canRun`, visibly | I can work without an implied right I cannot audit |
| US-4 | tenant Admin | to add a tenant member who was invited without a project | nobody is stranded without a way into the work |
| US-5 | administrator removing a contractor from a project | to be asked once, told what the removal does, and have it apply from their next action | removal is deliberate, immediate and honest about what stays |
| US-6 | administrator editing while a colleague edits the same member | to be told my save is stale and shown the current values | we never overwrite each other silently |
| US-7 | administrator looking at a suspended member's row | to see their grants kept but read-only, with their state | I understand reinstatement will restore them unchanged |
| US-8 | administrator of an archived project | to still grant Viewer so an auditor can read it | archiving does not lock evidence away |
| US-9 | tenant Member | to be told plainly that this page is not mine | I do not mistake a denied page for a broken one |
| US-10 | administrator on a phone or tablet | the list, the panel, the picker and the dialog reflowed with every statement and action | I can fix a grant from anywhere |
| US-11 | attacker or over-reaching party (negative story) | — | I cannot administer grants as a Member or through a project role, reach another tenant's project, write a grant for a suspended or revoked membership, commit over a stale state, learn project content from this page, or remove someone's authored evidence by removing their grant |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the page* means R17 for one project of the current tenant; *the actor* means the signed-in Owner or Admin acting on it; *the target* means the member whose grant is listed, edited, added or removed; *the panel* means the inline "Edit <name>'s grants" editor; *the picker* means the "Add project member" chooser of decision Q1; *the dialog* means the removal confirmation of decision Q3; *grant* and *permission version* have the meaning of specification 009.

### RF-1 · Reaching the page and who may act

The page belongs to tenant administration, not to project content. *Why:* an Owner or Admin must be able to set grants on a project they cannot read, and nobody else may set any. *(Decisions: Q2, Q7.)*

- RF-1.1 The system shall present the page under the project's settings as the "Project members" tab, beside the Integrations, Environments and Configuration tabs, with the heading "Project grants" and the sub-statement "Author, Reviewer and Viewer are project roles. canRun is an independent permission.", and shall reach it also from "Review project grants" on the members page (specification 010 RF-3.3) for the current project. *(FR-SET-052, FR-SET-033; R17)*
- RF-1.2 While the actor's membership is active with the Owner or Admin role and the tenant is active, the system shall open the page for any project of the tenant, whether or not the actor holds a grant on it. *(FR-IDN-098; specification 009 RF-8.4; decision Q2)*
- RF-1.3 If the actor's membership holds the Member role, is suspended, or does not exist in the tenant, then the system shall deny the page with the denied-scope handling of specification 006 and disclose nothing about the project or its members. *(FR-IDN-091, FR-X-002; specification 006 RF-5.1; decision Q2)*
- RF-1.4 While the tenant is suspended or deleting, the page shall be unreachable and no change shall be accepted (D-a). *(FR-IDN-116; specification 014)*
- RF-1.5 While the project is trashed or deleting, the system shall present the page read-only: the list without edit actions, no "Add project member", and a statement naming the reason; while the project is active or archived every action of this feature shall be available. *(FR-PRJ-014, FR-PRJ-040; decision Q7)*
- RF-1.6 The page shall show nothing of the project's content — no suite, feature, scenario, source, environment, credential or run — and nothing of any other tenant; it shall show only the members of this tenant and their grants on this project. *(FR-IDN-092, FR-X-030; P12)*
- RF-1.7 When the actor loses the Owner or Admin role, or their membership is suspended, while on the page, the next request shall be denied as RF-1.3 states and a save in flight shall be refused as RF-4.6 states (D-e). *(specification 009 RF-6.3, RF-7.1)*

### RF-2 · The list

The list shows every grant on the project, and only grants. *Why:* a member is "in the project" exactly when a grant record exists (specification 009 RF-9.4); the list is that fact made visible. *(Decision: Q6.)*

- RF-2.1 The system shall present a table with the columns Member, Project grants, canRun and Tenant role, one row per membership of the tenant that holds a grant on this project, ordered by member display name, with the record count and "Previous · Next" pagination of a configured page size (D-d). *(FR-IDN-098, FR-SET-052; R17; decision Q6)*
- RF-2.2 The Member cell shall show the member's display name, with "/ you" appended on the actor's own row; the Project grants cell the roles held joined by "+" (for example "Author + Reviewer"); the canRun cell "Granted" or "Not granted"; the Tenant role cell Owner, Admin or Member, for contrast. *(FR-IDN-098; R17)*
- RF-2.3 A revoked membership shall never appear; a membership with no grant on the project shall not appear in the list and is reachable only through the picker (RF-5). *(specification 009 RF-4.4, RF-9.4; decision Q6)*
- RF-2.4 While a listed member's membership is suspended, the row shall show the state "suspended" beside the member, its grants shall be shown read-only with the statement that they are kept and will apply again on reinstatement, and the row shall offer no edit. *(FR-IDN-093; specification 009 RF-4.3, RF-9.7; decision Q6)*
- RF-2.5 The system shall provide a search field "Find a project member" matching the display name or verified email of listed members, and a filter "All statuses" with the values all, active and suspended; the "Current revisions" control drawn on R17 shall not be rendered. *(R17; decision Q6; S-13)*
- RF-2.6 Where `canRun` is granted to a member who holds neither Author nor Reviewer, the row shall state beside "Granted" that `canRun` has no effect until a project role is held. *(specification 009 RF-10.8; S-47)*
- RF-2.7 The list shall be read when the page opens and again after each save, addition or removal of this feature; it shall never be polled; a grant that changed since the last read is shown at the next read, and a save over it is handled by RF-4.6. *(FR-IDN-099)*
- RF-2.8 The system shall present the route "Decision history ↗" to the audit trail scoped as D-c states. *(FR-IDN-099, FR-X-044; F-APR-04; R17)*

### RF-3 · The edit panel

One member, one panel, one explicit save. *Why:* R17 draws an inline editor with independent checkboxes; independence of the roles and of `canRun` is the product's rule, and the panel must make it visible. *(Decision: Q3.)*

- RF-3.1 When the actor activates a row of an active membership, the system shall open the panel "Edit <member>'s grants" below the list, with the group "Project roles" holding the checkboxes Author, Reviewer and Viewer, the group "Independent execution permission" holding the checkbox canRun, each reflecting the member's current grant, and the actions "Cancel changes" and "Save project grants". *(FR-IDN-098; R17 `159:30526`)*
- RF-3.2 The panel shall carry the statements "Author can create revisions and import sources. Reviewer can decide approval. canRun authorizes only an explicit Run with valid current approval." and "Current permission is checked at each protected action. Revocation blocks new work; existing evidence and authored provenance remain." *(FR-IDN-098, FR-SET-052; R17)*
- RF-3.3 The checkboxes shall be independent: any subset of Author, Reviewer and Viewer may be ticked with or without canRun; ticking one shall never tick or untick another. *(FR-IDN-090, FR-IDN-091; specification 009 RF-9.2, RF-9.3)*
- RF-3.4 While canRun is ticked and neither Author nor Reviewer is, the panel shall state beside canRun that it has no effect until a project role is held; the save shall still be permitted. *(specification 009 Q2, RF-10.8; S-47)*
- RF-3.5 While no role and no canRun is ticked, the panel shall state that saving removes the member from this project, and the save shall lead to the dialog of RF-6. *(specification 009 RF-9.4; decision Q3)*
- RF-3.6 "Cancel changes" shall discard the panel's changes and close it; nothing shall be written. *(R17)*
- RF-3.7 The panel's unsaved changes shall count as unsaved work for the dirty guard of the guarded switch and of sign-out (D-b); at most one panel shall be open at a time, and opening another row shall pass through the same guard. *(FR-IDN-130; specification 007 Q4)*
- RF-3.8 The panel shall show the member's tenant role beside the heading for contrast and shall offer no control over the tenant role, the membership status or anything but the four grant flags. *(FR-IDN-098; specification 010 RF-11.1)*

### RF-4 · Saving a change

A save is one indivisible change of one grant record, applied by specification 009's rules, over exactly the state the actor saw. *Why:* FR-IDN-099 and FR-X-005 — authority that changes must be versioned, audited and never committed over a stale fact. *(Decisions: Q3, Q4, Q5.)*

- RF-4.1 When "Save project grants" is activated with at least one role or canRun ticked, the system shall set the target's grant on this project to exactly the ticked subset and flag, as one indivisible change that advances the target membership's permission version and writes the audit record of specification 009 RF-11.1, and shall report the result in the row. *(FR-IDN-098, FR-IDN-099, FR-X-005; specification 009 RF-8.6, RF-9.1)*
- RF-4.2 The system shall permit the save only when the actor's membership is active with the Owner or Admin role at the moment of the save; the actor may save a change to their own grant, which is audited with actor and target the same. *(specification 009 RF-8.4, Q6; A-57)*
- RF-4.3 The system shall require no re-authentication and no further confirmation for a save that leaves at least one role or canRun. *(decision Q3)*
- RF-4.4 If the save changes nothing — the ticked state equals the current grant — then the system shall write nothing, advance no version, record nothing and report the panel as unchanged. *(FR-X-020; specification 009 RF-11.4)*
- RF-4.5 If the target's membership is not active, or the project is trashed or deleting, at the moment of the save, then the system shall refuse the save with a stated reason and change nothing. *(specification 009 RF-9.7; decision Q7)*
- RF-4.6 If the target's grant, their membership status or the project's state changed since the panel was opened, then the system shall refuse the save as stale with the statement that the grants changed since the panel was opened, change nothing, reload the row and reopen the panel on the current values. *(FR-X-005; specification 009 LC-18; decision Q4)*
- RF-4.7 A repeated submission of the same save shall yield one change and the same answer. *(FR-X-020, FR-X-022; specification 009 RF-11.4)*
- RF-4.8 The system shall send no email for a grant change; the effect reaches the target at their next request (RF-9). *(decision Q5)*
- RF-4.9 A save shall never change the target's membership, tenant role or status, any other member's grant, any other project, the tenant policy, the project's content or lifecycle, or any session. *(FR-IDN-092, FR-IDN-116; specification 010 RF-11.1; P12)*

### RF-5 · Adding a project member

A tenant member with no grant joins the project by being given one. *Why:* a grant is the only way into a project, and the invitation (008) and creation (012, F-PRJ-02) paths cannot reach a member already in the tenant. *(Decision: Q1.)*

- RF-5.1 While the project is active or archived, the system shall present "Add project member"; when activated it shall open the picker listing the tenant's active memberships that hold no grant on this project, with their display name and tenant role, searchable by display name or verified email, paged by the configured size (D-d). *(FR-IDN-098; decision Q1; S-51)*
- RF-5.2 The picker shall include the actor's own membership when it holds no grant on the project, and shall never list a suspended or revoked membership, a membership of another tenant, or an identity that is not a member. *(specification 009 RF-9.7, Q6; P12)*
- RF-5.3 When a membership is chosen, the system shall open the panel of RF-3 titled "Add <member> to this project" with every checkbox unticked and the same statements and actions. *(decision Q1)*
- RF-5.4 When "Save project grants" is activated with at least one role or canRun ticked, the system shall create the grant as RF-4.1 states; a new row appears in the list. *(FR-IDN-098, FR-IDN-099; specification 009 RF-9.1)*
- RF-5.5 If nothing is ticked when saving from the picker, then the system shall not save and shall state that at least one role or canRun must be chosen; no removal dialog applies, since no grant exists. *(decision Q3)*
- RF-5.6 If the chosen membership gained a grant, was suspended or revoked, or the project left the editable states since the picker was opened, then the save shall be refused as RF-4.5 or RF-4.6 states and the picker reloaded. *(decision Q4)*
- RF-5.7 Adding a member to a project shall never create a membership, send an invitation, change a tenant role or lift a suspension. *(FR-IDN-080; specification 008; specification 010 RF-11.4)*

### RF-6 · Removing a member from the project

Removal is a save that empties the grant, confirmed once. *Why:* FR-IDN-098 requires the surface to say that revocation blocks new work while evidence remains; a confirmation is where that statement is read before it applies. *(Decision: Q3.)*

- RF-6.1 When "Save project grants" is activated with no role and no canRun ticked for a member who holds a grant, the system shall open the dialog stating that the member will lose access to this project from their next action, that queued work of theirs on this project will not run and running work will be asked to stop at a bounded checkpoint, that their authored revisions, accepted mappings, approvals, runs and evidence remain with their provenance, and that their tenant membership is unchanged; the dialog shall offer "Remove from project" and "Cancel". *(FR-IDN-098, FR-IDN-094, FR-IDN-067; specification 009 RF-7; decision Q3; S-52)*
- RF-6.2 When the removal is confirmed, the system shall end the grant as one indivisible change that advances the target's permission version and writes the audit record, under the same stale and state checks as RF-4.5 and RF-4.6; the row shall leave the list at the next read. *(specification 009 RF-9.4, RF-11.1; decision Q4)*
- RF-6.3 When the dialog is cancelled, the system shall return to the panel with its ticked state unchanged and write nothing. *(decision Q3)*
- RF-6.4 The actor may remove their own grant; the dialog shall say so in the second person and the actor's own next protected action on the project shall be denied as any member's. *(specification 009 RF-8.4, Q6)*
- RF-6.5 A removal shall never revoke, suspend or otherwise change the membership, and a removed member shall be addable again through RF-5 with a new grant, never a restored one. *(specification 009 RF-4.4, RF-12.3; specification 010 RF-7.10)*

### RF-7 · What the page says

The statements of specification 009 RF-10.2, verbatim. *Why:* consistency across surfaces is what makes the promise believable.

- RF-7.1 The page shall state, in the card "Workspace administration is a different scope", "Being Owner or Admin does not implicitly grant access to project content, approval or Run." *(FR-IDN-092; specification 009 RF-10.2; R17)*
- RF-7.2 The page shall carry the sub-statement of RF-1.1 and the panel statements of RF-3.2 for every actor, on every row and at every viewport width; the Admin view shall be identical to the Owner view. *(FR-SET-052; specification 009 RF-10.9; S-06, S-49)*
- RF-7.3 Where an action is unavailable — a suspended member's row, a trashed project, a Member's attempt to open the page — the surface shall say why or deny safely, never fail silently. *(FR-PRJ-012; specification 009 RF-10.7)*

### RF-8 · Boundaries

The page administers grants and nothing else. *Why:* every extra capability on an administration page is an escalation path.

- RF-8.1 No action of this feature shall create, suspend, reinstate or revoke a membership, change a tenant role, transfer ownership, change the tenant policy, invite anyone, end a session, or touch another identity's password, factor, recovery codes or erasure. *(FR-IDN-049, FR-IDN-097; specifications 010, 007, 004, 003, 005)*
- RF-8.2 No action of this feature shall accept a mapping, approve a bundle, launch, cancel, replay or re-verdict a Run, or change the project's content or lifecycle. *(FR-IDN-067; P1)*
- RF-8.3 The system shall offer no grant other than Author, Reviewer, Viewer and canRun, no team, no custom role and no grant on more than one project per panel. *(FR-IDN-090)*
- RF-8.4 Every request of this feature shall be bound to the actor's current tenant and the project requested in the route; a project of another tenant, a route and body that disagree, or a missing context shall be denied safely without disclosing existence. *(FR-X-001, FR-X-002; P12)*
- RF-8.5 A Member, a project role, canRun or any grant shall never authorise opening the page, saving, adding or removing. *(FR-IDN-091; specification 009 RF-8.4, RF-12.2)*

### RF-9 · Effect on the affected member

What this page writes, specification 009 enforces. *Why:* the page must not promise an effect the derivation does not deliver, nor a gentler one.

- RF-9.1 When a grant is changed, added or removed, the target's next protected action on the project shall be derived under the new grant and the advanced permission version; a sensitive write prepared under the old version shall be refused as stale and the target shall meet the handling of specification 006 RF-5.1. *(FR-X-004, FR-X-005; specification 009 RF-6.3, RF-6.4, RF-7.1)*
- RF-9.2 When a grant is removed or narrowed, the target's subscriptions in the affected scope shall be closed, their queued tickets on the project refused as NOT EXECUTED, their running attempts asked to stop at a bounded checkpoint, their recorded facts and verdicts untouched, and their unsaved client work handled under specification 006. *(FR-IDN-094, FR-IDN-067; specification 009 RF-7.2–7.6)*
- RF-9.3 A grant change shall never end the target's session, and a grant added shall be usable from the target's next request without a new sign-in. *(specification 007; specification 006 RF-2.5)*
- RF-9.4 Authored revisions, accepted mappings, approvals, runs and evidence of the target shall remain in the project with their provenance whatever the change. *(FR-IDN-098; specification 009 RF-7.5)*

### RF-10 · Audit, secrecy and safe problems

Every change is recorded with who, whom, what and from what; every refusal is safe. *Why:* FR-IDN-099; a grant that cannot be reconstructed cannot be reviewed.

- RF-10.1 Every save, addition and removal shall be recorded as specification 009 RF-11.1 states — acting membership and identity, target membership and identity, tenant, project, previous grant, new grant, resulting permission version, time, correlation id — and every refused save (stale, membership not active, project not editable, actor not permitted) shall be recorded with its reason class. *(FR-IDN-099, FR-X-044)*
- RF-10.2 Every refusal of this feature shall be a safe problem with a correlation id, distinguishing at least: denied scope, stale state, membership not active, project not editable, and nothing chosen; no problem shall disclose project content, another member's data or whether a project exists outside the actor's tenant. *(FR-X-033)*
- RF-10.3 No response, log, export or analytics event of this feature shall contain a credential, a secret or project content. *(FR-X-030)*
- RF-10.4 Every write of this feature shall confirm persistence before reporting success and shall be observable as done or not done, never half-done. *(FR-X-020)*

### RF-11 · Responsive and accessible surfaces

Administering grants is a critical journey. *Why:* the person fixing a grant may be on a phone.

- RF-11.1 Where no designed mobile or tablet counterpart exists for the page, the panel, the picker or the dialog, the system shall present them reflowed under the desktop rules, converting the table to one record per member with every cell, statement and action intact. *(FR-X-055; S-02)*
- RF-11.2 Every checkbox, action and route shall have a visible label and focus state and, on responsive surfaces, a touch target of at least 44 px; the checkboxes shall be grouped under their group labels; the result of a save and a refusal shall be conveyed by text; when the panel, the picker or the dialog opens, focus shall enter its heading or first control and return to the invoking element when it closes; a live announcement shall never steal focus. *(FR-X-040, FR-X-042, FR-X-053)*
- RF-11.3 All states and messages of this feature shall exist in EN / ES / FR; the interface language shall never change a grant, a role name as recorded, or the outcome of an action. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Explicit records only.** Every grant shown or written is a record of specification 009; nothing is inferred from a tenant role, from creating or administering a project, or from opening this page. | FR-IDN-096, FR-IDN-098 |
| RNF-2 | **Server-derived authority.** Who may open the page, save, add or remove is derived on the server at the moment of the action from the actor's current membership; the client only hides or disables. | FR-X-001, FR-X-004; P3 |
| RNF-3 | **Atomic change.** A save, addition or removal, its version advance and its audit record commit together or not at all. | FR-IDN-099, FR-X-005 |
| RNF-4 | **No stale write.** A save over a grant, membership or project state that changed since the panel opened never commits. | FR-X-005; decision Q4 |
| RNF-5 | **Fail closed.** A non-active membership, a trashed or deleting project, a suspended tenant, a missing context or an unknown project yields no change. | P6 |
| RNF-6 | **Tenant isolation.** The page lists and writes only memberships and grants of the actor's current tenant on a project of that tenant. | FR-X-002, FR-X-010; P12 |
| RNF-7 | **Revocation latency.** From the moment a change commits, the target's next request is derived under it. | FR-IDN-094; specification 009 RNF-7 |
| RNF-8 | **No fabricated outcome.** A removal never marks work failed, never rewrites a verdict, never erases evidence or provenance. | FR-IDN-094, FR-IDN-067; P8 |
| RNF-9 | **No content disclosure.** The page carries names, tenant roles and grants only. | FR-IDN-092, FR-X-030 |
| RNF-10 | **Idempotent writes.** A repeated save yields one change and the same answer; an unchanged save writes nothing. | FR-X-020, FR-X-022 |
| RNF-11 | **Configured limits.** The page sizes of the list and the picker are validated configuration with no default. | FR-X-046; P13 |
| RNF-12 | **No notification.** No email is sent by this feature. | decision Q5 |
| RNF-13 | **Accessibility.** The page, the panel, the picker and the dialog meet WCAG 2.2 AA with automated and manual assistive evidence. | FR-X-040, FR-X-042 |
| RNF-14 | **Language.** EN / ES / FR, display only. | FR-X-043 |
| RNF-15 | **Reflow.** Mobile and tablet reflows preserve every cell, statement and action. | FR-X-055 |
| RNF-16 | **Verification.** Grant races, stale-version writes, revoked grants during jobs, administration by each tenant role, and cross-tenant attempts are verified as areas A and B require. | requirements A.12, B.11; P9 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Owner without a grant on the project opens the page | Page opens; list shown; the Owner is not listed; "Add project member" lists them (RF-1.2, RF-5.2). |
| LC-2 | Member holding Author, Reviewer and canRun opens the page | Denied scope; nothing disclosed (RF-1.3). |
| LC-3 | Admin opens the page | Identical to the Owner view; every action available (RF-1.2, RF-7.2). |
| LC-4 | Solo Owner adds themselves with Author, Reviewer and canRun | Grant created; row "/ you"; audited with actor = target (RF-5, RF-4.2). |
| LC-5 | canRun ticked for a Viewer-only member | Saved; inert statement in the panel and the row; launch refused by 009 (RF-3.4, RF-2.6). |
| LC-6 | Viewer ticked beside Author | Saved as recorded; no change of authority (RF-3.3; 009 RF-9.2). |
| LC-7 | All boxes unticked, then Save | Dialog; confirm → grant ended, row gone at next read; cancel → panel unchanged (RF-6.1–6.3). |
| LC-8 | Save with the same state as the current grant | Nothing written, no version advance, panel reported unchanged (RF-4.4). |
| LC-9 | Two Owners edit the same member; the second saves after the first | Second save refused as stale; row reloaded; panel reopened on the current values (RF-4.6). |
| LC-10 | Target suspended by another Admin while the panel is open | Save refused: membership not active; row reloaded showing "suspended" (RF-4.5, RF-2.4). |
| LC-11 | Target removed from the tenant while the panel is open | Save refused; row gone at reload (RF-4.5, RF-2.3). |
| LC-12 | Project trashed while the panel is open | Save refused: project not editable; page becomes read-only with the reason (RF-1.5, RF-4.5). |
| LC-13 | Project archived | Page fully editable; Viewer granted to an auditor (RF-1.5; decision Q7). |
| LC-14 | Project deleted | Grants end with the project; the page is unreachable; audit records the ends (009 LC-30). |
| LC-15 | Tenant suspended while the actor is on the page | Next request routes to the suspended-tenant screen; a save in flight refused (RF-1.4; D-a). |
| LC-16 | Actor demoted to Member while on the page | Next request denied scope; a save in flight refused as stale (RF-1.7; D-e). |
| LC-17 | Actor removes their own last grant | Dialog in the second person; confirmed → the actor's next protected action on the project is denied; the page itself stays open (RF-6.4). |
| LC-18 | Picker opened; the chosen member was just given a grant by a colleague | Save refused as stale; picker reloaded without that member (RF-5.6). |
| LC-19 | Picker save with nothing ticked | Not saved; "choose at least one role or canRun"; no dialog (RF-5.5). |
| LC-20 | Suspended member's row activated | No panel; grants read-only with the reinstatement statement (RF-2.4). |
| LC-21 | Revoked former member | Not listed; not in the picker; re-invitation creates a new membership first (RF-2.3, RF-5.2). |
| LC-22 | Search by email of a member not on the project | No result in the list; found in the picker (RF-2.5, RF-5.1). |
| LC-23 | Target is editing a scenario when their Author grant is removed | Next sensitive write refused as stale; one permission refresh; denied-scope state; draft shown only if still permitted (RF-9.1; 006 RF-5.3). |
| LC-24 | Target has a queued ticket and a running attempt when canRun is removed | Ticket refused as NOT EXECUTED; attempt asked to stop at a checkpoint; facts and verdict untouched (RF-9.2). |
| LC-25 | Panel open with changes; actor signs out from the shell | Dirty guard Save / Discard / Stay; Save applies the grant under the actor's current authority (RF-3.7; 007 Q4). |
| LC-26 | Panel open with changes; actor switches project from the header | Dirty guard of the switch; the save goes to the original project (RF-3.7; F-IDN-18). |
| LC-27 | Request names a project of another tenant | Denied safely; existence undisclosed (RF-8.4). |
| LC-28 | Page opened on a phone | One record per member with the four cells and the edit action; panel, picker and dialog reflowed (RF-11.1). |
| LC-29 | Language switched while the panel is open | Display changes; ticked state and the pending save unchanged (RF-11.3). |
| LC-30 | Repeated save after a lost answer | One change; the same answer; one audit entry (RF-4.7). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The authority matrix, the grant record, valid combinations, inert canRun, the permission version and revocation in flight** — F-IDN-11, specification 009; this page writes what 009 defines and claims no other effect.
- **Memberships, tenant roles, suspension, removal, ownership transfer and the members page** — F-IDN-12, specification 010; "Review project grants" only routes here.
- **Invitations** — F-IDN-10, specification 008; adding a project member never invites.
- **Project creation and its initial grants (N22)** and the "My grants" filter of the projects list — F-PRJ-02.
- **The founding Owner's bootstrap grants** — F-IDN-14, specification 012.
- **Project lifecycle itself** — archive, trash, restore, purge, deletion — F-PRJ-07; this page only reads the project's state.
- **The approval surface's review-authority display** — F-APR-04 / FR-APR-006.
- **The audit trail surface** — F-APR-04; "Decision history" only routes there.
- **Tenant policy** — F-SET-05.
- **Integrations, Environments and Configuration tabs** of project settings — their own specifications.
- **Notifications** for grant changes — decided out (Q5).
- **Teams, custom roles, grants on several projects at once, bulk edits, grant templates, time-bound grants, machine principals with grants** — product boundary (FR-IDN-090; FR-EXT-041 deferred).
- **Tablet- or mobile-specific designs** of R17, the picker and the dialog — none; the D01 responsive policy applies (S-02).
- **"Help & shortcuts"** in the app shell — the shell's N43 HelpShortcutsPanel (`430:4129`) (S-17).

---

## 7. Criteria of completion

The project grants editor is complete when all of the following hold:

1. Every user story US-1 … US-10 can be walked end to end on the desktop design and in the reflows with no step outside this specification, and US-11 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-11 has at least one verification that fails before and passes after the feature exists.
3. Access tests show the page open for an Owner and an Admin with and without a grant on the project, denied for a Member and a suspended actor, unreachable in a suspended tenant, read-only for a trashed or deleting project and editable for an archived one (RF-1, LC-1 … LC-3, LC-12 … LC-16).
4. List tests show one row per grant, the four cells, "/ you", the suspended state read-only, no revoked membership, the inert-canRun statement, the search and status filter, configured pagination, on-demand reading and the absent "Current revisions" control (RF-2, RNF-11, LC-5, LC-20 … LC-22).
5. Panel tests show independent checkboxes, the statements, the inert and removal notices, Cancel, the dirty-guard participation and the single open panel (RF-3, LC-5, LC-6, LC-25, LC-26).
6. Save tests show one atomic change with version advance and audit, self-edits permitted, no step-up, an unchanged save writing nothing, refusals for a non-active membership and a non-editable project, stale refusal with reload, idempotent repeats and no email (RF-4, RNF-3, RNF-4, RNF-10, RNF-12, LC-4, LC-8 … LC-11, LC-30).
7. Picker tests show active memberships without a grant only, the actor included, suspended and revoked excluded, the titled panel, creation on save, the nothing-ticked refusal and the stale refusal (RF-5, LC-1, LC-4, LC-18, LC-19, LC-21, LC-22).
8. Removal tests show the dialog's statements, confirm and cancel, the ended grant, the second-person self-removal, and the membership untouched (RF-6, LC-7, LC-17).
9. Statement tests show the texts of RF-7 verbatim on every role variant and at every width, and every unavailable action explained (RF-7, RNF-9).
10. Boundary and effect tests show that nothing but the grant record changes, that another tenant's project is denied, that the target's next request is derived under the new grant, that queued and running work follow specification 009, that no session ends and that evidence remains (RF-8, RF-9, RNF-5 … RNF-8, LC-23, LC-24, LC-27).
11. Audit entries exist for every change and every refusal with the fields of RF-10.1, contain no credential or content, and every refusal is a safe problem with a correlation id (RF-10).
12. The page, the panel, the picker and the dialog pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RF-11, RNF-13 … RNF-15, LC-28, LC-29).
13. The register items this specification relies on — S-51 and S-52 (decided here), S-47, S-49, S-04, S-06, S-13, S-02 and S-17 in the D03 Screen decisions ledger (`148:4`); A-53, A-55, A-57, A-58, A-11 and A-36 in the architecture register — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
