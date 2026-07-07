# Managing HealthcareServices

## Overview

A `HealthcareService` resource represents a clinical service in the Endpoint Catalogue — for
example, a pharmacy, an urgent treatment centre, or a GP practice offering a specific
pathway. Each HealthcareService can have one or more `Endpoint` resources associated with
it, describing how senders can connect to that service.

The run/maintain team creates HealthcareService records as part of service onboarding. The
HealthcareService links a clinical service identity (typically a DoS Service ID) to the
technical Endpoints that serve it. When Endpoints are created from a Template, the
HealthcareService is what binds the child Endpoint to a specific service.

> **Note:** Endpoint Templates should be created before HealthcareServices. The Template
> represents the supplier product; the HealthcareService represents the individual service
> instance. See [Managing Endpoint Templates](./manage-endpoint-template.md) for Template
> creation.

---

## What is a HealthcareService?

In the Endpoint Catalogue, a `HealthcareService` is a FHIR R4 resource that carries:

| Field | Purpose |
|-------|---------|
| `identifier` | The service identity — typically a DoS Service ID |
| `name` | Human-readable service name |
| `active` | Whether the service is currently operational |
| `providedBy` | The organisation (ODS code) that provides the service |
| `endpoint[]` | References to the Endpoint resources associated with this service |

The HealthcareService does not hold technical connection details itself — those live on the
Endpoint resources. The HealthcareService is the anchor that ties a service identity to its
Endpoints.

---

## Process

### Step 1 — Gather the required data

The run/maintain team collects the required information and prepares a CSV file.

#### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the organisation that provides this service | Supplier / Commissioner | `A1001` |
| `ServiceId` | DoS Service ID for the service | DoS / Commissioner | `2000099999` |
| `ServiceName` | Human-readable name of the service | Supplier / Commissioner | `Anytown Urgent Treatment Centre` |
| `EndpointId` | FHIR resource id of the Endpoint to associate (optional) | EPC (from Template/Endpoint creation) | `e1a2b3c4-0000-0000-0000-000000000001` |

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001
```

> **Naming convention:** `epc-healthcareservices-YYYY-MM-DD.csv` (e.g.,
> `epc-healthcareservices-2026-07-07.csv`)

> **Note:** `EndpointId` is optional. A HealthcareService can be created without any
> associated Endpoints. Endpoints can be associated later via an update.

The CSV may contain multiple rows — one per service. Each row is processed independently.

Multiple Endpoints can be associated with a single HealthcareService by including multiple
`EndpointId` values (comma-separated).

##### Example: Multiple Endpoints (comma-separated)

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002
```

This creates a single HealthcareService with two Endpoint references in its `endpoint[]`
array. The order of the IDs determines the priority order.

---

### Step 1a — Upload the CSV to the S3 processing bucket

Upload the CSV to the designated S3 bucket. This triggers the `epc-healthcareservice-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-healthcareservices-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/epc-healthcareservices-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/healthcareservices/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-healthcareservice-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently, executing Steps 2 and 3 for every row.

---

### Lambda processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-healthcareservice-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-healthcareservice-processor` Lambda executes the following for **each row** in the
CSV:

1. **Check whether the HealthcareService already exists** (Step 2 below)
2. **Create the HealthcareService** if it does not exist (Step 3 below), or **update** it
   if it already exists and the data has changed
3. **Record the outcome** in the processing report

#### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/healthcareservices/epc-healthcareservices-2026-07-07-report.csv
```

##### Report CSV structure

```csv
ODSCode,ServiceId,ServiceName,EndpointId,Status,Detail
A1001,2000099999,Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001,CREATED,
A1001,2000088888,Anytown Pharmacy,e2b3c4d5-1111-2222-3333-444455556666,SKIPPED,Already exists
A1001,2000077777,Anytown GP,,FAILED,Endpoint e3c4d5e6-0000-0000-0000-000000000001 not found
```

##### Action on results

| Status | Meaning | Action |
|--------|---------|--------|
| `CREATED` | HealthcareService created successfully | No action needed |
| `UPDATED` | Existing HealthcareService updated (e.g., Endpoint association changed) | No action needed |
| `SKIPPED` | HealthcareService already exists with matching data | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

##### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-healthcareservices-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/epc-healthcareservices-2026-07-07-fixes.csv
```

##### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/healthcareservices/epc-healthcareservices-2026-07-07-report.csv \
  ./epc-healthcareservices-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/healthcareservices/`

---

### Step 2 — Check whether the HealthcareService already exists

Before creating a HealthcareService, the processing pipeline checks that one does not
already exist for this service. The check uses `GET /HealthcareService` with the `ServiceId`
from the CSV as the search parameter.

#### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ServiceId` | `identifier` query parameter | The primary lookup key — DoS Service ID |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

#### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: a1b2c3d4-1111-2222-3333-444455556666
X-Correlation-Id: b2c3d4e5-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: A1001
```

#### Parameters

| Parameter | In | Example | Source | Notes |
|-----------|----|---------|--------|-------|
| `identifier` | query | `https://fhir.nhs.uk/Id/dos-service-id\|2000099999` | CSV `ServiceId` | URL-encode the `\|` character |
| `X-Request-Id` | header | `a1b2c3d4-1111-2222-3333-444455556666` | Runtime | UUID |
| `X-Correlation-Id` | header | `b2c3d4e5-2222-3333-4444-555566667777` | Runtime | UUID |
| `NHSD-End-User-Organisation-ODS` | header | `A1001` | CSV `ODSCode` | ODS code of the requesting organisation |

#### Pipeline behaviour — HealthcareService exists (200 OK, total: 1)

If the API returns an existing HealthcareService, the pipeline compares the CSV data against
the existing resource:

- **Data matches** (same name, same Endpoints) → the pipeline skips the row and records
  `SKIPPED` in the processing report.
- **Data differs** (e.g., different Endpoint associations or a name change) → the pipeline
  updates the resource via `PUT /HealthcareService/{id}` and records `UPDATED`.

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
        "meta": {
          "lastUpdated": "2026-05-01T09:00:00+00:00",
          "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"]
        },
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/Id/dos-service-id",
            "value": "2000099999"
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
          }
        ]
      },
      "search": { "mode": "match" }
    }
  ]
}
```

#### Pipeline behaviour — HealthcareService does not exist (200 OK, total: 0)

The pipeline proceeds to Step 3 to create the HealthcareService.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0,
  "entry": []
}
```

#### Pipeline behaviour — Error responses

If the lookup call fails, the pipeline records the row as `FAILED` in the processing report
and moves to the next row. It does **not** attempt to create or update the HealthcareService.

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 0` | Proceed to Step 3 (create) | — |
| `200 OK`, `total: 1`, data matches CSV | Skip — no changes needed | `SKIPPED` — "Already exists with matching data" |
| `200 OK`, `total: 1`, data differs from CSV | Proceed to update (`PUT`) | `UPDATED` (after successful PUT) |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

> **Note:** Validation of the `EndpointId` is also performed in this step. If an
> `EndpointId` is provided in the CSV, the Lambda calls `GET /Endpoint/{EndpointId}` to
> confirm it exists. If the Endpoint does not exist (`404`), the row is recorded as `FAILED`
> with detail "Endpoint {EndpointId} not found" — the HealthcareService is not created or
> updated without a valid Endpoint reference.

---

### Step 3 — Create the HealthcareService

Call `POST /HealthcareService` with the payload built from the CSV data. The EPC assigns
the resource `id`.

#### How the CSV data is used

| CSV column | Maps to payload field | Example value |
|------------|-----------------------|---------------|
| `ODSCode` | `providedBy.identifier.value` | `A1001` |
| `ServiceId` | `identifier[].value` | `2000099999` |
| `ServiceName` | `name` | `Anytown Urgent Treatment Centre` |
| `EndpointId` | `endpoint[].reference` | `Endpoint/e1a2b3c4-0000-0000-0000-000000000001` |

`ODSCode` is also used as the `NHSD-End-User-Organisation-ODS` request header.

#### Payload field reference

| Field | Source | Value / Notes |
|-------|--------|---------------|
| `resourceType` | Static | Always `HealthcareService` |
| `meta.lastUpdated` | Runtime | Current date/time in `yyyy-MM-DDThh:mm:ss+hh:mm` format |
| `meta.profile` | Static | `https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService` |
| `identifier[].system` | Static | Always `https://fhir.nhs.uk/Id/dos-service-id` |
| `identifier[].value` | **CSV `ServiceId`** | e.g. `2000099999` |
| `active` | Static | Always `true` on creation |
| `name` | **CSV `ServiceName`** | e.g. `Anytown Urgent Treatment Centre` |
| `providedBy.identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `providedBy.identifier.value` | **CSV `ODSCode`** | e.g. `A1001` |
| `endpoint[].reference` | **CSV `EndpointId`** | e.g. `Endpoint/e1a2b3c4-0000-0000-0000-000000000001` |
| `id` | EPC | Assigned by the EPC on creation — **do not include in payload** |

#### Request

```http
POST /HealthcareService HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c3d4e5f6-3333-4444-5555-666677778888
X-Correlation-Id: d4e5f6g7-4444-5555-6666-777788889999
NHSD-End-User-Organisation-ODS: A1001
```

#### Request payload

```json
{
  "resourceType": "HealthcareService",
  "meta": {
    "lastUpdated": "2026-06-18T10:00:00+00:00",
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
    }
  ]
}
```

#### Response — 200 OK

The EPC returns the created HealthcareService with its assigned `id`. Record this `id` — it
is needed for updates, endpoint association changes, and deletion.

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-06-18T10:00:00+00:00",
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
    }
  ]
}
```

---

## Updating a HealthcareService

When a service needs to be updated — for example, to change its name, add or remove
Endpoint associations, or change the providing organisation — the HealthcareService is
updated via `PUT /HealthcareService/{id}`.

`PUT` is a full replacement. The entire resource must be included in the payload, even if
only one field is changing.

---

### Step 1 — Gather the required data

The run/maintain team collects the updated information and prepares a CSV file.

#### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the providing organisation | Supplier / Commissioner | `A1001` |
| `ServiceId` | DoS Service ID for the service | DoS / Commissioner | `2000099999` |
| `ServiceName` | Updated human-readable name | Supplier / Commissioner | `Anytown UTC (Extended Hours)` |
| `EndpointId` | Endpoint(s) to associate (comma-separated if multiple) | EPC | `e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002` |

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown UTC (Extended Hours),e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002
```

#### Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the
`epc-healthcareservice-processor` Lambda function automatically.

```bash
aws s3 cp epc-healthcareservices-update-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/epc-healthcareservices-update-2026-07-07.csv
```

The Lambda detects that the HealthcareService already exists (via the `ServiceId` lookup)
and performs a `PUT /HealthcareService/{id}` with the updated data. The processing report
records the outcome as `UPDATED` for each successfully modified row.

---

### Step 2 — Locate the HealthcareService

The HealthcareService's logical `id` is not in the CSV — it must be obtained by calling
`GET /HealthcareService` using the `ServiceId` from the CSV as the search key.

#### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ServiceId` | `identifier` query parameter | The primary lookup key |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

#### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e5f6g7h8-5555-6666-7777-888899990000
X-Correlation-Id: f6g7h8i9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: A1001
```

#### Response — 200 OK

Returns a Bundle. Extract `entry[0].resource.id` to get the HealthcareService's logical `id`
for use in Step 3.

---

### Step 3 — Update the HealthcareService

Call `PUT /HealthcareService/{id}` with the full replacement payload. The `id` from Step 2
is used in the path. All fields from the existing HealthcareService are carried forward,
with updated values replacing old ones.

#### How the CSV data is used

| CSV column | Maps to payload field | Notes |
|------------|-----------------------|-------|
| `ODSCode` | `providedBy.identifier.value` and `NHSD-End-User-Organisation-ODS` header | |
| `ServiceId` | `identifier[].value` | |
| `ServiceName` | `name` | The updated value |
| `EndpointId` | `endpoint[].reference` | Full set of Endpoint references — replaces the existing list |

> **Note:** `PUT` is a full replacement — the entire `endpoint[]` array must be included.
> Any Endpoint references omitted from the payload will be disassociated from the service.

#### Request

```http
PUT /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: g7h8i9j0-7777-8888-9999-000011112222
X-Correlation-Id: h8i9j0k1-8888-9999-0000-111122223333
NHSD-End-User-Organisation-ODS: A1001
```

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-06-18T11:00:00+00:00",
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
  "name": "Anytown UTC (Extended Hours)",
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
}
```

#### Response — 200 OK

Returns the updated HealthcareService.

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-06-18T11:00:00+00:00",
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
  "name": "Anytown UTC (Extended Hours)",
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
}
```

---

## Deleting a HealthcareService

There are two ways to remove a HealthcareService from active use.

| Type | Mechanism | Reversible? | Use when |
|------|-----------|-------------|----------|
| Soft delete | Set `active` to `false` via `PUT /HealthcareService/{id}` | Yes — set `active` back to `true` to reinstate | Service is being temporarily withdrawn |
| Hard delete | `DELETE /HealthcareService/{id}` | No — permanently removes the resource | Service is fully decommissioned. **Requires admin access.** |

> **Note:** Deleting a HealthcareService does not automatically delete its associated
> Endpoints. Endpoints must be managed independently. If the HealthcareService is deleted
> while Endpoints still reference it, those Endpoints become orphaned — they exist but are
> not discoverable via HealthcareService-based searches.

---

### Soft delete

A soft delete marks the HealthcareService as inactive without physically removing it. The
record is retained and can be reinstated by setting `active` back to `true`.

---

#### Step 1 — Gather the required data

##### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the providing organisation | Supplier / Commissioner | `A1001` |
| `ServiceId` | DoS Service ID for the service | DoS / Commissioner | `2000099999` |

```csv
ODSCode,ServiceId
A1001,2000099999
```

---

#### Step 2 — Locate the HealthcareService

Call `GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|{ServiceId}`
to obtain the resource `id` and full current resource.

##### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: i9j0k1l2-9999-0000-1111-222233334444
X-Correlation-Id: j0k1l2m3-0000-1111-2222-333344445555
NHSD-End-User-Organisation-ODS: A1001
```

Extract `entry[0].resource.id` and retain the full resource for the PUT payload.

---

#### Step 3 — Soft delete the HealthcareService

Call `PUT /HealthcareService/{id}` with the full existing resource, changing only `active`
to `false`.

> **Note:** `PUT` is a full replacement — the entire resource must be included in the payload
> even though only `active` is changing.

##### Request

```http
PUT /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: k1l2m3n4-1111-2222-3333-444455556666
X-Correlation-Id: l2m3n4o5-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: A1001
```

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-06-18T12:00:00+00:00",
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
  "active": false,
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
    }
  ]
}
```

##### Response — 200 OK

Returns the updated HealthcareService with `active: false`.

---

### Hard delete

A hard delete permanently removes the HealthcareService resource from the EPC. This cannot
be undone. Use only when the service is being fully decommissioned.

> **Note:** Hard delete requires **admin access**. Standard operational roles cannot
> perform this action.

---

#### Step 1 — Gather the required data

##### CSV structure

| Column | Description | Provided by | Example |
|--------|-------------|-------------|---------|
| `ODSCode` | ODS code of the providing organisation | Supplier / Commissioner | `A1001` |
| `ServiceId` | DoS Service ID for the service | DoS / Commissioner | `2000099999` |

```csv
ODSCode,ServiceId
A1001,2000099999
```

---

#### Step 2 — Locate the HealthcareService

Call `GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|{ServiceId}`
to obtain the resource `id`.

##### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: m3n4o5p6-3333-4444-5555-666677778888
X-Correlation-Id: n4o5p6q7-4444-5555-6666-777788889999
NHSD-End-User-Organisation-ODS: A1001
```

Extract `entry[0].resource.id`.

---

#### Step 3 — Hard delete the HealthcareService

Call `DELETE /HealthcareService/{id}` using the `id` obtained in Step 2. No request body
is required.

##### Request

```http
DELETE /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: o5p6q7r8-5555-6666-7777-888899990000
X-Correlation-Id: p6q7r8s9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: A1001
```

##### Response — 200 OK

The HealthcareService is permanently removed. No response body is returned.

---

## Adding or removing Endpoint associations

To add a new Endpoint to an existing HealthcareService, or remove one, use
`PUT /HealthcareService/{id}` with the updated `endpoint[]` array.

### Adding an Endpoint

Include the new Endpoint reference in the `endpoint[]` array alongside the existing ones:

```json
"endpoint": [
  { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" },
  { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000002" },
  { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000003" }
]
```

### Removing an Endpoint

Omit the Endpoint reference from the `endpoint[]` array. Because `PUT` is a full
replacement, any reference not included will be disassociated from the service:

```json
"endpoint": [
  { "reference": "Endpoint/e1a2b3c4-0000-0000-0000-000000000001" }
]
```

> **Note:** Removing an Endpoint reference from a HealthcareService does not delete the
> Endpoint resource itself. The Endpoint continues to exist and can be reassociated later.

---

## API operations reference

| Operation | Path | Description |
|-----------|------|-------------|
| Search for a HealthcareService | `GET /HealthcareService?identifier={system}\|{value}` | Find by DoS Service ID |
| Search with included Endpoints | `GET /HealthcareService?_id={id}&_include=HealthcareService:endpoint` | Returns HealthcareService + all associated Endpoints |
| Create a HealthcareService | `POST /HealthcareService` | Creates a new service record; EPC assigns `id` |
| Update a HealthcareService | `PUT /HealthcareService/{id}` | Full replacement — all fields must be included |
| Soft delete (deactivate) | `PUT /HealthcareService/{id}` with `active: false` | Reversible — set `active: true` to reinstate |
| Hard delete | `DELETE /HealthcareService/{id}` | Permanent removal |

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
| 409 | Conflict — e.g. duplicate identifier |
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
      "diagnostics": "Invalid value for parameter identifier: must include system and value"
    }
  ]
}
```

---

## Related documents

| Document | Description |
|----------|-------------|
| [Managing Endpoint Templates](./manage-endpoint-template.md) | Creating and managing the parent Templates that Endpoints inherit from |
| [Searching for Endpoint Information by Service ID](./search-endpoint-by-service-id.md) | Consumer-facing patterns for retrieving Endpoints via HealthcareService |
| [DUEC Endpoints](./duec-endpoints.md) | DUEC-specific HealthcareService and multi-Endpoint patterns |
| [Endpoint Ordering using the FHIR List Resource](./endpoint-ordering-with-list.md) | How endpoint priority is managed via the List resource |
