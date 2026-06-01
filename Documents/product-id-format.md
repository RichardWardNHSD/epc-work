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

The documentation currently uses an illustrative format for the Product ID value:

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

## Where the Product ID value comes from

The Product ID value could come from one of two sources:

### Option 1 — Issued by the Digital Onboarding Service (preferred)

The [Digital Onboarding Service](https://digital.nhs.uk/services/digital-onboarding-service)
is the NHS England assurance process for organisations and suppliers integrating with
NHS APIs. Key points relevant to Product ID:

1. **Each API is assigned to one or more products, and each product is linked to an
   organisation.** This confirms that a Product ID is a per-product, per-organisation
   concept — not per-API or per-endpoint.

2. **The onboarding process is product-centric.** An applicant registers their
   organisation and then registers the product they want to onboard. The Product ID
   is issued as part of this registration.

3. **The Product ID is issued by the Digital Onboarding Service** during Step 2
   ("Register your organisation and product"). It is not self-assigned by the supplier.

4. **The service is the primary route for NHS API assurance.** Any supplier wanting to
   use NHS APIs (including the Endpoint Catalog) must go through this process and will
   receive a Product ID as an output.

This is the preferred source because it provides a centrally managed, unique, and
authoritative identifier with a clear issuance process.

### Option 2 — Concatenation of supplier name, software name, and version

If a formally issued Product ID is not available (e.g. for legacy suppliers who
pre-date the Digital Onboarding Service, or during an interim period before onboarding
is complete), the Product ID value could be constructed as a concatenation:

```
{SupplierName}-{SoftwareName}-v{Version}
```

Examples:
- `Sonar-PharmacyBaRS-v2024.12.12`
- `Pharmoutcomes-CPCS-v3.1.0`
- `MedRec-PharmacyFirst-v2025.01`

This approach has significant drawbacks:
- **No central authority** — the format is self-assigned, risking inconsistency
- **Version in the identifier** — contradicts the version-agnostic recommendation
  (see [Versioning](#versioning-what-happens-when-a-supplier-releases-a-new-software-version)
  above); every version upgrade would change the Product ID
- **No uniqueness guarantee** — two suppliers could independently construct the same string
- **Migration complexity** — the existing database only has supplier name, not a
  standardised concatenation

### Why the FHIR Identifier approach handles both

Regardless of whether the Product ID value comes from the Digital Onboarding Service
or is a constructed concatenation, the EPC treats it the same way: as an **opaque string**
stored in a FHIR Identifier with system `https://fhir.nhs.uk/id/product-id`.

The EPC does not parse, validate, or interpret the value's internal format. It simply
stores it and matches on it. This means:

- If the Digital Onboarding Service issues `PROD-12345`, the EPC stores and matches on
  `PROD-12345`
- If a legacy migration uses `Sonar-PharmacyBaRS-v2024.12.12`, the EPC stores and
  matches on that string
- The EPC doesn't care which source produced the value — it's just an identifier

This decoupling is the key advantage of the Identifier approach. The EPC is insulated
from the decision about where Product IDs come from and what format they take.

### Recommendation

Use Digital Onboarding Service-issued Product IDs wherever possible. Fall back to
constructed concatenations only for legacy data migration where no formal ID exists.
In either case, the EPC treats the value as opaque and the `system|value` Identifier
pattern applies consistently.

### Implications for the EPC

- **Product IDs already exist** for any supplier that has been through the Digital
  Onboarding process. The format is whatever the onboarding service issues — the EPC
  does not need to define or validate it beyond treating it as an opaque string.
- **For existing BaRS suppliers**, Product IDs should already have been issued during
  their original BaRS onboarding. The migration team needs to obtain the mapping of
  supplier name → Product ID from the Digital Onboarding Service's records.
- **For new suppliers**, the Product ID will be issued as part of their onboarding
  before they can interact with the EPC. This is a natural prerequisite — no additional
  process is needed.
- **The EPC's role is to store and match on the Product ID**, not to issue or validate
  its format. This reinforces the proposed approach of treating it as an opaque
  `system|value` identifier.

### Remaining gap

The Digital Onboarding Service page does not publish the format of the Product ID value
it issues. The migration team needs to:

1. Contact the Digital Onboarding Service (Shan Rahulan — shan.rahulan@nhs.net) to
   obtain the list of Product IDs already issued to BaRS suppliers
2. Confirm the format so that documentation examples can be updated
3. Confirm whether a lookup API exists to resolve supplier name → Product ID
   programmatically

---

## Versioning: what happens when a supplier releases a new software version?

This is a critical design question. The answer determines whether a new software version
requires a new Template (heavy migration) or an in-place update (lightweight).

### Two models

| Model | Product ID changes on new version? | Impact on Templates | Impact on child Endpoints |
|-------|-----------------------------------|--------------------|--------------------------| 
| **Version-inclusive** (e.g. `PinnaclePharmOutcomes-v2024.12.12`) | Yes — new version = new Product ID | New Template must be created; old Template retired | All child Endpoints must be migrated from old Template to new Template (one by one) |
| **Version-agnostic** (e.g. `PROD-12345` stays the same) | No — Product ID is stable across versions | Existing Template updated in place (`PUT /Endpoint/{id}/$template`) | No action needed — child Endpoints automatically inherit the updated Template fields at read time |

### Why version-agnostic is the right model

The Template design was built specifically to support the version-agnostic model. The
core benefit of Templates is:

> *"If the supplier changes their URL or upgrades their product, only the Template needs
> updating — the change is immediately visible on every child Endpoint without any
> further writes."*

If Product ID changes on every version, this benefit is lost. A version upgrade would
require:

1. Creating a new Template with the new Product ID
2. Creating new child Endpoints for every HealthcareService currently using the old Template
3. Updating every HealthcareService's `endpoint[]` references
4. Updating every List to reference the new Endpoints
5. Retiring the old Template and its child Endpoints

For a supplier like Sonar or Pharmoutcomes with hundreds of pharmacy services, this is
an enormous operational burden — exactly the kind of manual, error-prone process the
Endpoint Catalog was designed to eliminate.

With a version-agnostic Product ID, a version upgrade is simply:

1. Supplier calls `PUT /Endpoint/{id}/$template` with the new `address` (if the URL changed)
2. Done — all child Endpoints immediately resolve the new address

### What version-agnostic means in practice

- **Product ID = the product**, not a specific release (e.g. "Sonar BaRS" not "Sonar BaRS v3.1.2")
- **The Template's `address` field** is what changes when a supplier deploys a new version
  (if the URL changes at all — many version upgrades don't change the URL)
- **Version tracking** (if needed for audit or reporting) should be handled separately —
  e.g. as a `meta.versionId` on the Template resource, or as an additional identifier
  with a version-specific system
- **Breaking changes** (where the new version is incompatible and requires a different
  `connectionType` or `payloadType`) would legitimately require a new Template — but
  this is a new *capability*, not just a new version of the same capability

### Recommendation

The Product ID should be **version-agnostic** — a stable identifier for the product that
does not change across software releases. This aligns with:

- The Template model's core design (single point of update)
- The Digital Onboarding Service's model (you onboard a *product*, not a *version*)
- The requirements (EPCFUNC-15: updating a Template cascades to all child Endpoints)

The illustrative format `PinnaclePharmOutcomes-v2024.12.12` in the current documentation
is misleading because it implies the version is part of the identifier. This should be
updated to a version-free format once confirmed (e.g. `PinnaclePharmOutcomes` or
`PROD-12345`).

---

## What happens when a supplier releases a new software version?

### Requirements V1.8 position

The requirements document (Scenario 2, Section 2.3.2) acknowledges that suppliers may
have "solution version differences to support breaking changes" and states there will be
"multiple templates (one for each connection type and version of the deployed solution)."

However, the requirements define Product ID as identifying "the specific software product
assured by NHS England" — not a specific version of that product. The document does not
explicitly state whether a new version results in a new Product ID.

### Guidance by upgrade type

Not all version upgrades are equal. The impact on the Endpoint Catalog depends on whether
the upgrade is **non-breaking** or **breaking**:

#### Non-breaking version upgrade (most common)

A supplier deploys a new version of their software that is backwards-compatible — same
connection type, same payload type, same (or updated) URL.

| What changes | Action in the EPC | Product ID changes? |
|-------------|-------------------|-------------------|
| Internal software version only (no URL change) | Nothing — the EPC is unaware of internal versions | No |
| URL changes (e.g. new base path) | `PUT /Endpoint/{id}/$template` to update `address` | No |
| URL changes + name update | `PUT /Endpoint/{id}/$template` to update `address` and `name` | No |

**Result:** The existing Template is updated in place. All child Endpoints automatically
inherit the new address at read time. No action needed on HealthcareServices, Lists, or
child Endpoints. This is the Template model working as designed.

#### Breaking version upgrade (new capability)

A supplier deploys a new version that introduces a fundamentally different capability —
a new connection type, a new payload type, or both. The old and new versions are
incompatible and may need to coexist during a transition period.

| What changes | Action in the EPC | Product ID changes? |
|-------------|-------------------|-------------------|
| New `connectionType` (e.g. ITK → FHIR REST) | Create a **new Template** with the new connection type | No — same product, new capability |
| New `payloadType` (e.g. `bars` → `bars-v2`) | Create a **new Template** with the new payload type | No — same product, new capability |
| Entirely new product (different supplier, different onboarding) | New Template with a **new Product ID** | Yes — this is a different product |

**Result:** A new Template is created under the same Product ID (same supplier, same
product, new capability). The old Template remains active for services that haven't
migrated yet. Services migrate by switching their `HealthcareService.endpoint[]`
reference from the old Endpoint (child of old Template) to a new Endpoint (child of
new Template) via the DoS switch workflow.

### The Product ID stays the same — the Template differentiates

The key insight is that **Product ID identifies the supplier's product** while
**`connectionType` + `payloadType` differentiate the capabilities** within that product.
A supplier with one Product ID can have multiple Templates:

```
Product ID: PROD-SONAR-001 (Sonar BaRS product)
├── Template-A: connectionType=hl7-fhir-rest, payloadType=bars      (current)
├── Template-B: connectionType=hl7-fhir-rest, payloadType=bars-v2   (new version)
└── Template-C: connectionType=ihe-xds, payloadType=xds-referral    (different protocol)
```

All three Templates share the same Product ID. The duplicate detection rule
(`productId` + `connectionType` + `payloadType` must be unique among active Templates)
ensures no two active Templates represent the same capability for the same product.

### Implications for delegated authority

If Product ID is version-agnostic and stable across releases, the delegated authority
model (where a supplier's Product ID on a HealthcareService grants them write access)
works cleanly:

- **Non-breaking upgrade:** The supplier updates their Template. Their Product ID hasn't
  changed, so their delegated authority over HealthcareServices is unaffected. No action
  needed.
- **Breaking upgrade (new Template):** The supplier creates a new Template under the same
  Product ID. Their delegated authority is still valid — the Product ID on the
  HealthcareService still matches their token. They can update the HealthcareService's
  `endpoint[]` to reference the new Endpoint.
- **Supplier switch (different product):** The new supplier has a different Product ID.
  The HealthcareService's Product ID identifier must be updated from the old supplier's
  to the new supplier's — this is the mechanism that transfers delegated authority.

If Product ID were version-inclusive, every version upgrade would invalidate the
supplier's delegated authority over all their HealthcareServices (because the token's
Product ID would no longer match the resource's). This would require updating the
Product ID on every HealthcareService on every release — an unacceptable operational
burden.

### Summary

| Upgrade type | Product ID | Template | Child Endpoints | HealthcareService | Delegated authority |
|-------------|-----------|----------|----------------|-------------------|-------------------|
| Non-breaking (URL change) | Unchanged | Updated in place | Inherit automatically | No change | Unaffected |
| Non-breaking (no URL change) | Unchanged | No change | No change | No change | Unaffected |
| Breaking (new capability) | Unchanged | New Template created | New Endpoints created for migrating services | `endpoint[]` updated via DoS switch | Unaffected (same Product ID) |
| Supplier switch | New Product ID | Different supplier's Template | Different supplier's Endpoints | `endpoint[]` updated + Product ID identifier updated | Transfers to new supplier |

---

## Proposed approach: Product ID as a FHIR Identifier (system|value)

The proposed approach is to treat the Product ID as a standard FHIR Identifier and pass
it as a `system|value` token wherever it's used — the same pattern used for ODS codes,
DoS service IDs, and other identifiers throughout the API. The EPC treats the value as
an **opaque string** and does not parse, validate, or interpret its internal format.

### How it would work

The Product ID is already stored on resources as a FHIR Identifier:

```json
{
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PROD-12345"
    }
  ]
}
```

When passed as a query parameter or header value, it would use the standard `system|value`
token format (URL-encoded):

```
https://fhir.nhs.uk/id/product-id|PROD-12345
```

This means:
- The **system** (`https://fhir.nhs.uk/id/product-id`) identifies what kind of identifier it is
- The **value** (`PROD-12345`) is the opaque identifier issued by the Digital Onboarding Service
- The format of the value itself doesn't matter to the EPC — it's just a string to match on

### Where this applies

| Usage | Current approach | Identifier approach |
|-------|-----------------|-------------------|
| Query parameter (`Endpoint.identifier`) | Already uses `system|value` format | No change needed — already works this way |
| Token claim (Product ID in bearer token) | Assumed to be the value only | Could carry the full `system|value` or just the value (with system implied) |
| Ownership check | Match token's Product ID against resource's `identifier[].value` | Match token's value against `identifier[].value` where `identifier[].system` = `https://fhir.nhs.uk/id/product-id` |
| CSV input (migration/template creation) | `ProductId` column holds the value | No change — CSV holds the value; the system is always `https://fhir.nhs.uk/id/product-id` |
| Duplicate detection | Match on `productId` value | Match on `identifier[].value` where system matches |

### Advantages

- **Format-agnostic** — the EPC doesn't need to know or validate the internal format of
  the Product ID value. It's an opaque string issued by the Digital Onboarding Service.
  Whether it's `PROD-12345`, `PinnaclePharmOutcomes-v2024.12.12`, or a UUID doesn't
  matter to the catalog.
- **Consistent with other identifiers** — ODS codes, DoS service IDs, and endpoint
  identifiers all use the `system|value` pattern. Product ID follows the same convention.
- **Decouples the EPC from the Digital Onboarding Service's format decisions** — if they
  change the format in future, the EPC doesn't need updating.
- **Already partially implemented** — the resource schema already stores Product ID as
  a FHIR Identifier with system and value. The query parameter `Endpoint.identifier`
  already accepts `system|value` tokens.

### What this means for the unresolved questions

With this approach, several of the open questions become less critical for the EPC:

| Question | Impact |
|----------|--------|
| What is the format? | **Reduced** — the EPC treats it as an opaque string; format is the Digital Onboarding Service's concern |
| Does it include a version? | **Reduced** — the EPC doesn't parse or interpret the value; versioning is the issuer's concern |
| Maximum length? | Still needed for DynamoDB key design, but can use a generous limit (e.g. 256 chars) |
| Character restrictions? | Still needed for URL encoding safety, but standard URL-safe characters suffice |

The **migration blocker** (question #3 — how to obtain Product IDs for existing suppliers)
remains regardless of this approach. The Digital Onboarding Service still needs to either
issue Product IDs for existing suppliers or confirm that existing identifiers can be used.


---

## Product ID on HealthcareService — delegated authority

### The concept

The `authorisation.md` document proposes a **delegated authority** model where a supplier
can write to a HealthcareService on behalf of the provider organisation. The mechanism
relies on the supplier's Product ID being present as an identifier on the HealthcareService
resource:

```json
{
  "resourceType": "HealthcareService",
  "identifier": [
    { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" },
    { "system": "https://fhir.nhs.uk/id/product-id", "value": "PROD-SONAR-001" }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FH123"
    }
  }
}
```

The authorisation rule is:

> A caller may write to a HealthcareService if EITHER:
> 1. Their token ODS code matches `providedBy` (they ARE the provider), OR
> 2. Their token Product ID matches a Product ID identifier on the HealthcareService
>    (they are the supplier managing it on behalf of the provider)

### How it would be established

The Product ID gets onto the HealthcareService when the **supplier creates it on behalf
of the provider**. In practice:

1. Supplier onboards via the Digital Onboarding Service → receives a Product ID
2. Supplier creates a HealthcareService for a pharmacy they serve, setting:
   - `providedBy` = the pharmacy's ODS code (the pharmacy owns the service)
   - `identifier[]` includes their own Product ID (establishes delegated authority)
3. The supplier can now update the HealthcareService because their token's Product ID
   matches one in the resource's `identifier[]`
4. The pharmacy can also update it because their ODS code matches `providedBy`

### Current gaps — this is not yet implemented

This delegated authority model is documented as a design concept in `authorisation.md`
but has several unresolved aspects:

| # | Gap | Question |
|---|-----|----------|
| 1 | **HealthcareService schema** | The OAS and data model don't explicitly show a `product-id` identifier on HealthcareService. The schema needs updating if this is adopted. |
| 2 | **Who adds the Product ID?** | Is it set at creation time only? Can it be added later? Can the provider add it (granting a supplier access)? Can the supplier add it themselves (self-granting)? |
| 3 | **Who can remove it?** | If the pharmacy switches supplier, who removes the old Product ID and adds the new one? The old supplier shouldn't be able to block their own removal. |
| 4 | **Multiple suppliers** | A HealthcareService may have multiple suppliers (e.g. one for BaRS, one for ITK). Does it carry multiple Product ID identifiers? If so, each supplier can only modify resources related to their own product — but the HealthcareService itself is shared. |
| 5 | **Duplicate detection impact** | HealthcareService uniqueness is defined as `identifier.system` + `identifier.value`. If Product ID is in `identifier[]`, does a second HealthcareService with the same DoS service ID but a different Product ID count as a duplicate? (It should — the DoS ID is the identity, not the Product ID.) |
| 6 | **Migration** | Existing HealthcareServices in the BaRS database don't have Product IDs. During migration, should the supplier's Product ID be added? If so, how is the supplier-to-service mapping obtained? |
| 7 | **Data migration document** | `data-migration.md` doesn't describe adding a Product ID to HealthcareServices during Step 3. This needs updating if delegated authority is adopted. |

### Design options

| Option | How it works | Pros | Cons |
|--------|-------------|------|------|
| **A — Product ID in `identifier[]`** | Supplier's Product ID stored as an additional identifier on the HealthcareService | Simple to query; standard FHIR pattern; visible in the resource | Pollutes the identifier array with an authorisation concern; affects duplicate detection logic |
| **B — Product ID in an extension** | A custom extension (e.g. `managingProduct`) holds the Product ID separately from `identifier[]` | Clean separation of identity (DoS ID) from authorisation (Product ID); no impact on duplicate detection | Non-standard; requires extension definition; less discoverable |
| **C — Derived from linked Endpoints** | No Product ID on the HealthcareService itself; delegated authority is inferred from the supplier owning a Template that has child Endpoints linked to this service | No schema change needed; authority is implicit from the relationship | Complex to evaluate at runtime; doesn't cover the case where a supplier creates the HealthcareService before any Endpoints exist |

### Recommendation

This needs a design decision. The authorisation document proposes Option A but it hasn't
been validated against the duplicate detection rules or the migration process. The key
question is:

> **Should the Product ID on a HealthcareService be treated as an identity (part of
> `identifier[]`) or as an authorisation grant (separate field/extension)?**

If it's an identity, it affects duplicate detection. If it's an authorisation grant, it
should be stored separately. This distinction matters and should be discussed with the
team before implementation.
