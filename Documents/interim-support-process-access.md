# Interim Support Process — Direct AWS Access vs API Access

## Context


The architecture diagram shows an "EPC Updation Interim Tactical Solution" where the BaRS
Run & Maintain (R&M) team accesses the EPC via a CSV-based process:

1. R&M team uploads a CSV file to S3
2. A Lambda function processes the CSV
3. Records are queued via SQS
4. Another Lambda calls directly to EPC API Gateway 
5. The EPC Backend API Lambda handles the request and updates DynamoDB

This bypasses the **Apigee-hosted API** — the interim process calls the EPC's AWS API
Gateway directly (using IAM authentication) rather than going through the published
Apigee API endpoint. This means the request does not pass through Apigee's token
validation, ODS spoofing protection, or rate limiting — though it does still hit the EPC
backend Lambda which performs business logic, validation, and audit.

This document evaluates the pros and cons of this approach, particularly given that the
R&M team will be making **numerous write requests** as part of their daily operations
(supplier switches, endpoint activations, bulk updates).

---

## The Audit Problem

When requests go through Apigee, the bearer token carries identity claims (application ID,
ODS code, Product ID, scopes) that are:
- Validated by Apigee (signature, expiry, audience)
- Extracted and forwarded to the backend as trusted headers
- Used by the backend to create an audit record (which application, from which organisation)

The R&M team will use **application-restricted tokens** (signed JWT) via Apigee — there
will be no CIS2 user-restricted access for this process. This means the audit trail
captures the application identity and ODS code, but not an individual user.

When the interim process calls the AWS API Gateway directly (bypassing Apigee), **there is
no bearer token at all**. The request is authenticated via AWS IAM (the Lambda's execution
role has permission to invoke the API Gateway), but IAM authentication provides only
infrastructure-level identity, not application-level identity:

| What IAM provides | What's missing (no Apigee token) |
|---|---|
| Which AWS role made the call | Which registered application initiated the batch |
| That the caller has permission to invoke the API | Which organisation the change is on behalf of (ODS code) |
| CloudTrail log of the API invocation | The Product ID for supplier ownership check |
| | Scopes (read/write permissions) |

### Impact on audit quality

| Audit field | Via Apigee (app-restricted token) | Via AWS API GW direct (no token) |
|---|---|---|
| **Who made the change** | Application client_id (e.g. "bars-rm-batch-tool") | "csv-processing-lambda" (IAM role ARN) |
| **Which organisation** | ODS code from token claims | Unknown — must be inferred from CSV content |
| **Product ID** | From token claims — enables ownership check | None |
| **Authorisation verified** | Yes — Apigee validated token + backend checked ODS/Product ID ownership | Partial — IAM permits the call, but no ODS/Product ID ownership check against token |
| **Tamper-proof identity** | Yes — token is cryptographically signed | No — the Lambda can pass any identity in the request headers |
| **Apigee gateway audit** | Yes — request logged in Splunk with full metadata | No — invisible to Apigee logging |

### The risk

Without a token, the audit record for a write operation can only say:

> "The csv-processing Lambda modified Endpoint/ep-001 at 16:05 UTC"

With an app-restricted token via Apigee, it can say:

> "Application bars-rm-batch-tool (client_id: abc-123) from organisation R778
> modified Endpoint/ep-001 at 16:05 UTC"

This is still less granular than CIS2 (no individual user identity), but it provides:
- **Application accountability** — which registered tool made the change
- **Organisation attribution** — which ODS code, verified against the token
- **Ownership enforcement** — Product ID and ODS checks still apply
- **Non-repudiation** — the token is cryptographically signed; the identity can't be forged

Without any token (direct AWS path), none of these are available.

---

## Apigee Onboarding Requirement

For Options 2 and 3 (which route through Apigee), the R&M team's batch tool must be
**registered as an application on the NHS England API Platform**. This is because the EPC
API is published on Apigee as a public-facing API — all consumers, including internal NHS
England teams, must onboard through the standard process.

### What onboarding involves

| Step | Detail |
|------|--------|
| 1. Register on the developer portal | Create an application entry on the NHS England API Platform developer portal |
| 2. Generate key pair | The application is issued a client ID and generates a public/private key pair for signed JWT authentication |
| 3. Register public key | Upload the public key (JWKS) to the developer portal so Apigee can validate signed JWTs |
| 4. Subscribe to the EPC API product | Request access to the Endpoint Catalog API product (which grants the appropriate scopes) |
| 5. Configure ODS code | The application's registered ODS code is set (e.g. the NHS England internal ODS code for the R&M team) |
| 6. Approval | The API product owner approves the subscription |

### Why this is required

- The EPC API is a **public-facing API** hosted on the NHS England API Platform — it is
  not an internal-only service
- All consumers (external suppliers, DoS, GP Connect, AND internal NHS England teams)
  use the same API through the same platform
- This ensures consistent security, rate limiting, audit logging, and monitoring
  regardless of whether the caller is internal or external
- The R&M team's batch tool is treated as any other registered application — it gets a
  client ID, scopes, and an ODS code claim in its token

### Effort

Onboarding is a one-time activity. Once registered, the application can obtain tokens
indefinitely without re-onboarding. The process typically takes 1–2 days for internal
NHS England applications (no assurance review required for internal tools).

---

## Option 1: Direct AWS API Gateway Access (Current Interim Design)

The R&M team uploads CSV files to S3. Internal Lambdas process the files and call the
EPC's AWS API Gateway directly (bypassing Apigee), which invokes the EPC backend Lambda.
The backend Lambda performs validation and writes to DynamoDB.

### Pros

| # | Advantage | Detail |
|---|-----------|--------|
| 1 | **Fast to implement** | No Apigee integration needed — direct AWS-to-AWS call |
| 2 | **Handles bulk operations well** | CSV can contain hundreds of changes processed in one batch |
| 3 | **No Apigee rate limiting** | Direct AWS API Gateway calls aren't subject to Apigee throttling policies |
| 4 | **No OAuth token management** | Uses IAM authentication (AWS-native) rather than OAuth tokens |
| 5 | **Familiar workflow** | R&M team already works with spreadsheets (Master Switch Log) |
| 6 | **Backend validation still applies** | The EPC backend Lambda still performs business logic, FHIR validation, and DynamoDB writes |
| 7 | **No Apigee downtime dependency** | If Apigee is down, the batch process still works |

### Cons

| # | Risk | Detail | Severity |
|---|------|--------|----------|
| 1 | **Bypasses Apigee token validation** | No OAuth bearer token is validated — the request uses IAM auth instead. The identity is an AWS role, not an NHS application or user. | **High** |
| 2 | **Bypasses ODS spoofing protection** | The Apigee-layer cross-check (token ODS claim vs header) doesn't apply — there's no OAuth token with ODS claims. | **High** |
| 3 | **No user-level audit trail** | The audit record can only attribute changes to the batch Lambda's IAM role — not to a specific R&M team member or their CIS2 identity. | **High** |
| 4 | **Bypasses Apigee rate limiting and monitoring** | Apigee metrics and dashboards don't capture these writes — they're invisible to the API-level observability. | **Medium** |
| 5 | **ODS ownership check may be weakened** | If the batch Lambda doesn't pass `NHSD-End-User-Organisation-ODS` correctly, the backend's ownership check may not function as intended. | **Medium** |
| 6 | **Two access paths to maintain** | The Apigee path and the direct AWS path must both be kept consistent. Changes to API Gateway configuration affect both. | **Medium** |
| 7 | **IAM security surface** | The batch Lambda's IAM role has direct invoke permissions on the API Gateway. A misconfigured role could allow broader access than intended. | **Medium** |
| 8 | **Difficult to retire** | Once operational processes depend on the direct AWS path, migrating to the Apigee path requires changing team workflows and IAM configuration. | **Low** |

---

## Option 2: R&M Team Uses the Published API (via Apigee)

The R&M team uses the same EPC API that all other consumers use — either via a simple
admin UI, a CLI tool, or a script that calls the API endpoints via the Apigee-hosted
public API.

### Apigee onboarding requirement

Because the EPC API is published on the NHS England API Platform (Apigee) as a
**public-facing API**, any application that calls it — including internal NHS England
tools — must be formally onboarded:

1. **Register the application** on the [NHS England Developer Portal](https://portal.developer.nhs.uk/create-a-developer-account)
2. **Generate a key pair** — the application's public key is registered with Apigee for
   JWT signature validation
3. **Assign API products** — the application is granted access to the EPC API product
   with appropriate scopes (read, write)
4. **Assign an ODS code** — the application's token will carry the R&M team's ODS code
   for ownership checks
5. **Assign a Product ID** (if applicable) — for supplier-level ownership checks

This is a one-time setup process. Once onboarded, the application can obtain tokens
and call the API indefinitely.

**Why this matters:** Even though the R&M team is internal to NHS England, the API
treats all callers equally. There is no "internal bypass" — the same authentication,
authorisation, and audit controls apply to internal tools as to external supplier
applications. This is by design: it ensures the audit trail is consistent and the
security model has no exceptions.

### Pros

| # | Advantage | Detail |
|---|-----------|--------|
| 1 | **Full authorisation enforcement** | Every write goes through ODS ownership check, Product ID validation |
| 2 | **Complete audit trail** | Every change is attributed to the registered application (client_id, ODS code) |
| 3 | **Single code path** | All business logic (validation, duplicate detection, status rules, List sync) is in one place |
| 4 | **Input validation guaranteed** | API rejects malformed data before it reaches DynamoDB |
| 5 | **Observability** | All writes appear in Apigee metrics, Splunk logs, and EPC audit records |
| 6 | **Consistent behaviour** | The R&M team sees the same errors and responses as any other consumer — no surprises |
| 7 | **No additional infrastructure** | No S3 bucket, no CSV-processing Lambda, no SQS queue to maintain |
| 8 | **Easier to retire** | When suppliers self-serve, the R&M team simply stops calling the API — no infrastructure to decommission |
| 9 | **Apigee gateway audit** | Every request is logged in Splunk at the gateway level — provides a second audit layer |

### Cons

| # | Risk | Detail | Severity |
|---|------|--------|----------|
| 1 | **Bulk operations are harder** | The API is designed for individual CRUD operations, not batch processing. 500 endpoint updates = 500 API calls. | **Medium** |
| 2 | **Rate limiting** | High-volume write bursts may hit API Gateway throttling limits | **Medium** |
| 3 | **Token management** | The R&M tool needs to obtain and refresh OAuth tokens (automated by the tool) | **Low** |
| 4 | **Latency per operation** | Each API call has ~100ms overhead vs direct invocation | **Low** |
| 5 | **Requires tooling** | The R&M team needs a CLI tool, script, or admin UI to make API calls (they can't just upload a spreadsheet) | **Medium** |
| 6 | **Apigee onboarding** | The R&M team's application must be registered on the NHS England Developer Portal (one-time setup) | **Low** |

### Mitigations for the cons

| Con | Mitigation |
|-----|-----------|
| Bulk operations | Implement a `POST /Endpoint/$bulk-update` operation that accepts an array of changes in one request. Or provide a CLI tool that reads a CSV and makes API calls in parallel. |
| Rate limiting | Set a higher rate limit for the R&M team's registered application. Or use a dedicated API product with elevated quotas. |
| Token management | The CLI tool handles token refresh automatically — transparent to the user. |
| Requires tooling | Build a simple CLI (`epc-admin update --csv switches.csv`) that wraps the API calls. Low effort, high value. |

---

## Option 3: Hybrid — CSV Upload Triggers Apigee API Calls (Recommended)

The R&M team uploads a CSV to S3 (familiar workflow), but instead of calling the AWS API
Gateway directly, the processing Lambda **calls the EPC via the Apigee-hosted API** —
the same path that all other consumers use.

### How it works

```
R&M team → uploads CSV to S3
         → S3 event triggers Lambda
         → Lambda reads CSV rows
         → For each row: Lambda calls Apigee-hosted EPC API (POST/PUT/DELETE)
           (using app-restricted OAuth token for the R&M service account)
         → Apigee validates token, extracts claims, forwards to EPC backend
         → EPC backend enforces auth, validation, ownership, audit
         → Results logged back to S3 (success/failure report)
```

### Pros

| # | Advantage |
|---|-----------|
| 1 | Familiar workflow for R&M team (CSV upload) |
| 2 | Full API authorisation and validation on every write |
| 3 | Complete audit trail (Lambda uses a service account with R&M team identity) |
| 4 | Single code path for business logic |
| 5 | Bulk-friendly (CSV can contain hundreds of rows) |
| 6 | Error handling — failed rows are reported back, not silently dropped |
| 7 | Easy to retire — when R&M team moves to self-service UI, just stop using the CSV path |

### Cons

| # | Risk | Mitigation |
|---|------|-----------|
| 1 | Slower than direct DynamoDB writes | Acceptable — batch processing is not time-critical (runs daily after 4pm) |
| 2 | Rate limiting on API | Use elevated quotas for the batch service account |
| 3 | Still requires S3 bucket + Lambda | Less infrastructure than Option 1 (no SQS needed) |

---

## Comparison

| Aspect | Option 1: Direct AWS API GW | Option 2: Apigee API Only | Option 3: CSV → Apigee API (Recommended) |
|--------|----------------------------|--------------------------|------------------------------------------|
| Apigee token validation | ❌ Bypassed (IAM auth) | ✅ Yes | ✅ Yes |
| ODS spoofing protection | ❌ Bypassed | ✅ Yes | ✅ Yes |
| Backend validation | ✅ Yes (Lambda still runs) | ✅ Yes | ✅ Yes |
| Audit trail (who) | ⚠️ IAM role only | ✅ Individual user/app | ✅ Service account + CSV metadata |
| Bulk-friendly | ✅ CSV native | ⚠️ Needs tooling | ✅ CSV native |
| Single access path | ❌ Two paths (Apigee + direct) | ✅ One path | ✅ One path (via Apigee) |
| R&M team workflow change | None | Significant | Minimal (still upload CSV) |
| Apigee observability | ❌ Invisible to Apigee | ✅ Full visibility | ✅ Full visibility |
| Infrastructure to maintain | S3 + Lambda + SQS | None | S3 + Lambda (no SQS) |
| Retirement difficulty | Hard | Easy | Easy |

---

## Recommendation

**Option 3 (CSV → API)** is recommended because it:
- Preserves the R&M team's existing CSV-based workflow
- Ensures all writes go through the API's authorisation, validation, and audit mechanisms
- Maintains a single code path for business logic
- Is easy to retire when suppliers move to self-service

The key principle: **no write path should bypass the API's authorisation and audit controls,
regardless of who is making the write or how it's initiated.**

---

## Open Questions

| # | Question | For |
|---|----------|-----|
| 1 | What identity does the batch Lambda use when calling the API? A dedicated service account with R&M team ODS code? | Architecture / Security |
| 2 | Should the CSV include a "requested by" field so the audit trail captures which R&M team member initiated the batch? | Product |
| 3 | What rate limit is appropriate for the batch service account? | Platform team |
| 4 | Should failed rows be retried automatically or reported for manual review? | R&M team |
| 5 | Is the interim process expected to run for months or years? This affects how much investment is justified. | Product |
