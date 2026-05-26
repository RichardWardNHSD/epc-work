# Resilience and Availability

## Overview

This document describes how the Endpoint Catalog API achieves resilience and high
availability using AWS serverless components. The architecture is designed so that no
single component failure causes a full service outage, and the service degrades gracefully
under load or partial failure.

---

## Service Level Target

| Metric | Target | Notes |
|--------|--------|-------|
| Availability | 99.9% (three nines) | Equates to ~8.7 hours downtime per year |
| Planned maintenance windows | Zero | Serverless architecture allows zero-downtime deployments |
| Maximum acceptable response time (P95) | 150 ms | End-to-end, measured at API Gateway |

> **Open action:** Confirm the availability target with the service owner. If the service
> is classified as Gold/Tier 1, a 99.95% target may be required, which would necessitate
> multi-region deployment.

---

## Resilience by Component

### API Gateway

| Capability | How it helps | Configuration |
|------------|-------------|---------------|
| Multi-AZ by default | API Gateway is a fully managed, regionally distributed service. No single-AZ failure takes it down. | No configuration needed — built-in |
| Throttling | Protects backend from traffic spikes. Excess requests receive `429 Too Many Requests`. | Default: 10,000 req/s burst, 5,000 req/s steady. Custom limits can be set per stage/route. |
| WAF integration (optional) | Protects against DDoS, SQL injection, and malicious payloads | AWS WAF can be attached to the API Gateway stage |
| Caching (optional) | Reduces Lambda invocations for repeated read requests | API Gateway cache (0.5–237 GB). Useful for `GET /metadata` and `GET /_health`. |

### Lambda

| Capability | How it helps | Configuration |
|------------|-------------|---------------|
| Multi-AZ execution | Lambda automatically runs across multiple Availability Zones within the region | No configuration needed — built-in |
| Auto-scaling | Concurrency scales from 0 to thousands of instances based on demand | Default: 1,000 concurrent executions per account (can be raised) |
| Reserved concurrency | Guarantees capacity for critical functions even when other functions consume the account limit | Set per-function: e.g. 100 reserved for `epc-resource-handler` |
| Provisioned concurrency (optional) | Eliminates cold starts for latency-sensitive paths | Keeps N instances warm. **Strongly recommended** given the 150ms P95 target — cold starts (~1–3s) would breach it. |
| Retry behaviour | Synchronous invocations (from API GW) are not retried by Lambda — the client retries. This prevents duplicate writes. | N/A for sync invocations |
| Timeout | Prevents runaway executions from consuming concurrency | Set to 30 seconds (well above expected P99 of ~2s) |

### DynamoDB

| Capability | How it helps | Configuration |
|------------|-------------|---------------|
| Multi-AZ replication | Data is automatically replicated across 3 AZs within the region | No configuration needed — built-in |
| On-demand capacity | Scales read/write throughput automatically with no capacity planning | Billing mode: PAY_PER_REQUEST |
| Consistent performance | Single-digit millisecond latency at any scale | Built-in |
| Point-in-Time Recovery | Continuous backups allow restore to any second in the last 35 days | Enabled on both tables |
| Global Tables (optional) | Active-active multi-region replication for regional failover | Not enabled by default — see Multi-Region section in DR document |
| DAX (optional) | In-memory cache for microsecond read latency | Not required at current scale — consider if read volume exceeds 10k req/s |

### CloudWatch

| Capability | How it helps | Configuration |
|------------|-------------|---------------|
| Regional service | CloudWatch is multi-AZ within a region | Built-in |
| Alarm evaluation | Continues even if the monitored service is down | Alarms evaluate independently of the EPC service |
| Cross-account/cross-region (optional) | Centralised monitoring across environments | Can be configured if multi-account strategy is adopted |

---

## Failure Modes and Mitigation

| Failure | Detection | Mitigation | User Impact |
|---------|-----------|-----------|-------------|
| Single Lambda cold start | CloudWatch Duration metric spike | Provisioned concurrency (**strongly recommended** at 150ms target) | One request sees ~1–3s extra latency — breaches P95 target |
| Lambda throttling | CloudWatch Throttles metric > 0 | Reserved concurrency + account limit increase request | Affected requests receive 429; clients retry |
| DynamoDB throttling | CloudWatch ThrottledRequests > 0 | On-demand mode auto-scales; if sustained, check for hot partitions | Affected requests see elevated latency or 500 |
| DynamoDB single-AZ failure | Transparent to application | Automatic failover to another AZ (built-in) | None — transparent |
| API Gateway 5xx spike | CloudWatch 5XXError alarm | Investigate Lambda errors; rollback if deployment-related | Affected requests fail; clients retry |
| Downstream dependency timeout | Structured log with timeout error | Circuit breaker pattern in Lambda code (if downstream calls exist) | Graceful error response to client |
| Region-wide outage | External health check (Route 53 or Apigee) | Failover to secondary region (if multi-region enabled) | Full outage until failover completes |

---

## Graceful Degradation Strategy

The service should degrade gracefully rather than fail completely:

| Scenario | Degraded behaviour | Full failure avoided |
|----------|-------------------|---------------------|
| Audit table write fails | Log the failure, return 500 to client (do not silently drop the audit record) | Resource write is NOT committed without audit — consistency preserved |
| Health check dependency timeout | Report specific dependency as unhealthy; other checks still return | Load balancer can make informed routing decisions |
| High latency on DynamoDB | Lambda timeout at 10s returns 504 to client | Other concurrent requests are unaffected |
| CloudWatch log ingestion delayed | Application continues serving requests; logs arrive eventually | No user impact |

---

## Rate Limiting and Throttling

| Layer | Mechanism | Default | Configurable |
|-------|-----------|---------|-------------|
| Apigee (internet-facing) | Spike arrest / quota policies | TBD by platform team | Yes |
| API Gateway | Stage-level throttling | 10,000 burst / 5,000 steady | Yes, per stage and per route |
| Lambda | Concurrency limit | 1,000 account-wide | Yes, per function (reserved) |
| DynamoDB | On-demand auto-scaling | Adapts to traffic | Automatic |

Throttling responses:
- Apigee: `429 Too Many Requests` with `Retry-After` header
- API Gateway: `429 Too Many Requests`
- Lambda throttle (surfaced via API GW): `502 Bad Gateway` or `429`

Clients should implement exponential backoff with jitter on 429 and 5xx responses.

---

## Timeout Strategy

| Component | Timeout | Rationale |
|-----------|---------|-----------|
| API Gateway integration timeout | 29 seconds | Maximum allowed by API Gateway for Lambda proxy integration |
| Lambda function timeout | 10 seconds | Well above the 150ms target but allows for occasional slow DynamoDB calls without timing out prematurely |
| DynamoDB SDK timeout | 2 seconds per call | Prevents a single slow DynamoDB call from consuming the full Lambda timeout |
| Health check Lambda timeout | 5 seconds | Fast failure — health checks must respond quickly |
| Client-side recommendation | 5 seconds with retry | Documented in API consumer guidance |

---

## Idempotency

Write operations should be idempotent where possible to allow safe client retries:

| Operation | Idempotency mechanism |
|-----------|----------------------|
| `POST /Endpoint/$template` | Client-supplied `X-Request-Id` used as idempotency key. If a request with the same `X-Request-Id` is received within 5 minutes, return the original response. |
| `PUT /Endpoint/{id}` | Naturally idempotent — same payload produces same result |
| `PATCH /Endpoint/{id}` | Conditional update using `If-Match` (ETag / `meta.versionId`). Prevents lost updates on retry. |
| `DELETE /Endpoint/{id}` | Naturally idempotent — deleting an already-deleted resource returns 404 or 204 |

---

## Deployment Resilience

| Practice | Description |
|----------|-------------|
| Blue/green via Lambda aliases | `live` alias points to current version; new version deployed and validated before alias switch |
| Canary deployments (optional) | Route 10% of traffic to new version; promote after 5 minutes if error rate is zero |
| Automated rollback | CI/CD pipeline monitors CloudWatch alarms for 5 minutes post-deploy; auto-rolls back if any alarm fires |
| Infrastructure-as-code | All resources defined in Terraform — no manual changes in production |
| Separate environments | Sandbox → Integration → Production pipeline with identical infrastructure |

---

## Monitoring and Alerting (Cross-Reference)

Full details of monitoring, metrics, alerting, and dashboards are in the
[Audit & Observability Design](../endpoint-catalog-poc/.kiro/specs/endpoint-catalog-audit/design.md)
and [Requirements](../endpoint-catalog-poc/.kiro/specs/endpoint-catalog-audit/requirements.md)
(Requirements 10–15).

Key availability-related alerts:

| Alert | Threshold | Action |
|-------|-----------|--------|
| Availability below target | Error rate > 0.1% sustained for 15 minutes | Page on-call engineer |
| P95 latency breach | > 150 ms for 5 minutes | Notify team channel |
| Health check failing | 2 consecutive failures | Page on-call engineer |
| Lambda concurrency at limit | ConcurrentExecutions > 80% of reserved | Notify team — consider scaling |

---

## Availability Calculation

```
Availability = (Total minutes in period − Downtime minutes) / Total minutes in period × 100

Monthly (30 days):
  99.9% = max 43.2 minutes downtime
  99.95% = max 21.6 minutes downtime
  99.99% = max 4.3 minutes downtime
```

AWS publishes the following SLAs for the components used:

| Service | AWS SLA |
|---------|---------|
| API Gateway | 99.95% |
| Lambda | 99.95% |
| DynamoDB (on-demand, single-region) | 99.99% |
| DynamoDB (Global Tables) | 99.999% |

The composite availability of the stack (API GW × Lambda × DynamoDB) is:
`0.9995 × 0.9995 × 0.9999 = 99.89%`

This means the 99.9% target is achievable but tight — any additional application-level
failures will erode the margin. Multi-region deployment would provide significant headroom.

---

## Summary

| Principle | Implementation |
|-----------|---------------|
| No single point of failure | All components are multi-AZ by default |
| Auto-scaling | Lambda concurrency + DynamoDB on-demand |
| Graceful degradation | Timeouts, circuit breakers, clear error responses |
| Fast recovery | Lambda alias rollback (< 5 min), PITR restore (< 30 min) |
| Safe deployments | Blue/green, canary, automated rollback |
| Observable | CloudWatch metrics, alarms, dashboard (see observability requirements) |
| Idempotent | Safe client retries via X-Request-Id and conditional updates |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Disaster Recovery](./disaster-recovery.md) | RPO/RTO targets, backup strategy, restore procedures, DR testing |
| [Authorisation](./authorisation.md) | ODS-based ownership model and token validation |
| Audit & Observability Requirements | Requirements 10–15 covering logging, tracing, metrics, alerting, retention |
| Audit & Observability Design | AWS implementation of observability stack |
