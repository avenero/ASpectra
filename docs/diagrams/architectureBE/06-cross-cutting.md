# Cross-cutting

Status: Current-verified (code, 2026-10-03; spot-checked 2026-10-20) | Owner: TBD

| Concern | Mechanism |
|---|---|
| Errors | `application/problem+json`, English `detail`, stable `code` extension (`ProblemResponseCodeGuardTest`) |
| AuthZ | One access guard per BC; `UserProjectPermission` READ / READ_WRITE; cross-project ids 404 |
| Secrets | `integration_credentials`, one encrypted row per `CredentialKind`, `CredentialCipherPort` |
| Outbound | `OutboundUrlPolicy` -> Core `OutboundHostPolicy`, on save and before each call, no redirects |
| Observability | `X-Correlation-ID` in MDC and async; Micrometer; resilience4j breakers and bulkheads |
| Runtime | `spring.threads.virtual.enabled: true` — blocking I/O parks a virtual thread, not a pooled one |
| Scale | Single instance today: in-memory STOMP broker, evidence on local disk (a second instance answers 404), runs inside the backend JVM, OIDC handoff and two-factor state in instance memory. The outbound limiter (connectors, AI) is database-backed and shared; the inbound HTTP limiter is in memory per instance. Target: alcance §3 and §9.2 (EVD-09, IAM-12, NFR-05, NFR-06) |

## AI-specific rows

| Concern | Mechanism |
|---|---|
| AI kill switch | `ai.enabled` (global, default false) + `ai.capabilities.<id>.enabled` (per-capability, default true except `failure-triage`, default false) |
| AI budget | Durable, `ai_budgets`/`ai_budget_reservations`, UTC calendar month, reserve-before-call/reconcile-after, swept by `AiBudgetSweeper` |
| AI rate limit | `FixedWindowRateLimiter`, shared with `ConnectorRateLimiter` — cluster-aware, not pure in-memory, per capability+project+minute |
| AI audit | `ai_invocations`, hashes only, one row per invocation including refusals, purged by `AiInvocationRetentionJob` on its own schedule |
| AI egress policy | `projects.ai_egress_posture`, project-write-settable, enforced in `ModelRouter`, never model-inferred. Decided target: an organisation setting changed only by the organisation administrator (ORG-12) |
| AI provider boot check | `AiMappingStartupCheck` — fails startup if a tier/pin resolves to a provider that can't satisfy a required feature; separate from `CapabilityRegistry`'s own startup check (prompt/schema/referential-check existence) |
| AI outbound | The same single `OutboundUrlPolicy` -> Core `OutboundHostPolicy` — `OutboundUrlAllowedRule` is a referential check, not a parallel policy |
| AI redaction | The same single `SensitiveText` redactor, applied at every text boundary: input before prompt, stream deltas, execution-record error text, connector field samples |
