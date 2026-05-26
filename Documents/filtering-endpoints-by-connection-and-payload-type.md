# Filtering Endpoints by Connection Type and Payload Type

## The Template Endpoint model

### What is a Template?

A Template is an Endpoint whose `environmentType` is set to `staging`. This is what
distinguishes a Template from a regular operational Endpoint at the data level, allowing
it to act as a reusable definition of shared characteristics.

`environmentType` is a FHIR R5 field ([Endpoint.environmentType](https://build.fhir.org/endpoint-definitions.html#Endpoint.environmentType))
that has been back-ported into this implementation to support the concept of a template
Endpoint. It is an **internal field** — it is not returned by the API in any response.

### Endpoint own fields

Every Endpoint in the Endpoint Catalog is a **template child**. An Endpoint resource
holds only three meaningful fields of its own:

| Field | Description |
|-------|-------------|
| status | Whether this endpoint is currently active, suspended, etc. |
| period | The date range during which this endpoint is valid for this service |
| template | A reference to the parent Template |

All other characteristics — connectionType, payloadType, address, name, identifier,
header, managingOrganization — live on the parent Template

### Why this design?

A single Template can be the parent of many Endpoint resources across many
HealthcareService instances. When the protocol or address changes, only the template needs
updating — all child Endpoints automatically reflect the new values without any cascade write
to individual Endpoint records.

---

## How filtering works: server-side template resolution

Because connectionType and payloadType live on the template rather than the Endpoint,
the API server (an AWS Lambda function) resolves the template join **before returning the
response**. From the consumer's perspective this is a single call — the Lambda handles the
fan-out internally.

The Lambda:
1. Fetches all child Endpoints for the HealthcareService
2. Fetches the parent template for each Endpoint
3. Applies the ConnectionType and/or PayloadType filter against the template fields
4. Discards non-matching Endpoints
5. Merges each passing Endpoint's status and period with its template fields
6. Returns the merged resources in a Bundle

The consumer sees a single HTTP request and a single response. Template resolution is an
implementation detail of the Lambda — not visible to the consumer.

---

## Example 1: filter by ConnectionType and PayloadType

### Scenario

A consumer wants all endpoints for HealthcareService 9f2c6f12-1a6d-4d9c-a111-123456789abc
that support connectionType: hl7-fhir-rest and payloadType: bars.

The service has three Endpoints with the following templates:

| Endpoint | Template connectionType | Template payloadType |
|----------|------------------------|----------------------|
| e001 | hl7-fhir-rest | bars |
| e002 | hl7-fhir-rest | bars-referral |
| e003 | ihe-xds | xds-referral |

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: 60E0B220-8136-4CA5-AE46-1D97EF59D068
X-Correlation-Id: 11C46F5F-CDEF-4865-94B2-0EE0EDCC26DA
NHSD-End-User-Organisation-ODS: A1001
```

### Response — 200 OK (1 match)

The Lambda resolves all three templates, applies the filter, and returns only e001:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "timestamp": "2026-05-06T10:00:00+00:00",
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS FHIR REST Endpoint",
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
        "header": "public",
        "managingOrganization": [
          {
            "identifier": {
              "system": "https://fhir.nhs.uk/Id/ods-organization-code",
              "value": "A1001"
            }
          }
        ]
      },
      "search": { "mode": "match" }
    }
  ]
}
```

e002 and e003 are excluded — the Lambda fetched their templates, evaluated the filter,
and dropped them before building the response.

---

## Example 2: filter by ConnectionType only

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-fhir-rest HTTP/1.1
```

### Response — 200 OK (2 matches)

Both e001 and e002 have connectionType: hl7-fhir-rest on their templates:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 2,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS FHIR REST Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS Referral Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars-referral", "display": "BaRS Referral" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

e003 is excluded — its template has connectionType: ihe-xds.

---

## Example 3: filter by PayloadType only

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&PayloadType=bars-referral HTTP/1.1
```

### Response — 200 OK (1 match)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS Referral Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars-referral", "display": "BaRS Referral" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

---

## Example 4: no filters — all Endpoints for a service

When no ConnectionType or PayloadType is specified, the Lambda still resolves all
templates to populate the response fields, but applies no filter — all Endpoints are
returned fully merged.

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
```

### Response — 200 OK (3 results)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 3,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS FHIR REST Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "BaRS Referral Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars-referral", "display": "BaRS Referral" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4/referral",
        "header": "public"
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000003",
        "status": "active",
        "Period": { "Start": "2026-01-01T00:00:00+00:00" },
        "name": "IHE XDS Referral Endpoint",
        "connectionType": { "coding": [{ "code": "ihe-xds", "display": "IHE XDS" }] },
        "payloadType": [{ "coding": [{ "code": "xds-referral", "display": "XDS Referral" }] }],
        "address": "https://xds.provider.nhs.uk/xds",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

---

## Example 5: no matching Endpoints

### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc&ConnectionType=hl7-v2-mllp HTTP/1.1
```

### Response — 200 OK (empty)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

---

## Lambda implementation notes

The following describes the internal logic the Lambda executes for each GET /Endpoint
request that includes ConnectionType and/or PayloadType filters:

1. **Resolve the service** — use _has:HealthcareService:endpoint:_id to identify the
   HealthcareService and retrieve its list of child Endpoint IDs.
2. **Fetch all child Endpoints** — retrieve each Endpoint record (which holds only status
   and period).
3. **Fetch each parent template** — for every child Endpoint, retrieve the template fields
   (connectionType, payloadType, address, name, header, etc.).
4. **Apply filters** — evaluate ConnectionType and/or PayloadType against each template.
   Discard Endpoints whose template does not match.
5. **Merge and respond** — for each passing Endpoint, merge the child's status and period
   with all fields from its parent template. Return the merged resources in a Bundle.

---

## Summary

| Filter combination | Lambda behaviour | Response |
|--------------------|-----------------|----------|
| ConnectionType + PayloadType | Resolves all templates, keeps Endpoints matching both | Bundle with 0..* fully-resolved Endpoints |
| ConnectionType only | Resolves all templates, keeps Endpoints matching connectionType | Bundle with 0..* fully-resolved Endpoints |
| PayloadType only | Resolves all templates, keeps Endpoints matching payloadType | Bundle with 0..* fully-resolved Endpoints |
| Neither | Resolves all templates, returns all | Bundle with all Endpoints fully resolved |

In all cases the consumer makes **one HTTP call**. The Lambda performs the template
fan-out internally.

---

## Why GET /HealthcareService with _include cannot filter Endpoints

### The problem in one sentence

`GET /HealthcareService?_id=...&_include=HealthcareService:endpoint` conflates two
distinct resources: a consumer is searching for a HealthcareService but filtering on
properties that belong to an Endpoint. Even if the Lambda resolved templates for the
included Endpoints, accepting `ConnectionType` and `PayloadType` as filters on a
`HealthcareService` request is not appropriate FHIR design.

### How _include works

When the server processes `_include=HealthcareService:endpoint`, it follows the references
in `HealthcareService.endpoint[]` and appends the referenced Endpoint resources directly
to the Bundle. By default this happens at the data layer — the Endpoint records are
fetched and included as-is, without template resolution.

The Lambda could be extended to intercept this and fetch each Endpoint's parent Template,
replacing the bare records with fully-resolved ones before returning the response. The
contrast between the two is shown below.

#### _include: bare Endpoint (default, no Lambda interception)

```json
{
  "resource": {
    "resourceType": "Endpoint",
    "id": "e1a2b3c4-0000-0000-0000-000000000001",
    "status": "active",
    "period": { "start": "2026-01-01T00:00:00+00:00" }
  },
  "search": { "mode": "include" }
}
```

#### _include: fully-resolved Endpoint (Lambda intercepts and merges template)

```json
{
  "resource": {
    "resourceType": "Endpoint",
    "id": "e1a2b3c4-0000-0000-0000-000000000001",
    "status": "active",
    "period": { "start": "2026-01-01T00:00:00+00:00" },
    "name": "BaRS FHIR REST Endpoint",
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
    "header": "public",
    "managingOrganization": {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "A1001"
      }
    }
  },
  "search": { "mode": "include" }
}
```

However, even with template resolution in place, the conflation problem remains — see
below.

### Why filtering is impossible

There are two compounding problems:

**1. connectionType and payloadType are not appropriate search parameters for a HealthcareService request.**
`GET /HealthcareService` retrieves a service — its name, category, location, and
organisational context. `connectionType` and `payloadType` are characteristics of how
you connect to an Endpoint, not properties of the service itself. Applying them as
filters on a `HealthcareService` search conflates two distinct resources and produces a
query that has no clear or consistent meaning in FHIR terms. This problem exists
regardless of whether the Lambda resolves templates or not.

**2. FHIR _include has no filter mechanism.**
There is no standard FHIR parameter that applies a filter to `_include` results. A request like:

```http
GET /HealthcareService?_id=9f2c6f12-...&_include=HealthcareService:endpoint&ConnectionType=hl7-fhir-rest
```

is not valid FHIR — `ConnectionType` is a search parameter on `Endpoint`, not on
`HealthcareService`, and `_include` does not accept per-resource filter qualifiers.

### Contrast with GET /Endpoint

`GET /Endpoint` is handled entirely by the Lambda, which:
1. Fetches the child Endpoints
2. Fetches each parent template
3. Applies the `ConnectionType`/`PayloadType` filter against the template fields
4. Merges and returns only the matching Endpoints, fully resolved

`GET /HealthcareService` with `_include` bypasses this Lambda logic — the `_include`
mechanism is handled at the infrastructure level, not by the Lambda handler.

### What this means for consumers

| Need | Use |
|------|-----|
| HealthcareService metadata + all Endpoints (no type filtering) | `GET /HealthcareService?_id=...&_include=HealthcareService:endpoint` — returns bare Endpoints (status + period only) |
| Filtered Endpoints by connectionType and/or payloadType | `GET /Endpoint?_has:HealthcareService:endpoint:_id=...&ConnectionType=X&PayloadType=Y` — Lambda resolves templates and filters server-side |
| HealthcareService metadata + filtered Endpoints | Two calls: `GET /Endpoint?_has:...&ConnectionType=X` then `GET /HealthcareService/{id}` |

