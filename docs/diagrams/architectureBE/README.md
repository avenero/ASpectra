# qa-platformBE — architecture

Spring Boot 3.5, Java 21 (virtual threads on), hexagonal architecture, PostgreSQL via Flyway. The
overlay backend: it owns every integration and all AI in the product. There is no `architecture.md` in
this workspace; the source of truth for anything not shown here is `qa-platformBE/.claude/CLAUDE.md`
and the code itself.

| File | Scope |
|---|---|
| [01-bounded-contexts.md](01-bounded-contexts.md) | The bounded-context map, legacy coexistence, ArchUnit enforcement — `intelligence` included as a first-class BC |
| [02-overlay-bounded-context.md](02-overlay-bounded-context.md) | The `testcase` BC in detail: api/application/domain/infrastructure |
| [03-launch-pipeline-and-jobs.md](03-launch-pipeline-and-jobs.md) | The single launch pipeline and the two durable-job shapes (single-phase and multi-phase row claims) |
| [04-result-sync.md](04-result-sync.md) | Outbox to sync job to TMS, at-least-once with idempotent consumers |
| [05-intelligence-bounded-context.md](05-intelligence-bounded-context.md) | The `intelligence` BC, fully current (not proposed): pipeline, capabilities, governance, provider adapters |
| [06-cross-cutting.md](06-cross-cutting.md) | Errors, authz, secrets, outbound, observability, runtime, scale |
