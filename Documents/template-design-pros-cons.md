# Endpoint Templates — Pros and Cons

## What this document covers

The Endpoint Catalog data model uses an **Endpoint Template** as a shared parent resource
from which individual Endpoint records are derived. This document examines the advantages
and disadvantages of this design decision, drawing on the full set of design documents
produced for the Endpoint Catalog.

For a description of what a Template is and how it works, see:
- [Filtering Endpoints by Connection Type and Payload Type](./filtering-endpoints-by-connection-and-payload-type.md)
- [Managing Endpoint Templates](./create-endpoint-template.md)

---

## What the Template design does

In the Template model, an `Endpoint` resource holds only three fields of its own — `status`,
`period`, and a reference to its parent Template. Everything else — `connectionType`,
`payloadType`, `address`, `name`, `header`, `managingOrganization` — lives on the Template
and is resolved at read time by the Lambda.

A single Template can be the parent of many Endpoints across many HealthcareServices. The
canonical use case is a multi-tenanted supplier: one product, one URL, many services. Rather
than storing the same URL, connection type, and payload type on hundreds of Endpoint records,
the Template stores it once and all child Endpoints inherit it automatically.

```
Template (PinnaclePharmOutcomes-v2024.12.12)
    address: https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk
    connectionType: hl7-fhir-rest
    payloadType: bars
    managingOrganization: R778
        │
        ├── Endpoint (DOS service 110549)  status: active, period: 2024-01-01 →
        ├── Endpoint (DOS service 149641)  status: active, period: 2024-01-01 →
        ├── Endpoint (DOS service 156812)  status: active, period: 2024-01-01 →
        └── ... (hundreds more)
```

---

## Pros

### 1. Single point of update for multi-tenanted suppliers

The most significant advantage. When a supplier changes their URL — for example, during a
product upgrade or infrastructure migration — only the Template needs updating. The change
is immediately visible on every child Endpoint without any cascade writes to individual
records. In the existing BaRS database, the same URL appears hundreds of times across
individual service records; a URL change requires updating every one of them.

The `targets.json` file illustrates the scale: many hundreds of DOS service ids map to a
small number of distinct URLs. The Template model collapses this many-to-one relationship
into a single record per supplier product.

### 2. Eliminates data duplication

Without Templates, every Endpoint record would need to store `connectionType`, `payloadType`,
`address`, `name`, `header`, and `managingOrganization` independently. For a supplier with
500 services, this means 500 copies of the same data. The Template model stores it once,
reducing storage, reducing the risk of inconsistency between records, and making the data
easier to reason about.

### 3. Clean separation of supplier configuration from service configuration

Templates are owned and managed by the supplier (via the run/maintain team). Endpoints are
associated with HealthcareServices by the service owner. These are distinct concerns — who
provides the technical capability vs who uses it — and the Template model reflects this
separation cleanly. A Template can exist before any HealthcareService uses it, and a
HealthcareService can reference an Endpoint without needing to know anything about the
supplier's infrastructure.

### 4. Supports private Endpoints cleanly

The `header: private` flag on a Template means the `address` is redacted from responses
for any caller whose ODS code does not match `managingOrganization`. This is a single flag
on one record that applies to all child Endpoints automatically. Without Templates, the
private flag would need to be set and enforced on every individual Endpoint record.

### 5. Authorisation is straightforward

Ownership of an Endpoint is inherited from its parent Template's `managingOrganization`.
The authorisation rule — the requesting ODS code must match `managingOrganization` to
perform write operations — applies uniformly to all Endpoints derived from the same
Template. There is no need to store or check ownership on each individual Endpoint record.

### 6. Duplicate detection is well-defined

A Template is a duplicate if another active Template exists with the same `productId`,
`connectionType`, and `payloadType`. This is a clear, enforceable rule that maps directly
to the business concept of "one product, one capability". The API enforces this on every
write operation, preventing accidental duplication without requiring callers to pre-check.

### 7. Filtering is transparent to consumers

Because the Lambda resolves the template join before returning responses, consumers always
see fully-populated Endpoints with all fields present. The Template is an implementation
detail — consumers do not need to know it exists. A `GET /Endpoint` request with
`ConnectionType` and `PayloadType` filters works exactly as expected, with the Lambda
handling the fan-out internally.

---

## Cons

### 1. Adds a mandatory onboarding step before services can be configured

A Template must exist before any Endpoint can reference it, and Templates are created by
the run/maintain team via a CSV-driven process — not by the service owner. This creates a
dependency: a service owner cannot configure their Endpoints until the run/maintain team
has created the appropriate Template. In a self-service model, this is a friction point.

### 2. The productId format is unresolved and is a migration blocker

The Template's identity depends on a `productId` in the format `{ProductName}-v{Version}`.
The existing database holds only the supplier name — the version component is not available.
A mechanism for constructing or obtaining the full `productId` has not yet been defined.
This is a hard blocker for the data migration: Step 1 cannot be executed until this is
resolved. See [Data Migration](./data-migration.md) for details.

### 3. The Template concept is not a standard FHIR R4 construct

Templates are implemented by setting `environmentType` to `staging` — a field back-ported
from FHIR R5. This field is internal: it is set by the EPC, never returned in responses,
and hidden behind the `$template` custom action. The result is that Templates look like
normal Endpoints to consumers but behave differently. This is a non-standard extension that
adds cognitive load for implementers and may cause confusion when the API is used with
generic FHIR tooling that does not understand the `$template` convention.

### 4. Lambda resolution adds latency on every read

Every `GET /Endpoint` request requires the Lambda to fetch the parent Template for each
returned Endpoint, apply filters, and merge the fields before responding. For a
HealthcareService with many Endpoints, this is a fan-out of N+1 lookups (one for the
Endpoints, one per Template). Without caching, this adds latency proportional to the number
of distinct Templates involved. The filtering document notes this is an implementation
detail invisible to consumers, but it is a real performance consideration for the
implementor.

### 5. Deletion is constrained by child Endpoint state

A Template cannot be deleted — hard or soft — if it has active child Endpoints. Before
decommissioning a supplier product, every child Endpoint must be deactivated or deleted
first. For a supplier with hundreds of services, this is a significant operational burden.
The run/maintain team must coordinate with service owners to deactivate all child Endpoints
before the Template can be removed. See [Managing Endpoint Templates](./create-endpoint-template.md)
for the pre-deletion checklist.

### 6. Filtering cannot be applied to GET /HealthcareService with _include

Because `connectionType` and `payloadType` live on the Template rather than the Endpoint,
they are not available at the data layer when `_include` is processed. The FHIR
specification provides no mechanism for filtering `_include` results, and applying
`ConnectionType` and `PayloadType` as filters on a `GET /HealthcareService` request
conflates two distinct resources. Consumers who want type-filtered Endpoints must use
`GET /Endpoint` instead, which may require two API calls where one might be expected.
See [Filtering Endpoints by Connection Type and Payload Type](./filtering-endpoints-by-connection-and-payload-type.md).

### 7. List ordering and type filtering require client-side work

The `List` resource orders all Endpoints for a HealthcareService regardless of type. When
a consumer wants ordered, type-filtered Endpoints, they must:
1. Fetch the List to get the priority sequence
2. Fetch the type-filtered Endpoints via `GET /Endpoint`
3. Apply the List order to the filtered results client-side

This two-step pattern is a direct consequence of the Template model — type information is
not on the Endpoint, so the List cannot be pre-filtered by type, and the server cannot
return ordered, filtered Endpoints in a single call without a non-standard extension. A
proposal to have the Lambda apply List order automatically on `GET /HealthcareService` and
`GET /Endpoint` responses exists but has not been decided. See
[Endpoint Ordering using the FHIR List Resource](./endpoint-ordering-with-list.md).

### 8. The data model is harder to explain and document

The Template model requires understanding several interacting concepts: the Template/Endpoint
parent-child relationship, the Lambda resolution mechanism, the `$template` custom action,
the `environmentType` internal field, the cascade behaviour on update, the deletion
constraints, and the authorisation inheritance. Each of these is individually
straightforward, but together they form a model that is significantly more complex than a
flat Endpoint record. This complexity is reflected in the volume of documentation required
to describe it.

---

## Summary

| | Detail |
|-|--------|
| **Primary benefit** | Single point of update for multi-tenanted suppliers — one Template change propagates to hundreds of Endpoints instantly |
| **Primary cost** | Adds a mandatory run/maintain team onboarding step; productId format is unresolved and blocks migration |
| **Best suited for** | Environments where many services share the same supplier product and URL — the multi-tenanted model |
| **Least suited for** | Environments where every service has a unique URL — the Template adds overhead without the deduplication benefit |
| **Key open question** | Whether the Lambda should automatically apply List order on `GET /HealthcareService` and `GET /Endpoint` responses, which would eliminate the two-step consumer pattern |
| **Migration risk** | ProductId construction mechanism is undefined — this must be resolved before migration can begin |

### Pros at a glance

| # | Pro |
|---|-----|
| 1 | Single point of update — URL change propagates to all child Endpoints instantly |
| 2 | Eliminates data duplication across hundreds of Endpoint records |
| 3 | Clean separation of supplier configuration from service configuration |
| 4 | Private Endpoints managed by a single flag on one record |
| 5 | Authorisation inherited from Template — no per-Endpoint ownership data needed |
| 6 | Duplicate detection is well-defined and API-enforced |
| 7 | Template resolution is transparent to consumers |

### Cons at a glance

| # | Con |
|---|-----|
| 1 | Mandatory run/maintain team onboarding step before services can be configured |
| 2 | ProductId format unresolved — hard blocker for data migration |
| 3 | Non-standard FHIR R4 construct — uses back-ported R5 field hidden behind custom action |
| 4 | Lambda N+1 lookups add read latency |
| 5 | Deletion blocked by active child Endpoints — operational burden at decommissioning |
| 6 | Type filtering cannot be applied to `GET /HealthcareService?_include` |
| 7 | Ordered, type-filtered Endpoints require a two-step consumer pattern |
| 8 | Model complexity requires significant documentation |
