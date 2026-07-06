# Source Database to FHIR Mapping

## Overview

This document describes how the existing EPC database tables (DynamoDB) are queried and
their data converted to FHIR R4 resources for ingestion through the new Endpoint Catalogue
API. Each section covers one source table, its schema, the query used to extract data, and
the field-by-field mapping to the target FHIR resource.

The migration processes tables in dependency order:

1. `int_organisations` → used for reference resolution (ODS codes)
2. `int_endpoint_templates` → creates FHIR `Endpoint` resources (Templates)
3. `int_endpoints` → creates FHIR `Endpoint` resources (child Endpoints)
4. `int_healthcareservices` → creates FHIR `HealthcareService` resources

Snapshot tables (`*_snapshots`) are **not migrated** — they are audit/history records. They
may be used for validation or investigation but do not produce FHIR resources.

---

## Source Tables

### Table: `int_organisations`

This table holds the organisations (providers and suppliers) referenced by Templates,
Endpoints, and HealthcareServices. It is not directly migrated as a FHIR resource but is
used as a lookup table to resolve ODS codes from `OrganisationId` foreign keys in other
tables.

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `OrganisationId` | String (UUID) | Primary key — internal identifier |
| `Active` | Boolean | Whether the organisation is currently active |
| `AdditionalSearch` | String | Additional search context (free text) |
| `Alias` | String | Short name / alias |
| `DataStatus` | Number | Internal status flag (0 = normal) |
| `Endpoints` | List | References to associated Endpoints (JSON array) |
| `EndpointTemplates` | List | References to associated Templates (JSON array) |
| `HealthcareServices` | List | References to associated HealthcareServices (JSON array) |
| `IsSupplierOnly` | Boolean | `true` if the org is a system supplier (not a provider) |
| `LastUpdated` | String (ISO 8601) | Last modification timestamp |
| `Name` | String | Full organisation name |
| `ODSCode` | String | ODS code (e.g., `FLG23`, `RK5`) |
| `SearchField` | String | Concatenated search index |

#### Query

```sql
-- Extract all organisations for ODS code lookup
SELECT OrganisationId, ODSCode, Name, Active, IsSupplierOnly
FROM int_organisations
```

#### Usage in migration

This table is used as a **lookup** — not directly migrated. When other tables reference a
`ManagingOrganisationId` or `ProviderOrganisationId`, the migration resolves it to an ODS
code by joining to this table:

```
ManagingOrganisationId → int_organisations.OrganisationId → int_organisations.ODSCode
```

---

### Table: `int_endpoint_templates`

This table holds the supplier product templates — each one represents a supplier's system
configuration (URL, connection type, product identity).

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `TemplateId` | String (UUID) | Primary key — internal identifier |
| `ProductId` | String | Supplier product identifier (currently ODS code or short code, e.g., `AC0`, `RK5`, `ygm04`) |
| `Address` | String | The endpoint URL |
| `ConnectionSystem` | String | FHIR CodeSystem for connection type |
| `ConnectionType` | String | Connection type code (e.g., `BARS`) |
| `DataStatus` | Number | Internal status flag (0 = normal) |
| `DerivedEndpoints` | List | References to child Endpoints (JSON array) |
| `IsPrivate` | Boolean | Visibility flag (`true` = private, `false` = public) |
| `LastUpdated` | String (ISO 8601) | Last modification timestamp |
| `ManagingOrganisationId` | String (UUID) | FK to `int_organisations` |
| `Name` | String | Human-readable name of the template/supplier |
| `SearchField` | String | Concatenated search index |

#### Query

```sql
-- Extract all active templates for migration
SELECT
  t.TemplateId,
  t.ProductId,
  t.Address,
  t.ConnectionSystem,
  t.ConnectionType,
  t.IsPrivate,
  t.LastUpdated,
  t.ManagingOrganisationId,
  t.Name,
  o.ODSCode AS ManagingOrganisationODS
FROM int_endpoint_templates t
JOIN int_organisations o ON t.ManagingOrganisationId = o.OrganisationId
WHERE t.DataStatus = 0
```

#### Mapping to FHIR: `POST /Endpoint/$template`

| Source column | FHIR field | Transformation |
|---------------|-----------|----------------|
| `TemplateId` | — | Not sent to API; recorded in migration log as `source_id` |
| `ProductId` | `identifier[0].value` | ⚠️ **Requires mapping** — current values are short codes (e.g., `AC0`, `ygm04`). Must be mapped to the agreed Product ID format via the Product ID mapping table (see ACTION REQUIRED in data-migration.md) |
| — | `identifier[0].system` | Static: `https://fhir.nhs.uk/id/product-id` |
| `Address` | `address` | Direct copy. Validate it is a valid URL (some records contain `addressHere` placeholder — these must be excluded or corrected) |
| `ConnectionSystem` | `connectionType.coding[0].system` | Direct copy (e.g., `http://terminology.hl7.org/CodeSystem/endpoint-connection-type`) |
| `ConnectionType` | `connectionType.coding[0].code` | Map `BARS` → `hl7-fhir-rest` |
| — | `connectionType.coding[0].display` | Static: `HL7 FHIR` |
| — | `payloadType[0].coding[0].system` | Static: `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` |
| `ConnectionType` | `payloadType[0].coding[0].code` | Map `BARS` → `bars` |
| — | `payloadType[0].coding[0].display` | Static: `BaRS` |
| `IsPrivate` | `header` | Map: `false` → `public`, `true` → `private` |
| `ManagingOrganisationODS` | `managingOrganization[0].identifier.value` | Resolved via join to `int_organisations` |
| — | `managingOrganization[0].identifier.system` | Static: `https://fhir.nhs.uk/Id/ods-organization-code` |
| — | `status` | Static: `active` |
| — | `name` | Static: `Endpoint Template` |
| — | `meta.profile[0]` | Static: `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| — | `meta.lastUpdated` | Runtime: current migration timestamp |

#### Example: Source row → FHIR payload

**Source:**
```
TemplateId: c45831fe-f701-433d-abdf-9d91858f705b
ProductId: RK5
Address: bars-prod-rk5.nervecentre.thirdparty.nhs.uk
ConnectionSystem: http://terminology.hl7.org/CodeSystem/endpoint-connection-type
ConnectionType: BARS
IsPrivate: false
ManagingOrganisationId: 21b2d67e-0323-4064-8246-118a635d20e9
Name: Sherwood Forest Hospitals
```

**Resolved ODS code:** `RK5` (from `int_organisations` join)

**FHIR payload:**
```json
{
  "resourceType": "Endpoint",
  "meta": {
    "lastUpdated": "2026-07-07T09:00:00+00:00",
    "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
  },
  "identifier": [{
    "system": "https://fhir.nhs.uk/id/product-id",
    "value": "NervecentreBaRS-v1.0.0"
  }],
  "status": "active",
  "name": "Endpoint Template",
  "connectionType": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
      "code": "hl7-fhir-rest",
      "display": "HL7 FHIR"
    }]
  },
  "payloadType": [{
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc",
      "code": "bars",
      "display": "BaRS"
    }]
  }],
  "managingOrganization": [{
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "RK5"
    }
  }],
  "address": "bars-prod-rk5.nervecentre.thirdparty.nhs.uk",
  "header": "public"
}
```

> ⚠️ **Note:** The `identifier[].value` above uses a placeholder Product ID
> (`NervecentreBaRS-v1.0.0`). The actual value must come from the agreed Product ID
> mapping table once established.

#### Data quality issues to address before migration

| Issue | Records affected | Resolution |
|-------|------------------|------------|
| `Address` = `addressHere` (placeholder) | Multiple | Exclude from migration or obtain real URL from supplier |
| `ProductId` is an ODS code (e.g., `AC0`, `RK5`) not a product identifier | All records | Map via the Product ID mapping table (NHS team action) |
| `ConnectionSystem` varies (`http://...` vs `https://...`) | Some records | Normalise to `http://terminology.hl7.org/CodeSystem/endpoint-connection-type` |
| Duplicate Templates (same ProductId, different TemplateId) | Some records | Deduplicate — use the most recently updated record |

---

### Table: `int_endpoints`

This table holds the child Endpoints — each one represents a specific service's use of a
supplier product (Template).

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `EndpointId` | String (UUID) | Primary key — internal identifier |
| `Active` | Boolean | Whether the Endpoint is currently active |
| `Address` | String | Endpoint URL (inherited from Template in new model) |
| `ConnectionSystem` | String | FHIR CodeSystem (inherited from Template in new model) |
| `ConnectionType` | String | Connection type code (inherited from Template in new model) |
| `DataStatus` | Number | Internal status flag |
| `EndDate` | String (ISO 8601) | Period end date (empty = open-ended) |
| `EndpointTemplate` | String (JSON) | Embedded Template data (denormalised snapshot) |
| `HealthcareServiceId` | String (UUID) | FK to `int_healthcareservices` |
| `LastUpdated` | String (ISO 8601) | Last modification timestamp |
| `ManagingOrganisationId` | String (UUID) | FK to `int_organisations` |
| `Name` | String | Supplier name (e.g., `Cegedim`, `Pharmoutcomes`, `Sonar`) |
| `PartitionKey` | String | DynamoDB partition key (e.g., `ALL`) |
| `ProductId` | String | Product identifier (short code, e.g., `ygm04`, `8hk48`) |
| `SearchField` | String | Concatenated search index |
| `ServiceId` | String | DoS Service ID of the associated HealthcareService |
| `StartDate` | String (ISO 8601) | Period start date |
| `TemplateId` | String (UUID) | FK to `int_endpoint_templates` |

#### Query

```sql
-- Extract all endpoints for migration
SELECT
  e.EndpointId,
  e.Active,
  e.EndDate,
  e.StartDate,
  e.HealthcareServiceId,
  e.ManagingOrganisationId,
  e.ProductId,
  e.ServiceId,
  e.TemplateId,
  e.Name AS SupplierName,
  o.ODSCode AS ManagingOrganisationODS
FROM int_endpoints e
JOIN int_organisations o ON e.ManagingOrganisationId = o.OrganisationId
WHERE e.DataStatus = 0
```

#### Mapping to FHIR: `POST /Endpoint`

| Source column | FHIR field | Transformation |
|---------------|-----------|----------------|
| `EndpointId` | — | Not sent to API; recorded in migration log as `source_id` |
| `TemplateId` | — | Used to look up the `catalog_id` from Step 1 migration log → becomes the Template reference |
| — (resolved from Step 1 log) | `identifier[0].value` | The Product ID of the parent Template (resolved from Step 1) |
| — | `identifier[0].system` | Static: `https://fhir.nhs.uk/id/product-id` |
| `Active` | `status` | Map: `true` → `active`, `false` → `off` |
| `StartDate` | `period.start` | Direct copy (ISO 8601). If empty, assign migration date as fallback |
| `EndDate` | `period.end` | Direct copy (ISO 8601). If empty, omit (open-ended) |
| — | `meta.profile[0]` | Static: `http://hl7.org/fhir/StructureDefinition/Endpoint` |
| — | `meta.lastUpdated` | Runtime: current migration timestamp |

> **Note:** In the new EPC model, child Endpoints do **not** carry `address`,
> `connectionType`, `payloadType`, `managingOrganization`, `name`, or `header` — these are
> all inherited from the parent Template at read time. Only `status` and `period` are stored
> on the child Endpoint.

#### Example: Source row → FHIR payload

**Source:**
```
EndpointId: 16c8e0f4-538b-4d13-80e0-dbec070ddbcf
Active: true
StartDate: 2026-06-01T16:04:05.168Z
EndDate: (empty)
TemplateId: 26a1070e-c4d3-4ad3-887d-e63635497bd5
ServiceId: 2000114950
ProductId: ygm04
```

**Resolved from Step 1 log:** `TemplateId 26a1070e...` → `catalog_id: 5fce3e6a-ba37-4289-84d1-cc3ebdb992f5`

**FHIR payload:**
```json
{
  "resourceType": "Endpoint",
  "meta": {
    "lastUpdated": "2026-07-07T09:00:00+00:00",
    "profile": ["http://hl7.org/fhir/StructureDefinition/Endpoint"]
  },
  "identifier": [{
    "system": "https://fhir.nhs.uk/id/product-id",
    "value": "CegedimBaRS-v1.0.0"
  }],
  "status": "active",
  "period": {
    "start": "2026-06-01T16:04:05.168Z"
  }
}
```

#### Relationship to HealthcareService

The `HealthcareServiceId` and `ServiceId` columns are **not** included in the Endpoint
payload. Instead, they are used in Step 3 (HealthcareService migration) to build the
`endpoint[]` reference array on the HealthcareService resource. The association is:

```
int_endpoints.HealthcareServiceId → int_healthcareservices.HealthcareServiceId
    → Step 3 creates HealthcareService with endpoint[] referencing this Endpoint's catalog_id
```

#### Data quality issues to address before migration

| Issue | Records affected | Resolution |
|-------|------------------|------------|
| `StartDate` is empty | Some records | Assign migration date as `period.start` to avoid unbounded overlap conflicts |
| `Active` = `true` but associated HealthcareService has `Active` = `false` | Some records | Decide whether to migrate as `active` (Endpoint still valid) or `off` (service withdrawn) |
| Multiple Endpoints with same `TemplateId` and no `EndDate` | Some records | Only one unbounded Endpoint per Template is allowed — assign `period.start` to differentiate |

---

### Table: `int_healthcareservices`

This table holds the healthcare services — each one represents a care service (e.g., a
pharmacy, an ED, a UTC) with its DoS Service ID.

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `HealthcareServiceId` | String (UUID) | Primary key — internal identifier |
| `ServiceId` | String | DoS Service ID (e.g., `2000023201`) |
| `Active` | Boolean | Whether the service is currently active |
| `DataStatus` | Number | Internal status flag |
| `Endpoints` | List | References to associated Endpoints (JSON array — may be empty) |
| `LastUpdated` | String (ISO 8601) | Last modification timestamp |
| `Name` | String | Human-readable service name |
| `ProviderOrganisationId` | String (UUID) | FK to `int_organisations` |
| `SearchField` | String | Concatenated search index |
| `ServiceIdType` | String | The identifier system (e.g., `https://fhir.nhs.uk/id/service-id`) |

#### Query

```sql
-- Extract all healthcare services for migration
SELECT
  hs.HealthcareServiceId,
  hs.ServiceId,
  hs.Active,
  hs.Name,
  hs.ProviderOrganisationId,
  hs.ServiceIdType,
  o.ODSCode AS ProviderODS
FROM int_healthcareservices hs
JOIN int_organisations o ON hs.ProviderOrganisationId = o.OrganisationId
WHERE hs.DataStatus = 0
```

**Additionally**, resolve the Endpoint associations by querying `int_endpoints`:

```sql
-- Get all Endpoints associated with this HealthcareService
SELECT EndpointId, TemplateId
FROM int_endpoints
WHERE HealthcareServiceId = '{HealthcareServiceId}'
  AND DataStatus = 0
```

Each `EndpointId` is then looked up in the Step 2 migration log to get its `catalog_id`.

#### Mapping to FHIR: `POST /HealthcareService`

| Source column | FHIR field | Transformation |
|---------------|-----------|----------------|
| `HealthcareServiceId` | — | Not sent to API; recorded in migration log as `source_id` |
| `ServiceId` | `identifier[0].value` | Direct copy |
| `ServiceIdType` | `identifier[0].system` | Map: `https://fhir.nhs.uk/id/service-id` → `https://fhir.nhs.uk/Id/dos-service-id` |
| `Active` | `active` | Direct copy (`true`/`false`) |
| `Name` | `name` | Direct copy. Strip surrounding quotes if present |
| `ProviderODS` | `providedBy.identifier.value` | Resolved via join to `int_organisations` |
| — | `providedBy.identifier.system` | Static: `https://fhir.nhs.uk/Id/ods-organization-code` |
| (resolved from Step 2 log) | `endpoint[]` | Array of `{"reference": "Endpoint/{catalog_id}"}` for each associated Endpoint |
| — | `meta.profile` | Static: `https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService` |
| — | `meta.lastUpdated` | Runtime: current migration timestamp |

#### Example: Source row → FHIR payload

**Source:**
```
HealthcareServiceId: e79e7a8a-eb25-4939-8635-56d43e71f292
ServiceId: 2000023201
Active: false
Name: "Pharm+: Victoria Pharmacy Golders Green, Barnet, London"
ProviderOrganisationId: 281b316c-3a38-41dc-8d52-12c55e4c72a5
ServiceIdType: https://fhir.nhs.uk/id/service-id
```

**Resolved ODS code:** `FLG23` (from `int_organisations` join)

**Resolved Endpoints:** (from `int_endpoints` where `HealthcareServiceId` matches → Step 2 log lookup)
- `catalog_id: 0cb21027-a246-43e6-9c7a-35b17163eab1`

**FHIR payload:**
```json
{
  "resourceType": "HealthcareService",
  "meta": {
    "lastUpdated": "2026-07-07T09:00:00+00:00",
    "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"]
  },
  "identifier": [{
    "system": "https://fhir.nhs.uk/Id/dos-service-id",
    "value": "2000023201"
  }],
  "active": false,
  "name": "Pharm+: Victoria Pharmacy Golders Green, Barnet, London",
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FLG23"
    }
  },
  "endpoint": [
    {"reference": "Endpoint/0cb21027-a246-43e6-9c7a-35b17163eab1"}
  ]
}
```

#### Data quality issues to address before migration

| Issue | Records affected | Resolution |
|-------|------------------|------------|
| `Name` contains surrounding quotes (`"""..."""`) | Many records | Strip leading/trailing quotes during transformation |
| `Endpoints` array in source is empty `[]` but `int_endpoints` has records with matching `HealthcareServiceId` | Some records | Use the `int_endpoints` query (not the denormalised `Endpoints` column) to resolve associations |
| `ServiceIdType` is `https://fhir.nhs.uk/id/service-id` (lowercase) | All records | Normalise to `https://fhir.nhs.uk/Id/dos-service-id` |
| `Active` = `false` with no associated Endpoints | Many records | Still migrate — creates a deactivated HealthcareService in the Catalogue |

---

## Snapshot Tables (Not Migrated)

The following tables are audit/history records and are **not** migrated into the new EPC:

| Table | Purpose |
|-------|---------|
| `int_endpoint_snapshots` | Change history for Endpoints (includes `AuditType`, `TimeOfChange`) |
| `int_endpoint_templates_snapshots` | Change history for Templates |
| `int_healthcareservices_snapshots` | Change history for HealthcareServices |
| `int_organisations_snapshots` | Change history for Organisations |

These tables may be used for:
- **Validation** — confirming the current state of a record matches expectations
- **Investigation** — tracing the history of a specific record if migration produces
  unexpected results
- **Audit continuity** — if historical audit data needs to be preserved in a separate
  archive post-migration

---

## Migration Execution Order

```
┌─────────────────────────────────────────────────────────────────┐
│  MIGRATION EXECUTION                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pre-requisite: Product ID mapping table (NHS team action)      │
│                                                                 │
│  Step 1 — Templates                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Query: int_endpoint_templates JOIN int_organisations    │    │
│  │ Transform: Apply Product ID mapping, fix ConnectionType │    │
│  │ API call: POST /Endpoint/$template                      │    │
│  │ Output: migration log with source_id → catalog_id       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  Step 2 — Endpoints                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Query: int_endpoints JOIN int_organisations             │    │
│  │ Transform: Map Active→status, resolve TemplateId from   │    │
│  │            Step 1 log, assign period.start if missing    │    │
│  │ API call: POST /Endpoint                                │    │
│  │ Output: migration log with source_id → catalog_id       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  Step 3 — HealthcareServices                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Query: int_healthcareservices JOIN int_organisations     │    │
│  │        + int_endpoints (for association resolution)      │    │
│  │ Transform: Resolve Endpoint catalog_ids from Step 2 log,│    │
│  │            strip quotes from Name, normalise ServiceIdType│   │
│  │ API call: POST /HealthcareService                       │    │
│  │ Output: migration log with source_id → catalog_id       │    │
│  │         (List auto-created by API)                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  Step 4 — Validation                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Compare migrated data against targets.json              │    │
│  │ Verify: each ServiceId resolves to correct Endpoint URL │    │
│  │ Report: discrepancies for investigation                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Transformations Summary

| Source value | Target value | Applied to |
|-------------|-------------|------------|
| `ProductId` (short code: `AC0`, `RK5`, `ygm04`) | Agreed Product ID (e.g., `AdvancedELMS-v1.0.0`) | Template `identifier[].value` |
| `ConnectionType` = `BARS` | `connectionType.coding[].code` = `hl7-fhir-rest` | Templates |
| `ConnectionType` = `BARS` | `payloadType[].coding[].code` = `bars` | Templates |
| `IsPrivate` = `false` / `true` | `header` = `public` / `private` | Templates |
| `Active` = `true` / `false` | `status` = `active` / `off` | Endpoints |
| `StartDate` (empty) | `period.start` = migration date | Endpoints |
| `ServiceIdType` = `https://fhir.nhs.uk/id/service-id` | `identifier[].system` = `https://fhir.nhs.uk/Id/dos-service-id` | HealthcareServices |
| `Name` with surrounding quotes | Stripped quotes | HealthcareServices |
| `ManagingOrganisationId` (UUID) | ODS code (via join) | Templates, Endpoints |
| `ProviderOrganisationId` (UUID) | ODS code (via join) | HealthcareServices |

---

## Blocking Dependencies

| Dependency | Owner | Status |
|-----------|-------|--------|
| Product ID mapping table (short code → agreed Product ID) | NHS team (R&M / BaRS programme) | 🔴 Not started |
| Placeholder URL resolution (`addressHere` → real URLs) | NHS team + suppliers | 🔴 Not started |
| ConnectionSystem normalisation (confirm canonical value) | Development team | 🟡 To confirm |
| Duplicate Template deduplication rules | Development team | 🟡 To confirm |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Data Migration](../Documents/data-migration.md) | Full migration process with API call examples and error handling |
| [Managing Endpoint Templates](../Documents/Processes/manage-endpoint-template.md) | Template CRUD operations in the new EPC |
| [Managing Endpoints](../Documents/Processes/manage-endpoint.md) | Endpoint CRUD operations in the new EPC |
| [Managing HealthcareServices](../Documents/Processes/manage-healthcare-service.md) | HealthcareService CRUD operations in the new EPC |
