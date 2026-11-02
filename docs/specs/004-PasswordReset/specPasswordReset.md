# Specification 004 — Password reset and forced password change

**Feature** Self-service password reset by emailed link, and the forced password change state · **Version** 0.2 · **Date** 26 October 2026 · **Status** Draft for review — open doubts closed 26 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-06 as the subject; F-IDN-01, F-IDN-02, F-IDN-03, F-IDN-04, F-IDN-05, F-IDN-07, F-IDN-08, F-IDN-09, F-IDN-16 and F-SET-03 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P13, P14) → `requirements.md` (FR-IDN-001, 003, 011, 012, 020, 021, 022, 023, 024, 025, 040, 041, 042, 043, 044, 045, 048, 049, 061, 062, 067, 068, 069; FR-SET-020; FR-X-020, 021, 022, 030, 033, 034, 040, 042, 043, 044, 045, 046, 050, 051, 053, 055) → the design of record, screens R03 ForgotPasswordPage `148:264`, R07 ResetPasswordPage `148:86`, R08 ForceChangePasswordPage `148:43`, and the D02 atlas `159:32045` entry "This reset link can't be used".

**Related specifications:** 001 Login (the sign-in whose first factor precedes the forced change, and the sign-in the reset returns to; its abuse-protection rule RF-4.7 is given its other half by RF-3.2 here, and its LC-13 — password changed while a transaction is open — is implemented by RF-5.5) · 002 Signup and email verification (only a verified identity can be reset; its security message on a signup collision links to the reset request) · 003 TOTP enrollment, recovery codes and factor replacement (neither flow here touches the factor) · 005 Controlled account recovery (the route for a lost authenticator, which is never the reset) · 006 Sessions and refresh (the sessions and refresh families revoked here) · 007 Logout and session management (the sign-out offered on R08).

This document states **what** password reset and forced password change do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global** and **local**: one principal holding an email address, a display name and a status, proven by a verified email address and a password, optionally hardened by a TOTP factor. The password is the identity's own credential; it is never duplicated per tenant, and no tenant Owner or Admin can reset, replace or see it, because the same identity may belong to other tenants (FR-IDN-049). The only ways a password changes are therefore the person's own: a **reset** through a link sent to the verified address when the password is forgotten, and a **forced change** when the platform requires a new password before the identity may continue.

Reset is a recovery of the first factor only. It never touches the second factor: a person who resets their password and has a factor enrolled is challenged for it at the next sign-in exactly as before, and a person who has lost their authenticator must use controlled account recovery (specification 005), not a reset. The reset screens say so.

Forced password change is a **gate at sign-in**: after a correct first factor, and before the second-factor challenge, an identity flagged for a required update must enter its current password and choose a new one before it can proceed. The flag is set only by the platform — in response to a credential compromise, a tightened password policy that the stored credential no longer satisfies, or a password older than the configured maximum age — never by a tenant action, and never as an "administrator-created password" step.

### 1.2 Objective

Let a person with a verified ASpectra identity regain or renew their password without help from anyone, while:

- disclosing nothing about which addresses have an account, which are restricted, and whether a request did anything;
- making a reset link usable exactly once, only by its recipient, only for the newest request, only before it expires;
- never turning a reset or a forced change into a session, a membership, a lifted suspension, a disabled factor or a launched Run;
- ending every other session of the identity when its credential changes, so a stolen credential stops working the moment the holder replaces it;
- keeping the forced change a bounded, generic, non-lockable step that preserves everything the identity already holds.

### 1.3 Why it matters

A password reset is the most attacked path of any identity system: it lets anyone with an email address trigger an effect on an account they do not own. A leak here undoes the enumeration resistance of sign-in and signup; a reusable or long-lived link turns one intercepted message into an account takeover; a reset that also silenced MFA would turn a mailbox compromise into tenant access. A forced change exists so the platform can respond to a compromise or a weak policy without locking anyone out and without an administrator ever holding a user's password. Both must be safe by construction and still simple enough that a person in a hurry completes them.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 26 October 2026. The body refers to them as *(Decision: Qn)*. Q1 to Q6 were decided before the first draft; Q7 to Q10 closed the product doubts of the first draft. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (architecture `122:2` §14: A-44, A-45, A-33, A-27; D03 Screen decisions ledger `148:4`: S-35, S-36, S-37, S-14, S-03, S-17), each with the assumption this specification builds until it is resolved, and the body cites them by identifier.

| Q | Question | Decision |
|---|---|---|
| Q1 | Which conditions raise the forced password change | Three, all platform-side: (a) an internal, audited compromise response; (b) the server password policy was tightened and the stored credential no longer satisfies it, detected at the next correct first factor; (c) the password is older than a configured maximum age. No tenant-side trigger exists (FR-IDN-049). |
| Q2 | When R08 is raised and where the user continues | At sign-in, after a correct first factor and before the TOTP challenge. On success the user continues to the challenge, or to landing when no factor is enrolled. A flag set while sessions exist revokes them, so the holder meets R08 at the next sign-in. |
| Q3 | What a reset request does for an identity that is not active | Only an active, verified identity receives a link. Unverified, restricted and under-erasure identities, and unknown addresses, receive nothing. Response and timing are identical in every case. |
| Q4 | Several links and the moment of consumption | Only the newest link is valid: each new request invalidates earlier links. Opening a link validates it; submitting a compliant new password consumes it atomically. A link that expired after being opened fails on submit into the safe "request a new link" state. |
| Q5 | New-password rules and a reset while a forced change is pending | The new password must differ from the current credential in both flows; no password history is kept. A successful reset clears any pending forced change, whatever raised it. |
| Q6 | Errors and side effects of the forced change | A wrong current password is a generic field error, counts as a failed sign-in attempt for throttling, and is bounded by the login transaction's attempt limit and expiry (void → back to sign-in). A success revokes every other session and refresh family, invalidates outstanding reset links and sends the security notification of RF-5.7, except when the requirement came from a compromise response, which was notified when it was applied (Q10). |
| Q7 | A third party floods reset requests for an address and suppresses the holder's reset message for the window | Accepted as a bounded residual risk: sign-in is never affected and the window is bounded (RNF-2). The configured per-address window is shorter than the network window, so a flood is cut off at its source first. No exemption for the holder's own network, which would reveal it. |
| Q8 | A third party's later request invalidates the holder's still-valid link under "newest link only" | Keep newest-only — it is what makes a leaked older link worthless; the throttles of RF-3 make repeated invalidation impractical, and the holder simply requests again. |
| Q9 | A reset link opened in a browser that already holds a session | The reset screen is public and session-agnostic: it neither requires, consults nor modifies the session. A success revokes every session of the reset identity, including one in that browser; another identity's session is untouched. |
| Q10 | Password-changed notification when the change was forced by a compromise response | Not sent for that source. Instead the compromise response itself sends one security message when it is applied (RF-7.2): a password change is required at the next sign-in, every session was signed out, when, and what to do if unexpected. Every other credential change notifies as in RF-5.7. |

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | user who forgot my password | to request reset instructions with my email address | I can get back in without contacting anyone |
| US-2 | user holding a fresh reset link | to choose a new password that meets the current policy | my old password stops working and the new one works at sign-in |
| US-3 | user who opened an old, used or broken link | a clear, safe state that lets me request a new link | I am not stuck and I am not told anything that helps an attacker |
| US-4 | identity flagged for a required update | to be asked for my current password and a new one right after my password check at sign-in | I keep my workspaces, grants and work, and my sign-in simply continues afterwards |
| US-5 | Owner or Admin with TOTP enrolled | my factor to stay exactly as it is after a reset or a forced change | neither flow can be used to get around MFA |
| US-6 | user who resets my password | every other session of mine to end | whoever had my old password is signed out everywhere |
| US-7 | user on a phone or tablet | the request, reset and forced-change screens to reflow and keep every statement | I can recover anywhere without losing information |
| US-8 | person who receives a security message I did not cause | to learn that my password was changed, or that someone requested a reset, and what to do | I can react before the account is misused |
| US-9 | attacker (negative story) | — | I cannot learn whether an address has an account, cannot lock a victim out, cannot reuse, forge or extend a link, cannot disable MFA through a reset, and cannot find an administrator backdoor |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

### RF-1 · Reset request screen

The request screen collects one email address and says what it will and will not do. *Why:* the request is public and anonymous; the only safe interface is a single field and a generic answer.

- RF-1.1 The system shall present the reset request screen with an email field, a "Request reset instructions" action, a "Return to sign in" route, the Privacy and Terms routes, and the interface-language switch (EN / ES / FR). *(FR-IDN-040, FR-X-043; R03)*
- RF-1.2 The system shall state on the request screen that recovery requests use the same response whether or not an account exists, that links expire according to policy, and that resetting a password does not reset MFA and does not sign the user in automatically. *(FR-IDN-040, FR-IDN-043; R03)*
- RF-1.3 The system shall be reachable from the sign-in screen's password-reset route (specification 001 RF-1.1) and from the security message sent on a signup collision (specification 002 RF-3.2). *(R02, M02; S-23)*
- RF-1.4 If the email field is empty or does not have the shape of an address when the action is activated, then the system shall identify the field with a message associated with it and shall not submit the request. *(FR-X-042)*
- RF-1.5 The system shall treat the submitted address case-insensitively and ignore surrounding whitespace, as sign-in does (specification 001 RF-2.6).
- RF-1.6 When a request has been submitted, the system shall show the "Request received" statement — "If an eligible account exists, password reset instructions will be sent to the address provided." — keep the address in the field, move focus to the statement, and leave the action available for a further request. *(FR-IDN-040, FR-X-042; R03; S-36)*

### RF-2 · Reset request processing

Behind the generic answer, a link is issued only to an identity that is allowed to use it. *Why:* a link is a capability to change a credential; it must exist only where the credential exists and the address is proven. *(Decision: Q3, Q4.)*

- RF-2.1 When a request is submitted, the system shall answer with the same statement, the same shape and comparable timing whether the address belongs to an active identity, an unverified, restricted or under-erasure identity, or to no identity at all. *(FR-IDN-040; decision Q3; A-27)*
- RF-2.2 When the address belongs to an active identity with a verified email, the system shall create a reset token that is cryptographically random, bound to that identity and to the reset purpose only, and valid for a configured expiry. *(FR-IDN-041, FR-X-046)*
- RF-2.3 The system shall store only a non-reversible hash of the token; the token itself shall exist only inside the message sent to the verified address. *(FR-IDN-041, FR-X-030)*
- RF-2.4 The system shall send the message only to the identity's verified address, through the transactional email channel, with a link built from the configured trusted origin and never from a value supplied by the request. *(FR-IDN-041, FR-X-045)*
- RF-2.5 When a new token is created, the system shall invalidate every earlier reset token of the same identity, so that only the newest link can be used, whoever made the newer request. *(FR-IDN-042; decision Q4, Q8)*
- RF-2.6 If the address belongs to an unverified, restricted or under-erasure identity, or to no identity, then the system shall create no token, send no message, create no identity and store nothing from the request except what abuse protection and audit require. *(FR-IDN-022, FR-IDN-040; decision Q3)*
- RF-2.7 The system shall not change the identity's status, memberships, sessions, factor or pending forced-change flag because a reset was requested; only a completed reset (RF-5) has effects. *(FR-IDN-043)*
- RF-2.8 The message shall name the product, state that the link expires, state that it does not reset MFA or sign the person in, and tell a person who did not request it that no action is needed and that their password has not changed. Its exact wording is a design deliverable (S-14). *(FR-IDN-043)*

### RF-3 · Reset-request abuse protection

Requests are slowed and bounded without ever locking the account or telling the caller. *Why:* a reset request must never become a way to lock a victim out, and the mail it causes must stay bounded. *(FR-IDN-040, FR-IDN-021.)*

- RF-3.1 The system shall throttle reset requests on two independent dimensions: per target address and per calling network / risk signal. *(FR-IDN-021, FR-IDN-040)*
- RF-3.2 The system shall keep these throttles independent of the sign-in throttles: a reset request shall never count as a failed sign-in attempt, and a sign-in restriction shall never block a reset request. *(FR-IDN-040; specification 001 RF-4.7)*
- RF-3.3 When requests on a dimension exceed the configured threshold, the system shall apply a progressive delay before answering, and beyond a further configured threshold shall stop issuing tokens and messages for that dimension during a bounded window, while still answering with the statement of RF-1.6. *(FR-IDN-021, FR-IDN-040)*
- RF-3.4 The system shall lift every window automatically when it elapses; no request by a third party shall prevent the legitimate holder from obtaining a link beyond that window, and no request shall ever restrict sign-in. *(FR-IDN-021, FR-IDN-040)*
- RF-3.5 The system shall bound the number of reset messages sent per address and per unit of time, so that a flood of requests cannot exhaust the email channel or flood a mailbox. *(FR-IDN-021, FR-X-045)*
- RF-3.6 Every threshold, delay, window and expiry of this feature shall be validated configuration; the system shall refuse to start with an absent or unbounded value and shall never substitute a default. *(FR-X-046; P13)*
- RF-3.7 The configured per-address suppression window shall be shorter than the configured network window, so that a flood is cut off at its source before the holder's suppression ends; the suppression is a bounded denial of recovery and never of sign-in, and no exemption shall exist for any network. *(decision Q7; RNF-2)*

### RF-4 · Reset screen

The reset screen collects a new password twice and repeats the promise about the factor. *Why:* the person is about to prove nothing but possession of the link; the screen must not suggest more. *(R07.)*

- RF-4.1 When a reset link is opened, the system shall present the reset screen with a new-password field and a confirmation field, each with a "show" toggle, an "Update password" action, a "Return to sign in" route, the Privacy and Terms routes, and the language switch. *(FR-IDN-042, FR-X-043; R07)*
- RF-4.2 The system shall display the password rules supplied by the current server policy beside the new-password field and give strength feedback from that policy; the screen shall not hard-code any rule. *(FR-IDN-011; R07)*
- RF-4.3 The system shall state on the reset screen that the update does not reset MFA, that the person will return to sign in and complete the required verification, and that used or expired links lead to a separate safe request-new-link state. *(FR-IDN-043; R07)*
- RF-4.4 When the user activates a "show" toggle, the system shall reveal that field's value in place until hidden again, and shall never persist a typed password anywhere.
- RF-4.5 If the confirmation does not match the new password, then the system shall identify the confirmation field with a message and shall not submit. *(FR-X-042)*
- RF-4.6 If a field is empty or the new password does not satisfy the displayed policy when the action is activated, then the system shall identify the field with a message and shall not submit. *(FR-IDN-011, FR-X-042)*
- RF-4.7 The system shall expose nothing about the identity on the reset screen beyond what the link proves: no display name, email, memberships or status. *(FR-IDN-024; P3)*
- RF-4.8 The system shall treat the reset screen as public: an existing session in the same browser, of the same or another identity, shall neither be required nor consulted, and shall not be modified by opening the link. *(decision Q9; RF-5.5 governs what a success does to sessions)*

### RF-5 · Reset consumption

A valid, newest, unexpired link plus a compliant new password changes the credential exactly once and ends every session. *Why:* atomic single use is what makes an intercepted link worthless after the holder has used it, and revocation is what makes a changed password mean something. *(Decision: Q4, Q5.)*

- RF-5.1 When "Update password" is activated with a link whose token matches a stored, unexpired, newest, unconsumed hash and a new password that satisfies the policy, the system shall consume the token and change the credential in one atomic operation, so that two concurrent submissions of the same link produce exactly one credential change and one failure. *(FR-IDN-042)*
- RF-5.2 If the new password equals the identity's current credential, then the system shall reject it with a field message and shall not consume the token. *(decision Q5)*
- RF-5.3 The system shall not consume the token on opening the link, on rendering the screen, on a failed validation or on a rejected submission; only the successful submission of RF-5.1 consumes it. *(decision Q4)*
- RF-5.4 When the credential has changed, the system shall store it only through the non-reversible credential scheme of sign-in; the old credential shall stop working immediately. *(FR-IDN-003)*
- RF-5.5 When the credential has changed, the system shall revoke every session and every refresh family of the identity, void any open login transaction of the identity, and invalidate every other outstanding reset link. *(FR-IDN-042, FR-IDN-062; specification 001 LC-13)*
- RF-5.6 When the credential has changed, the system shall clear any pending forced-change flag of the identity, whatever raised it. *(decision Q5)*
- RF-5.7 When the credential has changed, the system shall send a security notification to the identity's verified address stating that the password was changed, when, and what to do if the person did not do it. *(FR-IDN-042; S-14)*
- RF-5.8 When the credential has changed, the system shall route the user to the sign-in screen with a confirmation that the password was updated; it shall not create a session, a limited identity state or any authority. *(FR-IDN-042, FR-IDN-043; S-35)*
- RF-5.9 The system shall not, as a consequence of a reset: sign the user in, restore or create a membership, lift a tenant suspension, change the identity's status, enroll, replace, disable or bypass the TOTP factor, consume or regenerate recovery codes, or replay any pending Run intent. *(FR-IDN-043, FR-IDN-048, FR-IDN-067; P1)*
- RF-5.10 When the user next signs in, the system shall apply the ordinary rules of specification 001: first factor with the new password, the TOTP challenge if a factor is enrolled, then landing by membership. *(FR-IDN-042)*

### RF-6 · Unavailable link state

Every link that cannot be used leads to one safe state that reveals nothing. *Why:* the difference between "used", "expired", "superseded" and "never existed" is information an attacker wants and the user does not need. *(FR-IDN-044; D02.)*

- RF-6.1 If a link is opened whose token is unknown, malformed, expired, consumed or superseded by a newer request, then the system shall present the "This reset link can't be used" state — the link is invalid, used or expired — with a "Request a new reset link" route to the request screen and a "Return to sign in" route. *(FR-IDN-044; D02 `159:32045` "Expired reset")*
- RF-6.2 The unavailable state shall not say which of the reasons applies, shall not name the identity or address, and shall state that no MFA reset and no automatic sign-in occur. *(FR-IDN-040, FR-IDN-043; D02)*
- RF-6.3 If a submission of the reset form fails because the token became expired, consumed or superseded after the screen was opened, then the system shall present the same unavailable state and shall not store or use the submitted password. *(decision Q4)*
- RF-6.4 The request screen reached from the unavailable state shall behave exactly as in RF-1 and RF-2, with the same generic answer. *(FR-IDN-044)*
- RF-6.5 The system shall count opens and submissions of unusable links toward the network-dimension throttle of RF-3.1, so that token guessing is slowed, without ever restricting the identity's sign-in. *(FR-IDN-021)*

### RF-7 · Raising the forced password change

The platform, and only the platform, marks an identity as requiring a new password, and the mark is met at the next sign-in. *Why:* a compromise response, a tightened policy or an aged password must be enforceable without an administrator ever touching a user's credential and without locking the user out. *(Decision: Q1, Q2.)*

- RF-7.1 The system shall hold, per identity, a pending forced-change requirement with its source and the time it was set. The source shall be one of: compromise response, policy change, password age. *(FR-IDN-045; decision Q1; A-44)*
- RF-7.2 When an internal, audited platform compromise response is applied to an identity, the system shall set the requirement with source "compromise response" and shall revoke every session and refresh family of the identity, so that the holder meets the requirement at the next sign-in, and shall send a security notification to the identity's verified address stating that a password change is required at the next sign-in, that every session was signed out, when, and what to do if the person did not expect it. *(decision Q1 (a), Q2, Q10; A-44; S-14)*
- RF-7.3 When a correct first factor is verified and the stored credential does not satisfy the current server password policy, the system shall set the requirement with source "policy change" before continuing. *(FR-IDN-011; decision Q1 (b))*
- RF-7.4 When a correct first factor is verified and the credential is older than the configured maximum password age, the system shall set the requirement with source "password age" before continuing. The age shall be measured from the last credential change. *(decision Q1 (c); FR-X-046)*
- RF-7.5 The system shall never set, clear or evaluate the requirement because of a tenant Owner's or Admin's action, a tenant policy, a membership change or an invitation; no tenant-side surface shall offer it. *(FR-IDN-049, FR-IDN-030)*
- RF-7.6 When the first factor is correct, the email is verified and a requirement is pending, the system shall present the forced change screen inside the login transaction, before any second-factor challenge and before any landing. *(FR-IDN-024, FR-IDN-045; decision Q2)*
- RF-7.7 While the forced change screen is shown, the identity shall be in the limited identity state of specification 001 RF-5: nothing but completing or abandoning the change shall be reachable, and no tenant, project, membership or personal data shall be exposed. *(FR-IDN-024; P3)*
- RF-7.8 If the first factor fails, then the system shall respond with the generic failure of specification 001 RF-2.2; the existence of a pending requirement shall never be revealed on a failed attempt. *(FR-IDN-020)*
- RF-7.9 If the email is not verified, then the system shall route to verification (specification 001 RF-3) and shall not show the forced change screen, even if a requirement is pending. *(FR-IDN-012)*
- RF-7.10 While a requirement is pending, the system shall not grant any session, tenant entry or authority to the identity on a sign-in that bypasses the screen; there shall be no path from first factor to challenge or landing that skips RF-7.6. *(FR-IDN-045; P6)*

### RF-8 · Forced change screen and completion

The person proves the current password and chooses a new one; then sign-in continues where it was. *Why:* requiring the current password keeps the change the holder's own act; continuing the transaction keeps everything they hold intact. *(Decision: Q2, Q5, Q6.)*

- RF-8.1 The system shall present the forced change screen with the "Required security update" marker, a current-password field, a new-password field, a confirmation field, each with a "show" toggle, an "Update & continue" action, a "Sign out safely" action, the Privacy and Terms routes and the language switch. *(FR-IDN-045, FR-X-043; R08)*
- RF-8.2 The system shall state on the screen that a security condition requires a password update before continuing (without naming the condition), that no administrator-created password is assumed, and that updating credentials never replays a pending Run request. *(FR-IDN-045, FR-IDN-049, FR-IDN-067; R08)*
- RF-8.3 The system shall not state which source raised the requirement; the screen is the same for compromise response, policy change and password age. *(FR-IDN-020; decision Q1)*
- RF-8.4 The system shall display the password rules of the current server policy beside the new-password field, as on the reset screen (RF-4.2), and apply the field rules of RF-4.4, RF-4.5 and RF-4.6. *(FR-IDN-011; R08)*
- RF-8.5 When the action is activated, the system shall verify the current password against the stored credential; a correct current password, a new password that satisfies the policy and differs from the current one, and a matching confirmation shall change the credential in one atomic operation. *(FR-IDN-003, FR-IDN-045; decision Q5)*
- RF-8.6 If the current password is wrong, then the system shall show a generic message on the current-password field that does not say whether the password was wrong, throttled or expired, shall clear the password fields, keep the user on the screen, and count the attempt as a failed sign-in attempt on both throttling dimensions and toward the login transaction's attempt limit. *(FR-IDN-020, FR-IDN-021; decision Q6)*
- RF-8.7 If the login transaction reaches its attempt limit or expires while on the forced change screen, then the system shall void the transaction and return the user to sign-in as in specification 001 RF-7; the requirement stays pending and is met again at the next correct first factor. *(FR-IDN-025; decision Q6)*
- RF-8.8 If the new password equals the current password, then the system shall reject it with a field message and shall not count it as a failed attempt. *(decision Q5)*
- RF-8.9 When the credential has changed, the system shall clear the pending requirement, revoke every other session and refresh family of the identity, invalidate every outstanding reset link, and send the security notification of RF-5.7 unless the requirement's source is a compromise response, which was notified when it was applied (RF-7.2). *(FR-IDN-042, FR-IDN-062; decision Q6, Q10)*
- RF-8.10 When the credential has changed, the system shall continue the same login transaction: if a factor is enrolled, present the TOTP challenge of specification 001 RF-6; otherwise complete sign-in and land as in specification 001 RF-8. *(FR-IDN-024, FR-IDN-025; decision Q2)*
- RF-8.11 The system shall not, as a consequence of a forced change: alter any membership, grant, draft or unsaved content of the identity, enroll, replace, disable or bypass the factor, lift a suspension, or replay any Run intent that was pending before sign-in. *(FR-IDN-043, FR-IDN-045, FR-IDN-067; P1)*
- RF-8.12 When "Sign out safely" is activated, the system shall void the login transaction and the limited identity state, leave the requirement pending, and return the user to the sign-in screen; nothing of the transaction shall be resumable. *(FR-IDN-025, FR-IDN-068; F-IDN-09)*
- RF-8.13 When the user leaves the screen by any other means (browser back, navigation, reload beyond the transaction), the system shall treat it as abandonment, exactly as specification 001 RF-5.4. *(FR-IDN-025)*

### RF-9 · Responsive surfaces

The drawn mobile counterparts are M15 MobileForgotPasswordPage (`431:4243`), M16 MobileResetPasswordPage (`431:4271`) and M17 MobileForceChangePasswordPage (`431:4300`); tablet follows the D01 responsive policy (`414:2645`); every statement is kept. *Why:* the security statements are the product's promise; a smaller viewport may not drop them. *(FR-X-051, FR-X-055.)*

- RF-9.1 The system shall present the request, reset, unavailable-link and forced change screens in a reflowed layout at mobile and tablet widths that meets WCAG 2.2 AA reflow, with every field, action, route and statement of RF-1, RF-4, RF-6 and RF-8 preserved in meaning. *(FR-X-051, FR-X-055; S-03)*
- RF-9.2 Every interactive element on those surfaces shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*
- RF-9.3 The system shall follow M15, M16 and M17 at mobile widths and the D01 responsive policy at tablet widths. *(FR-X-050; S-03)*

### RF-10 · Audit, notification and secrecy

Every outcome is recorded and nothing secret is ever written. *Why:* a reset is the event most worth reconstructing after an incident, and the token or password must never appear in that reconstruction. *(FR-X-044.)*

- RF-10.1 The system shall append to the audit log: each reset request (with the matched identity when one exists, and the fact that no token was issued when none was); each token issued; each token consumed; each unusable-link open or submission; each forced-change requirement set, with its source; each forced change completed; each wrong current-password attempt; each session revocation caused by this feature; each security notification sent, and each one deliberately not sent under RF-8.9; each restriction applied or lifted. *(FR-X-044)*
- RF-10.2 The audit record shall identify the identity when one is matched, the time and the calling context, and shall never contain the token, the link, the submitted address when no identity matched, a password or the message body. *(FR-X-030, FR-X-044)*
- RF-10.3 The system shall never include a token, a raw link, a password or a notification body in a log, an error, a trace, a client-visible problem or analytics; client-visible problems shall use safe problem codes with a correlation identifier. *(FR-X-030, FR-X-033)*
- RF-10.4 The system shall never write a token, a link, a password or the reset form's contents to browser storage; a token present in a page address shall not be retained beyond what rendering the form requires. *(FR-X-034)*
- RF-10.5 Where the audit log distinguishes a reset request whose address matched no identity, that fact shall be visible only inside the audit log and never in any response. *(FR-IDN-040)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Indistinguishability.** The request response for an active, unverified, restricted, erasing or unknown address, and for a throttled request, is identical in content and shape, with timing comparable within normal variance. The unavailable-link state is identical for unknown, malformed, expired, consumed and superseded tokens. | FR-IDN-040, FR-IDN-044 |
| RNF-2 | **No lockout.** No sequence of reset requests, link opens or forced-change attempts by a third party restricts the legitimate holder's sign-in, and no restriction of this feature outlasts its configured window. | FR-IDN-021, FR-IDN-040 |
| RNF-3 | **Single use.** A reset token can change a credential at most once, under concurrency, and only while it is the newest unexpired token of its identity. | FR-IDN-042 |
| RNF-4 | **Token secrecy.** Tokens are cryptographically random, purpose-bound, stored only as non-reversible hashes, delivered only to the verified address over the configured trusted origin, and never logged, exported, persisted in plaintext or written to browser storage. | FR-IDN-041, FR-X-030, FR-X-034 |
| RNF-5 | **Bounded workload.** Verification work and outgoing mail per unit of time are bounded regardless of request volume. | FR-IDN-021, FR-X-045 |
| RNF-6 | **Configured limits.** Token expiry, maximum password age, every threshold, delay and window are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-7 | **Fail closed.** An absent, invalid, expired, consumed or superseded token, an absent policy, or an absent configuration value blocks the effect; nothing is assumed or defaulted. | P6 |
| RNF-8 | **No authority before proof.** Neither the reset screen nor the forced change screen exposes tenant, project, membership or personal data, and neither creates a session or authority on its own. | FR-IDN-024; P3 |
| RNF-9 | **Factor integrity.** No path in this feature reads, changes, disables or bypasses the TOTP factor or the recovery codes. | FR-IDN-043, FR-IDN-030 |
| RNF-10 | **Revocation latency.** When a credential changes, every other session and refresh family of the identity is unusable from the moment the change is committed. | FR-IDN-042, FR-IDN-062 |
| RNF-11 | **Email as prerequisite.** The reset instructions, the compromise-response notice and the password-changed notification depend on the transactional email channel; its absence is an operational failure, not a silent skip. | FR-X-045 |
| RNF-12 | **Accessibility.** The request, reset, unavailable-link and forced change journeys meet WCAG 2.2 AA with automated and manual assistive-technology evidence: field errors associated with their fields, focus moved to messages, status conveyed by text, no focus stealing. | FR-X-040, FR-X-042 |
| RNF-13 | **Language.** All four surfaces and both email messages exist in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-14 | **Reflow.** Mobile and tablet reflows preserve every statement, gate and route of the desktop screens. | FR-X-051, FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Request for an address with no identity | Generic statement; no token, no message, nothing stored but throttle and audit (RF-2.1, RF-2.6). |
| LC-2 | Request for an unverified identity | Generic statement; nothing sent; the identity is not re-verified and not changed (RF-2.6; decision Q3). |
| LC-3 | Request for a restricted or under-erasure identity | Generic statement; nothing sent (RF-2.6; A-27). |
| LC-4 | Two requests in quick succession for the same active identity | Two generic statements; the second token invalidates the first link; only the newest works (RF-2.5; decision Q8). |
| LC-5 | Holder opens the first link after a second request was made | Unavailable state (RF-6.1); the newest link still works; a third party's later request has the same effect and the holder requests again (decision Q8). |
| LC-6 | Same link opened in two tabs, submitted in both | Exactly one submission changes the credential; the other receives the unavailable state (RF-5.1, RF-6.3). |
| LC-7 | Link opened, form left idle past expiry, then submitted | Unavailable state; the typed password is discarded (RF-6.3). |
| LC-8 | Mail scanner opens the link before the person | Nothing is consumed (RF-5.3); the person opens it later within its lifetime and resets normally. |
| LC-9 | New password equals the current one | Field rejection; token not consumed (RF-5.2). |
| LC-10 | Confirmation mismatch or policy failure | Field rejection; no submission; token untouched (RF-4.5, RF-4.6). |
| LC-11 | Link opened in a browser with an active session of another identity | The reset screen behaves as public (RF-4.8); the other identity's session is untouched by the open and by the success (decision Q9). |
| LC-12 | Link opened in a browser with an active session of the same identity | Same as LC-11 until success; success revokes that session with the others (RF-5.5; decision Q9). |
| LC-13 | Reset completed while a login transaction of the identity is open | The transaction is voided (RF-5.5); the next submission in it fails terminally (specification 001 LC-13). |
| LC-14 | Reset completed while a forced change is pending | Flag cleared (RF-5.6); the next sign-in does not show R08. |
| LC-15 | Reset for an identity with a factor enrolled | Success; next sign-in still raises the TOTP challenge (RF-5.9, RF-5.10). |
| LC-16 | Reset for an identity with no factor | Success; next sign-in completes without a challenge and lands by membership. |
| LC-17 | Reset for an identity whose only tenant is suspended | Success; suspension untouched; next sign-in lands on the tenant chooser showing the state (RF-5.9; specification 001 LC-16). |
| LC-18 | Reset for the sole Owner of a tenant | Success; ownership and membership untouched (RF-5.9). |
| LC-19 | Compromise response applied while the identity has three active sessions and an accepted Run in progress | All three sessions and families revoked (RF-7.2); the accepted Run continues under its run-scoped authority (FR-IDN-067; A-39); next sign-in meets R08. |
| LC-20 | Policy tightened; identity signs in with a now non-compliant password | Requirement set with source "policy change" at that first factor (RF-7.3); R08 shown before the challenge. |
| LC-21 | Password older than the maximum age; wrong password submitted | Generic sign-in failure; nothing revealed (RF-7.8); the requirement is set only on a correct first factor. |
| LC-22 | Requirement pending; email unverified | Verification route (RF-7.9); R08 not shown. |
| LC-23 | Wrong current password on the last permitted attempt of the transaction | Transaction voided; back to sign-in; requirement still pending (RF-8.7). |
| LC-24 | Transaction expires while the person types on R08 | Next submission is treated as terminal failure (RF-8.7); the typed passwords are discarded. |
| LC-25 | "Sign out safely" on R08 | Transaction voided; back to sign-in; requirement pending (RF-8.12). |
| LC-26 | Browser back from R08 to sign-in | Abandonment (RF-8.13); the screen cannot be reopened. |
| LC-27 | R08 completed by an identity with no factor and no membership | Credential changed; sign-in completes; landing routes to workspace creation (RF-8.10; specification 001 RF-8.2). |
| LC-28 | R08 completed by an Owner with a factor | Credential changed; TOTP challenge follows; Owner authority is derived only on tenant entry (RF-8.10). |
| LC-29 | Two first-factor sign-ins in parallel for a flagged identity | Each transaction shows R08; the first completed change clears the flag and voids the other transaction (RF-5.5 by analogy, RF-8.9). |
| LC-30 | Language switched on R03, R07 or R08 | Display changes; token, transaction, attempts and expiry unaffected. |
| LC-31 | Interface reloads during "updating" on R07 or R08 | The client re-submits the same attempt; the server answers with the recorded outcome of that attempt and never applies the change twice (F-X-02). |
| LC-32 | Email channel unavailable when a token is issued | The request still answers generically; the failure to send is an operational event in audit and monitoring, never a client-visible difference (RNF-11, RF-2.1). |
| LC-33 | Address throttled by a third party's flood; legitimate holder requests | Generic statement; no message during the window; a link is available again when the window elapses (RF-3.3, RF-3.4, RF-3.7; decision Q7). |
| LC-34 | Forced change completed after a compromise response | Requirement cleared, sessions revoked, links invalidated; no password-changed message — the RF-7.2 notice was the notification (RF-8.9; decision Q10). |
| LC-35 | Compromise response applied while the identity is signed out | Notice sent, requirement pending; met at the next sign-in (RF-7.2, RF-7.6). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **Voluntary password change from Security Settings** — the "password" area of FR-SET-020 is outside the implementation baseline (F-SET-03 covers the TOTP and Sessions areas only); until it is included, the only ways a password changes are the two flows here.
- **Sign-in, the first factor, the TOTP challenge and landing** — specification 001; this feature inserts one step into its transaction (RF-7.6) and otherwise defers to it.
- **Email verification** — specification 002; an unverified identity is routed there and never reset.
- **TOTP enrollment, recovery-code regeneration, factor replacement** — specification 003.
- **Controlled account recovery** for a lost authenticator — specification 005; the reset screens state that the reset is not that route.
- **Session lifetime, refresh, the expiry dialog** — specification 006; this feature only revokes sessions and families.
- **Session listing and individual revocation, the semantics of "sign out everywhere"** — specification 007; "Sign out safely" on R08 is defined here only as abandoning the transaction.
- **The surface or procedure by which the platform applies a compromise response** — internal, audited and out-of-band (scope §2); only the effect of the resulting requirement is specified (RF-7.2; A-44).
- **The password policy's content and its tightening** — supplied by the server as configuration; this feature only displays and applies it.
- **Password history** beyond "must differ from the current password" (decision Q5).
- **Administrator- or operator-initiated reset of another identity's password or factor** — there is no such capability (FR-IDN-049).
- **In-session forced change** (interrupting an active session in place) — not in this release (decision Q2; A-45).
- **Changing the email address**, SMS, phone or alternative recovery channels, security questions, federated identity, passkeys — not part of the product in this release.
- **Tablet-specific designs** of R03, R07 and R08 — none; the D01 responsive policy applies.
- **The choice and configuration of the transactional email channel** — an operational prerequisite, not a feature.
- **The content of the Help panel** — "Need help?" on the three screens opens N43 HelpShortcutsPanel (`430:4129`; S-17); the panel is the shell's, not this feature's.

---

## 7. Criteria of completion

Password reset and forced password change are complete when all of the following hold:

1. Every user story US-1 … US-8 can be walked end to end on the desktop designs and in the mobile and tablet reflows with no step outside this specification, and US-9 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-10 has at least one verification that fails before and passes after the feature exists.
3. Enumeration tests show no observable difference — content, shape, timing within normal variance — between requests for active, unverified, restricted, erasing, unknown and throttled addresses, and between unknown, malformed, expired, consumed and superseded links (RNF-1).
4. Throttle tests show progressive delay, bounded suppression, automatic lifting, independence from the sign-in throttles in both directions, and the absence of any permanent lockout (RNF-2, RF-3).
5. Link tests show: only the newest token works; opening never consumes; two concurrent submissions yield exactly one change; a submission after expiry or supersession lands in the unavailable state; the stored form of the token is a hash and the token appears in no log, audit entry, error or browser storage (RNF-3, RNF-4).
6. Revocation tests show that a reset and a forced change each revoke every other session and refresh family, void open login transactions, invalidate other links, and that an accepted Run is not cancelled (RNF-10, LC-19).
7. Factor tests show that neither flow reads, changes or bypasses the TOTP factor or recovery codes, and that the next sign-in after each flow still challenges an enrolled factor (RNF-9, LC-15).
8. Forced-change tests show each of the three sources sets the requirement, the screen appears only after a correct first factor and before the challenge, wrong current passwords are generic, counted and bounded, success clears the flag and continues the same transaction, abandonment leaves the flag pending, and no tenant-side action can set or clear it (RF-7, RF-8).
9. The product refuses to start when token expiry, maximum password age or any threshold, delay or window of this feature is absent or unbounded (RNF-6).
10. Audit entries exist for every event of RF-10.1 and contain no token, link, password or message body (RF-10.2).
11. The three messages — reset instructions, compromise-response notice and password-changed notification — are sent only to the verified address, exist in EN, ES and FR, and carry the statements of RF-2.8, RF-7.2 and RF-5.7; the password-changed notification is demonstrably not sent after a compromise-forced change (LC-34).
12. The request, reset, unavailable-link and forced change surfaces pass WCAG 2.2 AA with automated evidence and manual assistive-technology evidence, in EN, ES and FR, at desktop, tablet and mobile widths (RNF-12, RNF-13, RNF-14).
13. The register items this specification relies on — A-44, A-45, A-33 and A-27 in the architecture register, S-35, S-36, S-37, S-14, S-03 and S-17 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.
