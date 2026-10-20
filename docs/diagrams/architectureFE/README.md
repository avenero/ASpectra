# qa-platformFE — architecture

React + TypeScript + Vite + TanStack Query + i18next + Tailwind. The single-page console. There is no
`architecture.md` in this workspace; the source of truth for anything not shown here is
`qa-platformFE/.claude/CLAUDE.md` and the code itself.

File counts, LOC and test counts below are dated (2026-10-03) and drift between sessions —
regenerate with `find`/`wc -l` before quoting a figure externally.

| File | Scope |
|---|---|
| [01-rings-and-modules.md](01-rings-and-modules.md) | The four lint-enforced rings, the 12 feature modules and their declared cross-imports |
| [02-overlay-feature.md](02-overlay-feature.md) | The `overlay` feature in detail: page, VM, components, hooks |
| [03-state-errors-i18n-tests.md](03-state-errors-i18n-tests.md) | State, errors, i18n, tests, and the console's accessibility/design-token contracts |
| [04-assistant-feature.md](04-assistant-feature.md) | The `assistant` feature, fully current (not proposed): every AI-assisted panel and hook in the console |
