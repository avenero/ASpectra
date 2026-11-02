# Specification 007 — Logout and session management

**Feature** Sign-out, the sessions list, individual revocation and "Sign out everywhere" · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-09 as the subject, as amended in v1.3; F-IDN-03, F-IDN-04, F-IDN-06, F-IDN-07, F-IDN-08, F-IDN-10, F-IDN-15, F-IDN-16, F-IDN-18 and F-SET-03 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P13, P14, P16) → `requirements.md` (FR-IDN-024, 045, 049, 060, 061, 062, 064, 066, 067, 068, 069, 082, 110, 116, 130, 131; FR-SET-010, 020, 023, 024, 034; FR-RUN-050; FR-X-001, 002, 020, 021, 022, 023, 030, 033, 034, 040, 042, 043, 044, 046, 050, 052, 053, 055) → the design of record, the Sessions area of R38 SecuritySettingsPage `148:3895`, R34 SessionExpiryDialog `148:489`, R08 ForceChangePasswordPage `148:43`, N03 TenantChooserPage `161:49326`, N20 TenantSuspendedPage `161:50995`, M03 MobileInvitationPage `161:51284`, M04 `161:51344` (the mobile menu), N24 TenantSwitchDirtyDialog (D02 `159:32045`), and the "Session expiry, refresh, logout and accepted runs" item of decision register `122:2`.

**Related specifications:** 001 Login (the sign-in screen every sign-out returns to; the limited identity state, which is not a session) · 003 TOTP enrollment, recovery codes and factor replacement (a completed replacement ends every other session, RF-9.7; the Sessions area shares R38 with the TOTP & recovery area that 003 owns) · 004 Password reset and forced password change ("Sign out safely" on R08 abandons a limited identity state, 004 RF-8.12, and is cited here, not redefined; a reset and a compromise response end every session) · 005 Controlled account recovery (a completed recovery ends every other session, RF-3.4) · 006 Sessions, refresh and expiry (what a session is, its deadlines and its session / security version; the definitive answer that a session ended; the dialog R34 and its "sign out safely" exit; the retry / offline status; multi-tab behaviour; the audit of session ends) · 008 Invitation acceptance (the "Switch identity safely" exit of N29 and M03, and the return to the invitation after a new sign-in) · 013 Tenant chooser (the "Sign out" of N03) · 014 Tenant suspension state (the "Sign out" of N20) · 016 Guarded tenant and project switch (the dirty guard Save / Discard / Stay that sign-out reuses) · the execution and monitoring specifications (an accepted Run survives every sign-out).

This document states **what** sign-out and session management do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

Specification 006 defines the **session**: the server's memory that one browser context is acting for one identity, represented to the browser by an access credential and a refresh credential, ended by its deadlines, by a change of the identity's session / security version, by individual revocation or by sign-out. A session carries no tenant or project authority and is never a trusted device.

This feature gives the person the two deliberate ways of ending sessions. **Sign-out** ends the session the person is using: it is offered on the tenant chooser (N03), the suspended-tenant screen (N20), the invitation screens (M03 and N29, "Switch identity safely"), the expiry dialog (R34, "sign out safely"), the forced-change screen (R08, "Sign out safely", which acts on a limited identity state and is defined by specification 004) and, by decision of this specification, in the identity area of the application shell on every screen reached with a session. **Session management** is the Sessions area of Security Settings (R38): a table "Authenticated sessions" listing every live session of the identity with its session, context, server-supplied expiry and action, marking the current session, letting the person revoke any other one, and — by decision of this specification — offering "Sign out everywhere". The area states that sessions are not trusted devices, that no session grants a TOTP bypass, and that expiring a session does not cancel an accepted run.

A sign-out or a revocation ends a session **on the server**. From that moment every request carrying its credentials receives the definitive answer of specification 006 that the session is no longer valid, and nothing is served on the strength of it. The browser that performed a sign-out clears everything it held and returns to the sign-in screen; any other tab or device that shared or held an ended session learns it at its next use and meets the expiry dialog, exactly as for any ended session. An accepted Run is never affected.

### 1.2 Objective

Let a person end their own session deliberately, see every session their identity holds, end any one of them, or end all of them at once — while:

- never claiming a sign-out or a revocation that the server has not confirmed, and never leaving a credential usable after a confirmed one;
- never discarding unsaved work in the application shell without asking, and never saving it under a context other than the one it was written in;
- never ending a session the signed-in identity does not own, and never letting a tenant administrator see or end another person's sessions;
- never disclosing about a session more than a person needs to recognise it;
- never cancelling, duplicating or replaying an accepted Run, and never touching the identity's password, factor, recovery codes, memberships or pending requirements;
- never deriving authority from a session, and never granting a factor bypass to a listed or revoked session.

### 1.3 Why it matters

A sign-out that only clears the browser is theatre: the refresh credential left behind is a standing door, and a person who signed out on a shared computer believes it is closed. The sessions list is the only way a person can discover a session they did not start and end it without changing their password, and "sign out everywhere" is the fastest answer to a lost device. The opposite failures matter as much: a sign-out that pretends to succeed on a dead network, a sign-out that silently loses an hour of authoring, or a revocation that cancels a Run already accepted by the platform. The feature has to be as honest about what the server confirmed as the session itself is about whether it exists.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Decisions that depend on an architecture or design decision are also recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row. Exact location, all written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14, summary table and entry paragraphs: **A-51** (the session record fields shown in the list and the scope of "Sign out everywhere") is new; a note is appended to **A-39** (revocation scope). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-43** ("Sign out everywhere" and its confirmation on R38, now `427:4040` and D02 `422:3292`) and **S-44** (content of the sessions table rows) are new; notes are appended to **S-07** (app-shell sign-out), **S-41** (retry status variant for an unconfirmed sign-out) and **S-02** (reflow of the Sessions area).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | Sign-out from every designed entry and from the application shell, what a sign-out does and what it never does, unsaved work and in-flight requests at sign-out, the unconfirmed sign-out, other tabs and devices, the sessions list, individual revocation, and "Sign out everywhere". FR-IDN-068 requires "Sign out everywhere"; it was not drawn on R38 at the time (now `427:4040`) and the constitution's F-IDN-09 limited FR-IDN-068 to logout, listing and individual revocation, while specification 006 (Q1, RF-2.5, LC-20) already assigns it to this feature. It is **included**: constitution v1.3 (27 October 2026) extends F-IDN-09 to the whole of FR-IDN-068, as v1.2 extended F-IDN-07. Register S-43. Session lifetime, refresh, expiry, R34 and re-authentication stay in 006; the forced change in 004; recovery in 005; the guarded switch in 016. |
| Q2 | What "Sign out everywhere" does | It ends **every** session and refresh family of the identity, the current one included, voids every open login transaction of the identity and advances the session / security version (specification 006 RF-2.5); the person returns to the sign-in screen and every other session meets R34 at its next use. The event is audited; **no email notification** is sent, because the person acted themselves. This differs on purpose from a factor replacement, which keeps the performing session (A-39). Register A-51. |
| Q3 | Sign-out from inside a workspace | At the time of the decision the application shell had no sign-out, user menu or tenant switcher (S-07): sign-out was drawn only on R08, N03, N20, M03 and R34. This specification adds a **"Sign out" action in the identity area of the application shell** — the persona block of the navigation footer on every desktop app-shell screen, and the M04 menu on mobile — now drawn in the user menu (`395:668`) and the mobile menu (`395:669`); the header WORKSPACE selector (`395:667`) is the same guarded route to the chooser that specification 016 Q1 states. D03 ledger S-07. |
| Q4 | Unsaved work at sign-out | From the application shell, the **dirty guard** of the guarded switch (F-IDN-18, N24: Save / Discard / Stay) runs before the sign-out; a guard save is submitted under the current tenant and project with the still-valid authority, and sign-out proceeds only after Save is acknowledged or Discard is chosen. R34 discards its preserved draft without a guard (specification 006 RF-7.5); R08, N03, N20 and M03 hold no draft. |
| Q5 | Sign-out the server cannot confirm | Nothing is cleared until the server confirms. When the sign-out request ends without a server answer (network loss, timeout, service unavailable), the person **stays signed in** and sees the retry / offline status of specification 006 RF-5.4 with "Retry sign out", stating that the sign-out is not complete. The product never claims a sign-out that did not happen. Note appended to S-41. |
| Q6 | Other tabs of the same browser after a sign-out | They treat it **as any ended session**: on their next eligible request they receive the definitive answer, mask their content, clear transient secrets, close their streams and show R34 with their unsaved work kept in memory (specification 006 RF-7, RF-9). The dialog never says why the session ended. |
| Q7 | What the sessions list shows | **Every live session of the identity across all tenants**, newest sign-in first, paginated with a configured page size. *Session:* the browser and platform family recorded at sign-in — never the raw user-agent string — and the sign-in time. *Context:* the tenant and project in which the session last acted, or "Identity session" when it has entered none. *Expiry:* the server-supplied time at which the session ends if unused (the nearer of its deadlines), for the current row too. *Action:* "Current session" or "Revoke session". No IP address, no location. Registers S-44 and A-51. |
| Q8 | Confirmation and step-up before acting | **"Revoke session" acts at once**: no dialog, no step-up — the session itself proves the identity and the revoked device simply re-authenticates; the row shows the result in text. **"Sign out everywhere" asks one confirmation**, because it also ends the current session, and needs no step-up. Both are audited; neither sends an email. |

Five rules below are derived from these decisions and from earlier specifications rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** A sign-out lands on the sign-in screen with an empty form and no message, as "Sign out safely" on R08 (specification 004 RF-8.12) and "sign out safely" on R34 (specification 006 RF-7.5) already do; whether a confirmation line is shown there is left to the design (S-35 pattern) and nothing in this feature depends on it.
- **D-b** Before asking the server to end the session, the signing-out tab aborts its in-flight requests and disconnects its realtime streams, as steps (2) of the guarded switch do; a consequential write whose acknowledgement was lost is never resubmitted and is looked up after the next sign-in (FR-X-022).
- **D-c** The current session is never revocable from the list — R38 shows "Current session" with no action on its row — and signing out is the way to end it.
- **D-d** "Sign out safely" on R34 acts on a session the server has already definitively ended (that is why the dialog opened): it discards the preserved context and draft and returns to the sign-in screen without waiting for a server answer, after asking the server to end whatever remains of the family; decision Q5 governs only a live session.
- **D-e** The sessions list is read when the area is opened and again after each action of this feature; it is never polled. Opening it is a deliberate navigation and counts as human activity under specification 006 Q5.

Register items this specification relies on: **A-51**, **S-43** and **S-44** (decided here); **S-07** (the app-shell user menu `395:668`; the "Sign out" action is built per Q3), **S-41** (the retry / offline status, D02 `424:3581`; its "Retry sign out" variant is built per Q5), **S-02** and **S-28** (the Sessions area of R38 follows the D01 responsive policy with the one-record-per-session form of D02 `422:3292`, RF-12.1), and the pre-existing **A-11** (configuration values), **A-27** (identity statuses), **A-39** (revocation scope on replacement; a note records that Q2 differs on purpose), **A-49** (human activity), **A-50** (reuse detection; its notification names the sessions list) and **A-31** (closed by 006; re-authentication from R34 after a sign-out learned by another tab).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | person leaving a shared computer | to sign out and know that my session is ended on the server, not only cleared in this browser | nobody can continue as me from that machine |
| US-2 | person signing out with unsaved work | to be asked whether to save, discard or stay | I do not lose an hour of authoring to a reflex click |
| US-3 | person who sees a session I do not recognise | to review every session of my identity and end that one | I protect my account without changing my password |
| US-4 | person who lost a device | to sign out everywhere in one action and sign in again | whoever holds the device is out, now |
| US-5 | person on a weak network | to be told that my sign-out did not complete, with a retry | I am never shown a sign-in screen while my session is still alive |
| US-6 | person with several tabs open | my other tabs to protect their content and keep my unsaved work when I sign out in one of them | I can re-authenticate there and continue |
| US-7 | person running a test | my accepted Run to continue and finish when I sign out | execution facts are not lost because I left |
| US-8 | person on the tenant chooser, in a suspended tenant or on an invitation | to sign out from where I am | I am never trapped on a screen without a way out |
| US-9 | person working inside a workspace | a sign-out action in the shell on every screen, on desktop and on my phone | I do not have to find the tenant chooser to leave |
| US-10 | privacy-conscious person | the list to show only what I need to recognise a session — browser, time, workspace — and never an address or a location | reviewing my sessions does not itself expose me |
| US-11 | attacker or over-reaching party (negative story) | — | I cannot use a credential of a signed-out or revoked session, cannot learn whether a session exists, cannot revoke another identity's session, cannot sign another identity out everywhere, cannot make a sign-out fail silently, and cannot cancel a Run by signing out |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *session* and *refresh family* have the meaning of specification 006 RF-1; *definitive answer* means a server answer stating that the session is or is not valid (006 RF-3.6); *the dialog* means R34; *the dirty guard* means the Save / Discard / Stay step of the guarded switch (F-IDN-18); *the list* means the table "Authenticated sessions" of the Sessions area of R38; *the identity* means the signed-in global identity.

### RF-1 · Sign-out entries

Sign-out is reachable from every place a signed-in person can be. *Why:* a person who cannot find the exit leaves the door open. *(Decision: Q3.)*

- RF-1.1 The system shall offer sign-out on the tenant chooser N03 ("Sign out"), on the suspended-tenant screen N20 ("Sign out", beside "Personal account security"), on the invitation screens M03 and N29 ("Switch identity safely", specification 008 RF-3.2), on the expiry dialog R34 ("sign out safely") and on the forced-change screen R08 ("Sign out safely"). *(FR-IDN-068, FR-IDN-082, FR-IDN-116; F-IDN-09; N03, N20, M03, R34, R08)*
- RF-1.2 The system shall present a "Sign out" action in the identity area of the application shell — the persona block of the navigation footer on every desktop app-shell screen, and the M04 menu on mobile — on every screen reached with a session; the action is drawn in the user menu (`395:668`) and the mobile menu (`395:669`). *(FR-IDN-068, FR-X-052; decision Q3; S-07)*
- RF-1.3 Sign-out shall be available whatever the identity's tenant, role, grants, enrollment state or tenant state: an Owner gated for enrollment, a Member without grants and an identity whose tenant is suspended can all sign out. *(FR-IDN-068; P6)*
- RF-1.4 Where a surface holds a limited identity state rather than a session — the forced-change screen R08, the challenge R05, the recovery screen N05 — leaving it voids that transaction exactly as specifications 004 (RF-8.12, "Sign out safely"), 001 (RF-5.4, abandoning the challenge) and 005 (leaving the recovery screen) define; R05 and N05 carry no "sign out" action; this feature adds nothing to those exits and never creates a session for them. *(FR-IDN-024, FR-IDN-045)*
- RF-1.5 A sign-out entry shall be a labelled action with a visible focus state and, on responsive surfaces, a touch target of at least 44 px; it shall never be the only content of a menu that requires a drag or hover to open. *(FR-X-042, FR-X-053)*

### RF-2 · What a sign-out does

A sign-out ends the session on the server first and in the browser second. *Why:* the browser can be cleared by anyone; only the server can make the credentials worthless. *(Decisions: Q4, Q5.)*

- RF-2.1 When sign-out is activated — after the dirty guard of RF-3 where it applies — the system shall end the session server-side: the session and its entire refresh family are revoked as one indivisible change, so that every later request carrying any of its credentials receives the definitive answer of specification 006 RF-2.6 and nothing is served on its strength. *(FR-IDN-068, FR-IDN-062, FR-IDN-061; P6)*
- RF-2.2 When the server confirms that the session has ended, the client shall discard from memory the access credential, the preserved tenant and project context, every draft, every server fact and every transient secret, close realtime subscriptions, stop polling, cause the browser to drop the refresh credential, and route to the sign-in screen. *(FR-IDN-060, FR-IDN-066, FR-X-034)*
- RF-2.3 The sign-in screen reached by a sign-out shall open with an empty form, as specification 001 RF-1 defines it; nothing of the ended session — address, tenant, draft — shall be carried into it. *(FR-IDN-066; D-a)*
- RF-2.4 A sign-out shall end only the session of the browser context that performed it; every other session of the identity continues unchanged. "Sign out everywhere" (RF-8) is the only action of this feature that ends more than one session. *(FR-IDN-068; decision Q2)*
- RF-2.5 A sign-out shall never change the identity's password, factor, recovery codes, memberships, grants, pending forced-change requirement or open recovery case, and shall never lift a suspension or close a case; the identity signs in afterwards exactly as it would have before. *(FR-IDN-043, FR-IDN-048; P6)*
- RF-2.6 A sign-out shall never cancel, duplicate, pause or replay an accepted Run (RF-10). *(FR-IDN-067)*
- RF-2.7 If the session has already ended when the sign-out is requested — a deadline passed, a revocation, a version change — then the server shall answer that the session is no longer valid, which is a definitive answer, and RF-2.2 shall apply as for a confirmed sign-out. *(FR-IDN-061; decision Q5)*
- RF-2.8 The sign-out request shall be defended against cross-site request forgery as every request that changes a session (specification 006 RF-1.4); if that defence refuses it, then the safe problem of specification 006 RF-5.5 is shown, the session is untouched and nothing is cleared. *(FR-IDN-064)*

### RF-3 · Unsaved work and in-flight work at sign-out

The product asks before it discards, and never saves into the wrong place. *Why:* a sign-out is the one exit a person takes in a hurry. *(Decision: Q4.)*

- RF-3.1 When sign-out is activated from the application shell while the screen holds unsaved work, the system shall run the dirty guard — Save / Discard / Stay — before anything else; "Stay" shall leave the session, the screen and the work untouched. *(FR-IDN-130; decision Q4; N24)*
- RF-3.2 When "Save" is chosen, the save shall be submitted under the tenant, project and revision context in which the work was written, with the still-valid authority and the operation's own idempotency context; sign-out shall proceed only after the save is acknowledged. *(FR-IDN-131, FR-X-020, FR-X-021)*
- RF-3.3 If the guard save fails, is refused or ends indeterminately, then the system shall return the person to the screen with the work intact, show the outcome as that operation defines it, and not perform the sign-out; an indeterminate save is looked up, never resubmitted. *(FR-X-022, FR-X-023)*
- RF-3.4 When "Discard" is chosen, the work shall be discarded and the sign-out shall proceed. *(decision Q4)*
- RF-3.5 "Sign out safely" on the dialog shall discard the preserved context and draft without a guard (specification 006 RF-7.5); R08, N03, N20, M03 and N29 hold no unsaved work and show no guard. *(decision Q4)*
- RF-3.6 When a sign-out proceeds, the client shall abort its in-flight requests and disconnect its realtime streams before asking the server to end the session; a consequential write whose acknowledgement was lost shall never be resubmitted and shall be looked up after the next sign-in. *(FR-X-021, FR-X-022; D-b)*
- RF-3.7 The guard, the save and the sign-out shall never retarget the work to another tenant or project, and the sign-out shall never be used by any feature as a way to discard work silently. *(FR-IDN-131, FR-X-021)*

### RF-4 · The unconfirmed sign-out

Only the server can say a session ended. *Why:* a sign-in screen shown over a live session is a lie the person acts on by walking away. *(Decision: Q5.)*

- RF-4.1 Every sign-out shall end either definitively — the server confirms that the session ended, or answers that it had already ended (RF-2.7) — or indeterminately, when no server answer reached the client; the client shall treat only a definitive answer as a completed sign-out. *(FR-IDN-064; decision Q5)*
- RF-4.2 When a sign-out ends indeterminately, the system shall keep the person signed in with nothing cleared and shall show the retry / offline status of specification 006 RF-5.4, stating that the sign-out is not complete, with "Retry sign out", which repeats the sign-out request once per activation; the status shall claim neither that the person is signed out nor that the session expired, and shall never retry on its own. *(FR-IDN-064; decision Q5; S-41)*
- RF-4.3 While the status is shown, the person may dismiss it and continue working; the session is intact, authority is derived as before, and the streams disconnected under RF-3.6 are reopened only once a request succeeds under the still-valid session. *(FR-IDN-064)*
- RF-4.4 When a retried sign-out is answered that the session is no longer valid because the first request did reach the server, the answer is definitive and RF-2.2 applies. *(RF-2.7)*
- RF-4.5 Work saved under RF-3.2 before an indeterminate sign-out shall remain saved; an indeterminate sign-out shall never undo, repeat or re-ask the save. *(FR-X-020)*

### RF-5 · Other tabs and other devices

A sign-out is learned by everyone who shared the session, and by no one else. *Why:* the server is the single source of truth about a session, and the dialog is the single honest way to say it ended. *(Decision: Q6.)*

- RF-5.1 When a session is signed out in one tab, every other tab of the same browser context shall learn it as an ended session: on its next eligible request it receives the definitive answer and applies specification 006 RF-7.1 — masks or unmounts restricted content, clears transient secrets, closes its streams, stops polling — and presents the dialog with its unsaved work kept in memory under 006 RF-9.2. *(FR-IDN-066; decision Q6; 006 RF-7.8)*
- RF-5.2 The dialog shall not say that the session was signed out (specification 006 RF-7.3); a re-authentication from it establishes a new session for the same identity and returns the tab to its preserved context under 006 RF-8; "sign out safely" from it discards that context under RF-3.5. *(FR-IDN-066, FR-IDN-067)*
- RF-5.3 Sessions held by other browsers or devices shall be unaffected by a sign-out (RF-2.4); a session ended by individual revocation (RF-7) or by "Sign out everywhere" (RF-8) shall learn it at its next use in the same way as RF-5.1 (specification 006 LC-20, LC-30). *(FR-IDN-068)*
- RF-5.4 A tab that performed the sign-out shall never show the dialog for that sign-out; it routes to the sign-in screen under RF-2.2. *(decision Q6)*

### RF-6 · The sessions list

The person sees every session of their identity, and only theirs, with what they need to recognise each one. *Why:* a session that cannot be seen cannot be ended. *(Decision: Q7.)*

- RF-6.1 The system shall present, in the Sessions area of Security Settings (R38), the heading "Authenticated sessions" and a table with the columns Session, Context, Expiry and Action, listing every live session of the identity across all its tenants, ordered by sign-in time with the newest first, with the record count and "Previous · Next" pagination. *(FR-SET-020, FR-SET-023, FR-IDN-068; R38; decision Q7)*
- RF-6.2 The Session cell shall show the browser and platform family recorded when the session was established — never the raw user-agent string — and the time the session was established; the Context cell shall show the tenant and project in which the session last acted, or "Identity session" when it has entered none; the Expiry cell shall show the server-supplied time at which the session ends if it is not used again (the nearer of its idle and absolute deadlines), for the current row as well; the Action cell shall show "Current session" on the row of the session viewing the list and "Revoke session" on every other row. *(FR-SET-023, FR-IDN-061; decision Q7; S-44; A-51)*
- RF-6.3 No row shall show an IP address, a location, a network, a credential, a family state, a session / security version, a deadline the server did not supply, or any content of the work the session did. *(FR-X-030, FR-X-033; decision Q7)*
- RF-6.4 The list shall contain only sessions of the identity viewing it; no identity's sessions shall ever appear in another identity's list, and no tenant Owner or Admin shall see or act on another member's sessions through any surface. *(FR-IDN-049, FR-SET-034, FR-X-001, FR-X-002)*
- RF-6.5 When the identity no longer holds the membership under which a session last acted, the Context cell shall show "Identity session"; no name of a workspace the identity cannot currently see shall be shown. *(FR-IDN-110; P3)*
- RF-6.6 The area shall state: "Sessions are not trusted devices. No session grants a TOTP bypass. Expiring a session does not cancel an accepted run." *(FR-SET-024, FR-IDN-069, FR-IDN-067; R38)*
- RF-6.7 The page size shall be validated configuration; "Previous" and "Next" shall be unavailable, with that state conveyed in text, when no page lies in that direction; a request for a page beyond the last shall show the last page. *(FR-X-046, FR-X-042; P13)*
- RF-6.8 The list shall be read when the area is opened and again after each action of RF-7 and RF-8; it shall never be polled. A session that ended since the last read shall be absent from the next one. *(FR-IDN-061; D-e)*
- RF-6.9 While the list is shown and the viewing session itself ends — a deadline, a revocation from another device, a version change — the next request learns the definitive answer and specification 006 RF-7 applies; the list shall never be shown on the strength of an ended session. *(FR-IDN-061; P6)*
- RF-6.10 The list shall never offer to revoke the current session (D-c), never offer to extend a session, never show or change a deadline, and never present a session as remembered or trusted. *(FR-IDN-069, FR-SET-024)*

### RF-7 · Individual revocation

Ending another session of one's own is immediate and needs no second proof. *Why:* the person holding this session has already proven the identity, and the revoked device loses nothing but the need to sign in again. *(Decision: Q8.)*

- RF-7.1 When "Revoke session" is activated on a row, the system shall, at once and without a confirmation dialog or a step-up, end that session and its entire refresh family server-side exactly as RF-2.1, and record the event (RF-11.1). *(FR-IDN-068, FR-SET-023; decision Q8)*
- RF-7.2 The system shall revoke only a session that belongs to the identity performing the action; a request naming any other session, or a session that does not exist, shall be refused with one safe problem that reveals neither which case applies nor whether the session exists. *(FR-X-002, FR-X-033; P3)*
- RF-7.3 When the server confirms the revocation, the row shall show the result in text ("Session revoked") until the list is read again, and the person shall remain in their current session with authority unchanged. *(FR-X-042)*
- RF-7.4 If the session had already ended, then the revocation shall be answered as complete and the row shall show the same result. If the request ends indeterminately, then the row shall show the retry / offline status of specification 006 RF-5.4 and the action may be repeated; the row shall never show "revoked" without a definitive answer. *(FR-IDN-064; decision Q5 by analogy)*
- RF-7.5 The revoked session shall learn its end at its next use and meet the dialog (RF-5.3); its unsaved work follows specification 006 RF-9; its accepted Runs continue (RF-10); no notification shall be sent for a revocation (specification 006 LC-30). *(FR-IDN-066, FR-IDN-067; decision Q8)*
- RF-7.6 The current session shall never be revocable from the list; "Current session" is its only action and sign-out is the way to end it. *(D-c; R38)*
- RF-7.7 A revocation shall never be performed by a tenant administrator on another identity, by any feature as a side effect other than those specifications 003, 004, 005 and 006 already define, or by any surface other than the list and "Sign out everywhere". *(FR-IDN-049)*

### RF-8 · Sign out everywhere

One action that ends every session of the identity, this one included. *Why:* when a device is lost the person wants certainty, not a list to work through. *(Decisions: Q1, Q2, Q8.)*

- RF-8.1 The Sessions area shall offer the action "Sign out everywhere" beside the list; it is drawn beside the list (`427:4040`) with its confirmation dialog in D02 `422:3292` (S-43). *(FR-IDN-068; decision Q1; S-43)*
- RF-8.2 When "Sign out everywhere" is activated, the system shall ask one confirmation stating that every session of the identity, including the current one, will be signed out, that accepted runs continue, and that the person will have to sign in again; the confirmation shall require no password or code. *(FR-SET-024; decision Q8)*
- RF-8.3 When the confirmation is given, the system shall, as one change: advance the identity's session / security version (specification 006 RF-2.5), end every session and refresh family of the identity including the current one, void every open login transaction of the identity, and record the event (RF-11.1); then the current browser shall apply RF-2.2 and route to the sign-in screen. *(FR-IDN-068, FR-IDN-061, FR-IDN-062; decision Q2; A-51)*
- RF-8.4 Every other session of the identity shall learn its end at its next use and meet the dialog (RF-5.3; specification 006 LC-20); its unsaved work follows specification 006 RF-9; no email notification shall be sent. *(FR-IDN-066; decision Q2)*
- RF-8.5 "Sign out everywhere" is activated from the Sessions area, which holds no unsaved work, so no dirty guard applies; unsaved work in other tabs of the same browser is handled as RF-5.1. *(decision Q4, Q6)*
- RF-8.6 If the request ends indeterminately, then RF-4.2 applies with "Retry sign out everywhere" and nothing is cleared; when a retry is answered that the session is no longer valid because the first request did reach the server, the answer is definitive and RF-2.2 applies. *(FR-IDN-064; decision Q5)*
- RF-8.7 "Sign out everywhere" shall never change the password, the factor, the recovery codes, memberships, grants, a pending requirement or an open case; it is not a compromise response, blocks no queued work and requests no stop of any Run. *(FR-IDN-067, FR-IDN-043; P6)*
- RF-8.8 "Sign out everywhere" shall be performed only by the identity on itself; no tenant administrator and no other feature shall trigger it for another identity. *(FR-IDN-049)*

### RF-9 · Sign-out from the public and gated screens

Each designed entry keeps its screen's own promise. *Why:* the chooser, the suspended screen and the invitation each stand outside a tenant, and signing out there must change nothing about the tenant or the invitation.

- RF-9.1 "Sign out" on the tenant chooser N03 shall perform RF-2 whether or not the session has entered a tenant — the chooser may be opened from inside a workspace (specification 013 RF-1.4), in which case the shell guard already ran before it opened (013 RF-1.5); nothing about any membership changes. *(FR-IDN-110; N03)*
- RF-9.2 "Sign out" on the suspended-tenant screen N20 shall perform RF-2; the suspension, the membership and the frozen work are untouched; "Personal account security" remains the separate route specification 014 defines. *(FR-IDN-116; N20)*
- RF-9.3 "Switch identity safely" on M03 and N29 shall perform RF-2 and route to the sign-in screen; the invitation shall remain unchanged and unaccepted; whether the person returns to the invitation after the next sign-in belongs to specification 008. *(FR-IDN-082; M03, N29)*
- RF-9.4 "sign out safely" on the dialog shall ask the server to end whatever remains of the ended session's family, discard the preserved context and draft, and route to the sign-in screen without waiting for the answer, because the dialog opens only on a definitive answer that the session has already ended. *(specification 006 RF-7.5; D-d)*
- RF-9.5 "Sign out safely" on R08 is specification 004 RF-8.12: it voids the login transaction and the limited identity state and leaves the requirement pending; no session exists to end. *(FR-IDN-045; 004 RF-8.12)*

### RF-10 · Accepted Runs

An accepted Run belongs to the platform, not to the session that requested it. *Why:* execution facts are evidence, and a sign-out is not a cancellation.

- RF-10.1 Sign-out, individual revocation and "Sign out everywhere" shall never cancel, duplicate, pause or replay an accepted Run; it continues under its own run-scoped authority and records its facts. *(FR-IDN-067, FR-RUN-050)*
- RF-10.2 A Run request that was pending or uncertain when its session was signed out or revoked shall be looked up after the next sign-in, never resubmitted. *(FR-X-022; specification 006 RF-10.2)*
- RF-10.3 The Sessions area (RF-6.6) and the confirmation of RF-8.2 shall state that expiring a session does not cancel an accepted run. *(FR-SET-024)*

### RF-11 · Audit, secrecy and safe problems

Every deliberate end of a session is recorded, and no credential or session detail leaves its place. *Why:* the audit log is where the reason for an ended session lives (specification 006 RF-7.3).

- RF-11.1 The system shall append to the audit log: each sign-out, with the entry used (shell, N03, N20, M03 / N29, R34); each individual revocation, with the acting session and the revoked session; each "Sign out everywhere", with the number of sessions ended; each refused revocation (RF-7.2); each sign-out or revocation request that the forgery defence refused. Each entry identifies the identity, the time and the calling context, with a correlation id, and complements the session-ended entries of specification 006 RF-11.1. *(FR-X-044)*
- RF-11.2 No audit entry, log line, problem detail, URL, export or analytics event shall contain an access credential, a refresh credential, a raw user-agent string, a password or an authenticator code. *(FR-X-030, FR-X-033)*
- RF-11.3 Every refusal of this feature shall be a safe problem with a correlation id, distinguishing at least: session no longer valid, request could not be validated (specification 006 RF-5.5), and the single generic refusal of RF-7.2; no problem shall disclose deadlines, versions, family state or whether a session exists. *(FR-X-033)*
- RF-11.4 Every write of this feature shall confirm persistence before reporting success; a sign-out, a revocation or a "Sign out everywhere" shall be observable as either done or not done, never half-done. *(FR-X-020)*
- RF-11.5 This feature shall send no email notification; the sessions list is the place a person reviews after the notification of specification 006 RF-11.4 or of a reset, replacement or compromise response. *(decisions Q2, Q8; A-30)*

### RF-12 · Responsive and accessible surfaces

Signing out and reviewing sessions are critical journeys. *Why:* everyone meets them, on every viewport.

- RF-12.1 Where no designed mobile or tablet counterpart exists for the Sessions area, the confirmation of RF-8.2 or the retry status variant of RF-4.2, the system shall present them reflowed under the desktop rules, converting the table to one record per session with every cell, statement and action intact. *(FR-X-055; S-02, S-28)*
- RF-12.2 The app-shell "Sign out" shall appear in the M04 menu on mobile, following the current server-derived grants for every other destination and changing no authority. *(FR-X-052; decision Q3)*
- RF-12.3 Every action of this feature shall have a visible label, a visible focus state and, on responsive surfaces, a touch target of at least 44 px; the accessible name of "Revoke session" shall include the row's session description; the result of a revocation, the retry status and the pagination state shall be conveyed by text; when the confirmation of RF-8.2 or the dirty guard opens, focus shall enter its heading or first control and return to the invoking action when it closes; a live announcement shall never steal focus. *(FR-X-040, FR-X-042, FR-X-053)*
- RF-12.4 All states and messages of this feature shall exist in EN / ES / FR; the interface language shall never change an expiry time, a session record or the outcome of an action. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Server-side truth.** A session is ended by a sign-out or a revocation only when the server has ended it; the client never treats a session as ended without a definitive answer. | FR-IDN-068, FR-IDN-064; decision Q5 |
| RNF-2 | **Atomic end.** Ending a session and its refresh family, and "Sign out everywhere" across every session of the identity, is one indivisible change; no observer sees a session ended while a credential of its family is still accepted. | FR-IDN-062; decision Q2 |
| RNF-3 | **Revocation latency.** From the moment a session ends on the server, every request carrying its credentials is refused. | FR-IDN-061, FR-IDN-062 |
| RNF-4 | **No authority from a session.** Listing, revoking and signing out grant no tenant or project authority and never bypass a factor. | FR-IDN-024, FR-IDN-069; P3 |
| RNF-5 | **Own sessions only.** Every read and every action of this feature is bound to the identity performing it; a tenant administrator holds no session action on another identity. | FR-IDN-049, FR-X-001 |
| RNF-6 | **Fail closed.** An absent, ended or unknown session blocks every dependent action; an indeterminate outcome is shown as such and nothing is assumed. | P6; FR-IDN-064 |
| RNF-7 | **No browser persistence.** No credential, draft, session record or secret of this feature is written to browser storage; a sign-out leaves nothing in memory. | FR-IDN-060, FR-X-034 |
| RNF-8 | **Server-owned time.** Expiry times shown in the list are supplied by the server; the client never computes one. | FR-IDN-061; R34 |
| RNF-9 | **Configured limits.** The page size of the list and any bound on repeated sign-out attempts are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-10 | **No lost write.** A sign-out never discards work without the dirty guard, never saves it elsewhere than where it was written, and never resubmits a write whose acknowledgement was lost. | FR-IDN-130, FR-IDN-131, FR-X-021, FR-X-022 |
| RNF-11 | **Run continuity.** An accepted Run is unaffected by any action of this feature. | FR-IDN-067, FR-RUN-050 |
| RNF-12 | **Minimal disclosure.** A session is described by browser and platform family, sign-in time, last context and server-supplied expiry, and by nothing else. | FR-X-030; decision Q7 |
| RNF-13 | **Accessibility.** The Sessions area, the confirmation, the dirty guard and the retry status meet WCAG 2.2 AA with automated and manual assistive evidence. | FR-X-040, FR-X-042 |
| RNF-14 | **Language.** EN / ES / FR, display only. | FR-X-043 |
| RNF-15 | **Reflow.** Mobile and tablet reflows preserve every cell, statement and action. | FR-X-055 |
| RNF-16 | **Verification.** Server-side revocation on sign-out, multi-tab behaviour after a sign-out, identity-scoped listing and revocation, "Sign out everywhere" across devices, and active Runs surviving every action are verified as areas A and F require. | requirements A.12, F; P9 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Sign-out from the shell with unsaved work; "Stay" | Nothing changes; the person is back on the screen with the work (RF-3.1). |
| LC-2 | Sign-out from the shell with unsaved work; "Save"; the save succeeds | Saved under the original context; session ended; sign-in screen (RF-3.2, RF-2.2). |
| LC-3 | "Save" refused because the grant changed meanwhile | Back on the screen with the work intact; the denied-scope state of 006 RF-5.1 applies; no sign-out (RF-3.3). |
| LC-4 | "Save" ends with no server answer | Back on the screen; the submission is looked up, never resubmitted; no sign-out (RF-3.3). |
| LC-5 | Sign-out from the shell with no unsaved work | No guard; in-flight requests aborted; streams closed; session ended; sign-in screen (RF-3.6, RF-2). |
| LC-6 | Sign-out request ends with no server answer | Person stays signed in; retry / offline status "sign-out not complete" with "Retry sign out"; nothing cleared (RF-4.2). |
| LC-7 | Retry after LC-6 finds the session already ended | Definitive answer; clear and sign-in screen (RF-4.4). |
| LC-8 | Sign-out requested after the idle deadline passed | Server answers "no longer valid"; treated as confirmed; sign-in screen (RF-2.7). |
| LC-9 | Forgery defence refuses the sign-out request | Safe problem "could not be validated, reload and try again"; session untouched (RF-2.8). |
| LC-10 | Sign-out in one of three tabs | The signing-out tab reaches the sign-in screen; the other two meet R34 at their next request with their drafts kept in memory (RF-5.1, RF-5.4). |
| LC-11 | Re-authentication in a tab that met R34 after LC-10 | New session for the same identity; preserved context restored (RF-5.2; 006 RF-8). |
| LC-12 | Sign-out on N03, N20, M03 or N29 | Session ended; sign-in screen; membership, suspension and invitation unchanged (RF-9.1 … 9.3). |
| LC-13 | "sign out safely" on R34 with no network | Context and draft discarded; sign-in screen without waiting; the session was already ended (RF-9.4; D-d). |
| LC-14 | Sessions list opened on a phone | One record per session with every cell and action; "Revoke session" at 44 px (RF-12.1, RF-12.3). |
| LC-15 | Identity holds sessions in three tenants | All listed, newest first; context shows each session's last tenant / project (RF-6.1, RF-6.2). |
| LC-16 | A session last acted in a tenant the identity has since left | Context shows "Identity session" (RF-6.5). |
| LC-17 | "Revoke session" on another device's session | Ends at once; row reads "Session revoked"; that device meets R34 at its next use; no email (RF-7.1, RF-7.3, RF-7.5). |
| LC-18 | "Revoke session" on a session that expired a moment earlier | Answered as complete; same row result (RF-7.4). |
| LC-19 | Revocation request names a session of another identity | One generic refusal; nothing revealed; audited (RF-7.2, RF-11.1). |
| LC-20 | Revocation request ends with no server answer | Row shows the retry status; action repeatable; never "revoked" without an answer (RF-7.4). |
| LC-21 | Revoked session was mid-edit with a draft | That tab meets R34 at its next request; draft kept in memory; reappears only under 006 RF-9.3 (RF-7.5). |
| LC-22 | Revoked session had an accepted Run executing | The Run continues and finishes; re-authentication shows its facts (RF-10.1). |
| LC-23 | "Sign out everywhere" confirmed from one of three devices | Version advanced; all three sessions ended; this browser to the sign-in screen; the other two meet R34 at next use; no email (RF-8.3, RF-8.4). |
| LC-24 | "Sign out everywhere" request ends with no server answer | Nothing cleared; "Retry sign out everywhere"; a retry answered "no longer valid" is definitive (RF-8.6). |
| LC-25 | "Sign out everywhere" while a Run request is pending | Pending request looked up after the next sign-in; never resubmitted (RF-10.2). |
| LC-26 | "Sign out everywhere" by an identity with a pending forced change | Sessions ended; requirement still pending; next sign-in meets R08 (RF-8.7; 004 Q2). |
| LC-27 | Viewing session ends while the list is open | Next request learns it; R34; the list is never served on an ended session (RF-6.9). |
| LC-28 | "Next" activated on the last page | Unavailable, stated in text; a direct request beyond the last page shows the last page (RF-6.7). |
| LC-29 | Language switched while the list is open | Display changes; expiry times and records unchanged (RF-12.4). |
| LC-30 | Identity becomes restricted while signed in | Sessions end at next use under 006 RF-2.5; a sign-out of an already-ended session is answered as complete (RF-2.7; A-27). |

---

## 6. Out of scope

- **Session lifetime, deadlines, the session / security version, refresh and family rotation, the expiry dialog R34, re-authentication, the retry / offline status itself** — F-IDN-08, specification 006; this feature reuses the definitive answer, the dialog and the status without redefining them.
- **"Sign out safely" on R08** — specification 004 RF-8.12; a limited identity state, not a session.
- **Abandoning the challenge or a recovery transaction** — specifications 001 and 005.
- **The sessions revoked by a reset, a forced change, a compromise response, a factor replacement or a recovery** — specifications 003, 004, 005; this feature only lists what remains.
- **The guarded tenant and project switch**, its dirty guard, scope epoch and dialog N24 — F-IDN-18, specification 016; this feature invokes the guard and defines nothing of it.
- **The return to an invitation after signing out and signing in again** — specification 008.
- **The user menu, tenant switcher and mobile navigation of the application shell** — shell families (`395:668`, `395:667`, `395:669`; S-07); this feature places only its "Sign out" action in the user menu.
- **Operator-side session tooling** — platform-wide revocation, session search, revocation of another identity's sessions: not designed and, for tenant administrators, forbidden (FR-IDN-049).
- **Trusted devices, remembered sessions, "keep me signed in", a device inventory with locations** — product boundary (FR-IDN-069) and decision Q7.
- **Notifications for sign-out, revocation or "Sign out everywhere"** — decided out (Q2, Q8).
- **Single sign-on, federated sign-out, native-app sessions** — deferred (FR-EXT-020).
- **Tablet-specific designs** of the Sessions area — none; the one-record-per-session form of D02 `422:3292` and the D01 responsive policy apply (S-02, S-28).

---

## 7. Criteria of completion

Logout and session management are complete when all of the following hold:

1. Every user story US-1 … US-10 can be walked end to end on the desktop design and in the reflows with no step outside this specification, and US-11 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-12 has at least one verification that fails before and passes after the feature exists.
3. Entry tests show sign-out on N03, N20, M03 / N29, R34 and in the application shell on every app-shell screen, desktop and M04 mobile menu, for every tenant, role, grant and enrollment state (RF-1, LC-12).
4. Revocation tests show that a sign-out ends the session and its whole refresh family server-side as one change, that every later request with its credentials is refused, that other sessions of the identity are untouched, and that the browser holds nothing afterwards (RF-2, RNF-2, RNF-3, RNF-7, LC-5, LC-8).
5. Guard tests show Save / Discard / Stay before a shell sign-out with unsaved work, the save under the original context, no sign-out on a failed or indeterminate save, and no guard on R34, N03, N20, M03 (RF-3, RNF-10, LC-1 … LC-4).
6. Confirmation tests show that an indeterminate sign-out, revocation or "Sign out everywhere" clears nothing and shows the retry status, that a retry answered "no longer valid" completes, and that the forgery-defence refusal leaves the session intact (RF-4, RF-7.4, RF-8.6, RNF-1, RNF-6, LC-6, LC-7, LC-9, LC-20, LC-24).
7. Multi-tab and multi-device tests show that other tabs meet R34 with drafts kept, that the signing-out tab never does, and that revoked and signed-out-everywhere sessions meet R34 at next use (RF-5, LC-10, LC-11, LC-17, LC-23).
8. List tests show every live session of the identity across tenants, newest first, the four cells as decided, "Current session" on the viewing row, no IP, location or raw user-agent, "Identity session" for a lost membership, identity-scoped visibility, configured pagination and on-demand reading (RF-6, RNF-5, RNF-8, RNF-9, RNF-12, LC-14 … LC-16, LC-27, LC-28).
9. Action tests show immediate revocation without dialog or step-up, idempotent revocation of an ended session, one generic refusal for a foreign or unknown session, the single confirmation of "Sign out everywhere", the version advance, the end of every session including the current one, the voided login transactions, and no email for either action (RF-7, RF-8, LC-17 … LC-19, LC-23, LC-26).
10. Run tests show that an accepted Run survives sign-out, revocation and "Sign out everywhere", and that a pending Run request is looked up and never resubmitted (RF-10, RNF-11, LC-22, LC-25).
11. Audit entries exist for every event of RF-11.1 and contain no credential, raw user-agent, password or code; every refusal is a safe problem with a correlation id (RF-11).
12. The Sessions area, the confirmation, the dirty guard, the retry status and the shell action pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RF-12, RNF-13 … RNF-15).
13. The register items this specification relies on — A-51 (decided here), A-39 (note), A-11, A-27, A-49, A-50 and A-31 in the architecture register; S-43 and S-44 (decided here), S-07, S-41, S-02 and S-28 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
