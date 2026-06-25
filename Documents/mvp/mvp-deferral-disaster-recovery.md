# EPC MVP Deferral — Disaster Recovery (Full DR Plan)

**Series:** EPC MVP Scope Deferrals
**Document:** 4 of N

---

## Purpose

This document analyses the deferral of the full disaster recovery plan from the EPC MVP.
The comprehensive DR strategy (multi-region, tested failover, defined RPO/RTO, quarterly
DR exercises) remains in scope for the final solution — the MVP relies on inherent AWS
serverless resilience and basic protective measures.

---

## What is the full DR plan

The full disaster recovery design defines:
- Explicit RPO (< 5 minutes) and RTO (< 30 minutes) targets
- DynamoDB Point-in-Time Recovery (PITR) with tested restore procedures
- On-demand backup schedule (pre-deployment + weekly)
- Lambda versioning and alias-based rollback
- API Gateway stage rollback
- Multi-region active-passive option (DynamoDB Global Tables + Route 53 failover)
- Operational runbook for each failure scenario
- Quarterly DR testing cadence
- Alignment with NHS England Red Lines Backup Scenario

---

## What is being deferred

| Deferred capability | Detail |
|---|---|
| Confirmed RPO/RTO targets | Formal agreement on recovery objectives with service owner |
| Multi-region failover | DynamoDB Global Tables, secondary-region Lambda/API Gateway, Route 53 health-check failover |
| Quarterly DR testing | Scheduled PITR restore tests, backup restore validation, health check alarm simulation |
| Full operational runbook | Documented procedures for each failure scenario with estimated RTO and named owners |
| On-call rota and escalation path | Defined ownership for incident response |
| Pre-deployment on-demand backups | Automated DynamoDB snapshots before each deployment |
| Weekly scheduled backups | EventBridge-triggered weekly DynamoDB snapshots |
| S3 log archive with Glacier lifecycle | Long-term log retention beyond CloudWatch |
| Service tier classification | Formal Gold/Silver classification determining availability requirements |
| Red Lines compliance review | Cross-reference with NHS England Engineering CoE Red Lines Backup Scenario |

---

## What is retained for MVP

| Retained capability | Detail |
|---|---|
| DynamoDB Point-in-Time Recovery (PITR) | Enabled on all tables from day one. Provides continuous backup with ~5-minute granularity. This is a non-negotiable baseline. |
| DynamoDB Deletion Protection | Enabled on all tables. Prevents accidental table deletion. |
| Lambda versioning | Each deployment publishes a new version. Rollback is possible by repointing the alias. |
| API Gateway stage deployments | Each deployment creates a stage snapshot. Previous deployments can be redeployed. |
| Infrastructure-as-code (Terraform) | All configuration is reproducible from source. Full environment can be recreated from scratch. |
| Multi-AZ by default | DynamoDB, Lambda, and API Gateway all operate across multiple Availability Zones in eu-west-2 by default — no single-AZ dependency. |
| CI/CD pipeline with health checks | Post-deployment verification; failed deployments can be rolled back manually. |
| CloudWatch alarms | Detect failures (5XX rate, Lambda errors, DynamoDB throttling) for rapid awareness. |

---

## Why this can be deferred

### Inherent serverless resilience covers most scenarios

The EPC is built on fully managed AWS serverless components. These provide significant
built-in resilience without any DR-specific configuration:

| Component | Inherent resilience |
|---|---|
| DynamoDB | Multi-AZ synchronous replication, automatic failover within region, 99.999% SLA |
| Lambda | Multi-AZ, automatic retry on transient failures, no server to fail |
| API Gateway | Multi-AZ, managed by AWS, no infrastructure to recover |

The most likely failure scenarios (single Lambda invocation failure, transient DynamoDB
throttling, bad deployment) are handled by retry logic and alias-based rollback — both
already in place for MVP.

### PITR provides the safety net

With PITR enabled from day one, data can be restored to any point in the last 35 days
with ~5-minute granularity. This covers the critical "accidental data deletion" and
"corruption" scenarios without needing the full DR runbook, tested procedures, or
scheduled backup cadence.

### The data is configuration, not clinical

The Endpoint Catalog holds **service configuration data** (endpoints, templates,
healthcare services) — not patient data or clinical records. If the worst happens and
data is lost:
- No patient safety impact
- Data can be reconstructed from source systems (DoS, supplier records)
- The service is temporarily unavailable, not permanently damaged

### MVP traffic volumes don't justify multi-region

Multi-region adds significant complexity and cost. It's justified for Gold/Tier 1 services
with strict availability requirements. For MVP, with limited consumers (BaRS Proxy,
R&M team), single-region with multi-AZ is sufficient.

---

## Implications of deferral

#### 1. No guaranteed RTO

**Impact: Medium**

Without a tested DR plan and defined runbook, recovery time from a major failure is
unpredictable. The team knows the mechanisms (PITR restore, alias rollback) but hasn't
practised them under pressure.

| With full DR | Without full DR (MVP) |
|---|---|
| Tested procedures, known RTO (< 30 min), named owners | Ad-hoc recovery, untested procedures, best-effort timeline |

**Mitigation:** The mechanisms exist (PITR, versioning, IaC). The team can recover — it
will just take longer the first time. The risk is acceptable for MVP traffic volumes.

---

#### 2. No multi-region failover

**Impact: Low (for MVP)**

If eu-west-2 experiences a full regional outage, the EPC is unavailable until AWS
recovers the region. There is no automatic failover to a secondary region.

| With full DR | Without full DR (MVP) |
|---|---|
| Route 53 failover to secondary region within minutes | Wait for AWS regional recovery (hours in worst case) |

**Mitigation:** Full AWS regional outages are extremely rare (< 1 per year) and typically
affect many NHS services simultaneously. The EPC would not be the only service down.
Multi-region is disproportionate for MVP.

---

#### 3. No scheduled DR testing

**Impact: Medium**

Without quarterly DR tests, confidence in the recoverability of the system degrades over
time. Schema changes, new tables, or IaC drift could mean a PITR restore doesn't work
as expected when needed.

| With full DR | Without full DR (MVP) |
|---|---|
| Quarterly restore tests validate procedures and data integrity | No scheduled testing — first real restore is during an actual incident |

**Mitigation:** Accept this risk for MVP. Schedule the first DR test within 3 months of
going live. PITR is a managed AWS feature — it doesn't drift — but the restore *procedure*
(swap table references, update Lambda config) should be validated.

---

#### 4. No pre-deployment backups

**Impact: Low**

Without automated pre-deployment on-demand backups, a bad deployment that corrupts data
can only be recovered via PITR (which requires identifying the exact point before
corruption). An on-demand backup provides a clean, named snapshot to restore from.

| With full DR | Without full DR (MVP) |
|---|---|
| On-demand backup taken before every deployment — clean restore point | PITR only — must identify correct timestamp |

**Mitigation:** PITR covers this scenario. The additional convenience of a named snapshot
is a quality-of-life improvement, not a functional gap. Can be added to the CI/CD pipeline
with minimal effort when prioritised.

---

#### 5. No formal service tier classification

**Impact: Low (for MVP)**

Without Gold/Silver classification, the availability target and DR investment level are
not formally agreed. The team operates on a best-effort basis.

**Mitigation:** The service is new and has limited consumers during MVP. Formal
classification can be completed as part of the production readiness review before wider
rollout.

---

## What the MVP still guarantees

Despite deferring the full DR plan, the MVP is not unprotected:

| Protection | How |
|---|---|
| No single point of failure | Multi-AZ across all components (DynamoDB, Lambda, API Gateway) |
| Data recoverable to last 35 days | PITR enabled from day one |
| Cannot accidentally delete tables | Deletion protection enabled |
| Bad deployment recoverable | Lambda alias rollback + API Gateway stage redeployment |
| Environment reproducible from scratch | Full IaC (Terraform) — can recreate everything from source |
| Failures detected quickly | CloudWatch alarms on error rate, latency, throttling |

---

## Decision

| Decision | Rationale |
|---|---|
| **Defer full DR plan from MVP; retain PITR + inherent resilience** | Serverless architecture provides inherent multi-AZ resilience. PITR provides continuous data backup. The EPC holds configuration data (not clinical), so the impact of temporary unavailability is limited. Full DR (multi-region, tested procedures, scheduled exercises) is delivered as part of production readiness before wider rollout. |

---

## When to deliver

The full DR plan should be delivered when:

1. **Production readiness review** — before the EPC is promoted from MVP to full
   production with broader consumers
2. **Service tier classification is agreed** — Gold classification triggers multi-region
   requirement
3. **Consumer base expands** — more consumers means higher availability expectations
4. **First quarterly DR test** — should be scheduled within 3 months of MVP go-live
   regardless of wider DR plan delivery

---

## Summary: MVP vs final DR posture

| Aspect | MVP | Final |
|---|---|---|
| Data protection | PITR (continuous, 35 days) | PITR + on-demand backups (pre-deployment + weekly) |
| Table protection | Deletion protection enabled | Same + formal change control |
| Regional resilience | Single region, multi-AZ | Multi-region active-passive (if Gold) |
| Recovery from bad deployment | Manual alias/stage rollback | Automated rollback in CI/CD with canary |
| Recovery procedures | Ad-hoc (mechanisms available, not tested) | Tested runbook with named owners and RTOs |
| DR testing | None scheduled | Quarterly (PITR restore, backup restore, alarm simulation) |
| RPO/RTO | Not formally agreed (effective RPO ~5 min via PITR) | Confirmed < 5 min RPO, < 30 min RTO |
| Log archive | CloudWatch 90 days | CloudWatch → S3 (1 year) → Glacier (7 years) |

---

## Related documents

| Document | Description |
|----------|-------------|
| [Disaster Recovery](../disaster-recovery.md) | Full DR design (target state) |
| [Resilience and Availability](../resilience-and-availability.md) | Availability requirements and patterns |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [mvp-deferral-rbac.md](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | [mvp-deferral-observability.md](./mvp-deferral-observability.md) |
| 3 | Endpoint Ordering (List) | [mvp-deferral-endpoint-ordering.md](./mvp-deferral-endpoint-ordering.md) |
| 4 | Disaster Recovery (Full DR Plan) | This document |
