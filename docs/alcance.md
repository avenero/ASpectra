# A·Spectra — General Scope Definition

> **Status: the whole document is defined.** §0.1 through §4.11 were decided by the owner directly; §4.12
> through §15 were defined on 2026-10-19 by delegation — derived from the decided sections and from the
> code, re-verified that day — and **await the owner's confirmation**. Nothing from §4.12 onward is a work
> order until it is confirmed. **35 decisions remain the owner's alone** — money, legal and contractual
> posture, and every number promised externally — and each is marked **Owner decision** under "Still open"
> in its own section. One decided answer is time-limited and says so in place: §1.3 is **provisional** and
> expires at release MVP-1.0.1, when machine access (§4.18) ships.
>
> **This defines GENERAL scope. The MVP 1.0 cut is a separate, later exercise.** An answer of
> "in scope" here does NOT mean "build it now".

| | |
|---|---|
| Owner | `avenero` — single owner (§0.1) |
| Created | 2026-10-05 |
| Supersedes | the 2026-10-01 `requirements.md` baseline, now re-derived from this document — see §13 |
| Source material | the code in the three repositories (authoritative), the Figma file `p06HxSrh2afAdXopnk4Qvs`, `requirements.md` (a 2026-10-01 FE baseline, partly stale) |

---

## 0. How to use this document

Each scope item has the same shape:

> **What it is, in plain words** — no jargon. What a non-engineer needs to know.
> **What we gain** — why it is worth having at all.
> **Where it stands today** — verified against code. `[VERIFIED]` means it was checked in this session.
> **Decision:** ☐ In general scope ☐ Out of scope ☐ Undecided
> **Rationale:**
> **Questions** — tagged `[BLOCKING]` (scope cannot be defined without an answer) or `[non-blocking]`.

Three conventions worth stating before you start:

1. **Reliability order of our own sources is: code > Figma > `requirements.md`.** Established by audit — see §13.
2. **"Built" ≠ "offered" ≠ "proven".** About 25 configuration flags hide finished features; two database
   engines are built and have never been executed against. §11 and §13 track this explicitly.
3. **Where a number or claim could not be confirmed, it says so.** Nothing here is filled with a guess.

### 0.1 Document rules — DECIDED

| Rule | Decision |
|---|---|
| Owner | Single owner. No second signature is required for an answer to count as decided. |
| Standing vs. the other documents | **The single source of truth for scope** (owner decision, 2026-10-20). `requirements.md` restates it as numbered requirements and loses any disagreement with it. Engineering guidance lives in the canonical rule documents of the ASpectra repository (`docs/development/engineering-rules.md` and its siblings); they point at this document instead of restating scope. |
| This document vs. the code | **On a scope question, this document wins.** Code that contradicts it is a defect to remove or a finished feature to gate — not a fact to document. |
| An unanswered question | **Out of general scope by default — but never by silence.** Every `OUT` carries a written reason. |
| Permitted answer values | `IN` / `OUT` / `LATER-RELEASE` / `UNDECIDED`. `OUT` and `LATER-RELEASE` each require a one-line reason; `LATER-RELEASE` names the release. |
| Audience | **Internal only** — team, development, engineering. Unflattering facts therefore stay written plainly; they are more useful that way. |
| Per-answer metadata | Not required. No date or author per answer. |
| Where this document lives | **The ASpectra governance repository** (`avenero/ASpectra`, private), at `docs/alcance.md`. |

**The folder structure, settled — revised by the owner on 2026-10-20.** The workspace root is itself the
private governance repository **ASpectra**, holding the workspace-level documents, the cross-repository
rules and the agent configuration:

```
ASpectra/                      # governance repository, branch main
  docs/alcance.md              # this document - the scope source of truth
  docs/requirements.md         # numbered requirements derived from this document
  docs/diagrams/               # the mermaid architecture views
  docs/architecture/, docs/development/, docs/deployment/
  .claude/, .cursor/, AGENTS.md
  qa-platformFE/ qa-platformBE/ qa-platformCore/   # independent repositories, ignored by ASpectra
```

This replaces the earlier answer — a separate `qa-platformDocs` repository beside a workspace root that
stayed a non-repository — for three reasons. The documents that needed versioning were not only these
three files but the whole cross-repository rule set and the agent configuration, which already lived at the
root. The concern about nesting three checkouts inside a fourth is answered by ignoring each of them by
explicit path, so the root can never record their files or a gitlink. And **no product repository gains a
dependency on this file**, so repositories remain self-contained (G17), which was the property the earlier
answer protected. Submodules stay rejected. The decision and its alternatives are recorded in ASpectra
`docs/architecture/decisions/0001-governance-repo-without-submodules.md`.

**`.claude/` is versioned where it carries shared instructions.** ASpectra versions its own `.claude/`
except local settings, and each application repository versions `.claude/CLAUDE.md` and
`.claude/settings.json`; `settings.local.json` stays uncommitted everywhere (ADR 0002).

---

## 1. Product definition and the explicit boundary

### 1.1 The one-sentence definition — DECIDED

**Where it stands today** `[VERIFIED]` — three different one-sentence definitions ship today, in
`.claude/CLAUDE.md`, `ORCHESTRATION.md` and the public landing page. They are not identical. None of them
is the sentence below.

> **Decision — the official sentence:**
>
> **A·Spectra is the automation framework a team does not have to build.**
>
> A QA whose test case in a test management system is nothing but a title and an ID opens A·Spectra,
> writes the step-by-step in Gherkin against a ready-made step catalog covering API, Web, Mobile and
> Database, runs it, and gets a result with evidence. The result is reported back against the ID. The
> step-by-step lives in A·Spectra because A·Spectra is where it runs — the test management system never
> held it and does not need to.
>
> **Rationale.** The founding problem is a real one, lived: a QA team whose cases in Xray were titles
> only, implementing each one in a separate automation framework per layer. Many organisations declare
> the name of a test case and nothing else, and then have to build and maintain a framework for API,
> Web, Mobile and Database. That framework — a repository, a stack, bindings, drivers, test data,
> reporting and a maintainer, four times over — is what A·Spectra replaces. It does not replace test
> management.

**The buyer.** No single persona. Any user, company or not, who needs a place to run test scenarios
without building a framework from scratch. A team whose cases are title-only, a team whose cases are
manual prose, and a team whose cases are already Gherkin are equally buyers.

**"BDD-first", clarified.** Gherkin is how a scenario is expressed and executed **inside** A·Spectra. It
is not a requirement of the customer's existing practice, and it does not exclude a customer whose test
cases today are a title or a paragraph of prose.

**The named alternative we expect to be compared against.** **The framework the team would otherwise
build itself** — one repository, one stack and one maintainer per layer. Commercial low-code suites that
sell the same promise are the secondary comparison.

**Differentiators** — the list is closed as it stands:

- modernisation;
- **one source of execution** instead of three or four frameworks for API, Web, Mobile and Database;
- online execution and online results;
- independence;
- scalability;
- an interactive console for runs.

**What the official sentence deliberately leaves out.** It does not lead with the TMS integration or
with human review. That is intentional: under §1.2 the integration is a convenience rather than the
product, so it does not belong in the buyer-facing sentence.

**Consequence recorded, not executed.** `.claude/CLAUDE.md`, `ORCHESTRATION.md` and the landing page all
lead with "execution overlay". Adopting the sentence above makes all three stale. Reconciling them is
tracked in §13; this document does not edit them.

**How a specification enters the platform.** A Postman collection or a Swagger/OpenAPI specification
enters only as a file the user holds and uploads: they download it from wherever it lives and import it in
the console. Reading one out of an attachment on a test-management issue is `OUT` of general scope — it
widens the token we must ask of the customer's Jira to include attachment read, and accepting a
customer-supplied archive brings decompression attack surface, in exchange for saving one download and one
upload. One upload door, well controlled, is the whole import story for specifications.

**Taking work out of the platform.** Two exports, both in general scope. The first already exists: a
Markdown documentation export of a scenario for a product owner, in English, Spanish and French. The second
is new: one standard `.feature` file per scenario. There is no exporter per target tool and there will not
be one — a family of per-tool exporters is recurring maintenance for something a customer uses once, on the
way out. What matters as much as the format is that the limitation is told to the customer plainly rather
than discovered by them: **the Gherkin text is portable, the step implementations are not.** The exported
`.feature` is readable and adaptable anywhere; what gives those steps behaviour is A·Spectra's own step
catalog.

**And the runtime does not leave with it.** A·Spectra does not distribute Core artifacts to customers.
Core compiles to `.jar` files and is consumable standalone, which would in principle let a customer execute
an exported `.feature` outside the platform — but obtaining those artifacts is not something A·Spectra
offers, and it is `OUT` of scope and off the product's goals. The export is portable text; the engine stays
ours.

**Results are reported per case, and aggregated.** One result is written back per source case, keyed by the
id it came from — not per container. Where one external case maps to several A·Spectra scenarios, or one
scenario runs several times through an `Examples` table, a **single aggregate result** is reported, and the
most severe outcome wins on the severity order the platform already applies: `FAILED`, `ERROR`,
`CANCELLED`, `SKIPPED`, `PASSED`. Letting the user choose the granularity, and reporting against a
container instead of a case, are `OUT`: both cost per-connector work on every connector the product will
ever have, for a choice no customer has asked for and in a vocabulary that is ours rather than theirs. This
reopens on the first real customer request, not before.

**Browsing the customer's test containers.** Their plans, executions and runs are reachable only through
the import-scope selector that already exists. There is no screen for navigating them: a browser over the
customer's test containers edges the product toward being a test management system, which §1.4 settles
permanently that it is not.

### 1.2 The primary-product question — DECIDED

**What it is, in plain words.** The product was read as containing two different products sharing one
engine: one that imports cases from a customer's test management system, makes a human approve each one
and runs them, and one that is a full editor where a user writes cases inside A·Spectra from scratch.

**Where it stands today** `[VERIFIED]` — the console's own structure:

| Surface | Routes | Page lines | Tests |
|---|---|---|---|
| Overlay (connect → import → review → run → sync) | **1** | 172 + ~2,000 across 20 components | mocked e2e + real-stack e2e; best-specified screen |
| Native authoring IDE | **5** | 1,669 across four pages | **no page-level test, no e2e** |
| Spec/story imports | **3** | 444 + **1,380** + 206 | thin; the 1,380-line page has none |

There is **no routed screen for canonical review at all** — the 7-state lifecycle lives inside panels of
one page.

> **Decision — one runtime, several entry doors.**
>
> There is no product split to resolve. A·Spectra is **one execution runtime with one stored scenario
> model**, reached through several entry doors: a test-management connector, a `.feature` upload, a
> Postman/Swagger/user-story import, and native authoring.
>
> **Native authoring is a first-class door, not a second product.** It is the door the founding use case
> walks through — a QA with a title and an ID — and it is open to anyone, with or without a test
> management system.
>
> **The test-management integration is a convenience, not the product.** It brings titles and context in,
> and sends results back out. Valuable; not constitutive.
>
> **Rationale.** The two-products reading came from describing the integration as the product. Once the
> product is the framework a team does not build (§1.1), the integration is a feature of it, and the
> apparent split disappears.

**The customer's test management system is the source of truth only for the cases that originated
there.** It holds no claim over a scenario authored in A·Spectra.

**What this decision costs.** The architecture review priced five consequences against an
"overlay-first" answer. Re-scored against the decision actually taken:

| | Consequence as the review stated it | Status |
|---|---|---|
| a | the single review gate moves into the shared launch validator | **Rejected.** §1.3 settles that review is a convention of the console, not an invariant of the backend. |
| b | the spec/story pipeline must terminate in the stored scenario model instead of writing executable rows directly | **Accepted.** Every door converges on one model. |
| c | Gherkin paste is re-routed through that same model rather than removed | **Accepted.** |
| d | roughly 12,000 FE lines and one bounded context become dead weight | **Rejected.** Authoring is a first-class door, so nothing is written off. |
| e | every natively authored or pasted scenario stops being launchable — a data migration, not a flag | **Rejected.** Same reason. |

Only (b) and (c) are accepted. Rejecting (d) and (e) is what makes this a convergence rather than a
rewrite; rejecting (a) follows from §1.3.

### 1.3 The human-approval guarantee — DECIDED (provisional: expires at MVP-1.0.1)

**What it is, in plain words.** The product has claimed that a machine never decided anything a human
did not approve. That claim is what a customer's auditor would be shown.

**Where it stands today** `[VERIFIED]` — the claim holds on one of three launch paths:

| Launch path | Review gate |
|---|---|
| `POST /api/projects/{id}/execution-plans` | **Yes** — requires `ReviewStatus.EXECUTABLE` |
| `POST /api/executions` | **No** |
| `POST /api/features/{id}/scenarios/bulk-execute` | **No** — and with no ids it runs *every* active scenario in the feature |

Three further facts, all verified:

1. **There is no approver grade.** Project permissions are exactly `READ, READ_WRITE`. The person who
   approves a case is the same grade as the person who wrote it. The audit records *who clicked*, not
   *who was entitled to*.
2. **Approval is revocable in one table and irrevocable in the other.** When a source case changes, the
   canonical case correctly returns to `IMPORTED` — but the projection row for the previous version stays
   `active = true` and remains launchable. There is no deactivation of superseded projections anywhere.
3. **Four different paths write executable rows**, and only one passes a review gate.

> **Decision — review is a convention of the console's flow, not an invariant of the backend.**
>
> The console walks a user through review before a run. The API does not enforce it, and will not be made
> to. The human-approval claim therefore describes **the flow**, never **the product**: it may say that
> the console asks for a review step before a scenario runs; it may not say that a scenario cannot run
> without one.
>
> **Rationale.** Once the product is the framework a team does not have to build (§1.1), review stops
> being constitutive — in a framework nobody approves a test, it is written and it runs. The stronger
> claim was inherited from the overlay framing, and that framing is no longer the product.

**What this settles:**

| Question | Settled |
|---|---|
| Does the guarantee hold for every entry door and every launch path? | **For none of them, as an enforced property.** Review is not claimed as an invariant on any path. |
| Fact 1 — there is no approver grade | **No longer a defect.** With no enforced approval there is nothing for an approver grade to gate. Where a hard control does belong is open below. |
| Fact 2 — a superseded projection stays `active = true` and launchable | **Correct behaviour. Keep it.** It follows directly from a changed source not blocking execution. **Do not "fix" this.** |
| Fact 3 — four paths write executable rows and one is gated | **The state we keep.** Consequence (a) of §1.2 is rejected: the review gate does not move into the shared launch validator. |

**Two things this decision does not loosen.**

- **A changed source does not block execution.** Because A·Spectra never writes steps into the customer's
  test management system, a source case may change while the scenario stays executable — as long as its
  steps still resolve against the step catalog, since Core fails closed on a step it cannot resolve.
- **The AI guarantees are untouched.** "Nothing a model produces reaches domain state without a human
  decision" is a separate and narrower rule about AI output, and it holds in full. This decision concerns
  the review lifecycle of a scenario, not AI governance. Reading it as a relaxation of the AI rules is
  reading it wrong.

**This decision is provisional, and the expiry date is known.** It holds only while the console is the
only way in. The product is indifferent to CI (§1.4) and has no machine entry point, so nothing bypasses
the console today — but machine access is not refused, it is **deferred to release MVP-1.0.1** (§1.4,
§4.18). **The day a public API, webhook or CLI trigger exists, the convention evaporates** and this
question reopens as *contract* or *auditable property*. That reopening is therefore scheduled rather than
hypothetical: at MVP-1.0.1 this section must be re-decided, and whoever answers §4.18 answers this one
with it. Deferring machine access rather than ruling it out is what makes the human-approval answer
temporary, and that is recorded here on purpose so the expiry is not discovered later.

**The gap this leaves, named plainly.** Under this answer there is no hard control anywhere in the
product. That is acceptable for the content of a scenario. It is a different question for running
against a sensitive environment, which is the one place an enterprise customer will ask for a real gate
— see the open item below and §1.5 tier 4.

**Consequence recorded, not executed.** `.claude/CLAUDE.md` (*"a machine never decided anything a human
did not approve"*, *"requires a human to review and approve them"*), `ORCHESTRATION.md` and the public
landing page all carry the stronger claim. All three are stale on this point and must be re-worded to
describe the console flow. Tracked in §13; this document does not edit them.

**The one hard control the product does have: sensitive environments.** Review is not enforced, but
execution against a sensitive environment is. The control is declared **on the environment**, per project:
an environment can be marked protected, and launching against a protected environment is gated. It covers
**production and pre-production** — the internal test stage — and nothing else; a development or QA
environment is never gated. The control is **available, not required**: a project may leave its
environments unprotected, and a one-person project is unaffected either way, since the same person holds
both roles. An organisation-level policy is `LATER-RELEASE`. When both exist, **the organisation policy
wins** — a project cannot opt out of an organisation rule.

**This control is designed before it is built, deliberately.** Two environment classifications already
exist in the data model and overlap: `type` (`DEV, QA, STAGING, PRODUCTION, MOBILE`, a Postgres enum) and
`kind` (`standard`, `production-like`, `sandbox`, a free string whose stated purpose is to drive banners and
product rules). Which of the two governs the gate — and what becomes of a mobile environment, given that
`MOBILE` is a value of `type`, so a mobile production environment cannot be typed `PRODUCTION` today — is a
design question this scope answer deliberately does not pre-empt. The scope is fixed; the mechanism is to
be analysed, refined and designed before any implementation starts.

**Drift is reported, never blocked.** A source case can change while its scenario stays executable, and
this section keeps it that way. What the console does is **say so**: where a project has a test-management
integration configured, a scenario whose source case has moved is shown as deviated. Nothing about the run
is prevented. This is the honest complement to keeping a superseded projection launchable — and because
A·Spectra never writes steps back into the customer's system, reporting the deviation is the only thing it
can do about it. Where that indicator lives, and what it says, is §4.14's to design.

**Project permissions are not a closed set.** They are `READ` and `READ_WRITE` today, as a Postgres enum
type, so a third grade is a migration rather than a setting. A third grade is expected, and it arrives in
**release 1.0.0** together with the environment control above. What this records is the instruction that
matters before then: two permissions are not to be treated as the final shape of the model, and anything
built against them must survive a third.

### 1.4 The explicit boundary — DECIDED

**Where it stands today** `[VERIFIED]` — the public page lists exactly four non-goals: it does not
replace authoring in your TMS, does not import from Azure DevOps, does not let AI decide pass/fail, does
not keep credentials outside an encrypted store.

**And the public page contradicts itself about what we support, in three languages:**

| String | Claims |
|---|---|
| `landing.scope.connector` | "Supported connector: Jira with Xray Server/Data Center" — **one** |
| `landing.scope.notSupported` | "Xray Cloud, Azure DevOps and TestRail imports are **not** supported" |
| `landing.heroSubtitle` | "Import from Jira + Xray **or TestRail**" |
| `landing.trust.*`, `landing.faq.tools.a`, `landing.not.azure` | Server/DC **+ Cloud + TestRail** — **three** |

The code offers **two**. This ships in English, Spanish and French.

**Decision — A·Spectra is NOT:**

| Not… | Permanent? |
|---|---|
| **a writer of test-case content into the customer's test management system.** Results and evidence only — never steps, never Gherkin | **Permanent** |
| **the customer's test case catalog.** The title and the ID stay in their system | **Permanent** |
| **a test management system.** No test planning, no cycles, no tester assignment, no requirement coverage, no defect management, no manual step-ticking execution by a human tester | **Permanent** |
| **a CI tool.** Indifferent to CI | **Permanent** as positioning. Whether a machine entry point exists at all is a separate scope question (§4.18) |
| a place where AI decides pass/fail | **Permanent** |
| a holder of credentials outside the encrypted store | **Permanent** |
| an Azure DevOps importer | **Not permanent** — no adapter exists today |
| ~~a replacement for authoring in the customer's TMS~~ | **Withdrawn.** Superseded by §1.2: authoring is a first-class door, open to everyone |

**On storage — corrected against the code.** A·Spectra **does** store the scenarios a user imports or
authors. Execution requires them, and they are the customer's asset: the step-by-step is the work the
customer paid for, and in the founding use case it is the only place that work exists. What A·Spectra is
not is the customer's **system of record** — the test management system keeps that role for the cases
that originated there, and everything built in A·Spectra can be exported out. **Retention, export and
deletion are therefore in general scope**, not as a legal obligation but because a customer must be able
to take their work and leave.

**Can a customer buy and use the product with no test management system connected at all?**
**Yes, unconditionally.** Native authoring and `.feature` upload are enough, with export as the way work
leaves the platform.

**"The customer's TMS stays the source of truth"** is **completely optional** — neither enforced nor the
default posture.

**The public claim surface** is the landing page. The README is internal. This document is internal and
is the owner's scope instrument. Because §0.1 settles that this document wins a scope question, **the
landing page is to be corrected to match this document and the code** — it contradicts itself today in
three languages.

**The canonical product name and spelling is `A·Spectra`,** everywhere.

**Reconciling the public claim surface, and when.** The landing page, `.claude/CLAUDE.md`,
`ORCHESTRATION.md` and the product-name spelling are all corrected **after this scope document is closed**,
as a task of its own. Nothing in them is edited while the scope is still being written: the scope is
settled first, and then every document that contradicts it is brought into line in one pass. This is
recorded here so that a stale or self-contradictory claim found in the meantime is understood as known and
queued, not as a new discovery.

**A missing integration never blocks a run.** Whatever "the customer's test management system stays the
source of truth" is set to, it is optional, and it is never a precondition for execution. A project with no
connector configured, and an organisation that has integrated nothing, run their scenarios exactly like any
other. The optionality is a posture, not a gate, and it is set **per project** — where integrations are
already configured — with no organisation-level equivalent.

**Machine access is deferred, not refused — and that defers §1.3 with it.** Indifference to CI is
positioning; it is not the same as having no machine entry point, so it does not settle §4.18 by
implication. A public API, webhooks and a CLI trigger are `LATER-RELEASE`, release **MVP-1.0.1**. The
consequence is deliberate: because a machine entry point is scheduled rather than ruled out, the
human-approval decision of §1.3 is **provisional** and expires when that entry point ships. §1.3 is marked
as such in place, and §4.18 remains its own question to answer when that release is designed.

### 1.5 The job to be done, and what the user can do — DECIDED (criterion)

**The job, in one paragraph.** A QA has a test case that is a title and an ID. Their only route to
executing it today is to build and maintain an automation framework per layer. A·Spectra replaces that
framework: the QA authors the step-by-step here, runs it here, and the result is reported against the ID.

The capabilities below are tiered by what the job requires, not by preference. This section fixes **what
the user can do**; the per-module detail, current state and decisions land in §4 when §4 is reviewed.

**Tier 1 — Constitutive. The objective is impossible without these.**

1. A place for the ID and title to enter: from the connector, typed by hand, or a bulk list.
2. Authoring Gherkin against a **discoverable** step catalog — a QA who cannot search the catalog is
   guessing.
3. Validation before execution: "this step does not exist" said while authoring, not at run time.
4. Environments and credentials held **outside** the scenario text, so one scenario runs against several
   environments.
5. Execute, and get pass/fail.
6. A per-step result and the reason for a failure.
7. **The scenario is stored.** Otherwise it is re-authored on every run. This is the customer's asset.

**Tier 2 — Viability. Without these the product demonstrates well and fails past roughly 20 scenarios.**

8. **A work queue: which cases still have no steps.** Not a catalog — a backlog of what is left to
   implement. The most underestimated requirement in the whole idea.
9. **Reuse** — macros, shared step sequences, `Background`. The same five steps recur in eighty
   scenarios; without reuse, eighty scenarios cost eighty times.
10. Data-driven scenarios: `Examples` tables, so one scenario covers many cases.
11. **Run one scenario while authoring it, fast.** The write → run → fail → fix loop. Without it, authoring
    is blind.
12. **Evidence at the failure:** screenshot, HTTP request and response, the SQL, the mobile screen, logs.
    A result without evidence forces a manual reproduction, which is what the product removes.
13. Parallel execution.
14. **Help obtaining selectors** — see the hard problems below.
15. **Context attached to the case** — the linked story, bug or task, acceptance criteria, notes,
    attachments. Not a separate feature: it is the input the author needs to write correct steps. A title
    such as "login with an invalid password shows an error" names neither the URL, nor the credentials,
    nor the error text.
16. Suites and grouping; run a selection.
17. History per scenario: did it ever pass, and when did it start failing?

**Tier 3 — Where automation programmes die. What makes this a platform rather than a tool.**

18. **Failure triage:** group many failures by cause. The application changes, forty scenarios break — say
    which and why, in one place.
19. **Flakiness detection.** A scenario that passes seven times in ten is worse than one that always fails.
20. **Impact analysis:** a catalog step changes — which scenarios break?
21. **Bulk edit** across scenarios.
22. **Versioning**, who changed what, and revert.
23. An owner per scenario.
24. Scheduled or unattended runs.
25. Notification on failure.

Tier 3 is where the hosted model beats a framework decisively: in a framework, maintenance needs a
developer and a release; here it is an edit in a console.

**Tier 4 — Governance.**

26. Roles: who authors, who approves, who may run against production.
27. An audit trail.
28. A gate on running against production.

*Tier 4 reads through §1.3 and §2: authoring and approving are both held by project write access,
because the product has no approver grade, and the one control that is a real gate is item 28 —
running against a protected environment, which takes the higher project grade.*

**The authoring loop has a committed speed, per layer.** The speed of running a single scenario is a
condition of the product existing, not a performance metric, so it is recorded as a target with a
breakdown rather than as an aspiration: an API or Database scenario in **under five seconds**, a Web or
Mobile scenario in **under twenty**, measured from the launch click to the first step result. Meeting the
Web target implies a warm browser pool, which the product does not have today; that consequence is deferred
to §9.1 as architecture work, not relaxed as a target.

**Browser choice is not negotiable.** When a user configures a Web execution they choose the browser —
Chromium, Firefox or WebKit, as the console already offers. No speed target may be bought by taking that
choice away. The consequence, recorded rather than solved: a warm pool must then exist per browser engine,
or the first run on a less-used engine pays the cold start and misses the target. §9.1 inherits that
problem together with the target.

**And the commitment already has one known breach, recorded so it is closed rather than discovered.** In
a managed run — the production default, where private hosts are not allowed — the engine rejects Firefox
and WebKit outright and only Chromium runs; on self-hosted, where private hosts are legitimately allowed,
all three run. Because browser choice is not negotiable, making managed runs accept all three engines is
required work, not a decision still to take.

**The selector cliff is answered with an element inspector, not with an assumption.** The product does
not assume its author can read a DOM — if it did, the dependency on a developer would have been moved
rather than removed, and the premise of §1.1 would fail. Instead: the user supplies the URL of the
application under test, A·Spectra opens it with the Playwright that Core already runs, the user picks the
element, and the platform proposes the selector. This is in general scope and it is not optional to the
premise; it is the single largest risk to the idea, and this is the answer to it.

**It reaches the customer's application through a runner inside the customer's network.** This is the part
that must be stated precisely, because the obvious assumption is wrong: a QA's own VPN puts **their laptop**
inside the customer's network, never A·Spectra's backend, and Playwright runs in the backend. A hosted
deployment denies private, loopback, CGNAT and unique-local addresses by design, so no amount of VPN on the
user's side makes a private application reachable. What makes it reachable is a **customer-side runner**
deployed inside their network, generalising a pattern the product already has: the mobile agent, reached
through an environment's agent URL over the versioned `/v1/runs` wire with token authentication, today
restricted to mobile environments.

**Consequence recorded, not executed.** That runner is not an inspector feature. It is the same thing that
makes **executing** Web and API scenarios against a private application possible at all, which the product
cannot do from a hosted deployment today. This decision therefore pre-answers §3's private-network
question in the affirmative — running inside the customer's own network is in scope — and §3 inherits the
work rather than the choice.

**The catalog's ceiling is answered in two steps, and the customer never writes a primitive.** First, when
a client needs something they cannot find, the platform proposes a **composed step**: a named, reusable
macro built out of steps that already exist and that does what they described. The mechanism exists today
as a custom step, recorded as either authored by hand or accepted from a suggestion — and a suggestion is
only ever a suggestion until a human accepts it, which is the AI rule applied here rather than loosened.
This costs no Core release and gives an immediate answer, and it covers the common case, where the gap is
one of composition.

**Where composition cannot reach, the step is requested.** Some needs have no primitive to compose from at
all — a protocol the catalog does not speak, an interaction it has no verb for, a database feature it does
not implement — and no arrangement of existing steps produces one. Those go through a documented step
request, and the step is added to Core in a release. The customer never authors a primitive themselves:
step patterns and step ids are the executable identity of every stored scenario and a frozen contract, so
they stay the platform's to write. The ceiling is therefore real but it has a door, and the door has a
published process rather than a dead end.

---

## 2. Users, personas, roles and permissions — DECIDED

**What it is, in plain words.** Who logs in, what each of them is allowed to do, and whether a customer
can enforce "the person who writes a test is not the person who approves it".

**Where it stands today** `[VERIFIED]`:
- **Four platform roles**: `ADMIN, QA_LEAD, QA_ENGINEER, VIEWER`.
- **Two project permissions**: `READ, READ_WRITE` — a Postgres enum type, so adding a third is a migration.
- **No approver grade, no org-scoped role, no group/team assignment, no auditor persona.**
- **Platform `ADMIN` is a vendor-wide superuser**: it resolves to `READ_WRITE` on *every* project, and
  `GET /api/users` and `GET /api/users/organizations` span every customer.
- MFA is self-service opt-in; there is no way to require it.

> **Decision — the permanent role model:**
>
> **Two kinds of people, and two levels of authority.** People with accounts author, run and administer.
> People without accounts receive results where they already work. Authority sits at two levels that are
> never the same thing: a **vendor** level that exists to support the product, and a **customer**
> organisation level that exists to govern one customer. Inside a project, authority is graded, and the
> grade above ordinary authoring is what guards credentials, data egress and sensitive environments.
>
> **Rationale.** The model follows from §1.3 rather than from a governance ambition. Review is not
> enforced, so there is nothing for an approver grade to gate and the role model does not need one. What
> does need a real gate is the small set of actions that reach outside the project — a protected
> environment, a connector credential, customer content leaving the deployment — and those are what the
> grading exists for. Everything else is ordinary authoring.

**The cast list.** Three personas hold accounts: the **QA author**, who writes and runs scenarios; the
**QA lead**, who holds the higher project grade; and the **platform administrator**. One persona holds no
account at all: the **product owner or stakeholder**, who receives results in the test management system
where they already live, and needs no seat to do it. There is no separate read-only auditor persona — the
output-only stakeholder is that persona, and nothing in the identity decisions below creates a compliance
role that would need more. One consequence is owed here: four platform roles exist today but collapse onto
two project permissions, so `QA_LEAD` and `QA_ENGINEER` are indistinguishable in practice. `QA_LEAD`
becomes real when it receives the third grade, or it is deleted; a role name that promises a hierarchy the
permissions do not deliver is worse than no role at all.

**Two administrators, deliberately separate.** Platform `ADMIN` stays a **vendor** role — it exists so we
can support the product — and it is renamed so that nobody mistakes it for the customer's own
administrator. Alongside it, a **customer organisation administrator** is a distinct role, scoped to one
organisation and holding no authority outside it. Today there is only the first, and it resolves to
`READ_WRITE` on every project while `GET /api/users` and `GET /api/users/organizations` span every
customer, so a customer has no administrator of their own and the vendor role has no boundary.

**Consequence recorded, not executed.** A customer-scoped administrator requires the organisation to be a
real boundary: an organisation column on project-owning data and tenant-scoped aggregate queries, neither
of which exists. §3 therefore inherits that work, not the choice — the choice is made here, which is the
entire point of making it now. Deciding it after the first hosted customer would be an incident rather
than a refactor.

**Write implies execute.** "Can execute" is not a separate permission from "can author". The protected
-environment gate of §1.3 already covers the only case where an unintended run is dangerous, so splitting
the two buys nothing and costs a dimension in the permission model. The feature-wide bulk-execute endpoint
is gated by an authoring permission today; that is now a decision rather than an accident of how the
endpoint was annotated.

**What the higher grade actually guards.** Two things, and they are the reason it exists: **launching
against a protected environment**, and **configuring a test-management connector** — which points
credentials with write access at the customer's own Jira and so has no business sitting behind ordinary
project write access. Everything else a QA does day to day needs nothing more than write.

**AI data egress is an organisation decision, not a project one.** The setting governs whether customer
test content may leave the deployment, which makes it the one switch a customer's security review will ask
about by name. It is off by default, it may be changed only by the customer's organisation administrator,
never by project write access, and the change is auditable.

**Approval needs no grade, because approval has no gate.** §1.3 settles that review is a convention of
the console rather than an enforced property. There is consequently no approver grade, no named reviewer
role and no per-project approver list: anyone with write access may move a canonical case through its
review states. The question is not deferred — the thing it would have gated does not exist.

**Deletion is soft where the asset is the customer's.** Projects and scenarios are soft-deleted with a
recovery window. In the founding use case the step-by-step exists nowhere else — the test management
system never held it — so a hard delete destroys work the customer paid for and that §1.4 makes us the
custodian of. Executions and evidence may be hard-deleted, subject to the retention rules of §9.4. On
permission: a customer organisation administrator may delete a project; an author may delete their own
scenarios.

**One defect, named as a defect and not as a scope choice.** `owner_id` is `ON DELETE SET NULL`, so
deleting a user currently dissolves a project's tenant boundary. No answer in this section resolves that
and none should be read as doing so: it is a defect to be fixed whichever way deletion goes.

**MFA is enforceable, and mandatory where it matters.** Optional per user is not sufficient. An
organisation may require MFA of all its members, and it is **mandatory** for anyone who can configure a
connector or change the AI data-egress setting. An administrator can see which users have enrolled.
Today MFA is self-service opt-in with no way to require it and no way to see it.

**SSO is in scope, as a paid enterprise capability.** It is not out of scope: it will be demanded, and
recording it out now would buy nothing except a scope reversal later. Pricing it as an enterprise
capability is what lets it fund the work it requires. Two facts about where it stands are worth keeping
in view: the OIDC sign-in handoff is held in a single-instance in-memory map, so SSO as built does not
survive a second backend instance, and the organisation claim written into the login token is never read
by any code.

**Impersonation is out of scope.** A vendor support role already provides a way in, and "view as" makes
the audit trail lie about who acted — a direct cost to the one guarantee §1.3 leaves standing. The reason
it is refused is the audit trail, not the difficulty.

**Assignment is per user, and groups are deferred.** A permission is granted to a person, never to a
team. Group- or team-based assignment is `LATER-RELEASE`, release **MVP-1.0.1**: it is an additive join
table rather than a change to the model, so deferring it costs nothing structural, and per-user assignment
stays correct until a customer has enough people on one project to feel the administrative weight.

→ See **§14.1 Identity: build or integrate**.

---

## 3. Tenancy and deployment model

**What it is, in plain words.** Do we run the product for customers, or do customers run it themselves? And
if we run it, what stops customer A from seeing customer B's data?

**What we gain by answering.** This decides the cost of everything in §9 and §10, and it is the one answer
that cannot be deferred cheaply — it is a data-model change, not a configuration.

**Where it stands today** `[VERIFIED]` — and the honest summary is that **multi-tenancy does not exist yet**:
- An `Organization` exists, but the code states its purpose verbatim: *"every sign-up gets an organization
  of its own, so no two customers share a rate-limit bucket."* It is a **rate-limit bucket**, not a tenant.
- `organization_id` exists **only on `users`**. `projects` has no organization column at all. A project's
  effective tenant is "the owner's organization" — and `owner_id` is `ON DELETE SET NULL`, so deleting an
  owner dissolves the boundary.
- `OrganizationPort` has no update and no delete. The entity has exactly one field: `name`.
- The organization claim written into the login token is **never read by any code**.
- Cross-tenant aggregate endpoints exist by design (`/dashboard/global`, `/dashboard/components`, `/api/users`).
- Rate limiting is in-memory **per instance**, so two instances give twice the limit.
- The OIDC sign-in handoff is held in a single-instance in-memory map.

> **Decision — both, sequenced, as one artifact.**
>
> **Self-hosted single-tenant first, vendor-hosted SaaS second**, shipped as **one artifact with
> configuration differences** rather than two products.
>
> **Rationale.** Self-hosted single-tenant is shippable today and costs nothing architecturally, and it is
> the natural home for the two capabilities §1 put in scope that a hosted deployment cannot perform at
> all: executing against a private application, and the element inspector that has to reach it.
> Vendor-hosted is the larger market and stays in general scope, but it is gated on work that does not
> exist rather than on a decision still to be taken. Two products would mean two scopes, two test matrices
> and two support stories for a single owner; one artifact with configuration differences keeps the
> surface single and the decision reversible in one direction only — forward.

**"Both" does not mean both work today, and the gap is named.** Multi-tenant hosted requires six things
that do not exist, three of them hard blockers: a tenant-scoped administrator role, an organization column
on project-owning data, and tenant-scoped aggregate queries. The first is decided in §2 and unbuilt. The
second has no partial version: `Project` carries no organization column at all, only a nullable
`owner_id`, so a project's effective tenant is "the owner's organization" and deleting that owner
dissolves the boundary. And `Organization` itself is today what the code says it is, verbatim — *"every
sign-up gets an organization of its own, so no two customers share a rate-limit bucket"* — a rate-limit
bucket rather than a tenant. The one place it already acts as a boundary is permission grants: a project
owner may only grant access to users of their own organization, a check added so that an owner could no
longer probe usernames across customers. It is not a boundary on project data.

**Isolation in the hosted model is the organization, row-scoped in one shared database.** A dedicated
deployment per customer is not a second isolation model to design: it is the self-hosted artifact sold as
managed, which this decision already produces and which therefore costs nothing extra to offer. What must
be stated so it is not deferred by accident: **row-scoping is a data-model change, not a configuration
setting.** It is the organization column plus every aggregate query learning which tenant is asking.

**Running inside the customer's own network is in scope**, as §1.5 settles — a customer-side runner,
generalising the mobile-agent pattern. The commercial question that used to follow from not having one,
whether the product is only sellable to customers whose application under test is reachable from our
egress address, therefore does not arise. That runner is what keeps the addressable market from narrowing
to publicly reachable applications.

**The organization becomes a first-class managed object.** It has to: §2 gives the customer an
administrator, and an administrator needs something to administer. It can be created, renamed, and list
its members and its projects. Its deletion follows the rule of §2 — soft, with a recovery window — since
deleting an organization would otherwise take every project inside it. Suspension is a commercial action
rather than a structural one and belongs to §10.

**One user belongs to exactly one organization.** The relation is permanently one-to-many: an organization
has many users, a user has one organization. Multi-organization membership is `OUT`, and the reason is
that the permission model stays single-valued — every authorization check resolves exactly one
organization for a user, with no notion of an active organization in the session and no check that has to
ask which tenant it is deciding for. The consequence is accepted and stated: a consultant working for two
clients holds two accounts.

**On self-hosting, the customer holds the keys and we cannot read their data.** That is most of the reason
a customer chooses to run it themselves, so the opposite answer would make the deployment model pointless.
Support is therefore served by a **diagnostics bundle the customer generates and sends**, redacted through
the single redactor the product already has (G23), rather than by our reading anything. On the hosted SKU
we hold the keys, and the vendor support role of §2 is the access path — stated plainly rather than left
implicit. One thing to know before this is revisited: key custody lives in the credential-store design, so
changing it later is a re-encryption migration over live customer secrets.

**High availability is in general scope, and losing an in-flight run on every deploy is not acceptable
permanently.** The important part is that this is the same piece of work as something already in scope: a
run lives inside the backend process today, and moving it out of process is exactly the customer-side
runner of §1.5 seen from the other side. The private-network question, the element inspector and run
durability are one piece of infrastructure, not three, and deciding them separately is how a team builds
it twice.

**The single-instance state, recorded as work rather than as choices.** A run lives in the instance's own
JVM, so a rolling restart ends it: a run that finishes within the 120-second shutdown drain survives,
anything longer is lost, and the next sweep marks it errored two to three minutes later — whichever
instance sweeps, including the restarted one. Evidence is a filesystem path, and the configuration's own
comment says two instances need storage both can read and write because *"a second instance with its own
disk answers 404"*; the sync worker then ships results without the screenshot, so whether evidence reaches
the customer's system depends on which instance claimed the sync job. The **inbound** HTTP rate limiter is
in-memory per instance, so two instances grant twice its allowance — while the **outbound** limiter for
connectors and AI is already database-backed and shared, so only half of rate limiting is on this path.
The OIDC sign-in handoff, the two-factor login challenge and the two-factor enrolment state are each held
in single-instance memory, so federated sign-in, two-factor sign-in and two-factor enrolment all break when
consecutive requests land on different instances, and the paid SSO capability of §2 cannot survive a
second instance until they move. None of these is a scope question; each is on the path to the answer
above.

**Data residency: one region, ours, and disclosed.** Customer-visible region choice is `OUT` — per-region
deployments, per-region storage and a routing story are the most expensive item in this section after
multi-tenancy, and the demand for it is speculative. There is no residency configuration anywhere in the
product today, so this is a commitment to make rather than a setting to flip. Worth noting as a sales
argument rather than a limitation: **the self-hosted SKU answers residency for free**, for exactly the
customers who ask the question hardest.

**Export: the scenarios, and an execution's evidence.** Scenario export is settled in §1.1 — Markdown
documentation and one standard `.feature` per scenario, with the engine staying ours. To that this section
adds **per-execution evidence export**: an execution's screenshots, transcripts and logs downloadable as an
archive, which earns its place in daily use rather than only on the way out. Bulk export of all historical
evidence is `OUT`; how long evidence exists at all is governed by the retention rules of §9.4.

**Still open:**

- `[BLOCKING]` **Owner decision — the region itself is not named.** The policy is decided — one region, ours, disclosed —
  but which region is a commitment the document does not yet carry, and residency is normally the second
  procurement question after SSO.

→ See **§14.9 Hosting, HA and the execution runner**.

---

## 4. Functional scope, module by module

### 4.1 Identity and access

**What it is.** Sign-in, passwords, two-factor, single sign-on, account administration.

**Where it stands today** `[VERIFIED]` — all built in-house: local login, access+refresh tokens with
rotation, per-account lockout, forced password rotation, password reset, TOTP two-factor with recovery
codes, generic OIDC federation (Google wired; any OIDC provider works by configuration), self-serve sign-up
capturing the *version* of terms consented to, admin activation, user administration, login-email recovery.
**Sign-up, SSO and email are all off by default.**

> **Decision — in scope, and owned, with one protocol to acquire.**
>
> Identity is the product's own: local password, two-factor and generic OIDC federation are built and
> stay built. **SAML** is the one method in general scope that does not exist, and whether it is written
> or integrated is deferred to §14.1 rather than assumed here.
>
> **Rationale.** What is already built covers every customer who has an OIDC provider, which is most of
> them, and it was cheaper to own than to rent. SAML is the exception because it is a second protocol
> rather than a second button, and because the customers who demand it — enterprise Jira Data Center
> installations — are precisely the 1.0 target.

**The permanent set of authentication methods.** Local password and **generic OIDC** — which already
covers Google, Entra and Okta by configuration rather than by adapter — are in scope and built. **SAML is
in general scope** as part of the paid single-sign-on tier of §2; it exists nowhere in the product today,
and enterprise Xray Data Center customers commonly require it, so recording it out would have been a
decision to decline that market. **LDAP and Active Directory binding are `OUT`**: direct directory binding
is a different trust model, and every modern deployment fronts a directory with OIDC or SAML anyway, so
supporting those two protocols reaches the same customers without holding their directory credentials.

**Account creation differs by deployment, which is a configuration difference and not a second product.**
On the hosted SKU, **self-service sign-up with administrative activation** — a trial has to be able to
start itself, and §1.1 already names admin-activated sign-up as the 1.0 behaviour. On self-hosted,
**administrative creation only**, because an enterprise running it themselves wants accounts to come from
their own process. Sign-up already captures the version of the terms the user consented to, which is the
part that would have been expensive to add later.

**Second factors: TOTP now, WebAuthn next, SMS never.** `MfaMethod` is a single-value enum today holding
only `TOTP`, and that enum is the extension point: **WebAuthn and passkeys are in general scope**, and
hardware keys arrive with them rather than as a third thing. **SMS is `OUT`** — it is the weakest factor
available, it carries a per-message cost and a delivery-failure mode, and recommending it would undercut
the enforcement §2 made mandatory. The Figma design is ahead of the code here on purpose: it already
contains the method picker, the WebAuthn prompt and the remove-a-factor confirmation that this answer
implies.

**Trusted devices are `OUT`.** "Remember this device for N days" is designed and unbuilt, and it should
stay unbuilt: §2 made two-factor **mandatory** for anyone who can configure a connector or change AI data
egress, and a trusted-device exemption is a hole in exactly that control. The reason it is refused is the
mandate, not the effort.

**Machine access follows §1.4.** An API token, a service account and a CI credential are all the same
question as a public API, and that is `LATER-RELEASE`, release **MVP-1.0.1**. None exists today.

**The session policy is promised rather than left emergent.** **Idle timeout and absolute lifetime are
configurable**, with defaults of **thirty minutes idle and twelve hours absolute**. A **forced global
logout is in scope** — it is the one session control that matters after a credential is compromised — and
a **concurrent-session limit is `OUT`**, so the same user may hold sessions on several devices at once.
The defaults are chosen for what this product is: a QA reading a page, thinking about a step or watching a
long run in another tab must not be signed out mid-task, which is what a short idle timeout would do to
the authoring loop §1.5 commits to. Stating a policy at all is most of the value here — a security
questionnaire reads an absent policy worse than a modest one.

→ See **§14.1**.

### 4.2 Projects

**Where it stands today** `[VERIFIED]` — a project is the unit of connector, environments, members, suites
and executions. **A project cannot be renamed or edited after creation** — no edit mutation or screen exists
anywhere, and the "General" settings tab was deliberately resolved to *hide* rather than build.

> **Decision — in scope. A project is one application under test.**
>
> **Rationale.** A project owns environments and a connector. That is the shape of an application, not of
> a team and not of a release train: the same team may test three applications, and the same application
> keeps its environments across every release. Defining it as the application is what makes the
> environment list and the connector belong where they already are.

**A project is one application under test**, in the customer's language. Not one test-management project,
not one team, not one release train — though a customer whose test-management project happens to match
their application will see no difference, which is the common case.

**A project is editable after creation: its name and description, not its key.** Today **nothing** about a
project can be changed — there is no edit mutation and no screen anywhere, and the "General" settings tab
was deliberately resolved to hide rather than to build. That is a defect of omission rather than a
decision, and this closes it: a name and a description are display text and must be correctable, while the
key is referenced by imported data and stays immutable.

**One connector per project**, which is the shape today; whether a project may draw from several sources is
decided with the rest of the connector scope in §4.4.

**Deletion is soft, and a hard purge exists on demand.** Ordinary deletion is the soft delete with a
recovery window that §2 settles, because a project holds scenarios that in the founding use case exist
nowhere else. Alongside it, an **administrative hard purge** is in general scope for the customer who is
legally entitled to demand irreversible erasure — and the resolution of the obvious tension is written
down rather than discovered later: **the audit record of the purge itself survives the purge**. Who asked,
what was erased, when. The subject is gone; the fact that it was erased is not.

### 4.3 Environments and secrets

**What it is.** Where tests point (base URL, database, device) and the credentials used to get there.

**Where it stands today** `[VERIFIED]` — credentials are encrypted at rest with the owning row's identity
bound into the ciphertext, write-only (never readable back), with two independent encryption keys the app
refuses to start without. `SecretProviderKind { LOCAL }` is a **single-value enum** that marks an
anticipated, unbuilt extension point for external secret managers. The design for it is already written
down in the code as a future `LocalEncrypted` / `ExternalRef` split.

> **Decision — in scope, and the customer's own secret manager is a supported source.**
>
> **Rationale.** "Paste your production database password into a form" ends the conversation with any
> customer who has a central secrets policy, and the extension point for avoiding it is already designed
> in the code. Taking it is the cheapest security credibility available anywhere in this document.

**The customer's own secret manager is a supported source.** Vault and AWS Secrets Manager are named;
anything else arrives through the same port rather than as its own feature. This is unusually cheap because
the work was anticipated: credentials are encrypted at rest today with the owning row's identity bound into
the ciphertext, write-only and never readable back, under two independent keys the application refuses to
start without — and `SecretProviderKind` is a single-value enum holding `LOCAL` whose intended successor
split is **already written down in the code** as `LocalEncrypted` and `ExternalRef`. The expensive part of
this decision has been paid for in advance.

**Rotation is in product scope**, for both credentials and signing keys. A customer security review asks
for it, and a store that cannot rotate is a store that accumulates.

**Running against production is supported, behind the gate of §1.3** — the protected-environment flag,
declared per project on the environment, covering production and pre-production. This section adds nothing
to that answer; it only records that the safeguard exists and is the one that applies.

**The no-leak promise is made, and it is conditional on two known gaps.** The product promises that a
credential never appears in a report, a log, an evidence artifact or a sync payload. One redactor exists
and is the single place that decides it (G23). But **two known gaps are open against that promise**, and
they are now named precisely: **step text** and **step warnings** are stored as written. Request and
response payloads, headers and error messages are redacted before anything is stored, but a credential
typed as a literal into a step — *I type "hunter2" into the password field* — is persisted verbatim in
three places. Until those two are closed the promise is a target rather than a claim: it may be written here, as the standard
the product holds itself to, and it may not be published to a customer. Closing those two gaps is the
condition for making it a public statement.

**A green "test connection" promises reachability and authentication, and nothing further.** The host
answered and the credential was accepted. It does not promise that the environment is correctly
configured, that the data is present, or that a run against it will pass — and **the console says so in
those words** rather than showing an unqualified success. A green mark followed by a failing run costs more
trust than a cautious one ever saves.

**File upload is supported, and the file is uploaded from the step while the scenario is configured.** The
earlier claim that the product structurally could not test file upload was **wrong**: the canonical step
`web.canonical.when#iUploadTheFileTo` exists, with translations in English, Spanish and French, a
`setInputFiles` implementation in the Playwright engine, and its own test. What was missing was never the
step — it was anywhere for the customer to put the file. That is what this decides: when a user configures
a scenario containing an upload step, **the console offers an upload control rather than a text field**,
the file is stored against the project, and the step holds a reference to it that is resolved to a real
path on whichever machine runs the browser. A per-project fixture store is therefore in general scope, and
it reuses the artifact storage that already holds evidence.

**Three properties of that decision worth recording, because they are what make it affordable.** The
**Gherkin pattern does not change** — the step still takes a string, and the upload control only decides
what fills it, so no frozen contract is touched (G34). The catalog already carries typed parameters, and
the type is a free string rather than an enumeration, so a file type is a new value rather than a
migration — traced end to end as G35 requires, from the catalog response through the console's types.
And resolving a stored reference into a real path at run time is the shape of a mechanism the product
already has for environment variables and secrets.

**Mutual TLS to the system under test is in scope, per environment.** It travels with the target, in the
same place as the base URL and the SSL configuration that an environment already carries.

**Test-data management is `OUT`, explicitly.** Seeded data, data pools and cleanup belong to the
customer's application, not to the thing that drives it. A scenario carries the data it needs through an
`Examples` table or through the fixture store above; beyond that, the product does not manage the state of
the system under test. This boundary is stated rather than left silent because it is the most natural
direction for scope to creep. **It is a boundary on a product feature, not on steps.** A scenario may write
to the system under test through its own steps — the database catalog runs SQL statements and scripts and
sets a table's contents, exactly as a Web scenario may submit a form — and a write against a protected
environment takes the higher grade like any other launch (§1.3). What the product does not offer is
managing that data for the customer.

→ See **§14.2 Secrets** and **§14.3 File storage**.

### 4.4 Integrations and connectors

**Where it stands today** `[VERIFIED]` — three adapters are built: Jira+Xray Server/DC, Jira+Xray Cloud,
and **TestRail**. The console offers **two**; TestRail is hidden by one configuration value, with its own
real-stack test suite maintained. `AZURE_DEVOPS` is an enum value with **no import adapter behind it**.

> **Decision — TMS products in general scope, in priority order:**
>
> 1. **Jira + Xray, Server/Data Center** — built and offered.
> 2. **Jira + Xray, Cloud** — built and offered.
> 3. **TestRail** — built and contract-tested; in scope and offered.
> 4. **Jira without Xray** — not built; the largest adjacent market.
>
> **Rationale.** The order is cost-ascending and market-descending at the same time, which is unusual and
> worth taking: the first three are already written, and the fourth is the only one that needs new work
> while also being the one that reaches the most customers.

**TestRail is in general scope, and three gates stand between it and a customer.** It is built and
contract-tested against its own simulator. Offering it opens three gates, not one: the backend allowlist
`offered-providers`, which defaults to `JIRA_XRAY,JIRA_XRAY_CLOUD` and needs a restart to change; a list
hard-coded into the console, which reads no configuration and so needs a source change and a rebuild; and
the landing copy, which denies TestRail in three languages. So this was never "do we build it" but "do we
support it", and the answer is yes. When the gates are opened is release planning rather than scope, and
this document does not decide it. Because the console's list reads no configuration, offering TestRail to
one customer and not another is impossible as built; the organisation-scoped settings of §4.17 are what
make it possible.

**The same mechanism gates the database engines, and an engine's proof bar is execution.**
`offered-database-engines` defaults to PostgreSQL, SQL Server and Oracle, and SQL Server and Oracle are
offered on evidence rather than on hope: since 2026-10-15 an integration test runs 51 of the 52 database
steps end to end against real SQL Server and Oracle instances, through the product's own API. MySQL and H2
*"stay out until proven"*, which is the posture this decision keeps — with one correction to the record:
MySQL's driver does ship to consumers, so it is excluded by the gate, not by absence. **Two proof bars,
stated out loud so the wrong one is never applied:** a connector is offered when it passes the shared
contract suite against its provider's simulator (G26); a database engine is offered when it has been
executed against end to end.

**`AZURE_DEVOPS` is removed as dead scope.** It is an enum value with no import adapter behind it, and
§1.1 already forbids ever claiming it. An enum value the product must never claim is a trap for whoever
reads the code next, not a roadmap; deleting it is the honest form of the same decision.

**Jira without Xray is in general scope**, as a distinct adapter. A large share of Jira users hold no Xray
licence, and today they cannot use the connector at all — which makes this the single largest market
question in this section. The connector port already supports the shape; what it needs is an adapter and
its pass through the shared contract suite (G26).

**One connector per project stays**, and a project drawing from several sources is `LATER-RELEASE`. The
canonical model already records a source system per case, so the data would permit it; the console and the
import flow assume one, and so does the sync filter. Deferring it costs nothing structural.

**Customer- and partner-written connectors are `OUT`.** The shared contract suite would make a third-party
adapter technically verifiable, so this is not refused for difficulty: it is refused because a
customer-written adapter is third-party code in the import path, reaching the customer's credentials and
our canonical model, for a market that has not asked. The connector set stays ours.

### 4.5 Import pipeline A — TMS connector (canonical model, review, sync back)

**What it is.** The flagship path: pull cases out of Jira/Xray, normalise them, a human reviews and approves
each one, run them, push results back.

**What we gain.** This is the only path that can send results back to the customer's TMS.

**Where it stands today** `[VERIFIED]` — the most carefully built logic in the repository. Re-import does a
three-way merge that **preserves reviewer edits** across source changes. Idempotent by content hash, so
re-importing an unchanged case is a no-op.

> **Decision — in scope. The flagship path, and the only one that reports results back.**
>
> **Rationale.** It is the most carefully built logic in the product and the only door that closes the
> loop to the customer's own system. The decisions below mostly ratify behaviour that already exists,
> which is the cheapest kind of scope answer there is.

**What may be imported, in product language.** A **test plan**, a **test set**, a **test execution**, and a
**single case by key**. A **folder is `OUT`**: a folder is a provider-specific organising idea, and §1.4
keeps the product out of the customer's structure. **Requirements and user stories are `OUT` as coverage
objects**, permanently — §1.4 settles that the product does no requirement coverage, which is also the
reason the requirement-coverage dashboard has nothing to populate it and should not exist. User stories
remain a **draft source** through the door of §4.6; that is a different use of the same object, and the two
are worth not confusing. **Typing an ID by hand is this same import.** §1.5 tier 1 lets a title and an ID
enter "from the connector, typed by hand, or a bulk list"; a typed key is resolved as a single case by key
through the connector, which is what gives it an external identity and lets its result be written back,
and a bulk list is the same import several times. Without a connector a typed ID has nothing to report to,
so the question of writing back does not arise.

**When the source case changes after approval, all three things happen, and that is deliberate.** The
canonical case returns to review, the previously approved projection stays runnable, and the drift is
reported. §1.3 settles each part separately: a changed source does not block execution; a superseded
projection staying active is correct behaviour to keep rather than a defect to fix; and the console says
the source moved without preventing the run. This document's earlier note that the combination was "almost
certainly not the intended rule" is withdrawn — it is the rule.

**When a case is deleted at the source, the copy survives and stops syncing.** The canonical case is
marked orphaned, result sync for it ends, and the scenario, its run history and its evidence are **kept**.
Deleting a case in the customer's system is not consent to destroy the record of what already ran against
it, and §2's soft-delete principle says the same about the customer's own assets. Removing it becomes an
explicit act inside A·Spectra, through the ordinary soft delete. The mechanism already exists and is
stronger than an end to syncing: a case deleted at the source, or taken out of the container it was
imported from, is **refused execution** on the path that writes results back, with its review status left
untouched as a separate human decision — and a later read of any container that lists the case again
clears the refusal.

**Unattended re-import is in scope, on one condition: it reports what it changed.** It was built and then
withdrawn, with the reason recorded in the code — an unattended re-import throws away reviewer work. That
reason is weaker after §1.3, which removed review as an enforced gate, but it is not gone: the three-way
merge **preserves reviewer edits** across source changes, and a silent scheduled job can discard that in
bulk with nobody told. So the schedule is permitted and the silence is not. A scheduled re-import must
report which cases re-queued, which edits were preserved and which drifted, through the drift reporting
§1.3 already puts in scope. **The condition is the notification, not the schedule.**

**The import size the product promises is 500 cases in one job.** A larger selection is split into several
jobs by the product rather than refused. A number that can be tested is worth more than an optimistic
silence, and this is the same class of commitment as the speed targets of §1.5.

### 4.6 Import pipeline B — Swagger/OpenAPI, Postman, user stories

**What it is.** Point the product at an API specification, a Postman collection or a written user story, and
it proposes draft test scenarios for a human to accept or reject.

**Where it stands today** `[VERIFIED], and this is the single most important fact in this section:**
this pipeline produces its content **using AI**, and **AI is off by default**. The code's own words describe
the fallback:

> *"When the capability is off or fails, every element still becomes a draft — named from the source, **with
> no steps and a zero confidence**. That is the declared deterministic fallback: an import that produces
> reviewable drafts with nothing invented in them, rather than an import that fails."*

**So in a default deployment, importing a Swagger spec produces empty drafts.** The design is principled —
it refuses to invent test steps — but the feature is present and inert.

Further: this pipeline **bypasses the canonical model entirely**. Drafts live in a separate bounded context
with a 3-state lifecycle (`NEW, ACCEPTED, REJECTED`) that records **no approver, no reason, no version
history**, and on accept it writes an executable row directly. It can never sync results back.

> **Decision — in scope, with two conditions: it converges on the canonical model, and the generative
> part does not appear where AI is off.**
>
> **Rationale.** The door is valuable and the design behind it is principled, but as it stands it is a
> second model with a weaker lifecycle, and in a default deployment it produces nothing. Both are fixable
> without giving up the door, and the fix for the second turns out to be deterministic extraction rather
> than more AI.

**This pipeline terminates in the canonical model.** §1.2 accepted that consequence: a draft no longer
writes an executable row directly, and the separate three-state lifecycle — `NEW, ACCEPTED, REJECTED`,
recording no approver, no reason and no version history — goes away. It was an inconsistency to resolve,
not an intended product difference. What a draft becomes on acceptance is a canonical scenario, carrying
the same review states, versioning and history as every other door.

**Accepting a draft promotes it, and it is then editable and runnable directly.** Suggestions are generated
against the canonical model, so acceptance moves a draft into that model rather than approving it for
execution; once there, the user may edit it or run it immediately, like any other scenario. There is no
second approval step, because §1.3 gates execution nowhere.

**The generative part is hidden where AI is off.** With AI off — the default — importing a specification
today produces drafts named from the source with **no steps and zero confidence**. The design is
principled; the code's own words say it refuses to invent steps rather than fail. But present-and-inert is
worse than absent: a customer importing a two-hundred-endpoint specification receives two hundred empty
drafts and reads the product as broken rather than as disabled. So the AI-dependent generation is offered
only where AI is enabled.

**And the door stays open in a default deployment, because a deterministic mode replaces it.** For
Swagger/OpenAPI and Postman — structured sources — a request step with its status-code and schema
assertions is **extracted** from the specification, not generated. That needs no model, so it works with
AI off, which is every default deployment. The user-story source has no such structure and therefore no
deterministic mode; it is part of the generative path only.

**Documented response codes are the source's expected result, not an invented one.** This was the sharpest
question in the section, and the answer turns on a distinction worth stating exactly. When a specification
says an endpoint returns `200` with a given schema, asserting that is **extraction**: the customer wrote
that contract, and nothing is guessed. The AI rule — never invent test data, credentials, URLs, selectors
or expected outcomes, which §1.3 left untouched in full — is therefore not engaged. Such a case **is
runnable**, and it is **labelled as derived from a specification**, so a reviewer knows its assertions are
contract-level rather than behavioural. Anything beyond the contract stays empty: a business expectation,
such as the wording of an error message, is not in the specification and must be written by a person. A
model may say that an expected result is missing; it may never supply one.

**The confidence score is not product-facing.** `OUT`. A number whose meaning cannot be explained to a
reviewer in one sentence implies a rigour it does not carry, and inviting trust in it is worse than
showing nothing. It may exist internally for evaluation; it is not shown to a user.

**Drafts are never pushed back into the customer's system as new cases.** §1.4, permanently: results and
evidence only, never steps and never Gherkin. The apparent tension with "the test management system stays
the source of truth" dissolves in §1.4 as well — that posture is optional and is not the product's
default.

**The specification versions supported are the ones that parse today.** Swagger/OpenAPI **2.x and 3.x**, in
JSON or YAML, and **Postman v2.1** collections, each up to 5 MB. Nothing beyond that is claimed: Postman
v2.0 is not, and any newer OpenAPI feature the parser does not read is not either. Removing 2.x would
withdraw a working capability for no gain, so it stays.

### 4.7 Import pipeline C — local `.feature` files

**Where it stands today** `[VERIFIED]` — enters the canonical model correctly and inherits the full 7-state
review, versioning and re-import idempotency. **But it can never sync results back**, by deliberate design:
a locally authored case has no external identity, and inventing one could hit a real key in another project.

> **Decision — in scope, and migrating an existing suite is an explicit use case.**
>
> **Rationale.** The capability already works properly and it answers a real buying question. What it
> needs is not more engineering but an honest boundary, so that a promise made in a sales conversation is
> the same promise the product keeps.

**Migrating an existing Cucumber suite is a promised use case, and the promise is bounded in the same
sentence.** A `.feature` file enters the canonical model correctly and inherits the full seven-state
review, versioning and re-import idempotency — so the scenario text, its structure, its tags and its
examples all survive the move. What does not come with it is the automation: **the customer's step
definitions are theirs and the catalog's are ours**, so their steps must be re-expressed against our
catalog. The honest value is that the authoring work survives, not the code behind it. This is the mirror
image of the export promise of §1.1 and is to be worded with the same care, in the same breath, rather
than discovered in week three of an evaluation.

**What this door cannot do, by deliberate design.** A locally authored case has no external identity, so
it can **never** sync results back, and inventing an identity for it could collide with a real key in
another project. That is a property to state to the customer, not a gap to close: a team that needs
results in their test management system imports through the connector door of §4.5 instead.

**A `.feature` file whose steps do not match the catalog is imported, and cannot run until they do.** It
enters the canonical model with its unresolved steps marked, and the executability rule of §4.9 — every
step resolves in the live catalog — keeps it out of `EXECUTABLE` until they are re-expressed. Rejecting the
file would make the migration this section promises impossible; importing it as inert text would hide the
problem. Marked steps are also the work queue §1.5 tier 2 asks for: they show exactly what is left to
re-express.

### 4.8 Ingestion path D — native authoring and Gherkin paste

**What it is.** Writing a test case directly in the product's own editor, or pasting Gherkin text into a feature.

**Where it stands today** `[VERIFIED]` — **this is a fourth way executable content enters the system, and it
was not previously counted.** Pasting Gherkin goes through a *different repository* than the scenario editor,
so it bypasses even that path's own guard. Neither has a review gate, a canonical case, or a sync path.

> **Decision — in scope, as a first-class door.**
>
> **Rationale.** §1.2 settles that this is the door the founding use case walks through — a QA with a
> title and an ID — and that it is open to anyone, with or without a test management system. Nothing
> here is a second-class path to be tolerated.

**A natively authored or pasted scenario needs no review before it runs.** §1.3 settles it: review is a
convention of the console, not an invariant of the backend. And §1.2 rejected the two consequences that
the opposite answer would have forced — roughly twelve thousand lines of console and one bounded context
written off, and every natively authored scenario becoming unlaunchable. Authoring is first-class, so
neither applies.

**Natively authored cases are never pushed into the customer's test management system.** §1.4, as a
permanent boundary: results and evidence only, never steps and never Gherkin. A customer may use the
product with **no such system connected at all**, unconditionally — and §1.1's founding case is exactly
that one: the step-by-step lives here because here is where it runs, and the customer's system never held
it and does not need to.

**One thing to resolve rather than to decide.** Pasting Gherkin goes through a different repository than
the scenario editor, so it bypasses even that path's own guard. That is not a scope question: §1.2 already
accepted the consequence that paste is re-routed through the one stored model. It is work on the way to
that answer, recorded here so it is not mistaken for an open choice.

### 4.9 Canonical review (7 states)

**What it is.** The human approval lifecycle that the product's value claim rests on.

**Where it stands today** `[VERIFIED]` — a 7-state machine where each transition is typed as human- or
system-triggered, audited with actor, reason, source and target state, under optimistic locking, with
approval tied to a source content hash. It is the strongest piece of domain logic in the product.

**The two "gaps" this section used to name are not gaps.** It previously recorded "no approver grade" and
"no revocation of superseded projections" as cheap to close. §1.3 settles both the other way: with no
enforced approval there is nothing for an approver grade to gate, and a superseded projection staying
launchable is correct behaviour to keep rather than a defect to fix. Neither is outstanding work.

> **Decision — in scope. Seven states, and the two human ones mean different things.**
>
> **Rationale.** The lifecycle is the strongest domain logic in the product and it stays. What needed
> deciding was not whether to keep it but what two consecutive human states mean now that §2 has removed
> the role difference that would once have justified them.

**The seven states, in the product's own words.** **`IMPORTED`** — it arrived from a source and nothing
has touched it. **`NORMALIZED`** — the platform has mapped its text onto the canonical model.
**`DRAFTED`** — a person has written or completed the step-by-step. **`REVIEWED`** — a person has read it
and judged the content correct. **`APPROVED`** — it is released for use in runs and suites.
**`EXECUTABLE`** — it satisfies the executability rule below and can be run. **`REJECTED`** — a person
has refused it.

**`REVIEWED` and `APPROVED` are kept apart on purpose, and the distinction is stated so it survives.**
One is about **correctness**, the other about **readiness**: a case may be correct and not yet released,
and the two judgements are made at different moments even when the same person makes both. That matters
because §2 gives the product no approver grade, so nothing but this definition keeps the two states from
collapsing into a step everybody clicks through without reading.

**Who may move a case between states is already settled by §2: anyone with write access.** No approver
grade, no named reviewer role, no per-project approver list. Each transition is typed as human- or
system-triggered and audited with actor, reason and the states it moved between, under optimistic
locking — so the record says who acted, which is the claim the product makes. **That record exists but
cannot yet be relied on**, and the gap is stated plainly: it is written and never read — nothing in the
product can display it — and it is deleted together with its case, so the evidence of who approved what
disappears with the thing approved. §4.17 closes both: every review transition is additionally written to
the product's audit trail, which survives the deletion of its subject.

**What makes a case executable is the rule the product already implements, adopted verbatim.** A
non-blank title; layer and BDD mode set; at least one step; **every step resolves in the live catalog, is
available, belongs to the case's layer and carries exactly the parameters its pattern declares**; **at
least one THEN step**; no blocking flag; and no open ambiguity question. A step's expected text is
optional.

**Two things are deliberately not part of executability, and must not become part of it.** **Approval is
not**, because review is its own state machine and folding it in is exactly the consequence §1.2
rejected. **Environment availability is not**, because an environment is chosen at launch rather than
baked into the case — which is what lets one case run against several environments, as §1.5 requires.

**An approved case may be edited, and a change to its steps returns it to `DRAFTED`.** A change to its
title or description does not, mirroring the name-versus-key distinction of §4.2. This needs no new
machinery: approval is already tied to a source content hash, which is precisely the ability to tell that
what is there now is not what was approved. And it does not stop anything running: returning to `DRAFTED`
leaves the existing projection launchable, exactly as a changed source does under §1.3.

**Bulk approval is in scope, and the audit records each case individually.** Approving two hundred
imported cases one at a time is the kind of friction that makes a team abandon the review flow entirely,
which is a worse outcome than approving them together. The claim the product makes is that a person took
the action, and one click on two hundred cases keeps that claim true as long as the record carries the
actor, the timestamp and the version for each of them — which it does.

**The reviewer's identity, the timestamp and the exact version approved must be visible later**, per case
and permanently. The bulk-approval answer above depends on it, and it is what makes the claim that a person
acted checkable rather than asserted. It is visible through the audit trail of §4.17, since the review
record itself is not yet readable.

### 4.10 Step catalog

**What it is.** The vocabulary of test actions the engine can perform — the customer's first real look at
what the platform can do.

**Where it stands today** `[VERIFIED]` — **515 steps** across four layers (HTTP 141, Web 186, Mobile 136,
Database 52) in 62 components, localised in English, Spanish and French, with the step patterns frozen as a
contract under an automated guard. **14 steps are hidden from the product**: 6 file-dependent ones (pending
a file store) and 8 conditional "if present" ones removed because they assert nothing.

> **Decision — a published, versioned product differentiator, and the definition of our capability.**
>
> **If there is no step, we do not claim it.**
>
> **Rationale.** One rule settles GraphQL, SOAP, gRPC, WebSocket and every future question of the same
> shape, permanently and without arguing them case by case. It is also the most honest capability
> statement a product like this can make, because a customer can read the catalog before buying and check
> it themselves. The price is giving up aspirational marketing — the product may never say "tests any
> API" again — and that price is worth paying.

**The catalog is the product's capability statement.** 515 steps across four layers — HTTP 141, Web 186,
Mobile 136, Database 52 — in 62 components, localised in English, Spanish and French, with the patterns
frozen as a contract under an automated guard. Published and versioned, not internal vocabulary. **It is
also the enforcement mechanism for the ceiling §1.5 names**: the catalog is the ceiling, the step request
of §1.5 is the door through it, and nothing has to be adjudicated feature by feature.

**Subtracting capability from the engine is a permanent policy, and the product decides — never the
customer, and never silently.** The rule is that a step is offered only when the product can honour it
end to end, which is fail-closed behaviour applied to the catalog itself rather than only to execution
(G20). What changes is the second half: **a hidden step is never silently absent.** Either it does not
appear at all, or it appears with the reason it is unavailable. Today a customer sees the catalog and
cannot use fourteen of its steps with no stated reason, which reads as broken rather than as curated, and
that combination ends here.

**And the hidden set shrinks by itself.** Of those fourteen, **six were file-dependent, pending a file
store** — and §4.3 puts the per-project fixture store in scope, so those six stop being hidden for a
reason that no longer exists. What remains is the eight conditional ones, which are excluded on principle
rather than on circumstance. The arithmetic is worth having: the catalog holds 515 steps and the product
shows 501.

**"If present" conditional steps are excluded as a determinism policy, permanently.** A step such as
*"the element is visible if present"* passes whether or not the element exists, so it is not a verdict,
and a product whose entire job is to decide pass or fail has no business offering an assertion that
cannot fail. This belongs in the product's determinism story rather than in a list of omissions, and it
is a real differentiator: most tools in this market let a user write assertions that are incapable of
failing. And there is a second reason, specific to this product: **six of the eight are THEN steps.**
§4.9's executability rule requires at least one THEN step, and a conditional THEN step would satisfy that
rule while asserting nothing, so hiding them is what keeps the rule meaningful. The other two are WHEN
steps that silently do nothing.

**A customer cannot add a step to the catalog.** §1.5 settles it: macros are the customer-side extension
point, and a primitive is only ever written by us. Step patterns and step ids are the executable identity
of every stored scenario and a frozen contract (G34), so they stay the product's to author.

**The rule binds the engine's own advertisement, starting now.** The HTTP plugin reports `graphql` and
`soap` as capabilities with no step, client or parser behind either, and a test locks that claim in.
Under the rule above the advertisement is removed, whether or not either protocol is ever built. GraphQL
and SOAP are the only false advertisements: gRPC, WebSocket under test, NTLM, Digest, AWS SigV4 and HAR are
all absent, and none of them is claimed.

**A customer can request a step, through a committed path and without a committed turnaround.** A request
is acknowledged, triaged, and answered with either a macro that solves it today or the release it is
targeted for. What is deliberately not promised is a number of days: committing to a turnaround for work
whose size cannot be seen in advance converts a support process into a liability, and the honest-ceiling
answer of §1.5 does not require one.

**The catalog is localised, and that is a promise — for display text only.** Step names and descriptions
are published in English, Spanish and French, and that is part of the capability statement. Step
**patterns** are never translated: they are the executable identity of every stored scenario (G34), so a
scenario written against the Spanish display reads and runs exactly like one written against the English.

**Step deprecation is a visible lifecycle.** A withdrawn step is never silently absent, by the rule above;
it is marked deprecated with the step that replaces it, which is what the catalog contract already
carries — a deprecated flag and a replacement step id. A scenario using it keeps running, and the console
says so.

**Still open:**

- `[non-blocking]` **Owner decision — the deprecation notice period.** How long a deprecated step keeps
  working before it is removed. *(Settled: deprecation is visible and names a replacement. Only the period
  is a support commitment, and it is the same kind of promise §4.10 refuses for step-request turnaround.)*

### 4.11 Macros (reusable step sequences)

**Where it stands today** `[VERIFIED]` — project-scoped, expanded before the test is generated so the engine
only ever sees catalog steps. Renaming or deleting a referenced macro is refused — **but changing its body
is allowed and silently re-expands into every scenario that uses it on its next run.** Macros have no version.

> **Decision — in scope, versioned, and shareable across the organisation once they are.**
>
> **Rationale.** §1.5 makes macros the product's primary answer to anything its catalog cannot say, so
> they will carry real weight rather than being a convenience. A mechanism that load-bearing cannot also
> be the product's main silent-change vector, and sharing an unversioned macro is the worst of the
> available combinations.

**Macros are versioned.** A scenario references a macro **version**; editing a macro creates a new
version; and a scenario moves to it deliberately rather than silently. What this replaces is the sharpest
surprise in this part of the product: today renaming or deleting a referenced macro is refused, but
**changing its body is allowed and silently re-expands into every scenario that uses it on the next run**
— so one edit can change the behaviour of eighty scenarios with no edit recorded against any of them and
nobody told. §12 already lists this as cheap now and expensive later, and two decisions have since made
it more load-bearing, not less: macros are the answer to the catalog ceiling, and §4.9 ties approval to
content — which a macro body change alters without ever touching the scenario, slipping straight past it.

**If full versioning is ever staged, the minimum is the report.** Editing a macro body must at least say
which scenarios it affects, the same shape as the conditioned re-import of §4.5. Silence is the part that
is refused, not the edit. That minimum is nearly free: the product already counts which scenarios
reference a macro, and already refuses a rename or a delete while any do — it simply never consults that
count when the body changes, which is the one edit that alters behaviour.

**Macros are shareable at organisation level, and the sharing depends on the versioning.** Project scope
alone would mean every project re-inventing the same macro, and would leave the suggestion mechanism of
§1.5 with no shared library to draw on. Shared and versioned is the combination that works; shared and
unversioned would multiply the silent-change problem by the number of projects that use it, so sharing
follows versioning rather than arriving before it.

**Expansion stays where it is.** Macros are expanded before the test is generated, so the engine only
ever sees catalog steps — which is what keeps Core free of any concept of a macro, and it is not changed
by either decision above.

### 4.12 Execution

**What it is.** Actually running the tests.

**Where it stands today** `[VERIFIED]` — runs happen **inside the backend process**: every API, Web and
Database run, and every Mobile run without an agent, executes in the backend's own JVM; only a Mobile run
with an agent goes out of process. Capacity is 4 concurrent plus a queue of 50 **per instance**; beyond that
a launch is refused with HTTP 429 and a `Retry-After` hardcoded to five seconds whatever the queue holds,
and nothing measures how many slots are in use. The four slots are blocked threads rather than running
engines, so a Mobile run on a remote agent holds one while doing nothing locally. A deploy or restart
**loses every in-flight run** that does not finish within the 120-second shutdown drain, and the next sweep
— by whichever instance runs it, the restarted one included — marks them errored two to three minutes
later. Cancellation is cooperative and honest: remaining steps report as skipped rather than inventing
failures. And Core's own documentation states, among its inviolable rules, that the backend in production
must never run tests in-process — the shipping deployment contradicts it, including the deployment that
most explicitly declares itself production.

> **Decision — in scope; every launch path stays, and execution moves out of process.**
>
> **Rationale.** Running is the product, and almost everything here either follows from a decision
> already taken or describes behaviour the code already has. The one thing that changes is where a run
> executes, and §3 has already decided it: out of process, on the customer-side runner — which is also
> what the contradiction with Core's own rule requires.

**Every launch path stays.** There are three, and they converge on one launch service, so they share
authorisation, capacity and lifecycle: a direct launch, the execution-plan path, and the legacy bulk
execute of a suite or a feature. §1.3 keeps them as they are — the review gate stays on the plan path and
is not moved into a shared validator. One defect sits in the third: the bulk execute passes no layer, no
Web configuration, no reporting configuration and no HTTP engine, so **a suite or feature run cannot
request a report or choose a browser**. Under §1.5's non-negotiable browser choice, the bulk path must
carry the same runtime configuration as the others.

**Execution moves out of process.** Core's rule that production never runs tests in-process becomes the
product's rule, and the customer-side runner of §1.5 and §3 is how it is met: the same component that
reaches private applications, serves the element inspector, keeps a run alive across a deploy and holds
the warm browser pool. In-process execution stays what Core documents it to be — the path for tests, CI
and a single host.

**Scheduled and unattended execution is in scope, and a schedule keeps the console the only way in.**
§1.5 tier 3 lists scheduled runs as a capability, and the read half already ships: a run records whether
it was scheduled and the console carries a translated label for it, while every launch path hardcodes that
flag to false and no scheduler exists — a contract field with no producer (G7). The resolution that keeps
§1.3 intact: **a schedule is created in the console by a person who chooses what it runs**, so the console
remains the only way a run comes into being, and §1.3's condition still holds. Every scheduled run
records that it was scheduled and who created the schedule. A schedule against a protected environment
requires the higher grade, checked when it is created and again each time it fires, so a schedule stops
the moment its creator loses the grade. Re-import scheduling is a complete subsystem already built and
switched off; §4.5 sets the condition for switching it on.

**A trigger from an external system follows §1.4.** A public API, a webhook and a CLI are
`LATER-RELEASE`, release MVP-1.0.1 — and that, not the scheduler above, is the event that expires §1.3.

**Parallel execution is a promise, and the part promised is not built yet.** §1.5 tier 2 makes it a
viability capability. Three things exist today at three granularities: **parallel HTTP requests within one
scenario** exist, as seven concurrency steps; **parallel runs** exist, four per instance; **parallel
scenarios within one run do not exist** — the switch for them is ignored in-process and refused by the
agent, whose sessions are per device. The promise is the third: the scenarios of one run execute in
parallel, with Mobile bounded by its devices. The **capacity number is not published**: it is a deployment
setting that scales with instances under §3, and the configuration itself calls the current four
provisional until measured.

**A run's maximum duration is two hours, configurable, and exceeding it ends the run as `ERROR`** — never
as a failure, because a timeout is not a verdict about the customer's application. That is the behaviour
already implemented.

**Cancel is a guarantee about the run and best-effort about the thread.** The run always reaches a
terminal state, the remaining steps report as skipped rather than as invented failures, and a Core thread
that ignores the cancel is released after a ten-minute grace with any late results discarded. That is what
a user needs from a cancel, and it is already true.

**Retry inside a run is `OUT`; a rerun is a new execution, and both are kept.** A retry loop inside the
engine is how a suite becomes permanently green and permanently meaningless — the same shape as an
assertion that cannot fail. A rerun of a whole execution or of one scenario, which the console already
offers, is a new execution: in the customer's system the later result replaces the earlier and the synced
comment says it was a rerun, while in A·Spectra both executions remain, which is what flakiness detection
reads. §1.1's severity rule aggregates within one execution and does not reach across a rerun.

**Flakiness is detected and labelled; quarantine is manual.** Detection is a §1.5 tier 3 capability and
already ships as an index on the project dashboard. **Automatic quarantine is `OUT`**: a machine removing a
test from the suite is the product deciding what gets tested, one step from deciding the verdict. A person
may quarantine a scenario, and the label is visible wherever the scenario appears.

**The case version a run executed is recoverable and re-launchable; a byte-identical run is not promised.**
§1.5 tier 3 puts versioning and revert in scope and §1.3 keeps superseded versions launchable, so what ran
six months ago can be run again. The run itself cannot be repeated byte for byte, because §4.3 disclaims
the state of the system under test. One fact turns this from a property into work: an execution records
only which scenarios it ran, not their versions, so **recording each case's version on the execution is
required** — and the same record fixes the result written back to the customer, which today names the
version current at sync time rather than the one that ran (§4.14).

### 4.13 Evidence and reporting

**What it is.** The proof of what happened: screenshots, request/response detail, logs, a report.

**Where it stands today** `[VERIFIED]` — evidence files are written to the backend's **local disk**; HTML
reports are stored in a database column, without a size cap, and are overwritten in place when
regenerated. Request and response payloads, headers and error messages are redacted before storage —
**step text and step warnings are not** (§4.3). Retention is two 30-day windows for evidence — screenshot
files and HTML reports — plus a third for outbox rows, which are not evidence, and an exemption keeps the
evidence of a run whose sync job is stuck, for up to 90 days. **The largest body of evidence has no
retention at all**: each run's step results, carrying the request and response summaries, the step text
and the errors, are never purged. **With two instances, the second one answers "not found" for a
screenshot the first one wrote** — the configuration says so explicitly. Only three artifact types are
ever written: a screenshot, a failure detail and an AI summary.

> **Decision — in scope, and evidence is tamper-evident.**
>
> **A·Spectra records what the engine observed at each step of a run, stores it unaltered, and can show
> that it has not changed since.**
>
> **Rationale.** "Proves what happened" is either a feature or marketing, and the difference is whether
> the product can show a third party that the record was not edited. Most of the mechanism is already
> built and simply unused, which makes this the cheapest credibility in the section.

**Tamper-evidence is in scope, and half of it is built.** A SHA-256 is already computed and stored for every
artifact file, and nothing ever checks it. What completes it: the hash is **verified whenever an artifact is
read**, the HTML report is **hashed and versioned** instead of overwritten in place, and each execution
carries a **manifest** of its artifacts' hashes. That is the difference between a good report and an audit
artifact.

**The report must carry the evidence it summarises.** The HTML report — the artifact attached to the
customer's system — today contains **no screenshots** and a stack-trace slot that is always empty, because
nothing writes either into the data the report is built from; the screenshots and the traces live
elsewhere, and the traces expire at thirty days. A report presented as the record of a run must include
the run's evidence.

**The artifact types captured are the five §1.5 names, plus one.** §1.5 tier 2 enumerates the mandatory
set: **the screenshot, the HTTP request and response, the SQL, the mobile screen, and the logs.** Today only
the screenshots — which also cover the mobile screen — exist in full; the HTTP evidence is a 500-character
redacted summary of each request and response rather than the payload; and **the SQL and the logs are not
captured at all.** Completing the five is required work. To them this section adds a **DOM snapshot on a Web
failure**, which is cheap from the engine and is what makes a selector failure diagnosable without
reproducing it. **Video, HAR, traces and device logs are `OUT`**: they multiply storage by an order of
magnitude for diagnosis the six above already support.

**Evidence retention: thirty days by default, set per organisation within a vendor range.** The
organisation administrator sets it and the change is audited; on self-hosted the range is the customer's
own. **Retention covers the detail of a run, not only its files.** The verdict history of a scenario — what
ran, when, and with what outcome — is kept, because §1.5 tier 2 requires knowing whether a scenario ever
passed; the **detail** of a run — step text, request and response summaries, errors and warnings — is
evidence, and expires with it. Today the detail never expires, so a thirty-day window offered as a privacy
property would not hold. The exemption for a stuck sync is kept exactly as implemented.

**Evidence survives the ordinary deletion of its project, its case and the user who ran it.** A project is
soft-deleted (§2), a case deleted at the source keeps its history (§4.5), and deleting a user must not
erase who ran what — which today it does, through the `ON DELETE SET NULL` defect §2 names. Evidence does
not survive an administrative hard purge or its own retention expiry, and the purge's own audit record
survives either way (§4.2). Two facts complete the picture: a purge that fails to delete a file leaves the
file behind with its row already gone, so "deleted after thirty days" is today a claim about rows rather
than bytes; and the column a legal hold would use already exists and is never set. A legal-hold feature is
`OUT` until a customer's obligation calls for one.

**JUnit XML is in scope; PDF is not.** JUnit XML is the one format every CI tool reads, and it pairs with
the external trigger of §1.4. The schema already anticipates it but allows one report per execution, so it
needs a schema change to sit beside the HTML report. **PDF is `OUT`**: the HTML report and the per-execution
evidence export of §3 already serve the human-readable record.

**Still open:**

- `[BLOCKING]` **Owner decision — the evidence retention range.** The vendor floor and ceiling within which
  an organisation sets its retention, and any minimum the law imposes where the product is sold.
  *(Settled: thirty days by default, organisation-set, audited, and covering a run's detail as well as its
  files.)*

→ See **§14.3**.

### 4.14 Result sync (write-back)

**What it is.** Sending results back into the customer's Jira/Xray.

**What we gain.** It closes the loop for a team whose cases live in a test management system: the result
appears against the case it belongs to. §1.2 settles that this is a convenience rather than the product —
valuable, not constitutive.

**Where it stands today** `[VERIFIED]` — **three conditions must all hold** for a result to sync: the run
came from an execution plan, the scenario is a canonical projection, and its source system matches the
connector. Consequence: **only connector-imported cases launched through the plan path ever sync.**
Spec/story drafts, `.feature` imports and natively authored scenarios never do, and neither does an
imported case run from the editor or from a suite. And today **every skip is silent**: a natively authored
scenario is dropped without a note, and the exclusion note built for a source-system mismatch is discarded
by the sync worker before it is written — a test that mocks that one step keeps the defect green.

> **Decision — in scope: one result per case, saying what was tested, never silent about what it skips.**
>
> **Rationale.** A result written into the customer's system is the one output a third party reads
> without opening A·Spectra, so it has to be correct and complete about itself. §1.1 and §1.4 fix most of
> its shape; this section adds that a result must say what it is a result of, and that nothing is left out
> without saying so.

**What is written back: one result per source case, its comment and its evidence.** One aggregate result
per case, keyed by the id it came from, the most severe outcome winning (§1.1); a comment carrying the
failed step, the sanitised error, a link to the execution and the identifiers; and the evidence
attachments, within the connector's file limits. Never steps and never Gherkin (§1.4). **A defect is never
created** — defect management is a permanent non-goal (§1.4), and creating one is writing content into the
customer's system. Linking an existing defect a person chose is `OUT` until a customer asks.

**Which runs write back.** A run of syncable cases writes back when it is launched as a **recorded run** —
from the execution plan, a suite or a feature, whichever screen launched it. A run launched from the editor
while a scenario is being authored is a **draft run** and never writes back: §1.5's write-run-fail-fix loop
would otherwise post a result against the case on every iteration. Today only the plan path writes back, so
suite and feature runs of imported cases must be brought up to the same rule.

**A result says what was tested.** An optional **build or version under test** is captured at launch — set
by the user or passed by the trigger — stored with the execution and included in the synced comment.
Optional, because a product that blocks a run for a missing label gets the label faked; present, because a
result without it is weak evidence. It is supplied, never inferred, and never produced by a model (§1.1).
And the case version the comment names must be the one that ran, recorded at run time (§4.12), not the one
current when the sync job finally succeeds.

**A failed sync is an operational problem, never a failed run.** The verdict is Core's and is final; a sync
that cannot reach the customer's system says nothing about their application. The three state machines stay
separate, as §1.3 and §4.9 require. What a failed sync owes the user is to be visible and retryable, and
both already ship.

**Locally authored and spec-derived cases never write back** — they have no external identity (§4.6, §4.7,
§4.8).

**Nothing is skipped silently.** Every scenario in a run that will not write back is reported with its
reason — authored in A·Spectra and so without an external identity, a different source system, or a draft
run — in the exclusion note the sync already builds, and **the console says so before the run, not after.**
The mechanism exists for one of those reasons already; what defeats it in production is a single line in
the sync worker that discards the note, which is a defect to fix rather than a scope question.

**Write-back can be turned off, per project.** A read-only integration imports and runs without writing
anything into the customer's system. §1.4 already makes the source-of-truth posture optional per project,
and a team evaluating the product will want to watch it run before it writes into their Jira — which also
makes this the lowest-friction way into a trial.

### 4.15 Dashboards and analytics

**Where it stands today** `[VERIFIED]` — a global dashboard, admin-only and **cross-tenant** — its query
takes no tenant at all — beside a second cross-tenant admin surface for deprecating components, and a
per-project dashboard with a flakiness index and a requirement-coverage card. The sidebar offers the global
dashboard to **every** user, and the per-project dashboard is reachable from exactly one place: the global
one. The requirement-coverage card is a full vertical — a table, a service, a link-writing API, a field on
the dashboard response, console types, the card, its test and keys in three languages — and **it is never
drawn**: the response hardcodes it to empty. Analytics read pre-aggregated tables that no retention job
ever purges, so a trend can outlive the evidence behind it.

> **Decision — one dashboard per persona that needs one, and requirement coverage removed.**
>
> **Rationale.** §2 fixes who has an account and what each comes to do, and §4.5 has already ruled
> requirement coverage out permanently. What remains is to make each dashboard reachable by the person it
> serves — which today, for the one that matters most, it is not.

**The project dashboard is for the QA author and the QA lead, and it is reached from the project.** It
supports four decisions, all of them §1.5 capabilities: what is failing and why, what is flaky, what is
still left to implement, and whether a scenario ever passed. It must be reachable from the project itself,
scoped to it — because once §2 makes the platform role vendor-only and §3 makes the organisation the
tenant, **no customer can reach any dashboard at all** through the path that exists today. That makes this
a sequencing constraint on §2's rename, not a separate improvement.

**The global dashboard is a vendor operational surface, never a customer feature.** It may span customers
because it belongs to the vendor support role of §2; customers never see it, and the sidebar stops offering
it to them. The component-deprecation surface beside it is the same kind of thing. The customer
organisation administrator's view across projects is the audit and job view of §4.17, not a second
analytics dashboard.

**Requirement coverage is removed, with everything behind it.** It is not a product promise — §4.5 rules
requirements out as coverage objects permanently. Because removing a screen while leaving its endpoints is
how a product acquires an undocumented API, the removal is complete: the card, its types, its test and its
keys in three languages; the dashboard field, the coverage endpoint and the link-writing endpoint; the
service; and the `requirement_links` table, dropped in a forward migration with its undo, once it is
confirmed to hold no customer data.

**Analytics ranges are not capped by evidence retention, and the console says so.** The console offers
seven, thirty and ninety days. Because the aggregates are domain computations that are never purged, a
ninety-day trend is answerable while the evidence behind its older half is gone — **the numbers survive,
the proof does not.** That is coherent rather than broken, as long as a figure older than the evidence
window is shown as computed rather than as substantiated.

→ See **§14.8**.

### 4.16 AI

See **§8** for the full treatment.

### 4.17 Administration, notifications, audit and retention

**What it is, in plain words — and this is the group the owner asked to have explained without jargon:**

- **Durable jobs.** When the product has to talk to something outside itself (Jira, a device, an AI
  provider), it first writes a row saying "this needs doing", then does the work, then records the outcome.
  **What we gain:** if the server restarts mid-import, nothing is lost and nothing is done twice. Without
  it, a crash in the middle of importing 500 cases leaves the customer with an unknown, unrepeatable state.
- **The outbox.** The single mailbox through which every "something finished" event leaves the system.
  **What we gain:** events can never be lost or duplicated, and anything that wants to listen later (a
  warehouse, a webhook, a notification channel) plugs in at one place instead of ten. That is the
  intended shape; today the outbox carries a single event type, and the decision below is what makes it
  true.
- **Retention.** Automatic deletion of old evidence, reports and logs on a schedule. **What we gain:** the
  disk does not fill up and the customer's data does not live forever by accident — which is also what a
  privacy request requires. Today there are fourteen separate retention mechanisms, nine of them
  configurable and five hardcoded.
- **Rate limits.** Caps on how many requests one user, one address, one customer or one launch can make.
  **What we gain:** one customer cannot starve another, and a runaway script cannot take the service down.
- **Audit trail.** A permanent record of who did what. **What we gain:** it is the record behind the one
  guarantee §1.3 leaves standing — that the product can always say who, or what, acted. If it cannot,
  the guarantee is a slogan.

**Where it stands today** `[VERIFIED]`:
- **No notification channel exists, and nothing is sent when a run finishes.** No webhook, Slack or Teams
  anywhere; the "email me the result" code exists and is reached only from tests. Email itself is a working
  subsystem — activation, password reset, welcome and email-change notices — off by default in every
  profile behind one switch. Every mail send runs on a worker pool that **discards a task silently** when
  it is full.
- **The outbox carries one event type.** Any other is marked dead on arrival, and delivery is an in-process
  call to five hard-wired listeners, so there is no consumer registry to plug a channel into.
- **Six audit-shaped stores exist with incompatible shapes, and exactly one is readable over HTTP** — the
  TLS-mode log, which no screen consumes. The review trail is never read and is deleted with its case; one
  AI trail is an orphan with no writer; who launched a run lives on a mutable row that loses its actor when
  the user is deleted; the credential trail writes the actor into the column that means the target.
- **Permission grants and revocations are not audited at all**, and the two log lines they produce do not
  record who made the change. The audit vocabulary has no permission action to record one with.
- **Retention:** fourteen mechanisms, nine configurable and five hardcoded — the worst of them a 90-day
  delete of the login and credential trail with no configuration key. Two trails have no retention at all.
- **Purge and export exist only as an operator script** outside the application, with no API and no audit
  row; its own header warns that deleted rows survive in database dumps until rotation, and backups are kept
  fourteen days.
- **About 27 deployment-wide switches**, all environment variables, each needing a restart. The console's
  notification bell shows customers a roadmap of unbuilt notifications, in three languages.

> **Decision — one audit trail, a durable notification path, and retention in three classes.**
>
> **Rationale.** Three actions the decided sections make load-bearing are recorded nowhere — who was
> granted project write (§2), who changed AI data egress (§2), and who launched against a protected
> environment (§1.3) — and §12.10 is exact that an audit trail cannot be backfilled. The rest follows
> from the workspace rule that every crossing of a process boundary is a durable job: a notification is
> one, and today it would be fire-and-forget.

**What is audited, in three tiers defined by who reads it.** The rule is one row per decided action,
recording the before and after of what changed, redacted through the single redactor (G23) — never a
per-request access log; a read is audited only where the read itself is the sensitive act, which is reading
a credential.
- **Organisation-visible**, read and exported by the customer organisation administrator, with the longest
  retention: project permission granted, changed or revoked; member added, removed or re-roled; two-factor
  requirement set and a factor removed by an administrator; AI data egress changed; connector configured;
  credential created, updated, read, deleted or rotated; protected-environment flag set or cleared;
  **launch against a protected environment**; project soft-deleted, restored or hard-purged; organisation
  renamed or soft-deleted; scenario or evidence exported; and, reserved now for §4.18, a machine
  credential issued, revoked or first used.
- **Project-visible**, to any project reader: review transitions, scenario created, edited or deleted,
  macro version published, execution launched with its actor, environment and case versions, and the
  outcome of each sync job.
- **Vendor-only**: sign-ins, refreshes, lock-outs, password resets and vendor support access to a hosted
  organisation. These stay where they are and are never shown to a customer.

**The audit trail is one table, append-only, and readable.** Four properties are forced by the code rather
than chosen. It is **scoped by organisation, with the project optional**, because its reader is the
organisation administrator and the organisation is the tenant (§3). The **actor is a stable string captured
at write time, never a foreign key** — every existing actor column dissolves when the user is deleted. The
**target is a real column**, not buried in a document. And it records the **kind of actor** — a person, a
service account, vendor support or the system — from its first migration, because §4.18 depends on it and
it cannot be added later. It is append-only at the database grant level, which means the retention sweep
runs under a separate database role; resolved in the migration, or the grant is quietly loosened later.
**A customer-visible audit screen is in scope**, for the organisation administrator, with export — §1.5
tier 4 puts an audit trail in scope, and a trail nobody can read is not one.

**How it composes with what exists.** The vendor-only login trail stays as it is. The review trail stays as
the review lifecycle's own history and **also writes a row to the audit trail**, so deleting a case no
longer deletes the record of who approved it — cheaper than changing the review aggregate. The orphan AI
trail is dropped in a forward migration with its undo; the AI invocation record stays as it is, since it
serves G32 rather than a customer. **One ordering constraint:** the organisation column on project-owning
data (§3) lands before or with the audit table, because writing the trail against "the owner's
organisation" bakes in exactly the boundary §3 says dissolves when an owner is deleted.

**Notifications: email and one signed outbound webhook.** Email already has its transport, its template and
its switch. A webhook URL is validated by the outbound policy on save and before every call, with no
redirects (G2), and is proven by an attack-path test (G21); everything it carries passes the single redactor
(G23). **Slack and Teams arrive through the webhook, not as native integrations** — both accept an incoming
webhook URL, and a native adapter would be two perpetual maintenance obligations against someone else's
API, which is §4.4's reason for refusing partner connectors. An outbound notification webhook is not an
entry point into the product, so it is not the machine-access webhook of §1.4 and does not touch §1.3.

**The events that notify.** A run that failed (§1.5 tier 3); a run that finished, by subscription, for
unattended runs; a scheduled re-import, **with what it changed** — required, because §4.5 makes that report
the condition for allowing unattended re-import; a failed sync job; a connector credential rejected; and a
macro version published, with the number of scenarios it affects (§4.11). Subscriptions are per project,
per user and per webhook endpoint. **A digest is `OUT`** until a customer asks: it needs a second scheduler
and a second renderer, and it is purely additive later.

**The mechanism is a durable job.** A notification is a row in a delivery table, claimed and retried like
every other crossing of a process boundary, fed by a consumer of the outbox — never a listener making an
HTTP call, and never a task on the pool that discards under load. **The outbox gains a consumer registry**,
so a second event type stops dying on arrival; scheduled runs (§4.12) and sync (§4.14) need the same
registry, so it is built once. The console's bell renders real deliveries or it is removed — a customer-
facing roadmap is the same defect as the self-contradicting landing page of §1.4.

**Impersonation is `OUT`**, as §2 settles: "view as" would make this trail lie about who acted.

**A settings screen is in scope, narrowly.** It is not a front end for the deployment switches — those are
the deployment's shape, and a restart is the right cost. It covers only what the decided sections require
to differ per customer, scoped to the organisation and changeable at runtime: AI data egress (§2 — which
the code still holds per project, a defect against a decided section), the offered connectors (§4.4's
TestRail answer), two-factor required for the organisation (§2), the retention values within the vendor
range, and notification subscriptions. The protected-environment flag stays on the environment (§1.3).

**Retention in three classes instead of fourteen windows.** **Evidence**, as §4.13 decides. **Operational**
— the outbox, import items, AI invocations, drafts and rate-limit windows — vendor-set and not shown to a
customer, with today's defaults. **Audit** — organisation- and project-visible rows kept long, vendor-only
rows ninety days — with a configuration key it does not have today. The two trails with no retention join
the audit class. Where retention is customer-configurable it is set by the organisation administrator,
never by project write access, and the change is audited — the pattern §2 set for AI egress.

**Delete-on-request uses the purge §4.2 already put in scope**, brought into the product as an audited
action in place of the operator script. It deletes everything the data manifest names and writes one
organisation-visible audit row that the purge does not delete — possible only because the trail carries no
foreign key to what it describes. **The honest limit is stated rather than discovered:** erasure reaches
backups only as the daily dump rotation passes it, fourteen days today.

**Still open:**

- `[BLOCKING]` **Owner decision — how long organisation- and project-visible audit rows are kept**, as a
  number. *(Settled: append-only, configurable within a vendor floor, and longer than the ninety days the
  vendor-only rows keep.)*
- `[BLOCKING]` **Owner decision — whether erasure on request is a contractual obligation, with what
  turnaround, and whether backups are inside it.** *(Settled: the mechanism, its audit, and the fact that
  any turnaround shorter than the fourteen-day backup rotation must exclude backups in writing.)*

→ See **§14.4 Email**, **§14.5 Audit**.

### 4.18 Machine access — surfaces that do not exist but are implied

**Where it stands today** `[VERIFIED]` — **there is no distributed CLI, no API token, no service account
and no webhook.** The product has **no machine entry point at all**: it can only run when a person acts in
the console. The OpenAPI document is **not served in production at all** — it is switched off in the
production profile — and where it is served it describes the console's internal surface. Authentication is
a token whose subject is a user, and every authorisation check resolves a username, so nothing in the model
can yet say that a caller is not a person. Two shapes already exist to build on: the mobile agent's
machine credential over the versioned `/v1/runs` wire, and the asynchronous job shape of imports and sync
— a 202 with a job id, then a poll.

This matters because the product is positioned for teams "who want to run their test cases without building
an automation framework" — which most buyers will read as including CI.

> **Decision — `LATER-RELEASE`, release MVP-1.0.1, designed now so that §1.3's expiry is a planned
> re-decision rather than an accident.**
>
> **Rationale.** §1.4 defers machine access rather than refusing it, and §1.3 expires when it ships. That
> expiry is only planned if the entry point is built to make the question answerable. The cheapest machine
> credential — a user row holding a long-lived token — is exactly the design that makes it an accident:
> the record would say a person acted when none did, which is the reason §2 refused impersonation.

**A machine is a first-class principal, not a user row.** A service account belongs to one organisation,
receives project grants through the same grant model as a person, and resolves through the same access
policy. It can never hold the vendor role and never own a project. The one refactor that comes first:
**authorisation takes a principal rather than a username and an administrator flag**, before the first
token is ever issued — afterwards it is a migration over live credentials.

**The audit records the kind of actor** from its first migration (§4.17). Without it, the question §1.3
schedules for MVP-1.0.1 — did a person do this? — has no answer in the data, and §12.10 forbids backfilling
one.

**A machine credential is revocable, expiring, attributable and scoped**: its own revocation, an expiry, a
last-used time, and a scope of projects and of read or write. Issuing, revoking and first use are
organisation-visible audit events. The user-level token version that serves global logout (§4.1) does not
generalise to this.

**The public surface is declared, versioned, and narrower than the console's.** Launch, poll, fetch the
result and fetch the evidence — in the 202-and-poll shape that already exists, under an explicit version
prefix, with **its own OpenAPI document served in production** and a compatibility guard. The console's API
stays internal and undocumented; declaring it public would freeze every console endpoint as a contract.

**One invariant holds now, whatever MVP-1.0.1 decides: a service account may never launch against a
protected environment.** That is §1.3's one hard control, and a long-lived token aimed at production is
exactly what it exists to stop. Beyond it, review transitions carry the kind of actor, so whether a machine
may also move a case through review is chosen at MVP-1.0.1 as configuration and wording, with no migration
— and that choice is the re-decision §1.3 schedules.

**The CLI is a client of the public API and nothing more**, at the same release: no engine, no local
execution and no second code path, so it cannot drift from the API's behaviour. **One GitHub Action** wraps
it. **Jenkins and GitLab plugins are `OUT`**: a shell step calling the CLI does the job, and each plugin is
per-platform maintenance for no capability the CLI lacks. The order is the API, then the CLI, then the
Action. **An inbound webhook as the first machine door** — a Jira transition triggering a run — is `OUT`
for now: it is a second inbound trust boundary that does not answer the CI question, and it can reuse the
signing and outbound-policy machinery of §4.17 later.

→ See **§14.7**.

---

## 5. Screen-by-screen scope

**Three sources, three counts of the same thing:** **38 frames** designed in Figma, **31 routed page
components** in code, **34 "screens"** in `requirements.md`. Use **31 routed page components** as the
mechanical anchor; it is the only number reproducible from the router.

### 5.1 Three-way comparison — the gaps

**Designed but NOT built** `[VERIFIED]` — exactly four frames, all on the same theme:

| Frame | Figma node | Code |
|---|---|---|
| `MfaEnrollmentWizard — method picker` | `2350:16` | implies ≥2 methods; one exists |
| `MfaEnrollmentWizard — WebAuthn prompt` | `2350:28` | **zero** references anywhere |
| `SettingsPage — Security tab (trusted devices)` | `2346:5` | **zero** references anywhere |
| `SecurityTab — remove-one-factor confirmation` | `2350:38` | implies multiple factors; one exists |

This is healthy: the design is ahead of the code, deliberately, and matches the two open MFA requirements.

**Built but NOT designed** `[VERIFIED]` — the real gap, and it is larger:

- **The overlay's internal panels.** Figma has **one** frame for the overlay; the code has six substantial
  panels inside it (import, review queue, case detail, step mapper, sync jobs, feature import). *The most
  important stage of the product is designed as a single screen.*
- **The entire AI assistant surface** — panels embedded in five screens, with no route and no frame.
- **The mobile agent surface** — onboarding, readiness checks, device selection. It is the onboarding for a
  component the customer installs on their own machine, and it has no design.

> **Decision — close two gaps, accept one, and bring the designed two-factor frames into line with §4.1.**
>
> **Rationale.** The criterion is §1.1's sentence: a title and an ID become a running scenario with
> evidence. The overlay is on that path and is designed as one frame for six panels; the agent surface has
> moved onto it, because §1.5 and §3 turn the mobile agent into the customer-side runner; the AI surface
> is not on it while AI is off.

**The overlay's panels: close the gap.** Six substantial panels designed as a single frame, on the
flagship, is the largest design debt in the product.

**The agent surface: close the gap, and widen it.** Its onboarding, readiness checks and device selection
stop being a mobile detail and become the onboarding for the component that makes private-network
execution, the element inspector and run durability work.

**The AI assistant surface: accept the gap for now.** AI is off by default, the generative surfaces are
hidden where it is off (§4.6), and designing panels before the capabilities they serve are settled in §8
is the one case here where waiting is right.

**The four designed-but-unbuilt frames follow §4.1.** The method picker, the WebAuthn prompt and the
remove-one-factor confirmation are built with WebAuthn, which §4.1 puts in scope. The trusted-devices tab
is removed from the design, because §4.1 rules trusted devices out.

### 5.2 Per-screen decisions

`Scope`: **K**eep / **R**edesign / **D**rop. **Bold** marks the screens that must be excellent.

| # | Route | Screen | Designed | Tested | Scope | Notes |
|---|---|---|---|---|---|---|
| 1 | `/` | LandingPage | ✔ | ✔ | R | copy corrected to match this document and the code (§1.4) |
| 2 | `/login` | LoginPage | ✔ | ✔ | K | |
| 3 | `/login/mfa-challenge` | MfaChallengePage | ✔ | ✔ | K | gains WebAuthn (§4.1) |
| 4 | `/forgot-password` | ForgotPasswordPage | ✔ | | K | |
| 5 | `/signup` | SignupPage | ✔ | ✔ | K | hosted: self-service with admin activation; self-hosted: off (§4.1) |
| 6 | `/reset-password` | ResetPasswordPage | ✔ | | K | |
| 7 | `/oidc/callback` | OidcCallbackPage | ✔ | ✔ | K | off by default |
| 8 | `/auth/force-change-password` | ForceChangePasswordPage | ✔ | | K | |
| 9 | `/onboarding` | OnboardingPage | ✔ | | K | reachable only by automatic redirect |
| 10 | `/dashboard` | GlobalDashboardPage | ✔ | ✔ | R | vendor-only operational surface; removed from the customer sidebar (§4.15) |
| 11 | `/dashboard/projects/:id` | ProjectDashboardPage | ✔ | partial | R | reached from the project; requirement-coverage card removed (§4.15) |
| 12 | `/projects` | ProjectsPage | ✔ | | K | |
| 13 | `/projects/:id` | ProjectDetailPage | ✔ | **none** | K | |
| 14 | `/projects/:id/settings/:tab` | ProjectSettingsPage | ✔ | ✔ | R | `general` gains name and description (§4.2); `execution-defaults` removed, since environment and runtime are chosen at launch |
| 15 | — | ProjectMembersTab | ✔ | ✔ | R | shows the project owner; the vendor role is not a member and is not shown |
| 16 | — | ProjectIntegrationsPage | ✔ | ✔ | R | offers what the organisation's settings allow (§4.4, §4.17); write-back can be turned off (§4.14) |
| 17 | `/projects/:id/environments` | **EnvironmentsPage** | ✔ | **none** | K | protected flag, secrets, fixtures and mTLS live here (§1.3, §4.3) |
| 18 | `/projects/:id/ssl/certificates` | CertificateManagerPage | ✔ | **none** | K | the "trust all" option goes to security review |
| 19 | `/projects/:id/suites` | SuitesPage | ✔ | **none** | K | |
| 20 | `/projects/:id/suites/:id` | SuiteDetailPage | ✔ | **none** | K | bulk execute carries the full runtime configuration (§4.12) |
| 21 | `.../features/:id` | **FeatureDetailPage** | ✔ | **none** | K | Gherkin paste re-routed through the stored model (§1.2) |
| 22 | `.../scenarios/new`, `/edit` | **ScenarioBuilderPage** | ✔ | **none** | K | the founding door; hosts the element inspector (§1.5) |
| 23 | `/steps` | **StepsPage** | ✔ | ✔ | K | the capability statement (§4.10) |
| 24 | `/projects/:id/custom-steps` | CustomStepsPage | ✔ | | R | macro versions and organisation sharing (§4.11) |
| 25 | `/projects/:id/imports` | ImportsPage | ✔ | thin | R | generative path hidden when AI is off; deterministic extraction (§4.6) |
| 26 | `/projects/:id/suggestions` | ScenarioSuggestionsPage | ✔ | **none** | R | 1,380 lines, no test; redesigned to §4.6 rather than locked into tests |
| 27 | `/projects/:id/suggestions/:id` | ScenarioDraftBuilderPage | ✔ | **none** | R | drafts converge on the canonical model (§4.6) |
| 28 | `/projects/:id/execution-overlay` | **ExecutionOverlayPage** | ✔ (1 frame for 6 panels) | ✔ | K | the flagship; its design gap is closed (§5.1) |
| 29 | `/projects/:id/executions` | HistoryPage | ✔ | **none** | K | |
| 30 | `/.../executions/:id` | **ExecutionLivePage** | ✔ | ✔ | K | |
| 31 | `/.../executions/:id/results` | **ExecutionResultsPage** | ✔ | ✔ | K | the evidence (§4.13) |
| 32 | `/settings` | SettingsPage | ✔ | **none** | K | no trusted-devices tab (§4.1) |
| 33 | `/admin/users` | AdminUsersPage | ✔ | ✔ | R | becomes the organisation administrator's member management, and is route-guarded |
| 34 | `/components/*` | ForcedNotFoundPage | ✗ | | K | a deliberate 404 for links to a retired surface |
| 35 | `*` | RouteErrorFallback | ✗ | | K | "Report issue" appears only where a deployment sets its URL, and is never set on a customer's |

**What must be excellent is what §1.1's sentence passes through.** The seven bold screens are the path by
which a title and an ID become a running scenario with evidence: the catalog, the two authoring screens,
the environment where protection, secrets and fixtures live, and the three execution screens. Everything
else must work; the landing page is a copy correction before it is a design problem.

**`ScenarioSuggestionsPage` is redesigned, not locked into tests.** §4.6 changes its behaviour twice over —
drafts converge on the canonical model, and the generative path gives way to deterministic extraction where
AI is off — so tests written against today's behaviour would freeze exactly what is being removed. It is
also the screen where "present-and-inert is worse than absent" has to become visible.

**Project general settings exist.** §4.2 settles it: name and description editable, key immutable. The
`general` tab already exists as an empty card, so the gap is one form and one mutation. The
`execution-defaults` tab is removed instead of filled: environment and runtime are chosen at launch, and a
defaults tab would be a second place to configure one thing (G25).

**The "Report issue" link is an internal affordance, never a customer one.** The mechanism is already
right: it appears only where a deployment sets its URL, has no default, and is wrapped as every external
link is (G22). Pointed at our own tracker on a customer deployment it would publish their stack traces; a
customer's support path is the redacted diagnostics bundle of §3.

**Dropping a screen drops its endpoints, and its data.** Leaving the endpoints is how a product acquires an
undocumented API, and once §4.18 issues machine credentials anything reachable is something a customer can
build against. §4.15's requirement coverage is the worked example.

**The console is desktop-first and usable down to 320 pixels.** The reflow tests already prove the second
half; a full tablet or phone experience is not promised. The authoring loop and the element inspector are
desktop work.

---

## 6. Test-execution capability, by layer

### 6.1 Cross-layer

> **Decision — the catalog is the promise, layer by layer, and nothing outside it is claimed.**
>
> **Rationale.** §4.10 settles it: if there is no step, we do not claim it. Each layer's promise is
> therefore a description of its catalog, and every question below about a capability the catalog lacks has
> the same answer.

**The promise, one sentence per layer, as a buyer would read it:**
- **HTTP/API** — calls REST APIs with six authentication schemes on either of two interchangeable engines,
  and asserts on status, headers, body and JSON Schema, on concurrent load and on response security.
- **Web** — drives Chromium, Firefox or WebKit through forms, frames, tabs and Shadow DOM, and asserts on
  what the page shows, including nineteen field formats.
- **Mobile** — drives native and hybrid apps on Android emulators, Android devices and iOS simulators,
  through an agent on the customer's own machine.
- **Database** — connects to PostgreSQL, SQL Server or Oracle, prepares and changes data with SQL, and
  asserts on what the tables hold.

**Nothing not implemented is published.** The engine's advertisement of GraphQL and SOAP is removed now,
whatever is later built (§4.10).

**No further layer is in scope until a step request brings it.** gRPC, WebSocket under test, message queues
and email under test are not claimed, and none is committed as a future layer; each can arrive only through
the step request of §1.5. **Performance, security and accessibility testing are `OUT` as layers.** The
seven concurrency steps and seven response-security steps of the HTTP layer are claimable as steps; no
performance, security or accessibility *layer* is.

**A scenario is strictly linear, and that is a product principle.** §4.10 excludes conditional steps as a
determinism policy, permanently; a scenario that branches is a scenario whose verdict depends on a path
nobody chose.

**A scenario does not span layers, and that is not a promise.** §4.9's executability rule binds every step
to the case's layer, so an HTTP call followed by a database assertion is not expressible in one case. Two
cases in one suite is the way to test such a flow today. This is the answer most easily given the other way
by accident, which is why it is written down.

### 6.2 HTTP / API — 141 steps

**Strong today:** two interchangeable engines, nine authentication steps covering six schemes, JSON Schema
validation, seven concurrency/load steps, seven security-assertion steps, polling and retry.
**Absent:** NTLM, Digest, AWS SigV4, mutual-TLS as a step, WebSocket, gRPC, HAR capture.

**The authentication schemes supported are the six the catalog implements:** Bearer token, Basic, API key in
a header or a query parameter, OAuth2, client credentials and JWT. Nine steps cover them; three of the nine
are fixtures for negative tests — no authentication, an expired token, an invalid token. NTLM, Digest and
AWS SigV4 are absent and not claimed; each can arrive only through a step request.

**File upload is in scope** — settled in §4.3, which puts the per-project fixture store in scope, and in
§4.10, which unhides the file steps for that reason. The engine side was always complete; what was missing
was a place for the file.

### 6.3 Web — 186 steps

**Strong today:** Chromium/Firefox/WebKit, Shadow DOM including nested hosts, 19 field-format validations,
iframes, multi-tab, and an outbound filtering proxy that checks every browser connection on managed runs.
**Absent:** video, trace, HAR, network mocking, visual regression, accessibility assertions.
**Dead:** Selenium Grid configuration exists and is read by nothing.

**All three engines — Chromium, Firefox and WebKit — and the choice is the user's** (§1.5). Making managed
runs accept all three is required work, as §1.5 records.

**No browser version is offered or promised.** The version is whatever the pinned Playwright release
bundles. "Chrome" in the configuration is an alias for Chromium, not Google Chrome, and no branded channel
— Chrome or Edge — is selectable.

**Selenium is `OUT`, and its leftover configuration is removed.** Playwright is the only Web engine. The
Selenium Grid keys are read by nothing, and the library they would configure is not even a dependency — so
removing them is the §4.4 precedent applied: dead configuration is a trap, not a roadmap.

**Testing an application behind a login is in scope; a second factor on it is not claimed.** Credentials are
held in the environment, outside the scenario text (§1.5 tier 1), so a login can be driven like any other
form, SSO pages included where they can be driven in the browser. No step generates a second factor for
the system under test, so that is not claimed and can arrive only through a step request; the usual
practice is a test account without one, in a non-production environment.

**Visual regression is `OUT`.** No step exists, and it is a different product — baselines to approve and
store, and a comparison that is a judgement rather than a verdict.

### 6.4 Mobile — 136 steps

**Where it stands today** `[VERIFIED]` — **physical Android devices ARE supported** (there is a step for
it). **Physical iOS is not**: device discovery is simulator-only and no step selects a physical iPhone.
There is **no cloud device farm integration**. A mobile run **requires the customer to install an agent** on
the machine with the devices; the published container image can never run iOS, because that needs macOS.

**The targets claimed are the ones that exist:** Android emulators, Android physical devices, iOS
simulators, and any Appium-compatible hub the customer points the product at. **Physical iOS and a vendor
device-farm integration are not claimed**, because neither exists; whether to build or buy either is a
spending decision, open below.

**The customer provides the devices**, through the agent on their own machine — the shape §1.5 and §3 already
decide for reaching anything in the customer's network. Vendor-provided devices would make A·Spectra a device
farm, which is the spending decision below.

**The app under test becomes a first-class, stored object.** Today it reaches the agent as a hand-copied
absolute path, which breaks on the second build and the second tester. It is stored against the project and
referenced by the scenario, by the same mechanism as §4.3's fixture store, and it carries its build
identity — which is what §4.14's "build under test" records for a mobile run.

**Native and hybrid apps are in scope; mobile web is not claimed.** The catalog switches between native and
web-view contexts, so hybrid apps are covered. A website in a device browser has no verified step or
configuration, so it is not claimed.

**Concurrent mobile runs are bounded by the devices, and no number is promised.** Devices are leased
exclusively and a second run asking for the same one fails fast rather than sharing it; the devices are the
customer's, so the ceiling is theirs too.

**Still open:**

- `[BLOCKING]` **Owner decision — physical iOS and a device farm.** Whether to buy access to physical iOS
  devices or a vendor device farm, and so to claim them. *(Settled: neither is claimed today; the product
  cannot build physical iOS itself, and the cost is priced in §14.6.)*

### 6.5 Database — 52 steps

**Where it stands today** `[VERIFIED]` — the thinnest layer by a wide margin. **PostgreSQL, SQL Server and
Oracle have all been executed against end to end**: since 2026-10-15 an integration test runs 51 of the 52
steps against real instances of each, through the product's own API, and the Core README that still calls
the other two unproven is out of date. MySQL is wired in — its driver ships to consumers — and is as
unproven as the other two were; H2 ships no driver. **34 of the 52 steps are assertions, and 23 of those are
variations on comparing one column value.**

> **Decision — three engines promised, writes allowed as steps, production behind the gate.**
>
> **Rationale.** §4.4 sets an engine's proof bar at execution end to end, and three engines now clear it.
> The rest follows from §1.3 and §4.3.

**The engines promised are PostgreSQL, SQL Server and Oracle.** MySQL and H2 stay out until proven, by the
bar §4.4 states.

**An engine that has never been executed against end to end is never claimed** — the same truthfulness
rule as GraphQL and SOAP.

**Connecting to a customer's production database is supported, behind the gate of §1.3**, and this section
adds nothing to that safeguard (§4.3).

**Writing to the database is in scope, as steps.** The catalog already prepares and changes data — it runs a
statement or a script, sets a table's contents and empties one — and §4.3's boundary is on a test-data
management feature, not on steps an author writes. A write against a production-like database is
precisely the case the protected-environment gate exists for. One step accepts a URL, a user and a password
inline: its password must be an environment reference, because a literal typed there is exactly the
step-text gap of §4.3.

---

## 7. Integration scope

> **Decision — four test management systems, connectors that stay ours, and a mapping the customer does
> not redefine.**
>
> **Rationale.** §4.4 already decides the set and its order, §1.4 decides what is written back, and §4.4's
> refusal of third-party connectors decides who owns the mapping. This section turns those into the
> connector contract.

| Test management system | Import | Write-back | Priority |
|---|---|---|---|
| Jira + Xray, Server/Data Center | yes | yes — results and evidence | 1, built and offered |
| Jira + Xray, Cloud | yes | yes — results and evidence | 2, built and offered |
| TestRail | yes | yes — results and evidence | 3, built and contract-tested |
| Jira without Xray | yes | yes — a comment and attachments on the issue, since plain Jira has no test-result object | 4, not built |

Azure DevOps is not in the table: §4.4 removes it as dead scope.

**Server/Data Center and Cloud are both permanently in scope, and equivalent by one definition.** They are
equivalent when they pass the same shared contract suite against their simulators (G26) — which all three
built connectors run today. Differences the providers force — the credential each accepts, the rate limit
each documents, the regional host Xray Cloud uses — are not inequivalence; a capability one offers and the
other lacks is.

**Each connector supports the credentials its provider issues:** a personal access token or a password for
Jira Server/Data Center, an API token for Jira Cloud, a client id and secret for Xray Cloud, and an API key
for TestRail — all held in the encrypted store, write-only (§4.3).

**The field mapping is ours, and the customer does not redefine it.** Which source field carries which
canonical role is the connector's contract, applied by the normaliser. Customer-defined mappings are `OUT`
— the connector set stays ours (§4.4), and a per-customer mapping is per-connector work for a choice no
customer has asked for. The one thing an integration does record is **which instance-specific field plays a
fixed canonical role**, where a provider keeps it in a field whose name differs per installation; that is
confirmed once per integration by a person, and it is configuration of the connector, not a mapping of the
customer's own.

**A source field the canonical model has no home for is dropped, and the import proceeds.** Storing it
opaquely would be a field nothing reads (G7); refusing the import would punish the customer for having a
richer system than ours; and provider concepts stay inside their connector (G27).

**The canonical model is not a published contract.** The published contract is the step catalog (§4.10);
the canonical model is internal, and there is no partner to build against it, since partner connectors are
`OUT`.

**A connector is offered when it passes the shared contract suite against its simulator** — not after
validation against a real customer instance. That is the connector proof bar §4.4 states beside the
engine's, and it is deliberately different: an engine's behaviour is the product's, a provider's is the
provider's.

**Credential health is checked proactively.** A scheduled connection check tells the customer a token has
expired before an import fails, through the "connector credential rejected" notification of §4.17. It
promises what §4.3 says a test connection promises — reachability and authentication, in those words, and
nothing further.

**Rate limits mirror each provider's published limits.** 300 calls a minute for Jira + Xray, 60 for Xray
Cloud, and 180 for TestRail — the conservative Cloud figure, raisable for an on-premise instance. They are
enforced by the outbound limiter, which is database-backed and shared across instances. A per-customer
override is `OUT` on the hosted SKU, where the limit is the provider's rather than ours to raise; on
self-hosted the customer sets it.

**The customer can see the exact payload written to their system.** It is deterministic — never produced
by a model — and already stored with its sync job, so showing it is cheap and earns trust: this is exactly
what we wrote into your Jira. Replaying it is the existing retry of a failed sync job, and nothing more; a
customer-triggered replay of a payload that already succeeded would write a duplicate result.

**Still open:**

- `[non-blocking]` **Owner decision — a Jira Marketplace listing.** Whether the product is also distributed
  as an Atlassian Marketplace app. *(It would change the authentication model, add Atlassian's security
  review, and change the trust story. Nothing built depends on it.)*

→ See **§14.10 Connector maintenance**.

---

## 8. AI scope

**Where it stands today** `[VERIFIED]` — **11 capabilities** are registered and the **entire area is off by
default**. The governance around them is unusually strong and almost entirely undocumented: a closed
capability registry validated at startup, three model tiers over a hosted and a self-hosted provider (a third,
legacy slot defaults to disabled), a **per-project
data-egress setting** (`HOSTED_ALLOWED` / `SELF_HOSTED_ONLY` / `AI_DISABLED`) that produces a typed refusal
rather than a silent downgrade, mandatory redaction with no "none" option, deterministic fallbacks per
capability, versioned prompt files, durable token budgets, and an audit that stores hashes and never prompts.

> **Decision — an accelerator the customer may switch on, never the product.**
>
> **Rationale.** §1.1's sentence and its closed list of differentiators do not mention AI, and the product
> works completely with it off. That is the posture: AI makes a QA faster at work a person still decides,
> and nothing becomes unavailable because a model is unavailable.

**AI is off by default, permanently, and every capability keeps a deterministic fallback.** On-by-default
is not the target. It follows from §2, which puts AI data egress in the hands of the customer organisation
administrator, and from §4.6, whose deterministic mode exists precisely so a default deployment loses
nothing.

**The switch belongs to the organisation.** §2 decides it: AI data egress is an organisation decision,
changed only by the customer organisation administrator, never by project write access, and audited. The
code still holds the egress posture per project — `HOSTED_ALLOWED`, `SELF_HOSTED_ONLY`, `AI_DISABLED` — which
is a defect against a decided section and moves to the organisation settings of §4.17. A project may still
opt into a capability the organisation allows, never into one it forbids.

**The eleven capabilities: nine in scope, two experiment-only.** Every one is subject to the switch above,
to a human decision before anything reaches domain state, and to the commitments below.
- **In scope:** structuring an imported free-text case into canonical steps; proposing a catalog step for an
  imported step the matcher left unmapped; explaining reasons the platform already computed; narrating a
  finished run as a separate artifact beside the deterministic report, which it never alters; proposing
  environment settings found in the source, under §4.6's extraction rule that a model may never supply a
  value the source does not contain; flagging near-duplicate cases in an import, never merging or deleting
  them; composing a macro from existing catalog steps, which is §1.5's composed-step suggestion; naming a
  step sequence a deterministic count found; and drafting canonical cases from parsed endpoints or
  criteria, hidden where AI is off (§4.6).
- **Experiment-only:** the two agentic capabilities — failure triage, which reads untrusted output from the
  system under test while holding tools, and proposing which unfamiliar provider field carries which
  canonical role. They stay opt-in, produce advisory metadata only, and are not sold or promised until each
  has proven itself against its own evaluation (G31). The deterministic grouping of failures by cause that
  §1.5 tier 3 asks for does not depend on the first.

**Consent is settled; the provider is not.** Customer test content leaves the deployment only when the
organisation administrator has allowed it, which is the default-off posture above, and a self-hosted
customer holds their own keys (§3). Which providers, in which region and under what agreement is open below.

**A self-hosted model is supported, and required by nothing.** The product already carries a self-hosted
provider beside the hosted one, and the egress posture that keeps content inside the deployment pairs with
it. No capability depends on it, because every capability has a deterministic fallback. Out of the box no
model is mapped to any tier — an unmapped tier is refused rather than defaulted — so an operator maps one
before anything runs.

**The "AI never" list is a commitment a customer can hold us to.** A model never decides pass or fail;
never produces a count, a duration or a status; never writes a sync payload or renders Gherkin; never makes
an authorisation or outbound-policy decision; never computes a source version hash; never touches a
credential; **never invents test data, credentials, URLs, selectors or expected results**; and never alters
a step pattern. §1.3 states that these guarantees are untouched by the withdrawal of human approval as an
enforced property, and they are enforced in code today (G28–G33).

**"A model may say an expected result is missing, but may never supply one" is published.** It is decided
in §4.6, enforced with no open gap, and it is the sharpest differentiator against products whose tests
assert a model's guess and therefore pass. Unlike the no-leak promise of §4.3, nothing stands between it
and publication.

**What a model touched is recorded and visible, permanently.** A spec-derived case is labelled as such
(§4.6), a composed step records whether it was written by hand or accepted from a suggestion (§1.5), and
review transitions are typed as human or system (§4.9).

**Still open:**

- `[BLOCKING]` **Owner decision — who pays for model usage**: absorbed by us, the customer's own key, or
  metered. *(No billing code exists, so nothing constrains the answer; §14.11 and §14.14 price it.)*
- `[BLOCKING]` **Owner decision — the hosted model provider, its region and its agreement**, and its place
  on the sub-processor list. *(Settled: consent is the organisation administrator's, default off, audited.)*

→ See **§14.11**.

---

## 9. Non-functional scope

**Where it stands today** `[VERIFIED]` — **no non-functional, SLA, runbook or known-limitations document
exists anywhere**, and the deploy workflow points at a runbook section of a file that has no such section.

> **Decision — commit to what the decided sections already force; leave every external number to the
> owner.**
>
> **Rationale.** Most of this section is not a set of decisions at all: §3 (high availability in scope,
> one region, row-scoped isolation) and §1.5 (the committed speed of the authoring loop) already determine
> it. What remains is a set of promises about numbers — availability, recovery, scale, remediation — and
> a promise is a business act. The two are separated below so neither is answered by accident.

### 9.1 Scale and performance

**The authoring-loop target is a promise per launch, not per system.** API or Database under five seconds
and Web or Mobile under twenty, from the launch click to the first step result, **when a run slot is free**
(§1.5). Behind a full queue no target can hold, so the promise says so.

**When capacity is exhausted the product queues to a bounded depth, then refuses explicitly.** Silent
degradation is excluded (G20). The refusal states a retry time computed from the actual queue, instead of
today's five seconds hardcoded whatever the queue holds; and how many run slots are in use is measured,
which today it is not — without that the speed target cannot be checked at all.

**The warm browser pool is the customer-side runner, not a Core feature.** Core binds a browser to one
execution and closes it at the end, in thread-bound state. Reusing it across executions inside Core would
change Core's frozen surface and put cross-execution state in the JVM, against G19. A warm runner process
per engine, with its browser already up, gives §1.5 its per-engine pool without touching Core — and it is
the same runner that §3 makes the answer to private networks, the inspector and run durability. One piece
of infrastructure, for four commitments.

**The limits the product enforces are published.** 500 cases per import job, larger selections split
(§4.5); a two-hour run timeout (§4.12); and the inbound request limits — published once they are shared
across instances (§3), since until then a published figure is wrong by the number of instances. An
enforced limit nobody published is one a customer discovers during an evaluation.

### 9.2 Availability and resilience

**A deploy is zero-downtime, as a committed property.** §3 makes losing an in-flight run on every deploy
unacceptable permanently, and out-of-process execution on the runner is how it is delivered.

**What high availability forces is recorded as work, not choice** — §3 lists the single-instance state.
Evidence moves to S3-compatible object storage, which also serves self-hosted through any compatible store;
a shared mount is rejected, because it keeps a filesystem path as the contract and answers neither
encryption at rest nor the per-execution evidence export of §3. The sign-in handoff and both two-factor
caches move to durable storage; the inbound rate limiter moves onto the database-backed pattern the
outbound limiter already uses, rather than onto a new datastore. Sticky sessions are rejected: a
load-balancer dependency hiding a correctness defect fails on the next rolling restart exactly the same
way.

**Recovery is stated as what exists.** A daily database dump, fourteen kept, gives a recovery point of about
a day. A restore needs a fresh database, the evidence files are not in the dump at all, and no restore has
ever been proven. **No recovery promise is published until a restore has been proven**, and the backup
must cover evidence — which the object storage above provides.

### 9.3 Security posture

**Encryption at rest is in scope for the database and the evidence, and published once it is true.**
Credentials are encrypted today, column by column, with AES-256-GCM and the row's identity bound into the
ciphertext — but the key is a plain environment variable, nothing rotates it, the database itself is
encrypted by nothing in the repository, and evidence is plain files on disk. Object storage makes evidence
encryption a matter of configuration. The rule is §4.3's: an internal standard may be written here, and is
published only once nothing contradicts it.

**A vulnerability-remediation window, once set, covers dependencies.** Dependency vulnerabilities are inside
the mandatory security gate before every promotion, and every security control is proven by an attack-path
test (G21); the mechanism exists, and only the window is open.

**A static egress address exists as a mechanism.** The configuration carries the public source address a
customer allowlists, and the console already displays it — empty, deliberately, until the hosted
deployment has one. Note that §1.5's runner answers reaching the customer's *application*; a connector
still calls their test management system from our address, so this does not go away.

### 9.4 Privacy, residency and legal

**Residency is one fixed region, ours, and disclosed** (§3). Which region is §3's open item.

### 9.5 Internationalisation and accessibility

**The interface languages are English, Spanish and French**, and further languages are deferrable. The three
locales carry 2,517 keys each and are held equal by a test.

**Content is localised where it is display text, and never where it is identity.** The interface and the
catalog's step names and descriptions are localised (§4.10); step patterns never are (G34); a message from
the customer's test management system is passed through as the provider wrote it, not translated.

**The accessibility target is WCAG 2.1 AA, because that is what the gate enforces.** The automated check
runs the 2.1 AA rules on the primary routes at two window widths, and the reflow check covers 320 to 1,440
pixels. Its own header claims 2.2 AA while its rules check 2.1 — a defect to fix before any conformance is
claimed, either by adding the 2.2 rules or by correcting the header. Automated checks cover only part of
the criteria; the rest need a manual checklist.

### 9.6 Client support

**The console supports current versions of Chromium-based browsers, Firefox and Safari, and the promise is
tested.** The build targets current browsers and declares nothing narrower, while the console's own
end-to-end gate runs Chromium only — so Firefox and WebKit are added to that gate, rather than the promise
narrowed to Chromium. A QA team uses whichever browser it uses.

**The console is desktop-first and usable down to 320 pixels** (§5.2).

### 9.7 Observability

**The customer sees their own runs and jobs; vendor telemetry stays the vendor's.** That follows from §3:
on self-hosted we cannot read the customer's data, so whatever the customer needs to operate the product
must be inside it. What exists already — live execution pages, a sync-jobs panel, import-job status, and a
banner that says when the system is degraded or a user is rate-limited — is completed by one
organisation-scoped view of durable jobs and their failures, which is the same read path as the audit of
§4.17. Health and info endpoints are public; metrics sit behind a dedicated credential no platform account
can reach; logs are structured and carry a correlation id into asynchronous work and outbound calls. There
is no tracing.

**Still open** — every item below is a promise, not a design:

- `[BLOCKING]` **Owner decision — target scale**: customers per deployment, projects per customer, cases per
  project, concurrent users, concurrent runs, runs per day. *(Every number in this section and in §10 is
  priced against it.)*
- `[BLOCKING]` **Owner decision — the promised time to import 500 cases**, and the console's promised
  responsiveness on the largest realistic data set. *(Settled: 5,000 cases is ten jobs of 500.)*
- `[BLOCKING]` **Owner decision — the availability promised, and to whom.**
- `[BLOCKING]` **Owner decision — the recovery point and recovery time promised**, once a restore is proven.
- `[BLOCKING]` **Owner decision — whether a status page and incident communication are part of the product.**
- `[BLOCKING]` **Owner decision — the security standard claimed or worked toward**: none, ISO 27001, SOC 2,
  or questionnaires only.
- `[BLOCKING]` **Owner decision — penetration-test cadence, and the remediation window by severity.**
- `[BLOCKING]` **Owner decision — a static egress address as a contractual commitment.**
- `[non-blocking]` **Owner decision — customer-managed encryption keys (BYOK).** *(Credentials already
  accept the customer's own secret manager (§4.3); BYOK would concern evidence and the database, and adding
  it later is a re-encryption migration over live data.)*
- `[BLOCKING]` **Owner decision — the jurisdictions and regulations that apply.**
- `[BLOCKING]` **Owner decision — sub-processor disclosure, and who maintains the list** — hosting, email,
  model provider, object storage, and any device farm.
- `[BLOCKING]` **Owner decision — who supplies the Terms of Service, Privacy Policy and DPA, and who owns the
  IP in the existing codebase.** *(Scope-blocking and commercial; nothing in code touches it.)*
- `[BLOCKING]` **Owner decision — whether WCAG conformance is claimed contractually, and whether any target
  customer requires it.** *(Settled: 2.1 AA is the target the gate enforces.)*
- `[non-blocking]` **Owner decision — the console's fallback language.** It is Spanish today, for a product
  whose engineering language is English; which one serves the customers you are selling to is market
  knowledge.

→ See **§14.13**.

---

## 10. Operations, lifecycle and commercial model

> **Decision — an upgrade never breaks the customer's asset; everything commercial is the owner's.**
>
> **Rationale.** The scenario is the customer's asset and we are its custodian (§1.4). Everything that
> follows from that is decided here; how the product is sold, supported and priced is not something any
> decided section or any line of code determines.

**An upgrade never breaks stored data or approved scenarios.** The mechanism is forward migrations and frozen
step patterns: an applied migration is never edited (working rule 14), and a pattern change would invalidate
every stored scenario (G34, §12.11). **The record is corrected on one point:** upgrading a non-fresh database
is proven — at least six migration tests seed a populated database in shapes the application can no longer
produce, then migrate on. What is unproven is **rollback**: fifty-five undo scripts exist, exactly one is run
by a test, and the Flyway edition in use has no native undo, so it cannot run them. A failed upgrade is
therefore recovered from a backup, and the promise to the customer is forward-only. A full upgrade from the
first migration over a realistically populated database is also unproven.

**Backward compatibility is guaranteed for stored and approved scenarios, indefinitely.** They are the
customer's asset and their executable identity is frozen. The canonical model carries no external
guarantee, because it is not published (§7).

**Withdrawal is never silent.** A connector, a step or a capability that is withdrawn is announced and named
in the product before it goes, as §4.10 requires of steps; the notice period is §4.10's open item.

**Versioning is release-numbered,** as this document already uses it; the catalog is versioned with it
(§4.10).

**A runbook is a deliverable.** §3 makes self-hosting the first SKU, which means the customer operates the
product, so the instructions for operating it are part of what is shipped. It is maintained with the product
by its owner, and the deploy workflow's pointer to a runbook that does not exist is a defect.

**A trial exists, and the read-only integration is its low-friction path.** §4.1 already presumes one — a
trial has to be able to start itself — and §4.14 lets it import and run before it writes anything into
the customer's system. Its shape is open below.

**The public website copy is owned by the scope owner, through this document.** §1.4 decides that the
landing page is corrected to match this document and the code, as its own task once the scope is closed.

**Still open** — each is a commercial or operational commitment:

- `[BLOCKING]` **Owner decision — the commercial model**: licensed, subscribed, metered or free; per user,
  per project, per run or per connector. *(No billing, licensing or metering code exists, so there is no
  sunk cost either way; §14.14 prices the options.)*
- `[BLOCKING]` **Owner decision — the trial's shape**: limited by time, by usage or by feature.
- `[BLOCKING]` **Owner decision — who operates the hosted service, and its support hours.**
- `[BLOCKING]` **Owner decision — the support model**: email, in-app, a named contact, severity levels with
  response targets.
- `[BLOCKING]` **Owner decision — whether onboarding is a paid service, self-service, or both.**
- `[BLOCKING]` **Owner decision — the release cadence** the customer sees.
- `[BLOCKING]` **Owner decision — the backup and restore promise**, once a restore has been proven (§9.2).
- `[BLOCKING]` **Owner decision — the first customer, and what success looks like.**

---

## 11. Explicit non-goals

Each item is a permanent non-goal, not yet, or in scope, with the section that decides it.

| Item | Permanent | Not yet | In scope | Decided in |
|---|---|---|---|---|
| Being a test management system | ✔ | | | §1.4 |
| Replacing the customer's TMS | ✔ | | | §1.4 — though authoring in A·Spectra is a first-class door (§1.2) |
| Test authoring as the primary workflow | | | ✔ | §1.2 — a first-class door among several |
| AI-generated test cases | | | ✔ | §4.6, §8 — as drafts a person accepts, never with an invented expected result |
| AI deciding pass/fail | ✔ | | | §1.4, §8 |
| Azure DevOps | | ✔ | | §1.4 not permanent; §4.4 removes the empty enum value |
| Selenium | ✔ | | | §6.3 — Playwright is the Web engine |
| Performance / load testing | ✔ | | | §6.1 — as a layer; the concurrency steps are claimable as steps |
| Security testing | ✔ | | | §6.1 — as a layer; the response-security steps are claimable as steps |
| Accessibility testing of the customer's app | ✔ | | | §6.1 |
| Visual regression | ✔ | | | §6.3 |
| Manual-test execution tracking | ✔ | | | §1.4 |
| Defect management | ✔ | | | §1.4, §4.14 |
| Requirements management | ✔ | | | §1.4, §4.5 |
| CI/CD orchestration | ✔ | | | §1.4 — being called from CI is §4.18's, at MVP-1.0.1 |
| Customer-hosted deployment | | | ✔ | §3 — the first SKU |
| Physical iOS devices | | ✔ | | §6.4 — an owner spending decision |
| Cloud device farm | | ✔ | | §6.4 — an owner spending decision |
| A CLI | | ✔ | | §4.18 — MVP-1.0.1 |
| A public API | | ✔ | | §4.18 — MVP-1.0.1 |
| MySQL / MariaDB | | ✔ | | §4.4 — out until executed against end to end |
| Plain Jira without Xray | | | ✔ | §4.4 |
| SAML | | | ✔ | §4.1 |
| File upload testing | | | ✔ | §4.3 |
| Run notifications (email/Slack/webhook) | | | ✔ | §4.17 — Slack and Teams through the webhook |
| Hosting devices ourselves (being a device farm) | | ✔ | | §6.4 — the same owner spending decision |
| Monitoring / synthetic uptime probing of production | ✔ | | | positioning — a scheduled run may target production behind the gate (§4.12), but the product is not built or sold as monitoring |
| User impersonation for support | ✔ | | | §2 |

**"Built but not offered" is never a silent state.** Every built area is offered, removed, or visibly gated
with a written reason — "out until proven" being the permitted third form. That is the rule §4.10 sets for
steps and §4.4 applies in both directions: TestRail offered, the Azure DevOps value removed. It applies to
every remaining flag: the email-the-result code with no caller, the scheduled-run label with no writer, the
Selenium configuration, the GraphQL and SOAP advertisement, and the other switches that hide finished work.

**The permanent non-goals are stated publicly; the "not yet" items are not.** A product that says plainly what
it is not earns trust a vaguer one cannot, and four of these are public already. The "not yet" items are
roadmap, and stating roadmap publicly is a promise.

---

## 12. Decisions that are cheap now and expensive later

Each was a breaking change for customers if deferred past the first release. **All twelve are now decided**;
the table records where, and replaces two remedies that would have undone §1.3.

| # | Decision | Answer | Decided in |
|---|---|---|---|
| 12.1 | May import-sourced and natively authored scenarios execute at all, and through which endpoint? | Yes, both, through all three launch paths. **A check in the shared launch validator is rejected** — it is consequence (a) of §1.2, and the gate stays on the plan path | §1.2, §1.3, §4.12 |
| 12.2 | Does an approved case stay executable after its source changes? | Yes, deliberately; the drift is reported. **Deactivating superseded projections is rejected** — §1.3: "do not fix this" | §1.3, §4.5 |
| 12.3 | Is there an approver grade distinct from a writer? | No. A third grade arrives at release 1.0.0 for protected environments and connector configuration instead | §1.3, §2 |
| 12.4 | Is the executable table one table with two row kinds, or two tables? | One table: every door converges on one stored scenario model. No per-project review-policy flag, since review is not enforced | §1.2, §1.3 |
| 12.5 | Does the spec/story pipeline terminate in canonical, or in executable rows? | In canonical | §1.2, §4.6 |
| 12.6 | Is the tenant boundary the organization or the project? | The organization, row-scoped; its column lands before the audit table | §3, §4.17 |
| 12.7 | Is platform ADMIN a vendor role or a customer role? | A vendor role, beside a separate customer organisation administrator | §2 |
| 12.8 | Are macros versioned? | Yes, and shared at organisation level once they are | §4.11 |
| 12.9 | Do locally authored and `.feature` cases ever sync back? | Never — they have no external identity, and the skip is reported, not silent | §4.7, §4.8, §4.14 |
| 12.10 | What is audited? | Three tiers by reader, in one append-only trail that records the kind of actor from its first migration — the one property that cannot be added later | §4.17 |
| 12.11 | Which step ids and patterns are public contract, and which locales? | All 515 ids and patterns are frozen (G34); the steps shown are the published catalog; display text in English, Spanish and French, patterns never translated | §4.10 |
| 12.12 | Is the hidden-step list a permanent boundary or temporary? | Hiding is a permanent policy, never silent. The six file steps stop being hidden with the fixture store; the eight conditional ones stay out permanently | §4.3, §4.10 |

### Safely deferrable to build — decided, and cheap to add later

TestRail, behind its three gates and with no data change; import schedules, already built and switched off,
arriving with §4.5's reporting condition; email and webhook notifications; additional AI capabilities and
providers; additional interface languages; observability depth; retention tuning; device-farm runs, if one
is bought.

**Not deferrable past a second instance:** the sign-in handoff and both two-factor caches must become
durable before two instances run, or federated and two-factor sign-in break (§3).

---

## 13. Traceability and the status of other documents

**Reliability order, established by audit:** code > Figma > `requirements.md`.

| Document | Status | Verdict |
|---|---|---|
| `requirements.md` | **Re-derived from this document on 2026-10-20** as numbered requirements by feature, module and screen | Follows this document and loses any disagreement with it. The 2026-10-01 FE baseline it replaced stays in ASpectra's Git history; its statuses were ~30% reliable and are not reused. |
| `.claude/CLAUDE.md` | Claude Code workspace memory; imports the canonical rule documents since 2026-10-20 | No longer restates scope. The product summary and the scope-as-configuration table live in ASpectra `docs/architecture/platform.md`, aligned with §1.1-§1.4 and §4.4 on 2026-10-20. |
| `ORCHESTRATION.md` | Claude Code adapter for the agent pipeline | Points at the canonical pipeline document and restates no scope, which removes the contradiction it carried. |
| Figma `p06HxSrh2afAdXopnk4Qvs` | 38 frames across 6 sections | **More current than `requirements.md`** — three frames depict shipped behaviour the document still lists as missing. |

> **Decision — every scope item is traceable, and drift is caught by configuration, not by reading.**

**Every scope item traces to a GitHub issue, its acceptance criteria, a test that proves each criterion, and
a commit with green gates** — and to a screen only where it has one. Most of that is already mandatory: the
backlog lives in GitHub issues per repository, a status claim needs a commit and green gates (G16), and the
pipeline's verification step repeats until every criterion is proven.

**No scope item exists without acceptance criteria.** An item without them cannot reach implementation at
all: the implementation step requires them as one of its three inputs.

**`requirements.md` is derived from this document**, requirement by requirement with the section each comes from; the earlier baseline is history in Git.

**Drift is detected automatically, by binding the answers to real configuration.** §13.1 places the scope
answers in the scope-as-configuration table of ASpectra `docs/architecture/platform.md` as configuration values rather than prose, precisely so that the
document and the running system cannot drift — the same standard by which every guardrail has an automated
guard.

**This document carries no built/offered/proven matrix.** §13.1 gives that record to the Release scope,
expressed as configuration values; a second copy here would be the next document to drift.

**Reconciling `CLAUDE.md` and `ORCHESTRATION.md` was done on 2026-10-20** with the owner's approval: both now import or point at the canonical documents and restate no scope. The landing page correction of §1.4 remains open.

### 13.1 Where the answers land

The scope answers belong in **ASpectra `docs/architecture/platform.md`, table "Scope as configuration"**, expressed
**in terms of the real configuration values** (`INTEGRATION_OFFERED_PROVIDERS`, `AI_ENABLED`, …) rather than
prose — so the document and the running system cannot drift. `.claude/CLAUDE.md` imports that document and
`ORCHESTRATION.md` points at it instead of repeating it.

---

## 14. Build vs. integrate

For every area: **(a)** we build and own it, **(b)** we integrate and own only the glue, **(c)** we build
now and plan to replace, **(d)** out of scope.

> **All pricing retrieved 2026-10-04, USD. Where a figure could not be confirmed, it says so.**
> Several specific figures remain unconfirmed and are marked inline — notably which paid Auth0 tier unlocks
> MFA, multi-parallel device-farm rates (all behind "contact sales"), PostHog's free tier, and whether
> Metabase's per-user charge applies to embedded viewers (that one swings its cost from $575 to $1,055/mo).

### 14.1 Identity, login, MFA, SSO — the owner's own example

**The premise does not survive contact with the pricing pages.** `[VERIFIED by research]`

- **Auth0's free tier excludes multi-factor authentication for production use.** A·Spectra already has
  working TOTP with recovery codes. Moving to Auth0's free tier would **lose a feature we already own**.
  *(Confirmed that it is not on Free; which paid tier unlocks it could not be reconciled.)*
- **Clerk's free tier also excludes MFA.**
- **The decisive cost variable is not user count — it is how enterprise SSO is metered, and the two models
  differ by about 100×:**

| Metering model | Vendors | Cost: 10 customers each federating their own identity provider |
|---|---|---|
| Per identity-provider connection ($75–125/mo each) | Auth0, WorkOS, Clerk, Stytch | **$625–1,250/month** |
| Per federated user ($0.015, first 50 free) | AWS Cognito, Google Identity Platform | **≈$0** |
| Unlimited | Keycloak (self-hosted) | **$0** licence |

| Option | Free tier reality | Plans |
|---|---|---|
| **Keycloak** (self-hosted, Apache 2.0) | Entirely free, no quotas. **Organizations supported since v26, with a different identity provider per organization** — exactly our multi-tenant shape | $0 licence; managed options from ~$25/mo (indicative, secondary sources) |
| **Stytch** | 10,000 users, **unlimited organizations, 5 SSO connections, MFA included**, "no feature gating" | $125 per SSO connection beyond 5; per-user overage above 10,000 not published |
| **AWS Cognito** | 10,000 users free; **federated users $0.015 each, first 50 free**; MFA on all tiers | $0.0055–0.020 per user per month |
| **WorkOS** | 1,000,000 users free | **$125 per SSO connection from the first** |
| **Auth0** | 25,000 users — **no production MFA** | B2B Essentials $150/mo + $100/connection; B2B Pro $800/mo |
| **Clerk** | 50,000 users — **no MFA** | Pro $25/mo; $75 per SSO connection after the first |

**Lock-in warning:** Auth0 does not include password hashes in its bulk export — they come only via a
support ticket, and reportedly not at all on free accounts. Migrating out means forcing every user to reset.

> **Decision — (a) build, and keep owning.** SAML is added through the framework's own SAML 2.0
> service-provider support, not through an identity vendor.
>
> **Rationale.** Identity is not one of §1.1's differentiators, but it is not plumbing to delegate either:
> §4.1 decides that identity is the product's own, and everything the vendors sell at a price is already
> built here — including the two-factor authentication the obvious vendors' free tiers exclude. Building
> SAML on the framework's own support keeps that true at no licence cost, with no per-connection metering,
> and keeps the actor record inside our own data.

**Authentication is not a differentiator, and it is not delegated.** Nothing that exists is removed: local
login, token issue and refresh, forced password change, reset, account lock, user administration, sign-up
activation, TOTP, recovery codes, step-up re-authentication and the OIDC layer all stay. WebAuthn is in
general scope and trusted devices are out (§4.1).

**The audit guarantee is unaffected**, because identity stays ours: who acted is recorded in our own trail
(§4.17), which is also why §2 refuses impersonation.

**The federation forecast no longer swings the cost.** The question of how many customers will federate their
own identity provider mattered only under per-connection vendor pricing. With SAML and OIDC built on the
framework, federation costs nothing per customer.

### 14.2 Secrets management

**Research verdict: do not replace this.** The single-value enum already in the code is the correct design —
a seam, not a gap. Every managed option either bills **per human seat** (the wrong meter for per-customer
machine credentials) or adds a hard runtime dependency on the path every connector call already depends on.
Free tiers are uniformly unusable: Doppler 3 users, Infisical 5 identities, 1Password none, and **HashiCorp's
lightweight managed product with a free tier has been discontinued**.

Cost if ever forced: **AWS Secrets Manager ≈$20/month** at 50 credentials.

> **Decision — (a) keep building.** The store stays ours, and the customer's own secret manager is a
> supported source through the same port (§4.3).
>
> **Rationale.** The single-value enum is a seam, not a gap, and the managed options bill per human seat —
> the wrong meter for per-customer machine credentials.

### 14.3 File and evidence storage — **the highest value-per-dollar item in this analysis**

**Two distinct needs:**
1. **Evidence storage** — today evidence is on the backend's local disk, which is also why a second
   instance cannot serve it.
2. **A fixture store** — this is why **6 file-upload test steps are disabled**. The steps already exist in
   the engine and are frozen; **no engine change is required**. What is missing is a project-scoped file
   store.

| Option | Free tier reality | Price |
|---|---|---|
| **Cloudflare R2** | **Perpetual: 10 GB, 1M/10M operations, and egress is free** | $0.015/GB-month, **no bandwidth charges** |
| Backblaze B2 | 10 GB always free, free egress up to 3× stored | $6.95/TB/month |
| AWS S3 | 5 GB for 12 months only | $0.023/GB-month + **$0.09/GB egress** |
| MinIO (self-hosted) | free | the right choice for local development, mirroring how Mailhog already serves email |

**Cost at our scale: under $3/month in every case.** Build **one** storage port against the S3-compatible
API and all four become configuration rather than integrations.

> **Decision — (b) integrate, through one storage port over the S3-compatible API.**
>
> **Rationale.** One port makes every provider configuration rather than an integration, and the same store
> serves both needs — evidence (§4.13, §9.2) and the per-project fixture store (§4.3). MinIO serves
> self-hosted and local development, the way Mailhog already serves email; the hosted provider is a
> provisioning choice at under three dollars a month.

**File upload testing is a committed capability** — §4.3 commits it and §4.10 unhides its steps.

### 14.4 Transactional email

**We are not building SMTP — we already have it. What we must not build is deliverability** (sender
authentication, bounce and complaint handling, reputation).

| Option | Free tier reality | Price |
|---|---|---|
| **Resend** | **3,000/month with a hard 100/day cap**, 30-day logs | Pro $20/mo (50,000) |
| Mailgun | 100/day, 1-day log retention | $15/mo (10,000) |
| AWS SES | **no SES-specific free tier for new accounts** | $0.10/1,000 à la carte |
| SendGrid | **free tier removed in 2025** | from $19.95/mo |
| Postmark | 100/month — unusable | $15/mo (10,000) |

**Cost: $0/month** at our volume. The limit that bites first is the **100/day cap**, not the monthly one.

> **Decision — (b) integrate a deliverability provider on the hosted SKU; the customer's own SMTP on
> self-hosted.**
>
> **Rationale.** The SMTP code stays ours; what must not be built is deliverability. A self-hosted customer
> sends through their own mail infrastructure, which is also the only way their mail stays inside their
> network. The hosted provider is a provisioning choice at no cost at our volume.

### 14.5 Audit logging — **build, and consolidate**

**The defect is not that we built it. It is that we built it five times.**

**Why no vendor can win here, stated precisely:** the audit row must commit **in the same database
transaction as the domain change** that caused it, or the trail is a best-effort story about what probably
happened. Every managed option is an HTTP call that can fail after the domain commit or succeed before a
rollback. And the read path must be scoped per customer by the authorisation model we already have and
already test — moving reads out means re-implementing that in a second system, in a different language,
tested by a different suite. That is a security regression dressed as a procurement win.

Volume removes the usual reason to buy entirely: ~20–30k events/month, 30–60 MB of JSON.

| Option | Reality |
|---|---|
| **WorkOS Audit Logs** | the one correctly-shaped funded option — and it publishes **no free allowance**: $125/mo per streaming connection + $99/mo per 1M events retained |
| AWS CloudTrail Lake | under $1/mo at this volume, but it is an investigation archive, **not a per-customer read API** |
| Grafana Loki | disqualified by 14-day free retention alone — a compliance trail is measured in years |
| Panther | $50,000/yr floor against ~50 MB/month |
| Retraced, BoxyHQ | right shape on paper, thin projects; worth reading their event schema before designing ours |

**Free-tier reality in one line:** free tiers are abundant here, and **not one is an audit system of record
with a transactionally-consistent write and a per-customer read API** — so the free tier is free of exactly
the thing we need.

**The consolidation, concretely:** one append-only `audit_event` table (time, actor, action as an enum,
target, redacted before/after payload, correlation id), **with no UPDATE or DELETE grant for the
application role** — append-only enforced at the grant level, not by convention. §4.17 refines it in four
ways the code forces: it is scoped by **organisation**, with the project optional, rather than by project;
the actor is a stable string rather than a foreign key; the target is a real column; and the kind of actor
is recorded from the first migration. One organisation-scoped read API, and one export endpoint as the
answer to "give us our audit log". Buying instead would have left all five
writers in place and added a sixth destination to keep consistent.

**Cost: $0 incremental.** The cost is one-time engineering. *(Note: this is a new forward migration with its
undo, it touches credential and auth auditing so it falls inside the security review scope, and it fails
every "trivial" sizing test — it is a full pipeline task, not a tidy-up.)*

> **Decision — (a) build, and consolidate**, as §4.17 specifies.
>
> **Rationale.** No vendor can commit in the same transaction as the domain change, and moving the read path
> out would re-implement our tenancy rules in a second system.

### 14.6 Mobile devices — the only area where integration costs real money

**And the only one that buys a capability we structurally cannot build: physical iOS.**

| Option | Free tier reality | Price |
|---|---|---|
| **BrowserStack App Automate** | trial only (100 minutes) | **$175/month per parallel session** (annual), unlimited minutes |
| Sauce Labs | trial only (60 min / 28 days) | $199/month per parallel |
| TestMu AI (formerly LambdaTest) | **100 minutes *lifetime*** | $199/month |
| AWS Device Farm | 1,000 device-minutes once | $0.17/device-minute — but **architecturally incompatible** with our in-process engine except through a session mode with a 150-minute cap |
| ~~Firebase Test Lab~~ | **disqualified** — deprecated, shutting down 2027-09-30, and **never supported Appium** | — |

**There is no genuinely free mobile device farm.** Cost at our scale: **≈$525/month at 3 parallel sessions**,
**$175–199 at one.** The cost is driven by **peak concurrency, not volume**.

> **Decision — the owner's, and the frame is settled.** Without a device farm, mobile runs on the customer's
> own devices through the agent (§6.4), at no cost to us. A device farm is bought only to claim physical
> iOS, which the product cannot build.

**Still open:**

- `[BLOCKING]` **Owner decision — whether physical iOS is a sales blocker today or a roadmap item**, which is
  the entire justification for this spend. *(Same decision as §6.4.)*
- `[BLOCKING]` **Owner decision — the real peak concurrency**, if a device farm is bought: it sets the bill,
  not the monthly run count.

### 14.7 Browser infrastructure and CI entry points

**Browsers: keep owning.** A self-hosted Playwright server is free, uses a supported Java API, and is one
small step from today's design. Note two disqualifications: **Microsoft's Playwright service successor has
no Java client**, and Sauce Labs' web path would invert control of execution away from our own engine.
The marginal cost of also integrating web becomes ≈$0 if BrowserStack is already bought for mobile.

**CI entry point: nothing exists.** No CLI, no token, no webhook.

> **Decision — browsers: (a) keep owning.** The warm per-engine pool is the customer-side runner (§9.1).
> **Decision — CI entry point: build**, at release MVP-1.0.1, as §4.18 designs it.
>
> **Rationale.** §1.4 defers machine access rather than refusing it, so declaring the product
> human-triggered only is already closed.

### 14.8 Analytics and dashboards — **keep the build**

Three reasons decide it:

1. **A BI tool does not replace what we built.** The flakiness index and the nightly pre-computation are
   **domain computations, not chart types**, and both survive every vendor unchanged, so buying would only
   cover chart rendering — the cheapest part. (Requirement coverage was a third, and §4.15 removes it.)
2. **Every white-label path is paid.** The free tiers are internal-BI tiers.
3. **Embedding adds a SECOND permission model.** Our "authorise before loading, cross-project ids return
   not-found" rule would have to be mirrored into vendor-configured sandboxing or row-level rules. *Two
   systems that must agree about tenancy is the largest hidden cost here.*

| Option | Free tier reality | Price |
|---|---|---|
| **Apache Superset** (self-hosted) | **Apache 2.0 — no branding or embedding restriction at all.** The only genuinely free customer-facing white-label embed | $0 licence; ~$20–40/mo hosted, or $0 co-located |
| Metabase | Open Source $0 unlimited users but **no interactive embedding, no white-label, no row permissions** | Pro **$575/mo** is the first tier with all three; ~$1,055/mo with 50 users |
| Preset (managed Superset) | Starter $0 up to 5 users but **excludes embedded dashboards** | $20/user/mo + a **$500/mo** embed add-on |
| Grafana Cloud | **caps at 3 visualization users** — a non-starter at 50 | ~$395/mo with 50 viewers, still no white-label |
| PostHog | free tier reported but **not vendor-confirmed** | solves a *different* problem: which console features customers actually use |

> ⚠ **Two licence traps worth recording in scope:** Metabase Open Source is AGPL — embedding it without
> making our own console AGPL requires keeping the "Powered by Metabase" badge visible. **Removing that
> badge is one CSS rule away from a compliance incident.** Grafana is AGPL too: unmodified use is fine, any
> bundled plugin is not, and removing its branding requires Enterprise.

**Superset's embedded SDK is production-usable and actively maintained**, but deliberately low-level: guest
tokens do not map to our roles, row-level filtering is whatever our backend injects, and tokens expire in
five minutes so refresh must be built. *That is actually the right shape — tenancy stays in code we own and
already test, instead of being duplicated into vendor UI rules.*

**Cost: $0 incremental to keep the build.**

> **Decision — (a) keep the build.** Superset is the path only if a customer asks for self-service BI.
>
> **Rationale.** Embedding a vendor adds a second permission model that must agree with ours about
> tenancy, which is the largest hidden cost in the area.

### 14.9 Hosting, high availability and the execution runner

> **Decision — build out-of-process execution, and own real-time delivery.**

**Out-of-process execution is built, as the generalised agent** over the existing mobile-agent wire, with no
engine contract change — §3's one piece of infrastructure for private networks, the element inspector, run
durability and the warm browser pool. A workflow product is rejected: the decided answer is infrastructure
the product is already half-way through for other decided reasons.

**Real-time delivery is owned.** The console's live updates already fan out across instances through the
database, switched off by default and switched on whenever a second instance runs (§9.2); every
subscription is authenticated and authorised by us (G14). A managed relay would add a dependency for a
problem that is already solved.

**Still open:**

- `[non-blocking]` **Owner decision — container orchestration and autoscaling, or a single managed host**, for
  the hosted SKU. *(A hosting spend; the product requires only that a second instance can run.)*

### 14.10 Connector maintenance

**Every connector is ours, permanently** (§4.4): a customer-written adapter is third-party code in the
import path, reaching the customer's credentials and our canonical model. The maintenance tax is accepted
and kept small by the shared contract suite every adapter passes against its simulator (G26).

### 14.11 AI providers

**Verdict: keep the current abstraction and change nothing structural.** It is the strongest seam in the
codebase. At our volume the model bill is **$0.20–13/month**; self-hosting costs **$200–470/month** in GPU,
i.e. 15–400× more. **Price must not be the deciding variable here** — schema-enforcement strength and data
residency should be.

| Provider | Per million tokens (in/out) | Schema enforcement |
|---|---|---|
| OpenAI `gpt-5-nano` | $0.05 / $0.40 | strongest documented guarantee |
| Anthropic Haiku 4.5 | $1 / $5 | genuinely enforced; cheapest cache reads |
| Anthropic Sonnet 5.5 | $2 / $10 | genuinely enforced |
| Google Gemini Flash-Lite | $0.10 / $0.40 | **weakest** — a documented subset of JSON Schema only |
| Ollama (self-hosted) | $0 marginal | client-side validation recommended |

> **Decision — (b) keep the current abstraction, and change nothing structural.**
>
> **Rationale.** It is the strongest seam in the codebase. The provider is chosen on schema-enforcement
> strength and data residency, not on price.

Whether a customer may bring their own model key is the same question as who pays for model usage, which is
§8's open owner decision.

### 14.12 Translation and content management

**The locale files stay hand-maintained, and the step-alias file keeps its human sign-off.** Three languages
of 2,517 keys each, held equal by a test, do not yet justify a platform; and a model may never write a step
alias, because patterns are frozen (G34), so any platform would have to preserve the sign-off anyway. This
is revisited when a fourth language is added.

### 14.13 Error tracking and monitoring

| Option | Free tier reality | Price |
|---|---|---|
| **Sentry** | 5,000 errors, 30-day retention — **but exactly 1 user**. The seat cap, not volume, forces payment | Team $26/month, unlimited users |
| **Grafana Cloud** | genuinely generous: 10,000 metric series, 50 GB logs, 14-day retention, 3 users | $19/month platform fee + usage |
| Better Stack | 10 monitors, 1 status page free | $45/month for logs |
| Datadog | **near-useless**: 1-day retention, no logs, no tracing | $15–31/host/month; **worst migration position** |

**Cost: $0 to start, $26/month once more than one person needs error access.**

> ⚠ **Before configuring anything:** sending logs or errors to a third party **moves log text across a trust
> boundary**. Our redaction must cover that path, or a connector credential inside a stack trace leaves the
> building. This is the highest-risk item in this section.

> **Decision — (b) integrate, on the hosted SKU only, and only behind the redactor.**
>
> **Rationale.** Nothing is sent until every log line and error leaving the deployment has passed the single
> redactor (G23) — this is the highest-risk item in the section. On self-hosted nothing leaves at all: the
> customer holds the keys and we cannot read their data (§3), so support there is the redacted diagnostics
> bundle.

### 14.14 Billing, licensing and metering

**Zero code exists**, so this is a pure build-vs-integrate decision with no sunk cost.

**Usage metering, if there is any, needs its own durable ledger.** The inbound rate-limit counters are
in-memory per instance, and shared runtime state belongs in the database (working rule 8), so they cannot be
reused.

**Still open:**

- `[BLOCKING]` **Owner decision — build billing and metering, or integrate a subscription provider.** *(Strictly
  downstream of §10's commercial model.)*

### 14.15 Feature flags

**The deployment switches stay for vendor-level gating, and an organisation-scoped settings layer is added
beside them** — the settings screen of §4.17. It is not a feature-flag product: six settings do not need
one, and a second system deciding what a customer may use would be a second authorisation model.

### 14.16 Summary — cost at 10 customers, 50 users, 1,000 runs/month

| Area | Verdict | Cost |
|---|---|---|
| Identity | **build** — SAML on the framework's own support | $0 |
| Secrets | **keep building** | $0 |
| File storage | **integrate**, one S3-compatible port | **under $3** |
| Email | integrate on hosted; the customer's SMTP on self-hosted | $0 |
| Observability | integrate, hosted only, behind the redactor | $0, then $26 |
| Analytics | **keep the build** | $0 |
| Audit | **build and consolidate** | $0 |
| Mobile devices | **owner decision** — bought only to claim physical iOS | $0, or $175–525 |
| Browsers | keep owning | $0 |
| LLM | keep the current abstraction | $0.20–13 |

**Total ≈$0–30/month without a device farm, and ≈$200–550 with one.** Every area but one can be built or
integrated for approximately nothing; the one that costs money buys the one capability the product cannot
build.

---

## 15. Appendix — facts to state once, and never re-ask

Facts about the code, re-verified on 2026-10-19. They describe the product **as it is**; the sections cited
are where a decision changes them. Several facts of the original appendix were wrong or had gone stale and
are corrected here rather than kept.

1. **Five paths create executable test content**, not one: the TMS connector and `.feature` import, both
   reviewed; spec/story drafts, with a three-state lifecycle and no approver recorded (§4.6 removes it); the
   scenario builder; and Gherkin paste, through a different code path again (§1.2 re-routes it).
2. **Only the TMS connector path is asynchronous and durable.** The other four complete inside the HTTP
   request.
3. **Three launch paths; one has a review gate**, and §1.3 keeps it that way.
4. **A superseded approval stays runnable**, and §1.3 keeps that too.
5. **There is no approver grade** — project permissions are `READ` and `READ_WRITE`. A third grade arrives at
   release 1.0.0 (§1.3).
6. **Only connector-imported cases launched through the plan path ever sync back.** A run of only locally
   authored or `.feature` cases creates no sync job. *Corrected:* in a mixed run the excluded cases are
   **not** reported — the exclusion note is built and then discarded by the sync worker, so nothing reaches
   the user (§4.14).
7. **Spec/story imports produce step-less drafts when AI is off**, named from the source with zero confidence
   — a documented deterministic fallback, not an error.
8. **515 steps exist and 501 are shown**; 14 are hidden — 6 pending a file store, 8 because they assert
   nothing, six of those being THEN steps.
9. **File upload testing works in the engine and is hidden from the product**, because there is nowhere to
   put the file (§4.3).
10. **Physical Android is supported; physical iOS is not**, and the product cannot build it.
11. *Corrected:* **PostgreSQL, SQL Server and Oracle have all been executed against end to end**, since
    2026-10-15. MySQL's driver ships and is unproven; H2's does not ship.
12. **GraphQL and SOAP are advertised by the engine with zero implementation**, and a test locks the claim in
    (§4.10 removes it).
13. **There is no distributed CLI, no API token and no webhook** — no machine entry point at all — and the
    OpenAPI document is not served in production.
14. *Corrected:* **nothing is sent when a run finishes.** Email works — activation, password reset, welcome,
    email-change notices — and is off by default; the run-result email exists and has no caller.
15. **Organizations are not a tenant boundary on project data** — `projects` has no organization column —
    though they already bound permission grants.
16. **Platform ADMIN is a vendor-wide superuser** across every customer.
17. **An in-flight run that outlasts the 120-second shutdown drain is lost on every deploy.**
18. **With two instances, evidence written by one is not readable by the other**, and the sync then ships
    results without it.
19. **The public site contradicts itself about supported connectors, three ways, in three languages** — it
    claims one connector and three at once, and the one-connector claim wrongly denies Xray Cloud.
20. **Macros are not versioned** — changing one silently changes every scenario using it, although the
    reference count that would warn about it already exists.
21. **Permission grants are not audited**, and their log lines do not record who made the change.
22. **The review trail is never read, and is deleted with its case.**
23. **Parallel scenarios within one run do not exist**; parallel HTTP requests within a scenario and four
    parallel runs per instance do.
24. **A managed run accepts Chromium only**; Firefox and WebKit are rejected where private hosts are denied,
    which is the production default.
25. **The HTML report carries no screenshots**, and its stack-trace slot is always empty.
26. **A run's step detail is never purged**; only screenshots and reports expire.
27. **A SHA-256 is stored for every artifact and never verified.**
28. **The scheduled-run flag is displayed and never set** — every launch path hardcodes it to false.
29. **The result written back names the case version current at sync time**, not the version that ran.
