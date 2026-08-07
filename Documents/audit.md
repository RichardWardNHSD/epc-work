# Audit and Logging

---

## Table of Contents

---

## 1 Introduction

The Endpoint Catalog Audit Trail feature provides a complete, immutable history of all changes made to resources managed by the Endpoint Catalog API. The audit trail covers all four resource types — HealthcareService, Template, Endpoint, and List — and records who made each change, from which organisation, when, and what kind of change it was. A dedicated query API allows Endpoint Administrators and support staff to search and retrieve audit records using a range of filters.

The feature satisfies the original user story EPCSe006:

> As an Endpoint Administrator, I want a full audit history of any changes applied to an endpoint so that I have a clear view of when endpoints are added, amended, changed status, etc. for audit and support purposes.

The scope has been extended to cover all resource types in the Endpoint Catalog, not just Endpoints.

### 1.1 Audit layers

The Endpoint Catalog has two distinct audit layers that together provide a complete audit picture:


| Layer             | Mechanism                    | Scope                                                                                 | Owner                          |
| ------------------- | ------------------------------ | --------------------------------------------------------------------------------------- | -------------------------------- |
| API gateway audit | Apigee → Splunk             | All inbound HTTP requests at the internet-facing API boundary, regardless of outcome  | Platform / infrastructure team |
| Application audit | Audit_Service (this feature) | Successful write operations on catalogued FHIR resources, with business-level context | Endpoint Catalog application   |

The Apigee/Splunk layer captures low-level request telemetry (e.g. caller IP, request path, HTTP method, response code, latency). The exact data items to be captured at this layer are **to be agreed by the team** — they are out of scope for this requirements document but must be defined before the service goes into production.

The application audit layer (this feature) captures business-level change events with resource context and actor identity derived from the bearer token. These two layers are complementary: the gateway layer provides a complete request log; the application layer provides a queryable, resource-centric change history.

## 2 Glossary

- **Audit_Service**: The component of the Endpoint Catalog API responsible for recording and retrieving audit records.
- **Audit_Record**: An immutable record capturing a single change event on a catalogued resource.
- **Actor**: The authenticated user who performed the change, identified by claims in the bearer token.
- **Actor_Organisation**: The ODS code of the organisation associated with the Actor, as asserted in the bearer token claims.
- **Resource_Type**: One of the four catalogued FHIR resource types: `HealthcareService`, `Template`, `Endpoint`, or `List`.
- **Change_Type**: A coded value describing the nature of the change. Valid values: `created`, `updated`, `deleted`.
- **Bearer_Token**: The OAuth 2.0 access token supplied by the caller in the `Authorization` header, issued by NHS CIS2.
- **ODS_Code**: An NHS Organisation Data Service code uniquely identifying an NHS organisation.
- **Product_Identifier**: The identifier of a product or programme associated with an Endpoint or Template (e.g. a BaRS product code).
- **Endpoint_Identifier**: The logical identifier of an Endpoint resource within the Endpoint Catalog.
- **Healthcare_Service_Identifier**: The logical identifier of a HealthcareService resource within the Endpoint Catalog.
- **Audit_Query_API**: The read-only API endpoint that allows callers to search and retrieve Audit_Records.
- **UTC**: Coordinated Universal Time. All timestamps in the system are stored and returned in UTC.
- **Apigee_Audit**: The audit mechanism provided by the Apigee API gateway for all inbound requests at the internet-facing API boundary.
- **Splunk**: The current log aggregation and search platform used to store and query Apigee_Audit data.

## 3 Requirements

### 3.1 Requirement 1: Audit Record Creation on Write Operations

**User Story:** As an Endpoint Administrator, I want every write operation on any catalogued resource to be automatically recorded, so that no change can occur without leaving a traceable audit entry.

#### 3.1.1 Acceptance Criteria

1. WHEN a `POST`, `PUT`, `PATCH`, or `DELETE` request on a `HealthcareService`, `Template`, `Endpoint`, or `List` resource succeeds, THE Audit_Service SHALL create an Audit_Record for that operation.
2. WHEN an Audit_Record is created, THE Audit_Service SHALL record the following fields:

   - `actorId`: the user identifier extracted from the Bearer_Token claims
   - `actorOrganisation`: the ODS_Code extracted from the Bearer_Token organisation claim
   - `timestamp`: the date and time of the change in UTC, to millisecond precision
   - `resourceType`: the Resource_Type of the changed resource
   - `resourceId`: the FHIR logical id of the changed resource
   - `changeType`: the Change_Type derived from the HTTP method and operation context
3. WHEN a write operation fails (the API returns a 4xx or 5xx response), THE Audit_Service SHALL NOT create an Audit_Record for that operation.
4. THE Audit_Service SHALL derive the Change_Type from the operation as follows:

   - `POST` resulting in a new resource → `created`
   - `PUT` or `PATCH` on an existing resource → `updated`
   - `DELETE` on an existing resource → `deleted`
5. THE Audit_Service SHALL store Audit_Records in a durable, append-only store such that existing records cannot be modified or deleted through any API operation.

### 3.2 Requirement 2: Actor Identity from Bearer Token

**User Story:** As an Endpoint Administrator, I want the audit record to capture who made each change using their authenticated identity, so that I can hold individuals and organisations accountable.

> **Note:** This requirement cannot be fully met until RBAC (Role-Based Access Control) is in place. Until then, audit will record identity at the **organisation level** only (ODS code from the bearer token). Individual user-level attribution will be added once RBAC is implemented and individual user claims are available in the token.

#### 3.2.1 Acceptance Criteria

1. WHEN creating an Audit_Record, THE Audit_Service SHALL extract the Actor identity exclusively from the Bearer_Token claims and SHALL NOT use the `NHSD-End-User-Organisation-ODS` request header as the source of the Actor_Organisation.
2. WHEN creating an Audit_Record, THE Audit_Service SHALL record the Actor_Organisation as the ODS_Code asserted in the Bearer_Token's organisation claim (the same claim used for ODS spoofing prevention as defined in the authorisation model).
3. IF the Bearer_Token does not contain a resolvable user identifier claim, THEN THE Audit_Service SHALL reject the write operation with a `401 Unauthorized` response and SHALL NOT create an Audit_Record.
4. THE Audit_Service SHALL store the `actorId` and `actorOrganisation` as separate, independently queryable fields on every Audit_Record.

### 3.3 Requirement 3: Audit Coverage for All Resource Types

**User Story:** As an Endpoint Administrator, I want the audit trail to cover all resource types in the Endpoint Catalog, so that I have a complete picture of all changes regardless of which resource was affected.

#### 3.3.1 Acceptance Criteria

1. THE Audit_Service SHALL record Audit_Records for write operations on all four Resource_Types: `HealthcareService`, `Template`, `Endpoint`, and `List`.
2. IF a `POST /HealthcareService` operation automatically creates an associated `List` resource, THE Audit_Service SHALL create two Audit_Records: one for the `HealthcareService` creation and one for the `List` creation, each with `changeType: created`.
3. IF a `PUT` or `PATCH` on a `HealthcareService` automatically synchronises the associated `List`, THE Audit_Service SHALL create an Audit_Record for the `List` update with `changeType: updated`, in addition to the Audit_Record for the `HealthcareService` change.
4. THE Audit_Service SHALL record the `resourceType` field on every Audit_Record using the exact FHIR resource type name: `HealthcareService`, `Endpoint`, or `List`. Template resources SHALL be recorded with `resourceType: Template`.

### 3.4 Requirement 4: Audit Query API — Search and Retrieval

**User Story:** As an Endpoint Administrator, I want to search the audit trail using a range of filters, so that I can quickly locate the history of a specific resource, organisation, or time period.

#### 3.4.1 Acceptance Criteria

1. THE Audit_Query_API SHALL expose a `GET /AuditEvent` endpoint that returns a FHIR `Bundle` of type `searchset` containing matching Audit_Records.
2. THE Audit_Query_API SHALL support the following search parameters, each independently optional:


| Parameter             | Filters on                                                                |
| ----------------------- | --------------------------------------------------------------------------- |
| date                  | Exact calendar date of the change (UTC)                                   |
| date-from             | Start of a date range (inclusive, UTC)                                    |
| date-to               | End of a date range (inclusive, UTC)                                      |
| organisation          | actorOrganisation (ODS_Code)                                              |
| product-id            | Product_Identifier associated with the changed resource                   |
| endpoint-id           | Endpoint_Identifier of the changed Endpoint                               |
| healthcare-service-id | Healthcare_Service_Identifier of the changed or related HealthcareService |
| url                   | The address field of the Endpoint or Template at the time of the change   |
| connection-type       | The connectionType code of the Endpoint or Template                       |
| payload-type          | The payloadType code of the Endpoint or Template                          |

3. WHEN multiple search parameters are supplied in a single request, THE Audit_Query_API SHALL apply them as a logical AND, returning only Audit_Records that satisfy all supplied parameters.
4. WHEN `date-from` and `date-to` are both supplied, THE Audit_Query_API SHALL return Audit_Records whose `timestamp` falls within the inclusive range `[date-from 00:00:00 UTC, date-to 23:59:59 UTC]`.
5. WHEN `date` is supplied without `date-from` or `date-to`, THE Audit_Query_API SHALL return Audit_Records whose `timestamp` falls within the calendar day `[date 00:00:00 UTC, date 23:59:59 UTC]`.
6. WHEN no search parameters are supplied, THE Audit_Query_API SHALL return the most recent 100 Audit_Records ordered by `timestamp` descending.
7. THE Audit_Query_API SHALL return results ordered by `timestamp` descending (most recent first) in all cases.
8. THE Audit_Query_API SHALL support pagination using `_count` (page size) and `_offset` (zero-based record offset) parameters, and SHALL include `next` and `previous` pagination links in the Bundle `link` array where applicable.

### 3.5 Requirement 5: Audit Record Content

**User Story:** As an Endpoint Administrator, I want each audit record to contain enough detail to understand exactly what changed and in what context, so that I can reconstruct the history of a resource without needing to cross-reference other systems.

#### 3.5.1 Acceptance Criteria

1. THE Audit_Service SHALL include the following fields on every Audit_Record:


| Field             | Type                       | Description                                                                                |
| ------------------- | ---------------------------- | -------------------------------------------------------------------------------------------- |
| id                | UUID                       | Unique identifier for the Audit_Record                                                     |
| timestamp         | ISO 8601 datetime (UTC)    | Date and time of the change                                                                |
| actorId           | string                     | User identifier from the Bearer_Token (organisation-level until RBAC is implemented)       |
| actorOrganisation | string                     | ODS_Code from the Bearer_Token organisation claim                                          |
| resourceType      | string                     | Resource_Type of the changed resource                                                      |
| resourceId        | string                     | FHIR logical id of the changed resource                                                    |
| changeType        | string                     | Change_Type code                                                                           |
| beforeSnapshot    | object (FHIR/JSON) or null | Full FHIR/JSON representation of the resource before the change (null for`created` events) |
| afterSnapshot     | object (FHIR/JSON) or null | Full FHIR/JSON representation of the resource after the change (null for`deleted` events)  |
| ttl               | integer (Unix epoch)       | Expiry timestamp (creation date + 1 year) used by DynamoDB TTL for automatic housekeeping  |

2. THE Audit_Service SHALL record both a **before** and **after** snapshot of the resource in FHIR/JSON format on every Audit_Record:
   - For `created` events: `beforeSnapshot` SHALL be `null`; `afterSnapshot` SHALL contain the full FHIR/JSON representation of the resource as created.
   - For `updated` events: `beforeSnapshot` SHALL contain the full FHIR/JSON representation of the resource immediately before the change; `afterSnapshot` SHALL contain the full representation after the change. This supports PUT operations where the entire record is replaced.
   - For `deleted` events: `beforeSnapshot` SHALL contain the full FHIR/JSON representation of the resource immediately before deletion; `afterSnapshot` SHALL be `null`.

> **Note:** Because the snapshots capture the complete FHIR/JSON representation of the resource, all resource-specific fields (e.g. endpointStatus, endpointUrl, connectionType, payloadType, productId, healthcareServiceId) are inherently included and do not need to be stored as separate top-level audit fields.

### 3.6 Requirement 6: Access Control for Audit Query API

**User Story:** As an Endpoint Administrator, I want the audit query API to be protected so that only authenticated and authorised callers can retrieve audit records.

> **Note:** The Audit Query API may be exposed via either the AWS API Gateway (IAM-secured) or the Apigee platform (bearer token-secured). Only one access control mode will be required in the initial implementation. Given that RBAC is delayed and the API is likely to be hosted on AWS API Gateway, **IAM-based access control is the recommended option** for the initial release. Bearer token access control is documented below as the alternative should the API be exposed via Apigee or once RBAC is in place.

#### 3.6.1 Acceptance Criteria — Common

1. THE Audit_Query_API SHALL only be accessible to authenticated and authorised callers — unauthenticated requests SHALL be rejected.
2. Authorised callers SHALL be permitted to retrieve Audit_Records for any resource without restriction on `actorOrganisation` (audit records are not restricted by ownership).
3. IF a request to the Audit_Query_API supplies an invalid search parameter name, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `400` and error code `INVALID_PARAMETER`.

#### 3.6.2 Option A: IAM-based access control (AWS API Gateway — recommended for initial release)

1. THE Audit_Query_API SHALL be deployed on AWS API Gateway with IAM authorisation enabled.
2. Requests SHALL be signed with valid AWS IAM credentials (SigV4). Requests with missing or invalid credentials SHALL be rejected with HTTP `403 Forbidden`.
3. Access SHALL be controlled via IAM policies attached to roles granted to authorised teams (e.g. Endpoint Administrators, support staff).

#### 3.6.3 Option B: Bearer token access control (Apigee — alternative / future with RBAC)

1. THE Audit_Query_API SHALL require a valid Bearer_Token on all requests and SHALL return `401 Unauthorized` for requests with a missing, expired, or invalid token.
2. IF a request is made without a valid Bearer_Token, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `401` and error code `SEND_UNAUTHORIZED`.

### 3.7 Requirement 7: Audit Record Immutability and Retention

**User Story:** As an Endpoint Administrator, I want audit records to be permanent and tamper-proof, so that the audit trail can be relied upon for compliance and support investigations.

#### 3.7.1 Acceptance Criteria

1. THE Audit_Service SHALL NOT expose any API operation that allows an Audit_Record to be modified after creation.
2. THE Audit_Service SHALL NOT expose any API operation that allows an Audit_Record to be deleted.
3. THE Audit_Service SHALL retain Audit_Records for a minimum of 1 year from the date of creation.
4. WHILE an Audit_Record exists in the store, THE Audit_Service SHALL return it in query results that match its fields, regardless of the age of the record.
5. THE Audit_Service SHALL automatically remove Audit_Records after the retention period has elapsed by using DynamoDB Time-to-Live (TTL). Each Audit_Record SHALL include a `ttl` attribute set to the Unix epoch timestamp 1 year from the record's creation date.
6. THE Audit_Service SHALL NOT rely on manual or scheduled housekeeping processes to remove expired records — TTL-based expiry SHALL be the sole mechanism for audit record deletion.

### 3.8 Requirement 8: Audit Query API — Error Handling

**User Story:** As an Endpoint Administrator, I want the audit query API to return clear, actionable error responses when I supply invalid parameters, so that I can correct my query without needing to contact support.

#### 3.8.1 Acceptance Criteria

1. IF `date-from` is supplied with a value that is not a valid ISO 8601 date, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `400` and a `diagnostics` message identifying the invalid parameter and its value.

2. IF `date-to` is supplied with a value that is not a valid ISO 8601 date, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `400` and a `diagnostics` message identifying the invalid parameter and its value.

3. IF `date-from` is supplied with a value that is later than `date-to`, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `400` and a `diagnostics` message stating that `date-from` must not be later than `date-to`.

4. IF `date` is supplied together with either `date-from` or `date-to`, THEN THE Audit_Query_API SHALL return a FHIR `OperationOutcome` with HTTP status `400` and a `diagnostics` message stating that `date` cannot be combined with `date-from` or `date-to`.

5. WHEN the Audit_Query_API returns an error response, THE Audit_Query_API SHALL use the FHIR `OperationOutcome` resource format consistent with the error response format used by the rest of the Endpoint Catalog API.

**User Story:** As a platform operator, I want all inbound requests to the Endpoint Catalog API to be logged at the gateway layer, so that I have a complete low-level request trail that complements the application-level audit records.

#### 3.9.1 Acceptance Criteria

1. THE Apigee API gateway SHALL emit an audit log entry to Splunk for every inbound HTTP request to the Endpoint Catalog API, regardless of whether the request succeeds or fails.
2. THE team SHALL agree and document the specific data items to be captured in the Apigee_Audit log before the service enters production. **This is an open action — the data items are not yet defined.**
3. AS A MINIMUM, the agreed data items SHOULD include:

   - Request timestamp (UTC)
   - HTTP method
   - Request path
   - HTTP response status code
   - Caller IP address or network identifier
   - `X-Request-Id` and `X-Correlation-Id` header values (to allow correlation with application logs)
   - `NHSD-End-User-Organisation-ODS` header value
4. THE Apigee_Audit log SHALL be queryable in Splunk by the platform team with a maximum query latency of 5 minutes from request time to search availability.
5. THE Apigee_Audit log SHALL retain entries for a minimum of 1 year.

### 3.10 Requirement 10: Structured Logging

**User Story:** As a support engineer, I want the Endpoint Catalog API to emit structured, machine-parseable logs for every request, so that I can quickly search and filter operational data when investigating issues.

#### 3.10.1 Acceptance Criteria

1. THE Endpoint Catalog API SHALL emit a structured JSON log entry for every inbound request, written to CloudWatch Logs.
2. EACH log entry SHALL include, at minimum:

   - `timestamp` (ISO 8601 UTC)
   - `level` (INFO, WARN, ERROR)
   - `correlationId` (the `X-Correlation-Id` value from the request, or a generated UUID if absent)
   - `requestId` (the `X-Request-Id` value from the request, or a generated UUID if absent)
   - `httpMethod`
   - `path`
   - `statusCode`
   - `durationMs` (request processing time in milliseconds)
   - `actorOrganisation` (ODS code from bearer token, if available)
3. THE Endpoint Catalog API SHALL NOT log bearer token values, patient data, or any data classified as sensitive in its structured log output.
4. THE structured logs SHALL be queryable via CloudWatch Logs Insights using the field names defined above.

### 3.11 Requirement 11: Distributed Tracing

**User Story:** As a support engineer, I want to trace a single request across all components of the Endpoint Catalog API, so that I can quickly identify where a failure or performance bottleneck occurs.

#### 3.11.1 Acceptance Criteria

1. THE Endpoint Catalog API SHALL propagate a `correlationId` through all internal processing for a given request (Lambda invocation, DynamoDB calls, audit writes).
2. IF the inbound request includes an `X-Correlation-Id` header, THEN THE Endpoint Catalog API SHALL use that value as the `correlationId` for the request.
3. IF the inbound request does not include an `X-Correlation-Id` header, THEN THE Endpoint Catalog API SHALL generate a UUID and use it as the `correlationId`.
4. THE `correlationId` SHALL appear in every structured log entry emitted during the processing of that request.
5. THE Endpoint Catalog API SHALL return the `correlationId` in the response as the `X-Correlation-Id` header.

### 3.12 Requirement 12: Health Check Endpoint

**User Story:** As a platform operator, I want the Endpoint Catalog API to expose a health check endpoint, so that monitoring systems can detect when the service is unavailable or degraded.

#### 3.12.1 Acceptance Criteria

1. THE Endpoint Catalog API SHALL expose a `GET /_status` endpoint that returns HTTP `200` with a JSON body `{"status": "pass"}` when the service is healthy.
2. THE health check SHALL verify connectivity to DynamoDB by performing a lightweight operation (e.g. `DescribeTable`) and SHALL return HTTP `503` with `{"status": "fail", "detail": "..."}` if connectivity is lost.
3. THE `GET /_status` endpoint SHALL NOT require authentication (no bearer token required).
4. THE `GET /_status` endpoint SHALL respond within 3 seconds under normal operating conditions.

### 3.13 Requirement 13: Metrics and Instrumentation

**User Story:** As a platform operator, I want the Endpoint Catalog API to emit operational metrics, so that I can build dashboards and detect anomalies in request volume, error rates, and latency.

#### 3.13.1 Acceptance Criteria

1. THE Endpoint Catalog API SHALL emit the following metrics to CloudWatch Metrics:

   - Request count (per endpoint, per HTTP method)
   - Error count (4xx and 5xx, per endpoint)
   - Request latency (P50, P95, P99, per endpoint)
   - Audit record write count
   - Audit query execution count
2. THE Endpoint Catalog API SHALL emit custom metrics using CloudWatch Embedded Metric Format (EMF) from within the Lambda function.
3. THE built-in AWS metrics (API Gateway request count, Lambda duration, Lambda errors, DynamoDB consumed capacity) SHALL be available without additional application code.

### 3.14 Requirement 14: Alerting Thresholds

**User Story:** As a platform operator, I want automated alerts when key operational metrics exceed defined thresholds, so that the team is notified of potential issues before they affect users.

#### 3.14.1 Acceptance Criteria

1. THE following CloudWatch Alarms SHALL be configured:


| Alarm                | Metric                        | Condition                         | Period | Action                   |
| ---------------------- | ------------------------------- | ----------------------------------- | -------- | -------------------------- |
| High error rate      | API GW 5XXError               | > 5% of total requests            | 5 min  | SNS → team notification |
| High latency         | API GW Latency                | P95 > 2000 ms                     | 5 min  | SNS → team notification |
| Health check failing | Custom HealthCheckStatus      | = 0 (unhealthy) for 2 consecutive | 1 min  | SNS → team notification |
| Audit write failure  | Custom AuditRecordWriteFailed | > 0                               | 1 min  | SNS → team notification |
| Lambda throttles     | Lambda Throttles              | > 0                               | 1 min  | SNS → team notification |
| DynamoDB throttles   | DynamoDB ThrottledRequests    | > 0                               | 5 min  | SNS → team notification |

2. All alarms SHALL publish to an SNS topic. The team subscribes via email, Slack (via AWS Chatbot), or PagerDuty (via SNS → HTTPS endpoint).

### 3.15 Requirement 15: Log Retention and Accessibility

**User Story:** As a platform operator, I want logs to be retained for a defined period and accessible for investigation, so that I can troubleshoot issues that occurred in the past.

#### 3.15.1 Acceptance Criteria

1. CloudWatch Logs for Lambda functions and API Gateway access logs SHALL be retained for 90 days in CloudWatch.
2. After 90 days, logs SHALL be archived to S3 for a further retention period of 1 year.
3. Archived logs SHALL be queryable using S3 Select or Athena if needed for historical investigation.
4. Audit records in DynamoDB SHALL be retained for 1 year, after which they are automatically removed via DynamoDB TTL (per Requirement 7).

## 4 Endpoint Catalogue NFR


| #      | Requirement                | Category      | Acceptance Criteria                                                                                                                                                                |
| --------| ----------------------------| ---------------| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| NFR-01 | Audit write latency        | Performance   | Adding an audit record SHALL NOT increase the P95 latency of the originating write operation by more than 100ms                                                                    |
| NFR-02 | Audit query latency        | Performance   | GET /AuditEvent queries SHALL return results within 2 seconds (P95) for result sets of up to 1000 records                                                                          |
| NFR-03 | Audit query pagination     | Performance   | Paginated queries SHALL return each page within 2 seconds (P95) regardless of offset                                                                                               |
| NFR-04 | Audit storage durability   | Reliability   | Audit_Records SHALL be stored with 99.999999999% (11 nines) durability                                                                                                             |
| NFR-05 | Audit availability         | Reliability   | The Audit_Query_API SHALL be available 99.9% of the time measured monthly                                                                                                          |
| NFR-06 | Audit write resilience     | Reliability   | IF the audit store is temporarily unavailable, THE Audit_Service SHALL retry the write with exponential backoff (max 3 retries) and SHALL NOT fail the originating write operation |
| NFR-07 | Audit write dead-letter    | Reliability   | IF all retries are exhausted, THE Audit_Service SHALL place the failed audit record on a dead-letter queue for later reprocessing and SHALL emit an AuditRecordWriteFailed metric  |
| NFR-08 | Structured log volume      | Capacity      | The logging infrastructure SHALL support ingestion of up to 100 requests/second without data loss                                                                                  |
| NFR-09 | Health check response time | Performance   | GET /_status SHALL respond within 3 seconds under normal conditions                                                                                                                |
| NFR-10 | Metric emission latency    | Observability | Custom metrics SHALL appear in CloudWatch within 60 seconds of emission                                                                                                            |
| NFR-11 | Alert notification latency | Observability | Alarm state changes SHALL trigger SNS notification within 5 minutes of the threshold breach                                                                                        |
| NFR-12 | Log search latency         | Observability | CloudWatch Logs Insights queries SHALL return results within 30 seconds for queries spanning 24 hours of log data                                                                  |

| NFR-13 | Audit record size          | Capacity      | Individual Audit_Records SHALL not exceed 400KB (DynamoDB item size limit)                                                                                                         |
| NFR-14 | Correlation ID propagation | Traceability  | 100% of structured log entries for a given request SHALL contain the same correlationId                                                                                            |
| NFR-15 | No sensitive data in logs  | Security      | Structured logs SHALL NOT contain bearer token values, patient identifiers, or data classified as sensitive                                                                        |

## 5 Suggested Design

### 5.1 Design Document — Endpoint Catalog Audit & Observability

#### 5.1.1 Overview

This design covers the implementation of audit, logging, metrics, alerting, and health-check capabilities for the Endpoint Catalog API. The design uses AWS-native services to minimise operational overhead and integrates with the existing serverless architecture (API Gateway + Lambda + DynamoDB).

**Key design decisions:**

- Audit records are written synchronously within the same Lambda invocation as the originating write to guarantee consistency
- A dedicated DynamoDB table (`epc-audit`) stores audit records separately from resource data
- Structured logs use CloudWatch Embedded Metric Format (EMF) to emit both logs and metrics in a single write
- Health checks are lightweight and unauthenticated
- All observability data flows to CloudWatch; Splunk integration is handled at the Apigee layer

#### 5.1.2 Architecture Diagram

```
Client → Apigee (→ Splunk) → API Gateway (access logs → CloudWatch) → Lambda
                                                                         ├── DynamoDB (resource tables)
                                                                         ├── DynamoDB (epc-audit table)
                                                                         └── CloudWatch (structured logs + EMF metrics)

CloudWatch Alarms → SNS → Team notification (Email/Slack/PagerDuty)
CloudWatch Dashboard ← CloudWatch Metrics + Logs Insights
```

#### 5.1.3 Component Design

##### 5.1.3.1 1. Audit Table (DynamoDB)

**Table name:** `epc-audit`

**Key schema:**


| Attribute | Type   | Role                                   |
| ----------- | -------- | ---------------------------------------- |
| PK        | String | `RESOURCE#{resourceType}#{resourceId}` |
| SK        | String | `AUDIT#{timestamp}#{id}`               |
| GSI1PK    | String | `ORG#{actorOrganisation}`              |
| GSI1SK    | String | `AUDIT#{timestamp}#{id}`               |

**Global Secondary Indexes:**

- **GSI1** (`GSI1PK`, `GSI1SK`): Supports queries by organisation + time range
- **GSI2** (`resourceType`, `SK`): Supports queries by resource type + time range

**Table settings:**

- Billing mode: On-demand (PAY_PER_REQUEST)
- Point-in-time recovery: Enabled
- Deletion protection: Enabled
- Encryption: AWS-managed key (default)
- TTL attribute: `ttl` (enables automatic deletion of records after the 1-year retention period)

##### 5.1.3.2 2. Audit Write Logic

The audit write occurs inside the same Lambda invocation as the resource write:

```
1. Validate request & authenticate
2. Perform resource write (DynamoDB PutItem/UpdateItem/DeleteItem)
3. IF resource write succeeds:
   a. Construct Audit_Record from operation context + bearer token claims
   b. Set `ttl` attribute to current timestamp + 1 year (Unix epoch seconds)
   c. Write Audit_Record to epc-audit table (PutItem)
   d. IF audit write fails:
      i.  Retry with exponential backoff (max 3 attempts)
      ii. IF all retries fail: emit AuditRecordWriteFailed metric, log error, continue
          (do NOT fail the originating response)
4. Return response to caller
```

**Change_Type derivation:**

```
IF method == POST → "created"
IF method == DELETE → "deleted"
IF method == PUT or PATCH → "updated"
```

##### 5.1.3.3 3. Audit Query Implementation

The `GET /AuditEvent` endpoint is implemented as a Lambda handler that:

1. Validates the bearer token (same auth as other endpoints)
2. Parses and validates query parameters
3. Determines the optimal DynamoDB query strategy:
   - If `healthcare-service-id` or `endpoint-id` → query by PK (most selective)
   - If `organisation` → query GSI1
   - If `resourceType` only → query GSI2
   - Otherwise → Scan with filter (acceptable for low-volume audit table)
4. Applies remaining parameters as DynamoDB filter expressions
5. Handles pagination via `_count` and `_offset`
6. Returns FHIR Bundle with pagination links

##### 5.1.3.4 4. Structured Logging

All Lambda handlers use a shared logging utility that outputs JSON to stdout (captured by CloudWatch Logs):

```json
{
  "timestamp": "2026-08-06T12:34:56.789Z",
  "level": "INFO",
  "correlationId": "abc-123-def",
  "requestId": "ghi-456-jkl",
  "httpMethod": "PUT",
  "path": "/Endpoint/ep-001",
  "statusCode": 200,
  "durationMs": 142,
  "actorOrganisation": "X26",
  "message": "Request completed"
}
```

Error logs include additional fields: `errorCode`, `errorMessage`, `stackTrace` (for 5xx only).

##### 5.1.3.5 5. Correlation ID Propagation

```
Inbound request
  └── X-Correlation-Id header present? 
       ├── YES → use as correlationId
       └── NO → generate UUID as correlationId
  └── Attach correlationId to:
       ├── All structured log entries
       ├── Audit_Record (optional metadata field)
       └── Response header (X-Correlation-Id)
```

##### 5.1.3.6 6. Health Check

`GET /_status` is routed through API Gateway to a dedicated Lambda handler (or shared handler with path routing):

```
1. Call DynamoDB DescribeTable on the resource table
2. IF successful within 3s → return 200 {"status": "pass"}
3. IF fails or times out → return 503 {"status": "fail", "detail": "DynamoDB connectivity check failed"}
```

- No authentication required
- Separate from business logic to avoid auth overhead
- Can be polled by Route 53 health checks or external monitors

##### 5.1.3.7 7. Metrics (CloudWatch EMF)

Custom metrics emitted via Embedded Metric Format:


| Metric Name            | Dimensions               | Unit          | Emitted When                         |
| ------------------------ | -------------------------- | --------------- | -------------------------------------- |
| AuditRecordWritten     | resourceType, changeType | Count         | Audit record successfully written    |
| AuditRecordWriteFailed | resourceType             | Count         | Audit write failed after all retries |
| AuditQueryExecuted     | —                       | Count         | GET /AuditEvent request completed    |
| AuditQueryLatency      | —                       | Milliseconds  | GET /AuditEvent request completed    |
| HealthCheckStatus      | —                       | None (0 or 1) | GET /_status invoked                 |

Built-in AWS metrics (no code required):

- API Gateway: `Count`, `4XXError`, `5XXError`, `Latency`, `IntegrationLatency`
- Lambda: `Invocations`, `Duration`, `Errors`, `Throttles`, `ConcurrentExecutions`
- DynamoDB: `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `ThrottledRequests`

##### 5.1.3.8 8. Alerting (CloudWatch Alarms)


| Alarm                | Metric                        | Condition                         | Period | Action                   |
| ---------------------- | ------------------------------- | ----------------------------------- | -------- | -------------------------- |
| High error rate      | API GW 5XXError               | > 5% of total requests            | 5 min  | SNS → team notification |
| High latency         | API GW Latency                | P95 > 2000 ms                     | 5 min  | SNS → team notification |
| Health check failing | Custom HealthCheckStatus      | = 0 (unhealthy) for 2 consecutive | 1 min  | SNS → team notification |
| Audit write failure  | Custom AuditRecordWriteFailed | > 0                               | 1 min  | SNS → team notification |
| Lambda throttles     | Lambda Throttles              | > 0                               | 1 min  | SNS → team notification |
| DynamoDB throttles   | DynamoDB ThrottledRequests    | > 0                               | 5 min  | SNS → team notification |

**Notification:** All alarms publish to an SNS topic. The team subscribes via email, Slack (via AWS Chatbot), or PagerDuty (via SNS → HTTPS endpoint).

##### 5.1.3.9 9. CloudWatch Dashboard

A single CloudWatch dashboard named `EPC-Operations` provides three sections:

###### Section 1: Audit Activity


| Widget                              | Type                  | Data Source                                          |
| ------------------------------------- | ----------------------- | ------------------------------------------------------ |
| Audit records created (last 24h)    | Number                | AuditRecordWritten sum                               |
| Audit records by resource type      | Stacked bar chart     | AuditRecordWritten grouped by resourceType dimension |
| Audit records by change type        | Pie chart             | AuditRecordWritten grouped by changeType dimension   |
| Audit query volume                  | Line graph            | AuditQueryExecuted over time                         |
| Audit query latency (P50, P95, P99) | Line graph            | AuditQueryLatency percentiles                        |
| Audit write failures                | Number (alarm status) | AuditRecordWriteFailed sum                           |

###### Section 2: System Health


| Widget                       | Type                  | Data Source                                                    |
| ------------------------------ | ----------------------- | ---------------------------------------------------------------- |
| API request volume           | Line graph            | API GW Count                                                   |
| Error rate (4xx / 5xx)       | Line graph            | API GW 4XXError, 5XXError                                      |
| API latency (P50, P95, P99)  | Line graph            | API GW Latency percentiles                                     |
| Lambda duration (P50, P95)   | Line graph            | Lambda Duration                                                |
| Lambda errors & throttles    | Line graph            | Lambda Errors, Throttles                                       |
| Lambda concurrent executions | Line graph            | Lambda ConcurrentExecutions                                    |
| DynamoDB consumed capacity   | Line graph            | DynamoDB ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits |
| DynamoDB throttled requests  | Number (alarm status) | DynamoDB ThrottledRequests                                     |
| Health check status          | Alarm status widget   | HealthCheckStatus alarm                                        |
| Active alarms                | Alarm status widget   | All EPC alarms                                                 |

###### Section 3: Logs


| Widget                      | Type              | Data Source                                                                  |
| ----------------------------- | ------------------- | ------------------------------------------------------------------------------ |
| Recent errors               | Log table (query) | CloudWatch Logs Insights:`filter level = "ERROR"` from Lambda log group      |
| Recent audit write failures | Log table (query) | CloudWatch Logs Insights:`filter message like "Audit write failed"`          |
| Request trace lookup        | Log table (query) | CloudWatch Logs Insights:`filter correlationId = "{input}"` — parameterised |
| Slow requests (> 1s)        | Log table (query) | CloudWatch Logs Insights:`filter durationMs > 1000`                          |

###### Dashboard-as-Code

The dashboard is defined in the infrastructure-as-code (Terraform) as a `aws_cloudwatch_dashboard` resource with the JSON body specifying all widgets. This ensures the dashboard is version-controlled and reproducible across environments.

#### 5.1.4 Log Retention Strategy


| Log source               | CloudWatch retention | Archive            | Notes                                                    |
| -------------------------- | ---------------------- | -------------------- | ---------------------------------------------------------- |
| Lambda application logs  | 90 days              | S3 export (1 year) | Via CloudWatch Logs subscription → S3                   |
| API Gateway access logs  | 90 days              | S3 export (1 year) | Same mechanism                                           |
| Audit records (DynamoDB) | 1 year (TTL-managed) | —                 | Automatically expired via DynamoDB TTL per Requirement 7 |

For the S3 archive:

- Use CloudWatch Logs subscription filter → Kinesis Data Firehose → S3 (Glacier after 30 days)
- Or use the native CloudWatch Logs `CreateExportTask` on a daily schedule via EventBridge + Lambda

#### 5.1.5 Security Considerations


| Concern                   | Mitigation                                                                                                                                                                            |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Audit record tampering    | DynamoDB table has no`DeleteItem` or `UpdateItem` IAM permissions granted to the Lambda execution role for the `epc-audit` table. Only `PutItem` and `Query`/`GetItem` are permitted. |
| Accidental table deletion | DynamoDB deletion protection enabled                                                                                                                                                  |
| Log data sensitivity      | Lambda logs do not include bearer token values or patient data. ODS codes and user IDs are logged for traceability.                                                                   |
| Dashboard access          | CloudWatch dashboard access controlled via IAM policies — only platform team roles can view                                                                                          |
| Audit query access        | GET /AuditEvent requires a valid bearer token (same auth as all other endpoints)                                                                                                      |

#### 5.1.6 Cost Estimate (Indicative)


| Component                       | Cost driver                                       | Estimate at 10k writes/day            |
| --------------------------------- | --------------------------------------------------- | --------------------------------------- |
| DynamoDB`epc-audit` (on-demand) | Write: $1.25/million WRU, Read: $0.25/million RRU | ~$0.40/month writes, reads negligible |
| DynamoDB storage                | $0.25/GB/month                                    | ~$0.10/month (first year, growing)    |
| CloudWatch Logs                 | $0.50/GB ingested                                 | ~$2–5/month                          |
| CloudWatch Metrics (custom)     | $0.30/metric/month (first 10k)                    | ~$1.50/month (5 custom metrics)       |
| CloudWatch Dashboard            | $3/dashboard/month                                | $3/month                              |
| CloudWatch Alarms               | $0.10/alarm/month                                 | ~$0.60/month (6 alarms)               |
| **Total incremental**           |                                                   | **~$8–11/month**                     |

#### 5.1.7 Summary of AWS Services Used


| Requirement                  | AWS Service                        | Feature                                                  |
| ------------------------------ | ------------------------------------ | ---------------------------------------------------------- |
| Audit storage (Req 1, 5, 7)  | DynamoDB                           | Dedicated table, PITR, deletion protection               |
| Audit query (Req 4, 8)       | DynamoDB + Lambda                  | GSI queries, filter expressions                          |
| Structured logging (Req 10)  | CloudWatch Logs                    | JSON log format, Log Insights                            |
| Distributed tracing (Req 11) | CloudWatch Logs + X-Ray (optional) | Correlation ID propagation                               |
| Health check (Req 12)        | Lambda + API Gateway               | Unauthenticated route, DynamoDB connectivity check       |
| Metrics (Req 13)             | CloudWatch Metrics                 | Built-in (API GW, Lambda, DynamoDB) + EMF custom metrics |
| Alerting (Req 14)            | CloudWatch Alarms + SNS            | Threshold-based alarms, team notification                |
| Log retention (Req 15)       | CloudWatch Logs + S3               | 90-day hot, 1-year cold archive                          |
| Dashboard                    | CloudWatch Dashboards              | Single pane: audit, health, logs                         |
| Gateway audit (Req 9)        | API Gateway access logging         | JSON access logs to CloudWatch                           |

---

## 6 NFR Coverage Mapping

The following table shows how each Endpoint Catalogue NFR (v1, 6 August 2026) is addressed by the requirements in this document.


| NFR   | NFR Summary                                                                 | Coverage Status        | Mapped Audit Requirements                                                                        | How Met                                                                                                                                                                                                                                                                                                               |
| ------- | ----------------------------------------------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NFR-1 | Every change is recorded automatically; failed changes produce no entry     | Fully met              | Req 1 (Audit Record Creation on Write Operations), Req 3 (Audit Coverage for All Resource Types) | All successful POST/PUT/PATCH/DELETE operations on all four resource types automatically produce an Audit_Record. Failed operations (4xx/5xx) do not create entries.                                                                                                                                                  |
| NFR-2 | Each entry shows who, what org, when, resource affected, and change type    | Mostly met             | Req 2 (Actor Identity from Bearer Token), Req 5 (Audit Record Content)                           | actorId, actorOrganisation (ODS), timestamp, resourceType, resourceId, and changeType are all recorded.**Note:** Full individual-user attribution cannot be met until RBAC is in place; until then audit records identity at organisation level only. Gaps: no local-time display; no distinct "renamed" change type. |
| NFR-3 | Before-and-after values for fields that changed                             | Fully met              | Req 5 (Audit Record Content)                                                                     | Both a`beforeSnapshot` and `afterSnapshot` are stored on every Audit_Record as full FHIR/JSON representations. This captures all field changes including full-record replacements via PUT. Consumers can diff the two snapshots to identify exactly which fields changed.                                             |
| NFR-4 | Search by ODS code, service name, endpoint, status, org, date range         | Mostly met             | Req 4 (Audit Query API — Search and Retrieval)                                                  | Supports search by organisation (ODS), healthcare-service-id, endpoint-id, date/date-from/date-to, url, connection-type, payload-type, product-id. Gap: no partial/fuzzy match on service name.                                                                                                                       |
| NFR-5 | History is tamper-proof, no duplicates, no gaps, retained 1 year            | Mostly met             | Req 7 (Audit Record Immutability and Retention), Req 1 (AC 5 — append-only store)               | No modify/delete API exposed; DynamoDB IAM restricts to PutItem + Query only; retention is 1 year (matches the NFR ask), with automatic TTL-based housekeeping. Gap: no explicit idempotency mechanism to guarantee zero duplicates.                                                                                  |
| NFR-6 | Drill-down from service to endpoints, suppliers, and history in 1–2 clicks | Not met (out of scope) | —                                                                                               | This is a UI/UX requirement. audit.md is an API-level specification only. Needs separate front-end spec.                                                                                                                                                                                                              |
| NFR-7 | Errors are clear, actionable, include a correlation ID                      | Fully met              | Req 8 (Audit Query API — Error Handling), Req 11 (Distributed Tracing)                          | FHIR OperationOutcome responses with diagnostics and specific field/value identification. Correlation ID returned in X-Correlation-Id header and propagated through all logs.                                                                                                                                         |
| NFR-8 | Performance holds as scope expands; downtime communicated                   | Partially met          | NFR-02 (Audit query latency ≤ 2s P95), NFR-09 (Health check ≤ 3s)                              | Latency targets defined. DynamoDB on-demand scaling handles load growth. Gap: no explicit data-growth/archiving strategy; no downtime communication process documented.                                                                                                                                               |
| NFR-9 | Changes visible to others within seconds; true ordering preserved           | Partially met          | Req 4 (AC 7 — results ordered by timestamp descending)                                          | Ordering by timestamp is guaranteed in query results. Gap: no explicit read-consistency model stated for audit queries (DynamoDB GSI reads are eventually consistent by default, typically < 1s).                                                                                                                     |

---

## 7 NFR Alignment Gap Analysis

The following table captures gaps between this document (EPC-INV010) and the Endpoint Catalogue NFR requirements (v1, 6 August 2026).


| NFR   | NFR Requirement                                                                       | Gap                                                                                                                                            | Severity | Notes / Suggested Action                                                                                                                                                                |
| -------| ---------------------------------------------------------------------------------------| ------------------------------------------------------------------------------------------------------------------------------------------------| ----------| -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| NFR-1 | "supplier add/remove" produces an audit entry                                         | audit.md does not reference a Supplier resource type — only HealthcareService, Template, Endpoint, and List                                    | Medium   | Clarify whether supplier management is in scope and how it maps to the four defined resource types                                                                                      |
| NFR-2 | Timestamps shown in readable local time, not just raw UTC                             | audit.md stores and returns UTC only — no local-time representation                                                                            | Medium   | Decide whether the API should return a secondary localised timestamp field or whether this is a UI-layer concern                                                                        |
| NFR-2 | "renamed" listed as a distinct change type                                            | audit.md only defines`created`, `updated`, `deleted` — a rename would be recorded as `updated` with no distinct code                           | Low      | Consider adding a`renamed` change type or document that renames are a subset of `updated`                                                                                               |
| NFR-2 | Individual user-level identity ("who made the change")                                | Cannot be fully met until RBAC is in place — until then, audit records identity at organisation level (ODS code) only                          | High     | Dependency on RBAC implementation. Once RBAC is live and individual user claims are available in the bearer token, actorId will capture the specific user. Track as a known constraint. |
| NFR-4 | Search by service name including partial/fuzzy matches                                | Audit Query API only supports exact-match identifiers (healthcare-service-id, endpoint-id, ODS code) — no free-text or partial-match parameter | Medium   | Either add a text-search parameter to the Audit Query API or document this as a UI-layer feature                                                                                        |
| NFR-5 | "The same change always produces exactly one entry — no duplicates, no gaps"          | Retry logic (exponential backoff on audit write failure) could produce duplicates if a write succeeds but the response times out               | Medium   | Add an explicit idempotency mechanism (e.g. conditional PutItem with audit record ID) to prevent duplicate entries                                                                      |
| NFR-6 | Drill-down navigation: service → endpoints → suppliers → history in one or two clicks | Not addressed — audit.md is an API-level specification with no UI/UX component                                                                 | Medium   | Acknowledge as a UI requirement and track separately in a front-end spec                                                                                                                |
| NFR-8 | Performance holds as scope expands; scheduled downtime communicated in advance        | Latency NFRs exist (2s P95) but no strategy for data growth over time (archiving, partitioning) and no downtime communication process          | Low      | Document a data-growth strategy (e.g. time-based partitioning) and define a downtime communication process                                                                              |
| NFR-9 | A change is visible to others within a few seconds; history reflects true order       | No explicit consistency model stated — DynamoDB GSI queries use eventually-consistent reads by default                                         | Medium   | Specify whether strongly-consistent reads are used for audit queries or document the expected propagation delay (typically < 1s for DynamoDB)                                           |
