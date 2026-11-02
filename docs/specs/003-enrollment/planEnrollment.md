# Plan 003 — TOTP enrollment, recovery codes and factor replacement

**Feature** Second-factor enrollment, recovery codes and authenticator replacement · **Plan version** 0.2 · **Date** 27 October 2026 · **Status** Draft for review — QA alignment of 27 October 2026 applied (endpoint paths, discard reasons and audit mapping, reload handling, ArchUnit, register split) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/003-enrollment/specEnrollment.md` v0.2 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; features F-IDN-04 in its enrollment half, F-IDN-05 and F-SET-03 as the subject; F-IDN-03, F-IDN-06…F-IDN-09, F-IDN-11, F-IDN-14, F-IDN-16, F-SET-01, F-SET-05 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma BE board `23:5587` (Java package DAG) and FE board `46:7847` (feature-modular SPA DAG) · **Builds on** plan 001 (Login), plan 002 (Signup and verification) and plan 004 (Password reset); elements they own are tagged **[001]** / **[002]** / **[004]** and are extended, never duplicated.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's twenty decisions (Q1–Q20) are built as decided; the fourteen of them recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-37…A-43, S-28…S-34) are therefore implemented, not assumed. Where the specification depends on items still open from earlier specifications (A-27, A-32, A-35, A-39), the plan builds the stated assumption and names the item (§6). The one item this plan raised itself, the QR encoder (A-46), is decided (D15). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers they implement are listed in the specification and are not repeated here except where a decision rests directly on one.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | No use case of this feature touches a mapping, approval, run or intent table; activation, regeneration and replacement end in the identity tables and the outbox. | RF-5.6, RF-9.8, RNF-13 |
| **P3** The browser expresses intent, never authority | The FE renders setup material, submits codes and acknowledgements, and shows server state. Eligibility, verification, activation, the gate and the reason for the gate are server decisions; the wizard's "reason" is carried from a server problem, never typed into a route. | RF-1.5, RF-3.4, RF-5.1, RF-7.2, RNF-13 |
| **P4** Only the platform catalog is global | The factor and the recovery codes are identity-scoped and never duplicated per tenant; one enrollment satisfies every tenant (FR-IDN-001). | RF-1.6, LC-21 |
| **P6** Fail closed | An absent, expired, superseded, discarded or unverified pending enrollment, an unconsumed or expired step-up, an unknown session or an absent configuration value refuses the dependent change; nothing is assumed and no fallback exists. | RF-3.7, RF-6, RF-8.4, RNF-6, RNF-7 |
| **P10** Fixed technology stack | BE: Java 21, Spring Boot 3.5, PostgreSQL, the existing Spring Security MFA support and `PasswordEncoder`, the existing outbox and durable jobs. FE: React 18, TypeScript, Vite, TanStack Query, i18next, Tailwind, axios. The QR encoder for the setup graphic is ZXing `core` 3.5.3, named in P10 by constitution v1.8 (register A-46, decided 2 November 2026; D15). Test-only dependencies are flagged in §5. | whole plan |
| **P11** Architecture styles | BE: the feature lives in the existing hexagonal bounded context `identity`; the enrollment gate is one pure rule inside `access` where C10 is derived, fed by a port the identity context publishes; outer adapters depend inward; mail through the outbox; expiry through durable jobs. FE: the same feature module as Login; TanStack Query owns server facts; Zustand is not used (no unsaved fork). | §2 |
| **P12** Two independent tenancy gates | Factor, recovery-code, pending-enrollment and step-up rows are **platform** rows of a global identity with no tenant column, protected by application authority. The gate is applied by `access` at C10 derivation (runtime authorisation) and changes no RLS predicate: an unenrolled Owner's membership row exists and is read; what is withheld is derived authority. | RF-1.3–1.5, LC-20, LC-23 |
| **P13** Configured values are never defaulted | Code count, code entropy, pending-enrollment expiry, verification attempt limit, step-up expiry and attempt limit are one validated configuration record; the authenticator time tolerance is the global value of `LoginPolicy` [001] (A-32); startup fails on any absent or unbounded value. | RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6 |
| **P14** Release bar | WCAG 2.2 AA on wizard, dialogs, reveal and settings area with automated and manual assistive evidence; EN / ES / FR for screens and the three messages; the setup material has a text alternative. | RF-12, RNF-10–12 |
| **F-X-01** C10 context | No C10 is derived by this feature's endpoints (identity-scoped, session-authorised). The gate is evaluated inside the `access` derivation of C10 for tenant requests and inside workspace creation's precondition (spec 012), both through `FactorStatusPort`. | RF-1.2–1.5, RF-5.3 |
| **F-X-02** Request integrity | Verify, acknowledge and step-up are consequential writes with no generic auto-retry; each carries an idempotency key whose outcome is recorded and returned on re-submission; a lost acknowledgement leads to a lookup, never to a second activation, set or revocation. | RF-11.5, LC-11 |
| **F-X-03** Secrets and safe problems | The seed exists in clear only inside the start response and the verifier; at rest under `SeedCipher` [001]. Recovery codes exist in clear only inside the one reveal response; at rest as maintained-encoder hashes. Problem details carry safe codes and a correlation id. Browser storage holds nothing from this feature. | RF-4.7, RF-4.8, RF-11.2, RF-11.3, RNF-1 |
| **F-X-05** Audit log | Every event of RF-11.1 is appended through the `audit` context port, redacted, in the same transaction as the effect. | RF-11.1, RF-11.2 |
| **F-X-06** Configuration validation and email port | `EnrollmentPolicyProperties` is part of startup validation; the three notifications go through `TransactionalEmailPort` behind the outbox, so delivery failure can never revert the change. | RF-5.5, RF-8.9, RF-9.7, RNF-9 |
| **F-X-07** Responsive surfaces | M13 (`431:4190`) and M14 (`431:4222`) are the drawn counterparts of the wizard and the reveal; R41, R35 and the R38 area follow the D01 responsive policy (S-28, decided) with `SecurityStatements` shared across widths. | RF-12.1, RNF-12 |
| **F-SET-01** Settings write discipline | Every write confirms persistence before the FE shows success (the response is the committed state); no settings surface reveals or persists a secret; the area shows metadata only. | RF-7.3, RF-7.7, RF-11.5 |

---

## 2. Module structure

### 2.1 Back end — bounded context `com.aspectra.identity`, plus one rule in `access`

The context, its four layers (`api / application / domain / infrastructure`) and its published ports exist from plans 001, 002 and 004. This plan adds two aggregates, one step-up challenge, seven use cases, one durable job, one published port and one outbound port. Contexts touched — `access` (gate and requirement reason), `audit`, `sharedsupport`, and the session service of F-IDN-08 — are reached only through published ports (allowed-import DAG: outer adapters → ports and use cases → pure domain; no foreign repositories).

| Layer | Element | Responsibility | Covers |
|---|---|---|---|
| `identity.domain` | `TotpFactor` **[001]** | Active factor: encrypted seed plus server key reference, `replayWatermark`, new `enrolledAt` and `seedVersion`. `replaceWith(pending, now)` returns the successor factor; the previous seed is invalidated by the same write. Never exposes the seed. | RF-5.1, RF-7.3, RF-9.6, RF-11.3 |
| | `RecoveryCodeSet` **[001]** | Hashed single-use codes; gains `setId`, `unusedCount()` and `invalidateAll(now)`. `consume` **[001]** is unchanged and is the only reader of the hashes. | RF-4.8, RF-7.6, RF-8.6, RF-9.6, LC-12 |
| | `PendingEnrollment` | Aggregate of one enrollment in progress: identity id, `purpose` (`INITIAL`, `REPLACEMENT`, `REGENERATE`), state machine `SET_UP → VERIFIED → REVEALED → ACKNOWLEDGED \| DISCARDED` (a `REGENERATE` enrollment is created directly in `VERIFIED`), encrypted pending seed and key reference (none for `REGENERATE`), `replayWatermark`, `expiresAt`, `attemptsRemaining`, `owningSessionId`, `revealedAt`, the pending code hashes (filled at reveal), `discardReason` (`SUPERSEDED`, `LEFT`, `SECOND_REVEAL`, `EXPIRED`, `ATTEMPTS`, `SESSION_ENDED`), and the recorded outcome of each submission by its idempotency key. Methods: `verify(codeOk, now)`, `reveal(hashes, now)`, `acknowledge(now)`, `discard(reason, now)`, `recordedOutcome(key)`. Pure; takes the clock as an argument. | RF-2.1, RF-2.6, RF-3.2, RF-3.4, RF-3.7, RF-4.1, RF-4.6, RF-6.1–6.4, RF-9.3–9.5, LC-1–LC-8 |
| | `StepUpChallenge` | Identity id, `purpose` (`REGENERATE`, `REPLACE`), `expiresAt`, `attemptsRemaining`, state `OPEN → SUCCEEDED(consumedByPendingId) \| VOIDED`, idempotency-keyed outcomes. A succeeded challenge is consumed in the same transaction by the pending enrollment it creates, so it can never be presented twice or for another purpose. | RF-8.1–8.4, RF-9.1, RF-9.2, RNF-5, LC-13–LC-15 |
| | `EnrollmentPolicy` | Immutable record: code count, code entropy (bits), pending-enrollment expiry, verification attempt limit, step-up expiry, step-up attempt limit. The time tolerance is read from `LoginPolicy` **[001]** (A-32). | RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6 |
| | `AbuseThrottlePolicy` **[001]** | Reused unchanged; a failed step-up is written on the sign-in keys `ACCOUNT` and `NETWORK` (it is a first factor plus a code, exactly what sign-in protects). | RF-8.3, LC-15 |
| `access.domain` | `EnrollmentGate` | Pure rule `(tenantRole, tenantPolicy.requireMfaForMembers, hasActiveFactor) → NOT_REQUIRED \| REQUIRED(reason)` with reasons `OWNER_ADMIN`, `MEMBER_POLICY`. Evaluated by the C10 derivation of `access` (F-X-01) on every tenant request; when `REQUIRED`, derivation yields no privileged authority and the request is answered with `auth.enrollment.required {reason}`. The rule reads the policy at derivation time, so a tightened policy applies to the next request (Q12). | RF-1.3–1.6, RF-10.2, LC-20, LC-21, LC-23 |
| `identity.application` | `StartEnrollmentUseCase` | Preconditions: identity `ACTIVE` and verified (A-27; otherwise the verification handoff of plan 001 D9 or a refusal), and for `INITIAL` no active factor (RF-1.6). Discards any pending enrollment of the identity (`SUPERSEDED`), generates a seed, encrypts it under `SeedCipher`, creates the `PendingEnrollment` bound to the caller's session, audits `ENROLLMENT_STARTED{purpose}`, and returns the setup material **once**: the provisioning content (issuer, account label, secret), the secret as copyable text, and the SVG rendered by `QrEncoder`. For `REPLACEMENT` it is invoked by `StepUpUseCase` after a step-up, or by plan 005's recovery use case after its own proof of control (005 RF-3) — never directly from a client. | RF-1.1, RF-1.7, RF-1.8, RF-2.1, RF-2.2, RF-2.6, RF-6.1, RF-9.3, LC-8, LC-24, LC-25 |
| | `VerifyEnrollmentCodeUseCase` | Locks the pending row; checks owning session, state `SET_UP`, expiry and attempts; returns the recorded outcome for a repeated key; strips separators; verifies with `TotpVerifier` against the pending seed, the global tolerance and the pending watermark; success → advance watermark, `VERIFIED`; failure → decrement, `mfa.code.rejected`; zero or expired → `discard(ATTEMPTS \| EXPIRED)`, `mfa.enrollment.restart`. Never activates anything. | RF-3.1–3.8, LC-2, LC-5–LC-8 |
| | `OpenRevealUseCase` | Locks the row; requires owning session and state `VERIFIED`; generates `codeCount` codes of `entropyBits` with `TokenGenerator`, stores their maintained-encoder hashes on the pending enrollment, moves to `REVEALED`, audits `REVEAL_OPENED`, and returns the plain codes with the purpose and issue time **once**. A second call on a `REVEALED` row discards the enrollment (`SECOND_REVEAL`) and answers `mfa.enrollment.restart` (RNF-2). | RF-4.1, RF-4.2, RF-4.6, RF-4.8, RF-8.5, LC-3, LC-4 |
| | `AcknowledgeRevealUseCase` | Locks the row; requires owning session and state `REVEALED`; returns the recorded outcome for a repeated key; then one transaction by purpose — `INITIAL`: create `TotpFactor` from the pending seed with `enrolledAt`, create the `RecoveryCodeSet` from the pending hashes, template `enrollment-completed`; `REGENERATE`: `invalidateAll` on the current set, create the new set, template `recovery-codes-regenerated`; `REPLACEMENT`: `TotpFactor.replaceWith`, `invalidateAll`, new set, `bumpSecurityVersion` **[004]**, `RevokeSessionsPort.revokeAll(identityId, except = owningSessionId)` **[004]**, template `authenticator-replaced`. All purposes: `ACKNOWLEDGED`, delete the pending hashes, audit `ENROLLMENT_COMPLETED{purpose}` (and `SESSIONS_REVOKED` for replacement). Never creates a session, enters a tenant or touches any run table. | RF-4.5, RF-5.1, RF-5.2, RF-5.5, RF-5.6, RF-8.6, RF-8.9, RF-9.6–9.8, RNF-3, RNF-4, LC-10, LC-11, LC-18, LC-19 |
| | `DiscardPendingEnrollmentUseCase` | `discard(LEFT)` for the caller's pending enrollment; idempotent (a missing or already discarded row is a success); destroys seed and hashes; leaves the active factor and set untouched; audits `ENROLLMENT_ABANDONED{LEFT}`. | RF-2.5, RF-4.6, RF-6.3, RF-8.7, RF-8.8, RF-9.5, RF-9.10, LC-1, LC-9, LC-13, LC-16 |
| | `ExpirePendingEnrollmentsJob` | Durable job (`sharedsupport/jobs` **[002]**): claims pending enrollments and step-ups past `expiresAt` or whose owning session no longer exists (through `CurrentSessionPort.exists`), discards them (`EXPIRED` / `SESSION_ENDED`), audits `ENROLLMENT_EXPIRED`. Check-on-access in every use case enforces expiry first; the job only cleans up. | RF-6.2, RF-6.3, LC-5 |
| | `StepUpUseCase` | Creates or continues a `StepUpChallenge` for the caller and purpose; locks it; checks expiry and attempts; verifies the password with `PasswordVerifier` and the code with `TotpVerifier` against the **active** factor (watermark advanced on success); both must pass — any failure is one generic `mfa.step-up.rejected`, counted on `ACCOUNT` and `NETWORK` and on the challenge's attempts; zero or expiry → `VOIDED`, `mfa.step-up.void`. On success, in the same transaction: `REGENERATE` → create a `PendingEnrollment(REGENERATE)` in `VERIFIED`; `REPLACE` → `StartEnrollmentUseCase(REPLACEMENT)` and return its setup material; the challenge records `consumedByPendingId`. The code is never offered to `RecoveryCodeSet.consume` (D6). | RF-8.1–8.5, RF-9.1–9.3, RNF-5, LC-13–LC-15 |
| | `SecurityMfaQuery` | For the caller's identity: factor state (enrolled or not), `enrolledAt`, display name and verified email, `unusedCount`, and the requirement reason from `RequirementReasonPort` (`OWNER_ADMIN`, `MEMBER_POLICY`, `NONE`, with the tenant names that impose it). Never the seed, a code, or consumed codes by value. | RF-7.2–7.7, LC-12, LC-30 |
| | `FactorStatusPort` (published) | `hasActiveFactor(identityId)`, `enrolledAt(identityId)`. Consumers: the `access` C10 derivation (`EnrollmentGate`) and workspace creation (spec 012, F-IDN-14). A pending enrollment is never reported as a factor. | RF-1.2–1.5, RF-6.4 |
| | Ports | New: `PendingEnrollmentRepository`, `StepUpRepository`, `CurrentSessionPort` (caller's session id, session existence → session service F-IDN-08), `RequirementReasonPort` (published by `access`), `QrEncoder` (→ `sharedsupport/qr`, D15 / A-46). Reused: `IdentityRepository`, `TotpVerifier`, `SeedCipher`, `PasswordVerifier`, `ThrottleStore`, `AuditPort`, `Clock` **[001]**; `TokenGenerator`, `JobScheduler`, `TransactionalEmailPort` (behind the outbox), `SecurityNotificationPort` **[002]**; `RevokeSessionsPort` **[004]**. | all |
| `identity.infrastructure` | Persistence | `identity_totp_factor` **[001]** gains `enrolled_at`, `seed_version`; `identity_recovery_code` **[001]** gains `set_id`, `consumed_at`. New `pending_enrollment` (identity id, purpose, state, encrypted seed + key ref, watermark, expires_at, attempts_remaining, owning_session_id, revealed_at, discard_reason, created_at; partial unique index on `identity_id WHERE state NOT IN ('ACKNOWLEDGED','DISCARDED')`), `pending_recovery_code` (pending id, hash; deleted on acknowledge or discard), `step_up_challenge` (identity id, purpose, state, expires_at, attempts_remaining, consumed_by). Every mutating use case reads its row `SELECT … FOR UPDATE`. Persistence access API and migration tool per A-35 (placeholders). | RF-6, RF-8, RF-9, RNF-3, LC-8, LC-11 |
| | Outbox relay and email adapter **[002]** | Three templates in EN / ES / FR — `enrollment-completed`, `recovery-codes-regenerated`, `authenticator-replaced` — each stating the event and its time, with no code, seed or link that grants access. Relay failures retried by the relay; never surfaced to the request. | RF-5.5, RF-8.9, RF-9.7, RF-11.4, RNF-9, LC-29 |
| | `EnrollmentPolicyProperties` | `@ConfigurationProperties` + `@Validated` binding into `EnrollmentPolicy`; constraints: present, positive, bounded above by a configured ceiling; entropy at least a configured floor. Any violation fails startup. | RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6 |
| | QR adapter | Implements `QrEncoder.svg(content)` with ZXing `core` 3.5.3 (A-46): encodes the provisioning content at error-correction level M with a four-module quiet zone and writes the module matrix as one SVG path; the ZXing `javase` module and AWT rendering are not used; output is an inline SVG string with no external reference. | RF-2.2, RF-12.4 |
| | Session adapters | `CurrentSessionPort` and `RevokeSessionsPort` **[004]** against the existing session service; no change to JWT / refresh mechanics. | RF-9.7, RF-6.3, LC-17, LC-18 |
| | Audit adapter | Events: `ENROLLMENT_STARTED{purpose}`, `ENROLLMENT_VERIFIED`, `ENROLLMENT_COMPLETED{purpose}`, `ENROLLMENT_ABANDONED{reason}` for the person-driven discards `SUPERSEDED`, `LEFT` and `SECOND_REVEAL`, `ENROLLMENT_EXPIRED{reason}` for the limit-driven discards `EXPIRED`, `ATTEMPTS` and `SESSION_ENDED` (together the "abandoned" and "expired" events of RF-11.1; every discard is audited under one of the two), `REVEAL_OPENED`, `REVEAL_ACKNOWLEDGED`, `STEP_UP_SUCCEEDED{purpose}`, `STEP_UP_FAILED{purpose}`, `STEP_UP_VOIDED`, `SESSIONS_REVOKED{cause=REPLACEMENT}`, `NOTIFICATION_ENQUEUED{template}`; `access` appends `ENROLLMENT_GATE_APPLIED{tenant, reason}`. Payload: identity id, correlation id, network key, timestamp. Never seed, material, code, recovery code, count by value or password. | RF-11.1, RF-11.2 |
| `identity.api` | `MfaController` — `/api/account` (requires an identity session; identity-scoped; no C10) | `POST mfa/enrollment` → 200 `{pendingId, purpose, setup: {uri, key, svg}, expiresAt}` or 409 `mfa.enrollment.already-enrolled` or 409 `auth.verification-pending` **[001]** · `POST mfa/enrollment/{id}/verify` {code, idempotencyKey} → 200 `{state: VERIFIED}` or 400 `mfa.code.rejected` (no cause, no count) or 410 `mfa.enrollment.restart` · `POST mfa/enrollment/{id}/reveal` → 200 `{codes[], purpose, issuedAt}` once or 410 `mfa.enrollment.restart` · `POST mfa/enrollment/{id}/acknowledge` {acknowledged: true, idempotencyKey} → 200 `{state: ACTIVE \| REGENERATED \| REPLACED, enrolledAt, unusedCount}` or 400 field problem (acknowledgement not ticked) or 410 · `DELETE mfa/enrollment/{id}` → 204 always · `POST mfa/step-up` {purpose, password, code, idempotencyKey} → 200 `{pendingId, purpose, setup?}` or 400 `mfa.step-up.rejected` or 410 `mfa.step-up.void` · `GET security/mfa` (`/api/account/security/mfa`, the server counterpart of the FE route `/account/security`) → 200 `{factor: {state, enrolledAt?}, identity: {displayName, email}, recoveryCodes: {unusedCount?}, requirement: {reason, tenants[]}}`. Tenant endpoints answer 403 `auth.enrollment.required {reason}` from `access`. All problems are RFC 9457 with a safe code, field paths and a correlation id. The server never redirects. | RF-1, RF-2.2, RF-3, RF-4, RF-5.4, RF-7, RF-8, RF-9 |

**Cross-context touch points.** `access` — `EnrollmentGate` inside C10 derivation, `RequirementReasonPort`, reads `FactorStatusPort`. `audit` — append-only. `sharedsupport` — `crypto` (seed cipher, token generation, hashing), `jobs`, `qr`, `tx`, `contextIds`, `outbox`. The session service through `CurrentSessionPort` and `RevokeSessionsPort`. Nothing reads or writes project, grant, approval or run tables; membership and tenant-policy rows are read by `access` only.

**Data model note (P12).** The five tables of this feature are platform-scoped rows of a global identity and carry no tenant column; RLS does not apply to them. The membership and policy rows the gate reads stay in `access` under RLS and are read through the normal tenant-scoped derivation.

### 2.2 Front end — feature module `src/features/identity-access` (shared with Login, Signup and Password reset)

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `routes` | `/account/security` (R38; the TOTP & recovery area is this feature's, the Sessions area is a slot owned by 007); `/account/security/enroll` (R39 wizard; purpose, reason and return destination are held in memory, never in the route). | RF-1.1, RF-1.9, RF-7 |
| | `useStartEnrollment()`, `useVerifyCode()`, `useOpenReveal()`, `useAcknowledge()`, `useDiscard()`, `useStepUp()`, `useSecurityMfa()` | Typed intent hooks; attach the UI locale header; return server state verbatim. | RF-2–RF-9 |
| | `contracts/mfa` (in `src/contracts`) | Request / response / problem types and runtime guards for the seven endpoints and the `auth.enrollment.required` problem. | all |
| Views | `SecuritySettingsPage` | R38 heading and subtitle, the TOTP & recovery area (`MfaCard`, `RecoveryCodesCard`), slots for Sessions (007), Password (004, statement only) and Privacy. | RF-7.1 |
| | `MfaCard` | Enrolled state: identity, requirement line, enrollment time, "Enrollment metadata only; seed never exposed", "Replace authenticator". Not-enrolled state: same position, "No authenticator enrolled", requirement line, "Set up authenticator". | RF-7.2–7.4, LC-30 |
| | `RecoveryCodesCard` | Statements, unused count as text, inline zero-codes warning above "Regenerate recovery codes", the "Controlled account recovery" route; no actions when no factor. | RF-7.4–7.6, LC-12 |
| | `EnrollmentWizard` | One component for the three purposes and the four entries: heading per RF-1.9, three-step indicator, statements, "Leave setup" on every step; `SetupStep` (inline SVG, copyable key, nothing persisted), `VerifyStep` (code field with paste and separator stripping, in-progress state, generic failure, restart handling), `RevealStep` (renders `RecoveryCodesReveal`). | RF-1.9, RF-2.2–2.6, RF-3.1, RF-3.3, RF-3.6, RF-3.7, RF-3.8, RF-9.3 |
| | `RecoveryCodesReveal` | R36 for step 3 and for regeneration ("Save your replacement codes."): codes in full, statements, "Download securely issued codes" (file built from the response per Q17), the unticked acknowledgement, finish action disabled until ticked; download and copy never acknowledge. | RF-4.1, RF-4.3–4.5, RF-8.5, LC-9, LC-10 |
| | `RegenerateCodesDialog` | R35: statements, warning, password with Show, code field, "Keep existing codes", "Verify & regenerate codes", generic failure. | RF-8.1, RF-8.8 |
| | `ReplaceAuthenticatorDialog` | R41: statements, warning, password with Show, code field, "Keep current factor", "Verify & begin replacement", the "I lost access · controlled recovery" route, generic failure. | RF-9.1, RF-9.9, RF-9.10, LC-27 |
| | `SecurityStatements` **[001]** | Extended with every statement of R38, R39, R41, R35 and R36 so desktop and reflow cannot drift. | RF-2.4, RF-7.2, RF-8.1, RF-9.1, RF-12.1 |
| | `GenericFailureMessage` **[001]** | Reused for `mfa.code.rejected` ("We couldn't verify the code") and `mfa.step-up.rejected`. | RF-3.6, RF-8.3 |
| State | TanStack Query | Mutations for start, verify, reveal, acknowledge, discard, step-up (verify, acknowledge and step-up generate their idempotency key once and reuse it on re-submission); query for `GET security/mfa`, invalidated after every completion. No retry on mutations (F-X-02). | RF-2–RF-9, RF-11.5, LC-11 |
| | Enrollment holder | Module-private in-memory record `{pendingId, purpose, reason, returnTo}`; cleared on acknowledge, discard, restart and route leave; `DELETE mfa/enrollment/{id}` fired on leave and, as a best-effort `keepalive` request, on `pagehide` so a reload or tab close also reaches the server. Nothing in storage; a reload therefore starts over (Q3, Q13): the holder is gone and, if the best-effort call did not land, the old row can never return codes again (any call on it discards) and is discarded by the next start (`SUPERSEDED`), by the expiry job or by the end of its session. | RF-2.5, RF-2.6, RF-4.6, RF-4.7, LC-1–LC-4 |
| | Reveal holder | The codes from the reveal response, held only while `RecoveryCodesReveal` is mounted; dropped on unmount; never in storage. | RF-4.6, RF-4.7, RNF-1, RNF-2 |
| | Setup holder | The setup material from the start response, held only while `SetupStep` is mounted. | RF-2.2, RF-2.6, RF-3.8 |
| Routing | `EnrollmentGateHandler` (in `shared/errors`) | Maps `auth.enrollment.required {reason}` from any tenant request, and the enrollment precondition of workspace creation (spec 012), to the wizard with `reason` and `returnTo` (the interrupted tenant route or the workspace-creation entry) in the holder; after activation navigates to `returnTo`, which re-requests and now derives authority. | RF-1.2–1.4, RF-5.3, LC-20, LC-22, LC-23 |
| | Handoffs | "Leave setup" → discard, then the tenant chooser, the workspace-creation entry or `/account/security` per the entry (Q20); `mfa.enrollment.restart` → step 1 with a fresh start; `mfa.step-up.void` → dialog closed, nothing changed; controlled recovery → its route (spec 005); `auth.verification-pending` → verification pending state **[002]**. | RF-2.5, RF-3.7, RF-5.4, RF-8.4, RF-9.9, LC-24 |
| Shared used | `shared/api` (correlation id, locale header, no POST retry), `shared/errors` (field paths, gate handler), `shared/i18n` (bundle keys for every statement, state and message), `shared/ui` (accessible `Field`, `PasswordField`, `CodeField` **[001]**, `Checkbox`, `Dialog`, `StepIndicator`, `LiveRegion`, 44 px targets). | RF-12.2, RF-12.3, RF-12.5, RNF-10, RNF-11 |

---

## 3. Behaviour-to-module trace

One row per EARS criterion of the specification.

| Criterion | Back end | Front end |
|---|---|---|
| RF-1.1 | `SecurityMfaQuery` reports no factor | `MfaCard` not-enrolled state with "Set up authenticator" |
| RF-1.2 | Workspace creation (spec 012) checks `FactorStatusPort` before accepting any field | `EnrollmentGateHandler` routes to the wizard with `returnTo` = creation entry |
| RF-1.3 | `EnrollmentGate` in `access` C10 derivation → no privileged authority; `auth.enrollment.required {OWNER_ADMIN}` | Handler → wizard with the Owner heading |
| RF-1.4 | `EnrollmentGate` with `tenantPolicy.requireMfaForMembers` → `MEMBER_POLICY` | Handler → wizard with the Member-policy subtitle |
| RF-1.5 | Derivation yields no authority while `REQUIRED`; a session, challenge or membership is not consulted as a substitute | No tenant data fetched before `returnTo` succeeds |
| RF-1.6 | `EnrollmentGate` reads `hasActiveFactor` only; `StartEnrollmentUseCase(INITIAL)` refuses when a factor exists | `MfaCard` enrolled state offers only "Replace authenticator" |
| RF-1.7 | `StartEnrollmentUseCase` → `auth.verification-pending` **[001]** | Handoff to the verification pending state |
| RF-1.8 | `StartEnrollmentUseCase` refuses for non-`ACTIVE` statuses (A-27) | — |
| RF-1.9 | `auth.enrollment.required.reason` and the step-up purpose supply the entry | `EnrollmentWizard` heading and subtitle by entry; steps and statements shared |
| RF-2.1 | `StartEnrollmentUseCase`: supersede, new seed, new pending row | — |
| RF-2.2 | Start response carries `setup.uri`, `setup.key`, `setup.svg` once | `SetupStep` inline SVG and copyable key |
| RF-2.3 | — | `StepIndicator` with the current step named in text |
| RF-2.4 | — | `SecurityStatements` (verify before acknowledgement, one-time codes, gate statement) |
| RF-2.5 | `DiscardPendingEnrollmentUseCase` | "Leave setup" → discard → chooser / creation entry / settings per entry |
| RF-2.6 | Setup material is never re-read: no endpoint returns it after start | Setup holder dropped on unmount |
| RF-3.1 | Separator and whitespace stripping before `TotpVerifier` | `CodeField` **[001]** paste handling |
| RF-3.2 | `PendingEnrollment` bound to identity, purpose, `expiresAt`, `attemptsRemaining` from `EnrollmentPolicy` | — |
| RF-3.3 | — | `VerifyStep` in-progress state, resubmission disabled |
| RF-3.4 | `VerifyEnrollmentCodeUseCase` → `VERIFIED`; no factor written | Advance to `RevealStep` |
| RF-3.5 | Pending `replayWatermark` advanced on success; code at or below it rejected | — |
| RF-3.6 | `mfa.code.rejected` without cause or count | `GenericFailureMessage` "We couldn't verify the code" |
| RF-3.7 | Zero attempts or expiry → `discard`, `mfa.enrollment.restart` | "Setup must be restarted" → fresh start at step 1 |
| RF-3.8 | Verify response carries no material | `VerifyStep` renders no material |
| RF-4.1 | `OpenRevealUseCase` returns plain codes once | `RecoveryCodesReveal` shows every code with the statements |
| RF-4.2 | `EnrollmentPolicyProperties` (count, entropy) | — |
| RF-4.3 | — | Download built client-side from the response: heading with product and issue date, one code per line |
| RF-4.4 | `acknowledge` refuses `acknowledged != true` with a field problem | Checkbox unticked by default; finish disabled until ticked |
| RF-4.5 | `AcknowledgeRevealUseCase` by purpose | Finish → acknowledge; download and copy call nothing |
| RF-4.6 | Second reveal call or any call on a `REVEALED` row from a new mount → `discard`, `mfa.enrollment.restart`; a row left behind by a reload is discarded by the next start, the expiry job or its session end | Reveal holder dropped on unmount; leave → discard; reload or tab close → best-effort `keepalive` discard on `pagehide` |
| RF-4.7 | — | Reveal holder in memory only; no storage write |
| RF-4.8 | Pending hashes only; deleted on acknowledge or discard; active set hashed | — |
| RF-5.1 | `AcknowledgeRevealUseCase(INITIAL)`: factor + set + `enrolledAt` in one transaction | — |
| RF-5.2 | `TotpFactor` exists → plan 001 RF-5.1 challenges | — |
| RF-5.3 | Next C10 derivation reads `hasActiveFactor = true` | Navigate to `returnTo`; the tenant route re-requests |
| RF-5.4 | — | Navigate to `/account/security`; `useSecurityMfa` refetched |
| RF-5.5 | Template `enrollment-completed` via `SecurityNotificationPort` | — |
| RF-5.6 | Use case touches identity tables and the outbox only | No navigation into a tenant without `returnTo` |
| RF-6.1 | Partial unique index; `StartEnrollmentUseCase` supersedes | — |
| RF-6.2 | `expiresAt` checked under lock in every use case; `ExpirePendingEnrollmentsJob` | — |
| RF-6.3 | `discard` deletes seed and hashes; factor and set untouched | Holders cleared |
| RF-6.4 | `FactorStatusPort` ignores pending rows; plan 001 reads `TotpFactor` only | — |
| RF-7.1 | — | `SecuritySettingsPage` heading, subtitle, four areas |
| RF-7.2 | `RequirementReasonPort` | `MfaCard` requirement line; `SecurityStatements` (cannot be disabled, never removal first) |
| RF-7.3 | `SecurityMfaQuery` metadata | `MfaCard` enrolled state |
| RF-7.4 | `SecurityMfaQuery` no-factor state | `MfaCard` not-enrolled; `RecoveryCodesCard` without actions |
| RF-7.5 | — | `RecoveryCodesCard` statements and actions |
| RF-7.6 | `unusedCount` | Count as text; zero-codes inline warning |
| RF-7.7 | Query returns no seed, code, material or consumed values | Nothing else rendered |
| RF-7.8 | Endpoints resolve the caller's identity only; no identity parameter exists | No other identity reachable |
| RF-8.1 | — | `RegenerateCodesDialog` |
| RF-8.2 | `StepUpUseCase` verifies password and TOTP code; the code never reaches `RecoveryCodeSet` | Both fields required |
| RF-8.3 | `StepUpChallenge(REGENERATE)` with expiry and attempts; watermark on the active factor; one generic problem; both throttle keys | `GenericFailureMessage` |
| RF-8.4 | Zero or expiry → `VOIDED`, `mfa.step-up.void` | Dialog closes; nothing changed |
| RF-8.5 | Success → `PendingEnrollment(REGENERATE)` in `VERIFIED`; `OpenRevealUseCase` | `RecoveryCodesReveal` with the replacement heading |
| RF-8.6 | `AcknowledgeRevealUseCase(REGENERATE)`: `invalidateAll` + new set in one transaction | — |
| RF-8.7 | Abandoned reveal → `discard`; current set untouched | Leave → discard |
| RF-8.8 | No call made before step-up submission | "Keep existing codes" closes the dialog |
| RF-8.9 | `REGENERATE` branch never touches `TotpFactor` or sessions; template `recovery-codes-regenerated` | — |
| RF-9.1 | — | `ReplaceAuthenticatorDialog` |
| RF-9.2 | `StepUpChallenge(REPLACE)` as RF-8.2–8.4 | Same dialog mechanics |
| RF-9.3 | Success → `StartEnrollmentUseCase(REPLACEMENT)`; setup material in the step-up response | Wizard opens at step 1 with the replacement heading |
| RF-9.4 | `TotpFactor` unchanged until acknowledge; plan 001 reads only `TotpFactor` | — |
| RF-9.5 | `discard` leaves factor and set; no revocation, no template | Leave → discard |
| RF-9.6 | `AcknowledgeRevealUseCase(REPLACEMENT)`: `replaceWith`, `invalidateAll`, new set in one transaction | — |
| RF-9.7 | `RevokeSessionsPort.revokeAll(except = owningSessionId)`; `bumpSecurityVersion`; template `authenticator-replaced` | Current session continues |
| RF-9.8 | Session service never touches run-scoped authority (A-39); no intent read | — |
| RF-9.9 | No call; controlled recovery (spec 005) re-enters `StartEnrollmentUseCase(REPLACEMENT)` with its own verification in place of the step-up | Route to controlled recovery; dialog closed |
| RF-9.10 | No call before step-up submission | "Keep current factor" closes the dialog |
| RF-10.1 | No endpoint, use case or tenant setting writes another identity's factor or set; `MfaController` resolves the caller only | No control rendered |
| RF-10.2 | `EnrollmentGate`: Owner / Admin always `REQUIRED` without a factor; policy only adds `MEMBER_POLICY` | — |
| RF-10.3 | Plan 004 `ChangeCredential` touches no factor or set | — |
| RF-10.4 | No device-trust state exists **[001]** | Statement in the Sessions area (007) |
| RF-10.5 | Use cases change identity rows only; suspension, membership and erasure untouched | — |
| RF-11.1 | Audit adapter events; `access` gate event | — |
| RF-11.2 | Audit payload shape; problem details exclude inputs; templates carry no secret | No secret in state, URL, storage or analytics |
| RF-11.3 | `SeedCipher` **[001]**; seed never serialised after the start response | Setup holder dropped on unmount |
| RF-11.4 | Templates to the verified address through the outbox | — |
| RF-11.5 | Responses carry committed state; idempotency-keyed outcomes on verify, acknowledge, step-up | Keys generated once per submission |
| RF-12.1 | — | Reflow at tablet and mobile widths; `SecurityStatements` shared |
| RF-12.2 | — | `shared/ui` targets, labels, focus |
| RF-12.3 | — | `StepIndicator` text, associated errors, polite `LiveRegion` |
| RF-12.4 | `setup.key` in the start response | Copyable key beside the SVG |
| RF-12.5 | Locale header selects the template language | i18n bundles; switch changes display only |

---

## 4. Technical decisions

Each decision states what, why, what was discarded, and the requirements it serves.

**D1 · One `PendingEnrollment` aggregate with three purposes, one per identity, locked on every mutation.**
*Why.* Initial enrollment, replacement and regeneration are the same three-step shape with different completion effects (RF-5.1, RF-8.6, RF-9.6); one aggregate with a `purpose` and a state machine keeps the step rules (verify before reveal, reveal before acknowledge, one reveal) in a single place. A row can be locked (`FOR UPDATE`) so two tabs cannot both succeed; a partial unique index makes "at most one pending per identity" a database fact (RF-6.1, LC-8); the recorded outcome per idempotency key gives LC-11 without a second code path.
*Discarded.* Three aggregates — three copies of the same state machine. A stateless signed token carrying the step — cannot be voided, superseded or locked. Keeping the pending material in the HTTP session — not lockable, not auditable, not consistent across instances.
*Covers.* RF-2.1, RF-3.2, RF-4.6, RF-6.1–6.4, RF-9.3, RNF-3, RNF-7, LC-1–LC-8, LC-11.

**D2 · The seed is encrypted under `SeedCipher` [001] from the moment it is generated; the setup material is returned once in the start response and never re-read.**
*Why.* Verification needs the seed, so it cannot be hashed (plan 001 D6); the key reference keeps it out of the database in clear; returning the provisioning content, the text key and the graphic in the start response only — and offering no endpoint that returns them later — makes "shown exactly once" structural (RF-2.6, RF-3.8, RF-11.3). The pending watermark lives on the same row, so the wizard has replay control from its first code (RF-3.5).
*Discarded.* Generating the seed on the client — the server could not verify what it did not create, and the seed would transit twice. A "show setup again" action — rejected by Q3.
*Covers.* RF-2.1, RF-2.2, RF-2.6, RF-3.5, RF-3.8, RF-11.3, RNF-1.

**D3 · Recovery codes are generated when the reveal opens, returned once, and stored only as maintained-encoder hashes; the reveal is a one-shot state transition.**
*Why.* FR-IDN-028 requires hashing at rest; generating the codes at reveal time means plaintext exists only inside the one response, and the row holds hashes from the first write. The set "created with the pending enrollment" (RF-2.1) is the set the pending enrollment owns; its values materialise at the reveal. `VERIFIED → REVEALED` happens once; a second reveal call on the same row, or any later call after a reload, discards the enrollment (RF-4.6), so no path shows a set twice (RNF-2). The maintained encoder gives the work factor a password-equivalent secret needs (plan 001 D5).
*Discarded.* Generating the codes at step 1 and holding them encrypted until step 3 — recoverable plaintext at rest for the whole pending lifetime, contrary to "hashed at rest". Storing the plain set until acknowledgement — F-X-030. A fast hash — brute-forceable at rest.
*Covers.* RF-4.1, RF-4.2, RF-4.6, RF-4.8, RF-8.5, RNF-1, RNF-2, LC-3, LC-4, LC-9.

**D4 · Activation, regeneration and replacement are each one transaction inside `AcknowledgeRevealUseCase`, and replacement bumps the identity's `securityVersion`.**
*Why.* RNF-3 forbids any observable state with a factor and no valid set, two factors or two sets; one transaction that writes the new factor, invalidates the old seed, invalidates the old set and inserts the new hashes gives that by construction, and the row lock serialises concurrent acknowledgements (LC-11). The version bump on replacement makes every open login transaction of the identity void itself at its next use (plan 001 D1, `FACTOR_CHANGED`) and lets the session service refuse already-issued credentials, complementing the explicit revocation of D11.
*Discarded.* Two-phase activation (activate the factor at verification, codes at acknowledgement) — a factor without acknowledged codes, rejected by Q3. Asynchronous invalidation of the old set — a window with two valid sets.
*Covers.* RF-4.5, RF-5.1, RF-5.2, RF-8.6, RF-9.6, RNF-3, RNF-4, LC-10, LC-11, LC-19.

**D5 · The step-up is its own bounded, purpose-bound challenge row, consumed by the pending enrollment it creates, and counted on the sign-in throttle keys.**
*Why.* RF-8.3 binds the step-up to identity, purpose, expiry and attempt limit; a row with those fields and a lock is the same proven pattern as the login transaction. Creating the pending enrollment in the step-up's own transaction and recording `consumedByPendingId` means a succeeded step-up can never be replayed for another action (RNF-5). Password plus code is what sign-in protects, so failures are written on `ACCOUNT` and `NETWORK` [001] and the generic problem matches plan 001 D2.
*Discarded.* A reusable "re-authenticated until" flag on the session — a bypass window usable for either purpose. A short-lived signed grant returned to the client — replayable within its lifetime and unbounded in attempts. Password-only or code-only — rejected by Q4.
*Covers.* RF-8.1–8.5, RF-9.1–9.3, RNF-5, LC-13–LC-15.

**D6 · The step-up code field is routed only to `TotpVerifier`, never to `RecoveryCodeSet.consume`.**
*Why.* RF-8.2 and Q4 refuse a recovery code in the dialogs; the simplest guarantee is that the recovery-code path is not wired there at all, so a recovery code fails exactly as a wrong authenticator code (LC-14) and no branch can be flipped by policy.
*Discarded.* Accepting a recovery code with a warning — rejected by Q4; it would also let a found code replace a factor.
*Covers.* RF-8.2, LC-14.

**D7 · The enrollment gate is a pure rule evaluated inside the `access` derivation of C10, fed by `FactorStatusPort`; workspace creation checks the same port.**
*Why.* Authority is derived at every effect boundary by `access` (F-X-01, P3); withholding Owner / Admin and policy-required Member authority is a derivation outcome, so the rule belongs where C10 is built and runs on every tenant request. Reading the tenant policy at derivation time gives Q12 (a tightened policy applies to the next action) with no push, no forced sign-out and no cache to invalidate. One port, consumed by `access` and by workspace creation, keeps "a pending enrollment is not a factor" (RF-6.4) in one place. The problem `auth.enrollment.required {reason}` lets the FE open the wizard with the right heading without deciding anything.
*Discarded.* Checking the gate at sign-in — Login does not know the tenant (plan 001 RF-8.5 explicitly leaves the gate to tenant entry). A membership status value like `PENDING_MFA` — duplicates a derivable fact and must be kept in sync. A FE-side gate — P3.
*Covers.* RF-1.2–1.6, RF-5.3, RF-6.4, RF-10.2, LC-20–LC-23.

**D8 · Replay control uses a per-seed watermark, on the pending seed during the wizard and on the active factor during step-ups.**
*Why.* Plan 001 D4 already keeps the last accepted time-step per factor; the pending enrollment carries its own watermark so a code observed during setup cannot be replayed within the wizard (RF-3.5, LC-7), and a step-up advances the active factor's watermark so a code used there cannot be reused at sign-in. The tolerance is the single global value (A-32).
*Discarded.* Storing every accepted code — unbounded, same guarantee. No replay control in the wizard — RF-3.5.
*Covers.* RF-3.5, RF-8.3, RNF-5, LC-7.

**D9 · A pending enrollment is bound to the session that started it; discard is idempotent.**
*Why.* Q13 (A-43): a pending enrollment does not survive its session. Storing `owningSessionId` and comparing it on every call means another session, or the same browser after its session ended, cannot resume it; the cleanup job discards rows whose session no longer exists. Making discard idempotent lets the FE call it on every leave path without caring whether the row still exists (RF-2.5, RF-4.6), and lets a reload simply start over: the in-memory holder is gone, the FE sends a best-effort `keepalive` discard on `pagehide`, and if that request does not land the old row can never return codes (any call on it discards, RF-4.6) and is discarded by the next start (`SUPERSEDED`), by the expiry job (D10) or by the end of its session. The discard RF-4.6 requires after a reload is therefore immediate when the best-effort call lands and bounded by the configured expiry otherwise; the set is never shown again in either case.
*Discarded.* Resumable enrollments across sessions — rejected by Q13 and a longer life for an unverified seed. Client-side "are you sure" without a server discard — the row would linger until expiry.
*Covers.* RF-2.5, RF-4.6, RF-6.3, RF-8.7, RF-9.5, LC-1–LC-4, LC-13, LC-16.

**D10 · Expiry is enforced on access and swept by a durable job.**
*Why.* Every use case checks `expiresAt` under lock first, so an expired enrollment or step-up can never act (RF-6.2, LC-5); the existing durable-jobs facility [002 D9] removes expired and orphaned rows in bounded batches and audits each removal, so no seed outlives its window on disk.
*Discarded.* Sweep only — an expired row could act between sweeps. Check-on-access only — expired seeds linger at rest.
*Covers.* RF-6.2, RF-6.3, RF-11.1, LC-5.

**D11 · Replacement revokes other sessions through `RevokeSessionsPort` [004] with `exceptSessionId`; the three notifications are outbox templates and can never revert a change.**
*Why.* Q5 and Q9 (A-39): every other session and refresh family ends, the acting session continues, run-scoped authority is untouched — exactly what the port built for password reset offers, with the exception parameter added. Notifications through the outbox (plan 002 D1) commit with the change, so a provider failure is retried by the relay and the change stands (RNF-9, LC-29); one template per event (Q7, A-37) keeps the rule uniform.
*Discarded.* Revoking the current session too — rejected by Q5. Synchronous mail — timing and failure coupled to the change. Notifying only on replacement — rejected by Q7.
*Covers.* RF-5.5, RF-8.9, RF-9.7, RF-9.8, RF-11.4, RNF-9, LC-18, LC-29.

**D12 · The download file is built on the client from the reveal response; no server endpoint serves the codes a second time.**
*Why.* Q17 fixes the content (product, issue date, one code per line); the FE already holds the codes while the reveal is open, so building the file there adds no second path to the set and keeps RNF-2 structural. The file is generated and offered with a transient object reference revoked after the save (F-X-03 preview rules).
*Discarded.* A `GET …/codes.txt` endpoint — a second, bookmarkable path to the set. Caching the file — storage.
*Covers.* RF-4.3, RNF-2, LC-9.

**D13 · Verify, acknowledge and step-up carry an idempotency key with a recorded outcome [001 D1].**
*Why.* F-X-02 and RF-11.5: a double submit or a lost answer must yield exactly one activation, one set or one revocation (LC-11). The pending and step-up rows record the outcome per key; the use case returns it before any other work.
*Discarded.* Generic POST retry — forbidden. No key — a lost acknowledgement would show "restart" for a completed activation.
*Covers.* RF-11.5, LC-11.

**D14 · The Security Settings area is one read model with the requirement reason supplied by `access`.**
*Why.* RF-7.2 needs to say *why* MFA is required of the viewer, which depends on memberships and tenant policies that `access` owns; a `RequirementReasonPort` returns the reason and the tenants that impose it, and the identity context adds factor metadata and the unused count as a number (RF-7.6, RF-7.7). Nothing by value, nothing about another identity (RF-7.8).
*Discarded.* Reading membership and policy tables from `identity` — a foreign repository. Computing the reason in the FE from the chooser payload — P3.
*Covers.* RF-7.2–7.8, LC-12, LC-30.

**D15 · The setup graphic is an SVG rendered on the server through a `QrEncoder` port; the text key is always present.**
*Why.* RF-2.2 and Q11 require a scannable graphic and a text alternative. Rendering on the server keeps the FE free of a second library and lets one adapter be tested once; an inline SVG carries no external reference and needs no image endpoint. The encoder is ZXing `core` 3.5.3 (Apache License 2.0, no transitive dependencies), decided as A-46 and named in P10; only its encoder and matrix are used, and the port keeps any later change of library inside one adapter.
*Discarded.* A FE QR library — a client dependency outside P10 with the same register question. No graphic — the design draws one and manual entry alone is hostile. Rendering the graphic from the FE's own canvas code — bespoke encoder code to maintain.
*Covers.* RF-2.2, RF-12.4.

**D16 · One wizard component with purpose variants; forced entry through a shared gate handler; the return destination lives in memory.**
*Why.* The three entries differ only in heading and subtitle (RF-1.9, Q19) and in where "Leave setup" and completion return (Q20, RF-5.3); one component with the entry carried in the holder keeps the three steps and every statement identical by construction. Mapping `auth.enrollment.required` in `shared/errors` means every tenant request and workspace creation reach the wizard the same way. `returnTo` is a route the FE itself was on, never a client-claimed authority: the destination re-requests and the server derives authority afresh.
*Discarded.* A wizard per entry — drift. Passing the reason in the URL — forgeable; the heading would lie.
*Covers.* RF-1.2–1.4, RF-1.9, RF-2.5, RF-5.3, RF-5.4, LC-20, LC-22, LC-23.

**D17 · Every configured number is one validated `EnrollmentPolicyProperties` record that fails startup.**
*Why.* P13 and FR-X-046; the two configured families of Q10 (A-40) are two groups of the same record; validation at bind time means the product never runs with a missing or unbounded limit.
*Discarded.* Defaults in code — forbidden. Lazy validation — the first enrollment would fail instead of startup.
*Covers.* RF-3.2, RF-4.2, RF-6.2, RF-8.3, RNF-6.

**D18 · No dedicated mobile pages; every surface reflows, and text comes from feature-scoped i18n bundles [002 D16, D15].**
*Why.* M13 and M14 are the drawn counterparts (Q14, S-28); the other surfaces follow the D01 responsive policy and FR-X-055 requires AA reflow at every width; `SecurityStatements` shared across widths keeps every statement present (RNF-12). The server sends codes only; the locale header selects the template language (RF-12.5).
*Discarded.* Separate mobile components — drift. Server-rendered messages — ties language to the request.
*Covers.* RF-12.1, RF-12.5, RNF-11, RNF-12.

**D19 · Errors are RFC 9457 problem details with safe codes, field paths and a correlation id [001 D12].**
*Why.* F-X-03; one code per class keeps RF-3.6 and RF-8.3 generic; field paths give RF-4.4; the correlation id joins the audit record to the user's report.
*Discarded.* Ad-hoc error bodies.
*Covers.* RF-3.6, RF-4.4, RF-8.3, RF-11.2.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| BE unit | JUnit 5 (**test-only**, via Spring Boot test support) | Pure domain: `PendingEnrollment` state machine, `StepUpChallenge`, `EnrollmentGate`, `RecoveryCodeSet.unusedCount` / `invalidateAll`, `TotpFactor.replaceWith`, watermark arithmetic with a fixed clock. |
| BE integration | Spring Boot test + **real PostgreSQL** (area A verification requires real PostgreSQL; Testcontainers is a **test-only** dependency) | Use cases through real adapters: locks, partial unique index, atomic completion, outbox rows, audit rows, revocation port calls, job sweep, configuration binding. |
| API contract | Spring MockMvc (**test-only**) | Status codes, problem shapes, safe codes, correlation id, absence of seed, material, codes or counts by value in every response after the start and reveal responses. |
| Concurrency | JUnit 5 with real PostgreSQL | Double acknowledge, two-tab start, concurrent step-ups, acknowledge racing a sign-in challenge. |
| Startup | Spring Boot test with invalid property sets | Context refresh fails for every absent or unbounded value. |
| Module boundary | ArchUnit (**test-only**) | The application module depends on no Spring or persistence class and imports no project, grant, approval, run or intent package (the allowed-import DAG of §2.1). |
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | Wizard states and headings per entry, reveal acknowledgement rules, holders cleared on leave, gate handler mapping, download content, nothing in storage. |
| FE e2e + accessibility | Playwright for the browser + axe (the evidence named in F-X-04) — **test-only** | Journeys in EN / ES / FR at desktop, tablet and mobile widths; automated AA; text alternative of the setup material. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Step indicator, focus order, announcements without focus theft, labels, 44 px targets, the copyable key. |

Playwright Java and Cucumber belong to Core and are not used for the FE.

### 5.2 Test matrix

| Spec item | Level | Test case (name) | Proves criterion |
|---|---|---|---|
| RF-1.1, RF-7.4, LC-30 | FE unit, API contract | `mfaCard.notEnrolledOffersSetUpOnly`; `securityMfa.noFactorShape` | 1 |
| RF-1.2, LC-22 | BE integration, FE e2e | `workspaceCreation.blockedWithoutFactor` (spec 012 stub using `FactorStatusPort`); `wizard.leaveDuringCreationCreatesNothing` | 3 |
| RF-1.3, RF-1.5, LC-23 | BE integration | `gate.unenrolledOwnerAdminDerivesNoAuthority`; `gate.sessionOrChallengeNeverSubstitutes` | 3 |
| RF-1.4, RF-10.2, LC-20 | BE unit, integration | `enrollmentGate.memberPolicyRequiresAtNextDerivation`; `enrollmentGate.policyNeverRelaxesOwnerAdmin` | 3, 8 |
| RF-1.6, LC-21 | BE integration | `gate.singleFactorSatisfiesEveryTenant`; `start.refusesWhenFactorExists` | 3 |
| RF-1.7, RF-1.8, LC-24, LC-25 | BE integration | `start.unverifiedRoutesToVerification`; `start.restrictedOrErasingRefused` | 2 |
| RF-1.9 | FE unit | `wizard.headingPerEntry` (owner, member policy, voluntary, replacement) | 1 |
| RF-2.1, RF-6.1, LC-8 | Concurrency | `start.supersedesPendingAndEarlierTabIsRefused` | 4 |
| RF-2.2, RF-12.4 | API contract, FE e2e, manual | `start.returnsUriKeyAndSvgOnce`; `a11y.setupKeyUsableWithoutGraphic` | 9, 12 |
| RF-2.3, RF-12.3 | FE unit, e2e + axe | `stepIndicator.currentStepInText` | 12 |
| RF-2.4, RF-7.2, RF-8.1, RF-9.1 | FE unit | `securityStatements.presentOnWizardDialogsAndArea` | 1 |
| RF-2.5, RF-6.3, LC-1 | BE integration, FE unit | `discard.destroysSeedLeavesFactor`; `wizard.leaveSetupDiscardsAndReturnsPerEntry` | 4 |
| RF-2.6, RF-3.8, RF-11.3 | API contract | `noEndpointReturnsSetupAfterStart`; `responses.neverContainSeed` | 9 |
| RF-3.1 | FE unit, BE unit | `codeField.pasteWithSeparators`; `verify.stripsSeparators` | 2 |
| RF-3.2, RF-6.2, LC-5 | BE integration | `verify.boundToExpiryAndAttempts`; `verify.expiredRefusesAndRestarts` | 4 |
| RF-3.3 | FE unit | `verifyStep.inProgressDisablesResubmit` | 1 |
| RF-3.4 | BE integration | `verify.validCodeAdvancesWithoutActivating` | 4 |
| RF-3.5, LC-7 | BE unit, integration | `verify.rejectsReplayAtOrBelowPendingWatermark` | 4 |
| RF-3.6 | API contract | `codeRejected.carriesNoCauseOrCount` | 4 |
| RF-3.7, LC-2, LC-6 | BE integration, FE unit | `verify.lastWrongAttemptDiscardsAndRestarts`; `wizard.restartShowsNewMaterial` | 4 |
| RF-4.1, RF-4.6, RNF-2, LC-3, LC-4 | BE integration, Concurrency | `reveal.returnsCodesExactlyOnce`; `reveal.secondOpenDiscards`; `reveal.reloadNeverReshows` | 5 |
| RF-4.2, RF-3.2, RF-8.3, RNF-6 | Startup | `enrollmentPolicy.startupFailsOnAbsentOrUnboundedValue` (one case per property) | 10 |
| RF-4.3, LC-9 | FE unit | `download.contentPerQ17`; `download.doesNotAcknowledge` | 5 |
| RF-4.4, RF-4.5, LC-10 | FE unit, API contract | `reveal.finishDisabledUntilTicked`; `acknowledge.refusesUnticked`; `reveal.finishWithoutDownloadCompletes` | 5 |
| RF-4.7, RNF-1 | FE unit | `storage.containsNothingFromFeature` | 5, 9 |
| RF-4.8 | BE integration | `pendingCodes.storedHashedAndDeletedOnDiscard` | 9 |
| RF-5.1, RNF-3 | BE integration, Concurrency | `acknowledge.initialActivatesFactorAndSetAtomically`; `acknowledge.noObserverSeesFactorWithoutSet` | 6 |
| RF-5.2 | FE e2e | `afterEnrollment.nextSignInChallenges` | 4 |
| RF-5.3 | BE integration, FE e2e | `gate.liftedOnNextDerivationWithoutNewSignIn`; `wizard.returnsToInterruptedDestination` | 3 |
| RF-5.4 | FE e2e | `wizard.voluntaryReturnsToSettingsEnrolled` | 1 |
| RF-5.5, RF-8.9, RF-9.7, RF-11.4 | BE integration | `notifications.threeTemplatesToVerifiedAddressWithoutSecrets` | 7 |
| RF-5.6, RF-9.8, RNF-13 | BE integration | `acknowledge.touchesNoSessionTenantOrRunTable` | 7 |
| RF-6.4 | BE integration | `factorStatus.ignoresPending`; `signIn.pendingEnrollmentIsNotChallenged` | 4 |
| RF-7.1, RF-7.3, RF-7.5 | FE unit | `securitySettingsPage.rendersAreasAndEnrolledCards` | 1 |
| RF-7.6, LC-12 | BE integration, FE unit | `securityMfa.unusedCountAfterLastCodeConsumed`; `recoveryCodesCard.zeroWarningNeverBlocks` | 1 |
| RF-7.7, RF-7.8 | API contract | `securityMfa.exposesMetadataOnly`; `securityMfa.noIdentityParameter` | 8, 9 |
| RF-8.2, LC-14 | BE integration | `stepUp.recoveryCodeRefusedAsWrongCode` | 6 |
| RF-8.3, LC-15 | API contract, BE integration | `stepUp.failureGenericForPasswordCodeOrBoth`; `stepUp.countsOnBothThrottleKeys` | 6 |
| RF-8.4 | BE integration, FE unit | `stepUp.limitOrExpiryVoidsNothingChanged`; `dialog.voidCloses` | 6 |
| RF-8.5, RF-8.6 | BE integration | `regenerate.stepUpOpensRevealAndAcknowledgeSwapsSets` | 6 |
| RF-8.7, LC-13 | BE integration | `regenerate.abandonedRevealLeavesPreviousSetValid` | 5, 6 |
| RF-8.8, RF-9.10 | FE unit | `dialogs.keepActionsCallNothing` | 6 |
| RF-9.2, RF-9.3 | BE integration | `replace.stepUpStartsReplacementEnrollment` | 6 |
| RF-9.4, LC-17 | BE integration | `signIn.duringPendingReplacementUsesOldFactor` | 6 |
| RF-9.5, LC-16 | BE integration | `replace.abandonedLeavesOldFactorNoRevocationNoMail` | 6 |
| RF-9.6, LC-19 | BE integration | `replace.acknowledgeSwapsFactorAndSetsAtomically`; `signIn.oldAuthenticatorRefusedAfterReplacement` | 6 |
| RF-9.7, RF-9.8, LC-18 | BE integration | `replace.revokesOthersKeepsCurrentLeavesRunAuthority` | 7 |
| RF-9.9, LC-27, LC-28 | FE e2e, BE integration | `replaceDialog.lostAccessRoutesWithoutChange`; `recovery.entersReplacementEnrollmentDirectly` (spec 005 stub) | 6 |
| RF-10.1, RF-10.5, RNF-8 | API contract, BE integration | `noAdminOrTenantPathTouchesAnotherFactor`; `feature.neverLiftsSuspensionMembershipOrErasure` | 8 |
| RF-10.3, LC-26 | BE integration | `passwordReset.leavesFactorAndCodesUntouched` (plan 004 `ChangeCredential`) | 8 |
| RF-10.4 | API contract | `auth.noDeviceTrustSurfaceExists` **[001]** | 8 |
| RF-11.1, RF-11.2 | BE integration | `audit.entryPerEvent`; `audit.containsNoSecret`; `logs.containNoSecret` | 11 |
| RF-11.5, LC-11 | Concurrency | `acknowledge.doubleSubmitActivatesOnce`; `verify.resubmitReturnsRecordedOutcome`; `stepUp.resubmitReturnsRecordedOutcome` | 6 |
| RF-12.1, RF-12.2, RNF-10, RNF-12 | FE e2e + axe, manual | `a11y.enrollmentDialogsSettings.{en,es,fr}.{desktop,tablet,mobile}`; manual NVDA / VoiceOver checklist | 12 |
| RF-12.5, RNF-11 | FE e2e | `languageSwitch.changesDisplayOnly` | 12 |
| RNF-4 | BE integration | `replacement.identityNeverWithoutFactor` (polling observer during the swap) | 6 |
| RNF-5 | BE integration | `stepUp.cannotBeReusedForAnotherPurpose`; `codes.acceptedOnceOnly` | 6 |
| RNF-7 | BE integration | `failClosed.absentExpiredSupersededUnverifiedRefuses` | 2 |
| RNF-9, LC-29 | BE integration | `notification.failureNeverRevertsChange` | 7 |
| US-12 | all adversarial cases above | seed exposure, replay, reuse, double reveal, unverified activation, admin backdoor suites | 1 |

### 5.3 Not tested here
Sign-in and the challenge (plan 001), verification (plan 002), password reset (plan 004), controlled recovery (005), session issue, refresh, listing and revocation mechanics (006, 007), workspace creation beyond its precondition (012), tenant policy editing (F-SET-05). Their stubs return fixed answers; `RevokeSessionsPort` and `CurrentSessionPort` are verified by recording stubs and, in the integration suite, by the session service's test double.

### 5.4 Exit criteria
Specification §7 items 1–13 map to the "Proves criterion" column; item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

The specification's own open doubts were closed as decisions Q7–Q20 and recorded in Figma as **A-37…A-43** and **S-28…S-34**; this plan implements each as decided (notification events D11; invalidation at acknowledgement D4; revocation scope D11; two configuration families D17; text key D2 / D15; next-derivation gate D7; session-bound pending enrollment D9; reflow D18; not-enrolled card, count and warning D14; download content D12; step-2 copy D19 / i18n; headings D16; "Leave setup" destinations D16). The items below remain open. Of them, A-27 and A-32 are the two the specification names in its completion criterion 13; A-35 and A-39 (the port's contract for all callers) are plan-level items that affect the HOW only and are reviewed together with them before the feature is declared done (§5.4).

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-27 identity statuses | `StartEnrollmentUseCase` accepts `ACTIVE` only; `UNVERIFIED` takes the verification handoff; `RESTRICTED` and `ERASING` are refused. | Change the predicate only. |
| A-32 time tolerance | The single global value of `LoginPolicy` [001], used for the pending seed and the active factor. | A per-identity value would move to `TotpFactor` and `PendingEnrollment`. |
| A-35 persistence tooling | Adapters and migrations against the placeholder API and tool. | Rename base types and migration format; no use case changes. |
| A-39 revocation scope | `RevokeSessionsPort.revokeAll(identity, except)` revokes every other session and refresh family; run-scoped authority untouched (decided for this feature by Q9; the port's contract for all callers is the open part). | Extend the port with the decided scope. |
| **A-46** QR encoding library (raised by this plan; **decided 2 November 2026**) | ZXing `core` 3.5.3 behind the `QrEncoder` port, one BE adapter rendering an inline SVG (D15); recorded in P10 and in the dependency inventory (A-18). | Not applicable: decided. A later change of library touches only the adapter. |

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.9) | `EnrollmentGate`, `FactorStatusPort`, `StartEnrollmentUseCase`, `EnrollmentGateHandler`, `MfaCard`, wizard headings | D7, D16 | RF-1 rows |
| RF-2 (2.1–2.6) | `StartEnrollmentUseCase`, `SeedCipher`, `QrEncoder`, `SetupStep`, `StepIndicator`, `SecurityStatements`, `DiscardPendingEnrollmentUseCase` | D1, D2, D9, D15, D16 | RF-2 rows |
| RF-3 (3.1–3.8) | `VerifyEnrollmentCodeUseCase`, `PendingEnrollment`, `TotpVerifier`, `CodeField`, `VerifyStep`, `GenericFailureMessage` | D1, D8, D13, D19 | RF-3 rows |
| RF-4 (4.1–4.8) | `OpenRevealUseCase`, `AcknowledgeRevealUseCase`, `RecoveryCodesReveal`, reveal holder, client download | D3, D4, D12, D13 | RF-4 rows |
| RF-5 (5.1–5.6) | `AcknowledgeRevealUseCase(INITIAL)`, `FactorStatusPort`, outbox template, `returnTo` | D4, D7, D11, D16 | RF-5 rows |
| RF-6 (6.1–6.4) | `PendingEnrollment`, partial unique index, `ExpirePendingEnrollmentsJob`, `FactorStatusPort` | D1, D9, D10 | RF-6 rows |
| RF-7 (7.1–7.8) | `SecurityMfaQuery`, `RequirementReasonPort`, `SecuritySettingsPage`, `MfaCard`, `RecoveryCodesCard` | D14 | RF-7 rows |
| RF-8 (8.1–8.9) | `StepUpUseCase`, `StepUpChallenge`, `RegenerateCodesDialog`, `OpenRevealUseCase`, `AcknowledgeRevealUseCase(REGENERATE)` | D3, D4, D5, D6, D11 | RF-8 rows |
| RF-9 (9.1–9.10) | `StepUpUseCase`, `StartEnrollmentUseCase(REPLACEMENT)`, `AcknowledgeRevealUseCase(REPLACEMENT)`, `RevokeSessionsPort`, `ReplaceAuthenticatorDialog` | D4, D5, D6, D9, D11 | RF-9 rows |
| RF-10 (10.1–10.5) | `MfaController` caller-only resolution, `EnrollmentGate`, plan 004 `ChangeCredential`, no device trust | D6, D7 | RF-10 rows |
| RF-11 (11.1–11.5) | Audit adapter, `SeedCipher`, outbox templates, idempotency outcomes | D2, D3, D11, D13, D19 | RF-11 rows |
| RF-12 (12.1–12.5) | Responsive components, `StepIndicator`, text key, i18n bundles | D15, D18 | RF-12 rows |
| RNF-1 | `SeedCipher`, hashed codes, holders, audit payload | D2, D3 | RF-2.6, RF-4.7, RF-11 rows |
| RNF-2 | one-shot reveal, client download | D3, D12 | RF-4.1 row |
| RNF-3 | one transaction per completion | D4 | RF-5.1, RF-8.6, RF-9.6 rows |
| RNF-4 | `replaceWith` in the same transaction | D4 | `replacement.identityNeverWithoutFactor` |
| RNF-5 | watermarks, consumed step-up | D5, D8 | RNF-5 row |
| RNF-6 | `EnrollmentPolicyProperties` | D17 | startup |
| RNF-7 | check-on-access, locks | D1, D10 | `failClosed` |
| RNF-8 | caller-only endpoints, gate rule, plan 004 scope | D6, D7 | RF-10 rows |
| RNF-9 | outbox templates | D11 | LC-29 |
| RNF-10, RNF-12 | `shared/ui`, reflow | D18 | RF-12 rows |
| RNF-11 | i18n bundles, locale header | D18 | `languageSwitch` |
| RNF-13 | use-case scope | D4, D11 | RF-5.6 row |
| LC-1 … LC-30 | as traced in §3 through their RF | as above | each LC named in §5.2 (LC-1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30) |

Self-check: every RF criterion (83), every RNF (13) and every limit case (30) of `specEnrollment.md` v0.2 appears in §3 or §5 and in this table; every element of §2 and every decision of §4 names at least one requirement.
