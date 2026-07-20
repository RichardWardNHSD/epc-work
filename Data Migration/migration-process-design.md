# Migration Process Design: Flat File Endpoints to EPC

## Purpose

This document describes the process for migrating the current flat-file-based endpoint routing (`targets.json`) into the new Endpoint Catalogue (EPC) as proper FHIR resources.

## Current State

The BaRS proxy currently routes referrals using `targets.json` — a flat JSON file that maps DoS service IDs directly to supplier endpoint URLs:

```
"<dos-service-id>": "<supplier-url>"
```

This file contains approximately 4,000+ service-to-URL mappings under the identifier system `https://fhir.nhs.uk/Id/dos-service-id`.

## Target State

Each service should be represented as a `HealthcareService` resource in the EPC, linked to an `Endpoint` resource that is managed by the correct supplier `Organization`.

---

## Unique Suppliers (URLs) in targets.json

From analysis of the targets.json data, the unique receiver URLs are:

| URL | Supplier Name | ProductId (from int_endpoint) | ManagingOrganisationId |
|-----|--------------|-------------------------------|----------------------|
| `https://bars-prod-ygm06-pharmoutcomes.emis.thirdparty.nhs.uk` | Pharmoutcomes (EMIS) | ygm06 | `7adf4fb8-da26-4e3e-b68e-3daecb61e7c5` |
| `https://bars-prod-ygm04.cegedim.thirdparty.nhs.uk/FHIR/R4/` | Cegedim | ygm04 | `2f594ac5-6bc8-4241-af41-ae0f92b88949` |
| `https://bars-prod-8hk48.sonar.thirdparty.nhs.uk` | Sonar | 8hk48 | `2af3b6ed-c929-4b5d-be4b-1dab43c4c514` |
| `https://bars-prod-ygm17.positivesolutions.thirdparty.nhs.uk` | Positive Solutions | ygm17 | `813a1b53-f962-445f-965e-e61c6a8bf969` |
| `https://BaRS-PROD-8JY34.WASPSoftware.thirdparty.nhs.uk/api/r4` | WASP | 8JY34 | `63bdc32b-fb44-4332-a7a0-b3c585957ea3` |
| `https://BARS-PROD-AC0.advanced.thirdparty.nhs.uk/fhirR4/api/bars` | Advanced | AC0 | (lookup needed) |
| `https://bars-prod-8hq44.stratahealth.thirdparty.nhs.uk:3120/bars/` | Strata Health | 8hq44 | (lookup needed) |
| `https://BARS-PROD-Y01061.advanced.thirdparty.nhs.uk/fhirr4/api/bars` | Advanced (Y01061) | Y01061 | (lookup needed) |
| `https://bars-prod-8ht86.fortrus.thirdparty.nhs.uk/` | Fortrus | 8ht86 | (lookup needed) |
| `https://bars-prod-rk5.nervecentre.thirdparty.nhs.uk` | Nervecentre | rk5 | (lookup needed) |
| `https://bars-prod-rk5.nervecentre.thirdparty.nhs.uk:9999/booking-and-referral/FHIR/R4/` | Nervecentre (alt) | rk5 | (lookup needed) |
| `https://BARS-PROD-GA9.gmupca.nhs.uk/fhirr4/api/bars` | GMUPCA | GA9 | (lookup needed) |
| `https://BaRS-PROD-RX7.NWAS.nhs.uk` | NWAS | RX7 | (lookup needed) |

---

## Migration Process

### Step 1: Identify Supplier from URL (using int_endpoint)

The `int_endpoint` table is the key to resolving which supplier owns a given URL. The relevant fields are:

| Field | Purpose |
|-------|---------|
| `Address` | The endpoint URL — **match this against unique URLs in targets.json** |
| `Name` | The supplier name (e.g., "Cegedim", "Pharmoutcomes", "Sonar") |
| `ManagingOrganisationId` | UUID of the supplier Organisation in `int_organisations` |
| `ProductId` | The product code (e.g., "ygm04", "ygm06", "8hk48") |
| `TemplateId` | The endpoint template UUID (defines capabilities/interaction patterns) |

**Process:**
1. Extract all unique URLs from `targets.json` (case-insensitive comparison recommended as URLs in int_endpoint are sometimes mixed-case)
2. For each unique URL, find a matching row in `int_endpoints.csv` where `Address` matches (case-insensitive)
3. From the matching row, extract:
   - `Name` → supplier name
   - `ManagingOrganisationId` → look this up in `int_organisations` to get the ODS code
   - `ProductId` → used for the Endpoint's `identifier`
   - `TemplateId` → links to capabilities defined in `int_endpoint_templates`

**Note:** Many endpoints in `int_endpoints.csv` have `Address = "addressHere"` (placeholder). Only rows with real URLs are useful for this lookup. The rows that have the actual URL populated also have the `EndpointTemplate` field filled with the full template JSON, confirming they are "template-derived" endpoints.

### Step 2: Create HealthcareService resources

For each unique DoS service ID in `targets.json`:

1. Create a `HealthcareService` FHIR resource with:
   - `identifier`: system = `https://fhir.nhs.uk/Id/dos-service-id`, value = the service ID
   - `active`: true
   - `providedBy`: reference to the Organisation (resolved from Step 1's ManagingOrganisationId, but note the provider org may differ from the supplier — this needs investigation for non-pharmacy services)

2. Link the HealthcareService to the appropriate Endpoint

### Step 3: Create/Verify Endpoint resources

For each unique URL (i.e., per supplier):

1. Create or verify an `Endpoint` FHIR resource with:
   - `address`: the URL
   - `connectionType`: BARS
   - `managingOrganization`: reference to the supplier Organisation
   - `identifier`: the ProductId

2. The Endpoint should be linked to the relevant `EndpointTemplate` (defines what message types/use cases it supports)

### Step 4: Link HealthcareService to Endpoint

Each HealthcareService needs to reference the Endpoint that serves it. This is the core of what targets.json provides:
- Service ID → URL → Endpoint resource

---

## Key Questions / Decisions

1. **Provider vs Supplier Organisation**: The `ManagingOrganisationId` in int_endpoint is the *supplier* (e.g., Cegedim, EMIS). The HealthcareService's `providedBy` should be the *provider* (e.g., the pharmacy). Do we need to populate provider Organisation separately?

2. **Placeholder addresses**: Many int_endpoint rows have `Address = "addressHere"`. These are endpoints that were onboarded but never had their URL configured (likely still using the flat file). The migration will populate these.

3. **Case sensitivity**: URLs in targets.json vs int_endpoint sometimes differ in casing (e.g., `https://BaRS-PROD-8JY34...` vs `https://bars-prod-8jy34...`). Matching should be case-insensitive.

4. **Deduplication**: Multiple services share the same endpoint URL. In the EPC model, there should be one Endpoint resource per unique URL, with multiple HealthcareServices referencing it.

5. **EndpointTemplate linkage**: Should each Endpoint in the EPC reference its template? The TemplateId in int_endpoint provides this mapping.

---

## Data Volume Summary

- ~4,000+ DoS service IDs in targets.json
- ~13 unique supplier URLs
- ~6 confirmed suppliers (from int_endpoint data with real addresses)
- Remaining suppliers need ManagingOrganisationId lookup from full int_endpoints dataset

## Execution Order

1. Ensure all supplier Organisations exist in EPC (from int_organisations)
2. Create Endpoint resources for each unique supplier URL
3. Create HealthcareService for each DoS service ID
4. Link each HealthcareService to its Endpoint
