# qa-platformFE — architecture

React + TypeScript + Vite + TanStack Query + i18next + Tailwind. The single-page console, about
43,200 lines of TypeScript under `src/` excluding tests (2026-10-20).

Status: Current-verified (code, 2026-10-03; counts refreshed 2026-10-20) | Owner: TBD

Anything not shown here: the code, `qa-platformFE/AGENTS.md` and `qa-platformFE/.claude/CLAUDE.md`, and the
shared [platform.md](../../architecture/platform.md). Counts drift; regenerate before quoting one.

| File | Scope |
|---|---|
| [01-rings-and-modules.md](01-rings-and-modules.md) | The four lint-enforced rings, the 12 feature modules and their declared cross-imports |
| [02-overlay-feature.md](02-overlay-feature.md) | The `overlay` feature in detail: page, VM, components, hooks |
| [03-state-errors-i18n-tests.md](03-state-errors-i18n-tests.md) | State, errors, i18n, tests, and the console's accessibility/design-token contracts |
| [04-assistant-feature.md](04-assistant-feature.md) | The `assistant` feature, fully current (not proposed): every AI-assisted panel and hook in the console |

## Decided changes that reshape the console

From [alcance.md](../../alcance.md) §5; per-screen detail in Part 3 of [requirements.md](../../requirements.md).

| Change | Requirements |
|---|---|
| The global dashboard leaves the customer sidebar; the project dashboard is reached from the project | DSH-01, DSH-02 |
| Requirement coverage is removed (card, types, test, i18n keys) | DSH-03 |
| Project settings `general` gains name and description; `execution-defaults` is removed | PRJ-02 |
| Offered connectors come from organisation settings instead of the hard-coded `OFFERED_CONNECTOR_PROVIDERS` | INT-03, INT-15, SET-01 |
| New surfaces: organisation settings, audit view, element inspector in the scenario builder, fixture upload control | SET-01, AUD-05, WEB-06, ENV-08 |
| Two-factor gains WebAuthn; the trusted-devices tab is not built | IAM-05, IAM-07 |
| The notification bell shows real deliveries or is removed | NTF-03 |
| The end-to-end gate adds Firefox and WebKit | NFR-12 |
