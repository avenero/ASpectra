# qa-platformCore — architecture

Deep-dive architecture of `qa-platformCore`, the deterministic test engine of A·Spectra. Java 21,
Gradle multi-module. There is no `architecture.md` in this workspace; the source of truth for anything
not shown here is `qa-platformCore/.claude/CLAUDE.md` and the code itself.

LOC and catalog-entry counts drift between sessions — every number below is dated. Regenerate with
`wc -l` / `COMPONENTS.md` / `cli-smoke.sh` before quoting a figure externally.

| File | Scope |
|---|---|
| [01-modules-and-runtime.md](01-modules-and-runtime.md) | The 6 modules and the plugin-host runtime, from `ServiceLoader` discovery to the stateless engine |
| [02-transports-and-execution.md](02-transports-and-execution.md) | Per-run execution internals, the two transports, the mobile-agent, and the frozen consumer-surface contract |
| [03-step-catalog.md](03-step-catalog.md) | The three step-id namespaces, discovery/resolution, and the frozen pattern contract |
| [04-security-guardrails.md](04-security-guardrails.md) | How Core stays deterministic and AI-free under untrusted step input (G11, G18, G19, G20) |
