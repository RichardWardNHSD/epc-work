# Endpoint Ordering — Full Options Document

This document presents three approaches to expressing endpoint priority/ordering in the
Endpoint Catalog API. All three solve the same core problem: giving consumers an explicit,
resilient mechanism to determine which Endpoint to try first when a HealthcareService has
multiple Endpoints.

**The problem:** Neither `Endpoint` nor `HealthcareService` provides a native FHIR mechanism
for expressing relative priority. Bundle entry order is not reliable (pagination, intermediary
reordering, no FHIR guarantee of preservation). Explicit metadata is required.

---

| Option | Mechanism | Status |
|--------|-----------|--------|
| **Option A** | Separate FHIR `List` resource with ordered `entry[]` array | ✅ Adopted |
| **Option B** | API auto-applies List order in responses (implicit List — server does the heavy lifting) | Future requirement (builds on Option A) |
| **Option C** | Custom `endpoint-priority` extension on the Endpoint resource | ❌ Not adopted (see limitations) |

---

## Table of Contents



---

# Option A: FHIR List Resource (Explicit Consumer-Side Ordering)

## Why a List resource is needed

The DUEC (Directory of Urgent and Emergency Care) requirement is that a service owner must be
able to express a **communication preference** — a preferred order in which a sender should
attempt to contact the Endpoints associated with a HealthcareService.

Neither `Endpoint` nor `HealthcareService` provide a mechanism for this:

- `HealthcareService.endpoint[]` holds an unordered array of Endpoint references. The FHIR R4
  specification does not define any ordering semantics for this array, and the server is free to
  return entries in any sequence.
- `Endpoint` has no field for expressing relative priority or preference against other Endpoints
  on the same service.

FHIR R4 provides the [`List`](https://hl7.org/fhir/R4/list.html) resource specifically for
curated, ordered collections:

> "A list is a curated collection of resources... All lists are considered ordered — the order in
> which items literally appear in the list may be an important part of the meaning of the list."

The `List` resource is therefore the correct mechanism for expressing Endpoint communication
preference in the Endpoint Catalog API.

---

## Design

### Structure

Each `List` resource represents the **priority-ordered set of Endpoints for a specific
HealthcareService**:

| Field | Value | Purpose |
|-------|-------|---------|
| `subject` | Reference to the `HealthcareService` | Ties the list to a service |
| `entry[].item` | Reference to an `Endpoint` | Each entry is one endpoint |
| `entry[]` array position | Index 0 = highest priority | Defines the communication preference order |
| `orderedBy` | `priority` | Declares the ordering semantics |
| `mode` | `working` | The list is maintained over time |
| `status` | `current` | The list is active |

Priority is determined entirely by **array position**. Index 0 is the highest priority Endpoint.
Consumers MUST use `List.entry[]` array order as the authoritative sequence — not the order of
resources in a Bundle response.

### Relationship diagram

```
HealthcareService (id: 9f2c6f12-1a6d-4d9c-a111-123456789abc)
    │
    └── referenced by List.subject
            │
            ├── entry[0].item → Endpoint/e1a2b3c4-...001  ← highest priority (try first)
            ├── entry[1].item → Endpoint/e1a2b3c4-...002
            └── entry[2].item → Endpoint/e1a2b3c4-...003  ← lowest priority (fallback)
```

`HealthcareService.endpoint[]` continues to hold the full unordered set of Endpoint references.
The `List` sits alongside this, providing the curated ordered view. No changes are required to
existing `Endpoint` or `HealthcareService` resources.

### Constraint: One List per HealthcareService

A `List` can only be associated with **one** HealthcareService (via `List.subject`). This is
a 1:1 relationship — each HealthcareService has at most one `current` List.

**On `POST /List` (create):**

If a `current` List already exists for the HealthcareService referenced in `List.subject`,
the API rejects the request with `409 Conflict`:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "conflict",
    "details": {
      "coding": [{
        "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
        "code": "LIST_ALREADY_EXISTS"
      }]
    },
    "diagnostics": "A current List already exists for HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc. Update the existing List or retire it before creating a new one."
  }]
}
```

**On `PUT /List/{id}` (update):**

The `List.subject` reference cannot be changed on an update. If the `List.subject` in the
PUT payload differs from the existing List's subject, the API rejects the request with
`422 Unprocessable Entity`:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "business-rule",
    "details": {
      "coding": [{
        "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
        "code": "SUBJECT_IMMUTABLE"
      }]
    },
    "diagnostics": "List.subject cannot be changed. The List is permanently bound to HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc."
  }]
}
```

To move priority ordering to a different HealthcareService, retire the existing List
(set `status` to `retired`) and create a new one for the target service.

### Reference validation on write

On both `POST /List` and `PUT /List/{id}`, the EPC validates that all references in the
payload point to resources that **exist** in the catalogue:

- **`List.subject`** — the referenced HealthcareService must exist
- **`List.entry[].item`** — every referenced Endpoint must exist

If any reference cannot be resolved, the request is rejected with `422 Unprocessable Entity`
and the List is not created or updated.

**Error response — HealthcareService does not exist:**

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "not-found",
    "details": {
      "coding": [{
        "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
        "code": "REFERENCE_NOT_FOUND"
      }]
    },
    "diagnostics": "Referenced HealthcareService does not exist: HealthcareService/9f2c6f12-1a6d-4d9c-a111-999999999999",
    "expression": ["List.subject"]
  }]
}
```

**Error response — Endpoint does not exist:**

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "not-found",
    "details": {
      "coding": [{
        "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
        "code": "REFERENCE_NOT_FOUND"
      }]
    },
    "diagnostics": "Referenced Endpoint does not exist: Endpoint/e1a2b3c4-0000-0000-0000-999999999999",
    "expression": ["List.entry[0].item"]
  }]
}
```

> **Note:** This is a pure existence check — it confirms the referenced resources exist.
> It does not check Endpoint status, period validity, or whether the Endpoint is associated
> with the HealthcareService. Those are query-time concerns, not write-time constraints.

---

## API operations

The following operations are available on the Endpoint Catalog API
(`https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4`):

| Operation | Path | Description |
|-----------|------|-------------|
| `POST /List` | [`POST /List`](#post-list--create-a-priority-list) | Create a new priority-ordered Endpoint list for a HealthcareService |
| `GET /List` | [`GET /List`](#get-list--search-for-a-list) | Search for lists by HealthcareService, with optional `_include` of Endpoints |
| `GET /List/{id}` | [`GET /List/{id}`](#get-listid--read-a-specific-list) | Read a specific list by its resource id |
| `PUT /List/{id}` | [`PUT /List/{id}`](#put-listid--update-the-order) | Replace the list — used to reorder, add, or remove Endpoints |
| `DELETE /List/{id}` | [`DELETE /List/{id}`](#delete-listid--remove-a-list) | Delete a list |

---

## POST /List — Create a priority list

A service owner creates a `List` that defines the priority order of Endpoints for their service.
The `subject` references the HealthcareService; each `entry[].item` references an Endpoint.
Array position is the priority — index 0 is tried first.

### Request

```http
POST /List HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 60E0B220-8136-4CA5-AE46-1D97EF59D068
X-Correlation-Id: 11C46F5F-CDEF-4865-94B2-0EE0EDCC26DA
NHSD-End-User-Organisation-ODS: A1001
```

```json
{
  "resourceType": "List",
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "code": {
    "coding": [
      {
        "system": "https://fhir.nhs.uk/CodeSystem/EPC-list-code",
        "code": "endpoint-priority",
        "display": "Endpoint Priority Order"
      }
    ]
  },
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "date": "2026-05-06T10:00:00+00:00",
  "orderedBy": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/list-order",
        "code": "priority",
        "display": "Sorted by Priority"
      }
    ]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
  ]
}
```

### Response — 201 Created

The server assigns a resource `id` and returns the created `List`.

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
  "meta": {
    "lastUpdated": "2026-05-06T10:00:01+00:00",
    "profile": [
      "https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList"
    ]
  },
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "code": {
    "coding": [
      {
        "system": "https://fhir.nhs.uk/CodeSystem/EPC-list-code",
        "code": "endpoint-priority",
        "display": "Endpoint Priority Order"
      }
    ]
  },
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "date": "2026-05-06T10:00:00+00:00",
  "orderedBy": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/list-order",
        "code": "priority",
        "display": "Sorted by Priority"
      }
    ]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
  ]
}
```

---

## GET /List — Search for a list

Search for `List` resources by HealthcareService. Use `_include=List:item` to return the
referenced `Endpoint` resources in the same Bundle, avoiding a second request.

### Request — basic search

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 7A3F1B90-1234-5678-ABCD-1D97EF59D068
X-Correlation-Id: 22D57G6G-EFAB-5976-C3DE-2E08FG60E179
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK

```json
{
  "resourceType": "Bundle",
  "id": "a1b2c3d4-0000-0000-0000-000000000010",
  "type": "searchset",
  "total": 1,
  "timestamp": "2026-05-06T10:05:00+00:00",
  "link": [
    {
      "relation": "self",
      "url": "https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4/List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current"
    }
  ],
  "entry": [
    {
      "resource": {
        "resourceType": "List",
        "id": "f7d3e2a1-0000-0000-0000-000000000099",
        "status": "current",
        "mode": "working",
        "title": "Anytown UTC — Endpoint Priority Order",
        "subject": {
          "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
          "display": "Anytown UTC"
        },
        "orderedBy": {
          "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/list-order", "code": "priority", "display": "Sorted by Priority" }]
        },
        "entry": [
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
        ]
      },
      "search": { "mode": "match" }
    }
  ]
}
```

The consumer reads `List.entry[]` in array order — index 0 is the highest priority Endpoint.

### Request — with `_include` to fetch Endpoints in one call

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current&_include=List:item HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 9C5D2E11-AAAA-BBBB-CCCC-1D97EF59D068
X-Correlation-Id: 33E68H7H-GHIJ-6087-D4EF-3F19GH71F280
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK (with included Endpoints)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "timestamp": "2026-05-06T10:05:00+00:00",
  "entry": [
    {
      "resource": {
        "resourceType": "List",
        "id": "f7d3e2a1-0000-0000-0000-000000000099",
        "status": "current",
        "mode": "working",
        "title": "Anytown UTC — Endpoint Priority Order",
        "subject": { "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc", "display": "Anytown UTC" },
        "orderedBy": {
          "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/list-order", "code": "priority", "display": "Sorted by Priority" }]
        },
        "entry": [
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
        ]
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "name": "Primary Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://provider.example.nhs.uk/FHIR/R4"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Secondary Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://provider2.example.nhs.uk/FHIR/R4"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000003",
        "status": "active",
        "name": "Fallback Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://provider3.example.nhs.uk/FHIR/R4"
      },
      "search": { "mode": "include" }
    }
  ]
}
```

> **Important:** The FHIR spec does not require servers to return `_include` resources in any
> particular order within the Bundle. Consumers MUST use `List.entry[]` array position as the
> source of truth for ordering, not the position of Endpoint resources in the Bundle.

---

## GET /List/{id} — Read a specific list

Read a known `List` resource directly by its logical id.

### Request

```http
GET /List/f7d3e2a1-0000-0000-0000-000000000099 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 4B2C9D33-1111-2222-3333-1D97EF59D068
X-Correlation-Id: 55F80J9J-KLMN-8209-F6GH-5H31IJ93H402
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK

Returns the `List` resource directly (not wrapped in a Bundle).

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
  "meta": {
    "lastUpdated": "2026-05-06T10:00:01+00:00",
    "profile": ["https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList"]
  },
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "orderedBy": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/list-order", "code": "priority", "display": "Sorted by Priority" }]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
  ]
}
```

---

## PUT /List/{id} — Update the order

To change the priority order, add or remove Endpoints, the service owner issues a `PUT` with the
full replacement `List`. The entire resource is replaced — partial updates are not supported.

In this example, the Fallback Endpoint (`...003`) is promoted to highest priority.

### Request

```http
PUT /List/f7d3e2a1-0000-0000-0000-000000000099 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 9C5D2E11-AAAA-BBBB-CCCC-1D97EF59D068
X-Correlation-Id: 33E68H7H-GHIJ-6087-D4EF-3F19GH71F280
NHSD-End-User-Organisation-ODS: A1001
```

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "orderedBy": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/list-order", "code": "priority", "display": "Sorted by Priority" }]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint (now primary)" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint (now secondary)" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint (now fallback)" } }
  ]
}
```

Endpoint `...003` is now at index 0 — it is the new highest priority.

### Response — 200 OK

Returns the updated `List` resource.

---

## DELETE /List/{id} — Remove a list

Removes the priority ordering for a HealthcareService. After deletion, consumers fall back to
the unordered results from `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}`.

### Request

```http
DELETE /List/f7d3e2a1-0000-0000-0000-000000000099 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 1A2B3C4D-5E6F-7A8B-9C0D-1D97EF59D068
X-Correlation-Id: 44F79I8I-HIJK-7198-E5FG-4G20HI82G391
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK

---

## Consumer patterns

### When no list exists

If a service has no priority list, `GET /List?subject=HealthcareService/{id}` returns an empty
Bundle (`total: 0`). Consumers should fall back to the unordered results from
`GET /Endpoint?_has:HealthcareService:endpoint:_id={id}`.

### Two-call pattern (Endpoints + HealthcareService metadata)

When the consumer needs both the ordered Endpoints and the HealthcareService metadata:

1. `GET /List?subject=HealthcareService/{id}&status=current` — get the ordered Endpoint references
2. `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}` — get all active Endpoints for the service

Apply the order client-side by iterating `List.entry[]` and resolving each reference against the
Endpoints returned in step 2. Any Endpoint in step 2 that does not appear in the List is appended
after the ordered entries.

### Single-call pattern (Endpoints only)

When the consumer only needs the Endpoints (not the HealthcareService metadata):

```
GET /List?subject=HealthcareService/{id}&status=current&_include=List:item
```

The response Bundle contains the `List` (mode: `match`) and all referenced `Endpoint` resources
(mode: `include`). Apply order by iterating `List.entry[]`.

### Decision guide

| Consumer need | Recommended pattern | Calls |
|---------------|---------------------|-------|
| Ordered Endpoints + HealthcareService metadata | `GET /List?subject=...` then `GET /Endpoint?_has:HealthcareService:endpoint:_id=...`, order client-side. Fetch `GET /HealthcareService/{id}` separately if service metadata is needed. | 2–3 |
| Ordered Endpoints only | `GET /List?subject=...&_include=List:item`, order by `List.entry[]` | 1 |
| Unordered Endpoints only | `GET /Endpoint?_has:HealthcareService:endpoint:_id=...` | 1 |
| Check if a custom order exists | `GET /List?subject=...` — if `total: 0`, fall back to unordered search | 1 |

---

## Relationship to existing search patterns

The `List` resource complements rather than replaces the existing ADR-114 search patterns:

| Pattern | Returns | Ordered? | Use when |
|---------|---------|----------|----------|
| `GET /Endpoint?_has:HealthcareService:endpoint:_id=` | Endpoints only | No | You only need endpoint details (unordered) |
| `GET /List?subject=HealthcareService/{id}&_include=List:item` | List + Endpoints | **Yes** | You need endpoints in a defined priority order |

No changes are required to the existing `/Endpoint` or `/HealthcareService` paths.
The `List` resource is purely additive.

---

# Option B: Implicit List — API Does the Heavy Lifting

Option B uses the **same List resource as Option A** for storage, but the API automatically
applies the List order when returning Endpoint results — the consumer never needs to know
the List exists or make a separate call to retrieve it.

The List becomes an internal implementation detail. The API does the heavy lifting: it
resolves the List, applies the priority order, and returns Endpoints pre-sorted in every
response. From the consumer's perspective, Endpoints simply arrive in the correct priority
order.

This option builds on the List lifecycle management described below (auto-creation,
auto-synchronisation) and adds server-side ordering as the final piece.

---

## Automatic List creation on POST /HealthcareService

### Behaviour

When a `POST /HealthcareService` request is received, the API **always** creates a `List`
resource for the new service in the same transaction — no separate `POST /List` call is
required, and no conditional logic is needed by the caller.

If the request payload includes `endpoint` references, the auto-created List is populated with
those references in the order they appear in `HealthcareService.endpoint[]`. The first reference
becomes index 0 (highest priority), the second becomes index 1, and so on.

If the payload contains no `endpoint` references, an empty List is still created. This
guarantees that every HealthcareService always has exactly one associated List, which
simplifies subsequent `PUT` and `PATCH` operations — the API can always locate the List to
update without needing to check whether one exists first.

### What the API creates automatically

Given a `POST /HealthcareService` payload with three Endpoint references:

```
HealthcareService.endpoint[0] → Endpoint/aaa...  ← becomes List entry[0] (highest priority)
HealthcareService.endpoint[1] → Endpoint/bbb...  ← becomes List entry[1]
HealthcareService.endpoint[2] → Endpoint/ccc...  ← becomes List entry[2] (lowest priority)
```

The API always creates:
1. The `HealthcareService` resource
2. A `List` resource with `subject` referencing the new HealthcareService and `entry[]`
   containing the Endpoint references in payload order (empty if no endpoints were supplied)

> **Note:** The auto-created List can be updated at any time using `PUT /List/{id}` to change
> the priority order, add new Endpoints, or remove existing ones. Changes to the HealthcareService
> via `PUT` or `PATCH` also automatically synchronise the List — see
> [Automatic List synchronisation on PUT and PATCH](#automatic-list-synchronisation-on-put-and-patch).

---

### Example: POST /HealthcareService with three Endpoints

The service owner creates a new HealthcareService for Anytown UTC, specifying three Endpoints
in priority order. The API creates both the HealthcareService and a corresponding List.

#### Request

```http
POST /HealthcareService HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: A1B2C3D4-1111-2222-3333-4D97EF59D068
X-Correlation-Id: B2C3D4E5-2222-3333-4444-5E08FG60E179
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "HealthcareService",
  "active": true,
  "name": "Anytown UTC",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000099999"
    }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "R778"
    }
  },
  "endpoint": [
    {
      "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001",
      "display": "Primary Endpoint"
    },
    {
      "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002",
      "display": "Secondary Endpoint"
    },
    {
      "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003",
      "display": "Fallback Endpoint"
    }
  ]
}
```

The `endpoint[]` array order expresses the desired priority:
- `...001` — try first
- `...002` — try second
- `...003` — fallback

#### Response — 200 OK

The API returns the created HealthcareService. The auto-created List is not included in this
response but is immediately available via `GET /List?subject=HealthcareService/{id}`.

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-05-06T10:00:00+00:00",
    "profile": [
      "http://hl7.org/fhir/StructureDefinition/HealthcareService",
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"
    ]
  },
  "active": true,
  "name": "Anytown UTC",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000099999"
    }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "R778"
    }
  },
  "endpoint": [
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" },
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" },
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" }
  ]
}
```

#### Auto-created List

The API creates the following List in the background. It can be retrieved immediately:

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current
```

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
  "meta": {
    "lastUpdated": "2026-05-06T10:00:00+00:00",
    "profile": [
      "https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList"
    ]
  },
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "code": {
    "coding": [
      {
        "system": "https://fhir.nhs.uk/CodeSystem/EPC-list-code",
        "code": "endpoint-priority",
        "display": "Endpoint Priority Order"
      }
    ]
  },
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "date": "2026-05-06T10:00:00+00:00",
  "orderedBy": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/list-order",
        "code": "priority",
        "display": "Sorted by Priority"
      }
    ]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "Secondary Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint" } }
  ]
}
```

The `entry[]` array mirrors the `endpoint[]` array from the original `POST /HealthcareService`
payload — the order is preserved exactly.

### Example: POST /HealthcareService with no Endpoints

If the HealthcareService is created before any Endpoints exist, the API still creates a List
with an empty `entry[]`. The List is ready to be populated when Endpoints are added later via
`PUT /HealthcareService/{id}` or `PATCH /HealthcareService/{id}`.

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000100",
  "meta": {
    "lastUpdated": "2026-05-06T10:00:00+00:00",
    "profile": ["https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList"]
  },
  "status": "current",
  "mode": "working",
  "title": "Anytown UTC — Endpoint Priority Order",
  "code": {
    "coding": [{
      "system": "https://fhir.nhs.uk/CodeSystem/EPC-list-code",
      "code": "endpoint-priority",
      "display": "Endpoint Priority Order"
    }]
  },
  "subject": {
    "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc",
    "display": "Anytown UTC"
  },
  "date": "2026-05-06T10:00:00+00:00",
  "orderedBy": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/list-order", "code": "priority", "display": "Sorted by Priority" }]
  },
  "entry": []
}
```

---

## Ordering, connectionType and payloadType

> **`GET /List` does not support `connectionType` or `payloadType` filtering.** This is because:
>
> 1. The `List` resource is a pure ordering mechanism — it holds references to Endpoints
>    by ID, with no knowledge of what type each Endpoint is.
> 2. `connectionType` and `payloadType` are properties of the Endpoint (resolved from its
>    parent Template at read time). They are not properties of the List or its entries.
> 3. FHIR `_include` has no filter mechanism — when `_include=List:item` is used, all
>    referenced Endpoints are returned regardless of their type.
>
> Consumers needing both ordering and type filtering must use the two-step pattern
> described below.

### The core tension

The `List` resource expresses a single priority order across **all** Endpoints for a
HealthcareService, regardless of their `connectionType` or `payloadType`. A real service,
however, will typically have Endpoints of different types — for example, a BaRS booking
Endpoint, a BaRS referral Endpoint, and a legacy IHE XDS Endpoint. A sender that only
supports `hl7-fhir-rest / bars` has no use for the IHE XDS Endpoint and should not be
routing to it at all.

This creates a question: **when a consumer retrieves the ordered List and then filters by
`connectionType` and `payloadType`, does the priority order still hold?**

The answer is yes — with one important caveat about how the filtering is applied.

---

### How connectionType and payloadType are stored

As described in [Filtering Endpoints by Connection Type and Payload Type](./filtering-endpoints-by-connection-and-payload-type.md),
`connectionType` and `payloadType` are not stored on the `Endpoint` resource itself. They
live on the parent **Template** Endpoint. The Lambda resolves the template join before
returning any response, so from the consumer's perspective these fields are always present
on the returned Endpoint — but they are not available on the bare `Endpoint` record in the
data store.

This matters for List ordering because:

- The `List.entry[]` array holds references to `Endpoint` resources by id
- The List has no knowledge of `connectionType` or `payloadType` — it is a pure ordering
  of Endpoint references
- Filtering by type must happen **after** the List is retrieved, either server-side (via
  `GET /Endpoint` with filters) or client-side (by inspecting the resolved Endpoints)

---

### The recommended pattern: List ordering + server-side type filtering

The cleanest approach is a two-step process:

**Step 1** — Retrieve the ordered List to get the priority sequence:

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: A1B2C3D4-1111-2222-3333-4D97EF59D068
X-Correlation-Id: B2C3D4E5-2222-3333-4444-5E08FG60E179
NHSD-End-User-Organisation-ODS: A1001
```

Response (abbreviated):

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [{
    "resource": {
      "resourceType": "List",
      "id": "f7d3e2a1-0000-0000-0000-000000000099",
      "entry": [
        { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" } },
        { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002" } },
        { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003" } }
      ]
    }
  }]
}
```

**Step 2** — Retrieve only the Endpoints matching the required `connectionType` and
`payloadType`, using the Lambda-resolved filter:

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: C3D4E5F6-3333-4444-5555-6F19GH71F280
X-Correlation-Id: D4E5F6G7-4444-5555-6666-7G20HI82G391
NHSD-End-User-Organisation-ODS: A1001
```

**Client-side**: apply the List order to the filtered results by iterating `List.entry[]`
and retaining only those references that appear in the Step 2 response. Entries in the
List that were filtered out are simply skipped — the relative order of the remaining
Endpoints is preserved.

---

### Worked example: mixed-type service

#### Setup

Anytown UTC has three Endpoints with the following types:

| Endpoint | connectionType | payloadType | List position |
|----------|---------------|-------------|---------------|
| `...001` | `hl7-fhir-rest` | `bars` | 0 (highest priority) |
| `...002` | `hl7-fhir-rest` | `bars-referral` | 1 |
| `...003` | `ihe-xds` | `xds-referral` | 2 (lowest priority) |

A sender supports `hl7-fhir-rest / bars` only. It needs the highest-priority Endpoint
that matches that type.

#### Step 1 — Get the ordered List

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current HTTP/1.1
```

Response:

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
  "subject": { "reference": "HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc" },
  "orderedBy": { "coding": [{ "code": "priority", "display": "Sorted by Priority" }] },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003" } }
  ]
}
```

Priority sequence extracted from `List.entry[]`:
1. `...001`
2. `...002`
3. `...003`

#### Step 2 — Filter by connectionType and payloadType

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
```

The Lambda resolves all three templates, applies the filter, and returns only `...001`:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [{
    "resource": {
      "resourceType": "Endpoint",
      "id": "e1a2b3c4-0000-0000-0000-000000000001",
      "status": "active",
      "name": "BaRS FHIR REST Endpoint",
      "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
      "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
      "address": "https://bars.provider.nhs.uk/FHIR/R4",
      "header": "public"
    },
    "search": { "mode": "match" }
  }]
}
```

#### Client-side resolution

Walk `List.entry[]` in order and retain only Endpoints present in the Step 2 response:

```
List position 0 → Endpoint/...001  ✓ matches filter  → use this one (priority 1)
List position 1 → Endpoint/...002  ✗ filtered out    → skip
List position 2 → Endpoint/...003  ✗ filtered out    → skip
```

Result: `Endpoint/...001` is the highest-priority Endpoint for this sender's capability.

---

### Why the List cannot be pre-filtered by type

It might seem convenient to store separate Lists per `connectionType`/`payloadType`
combination — one List for `bars`, another for `bars-referral`, and so on. This approach
has significant drawbacks:

- **Maintenance burden**: every time an Endpoint is added, removed, or reordered, multiple
  Lists must be updated in sync. A single `PUT /List/{id}` becomes several.
- **Consistency risk**: Lists can drift out of sync with each other, producing contradictory
  priority orders for the same service.
- **Type information is on the Template, not the Endpoint**: the List stores Endpoint
  references, and Endpoints do not carry `connectionType` or `payloadType` directly. A
  List cannot self-describe which types its entries represent without resolving each
  Template — which is the Lambda's job, not the List's.

The single-List-per-service model is simpler and more maintainable. Type filtering is a
read-time concern, handled by the Lambda on `GET /Endpoint`.

---

### Ordering within a type group

When a service has multiple Endpoints of the **same** `connectionType` and `payloadType`,
the List order is the definitive priority between them. For example:

| Endpoint | connectionType | payloadType | List position |
|----------|---------------|-------------|---------------|
| `...001` | `hl7-fhir-rest` | `bars` | 0 |
| `...002` | `hl7-fhir-rest` | `bars` | 1 |
| `...003` | `hl7-fhir-rest` | `bars` | 2 |

All three match `hl7-fhir-rest / bars`. After filtering, the consumer applies the List
order and gets:

```
Priority 1 → Endpoint/...001  (try first)
Priority 2 → Endpoint/...002  (try if ...001 fails)
Priority 3 → Endpoint/...003  (last resort)
```

This is the primary use case for the List resource in a DUEC context — expressing a
preference between functionally equivalent Endpoints that serve the same protocol and
payload type.

---

### Summary

| Concern | Mechanism |
|---------|-----------|
| Which Endpoints are relevant to this sender? | `GET /Endpoint` with `ConnectionType` and `PayloadType` filters — Lambda resolves Templates server-side |
| In what order should this sender try them? | `GET /List?subject=HealthcareService/{id}` — array position is the priority |
| Combined: ordered, type-filtered Endpoints | Step 1: get List; Step 2: get filtered Endpoints; client applies List order to filtered results |
| Multiple Endpoints of the same type | List order is the tiebreaker — index 0 is always tried first |
| Type filtering on the List itself | Not supported — the List is type-agnostic by design |

---

## Automatic List synchronisation on PUT and PATCH

### Behaviour

When a `PUT /HealthcareService/{id}` or `PATCH /HealthcareService/{id}` request modifies the
`endpoint[]` array of an existing HealthcareService, the API automatically synchronises the
associated `List` in the same transaction. Because a List is always created at `POST` time,
the API can always locate it without any conditional logic.

The synchronisation rules differ between `PUT` and `PATCH` to reflect the semantics of each
operation.

---

### PUT — full replacement

`PUT` replaces the entire HealthcareService resource. The associated List is rebuilt from
scratch using the `endpoint[]` array in the PUT payload, in the order those references appear.
Any priority ordering previously set — whether by the auto-create logic or by a subsequent
`PUT /List/{id}` — is **replaced**.

The rationale is that a `PUT` is an authoritative statement of the complete resource state.
If the caller supplies a new `endpoint[]` array, that array represents the new intended state,
including the intended priority order.

#### Synchronisation logic for PUT

```
1. Replace the HealthcareService resource with the PUT payload
2. Locate the associated List via List.subject = HealthcareService/{id}
3. Rebuild List.entry[] from HealthcareService.endpoint[] in payload order
4. Update List.date to the current timestamp
5. Persist both resources atomically
```

#### Example: PUT reorders and removes an Endpoint

**Before PUT** — the List has three entries in this order:

```
entry[0] → Endpoint/...001  (primary)
entry[1] → Endpoint/...002  (secondary)
entry[2] → Endpoint/...003  (fallback)
```

**PUT request** — the payload supplies only two Endpoints, in a different order:

```http
PUT /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: C3D4E5F6-3333-4444-5555-6F19GH71F280
X-Correlation-Id: D4E5F6G7-4444-5555-6666-7G20HI82G391
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "active": true,
  "name": "Anytown UTC",
  "identifier": [
    { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "R778"
    }
  },
  "endpoint": [
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Fallback Endpoint (now primary)" },
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "Primary Endpoint (now secondary)" }
  ]
}
```

**After PUT** — the List is rebuilt from the payload. `...002` is gone; `...003` is now first:

```
entry[0] → Endpoint/...003  (new primary)
entry[1] → Endpoint/...001  (new secondary)
```

The List `date` is updated to the time of the PUT. The response returns the updated
HealthcareService only — the List update is silent and can be confirmed via
`GET /List?subject=HealthcareService/{id}`.

---

### PATCH — partial update

`PATCH` modifies specific fields of the HealthcareService without replacing the whole resource.
The synchronisation behaviour depends on whether the patch touches `endpoint[]`:

- **If the patch does not modify `endpoint[]`** — the List is not touched. The existing
  priority order is preserved exactly as it was.
- **If the patch adds one or more Endpoint references** — the new references are **appended**
  to the end of the existing List. Existing entries retain their current positions. The new
  Endpoints are given the lowest priority by default, on the assumption that a newly added
  Endpoint has not yet been assigned a deliberate position. The service owner can reorder
  using `PUT /List/{id}` afterwards.
- **If the patch removes one or more Endpoint references** — the corresponding entries are
  removed from the List. The remaining entries retain their relative order and are re-indexed
  from 0.
- **If the patch both adds and removes** — removals are processed first, then additions are
  appended.

#### Synchronisation logic for PATCH

```
1. Apply the patch to the HealthcareService resource
2. Compute the diff: added_endpoints = new endpoint[] − old endpoint[]
                     removed_endpoints = old endpoint[] − new endpoint[]
3. Locate the associated List via List.subject = HealthcareService/{id}
4. Remove entries for removed_endpoints from List.entry[]
5. Append entries for added_endpoints to the end of List.entry[]
6. Update List.date to the current timestamp
7. Persist both resources atomically
```

#### Example: PATCH adds a new Endpoint

**Before PATCH** — the List has two entries:

```
entry[0] → Endpoint/...001  (primary)
entry[1] → Endpoint/...002  (secondary)
```

**PATCH request** — adds a third Endpoint:

```http
PATCH /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/json-patch+json
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: E5F6G7H8-5555-6666-7777-8H31IJ93H402
X-Correlation-Id: F6G7H8I9-6666-7777-8888-9I42JK04I513
NHSD-End-User-Organisation-ODS: R778
```

```json
[
  {
    "op": "add",
    "path": "/endpoint/-",
    "value": {
      "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003",
      "display": "New Fallback Endpoint"
    }
  }
]
```

**After PATCH** — the new Endpoint is appended to the end of the List:

```
entry[0] → Endpoint/...001  (primary — unchanged)
entry[1] → Endpoint/...002  (secondary — unchanged)
entry[2] → Endpoint/...003  (new — lowest priority by default)
```

The service owner can promote `...003` to a higher position at any time using
`PUT /List/{id}`.

---

#### Example: PATCH removes an Endpoint

**Before PATCH** — the List has three entries:

```
entry[0] → Endpoint/...001  (primary)
entry[1] → Endpoint/...002  (secondary)
entry[2] → Endpoint/...003  (fallback)
```

**PATCH request** — removes the secondary Endpoint:

```json
[
  {
    "op": "remove",
    "path": "/endpoint/1"
  }
]
```

**After PATCH** — `...002` is removed; the remaining entries are re-indexed:

```
entry[0] → Endpoint/...001  (primary — unchanged)
entry[1] → Endpoint/...003  (promoted from index 2 to index 1)
```

The relative priority of `...001` and `...003` is preserved.

---

### Relationship between automatic sync and manual List management

The automatic synchronisation is designed to keep the List consistent with the
HealthcareService without requiring the caller to manage both resources explicitly. However,
the service owner retains full control via `PUT /List/{id}`:

| Operation | List effect |
|-----------|-------------|
| `POST /HealthcareService` | List always created; `entry[]` populated from `endpoint[]` in payload order (empty if no endpoints) |
| `PUT /HealthcareService/{id}` | List rebuilt from scratch using `endpoint[]` in PUT payload order |
| `PATCH /HealthcareService/{id}` — no `endpoint[]` change | List unchanged |
| `PATCH /HealthcareService/{id}` — endpoints added | New endpoints appended to end of List |
| `PATCH /HealthcareService/{id}` — endpoints removed | Corresponding entries removed; remaining order preserved |
| `PUT /List/{id}` | Manual override — sets the priority order explicitly; not affected by subsequent HealthcareService reads |
| `PUT /HealthcareService/{id}` after `PUT /List/{id}` | List is rebuilt from the PUT payload, overwriting any manual ordering |

> **Important:** A `PUT /HealthcareService/{id}` always rebuilds the List from the payload,
> even if the `endpoint[]` array is unchanged. If the service owner has manually reordered the
> List via `PUT /List/{id}` and then issues a `PUT /HealthcareService/{id}`, the manual ordering
> will be overwritten. To preserve a custom order, use `PATCH /HealthcareService/{id}` for
> changes that do not affect the endpoint set, or re-apply the desired order via `PUT /List/{id}`
> after the PUT.

---

## Automatically ordered responses from GET /HealthcareService

### The idea

Currently, `GET /HealthcareService?_id=...&_include=HealthcareService:endpoint` returns a
Bundle containing the HealthcareService and its associated Endpoints, but the Endpoints are
returned in server-determined order — the FHIR specification provides no mechanism for a
search operation to return `_include` results in a List-defined sequence.

An alternative approach would be for the API to **automatically apply the List order** when
responding to `GET /HealthcareService` requests that include Endpoints. Rather than
requiring the consumer to make a separate `GET /List` call and apply the order client-side,
the server would do this transparently.

### How it could work

When `GET /HealthcareService?_id={id}&_include=HealthcareService:endpoint` is received, the
Lambda would:

1. Retrieve the HealthcareService
2. Look up the associated `current` List for that HealthcareService
3. Retrieve and resolve all included Endpoints
4. Return the Endpoints in the Bundle **in the order defined by `List.entry[]`**, rather
   than in server-determined order

The consumer would receive a single response with Endpoints already in priority order,
without needing to know about the List resource at all.

### What this would change

| Aspect | Current behaviour | Proposed behaviour |
|--------|------------------|-------------------|
| Endpoint order in Bundle | Server-determined | List-defined priority order |
| Consumer responsibility | Fetch List separately; apply order client-side | None — order is applied server-side |
| API calls required | 2 (GET /HealthcareService + GET /List) | 1 |
| Visibility of List | Consumer must know to call GET /List | Transparent — consumer sees ordered Endpoints without knowing a List exists |
| Fallback if no List exists | N/A | Return Endpoints in server-determined order (existing behaviour) |

### Considerations

- **FHIR conformance:** The FHIR R4 specification does not define a mechanism for ordering
  `_include` results. Applying List order server-side would be a non-standard extension of
  the `_include` behaviour. This should be clearly documented as a custom behaviour of this
  implementation.

- **Consistency with GET /Endpoint:** `GET /Endpoint?_has:HealthcareService:endpoint:_id=...`
  would also benefit from the same treatment — returning Endpoints in List order rather than
  server-determined order. Both endpoints should behave consistently.

- **Transparency:** If the server silently reorders Endpoints, consumers who are unaware of
  the List resource may not realise the order is meaningful. This could be addressed by
  including a `List` resource in the Bundle response (as an additional `include` entry) so
  the ordering rationale is visible.

- **Performance:** The Lambda would need to perform an additional lookup for the List on
  every `GET /HealthcareService` request that includes Endpoints. For services with no List,
  this adds a lookup that returns nothing. Caching the List lookup could mitigate this.

- **No List defined:** If no `current` List exists for the HealthcareService, the server
  should fall back to returning Endpoints in server-determined order, preserving existing
  behaviour.

### Relationship to GET /Endpoint

The same principle applies to `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}`.
Currently this returns Endpoints in server-determined order. If the server automatically
applied the List order here too, a consumer could retrieve ordered, type-filtered Endpoints
in a single call:

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
```

The Lambda would:
1. Retrieve the matching Endpoints (filtered by type)
2. Look up the associated List
3. Return the Endpoints in List order, skipping any List entries that were filtered out

This would collapse the current two-step pattern (GET /List + GET /Endpoint) into a single
call while preserving the priority ordering — the consumer gets type-filtered Endpoints in
the correct priority sequence without any client-side ordering logic.

### Summary

| Scenario | Current calls | With auto-ordering |
|----------|--------------|-------------------|
| Ordered Endpoints + HealthcareService metadata | 2 | 1 |
| Ordered, type-filtered Endpoints only | 2 | 1 |
| Consumer needs to know about List | Yes | No — ordering is transparent |
| FHIR standard compliance | Fully compliant | Custom extension — must be documented |

---
---

# Option C: Priority Extension on Endpoint Resource

> **❌ Not adopted.** This option was evaluated but not selected due to fundamental
> architectural limitations (Endpoints shared across multiple HealthcareServices cannot
> carry per-service priority values). Retained below for reference only.

## Context

This document presents an alternative approach to endpoint ordering in the Endpoint Catalog.
It should be read alongside [Option A: FHIR List Resource](./endpoint-ordering-with-list.md)
which uses a separate `List` resource to express priority order.

This alternative was raised during review of Option A, based on the following concerns:

---

## Concerns with Option A (List-based ordering)

### 1. Implicit ordering via Bundle position is fragile

The current DoS (Directory of Services) model implies endpoint priority through the order of
Endpoint objects in a Bundle response. This is problematic because:

- Pagination can reorder results across pages
- Intermediaries (caches, CDNs, API gateways) may reorder Bundle entries
- Downstream indexing layers may not preserve insertion order
- Future implementers may not realise that array position carries semantic meaning
- The FHIR specification does not guarantee Bundle entry order is preserved

While Option A (List resource) addresses this by making the List the authoritative source of
order rather than Bundle position, it introduces a separate resource that must be managed,
synchronised, and queried alongside the Endpoint and HealthcareService.

### 2. "Primary" and "Copy" labels are ambiguous in the current DoS model

In the DoS example below, endpoints are labelled "Primary" and "Copy" based on their position
in the list and whether another endpoint of the same connection type exists further down:

![DoS Endpoint Details — Bradford Royal Urgent Treatment Centre](./image-2026-5-17_15-36-59.png)

| Order | Transport | Interaction | Format | Business Scenario |
|-------|-----------|-------------|--------|-------------------|
| 1 | itk | urn:nhs-itk:interaction:primaryOutOfHourRecipientNHS111CDADocument-v2-0 | CDA | Primary |
| 2 | itk | urn:nhs-itk:interactioncopyRecipientNHS111CDADocument-v2-0 | CDA | Copy |
| 3 | http | scheduling | FHIR | Primary |







This labelling is derived from position + connection type matching — a fragile inference that
leaves too much to chance. In EPCS, these could instead be declared as two distinct endpoints
with different connection types (even if their URL is the same), making the relationship
explicit rather than implied.

### 3. EPCS should support multiple endpoints of the same connection type with explicit ranking

The generic requirement is: **EPCS should cater for two or more endpoints of the same
connection type and give them explicit ranking/priority/ordering.**

This is not well served by a separate List resource when the priority is an intrinsic property
of the Endpoint itself within the context of its parent HealthcareService.

---

## Option B: `endpoint-priority` Extension on the Endpoint Resource

### Approach

Add a custom FHIR extension directly on the `Endpoint` resource that declares its priority
(ranking/failover order) within the context of its parent HealthcareService.

### Extension Definition

```json
{
  "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
  "valuePositiveInt": 1
}
```

| Field | Value | Description |
|-------|-------|-------------|
| `url` | `https://fhir.nhs.uk/StructureDefinition/endpoint-priority` | Extension URL (could also be named `endpoint-ranking` or `endpoint-failover-priority`) |
| `valuePositiveInt` | 1, 2, 3... | Priority rank. 1 = highest priority (try first). Lower numbers = higher priority. |

### Example: Endpoint with priority extension

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000001",
  "meta": {
    "profile": ["https://fhir.nhs.uk/StructureDefinition/EPC-Endpoint"]
  },
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
      "valuePositiveInt": 1
    }
  ],
  "status": "active",
  "name": "Primary BaRS Endpoint",
  "connectionType": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "hl7-fhir-rest", "display": "HL7 FHIR" }]
  },
  "payloadType": [
    { "coding": [{ "code": "bars", "display": "BaRS" }] }
  ],
  "address": "https://provider.example.nhs.uk/FHIR/R4"
}
```

### Example: Multiple endpoints of the same connection type with ranking

Replicating the DoS example (Bradford Royal UTC) using explicit priority:

```json
[
  {
    "resourceType": "Endpoint",
    "id": "ep-001",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 1 }
    ],
    "status": "active",
    "name": "Primary CDA Endpoint",
    "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
    "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
    "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
  },
  {
    "resourceType": "Endpoint",
    "id": "ep-002",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 2 }
    ],
    "status": "active",
    "name": "Copy CDA Endpoint",
    "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
    "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
    "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
  },
  {
    "resourceType": "Endpoint",
    "id": "ep-003",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 1 }
    ],
    "status": "active",
    "name": "Primary FHIR Scheduling Endpoint",
    "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
    "payloadType": [{ "coding": [{ "code": "scheduling", "display": "Scheduling" }] }],
    "address": "ASID:200000064489"
  }
]
```

In this model:
- `ep-001` and `ep-002` are both ITK/CDA endpoints. Priority 1 vs 2 makes the primary/copy
  relationship explicit without relying on Bundle order or naming conventions.
- `ep-003` is a different connection type (FHIR). Its priority of 1 means it's the primary
  endpoint for that type — there's no ambiguity.
- Priority is **per connection type within a HealthcareService** — two endpoints can both be
  priority 1 if they have different connection types.

---

## API Impact

### No new resource type required

Unlike Option A, this approach does not introduce a `List` resource. Priority is stored
directly on the Endpoint, which means:

- No separate `POST /List`, `PUT /List`, `DELETE /List` operations
- No List synchronisation logic when HealthcareService endpoints change
- No need for consumers to make a separate call to retrieve ordering

### Retrieving ordered endpoints

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
```

The response Bundle contains Endpoint resources with the `endpoint-priority` extension.
The consumer sorts by `extension[endpoint-priority].valuePositiveInt` ascending to determine
the priority order.

### Filtering by connection type + priority

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-...&ConnectionType=hl7-fhir-rest HTTP/1.1
```

Returns only FHIR endpoints. The consumer sorts by the priority extension to determine
which is primary, secondary, etc.

### Setting/changing priority

Priority is set when creating or updating an Endpoint:

```http
PUT /Endpoint/ep-001 HTTP/1.1
```

```json
{
  "resourceType": "Endpoint",
  "id": "ep-001",
  "extension": [
    { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 2 }
  ],
  "status": "active",
  "name": "Demoted CDA Endpoint",
  "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
  "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
  "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
}
```

No separate List update is needed — the priority moves with the Endpoint.

---

## Comparison: Option A (List) vs Option B (Extension)

| Aspect | Option A: List Resource | Option B: Priority Extension |
|--------|------------------------|------------------------------|
| **Ordering mechanism** | Separate `List` resource with `entry[]` array position | `endpoint-priority` extension on each Endpoint |
| **Explicit semantics** | Yes — List is purpose-built for ordered collections | Yes — extension value is unambiguous |
| **Survives Bundle reordering** | Yes — List.entry[] is authoritative, not Bundle position | Yes — priority is on the resource itself |
| **Survives pagination** | Yes | Yes |
| **Additional API surface** | POST/GET/PUT/DELETE /List | None — uses existing Endpoint CRUD |
| **Synchronisation complexity** | High — List must stay in sync with HealthcareService.endpoint[] | None — priority is intrinsic to the Endpoint |
| **Multiple endpoints same type** | Supported (by List position) | Supported (by priority value per type) |
| **Consumer complexity** | Must retrieve List + Endpoints (1–2 calls), apply order client-side | Single call, sort by extension value |
| **FHIR conformance** | Strong — List is a standard FHIR resource for ordered collections | Acceptable — extensions are a standard FHIR extensibility mechanism |
| **Precedent** | Used in some UK FHIR implementations | Recommended by multiple AI-assisted architecture reviews; common pattern in FHIR IGs |
| **Supports "primary/copy" semantics** | Implicitly via position | Explicitly via numeric rank |
| **Operational overhead** | List lifecycle management, auto-creation, auto-sync | Minimal — just an extension field |
| **Risk of stale ordering** | If List gets out of sync with actual Endpoints | None — priority is on the Endpoint itself |

---

## Why Bundle order alone is insufficient

Even without a List or extension, some implementations rely on Bundle entry order to imply
priority. This is problematic because:

1. **Pagination can split results** — priority 1 and priority 2 may end up on different pages
2. **Intermediaries may reorder** — caches, CDNs, and API gateways do not guarantee entry order
3. **Downstream indexing layers may reorder** — databases and search engines have their own sort
4. **Future implementers may not realise order is semantic** — implicit contracts are fragile
5. **The FHIR specification does not mandate Bundle entry order preservation**

For operational failover logic, **explicit metadata is always preferable to implicit ordering**.

---

## Alternative considered: `_include` of List in Endpoint search

One option discussed was to include the List object in the `GET /Endpoint` response Bundle
using the `_include` mechanism (Endpoint entries with `search.mode = 'match'`, List with
`search.mode = 'include'`).

**Why this was rejected:**
- `_include` is designed for following references already present in resources
- `Endpoint` does not reference a `List` — the relationship is the other way around
- While technically defensible, it goes against FHIR design principles
- A `GET /List` with `_include=List:item` (where List is the match and Endpoints are included)
  is more natural but contradicts the core EPCS API design pattern of `GET /Endpoint` as the
  primary query

---

## Fundamental Limitations of Option C

### Problem 1: Endpoints Can Be Associated with Multiple HealthcareServices

An Endpoint in the Endpoint Catalog can be referenced by **multiple HealthcareServices**.
For example, a single BaRS Endpoint at `https://provider.nhs.uk/FHIR/R4` might serve both
an Urgent Treatment Centre service and a GP Out-of-Hours service.

This breaks the extension-based priority model entirely:

- If `Endpoint/ep-001` has `endpoint-priority: 1`, that value applies globally to the
  Endpoint resource — it cannot express "priority 1 for Service A but priority 3 for
  Service B"
- Priority is inherently a **relationship property** (the priority of this Endpoint
  *within the context of* a specific HealthcareService), not an intrinsic property of the
  Endpoint itself
- A single extension value on the Endpoint cannot represent different priorities for
  different services

**This is a fundamental architectural mismatch.** The extension approach only works if
every Endpoint belongs to exactly one HealthcareService — which is not the case in the
EPC data model.

Options A and B (List-based) do not have this problem because the List is scoped to a
specific HealthcareService (`List.subject = HealthcareService/{id}`). Each service has its
own List with its own ordering, and the same Endpoint can appear at different positions in
different Lists.

| Scenario | Option A/B (List) | Option C (Extension) |
|----------|-------------------|---------------------|
| Endpoint shared by 2 services | Each service has its own List with independent ordering | ❌ Single priority value — cannot differ per service |
| Endpoint priority 1 for Service A, priority 3 for Service B | ✅ Supported | ❌ Impossible without complex workarounds |
| Endpoint removed from one service but not another | ✅ Remove from that service's List only | ✅ Remove from HealthcareService.endpoint[] |

---

### Problem 2: Template Model — Endpoint Doesn't Know Its Own Type

In the EPC architecture, `connectionType`, `payloadType`, and `address` (URL) are **not
stored on the Endpoint resource itself** — they live on the parent **Template**. An Endpoint
is essentially an instance of a Template, inheriting those fields at read time when the
Lambda resolves the join.

This means the bare Endpoint record in the data store looks like:

```json
{
  "resourceType": "Endpoint",
  "id": "ep-001",
  "status": "active",
  "name": "Primary BaRS Endpoint",
  "managingOrganization": [{ "identifier": { "value": "R778" } }]
}
```

The `connectionType`, `payloadType`, and `address` are projected onto the response by the
Lambda at query time — they don't exist on the Endpoint itself.

### Why This Matters for "Priority Per Connection Type"

The review comment suggests priority should be **per connection type** — two ITK/CDA
endpoints can both be priority 1 if they're a different type from a FHIR/scheduling
endpoint. But if the Endpoint doesn't know its own connection type (the Template does):

1. **The extension can't self-describe what it's ranking against.** An
   `endpoint-priority: 1` on `ep-001` means "I'm priority 1" — but priority 1 *of what*?
   The Endpoint doesn't know it's an ITK/CDA endpoint until the Template is resolved.

2. **Priority becomes ambiguous without Template context.** If a HealthcareService has 3
   Endpoints — two using a CDA Template and one using a FHIR Template — the priority
   extension alone can't express "priority 1 within CDA" vs "priority 1 within FHIR"
   without the consumer already knowing which Template each Endpoint belongs to.

3. **The extension would need to be either:**
   - **Global** (priority across all Endpoints on the service regardless of type) — simpler
     but less expressive
   - **Scoped to the Template** (priority within Endpoints sharing the same Template) —
     more expressive but requires the consumer to group by Template first, which they can
     only do after the Lambda resolves the join

### How Option A (List) Handles This

Option A sidesteps this problem because:
- The List is a **single global ordering** across all Endpoints on a service
- Type filtering happens **after** the List is retrieved (via `GET /Endpoint` with
  `ConnectionType`/`PayloadType` filters)
- The Lambda resolves the Template join during the filter query, so the consumer never
  needs to know about Templates
- The List order is applied client-side to the already-filtered results

The List doesn't need to know about Templates because it's just ordering Endpoint
references — the type resolution is a separate concern handled by the Lambda.

### Approaches for Option B to Work with Templates

**Approach 1: Global priority (simple but less expressive)**
- The extension is a single global rank across all Endpoints on the service
- Consumer retrieves filtered Endpoints (Lambda resolves Templates), then sorts by priority
- Two endpoints of different types can have the same priority value — it only matters
  within a type group after filtering
- Functionally equivalent to Option A's List position but without the separate resource

**Approach 2: Put the extension on the Template instead**
- Since `connectionType` and `payloadType` live on the Template, put `endpoint-priority`
  there too
- **Problem:** Templates are shared across multiple Endpoints and HealthcareServices — a
  Template-level priority doesn't make sense per-service

**Approach 3: Composite extension with Template reference**
```json
{
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
      "extension": [
        { "url": "priority", "valuePositiveInt": 1 },
        { "url": "template", "valueReference": { "reference": "Endpoint/template-001" } }
      ]
    }
  ]
}
```
This makes the priority explicitly scoped to a Template — but adds complexity and couples
the Endpoint to its Template in a way the current design avoids.

### Template Impact Summary

| Concern | Option A (List) | Option B (Extension) |
|---------|----------------|---------------------|
| Needs to know about Templates? | No — List is type-agnostic; filtering is separate | Yes — "per type" priority requires Template awareness |
| Where does type info live? | Resolved by Lambda at query time | Same — but extension can't self-describe its type scope |
| Global priority works? | Yes (single ordered list) | Yes — but loses "per type" expressiveness |
| Per-type priority works? | Yes (filter after ordering) | Only if consumer groups by Template first (after Lambda resolves) |
| Complexity | Higher (separate resource) | Lower operationally, but semantically ambiguous without Template context |

**Conclusion:** Option B works cleanly with a **global priority** model (one rank per
Endpoint, consumers filter by type then sort by priority). It becomes awkward if priority
needs to be explicitly scoped per connection type, because the Endpoint doesn't know its
own type without Template resolution. Option A avoids this entirely by keeping ordering and
type filtering as orthogonal concerns.

---

## Open Questions

| # | Question | For |
|---|----------|-----|
| 1 | Should priority be **per connection type** (two endpoints of different types can both be priority 1) or **global across all endpoints** on a service? | Architecture / IOPS |
| 2 | What is the correct extension URL namespace? (`https://fhir.nhs.uk/StructureDefinition/endpoint-priority` vs `https://fhir.nhs.uk/StructureDefinition/endpoint-failover-priority`) | IOPS / FHIR governance |
| 3 | Should the extension be mandatory on all Endpoints, or optional (with unranked endpoints treated as lowest priority)? | Product |
| 4 | Does DoS have requirements beyond simple numeric ranking (e.g. weighted routing, percentage-based distribution)? | DoS leads |
| 5 | Can we take this proposal to IOPS for a second opinion on the extension approach vs the List approach? | Architecture |

---

## Recommendation

### Summary

| Context | Recommended Option | Rationale |
|---------|-------------------|-----------|
| **PoC** | Option A (Explicit List) | Already designed and documented; FHIR-conformant |
| **Production (consumer experience)** | Option A + B (Implicit List) | List for storage/admin, API applies order transparently for consumers |
| **Production (if List complexity is rejected)** | Option C (Extension) | Simpler operationally but requires IOPS sign-off on Template model implications |

### Pros and Cons

| | Option A: Explicit List | Option B: Implicit List | Option C: Priority Extension |
|---|---|---|---|
| **Pros** | • FHIR-conformant (List is purpose-built for ordered collections) | • Simplest consumer experience (one call, pre-ordered results) | • No additional resource to manage |
| | • Single authoritative ordering document per service | • Backward compatible (no consumer code changes) | • Priority is intrinsic to the Endpoint — no sync risk |
| | • Clear separation of ordering from type filtering | • Template resolution + ordering in one Lambda call | • Lowest operational overhead |
| | • Well-understood FHIR pattern | • List still available for admin/debugging | • Simple to implement |
| | • Supports `_include=List:item` for single-call retrieval | • Graceful fallback if no List exists | • Works well with global priority model |
| **Cons** | • Additional API surface (5 CRUD operations on /List) | • Non-standard FHIR behaviour (must be documented) | • **Critical: Endpoints shared across multiple HealthcareServices cannot have per-service priority** |
| | • Synchronisation complexity (auto-create, auto-sync on PUT/PATCH) | • Implicit contract (consumers may not realise order is meaningful) | • Template model means Endpoint doesn't know its own connectionType |
| | • Consumer must retrieve List separately and apply order client-side | • Harder to debug (List not visible without admin call) | • "Per connection type" priority is ambiguous without Template resolution |
| | • Risk of List drifting out of sync with actual Endpoints | • Every query requires additional List lookup (mitigated by caching) | • Requires IOPS review for extension URL governance |
| | • 1–2 API calls required for ordered results | • Same sync complexity as Option A (List still exists internally) | • No FHIR precedent for this specific extension in NHS IGs |

### Decision Factors

> **Decision: Option A adopted. Option B is a future enhancement. Option C is not adopted.**

- **Option A** (FHIR List resource) is the adopted implementation. It provides explicit,
  FHIR-conformant priority ordering that consumers can query directly. This is the current
  design and is being implemented.

- **Option B** (implicit ordering — API auto-applies List order in responses) is a planned
  **future enhancement** that builds on Option A. When implemented, consumers will receive
  Endpoints pre-sorted without needing to query the List separately. The List resource
  still exists as the storage mechanism and for admin management — Option B simply adds
  server-side ordering to consumer-facing responses.

- **Option C** (priority extension on Endpoint resource) is **not adopted** due to a
  fundamental architectural limitation: Endpoints can be shared across multiple
  HealthcareServices, so a single priority extension value on the Endpoint cannot express
  different priorities for different services. Combined with the Template model (Endpoint
  doesn't hold its own connectionType), Option C is unsuitable for the EPC data model.

### Next Steps

| # | Action | Owner | Status |
|---|--------|-------|--------|
| 1 | Confirm whether priority should be global or per connection type | Architecture / DoS leads | Open |
| 2 | Implement Option A List operations in the Lambda | Development team | In progress |
| 3 | Plan Option B (implicit ordering in GET /Endpoint responses) as a future iteration | Architecture | Future |
