# Open Questions and Outstanding Items

This page consolidates all unresolved questions, TBDs, and items needing decisions
from across the Endpoint Catalog documentation. Extracted on 28 May 2026.

---

| # | Source Document | Section | Question / Open Item | Owner | Answer | Status |
|---|---|---|---|---|---|---|
| 1 | Authentication and Authorisation | Open questions | Is CIS2 available to non-NHS users (e.g. private supplier staff)? If not, what's the auth path for suppliers? | Security / CIS2 team | | Open |
| 2 | Authentication and Authorisation | Open questions | Should write operations be restricted to user-restricted mode only, or should some writes be permitted via app-restricted (for supplier automation)? | Product / Architecture | | Open |
| 3 | Authentication and Authorisation | Open questions | How are EPC-specific roles (Endpoint Admin, HS Admin, etc.) mapped to CIS2 role assertions? Custom role codes or existing CIS2 roles? | IOPS / CIS2 team | | Open |
| 4 | Authentication and Authorisation | Open questions | Is the admin UI a separate application or part of the Digital Onboarding Service? | Product | | Open |
| 5 | Authentication and Authorisation | Open questions | Should Product ID be the primary ownership key instead of (or in addition to) ODS code? | Architecture | | Open |
| 6 | Authentication and Authorisation | Authorisation (Note) | The `header: private` requirement is currently under discussion and may be struck from the specification. Decision needed on whether private endpoint address redaction will remain. | Not assigned | | Open |
| 7 | BaRS OAS Alignment, Authentication, and API Routing | Open Questions | Should the BaRS OAS retain `GET /Endpoint` (read-only consumer lookup) or should all EPC operations be removed? | Architecture | | Open |
| 8 | BaRS OAS Alignment, Authentication, and API Routing | Open Questions | What Apigee header names are used to forward token claims to the backend? (e.g. `NHSD-Client-Id`, `NHSD-User-Id`) | Platform team | | Open |
| 9 | BaRS OAS Alignment, Authentication, and API Routing | Open Questions | Should `/metadata` return a combined capability statement or separate ones per service? Depends on whether BaRS and EPC share a base URL. | Architecture | | Open |
| 10 | BaRS OAS Alignment, Authentication, and API Routing | Open Questions | How does the BaRS Proxy's internal EPC call authenticate? Service account credentials or internal Apigee policy? | Platform team | | Open |
| 11 | BaRS OAS Alignment, Authentication, and API Routing | Open Questions | Are the `X-Request-Id` and `X-Correlation-Id` headers sufficient, or do the EPC-specific definitions need to be added separately? | Architecture | | Open |
| 12 | Managing Endpoint Templates | Step 1 — Gather the required data | Product Id format is under investigation and has not yet been confirmed. The format (e.g. `PinnaclePharmOutcomes-v2024.12.12`) is illustrative only. | Not assigned | | Open |
| 13 | Data Migration to the Endpoint Catalog | Overview (warning) | Document is incomplete — source database details including database name, table names, table structures, and field mappings have not yet been documented. A database schema review is needed. | Not assigned | | Open |
| 14 | Data Migration to the Endpoint Catalog | Step 1 — Migrate Templates | Product Id construction mechanism not yet defined. Existing database holds only supplier name, not a versioned product identifier. Must be resolved before Step 1 can be executed. | Not assigned | | Open |
| 15 | Disaster Recovery | Open Actions | Confirm RPO/RTO targets with service owner. | Tech lead | | Open |
| 16 | Disaster Recovery | Open Actions | Confirm service tier classification (Gold/Silver). | Product owner | | Open |
| 17 | Disaster Recovery | Open Actions | Decide on multi-region requirement. | Architecture review | | Open |
| 18 | Disaster Recovery | Open Actions | Schedule first quarterly DR test. | Platform team | | Open |
| 19 | Disaster Recovery | Open Actions | Define on-call rota and escalation path. | Delivery manager | | Open |
| 20 | Disaster Recovery | References | Review the Red Lines Backup Scenario document and confirm that RPO/RTO targets, backup retention periods, and testing cadence are compliant. | Not assigned | | Open |
| 21 | Duplicate Detection in the Endpoint Catalog | Endpoints (Status note) | Whether `status` should be included in the Endpoint duplicate definition has not yet been confirmed (e.g. should an `entered-in-error` Endpoint with overlapping period block creation of a new one?). | Not assigned | | Open |
| 22 | Endpoint Ordering — Option B: Priority Extension | Open Questions | Should priority be per connection type or global across all endpoints on a service? | Architecture / IOPS | | Open |
| 23 | Endpoint Ordering — Option B: Priority Extension | Open Questions | What is the correct extension URL namespace? (`endpoint-priority` vs `endpoint-failover-priority`) | IOPS / FHIR governance | | Open |
| 24 | Endpoint Ordering — Option B: Priority Extension | Open Questions | Should the extension be mandatory on all Endpoints, or optional (with unranked endpoints treated as lowest priority)? | Product | | Open |
| 25 | Endpoint Ordering — Option B: Priority Extension | Open Questions | Does DoS have requirements beyond simple numeric ranking (e.g. weighted routing, percentage-based distribution)? | DoS leads | | Open |
| 26 | Endpoint Ordering — Option B: Priority Extension | Open Questions | Can we take this proposal to IOPS for a second opinion on the extension approach vs the List approach? | Architecture | | Open |
| 27 | Endpoint Ordering — Options Comparison (Condensed) | Open Questions | Should priority be per connection type or global across all endpoints on a service? | IOPS / DoS Leads | | Open |
| 28 | Endpoint Ordering — Options Comparison (Condensed) | Open Questions | What is the correct extension URL namespace? | IOPS / DoS Leads | | Open |
| 29 | Endpoint Ordering — Options Comparison (Condensed) | Open Questions | Should the extension be mandatory or optional (unranked = lowest priority)? | IOPS / DoS Leads | | Open |
| 30 | Endpoint Ordering — Options Comparison (Condensed) | Open Questions | Does DoS need anything beyond simple numeric ranking (weighted routing, percentage-based)? | IOPS / DoS Leads | | Open |
| 31 | Endpoint Ordering — Options Comparison (Condensed) | Open Questions | Can we take this to IOPS for a second opinion? | IOPS / DoS Leads | | Open |
| 32 | Endpoint Visibility: Status and Period | `_include` differences | Architecture decision needed: should `_include` on GET /HealthcareService return bare Endpoints, fully-resolved Endpoints, or not be supported at all? Affects Lambda complexity and consumer contract. | Architecture | | Open |
| 33 | Endpoint Visibility: Status and Period | Gap 1: Template status mutability | Requirements say Template status is "fixed" but design allows status changes. Needs confirmation on whether Templates can transition between active/suspended. | Product owner | | Open |
| 34 | Endpoint Visibility: Status and Period | Gap 2: Period requirement | Confirm whether the API should enforce `period.start` as mandatory on `POST /Endpoint` (aligning with requirements flowchart) or treat it as optional. | Product owner | | Open |
| 35 | Endpoint Visibility: Status and Period | Gap 5: Connectivity validation | Requirements state endpoint must be live and connectable when status is set to Active (EPCFUNC-52). Not currently in the API design — noted as a UI concern. | Not assigned | | Open |
| 36 | Interim Support Process — Direct AWS Access vs API Access | Open Questions | What identity does the batch Lambda use when calling the API? A dedicated service account with R&M team ODS code? | Architecture / Security | | Open |
| 37 | Interim Support Process — Direct AWS Access vs API Access | Open Questions | Should the CSV include a "requested by" field so the audit trail captures which R&M team member initiated the batch? | Product | | Open |
| 38 | Interim Support Process — Direct AWS Access vs API Access | Open Questions | What rate limit is appropriate for the batch service account? | Platform team | | Open |
| 39 | Interim Support Process — Direct AWS Access vs API Access | Open Questions | Should failed rows be retried automatically or reported for manual review? | R&M team | | Open |
| 40 | Interim Support Process — Direct AWS Access vs API Access | Open Questions | Is the interim process expected to run for months or years? This affects how much investment is justified. | Product | | Open |
| 41 | Observability | Open items | Confirm ITOC log forwarding mechanism (Logs destination vs Firehose). | Platform team | | Open |
| 42 | Observability | Open items | Confirm ODIN onboarding timeline and integration pattern. | ODIN team | | Open |
| 43 | Observability | Open items | Confirm CSOC log forwarding requirements (format, destination, frequency). | Security team | | Open |
| 44 | Observability | Open items | Agree custom metric naming conventions with ODIN team. | EPC + ODIN | | Open |
| 45 | Observability | Open items | Confirm funding model for ODIN usage. | Programme | | Open |
| 46 | Observability | Open items | Define log retention beyond 90 days (archive to S3 Glacier?). | Architecture | | Open |
| 47 | Observability — ODIN-First Approach | Prerequisites and open items | ODIN onboarding — register EPC as a service, obtain OTLP endpoint and auth token. | EPC team + ODIN team | | Open |
| 48 | Observability — ODIN-First Approach | Prerequisites and open items | Confirm ODIN supports AWS Lambda OTel layer export (OTLP/HTTP). | ODIN team | | Open |
| 49 | Observability — ODIN-First Approach | Prerequisites and open items | Confirm ODIN funding model for EPC (included in platform cost or per-service charge). | Programme | | Open |
| 50 | Observability — ODIN-First Approach | Prerequisites and open items | Confirm CSOC access pattern (shared Grafana dashboard or dedicated log export). | Security team | | Open |
| 51 | Observability — ODIN-First Approach | Prerequisites and open items | Confirm ITOC already uses ODIN for service health (avoids duplicate integration). | ITOC team | | Open |
| 52 | Observability — ODIN-First Approach | Prerequisites and open items | Agree metric naming conventions and label cardinality limits with ODIN team. | EPC + ODIN | | Open |
| 53 | Observability — ODIN-First Approach | Prerequisites and open items | Confirm trace sampling strategy for production volume. | EPC team | | Open |
| 54 | Resilience and Availability | Service Level Target | Confirm the availability target with the service owner. If classified as Gold/Tier 1, a 99.95% target may be required, necessitating multi-region deployment. | Not assigned | | Open |
| 55 | Endpoint Templates — Pros and Cons | Summary (Key open question) | Whether the Lambda should automatically apply List order on `GET /HealthcareService` and `GET /Endpoint` responses, eliminating the two-step consumer pattern. | Not assigned | | Open |
| 56 | Endpoint Templates — Pros and Cons | Cons (Con 2) | ProductId format is unresolved — this is a hard blocker for data migration. | Not assigned | | Open |
| 57 | DUEC Endpoints | Overview (warning) | The connectionType and payloadType codes used for DUEC have not been confirmed with the DUEC programme. Needs review and validation before implementation. | DUEC team | | Open |

---

## Decisions Made

The following design decisions have been documented and adopted across the Endpoint
Catalog documentation.

| # | Source Document | Section | Decision | Rationale | Validated | Validated Date |
|---|---|---|---|---|---|---|
| 1 | Duplicate Detection in the Endpoint Catalog | Templates | A Template is a duplicate only if another **active** Template exists with the same `productId` + `connectionType` + `payloadType`; non-active Templates do not block creation. | A withdrawn Template should not prevent a legitimate replacement. | | |
| 2 | Duplicate Detection in the Endpoint Catalog | Endpoints | An Endpoint is a duplicate if another Endpoint exists with the same parent Template and an overlapping `period`. | The combination of *which template* and *when* defines identity. | | |
| 3 | Duplicate Detection in the Endpoint Catalog | Endpoints (no period) | An Endpoint with no `period` is treated as unbounded — overlaps everything; only one per Template can exist without a period. | Prevents ambiguity from unbounded Endpoints conflicting with everything. | | |
| 4 | Duplicate Detection in the Endpoint Catalog | HealthcareServices | A HealthcareService is a duplicate if another exists with the same `identifier.system` + `identifier.value`. | System and value together form the identity. | | |
| 5 | Duplicate Detection in the Endpoint Catalog | Lists | Only one `current` List per HealthcareService; historical (`retired`) Lists are permitted. | Only one active ordering should exist per service. | | |
| 6 | Duplicate Detection in the Endpoint Catalog | HealthcareService endpoint array | `HealthcareService.endpoint[]` must not contain repeated references; rejected with `422`. | Repeated references are meaningless. | | |
| 7 | Duplicate Detection in the Endpoint Catalog | API responses | All duplicate detection is enforced server-side; callers do not pre-check — API returns `409 Conflict`. | Simplifies clients and guarantees consistency. | | |
| 8 | Authentication and Authorisation | Access modes | Two access modes: application-restricted (signed JWT) for system-to-system, and user-restricted (CIS2) for admin operations. | Automated lookups don't need a human; admin changes need individual identity for audit. | | |
| 9 | Authentication and Authorisation | Ownership model | Authorisation is based on resource ownership via ODS code match; mismatch → `403 Forbidden`. | Prevents cross-organisation modification. | | |
| 10 | Authentication and Authorisation | ODS spoofing protection | API cross-checks `NHSD-End-User-Organisation-ODS` header against the bearer token ODS claim before any resource check. | Prevents identity spoofing. | | |
| 11 | Authentication and Authorisation | Ownership by resource type | Templates/Endpoints owned by supplier (`managingOrganization`); HealthcareServices owned by provider (`providedBy`); Lists inherit from their HealthcareService. | Reflects real-world ownership split. | | |
| 12 | Authentication and Authorisation | Delegated authority | Suppliers may write to a HealthcareService if their Product ID matches one on the resource, even without matching `providedBy`. | Suppliers manage services on behalf of providers. | | |
| 13 | Authentication and Authorisation | Product ID + ODS | Product ID and ODS code together form the ownership key. | Prevents one supplier modifying another's resources within the same organisation. | | |
| 14 | Endpoint Header Attribute | Privacy rule | Template's `header` field governs address visibility in merged responses (`public` = visible to all, `private` = owner only). | Child Endpoint inherits address from Template, so Template's setting governs. | | |
| 15 | Endpoint Visibility: Status and Period | Combined visibility rule | Endpoint available only when own status is `active`, Template status is `active`, AND current time is within `period`. | Both checks must pass; more restrictive status takes precedence. | | |
| 16 | Endpoint Visibility: Status and Period | Consumer filtering | Consumers only see Endpoints passing all visibility checks; managing organisation sees all. | Consumers trust results are usable; owners need full lifecycle visibility. | | |
| 17 | Endpoint Visibility: Status and Period | Templates have no period | Templates are configuration artefacts without time-bounded validity. | They represent reusable protocol definitions, not service instances. | | |
| 18 | Endpoint Visibility: Status and Period | Visibility via `_include` | Visibility filtering MUST apply to Endpoints returned via `_include` on `GET /HealthcareService`. | Prevents inconsistency between API paths. | | |
| 19 | Filtering Endpoints by Connection Type and Payload Type | Template model | `connectionType`, `payloadType`, and `address` live on the Template; Lambda resolves the join server-side. | Single source of truth; eliminates duplication across child Endpoints. | | |
| 20 | Filtering Endpoints by Connection Type and Payload Type | `_include` cannot filter | `ConnectionType`/`PayloadType` cannot be applied as filters on `GET /HealthcareService?_include=...`. | They are Endpoint properties, not HealthcareService properties; FHIR `_include` has no filter mechanism. | | |
| 21 | Endpoint Ordering — Options Comparison (Condensed) | Recommendation | Use FHIR List resource (Option A) for PoC; priority extension (Option C) is not recommended. | Option C breaks when Endpoints are shared across multiple services (can't have per-service priority on a single resource). | | |
| 22 | Endpoint Ordering — Full Options Document | Ordering mechanism | Priority is `List.entry[]` array position — index 0 is highest; consumers MUST use List order, not Bundle order. | Bundle entry order is unreliable due to pagination and intermediary reordering. | | |
| 23 | Endpoint Ordering — Full Options Document | Auto-creation | API always creates a List when a HealthcareService is created, populated from `endpoint[]` order. | Guarantees every service always has exactly one associated List. | | |
| 24 | BaRS OAS Alignment, Authentication, and API Routing | OAS separation | Remove EPC write operations from BaRS OAS; EPC OAS is authoritative for catalog operations. | Avoids duplicate specs that risk drift. | | |
| 25 | BaRS OAS Alignment, Authentication, and API Routing | Routing | BaRS Proxy runs inside Apigee; EPC paths route to AWS backend; Proxy calls EPC internally for resolution. | Clean separation of messaging (Proxy) and catalog management (EPC). | | |
| 26 | BaRS OAS Alignment, Authentication, and API Routing | Token validation split | Apigee validates token (signature, expiry); backend enforces ODS match, RBAC, and ownership. | Separates infrastructure security from business authorisation. | | |
| 27 | Disaster Recovery | DR strategy | Single-region (eu-west-2) with PITR and on-demand backups; multi-region evaluated at production readiness if classified Gold. | Serverless provides multi-AZ resilience; multi-region only justified for Tier 1. | | |
| 28 | Disaster Recovery | PITR and deletion protection | Point-in-Time Recovery and deletion protection enabled on all DynamoDB tables from day one — non-negotiable. | Covers accidental deletes and corruption with < 5 min RPO. | | |
| 29 | Disaster Recovery | Infrastructure-as-code | Terraform is source of truth; manual console changes prohibited in production. | Ensures reproducibility and prevents drift. | | |
| 30 | Resilience and Availability | Availability target | 99.9% availability, zero planned maintenance windows, P95 ≤ 150ms. | Serverless allows zero-downtime deploys; composite AWS SLA makes 99.9% achievable. | | |
| 31 | Resilience and Availability | Provisioned concurrency | Strongly recommended for Lambda given 150ms P95 target — cold starts would breach it. | Cold starts are the primary latency risk. | | |
| 32 | Resilience and Availability | Idempotency | `POST` uses `X-Request-Id` as idempotency key (5-min window); `PATCH` uses `If-Match` (ETag). | Allows safe retries without duplicates or lost updates. | | |
| 33 | Resilience and Availability | DynamoDB capacity | On-demand (PAY_PER_REQUEST) billing — scales automatically. | Eliminates throttling from under-provisioned capacity. | | |
| 34 | Data Migration to the Endpoint Catalog | Migration sequence | Templates → Endpoints → HealthcareServices → Validation; each step depends on IDs from the previous. | Resource types have parent-child dependencies. | | |
| 35 | Data Migration to the Endpoint Catalog | Idempotent migration | No pre-check for duplicates; API returns `409` and migration logs and continues. | Makes migration naturally re-runnable. | | |
| 36 | Data Migration to the Endpoint Catalog | Static values for BaRS | All existing Templates use fixed `connectionType: hl7-fhir-rest` and `payloadType: bars`. | Existing database is exclusively BaRS endpoints. | | |
| 37 | Managing Endpoint Templates | Template identity | Templates distinguished by `environmentType: staging` (internal, never returned); `$template` action scopes operations. | Allows Templates and Endpoints to coexist as FHIR Endpoint resources. | | |
| 38 | Managing Endpoint Templates | Deletion constraint | Template cannot be deleted if it has active child Endpoints — `409 Conflict`. | Prevents orphaning active Endpoints. | | |
| 39 | Managing Endpoint Templates | Creation process | Templates created via CSV → S3 → Lambda → API pipeline by the run/maintain team. | Separates supplier onboarding from service configuration; supports bulk. | | |
| 40 | Interim Support Process — Direct AWS Access vs API Access | Recommendation | Option 3 (CSV → Apigee API calls) — no write path should bypass API authorisation and audit. | Preserves CSV workflow while ensuring full auth, validation, and audit. | | |
| 41 | DUEC Endpoints | Tiered communication | DUEC uses three connection methods in preference order: FHIR REST → ITK3 → Secure email, expressed via a List. | Explicit failover semantics for urgent care reliability. | | |
| 42 | Endpoint Templates — Pros and Cons | Template design | Child Endpoints hold only `status`, `period`, and template reference; all other fields resolved at read time from Template. | Single point of update; one Template change propagates to all children instantly. | | |
