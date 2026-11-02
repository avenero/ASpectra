# Specification 003 — TOTP enrollment, recovery codes and factor replacement

**Feature** Second-factor enrollment, recovery codes and authenticator replacement · **Version** 0.2 · **Date** 26 October 2026 · **Status** Draft for review — open doubts closed 26 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (features F-IDN-04 in its enrollment half, F-IDN-05 and F-SET-03 as the subject; F-IDN-03, F-IDN-06, F-IDN-07, F-IDN-08, F-IDN-09, F-IDN-14, F-IDN-16, F-SET-01, F-SET-05 where a criterion touches their boundary; F-X-02, F-X-03, F-X-04, F-X-05, F-X-06, F-X-07; principles P3, P4, P6, P13, P14) → `requirements.md` (FR-IDN-001, 012, 023, 024, 025, 026, 027, 028, 029, 030, 043, 046, 048, 049, 067, 068, 069, 113; FR-SET-001, 020, 021, 022, 043; FR-X-030, 031, 033, 034, 040, 042, 043, 044, 045, 046, 053, 055) → the design of record, screens R39 TOTPEnrollmentWizard `148:3973`, R41 FactorReplacementDialog `148:3995`, R35 RegenerateRecoveryCodesDialog `148:1317`, R36 RecoveryCodesRevealDialog `148:1333` and the "TOTP & recovery" area of R38 SecuritySettingsPage `148:3895`, plus the D02 atlas `159:32045` entry "We couldn't verify the code".

**Related specifications:** 001 Login (the challenge that consumes the factor and the recovery codes this feature issues; its register item A-29 is answered here by decision Q6) · 002 Signup and email verification (a verified email is the prerequisite for enrollment) · 004 Password reset (reset never touches the factor) · 005 Controlled account recovery (the "I lost access" exit of this feature; recovery re-enters the replacement enrollment of RF-9) · 007 Sessions and session management (the sessions revoked by a replacement) · 012 Workspace creation (the first forced entry into enrollment).

This document states **what** enrollment, recovery codes and factor replacement do and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

An ASpectra identity is **global**: one principal with an email, a display name and a status, which may hold memberships in several tenants. Its password and its second factor belong to the identity, never to a tenant: one authenticator and one set of recovery codes serve every workspace the person belongs to, and no tenant holds a copy.

The second factor is a **time-based one-time code (TOTP)** from an authenticator app the person controls. It is **mandatory for every Owner and Admin**, including the solo founding Owner of a new workspace, before any privileged or customer authority is granted; a tenant policy may additionally require it for Members. Required MFA can never be switched off by any tenant action, and no administrator can reset, replace or disable another person's factor — a global identity may be shared with other tenants, so there is no administrator backdoor.

Enrollment follows **three steps — set up, verify code, save recovery codes — and is confirmed before activation**: nothing protects the identity, and nothing is required of it, until the person has proven that their authenticator produces valid codes *and* has acknowledged that the recovery codes are saved. The seed that drives the authenticator is held by the product in a form it can use for verification and is never shown to a client after the setup step; only enrollment metadata is ever displayed.

**Recovery codes** are the way back in when the authenticator is lost. They are high-entropy, single-use, kept only in a non-reversible form, and **revealed exactly once**. Regenerating them requires re-authentication and invalidates the whole previous set.

**Replacing** an authenticator — a new phone, a suspected compromise — **never removes protection first**: the existing factor stays required until the replacement is verified and its recovery codes acknowledged, and the swap is one atomic change followed by a security notification and the revocation of every other session.

### 1.2 Objective

Let a verified identity add, prove and keep a second factor, hold recovery codes for the day the authenticator is lost, and replace that authenticator — while:

- never exposing the seed after setup and never showing a recovery-code set twice;
- never leaving an identity that is required to have a factor without one, not even for an instant during replacement;
- never letting a tenant, an administrator or a password reset weaken or bypass the factor;
- gating Owner, Admin and policy-required Member authority on completed enrollment, and blocking — never bypassing — workspace creation until the founding Owner is enrolled;
- keeping every step honest about what it does and does not do, in the words the design draws.

### 1.3 Why it matters

Owner and Admin authority sits at the top of the two-gate tenancy model: whoever holds it can invite, grant, approve policy and reach every project of the tenant. A password alone is one phishing message away from a takeover. Enrollment that can be skipped, a seed that leaks, a recovery set that can be read twice, or a replacement that disables before it re-enables each turn the second factor into decoration. At the same time, a founder who cannot complete enrollment cannot create a workspace, and a person who loses a phone without a way back loses their company's test estate. The feature has to be strict and survivable at once.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 26 October 2026. The body refers to them as *(Decision: Qn)*. Q1 to Q6 were decided before the first draft; Q7 to Q20 closed the open doubts of the first draft and are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) under the identifier given in each row (architecture `122:2` §14 as A-nn, D03 Screen decisions ledger `148:4` as S-nn).

| Q | Question | Decision |
|---|---|---|
| Q1 | Scope of this specification | The three-step wizard (R39), the regeneration and reveal dialogs (R35, R36), the replacement dialog (R41), the "TOTP & recovery" area of Security Settings (R38) as the home of the entry points and statements, and the rule that Owner / Admin — and policy-required Member — authority is withheld on tenant entry until required enrollment completes. The Sessions tab belongs to 007, password to 004, controlled recovery to 005 (only its entry links are here). |
| Q2 | Who may enroll; whether a factor can be removed | Any verified identity may enroll voluntarily from Security Settings; enrollment is forced when required (workspace creation, Owner / Admin tenant entry, Member policy). Once enrolled, a factor is **never removed, only replaced** — even for an identity that is not required to have one. |
| Q3 | Abandoned or partial enrollment | The factor becomes active, and its recovery codes valid, **only** when the "I have saved" acknowledgement of step 3 is submitted. Leaving at any step — "Leave setup", close, reload, session end — discards the pending seed and codes; a new attempt generates a new seed and a new set; a pending enrollment also expires after a configured time. Nothing is ever shown again. |
| Q4 | Re-authentication for regeneration and replacement | Current password **and** a fresh authenticator code, always; there is no policy switch in this release and a recovery code is not accepted in these dialogs. A lost authenticator goes through the "I lost access · controlled recovery" link (005). |
| Q5 | Effects of a completed replacement | On acknowledgement of the new codes: the old factor and the entire old recovery-code set are invalidated atomically; every **other** session and refresh family of the identity is revoked; the current session continues; a security notification goes to the verified email. A replacement always ends with a fresh recovery-code reveal. |
| Q6 | Acknowledgement at the one-time reveal; running out of codes | Only ticking "I have saved the new codes in a safe place" and activating the finish action completes the reveal; downloading or copying never counts. The download is offered only during the reveal. When an identity reaches zero unused codes, Security Settings shows a prominent notice to regenerate; nothing is forced and sign-in is never blocked. |
| Q7 | Which events send a security notification to the verified email | Enrollment completed, recovery codes regenerated and authenticator replaced all notify; a delivery failure never reverts the change and is recorded. Register A-37. |
| Q8 | When regeneration invalidates the previous recovery-code set | When the new set is acknowledged, never at the step-up; an abandoned reveal leaves the previous set valid. Register A-38. |
| Q9 | What session revocation on replacement covers | Every other identity session and refresh family; the session that performed the replacement continues; run-scoped authority of accepted Runs is untouched. Register A-39. |
| Q10 | How expiries and attempt limits are configured | Two configured families — enrollment (pending-enrollment expiry, verification attempt limit) and step-up (expiry, attempt limit); the authenticator time tolerance is the single global value of A-32. Register A-40. |
| Q11 | Accessible alternative to the setup graphic | The full setup key as copyable text beside the graphic, in step 1 only, never shown again. Register A-41. |
| Q12 | A tenant policy starts requiring MFA for Members while an unenrolled Member is signed in | The gate applies on the next authority derivation: the Member's next action in that tenant is denied and routes to enrollment; no forced sign-out. Register A-42. |
| Q13 | Whether a pending enrollment survives the end of the session that started it | It does not; a new session starts at step 1 with new material. Register A-43. |
| Q14 | Mobile and tablet counterparts of R39, R41, R35, R36 and the R38 area | M13 MobileTOTPEnrollmentWizard (`431:4190`) and M14 MobileRecoveryCodesReveal (`431:4222`) are the drawn counterparts; R41, R35 and the R38 area follow the binding D01 responsive policy (`414:2645`), preserving every statement, field, action and gate. D03 ledger S-28. |
| Q15 | The Authenticator card of R38 when no factor exists | Same card and position, state "No authenticator enrolled", the requirement line and one action "Set up authenticator"; the Recovery codes card shows its statement and no actions. D03 ledger S-29 (D02 `422:3292`, `422:3452`). |
| Q16 | Unused-code count and zero-codes notice on R38 | The count as text in the Recovery codes card; at zero, an inline warning above "Regenerate recovery codes" saying no code remains and a new set should be generated now. D03 ledger S-30 (D02 `422:3292`, `422:3452`). |
| Q17 | Content of the "Download securely issued codes" file | Plain text: a one-line heading naming the product and the issue date, then one code per line; no identity, tenant or seed data; a download is a user-initiated file save, not browser storage (FR-IDN-028). D03 ledger S-31 (D02 `422:3292`, `422:3452`). |
| Q18 | Step-2 error copy on R39 | A wrong code reuses the D02 text "We couldn't verify the code"; attempt limit or expiry shows "Setup must be restarted" and returns to step 1 with new material; both are drawn in D02 `422:3452`. D03 ledger S-32 (D02 `422:3292`, `422:3452`). |
| Q19 | Wizard heading for the Member-policy, voluntary and replacement entries | "Protect your identity." with a one-line subtitle naming the reason; the three steps and every statement stay unchanged. D03 ledger S-33 (D02 `422:3292`, `422:3452`). |
| Q20 | Where "Leave setup" lands when enrollment was forced | The tenant chooser, or the workspace-creation entry when creation was interrupted; no tenant authority on the way. D03 ledger S-34 (D02 `422:3292`, `422:3452`). |

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | founder creating my first workspace | to be taken through authenticator setup before the workspace exists | my Owner identity is protected from the first day and creation is never silently bypassed |
| US-2 | invited Admin who has not enrolled | to be gated at tenant entry with a clear route into enrollment | I understand why I cannot administer yet and how to fix it |
| US-3 | Member of a tenant whose policy requires MFA | the same enrollment route when I enter that tenant | I keep access to a workspace that tightened its policy |
| US-4 | any verified user, not required to have a factor | to enroll an authenticator voluntarily from my security settings | my identity is protected before I ever become an Owner |
| US-5 | person enrolling | to verify that my authenticator produces an accepted code before anything is activated | a mistyped setup never locks me out |
| US-6 | person enrolling | to see my recovery codes once, download them, and confirm that I saved them | I have a way back in if I lose my phone |
| US-7 | user who lost some recovery codes | to regenerate a fresh set after proving my password and my authenticator | the old set is worthless to anyone who found it |
| US-8 | user with a new phone | to replace my authenticator without ever being unprotected in between | a replacement cannot be used as a way to disable MFA |
| US-9 | user who used my last recovery code | to be told plainly that I have none left and where to regenerate | I am not surprised the next time I lose my phone |
| US-10 | user in my security settings | to see when MFA is required of me, that it cannot be disabled, and that replacement never removes first | I know what the product promises |
| US-11 | user on a phone or tablet | the wizard and dialogs reflowed for my viewport with every statement intact | I can enroll anywhere without losing information |
| US-12 | attacker or over-reaching administrator (negative story) | — | I cannot obtain the seed, replay a code, reuse a recovery code, read a code set twice, activate a factor I never verified, or disable or replace someone else's factor through any tenant or administrator action |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *factor* means the identity's active TOTP enrollment, *pending enrollment* means a seed and code set generated but not yet acknowledged, *step-up* means the re-authentication of RF-8.1 / RF-9.1, and *the reveal* means the one-time display of RF-4.

### RF-1 · Entry points and the enrollment gate

Enrollment is reachable voluntarily from Security Settings and is forced wherever the product requires a factor before granting authority. *Why:* TOTP is mandatory for Owner and Admin before any privileged authority, and the design is explicit that access "stays gated until required enrollment is complete". *(Decision: Q1, Q2.)*

- RF-1.1 The system shall offer "Set up authenticator" in the TOTP & recovery area of Security Settings to every verified identity that has no factor. *(FR-SET-022; decision Q2, Q15)*
- RF-1.2 When a verified identity with no factor starts workspace creation, the system shall route it into enrollment before any workspace field is accepted, and shall block creation — never bypass it — until enrollment completes. *(FR-IDN-113, FR-IDN-026; F-IDN-14)*
- RF-1.3 When an identity with no factor enters a tenant in which it holds an Owner or Admin membership, the system shall withhold every privileged authority of that membership and present the enrollment wizard as the only way forward in that tenant. *(FR-IDN-023, FR-IDN-026, FR-IDN-030; R39)*
- RF-1.4 When an identity with no factor enters a tenant whose policy requires MFA for Members and it holds a Member membership there, the system shall withhold that membership's authority in the same way as RF-1.3. *(FR-IDN-023, FR-SET-043; decision Q12)*
- RF-1.5 While enrollment is required and incomplete, the system shall derive no privileged or customer authority for the identity in that tenant; a first factor, a completed challenge, a session or a membership record shall never substitute for the completed enrollment. *(FR-IDN-024, FR-IDN-026; P3)*
- RF-1.6 The system shall never ask an identity that already holds an active factor to enroll again; a tenant that requires MFA is satisfied by the identity's single factor, whichever tenant prompted it. *(FR-IDN-001; P4)*
- RF-1.7 If the identity's email is not verified, then the system shall not start an enrollment and shall route to the email-verification pending state (specification 002). *(FR-IDN-012)*
- RF-1.8 If the identity's status permits no sign-in (restricted or under erasure), then the system shall not start an enrollment. *(A-27)*
- RF-1.9 The wizard shall carry the heading "Protect your Owner identity." with "Set up your authenticator app before protected workspace administration." when forced by an Owner or Admin requirement, and "Protect your identity." with a one-line subtitle naming the actual reason — Member policy, voluntary setup or replacement — otherwise; the three-step indicator and every statement shall stay the same. *(R39; decision Q19)*

### RF-2 · Step 1 — Set up

The person adds the identity to an authenticator app from material the product generates for this enrollment only. *Why:* the seed is created and kept on the product's side and shown to the person exactly once, in the form their authenticator needs.

- RF-2.1 When step 1 opens, the system shall create a new pending enrollment for the identity — a fresh seed and a fresh recovery-code set — superseding any earlier pending enrollment of the same identity. *(FR-IDN-026, FR-IDN-027; decision Q3)*
- RF-2.2 The system shall present the setup material as a scannable graphic and, as an accessible alternative, the full setup key as copyable text the person can enter by hand; both shall be visible in step 1 only. *(FR-IDN-027, FR-X-040; R39 "Setup graphic supplied securely"; decision Q11)*
- RF-2.3 The system shall present the three-step indicator "1 · Set up · 2 · Verify code · 3 · Save recovery codes" with the current step identified by text, not colour alone. *(FR-IDN-026, FR-X-042; R39)*
- RF-2.4 The system shall state on the wizard that verification must succeed before recovery-code acknowledgement, that after verification securely issued one-time codes are shown once and must be saved and acknowledged before enrollment completes, and that Owner / Admin access stays gated until required enrollment is complete. *(FR-IDN-026, FR-IDN-028; R39)*
- RF-2.5 The system shall offer "Leave setup" at every step. When it is activated, the system shall discard the pending enrollment (RF-6.3) and return the person, with no factor and no change, to the tenant chooser when enrollment was forced at tenant entry, to the workspace-creation entry when creation was interrupted, and to Security Settings when enrollment was voluntary; no tenant authority shall be granted on the way. *(R39; decision Q3, Q20)*
- RF-2.6 After step 1 has been left, the system shall never show the same setup material again; a new attempt begins with new material. *(FR-IDN-027; decision Q3)*

### RF-3 · Step 2 — Verify code

The person proves that the authenticator was set up correctly by submitting a current code. *Why:* activating a factor the person cannot actually produce codes for would lock them out of their own identity.

- RF-3.1 The system shall present a code field labelled "Code from your authenticator", accept a complete code pasted in one action as well as typed, ignore separators and surrounding whitespace, and offer "Verify code & continue". *(FR-IDN-025, FR-IDN-026; R39)*
- RF-3.2 The system shall bind verification to the pending enrollment of this identity, to the purpose "enrollment", to the configured pending-enrollment expiry and to the configured verification attempt limit. *(FR-IDN-025, FR-X-046; P13; decision Q10)*
- RF-3.3 When a code is submitted, the system shall show a verification-in-progress state and disable resubmission until the answer arrives. *(R39)*
- RF-3.4 When the submitted code is valid for the pending seed within the configured time tolerance, the system shall mark the pending enrollment as verified and advance to step 3. Verification shall **not** activate the factor. *(FR-IDN-026; decision Q3)*
- RF-3.5 The system shall reject a code whose time-step is at or before the last accepted time-step of this pending seed, so that an observed code cannot be replayed within the wizard. *(FR-IDN-025; specification 001 RF-6.5; A-32)*
- RF-3.6 If the code is wrong, then the system shall show a generic "We couldn't verify the code" message that does not state whether the code was wrong, expired or already used, shall not disclose the remaining attempt count, and shall keep the person on step 2. *(FR-IDN-025, FR-X-042; D02 `159:32045`; decision Q18)*
- RF-3.7 If the attempt limit is reached or the pending enrollment expires, then the system shall discard the pending enrollment, tell the person "Setup must be restarted", and return them to step 1 with new material. *(FR-IDN-025; decision Q3, Q18)*
- RF-3.8 The system shall never display the seed, the setup material or any recovery code on step 2. *(FR-IDN-027)*

### RF-4 · Step 3 — Save recovery codes (the one-time reveal)

The person is shown their recovery codes once, may download them, and must acknowledge that they are saved. *Why:* recovery codes are the only way back without the authenticator; shown twice they are a liability, shown without acknowledgement they are forgotten. *(Decision: Q3, Q6.)* This section also governs the reveal reached from regeneration (RF-8) and from replacement (RF-9); the design draws it as R36.

- RF-4.1 When step 3 opens, the system shall display every code of the pending set, in full, exactly once, with the statements that each code can be used once and that the codes are shown this one time. *(FR-IDN-028; R36)*
- RF-4.2 The number of codes in a set and their entropy shall be validated configuration; the system shall refuse to start without them and shall never substitute a default. *(FR-IDN-028, FR-X-046; P13)*
- RF-4.3 The system shall offer "Download securely issued codes" during the reveal only; the downloaded content shall be plain text with a one-line heading naming the product and the issue date, then one code per line, and shall contain no identity, tenant or seed data. *(Design rule R36 `148:1333`; decision Q17)*
- RF-4.4 The system shall present the acknowledgement "I have saved the new codes in a safe place" unticked, and shall keep the finish action unavailable until it is ticked. *(FR-IDN-028; R36)*
- RF-4.5 When the acknowledgement is ticked and the finish action is activated, the system shall complete the enrollment as in RF-5 (or the regeneration of RF-8.6, or the replacement of RF-9.6). Neither a download nor a copy shall count as acknowledgement. *(FR-IDN-028; decision Q6)*
- RF-4.6 If the reveal is closed, left, reloaded or its session ends before the finish action, then the system shall discard the pending set together with its pending enrollment, and shall never show that set again. *(FR-IDN-028; decision Q3)*
- RF-4.7 The system shall never write a recovery code, or the set, to browser storage of any kind; the codes exist on the client only while the reveal is open. *(FR-IDN-028, FR-X-034)*
- RF-4.8 The system shall retain each code only in a non-reversible form once the set is acknowledged, and shall retain nothing of a discarded set. *(FR-IDN-028, FR-X-030)*

### RF-5 · Activation and its effects

The factor and its codes come into force together, in one step, when the person acknowledges the codes. *Why:* "confirmed before activation" — the product must never hold a factor without acknowledged recovery codes, nor codes without a factor. *(Decision: Q3.)*

- RF-5.1 When the acknowledgement of RF-4.5 is submitted for an initial enrollment, the system shall, as one indivisible change: activate the verified seed as the identity's factor, make the acknowledged set the identity's valid recovery codes, and record the enrollment time. *(FR-IDN-026, FR-IDN-028)*
- RF-5.2 From activation on, every sign-in of the identity shall be challenged for the factor (specification 001, decision Q2), in every tenant. *(FR-IDN-025)*
- RF-5.3 When activation completes inside a forced entry (RF-1.2 … RF-1.4), the system shall return the person to the interrupted destination — workspace creation, or the tenant it was entering — and the withheld authority shall be derivable from the next request on. *(FR-IDN-026, FR-IDN-113)*
- RF-5.4 When activation completes from Security Settings, the system shall return to the TOTP & recovery area showing the enrolled state. *(R38)*
- RF-5.5 The system shall send a security notification to the identity's verified email when enrollment completes. *(FR-X-045; decision Q7)*
- RF-5.6 Activation shall never create, extend or upgrade a session, accept a mapping, approve a bundle, launch a Run or replay any pending intent. *(FR-IDN-024, FR-IDN-067; P1)*

### RF-6 · Lifetime of a pending enrollment

A pending enrollment is short-lived, singular and disposable. *Why:* an unverified seed lying around is attack surface; a verified-but-unacknowledged one would be a factor nobody can recover.

- RF-6.1 The system shall hold at most one pending enrollment per identity; starting another supersedes and discards the previous one. *(decision Q3)*
- RF-6.2 The system shall give every pending enrollment a configured expiry; when it elapses the pending enrollment is discarded whatever step it reached. *(FR-X-046; P13; decision Q10)*
- RF-6.3 When a pending enrollment is discarded for any reason — leave, close, reload, expiry, attempt limit, the end of the session that started it, supersession — the system shall destroy its seed and set, retain nothing reusable, and leave the identity's active factor and codes, if any, exactly as they were. *(decision Q3, Q13)*
- RF-6.4 A pending enrollment shall grant no authority and shall not count as a factor for the challenge of specification 001 or for the gate of RF-1. *(FR-IDN-024, FR-IDN-026)*

### RF-7 · Security Settings — the TOTP & recovery area

The area is the identity's view of its own protection and the home of every action in this feature. *Why:* the design states the promises of the feature on this page, and requires that only metadata is ever shown. *(Decision: Q1.)*

- RF-7.1 The system shall present, under the Security Settings heading "Your identity, protected." and the subtitle "Global sign-in security. Workspace membership policies may require stronger verification.", a "TOTP & recovery" area beside the Sessions, Password and Privacy areas. *(FR-SET-020; R38)*
- RF-7.2 The area shall state whether and why MFA is required of the viewer — by an Owner or Admin membership, by a tenant policy for Members, or not required — in the terms drawn ("TOTP required for your Owner membership"), and shall state that required MFA cannot be disabled and that replacing a factor requires the current factor or a controlled recovery process, never removal first. *(FR-SET-021, FR-IDN-030; R38)*
- RF-7.3 While a factor is active, the "Authenticator app" card shall show the enrolled state, the identity (display name and verified local email), the requirement ("Owner / Admin mandatory; workspace may tighten"), the enrollment time, and the statement "Enrollment metadata only; seed never exposed"; it shall offer "Replace authenticator". *(FR-SET-022, FR-IDN-027; R38)*
- RF-7.4 While no factor is active, the "Authenticator app" card shall keep its position and layout, show the state "No authenticator enrolled" and the requirement line, and offer the single action "Set up authenticator" (RF-1.1); the "Recovery codes" card shall show its statement and no actions. *(decision Q2, Q15)*
- RF-7.5 While a factor is active, the "Recovery codes" card shall state that one-time codes must be stored safely and that regeneration requires policy-approved re-authentication and invalidates the old set, and shall offer "Regenerate recovery codes" and the "Controlled account recovery" route. *(FR-SET-022, FR-IDN-028; R38)*
- RF-7.6 The "Recovery codes" card shall show the number of unused codes remaining as text; when it is zero, the card shall show an inline warning above "Regenerate recovery codes" that no recovery code remains and that a new set should be generated now. The warning shall never block sign-in or any other action. *(decision Q6, Q16)*
- RF-7.7 The system shall never render the seed, any recovery code, the setup material or any count of consumed codes by value on this surface. *(FR-IDN-027, FR-SET-022)*
- RF-7.8 Every statement and action of this area shall be available to the viewer for their own identity only; no view of another identity's factor or codes shall exist anywhere in the product. *(FR-IDN-049)*

### RF-8 · Regenerate recovery codes

A fresh set replaces the old one after a step-up, and only once the new set is acknowledged. *Why:* a found or half-used set must be made worthless, and the person must never be left with no valid set. *(Decision: Q4.)*

- RF-8.1 When "Regenerate recovery codes" is activated, the system shall present the dialog "Replace your recovery codes?" with the statement that re-authentication uses the current password and factor, the warning "Existing codes will be invalidated" with its explanation, a current-password field with "Show", a current-authenticator-code field, the actions "Keep existing codes" and "Verify & regenerate codes", and the statement "This does not disable TOTP or bypass required Owner / Admin MFA." *(FR-IDN-028; R35)*
- RF-8.2 The step-up shall require both the current password and a fresh authenticator code of the active factor; a recovery code shall not be accepted in place of the authenticator code. *(FR-IDN-028; decision Q4)*
- RF-8.3 The step-up shall be bound to the identity, the purpose "regenerate", the configured step-up expiry and the configured step-up attempt limit, with the replay rule of RF-3.5 applied to the active factor; a failed step-up shall show one generic failure for a wrong password, a wrong code or both, and shall count toward abuse protection. *(FR-IDN-025, FR-IDN-021; decision Q10)*
- RF-8.4 If the step-up attempt limit is reached or its expiry elapses, then the system shall close the dialog with nothing changed. *(P6)*
- RF-8.5 When the step-up succeeds, the system shall generate a new pending set and open the reveal (RF-4) with the heading "Save your replacement codes." and the statement that previous codes are no longer valid once this reveal is finished. *(FR-IDN-028; R36)*
- RF-8.6 When the acknowledgement of RF-4.5 is submitted, the system shall, as one indivisible change, invalidate every code of the previous set and make the new set the identity's valid codes. *(FR-IDN-028; decision Q8)*
- RF-8.7 If the reveal is abandoned before acknowledgement (RF-4.6), then the previous set shall remain valid and unchanged. *(decision Q3, Q8)*
- RF-8.8 "Keep existing codes" and any close of the dialog before the reveal shall change nothing. *(R35)*
- RF-8.9 Regeneration shall never touch the factor, never revoke a session and never alter any requirement; the system shall send a security notification to the verified email when the new set is acknowledged. *(FR-IDN-030; decision Q7)*

### RF-9 · Replace authenticator

A new authenticator is enrolled while the old one stays required, and the swap is atomic. *Why:* "Replace, don't remove protection." — there must be no silent disabling step. *(Decision: Q4, Q5.)*

- RF-9.1 When "Replace authenticator" is activated, the system shall present the dialog "Replace, don't remove protection." with the subtitle "Verify the current factor before enrolling its replacement.", the warning "Required MFA remains active" explaining that the existing authenticator stays required until the replacement is verified and recovery acknowledgement is complete and that there is no silent disabling step, a current-password field with "Show", a current-authenticator-code field, the actions "Keep current factor" and "Verify & begin replacement", the route "I lost access · controlled recovery", and the statement "Lost-factor recovery is separately verified. Email password reset alone never replaces or resets MFA." *(FR-IDN-029, FR-IDN-043; R41)*
- RF-9.2 The step-up shall follow RF-8.2 … RF-8.4 with the purpose "replace". *(FR-IDN-029; decision Q4)*
- RF-9.3 When the step-up succeeds, the system shall start a replacement enrollment: steps 1 to 3 of RF-2 … RF-4 with a new seed and a new pending set, under the heading of the replacement purpose. *(FR-IDN-029; decision Q5)*
- RF-9.4 While the replacement enrollment is pending, the existing factor and the existing recovery codes shall remain active and required everywhere; a sign-in during this time is challenged against the existing factor only. *(FR-IDN-029)*
- RF-9.5 If the replacement enrollment is abandoned, expires or exhausts its attempts at any step, then the system shall discard it and the existing factor and codes shall remain exactly as they were; no session shall be revoked and no notification of a completed replacement shall be sent. *(FR-IDN-029; decision Q3)*
- RF-9.6 When the acknowledgement of RF-4.5 is submitted for a replacement, the system shall, as one indivisible change: activate the new seed as the identity's factor, invalidate the previous seed, invalidate every code of the previous set, and make the new set the identity's valid codes. *(FR-IDN-029, FR-IDN-028; decision Q5)*
- RF-9.7 When RF-9.6 completes, the system shall revoke every other session and refresh family of the identity and void every other open login transaction of the identity (the session / security version advances, specification 006 RF-2.5), leave the session that performed the replacement active, and send a security notification to the verified email stating that the authenticator was replaced. *(FR-IDN-029, FR-IDN-068; decision Q5, Q9)*
- RF-9.8 Revocation under RF-9.7 shall never cancel an accepted Run and shall never replay any Run intent. *(FR-IDN-067)*
- RF-9.9 When "I lost access · controlled recovery" is activated, the system shall leave the dialog with nothing changed and route to the controlled-recovery entry (specification 005); controlled recovery shall never bypass suspension, revoked membership or required MFA, and when it succeeds it shall enter the replacement enrollment of RF-9.3 directly, the step-up of RF-9.1 having been replaced by its own verification. *(FR-IDN-046 path (a), FR-IDN-048; R41)*
- RF-9.10 "Keep current factor" and any close of the dialog before the replacement enrollment shall change nothing. *(R41)*

### RF-10 · What nobody else can do

The factor is the identity's alone. *Why:* a global identity may belong to other tenants; a weakening by one tenant or administrator would expose all of them.

- RF-10.1 The system shall provide no tenant setting, tenant action or administrator action that disables, resets, replaces or regenerates another identity's factor or recovery codes, or that disables required MFA for anyone. *(FR-IDN-030, FR-IDN-049)*
- RF-10.2 A tenant policy shall only be able to extend the MFA requirement to Members; it shall never relax the Owner / Admin requirement. *(FR-IDN-030, FR-SET-043)*
- RF-10.3 A password reset or forced password change shall never enroll, reset, replace or bypass the factor or touch the recovery codes. *(FR-IDN-043)*
- RF-10.4 No session, device or browser shall ever be remembered as trusted or grant a factor bypass; the Sessions area states this (specification 007). *(FR-IDN-069)*
- RF-10.5 Suspension of a tenant, revocation of a membership or erasure of an identity shall never be lifted or bypassed by any step of this feature. *(FR-IDN-048)*

### RF-11 · Audit, secrecy and notification

Every consequential event is recorded, and no secret ever leaves the place it belongs. *Why:* FR-X-030 is absolute for seeds and recovery codes.

- RF-11.1 The system shall append to the audit log: enrollment started, verified, completed, abandoned and expired; reveal opened and acknowledged; regeneration step-up succeeded or failed and regeneration completed; replacement step-up succeeded or failed, replacement completed and the sessions it revoked; every gate applied under RF-1. Each entry identifies the identity, the time and the calling context. *(FR-X-044)*
- RF-11.2 No audit entry, log line, error, problem detail, URL, export, notification or analytics event shall ever contain the seed, the setup material, an authenticator code, a recovery code, the number of codes by value, or the password. *(FR-X-030, FR-X-033)*
- RF-11.3 The seed shall be held so that the product can verify codes against it and never in a one-way form; it shall be protected under a server-held key reference, shall never be sent to any client after step 1 of its enrollment, and shall never be exported. *(FR-IDN-027, FR-X-031)*
- RF-11.4 Every security notification of this feature shall go only to the identity's verified email through the transactional email channel, shall describe the event and when it happened, and shall contain no code, seed or link that grants access. *(FR-X-045)*
- RF-11.5 Every write of this feature shall confirm persistence before reporting success, and shall be idempotent so that a lost acknowledgement leads to a lookup, never to a second activation, second set or second revocation. *(FR-SET-001, FR-X-020, FR-X-022)*

### RF-12 · Responsive and accessible surfaces

Every screen of this feature is a critical journey. *Why:* enrollment stands between a founder and their workspace; it must work for everyone, on every viewport.

- RF-12.1 Where no designed mobile or tablet counterpart exists for R39, R41, R35, R36 or the R38 area, the system shall present them reflowed under the desktop rules, preserving every statement, field, action, route and gate. *(FR-X-055; decision Q14)*
- RF-12.2 Every interactive element shall have a touch target of at least 44 px, a visible label and visible focus. *(FR-X-053)*
- RF-12.3 The step indicator, every error and every status shall be conveyed by text and programmatically associated with its field or region; live announcements shall never steal focus from the code field or the acknowledgement. *(FR-X-040, FR-X-042)*
- RF-12.4 The setup material of RF-2.2 shall be usable without sight of the graphic, through its text alternative. *(FR-X-040; decision Q11)*
- RF-12.5 All screens and messages shall exist in EN / ES / FR; the interface language shall never change a code, a seed or a requirement. *(FR-X-043)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Secrecy.** The seed, the setup material, authenticator codes, recovery codes and passwords never appear in logs, audit, errors, URLs, exports, notifications, analytics or browser storage; the seed is never sent to a client after step 1. | FR-X-030, FR-X-034, FR-IDN-027, FR-IDN-028 |
| RNF-2 | **One-time reveal.** A recovery-code set is displayable exactly once; no path — reload, back, second tab, support request, administrator — shows it again. | FR-IDN-028; decision Q3 |
| RNF-3 | **Atomicity.** Activation (RF-5.1), regeneration (RF-8.6) and replacement (RF-9.6) each happen as one indivisible change; no observer can see a factor without valid codes, two active factors, or two valid sets. | FR-IDN-026, FR-IDN-029 |
| RNF-4 | **Never unprotected.** An identity with an active factor has one at every instant, including throughout a replacement; a required identity is never granted authority without one. | FR-IDN-029, FR-IDN-030; decision Q5 |
| RNF-5 | **Replay resistance.** A code accepted once for a seed is never accepted again; a consumed recovery code is never accepted again; a step-up is bound to its purpose and cannot be reused for another action. | FR-IDN-025, FR-IDN-028 |
| RNF-6 | **Configured limits.** Code count, code entropy, pending-enrollment expiry, verification and step-up attempt limits and expiries, and the time tolerance are validated configuration with no default; the product refuses to start without them. | FR-X-046; P13 |
| RNF-7 | **Fail closed.** Any absent, invalid, expired, superseded or unverified pending enrollment, step-up or set blocks the dependent change; nothing is assumed and no fallback is attempted. | P6 |
| RNF-8 | **No override.** No tenant policy, tenant action, administrator action, password reset or session grants a bypass, removal or reset of another identity's — or one's own required — factor. | FR-IDN-030, FR-IDN-043, FR-IDN-049, FR-IDN-069 |
| RNF-9 | **Delivery prerequisite.** Security notifications depend on the transactional email channel; their failure never blocks or reverts the change they announce, and is recorded. | FR-X-045 |
| RNF-10 | **Accessibility.** Wizard, dialogs, reveal and the settings area meet WCAG 2.2 AA with automated and manual assistive evidence. | FR-X-040, FR-X-042 |
| RNF-11 | **Language.** EN / ES / FR, display only. | FR-X-043 |
| RNF-12 | **Reflow.** Mobile and tablet reflows preserve every statement, field, action and gate. | FR-X-055 |
| RNF-13 | **No authority side effect.** Nothing in this feature creates a session, enters a tenant, accepts, approves, launches or replays anything. | FR-IDN-024, FR-IDN-067; P1 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | "Leave setup" at step 1 | Pending enrollment discarded; no factor; setup material never shown again (RF-2.5, RF-2.6). |
| LC-2 | Leave, close or reload at step 2 | Pending enrollment discarded; restart from step 1 with new material (RF-6.3). |
| LC-3 | Correct code at step 2, then close before acknowledging at step 3 | Nothing activated; pending enrollment and set discarded; next attempt starts from step 1 (RF-3.4, RF-4.6; decision Q3). |
| LC-4 | Reload while the reveal is open | Set discarded and never re-shown; enrollment restarts (RF-4.6). |
| LC-5 | Pending enrollment expires while the person is on step 2 | Next submission is refused; restart from step 1 (RF-3.7, RF-6.2). |
| LC-6 | Wrong code on the last permitted attempt at step 2 | Pending enrollment discarded; restart from step 1 (RF-3.7). |
| LC-7 | Same code submitted twice at step 2 within the tolerance | Second submission treated as wrong (RF-3.5). |
| LC-8 | Two tabs open enrollment | The later start supersedes; the earlier tab's code is refused as belonging to a discarded enrollment (RF-6.1). |
| LC-9 | Download activated, dialog closed without ticking the acknowledgement | Nothing activated; the downloaded file holds a set that is never valid (RF-4.5, RF-4.6; decision Q6). |
| LC-10 | Acknowledgement ticked, finish activated, no download | Activation completes; downloading is optional (RF-4.5). |
| LC-11 | Finish activated twice (double submit or lost acknowledgement) | Exactly one activation; the second request returns the same outcome (RF-11.5). |
| LC-12 | Last unused recovery code consumed at sign-in (specification 001 LC-8) | Sign-in completes; Security Settings shows the zero-codes notice; nothing is forced (RF-7.6; decision Q6). |
| LC-13 | Regeneration step-up succeeds, reveal abandoned | Previous set stays valid and unchanged (RF-8.7). |
| LC-14 | Regeneration step-up with a recovery code in the code field | Refused as a wrong code (RF-8.2). |
| LC-15 | Regeneration with correct password and wrong code, or wrong password and correct code | Same generic failure; counts toward abuse protection (RF-8.3). |
| LC-16 | Replacement enrollment abandoned after its step 2 | Old factor and codes untouched; no revocation, no notification (RF-9.5). |
| LC-17 | Sign-in from another device while a replacement is pending | Challenged against the old factor; the pending replacement is invisible to it (RF-9.4). |
| LC-18 | Replacement completes while another session is mid-action | That session is revoked; its accepted Runs continue; any unsaved work is subject to specification 006's expiry handling and specification 007's end-of-session rules (RF-9.7, RF-9.8). |
| LC-19 | Replacement completes; the person signs in again on the same browser | Challenge uses the new factor; a code from the old authenticator is refused (RF-9.6). |
| LC-20 | Tenant policy starts requiring MFA for Members while an unenrolled Member is signed in | Authority withheld from the next derivation on; the Member is routed to enrollment on its next tenant interaction (RF-1.4; decision Q12). |
| LC-21 | Identity enrolled through tenant A enters tenant B as Admin | Not asked to enroll again; the single factor satisfies B (RF-1.6). |
| LC-22 | Founding Owner closes the wizard during workspace creation | No workspace created; next attempt starts at enrollment again (RF-1.2; decision Q3). |
| LC-23 | Invited Admin accepts an invitation without a factor | Membership exists; entering the tenant withholds Admin authority until enrollment (RF-1.3; specification 008). |
| LC-24 | Unverified identity reaches an enrollment entry point | Routed to verification pending; no pending enrollment created (RF-1.7). |
| LC-25 | Restricted or erasing identity | No enrollment started (RF-1.8). |
| LC-26 | Password reset completed by an enrolled identity | Factor and codes untouched; the next sign-in is still challenged (RF-10.3). |
| LC-27 | "I lost access · controlled recovery" activated from R41 | Dialog closes with nothing changed; the person is in specification 005 (RF-9.9). |
| LC-28 | Controlled recovery succeeds | The person enters the replacement enrollment directly; its completion applies RF-9.6 … RF-9.8 (RF-9.9). |
| LC-29 | Notification email cannot be delivered | The activation, regeneration or replacement stands; the failure is recorded (RNF-9). |
| LC-30 | Security Settings opened by an identity with no factor | Not-enrolled card with "Set up authenticator"; the recovery-codes actions are not offered (RF-7.4). |

---

## 6. Out of scope

- **The sign-in challenge** that consumes the factor and the recovery codes — specification 001.
- **Controlled account recovery** beyond its entry links: paths (a), (b) and (c), case verification and case review — F-IDN-07, specification 005.
- **The Sessions area**, listing and individual revocation, the session-expiry dialog and re-authentication — F-IDN-08 and F-IDN-09, specifications 006 and 007.
- **Password reset and forced password change** — F-IDN-06, specification 004.
- **Editing the tenant policy** that requires MFA for Members — F-SET-05.
- **Workspace creation** itself; this feature only supplies its enrollment prerequisite — F-IDN-14.
- **Removal of a factor** in any circumstance *(decision Q2)*.
- **More than one active factor** per identity, or any factor other than TOTP: WebAuthn security keys, passkeys, SMS or email codes — FR-EXT-030, product boundary.
- **Trusted devices, remembered devices or any challenge bypass** — product boundary.
- **Any administrator or support path** that views, resets, replaces or disables another identity's factor or codes — product boundary.
- **A policy switch** making the password optional in the step-up *(decision Q4)*.
- **Forcing regeneration** when the set runs out *(decision Q6)*.

---

## 7. Criteria of completion

Enrollment, recovery codes and factor replacement are complete when all of the following hold:

1. Every user story US-1 … US-11 can be walked end to end on the desktop design and in the reflows with no step outside this specification, and US-12 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-12 has at least one verification that fails before and passes after the feature exists.
3. Gate tests show that an unenrolled Owner, Admin and policy-required Member obtains no privileged authority in the tenant, that workspace creation is blocked until enrollment completes, and that completion lifts the gate without a new sign-in (RF-1, RF-5.3).
4. Enrollment tests show: no activation at step 2; activation only on acknowledgement; discard on leave, close, reload, expiry, attempt limit and supersession; new material on every restart; the same code replayed is refused (RF-2 … RF-6, LC-1 … LC-8).
5. Reveal tests show that a set is displayed exactly once, that download and copy never acknowledge, that an abandoned reveal leaves the previous set (if any) valid, and that no code reaches browser storage (RF-4, RF-8.7, RNF-2).
6. Regeneration and replacement tests show the step-up requires password and authenticator code, refuses a recovery code, binds to its purpose and limits; that the old set and old seed are wholly invalid after completion and wholly intact after abandonment; and that activation, regeneration and replacement are atomic under concurrent requests and lost acknowledgements (RF-8, RF-9, RNF-3, RNF-5, LC-11, LC-13, LC-16).
7. Replacement revokes every other session and refresh family, keeps the current session, sends the notification, and leaves an accepted Run untouched (RF-9.7, RF-9.8, LC-18).
8. No tenant policy, administrator action or password reset can disable, reset, replace or regenerate a factor or set; the only paths are those in RF-8 and RF-9 and controlled recovery (RF-10, RNF-8).
9. The seed is verifiably never returned to a client after step 1, never logged, never exported, and held in a form that verifies codes (RF-11.3, RNF-1).
10. The product refuses to start when code count, entropy, any expiry, any attempt limit or the time tolerance is absent or unbounded (RNF-6).
11. Audit entries exist for every event in RF-11.1 and contain no seed, code, recovery code or password (RF-11.2).
12. Wizard, dialogs, reveal and the settings area pass WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR, at desktop, tablet and mobile widths; the setup material is usable through its text alternative (RNF-10 … RNF-12, RF-12.4).
13. The two register items this specification relies on from earlier specifications — A-27 (identity status values) and A-32 (authenticator time tolerance) — are resolved, or explicitly deferred by the product owner, before the feature is declared done.

