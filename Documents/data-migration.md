# Data Migration to the Endpoint Catalog

> ⚠️ **This document is incomplete and requires updating.** The source database details —
> including the database name, table names, table structures, and field mappings from the
> existing schema to the Endpoint Catalog data model — have not yet been documented. These
> must be added to each step before the migration process can be executed. A database
> schema review should be conducted and the findings incorporated into this document before
> any migration work begins.

## Overview

This document describes the process for migrating endpoint data from the existing BaRS
endpoint database to the new Endpoint Catalog API. The existing database holds the live
service-to-endpoint configuration that currently drives the BaRS proxy routing. The
migration transfers this data into the Endpoint Catalog's FHIR R4 resource model, replacing
the existing flat configuration with structured `Template`, `Endpoint`, `HealthcareService`,
and `List` resources.

The migration is performed in a defined sequence to respect the dependencies between
resource types:

1. **Templates** — must exist before Endpoints can reference them
2. **Endpoints** — created after their parent Templates exist
3. **HealthcareServices** — must exist before Endpoints can be associated with them; creating a HealthcareService also auto-creates its associated List
4. **Validation** — the migrated data is checked against `targets.json`, the live service-to-endpoint mapping file, to confirm completeness and correctness

The Endpoint Catalog API enforces all duplicate detection rules server-side. The migration
process does not need to pre-check for duplicates — the API will return a `409 Conflict` if
a duplicate is detected, and the migration should handle this gracefully by logging the
outcome and continuing. See [Duplicate Detection in the Endpoint Catalog](./duplicate-detection.md)
for the full rules.

A **migration log** is maintained throughout all steps. Every resource processed — whether
successfully created, skipped, or failed — must be recorded. The log is the primary
artefact for validating the migration, tracking progress, and investigating any issues. The
log structure and usage are defined in the [Migration Log](#migration-log) section below.

> **Note:** The migration should be run in a non-production environment first to validate
> the data mapping and identify any anomalies before running against production.
> the data mapping and identify any anomalies before running against production.

---

## Migration log

A single migration log covers all steps. Every resource processed across all steps —
Templates, Endpoints, HealthcareServices, and Lists — must have an entry. The log must be
retained after the migration completes as the audit record of what was migrated.

### Log structure

| Field | Description |
|-------|-------------|
| `step` | The migration step: `1-template`, `2-endpoint`, `3-healthcareservice`, `4-list` |
| `source_id` | The id of the record in the existing database |
| `source_parent_id` | The id of the parent record in the existing database (e.g. template id for an Endpoint); blank if not applicable |
| `catalog_id` | The Endpoint Catalog resource id assigned by the EPC on creation; blank if not created |
| `catalog_parent_id` | The Endpoint Catalog id of the parent resource resolved from a previous step's log entries; blank if not applicable |
| `status` | Outcome: `created`, `skipped` (duplicate), or `failed` |
| `http_status` | The HTTP status code returned by the API |
| `diagnostics` | The `OperationOutcome.diagnostics` message if the request was rejected; blank on success |
| `timestamp` | Date/time the record was processed |

### Example log entries

```
step              | source_id | source_parent_id | catalog_id                           | catalog_parent_id                    | status  | http_status | diagnostics                          | timestamp
------------------|-----------|-----------------|--------------------------------------|--------------------------------------|---------|-------------|--------------------------------------|--------------------
1-template        | TMP-A     |                 | 5fce3e6a-ba37-4289-84d1-cc3ebdb992f5 |                                      | created | 200         |                                      | 2026-05-08T10:00:00
1-template        | TMP-B     |                 | —                                    |                                      | failed  | 400         | Missing required field: identifier   | 2026-05-08T10:00:01
2-endpoint        | EP-001    | TMP-A           | 0cb21027-a246-43e6-9c7a-35b17163eab1 | 5fce3e6a-ba37-4289-84d1-cc3ebdb992f5 | created | 200         |                                      | 2026-05-08T10:01:00
2-endpoint        | EP-002    | TMP-A           | —                                    | 5fce3e6a-ba37-4289-84d1-cc3ebdb992f5 | skipped | 409         | Duplicate — overlapping period       | 2026-05-08T10:01:01
3-healthcareservice | SVC-001 |                 | 9f2c6f12-1a6d-4d9c-a111-123456789abc |                                      | created | 200         |                                      | 2026-05-08T10:02:00
4-list            | SVC-001   |                 | f7d3e2a1-0000-0000-0000-000000000099 |                                      | created | 200         |                                      | 2026-05-08T10:02:01
```

### Log review before proceeding between steps

Before starting each new step, review the log entries from the previous step:

- All `failed` entries must be investigated and either resolved and retried, or explicitly
  accepted as out-of-scope before proceeding
- `skipped` entries (duplicates) are expected if the migration is re-run; verify they
  correspond to resources already present in the Endpoint Catalog
- The `catalog_id` values from each step feed into subsequent steps as `catalog_parent_id`
  values — any `failed` entries will result in missing parent references in later steps

---

---

## Step 1 — Migrate Templates

### Overview

Templates are the foundation of the data model. Every Endpoint in the Endpoint Catalog is a
child of a Template, which holds the shared technical characteristics (`connectionType`,
`payloadType`, `address`, `managingOrganization`) that would otherwise be duplicated across
many Endpoint records.

The first migration step extracts all unique Templates from the existing database and creates
a corresponding Template resource in the Endpoint Catalog for each one.

### Source data

From the existing database, extract all unique combinations of:

| Source field | Maps to | Notes |
|-------------|---------|-------|
| Supplier ODS code | `managingOrganization[].identifier.value` | The organisation that owns the Template |
| Product identifier | `identifier[].value` | Stored under system `https://fhir.nhs.uk/id/product-id` — see note below ⚠️ |
| Endpoint URL | `address` | The target address of the Endpoint |
| Visibility | `header` | `public` or `private` |

All existing Templates are BaRS Templates. `connectionType` and `payloadType` are therefore
fixed values for this migration — they do not need to be extracted from the source database:

| Field | Fixed value |
|-------|-------------|
| `connectionType.coding[].code` | `hl7-fhir-rest` |
| `connectionType.coding[].display` | `HL7 FHIR` |
| `connectionType.coding[].system` | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` |
| `payloadType[].coding[].code` | `bars` |
| `payloadType[].coding[].display` | `BaRS` |
| `payloadType[].coding[].system` | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` |

> ⚠️ **Product Id — construction mechanism not yet defined:** The existing database holds
> only the **supplier name**, not a versioned product identifier. The Endpoint Catalog
> requires a `productId` in the format `{ProductName}-v{Version}` (e.g.
> `PinnaclePharmOutcomes-v2024.12.12`), but the version component is not available in the
> source data. A mechanism for constructing or obtaining the full `productId` during
> migration has not yet been defined. This must be resolved before Step 1 can be executed.
> Options may include: contacting suppliers directly for their product version, deriving a
> version from a release date held elsewhere in the source system, or agreeing a placeholder
> convention for the initial migration.

A Template is unique on the combination of `productId` + `connectionType` + `payloadType`.
If the existing database contains multiple records with the same combination, only one
Template should be created — the most recent or currently active record should be used as
the source.

### Process for each Template

```
For each unique Template in the source database:
    │
    └── 1. Create the Template
            POST /Endpoint/$template
            Record the returned id for use in Step 3 (Endpoint migration)
            On 409 Conflict — Template already exists, record the id from the error response
```

> **Note:** There is no need to pre-check whether a Template already exists before calling
> `POST /Endpoint/$template`. The Endpoint Catalog API enforces the duplicate rule
> server-side and will return a `409 Conflict` if an active Template with the same
> `productId`, `connectionType`, and `payloadType` already exists. The migration process
> should handle the `409` response gracefully — retrieve the existing Template's `id` and
> continue rather than treating it as a failure. This makes the migration naturally
> idempotent without requiring a separate pre-check call for every record.

### Step 1.1 — Create the Template

Call `POST /Endpoint/$template` with the mapped payload. The EPC assigns the resource `id`
and sets `environmentType` to `staging` internally.

#### Request

```http
POST /Endpoint/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: d2bc4fca-7cbf-5cb5-c368-6b98d55e5b02
X-Correlation-Id: a6673577-d093-5ce6-bf1f-366f0g6f7790
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

#### Payload field mapping

| Payload field | Source | Notes |
|---------------|--------|-------|
| `resourceType` | Static | Always `Endpoint` |
| `meta.lastUpdated` | Runtime | Current date/time — not the source record's date |
| `meta.profile` | Static | `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| `identifier[].system` | Static | Always `https://fhir.nhs.uk/id/product-id` |
| `identifier[].value` | Source ⚠️ | Product identifier from the existing database — **only supplier name is available; version component must be obtained via a yet-to-be-defined mechanism before migration can proceed** |
| `status` | Static | Always `active` for migration |
| `name` | Static | Set to `Endpoint Template` |
| `connectionType.coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` |
| `connectionType.coding[].code` | Static | Always `hl7-fhir-rest` — all existing Templates are BaRS |
| `connectionType.coding[].display` | Static | Always `HL7 FHIR` |
| `payloadType[].coding[].system` | Static | `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` |
| `payloadType[].coding[].code` | Static | Always `bars` — all existing Templates are BaRS |
| `payloadType[].coding[].display` | Static | Always `BaRS` |
| `managingOrganization[].identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `managingOrganization[].identifier.value` | Source | Supplier ODS code from the existing database |
| `address` | Source | Endpoint URL from the existing database |
| `header` | Source | Visibility flag from the existing database (`public` or `private`) |
| `environmentType` | EPC (internal) | Set to `staging` by the EPC — do not include in payload |
| `id` | EPC | Assigned by the EPC — do not include in payload |

#### Response — 200 OK

The EPC returns the created Template with its assigned `id`. **Record this `id`** — it is
required in Step 3 when creating child Endpoints.

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

### Step 1 — Output

At the end of Step 1 the migration log should contain an entry for every Template in the
source database. Review all `failed` entries before proceeding to Step 2.

The `catalog_id` values recorded for successfully created Templates are used in Step 2 as
`catalog_parent_id` when creating child Endpoints — ensure these are present in the log
before proceeding.

### Step 1 — Error handling

| Response | Likely cause | Action |
|----------|-------------|--------|
| `409 Conflict` | An active Template with the same identity already exists | Expected during migration — retrieve the existing Template's `id` via `GET /Endpoint/$template?productId=...&ConnectionType=...&PayloadType=...&status=active`, record it as `catalog_id` with `status: skipped` in the log, and continue |
| `400 Bad Request` | Malformed payload — missing required field or invalid code | Log as `failed` with `http_status: 400` and `diagnostics`; correct the source data mapping and retry |
| `422 Unprocessable Entity` | Payload is valid but fails a business rule | Log as `failed` with `http_status: 422` and `diagnostics`; correct and retry |
| `401` / `403` | Authentication or authorisation failure | Check the bearer token and ODS code header |

---

## Next steps

The following steps will be documented in subsequent sections of this migration guide:

| Step | Description | Depends on |
|------|-------------|------------|
| Step 1 | ✅ Migrate Templates | — |
| Step 2 | ✅ Migrate Endpoints | Step 1 (Template ids) |
| Step 3 | ✅ Migrate HealthcareServices | Step 2 (Endpoint ids) |
| Step 4 | ✅ Validate migrated data | Steps 1–3 |

---

## Step 2 — Migrate Endpoints

### Overview

Each Endpoint in the existing database represents a specific service's use of a supplier
product. In the Endpoint Catalog data model, an Endpoint holds only three meaningful fields
of its own — `status`, `period`, and a reference to its parent Template. All other
characteristics (`connectionType`, `payloadType`, `address`, etc.) are resolved at read
time from the parent Template.

Step 2 extracts all Endpoints from the existing database and creates a corresponding
Endpoint resource in the Endpoint Catalog for each one, linking it to the Template created
in Step 1 via the `template` reference.

> **Prerequisite:** Step 1 must be complete and the migration log must contain `catalog_id`
> entries for all successfully created Templates before Step 2 begins.

### Source data

From the existing database, extract all Endpoints with the following fields:

| Source field | Maps to | Notes |
|-------------|---------|-------|
| Endpoint id | `source_id` in migration log | Used for traceability only — not sent to the API |
| Parent template id | `source_parent_id` in migration log | Used to look up `catalog_parent_id` from Step 1 log entries |
| Status | `status` | Map to FHIR status values — see mapping below |
| Period start | `period.start` | The date from which this Endpoint is valid |
| Period end | `period.end` | The date to which this Endpoint is valid; omit if open-ended |

Since all existing Endpoints are BaRS, `connectionType` and `payloadType` are inherited
from the parent Template and do not need to be extracted from the source.

#### Status mapping

| Existing database value | FHIR `status` value |
|------------------------|---------------------|
| Active / Live | `active` |
| Suspended | `suspended` |
| Retired / Decommissioned | `off` |
| Error / Invalid | `entered-in-error` |

### Process for each Endpoint

```
For each Endpoint in the source database:
    │
    ├── 1. Look up catalog_parent_id from Step 1 log entries using source_parent_id
    │       │
    │       └── Not found  →  Log as failed (missing template mapping); skip to next
    │
    └── 2. Create the Endpoint
            POST /Endpoint
            On 200  →  Log as created with returned catalog_id
            On 409  →  Log as skipped (duplicate — overlapping period)
            On 4XX  →  Log as failed with http_status and diagnostics
```

> **Note:** As with Templates, there is no need to pre-check for an existing Endpoint. The
> API enforces the duplicate rule server-side and returns `409 Conflict` if an Endpoint with
> the same parent Template and an overlapping period already exists. Handle `409` gracefully
> by logging it as `skipped` and continuing.

### Step 2.1 — Create the Endpoint

Call `POST /Endpoint` with the mapped payload.

#### Request

```http
POST /Endpoint HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e3cd5gdb-8dcg-6dc6-d479-7c09e66f6c13
X-Correlation-Id: b7784688-e194-6df7-cg2g-477g1h7h8901
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
  "status": "active",
  "period": {
    "start": "2024-01-01T00:00:00+00:00"
  },
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/Extension-EPC-EndpointTemplate",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5"
      }
    }
  ]
}
```

#### Payload field mapping

| Payload field | Source | Notes |
|---------------|--------|-------|
| `resourceType` | Static | Always `Endpoint` |
| `meta.lastUpdated` | Runtime | Current date/time — not the source record's date |
| `meta.profile` | Static | `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| `status` | Source | Mapped from existing database status — see status mapping above |
| `period.start` | Source | Period start date from the existing database |
| `period.end` | Source | Period end date from the existing database; omit if open-ended |
| `extension[].url` | Static | `https://fhir.nhs.uk/StructureDefinition/Extension-EPC-EndpointTemplate` |
| `extension[].valueReference.reference` | Step 1 mapping | `Endpoint/{catalog_template_id}` — the Template id from the Step 1 output |
| `id` | EPC | Assigned by the EPC — do not include in payload |

#### Response — 200 OK

The EPC returns the created Endpoint with its assigned `id`. Record this `id` as
`catalog_id` in the migration log.

```json
{
  "resourceType": "Endpoint",
  "id": "0cb21027-a246-43e6-9c7a-35b17163eab1",
  "meta": {
    "lastUpdated": "2026-05-08T10:00:00+00:00",
    "profile": [
      "http://hl7.org/fhir/StructureDefinition/Endpoint"
    ]
  },
  "status": "active",
  "period": {
    "start": "2024-01-01T00:00:00+00:00"
  },
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/Extension-EPC-EndpointTemplate",
      "valueReference": {
        "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5"
      }
    }
  ]
}
```

### Step 2 — Output

At the end of Step 2 the migration log should contain an entry for every Endpoint in the
source database. Review all `failed` entries before proceeding to Step 3:

- All `failed` entries must be investigated and resolved or explicitly accepted as
  out-of-scope before proceeding
- `skipped` entries (duplicates) are expected if the migration is re-run; verify they
  correspond to Endpoints already present in the Endpoint Catalog
- The `catalog_id` values recorded for successfully created Endpoints are used in Step 3
  when associating Endpoints with HealthcareServices

### Step 2 — Error handling

| Response | Likely cause | Action |
|----------|-------------|--------|
| `409 Conflict` | Endpoint with same Template and overlapping period already exists | Expected if re-running — log as `skipped` with `http_status: 409` and continue |
| `400 Bad Request` | Malformed payload — missing required field or invalid value | Log as `failed` with `http_status: 400` and `diagnostics`; correct the source data mapping and retry |
| `422 Unprocessable Entity` | Payload is valid but fails a business rule | Log as `failed` with `http_status: 422` and `diagnostics`; correct and retry |
| `404 Not Found` | Template id in the `extension` reference does not exist | Log as `failed` with `http_status: 404`; verify the Step 1 log entry for this Template's `catalog_id` is correct |
| `401` / `403` | Authentication or authorisation failure | Check the bearer token and ODS code header |

---

## Step 3 — Migrate HealthcareServices

### Overview

Each HealthcareService in the existing database represents a care
service. In the Endpoint Catalog, a HealthcareService holds the service identity and
references the Endpoints that can be used to contact it.

Step 3 extracts all HealthcareServices from the existing database, resolves their Endpoint
references to the Endpoint Catalog ids created in Step 2, and creates a corresponding
HealthcareService resource in the Endpoint Catalog for each one.

When a HealthcareService is created via `POST /HealthcareService`, the API automatically
creates an associated `List` resource in the same transaction. The `endpoint[]` array in
the request payload determines the initial priority order of the List — the first reference
becomes the highest priority. No separate List creation step is required.

> **Prerequisite:** Step 2 must be complete and the migration log must contain `catalog_id`
> entries for all successfully created Endpoints before Step 3 begins. Any Endpoint that
> failed in Step 2 will be absent from the HealthcareService's `endpoint[]` array — this
> must be reviewed and accepted before proceeding.

### Source data

From the existing database, extract all HealthcareServices with the following fields:

| Source field | Maps to | Notes |
|-------------|---------|-------|
| Service id | `source_id` in migration log | Used for traceability |
| DOS service id | `identifier[].value` | Stored under system `https://fhir.nhs.uk/Id/dos-service-id` |
| Service name | `name` | The human-readable name of the service |
| Organisation ODS code | `providedBy[].identifier.value` | The organisation that provides the service |
| Active flag | `active` | `true` if the service is currently active |
| Associated Endpoint ids | `endpoint[]` | The list of Endpoint ids from the existing database — each must be resolved to a `catalog_id` from the Step 2 log |

### Endpoint reference resolution

Before creating each HealthcareService, resolve each of its Endpoint references from the
existing database to the corresponding Endpoint Catalog id using the Step 2 migration log:

```
For each source_endpoint_id in the HealthcareService's endpoint list:
    │
    ├── Look up source_endpoint_id in Step 2 log entries
    │       │
    │       ├── Found, status: created  →  use catalog_id as the Endpoint reference
    │       ├── Found, status: skipped  →  use catalog_id from the existing Endpoint Catalog resource
    │       └── Found, status: failed   →  Endpoint was not migrated
    │                                       Log a warning; exclude this reference from endpoint[]
    │                                       Note in the migration log diagnostics field
    │
    └── Not found in log  →  Log a warning; exclude this reference from endpoint[]
```

> **Important:** The order of `endpoint[]` references in the `POST /HealthcareService`
> payload determines the initial priority order of the auto-created List. The order should
> reflect the priority order from the existing database. If no explicit ordering exists in
> the source data, the order of references as extracted from the database is used.

### Process for each HealthcareService

```
For each HealthcareService in the source database:
    │
    ├── 1. Resolve all Endpoint references from Step 2 log
    │       Record any unresolved references as warnings in the migration log
    │
    └── 2. Create the HealthcareService
            POST /HealthcareService
            On 200  →  Log as created with returned catalog_id
                        The API auto-creates the List — log this too (step: 3-list)
            On 409  →  Log as skipped (duplicate identifier)
            On 4XX  →  Log as failed with http_status and diagnostics
```

### Step 3.1 — Create the HealthcareService

Call `POST /HealthcareService` with the mapped payload. The API creates the HealthcareService
and automatically creates an associated `List` in the same transaction.

#### Request

```http
POST /HealthcareService HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: f4de6hec-9edh-7ed7-e58a-8d10f77g7d24
X-Correlation-Id: c8895799-f205-7eg8-dh3h-588h2i8i9012
NHSD-End-User-Organisation-ODS: R778
```

#### Request payload

```json
{
  "resourceType": "HealthcareService",
  "meta": {
    "lastUpdated": "2026-05-08T10:00:00+00:00",
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
  "providedBy": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R778"
      }
    }
  ],
  "endpoint": [
    { "reference": "Endpoint/0cb21027-a246-43e6-9c7a-35b17163eab1" },
    { "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5" }
  ]
}
```

#### Payload field mapping

| Payload field | Source | Notes |
|---------------|--------|-------|
| `resourceType` | Static | Always `HealthcareService` |
| `meta.lastUpdated` | Runtime | Current date/time — not the source record's date |
| `meta.profile` | Static | See payload above |
| `active` | Source | Active flag from the existing database |
| `name` | Source | Service name from the existing database |
| `identifier[].system` | Static | Always `https://fhir.nhs.uk/Id/dos-service-id` |
| `identifier[].value` | Source | DOS service id from the existing database |
| `providedBy[].identifier.system` | Static | Always `https://fhir.nhs.uk/Id/ods-organization-code` |
| `providedBy[].identifier.value` | Source | Organisation ODS code from the existing database |
| `endpoint[]` | Step 2 log | Resolved Endpoint Catalog ids in priority order — see Endpoint reference resolution above |
| `id` | EPC | Assigned by the EPC — do not include in payload |

#### Response — 200 OK

The EPC returns the created HealthcareService with its assigned `id`. Record this `id` as
`catalog_id` in the migration log with `step: 3-healthcareservice`.

The API also creates a `List` resource automatically. Retrieve it immediately and record it
in the migration log with `step: 3-list`:

```http
GET /List?subject=HealthcareService/{catalog_id}&status=current HTTP/1.1
```

Log the returned List `id` as `catalog_id` with `source_id` set to the same service
`source_id` and `step: 3-list`.

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-05-08T10:00:00+00:00",
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
  "providedBy": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R778"
      }
    }
  ],
  "endpoint": [
    { "reference": "Endpoint/0cb21027-a246-43e6-9c7a-35b17163eab1" },
    { "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5" }
  ]
}
```

### Step 3 — Output

At the end of Step 3 the migration log should contain entries for:

- Every HealthcareService processed (`step: 3-healthcareservice`)
- Every auto-created List (`step: 3-list`) — one per successfully created HealthcareService

Review all `failed` entries before proceeding to Step 4. Pay particular attention to
HealthcareServices where Endpoint references were excluded due to Step 2 failures — the
auto-created List for those services will be missing Endpoints and may need manual
correction.

### Step 3 — Error handling

| Response | Likely cause | Action |
|----------|-------------|--------|
| `409 Conflict` | A HealthcareService with the same `identifier.system` + `identifier.value` already exists | Expected if re-running — log as `skipped` with `http_status: 409`; retrieve the existing `catalog_id` via `GET /HealthcareService?HealthcareService.identifier=...\|...` and record it |
| `422 Unprocessable Entity` | Repeated Endpoint reference in `endpoint[]` | Log as `failed` with `http_status: 422` and `diagnostics`; check the Endpoint reference resolution step for duplicates and retry |
| `400 Bad Request` | Malformed payload — missing required field or invalid value | Log as `failed` with `http_status: 400` and `diagnostics`; correct the source data mapping and retry |
| `401` / `403` | Authentication or authorisation failure | Check the bearer token and ODS code header |

---

## Step 4 — Validate migrated data

### Overview

Step 4 validates that the data in the newly populated Endpoint Catalog is consistent with
the existing source of truth — `targets.json`, a flat JSON file containing all live
service-to-endpoint mappings from the current BaRS proxy configuration. Every mapping in
the file is checked against the Endpoint Catalog, and any discrepancy is recorded in the
validation log.

The validation is read-only — no data is created or modified. It produces a log that
confirms what has been successfully migrated, identifies gaps, and flags any data that
requires manual review before the Endpoint Catalog can be considered production-ready.

> **Prerequisite:** Steps 1–3 must be complete. The migration log from all previous steps
> must be available as it is used to cross-reference source ids with Endpoint Catalog ids
> during validation.

### Source data — targets.json

The `targets.json` file has the following structure:

```json
{
  "NHSD-Target-Identifier": {
    "tests": {
      "NHS0001": "https://internal-dev.api.service.nhs.uk/bars-mock-receiver-proxy",
      "NHS0123": "https://dev.bars.dev.api.platform.nhs.uk/mock-receiver"
    },
    "https://fhir.nhs.uk/Id/dos-service-id": {
      "55": "https://BaRS-PROD-8JY34.WASPSoftware.thirdparty.nhs.uk/api/r4",
      "101383": "https://BaRS-PROD-8JY34.WASPSoftware.thirdparty.nhs.uk/api/r4",
      "110549": "https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk",
      "2000017562": "https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/"
    }
  }
}
```

The validation uses only the entries under `NHSD-Target-Identifier["https://fhir.nhs.uk/Id/dos-service-id"]`.
The `tests` section is excluded — it contains non-production test environment mappings.

Each entry is a key-value pair:

| Key | Value | Meaning |
|-----|-------|---------|
| DOS service id (e.g. `"110549"`) | Endpoint URL (e.g. `"https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk"`) | This DOS service id should be reachable at this URL in the Endpoint Catalog |

Note that multiple DOS service ids can share the same URL — this is expected and reflects
the multi-tenanted supplier model where one Template (and therefore one URL) serves many
HealthcareServices.

### What the validation checks

For each `dosServiceId → address` entry in the file, the validation confirms:

1. A `HealthcareService` exists in the Endpoint Catalog with that DOS service id
2. The HealthcareService has at least one associated `Endpoint` whose resolved `address`
   matches the URL from the file

Since all entries are BaRS, `connectionType` is always `hl7-fhir-rest` and `payloadType`
is always `bars` — these are used as fixed filters when querying the Endpoint Catalog.

### Validation log

Validation results are recorded in the migration log using `step: 4-validation`. Each
entry in `targets.json` produces one or more log entries.

| Field | Value for validation entries |
|-------|------------------------------|
| `step` | `4-validation` |
| `source_id` | The DOS service id from `targets.json` |
| `source_parent_id` | The URL (address) from `targets.json` |
| `catalog_id` | The Endpoint Catalog HealthcareService id found (blank if not found) |
| `catalog_parent_id` | The Endpoint Catalog Endpoint id found with matching address (blank if not found) |
| `status` | `passed`, `warning`, or `failed` |
| `http_status` | HTTP status from the Endpoint Catalog API call |
| `diagnostics` | Description of the discrepancy if `warning` or `failed` |
| `timestamp` | Date/time the check was performed |

| Status | Meaning |
|--------|---------|
| `passed` | HealthcareService found and an Endpoint with the matching address exists |
| `warning` | HealthcareService found but a minor discrepancy exists — requires review |
| `failed` | HealthcareService not found, or no Endpoint with the matching address exists |

### Validation checks

For each entry `{ dosServiceId: address }` in the file:

---

#### Check 1 — HealthcareService exists

```http
GET /HealthcareService?HealthcareService.identifier=https://fhir.nhs.uk/Id/dos-service-id|{dosServiceId} HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: {uuid}
X-Correlation-Id: {uuid}
NHSD-End-User-Organisation-ODS: R778
```

| Result | Log status | Diagnostics |
|--------|-----------|-------------|
| `total: 1` | Continue to Check 2 | — |
| `total: 0` | `failed` | `HealthcareService not found for DOS id '{dosServiceId}'` |
| `total: > 1` | `failed` | `Multiple HealthcareServices found for DOS id '{dosServiceId}' — data integrity issue` |

If Check 1 fails, log as `failed` and skip Check 2 for this entry.

---

#### Check 2 — Endpoint with matching address exists

Using the HealthcareService `catalog_id` from Check 1, retrieve all associated BaRS
Endpoints and check whether any have an `address` matching the value from `targets.json`.

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id={catalog_id}&ConnectionType=hl7-fhir-rest&PayloadType=bars HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: {uuid}
X-Correlation-Id: {uuid}
NHSD-End-User-Organisation-ODS: R778
```

Compare the `address` field of each returned Endpoint against the URL from `targets.json`.

| Result | Log status | Diagnostics |
|--------|-----------|-------------|
| One Endpoint found with matching `address` | `passed` | — |
| Endpoints found but none match the `address` | `failed` | `No Endpoint found with address '{address}' for DOS id '{dosServiceId}'. Found: {list of actual addresses}` |
| `total: 0` — no Endpoints at all | `failed` | `HealthcareService '{catalog_id}' has no associated BaRS Endpoints` |

---

### Worked example

Given this entry in `targets.json`:

```json
"110549": "https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk"
```

**Check 1:**
```
GET /HealthcareService?HealthcareService.identifier=https://fhir.nhs.uk/Id/dos-service-id|110549
→ total: 1, id: 9f2c6f12-1a6d-4d9c-a111-123456789abc
```

**Check 2:**
```
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-...&ConnectionType=hl7-fhir-rest&PayloadType=bars
→ total: 1, address: "https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk"
→ address matches targets.json value ✓
```

**Log entry:**
```
step: 4-validation | source_id: 110549 | source_parent_id: https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk | catalog_id: 9f2c6f12-... | catalog_parent_id: 0cb21027-... | status: passed | http_status: 200
```

---

### Validation process flow

```
Load targets.json
Extract entries under NHSD-Target-Identifier["https://fhir.nhs.uk/Id/dos-service-id"]
Skip entries under "tests"

For each { dosServiceId → address }:
    │
    ├── Check 1: GET /HealthcareService?identifier=...|{dosServiceId}
    │       ├── total: 0 or >1  →  log as failed; skip to next entry
    │       └── total: 1        →  record catalog_id; continue to Check 2
    │
    └── Check 2: GET /Endpoint?_has:HealthcareService:endpoint:_id={catalog_id}&ConnectionType=hl7-fhir-rest&PayloadType=bars
                ├── address match found  →  log as passed
                └── no address match    →  log as failed
```

### Step 4 — Output

At the end of Step 4 the migration log should contain one entry per `targets.json` entry.
Produce a summary report:

| Metric | Description |
|--------|-------------|
| Total entries in targets.json | Count of entries under `dos-service-id` (excluding `tests`) |
| Passed | HealthcareService found and address matched |
| Failed — HealthcareService not found | DOS service id not in Endpoint Catalog |
| Failed — address mismatch | HealthcareService found but no Endpoint with matching address |
| Failed — no Endpoints | HealthcareService found but has no BaRS Endpoints |

All `failed` entries must be investigated and resolved before the Endpoint Catalog is
considered production-ready.

### Step 4 — Error handling

| Response | Likely cause | Action |
|----------|-------------|--------|
| `400 Bad Request` on any GET | Malformed query parameter | Log as `failed` with `diagnostics`; correct the query and retry |
| `401` / `403` | Authentication or authorisation failure | Check the bearer token and ODS code header |
| `500` / `503` | API unavailable | Log as `failed` with `diagnostics`; retry after a delay |
