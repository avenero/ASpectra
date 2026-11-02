# ASpectra — Functional requirements

**Version** 2.1 · **Date** 2 November 2026 · **Status** Living specification — the complete functional definition of the product · **Changes** v2.1: FR-IDN-027 reworded (the setup key is shown once in enrollment step 1), FR-X-050 extended to the counterparts M10–M31, table J relabelled as derivation provenance, the authority pointer re-pointed to `constitution.md` P16.1; the `GAP-*` source tags are kept as the provenance of 23 October 2026 (those sections were consolidated into boards 1–7 and register `122:2` §15 on 1–2 November 2026)

This document specifies the functional requirements of **the whole ASpectra product**, across nine feature areas and one cross-cutting section:

| | Area | Prefix |
|---|---|---|
| — | Cross-cutting requirements | `FR-X` |
| A | Authentication and tenancy | `FR-IDN` |
| B | Project and suite structure | `FR-PRJ` |
| C | Step catalog and library | `FR-CAT` |
| D | Authoring and review flow | `FR-AUT` |
| E | Approval path | `FR-APR` |
| F | Settings | `FR-SET` |
| G | Operator-side contracts | `FR-OPS` |
| H | Execution, evidence and delivery | `FR-RUN` |
| I | Extended capabilities | `FR-EXT` |

Which of these requirements form the implementation baseline is decided in **`constitution.md`**, which references them by identifier. This document does not describe releases, status or open work.

---

## 0. How to read this document

### 0.1 Source and authority

Every requirement is derived from the ASpectra design file (Figma `DpfPk8xDraLcDzBhMYwDTo`).


### 0.2 Requirement language

- **MUST** — mandatory; its absence is a defect.
- **MUST NOT** — prohibited; its presence is a defect, and in most cases a security defect.
- **MUST, configured** — mandatory behaviour whose numeric value is supplied by validated configuration. **No numeric default may be invented.** Where this document shows a number, it is illustrative and taken from a design fixture.

### 0.3 What this document covers

Every capability of the product is specified here — including those delivered after the first release, which area I gathers in one place so the core areas read cleanly. Capabilities ASpectra deliberately does not have are not requirements; they are product boundaries, recorded in `scope.md` §6.

### 0.4 The invariants that govern every requirement

These are not features. They are the properties the product exists to guarantee, and every requirement below is subordinate to them.

| | Invariant |
|---|---|
| **I1** | **Acceptance ≠ approval ≠ Run.** Semantic acceptance (C11) creates a revision. Approval (C04) authorises exact bytes. Run (C05) is a separate explicit human command. None implies the next. |
| **I2** | **AI proposes. Humans authorize. Core decides.** No AI path may approve, launch, mutate a revision or reach a runner. |
| **I3** | **The browser expresses intent, never authority.** A tenant or project ID from a client payload is a selector. Authority is derived server-side at every effect boundary (C10). |
| **I4** | **Only the platform catalog (C01) is global.** All customer data — sources, revisions, conversations, assets, results, credentials, indexes — is tenant and project scoped. |
| **I5** | **Pure compilation has no side effects.** C03 contacts no target, database, device or provider, performs no probe, and admits no wall-clock value into its hash. |
| **I6** | **Fail closed.** Missing, invalid, revoked, expired or unconfigured input blocks the dependent effect. No default is assumed, no value is invented, no fallback is attempted. |
| **I7** | **Approved bytes are the executed bytes.** Any change to an execution-affecting pin invalidates the affected approval. Pins are never silently upgraded. |
| **I8** | **No fabricated outcome.** Unknown is never PASS and never a failed assertion. Not-executed is not failure. |
| **I9** | **Verification is part of the requirement.** A requirement is met only when its verification passes. Every area ends with the verification it requires before release, and no document — this one included — is evidence that a test has passed. |

---

## 1. Cross-cutting requirements

These apply to **every** requirement in sections A–G. They are stated once and referenced, not repeated.

### 1.1 C10 — trusted actor and tenant context

**FR-X-001** · Every protected operation MUST resolve a server-derived `C10 TrustedActorTenantContext` before any effect, carrying: issuer + subject, `actorId`, `tenantId`, `membershipId`, `projectScope`, `permissionVersion`, and correlation / job / run identifiers as relevant.
*Source: register §4; board 1 §01; GAP-BE.*

**FR-X-002** · A tenant or project identifier supplied in a URL, request body or JWT claim MUST be treated as a **requested scope**, never as authorization. Route-versus-body ownership mismatch, or missing context, MUST deny safely.
*Source: register §4; GAP-BE; board 3.*

**FR-X-003** · Each transaction, each asynchronous job, each `REQUIRES_NEW` boundary and each retry MUST establish C10 afresh. A pooled connection reused after error or cancellation MUST NOT inherit another scope.
*Source: register §4; GAP-BE.*

**FR-X-004** · Authority MUST be rechecked at every effect boundary — not only at login. In scope: retrieval, provider submission, semantic acceptance, save, approval, launch, cancellation, download, delivery and export.
*Source: board 1 §05; GAP-FAILURE.*

**FR-X-005** · Sensitive writes MUST use a transactional `permissionVersion` check so a stale authorization cannot commit.
*Source: register §4.*

**FR-X-006** · Core MUST receive only a minimal **opaque** execution context (opaque tenant / project / run / attempt / submission / correlation IDs, policy reference, bundle digest, lease fence). Core MUST NOT receive IAM semantics, role names, a user directory or a tenant database connection.
*Source: GAP-CORE; board 2 §02; register §11.*

### 1.2 Database isolation

**FR-X-010** · Every customer-owned row MUST carry `tenant_id`; every project-scoped row MUST also carry `project_id`. **Composite** foreign keys and unique keys MUST enforce ownership, so a cross-tenant or cross-project parent reference fails in PostgreSQL even when an application check is missed.
*Source: register §4; board 1 §03; GAP-BE.*

**FR-X-011** · PostgreSQL `FORCE RLS` MUST be enabled with both `USING` and `WITH CHECK` predicates. The application runtime role MUST be non-owner, non-superuser, without `BYPASSRLS`, without migration privilege and without escalation. Missing context MUST fail closed.
*Source: register §4; board 1 §05.*

**FR-X-012** · Tenant context MUST be established with `SET LOCAL` **inside** the transaction that reads or writes. Session-level tenant state MUST NOT be used. Lazy access after transaction close MUST NOT occur. A transaction-pooling proxy MUST keep context and queries in one transaction. `LISTEN` connections MUST carry no customer context.
*Source: register §4.*

**FR-X-013** · RLS and runtime authorization are **two independent mandatory gates**. Neither substitutes for the other. RLS supplements application permissions; it cannot defend against arbitrary SQL from a compromised application.
*Source: board 1 §05; register §4.*

**FR-X-014** · Schedulers and background workers MUST enumerate only tenant and job **metadata** through a narrow control plane, then claim work in tenant scope. A narrow scheduler cursor is not an RLS bypass and MUST NOT read customer tables.
*Source: register §4; GAP-BE.*

**FR-X-015** · `SECURITY DEFINER` functions and views MUST be audited.
*Source: register §4.*

### 1.3 Idempotency and request integrity

**FR-X-020** · Mutating operations MUST be idempotent on the tuple `(tenantId, projectId, operation, idempotencyKey, bodyHash)`. Same key + same body MUST yield exactly one acceptance and return the same result. Same key + **different** body MUST be rejected as a conflict.
*Source: register §4, §8; board 1 §05.*

**FR-X-021** · Request context MUST be frozen per operation across retries and token refresh. A tenant switch MUST NOT retarget an in-flight save or command.
*Source: register §10; board 3 §02.*

**FR-X-022** · A lost acknowledgement MUST trigger a **lookup of the original submission**, never a blind resubmission.
*Source: register §8; board 1; GAP-FAILURE.*

**FR-X-023** · Safe reads MAY retry with bounds. Consequential writes MUST NOT use generic auto-retry; they require an established server idempotency / operation-status contract. A generic POST replay queue MUST NOT exist.
*Source: register "Session expiry" completion; board 3.*

### 1.4 Secrets, redaction and safe problems

**FR-X-030** · Passwords, OTPs, MFA seeds, recovery codes, reset tokens and raw reset links MUST NEVER be logged, exported or persisted in plaintext.
*Source: register identity completion; board 1.*

**FR-X-031** · Credentials MUST be stored only as references in the existing encrypted credential repository with external master-key injection and key IDs. The client MUST receive write-only status and references, never plaintext. No new secrets vault is mandated.
*Source: register §9; board 1 §06.*

**FR-X-032** · Redaction MUST occur **before** content enters a prompt, log, index, spool, report or export. Sensitive literals in authored text MUST be extracted to `SecretRef` and the transformation MUST be shown to the author.
*Source: register §6, §9; board 1 §06.*

**FR-X-033** · Errors surfaced to a client MUST use safe problem codes with field paths and a correlation ID. Raw transport payloads, secrets and prompts MUST NOT appear. Problem codes MUST distinguish at minimum: tenant denial, stale revision, missing mapping, policy change, AI quota, uncertain acceptance and expired evidence.
*Source: register §11; board 3 §07.*

**FR-X-034** · Browser storage MUST hold only an allowlist of harmless preferences. Tokens, drafts, prompts, query cache, artifact payloads and provider secrets MUST NOT be persisted to `localStorage` or `sessionStorage`.
*Source: board 3 §02; GAP-FE.*

**FR-X-035** · Artifact and source preview MUST NOT inline untrusted HTML or SVG. Preview MUST validate MIME, scheme, filename and size, use a safe download or sandbox boundary, and revoke blob URLs.
*Source: board 3 §07; N08.*

### 1.5 Accessibility, language and audit

**FR-X-040** · **WCAG 2.2 AA on critical journeys is a release bar.** Evidence MUST comprise automated axe + Playwright **and** manual assistive testing: NVDA on Chrome and Firefox (Windows) and VoiceOver on Safari (macOS and iOS). Actual versions MUST be pinned in a release-support matrix. A critical AA failure MUST gate handoff.
*Source: register §13.1 finding 11; GAP-FE.*

**FR-X-041** · All ordering and reordering MUST be keyboard-operable with retained focus and an **announced new position**. Drag-only ordering MUST NOT exist anywhere.
*Source: GAP-FE; N36; register §13.1.*

**FR-X-042** · Status MUST be conveyed by text, not colour alone. Field instructions and errors MUST be programmatically associated with their field and carry a focus target. Live announcements MUST be bounded and MUST NOT steal focus.
*Source: board 3 §07; GAP-FE.*

**FR-X-043** · The interface MUST support EN / ES / FR display via i18next. **A UI locale switch MUST NOT** change a Gherkin dialect, translate an executable expression, register an alias, rebind a step or rewrite approved bytes. These four concepts MUST remain separate in the data model and in the UI: UI locale · Gherkin keyword dialect · registered executable expression alias · stable C01 `stepId`.
*Source: board 2 completion; board 3 completion; register §"EN/ES/FR"; R25.*

**FR-X-044** · Security-relevant and business-relevant events MUST be appended to an append-only redacted audit log with actor, object lineage, decision or fact, and retention. Audit visibility MUST follow current authority: tenant Owner or Admin membership alone MUST NOT authorize reading a project audit payload.
*Source: register §5; N18.*

**FR-X-045** · A reliable transactional email adapter is a **mandatory operational prerequisite** for verification, invitations, recovery and security notification. Optional run notifications are not a substitute. No vendor is assumed.
*Source: board 1 completion; register identity completion.*

**FR-X-046** · Retention windows, deadlines, attempt counts, delays, caps and limits MUST be validated configuration. Unsafe missing or unbounded production configuration MUST fail startup or fail admission. No numeric SLO may be invented.
*Source: register §9; board 1 §06; GAP-OPERATOR.*

### 1.6 Responsive surfaces

**FR-X-050** · The product MUST ship **genuine mobile and tablet reflows**, not a scaled desktop layout. Designed counterparts exist for: M01 landing · M02 login · M03 invitation · M04 tenant overview · M05 mapping review · M06 approval review · M07 execution live · M08 execution results · M09 queued run · M10–M31 (signup, verify email, MFA challenge, TOTP enrollment, recovery-codes reveal, forgot / reset / forced-change password, account recovery, invitation pre-match, workspace chooser, workspace suspended, account privacy, session-expiry sheet, guarded-switch sheet, execution queue, queued run detail, admission blocked, delivery jobs, partial-delivery reconcile, system error) · T01 tablet authoring · T02 tablet project dashboard; every other surface follows the D01 responsive policy.
*Source: D03 responsive surfaces; product-experience sections 09 and 11.*

**FR-X-051** · A responsive counterpart MUST preserve **every** safety statement, gate, pin, coverage boundary and invariant of its desktop surface. No warning, gap or boundary MUST be dropped to fit a smaller viewport.
*Source: M05; M06; M08; M09.*

**FR-X-052** · Mobile navigation MUST collapse to a menu plus a bar of permitted destinations **without changing authority**: the destinations shown MUST still follow current server-derived grants.
*Source: M04; FR-X-001.*

**FR-X-053** · Touch targets on responsive surfaces MUST be at least 44 px, with labelled controls and visible focus.
*Source: M09; FR-X-040.*

**FR-X-054** · Tablet and mobile authoring MUST retain **keyboard-operable reordering** with announced position. Drag-only ordering MUST NOT be introduced at any viewport.
*Source: T01; FR-X-041.*

**FR-X-055** · Every critical-journey surface MUST satisfy WCAG 2.2 AA **reflow**, whether or not it has a dedicated responsive counterpart, and a surface with a designed counterpart MUST follow that design.
*Source: GAP-ALIGN finding 11; FR-X-040.*

---

# A · Authentication and tenancy — `FR-IDN`

**Scope.** Global identity, local credentials, MFA, recovery, sessions, invitations, tenant membership, project grants, tenant lifecycle and the tenant switch.

**Screens.** R01 Landing · R02 Login · R03 ForgotPassword · R04 Signup · R05 MfaChallenge · R07 ResetPassword · R08 ForceChangePassword · R38 SecuritySettings · N01 VerifyEmail · N02 InvitationAcceptance · N05 ControlledAccountRecovery · N03 TenantChooser · N04 CreateWorkspace · N19 AccountDeletion · N20 TenantSuspended · R34 SessionExpiryDialog · R39 TOTPEnrollmentWizard · R41 FactorReplacementDialog · R35 / R36 RecoveryCodes dialogs · responsive counterparts M01, M02 and M03.

## A.1 Global identity

**FR-IDN-001** · A `GlobalIdentity` MUST be a single principal identified by a stable `issuer + subject`, holding only: email, display name and status. Unrelated personal data MUST NOT be stored on it. Credentials and MFA MUST NOT be duplicated per tenant.
*Source: register §3; board 1 §03.*

**FR-IDN-002** · Identity MUST be separate from `TenantMembership` and `ProjectGrant`. A single identity MAY hold memberships in several tenants.
*Source: register §3; N03; N19.*

**FR-IDN-003** · Passwords MUST be stored using the existing maintained Spring Security `PasswordEncoder`. Plaintext or reversible storage MUST NOT occur.
*Source: register §3; board 1 completion.*

## A.2 Signup and email verification

**FR-IDN-010** · Signup MUST collect name, email and password only, plus explicit terms acknowledgement. No organization data MUST be required at identity signup, and no activation queue MUST exist.
*Source: R04.*

**FR-IDN-011** · Password policy requirements MUST be supplied by current server policy and displayed; the client MUST NOT hard-code them.
*Source: R04; R07.*

**FR-IDN-012** · Email MUST be verified **before** the identity may create a workspace or accept an invitation. Verification alone MUST grant no tenant or project access.
*Source: R04; N01; board 1 completion.*

**FR-IDN-013** · The verification screen MUST support resend and "use a different email". Resend responses MUST NOT disclose whether an account exists.
*Source: N01.*

**FR-IDN-014** · The verification flow MUST render three distinct states: **pending**, **confirmed**, and **link expired or invalid**. The expired state MUST disclose no account details and MUST offer requesting a new link. Only the newest verification message MUST be usable.
*Source: N01.*

**FR-IDN-015** · Email-domain matching MUST NOT auto-join an identity to a tenant. Identities MUST NOT be auto-linked solely by matching email address; account linking requires verified control and re-authentication.
*Source: register §10, identity completion; board 3 completion.*

## A.3 Sign-in, abuse protection and MFA

**FR-IDN-020** · Sign-in MUST use local verified email and password. Responses for unknown identity, wrong password and restricted account MUST be **generic and comparable**. A dummy maintained-encoder verification MUST be performed for an unknown identity where appropriate.
*Source: R02; register identity completion; board 1 completion.*

**FR-IDN-021** · Abuse protection MUST combine account-aware throttling with IP / network / risk throttling, progressive delays and **bounded temporary** restriction. A permanent lockout an attacker can trigger MUST NOT exist. Abuse limits MUST bound the verification workload.
*Source: board 1 completion; register identity completion.*

**FR-IDN-022** · An account MUST NOT be created, and a submitted password MUST NOT be stored, for an unknown username.
*Source: register identity completion.*

**FR-IDN-023** · **TOTP MUST be mandatory for Owner and Admin**, including a solo founding Owner, before privileged or customer authority is granted. Tenant policy MAY additionally require it for Members or for sensitive actions.
*Source: register §3; GAP-BE; N06; R02.*

**FR-IDN-024** · A first-factor success or an MFA challenge MUST constitute a **limited identity state**, not a C10-authorized customer session.
*Source: R05; register identity completion.*

**FR-IDN-025** · The MFA challenge MUST be bound to identity, login transaction, purpose, expiry and attempt limit, with replay control. It MUST accept a pasted complete code. It MUST offer "use a recovery code" and an entry point to controlled recovery. A remembered-device bypass MUST NOT exist.
*Source: R05.*

**FR-IDN-026** · TOTP enrollment MUST follow three steps — set up, verify code, save recovery codes — and MUST be **confirmed before activation**. Owner / Admin access MUST remain gated until required enrollment completes.
*Source: R39; register identity completion.*

**FR-IDN-027** · The TOTP seed MUST be encrypted with a server key reference. It MUST NOT be one-way hashed (verification requires it). It MUST be shown to the client once only, during enrollment step 1, as the manual-entry alternative to the QR code; it MUST NOT be stored client-side and MUST NOT be retrievable afterwards. After step 1 only enrollment metadata MUST be shown.
*Source: board 1 completion; R38.*

**FR-IDN-028** · Recovery codes MUST be high-entropy, hashed at rest and single-use. They MUST be revealed exactly once, require explicit acknowledgement that they were saved, and MUST NOT be written to browser storage. Regeneration MUST require policy-approved re-authentication and MUST invalidate the entire previous set.
*Source: R35; R36; N05; register identity completion.*

**FR-IDN-029** · Factor replacement MUST require the current factor (plus password per policy) **or** separately controlled recovery. The existing factor MUST remain required until the replacement is verified and recovery acknowledgement completes. There MUST be no silent disabling step. Replacement MUST trigger notification and session revocation.
*Source: R41; R38; board 1 completion.*

**FR-IDN-030** · Required MFA MUST NOT be disabled by any tenant-side action. Tenant settings MAY tighten platform security boundaries, never weaken them.
*Source: R38; N06; register identity completion.*

## A.4 Password reset and controlled recovery

**FR-IDN-040** · Password recovery MUST have **independent** rate limits. Submitting a reset request MUST NOT lock the account. The response MUST be generic whether or not the account exists.
*Source: R03; board 1 completion.*

**FR-IDN-041** · The reset token MUST be cryptographically random and purpose-bound, with server-side storage of its **hash** only, and an expiry. It MUST be delivered only to the verified recovery channel, over a configured trusted HTTPS origin — never a request-controlled `Host` value.
*Source: board 1 completion; register identity completion.*

**FR-IDN-042** · Reset consumption MUST be atomic and single-use. A successful reset MUST: change the credential, revoke refresh families and sessions, invalidate outstanding reset links, send a security notification, and return the user to normal password + MFA sign-in.
*Source: R07; board 1 completion.*

**FR-IDN-043** · A password reset MUST NOT auto-login, restore membership, lift suspension, or **reset, replace or bypass MFA**. This MUST be stated on the reset screen.
*Source: R07; R03; N05.*

**FR-IDN-044** · Used or expired reset links MUST lead to a distinct safe "request a new link" state.
*Source: R07.*

**FR-IDN-045** · A forced password change MUST be supported as its own state, preserving the existing workspace context, requiring the current password, and MUST NOT replay a pending Run request.
*Source: R08.*

**FR-IDN-046** · Controlled account recovery MUST offer three distinct paths: (a) an unused recovery code; (b) a verified recovery **request** when no code is available; (c) continuing an existing recovery case by reference.
*Source: N05.*

**FR-IDN-047** · A recovery case reference alone MUST grant no access. Case details MUST be shown only after separate verification. Identity, ownership and factor-replacement checks MUST remain required. No completion time MUST be promised.
*Source: N05.*

**FR-IDN-048** · Recovery MUST NOT bypass suspension, revoked membership or required MFA. Existing MFA and tenant boundaries MUST remain in place while a case is reviewed.
*Source: N05.*

**FR-IDN-049** · **A tenant Admin MUST NOT reset, replace or disable another identity's password or MFA**, nor erase another identity, because a global identity may be shared with other tenants. There MUST be no administrator backdoor.
*Source: N05; R13; N19; board 1 completion.*

## A.5 Sessions and refresh

**FR-IDN-060** · Access credentials MUST be short-lived and held in memory only. The refresh credential MUST be a `Secure`, `HttpOnly` cookie with `SameSite` appropriate to configured origins, with CSRF and CORS defences configured against actual origins.
*Source: board 3 completion; register identity completion.*

**FR-IDN-061** · The server MUST enforce idle and absolute session deadlines and a current session / security version. Background polling or refresh MUST NOT count as human activity and MUST NOT extend a session indefinitely.
*Source: register identity completion.*

**FR-IDN-062** · The refresh family MUST rotate **atomically** with reuse / replay detection. Accepting an arbitrary recently consumed token MUST NOT be the concurrency strategy; concurrent tabs and ambiguous retry MUST be tested.
*Source: board 3 completion; register identity completion.*

**FR-IDN-063** · The client MUST implement a **single-flight** refresh for eligible control-plane 401 responses only, coalescing concurrent requests, then retry the eligible operation **at most once**, preserving the original identity, tenant, project, body and idempotency context. It MUST NOT reconstruct a different request.
*Source: board 3 completion; R34.*

**FR-IDN-064** · A 403 or access-denial response MUST refresh current permissions or display denied scope. It MUST NOT enter a refresh or login loop. A transport failure MUST NOT be presented as expiry. CSRF errors MUST have their own safe handling.
*Source: board 3 completion.*

**FR-IDN-065** · **An SUT HTTP 401 MUST NEVER refresh ASpectra credentials.** It is target protocol and test data. ASpectra bearer tokens MUST NEVER be attached to a customer-target request, and the console origin MUST NEVER become an SUT target or credential.
*Source: board 3 completion; board 2 completion; GAP-FE; GAP-FAILURE.*

**FR-IDN-066** · On genuine expiry or revocation the client MUST mask or unmount restricted content, clear transient secrets and require sign-in. It MUST offer explicit save **while authorization is still valid**, and MUST NOT silently persist private data after expiry. Any retained in-memory draft MUST be shown only after the same identity and tenant scope is re-authorized; a different login MUST NOT inherit it.
*Source: board 3 completion; R34.*

**FR-IDN-067** · **Session expiry MUST NOT cancel an accepted Run.** An accepted backend execution continues under narrow run-scoped service authority. Re-authentication MUST restore only currently authorized visibility and MUST NOT replay Run intent. Membership or tenant revocation, compromise response and resource deletion are separate policies that MAY block queued work and request bounded stop.
*Source: R34; board 3 completion; register identity completion; GAP-FAILURE.*

**FR-IDN-068** · Explicit logout MUST revoke the session server-side. "Sign out everywhere" MUST revoke identity sessions. Sessions MUST be listable with context and server-supplied expiry, and individually revocable.
*Source: R38; register identity completion.*

**FR-IDN-069** · A session MUST NOT be a trusted device and MUST NOT grant a TOTP bypass. This MUST be stated on the sessions surface.
*Source: R38.*

## A.6 Invitations

**FR-IDN-080** · An invitation MUST be hashed at rest, expiring, **atomically single-use**, and bound to the intended verified email, the tenant and the permitted roles.
*Source: register §3; board 1 §05.*

**FR-IDN-081** · The acceptance screen MUST display, before acceptance: the tenant, the tenant membership role, the project, the explicit project grants, and explicitly whether `canRun` is granted. The invitee MUST confirm understanding that membership and project grants are separate.
*Source: N02.*

**FR-IDN-082** · Acceptance MUST require that the signed-in identity matches the invited address. Switching identity MUST require sign-out; the invitation MUST NOT be accepted for the wrong account.
*Source: N02.*

**FR-IDN-083** · Expired, revoked or already-used invitations MUST NOT be acceptable, and MUST direct the user to request a fresh invitation.
*Source: N02.*

**FR-IDN-084** · An invitation MUST NOT permit role self-escalation. Roles MUST NOT be self-assigned.
*Source: board 1 §05; register §3.*

**FR-IDN-085** · A new identity MUST be able to create a local account with the invited address, verify it, then return to the invitation. Required MFA MUST complete before protected access.
*Source: N02.*

## A.7 Tenant membership and project grants

**FR-IDN-090** · Tenant roles MUST be exactly **Owner, Admin, Member**. Project roles MUST be exactly **Author, Reviewer, Viewer**. `canRun` MUST be a **separate fixed project grant**. No custom role builder and no team hierarchy MUST exist.
*Source: register §3; board 1 §04; GAP-BE; N06.*

**FR-IDN-091** · The authority matrix MUST be implemented exactly as follows.

| Role / grant | Tenant administration | Project content | Approve C04 | Launch C05 |
|---|---|---|---|---|
| **Owner** | Tenant lifecycle, policy, memberships; last-active-Owner protected | Only with explicit project grant | Only with Reviewer grant | Only with `canRun` |
| **Admin** | Delegated administration; cannot evade last-Owner guard | Only with explicit project grant | Only with Reviewer grant | Only with `canRun` |
| **Member** | None | Per project grants | Only with Reviewer grant | Only with `canRun` |
| **Author** | No implied tenant role | Read / draft / author / accept mapping; upload assets; author macros | Not implied; may also hold Reviewer | Not implied |
| **Reviewer** | No implied tenant role | Review and read | Yes — publishes exact revisions | Not implied |
| **Viewer** | No implied tenant role | Authorized read only | No | Not implied |
| **`canRun`** | No implied tenant role | Does not grant author or review access | Independent of approve | Eligible Author/Reviewer + explicit `canRun` + valid C04 + current policy |

*Source: board 1 §04; GAP-BE; R17; N06.*

**FR-IDN-092** · **Tenant administration MUST NOT imply project content access.** A tenant Owner or Admin MUST have no implicit read of any project. This MUST be stated in the membership and policy surfaces.
*Source: R13; N06; R17; GAP-BE.*

**FR-IDN-093** · Membership status MUST be `active | suspended | revoked` and MUST carry `permissionVersion`. Tenant status MUST be `active | suspended | deleting`.
*Source: register §3; board 1 §03.*

**FR-IDN-094** · Revocation MUST deny future commands and subscriptions. In-flight remote work MUST be cancelled at bounded checkpoints where possible, and MUST NOT be presented as undone.
*Source: register §3; GAP-FAILURE.*

**FR-IDN-095** · The **last active Owner** MUST NOT be removable, demotable or suspendable. The implementation MUST lock the tenant ownership row during concurrent owner mutations. An unsafe count-then-write race MUST NOT be used. Ownership transfer MUST recheck current membership at confirmation.
*Source: register §3; board 1 §04; R13; N19.*

**FR-IDN-096** · A **founding Owner**'s grants on the first project — Author, Reviewer and `canRun`, each chosen explicitly at creation as FR-IDN-114 requires — MUST be created through an auditable, transactional bootstrap. These MUST be explicit grant records, never inferred rights; the Owner role implies none of them.
*Source: register §3; N04; GAP-AUTHOR.*

**FR-IDN-097** · Tenant member administration MUST support: invite; list members with tenant role, membership state and administration action; list pending invitations; edit tenant role; transfer ownership; manage or remove membership.
*Source: R13.*

**FR-IDN-098** · Project grant administration MUST allow setting Author / Reviewer / Viewer and `canRun` **independently** per member, MUST display the member's tenant role alongside for contrast, and MUST state that revocation blocks new work while authored evidence and provenance remain.
*Source: R17.*

**FR-IDN-099** · Membership and administration changes MUST be versioned and auditable.
*Source: register §10.*

## A.8 Tenant selection, creation and suspension

**FR-IDN-110** · The tenant chooser MUST list **only** memberships belonging to the signed-in identity, with tenant role, membership state and eligible projects with their grants. Tenants the identity does not belong to MUST NOT be listed.
*Source: N03.*

**FR-IDN-111** · A suspended tenant MUST appear with its access state but MUST NOT expose project names, sources or run details.
*Source: N03; N20.*

**FR-IDN-112** · The chooser MUST state that changing workspace context does not retarget previously accepted runs, which keep their original tenant, project and environment pins.
*Source: N03.*

**FR-IDN-113** · Workspace creation MUST require: verified email, completed TOTP enrollment with recovery codes acknowledged, and a workspace name. Address, phone number and connector MUST NOT be required. If TOTP enrollment is incomplete, creation MUST be blocked — never bypassed.
*Source: N04.*

**FR-IDN-114** · Workspace creation MAY optionally create a first project, and MUST then require **explicitly ticked** Author / Reviewer / `canRun` grants for the founding Owner, with a statement that the Owner role does not grant these automatically.
*Source: N04.*

**FR-IDN-115** · Tenant creation and first-Owner / first-grant bootstrap MUST be a single auditable transaction.
*Source: register §3, identity completion.*

**FR-IDN-116** · Tenant suspension MUST freeze new effects, refresh permissions and subscriptions, and request bounded cancellation of active work. The suspended-tenant screen MUST state that accepted runs do not become failed tests and that this state does not rewrite run verdicts. It MUST offer only the approved verified recovery process, personal account security / sign-out and, where the identity holds an active membership in another active tenant, entry to the tenant chooser.
*Source: N20; register §9; GAP-FAILURE.*

## A.9 Leaving, erasure and export

**FR-IDN-120** · Leaving a tenant and erasing a global identity MUST be presented and implemented as **two different operations**.
*Source: N19; board 1 completion.*

**FR-IDN-121** · Membership removal MUST revoke that tenant's access and grants, and MUST preserve the global identity, other memberships and credentials. It MUST be blocked while the identity is a last Owner, directing to ownership transfer.
*Source: N19; R13; board 1 completion.*

**FR-IDN-122** · Verified global self-erasure MUST: verify the identity independently; require last-Owner transfer in **every** owned tenant; revoke credentials and sessions; minimize or pseudonymize retained attribution subject to lawful retention. It MUST require typed confirmation, MUST NOT be one-click, and MUST promise no instant erasure and no invented retention duration.
*Source: N19; board 1 completion.*

**FR-IDN-123** · The erasure surface MUST show only the signed-in identity's own ownership checks. It MUST NOT expose project content or other customers' data.
*Source: N19.*

**FR-IDN-124** · Export MUST be an asynchronous durable scoped job that **excludes secrets** and obeys project-content grants. Tenant administration alone MUST NOT grant full-data export: whole-organization export requires explicit access to every selected project plus export permission.
*Source: register §3, §9, §10.*

**FR-IDN-125** · Deletion MUST be a durable scoped job that freezes launch / import / AI, revokes grants and connections, cancels or settles work, then purges objects, indexes and rows in policy order, retaining only permitted minimal audit and tombstones. Late callbacks MUST NOT resurrect content. Backups MUST expire by policy; restore MUST apply deletion tombstones before access. No instant backup-erasure MUST be promised.
*Source: register §9; board 1 §06; GAP-FAILURE.*

## A.10 Tenant switch (client state transition)

**FR-IDN-130** · A tenant or project switch MUST be implemented as a guarded state transition in this exact order: (1) dirty-editor guard offering **Save / Discard / Stay**; (2) abort in-flight requests and disconnect STOMP; (3) advance the scope epoch and clear scoped Query cache and Zustand drafts; (4) fetch current membership and grant `permissionVersion`; (5) only then load data or open a stream.
*Source: board 3 §03; GAP-FE.*

**FR-IDN-131** · A save triggered by the dirty guard MUST be submitted under the **original** tenant / project / `baseETag` — never the newly selected tenant.
*Source: board 3 §03; GAP-FE; register §10.*

**FR-IDN-132** · Late responses and events MUST be discarded by epoch check. Cache clearing MUST NOT be treated as authorization: subscriptions, downloads and every request MUST be authorized at runtime, never by payload ID.
*Source: board 3 §03; GAP-FE.*

## A.11 Public landing and unauthenticated entry

**FR-IDN-140** · A public, unauthenticated landing surface MUST exist, presenting the workflow, the capabilities, the human-control boundary and an FAQ, with exactly two entry actions: **create workspace** and **sign in**.
*Source: R01; M01.*

**FR-IDN-141** · The landing MUST state the product boundary rather than imply autonomy: acceptance, approval and Run are separate decisions; AI is optional; a verified email is required; the managed runner is started by a user decision.
*Source: R01.*

**FR-IDN-142** · Every preview, scenario, identifier, count and outcome shown on the landing MUST be labelled **illustrative**, and MUST NOT be presented as an operational result, a benchmark or a coverage claim.
*Source: R01; M01.*

**FR-IDN-143** · The landing MUST answer, in product copy, that importing or approving does not start a run, that AI and connectors are optional, and that changing interface language does not change a test.
*Source: R01; FR-X-043.*

**FR-IDN-144** · The landing MUST offer EN / ES / FR and the privacy, terms and contact routes, and MUST present only identity entry methods that are active. Federation and passkeys are specified in I.2 and I.3.
*Source: R01; M02; register §10.*

**FR-IDN-145** · "Create workspace" from the landing MUST route into verified-email signup (A.2) and then workspace creation. It MUST NEVER create or enter a tenant directly.
*Source: R01; R04; N04.*

## A.12 Area A — verification requirements

Real PostgreSQL cross-tenant read / write / FK / RLS attacks and pooled-context contamination · concurrent last-Owner and grant transitions and authorization races · MFA challenge, enrollment and recovery-code replay · concurrent refresh, reuse detection and multi-tab behaviour · account enumeration, throttle and reset-token races · verified signup and invitation concurrency · membership removal versus global erasure · tenant-switch isolation and stale responses · cookie / CSRF / CORS configuration · tenant STOMP and object-download isolation · deletion, export, tombstone and late-callback safety · email delivery · WCAG 2.2 AA on the access and recovery journeys.
*Source: board 1 §08; board 3 §05; register §12, identity completion.*

---

# B · Project and suite structure — `FR-PRJ`

**Scope.** The ownership tree, project lifecycle and configuration, suites and features, selection, the environment domain object, trust material, integrations configuration and the project dashboard.

**Screens.** R14 Projects · R15 ProjectDetail · R16 ProjectSettings · R17 ProjectMembers · R11 ProjectDashboard · R18 ProjectIntegrations · R19 Environments · R20 CertificateManager · N07 EnvironmentEditor · R21 Suites · R22 SuiteDetail · R23 FeatureDetail · N15 TrashRestore · R10 TenantOverview · R09 Onboarding · responsive counterparts M04 and T02.

## B.1 The ownership tree

**FR-PRJ-001** · The domain hierarchy MUST be `Tenant → Project → Suite → Feature → Scenario → immutable TestRevision`. Every child MUST carry its ownership foreign keys per FR-X-010.
*Source: board 1 §03; register §5; GAP-BE.*

**FR-PRJ-002** · `ExecutionPlan` / selection MUST be a **versioned snapshot**, not a running execution.
*Source: register §5; board 1 §03.*

**FR-PRJ-003** · Reordering MUST preserve `stepOccurrenceID`s. Duplication MUST create new ones.
*Source: register §5; board 3 §03.*

**FR-PRJ-004** · Cross-project reuse MUST copy, or explicitly reference an authorized versioned asset. Silent ownership moves of approved content MUST NOT occur. Cross-project sharing of macros and assets is specified in **I.7**.
*Source: register §5; GAP-AUTHOR; N37.*

## B.2 Projects

**FR-PRJ-010** · The projects list MUST show only projects the identity can access in the current tenant, each with the viewer's own grants, lifecycle state and working-revision readiness.
*Source: R14.*

**FR-PRJ-011** · Project creation MUST require explicit initial grants, performed by Owner or Admin.
*Source: R14; register §3.*

**FR-PRJ-012** · An action unavailable for lack of a grant MUST be **hidden or disabled with the missing grant named in its label or explanation**. A Viewer MUST NOT be able to create revisions, approve or execute.
*Source: R14; board 3 §03.*

**FR-PRJ-013** · Project detail MUST present: suites and features with working revision and readiness; pinned context (selected environment revision, approval state for the current revision, the viewer's grants); and the next explicit decision. It MUST state that an active suite is not execution eligibility.
*Source: R15.*

**FR-PRJ-014** · Archived content MUST block new authoring while remaining readable.
*Source: R14; R21.*

## B.3 Project configuration

**FR-PRJ-020** · Project configuration MUST be saved as a **new versioned configuration revision**, not mutated in place.
*Source: R16.*

**FR-PRJ-021** · Configuration MUST include: project name, default environment selection, fixture set and runtime profile.
*Source: R16.*

**FR-PRJ-022** · A default environment MUST be a **suggestion only**. Every Run MUST still select and pin its environment explicitly. Auto-launch from a default MUST NOT exist.
*Source: R16; board 3 completion.*

**FR-PRJ-023** · The configuration editor MUST display an **impact preview** stating that changing an environment, fixture or runtime dependency requires a new compile and human review for impacted selections, and that current dependent approval becomes stale.
*Source: R16.*

**FR-PRJ-024** · Runtime overrides MUST NOT exist. An automatic failed-once retry MUST NOT be configured or implied.
*Source: R16.*

**FR-PRJ-025** · Saving configuration MUST NOT contact a target. Any connection check MUST be a separate, explicit, authorized action.
*Source: R16; N07; I5.*

## B.4 Suites, features and selection

**FR-PRJ-030** · Suites MUST be listable in **Active / Archived / Trash** views, with features and scope, working state and the next decision, searchable and filterable by readiness.
*Source: R21.*

**FR-PRJ-031** · Suite detail MUST allow selecting **exact scenario revisions** for compilation, with per-row eligibility, and MUST render a selection preview that lists **pre-review exclusions** (unmapped, missing environment, archived) explicitly.
*Source: R22.*

**FR-PRJ-032** · Exclusions MUST be reviewed before approval. A runtime override MUST NOT silently re-include an excluded scenario, and no new scope exclusion MUST appear at Run.
*Source: R22; N12.*

**FR-PRJ-033** · Feature detail MUST show scenarios with revision, linked criteria and readiness, plus a **shared Background as a versioned dependency** with its dependent scenarios.
*Source: R23.*

**FR-PRJ-034** · Editing a Background MUST create a new revision, MUST mark dependent bundles stale, and MUST NOT run them.
*Source: R23.*

**FR-PRJ-035** · An authorized snapshot export MUST be available with revision and source lineage. Export MUST NOT push tests to a repository, approve, or execute.
*Source: R22.*

## B.5 Lifecycle — archive, trash, restore, purge

**FR-PRJ-040** · **Archive, Trash, Restore and Purge MUST be four distinct operations with distinct semantics.** Archive retains inactive content. Trash is reversible soft deletion within a configured window. Purge is separately authorized, policy-driven and irreversible. They MUST NOT be interchangeable delete buttons.
*Source: register identity completion; N15; board 1 completion.*

**FR-PRJ-041** · Deletion of a project, suite, feature or scenario MUST atomically record a tenant-scoped tombstone with `deletionOperationId` and cascade provenance, plus outbox work.
*Source: board 1 completion; register identity completion.*

**FR-PRJ-042** · Parent state MUST **immediately** block descendant active reads, edits, approvals and new runs, before asynchronous cleanup completes.
*Source: board 1 completion; register identity completion.*

**FR-PRJ-043** · Deletion MUST cancel queued work and request bounded stop of active work per policy, preserving known facts. Acknowledgement MUST NOT be presented as instant termination or rollback.
*Source: board 1 completion; GAP-FAILURE.*

**FR-PRJ-044** · Lifecycle authority: whole-project lifecycle MUST require tenant Owner or Admin **plus** an explicit project grant. Suite / feature / scenario content lifecycle MUST require an authorized Author. **Reviewer, Viewer or `canRun` alone MUST NOT authorize deletion.**
*Source: R21; N15; board 1 completion.*

**FR-PRJ-045** · Restore MUST check current grants, an active or explicitly restored parent, name conflicts and dependencies, and MUST display deletion provenance and the eligibility window.
*Source: N15.*

**FR-PRJ-046** · Restore MUST NOT resurrect children deleted independently earlier — cascade provenance MUST be recorded and respected. The required active parent MUST be restored first.
*Source: N15; register identity completion.*

**FR-PRJ-047** · **Restore MUST produce a new C02 working revision** requiring new C03 compilation and new human C04 review. It MUST NEVER reactivate a C04 approval or old privileges. Retained historical runs and approvals MUST continue to reference immutable C02 / C03 snapshots, not the latest mutable row.
*Source: N15; board 3 completion; register identity completion.*

**FR-PRJ-048** · Purge MUST be irreversible, policy-gated, performed by retention workers, with its **own per-item impact review and confirmation**. A bulk purge shortcut and an unrestricted generic hard-delete MUST NOT exist.
*Source: N15; register identity completion.*

**FR-PRJ-049** · Purge eligibility MUST consider running or unknown work, retained snapshot references, retention and legal obligations, and backup expiry.
*Source: register identity completion.*

**FR-PRJ-050** · Trash and history views MUST have dedicated scope checks and MUST NOT expose other tenants or unauthorized project data.
*Source: N15; register identity completion.*

**FR-PRJ-051** · Database cascades MUST NOT unexpectedly erase required execution history.
*Source: register identity completion; board 1 completion.*

## B.6 Environments

**FR-PRJ-060** · A customer target `Environment` MUST be a first-class tenant / project-owned domain object, **distinct from any ASpectra hosting stage**.
*Source: board 1 completion; register "Customer target environment".*

**FR-PRJ-061** · `EnvironmentRevision` MUST be immutable and MUST carry: `tenantId` / `projectId`, stable `environmentId`, immutable `envRevision`, display name, status, owner and `envSnapshotDigest`.
*Source: board 1 completion; N07.*

**FR-PRJ-062** · Target bindings MUST include: `targetAliases`; allowed origins, ports and path constraints; API and Web base URL values; DB connection `SecretRef`s; mobile app digest, device and runtime profile; plus typed references to Fixture, Locator, Schema and PublicTrustMaterial revisions.
*Source: board 1 completion; N07; R19.*

**FR-PRJ-063** · Execution policy on the environment MUST include: `fixtureRefs`, locale, timezone, limits, reset policy, egress policy and secret binding / version policy. Credential **rotation MUST NOT silently change target or privilege scope**; the resolved key version MUST be recorded, never the value.
*Source: board 1 completion; register "Customer target environment".*

**FR-PRJ-064** · The preferred target form MUST be a typed `EnvTargetRef(alias, relativePath)`. An explicit full URL MUST be permitted only where it satisfies the selected environment policy, and a legacy absolute URL MUST NOT be silently replaced.
*Source: board 1 completion; board 2 completion; register.*

**FR-PRJ-065** · URL handling MUST use maintained URI semantics covering trailing slash, root-relative paths, encoding, and scheme / host / port / path policy. String concatenation MUST NOT be used.
*Source: board 1 completion; board 2 completion.*

**FR-PRJ-066** · The editor MUST render a **non-secret** resolved full-URL preview and MUST ask missing-context questions. A guessed URL, a default environment and any credential MUST NOT appear.
*Source: N07; board 3 completion.*

**FR-PRJ-067** · Editing an environment MUST create a **new revision from** the prior one, with no "latest" fallback. Saving MUST produce a new environment hash, and the surface MUST state that changed pins require recompile and a fresh exact C04.
*Source: N07; R19.*

**FR-PRJ-068** · An unset, archived, revoked or incompatible environment MUST block compile, approval or Run with an **actionable** finding.
*Source: board 1 completion; R19.*

**FR-PRJ-069** · The environments list MUST separate **configuration validity** from **fresh profile assessment** as two distinct columns, because they answer different questions.
*Source: R19.*

**FR-PRJ-070** · **Save, Validate and Approve MUST NEVER contact the SUT.** "Check from managed profile" MUST be a separate explicit consented action. A missing assessment MUST block **Run only** — never configuration, manual authoring, static validation or human approval.
*Source: R19; N07; GAP-BE; GAP-AUTHOR.*

**FR-PRJ-071** · An environment display name MUST NOT be presented as evidence of execution region or data residency. The surface MUST state that execution region does not establish SaaS storage or AI residency, and MUST make no VPN or private-network promise.
*Source: R19; N07; GAP-BE; GAP-OPERATOR.*

**FR-PRJ-072** · Where a `ManagedNetworkProfile` is unpublished, the environment surface MUST render it as **`UNCONFIGURED`** with admission blocked, MUST NOT infer an allowlist address or region, and MUST direct the user to review the missing descriptor.
*Source: R19; N07; GAP-OPERATOR; FR-OPS-020.*

**FR-PRJ-073** · Mobile and DB references MUST require `READY` + published state, signing / provisioning verification and device-profile compatibility. The surface MUST state that scan status alone is not safety proof.
*Source: N07; N34.*

**FR-PRJ-074** · Secrets on the environment MUST be write-only references whose value is never shown and never browser-persisted.
*Source: N07; FR-X-031.*

**FR-PRJ-075** · Static URI checks MUST NOT be presented as proof of availability or complete SSRF protection. Runtime outbound controls (actual DNS, redirects, Web subresources, WebSockets, API and JDBC destinations) MUST forbid metadata, loopback and unapproved targets, with no fallback. These belong to the protocol runtime, not to C03.
*Source: board 1 completion; board 2 completion; GAP-CORE.*

The consented reachability check itself — its stages, consent presentation, coverage semantics and expiry behaviour — is specified in **H.4**.

## B.7 Public trust material

**FR-PRJ-080** · Public trust material MUST be a project-scoped, immutable `PublicTrustMaterialRevision` containing **only** a public X.509 CA chain, fingerprints, validity, CA constraints, scope / profile usage and digest.
*Source: R20; N39; GAP-BE.*

**FR-PRJ-081** · The library MUST list trust revisions with digest, review / lifecycle state, validity and usage, and MUST support inspecting an exact revision and reviewing revoke impact.
*Source: R20.*

**FR-PRJ-082** · Replacement MUST create a new immutable trust revision and new dependent environment pins, requiring recompile and fresh approval. Existing pins MUST NOT be overwritten.
*Source: R20; N39.*

**FR-PRJ-083** · Expiry and revocation MUST be checked at **admission and run time**, never frozen into the compiled hash. Revoked, expired or unsupported trust MUST block new admission.
*Source: R20; N39; GAP-CORE.*

**FR-PRJ-084** · Raw certificate bytes MUST NOT enter a prompt, report or AI context. Only safe fingerprint metadata MUST appear in lineage, and it MUST NOT be presented as proof of reachability.
*Source: R20; N39.*

Intake, rejection and publication authority for trust material are specified in **C.6**.

## B.8 Integrations configuration

**FR-PRJ-090** · A project MUST support multiple versioned **source connections** and **delivery profiles**, with separate identities. The console identity MUST NEVER be reused as target or provider authentication.
*Source: R18.*

**FR-PRJ-091** · A source connection MUST belong to one enabled provider family, MUST be read-scoped to the selected items, and MUST hold a protected credential reference whose value is never revealed. **Jira Cloud issues** is the reference source family; further families follow I.6.
*Source: R18.*

**FR-PRJ-092** · Saving a connection MUST store configuration **only**. An authorized provider read or check MUST be a separate, explicit, external-contact action. Saving MUST NOT send a comment, create a defect, upload an attachment, open a pull request or run a test.
*Source: R18; N40.*

**FR-PRJ-093** · A provider family that is not enabled MUST appear **disabled**, with the reason and the evidence that would enable it (I.6). A generic TMS claim, a Git write or a PR MUST NOT be offered.
*Source: R18; N40; GAP-BE.*

**FR-PRJ-094** · The surface MUST state that one connector does not imply permission for every action, and that provider contract tests plus permission and dedupe verification are required before activation.
*Source: R18; N40.*

Delivery profile capability gating is specified in **F.7**.

## B.9 Project dashboard

**FR-PRJ-100** · The dashboard MUST show only the viewer's permitted tenant and project facts: linked criteria with an **explicit denominator**, latest completed verdict, review readiness, and the viewer's own execution activity.
*Source: R11.*

**FR-PRJ-101** · Coverage MUST be expressed as linked, versioned criteria within a **named source snapshot**, plus unmapped gaps. Criteria outside the snapshot MUST be shown as unknown. Complete product coverage MUST NOT be inferred from passing tests.
*Source: R11; register §9.*

**FR-PRJ-102** · Queued and NOT EXECUTED tickets MUST be **excluded from verdict denominators** and from any pass-rate measurement.
*Source: R11; GAP-FE.*

**FR-PRJ-103** · The dashboard MUST state that a statically reviewable bundle is not a completed run, and that cancelled or expired queue intents are NOT EXECUTED.
*Source: R11; I8.*

**FR-PRJ-104** · No dashboard action MUST start execution. There MUST be no global customer dashboard, no cross-tenant activity and no cross-client learning corpus.
*Source: R11; register §9.*

## B.10 Tenant overview and first-run readiness

**FR-PRJ-110** · After sign-in and tenant selection, a tenant overview MUST list only the projects the viewer's **current grants** permit, each with the viewer's own grants, its working state and one explicit next step.
*Source: R10; M04.*

**FR-PRJ-111** · The overview MUST state that **tenant administration does not grant project content access**.
*Source: R10; FR-IDN-110.*

**FR-PRJ-112** · It MUST show the accessible scope — the number of permitted projects, the viewer's tenant role and identity-protection state — and MUST NOT present cross-customer activity, global metrics or implied product-wide coverage. It is **not a global customer dashboard**.
*Source: R10; M04.*

**FR-PRJ-113** · Search and filters on the overview MUST operate only within permitted scope.
*Source: R10.*

**FR-PRJ-114** · A first-run readiness checklist MUST be available covering: verified local identity · Owner TOTP protection · a project with explicit grants · a reviewable environment revision · a first scenario authored, validated and reviewed.
*Source: R09.*

**FR-PRJ-115** · The checklist MUST state that source and delivery integrations are optional and skippable, that **saving an environment contacts no target**, and that **no first run is automatic**.
*Source: R09; FR-PRJ-070.*

**FR-PRJ-116** · The checklist MUST state that Author, Reviewer and `canRun` remain separate explicit grants **even for a solo Owner**.
*Source: R09; FR-IDN-112.*

## B.11 Area B — verification requirements

Composite ownership / FK enforcement and RLS under attack · tombstone and restore races, cascade provenance, name conflicts and retained history · whole-project versus content lifecycle authority · target resolution, full-URL policy, redirect and DNS behaviour, secret rotation · missing environment and changed environment digest · archived / revoked / incompatible environment blocking · trust validity and revocation at admission · delivery and source grant separation · dashboard scope isolation · WCAG 2.2 AA on the configuration and lifecycle journeys.
*Source: board 1 §08, completion; register §12; GAP-BE.*

---

# C · Step catalog and library — `FR-CAT`

**Scope.** The C01 platform catalog and its publication pipeline; the four availability axes; C01-M project macros; C01-A reusable assets (fixture, schema, locator), mobile app binaries and public trust material intake.

**Screens.** R25 StepLibrary · R26 MacroLibrary · N36 MacroRevisionEditor · N37 ReusableAssetLibrary · N38 ReusableAssetRevision · N34 MobileAppRevisions · N35 MobileAppIntakeReview · N39 PublicTrustRevision.

## C.1 C01 CatalogManifest — publication

**FR-CAT-001** · `C01 CatalogManifest` MUST be produced by the **Core build** from actual Java glue and descriptors, and MUST carry: `schemaVersion`; `catalogVersion` and hash; plugin and runtime artifact digests; stable `stepId` + `contractRevision`; the actual binding fingerprint; canonical renderer plus parameter, table and docstring schemas; prerequisite, output, effect and retry metadata; deprecation and unavailable reasons; and EN / ES / FR display labels.
*Source: board 3 §05; board 2 §02; GAP-ALIGN.*

**FR-CAT-002** · Step authoring in Core MUST annotate an explicit **stable `StepId` + `contractRevision`** alongside the actual Cucumber expression, and MUST publish typed argument, output, table / docstring and prerequisite / effect / retry descriptors. Handler implementation MUST stay behind protocol glue.
*Source: board 2 §02.*

**FR-CAT-003** · Build extraction MUST read compiled metadata **without class initialization and without calling any `ResourceFactory`**. It MUST assemble a binding index, sort manifest entries deterministically, serialize canonical bytes and hash C01.
*Source: board 2 §02.*

**FR-CAT-004** · Extraction MUST **fail** on: duplicate step IDs or aliases, alias cycles, ambiguous bindings, and schema mismatch. Published handler artifact digests MUST accompany the metadata.
*Source: board 2 §02.*

**FR-CAT-005** · BE MUST ingest and retain the **exact historical manifest** with its plugin and runtime artifact digests. The manifest MUST NOT be reconstructed from a hand-maintained catalog, and the client MUST NOT build a separate catalog.
*Source: board 2 §02; board 3 §05; register §8.*

**FR-CAT-006** · Only the immutable C01 platform release MUST be globally read-only. Macros, fixtures, locators, schemas, evidence, conversations, approvals and execution history MUST remain tenant-owned.
*Source: register §5; board 1 §02; I4.*

**FR-CAT-007** · `Given` / `When` / `Then` / `And` / `But` MUST NOT create independent matching namespaces. Identical expressions MAY conflict and MUST be reported as ambiguous.
*Source: board 2 §02, completion; register.*

## C.2 The four availability axes

**FR-CAT-010** · The system MUST model and display **four distinct availability axes**: `implemented` ≠ `offered` ≠ `runner-compatible` ≠ `live ready`. Current runtime availability belongs to C06 and MUST be separate from catalog truth. Live runner changes MUST NOT rewrite the immutable C01 catalog.
*Source: R25; board 3 §03; GAP-CORE.*

**FR-CAT-011** · Availability MUST carry an explicit reason when a capability is unavailable, deprecated, revoked or profile-incompatible, and such a capability MUST block use.
*Source: R25; board 2 §02.*

**FR-CAT-012** · **The six file handlers MUST remain `UNAVAILABLE`** until **both** an actual offered C01 release **and** a tested resolver exist. Publishing a fixture or providing fixture storage MUST NOT activate a handler. The surface MUST state this explicitly.
*Source: R25; N37; GAP-CORE; register §13.1 finding 6.*

**FR-CAT-013** · **The eight conditional "if-present" steps MUST remain `UNAVAILABLE` intentionally**, with the reason shown. No conditional assertion weakening MUST be offered.
*Source: R25; N37; GAP-CORE.*

**FR-CAT-014** · The historical "515 steps, 14 hidden" figures MUST NOT be used as capability proof, a count-based support promise or a compatibility guarantee. Actual C01 MUST be the only source of truth.
*Source: R25; GAP-CORE; GAP-ALIGN.*

**FR-CAT-015** · Step resolution MUST NOT use nearest-match, similar-step substitution, guessed aliases, dynamic `eval` or capability inference from historical counts. Unknown fields and schemas MUST be rendered read-only with an actionable reason.
*Source: R25; board 3 §03; GAP-CORE.*

**FR-CAT-016** · New executable capability MUST require a **developer-controlled Core plugin, tests and an immutable release**. Arbitrary Java handlers, Cucumber glue, scripts and generated handlers MUST NOT be creatable from the product — by a user or by AI.
*Source: register §6; GAP-CORE; GAP-AUTHOR; I2.*

## C.3 Step library surface

**FR-CAT-020** · The step library MUST present three tabs: **global C01 capabilities**, **project C01-M macros**, **project C01-A assets**.
*Source: R25.*

**FR-CAT-021** · The C01 list MUST be searchable by exact ID, label or typed argument, filterable by protocol (HTTP / Web / Mobile / DB) and by manifest version, and MUST show per row: exact ID, typed arguments, availability and profile / readiness.
*Source: R25.*

**FR-CAT-022** · Step detail MUST show manifest version, EN / ES / FR labels, typed arguments with the expression alias **and the language that alias is actually registered in**, plus the four-axis availability statement.
*Source: R25.*

**FR-CAT-023** · The library MUST state that project macros are typed compositions, never new handlers, and never names that shadow Core steps.
*Source: R25; R26.*

## C.4 C01-M MacroRevision

**FR-CAT-030** · A `MacroRevision` MUST carry: `tenantId` / `projectId`, kind, namespace, revision; typed inputs and outputs; an **ordered AST of `CoreRef` and `MacroRef` nodes only**; exact dependency revisions; `expandedHash` and `sourceMap`; and author / review / audit records.
*Source: R26; N36; GAP-ALIGN.*

**FR-CAT-031** · A macro namespace MUST NEVER shadow a global C01 `stepId`. A namespace check MUST be a blocking diagnostic.
*Source: N36; R26; GAP-BE.*

**FR-CAT-032** · The composition MUST accept **only** `CoreRef` and `MacroRef` nodes. Java, Cucumber glue, scripts, generated handlers and glue injection MUST be impossible.
*Source: N36; R26; GAP-AUTHOR.*

**FR-CAT-033** · The editor MUST enforce and display three bounds — **expansion depth**, **expanded step count** and **cycle check** — as configured values, not invented numbers, each shown as `actual of permitted`.
*Source: N36; GAP-BE; FR-X-046.*

**FR-CAT-034** · Macro lifecycle MUST be `DRAFT → IN_REVIEW → PUBLISHED → DEPRECATED | REVOKED`. An **Author** creates and submits a Draft; a **Reviewer** publishes an exact immutable revision.
*Source: R26; N36; GAP-AUTHOR.*

**FR-CAT-035** · An invalid draft MUST be **savable with its diagnostics** so no work is lost, and MUST NOT be publishable. A scenario selecting an invalid or unpublished macro MUST NOT compile.
*Source: N36.*

**FR-CAT-036** · Each diagnostic MUST be associated with the **exact AST node** it belongs to, with a focus target and a named resolution. Diagnostics MUST NOT be collapsed into a single error.
*Source: N36.*

**FR-CAT-037** · `expandedHash` MUST NOT be computed while a draft is invalid. On publish, the `sourceMap` (invocation → expanded occurrence) MUST be retained.
*Source: N36.*

**FR-CAT-038** · **BE MUST expand a published macro before C03.** Core MUST only ever receive Core steps.
*Source: N36; GAP-CORE; board 2 §02.*

**FR-CAT-039** · **Publishing a macro MUST NEVER constitute scenario C04 approval**, and MUST NEVER launch a Run. This MUST be stated on the surface.
*Source: N36; R26; GAP-AUTHOR.*

**FR-CAT-040** · Deprecation MUST warn on new bindings and MUST NOT upgrade an existing pin. Revocation MUST block new admission while history is retained. Existing bundles MUST keep their original version until recompile and human review.
*Source: R26; N36.*

**FR-CAT-041** · The macro library MUST list revisions with namespace, immutable version, dependency and `expandedHash`, and reviewer publication state; and MUST offer a **usage impact** view.
*Source: R26.*

**FR-CAT-042** · Where a macro dependency is itself unavailable (for example an unpublished resolver), the surface MUST state that review or publication of a macro **cannot create runtime support**.
*Source: R26; N36.*

## C.5 C01-A AssetRevision — fixtures, schemas, locators

**FR-CAT-050** · An `AssetRevision` MUST carry: `tenantId` / `projectId` / `assetId` / `revision` / `kind` / `schemaVersion`; `contentDigest`; `mediaType`; size; `privateStorageRef`; validation and `scanPolicyVersion`; author and reviewer; `dependencyClosure`; and publication / revocation audit.
*Source: N38; GAP-ALIGN; GAP-BE.*

**FR-CAT-051** · Assets MUST have **two independent lifecycle axes that are never collapsed**:
- **Intake** — `UPLOADING → QUARANTINED → VALIDATING → READY | REJECTED` — answers whether the bytes are safe to keep.
- **Publication** — `DRAFT → IN_REVIEW → PUBLISHED → DEPRECATED | REVOKED` — answers whether they may be pinned.

`READY` is not published; published is not approved; neither is ever a Run.
*Source: N34; N37; GAP-BE.*

**FR-CAT-052** · Per-kind contracts MUST be enforced:
- **Fixture** — bounded JSON, CSV or approved file type; declared shape, encoding and sensitivity; resolved read-only into the run sandbox.
- **Schema** — supported dialect and root, with the **full local dependency closure pinned by digest**.
- **Locator** — typed Web or Mobile strategy, selector and scope. **No interactive DOM inspector MUST be promised.**

*Source: N37; GAP-BE; GAP-CORE.*

**FR-CAT-053** · Every asset kind MUST be immutable per revision and pinned into C03 like any other dependency.
*Source: N37; N38.*

**FR-CAT-054** · Resolver guardrails MUST be enforced and displayed. The resolver MUST NOT: read an arbitrary filesystem path; fetch a fixture over the network at run time; run an executable generator; or fetch a remote `$ref` during compile or execution.
*Source: N37; GAP-CORE.*

**FR-CAT-055** · A `RuntimeFixtureRef` MUST resolve **only** to a read-only path inside the run sandbox, validated against its `contentDigest`.
*Source: N37; N38; GAP-CORE.*

**FR-CAT-056** · Asset revision detail MUST be **non-editable in place**. A change MUST produce a new revision with a new `contentDigest`. Existing pins MUST keep the revision they were approved with and MUST NOT be silently upgraded.
*Source: N38; I7.*

**FR-CAT-057** · Asset revision detail MUST display a **"pinned by" impact table**: for each pin (macro, bundle, environment, approval) the exact pin, what a replacement invalidates, and what a revocation blocks.
*Source: N38.*

**FR-CAT-058** · Revocation MUST be **forward-only**: it blocks new admission and new bindings, and MUST NOT rewrite a recorded fact, cancel a completed run, or delete the lineage of work that already used the revision.
*Source: N38.*

**FR-CAT-059** · Upload authority MUST require an Author grant in project scope, with configured type, size and storage caps. **The actual bytes MUST determine the kind** — a client-supplied filename or hash MUST be recorded for lineage but MUST NOT be trusted as authority.
*Source: GAP-BE; GAP-IMPORT; N35.*

**FR-CAT-060** · Intake validation MUST run in private quarantine with isolated parse, scan and signature policy, including archive, path traversal, zip-bomb, symlink and prohibited-reference controls. **Intake MUST NOT install, execute or unpack to a shared path.**
*Source: GAP-BE; GAP-IMPORT; N34; N35.*

**FR-CAT-061** · Scan status MUST be recorded as "a scan ran under a stated policy" and MUST NOT be presented as a guarantee that the bytes are harmless.
*Source: N34; N35; N38.*

**FR-CAT-062** · Sensitive values MUST be `SecretRef`s, never embedded literals. Raw asset content MUST NEVER be placed in a prompt, log, index or report.
*Source: N38; FR-X-032.*

**FR-CAT-063** · An asset rejected by sensitivity policy MUST remain unpublished with its reason, and MUST NOT be publishable by any later action on the same revision.
*Source: N37.*

## C.6 MobileAppRevision and PublicTrustMaterialRevision intake

**FR-CAT-070** · A `MobileAppRevision` MUST record: platform, `packageId`, version, `minOS`, architecture, signing identity verified **from the actual bytes**, provisioning metadata, `contentDigest`, size, scan status with policy version, and compatible device profiles.
*Source: N34; N35; GAP-BE.*

**FR-CAT-071** · A new binary MUST always be a new revision. **An alias MUST NEVER be repointed.** Cross-project reuse follows I.7.
*Source: N34.*

**FR-CAT-072** · Intake review MUST evaluate and display each check with its policy, outcome and **whether it blocks publication**: archive and path safety; malware scan; signing identity; provisioning profile match against an offered device profile; device profile compatibility. A failing check MUST name the exact field it belongs to.
*Source: N35.*

**FR-CAT-073** · **iOS MUST accept compatible signed and provisioned IPAs only.** ASpectra MUST NEVER hold a customer private key and MUST NEVER re-sign a binary. An arbitrary IPA MUST NOT be guaranteed.
*Source: N35; N34; GAP-BE.*

**FR-CAT-074** · Rejection MUST retain the quarantined bytes, every finding and full audit lineage, and MUST create no published revision, no environment pin and no approval. The next step MUST be stated.
*Source: N35.*

**FR-CAT-075** · **Only an admitted C05 + C06 run MUST install an approved digest**, and only onto an exclusively leased device.
*Source: N34; GAP-CORE.*

**FR-CAT-076** · Replacing a pinned binary MUST create a new revision and a new environment hash, and MUST require recompile plus a fresh human C04.
*Source: N34.*

**FR-CAT-077** · Public trust intake MUST classify material **from the actual bytes, never from the file name or declared type**, and MUST display declared-versus-detected content.
*Source: N39.*

**FR-CAT-078** · Public trust intake MUST **reject before storage**: any private key in any format, any client identity, and any key container (for example PKCS#12). Nothing MUST be stored on rejection; the audit record MUST retain the rejection reason and a safe fingerprint only, and no key material.
*Source: N39; GAP-BE.*

**FR-CAT-079** · The accepted-material table MUST be explicit per material type: public CA chain (accepted), intermediate CAs (accepted), server leaf (accepted as reference only), private key (rejected), client identity or container (rejected; client certificate identity is governed by I.8) — each with the reason and the action to take.
*Source: N39.*

**FR-CAT-080** · Validation MUST record subject and issuer **read from the chain**, SHA-256 fingerprint and `contentDigest`, server-supplied validity window, CA basic constraints and key usage, and the scope / profile binding.
*Source: N39; R20.*

**FR-CAT-081** · At run time, trust MUST be applied **per connection and per profile, only on an adapter whose capability is published and tested**. `ignoreHTTPSErrors`, hostname-verification bypass and mutation of global JVM or browser trust MUST NOT exist anywhere in the system.
*Source: N39; GAP-CORE; R20.*

**FR-CAT-082** · **Publishing trust material MUST require two authorities**: a project Reviewer grant **and** tenant Owner or Admin. It is the one asset kind where project authority alone is insufficient.
*Source: N39; GAP-BE.*

## C.7 Area C — verification requirements

Deterministic no-I/O manifest extraction and actual binding parity · duplicate / alias / cycle / ambiguity rejection · golden-fixture compatibility in every actually supported language, including Unicode, escaping, tables and docstrings · macro cycle, depth, count, revocation and expansion-mapping limits · quarantine, archive, signature, provisioning and trust gates · resolver compatibility and read-only sandbox enforcement · unavailable-handler guards · asset revocation and pin-invalidation behaviour · WCAG 2.2 AA on the library and editor journeys.
*Source: board 2 §04, completion; GAP-CORE; GAP-AUTHOR; register §12.*

---

# D · Authoring and review flow — `FR-AUT`

**Scope.** Structured and free-form authoring, source import, C11 semantic mapping and acceptance, C02 canonical revisions, the editor state model, C03 pure compilation, revision diff and the project chat.

**Screens.** R24 ScenarioBuilder · N09 FreeformAuthoring · R27 Imports · N08 SourceEvidenceDetail · R28 ProposalsQueue · R29 SourceMappingReview · N10 RevisionDiff · N11 ValidationBundle · N13 ProjectChat.

## D.1 Input paths

**FR-AUT-001** · Three input paths MUST converge on **one** canonical editing model: (a) the structured catalog builder; (b) original free-form wording; (c) source import. Manual and import paths MUST be fully usable with AI disabled.
*Source: board 6; board 5; GAP-AUTHOR; GAP-FE.*

**FR-AUT-002** · Free-form authoring MUST NOT require catalog browsing. The author's **original wording MUST be preserved exactly** — text, hash and span.
*Source: N09; board 6; GAP-AUTHOR.*

**FR-AUT-003** · The single exception to exact storage MUST be sensitive literals: they MUST be extracted to `SecretRef` and redacted before persistence or egress, **and the transformation MUST be shown to the author**. Plaintext secrets MUST NOT persist in original-text history.
*Source: register §6; FR-X-032.*

**FR-AUT-004** · Save MUST preserve wording and history, and MUST NOT accept a mapping, approve a bundle or Run.
*Source: N09; I1.*

## D.2 C11 AuthoredIntentMapping

**FR-AUT-010** · `C11 AuthoredIntentMapping` MUST record: exact sanitized authored text, hash and span; candidate and accepted typed bindings with typed arguments; evidence, citations, assumptions and gaps; mapping revision and status; and the **human acceptance digest**.
*Source: board 3 §01; register §11; GAP-ALIGN.*

**FR-AUT-011** · The mapping surface MUST display the original wording **beside** the derived executable meaning, with explicit **executable differences**. Internal IDs MAY be hidden; executable meaning and differences MUST NOT be.
*Source: board 3 §01; R29; N09.*

**FR-AUT-012** · Unsupported or ambiguous meaning MUST produce a **question or a retained gap**. A handler MUST NEVER be guessed, and expected behaviour MUST NEVER be invented from implementation or from observed failing output.
*Source: register §6; N09; R29; GAP-AUTHOR.*

**FR-AUT-013** · Where a candidate mapping does not satisfy the full criterion, the surface MUST say so plainly, name the unmapped part, and offer either clarification or an **explicitly narrowed** selection with the original gap preserved.
*Source: N09; R29; N10.*

**FR-AUT-014** · A deliberate scope narrowing MUST be recorded as an auditable **decision record** with rationale, the accepting actor, the preserved source link and the retained gap identifier. It MUST state that acceptance is not approval and does not start a Run.
*Source: N10; R29; N18.*

**FR-AUT-015** · Cosmetic / formatting normalization MUST be recorded **without repeated prompts**. A semantic change MUST require visible acceptance and MUST invalidate the affected C11 acceptance, C03 and C04.
*Source: register §6; board 3 §01; GAP-FE.*

**FR-AUT-016** · Direct structured catalog selection MUST be able to record acceptance on explicit save, without mandatory AI and without a redundant prompt.
*Source: register §6; board 6.*

**FR-AUT-017** · A preserved source gap MUST be **excluded from any linked-criteria coverage claim** and MUST remain visible against the original criterion.
*Source: N10; N08; R11.*

## D.3 C02 canonical revision

**FR-AUT-020** · `C02 TestRevision` MUST be immutable and MUST carry: tenant and project ownership; `sourceIntentRef`; `acceptedMappingDigest`; exact `stepId` + `contractRevision` per step; stable `stepOccurrenceID`s; typed arguments, variable-output refs, `secretRef`s, `fixtureRef`s, actor and resource refs; and pinned macro and asset revisions with source provenance.
*Source: board 3 §05; register §11; board 2 §02.*

**FR-AUT-021** · C02 MUST be produced by a **deterministic BE domain service**. It MUST NOT be produced by an agent, and AI MUST NOT hold canonical authority or silently rewrite intent.
*Source: board 3 §01; board 6; register §6; I2.*

**FR-AUT-022** · New executable meaning MUST produce a **new revision**. Approved history MUST NEVER be mutated or overwritten.
*Source: GAP-ALIGN; register §5.*

**FR-AUT-023** · A stale base revision or a foreign-owned reference MUST be rejected. Visible mapping gaps MUST remain, with no silent repair.
*Source: GAP-ALIGN.*

**FR-AUT-024** · **Gherkin MUST be a human view, an import format or a compiled artifact — never a second mutable source model.** C02 is the only editable canonical model.
*Source: board 2 §02; board 5; board 6.*

## D.4 The editor state model

**FR-AUT-030** · TanStack Query MUST own server facts: C01 manifests, saved C02 revisions, jobs and environments, C03 bundles, C04 approvals, C05 runs, C07 results and C09 delivery. Query keys MUST include tenant + project + `schemaVersion` + the immutable hash or exact version where relevant, plus capability / profile when filtered.
*Source: board 3 §02; GAP-FE; register §10.*

**FR-AUT-031** · Zustand MUST own only a **deliberate unsaved editor fork**, pinned to `tenantId` / `projectId` / `baseETag` / `editSeq`, with base snapshot, working copy, dirty flag and undo history. It MUST NOT mirror the server cache and MUST NOT be a tenant-global store.
*Source: board 3 §02; GAP-FE.*

**FR-AUT-032** · A Query refetch MUST NEVER overwrite a dirty draft. Only facts affected by an accepted server mutation MUST be invalidated. "Latest" MUST be discovery only — **resolve then pin, never silently rebind a revision**.
*Source: board 3 §02.*

**FR-AUT-033** · Saves MUST be serialized. A save MUST capture content plus `editSeq`; if typing continues while a save is pending, only the captured edit MUST be acknowledged and newer edits MUST remain dirty.
*Source: board 3 §02.*

**FR-AUT-034** · Conditional save MUST use a strong `If-Match` ETag. `412` MUST mean stale and `409` MUST mean workflow conflict. The UI MUST present **base / local / server** with explicit resolution. Blind merge, blind overwrite and silent re-approval MUST NOT occur.
*Source: board 3 §02.*

**FR-AUT-035** · The server MUST reject outdated revisions and changed `permissionVersion`. The client MUST NOT seal executable hashes or decide a verdict.
*Source: GAP-FE; board 3.*

**FR-AUT-036** · An unsaved-work route guard MUST offer **Save / Discard / Stay**.
*Source: board 3 §07; FR-IDN-130.*

**FR-AUT-037** · Diagnostics MUST be addressed by `stepOccurrenceID` and parameter path, **never by row index**, so there is no diagnostic drift after reordering.
*Source: board 3 §03; board 2 §02.*

**FR-AUT-038** · Static validation requests MUST be debounced and MUST carry request identity + `editSeq` + input pins. Out-of-order or stale responses MUST be discarded. A locally "valid" draft MUST NOT be presented as sealed executable bytes.
*Source: board 3 §03.*

**FR-AUT-039** · The URL MUST hold resource IDs and filters only; component state MUST hold ephemeral focus, menus and selection only.
*Source: board 3 §02.*

**FR-AUT-040** · The structured builder MUST render fields from supported C01 schemas with availability and reasons, and MUST allow selecting exact immutable macro and asset revisions by type and `contentDigest`, with **no "latest" fallback**.
*Source: R24; board 3 §03.*

**FR-AUT-041** · The builder MUST offer a bounded **expansion preview** with retained `sourceMap`, and provenance inspection.
*Source: R24.*

**FR-AUT-042** · An unavailable step selected in a draft MUST be marked `UNAVAILABLE` with its reason inline and MUST block compilation.
*Source: R24; R25.*

## D.5 Source import

**FR-AUT-050** · Supported source families MUST be exactly: user stories (HU / UH) with acceptance criteria; product / API / architecture / test documentation; OpenAPI or Swagger; Postman collections with environment definitions; existing `.feature` files. Pasted non-sensitive original wording MUST also be accepted. Repository-informed intake is specified in **I.1** and reuses this evidence model.
*Source: R27; board 5; GAP-IMPORT.*

**FR-AUT-051** · Import MUST be a **durable job** that validates source type and size and records a versioned private snapshot with its hash and provenance, preserving the original sanitized wording.
*Source: R27; board 5; N08.*

**FR-AUT-052** · Intake MUST enforce: size, type and archive bounds; XXE, path traversal, SSRF, zip-bomb, symlink and prohibited-reference controls. **No imported script, Postman script, active document content or source code MUST be executed**, during ingestion or ever.
*Source: R27; board 5; GAP-IMPORT.*

**FR-AUT-053** · Structured parsers MUST extract OpenAPI operations and schemas, Postman requests and variables, and Gherkin parse plus catalog mapping. Unsupported Postman scripts and unresolved Gherkin MUST remain **explicit visible gaps** — never executed and never silently dropped.
*Source: board 5; R27.*

**FR-AUT-054** · Document and story extraction MUST preserve story text, acceptance criteria and citations with spans. Requirements MUST ground assertions; observed application output MUST NOT.
*Source: board 5; N08.*

**FR-AUT-055** · Source evidence detail MUST show: the sanitized read-only snapshot with a safe text preview; per-criterion mapping with its coverage claim; parser report with exclusions; duplicates; and snapshot identity (revision, sanitized content digest, importer, parser, access requirement).
*Source: N08.*

**FR-AUT-056** · Duplicate content MUST be **linked, not silently discarded**; citations and provenance MUST remain available.
*Source: N08.*

**FR-AUT-057** · Mechanically valid imports MUST normalize deterministically and **bypass AI**. Narrative requirements MUST require semantic composition and human mapping review.
*Source: board 5; GAP-IMPORT.*

**FR-AUT-058** · A current tenant / project / source grant MUST be checked **before retrieval and again before any provider submission**. Source-grant revocation MUST block new context and dependent new approval or use pending policy review, while immutable history and lineage are retained.
*Source: board 5; register §7; GAP-IMPORT; GAP-FAILURE.*

**FR-AUT-059** · Source or meaning changes MUST create a new C02, require re-acceptance of affected C11, recompile and human approval. A stale import MUST NOT overwrite.
*Source: GAP-IMPORT; board 5.*

## D.6 Proposals and mapping review

**FR-AUT-070** · `C08 ProposalEnvelope` MUST carry typed proposed changes pinned to tenant / project / source / `baseRevision` / catalog / C12 policy, with evidence, citations and gaps. **It MUST never carry approval, Run or integration-write authority.**
*Source: board 3 §05; register §11; I2.*

**FR-AUT-071** · The proposals queue MUST list candidates with criterion, source and citation, mapping state and the decision to take, and MUST state that **confidence is not correctness**. Bulk approval, hidden mutation, launch and approve tools MUST NOT exist.
*Source: R28.*

**FR-AUT-072** · The queue MUST show the assistance identity — provider and model from the tenant-approved selection — and its authority: propose and explain only.
*Source: R28; N13.*

**FR-AUT-073** · Mapping review MUST present, side by side: the preserved original with source snapshot and citation; the proposed catalog meaning step by step with typed arguments; and an explicit **meaning difference** statement. It MUST carry the citation, the assumption and the gap as separate labelled facts.
*Source: R29.*

**FR-AUT-074** · The reviewer MUST be able to **edit mapping**, **accept** (with the exact scope stated, including a narrowed scope), or **reject**. Acceptance MUST create a C02 revision and MUST NOT approve or run it.
*Source: R29; N09.*

**FR-AUT-075** · Revision diff MUST show, for two revisions: preserved original intent with citation; the authored change and who made it; what changed; the dependency / meaning / approval impact per pin; and the explicit statement that no prior approval carries forward.
*Source: N10.*

**FR-AUT-076** · A stale proposal MUST be rebased **explicitly**, without weakening assertion intent.
*Source: board 3 §03.*

**FR-AUT-077** · An AI-proposed repair MUST read persisted evidence only and MUST propose a new visible C11 meaning and revision. It MUST NEVER weaken an assertion, skip conditionally, swallow a failure, loop until green, or auto-rerun. The original failed run and assertion intent MUST be preserved.
*Source: register §8; board 1 §04; board 7; I2.*

## D.7 C03 pure compilation

**FR-AUT-080** · `C03` MUST produce a `ValidationReport` + `CompiledBundle` sealing: exact input, dependency, compiler, catalog and runtime digests; static diagnostics with source paths; expanded macros; generated feature bytes; the binding and source map; and the sealed executable digest.
*Source: board 3 §05; board 2 §02; N11.*

**FR-AUT-081** · **C03 MUST make no target, database, browser, device, JDBC or provider call, and MUST perform no probe or DNS lookup.** It MUST NOT install an app, allocate execution capacity or admit any wall-clock value (such as trust expiry) into its hash.
*Source: N11; GAP-CORE; I5.*

**FR-AUT-082** · Compiler purity MUST be **provable**: fail-on-use resource factories plus network-blocked compilation tests. A Cucumber dry-run alone MUST NOT be accepted as proof of no I/O.
*Source: board 2 §02.*

**FR-AUT-083** · The compiler MUST statically check: types; variable and output reference before use; prerequisites; resource state transitions; availability; and declared effects, against exact C01 plus dependencies.
*Source: board 2 §02.*

**FR-AUT-084** · Renderer / binding parity MUST be enforced: the deterministic typed renderer → generated feature bytes → real Cucumber matcher MUST yield **exactly one** expected binding per occurrence, with transformed parameter, table and docstring fidelity. The `sourceMap` MUST carry the expected binding fingerprint per `stepOccurrenceID`.
*Source: board 2 §02; register §8.*

**FR-AUT-085** · Unsupported typed renderers MUST be blocked until an explicit adapter exists. Regex MUST NOT be assumed invertible. Adversarial escaping MUST be part of acceptance.
*Source: board 2 §02.*

**FR-AUT-086** · The canonical serializer MUST be deterministic. A browser `JSON.stringify` MUST NOT produce canonical bytes.
*Source: board 2 §02; board 3 §03.*

**FR-AUT-087** · Blocking findings MUST include: unknown types; unpublished or revoked references; unsupported local schema dependencies; unavailable resolvers; and unsupported language or profile bindings. **A fixture store MUST NOT establish file-handler support.**
*Source: N11; GAP-IMPORT.*

**FR-AUT-088** · Static errors MUST be presented as **compiler findings that block approval**, never as target failures.
*Source: N11; GAP-CORE.*

**FR-AUT-089** · The validation bundle surface MUST display the exact pins (selection and accepted C11 scope; environment ID / revision / digest; C01 catalog and runtime; sealed bytes and bundle digest; macro revision and `expandedHash`; typed asset revisions and content digests; locator / schema / trust; policy and profile references) and MUST state that pins never silently upgrade.
*Source: N11; N12.*

**FR-AUT-090** · The surface MUST state plainly that **static validity is not network readiness**: a valid bundle may proceed to human C04 review without a fresh assessment, while C05 / C06 still require matching freshness, current asset eligibility, valid trust and published profile metadata.
*Source: N11.*

**FR-AUT-091** · Each diagnostic area MUST name its pinned evidence, its static boundary and its resolution path.
*Source: N11.*

## D.8 Project chat

**FR-AUT-100** · Project chat MUST be bounded to the active project and MUST flow: client message → C10-authorized BE AI use case → authorized minimal catalog / evidence / revision / run facts → bounded model adapter → **cited** answer or validated C08 / C11 proposal.
*Source: register §7; board 3 §02; N13.*

**FR-AUT-101** · Chat MUST explain steps and diagnostics, clarify requirements, propose scenarios and interpret persisted evidence. It MUST NOT have approval, launch, database, shell, repository, Core-handler generation or arbitrary integration-write tools. **Chat text saying "Run" MUST NOT bypass explicit Run.**
*Source: register §7; N13; I2.*

**FR-AUT-102** · Every assistant claim MUST carry a citation to a source, mapping or gap identifier. Advice MUST be labelled as advice, not fact.
*Source: N13.*

**FR-AUT-103** · A chat-originated proposal MUST be a candidate that is **not applied**, offering read differences / accept / edit / reject, with required clarifications named and acceptance blocked while required arguments or outcomes are missing. The original wording MUST remain untouched.
*Source: N13.*

**FR-AUT-104** · The chat surface MUST display its context and policy: tenant / project, allowed context, the C12 approved model alias, provider and funding identity, quota reservation and egress scope.
*Source: N13.*

**FR-AUT-105** · The message input MUST state that no secrets or credential payloads may be sent, and redaction MUST be applied server-side regardless.
*Source: N13; FR-X-032.*

**FR-AUT-106** · On provider or funding failure the AI request MUST stop with **no silent fallback**. Manual authoring, static compilation and base reports MUST remain available, and the UI MUST offer "continue manually".
*Source: N13; N14; GAP-FE.*

**FR-AUT-107** · Conversations MUST be tenant / project access-controlled and retained by policy, storing minimal permitted redacted messages and provenance — **not raw system or retrieval prompts** or payload logs.
*Source: register §7.*

**FR-AUT-108** · Shared client RAG, cross-tenant prompt / model-context / cache reuse, and training reuse MUST NOT exist.
*Source: register §5, §7; GAP-FE; I4.*

## D.9 Area D — verification requirements

C11 acceptance and hash drift · compiler parity and golden fixtures in every supported language · deterministic no-I/O compilation · macro expansion and `sourceMap` correctness · save race while typing, refetch over dirty draft, serialized saves, `412` / `409` base-local-server resolution, stale validation and `editSeq` races · reorder and duplicate identity · targeted proposal acceptance and explicit rebase · parser archive / XXE / traversal / SSRF limits · source-grant revocation races · AI-disabled operation and base-report independence · secret and log canaries · WCAG 2.2 AA on the authoring, mapping and review journeys, including keyboard-only non-pointer authoring.
*Source: board 3 §05, §07; board 2 §04; GAP-AUTHOR; GAP-IMPORT; register §12.*

---

# E · Approval path — `FR-APR`

**Scope.** The C04 human approval decision: what is approved, who may approve, what invalidates it, and what it explicitly does not do.

**Screens.** N12 ApprovalReview · N11 ValidationBundle (entry) · R28 ProposalsQueue (queue) · N18 AuditTrail (record).

## E.1 What is approved

**FR-APR-001** · `C04 ApprovalRecord` MUST record a human decision on: the **exact** `bundleDigest`; the exact selection; the exact `envRevision`; every execution-affecting dependency pin; the policy revision; and the actor with tenant, project and membership lineage.
*Source: board 3 §05; register §11; N12.*

**FR-APR-002** · The approval surface MUST display the full pin set — selection and accepted C11 scope; environment ID / revision / digest; C01 catalog and runtime; sealed bytes and bundle digest; C01-M macro revision and `expandedHash`; typed asset revisions with content digests; locator / schema / public trust; policy and profile references.
*Source: N12.*

**FR-APR-003** · The surface MUST display the **preserved original authored wording**, the accepted selection with provenance, and every **open gap explicitly excluded** from the linked-criteria claim.
*Source: N12.*

**FR-APR-004** · The surface MUST state that an explicit narrowing is **not assertion removal**, that the browser or other uncovered requirement remains visible and uncovered, and that approval **cannot silently weaken the original criterion or convert partial evidence into proof of the whole**.
*Source: N12; N10.*

**FR-APR-005** · Selection eligibility and static diagnostics MUST be shown, with blocking findings surfaced before the approve action. **No new scope exclusion MUST appear at Run.**
*Source: N12; R22.*

**FR-APR-006** · The surface MUST show review authority: author and reviewer identities; the independence of Author / Reviewer / `canRun`; the separateness of the tenant role; the self-approval policy in force; and the publication / eligibility pins reviewed with the bundle.
*Source: N12.*

**FR-APR-007** · An approval rationale MUST be capturable, and an explicit acknowledgement checkbox MUST confirm that the exact selection, environment revision, asset and macro dependencies and preserved gaps were reviewed.
*Source: N12.*

## E.2 What approval does not do

**FR-APR-010** · **Approval MUST have no launch side effect.** It MUST NOT Run, reserve capacity, install an app, allocate a device or test the network. This MUST be stated on the surface.
*Source: N12; I1; register §8.*

**FR-APR-011** · Membership or administrative status MUST NOT constitute approval.
*Source: register §8; FR-IDN-092.*

**FR-APR-012** · **Macro or asset review MUST NEVER be scenario approval.** Publishing an asset revision makes bytes eligible to be pinned; a scenario pinning them still requires its own C03 compile and its own C04.
*Source: N35; N36; N38; FR-CAT-039.*

**FR-APR-013** · A **pending fresh reachability assessment MUST NOT block human approval** and MUST NOT cause a network action. It MUST block Run until an explicit authorized managed-profile check and all admission prerequisites are satisfied.
*Source: N12; N11; R19.*

**FR-APR-014** · Reject or revise MUST produce a new C02 revision requiring recompile and fresh review. An existing review MUST NOT approve changed bytes.
*Source: board 3 §04; board 7; N12.*

## E.3 Who may approve

**FR-APR-020** · Approval MUST require an explicit project **Reviewer** grant. A tenant role MUST NOT substitute for it.
*Source: FR-IDN-091; N12.*

**FR-APR-021** · **Self-approval MUST default to allowed** for solo operation, and MUST be visible and audited. Tenant policy `requireDifferentApprover` MAY tighten it to require a reviewer other than the author.
*Source: register §3; N06; GAP-BE; N12.*

**FR-APR-022** · The self-approval policy changes decision **eligibility** only. It MUST NEVER remove the separate Run requirement.
*Source: register §3; I1.*

**FR-APR-023** · At the moment of decision the server MUST recheck current authority: membership, project grants, `permissionVersion`, tenant state and the current policy revision.
*Source: register §8; FR-X-004.*

## E.4 What invalidates an approval

**FR-APR-030** · **Any** change to an execution-affecting dependency MUST invalidate the affected approval and require recompile and renewed review. The invalidating set MUST include at minimum: the scenario revision, environment digest, C01 catalog, macro revision or `expandedHash`, fixture, locator or schema revision, public trust revision, runtime profile, and policy revision.
*Source: N12; register §8; I7.*

**FR-APR-031** · A stale, revoked or incompatible approval MUST block launch. An old approval record MUST NOT revive revoked authority, a revoked asset or a deleted parent.
*Source: GAP-ALIGN; board 3 §04; N15.*

**FR-APR-032** · Current revocation, expiry, capability loss or probe staleness MUST be able to block a new effect **despite** a historical approval. This is mutable eligibility, and MUST NOT be treated as nondeterministic compiler input.
*Source: GAP-ALIGN; GAP-BE.*

**FR-APR-033** · A restored resource MUST NOT revive its approval (FR-PRJ-047). Retained runs and approvals MUST continue to reference immutable C02 / C03 snapshots.
*Source: N15; register identity completion.*

## E.5 Audit

**FR-APR-040** · The approval decision MUST be recorded in the audit trail with actor, object lineage, the exact digest, the policy revision in force, and whether it was a self-approval.
*Source: N18; N12; FR-X-044.*

**FR-APR-041** · The audit trail MUST be able to present the full lineage in order: source preserved → scope accepted (C11) → revision (C02) → bundle sealed (C03) → approval (C04) → run accepted (C05) → facts (C07) → delivery (C09), each with a safe actor, object lineage, the decision or fact, and a detail link.
*Source: N18.*

## E.6 Area E — verification requirements

Approval integrity against changed digests and dependency revisions · concurrent approval and grant-revocation races · self-approval policy enforcement and audit completeness · stale and revoked approval blocking at launch · uniform gating across manual, import and AI inputs and across plan / direct / bulk / suite / feature paths · WCAG 2.2 AA on the review and approval journey.
*Source: board 1 §06 guardrails; register §12; board 3 §05.*

---

# F · Settings — `FR-SET`

**Scope.** Personal settings, identity security settings, tenant members, tenant policy, project settings and grants, integrations and delivery profiles, AI policy and usage, trash and audit entry points, account deletion.

**Screens.** R12 PersonalSettings · R38 SecuritySettings · R13 TenantMembers · N06 TenantPolicy · R16 ProjectSettings · R17 ProjectMembers · R18 ProjectIntegrations · N40 DeliveryProfileEditor · N14 AIPolicyUsage · N15 TrashRestore · N18 AuditTrail · N19 AccountDeletion.

## F.1 General settings behaviour

**FR-SET-001** · Every settings write MUST **confirm persistence before showing success**.
*Source: N06.*

**FR-SET-002** · Every policy or configuration change MUST record actor, previous value, new value, rationale where required, and the resulting policy or configuration revision.
*Source: N06; R16.*

**FR-SET-003** · No settings surface MUST reveal, copy or browser-persist a stored secret. Credential changes MUST be performed through an authorized write-only setup path.
*Source: N14; R18; FR-X-031.*

**FR-SET-004** · A settings change MUST NEVER constitute bundle approval or a Run.
*Source: N06; I1.*

## F.2 Personal settings

**FR-SET-010** · Personal settings MUST present: profile; security and sessions; connection visibility; account privacy.
*Source: R12.*

**FR-SET-011** · Profile MUST allow editing the display name, and MUST show the verified email with the statement that changing it requires a separate verification flow.
*Source: R12.*

**FR-SET-012** · Interface language MUST be selectable among EN / ES / FR, with the explicit statement that it does **not** change the Gherkin dialect or registered executable expressions.
*Source: R12; FR-X-043.*

**FR-SET-013** · Personal settings MUST show the identity's memberships and MUST state that leaving a tenant differs from deleting the global identity, and that a last Owner must transfer ownership first.
*Source: R12; N19.*

**FR-SET-014** · Connection visibility MUST show scoped connection metadata **read-only**, following current project grants, and MUST NOT expose connection credentials.
*Source: R12.*

## F.3 Identity security settings

**FR-SET-020** · Security settings MUST present four areas: TOTP and recovery; sessions; password; privacy.
*Source: R38.*

**FR-SET-021** · The TOTP area MUST state when MFA is required by the viewer's membership and that required MFA cannot be disabled, and that replacing a factor requires the current factor or controlled recovery — **never removal first**.
*Source: R38; FR-IDN-029.*

**FR-SET-022** · The factor area MUST show enrollment metadata only — never the seed — and MUST offer replace-authenticator, regenerate-recovery-codes and controlled-account-recovery entry points.
*Source: R38.*

**FR-SET-023** · The sessions table MUST list session, context, server-supplied expiry and action, marking the current session and allowing revocation of others.
*Source: R38.*

**FR-SET-024** · The sessions area MUST state that sessions are not trusted devices, that no session grants a TOTP bypass, and that **expiring a session does not cancel an accepted run**.
*Source: R38; FR-IDN-067.*

## F.4 Tenant members

**FR-SET-030** · Tenant member settings MUST present members, pending invitations and tenant policy as separate views.
*Source: R13.*

**FR-SET-031** · The member list MUST show member, tenant role, membership state (including whether TOTP is required) and the administration action available.
*Source: R13.*

**FR-SET-032** · The surface MUST state the last-Owner protection and that ownership transfer rechecks current membership at confirmation.
*Source: R13; FR-IDN-095.*

**FR-SET-033** · The surface MUST state that **membership is not project access**, and MUST link to project grant review.
*Source: R13; FR-IDN-092.*

**FR-SET-034** · The surface MUST state that removing a membership revokes that tenant's access while retaining authored evidence under policy, and that global identity controls (another person's password or MFA) are **not** available to a tenant administrator.
*Source: R13; FR-IDN-049.*

## F.5 Tenant policy

**FR-SET-040** · Tenant policy MUST be a **versioned record** configurable by Owner or Admin, with auditable changes.
*Source: N06.*

**FR-SET-041** · Policy MUST include human-approval configuration: `requireDifferentApprover` with its current value and the explicit statement that solo ownership never approves a bundle automatically — a person must hold the project Reviewer grant and record a deliberate C04 decision.
*Source: N06; FR-APR-021.*

**FR-SET-042** · A policy change MUST require a **rationale**, and the change record MUST capture actor, previous value, new value, rationale and policy revision.
*Source: N06.*

**FR-SET-043** · Policy MUST include identity requirements: TOTP mandatory for Owners and Admins (not switchable off), and an optional tenant policy to require MFA for Members. It MUST state that only local verified email and local accounts are supported, with **no active SSO, SCIM or custom-role controls**.
*Source: N06.*

**FR-SET-044** · Policy MUST display authority boundaries: who can configure; who can read project content; and that Admin does not imply Author, Reviewer, Viewer or `canRun`.
*Source: N06.*

**FR-SET-045** · Policy MUST display configured retention and usage / storage limits by reference to the policy record, asserting **no default duration or limit**.
*Source: N06; FR-X-046.*

**FR-SET-046** · Policy MUST present the three-scope summary: tenant membership (Owner / Admin / Member, no implicit project read) · project content (Author / Reviewer / Viewer, `canRun` independent) · global identity (self-service and verified recovery, **no global customer Admin**).
*Source: N06.*

**FR-SET-047** · Tenant policy MAY only **tighten** platform security boundaries, never weaken them.
*Source: register identity completion; N06.*

## F.6 Project settings and grants

**FR-SET-050** · Project settings MUST present configuration, approval policy, AI restrictions and lifecycle as separate views, with configuration saved as a new revision (FR-PRJ-020 – FR-PRJ-025).
*Source: R16.*

**FR-SET-051** · Project AI restrictions MUST be able to **tighten, never expand**, tenant and operator permissions.
*Source: N14; register §7.*

**FR-SET-052** · Project grant administration MUST behave per FR-IDN-098, showing each member's project grants, `canRun` and tenant role, and stating that current permission is checked at each protected action.
*Source: R17.*

## F.7 Integrations and delivery profiles

**FR-SET-060** · Integration settings MUST present source connections, delivery profiles and protected references, per FR-PRJ-090 – FR-PRJ-094.
*Source: R18.*

**FR-SET-061** · A `DeliveryProfileRevision` MUST carry: tenant / project / connection; provider family and `adapterContractVersion`; remote project and issue type; field and entity mapping; payload template and capabilities; auth grant reference; redaction, attachment, retry and reconcile policy; and per-capability verification evidence.
*Source: N40; GAP-BE; GAP-ALIGN.*

**FR-SET-062** · A delivery provider family MUST be enabled only by its own adapter contract and evidence. **`JIRA_CLOUD_ISSUES`** is the reference family. Support MUST be stated per enabled family and capability, never as a blanket claim.
*Source: N40; R18; register §13.1 finding 9.*

**FR-SET-063** · **Saving a delivery profile MUST send nothing.** It MUST NOT post a comment, create a defect, upload an attachment, open a pull request or run a test. This MUST be stated on the surface.
*Source: N40.*

**FR-SET-064** · Each delivery capability MUST be **individually gated** by its own required evidence (published API, auth scope, schema fixture, field mapping, permission, policy), with its state and the explicit effect if unverified. A capability without evidence MUST be **unavailable, not best-effort**.
*Source: N40.*

**FR-SET-065** · Entity and field mapping MUST be explicit. **An unmapped source entity MUST surface as `UNMAPPED` and MUST NEVER be mapped to a nearest match.** A missing mapping is a visible gap, exactly as in scenario authoring.
*Source: N40.*

**FR-SET-066** · Redaction MUST remove secrets **before any payload is built**. Attachments MUST be sanitized artifacts only, with no inline untrusted HTML or SVG.
*Source: N40; FR-X-032.*

**FR-SET-067** · Retry MUST be limited to **confirmed-safe failures** and MUST NEVER retry a confirmed or unknown write. Reconciliation MUST use an owned marker or receipt lookup, and only where the provider proves dedupe; otherwise the item MUST go to manual hold. **Dedupe-on-retry MUST be disabled unless the provider proves an owned marker or receipt.**
*Source: N40; GAP-BE; GAP-FAILURE.*

**FR-SET-068** · Provider families that are not enabled MUST be listed with their status, the reason, and the evidence that would enable them (I.6), and MUST be disabled.
*Source: N40; R18.*

**FR-SET-069** · A new profile revision MUST NOT change delivered history: past delivery intents MUST keep the profile revision they were published under, with receipts, payload hashes and reconciliation state frozen to that revision.
*Source: N40.*

**FR-SET-070** · The delivery identity MUST be separate from the source connection identity and MUST NEVER be the console identity.
*Source: N40; R18.*

## F.8 AI policy and usage — C12

**FR-SET-080** · The AI settings surface MUST present policy and funding, usage and reservations, and change history.
*Source: N14.*

**FR-SET-081** · Operator allocation MUST be displayed **read-only**, showing: platform-funded status (default **DISABLED**), the `TenantAIEntitlement` record or its absence, allocated platform usage, and explicitly **who may authorize or raise the platform cap — PlatformOperator only, with an audited entitlement decision**.
*Source: N14; GAP-OPERATOR; FR-OPS-040.*

**FR-SET-082** · The surface MUST state that Owner / Admin may **lower** policy limits or select approved BYOK, and **can never increase** the operator-funded cap or platform service caps.
*Source: N14; GAP-BE.*

**FR-SET-083** · A capability allowlist MUST be shown per configuration alias with its capability and effective state, labelled as illustrative aliases rather than vendor or model availability claims.
*Source: N14.*

**FR-SET-084** · Provider and funding MUST be explicit: approved provider configuration; a write-only credential reference; and an explicit funding mode — platform-managed **or** BYOK — with the credential owner named. **Silent provider, model or funding fallback MUST NOT exist.**
*Source: N14; register §7.*

**FR-SET-085** · Two policy switches MUST exist: block requests when the selected provider or funding is unavailable; restrict egress to sanitized authorized project context.
*Source: N14.*

**FR-SET-086** · The surface MUST state that **BYOK does not bypass service caps**, and that the effective policy is the **intersection** `operator ∩ tenant ∩ project ∩ request`, disabled until all authorities allow the request.
*Source: N14; GAP-BE; GAP-ALIGN.*

**FR-SET-087** · The surface MUST state plainly that **AI quota is not run capacity**: `ExecutionCapacityPolicy` is separate.
*Source: N14; GAP-BE.*

**FR-SET-088** · Usage reconciliation MUST show per request / funding: estimate versus actual, and reconciliation state. **Estimate MUST NOT be presented as provider actual.** Unknown usage MUST be retained for reconciliation and MUST NEVER be zeroed or silently funded elsewhere.
*Source: N14; register §7.*

**FR-SET-089** · Budget MUST be reserved **atomically** across chat, planning, composition, review, analysis, retries and applicable indexing. Reservation MUST occur before external I/O; the call MUST be made outside the transaction; estimates MUST be reconciled against provider-reported actuals. An ambiguous timeout MUST NOT release a reservation before reconciliation.
*Source: register §7; GAP-BE.*

**FR-SET-090** · Context, tool fan-out, output and time MUST be bounded.
*Source: register §7.*

**FR-SET-091** · The surface MUST state that admission and request limits control exposure but **do not promise mathematically exact financial caps**, and that no invoices or payment processing exist.
*Source: register §7; N14.*

**FR-SET-092** · Changing model, provider or funding MUST require an explicit authorized decision, audited without secret payloads.
*Source: N14.*

**FR-SET-093** · The surface MUST offer "continue manually" and state that manual authoring, pure static compile and the base report remain available when AI is off or quota is exhausted.
*Source: N14; FR-AUT-106.*

**FR-SET-094** · The surface MUST state that it is **not** a customer SuperAdmin or global-content portal.
*Source: N14.*

## F.9 Area F — verification requirements

Policy persistence and audit completeness · concurrent policy and grant changes · budget reservation and reconciliation races, and unknown-usage retention · provider-specific API, auth and schema fixtures for every gated delivery capability · partial write and lost-ACK reconciliation with owned-marker evidence · credential reference rotation without scope change · settings scope isolation (no cross-tenant or unauthorized project read) · WCAG 2.2 AA on the settings journeys.
*Source: register §12; GAP-BE; N40; N14.*

---

# G · Operator-side contracts — `FR-OPS`

**Scope.** The `PlatformOperator` principal and the versioned records it publishes, their validation, their fail-closed behaviour, their tenant-visible projection, and the hard boundary on what the operator may never do.

**Source of record.** Register `122:2` §15 (formerly `GAP-OPERATOR`, spec `227:2`) and register §13.2.

## G.1 The scope decision

**FR-OPS-001** · The platform operator surface MUST be an **internal, audited, out-of-band configuration path**. It MUST NEVER be exposed to customers, and it MUST NOT be a tenant role. Any operator tooling MUST write only through the record discipline of G.2.
*Source: GAP-OPERATOR; register §13.2.*

**FR-OPS-002** · `PlatformOperator` MUST be a **platform principal outside tenant membership and outside every project grant**. It MUST NEVER be grantable, delegable or escalatable from inside a tenant.
*Source: GAP-OPERATOR.*

**FR-OPS-003** · This authority MUST be additive: it MUST NOT renumber a contract, introduce a C13, add a tenant role, or change any existing boundary.
*Source: GAP-OPERATOR; register §13.3.*

## G.2 Record discipline

**FR-OPS-010** · Every operator decision MUST be an **immutable versioned record** with an id, a reason, an actor and a timestamp. The record set MUST be: `ManagedNetworkProfile`, `ExecutionCapacityPolicy`, `TenantAIEntitlement`, `ProbePolicy`, `IntakePolicy`, `ScanPolicy`. Operator configuration MUST NOT be a live mutable knob.
*Source: GAP-OPERATOR.*

**FR-OPS-011** · A running effect MUST reference the **exact version** it was admitted under.
*Source: GAP-OPERATOR; GAP-BE.*

**FR-OPS-012** · Configuration MUST be validated at **application startup and again at every effect boundary**, because a record can be revoked between the two.
*Source: GAP-OPERATOR; FR-X-046.*

**FR-OPS-013** · **Absent or invalid configuration MUST be `UNCONFIGURED` and MUST block the dependent effect.** No default MUST be assumed, no value MUST be invented, and no next-profile fallback MUST be attempted.
*Source: GAP-OPERATOR; GAP-BE; I6.*

**FR-OPS-014** · Every publication, change and revocation MUST be audited with actor, reason and version. The operator path MUST grant **no unrestricted access to customer content, source snapshots, artifacts or prompts**.
*Source: GAP-OPERATOR.*

**FR-OPS-015** · A tenant MUST see the **effect** of operator configuration and the authority responsible for it, **read-only**. A tenant MUST NEVER read another tenant's allocation, and MUST NEVER change its own ceiling upward.
*Source: GAP-OPERATOR; N14; N33.*

## G.3 ManagedNetworkProfile — C06

**FR-OPS-020** · `ManagedNetworkProfile` MUST be published per managed runner profile and MUST carry: `profileId`, `profileVersion`, `networkVersion`, the **actual egress CIDR or address identity**, the execution region, and the capability set.
*Source: GAP-OPERATOR; GAP-BE.*

**FR-OPS-021** · Until **every** required field carries a real published value, the profile MUST be `UNCONFIGURED` and **every admission that needs it MUST be blocked**.
*Source: GAP-OPERATOR; R19; N07.*

**FR-OPS-022** · A published egress identity is a commitment customers will place in a firewall. It MUST NEVER be a placeholder, an example or an inferred address.
*Source: GAP-OPERATOR.*

**FR-OPS-023** · **An execution region MUST NOT be presented as a data-residency statement.** All declared SaaS data locations (database, object store, logs, backups, AI) MUST separately satisfy tenant policy or eligibility MUST be blocked. No invented multi-region routing MUST exist.
*Source: GAP-OPERATOR; GAP-BE; R19.*

**FR-OPS-024** · Tenant region and location policy MAY select only published compatible profiles.
*Source: GAP-BE.*

**FR-OPS-025** · A separate, bounded `ProbePolicy` MUST govern reachability probing: explicitly configured protocol and path, bounded consent, no secret exposure, no test steps and no app-install side effects. A guessed health path MUST NOT exist.
*Source: GAP-BE; GAP-OPERATOR.*

**FR-OPS-026** · The system MUST state and uphold that an SUT 401 may prove **network reachability** and does not prove application correctness or a console sign-in failure.
*Source: GAP-BE; GAP-FE.*

## G.4 ExecutionCapacityPolicy — C05

**FR-OPS-030** · `ExecutionCapacityPolicy` MUST be an operator-versioned record of global and per-tenant ceilings: `maxActiveRuns`, `maxScenarioWorkerSlots`, `maxQueuedTickets`, `maxDeviceLeasesByPool`, `maxQueueWait`, with separate bounded probe and intake policies.
*Source: GAP-OPERATOR; GAP-BE; GAP-ALIGN.*

**FR-OPS-031** · A tenant or project MAY **tighten** its own ceiling. Neither MUST be able to raise it.
*Source: GAP-OPERATOR; GAP-BE.*

**FR-OPS-032** · Missing capacity configuration MUST **fail closed rather than default**, so an unconfigured platform refuses work instead of silently admitting unbounded work. **No numeric default MUST be invented.**
*Source: GAP-OPERATOR; GAP-BE.*

**FR-OPS-033** · `ExecutionCapacityPolicy` MUST be **independent of C12 AI budgets**. The two MUST NOT be conflated in the data model or in any surface.
*Source: GAP-BE; N14.*

**FR-OPS-034** · Suite fan-out MUST be bounded by worker slots and device leases, not by run count alone.
*Source: GAP-BE.*

## G.5 TenantAIEntitlement — C12

**FR-OPS-040** · `TenantAIEntitlement` MUST be an audited allocation record carrying: window, caps, funding source, approved provider capabilities, version, reason and status. The default MUST be **`DISABLED` with no allocated usage until a record exists**.
*Source: GAP-OPERATOR; N14.*

**FR-OPS-041** · Owner or Admin MAY lower limits or select approved BYOK **within platform service caps**. **Only the operator MUST be able to raise a funded cap.**
*Source: GAP-OPERATOR; N14; FR-SET-082.*

**FR-OPS-042** · There MUST be **no automatic funding fallback and no payment processor**.
*Source: GAP-OPERATOR; register §7.*

**FR-OPS-043** · Usage MUST be reserved atomically and reconciled per FR-SET-088 / FR-SET-089; unknown usage MUST be retained.
*Source: GAP-OPERATOR; GAP-BE.*

## G.6 What the operator can never do

**FR-OPS-050** · The following MUST be structurally impossible for a `PlatformOperator`, not merely disallowed by convention:

| | The operator MUST NEVER |
|---|---|
| 1 | Accept a semantic mapping, approve a bundle, or launch a Run. Acceptance, approval and Run remain human tenant decisions under C11, C04 and C05. |
| 2 | Mutate a C02 revision, a sealed C03 bundle, a C07 fact or an immutable Core verdict. |
| 3 | Read customer source snapshots, authored content, artifacts, prompts or secrets as a side effect of configuration. |
| 4 | Grant itself tenant membership or a project grant, or bypass RLS. Operator reads of operational metadata stay narrow, exactly as the scheduler cursor already does. |
| 5 | Raise a ceiling **retroactively** for work already refused — a refused intent was never accepted, and a later ceiling change MUST NOT resurrect it. |

*Source: GAP-OPERATOR.*

## G.7 Area G — verification requirements

Operator-record versioning and revocation races against in-flight admission · startup and effect-boundary validation with deliberately absent and malformed configuration · audit completeness without customer content · proof that a published egress identity matches the address a runner actually leaves from · tenant-visible read-only projection with no cross-tenant leakage · capacity ceiling intersection, reservation and unknown-usage races · no funded-cap escalation and no auto-funding.
*Source: GAP-OPERATOR; GAP-BE; register §12.*

---

# H · Execution, evidence and delivery — `FR-RUN`

**Scope.** Everything after a valid C04 approval: the explicit C05 Run request, admission and the queue, the reachability prerequisite, live monitoring, immutable facts and evidence, run history and lineage, and controlled delivery with per-item reconciliation.

**Screens.** R30 RunConfirmation · N30 ExecutionQueue · N32 QueuedRunDetail · N33 AdmissionBlocked · N31 EnvironmentReachability · R31 ExecutionLive · R37 ResultsFetchDegradedPanel · R32 ExecutionResults · N16 ArtifactDetail · R33 ExecutionHistory · N21 RunAcceptanceUnknown · N17 DeliveryJobs · N41 PartialDeliveryReconcile · responsive counterparts M07, M08 and M09.

**Source of record.** GAP-BE (fair admission, managed network descriptor) · GAP-CORE (admitted protocol execution) · GAP-FE (queue, connectivity, honest states) · GAP-FAILURE (queue, probe, asset and delivery recovery) · GAP-ALIGN (state axes) · GAP-OPERATOR (ceilings, profiles, entitlement) · register §13.1 findings 1–3, 7, 9 and §13.4.

## H.1 Execution contracts

**FR-RUN-000** · The execution-side contracts MUST behave as follows:

| Contract | Required behaviour |
|---|---|
| `C05 LaunchCommand` / `RunnerSubmission` | Atomic acceptance transaction creating an `AdmissionTicket`; idempotency on key + bodyHash; `202` means accepted, not started; a full queue rejects **before** acceptance. |
| `AdmissionTicket` + fair scheduler | Persisted eligible-tenant round-robin per compatible pool / profile, FIFO within a tenant; revalidation before lease; atomic reservation of run + worker slots + device leases with deterministic lock order and attempt / fence. `SKIP LOCKED` alone is **not** fairness. |
| Lifecycle `QUEUED → ADMITTING → RUNNING → COMPLETED` | `CANCEL_REQUESTED` may race completion. `ADMITTING → QUEUED` only with proof of no remote submit **and** a valid deadline. |
| `NOT EXECUTED` / `INDETERMINATE` | Queued cancel / reject / expiry = NOT EXECUTED, with no Core verdict. Possible execution with a lost ACK = INDETERMINATE. Never a fabricated Failed. |
| `C06 RunnerCapabilities` + admission | Profile / build / network version / capabilities; asset digests and current eligibility; actual leases and fence; `assessmentRef` with coverage and expiry. |
| `ReachabilityAssessment` | Bounded `ProbeJob` on a restricted worker: DNS / TCP / TLS and explicitly configured safe HTTP only. A fresh match is required at C05 and C06; expiry while QUEUED holds the ticket, and recheck happens only within the original consent, intent and deadline. |
| `C07` facts, journal, snapshot and replay | Immutable Core verdict only after execution; separate admission / infrastructure / cleanup / evidence axes; snapshot watermark and replay; a projection is never verdict authority. |
| `C09 DeliveryIntent` + `ItemReconciliation` | Frozen itemset; per-item `PENDING / CONFIRMED / UNMAPPED / RETRYABLE_FAILURE / PERMANENT_FAILURE / UNKNOWN`; **PARTIAL is never DELIVERED**; see FR-SET-061 – FR-SET-070 and H.8. |
| Core runtime resource ownership | One logical scenario owner; per-scenario browser context / HTTP state / JDBC connection / exclusive device lease; Playwright Java stays on its owner thread; LIFO cleanup; device quarantine on uncertain cleanup. |

*Source: GAP-BE; GAP-CORE; GAP-FAILURE; GAP-ALIGN.*

## H.2 The explicit C05 Run request

**FR-RUN-001** · Run MUST be a separate explicit human request, reachable only from a valid C04 approval. The surface MUST state that it requests **fair-queue admission, not an immediate worker start**.
*Source: R30; I1; GAP-FE.*

**FR-RUN-002** · The surface MUST display the exact pins that would execute: selection and accepted C11 scope; `environmentId` / `envRevision` / environment digest; C01 manifest and runtime pin; sealed bundle digest; C01-M `expandedHash`; typed `AssetRevision` content digests for fixture, locator, schema and public trust material; and policy / profile references. **Pins MUST NOT silently upgrade**, and an unknown or unavailable resolver MUST block compile or admission.
*Source: R30; R32; FR-APR-030.*

**FR-RUN-003** · An **admission prerequisite checklist** MUST be rendered. For each prerequisite it MUST state the current state, what is re-checked at admission, and one safe next step. The prerequisites MUST be at least: exact C04 and sealed bytes; managed profile and fresh assessment; assets, macros and trust eligibility; `ExecutionCapacityPolicy`.
*Source: R30; N33.*

**FR-RUN-004** · Where any prerequisite is unmet, the Run control MUST be disabled, the reason MUST be stated **once**, and the state MUST NOT be presented as a failure of the test.
*Source: R30; N33; I8.*

**FR-RUN-005** · "Check from managed profile" MUST be reachable from this surface and MUST NOT submit a Run. Completing a check MUST return the user to a deliberate Run request.
*Source: R30; N31; FR-PRJ-070.*

**FR-RUN-006** · Confirmation MUST be an affirmative acknowledgement that the user is requesting admission of **this exact approved bundle, with no runtime override**.
*Source: R30.*

**FR-RUN-007** · Acceptance MUST create an `AdmissionTicket` in `QUEUED` and MUST be reported as **202 accepted, not started**. The surface MUST state that no Core verdict exists while waiting.
*Source: R30; N32; GAP-BE.*

**FR-RUN-008** · The original idempotency key MUST be displayed and retained. On a lost acknowledgement the user MUST look up the original ticket by intent key and MUST NOT send a fresh Run. Save, Approve, chat, reconnect and browser reauthentication MUST never execute or duplicate the request.
*Source: R30; N21; N32; FR-X-020.*

**FR-RUN-009** · Coverage limitations and eligibility exclusions accepted at C11 MUST remain visible at Run. **No new exclusion MUST be introduced at Run**, and a changed selection MUST return to compile and review.
*Source: R30; FR-AUT-031.*

**FR-RUN-010** · Every launch MUST originate from an explicit human C05 Run or from a `C05-S` service principal under I.4. No other path MUST create a Run.
*Source: register §10; register §13.1 finding 1.*

## H.3 Admission, the queue and refusal

**FR-RUN-020** · The execution queue MUST show only the viewer's own tenant, project and compatible-pool tickets, grouped as **Queued and held · Admitting · Active · Ticket history**, filterable by project scope, compatible pool and state.
*Source: N30.*

**FR-RUN-021** · Each queue row MUST carry: ticket and own scope; state and `holdReason`; `queueDeadline` and the governing policy version; and one safe action.
*Source: N30.*

**FR-RUN-022** · **No global queue rank, no other tenant's identity or demand, and no ETA MUST be shown**, on any queue surface or its responsive counterpart.
*Source: N30; N32; M09.*

**FR-RUN-023** · Where a deadline, ceiling or threshold is not supplied by validated configuration, the surface MUST say so explicitly and MUST NOT display an invented number.
*Source: N30; N32; FR-OPS-032; I6.*

**FR-RUN-024** · The queue MUST explain fair admission in product terms: eligible tenants rotate round-robin per compatible pool, original tickets are FIFO within a tenant and pool, and **queue-to-worker transition fulfils the original explicit human request** — it is not a machine-initiated new Run.
*Source: N30; GAP-BE; register §13.1 finding 1.*

**FR-RUN-025** · At admission the system MUST revalidate grants, approval, environment, typed assets, trust material and a fresh matching assessment **before** atomic leases, counters and fencing.
*Source: N30; N32; GAP-BE.*

**FR-RUN-026** · A queued-ticket detail surface MUST present: the original human intent (ticket and idempotency key, requester, acceptance, approval lineage); the pins carried unchanged to the worker; why it is waiting (`holdReason`, policy in force, FIFO position within own scope, `queueDeadline`, `projectionVersion`); and the lifecycle legend.
*Source: N32; M09.*

**FR-RUN-027** · A `projectionVersion` MUST prevent a stale projection from regressing a newer state.
*Source: N32.*

**FR-RUN-028** · The lifecycle legend MUST map each state to its meaning, whether a Core verdict exists, and what the user may do: `QUEUED · eligible` · `QUEUED · held` · `ADMITTING` · `CANCEL_REQUESTED` · `CANCELLED / REJECTED / EXPIRED` = **NOT EXECUTED** · `INDETERMINATE`. Lease expiry MUST NOT be presented as proof that execution stopped.
*Source: N32; M09; I8.*

**FR-RUN-029** · Cancellation MUST apply to the user's **own queued ticket, never to a worker**, MUST require current project authorization rechecked by the backend, and MUST be presented as possibly racing completion and never as rollback.
*Source: N30; N32; R31.*

**FR-RUN-030** · Where C05 acceptance is **refused**, the surface MUST state one reason once, MUST NOT stack alerts, and MUST state that no ticket was created, that nothing is queued or running, that no Core verdict exists, and that the approval, environment and pins are untouched.
*Source: N33.*

**FR-RUN-031** · The refusal surface MUST name the gate that refused, whether a ticket exists, and who can unblock it, for at least: queue full; managed profile `UNCONFIGURED`; `ExecutionCapacityPolicy` unconfigured; assessment missing or expired; asset, macro or trust not eligible; device pool exhausted; grant or approval revoked; region policy unsatisfied.
*Source: N33.*

**FR-RUN-032** · A refused intent and a cancelled ticket MUST both read as **NOT EXECUTED**, MUST be excluded from any pass-rate denominator, and requesting Run again MUST be a new explicit human C05 decision.
*Source: N33; FR-PRJ-102; I8.*

**FR-RUN-033** · A tenant MUST see its own `ExecutionCapacityPolicy` effect read-only — active policy version, its own ceilings, device-lease and global-cap authority, and the statement that C12 AI quota is independent — and MUST NEVER see another client's data or be able to raise a ceiling.
*Source: N30; N33; FR-OPS-031; FR-OPS-033.*

## H.4 The reachability prerequisite

**FR-RUN-040** · The reachability surface MUST be reached from the environment (B.6) and MUST present the assessment for **one pinned `envRevision`** as an explicitly consented check that answers reachability for one tested path at one time, and nothing else.
*Source: N31.*

**FR-RUN-041** · Where the managed profile is `UNCONFIGURED`, the surface MUST state that no check can be consented to or performed, MUST show which descriptor fields are unpublished, MUST state that only the platform operator can publish them, and MUST NOT infer an allowlist address, region or capability.
*Source: N31; FR-OPS-020; FR-OPS-021; FR-PRJ-072.*

**FR-RUN-042** · The assessment record MUST display pinned environment, probe policy version, consent state, `assessedAt` and expiry, and coverage — rendered `UNCHECKED` when absent.
*Source: N31.*

**FR-RUN-043** · The full stage list MUST be presented **before any outbound packet**, and consent MUST cover exactly that list and nothing else: DNS resolution of the named aliases from the published egress; TCP to configured ports only; TLS handshake validated against the pinned `PublicTrustMaterialRevision`; HTTP on the **explicitly configured safe path only** — no guessed health endpoint, no secret, no test step.
*Source: N31; FR-OPS-025.*

**FR-RUN-044** · A check MUST NEVER install an application, submit a Run or produce a Core verdict.
*Source: N31.*

**FR-RUN-045** · Per-target outcomes MUST carry coverage. **Partial coverage MUST NEVER be summarised as reachable.** "Reachable" MUST NOT be rendered as "connected" or "healthy", and a target returning HTTP 401 MUST be stated as proof of network reachability only — not application correctness and not a console sign-in result.
*Source: N31; FR-OPS-026.*

**FR-RUN-046** · A missing assessment MUST block **Run only**. An expired assessment MUST hold a queued ticket, and rechecking MUST be permitted only within the original intent, consent and deadline.
*Source: N31; N32; FR-PRJ-070.*

## H.5 Live monitoring

**FR-RUN-050** · Monitoring MUST be non-blocking. Navigation, tenant switch, console session expiry and browser reauthentication MUST NEVER cancel, duplicate or replay an accepted job; reauthentication MUST grant inspection only.
*Source: R31; M07; M09.*

**FR-RUN-051** · Lifecycle and Core verdict, evidence, delivery and optional AI analysis MUST be displayed as **separate axes that never collapse into one label**. While running, the verdict MUST read "no final verdict · last confirmed snapshot".
*Source: R31; GAP-ALIGN; I8.*

**FR-RUN-052** · The live view MUST be labelled a projection. On stream loss it MUST state that **the stream disconnected, not the run**, that displayed facts may be stale, and that recovery is retrieving an authorized snapshot and replaying events from the watermark — never replaying Run.
*Source: R31; M07; GAP-FE.*

**FR-RUN-053** · The admission context MUST remain visible while running: ticket and original intent; the queue → admission → worker transition with pins unchanged; own policy and compatible pool; the retained admission-check snapshot; and a link to queue and hold history.
*Source: R31; M07.*

**FR-RUN-054** · Step rows MUST distinguish a known fact, an in-progress step whose snapshot may be stale, and a pending dependency. An unevaluated step MUST NEVER be rendered as success.
*Source: R31; R32; I8.*

**FR-RUN-055** · A target HTTP 401 MUST be rendered as an SUT response and never as console session expiry. Stream loss MUST NEVER be rendered as a failed assertion.
*Source: R31.*

**FR-RUN-056** · Cancellation from the live view MUST be presented as separate from lifecycle and verdict, MUST await the authoritative terminal state, and MUST route an `INDETERMINATE` outcome to reconciliation rather than to an assumption of completion.
*Source: R31; M07; GAP-FAILURE.*

## H.6 Facts, evidence and artifacts

**FR-RUN-060** · The results surface MUST present the immutable C07 outcome for a run and attempt with **lifecycle, Core verdict, evidence completeness and C09 delivery as four separate statements**.
*Source: R32; M08; GAP-ALIGN.*

**FR-RUN-061** · Facts, evidence, AI advice and delivery MUST be separately navigable. **Missing or unretrievable evidence MUST NEVER rewrite the sealed Core verdict.**
*Source: R32; N16.*

**FR-RUN-062** · Each step row MUST carry the recorded fact, expected versus actual where applicable, the evidence reference and the context. "Not evaluated — dependency not met" MUST NEVER be rendered as success.
*Source: R32; M08; I8.*

**FR-RUN-063** · AI interpretation MUST be optional, absent unless requested, and labelled advice or hypothesis, never fact. The base report MUST remain available without AI. Where no `TenantAIEntitlement` exists the surface MUST state that platform-funded AI is `DISABLED` with no operator allocation, and MUST offer continuing with the base report. **There MUST be no provider or funding fallback.**
*Source: R32; M08; FR-OPS-040; FR-SET-080.*

**FR-RUN-064** · The exact bundle pins MUST be reproduced on the results surface as they were at Run, and MUST be labelled dependency metadata rather than proof of a published runtime capability.
*Source: R32; R30.*

**FR-RUN-065** · A coverage boundary accepted at C11 MUST be restated with the result: an API-only selection MUST state that API facts cannot prove Web visibility, and the uncovered criterion MUST remain named.
*Source: R32; M08; FR-AUT-021.*

**FR-RUN-066** · NOT EXECUTED tickets MUST be stated as belonging to ticket history, not to these execution facts.
*Source: R32; N33.*

**FR-RUN-067** · A new Run requested from results MUST require separate C05 confirmation, current `canRun`, a valid C04, unchanged pins and current admission prerequisites.
*Source: R32; I1.*

**FR-RUN-068** · An artifact MUST have an identity: run and attempt, step identity, `environmentId` / `envRevision`, media digest, redaction record, and availability under the current retention policy and project authorization.
*Source: N16.*

**FR-RUN-069** · Artifact preview MUST escape content as text. **Customer HTML and SVG MUST NEVER be rendered inline**; unsupported active media MUST be download-only after authorization; every download MUST recheck permission; and **no permanent public artifact URL MUST be exposed**.
*Source: N16; FR-X-032.*

**FR-RUN-070** · A missing artifact MUST be shown as missing, with the base report still available carrying a missing-artifact notice. **A retrieval retry MUST NOT execute a test or create an attempt.**
*Source: N16; R37.*

**FR-RUN-071** · Where the retrieval service does not return a current snapshot, the degraded panel MUST state that this is **not a failed-run verdict**, MUST offer retry of retrieval only, MUST NOT imply a fixed retry count, and MUST state that the current lifecycle remains unknown until an authoritative snapshot arrives.
*Source: R37.*

## H.7 Run history, lineage and unknown acceptance

**FR-RUN-080** · Run history MUST list explicit user intents, accepted runs and attempts in the current project, searchable by run, intent or attempt identity and filterable by lifecycle state and environment revision, with **Runs · Pending acceptance · Delivery jobs** views. It MUST state that there is **no implicit schedule**.
*Source: R33.*

**FR-RUN-081** · Each history row MUST carry run and intent lineage, lifecycle and verdict, evidence completeness and destination delivery as separate columns — including "acceptance unknown / no run" and "cancelled / incomplete, known facts retained".
*Source: R33.*

**FR-RUN-082** · Selected lineage MUST chain author revision → bundle → human approval → Run intent → run → attempt → environment revision.
*Source: R33.*

**FR-RUN-083** · The surface MUST state that retrying fact retrieval or delivery never creates an execution, and that a rerun is a **new explicit intent with a valid approval**.
*Source: R33; register §13.1 finding 1.*

**FR-RUN-084** · Where the Run response did not arrive, the surface MUST state that acceptance is unknown and execution may already have been accepted, MUST retain and display the original intent key for lookup, MUST offer an authoritative status lookup rather than a second submission, and **MUST NEVER automatically resubmit Run**.
*Source: N21; GAP-FAILURE.*

**FR-RUN-085** · **No exactly-once external-effect guarantee MUST be implied** anywhere in this flow.
*Source: N21.*

## H.8 Delivery jobs and item reconciliation

**FR-RUN-090** · Delivery MUST be presented as its **own outcome**. The immutable Core verdict and the base report MUST be shown as unaffected by any delivery state or delivery action.
*Source: N17; N41; FR-SET-061.*

**FR-RUN-091** · The delivery job view MUST show the typed provider family and the immutable `DeliveryProfileRevision` it was published under, the remote project, issue type and source entity mapping, and MUST state that provider contract tests and dedupe or permission proof are **activation gates, not claimed built proof**.
*Source: N17; FR-SET-062; FR-SET-064.*

**FR-RUN-092** · Each item MUST carry entity, mapping and payload, receipt and state, and one next safe step, across `CONFIRMED` · `UNMAPPED` · `UNKNOWN` · `RETRYABLE_FAILURE` · `PERMANENT_FAILURE`.
*Source: N17; N41.*

**FR-RUN-093** · A parent `PARTIAL` MUST read **"not delivered"** and MUST remain `PARTIAL` until every required item has an authoritative outcome. A confirmed item MUST NEVER be read as proof that another item was delivered.
*Source: N17; N41; FR-SET-067.*

**FR-RUN-094** · The reconciliation surface MUST show the **frozen intent**: delivery intent and correlation, profile revision, the itemset frozen at publication with planned and excluded counts, frozen payload hashes, and authority rechecked at the moment of action. **The itemset MUST NOT grow during reconciliation** — a new item requires a new human publication.
*Source: N41.*

**FR-RUN-095** · For every item the surface MUST state **what a retry would do** before the user acts, including "nothing" for confirmed, unmapped and permanently failed items, and "blocked" for unknown items.
*Source: N41.*

**FR-RUN-096** · An `UNKNOWN` item MUST be **held** wherever the provider contract cannot prove dedupe. A typed remote ID alone MUST NOT be treated as proof of a write, and silence MUST NOT be treated as proof of absence.
*Source: N41; N17; FR-SET-067.*

**FR-RUN-097** · Retry scope MUST be stated before confirmation: exactly which items would be resent, which would not be touched, that the payload is identical frozen bytes under the recorded redaction policy, and that the parent state remains `PARTIAL` afterwards.
*Source: N41.*

**FR-RUN-098** · Retrying a delivery, reconciling a receipt or fixing a mapping MUST NEVER execute a scenario, change a verdict or weaken an assertion. **No Run, pull request, Xray `TestExecution` or `TestRun` creation MUST be available from any delivery surface.**
*Source: N17; N41; FR-SET-063.*

## H.9 Area H — verification requirements

Fair-admission behaviour under real contention across tenants, pools and device leases · atomic reservation, deterministic lock order, fencing and attempt accounting · `ADMITTING → QUEUED` only with proof of no remote submit · lost ACK, restart, replay and reconciliation without double execution · cancellation racing completion and cancellation during writes · partial cleanup and device reuse after uncertain cleanup · snapshot watermark, cursor recovery and stale-projection regression · tenant isolation of streams, artifact downloads and device leases · artifact retrieval authorization, redaction and absence of a durable public URL · delivery idempotency, receipt reconciliation and proof that a confirmed or unknown write is never resent · WCAG 2.2 AA on the Run, queue, monitoring, results and delivery journeys.
*Source: GAP-BE; GAP-CORE; GAP-FAILURE; register §12.*

---

# I · Extended capabilities — `FR-EXT`

**Scope.** The capabilities that extend the core product without changing its authorities, contracts or invariants: repository-informed proposals, federated identity, phishing-resistant factors, machine-initiated execution, customer-network execution, additional provider families, cross-project reuse, client certificate identity, AI and retrieval extensions, and commercial billing.

**Rule for the whole area.** Every capability here reuses the C01–C12 contract set and the invariants in §0.4 unchanged. None introduces a new execution authority, a new approval path or a new contract number.

**Source of record.** Board 8 (repository-informed proposals) · R06 and R40 reference frames · register §1, §7, §9, identity completion and §13.1 findings 1, 7, 8, 9 · GAP-ALIGN preserved platform scope · N40 provider-family table.

## I.1 Repository-informed scenario proposals

**FR-EXT-001** · Repository intake MUST be **read-only evidence acquisition** from GitHub or Bitbucket through BE provider ports. The user explicitly selects repository, ref, allowed paths and project scope; the FE submits intent only; the BE records a durable import request and enforces authorization, read scope, size limits and server-side credential references. Provider tokens MUST NOT be persisted in the browser.
*Source: board 8 §01; board 8 entry contract.*

**FR-EXT-002** · The BE MUST resolve the ref **once** to an immutable commit and MUST pin commit, path and content hash for everything fetched, filtering by supported type, size and sensitive data, with pagination and backoff.
*Source: board 8 §02.*

**FR-EXT-003** · Repository intake MUST NEVER execute repository code, run builds or hooks, install dependencies, trigger CI, write, commit, create branches or open pull requests, and MUST NOT require an automatic webhook or sync subsystem.
*Source: board 8; register §1.*

**FR-EXT-004** · Supported artifacts found in a repository (documentation, user stories and acceptance criteria, OpenAPI, Postman, Gherkin) MUST reuse the versioned evidence schema of D.5: parser version, source revision and hash, citation spans, capture time and freshness, facts and conflicts. Unsupported or filtered evidence and the inspected-scope boundary MUST be retained explicitly.
*Source: board 8 §03A; FR-AUT-050.*

**FR-EXT-005** · Code excerpts MUST be allowlisted, bounded and **cited context only — never the business-assertion oracle**. General AST analysis, arbitrary code-path following, generated Core handlers and Postman-script execution MUST NOT exist. "No test evidence found in the inspected scope" MUST NOT be presented as "the functionality has no tests".
*Source: board 8 §03B.*

**FR-EXT-006** · Requirements and client criteria MUST ground assertions. Already-valid `.feature` or catalog mappings MUST go through deterministic normalization to an explicit draft apply, bypassing AI. Narrative or unresolved evidence MUST go to optional bounded AI assistance or to client clarification and manual mapping, and an AI outage MUST NOT block the manual path.
*Source: board 8 §03; FR-AUT-001; FR-AUT-106.*

**FR-EXT-007** · Repository proposals MUST be C08 `ProposalEnvelope`s pinned to tenant, project, source, base revision, catalog and policy. A stale base or catalog MUST require explicit rebase and review, never silent rebinding. Explicit client Apply MUST create a new BE-authoritative C02 revision under optimistic concurrency, preserving occurrence IDs, typed references, macro versions and provenance. Gherkin MUST remain import, view and compiled output — never a second editable authority.
*Source: board 8 §03; FR-AUT-020.*

**FR-EXT-008** · Repository content MUST be treated as untrusted: redaction before AI or index persistence, prompt-injection defence, constrained outbound destinations, size, type and archive limits, path-traversal, XXE and SSRF defences, authorized downloads only, and no inline untrusted HTML or SVG.
*Source: board 8 guardrails; FR-X-032.*

**FR-EXT-009** · Every inspection MUST end as **complete, partial or failed**, with its exclusions. Denied or expired access, missing refs or content, rate limits, interrupted pagination and parser or filter failures MUST be surfaced. Partial input MUST NEVER imply exhaustive coverage, and a bounded retry MUST retain the pinned commit and MUST NOT Apply or Run.
*Source: board 8 inspection failures.*

**FR-EXT-010** · A requested refresh MUST create new evidence and a new revision and MUST NEVER mutate approved history. Revoking a source grant MUST stop new AI context and block dependent new approval or use pending policy review. Late provider callbacks MUST NOT resurrect data of a tenant being deleted.
*Source: board 8 refresh and revocation; FR-PRJ-041; FR-IDN-125.*

**FR-EXT-011** · Repository intake MUST hand off to the shared C02 → C07 lifecycle with no new execution authority, and C09 MUST grant **no repository-write permission**.
*Source: board 8 shared lifecycle; board 8 §C08 + C09.*

**FR-EXT-012** · Further repository providers MUST be added as extension ports with their own provider contracts. Repository evidence MUST NOT be presented as coverage detection or behaviour-to-assertion traceability.
*Source: board 8 entry boundary; board 8 reading notes.*

## I.2 Federated identity

**FR-EXT-020** · Federation MUST be implemented with Spring Security–supported protocols on an `issuer + subject` identity, validating state, nonce, signature, audience, expiry and redirect, with PKCE where applicable.
*Source: register identity completion; FR-IDN-001.*

**FR-EXT-021** · Account linking MUST be **explicit** and bound to an invitation or tenant. **Email domain or SSO success alone MUST NEVER grant tenant membership**, and no automatic joining by email domain MUST exist.
*Source: register identity completion; M03.*

**FR-EXT-022** · A federation error MUST NEVER reveal whether another identity exists.
*Source: R06.*

**FR-EXT-023** · Federation MUST NOT bypass required MFA policy for Owner and Admin, and MUST NOT replace tenant membership and project grants as the source of authority.
*Source: register §3; FR-IDN-023; R06.*

## I.3 Phishing-resistant factors and trusted devices

**FR-EXT-030** · WebAuthn security keys and passkeys MUST be offered as **MFA factors under the same required-MFA policy** as TOTP. Factor replacement MUST preserve required MFA and independently reviewed recovery controls.
*Source: R40; register identity completion.*

**FR-EXT-031** · **No trusted-device mechanism MUST bypass required MFA** for an Owner or Admin.
*Source: M02; R40; FR-IDN-069; register §13.1 finding 7.*

**FR-EXT-032** · A factor or federation method that is not active MUST NOT appear in sign-in, enrollment or navigation. A regression guard MUST prevent inactive identity controls from becoming reachable.
*Source: R06; R40; GAP-ALIGN finding 7.*

## I.4 Machine-initiated execution

**FR-EXT-040** · CI or pipeline triggers, scheduled or recurring runs, and any customer-facing launch API MUST launch **only through a `C05-S` service principal**.
*Source: register §10; register §13.1 finding 1.*

**FR-EXT-041** · A `C05-S` principal MUST only re-launch an **already human-approved exact `bundleDigest` and `envRevision`**, under its own `canRun` grant, its own idempotency key and its own concurrency ceiling.
*Source: register §10; register §13.1 finding 1.*

**FR-EXT-042** · A `C05-S` principal MUST NEVER approve, alter bytes or widen its own scope.
*Source: register §10.*

**FR-EXT-043** · Trigger absence, replay and idempotency guards MUST ensure no Run is created autonomously. Queue-to-worker movement of a human ticket fulfils that human's intent and MUST NOT be classed as machine-initiated.
*Source: GAP-ALIGN finding 1; N30.*

## I.5 Customer-network execution

**FR-EXT-050** · Customer-side or private-network runners and warm browser pools MUST be subject to every invariant in §0.4; in particular approved bytes are the executed bytes (I7) and no AI path reaches a runner (I2).
*Source: register §1; §0.4.*

**FR-EXT-051** · An incompatible or unreachable target and profile union MUST fail admission. A scenario MUST NOT be silently distributed across runners or profiles.
*Source: register §1; FR-RUN-025.*

**FR-EXT-052** · The product MUST NOT promise that every private customer network is reachable, and MUST NOT present execution region as data residency.
*Source: register §1; FR-OPS-023.*

## I.6 Additional provider families

**FR-EXT-060** · Each source or delivery provider family MUST be enabled only by its own adapter contract, auth model, published API, schema fixtures and per-capability evidence, exactly as FR-SET-064 requires of `JIRA_CLOUD_ISSUES`.
*Source: N40; FR-SET-064.*

**FR-EXT-061** · Jira Server / Data Center MUST be enabled through provider-specific fixtures and tests for its distinct auth model and API surface.
*Source: N40.*

**FR-EXT-062** · Xray Cloud / Data Center MUST be enabled through a dedicated entity-mapping contract for its test, test-execution and test-run entities.
*Source: N40.*

**FR-EXT-063** · Native Slack and Teams adapters MUST be enabled capability by capability, each with its own evidence.
*Source: N40.*

**FR-EXT-064** · No provider family MUST be granted repository write, pull-request creation or Run creation, and a generic test-management-system claim MUST NOT be made.
*Source: N17; R18; board 8.*

## I.7 Cross-project reuse

**FR-EXT-070** · Sharing a macro or asset across projects MUST be by **copy** or by **explicit reference to an authorized immutable version**. A silent ownership move of approved content MUST NOT occur, and an alias MUST NEVER be repointed.
*Source: register §5; FR-PRJ-004; FR-CAT-071.*

**FR-EXT-071** · A cross-project reference MUST be a pin like any other: its replacement or revocation MUST invalidate dependent approvals per I7.
*Source: register §5; I7.*

## I.8 Client certificate identity

**FR-EXT-080** · Client mTLS, private keys, client identities and key containers, and application re-signing MUST NOT introduce a TLS bypass or a global trust mutation.
*Source: register §13.1 finding 8; GAP-ALIGN finding 8.*

**FR-EXT-081** · Private key material MUST be held only as scoped secret references, under FR-X-030 – FR-X-035, and MUST NEVER reach a prompt, log, index, artifact or report.
*Source: FR-X-031; register §13.1 finding 8.*

## I.9 AI and retrieval extensions

**FR-EXT-090** · Customer-hosted inference endpoints MUST be governed by C12 exactly like any provider: approved endpoint, egress rules, explicit funding and budget, and no silent provider, model or funding fallback.
*Source: register §7.*

**FR-EXT-091** · Vector indexes MUST be **derived and replaceable**, MUST NEVER be domain authority, and MUST honour tenant and project scope and source-grant checks before retrieval.
*Source: register §9; I4.*

**FR-EXT-092** · Python workers MUST be optional adapters behind existing Java BE application ports. They MUST NOT become a required runtime and MUST NOT hold domain authority.
*Source: register §1; GAP-ALIGN preserved scope.*

## I.10 Commercial billing

**FR-EXT-100** · A billing or subscription processor MUST consume the usage ledger and MUST NOT replace it. Platform-managed inference charges, customer quotas and charging, and BYOK provider charges MUST remain separate, with credential owner and funding mode always explicit.
*Source: register §7.*

**FR-EXT-101** · Billing MUST NOT create an automatic funding fallback and MUST NOT let a tenant raise an operator-funded cap.
*Source: FR-OPS-041; FR-OPS-042.*

## I.11 Area I — verification requirements

Provider contracts, permission and revocation, snapshot and hash lineage, secret screening, parser limits and no-execution isolation for repository intake · federation protocol validation, explicit linking and no membership-by-domain · factor replacement preserving required MFA, and a regression guard against inactive identity controls · `C05-S` trigger absence, replay and idempotency, and proof it can never approve or widen scope · customer-network runners under the same admission and pin invariants · provider-specific API, auth and schema fixtures per family · cross-project pin invalidation · absence of TLS bypass and global trust mutation · vector-index scope and grant checks · billing separation from the usage ledger.
*Source: board 8 implementation gates; GAP-ALIGN findings disposition.*

---

# J · Traceability

| Area | Source at derivation (23 October 2026; the `GAP-*` sections were consolidated into boards 1–7 and register `122:2` §15 on 1–2 November 2026) | Primary board source | Primary screens |
|---|---|---|---|
| A · Identity and tenancy | GAP-BE (roles, RLS); GAP-FE (tenant switch); register §3, §4, identity completion | Board 1 §03–§05 + completion; board 3 completion | R02–R08, R38, R39, R41, R34–R36, N01–N05, N19, N20 |
| B · Project and suite structure | GAP-BE (ERD, assets); register §5, §9, identity completion | Board 1 §03 + completion; board 3 completion | R14–R17, R11, R18–R20, N07, R21–R23, N15 |
| C · Step catalog and library | GAP-CORE; GAP-AUTHOR; GAP-ALIGN (C01-M, C01-A); register §13.1 findings 4–6, 8 | Board 2 §01–§03 + completion; board 3 §05 | R25, R26, N36, N37, N38, N34, N35, N39 |
| D · Authoring and review | GAP-AUTHOR; GAP-IMPORT; GAP-FE; register §6, §7, §8 | Boards 5 and 6; board 3 §01–§03; board 2 §02 | R24, N09, R27, N08, R28, R29, N10, N11, N13 |
| E · Approval path | GAP-ALIGN (C04 row); register §8, §11 | Board 3 §04–§05; board 7 step 08 | N12, N11, N18 |
| F · Settings | GAP-BE (delivery, entitlement); GAP-FE; register §3, §7, §9 | Board 1 §06; board 3 completion | R12, R38, R13, N06, R16, R17, R18, N40, N14, N15, N18, N19 |
| G · Operator contracts | Register `122:2` §15 (formerly **GAP-OPERATOR**); register §13.2–§13.3 | — (new material) | N14 and N33 projections only |
| H · Execution, evidence and delivery | GAP-BE (fair admission); GAP-CORE (admitted execution); GAP-FE (queue, honest states); GAP-FAILURE; GAP-ALIGN (state axes); register §13.1 findings 1–3, 7, 9 | Board 2 (runner durability); board 4; board 7 | R30, N30, N32, N33, N31, R31, R37, R32, N16, R33, N21, N17, N41, M07, M08, M09 |
| §1.6 Responsive | GAP-FE and GAP-ALIGN finding 11 (WCAG 2.2 AA) | Board 3 §07 | M01–M09, T01, T02 |
| I · Extended capabilities | Register §1, §7, §9, §10, identity completion; GAP-ALIGN preserved scope; register §13.1 findings 1, 7, 8, 9 | Board 8 | R06, R40, N40 provider table |
| A.11 Public landing | Board 3 (FE intent); register §1 | Board 3 | R01, M01 |

### Compact states N22–N29

These eight states are designed inside the D02 interaction atlas rather than as separate frames. They carry no requirements of their own; each is the rendering of a requirement stated elsewhere.

| State | Specified by |
|---|---|
| N22 CreateProjectDialog | FR-PRJ-011 (explicit initial grants, Owner or Admin) |
| N23 OwnershipTransferDialog | FR-IDN-095, FR-IDN-121, FR-SET-032 (last-Owner protection, recheck at confirmation) |
| N24 TenantSwitchDirtyDialog | FR-IDN-130, FR-IDN-131 (guarded transition, Save / Discard / Stay, original-scope save) |
| N25 StaleRevisionConflictDialog | FR-X-033, FR-AUT-032 (stale-revision problem code, no overwrite of a dirty draft) |
| N26 ApprovalInvalidatedDialog | E.4 in full, I7 (any execution-affecting pin change invalidates the approval) |
| N27 AIQuotaExceededPanel | FR-SET-087, FR-SET-093 (AI quota is not run capacity; continue manually) |
| N28 ProjectAccessDeniedPanel | FR-IDN-064 (refresh permissions or show denied scope, no loop) |
| N29 InvitationWrongIdentityPanel | FR-IDN-082, FR-IDN-083 (identity must match; expired, revoked or used invitations) |

**Where two sources disagree, the design authority order of `constitution.md` P16.1 governs.**

---

**AI proposes. Humans authorize. Core decides.**
