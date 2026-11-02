# Specification 001 — Login

**Feature** Sign-in (first factor, TOTP challenge and landing) · **Version** 0.1 · **Date** 25 October 2026 · **Status** Draft for review · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (features F-IDN-03 and F-IDN-04 as the subject; F-IDN-01, F-IDN-02, F-IDN-05, F-IDN-06, F-IDN-07, F-IDN-08, F-IDN-09, F-IDN-14 and F-IDN-15 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P1, P3, P6, P13, P14) → `requirements.md` (FR-IDN-003, 011, 012, 013, 014, 020, 021, 022, 023, 024, 025, 026, 027, 028, 030, 040, 043, 046, 048, 049, 067, 069, 110, 111; FR-X-030, 033, 034, 040, 042, 043, 044, 046, 050, 051, 053, 055) → the design of record, screens R02 LoginPage `148:290`, M02 MobileLoginPage `161:51247`, R05 MfaChallengePage `148:179` and the D02 atlas `159:32045`.

This document states **what** Login does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

ASpectra is a multi-tenant product in which a single **global identity** (email, display name, status) is separate from tenant membership and project grants. Authority is never claimed by the browser; it is derived on the server after the identity has been established. Login is the first step of every authenticated journey and the only entry into the product for an existing identity: the public landing offers exactly two actions, *create workspace* and *sign in*.

Sign-in is **local**: a verified email address and a password. No federated provider, social login, passkey or trusted device exists in this release, and the sign-in screens say so.

A successful first factor does **not** make an authenticated session. It creates a *limited identity state* that can only complete, or abandon, a second-factor challenge. TOTP is mandatory for Owners and Admins; a tenant policy may also require it for Members. Because the identity may belong to several tenants and has not chosen one at sign-in, the product applies one simple rule: **an identity that has an enrolled factor is always challenged**.

### 1.2 Objective

Let a person with an existing, verified ASpectra identity prove who they are and reach the right place — their workspace, the tenant chooser, or workspace creation — while:

- disclosing nothing about which accounts exist, which are restricted, or why an attempt failed;
- making credential guessing impractical without ever creating a lockout an attacker can trigger;
- never granting tenant or project authority before the identity, and its second factor when enrolled, are both proven;
- preserving the product's three-decision model: signing in never accepts a mapping, approves a bundle or launches a Run, and never replays a pending Run intent.

### 1.3 Why it matters

Every other feature assumes a correctly established identity. A weak or leaky sign-in undermines tenant isolation (the first of the two tenancy gates) and the human-authority model the product is built on. A sign-in that is safe but hostile to users (permanent lockouts, unexplained dead ends, unreachable recovery) costs the trust the product sells.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 25 October 2026. The body refers to them as *(Decision: Qn)*.

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of "Login" | First factor, TOTP challenge and landing. Enrollment, recovery, reset, signup and session management are separate features reached only as exits. |
| Q2 | When the TOTP challenge is triggered | Whenever a factor is enrolled, at every sign-in, regardless of tenant or role. An identity without a factor passes the first factor only; Owner / Admin authority is withheld on tenant entry until enrollment. |
| Q3 | Correct password, unverified email | Route to the email-verification pending state; no session or limited state is created. |
| Q4 | Where a completed sign-in lands | By membership count: none → workspace creation; exactly one active membership in an active tenant → that tenant; otherwise the tenant chooser. |
| Q5 | What a throttled or restricted attempt shows | The same generic failure as any other failure; throttling is felt only as delay. |
| Q6 | Terminal challenge failure | The transaction is voided and the user returns to sign-in for a full restart; the failure counts toward abuse protection. |

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | returning user with a verified identity | to sign in with my email and password | I can reach my workspace |
| US-2 | Owner or Admin with TOTP enrolled | to be asked for my authenticator code right after my password | privileged authority is never granted on a password alone |
| US-3 | user who has lost their authenticator | to use one of my saved recovery codes at the challenge, or to start controlled recovery | I am not locked out of my own identity |
| US-4 | user who signed up but never verified my email | to be sent to the verification step when I sign in | I understand what is pending instead of hitting a dead end |
| US-5 | user who belongs to several workspaces | to choose the workspace after signing in | I never land in the wrong tenant |
| US-6 | user with no workspace yet | to be led to workspace creation after signing in | the product does not strand me on an empty screen |
| US-7 | user who mistypes a credential | a clear, non-blaming failure message and the chance to try again | I am not locked out, and I am not told which part was wrong |
| US-8 | user on a phone | a sign-in screen designed for my viewport that keeps every security statement | I can sign in anywhere without losing information |
| US-9 | user who forgot my password | a route to password reset from the sign-in screen | I can recover without contacting anyone |
| US-10 | security-conscious user | the sign-in screen to tell me that a password challenge is not yet a session and that there is no remembered-device bypass | I know what the product promises |
| US-11 | attacker (negative story) | — | I cannot learn whether an email has an account, cannot lock a victim out permanently, and cannot replay a code or a challenge |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

### RF-1 · Sign-in form

The sign-in screen collects the identity's email address and password and nothing else. *Why:* local verified identity is the only entry method; collecting anything more would imply a capability that does not exist.

- RF-1.1 The system shall present a sign-in screen with an email field, a password field, a "show password" toggle, a "Sign in" action, a route to password reset, a route to account creation, and the interface-language switch (EN / ES / FR). *(FR-IDN-020, FR-X-043; R02, M02)*
- RF-1.2 The system shall state on the sign-in screen that Owner and Admin access requires TOTP, that a factor challenge is not yet a fully authenticated session, and that there is no trusted-device bypass. *(FR-IDN-023, FR-IDN-024, FR-IDN-025; R02, M02)*
- RF-1.3 The system shall state on the sign-in screen that only a local verified email identity is accepted and that no federated, social or passkey sign-in is active. *(FR-IDN-020; M02)*
- RF-1.4 The system shall offer, from the sign-in screen, a route to lost-authenticator recovery and shall state that password reset is separate from lost-factor recovery and never disables MFA. *(FR-IDN-043; FR-IDN-046 path (a); M02)*
- RF-1.5 When the user activates "show password", the system shall reveal the typed password in place until it is hidden again, and shall never persist the typed password anywhere.
- RF-1.6 If the email or password field is empty when "Sign in" is activated, then the system shall identify the empty field with a message associated with that field, and shall not submit the attempt. *(FR-X-042)*
- RF-1.7 The system shall present the password rules only where a password is being set, never on the sign-in screen. *(FR-IDN-011)*

### RF-2 · Credential verification

The product verifies the submitted email and password against the identity store and answers in a way that reveals nothing about the account. *Why:* account enumeration is the first step of most credential attacks.

- RF-2.1 When a sign-in attempt is submitted, the system shall verify the submitted password against the stored, non-reversible credential of the identity holding that email address. *(FR-IDN-003, FR-IDN-020)*
- RF-2.2 The system shall produce the same failure response — same text, same shape, comparable timing — for an unknown email, a wrong password and a restricted identity. *(FR-IDN-020)*
- RF-2.3 When the submitted email matches no identity, the system shall still perform a credential verification of equivalent cost, so that the response time does not reveal the absence of the account. *(FR-IDN-020)*
- RF-2.4 If the submitted email matches no identity, then the system shall not create an identity and shall not store the submitted password in any form. *(FR-IDN-022)*
- RF-2.5 When an attempt fails, the system shall show the generic failure text ("We couldn't sign you in. Check your details and try again. This response does not confirm whether an account exists."), keep the email entered, clear the password field, and move focus to the message. *(FR-IDN-020, FR-X-042; M02)*
- RF-2.6 The system shall treat the email address case-insensitively and ignore surrounding whitespace when matching an identity.
- RF-2.7 If the identity's status permits neither sign-in nor the verification branch of RF-3 (for example, restricted or under erasure), then the system shall respond exactly as for a wrong password. An unverified identity is governed by RF-3, not by this rule. *(FR-IDN-020; A-27)*

### RF-3 · Unverified email

An identity that proves its password but whose email has never been verified is sent to verification, not into the product. *Why:* verification is a precondition of every tenant action; a correct password proves control of the account, so routing to verification discloses nothing the user does not already know. *(Decision: Q3.)*

- RF-3.1 When the password is correct and the identity's email is not verified, the system shall not create any session or limited identity state, and shall route the user to the email-verification screen in its *pending* state, with resend available. The handoff may carry a pending reference that authorises re-addressing of the unverified identity (specification 002, RF-7; A-34); such a reference is neither a session nor a limited identity state. *(FR-IDN-012, FR-IDN-013, FR-IDN-014)*
- RF-3.2 While the email is unverified, the system shall not raise a second-factor challenge, even if a factor is enrolled.
- RF-3.3 When the password is wrong and the email is unverified, the system shall respond with the generic failure of RF-2.2; verification status shall never be revealed on a failed attempt.

### RF-4 · Abuse protection

Repeated attempts are slowed down and, when necessary, temporarily restricted, without ever telling the caller that this is happening. *Why:* bound the cost of guessing while denying attackers both information and a way to lock out a victim. *(Decision: Q5.)*

- RF-4.1 The system shall throttle sign-in attempts on two independent dimensions: per target account and per calling network / risk signal. *(FR-IDN-021)*
- RF-4.2 When attempts on a dimension exceed its configured threshold, the system shall apply a progressive delay before answering, and beyond a further configured threshold shall place that dimension under a bounded temporary restriction during which every attempt fails. *(FR-IDN-021, FR-X-046)*
- RF-4.3 While an account or network is throttled or restricted, the system shall return the same generic failure text as RF-2.5; the restriction shall never be named, and no remaining wait time shall be shown. *(FR-IDN-020, FR-IDN-021)*
- RF-4.4 The system shall lift every restriction automatically when its configured window elapses; a permanent lockout shall not exist and no sequence of attempts by a third party shall deny the legitimate holder indefinitely. *(FR-IDN-021)*
- RF-4.5 The system shall count a terminally failed second-factor challenge (RF-7) as a failed attempt on both dimensions. *(FR-IDN-021)*
- RF-4.6 The system shall bound the total verification work it performs per unit of time so that a flood of attempts cannot exhaust it. *(FR-IDN-021)*
- RF-4.7 The system shall keep the sign-in throttles independent of the password-reset throttles; a reset request shall never count as a sign-in failure and shall never lock the account. *(FR-IDN-040)*
- RF-4.8 Every threshold, delay and restriction window shall be validated configuration; the system shall refuse to start with an absent or unbounded value and shall never substitute a default. *(FR-X-046; P13)*

### RF-5 · Limited identity state

A correct first factor yields a state that can only finish or abandon the challenge. *Why:* a password alone must never confer customer authority (P3, FR-IDN-024).

- RF-5.1 When the password is correct, the email is verified and a TOTP factor is enrolled, the system shall create a limited identity state bound to a new login transaction and shall present the challenge screen. *(FR-IDN-024, FR-IDN-025; R05)*
- RF-5.2 While in the limited identity state, the system shall permit only: submitting an authenticator code, submitting a recovery code, starting controlled account recovery, and abandoning the transaction — plus, on its own screen, the actions of the specification that opened the state (004 RF-7.7 for the forced change, 005 RF-2.5 for the replacement enrollment, 006 RF-8.4 for re-authentication). It shall expose no tenant, project, membership or personal data. *(FR-IDN-024)*
- RF-5.3 The system shall give each login transaction an identity binding, a purpose (sign-in), a configured expiry and a configured attempt limit. *(FR-IDN-025, FR-X-046)*
- RF-5.4 When the user abandons the challenge (leaves the screen or returns to sign-in), the system shall void the transaction; a voided transaction shall never be resumable. *(FR-IDN-025)*
- RF-5.5 When the password is correct, the email is verified and **no** factor is enrolled, the system shall complete sign-in without a challenge and proceed to landing (RF-8). *(Decision: Q2.)*

### RF-6 · TOTP challenge

The second factor is a time-based one-time code from the enrolled authenticator, or a single-use recovery code. *Why:* this is the only way to make Owner/Admin authority resistant to a stolen password. *(Decision: Q2 — any enrolled factor is always challenged.)*

- RF-6.1 The system shall present the challenge screen with a code field, a "Verify & continue" action, a "Use a recovery code" alternative, an entry to controlled account recovery, and the statement that the challenge does not create a fully authenticated session until verification succeeds and that there is no remembered-device bypass. *(FR-IDN-025; R05)*
- RF-6.2 The system shall accept a complete code pasted in one action as well as typed digit by digit. *(FR-IDN-025)*
- RF-6.3 When a code is submitted, the system shall show a "verification in progress" state, disable resubmission until the answer arrives, and then either complete sign-in or show a failure. *(R05)*
- RF-6.4 When a valid, unused authenticator code is verified within the transaction's expiry and attempt limit, the system shall mark the transaction complete and proceed to landing (RF-8).
- RF-6.5 The system shall reject any code whose time-step is at or before the last accepted time-step of this factor, within the configured time tolerance, so that an observed code cannot be replayed. *(FR-IDN-025; A-32)*
- RF-6.6 If the code is wrong, then the system shall decrement the remaining attempts, show a generic "code not accepted" message that does not say whether the code was wrong, expired or already used and does not disclose the remaining attempt count, and keep the user on the challenge. *(FR-IDN-025; D02 `159:32045` "We couldn't verify the code")*
- RF-6.7 When the user chooses "Use a recovery code", the system shall accept one unused recovery code of the identity; on success the code shall be consumed permanently and the transaction completed. *(FR-IDN-025, FR-IDN-028)*
- RF-6.8 If a submitted recovery code is unknown or already consumed, then the system shall treat it as a wrong code (RF-6.6). *(FR-IDN-028)*
- RF-6.9 When the user chooses controlled account recovery, the system shall void the transaction and route to the controlled-recovery entry. Recovery shall not bypass suspension, revoked membership or required MFA. *(FR-IDN-025, FR-IDN-048)*
- RF-6.10 The system shall never offer, store or honour a "remember this device", "skip next time" or equivalent bypass. *(FR-IDN-025, FR-IDN-069)*
- RF-6.11 The system shall never display the TOTP secret or any enrollment material on the challenge screen. *(FR-IDN-027)*

### RF-7 · Terminal challenge failure

An exhausted or expired challenge sends the user back to the start. *Why:* a fresh transaction must begin from a fresh first factor so that neither state nor codes can be carried over. *(Decision: Q6.)*

- RF-7.1 If the attempt limit is reached, then the system shall void the login transaction, discard the limited identity state and return the user to the sign-in screen.
- RF-7.2 If the transaction expires before a valid code is verified, then the system shall act as in RF-7.1 on the next submission or screen interaction.
- RF-7.3 When returning the user to sign-in after a terminal failure, the system shall require email and password again; no part of the previous transaction shall be reused.
- RF-7.4 The system shall not show, on return to sign-in, whether the failure was due to attempts or to time.
- RF-7.5 The system shall count the terminal failure toward abuse protection (RF-4.5).

### RF-8 · Landing after sign-in

Where the user goes depends only on their own memberships. *Why:* the chooser exists for ambiguity, workspace creation for emptiness, and a single workspace deserves no detour. *(Decision: Q4.)*

- RF-8.1 When sign-in completes (RF-5.5 or RF-6.4 / RF-6.7), the system shall establish the identity's session — an identity session that carries no tenant or project authority, which is derived only on tenant entry — and determine landing from the identity's memberships only. *(FR-IDN-024; decision Q4; FR-IDN-110 as the data source)*
- RF-8.2 If the identity holds no membership of any state, then the system shall route to workspace creation, which itself requires TOTP enrollment before a workspace can be created (F-IDN-14; specification 012). *(decision Q4)*
- RF-8.3 If the identity holds exactly one membership and it is active and its tenant is active, then the system shall enter that tenant directly.
- RF-8.4 If the identity holds more than one membership, or a single membership that is not active, or a single membership whose tenant is not active, then the system shall route to the tenant chooser, which shows each membership's role and state. *(FR-IDN-110, FR-IDN-111)*
- RF-8.5 When entering a tenant, the system shall apply that tenant's authority rules — including withholding Owner / Admin authority, and Member authority where tenant policy requires it, until TOTP enrollment is complete. This gate belongs to tenant entry, not to Login, and Login shall never bypass it. *(FR-IDN-023, FR-IDN-026, FR-IDN-030)*
- RF-8.6 The system shall never, as part of landing, accept a mapping, approve a bundle, launch a Run or replay a Run intent that was pending before sign-in. *(FR-IDN-067; P1)*
- RF-8.7 Landing shall grant visibility only of what the identity is currently authorised to see; it shall never restore access that a membership change has since removed. *(FR-IDN-067)*

### RF-9 · Responsive sign-in

The mobile counterpart is a genuine reflow that keeps every statement. *Why:* the security statements are part of the product's promise, not decoration. *(FR-X-050, FR-X-051)*

- RF-9.1 The system shall present the mobile sign-in screen as designed (M02), with the same fields, actions, routes and statements as the desktop screen.
- RF-9.2 The mobile sign-in screen shall preserve every statement of RF-1.2, RF-1.3 and RF-1.4 verbatim in meaning. *(FR-X-051)*
- RF-9.3 Every interactive element on the mobile sign-in screen shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*
- RF-9.4 The challenge screen shall follow its drawn mobile counterpart M12 MobileMfaChallengePage (`431:4162`); at tablet widths it follows the D01 responsive policy and meets the same accessibility bar. *(FR-X-050, FR-X-055)*

### RF-10 · Audit and notification

Sign-in outcomes are recorded. *Why:* security events must be reconstructable, and the password must never leak into the record. *(FR-X-044)*

- RF-10.1 The system shall append to the audit log: each successful sign-in, each failed first-factor attempt (without the submitted password and without stating the failure reason beyond "failed"), each challenge outcome, each recovery-code use, and each restriction applied or lifted.
- RF-10.2 The audit record shall identify the identity when one is matched, the time and the calling context, and shall never contain the password, the code or the recovery code.
- RF-10.3 The system shall never include any credential, code or secret in a log, error message, trace or client-visible problem. *(FR-X-030, FR-X-033)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Indistinguishability.** Failure responses for unknown email, wrong password, restricted identity and throttled attempt are identical in content and shape, and their timing is comparable within normal variance. | FR-IDN-020 |
| RNF-2 | **No lockout.** No sequence of requests by a third party can deny a legitimate holder sign-in beyond the configured temporary window. | FR-IDN-021 |
| RNF-3 | **Bounded workload.** Verification cost per unit of time is bounded regardless of attempt volume. | FR-IDN-021 |
| RNF-4 | **Configured limits.** Every expiry, attempt limit, delay, threshold and restriction window is validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-5 | **Accessibility.** The sign-in and challenge journey meets WCAG 2.2 AA: field errors programmatically associated with their fields, focus moved to messages, status conveyed by text not colour, no focus stealing by live announcements. Evidence is both automated and manual assistive testing on the designated screen readers. | FR-X-040, FR-X-042 |
| RNF-6 | **Language.** Sign-in and challenge are available in EN / ES / FR; the switch changes display only. | FR-X-043 |
| RNF-7 | **Secrecy.** Passwords, codes and recovery codes never appear in logs, audit, errors, URLs, browser storage or analytics. | FR-X-030, FR-X-034, FR-IDN-028 |
| RNF-8 | **Fail closed.** Any absent, invalid or expired element of the flow (transaction, code, configuration) blocks progress; nothing is assumed or defaulted. | P6 |
| RNF-9 | **No authority before proof.** No tenant or project data is reachable in the limited identity state. | FR-IDN-024; P3 |
| RNF-10 | **Reflow.** The mobile counterpart preserves every statement and gate of the desktop screen. | FR-X-051 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Correct password, unverified email, factor enrolled | Route to verification (RF-3.1); no challenge (RF-3.2). |
| LC-2 | Correct password for a restricted / under-erasure identity | Generic failure (RF-2.7); the attempt still counts for throttling. |
| LC-3 | Same code submitted twice within tolerance | Second submission rejected as a wrong code (RF-6.5). |
| LC-4 | Code valid but transaction expired | Terminal failure (RF-7.2); back to sign-in; the code is not consumed and not revealed as valid. |
| LC-5 | Valid code on the last permitted attempt | Accepted; sign-in completes. |
| LC-6 | Wrong code on the last permitted attempt | Terminal failure (RF-7.1). |
| LC-7 | Recovery code already consumed | Treated as a wrong code (RF-6.8); counts as an attempt. |
| LC-8 | Last remaining recovery code used | Sign-in completes; the identity now has zero recovery codes (see A-29 in §8). |
| LC-9 | Two challenge screens open for the same transaction (two tabs) | One success completes the transaction; the other tab's later submission is rejected because the transaction is already complete or voided. |
| LC-10 | Two first-factor sign-ins in parallel for the same identity | Each creates its own transaction; neither affects the other's attempts, but both count toward the account throttle. |
| LC-11 | Password correct while the account throttle is active | Generic failure (RF-4.3); the correct password is not honoured until the window elapses. |
| LC-12 | Network restriction active, different account | Generic failure for every account from that network (RF-4.1, RF-4.3). |
| LC-13 | Password changed or reset while a transaction is open | The open transaction is voided (sessions and families are revoked by the reset); the next submission fails terminally. |
| LC-14 | Factor replaced or removed by the identity while a transaction is open | The transaction is voided; sign-in restarts. |
| LC-15 | Membership revoked between first factor and landing | Landing uses current memberships (RF-8.1, RF-8.7); a single revoked membership is a non-active membership and routes to the tenant chooser, which lists no revoked membership and shows its "No membership for a workspace?" block (RF-8.4; specification 009 Q3, specification 013 RF-3.2); only an identity with no membership record at all routes to workspace creation. |
| LC-16 | Single membership in a suspended tenant | Tenant chooser showing access state only (RF-8.4; FR-IDN-111). |
| LC-17 | Single active membership, tenant deleting | Tenant chooser (RF-8.4). |
| LC-18 | Owner with no factor enrolled signs in | No challenge (RF-5.5); on entering the tenant, authority is withheld until enrollment (RF-8.5). |
| LC-19 | Email with different letter case or surrounding spaces | Matched as the same identity (RF-2.6). |
| LC-20 | Email field contains a value that is not an address | Field-level validation message; no attempt submitted (RF-1.6). |
| LC-21 | User changes interface language on the challenge screen | Language changes; the transaction, attempts and expiry are unaffected. |
| LC-22 | Browser back from the challenge to sign-in | Transaction voided (RF-5.4); the challenge cannot be reopened. |
| LC-23 | Paste of a code with spaces or separators | Separators are ignored; the digits are submitted as one complete code (RF-6.2). |
| LC-24 | Interface reloads during "verification in progress" | The answer is lost on the client; the client re-submits the same attempt and the server answers with the recorded outcome of that attempt — it never evaluates the code a second time (F-X-02). |
| LC-25 | No memberships and no factor enrolled | No challenge (RF-5.5); landing routes to workspace creation, which routes to TOTP enrollment before creating (RF-8.2). |

---

## 6. Out of scope

The following are separate features. Login reaches them only through the routes named above and specifies nothing about their behaviour.

- **TOTP enrollment** (three-step wizard, R39) and the Owner / Admin enrollment gate on tenant entry — F-IDN-04 (the gate) and F-IDN-14 (the creation prerequisite).
- **Recovery-code regeneration, reveal and factor replacement** — F-IDN-05.
- **Controlled account recovery** beyond its entry link — F-IDN-07.
- **Password reset and forced password change** — F-IDN-06. Forced change is a workspace-side state and never a Login outcome.
- **Signup and email verification** beyond routing to the pending state — F-IDN-01 / F-IDN-02.
- **Session lifetime, refresh, expiry dialog and re-authentication from an expired session** (R34) — F-IDN-08. Whether the landing rule of RF-8 applies to that re-authentication is register item A-31 (§8).
- **Logout, session listing and revocation** — F-IDN-09.
- **Invitation acceptance** — F-IDN-10, including a sign-in triggered from an invitation link.
- **Tenant chooser contents, workspace creation and tenant suspension screens** — F-IDN-15, F-IDN-14, F-IDN-16; Login only routes to them.
- **Tenant policy that requires MFA for Members** — F-SET-05; its enforcement is on tenant entry.
- **Federated identity, social login, passkeys, security keys, "remember me", trusted devices, SMS or email one-time codes** — not part of the product in this release; the screens state this.
- **Return-to-requested-page after sign-in** (deep links) — not supported by this specification; landing follows RF-8 only. The single exception is the invitation link held in the same tab (specification 008 RF-4.4), which re-opens the invitation after a completed sign-in; it is not a generic deep link.
- **Operator or administrator actions on another identity's credentials or factors** — there is no such capability (FR-IDN-049).

---

## 7. Criteria of completion

Login is complete when all of the following hold:

1. Every user story US-1 … US-10 can be walked end to end on the desktop and mobile designs with no step outside this specification, and US-11 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-10 has at least one verification that fails before and passes after the feature exists.
3. Enumeration tests show no observable difference — content, shape, timing within normal variance — between unknown email, wrong password, restricted identity and throttled attempt (RNF-1).
4. Throttle tests show progressive delay, bounded temporary restriction, automatic lifting, and the absence of any permanent lockout (RNF-2), with the reset throttle demonstrably independent (RF-4.7).
5. Challenge tests show replay rejection of an accepted code, rejection of a consumed recovery code, correct attempt-limit and expiry behaviour, and the full restart on terminal failure (RF-6, RF-7); two-tab and parallel-transaction cases LC-9 and LC-10 behave as stated.
6. Landing tests cover zero, one active, one inactive and multiple memberships, plus the revoked-between-steps case LC-15.
7. The limited identity state is shown to expose no tenant, project, membership or personal data (RNF-9).
8. The product refuses to start when any Login threshold, delay, expiry or attempt limit is absent or unbounded (RNF-4).
9. Audit entries exist for every outcome in RF-10.1 and contain no credential, code or recovery code (RNF-7).
10. The access journey — sign-in, challenge, landing — passes WCAG 2.2 AA with automated evidence and manual assistive-technology evidence, in EN, ES and FR, on desktop and the M02 mobile design (RNF-5, RNF-6, RNF-10).
11. Every register item listed in §8 is resolved, or explicitly deferred by the product owner, before the feature is declared done.

---

## 8. Register items raised by this specification

Questions this specification could not answer from the design are not recorded here. Each is tracked in the Figma design file, in the architecture register (`122:2` §14) or the D03 Screen decisions ledger (`148:4`), for the architecture or design team to resolve under the identifier below.

| Register | Item | Question | Assumption built until resolved (plan §6) |
|---|---|---|---|
| Architecture | A-27 | Identity status values: which permit sign-in and which produce the generic failure of RF-2.7 | `UNVERIFIED`, `ACTIVE`, `RESTRICTED`, `ERASING`; `ACTIVE` signs in, `UNVERIFIED` takes RF-3, the others the generic failure |
| Architecture | A-29 | Follow-up after a sign-in completed with a recovery code (LC-8) | No follow-up; landing proceeds normally |
| Architecture | A-30 | Security notifications for sign-in events | None sent; the notification port exists for specification 002's use |
| Architecture | A-31 | Re-authentication from the session-expiry dialog and the landing rule of RF-8 | Out of scope; R34 belongs to specification 006 |
| Architecture | A-32 | Owner and scope of the authenticator time tolerance (RF-6.5) | One global configured value |
| Architecture | A-34 | Pending reference carried by the RF-3.1 handoff | Issued only after the password is verified; authorises re-addressing only |
| Architecture | A-35 | Persistence access API and schema-migration tool | Placeholders in the plan and tasks |
| Architecture | A-36 | Identity-scoped membership listing under RLS for the landing rule (RF-8.1) | Read published by the `access` context for the signed-in identity |
| D03 ledger | S-03 | Mobile counterpart of the challenge screen (RF-9.4) | M12 MobileMfaChallengePage (`431:4162`) is the drawn counterpart |
| D03 ledger | S-17 | "Need help?" destination on the sign-in and challenge screens | Rendered; opens N43 HelpShortcutsPanel (`430:4129`) |
| D03 ledger | S-26 | Sign-in or signup screen opened with a valid session (here: sign-in) | The form is shown; success issues a new session |
