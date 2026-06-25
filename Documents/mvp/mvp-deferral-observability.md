# EPC MVP Deferral — Observability (ODIN)

**Series:** EPC MVP Scope Deferrals
**Document:** 2 of N

---

## Requirement references

| Requirement | Description |
|---|---|
| EPC-NF06 | Audit all access attempts and actions, forward to CSOC |
| EPC-NF07 | Operational logging for incident troubleshooting, forward to ITOC |
| EPC-NF08 | Intentional log events conforming to agreed design |
| EPC-NF09 | Platinum service class for APIs |

---

## Purpose

This document analyses the deferral of ODIN-based observability from the EPC MVP. The
full ODIN integration remains in scope for the final solution — the MVP will use native
AWS CloudWatch as the observability layer with a planned migration path to ODIN.

---

## What is ODIN

ODIN is NHS England's centralised observability platform built on Grafana, Loki, Mimir,
and Tempo. It provides dashboarding, alerting, log querying, distributed tracing, and
cross-service correlation. The NHS England Tech Radar (draft, May 2026) positions ODIN
as the primary choice for all services.

---

## What is being deferred

| Deferred capability | Detail |
|---|---|
| OpenTelemetry Lambda Layer | OTel instrumentation layer on each Lambda function for automatic trace/metric/log collection |
| OTLP export to ODIN | Exporting logs, metrics, and traces via OpenTelemetry Protocol to ODIN's ingestion endpoints |
| Grafana dashboards | Best-in-class dashboarding in ODIN's centralised Grafana instance |
| Grafana Alerting | Multi-signal alerting via ODIN's Grafana Alerting / Mimir Ruler |
| Distributed tracing (Tempo) | OpenTelemetry-native traces stored in ODIN's Tempo backend |
| Centralised log querying (Loki) | Structured logs in Loki with LogQL querying |
| Custom metrics in Mimir | OpenTelemetry metrics exported to ODIN's Mimir backend |
| Cross-service correlation | Ability to correlate EPC traces/logs with BaRS Proxy and other services in a single view |
| Native ITOC visibility | ITOC accessing EPC health via shared Grafana dashboards (no separate forwarding pipeline) |
| ODIN onboarding | Service registration, OTLP endpoint provisioning, auth token setup |

---

## What is retained for MVP

| Retained capability | Detail |
|---|---|
| CloudWatch Logs | Structured JSON logs from all Lambda functions and API Gateway. 90-day retention. |
| CloudWatch Metrics | Standard AWS metrics (API Gateway, Lambda, DynamoDB) + custom business metrics via Embedded Metric Format. |
| CloudWatch Alarms | Error rate, latency, Lambda errors, DynamoDB throttling, sustained 4XX. Routes to SNS → team. |
| CloudWatch Dashboards | Service overview dashboard (request volume, error rate, latency percentiles, DynamoDB capacity). |
| AWS X-Ray | End-to-end request tracing (API Gateway → Lambda → DynamoDB). 30-day retention. |
| ITOC log forwarding | CloudWatch Logs subscription filter forwarding ERROR/CRITICAL logs to ITOC destination. |
| Structured log format | JSON structured logs with requestId, correlationId, odsCode, operation, statusCode, duration. |
| Custom business metrics | EndpointCreated, EndpointStatusChange, SearchLatency, DuplicateRejected, AuthorisationDenied. |

---

## Implications of deferral

#### 1. No cross-service correlation

**Impact: Medium**

Without ODIN, the EPC cannot correlate its traces and logs with other services (e.g. BaRS
Proxy) in a single view. Debugging cross-service issues requires manually matching
`correlationId` values across separate CloudWatch Log Groups in different AWS accounts.

| With ODIN | Without ODIN (MVP) |
|---|---|
| Single Grafana view showing BaRS Proxy → EPC lookup → response, with latency breakdown at each hop | Separate CloudWatch consoles; manual correlationId search across accounts |

**Mitigation:** The structured log format includes `correlationId` on every log entry.
Cross-service debugging is possible but manual. This is acceptable for MVP traffic volumes.

---

#### 2. CloudWatch dashboarding is less capable

**Impact: Low**

CloudWatch Dashboards are functional but lack Grafana's flexibility — no variables,
limited panel types, no log-to-trace linking, no cross-account views in a single
dashboard.

| With ODIN | Without ODIN (MVP) |
|---|---|
| Grafana dashboards with variables, drill-down, log-to-trace links, cross-service panels | CloudWatch Dashboards — static panels, single-account view, basic visualisations |

**Mitigation:** CloudWatch Dashboards cover the essential operational panels (request
volume, error rate, latency, DynamoDB capacity). The team can troubleshoot effectively.
The experience is less polished but functionally adequate.

---

#### 3. X-Ray tracing has 30-day retention and limited querying

**Impact: Low**

X-Ray traces are retained for 30 days only. Querying is limited to the X-Ray console —
no LogQL equivalent, no correlation with metrics in a single view.

| With ODIN | Without ODIN (MVP) |
|---|---|
| Tempo traces with configurable retention, TraceQL querying, direct log correlation | X-Ray traces with 30-day retention, console-based querying, no log linking |

**Mitigation:** 30 days is sufficient for operational troubleshooting. Historical trace
analysis is a nice-to-have, not a requirement for MVP.

---

#### 4. ITOC integration requires a separate forwarding pipeline

**Impact: Medium**

Without ODIN, ITOC cannot simply access shared Grafana dashboards. Instead, a CloudWatch
Logs subscription filter must be configured to forward logs cross-account to ITOC's
aggregation destination.

| With ODIN | Without ODIN (MVP) |
|---|---|
| ITOC accesses EPC health natively in Grafana (shared dashboards, shared alerts) | CloudWatch Logs subscription filter → cross-account delivery (Kinesis Firehose or Logs destination) to ITOC |

**Mitigation:** CloudWatch cross-account log forwarding is a well-understood pattern.
It requires additional infrastructure (subscription filter + delivery destination) but
is straightforward to implement. This infrastructure is decommissioned when ODIN is
delivered.

---

#### 5. Alerting is per-metric rather than multi-signal

**Impact: Low**

CloudWatch Alarms are single-metric. Grafana Alerting can combine multiple signals in
a single alert rule (e.g. "error rate > 1% AND latency > 100ms AND traffic > baseline").

| With ODIN | Without ODIN (MVP) |
|---|---|
| Multi-signal alert rules with templating and silencing | Single-metric alarms; separate alarm per condition |

**Mitigation:** The MVP alarm set covers the critical scenarios individually. Composite
alerting (reducing alert fatigue by correlating multiple signals) is a quality-of-life
improvement, not a functional gap.

---

#### 6. Vendor lock-in to AWS observability

**Impact: Low (temporary)**

The MVP builds on CloudWatch, X-Ray, and EMF (Embedded Metric Format) — all AWS-specific.
The ODIN approach uses OpenTelemetry, which is vendor-neutral.

**Mitigation:** This lock-in is temporary and planned. The migration path is documented.
When ODIN is delivered, the OTel Lambda Layer replaces EMF for metrics, OTel replaces
X-Ray for traces, and logs are exported to Loki. The CloudWatch infrastructure is then
retained only for operational automation (auto-scaling, dead-letter alarms).

---

## API changes for MVP

#### Unchanged between MVP and final solution

| Aspect | Detail |
|---|---|
| **Structured log format** | Same JSON structure in both. Only the transport changes (CloudWatch → OTel → Loki). |
| **Custom business metrics** | Same metrics emitted in both. MVP uses EMF; final uses OTel Metrics API. Metric names and dimensions are aligned to enable migration without dashboard rework. |
| **Request/correlation ID propagation** | `X-Request-Id` and `X-Correlation-Id` are captured and logged in both approaches. |
| **Audit logging** | Same audit events, same fields. Transport to CSOC/ITOC changes, not the content. |
| **Alarm thresholds** | Same thresholds (1% error rate, 100ms P90 latency, etc.) apply in both. |

#### What changes when ODIN is delivered

| Item | MVP (CloudWatch) | Final (ODIN) |
|---|---|---|
| **Telemetry collection** | CloudWatch Logs + EMF | OpenTelemetry Lambda Layer (OTLP export) |
| **Log storage** | CloudWatch Logs (90 days) | Loki (configurable retention) |
| **Metric storage** | CloudWatch Metrics | Mimir |
| **Trace storage** | X-Ray (30 days) | Tempo (configurable retention) |
| **Dashboards** | CloudWatch Dashboards | Grafana (ODIN) |
| **Alerting engine** | CloudWatch Alarms → SNS | Grafana Alerting → SNS / PagerDuty / Teams |
| **ITOC integration** | Subscription filter → cross-account forwarding | Shared Grafana dashboards (native) |
| **CSOC access** | Subscription filter → CSOC destination | Shared Grafana or dedicated Loki export |
| **Cross-service traces** | Not possible | Native in Tempo |

#### Migration effort (MVP → Final)

| Step | Effort | Detail |
|---|---|---|
| Add OTel Lambda Layer | Small | Add Lambda layer + environment variables per function |
| Configure OTLP endpoint | Small | Point at ODIN's ingestion URL; add auth token |
| Retire EMF metric emission | Small | Replace EMF calls with OTel Metrics API calls |
| Disable X-Ray, rely on OTel traces | Small | Remove X-Ray active tracing config |
| Build Grafana dashboards | Medium | Recreate dashboard panels in Grafana (metric names are pre-aligned) |
| Configure Grafana alerts | Medium | Recreate alarm rules in Grafana Alerting |
| Retire CloudWatch Dashboards | Small | Delete dashboards (retain alarms for auto-scaling) |
| Retire ITOC subscription filter | Small | Decommission cross-account forwarding |
| Reduce CloudWatch Logs retention | Small | Drop from 90 days to 7 days (OTel reads and exports) |

**Total migration effort:** Medium. Primarily configuration and dashboard recreation — no
application logic changes required. The structured log format and metric names are designed
to be the same in both approaches, minimising rework.

---

## Decision

| Decision | Rationale |
|---|---|
| **Defer ODIN from MVP; use CloudWatch** | ODIN onboarding prerequisites (service registration, OTLP endpoint, auth token, funding confirmation) are unresolved. CloudWatch provides adequate observability for MVP operations. The structured log format and metric names are pre-aligned to ODIN conventions so migration is low-friction when ODIN becomes available. |

---

## When to deliver

ODIN integration should be delivered when:

1. **ODIN onboarding is available** — service registration complete, OTLP endpoint
   provisioned, auth token issued
2. **Funding model confirmed** — ODIN usage cost for EPC is agreed
3. **ITOC confirms ODIN as their access path** — no point integrating if ITOC still
   requires CloudWatch forwarding anyway
4. **BaRS Proxy also exports to ODIN** — cross-service correlation only has value when
   both services are in the same platform

---

## Summary: MVP vs final observability

| Aspect | MVP (CloudWatch) | Final (ODIN) |
|---|---|---|
| Operational monitoring | ✅ Fully functional | ✅ Enhanced (Grafana) |
| Alerting | ✅ Per-metric alarms | ✅ Multi-signal rules |
| Troubleshooting | ✅ Logs + X-Ray | ✅ Logs + Traces + Metrics correlated |
| ITOC visibility | ✅ Via subscription filter | ✅ Native shared dashboards |
| Cross-service correlation | ❌ Manual only | ✅ Native |
| Vendor portability | ❌ AWS-specific | ✅ OpenTelemetry standard |

---

## Related documents

| Document | Description |
|----------|-------------|
| [Observability — ODIN-First Approach](./observability-odin.md) | Full ODIN design (target state) |
| [Observability — CloudWatch](./observability.md) | CloudWatch-first design (MVP implementation) |
| [EPC MVP Deferral — RBAC](./mvp-deferral-rbac.md) | Companion deferral document |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [EPC MVP Deferral — RBAC](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | This document |
