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

## Creating a HealthcareService

### Step 1 — Gather the required data

The run/maintain team collects the required information and prepares a CSV file.

#### CSV structure

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the organisation that provides this service | Supplier / Commissioner | `A1001` |
| `ServiceId` | **Mandatory** | Service identifier(s). May be a single value or multiple identifiers in brace-delimited format. | DoS / Commissioner | `2000099999` or `{2000099999,SVC-INT-001}` |
| `ServiceName` | Optional | Human-readable name of the service | Supplier / Commissioner | `Anytown Urgent Treatment Centre` |
| `EndpointId` | Optional | FHIR resource id of the Endpoint to associate | EPC (from Template/Endpoint creation) | `e1a2b3c4-0000-0000-0000-000000000001` |

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001
```

> **Naming convention:** `epc-healthcareservice-create-YYYY-MM-DD.csv` (e.g.,
> `epc-healthcareservice-create-2026-07-07.csv`)

> **Note:** `EndpointId` is optional. A HealthcareService can be created without any
> associated Endpoints. Endpoints can be associated later via an update.

##### Note: identifier format and system assumption

`ServiceId` can be a single identifier or multiple identifiers in brace-delimited format.
Each identifier maps to a separate entry in the FHIR `identifier[]` array on the
HealthcareService.

The system `https://fhir.nhs.uk/Id/dos-service-id` is **assumed by default** for every
identifier value, unless a different system is explicitly stated using the `system|value`
token format. This is consistent with the `system|value` pattern used elsewhere in this
document (e.g., query parameters).

**Example — single identifier (system assumed):**

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001
```

Produces:
```json
"identifier": [
  { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" }
]
```

**Example — multiple identifiers, same system assumed:**

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,{2000099999,2000088888},Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001
```

Produces:
```json
"identifier": [
  { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" },
  { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000088888" }
]
```

**Example — multiple identifiers, different systems explicitly stated:**

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,{2000099999,https://fhir.nhs.uk/Id/ods-organization-code|A1001},Anytown Urgent Treatment Centre,e1a2b3c4-0000-0000-0000-000000000001
```

Produces:
```json
"identifier": [
  { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" },
  { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "A1001" }
]
```

Any identifier value containing a `|` is treated as an explicit `system|value` pair. Any
value without a `|` is assumed to use `https://fhir.nhs.uk/Id/dos-service-id`.

The CSV may contain multiple rows — one per service. Each row is processed independently.

Multiple Endpoints can be associated with a single HealthcareService by including multiple
`EndpointId` values wrapped in braces and separated by commas: `{id1,id2,id3}`.

> ⚠️ **Proposed format — awaiting engineering confirmation:** The proposed delimiter for
> multiple `EndpointId` values is brace-wrapped comma-separated:
> `{e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002}`.
> This avoids ambiguity with the CSV column delimiter. Engineering to confirm this is
> straightforward to parse in the Lambda before implementation proceeds.

##### Example: Multiple Endpoints

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown Urgent Treatment Centre,{e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002}
```

This creates a single HealthcareService with two Endpoint references in its `endpoint[]`
array. The order of the IDs determines the priority order.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-healthcareservice-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-healthcareservice-create-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/create/epc-healthcareservice-create-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/healthcareservices/create/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-healthcareservice-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently, executing Steps 2a and 3 for every row.
>
> **After processing:** The CSV file is moved from the `incoming/` folder to an
> `archive/` folder in the same S3 bucket and retained for 30 days before automatic
> deletion.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-healthcareservice-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-healthcareservice-processor` Lambda executes the following for **each row** in the
CSV:

---

#### Step 2a — Check whether the HealthcareService already exists

Before creating a HealthcareService, the pipeline checks that one does not already exist
for this service. The check uses `GET /HealthcareService` with the `ServiceId` from the CSV
as the search parameter.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ServiceId` | `identifier` query parameter | The primary lookup key — DoS Service ID is assumed but could be be a different service, as per csv structure above |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

##### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: a1b2c3d4-1111-2222-3333-444455556666
X-Correlation-Id: b2c3d4e5-2222-3333-4444-555566667777
NHSD-End-User-Organisation-ODS: A1001
```

##### Parameters

| Parameter | In | Example | Source | Notes |
|-----------|----|---------|--------|-------|
| `identifier` | query | `https://fhir.nhs.uk/Id/dos-service-id\|2000099999` | CSV `ServiceId` | URL-encode the `\|` character |
| `X-Request-Id` | header | `a1b2c3d4-1111-2222-3333-444455556666` | Runtime | UUID |
| `X-Correlation-Id` | header | `b2c3d4e5-2222-3333-4444-555566667777` | Runtime | UUID |
| `NHSD-End-User-Organisation-ODS` | header | `A1001` | CSV `ODSCode` | ODS code of the requesting organisation |

##### Pipeline behaviour — HealthcareService already exists (200 OK, total: 1)

If the API returns an existing HealthcareService, the pipeline skips the row and records
`SKIPPED` in the processing report.

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

##### Pipeline behaviour — HealthcareService does not exist (200 OK, total: 0)

The pipeline proceeds to Step 3 (create the HealthcareService).

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
and moves to the next row. It does **not** attempt to create the HealthcareService.

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 0` | Proceed to Step 3 (create) | — |
| `200 OK`, `total: 1` | Skip — already exists | `SKIPPED` — "Already exists" |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

---

#### Step 3 — Create the HealthcareService

Call `POST /HealthcareService` with the payload built from the CSV data. The EPC assigns
the resource `id`.

> **Note:** The EPC validates EndpointId references internally when processing the POST
> request. If any `endpoint[]` reference points to a non-existent Endpoint, the API returns
> a `422 Unprocessable Entity` or `400 Bad Request` error. The pipeline does not need to
> pre-validate Endpoint references — this is handled server-side.

##### How the CSV data is used

| CSV column | Maps to payload field | Example value |
|------------|-----------------------|---------------|
| `ODSCode` | `providedBy.identifier.value` | `A1001` |
| `ServiceId` | `identifier[].value` (one entry per identifier — see below) | `2000099999` or `{2000099999,https://fhir.nhs.uk/Id/ods-organization-code\|A1001}` |
| `ServiceName` | `name` | `Anytown Urgent Treatment Centre` |
| `EndpointId` | `endpoint[].reference` | `Endpoint/e1a2b3c4-0000-0000-0000-000000000001` |

`ODSCode` is also used as the `NHSD-End-User-Organisation-ODS` request header.

> **Note:** `ServiceId` may contain a single identifier or multiple identifiers in
> brace-delimited format. Each identifier becomes a separate entry in the `identifier[]`
> array. The system `https://fhir.nhs.uk/Id/dos-service-id` is assumed for any value that
> does not explicitly state a system. A different system can be stated per identifier
> using the `system|value` token format — this means a single HealthcareService can carry
> identifiers from multiple different systems (e.g., a DoS service ID alongside an
> internal reference or a different national identifier). See
> [Note: identifier format and system assumption](#note-identifier-format-and-system-assumption)
> above for full examples.

##### Payload field reference

| Field | Source | Value / Notes |
|-------|--------|---------------|
| `resourceType` | Static | Always `HealthcareService` |
| `meta.lastUpdated` | Runtime | Current date/time in `yyyy-MM-DDThh:mm:ss+hh:mm` format |
| `meta.profile` | Static | `https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService` |
| `identifier[].system` | **CSV `ServiceId`** (assumed or explicit) | `https://fhir.nhs.uk/Id/dos-service-id` unless a different system is explicitly stated per identifier |
| `identifier[].value` | **CSV `ServiceId`** | One `identifier[]` entry is created per value in `ServiceId` — single value produces one entry, brace-delimited values produce multiple entries |
| `active` | Static | Always `true` on creation |
| `name` | **CSV `ServiceName`** | e.g. `Anytown Urgent Treatment Centre` |
| `providedBy.identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `providedBy.identifier.value` | **CSV `ODSCode`** | e.g. `A1001` |
| `endpoint[].reference` | **CSV `EndpointId`** | e.g. `Endpoint/e1a2b3c4-0000-0000-0000-000000000001` |
| `id` | EPC | Assigned by the EPC on creation — **do not include in payload** |

##### Request

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

##### Request payload — single identifier

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

##### Request payload — multiple identifiers, mixed systems

Built from `ServiceId: {2000099999,https://fhir.nhs.uk/Id/ods-organization-code|A1001}`:

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
    },
    {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "A1001"
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

##### Response — 200 OK

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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/healthcareservices/create/epc-healthcareservice-create-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ServiceId,ServiceName,EndpointId,Status,Detail
A1001,2000099999,Anytown Urgent Treatment Centre,{e1a2b3c4-0000-0000-0000-000000000001},CREATED,
A1001,2000088888,Anytown Pharmacy,{e2b3c4d5-1111-2222-3333-444455556666},SKIPPED,Already exists
A1001,2000077777,Anytown GP,{e3c4d5e6-0000-0000-0000-000000000001},FAILED,Endpoint e3c4d5e6-0000-0000-0000-000000000001 not found
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `CREATED` | HealthcareService created successfully | No action needed |
| `SKIPPED` | HealthcareService already exists | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/healthcareservices/create/epc-healthcareservice-create-2026-07-07-report.csv \
  ./epc-healthcareservice-create-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/healthcareservices/create/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-healthcareservice-create-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/create/epc-healthcareservice-create-2026-07-07-fixes.csv
```

---

## Updating a HealthcareService

When a service needs to be updated — for example, to change its name, add or remove
Endpoint associations, or change the providing organisation — the HealthcareService is
updated via `PUT /HealthcareService/{id}`.

`PUT` is a full replacement. The entire resource must be included in the payload, even if
only one field is changing. Any Endpoint references omitted from the `endpoint[]` array
will be disassociated from the service.

> **Note:** Removing an Endpoint reference from a HealthcareService does not delete the
> Endpoint resource itself. The Endpoint continues to exist and can be reassociated later.

---

### Step 1 — Gather the required data

The run/maintain team collects the updated information and prepares a CSV file.

#### CSV structure

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the providing organisation | Supplier / Commissioner | `A1001` |
| `ServiceId` | **Mandatory** | Service identifier(s) used to locate the HealthcareService. See [identifier format](#note-identifier-format-and-system-assumption) for single/multiple/system rules. | DoS / Commissioner | `2000099999` or `{2000099999,SVC-INT-001}` |
| `ServiceName` | Optional | Updated human-readable name | Supplier / Commissioner | `Anytown UTC (Extended Hours)` |
| `EndpointId` | Optional | Full set of Endpoint(s) to associate | EPC | `{e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002}` |

```csv
ODSCode,ServiceId,ServiceName,EndpointId
A1001,2000099999,Anytown UTC (Extended Hours),{e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002}
```

> **Naming convention:** `epc-healthcareservice-update-YYYY-MM-DD.csv` (e.g.,
> `epc-healthcareservice-update-2026-07-07.csv`)

The CSV may contain multiple rows — one per service. Each row is processed independently.

> **Note:** The rules for multiple `ServiceId` values and multiple `EndpointId` values
> (brace-delimited format, system assumptions, and engineering confirmation status) are
> the same as for creating a HealthcareService. See
> [Note: identifier format and system assumption](#note-identifier-format-and-system-assumption)
> and the [multiple Endpoints note](#creating-a-healthcareservice) in the Create section
> for full details and examples.

> **Note:** The `endpoint[]` array in the PUT payload represents the **complete** set of
> Endpoint associations. To add a new Endpoint, include it alongside the existing ones.
> To remove an Endpoint, omit it from the list.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-healthcareservice-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-healthcareservice-update-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/update/epc-healthcareservice-update-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/healthcareservices/update/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-healthcareservice-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently, executing Steps 2a and 3 for every row.
>
> **After processing:** The CSV file is moved from the `incoming/` folder to an
> `archive/` folder in the same S3 bucket and retained for 30 days before automatic
> deletion.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-healthcareservice-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-healthcareservice-processor` Lambda executes the following for **each row** in the
CSV:

---

#### Step 2a — Locate the HealthcareService

The pipeline locates the existing HealthcareService using `GET /HealthcareService` with the
`ServiceId` from the CSV. If the HealthcareService is not found, the row is recorded as
`FAILED` — you cannot update a resource that does not exist.

##### How the CSV data is used

| CSV column | Used as | Notes |
|------------|---------|-------|
| `ServiceId` | `identifier` query parameter | The primary lookup key — DoS Service ID |
| `ODSCode` | `NHSD-End-User-Organisation-ODS` header | Identifies the requesting organisation |

##### Request

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000099999 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e5f6g7h8-5555-6666-7777-888899990000
X-Correlation-Id: f6g7h8i9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: A1001
```

##### Pipeline behaviour

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK`, `total: 1` | Extract `entry[0].resource.id` — proceed to Step 3 (update) | — |
| `200 OK`, `total: 0` | HealthcareService not found — do not proceed | `FAILED` — "HealthcareService not found for ServiceId {ServiceId}" |
| `401 Unauthorized` | Do not proceed | `FAILED` — "Authentication error on lookup" |
| `403 Forbidden` | Do not proceed | `FAILED` — "Authorisation denied for ODS code {ODSCode}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error on lookup after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout on lookup after 3 retries" |

---

#### Step 3 — Update the HealthcareService

Call `PUT /HealthcareService/{id}` with the full replacement payload. The `id` from Step 2a
is used in the path. All fields from the CSV are used to construct the complete resource.

> **Note:** `PUT` is a full replacement — the entire resource must be included in the
> payload. Any Endpoint references omitted from the `endpoint[]` array will be
> disassociated from the service.

> **Note:** The EPC validates EndpointId references internally when processing the PUT
> request. If any `endpoint[]` reference points to a non-existent Endpoint, the API returns
> an error. The pipeline does not need to pre-validate Endpoint references — this is
> handled server-side.

##### How the CSV data is used

| CSV column | Maps to payload field | Notes |
|------------|-----------------------|-------|
| `ODSCode` | `providedBy.identifier.value` and `NHSD-End-User-Organisation-ODS` header | |
| `ServiceId` | `identifier[].value` | |
| `ServiceName` | `name` | The updated value |
| `EndpointId` | `endpoint[].reference` | Full set of Endpoint references — replaces the existing list |

##### Request

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

##### Request payload

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

##### Response — 200 OK

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

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/healthcareservices/update/epc-healthcareservice-update-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ServiceId,ServiceName,EndpointId,Status,Detail
A1001,2000099999,Anytown UTC (Extended Hours),{e1a2b3c4-0000-0000-0000-000000000001,e1a2b3c4-0000-0000-0000-000000000002},UPDATED,
A1001,2000077777,Anytown GP,,FAILED,HealthcareService not found for ServiceId 2000077777
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `UPDATED` | HealthcareService updated successfully | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/healthcareservices/update/epc-healthcareservice-update-2026-07-07-report.csv \
  ./epc-healthcareservice-update-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/healthcareservices/update/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-healthcareservice-update-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/update/epc-healthcareservice-update-2026-07-07-fixes.csv
```

---

## Deleting a HealthcareService

There are two ways to remove a HealthcareService from active use:

| Type | Mechanism | Reversible? | Use when |
|------|-----------|-------------|----------|
| Soft delete | Set `active` to `false` via `PUT /HealthcareService/{id}` | Yes — set `active` back to `true` to reinstate | Service is being temporarily withdrawn |
| Hard delete | `DELETE /HealthcareService/{id}` | No — permanently removes the resource | Service is fully decommissioned. **Requires admin access.** |

> **Note:** Deleting a HealthcareService does not automatically delete its associated
> Endpoints. Endpoints must be managed independently. If the HealthcareService is deleted
> while Endpoints still reference it, those Endpoints become orphaned — they exist but are
> not discoverable via HealthcareService-based searches.

The delete type is controlled by a `DeleteType` column in the CSV.

---

### Step 1 — Gather the required data

The run/maintain team collects the required information and prepares a CSV file.

#### CSV structure

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the providing organisation | Supplier / Commissioner | `A1001` |
| `ServiceId` | **Mandatory** | Service identifier(s) used to locate the HealthcareService. See [identifier format](#note-identifier-format-and-system-assumption) for single/multiple/system rules. | DoS / Commissioner | `2000099999` or `{2000099999,SVC-INT-001}` |
| `DeleteType` | Optional | Type of deletion: `soft` or `hard` (defaults to `soft` if omitted) | Run/maintain team | `soft` |

```csv
ODSCode,ServiceId,DeleteType
A1001,2000099999,soft
```

> **Naming convention:** `epc-healthcareservice-delete-YYYY-MM-DD.csv` (e.g.,
> `epc-healthcareservice-delete-2026-07-07.csv`)

The CSV may contain multiple rows — one per service. Each row is processed independently.

> **Note:** Hard delete requires **admin access**. If the processing credentials do not have
> admin permissions, rows with `DeleteType: hard` will fail.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the `epc-healthcareservice-processor`
Lambda function automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-healthcareservice-delete-2026-07-07.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/delete/epc-healthcareservice-delete-2026-07-07.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/healthcareservices/delete/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-healthcareservice-processor` Lambda function. The Lambda reads the CSV and processes
> each row independently.
>
> **After processing:** The CSV file is moved from the `incoming/` folder to an
> `archive/` folder in the same S3 bucket and retained for 30 days before automatic
> deletion.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-healthcareservice-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-healthcareservice-processor` Lambda executes the following for **each row** in the
CSV:

1. Examine the `DeleteType` column to determine the deletion mechanism
2. Attempt the delete operation (soft or hard) — if the HealthcareService does not exist,
   the API returns an error and the pipeline records `FAILED`

> **Note:** There is no separate existence check. The pipeline calls the API directly —
> if the resource does not exist, `PUT` returns `404` and `DELETE` returns `404`. The
> pipeline records the row as `FAILED` with the appropriate error detail.

---

#### Step 3 — Delete the HealthcareService

The pipeline examines the `DeleteType` column to determine the deletion mechanism.

---

##### Soft delete (`DeleteType: soft`)

The pipeline calls `PUT /HealthcareService/{ServiceId}` setting `active` to `false`.
The `ServiceId` from the CSV is used as the resource identifier. If the HealthcareService
does not exist, the API returns `404` and the row is recorded as `FAILED`.

> **Note:** `PUT` is a full replacement — the entire resource must be included in the payload
> even though only `active` is changing.

###### Request

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

###### Request payload

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

###### Response — 200 OK

Returns the updated HealthcareService with `active: false`. The pipeline records
`SOFT_DELETED` in the processing report.

###### Pipeline behaviour — Soft delete errors

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK` | Soft delete successful | `SOFT_DELETED` |
| `404 Not Found` | HealthcareService does not exist | `FAILED` — "HealthcareService not found" |
| `401` / `403` | Auth error | `FAILED` — "Authentication/authorisation error" |
| `5XX` / timeout | Retry up to 3 times; if still failing, record failure | `FAILED` — "Server error after 3 retries" |

---

##### Hard delete (`DeleteType: hard`)

Call `DELETE /HealthcareService/{ServiceId}` using the `ServiceId` from the CSV as the
resource identifier. No request body is required. If the HealthcareService does not exist,
the API returns `404` and the row is recorded as `FAILED`.

> **Note:** Hard delete requires **admin access**. Standard operational roles cannot
> perform this action.

###### Request

```http
DELETE /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: o5p6q7r8-5555-6666-7777-888899990000
X-Correlation-Id: p6q7r8s9-6666-7777-8888-999900001111
NHSD-End-User-Organisation-ODS: A1001
```

###### Response — 200 OK

The HealthcareService is permanently removed. No response body is returned. The pipeline
records `DELETED` in the processing report.

###### Pipeline behaviour — Hard delete errors

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK` | Hard delete successful | `DELETED` |
| `404 Not Found` | HealthcareService does not exist | `FAILED` — "HealthcareService not found" |
| `401` / `403` | Auth error or insufficient permissions | `FAILED` — "Insufficient permissions for hard delete" |
| `5XX` / timeout | Retry up to 3 times; if still failing, record failure | `FAILED` — "Server error after 3 retries" |

---

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/healthcareservices/delete/epc-healthcareservice-delete-2026-07-07-report.csv
```

#### Report CSV structure

```csv
ODSCode,ServiceId,DeleteType,Status,Detail
A1001,2000099999,soft,SOFT_DELETED,
A1001,2000088888,hard,DELETED,
A1001,2000077777,hard,FAILED,Insufficient permissions for hard delete
```

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `DELETED` | HealthcareService permanently removed (hard delete) | No action needed |
| `SOFT_DELETED` | HealthcareService deactivated (`active: false`) | No action needed — reinstate by updating `active` to `true` |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/healthcareservices/delete/epc-healthcareservice-delete-2026-07-07-report.csv \
  ./epc-healthcareservice-delete-2026-07-07-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/healthcareservices/delete/`

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-healthcareservice-delete-2026-07-07-fixes.csv \
  s3://epc-switch-processing-prod/incoming/healthcareservices/delete/epc-healthcareservice-delete-2026-07-07-fixes.csv
```

---

## API operations reference

| Operation | Path | Description |
|-----------|------|-------------|
| Search for a HealthcareService | `GET /HealthcareService?identifier={system}\|{value}` | Find by DoS Service ID |
| Search with included Endpoints | `GET /HealthcareService?_id={id}&_include=HealthcareService:endpoint` | Returns HealthcareService + all associated Endpoints |
| Create a HealthcareService | `POST /HealthcareService` | Creates a new service record; EPC assigns `id` |
| Update a HealthcareService | `PUT /HealthcareService/{id}` | Full replacement — all fields must be included |
| Soft delete (deactivate) | `PUT /HealthcareService/{id}` with `active: false` | Reversible — set `active: true` to reinstate |
| Hard delete | `DELETE /HealthcareService/{id}` | Permanent removal — requires admin access |

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
