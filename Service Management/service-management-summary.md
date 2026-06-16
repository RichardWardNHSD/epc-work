# EPC Service Management Summary

## Overview

This document summarises the service management requirements and operational readiness
position for the Endpoint Catalogue (EPC) API, based on the technical design documents
and the draft service management requirements (V0.2).

> **Note:** This summary is derived from the technical architecture documents. The
> authoritative service management requirements are in the accompanying `.docx` files
> in this folder. This markdown summary provides a cross-reference between those
> requirements and the technical design decisions already made.

---

## Service Classification

| Aspect | Current Position | Open Action |
|---|---|---|
| **Service tier** | Targeting Silver/Tier 2 (99.9% availability) | Confirm with service owner — if Gold/Tier 1, multi-region required |
| **Availability target** | 99.9% (≈8.7 hours downtime/year) | Needs formal sign-off |
| **Planned maintenance windows** | Zero (serverless architecture) | Confirmed by design |
| **Response time (P95)** | ≤ 150ms | Provisioned concurrency recommended to meet this |
| **RPO** | < 5 minutes | Achieved via DynamoDB PITR |
| **RTO** | < 30 minutes | Achieved via PITR restore + redeployment |

---

## Operational Support Model

### Current State (Interim)

| Aspect | Detail |
|---|---|
| **Support team** | BaRS Run & Maintain (R&M) team |
| **Support mechanism** | CSV → S3 → Lambda → API pipeline for bulk operations |
| **Access model** | Application-restricted via Apigee (recommended); direct AWS API Gateway access (interim tactical) |
| **Hours** | Business hours (on-call TBD) |
| **Escalation** | National Service Desk → BaRS Programme (if required) |
| **Self-service** | Not available — all changes via R&M team |

### Target State

| Aspect | Detail |
|---|---|
| **Support team** | Dedicated EPC support + supplier self-service |
| **Access model** | App-restricted for automation, CIS2 for admin UI |
| **Self-service** | Supplier self-service via admin UI (CIS2 authenticated) |
| **Automation** | Supplier systems manage own endpoints via API |

---

## Disaster Recovery

| Mechanism | Purpose | RPO | RTO |
|---|---|---|---|
| **DynamoDB PITR** | Continuous backup; restore to any point in last 35 days | ~5 minutes | 15–30 minutes |
| **On-Demand Backups** | Snapshot before deployments/schema changes | Zero (consistent snapshot) | 15–30 minutes |
| **Deletion Protection** | Prevents accidental table deletion | N/A (preventive) | N/A |
| **Infrastructure-as-Code (Terraform)** | Full environment reproducibility | N/A | Minutes (redeploy) |
| **Multi-region (Global Tables)** | Regional failover (if Gold classification) | ~1 second | Minutes (DNS) |

### DR Open Actions

- Confirm RPO/RTO targets with service owner
- Confirm service tier classification (Gold/Silver)
- Decide on multi-region requirement
- Schedule first quarterly DR test
- Define on-call rota and escalation path

---

## Observability and Monitoring

| Layer | Tool | Purpose |
|---|---|---|
| **Logs** | CloudWatch Logs (structured JSON) | Operational troubleshooting, audit |
| **Metrics** | CloudWatch Metrics + custom (EMF) | Latency, errors, throughput |
| **Alerting** | CloudWatch Alarms → SNS | Team notifications, ITOC escalation |
| **Tracing** | AWS X-Ray | Request tracing through API GW → Lambda → DynamoDB |
| **Centralised** | ODIN (future) | Cross-service visibility |
| **ITOC forwarding** | CloudWatch → (mechanism TBD) | Operational visibility for ITOC |
| **CSOC** | Log export (format TBD) | Security monitoring |

### Log Retention

| Log type | Retention |
|---|---|
| Application logs | 90 days |
| API Gateway access logs | 90 days |
| Audit event logs | 365 days |

### Key Alarms

Expected alarm coverage (to be confirmed):
- API error rate > threshold (5xx responses)
- API latency P95 > 150ms
- DynamoDB throttling events
- Lambda concurrent execution > 80% of limit
- Failed authentication attempts (potential abuse)

---

## Audit

| Aspect | Detail |
|---|---|
| **Audit store** | Dedicated `epc-audit` DynamoDB table |
| **Retention** | 365 days minimum |
| **What is captured** | Who made the change (app/user ID), what changed (resource + field-level diff), when, on whose behalf (ODS code) |
| **Identity source (app-restricted)** | Application client_id, Product ID, ODS code (from token claims forwarded by Apigee) |
| **Identity source (CIS2)** | Individual user UUID, role, organisation (from CIS2 token claims) |

### Audit Gap (Interim Process)

The interim tactical support process (direct AWS access, bypassing Apigee) has reduced
audit quality — only IAM role identity is captured, not application/user identity.
The recommended path (CSV → Apigee API calls) preserves full audit trail.

---

## Incident Management

| Severity | Example | Response target | Escalation |
|---|---|---|---|
| **P1 — Critical** | Full service unavailable, data loss | 15 minutes (acknowledge), 1 hour (resolve) | Immediate — on-call + service owner |
| **P2 — High** | Degraded performance, partial functionality loss | 30 minutes (acknowledge), 4 hours (resolve) | Development team + service owner |
| **P3 — Medium** | Single consumer affected, workaround available | Next business day | Development team |
| **P4 — Low** | Minor issue, cosmetic, enhancement request | Planned sprint | Backlog |

### Incident Routing

| Source | Route |
|---|---|
| External consumer (supplier) | National Service Desk → BaRS Programme → EPC team |
| Internal consumer (BaRS Proxy) | Automated alerting (CloudWatch Alarms) → EPC on-call |
| Platform issue (Apigee/AWS) | ITOC → EPC team (if EPC-specific) |

---

## Change Management

| Change type | Approval | Deployment |
|---|---|---|
| **Standard change** (config, non-breaking) | Team lead approval | Automated CI/CD → INT → PROD |
| **Normal change** (new feature, API change) | Architecture review + product owner | CI/CD with manual promotion gate |
| **Emergency change** | Verbal approval, retrospective documentation | Direct deployment with immediate rollback plan |

### Deployment Strategy

- **Infrastructure**: Terraform (immutable, versioned)
- **Code**: CI/CD pipeline (automated tests → INT → manual gate → PROD)
- **Rollback**: Previous Lambda version alias swap (< 1 minute)
- **Zero downtime**: Lambda aliases + API Gateway stage deployment

---

## Data Management

| Aspect | Detail |
|---|---|
| **Data store** | DynamoDB (on-demand, multi-AZ) |
| **Backup** | PITR (continuous) + on-demand before changes |
| **Data migration** | CSV → S3 → Lambda pipeline (run by R&M team) |
| **Data ownership** | Templates/Endpoints owned by suppliers; HealthcareServices owned by providers |
| **Deletion** | Soft delete (status → `entered-in-error`) preferred; hard delete requires System Admin role |

---

## Dependencies

| Dependency | Impact if unavailable | Mitigation |
|---|---|---|
| **Apigee** | No external requests reach EPC | None — Apigee is the gateway. Platform SLA applies. |
| **DynamoDB** | No data reads or writes | Multi-AZ (built-in); multi-region (optional) |
| **CloudWatch** | No monitoring/alerting | AWS SLA; manual health checks as fallback |
| **BaRS Proxy** (consumer) | BaRS cannot resolve endpoints | EPC outage becomes a BaRS outage for message routing |
| **CIS2** (for admin operations) | Admin users cannot authenticate | App-restricted path remains available for automated operations |

---

## Runbook Coverage

The following operational procedures should be documented in the runbook:

| # | Procedure | Status |
|---|---|---|
| 1 | PITR restore from DynamoDB backup | Documented in DR design |
| 2 | Lambda rollback to previous version | To document |
| 3 | Emergency Terraform redeploy | To document |
| 4 | Bulk endpoint update via CSV pipeline | Documented in interim support process |
| 5 | Supplier switch (change HS → Endpoint association) | Documented in visibility/status design |
| 6 | Endpoint decommissioning (status → off) | Documented in visibility design |
| 7 | On-call escalation procedure | To define |
| 8 | ITOC handover (what to check, what to escalate) | To define |
| 9 | Consumer onboarding (adding a new API consumer) | Documented in onboarding guides |
| 10 | Certificate renewal (EPC server cert for mTLS) | To document |

---

## Open Actions (Service Management)

| # | Action | Owner | Status |
|---|---|---|---|
| 1 | Confirm service tier (Gold/Silver) and availability target | Service owner | Open |
| 2 | Define on-call rota and escalation path | Delivery lead | Open |
| 3 | Schedule first quarterly DR test | Delivery lead | Open |
| 4 | Confirm ITOC log forwarding mechanism | Platform team | Open |
| 5 | Confirm ODIN onboarding timeline | ODIN team | Open |
| 6 | Define log retention beyond 90 days (archive?) | Architecture | Open |
| 7 | Confirm CSOC log forwarding requirements | Security | Open |
| 8 | Complete runbook procedures (items 2, 3, 7, 8, 10 above) | Development team | Open |
| 9 | Agree monitoring dashboards with ITOC | Development team / ITOC | Open |
| 10 | Confirm P1/P2 response time SLAs with service owner | Service owner | Open |
