# Specification 005 — Controlled account recovery

**Feature** Recovery of a lost authenticator: an unused recovery code, a verified recovery request, and the continuation of a recovery case · **Version** 0.1 · **Date** 27 October 2026 · **Status** Draft for review — open doubts closed 27 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-07 as the subject; F-IDN-03, F-IDN-04, F-IDN-05, F-IDN-06, F-IDN-08, F-IDN-09, F-IDN-15, F-IDN-16 and F-SET-03 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P13, P14) → `requirements.md` (FR-IDN-001, 003, 012, 020, 021, 022, 023, 024, 025, 028, 029, 030, 040, 041, 043, 046, 047, 048, 049, 061, 062, 067, 068, 069, 116; FR-SET-021, 022; FR-X-020, 021, 022, 030, 033, 034, 040, 042, 043, 044, 045, 046, 050, 051, 053, 055) → the design of record, screen N05 ControlledAccountRecoveryPage `161:49444`, and the entries that lead to it on R02 LoginPage, R05 MfaChallengePage, R41 FactorReplacementDialog and the "TOTP & recovery" area of R38 SecuritySettingsPage.

**Scope note.** Constitution v1.1 stated F-IDN-07 as FR-IDN-046 path (a) only. The design of record draws all three paths of FR-IDN-046 on N05, and the product owner decided on 27 October 2026 (decision Q1) that this specification covers the three: path (a) in full, and paths (b) and (c) as the person-facing request and continuation surfaces, with the review of a case itself internal, audited and out-of-band. The constitution was amended the same day (v1.2): F-IDN-07 is now "Controlled account recovery", binds FR-IDN-046 in full and FR-IDN-047, and carries the acceptance rules this specification details. Constitution and specification are consistent; no partial-scope line remains.

**Related specifications:** 001 Login (the sign-in screen and the MFA challenge from which recovery is entered; the challenge voids its transaction when recovery is chosen, RF-6.9; a recovery code may also complete the challenge directly, RF-6.7, which is not this feature) · 002 Signup and email verification (only a verified identity can hold a factor and therefore be recovered) · 003 TOTP enrollment, recovery codes and factor replacement (the codes consumed here; the replacement enrollment that recovery enters, RF-9.3, and its effects, RF-9.6–9.8; the "I lost access · controlled recovery" route of R41, RF-9.9) · 004 Password reset and forced password change (reset never touches the factor and recovery never touches the password) · 006 Sessions and refresh (the sessions and refresh families revoked when a factor is replaced) · 007 Logout and session management · 013 Tenant chooser and 014 Tenant suspension state (where a recovered identity lands and what it still cannot do).

This document states **what** controlled account recovery does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global**: one principal with an email address, a display name and a status, proven by a verified email and a password, and — mandatorily for every Owner and Admin — hardened by a TOTP authenticator with a set of single-use recovery codes. The authenticator and the codes belong to the identity, never to a tenant: no tenant Owner or Admin can reset, replace or disable them, because the same identity may belong to other tenants (FR-IDN-049). There is no administrator backdoor.

When the authenticator is lost, three situations exist and the design of record gives each a path on one screen, N05:

- **(a)** the person still holds an **unused recovery code**: the code, together with the password, proves the identity and leads straight into the protected replacement of the factor (specification 003 RF-9);
- **(b)** the person holds **no code**: they open a **verified recovery request**, which becomes a *case* reviewed through the platform's verified recovery process, out of band; access is not immediate and no completion time is promised;
- **(c)** the person holds a **case reference** received through the verified channel and wants to **continue** that case: a reference alone grants nothing, and the case is shown only after separate verification.

Whatever the path, recovery ends in exactly one effect — a **new authenticator and a new recovery-code set replacing the old ones** — and never in a session granted without proof, a membership restored, a suspension lifted, a password changed or a factor removed. Password reset (specification 004) is a different thing and never disables MFA; the screen says so.

### 1.2 Objective

Let a person who has lost their authenticator get a new one without help from any tenant administrator, while:

- requiring, on every path, proof of the first factor (the password) **and** proof of something only the holder has — an unused recovery code, or control of the verified address plus a case approved by the verified recovery process;
- consuming every code, reference step and approval exactly once, bounded in time and attempts, with one generic answer for every failure;
- never turning recovery into a session without proof, a membership, a lifted suspension, a disabled factor or a launched Run;
- revoking every other session of the identity when its factor changes, so that whoever held the old factor is signed out everywhere;
- disclosing nothing about which addresses have an account, which have a factor, which have an open case, and what a case contains, to anyone who has not verified.

### 1.3 Why it matters

A lost authenticator is the moment the second factor stops protecting and starts excluding: a founder who cannot sign in loses their company's test estate, and a product that answers "ask your administrator" has built the backdoor it promised not to have. The opposite failure is worse: a recovery that accepts an email alone turns every mailbox compromise into an MFA bypass, and a recovery that an Admin can trigger turns one tenant's administrator into every tenant's. The feature must be survivable for the honest holder and useless to everyone else, with a human review as the path of last resort — bounded, audited and never promising a time.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 27 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Exact location, all written on 27 October 2026: Figma file `DpfPk8xDraLcDzBhMYwDTo` → page "ASpectra Architecture" (`0:1`) → spec block "Final multi-tenant MVP decision register" (`122:2`) → §14, summary table and entry paragraphs: **A-47** (verified recovery process: no mechanism for case review and approval) and **A-48** (recovery case and one-time email verification code as records) are new; notes appended to **A-27** (identity statuses), **A-30** (security notifications) and **A-39** (revocation scope). Page "ASpectra · Product experience UI" (`148:2`) → D03 `148:4` → Screen decisions ledger: **S-38** (N05 fields and states) is new; notes appended to **S-14** (email templates), **S-03** (mobile counterparts) and **S-17** (Help link).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | All three paths of FR-IDN-046: (a) recovery code in full; (b) verified recovery request and (c) case continuation as the person-facing surfaces of N05. The review of a case is internal, audited and out-of-band — no customer-facing operator surface exists (scope §2) — and only its effects are specified: case states, notifications, and what an approved case grants. This extended the constitution's F-IDN-07 baseline from path (a) to the three paths; constitution v1.2 (27 October 2026) records the amendment. |
| Q2 | What proves the person on path (a) | The current password **and** an unused recovery code, always. From a session (R41, R38) the address is the session's and read-only. From every public entry (R02, and R05 after its transaction is voided by specification 001 RF-6.9) the card asks address, password and code; an address carried over from the challenge may be pre-filled for convenience and proves nothing. The password field is drawn on every N05 card in D02 `423:3542` (S-38). A wrong address, password or code gives one generic failure and consumes nothing. |
| Q3 | What follows a verified recovery code | The code is consumed the moment it verifies, even if the replacement is later abandoned. The replacement enrollment of specification 003 RF-9.3 starts at once: from a public entry inside a limited identity state (a recovery transaction), from a session inside that session. On completion the old factor and the old code set are invalidated, every other session and refresh family is revoked, a security notification is sent, and the recovery transaction completes as a sign-in and lands (the code having counted as the second factor, as specification 001 RF-6.7 already allows); from a session, that session continues. Abandonment leaves the old factor active with one code fewer. |
| Q4 | What a verified recovery request collects, answers and creates | Address, current password and the acknowledgement "I understand that email access alone cannot disable MFA." The screen always answers the same generic statement. Only for an active, verified identity whose password is correct is a recovery case created and its reference sent to the verified address — the verified recovery channel. The review ends in approved, rejected or expired. Approval grants one thing only: a single-use, time-limited entry into the replacement enrollment, reached by continuing the case (path (c)) — never a session, a membership, a lifted suspension or a password change. |
| Q5 | The "separate verification" of a case continuation | Reference and current password, then a short-lived one-time code sent to the case's verified address. Only then are details shown: the case state, the masked address and the request time; an approved case offers "Continue to replace your authenticator". Never a display name, tenant, membership, reviewer identity or note. An unknown reference, a wrong password or a wrong code leads to one generic failure. |
| Q6 | Case lifecycle, limits and errors | At most one open case per identity: a new request while one is open re-sends the same reference and creates nothing. A case expires after a configured time; an approval is valid for a configured time and is consumed by the completed replacement. A case closes automatically when the identity recovers by any other route — a recovery code consumed at the challenge or on path (a), a replacement completed from R41 — and when the identity becomes restricted or under erasure; a password reset or a forced password change does not close it. Requests, continuations and code entries have their own throttles, independent of sign-in and of password reset, and never lock the account; an unknown, expired or closed reference and a wrong code give one generic failure that reveals nothing. |
| Q7 | Entry points | N05 is reached from the sign-in screen R02, the MFA challenge R05, the replacement dialog R41 and the TOTP & recovery area of R38. The "approved verified recovery process" offered on the suspended-tenant screen N20 concerns the tenant (F-IDN-16) and is out of scope here. |

Two rules were derived from these decisions rather than asked, because each follows from an earlier specification; the product owner may overturn them:

- **D-a** A wrong password entered on any card of N05 counts as a failed first-factor attempt on the sign-in throttles, exactly as a wrong current password does on the forced-change screen (specification 004 Q6), because it is a password guess; recovery's own throttles (RF-8) apply in addition. Neither ever locks the account permanently.
- **D-b** An approved case remains usable — with the full verification of RF-6 on every entry — until the replacement completes or the approval expires, rather than being consumed by the first entry; a replacement abandoned halfway therefore does not cost the person their approval.

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | user who lost my phone but kept my recovery codes | to prove my password and one unused code and set up a new authenticator right away | I am back in without waiting for anyone |
| US-2 | user who lost my phone and my codes | to open a recovery request and be told plainly that it will be reviewed, with no promised time | I have a route that is not "ask your administrator" |
| US-3 | user holding a case reference | to continue my case after proving my password and my mailbox, and to see where it stands | I know whether I can proceed, wait or request again |
| US-4 | user whose case was approved | to set up my new authenticator and new codes and then land in my workspaces | the recovery actually ends with access |
| US-5 | Owner or Admin whose membership requires MFA | my tenant authority to return only once the new factor is verified and its codes acknowledged | recovery never weakens the requirement |
| US-6 | user who recovers | every other session of mine to end when the factor is replaced | whoever had my old phone or codes is signed out everywhere |
| US-7 | person who receives a recovery message I did not cause | to learn that a code was used, a case opened, approved or closed, and what to do | I can react before the account is misused |
| US-8 | user on a phone or tablet | the recovery screen to reflow and keep every statement, field and route | I can recover anywhere without losing information |
| US-9 | tenant Admin | to be told on the screen that I cannot reset another identity's MFA | I do not look for a backdoor that does not exist, and my users do not expect one |
| US-10 | attacker or over-reaching administrator (negative story) | — | I cannot learn whether an address has an account, a factor or a case; cannot use a code without the password or a reference without the mailbox; cannot reuse a code, a one-time code or an approval; cannot get a session, a membership or a lifted suspension out of recovery; cannot have MFA disabled; and cannot find an administrator or operator surface that does any of this |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *factor* means the identity's active TOTP enrollment; *code* means one of its unused recovery codes; *case* means a recovery request accepted into the verified recovery process; *reference* means the identifier of a case delivered to the verified address; *the verified channel* means the identity's verified email address and nothing else; *the replacement enrollment* means steps 1 to 3 of specification 003 RF-2 … RF-4 run under the replacement purpose (RF-9.3 there); *a recovery transaction* means the limited identity state of specification 001 RF-5 opened by this feature.

### RF-1 · The recovery screen and its entries

One public screen holds the three paths and says what recovery is and is not. *Why:* a person who has lost a factor is anxious and a target; one screen, one set of statements and one generic answer keep them safe and oriented. *(Decision: Q7.)*

- RF-1.1 The system shall present the recovery screen with the heading "Recover access. Keep protection.", the subtitle "Lost your authenticator? Recovery is separate from password reset.", two tabs "Recovery code" and "Lost all factors", the card "Use an unused recovery code" (RF-2), the card "No recovery code available?" (RF-4), the card "Continue a verified recovery case" (RF-6), the notice "No administrator backdoor — A Workspace Admin cannot reset another identity's MFA. A password-reset email does not disable, replace or bypass required factors.", the routes "← Back to sign in" and "Return to local sign in", the Privacy and Terms routes, and the interface-language switch (EN / ES / FR). *(FR-IDN-046, FR-IDN-049, FR-IDN-043, FR-X-043; N05)*
- RF-1.2 The system shall be reachable from the sign-in screen's lost-authenticator route (specification 001 RF-1.4), from the MFA challenge's entry to controlled recovery (specification 001 RF-6.1, RF-6.9), from "I lost access · controlled recovery" on the replacement dialog (specification 003 RF-9.9) and from the controlled-account-recovery entry of the TOTP & recovery area of Security Settings. *(FR-IDN-025, FR-IDN-029, FR-SET-022; decision Q7; R02, R05, R41, R38)*
- RF-1.3 When the screen is reached from the MFA challenge, the login transaction shall already be void (specification 001 RF-6.9); the address of that transaction may be pre-filled in the address fields for convenience and shall prove nothing. *(decision Q2)*
- RF-1.4 When the screen is reached from a session (R41 or R38), the address fields shall show the session identity's address read-only, the cards shall otherwise behave as in RF-2, RF-4 and RF-6, and a replacement that completes shall continue in that session (RF-3.6). *(decision Q2, Q7)*
- RF-1.5 The system shall not offer this screen from the suspended-tenant screen; the recovery process offered there concerns the tenant and is not this feature. *(FR-IDN-116; decision Q7)*
- RF-1.6 The system shall expose nothing about any identity on the screen beyond what the person typed or, in a session, the session identity's own address: no display name, membership, factor state, code count or case existence. *(FR-IDN-024; P3)*
- RF-1.7 If a field is empty or does not have the required shape when a card's action is activated, then the system shall identify the field with a message associated with it and shall not submit. *(FR-X-042)*
- RF-1.8 The system shall treat every submitted address case-insensitively and ignore surrounding whitespace, as sign-in does (specification 001 RF-2.6).
- RF-1.9 The system shall render a "Help" action on the screen opening N43 HelpShortcutsPanel (`430:4129`; S-17); Privacy and Terms open the documents the platform operator publishes outside the product, with an external-link cue (specification 017 Q3; register `122:2` §15) (S-14).
- RF-1.10 When the language switch is used, the system shall change the display only; nothing typed, verified, consumed or pending shall change. *(FR-X-043)*

### RF-2 · Path (a) — verification with an unused recovery code

A code and the password together prove the identity; a code alone proves only that a piece of paper was found. *Why:* factor replacement always requires the password plus the current factor (specification 003 Q4); here the code stands in for the factor and the password still stands for itself. *(Decision: Q2, Q3.)*

- RF-2.1 The card "Use an unused recovery code" shall present an address field ("Local account email"), a current-password field with a "show" toggle, a recovery-code field ("Recovery code · paste supported") that accepts a pasted code and ignores separators, the action "Verify recovery code", and the statement "Codes are sensitive and single-use. They are not saved in browser storage. Verification continues to protected factor replacement, not MFA removal." *(FR-IDN-025, FR-IDN-028, FR-IDN-046 (a); N05; decision Q2; S-38)*
- RF-2.2 When the action is activated, the system shall verify, as one evaluation, that the address belongs to an active identity with a verified email, that the password matches its credential, and that the code is an unused recovery code of that identity. *(FR-IDN-003, FR-IDN-028; decision Q2; A-27)*
- RF-2.3 If any of the three checks fails, then the system shall present one generic failure that does not say which check failed, shall consume nothing, shall keep the address and clear the password and code fields, and shall count the attempt as in RF-8.4. *(FR-IDN-020, FR-IDN-021; decision Q2)*
- RF-2.4 When the three checks succeed, the system shall consume the code permanently in the same step, so that it can never be accepted again anywhere, whether or not the replacement that follows completes. *(FR-IDN-028; decision Q3)*
- RF-2.5 When the three checks succeed from a public entry, the system shall open a recovery transaction: a limited identity state bound to the identity, with a purpose of recovery, an expiry and an attempt limit, in which nothing is reachable but the replacement enrollment (RF-3) and abandoning. It shall create no session. *(FR-IDN-024, FR-IDN-025; decision Q3)*
- RF-2.6 When the three checks succeed from a session, the system shall start the replacement enrollment inside that session without the step-up of specification 003 RF-9.1, the code having replaced it. *(FR-IDN-029; specification 003 RF-9.9)*
- RF-2.7 The system shall never show how many codes remain, before or after verification, on this screen. *(FR-IDN-020)*
- RF-2.8 The system shall send a security notification to the verified address when a code is consumed on this path, stating that a recovery code was used to start replacing the authenticator, when, and what to do if the person did not do it. *(FR-IDN-029; S-14; A-30)*

### RF-3 · The protected replacement and its completion

Every path ends in the same place: the replacement enrollment of specification 003, which keeps the old factor required until the new one is proven. *Why:* there is no silent disabling step; recovery replaces, never removes. *(Decision: Q3; D-b.)*

- RF-3.1 When path (a) verifies (RF-2.4) or an approved case is continued (RF-6.8), the system shall start the replacement enrollment of specification 003 RF-9.3 — a new seed, steps "Set up", "Verify code" and "Save recovery codes" under the replacement purpose — with the statement that the existing factor remains active until the replacement is verified and its codes are acknowledged. *(FR-IDN-026, FR-IDN-029; specification 003 RF-9.3)*
- RF-3.2 While the replacement enrollment is pending, the existing factor and the identity's remaining codes shall stay active and required everywhere; a sign-in elsewhere shall be challenged against the existing factor only. *(FR-IDN-029; specification 003 RF-9.4)*
- RF-3.3 When the acknowledgement of specification 003 RF-4.5 is submitted, the system shall apply specification 003 RF-9.6: activate the new seed, invalidate the previous seed and every code of the previous set, and make the new set the identity's valid codes, as one indivisible change. *(FR-IDN-028, FR-IDN-029)*
- RF-3.4 When RF-3.3 completes, the system shall revoke every other session and every other refresh family of the identity (specification 003 RF-9.7), void every other open login transaction of the identity, close any open case of the identity as completed (RF-5.6), and send a security notification to the verified address stating that the authenticator was replaced through account recovery, when, and what to do if the person did not do it. *(FR-IDN-029, FR-IDN-062, FR-IDN-068; A-39; S-14)*
- RF-3.5 When RF-3.3 completes inside a recovery transaction, the system shall complete that transaction as a sign-in — the password and the new, verified authenticator having been proven — and land the person by the ordinary rules of specification 001 RF-8: tenant chooser, workspace creation or the single tenant, with any suspension shown and not lifted. *(FR-IDN-024, FR-IDN-048; decision Q3)*
- RF-3.6 When RF-3.3 completes inside a session, that session shall continue as in specification 003 RF-9.7. *(decision Q3)*
- RF-3.7 If the replacement enrollment is abandoned, expires or exhausts its attempts at any step, then the system shall discard it; the existing factor and the remaining codes shall stay exactly as they were, a code consumed under RF-2.4 shall stay consumed, an approved case shall stay approved until its approval expires (D-b), nothing shall be revoked, and no replacement notification shall be sent. A public entry returns to the sign-in screen; a session entry returns to Security Settings. *(FR-IDN-029; specification 003 RF-9.5)*
- RF-3.8 The system shall not, as a consequence of recovery: remove or disable the factor, regenerate codes on their own, change the password or the address, create a session before RF-3.5, restore or create a membership, lift a tenant suspension, change the identity's status, or replay any pending Run intent; an accepted Run continues under its own authority. *(FR-IDN-043, FR-IDN-048, FR-IDN-067; P1)*
- RF-3.9 When a recovered Owner, Admin or policy-required Member enters a tenant, the system shall derive their authority from the new, acknowledged factor exactly as specification 003's enrollment gate does (003 RF-1, RF-5.3); nothing in recovery grants authority earlier. *(FR-IDN-023, FR-IDN-030)*

### RF-4 · Path (b) — the verified recovery request

A person with no code asks for a review; the screen promises nothing and reveals nothing. *Why:* the request is public and can be made by anyone for any address; only the verified channel and the review turn it into anything. *(Decision: Q4, Q6; D-a.)*

- RF-4.1 The card "No recovery code available?" shall present the statement "Start a controlled recovery request. Access is not immediate: identity and ownership checks must complete through the verified recovery process.", the heading "What remains protected" with "Existing MFA and workspace boundaries remain in place while the case is reviewed.", an address field, a current-password field with a "show" toggle, the acknowledgement "I understand that email access alone cannot disable MFA." as a checkbox that is never pre-ticked, the action "Start verified recovery request", and the statement "After a request is accepted, keep its case reference and follow only instructions in the verified recovery channel. No completion time is promised." *(FR-IDN-046 (b), FR-IDN-047, FR-IDN-048; N05; decision Q4; S-38)*
- RF-4.2 If the acknowledgement is not ticked when the action is activated, then the system shall identify it with a message and shall not submit. *(FR-X-042)*
- RF-4.3 When a request is submitted, the system shall answer with the same statement — "If an eligible account exists, recovery instructions have been sent to its verified address." — in the same shape and with comparable timing whether the address belongs to an active identity with a correct password, to an active identity with a wrong password, to an unverified, restricted or under-erasure identity, or to no identity; the fields shall keep the address, clear the password, and the action shall stay available. *(FR-IDN-040 by analogy, FR-IDN-020; decision Q4; S-38)*
- RF-4.4 When the address belongs to an active identity with a verified email, the password matches, and no case of the identity is open, the system shall create a case in state *received* with a reference that is random, high in entropy, bound to that identity and usable for nothing but continuing the case, and shall send the case-opened message to the verified address: the reference, that nothing has changed yet, that the case will be reviewed and no completion time is promised, that the person must keep the reference and follow only instructions received through this address, and that a person who did not request it need only keep their password and codes safe and may ignore it. *(FR-IDN-046 (b), FR-IDN-047, FR-IDN-041 by analogy; decision Q4; S-14)*
- RF-4.5 If a case of the identity is already open, then the system shall create nothing and shall re-send the case-opened message with the same reference, within the message bound of RF-8.6. *(decision Q6)*
- RF-4.6 If the address belongs to no identity, or to an unverified, restricted or under-erasure identity, or the password is wrong, then the system shall create no case, send no message, create no identity and store nothing from the request except what abuse protection and audit require. *(FR-IDN-022; decision Q4; A-27)*
- RF-4.7 The system shall not change the identity's status, factor, codes, sessions, memberships, pending forced-change flag or outstanding reset links because a request was made; only a completed replacement (RF-3) has effects. *(FR-IDN-043)*
- RF-4.8 The system shall store the reference only in a non-reversible form; the reference itself shall exist only inside the messages sent to the verified address. *(FR-X-030)*

### RF-5 · The case: review, outcomes and closure

A case is a bounded, audited promise to look — never a promise of time, and never a grant of access. *Why:* FR-IDN-047 forbids promising a completion time and FR-IDN-048 forbids bypassing anything while a case is reviewed; the review must be able to say no. *(Decision: Q1, Q4, Q6; A-47, A-48.)*

- RF-5.1 The system shall hold, per case, its identity, its state, the time of each transition and, for transitions made by the review, the acting platform actor. The states shall be: *received* → *under review* → *approved* | *rejected*; *received* and *under review* → *expired* (configured case lifetime) | *closed* (RF-5.6); *approved* → *completed* (RF-3.4) | *expired* (configured approval validity) | *closed* (RF-5.6). *(FR-IDN-047; decision Q6; A-48)*
- RF-5.2 The review shall be an internal, audited, out-of-band platform process that performs the identity and ownership checks; it shall have no customer-facing surface, and no tenant Owner, Admin or Member action, tenant policy, membership change or invitation shall open, move, approve, reject or close a case. *(FR-IDN-047, FR-IDN-049; scope §2; decision Q1; A-47)*
- RF-5.3 When the review approves a case, the system shall record the approval with its actor and its configured validity and shall send the case-approved message to the verified address: that the case was approved, that the person must continue the case on the recovery screen with their reference within the stated validity, that the step asks for their password and a code sent to this address, that nothing has changed yet, and what to do if they did not request it. *(decision Q4; S-14; A-47)*
- RF-5.4 When the review rejects a case, or a case expires, the system shall send the case-closed message to the verified address: that the case was closed without approval and why (rejected or expired), that nothing was changed, and that a new request may be made. *(decision Q6; S-14)*
- RF-5.5 The system shall promise no completion time anywhere: not on the screen, not in a message, not in any case detail. *(FR-IDN-047)*
- RF-5.6 When a recovery code of the identity is consumed — at the MFA challenge (specification 001 RF-6.7) or on path (a) (RF-2.4) — or a replacement of the identity's factor completes by any route (specification 003 RF-9.6, RF-3.3 here), or the identity becomes restricted or under erasure, the system shall close any open case of the identity automatically, record the cause, and send the case-closed message stating that the case was closed because access was recovered by another route or because the account is no longer eligible. *(decision Q6; A-27, A-48)*
- RF-5.7 A password reset or a forced password change of the identity shall not close, move or affect a case. *(decision Q6; specification 004)*
- RF-5.8 An approval shall grant exactly one thing: the entry into the replacement enrollment through RF-6.8, valid until the replacement completes or the approval expires. It shall never grant a session, a membership, a lifted suspension, a password change or a factor removal. *(FR-IDN-047, FR-IDN-048; decision Q4; D-b)*
- RF-5.9 While a case is open, every existing MFA requirement, tenant boundary, suspension and membership state of the identity shall remain exactly in place; the case shall be invisible to every tenant surface. *(FR-IDN-048)*
- RF-5.10 The case record and the review shall hold no password, no code, no one-time code, no reference in clear and no tenant or project content; the identity, its address, the times and the states are all the review needs. *(FR-X-030; P3)*

### RF-6 · Path (c) — continuing a case

The reference opens nothing; the password and the mailbox open the case; the approved case opens the replacement. *Why:* FR-IDN-047 — a reference alone grants no access, and details are shown only after separate verification. *(Decision: Q5; D-a, D-b.)*

- RF-6.1 The card "Continue a verified recovery case" shall present a reference field ("Recovery case reference · not a sign-in code"), a current-password field with a "show" toggle, the action "Continue case verification", and the statement "A reference alone grants no access. Case details are shown only after separate verification. Identity, ownership and factor-replacement checks remain required; no instant restoration is promised." *(FR-IDN-046 (c), FR-IDN-047; N05; decision Q5; S-38)*
- RF-6.2 When the action is activated, the system shall check, as one evaluation, that the reference belongs to an open case (received, under review or approved) and that the password matches that case's identity; when both hold, it shall send a one-time code to the case's verified address, single-use, with a configured lifetime and attempt limit. *(FR-IDN-047; decision Q5; A-48)*
- RF-6.3 Whether or not the checks of RF-6.2 hold, the system shall present the same one-time-code step with the statement "If the reference and password match an open case, a verification code has been sent to its verified address.", in the same shape and with comparable timing, so that nothing reveals whether a case exists. *(FR-IDN-020; decision Q5; S-38)*
- RF-6.4 If the submitted one-time code is wrong, expired or already used, or no code was sent, then the system shall present one generic failure, keep the person on the step until the attempt limit is reached, and then return to the card with every pending code void; a new continuation sends a new code within the bounds of RF-8. *(FR-IDN-021; decision Q5, Q6)*
- RF-6.5 When the one-time code verifies, the system shall show the case: its state, the address it belongs to in masked form, the time of the request, and, for a closed case reached before its closure message arrived, the outcome; it shall never show a display name, a tenant, a membership, a reviewer identity or note, or a completion time. *(FR-IDN-047; decision Q5)*
- RF-6.6 While the case is *received* or *under review*, the system shall offer no action but "Return to local sign in" and shall state that the person will be notified through the verified address. *(FR-IDN-047)*
- RF-6.7 While the case is *approved* and its approval unexpired, the system shall offer the action "Continue to replace your authenticator". *(decision Q5; S-38)*
- RF-6.8 When that action is activated, the system shall start the replacement enrollment of RF-3: from a public entry inside a recovery transaction (as RF-2.5), from a session inside that session (as RF-2.6); the approval shall remain valid, under a fresh verification of RF-6.2 … RF-6.4 on every entry, until RF-3.4 marks the case completed or the approval expires. *(decision Q4; D-b)*
- RF-6.9 The verified case details shall be shown for the current continuation only: leaving the screen, reloading it or letting it idle beyond a configured time shall require the full verification again, and nothing of the case, the reference, the password or the one-time code shall persist in the browser. *(FR-X-034; RNF-4)*
- RF-6.10 A reference of a case that is completed, expired, rejected or closed shall behave exactly as an unknown reference (RF-6.3, RF-6.4). *(decision Q6)*

### RF-7 · Generic states

Every failure of this feature looks the same from outside. *Why:* "wrong password", "no such case", "expired approval" and "already used" are all answers an attacker wants. *(FR-IDN-020, FR-IDN-040.)*

- RF-7.1 The system shall present one generic failure state for a failed code verification (RF-2.3), a failed one-time code (RF-6.4) and a continuation that led nowhere; it shall say that the details could not be verified, name no cause, name no identity, and offer only to try again or return to sign-in. *(FR-IDN-020; S-38)*
- RF-7.2 The "request received" statement of RF-4.3 and the one-time-code step of RF-6.3 shall be identical in content, shape and timing within normal variance across every class of input. *(FR-IDN-040; RNF-1)*
- RF-7.3 The system shall not state, on any surface, whether an address has a factor, how many codes remain, whether a case exists or what its state is, except to a person who has completed RF-6.2 … RF-6.4 for that case. *(FR-IDN-047)*

### RF-8 · Abuse protection

Every entry of this feature is a guess at something; guesses are slowed and bounded, and nobody is locked out. *Why:* FR-IDN-040 and FR-IDN-021 for recovery; a recovery flood must never become a denial of sign-in. *(Decision: Q6; D-a.)*

- RF-8.1 The system shall throttle code verifications, requests, continuations and one-time-code entries on independent dimensions: per target address, per case reference and per calling network / risk signal, with their own configured thresholds, progressive delays and bounded suppression windows. *(FR-IDN-021, FR-IDN-040)*
- RF-8.2 These throttles shall be independent of the password-reset throttles (specification 004 RF-3) and, except for RF-8.4, of the sign-in throttles: a recovery flood shall never restrict sign-in or reset, and a sign-in or reset restriction shall never block recovery. *(FR-IDN-040; specification 001 RF-4.7)*
- RF-8.3 When a dimension is beyond its suppression threshold, the system shall create no case, send no message, send no one-time code and consume no code for that dimension during the window, while still answering as in RF-4.3, RF-6.3 or RF-7.1. *(FR-IDN-021)*
- RF-8.4 A wrong password submitted on any card shall count as a failed first-factor attempt on the sign-in throttles of specification 001 RF-4, exactly as on the forced-change screen (specification 004 RF-8.6), in addition to the recovery dimensions; a wrong code, wrong reference or wrong one-time code with a correct password shall count on the recovery dimensions only. *(FR-IDN-021; D-a)*
- RF-8.5 The system shall lift every window automatically when it elapses; no sequence of requests by a third party shall prevent the legitimate holder from recovering beyond that window, and no restriction shall be permanent. *(FR-IDN-021, FR-IDN-040)*
- RF-8.6 The system shall bound the number of case-opened re-sends, one-time codes and recovery notifications per address and per unit of time, so that a flood cannot exhaust the email channel or flood a mailbox. *(FR-IDN-021, FR-X-045)*
- RF-8.7 Every threshold, delay, window, lifetime, validity and attempt limit of this feature shall be validated configuration; the system shall refuse to start with an absent or unbounded value and shall never substitute a default. *(FR-X-046; P13)*

### RF-9 · What recovery never does

The boundaries that make the feature safe to offer at all. *Why:* FR-IDN-048 and FR-IDN-049 are the reason an identity can be global.

- RF-9.1 The system shall never, through any path of this feature, lift a tenant suspension, restore a revoked membership, create a membership, change a role or grant, or bypass a required MFA: a recovered identity lands where specification 001 would land it and holds only what it already held. *(FR-IDN-048)*
- RF-9.2 The system shall offer no tenant-side or administrator surface that opens, approves or closes a case, consumes a code, replaces or disables another identity's factor, resets its password or erases it; the screen shall say so in the "No administrator backdoor" notice. *(FR-IDN-049, FR-IDN-030; N05)*
- RF-9.3 The system shall never disable, remove or weaken the factor as a result of recovery; the only outcome of a successful recovery is a verified replacement with acknowledged codes. *(FR-IDN-029, FR-IDN-043)*
- RF-9.4 The system shall never change the identity's password or verified address as a result of recovery; a person whose verified address is inaccessible has no route in this feature, and the screen shall not suggest one. *(FR-IDN-041, FR-IDN-047)*
- RF-9.5 A session, a remembered device or a prior sign-in shall never substitute for any proof of this feature. *(FR-IDN-069)*

### RF-10 · Responsive surface

The drawn mobile counterpart is M18 MobileAccountRecoveryPage (`432:4185`); tablet follows the D01 responsive policy (`414:2645`); every statement is kept. *Why:* the security statements are the product's promise; a smaller viewport may not drop them. *(FR-X-051, FR-X-055.)*

- RF-10.1 The system shall present the recovery screen, its request-received state, its one-time-code step, its case-details states and its generic failure state in a reflowed layout at mobile and tablet widths that meets WCAG 2.2 AA reflow, with every field, action, route and statement of RF-1, RF-2, RF-4, RF-6 and RF-7 preserved in meaning. *(FR-X-051, FR-X-055; S-03)*
- RF-10.2 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*
- RF-10.3 The system shall follow M18 MobileAccountRecoveryPage (`432:4185`) at mobile widths and the D01 responsive policy at tablet widths. *(FR-X-050; S-03)*

### RF-11 · Audit, notification and secrecy

Every outcome is recorded and nothing secret is ever written. *Why:* a recovery is the event most worth reconstructing after a takeover, and the code, the reference or the one-time code must never appear in that reconstruction. *(FR-X-044.)*

- RF-11.1 The system shall append to the audit log: each code verification attempt and its outcome (with the matched identity when one exists); each code consumed; each recovery transaction opened, completed, abandoned or expired; each replacement completed through recovery; each request (with the matched identity when one exists, and whether a case was created, re-sent or nothing done); each case transition with its cause and, for review transitions, its actor; each continuation attempt and its outcome; each one-time code sent, verified, failed or voided; each session revocation caused by this feature; each message sent and each one suppressed under RF-8.6; each restriction applied or lifted. *(FR-X-044)*
- RF-11.2 The audit record shall identify the identity when one is matched, the time and the calling context, and shall never contain a password, a recovery code, a one-time code, a reference in clear, an unmatched address or a message body. *(FR-X-030, FR-X-044)*
- RF-11.3 The system shall never include a password, a recovery code, a one-time code or a reference in a log, an error, a trace, a client-visible problem or analytics; client-visible problems shall use safe codes with a correlation identifier. *(FR-X-030, FR-X-033)*
- RF-11.4 The system shall never write a password, a recovery code, a one-time code, a reference, a case detail or any form content of this screen to browser storage. *(FR-X-034)*
- RF-11.5 The five messages of this feature — recovery code used (RF-2.8; D04 M-16), case opened (RF-4.4; M-17), case approved (RF-5.3; M-18), case closed (RF-5.4, RF-5.6; M-19), continuation code (RF-6; M-20) — and the replacement notification of RF-3.4 (M-15) shall be sent only to the verified address, through the transactional email channel, in EN, ES and FR; their wording is the D04 inventory (`437:4431`; S-14), and a delivery failure shall be an operational event that never reverts an effect and is never visible on the screen. *(FR-X-045; RNF-11)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Indistinguishability.** The request answer and the one-time-code step are identical in content and shape, with timing comparable within normal variance, for active, wrong-password, unverified, restricted, erasing and unknown addresses, for open, closed and unknown references, and for throttled inputs. The generic failure is identical for every cause. | FR-IDN-020, FR-IDN-040, FR-IDN-047 |
| RNF-2 | **No lockout.** No sequence of requests, continuations, code or one-time-code entries by a third party restricts the legitimate holder's sign-in or reset, and no restriction of this feature outlasts its configured window. | FR-IDN-021, FR-IDN-040 |
| RNF-3 | **Single use.** A recovery code is consumed at most once; a one-time code verifies at most once; an approval is completed at most once; under concurrency, two submissions of the same code or one-time code yield exactly one success. | FR-IDN-028, FR-IDN-047 |
| RNF-4 | **Secrecy.** Passwords, recovery codes, one-time codes and references are never logged, exported, persisted in clear or written to browser storage; references and one-time codes are stored only in non-reversible form and delivered only to the verified address. | FR-X-030, FR-X-034, FR-IDN-041 |
| RNF-5 | **Bounded workload.** Verification work, review load and outgoing mail per unit of time are bounded regardless of request volume: one open case per identity, capped re-sends and codes. | FR-IDN-021, FR-X-045 |
| RNF-6 | **Configured limits.** Case lifetime, approval validity, one-time-code lifetime and attempts, recovery-transaction expiry and attempts, idle time of case details, every threshold, delay and window are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-7 | **Fail closed.** An unknown, consumed or malformed code, an unknown, closed or expired reference, a missing or expired one-time code, an expired approval, an absent policy or an absent configuration value blocks the effect; nothing is assumed or defaulted. | P6 |
| RNF-8 | **No authority before proof.** Neither the screen nor any of its states exposes tenant, project, membership or personal data; no session exists before RF-3.5, and no tenant authority before RF-3.9. | FR-IDN-024; P3 |
| RNF-9 | **Human review, bounded.** A case is approved only by an audited human decision of the platform process; no automatic approval exists, no completion time is promised, and the review can say no. | FR-IDN-047; scope §2 |
| RNF-10 | **Factor integrity.** No path of this feature disables, removes or bypasses the factor or the required-MFA policy; the only change is a verified replacement with acknowledged codes. | FR-IDN-029, FR-IDN-043, FR-IDN-048 |
| RNF-11 | **Revocation latency.** When the factor is replaced, every other session and refresh family of the identity is unusable from the moment the change is committed. | FR-IDN-062, FR-IDN-068 |
| RNF-12 | **Email as prerequisite.** The five messages depend on the transactional email channel; its absence is an operational failure, not a silent skip, and never a client-visible difference. | FR-X-045 |
| RNF-13 | **Accessibility.** The recovery screen and all its states meet WCAG 2.2 AA with automated and manual assistive-technology evidence: field errors associated with their fields, focus moved to messages and states, status conveyed by text, tabs operable by keyboard, no focus stealing. | FR-X-040, FR-X-042 |
| RNF-14 | **Language.** The screen, its states and the five messages exist in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-15 | **Reflow.** Mobile and tablet reflows preserve every statement, field, action, gate and route of the desktop screen. | FR-X-051, FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Correct password and unused code; identity has no factor | Impossible by construction (codes exist only with a factor); treated as a wrong code: generic failure, nothing consumed (RF-2.3). |
| LC-2 | Correct password, code already consumed at an earlier challenge | Generic failure; nothing consumed; counts on the recovery dimensions (RF-2.3, RF-8.4). |
| LC-3 | Wrong password, valid code | Generic failure; the code is **not** consumed; counts as a failed first-factor attempt (RF-2.3, RF-8.4). |
| LC-4 | Correct password and code; replacement abandoned at step 2 | Code consumed; old factor and remaining codes unchanged; nothing revoked; no replacement notification; back to sign-in (RF-2.4, RF-3.7). |
| LC-5 | Correct password and last remaining code; replacement completes | New factor and new set active; old set gone; every other session revoked; landing (RF-3.3 … RF-3.5). |
| LC-6 | Path (a) completed by an Owner whose only tenant is suspended | Landing shows the tenant chooser with the suspended state; nothing lifted (RF-3.5, RF-9.1). |
| LC-7 | Path (a) from the challenge of another device while the holder is signed in elsewhere | On completion the other session is revoked (RF-3.4); its accepted Runs continue (RF-3.8). |
| LC-8 | Path (a) from R41 by a signed-in person | Replacement inside the session; on completion the current session continues, every other one is revoked (RF-2.6, RF-3.6). |
| LC-9 | Same code submitted in two tabs at once | Exactly one verification succeeds and opens a recovery transaction; the other receives the generic failure (RNF-3). |
| LC-10 | Request for an unknown address | Generic statement; nothing created or sent (RF-4.3, RF-4.6). |
| LC-11 | Request for an active identity with a wrong password | Generic statement; nothing created or sent; failed first-factor attempt counted (RF-4.3, RF-4.6, RF-8.4). |
| LC-12 | Request for an unverified, restricted or under-erasure identity | Generic statement; nothing created or sent (RF-4.6). |
| LC-13 | Request for an identity that has no factor | Generic statement; a case is created — the review may still find nothing to replace and reject it (RF-4.4, RF-5.4). |
| LC-14 | Second request while a case is open | Generic statement; the same reference re-sent; nothing created (RF-4.5). |
| LC-15 | Second request while the re-send bound is exhausted | Generic statement; nothing sent during the window; the case is unchanged (RF-8.3, RF-8.6). |
| LC-16 | Third party floods requests for a holder's address | Generic statements; after the threshold no message during the window; the holder's sign-in and reset untouched; the window lifts (RF-8.3, RF-8.5, RNF-2). |
| LC-17 | Case expires while under review | State *expired*; case-closed message sent; a continuation with its reference behaves as unknown (RF-5.1, RF-5.4, RF-6.10). |
| LC-18 | Holder signs in with a recovery code at the challenge while a case is open | Case closed automatically with the cause; closure message sent (RF-5.6). |
| LC-19 | Holder completes a replacement from R41 while a case is open | Same as LC-18 (RF-5.6). |
| LC-20 | Holder resets the password while a case is open | Case unchanged (RF-5.7). |
| LC-21 | Identity becomes restricted while a case is approved | Case closed; the approval is void; a continuation behaves as unknown (RF-5.6, RF-6.10). |
| LC-22 | Continuation with an unknown reference and any password | One-time-code step shown; no code sent; any code fails generically (RF-6.3, RF-6.4). |
| LC-23 | Continuation with the right reference and a wrong password | Same as LC-22; failed first-factor attempt counted (RF-6.3, RF-8.4). |
| LC-24 | Continuation correct; one-time code typed after its lifetime | Generic failure; after the attempt limit back to the card, codes void (RF-6.4). |
| LC-25 | Continuation correct while the case is *under review* | Details shown: state, masked address, time; only "Return to local sign in" (RF-6.5, RF-6.6). |
| LC-26 | Continuation correct while the case is *approved*; replacement abandoned | Approval stays valid until its validity ends; a later continuation with full verification offers the action again (RF-6.8, D-b). |
| LC-27 | Continuation correct; approval validity ended before the action | Case *expired*; closure message; the reference now behaves as unknown (RF-5.1, RF-6.10). |
| LC-28 | Approved case; replacement completes | Case *completed*; new factor; every other session revoked; replacement notification; landing or session continues (RF-3.4 … RF-3.6). |
| LC-29 | Approved case continued in two browsers; both start the replacement | Two pending replacements of one identity: the first acknowledged completes and marks the case completed; the other, on its next step, is refused and discarded — the approval is completed at most once (RNF-3, RF-3.3). |
| LC-30 | Case details left open and idle beyond the configured time | Details cleared; a new full verification is required (RF-6.9). |
| LC-31 | Language switched on any state | Display changes; nothing typed, sent, pending or consumed changes (RF-1.10). |
| LC-32 | Interface reloads during the replacement enrollment | Specification 003's rules for a pending enrollment apply; a consumed code stays consumed; an approval stays valid (RF-3.7). |
| LC-33 | Email channel unavailable when a case is created | The request still answers generically; the case exists; the failure to send is an operational event in audit and monitoring (RNF-12). |
| LC-34 | Tenant Admin looks for a way to reset a member's MFA | None exists on any tenant surface; the member's own path is this screen; the notice on N05 says so (RF-9.2). |
| LC-35 | Identity whose verified address is inaccessible | No route: path (b) and (c) messages go only to that address, and nothing on the screen offers another channel (RF-9.4). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The review procedure itself** — how the platform performs the identity and ownership checks, who the reviewers are, their tooling and their surface: internal, audited, out-of-band (scope §2; A-47). Only the case states, their transitions and their effects are specified here.
- **Sign-in, the first factor, the challenge and landing** — specification 001; a recovery code submitted *at the challenge* completes the sign-in there (001 RF-6.7) and is not this feature.
- **TOTP enrollment, the replacement enrollment's steps, regeneration and the reveal** — specification 003; this feature only enters RF-9.3 there and inherits RF-9.4 … RF-9.8.
- **Password reset and forced password change** — specification 004; neither touches the factor, and recovery never touches the password.
- **Session lifetime, refresh, listing and individual revocation** — specifications 006 and 007; this feature only revokes.
- **Tenant suspension and its recovery process** — F-IDN-16 and N20 (decision Q7).
- **Recovery of an identity whose verified address is inaccessible or changed** — not in this release; the register notes that it needs controlled independent evidence (RF-9.4, LC-35).
- **Changing the email address, SMS, phone, security questions, trusted contacts, federated identity, passkeys, WebAuthn keys** — not part of the product in this release.
- **Any administrator- or operator-initiated reset, replacement or disabling of another identity's factor or password** — there is no such capability (FR-IDN-049).
- **Tablet-specific designs** of N05 — none; the D01 responsive policy applies.
- **The content of the Help panel** — "Help" on the screen opens N43 HelpShortcutsPanel (`430:4129`; S-17); the panel is the shell's, not this feature's.
- **The choice and configuration of the transactional email channel** — an operational prerequisite, not a feature.

---

## 7. Criteria of completion

Controlled account recovery is complete when all of the following hold:

1. Every user story US-1 … US-9 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-10 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-11 has at least one verification that fails before and passes after the feature exists.
3. Enumeration tests show no observable difference — content, shape, timing within normal variance — between requests and continuations for active, wrong-password, unverified, restricted, erasing and unknown addresses, between open, closed and unknown references, and between every cause of the generic failure (RNF-1).
4. Throttle tests show progressive delay, bounded suppression, automatic lifting, independence from the reset throttles and from the sign-in throttles except for the wrong-password rule of RF-8.4, and the absence of any permanent lockout (RNF-2, RF-8).
5. Proof tests show that no path succeeds without the password, that a code alone, a reference alone, a mailbox alone and a session alone each grant nothing, and that every failure is generic (RF-2, RF-6, RF-9.5).
6. Single-use tests show that a code is consumed at verification and never accepted again, that a one-time code verifies once, that an approval completes once, and that two concurrent submissions yield exactly one success (RNF-3, LC-9, LC-29).
7. Replacement tests show that each path enters the replacement enrollment of specification 003 RF-9.3, that the old factor stays required until acknowledgement, that completion invalidates the old factor and codes, revokes every other session and refresh family, notifies, closes the case and lands or continues as decided, and that abandonment changes nothing but the consumed code (RF-3).
8. Case tests show every state transition of RF-5.1 with its cause and actor, the one-open-case rule, automatic closure on every trigger of RF-5.6, indifference to password reset, and that no tenant-side action can open, move or close a case (RF-5, LC-13 … LC-21).
9. Boundary tests show that recovery never lifts a suspension, restores a membership, grants authority before the acknowledged factor, disables a factor or changes a password or address (RF-3.8, RF-3.9, RF-9).
10. The product refuses to start when any lifetime, validity, attempt limit, threshold, delay or window of this feature is absent or unbounded (RNF-6).
11. Audit entries exist for every event of RF-11.1 and contain no password, code, one-time code, reference, unmatched address or message body (RF-11.2).
12. The five messages are sent only to the verified address, exist in EN, ES and FR, carry the statements of RF-2.8, RF-3.4, RF-4.4, RF-5.3 and RF-5.4, and promise no completion time (RF-11.5, RF-5.5).
13. The recovery screen and all its states pass WCAG 2.2 AA with automated evidence and manual assistive-technology evidence, in EN, ES and FR, at desktop, tablet and mobile widths (RNF-13, RNF-14, RNF-15).
14. The register items this specification relies on — A-47, A-48, A-39, A-30 and A-27 in the architecture register, S-38, S-14, S-03 and S-17 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. The constitution's F-IDN-07 entry (v1.2) already reflects decision Q1.
