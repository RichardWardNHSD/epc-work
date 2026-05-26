# Observability — ODIN-First Approach

## Overview

This document describes an observability approach for the Endpoint Catalog Service
using **ODIN** as the primary observability platform from day one, with AWS CloudWatch
retained only for operational automation (alarms triggering scaling, Lambda-level
error detection) rather than as the primary monitoring and dashboarding layer.

ODIN is NHS England's centralised observability platform. It provides best-in-class
dashboarding, alerting, and supports all OpenTelemetry signal types (logs, metrics,
traces, profiles). It is recommended for all services including National Infrastructure
and High Availability Services.

---

## Strategy alignment

The NHS England Tech Radar for Observability (draft, May 2026) positions ODIN as the
primary choice for all services. This approach adopts that recommendation directly
rather than building a CloudWatch-first stack with a future migration path.

| Layer | Tool | Purpose |
|-------|------|---------|
| Centralised observability | ODIN (Grafana + Loki + Mimir + Tempo) | Dashboarding, alerting, log querying, cross-service correlation |
| Telemetry collection | OpenTelemetry Collector (Lambda layer) | Collects and exports logs, metrics, traces to ODIN |
| Operational automation | AWS CloudWatch Alarms (minimal) | Auto-scaling triggers, Lambda dead-letter alerting |
| Tracing | OpenTelemetry → ODIN (Tempo) | Distributed traces across API Gateway → Lambda → DynamoDB |
| Metrics | OpenTelemetry → ODIN (Mimir) | Custom and runtime metrics |
| Logs | OpenTelemetry → ODIN (Loki) | Structured application logs |
| ITOC visibility | ODIN (shared dashboards/alerts) | Cross-service health reporting |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  AWS Account — Endpoint Catalog Service                         │
│                                                                 │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │API Gateway│───►│  Lambda Function │───►│    DynamoDB      │  │
│  └──────────┘    │  + OTel Layer    │    └──────────────────┘  │
│                  └────────┬─────────┘                           │
│                           │                                     │
│                           │ OTel Protocol (OTLP)                │
│                           ▼                                     │
│                  ┌──────────────────┐                           │
│                  │  OTel Collector  │                           │
│                  │  (Lambda Layer)  │                           │
│                  └────────┬─────────┘                           │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │ OTLP/gRPC or OTLP/HTTP
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  ODIN Platform                                                  │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐ │
│  │  Loki   │    │  Mimir  │    │  Tempo  │    │   Grafana   │ │
│  │ (Logs)  │    │(Metrics)│    │(Traces) │    │(Dashboards) │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Alerting (Grafana Alerting / Mimir Ruler)                  ││
│  │  → SNS / PagerDuty / Email / Teams                          ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Telemetry collection

### OpenTelemetry Lambda Layer

Each Lambda function includes the AWS OpenTelemetry Lambda Layer which:

- Auto-instruments the Lambda runtime (Python/Node)
- Captures traces (function invocation, DynamoDB calls, HTTP calls)
- Captures runtime metrics (duration, memory, cold starts)
- Exports all signals via OTLP to the ODIN collector endpoint

Configuration is via environment variables on the Lambda:

| Variable | Value |
|----------|-------|
| `OTEL_SERVICE_NAME` | `epc-endpoint-handler` (per function) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | ODIN's OTLP ingestion URL |
| `OTEL_EXPORTER_OTLP_HEADERS` | Authentication token for ODIN |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.namespace=endpoint-catalog,deployment.environment=production` |
| `OTEL_TRACES_SAMPLER` | `parentbased_traceidratio` |
| `OTEL_TRACES_SAMPLER_ARG` | `1.0` (100% in production — adjust if volume requires) |

### Structured logs

Application logs are emitted as structured JSON and collected by the OTel layer for
export to Loki. The log format:

```json
{
  "timestamp": "2026-05-26T10:00:00.000Z",
  "level": "INFO",
  "traceId": "abc123def456...",
  "spanId": "789ghi...",
  "requestId": "c1ab3fba-6bae-4ba4-b257-5a87c44d4a91",
  "correlationId": "9562466f-c982-4bd5-bb0e-255e9f5e6689",
  "odsCode": "R778",
  "operation": "POST /Endpoint",
  "resourceType": "Endpoint",
  "resourceId": "e1a2b3c4-0000-0000-0000-000000000001",
  "statusCode": 200,
  "duration": 45,
  "message": "Endpoint created successfully"
}
```

The `traceId` and `spanId` fields enable direct correlation between logs and traces
in Grafana — clicking a log line jumps to the associated trace.

### Custom metrics

Custom business metrics are emitted via the OpenTelemetry Metrics API and exported
to Mimir:

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `epc.endpoint.created` | Counter | Endpoints created | `connection_type`, `payload_type` |
| `epc.endpoint.status_change` | Counter | Status transitions | `from_status`, `to_status` |
| `epc.search.duration` | Histogram | Search resolution time | `search_type` |
| `epc.search.results` | Histogram | Number of results returned | `search_type` |
| `epc.visibility.excluded` | Counter | Endpoints excluded by filtering | `reason` |
| `epc.duplicate.rejected` | Counter | Writes rejected (duplicate) | `resource_type` |
| `epc.auth.denied` | Counter | Requests rejected (ownership) | `operation` |
| `epc.request.duration` | Histogram | Total request duration | `method`, `path`, `status_code` |

---

## Dashboards (Grafana via ODIN)

### Service overview dashboard

| Panel | Query (LogQL / PromQL) | Visualisation |
|-------|------------------------|---------------|
| Request rate | `sum(rate(epc.request.duration_count[5m]))` | Time series |
| Error rate | `sum(rate(epc.request.duration_count{status_code=~"5.."}[5m])) / sum(rate(epc.request.duration_count[5m]))` | Time series + threshold |
| P50/P90/P99 latency | `histogram_quantile(0.9, sum(rate(epc.request.duration_bucket[5m])) by (le))` | Time series |
| Endpoints created | `sum(increase(epc.endpoint.created[1h]))` | Stat panel |
| Status changes | `sum by (to_status)(increase(epc.endpoint.status_change[1h]))` | Bar chart |
| Duplicate rejections | `sum(increase(epc.duplicate.rejected[1h]))` | Stat panel |
| Auth denials | `sum(increase(epc.auth.denied[1h]))` | Stat panel |
| Recent errors | `{service_name="epc-*"} |= "ERROR"` | Log panel |

### Trace explorer

ODIN's Tempo integration allows:
- Search traces by `requestId` or `correlationId`
- Filter by duration (find slow requests)
- View the full call chain: API Gateway → Lambda → DynamoDB
- Identify cold start impact on latency
- Correlate traces with logs (jump from trace span to associated log lines)

### Cross-service dashboard (future)

When BaRS Proxy also exports to ODIN, a cross-service dashboard can show:
- BaRS Proxy → Endpoint Catalog lookup latency
- End-to-end referral routing time (Proxy lookup + forward to receiver)
- Correlation of Proxy errors with Catalog availability

---

## Alerting (Grafana Alerting via ODIN)

| Alert | Condition | For | Severity | Notification |
|-------|-----------|-----|----------|--------------|
| High 5XX rate | `5XX rate > 1%` | 5 min | Critical | SNS → team + ITOC |
| P90 latency breach | `P90 > 100ms` | 5 min | Warning | SNS → team |
| Lambda errors | `epc.request.duration_count{status_code="500"} > 0` | 1 min | Critical | SNS → team |
| Sustained 4XX rate | `4XX rate > 10%` | 5 min | Warning | SNS → team |
| No traffic | `sum(rate(epc.request.duration_count[10m])) == 0` | 10 min | Critical | SNS → team + ITOC |
| Duplicate spike | `sum(increase(epc.duplicate.rejected[5m])) > 10` | 5 min | Warning | SNS → team |

Alerts route to the team via SNS (email/Slack/Teams). Critical alerts also forward
to ITOC for 24/7 coverage.

---

## ITOC visibility

With ODIN as the primary platform, ITOC visibility is achieved natively:

1. **Shared dashboards** — ITOC has read access to the EPC service dashboard in Grafana
2. **Shared alerts** — critical alerts route to both the team and ITOC notification channels
3. **Log access** — ITOC can query EPC logs directly in Grafana/Loki using the service
   namespace filter `{service_namespace="endpoint-catalog"}`
4. **No separate log forwarding pipeline needed** — ODIN is already the centralised
   platform that ITOC uses

This eliminates the need for CloudWatch Logs subscription filters, cross-account log
delivery, or separate ITOC integration work.

---

## What remains in CloudWatch

CloudWatch is retained only for AWS-native operational automation that cannot be
replaced by ODIN:

| Use | Why CloudWatch |
|-----|----------------|
| Lambda dead-letter queue alarm | Triggers Lambda retry/alerting natively |
| DynamoDB auto-scaling | CloudWatch metrics drive DynamoDB capacity scaling |
| API Gateway throttling metrics | Used by AWS WAF and rate limiting |
| CloudWatch Logs (short-term) | Lambda writes here by default; OTel layer reads and exports |

CloudWatch Logs retention is set to **7 days** (minimum for OTel layer to read and
export). ODIN is the long-term store.

---

## Comparison with CloudWatch-first approach

| Aspect | CloudWatch-first | ODIN-first |
|--------|-----------------|------------|
| Dashboarding | Basic (CloudWatch Dashboards) | Best-in-class (Grafana) |
| Alerting | CloudWatch Alarms (per-metric) | Grafana Alerting (multi-signal, templated) |
| Cross-service correlation | Not possible without ODIN | Native — all services in one platform |
| Log querying | CloudWatch Insights (pay-per-query, limited) | Loki (LogQL, fast, included) |
| Tracing | X-Ray (AWS-only, 30-day retention) | Tempo (OpenTelemetry native, configurable retention) |
| ITOC integration | Requires subscription filters + cross-account delivery | Native — shared platform |
| Vendor lock-in | AWS-specific APIs and formats | OpenTelemetry standard — portable |
| Migration effort | Build now, migrate later | Build once |
| Cost model | Pay per metric, query, alarm, log GB | Included in ODIN platform (funding TBD) |

---

## Requirements traceability

| Requirement | How addressed |
|---|---|
| EPC-NF06: Audit all access attempts, forward to CSOC | Structured logs in Loki; CSOC accesses via ODIN shared dashboards or dedicated export |
| EPC-NF07: Operational logging, forward to ITOC | Logs in Loki; ITOC queries directly via Grafana |
| EPC-NF08: Intentional log events conforming to agreed design | Structured JSON format with trace correlation; consistent across all functions |
| EPC-NF09: Platinum service class | Grafana alerting with < 5 min detection; 24/7 ITOC visibility; cross-service traces |
| EPC-NF02: API response time 100ms at P90 | `epc.request.duration` histogram with P90 alert at 100ms |

---

## Prerequisites and open items

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | ODIN onboarding — register EPC as a service, obtain OTLP endpoint and auth token | EPC team + ODIN team | TBD |
| 2 | Confirm ODIN supports AWS Lambda OTel layer export (OTLP/HTTP) | ODIN team | TBD |
| 3 | Confirm ODIN funding model for EPC (included in platform cost or per-service charge) | Programme | TBD |
| 4 | Confirm CSOC access pattern (shared Grafana dashboard or dedicated log export) | Security team | TBD |
| 5 | Confirm ITOC already uses ODIN for service health (avoids duplicate integration) | ITOC team | TBD |
| 6 | Agree metric naming conventions and label cardinality limits with ODIN team | EPC + ODIN | TBD |
| 7 | Confirm trace sampling strategy for production volume | EPC team | TBD |

---

## Recommendation

If ODIN onboarding is available before the EPC goes live, adopt this ODIN-first approach.
It avoids building a CloudWatch observability stack that will need migrating later, gives
immediate cross-service visibility, and aligns with the Tech Radar recommendation.

If ODIN onboarding is not available in time, use the [CloudWatch-first approach](./observability.md)
as an interim measure with a planned migration to ODIN post-launch.
