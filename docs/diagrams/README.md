# Architecture diagrams

Verified, code-grounded architecture of A·Spectra: the execution-overlay product spanning
`qa-platformCore`, `qa-platformBE` and `qa-platformFE`. Every diagram node in these folders is traceable
to a real file — nothing here is proposed or aspirational, and the former "proposed AI architecture"
document has been folded in as current, implemented architecture (see below). Renders directly in
GitHub, GitLab, VS Code and IntelliJ.

There is no `architecture.md` companion document anywhere in this workspace — any earlier reference to
one was a dead link and has been removed. The source of truth for anything not shown in a diagram is the
relevant repository's own `.claude/CLAUDE.md`, plus the workspace root `.claude/CLAUDE.md`.

## How this is organized

| Folder | Scope |
|---|---|
| [architectureGeneral/](architectureGeneral/README.md) | The whole solution at macro level: the 8-stage overlay loop, critical example flows (Jira/Xray, Postman, Swagger/OpenAPI, user-story imports, an AI-assisted review path, a mobile run), and where the `intelligence` capability sits across repos |
| [architectureCore/](architectureCore/README.md) | `qa-platformCore` internals: modules, the plugin-host runtime, transports, the step catalog, and the guardrails that keep AI out |
| [architectureBE/](architectureBE/README.md) | `qa-platformBE` internals: bounded contexts, the overlay BC, the launch pipeline and durable-job patterns, result sync, the `intelligence` bounded context, cross-cutting concerns |
| [architectureFE/](architectureFE/README.md) | `qa-platformFE` internals: the four lint-enforced rings, the overlay feature, state/errors/i18n/tests, the `assistant` feature |

Start with `architectureGeneral/` for the product-level picture, then open the folder for the repository
you need to go deeper on.

## What changed in this rewrite (2026-10-03)

This folder previously held five flat files (`01-general-architecture.md` through
`05-proposed-ai-architecture.md`). They have been replaced by the four folders above, each holding
several focused files instead of one dense page per repository. Two corrections drove the restructure,
both verified directly against the code by independent review of each repository:

- **The `intelligence` bounded context (BE) and the `assistant` feature (FE) are implemented, not
  proposed.** The former `05-proposed-ai-architecture.md` described a design as "PROPOSED, not
  implemented." It was not: 97 files under `qa-platformBE/.../intelligence/` and 34 files under
  `qa-platformFE/src/features/assistant/` implement it today, each with its own architecture test
  coverage (`BoundedContextRulesTest` on the BE side, declared `eslint.config.js` boundary rules on the
  FE side). It is governed and off by default (`AI_ENABLED=false`), which is a deployment posture, not
  an implementation gap. This content now lives as current architecture in `architectureBE/
  05-intelligence-bounded-context.md`, `architectureFE/04-assistant-feature.md`, and
  `architectureGeneral/03-intelligence-overview.md`.
- **A companion `architecture.md` was cited throughout the old files and does not exist anywhere in this
  workspace.** Every reference to it has been removed; citations now point to the real source (each
  repository's own `.claude/CLAUDE.md`) or to the code directly.

Several other stale facts, caught during the same review, are corrected in place rather than carried
forward silently — notably: TestRail's connector is fully built in both BE and FE and is withheld from
the 1.0 connector picker by a single FE allowlist, not missing; several Core step-audit findings from
2026-09-22 are resolved in current code; FE's overlay feature and i18n key count have grown
substantially since they were last measured; and the AI capability pipeline's real capability count is
11, not the 12 the old proposed doc listed.
