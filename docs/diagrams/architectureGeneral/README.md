# General architecture

Status: Current-verified (code) unless a section says Intended | Owner: TBD

Macro-level, cross-repository view of A·Spectra. Diagram nodes marked as today's state are traceable to
real files in `qa-platformCore`, `qa-platformBE` or `qa-platformFE`; target-state diagrams are labelled
Intended and come from [alcance.md](../../alcance.md). Shared facts not shown here are in
[platform.md](../../architecture/platform.md); each repository's own rules are in its `AGENTS.md`.

| File | Scope |
|---|---|
| [00-entry-doors-and-runtime.md](00-entry-doors-and-runtime.md) | The five entry doors, the two stored models and three launch paths today, how they diverge from the decided scope, and the target runtime with the customer-side runner |
| [01-overlay-flow.md](01-overlay-flow.md) | The connector door end to end (the 8-stage loop), how each stage hands off to the next, and the three independent state machines |
| [02-critical-flows.md](02-critical-flows.md) | One grounded diagram per critical flow: Jira/Xray import, Postman import, Swagger/OpenAPI import, user-story import, an AI-assisted review path, and a mobile execution run |
| [03-intelligence-overview.md](03-intelligence-overview.md) | Macro placement of the `intelligence` capability across FE/BE/Core and its governance posture |

Per-repository internals live in [architectureCore](../architectureCore/README.md),
[architectureBE](../architectureBE/README.md) and [architectureFE](../architectureFE/README.md).
