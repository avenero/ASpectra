# General architecture

Macro-level, cross-repository view of the A·Spectra execution overlay. Every diagram node in this
folder is traceable to a real file in `qa-platformCore`, `qa-platformBE` or `qa-platformFE` — nothing
here is proposed or aspirational. There is no `architecture.md` companion document in this workspace;
the source of truth for anything not shown here is each repository's own `.claude/CLAUDE.md` plus the
workspace root `.claude/CLAUDE.md`.

| File | Scope |
|---|---|
| [01-overlay-flow.md](01-overlay-flow.md) | The 8-stage overlay loop, how each stage hands off to the next, and the three independent state machines |
| [02-critical-flows.md](02-critical-flows.md) | One grounded diagram per critical flow: Jira/Xray import, Postman import, Swagger/OpenAPI import, user-story import, an AI-assisted review path, and a mobile execution run |
| [03-intelligence-overview.md](03-intelligence-overview.md) | Macro placement of the `intelligence` capability across FE/BE/Core and its governance posture |

Per-repository internals live in [architectureCore](../architectureCore/README.md),
[architectureBE](../architectureBE/README.md) and [architectureFE](../architectureFE/README.md).
