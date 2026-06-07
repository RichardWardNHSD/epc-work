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

The UEC team is assigning IDs to many services at once. The update will be applied as a
batch operation by the run/maintain team, following the same CSV-driven pattern used for
endpoint onboarding and other bulk data operations.

#### Input format

The UEC team provides a CSV mapping file containing the existing DoS service ID and the
corresponding new UEC service ID:

```csv
dos-service-id,uec-service-id
2000072489,UEC-NEW-12345
2000073917,UEC-NEW-12346
2000081230,UEC-NEW-12347
2000093816,UEC-NEW-12348
...
```

#### Process

The batch follows the established pattern: CSV → S3 → Lambda → API.

```
┌──────────────┐     ┌─────────┐     ┌──────────────┐     ┌─────────────┐
│  UEC team    │────▶│   S3    │────▶│ Batch Lambda │────▶│  EPC API    │
│  provides    │     │  bucket │     │              │     │  (via       │
│  CSV         │     └─────────┘     │  For each    │     │   Apigee)   │
└──────────────┘                     │  row:        │     └─────────────┘
                                     │  1. GET HS   │
                                     │  2. Add ID   │
                                     │  3. PUT HS   │
                                     └──────────────┘
```

**Step by step:**

1. **UEC team provides the CSV** — delivered via agreed channel (email, shared S3 bucket,
   or service desk ticket) to the run/maintain team.

2. **Run/maintain team uploads to S3** — the CSV is placed in the designated batch input
   bucket. This triggers the batch Lambda (or is triggered manually).

3. **Batch Lambda processes each row:**

   For each `dos-service-id` → `uec-service-id` pair:

   a. **GET** the existing HealthcareService by identifier:
      ```
      GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|{dos-service-id}
      ```

   b. **Validate** the response — confirm exactly one HealthcareService is returned.
      If zero results: log an error, skip the row, continue.
      If multiple results: log an error, skip the row, continue.

   c. **Check** whether the UEC identifier already exists on the resource (idempotency).
      If the identifier is already present with the correct value: log as "already applied",
      skip the row, continue.

   d. **Add** the UEC identifier to the `identifier[]` array:
      ```json
      {
        "system": "https://fhir.nhs.uk/Id/uec-service-id",
        "value": "UEC-NEW-12345"
      }
      ```

   e. **PUT** the updated HealthcareService back:
      ```
      PUT /HealthcareService/{id}
      NHSD-End-User-Organisation-ODS: {providedBy ODS code}
      ```

   f. **Log** the outcome: success, already-applied, not-found, error.

4. **Output report** — The Lambda produces a results CSV/log indicating the outcome for
   each row:

   ```csv
   dos-service-id,uec-service-id,status,detail
   2000072489,UEC-NEW-12345,success,identifier added
   2000073917,UEC-NEW-12346,success,identifier added
   2000081230,UEC-NEW-12347,already-applied,identifier already present
   2000093816,UEC-NEW-12348,not-found,no HealthcareService found for DoS ID
   ```

5. **Run/maintain team reviews** — any failures are investigated and re-run or resolved
   manually.

#### Authorisation

The batch Lambda authenticates using the run/maintain team's service account
(application-restricted, signed JWT). The `NHSD-End-User-Organisation-ODS` header must
match the `providedBy` ODS code on each HealthcareService being updated.

For services provided by different organisations, the Lambda will need either:
- A **System Admin** token (can write to any resource regardless of ODS), or
- To submit each update with the correct `providedBy` ODS code in the header, matching
  the service owner — which requires the service account to have cross-organisation write
  permissions, or the batch to be scoped to one provider at a time.

**Recommended approach:** Use a System Admin service account for the batch, with full
audit trail capturing that the update was performed by the run/maintain team on behalf
of the UEC programme.

#### Idempotency

The batch is designed to be **re-runnable**:
- If a row has already been applied (UEC identifier already present), it is skipped
- If a row fails mid-batch, the batch can be re-run from the start — already-applied rows
  are no-ops
- The PUT operation itself is idempotent (same content, same outcome)

#### Error handling

| Scenario | Behaviour |
|---|---|
| DoS ID not found in EPC | Log error, skip row, continue batch |
| Multiple HealthcareServices found for same DoS ID | Log error, skip row, continue (shouldn't happen — duplicate detection prevents this) |
| UEC ID already present on the resource | Log "already applied", skip row, continue |
| PUT returns 403 (auth failure) | Log error, skip row, continue — review auth setup |
| PUT returns 409 (conflict) | Log error, skip row, continue — concurrent modification |
| PUT returns 5xx (server error) | Log error, mark for retry |
| UEC ID already exists on a *different* HealthcareService | Log error, skip row — this is a data quality issue in the input CSV |

#### Volume and timing

| Aspect | Detail |
|---|---|
| Expected volume | TBD — depends on how many services the UEC team is assigning IDs to |
| Batch frequency | One-off initial load + incremental updates as new services are assigned |
| Rate limiting | The batch Lambda should respect API rate limits; include a small delay between requests if volume is high |
| Environment | Apply to INT first for validation, then to PROD |
| Rollback | If the batch needs to be undone, a reverse batch can strip the UEC identifier from each affected HealthcareService |

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
