# Plan 017 — Public landing

**Feature** The public, unauthenticated landing: what it explains, the two ways in, language, footer routes, and what it never does · **Plan version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — aligned with specification v0.1 (decisions Q1–Q8, derived rules D-a, D-b) · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Companion of** `specs/017-PublicLanding/specPublicLanding.md` v0.1 (the WHAT) · **Governed by** `constitution.md` v1.8 (principles P1–P16; feature F-IDN-19 as the subject; F-IDN-01, F-IDN-03, F-IDN-10 and F-IDN-14 at the boundary; F-X-01…F-X-07) · **Architecture source** Figma FE board 3 `46:7847` (feature-modular SPA, app-level composition, acyclic imports); screens R01 `148:341`, M01 `161:51034`.

This plan is the HOW. It adds no behaviour the specification does not state. The specification's eight decisions (Q1–Q8) and two derived rules (D-a, D-b) are built as decided. Where the specification depends on items in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`) (A-65, A-02, A-08; S-60, S-61, S-62, S-26, S-02) — and where this plan needs something outside P10's reach (A-66, raised by this plan on 28 October 2026) — this plan builds the stated assumption and names the item (§6). Requirement identifiers are the specification's (RF-x.y, RNF-n, LC-n); the FR identifiers of `requirements.md` are cited where a decision rests on them. Elements that another plan builds are tagged **[001]**-style.

---

## 1. Constitution compliance map

| Principle / feature | How this plan obeys it | Spec items |
|---|---|---|
| **P1** Acceptance ≠ approval ≠ Run | The landing states the separation (RF-3.1, RF-5.1) and has no path to any product command: it calls no endpoint at all (D2), so it cannot accept, approve or launch anything. | RF-3.1, RF-5.1, RF-2.4 |
| **P2** AI proposes, humans authorise | The AI statement is fixed product copy (RF-3.2); no AI service is called. | RF-3.2 |
| **P3** The browser expresses intent, never authority | The landing renders only copy built into the bundle; it reads no session, membership or tenant and derives nothing; its two actions are plain navigations whose target screens ask the server (plans 001, 002). | RF-1.4, RF-10, RNF-3 |
| **P6** Fail closed | No availability pre-check exists to fail open or closed (D2); a build whose operator addresses are absent or malformed fails (D7) rather than shipping a dead link; an unknown section address opens the top (D8). | RF-7.3, RF-9.4, RF-14 |
| **P8** No fabricated outcome | Every preview is rendered through one component that always carries the illustrative label (D11); sample identifiers carry the `sample:` mark; a content test refuses metrics, logos, testimonials and capabilities outside the baseline. | RF-4, RF-3.4, RNF-2, RNF-11 |
| **P10** Fixed technology stack | FE only: React 18, TypeScript, Vite, i18next, Tailwind. Prerendering uses React's own server renderer through Vite's native server build (D3) — no new dependency. TanStack Query, Zustand, axios and STOMP are **not used** by this feature (nothing to fetch, no draft, no stream). No BE change. Test-only tools are flagged in §5. | whole plan |
| **P11** Architecture styles | A new feature module `public-landing` composed by `app`; it imports only the exported route paths of `identity-access` **[001/002]** and generic `shared/*` adapters; no sibling-internal import; no back-end context touched. | §2 |
| **P12** Two tenancy gates | Not engaged: the landing reads no tenant row and builds no tenant context. | RF-1.4 |
| **P13** Configured values are never defaulted | The nine operator destinations (A-65; register §15) are build-time configuration validated by `PublicDocumentsConfig`; absent or malformed fails the build; no address is written in code or defaulted. | RF-7.3, RNF-9, LC-11 |
| **P14** Release bar | WCAG 2.2 AA with automated (axe) and manual assistive evidence in EN / ES / FR at three widths; interface language never changes executable text (the FAQ and examples are display copy only). | RNF-1, RF-6.6 |
| **F-X-01** C10 context | No C10 is built; the landing is outside `ProtectedBoundary` **[006]** and outside every session-aware provider. | RF-10.3 |
| **F-X-02** Request integrity | No request is issued by the landing; entry actions are navigations guarded against double activation (D12). | RF-2.7, LC-14 |
| **F-X-03** Secrets and safe problems | Nothing is written to cookies, localStorage, sessionStorage or IndexedDB; no address parameter is read, displayed or forwarded (D6). | RF-11, RF-12.2 |
| **F-X-04** Accessibility and languages | Skip link, ordered headings, focus to the target heading after in-page moves, menu focus trap, current language as text, `lang` attribute updated on switch. | RF-6.5, RF-9.5, RF-9.6, RNF-1 |
| **F-X-05** Audit log | Not engaged: the landing performs no security- or business-relevant act (RF-12.4). | RF-12.4 |
| **F-X-06** Configuration validation | `PublicDocumentsConfig` is part of the release validation, at build time (D7). | RF-7.3, RNF-9 |
| **F-X-07** Responsive surfaces | One `LandingPage` component reflows R01 → tablet → M01 by breakpoints; M01 is the drawn counterpart (menu open `429:4200`), tablet follows the D01 responsive policy (S-02). | RF-15 |

---

## 2. Module structure

### 2.1 Back end

**No back-end change.** The landing calls no endpoint, reads no record and opens no session (decisions Q1, Q4, Q5; D2). No bounded context, table, port, endpoint, configuration record or audit event is added. Static delivery of the built documents is outside the back end by assumption (A-66, §6).

### 2.2 Front end — feature module `src/features/public-landing` (new), `app/*`, `shared/*`

The FE section's feature list names no landing feature, so the module is a plan decision (D1). It follows the app-composed DAG: `app` → feature public APIs → generic `shared` adapters.

| Part | Element | Responsibility | Covers |
|---|---|---|---|
| Public API | `landingRoute` | The `/` route object; nothing else is exported from views. | RF-1.1 |
| | `LandingDocument` | The root element rendered at build time (D3) and hydrated in the browser; takes the validated `PublicDocuments` value and nothing else. | RF-1.1, RF-14.4 |
| Pure model | `sectionAnchors` | Stable in-page identifiers `workflow`, `capabilities`, `human-control`, `faq` (plus ids for every other section heading); one list used by header links, menu, "Explore the workflow" and the fragment resolver. | RF-9.1–9.4 |
| | `EntryActions` | The only two destinations, `CREATE_WORKSPACE → /sign-up` and `SIGN_IN → /sign-in`, imported as route paths from `identity-access` **[001/002]**; every entry control in the page is built from this list, so a third destination cannot be expressed. | RF-2.1–2.3, RF-2.5, D-a |
| | `BaselineCapabilityList` | The capability keys the copy may name (HTTP, Web, DB, macros, structured authoring, sources, approval, evidence, delivery, roles); the content test (D11) checks every capability card against it. | RF-3.4, RF-4.3, RNF-11 |
| | `landingContent` | The ordered section structure as i18n keys only — headings, statements, cards, FAQ questions and answers, illustrative preview fields; no literal text in components. | RF-1.2, RF-1.3, RF-3, RF-5, RNF-7 |
| | `resolveFragment(hash)` | Pure: known anchor → that section; anything else → top. Reads only the fragment, never the query. | RF-9.3, RF-9.4, RF-11.1 |
| Views | `LandingPage` | R01 at desktop, tablet reflow, M01 at phone widths by Tailwind breakpoints — one component; renders sections in the order of `landingContent`. | RF-1.2, RF-15.1–15.3 |
| | `LandingHeader` | Wordmark, the four in-page links, "Create workspace" and "Sign in" (S-60 assumption) at desktop and tablet; the "Menu ☰" control at phone width. | RF-2.2, RF-9.1 |
| | `LandingMenu` | M01 sheet: the four in-page links and the two entry actions only; focus trapped while open, Escape and choosing an entry close it, focus returns to the menu control. | RF-9.6, LC-22 |
| | `HeroSection` | Eyebrow, headline, lead, "Create workspace →", "Explore the workflow", the hero statement of RF-3.5 placed above the fold at every width, and the `IllustrativePreview` of the checkout scenario. | RF-2.2, RF-3.5, RF-4.1, RF-9.1 |
| | `IllustrativePreview` | The only component that may render a sample scenario, identifier, environment, count or outcome; always renders `IllustrativeLabel` first in reading order and as the preview's accessible name prefix; sample identifiers rendered with the `sample:` mark. | RF-4.1, RF-4.2, LC-24 |
| | `WorkflowSection`, `CapabilitiesSections`, `HumanControlSections`, `BeginSection` | Sections 02–09 of R01 as drawn, with the S-61 copy corrections (no mobile or fixture claim; HTTP, Web and DB examples). | RF-1.2, RF-3.1–3.4, RF-4.3 |
| | `FaqSection` | The three questions and answers as visible text (no disclosure widget). | RF-5.1–5.4 |
| | `ClosingCta` | "Give every run a reason.", "Create workspace →", "Sign in". | RF-2.2 |
| | `LandingFooter` | Wordmark, `PublicDocumentLinks`, `LanguageSwitch`, copyright line. | RF-7.1, RF-7.5, RF-6.2 |
| | `LanguageSwitch` | Three choices EN / ES / FR as a labelled group; the current one marked as selected in text and in its accessible state; switching keeps scroll position and focus. | RF-6.2, RF-6.5, LC-9 |
| | `PublicDocumentLinks` | Privacy, Terms, Contact as `ExternalLink`s to the three validated addresses. | RF-7.1, RF-7.2, RF-7.4 |
| Interaction | `useEntryNavigation()` | Navigates to an `EntryActions` destination once; ignores further activations until the navigation completes; passes nothing but the in-memory language (D4, D12). | RF-2.6, RF-2.7, RF-6.3 |
| | `useSectionFocus()` | After a fragment change (link, menu, initial load), scrolls to the section (instant when reduced motion is preferred) and moves focus to its heading. | RF-9.3, RF-9.5, LC-19, LC-26 |
| | `useStripQuery()` | After hydration, replaces the address with the same path and fragment and no query, without a new history entry (D6). | RF-11.1, RF-11.4 |
| `app` | `app/entries/landing` | Build-time entry: renders `LandingDocument` in EN to a complete HTML document `landing.html` with title, description, `robots: index`, CSP and referrer meta, preload of self-hosted fonts, and the hydration script reference (D3, D9, D10). | RF-1.1, RF-13.1, RF-13.3, RF-14.4, RNF-6 |
| | `app/entries/shell` | The SPA shell document `app.html` for every other route, with `robots: noindex` and the same CSP and referrer meta (D10). | RF-13.2 |
| | `app/routes` | Registers `landingRoute` at `/` **outside** the session provider and `ProtectedBoundary` **[006]**; lazy-loads the `identity-access` chunks so the landing's bundle holds only its own code (D4). | RF-10.3, RF-14.1, RNF-6 |
| | `robots.txt` (static asset) | Allows `/` only; disallows every other path. | RF-13.1, RF-13.2, LC-27 |
| Build | `PublicDocumentsConfig` | Reads the nine operator destinations from the release configuration (A-65; register §15); each must be present, absolute, `https`, not the product's own origin; any violation fails the build with the name of the missing value; the validated value is passed to `LandingDocument` (D7). | RF-7.3, RNF-9, LC-11 |
| | `LandingBuildChecks` | Post-build assertions over `landing.html` and its assets: no external origin in any URL, no inline script, CSP and robots meta present, the whole content present without running scripts (§5). | RF-12.1, RF-14.4, RNF-4 |
| Shared used | `shared/i18n` | Extended: the i18next instance is created with `lng: 'en'` and `fallbackLng: 'en'`, **no language detector and no cache**; the `public-landing.{en,es,fr}.json` bundles; ES and FR loaded on demand as same-origin static chunks; `document.documentElement.lang` follows the current language (D5). Shared with `identity-access`, so the language chosen on the landing is the one sign-in and signup render (RF-6.3). | RF-6.1–6.4, RF-6.6, RNF-7, LC-4–LC-8 |
| | `shared/ui` | New: `IllustrativeLabel`, `ExternalLink` (visible "opens outside A·Spectra" cue and accessible description, `rel="noopener noreferrer"`, `referrerpolicy="no-referrer"`), `SkipLink`. Existing: `Button`, 44 px targets at touch widths **[001]**. | RF-4.1, RF-7.2, RNF-1 |
| | `shared/assets` | Self-hosted font files and every image the landing uses, bundled with hashed names (D9). | RF-12.1 |

**Not used by this feature:** `shared/api`, `shared/errors`, TanStack Query, Zustand, STOMP, `shared/security` scope epoch, `ProtectedBoundary` **[006]**. A lint rule on the module forbids importing them (D2).

---

## 3. Behaviour-to-module trace

| Spec item | Build / document | Front end |
|---|---|---|
| RF-1.1 | `landing.html` served at `/` (A-66) | `landingRoute`, `LandingDocument` |
| RF-1.2 | — | `landingContent` order, `LandingPage` |
| RF-1.3 | — | `landingContent` keys with S-61 corrections |
| RF-1.4 | No endpoint exists for the landing | Module lint: no `shared/api`, no query client |
| RF-2.1 | — | `EntryActions` (two values) |
| RF-2.2 | — | `LandingHeader`, `LandingMenu`, `HeroSection`, `ClosingCta` |
| RF-2.3 | — | `EntryActions.CREATE_WORKSPACE → /sign-up` **[002]** |
| RF-2.4 | No endpoint; signup screen owns creation **[002]**, workspace creation **[012]** | — |
| RF-2.5 | — | `EntryActions.SIGN_IN → /sign-in` **[001]** |
| RF-2.6 | — | `useEntryNavigation` (no query, no state but language) |
| RF-2.7 | — | `useEntryNavigation` guard |
| RF-3.1–3.3 | — | `WorkflowSection`, `HumanControlSections`, `BeginSection` copy keys |
| RF-3.4 | `LandingBuildChecks` content scan | `BaselineCapabilityList` |
| RF-3.5 | — | `HeroSection` statement above the fold |
| RF-4.1 | — | `IllustrativePreview` + `IllustrativeLabel` |
| RF-4.2 | — | `IllustrativePreview` `sample:` mark |
| RF-4.3 | — | `BaselineCapabilityList`, example keys (HTTP, Web, DB) |
| RF-4.4 | `LandingBuildChecks` forbidden-content scan | — |
| RF-5.1–5.3 | — | `FaqSection` keys |
| RF-5.4 | Present in prerendered HTML | `FaqSection` (visible text, no disclosure) |
| RF-6.1 | `landing.html` rendered in EN | `shared/i18n` `lng: 'en'`, no detector |
| RF-6.2 | ES / FR bundles as static chunks | `LanguageSwitch` |
| RF-6.3 | — | Shared i18next instance; SPA navigation (D4) |
| RF-6.4 | — | No detector, no cache, no storage (D5) |
| RF-6.5 | — | `LanguageSwitch` selected state as text |
| RF-6.6 | — | Display keys only; no executable text in bundles |
| RF-7.1 | — | `LandingFooter`, `PublicDocumentLinks` |
| RF-7.2 | — | `ExternalLink` cue |
| RF-7.3 | `PublicDocumentsConfig` fails the build | — |
| RF-7.4 | Referrer meta (D6) | `ExternalLink` `noreferrer` |
| RF-7.5 | — | `LandingFooter` |
| RF-8.1, RF-8.2 | `LandingBuildChecks` forbidden-entry scan | `EntryActions` has no federation or passkey value |
| RF-9.1, RF-9.2 | Anchor ids in prerendered HTML | `sectionAnchors`, `LandingHeader` |
| RF-9.3, RF-9.4 | — | `resolveFragment`, `useSectionFocus` |
| RF-9.5 | — | `useSectionFocus` |
| RF-9.6 | — | `LandingMenu` |
| RF-10.1, RF-10.2 | One static document for all visitors | No identity-aware import |
| RF-10.3 | — | `app/routes`: landing outside session provider and `ProtectedBoundary` **[006]** |
| RF-10.4 | — | Target screens' S-26 behaviour **[001/002]** |
| RF-11.1–11.4 | Same document for any query (A-66) | `resolveFragment` reads hash only; `useStripQuery`; fixed hrefs |
| RF-12.1 | CSP `default-src 'self'` (meta + A-66 header); `LandingBuildChecks` | `shared/assets` self-hosted |
| RF-12.2 | — | No storage API used (lint) |
| RF-12.3 | — | No consent component exists |
| RF-12.4 | Delivery-tier logs only (A-66) | — |
| RF-13.1–13.3 | `landing.html` robots index + title/description; `app.html` noindex; `robots.txt` | `public-landing` meta keys (EN) |
| RF-14.1 | No request | Module lint |
| RF-14.2, RF-14.3 | Static delivery independent of the BE (A-66) | Entry actions always enabled |
| RF-14.4 | Whole content and plain `href`s in `landing.html` | Hydration adds switch, menu, focus |
| RF-15.1–15.3 | — | `LandingPage` breakpoints; `LandingMenu` |

---

## 4. Technical decisions

**D1 · A new feature module `public-landing`, separate from `identity-access`.**
*Why.* The landing holds no identity behaviour: it calls nothing and knows no session (Q1, Q4). Keeping it apart lets the module forbid `shared/api`, the query client and the session provider by rule (D2), and keeps its bundle free of sign-in code (RNF-6). It depends on `identity-access` only through the two exported route paths, which is what P11's DAG allows.
*Discarded.* Placing it inside `identity-access` — couples a public, cacheable page to session-aware code and makes the "no request" rule unenforceable by import boundaries. A standalone marketing site in another toolchain — leaves P10.

**D2 · No back end and no request of any kind.**
*Why.* RF-1.4, RF-14.1 and Q4: the landing must render when every service is down and must not depend on any record. All copy is in the bundle; nothing is fetched, so no failure can reach it and no session can be touched (RF-10.3). A module lint forbids `shared/api`, TanStack Query, `shared/security`, STOMP and every browser storage API inside `public-landing`.
*Discarded.* Content served by a BE endpoint or a CMS — adds a dependency that can fail and a place where tenant data could leak. An availability pre-check — rejected by Q4.

**D3 · Prerender `/` at build time with React's own server renderer through Vite's native server build, then hydrate.**
*Why.* RF-14.4 (readable with scripts disabled), RF-13 (indexable content), RNF-6 (statement and actions usable within three seconds on a slow phone) and RNF-3 (byte-identical for everyone) all point to a complete static HTML document. React 18 ships its server renderer and Vite has a native server build, so the prerender is one build step and adds no dependency (P10). Because the content is static per build, there is no run-time rendering at all.
*Discarded.* Client-only rendering — the page is empty without scripts, slow on a slow phone and weakly indexable. A third-party prerender plugin — a new dependency outside P10. Run-time server rendering by the Spring Boot application — couples availability to the back end (Q4) and puts the session filter chain in front of `/` (Q1).

**D4 · The landing is a route of the same SPA; sign-in and signup chunks load lazily.**
*Why.* RF-6.3 requires the chosen language to carry into sign-in and signup without storing it (Q2, Q5) and without passing anything in the address (Q7). After hydration, an entry action is an in-app navigation, so the shared i18next instance keeps the language in memory. Lazy chunks keep the landing's own payload small (RNF-6). Without scripts the actions are plain links and the target opens in EN, which RF-14.4 accepts.
*Discarded.* A separate static site — a full document load would lose the language. A language query parameter or a cookie — forbidden by Q7 and Q5.

**D5 · The i18n instance starts in EN, with no language detector and no cache.**
*Why.* Q2 says EN on every visit whatever the browser prefers, and the choice is never remembered. Q5 says nothing is stored. Starting with a fixed `lng` and omitting the detector does exactly that. The language lives only in the memory of the open document; reloading or leaving ends it (RF-6.4). The `lang` attribute follows the switch for assistive technology (RNF-1).
*Discarded.* The i18next browser language detector — a new dependency, and it reads and writes storage and cookies. Reading `navigator.language` — contradicts Q2. A per-language URL — that was the rejected Q2 option.

**D6 · The address is never read except for its fragment; the query is stripped; no referrer is sent.**
*Why.* Q7 and RF-11: ignoring parameters is only safe if nothing reads them. `resolveFragment` reads the hash alone. Every link is a fixed `href` built from `EntryActions` or the validated addresses. After hydration `useStripQuery` removes any query from the visible address without adding a history entry, so a token or email is not copied into a later share. `Referrer-Policy: no-referrer` (meta, and a header under A-66) plus `noreferrer` on external links keep the original address from reaching the operator documents (RF-7.4).
*Discarded.* Validating a return target against an allow-list — the rejected Q7 option. Leaving the query in place — harmless to the page but leaks into referrers and shares.

**D7 · Operator addresses are build-time configuration, validated before release.**
*Why.* P13 and Q3: never defaulted, and a missing or malformed value blocks release (RF-7.3). Injecting the nine destinations into the prerender makes the build the release gate. A run-time fetch would break RF-14.1, and a value in code would be a default. Validation requires presence, an absolute `https` address and an origin other than the product's. Register `122:2` §15 (1 November 2026) closed A-65's question: one explicit destination per purpose and language, no fallback; a shared destination is recorded explicitly.
*Discarded.* Fetching addresses from the back end at run time — breaks the no-request rule. Hard-coded addresses — a default (P13). Hiding a link whose address is missing — the rejected Q3 option.

**D8 · In-page navigation uses native fragment anchors, with focus to the heading.**
*Why.* Q6 and RF-9: stable addresses that work without scripts and survive sharing (LC-19). Native anchors need no router state. `useSectionFocus` adds the focus move WCAG needs (RF-9.5), and an unknown fragment simply opens the top (RF-9.4).
*Discarded.* Router routes per section (`/faq`) — creates indexable pseudo-pages (against Q8) and needs scripts. Scroll-only handlers without anchors — no shareable address.

**D9 · No third-party resource: fonts and images self-hosted, strict CSP in the document and at the delivery tier.**
*Why.* Q5 and RF-12.1. Every font and image is a hashed asset of the bundle. The prerendered document carries a CSP limiting every resource to the same origin with no inline script. Vite emits module scripts as files, so no `unsafe-inline` is needed. The same policy as a response header is the A-66 assumption; the meta element keeps the rule if the header is missing.
*Discarded.* A font CDN — a third-party request on every visit. A CSP header only — unenforced wherever the delivery configuration drifts.

**D10 · Indexing is set per document: `landing.html` index, `app.html` noindex, plus `robots.txt`.**
*Why.* Q8 and RF-13. Only `/` returns the landing document; every other route returns the shell, which carries `noindex`, so sign-in, signup, verification, reset, recovery, invitation and every authenticated route are excluded together without per-route work. `robots.txt` allows `/` only, and `X-Robots-Tag` is the A-66 header backstop. The title and description are EN keys of `public-landing`, used for search results and link previews (RF-13.3). No preview image shows a sample as a result.
*Discarded.* Per-route meta set by scripts — crawlers that don't run scripts would miss it. No `robots.txt` — leaves crawler budget on the shell.

**D11 · Content discipline is enforced by structure and by a content test.**
*Why.* FR-IDN-142 and RF-4, RNF-2, RNF-11 (S-61). Samples can only be rendered through `IllustrativePreview`, which always prints the label, so an unlabelled preview cannot be written. A content test walks the EN, ES and FR bundles:
- Every capability card's key is in `BaselineCapabilityList`.
- Examples cover HTTP, Web and DB only.
- No bundle contains a percentage, metric, customer name, logo reference or testimonial key.
- Every language bundle has the same key set (RNF-7).
When a capability leaves the baseline, removing it from `BaselineCapabilityList` fails the build until the copy is removed (LC-29).
*Discarded.* Editorial review alone — drifts between releases and languages.

**D12 · An entry action navigates once.**
*Why.* RF-2.7, LC-14: a double activation must open one screen. `useEntryNavigation` ignores activations while a navigation is in progress. With scripts disabled the browser's own link behaviour applies, and repeated activation still opens the same single destination.
*Discarded.* Disabling the control visually — a disabled entry action contradicts RF-14.2.

---

## 5. Test strategy

### 5.1 Levels and tools

| Level | Tool (constitution status) | What it proves |
|---|---|---|
| FE unit | Vitest + Testing Library (**test-only**, Vite toolchain) | `resolveFragment`, `EntryActions` (two values), `useEntryNavigation` guard, `useStripQuery`, `LanguageSwitch` states, `LandingMenu` focus trap, `IllustrativePreview` label order, i18n start state. |
| Content | Vitest over the three bundles (**test-only**) | Key parity across EN / ES / FR, capability keys against `BaselineCapabilityList`, forbidden content (metrics, logos, testimonials, federation or passkey wording), FAQ answers present. |
| Build | Vitest over the built output (**test-only**) | `landing.html` complete without scripts; robots, title, description, CSP and referrer meta; no external origin in any URL; no inline script; `app.html` noindex; `robots.txt`; `PublicDocumentsConfig` refusals. |
| FE e2e | Playwright for the browser (**test-only**) with request interception | Zero requests to the product API and to any third-party origin; zero cookies and zero storage entries after a visit; signed-in vs anonymous document identical; parameters ignored and stripped; language carry-over; scripts-disabled run; identity services stubbed down. |
| Accessibility | axe in Playwright (the evidence named in F-X-04, **test-only**) | Automated WCAG 2.2 AA at 1440, 768 and 390 CSS px in EN, ES and FR, plus 320 px reflow and 400 % zoom. |
| Performance | Playwright with throttled CPU and network (**test-only**) | Hero statement, illustrative label and both entry actions readable and usable within three seconds on the reference slow-phone profile; no layout shift after first render. |
| Manual assistive | NVDA (Chrome, Firefox, Windows), VoiceOver (Safari, macOS and iOS) — versions pinned in the release-support matrix | Reading order, headings, skip link, preview announced as illustrative, menu focus, current language announced, external-link cue, focus after in-page moves. |

Playwright Java and Cucumber belong to Core and are not used for the FE. No BE test level applies (§2.1).

### 5.2 Test matrix

| Spec items | Level | Test | Proves criterion (spec §7) |
|---|---|---|---|
| RF-1.1 | e2e | `landing.opensWithoutSession` | 1, 2 |
| RF-1.2, RF-1.3 | Content, unit | `content.sectionsInDesignedOrder`; `content.s61CorrectionsApplied` | 3 |
| RF-1.4, RF-14.1, RNF-5 | e2e | `network.noProductRequestOnOpenOrUse` | 9 |
| RF-2.1, RF-2.2, D-a | Unit, e2e | `entryActions.exactlyTwoDestinations`; `entryActions.reachableFromTopAtEveryWidth` | 4 |
| RF-2.3, RF-2.4, LC-3 | e2e | `createWorkspace.opensSignupAndCreatesNothing` | 4 |
| RF-2.5, LC-2 | e2e | `signIn.opensSignInScreen` | 4 |
| RF-2.6, LC-17 | e2e | `entryActions.passOnlyLanguage` | 4, 6 |
| RF-2.7, LC-14 | Unit, e2e | `entryNavigation.doubleActivationOpensOnce` | 4 |
| RF-3.1–RF-3.3, RF-3.5 | Content, e2e | `content.boundaryStatementsPresent`; `hero.statementAboveFoldAllWidths` | 3 |
| RF-3.4, RNF-11, LC-29 | Content | `content.capabilitiesWithinBaseline`; `content.removedCapabilityFailsBuild` | 3 |
| RF-4.1, RF-4.2, LC-24 | Unit, e2e, manual | `preview.alwaysLabelledIllustrative`; `preview.sampleIdentifiersMarked`; screen-reader check | 3, 12 |
| RF-4.3 | Content | `content.examplesHttpWebDbOnly` | 3 |
| RF-4.4, RNF-2 | Content | `content.noMetricsLogosTestimonials` | 3 |
| RF-5.1–RF-5.4 | Content, build | `faq.threeAnswersPresent`; `faq.visibleWithoutScripts` | 3 |
| RF-6.1, LC-4, LC-5, LC-7, LC-8 | Unit, e2e | `i18n.startsInEnglishWhateverBrowserPrefers`; `i18n.queryLangIgnored`; `i18n.notRememberedAfterReload` | 7 |
| RF-6.2, LC-9 | e2e | `languageSwitch.keepsPosition` | 7 |
| RF-6.3, LC-6 | e2e | `languageSwitch.carriesIntoSignInAndSignup` | 7 |
| RF-6.4, RF-12.2 | e2e | `storage.noCookieNoStorageAfterVisit` | 7, 8 |
| RF-6.5 | Unit, manual | `languageSwitch.currentShownAsText` | 7, 12 |
| RF-6.6, RNF-7 | Content | `content.bundleKeyParity` | 3 |
| RF-7.1, RF-7.2, RF-7.4 | Unit, e2e | `footer.routesExternalWithCue`; `footer.noReferrerSent` | 10 |
| RF-7.3, RNF-9, LC-11 | Build | `publicDocuments.missingFailsBuild`; `publicDocuments.malformedFailsBuild` | 10 |
| RF-7.5 | Unit | `footer.copyrightLine` | 3 |
| LC-12 | e2e | `footer.unreachableDocumentLeavesLandingIntact` | 10 |
| RF-8.1, RF-8.2, LC-31 | Content, unit | `entry.noFederationOrPasskey` | 4 |
| RF-9.1, RF-9.2 | Build, unit | `anchors.stableIdsInPrerender` | 6 |
| RF-9.3, RF-9.5, LC-19 | e2e | `fragment.knownSectionOpensWithFocus` | 6 |
| RF-9.4, LC-20 | Unit, e2e | `fragment.unknownOpensTop` | 6 |
| RF-9.6, LC-22 | Unit, e2e | `menu.twoActionsFourLinksFocusTrapped` | 12 |
| RF-10.1, RF-10.2, RNF-3, LC-1, LC-30 | e2e | `identity.signedInAndAnonymousDocumentsIdentical` | 5 |
| RF-10.3 | e2e | `session.notReadRefreshedOrEnded` (no refresh call, cookie unchanged) | 5 |
| RF-10.4 | e2e | `signedIn.entryActionsFollowS26` (stubbed targets **[001/002]**) | 5 |
| RF-11.1, RF-11.2, LC-10 | e2e | `params.redirectIgnored` | 6 |
| RF-11.3, LC-16 | e2e | `params.invitationTokenIgnoredAndStripped` | 6 |
| RF-11.4, RNF-8, LC-18 | e2e | `params.markupNeverRendered`; `params.queryStripped` | 6 |
| RF-12.1, RNF-4 | Build, e2e | `build.noExternalOrigin`; `network.noThirdPartyRequest` | 8 |
| RF-12.3 | Unit | `consent.noBannerExists` | 8 |
| RF-12.4 | Inspection | No landing code path records a visit (A-66 log scope) | 8 |
| RF-13.1–RF-13.3, RNF-10, LC-27, LC-28 | Build | `robots.landingIndexOnly`; `robots.shellNoindex`; `robots.txtAllowsRootOnly`; `meta.titleAndDescription` | 11 |
| RF-14.2, RF-14.3, LC-13 | e2e | `outage.landingRendersActionsEnabled` (all services stubbed down) | 9 |
| RF-14.4, LC-21 | Build, e2e | `noScript.contentActionsFooterWork` | 9 |
| LC-15 | e2e | `back.fromSignupShowsLandingUnchanged` | 4 |
| RF-15.1–RF-15.3, LC-23 | e2e | `responsive.sameStatementsAllWidths`; `responsive.tabletReflow` | 1, 12 |
| RNF-1, LC-25, LC-26 | Accessibility, manual | `a11y.axeAllWidthsAllLanguages`; `a11y.reflow320AndZoom400`; `a11y.reducedMotion`; manual NVDA / VoiceOver run | 12 |
| RNF-6 | Performance | `perf.heroUsableWithinThreeSecondsSlowPhone`; `perf.noLayoutShift` | 12 |

### 5.3 Not tested here
Sign-in, the challenge and S-26 behaviour (plan 001); signup, verification and S-26 behaviour (plan 002); workspace creation (plan 012); invitations (plan 008); sessions (plan 006). Their routes are stubbed in landing e2e tests and only the hand-off is asserted. The content of the operator documents and the delivery tier's headers (A-65, A-66) are verified by their owners. The landing's own meta fallbacks are tested here.

### 5.4 Exit criteria
Specification §7 items 1–12 map to the "Proves criterion" column. Item 13 is satisfied by §6 of this plan being resolved or deferred by the product owner.

---

## 6. Register dependencies

| Item | Assumption built by this plan | If it resolves otherwise |
|---|---|---|
| A-65 operator document addresses | Nine destinations (three purposes × EN / ES / FR) in the release configuration, validated by `PublicDocumentsConfig` at build time, injected into the prerender; each explicit, none defaulted (register §15). A change takes a new release. | If the operator record must change without a release, the delivery tier rewrites the links at serve time or serves a small same-origin static file read at build-equivalent time. No run-time call to the back end is introduced either way (D2). |
| A-66 static delivery and headers (raised by this plan, 28 October 2026) | A static tier independent of the back end serves `landing.html` at `/` only, `app.html` for every other route, hashed assets and `robots.txt`. It sets CSP same-origin, `Referrer-Policy: no-referrer`, `X-Robots-Tag: noindex` except on `/` and `robots.txt`, no `Set-Cookie`, long-lived caching for hashed assets, revalidation for documents. | If the Spring Boot application serves the front end, `/` must be excluded from the session filter chain and from any cookie-setting filter, and the header set moves to its configuration. D9 and D10 keep the in-document fallbacks unchanged. |
| A-02 fixtures, A-08 mobile execution | Neither is named on the landing (S-61 assumption), and `BaselineCapabilityList` excludes both. | When either enters the baseline, add its key to the list and its copy to the bundles. |
| S-60 header "Sign in", M01 menu | "Sign in" in the header beside "Create workspace", and the menu sheet as specified in RF-9.6. | Move the control to the drawn position; `EntryActions` is unchanged. |
| S-61 copy corrections | Copy keys follow the corrections: no mobile or fixture claim, HTTP, Web and DB examples, R01 and M01 statements aligned, every preview labelled. | Replace the keys with the drawn copy; the content test still enforces the rules. |
| S-62 footer states, ES / FR copy, preview text | Current language as selected text, external cue as text plus accessible description, interim ES / FR copy produced from the EN keys for review, title and description written as EN keys. | Replace with the drawn states and authored copy. |
| S-26 already signed in | The landing does not change (Q1). The target screens apply plans 001 and 002. | No change here. |
| S-02 responsive coverage | The tablet width reflows R01 under the desktop rules. | Add the drawn tablet layout as a breakpoint of `LandingPage`. |

**Cross-plan item (applied 2 November 2026).** Plan 002 now renders Privacy and Terms on R04 and N01 as the same operator documents outside the product (plan 002 §6), through this plan's `PublicDocumentLinks` from `shared/ui` with the same validated addresses; specifications 004, 005, 008 and 012 state the same rule (product owner's decision, 2 November 2026).

---

## 7. Requirements coverage check

| Spec item | §2 element | §4 decision | §5 test |
|---|---|---|---|
| RF-1 (1.1–1.4) | `landingRoute`, `LandingDocument`, `landingContent`, `LandingPage`, module lint | D1, D2, D3 | RF-1 rows |
| RF-2 (2.1–2.7) | `EntryActions`, `useEntryNavigation`, header, menu, hero, closing | D4, D12 | RF-2 rows |
| RF-3 (3.1–3.5) | Section components, `BaselineCapabilityList`, `HeroSection` | D11 | RF-3 rows |
| RF-4 (4.1–4.4) | `IllustrativePreview`, `IllustrativeLabel`, `BaselineCapabilityList` | D11 | RF-4 rows |
| RF-5 (5.1–5.4) | `FaqSection` | D3 | RF-5 rows |
| RF-6 (6.1–6.6) | `shared/i18n`, `LanguageSwitch` | D4, D5 | RF-6 rows |
| RF-7 (7.1–7.5) | `LandingFooter`, `PublicDocumentLinks`, `ExternalLink`, `PublicDocumentsConfig` | D6, D7 | RF-7 rows |
| RF-8 (8.1–8.2) | `EntryActions`, content test | D11 | RF-8 row |
| RF-9 (9.1–9.6) | `sectionAnchors`, `resolveFragment`, `useSectionFocus`, `LandingMenu` | D8 | RF-9 rows |
| RF-10 (10.1–10.4) | `app/routes` outside the session provider; one static document | D2, D3 | RF-10 rows |
| RF-11 (11.1–11.4) | `resolveFragment`, `useStripQuery`, fixed hrefs | D6 | RF-11 rows |
| RF-12 (12.1–12.4) | `shared/assets`, CSP meta, module lint | D2, D5, D9 | RF-12 rows |
| RF-13 (13.1–13.3) | `landing.html`, `app.html`, `robots.txt`, meta keys | D10 | RF-13 rows |
| RF-14 (14.1–14.4) | Prerendered document, no request | D2, D3 | RF-14 rows |
| RF-15 (15.1–15.3) | `LandingPage` breakpoints, `LandingMenu` | D1 | RF-15 rows |
| RNF-1 | `SkipLink`, focus hooks, `LanguageSwitch`, `lang` attribute | D5, D8 | a11y rows |
| RNF-2 | `IllustrativePreview` | D11 | RF-4.4 row |
| RNF-3 | One static document | D2, D3 | RF-10 row |
| RNF-4 | Self-hosted assets, CSP | D9 | RF-12.1 row |
| RNF-5 | No request; static delivery (A-66) | D2 | RF-1.4 row |
| RNF-6 | Prerender, lazy chunks, preloaded fonts | D3, D4 | RNF-6 row |
| RNF-7 | Bundle parity | D11 | RF-6.6 row |
| RNF-8 | `resolveFragment`, `useStripQuery` | D6 | RF-11.4 row |
| RNF-9 | `PublicDocumentsConfig` | D7 | RF-7.3 row |
| RNF-10 | Robots per document, `robots.txt` | D10 | RF-13 row |
| RNF-11 | `BaselineCapabilityList` | D11 | RF-3.4 row |
| LC-1 … LC-31 | As traced in §5.2: LC-1, 30 (identity); LC-2, 3, 14, 15, 17, 31 (entry); LC-4 … 9 (language); LC-10, 16, 18 (parameters); LC-11, 12 (footer); LC-13, 21 (availability); LC-19, 20 (fragments); LC-22 … 26 (responsive and accessibility); LC-27, 28 (indexing); LC-29 (content) | — | §5.2 |
| Q1 … Q8, D-a, D-b | Q1 → RF-10; Q2 → D5; Q3 → D7; Q4 → D2; Q5 → D9; Q6 → D8; Q7 → D6; Q8 → D10; D-a → `EntryActions`; D-b → RF-8 content test | D2 … D10 | §5.2 |

Every RF, RNF and LC of the specification appears in §3 or §5.2. No requirement is covered by assumption alone: those that depend on a register item (A-65, A-66, S-60 … S-62) are built on the stated assumption and tested against it.
