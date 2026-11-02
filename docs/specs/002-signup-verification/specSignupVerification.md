# Specification 002 — Signup and email verification

**Feature** Local identity creation and email verification · **Version** 0.1 · **Date** 25 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (features F-IDN-01 and F-IDN-02 as the subject; F-IDN-03, F-IDN-04 and F-IDN-07 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P3, P6, P13, P14) → `requirements.md` (FR-IDN-001, 002, 003, 010, 011, 012, 013, 014, 015, 021, 022, 023, 024, 030, 049; FR-X-030, 033, 034, 040, 042, 043, 044, 045, 046, 053, 055) → the design of record, screens R04 SignupPage and N01 VerifyEmailPage (pending, confirmed and link-unavailable states), plus the D02 atlas entries "Email verified" and "This verification link is unavailable".

**Related specifications:** 001 Login (sign-in with an unverified email routes to the pending state of this feature; sign-in after confirmation is 001's job).

This document states **what** signup and verification do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global**: one principal, identified by a stable issuer and subject, holding only an email address, a display name and a status. It is created once and may later hold memberships in several tenants. It is not a tenant, not a membership and not a grant — signup therefore asks for nothing about an organisation, and no administrator activates anything.

Before an identity may do anything that touches a tenant — create a workspace or accept an invitation — its email address must be **verified**: the person must prove they receive mail at that address. Verification proves control of the address and nothing more; it grants no tenant or project access, accepts no invitation, and is not a substitute for the TOTP factor that Owners and Admins must enrol later.

The public landing routes "Create workspace" into this feature first and only then into workspace creation; it never creates or enters a tenant directly.

### 1.2 Objective

Let a person create a local identity with a name, an email address and a password, and prove control of that address, while:

- disclosing nothing about whether an address already belongs to an identity — on the form, on the pending screen, on resend and on the verification link;
- keeping every verification link short-lived, effective once and superseded by the next one, so that an old message can never verify an address;
- tolerating the mail scanners and previewers that open links automatically, so that they never strand the person on a dead end;
- creating no access of any kind as a result of verification.

### 1.3 Why it matters

Every tenant action is later attributed to an identity and delivered to its address — invitations, security notifications, recovery. If that address is not under the person's control, the product's recovery and notification promises are void. If signup leaks which addresses exist, it undoes the enumeration resistance built into sign-in, reset and resend. The feature is small, but every later feature rests on it.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 25 October 2026. The body refers to them as *(Decision: Qn)*.

| Q | Question | Decision |
|---|---|---|
| Q1 | Signup with the address of a verified identity | The same generic pending state as for a new address; nothing is created or changed; the existing identity receives a security message. |
| Q2 | Signup with the address of an unverified identity | The pending identity is replaced (name, password, acknowledgement); earlier links are invalidated; a fresh link is issued. |
| Q3 | When the address becomes verified | On opening the newest link; a re-open while it is still the newest shows the same confirmed state. |
| Q4 | Lifetime of an unverified identity | Removed after a configured window; the address becomes free again. |
| Q5 | "Use a different email" | Re-addresses the pending identity without re-entering the password; a collision changes nothing. |
| Q6 | Terms acknowledgement | The versions presented and the time are recorded; no re-acknowledgement in this release. |

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | new user | to create an identity with my name, email and a password | I can go on to create a workspace or accept an invitation |
| US-2 | new user | to see the password rules while I choose a password | I do not guess and fail |
| US-3 | new user | to be told to check my inbox and be able to resend the message | I am not stuck when the first message does not arrive |
| US-4 | new user who mistyped my address | to correct the address without starting over | the message reaches me |
| US-5 | new user | to confirm my address from the message and then be led to sign in | I know what to do next |
| US-6 | user who opens an old or already-used message | a clear, safe explanation and a way to request a new one | I am not left on a dead end and nothing about my account is revealed |
| US-7 | user who signed up but gave up | my unclaimed address to become free again after a while | I, or someone else who owns it, can use it later |
| US-8 | existing user whose address someone tried to register | to be told about the attempt | I can act if it was not me |
| US-9 | user who signed up on a phone or tablet | the screens to reflow and keep every statement | I can complete signup anywhere |
| US-10 | attacker (negative story) | — | I cannot learn which addresses exist, cannot verify an address I do not control, and cannot take over a pending identity |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

### RF-1 · Signup form

The signup screen collects name, email, password and one acknowledgement, and nothing else. *Why:* identity is separate from organisation; asking for more would imply that signup creates a tenant. *(FR-IDN-010)*

- RF-1.1 (Where the signup is opened from an invitation, specification 008 RF-4.1 applies in addition.) The system shall present a signup screen with a name field, an email field, a password field with a "show password" toggle, the current password rules, one checkbox "I agree to the Terms and acknowledge the Privacy notice", a "Create account & verify email" action, a route to sign in, and the interface-language switch (EN / ES / FR). *(FR-IDN-010, FR-IDN-011, FR-X-043; R04)*
- RF-1.2 The system shall state on the signup screen that the email must be verified before creating a workspace or accepting an invitation, that there is no activation queue and that no organisation data is required. *(FR-IDN-010, FR-IDN-012; R04)*
- RF-1.3 The system shall not ask for, accept or store any organisation, address, phone, role or connector information at signup. *(FR-IDN-001, FR-IDN-010)*
- RF-1.4 If any field is empty, the email is not a well-formed address, or the checkbox is not ticked when the action is activated, then the system shall identify each failing field with a message associated with that field and shall not submit. *(FR-X-042)*
- RF-1.5 When the user activates "show password", the system shall reveal the typed password in place until hidden again, and shall never persist the typed password anywhere.
- RF-1.6 The system shall trim surrounding whitespace from the name and email and shall match the email case-insensitively against existing identities.
- RF-1.7 The system shall offer, from the signup screen, the route "Already have an account? Sign in". *(R04)*

### RF-2 · Password rules

The rules come from the current policy and are shown where the password is chosen. *Why:* the client must never hard-code a policy that the product may tighten. *(FR-IDN-011)*

- RF-2.1 The system shall obtain the password rules from the current policy and display them on the signup screen as "supplied by the current policy". *(FR-IDN-011; R04)*
- RF-2.2 When a password is submitted, the system shall validate it against the current policy on the server, whatever the screen displayed. *(FR-IDN-011)*
- RF-2.3 If the password fails the policy, then the system shall name the unmet rule(s) in a message associated with the password field, and shall not create or change any identity. *(FR-X-042)*
- RF-2.4 The system shall store the password only in a non-reversible form; plaintext or reversible storage shall not occur. *(FR-IDN-003)*

### RF-3 · Identity creation and the generic response

Submitting the form always leads to the same place, whatever the address's history. *Why:* the form must not reveal which addresses are registered. *(Decisions: Q1, Q2.)*

- RF-3.1 When a valid form is submitted for an email that belongs to no identity, the system shall create an identity with status *unverified*, holding the name, the email, the non-reversible credential, and the acknowledgement record of RF-9; issue a verification link (RF-4); and show the pending state (RF-5). *(FR-IDN-001, FR-IDN-012)*
- RF-3.2 When a valid form is submitted for an email that belongs to a **verified** identity, the system shall create nothing and change nothing on that identity, shall send that identity a security message — at most once per configured window — stating that someone attempted to sign up with this address and offering the sign-in and password-reset routes, and shall show the same pending state as RF-3.1. *(Decision Q1)*
- RF-3.3 When a valid form is submitted for an email that belongs to an **unverified** identity, the system shall replace that identity's name, credential and acknowledgement record with the submitted ones, invalidate every earlier verification link for it, issue a new link, and show the same pending state as RF-3.1. *(Decision Q2)*
- RF-3.4 The system shall produce the same screen, the same text and comparable timing for RF-3.1, RF-3.2 and RF-3.3; nothing in the response shall disclose which case occurred. *(Decision Q1; FR-IDN-013 for the resend channel)*
- RF-3.5 The system shall create no tenant, membership, grant or project as part of signup. *(FR-IDN-002, FR-IDN-012)*
- RF-3.6 The system shall never link the new identity to any existing identity on the basis of a matching or similar address, and shall never join it to a tenant on the basis of its email domain. *(FR-IDN-015)*
- RF-3.7 If the email belongs to an identity whose status permits neither sign-in nor re-registration (see A-27 in §8), then the system shall behave as RF-3.2 without sending the security message.

### RF-4 · Verification link

Each link is unguessable, purpose-bound, short-lived, effective once, and superseded by the next; opening it again while it is still the newest shows the same result. *Why:* an old or forwarded message must never verify an address. *(FR-IDN-014)*

- RF-4.1 When a link is issued, the system shall generate an unguessable token bound to the identity and to the purpose "email verification", with a configured expiry; the system shall retain only a non-reversible form of the token. *(FR-IDN-014, FR-X-046)*
- RF-4.2 The system shall deliver the message containing the link only to the email address being verified, through the product's transactional email channel. *(FR-X-045)*
- RF-4.3 The link in the message shall point to the product's configured public address, never to an address derived from the request that triggered it.
- RF-4.4 When a new link is issued for an identity (signup, replacement, resend or re-addressing), the system shall invalidate every earlier link for that identity; only the newest link shall be usable. *(FR-IDN-014)*
- RF-4.5 The message shall state that only the newest message is usable, that opening the link does not sign the person in, and that it grants no workspace access.
- RF-4.6 The expiry of a link shall be validated configuration; the system shall refuse to start without it and shall never substitute a default. *(FR-X-046; P13)*

### RF-5 · Pending state

The screen after submission says "check your inbox" and offers exactly three ways on. *Why:* the person needs resend and correction, and nothing that discloses. *(FR-IDN-013, FR-IDN-014; N01 pending)*

- RF-5.1 The system shall show the pending state with: the heading "Check your inbox", the status "Verification request pending", the email entered, the generic texts "If this address is eligible, you'll receive verification instructions." and "Open the newest verification message to continue.", the actions "Resend verification email" and "Use a different email", and the route "Back to sign in". *(N01)*
- RF-5.2 The pending state shall state that resend responses do not disclose whether an account exists, that verification does not accept invitations or grant tenant access, that no administrator activation is needed, and that a person already verified in another tab should return to sign in. *(FR-IDN-012, FR-IDN-013; N01)*
- RF-5.3 The system shall show the same pending state when sign-in (specification 001, RF-3) is attempted with a correct password for an unverified identity. *(FR-IDN-012)*
- RF-5.4 While in the pending state, the system shall expose nothing about the identity beyond the email entered, and shall create no session. *(FR-IDN-012)*

### RF-6 · Resend

Resend issues a fresh link and answers generically. *Why:* the first message may be lost, and resend is a classic enumeration channel. *(FR-IDN-013)*

- RF-6.1 When "Resend verification email" is activated for an unverified identity, the system shall issue a new link (RF-4), invalidating earlier ones, and shall show the pending state unchanged except for a neutral "message requested" confirmation.
- RF-6.2 When "Resend verification email" is activated for an address with no unverified identity (none, verified, or expired), the system shall send nothing, change nothing, and show exactly the same confirmation as RF-6.1. *(FR-IDN-013)*
- RF-6.3 The system shall throttle resend per address and per calling network with configured thresholds and a bounded temporary restriction; while throttled it shall show the same confirmation as RF-6.1 and send nothing. The resend throttle shall be independent of the sign-in and reset throttles. *(Modelled on FR-IDN-021; FR-X-046)*
- RF-6.4 The system shall never state on the screen, or vary the timing of the response, according to whether a message was actually sent. *(FR-IDN-013)*

### RF-7 · Use a different email

A pending identity may be re-addressed once it is clear the address was wrong. *Why:* nothing has been proven for the old address, so re-addressing is safe and avoids a second signup. *(Decision Q5.)*

- RF-7.1 When "Use a different email" is activated from the pending state, the system shall let the person enter a new email address and submit it.
- RF-7.2 When a new address is submitted for an unverified identity and the new address belongs to no identity, the system shall replace the identity's email, invalidate every link for the old address, issue a new link to the new address, and show the pending state with the new address.
- RF-7.3 If the new address belongs to any existing identity other than the requester's own pending identity — verified or not — then the system shall change nothing and shall show the pending state with the new address, identically to RF-7.2; where the owning identity is verified, the security message of RF-3.2 shall be sent to it. A new address equal to the requester's own pending address is handled as a resend (RF-6.1).
- RF-7.4 If the identity is no longer unverified (already confirmed, or expired under RF-10), then the system shall show the pending state with the new address as in RF-7.2 and change nothing. This rule is evaluated before RF-7.3, so no security message is sent in that case.
- RF-7.5 The system shall treat re-addressing as a resend for the purposes of the throttle in RF-6.3.
- RF-7.6 The system shall never allow re-addressing of a verified identity through this feature; changing a verified address is not part of this feature (see §6).

### RF-8 · Confirmation

Opening the newest link verifies the address; opening it again shows the same result. *Why:* verification proves control of the mailbox and grants no access, so no further action is needed; tolerating a re-open means a mail scanner that opens the link first cannot strand the person. *(Decision Q3.)*

- RF-8.1 When a link is opened and it is the newest, unexpired link of an identity that is still unverified, the system shall, in one indivisible step, mark the identity *verified*, record the time and invalidate every other link for the identity; then show the confirmed state. *(FR-IDN-012, FR-IDN-014)*
- RF-8.2 When a link that has already verified its identity is opened again while it is still the newest link and unexpired, the system shall show the confirmed state again and change nothing. *(FR-IDN-014)*
- RF-8.3 (Where the browser still holds an invitation link, specification 008 RF-4.4 adds "Return to your invitation".) The confirmed state shall show: the heading "Email verified.", the status "Verification complete · Confirmed", the verified address, the statement that the person should sign in to view memberships, accept a valid invitation or create a workspace, the statement that verification alone grants no tenant or project access, the statement that TOTP enrollment is required for Owner / Admin roles and that email confirmation is not a substitute for MFA, and the action "Continue to sign in". *(FR-IDN-012, FR-IDN-023; N01 confirmed; D02 "Email verified")*
- RF-8.4 The system shall never sign the person in as a result of confirmation; "Continue to sign in" leads to the sign-in screen (specification 001). *(FR-IDN-012)*
- RF-8.5 If the opened link is expired, superseded by a newer link, invalidated because its identity was verified through another link, malformed, or belongs to no identity, then the system shall show the *link-unavailable* state: the heading "Request a fresh link", the status "Link unavailable · Expired or invalid", the statement that no account details are disclosed by this link, the actions "Request a new verification link" and "Back to sign in", and the statement that only the newest message is usable and that this does not change the password or reset MFA. *(FR-IDN-014; N01 unavailable; D02 "link is unavailable")*
- RF-8.6 The link-unavailable state shall be identical for every failure cause; neither the state nor its timing shall disclose whether an identity exists for the link. *(FR-IDN-014)*
- RF-8.7 When "Request a new verification link" is activated from the link-unavailable state, the system shall take the person to the pending state's resend (RF-6) for the address they enter, with the same generic behaviour.
- RF-8.8 Opening a link shall never create a session and shall change nothing beyond RF-8.1; in particular it shall never enrol, reset or bypass a TOTP factor, and never change the password.
- RF-8.9 The link shall work in any browser or device, not only the one used for signup.

### RF-9 · Terms acknowledgement

The acknowledgement is recorded with the identity. *Why:* it is the one legal fact signup captures; it must survive and be attributable. *(Decision Q6.)*

- RF-9.1 When an identity is created or replaced (RF-3.1, RF-3.3), the system shall record the version of the Terms and of the Privacy notice presented, and the time of acknowledgement.
- RF-9.2 The system shall require the checkbox to be ticked to submit; it shall never be pre-ticked.
- RF-9.3 The system shall take no action in this release when the Terms or Privacy notice change after acknowledgement; no re-acknowledgement is requested.
- RF-9.4 The signup screen and the pending, confirmed and unavailable states shall offer the Privacy and Terms routes as drawn. *(R04, N01; the routes open the documents the platform operator publishes outside the product, with an external-link cue (specification 017 Q3; register `122:2` §15); S-14)*

### RF-10 · Lifetime of an unverified identity

An address that is never verified is released after a configured window. *Why:* the product must not hold an address the person may not control, and a configured window is the only honest bound. *(Decision Q4.)*

- RF-10.1 The system shall remove an identity that is still unverified when a configured window since its creation (or its last replacement under RF-3.3) has elapsed; removal shall free the address and invalidate every link for it.
- RF-10.2 The window shall be validated configuration; the system shall refuse to start without it and shall never substitute a default. *(FR-X-046; P13)*
- RF-10.3 The system shall never remove a verified identity under this rule.
- RF-10.4 While the window has not elapsed, the address shall remain re-claimable only through RF-3.3 (replacement) or RF-7 (re-addressing by the same pending identity).

### RF-11 · What verification does not do

Verification proves control of an address, and nothing else. *(FR-IDN-012, FR-IDN-015)*

- RF-11.1 The system shall grant no tenant membership, project grant, workspace or invitation acceptance as a result of verification.
- RF-11.2 The system shall not accept an invitation on the person's behalf; a pending invitation is accepted only through its own flow after sign-in.
- RF-11.3 The system shall not enrol, reset or bypass any TOTP factor as a result of verification. *(FR-IDN-030)*
- RF-11.4 The system shall never allow a tenant Owner or Admin to verify, create or alter another person's identity; there is no administrator activation. *(FR-IDN-010, FR-IDN-049)*

### RF-12 · Audit and secrecy

Signup and verification events are recorded; secrets never are. *(FR-X-044, FR-X-030, FR-X-033, FR-X-034)*

- RF-12.1 The system shall append to the audit log: identity created, identity replaced, identity re-addressed, link issued (signup, replacement, resend, re-addressing), verification, link failure at open (by cause), security message sent to an existing identity, signup attempt on a non-registrable address, and unverified-identity removal.
- RF-12.2 The audit record shall never contain the password, the verification token or the contents of the message, and shall never record a submitted password for an address that this submission did not register (a verified or non-registrable collision). *(FR-IDN-022)*
- RF-12.3 The system shall never include a password or token in a log, error message, trace, address bar history beyond the link itself, or client-visible problem.

### RF-13 · Responsive signup

The drawn mobile counterparts are M10 MobileSignupPage (`431:4100`) for R04 and M11 MobileVerifyEmailPage (`431:4135`) for N01; tablet follows the D01 responsive policy. *(FR-X-050, FR-X-055)*

- RF-13.1 The system shall present the signup screen and every verification state in a reflowed layout at mobile and tablet widths that keeps every field, action, route and statement of the desktop design. *(FR-X-055)*
- RF-13.2 Every interactive element shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Indistinguishability.** For a given input, the response to signup, resend, re-addressing and link opening is identical in content, shape and comparable timing whether the address is new, unverified, verified or otherwise registered. | Decision Q1; FR-IDN-013, FR-IDN-014 |
| RNF-2 | **Link safety.** A token is unguessable, bound to identity and purpose, expiring, effective once (a re-open while it is still the newest shows the same confirmed state, RF-8.2) and superseded by any newer token; only its non-reversible form is retained. | FR-IDN-014 |
| RNF-3 | **Idempotent verification.** Opening the newest valid link verifies the address exactly once; opening it again shows the same confirmed state; no open ever creates a session or grants access. | Decision Q3 |
| RNF-4 | **Configured limits.** Link expiry, unverified-identity window, resend thresholds and restriction windows are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-5 | **Delivery prerequisite.** Verification depends on a working transactional email channel to the person's address; no other channel substitutes. | FR-X-045 |
| RNF-6 | **Accessibility.** Signup and all verification states meet WCAG 2.2 AA: errors associated with their fields, focus management, status by text not colour, bounded live announcements; evidence automated and manual assistive. | FR-X-040, FR-X-042 |
| RNF-7 | **Language.** All screens and messages exist in EN / ES / FR; the switch changes display only. The message is sent in the language selected at the time it was requested. | FR-X-043 |
| RNF-8 | **Secrecy.** Passwords and tokens never appear in logs, audit, errors, analytics or browser storage. | FR-X-030, FR-X-034 |
| RNF-9 | **Fail closed.** Any absent, invalid, expired or superseded token blocks confirmation; nothing is assumed. | P6 |
| RNF-10 | **No access.** No session, membership or grant exists at any point in this feature. | FR-IDN-012, FR-IDN-024 |
| RNF-11 | **Reflow.** The mobile and tablet reflow preserves every statement and action. | FR-X-055 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Signup with an address of a verified identity | Pending state shown; nothing created; security message to the owner (RF-3.2). |
| LC-2 | Signup with an address of an unverified identity, different name and password | Identity replaced; old links dead; new link sent (RF-3.3). |
| LC-3 | Old link (from before replacement) opened | Link-unavailable state (RF-8.5). |
| LC-4 | Two resends in quick succession (within the resend throttle) | Only the second link works; the first shows unavailable (RF-4.4). |
| LC-5 | Mail scanner opens the link before the person | Whichever open performs the verification, the person opens the link later — within its lifetime — and sees the confirmed state (RF-8.1, RF-8.2; the consumption path is register item A-33). |
| LC-6 | Person opens the link in two tabs at the same time | Exactly one open verifies; both tabs show the confirmed state (RF-8.1, RF-8.2). |
| LC-7 | Link opened after its expiry | Unavailable (RF-8.5); a new link must be requested. |
| LC-8 | Link opened after the identity was already verified through another link | Unavailable state; its "Back to sign in" route is the way on (RF-8.5). |
| LC-9 | Link opened after the unverified identity expired (RF-10) | Unavailable; "Request a new verification link" leads to generic resend that sends nothing; the person must sign up again. |
| LC-10 | Resend for an address that has no identity, or a verified one | Same confirmation; nothing sent (RF-6.2). |
| LC-11 | Resend flood from one network | Throttled; same confirmation; nothing sent; lifts after the window (RF-6.3). |
| LC-12 | Re-address to the same address | Treated as a resend (RF-7.3; the throttle of RF-7.5 applies). |
| LC-13 | Re-address to an address owned by a verified identity | Nothing changes; pending shown with the new address; owner notified (RF-7.3). |
| LC-14 | Re-address to an address owned by another unverified identity | Nothing changes; pending shown with the new address (RF-7.3). |
| LC-15 | Re-address after the identity was confirmed in another tab | Nothing changes; pending shown (RF-7.4). |
| LC-16 | Email in different case or with surrounding spaces | Same identity (RF-1.6). |
| LC-17 | Name consisting only of whitespace | Field-level validation; not submitted (RF-1.4). |
| LC-18 | Password that meets the displayed rules but fails a policy tightened since the screen loaded | Server rejects; unmet rule named (RF-2.2, RF-2.3). |
| LC-19 | Checkbox not ticked | Not submitted; field-level message (RF-1.4, RF-9.2). |
| LC-20 | Sign-in attempted with correct password before verifying | Pending state (RF-5.3; specification 001 RF-3). |
| LC-21 | Sign-in attempted with wrong password before verifying | Generic sign-in failure; verification status not revealed (specification 001 RF-3.3). |
| LC-22 | Link opened on a different device from the signup | Works (RF-8.9). |
| LC-23 | Interface language changed between signup and resend | The resent message uses the language current at resend (RNF-7). |
| LC-24 | Signup for an address whose identity is under erasure or otherwise non-registrable | Pending shown; nothing created; no message (RF-3.7; see A-27 in §8). |
| LC-25 | Address with a domain that matches an existing tenant's domain | No effect; no auto-join (RF-3.6). |

---

## 6. Out of scope

- **Sign-in**, including the route from sign-in to the pending state — specification 001.
- **TOTP enrollment** — specification 003; verification only states that it is required for Owner / Admin.
- **Workspace creation** — specification 012; "Create workspace" on the landing routes here first.
- **Invitation acceptance**, including a signup started from an invitation link and the return to that invitation — specification 008 (A-28 and S-24 in §8 cover the handoff).
- **Password reset** — specification 004; the security message of RF-3.2 only links to it.
- **Changing the email address of a verified identity** — not in the baseline (F-SET-02 only states that it is a separate verification flow).
- **Re-acknowledgement of changed Terms** — not in this release (decision Q6).
- **Federated identity, social signup, passkeys, invitation-only or domain-restricted signup** — not part of the product in this release.
- **Identity erasure request** — specification 015; erasure execution and data export — not in the baseline.
- **The choice or configuration of the transactional email channel** — an operational prerequisite, not a feature.

---

## 7. Criteria of completion

Signup and verification are complete when all of the following hold:

1. Every user story US-1 … US-9 can be walked end to end on the desktop design and in the mobile and tablet reflows with no step outside this specification, and US-10 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-13 has at least one verification that fails before and passes after the feature exists.
3. Enumeration tests show no observable difference — content, shape, timing within normal variance — across new, unverified, verified and non-registrable addresses for signup, resend, re-addressing and link opening (RNF-1).
4. Link tests show: only the newest link works; an expired, superseded, invalidated or malformed link shows the unavailable state; the first open verifies exactly once and a re-open is idempotent across two tabs; no open creates a session (RNF-2, RNF-3, LC-3…LC-8).
5. Replacement and re-addressing tests show that old links die, that collisions change nothing, and that verified identities can never be touched through this feature (RF-3.3, RF-7).
6. The unverified-identity window is enforced: removal frees the address and kills links; verified identities are never removed (RF-10).
7. The product refuses to start when link expiry, the unverified window or any resend threshold is absent or unbounded (RNF-4).
8. Audit entries exist for every event in RF-12.1 and contain no password, token or message body (RNF-8).
9. The security message of RF-3.2 is sent for a verified collision and not sent for a non-registrable address (LC-1, LC-24).
10. Signup, pending, confirmed and unavailable states pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths (RNF-6, RNF-7, RNF-11).
11. Every register item listed in §8 is resolved, or explicitly deferred by the product owner, before the feature is declared done.

---

## 8. Register items raised by this specification

Questions this specification could not answer from the design are not recorded here. Each is tracked in the Figma design file, in the architecture register (`122:2` §14) or the D03 Screen decisions ledger (`148:4`), for the architecture or design team to resolve under the identifier below.

| Register | Item | Question | Assumption built until resolved (plan §6) |
|---|---|---|---|
| Architecture | A-27 | Identity status values (RF-3.7, LC-24) and display-name constraints (RF-1) | `UNVERIFIED` is replaced, `ACTIVE` receives the notice, `RESTRICTED` and `ERASING` change nothing; name bounds from configuration |
| Architecture | A-28 | Signup started from an invitation: context handoff and return route | Decided by specification 008 Q3 (27 October 2026): the invitation link itself is the return route; signup from the invitation page sends the link secret with the form and the server refuses any address other than the invited one (plan 008 D11); a signup without a link stays the plain signup of this specification |
| Architecture | A-33 | How the verification link is consumed (RF-8.1) | The verification route submits the token when it loads |
| Architecture | A-34 | Proof of control for re-addressing a pending identity (RF-7) | A pending reference issued at signup and at the sign-in handoff |
| Architecture | A-35 | Persistence access API and schema-migration tool | Placeholders in the plan and tasks |
| D03 ledger | S-14 | Email templates and legal links (here: the destinations of the Privacy and Terms routes, RF-9.4) | External operator documents with an external-link cue (017 Q3; `122:2` §15); the messages of this feature are rows of the D04 inventory (`437:4431`) |
| D03 ledger | S-17 | "Need help?" / "Help" destination on R04 and N01 | Rendered; opens N43 HelpShortcutsPanel (`430:4129`) |
| D03 ledger | S-23 | Security message to an existing identity on a signup collision (RF-3.2, RF-7.3) | The D04 copy (`437:4431`, M-nn rows of this feature) |
| D03 ledger | S-24 | Return to the invitation after confirmation | Decided by specification 008 Q3: the confirmed state offers "Return to your invitation" while the browser still holds the link (008 RF-4.4); otherwise "Continue to sign in" only |
| D03 ledger | S-25 | Where the new address is entered for "Use a different email" (RF-7) | Inline field under the pending state; unavailable after a reload |
| D03 ledger | S-26 | Sign-in or signup screen opened with a valid session (here: signup) | The form is shown; the session is untouched |
| D03 ledger | S-27 | Pre-filling the sign-in form from the confirmed state (RF-8.4) | The sign-in form opens empty |
