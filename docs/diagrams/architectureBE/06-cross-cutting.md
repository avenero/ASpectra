# Cross-cutting

| Concern | Mechanism |
|---|---|
| Errors | `application/problem+json`, English `detail`, stable `code` extension (`ProblemResponseCodeGuardTest`) |
| AuthZ | One access guard per BC; `UserProjectPermission` READ / READ_WRITE; cross-project ids 404 |
| Secrets | `integration_credentials`, one encrypted row per `CredentialKind`, `CredentialCipherPort` |
| Outbound | `OutboundUrlPolicy` -> Core `OutboundHostPolicy`, on save and before each call, no redirects |
| Observability | `X-Correlation-ID` in MDC and async; Micrometer; resilience4j breakers and bulkheads |
| Runtime | `spring.threads.virtual.enabled: true` — blocking I/O parks a virtual thread, not a pooled one |
| Scale | Single instance in Stage 1: in-memory STOMP broker, local artifact disk; rate limiting is already cluster-aware (see below) |

## AI-specific rows

| Concern | Mechanism |
|---|---|
| AI kill switch | `ai.enabled` (global, default false) + `ai.capabilities.<id>.enabled` (per-capability, default true except `failure-triage`, default false) |
| AI budget | Durable, `ai_budgets`/`ai_budget_reservations`, UTC calendar month, reserve-before-call/reconcile-after, swept by `AiBudgetSweeper` |
| AI rate limit | `FixedWindowRateLimiter`, shared with `ConnectorRateLimiter` — cluster-aware, not pure in-memory, per capability+project+minute |
| AI audit | `ai_invocations`, hashes only, one row per invocation including refusals, purged by `AiInvocationRetentionJob` on its own schedule |
| AI egress policy | `projects.ai_egress_posture`, project-write-settable, enforced in `ModelRouter`, never model-inferred |
| AI provider boot check | `AiMappingStartupCheck` — fails startup if a tier/pin resolves to a provider that can't satisfy a required feature; separate from `CapabilityRegistry`'s own startup check (prompt/schema/referential-check existence) |
| AI outbound | The same single `OutboundUrlPolicy` -> Core `OutboundHostPolicy` — `OutboundUrlAllowedRule` is a referential check, not a parallel policy |
| AI redaction | The same single `SensitiveText` redactor, applied at every text boundary: input before prompt, stream deltas, execution-record error text, connector field samples |
