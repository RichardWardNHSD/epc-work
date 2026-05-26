# Disaster Recovery

## Overview

This document describes the disaster recovery (DR) strategy for the Endpoint Catalog API.
The service is built on AWS serverless components (API Gateway, Lambda, DynamoDB) which
provide inherent resilience, but a deliberate DR plan is still required to cover data loss,
regional failure, and operational errors.

The strategy is designed around two key metrics:

| Metric | Target | Rationale |
|--------|--------|-----------|
| **RPO** (Recovery Point Objective) | < 5 minutes | Maximum acceptable data loss window |
| **RTO** (Recovery Time Objective) | < 30 minutes | Maximum acceptable downtime |

These targets are initial recommendations and should be confirmed against the service's
SLA commitments and the NHS service level classification.

---

## Failure Scenarios

| # | Scenario | Impact | Likelihood | Severity |
|---|----------|--------|-----------|----------|
| 1 | Single Lambda invocation failure | One request fails; client retries | High | Low |
| 2 | Lambda service degradation (throttling) | Elevated error rates | Medium | Medium |
| 3 | DynamoDB single-table corruption or accidental deletion | Data loss for affected table | Low | Critical |
| 4 | DynamoDB regional outage | Full service unavailable | Very low | Critical |
| 5 | API Gateway misconfiguration / deployment failure | All requests fail | Low | High |
| 6 | Accidental data deletion (operator error) | Partial or full data loss | Low | Critical |
| 7 | AWS region-wide outage | Full service unavailable | Very low | Critical |
| 8 | Deployment introduces breaking bug | Service errors on all requests | Medium | High |

---

## Recovery Mechanisms by Component

### DynamoDB

| Mechanism | Covers | RPO | RTO | Configuration |
|-----------|--------|-----|-----|---------------|
| **Point-in-Time Recovery (PITR)** | Accidental deletes, corruption, operator error | ~5 minutes (continuous backup) | 15–30 minutes (restore to new table) | Enabled on both `epc-resources` and `epc-audit` tables |
| **On-Demand Backups** | Full table snapshot before major changes | Zero (snapshot is consistent) | 15–30 minutes | Triggered before deployments and schema changes |
| **Global Tables** (optional, multi-region) | Regional outage | ~1 second (async replication) | Minutes (DNS failover) | Not enabled by default — see Multi-Region section |
| **Deletion Protection** | Accidental table deletion via API/console | N/A (preventive) | N/A | Enabled on all tables |

#### PITR Restore Procedure

1. Identify the point in time to restore to (before the corruption/deletion occurred)
2. Restore the table to a new table name: `epc-resources-restored-{timestamp}`
3. Validate the restored data
4. Update Lambda environment variables or table name configuration to point to the restored table
5. Redeploy Lambda functions
6. Verify service health via `GET /_health`
7. Once confirmed, rename or swap table references in infrastructure-as-code

#### On-Demand Backup Schedule

| Trigger | Tables | Retention |
|---------|--------|-----------|
| Pre-deployment (CI/CD pipeline step) | Both tables | 30 days |
| Weekly (EventBridge scheduled rule) | Both tables | 90 days |
| Ad-hoc (manual, before risky operations) | As needed | 30 days |

---

### Lambda

| Mechanism | Covers | Notes |
|-----------|--------|-------|
| **Versioning and Aliases** | Bad deployment | Each deployment publishes a new version. The `live` alias points to the current version. Rollback = repoint alias to previous version. |
| **Reserved Concurrency** | Throttling from noisy neighbours | Set per-function reserved concurrency to guarantee capacity |
| **Dead Letter Queue (DLQ)** | Async invocation failures | SQS DLQ captures failed async events for replay (if async patterns are introduced later) |

#### Rollback Procedure

```
1. Identify the failing Lambda version (CloudWatch Errors metric spike)
2. Retrieve the previous version number from deployment history
3. Update the alias to point to the previous version:
   aws lambda update-alias --function-name epc-resource-handler \
     --name live --function-version {previous-version}
4. Verify service health
5. Investigate and fix the broken version
6. Redeploy with fix
```

---

### API Gateway

| Mechanism | Covers | Notes |
|-----------|--------|-------|
| **Stage Deployments** | Bad configuration | Each deployment creates a new stage deployment. Rollback = redeploy previous stage deployment ID. |
| **Canary Deployments** (optional) | Gradual rollout | Route a percentage of traffic to the new deployment; promote or roll back based on error metrics |

#### Rollback Procedure

```
1. Identify the last known good deployment ID from API Gateway deployment history
2. Create a new deployment pointing to the previous stage:
   aws apigateway create-deployment --rest-api-id {api-id} --stage-name prod
   (or redeploy the previous deployment via console/IaC)
3. Verify service health
```

---

### CloudWatch (Logs and Metrics)

| Mechanism | Covers | Notes |
|-----------|--------|-------|
| **Log Group Retention** | Log availability | 90 days hot retention configured |
| **S3 Export** | Long-term log archive | Daily export to S3 with Glacier lifecycle (1 year) |
| **Cross-Region Log Replication** (optional) | Regional log loss | CloudWatch Logs subscription → Kinesis → S3 in secondary region |

Logs and metrics are not critical-path for service recovery but are essential for
post-incident investigation. The S3 archive ensures logs survive even if the primary
region's CloudWatch is unavailable.

---

## Multi-Region Strategy

The default deployment is **single-region** (eu-west-2, London). For services requiring
higher availability, a multi-region active-passive configuration can be adopted:

| Component | Multi-Region Approach |
|-----------|----------------------|
| DynamoDB | Global Tables (active-active replication to secondary region) |
| Lambda | Deploy identical functions in secondary region |
| API Gateway | Regional API in secondary region |
| DNS Failover | Route 53 health-check-based failover to secondary region |
| CloudWatch | Independent per-region (no cross-region replication needed for DR) |

**When to adopt multi-region:**
- If the service is classified as a Tier 1 / Gold service requiring 99.95%+ availability
- If a single-region RTO of 30 minutes is unacceptable
- If regulatory requirements mandate geographic redundancy

**Current recommendation:** Single-region with PITR and on-demand backups is sufficient for
the PoC and initial production. Multi-region should be evaluated as part of the production
readiness review if the service is classified as Gold.

---

## Operational Runbook Summary

| Scenario | Action | Owner | Estimated RTO |
|----------|--------|-------|---------------|
| Lambda deployment failure | Repoint alias to previous version | On-call engineer | < 5 minutes |
| API Gateway misconfiguration | Redeploy previous stage deployment | On-call engineer | < 5 minutes |
| Accidental data deletion (small scale) | PITR restore to new table, validate, swap | On-call engineer + tech lead | 15–30 minutes |
| Accidental table deletion | Restore from on-demand backup or PITR | On-call engineer + tech lead | 15–30 minutes |
| DynamoDB regional degradation | Wait for AWS resolution (or failover if multi-region) | On-call engineer | Depends on AWS |
| Full region outage | Failover to secondary region (if multi-region enabled) | On-call engineer + incident commander | 15–30 minutes |
| Data corruption discovered late | PITR restore to point before corruption, reconcile delta | Tech lead + data team | 1–4 hours |

---

## Testing the DR Plan

| Test | Frequency | Method |
|------|-----------|--------|
| Lambda rollback | Every deployment (automated in CI/CD) | Deploy, verify, rollback, verify |
| PITR restore | Quarterly | Restore to a test table, validate record counts and sample data |
| On-demand backup restore | Quarterly | Restore from latest backup, validate |
| Health check alarm | Monthly | Simulate dependency failure (e.g. block DynamoDB access via IAM), verify alarm fires |
| Full failover (multi-region) | Annually (if enabled) | Simulate region failure, verify DNS failover and data consistency |

---

## Dependencies and Assumptions

- **PITR** is enabled on both DynamoDB tables from day one. This is a non-negotiable
  baseline.
- **Deletion protection** is enabled on both tables. Disabling it requires a deliberate
  infrastructure change with approval.
- **Infrastructure-as-code** (Terraform) is the source of truth for all configuration.
  Manual console changes are prohibited in production.
- **CI/CD pipeline** includes a pre-deployment backup step and a post-deployment health
  check with automatic rollback on failure.
- **Monitoring** (Requirements 10–15) must be in place before production launch to enable
  rapid detection of failures.

---

## References

| Document | Location | Relevance |
|----------|----------|-----------|
| NHS England Engineering CoE — Red Lines Backup Scenario Overview | [SharePoint](https://nhs.sharepoint.com/sites/X26_EngineeringCOE/SitePages/Red-Lines-Backup-Scenario---Overview.aspx) | Defines the organisational red-line requirements for backup and recovery. The RPO/RTO targets and backup schedules in this document must align with the red-line scenarios defined there. |

The Red Lines Backup Scenario document establishes the minimum acceptable backup and
recovery posture for NHS England services. Key points to cross-reference:

- **Backup frequency and retention** — confirm that PITR (continuous) and weekly on-demand
  backups meet or exceed the red-line minimum backup frequency.
- **Recovery testing cadence** — confirm that quarterly DR tests satisfy the red-line
  requirement for regular restore validation.
- **Data classification** — the Endpoint Catalog holds configuration data (not patient
  data), but the red-line classification still applies and may influence retention periods.
- **Accountability** — the red-line document defines who is accountable for backup
  compliance. Ensure the EPC service owner is named and aware.

> **⚠️ Action:** Review the Red Lines Backup Scenario document and confirm that the
> RPO/RTO targets, backup retention periods, and testing cadence defined in this DR plan
> are compliant. Update this document if any adjustments are required.

---

## Open Actions

| # | Action | Owner | Status |
|---|--------|-------|--------|
| 1 | Confirm RPO/RTO targets with service owner | Tech lead | Pending |
| 2 | Confirm service tier classification (Gold/Silver) | Product owner | Pending |
| 3 | Decide on multi-region requirement | Architecture review | Pending |
| 4 | Schedule first quarterly DR test | Platform team | Pending |
| 5 | Define on-call rota and escalation path | Delivery manager | Pending |
