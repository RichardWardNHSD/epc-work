# Searching for Endpoint Information by Service ID

This guide explains how to retrieve `Endpoint` resources associated with a specific
`HealthcareService` using the Endpoint Catalog API. Two patterns are supported, corresponding
to ADR-114 Option 1 and Option 3B.

**Base URL:** `https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4`

---

## Required headers

All requests to `/HealthcareService` and `/Endpoint` require the following headers:

| Header | Required | Description |
|--------|----------|-------------|
| `X-Request-Id` | Yes | UUID for this request, mirrored in the response |
| `X-Correlation-Id` | Yes | UUID for tracing across systems |
| `Accept` | Yes | Must include the BaRS version, e.g. `application/fhir+json;version=1.4.0` |
| `NHSD-End-User-Organisation-ODS` | Yes | ODS code of the requesting organisation, e.g. `A1001` |
| `Authorization` | Yes | Bearer token from OAuth2 token endpoint |

---

## Option 1 — FHIR `_include` on the HealthcareService search endpoint

**Use this when** you want to retrieve a HealthcareService and all its associated Endpoints
in a single request.

### How it works

`GET /HealthcareService` supports two query parameters that combine to deliver this:

| Parameter | Type | Description |
|-----------|------|-------------|
| `_id` | uuid | The FHIR resource `id` of the HealthcareService (the Service ID) |
| `_include` | string | Must be `HealthcareService:endpoint` — instructs the server to include all referenced Endpoint resources in the response Bundle |

The server returns a FHIR `Bundle` of type `searchset` containing:
- **1** `HealthcareService` resource (the matched service)
- **0..\*** `Endpoint` resources (all endpoints referenced by that service via `HealthcareService.endpoint[]`)

### Request

```http
GET /HealthcareService?_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&_include=HealthcareService:endpoint HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 60E0B220-8136-4CA5-AE46-1D97EF59D068
X-Correlation-Id: 11C46F5F-CDEF-4865-94B2-0EE0EDCC26DA
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK

```json
{
  "resourceType": "Bundle",
  "id": "b4f3a1c2-0000-0000-0000-000000000001",
  "type": "searchset",
  "total": 1,
  "timestamp": "2026-05-06T10:00:00+00:00",
  "link": [
    {
      "relation": "self",
      "url": "https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4/HealthcareService?_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&_include=HealthcareService:endpoint"
    }
  ],
  "entry": [
    {
      "resource": {
        "resourceType": "HealthcareService",
        "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
        "meta": {
          "lastUpdated": "2026-04-01T09:00:00+00:00",
          "profile": [
            "https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"
          ]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/Id/dos-service-id",
            "value": "1000099999"
          }
        ],
        "active": true,
        "name": "Anytown Urgent Treatment Centre",
        "providedBy": {
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "A1001"
          }
        },
        "endpoint": [
          {
            "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001"
          },
          {
            "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002"
          }
        ]
      },
      "search": {
        "mode": "match"
      }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "meta": {
          "lastUpdated": "2026-04-01T09:00:00+00:00",
          "profile": [
            "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Endpoint"
          ]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/Id/EndpointSystem",
            "value": "urn:nhs:addressing:asid:200000000001"
          }
        ],
        "status": "active",
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
        "address": "https://provider.example.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": {
        "mode": "include"
      }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Anytown UTC — Referral Endpoint",
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
                "code": "bars-referral",
                "display": "BaRS Referral"
              }
            ]
          }
        ],
        "address": "https://provider.example.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": {
        "mode": "include"
      }
    }
  ]
}
```

### Key points

- The `HealthcareService` entry has `search.mode: "match"` — it is the primary matched resource.
- Each `Endpoint` entry has `search.mode: "include"` — they are included because of `_include`.
- If the service has no endpoints, the Bundle will contain only the `HealthcareService` entry and `total: 1`.
- If no HealthcareService matches `_id`, the response is an empty Bundle with `total: 0`.
- An Endpoint with `header: "private"` will have its `address` field omitted unless the requesting organisation is the owner.

### Combining with other filters

`_id` can be combined with other search parameters:

```http
GET /HealthcareService?_id=9f2c6f12-1a6d-4d9c-a111-123456789abc
  &_include=HealthcareService:endpoint
  &HealthcareService.providedBy=https://fhir.nhs.uk/Id/ods-organization-code|A1001
```

### Identifying Endpoints of the correct type

`GET /HealthcareService?_id=...&_include=HealthcareService:endpoint` always returns **all**
Endpoints associated with the service — there is no server-side filter on `connectionType`
or `payloadType` for this call. A service may have several Endpoints supporting different
protocols and message standards, so the consumer must inspect the `connectionType` and
`payloadType` fields on each included Endpoint and select the ones relevant to their use case.

#### What the fields mean

| Field | Location | Purpose |
|-------|----------|---------|
| `connectionType.coding.code` | On each Endpoint | The technical protocol, e.g. `hl7-fhir-rest`, `ihe-xds` |
| `payloadType.coding.code` | On each Endpoint | The message standard, e.g. `bars`, `bars-referral` |

#### Example: service with three Endpoints of different types

A service has three Endpoints:

| Endpoint id | connectionType | payloadType |
|-------------|---------------|-------------|
| `e001` | `hl7-fhir-rest` | `bars` |
| `e002` | `hl7-fhir-rest` | `bars-referral` |
| `e003` | `ihe-xds` | `xds-referral` |

The request is the same regardless of which type you want:

```http
GET /HealthcareService?_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&_include=HealthcareService:endpoint HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 60E0B220-8136-4CA5-AE46-1D97EF59D068
X-Correlation-Id: 11C46F5F-CDEF-4865-94B2-0EE0EDCC26DA
NHSD-End-User-Organisation-ODS: A1001
```

The response Bundle contains all three Endpoints:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "HealthcareService",
        "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
        "name": "Anytown Urgent Treatment Centre",
        "endpoint": [
          { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" },
          { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002" },
          { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003" }
        ]
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "name": "Anytown UTC — BaRS Booking Endpoint",
        "connectionType": {
          "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "hl7-fhir-rest", "display": "HL7 FHIR" }]
        },
        "payloadType": [
          { "coding": [{ "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type", "code": "bars", "display": "BaRS" }] }
        ],
        "address": "https://bars.provider.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Anytown UTC — BaRS Referral Endpoint",
        "connectionType": {
          "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "hl7-fhir-rest", "display": "HL7 FHIR" }]
        },
        "payloadType": [
          { "coding": [{ "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type", "code": "bars-referral", "display": "BaRS Referral" }] }
        ],
        "address": "https://bars.provider.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": { "mode": "include" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000003",
        "status": "active",
        "name": "Anytown UTC — IHE XDS Endpoint",
        "connectionType": {
          "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "ihe-xds", "display": "IHE XDS" }]
        },
        "payloadType": [
          { "coding": [{ "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type", "code": "xds-referral", "display": "XDS Referral" }] }
        ],
        "address": "https://xds.provider.nhs.uk/xds",
        "header": "public"
      },
      "search": { "mode": "include" }
    }
  ]
}
```

#### Client-side filtering examples

**Example A — consumer wants only BaRS booking endpoints (`bars`)**

Iterate the Bundle entries, keep only those where:
- `resource.resourceType == "Endpoint"`
- `resource.connectionType.coding.code == "hl7-fhir-rest"`
- `resource.payloadType.coding.code == "bars"`

Result: `e001` only.

```
e001  connectionType=hl7-fhir-rest  payloadType=bars          ✓ keep
e002  connectionType=hl7-fhir-rest  payloadType=bars-referral ✗ skip
e003  connectionType=ihe-xds        payloadType=xds-referral  ✗ skip
```

**Example B — consumer wants all FHIR REST endpoints regardless of payload type**

Keep entries where:
- `resource.resourceType == "Endpoint"`
- `resource.connectionType.coding.code == "hl7-fhir-rest"`

Result: `e001` and `e002`.

```
e001  connectionType=hl7-fhir-rest  ✓ keep
e002  connectionType=hl7-fhir-rest  ✓ keep
e003  connectionType=ihe-xds        ✗ skip
```

**Example C — consumer wants all endpoints for a specific payload type (`bars-referral`)**

Keep entries where:
- `resource.resourceType == "Endpoint"`
- `resource.payloadType.coding.code == "bars-referral"`

Result: `e002` only.

```
e001  payloadType=bars          ✗ skip
e002  payloadType=bars-referral ✓ keep
e003  payloadType=xds-referral  ✗ skip
```

**Example D — consumer wants all active endpoints**

Keep entries where:
- `resource.resourceType == "Endpoint"`
- `resource.status == "active"`

Result: all three — `e001`, `e002`, `e003`.

#### Separating Endpoints from the HealthcareService in the Bundle

The Bundle mixes `HealthcareService` (mode: `match`) and `Endpoint` (mode: `include`)
entries. To process only Endpoints, filter by both `resourceType` and `search.mode`:

```
For each entry in Bundle.entry:
  if entry.resource.resourceType == "Endpoint"
  and entry.search.mode == "include":
    → this is an included Endpoint, inspect connectionType and payloadType
```

The `HealthcareService` entry (`search.mode: "match"`) should be handled separately for
service metadata — name, identifier, active status, providedBy organisation.

---

## Option 3B — Reverse chaining using `_has`

**Use this when** you want only `Endpoint` resources, without the parent `HealthcareService`
in the response.

### How it works

FHIR R4 supports reverse chaining via `_has`, which selects resources based on properties of
other resources that refer to them (see [FHIR R4 §3.1.1.4.16](https://hl7.org/fhir/R4/search.html#has)).

The parameter name encodes the full reverse chain:

```
_has:HealthcareService:endpoint:_id={serviceId}
```

| Segment | Meaning |
|---------|---------|
| `_has` | Reverse chaining modifier |
| `HealthcareService` | The resource type that references back to Endpoint |
| `endpoint` | The search parameter on HealthcareService that holds the Endpoint reference |
| `_id` | Filter the HealthcareService by its FHIR resource id |
| `{serviceId}` | The UUID of the HealthcareService whose Endpoints you want |

The server returns a FHIR `Bundle` of type `searchset` containing:
- **0..\*** `Endpoint` resources only — no HealthcareService in the response

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
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
  "id": "c5e4d2f1-0000-0000-0000-000000000002",
  "type": "searchset",
  "total": 2,
  "timestamp": "2026-05-06T10:00:00+00:00",
  "link": [
    {
      "relation": "self",
      "url": "https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4/Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc"
    }
  ],
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "meta": {
          "lastUpdated": "2026-04-01T09:00:00+00:00",
          "profile": [
            "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Endpoint"
          ]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/Id/EndpointSystem",
            "value": "urn:nhs:addressing:asid:200000000001"
          }
        ],
        "status": "active",
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
        "address": "https://provider.example.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": {
        "mode": "match"
      }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Anytown UTC — Referral Endpoint",
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
                "code": "bars-referral",
                "display": "BaRS Referral"
              }
            ]
          }
        ],
        "address": "https://provider.example.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": {
        "mode": "match"
      }
    }
  ]
}
```

### Key points

- All entries have `search.mode: "match"` — every resource in the Bundle is a directly matched Endpoint.
- The parent HealthcareService is **not** included in the response.
- If no HealthcareService with that `_id` exists, or it has no Endpoints, the response is an empty Bundle with `total: 0`.
- An Endpoint with `header: "private"` will have its `address` field omitted unless the requesting organisation is the owner.

### Combining with other filters

The `_has` parameter can be combined with standard Endpoint search parameters to narrow results further:

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc
  &ConnectionType=hl7-fhir-rest
  &PayloadType=bars
```

This returns only the Endpoints for that service that use the `hl7-fhir-rest` connection type
and `bars` payload type.

---

## Comparison

| | Option 1 | Option 3B |
|---|---|---|
| **Endpoint** | `GET /HealthcareService` | `GET /Endpoint` |
| **Key parameter** | `_id={serviceId}&_include=HealthcareService:endpoint` | `_has:HealthcareService:endpoint:_id={serviceId}` |
| **Response contains** | HealthcareService + Endpoints | Endpoints only |
| **FHIR pattern** | Standard search + `_include` | Standard reverse chaining (`_has`) |
| **Best for** | When you need service metadata alongside endpoint details | When you only need endpoint details |
| **API calls needed** | 1 | 1 |

Both patterns are fully FHIR R4 compliant and supported by the Endpoint Catalog API.

---

## Error responses

Both endpoints return FHIR `OperationOutcome` resources for errors:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "invalid",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/CodeSystem/Spine-ErrorOrWarningCode",
            "code": "INVALID_PARAMETER",
            "display": "Invalid parameter"
          }
        ]
      },
      "diagnostics": "Invalid value for parameter _id: must be a valid UUID"
    }
  ]
}
```

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Success — Bundle returned (may have `total: 0` if no matches) |
| 400 | Bad request — invalid parameter value or format |
| 401 | Unauthorised — missing or invalid Bearer token |
| 403 | Forbidden — valid token but insufficient permissions |
| 4XX | Other client error — see OperationOutcome for details |
| 5XX | Server error — see OperationOutcome for details |
