# Open Questions and Outstanding Items

This page consolidates all unresolved questions, TBDs, and items needing decisions
from across the Endpoint Catalog documentation. Extracted on 28 May 2026.

---

| # | Source Document | Section | Question / Open Item | Owner |
|---|---|---|---|---|
| 1 | authorisation.md | Open questions | Is CIS2 available to non-NHS users (e.g. private supplier staff)? If not, what's the auth path for suppliers? | Security / CIS2 team |
| 2 | authorisation.md | Open questions | Should write operations be restricted to user-restricted mode only, or should some writes be permitted via app-restricted (for supplier automation)? | Product / Architecture |
| 3 | authorisation.md | Open questions | How are EPC-specific roles (Endpoint Admin, HS Admin, etc.) mapped to CIS2 role assertions? Custom role codes or existing CIS2 roles? | IOPS / CIS2 team |
| 4 | authorisation.md | Open questions | Is the admin UI a separate application or part of the Digital Onboarding Service? | Product |
| 5 | authorisation.md | Open questions | Should Product ID be the primary ownership key instead of (or in addition to) ODS code? | Architecture |
| 6 | authorisation.md | Authorisation (Note) | The `header: private` requirement is currently under discussion and may be struck from the specification. Decision needed on whether private endpoint address redaction will remain. | Not assigned |
| 7 | bars-oas-alignment-and-routing.md | Open Questions | Should the BaRS OAS retain `GET /Endpoint` (read-only consumer lookup) or should all EPC operations be removed? | Architecture |
| 8 | bars-oas-alignment-and-routing.md | Open Questions | What Apigee header names are used to forward token claims to the backend? (e.g. `NHSD-Client-Id`, `NHSD-User-Id`) | Platform team |
| 9 | bars-oas-alignment-and-routing.md | Open Questions | Should `/metadata` return a combined capability statement or separate ones per service? Depends on whether BaRS and EPC share a base URL. | Architecture |
| 10 | bars-oas-alignment-and-routing.md | Open Questions | How does the BaRS Proxy's internal EPC call authenticate? Service account credentials or internal Apigee policy? | Platform team |
| 11 | bars-oas-alignment-and-routing.md | Open Questions | Are the `X-Request-Id` and `X-Correlation-Id` headers sufficient, or do the EPC-specific definitions need to be added separately? | Architecture |
| 12 | create-endpoint-template.md | Step 1 — Gather the required data | Product Id format is under investigation and has not yet been confirmed. The format (e.g. `PinnaclePharmOutcomes-v2024.12.12`) is illustrative only. | Not assigned |
| 13 | data-migration.md | Overview (warning) | Document is incomplete — source database details including database name, table names, table structures, and field mappings have not yet been documented. A database schema review is needed. | Not assigned |
| 14 | data-migration.md | Step 1 — Migrate Templates | Product Id construction mechanism not yet defined. Existing database holds only supplier name, not a versioned product identifier. Must be resolved before Step 1 can be executed. | Not assigned |
| 15 | disaster-recovery.md | Open Actions | Confirm RPO/RTO targets with service owner. | Tech lead |
| 16 | disaster-recovery.md | Open Actions | Confirm service tier classification (Gold/Silver). | Product owner |
| 17 | disaster-recovery.md | Open Actions | Decide on multi-region requirement. | Architecture review |
| 18 | disaster-recovery.md | Open Actions | Schedule first quarterly DR test. | Platform team |
| 19 | disaster-recovery.md | Open Actions | Define on-call rota and escalation path. | Delivery manager |
| 20 | disaster-recovery.md | References | Review the Red Lines Backup Scenario document and confirm that RPO/RTO targets, backup retention periods, and testing cadence are compliant. | Not assigned |
| 21 | duplicate-detection.md | Endpoints (Status note) | Whether `status` should be included in the Endpoint duplicate definition has not yet been confirmed (e.g. should an `entered-in-error` Endpoint with overlapping period block creation of a new one?). | Not assigned |
| 22 | endpoint-ordering-extension-alternative.md | Open Questions | Should priority be per connection type or global across all endpoints on a service? | Architecture / IOPS |
| 23 | endpoint-ordering-extension-alternative.md | Open Questions | What is the correct extension URL namespace? (`endpoint-priority` vs `endpoint-failover-priority`) | IOPS / FHIR governance |
| 24 | endpoint-ordering-extension-alternative.md | Open Questions | Should the extension be mandatory on all Endpoints, or optional (with unranked endpoints treated as lowest priority)? | Product |
| 25 | endpoint-ordering-extension-alternative.md | Open Questions | Does DoS have requirements beyond simple numeric ranking (e.g. weighted routing, percentage-based distribution)? | DoS leads |
| 26 | endpoint-ordering-extension-alternative.md | Open Questions | Can we take this proposal to IOPS for a second opinion on the extension approach vs the List approach? | Architecture |
| 27 | endpoint-ordering-options-summary.md | Open Questions | Should priority be per connection type or global across all endpoints on a service? | IOPS / DoS Leads |
| 28 | endpoint-ordering-options-summary.md | Open Questions | What is the correct extension URL namespace? | IOPS / DoS Leads |
| 29 | endpoint-ordering-options-summary.md | Open Questions | Should the extension be mandatory or optional (unranked = lowest priority)? | IOPS / DoS Leads |
| 30 | endpoint-ordering-options-summary.md | Open Questions | Does DoS need anything beyond simple numeric ranking (weighted routing, percentage-based)? | IOPS / DoS Leads |
| 31 | endpoint-ordering-options-summary.md | Open Questions | Can we take this to IOPS for a second opinion? | IOPS / DoS Leads |
| 32 | endpoint-visibility-status-and-period.md | `_include` differences | Architecture decision needed: should `_include` on GET /HealthcareService return bare Endpoints, fully-resolved Endpoints, or not be supported at all? Affects Lambda complexity and consumer contract. | Architecture |
| 33 | endpoint-visibility-status-and-period.md | Gap 1: Template status mutability | Requirements say Template status is "fixed" but design allows status changes. Needs confirmation on whether Templates can transition between active/suspended. | Product owner |
| 34 | endpoint-visibility-status-and-period.md | Gap 2: Period requirement | Confirm whether the API should enforce `period.start` as mandatory on `POST /Endpoint` (aligning with requirements flowchart) or treat it as optional. | Product owner |
| 35 | endpoint-visibility-status-and-period.md | Gap 5: Connectivity validation | Requirements state endpoint must be live and connectable when status is set to Active (EPCFUNC-52). Not currently in the API design — noted as a UI concern. | Not assigned |
| 36 | interim-support-process-access.md | Open Questions | What identity does the batch Lambda use when calling the API? A dedicated service account with R&M team ODS code? | Architecture / Security |
| 37 | interim-support-process-access.md | Open Questions | Should the CSV include a "requested by" field so the audit trail captures which R&M team member initiated the batch? | Product |
| 38 | interim-support-process-access.md | Open Questions | What rate limit is appropriate for the batch service account? | Platform team |
| 39 | interim-support-process-access.md | Open Questions | Should failed rows be retried automatically or reported for manual review? | R&M team |
| 40 | interim-support-process-access.md | Open Questions | Is the interim process expected to run for months or years? This affects how much investment is justified. | Product |
| 41 | observability.md | Open items | Confirm ITOC log forwarding mechanism (Logs destination vs Firehose). | Platform team |
| 42 | observability.md | Open items | Confirm ODIN onboarding timeline and integration pattern. | ODIN team |
| 43 | observability.md | Open items | Confirm CSOC log forwarding requirements (format, destination, frequency). | Security team |
| 44 | observability.md | Open items | Agree custom metric naming conventions with ODIN team. | EPC + ODIN |
| 45 | observability.md | Open items | Confirm funding model for ODIN usage. | Programme |
| 46 | observability.md | Open items | Define log retention beyond 90 days (archive to S3 Glacier?). | Architecture |
| 47 | observability-odin.md | Prerequisites and open items | ODIN onboarding — register EPC as a service, obtain OTLP endpoint and auth token. | EPC team + ODIN team |
| 48 | observability-odin.md | Prerequisites and open items | Confirm ODIN supports AWS Lambda OTel layer export (OTLP/HTTP). | ODIN team |
| 49 | observability-odin.md | Prerequisites and open items | Confirm ODIN funding model for EPC (included in platform cost or per-service charge). | Programme |
| 50 | observability-odin.md | Prerequisites and open items | Confirm CSOC access pattern (shared Grafana dashboard or dedicated log export). | Security team |
| 51 | observability-odin.md | Prerequisites and open items | Confirm ITOC already uses ODIN for service health (avoids duplicate integration). | ITOC team |
| 52 | observability-odin.md | Prerequisites and open items | Agree metric naming conventions and label cardinality limits with ODIN team. | EPC + ODIN |
| 53 | observability-odin.md | Prerequisites and open items | Confirm trace sampling strategy for production volume. | EPC team |
| 54 | resilience-and-availability.md | Service Level Target | Confirm the availability target with the service owner. If classified as Gold/Tier 1, a 99.95% target may be required, necessitating multi-region deployment. | Not assigned |
| 55 | template-design-pros-cons.md | Summary (Key open question) | Whether the Lambda should automatically apply List order on `GET /HealthcareService` and `GET /Endpoint` responses, eliminating the two-step consumer pattern. | Not assigned |
| 56 | template-design-pros-cons.md | Cons (Con 2) | ProductId format is unresolved — this is a hard blocker for data migration. | Not assigned |
| 57 | duec-endpoints.md | Overview (warning) | The connectionType and payloadType codes used for DUEC have not been confirmed with the DUEC programme. Needs review and validation before implementation. | DUEC team |
