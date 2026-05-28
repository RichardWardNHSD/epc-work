# Endpoint Visibility: Status and Period

## Overview

The Endpoint Catalog uses two complementary mechanisms to control whether an Endpoint
is visible and usable by consumers: **status** and **period**. Together, these fields
determine the *effective availability* of an Endpoint at any given point in time.

- **Status** is a required, coded field indicating the operational lifecycle state of the
  resource (active, suspended, error, off, entered-in-error, test).
- **Period** is a time-bounded window (`period.start` and `period.end`) defining *when*
  the Endpoint is valid for use.

Both fields exist on the child Endpoint resource. Templates do not carry a `period` — they
are configuration artefacts not bound to a time window. Templates do carry a `status` which
supersedes child Endpoint availability (see [Status interaction](#status-interaction-between-endpoint-and-template)).

> **Note:** A Template is a specialised Endpoint. Both are represented as FHIR `Endpoint`
> resources and share the same `status` field and value set. The distinction is in their
> role: an **Endpoint** is a service-specific child resource holding `status` and `period`
> for a particular `HealthcareService`, while a **Template** is a parent resource holding
> the shared protocol characteristics (`connectionType`, `payloadType`, `address`, etc.)
> that child Endpoints inherit. The `$template` suffix in the API paths (`/Endpoint/$template`,
> `/Endpoint/{id}/$template`) is what distinguishes Template operations from Endpoint operations.

---

## Status values

FHIR R4 defines six status codes for `Endpoint.status`, drawn from the
[EndpointStatus](https://hl7.org/fhir/R4/valueset-endpoint-status.html) value set:

| Code | Display | FHIR definition |
|------|---------|-----------------|
| `active` | Active | This endpoint is expected to be active and can be used. |
| `suspended` | Suspended | This endpoint is temporarily unavailable. |
| `error` | Error | This endpoint has exceeded connectivity thresholds and is considered in an error state and should no longer be attempted until corrective action is taken. |
| `off` | Off | This endpoint is no longer to be used. |
| `entered-in-error` | Entered in error | This instance should not have been part of this patient's medical record. |
| `test` | Test | This endpoint is not intended for production usage. |

---

## Period

`Endpoint.period` is a FHIR `Period` datatype composed of two optional date/time fields:

| Field | Type | Meaning |
|-------|------|---------|
| `period.start` | `dateTime` | The date/time from which this Endpoint is valid |
| `period.end` | `dateTime` | The date/time after which this Endpoint is no longer valid |

### Period rules

- If `period.start` is set, the Endpoint is not valid until that date/time has passed.
- If `period.end` is set, the Endpoint is no longer valid once that date/time has passed.
- If neither is set, the Endpoint has no time constraint (always-valid from a time
  perspective — availability is governed by `status` alone).
- If only `period.start` is set (no end), the Endpoint is open-ended — valid from the
  start date indefinitely.

> **Design note — open-ended periods:** The "open-ended" behaviour (start set, no end)
> is not explicitly stated in the Requirements V1.8. It is inferred from two sources:
>
> 1. **FHIR R4 `Period` semantics** — both `start` and `end` are optional fields in the
>    FHIR Period datatype. The standard interpretation is that an absent `end` means the
>    period has no defined end.
> 2. **Requirements flowchart (Section 4.13, page 40)** — the check is "Has EndDate AND
>    its in past → Yes → False". If there is no end date, this check simply does not
>    fail, so the Endpoint remains valid.
>
> Additionally, EPCFUNC-14 notes that `period.end` will "typically change if either the
> Healthcare Service is to become suspended or it's switching to a new supplier" —
> implying end date is set reactively when circumstances change, not necessarily at
> creation time.

### Period on Templates vs Endpoints

| Resource | Has `period`? | Notes |
|----------|---------------|-------|
| **Template** | No | Templates are configuration artefacts. They do not have a time-bounded validity window. |
| **Endpoint** | Yes | Period defines when this specific service instance is valid for use. |

When a new Endpoint is created from a Template (EPCFUNC-03), the `status` and `period`
are the two fields assigned by the user — all other fields are inherited from the Template.

### Period and duplicate detection

Two Endpoints with the same parent Template and **overlapping periods** are considered
duplicates. The API enforces this rule and returns `409 Conflict` if an Endpoint would
overlap with an existing one. Non-overlapping periods represent valid succession — the
same capability at different points in time. An Endpoint with **no period set** is treated
as unbounded — it overlaps with everything, meaning only one Endpoint per Template can
exist without a period. See [duplicate-detection.md](./duplicate-detection.md) for full
rules including no-period and open-ended scenarios.

---

## How status and period combine to determine visibility

An Endpoint is considered **effectively available** only when ALL of the following
conditions are true:

1. The Endpoint's own `status` is `active`
2. The parent Template's `status` is `active`
3. The current date/time falls within the Endpoint's `period`:
   - If `period.start` is set, it must be in the past (or now)
   - If `period.end` is set, it must be in the future (or now)
4. If no `period` is set, the time check is skipped

This is a logical AND — failure of any single condition renders the Endpoint unavailable
to consumers.

### Decision flowchart

The following flowchart describes the full evaluation. This diverges from the Requirements
V1.8 Section 4.13 flowchart on one point: the requirements flowchart treats a missing
`period.start` as "not active", whereas this design treats a missing period as "no time
constraint" (always-valid from a time perspective). See
[Gap 2](#2-period-requirement-on-endpoint-creation) for discussion.

```
Is Endpoint Active?
─────────────────────────────────────────────────────────────────

  ┌─────────────────────┐
  │ Is Status Active?   │──── No ───► NOT AVAILABLE
  └─────────┬───────────┘
            │ Yes
            ▼
  ┌─────────────────────┐
  │ Has Period?         │──── No ───► skip time checks
  └─────────┬───────────┘                    │
            │ Yes                            │
            ▼                                │
  ┌─────────────────────┐                   │
  │ Has StartDate AND   │──── Yes ──► NOT AVAILABLE
  │ StartDate in future?│     (not yet valid)│
  └─────────┬───────────┘                   │
            │ No                             │
            ▼                                │
  ┌─────────────────────┐                   │
  │ Has EndDate AND     │──── Yes ──► NOT AVAILABLE
  │ EndDate in past?    │     (expired)      │
  └─────────┬───────────┘                   │
            │ No                             │
            ▼◄───────────────────────────────┘
  ┌─────────────────────┐
  │ Has HealthcareService│─── Yes ──► NOT AVAILABLE
  │ AND it is inactive? │     (parent service deactivated)
  └─────────┬───────────┘
            │ No
            ▼
  ┌─────────────────────────────┐
  │ Is the Template's           │── No ──► NOT AVAILABLE
  │ Organisation Active?        │   (supplier org deactivated;
  └─────────┬───────────────────┘    all templates inactive)
            │ Yes
            ▼
       ┌──────────┐
       │ AVAILABLE │
       └──────────┘
```

---

## Status meaning by resource type

### active

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | This service instance is live and accepting connections during its `period`. | Use this Endpoint — it is the correct one to connect to for this service. |
| **Template** | The template is published and available for child Endpoints to reference. The protocol and address it defines are current. | The template's `connectionType`, `payloadType` and `address` are valid and should be used when resolving a child Endpoint. |

### suspended

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | This service instance is temporarily unavailable — for example, due to planned maintenance or a temporary outage. The Endpoint is expected to return to `active`. | Do not attempt to connect. Retry after the suspension period or check for updates. The Endpoint remains in `HealthcareService.endpoint[]` but should not be used. |
| **Template** | The template is temporarily withdrawn — all child Endpoints that reference it are effectively suspended regardless of their own `status`. The protocol or address may be under review or temporarily invalid. | Treat all child Endpoints of this template as unavailable, even if the child's own `status` is `active`. |

### error

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | Repeated connection attempts to this service instance have failed beyond an acceptable threshold. The system has automatically flagged it as degraded. | Do not attempt to connect. The endpoint owner must investigate and resolve the issue before the status is reset. |
| **Template** | The template's address or configuration has been found to be in error — for example, the address is unreachable or the protocol definition is invalid. All child Endpoints are affected. | Treat all child Endpoints of this template as unavailable. Do not attempt to connect using the template's `address`. |

### off

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | This service instance has been permanently decommissioned. It will not return to `active`. | Do not use. The Endpoint may be retained in the catalog for audit purposes but should be excluded from any routing or selection logic. |
| **Template** | The template has been permanently retired. No new child Endpoints should reference it, and existing children should be migrated to a replacement template. | Treat all child Endpoints of this template as permanently unavailable. |

### entered-in-error

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | This Endpoint was created in error and should never have existed. It is not clinically or operationally valid. | Ignore entirely. Do not use, do not display, do not include in any selection logic. |
| **Template** | This template was created in error. Any child Endpoints referencing it are also invalid. | Ignore the template and all its child Endpoints entirely. |

### test

| Resource | Meaning | Consumer behaviour |
|----------|---------|-------------------|
| **Endpoint** | This Endpoint exists for testing and integration purposes only. It must not be used in production workflows. | Use only in sandbox or integration test environments. Exclude from production routing logic. |
| **Template** | This template defines a test protocol configuration. Child Endpoints referencing it are test instances. | Use only in non-production environments. |

---

## Status interaction between Endpoint and Template

Because an Endpoint inherits its protocol characteristics from its parent Template, the
**effective availability** of an Endpoint is determined by the combination of both statuses.
The more restrictive status takes precedence:

| Endpoint status | Template status | Effective availability | Notes |
|-----------------|-----------------|----------------------|-------|
| `active` | `active` | **Available** (subject to period) | Normal operating state |
| `active` | `suspended` | **Unavailable** | Template suspension overrides child active status |
| `active` | `error` | **Unavailable** | Template error overrides child active status |
| `active` | `off` | **Unavailable** | Template retired — child cannot be used |
| `active` | `entered-in-error` | **Invalid** | Template is invalid — child is also invalid |
| `active` | `test` | **Test only** | Child is a test instance regardless of its own status |
| `suspended` | `active` | **Temporarily unavailable** | Service-level suspension |
| `suspended` | `suspended` | **Unavailable** | Both suspended |
| `error` | `active` | **Error — do not connect** | Service-level error |
| `off` | `active` | **Permanently unavailable** | Service decommissioned |
| `entered-in-error` | `active` | **Invalid** | Child is invalid regardless of template |
| `test` | `active` | **Test only** | Child is a test instance |
| `test` | `test` | **Test only** | Both are test resources |

**Rule of thumb:** if either the Endpoint or its template has a status of `suspended`,
`error`, `off`, or `entered-in-error`, the Endpoint should not be used in production.

> **Important:** Even when both statuses are `active`, the Endpoint is still only available
> if the current date/time falls within its `period`. Status and period are independent
> checks that must both pass.

---

## Visibility in search results

### The combined rule — consumers see only usable Endpoints

When a consumer searches for Endpoints (e.g. `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}`),
the API applies **both status-based and period-based filtering** to ensure only genuinely
usable Endpoints are returned:

> An Endpoint SHALL only be returned in consumer search results if:
>
> 1. The Endpoint `status` is `active`
> 2. The parent Template `status` is `active`
> 3. The current date/time falls within the Endpoint's `period`:
>    - If `period.start` is set, it must be ≤ now
>    - If `period.end` is set, it must be ≥ now
> 4. If no `period` is set on the Endpoint, the period check is skipped (the Endpoint is
>    considered always-valid from a time perspective)

This means consumers can trust that every Endpoint in a search response is genuinely
available for use — they do not need to perform their own status or period filtering.

### Filtering summary

| Check | What is evaluated | Effect if failed |
|-------|-------------------|-----------------|
| Endpoint status | `status == active` | Excluded from results |
| Template status | Parent template `status == active` | Excluded from results |
| Period start | `period.start <= now` (if set) | Excluded — not yet valid |
| Period end | `period.end >= now` (if set) | Excluded — expired |

### Exception — managing organisation sees all

The one exception to visibility filtering is when the requesting organisation **is the
managing organisation** of the Endpoint (i.e. the ODS code in the
`NHSD-End-User-Organisation-ODS` header matches the `managingOrganization` on the
Endpoint's parent Template).

In this case, the API returns **all** Endpoints regardless of status or period, because:

- The managing organisation owns and administers these Endpoints
- They need visibility of suspended, error, off, test, and expired Endpoints to manage
  their lifecycle (reactivate, decommission, troubleshoot, adjust periods)
- Status and period are operational state indicators, not access control mechanisms —
  the owner must always be able to see their own resources

| Requester | Filtering applied? | What is returned |
|-----------|-------------------|------------------|
| Any organisation (consumer) | Yes — status AND period | Only Endpoints where Endpoint status is `active`, Template status is `active`, AND current time is within `period` |
| Managing organisation (owner) | No | All Endpoints regardless of status or period |

### Examples

**Consumer request (non-owner):**

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-... HTTP/1.1
NHSD-End-User-Organisation-ODS: A1001
```

Returns only Endpoints where:
- Endpoint `status` = `active`
- Parent Template `status` = `active`
- `period.start` ≤ now (if set)
- `period.end` ≥ now (if set)

Endpoints with non-active status, inactive templates, or outside their period are excluded.

**Owner request (managing organisation):**

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-... HTTP/1.1
NHSD-End-User-Organisation-ODS: R778
```

Where `R778` is the `managingOrganization` of the Template — returns **all** Endpoints
for the service, including those with `suspended`, `error`, `off`, or `test` status, and
those outside their period window.

---

## Visibility when using `_include` on GET /HealthcareService

### Two ways to retrieve Endpoints for a service

There are two API patterns for retrieving Endpoints associated with a HealthcareService.
They differ in how they work internally, but the **visibility rules must apply equally
to both**:

| Pattern | Request | What it returns |
|---------|---------|-----------------|
| **Direct Endpoint search** | `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}` | A Bundle containing only Endpoint resources (fully resolved with template fields) |
| **HealthcareService with `_include`** | `GET /HealthcareService?_id={id}&_include=HealthcareService:endpoint` | A Bundle containing the HealthcareService (mode: `match`) plus its Endpoints (mode: `include`) |

### How they differ internally

**Direct Endpoint search (`GET /Endpoint?_has:...`):**

1. The Lambda fetches all child Endpoints for the HealthcareService
2. Fetches the parent Template for each Endpoint
3. Applies visibility filtering — excludes Endpoints where status is not `active`,
   Template status is not `active`, or current time is outside `period`
4. Applies `ConnectionType`/`PayloadType` filters (if specified)
5. Merges each passing Endpoint with its Template fields (`connectionType`, `payloadType`,
   `address`, `name`, `header`)
6. Returns fully-resolved Endpoint resources in the Bundle

**HealthcareService with `_include` (`GET /HealthcareService?_id=...&_include=HealthcareService:endpoint`):**

1. The Lambda fetches the HealthcareService resource
2. Follows the references in `HealthcareService.endpoint[]` to fetch each child Endpoint
3. Fetches the parent Template for each Endpoint
4. **Applies the same visibility filtering** — excludes Endpoints where status is not
   `active`, Template status is not `active`, or current time is outside `period`
5. Merges each passing Endpoint with its Template fields
6. Returns the HealthcareService (mode: `match`) plus the filtered, fully-resolved
   Endpoints (mode: `include`) in the Bundle

### The key rule

> **Visibility filtering MUST be applied to Endpoints returned via `_include` on
> `GET /HealthcareService`, using the same rules as `GET /Endpoint` searches.**
>
> A consumer must be able to trust that every Endpoint in a response — whether returned
> directly or via `_include` — is genuinely available for use. The API must not return
> suspended, expired, or otherwise unavailable Endpoints through either path.

### What this means for the implementation

The `_include` path cannot be handled as a simple database-level join that blindly
appends all referenced Endpoints. The Lambda must intercept the `_include` processing
and apply:

1. Status check (Endpoint `status == active`)
2. Template status check (parent Template `status == active`)
3. Period check (`period.start` ≤ now, `period.end` ≥ now, if set)
4. Template resolution (merge fields from Template into the response)
5. Private address redaction (omit `address` if `header == private` and requester is
   not the owner)

Without this, `_include` would leak Endpoints that the direct search would have filtered
out — creating an inconsistency where the same Endpoint is visible via one path but not
the other.

### Differences that remain between the two patterns

Even with visibility filtering applied to both, there are legitimate differences:

| Aspect | Direct Endpoint search | HealthcareService with `_include` |
|--------|----------------------|----------------------------------|
| **Response contains** | Endpoints only | HealthcareService + Endpoints |
| **ConnectionType/PayloadType filtering** | ✅ Supported as query parameters | ❌ Not supported — all active Endpoints are included regardless of type |
| **Use case** | When you need filtered Endpoints only | When you need service metadata alongside all its active Endpoints |
| **Consumer-side filtering needed?** | No — server filters by type | Yes — consumer must filter by `connectionType`/`payloadType` if they only want specific types |

### The managing organisation exception

The managing organisation exception applies equally to both patterns. When the requester's
ODS code matches the `managingOrganization` of the Template, **all** Endpoints are
returned regardless of status or period — via both `GET /Endpoint` and
`GET /HealthcareService?_include=HealthcareService:endpoint`.

---

## Period scenarios

### Scenario 1 — Endpoint not yet active (future start date)

An Endpoint has been created with `status: active` and `period.start: 2026-07-01`. A
consumer queries on 2026-06-15. The Endpoint is **not returned** because `period.start`
is in the future — the Endpoint is not yet valid even though its status is active.

The managing organisation can see it and confirm the scheduled go-live date.

### Scenario 2 — Endpoint expired (past end date)

An Endpoint has `status: active`, `period.start: 2025-01-01`, `period.end: 2026-01-01`.
A consumer queries on 2026-03-15. The Endpoint is **not returned** because `period.end`
has passed — the Endpoint has expired.

This pattern supports planned supplier switches: the old Endpoint's period ends on the
switch date, and the new Endpoint's period starts on the same date.

### Scenario 3 — Open-ended Endpoint (no end date)

An Endpoint has `status: active`, `period.start: 2025-06-01`, no `period.end`. A consumer
queries on 2026-05-22. The Endpoint **is returned** — it is open-ended and still within
its validity window.

### Scenario 4 — Supplier switch with non-overlapping periods

Two Endpoints exist for the same HealthcareService with the same connection type but
different parent Templates (different suppliers):

| Endpoint | Template | period.start | period.end | Status |
|----------|----------|-------------|-----------|--------|
| EP-001 (old supplier) | Template-A | 2024-01-01 | 2026-06-30 | `active` |
| EP-002 (new supplier) | Template-B | 2026-07-01 | (none) | `active` |

On 2026-06-15, only EP-001 is returned. On 2026-07-01, only EP-002 is returned. The
transition is seamless — no overlap, no gap.

### Scenario 5 — Status overrides period

An Endpoint has `status: suspended`, `period.start: 2025-01-01`, no `period.end`. Even
though the current time is within the period, the Endpoint is **not returned** because
status is not `active`. Status and period are both required to pass.

---

## Status in API responses

When the Lambda resolves an Endpoint and merges it with its parent template, both the
child's `status` and `period` are available in the response. The template's `status` is
not separately surfaced — consumers who need to check the template status should call
`GET /Endpoint/{id}/$template` directly.

Example of a fully-resolved Endpoint response:

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000001",
  "status": "active",
  "period": {
    "start": "2026-01-01T00:00:00+00:00"
  },
  "name": "Anytown UTC — BaRS Booking Endpoint",
  "connectionType": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
        "code": "hl7-fhir-rest",
        "display": "HL7 FHIR"
      }
    ]
  },
  "payloadType": [
    {
      "coding": [
        {
          "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type",
          "code": "bars",
          "display": "BaRS"
        }
      ]
    }
  ],
  "address": "https://bars.provider.nhs.uk/FHIR/R4",
  "header": "public"
}
```

The `status: "active"` and `period.start` here reflect the child Endpoint's own values.
To confirm the template is also active, call:

```http
GET /Endpoint/e1a2b3c4-0000-0000-0000-000000000001/$template HTTP/1.1
```

And check `status` on the returned template resource.

---

## Lifecycle transitions

Typical status transitions for an Endpoint and Template:

```
                    ┌─────────┐
         created ──►│  test   │──► promoted to production
                    └─────────┘
                         │
                         ▼
                    ┌─────────┐
         ◄──────────│ active  │────────────►
         │          └─────────┘            │
         │               │                 │
         ▼               ▼                 ▼
    ┌──────────┐    ┌──────────┐      ┌─────────┐
    │suspended │    │  error   │      │   off   │
    └──────────┘    └──────────┘      └─────────┘
         │               │
         └───────┬────────┘
                 ▼
            ┌─────────┐
            │ active  │  (after remediation)
            └─────────┘

    At any point:
    ┌──────────────────┐
    │ entered-in-error │  (record should never have existed)
    └──────────────────┘
```

- `test` → `active`: endpoint promoted to production use
- `active` → `suspended`: planned maintenance or temporary outage
- `suspended` → `active`: maintenance complete, service restored
- `active` → `error`: automated detection of connectivity failure
- `error` → `active`: issue resolved, endpoint restored
- `active` → `off`: endpoint permanently decommissioned
- any → `entered-in-error`: record identified as incorrectly created

### Period lifecycle

Period changes are independent of status transitions:

- **Set `period.start`**: when the Endpoint is activated and linked to a HealthcareService
  (EPCFUNC-03). This defines when the Endpoint becomes valid.
- **Set `period.end`**: when the service is switching supplier or the Endpoint is being
  retired on a known date. The end date allows a clean handover.
- **Extend `period.end`**: if the service continues longer than originally planned.
- **Remove `period.end`**: to make an Endpoint open-ended (no planned expiry).

---

## Alignment with EPCSe Requirements V1.8

This section documents how the status and period design in this document aligns with (and
in some cases extends) the Endpoint Catalogue Service Requirements V1.8 (12/02/2025).

### Aligned areas

| Requirement | How this document addresses it |
|---|---|
| EPCFUNC-31: Returned Endpoints MUST have Status set as Active, and retrieval time MUST fall within Period | Covered by "Visibility in search results" — both status (`active`) and period (current time within window) are enforced as filters on consumer queries |
| EPCFUNC-32: Retrieve Endpoints of specific Connection Type with Status Active and within Period | Covered — type filtering is applied alongside status and period filtering |
| EPCFUNC-33: Retrieve Endpoints in an Agnostic Manner with Status Active and within Period | Covered — the same combined status + period rule applies regardless of query pattern |
| EPCFUNC-14: Updating Endpoint objects — status and period at once or individually | Covered — both fields are independently updatable; the "Period lifecycle" section describes when each changes |
| EPCFUNC-01: Registering a new Endpoint — Period will not be set, Status set to suspended | Covered — Templates are created with `status: active` (they are configuration artefacts); child Endpoints get `status` and `period` assigned when linked to a HealthcareService (EPCFUNC-03) |
| EPCFUNC-03: Linking a HealthcareService with Endpoints — Status and Period assigned by user | Covered — when a child Endpoint is created from a Template, the user provides `status` and `period` values |
| EPCFUNC-05: De-duplication — overlapping period with same Connection Type | Covered — duplicate detection uses period overlap as a key criterion (see [duplicate-detection.md](./duplicate-detection.md)) |
| EPCSe001 AC6: Users with manage access shall view information regardless of status | Covered by "Managing organisation exception" — owner sees all statuses and all periods |
| EPCSe001 AC3: Set status to suspended | Covered — `suspended` is a defined status with clear meaning and transition rules |
| Section 4.9.2: Status and Period as key Endpoint attributes | Covered — both are first-class visibility mechanisms documented in this design |
| Section 4.13 flowchart: "Is Endpoint Active" decision tree | Covered — the decision flowchart in this document mirrors the requirements flowchart, checking status, period.start, period.end, HealthcareService active, and template organisation active |
| Figure 3 (Requirements doc): Suspended organisation → all child endpoints suspended | Covered by "Status interaction between Endpoint and Template" — Template status supersedes child |

### Gaps and divergences

#### 1. Template status mutability

**Requirements V1.8 states:** "The status of an endpoint template is fixed i.e. cannot be
modified."

**This document states:** Templates have a full status lifecycle (active, suspended, error,
off, etc.) and Template status supersedes child Endpoint status.

**Resolution needed:** The requirements doc appears to intend that Templates are always
created with `suspended` status and remain in that state (they are configuration
artefacts, not live operational resources). However, the diagrams in the requirements doc
(Figures 2–7) show Templates with various statuses including `Suspended` labels that
imply they could be changed by the managing organisation.

**Proposed interpretation:** Template status is set by the managing organisation (supplier)
and can be changed between `active` and `suspended` to control whether child Endpoints
are usable. The "fixed" statement in the requirements likely refers to the fact that
*consumers* cannot change Template status — only the managing organisation can. This
needs confirmation with the product owner.

---

#### 2. Period requirement on Endpoint creation

**Requirements V1.8 states (EPCFUNC-01):** "the Endpoint Period will not be set" when
registering a new Endpoint without an associated Healthcare Service (i.e. creating a
Template).

**Requirements V1.8 states (EPCFUNC-03):** "For each such newly created object, the
Status and Period attributes will be assigned values provided by the user" when linking
a HealthcareService with Endpoints.

**Requirements V1.8 flowchart (Section 4.13, page 40):** Shows "Has StartDate? → No →
False" — implying `period.start` is required for an operational Endpoint to be considered
active.

**This document:** Treats a missing period as "no time constraint" — the Endpoint is
always-valid from a time perspective and availability is governed by status alone. This
is a deliberate divergence from the requirements flowchart.

**Rationale:** In practice, EPCFUNC-03 ensures that `period` is set when an Endpoint is
linked to a HealthcareService, so operational Endpoints will almost always have a period.
However, making the API reject Endpoints without a period would be overly restrictive —
it would prevent legitimate scenarios such as migrated data where period was not
historically recorded, or Endpoints that genuinely have no planned expiry and no
meaningful start constraint. The safer design is: if period is absent, skip the time
check; if period is present, enforce it strictly.

**Recommendation:** Confirm with the product owner whether the API should enforce
`period.start` as mandatory on `POST /Endpoint` (aligning with the requirements
flowchart) or treat it as optional (aligning with this design). Either way, the
visibility rule is clear: if period is present, it is enforced; if absent, it is skipped.

---

#### 3. "Private" as a status vs header field

**Requirements V1.8 states (EPCSe001 AC4):** "set the status to private"

**This document:** Uses `header: private` as a separate field controlling address redaction,
not as a status value. The FHIR `Endpoint.status` value set does not include "private" —
it is not a lifecycle state.

**Design decision:** "Private" is an access/visibility concern (who can see the address),
not a lifecycle state (is the endpoint usable). Implementing it as a `header` field rather
than a status value is the correct FHIR-conformant approach. The requirements doc's
phrasing "set status to private" should be interpreted as "mark the endpoint as private"
rather than literally setting the FHIR status field to a non-standard value.

This is documented in [authorisation.md](./authorisation.md) under the private Endpoint
address redaction rules.

---

#### 4. Additional statuses beyond the requirements

**Requirements V1.8** only references `active` and `suspended` as operational statuses.

**This document** defines six FHIR R4 statuses: `active`, `suspended`, `error`, `off`,
`entered-in-error`, and `test`.

**Rationale:** The FHIR R4 EndpointStatus value set defines all six codes. The requirements
doc focuses on the two most common operational states but does not prohibit the others.
The additional statuses (`error`, `off`, `entered-in-error`, `test`) are needed for:
- Automated error detection (`error`)
- Permanent decommissioning (`off`)
- Data quality correction (`entered-in-error`)
- Non-production environments (`test`)

These are extensions of the requirements, not contradictions.

---

#### 5. Connectivity validation before activation

**Requirements V1.8 states (EPCFUNC-52):** "The solution MUST ensure that an endpoint
is live and can be connected to when its Status is set as Active."

**This document:** Does not currently describe connectivity validation as a prerequisite
for the `suspended` → `active` transition.

**Note:** The requirements doc clarifies this is "for a UI integration and not specifically
for the api" and is "normally carried out when a new endpoint is created, or when an
existing suspended endpoint is being switched to active." This is an operational/UI
concern rather than an API-level status rule. It should be implemented in the admin UI
(future consideration) rather than enforced by the API on every status change.

---

### Cross-reference to requirements

| This document section | Requirements reference |
|---|---|
| Status values | Section 4.9.2 (Endpoint key attributes) |
| Period | Section 4.9.2 (Period — Start and End date/times defining when endpoint is available) |
| How status and period combine (visibility rule) | EPCFUNC-31, 32, 33 (Status Active AND within Period) |
| Decision flowchart | Section 4.13 (Flow for if an endpoint configuration is active) — this design diverges on the "Has StartDate" check; see Gap 2 |
| Status interaction (Template supersedes child) | Figures 2–7 (Section 3.1) |
| Visibility in search results | EPCFUNC-31, 32, 33 |
| Managing organisation exception | EPCSe001 AC6 |
| Period lifecycle | EPCFUNC-03 (Status and Period assigned during linking), EPCFUNC-14 (updating status/period) |
| Period and duplicate detection | EPCFUNC-05 (overlapping period = duplicate) |
| Private endpoint handling | EPCSe001 AC4, Section 5.0 (Private Endpoint Template) |
| Lifecycle transitions | EPCFUNC-14 (updating status/period) |
