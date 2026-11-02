# Specification 006 — Sessions, refresh and expiry

**Feature** Identity session lifetime, coordinated refresh, expiry and re-authentication · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-08 as the subject; F-IDN-03, F-IDN-04, F-IDN-05, F-IDN-06, F-IDN-07, F-IDN-09, F-IDN-14, F-IDN-15, F-IDN-16, F-IDN-18 and F-SET-03 where a criterion touches their boundary; F-X-01, F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P12, P13, P14) → `requirements.md` (FR-IDN-020, 021, 023, 024, 025, 026, 029, 042, 045, 048, 060, 061, 062, 063, 064, 065, 066, 067, 068, 069, 116, 122, 130, 131, 132; FR-SET-023, 024; FR-RUN-050, 055; FR-X-001, 002, 020, 021, 022, 023, 030, 033, 034, 040, 042, 043, 044, 045, 046, 053, 055) → the design of record, screen R34 SessionExpiryDialog `148:489`, the D02 atlas `159:32045` entries "Verify to continue protected work", "This project isn't available", "Your project grant changed" and "Stream interrupted · facts stale", N28 ProjectAccessDeniedPanel `159:32104`, the Sessions statements of R38 SecuritySettingsPage `148:3895`, the refresh flow of board 3 `46:7847` ("One coordinated refresh, one eligible retry, no surprise Run") and the FE board `46:7847`.

**Related specifications:** 001 Login (the sign-in that establishes the session; its challenge, abuse protection and landing rule are reused by re-authentication; its register item A-31 is answered here by decision Q2) · 003 TOTP enrollment, recovery codes and factor replacement (a completed replacement ends every other session, RF-9.7; the enrollment gate applies again after re-authentication; its LC-18 is this feature's RF-2.5) · 004 Password reset and forced password change (a reset and a compromise response end every session, RF-5.5 and RF-7.2; the forced change meets a re-authentication exactly as it meets a sign-in) · 005 Controlled account recovery (the "local account recovery" route of R34; a completed recovery ends every other session, RF-3.4) · 007 Logout and session management ("sign out safely", the sessions list, individual revocation and "sign out everywhere") · 012 Workspace creation and 013 Tenant chooser (where a re-authenticated identity goes when its interrupted context is no longer available) · 016 Guarded tenant and project switch (the dirty guard and scope epoch it owns, FR-IDN-130 … 132) · the execution and monitoring specifications (the rendering of a target's authentication failure and of stream loss, FR-RUN-050, FR-RUN-055).

This document states **what** sessions, refresh, expiry and re-authentication do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

A completed sign-in (specification 001) gives an identity a **session**: the product's memory, held on the server, that this browser is acting for this identity. A session carries **no tenant or project authority**; authority is derived on the server from the identity's current memberships at every request (P3). A session is not a trusted device and grants no factor bypass.

The browser holds a session as two credentials: a **short-lived access credential** that exists only in memory and proves the session on each request, and a **longer-lived refresh credential** that the browser keeps in a form page scripts cannot read and that is not sent on cross-site requests. When the access credential stops being accepted, the client performs **one coordinated refresh** for the whole browser: the server rotates the family atomically, detects any reuse of a retired credential, and the waiting requests are retried at most once, unchanged. Nothing of this is ever written to browser storage.

A session ends when its **idle deadline** or its **absolute deadline** passes, when the identity's **session / security version** moves on (a password change, a factor replacement, a compromise response, "sign out everywhere"), when it is revoked individually or by logout (specification 007), or when a retired refresh credential is presented again. Only requests caused by a deliberate human action move the idle deadline; polling, heartbeats and refreshes never do.

When the client learns **definitively** that the session is no longer valid, it masks restricted content, clears transient secrets and presents the dialog R34, "Reauthenticate to continue.", which preserves the tenant, project and unsaved context in memory, and offers re-authentication for the same identity. A transport failure is never presented as expiry. An **accepted Run keeps running** under its own narrow authority whatever happens to the browser; signing in again restores visibility only and never replays a Run request.

### 1.2 Objective

Keep an identity signed in exactly as long as the product's policy allows and no longer, let the browser recover from an expired access credential without the person noticing, and when a session has genuinely ended, protect what the person was doing while asking them to prove themselves again — while:

- never deriving authority from a session, a refresh or a re-authentication by itself;
- never accepting a replayed credential, never letting background activity extend a session, never letting a local clock decide expiry;
- never presenting a network failure, a target's authentication failure or a denied scope as expiry, and never looping through sign-in or refresh;
- never persisting a credential or a draft in the browser, and never showing one person's draft to another;
- never cancelling, duplicating or replaying an accepted Run because a browser session ended.

### 1.3 Why it matters

The session is the thread that ties every protected action to a verified person. A session that outlives its holder's attention, that can be extended by a polling tab, or whose refresh credential can be replayed from a stolen copy is a standing door into every tenant the identity belongs to. At the same time, an authoring tool that loses an hour of work because an access credential rotated, or that fires a Run twice because a retry rebuilt a request, is unusable. The feature has to be invisible when the session is healthy, honest when it is not, and harmless to work already accepted by the platform.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Decisions that depend on an architecture or design decision are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row (architecture `122:2` §14 as A-nn, D03 Screen decisions ledger `148:4` as S-nn).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | Session lifetime (idle and absolute deadlines, session / security version), refresh and family rotation with reuse detection, the coordinated client refresh and its single retry, the handling of denied scope, forgery defence failures and transport failures, the customer-target wall, the expiry dialog R34 and re-authentication, the protection of unsaved work, and the continuity of accepted Runs. Logout, the sessions list, individual revocation and "sign out everywhere" belong to 007; controlled recovery to 005 (only its route from R34 is here); the guarded tenant switch to 016. |
| Q2 | What "Reauthenticate" on R34 runs | The full sign-in proof of specification 001 — password and, whenever a factor is enrolled, the TOTP challenge, with Login's throttles, generic failures, attempt limit and transaction expiry — for the **identity of the ended session only**, performed from the dialog. On success the person returns to the interrupted tenant, project and screen with authority derived afresh; the landing-by-membership rule of 001 RF-8 does not apply. The ended session's earlier challenge never counts. Signing in as a different identity is done by "sign out safely" and the sign-in screen, which discards the preserved context and draft. Closes register A-31. |
| Q3 | When R34 opens; what "Retry session refresh" does | R34 opens only on a **definitive** server answer that the session is no longer valid (idle or absolute deadline passed, revoked, version stale, reuse detected). A refresh that ends without a definitive answer (network loss, timeout, service unavailable) shows an accessible retry / offline status that never claims expiry; "Retry session refresh" repeats the coordinated refresh once per activation. There is no proactive countdown dialog. Register S-41. |
| Q4 | Unsaved work around expiry | While the session is valid and the server-supplied remaining time is under a configured threshold, a non-blocking notice offers "Save now" for unsaved work, while authorisation still holds. After a definitive expiry the dirty work is kept in memory only, never in browser storage, so a reload loses it; it reappears only when the same identity re-authenticates into the same tenant and project with authority that still permits the edit; otherwise it is discarded. Register S-40. |
| Q5 | What counts as human activity for the idle deadline | Only a request caused by a deliberate action of the person — navigating, opening a record, searching, saving, issuing a command. Never: polling, a refetch on focus or reconnection, a realtime heartbeat or subscription, the refresh itself, input that produces no request, or a tab that is merely open. The absolute deadline never moves. Register A-49. |
| Q6 | Outcome of refresh-reuse detection | The whole refresh family and its session end at once, in every tab and device that shares it; the holder meets R34; the event is audited with its correlation id; a security notification goes to the verified email saying that a session was ended because its credential was presented twice and what to do if unexpected. Register A-50; template added to the S-14 inventory; note on A-30. |

Two rules below are derived from these decisions rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** A terminal failure of the re-authentication (attempt limit or expiry of its transaction, RF-8.6) voids that transaction and returns the dialog to its initial state with the context and draft still preserved; the throttles of 001 RF-4 bound how often this can happen. Abandoning is "sign out safely".
- **D-b** A page reloaded after its session ended holds no draft (RF-9.2); the first request learns the definitive answer, R34 is shown over the empty shell, and a successful re-authentication returns to the route that was requested when authority for it derives, otherwise to the tenant chooser.

Register items this specification relies on: **A-31** (closed by Q2), **A-49** and **A-50** (decided here), **S-39** (the re-authentication step of R34, D02 `424:3581`; built per Q2), **S-40** (the pre-expiry save notice, D02 `424:3581`; built per Q4), **S-41** (the retry / offline status, D02 `424:3581`; built per Q3), **S-42** (the forgery-defence failure state, D02 `424:3581`; built per RF-5.5), and the pre-existing **A-11** (configuration values), **A-27** (identity statuses), **A-30** (sign-in notifications; a note records the reuse notification), **A-39** (revocation scope), **A-45** (forced change as a sign-in gate), **S-02** (responsive coverage), **S-07** (app-shell sign-out) and **S-14** (email templates).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | person working in a workspace | my access to renew itself in the background while I am active | I am not interrupted by a credential I never see |
| US-2 | person who stepped away | my session to end after a period of inactivity and, regardless of activity, after a maximum time | an unattended browser does not stay signed in |
| US-3 | person whose session ended mid-edit | to see a clear dialog, keep my tenant, project and unsaved work, and prove myself again | I do not lose where I was or what I typed |
| US-4 | person who was about to lose the session | to be told in time that I should save | my work is saved while I still have the right to save it |
| US-5 | person with several tabs open | one re-authentication to be enough for all of them | I do not sign in once per tab |
| US-6 | person who changed my password or replaced my authenticator | every other session of mine to end at once | whoever held my old credential is signed out |
| US-7 | person whose project grant was removed | to be shown that the project is unavailable and offered permitted work | I am not stuck in a sign-in loop |
| US-8 | person on a weak network | a network failure to be shown as such, with a retry | I am not told my session expired when it did not |
| US-9 | person running a test | an accepted Run to continue and finish while my browser session expires | execution facts are not lost because I closed my laptop |
| US-10 | person authoring a scenario against a target that answers "unauthorised" | that answer to be shown as the target's response | the console never confuses my test's result with my own sign-in |
| US-11 | person on a phone | the expiry dialog and its states reflowed for my viewport with every statement and action intact | I can recover my session anywhere |
| US-12 | attacker or over-reaching party (negative story) | — | I cannot replay a stolen refresh credential, keep a session alive by polling, make a customer target's response refresh an ASpectra credential, read another person's draft, relaunch a Run by signing in again, or learn why a session ended from the dialog |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *session* means the server-held identity session of one browser context; *access credential* and *refresh credential* are the two client-side proofs of RF-1.2; *family* means the chain of refresh credentials issued for one session; *definitive answer* means a server answer stating that the session is or is not valid (RF-3.6); *re-authentication* means the proof of RF-8; *the dialog* means R34.

### RF-1 · The session and its credentials

A session is server truth about one browser acting for one identity, represented to the browser by two credentials that grant nothing by themselves. *Why:* authority is derived on the server at every request (P3); the credentials only identify the session that the derivation starts from.

- RF-1.1 When sign-in completes (specification 001 RF-8.1) or re-authentication succeeds (RF-8.3), the system shall establish one session for that browser context, bound to the identity, carrying no tenant or project authority; tenant and project authority shall be derived on the server at every request from the session and the identity's current memberships, never from a credential. *(FR-IDN-024, FR-X-001, FR-X-002; P3)*
- RF-1.2 The system shall represent a session to the browser as a short-lived access credential that exists only in memory and a longer-lived refresh credential that the browser holds in a form page scripts cannot read and does not send with cross-site requests. *(FR-IDN-060)*
- RF-1.3 The system shall never write an access credential, a refresh credential, a draft, a server fact or a secret of this feature to browser storage of any kind; everything the client keeps lives in memory and is lost on reload. *(FR-IDN-060, FR-X-034)*
- RF-1.4 The system shall defend every request that uses or changes a session against cross-site request forgery and cross-origin calls, configured against the product's actual origins; a request that fails these defences shall be refused as in RF-5.5. *(FR-IDN-060, FR-IDN-064)*
- RF-1.5 The system shall hold every session on the server with its identity, creation time, time of last human activity, idle and absolute deadlines, the session / security version it was issued under, and the context the sessions list of specification 007 displays; a session shall exist exactly as long as the server says it does. *(FR-IDN-061, FR-IDN-068, FR-SET-023)*
- RF-1.6 A session shall never be a trusted device, shall never grant a factor bypass, and shall never substitute for a proof another feature requires — the step-up of specification 003, the proofs of specification 005, the forced change of specification 004. *(FR-IDN-069, FR-SET-024; specification 003 RF-10.4, specification 005 RF-9.5)*

### RF-2 · Deadlines, activity and versions

A session lives only while a person is actually using it, never beyond a maximum, and never after the identity's security posture changed. *Why:* an unattended or stale session is the simplest takeover there is. *(Decision: Q5.)*

- RF-2.1 The system shall give every session a configured idle deadline and a configured absolute deadline, both enforced on the server; the product shall refuse to start without them and shall never substitute a default. *(FR-IDN-061, FR-X-046; P13)*
- RF-2.2 The system shall move the idle deadline forward only when it serves a request caused by a deliberate action of the person — navigating, opening a record, searching, saving, issuing a command — and only while the session is valid. *(FR-IDN-061; decision Q5; A-49)*
- RF-2.3 The system shall never count as human activity: background polling, a refetch triggered by focus or reconnection, a realtime heartbeat or subscription, the refresh itself, a tab that is merely open, or input that produces no request. *(FR-IDN-061; decision Q5)*
- RF-2.4 The absolute deadline shall never move; when it passes, the session ends whatever the person is doing, and only re-authentication (RF-8) starts a new session with fresh deadlines. *(FR-IDN-061; decision Q5)*
- RF-2.5 The system shall hold a current session / security version per identity and shall end, at its next use, every session issued under an older version; the version advances when the credential changes by reset or forced change (specification 004 RF-5.5, RF-8.9), when the factor is replaced (specification 003 RF-9.6, specification 005 RF-3.4), when a compromise response is applied (specification 004 RF-7.2), when "sign out everywhere" is used (specification 007), and when the identity becomes restricted or under erasure. The session that performs a replacement continues (specification 003 Q9, A-39). *(FR-IDN-061, FR-IDN-042, FR-IDN-029, FR-IDN-068, FR-IDN-122; specification 003 LC-18, specification 004 LC-19)*
- RF-2.6 When a session has ended for any reason — a deadline, a version change, individual revocation or logout (specification 007), reuse detection (RF-3.5) — the system shall answer every later request carrying its credentials with the definitive answer that the session is no longer valid, and shall serve nothing on the strength of an ended session. *(FR-IDN-061; P6)*
- RF-2.7 The system shall supply the client, with its answers, the time remaining before the nearer of the two deadlines; the client shall never derive expiry from a local countdown. *(R34 "Expiry and any remaining time are supplied by the server policy, not a local countdown.")*

### RF-3 · Refresh and family rotation

Renewing the access credential is a server-side rotation with memory, not a reissue. *Why:* a refresh credential that can be presented twice is a credential that can be stolen and used; detecting the second presentation is what makes theft visible. *(Decision: Q6.)*

- RF-3.1 When a valid refresh credential of a live family is presented before its session's deadlines, the system shall, as one indivisible change, issue a new access credential and a new refresh credential and retire the presented one; the credentials of one browser context shall form one family from sign-in to the session's end. *(FR-IDN-062)*
- RF-3.2 A refresh shall never move the idle deadline, never extend the absolute deadline, and never grant authority the identity does not currently hold. *(FR-IDN-061; decision Q5)*
- RF-3.3 If the session has ended for any reason of RF-2.6, if the presented credential belongs to no live family, or if the identity's status no longer permits sign-in (A-27), then the system shall refuse the refresh with the definitive answer of RF-2.6. *(FR-IDN-061, FR-IDN-048; P6)*
- RF-3.4 When concurrent refreshes of one family arrive — two tabs, or a duplicate from an ambiguous retry — the system shall honour exactly one and answer the others so that the client coalesces on the honoured result; it shall never accept an arbitrary recently retired credential as its way of tolerating concurrency. *(FR-IDN-062)*
- RF-3.5 If a retired refresh credential is presented, then the system shall treat it as reuse: end the entire family and its session at once for every tab and device sharing it, refuse the request with the definitive answer of RF-2.6, record the event (RF-11.1) and send the notification of RF-11.4. *(FR-IDN-062; decision Q6; A-50)*
- RF-3.6 Every refresh shall end either definitively — new credentials issued, or the session is no longer valid — or indeterminately, when no server answer reached the client; the client shall treat only a definitive answer as knowledge about the session. *(FR-IDN-064; decision Q3)*

### RF-4 · The coordinated client refresh

The browser renews once for everyone and retries each interrupted operation once, unchanged. *Why:* several tabs refreshing at once would trip reuse detection on their own credentials, and a rebuilt retry is how a Run gets launched twice.

- RF-4.1 When a request to the ASpectra control plane is answered that its access credential is no longer accepted (an *eligible* credential-rejected answer), the system shall run one coordinated refresh for the whole browser context; every concurrent eligible request shall wait for that one refresh instead of starting its own. *(FR-IDN-063; board 3)*
- RF-4.2 When the coordinated refresh succeeds, the system shall retry each waiting eligible operation at most once, with the identity, tenant, project, body and idempotency context it had before; it shall never reconstruct a different request, never retarget an operation to a newly selected tenant or project, and never retry an operation that already received a definitive answer. *(FR-IDN-063, FR-IDN-131, FR-X-021)*
- RF-4.3 The following shall never be eligible for refresh: an answer from a customer target (RF-6); an answer of the sign-in, challenge, recovery or re-authentication steps themselves, since a limited identity state is not a session; the refresh request itself; any answer other than a credential-rejected one. *(FR-IDN-063, FR-IDN-065, FR-IDN-024)*
- RF-4.4 When a retried operation is a consequential write, the retry shall carry its original idempotency key and body so that the server returns the recorded outcome of the first submission; if the acknowledgement is lost after the retry, then the client shall look the submission up, never resubmit; no queue of writes to replay after refresh or re-authentication shall exist. *(FR-X-020, FR-X-022, FR-X-023)*
- RF-4.5 When the coordinated refresh ends with the definitive answer that the session is no longer valid, the system shall treat the session as ended (RF-7); when it ends indeterminately, the system shall show the status of RF-5.4 and shall not claim expiry. *(FR-IDN-064; decision Q3)*
- RF-4.6 Safe reads may be retried within configured bounds; consequential writes shall never be retried automatically except as RF-4.2 and RF-4.4 state. *(FR-X-023)*

### RF-5 · Denied scope, forgery defence and transport failure

Three answers look like expiry and are not. *Why:* each needs its own honest state, and none may become a loop. *(Decision: Q3.)*

- RF-5.1 When a request is answered with an access denial for the current tenant or project, the system shall refresh the person's current permissions once and then either resume with the refreshed scope or show the denied scope; it shall never start a credential refresh, never route to sign-in, and never loop. *(FR-IDN-064; D02 "Your project grant changed"; N28)*
- RF-5.2 The denied-scope state shall use the designed copy — "This project isn't available" with "You do not have current permission, or the resource is unavailable. No project content is disclosed." — with the actions "Refresh permissions" and "Choose permitted project"; it shall disclose no project content and shall remove restricted data and realtime subscriptions from the screen. *(FR-IDN-064; N28 `159:32104`)*
- RF-5.3 When a grant changes under a screen that holds unsaved work, the system shall block new protected work, keep the draft as RF-9 states and offer "Refresh permissions" once; the draft is shown again only if the refreshed scope still permits the edit. *(FR-IDN-064, FR-IDN-066; D02 "Your project grant changed")*
- RF-5.4 When a request or a refresh ends indeterminately — network loss, timeout, service unavailable — the system shall show an accessible retry / offline status that names neither expiry nor sign-in and offers a retry; when the indeterminate outcome was the coordinated refresh, the status shall offer "Retry session refresh", which repeats the coordinated refresh once per activation. *(FR-IDN-064; decision Q3; S-41)*
- RF-5.5 If a request fails the forgery defence of RF-1.4, then the system shall show a safe problem stating that the request could not be validated and inviting the person to reload and try again; it shall not end the session, not refresh, not retry automatically, and shall never describe the defence. *(FR-IDN-064, FR-X-033; S-42)*
- RF-5.6 A failed or unanswered request shall never be taken as proof that its operation was not accepted; for a consequential operation the client shall look the submission up before offering to submit it again. *(FR-X-022; board 3)*

### RF-6 · The customer-target wall

Whatever a customer target answers is test data. *Why:* an ASpectra credential that reaches a target, or a target that can sign the console out, turns the test tool into an attack surface.

- RF-6.1 An authentication failure returned by a customer target during a run (the target's own "unauthorised" answer) shall be treated as data about the target: rendered in the run monitor as a target response, never as console expiry, and never starting a refresh or a re-authentication. *(FR-IDN-065, FR-RUN-055)*
- RF-6.2 The system shall never attach an ASpectra credential of any kind to a request sent to a customer target, and the console's own origin shall never be used as a target address or a target credential. *(FR-IDN-065)*
- RF-6.3 Loss of a run's event stream shall be rendered as stream interruption with the designed recovery — "Stream interrupted · facts stale", "Refresh snapshot", "Leave monitor" — never as expiry and never as a failed assertion. *(FR-RUN-055; D02 `159:32045`)*

### RF-7 · The expiry dialog

When the session has genuinely ended, the product says so once, protects the screen and keeps the person's place. *Why:* the dialog is the only honest moment between "signed in" and "signed in again"; it must neither leak what is behind it nor pretend to know why it happened. *(Decision: Q3.)*

- RF-7.1 When the client receives the definitive answer that the session is no longer valid (RF-4.5), the system shall mask or unmount every piece of restricted content, clear transient secrets from memory, close realtime subscriptions, stop polling, and present the dialog over the masked screen. *(FR-IDN-066; decision Q3)*
- RF-7.2 The dialog shall carry the designed content: the marker "A·Spectra / protected action"; the heading "Reauthenticate to continue."; the statement "Your protected session needs verification. Your current workspace, project and unsaved context are preserved."; the block "Accepted runs continue — Session expiry does not cancel an accepted run. Signing in again never replays a Run request."; the statement "Refresh is single-flight. Expiry and any remaining time are supplied by the server policy, not a local countdown."; the actions "Retry session refresh" and "Reauthenticate"; and the footer "Need help? Use local account recovery. You may also sign out safely." *(R34 `148:489`)*
- RF-7.3 The dialog shall not say why the session ended — deadline, revocation, version change or reuse; the cause exists in the audit log (RF-11.1) and, where a feature sends one, in that feature's notification. *(FR-X-033; decision Q3)*
- RF-7.4 "Reauthenticate" shall be the primary action after a definitive answer and shall start RF-8. "Retry session refresh" shall repeat the coordinated refresh once per activation; a definitive refusal shall leave the dialog unchanged, and the dialog shall never retry on its own. *(decision Q3)*
- RF-7.5 "Use local account recovery" shall discard the preserved context and draft and route to the public entry of controlled recovery (specification 005); "sign out safely" shall end whatever remains of the session server-side (specification 007), discard the preserved context and draft, and return to the sign-in screen. *(R34; specifications 005 and 007)*
- RF-7.6 While the dialog is shown, the system shall send no request that needs the session, reopen no realtime stream, and expose nothing under the mask; the dialog shall be modal and leave only through its actions. *(FR-IDN-066)*
- RF-7.7 Where the ended session is learned inside a panel, the system shall present the compact D02 form "Verify to continue protected work — Current context is preserved. Accepted runs continue; signing in never replays a Run request." with "Reauthenticate" and "Retry refresh", under the same rules as the dialog. *(D02 `159:32045`)*
- RF-7.8 When several tabs of one browser context share the session, each tab shall learn the definitive answer on its own next eligible request; when re-authentication succeeds in one tab, the other tabs shall recover at their next coordinated refresh without asking for credentials again, and their masked content shall be restored only after that refresh succeeds and authority is derived again. *(FR-IDN-062, FR-IDN-063)*

### RF-8 · Re-authentication

Proving oneself again is a sign-in that remembers where the person was. *Why:* R34 promises that tenant, project and unsaved context are preserved; the proof itself must be as strong as the first one, or the dialog becomes the weakest door. *(Decision: Q2.)*

- RF-8.1 When "Reauthenticate" is activated, the system shall run the full sign-in proof of specification 001 — the password and, whenever a factor is enrolled, the TOTP challenge — under Login's abuse protection (001 RF-4), generic failures, attempt limit and transaction expiry (001 RF-5, RF-6); the challenge completed by the ended session shall never count. *(FR-IDN-023, FR-IDN-025, FR-IDN-069; decision Q2; A-31; S-39)*
- RF-8.2 The re-authentication shall be for the identity of the ended session only: its address shown read-only, the person proving the password and the factor. Signing in as a different identity is done through "sign out safely" and the sign-in screen, which discards the preserved context and draft (RF-7.5); a different identity shall never inherit either. *(FR-IDN-066; decision Q2)*
- RF-8.3 When the proof succeeds, the system shall establish a new session (RF-1.1) with fresh deadlines and the current version, and shall return the person to the interrupted tenant, project and screen with authority derived afresh for that tenant; the landing rule of specification 001 RF-8 shall not apply. *(FR-IDN-024, FR-IDN-067; decision Q2; A-31)*
- RF-8.4 The gates of other features shall apply to a re-authentication exactly as to a sign-in: a pending forced password change (specification 004 R08) is met inside the re-authentication transaction; the enrollment gate (specification 003 RF-1.3, RF-1.4) is applied when the tenant is re-entered; a membership, grant or tenant state that changed meanwhile shall lead to the denied-scope state (RF-5.1) or to the tenant chooser, never to restored access. *(FR-IDN-026, FR-IDN-045, FR-IDN-067; specification 004 Q2, A-45; specification 003 Q12)*
- RF-8.5 Re-authentication shall never replay a pending request, launch a Run, accept a mapping or approve a bundle; it restores visibility only. *(FR-IDN-067, FR-RUN-050; P1)*
- RF-8.6 If the re-authentication fails, then the system shall show Login's generic failure; if it fails terminally — attempt limit or transaction expiry — then the system shall void the transaction and return the dialog to its initial state with the context and draft still preserved. *(FR-IDN-020, FR-IDN-025; D-a)*
- RF-8.7 When a page is reloaded or opened after its session ended, the first request shall learn the definitive answer, the dialog shall be shown over the empty shell, and a successful re-authentication shall return to the route that was requested when authority for it derives, otherwise to the tenant chooser; no draft exists in this case (RF-9.2). *(decision Q2; D-b)*
- RF-8.8 If the identity's status no longer permits sign-in, then the re-authentication shall fail with Login's generic failure; an identity that holds a session has a verified address, so the verification handoff of specification 001 cannot arise here. *(FR-IDN-048; A-27)*

### RF-9 · Unsaved work

The product offers to save while it still may, and afterwards keeps a draft only for the person who wrote it, in the place they wrote it. *Why:* losing work is the usual price of security; a draft in storage or shown to the wrong login is the usual price of convenience. *(Decision: Q4.)*

- RF-9.1 While the session is valid, the server-supplied remaining time is under a configured threshold and the screen holds unsaved work, the system shall show a non-blocking notice offering "Save now"; the save shall run under the still-valid authority with the original tenant and project context; the notice shall never extend the session by itself and shall disappear when nothing is unsaved. *(FR-IDN-066, FR-X-046; decision Q4; S-40)*
- RF-9.2 After a definitive expiry, unsaved work shall be kept in memory only, hidden behind the mask, and never written to browser storage; a reload or a closed tab loses it. *(FR-IDN-066, FR-X-034; decision Q4)*
- RF-9.3 The retained draft shall reappear only when the same identity re-authenticates into the same tenant and project and the derived authority still permits the edit; otherwise it shall be discarded without being shown. *(FR-IDN-066; decision Q4)*
- RF-9.4 The system shall never save a draft automatically after expiry, under the mask, or as part of re-authentication; saving remains an explicit action of the person. *(FR-IDN-066, FR-X-023)*
- RF-9.5 If a save was in flight when the session ended, then any answer that arrives shall be honoured; if none arrives, the draft is retained as RF-9.2 and, after re-authentication, the client shall look up whether the submission was accepted before offering to save again. *(FR-X-022)*
- RF-9.6 The dirty guard of the guarded tenant switch (specification 016) is unchanged by this feature; a save it triggers shall be submitted under the original tenant and project, never the newly selected one. *(FR-IDN-130, FR-IDN-131)*

### RF-10 · Accepted Runs

An accepted Run belongs to the platform, not to the browser that requested it. *Why:* execution facts are evidence; a laptop lid closing must not erase them, and a sign-in must not create a second Run.

- RF-10.1 Session expiry, revocation, logout and re-authentication shall never cancel, duplicate, pause or replay an accepted Run; it continues under its own narrow run-scoped authority and records its facts. *(FR-IDN-067, FR-RUN-050)*
- RF-10.2 Re-authentication shall restore inspection of the runs the identity is currently authorised to see; a Run request that was pending or uncertain when the session ended shall be looked up, never resubmitted. *(FR-IDN-067, FR-X-022)*
- RF-10.3 Membership or tenant revocation, compromise response and resource deletion are separate policies that may block queued work and request a bounded stop; session expiry shall never trigger them. *(FR-IDN-067, FR-IDN-116)*
- RF-10.4 The dialog (RF-7.2) and the Sessions area of specification 007 shall state that expiring a session does not cancel an accepted run. *(FR-SET-024; R38)*

### RF-11 · Audit, secrecy and notification

Every end of a session is recorded, no credential ever leaves its place, and a replayed credential is told to its owner. *Why:* the audit log is the only place the reason for an ended session may live (RF-7.3).

- RF-11.1 The system shall append to the audit log: each session established (by sign-in or by re-authentication); each session ended, with its cause — idle deadline, absolute deadline, version change and its source, individual revocation, logout, reuse detection; each reuse detection; each re-authentication succeeded, failed or voided. Each entry identifies the identity, the time and the calling context, with a correlation id. *(FR-X-044)*
- RF-11.2 No audit entry, log line, problem detail, URL, export, notification or analytics event shall ever contain an access credential, a refresh credential, a password or an authenticator code. *(FR-X-030, FR-X-033)*
- RF-11.3 Every refusal of this feature shall be a safe problem with a correlation id, distinguishing at least: session no longer valid, request could not be validated (RF-5.5) and denied scope (RF-5.1); no problem shall disclose deadlines, versions, family state or the defence that refused. *(FR-X-033)*
- RF-11.4 When reuse is detected (RF-3.5), the system shall send to the identity's verified email, through the transactional channel, a security notification stating that a session was ended because its credential was presented twice, when it happened, and what to do if unexpected — change the password and review the sessions list; it shall contain no credential and no link that grants access; a delivery failure shall never revert the revocation and shall be recorded. *(FR-X-045; decision Q6; A-50; S-14)*
- RF-11.5 Every write of this feature shall confirm persistence before reporting success; a rotation shall be observable as either done or not done, never half-done. *(FR-X-020; RNF-2)*

### RF-12 · Responsive and accessible surfaces

Session recovery is a critical journey. *Why:* everyone meets it, on every viewport.

- RF-12.1 The dialog shall follow M23 MobileSessionExpirySheet (`432:4427`) at mobile widths; the pre-expiry notice, the retry / offline status and the denied-scope state follow the D01 responsive policy, preserving every statement and action. *(FR-X-050, FR-X-055; S-02)*
- RF-12.2 When the dialog or a panel state opens, focus shall enter its heading or first meaningful control and follow reading order; every action shall have a touch target of at least 44 px, a visible label and visible focus; status shall be conveyed by text; a live announcement shall never steal focus from the password or code field of the re-authentication. *(FR-X-040, FR-X-042, FR-X-053; D02 interaction rule)*
- RF-12.3 All states and messages of this feature shall exist in EN / ES / FR; the interface language shall never change a deadline, a version, a remaining time or a credential. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **No authority from a session.** A session, a refresh or a re-authentication by itself grants no tenant or project authority; authority is derived on the server at every request. | FR-IDN-024, FR-X-001; P3 |
| RNF-2 | **Atomic rotation.** Issuing new credentials and retiring the presented one is one indivisible change; no observer sees two live refresh credentials of one family. | FR-IDN-062 |
| RNF-3 | **Replay resistance.** A retired refresh credential is never accepted; its presentation ends the family. | FR-IDN-062; decision Q6 |
| RNF-4 | **Server-owned time.** Deadlines, remaining time and expiry are decided on the server; the client never decides expiry from a local clock. | FR-IDN-061; R34 |
| RNF-5 | **Configured limits.** Idle deadline, absolute deadline, the save-notice threshold, safe-read retry bounds, and the attempt and expiry values re-authentication inherits from Login are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-6 | **Fail closed.** An absent, ended, superseded or unknown session, family or credential blocks every dependent request; nothing is assumed and no fallback is attempted. | P6 |
| RNF-7 | **Transport is not expiry.** Only a definitive server answer ends a session on the client; an indeterminate outcome is shown as such. | FR-IDN-064; decision Q3 |
| RNF-8 | **No browser persistence.** No credential, draft, server fact or secret of this feature is written to browser storage; a reload loses every in-memory draft. | FR-IDN-060, FR-X-034 |
| RNF-9 | **One retry, unchanged.** After a coordinated refresh, an interrupted operation is retried at most once with its original context; a consequential write is reconciled by lookup, never resubmitted. | FR-IDN-063, FR-X-021, FR-X-022, FR-X-023 |
| RNF-10 | **Run continuity.** An accepted Run is unaffected by anything that happens to the browser session. | FR-IDN-067, FR-RUN-050 |
| RNF-11 | **Revocation latency.** From the moment a session ends on the server, every request carrying its credentials is refused. | FR-IDN-061, FR-IDN-062 |
| RNF-12 | **Accessibility.** The dialog, the panel states and the notices meet WCAG 2.2 AA with automated and manual assistive evidence. | FR-X-040, FR-X-042 |
| RNF-13 | **Language.** EN / ES / FR, display only. | FR-X-043 |
| RNF-14 | **Reflow.** Mobile and tablet reflows preserve every statement and action. | FR-X-055 |
| RNF-15 | **Verification.** Concurrent refresh, reuse detection and multi-tab behaviour, cookie / forgery / origin configuration and active runs surviving browser expiry are verified as area A requires. | requirements A.12; P9 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Two tabs refresh at the same instant | One refresh honoured; the other tab coalesces on its result; no reuse detected (RF-3.4, RF-4.1). |
| LC-2 | A retired refresh credential is presented from a copied cookie | Family and session ended everywhere; dialog in every tab; audit and notification (RF-3.5, RF-11.4). |
| LC-3 | An ambiguous retry presents the same refresh credential twice | Treated as reuse; the family ends; the person re-authenticates; no credential is reissued (RF-3.5; decision Q6). |
| LC-4 | Person types for a long time without any request | No activity is counted; the notice offers "Save now" when the remaining time passes the threshold; the idle deadline then ends the session (RF-2.3, RF-9.1). |
| LC-5 | Absolute deadline passes while a save is in flight | The save's answer, if any, is honoured; the next request learns the definitive answer; the draft is retained; after re-authentication the submission is looked up (RF-2.4, RF-9.5). |
| LC-6 | Access credential expires during a consequential write | One coordinated refresh; one retry with the original key and body; the server returns the recorded outcome (RF-4.2, RF-4.4). |
| LC-7 | Refresh ends with no server answer | Retry / offline status; "Retry session refresh"; no expiry claimed; no mask (RF-5.4; decision Q3). |
| LC-8 | Project grant removed while editing | Denied-scope state after one permission refresh; draft kept and shown only if the refreshed scope permits (RF-5.1 … 5.3). |
| LC-9 | Forgery defence refuses a request | Safe problem with "reload and try again"; session untouched; no refresh (RF-5.5). |
| LC-10 | Customer target answers "unauthorised" inside a run | Rendered as the target's response in the monitor; no refresh, no dialog (RF-6.1). |
| LC-11 | Run event stream lost | "Stream interrupted · facts stale"; snapshot refresh; never expiry, never a failed assertion (RF-6.3). |
| LC-12 | Page reloaded after the session ended | Dialog over the empty shell; no draft; re-authentication returns to the requested route or the chooser (RF-8.7; D-b). |
| LC-13 | Person wants to sign in as another identity from the dialog | "Sign out safely" → sign-in screen; context and draft discarded; landing rule applies (RF-7.5, RF-8.2). |
| LC-14 | Re-authentication by an identity with a pending forced password change | R08 inside the re-authentication transaction, then the challenge, then the return to context (RF-8.4; specification 004 Q2). |
| LC-15 | Re-authentication by an Owner whose factor was never enrolled | Password only (001 Q2); on re-entering the tenant the enrollment gate withholds authority and routes to enrollment (RF-8.4; specification 003 RF-1.3). |
| LC-16 | Re-authentication fails terminally | Transaction voided; dialog back to its initial state with context and draft kept; throttles apply (RF-8.6; D-a). |
| LC-17 | Membership removed while the session was expired | Re-authentication succeeds; the interrupted tenant is no longer derivable; tenant chooser; draft discarded (RF-8.4, RF-9.3). |
| LC-18 | Password reset completed on another device | Version advances; this session ends at its next use; dialog; the reset's notification explains (RF-2.5; specification 004 RF-5.5). |
| LC-19 | Authenticator replaced from this session | This session continues; every other session ends at its next use (RF-2.5; specification 003 RF-9.7, Q9). |
| LC-20 | "Sign out everywhere" used in another session | This session ends at its next use; dialog (RF-2.5; specification 007). |
| LC-21 | Only a polling tab is open until the idle deadline | Polling never counts; the session ends; the tab learns it on its next eligible request and shows the dialog (RF-2.3, RF-7.1). |
| LC-22 | Realtime subscription open at expiry | Closed with the mask; reopened only after re-authentication and re-derived authority (RF-7.1, RF-7.8). |
| LC-23 | Re-authentication succeeds in one of three tabs | The other two recover at their next refresh without credentials; their content is restored after that refresh (RF-7.8). |
| LC-24 | Run accepted, then the session expires, then the person re-authenticates | The Run continued and finished; re-authentication shows its facts; nothing is relaunched (RF-10.1, RF-10.2). |
| LC-25 | Run request sent, session expired before the acknowledgement | After re-authentication the submission is looked up; never resubmitted (RF-10.2, RF-4.4). |
| LC-26 | Language switched while the dialog is open | Display changes; deadlines, versions and the re-authentication transaction unaffected (RF-12.3). |
| LC-27 | Identity becomes restricted while signed in | Version advances; session ends at next use; re-authentication fails generically (RF-2.5, RF-8.8; A-27). |
| LC-28 | Pre-expiry notice shown, person saves, then keeps working | Save under valid authority; the save counts as activity and moves the idle deadline; the notice disappears (RF-9.1, RF-2.2). |
| LC-29 | Reuse notification cannot be delivered | The revocation stands; the failure is recorded (RF-11.4). |
| LC-30 | Session revoked individually from the sessions list of another device | This session ends at its next use; dialog; no notification by this feature (RF-2.6; specification 007). |

---

## 6. Out of scope

- **Logout, "sign out everywhere", the sessions list and individual revocation** — F-IDN-09, specification 007; "sign out safely" on R34 is defined here only as the exit of RF-7.5.
- **Controlled account recovery** — F-IDN-07, specification 005; only the "local account recovery" route of R34 is here.
- **The guarded tenant and project switch**, its dirty guard and scope epoch — F-IDN-18, specification 016.
- **Sign-in itself**, the limited identity state and the landing rule — specification 001; re-authentication reuses them without redefining them.
- **In-session interruption for a forced password change** — not in this release (specification 004 Q2, A-45); the forced change meets a re-authentication as it meets a sign-in.
- **Trusted devices, remembered sessions, "keep me signed in"** — product boundary (FR-IDN-069).
- **Single sign-on, federated sessions, native-app sessions** — deferred (FR-EXT-020).
- **Operator-side session tooling** (platform-wide revocation, session search) — not designed.
- **The run monitor** beyond the two rendering rules of RF-6 — execution and monitoring specifications.
- **Tablet-specific designs** of the dialog and its states — none; the D01 responsive policy applies.

---

## 7. Criteria of completion

Sessions, refresh and expiry are complete when all of the following hold:

1. Every user story US-1 … US-11 can be walked end to end on the desktop design and in the reflows with no step outside this specification, and US-12 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-12 has at least one verification that fails before and passes after the feature exists.
3. Deadline tests show that only deliberate-action requests move the idle deadline, that polling, focus refetches, heartbeats and refreshes never do, that the absolute deadline never moves, and that an ended session is refused on its next request (RF-2, LC-4, LC-21).
4. Rotation tests show atomic rotation, exactly one honoured refresh under concurrency, family-wide revocation on reuse with its audit entry and notification, and refusal of every retired credential (RF-3, RNF-2, RNF-3, LC-1 … LC-3).
5. Client tests show one coordinated refresh per browser context, one unchanged retry per interrupted operation, lookup instead of resubmission for consequential writes, and no retry of ineligible answers (RF-4, RNF-9, LC-6).
6. State tests show that denied scope, forgery-defence failure and transport failure each reach their own state without a refresh or sign-in loop, and that an indeterminate refresh never claims expiry (RF-5, RNF-7, LC-7 … LC-9).
7. Wall tests show that a customer target's "unauthorised" answer and a stream loss are rendered as such and never refresh an ASpectra credential, and that no ASpectra credential reaches a target (RF-6, LC-10, LC-11).
8. Dialog and re-authentication tests show the designed copy, the mask, the cleared secrets, the full proof for the same identity, the return to the interrupted context without the landing rule, the gates of specifications 003 and 004 applied, and the multi-tab recovery (RF-7, RF-8, LC-12 … LC-17, LC-23).
9. Draft tests show the pre-expiry notice, the in-memory retention, the same-identity-same-scope rule, no automatic save, and nothing in browser storage (RF-9, RNF-8, LC-5, LC-28).
10. Run tests show that an accepted Run survives expiry, revocation and re-authentication, and that a pending Run request is looked up and never resubmitted (RF-10, RNF-10, LC-24, LC-25).
11. Audit entries exist for every event of RF-11.1 and contain no credential, password or code; the reuse notification exists in EN, ES and FR and carries no credential or access-granting link (RF-11).
12. The dialog, the panel states, the notice and the retry status pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RNF-12 … RNF-14).
13. The register items this specification relies on — A-31 (closed here), A-49, A-50, A-11, A-27, A-30, A-39 and A-45 in the architecture register; S-39, S-40, S-41, S-42, S-02, S-07 and S-14 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
