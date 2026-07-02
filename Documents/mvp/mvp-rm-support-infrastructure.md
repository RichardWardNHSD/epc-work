# EPC MVP — R&M Support Infrastructure

## Purpose

This document describes the support infrastructure required by the BaRS Run & Maintain
(R&M) team to perform their daily operational tasks against the Endpoint Catalog. This
infrastructure is **required for MVP** — without it, the R&M team cannot perform supplier
switches, endpoint activations, or bulk updates, and the EPC cannot replace the current
operational process.

---

## Context

The R&M team is the primary write operator for the Endpoint Catalog during MVP (and
beyond). Their daily workload includes:

| Task | Frequency | Volume |
|------|-----------|--------|
| Pharmacy supplier switches | Daily | 10–50 per day (peaks at contract changes) |
| Endpoint activations (new pharmacy onboarding) | Daily | 5–20 per day |
| Endpoint deactivations (pharmacy closure/decommission) | Weekly | 1–5 per week |
| Template updates (supplier URL change) | Occasional | 1–2 per month |
| Bulk updates (migration, data correction) | Occasional | Up to hundreds in a single batch |

These operations are currently managed via the **Master Switch Log** (a spreadsheet
tracking every change) and executed manually. The EPC MVP must provide the tooling to
execute these operations via the API.

---

## Required infrastructure

### 1. CSV-to-API processing pipeline

The R&M team's operational workflow is CSV-based. They maintain a daily switch log and
prepare CSV files containing the changes to be made. The processing pipeline translates
these CSVs into API calls.

```
R&M team prepares CSV (from Master Switch Log)
    │
    ▼
Upload CSV to designated S3 bucket
    │
    ▼
S3 event triggers processing Lambda
    │
    ▼
Lambda reads CSV rows
    │
    ▼
For each row: Lambda calls EPC API (via Apigee)
    using app-restricted OAuth token
    │
    ▼
EPC API validates token, enforces auth, executes operation
    │
    ▼
Results logged back to S3 (success/failure report)
    │
    ▼
R&M team reviews results report
```

#### Components

| Component | Purpose | MVP required? |
|---|---|---|
| **S3 input bucket** | R&M team uploads CSV files here | ✅ Yes |
| **S3 output bucket** | Processing results (success/failure per row) written here | ✅ Yes |
| **Processing Lambda** | Reads CSV, translates to API calls, handles errors | ✅ Yes |
| **App-restricted OAuth token** | Authenticates the pipeline to the EPC API via Apigee | ✅ Yes |
| **Apigee application registration** | The pipeline is a registered application on the NHS England API Platform | ✅ Yes |

---

### 2. Apigee application onboarding

The processing pipeline must be registered as an application on the NHS England API
Platform (Apigee). This is a one-time setup that grants the pipeline:

- A `client_id` for audit attribution
- An ODS code claim for ownership enforcement
- A Product ID (if applicable) for delegated writes
- `read` and `write` scopes

| Step | Detail | Effort |
|------|--------|--------|
| Register on developer portal | Create application entry | 1 day |
| Generate key pair | Public/private key for signed JWT | Included |
| Register public key | Upload JWKS to portal | Included |
| Subscribe to EPC API product | Request write access | 1 day |
| Configure ODS code | R&M team's ODS code assigned to application | Included |
| Approval | API product owner approves | 1–2 days |

**Total:** 2–3 days elapsed. One-time activity.

---

### 3. CSV format definitions

The pipeline needs defined CSV formats for each operation type. These are the input
contract between the R&M team and the processing pipeline.

| Operation | CSV columns | Reference |
|---|---|---|
| Supplier switch | `ODSCode`, `ServiceId`, `OldProductId`, `NewProductId`, `SwitchDate` | [Managing Endpoints — Supplier Switch](../manage-endpoint.md#pharmacy-endpoint-switching-supplier-switch) |
| Endpoint activation | `ODSCode`, `ProductId`, `ServiceId`, `Status`, `PeriodStart`, `PeriodEnd` | [Managing Endpoints — Create](../manage-endpoint.md#step-1--gather-the-required-data) |
| Endpoint deactivation | `ODSCode`, `ProductId`, `ServiceId`, `NewStatus` | [Managing Endpoints — Update](../manage-endpoint.md#updating-an-endpoint) |
| Template creation | `ODSCode`, `ProductId`, `Address` | [Managing Templates — Create](../manage-endpoint-template.md) |
| Template update | `ODSCode`, `ProductId`, `Address` | [Managing Templates — Update](../manage-endpoint-template.md#updating-a-template) |
| HealthcareService creation | `ODSCode`, `ServiceId`, `ServiceName`, `EndpointId` | [Managing HealthcareServices — Create](../manage-healthcare-service.md) |

---

### 4. Error handling and reporting

The pipeline must handle partial failures gracefully. A CSV with 50 rows should not fail
entirely because row 12 has a problem.

| Behaviour | Detail |
|---|---|
| **Row-level processing** | Each CSV row is processed independently. A failure on one row does not block others. |
| **Results report** | For every input CSV, a results file is written to the output S3 bucket showing success/failure per row with error detail. |
| **Retry logic** | Transient failures (5XX, timeout) are retried up to 3 times with exponential backoff. Permanent failures (4XX) are not retried. |
| **Alerting** | If more than 10% of rows in a batch fail, an alert is raised to the R&M team. |

#### Example results report

```csv
Row,ODSCode,ServiceId,Status,Error
1,FH123,2000099999,SUCCESS,
2,FH456,2000088888,SUCCESS,
3,FH789,2000077777,FAILED,"403 Forbidden: Organisation mismatch"
4,FH012,2000066666,SUCCESS,
```

---

### 5. Elevated rate limits

The R&M team's batch pipeline processes many writes in quick succession. Standard API
rate limits (designed for individual consumer lookups) may throttle bulk operations.

| Requirement | Detail |
|---|---|
| **Dedicated API product** | The R&M pipeline's Apigee application is assigned to a dedicated API product with elevated rate limits |
| **Suggested limit** | 50 requests/second (sufficient for processing 500 rows in ~10 seconds with concurrency) |
| **Throttle behaviour** | If rate-limited (429), the pipeline backs off and retries — does not fail the batch |

---

### 6. Audit trail for R&M operations

Every operation performed by the pipeline must be auditable. Because RBAC is deferred
(no individual user identity), audit records attribute changes to the pipeline's
application identity.

| Audit field | Value | Source |
|---|---|---|
| `client_id` | `bars-rm-batch-pipeline` | Bearer token |
| `ods_code` | R&M team's ODS code | Bearer token |
| `operation` | `PUT /HealthcareService/{id}` | Request |
| `resource_id` | `9f2c6f12-...` | Response |
| `timestamp` | `2026-07-01T08:15:00Z` | Runtime |
| `correlation_id` | UUID linking back to the CSV batch | Request header |
| `csv_filename` | `switches-2026-07-01.csv` | Lambda context |
| `csv_row` | `3` | Lambda context |

The `csv_filename` and `csv_row` fields allow every API change to be traced back to the
specific CSV input and row that triggered it.

---

## Why this is required for MVP

| Reason | Detail |
|---|---|
| **R&M team is the primary writer** | Without RBAC and self-service UI, the R&M team performs all write operations. They need tooling to do this efficiently. |
| **Daily operations cannot stop** | Pharmacy switches happen every day. The R&M team cannot pause operations while waiting for an admin UI or supplier self-service. |
| **Replaces the current process** | The MVP must be a functional replacement for how switches are done today. If the R&M team can't operate, the EPC delivers no value. |
| **Batch volumes require automation** | Processing 50 switches manually via individual API calls (e.g. Postman) is not viable. A pipeline is essential. |
| **Audit compliance** | Even without RBAC, every change must be traceable. The pipeline provides structured audit with CSV-level traceability. |
| **Security model requires it** | The MVP auth model (application-restricted tokens) means the pipeline must be onboarded on Apigee with correct ODS/scope. Without this, it cannot authenticate. |

---

## What happens without it

If the R&M support infrastructure is not delivered with MVP:

- The R&M team cannot perform daily supplier switches → pharmacies remain on old
  suppliers indefinitely → live patient pathways are affected
- The R&M team resorts to manual API calls (Postman/curl) → error-prone, unscalable,
  no batch processing, poor audit trail
- The EPC exists but nobody can write to it → a database with no operational process
- The current manual process cannot be retired → no operational benefit from the EPC

---

## Infrastructure summary

| Component | Type | Effort | Notes |
|---|---|---|---|
| S3 input bucket | AWS S3 | Terraform | Part of EPC infrastructure |
| S3 output bucket | AWS S3 | Terraform | Part of EPC infrastructure |
| Processing Lambda | AWS Lambda (TypeScript/Python) | Development | Core pipeline logic |
| Apigee application registration | NHS England API Platform | One-time setup | 2–3 days |
| CSV format definitions | Documentation | Documented in manage-* pages | Already done |
| Error handling + results report | Lambda logic | Development | Part of pipeline |
| Elevated rate limit (API product) | Apigee configuration | One-time setup | Request from platform team |
| CloudWatch alarm (batch failure rate) | CloudWatch | Terraform | Standard alarm |



This covers the Lambda development, S3 configuration, Apigee onboarding, error handling,
results reporting, and testing. The CSV formats are already defined in the operational
documents.

---

## Related documents

| Document | Description |
|----------|-------------|
| [Interim Support Process Access](../interim-support-process-access.md) | Options analysis for R&M team access (direct AWS vs Apigee) |
| [Managing Endpoints](../manage-endpoint.md) | Endpoint operations including supplier switches |
| [Managing HealthcareServices](../manage-healthcare-service.md) | HealthcareService operations |
| [Managing Endpoint Templates](../manage-endpoint-template.md) | Template operations |
| [EPC MVP Deferral — RBAC](./mvp-deferral-rbac.md) | Why there is no admin UI or self-service — R&M team is the operational backstop |
