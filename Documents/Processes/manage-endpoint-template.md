# Managing Endpoint Templates

## Overview

A multi-tenanted supplier deploys a single instance of their system that serves many
HealthcareServices simultaneously. All those services share the same technical endpoint — the
same URL, connection type, and payload type. Rather than duplicating this data across every
Endpoint record, the Endpoint Catalogue uses an **Endpoint Template** as the single source of
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

In the Endpoint Catalogue, a Template carries:

| Field | Purpose |
|-------|---------|
| `identifier` | The product identity — a unique Product Id for the supplier product |
| `name` | Human-readable name (e.g., `Endpoint Template` for BaRS; may vary for other Template types) |
| `status` | Whether the Template is currently active |
| `connectionType` | The technical protocol — `hl7-fhir-rest` for BaRS, but may differ for other Template types |
| `payloadType` | The message standard — `bars` for BaRS, but may differ for other Template types |
| `address` | The target URL of the Endpoint |
| `managingOrganization` | The supplier organisation (ODS code) that owns the Template |
| `header` | Always `public` |

---

## Creating a Template

### Step 1 — Gather the required data

The run/maintain team collects the required information from the supplier and prepares a CSV
file.

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

> **Naming convention:** `epc-endpoint-template-create-YYYY-MM-DD.csv` (e.g.,
> `epc-endpoint-template-create-2026-07-07.csv`)

> ⚠️ **Product Id format — under investigation:** The format of the Product Id (currently
> documented as a concatenation of product name and version, e.g.
> `PinnaclePharmOutcomes-v2024.12.12`) is under investigation and has not yet been confirmed.
> The examples in this document should be treated as illustrative only. This document will be
> updated once the format is agreed.

The CSV may contain multiple rows — one per supplier product. Each row is processed
independently.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-endpoint-template-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-endpoint-template-create-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/create/epc-endpoint-template-create-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/endpoint-templates/create/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-endpoint-template-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently, executing Steps 2a, 2b, and 3 for every row.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-endpoint-template-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-endpoint-template-processor` Lambda executes the following for **each row** in the
CSV:

---

#### Step 2a — Check whether the Template already exists

Before creating a Template, the pipeline checks that one does not already exist for this
product. A duplicate is defined as a Template with the same `ProductId` + `ConnectionType` +
`PayloadType` combination. The check uses `GET /Endpoint/$template` with the `ProductId`
from the CSV as the key search parameter.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key — identifies the supplier product |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |
| `Address` | Not used in this step | Only needed when creating the Template |

`ConnectionType` and `PayloadType` are static values known to the processing pipeline (not
in the CSV) and are included in the query to narrow the search to the correct Template type.

> **Note:** `$template` is a custom action on `GET /Endpoint` that returns only Endpoints
> that have been designated as Templates. It filters out all operational Endpoints.

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

##### Parameters

| Parameter | In | Example | Source | Notes |
|-----------|----|---------|--------|-------|
| `productId` | query | `PinnaclePharmOutcomes-v2024.12.12` | CSV `ProductId` | Matches `Endpoint.identifier` where system is `https://fhir.nhs.uk/id/product-id` |
| `ConnectionType` | query | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type\|hl7-fhir-rest` | Static | URL-encode the `\|` character |
| `PayloadType` | query | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc\|bars` | Static | URL-encode the `\|` character |
| `X-Request-Id` | header | `c1ab3fba-6bae-4ba4-b257-5a87c44d4a91` | Runtime | UUID |
| `X-Correlation-Id` | header | `9562466f-c982-4bd5-bb0e-255e9f5e6689` | Runtime | UUID |
| `NHSD-End-User-Organisation-ODS` | header | `R778` | CSV `ODSCode` | ODS code of the requesting organisation |

##### Pipeline behaviour — Template already exists (200 OK, total: 1)

If the API returns an existing Template, the pipeline skips the row and records
`SKIPPED` in the processing report.

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

##### Pipeline behaviour — Template does not exist (200 OK, total: 0)

The pipeline proceeds to Step 2b (validate).

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

##### Pipeline behaviour — Error responses

If the lookup call fails, the pipeline records the row as `FAILED` in the processing report
and moves to the next row. It does **not** attempt to create the Template.

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 0` | Proceed to Step 2b (validate) | — |
| `200 OK`, `total: 1` | Skip — already exists | `SKIPPED` — "Already exists" |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

---

#### Step 2b — Validate the CSV data

The pipeline validates the CSV row before proceeding to create the Template. Validation
checks include:

- `ODSCode` is a valid, non-empty ODS code
- `ProductId` is a valid, non-empty identifier
- `Address` is a valid, well-formed URL

If validation fails, the row is recorded as `FAILED` and the pipeline moves to the next row.

| Validation check | Pipeline Action | Processing Report Entry |
|------------------|-----------------|-------------------------|
| All fields valid | Proceed to Step 3 (create) | — |
| `ODSCode` empty or invalid | Do not proceed | `FAILED` — "Invalid ODSCode" |
| `ProductId` empty or invalid | Do not proceed | `FAILED` — "Invalid ProductId" |
| `Address` empty or invalid URL | Do not proceed | `FAILED` — "Invalid Address" |

---

#### Step 3 — Create the Template

Call `POST /Endpoint/$template` with the Template payload built from the CSV data and static
pipeline configuration. The EPC assigns the resource `id` and sets `environmentType` to
`staging` internally — neither field needs to be in the request.

> **Note:** `$template` is a custom action on `POST /Endpoint` that creates a Template without
> the caller needing to know which internal fields distinguish a Template from an operational
> Endpoint.

##### How the CSV data is used

The three columns from the Step 1 CSV map directly into the payload. All other fields are
static values set by the processing pipeline.

| CSV column | Maps to payload field | Example value |
|------------|-----------------------|---------------|
| `ODSCode` | `managingOrganization[].identifier.value` | `R778` |
| `ProductId` | `identifier[].value` | `PinnaclePharmOutcomes-v2024.12.12` |
| `Address` | `address` | `https://myService.nhs.uk/Base/Address` |

`ODSCode` is also used as the `NHSD-End-User-Organisation-ODS` request header.

##### Payload field reference

| Field | Source | Value / Notes |
|-------|--------|---------------|
| `resourceType` | Static | Always `Endpoint` |
| `meta.lastUpdated` | Runtime | Current date/time in `yyyy-MM-DDThh:mm:ss+hh:mm` format |
| `meta.profile` | Static | `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| `identifier[].system` | Static | Always `https://fhir.nhs.uk/id/product-id` |
| `identifier[].value` | **CSV `ProductId`** | e.g. `PinnaclePharmOutcomes-v2024.12.12` ⚠️ |
| `status` | Static | Always `active` |
| `name` | Static (BaRS) | `Endpoint Template` for BaRS Templates. May vary for other Template types. |
| `connectionType.coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` |
| `connectionType.coding[].code` | Static (BaRS) | `hl7-fhir-rest` for BaRS Templates. May vary for other Template types. |
| `connectionType.coding[].display` | Static (BaRS) | `HL7 FHIR` for BaRS Templates. May vary for other Template types. |
| `payloadType[].coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` |
| `payloadType[].coding[].code` | Static (BaRS) | `bars` for BaRS Templates. May vary for other Template types. |
| `payloadType[].coding[].display` | Static (BaRS) | `BaRS` for BaRS Templates. May vary for other Template types. |
| `managingOrganization[].identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `managingOrganization[].identifier.value` | **CSV `ODSCode`** | e.g. `R778` |
| `address` | **CSV `Address`** | e.g. `https://myService.nhs.uk/Base/Address` |
| `header` | Static | Always `public` |
| `environmentType` | EPC (internal) | Set to `staging` by the EPC — **do not include in payload** |
| `id` | EPC | Assigned by the EPC on creation — **do not include in payload** |

##### Request

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

##### Request payload

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

##### Response — 200 OK

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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/endpoint-templates/create/epc-endpoint-template-create-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ProductId,Address,Status,Detail
R778,PinnaclePharmOutcomes-v2024.12.12,https://myService.nhs.uk/Base/Address,CREATED,
R778,AnotherProduct-v1.0.0,https://another.nhs.uk/Base,SKIPPED,Already exists
R778,InvalidProduct,,FAILED,Invalid Address
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `CREATED` | Template created successfully | No action needed |
| `SKIPPED` | Template already exists (same ProductId + ConnectionType + PayloadType) | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/endpoint-templates/create/epc-endpoint-template-create-2026-07-07-report.csv \
  ./epc-endpoint-template-create-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/endpoint-templates/create/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-endpoint-template-create-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/create/epc-endpoint-template-create-2026-07-07-fixes.csv
```

---

## Updating a Template

When a supplier notifies the run/maintain team of a change — such as a new URL or a product
version upgrade — the Template must be updated. Because child Endpoints resolve all fields
from the Template at read time, the change is immediately visible across all child Endpoints
without any further writes.

`PUT` is a full replacement. The entire resource must be included in the payload, even if
only one field is changing (e.g., only `address`).

---

### Step 1 — Gather the required data

The run/maintain team collects the updated information from the supplier and prepares a CSV
file.

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

> **Naming convention:** `epc-endpoint-template-update-YYYY-MM-DD.csv` (e.g.,
> `epc-endpoint-template-update-2026-07-07.csv`)

The CSV may contain multiple rows. Each row is processed independently.

> ⚠️ **Product Id format — under investigation:** The format of the Product Id is under
> investigation and has not yet been confirmed. The examples in this document should be
> treated as illustrative only.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-endpoint-template-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-endpoint-template-update-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/update/epc-endpoint-template-update-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/endpoint-templates/update/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-endpoint-template-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently, executing Steps 2a, 2b, and 3 for every row.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-endpoint-template-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-endpoint-template-processor` Lambda executes the following for **each row** in the
CSV:

---

#### Step 2a — Locate the Template

The pipeline locates the existing Template using `GET /Endpoint/$template` with the
`ProductId` from the CSV. If the Template is not found, the row is recorded as `FAILED` —
you cannot update a resource that does not exist.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ProductId` | `productId` query parameter | The primary lookup key |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |
| `Address` | Not used in this step | Only needed when building the PUT payload |

`ConnectionType` and `PayloadType` are static values known to the processing pipeline and
are included in the query to identify the correct Template.

##### Request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e5f6g7h8-5555-6666-7777-888899990000
X-Correlation-Id: f6g7h8i9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: R778
```

##### Pipeline behaviour

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 1` | Extract `entry[0].resource.id` — proceed to Step 2b | — |
| `200 OK`, `total: 0` | Template not found — do not proceed | `FAILED` — "Template not found for ProductId {ProductId}" |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

---

#### Step 2b — Validate the CSV data

The pipeline validates the CSV row before proceeding to update the Template. Validation
checks include:

- `Address` is a valid, well-formed URL

If validation fails, the row is recorded as `FAILED` and the pipeline moves to the next row.

| Validation check | Pipeline Action | Processing Report Entry |
|------------------|-----------------|-------------------------|
| All fields valid | Proceed to Step 3 (update) | — |
| `Address` empty or invalid URL | Do not proceed | `FAILED` — "Invalid Address" |

---

#### Step 3 — Update the Template

Call `PUT /Endpoint/{id}/$template` with the full replacement payload. The `id` from Step 2a
is used in the path. All fields from the existing Template are carried forward, with the
`Address` from the CSV replacing the old value.

> **Note:** `PUT` is a full replacement — the entire resource must be included in the
> payload even though only `address` may be changing. All other field values are copied
> from the Template retrieved in Step 2a.

##### How the CSV data is used

| CSV column | Maps to payload field | Notes |
|------------|-----------------------|-------|
| `ODSCode` | `managingOrganization[].identifier.value` and `NHSD-End-User-Organisation-ODS` header | |
| `ProductId` | `identifier[].value` | |
| `Address` | `address` | The updated value from the supplier |

All other payload fields are carried forward unchanged from the existing Template retrieved
in Step 2a.

##### Request

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

##### Request payload

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

##### Response — 200 OK

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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/endpoint-templates/update/epc-endpoint-template-update-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ProductId,Address,Status,Detail
R778,PinnaclePharmOutcomes-v2024.12.12,https://myService-new.nhs.uk/Base/Address,UPDATED,
R778,UnknownProduct-v1.0.0,https://other.nhs.uk/Base,FAILED,Template not found for ProductId UnknownProduct-v1.0.0
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `UPDATED` | Template updated successfully | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/endpoint-templates/update/epc-endpoint-template-update-2026-07-07-report.csv \
  ./epc-endpoint-template-update-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/endpoint-templates/update/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-endpoint-template-update-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/update/epc-endpoint-template-update-2026-07-07-fixes.csv
```

---

## Deleting a Template

There are two ways to remove a Template from active use:

| Type | Mechanism | Reversible? | Use when |
|------|-----------|-------------|----------|
| Soft delete | Set `status` to `entered-in-error` via `PUT /Endpoint/{id}/$template` | Yes — set `status` back to `active` to reinstate | Template should no longer be used but may need to be reinstated |
| Hard delete | `DELETE /Endpoint/{id}/$template` | No — permanently removes the resource | Supplier product is fully decommissioned. **Requires admin access.** |

In both cases, **a Template cannot be soft or hard deleted if it has active child Endpoints**
— the API will return a `409 Conflict`. All child Endpoints must be deactivated or deleted
first.

> **Note:** Deleting a Template does not automatically delete its child Endpoints. Child
> Endpoints must be managed independently before the Template can be removed.

The delete type is controlled by a `DeleteType` column in the CSV.

---

### Step 1 — Gather the required data

The run/maintain team collects the required information and prepares a CSV file.

#### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the supplier organisation | Supplier | `R778` |
| `ProductId` | Unique identifier for the supplier product ⚠️ | Supplier | `PinnaclePharmOutcomes-v2024.12.12` |
| `DeleteType` | Type of deletion: `soft` or `hard` | Run/maintain team | `soft` |

```csv
ODSCode,ProductId,DeleteType
R778,PinnaclePharmOutcomes-v2024.12.12,soft
```

> **Naming convention:** `epc-endpoint-template-delete-YYYY-MM-DD.csv` (e.g.,
> `epc-endpoint-template-delete-2026-07-07.csv`)

The CSV may contain multiple rows — one per Template. Each row is processed independently.

> **Note:** Hard delete requires **admin access**. If the processing credentials do not have
> admin permissions, rows with `DeleteType: hard` will fail.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-endpoint-template-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-endpoint-template-delete-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/delete/epc-endpoint-template-delete-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/endpoint-templates/delete/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-endpoint-template-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-endpoint-template-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-endpoint-template-processor` Lambda executes the following for **each row** in the
CSV:

1. Examine the `DeleteType` column to determine the deletion mechanism
2. Attempt the delete operation (soft or hard) — if the Template does not exist,
   the API returns an error and the pipeline records `FAILED`

> **Note:** There is no separate existence check. The pipeline calls the API directly —
> if the resource does not exist, `PUT` returns `404` and `DELETE` returns `404`. The
> pipeline records the row as `FAILED` with the appropriate error detail.

---

#### Step 3 — Delete the Template

The pipeline examines the `DeleteType` column to determine the deletion mechanism.

---

##### Soft delete (`DeleteType: soft`)

The pipeline locates the Template using `GET /Endpoint/$template` with the `ProductId` from
the CSV, then calls `PUT /Endpoint/{id}/$template` setting `status` to `entered-in-error`.
If the Template does not exist, the API returns `404` and the row is recorded as `FAILED`.

> **Note:** `PUT` is a full replacement — the entire resource must be included in the payload
> even though only `status` is changing. All other field values are copied from the Template
> retrieved during the locate step.

###### Locate request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

###### Soft delete request

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

###### Request payload

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

###### Response — 200 OK

Returns the updated Template with `status: entered-in-error`. The pipeline records
`SOFT_DELETED` in the processing report.

###### Pipeline behaviour — Soft delete errors

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK` | Soft delete successful | `SOFT_DELETED` |
| `404 Not Found` | Template does not exist | `FAILED` — "Template not found" |
| `409 Conflict` | Active child Endpoints exist | `FAILED` — "Cannot delete Template — active child Endpoints exist" |
| `401` / `403` | Auth error | `FAILED` — "Authentication/authorisation error" |
| `5XX` / timeout | Retry up to 3 times; if still failing, record failure | `FAILED` — "Server error after 3 retries" |

###### Error response — active Endpoints exist (409 Conflict)

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

##### Hard delete (`DeleteType: hard`)

The pipeline locates the Template using `GET /Endpoint/$template` with the `ProductId` from
the CSV, then calls `DELETE /Endpoint/{id}/$template`. No request body is required. If the
Template does not exist, the API returns `404` and the row is recorded as `FAILED`.

> **Note:** Hard delete requires **admin access**. Standard operational roles cannot
> perform this action.

###### Locate request

```http
GET /Endpoint/$template?productId=PinnaclePharmOutcomes-v2024.12.12&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

###### Hard delete request

```http
DELETE /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: f4de6hec-9edh-7ed7-e58a-8d10f77g7d24
X-Correlation-Id: c8895799-f205-7eg8-dh3h-588h2i8i9012
NHSD-End-User-Organisation-ODS: R778
```

###### Response — 200 OK

The Template is permanently removed. No response body is returned. The pipeline records
`DELETED` in the processing report.

###### Pipeline behaviour — Hard delete errors

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK` | Hard delete successful | `DELETED` |
| `404 Not Found` | Template does not exist | `FAILED` — "Template not found" |
| `409 Conflict` | Active child Endpoints exist | `FAILED` — "Cannot delete Template — active child Endpoints exist" |
| `401` / `403` | Auth error or insufficient permissions | `FAILED` — "Insufficient permissions for hard delete" |
| `5XX` / timeout | Retry up to 3 times; if still failing, record failure | `FAILED` — "Server error after 3 retries" |

###### Error response — active Endpoints exist (409 Conflict)

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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/endpoint-templates/delete/epc-endpoint-template-delete-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ProductId,DeleteType,Status,Detail
R778,PinnaclePharmOutcomes-v2024.12.12,soft,SOFT_DELETED,
R778,AnotherProduct-v1.0.0,hard,DELETED,
R778,ActiveProduct-v2.0.0,hard,FAILED,Cannot delete Template — active child Endpoints exist
R778,UnknownProduct-v3.0.0,soft,FAILED,Template not found
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `DELETED` | Template permanently removed (hard delete) | No action needed |
| `SOFT_DELETED` | Template deactivated (`status: entered-in-error`) | No action needed — reinstate by updating `status` to `active` |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/endpoint-templates/delete/epc-endpoint-template-delete-2026-07-07-report.csv \
  ./epc-endpoint-template-delete-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/endpoint-templates/delete/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-endpoint-template-delete-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/endpoint-templates/delete/epc-endpoint-template-delete-2026-07-07-fixes.csv
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

---

## Error responses

The API returns FHIR `OperationOutcome` resources for errors:

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Success — resource returned or Bundle with results |
| 400 | Bad request — invalid parameter value or format |
| 401 | Unauthorised — missing or invalid Bearer token |
| 403 | Forbidden — valid token but insufficient permissions |
| 404 | Not found — no resource with the specified `id` |
| 409 | Conflict — e.g. active child Endpoints exist, cannot delete Template |
| 4XX | Other client error — see OperationOutcome for details |
| 5XX | Server error — see OperationOutcome for details |

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
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "INVALID_PARAMETER"
          }
        ]
      },
      "diagnostics": "Invalid value for parameter productId: must be a non-empty string"
    }
  ]
}
```

---

## Related documents

| Document | Description |
|----------|-------------|
| [Managing HealthcareServices](./manage-healthcare-service.md) | Creating and managing HealthcareService records that bind services to Endpoints |
| [Searching for Endpoint Information by Service ID](./search-endpoint-by-service-id.md) | Consumer-facing patterns for retrieving Endpoints via HealthcareService |
| [DUEC Endpoints](./duec-endpoints.md) | DUEC-specific HealthcareService and multi-Endpoint patterns |
| [Endpoint Ordering using the FHIR List Resource](./endpoint-ordering-with-list.md) | How endpoint priority is managed via the List resource |
