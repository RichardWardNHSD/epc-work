# Endpoint Reference Validation on HealthcareService Write Operations

## Overview

When a `POST /HealthcareService` (create) or `PUT /HealthcareService/{id}` (update) request
includes one or more `endpoint[]` references, the EPC validates that **every referenced
Endpoint exists** before writing the resource. If any referenced Endpoint does not exist,
the entire request is rejected — the HealthcareService is not created or updated.

This is a server-side validation performed by the EPC API. Callers do not need to
pre-validate Endpoint references before submitting the request.

---

## Why this validation exists

Without endpoint reference validation, a HealthcareService could be created or updated to
reference an Endpoint that:

- Has been deleted
- Never existed (typo in the Endpoint ID)
- Belongs to a different organisation (authorisation check failure)

This would result in broken routing — the BaRS Proxy would attempt to resolve the
HealthcareService's Endpoint reference and find nothing, causing a runtime failure for
senders trying to reach the service.

By validating at write time, the EPC ensures that every HealthcareService in the catalogue
has valid, resolvable Endpoint references.

---

## Behaviour

### On `POST /HealthcareService` (create)

Before creating the HealthcareService, the EPC:

1. Extracts all `endpoint[].reference` values from the request body
2. For each reference, resolves the Endpoint resource by ID
3. If **all** referenced Endpoints exist → the HealthcareService is created (200 OK)
4. If **any** referenced Endpoint does not exist → the request is rejected (422)

### On `PUT /HealthcareService/{id}` (update)

Before updating the HealthcareService, the EPC:

1. Extracts all `endpoint[].reference` values from the request body
2. For each reference, resolves the Endpoint resource by ID
3. If **all** referenced Endpoints exist → the HealthcareService is updated (200 OK)
4. If **any** referenced Endpoint does not exist → the request is rejected (422)

### When `endpoint[]` is empty or omitted

If the request body contains no `endpoint[]` references (empty array or field omitted),
validation is skipped. A HealthcareService can exist without Endpoint associations.

---

## Error response

When validation fails, the EPC returns a `422 Unprocessable Entity` with a FHIR
`OperationOutcome` identifying which Endpoint reference(s) could not be resolved:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "REFERENCE_NOT_FOUND"
          }
        ]
      },
      "diagnostics": "Referenced Endpoint does not exist: Endpoint/e1a2b3c4-0000-0000-0000-999999999999",
      "expression": ["HealthcareService.endpoint[0]"]
    }
  ]
}
```

### Key fields

| Field | Purpose |
|-------|---------|
| `severity` | Always `error` — the request cannot proceed |
| `code` | `not-found` — the referenced resource does not exist |
| `details.coding[].code` | `REFERENCE_NOT_FOUND` — EPC-specific error code |
| `diagnostics` | Human-readable message identifying the missing Endpoint |
| `expression` | FHIR path to the offending reference in the request body |

### Multiple invalid references

If multiple Endpoint references are invalid, the `OperationOutcome` may contain multiple
`issue[]` entries — one per invalid reference:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "REFERENCE_NOT_FOUND"
          }
        ]
      },
      "diagnostics": "Referenced Endpoint does not exist: Endpoint/e1a2b3c4-0000-0000-0000-999999999999",
      "expression": ["HealthcareService.endpoint[0]"]
    },
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "REFERENCE_NOT_FOUND"
          }
        ]
      },
      "diagnostics": "Referenced Endpoint does not exist: Endpoint/f2b3c4d5-1111-2222-3333-888888888888",
      "expression": ["HealthcareService.endpoint[1]"]
    }
  ]
}
```

---

## Impact on the processing pipeline

Because the EPC performs this validation server-side, the processing pipeline (Lambda) does
**not** need to pre-validate Endpoint references before calling `POST` or `PUT`. Instead:

1. The pipeline constructs the HealthcareService payload from the CSV data (including any
   `EndpointId` values in the `endpoint[]` array)
2. The pipeline calls the API
3. If the API returns `422` with `REFERENCE_NOT_FOUND`, the pipeline records the row as
   `FAILED` in the processing report with the diagnostics message from the OperationOutcome
4. The pipeline continues to the next row

This simplifies the pipeline — it does not need to make separate `GET /Endpoint/{id}` calls
to validate references. The API handles it atomically as part of the write operation.

---

## What this validation does NOT check

| Check | Performed? | Notes |
|-------|-----------|-------|
| Endpoint exists | ✅ Yes | The reference must resolve to an existing Endpoint resource |
| Endpoint is `active` | ❌ No | A HealthcareService can reference a suspended or off Endpoint — visibility rules handle this at query time |
| Endpoint belongs to the same organisation | ❌ No | Cross-organisation references are permitted (a service may use a supplier's shared Endpoint) |
| Endpoint's Template is active | ❌ No | Template status is checked at query time, not at write time |
| Endpoint's period is valid | ❌ No | Period-based visibility is enforced at query time, not at write time |

The validation is purely an **existence check** — it confirms the referenced resource exists
in the catalogue. All other business rules (visibility, status, period) are applied when
consumers query the HealthcareService, not when it is written.

---

## Related documents

| Document | Description |
|----------|-------------|
| [Managing HealthcareServices](./Processes/IP001-manage-healthcare-service.md) | Full CRUD lifecycle including pipeline behaviour |
| [Endpoint Catalogue API OAS](../OAS/endpoint-catalog-api.json) | OpenAPI specification with validation described in POST/PUT operation descriptions |
