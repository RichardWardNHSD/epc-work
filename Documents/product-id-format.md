# Product ID Format — Unresolved Blocker

## Status

**Unresolved — blocks data migration and all examples are illustrative only.**

---

## What is a Product ID?

A Product ID is a unique identifier issued by the NHS England Digital Onboarding Service
for a specific software product assured by NHS England. It is used throughout the Endpoint
Catalog for:

- **Template identity** — the combination of `productId` + `connectionType` + `payloadType`
  uniquely identifies a Template
- **Ownership and authorisation** — Product ID + ODS code together form the ownership key;
  write operations check that the caller's token Product ID matches the resource's
- **Delegated authority** — a supplier's Product ID on a HealthcareService grants them
  write access on behalf of the provider organisation
- **Duplicate detection** — Template uniqueness depends on `productId`
- **Data migration** — every Template in the Endpoint Catalog requires a `productId`

---

## The problem

The format of the Product ID has not been confirmed. The documentation currently uses
an illustrative format (`PinnaclePharmOutcomes-v2024.12.12`) which is a concatenation
of product name and version. This is assumed but not validated.

The existing BaRS endpoint database holds only the **supplier name** — it does not have
a versioned product identifier. This means:

1. The migration pipeline cannot construct a valid `productId` from the source data
2. All CSV examples in the template management documentation are illustrative
3. Validation rules (regex, max length) cannot be defined for the API
4. The OAS query parameter examples use a placeholder format

---

## Questions that need answering

| # | Question | Who needs to answer | Impact if unresolved |
|---|----------|--------------------|--------------------|
| 1 | What is the format of a Product ID? (e.g. `{Name}-v{Version}`, UUID, opaque string, other) | Digital Onboarding Service | Cannot define API validation rules or schema constraints |
| 2 | Does the Product ID include a version component? | Digital Onboarding Service | Determines whether a new product version = new Product ID or same Product ID |
| 3 | How are Product IDs obtained for existing suppliers during migration? | Digital Onboarding Service + BaRS team | **Blocks data migration** — source data only has supplier name |
| 4 | Is the Product ID mutable? Can it change when a supplier releases a new version? | Digital Onboarding Service | Affects whether version upgrades require new Templates or updates to existing ones |
| 5 | Is there a registry or API to look up existing Product IDs by supplier name? | Digital Onboarding Service | Needed for migration tooling to resolve supplier names to Product IDs |
| 6 | What is the maximum length? | Digital Onboarding Service | Needed for DynamoDB key design and API validation |
| 7 | Are there character restrictions? (alphanumeric only, hyphens allowed, etc.) | Digital Onboarding Service | Needed for API validation regex |

---

## Documents affected

Once the format is confirmed, the following documents need updating:

| Document | What needs changing |
|----------|-------------------|
| Managing Endpoint Templates | All CSV examples (create, update, soft delete, hard delete) use `PinnaclePharmOutcomes-v2024.12.12` — replace with real format. Remove ⚠️ warnings. |
| Data Migration to the Endpoint Catalog | Template mapping table, construction mechanism note, payload examples. Remove ⚠️ warnings. Define the migration lookup/construction step. |
| Endpoint Templates — Pros and Cons | Con 2 ("ProductId format unresolved") can be removed or marked resolved. |
| Duplicate Detection in the Endpoint Catalog | Examples use illustrative format — update to real format. |
| Authentication and Authorisation | Define validation rules for Product ID in ownership checks. |
| BaRS OAS (`bars api OAS.json`) | `EndpointIdentifier_QParam` example uses `SupplierProduct-v2024.12.12` — update. Schema for `identifier[].value` on Endpoint/Template needs format constraint. |
| Open Questions | Items #12, #14, #56 can be consolidated and closed. |

---

## Proposed next steps

1. **Contact the Digital Onboarding Service** to confirm the format, versioning model,
   and whether a lookup API exists
2. **Check whether existing suppliers already have Product IDs** issued through the
   onboarding process — if so, obtain a mapping of supplier name → Product ID
3. **If no Product IDs exist for current suppliers**, agree a mechanism to issue them
   (bulk issuance as part of migration prep)
4. **Once confirmed**, update all affected documents and remove the ⚠️ warnings
5. **Unblock data migration** — the migration pipeline's Step 1 (Template creation)
   depends entirely on this

---

## Current illustrative format (not confirmed)

The documentation currently assumes:

```
{ProductName}-v{Version}
```

Examples:
- `PinnaclePharmOutcomes-v2024.12.12`
- `SonarBaRS-v3.1.0`
- `MedRecPharmacy-v2025.01`

The `identifier` system is:
```
https://fhir.nhs.uk/id/product-id
```

This system URI is confirmed (it's the NHS England standard for product identifiers).
Only the **value format** is unresolved.
