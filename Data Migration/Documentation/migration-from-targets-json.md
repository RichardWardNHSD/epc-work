# Migration Process Design: targets.json → EPC

## Objective

Populate the new Endpoint Catalogue (EPC) using `targets.json` as the principal data source, enriched with supplier metadata from the `int_` DynamoDB tables. This approach treats the flat file as the source of truth for service-to-endpoint routing and builds the minimum viable EPC state to reproduce that routing.

---

## Why targets.json as primary source?

The `targets.json` file is the live routing configuration — it defines which DoS service IDs resolve to which supplier URLs. Starting from this file ensures:

1. Only services that are actively routed get migrated (no stale/orphaned data)
2. The validation step is a direct comparison against the same source
3. Simpler process — fewer records, no filtering of inactive/placeholder data
4. Guarantees the EPC can reproduce current production routing on day one

---

## Source Data Structure

`targets.json` structure:

```json
{
  "NHSD-Target-Identifier": {
    "tests": { ... },
    "https://fhir.nhs.uk/Id/dos-service-id": {
      "<service_id>": "<endpoint_url>",
      "<service_id>": "<endpoint_url>",
      ...
    }
  }
}
```

We only process the `https://fhir.nhs.uk/Id/dos-service-id` section. The `tests` section is ignored.

Each key-value pair represents:

- **Key:** DoS service ID (numeric string, e.g., `"2000017562"`)
- **Value:** The supplier's FHIR receiver URL (e.g., `"https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/"`)

---

## Overview

```mermaid
flowchart TD
    TJ[targets.json] --> PARSE[Parse: extract unique URLs + service-to-URL pairs]
    PARSE --> ENRICH[Enrich: resolve supplier metadata + provider organisations]
    ENRICH --> S1[Step 1: Create Endpoint Templates per unique URL]
    S1 --> S2[Step 2: Create child Endpoints per unique URL]
    S2 --> S3[Step 3: Create HealthcareServices per service ID]
    S3 --> VALIDATE[Step 4: Validate by regenerating targets.json from EPC]

    subgraph Enrichment["Enrichment Sources (DynamoDB)"]
        INT_EP[int_endpoints]
        INT_TPL[int_endpoint_templates]
        INT_ORG[int_organisations]
        INT_HCS[int_healthcareservices]
    end

    ENRICH --> INT_EP
    ENRICH --> INT_TPL
    ENRICH --> INT_ORG
    ENRICH --> INT_HCS
```

---

## Pre-requisites


| Item                             | Description                                                                                             | Status                     |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `targets.json`                   | Current production routing file                                                                         | Required                   |
| EPC API available                | Target environment (INT or DEV) accessible                                                              | Required                   |
| API credentials                  | Bearer token or OAuth2 client credentials for EPC API                                                   | Required                   |
| AWS access                       | IAM role/credentials with read access to`int_` DynamoDB tables (for enrichment and provider resolution) | Required                   |
| Product ID mapping               | Persistent `product-id-lookup.json` file mapping short codes (ygm04, AC0, etc.) → agreed EPC Product IDs. Must be accessible to all scripts (migration, delta, validation). | Required                   |
| Provider organisation resolution | `int_healthcareservices` + `int_organisations` scanned to build service_id → provider ODS lookup       | Required (built in Step 0) |
| Migration log store              | Persistent map of`source_id → catalog_id` for cross-referencing between steps                          | Required                   |

---

## Step 0: Parse targets.json and Derive Unique URLs

**Input:** `targets.json`

**Action:** Extract the `https://fhir.nhs.uk/Id/dos-service-id` section, build two data structures:

1. **service_to_url** — the full mapping (used in Steps 2-3)
2. **unique_urls** — deduplicated list of endpoint URLs (used in Step 1)

```python
import json

with open('targets.json') as f:
    data = json.load(f)

service_section = data["NHSD-Target-Identifier"]["https://fhir.nhs.uk/Id/dos-service-id"]

# Full mapping: service_id → url
service_to_url = service_section  # ~4000+ entries

# Unique URLs (case-insensitive dedup)
seen = {}
unique_urls = []
for url in service_to_url.values():
    normalised = url.lower().rstrip('/')
    if normalised not in seen:
        seen[normalised] = url  # keep original casing
        unique_urls.append(url)

print(f"Total service mappings: {len(service_to_url)}")
print(f"Unique endpoint URLs: {len(unique_urls)}")
```

**Output:**

- `service_to_url`: dict of ~4,000+ entries
- `unique_urls`: list of ~13 unique URLs

---

## Step 0a: Build Provider Organisation Lookup (Required)

targets.json does not contain the provider organisation (the pharmacy, hospital, or service that delivers care). This must be resolved before Step 3, so it is built as part of Step 0.

**Source:** `int_healthcareservices` DynamoDB table + `int_organisations` (for ODS code resolution)

**Action:** Scan `int_healthcareservices` and build a dictionary keyed by `ServiceId` that maps each service to its provider organisation ODS code and service name.

```python
hcs_table = dynamodb.Table('int_healthcareservices')

# Build service_id → provider organisation + service name lookup
provider_lookup = {}
response = hcs_table.scan(
    FilterExpression=Attr('DataStatus').eq(0)
)
for item in response['Items']:
    service_id = item.get('ServiceId', '')
    provider_org_id = item.get('ProviderOrganisationId', '')
    org = org_lookup.get(provider_org_id, {})
    provider_lookup[service_id] = {
        "provider_ods": org.get('ods_code', ''),
        "provider_name": org.get('name', ''),
        "name": item.get('Name', '').strip('"'),
    }

# Handle pagination
while 'LastEvaluatedKey' in response:
    response = hcs_table.scan(
        FilterExpression=Attr('DataStatus').eq(0),
        ExclusiveStartKey=response['LastEvaluatedKey']
    )
    for item in response['Items']:
        service_id = item.get('ServiceId', '')
        provider_org_id = item.get('ProviderOrganisationId', '')
        org = org_lookup.get(provider_org_id, {})
        provider_lookup[service_id] = {
            "provider_ods": org.get('ods_code', ''),
            "provider_name": org.get('name', ''),
            "name": item.get('Name', '').strip('"'),
        }

print(f"Provider lookup entries: {len(provider_lookup)}")
```

**Validation:** After building, check coverage against targets.json:

```python
missing_providers = []
for service_id in service_to_url.keys():
    if service_id not in provider_lookup:
        missing_providers.append(service_id)

if missing_providers:
    print(f"WARNING: {len(missing_providers)} services in targets.json have no provider in int_healthcareservices")
    print(f"These will be created without providedBy — must be resolved before go-live")
else:
    print("All services have provider organisation resolved")
```

**Output:** `provider_lookup` — dict of `service_id → { provider_ods, provider_name, name }`

This is a **required** pre-requisite for Step 3. Any services missing from this lookup must be flagged for manual resolution before migration is considered complete.

---

## Step 0b: Enrich — Resolve Supplier Metadata for Each Unique URL

For each unique URL, we need to know: which supplier owns it, their ODS code, and the product ID. This comes from the `int_` DynamoDB tables.

**Process:** For each URL in `unique_urls`, query `int_endpoint_templates` or `int_endpoints` to find a matching `Address` and extract the supplier metadata.

```python
import boto3
from boto3.dynamodb.conditions import Attr

dynamodb = boto3.resource('dynamodb')
templates_table = dynamodb.Table('int_endpoint_templates')
orgs_table = dynamodb.Table('int_organisations')

# Build org lookup (same as main migration doc Step 1)
org_lookup = {}
response = orgs_table.scan()
for item in response['Items']:
    org_lookup[item['OrganisationId']] = {
        "ods_code": item.get('ODSCode', ''),
        "name": item.get('Name', ''),
    }
while 'LastEvaluatedKey' in response:
    response = orgs_table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
    for item in response['Items']:
        org_lookup[item['OrganisationId']] = {
            "ods_code": item.get('ODSCode', ''),
            "name": item.get('Name', ''),
        }

# Scan templates to build URL → supplier metadata map
templates_response = templates_table.scan(
    FilterExpression=Attr('DataStatus').eq(0) & Attr('Address').ne('addressHere')
)

url_metadata = {}
for item in templates_response['Items']:
    address = item.get('Address', '')
    managing_org_id = item.get('ManagingOrganisationId', '')
    org = org_lookup.get(managing_org_id, {})
  
    url_metadata[address.lower().rstrip('/')] = {
        "address": address,
        "product_id": item.get('ProductId', ''),
        "supplier_name": item.get('Name', ''),
        "managing_org_id": managing_org_id,
        "managing_org_ods": org.get('ods_code', ''),
        "template_id": item.get('TemplateId', ''),
        "is_private": item.get('IsPrivate', False),
    }
```

### URL to Supplier Metadata (Expected Output)


| URL (normalised)                                              | ProductId | Supplier Name      | Managing Org ODS  | Template Source        |
| --------------------------------------------------------------- | ----------- | -------------------- | ------------------- | ------------------------ |
| `bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk`        | ygm06     | Pharmoutcomes      | (from org_lookup) | int_endpoint_templates |
| `bars-prod-ygm04.cegedim.thirdparty.nhs.uk/fhir/r4/`          | ygm04     | Cegedim            | (from org_lookup) | int_endpoint_templates |
| `bars-prod-8hk48.sonar.thirdparty.nhs.uk`                     | 8hk48     | Sonar              | (from org_lookup) | int_endpoint_templates |
| `bars-prod-ygm17.positivesolutions.thirdparty.nhs.uk`         | ygm17     | Positive Solutions | (from org_lookup) | int_endpoint_templates |
| `bars-prod-8jy34.waspsoftware.thirdparty.nhs.uk/api/r4`       | 8JY34     | WASP               | (from org_lookup) | int_endpoint_templates |
| `bars-prod-ac0.advanced.thirdparty.nhs.uk/fhirr4/api/bars`    | AC0       | Advanced           | (from org_lookup) | int_endpoint_templates |
| `bars-prod-8hq44.stratahealth.thirdparty.nhs.uk:3120/bars/`   | 8hq44     | Strata Health      | (from org_lookup) | int_endpoint_templates |
| `bars-prod-y01061.advanced.thirdparty.nhs.uk/fhirr4/api/bars` | Y01061    | Advanced           | (from org_lookup) | int_endpoint_templates |
| `bars-prod-8ht86.fortrus.thirdparty.nhs.uk/`                  | 8ht86     | Fortrus            | (from org_lookup) | int_endpoint_templates |
| `bars-prod-rk5.nervecentre.thirdparty.nhs.uk`                 | RK5       | Nervecentre        | (from org_lookup) | int_endpoint_templates |
| `bars-prod-rx7.nwas.nhs.uk`                                   | RX7       | NWAS               | (from org_lookup) | int_endpoint_templates |
| `bars-prod-ga9.gmupca.nhs.uk/fhirr4/api/bars`                 | GA9       | GMUPCA             | (from org_lookup) | int_endpoint_templates |
| `bars-prod-rk5.nervecentre.thirdparty.nhs.uk:9999/...`        | RK5       | Nervecentre        | (from org_lookup) | int_endpoint_templates |

### Product ID Resolution

See the dedicated document: **[Resolving ProductId](./resolving-product-id.md)**

The source data (int_ tables and targets.json) uses internal short codes to identify supplier products — e.g., `ygm04` for Cegedim, `8hk48` for Sonar, `AC0` for Advanced. These short codes are not meaningful outside the existing system and are not used in the EPC.

The EPC uses a formal Product Identifier (e.g., `CegedimPharmacyServices-v6.0`) that uniquely identifies the supplier's product and version. This is what gets stored in the `identifier` field on Endpoint Templates and child Endpoints.

The resolution process:
1. Take the `ProductId` short code from the source data (e.g., from `url_metadata` enrichment)
2. Look it up in the persistent `product-id-lookup.json` file (case-insensitive)
3. Return the agreed EPC Product Identifier

If the short code is not found in the lookup, the record cannot be migrated — it's logged as a gap for the R&M team to resolve by adding the mapping.

```python
from resolving_product_id import load_product_id_map
PRODUCT_ID_MAP = load_product_id_map()
```

---

## Step 1: Create Endpoint Templates (one per unique URL)

For each unique URL in `unique_urls`, create an Endpoint Template in the EPC. This represents the supplier's receiver system at that address.

**For each unique URL:**

1. Look up the URL in `url_metadata` (case-insensitive, trailing-slash-tolerant)
2. Resolve `ProductId` → EPC Product Identifier via `PRODUCT_ID_MAP`
3. Build the FHIR Endpoint Template payload
4. Call: `POST /Endpoint/$template`
5. Record: `{ url: catalog_id }` in `template_log`

### Payload Parameter Table


| FHIR Field                                  | Example Value                                                       | Source                                        | How to derive                                                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `resourceType`                              | `"Endpoint"`                                                        | Static                                        | Always`"Endpoint"`                                                                                                                     |
| `identifier[0].system`                      | `"https://fhir.nhs.uk/id/product-id"`                               | Static                                        | Always this system URI                                                                                                                 |
| `identifier[0].value`                       | `"CegedimPharmacyServices-v6.0"`                                    | `url_metadata.product_id` → `PRODUCT_ID_MAP` | Look up the`product_id` for this URL from the enrichment step. Then resolve via `PRODUCT_ID_MAP` to the agreed EPC Product Identifier. |
| `status`                                    | `"active"`                                                          | Static                                        | Always`"active"` — these URLs are in the live routing file.                                                                           |
| `connectionType.coding[0].system`           | `"http://terminology.hl7.org/CodeSystem/endpoint-connection-type"`  | Static                                        | Always this system URI                                                                                                                 |
| `connectionType.coding[0].code`             | `"hl7-fhir-rest"`                                                   | Static                                        | Always`"hl7-fhir-rest"` — all BaRS endpoints are FHIR REST.                                                                           |
| `connectionType.coding[0].display`          | `"HL7 FHIR"`                                                        | Static                                        | Always`"HL7 FHIR"`                                                                                                                     |
| `payloadType[0].coding[0].system`           | `"http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc"` | Static                                        | Always this system URI                                                                                                                 |
| `payloadType[0].coding[0].code`             | `"bars"`                                                            | Static                                        | Always`"bars"` — all entries in targets.json are BaRS routing.                                                                        |
| `payloadType[0].coding[0].display`          | `"BaRS"`                                                            | Static                                        | Always`"BaRS"`                                                                                                                         |
| `managingOrganization[0].identifier.system` | `"https://fhir.nhs.uk/Id/ods-organization-code"`                    | Static                                        | Always this system URI                                                                                                                 |
| `managingOrganization[0].identifier.value`  | `"RK5"`                                                             | `url_metadata.managing_org_ods`               | The ODS code of the supplier organisation that manages this endpoint. Resolved during enrichment via`int_organisations`.               |
| `address`                                   | `"https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/"`      | `targets.json` URL value                      | Direct copy of the URL from targets.json. Preserve original casing. Ensure`https://` scheme is present.                                |
| `header`                                    | `"public"`                                                          | `url_metadata.is_private`                     | Map:`false` → `"public"`, `true` → `"private"`. Default to `"public"` if enrichment data is unavailable.                             |

### Example payload

URL: `https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/`

Enrichment resolved:

- ProductId: `ygm04` → `"CegedimPharmacyServices-v6.0"`
- ManagingOrg ODS: `"(resolved from org_lookup)"`  — e.g., supplier ODS for Cegedim
- IsPrivate: `false` → `"public"`

```json
{
  "resourceType": "Endpoint",
  "identifier": [{
    "system": "https://fhir.nhs.uk/id/product-id",
    "value": "CegedimPharmacyServices-v6.0"
  }],
  "status": "active",
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
      "value": "YGM04"
    }
  }],
  "address": "https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/",
  "header": "public"
}
```

**Output:** `template_log` — map of `normalised_url → { catalog_id, product_id }`

---

## Step 2: Create Child Endpoints (one per unique URL)

In the EPC model, each Template needs at least one child Endpoint to be routable. Since targets.json doesn't carry per-service endpoint identity (just URL), we create **one child Endpoint per Template** — all services sharing that URL will reference the same child Endpoint.

**For each unique URL:**

1. Look up the URL in `template_log` to get the parent Template's `catalog_id`
2. Build the FHIR child Endpoint payload
3. Call: `POST /Endpoint`
4. Record: `{ url: endpoint_catalog_id }` in `endpoint_log`

### Payload Parameter Table


| FHIR Field                              | Example Value                         | Source                               | How to derive                                                                                                                                                                          |
| ----------------------------------------- | --------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `resourceType`                          | `"Endpoint"`                          | Static                               | Always`"Endpoint"`                                                                                                                                                                     |
| `identifier[0].system`                  | `"https://fhir.nhs.uk/id/product-id"` | Static                               | Always this system URI                                                                                                                                                                 |
| `identifier[0].value`                   | `"CegedimPharmacyServices-v6.0"`      | Copied from parent Template          | Use the same Product ID value that was sent to the Template in Step 1. No separate lookup required — copy from the Template payload already built for this URL.                       |
| `extension[0].url`                      | `"http://hl7.org"`                    | Static                               | Always this URL — identifies the "basedOn" extension.                                                                                                                                 |
| `extension[0].valueReference.reference` | `"Endpoint/5fce3e6a-..."`             | `template_log`                       | Look up the normalised URL in`template_log` to get the parent Template's catalog_id. Format as `"Endpoint/{catalog_id}"`.                                                              |
| `extension[0].valueReference.display`   | `"Parent Template Endpoint"`          | Static                               | Always`"Parent Template Endpoint"`                                                                                                                                                     |
| `status`                                | `"active"`                            | `endpoint_details` or Static         | Look up`service_id` in `endpoint_details`. Map `Active`: `"true"` → `"active"`, `"false"` → `"off"`. If service_id not found, default to `"active"` (it's in the live routing file). |
| `period.start`                          | `"2026-06-01T16:04:05.168Z"`          | `endpoint_details` or migration date | See decision note below.                                                                                                                                                               |
| `period.end`                            | `"2026-04-22T15:40:17.423Z"`          | `endpoint_details` (if populated)    | See decision note below.**Only include if populated.**                                                                                                                                 |

### Decision: How to resolve `period.start` and `period.end`

targets.json has no concept of start or end dates — every entry is implicitly "active now". However, child Endpoints in the EPC require a `period.start` and optionally `period.end`. There are two options:

#### Option 1: Default to migration date (simple)

Set `period.start` to the migration execution date for all endpoints. Omit `period.end`.

- **Pros:** Simple, no dependency on `int_endpoints` data quality, no additional DynamoDB scan
- **Cons:** Loses historical information about when the endpoint was actually activated; all endpoints appear to have started on the same day

#### Option 2: Resolve from int_endpoints (recommended)

For each `service_id` in targets.json, find matching records in `int_endpoints` (by `ServiceId`) and derive the period from those records.

**Logic:**

1. Query `int_endpoints` for all items where `ServiceId == service_id` and `DataStatus == 0`
2. If multiple items exist (e.g., endpoint was re-onboarded), use the **most recent active** record — latest `StartDate` with no `EndDate`, or if all have `EndDate`, use the one with the latest `LastUpdated`
3. Copy `StartDate` → `period.start`
4. If `EndDate` is populated, copy to `period.end` (unlikely for services in targets.json since they're actively routed, but handles edge cases)
5. If `StartDate` is empty on the matched record, fall back to migration date

If no matching record found in `int_endpoints` at all, fall back to Option 1 (migration date).

```python
MIGRATION_DATE = "2026-07-20T00:00:00Z"

def resolve_period(service_id, endpoint_details):
    """
    Resolve period for a service.
    Option 2: use int_endpoints data, with migration date as fallback.
    """
    details = endpoint_details.get(service_id)
  
    if not details:
        # No int_endpoints record — fall back to migration date
        return {"start": MIGRATION_DATE}, "fallback_no_record"
  
    period = {}
  
    if details["start_date"]:
        period["start"] = details["start_date"]
    else:
        # Record exists but StartDate is empty — use migration date
        period["start"] = MIGRATION_DATE
  
    if details["end_date"]:
        period["end"] = details["end_date"]
  
    return period, "resolved"
```

#### Recommendation

Use **Option 2** — resolve from `int_endpoints` where possible, with migration date as fallback. This preserves historical accuracy while ensuring no endpoint is missing a `period.start`.

Expected coverage: most service IDs in targets.json will have a matching `int_endpoints` record. Services without a match are likely new additions to the flat file that weren't yet onboarded to the EPC — these get the migration date.

### Fields NOT included (inherited from Template at read time)


| Field                  | Reason                         |
| ------------------------ | -------------------------------- |
| `address`              | Inherited from parent Template |
| `connectionType`       | Inherited from parent Template |
| `payloadType`          | Inherited from parent Template |
| `managingOrganization` | Inherited from parent Template |
| `name`                 | Inherited from parent Template |
| `header`               | Inherited from parent Template |

### Example payload

For URL `https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/` where `template_log` gives catalog_id `"5fce3e6a-ba37-4289-84d1-cc3ebdb992f5"`:

```json
{
  "resourceType": "Endpoint",
  "identifier": [{
    "system": "https://fhir.nhs.uk/id/product-id",
    "value": "CegedimPharmacyServices-v6.0"
  }],
  "extension": [{
    "url": "http://hl7.org",
    "valueReference": {
      "reference": "Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5",
      "display": "Parent Template Endpoint"
    }
  }],
  "status": "active",
  "period": {
    "start": "2026-06-01T16:04:05.168Z"
  }
}
```

**Output:** `endpoint_log` — map of `service_id → endpoint_catalog_id`

---

## Step 3: Create HealthcareServices (one per service ID in targets.json)

For each key-value pair in `service_to_url`, create a HealthcareService that references the child Endpoint for its URL.

**For each service_id, url pair in `service_to_url`:**

1. Look up `service_id` in `endpoint_log` to get the child Endpoint's `catalog_id`
2. Resolve provider organisation ODS code from `provider_lookup` (built in Step 0b)
3. Resolve service name from `provider_lookup` (built in Step 0b)
4. Build the FHIR HealthcareService payload
5. Call: `POST /HealthcareService`
6. Record: `{ service_id: hcs_catalog_id }` in `hcs_log`

### Provider Organisation (from Step 0b)

The `provider_lookup` dictionary built in Step 0b provides:

- `provider_ods` — the ODS code of the provider organisation (pharmacy/hospital)
- `provider_name` — the organisation name
- `name` — the human-readable service name

These are required fields. If a service_id is not found in `provider_lookup`, log it as a migration gap that must be resolved.

### Payload Parameter Table


| FHIR Field                     | Example Value                                                            | Source                           | How to derive                                                                                                                                            |
| -------------------------------- | -------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `resourceType`                 | `"HealthcareService"`                                                    | Static                           | Always`"HealthcareService"`                                                                                                                              |
| `meta.profile[0]`              | `"https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"` | Static                           | Always this profile URI                                                                                                                                  |
| `identifier[0].system`         | `"https://fhir.nhs.uk/Id/dos-service-id"`                                | Static                           | Always`"https://fhir.nhs.uk/Id/dos-service-id"` — this is the identifier system used by the BaRS proxy to query the EPC.                                |
| `identifier[0].value`          | `"2000017562"`                                                           | `targets.json` key               | Direct copy of the service ID key from the JSON.                                                                                                         |
| `active`                       | `true`                                                                   | Static                           | Always`true` — the service is in the live routing file, so it's active.                                                                                 |
| `name`                         | `"Pharm+: Victoria Pharmacy Golders Green"`                              | `provider_lookup` (from Step 0b) | Look up`service_id` in `provider_lookup`. Use the `name` field. If not found, log as a migration gap — this must be resolved. Strip surrounding quotes. |
| `providedBy.identifier.system` | `"https://fhir.nhs.uk/Id/ods-organization-code"`                         | Static                           | Always this system URI.                                                                                                                                  |
| `providedBy.identifier.value`  | `"FLG23"`                                                                | `provider_lookup` (from Step 0b) | Look up`service_id` in `provider_lookup`. Use the `provider_ods` field. If not found, log as a migration gap — must be resolved before go-live.         |
| `endpoint[0].reference`        | `"Endpoint/abc123-..."`                                                  | `endpoint_log`                   | Look up`service_id` in `endpoint_log` to get the child Endpoint's catalog_id. Format as `"Endpoint/{catalog_id}"`.                                       |

### Example payload

For targets.json entry: `"2000017562": "https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/"`

Enrichment resolved:

- `provider_lookup["2000017562"]` → `{ provider_ods: "FE284", name: "Pharm+: Boots Pharmacy Bromley" }`
- `endpoint_log["2000017562"]` → catalog_id `"0cb21027-a246-43e6-9c7a-35b17163eab1"`

```json
{
  "resourceType": "HealthcareService",
  "meta": {
    "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"]
  },
  "identifier": [{
    "system": "https://fhir.nhs.uk/Id/dos-service-id",
    "value": "2000017562"
  }],
  "active": true,
  "name": "Pharm+: Boots Pharmacy Bromley",
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FE284"
    }
  },
  "endpoint": [
    {"reference": "Endpoint/0cb21027-a246-43e6-9c7a-35b17163eab1"}
  ]
}
```

**Output:** `hcs_log` — map of `service_id → hcs_catalog_id`

---

## Step 4: Validate — Rebuild targets.json from EPC

Query the EPC for every service ID from the original targets.json and verify the resolved Endpoint address matches.

### 4a. Query the EPC

```python
import requests

EPC_BASE = "https://int.api.service.nhs.uk/endpoint-catalogue"
reconstructed = {}
errors = []

for service_id, expected_url in service_to_url.items():
    response = requests.get(
        f"{EPC_BASE}/HealthcareService",
        params={
            "identifier": f"https://fhir.nhs.uk/Id/dos-service-id|{service_id}",
            "_include": "HealthcareService:endpoint"
        },
        headers={"Authorization": f"Bearer {token}"}
    )
  
    bundle = response.json()
  
    # Find the Endpoint in the included resources
    endpoint_address = None
    for entry in bundle.get("entry", []):
        resource = entry.get("resource", {})
        if resource.get("resourceType") == "Endpoint" and resource.get("status") == "active":
            endpoint_address = resource.get("address")
            break
  
    if endpoint_address:
        reconstructed[service_id] = endpoint_address
    else:
        reconstructed[service_id] = "NOT_FOUND"
        errors.append(service_id)
```

### 4b. Compare

```python
def urls_match(url1, url2):
    """Case-insensitive, trailing-slash-tolerant URL comparison."""
    if not url1 or not url2:
        return False
    return url1.lower().rstrip('/') == url2.lower().rstrip('/')

differences = []
for service_id, expected_url in service_to_url.items():
    actual_url = reconstructed.get(service_id)
    if not urls_match(expected_url, actual_url):
        differences.append({
            "service_id": service_id,
            "expected": expected_url,
            "actual": actual_url
        })

print(f"Total services: {len(service_to_url)}")
print(f"Matched: {len(service_to_url) - len(differences)}")
print(f"Mismatched: {len(differences)}")
print(f"Not found: {len(errors)}")
```

### 4c. Generate reconstructed targets.json

```python
output = {
    "NHSD-Target-Identifier": {
        "https://fhir.nhs.uk/Id/dos-service-id": reconstructed
    }
}
with open('targets-reconstructed.json', 'w') as f:
    json.dump(output, f, indent=2)
```

### 4d. Success Criteria


| Metric                                             | Target |
| ---------------------------------------------------- | -------- |
| Services with correct URL match (case-insensitive) | 100%   |
| Services returning NOT_FOUND                       | 0      |
| Total differences                                  | 0      |

### Execution Sequence Diagram

```mermaid
sequenceDiagram
    participant Script as Migration Script
    participant TJ as targets.json
    participant DDB as DynamoDB (int_ tables)
    participant EPC as EPC API
    participant Log as Migration Log

    Note over Script: Step 0 - Parse & Enrich
    Script->>TJ: Load targets.json
    Script->>Script: Extract service_to_url + unique_urls
    Script->>DDB: Scan int_organisations (build org_lookup)
    Script->>DDB: Scan int_endpoint_templates (build url_metadata)
    Script->>DDB: Scan int_healthcareservices (build provider_lookup — REQUIRED)
    Script->>Script: Validate provider_lookup covers all service IDs in targets.json

    Note over Script: Step 1 - Templates (~13 URLs)
    loop For each unique URL
        Script->>Script: Resolve ProductId, ODS code from url_metadata
        Script->>EPC: POST /Endpoint/$template
        EPC-->>Script: 201 Created {id}
        Script->>Log: Record url → template_catalog_id
    end

    Note over Script: Step 2 - Child Endpoints (~13)
    loop For each unique URL
        Script->>Log: Lookup parent template catalog_id
        Script->>EPC: POST /Endpoint
        EPC-->>Script: 201 Created {id}
        Script->>Log: Record url → endpoint_catalog_id
    end

    Note over Script: Step 3 - HealthcareServices (~4000+)
    loop For each service_id in targets.json
        Script->>Log: Lookup endpoint catalog_id by URL
        Script->>Script: Resolve provider ODS from provider_lookup
        Script->>EPC: POST /HealthcareService
        EPC-->>Script: 201 Created {id}
        Script->>Log: Record service_id → hcs_catalog_id
    end

    Note over Script: Step 4 - Validation (~4000 queries)
    loop For each service_id in targets.json
        Script->>EPC: GET /HealthcareService?identifier=...&_include=endpoint
        EPC-->>Script: Bundle {HealthcareService + Endpoint}
        Script->>Script: Compare endpoint.address with expected URL
    end
    Script->>Script: Generate diff report
```

---

### Data Volumes & Estimates


| Step                        | Records                   | EPC API Calls          | Notes                                                              |
| ----------------------------- | --------------------------- | ------------------------ | -------------------------------------------------------------------- |
| Step 0 (parse + enrich)     | 4,000+ services, ~13 URLs | 0 (DynamoDB only)      | 4 table scans (orgs, templates, endpoints, healthcareservices)     |
| Step 1 (templates)          | ~13 unique URLs           | ~13 POSTs              | One template per unique supplier URL                               |
| Step 2 (child endpoints)    | ~4,000+                   | ~4,000+ POSTs          | One child endpoint per service ID (with period from int_endpoints) |
| Step 3 (HealthcareServices) | ~4,000+                   | ~4,000+ POSTs          | One per service ID                                                 |
| Step 4 (validation)         | ~4,000+                   | ~4,000+ GETs           | One query per service ID                                           |
| **Total**                   |                           | **~12,000+ API calls** |                                                                    |

At ~10 requests/second, estimated runtime: ~20 minutes.

This is significantly fewer API calls than the full int_ table migration because:

- Only ~13 Templates/Endpoints (not thousands — one per URL, not one per service)
- No inactive/placeholder data to process

---

### Error Handling


| Error                                                            | Action                                                                                                                                                                                       |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **URL not found in url_metadata** (enrichment miss)              | Log warning. If ProductId/ODS can be inferred from URL pattern (e.g.,`ygm04` in hostname), use that. Otherwise skip and flag for manual resolution.                                          |
| **ProductId not in PRODUCT_ID_MAP**                              | Log as unmapped, skip the template creation, and all services using that URL will fail in Step 3. Add to "needs mapping" report.                                                             |
| **provider_lookup miss** (service not in int_healthcareservices) | Log as migration gap. Create HealthcareService without`providedBy` temporarily but flag as **must-resolve** before go-live. These gaps must be zero for migration to be considered complete. |
| Template POST fails (409 conflict / already exists)              | Query by ProductId to get existing catalog_id. Use that in template_log. Continue.                                                                                                           |
| Endpoint POST fails                                              | Log and skip. Services referencing this URL will have no endpoint reference.                                                                                                                 |
| HealthcareService POST fails                                     | Log service_id and error. Continue with next.                                                                                                                                                |
| API rate limit (429)                                             | Exponential backoff with jitter. Retry up to 3 times.                                                                                                                                        |
| Validation: URL mismatch                                         | Log full detail (expected vs actual). Common causes: case difference, trailing slash, scheme mismatch.                                                                                       |

---

## Key Differences from Full int_ Table Migration


| Aspect                     | Full Migration (int_ tables)                       | targets.json Migration                                              |
| ---------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------- |
| Source of truth            | DynamoDB tables                                    | targets.json flat file                                              |
| Templates created          | One per template row (~50-100)                     | One per unique URL (~13)                                            |
| Child Endpoints created    | One per endpoint row (~5,000)                      | One per unique URL (~13)                                            |
| HealthcareServices created | One per HCS row (~5,000, includes inactive)        | One per targets.json entry (~4,000, all active)                     |
| Inactive services included | Yes                                                | No — only actively routed services                                 |
| Provider organisation      | From int_healthcareservices.ProviderOrganisationId | Required — resolved from int_healthcareservices in Step 0b         |
| Service name               | From int_healthcareservices.Name                   | Required — resolved from int_healthcareservices in Step 0b         |
| Endpoint per service       | Dedicated endpoint per service                     | Dedicated endpoint per service (period resolved from int_endpoints) |
| Total API calls            | ~14,000                                            | ~12,000+                                                            |
| Complexity                 | Higher (more data, more lookups)                   | Lower (flat file drives everything)                                 |

---

## Step 5: Optional Delta Detection — Generate CSV Files for Missing Items

After Step 4 validation (or as a standalone process), compare targets.json against the current EPC state and generate CSV files in the IP001/IP002/IP003 format for any items that are missing. These CSVs can be handed to the R&M team to onboard the missing data via the standard pipeline.

### 5a. Query the EPC for all service IDs in targets.json

```python
missing_templates = []    # URLs with no Template in the EPC
missing_endpoints = []    # Services with no child Endpoint in the EPC
missing_hcs = []          # Services with no HealthcareService in the EPC

for service_id, expected_url in service_to_url.items():
    # Query EPC for this service
    response = requests.get(
        f"{EPC_BASE}/HealthcareService",
        params={
            "identifier": f"https://fhir.nhs.uk/Id/dos-service-id|{service_id}",
            "_include": "HealthcareService:endpoint"
        },
        headers={"Authorization": f"Bearer {token}"}
    )
  
    bundle = response.json()
    entries = bundle.get("entry", [])
  
    # Check if HealthcareService exists
    hcs_found = any(
        e["resource"]["resourceType"] == "HealthcareService" 
        for e in entries if "resource" in e
    )
  
    # Check if active Endpoint exists with correct address
    endpoint_found = False
    for entry in entries:
        resource = entry.get("resource", {})
        if (resource.get("resourceType") == "Endpoint" 
            and resource.get("status") == "active"
            and urls_match(resource.get("address", ""), expected_url)):
            endpoint_found = True
            break
  
    if not hcs_found:
        missing_hcs.append(service_id)
  
    if not endpoint_found:
        missing_endpoints.append(service_id)

# Check for missing Templates (unique URLs with no Template)
for url in unique_urls:
    normalised = url.lower().rstrip('/')
    metadata = url_metadata.get(normalised, {})
    product_id = PRODUCT_ID_MAP.get(metadata.get('product_id', '').upper(), '')
    
    if not product_id:
        # Can't look up — no product ID mapping
        missing_templates.append(url)
        continue
    
    # Query EPC for Template by Product ID
    response = requests.get(
        f"{EPC_BASE}/Endpoint/$template",
        params={
            "Endpoint.identifier": f"https://fhir.nhs.uk/id/product-id|{product_id}"
        },
        headers={"Authorization": f"Bearer {token}"}
    )
    if response.status_code != 200 or not response.json().get("entry"):
        missing_templates.append(url)

print(f"Missing Templates: {len(missing_templates)}")
print(f"Missing Endpoints: {len(missing_endpoints)}")
print(f"Missing HealthcareServices: {len(missing_hcs)}")
```

### 5b. Generate IP002 CSV — Missing Endpoint Templates

For each URL in `missing_templates`, generate a row in the IP002 format.

**CSV format (per IP002):**


| Column      | Source                                             | How to populate                   |
| ------------- | ---------------------------------------------------- | ----------------------------------- |
| `ODSCode`   | `url_metadata[url].managing_org_ods`               | Supplier ODS code from enrichment |
| `ProductId` | `url_metadata[url].product_id` → `PRODUCT_ID_MAP` | Resolved EPC Product Identifier   |
| `Address`   | `url` from targets.json                            | The endpoint URL                  |

```python
import csv
from datetime import datetime

timestamp = datetime.now().strftime("%Y-%m-%dT%H%M%S")

if missing_templates:
    filename = f"epc-endpoint-template-create-{timestamp}.csv"
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['ODSCode', 'ProductId', 'Address'])
      
        for url in missing_templates:
            normalised = url.lower().rstrip('/')
            metadata = url_metadata.get(normalised, {})
            ods_code = metadata.get('managing_org_ods', 'UNKNOWN')
            product_id = PRODUCT_ID_MAP.get(
                metadata.get('product_id', '').upper(), 'UNKNOWN'
            )
            writer.writerow([ods_code, product_id, url])
  
    print(f"Generated: {filename} ({len(missing_templates)} rows)")
```

**Example output:**

```csv
ODSCode,ProductId,Address
YGM04,CegedimPharmacyServices-v6.0,https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/
```

### 5c. Generate IP003 CSV — Missing Endpoints

For each service_id in `missing_endpoints`, generate a row in the IP003 format.

**CSV format (per IP003):**


| Column        | Source                                             | How to populate                                                  |
| --------------- | ---------------------------------------------------- | ------------------------------------------------------------------ |
| `ODSCode`     | `url_metadata[url].managing_org_ods`               | Supplier ODS from enrichment (based on the URL for this service) |
| `ProductId`   | `url_metadata[url].product_id` → `PRODUCT_ID_MAP` | Resolved EPC Product Identifier                                  |
| `ServiceId`   | `service_id` from targets.json                     | The DoS service ID                                               |
| `Name`        | `provider_lookup[service_id].name`                 | Service name from int_healthcareservices (or blank)              |
| `Status`      | `"active"`                                         | Always active — it's in the live routing file                   |
| `PeriodStart` | `endpoint_details[service_id].start_date`          | From int_endpoints, or blank for R&M to fill in                  |
| `PeriodEnd`   | (blank)                                            | Open-ended — service is actively routed                         |

```python
if missing_endpoints:
    filename = f"epc-endpoint-create-{timestamp}.csv"
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['ODSCode', 'ProductId', 'ServiceId', 'Name', 'Status', 'PeriodStart', 'PeriodEnd'])
      
        for service_id in missing_endpoints:
            url = service_to_url[service_id]
            normalised = url.lower().rstrip('/')
            metadata = url_metadata.get(normalised, {})
          
            ods_code = metadata.get('managing_org_ods', 'UNKNOWN')
            product_id = PRODUCT_ID_MAP.get(
                metadata.get('product_id', '').upper(), 'UNKNOWN'
            )
          
            # Service name from provider_lookup
            provider = provider_lookup.get(service_id, {})
            name = provider.get('name', '')
          
            # Period from endpoint_details
            details = endpoint_details.get(service_id, {})
            period_start = details.get('start_date', '')
          
            writer.writerow([ods_code, product_id, service_id, name, 'active', period_start, ''])
  
    print(f"Generated: {filename} ({len(missing_endpoints)} rows)")
```

**Example output:**

```csv
ODSCode,ProductId,ServiceId,Name,Status,PeriodStart,PeriodEnd
YGM04,CegedimPharmacyServices-v6.0,2000017562,Pharm+: Boots Pharmacy Bromley,active,2026-06-01T16:04:05.168Z,
```

### 5d. Generate IP001 CSV — Missing HealthcareServices

For each service_id in `missing_hcs`, generate a row in the IP001 format.

**CSV format (per IP001):**


| Column        | Source                                     | How to populate                                                                             |
| --------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `ODSCode`     | `provider_lookup[service_id].provider_ods` | Provider ODS (pharmacy/hospital) — NOT the supplier                                        |
| `ServiceId`   | `service_id` from targets.json             | The DoS service ID                                                                          |
| `ServiceName` | `provider_lookup[service_id].name`         | Service name from int_healthcareservices (or blank for R&M to fill in)                      |
| `EndpointId`  | (blank)                                    | Left blank — R&M will associate after Endpoint is created, or the pipeline auto-associates |

```python
if missing_hcs:
    filename = f"epc-healthcareservice-create-{timestamp}.csv"
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['ODSCode', 'ServiceId', 'ServiceName', 'EndpointId'])
      
        for service_id in missing_hcs:
            provider = provider_lookup.get(service_id, {})
            ods_code = provider.get('provider_ods', 'UNKNOWN')
            name = provider.get('name', '')
          
            writer.writerow([ods_code, service_id, name, ''])
  
    print(f"Generated: {filename} ({len(missing_hcs)} rows)")
```

**Example output:**

```csv
ODSCode,ServiceId,ServiceName,EndpointId
FE284,2000017562,Pharm+: Boots Pharmacy Bromley,
```

### 5e. Delta Summary Report

```python
report = {
    "run_date": timestamp,
    "source": "targets.json",
    "total_services_in_targets": len(service_to_url),
    "delta": {
        "missing_templates": {
            "count": len(missing_templates),
            "csv_file": f"epc-endpoint-template-create-{timestamp}.csv" if missing_templates else None,
            "urls": missing_templates
        },
        "missing_endpoints": {
            "count": len(missing_endpoints),
            "csv_file": f"epc-endpoint-create-{timestamp}.csv" if missing_endpoints else None,
        },
        "missing_healthcareservices": {
            "count": len(missing_hcs),
            "csv_file": f"epc-healthcareservice-create-{timestamp}.csv" if missing_hcs else None,
        }
    },
    "action_required": "Upload generated CSVs to S3 for R&M pipeline processing"
}

with open(f"delta-report-{timestamp}.json", 'w') as f:
    json.dump(report, f, indent=2)
```

### 5f. Handling UNKNOWN values

Any row with `UNKNOWN` in the `ODSCode` or `ProductId` column indicates data that could not be resolved from the enrichment sources. The R&M team must manually fill these in before uploading the CSV to the pipeline.


| Column with UNKNOWN                    | Resolution                                                                          |
| ---------------------------------------- | ------------------------------------------------------------------------------------- |
| `ODSCode` in IP002 (Template)          | Contact supplier to confirm their ODS code                                          |
| `ProductId` in IP002/IP003             | Check Product ID mapping table; may need onboarding with Digital Onboarding Service |
| `ODSCode` in IP001 (HealthcareService) | Look up service in DoS to find the providing organisation                           |
| `Name` (blank) in IP001/IP003          | Look up service name in DoS or ask commissioner                                     |

### Delta Process Sequence Diagram

```mermaid
sequenceDiagram
    participant Script as Delta Script
    participant TJ as targets.json
    participant EPC as EPC API
    participant DDB as DynamoDB (int_ tables)
    participant CSV as Output CSVs

    Note over Script: Load source data
    Script->>TJ: Load targets.json
    Script->>Script: Extract service_to_url + unique_urls
    Script->>DDB: Scan int_organisations (org_lookup)
    Script->>DDB: Scan int_endpoint_templates (url_metadata)
    Script->>DDB: Scan int_endpoints (endpoint_details)
    Script->>DDB: Scan int_healthcareservices (provider_lookup)

    Note over Script: Check Templates (by ProductId)
    loop For each unique URL
        Script->>Script: Resolve ProductId from url_metadata
        Script->>EPC: GET /Endpoint/$template?Endpoint.identifier=https://fhir.nhs.uk/id/product-id|{product_id}
        EPC-->>Script: 200 Found / 4XX Not Found
        alt Not Found
            Script->>Script: Add to missing_templates
        end
    end

    Note over Script: Check Services + Endpoints
    loop For each service_id in targets.json
        Script->>EPC: GET /HealthcareService?identifier=...{service_id}&_include=endpoint
        EPC-->>Script: Bundle (may be empty)
        alt No HealthcareService in bundle
            Script->>Script: Add to missing_hcs
        end
        alt No active Endpoint with matching address
            Script->>Script: Add to missing_endpoints
        end
    end

    Note over Script: Generate CSVs for R&M
    alt missing_templates > 0
        Script->>Script: Resolve ODSCode + ProductId from url_metadata
        Script->>CSV: Write epc-endpoint-template-create-{timestamp}.csv (IP002)
    end
    alt missing_endpoints > 0
        Script->>Script: Resolve ODSCode, ProductId, PeriodStart from enrichment
        Script->>CSV: Write epc-endpoint-create-{timestamp}.csv (IP003)
    end
    alt missing_hcs > 0
        Script->>Script: Resolve provider ODS + name from provider_lookup
        Script->>CSV: Write epc-healthcareservice-create-{timestamp}.csv (IP001)
    end

    Script->>CSV: Write delta-report-{timestamp}.json
    Note over CSV: R&M reviews CSVs, fills UNKNOWN values, uploads to S3 pipeline
```

---


## Key Decisions


| Decision                                               | Choice                                                              | Rationale                                                                                                      |
| -------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| One child Endpoint per URL (shared) vs one per service | One per URL (shared)                                                | targets.json shows many services using the same URL. Creating thousands of identical endpoints is wasteful.    |
| All services set to`active: true`                      | Yes                                                                 | They're in the live routing file — by definition, they're active.                                             |
| `providedBy` optional                                  | No — required                                                      | Provider resolution is built in Step 0b from int_healthcareservices. Any gaps must be resolved before go-live. |
| `name` optional                                        | No — required (with fallback for gaps)                             | Resolved from int_healthcareservices in Step 0b. Missing names are logged as gaps.                             |
| URL matching strategy                                  | Case-insensitive, strip trailing slash, strip scheme for comparison | Source data has inconsistent casing                                                                            |
| Period.start for child endpoints                       | Resolve from int_endpoints (Option 2), migration date as fallback   | Preserves historical start date where available; ensures no missing values                                     |

---

## Comparison: Which Approach to Use?


| Use Case                                                            | Recommended Approach                           |
| --------------------------------------------------------------------- | ------------------------------------------------ |
| Need to reproduce live routing ASAP with minimal risk               | **targets.json approach** (this document)      |
| Need full historical data including inactive services               | Full int_ table migration                      |
| Need dedicated endpoint per service (for future per-service config) | Full int_ table migration                      |
| Need provider organisation and service names                        | Either (with enrichment), or full migration    |
| Proof-of-concept / demo                                             | **targets.json approach** (faster, simpler)    |
| Production migration (final)                                        | Full int_ table migration (more complete data) |

---

## Files & Outputs


| Artifact              | Location                         | Purpose                          |
| ----------------------- | ---------------------------------- | ---------------------------------- |
| Migration script      | TBD                              | Executes Steps 0-3               |
| Validation script     | TBD                              | Executes Step 4                  |
| Migration log         | `migration-log-targets.json`     | Source → catalog_id mappings    |
| Validation report     | `validation-report-targets.json` | Diff between expected and actual |
| Reconstructed targets | `targets-reconstructed.json`     | Rebuilt from EPC queries         |
