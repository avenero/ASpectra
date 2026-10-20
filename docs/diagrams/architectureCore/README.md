# qa-platformCore — architecture

Deep-dive architecture of `qa-platformCore`, the deterministic test engine of A·Spectra. Java 21,
Gradle multi-module, six modules, about 59,200 lines of main Java source (2026-10-20).

Status: Current-verified (code, 2026-10-03; spot-checked 2026-10-20) | Owner: TBD

Anything not shown here: the code, `qa-platformCore/AGENTS.md` and `qa-platformCore/.claude/CLAUDE.md`, and
the shared [platform.md](../../architecture/platform.md).

LOC and catalog-entry counts drift between sessions — every number below is dated. Regenerate with
`wc -l` / `COMPONENTS.md` / `cli-smoke.sh` before quoting a figure externally.

| File | Scope |
|---|---|
| [01-modules-and-runtime.md](01-modules-and-runtime.md) | The 6 modules and the plugin-host runtime, from `ServiceLoader` discovery to the stateless engine |
| [02-transports-and-execution.md](02-transports-and-execution.md) | Per-run execution internals, the two transports, the mobile-agent, and the frozen consumer-surface contract |
| [03-step-catalog.md](03-step-catalog.md) | The three step-id namespaces, discovery/resolution, and the frozen pattern contract |
| [04-security-guardrails.md](04-security-guardrails.md) | How Core stays deterministic and AI-free under untrusted step input (G11, G18, G19, G20) |

## Decided changes that touch Core

From [alcance.md](../../alcance.md); none of them changes a frozen step pattern or id (G34).

| Change | Requirements |
|---|---|
| The GraphQL and SOAP capability advertisement, and the test that locks it in, are removed | CAT-07 |
| Hidden steps are never silently absent; the six file-dependent steps return with the fixture store | CAT-02, CAT-03 |
| Managed runs accept Firefox and WebKit as well as Chromium | WEB-02 |
| The dead Selenium Grid configuration is removed | WEB-04 |
| Evidence adds the SQL, the logs and a DOM snapshot on a Web failure | EVD-03 |
| The warm browser pool lives in the customer-side runner, never in Core (G19) | NFR-03 |
| The Core README's statement that SQL Server and Oracle are unproven is outdated: both run end to end since 2026-10-15 | DB-01 |
