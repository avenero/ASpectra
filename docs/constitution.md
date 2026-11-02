# ASpectra — Constitution

**Version** 1.8 · **Date** 2 November 2026 · **Status** Implementation baseline · **Amendments** v1.1 clarifies P4 and adds P16.5 · v1.2 extends F-IDN-07 from path (a) to the three paths of FR-IDN-046 and binds FR-IDN-047 (specification 005, decision Q1, 27 October 2026) · v1.3 extends F-IDN-09 to the whole of FR-IDN-068, adding "Sign out everywhere" (specification 007, decision Q1, 27 October 2026) · v1.4 extends F-IDN-12 to invitations, tenant role edits and membership management (FR-IDN-097 fully bound) and F-SET-04 to the Pending invitations view (FR-SET-030 fully bound) (specification 010, decisions Q1 and Q8, 27 October 2026) · v1.5 (editorial, 27 October 2026) removes the three "(part)" markings whose scope clause restated the whole requirement (FR-X-023, FR-X-032, FR-X-055) and adds the specification 010 design source to F-SET-04; no feature changes · v1.6 extends F-IDN-17's part of FR-IDN-122 from the erasure-request gates to the independent verification, the recording of the request with the transition of the identity to under erasure, and the revocation of its sessions; carrying out the erasure stays outside the baseline (specification 015, decisions Q1 and Q2, 28 October 2026) · v1.7 (2 November 2026) re-points every design citation after the architecture consolidation (the seven `GAP-*` sections → canonical boards 1–7; GAP-OPERATOR `227:2` → register `122:2` §15), records the retirement of the screen register `268:2` into the D03 Screen decisions ledger `148:4`, extends F-X-07 to the drawn mobile counterparts M10–M31 under the binding D01 responsive policy, binds FR-IDN-144 fully in F-IDN-19 (Privacy, Terms and Contact open operator-published documents outside the product; specification 017, decision Q3) and adds the customer-vocabulary rule to F-X-04 (closure round of 2 November 2026) · v1.8 (2 November 2026, QA round) binds the first-release fixture and Android / iOS scope decided in register `122:2` §13.7–§13.8 (F-CAT-02 gains FR-CAT-012/013, F-CAT-05 gains Fixture revisions with FR-CAT-052 fully and FR-CAT-055, new F-CAT-06 Mobile app revisions binds FR-CAT-070–074, FR-CAT-076 and FR-PRJ-073, F-RUN-07 adds Mobile with FR-CAT-075), binds FR-IDN-116 fully in F-IDN-16 and FR-X-052 fully in F-X-07, aligns F-IDN-04 (setup key shown once in enrollment step 1), F-IDN-06 and F-IDN-09 (forced change as a sign-in gate, specification 004 Q2 / A-45), F-IDN-10 (masked invited address, MFA gated at tenant entry, specification 008 Q3 / Q6), F-IDN-17 (credentials), F-OPS-01 (six operator records), F-PRJ-07 (Purge outside), P10 (Core modules and runner host) and P16.5 (product-owner decisions in §1.4), adds III.4 (requirements outside this baseline), completes the vocabulary (WORKSPACE selector, US spelling) and names in P10 the QR encoder of the TOTP setup graphic (register A-46, decided 2 November 2026)

This document is the implementation baseline of ASpectra: every feature in it is completely defined by the design — behaviour, rules, contracts and, where it has a user interface, its screens — and can be built as written. It governs how the product is built (Part I), defines what is built (Part II), and binds every feature to the functional requirements it fulfils (Part III).

| Document | Owns |
|---|---|
| `constitution.md` | The governing principles and the set of features to build, each traced to its requirements |
| `requirements.md` | The text of every functional requirement, by identifier |
| `scope.md` | The product scope, the authorities and the product boundaries |
| `specs/` | One folder per feature specification: the specification (WHAT), its technical plan (HOW) and its task list; see `specs/README.md` |
| Figma `DpfPk8xDraLcDzBhMYwDTo` | The design of record: architecture boards and the product experience page |

**How to read a feature.** Each feature has an identifier `F-<AREA>-nn`, a statement of what it does, its acceptance rules, and the requirement identifiers it implements. A requirement marked **(part)** is implemented by that feature only in the clause stated under *Scope of partial requirements*; the rest of that requirement is not part of this baseline. A *design rule* is a behaviour drawn in the design that the feature must also satisfy; it cites the Figma node that defines it. Node identifiers refer to the Figma file above; screen codes (R01, N12, M07 …) refer to the product experience page `148:2`.

---

# Part I · Principles

These principles bind every feature in Part II. Where a feature and a principle appear to disagree, the principle governs.

## P1 — Acceptance ≠ approval ≠ Run

Semantic acceptance (C11) creates a revision. Approval (C04) authorises exact bytes. Run (C05) is a separate explicit human command. None implies the next, and no surface, job or message may collapse them.
*Source: `requirements.md` I1.*

## P2 — AI proposes. Humans authorise. Core decides.

No AI path approves, launches, mutates a revision or reaches a runner. AI output is a proposal or labelled advice, always reviewable, never applied by itself.
*Source: I2.*

## P3 — The browser expresses intent, never authority

A tenant or project identifier from a client is a requested scope. Authority is derived on the server, as C10 `TrustedActorTenantContext`, at every effect boundary.
*Source: I3; register `122:2` §4.*

## P4 — Only the platform catalog is global

The immutable C01 platform release is the only global, read-only data. Sources, revisions, conversations, assets, results, credentials and indexes are tenant- and project-scoped. "Credentials" here are customer-environment credentials; a global identity's own credential and factors are identity-scoped and never duplicated per tenant (FR-IDN-001).
*Source: I4.*

## P5 — Pure compilation

C03 contacts no target, database, device or provider, performs no probe and admits no wall-clock value into its hash. Purity is proven by tests, not asserted.
*Source: I5.*

## P6 — Fail closed

Absent, invalid, revoked, expired or `UNCONFIGURED` input blocks the dependent effect. No default is assumed, no value is invented, no fallback is attempted.
*Source: I6.*

## P7 — Approved bytes are the executed bytes

Any change to an execution-affecting pin invalidates the affected approval. Pins are never silently upgraded; "latest" is discovery only — resolve, then pin.
*Source: I7.*

## P8 — No fabricated outcome

Unknown is never PASS and never a failed assertion. NOT EXECUTED is not failure. A projection is never verdict authority.
*Source: I8.*

## P9 — Verification is part of the requirement

A feature is done when the verification stated for its area in `requirements.md` passes. This document is not evidence that any test has passed.
*Source: I9.*

## P10 — Fixed technology stack

| Tier | Stack |
|---|---|
| BE | Java 21 · Spring Boot 3.5 · PostgreSQL · existing Spring Security with JWT, refresh and MFA · ZXing `core` (`com.google.zxing:core`, Apache License 2.0) only to encode the TOTP setup graphic as SVG (register `122:2` A-46; specification 003 plan D15) |
| FE | React 18 · TypeScript · Vite · TanStack Query · Zustand · i18next · Tailwind · axios · STOMP |
| Core | Java 21 · Gradle · Cucumber · six modules `common`, `http-core`, `web-core`, `mobile-core`, `database-core`, `mobile-agent` · Playwright Java for Web · Appium Java for Mobile on ASpectra-managed device hosts (macOS hosts for iOS) · the existing HTTP client and Hikari / JDBC. The runner host is a BE-owned separate deployment that embeds pinned Core (P11; scope §2) |

Bounded chat and semantic assistance run behind existing Java BE application ports. No new identity provider, mandatory broker, custom-role builder, team hierarchy, payment processor, customer-hosted inference, mandatory vector database or new DI or form framework is introduced.
*Source: register `122:2` §1.*

## P11 — Architecture styles

- **BE** is a hexagonal, bounded-context modular monolith with PostgreSQL durable claims and a transactional outbox. Domains own their rules and data and coordinate through application ports. The runner is a separate deployment, not a microservice per agent.
- **FE** is a feature-modular SPA with app-level composition and acyclic imports. TanStack Query owns server facts; Zustand owns only deliberate unsaved editor forks. The FE expresses intent and projects BE truth; it never decides authorisation or a verdict.
- **Core** is a deterministic plugin-based kernel with a public API / SPI and explicit resource scopes. Pure validation and compilation are separate from Cucumber and protocol execution. Membership, AI, provider delivery and business persistence stay outside Core.

*Source: register `122:2` §2.*

## P12 — Two independent tenancy gates

Every customer row carries `tenant_id` (and `project_id` when project-scoped) with composite keys; PostgreSQL `FORCE RLS` applies with `USING` and `WITH CHECK`; context is set with `SET LOCAL` inside the transaction. RLS and runtime authorisation are both mandatory, and neither replaces the other.
*Source: register §4; BE board `23:5587`.*

## P13 — Configured values are never defaulted

Every retention window, deadline, attempt count, delay, cap, ceiling and limit is validated configuration. Unsafe absent or unbounded configuration fails startup or fails admission. Where the design shows a number, it is an illustrative sample value, never a default.
*Source: FR-X-046; register `122:2` §15.*

## P14 — Release bar

WCAG 2.2 AA on critical journeys, with automated and manual assistive evidence. The interface ships in EN / ES / FR; interface language never changes an executable test.
*Source: FR-X-040; FR-X-043; FE board `46:7847`.*

## P15 — Managed execution

Execution runs only on ASpectra-managed runners with explicitly offered profiles. Incompatible or unreachable target-profile combinations fail admission; a scenario is never silently distributed.
*Source: register `122:2` §1; Core board `46:7241`.*

## P16 — Governance

1. **Design authority order.** Decision register `122:2` (its §13 record and its §15 operator authority) → the canonical boards on page `0:1` (BE `23:5587`, Core `46:7241`, FE `46:7847`, Alignment `46:8271`, Import `79:3652`, Authoring `79:4061`, Failure `3:2192`) → the product experience screens on page `148:2`, with the D01 responsive policy (`414:2645`) binding and the D03 Screen decisions ledger (`148:4`) recording every screen decision S-nn. Where two design sources disagree, the higher one governs.
2. **Admission to this document.** A feature enters this constitution only when its behaviour, rules and screens are completely defined in the design.
3. **Amendment.** Changes are made by a new version of this document. A feature is never edited in place without a version change.
4. **Division of ownership.** `requirements.md` owns the wording of each requirement; this document owns which requirements, and which parts of them, form the baseline.
5. **Specification-level questions.** A question that a feature specification cannot answer from the design is recorded in decision register `122:2` §14 (A-nn) or in the D03 Screen decisions ledger `148:4` (S-nn) and listed by that specification; a question that depends on neither is decided with the product owner and recorded in the specification's decisions table (§1.4). It does not remove the feature from this baseline unless its resolution changes the feature's behaviour, rules or screens.

---

# Part II · Feature catalog

## X · Cross-cutting

### F-X-01 · C10 trusted context and database isolation

Every protected operation resolves a server-derived `C10 TrustedActorTenantContext` before any effect and rechecks authority at every effect boundary. PostgreSQL enforces ownership through composite keys and `FORCE RLS` under a non-privileged runtime role. Core receives only an opaque execution context.

**Acceptance rules**
- Client-supplied tenant or project identifiers are requested scope only; route-versus-body mismatch or absent context denies safely.
- Each transaction, async job, `REQUIRES_NEW` boundary and retry establishes C10 afresh; pooled connections never inherit scope.
- Sensitive writes check `permissionVersion` transactionally.
- Schedulers enumerate tenant and job metadata through a narrow control plane, then claim work in tenant scope; the cursor never reads customer tables.
- `SECURITY DEFINER` functions and views are audited.

**FRs:** FR-X-001, FR-X-002, FR-X-003, FR-X-004, FR-X-005, FR-X-006, FR-X-010, FR-X-011, FR-X-012, FR-X-013, FR-X-014, FR-X-015
**Design source:** register `122:2` §4; BE board `23:5587`; Core board `46:7241`.

### F-X-02 · Request integrity

Mutating operations are idempotent on `(tenantId, projectId, operation, idempotencyKey, bodyHash)`; request context is frozen per operation; a lost acknowledgement leads to a lookup, never a resubmission.

**Acceptance rules**
- Same key and same body returns the same result; same key with a different body is a conflict.
- A tenant switch never retargets an in-flight save or command.
- Safe reads retry with configured bounds; consequential writes never use generic auto-retry, and no POST replay queue exists.

**FRs:** FR-X-020, FR-X-021, FR-X-022, FR-X-023
**Design source:** register `122:2` §4, §8; Failure board `3:2192`.

### F-X-03 · Secrets, redaction and safe problems

Secrets exist only as references in the existing encrypted credential repository. Redaction happens before content reaches any prompt, log, index, spool, report or export. Clients receive safe problem codes.

**Acceptance rules**
- Passwords, OTPs, MFA seeds, recovery codes and reset tokens are never logged, exported or stored in plaintext.
- Sensitive literals in authored text become `SecretRef`, and the author is shown the transformation.
- Problem codes carry field paths and a correlation ID, and distinguish at least tenant denial, stale revision, absent mapping, policy change, AI quota, uncertain acceptance and expired evidence.
- Browser storage holds only an allowlist of harmless preferences.
- Previews never inline untrusted HTML or SVG; MIME, scheme, filename and size are validated and blob URLs revoked.

**FRs:** FR-X-030, FR-X-031, FR-X-032, FR-X-033, FR-X-034, FR-X-035
**Screens:** D02 `159:32045` safe problem panels; N42 SystemErrorPage `430:4056` (403 `430:4071`, 500 `430:4086`, 429 `430:4101`, maintenance `430:4116`); M31 `435:4523`.
**Design source:** register `122:2` §6, §9, §11; FE board `46:7847`.

### F-X-04 · Accessibility and languages

Critical journeys meet WCAG 2.2 AA. All ordering is keyboard-operable with announced position. The UI locale, Gherkin dialect, registered expression alias and C01 `stepId` remain four separate concepts.

**Acceptance rules**
- Evidence: axe + Playwright, plus NVDA (Chrome, Firefox) and VoiceOver (Safari macOS / iOS), versions pinned in a release-support matrix.
- Status is conveyed by text, not colour alone; instructions and errors are programmatically associated with their fields; live announcements never steal focus.
- Touch targets on responsive surfaces are at least 44 px; no viewport introduces drag-only ordering.
- Every critical-journey surface satisfies AA reflow; a surface with a designed counterpart follows that design.
- Customer-facing copy says "workspace"; "tenant" remains the model and specification term (D01 block 09 `414:2645`, which holds the responsive policy and the glossary). Contract codes (C01 … C12) never appear in customer copy; US spelling ("Canceled") is used.

**FRs:** FR-X-040, FR-X-041, FR-X-042, FR-X-043, FR-X-053, FR-X-054, FR-X-055
**Design rule:** "Help & shortcuts" (shell) and "Need help?" / "Help" (public screens) open N43 HelpShortcutsPanel `430:4129` — keyboard map, static guidance, no chat.
**Screens:** N43 HelpShortcutsPanel `430:4129`.
**Design source:** FE board `46:7847` WCAG table; D01 `159:31949` (policy and glossary `414:2645`).

### F-X-05 · Audit log

Security- and business-relevant events are appended to an append-only, redacted audit log with actor, object lineage and decision or fact. Visibility follows current authority.

**Acceptance rules**
- Tenant Owner or Admin membership alone never authorises reading a project audit payload.
- The audit trail renders entries in lineage order with safe actor, decision or fact and a detail link.

**FRs:** FR-X-044 (part)
**Scope of partial requirements:** FR-X-044 — append-only redacted log, its content and authority-scoped visibility.
**Screens:** N18 AuditTrail `161:50756`.
**Design source:** register `122:2` §5.

### F-X-06 · Configuration validation and transactional email port

Validated configuration supplies every numeric value; a transactional email port is an operational prerequisite for verification, invitations, recovery and security notification.

**Acceptance rules**
- Unsafe absent or unbounded production configuration fails startup or admission.
- The email adapter sits behind a port; optional run notifications are not a substitute for it.

**FRs:** FR-X-045, FR-X-046
**Design source:** board 1 completion `23:5587`; register `122:2` §15.

### F-X-07 · Responsive surfaces

The designed mobile and tablet counterparts are genuine reflows that preserve every safety statement, gate, pin and boundary of their desktop surface.

**Acceptance rules**
- Counterparts: M01 landing (and its menu-open state) · M02 login (email and password step) · M03 invitation (review and decision screen) · M04 tenant overview · M05 mapping review · M06 approval review · M07 execution live · M08 execution results · M09 queued run · M10 signup · M11 verify email · M12 MFA challenge · M13 TOTP enrollment · M14 recovery-codes reveal · M15 forgot password · M16 reset password · M17 forced password change · M18 account recovery · M19 invitation pre-match · M20 workspace chooser · M21 workspace suspended · M22 account privacy · M23 session-expiry sheet · M24 guarded-switch sheet · M25 execution queue · M26 queued run detail · M27 admission blocked · M29 delivery jobs · M30 partial-delivery reconcile · M31 system error · T01 tablet authoring · T02 tablet project dashboard (M28 is unassigned; N12's counterpart is M06).
- Every other surface follows the binding D01 responsive policy (`414:2645`): breakpoints 390 / 768 / 1024 / 1440; one record card per table row carrying every cell, statement and action; dialogs become sheets at or below 768 px; the primary action stays reachable; every safety statement, gate, pin and boundary is kept.
- Mobile navigation is the M04 pattern: a menu plus a bar of permitted destinations, following current server-derived grants.

**FRs:** FR-X-050 (part), FR-X-051, FR-X-052
**Scope of partial requirements:** FR-X-050 — the counterparts listed above, as drawn, and the D01 policy for every other surface.
**Screens:** M01 `161:51034` (menu open `429:4200`), M02 `161:51247`, M03 `161:51284`, M04 `161:51344`, M05 `161:51423`, M06 `161:51536`, M07 `191:34119`, M08 `191:34221`, M09 `191:34883`, M10 `431:4100`, M11 `431:4135`, M12 `431:4162`, M13 `431:4190`, M14 `431:4222`, M15 `431:4243`, M16 `431:4271`, M17 `431:4300`, M18 `432:4185`, M19 `432:4231`, M20 `432:4257`, M21 `432:4317`, M22 `432:4356`, M23 `432:4427`, M24 `432:4464`, M25 `435:4294`, M26 `435:4351`, M27 `435:4392`, M29 `435:4416`, M30 `435:4460`, M31 `435:4523`, T01 `191:34359`, T02 `191:34521`.
**Design source:** D01 `159:31949` responsive policy (`414:2645`); D03 `148:4` responsive table.

---

## IDN · Identity and tenancy

### F-IDN-01 · Global identity and signup

A single `GlobalIdentity` (issuer + subject, email, display name, status), separate from tenant membership and project grants. Signup collects name, email, password and terms acknowledgement only.

**Acceptance rules**
- Passwords use the maintained Spring Security `PasswordEncoder`; no reversible storage.
- Password rules are supplied by server policy and displayed; the client never hard-codes them.
- No organisation data and no activation queue at signup.

**FRs:** FR-IDN-001, FR-IDN-002, FR-IDN-003, FR-IDN-010, FR-IDN-011
**Screens:** R04 SignupPage `148:223`; M10 `431:4100`.
**Design source:** register `122:2` §3.

### F-IDN-02 · Email verification

Email is verified before the identity may create a workspace or accept an invitation; verification alone grants no access.

**Acceptance rules**
- Three states: pending, confirmed, link expired or invalid; the expired state discloses nothing and offers a new link; only the newest link works.
- Resend and "use a different email" never disclose whether an account exists.
- Email domain never auto-joins a tenant; identities are never auto-linked by matching email.

**FRs:** FR-IDN-012, FR-IDN-013, FR-IDN-014, FR-IDN-015
**Screens:** N01 VerifyEmailPage `161:49169`; D02 `159:32045` link state; M11 `431:4135`.
**Design source:** register identity completion.

### F-IDN-03 · Sign-in and abuse protection

Local verified email and password with generic, comparable responses and layered throttling.

**Acceptance rules**
- Unknown identity, wrong password and restricted account produce comparable responses, with a dummy encoder verification for unknown identities.
- Account-aware plus IP / network / risk throttling with progressive delays and bounded temporary restriction; no attacker-triggerable permanent lockout.
- No account is created and no password stored for an unknown username.

**FRs:** FR-IDN-020, FR-IDN-021, FR-IDN-022
**Screens:** R02 LoginPage `148:290`; M02 `161:51247`; M12 `431:4162` (challenge).
**Design source:** register identity completion; board 1 completion.

### F-IDN-04 · TOTP challenge and enrollment

TOTP is mandatory for Owner and Admin before privileged authority; first-factor success is a limited identity state until the challenge completes.

**Acceptance rules**
- Tenant policy may additionally require MFA for Members.
- The challenge is bound to identity, login transaction, purpose, expiry and attempt limit, with replay control; it accepts a pasted code and offers "use a recovery code" and controlled recovery; no remembered-device bypass.
- Enrollment is three steps — set up, verify, save recovery codes — and is confirmed before activation.
- The seed is encrypted under a server key reference and never hashed. It is shown to the client once, during enrollment step 1, as the manual-entry alternative to the QR code (R39); it is never stored client-side and never retrievable afterwards (specification 003 RF-2.2, RF-11.3).
- Required MFA cannot be disabled by any tenant action.

**FRs:** FR-IDN-023 (part), FR-IDN-024, FR-IDN-025, FR-IDN-026, FR-IDN-027, FR-IDN-030
**Scope of partial requirements:** FR-IDN-023 — mandatory TOTP for Owner and Admin, including a solo founding Owner, and the optional tenant policy requiring it for Members.
**Screens:** R05 MfaChallengePage `148:179`; R39 TOTPEnrollmentWizard `148:3973`; M12 `431:4162`; M13 `431:4190`.
**Design source:** register identity completion; BE board `23:5587`.

### F-IDN-05 · Recovery codes and factor replacement

High-entropy, hashed, single-use recovery codes revealed exactly once; factor replacement never removes the existing factor first.

**Acceptance rules**
- Reveal requires explicit "saved" acknowledgement; codes never reach browser storage.
- Regeneration requires policy-approved re-authentication and invalidates the whole previous set.
- Replacement requires the current factor (plus password per policy) or controlled recovery; it triggers notification and session revocation.

**Design rule:** R36 offers "Download securely issued codes" as part of the one-time reveal (`148:1333`); a user-initiated file download is not browser storage in the sense of FR-IDN-028, and the codes are never written to local or session storage.

**FRs:** FR-IDN-028, FR-IDN-029
**Screens:** R35 RegenerateRecoveryCodesDialog `148:1317`; R36 RecoveryCodesRevealDialog `148:1333`; R41 FactorReplacementDialog `148:3995`; R38 SecuritySettingsPage `148:3895`; M14 `431:4222`.
**Design source:** board 1 completion.

### F-IDN-06 · Password reset and forced change

Generic, independently rate-limited reset requests; hashed, purpose-bound, expiring single-use tokens; atomic consumption.

**Acceptance rules**
- Requests never lock the account and respond generically.
- Tokens are delivered only to the verified channel over a configured trusted HTTPS origin, never a request-controlled `Host`.
- Successful reset changes the credential, revokes refresh families and sessions, invalidates other links, notifies, and returns to normal password + MFA sign-in.
- Reset never auto-logs-in, restores membership, lifts suspension or touches MFA, and says so on screen.
- Used or expired links lead to a distinct safe "request a new link" state.
- Forced password change is a gate inside the sign-in transaction, before any second-factor challenge and before any landing (specification 004, decision Q2; register A-45): it requires the current password, never replays a pending Run, and "Sign out safely" voids the transaction.

**FRs:** FR-IDN-040, FR-IDN-041, FR-IDN-042, FR-IDN-043, FR-IDN-044, FR-IDN-045
**Screens:** R03 ForgotPasswordPage `148:264`; R07 ResetPasswordPage `148:86`; R08 ForceChangePasswordPage `148:43`; D02 `159:32045`; M15 `431:4243`; M16 `431:4271`; M17 `431:4300`.
**Design source:** board 1 completion; register identity completion.

### F-IDN-07 · Controlled account recovery

An identity that has lost its authenticator recovers through one of three paths on one screen: (a) an unused recovery code, with the password, which leads straight into protected factor replacement; (b) a verified recovery request when no code is available, which opens a case reviewed through the platform's internal, audited, out-of-band verified recovery process; (c) the continuation of an existing case by its reference, shown only after separate verification. Every path ends in a verified replacement of the factor and in nothing else.

**Acceptance rules**
- Every path requires the password plus something only the holder has — an unused recovery code, or control of the verified address and a case approved by the review; a code, a reference, a mailbox or a session alone grants nothing.
- A case reference alone grants no access; case details are shown only after separate verification; identity, ownership and factor-replacement checks remain required; no completion time is promised.
- Recovery never bypasses suspension, revoked membership or required MFA; existing MFA and tenant boundaries remain in place while a case is reviewed.
- The review of a case has no customer-facing surface; a tenant Admin never opens, moves or closes a case, never resets, replaces or disables another identity's password or MFA, and never erases another identity; there is no administrator backdoor.
- A recovery code is consumed when it verifies; a one-time verification code, an approval and a replacement each take effect at most once.

**FRs:** FR-IDN-046, FR-IDN-047, FR-IDN-048, FR-IDN-049
**Screens:** N05 ControlledAccountRecoveryPage `161:49444`; M18 `432:4185`.
**Design source:** board 1 completion; specification 005 (27 October 2026); register items A-47, A-48, S-38.

### F-IDN-08 · Sessions and refresh

Short-lived in-memory access credential; `Secure`, `HttpOnly` refresh cookie; atomic refresh-family rotation with reuse detection; a single-flight client refresh.

**Acceptance rules**
- Idle and absolute deadlines and a session / security version are enforced server-side; background polling is not human activity.
- Single-flight refresh applies only to eligible control-plane 401s, retries at most once with the original identity, tenant, project, body and idempotency context.
- A 403 refreshes permissions or shows denied scope without loops; transport failure is never shown as expiry; CSRF has its own handling.
- **An SUT 401 never refreshes ASpectra credentials**; ASpectra tokens never reach a customer target.
- On genuine expiry restricted content is masked, transient secrets cleared, explicit save offered while authorisation is still valid; a retained draft reappears only for the same identity and scope.
- Session expiry never cancels an accepted Run, and re-authentication never replays Run intent.

**Design rule:** R34 offers "Retry session refresh" (`148:489`).

**FRs:** FR-IDN-060, FR-IDN-061, FR-IDN-062, FR-IDN-063, FR-IDN-064, FR-IDN-065, FR-IDN-066, FR-IDN-067
**Screens:** R34 SessionExpiryDialog `148:489`; D02 "Verify to continue"; N28 ProjectAccessDeniedPanel (D02 `159:32045`); M23 `432:4427`.
**Design source:** register identity completion; board 3 completion `46:7847`.

### F-IDN-09 · Session management

Explicit logout revokes the session server-side; sessions are listed and individually revocable; "Sign out everywhere" ends every session of the identity.

**Acceptance rules**
- Logout is offered on N03, N20, M03 and from R34 ("sign out safely"); on R08 "Sign out safely" voids a sign-in transaction, not a session (F-IDN-06).
- The sessions table lists session, context, server-supplied expiry and action, marks the current session and revokes others.
- The surface states that a session is not a trusted device, grants no TOTP bypass, and that expiring a session does not cancel an accepted Run.
- "Sign out everywhere" asks one confirmation, advances the session / security version and ends every session and refresh family of the identity, the current one included; the person signs in again.

**FRs:** FR-IDN-068, FR-IDN-069, FR-SET-023, FR-SET-024
**Screens:** R38 SecuritySettingsPage `148:3895`.
**Design source:** register identity completion; specification 007 (27 October 2026); register items A-51, S-43, S-44.

### F-IDN-10 · Invitation acceptance

An issued invitation is hashed at rest, expiring, atomically single-use and bound to the invited verified email, tenant and roles. The invitee reviews exactly what is granted, then accepts or declines.

**Acceptance rules**
- Before acceptance the screen shows tenant, tenant role, project, explicit project grants and whether `canRun` is granted; the invitee confirms that membership and project grants are separate.
- The signed-in identity must match the invited address; switching identity requires sign-out.
- Expired, revoked or used invitations are not acceptable and direct the user to request a fresh one.
- No role self-escalation.
- A new identity creates a local account with the invited address (shown masked; the server refuses any other address), verifies it, signs in and returns to the invitation; required MFA is gated at tenant entry, not before acceptance (specification 008, decisions Q3 and Q6).

**Design rule:** M03 offers "Decline invitation" beside acceptance (`161:51284`).

**FRs:** FR-IDN-080, FR-IDN-081, FR-IDN-082, FR-IDN-083, FR-IDN-084, FR-IDN-085
**Screens:** N02 InvitationAcceptancePage `161:49265`; N29 InvitationWrongIdentityPanel (D02 `159:32045`); M03 `161:51284`; M19 `432:4231`.
**Design source:** register `122:2` §3.

### F-IDN-11 · Roles, grants and the authority matrix

Tenant roles are exactly Owner, Admin, Member; project roles are exactly Author, Reviewer, Viewer; `canRun` is a separate fixed project grant. The authority matrix of FR-IDN-091 is implemented exactly.

**Acceptance rules**
- Tenant administration never implies project content access, and the membership and policy surfaces say so.
- Membership status `active | suspended | revoked` with `permissionVersion`; tenant status `active | suspended | deleting`.
- Revocation denies future commands and subscriptions; in-flight work is cancelled at bounded checkpoints and never presented as undone.
- Membership and administration changes are versioned and audited.

**FRs:** FR-IDN-090, FR-IDN-091, FR-IDN-092, FR-IDN-093, FR-IDN-094, FR-IDN-099
**Screens:** D03 authority matrix `148:4`.
**Design source:** board 1 §04 `23:5587`.

### F-IDN-12 · Tenant members, invitations and ownership transfer

Owners and Admins see the member list, invite people, review, resend or revoke pending invitations, change tenant roles, suspend, reinstate or remove memberships, and an Owner transfers ownership; the last active Owner is protected.

**Acceptance rules**
- The last active Owner cannot be removed, demoted or suspended; every change that creates or removes an Owner is serialised under the tenant ownership lock, never decided by count-then-write.
- Ownership transfer is a step-down to a verified, enrolled member: the actor re-authenticates with password and authenticator code, current membership and eligibility are rechecked at confirmation, and the successor becomes Owner while the actor becomes Admin in one indivisible change. Promoting to Owner, demoting an Owner and suspending or removing an Owner require the same re-authentication and recheck.
- An Admin administers Members and Admins only, never an Owner's row, and never assigns or invites Owner; nobody edits or manages their own row.
- An invitation carries Admin or Member and at most one project's explicit grants, one pending per address; it is resendable with a fresh secret and revocable; issuing never discloses whether an identity exists for the address.
- Suspension keeps grants inert and is reversible; removal revokes membership and grants and retains authored evidence; neither touches the identity, its credentials or its other memberships, and neither ends a session: authority is re-derived at the next request.
- Every change is versioned, audited with actor and previous and new value, and notified to the person concerned (an invitation's revocation or expiry is not a membership change and sends nothing, specification 010 Q10); the inviter is notified of acceptance and decline.

**FRs:** FR-IDN-095, FR-IDN-097
**Screens:** R13 TenantMembersPage `148:503`; N23 OwnershipTransferDialog (D02 `159:32045`).
**Design source:** register `122:2` §3; specification 010 (27 October 2026); register item S-50.

### F-IDN-13 · Project grants editor

Project grant administration sets Author, Reviewer, Viewer and `canRun` independently per member, beside the member's tenant role.

**Acceptance rules**
- The inline editor ("Edit grants" panel, "Save project grants") changes one member's grants; `canRun` is independent of the roles.
- The surface states that revocation blocks new work while authored evidence and provenance remain, and that permission is rechecked at each protected action.

**FRs:** FR-IDN-098, FR-SET-052
**Screens:** R17 ProjectMembersPage `148:1754` (panel `159:30526`, save `159:30539`).
**Design source:** board 1 §04.

### F-IDN-14 · Workspace creation and bootstrap

A verified, TOTP-enrolled identity creates a workspace and becomes its founding Owner, optionally with a first project.

**Acceptance rules**
- Creation requires verified email, completed TOTP enrollment with recovery codes acknowledged, and a workspace name; it is blocked, never bypassed, while enrollment is incomplete. TOTP enrollment precedes workspace creation.
- An optional first project requires explicitly ticked Author / Reviewer / `canRun` grants for the founding Owner, with the statement that the Owner role grants none of them automatically.
- Tenant creation and first-Owner / first-grant bootstrap form one auditable transaction of explicit grant records.

**FRs:** FR-IDN-096, FR-IDN-113, FR-IDN-114, FR-IDN-115
**Screens:** N04 CreateWorkspacePage `161:49374`; N22 CreateProjectDialog (D02 `159:32045`).
**Design source:** register `122:2` §3; Authoring board `79:4061`.

### F-IDN-15 · Tenant chooser

The chooser lists only the signed-in identity's memberships, with role, state and eligible projects.

**Acceptance rules**
- A suspended tenant shows its access state only — no project names, sources or run details.
- The chooser states that switching context does not retarget accepted runs.

**FRs:** FR-IDN-110, FR-IDN-111, FR-IDN-112
**Screens:** N03 TenantChooserPage `161:49326`; M20 `432:4257`.
**Design source:** register `122:2` §3.

### F-IDN-16 · Tenant suspension state

A suspended tenant freezes new effects, refreshes permissions and subscriptions, and requests bounded cancellation of active work.

**Acceptance rules**
- Freezing happens before retrieval, acceptance, approval, launch, provider submission or delivery; bounded cancel preserves known facts; acknowledgement is not rollback.
- The suspended screen states that accepted runs do not become failed tests and verdicts are not rewritten, and offers personal account security and sign-out.
- The screen offers "Review verified workspace recovery" — a read-only panel with the configured recovery contact, or the statement that none is configured — and the route to the workspace chooser (specification 014, decision Q2; D02 `421:3131`).

**FRs:** FR-IDN-116
**Screens:** N20 TenantSuspendedPage `161:50995`; M21 `432:4317`.
**Design source:** Failure board `3:2192` ("Tenant suspended / membership revoked").

### F-IDN-17 · Leave tenant and identity erasure request

Leaving a tenant and erasing the global identity are two different operations.

**Acceptance rules**
- Leaving revokes that tenant's access and grants, preserves the identity, other memberships and credentials, and is blocked while the identity is a last Owner (directing to transfer).
- An erasure request requires the stated acknowledgements, typed confirmation, and last-Owner transfer in every owned tenant; it is never one-click and promises no instant erasure.
- The identity is verified independently of the session — current password with a fresh authenticator code, then a one-time code sent to the verified address — before a request is recorded.
- Recording the request places the identity under erasure and ends every one of its sessions in one change; the request is handed to the platform's internal, audited erasure process, and no completion time or retention duration is promised.
- The surface shows only the signed-in identity's own ownership checks.

**FRs:** FR-IDN-120, FR-IDN-121, FR-IDN-122 (part), FR-IDN-123
**Scope of partial requirements:** FR-IDN-122 — the erasure-request gates (acknowledgements, typed confirmation, last-Owner checks across all memberships, no one-click and no instant-erasure promise), the independent verification of the identity, the recording of the request with the transition of the identity to under erasure, and the revocation of its sessions; the identity's credentials become unusable at once because an identity under erasure cannot sign in (F-IDN-03; specification 001 RF-2.7), while erasing the credentials themselves and minimising or pseudonymising retained attribution are the internal erasure process and not part of this baseline.
**Screens:** N19 AccountDeletionPage `161:50881`; M22 `432:4356`.
**Design source:** board 1 completion; specification 015 (28 October 2026); register items A-64, S-63.

### F-IDN-18 · Guarded tenant and project switch

A tenant or project switch is a guarded client state transition in a fixed order.

**Acceptance rules**
- Order: (1) dirty-editor guard Save / Discard / Stay; (2) abort in-flight requests and disconnect STOMP; (3) advance the scope epoch and clear scoped Query cache and Zustand drafts; (4) fetch current membership and grants; (5) only then load data or open a stream.
- A guard-triggered save is submitted under the original tenant / project / `baseETag`.
- Late responses and events are discarded by epoch; cache clearing is never treated as authorisation.
- Entry points are the tenant chooser and the header WORKSPACE / PROJECT selectors.

**FRs:** FR-IDN-130, FR-IDN-131, FR-IDN-132
**Screens:** N24 TenantSwitchDirtyDialog (D02 `159:32045`); N03 `161:49326`; header selectors on the app-shell screens; M24 `432:4464`.
**Design source:** board 3 §03 `46:7847`.

### F-IDN-19 · Public landing

An unauthenticated landing presents the workflow, capabilities, human-control boundary and FAQ, with exactly two entry actions: create workspace and sign in.

**Acceptance rules**
- The landing states that acceptance, approval and Run are separate, AI is optional, verified email is required and the managed runner starts only by user decision.
- Every preview, identifier, count and outcome is labelled illustrative.
- The FAQ answers that importing or approving does not start a run, AI and connectors are optional, and interface language does not change a test.
- EN / ES / FR; only active identity entry methods appear.
- "Create workspace" routes to verified-email signup, then workspace creation — never directly into a tenant.
- Landing examples use HTTP, Web and DB scenarios.
- Privacy, Terms and Contact open documents the platform operator publishes outside the product, with an external-link cue before activation; their EN / ES / FR destinations are operator configuration (register `122:2` §15), never invented or derived from a tenant, and a release with any destination absent or malformed is refused.

**FRs:** FR-IDN-140, FR-IDN-141, FR-IDN-142, FR-IDN-143, FR-IDN-144, FR-IDN-145
**Screens:** R01 LandingPage `148:341`; M01 `161:51034` (menu open `429:4200`).
**Design source:** board 3 `46:7847`; register `122:2` §15 (public documents); D04 `437:4431` (landing strings, share metadata); specification 017, decision Q3.

---

## PRJ · Projects and structure

### F-PRJ-01 · Ownership tree

`Tenant → Project → Suite → Feature → Scenario → immutable TestRevision`, every child carrying ownership foreign keys; selection is a versioned snapshot.

**Acceptance rules**
- Reordering preserves `stepOccurrenceID`s; duplication creates new ones.
- Approved content is never silently moved between owners; content belongs to one project.
- Trash and history views have dedicated scope checks; database cascades never erase required execution history.

**FRs:** FR-PRJ-001, FR-PRJ-002, FR-PRJ-003, FR-PRJ-004 (part), FR-PRJ-050, FR-PRJ-051
**Scope of partial requirements:** FR-PRJ-004 — no silent ownership move of approved content.
**Design source:** BE board ownership ERD `23:5587`; register `122:2` §5.

### F-PRJ-02 · Project directory and creation

The projects list shows only accessible projects in the current tenant with the viewer's grants, lifecycle state and readiness. Owners and Admins create projects with explicit initial grants.

**Acceptance rules**
- An action unavailable for lack of a grant is hidden or disabled with the required grant named; a Viewer cannot create revisions, approve or execute.
- Archived content blocks new authoring while remaining readable.

**Design rules:** the create dialog sets initial roles Author / Reviewer / Viewer plus `canRun` by explicit checkboxes (N22, D02 `159:32045`); the list offers a "My grants" filter (R14 `148:2226`).

**FRs:** FR-PRJ-010, FR-PRJ-011, FR-PRJ-012, FR-PRJ-014
**Screens:** R14 ProjectsPage `148:2226`; N22 CreateProjectDialog (D02).
**Design source:** register `122:2` §3.

### F-PRJ-03 · Project detail

Project detail presents suites and features with working revision and readiness, the pinned context and the next explicit decision.

**Acceptance rules**
- Pinned context shows selected environment revision, approval state for the current revision and the viewer's grants.
- The surface states that an active suite is not execution eligibility.

**Design rule:** a progress header INTENT → MAPPING → REVISION → BUNDLE → APPROVAL → RUN locates the current object in the workflow (R15, R22, N09).

**FRs:** FR-PRJ-013
**Screens:** R15 ProjectDetailPage `148:2024`.

### F-PRJ-04 · Versioned project configuration

Project configuration is saved as a new configuration revision with an impact preview; saving contacts no target.

**Acceptance rules**
- Configuration holds project name, default environment selection and runtime profile.
- A default environment is a suggestion; every Run pins its environment explicitly; no auto-launch.
- The impact preview states that changing the environment or runtime dependency requires recompile and human review, and that dependent approval becomes stale.
- No runtime overrides, no automatic failed-once retry.
- Project AI restrictions can only tighten tenant and operator permissions.

**FRs:** FR-PRJ-020, FR-PRJ-021 (part), FR-PRJ-022, FR-PRJ-023 (part), FR-PRJ-024, FR-PRJ-025, FR-SET-050 (part), FR-SET-051 (part)
**Scope of partial requirements:** FR-PRJ-021 — name, default environment, runtime profile. FR-PRJ-023 — impact of environment and runtime changes. FR-SET-050 — the Configuration view saved as a revision. FR-SET-051 — the tighten-never-expand rule within the effective AI policy intersection.
**Screens:** R16 ProjectSettingsPage `148:1885` (Configuration).

### F-PRJ-05 · Suites and selection

Suites are listed in Active / Archived / Trash views; suite detail selects exact scenario revisions with a pre-review exclusion preview.

**Acceptance rules**
- Lists are searchable and filterable by readiness.
- The selection preview lists exclusions explicitly (unmapped, environment unset, archived); exclusions are reviewed before approval; no runtime override re-includes a scenario; no new exclusion appears at Run.

**FRs:** FR-PRJ-030, FR-PRJ-031, FR-PRJ-032
**Screens:** R21 SuitesPage `148:2958`; R22 SuiteDetailPage `148:2854`.

### F-PRJ-06 · Feature detail and versioned Background

Feature detail shows scenarios with revision, linked criteria and readiness, and the shared Background as a versioned dependency.

**Acceptance rules**
- Editing a Background creates a new revision, marks dependent bundles stale and runs nothing.

**FRs:** FR-PRJ-033, FR-PRJ-034
**Screens:** R23 FeatureDetailPage `148:2738`.

### F-PRJ-07 · Lifecycle semantics, Trash and Restore

Archive, Trash and Restore are distinct operations; Purge is performed by retention workers outside this baseline (III.4). Deletion records tombstones with cascade provenance and immediately blocks descendants; Restore produces a new working revision.

**Acceptance rules**
- Deletion of a project, suite, feature or scenario atomically records a tenant-scoped tombstone with `deletionOperationId`, cascade provenance and outbox work.
- Parent state blocks descendant reads, edits, approvals and new runs immediately, before asynchronous cleanup.
- Deletion cancels queued work and requests bounded stop of active work; acknowledgement is not termination.
- Whole-project lifecycle needs Owner or Admin plus an explicit project grant; content lifecycle needs an authorised Author; Reviewer, Viewer or `canRun` alone never authorise deletion.
- Restore checks grants, an active parent, name conflicts and dependencies, shows deletion provenance and the eligibility window, never resurrects independently deleted children, and produces a new C02 requiring new C03 and C04 — never reviving an approval.

**FRs:** FR-PRJ-040 (part), FR-PRJ-041, FR-PRJ-042, FR-PRJ-043, FR-PRJ-044, FR-PRJ-045, FR-PRJ-046, FR-PRJ-047
**Scope of partial requirements:** FR-PRJ-040 — Archive, Trash and Restore as distinct operations; Trash is reversible within a configured window.
**Screens:** N15 TrashRestorePage `161:50390`.
**Design source:** board 1 completion; register identity completion.

### F-PRJ-08 · Environment revisions

A customer target `Environment` is a project-owned domain object with immutable `EnvironmentRevision`s. Saving, validating and approving never contact the target.

**Acceptance rules**
- A revision carries tenant / project, `environmentId`, `envRevision`, name, status, owner and `envSnapshotDigest`.
- Bindings: `targetAliases`; allowed origins, ports and paths; API and Web base URLs; DB connection `SecretRef`s; typed references to Locator, Schema and PublicTrustMaterial revisions.
- Execution policy: locale, timezone, limits, egress policy, secret binding policy; the resolved key version is recorded, never the value.
- Targets are `EnvTargetRef(alias, relativePath)`; a full URL only where policy allows; URI semantics, never string concatenation; a non-secret resolved-URL preview.
- Editing creates a new revision from the prior one, with a new hash and the statement that changed pins require recompile and a fresh C04.
- Unset, archived, revoked or incompatible environments block compile, approval or Run with an actionable finding.
- Configuration validity and fresh profile assessment are separate columns; an absent assessment blocks Run only.
- An unpublished `ManagedNetworkProfile` renders `UNCONFIGURED`, admission blocked, with no inferred address or region.
- Display name is never evidence of region or residency; no VPN or private-network promise.
- Secrets are write-only references.
- Runtime outbound controls forbid metadata, loopback and unapproved targets; static URI checks are not proof of availability.

**FRs:** FR-PRJ-060, FR-PRJ-061, FR-PRJ-062 (part), FR-PRJ-063 (part), FR-PRJ-064, FR-PRJ-065, FR-PRJ-066, FR-PRJ-067, FR-PRJ-068, FR-PRJ-069, FR-PRJ-070, FR-PRJ-071, FR-PRJ-072, FR-PRJ-074, FR-PRJ-075
**Scope of partial requirements:** FR-PRJ-062 — the bindings listed above. FR-PRJ-063 — locale, timezone, limits, egress policy, secret binding with recorded key version; every reference is an exact pin.
**Screens:** R19 EnvironmentsPage `191:32214`; N07 EnvironmentEditorPage `191:33369`.
**Design source:** board 1 completion; register "Customer target environment"; BE board `23:5587`.

### F-PRJ-09 · Public trust material

Project-scoped immutable `PublicTrustMaterialRevision`s hold only a public X.509 CA chain and its metadata. Intake classifies from the actual bytes and rejects keys before storage.

**Acceptance rules**
- The library lists revisions with digest, review and lifecycle state, validity and usage, and opens an exact revision.
- Intake shows declared-versus-detected content; private keys, client identities and key containers are rejected before storage, with only the reason and a safe fingerprint audited.
- The accepted-material table states per type: public CA chain and intermediates accepted, server leaf accepted as reference, private key and client identity or container rejected.
- Validation records subject and issuer from the chain, SHA-256 fingerprint, `contentDigest`, validity, CA constraints, key usage and scope binding.
- Replacement creates a new revision and new environment pins requiring recompile and fresh approval.
- Expiry and revocation are checked at admission and run time, never in the compiled hash.
- Trust applies per connection and profile on a published, tested adapter; `ignoreHTTPSErrors`, hostname bypass and global trust mutation never exist.
- Publication requires a project Reviewer **and** a tenant Owner or Admin.
- Raw certificate bytes never enter a prompt, report or AI context.

**FRs:** FR-PRJ-080, FR-PRJ-081 (part), FR-PRJ-082, FR-PRJ-083, FR-PRJ-084, FR-CAT-077, FR-CAT-078, FR-CAT-079, FR-CAT-080, FR-CAT-081, FR-CAT-082
**Scope of partial requirements:** FR-PRJ-081 — listing and inspecting exact revisions.
**Screens:** R20 CertificateManagerPage `191:32339` (library); N39 PublicTrustRevisionPage `191:34878` (intake and revision).
**Design source:** BE board `23:5587`; Core board `46:7241`.

### F-PRJ-10 · Jira Cloud source connection

A project holds versioned source connections with their own identity; Jira Cloud issues is the source family offered.

**Acceptance rules**
- A connection belongs to one provider family, is read-scoped to selected items and holds a protected credential reference.
- Saving stores configuration only; it never sends a comment, creates a defect, uploads, opens a pull request or runs a test.
- Provider families that are not offered appear disabled with their reason; no generic TMS claim, Git write or PR is offered.
- The surface states that one connector does not imply permission for every action and that provider contract tests and permission and dedupe verification precede activation.
- The console identity is never reused as target or provider authentication.

**FRs:** FR-PRJ-090, FR-PRJ-091 (part), FR-PRJ-092, FR-PRJ-093 (part), FR-PRJ-094, FR-SET-060 (part)
**Scope of partial requirements:** FR-PRJ-091 — Jira Cloud issues as the source family. FR-PRJ-093 — a disabled row with reason for each family not offered. FR-SET-060 — the Source connections and Delivery profiles views.
**Screens:** R18 ProjectIntegrationsPage `191:32090`.

### F-PRJ-11 · Project dashboard

The dashboard shows only permitted facts: linked criteria with an explicit denominator, the latest completed verdict, review readiness and the viewer's own activity.

**Acceptance rules**
- Coverage is linked, versioned criteria within a named source snapshot plus uncovered criteria; criteria outside the snapshot are unknown; complete coverage is never inferred from passing tests.
- Queued and NOT EXECUTED tickets are excluded from verdict denominators and pass rates.
- The dashboard states that a reviewable bundle is not a completed run.
- No dashboard action starts execution; no cross-tenant activity.

**FRs:** FR-PRJ-100, FR-PRJ-101, FR-PRJ-102, FR-PRJ-103, FR-PRJ-104
**Screens:** R11 ProjectDashboardPage `191:31936`; T02 `191:34521`.

### F-PRJ-12 · Tenant overview and first-run readiness

After tenant selection the overview lists only projects the viewer's grants permit, and a first-run checklist guides setup.

**Acceptance rules**
- The overview states that tenant administration does not grant content access, shows accessible scope, role and identity-protection state, and searches only within permitted scope.
- The checklist covers verified identity · Owner TOTP · a project with explicit grants · a reviewable environment revision · a first scenario authored, validated and reviewed.
- It states that integrations are optional, saving an environment contacts no target, no first run is automatic, and Author / Reviewer / `canRun` stay separate even for a solo Owner.

**FRs:** FR-PRJ-110, FR-PRJ-111, FR-PRJ-112, FR-PRJ-113, FR-PRJ-114, FR-PRJ-115, FR-PRJ-116
**Screens:** R10 TenantOverviewPage `148:989`; R09 OnboardingPage `148:1193`; M04 `161:51344`.

---

## CAT · Step catalog and library

### F-CAT-01 · C01 manifest registry

The Core build produces `C01 CatalogManifest` from actual glue and descriptors; BE ingests and retains the exact historical manifest.

**Acceptance rules**
- The manifest carries schema and catalog versions and hash, artifact digests, stable `stepId` + `contractRevision`, binding fingerprint, renderer and parameter schemas, prerequisite / output / effect / retry metadata, deprecation and unavailable reasons, and EN / ES / FR labels.
- Extraction reads compiled metadata without class initialisation or `ResourceFactory` calls, sorts deterministically and hashes canonical bytes.
- Extraction fails on duplicate IDs or aliases, alias cycles, ambiguous bindings and schema mismatch.
- No hand-maintained catalog; the client builds no catalog.
- Only the C01 release is global; everything else is tenant-owned.
- Given / When / Then / And / But share one matching namespace.

**FRs:** FR-CAT-001, FR-CAT-002, FR-CAT-003, FR-CAT-004, FR-CAT-005, FR-CAT-006, FR-CAT-007
**Design source:** board 2 §02 `46:7241`; Alignment board `46:8271`.

### F-CAT-02 · Availability axes and step resolution

Four availability axes — implemented ≠ offered ≠ runner-compatible ≠ live ready — each with an explicit reason when not satisfied; the six file handlers and the eight conditional "if-present" steps are `UNAVAILABLE` by construction.

**Acceptance rules**
- Runtime availability (C06) never rewrites the immutable C01 catalog.
- An unavailable, deprecated, revoked or incompatible capability blocks use and shows its reason.
- Historical step counts are never capability proof.
- No nearest-match, similar-step substitution, guessed alias, `eval` or inference; unknown fields render read-only with a reason.
- New executable capability comes only from a developer-controlled Core plugin, tests and an immutable release.

- The six file handlers stay `UNAVAILABLE` until both an actual offered C01 release and a tested resolver exist; publishing a fixture never activates a handler, and the surface says so. The eight "if-present" steps stay `UNAVAILABLE` intentionally, with the reason shown (register `122:2` §13.7).

**FRs:** FR-CAT-010, FR-CAT-011, FR-CAT-012, FR-CAT-013, FR-CAT-014, FR-CAT-015, FR-CAT-016
**Screens:** R25 StepLibraryPage `191:32643`; N37 ReusableAssetLibraryPage `191:34853`.
**Design source:** Core board `46:7241`.

### F-CAT-03 · Step library

Three tabs — global C01 capabilities, project macros, project assets — with exact-ID search and protocol and manifest-version filters.

**Acceptance rules**
- Rows show exact ID, typed arguments, availability and profile readiness.
- Detail shows manifest version, EN / ES / FR labels, typed arguments with the alias and the language that alias is registered in, and the four-axis statement.
- The library states that macros are typed compositions, never handlers, and never shadow Core steps.

**FRs:** FR-CAT-020, FR-CAT-021, FR-CAT-022, FR-CAT-023
**Screens:** R25 StepLibraryPage `191:32643`.

### F-CAT-04 · Macro library and macro editor

`MacroRevision`s are typed compositions of `CoreRef` and `MacroRef` nodes only, authored by an Author and published by a Reviewer.

**Acceptance rules**
- A revision carries ownership, kind, namespace, revision, typed inputs and outputs, ordered AST, exact dependency revisions, `expandedHash`, `sourceMap` and audit.
- A namespace never shadows a C01 `stepId` (blocking diagnostic); Java, glue, scripts and generated handlers are impossible.
- Depth, expanded step count and cycle check are enforced and shown as "actual of permitted" against configured values.
- Lifecycle `DRAFT → IN_REVIEW → PUBLISHED → DEPRECATED | REVOKED`.
- An invalid draft saves with its diagnostics, never publishes, and never compiles into a scenario; diagnostics attach to exact AST nodes with focus targets; no `expandedHash` while invalid; `sourceMap` retained on publish.
- BE expands published macros before C03; Core only receives Core steps.
- Publishing is never scenario approval and never launches a Run.
- Deprecation warns without upgrading pins; revocation blocks new admission and keeps history.
- The library lists namespace, version, dependencies, `expandedHash`, publication state and a usage-impact view; an unavailable dependency means review cannot create runtime support.

**Design rule:** macro review follows a separation policy; self-review is explicit and audited (R26 `191:32782`; Authoring board `79:4061`).

**FRs:** FR-CAT-030, FR-CAT-031, FR-CAT-032 (part), FR-CAT-033, FR-CAT-034, FR-CAT-035, FR-CAT-036, FR-CAT-037, FR-CAT-038, FR-CAT-039, FR-CAT-040, FR-CAT-041, FR-CAT-042
**Scope of partial requirements:** FR-CAT-032 — the `CoreRef` / `MacroRef`-only AST, node editing, keyboard reorder and removal.
**Screens:** R26 MacroLibraryPage `191:32782`; N36 MacroRevisionEditorPage `191:34848`.
**Design source:** Authoring board `79:4061`; Alignment board `46:8271`.

### F-CAT-05 · Schema, Locator and Fixture asset revisions

Reusable `AssetRevision`s of kind Schema, Locator and Fixture, with two independent lifecycle axes, immutable per revision and pinned into C03.

**Acceptance rules**
- A revision carries ownership, kind, schema version, `contentDigest`, media type, size, private storage reference, validation and scan policy version, author, reviewer, dependency closure and publication / revocation audit.
- Intake `UPLOADING → QUARANTINED → VALIDATING → READY | REJECTED` and publication `DRAFT → IN_REVIEW → PUBLISHED → DEPRECATED | REVOKED` are never collapsed; READY is not published, published is not approved, neither is a Run.
- Schema: supported dialect and root with the full local dependency closure pinned by digest. Locator: typed Web strategy, selector and scope; no interactive DOM inspector. Fixture: an immutable `FixtureRevision` selected by an exact typed `FixtureRef`; at run time it resolves only to a read-only `RuntimeFixtureRef` inside the run sandbox, validated against its `contentDigest`, and consuming steps take an explicit typed reference, never an implicit global or a latest-revision fallback (register `122:2` §13.7).
- The resolver never reads arbitrary paths, fetches asset content or a remote `$ref` over the network, or runs a generator.
- Upload authority needs an Author grant and configured caps; the actual bytes determine the kind.
- Intake runs in private quarantine with archive, traversal, zip-bomb, symlink and prohibited-reference controls, and never installs, executes or unpacks to a shared path.
- Scan status reads "a scan ran under a stated policy".
- Detail is non-editable; changes create new revisions; existing pins keep their revision; a "pinned by" table shows each pin and what replacement invalidates or revocation blocks; revocation is forward-only.
- Sensitive values are `SecretRef`s; raw content never enters a prompt, log, index or report; a sensitivity-rejected revision stays unpublished forever.

**FRs:** FR-CAT-050, FR-CAT-051, FR-CAT-052, FR-CAT-053, FR-CAT-054, FR-CAT-055, FR-CAT-056, FR-CAT-057, FR-CAT-058, FR-CAT-059, FR-CAT-060, FR-CAT-061, FR-CAT-062, FR-CAT-063
**Screens:** N37 ReusableAssetLibraryPage `191:34853`; N38 ReusableAssetRevisionPage `191:34858`.
**Design source:** BE board `23:5587`; Core board `46:7241`.

---

### F-CAT-06 · Mobile app revisions

Customer Android and iOS binaries enter as `MobileAppRevision`s: quarantined, validated on the actual bytes, reviewed, published immutably and pinned by digest into an environment revision; they run only on an explicitly offered compatible managed device profile.

**Acceptance rules**
- A revision records platform, `packageId`, version, `minOS`, architecture, signing identity verified from the actual bytes, provisioning metadata, `contentDigest`, size, scan status with policy version and compatible device profiles.
- A new binary is always a new revision; an alias is never repointed; replacing a pinned binary creates a new revision and a new environment hash and requires recompile plus a fresh human C04.
- Intake review shows each check — archive and path safety, malware scan, signing identity, provisioning profile against an offered device profile, device-profile compatibility — with its policy, outcome and whether it blocks publication; a failing check names its field.
- iOS accepts compatible signed and provisioned IPAs only; ASpectra never holds a customer private key and never re-signs a binary; an arbitrary IPA is never guaranteed.
- Rejection retains the quarantined bytes, every finding and the audit lineage, creates no published revision, no environment pin and no approval, and states the next step.
- An environment reference to a mobile or DB asset requires `READY` plus published state, signing / provisioning verification and device-profile compatibility; the surface states that scan status alone is not safety proof.
- Availability is evidence-gated: an offered profile name, a sample revision or this design is not proof of implementation, compatibility or readiness (register `122:2` §13.8).

**FRs:** FR-CAT-070, FR-CAT-071, FR-CAT-072, FR-CAT-073, FR-CAT-074, FR-CAT-076, FR-PRJ-073
**Screens:** N34 MobileAppRevisionsPage `191:34838`; N35 MobileAppIntakeReviewPage `191:34843`; N07 EnvironmentEditorPage `191:33369` (device profile and `MobileAppRevisionRef`).
**Design source:** register `122:2` §13.8; Core board `46:7241` (`362:173`); BE board `23:5587`.

---

## AUT · Authoring and review

### F-AUT-01 · Free-form intent

Authors write original wording without browsing the catalog; the wording is preserved exactly — text, hash and span.

**Acceptance rules**
- Structured building, free-form wording and source import converge on one canonical editing model, fully usable with AI disabled.
- Sensitive literals are extracted to `SecretRef` before persistence and the author sees the transformation.
- Save preserves wording and history and never accepts, approves or runs.

**FRs:** FR-AUT-001, FR-AUT-002, FR-AUT-003, FR-AUT-004
**Screens:** N09 FreeformAuthoringPage `161:49884`.
**Design source:** board 6 `79:4061`.

### F-AUT-02 · C11 mapping and acceptance

`C11 AuthoredIntentMapping` places original wording beside its executable meaning; a human accepts the meaning, producing an acceptance digest.

**Acceptance rules**
- The mapping records sanitized text, hash and span; candidate and accepted typed bindings; evidence, citations, assumptions and uncovered items; revision, status and acceptance digest.
- Executable differences are always shown.
- Unsupported or ambiguous meaning becomes a question or a retained uncovered item; no handler is guessed and no expectation is invented from observed output.
- A partial mapping is stated plainly, names the unmapped part, and offers clarification or an explicit narrowing recorded as an auditable decision record.
- Cosmetic normalisation is recorded without prompts; semantic change requires visible acceptance and invalidates C11, C03 and C04.
- Direct catalog selection records acceptance on explicit save.
- Uncovered items are excluded from coverage claims and remain visible against their criterion.

**FRs:** FR-AUT-010, FR-AUT-011, FR-AUT-012, FR-AUT-013, FR-AUT-014, FR-AUT-015, FR-AUT-016, FR-AUT-017
**Screens:** N09 `161:49884`; R29 SourceMappingReviewPage `148:3072`; N10 RevisionDiffPage `161:50000`; M05 `161:51423`.
**Design source:** board 3 §01 `46:7847`; Authoring board `79:4061`.

### F-AUT-03 · C02 canonical revision service

A deterministic BE domain service produces immutable `C02 TestRevision`s; C02 is the only editable canonical model.

**Acceptance rules**
- A revision carries ownership, `sourceIntentRef`, `acceptedMappingDigest`, exact `stepId` + `contractRevision`, stable `stepOccurrenceID`s, typed arguments, output refs, `secretRef`s, actor and resource refs, and pinned macro and asset revisions with provenance.
- No agent produces C02; new meaning creates a new revision; approved history is never mutated.
- Stale bases and foreign-owned references are rejected; uncovered items remain visible.
- Gherkin is a view, an import format or a compiled artifact — never a second source model.

**FRs:** FR-AUT-020 (part), FR-AUT-021, FR-AUT-022, FR-AUT-023, FR-AUT-024
**Scope of partial requirements:** FR-AUT-020 — every listed field except runtime asset-file references.
**Design source:** register `122:2` §6, §11; Alignment board `46:8271`.

### F-AUT-04 · Editor state and concurrency

Query owns server facts; Zustand owns only a deliberate unsaved fork. Saves are serialized and conditional.

**Acceptance rules**
- Query keys include tenant, project, `schemaVersion` and the immutable hash or version.
- The fork is pinned to `tenantId` / `projectId` / `baseETag` / `editSeq` with base, working copy, dirty flag and undo.
- A refetch never overwrites a dirty draft; "latest" is discovery only.
- A save captures content plus `editSeq`; newer edits stay dirty.
- `If-Match` strong ETag; 412 stale, 409 workflow conflict; base / local / server resolution, never blind merge or overwrite.
- The client never seals hashes or decides verdicts.
- Route guard Save / Discard / Stay.
- Diagnostics address `stepOccurrenceID` and parameter path, never row index.
- Validation requests are debounced and carry request identity, `editSeq` and pins; stale responses are discarded.
- The URL holds resource IDs and filters only.

**FRs:** FR-AUT-030, FR-AUT-031, FR-AUT-032, FR-AUT-033, FR-AUT-034, FR-AUT-035, FR-AUT-036, FR-AUT-037, FR-AUT-038, FR-AUT-039
**Screens:** N25 StaleRevisionConflictDialog and N24 (D02 `159:32045`).
**Design source:** board 3 §02–§03 `46:7847`.

### F-AUT-05 · Structured scenario builder

The builder renders fields from supported C01 schemas with availability, and selects exact immutable macro and asset revisions.

**Acceptance rules**
- Asset revisions are chosen by type and `contentDigest`, with no "latest" fallback.
- A bounded expansion preview with retained `sourceMap` and provenance inspection.
- An unavailable step is marked `UNAVAILABLE` inline with its reason and blocks compilation.
- Step order is keyboard-operable with announced position.

**FRs:** FR-AUT-040 (part), FR-AUT-041, FR-AUT-042
**Scope of partial requirements:** FR-AUT-040 — schema-driven fields with availability and reasons for the steps of a draft, and exact macro and asset revision selection.
**Screens:** R24 ScenarioBuilderPage `191:32450`; T01 `191:34359`.

### F-AUT-06 · Source import and evidence

An import is a durable job that records a versioned private snapshot; parsers extract structure without executing anything, and source evidence is reviewable.

**Acceptance rules**
- Families: user stories with acceptance criteria; product, API, architecture and test documentation; OpenAPI / Swagger; Postman collections with environments; `.feature` files; pasted non-sensitive wording.
- The job validates type and size and records snapshot hash and provenance, preserving sanitized wording.
- Intake enforces size, type and archive bounds and XXE, traversal, SSRF, zip-bomb, symlink and prohibited-reference controls; no imported script, Postman script, active content or source code is ever executed.
- Parsers extract OpenAPI operations and schemas, Postman requests and variables, and Gherkin parse with catalog mapping; Postman scripts and Gherkin that does not map to the catalog are listed as excluded in the parser report.
- Document extraction preserves story text, criteria and cited spans; requirements ground assertions, observed output never does.
- Evidence detail shows the read-only snapshot with safe preview, per-criterion mapping and coverage claim, parser report with exclusions, duplicates and snapshot identity; duplicates are linked, never dropped.
- Mechanically valid imports normalise deterministically without AI; narrative input requires composition and human mapping review.
- Grants are checked before retrieval and before any provider submission; source-grant revocation blocks new context and dependent approval while history stays.
- Source or meaning changes create a new C02 and require re-acceptance, recompile and approval; a stale import never overwrites.

**FRs:** FR-AUT-050 (part), FR-AUT-051, FR-AUT-052, FR-AUT-053 (part), FR-AUT-054, FR-AUT-055, FR-AUT-056, FR-AUT-057, FR-AUT-058, FR-AUT-059
**Scope of partial requirements:** FR-AUT-050 — the listed families. FR-AUT-053 — the listed extraction, with Postman scripts and Gherkin that does not map to the catalog reported as excluded and never executed.
**Screens:** R27 ImportsPage `148:3277`; N08 SourceEvidenceDetailPage `161:49761`.
**Design source:** board 5 `79:3652`.

### F-AUT-07 · Proposals queue and mapping review

`C08 ProposalEnvelope`s are typed, pinned proposals that a human edits, accepts or rejects; they carry no approval, Run or write authority.

**Acceptance rules**
- The queue lists criterion, source and citation, mapping state and the decision, states that confidence is not correctness, and offers no bulk approval, hidden mutation, launch or approve tool.
- The queue shows the assistance identity and its authority: propose and explain only.
- Review shows original with citation, proposed meaning step by step and an explicit meaning difference, with citation, assumption and uncovered item as separate facts.
- Acceptance states its exact scope, creates a C02 and never approves or runs.
- A proposal whose base revision is stale is rejected as stale.
- AI-proposed repair reads persisted evidence only and proposes a new visible C11 meaning; it never weakens an assertion, skips, swallows a failure, loops until green or reruns.

**FRs:** FR-AUT-070, FR-AUT-071, FR-AUT-072, FR-AUT-073, FR-AUT-074, FR-AUT-076 (part), FR-AUT-077
**Scope of partial requirements:** FR-AUT-076 — a stale proposal is rejected without weakening assertion intent.
**Screens:** R28 ProposalsQueuePage `148:3170`; R29 `148:3072`.
**Design source:** board 3 §05 `46:7847`; register `122:2` §11.

### F-AUT-08 · Revision diff

A two-revision diff shows preserved intent, the authored change and its author, what changed, and per-pin impact.

**Acceptance rules**
- It states explicitly that no prior approval carries forward.

**FRs:** FR-AUT-075
**Screens:** N10 RevisionDiffPage `161:50000`.

### F-AUT-09 · Pure C03 compilation and the validation bundle

C03 produces a `ValidationReport` and a sealed `CompiledBundle` with no side effects; the bundle surface shows every pin.

**Acceptance rules**
- The bundle seals input, dependency, compiler, catalog and runtime digests; diagnostics with source paths; expanded macros; generated feature bytes; binding and source map; sealed digest.
- No target, database, browser, device, JDBC or provider call; no probe or DNS; no wall-clock value in the hash; purity proven by fail-on-use factories and network-blocked tests.
- Static checks: types, references before use, prerequisites, resource states, availability, declared effects.
- Renderer / binding parity yields exactly one binding per occurrence; unsupported renderers block; adversarial escaping is tested; the serializer is deterministic, never browser `JSON.stringify`.
- Blocking findings include unknown types, unpublished or revoked references, unsupported schema dependencies, unavailable resolvers and unsupported language or profile bindings; static errors block approval and are never shown as target failures.
- The surface lists every pin, states that pins never upgrade, and that static validity is not network readiness.
- Each diagnostic area names its evidence, static boundary and resolution path.

**FRs:** FR-AUT-080, FR-AUT-081, FR-AUT-082, FR-AUT-083, FR-AUT-084, FR-AUT-085, FR-AUT-086, FR-AUT-087 (part), FR-AUT-088, FR-AUT-089, FR-AUT-090, FR-AUT-091
**Scope of partial requirements:** FR-AUT-087 — the blocking findings listed above.
**Screens:** N11 ValidationBundlePage `191:33527`.
**Design source:** board 2 §02 `46:7241`.

### F-AUT-10 · Project chat

Project-bounded, cited, propose-only assistance through a C10-authorised BE use case.

**Acceptance rules**
- Flow: message → C10-authorised BE AI use case → authorised minimal facts → bounded model adapter → cited answer or validated C08 / C11 proposal.
- Chat explains, clarifies, proposes and interprets persisted evidence; it has no approval, launch, database, shell, repository, handler-generation or integration-write tool; the word "Run" in chat never runs.
- Every claim carries a citation; advice is labelled advice.
- Proposals are not applied; read differences / accept / edit / reject; acceptance blocked while required arguments are absent.
- The surface shows tenant / project, allowed context, C12 model alias, provider and funding identity, quota reservation and egress scope.
- Input states no secrets may be sent; server-side redaction applies anyway.
- Provider or funding failure stops with no fallback and offers "continue manually".
- Conversations are access-controlled and store minimal redacted messages and provenance, never raw prompts.
- No shared client RAG, cross-tenant reuse or training reuse.

**FRs:** FR-AUT-100, FR-AUT-101, FR-AUT-102, FR-AUT-103, FR-AUT-104, FR-AUT-105, FR-AUT-106, FR-AUT-107 (part), FR-AUT-108
**Scope of partial requirements:** FR-AUT-107 — access control and minimal redacted storage with provenance.
**Screens:** N13 ProjectChatPage `161:50126`; N27 AIQuotaExceededPanel (D02 `159:32045`).
**Design source:** register `122:2` §7.

---

## APR · Approval

### F-APR-01 · Approval review and the C04 record

`C04 ApprovalRecord` captures a human decision on the exact bundle digest, selection, environment revision, every execution-affecting pin, policy revision and actor lineage. Approval has no launch side effect.

**Acceptance rules**
- The surface shows the full pin set, the preserved original wording, the accepted selection with provenance and every uncovered item excluded from the claim.
- It states that narrowing is not assertion removal and approval cannot convert partial evidence into proof of the whole.
- Eligibility and diagnostics appear before the approve action; no new exclusion at Run.
- It shows review authority: author and reviewer, independence of Author / Reviewer / `canRun`, tenant role separateness, the self-approval policy in force.
- Rationale and an explicit acknowledgement checkbox are captured.
- Approval never runs, reserves capacity, installs, allocates a device or tests the network.
- Membership or administrative status is never approval; macro or asset review is never scenario approval.
- A pending reachability assessment never blocks approval.

**FRs:** FR-APR-001, FR-APR-002, FR-APR-003, FR-APR-004, FR-APR-005, FR-APR-006, FR-APR-007, FR-APR-010, FR-APR-011, FR-APR-012, FR-APR-013
**Screens:** N12 ApprovalReviewPage `191:33673`; M06 `161:51536`.
**Design source:** board 3 §04–§05 `46:7847`; register `122:2` §8.

### F-APR-02 · Approval authority

Approval requires an explicit project Reviewer grant; the server rechecks authority at the moment of decision.

**Acceptance rules**
- Self-approval is allowed by default for solo operation; it is a deliberate C04 decision by a person holding Reviewer, visible and audited.
- Tenant policy `requireDifferentApprover` tightens it to a reviewer other than the author; this changes eligibility only and never removes the separate Run.
- At decision the server rechecks membership, grants, `permissionVersion`, tenant state and policy revision.

**FRs:** FR-APR-020, FR-APR-021, FR-APR-022, FR-APR-023, FR-SET-041
**Screens:** N12 `191:33673`; M06 `161:51536`; N06 TenantPolicyPage `161:49504`.
**Design source:** register `122:2` §3.

### F-APR-03 · Approval invalidation and launch blocking

Any execution-affecting change invalidates the affected approval; stale or revoked approvals block launch.

**Acceptance rules**
- The invalidating set includes scenario revision, environment digest, C01 catalog, macro revision or `expandedHash`, locator or schema revision, public trust revision, runtime profile and policy revision.
- Changed bytes require a new C02, recompile and fresh review; an existing review never approves changed bytes.
- Old approvals never revive revoked authority, revoked assets or deleted parents; restore never revives an approval.
- Current revocation, expiry, capability loss or probe staleness blocks a new effect despite a historical approval.

**FRs:** FR-APR-014 (part), FR-APR-030, FR-APR-031, FR-APR-032, FR-APR-033
**Scope of partial requirements:** FR-APR-014 — changed bytes require a new C02, recompile and fresh review.
**Screens:** N26 ApprovalInvalidatedDialog (D02 `159:32045`).
**Design source:** Alignment board `46:8271`.

### F-APR-04 · Approval audit and lineage

The approval decision is audited with actor, lineage, digest, policy revision and self-approval flag; the audit trail presents full lineage.

**Acceptance rules**
- Lineage order: source preserved → scope accepted (C11) → revision (C02) → bundle sealed (C03) → approval (C04) → run accepted (C05) → facts (C07) → delivery (C09), each with safe actor, decision or fact and detail link.

**FRs:** FR-APR-040, FR-APR-041
**Screens:** N18 AuditTrailPage `161:50756`.

---

## SET · Settings

### F-SET-01 · Settings write discipline

Every settings write confirms persistence before success and records actor, previous value, new value, rationale and resulting revision.

**Acceptance rules**
- No settings surface reveals, copies or browser-persists a stored secret; credentials change only through a write-only path.
- A settings change is never approval or a Run.

**FRs:** FR-SET-001, FR-SET-002, FR-SET-003, FR-SET-004
**Screens:** N06 `161:49504`.

### F-SET-02 · Personal settings

Profile, security and sessions, connection visibility and account privacy.

**Acceptance rules**
- Display name is editable; the verified email is shown with the statement that changing it is a separate verification flow.
- Interface language EN / ES / FR, stated not to change Gherkin dialect or expressions.
- Memberships are shown with the leave-versus-delete and last-Owner statements.
- Connection metadata is read-only, follows grants and never exposes credentials.

**FRs:** FR-SET-010, FR-SET-011, FR-SET-012, FR-SET-013, FR-SET-014
**Screens:** R12 PersonalSettingsPage `148:644`.

### F-SET-03 · Security settings

TOTP and recovery, and sessions.

**Acceptance rules**
- The TOTP area states when MFA is required, that required MFA cannot be disabled and that replacement never removes first.
- Only enrollment metadata is shown; entries for replace authenticator, regenerate recovery codes and controlled recovery.

**FRs:** FR-SET-020 (part), FR-SET-021, FR-SET-022
**Scope of partial requirements:** FR-SET-020 — the TOTP-and-recovery and Sessions areas.
**Screens:** R38 SecuritySettingsPage `148:3895`.

### F-SET-04 · Tenant members view

Members, pending invitations and tenant policy as separate views; the member list states the boundaries of tenant administration.

**Acceptance rules**
- Rows show member, tenant role, membership state including TOTP requirement, and the available action.
- Statements: last-Owner protection and recheck at transfer; membership is not project access (with a link to grant review); removing membership retains authored evidence; another person's password or MFA is never available to an administrator.

**FRs:** FR-SET-030, FR-SET-031, FR-SET-032, FR-SET-033, FR-SET-034
**Screens:** R13 TenantMembersPage `148:503`.
**Design source:** register `122:2` §3; specification 010 (27 October 2026), decision Q8.

### F-SET-05 · Tenant policy

A versioned tenant policy record, configured by Owner or Admin, that can only tighten platform boundaries.

**Acceptance rules**
- Human approval configuration `requireDifferentApprover` with its current value and the statement that solo ownership never approves automatically.
- Every change requires a rationale and records actor, previous and new value, rationale and policy revision.
- Identity requirements: TOTP mandatory for Owners and Admins; optional MFA for Members; local verified email and local accounts only, no SSO, SCIM or custom roles.
- Authority boundaries, configured retention and limits by reference (no default values), and the three-scope summary (tenant membership · project content · global identity).

**FRs:** FR-SET-040, FR-SET-042, FR-SET-043, FR-SET-044, FR-SET-045, FR-SET-046, FR-SET-047
**Screens:** N06 TenantPolicyPage `161:49504`.

### F-SET-06 · Jira Cloud delivery profile

`DeliveryProfileRevision`s configure delivery to the `JIRA_CLOUD_ISSUES` family; saving sends nothing.

**Acceptance rules**
- A revision carries tenant / project / connection, provider family and `adapterContractVersion`, remote project and issue type, field and entity mapping, payload template and capabilities, auth grant reference, redaction / attachment / retry / reconcile policy and per-capability evidence.
- Support is stated per capability, never as a blanket claim.
- Saving posts nothing, creates nothing, uploads nothing, opens no pull request and runs no test, and says so.
- Each capability is individually gated and shows its state and the effect when unverified; a capability without evidence is unavailable, not best-effort.
- Unmapped source entities surface as `UNMAPPED` and are never nearest-matched.
- Redaction precedes payload construction; attachments are sanitized artifacts only.
- A confirmed or unknown write is never retried; reconciliation uses an owned marker or receipt lookup where the provider proves dedupe, otherwise manual hold; dedupe-on-retry is off unless proven.
- Families not offered are listed disabled with their status and reason.
- New revisions never change delivered history.
- The delivery identity is separate from the source identity and never the console identity.

**FRs:** FR-SET-061, FR-SET-062 (part), FR-SET-063, FR-SET-064 (part), FR-SET-065, FR-SET-066, FR-SET-067 (part), FR-SET-068 (part), FR-SET-069, FR-SET-070
**Scope of partial requirements:** FR-SET-062 — `JIRA_CLOUD_ISSUES` with per-capability support statements. FR-SET-064 — per-capability gating display and the unavailable-when-unverified rule. FR-SET-067 — never retry confirmed or unknown writes, owned-marker reconciliation or manual hold, dedupe off unless proven. FR-SET-068 — static disabled listing with status and reason.
**Screens:** N40 DeliveryProfileEditorPage `191:34863`; R18 `191:32090`.
**Design source:** BE board `23:5587`; Failure board `3:2192`.

### F-SET-07 · AI policy and usage (C12)

AI policy and funding, and usage and reservations, under an effective policy `operator ∩ tenant ∩ project ∩ request`.

**Acceptance rules**
- Operator allocation is read-only: platform-funded status (default DISABLED), the `TenantAIEntitlement` or its absence, allocated usage, and that only the PlatformOperator raises the platform cap.
- Owner / Admin may lower limits or select approved BYOK, never raise funded or service caps.
- Capability allowlist per configuration alias, labelled illustrative.
- Explicit provider configuration, write-only credential reference, explicit funding mode with named credential owner; no silent provider, model or funding fallback.
- Two switches: block when provider or funding is unavailable; restrict egress to sanitized authorised project context.
- Statements: BYOK does not bypass service caps; AI quota is not run capacity; limits are not exact financial caps; no invoices.
- Usage shows estimate versus actual and reconciliation state; unknown usage is retained, never zeroed.
- Budget is reserved atomically before external I/O, the call runs outside the transaction, and an ambiguous timeout never releases a reservation before reconciliation.
- Context, fan-out, output and time are bounded.
- Model, provider or funding changes are explicit, authorised and audited.
- "Continue manually" is offered; manual authoring, static compile and base report stay available.
- The surface is not a SuperAdmin or global-content portal.

**FRs:** FR-SET-080 (part), FR-SET-081, FR-SET-082, FR-SET-083, FR-SET-084, FR-SET-085, FR-SET-086, FR-SET-087, FR-SET-088, FR-SET-089, FR-SET-090, FR-SET-091, FR-SET-092, FR-SET-093, FR-SET-094, FR-OPS-041
**Scope of partial requirements:** FR-SET-080 — the Policy-and-funding and Usage-and-reservations views.
**Screens:** N14 AIPolicyUsagePage `191:33803`; N27 (D02 `159:32045`).
**Design source:** BE board `23:5587`; register `122:2` §15.

---

## OPS · Operator records

The platform operator is an internal, out-of-band principal. ASpectra reads the operator's versioned records from a dedicated record store, validates them, and projects their effect to tenants read-only.

### F-OPS-01 · Operator record store (read side)

`ManagedNetworkProfile`, `ExecutionCapacityPolicy`, `TenantAIEntitlement`, `ProbePolicy`, `ScanPolicy` and `PublicDocumentLinksRevision` are immutable versioned records with id, reason, actor and timestamp (register `122:2` §15). Every effect references the exact record version it was admitted under.

**Acceptance rules**
- Records are validated at startup and again at every effect boundary.
- An absent or invalid record is `UNCONFIGURED` and blocks the dependent effect; no default, invented value or fallback profile.
- `ManagedNetworkProfile` carries `profileId`, `profileVersion`, `networkVersion`, the actual egress CIDR or address identity, execution region and capability set; until every field carries a real value it is `UNCONFIGURED`; a published egress identity is never a placeholder.
- `ExecutionCapacityPolicy` carries global and per-tenant `maxActiveRuns`, `maxScenarioWorkerSlots`, `maxQueuedTickets`, `maxQueueWait`; absent capacity configuration refuses work.
- `TenantAIEntitlement` carries window, caps, funding source, approved provider capabilities, version, reason and status; default DISABLED with no allocation; no automatic funding fallback and no payment processor; usage is reserved atomically and reconciled.
- `ProbePolicy`, `ScanPolicy` and `PublicDocumentLinksRevision` follow the same discipline; their consumers are F-RUN-06, F-CAT-05 / F-CAT-06 and F-IDN-19.
- The operator authority adds no contract number, no tenant role and no boundary change.

**FRs:** FR-OPS-003, FR-OPS-010 (part), FR-OPS-011, FR-OPS-012, FR-OPS-013, FR-OPS-020, FR-OPS-021, FR-OPS-022, FR-OPS-030 (part), FR-OPS-032, FR-OPS-040, FR-OPS-042, FR-OPS-043
**Scope of partial requirements:** FR-OPS-010 — the record discipline for the six records named above. FR-OPS-030 — the four ceilings named above.
**Design source:** register `122:2` §15; BE board `23:5587`.

### F-OPS-02 · Operator authority boundary

`PlatformOperator` is a platform principal outside every tenant membership and project grant, and certain actions are structurally impossible for it.

**Acceptance rules**
- It is never exposed to customers and never a tenant role; no tenant API grants, delegates or escalates it; it writes only through the record discipline.
- It has no unrestricted access to customer content, source snapshots, artifacts or prompts.
- It can never: accept a mapping, approve or launch; mutate C02, a sealed C03, a C07 fact or a Core verdict; read customer content as a side effect of configuration; grant itself membership or bypass RLS; raise a ceiling retroactively for refused work.

**FRs:** FR-OPS-001 (part), FR-OPS-002 (part), FR-OPS-014 (part), FR-OPS-050
**Scope of partial requirements:** FR-OPS-001 — never customer-exposed, never a tenant role, writes only through record discipline. FR-OPS-002 — outside membership and grants, never grantable, delegable or escalatable from a tenant. FR-OPS-014 — no unrestricted access to customer content.
**Design source:** register `122:2` §15, §13.2.

### F-OPS-03 · Tenant read-only projection

A tenant sees the effect of operator records and the authority responsible, read-only, and only its own.

**Acceptance rules**
- Never another tenant's allocation; never a way to raise its own ceiling.
- `ExecutionCapacityPolicy` and C12 AI budgets are independent in the data model and every surface.
- An execution region is never presented as data residency; no inferred region and no multi-region routing; region policy selects only published compatible profiles.
- An SUT 401 proves network reachability only, never application correctness or a console sign-in failure.

**FRs:** FR-OPS-015, FR-OPS-023 (part), FR-OPS-024, FR-OPS-026, FR-OPS-031 (part), FR-OPS-033
**Scope of partial requirements:** FR-OPS-023 — region is not residency; no inferred region; no invented multi-region routing. FR-OPS-031 — a tenant or project can never raise a ceiling.
**Screens:** N14 `191:33803`; N30 `191:34696`; N31 `191:34833`; N33 `191:34868`; R30 `191:32908`.
**Design source:** register `122:2` §15.

---

## RUN · Execution, evidence and delivery

### F-RUN-01 · Run confirmation

Run is a separate explicit human request reachable only from a valid C04; it requests fair-queue admission, not an immediate start.

**Acceptance rules**
- The surface shows the exact pins that would execute; pins never upgrade; every reference is an exact pin; an unknown or unavailable resolver blocks.
- An admission prerequisite checklist states, per prerequisite, current state, what is rechecked at admission and one next step: exact C04 and sealed bytes; managed profile and fresh assessment; asset, macro and trust eligibility; `ExecutionCapacityPolicy`.
- Any unmet prerequisite disables Run with the reason stated once, never as a test failure.
- "Check from managed profile" is reachable and never submits a Run.
- Confirmation is an affirmative acknowledgement of this exact bundle with no runtime override.
- Acceptance reports 202 accepted, not started, with no verdict yet; the idempotency key is displayed and retained.
- Accepted coverage limits stay visible; no new exclusion at Run; a changed selection returns to compile and review.
- Every launch comes from an explicit human C05.

**FRs:** FR-RUN-001, FR-RUN-002 (part), FR-RUN-003, FR-RUN-004, FR-RUN-005, FR-RUN-006, FR-RUN-007, FR-RUN-008, FR-RUN-009, FR-RUN-010 (part)
**Scope of partial requirements:** FR-RUN-002 — pins for selection, environment, C01, bundle, `expandedHash`, locator / schema / trust digests and policy / profile references. FR-RUN-010 — the explicit human C05 as the only launch path.
**Screens:** R30 RunConfirmationPage `191:32908`.
**Design source:** FE board `46:7847`; BE board `23:5587`.

### F-RUN-02 · C05 acceptance transaction

Acceptance inserts the human intent, an `AdmissionTicket` in `QUEUED`, the Run / attempt and the outbox entry in one transaction.

**Acceptance rules**
- Idempotent on key and body hash: same key and body returns the same ticket; a changed body conflicts.
- A full queue rejects before acceptance.
- Acceptance requires a fresh matching reachability assessment; without one C05 is refused and no ticket exists.

**FRs:** FR-RUN-000 (part), FR-RUN-007, FR-RUN-008
**Scope of partial requirements:** FR-RUN-000 — the `C05 LaunchCommand` row.
**Design source:** BE board `23:5587` ("Insert human intent + AdmissionTicket(QUEUED) + Run/attempt + outbox in one transaction").

### F-RUN-03 · Fair admission scheduler

A persisted round-robin of eligible tenants per compatible pool, FIFO within a tenant, admits tickets to worker slots.

**Acceptance rules**
- Revalidation of grants, approval, environment, assets, trust and assessment precedes atomic reservation of run and worker slots with deterministic lock order, attempt and fence.
- Lifecycle `QUEUED → ADMITTING → RUNNING → COMPLETED`; `CANCEL_REQUESTED` may race completion; `ADMITTING → QUEUED` only with proof of no remote submit and a valid deadline, under the same accepted intent.
- Queued cancel / reject / expiry is NOT EXECUTED with no verdict; possible execution with a lost ACK is INDETERMINATE, resolved by lookup and reconciliation of the same submission.
- `SKIP LOCKED` alone is not fairness.
- A `projectionVersion` prevents stale projections from regressing state.
- Suite fan-out is bounded by worker slots.

**Design rules** (BE board `23:5587`):
- `AdmissionTicket` fields: tenant, project, actor, approval, bundle, environment, profile; demand; `policyVersion`; `serverFIFOSequence`; deadline; `holdReason`; `assessmentRef`; lease / attempt / fence; `projectionVersion`.
- One blocked head never blocks other tenants or pools; the global cursor holds tenant IDs and narrow operational metadata only.
- Claims are released idempotently only after known cleanup; lost ACK or expiry is not proof of stop: reconcile, quarantine uncertain resources, rebuild counters.
- Scheduler and retry jobs act under the accepted human ticket; they never mint a Run or bypass RLS; there is no customer launch API, cron or CI trigger.

**FRs:** FR-RUN-000 (part), FR-RUN-024, FR-RUN-025, FR-RUN-027, FR-OPS-034 (part)
**Scope of partial requirements:** FR-RUN-000 — the `AdmissionTicket` + fair scheduler row with run and worker-slot reservation, the lifecycle row, the NOT EXECUTED / INDETERMINATE row and the `C06 RunnerCapabilities` row with profile, build, network version, capabilities, asset eligibility, leases, fence and `assessmentRef`. FR-OPS-034 — fan-out bounded by worker slots.
**Design source:** BE board transition table `23:5587`.

### F-RUN-04 · Execution queue and queued-ticket detail

The queue shows only the viewer's own tickets, grouped Queued and held · Admitting · Active · Ticket history.

**Acceptance rules**
- Filters: project scope, compatible pool, state; rows carry ticket and scope, state and `holdReason`, `queueDeadline` and policy version, and one safe action.
- No global rank, no other tenant's identity or demand, no ETA — on any surface.
- An unconfigured deadline or ceiling is stated as such, never shown as an invented number.
- Fair admission is explained in product terms, and queue-to-worker transition fulfils the original human request.
- Ticket detail shows the original intent (ticket, idempotency key, requester, acceptance, approval lineage), the unchanged pins, why it waits (`holdReason`, policy, FIFO position within own scope, deadline, `projectionVersion`) and the lifecycle legend.
- The legend maps `QUEUED · eligible`, `QUEUED · held`, `ADMITTING`, `CANCEL_REQUESTED`, `CANCELED / REJECTED / EXPIRED` = NOT EXECUTED and `INDETERMINATE` to meaning, verdict existence and allowed action; lease expiry is never proof of stop.
- An assessment that expires while queued holds the ticket; recheck only within the original consent, intent and deadline.
- Cancelling applies to the user's own queued ticket, is rechecked by the backend and may race completion; it is never rollback.

**Design rule:** queue-to-worker movement never changes a pin; a changed pin supersedes the ticket and returns it to compile and review (N32 `191:34828`).

**FRs:** FR-RUN-020, FR-RUN-021, FR-RUN-022, FR-RUN-023, FR-RUN-026, FR-RUN-028, FR-RUN-029
**Screens:** N30 ExecutionQueuePage `191:34696`; N32 QueuedRunDetailPage `191:34828`; M09 `191:34883`; M25 `435:4294`; M26 `435:4351`.
**Design source:** FE board `46:7847`.

### F-RUN-05 · Admission refusal

A refused C05 states one reason once and that nothing was created or changed.

**Acceptance rules**
- No ticket was created, nothing is queued or running, no verdict exists, approval, environment and pins are untouched.
- The surface names the refusing gate, whether a ticket exists and who can unblock, for: queue full; managed profile `UNCONFIGURED`; `ExecutionCapacityPolicy` unconfigured; assessment absent; asset, macro or trust not eligible; grant or approval revoked; region policy unsatisfied.
- Refused intents and cancelled tickets read NOT EXECUTED, are excluded from pass rates, and Run again is a new human C05.
- The tenant sees its own capacity policy effect read-only: policy version, own ceilings, global-cap authority and AI-quota independence.

**FRs:** FR-RUN-030, FR-RUN-031 (part), FR-RUN-032, FR-RUN-033 (part)
**Scope of partial requirements:** FR-RUN-031 — the gates listed above. FR-RUN-033 — policy version, own ceilings, global-cap authority and AI-quota independence.
**Screens:** N33 AdmissionBlockedPage `191:34868`; M27 `435:4392`.

### F-RUN-06 · Reachability assessment

An explicitly consented, bounded `ProbeJob` answers reachability for one pinned `envRevision`, one tested path, at one time.

**Acceptance rules**
- Where the managed profile is `UNCONFIGURED`, no check can be consented to; the unpublished descriptor fields are shown, with the platform operator as their authority; nothing is inferred.
- The record shows pinned environment, probe policy version, consent, `assessedAt`, expiry and coverage (`UNCHECKED` when absent).
- The full stage list is shown before any packet, and consent covers exactly it: DNS of named aliases from the published egress; TCP to configured ports; TLS against the pinned trust revision; HTTP on the explicitly configured safe path — no guessed endpoint, secret or test step.
- A check never installs, submits a Run or produces a verdict.
- Partial coverage is never "reachable"; "reachable" is never "connected" or "healthy"; HTTP 401 proves network reachability only.
- An absent assessment blocks Run only; an expired one holds a queued ticket.

**FRs:** FR-RUN-040, FR-RUN-041, FR-RUN-042, FR-RUN-043, FR-RUN-044, FR-RUN-045, FR-RUN-046, FR-OPS-025 (part)
**Scope of partial requirements:** FR-OPS-025 — the probe's behavioural bounds: explicitly configured protocol and path, bounded consent, no secret exposure, no test steps, no install side effects, no guessed path.
**Screens:** N31 EnvironmentReachabilityPage `191:34833`.
**Design source:** BE board `23:5587`.

### F-RUN-07 · Managed protocol execution (HTTP, Web, Mobile, DB)

The runner executes an admitted bundle for HTTP, Web, Mobile and DB protocols with one logical owner per scenario resource and an immutable C07 journal.

**Acceptance rules**
- The runner verifies the envelope and persists acceptance before acknowledging.
- Per-scenario browser context, HTTP state and JDBC connection; Playwright Java stays on its owning thread; LIFO cleanup.
- Mobile: one scenario-isolated Appium process on an ASpectra-managed device host (macOS hosts for iOS), supervised by the runner; the approved app digest is installed only under an admitted C05 + C06 run onto an exclusively leased device; an uncertain uninstall, reset or cleanup quarantines the device instead of returning it as reusable (register `122:2` §13.8).
- One attempt per acceptance; uncertain cleanup quarantines the resource.
- C07: immutable Core verdict only after execution; separate admission / infrastructure / cleanup / evidence axes; snapshot watermark and replay.
- C09 `DeliveryIntent` items follow F-RUN-13.

**Design rules:**
- Runner egress never turns customer URLs into access to BE or internal metadata networks (register `122:2` §8).
- Fencing rejects late submissions, results and delivery callbacks; tenant suspension freezes new effects with bounded cancel (Failure board `3:2192`).

**FRs:** FR-RUN-000 (part), FR-CAT-075
**Scope of partial requirements:** FR-RUN-000 — the `ReachabilityAssessment`, `C07` and `C09` rows, and the Core runtime resource-ownership row for HTTP, Web, Mobile and DB resources.
**Design source:** Core board `46:7241`; Failure board `3:2192`.

### F-RUN-08 · Live monitoring

A non-blocking, labelled projection of an accepted run with separate axes.

**Acceptance rules**
- Navigation, tenant switch, session expiry and reauthentication never cancel, duplicate or replay a job; reauthentication grants inspection only.
- Lifecycle, verdict, evidence, delivery and AI analysis are separate axes; while running the verdict reads "no final verdict · last confirmed snapshot".
- On stream loss: the stream disconnected, not the run; facts may be stale; recovery retrieves an authorised snapshot and replays from the watermark — never Run.
- Admission context stays visible: ticket and intent, queue → admission → worker with unchanged pins, policy and pool, retained admission snapshot, link to queue history.
- Step rows distinguish known fact, in-progress step with possibly stale snapshot, and pending dependency; unevaluated is never success.
- A target 401 is an SUT response; stream loss is never a failed assertion.
- Live cancellation is separate from lifecycle and verdict, awaits the authoritative terminal state, and routes `INDETERMINATE` to reconciliation.

**FRs:** FR-RUN-050, FR-RUN-051, FR-RUN-052, FR-RUN-053, FR-RUN-054, FR-RUN-055, FR-RUN-056
**Screens:** R31 ExecutionLivePage `191:33061`; M07 `191:34119`.
**Design source:** FE board `46:7847`; Alignment board `46:8271`.

### F-RUN-09 · Results and evidence

The immutable C07 outcome for a run and attempt, as four separate statements: lifecycle, Core verdict, evidence completeness, C09 delivery.

**Acceptance rules**
- Facts, evidence, advice and delivery are separately navigable; unretrievable evidence never rewrites the verdict.
- Step rows carry recorded fact, expected versus actual, evidence reference and context; "not evaluated — dependency not met" is never success.
- AI interpretation is optional, absent unless requested and labelled advice; without a `TenantAIEntitlement` the surface states platform-funded AI is DISABLED and offers continuing with the base report; no fallback.
- Pins are reproduced as at Run and labelled dependency metadata.
- Accepted coverage boundaries are restated; uncovered criteria remain named.
- NOT EXECUTED tickets belong to ticket history.
- A new Run needs a separate C05, current `canRun`, a valid C04, unchanged pins and current prerequisites.

**Design rule:** the base report is downloadable from results, artifact and delivery surfaces ("Download base report", R32, N16, N41).

**FRs:** FR-RUN-060, FR-RUN-061, FR-RUN-062, FR-RUN-063 (part), FR-RUN-064, FR-RUN-065, FR-RUN-066, FR-RUN-067
**Scope of partial requirements:** FR-RUN-063 — optional, labelled, absent by default, the DISABLED state and the base-report continuation without fallback.
**Screens:** R32 ExecutionResultsPage `191:33192`; M08 `191:34221`.

### F-RUN-10 · Artifacts and degraded retrieval

Artifacts have an identity and are previewed and downloaded safely; retrieval failure is never a verdict.

**Acceptance rules**
- Identity: run and attempt, step, `environmentId` / `envRevision`, media digest, redaction record, availability under project authorisation.
- Preview escapes content as text; customer HTML and SVG never render inline; active media is download-only after authorisation; every download rechecks permission; no permanent public URL.
- An absent artifact is shown as absent with the base report still available.
- A retrieval retry never executes a test or creates an attempt; the degraded panel states this is not a failed-run verdict, implies no fixed retry count and that lifecycle is unknown until an authoritative snapshot arrives.

**FRs:** FR-RUN-068 (part), FR-RUN-069, FR-RUN-070, FR-RUN-071
**Scope of partial requirements:** FR-RUN-068 — the identity fields and availability under project authorisation.
**Screens:** N16 ArtifactDetailPage `161:50521`; R37 ResultsFetchDegradedPanel `148:3890`.

### F-RUN-11 · Run history and lineage

History lists explicit intents, accepted runs and attempts in the current project.

**Acceptance rules**
- Search by run, intent or attempt identity; filter by lifecycle and environment revision; views Runs · Pending acceptance (acceptance-unknown intents) · Delivery jobs; the surface states there is no implicit schedule.
- Columns keep lineage, lifecycle and verdict, evidence completeness and delivery (in C09 states) separate, including "acceptance unknown / no run" and "canceled / incomplete, known facts retained".
- Lineage chains author revision → bundle → approval → Run intent → run → attempt → environment revision.
- Retrying retrieval or delivery never creates an execution; a rerun is a new explicit intent with a valid approval.

**FRs:** FR-RUN-080 (part), FR-RUN-081 (part), FR-RUN-082, FR-RUN-083
**Scope of partial requirements:** FR-RUN-080 — the listed search, filters and views, with Pending acceptance holding acceptance-unknown intents. FR-RUN-081 — the listed columns with delivery in C09 states.
**Screens:** R33 ExecutionHistoryPage `148:3378`.

### F-RUN-12 · Acceptance-unknown lookup

When the Run response did not arrive, the user looks up the original intent instead of resubmitting.

**Acceptance rules**
- The surface states acceptance is unknown and execution may already be accepted, keeps and shows the original intent key, offers an authoritative status lookup, and never resubmits automatically.
- No exactly-once external-effect guarantee is implied.

**FRs:** FR-RUN-084, FR-RUN-085
**Screens:** N21 RunAcceptanceUnknownPage `159:31909`.
**Design source:** Failure board `3:2192`.

### F-RUN-13 · Delivery jobs and item reconciliation (`JIRA_CLOUD_ISSUES`)

A `C09 DeliveryIntent` with a frozen itemset is delivered per item and reconciled per item; delivery is its own outcome.

**Acceptance rules**
- The Core verdict and base report are unaffected by delivery.
- The job view shows provider family, the `DeliveryProfileRevision` it was published under, remote project, issue type and entity mapping, and that contract tests and dedupe or permission proof are activation gates.
- Items carry entity, mapping and payload, receipt and state, and one next safe step, across `CONFIRMED` · `UNMAPPED` · `UNKNOWN` · `RETRYABLE_FAILURE` · `PERMANENT_FAILURE`.
- A parent `PARTIAL` reads "not delivered" until every required item has an authoritative outcome; one confirmed item proves nothing about another.
- Reconciliation shows the frozen intent: intent and correlation, profile revision, frozen itemset with planned and excluded counts, frozen payload hashes, authority rechecked at action; the itemset never grows.
- Per item the surface states what a retry would do: "nothing" for confirmed, unmapped and permanently failed, "blocked" for unknown.
- `UNKNOWN` items are held where the provider cannot prove dedupe; a remote ID alone is not proof of write; silence is not proof of absence.
- Retry scope is stated before confirmation, with identical frozen bytes under the recorded redaction policy and the parent staying `PARTIAL`.
- Delivery actions never execute a scenario, change a verdict or weaken an assertion; no Run, pull request, Xray `TestExecution` or `TestRun` is available from delivery.

**FRs:** FR-RUN-090, FR-RUN-091, FR-RUN-092 (part), FR-RUN-093, FR-RUN-094, FR-RUN-095, FR-RUN-096, FR-RUN-097, FR-RUN-098
**Scope of partial requirements:** FR-RUN-092 — items of the comment, attachment and read / backlink capabilities.
**Screens:** N17 DeliveryJobsPage `191:33975`; N41 PartialDeliveryReconcilePage `191:34873`; M29 `435:4416`; M30 `435:4460`.
**Design source:** BE board `23:5587`; Failure board `3:2192`.

---

# Part III · Traceability

## III.1 Requirement → feature

| Requirement | Feature(s) |
|---|---|
| FR-X-001 | F-X-01 |
| FR-X-002 | F-X-01 |
| FR-X-003 | F-X-01 |
| FR-X-004 | F-X-01 |
| FR-X-005 | F-X-01 |
| FR-X-006 | F-X-01 |
| FR-X-010 | F-X-01 |
| FR-X-011 | F-X-01 |
| FR-X-012 | F-X-01 |
| FR-X-013 | F-X-01 |
| FR-X-014 | F-X-01 |
| FR-X-015 | F-X-01 |
| FR-X-020 | F-X-02 |
| FR-X-021 | F-X-02 |
| FR-X-022 | F-X-02 |
| FR-X-023 | F-X-02 |
| FR-X-030 | F-X-03 |
| FR-X-031 | F-X-03 |
| FR-X-032 | F-X-03 |
| FR-X-033 | F-X-03 |
| FR-X-034 | F-X-03 |
| FR-X-035 | F-X-03 |
| FR-X-040 | F-X-04 |
| FR-X-041 | F-X-04 |
| FR-X-042 | F-X-04 |
| FR-X-043 | F-X-04 |
| FR-X-044 | F-X-05 (part) |
| FR-X-045 | F-X-06 |
| FR-X-046 | F-X-06 |
| FR-X-050 | F-X-07 (part) |
| FR-X-051 | F-X-07 |
| FR-X-052 | F-X-07 |
| FR-X-053 | F-X-04 |
| FR-X-054 | F-X-04 |
| FR-X-055 | F-X-04 |
| FR-IDN-001 | F-IDN-01 |
| FR-IDN-002 | F-IDN-01 |
| FR-IDN-003 | F-IDN-01 |
| FR-IDN-010 | F-IDN-01 |
| FR-IDN-011 | F-IDN-01 |
| FR-IDN-012 | F-IDN-02 |
| FR-IDN-013 | F-IDN-02 |
| FR-IDN-014 | F-IDN-02 |
| FR-IDN-015 | F-IDN-02 |
| FR-IDN-020 | F-IDN-03 |
| FR-IDN-021 | F-IDN-03 |
| FR-IDN-022 | F-IDN-03 |
| FR-IDN-023 | F-IDN-04 (part) |
| FR-IDN-024 | F-IDN-04 |
| FR-IDN-025 | F-IDN-04 |
| FR-IDN-026 | F-IDN-04 |
| FR-IDN-027 | F-IDN-04 |
| FR-IDN-028 | F-IDN-05 |
| FR-IDN-029 | F-IDN-05 |
| FR-IDN-030 | F-IDN-04 |
| FR-IDN-040 | F-IDN-06 |
| FR-IDN-041 | F-IDN-06 |
| FR-IDN-042 | F-IDN-06 |
| FR-IDN-043 | F-IDN-06 |
| FR-IDN-044 | F-IDN-06 |
| FR-IDN-045 | F-IDN-06 |
| FR-IDN-046 | F-IDN-07 |
| FR-IDN-047 | F-IDN-07 |
| FR-IDN-048 | F-IDN-07 |
| FR-IDN-049 | F-IDN-07 |
| FR-IDN-060 | F-IDN-08 |
| FR-IDN-061 | F-IDN-08 |
| FR-IDN-062 | F-IDN-08 |
| FR-IDN-063 | F-IDN-08 |
| FR-IDN-064 | F-IDN-08 |
| FR-IDN-065 | F-IDN-08 |
| FR-IDN-066 | F-IDN-08 |
| FR-IDN-067 | F-IDN-08 |
| FR-IDN-068 | F-IDN-09 |
| FR-IDN-069 | F-IDN-09 |
| FR-IDN-080 | F-IDN-10 |
| FR-IDN-081 | F-IDN-10 |
| FR-IDN-082 | F-IDN-10 |
| FR-IDN-083 | F-IDN-10 |
| FR-IDN-084 | F-IDN-10 |
| FR-IDN-085 | F-IDN-10 |
| FR-IDN-090 | F-IDN-11 |
| FR-IDN-091 | F-IDN-11 |
| FR-IDN-092 | F-IDN-11 |
| FR-IDN-093 | F-IDN-11 |
| FR-IDN-094 | F-IDN-11 |
| FR-IDN-095 | F-IDN-12 |
| FR-IDN-096 | F-IDN-14 |
| FR-IDN-097 | F-IDN-12 |
| FR-IDN-098 | F-IDN-13 |
| FR-IDN-099 | F-IDN-11 |
| FR-IDN-110 | F-IDN-15 |
| FR-IDN-111 | F-IDN-15 |
| FR-IDN-112 | F-IDN-15 |
| FR-IDN-113 | F-IDN-14 |
| FR-IDN-114 | F-IDN-14 |
| FR-IDN-115 | F-IDN-14 |
| FR-IDN-116 | F-IDN-16 |
| FR-IDN-120 | F-IDN-17 |
| FR-IDN-121 | F-IDN-17 |
| FR-IDN-122 | F-IDN-17 (part) |
| FR-IDN-123 | F-IDN-17 |
| FR-IDN-130 | F-IDN-18 |
| FR-IDN-131 | F-IDN-18 |
| FR-IDN-132 | F-IDN-18 |
| FR-IDN-140 | F-IDN-19 |
| FR-IDN-141 | F-IDN-19 |
| FR-IDN-142 | F-IDN-19 |
| FR-IDN-143 | F-IDN-19 |
| FR-IDN-144 | F-IDN-19 |
| FR-IDN-145 | F-IDN-19 |
| FR-PRJ-001 | F-PRJ-01 |
| FR-PRJ-002 | F-PRJ-01 |
| FR-PRJ-003 | F-PRJ-01 |
| FR-PRJ-004 | F-PRJ-01 (part) |
| FR-PRJ-010 | F-PRJ-02 |
| FR-PRJ-011 | F-PRJ-02 |
| FR-PRJ-012 | F-PRJ-02 |
| FR-PRJ-013 | F-PRJ-03 |
| FR-PRJ-014 | F-PRJ-02 |
| FR-PRJ-020 | F-PRJ-04 |
| FR-PRJ-021 | F-PRJ-04 (part) |
| FR-PRJ-022 | F-PRJ-04 |
| FR-PRJ-023 | F-PRJ-04 (part) |
| FR-PRJ-024 | F-PRJ-04 |
| FR-PRJ-025 | F-PRJ-04 |
| FR-PRJ-030 | F-PRJ-05 |
| FR-PRJ-031 | F-PRJ-05 |
| FR-PRJ-032 | F-PRJ-05 |
| FR-PRJ-033 | F-PRJ-06 |
| FR-PRJ-034 | F-PRJ-06 |
| FR-PRJ-040 | F-PRJ-07 (part) |
| FR-PRJ-041 | F-PRJ-07 |
| FR-PRJ-042 | F-PRJ-07 |
| FR-PRJ-043 | F-PRJ-07 |
| FR-PRJ-044 | F-PRJ-07 |
| FR-PRJ-045 | F-PRJ-07 |
| FR-PRJ-046 | F-PRJ-07 |
| FR-PRJ-047 | F-PRJ-07 |
| FR-PRJ-050 | F-PRJ-01 |
| FR-PRJ-051 | F-PRJ-01 |
| FR-PRJ-060 | F-PRJ-08 |
| FR-PRJ-061 | F-PRJ-08 |
| FR-PRJ-062 | F-PRJ-08 (part) |
| FR-PRJ-063 | F-PRJ-08 (part) |
| FR-PRJ-064 | F-PRJ-08 |
| FR-PRJ-065 | F-PRJ-08 |
| FR-PRJ-066 | F-PRJ-08 |
| FR-PRJ-067 | F-PRJ-08 |
| FR-PRJ-068 | F-PRJ-08 |
| FR-PRJ-069 | F-PRJ-08 |
| FR-PRJ-070 | F-PRJ-08 |
| FR-PRJ-071 | F-PRJ-08 |
| FR-PRJ-072 | F-PRJ-08 |
| FR-PRJ-073 | F-CAT-06 |
| FR-PRJ-074 | F-PRJ-08 |
| FR-PRJ-075 | F-PRJ-08 |
| FR-PRJ-080 | F-PRJ-09 |
| FR-PRJ-081 | F-PRJ-09 (part) |
| FR-PRJ-082 | F-PRJ-09 |
| FR-PRJ-083 | F-PRJ-09 |
| FR-PRJ-084 | F-PRJ-09 |
| FR-PRJ-090 | F-PRJ-10 |
| FR-PRJ-091 | F-PRJ-10 (part) |
| FR-PRJ-092 | F-PRJ-10 |
| FR-PRJ-093 | F-PRJ-10 (part) |
| FR-PRJ-094 | F-PRJ-10 |
| FR-PRJ-100 | F-PRJ-11 |
| FR-PRJ-101 | F-PRJ-11 |
| FR-PRJ-102 | F-PRJ-11 |
| FR-PRJ-103 | F-PRJ-11 |
| FR-PRJ-104 | F-PRJ-11 |
| FR-PRJ-110 | F-PRJ-12 |
| FR-PRJ-111 | F-PRJ-12 |
| FR-PRJ-112 | F-PRJ-12 |
| FR-PRJ-113 | F-PRJ-12 |
| FR-PRJ-114 | F-PRJ-12 |
| FR-PRJ-115 | F-PRJ-12 |
| FR-PRJ-116 | F-PRJ-12 |
| FR-CAT-001 | F-CAT-01 |
| FR-CAT-002 | F-CAT-01 |
| FR-CAT-003 | F-CAT-01 |
| FR-CAT-004 | F-CAT-01 |
| FR-CAT-005 | F-CAT-01 |
| FR-CAT-006 | F-CAT-01 |
| FR-CAT-007 | F-CAT-01 |
| FR-CAT-010 | F-CAT-02 |
| FR-CAT-011 | F-CAT-02 |
| FR-CAT-012 | F-CAT-02 |
| FR-CAT-013 | F-CAT-02 |
| FR-CAT-014 | F-CAT-02 |
| FR-CAT-015 | F-CAT-02 |
| FR-CAT-016 | F-CAT-02 |
| FR-CAT-020 | F-CAT-03 |
| FR-CAT-021 | F-CAT-03 |
| FR-CAT-022 | F-CAT-03 |
| FR-CAT-023 | F-CAT-03 |
| FR-CAT-030 | F-CAT-04 |
| FR-CAT-031 | F-CAT-04 |
| FR-CAT-032 | F-CAT-04 (part) |
| FR-CAT-033 | F-CAT-04 |
| FR-CAT-034 | F-CAT-04 |
| FR-CAT-035 | F-CAT-04 |
| FR-CAT-036 | F-CAT-04 |
| FR-CAT-037 | F-CAT-04 |
| FR-CAT-038 | F-CAT-04 |
| FR-CAT-039 | F-CAT-04 |
| FR-CAT-040 | F-CAT-04 |
| FR-CAT-041 | F-CAT-04 |
| FR-CAT-042 | F-CAT-04 |
| FR-CAT-050 | F-CAT-05 |
| FR-CAT-051 | F-CAT-05 |
| FR-CAT-052 | F-CAT-05 |
| FR-CAT-053 | F-CAT-05 |
| FR-CAT-054 | F-CAT-05 |
| FR-CAT-055 | F-CAT-05 |
| FR-CAT-056 | F-CAT-05 |
| FR-CAT-057 | F-CAT-05 |
| FR-CAT-058 | F-CAT-05 |
| FR-CAT-059 | F-CAT-05 |
| FR-CAT-060 | F-CAT-05 |
| FR-CAT-061 | F-CAT-05 |
| FR-CAT-062 | F-CAT-05 |
| FR-CAT-063 | F-CAT-05 |
| FR-CAT-070 | F-CAT-06 |
| FR-CAT-071 | F-CAT-06 |
| FR-CAT-072 | F-CAT-06 |
| FR-CAT-073 | F-CAT-06 |
| FR-CAT-074 | F-CAT-06 |
| FR-CAT-075 | F-RUN-07 |
| FR-CAT-076 | F-CAT-06 |
| FR-CAT-077 | F-PRJ-09 |
| FR-CAT-078 | F-PRJ-09 |
| FR-CAT-079 | F-PRJ-09 |
| FR-CAT-080 | F-PRJ-09 |
| FR-CAT-081 | F-PRJ-09 |
| FR-CAT-082 | F-PRJ-09 |
| FR-AUT-001 | F-AUT-01 |
| FR-AUT-002 | F-AUT-01 |
| FR-AUT-003 | F-AUT-01 |
| FR-AUT-004 | F-AUT-01 |
| FR-AUT-010 | F-AUT-02 |
| FR-AUT-011 | F-AUT-02 |
| FR-AUT-012 | F-AUT-02 |
| FR-AUT-013 | F-AUT-02 |
| FR-AUT-014 | F-AUT-02 |
| FR-AUT-015 | F-AUT-02 |
| FR-AUT-016 | F-AUT-02 |
| FR-AUT-017 | F-AUT-02 |
| FR-AUT-020 | F-AUT-03 (part) |
| FR-AUT-021 | F-AUT-03 |
| FR-AUT-022 | F-AUT-03 |
| FR-AUT-023 | F-AUT-03 |
| FR-AUT-024 | F-AUT-03 |
| FR-AUT-030 | F-AUT-04 |
| FR-AUT-031 | F-AUT-04 |
| FR-AUT-032 | F-AUT-04 |
| FR-AUT-033 | F-AUT-04 |
| FR-AUT-034 | F-AUT-04 |
| FR-AUT-035 | F-AUT-04 |
| FR-AUT-036 | F-AUT-04 |
| FR-AUT-037 | F-AUT-04 |
| FR-AUT-038 | F-AUT-04 |
| FR-AUT-039 | F-AUT-04 |
| FR-AUT-040 | F-AUT-05 (part) |
| FR-AUT-041 | F-AUT-05 |
| FR-AUT-042 | F-AUT-05 |
| FR-AUT-050 | F-AUT-06 (part) |
| FR-AUT-051 | F-AUT-06 |
| FR-AUT-052 | F-AUT-06 |
| FR-AUT-053 | F-AUT-06 (part) |
| FR-AUT-054 | F-AUT-06 |
| FR-AUT-055 | F-AUT-06 |
| FR-AUT-056 | F-AUT-06 |
| FR-AUT-057 | F-AUT-06 |
| FR-AUT-058 | F-AUT-06 |
| FR-AUT-059 | F-AUT-06 |
| FR-AUT-070 | F-AUT-07 |
| FR-AUT-071 | F-AUT-07 |
| FR-AUT-072 | F-AUT-07 |
| FR-AUT-073 | F-AUT-07 |
| FR-AUT-074 | F-AUT-07 |
| FR-AUT-075 | F-AUT-08 |
| FR-AUT-076 | F-AUT-07 (part) |
| FR-AUT-077 | F-AUT-07 |
| FR-AUT-080 | F-AUT-09 |
| FR-AUT-081 | F-AUT-09 |
| FR-AUT-082 | F-AUT-09 |
| FR-AUT-083 | F-AUT-09 |
| FR-AUT-084 | F-AUT-09 |
| FR-AUT-085 | F-AUT-09 |
| FR-AUT-086 | F-AUT-09 |
| FR-AUT-087 | F-AUT-09 (part) |
| FR-AUT-088 | F-AUT-09 |
| FR-AUT-089 | F-AUT-09 |
| FR-AUT-090 | F-AUT-09 |
| FR-AUT-091 | F-AUT-09 |
| FR-AUT-100 | F-AUT-10 |
| FR-AUT-101 | F-AUT-10 |
| FR-AUT-102 | F-AUT-10 |
| FR-AUT-103 | F-AUT-10 |
| FR-AUT-104 | F-AUT-10 |
| FR-AUT-105 | F-AUT-10 |
| FR-AUT-106 | F-AUT-10 |
| FR-AUT-107 | F-AUT-10 (part) |
| FR-AUT-108 | F-AUT-10 |
| FR-APR-001 | F-APR-01 |
| FR-APR-002 | F-APR-01 |
| FR-APR-003 | F-APR-01 |
| FR-APR-004 | F-APR-01 |
| FR-APR-005 | F-APR-01 |
| FR-APR-006 | F-APR-01 |
| FR-APR-007 | F-APR-01 |
| FR-APR-010 | F-APR-01 |
| FR-APR-011 | F-APR-01 |
| FR-APR-012 | F-APR-01 |
| FR-APR-013 | F-APR-01 |
| FR-APR-014 | F-APR-03 (part) |
| FR-APR-020 | F-APR-02 |
| FR-APR-021 | F-APR-02 |
| FR-APR-022 | F-APR-02 |
| FR-APR-023 | F-APR-02 |
| FR-APR-030 | F-APR-03 |
| FR-APR-031 | F-APR-03 |
| FR-APR-032 | F-APR-03 |
| FR-APR-033 | F-APR-03 |
| FR-APR-040 | F-APR-04 |
| FR-APR-041 | F-APR-04 |
| FR-SET-001 | F-SET-01 |
| FR-SET-002 | F-SET-01 |
| FR-SET-003 | F-SET-01 |
| FR-SET-004 | F-SET-01 |
| FR-SET-010 | F-SET-02 |
| FR-SET-011 | F-SET-02 |
| FR-SET-012 | F-SET-02 |
| FR-SET-013 | F-SET-02 |
| FR-SET-014 | F-SET-02 |
| FR-SET-020 | F-SET-03 (part) |
| FR-SET-021 | F-SET-03 |
| FR-SET-022 | F-SET-03 |
| FR-SET-023 | F-IDN-09 |
| FR-SET-024 | F-IDN-09 |
| FR-SET-030 | F-SET-04 |
| FR-SET-031 | F-SET-04 |
| FR-SET-032 | F-SET-04 |
| FR-SET-033 | F-SET-04 |
| FR-SET-034 | F-SET-04 |
| FR-SET-040 | F-SET-05 |
| FR-SET-041 | F-APR-02 |
| FR-SET-042 | F-SET-05 |
| FR-SET-043 | F-SET-05 |
| FR-SET-044 | F-SET-05 |
| FR-SET-045 | F-SET-05 |
| FR-SET-046 | F-SET-05 |
| FR-SET-047 | F-SET-05 |
| FR-SET-050 | F-PRJ-04 (part) |
| FR-SET-051 | F-PRJ-04 (part) |
| FR-SET-052 | F-IDN-13 |
| FR-SET-060 | F-PRJ-10 (part) |
| FR-SET-061 | F-SET-06 |
| FR-SET-062 | F-SET-06 (part) |
| FR-SET-063 | F-SET-06 |
| FR-SET-064 | F-SET-06 (part) |
| FR-SET-065 | F-SET-06 |
| FR-SET-066 | F-SET-06 |
| FR-SET-067 | F-SET-06 (part) |
| FR-SET-068 | F-SET-06 (part) |
| FR-SET-069 | F-SET-06 |
| FR-SET-070 | F-SET-06 |
| FR-SET-080 | F-SET-07 (part) |
| FR-SET-081 | F-SET-07 |
| FR-SET-082 | F-SET-07 |
| FR-SET-083 | F-SET-07 |
| FR-SET-084 | F-SET-07 |
| FR-SET-085 | F-SET-07 |
| FR-SET-086 | F-SET-07 |
| FR-SET-087 | F-SET-07 |
| FR-SET-088 | F-SET-07 |
| FR-SET-089 | F-SET-07 |
| FR-SET-090 | F-SET-07 |
| FR-SET-091 | F-SET-07 |
| FR-SET-092 | F-SET-07 |
| FR-SET-093 | F-SET-07 |
| FR-SET-094 | F-SET-07 |
| FR-OPS-001 | F-OPS-02 (part) |
| FR-OPS-002 | F-OPS-02 (part) |
| FR-OPS-003 | F-OPS-01 |
| FR-OPS-010 | F-OPS-01 (part) |
| FR-OPS-011 | F-OPS-01 |
| FR-OPS-012 | F-OPS-01 |
| FR-OPS-013 | F-OPS-01 |
| FR-OPS-014 | F-OPS-02 (part) |
| FR-OPS-015 | F-OPS-03 |
| FR-OPS-020 | F-OPS-01 |
| FR-OPS-021 | F-OPS-01 |
| FR-OPS-022 | F-OPS-01 |
| FR-OPS-023 | F-OPS-03 (part) |
| FR-OPS-024 | F-OPS-03 |
| FR-OPS-025 | F-RUN-06 (part) |
| FR-OPS-026 | F-OPS-03 |
| FR-OPS-030 | F-OPS-01 (part) |
| FR-OPS-031 | F-OPS-03 (part) |
| FR-OPS-032 | F-OPS-01 |
| FR-OPS-033 | F-OPS-03 |
| FR-OPS-034 | F-RUN-03 (part) |
| FR-OPS-040 | F-OPS-01 |
| FR-OPS-041 | F-SET-07 |
| FR-OPS-042 | F-OPS-01 |
| FR-OPS-043 | F-OPS-01 |
| FR-OPS-050 | F-OPS-02 |
| FR-RUN-000 | F-RUN-02 (part), F-RUN-03 (part), F-RUN-07 (part) |
| FR-RUN-001 | F-RUN-01 |
| FR-RUN-002 | F-RUN-01 (part) |
| FR-RUN-003 | F-RUN-01 |
| FR-RUN-004 | F-RUN-01 |
| FR-RUN-005 | F-RUN-01 |
| FR-RUN-006 | F-RUN-01 |
| FR-RUN-007 | F-RUN-01, F-RUN-02 |
| FR-RUN-008 | F-RUN-01, F-RUN-02 |
| FR-RUN-009 | F-RUN-01 |
| FR-RUN-010 | F-RUN-01 (part) |
| FR-RUN-020 | F-RUN-04 |
| FR-RUN-021 | F-RUN-04 |
| FR-RUN-022 | F-RUN-04 |
| FR-RUN-023 | F-RUN-04 |
| FR-RUN-024 | F-RUN-03 |
| FR-RUN-025 | F-RUN-03 |
| FR-RUN-026 | F-RUN-04 |
| FR-RUN-027 | F-RUN-03 |
| FR-RUN-028 | F-RUN-04 |
| FR-RUN-029 | F-RUN-04 |
| FR-RUN-030 | F-RUN-05 |
| FR-RUN-031 | F-RUN-05 (part) |
| FR-RUN-032 | F-RUN-05 |
| FR-RUN-033 | F-RUN-05 (part) |
| FR-RUN-040 | F-RUN-06 |
| FR-RUN-041 | F-RUN-06 |
| FR-RUN-042 | F-RUN-06 |
| FR-RUN-043 | F-RUN-06 |
| FR-RUN-044 | F-RUN-06 |
| FR-RUN-045 | F-RUN-06 |
| FR-RUN-046 | F-RUN-06 |
| FR-RUN-050 | F-RUN-08 |
| FR-RUN-051 | F-RUN-08 |
| FR-RUN-052 | F-RUN-08 |
| FR-RUN-053 | F-RUN-08 |
| FR-RUN-054 | F-RUN-08 |
| FR-RUN-055 | F-RUN-08 |
| FR-RUN-056 | F-RUN-08 |
| FR-RUN-060 | F-RUN-09 |
| FR-RUN-061 | F-RUN-09 |
| FR-RUN-062 | F-RUN-09 |
| FR-RUN-063 | F-RUN-09 (part) |
| FR-RUN-064 | F-RUN-09 |
| FR-RUN-065 | F-RUN-09 |
| FR-RUN-066 | F-RUN-09 |
| FR-RUN-067 | F-RUN-09 |
| FR-RUN-068 | F-RUN-10 (part) |
| FR-RUN-069 | F-RUN-10 |
| FR-RUN-070 | F-RUN-10 |
| FR-RUN-071 | F-RUN-10 |
| FR-RUN-080 | F-RUN-11 (part) |
| FR-RUN-081 | F-RUN-11 (part) |
| FR-RUN-082 | F-RUN-11 |
| FR-RUN-083 | F-RUN-11 |
| FR-RUN-084 | F-RUN-12 |
| FR-RUN-085 | F-RUN-12 |
| FR-RUN-090 | F-RUN-13 |
| FR-RUN-091 | F-RUN-13 |
| FR-RUN-092 | F-RUN-13 (part) |
| FR-RUN-093 | F-RUN-13 |
| FR-RUN-094 | F-RUN-13 |
| FR-RUN-095 | F-RUN-13 |
| FR-RUN-096 | F-RUN-13 |
| FR-RUN-097 | F-RUN-13 |
| FR-RUN-098 | F-RUN-13 |

## III.2 Design rules → feature

| Design rule | Feature | Node |
|---|---|---|
| One-time download of recovery codes | F-IDN-05 | R36 `148:1333` |
| Retry session refresh | F-IDN-08 | R34 `148:489` |
| Decline invitation | F-IDN-10 | M03 `161:51284` |
| Initial roles and `canRun` set by explicit checkboxes at project creation | F-PRJ-02 | N22 in D02 `159:32045` |
| "My grants" project filter | F-PRJ-02 | R14 `148:2226` |
| Workflow progress header INTENT → … → RUN | F-PRJ-03 | R15 `148:2024`, R22 `148:2854`, N09 `161:49884` |
| Macro review separation policy, self-review explicit and audited | F-CAT-04 | R26 `191:32782`; Authoring board `79:4061` |
| `AdmissionTicket` field set | F-RUN-03 | BE board `23:5587` |
| Blocked head isolation, narrow cursor, release after known cleanup, no machine-minted Run | F-RUN-03 | BE board `23:5587` |
| A changed pin supersedes the queued ticket | F-RUN-04 | N32 `191:34828` |
| Runner egress isolated from BE and internal metadata networks | F-RUN-07 | register `122:2` §8 |
| Fencing of late results and callbacks; suspension freeze | F-RUN-07 (design rule); F-IDN-16 (acceptance rule 1) | Failure board `3:2192` |
| Base report download | F-RUN-09 | R32 `191:33192`, N16 `161:50521`, N41 `191:34873` |

## III.3 Glossary

| Term | Meaning |
|---|---|
| **Uncovered item** | The part of an authored criterion with no accepted executable meaning. It is recorded in C11, stays visible against its criterion and is excluded from every coverage claim. |
| **NOT EXECUTED** | A queued intent that was canceled, rejected or expired before any execution. It has no Core verdict and is not a failure. |
| **INDETERMINATE** | Execution may have happened but the acknowledgement was lost. It is resolved by lookup and reconciliation of the same submission, never by assumption. |
| **UNCONFIGURED** | An operator record that is absent or invalid. The dependent effect is blocked. |
| **(part)** | The requirement is implemented by this feature only in the clause given under *Scope of partial requirements*. |

---

**AI proposes. Humans authorise. Core decides.**

## III.4 Requirements outside this baseline

The requirements below are defined in `requirements.md` and are not part of this implementation baseline. They are listed so that their absence from III.1 is a stated decision, not an omission; each is a scope statement, not a defect. The capabilities they describe are marked *(extension — outside the MVP baseline)* in `scope.md` §5 and, where designed, live on the Figma Post-MVP page (`204:2`, "Deferred references · NOT active MVP").

| Requirements | Capability and where it lives |
|---|---|
| FR-EXT-001 … FR-EXT-012 | Repository-informed scenario proposals (area I.1) — Post-MVP page, board 8 |
| FR-EXT-020 … FR-EXT-023 | Federated identity (I.2) — Post-MVP page, deferred reference frames |
| FR-EXT-030 … FR-EXT-032 | Phishing-resistant factors and trusted devices (I.3) — Post-MVP page |
| FR-EXT-040 … FR-EXT-043 | Machine-initiated execution through `C05-S` (I.4) |
| FR-EXT-050 … FR-EXT-052 | Customer-network execution and warm browser pools (I.5) |
| FR-EXT-060 … FR-EXT-064 | Provider families beyond `JIRA_CLOUD_ISSUES` (I.6) |
| FR-EXT-070, FR-EXT-071 | Cross-project reuse of macros and assets (I.7) |
| FR-EXT-080, FR-EXT-081 | Client certificate identity (I.8) |
| FR-EXT-090 … FR-EXT-092 | Customer-hosted inference, vector indexes and Python workers (I.9) |
| FR-EXT-100, FR-EXT-101 | Commercial billing (I.10) |
| FR-IDN-124, FR-IDN-125 | Tenant export and deletion as durable jobs (A.9); F-IDN-17 records the erasure request only |
| FR-PRJ-035 | Authorized snapshot export (B) |
| FR-PRJ-048, FR-PRJ-049 | Purge by retention workers with per-item impact review (B.5); F-PRJ-07 covers Archive, Trash and Restore |
