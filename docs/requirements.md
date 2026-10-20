# A·Spectra — Screen Inventory and Requirements Baseline

Status: all four parts complete; extended 2026-10-02 (ralph-loop, new run, iteration 1 of 3) with a
deep-dive pass on four areas the user named explicitly: the catalog screen (`StepsPage`), two-factor
authentication (`SecurityTab`/`MfaEnrollmentWizard`/`MfaChallengePage`), session expiration
(`useSessionPolicy`/`SessionExpiryWarningModal`/`endSession`), and error screens/messages
(`RouteErrorFallback`/`ForcedNotFoundPage`/the API error-toast catalog). That pass added one expanded
Part 1 entry (`StepsPage`), one new embedded-component Part 1 entry (`SecurityTab` +
`MfaEnrollmentWizard`), a new "Cross-cutting behaviors" section covering session expiration and error
screens/messages (deliberately outside the 33/34-screen count — see Methodology), four new rows in the
Part 2.2 AC-gap table, five new Part 3 FRs (`FR-AC-14` through `FR-AC-17`, `FR-UX-02`), and a third
mockup (Mockup C, catalog screen) added to the Part 4 HTML artifact. Non-MVP ideas surfaced along the
way are not in this document — they are in the separate `additional-requirements.md` at the workspace
root, per the user's explicit instruction to keep MVP and non-MVP material apart.

**This 2026-10-02 extension was itself then reviewed once by `requirements-analyst`**, the same
verification step the 2026-10-01 baseline went through. It found 7 issues, all since corrected: the
"not alphabetical" layer-order example named the wrong alphabetical ordering; `StepsPage.test.tsx`'s
case count was given as 6 instead of the actual 9; the session-expiration section wrongly claimed a
silent refresh is attempted before *every* forced logout (true only for `access_expired`, and even
there the warning is shown first, not after — the opposite of what an earlier draft of that paragraph
said, and inconsistent with `FR-AC-14` in the same pass, which had it right); the error-toast catalog's
"every toast carries a correlation id" claim missed one exception (`403 AUTHZ_DENIED`); and the claim
that no dedicated test exists for that catalog at all was wrong — three test files already cover most
of it, narrowing `FR-AC-17` to the three branches that are genuinely untested. One non-blocking note
(a "download as .txt" action on the recovery-codes screen) was also folded in.

Earlier history: Parts 1-3 were reviewed once by `requirements-analyst` (one numeric error in the route
count and two internal-consistency problems in the gap list were found and corrected; two
judgment-language sentences in Part 1 were rephrased as facts). All three originally-blocking questions
are resolved by explicit user decision (2026-10-01): **FR-PLAN-01** (Plan stays folded into Run,
documented in `.claude/CLAUDE.md` and `docs/diagrams/01-general-architecture.md`, no new screen),
**FR-SCOPE-01** (Swagger/Postman/user-story imports are part of MVP 1.0 — `.claude/CLAUDE.md` and
`.claude/ORCHESTRATION.md` updated), **FR-UX-01** (General tab hidden; Execution-defaults and
personal-Integrations tabs to be built, content scoped by a second `requirements-analyst` pass, two
blocking sub-questions resolved by the user — see FR-UX-01 for the final decision and its "next step":
call `architect`). Part 4 mockups for FR-UX-01's two build items are published (link in Part 4).
**Nothing in this document has been implemented in code — it records decisions and open next steps
only.**

**2026-10-03 extension**: the product owner requested Google SSO login for MVP 1.0 and, separately,
promoted every item in `additional-requirements.md` (14 items across the catalog screen, two-factor
authentication, session expiration and error screens) into MVP 1.0 scope. Each went through
`requirements-analyst`, then `architect` for cross-repo/BE-impacting items, with the user resolving every
blocking product question raised along the way. See `FR-AUTH-01`/`FR-AUTH-02` (Google SSO) and the
`FR-CAT-*`/`FR-MFA-*`/`FR-SESS-*`/`FR-ERR-*` block in Part 3 for the full set. A corresponding Figma design
pass is complete for every item except where explicitly marked "no Figma change needed." As with the rest
of this document, nothing here has been implemented in code.

Scope: `qa-platformFE` (the console SPA), as it exists in the current working tree. Factual baseline
only — what the code currently does, no judgment, no "should be" language, except where explicitly
marked as a gap or a proposal. Sourced from: `qa-platformFE/src/router/index.tsx`,
`qa-platformFE/src/router/routes.ts`, each page's own source file and adjacent test file, the e2e
specs under `qa-platformFE/e2e/`, `docs/diagrams/01-general-architecture.md`,
`docs/diagrams/04-fe-architecture.md`, and `qa-platformFE/.claude/CLAUDE.md` (which contains no
enumerated page list of its own to cross-reference — noted as a fact, not a defect).

This document has three completed parts and one deferred part:
1. **Screen inventory** (34 screens, after the 2026-10-02 addition of the `SecurityTab` +
   `MfaEnrollmentWizard` entry) — route, purpose, UI, actions, data source, states, and whether
   written acceptance criteria exist today. A "Cross-cutting behaviors" section follows Part 1 for two
   behaviors that reach every screen rather than owning one (session expiration, error
   screens/messages) — deliberately not counted in the 34, per Methodology below.
2. **Gap list** — the inventory mapped against the 8-stage flow from the workspace `.claude/CLAUDE.md`
   (Connect -> Import -> Canonical/Review -> Plan -> Run -> Evidence -> Sync -> Console).
3. **Functional requirements** — one FR per gap found.
4. **Mockups** (deferred) — HTML artifact mockups from real component code, for screens that need
   visual exploration rather than a spec/test correction.

## Methodology

- A screen below is a routed, user-navigable view, **or** a substantial embedded component that is a
  full tab/panel body of a routed page with its own non-trivial UI, data and actions (the precedent is
  `ProjectMembersTab`/`ProjectIntegrationsPage`; `SecurityTab`/`MfaEnrollmentWizard` follows the same
  precedent, added 2026-10-02) — but not every component. Route and purpose are taken from the router
  configuration and the page component's own source.
- A behavior that reaches every screen rather than belonging to one (session expiration, the
  route-error fallback, the API error-toast catalog) is documented in its own "Cross-cutting behaviors"
  section instead of as a numbered screen, and is not counted in the screen-inventory total.
- "States handled" quotes only what the code implements — nothing is assumed.
- "Acceptance criteria" (AC) means a test or spec that states an expected user-facing outcome,
  wherever it lives: a page-level unit test, a mocked or real-stack Playwright e2e spec, or an
  adjacent spec doc. A hook-level or component-level test that only proves implementation detail, with
  no page-level or e2e test proving the user-facing behavior, is recorded as **none at page level**.
- Four AC labels are used: **Yes** (page-level test and/or e2e coverage of the screen's behavior),
  **Partial/thin** (some coverage exists but narrow relative to the screen's surface), **None at page
  level** (only hook/component tests exist for pieces of it), **None** (no test of any kind found).

---

## 8-stage flow (from workspace `.claude/CLAUDE.md`)

| # | Stage | Entry point | Owner |
|---|---|---|---|
| 1 | Connect | `PUT /api/projects/{id}/integrations`, `POST .../integrations/detect` | BE `project` BC + connectors; FE `ProjectIntegrationsPage` |
| 2 | Import | `POST /api/projects/{p}/connectors/{c}/imports` | BE `ConnectorImportApplicationService` |
| 3 | Canonical + review | `CanonicalTestCaseApplicationService`, `SourceCaseNormalizer` | BE `testcase` BC |
| 4 | Plan + projection | `ExecutionPlanApplicationService` -> `JpaExecutableProjectionAdapter` | BE `testcase` -> legacy `TestScenario` |
| 5 | Run | `ExecutionApplicationService.launch` -> `ExecutionAsyncRunner` | BE `execution` BC + Core |
| 6 | Evidence | `BEStepReporter`, `ExecutionArtifactService` | BE `execution` BC |
| 7 | Sync | outbox -> `IntegrationSyncJobApplicationService` | BE `testcase` BC |
| 8 | Console | `ExecutionOverlayPage` + `useExecutionOverlayVM` | FE `features/overlay` |

## FE feature modules found in code (13 directories under `src/features`)

`admin`, `assistant`, `auth`, `catalog`, `dashboard`, `execution`, `imports`, `onboarding`, `overlay`,
`project`, `scenario`, `settings`. **Fact**: `docs/diagrams/04-fe-architecture.md` lists only 11
modules and omits `assistant` — `assistant` exists in code (panels embedded in five screens, see Part
1) but is not yet reflected in that architecture doc. `assistant` has no `pages/` directory — it is
never a routed screen, only embedded UI.

---

# Part 1 — Screen inventory (34 screens: 33 from the 2026-10-01 baseline + `SecurityTab`/
`MfaEnrollmentWizard` added 2026-10-02)

## `src/pages/landing`

### LandingPage — AC: **Yes**
- Route `/` — `src/pages/landing/LandingPage.tsx:92`. Stage: pre-auth (none of the 8).
- Purpose: public marketing/explainer page for unauthenticated visitors; sign-in entry point.
- UI: sticky header (logo, anchor nav, language switcher, sign-in link), hero with a `DeterminismProof`
  panel showing a real `.feature` file snippet (code comment: deliberate, "not a mocked console UI"),
  trust strip (Xray Server, Xray Cloud, TestRail), 8 content sections including a condensed 5-stage
  flow rail (`connect, import, review, execute, sync`), an explicit "what it is not" grid
  (`authoring, azure, ai, credentials`), FAQ accordion, footer.
- Actions: anchor nav, "Sign in" links -> `/login`.
- Data: none — fully static, i18n-driven.
- States: none (static page).
- AC: `src/__tests__/pages/LandingPage.test.tsx` asserts, per locale: no unsupported claims leak
  (`/Selenium|EUR|€|Testimonial|Pricing/` never appear, no "N% faster" claim), "Azure DevOps" appears
  exactly once in the "what it is not" section, never says "request access", exactly 5 flow steps,
  exactly one `<h1>`, exactly 6 FAQ entries. Also covered by `e2e/specs/axe.spec.ts` and
  `e2e/specs/reflow.spec.ts` (accessibility/responsive only, not functional).

---

## `src/features/auth`

### LoginPage — AC: **Yes**
- Route `/login` — `src/features/auth/pages/LoginPage.tsx:16`. Stage: pre-auth.
- Purpose: username/password sign-in; entry points to forgot-password, sign-up, federated sign-in.
- UI: two-pane layout, username/password inputs with visibility toggle, `FederatedSignIn`, conditional
  sign-up link gated on `signup.enabled`.
- Actions: submit -> `useLogin` (`POST /api/auth/login`); "Forgot password?" -> `/forgot-password`;
  conditional "Sign up" -> `/signup`.
- Data: `useLogin()`, `useSignupConfig()` (`GET /api/auth/signup`).
- States: `login.isPending` on submit; no explicit inline error block in this file (errors likely
  surfaced via toast).
- AC: `src/__tests__/features/auth/pages/LoginPage.test.tsx` — hero image fallback, required fields,
  password toggle, multi-language text, legal notice. Plus `axe.spec.ts`/`reflow.spec.ts`.

### ForgotPasswordPage — AC: **Yes**
- Route `/forgot-password` — `src/features/auth/pages/ForgotPasswordPage.tsx:11`.
- Purpose: request a password-reset email.
- UI: email input, submit, post-submit success panel, back-to-login link.
- Actions: submit -> `useRequestPasswordReset` (`POST /api/auth/password-reset/request`).
- States: `submitted` swaps form for a fixed anti-enumeration success message; `reset.isPending`.
- AC: `src/__tests__/features/auth/ForgotPasswordPage.test.tsx` — anti-enumeration success after 202,
  lost-mailbox guidance, rate-limit (429) toast.

### SignupPage — AC: **Yes**
- Route `/signup` — `src/features/auth/pages/SignupPage.tsx:23`.
- Purpose: request a new account; an administrator activates it later (not self-service).
- UI: email/full-name/organization inputs, terms checkbox with `safeHref`-gated legal links, submit,
  "not offered" notice when `config.enabled` is false, post-submit `PendingActivation` panel.
- Actions: submit (disabled until all required fields + terms) -> `useRequestSignup`
  (`POST /api/auth/signup`).
- Data: `useSignupConfig()` (`GET /api/auth/signup`) drives both the "not offered" state and legal URLs.
- States: `!config.enabled` -> "not offered"; `sent` -> `PendingActivation`; same outcome regardless of
  whether the address was new, existing or already pending (anti-enumeration, per code comment).
- AC: `src/__tests__/features/auth/pages/SignupPage.test.tsx` (6 cases) + `e2e/specs/signup.spec.ts`
  (S3: "a visitor asks for an account and is told what happens next"; legal docs link and open in a new
  tab; "a deployment that does not offer sign-up says so"; no accessibility violation).

### MfaChallengePage — AC: **Yes**
- Route `/login/mfa-challenge` — `src/features/auth/pages/MfaChallengePage.tsx:15`.
- Purpose: second factor after a password login that required MFA.
- UI: 6 digit inputs (auto-advance, paste-fills-all), recovery-code fallback toggle.
- Actions: 6th digit or paste auto-submits -> `POST /api/auth/mfa/verify`; recovery mode submit ->
  `POST /api/auth/mfa/recovery/use`.
- States: no `mfaChallengeToken` in the auth store -> redirect to `/login`; both mutations expose
  `isPending`.
- AC: `src/__tests__/features/auth/MfaChallengePage.test.tsx` — TOTP submit, recovery-flow switch,
  paste-fills-and-verifies.

### OidcCallbackPage — AC: **Yes**
- Route `/oidc/callback` — `src/features/auth/pages/OidcCallbackPage.tsx:17`.
- Purpose: federated-login landing page; exchanges a one-time authorization code for a session.
- UI: spinner + "signing in..." text only.
- Actions: none user-initiated — on mount, exchanges `code` query param (`useOidcExchange`,
  `POST /api/auth/oidc/exchange`) exactly once (a `useRef` guard, with a code comment explaining the
  code is single-use).
- AC: `src/__tests__/features/auth/oidcSignIn.test.tsx` — exchanges once then follows the normal
  post-login path; no code -> back to sign-in with nothing exchanged; a refused code -> one message.

### ResetPasswordPage — AC: **Yes**
- Route `/reset-password` (`?token=`) — `src/features/auth/pages/ResetPasswordPage.tsx:21`.
- Purpose: complete a password reset from an emailed link.
- UI: strength meter, new/confirm password inputs, requirements hint, 4 alternate render branches
  (form / missing-token / expired-or-used / invalid-link).
- Actions: submit -> client validation then `useConfirmPasswordReset`
  (`POST /api/auth/password-reset/confirm`).
- Data: `useValidatePasswordResetToken(token)` (`GET /api/auth/password-reset/validate`) on mount
  chooses the render branch.
- AC: `src/__tests__/features/auth/ResetPasswordPage.test.tsx` — missing token, HTTP 410 expired,
  successful reset + redirect, backend-detail error toast.

### ForceChangePasswordPage — AC: **Yes**
- Route `/auth/force-change-password` — `src/features/auth/pages/ForceChangePasswordPage.tsx:18`. Fact:
  the only route rendered inside `ProtectedRoute` but outside `AppLayout` — no logout/skip affordance
  by design (code comment: "no way out in the header").
- Purpose: force a password change on first login with a temporary/admin-set password.
- UI: info banner, strength meter, new/confirm inputs, requirements hint, submit.
- Actions: submit (disabled while invalid/empty) -> `useForceChangePassword`
  (`POST /api/auth/force-change-password`) -> renewed tokens + `/dashboard` on success.
- AC: `src/__tests__/features/auth/ForceChangePasswordPage.test.tsx` — submit + redirect, error toast.

---

## `src/features/dashboard`

### GlobalDashboardPage — AC: **Partial/thin** (gap)
- Route `/dashboard` — `src/features/dashboard/pages/GlobalDashboardPage.tsx:19`. `/dashboard/global`
  is a pure redirect to `/dashboard`.
- Purpose: cross-project KPI dashboard (ADMIN-only data) plus first-run onboarding/empty states.
- UI: conditional "quick start" card (`OnboardingGuide`) for users with projects but zero executions;
  non-ADMIN "go to your project dashboards" card; `TimeRangePicker`; 4 KPI `StatCard`s; empty state for
  zero-project users; `TrendChartCard`, `LayerHeatmap`, top-projects grid, top-step-components list.
- Actions: "go to projects" (empty state / non-ADMIN card) -> `/projects`; time-range change; project
  card click -> `ProjectDashboardPage`.
- Data: `useDashboardGlobal()` -> `GET /api/v1/dashboard/global?window=`.
- States: `isLoading` -> skeleton KPI row; `isNewUser` -> empty state; no top projects -> text.
- AC: only `src/__tests__/features/dashboard/hooks/useDashboardGlobal.test.tsx` (hook-level: role gating
  of the ADMIN-only endpoint, 403 handling). No page-level test. `axe.spec.ts`/`reflow.spec.ts` cover
  this route for accessibility/reflow only, not function.

### ProjectDashboardPage — AC: **Partial/thin** (gap)
- Route `/dashboard/projects/:projectId` — `src/features/dashboard/pages/ProjectDashboardPage.tsx:17`.
- Purpose: per-project KPI dashboard — success rate, trend, layer/suite breakdown, flakiness,
  requirement coverage.
- UI: `TimeRangePicker`, 4 `StatCard`s, `TrendChartCard` + `LayerHeatmap`, suite-breakdown list,
  flakiness card (stable/moderate/high counts + top-5 flaky scenarios), conditional requirement
  coverage card with an explicit "coverage unknown" hint when the rate cannot be computed.
- Actions: time-range change only (read-only dashboard otherwise).
- Data: `useProjectDashboard(projectId)` -> `GET /api/v1/dashboard/projects/{id}` (+ requirement
  endpoints).
- States: `project.isLoading` -> full-page skeleton; empty suite/flaky lists -> text; coverage
  explicitly renders "unknown" rather than guessing a number when it cannot be computed.
- AC: one narrow test, `ProjectDashboardCoverage.test.tsx` ("shows unknown total and coverage with an
  explanation") — covers only the coverage-unknown branch, not the rest of the page.

---

## `src/features/catalog`

### StepsPage — AC: **Yes** — the "catalog screen" (deep dive, 2026-10-02 review)
- Route `/steps` — `src/features/catalog/pages/StepsPage.tsx:37`.
- Purpose: browse the global deterministic step catalog; a per-environment step-availability check.
  Code comment: "a 504-step catalog," explaining why search is primary. This is the client's first real
  look at what the platform can execute, so its containers and ordering are specified here in full
  (the prior draft of this document covered it in one paragraph; the user flagged it as under-analyzed
  and as the single most important screen to get right, since it is where a new client either
  understands the product and wants to keep exploring, or does not).
- **Two views, toggled by a segmented control at the top of the page body** (`steps.viewPalette` /
  `steps.viewCatalog`), default **Palette**:

  **1. Palette view (default) — containers top to bottom:**
  1. **First-run promo banner** (shown until dismissed, `localStorage` key
     `qa.catalog.promoDismissed`, permanent per browser) — flask icon, title/description text, a
     "go build a scenario" action (navigates to the last-visited new-scenario path from
     `sessionStorage`, or `/projects` if none), and a dismiss (X) button. No data beyond i18n copy.
  2. **Environment selector** (rendered only when the active project has >=1 environment) — a
     `<Select>` of the project's environments (`useProjectEnvironments`) plus a "none selected" option;
     choosing one fetches `useEnvironmentStatus(projectId, envId)` and turns on per-step
     availability badges in container 4. No environment selected (the default) shows no status data
     at all — not a loading state, a "nothing requested yet" state.
  3. **Search + layer filter bar**, one row (`medium:flex-row`, stacked on narrow): a `SearchField`
     (free-text, token search over the localized step name/pattern/description via
     `stepLocaleSearchText`) to the left, and **4 layer-filter toggle chips to the right, in this
     fixed order: `API`, `WEB`, `MOBILE`, `DATABASE`** (`StepsPage.tsx:36`,
     `const LAYERS = ['API', 'WEB', 'MOBILE', 'DATABASE']`). **Fact worth flagging**: this is not
     alphabetical (alphabetical would read API, DATABASE, MOBILE, WEB) — it matches the exact layer
     order the workspace `.claude/CLAUDE.md` states as the product's "Execution coverage: HTTP/API,
     Web, Mobile, Database." The two lists are independently maintained source (one a TS const array,
     the other a documentation sentence) but currently agree; a future edit to either should keep
     them in sync, since the layer order on the client's very first screen is effectively the
     product's own stated order of what it executes. Chips are **multi-select, OR logic** (any
     active chip's layer passes; zero active chips means "all layers"); a step passes the filter only
     if it matches at least one active layer AND the free-text search (AND between search and the
     layer-OR-group). A "reset filters" ghost button appears only once `hasFilters` is true (search
     non-empty or >=1 layer chip active) and clears both at once.
  4. **Two-pane `ListDetailLayout`** (component list <-> step detail), collapsing to one pane at a
     time below the `large` window class, two panes side by side at `large`:
     - **List pane** (left): one row per **component** (not per step) — steps are grouped by
       `componentId` first. Each row renders `ComponentSummary` in `compact` mode: the component's
       localized display name/title, its category, and (implicitly, via the shared category/phase
       color helpers) a phase-colored marker. **Group order is by `stepCategoryRank(primaryPhase)`
       of the component's first step, not by layer** — so when multiple layers are shown at once
       (no chip active, or several active), components interleave by phase/category rather than
       clustering by layer. The layer chips are therefore a **pre-filter**, not a sort key: picking
       one layer narrows the list to that layer's components only, in the same phase-ranked order.
     - **Detail pane** (right): the selected component's header (localized title + a step count,
       `steps.componentSteps`) and then one `StepRow` per step belonging to that component, in catalog
       order. Each `StepRow` shows: a phase badge (color-coded, e.g. Given/When/Then-style phase),
       the step's localized title (or, per the frozen F-16 decision in this repository's own
       `CLAUDE.md`, the raw Gherkin **pattern** in monospace when no true per-step name exists yet —
       "the pattern is what tells them apart until the backend carries `nameKey`" — do not read a
       pattern-only row as unfinished UI), its parameter chips (`name: type`, a `*` marker for
       required params, or "no parameters" text), and its localized description. **When an
       environment is selected** (container 2), each row additionally carries one of two optional
       badges from `useEnvironmentStatus`: a lock icon + "blocked" chip with an `actionHint` tooltip/
       line (status `BLOCKED_MISSING_ENV` — the step exists but this environment is missing a
       required credential/config key for it) or a plain "deprecated" chip (status `DEPRECATED`).
       With no environment selected, neither badge can ever appear — this is the catalog's default,
       environment-agnostic state.
     - At the `large` window class, a component is auto-selected (the first one in the filtered/
       grouped list) so the detail pane is never empty beside a full list; below `large`,
       `ListDetailLayout` shows one pane at a time with an explicit back control.
     - Empty state (list pane, zero groups after filtering): a search icon, "no results" text, and —
       only if filters are active — a "reset" link, inside the same card the list normally occupies
       (not a full-page empty state).

  **2. Catalog view** (`StepCatalogPanel`, `src/features/catalog/components/StepCatalogPanel.tsx`) —
  a **read-only, strictly hierarchical browser**, structurally different from the Palette view:
  **layer -> category -> step**, all three levels independently collapsible (default open), each with
  a step-count badge next to its name. Layer sections render in whatever order
  `useStepCatalog()`'s `GET` response returns them (server-driven order — **fact, not necessarily the
  same `API, WEB, MOBILE, DATABASE` order as the Palette view's chips**, since this view has no
  client-side `LAYERS` constant of its own; worth a product check whether the two views should agree).
  A step row here has no project/environment context at all — no `BLOCKED`/`DEPRECATED` badge is
  possible in this view (its `StepRow` takes no `status` prop, unlike the Palette view's `StepRow`
  component of the same name but a different file) — so an environment-blocked step looks identical
  to an available one in Catalog view. This view has no search and no layer-filter chips; its own
  loading state is a 4-row skeleton, and its error state (`isError`) is a dedicated inline alert, not
  a shared app-level error boundary.
- Actions: view toggle, promo navigate/dismiss, environment select, search, layer-chip toggle, filter
  reset, component selection, back (narrow layout), param-row expand/collapse (Catalog view only).
- Data: `useStepsList()` -> `GET /api/catalog/steps` (+ `/components`, project-scoped variant);
  `useProjectEnvironments`/`useEnvironmentStatus`; Catalog view's own `useStepCatalog()`.
- States: `isLoading` -> skeleton list (Palette); Catalog view has its own loading skeleton and error
  alert, independent of the Palette view's.
- AC: `src/__tests__/features/catalog/pages/StepsPage.test.tsx` (**9 cases**, corrected 2026-10-02 —
  verified by `requirements-analyst`; an initial pass of this review undercounted it as 6) across 3
  describe blocks: list/WEB-layer-filter/search/reset (4 cases), the two-pane `large`-layout behavior —
  default first-component selection, detail-pane selection, selection survives a filter change (3
  cases), and the one-pane narrow-layout behavior — initial list render, back navigation (2 cases).
  **Gap found in this review**: none of the 9 cases exercises the Catalog view (`StepCatalogPanel`), the
  environment-status badges (`BLOCKED_MISSING_ENV`/`DEPRECATED`), or the promo banner's dismiss
  persistence — see FR-AC-15.

### CustomStepsPage — AC: **Yes**
- Route `/projects/:projectId/custom-steps` — `src/features/catalog/pages/CustomStepsPage.tsx:18`.
  Fact: previously gated by an env flag (`VITE_ENABLE_CUSTOM_STEPS_UI`) that no build set while the
  route stayed reachable; now unconditionally offered to every author.
- Purpose: manage project-scoped reusable step macros ("custom steps").
- UI: "Repeated sequences" (deterministic mining) and "Create macro" actions; macro row list (name, id,
  description, step count, deprecated badge, edit/delete); `MacroEditorModal`, `MacroSuggestionsModal`.
- Actions: create/edit/delete a macro; view suggestions; all write actions disabled without
  `canAuthorScenarios`.
- Data: `useCustomStepsVM(projectId)` -> `GET/POST/PATCH/DELETE /api/projects/{id}/custom-steps`,
  `GET /api/projects/{id}/macro-suggestions`.
- States: `isLoading` -> spinner; empty -> `EmptyState`.
- AC: `src/__tests__/features/catalog/pages/CustomStepsPage.test.tsx` — READ disables writes,
  READ_WRITE enables them.

---

## `src/features/admin`

### AdminUsersPage — AC: **Yes**
- Route `/admin/users` — `src/features/admin/pages/AdminUsersPage.tsx:23`. Code comment: "Only ADMIN
  may open it; the sidebar hides it for others, and a direct URL shows a forbidden state instead of a
  page of failing requests (FE-SHELL-19)."
- Purpose: system-wide user administration — invite/create, edit roles, lock/unlock, deactivate, reset
  passwords, recover email access, reset MFA, review pending sign-ups.
- UI: non-ADMIN -> single forbidden paragraph, nothing else. ADMIN: `PendingSignupsPanel`, search,
  invite/create buttons, `UsersTable` with per-row actions, pagination, multiple modals.
- Actions: per-row modals -> `useAdminUsersVM` mutations (`GET/POST/PATCH/DELETE /api/users` + specific
  sub-paths).
- States: non-ADMIN -> forbidden text, no data fetched; per-action `isPending` flags.
- AC: `src/__tests__/features/admin/AdminUsersPage.test.tsx` + `adminResets.test.tsx`,
  `adminEmailRecovery.test.tsx`, `useAdminUsersVM.test.tsx`, `PendingSignupsPanel.test.tsx`.

---

## `src/features/onboarding`

### OnboardingPage — AC: **Yes**
- Route `/onboarding` — `src/features/onboarding/pages/OnboardingPage.tsx:13` (40 lines). **Not a
  sidebar link** — reached only via `ProtectedRoute`'s automatic redirect for a user with no projects
  and no completed onboarding.
- Purpose: first-run guided checklist for a brand-new user.
- UI: intro text, `OnboardingGuide` component (shared, also embedded in `GlobalDashboardPage`), "Finish"
  button.
- Actions: "Finish" -> `useOnboardingGuide().finish()` (marks completion in `localStorage` per-user key).
- States: effect redirects to `/projects` if already completed in this browser.
- AC: `src/__tests__/features/onboarding/pages/OnboardingPage.test.tsx` (4 cases) — already-completed
  redirect, project-step-only without a project, guide points to Integrations/environments/overlay with
  a project, finish marks completion and opens projects.

---

## `src/features/settings`

### SettingsPage — AC: **None at page level** (gap)
- Route `/settings` — `src/features/settings/pages/SettingsPage.tsx:27`. Fact: this is the user's
  personal/account settings, structurally unrelated to `ProjectSettingsPage` despite the similar name
  (local tab state here vs. route-driven `:tab` there).
- Purpose: personal account settings — profile, password, security/MFA, integrations redirect, about.
- UI: 4-tab side nav (General/Security/Integrations/About), all client-side (no URL sync). Integrations
  tab is a **static placeholder** (one paragraph + a button to `/projects`). About tab is static text.
- Actions: "save profile" -> `useSettingsProfileVM` (`PATCH /api/auth/me[/preferences]`); "update
  password" -> `useChangePassword` (`POST /api/auth/me/password`).
- States: email-change step-up fields appear conditionally; no page-level loading/error/empty state
  (static shell; each sub-section manages its own state).
- AC: hook-level only (`useChangePassword.test.ts`, `useSettingsProfileVM.test.ts`) and component-level
  (`SecurityTab.test.tsx`, `MfaEnrollmentWizard.test.tsx`). **No `SettingsPage.test.tsx` found.**

### SecurityTab + MfaEnrollmentWizard (no own route — `SettingsPage`'s "Security" tab body) — two-factor authentication, deep dive added 2026-10-02
- Files: `src/features/settings/components/SecurityTab.tsx:15`,
  `src/features/settings/components/MfaEnrollmentWizard.tsx:22`,
  `src/features/settings/components/RecoveryCodesDisplay.tsx`,
  `src/features/auth/pages/MfaChallengePage.tsx:15` (login-time challenge, already in Part 1 above).
- Purpose: let a user turn TOTP-based MFA on/off for their own account (Google Authenticator and
  compatible apps — the exact scheme the user confirmed working in manual testing).
- **SecurityTab, two states**:
  - **Disabled** (default): intro text + a single "Enable MFA" button opening the wizard.
  - **Enabled**: a pass-toned panel (shield-check icon, "enabled" title + hint) with two actions:
    "Regenerate recovery codes" and "Disable". **Fact**: "Regenerate recovery codes" does not open any
    flow — its handler is `toast.info(t('settings.security.mfa.regenerateNotAvailable'))` only. This is
    a button that is always visible and always clickable, but can never do the thing it names; see
    FR-UX-02 in Part 3 and the additional-requirements document for the regeneration flow itself.
  - **Disable flow**: a modal requiring the current account password **and** a valid 6-digit TOTP code
    together (`useMfaDisable`) — disabling MFA is itself a re-authenticated action, not a single click.
- **MfaEnrollmentWizard — 4 linear steps, one `Modal`, no back-skipping past `recovery`**:
  1. `password` — re-enter the current account password (D17.10: enrollment itself requires
     re-authentication) -> `useMfaStartEnrollment` -> `POST` starts enrollment and returns a secret.
  2. `qr` — shows a QR code (`data:image/png;base64,...`) and the raw `otpauthUrl` as selectable
     monospace text (manual-entry fallback for a device that cannot scan). No camera/scanning happens
     in the browser; the QR is rendered for an external authenticator app to scan.
  3. `code` — a 6-digit numeric input (`inputMode="numeric"`, `autoComplete="one-time-code"`,
     digits-only sanitization) to prove the app was enrolled correctly -> `useMfaConfirmEnrollment`.
     Can step back to `qr`; cannot proceed with fewer than 6 digits.
  4. `recovery` — `RecoveryCodesDisplay` shows the one-time recovery codes **this is the only time
     they are ever shown** (code comment: "they are not shown again," FE-SHELL-21), with its own
     "download as .txt" action alongside the on-screen list (`RecoveryCodesDisplay.tsx:25-27,46-49`,
     noted 2026-10-02). The modal's close
     handler (`handleClose`) explicitly blocks the X button, Escape and backdrop click while on this
     step until the user checks an explicit "I saved these codes" acknowledgement; only then does
     "Done" become enabled, and only then does a success toast fire and the modal close.
- **Login-time challenge** (`MfaChallengePage`, already documented in Part 1): 6 auto-advancing digit
  inputs with paste-fill, or a recovery-code fallback mode — consumes one of the codes shown once in
  step 4 above.
- Data: `src/features/settings/api/mfa.ts`, `src/features/settings/hooks/useMfa.ts` ->
  `POST /api/settings/mfa/enroll/start`, `POST /api/settings/mfa/enroll/confirm`,
  `POST /api/settings/mfa/disable`, `GET` status (exact paths per `useMfa.ts`).
- States: enrollment-start error and confirm-code error each render inline under their own field
  (`messageFromAxiosError`), not a page-level banner; `SecurityTab`'s own loading state is a 3-row
  skeleton while `useMfaStatus` resolves.
- AC: `src/__tests__/features/settings/SecurityTab.test.tsx`,
  `src/__tests__/features/settings/MfaEnrollmentWizard.test.tsx` (component-level — covers the wizard
  and tab in isolation); `src/__tests__/features/auth/MfaChallengePage.test.tsx` (the login-time
  challenge). **No test found that drives the two together** — i.e. enroll in the wizard, then
  actually clear a login challenge with a code from that same enrollment, or with one of the
  just-issued recovery codes — see FR-AC-16.

---

## `src/features/project`

### ProjectsPage — AC: **Yes**
- Route `/projects` — `src/features/project/pages/ProjectsPage.tsx:23`.
- Purpose: list, search, create, (role-gated) deactivate projects — entry point into every
  project-scoped screen.
- UI: responsive card grid, search, create modal, delete confirm, pagination, skeleton grid.
- Actions: card click -> project drill-down; create -> `POST /api/projects`; delete (role-gated) ->
  `DELETE /api/projects/{id}`.
- Data: `useProjectsVM()` -> `GET /api/projects`.
- States: loading -> skeleton grid; zero projects+no search -> `EmptyState`; search-no-results -> text.
- AC: `src/__tests__/features/project/pages/ProjectsPage.test.tsx` ("CRUD + RBAC, QA-02") — list, empty
  state+CTA, create flow, delete flow, permission gating (QA_LEAD can create, not deactivate), skeleton.
- **Fact**: the create form collects `name` + `description`, but no edit mutation or edit screen exists
  anywhere in the app for an existing project's own name/description afterward (confirmed by direct
  search; relevant to FR-UX-01's "General" tab question in Part 3).

### ProjectDetailPage — AC: **None** (gap)
- Route `/projects/:projectId` — `src/features/project/pages/ProjectDetailPage.tsx:45`.
- Purpose: a project's home screen — readiness checklist, stats, recent executions, quick links.
- UI: Jira-key + permission badge, 4 stat cards, conditional onboarding banner, conditional setup
  checklist (hidden once complete — code comment "F-28: setup progress is a checklist, not a verdict...
  it disappears once complete, because a list of things you have already done is not information"),
  recent-executions list + quick-action cards (Suites/Executions/Environments) + last-execution card.
- Actions: quick-action cards -> Suites/Executions/Environments; recent-execution row -> live or
  results view; "view all" -> `HistoryPage`.
- Data: `useProjectDetailQueries(projectId)` — 5 queries (project, stats, recent executions, metrics)
  plus environments and a pending-suggestions count.
- States: `loadingProject` -> full-page skeleton; stat values show `'—'` while loading; empty recent
  executions -> icon + text. **No explicit page-level error state found.**
- AC: **none found.**

### SuiteDetailPage — AC: **None at page level** (gap)
- Route `/projects/:projectId/suites/:suiteId` — `src/features/project/pages/SuiteDetailPage.tsx:25`.
- Purpose: a suite's feature list, plus import/sync/execute actions scoped to the suite.
- UI: feature search, feature card grid (name, scenario count, tags, path), pagination; modals: Sync
  (push or zip download), Create feature, Execute suite.
- Actions: "Import features" -> `ExecutionOverlayPage` import stage (code comment "#106: .feature files
  are imported into the execution overlay, where they are reviewed"); "Sync" -> save or zip download;
  "Run suite" (disabled unless every feature has >=1 scenario and the list finished loading).
- Data: `useSuiteDetailVM` — suite + paged features (+ a full run-eligibility fetch).
- States: loading -> skeleton cards; empty -> `EmptyState` with create action; filtered-empty -> text.
- AC: **none found at page level.**

### HistoryPage — AC: **None** (gap)
- Route `/projects/:projectId/executions` — `src/features/project/pages/HistoryPage.tsx:24`.
- Purpose: a project's full execution history as a sortable table.
- UI: single `DataTable` (status, name-as-link, passed/total, success %, duration, executed-by, date),
  pagination. Code comment: table replaced "a hand-built table with a clickable `<tr>`" for
  accessibility.
- Actions: execution-name link -> live or results view.
- Data: `useProjectHistoryQueries(projectId)` -> executions list + stats (stats used only to pick the
  empty-state message).
- States: loading -> skeleton; empty with zero suites -> "no suites yet"; empty otherwise -> "no
  executions yet".
- AC: **none found.**

### EnvironmentsPage — AC: **None at page level** (gap)
- Routes: `/projects/:projectId/environments` **and** embedded inside `ProjectSettingsPage`'s
  `environments` tab — `src/features/project/pages/EnvironmentsPage.tsx:37`. Same component serves both
  contexts.
- Purpose: manage per-project execution environments — base URL, type, mobile config, credentials,
  connection testing, per-environment SSL.
- UI: environment card grid (type badge, base-URL link, credential chips, a 5-state connection-test
  button: idle/loading/ok/warn/fail); modals for SSL, create, edit, add/update credential.
- Actions: create/edit/SSL/credential modals; "test connection" -> `testConnection(env.id)`. The SSL
  modal's panel links to `CertificateManagerPage` — the only way into it besides a direct URL.
- Data: `useEnvironmentsPageVM`; `canManageEnvironments` gates every write control, and an effect
  resets any open dialog state if authorization becomes false mid-session.
- States: loading -> skeleton; no project selected -> `EmptyState`; no environments -> `EmptyState` with
  create action. **No page-level error state found** (errors surface per-action instead).
- AC: no page-level test; only `useEnvironmentsPageVM.test.tsx` (hook-level). The real-stack e2e specs
  touch environment *creation* as one step inside a larger flow (`overlay-real-stack.spec.ts`:
  "create an environment with the e2e Postgres as database connection 'lab'") but do not exercise the
  SSL modal, credential editing, or the connection-test states as dedicated criteria.

### CertificateManagerPage — AC: **None** (gap, low severity — thin wrapper)
- Route `/projects/:projectId/ssl/certificates` — `src/features/project/pages/CertificateManagerPage.tsx:8`
  (26 lines) — a back button plus the `CertificateManager` component.
- Purpose: thin wrapper around certificate management.
- UI/States: no own header, loading or error handling.
- Actions: back -> `EnvironmentsPage`.
- AC: **none found for the wrapper itself.**

### ProjectSettingsPage (tab shell) — AC: **Yes** (for the shell itself)
- Route `/projects/:projectId/settings/:tab`. Also: `/settings` -> redirect to `general`; legacy
  `/projects/:projectId/integrations` -> redirect to `../settings/integrations`.
- Purpose: tabbed settings shell for a project — owns the `PageHeader`, renders one of 5 tab bodies.
- UI: 5 tabs (General/Members/Integrations/Environments/Execution defaults). **General and Execution
  defaults are static placeholder cards — i18n text only, no form, no query, no mutation.** Members ->
  `ProjectMembersTab`; Integrations -> `ProjectIntegrationsPage` (embedded); Environments ->
  `EnvironmentsPage` (embedded).
- States: invalid `:tab` -> redirect to `general`; missing `projectId` -> `/projects`.
- AC: `src/__tests__/features/project/pages/ProjectSettingsPage.test.tsx` — shell renders with the
  right tab highlighted, correct body per tab, invalid-tab redirect, legacy-route redirect.

### ProjectMembersTab (no own route — rendered only as the "members" tab body) — AC: **Covered at e2e level only**
- File `src/features/project/components/ProjectMembersTab.tsx:21`.
- Purpose: view and (owner/admin only) manage project access — add a user, change permission, revoke.
- UI: add-member form (gated on `vm.canManage`, else read-only notice), members table (editable
  permission per row if allowed), revoke confirm. Owner/admins are deliberately absent from the list
  (code comment: "the backend exposes neither the owner's name nor the administrators").
- Actions: add/change-permission/revoke -> `useProjectMembersVM`.
- States: loading/error/empty text states; success -> table.
- AC: no component-level test found, but `e2e/specs/project-members.spec.ts` covers it functionally
  (S4: owner adds a READ member, upgrades to READ_WRITE, revokes; a non-managing member sees a
  read-only list; no accessibility violation on the tab).

### ProjectIntegrationsPage — AC: **Yes**
- Not directly routed — rendered only `embedded` inside `ProjectSettingsPage`'s `integrations` tab —
  `src/features/project/pages/ProjectIntegrationsPage.tsx:81`.
- Purpose: configure the project's single TMS connector (Jira/Xray family) — base URL, provider, auth,
  credentials, Jira sync settings — plus the AI field-mapping review panel and the egress-posture panel.
- UI: base-URL input + Detect button (4-variant result text), `RadioCards` provider selector (code
  comment: "a project already saved with a hidden provider keeps its radio and its fields"),
  provider-specific fieldset, credential fields, `FieldMappingPanel`, `EgressPosturePanel`, Test
  connection/Open overlay/Save actions.
- Data: `useProjectIntegrationsVM`. Code comment: **Azure DevOps is a BE extension point but
  intentionally not offered ("Q8")** — consistent with the documented 1.0 scope.
- States: loading -> skeleton; field-level validation only (no page-level error banner found).
- AC: `src/__tests__/features/project/pages/ProjectIntegrationsPage.test.tsx` (19+ cases) — offers only
  None/Jira-Server/Jira-Cloud (never TestRail or Azure DevOps), hidden-provider retention, credential
  value never re-rendered, save blocked until a missing credential is entered, READ members read-only,
  plus a unit-level `describe` for URL validation and payload shaping.

---

## `src/features/scenario`

### SuitesPage — AC: **None** (gap)
- Route `/projects/:projectId/suites` — `src/features/scenario/pages/SuitesPage.tsx:26`.
- Purpose: list, search, create suites; switches card grid <-> `DataTable` by volume (`CARDS_MAX = 8`,
  code comment: "beyond this many suites, a grid of cards is a worse table than a table").
- UI: card grid or table, search, pagination, create modal (name/description/layer).
- Actions: card/row click -> `SuiteDetailPage`; create -> `POST`.
- Data: `useSuitesVM(projectId)`; `canAuthorScenarios` gates create.
- States: loading -> skeleton cards; empty -> `EmptyState`; filtered-empty -> text.
- AC: **none found at page level.**

### FeatureDetailPage — AC: **None** (gap)
- Route `/projects/:projectId/suites/:suiteId/features/:featureId` —
  `src/features/scenario/pages/FeatureDetailPage.tsx:31` (567 lines).
- Purpose: a feature's scenario list — per-scenario run/edit/duplicate/delete, feature-level run,
  paste-from-Gherkin import, doc export.
- UI: header actions (Paste/Export/Run feature), Background section, scenario list with lint-issue
  indicators, per-scenario actions, search+pagination, several modals.
- Actions: paste-Gherkin import; export; run feature or single scenario; edit -> `ScenarioBuilderPage`;
  duplicate; delete (with an extra warning when background steps exist).
- Data: `useFeatureDetailVM()`.
- States: loading -> skeleton; empty-scenarios -> `EmptyState` with Paste surfaced again.
- AC: **none found at page level.**

### ScenarioBuilderPage — AC: **None** (gap)
- Routes: `.../scenarios/new` and `.../scenarios/:scenarioId/edit` (same component) —
  `src/features/scenario/pages/ScenarioBuilderPage.tsx:34` (540 lines).
- Purpose: drag-and-drop Gherkin scenario builder — compose catalog steps on a canvas, preview, save,
  execute.
- UI: `DndContext` two-panel layout (`StepPalette` / `ScenarioCanvas`), Preview/Save/Execute actions,
  `ExamplesTable`, several modals.
- Fact (read-only rule): `readOnly = !canAuthorScenarios || isProjection`, with a code comment: "a
  projection of an approved imported test case is read only whatever the caller may author" — i.e. a
  scenario that originated from an approved canonical/imported case cannot be edited here even by an
  authoring-capable user.
- Data: `useScenarioBuilderVM()`, `useCustomStepsList()`, `useScenarioBaseUrlAnalysis()`.
- States: `loadingSteps` -> spinner in the palette only; no other explicit loading/error/empty state in
  this component (parent-level loading has already gated entry).
- AC: **none found at page level.**

---

## `src/features/imports`

### ImportsPage — AC: **Partial/thin**
- Route `/projects/:projectId/imports` — `src/features/imports/pages/ImportsPage.tsx:74`.
- Purpose: import a Swagger/OpenAPI spec, Postman collection, or user story (file/text), generating
  scenario drafts reviewed on `ScenarioSuggestionsPage`.
- UI: summary strip, 3-tab selector, testing-profile chips (Swagger/Postman only), file dropzone or
  textarea, 422-validation lists, "wrong tab" hint banners, an explicit disabled-reason next to the
  submit button (code comment, F-11: "the call to action was disabled with the reason stated nowhere at
  all" — now fixed). Post-submit -> `ImportReportPanel`.
- Data: `useImportsVM(...)` -> one of 4 import endpoints; `GET /api/projects/{id}/imports/summary`.
- **Fact**: the workspace `.claude/CLAUDE.md` states imports "from Postman collections, Swagger and
  user stories come in 1.1" (1.0 scope is Jira + Xray only). This page implements exactly those three
  source types and is reachable today (nav link + route both exist), ahead of the documented 1.0/1.1
  boundary. Stated as a fact only — see Gap list.
- AC: `src/__tests__/features/imports/pages/ImportsPage.test.tsx` — only 2 cases (write-capability
  gating: READ keeps import disabled, READ_WRITE enables it) for a page with 3 import modes, a
  testing-profile selector, and heuristic wrong-tab detection — thin relative to its surface.

### ScenarioSuggestionsPage — AC: **None** (gap, high severity)
- Route `/projects/:projectId/suggestions` —
  `src/features/imports/pages/ScenarioSuggestionsPage.tsx:121` (**1318 lines — the largest page in the
  app**).
- Purpose: review, filter, accept/reject and bulk-manage AI-generated scenario drafts from an import,
  organized as a suite -> feature -> draft tree; the entry point to run accepted drafts.
- UI: dismissible intro note, top-level `EmptyState` for zero drafts, status filter chips with live
  counts, "mine" toggle, bulk actions, paginated collapsible suite->feature->draft tree, per-draft
  method/phase chips (deliberately neutral-toned — code comment: "An HTTP verb and a Gherkin phase are
  categories... drawn in the status vocabulary they read as verdicts"), `ConfidenceBar`,
  accept/reject, a shared reject-confirm modal reused for 3 distinct confirms, an execute modal with 3
  states (preparing/error/ready).
- Actions: filter/toggle/expand-collapse; accept (-> `ScenarioDraftBuilderPage`); reject / reject-all /
  purge-rejected; run at suite/feature/scenario scope.
- Data: `useScenarioDraftsVM`, `useScenarioSuggestionsHubVM` -> `GET /scenario-drafts`,
  `POST .../accept|reject|restore`, `POST .../purge-rejected|reject-all-new|reject-by-profile|batch`.
- States: loading -> skeleton; zero drafts -> `EmptyState`; execute modal has explicit
  loading/error/ready sub-states.
- AC: **none found at all** — no page-level unit test and no e2e spec was found. The screen is 1,318
  lines, the largest page in the app. Only adjacent hooks have tests (`useScenarioDraftsVM.test.tsx`,
  `useScenarioDraftBuilderVM.test.tsx`).

### ScenarioDraftBuilderPage — AC: **None at page level** (gap)
- Route `/projects/:projectId/suggestions/:draftId` —
  `src/features/imports/pages/ScenarioDraftBuilderPage.tsx:26`.
- Purpose: inspect a single generated draft's steps (read-only) and accept it into a target
  suite/feature as a real scenario.
- UI: `ConfidenceIndicator` (color-coded at 75%/50%), "accept as scenario", read-only `ScenarioCanvas`,
  accept modal with target-suite/feature selects (each offering "auto-decide" as default).
- Actions: accept (disabled with zero steps) -> `POST /scenario-drafts/{id}/accept` -> navigates to
  `ScenarioBuilderPage` as an editable scenario.
- Data: `useScenarioDraftBuilderVM()` -> `GET /scenario-drafts/{draftId}`.
- States: loading -> skeleton; zero generated steps -> warning banner.
- AC: no page-level test; only `useScenarioDraftBuilderVM.test.tsx` (hook-level).

---

## `src/features/overlay` — the "execution overlay" console

### ExecutionOverlayPage — AC: **Yes** (most thoroughly specified screen in the app)
- Route `/projects/:projectId/execution-overlay` —
  `src/features/overlay/pages/ExecutionOverlayPage.tsx:37`. The page's own file docstring maps directly
  onto the product's flow: **"connect and import, review and map, approve, run, follow
  synchronization."**
- Purpose: single console for the Connect -> Import -> Review -> Run -> Sync stages of one project's
  overlay workflow, one stage panel visible at a time.
- UI: `OverlayFlow` 5-stage bar (connect, import, review, run, sync) as both status display and nav
  control; live-region announcements for import/sync progress (code comment: "An import and a sync
  both run for minutes and update in place without moving focus, so the only thing a screen reader
  would otherwise notice is silence"); per-stage panels (`ImportPanel`, `FeatureImportPanel`,
  conditional `ImportSchedulePanel`; `ImportAssistantPanel` + `ReviewBatchPanel` + `ReviewQueue` +
  `CaseDetail`; `RunPanel`; `SyncJobsTable`).
- **Fact**: the recurring re-import schedule (`ImportSchedulePanel`) is offered only when
  `vm.schedules.offered` — a code comment states it was intentionally withdrawn for 1.0 ("W9 takes the
  offer away for 1.0, because an unattended re-import throws away reviewer work").
- **Fact**: which stage opens is computed, not fixed (`deriveOverlayFlow`), from live query data only
  ("Every value here is computed from a query result; nothing is inferred or estimated"), then pinned
  via state so a background import/sync completing does not silently move the reader — a documented fix
  for a prior defect ("F-04: all 5 panels used to render in one scroll").
- Data: `useExecutionOverlayVM(projectId)` -> canonical-tests (review queue), connector config,
  `POST /execution-plans` (**the actual trigger for the Plan stage — folded into the Run UI, not a
  separate screen**; see Gap list), sync-jobs + retry.
- AC: `src/__tests__/features/overlay/pages/ExecutionOverlayPage.test.tsx` (extensive) +
  `e2e/specs/execution-overlay.spec.ts` (INT-9, INT-10, PRD-2, AI-3, W9 tagged cases: AI proposals never
  auto-apply, blocking-unmapped-import resolution, stale-save 409 handling, not-executable reasons
  shown, READ-only members, synced-then-retry-on-demand, bulk decisions, batch step-mapping jobs) +
  3 real-stack specs exercising the full Connect->Import->Review->Run->Sync loop against Jira/Xray
  Server, Xray Cloud and TestRail.

---

## `src/features/execution`

### ExecutionLivePage — AC: **Yes**
- Routes: `/projects/:projectId/executions/:id` and `/executions/:id` (project-less — router comment:
  "direct access to executions / external links / WebSocket URIs") —
  `src/features/execution/pages/ExecutionLivePage.tsx:26`.
- Purpose: live, real-time view of a running (or just-finished) execution.
- UI: conditional Cancel action, progress card (status badge, websocket connected/disconnected/rejected
  indicator, progress bar), details card, scenario list with pending-skeleton rows while running,
  activity panel, a full-screen blocking modal for the whole duration of `vm.isRunning`.
- Actions: Cancel (header + overlay) -> `vm.handleCancel`; "view full results" once not running. A
  `useBlocker(vm.isRunning)` intercepts in-app navigation during a run and shows a toast instead.
- Data: `useExecutionLiveVM()` (websocket store state, counts, `isRunning`, `isCancelling`).
- States: loading -> skeleton; zero results while running -> "waiting"; `ERROR` -> specific icon/text;
  `CANCELLED` -> specific icon/text; the blocking overlay is itself a distinct whole-screen state.
- AC: `src/__tests__/features/execution/pages/ExecutionLivePage.test.tsx` — READ hides/disables cancel,
  READ_WRITE enables it, no legacy attach-report actions on a finished execution.

### ExecutionResultsPage — AC: **Yes**
- Routes: `/projects/:projectId/executions/:id/results` and `/executions/:id/results` —
  `src/features/execution/pages/ExecutionResultsPage.tsx:26`.
- Purpose: finished-execution results/evidence screen — summary stats, per-scenario detail, re-run,
  HTML report download, on-demand Xray sync.
- UI: "download report" action, "Rerun" (disabled-reason tooltip covering 4 distinct cases), conditional
  "Sync to Xray", error banner for a stable error code, 4 summary cards, `RunAssistantPanel`,
  per-scenario list with per-row rerun, info card with copy-to-clipboard execution id.
- Actions: download (code comment: "an authorized API path: download through the API client, never
  window.open"); page-level and per-scenario rerun; on-demand sync.
- Data: `useExecutionResultsVM()`, `useHtmlReport(reportUrl)`.
- States: loading -> skeleton; `!execution` -> bare "not found" text; error-reason banner conditional;
  empty scenario list -> text.
- AC: `src/__tests__/features/execution/pages/ExecutionResultsPage.test.tsx` (multiple describe blocks)
  — rerun RBAC (page + per-row), errorScenarios-as-skipped display, overlay-run rerun explanation, ERROR
  display, on-demand sync availability/disabled states. Plus real-stack e2e coverage of the Evidence and
  Sync stages end to end.

---

## Cross-cutting behaviors (no own route, no single owning page) — added 2026-10-02

Two behaviors reach every protected screen and every API call, so neither belongs under one page in
Part 1 above, but the user specifically asked for an expired-session analysis and an error-screens
analysis. Both are documented here instead.

### Session expiration

Two independent reasons end a session, both funneling through the same `endSession()` (D8.2):
`idle` (no mouse/keyboard/touch/scroll/click activity for a configured window) and `access_expired`
(the access JWT's own expiry, when "strict access expiry" is enabled). A third and fourth reason exist
only as direct calls, not through the warning flow below: `session_expired` (a background API call's
401 survived a silent refresh attempt) and `security` (account disabled/locked mid-session).

- **Silent refresh is not symmetric between the two reasons** (corrected 2026-10-02 — an initial pass
  of this review claimed it was attempted for both; `requirements-analyst` found it is not):
  - **`access_expired`**: a silent `refreshSession()` is attempted before the session is ever force-
    ended, but the timing depends on how the expiry is noticed. The common case —  watching the
    countdown — shows the 10-second warning first (`useSessionPolicy.ts:100-122`, the warning is
    rendered progressively for the whole 10 seconds while `remaining > 0`) and only attempts the
    silent refresh once the countdown reaches zero (`remaining <= 0`), logging out only if that
    refresh fails. A second, separate effect (`useSessionPolicy.ts:90-97`) covers mounting (or
    navigating) with a token that is *already* expired: there, a silent refresh is attempted
    immediately, with no warning shown first if it succeeds.
  - **`idle`**: no silent refresh is attempted automatically at all. When the idle timer reaches zero,
    `forceLogout('idle')` is called directly (`useSessionPolicy.ts:147-155`) — the only silent-refresh
    call on the idle path is inside `extendSession()`, the "Continue session" button's own handler, which
    is a user-initiated action, not something that happens on the user's behalf before logging out.
  - A background API call's own 401 (`setupApiInterceptors.ts`) is a third, independent path: retry
    once after a refresh, and only call `endSession('session_expired')` if that refresh itself fails.
- **`SessionExpiryWarningModal` — the expired-session screen itself**
  (`src/features/auth/components/SessionExpiryWarningModal.tsx`, rendered by `ProtectedRoute` beside
  `<Outlet />` so it can appear over any protected screen): an `alertdialog` with a 10-second countdown
  (`WARNING_BEFORE_LOGOUT_S`), shown once remaining time drops to 10s for either reason. Containers:
  a thin countdown progress bar at the top of the modal, an alert-triangle icon + title + reason text
  (`sessionPolicy.warningIdle` or `sessionPolicy.warningAccess`, both receiving the live `secondsLeft`
  count), a circular countdown ring with the number of seconds inside it, and two actions: "Log out
  now" (always present) and "Continue session" (**only when `reason === 'idle'`** — confirmed; an
  about-to-expire access token has no "continue" action of its own, since by the time the access-expiry
  warning's countdown reaches zero, the app already attempts the one silent refresh it is going to
  attempt as part of ending that session, per the bullet above — not before showing the warning, as an
  earlier draft of this paragraph said).
- **What happens after logout, regardless of reason**: `endSession()` calls
  `POST /api/auth/logout {global:true}` (best-effort — proceeds to local cleanup even if that call
  fails or times out at 5s), clears the FE auth store and the React Query cache, shows a `toast.warning`
  with a reason-specific message (`sessionPolicy.idleLogout` / `sessionPolicy.accessExpired` /
  `sessionPolicy.securityLogout` / `apiClient.sessionExpired`), then **hard-navigates**
  (`window.location.replace`) to `/login?reason=<reason>` — a full page reload, not a client-side
  route change, so no protected component can render mid-logout.
- **Fact worth a product decision**: `LoginPage` does not currently appear to read the `?reason=`
  query parameter in the file reviewed in Part 1 (it is exposed to the toast only). If a
  reason-specific message on the login screen itself (beyond the toast, which may already be gone by
  the time the full reload completes) is desired, that is a gap, not yet filed as an FR pending
  confirmation this is actually wanted.
- AC: `src/__tests__/features/auth/lib/endSession.test.ts`,
  `src/__tests__/features/auth/lib/refreshSession.crossTab.test.ts`,
  `src/__tests__/shared/lib/sessionPolicyEnv.test.ts` — all test the underlying library functions.
  **Gap found in this review**: no test exercises `useSessionPolicy` itself (the idle timer, the
  access-expiry countdown, or the transition into the 10-second warning window) or
  `SessionExpiryWarningModal`'s own rendering/countdown/action-availability logic — see FR-AC-14.
  Severity: **Medium-high** — this is the mechanism that forcibly ends every session in the app, with
  no page-level or component-level proof that the warning actually appears when it should, counts down
  correctly, or offers "Continue" only for the idle reason.

### Error screens and messages

- **Route-level fallback — `RouteErrorFallback` / `ForcedNotFoundPage`**
  (`src/router/RouteErrorFallback.tsx`): one shared presentation component, two entry points.
  `RouteErrorFallback` is the router's `errorElement` (reads `useRouteError()`); `ForcedNotFoundPage`
  renders the same presentation directly for a path that is unreachable by design (the retired
  `/components/*` marketplace route, per code comment "T-FE8 post-v1"). Containers: a faint dotted-grid
  background decoration, the app logo linking to the landing page, a large icon tile (either a literal
  "404" in mono type, or a triangle-warning icon for any other error), a title, an optional raw status
  code line (`statusCode`, non-404 only), the error detail text, and four actions: **Retry**
  (`window.location.reload()` — a full reload, not a React re-render), **Go to dashboard**, **Report
  issue** (opens a pre-filled GitHub-style new-issue link via `buildFeedbackNewIssueUrl`, through
  `safeHref`, with the current page URL, context tag and detail text templated into the issue body —
  so a user hitting a route error can file a usable bug report in one click), and **Go back**
  (`window.history.back()`). A thrown `Error` with a stack is logged to the console in dev builds only.
  This is the only screen in the app that offers a "report this" action to an ordinary user — worth
  noting as a pattern other error states (below) do not share.
- **API-level error messages — not a screen, a toast convention**: `setupApiInterceptors.ts` and
  `errorMapper.ts` define one small, closed set of HTTP-status + BE-problem-`type` combinations that
  always resolve to a specific i18n'd toast, so that no raw backend `detail` string ever needs to reach
  a user untranslated for these cases: `403 PASSWORD_CHANGE_REQUIRED` (no toast — silently redirects to
  the force-change-password screen instead), `403 ACCOUNT_DISABLED` / `423 ACCOUNT_LOCKED` (error toast
  + full session end, reason `security`), `403 AUTHZ_DENIED` (error toast only, session kept — a
  project-permission gate, not an account-level event), `429 TOO_MANY_REQUESTS` (warning toast whose
  duration matches the server's `Retry-After`), `429`/`503` `TOO_MANY_CONCURRENT_OPERATIONS` (silently
  retried up to twice with backoff before any toast), a `429` with `extensions.scope =
  "execution-capacity"` (its own dedicated toast, D21.1: "a launch refused for run capacity is not
  retried; the user sees when to try again"), `503` (retried up to 3 times with a `toast.loading`
  placeholder, falling back to a "service unavailable" error toast — except problem types in
  `FEATURE_HANDLED_503_TYPES`, such as `ai_unavailable`, which a feature explains itself and the
  interceptor leaves alone), and `401` on any non-public, non-already-retried request (silent refresh
  attempt; on failure, the session-expiration path above). Most toasts created this way also carry the
  request's `X-Correlation-ID` as a description line (`errors.api.reference`), so a user reporting a
  problem can hand support a lookup key — **with one exception, corrected 2026-10-02**: the
  `403 AUTHZ_DENIED` branch (`setupApiInterceptors.ts:65-68`) calls `toast.error(...)` with no second
  argument, so that one toast alone in the catalog carries no correlation-id description, unlike every
  other branch listed above. **Fact**: `messageFromAxiosError()`'s fallback order is
  known problem `type` -> a generic status+code mapping (`codeMessage`) -> the BE's own `detail` text
  for a domain rule with no dedicated message -> a final generic "unexpected error" string — so an
  un-catalogued backend error still reaches the user as readable text, never raw JSON, but may not be
  translated if it falls through to the BE `detail` branch.
- AC: `src/__tests__/app/RouteErrorFallback.test.tsx`, `src/__tests__/router/ForcedNotFoundPage.test.tsx`
  cover the route-level screen. **Corrected 2026-10-02** — an initial pass of this review claimed no
  dedicated test covers the interceptor's toast catalog at all; `requirements-analyst` found that
  claim wrong. Three dedicated files already exist:
  `src/__tests__/app/providers/setupApiInterceptors.test.ts` (`AUTHZ_DENIED`, execution-capacity `429`,
  `ai_unavailable` `503` left untouched), `setupApiInterceptors.auth.test.ts` (concurrent 401
  refresh-and-retry, refresh token read at call time, failed-refresh session clear, 401-then-redirect,
  correlation id on every request, password-change-required flagging, rate-limited `429` toast with its
  correlation id), and `src/__tests__/features/auth/lib/errorMapper.test.ts` (the
  `messageFromAxiosError` fallback chain, `problemTypeIs`, correlation-id resolution,
  execution-capacity mapping). What remains genuinely untested is narrower:
  `ACCOUNT_DISABLED`/`ACCOUNT_LOCKED` (the 403/423 branches that also end the session),
  `TOO_MANY_CONCURRENT_OPERATIONS`'s retry-then-toast, and the generic (non-`FEATURE_HANDLED_503_TYPES`)
  `503` retry-then-toast sequence — see the narrowed FR-AC-17.

---

# Part 2 — Gap list

## 2.1 Stage-level gaps (against the 8-stage flow)

The "AC note" column cross-references §2.2 so this table does not read as implying a stage is
AC-complete just because a screen exists for it — several "Covered" stages contain screens that are
flagged as AC gaps in §2.2.

| Stage | Screen coverage | Finding | AC note (see §2.2) |
|---|---|---|---|
| 1 Connect | `ProjectIntegrationsPage`, `EnvironmentsPage`, `CertificateManagerPage`, overlay connect/import panel | Covered | `EnvironmentsPage` (Medium), `CertificateManagerPage` (Low) are AC gaps |
| 2 Import | Overlay import panel, `ImportsPage`, `SuiteDetailPage` deep link | Covered | `ImportsPage` (thin), `SuiteDetailPage` (Medium) are AC gaps |
| 3 Canonical/Review | Overlay review stage, `ScenarioSuggestionsPage`, `ScenarioDraftBuilderPage` | Covered | `ScenarioSuggestionsPage` (**High**), `ScenarioDraftBuilderPage` (Low-medium) are AC gaps |
| **4 Plan** | **No screen** | **GAP** — see below | n/a |
| 5 Run | Overlay run stage, `ExecutionLivePage`, Run/Execute modals on `SuiteDetailPage`, `FeatureDetailPage`, `ScenarioBuilderPage`, `ScenarioSuggestionsPage` | Covered | `SuiteDetailPage`, `FeatureDetailPage`, `ScenarioBuilderPage`, `ScenarioSuggestionsPage` are AC gaps (Medium/High) |
| 6 Evidence | `ExecutionResultsPage`, `ExecutionLivePage`, `HistoryPage`, dashboards | Covered | `HistoryPage` (Medium), `GlobalDashboardPage`/`ProjectDashboardPage` (Low-medium/Low) are AC gaps |
| 7 Sync | Overlay sync stage, `ExecutionResultsPage`'s on-demand sync | Covered | none — both screens have AC (Yes) |
| 8 Console (workspace `CLAUDE.md`'s `ExecutionOverlayPage` entry point) | `ExecutionOverlayPage` | Covered | none — `ExecutionOverlayPage` has AC (Yes, most thoroughly specified screen) |
| *(outside the 8 stages)* Pre-auth / general platform | `LandingPage`, all of `src/features/auth`, `GlobalDashboardPage`, `ProjectDashboardPage`, `ProjectsPage`, `ProjectDetailPage`, `SuitesPage`, `SuiteDetailPage`, `FeatureDetailPage`, `ScenarioBuilderPage`, `StepsPage`, `CustomStepsPage`, `AdminUsersPage`, `OnboardingPage`, `SettingsPage`, `ProjectSettingsPage` shell | n/a — not part of the 8-stage flow | Several are AC gaps per §2.2; listed here, not under stage 8, because Part 1 states `LandingPage` and the rest of this group sit outside the 8 BE stages, and the workspace `CLAUDE.md`'s stage-8 entry point names specifically `ExecutionOverlayPage`, not a catch-all |

**Correction from the prior draft of this section**: stage 8 ("Console") previously listed as a catch-all
for every screen not otherwise mapped, including pre-auth screens. Part 1 explicitly states
`LandingPage`'s stage as "pre-auth (none of the 8)", which contradicted that catch-all framing. Fixed
here by giving pre-auth/general-platform screens their own row outside the 8-stage table, consistent
with Part 1, and by reserving stage 8 for what the workspace `CLAUDE.md` actually names as its entry
point (`ExecutionOverlayPage`).

### GAP-PLAN-01 — the Plan stage has no dedicated screen anywhere in the FE — **RESOLVED 2026-10-01**
No screen exposes an "Execution Plan" object directly. The `POST /api/projects/{id}/execution-plans`
call is issued by `ExecutionOverlayPage`'s `RunPanel` and by the "Run"/"Execute" modals on
`SuiteDetailPage`, `FeatureDetailPage`, `ScenarioBuilderPage` and `ScenarioSuggestionsPage` — each
collecting environment + runtime-config fields (including, for a WEB run, whether Playwright is used)
immediately before launch. **User decision: this is the permanent, intended design — no new screen is
needed.** Documented in the workspace `.claude/CLAUDE.md` (8-stage table) and
`docs/diagrams/01-general-architecture.md`. See FR-PLAN-01.

## 2.2 Screens with no, or thin, written acceptance criteria

| Screen | AC status | Severity |
|---|---|---|
| `ScenarioSuggestionsPage` | None at all (largest, most complex page: 1318 lines) | **High** |
| `ProjectDetailPage` | None | Medium |
| `SuiteDetailPage` | None at page level | Medium |
| `HistoryPage` | None | Medium |
| `SuitesPage` | None | Medium |
| `FeatureDetailPage` | None (567 lines) | Medium |
| `ScenarioBuilderPage` | None (540 lines, core authoring tool) | Medium |
| `ScenarioDraftBuilderPage` | None at page level | Low-medium |
| `EnvironmentsPage` | None at page level (hook test only; real-stack e2e touches only "create") | Medium |
| `SettingsPage` | None at page level | Low-medium |
| `GlobalDashboardPage` | Hook-level only | Low-medium |
| `ProjectDashboardPage` | One narrow test (coverage-unknown branch only) | Low |
| `ImportsPage` | Thin (2 cases for a 3-mode page with validation and heuristics) | Low |
| `CertificateManagerPage` | None (thin wrapper) | Low |
| `useSessionPolicy` / `SessionExpiryWarningModal` (added 2026-10-02) | None — only `endSession`, `refreshSession` and `sessionPolicyEnv` lib functions are tested, not the idle/access-expiry timers or the warning modal itself | **Medium-high** |
| `StepsPage` Catalog view, env-status badges, promo persistence (added 2026-10-02) | Existing 9 cases cover only list/search/layer-filter/selection in Palette view | Low-medium |
| MFA enrollment -> login-challenge, as one joint flow (added 2026-10-02) | Each half (`MfaEnrollmentWizard`/`SecurityTab` and `MfaChallengePage`) is tested in isolation; no test enrolls then clears a challenge with the result | Low-medium |
| API error-toast catalog (`setupApiInterceptors.ts`/`errorMapper.ts`, added 2026-10-02, narrowed after verification) | `ACCOUNT_DISABLED`/`ACCOUNT_LOCKED` session-ending branches, `TOO_MANY_CONCURRENT_OPERATIONS` retry-then-toast, and the generic `503` retry-then-toast sequence have no dedicated test; most other branches already do (`setupApiInterceptors.test.ts`/`.auth.test.ts`, `errorMapper.test.ts`) | Low |

`ProjectMembersTab` has no component-level test but is functionally covered by
`e2e/specs/project-members.spec.ts` — not listed as a gap.

## 2.3 Other factual findings worth a decision

- **GAP-SCOPE-01 — RESOLVED 2026-10-01**: `ImportsPage` (Swagger/OpenAPI, Postman, user-story import) is
  fully built, routed and nav-linked today; the workspace `.claude/CLAUDE.md` previously scoped those
  three import types to a "1.1" release. **User decision: these import types are part of MVP 1.0.**
  `.claude/CLAUDE.md` and `.claude/ORCHESTRATION.md` have been updated to say so. See FR-SCOPE-01.
- **GAP-UX-01 — fully RESOLVED 2026-10-01**: Three tabs render as static placeholders with no form,
  query or mutation: `ProjectSettingsPage`'s "General" and "Execution defaults" tabs, and `SettingsPage`'s
  "Integrations" tab (a paragraph + a redirect button). **Final user decision**: hide the "General" tab;
  build "Execution defaults" (project-level default environment + runtime-config, overridable per run);
  build `SettingsPage`'s "Integrations" tab as a real cross-project connector-status summary. See
  FR-UX-01 for full detail and the next step.
- **Fact, not a gap**: no orphaned/unreachable route and no nav-entry-without-a-route were found
  anywhere in the router. Verified directly against `qa-platformFE/src/router/index.tsx`: 39 `path:`
  entries total, of which 34 render a page component and 5 do not (3 `Navigate` redirects, 1
  `ForcedNotFoundPage` for the former-marketplace `components/*` dead end, 1 catch-all
  `RouteErrorFallback`). The 34 page-rendering routes resolve to **31 distinct routed page components**
  (`ScenarioBuilderPage`, `ExecutionLivePage` and `ExecutionResultsPage` are each reused by two routes).
  31 = the 34 screens now in Part 1 minus the 3 that are embedded-only and never directly routed
  (`ProjectMembersTab`, `ProjectIntegrationsPage`, and — added 2026-10-02, same precedent —
  `SecurityTab`/`MfaEnrollmentWizard`, both tab bodies of already-counted routed pages). This count
  has not changed with the 2026-10-02 addition: `SecurityTab` was always reachable as part of
  `SettingsPage` (route `/settings`), which was already one of the 31; only Part 1's own
  description of that tab's content grew, not the route graph. Every one of the 31 was traced to at
  least one reachable navigation path — sidebar, in-page action, breadcrumb, or an automatic redirect.
  *(Correction from the prior draft: this section previously cited "28," a figure carried over from the
  research pass without independent verification; it did not reconcile against Part 1's own 33-screen
  count. Recounted directly against the router file above.)*

---

# Part 3 — Functional requirements

**FR-PLAN-01** — **RESOLVED 2026-10-01.** User decision: folding Plan into the environment +
runtime-config step of every "Run"/"Execute" action (environment choice, and for WEB runs whether
Playwright is used) is the permanent intended design. No new screen is needed. Documented in the
workspace `.claude/CLAUDE.md` (8-stage table) and `docs/diagrams/01-general-architecture.md`. No further
action required.

**FR-AC-01** (high priority) — Write acceptance criteria for `ScenarioSuggestionsPage`.
Cover at minimum: status-filter and "mine" correctness, suite/feature tree expand-collapse and
pagination, accept navigates to the correct draft builder, reject and the 3 bulk-reject paths
(reject-one, reject-all-new, purge-rejected) each call the correct endpoint and update counts, and the
execute-modal's preparing/error/ready states for suite/feature/scenario scope. Priority: high — this is
both the largest/most complex page in the app and the FE's primary expression of the Canonical/Review
stage for the non-TMS import path, so an undetected regression here directly touches the product's core
"human approves before anything runs" claim. **Before `test-strategist` writes a plan against this FR**,
get one explicit confirmation that the current accept/reject/bulk-reject behavior is itself believed
correct and is the intended target to lock into tests — not merely "whatever the code currently does."

**FR-AC-02** — Write page-level acceptance criteria for `ProjectDetailPage` (readiness-checklist
visibility logic per F-28, stat-loading placeholders, recent-executions empty state, quick-action
links), and add an explicit page-level error state if the design intends one (none exists today).

**FR-AC-03** — Write page-level acceptance criteria for `SuiteDetailPage` (feature search/filter,
create-feature flow, sync-modal's two paths, run-suite eligibility gating).

**FR-AC-04** — Write page-level acceptance criteria for `HistoryPage` (sortable table rendering,
the two distinct empty-state messages keyed off suite count, row-link destination logic).

**FR-AC-05** — Write page-level acceptance criteria for `SuitesPage` (card-vs-table switch at the
`CARDS_MAX = 8` threshold, search, create flow, RBAC gating of create).

**FR-AC-06** — Write page-level acceptance criteria for `FeatureDetailPage` (paste-Gherkin import,
export, per-scenario run/edit/duplicate/delete, the background-steps delete-warning variant, lint-issue
indicators).

**FR-AC-07** — Write page-level acceptance criteria for `ScenarioBuilderPage` (drag-and-drop step
composition, preview, save, execute, and — importantly — the `isProjection` read-only rule, since an
approved imported case becoming silently editable would be a direct violation of the product's review
guarantee).

**FR-AC-08** — Write page-level acceptance criteria for `ScenarioDraftBuilderPage` (zero-steps warning
and disabled accept, target-suite/feature "auto-decide" default, accept navigation target).

**FR-AC-09** — Write page-level acceptance criteria for `EnvironmentsPage` beyond the current
real-stack "create one environment" coverage: the 5-state connection-test button, SSL modal, credential
add/edit/delete, and the effect that resets open dialogs if authorization is revoked mid-session.

**FR-AC-10** — Write a `SettingsPage.test.tsx` covering tab switching, the profile-save and
change-password flows at the page level (today only their hooks are tested), and the email-change
step-up field conditions.

**FR-AC-11** — Write page-level acceptance criteria for `GlobalDashboardPage` (new-user empty state,
non-ADMIN card, quick-start card visibility condition, top-projects navigation) and
`ProjectDashboardPage` (beyond the existing coverage-unknown case: suite-breakdown and flakiness
rendering, time-range change).

**FR-AC-12** — Broaden `ImportsPage.test.tsx` beyond write-capability gating to cover: tab switching
and its reset-on-switch behavior, the "wrong tab" heuristic banners, the testing-profile multi-select,
and both file and text input modes for the User Story tab.

**FR-AC-13** (low priority) — Add a minimal test for `CertificateManagerPage` (back-navigation target)
if the page remains a standalone route; otherwise consider documenting it as intentionally untested
scaffolding.

**FR-AC-14** (medium-high priority, added 2026-10-02) — Write a test for `useSessionPolicy` and
`SessionExpiryWarningModal` covering: the idle timer actually shows the 10-second warning and then logs
out with reason `idle` when no activity event fires in time; any activity event clears a visible idle
warning; the access-expiry countdown shows the warning and attempts a silent refresh before forcing
logout with reason `access_expired`; "Continue session" is offered only when `reason === 'idle'`; and a
successful silent refresh at any point hides the warning without logging out. Priority: medium-high —
this is the mechanism that forcibly ends every session in the app and today has zero test coverage of
its own timing/branching logic, only of the library calls it uses.

**FR-AC-15** (low-medium priority, added 2026-10-02) — Extend `StepsPage.test.tsx` to cover the Catalog
view (`StepCatalogPanel`'s layer/category/step collapse-expand and its own loading/error states), the
environment-status badges (`BLOCKED_MISSING_ENV` lock chip with its `actionHint`, and `DEPRECATED`)
once an environment is selected, and that dismissing the first-run promo banner persists across a
remount (the `localStorage` key).

**FR-AC-16** (low-medium priority, added 2026-10-02) — Add one test (unit or e2e) that drives the MFA
enrollment wizard to completion and then uses the resulting TOTP secret or a just-issued recovery code
to clear `MfaChallengePage` at a subsequent login — proving the two screens actually agree on what a
valid code is, not just that each renders correctly in isolation.

**FR-AC-17** (low priority, added 2026-10-02, **narrowed after `requirements-analyst` verification**
found most of the catalog already has dedicated coverage in `setupApiInterceptors.test.ts`,
`setupApiInterceptors.auth.test.ts` and `errorMapper.test.ts`) — Extend that existing coverage with the
three branches confirmed still untested: `ACCOUNT_DISABLED`/`ACCOUNT_LOCKED` (the 403/423 branches that
also call `endSession('security')`), `TOO_MANY_CONCURRENT_OPERATIONS`'s retry-then-toast, and the
generic `503` retry-then-toast sequence for a problem type outside `FEATURE_HANDLED_503_TYPES`.

**FR-UX-02** (product decision needed, added 2026-10-02) — `SecurityTab`'s "Regenerate recovery codes"
button is always visible once MFA is enabled, but its handler only shows an
"not available" info toast (`settings.security.mfa.regenerateNotAvailable|regenerateNotAvailable`) —
there is no regeneration flow behind it today. This is a UX gap, not a missing test: a control that is
always clickable but can never perform the action it names. Needs a product decision — build the
regeneration flow (new BE endpoint re-issuing codes, re-authenticated the same way enrollment is), or
replace the button with a disabled/explained state until it exists. Candidate for the separate
non-MVP additional-requirements document if deferred past 1.0.

**FR-AUTH-01** (added 2026-10-03, **RESOLVED by explicit user decision**) — Google sign-in for MVP 1.0,
sign-in only. **Fact established by this review**: a complete, generic OIDC federated-login mechanism
("IDP-1") already exists end to end in both repos (`qa-platformFE`'s `FederatedSignIn`/`OidcCallbackPage`,
`qa-platformBE`'s `FederatedLoginService`/`OidcFederationProperties`/`ClientRegistrationProviderAdapter`),
fully built and tested, dormant behind two false-by-default flags (`qa.oidc.enabled`,
`qa.oidc.provisioning-enabled`) and mentioned nowhere in this document or any `CLAUDE.md` before now. No
Google-specific or Apple-specific code exists anywhere (confirmed by search). Resolved decisions:
1. Google sign-in is **sign-in only** — `qa.oidc.provisioning-enabled` stays `false` in production; a
   first-time Google sign-in from an email with no existing local account is refused
   (`OIDC_PROVISIONING_DISABLED`), not auto-provisioned. No new-account creation via Google in MVP.
2. A Google email colliding with an existing local password account keeps today's code behavior:
   refused in-app, resolved by an administrator manually outside the app. No in-app account-linking flow.
   **Fact**: with provisioning off per decision 1, this collision case and `OIDC_EMAIL_NOT_VERIFIED` are
   both currently unreachable for Google specifically — every unlinked Google identity is refused as
   `OIDC_PROVISIONING_DISABLED` one check earlier. The three-reason surfacing in decision 3 is still
   implemented for when a future decision might enable provisioning.
3. `OidcCallbackPage` must surface the specific refusal reason (`OIDC_ACCOUNT_EXISTS` /
   `OIDC_EMAIL_NOT_VERIFIED` / `OIDC_PROVISIONING_DISABLED`) instead of today's single generic message —
   closes a live G7 gap: the BE already sends the reason as `?error=`, the FE discards it
   (`OidcCallbackPage.tsx` branches only on `code`, never reads `error`).
4. Apple Sign-In is **excluded from MVP entirely** — no UI affordance of any kind, not even "coming later".
Enabling Google is almost entirely configuration (`qa.oidc.enabled=true`, a
`spring.security.oauth2.client.registration.google.*` entry with real client credentials) — no new BE
code, no Flyway migration, no Core involvement. `architect`-scoped 2026-10-03; two Figma screen changes
identified (Login screen's federated-sign-in button, three new toast-error copy variants on the same
screen) — see the design proposal for exact text. `security-guardian` review is mandatory before any
future merge of this work, per the standing rule for anything touching auth.

**FR-AUTH-02** (added 2026-10-03, **explicitly deferred — not in this task's scope**) — No FE surface
anywhere (`AdminUsersPage`, personal `SettingsPage`) shows whether an account is federated (signed in via
Google) versus local (password). Consequence found during `architect` review: an administrator's "reset
password" action on a federated account today silently "succeeds" (sends a reset email, 202 response) —
`PasswordResetApplicationService.requestResetByAdmin` never checks `FederatedIdentityPort`, and a
federated user's local-password login is refused regardless of any hash set this way, so the action has
no real effect with nothing telling the admin that. Separately, `FederatedIdentityPort.java`'s own javadoc
is factually wrong — it claims password reset is refused for federated accounts; the code does not do
this. Needs its own `architect` pass (new BE domain surface — a `federated` marker derived from the
existing `FederatedIdentityPort` lookup, no new migration required — plus `AdminUsersPage` and
`SettingsPage` FE changes) and its own product decision (block the reset action outright for a federated
account, or keep it available but clearly labeled as having no effect). Not pulled into `FR-AUTH-01` to
avoid silently expanding that task's scope.

**FR-UX-01 — Figma status update (2026-10-03):** the shared Figma file already depicts both build items
at the fidelity this document calls for — `ProjectSettingsPage`'s frame (node `2325:1475`) shows the
"General" tab already absent (tab order `Members, Integrations, Environments, Execution defaults`,
matching `architect`'s recommended redirect target) and a fully-built "Execution defaults" tab (default
environment + runtime fields, a run-time-configuration variables table, a "prefill preview" side panel,
owner/admin-only save, a read-only-members banner). `SettingsPage`'s frame (node `2325:985`) shows the
"Integrations" tab already built as a live per-project connector-status table (Connection/Sync/Last
activity columns) — this already matches the fuller, "live Connected/Failing + last-tested" fidelity the
user chose over `architect`'s simpler "Configured/Not configured" recommendation (see that architect
proposal for the BE work the live-status columns require: new persisted connection-test-result state,
not yet built). **No further Figma work needed for FR-UX-01.**

---

**The following 14 FRs (`FR-CAT-*`, `FR-MFA-*`, `FR-SESS-*`, `FR-ERR-*`) resolve the items promoted to
MVP 1.0 from `additional-requirements.md` on 2026-10-03** — see that document's updated header for the
promotion decision and each item's original text/rationale. Each FR below went through
`requirements-analyst`, then `architect` where cross-repo/BE work was needed, then a Figma design pass.
None of this is implemented in code.

**FR-CAT-01** (`additional-requirements.md` item 1, "used in N scenarios") — **RESOLVED by explicit user
decision 2026-10-04: project-scoped, not global** (overriding the architect's initial global
recommendation), specifically to close the cross-tenant signal-leak concern a global count would have had
in a shared-SaaS deployment. Implemented design: `GET /api/catalog/steps` (`/api/steps` alias) gains an
optional `projectId` query param; when present and the caller can read that project
(`CatalogAuthorizationPort.canRead`, reusing the catalog BC's existing ACL pattern), each step gains
`usageCount` (per step) and `componentUsageCount` (per component, deduped per scenario at the component
grain too) counting the project's own active `test_scenarios` rows — reusing the
`TestScenarioRepository.countReferencingCustomStep` JSONB-scan technique, generalized to a per-scenario
batch query rather than per-step-id. Absent/malformed/unauthorized `projectId` silently omits the two
fields (200, not 403/404) rather than failing the whole catalog request — a deliberate G3 reading for this
specific case (enrichment of a documented, deliberately-global/documentary endpoint, not primary resource
loading), reviewed and accepted via `security-guardian`. `StepsPage` already had a non-route
active-project mechanism (`getActiveProjectId`, already driving its environment selector) reused as-is —
no new project-selector UI was needed. Figma: `StepsPage` detail pane already shows "Used in 48 scenarios"
in the metadata line (node `2325:2368`) — the badge's visual placement is unchanged by project-scoping,
only the BE semantics changed; no further Figma change needed. Shipped via `qa-platformBE`#138 /
`qa-platformFE`#72.

**FR-CAT-02** (item 2, worked example per component) — Resolved design: a "Worked example" block in the
detail pane showing real Gherkin lines for the step. Sourcing (static/curated vs. generated from a real
stored scenario) remains an open product decision the architect pass deliberately did not resolve —
flagged explicitly: if "generated," the lookup must be deterministic (a real stored scenario), never
LLM-assembled, consistent with the product's rule that AI never invents test data. Figma: built on
`StepsPage`'s detail pane (node `2356:4`, "Worked example" block with sample Gherkin).

**FR-CAT-03** (item 3, copy-pattern-to-clipboard) — Trivial, FE-only, zero open questions. Figma: built — a
"Copy" affordance next to the step pattern/phrase text (node `2356:8`, "Phrase row").

**FR-CAT-04** (item 4, Catalog view search/filter parity) — **RESOLVED by user decision 2026-10-03: add
parity.** The existing `StepsPage` Figma frame already shows a search bar and layer-filter chips present
alongside the Catalog-view tab (node `2325:2335`, "Catalog controls") — matches this decision; no further
Figma change needed. Remaining open question: independent or shared filter state between the Palette and
Catalog views (FE-only detail, does not block design).

**FR-CAT-05** (item 5, Catalog view layer order) — The existing Figma design already uses a fixed,
deliberate chip order (`All, UI, API, DB, Integration`) rather than raw server order — satisfies this
item's intent. Remaining open question (does not block design): whether the FE client-sorts to a shared
order constant (recommended, trivial) or the BE's own response order changes — needs confirming no other
consumer of the catalog endpoint depends on today's order.

**FR-CAT-06** (item 6, blocked-step deep link to the missing credential) — FE-only; the data already exists
end-to-end (`StepStatusItem.missingKeys`, `EnvironmentsPage`'s existing `openUpdateCredential`). Figma:
built — the availability banner in `StepsPage`'s detail pane now carries a "Fix in Environments →" link
(node `2325:2376`).

**FR-CAT-07** (item 7, account-scoped promo-banner persistence) — Resolved design: an additive
`catalog_promo_dismissed` boolean on the existing `user_preferences` table (mirrors `onboardingCompleted`
exactly), replacing the current `localStorage` key. A pre-existing `localStorage` dismissal is **not**
migrated — every user sees the banner once more after this ships, then the account flag takes over
permanently. **No Figma change needed** — the banner's visual is identical regardless of where its
dismissal is persisted; `requirements.md`'s `StepsPage` entry (line ~260-261) should be corrected to say
"account-scoped via `user_preferences.catalog_promo_dismissed`" once implemented.

**FR-MFA-01** (`additional-requirements.md` item 8, "Regenerate recovery codes" actually works, same gap
as `FR-UX-02` above) — Resolved design: re-authentication via password + current TOTP code (reusing
`StepUpVerifier`, the same pattern `disableMfa` already uses), a full replace of the 10-code set (not an
append), reusing `RecoveryCodesDisplay` as-is for the one-time reveal. New BE endpoint
`POST /api/auth/mfa/recovery-codes/regenerate`, new `AuthAuditAction.MFA_RECOVERY_CODES_REGENERATED`.
Figma: built — `SecurityTab`'s enabled-state panel (node `2346:5`) now has a "Regenerate recovery codes"
button; the re-authenticate + new-codes-revealed modal flow is built as two reference frames (nodes
`2346:115`, `2346:131`).

**FR-MFA-02** (item 9, "remember this device for N days") — **RESOLVED by user decision 2026-10-03: the
trusted-devices list/revoke UI is in scope, not deferred.** Needs a new persisted `trusted_devices` entity
(BE, two-instance-safe per G19 — not an in-memory cache), delivered via a long-lived httpOnly cookie, never
substituting for step-up re-authentication (D17.10 stays intact for MFA-disable/regenerate/etc. even from
a trusted device). **Open product decisions, not yet resolved** (do not block the design already built):
default/selectable trust duration in days; whether "remember" fully bypasses the challenge screen or only
changes what it shows; whether a recovery-code login is eligible to mark a device trusted (recommend: no —
weakest-assurance path). Figma: built — `MfaChallengePage` has a "Remember this device for 30 days"
checkbox (node `2346:2`); `SecurityTab`'s enabled-state panel has a "Trusted devices" section with
per-device revoke and "Revoke all" (node `2346:5`, "Trusted devices panel").

**FR-MFA-03** (item 10, alternate second factor) — **RESOLVED by user decision 2026-10-03: WebAuthn/passkey,
addable anytime alongside TOTP (proactive/parallel, not reactive-recovery-only); Apple/SMS/email not
pursued.** Largest item in this batch: generalizes `MfaCredential` from a per-user singleton to a
per-credential row (new nullable WebAuthn columns, a partial unique index keeping "at most one TOTP
credential" as a DB invariant), generalizes `StepUpVerifier` to accept a TOTP-or-WebAuthn proof, adds
WebAuthn ceremony endpoints (register/login options+finish), widens `GET /api/auth/mfa/status` into
`GET /api/auth/mfa/factors`. Recovery codes stay one shared, account-level set regardless of factor count.
**Open product decisions, not yet resolved**: uncapped multiple WebAuthn credentials per account (assumed
yes, e.g. "home key"/"work key"); whether `MfaChallengePage` auto-triggers the WebAuthn browser prompt on
load with TOTP/recovery as fallback links (recommended) or shows an explicit factor-picker first. Figma:
built — `SecurityTab`'s factor list with per-factor "Remove" (node `2346:5`), the method-picker step (node
`2350:16`), the WebAuthn ceremony prompt with its error sub-state (node `2350:28`), and the remove-factor
confirmation modal (node `2350:38`). Standard TOTP QR/code wizard steps were not separately illustrated
(low-novelty, already well-specified by the existing `MfaEnrollmentWizard` code) — build directly from the
architect proposal when implemented.

**FR-SESS-01** (`additional-requirements.md` item 11, `LoginPage` reads `?reason=`) — FE-only, zero
blocking questions. The banner is additive to the existing toast, not a replacement; the `reason` param is
scrubbed from the URL after reading; an unrecognized reason renders no banner. Figma: built — a
"Signed out" reason banner on `LoginPage` (node `2357:2`).

**FR-SESS-02** (item 12, explicit "Retry" on the access-expiry warning) — **RESOLVED by user decision
2026-10-03: "Retry" lets the user trigger the existing single automatic refresh attempt early; a manual
failure still force-logs-out immediately** (not a multi-attempt/paused-countdown redesign). Reuses
`trySilentRefresh()` as-is — `refreshSession()` is already single-flight, so no new coordination is needed
even if a manual click and the automatic t=0 attempt overlap. Pairs with closing the existing FR-AC-14 test
gap for the new code paths this adds. Figma: built — a new `SessionExpiryWarningModal` reference frame
(node `2357:4`) with countdown ring, "Log out now" and "Retry" actions, and an explanatory note that
"Continue session" (idle-reason only) is a separate, pre-existing control not shown in this frame.

**FR-ERR-01** (item 13, extend "Report issue" beyond `RouteErrorFallback`) — **RESOLVED by user decision
2026-10-03: `ExecutionResultsPage` only for this pass** (not every comparable banner in the app), **and the
legacy raw backend-exception-class fallback text is redacted from the report body** (shown on-screen as
today, excluded from the public GitHub report) — avoids repeating the leak pattern already open as HIGH-
severity issues `qa-platformFE#67`/`#68`. Figma: built — a "Report issue" link added to `ExecutionResultsPage`'s
stable-error-code banner (node `2325:2926`).

**FR-ERR-02** (item 14, dedicated full-page 503-degraded state) — **RESOLVED by user decision 2026-10-03:
retarget away from `ExecutionLivePage`** (whose central content streams over WebSocket, not the REST path
this item is about — a 503 there could only come from an ancillary call) **to a page whose main content
genuinely depends on the retried REST call (e.g. `ExecutionResultsPage`'s initial load), reusing the
existing (unscoped) `isServiceDegraded` global signal** rather than building new per-page/per-request
plumbing. The existing global `SystemStatusBanner` should be suppressed on a page showing its own dedicated
panel, to avoid showing the same message twice. **Documentation correction**: `requirements.md`'s existing
"Error screens and messages" section describes the 503 retry-then-toast sequence but omits that
`SystemStatusBanner`/`systemStatus.store.ts` already provides a persistent, shell-wide (if unscoped) banner
for this exact condition — correct this independent of what ships for this FR. Figma: built — a "the
platform is catching up" degraded-state reference panel below `ExecutionResultsPage` (node `2357:18`).

---

**FR-SCOPE-01** — **RESOLVED 2026-10-01.** User decision: option (a) — Swagger, Postman and user-story
import are part of MVP 1.0, not deferred to 1.1. `.claude/CLAUDE.md`'s and `.claude/ORCHESTRATION.md`'s
release-scope statements have been updated accordingly; no code change was needed since the feature was
already fully built, routed and nav-linked. No further action required.

**FR-UX-01** — **Content scope RESOLVED 2026-10-01** by the user, answering the blocking questions
`requirements-analyst` raised on the same date. Final decision, superseding the original "all three must
become functional" disposition for one of the three:

- **`ProjectSettingsPage`'s "General" tab — HIDE, not implemented.** The user chose to hide this tab
  rather than define what it would edit. **Implementation note**: `src/router/index.tsx`'s
  `projects/:projectId/settings` route currently redirects to `general` as the default `:tab` — if
  "General" is removed from the tab list, that default redirect target must move to a remaining tab
  (e.g. "members"), or the route structure needs its own small decision. This is a Trivial-to-small FE
  task (remove a tab, retarget one redirect) — no BE impact.
- **`ProjectSettingsPage`'s "Execution defaults" tab — BUILD.** A project-level **default environment +
  runtime-config, pre-filled into every Run/Execute modal and overridable per run** (reusing the existing
  `ExecutionRuntimeFormValues` shape from `useExecutionLaunchModal.ts` — not a new config shape). This is
  new persisted state (a default differs from "last used," so it needs its own field(s)), which means new
  BE contract surface in `qa-platformBE`'s `project` bounded context — cross-repo impact, Feature-sized.
- **`SettingsPage`'s personal "Integrations" tab — BUILD.** Replace the placeholder with a **real
  cross-project summary**: a list of the user's projects and each one's connector/integration status (not
  a new personal-integration concept — connectors stay per-project). Needs a BE query that can return
  per-project connector status across all of a user's projects (may already be assemblable from existing
  per-project endpoints; `architect` to confirm) — Feature-sized.
- **Edit authorization (both remaining build items)**: **owner + platform admin**, matching the existing
  precedent already implemented on `ProjectMembersTab`.

*Naming collision, still worth flagging to whoever picks this up*: `ProjectSettingsPage` already has a
different, fully-functional "integrations" tab (embeds `ProjectIntegrationsPage`) — distinct from
`SettingsPage`'s tab addressed here. Keep the two clearly distinguished in any UI copy.

**Next step**: call `architect` to scope "Execution defaults" (`ProjectSettingsPage`) and the personal
"Integrations" summary (`SettingsPage`) as Feature-sized tasks (each needs its own BE contract work), and
separately size "hide the General tab" as its own small FE-only task. None of this is implemented yet —
this document records the decision and the next step only.

**Next step**: present the 5 blocking questions above to the user; once answered, call `architect` to
scope this as a Feature per the workspace sizing rule.

---

# Part 4 — Mockups

FR-PLAN-01 and FR-SCOPE-01 resolved as documentation-only changes (no new or changed screen) — neither
needed a mockup. **FR-UX-01 did**: once its content scope was resolved (2026-10-01), two of its three
screens are existing, routed screens with real code today (`ProjectSettingsPage`, `SettingsPage`) — not
genuinely new screens — so per the user's own escalation rule this is an HTML artifact mockup built from
the real component code, not a Figma escalation. The third (the "General" tab) was resolved to **hide**,
not build, so it has no mockup.

**Mockup published**: https://claude.ai/artifact/X5z5gSpWYq62LaKECrNpCm — built from the real
`ExecutionRuntimeConfigFields.tsx` field set, `EnvironmentsPage.tsx`/`HistoryPage.tsx` layout patterns, and
the exact hex values in `src/shared/design/tokens.ts` (both the dark and light palettes, with a theme
toggle). Covers:
- **Mockup A** — `ProjectSettingsPage`'s new "Execution defaults" tab (environment + runtime-config
  pre-fill, owner/admin-only save, "General" shown struck through in the tab bar as removed).
- **Mockup B** — `SettingsPage`'s redesigned "Integrations" tab (cross-project connector-status table).

This is a design proposal, not a final spec — it is explicitly labeled as such in the artifact itself:
exact field names and copy are illustrative pending the BE contract work `architect` still needs to scope.

**Mockup C — added 2026-10-02, a different kind of mockup**: an annotated walkthrough of `StepsPage` as
it exists in code today — not a proposal, a review aid, built because the user specifically asked for
the catalog screen to be visible in the artifact and called it the single most important screen to get
right. It labels each of the four Palette-view containers (promo banner, environment selector,
search + the fixed `API -> WEB -> MOBILE -> DATABASE` layer-chip order, and the two-pane component
list/step-detail layout, including a `BLOCKED_MISSING_ENV` example with its lock badge and action hint,
and a `DEPRECATED` example) with a one-line caption under each explaining what it is and how it behaves,
then separately illustrates the Catalog (i18n) view's `layer -> category -> step` tree so the two views'
different structures (and the fact that the Catalog view's layer order comes from the server response
rather than the Palette view's own `LAYERS` constant) are visible side by side. The artifact's own banner
makes the distinction explicit so a reader does not mistake the annotations for a proposed redesign.

# A·Spectra — Additional Requirements (promoted to MVP 1.0 on 2026-10-03)

Added 2026-10-02, alongside the `requirements.md` deep-dive review of the catalog screen, two-factor
authentication, session expiration, and error screens/messages, originally framed as a non-MVP parking
lot. **Status changed 2026-10-03: the product owner explicitly promoted every item below into MVP 1.0
scope**, as part of the same decision that added Google SSO sign-in (see `requirements.md`'s `FR-AUTH-01`).
Each item has since been through a `requirements-analyst` pass (resolving what could be resolved,
surfacing the rest as explicit product decisions, several of which the user then answered directly) and,
for the items needing backend/architecture work, an `architect` pass. The resolved design for each item
is now recorded in `requirements.md` as a numbered FR (`FR-CAT-*` for the catalog items, `FR-MFA-*` for
two-factor, `FR-SESS-*`/`FR-ERR-*` for session/error screens) — this file keeps the original item text and
review rationale for traceability, but the FRs in `requirements.md` are the current source of truth for
scope and design. Figma design work for every item below is complete in the shared Figma file (fileKey
`p06HxSrh2afAdXopnk4Qvs`) except where a Figma impact is explicitly marked "none" per the architect
proposal (item 7). No code has been implemented for any of these items — this remains a documentation
and design record only; the normal `architect` → `test-strategist` → `implementer` pipeline still applies
before any of this ships.

Each item names the screen/behavior it extends, why it came up during the review, and whether it needs
a BE contract change (cross-repo, Feature-sized) or is FE-only.

---

## Catalog screen (`StepsPage`)

1. **"Used in N scenarios" on each component/step.** The review found no usage-count signal anywhere in
   the catalog — a new author cannot tell whether a step is proven-in-use or obscure. Needs a BE
   aggregation query (cross-repo, Feature-sized) across existing scenarios' step references.
2. **A worked example per component**, shown in the detail pane — the real Gherkin lines a scenario
   using this component looks like, not just the bare pattern and parameter list. This is the single
   change most directly aimed at the "eager to keep exploring" goal the user named: today the detail
   pane proves a step exists and what it needs, but not what using it actually looks like end to end.
   FE-only if sourced from a static/curated example per component; cross-repo if generated from real
   stored scenarios.
3. **Copy-pattern-to-clipboard** on each step row — a small affordance next to the monospace pattern.
   FE-only, trivial.
4. **Catalog (i18n) view parity**: today only the Palette view has search and layer filters; the Catalog
   view is a plain, unfiltered tree. Giving it the same search/filter bar (or deciding it is
   intentionally a different, simpler reading mode and documenting that decision) is a product call,
   not yet made either way.
5. **Catalog view's layer order should probably match the Palette view's fixed `API, WEB, MOBILE,
   DATABASE` order** instead of whatever the backend response order happens to be — low effort, but
   needs a product/architecture decision on whether the client should re-sort rather than trust the
   server order, and whether the server's own ordering should simply be changed to match instead.
6. **A clickable path from a `BLOCKED_MISSING_ENV` step straight to the missing credential field** on
   `EnvironmentsPage`, instead of only a text hint. Would turn "this step is blocked" into a one-click
   fix rather than a dead end, which matters for a first-time client's confidence in the product.
7. **First-run promo banner could survive a browser change** by being account-scoped (a backend flag or
   user preference) rather than only `localStorage` — today a client who clears site data or opens a
   second device sees the banner again, which is not wrong but is inconsistent with "permanent" framing
   in the current copy. BE-contract change if pursued.

## Two-factor authentication

8. **Make "Regenerate recovery codes" actually work.** Already flagged as `FR-UX-02` in `requirements.md`
   because the control exists and is always clickable today — repeated here because, if the product
   decision is to defer it rather than fix it for 1.0, this is where that deferred scope belongs. Needs
   a new BE endpoint (re-authenticated the same way enrollment is) plus the FE flow to show the new
   codes exactly once, mirroring the enrollment wizard's own `recovery` step.
9. **A "remember this device for N days" option** on the login-time MFA challenge, so a recognized
   device does not re-prompt every login. Cross-repo (BE device/session trust state).
10. **An alternate second factor** (WebAuthn/passkey, or SMS/email as a fallback) alongside TOTP, for a
    user whose authenticator app is unavailable and who has exhausted recovery codes. Large, cross-repo;
    explicitly out of scope unless requested.

## Session expiration

11. **`LoginPage` reading and displaying the `?reason=` query parameter** after a forced logout, instead
    of relying solely on the toast shown just before the hard page reload (which may already have
    dismissed by the time the reload completes). Noted inline in `requirements.md`'s cross-cutting
    section as a fact pending confirmation this is wanted; repeated here as the concrete enhancement if
    so. FE-only, small.
12. **An explicit "retry" action on the access-expiry warning**, not just "log out now" — today only the
    idle-timeout warning offers "Continue session" because the access-expiry path already attempts one
    silent refresh automatically before the warning ever appears; a manual retry button would cover the
    case where that automatic attempt failed due to a transient network issue rather than an actually
    expired/revoked session. FE-only.

## Error screens and messages

13. **Extend the "Report issue" pre-filled-link pattern** (today only on `RouteErrorFallback`/
    `ForcedNotFoundPage`) to other hard-error states in the app — for example `ExecutionResultsPage`'s
    stable-error-code banner — so a user hitting any dead end, not just a routing error, has the same
    one-click path to file a usable bug report. FE-only, moderate (needs the same context-building logic
    reused per page).
14. **A dedicated, friendlier empty/degraded state for the `503` service-unavailable retry sequence**
    (today a `toast.loading` placeholder followed by an error toast after 3 retries) on pages where a
    long-running action is central — e.g. a full-page "the platform is catching up, retrying..." panel
    on `ExecutionLivePage` instead of a toast easily missed while watching a live run.

---

**Next step for any of the above**: present to the user for prioritization; anything picked up moves into
`requirements.md` as a numbered FR once scoped, and follows the normal `requirements-analyst` ->
`architect` -> `test-strategist` -> `implementer` pipeline from the workspace `ORCHESTRATION.md`.
