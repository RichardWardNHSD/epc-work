# Observability

## Overview

The Endpoint Catalog Service requires observability to support live operations,
incident troubleshooting, and compliance with NHS England non-functional requirements
(EPC-NF06 Auditing, EPC-NF07 Logging, EPC-NF08 Log events).

This document describes the observability approach using **native AWS components**
(CloudWatch) as the primary local observability layer, with a forward path to the
NHS England centralised observability platform (ODIN) for cross-service visibility.

---

## Strategy alignment

The NHS England Tech Radar for Observability (draft, May 2026) identifies four options.
Our approach uses **Cloud Native** (AWS CloudWatch) as the local service observability
layer, positioned as a complement to ODIN for centralised cross-service observability
when that integration becomes available.

| Layer | Tool | Purpose |
|-------|------|---------|
| Local service observability | AWS CloudWatch (Logs, Metrics, Alarms, Dashboards) | Real-time operational monitoring, automated actions, ITOC log forwarding |
| Centralised observability | ODIN (future) | Cross-service traces, centralised dashboarding, alerting at scale |
| Structured logging | CloudWatch Logs with JSON structured format | Queryable logs, metric filters, log-based alarms |
| Metrics | CloudWatch Metrics + custom metrics via EMF | API latency, error rates, throughput, business metrics |
| Alerting | CloudWatch Alarms → SNS | Team notifications, escalation to ITOC |
| Tracing | AWS X-Ray | Request tracing through API Gateway → Lambda → DynamoDB |

---

## Components

### CloudWatch Logs

All Lambda functions and API Gateway stages emit structured JSON logs to CloudWatch
Log Groups.

| Log Group | Source | Retention |
|-----------|--------|-----------|
| `/aws/lambda/epc-endpoint-handler` | Endpoint CRUD Lambda | 90 days |
| `/aws/lambda/epc-healthcareservice-handler` | HealthcareService CRUD Lambda | 90 days |
| `/aws/lambda/epc-list-handler` | List CRUD Lambda | 90 days |
| `/aws/apigateway/epc-api` | API Gateway access logs | 90 days |
| `/aws/lambda/epc-audit-handler` | Audit event processing Lambda | 365 days |

#### Structured log format

All application logs use JSON structured format for queryability:

```json
{
  "timestamp": "2026-05-26T10:00:00.000Z",
  "level": "INFO",
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

### CloudWatch Metrics

#### Standard metrics (automatic)

| Metric | Source | Namespace |
|--------|--------|-----------|
| `Count` | API Gateway | `AWS/ApiGateway` |
| `4XXError` | API Gateway | `AWS/ApiGateway` |
| `5XXError` | API Gateway | `AWS/ApiGateway` |
| `Latency` | API Gateway | `AWS/ApiGateway` |
| `IntegrationLatency` | API Gateway | `AWS/ApiGateway` |
| `Duration` | Lambda | `AWS/Lambda` |
| `Errors` | Lambda | `AWS/Lambda` |
| `Throttles` | Lambda | `AWS/Lambda` |
| `ConcurrentExecutions` | Lambda | `AWS/Lambda` |
| `ConsumedReadCapacityUnits` | DynamoDB | `AWS/DynamoDB` |
| `ConsumedWriteCapacityUnits` | DynamoDB | `AWS/DynamoDB` |
| `ThrottledRequests` | DynamoDB | `AWS/DynamoDB` |

#### Custom metrics (via CloudWatch Embedded Metric Format)

| Metric | Description | Dimensions |
|--------|-------------|------------|
| `EndpointCreated` | Count of Endpoints created | `connectionType`, `payloadType` |
| `EndpointStatusChange` | Count of status transitions | `fromStatus`, `toStatus` |
| `SearchLatency` | Time to resolve Endpoint search | `searchType` (has, include, template) |
| `VisibilityFilterExcluded` | Endpoints excluded by status/period filtering | `reason` (status, period, template) |
| `DuplicateRejected` | Write rejected due to duplicate detection | `resourceType` |
| `AuthorisationDenied` | Requests rejected by ownership check | `operation` |

### CloudWatch Alarms

| Alarm | Metric | Condition | Period | Action |
|-------|--------|-----------|--------|--------|
| High error rate | API GW `5XXError` / `Count` | > 1% | 5 min | SNS → team notification |
| High latency | API GW `Latency` P90 | > 100ms | 5 min | SNS → team notification |
| Lambda errors | Lambda `Errors` | > 0 | 1 min | SNS → team notification |
| DynamoDB throttling | DynamoDB `ThrottledRequests` | > 0 | 1 min | SNS → team notification |
| Sustained 4XX rate | API GW `4XXError` / `Count` | > 10% | 5 min | SNS → team notification |

Alarm actions route to an SNS topic subscribed by the team. Critical alarms (sustained
5XX, DynamoDB throttling) should also forward to ITOC via the agreed integration path.

### AWS X-Ray (Tracing)

X-Ray is enabled on API Gateway and Lambda to provide end-to-end request tracing.
This gives visibility into:

- Time spent in API Gateway routing
- Lambda cold start vs warm execution
- DynamoDB query latency per operation
- Error propagation through the call chain

X-Ray traces are retained for 30 days. For longer-term trace analysis and cross-service
correlation, traces should be forwarded to ODIN when that integration is available.

### CloudWatch Dashboards

A service dashboard provides at-a-glance operational health:

| Panel | Metric | Visualisation |
|-------|--------|---------------|
| Request volume | API GW `Count` | Time series (5 min) |
| Error rate | `5XXError` / `Count` | Time series + threshold line |
| P50/P90/P99 latency | API GW `Latency` | Time series (percentiles) |
| Lambda duration | Lambda `Duration` | Time series (P50/P90) |
| DynamoDB capacity | Read/Write consumed vs provisioned | Time series |
| Active endpoints | Custom metric from periodic scan | Single value |
| Recent status changes | Custom metric `EndpointStatusChange` | Time series |

---

## ITOC visibility

Per the Tech Radar guidance, teams should make logs visible to ITOC for overall service
health reporting. The approach:

1. **CloudWatch Logs subscription filter** → forwards ERROR and CRITICAL level logs to
   the ITOC centralised log aggregation destination (cross-account delivery via
   CloudWatch Logs destination or Kinesis Data Firehose — TBC with ITOC team)
2. **CloudWatch Alarms** → critical alarms forward to ITOC SNS topic for their
   monitoring dashboards
3. **Structured log format** ensures ITOC can parse and correlate EPC logs with other
   services without custom parsing rules

---

## ODIN integration (future)

When ODIN integration becomes available, the EPC will:

1. **Export metrics** via OpenTelemetry Collector sidecar or CloudWatch metric stream
   to ODIN's metrics ingestion endpoint
2. **Export traces** via X-Ray → OpenTelemetry export to ODIN's trace backend
3. **Export logs** via CloudWatch Logs subscription to ODIN's log ingestion (Loki)
4. **Retire local CloudWatch dashboards** in favour of ODIN's centralised Grafana
   dashboards (retain CloudWatch alarms for automated actions)

This positions CloudWatch as the operational layer (alarms, automated scaling, immediate
troubleshooting) and ODIN as the analytical layer (cross-service correlation, long-term
trending, centralised alerting).

---

## Requirements traceability

| Requirement | How addressed |
|---|---|
| EPC-NF06: Audit all access attempts and actions, forward to CSOC | Structured logs capture all operations; subscription filter forwards to CSOC/ITOC |
| EPC-NF07: Operational logging for incident troubleshooting, forward to ITOC | CloudWatch Logs with structured JSON; cross-account forwarding to ITOC |
| EPC-NF08: Intentional log events conforming to agreed design | Structured log format defined above; consistent across all Lambdas |
| EPC-NF09: Platinum service class for APIs | CloudWatch Alarms with < 5 min detection; X-Ray tracing; 24/7 alerting via SNS |
| EPC-NF02: API response time 100ms at P90 | `Latency` P90 alarm at 100ms threshold; dashboard panel for continuous monitoring |

---

## Open items

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Confirm ITOC log forwarding mechanism (Logs destination vs Firehose) | Platform team | TBD |
| 2 | Confirm ODIN onboarding timeline and integration pattern | ODIN team | TBD |
| 3 | Confirm CSOC log forwarding requirements (format, destination, frequency) | Security team | TBD |
| 4 | Agree custom metric naming conventions with ODIN team | EPC + ODIN | TBD |
| 5 | Confirm funding model for ODIN usage | Programme | TBD |
| 6 | Define log retention beyond 90 days (archive to S3 Glacier?) | Architecture | TBD |
