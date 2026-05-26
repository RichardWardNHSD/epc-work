# DUEC Endpoints

> ⚠️ **Draft — best guess only:** The content of this document, particularly the
> `connectionType` and `payloadType` codes used for DUEC, has not been confirmed with the
> DUEC programme. The codes, address formats, and modelling approach are based on the
> author's best interpretation of the FHIR R4 standard and the Endpoint Catalog design.
> This document should be reviewed and validated with the DUEC team before being used to
> inform any implementation.

## Overview

The Directory of Urgent and Emergency Care (DUEC) uses the Endpoint Catalog API to manage
the technical connection details for urgent and emergency care services. Each service is
represented as a `HealthcareService` resource with one or more associated `Endpoint`
resources describing how senders can connect to it.

DUEC services support a **tiered communication model** with three connection methods in
descending preference:

| Priority | Protocol | connectionType | payloadType | Description |
|----------|----------|---------------|-------------|-------------|
| 1 (primary) | FHIR REST | `hl7-fhir-rest` | `duec` | Direct FHIR R4 REST interaction — preferred method |
| 2 (secondary) | ITK3 FHIR Messaging | `hl7-fhir-msg` | `duec-itk` | ITK3 FHIR messaging — fallback when FHIR REST is unavailable |
| 3 (fallback) | Secure email | `secure-email` | `duec-email` | Email — last resort when both FHIR channels are unavailable |

A service owner expresses this preference using a FHIR `List` resource. Senders retrieve
the List to determine which Endpoint to try first, then filter by the `connectionType` and
`payloadType` they support.

This document covers:
- How each connection type is modelled as an Endpoint
- How the `List` resource expresses the communication preference
- Worked examples for creating and managing DUEC Endpoints

For full detail on the List resource design and all available operations, see
[Endpoint Ordering using the FHIR List Resource](./endpoint-ordering-with-list.md).

For detail on how `connectionType` and `payloadType` filtering works via the Lambda and
Template model, see
[Filtering Endpoints by Connection Type and Payload Type](./filtering-endpoints-by-connection-and-payload-type.md).

---

## connectionType and payloadType

Every Endpoint carries a `connectionType` and a `payloadType`. These are stored on the
parent **Template** and resolved at read time — consumers always see fully-populated
Endpoints.

### connectionType values used by DUEC

| Code | System | Display | Use |
|------|--------|---------|-----|
| `hl7-fhir-rest` | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` | HL7 FHIR | Direct FHIR R4 REST — primary DUEC connection method |
| `hl7-fhir-msg` | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` | HL7 FHIR Messaging | ITK3 FHIR messaging — secondary connection method |
| `secure-email` | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` | Secure email | Email fallback — last resort |

### payloadType values used by DUEC

| Code | System | Display | Use |
|------|--------|---------|-----|
| `duec` | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` | DUEC | DUEC interactions over FHIR REST |
| `duec-itk` | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` | DUEC ITK | DUEC interactions over ITK3 FHIR messaging |
| `duec-email` | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` | DUEC Email | DUEC interactions via secure email |

---

## How each connection type is modelled

### Primary — FHIR REST (`hl7-fhir-rest / duec`)

The FHIR REST Endpoint is the preferred connection method. The `address` is the base URL
of the receiving FHIR server. Senders interact with it using standard FHIR R4 REST
operations.

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000001",
  "status": "active",
  "name": "Anytown UTC — FHIR REST Endpoint",
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
          "system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc",
          "code": "duec",
          "display": "DUEC"
        }
      ]
    }
  ],
  "managingOrganization": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R778"
      }
    }
  ],
  "address": "https://anytown-utc.provider.nhs.uk/FHIR/R4",
  "header": "public"
}
```

---

### Secondary — ITK3 FHIR Messaging (`hl7-fhir-msg / duec-itk`)

The ITK3 Endpoint is used when the FHIR REST Endpoint is unavailable. ITK3 uses FHIR
messaging (`hl7-fhir-msg`) — the `address` is the base URL of the ITK3 messaging endpoint.
Senders construct a FHIR `Bundle` of type `message` and `POST` it to this address.

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000002",
  "status": "active",
  "name": "Anytown UTC — ITK3 Messaging Endpoint",
  "connectionType": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
        "code": "hl7-fhir-msg",
        "display": "HL7 FHIR Messaging"
      }
    ]
  },
  "payloadType": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc",
          "code": "duec-itk",
          "display": "DUEC ITK"
        }
      ]
    }
  ],
  "managingOrganization": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R778"
      }
    }
  ],
  "address": "https://anytown-utc.provider.nhs.uk/itk3/messaging",
  "header": "public"
}
```

---

### Fallback — Secure email (`secure-email / duec-email`)

The email Endpoint is the last resort when both FHIR channels are unavailable. The
`address` is an `mailto:` URI. Senders should use secure/encrypted email where possible.

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000003",
  "status": "active",
  "name": "Anytown UTC — Email Fallback Endpoint",
  "connectionType": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
        "code": "secure-email",
        "display": "Secure email"
      }
    ]
  },
  "payloadType": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc",
          "code": "duec-email",
          "display": "DUEC Email"
        }
      ]
    }
  ],
  "managingOrganization": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R778"
      }
    }
  ],
  "address": "mailto:referrals@anytown-utc.nhs.uk",
  "header": "public"
}
```

---

## Communication preference and the List resource

A DUEC service will have all three Endpoints registered. The `List` resource expresses the
preferred order in which senders should attempt them. The List is automatically created when
the `HealthcareService` is registered — the order of `endpoint[]` references in the
`POST /HealthcareService` payload sets the initial priority.

```
HealthcareService (Anytown UTC)
    │
    └── List.subject
            │
            ├── entry[0] → Endpoint/...001  hl7-fhir-rest / duec      ← try first
            ├── entry[1] → Endpoint/...002  hl7-fhir-msg  / duec-itk  ← try second
            └── entry[2] → Endpoint/...003  secure-email  / duec-email ← last resort
```

Senders filter by the `connectionType` and `payloadType` they support, then apply the List
order to the filtered results. A sender that only supports FHIR REST will only see
`...001`; a sender that supports both FHIR REST and ITK3 will see `...001` and `...002` and
know to try `...001` first.

---

## Worked examples

### Example 1: Create the HealthcareService with all three Endpoints

The `endpoint[]` array order in the `POST /HealthcareService` payload sets the initial
priority. The API creates the HealthcareService and its associated List automatically.

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
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "FHIR REST Endpoint" },
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "ITK3 Messaging Endpoint" },
    { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Email Fallback Endpoint" }
  ]
}
```

#### Auto-created List

```json
{
  "resourceType": "List",
  "id": "f7d3e2a1-0000-0000-0000-000000000099",
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
  "orderedBy": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/list-order",
      "code": "priority",
      "display": "Sorted by Priority"
    }]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "FHIR REST Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "ITK3 Messaging Endpoint" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Email Fallback Endpoint" } }
  ]
}
```

---

### Example 2: FHIR REST sender — retrieve the highest-priority usable Endpoint

A sender that supports only `hl7-fhir-rest / duec` retrieves the priority List, then
filters Endpoints by type. Only `...001` matches — it is the correct Endpoint to use.

#### Step 1 — Get the priority List

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: C3D4E5F6-3333-4444-5555-6F19GH71F280
X-Correlation-Id: D4E5F6G7-4444-5555-6666-7G20HI82G391
NHSD-End-User-Organisation-ODS: A1001
```

Priority sequence from `List.entry[]`:
1. `Endpoint/...001` (FHIR REST)
2. `Endpoint/...002` (ITK3)
3. `Endpoint/...003` (Email)

#### Step 2 — Filter by connectionType and payloadType

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=duec HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: E5F6G7H8-5555-6666-7777-8H31IJ93H402
X-Correlation-Id: F6G7H8I9-6666-7777-8888-9I42JK04I513
NHSD-End-User-Organisation-ODS: A1001
```

Returns only `...001`. The sender uses `https://anytown-utc.provider.nhs.uk/FHIR/R4`.

---

### Example 3: Multi-protocol sender — apply List order across supported types

A sender supports both `hl7-fhir-rest / duec` and `hl7-fhir-msg / duec-itk`. It retrieves
all matching Endpoints and applies the List order to determine which to try first.

#### Step 1 — Get the priority List

Same as Example 2. Priority sequence: `...001` → `...002` → `...003`.

#### Step 2 — Filter by supported types

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=duec HTTP/1.1
```

Returns `...001`.

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-msg&PayloadType=duec-itk HTTP/1.1
```

Returns `...002`.

#### Step 3 — Apply List order

```
List position 0 → Endpoint/...001  ✓ hl7-fhir-rest / duec     → Priority 1 (try first)
List position 1 → Endpoint/...002  ✓ hl7-fhir-msg  / duec-itk → Priority 2 (fallback)
List position 2 → Endpoint/...003  ✗ not supported by sender   → skip
```

The sender tries the FHIR REST Endpoint first. If that fails, it falls back to ITK3.

---

### Example 4: FHIR REST Endpoint goes offline — promote ITK3 to primary

The FHIR REST Endpoint is temporarily unavailable. The service owner updates the List to
promote the ITK3 Endpoint to index 0 so senders route to it immediately.

```http
PUT /List/f7d3e2a1-0000-0000-0000-000000000099 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: G7H8I9J0-7777-8888-9999-0J53KL15J624
X-Correlation-Id: H8I9J0K1-8888-9999-0000-1K64LM26K735
NHSD-End-User-Organisation-ODS: R778
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
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/list-order",
      "code": "priority",
      "display": "Sorted by Priority"
    }]
  },
  "entry": [
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002", "display": "ITK3 Messaging Endpoint (now primary)" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001", "display": "FHIR REST Endpoint (temporarily secondary)" } },
    { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003", "display": "Email Fallback Endpoint" } }
  ]
}
```

Senders that support ITK3 will now route to `...002` first. When the FHIR REST Endpoint
is restored, a further `PUT /List/{id}` reinstates the original order.

---

### Example 5: Retrieve all three Endpoints in one call

```http
GET /List?subject=HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc&status=current&_include=List:item HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: I9J0K1L2-9999-0000-1111-1L75MN37L846
X-Correlation-Id: J0K1L2M3-0000-1111-2222-2M86NO48M957
NHSD-End-User-Organisation-ODS: A1001
```

The response Bundle contains the `List` (mode: `match`) and all three `Endpoint` resources
(mode: `include`). The sender applies the order by iterating `List.entry[]` and retaining
only the Endpoints it supports — the Bundle entry sequence is not authoritative.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "List",
        "id": "f7d3e2a1-0000-0000-0000-000000000099",
        "entry": [
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002" } },
          { "item": { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003" } }
        ]
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "name": "Anytown UTC — FHIR REST Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "duec", "display": "DUEC" }] }],
        "address": "https://anytown-utc.provider.nhs.uk/FHIR/R4"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Anytown UTC — ITK3 Messaging Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-msg", "display": "HL7 FHIR Messaging" }] },
        "payloadType": [{ "coding": [{ "code": "duec-itk", "display": "DUEC ITK" }] }],
        "address": "https://anytown-utc.provider.nhs.uk/itk3/messaging"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000003",
        "status": "active",
        "name": "Anytown UTC — Email Fallback Endpoint",
        "connectionType": { "coding": [{ "code": "secure-email", "display": "Secure email" }] },
        "payloadType": [{ "coding": [{ "code": "duec-email", "display": "DUEC Email" }] }],
        "address": "mailto:referrals@anytown-utc.nhs.uk"
      },
      "search": { "mode": "include" }
    }
  ]
}
```

---

## Summary

| Task | API call | Notes |
|------|----------|-------|
| Create a DUEC service with all three Endpoints | `POST /HealthcareService` | Order `endpoint[]` as FHIR REST → ITK3 → Email — List is created automatically |
| Retrieve Endpoints in priority order | `GET /List?subject=...` then `GET /Endpoint?...&ConnectionType=...&PayloadType=...` | Apply List order to filtered results client-side |
| Retrieve all Endpoints in one call | `GET /List?subject=...&_include=List:item` | Apply `List.entry[]` order — not Bundle entry sequence |
| Temporarily promote ITK3 to primary | `PUT /List/{id}` | Resequence `entry[]` — no changes to Endpoints or HealthcareService needed |
| Restore original order after outage | `PUT /List/{id}` | Resequence `entry[]` back to FHIR REST → ITK3 → Email |
| Permanently deactivate an Endpoint | Set `Endpoint.status` to `entered-in-error`, then `PUT /List/{id}` to remove it | Remove from List to keep priority order clean |

---

## Related documents

- [Endpoint Ordering using the FHIR List Resource](./endpoint-ordering-with-list.md) — full
  detail on the List resource design, all API operations, consumer patterns, and the
  interaction between ordering and type filtering
- [Filtering Endpoints by Connection Type and Payload Type](./filtering-endpoints-by-connection-and-payload-type.md) — how `connectionType` and `payloadType` filtering works via the Lambda and Template model
