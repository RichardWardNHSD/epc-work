# UEC Service ID Mapping in the Endpoint Catalogue

## Overview

The UEC (Urgent and Emergency Care) team is creating new service identifiers for each
service in their domain. These new IDs need to be mapped into the Endpoint Catalogue so
that UEC consumers can discover services using the new identifiers.

This paper examines how these new UEC service IDs should be represented in the EPC's
`HealthcareService` resources — whether as an additional identifier on the existing
resource or as a separate cloned resource.

---

## Context

### Current state

Today, HealthcareServices in the EPC are identified by a single service identifier
from the Directory of Services (DoS):

```json
{
  "resourceType": "HealthcareService",
  "id": "hs-001",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000072489"
    }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "RXF"
    }
  },
  "name": "Anytown UTC",
  "active": true,
  "endpoint": [
    { "reference": "Endpoint/ep-001" }
  ]
}
```

Consumers (including the BaRS Proxy) discover services by querying:

```
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000072489
```

### What the UEC team is introducing

The UEC team is assigning new service IDs to their services. These are identifiers in a
new namespace (e.g., `https://fhir.nhs.uk/Id/uec-service-id`) that identify the same
underlying services but from the UEC system's perspective.

The EPC needs to support consumers looking up services by either the existing DoS ID or
the new UEC ID — and resolving to the same Endpoints in both cases.

---

## Key Question

> Does the new UEC service ID represent the **same service** (same Endpoints, same
> provider, same capability) just with a different business identifier? Or does it
> represent a **different view** of the service that might need different Endpoints,
> metadata, or behaviour?

The answer to this question determines the correct approach.

---

## Options

### Option A: Add the UEC ID as a second identifier (recommended)

The existing HealthcareService gains an additional identifier in the UEC namespace.
One resource, multiple identifiers — both resolve to the same service and the same
Endpoints.

#### Example

```json
{
  "resourceType": "HealthcareService",
  "id": "hs-001",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000072489"
    },
    {
      "system": "https://fhir.nhs.uk/Id/uec-service-id",
      "value": "UEC-NEW-12345"
    }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "RXF"
    }
  },
  "name": "Anytown UTC",
  "active": true,
  "endpoint": [
    { "reference": "Endpoint/ep-001" }
  ]
}
```

#### Consumer queries

Both of the following resolve to the same HealthcareService and the same Endpoints:

```
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000072489
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/uec-service-id|UEC-NEW-12345
```

Then:

```
GET /Endpoint?_has:HealthcareService:endpoint:_id=hs-001
```

Returns the same Endpoints regardless of which identifier was used to find the service.

#### Pros

| # | Benefit |
|---|---|
| 1 | **Single source of truth** — one resource represents the service regardless of which system is looking it up |
| 2 | **No data duplication** — Endpoints are associated once, no sync problem |
| 3 | **FHIR-idiomatic** — the FHIR identifier model is specifically designed for "one resource, many business identifiers" |
| 4 | **No impact on existing consumers** — DoS-based lookups continue to work unchanged |
| 5 | **List (ordering) works unchanged** — one List per HealthcareService, applies to all lookups |
| 6 | **Duplicate detection unchanged** — the resource identity is the same; adding an identifier doesn't create a duplicate |
| 7 | **`HealthcareService.active` is consistent** — deactivating the service affects all consumers equally |
| 8 | **Ownership model unchanged** — `providedBy` and delegated authority (Product ID) remain on one resource |
| 9 | **Simple to implement** — a PATCH or PUT to add the second identifier |

#### Cons

| # | Drawback |
|---|---|
| 1 | If the two IDs need to resolve to **different Endpoints** (different behaviour per system), this doesn't work |
| 2 | If the UEC team needs **different metadata** (different name, category, location), a single resource can't serve both without compromise |
| 3 | The UEC team doesn't "own" their identifier independently — updates to the HS affect both ID consumers |

#### When this is the right choice

- The UEC ID identifies the **same service** that the DoS ID identifies
- Both IDs should resolve to the same Endpoints
- The service metadata (name, provider, location, active status) is the same for both consumers
- The only difference is the identifier namespace

---

### Option B: Clone the HealthcareService (separate resource per ID)

A new HealthcareService is created with the UEC ID, referencing the same Endpoints as
the original. Two resources exist for what is logically the same service.

#### Example

```json
// Original (DoS consumers)
{
  "resourceType": "HealthcareService",
  "id": "hs-001",
  "identifier": [
    { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000072489" }
  ],
  "providedBy": { "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "RXF" } },
  "name": "Anytown UTC",
  "active": true,
  "endpoint": [ { "reference": "Endpoint/ep-001" } ]
}

// Clone (UEC consumers)
{
  "resourceType": "HealthcareService",
  "id": "hs-002",
  "identifier": [
    { "system": "https://fhir.nhs.uk/Id/uec-service-id", "value": "UEC-NEW-12345" }
  ],
  "providedBy": { "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "RXF" } },
  "name": "Anytown UTC — UEC",
  "active": true,
  "endpoint": [ { "reference": "Endpoint/ep-001" } ]
}
```

#### Pros

| # | Benefit |
|---|---|
| 1 | Each system owns its HealthcareService independently with its own metadata |
| 2 | If UEC needs different Endpoints, they can diverge without affecting DoS consumers |
| 3 | If UEC needs different service metadata (name, category, type), they can set it independently |
| 4 | Separate Lists allow different Endpoint priority ordering per consumer type |
| 5 | UEC team can manage their resource independently (different `active` lifecycle, different Product ID) |

#### Cons

| # | Drawback |
|---|---|
| 1 | **Duplication and drift** — two resources pointing at the same Endpoints; metadata can diverge unintentionally |
| 2 | **Two Lists to maintain** — if ordering matters, both need to be kept in sync (or intentionally different) |
| 3 | **`HealthcareService.active` is independent** — deactivating one doesn't deactivate the other; a provider must manage both |
| 4 | **Provider confusion** — the provider (`RXF`) now has two HealthcareService resources for what they consider one service |
| 5 | **Delegated authority must be on both** — the supplier's Product ID needs to be on both resources |
| 6 | **Endpoint overlap ambiguity** — the same Endpoint appears in two HealthcareServices; changes to the Endpoint affect both |
| 7 | **Admin tooling complexity** — admin UI needs to show the relationship between clones |
| 8 | **Duplicate detection interaction** — are two HealthcareServices with the same `providedBy` and `endpoint[]` but different `identifier.system` considered duplicates? Currently no (different system = different identity), but semantically they're the same service |

#### When this is the right choice

- The UEC ID represents a **different view** of the service with different metadata
- UEC consumers need **different Endpoints** than DoS consumers
- The UEC team needs independent lifecycle control (`active`, ordering, metadata)
- The two IDs are genuinely different services that happen to share infrastructure

---

## Comparison

| Aspect | Option A: Second identifier | Option B: Clone |
|---|---|---|
| **Number of HS resources** | 1 | 2 |
| **Endpoint association** | Once | Duplicated (or referenced from both) |
| **List (ordering)** | One List, applies to all | Two Lists, independently managed |
| **`active` lifecycle** | Unified | Independent |
| **Metadata divergence** | Not possible (single resource) | Possible (different names, categories, etc.) |
| **Ownership** | Single `providedBy` + Product ID | Duplicated on both |
| **Implementation effort** | PATCH to add identifier | Create new HS + manage relationship |
| **Ongoing maintenance** | Low (one resource) | Higher (two resources, sync risk) |
| **FHIR conformance** | Idiomatic (multiple identifiers is the norm) | Valid but unconventional for same-service |

---

## Recommendation

**Adopt Option A (second identifier)** unless the UEC team confirms they need:

1. Different Endpoints for UEC consumers vs DoS consumers, OR
2. Different service metadata (name, category, type, location) that can't coexist on one resource, OR
3. Independent `active` lifecycle (service is live for UEC but not for DoS, or vice versa)

If any of these are true, Option B is justified — but should be treated as two genuinely
separate services that happen to share Endpoints, not as a "clone."

---

## Implementation (Option A)

### Adding the UEC identifier

The UEC identifier can be added to an existing HealthcareService via a standard `PUT`
operation:

```http
PUT /HealthcareService/hs-001
Content-Type: application/fhir+json
NHSD-End-User-Organisation-ODS: RXF
```

With the full resource including the new identifier in the `identifier[]` array.

### Bulk migration

If the UEC team is assigning IDs to many services at once, the run/maintain team can
process this as a CSV batch (same pattern as endpoint onboarding):

| dos-service-id | uec-service-id |
|---|---|
| 2000072489 | UEC-NEW-12345 |
| 2000073917 | UEC-NEW-12346 |
| ... | ... |

The batch Lambda would:
1. Look up each HealthcareService by `dos-service-id`
2. Add the `uec-service-id` identifier to the existing `identifier[]` array
3. PUT the updated resource back

### Impact on duplicate detection

Adding a second identifier to an existing resource does not trigger duplicate detection —
the resource identity (`id`) is unchanged. Duplicate detection only fires on `POST` (new
resource creation), not on `PUT` (update to existing).

However, the duplicate detection rule for HealthcareServices should be extended to check
across identifier systems: if a `POST` attempts to create a new HealthcareService with
an identifier that already exists on another HealthcareService (regardless of system),
a `409 Conflict` should be returned with a diagnostic indicating which existing resource
holds that identifier.

### Impact on consumer queries

No change to the API or query patterns. The existing `identifier` search parameter
already supports system-qualified queries:

```
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/uec-service-id|UEC-NEW-12345
```

This works out of the box because identifier search matches any identifier in the array.

---

## Decision required

| ID | Decision | By when | Owner |
|---|---|---|---|
| UEC-1 | Confirm whether UEC service IDs represent the same service (Option A) or a different view (Option B) | TBD | UEC team / Architecture |
| UEC-2 | If Option A: confirm the identifier system URI for UEC service IDs (e.g., `https://fhir.nhs.uk/Id/uec-service-id`) | TBD | UEC team / IOPS |
| UEC-3 | If Option A: confirm the batch migration mechanism (CSV via run/maintain team, or UEC team self-service) | TBD | EPC team / UEC team |
| UEC-4 | If Option B: confirm who owns the cloned HealthcareService (`providedBy` and Product ID) | TBD | Architecture |

---

## Open questions

| # | Question | For review by |
|---|---|---|
| 1 | Does the UEC team need different Endpoints for services looked up via the UEC ID vs the DoS ID? | UEC team |
| 2 | Does the UEC team need different service metadata (name, category, type) on their version? | UEC team |
| 3 | Does the UEC team need independent `active` lifecycle control? | UEC team |
| 4 | What is the identifier system URI for the new UEC service IDs? | UEC team / IOPS |
| 5 | How many services need the UEC ID mapped? (Determines batch vs manual approach) | UEC team |
| 6 | Is there a timeline for the UEC ID assignment — do existing DoS IDs continue to work indefinitely, or is there a sunset? | UEC team |
