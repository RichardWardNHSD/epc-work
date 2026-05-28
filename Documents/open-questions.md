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
