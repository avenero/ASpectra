# Tasks 017 — Public landing

**Feature** The public, unauthenticated landing: what it explains, the two ways in, language, footer routes, and what it never does · **Tasks version** 0.1 · **Date** 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Governed by** `constitution.md` v1.8 · **Derived from** `specPublicLanding.md` v0.1 (requirements RF-x.y, RNF-n, LC-n, US-n; decisions Q1–Q8, D-a, D-b) and `planPublicLanding.md` v0.1 (modules §2, decisions D1–D12, tests §5, register assumptions and cross-plan item §6). Every task names the plan element it builds and the requirements it covers, and has one verifiable "Done when" line. Test names in "Done when" lines are taken verbatim from plan §5.2. Tasks are sized at 20–30 minutes and listed in dependency order. "after" names the tasks that must be complete first, when that is not simply the previous task.

Tasks marked **[001]** / **[002]** / **[006]** touch an element that plan 001 (Login), 002 (Signup and verification) or 006 (Sessions) builds. The element is extended or consumed, never duplicated. If the owning feature is not built yet, its route is stubbed in the shape its plan states (`/sign-in`, `/sign-up`). **No back-end task exists:** plan §2.1 adds nothing to the back end. Test-only tools: Vitest, Testing Library, Playwright, axe. Static delivery and its headers are the A-66 assumption. Tasks build the in-document rules and a local preview server that applies the assumed header set, so the e2e suites run against the assumption.

Phases:
- **A** foundations
- **B** pure model
- **C** shared components
- **D** views
- **E** interaction
- **F** content and copy
- **G** app composition and build
- **H** verification suites
- **I** closing

Phases B, C and F depend only on phase A and may run side by side. Phase D needs B and C. Phase G needs D. Phase H needs G.

---

## A · Foundations

- [ ] **T-01 · Feature module skeleton** — create `src/features/public-landing` with the folders of plan §2.2 (`model`, `views`, `interaction`, `index` public API exporting only `landingRoute` and `LandingDocument`), registered nowhere yet. *(plan §2.2, D1; RF-1.1)*
  **Done when:** the module type-checks, and its public API exports exactly `landingRoute` and `LandingDocument` (a Vitest export-surface test lists two names).

- [ ] **T-02 · Module import and API lint rule** — a lint rule scoped to `public-landing` forbids importing `shared/api`, `shared/errors`, `shared/security`, TanStack Query, Zustand, the STOMP client, axios and `ProtectedBoundary` **[006]**. It also forbids referencing `document.cookie`, `localStorage`, `sessionStorage`, `indexedDB`, `location.search` and `URLSearchParams`. From `identity-access` **[001/002]**, only the exported route paths may be imported. *(plan §2.2 "Not used", D1, D2, D6; RF-1.4, RF-10.3, RF-12.2, RF-14.1, RF-11.1)* — after T-01
  **Done when:** the lint passes on the empty module, and a fixture file containing each forbidden import or reference produces one lint error per item (checked by a lint test).

- [ ] **T-03 · `shared/i18n` start state without detector** — change the i18next instance creation to `lng: 'en'` and `fallbackLng: 'en'`, with no language-detector plugin, no cache and no storage back end. Add `public-landing` as a namespace. ES and FR resources become on-demand same-origin static chunks. `document.documentElement.lang` follows `languageChanged`. *(plan §2.2 Shared used `shared/i18n`, D5; RF-6.1, RF-6.4, RNF-7, Q2)* — after T-01
  **Done when:** a Vitest unit test with `navigator.languages = ['fr-FR', 'es']` and preset storage entries shows the instance starts in `en` and reads no storage, and `i18n.startsInEnglishWhateverBrowserPrefers` (unit part) passes. The existing `identity-access` i18n tests of plans 001 and 002 still pass.

## B · Pure model

- [ ] **T-04 · `sectionAnchors`** — one ordered list of stable anchor ids: `workflow`, `capabilities`, `human-control`, `faq` for the header and menu, plus ids for every other section heading (hero, sources, catalog, suggestions, consent, evidence, teams, begin, closing). It exports a `headerLinks` subset of the four. *(plan §2.2 `sectionAnchors`, D8; RF-9.1, RF-9.2)* — after T-01
  **Done when:** a unit test asserts unique, lowercase, stable ids and that `headerLinks` is exactly the four, in R01's order.

- [ ] **T-05 · `resolveFragment(hash)`** — pure function: a known anchor id → that section, and anything else (empty, unknown, encoded markup, a query-like string) → `TOP`. It reads only its argument. *(plan §2.2 `resolveFragment`, D8; RF-9.3, RF-9.4, RF-11.1, LC-19, LC-20)* — after T-04
  **Done when:** `fragment.unknownOpensTop` (unit part) passes over known ids, an unknown id, `#<script>`, `#%3Cimg%3E` and an empty hash.

- [ ] **T-06 · `EntryActions`** — a closed list of exactly two values, `CREATE_WORKSPACE → /sign-up` and `SIGN_IN → /sign-in`, using the route paths exported by `identity-access` **[001/002]** (stubbed with those paths if not yet built). There is no federation, passkey or third value. *(plan §2.2 `EntryActions`, D-a, D-b; RF-2.1, RF-2.3, RF-2.5, RF-8.1, RF-8.2)* — after T-02
  **Done when:** `entryActions.exactlyTwoDestinations` and `entry.noFederationOrPasskey` (unit part) pass.

- [ ] **T-07 · `BaselineCapabilityList`** — the capability keys the copy may name (HTTP, Web, DB, versioned tenant macros, structured authoring, docs and user stories, OpenAPI and Postman sources, static validation, human approval, base report and evidence, delivery, roles and grants). Fixtures and mobile are excluded (A-02, A-08), and the list documents the S-61 assumption. *(plan §2.2 `BaselineCapabilityList`, D11; RF-3.4, RF-4.3, RNF-11)* — after T-01
  **Done when:** a unit test asserts the list has no `mobile` or `fixture` key and contains `http`, `web` and `db`.

- [ ] **T-08 · `landingContent` structure** — the ordered section structure as i18n keys only: per section its anchor (T-04), eyebrow, heading, lead, cards (each with a capability key from T-07 where it names one), the illustrative preview fields, and the FAQ items. It contains no literal text. *(plan §2.2 `landingContent`, D11; RF-1.2, RF-1.3, RF-5.1, RF-5.2, RF-5.3)* — after T-04, T-07
  **Done when:** `content.sectionsInDesignedOrder` passes (the order matches R01 sections 01–10, closing, footer), and a unit test finds no string in the structure that is not an i18n key.

## C · Shared components

- [ ] **T-09 · `shared/ui` `IllustrativeLabel`** — a visible "Illustrative" text label with its i18n key in EN, ES and FR. It is exposed as the first text node of its container, so assistive technology reads it before the content. *(plan §2.2 Shared used `shared/ui`, D11; RF-4.1, LC-24)* — after T-03
  **Done when:** a Testing Library test finds the label by its text in all three languages, and it comes first in the container's accessible name.

- [ ] **T-10 · `shared/ui` `ExternalLink`** — a link to an absolute address with a visible "opens outside A·Spectra" cue and the same wording as its accessible description. It sets `rel="noopener noreferrer"` and `referrerpolicy="no-referrer"`. It refuses a relative or same-origin address at type level and at run time in development. *(plan §2.2 `shared/ui`, D6; RF-7.2, RF-7.4)* — after T-03
  **Done when:** a unit test shows the cue text, the accessible description, both attributes, and a development-time error for a same-origin address.

- [ ] **T-11 · `shared/ui` `SkipLink`** — "Skip to content", first in tab order, visible on focus, targeting the main landmark. *(plan §2.2 `shared/ui`; RNF-1)* — after T-03
  **Done when:** a Testing Library test shows it is the first focusable element and moves focus to `main` on activation.

## D · Views

- [ ] **T-12 · `IllustrativePreview`** — the only component allowed to render a sample scenario, identifier, environment, count or outcome. It renders `IllustrativeLabel` first, and renders every sample identifier with the `sample:` mark through a typed `SampleId` prop (plain strings are not accepted). It holds the hero checkout preview and its workflow strip as fields from `landingContent`. *(plan §2.2 `IllustrativePreview`, D11; RF-4.1, RF-4.2, RNF-2, LC-24)* — after T-08, T-09
  **Done when:** `preview.alwaysLabelledIllustrative` and `preview.sampleIdentifiersMarked` (unit parts) pass, and a type test shows a raw string cannot be passed as an identifier.

- [ ] **T-13 · `HeroSection`** — eyebrow, headline, lead, "Create workspace →" and "Explore the workflow" (to `#workflow`), the hero statement "Verified email. Optional AI. Human approval and Run are always separate.", and the `IllustrativePreview`. The statement sits in the first viewport at 1440, 768 and 390 CSS px. *(plan §2.2 `HeroSection`; RF-2.2, RF-3.3, RF-3.5, RF-4.1, RF-9.1)* — after T-06, T-12
  **Done when:** a Testing Library test finds both controls, the statement and the preview label, and "Explore the workflow" has `href="#workflow"`.

- [ ] **T-14 · `WorkflowSection` and `CapabilitiesSections`** — section 02 (three steps, the Run step stating that a separate user decision starts the runner) and sections 03–04 (sources and the catalog layers). Each capability card is driven by its `BaselineCapabilityList` key. Copy keys follow the S-61 assumption (no "Mobile" card, HTTP and Web, and DB as its own card). *(plan §2.2 section components, D11; RF-1.2, RF-3.1, RF-3.3, RF-3.4, RF-4.3)* — after T-08
  **Done when:** a Testing Library test renders the sections with their anchors, and every card carries a key present in `BaselineCapabilityList`.

- [ ] **T-15 · `HumanControlSections` and `BeginSection`** — sections 05–08 (suggestions are not facts, exact context and explicit consent, evidence, team boundaries) with the `human-control` anchor on section 05, and section 09 (verify email, protect and configure, first scenario). There is no fixture wording in the consent section (S-61). *(plan §2.2 section components; RF-1.2, RF-3.1, RF-3.2, RF-3.3, RF-3.4)* — after T-08
  **Done when:** a Testing Library test finds the AI-optional, approval-separate and verified-email statements, and no rendered text contains a fixture term in EN.

- [ ] **T-16 · `FaqSection`** — the three questions and answers as visible headings and paragraphs, with no disclosure widget, under the `faq` anchor. *(plan §2.2 `FaqSection`; RF-5.1, RF-5.2, RF-5.3, RF-5.4)* — after T-08
  **Done when:** `faq.threeAnswersPresent` (unit part) passes, and no `details`, `aria-expanded` or hidden answer exists in the rendered output.

- [ ] **T-17 · `LanguageSwitch`** — a labelled group of three choices, EN / ES / FR. The current one is marked as selected in text (for example "EN · current", per the S-62 assumption) and in its accessible state. Choosing one calls `changeLanguage`, keeps the scroll position, and returns focus to the chosen control. *(plan §2.2 `LanguageSwitch`, D5; RF-6.2, RF-6.5, LC-9)* — after T-03
  **Done when:** `languageSwitch.currentShownAsText` passes, and a unit test shows `scrollY` and the focused control unchanged after switching.

- [ ] **T-18 · `PublicDocumentLinks`, `LandingFooter`, `ClosingCta`** — Privacy, Terms and Contact as `ExternalLink`s fed only by the validated `PublicDocuments` value (T-29), plus the wordmark, `LanguageSwitch` and the copyright line. The closing section has "Give every run a reason.", "Create workspace →" and "Sign in". *(plan §2.2 `LandingFooter`, `PublicDocumentLinks`, `ClosingCta`, D7; RF-2.2, RF-7.1, RF-7.2, RF-7.5)* — after T-06, T-10, T-17
  **Done when:** `footer.routesExternalWithCue` (unit part) and `footer.copyrightLine` pass with a test `PublicDocuments` value.

- [ ] **T-19 · `LandingHeader`** — wordmark, the four `headerLinks` as fragment anchors, "Create workspace" and "Sign in" (S-60 assumption) at desktop and tablet widths, and the "Menu ☰" control at phone width. *(plan §2.2 `LandingHeader`; RF-2.2, RF-9.1, S-60)* — after T-04, T-06
  **Done when:** a Testing Library test at desktop width finds both entry actions and four anchors, and at phone width finds the menu control instead.

- [ ] **T-20 · `LandingMenu`** — the M01 sheet with the four anchors and the two entry actions only. Focus is trapped while open, and Escape, an anchor or an entry closes it. Focus returns to the menu control on close, and opening it never navigates. *(plan §2.2 `LandingMenu`; RF-9.6, LC-22)* — after T-19
  **Done when:** `menu.twoActionsFourLinksFocusTrapped` (unit part) passes: six items, Tab cycles inside, and Escape closes with focus returned.

- [ ] **T-21 · `LandingPage` composition and breakpoints** — `SkipLink`, header, `main` with the sections in `landingContent` order, footer. The R01 layout is at desktop, the tablet width reflows R01 under the desktop rules (S-02), and M01 is at phone widths, all by Tailwind breakpoints in one component. The `LandingDocument` root takes `PublicDocuments` and renders `LandingPage`. *(plan §2.2 `LandingPage`, `LandingDocument`, F-X-07; RF-1.2, RF-15.1, RF-15.2, RF-15.3)* — after T-11, T-13 … T-20
  **Done when:** a Testing Library test renders the page at three widths and finds the same set of statement keys, illustrative labels, FAQ answers, entry actions, language choices and footer routes at each.

## E · Interaction

- [ ] **T-22 · `useEntryNavigation()`** — navigates to an `EntryActions` destination with a fixed path, no query and no history state. It ignores further activations until the navigation settles. It is wired into every entry control of the header, menu, hero and closing section, which keep a plain `href` for the no-script case. *(plan §2.2 `useEntryNavigation`, D4, D12; RF-2.6, RF-2.7, LC-14, LC-17)* — after T-21
  **Done when:** `entryNavigation.doubleActivationOpensOnce` (unit part) passes, and a unit test shows the navigation call carries no query and no state.

- [ ] **T-23 · `useSectionFocus()`** — on initial load and on every fragment change, resolves through `resolveFragment` and scrolls to the section. The scroll is instant when `prefers-reduced-motion` is set. Focus moves to the section heading (made focusable with `tabindex="-1"`). `TOP` leaves focus on the document start. *(plan §2.2 `useSectionFocus`, D8; RF-9.3, RF-9.5, LC-19, LC-26)* — after T-05, T-21
  **Done when:** a unit test shows focus on the FAQ heading after `#faq`, and no smooth scrolling under reduced motion.

- [ ] **T-24 · `useStripQuery()`** — after hydration, if the address has a query, replaces it with the same path and fragment and no query, using `history.replaceState` with no new entry. It never reads, decodes or displays the query values. *(plan §2.2 `useStripQuery`, D6; RF-11.1, RF-11.3, RF-11.4, RNF-8)* — after T-21
  **Done when:** a unit test starting at `/?next=https://evil.example&token=abc#faq` ends at `/#faq` with the same history length, and the lint rule of T-02 still passes (the hook uses only `location.pathname` and `location.hash`).

## F · Content and copy

- [ ] **T-25 · EN bundle `public-landing.en.json`** — every key of `landingContent`, header, menu, footer, preview, labels and meta (title and description). The copy follows R01 with the S-61 corrections: no "Mobile + DB" card, no mobile profiles in the environment preview, no fixture step (the hero GIVEN becomes a baseline step), an added Web and DB example, and R01 and M01 statements aligned. Every sample identifier uses the `sample:` form. *(plan §2.2 `landingContent`, D11; RF-1.3, RF-3.1, RF-3.2, RF-3.3, RF-3.4, RF-4.3, RF-4.4, RF-5.1, RF-5.2, RF-5.3, RF-13.3, S-61)* — after T-08
  **Done when:** `content.s61CorrectionsApplied`, `content.boundaryStatementsPresent`, `content.examplesHttpWebDbOnly` and `faq.threeAnswersPresent` pass on the EN bundle.

- [ ] **T-26 · ES and FR bundles** — `public-landing.es.json` and `public-landing.fr.json` with the same key set. The interim copy is produced from EN and marked for review (S-62 assumption). Executable text such as Gherkin keywords and aliases stays untranslated in the examples. *(plan §2.2 `shared/i18n`, D11; RF-6.2, RF-6.6, RNF-7)* — after T-25
  **Done when:** `content.bundleKeyParity` passes, and a content test shows the GIVEN / WHEN / THEN example strings are byte-identical across the three bundles.

- [ ] **T-27 · Content guard suite** — Vitest content tests over the three bundles:
  - every capability card key is in `BaselineCapabilityList`;
  - no percentage, metric, customer name, logo key, testimonial or federation / passkey wording appears;
  - removing a key from the list makes the suite fail.
  The suite is wired into the build so a failure stops it. *(plan §5.1 Content, D11; RF-3.4, RF-4.4, RF-8.2, RNF-2, RNF-11, LC-29, LC-31)* — after T-26
  **Done when:** `content.capabilitiesWithinBaseline`, `content.noMetricsLogosTestimonials`, `content.removedCapabilityFailsBuild` and `entry.noFederationOrPasskey` pass, and the build exits non-zero when one is made to fail.

## G · App composition and build

- [ ] **T-28 · `app/routes` registration** — register `landingRoute` at `/` outside the session provider and outside `ProtectedBoundary` **[006]**. `identity-access` routes **[001/002]** load as lazy chunks. No session query, refresh or realtime connection is started on `/`. *(plan §2.2 `app/routes`, D4; RF-10.1, RF-10.3, RF-14.1, RNF-6)* — after T-22
  **Done when:** a Vitest app-composition test mounting `/` records zero calls to the session refresh **[006]** and to the transport, and the landing chunk contains no `identity-access` view module (checked against the build manifest).

- [ ] **T-29 · `PublicDocumentsConfig`** — build-time reader of the nine operator destinations (Privacy, Terms, Contact × EN, ES, FR; register `122:2` §15) from the release configuration (A-65). Each must be present, absolute, `https`, and not the product's public origin; no fallback between languages. A violation fails the build and names the value. The validated object is passed to `LandingDocument`. *(plan §2.2 `PublicDocumentsConfig`, D7; RF-7.3, RNF-9, LC-11, P13)* — after T-01
  **Done when:** `publicDocuments.missingFailsBuild` and `publicDocuments.malformedFailsBuild` pass (absent, relative, `http`, same-origin each fail with the value's name), and a valid set builds.

- [ ] **T-30 · `app/entries/landing` prerender** — a build step using Vite's native server build and React's server renderer renders `LandingDocument` in EN to `landing.html`. The document is complete, with:
  - `lang="en"`, the title and description from the EN meta keys and `robots` set to `index`;
  - a CSP meta (`default-src 'self'`, no inline script, no other origin) and `referrer` set to `no-referrer`;
  - preload of the self-hosted fonts and the hashed module script for hydration;
  - all anchors as ids.
  No inline script is emitted. *(plan §2.2 `app/entries/landing`, D3, D6, D9, D10; RF-1.1, RF-5.4, RF-9.2, RF-13.1, RF-13.3, RF-14.4)* — after T-21, T-25, T-29
  **Done when:** the build emits `landing.html`, and `anchors.stableIdsInPrerender`, `faq.visibleWithoutScripts` and `meta.titleAndDescription` pass over it.

- [ ] **T-31 · Hydration** — the browser entry hydrates `landing.html` (no client re-render of different markup) and then attaches `useStripQuery`, `useSectionFocus`, `LanguageSwitch` and `LandingMenu` behaviour. *(plan §2.2 `LandingDocument`, D3, D4; RF-6.2, RF-9.3, RF-11.1, RF-14.4)* — after T-23, T-24, T-28, T-30
  **Done when:** a Playwright smoke run on the local preview shows no hydration-mismatch warning in the console, and the language switch works after load.

- [ ] **T-32 · `app/entries/shell` and `robots.txt`** — the SPA shell document `app.html` for every route other than `/`, with `robots` set to `noindex` and the same CSP and referrer meta. A static `robots.txt` allows `/` and disallows everything else. *(plan §2.2 `app/entries/shell`, `robots.txt`, D10; RF-13.1, RF-13.2, RNF-10, LC-27)* — after T-30
  **Done when:** `robots.landingIndexOnly`, `robots.shellNoindex` and `robots.txtAllowsRootOnly` pass over the build output.

- [ ] **T-33 · Self-hosted assets and `LandingBuildChecks`** — move every font and image the landing uses into `shared/assets` as hashed files. A post-build check over `landing.html`, `app.html` and the landing's assets fails the build on any absolute URL to another origin, on an inline script, or on a missing CSP or robots meta. *(plan §2.2 `shared/assets`, `LandingBuildChecks`, D9; RF-12.1, RNF-4)* — after T-30, T-32
  **Done when:** `build.noExternalOrigin` passes, and inserting a font CDN link makes the build fail.

- [ ] **T-34 · Local preview server with the A-66 header set** — a test-only preview configuration that serves `landing.html` at `/` (for any query), `app.html` for every other route, the assets and `robots.txt`. It applies the assumed headers: CSP, `Referrer-Policy: no-referrer`, `X-Robots-Tag: noindex` except on `/` and `robots.txt`, and no `Set-Cookie`. The e2e suites run against it. *(plan §6 A-66; RF-11.2, RF-12.1, RF-13.2, RF-14.2)* — after T-32
  **Done when:** a Playwright request test shows `/` and `/?x=1` return the same `landing.html` bytes, `/sign-in` returns `app.html`, and the header set is present with no `Set-Cookie`.

## H · Verification suites

- [ ] **T-35 · Entry and routing e2e** — Playwright with stubbed `/sign-in` and `/sign-up` targets **[001/002]**. *(plan §5.2; RF-1.1, RF-2.1, RF-2.2, RF-2.3, RF-2.4, RF-2.5, RF-2.6, RF-2.7, RF-8.1, RF-8.2, LC-3, LC-14, LC-15, LC-17, LC-31, D-a, D-b)* — after T-34
  **Done when:** `landing.opensWithoutSession`, `entryActions.reachableFromTopAtEveryWidth`, `createWorkspace.opensSignupAndCreatesNothing`, `signIn.opensSignInScreen`, `entryActions.passOnlyLanguage`, `entryNavigation.doubleActivationOpensOnce`, `back.fromSignupShowsLandingUnchanged` and `entry.noFederationOrPasskey` pass.

- [ ] **T-36 · Identity-independence and session e2e** — two browser contexts, one with a valid session cookie **[006]** and one without; the landing documents and rendered DOM are compared. *(plan §5.2; RF-10.1, RF-10.2, RF-10.3, RF-10.4, RNF-3, LC-1, LC-2, LC-30, Q1)* — after T-34
  **Done when:** `identity.signedInAndAnonymousDocumentsIdentical`, `session.notReadRefreshedOrEnded` and `signedIn.entryActionsFollowS26` pass.

- [ ] **T-37 · Network, privacy and outage e2e** — request interception recording every request, cookie and storage write while opening and using the landing (scroll, switch language, open menu, follow anchors). A second run has every product service stubbed down. *(plan §5.2; RF-1.4, RF-6.4, RF-12.1, RF-12.2, RF-12.3, RF-14.1, RF-14.2, RF-14.3, RNF-4, RNF-5, LC-13, Q4, Q5)* — after T-34
  **Done when:** `network.noProductRequestOnOpenOrUse`, `network.noThirdPartyRequest`, `storage.noCookieNoStorageAfterVisit`, `consent.noBannerExists` and `outage.landingRendersActionsEnabled` pass.

- [ ] **T-38 · Address-parameter adversarial e2e** — the landing opened with redirect targets, an invitation token, a pre-filled email, `lang=fr`, markup and script payloads, and very long queries. *(plan §5.2; RF-6.1, RF-11.1, RF-11.2, RF-11.3, RF-11.4, RNF-8, LC-8, LC-10, LC-16, LC-18, Q7, US-12)* — after T-34
  **Done when:** `params.redirectIgnored`, `params.invitationTokenIgnoredAndStripped`, `params.markupNeverRendered`, `params.queryStripped` and `i18n.queryLangIgnored` pass.

- [ ] **T-39 · Language e2e** — browser locales `es-ES`, `fr-FR` and `de-DE`; switch and carry-over to sign-in and signup **[001/002]**; reload and a new context the next "day". *(plan §5.2; RF-6.1, RF-6.2, RF-6.3, RF-6.4, RF-6.5, LC-4, LC-5, LC-6, LC-7, LC-9, Q2)* — after T-34
  **Done when:** `i18n.startsInEnglishWhateverBrowserPrefers`, `languageSwitch.keepsPosition`, `languageSwitch.carriesIntoSignInAndSignup`, `i18n.notRememberedAfterReload` and `languageSwitch.currentShownAsText` pass.

- [ ] **T-40 · Fragment, menu and footer e2e** — deep links to known and unknown sections, the M01 menu at 390 px, the footer routes with stubbed external documents (one reachable, one unreachable). *(plan §5.2; RF-7.1, RF-7.2, RF-7.4, RF-9.3, RF-9.4, RF-9.5, RF-9.6, LC-12, LC-19, LC-20, LC-22, Q3, Q6)* — after T-34
  **Done when:** `fragment.knownSectionOpensWithFocus`, `fragment.unknownOpensTop`, `menu.twoActionsFourLinksFocusTrapped`, `footer.routesExternalWithCue`, `footer.noReferrerSent` and `footer.unreachableDocumentLeavesLandingIntact` pass.

- [ ] **T-41 · Content, preview and no-script e2e** — the built page at three widths with scripts enabled and with scripts disabled. *(plan §5.2; RF-1.2, RF-1.3, RF-3.1, RF-3.2, RF-3.3, RF-3.5, RF-4.1, RF-4.2, RF-5.4, RF-14.4, RF-15.1, RF-15.2, RF-15.3, LC-21, LC-23, LC-24)* — after T-34
  **Done when:** `hero.statementAboveFoldAllWidths`, `preview.alwaysLabelledIllustrative`, `preview.sampleIdentifiersMarked`, `noScript.contentActionsFooterWork`, `responsive.sameStatementsAllWidths` and `responsive.tabletReflow` pass.

- [ ] **T-42 · Indexing and link-preview checks** — run the build-output checks of T-32 and T-30 inside the e2e pipeline against the preview server, including `X-Robots-Tag` on `/sign-in`, `/sign-up`, `/verify-email`, an invitation route and an authenticated route. *(plan §5.2; RF-13.1, RF-13.2, RF-13.3, RNF-10, LC-27, LC-28, Q8)* — after T-34
  **Done when:** `robots.landingIndexOnly`, `robots.shellNoindex`, `robots.txtAllowsRootOnly` and `meta.titleAndDescription` pass against the served responses.

- [ ] **T-43 · Automated accessibility** — axe in Playwright at 1440, 768 and 390 CSS px in EN, ES and FR, with the menu open and closed. Also a 320 px reflow check, 400 % zoom at desktop, and reduced motion. *(plan §5.1 Accessibility; RNF-1, LC-25, LC-26, P14)* — after T-41
  **Done when:** `a11y.axeAllWidthsAllLanguages`, `a11y.reflow320AndZoom400` and `a11y.reducedMotion` pass with zero AA violations.

- [ ] **T-44 · Performance** — Playwright with the reference slow-phone profile (throttled CPU and slow mobile network) over the preview server. *(plan §5.1 Performance; RNF-6)* — after T-41
  **Done when:** `perf.heroUsableWithinThreeSecondsSlowPhone` and `perf.noLayoutShift` pass on three consecutive runs.

- [ ] **T-45 · Manual assistive-technology run** — NVDA (Chrome, Firefox) and VoiceOver (Safari macOS, iOS) at the versions pinned in the release-support matrix. Check reading order, skip link, headings, the preview announced as illustrative, menu focus, the current language announced, the external-link cue, and focus after in-page moves, in EN, ES and FR. *(plan §5.1 Manual assistive; RNF-1, RF-4.1, RF-6.5, RF-7.2, RF-9.5, RF-9.6, LC-24)* — after T-43
  **Done when:** the manual checklist is filled for each pairing with no blocking finding, and the report is attached to the release evidence.

- [ ] **T-46 · Copy-discipline inspection** — confirm that no landing code path records a visit (RF-12.4, inspection row of plan §5.2), and that the rendered EN page contains every RF-3 statement and no capability outside the baseline at release. *(plan §5.2 inspection row; RF-12.4, RF-3.4, RNF-2, RNF-11)* — after T-27, T-37
  **Done when:** the inspection note is recorded with the commit reviewed, and `content.capabilitiesWithinBaseline` passes on the release build.

## I · Closing

- [ ] **T-47 · Requirement coverage check** — script listing every RF-x.y, RNF-n and LC-n of the spec against the passing test names of plan §5.2 and this file. *(spec §7 item 2)* — after T-35 … T-46
  **Done when:** the script reports 107 of 107 identifiers (65 RF, 11 RNF, 31 LC) covered by at least one passing test.

- [ ] **T-48 · Register-dependency and cross-plan review** — walk §6 of the plan (A-65, A-66, A-02, A-08; S-60, S-61, S-62, S-26, S-02). For each, confirm the built assumption matches the current register state or record the deferral. Also confirm that the plan 002 cross-plan item (Privacy and Terms as `PublicDocumentLinks` with the A-65 addresses) is either approved and applied or deferred by the product owner. *(spec §7 item 13)* — after T-47
  **Done when:** each of the nine register items has "resolved as built", "resolved otherwise — change ticket raised" or "deferred by product owner" recorded beside it, and the plan 002 item has its decision recorded.

- [ ] **T-49 · Completion criteria sign-off** — tick spec §7 items 1–13 with their evidence (test run id, build log, a11y report, manual checklist, performance report, coverage report, register review). *(spec §7)* — after T-48
  **Done when:** all thirteen items are ticked with a link to their evidence.

---

## Coverage index

| Requirement | Tasks |
|---|---|
| RF-1.1 | T-01, T-30, T-35 |
| RF-1.2 | T-08, T-14, T-15, T-21, T-41 |
| RF-1.3 | T-08, T-25, T-41 |
| RF-1.4 | T-02, T-37 |
| RF-2.1 | T-06, T-35 |
| RF-2.2 | T-13, T-18, T-19, T-35 |
| RF-2.3 | T-06, T-35 |
| RF-2.4 | T-35 |
| RF-2.5 | T-06, T-35 |
| RF-2.6 | T-22, T-35 |
| RF-2.7 | T-22, T-35 |
| RF-3.1 | T-14, T-15, T-25, T-41 |
| RF-3.2 | T-15, T-25, T-41 |
| RF-3.3 | T-13, T-14, T-15, T-25, T-41 |
| RF-3.4 | T-07, T-14, T-15, T-25, T-27, T-46 |
| RF-3.5 | T-13, T-41 |
| RF-4.1 | T-09, T-12, T-13, T-41, T-45 |
| RF-4.2 | T-12, T-41 |
| RF-4.3 | T-07, T-14, T-25 |
| RF-4.4 | T-25, T-27 |
| RF-5.1 | T-08, T-16, T-25 |
| RF-5.2 | T-08, T-16, T-25 |
| RF-5.3 | T-08, T-16, T-25 |
| RF-5.4 | T-16, T-30, T-41 |
| RF-6.1 | T-03, T-38, T-39 |
| RF-6.2 | T-17, T-26, T-31, T-39 |
| RF-6.3 | T-39 |
| RF-6.4 | T-03, T-37, T-39 |
| RF-6.5 | T-17, T-39, T-45 |
| RF-6.6 | T-26 |
| RF-7.1 | T-18, T-40 |
| RF-7.2 | T-10, T-18, T-40, T-45 |
| RF-7.3 | T-29 |
| RF-7.4 | T-10, T-40 |
| RF-7.5 | T-18 |
| RF-8.1 | T-06, T-35 |
| RF-8.2 | T-06, T-27, T-35 |
| RF-9.1 | T-04, T-13, T-19 |
| RF-9.2 | T-04, T-30 |
| RF-9.3 | T-05, T-23, T-31, T-40 |
| RF-9.4 | T-05, T-40 |
| RF-9.5 | T-23, T-40, T-45 |
| RF-9.6 | T-20, T-40, T-45 |
| RF-10.1 | T-28, T-36 |
| RF-10.2 | T-36 |
| RF-10.3 | T-02, T-28, T-36 |
| RF-10.4 | T-36 |
| RF-11.1 | T-02, T-05, T-24, T-31, T-38 |
| RF-11.2 | T-34, T-38 |
| RF-11.3 | T-24, T-38 |
| RF-11.4 | T-24, T-38 |
| RF-12.1 | T-33, T-34, T-37 |
| RF-12.2 | T-02, T-37 |
| RF-12.3 | T-37 |
| RF-12.4 | T-46 |
| RF-13.1 | T-30, T-32, T-42 |
| RF-13.2 | T-32, T-34, T-42 |
| RF-13.3 | T-25, T-30, T-42 |
| RF-14.1 | T-02, T-28, T-37 |
| RF-14.2 | T-34, T-37 |
| RF-14.3 | T-37 |
| RF-14.4 | T-30, T-31, T-41 |
| RF-15.1 | T-21, T-41 |
| RF-15.2 | T-21, T-41 |
| RF-15.3 | T-21, T-41 |
| RNF-1 | T-11, T-43, T-45 |
| RNF-2 | T-12, T-27, T-46 |
| RNF-3 | T-36 |
| RNF-4 | T-33, T-37 |
| RNF-5 | T-37 |
| RNF-6 | T-28, T-44 |
| RNF-7 | T-03, T-26 |
| RNF-8 | T-24, T-38 |
| RNF-9 | T-29 |
| RNF-10 | T-32, T-42 |
| RNF-11 | T-07, T-27, T-46 |
| LC-1 | T-36 |
| LC-2 | T-36 |
| LC-3 | T-35 |
| LC-4 | T-39 |
| LC-5 | T-39 |
| LC-6 | T-39 |
| LC-7 | T-39 |
| LC-8 | T-38 |
| LC-9 | T-17, T-39 |
| LC-10 | T-38 |
| LC-11 | T-29 |
| LC-12 | T-40 |
| LC-13 | T-37 |
| LC-14 | T-22, T-35 |
| LC-15 | T-35 |
| LC-16 | T-38 |
| LC-17 | T-22, T-35 |
| LC-18 | T-38 |
| LC-19 | T-05, T-23, T-40 |
| LC-20 | T-05, T-40 |
| LC-21 | T-41 |
| LC-22 | T-20, T-40 |
| LC-23 | T-41 |
| LC-24 | T-09, T-12, T-41, T-45 |
| LC-25 | T-43 |
| LC-26 | T-23, T-43 |
| LC-27 | T-32, T-42 |
| LC-28 | T-42 |
| LC-29 | T-27 |
| LC-30 | T-36 |
| LC-31 | T-27, T-35 |
| US-1 … US-11 | T-35 … T-41 (journeys), T-43, T-45 |
| US-12 | T-38 (adversarial suite), T-36, T-37 |

Self-check: all 65 RF criteria, 11 RNF and 31 limit cases of `specPublicLanding.md` v0.1 appear above; every task names at least one requirement; every test name of plan §5.2 appears in at least one "Done when" line.
