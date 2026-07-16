# Managing Endpoints (BaRS)

## Overview

> **Scope:** This document describes the process for managing **BaRS Endpoints only** —
> specifically child Endpoints linked to BaRS Endpoint Templates (`connectionType` of
> `hl7-fhir-rest` and `payloadType` of `bars`). Other Endpoint types (if introduced in
> future) will be documented separately.

An `Endpoint` resource represents a specific technical connection for a `HealthcareService`
in the Endpoint Catalogue. Each Endpoint is a child of an **Endpoint Template** — it inherits
the shared protocol characteristics (`connectionType`, `payloadType`, `address`, `name`,
`header`, `managingOrganization`) from its parent Template and holds only two fields of its
own: `status` and `period`.

The run/maintain team creates Endpoint records when onboarding a service, and manages their
lifecycle through status changes, period adjustments, and supplier switches.

> **Note:** A BaRS Endpoint Template must exist before an Endpoint can be created. See
> [Managing Endpoint Templates (BaRS)](./manage-endpoint-template.md) for Template creation.
> A HealthcareService must also exist (or be created simultaneously) for the Endpoint to
> be discoverable. See [Managing HealthcareServices](./manage-healthcare-service.md).

---

## What is an Endpoint?

In the Endpoint Catalogue, a child Endpoint is a FHIR R4 `Endpoint` resource that carries:

| Field | Source | Purpose |
|-------|--------|---------|
| `status` | Set by the run/maintain team or supplier | Operational lifecycle state (`active`, `suspended`, `off`, etc.) |
| `period` | Set by the run/maintain team or supplier | Time-bounded validity window (`start` and/or `end`) |
| `name` | Set by the run/maintain team | Human-readable name for the Endpoint (e.g., "Shirley Pharmacy BaRS Endpoint") |
| `connectionType` | Inherited from Template | The technical protocol (e.g. `hl7-fhir-rest`) |
| `payloadType` | Inherited from Template | The message standard (e.g. `bars`) |
| `address` | Inherited from Template | The target URL |
| `header` | Inherited from Template | Visibility (`public` or `private`) |
| `managingOrganization` | Inherited from Template | The supplier organisation |

All inherited fields are resolved at read time from the parent Template. If the Template
is updated (e.g. a URL change), all child Endpoints inherit the change immediately without
any further writes.

---

## Creating an Endpoint

### Step 1 — Gather the required data

The run/maintain team collects the required information and prepares a CSV file.

#### CSV structure

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the supplier organisation that owns the Template | Supplier | `R778` |
| `ProductId` | **Mandatory** | Product ID identifying the parent Template | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |
| `ServiceId` | **Mandatory** | DoS Service ID of the HealthcareService this Endpoint serves | DoS / Commissioner | `2000099999` |
| `Name` | Optional | Human-readable name for the Endpoint | Run/maintain | `Shirley Pharmacy BaRS Endpoint` |
| `Status` | **Mandatory** | Initial status of the Endpoint | Run/maintain | `active` |
| `PeriodStart` | Optional | Start date/time for the Endpoint's validity | Run/maintain | `2026-07-01T00:00:00+00:00` |
| `PeriodEnd` | Optional | End date/time for the Endpoint's validity | Run/maintain | |

```csv
ODSCode,ProductId,ServiceId,Name,Status,PeriodStart,PeriodEnd
R778,PinnaclePharmOutcomes-v2024.12.12,2000099999,Shirley Pharmacy BaRS Endpoint,active,2026-07-01T00:00:00+00:00,
```

> **Naming convention:** `epc-endpoint-create-YYYY-MM-DDTHHmmss.csv` (e.g.,
> `epc-endpoint-create-2026-07-07T093000.csv`)

The CSV may contain multiple rows — one per Endpoint. Each row is processed independently.

> **Note:** `PeriodStart` and `PeriodEnd` are both optional. If neither is set, the
> Endpoint has no time constraint — availability is governed by `status` alone. If only
> `PeriodStart` is set, the Endpoint is open-ended (valid from the start date indefinitely).

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-endpoint-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-endpoint-create-2026-07-07T093000.csv \
  s3://epc-switch-processing-prod/incoming/endpoints/create/epc-endpoint-create-2026-07-07T093000.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/endpoints/create/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-endpoint-processor` Lambda function. The Lambda reads the CSV and processes each
> row independently, executing Steps 2a, 2b, 2c, and 3 for every row.
>
> **After processing:** The CSV file is moved from the `incoming/` folder to an
> `archive/` folder in the same S3 bucket and retained for 30 days before automatic
> deletion.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-endpoint-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-endpoint-processor` Lambda executes the following for **each row** in the
CSV:

---

#### Step 2a — Locate the parent Template

The Endpoint must reference a parent BaRS Template. The processing pipeline looks up the
Template using the `ProductId` from the CSV.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key — identifies the supplier product |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

##### Request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: a1b2c3d4-1111-2222-3333-444455556666
X-Correlation-Id: b2c3d4e5-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: R778
```

##### Response — 200 OK (Template found)

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
          "lastUpdated": "2026-05-08T10:00:00+00:00",
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
        "address": "https://myService.nhs.uk/Base/Address",
        "header": "public"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

Extract the Template `id` from `entry[0].resource.id` — in this example,
`5fce3e6a-ba37-4289-84d1-cc3ebdb992f5`. This is used as the `extension[].valueReference.reference`
value when creating the child Endpoint in Step 4.

##### Response — 200 OK (Template not found)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

If no Template is found, the Endpoint cannot be created. The pipeline records `FAILED` in
the processing report and moves to the next row. The Template must be created first — see
[Managing Endpoint Templates (BaRS)](./manage-endpoint-template.md).

##### Pipeline behaviour — Error responses

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 1` | Extract Template `id` — proceed to Step 2b | — |
| `200 OK`, `total: 0` | Template not found — do not proceed | `FAILED` — "Template not found for ProductId {ProductId}" |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

---

#### Step 2b — Check whether an Endpoint already exists

Before creating an Endpoint, the pipeline checks that one does not already exist for this
HealthcareService and Template combination with an overlapping period.

##### Request

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id={healthcareServiceId}&identifier=https://fhir.nhs.uk/id/product-id|PinnaclePharmOutcomes-v2024.12.12 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c3d4e5f6-3333-4444-5555-666677778888
X-Correlation-Id: d4e5f6g7-4444-5555-6666-777788889999
NHSD-End-User-Organisation-ODS: R778
```

##### Pipeline behaviour — No existing Endpoint (200 OK, total: 0)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

Proceed to Step 3.

##### Pipeline behaviour — Endpoint already exists (200 OK, total: 1)

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "ep-existing-0000-0000-0000-111122223333",
        "meta": {
          "lastUpdated": "2026-05-01T09:00:00+00:00",
          "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/id/product-id",
            "value": "PinnaclePharmOutcomes-v2024.12.12"
          }
        ],
        "extension": [
          {
            "url": "http://hl7.org",
            "valueReference": {
              "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
              "display": "Parent Template Endpoint"
            }
          }
        ],
        "status": "active",
        "period": {
          "start": "2026-01-01T00:00:00+00:00"
        },
        "name": "Test Endpoint",
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

If an active Endpoint already exists with an overlapping period, **do not create a new
one** — attempting to do so will result in a `409 Conflict`. The pipeline skips the row
and records `SKIPPED` in the processing report.

---

#### Step 3 — Create the Endpoint

Call `POST /Endpoint` with the Endpoint payload. The EPC assigns the resource `id` and
resolves inherited fields from the parent Template.

> **Note:** The EPC validates the parent Template reference internally when processing the
> POST request. The pipeline does not need to pre-validate — this is handled server-side.

##### How the CSV data is used

| CSV column | Maps to payload field | Notes |
|------------|-----------------------|-------|
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |
| `ProductId` | `identifier[].value` | Links to the parent Template |
| — | `extension[].valueReference.reference` | `Endpoint/{template-id}` — the Template `id` returned from the `$template` lookup in Step 2. This is not derived from the CSV; it comes from the API response in Step 2. |
| `Name` | `name` | Human-readable name. If not provided in the CSV, derive as `"Endpoint for service {ServiceId}"` |
| `Status` | `status` | Initial lifecycle state |
| `PeriodStart` | `period.start` | Optional — omit if not set |
| `PeriodEnd` | `period.end` | Optional — omit if not set |

##### Payload field reference

| Field | Source | Value / Notes |
|-------|--------|---------------|
| `resourceType` | Static | Always `Endpoint` |
| `meta.lastUpdated` | Runtime | Current date/time |
| `meta.profile` | Static | `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| `identifier[].system` | Static | `https://fhir.nhs.uk/id/product-id` |
| `identifier[].value` | **CSV `ProductId`** | Links this Endpoint to its parent Template |
| `extension[].url` | Static | `http://hl7.org` |
| `extension[].valueReference.reference` | **Step 2 output** | `Endpoint/{template-id}` — the `id` of the parent Template resolved in Step 2 |
| `extension[].valueReference.display` | Static | `Parent Template Endpoint` |
| `name` | **CSV `Name`** | Human-readable name. If not provided, derive as `"Endpoint for service {ServiceId}"` |
| `status` | **CSV `Status`** | e.g. `active` |
| `period.start` | **CSV `PeriodStart`** | Optional |
| `period.end` | **CSV `PeriodEnd`** | Optional |

> **Note:** The `extension` with URL `http://hl7.org` is how the EPC links a child
> Endpoint to its parent Template. The EPC uses this reference to resolve inherited fields
> (`connectionType`, `payloadType`, `address`, `header`, `managingOrganization`)
> from the Template at read time. Do not include these inherited fields in the payload.
> The `name` field is set directly on the child Endpoint (not inherited).

##### Request

```http
POST /Endpoint HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e5f6g7h8-5555-6666-7777-888899990000
X-Correlation-Id: f6g7h8i9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: R778
```

##### Request payload

```json
{
  "resourceType": "Endpoint",
  "meta": {
    "lastUpdated": "2026-06-18T10:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "active",
  "period": {
    "start": "2026-07-01T00:00:00+00:00"
  }
}
```

##### Response — 200 OK

The EPC returns the created Endpoint with its assigned `id` and all inherited fields
resolved from the parent Template.

```json
{
  "resourceType": "Endpoint",
  "id": "ep-a1b2c3d4-0000-0000-0000-111122223333",
  "meta": {
    "lastUpdated": "2026-06-18T10:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "active",
  "period": {
    "start": "2026-07-01T00:00:00+00:00"
  },
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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/endpoints/create/epc-endpoint-create-2026-07-07T093000-report.csv
```

#### Report CSV structure

```csv
ODSCode,ProductId,ServiceId,Name,Status,Detail
R778,PinnaclePharmOutcomes-v2024.12.12,2000099999,Shirley Pharmacy BaRS Endpoint,CREATED,
R778,PinnaclePharmOutcomes-v2024.12.12,2000088888,Anytown Pharmacy BaRS Endpoint,SKIPPED,Endpoint already exists
R778,UnknownProduct-v1.0.0,2000077777,Anytown GP BaRS Endpoint,FAILED,Template not found for ProductId UnknownProduct-v1.0.0
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `CREATED` | Endpoint created successfully | No action needed |
| `SKIPPED` | Endpoint already exists (overlapping period with same Template) | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/endpoints/create/epc-endpoint-create-2026-07-07T093000-report.csv \
  ./epc-endpoint-create-2026-07-07T093000-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/endpoints/create/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-endpoint-create-2026-07-07T093000-fixes.csv \
  s3://epc-switch-processing-prod/incoming/endpoints/create/epc-endpoint-create-2026-07-07T093000-fixes.csv
```

---

### Step 4 — Associate the Endpoint with a HealthcareService

After creation, the Endpoint must be associated with a HealthcareService to be discoverable
by consumers. Update the HealthcareService's `endpoint[]` array to include a reference to
the new Endpoint.

See [Managing HealthcareServices — Adding or removing Endpoint associations](./manage-healthcare-service.md#adding-or-removing-endpoint-associations).

---

## Updating an Endpoint

An Endpoint's `status` and `period` can be changed via `PUT /Endpoint/{id}`. Only these
two fields are mutable on a child Endpoint — all other fields are inherited from the
Template.

### Changing status

##### Request

```http
PUT /Endpoint/ep-a1b2c3d4-0000-0000-0000-111122223333 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: g7h8i9j0-7777-8888-9999-000011112222
X-Correlation-Id: h8i9j0k1-8888-9999-0000-111122223333
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "Endpoint",
  "id": "ep-a1b2c3d4-0000-0000-0000-111122223333",
  "meta": {
    "lastUpdated": "2026-06-18T14:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "suspended",
  "period": {
    "start": "2026-07-01T00:00:00+00:00"
  }
}
```

##### Response — 200 OK

Returns the updated Endpoint with the new status and all inherited fields resolved from
the parent Template.

```json
{
  "resourceType": "Endpoint",
  "id": "ep-a1b2c3d4-0000-0000-0000-111122223333",
  "meta": {
    "lastUpdated": "2026-06-18T14:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "suspended",
  "period": {
    "start": "2026-07-01T00:00:00+00:00"
  },
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

### Setting a period end date

To expire an Endpoint (e.g. ahead of a supplier switch), set `period.end`:

##### Request

```http
PUT /Endpoint/ep-a1b2c3d4-0000-0000-0000-111122223333 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: h8i9j0k1-1111-2222-3333-444455556666
X-Correlation-Id: i9j0k1l2-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resourceType": "Endpoint",
  "id": "ep-a1b2c3d4-0000-0000-0000-111122223333",
  "meta": {
    "lastUpdated": "2026-06-18T14:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "active",
  "period": {
    "start": "2026-01-01T00:00:00+00:00",
    "end": "2026-06-30T23:59:59+00:00"
  }
}
```

##### Response — 200 OK

Returns the updated Endpoint with the period end date set and all inherited fields resolved.

```json
{
  "resourceType": "Endpoint",
  "id": "ep-a1b2c3d4-0000-0000-0000-111122223333",
  "meta": {
    "lastUpdated": "2026-06-18T14:00:00+00:00",
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
  "extension": [
    {
      "url": "http://hl7.org",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
        "display": "Parent Template Endpoint"
      }
    }
  ],
  "name": "Shirley Pharmacy BaRS Endpoint",
  "status": "active",
  "period": {
    "start": "2026-01-01T00:00:00+00:00",
    "end": "2026-06-30T23:59:59+00:00"
  },
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

Once `period.end` passes, the Endpoint is no longer returned to consumers in search
results — it has expired.

---

## Deleting an Endpoint

| Type | Mechanism | Reversible? | Use when |
|------|-----------|-------------|----------|
| Soft delete | Set `status` to `entered-in-error` via `PUT /Endpoint/{id}` | Yes — set `status` back to `active` to reinstate | Endpoint should no longer be used but may need to be reinstated |
| Hard delete | `DELETE /Endpoint/{id}` | No — permanently removes the resource | Endpoint is fully decommissioned. **Requires admin access.** |

### Soft delete

Set `status` to `entered-in-error` using `PUT /Endpoint/{id}` with the full payload.
The Endpoint is retained but excluded from all consumer search results.

### Hard delete

> **Note:** Hard delete requires **admin access**. Standard operational roles cannot
> perform this action.

```http
DELETE /Endpoint/ep-a1b2c3d4-0000-0000-0000-111122223333 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: i9j0k1l2-9999-0000-1111-222233334444
X-Correlation-Id: j0k1l2m3-0000-1111-2222-333344445555
NHSD-End-User-Organisation-ODS: R778
```

> **Note:** Before hard-deleting an Endpoint, remove its reference from any
> HealthcareService `endpoint[]` arrays and any List `entry[]` arrays. Orphaned references
> will cause errors in consumer queries.

---

## Pharmacy Endpoint Switching (Supplier Switch)

The pharmacy supplier switch is the most common operational process in the Endpoint
Catalog. It occurs when a pharmacy changes its BaRS-capable system supplier — for example,
moving from Sonar to PharmOutcomes or vice versa. The run/maintain team processes these
switches daily using the Master Switch Log.

### What a supplier switch is

A supplier switch changes **which Endpoint a HealthcareService points to**. The old
supplier's Endpoint is disassociated and the new supplier's Endpoint is associated in its
place. This is an association change on the HealthcareService, not a modification of
either Endpoint.

```
Before switch:
  HealthcareService (Swan Lake Pharmacy)
    └── endpoint[] → Endpoint/ep-001 (child of Template-A, Supplier Sonar)

After switch:
  HealthcareService (Swan Lake Pharmacy)
    └── endpoint[] → Endpoint/ep-002 (child of Template-B, Supplier PharmOutcomes)
```

### Key principles

- **EP-001 (old supplier) is not deleted** — it may continue to serve other
  HealthcareServices. Its status and period are unchanged unless it is no longer needed
  anywhere.
- **EP-002 (new supplier) may already exist** — the new supplier's Endpoint is likely
  already active and serving other pharmacies. It does not need to be created fresh for
  each switch.
- **The switch is atomic** — the HealthcareService's `endpoint[]` is updated in a single
  `PUT` operation. There is no intermediate state where the pharmacy has no Endpoint.
- **Delegated authority transfers** — if Product ID-based delegation is in use, the
  HealthcareService's Product ID identifier is updated from the old supplier's to the new
  supplier's, transferring write access.

### CSV structure for supplier switches

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the pharmacy | Master Switch Log | `FH123` |
| `ServiceId` | **Mandatory** | DoS Service ID of the pharmacy service | Master Switch Log | `2000099999` |
| `OldProductId` | **Mandatory** | Product ID of the outgoing supplier | Master Switch Log | `PROD-SONAR-001` |
| `NewProductId` | **Mandatory** | Product ID of the incoming supplier | Master Switch Log | `PROD-PHARM-001` |
| `SwitchDate` | **Mandatory** | Effective date of the switch | Master Switch Log | `2026-07-01` |

```csv
ODSCode,ServiceId,OldProductId,NewProductId,SwitchDate
FH123,2000099999,PROD-SONAR-001,PROD-PHARM-001,2026-07-01
```

---

### Step 1 — Locate the HealthcareService

Find the pharmacy's HealthcareService using the DoS Service ID.

#### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: s1t2u3v4-1111-2222-3333-444455556666
X-Correlation-Id: t2u3v4w5-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: FH123
```

Extract the HealthcareService `id` and current `endpoint[]` array.

---

### Step 2 — Locate the new supplier's Endpoint

Find the Endpoint that belongs to the new supplier's Template. The new supplier's Endpoint
may already exist and be active (serving other pharmacies).

#### Request

```http
GET /Endpoint?identifier=https://fhir.nhs.uk/id/product-id|PROD-PHARM-001&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: u3v4w5x6-3333-4444-5555-666677778888
X-Correlation-Id: v4w5x6y7-4444-5555-6666-777788889999
NHSD-End-User-Organisation-ODS: FH123
```

Extract the new Endpoint `id` (e.g. `ep-002`).

> **Note:** If the new supplier does not yet have an Endpoint for this `connectionType`
> and `payloadType`, one must be created first. See [Creating an Endpoint](#creating-an-endpoint)
> above.

---

### Step 3 — Update the HealthcareService endpoint reference

Replace the old Endpoint reference with the new one in the HealthcareService's `endpoint[]`
array.

#### Request

```http
PUT /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: w5x6y7z8-5555-6666-7777-888899990000
X-Correlation-Id: x6y7z8a9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: FH123
```

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-07-01T00:00:00+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"
    ]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000099999"
    }
  ],
  "active": true,
  "name": "Swan Lake Pharmacy",
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FH123"
    }
  },
  "endpoint": [
    {
      "reference": "Endpoint/ep-002"
    }
  ]
}
```

#### Response — 200 OK

The HealthcareService now points to the new supplier's Endpoint. Consumers querying this
pharmacy will immediately receive the new supplier's connection details.

---

### Step 4 (optional) — Update delegated authority

If Product ID-based delegated authority is in use, the HealthcareService's Product ID
identifier should be updated from the old supplier's to the new supplier's. This ensures
the new supplier has write access and the old supplier loses it.

This is included in the same `PUT /HealthcareService/{id}` call in Step 3 by updating the
Product ID identifier:

```json
"identifier": [
  {
    "system": "https://fhir.nhs.uk/Id/dos-service-id",
    "value": "2000099999"
  },
  {
    "system": "https://fhir.nhs.uk/id/product-id",
    "value": "PROD-PHARM-001"
  }
]
```

The old supplier's Product ID (`PROD-SONAR-001`) is removed and the new supplier's
(`PROD-PHARM-001`) is added. This transfers delegated authority in the same atomic
operation.

---

### Supplier switch — what does NOT change

| Resource | What happens |
|----------|-------------|
| Old Endpoint (ep-001) | **Unchanged.** It remains active and may still serve other HealthcareServices. |
| Old Template (Template-A) | **Unchanged.** It continues to define the old supplier's product. |
| New Endpoint (ep-002) | **Unchanged.** It was already active (or is freshly created). |
| New Template (Template-B) | **Unchanged.** It already defines the new supplier's product. |
| HealthcareService | **Updated.** `endpoint[]` now references ep-002 instead of ep-001. Product ID identifier updated if delegation is in use. |

---

### Supplier switch — timing options

There are two approaches to scheduling a switch:

#### Immediate switch

The `PUT /HealthcareService/{id}` is executed on the switch date. The change takes effect
immediately. This is the current operational model used by the run/maintain team — switches
are processed daily from the Master Switch Log.

#### Scheduled switch using period

If the switch needs to be pre-staged:

1. Set `period.end` on the old Endpoint (ep-001) to the switch date
2. Create a new Endpoint (ep-003) with `period.start` set to the switch date, linked to
   the new supplier's Template
3. Update the HealthcareService `endpoint[]` to reference **both** Endpoints

The EPC's period-based visibility filtering ensures that before the switch date, consumers
see ep-001; after the switch date, consumers see ep-003. No manual intervention is needed
on the switch date itself.

| Time | Consumer sees |
|------|---------------|
| Before switch date | ep-001 (old supplier) — period still valid |
| After switch date | ep-003 (new supplier) — period now valid; ep-001 expired |

> **Note:** This approach requires two Endpoints from **different Templates** to be
> associated with the same HealthcareService simultaneously. Since they have different
> parent Templates, the duplicate detection overlap check does not fire.

---

## Bulk switch operations

The run/maintain team typically processes multiple pharmacy switches in a single batch.
The CSV file contains one row per pharmacy being switched:

```csv
ODSCode,ServiceId,OldProductId,NewProductId,SwitchDate
FH123,2000099999,PROD-SONAR-001,PROD-PHARM-001,2026-07-01
FH456,2000088888,PROD-SONAR-001,PROD-PHARM-001,2026-07-01
FH789,2000077777,PROD-PHARM-001,PROD-SONAR-001,2026-07-01
```

The processing pipeline executes Steps 1–4 for each row independently. Failed rows are
reported back to the run/maintain team for manual review — they do not block other rows
from processing.

---

## Endpoint status transitions

| From | To | Use case |
|------|----|----------|
| `active` | `suspended` | Planned maintenance or temporary outage |
| `active` | `off` | Permanent decommission |
| `active` | `entered-in-error` | Soft delete — created in error |
| `suspended` | `active` | Maintenance complete — restored to service |
| `suspended` | `off` | Decision made to permanently retire during suspension |
| `off` | (no transition) | Terminal state — cannot be reactivated |
| `entered-in-error` | `active` | Reinstatement after review |

> **Note:** The `off` status is terminal — once set, the Endpoint cannot be returned to
> `active`. Use `suspended` for temporary withdrawal where reinstatement is expected.

---

## Visibility rules summary

An Endpoint is visible to consumers in search results only when ALL of the following
are true:

1. The associated `HealthcareService.active` is `true`
2. The Endpoint's own `status` is `active`
3. The parent Template's `status` is `active`
4. The current date/time is within the Endpoint's `period` (if set)

The managing organisation (supplier) always sees all their Endpoints regardless of status
or period — this allows them to manage the lifecycle.

See [Endpoint Visibility: Status and Period](./endpoint-visibility-status-and-period.md)
for full detail.

---

## API operations reference

| Operation | Path | Description |
|-----------|------|-------------|
| Locate parent Template | `GET /Endpoint/$template?productId={id}&ConnectionType={ct}&PayloadType={pt}` | Find the Template for the child Endpoint |
| Check for existing Endpoint | `GET /Endpoint?_has:HealthcareService:endpoint:_id={hsId}&identifier={productId}` | Check for duplicates before creating |
| Create an Endpoint | `POST /Endpoint` | Creates a child Endpoint linked to a Template |
| Update an Endpoint | `PUT /Endpoint/{id}` | Change `status` and/or `period` |
| Soft delete | `PUT /Endpoint/{id}` with `status: entered-in-error` | Reversible removal |
| Hard delete | `DELETE /Endpoint/{id}` | Permanent removal |
| Retrieve Endpoints for a service | `GET /Endpoint?_has:HealthcareService:endpoint:_id={hsId}` | Consumer query |

---

## Error responses

The API returns FHIR `OperationOutcome` resources for errors:

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Success — resource returned or Bundle with results |
| 400 | Bad request — invalid parameter or payload |
| 401 | Unauthorised — missing or invalid Bearer token |
| 403 | Forbidden — valid token but insufficient permissions |
| 404 | Not found — no Endpoint with the specified `id` |
| 409 | Conflict — duplicate Endpoint (overlapping period with same Template) |
| 4XX | Other client error — see OperationOutcome for details |
| 5XX | Server error — see OperationOutcome for details |

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
      "diagnostics": "An active Endpoint already exists for this HealthcareService and Template with an overlapping period"
    }
  ]
}
```

---

## Related documents

| Document | Description |
|----------|-------------|
| [Managing Endpoint Templates (BaRS)](./manage-endpoint-template.md) | Creating and managing parent BaRS Templates |
| [Managing HealthcareServices](./manage-healthcare-service.md) | Creating services and managing Endpoint associations |
| [Endpoint Visibility: Status and Period](./endpoint-visibility-status-and-period.md) | Full status/period lifecycle rules |
| [DUEC Endpoints](./duec-endpoints.md) | Multi-Endpoint and priority ordering patterns |
| [Interim Support Process Access](./interim-support-process-access.md) | How the R&M team processes switches via CSV |
| [Product ID Format](./product-id-format.md) | Product ID conventions and supplier switch implications |
