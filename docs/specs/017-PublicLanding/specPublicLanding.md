# Specification 017 — Public landing

**Feature** The public, unauthenticated landing: what it explains, the two ways in, language, footer routes, and what it never does · **Version** 0.1 · **Date** 28 October 2026 · **Status** Draft for review — open doubts closed 28 October 2026 · **Revision** 2 November 2026 — aligned to the closed designs (D03 ledger) and constitution v1.8 (QA round)

**Sources of truth, in order:** `constitution.md` (feature F-IDN-19 as the subject; F-IDN-01, F-IDN-03, F-IDN-14 and F-IDN-10 where a criterion touches their boundary; principles P1, P2, P3, P6, P8, P13, P14) → `requirements.md` (FR-IDN-140, 141, 142, 143, 144, 145; FR-X-042, FR-X-043, FR-X-050) → `scope.md` (product experience: a public landing surface with exactly two entry actions; federation and phishing-resistant factors as extension capabilities outside the MVP baseline) → the Figma screens R01 LandingPage `148:341` and M01 MobileLandingPage `161:51034` → the Figma decision records (register `122:2` §14 and §15, D03 Screen decisions ledger `148:4`).

**Related specifications:** 001 Login (the sign-in screen "Sign in" opens; S-26 for a visitor already signed in) · 002 Signup and email verification (the signup screen "Create workspace" opens; S-26) · 012 Workspace creation and bootstrap (reached only after signup, verification and sign-in, never from the landing directly) · 008 Invitation acceptance (an invitation keeps its own link and never passes through the landing) · 005 Controlled account recovery and 004 Password reset (reached from the sign-in screen, not from the landing) · 006 Sessions, refresh and expiry and 007 Logout and session management (a session, if any, is neither read nor changed by the landing).

This document states **what** the public landing does and **why**. It contains no technology, architecture or implementation decision.

---

## 1. Context and objective

### 1.1 Context

ASpectra turns requirements into reviewable test scenarios and keeps intent, human decisions and evidence on one traceable line. Its central promise is a boundary: acceptance of a mapping, approval of a bundle and the launch of a Run are three separate human decisions; AI only proposes; the managed runner starts only when a person decides. The **public landing** is the one surface every visitor can open without an identity. It is where an evaluator first learns what the product does, and where a returning person finds the way in.

R01 (desktop) and M01 (mobile) draw it as ten sections — hero with an illustrative scenario preview, the deliberate workflow, starting from what you have, one catalog with layers, suggestions are not facts, exact context and explicit consent, evidence without interpretive blur, clear boundaries for teams, beginning without the integration tax, and questions answered — followed by a closing call to action and a footer with Privacy · Terms · Contact and EN / ES / FR. The landing offers exactly two entry actions, **create workspace** and **sign in**. "Create workspace" never creates anything: it opens verified-email signup (specification 002), and workspace creation (specification 012) follows only after the email is verified and the person has signed in.

### 1.2 Objective

Give every visitor an honest, public description of the product and exactly two ways in, while:

- stating the product boundary in plain words — acceptance, approval and Run are separate; AI and connectors are optional; a verified email is required; the runner starts only by a user decision — and answering the three questions a cautious evaluator asks first;
- labelling every sample on the page as illustrative, and never presenting an identifier, count, scenario or outcome as a real result, a benchmark or a coverage claim;
- being the same page for everyone: it reads no session, no identity and no tenant, shows nothing that depends on who is looking, and takes nothing from the address it was opened with;
- respecting the visitor: no tracking, no third-party resources at run time, nothing stored on the device;
- always being available, whatever the state of the identity services, and leaving every failure to the screen that owns it.

### 1.3 Why it matters

A landing that overstates is the first broken promise: a page that implies autonomy, shows a sample count as if it were coverage or names a capability the product does not ship tells the evaluator that the rest of the product may overstate too — the opposite of what ASpectra sells. A landing that knows who is looking is a disclosure surface: a cached page showing a workspace name, a "welcome back", or an invitation token carried in its address leaks to the next person on a shared machine. A landing that trusts its address is an attack surface: a redirect parameter forwarded to sign-in is an open redirect and a phishing aid. And a landing that collapses when sign-in is down turns an outage into a blank page with no explanation. Keeping it static, identical for everyone and strictly worded removes all four.

### 1.4 Decisions taken during specification

The design left these points open; the product owner decided them on 28 October 2026. The body refers to them as *(Decision: Qn)*. Items that depend on an architecture or design decision are not listed here: they are recorded in the Figma decision records (`122:2` §14 and the D03 ledger `148:4`), each with the assumption this specification builds until it is resolved, and the body cites them by identifier. Written on 28 October 2026: architecture register `122:2` §14 **A-65** (with its operator part in `122:2` §15); D03 Screen decisions ledger `148:4` **S-60**, **S-61**, **S-62**; notes appended to **S-26** and **S-02**.

| Q | Question | Decision |
|---|---|---|
| Q1 | What a visitor who is already signed in sees | The same landing as every other visitor. Nothing on it depends on a session: no name, workspace, avatar, "continue" action or greeting. "Sign in" and "Create workspace" open the sign-in and signup screens, which then apply the rules of specifications 001 and 002 for an identity that is already signed in (S-26). The landing neither reads, refreshes nor ends a session. |
| Q2 | Which interface language the landing opens in, and whether a choice carries over | The landing always opens in **EN**, on every visit, whatever the browser prefers. The visitor may switch to ES or FR at any time; the chosen language carries into the public entry screen the visitor goes to from the landing (sign-in or signup) during that navigation. The choice is not remembered between visits and is never written to an account. |
| Q3 | Privacy · Terms · Contact | They are shown and open documents the platform operator publishes **outside** the product. Their content is out of scope. Their addresses are operator-published configuration (A-65): never defaulted or invented, and a missing or malformed address blocks release rather than rendering a dead or placeholder link (P13). |
| Q4 | Behaviour when identity services are degraded or unavailable | The landing is static: it loads no live data and never checks availability. It always renders, and both entry actions always route. Any failure is reported on the screen the visitor reaches, by that screen's own states (specifications 001 and 002). The landing shows no status banner and never hides or disables an action because of an outage. |
| Q5 | Visitor measurement and tracking | None. No analytics, no third-party script, font or other resource loaded at run time, no cookie and no device storage set by the landing. No consent banner is needed and none is shown. |
| Q6 | The in-page controls (section links, "Explore the workflow", the M01 menu) | In-page navigation only: they move within the landing and never leave it, open a product screen or count as entry actions. Each section has a stable in-page address, so a shared link to a section (for example the FAQ) opens the landing at that section. |
| Q7 | Parameters in the landing's address | Every parameter — redirect or return target, invitation token, referral or campaign tag, pre-filled email, language, anything else — is ignored and none is forwarded to sign-in, signup or anywhere else. The in-page section address of Q6 is the only part of the address honoured. Invitations keep their own link (specification 008). |
| Q8 | Search-engine indexing and link previews | Only the landing is indexable, in EN. Sign-in, signup, verification, reset, recovery, invitation and every authenticated screen are marked not indexable. A shared link to the landing previews with the landing's own title and description, never with illustrative data presented as a result (S-62). |

Two rules below are derived from the design and from the decisions rather than decided separately; they are flagged so the product owner can veto them:

- **D-a** "Exactly two entry actions" counts destinations, not controls: "Create workspace" appears in the header, the hero and the closing section, and "Sign in" in the header (S-60), the M01 menu and the closing section; every control on the page either opens the signup screen, opens the sign-in screen, moves within the page (Q6) or opens one of the three operator documents (Q3). No third destination exists.
- **D-b** "Only active identity entry methods" means, for the MVP, local email-and-password sign-in and signup only. No federation ("Continue with …"), passkey or security-key entry appears on the landing, in any language or width, because those are post-MVP (scope; requirements I.2, I.3).

---

## 2. User stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US-1 | evaluator who has never heard of ASpectra | to understand on one page what the product does, how a scenario goes from intent to Run, and where humans decide | I can judge whether it fits my team before creating anything |
| US-2 | cautious evaluator | plain answers to "does importing or approving start a run?", "do I need AI or a connector?" and "does changing language change my test?" | I know the product will not act on its own |
| US-3 | evaluator ready to try | "Create workspace" to take me to signup with a verified email, then to workspace creation | I start properly, without a tenant being created behind my back |
| US-4 | returning person | to find "Sign in" without scrolling the whole page, on desktop and on my phone | I get to my work quickly |
| US-5 | person already signed in who opens the landing | the same public page, with nothing about me or my workspace on it | a shared or cached page never discloses me |
| US-6 | Spanish- or French-speaking visitor | to switch the page to ES or FR and keep that language on the sign-in or signup screen I go to next | I can read the product in my language |
| US-7 | visitor on a phone or tablet | the same content, statements and two actions reflowed for my screen | I learn the same things as on a desktop |
| US-8 | keyboard or screen-reader user | every section, link, action and the language switch reachable and announced correctly | I can use the landing on equal terms |
| US-9 | privacy-conscious visitor | to read the page without being tracked or having anything stored on my device | visiting does not cost me my privacy |
| US-10 | visitor who wants the legal terms or a contact | Privacy, Terms and Contact to open the operator's published documents | I can check them before signing up |
| US-11 | visitor who receives a link to the FAQ | the link to open the landing at the FAQ | I land on the answer I was sent |
| US-12 | attacker or curious visitor (negative story) | — | I cannot make the landing redirect anywhere through its address, carry a token or address into sign-in or signup, show a sample as a real result, reveal who is signed in, create or enter a tenant, or disappear when the identity services fail |

---

## 3. Functional requirements

Acceptance criteria use EARS notation: *Ubiquitous* ("The system shall…"), *Event-driven* ("When…"), *State-driven* ("While…"), *Unwanted behaviour* ("If…, then…"), *Optional* ("Where…").

Throughout, *the landing* means R01 and its counterparts (M01 and the tablet reflow); *visitor* means anyone who opens it, with or without a session; *entry action* means "Create workspace" or "Sign in"; *the operator documents* means the Privacy, Terms and Contact documents of Q3; *illustrative* means a sample made up to explain the product and labelled as such.

### RF-1 · The surface and its sections

- RF-1.1 The system shall serve the landing to any visitor without an identity, a session, a tenant or any other precondition. *(FR-IDN-140)*
- RF-1.2 The system shall present on the landing, in this order, the hero with the illustrative scenario preview, the workflow, the capabilities (starting from what you have, the catalog and its layers), the human-control boundary (suggestions are not facts, exact context and explicit consent, evidence without interpretive blur, clear boundaries for teams), beginning without the integration tax, the FAQ, the closing call to action and the footer, as R01 and M01 draw them. *(FR-IDN-140; R01, M01)*
- RF-1.3 The system shall present the landing's statements in product copy as designed, apart from the corrections recorded in S-61. *(R01, M01; S-61)*
- RF-1.4 The system shall present nothing on the landing that is read from a tenant, a project, an identity, a run, a catalog release or any other product record; every word, sample and number on it is part of the page itself. *(P3; decisions Q1, Q4)*

### RF-2 · The two entry actions

- RF-2.1 The system shall offer exactly two entry actions on the landing: "Create workspace" and "Sign in". *(FR-IDN-140; decision D-a)*
- RF-2.2 The system shall offer "Create workspace" in the header, in the hero and in the closing section, and "Sign in" in the header, in the M01 menu and in the closing section, so that each is reachable from the top of the page at every width. *(R01, M01; S-60)*
- RF-2.3 When a visitor activates "Create workspace", the system shall open the signup screen of specification 002. *(FR-IDN-145; specification 002)*
- RF-2.4 The system shall never create, reserve or enter a tenant, create an identity or send any message as a result of an action on the landing; workspace creation is reached only after signup, email verification and sign-in, as specification 012 RF-1.1 defines. *(FR-IDN-145; specification 012 RF-1.1, RF-7.2)*
- RF-2.5 When a visitor activates "Sign in", the system shall open the sign-in screen of specification 001. *(FR-IDN-140; specification 001 RF-1.1)*
- RF-2.6 The system shall pass nothing from the landing to the screen an entry action opens except the interface language of RF-6.3: no address parameter, email, token, referral, return target or section. *(decision Q7)*
- RF-2.7 If an entry action is activated more than once in quick succession, then the system shall open its screen once. *(LC-14)*

### RF-3 · The product boundary

- RF-3.1 The system shall state on the landing that acceptance of a mapping, approval of a bundle and the launch of a Run are separate decisions, and that acceptance and approval never launch a Run. *(FR-IDN-141; P1)*
- RF-3.2 The system shall state on the landing that AI is optional and only proposes, explains and cites, and that a person decides. *(FR-IDN-141; P2)*
- RF-3.3 The system shall state on the landing that a verified email is required, and that the managed runner is started only by a user decision. *(FR-IDN-141)*
- RF-3.4 The system shall not describe on the landing any capability that the implementation baseline does not offer, nor any autonomy — no statement that the product runs, approves, accepts, fixes or decides on its own. *(FR-IDN-141; P8; S-61)*
- RF-3.5 The system shall present the hero statement "Verified email. Optional AI. Human approval and Run are always separate." above the first scroll on every width. *(R01, M01; FR-IDN-141)*

### RF-4 · Illustrative content

- RF-4.1 The system shall label every preview block on the landing — the hero scenario card, its workflow strip and any other sample of a scenario, identifier, environment, count or outcome — with a visible "Illustrative" label that is also exposed to assistive technology. *(FR-IDN-142; S-61)*
- RF-4.2 The system shall mark every sample identifier on the landing as a sample (as R01 does with "sample:r4" and "sample:env-r3") and shall never show an identifier, count or outcome in a form that reads as a real tenant record, a benchmark or a coverage claim. *(FR-IDN-142; P8)*
- RF-4.3 The system shall draw the landing's examples from HTTP, Web and DB scenarios only. *(F-IDN-19; S-61)*
- RF-4.4 The system shall present no metric, customer name, customer logo, testimonial, performance figure or coverage percentage on the landing. *(FR-IDN-142; P8)*

### RF-5 · The FAQ

- RF-5.1 The system shall answer, in the FAQ, "Does importing or approving start a run?" with: no — mapping acceptance creates a revision, approval records a human decision, and Run is a separate explicit action. *(FR-IDN-143; R01)*
- RF-5.2 The system shall answer, in the FAQ, "Do I need AI or a connector?" with: no — structured manual authoring, pure static validation and base reports work without AI, and source integrations are optional. *(FR-IDN-143; R01)*
- RF-5.3 The system shall answer, in the FAQ, "Does changing language change my test?" with: no — EN / ES / FR interface labels, the Gherkin dialect and registered executable aliases are separate. *(FR-IDN-143; FR-X-043; R01)*
- RF-5.4 The system shall present the FAQ answers as visible text, not hidden behind an interaction that assistive technology or a page without scripts cannot reach. *(FR-X-042; RNF-1)*

### RF-6 · Interface language

- RF-6.1 The system shall open the landing in EN on every visit, regardless of the browser's preferred language or of any language chosen on an earlier visit. *(Decision: Q2)*
- RF-6.2 When a visitor chooses EN, ES or FR in the footer, the system shall present the whole landing in that language without leaving the page or losing the visitor's position. *(FR-IDN-144, FR-X-043; decision Q2)*
- RF-6.3 When a visitor who has chosen a language activates an entry action, the system shall open the sign-in or signup screen in that language. *(Decision: Q2; specifications 001 RNF-6, 002 RNF-7)*
- RF-6.4 The system shall not remember the chosen language once the visitor leaves the public entry journey, and shall not store it on the device or in any account. *(Decisions Q2, Q5)*
- RF-6.5 The system shall indicate the current language in text, not by colour alone. *(FR-X-042; S-62)*
- RF-6.6 A change of interface language shall change display only: it shall not change any statement's meaning, any example's executable text or any FAQ answer. *(FR-X-043)*

### RF-7 · Footer routes

- RF-7.1 The system shall present Privacy, Terms and Contact in the footer of the landing at every width. *(FR-IDN-144; decision Q3)*
- RF-7.2 When a visitor activates Privacy, Terms or Contact, the system shall open the corresponding document the platform operator publishes outside the product, and shall indicate before activation that the route leaves the product. *(Decision: Q3; A-65; S-62)*
- RF-7.3 If any of the nine operator destinations (Privacy, Terms and Contact × EN, ES, FR; a shared destination is an explicit entry, never a fallback — register `122:2` §15) is absent or malformed, then the landing shall not be released; the system shall never render a dead, placeholder or invented address. *(P13; decision Q3; A-65)*
- RF-7.4 The system shall pass nothing of the visitor — no address parameter, identity or session — to an operator document. *(Decisions Q5, Q7)*
- RF-7.5 The system shall present the footer copyright line as designed. *(R01, M01)*

### RF-8 · Active entry methods only

- RF-8.1 The system shall present on the landing only the identity entry methods that are active, which for this release are local email-and-password signup and sign-in. *(FR-IDN-144; decision D-b)*
- RF-8.2 The system shall not present any federation, single sign-on, passkey or security-key entry, nor any statement that one is available, on the landing. *(FR-IDN-144; scope post-MVP; decision D-b)*

### RF-9 · In-page navigation

- RF-9.1 The system shall offer the header links Workflow, Capabilities, Human control and FAQ, and the hero action "Explore the workflow", as moves within the landing to the corresponding section. *(Decision: Q6; R01)*
- RF-9.2 The system shall give each of those sections a stable in-page address. *(Decision: Q6)*
- RF-9.3 When the landing is opened with the in-page address of a known section, the system shall open the landing at that section. *(Decision: Q6)*
- RF-9.4 If the landing is opened with an in-page address that matches no section, then the system shall open the landing at its top without any message. *(Decision: Q6)*
- RF-9.5 When an in-page move completes, the system shall move focus to the heading of the target section. *(FR-X-042)*
- RF-9.6 Where the width is that of M01, the system shall offer the header links and the two entry actions through the "Menu ☰" control; opening it shall not leave the landing, focus shall stay inside the open menu, and Escape or choosing an entry shall close it. *(M01; decision Q6; S-60)*

### RF-10 · The same page for everyone

- RF-10.1 The system shall serve the same landing to a visitor with a valid session as to a visitor without one. *(Decision: Q1)*
- RF-10.2 The system shall show nothing on the landing that depends on a session, identity, membership or tenant — no name, address, avatar, workspace, greeting or "continue" action. *(Decision: Q1; P3)*
- RF-10.3 The system shall neither read, refresh, extend nor end a session when the landing is opened or used. *(Decision: Q1; specifications 006, 007)*
- RF-10.4 When a signed-in visitor activates an entry action, the system shall open the corresponding screen, whose behaviour for an already signed-in identity is that of specifications 001 and 002. *(Decision: Q1; S-26)*

### RF-11 · The address is not an instruction

- RF-11.1 The system shall ignore every parameter in the landing's address — redirect or return target, invitation token, referral or campaign tag, pre-filled email, language or any other — and shall forward none of them. *(Decision: Q7)*
- RF-11.2 The system shall never redirect a visitor from the landing to a destination named in its address. *(Decision: Q7)*
- RF-11.3 If the landing is opened with an invitation token in its address, then the system shall present the landing unchanged, without accepting, revealing or forwarding the token; invitations are opened only through their own link. *(Decision: Q7; specification 008)*
- RF-11.4 The system shall never display on the landing any value that arrives with its address. *(Decision: Q7)*

### RF-12 · No tracking

- RF-12.1 The system shall load no analytics, advertising, tracking or other third-party script, font, image or resource when the landing is opened or used. *(Decision: Q5)*
- RF-12.2 The system shall set no cookie and store nothing on the visitor's device from the landing. *(Decisions Q2, Q5)*
- RF-12.3 The system shall show no consent banner on the landing, because nothing that would need consent happens. *(Decision: Q5)*
- RF-12.4 The system shall keep no record of individual landing visits beyond what the platform's protective logging requires, and shall attach no visit to an identity. *(Decision: Q5)*

### RF-13 · Indexing and link previews

- RF-13.1 The system shall allow search engines to index the landing, in EN. *(Decision: Q8)*
- RF-13.2 The system shall mark sign-in, signup, email verification, password reset, account recovery, invitation and every authenticated screen as not indexable. *(Decision: Q8)*
- RF-13.3 The system shall give the landing its own title and description for search results and link previews, and shall never use an illustrative sample as a preview of a real result. *(Decision: Q8; FR-IDN-142; S-62)*

### RF-14 · Always available, never pre-checking

- RF-14.1 The system shall render the landing completely without any request to the identity, tenant, project, run or AI services. *(Decision: Q4; P6)*
- RF-14.2 While the identity services are degraded or unavailable, the system shall still render the landing with both entry actions enabled. *(Decision: Q4)*
- RF-14.3 If the screen an entry action opens cannot be served, then the failure shall be reported by that screen's own states (specifications 001 and 002); the landing shall show no status banner and shall not hide or disable an entry action. *(Decision: Q4)*
- RF-14.4 The system shall render the landing's content, entry actions, FAQ and footer routes without relying on scripts running in the browser; the language switch and in-page moves may need them, and without them the page remains readable in EN with both entry actions working. *(Decision: Q4; RNF-1)*

### RF-15 · Responsive surface

- RF-15.1 The system shall present the landing as M01 draws it at phone widths, as a genuine reflow, not a scaled desktop page. *(FR-X-050; M01)*
- RF-15.2 The system shall carry the same statements, illustrative labels, FAQ answers, two entry actions, language switch and footer routes at every width; only layout differs between R01, M01 and the tablet reflow. *(FR-X-050; S-61)*
- RF-15.3 Where no tablet landing is drawn, the system shall reflow R01 for tablet widths under the desktop rules, with every statement and action intact. *(FR-X-050; S-02)*

---

## 4. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| RNF-1 | **Accessibility.** The landing passes WCAG 2.2 AA in EN, ES and FR at desktop, tablet and mobile widths: one main heading, ordered section headings, skip link to the content, visible focus on every control, text alternatives for every illustrative image, status conveyed by text, reflow without horizontal scrolling at 320 CSS px, reduced motion honoured. | P14; FR-X-042, FR-X-050 |
| RNF-2 | **Honesty.** No statement, sample or preview on the landing can be read as a real result, benchmark, coverage claim or an autonomy the product does not have; every preview carries its label in every language. | FR-IDN-141, FR-IDN-142; P8 |
| RNF-3 | **Identity-independence.** The page a visitor receives is byte-for-byte the same in content whether or not they hold a session, for a given language; no response of the landing carries identity, tenant or session data. | Decisions Q1, Q4 |
| RNF-4 | **Privacy.** Opening or using the landing sends nothing to any third party and leaves nothing on the device. | Decision Q5 |
| RNF-5 | **Availability.** The landing stays available when every identity, tenant and execution service is down; its availability depends on nothing but the delivery of the page itself. | Decision Q4; P6 |
| RNF-6 | **Speed.** On a mid-range phone over a slow mobile connection, the hero statement, the illustrative label and both entry actions are readable and usable within three seconds of the request, and nothing moves under the visitor's pointer once shown. | FR-X-050 |
| RNF-7 | **Language parity.** EN, ES and FR carry the same statements, labels, FAQ answers and actions; no language omits or adds a claim. | FR-IDN-144, FR-X-043 |
| RNF-8 | **Address safety.** No value in the landing's address can change what the page shows, where an action goes, or what another screen receives. | Decision Q7 |
| RNF-9 | **Configuration integrity.** The nine operator destinations are validated before release; a release with any of them absent or malformed is refused. | P13; decision Q3; A-65 |
| RNF-10 | **Indexing correctness.** Exactly one public page — the landing — is indexable; every other screen is marked not indexable. | Decision Q8 |
| RNF-11 | **Copy discipline.** Every capability the landing names is one the implementation baseline offers at release; a capability removed from the baseline is removed from the landing in the same release. | RF-3.4; S-61 |

---

## 5. Limit cases

| # | Case | Expected behaviour |
|---|---|---|
| LC-1 | Visitor with a valid session opens the landing | Same page as anyone; nothing personal; session not read or touched (RF-10.1 … RF-10.3). |
| LC-2 | Signed-in visitor activates "Sign in" | Sign-in screen opens; specification 001 applies for an identity already signed in (RF-10.4; S-26). |
| LC-3 | Signed-in visitor activates "Create workspace" | Signup screen opens; specification 002 applies for an identity already signed in; no tenant created (RF-2.3, RF-2.4, RF-10.4). |
| LC-4 | Browser prefers ES or FR | Landing opens in EN; the visitor may switch (RF-6.1, RF-6.2). |
| LC-5 | Browser prefers a language the product does not offer | Landing opens in EN (RF-6.1). |
| LC-6 | Visitor chose FR, then activates "Sign in" | Sign-in screen opens in FR (RF-6.3). |
| LC-7 | Visitor chose ES, closes the tab, returns the next day | Landing opens in EN; nothing was stored (RF-6.1, RF-6.4, RF-12.2). |
| LC-8 | Address carries `?lang=fr` or similar | Ignored; landing opens in EN (RF-6.1, RF-11.1). |
| LC-9 | Visitor switches language while scrolled to the FAQ | Page shown in the new language at the same section; no reload to the top (RF-6.2). |
| LC-10 | Address carries a return or redirect target to another site | Ignored; no redirect; nothing forwarded to sign-in (RF-11.1, RF-11.2). |
| LC-11 | One of the Privacy, Terms or Contact addresses is missing at release | Release refused; the landing never ships with a dead or placeholder route (RF-7.3, RNF-9; A-65). |
| LC-12 | An operator document is unreachable when a visitor opens it | The failure is that external site's; the landing is unaffected and makes no claim about it (RF-7.2, RF-14.1). |
| LC-13 | Identity services down when a visitor activates "Sign in" | Landing rendered normally; the sign-in screen reports its own failure (RF-14.2, RF-14.3). |
| LC-14 | Visitor double-clicks "Create workspace" | Signup screen opens once (RF-2.7). |
| LC-15 | Visitor returns to the landing with Back from signup | Landing shown as before; nothing entered on signup is carried back or kept by the landing (RF-2.6, RF-12.2). |
| LC-16 | Address carries an invitation token | Landing unchanged; token neither accepted, displayed nor forwarded; the invitee uses the invitation link (RF-11.3; specification 008). |
| LC-17 | Address carries a pre-filled email | Ignored; signup and sign-in open with empty fields (RF-2.6, RF-11.1). |
| LC-18 | Address carries script or markup in a parameter | Nothing displayed or executed; page unchanged (RF-11.4, RNF-8). |
| LC-19 | Link to `#faq` (or another known section) shared | Landing opens at the FAQ, focus at its heading (RF-9.3, RF-9.5). |
| LC-20 | Link to an unknown section | Landing opens at the top, no message (RF-9.4). |
| LC-21 | Scripts disabled in the browser | Content, FAQ, entry actions and footer routes work in EN; language switch and smooth in-page moves may not (RF-14.4). |
| LC-22 | Phone width, visitor opens the menu | Menu holds the four section links and the two entry actions only; focus stays inside; Escape closes (RF-9.6; S-60). |
| LC-23 | Tablet width | R01 reflowed under the desktop rules with every statement and action intact (RF-15.3; S-02). |
| LC-24 | Screen reader on the hero preview | Preview announced as illustrative before its content (RF-4.1, RNF-1). |
| LC-25 | Zoom to 400 % on desktop | Page reflows to one column without horizontal scrolling; nothing hidden (RNF-1). |
| LC-26 | Visitor prefers reduced motion | No decorative motion on the landing (RNF-1). |
| LC-27 | Search engine crawls sign-in or signup | Marked not indexable; only the landing is indexed (RF-13.1, RF-13.2). |
| LC-28 | Landing shared on a chat tool | Preview shows the landing's title and description, never a sample as a real result (RF-13.3). |
| LC-29 | A capability named on the landing is removed from the baseline | Removed from the landing in the same release (RNF-11; S-61). |
| LC-30 | Shared computer: previous person signed in, next person opens the landing | Nothing of the previous person shown (RF-10.2, RNF-3). |
| LC-31 | Visitor looks for "Continue with Google" or a passkey | None offered; only local signup and sign-in (RF-8.2). |

---

## 6. Out of scope

The following are separate features or absent from the product. This specification reaches them only through the routes named above and specifies nothing about their behaviour.

- **The content of the Privacy, Terms and Contact documents**, and the operator's mechanism for publishing their addresses — outside the product (decision Q3; A-65). Only their presence, their external cue and the release rule are specified here.
- **Signup, email verification, sign-in, the TOTP challenge, password reset and controlled account recovery** — specifications 002, 001, 004 and 005; the landing only opens the first two.
- **Workspace creation and bootstrap** — specification 012; the landing never reaches it directly.
- **Invitation acceptance** — specification 008; invitations never pass through the landing.
- **Sessions and sign-out** — specifications 006 and 007; the landing neither reads nor changes a session.
- **Federation, single sign-on, passkeys and security keys** — post-MVP (scope; requirements I.2, I.3).
- **Visitor analytics, A/B tests, marketing attribution, newsletters, chat widgets and consent management** — none in this release (decision Q5).
- **Pricing, plans, billing, demo booking, a blog, documentation, a status page, customer stories or any other public page** — not part of the product in this release.
- **Session- or account-aware content** on the landing — excluded by decision Q1.
- **Remembering a language between visits or across devices**, and language detection from the browser — excluded by decision Q2.
- **Indexing in ES and FR** — decision Q8 indexes EN only.
- **A dedicated tablet landing** — none; R01 follows the D01 responsive policy at tablet widths (S-02).

---

## 7. Criteria of completion

The public landing is complete when all of the following hold:

1. Every user story US-1 … US-11 can be walked end to end on R01, M01 and the tablet reflow with no step outside this specification, and US-12 is demonstrably impossible.
2. Every acceptance criterion RF-1 … RF-15 has at least one verification that fails before and passes after the feature exists.
3. Content tests show every section, statement and FAQ answer of RF-1, RF-3 and RF-5 present in EN, ES and FR, every preview labelled illustrative, every sample identifier marked as a sample, examples drawn from HTTP, Web and DB only, and no capability named that the baseline does not offer (RF-1, RF-3, RF-4, RF-5, RNF-2, RNF-7, RNF-11).
4. Routing tests show exactly two entry destinations, "Create workspace" opening signup and "Sign in" opening sign-in, no tenant, identity or message created by the landing, and nothing but the interface language passed on (RF-2, RF-8, LC-3, LC-14, LC-17).
5. Identity-independence tests show the same content with and without a session, no session read, refreshed or ended, and nothing personal on a shared computer (RF-10, RNF-3, LC-1, LC-2, LC-30).
6. Address tests show every parameter ignored — redirect, token, email, language, markup — no redirect, nothing displayed, nothing forwarded, and only known in-page section addresses honoured (RF-9.3, RF-9.4, RF-11, RNF-8, LC-8, LC-10, LC-16 … LC-20).
7. Language tests show EN on every visit whatever the browser prefers, the chosen language carried into sign-in or signup, nothing remembered or stored, and the current language shown in text (RF-6, LC-4 … LC-9).
8. Privacy tests show no third-party request, no cookie, no device storage and no consent banner when the landing is opened and used (RF-12, RNF-4).
9. Availability tests show the landing rendered with both actions enabled while identity, tenant and execution services are down, failures reported only by the target screens, and the page readable with scripts disabled (RF-14, RNF-5, LC-13, LC-21).
10. Footer tests show Privacy, Terms and Contact opening the operator documents with an external cue, and a release with any address absent or malformed refused (RF-7, RNF-9, LC-11).
11. Indexing tests show the landing indexable in EN with its own title and description, and every other screen marked not indexable (RF-13, RNF-10, LC-27, LC-28).
12. The landing passes WCAG 2.2 AA with automated and manual assistive-technology evidence in EN, ES and FR at desktop, tablet and mobile widths, and meets the speed target of RNF-6 (RNF-1, RNF-6, LC-22 … LC-26).
13. The register items this specification relies on — A-65 (with `122:2` §15), A-02 and A-08 in the architecture register; S-60, S-61, S-62, S-26 and S-02 in the D03 Screen decisions ledger (`148:4`) — are resolved, or explicitly deferred by the product owner, before the feature is declared done. Decision Q3 brings the privacy, terms and contact routes of FR-IDN-144 into this feature; constitution v1.7 (2 November 2026) binds FR-IDN-144 fully in F-IDN-19.
