# Managing Endpoint Templates

## Overview

A multi-tenanted supplier deploys a single instance of their system that serves many
HealthcareServices simultaneously. All those services share the same technical endpoint — the
same URL, connection type, and payload type. Rather than duplicating this data across every
Endpoint record, the Endpoint Catalog uses an **Endpoint Template** as the single source of
truth.

The run/maintain team creates one Template per supplier product. That Template is then used
as the parent when individual Endpoint records are created for each HealthcareService. Child
Endpoints store only two fields of their own — `status` and `period`. Everything else
(`connectionType`, `payloadType`, `address`, `name`, `header`, `managingOrganization`) is
resolved at read time from the parent Template. This means if the supplier changes their URL
or upgrades their product, only the Template needs updating — the change is immediately
visible on every child Endpoint without any further writes.

> **Note:** A Template does not need a HealthcareService to exist before it is created. Template
> creation is a supplier onboarding step that happens independently of service configuration.
> A Template should never be directly assigned to a HealthcareService — it is an internal
> mechanism used by the EPC to create and manage child Endpoints.

---

## What is a Template?

A Template is an `Endpoint` resource whose `environmentType` is set to `staging` by the EPC
internally. This field is a back-ported FHIR R5 feature
([Endpoint.environmentType](https://build.fhir.org/endpoint-definitions.html#Endpoint.environmentType))
used to distinguish Templates from operational Endpoints. It is set by the EPC — it does not
need to be included in the request payload and is never returned in API responses.

From the consumer's perspective, a Template is a normal `Endpoint` resource. The `$template`
custom action on the API is what scopes operations to Templates only, hiding this internal
distinction from callers.

---

## Process

### Step 1 — Gather the required data

The run/maintain team collects the required information from the supplier and prepares a CSV
file. This CSV is the input to the create process — it is uploaded to an S3 bucket which
triggers the processing pipeline before Step 2 begins.

#### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the supplier organisation that will own and manage the Template | Supplier | `R778` |
| `ProductId` | Unique identifier for the supplier product ⚠️ | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |
| `Address` | The target URL of the Endpoint | Supplier | `https://myService.nhs.uk/Base/Address` |

```csv
ODSCode,ProductId,Address
R778,PinnaclePharmOutcomes-v2024.12.12,https://myService.nhs.uk/Base/Address
```

The CSV may contain multiple rows — one per supplier product. Each row is processed
independently. The file is uploaded to the designated S3 bucket by the run/maintain team
before any API calls are made.

> ⚠️ **Product Id format — under investigation:** The format of the Product Id (currently
> documented as a concatenation of product name and version, e.g.
> `PinnaclePharmOutcomes-v2024.12.12`) is under investigation and has not yet been confirmed.
> The examples in this document should be treated as illustrative only. This document will be
> updated once the format is agreed.

---

### Step 2 — Check whether the Template already exists

Before creating a Template, the processing pipeline checks that one does not already exist
for this product. The check uses `GET /Endpoint/$template` with the `ProductId` from the CSV
as the key search parameter.

#### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key — identifies the supplier product |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |
| `Address` | Not used in this step | Only needed when creating or updating the Template |

`ConnectionType` and `PayloadType` are static values known to the processing pipeline (not
in the CSV) and are included in the query to narrow the search to the correct Template type.

> **Note:** `$template` is a custom action on `GET /Endpoint` that returns only Endpoints
> that have been designated as Templates. It filters out all operational Endpoints.

#### Request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

#### Parameters

| Parameter | In | Example | Source | Notes |
|-----------|----|---------|--------|-------|
| `productId` | query | `PinnaclePharmOutcomes-v2024.12.12` | CSV `ProductId` | Matches `Endpoint.identifier` where system is `https://fhir.nhs.uk/id/product-id` |
| `ConnectionType` | query | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type\|hl7-fhir-rest` | Static | URL-encode the `\|` character |
| `PayloadType` | query | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc\|bars` | Static | URL-encode the `\|` character |
| `X-Request-Id` | header | `c1ab3fba-6bae-4ba4-b257-5a87c44d4a91` | Runtime | UUID |
| `X-Correlation-Id` | header | `9562466f-c982-4bd5-bb0e-255e9f5e6689` | Runtime | UUID |
| `NHSD-End-User-Organisation-ODS` | header | `R778` | CSV `ODSCode` | ODS code of the requesting organisation |

> **Note:** `$template` is a custom action on `GET /Endpoint` that returns only Endpoints
> that have been designated as Templates. It filters out all operational Endpoints.

#### Response — Template exists (200 OK, total: 1)

If a matching Template is found, **do not create a new one**. The existing Template id can
be used to update it via `PUT /Endpoint/{id}/$template` if needed.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "meta": {
          "lastUpdated": "2021-10-11T15:23:30+00:00",
          "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/id/product-id",
            "value": "PinnaclePharmOutcomes-v2024.12.12"
          }
        ],
        "status": "active",
        "name": "Example Template",
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
                "code": "bars",
                "display": "BaRS"
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
        "address": "https://myService.nhs.uk/Base/Address",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

#### Response — Template does not exist (200 OK, total: 0)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

Proceed to Step 3.

---

### Step 3 — Create the Template

Call `POST /Endpoint/$template` with the Template payload built from the CSV data and static
pipeline configuration. The EPC assigns the resource `id` and sets `environmentType` to
`staging` internally — neither field needs to be in the request.

> **Note:** `$template` is a custom action on `POST /Endpoint` that creates a Template without
> the caller needing to know which internal fields distinguish a Template from an operational
> Endpoint.

#### How the CSV data is used

The three columns from the Step 1 CSV map directly into the payload. All other fields are
static values set by the processing pipeline.

| CSV column | Maps to payload field | Example value |
|------------|-----------------------|---------------|
| `ODSCode` | `managingOrganization[].identifier.value` | `R778` |
| `ProductId` | `identifier[].value` | `PinnaclePharmOutcomes-v2024.12.12` |
| `Address` | `address` | `https://myService.nhs.uk/Base/Address` |

`ODSCode` is also used as the `NHSD-End-User-Organisation-ODS` request header.

#### Payload field reference

| Field | Source | Value / Notes |
|-------|--------|---------------|
| `resourceType` | Static | Always `Endpoint` |
| `meta.lastUpdated` | Runtime | Current date/time in `yyyy-MM-DDThh:mm:ss+hh:mm` format |
| `meta.profile` | Static | `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| `identifier[].system` | Static | Always `https://fhir.nhs.uk/id/product-id` |
| `identifier[].value` | **CSV `ProductId`** | e.g. `PinnaclePharmOutcomes-v2024.12.12` ⚠️ |
| `status` | Static | Always `active` |
| `name` | Static | Always `Endpoint Template` |
| `connectionType.coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` |
| `connectionType.coding[].code` | Static | `hl7-fhir-rest` |
| `connectionType.coding[].display` | Static | `HL7 FHIR` |
| `payloadType[].coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` |
| `payloadType[].coding[].code` | Static | `bars` |
| `payloadType[].coding[].display` | Static | `BaRS` |
| `managingOrganization[].identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `managingOrganization[].identifier.value` | **CSV `ODSCode`** | e.g. `R778` |
| `address` | **CSV `Address`** | e.g. `https://myService.nhs.uk/Base/Address` |
| `header` | Static | Always `public` |
| `environmentType` | EPC (internal) | Set to `staging` by the EPC — **do not include in payload** |
| `id` | EPC | Assigned by the EPC on creation — **do not include in payload** |

#### Request

```http
POST /Endpoint/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

#### Request payload

```json
{
  "resourceType": "Endpoint",
  "meta": {
    "lastUpdated": "2026-05-08T10:00:00+00:00",
    "profile": [
      "http://hl7.org/fhir/StructureDefinition/Endpoint"
    ]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PinnaclePharmOutcomes-v2024.12.12"
    }
  ],
  "status": "active",
  "name": "Endpoint Template",
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
          "code": "bars",
          "display": "BaRS"
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
  "address": "https://myService.nhs.uk/Base/Address",
  "header": "public"
}
```

#### Response — 200 OK

The EPC returns the created Template with its assigned `id`. Record this `id` — it is needed
if the Template is updated or deleted later.

```json
{
  "resourceType": "Endpoint",
  "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
  "meta": {
    "lastUpdated": "2026-05-08T10:00:00+00:00",
    "profile": [
      "http://hl7.org/fhir/StructureDefinition/Endpoint"
    ]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PinnaclePharmOutcomes-v2024.12.12"
    }
  ],
  "status": "active",
  "name": "Endpoint Template",
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
          "code": "bars",
          "display": "BaRS"
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
  "address": "https://myService.nhs.uk/Base/Address",
  "header": "public"
}
```

---

## Updating a Template

When a supplier notifies the run/maintain team of a change — such as a new URL or a product
version upgrade — the Template must be updated. Because child Endpoints resolve all fields
from the Template at read time, the change is immediately visible across all child Endpoints
without any further writes.

---

### Step 1 — Gather the required data

The run/maintain team collects the updated information from the supplier and prepares a CSV
file. This CSV is uploaded to the S3 bucket to trigger the update pipeline.

#### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the supplier organisation | Supplier | `R778` |
| `ProductId` | Unique identifier for the supplier product ⚠️ | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |
| `Address` | The new target URL of the Endpoint | Supplier | `https://myService-new.nhs.uk/Base/Address` |

```csv
ODSCode,ProductId,Address
R778,PinnaclePharmOutcomes-v2024.12.12,https://myService-new.nhs.uk/Base/Address
```

The CSV may contain multiple rows. Each row is processed independently. The file is uploaded
to the designated S3 bucket before any API calls are made.

---

### Step 2 — Locate the Template

The Template's logical `id` is not in the CSV — it must be obtained by calling
`GET /Endpoint/$template` using the `ProductId` from the CSV as the search key.

#### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |
| `Address` | Not used in this step | Only needed when building the PUT payload |

`ConnectionType` and `PayloadType` are static values known to the processing pipeline and
are included in the query to identify the correct Template.

#### Request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

#### Response — 200 OK

Returns a Bundle. Extract `entry[0].resource.id` to get the Template's logical `id` for use
in Step 3.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/id/product-id",
            "value": "PinnaclePharmOutcomes-v2024.12.12"
          }
        ],
        "status": "active",
        "name": "Endpoint Template",
        "connectionType": {
          "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }]
        },
        "payloadType": [
          { "coding": [{ "code": "bars", "display": "BaRS" }] }
        ],
        "managingOrganization": [
          { "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "R778" } }
        ],
        "address": "https://myService.nhs.uk/Base/Address",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

---

### Step 3 — Update the Template

Call `PUT /Endpoint/{id}/$template` with the full replacement payload. The `id` from Step 2
is used in the path. All fields from the existing Template are carried forward, with the
`Address` from the CSV replacing the old value.

#### How the CSV data is used

| CSV column | Maps to payload field | Notes |
|------------|-----------------------|-------|
| `ODSCode` | `managingOrganization[].identifier.value` and `NHSD-End-User-Organisation-ODS` header | |
| `ProductId` | `identifier[].value` | |
| `Address` | `address` | The updated value from the supplier |

All other payload fields are carried forward unchanged from the existing Template retrieved
in Step 2.

#### Request

```http
PUT /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: d2bc4fca-7cbf-5cb5-c368-6b98d55e5b02
X-Correlation-Id: a6673577-d093-5ce6-bf1f-366f0g6f7790
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "Endpoint",
  "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
  "meta": {
    "lastUpdated": "2026-05-08T11:00:00+00:00",
    "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PinnaclePharmOutcomes-v2024.12.12"
    }
  ],
  "status": "active",
  "name": "Endpoint Template",
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
          "code": "bars",
          "display": "BaRS"
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
  "address": "https://myService-new.nhs.uk/Base/Address",
  "header": "public"
}
```

#### Response — 200 OK

Returns the updated Template. Because child Endpoints hold only `status` and `period` of
their own and resolve all other fields from the Template at read time, the new `address` is
immediately visible on every child Endpoint — no further writes are required.

```json
{
  "resourceType": "Endpoint",
  "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
  "meta": {
    "lastUpdated": "2026-05-08T11:00:00+00:00",
    "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PinnaclePharmOutcomes-v2024.12.12"
    }
  ],
  "status": "active",
  "name": "Endpoint Template",
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
          "code": "bars",
          "display": "BaRS"
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
  "address": "https://myService-new.nhs.uk/Base/Address",
  "header": "public"
}
```

---

## Deleting a Template

There are two ways to remove a Template from active use, depending on whether the supplier
product is being temporarily withdrawn or permanently decommissioned.

| Type | Mechanism | Reversible? | Use when |
|------|-----------|-------------|----------|
| Soft delete | Set `status` to `entered-in-error` via `PUT /Endpoint/{id}/$template` | Yes — set `status` back to `active` to reinstate | Template should no longer be used but may need to be reinstated |
| Hard delete | `DELETE /Endpoint/{id}/$template` | No — permanently removes the resource | Supplier product is fully decommissioned and the Template will never be needed again |

In both cases, **a Template cannot be soft or hard deleted if it has active child Endpoints**
— the API will return a `409 Conflict`. All child Endpoints must be deactivated or deleted
first.

---

### Soft delete

A soft delete marks the Template as no longer valid without physically removing it. The
Template record is retained and can be reinstated by setting `status` back to `active` via
another `PUT`. Because child Endpoints resolve their fields from the Template at read time,
setting `status` to `entered-in-error` signals the Template is withdrawn — but it does not
automatically change the `status` of child Endpoints. Child Endpoints must be soft-deleted
first.

---

#### Step 1 — Gather the required data

The run/maintain team collects the required information from the supplier and prepares a CSV
file. This CSV is uploaded to the S3 bucket to trigger the soft delete pipeline.

##### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the supplier organisation | Supplier | `R778` |
| `ProductId` | Unique identifier for the supplier product ⚠️ | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |

```csv
ODSCode,ProductId
R778,PinnaclePharmOutcomes-v2024.12.12
```

The CSV may contain multiple rows. `Address` is not required — the existing Template values
are carried forward unchanged; only `status` is modified.

---

#### Step 2 — Locate the Template

The Template's logical `id` must be obtained by calling `GET /Endpoint/$template` using the
`ProductId` from the CSV.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

`ConnectionType` and `PayloadType` are static pipeline values included to identify the
correct Template.

##### Request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

##### Response — 200 OK

Extract `entry[0].resource.id` and retain the full resource — all fields are needed to
build the PUT payload in Step 3.

---

#### Step 3 — Soft delete the Template

Call `PUT /Endpoint/{id}/$template` with the full existing resource, changing only `status`
to `entered-in-error`. `PUT` is a full replacement so all other fields must be included
unchanged.

##### How the CSV data is used

| CSV column | Maps to | Notes |
|------------|---------|-------|
| `ODSCode` | `managingOrganization[].identifier.value` and `NHSD-End-User-Organisation-ODS` header | Carried forward from existing Template |
| `ProductId` | `identifier[].value` | Carried forward from existing Template |

> **Note:** `PUT` is a full replacement — the entire resource must be included in the payload
> even though only `status` is changing. All other field values are copied from the Template
> retrieved in Step 2.

##### Request

```http
PUT /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e3cd5gdb-8dcg-6dc6-d479-7c09e66f6c13
X-Correlation-Id: b7784688-e194-6df7-cg2g-477g1h7h8901
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "Endpoint",
  "id": "5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
  "meta": {
    "lastUpdated": "2026-05-08T12:00:00+00:00",
    "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PinnaclePharmOutcomes-v2024.12.12"
    }
  ],
  "status": "entered-in-error",
  "name": "Endpoint Template",
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
          "code": "bars",
          "display": "BaRS"
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
  "address": "https://myService.nhs.uk/Base/Address",
  "header": "public"
}
```

##### Response — 200 OK

Returns the updated Template with `status: entered-in-error`.

##### Error response — active Endpoints exist (409 Conflict)

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "conflict",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "PROXY_CONFLICT"
          }
        ]
      },
      "diagnostics": "Cannot delete Template — one or more active Endpoints are derived from this Template."
    }
  ]
}
```

---

### Hard delete

A hard delete permanently removes the Template resource from the EPC. This cannot be undone.
Use only when the supplier product is being fully decommissioned.

---

#### Step 1 — Gather the required data

The run/maintain team collects the required information from the supplier and prepares a CSV
file. This CSV is uploaded to the S3 bucket to trigger the hard delete pipeline.

##### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the supplier organisation | Supplier | `R778` |
| `ProductId` | Unique identifier for the supplier product ⚠️ | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |

```csv
ODSCode,ProductId
R778,PinnaclePharmOutcomes-v2024.12.12
```

The CSV may contain multiple rows. No payload is sent with a `DELETE` request so `Address`
is not required.

---

#### Step 2 — Locate the Template and check for active child Endpoints

Before deleting, the pipeline must obtain the Template's logical `id` and confirm no active
child Endpoints exist.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter on `GET /Endpoint/$template` | Locates the Template |
| `ProductId` | `identifier` filter on `GET /Endpoint` | Retrieves child Endpoints for the 409 check |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

##### Pre-deletion checklist

1. Locate the Template: `GET /Endpoint/$template?productId={ProductId}&ConnectionType=...&PayloadType=...` — extract `entry[0].resource.id`
2. Retrieve all child Endpoints: `GET /Endpoint?identifier=https://fhir.nhs.uk/id/product-id|{ProductId}`
3. Confirm all returned Endpoints have `status` other than `active`, or delete them individually
4. Once no active child Endpoints remain, proceed to Step 3

---

#### Step 3 — Hard delete the Template

Call `DELETE /Endpoint/{id}/$template` using the `id` obtained in Step 2. No request body
is required.

##### Request

```http
DELETE /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: f4de6hec-9edh-7ed7-e58a-8d10f77g7d24
X-Correlation-Id: c8895799-f205-7eg8-dh3h-588h2i8i9012
NHSD-End-User-Organisation-ODS: R778
```

##### Response — 200 OK

The Template is permanently removed. No response body is returned.

##### Error response — active Endpoints exist (409 Conflict)

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "conflict",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "PROXY_CONFLICT"
          }
        ]
      },
      "diagnostics": "Cannot delete Template — one or more active Endpoints are derived from this Template."
    }
  ]
}
```

---

## API operations reference

| Operation | Path | Description |
|-----------|------|-------------|
| Check for existing Template | `GET /Endpoint/$template` | Returns Templates matching the given `productId`, `ConnectionType`, and `PayloadType` |
| Create a Template | `POST /Endpoint/$template` | Creates a new Template; EPC assigns `id` and sets `environmentType` internally |
| Update a Template | `PUT /Endpoint/{id}/$template` | Replaces the Template; child Endpoints resolve all fields except `status` and `period` from the Template at read time, so changes are immediately visible across all child Endpoints |
| Soft delete a Template | `PUT /Endpoint/{id}/$template` | Set `status` to `entered-in-error`; retains the record, reversible — blocked if active child Endpoints exist |
| Hard delete a Template | `DELETE /Endpoint/{id}/$template` | Permanently removes the Template; blocked if active child Endpoints exist |
