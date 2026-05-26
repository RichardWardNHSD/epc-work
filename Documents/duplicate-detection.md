# Duplicate Detection in the Endpoint Catalog

## Overview

The Endpoint Catalog data model has four resource types that can be subject to duplication:
**Templates**, **Endpoints**, **HealthcareServices**, and **Lists**. Each has a different
definition of what constitutes a duplicate, because each resource has a different identity
and purpose.

**All duplicate rules defined in this document are enforced by the Endpoint Catalog API.**
The implementor of the API is responsible for applying these checks server-side on every
write operation (`POST`, `PUT`, `PATCH`). Callers do not need to perform their own duplicate
checks before submitting a request — if a duplicate is detected, the API will reject the
request with a `409 Conflict` response. The detection queries described in each section are
provided for informational purposes and for tooling that wishes to pre-validate before
calling the API.

This document defines:
- What is and is not a duplicate for each resource type
- The rules the API enforces on write operations
- The API responses returned when a duplicate is detected

---

## Templates

### What is a duplicate Template?

A Template is a duplicate if another **active** Template already exists with the same
combination of:

- `productId` (`identifier[].value` where system is `https://fhir.nhs.uk/id/product-id`)
- `connectionType` (`connectionType.coding[].code`)
- `payloadType` (`payloadType[].coding[].code`)

All three fields must match for a duplicate to exist. A Template with the same `productId`
but a different `connectionType` or `payloadType` is **not** a duplicate — it represents a
different technical capability of the same product.

### Why this definition?

A Template represents a specific technical capability of a supplier product — the combination
of *what product it is* (`productId`), *how you connect to it* (`connectionType`), and *what
message standard it uses* (`payloadType`). Two active Templates with the same values for all
three fields would be indistinguishable and would create ambiguity when child Endpoints are
created from them.

### Does status affect the duplicate definition?

`status` **is** part of the duplicate definition. Only an `active` Template with the same
`productId`, `connectionType`, and `payloadType` constitutes a duplicate. A Template with
`status` of `entered-in-error` or `off` has been withdrawn and a new Template can
legitimately be created to replace it.

| Existing Template status | New Template has same productId + connectionType + payloadType | Duplicate? | API enforces? |
|--------------------------|---------------------------------------------------------------|------------|---------------|
| `active` | Yes | ✅ Yes | ✅ Yes — `409 Conflict` |
| `entered-in-error` | Yes | ❌ No | ❌ No — creation permitted |
| `off` | Yes | ❌ No | ❌ No — creation permitted |

### Duplicate vs not duplicate — examples

| Scenario | productId | connectionType | payloadType | Duplicate? |
|----------|-----------|---------------|-------------|------------|
| Identical active Template already exists | `ProductA-v1.0` | `hl7-fhir-rest` | `bars` | ✅ Yes |
| Same product, different connectionType | `ProductA-v1.0` | `hl7-fhir-msg` | `bars` | ❌ No — different capability |
| Same product, different payloadType | `ProductA-v1.0` | `hl7-fhir-rest` | `bars-referral` | ❌ No — different capability |
| Same connectionType and payloadType, different product | `ProductB-v1.0` | `hl7-fhir-rest` | `bars` | ❌ No — different product |
| New version of same product | `ProductA-v2.0` | `hl7-fhir-rest` | `bars` | ❌ No — different productId |

### API enforcement

The API checks for an active duplicate on every write operation that could introduce a
conflicting Template:

| Operation | Duplicate check applied? | Scenario that could create a duplicate |
|-----------|--------------------------|----------------------------------------|
| `POST /Endpoint/$template` | ✅ Yes | Creating a new Template with the same identity as an existing active one |
| `PUT /Endpoint/{id}/$template` | ✅ Yes | Updating a Template's `productId`, `connectionType`, or `payloadType` to match another active Template |

`PATCH` is not applicable to Templates — Templates are replaced in full via `PUT`.

If a duplicate is detected the request is rejected — see [API responses for duplicates](#api-responses-for-duplicates).

### How to detect a duplicate Template (pre-validation)

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars&status=active HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

| Response | Meaning |
|----------|---------|
| `total: 0` | No active duplicate — creation will be permitted |
| `total: 1` | Active duplicate exists — `POST` will be rejected with `409 Conflict` |
| `total: > 1` | Data integrity issue — investigate |

---

## Endpoints

### What is a duplicate Endpoint?

An Endpoint is a duplicate if another Endpoint already exists that:

- References the **same parent Template**, **and**
- Has an **overlapping `period`** with the new Endpoint

An Endpoint can exist independently without being allocated to a HealthcareService — the
duplicate check applies to the Endpoint resource itself, not to its association with any
service. Two Endpoints with the same Template and an overlapping period are
indistinguishable regardless of which HealthcareServices they are assigned to.

Two Endpoints with the same parent Template but **non-overlapping periods** are **not**
duplicates — they represent the same capability at different points in time, which is a
valid succession pattern (e.g. when a supplier renews or replaces an Endpoint).

> ⚠️ **Status — further consideration needed:** Whether `status` should be included in the
> duplicate definition has not yet been confirmed. For example, it is not yet decided whether
> an `entered-in-error` or `off` Endpoint with an overlapping period should block creation of
> a new Endpoint, or whether only `active` Endpoints should be considered. This document will
> be updated once this is agreed.

### Why this definition?

An Endpoint represents a specific use of a Template during a specific time window. The
combination of *which template* and *when* defines the identity. Two Endpoints with the same
Template active at the same time are indistinguishable.

### Period overlap — examples

| Scenario | Template | Period A | Period B | Duplicate? |
|----------|----------|----------|----------|------------|
| Same Template, identical periods | `Template-X` | 2026-01-01 → 2026-12-31 | 2026-01-01 → 2026-12-31 | ✅ Yes |
| Same Template, overlapping periods | `Template-X` | 2026-01-01 → 2026-12-31 | 2026-06-01 → 2027-06-30 | ✅ Yes |
| Same Template, open-ended period overlaps new | `Template-X` | 2026-01-01 → (no end) | 2026-06-01 → 2026-12-31 | ✅ Yes |
| Same Template, non-overlapping periods | `Template-X` | 2025-01-01 → 2025-12-31 | 2026-01-01 → 2026-12-31 | ❌ No — successive periods |
| Different Template, overlapping periods | `Template-Y` | 2026-01-01 → 2026-12-31 | 2026-01-01 → 2026-12-31 | ❌ No — different capability |

### API enforcement

The API checks for a period overlap against existing Endpoints with the same parent Template
on every write operation that could introduce a conflicting Endpoint:

| Operation | Duplicate check applied? | Scenario that could create a duplicate |
|-----------|--------------------------|----------------------------------------|
| `POST /Endpoint` | ✅ Yes | Creating a new Endpoint whose period overlaps an existing one with the same Template |
| `PUT /Endpoint/{id}` | ✅ Yes | Updating an Endpoint's `period` or `template` reference such that it now overlaps another Endpoint |
| `PATCH /Endpoint/{id}` | ✅ Yes | Patching `period.start`, `period.end`, or the `template` reference such that an overlap is introduced |

If a duplicate is detected the request is rejected — see [API responses for duplicates](#api-responses-for-duplicates).

### How to detect a duplicate Endpoint (pre-validation)

Retrieve all existing Endpoints that share the same Template (identified by `connectionType`
and `payloadType`) and check whether any have a `period` that overlaps with the intended
period of the new Endpoint.

```http
GET /Endpoint?ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

Two periods overlap if either has no end date (open-ended), or the start of one falls before
the end of the other.

| Response | Meaning |
|----------|---------|
| `total: 0` | No Endpoints with this Template — creation will be permitted |
| `total: 1+`, no period overlap | No duplicate — creation will be permitted |
| `total: 1+`, period overlaps | Duplicate — `POST` will be rejected with `409 Conflict` |

---

### HealthcareService endpoint array constraint

Although an Endpoint can exist without being assigned to a HealthcareService, once it is
assigned the `HealthcareService.endpoint[]` array must not contain the same Endpoint
reference more than once. The API enforces this on `POST /HealthcareService` and
`PUT /HealthcareService/{id}` — a repeated reference in `endpoint[]` will be rejected with
`422 Unprocessable Entity`.

| Scenario | Valid? |
|----------|--------|
| `endpoint[]` contains `Endpoint/abc` once | ✅ Valid |
| `endpoint[]` contains `Endpoint/abc` twice | ❌ Invalid — `422` returned |
| `endpoint[]` contains `Endpoint/abc` and `Endpoint/xyz` | ✅ Valid — two distinct references |

---

## HealthcareServices

### What is a duplicate HealthcareService?

A HealthcareService is a duplicate if another HealthcareService already exists with the
same `identifier.system` **and** `identifier.value` combination. The system and value
together form the identity — the same value under a different system is a different
identifier and is not a duplicate.

### Duplicate vs not duplicate — examples

| Scenario | identifier.system | identifier.value | Duplicate? |
|----------|------------------|-----------------|------------|
| Same system, same value | `https://fhir.nhs.uk/Id/dos-service-id` | `111111111` | ✅ Yes |
| Different system, same value | `https://fhir.nhs.uk/Id/ods-organization-code` | `111111111` | ❌ No — different identifier system |
| Same system, different value | `https://fhir.nhs.uk/Id/dos-service-id` | `222222222` | ❌ No — different service |
| Different system, different value | `https://fhir.nhs.uk/Id/ods-organization-code` | `222222222` | ❌ No |

### API enforcement

The API checks for a matching `identifier.system` + `identifier.value` on every write
operation that could introduce a conflicting HealthcareService:

| Operation | Duplicate check applied? | Scenario that could create a duplicate |
|-----------|--------------------------|----------------------------------------|
| `POST /HealthcareService` | ✅ Yes | Creating a new HealthcareService with an identifier that already exists |
| `PUT /HealthcareService/{id}` | ✅ Yes | Replacing a HealthcareService with an identifier that matches a different existing resource |
| `PATCH /HealthcareService/{id}` | ✅ Yes | Patching `identifier` such that the updated value matches another existing HealthcareService |

If a duplicate is detected the request is rejected — see [API responses for duplicates](#api-responses-for-duplicates).

### How to detect a duplicate HealthcareService (pre-validation)

```http
GET /HealthcareService?HealthcareService.identifier=https://fhir.nhs.uk/Id/dos-service-id|111111111 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

| Response | Meaning |
|----------|---------|
| `total: 0` | No duplicate — creation will be permitted |
| `total: 1` | Duplicate exists — `POST` will be rejected with `409 Conflict` |
| `total: > 1` | Data integrity issue — investigate |

---

## Lists

### What is a duplicate List?

A List is a duplicate if another `current` List already exists with the same `subject`
reference (i.e. the same HealthcareService). Each HealthcareService should have exactly one
`current` List at any time.

A `retired` or `entered-in-error` List for the same HealthcareService is **not** a
duplicate — historical Lists are expected and valid.

### Duplicate vs not duplicate — examples

| Scenario | subject | status | Duplicate? |
|----------|---------|--------|------------|
| Second `current` List for same service | `HealthcareService/9f2c6f12-...` | `current` | ✅ Yes |
| `retired` List for same service | `HealthcareService/9f2c6f12-...` | `retired` | ❌ No — historical record |
| `current` List for different service | `HealthcareService/a1b2c3d4-...` | `current` | ❌ No — different service |

### API enforcement

The API checks for an existing `current` List with the same `subject` on every write
operation that could introduce a conflicting List:

| Operation | Duplicate check applied? | Scenario that could create a duplicate |
|-----------|--------------------------|----------------------------------------|
| `POST /List` | ✅ Yes | Creating a new `current` List for a HealthcareService that already has one |
| `PUT /List/{id}` | ✅ Yes | Updating a List's `subject` or `status` such that a second `current` List would exist for the same HealthcareService |
| `PATCH /List/{id}` | ✅ Yes | Patching `subject` or `status` such that a conflict is introduced |

If a duplicate is detected the request is rejected — see [API responses for duplicates](#api-responses-for-duplicates).

> **Note:** In normal operation a `current` List is created automatically when a
> HealthcareService is created via `POST /HealthcareService`. A direct `POST /List` for a
> service that already has a `current` List will therefore always be rejected. Use
> `PUT /List/{id}` to update the existing List instead.

### How to detect a duplicate List (pre-validation)

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

| Response | Meaning |
|----------|---------|
| `total: 0` | No current List — creation will be permitted |
| `total: 1` | Current List exists — `POST` will be rejected with `409 Conflict`; use `PUT /List/{id}` instead |
| `total: > 1` | Data integrity issue — investigate |

---

## API responses for duplicates

When the API detects a duplicate on a write operation it returns one of the following
responses. All error responses use the FHIR `OperationOutcome` resource.

### 409 Conflict — duplicate resource

Returned when a `POST` would create a resource that duplicates an existing one. Applies to
Templates, Endpoints, HealthcareServices, and Lists.

```http
HTTP/1.1 409 Conflict
Content-Type: application/fhir+json
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "duplicate",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "PROXY_CONFLICT"
          }
        ]
      },
      "diagnostics": "A duplicate resource already exists. See details below."
    }
  ]
}
```

The `diagnostics` field will contain a resource-specific message identifying the conflict.
Examples by resource type:

| Resource | Example diagnostics message |
|----------|-----------------------------|
| Template | `An active Template already exists with productId 'PinnaclePharmOutcomes-v2024.12.12', connectionType 'hl7-fhir-rest', and payloadType 'bars'.` |
| Endpoint | `An Endpoint already exists with the same parent Template and an overlapping period (existing period: 2026-01-01 to open-ended).` |
| HealthcareService | `A HealthcareService already exists with identifier system 'https://fhir.nhs.uk/Id/dos-service-id' and value '111111111'.` |
| List | `A current List already exists for HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc. Use PUT /List/{id} to update the existing List.` |

### 422 Unprocessable Entity — constraint violation

Returned when the request payload is structurally valid but violates a data constraint.
Applies to the `HealthcareService.endpoint[]` repeated reference rule.

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/fhir+json
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "invariant",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "SEND_UNPROCESSABLE_ENTITY"
          }
        ]
      },
      "diagnostics": "HealthcareService.endpoint[] contains a duplicate reference: 'Endpoint/e1a2b3c4-0000-0000-0000-000000000001' appears more than once."
    }
  ]
}
```

### Caller actions on receiving a duplicate response

| HTTP status | Resource | Recommended action |
|-------------|----------|--------------------|
| `409` | Template | Retrieve the existing Template via `GET /Endpoint/$template?productId=...`; if the `PUT` caused the conflict, revise the fields being changed so they no longer match another active Template |
| `409` | Endpoint | Retrieve the conflicting Endpoint; adjust the `period` or `template` reference on the new or updated Endpoint so there is no overlap |
| `409` | HealthcareService | Retrieve the existing HealthcareService via `GET /HealthcareService?HealthcareService.identifier=...\|...`; update it with `PUT /HealthcareService/{id}` rather than creating a new one |
| `409` | List | Retrieve the existing List via `GET /List?subject=...&status=current`; update it with `PUT /List/{id}` |
| `422` | HealthcareService | Remove the repeated reference from `endpoint[]` and resubmit |

---

## Summary

| Resource | Duplicate condition | Enforced on | Response |
|----------|--------------------|-----------|---------| 
| Template | Active Template with same `productId` + `connectionType` + `payloadType` | `POST /Endpoint/$template`, `PUT /Endpoint/{id}/$template` | `409 Conflict` |
| Endpoint | Endpoint with same parent Template + overlapping `period` | `POST /Endpoint`, `PUT /Endpoint/{id}`, `PATCH /Endpoint/{id}` | `409 Conflict` |
| HealthcareService | Same `identifier.system` + `identifier.value` | `POST /HealthcareService`, `PUT /HealthcareService/{id}`, `PATCH /HealthcareService/{id}` | `409 Conflict` |
| HealthcareService `endpoint[]` | Repeated Endpoint reference in `endpoint[]` | `POST /HealthcareService`, `PUT /HealthcareService/{id}`, `PATCH /HealthcareService/{id}` | `422 Unprocessable Entity` |
| List | Second `current` List for the same `subject` HealthcareService | `POST /List`, `PUT /List/{id}`, `PATCH /List/{id}` | `409 Conflict` |
