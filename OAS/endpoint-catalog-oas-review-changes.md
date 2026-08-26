# Endpoint Catalog API — OAS Review: Findings and Changes

This document is both a record of the changes made to `endpoint-catalog-api.json` during review and a register of all outstanding findings against FHIR R4, UK Core R4 STU2, and general consistency. Each item explains the problem in plain language, shows before/after where a change was applied, and states its status.

**Standards baseline:** HL7 FHIR R4 (4.0.1) and UK Core R4 STU2.

**Status key:**

- **Fixed** — change applied to the OAS.
- **Pending** — identified, not yet actioned.
- **Pending (team)** — needs a decision before it can be actioned.

**Verified column:** records whether an item has been independently double-checked after being actioned. Leave blank until checked; mark (e.g. `Yes`, initials, or a date) once confirmed.

---

## Summary


| #  | Issue                                                                                                                   | Status                                                                        | Verified      |
| ---- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------- |
| 1  | `Period`/`Start`/`End` capitalisation wrong                                                                             | Fixed                                                                         | RW - 26/08/26 |
| 2  | `connectionType`/`payloadType` search params modelled as objects, not tokens; wrong system URL in connectionType schema | Fixed                                                                         |               |
| 3  | Custom`endpoint-payload-type-epc` system used instead of standard                                                       | Fixed                                                                         | RW - 26/08/26 |
| 4  | Schema named`OperationalOutcome` instead of `OperationOutcome`                                                          | Pending (team)                                                                |               |
| 5  | HealthcareService identifier systems use`http://` instead of `https://`                                                 | Fixed                                                                         |               |
| 6  | `product-id` system uses lowercase `/id/` instead of `/Id/`                                                             | Fixed                                                                         |               |
| 7  | Search param names disagree (`ConnectionType` vs `connection-type`)                                                     | Fixed                                                                         |               |
| 8  | Identifier param names use element paths (`Endpoint.identifier`) instead of FHIR search param names                     | Fixed                                                                         |               |
| 9  | `providedBy` param vs CapabilityStatement's `organization`                                                              | Fixed (with follow-up correction:`organization` restored to `reference`)      |               |
| 10 | `Accept` header example has trailing semicolon                                                                          | Fixed                                                                         |               |
| 11 | Stale version numbers and publisher in Capability schema                                                                | Fixed                                                                         |               |
| 12 | Capability schema`format` example says `xml`, API serves `json`                                                         | Fixed                                                                         |               |
| 13 | `managingOrganization` modelled as array, FHIR says `0..1` single reference                                             | Fixed                                                                         | RW - 26/08/26 |
| 14 | `Endpoint.header` misused as visibility flag                                                                            | Fixed (schema/examples aligned to FHIR; visibility moved out of descriptions) | RW - 26/08/26 |
| 15 | `connectionType` modelled as CodeableConcept, FHIR says it's a Coding                                                   | Fixed                                                                         |               |
| 16 | Request body media type carried BaRS version parameter                                                                  | Fixed                                                                         |               |
| 17 | PUT operations for HealthcareService, Endpoint, and EndpointTemplate missing`requestBody`                               | Fixed                                                                         |               |
| 18 | `HealthcareService.type` absent despite being UK Core MustSupport                                                       | Pending (team)                                                                |               |
| 19 | `HealthcareService.providedBy` modelled as array, FHIR/UK Core say `0..1`                                               | Fixed                                                                         | RW - 26/08/26 |
| 20 | Create interactions return`200` instead of `201 Created`, and omit `Location`/`ETag`/`Last-Modified`                    | Pending                                                                       |               |
| 21 | `$template` routes do not follow FHIR operation invocation rules                                                        | Pending (team)                                                                |               |
| 22 | `Identifier.display` used, which is not a valid FHIR Identifier element                                                 | Fixed                                                                         |               |
| 23 | Resource schemas require server-assigned`id` but not the mandatory R4 elements; `resourceType` not enum-locked          | Pending                                                                       |               |
| 24 | CapabilityStatement does not advertise supported profiles,`updateCreate`, or `$template` operations                     | Pending                                                                       |               |
| 25 | `EPC-EndpointList` List profile conformance cannot be verified (profile not supplied)                                   | Pending                                                                       |               |
| 26 | `OperationOutcome` schema does not enforce base/profile constraints (beyond the naming issue in #4)                     | Pending                                                                       |               |
| 27 | `Endpoint.address` redaction produces an incomplete FHIR resource; business rule may have regressed                     | Pending (team)                                                                |               |
| 28 | Create examples include server-managed fields (`id`, `meta.lastUpdated`)                                                | Pending                                                                       |               |
| 29 | Update-as-create not supported: removed`201` from PUTs and upsert wording (`updateCreate` left absent)                  | Fixed                                                                         |               |
| 30 | `Accept` header marked required; FHIR treats it as optional                                                             | Pending                                                                       |               |
| 31 | Logical ids / path params constrained to UUID only; FHIR`id` is broader                                                 | Pending                                                                       |               |
| 32 | CapabilityStatement and OAS not kept in sync (names, types, profiles, operations)                                       | Pending                                                                       |               |

> **Provenance:** Items #1–#18 arose during the interactive review session. Items #19–#26 were carried over from the earlier standalone conformance review (`endpoint-catalog-oas-fhir-r4-uk-core-review.md`, since merged into this document). Items #27–#32 were added from a later set of review comments.

---

## Findings — Detail (numeric order)

---

### #1 — `Period`/`Start`/`End` capitalisation

**Status: Fixed**

**Problem**

FHIR is case-sensitive. The element is `period` with sub-fields `start` and `end` (all lowercase). The spec had them capitalised: `Period`, `Start`, `End`. Any FHIR validator or generated SDK would not recognise these fields.

**Before**

```json
"Period": {
  "Start": "2021-10-11T15:23:30+00:00",
  "End": "2021-10-11T15:23:30+00:00"
}
```

**After**

```json
"period": {
  "start": "2021-10-11T15:23:30+00:00",
  "end": "2021-10-11T15:23:30+00:00"
}
```

**Scope:** 2 schema definitions (`Endpoint`, `EndpointBundle`) + 5 example blocks. All instances corrected.

---

### #2 — connectionType/payloadType search params and wrong system URL

**Status: Fixed**

**Problem (two parts)**

Part A: The `ConnectionType` and `PayloadType` query parameters referenced nested object schemas (`connectionType.coding[].{system,code,display}`), but query parameters are flat strings on a URL. They should use the `system|value` token format — the same pattern used everywhere else in this spec (identifiers, `_has`, etc.).

Part B: Inside the Endpoint/EndpointTemplate/EndpointBundle schema definitions, the `connectionType.coding.system` example value incorrectly showed `endpoint-payload-type` (the payload system) instead of `endpoint-connection-type`. A copy-paste error.

**Before — param schema**

```json
"ConnectionType_QParam": {
  "name": "ConnectionType",
  "schema": { "$ref": "#/components/schemas/ConnectionType" }
}
// where ConnectionType was a nested object schema
```

**After — param schema**

```json
"ConnectionType_QParam": {
  "name": "ConnectionType",
  "description": "Filter by the technical protocol... Pass as a token in system|value form...",
  "schema": { "$ref": "#/components/schemas/ConnectionTypeToken" }
}
```

```json
"ConnectionTypeToken": {
  "type": "string",
  "example": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest"
}
```

**Before — schema system example**

```json
"connectionType": {
  "coding": [{
    "system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type",  // WRONG
    "code": "hl7-fhir-msg"
  }]
}
```

**After — schema system example**

```json
"connectionType": {
  "coding": [{
    "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",  // correct
    "code": "hl7-fhir-msg"
  }]
}
```

**Scope:** 2 parameter definitions restructured, 2 token schemas added, 2 orphaned object schemas removed, 3 schema system examples corrected.

> Note: the param `name` values (`ConnectionType`/`PayloadType`) were later renamed to kebab-case under #7.

---

### #3 — Custom `-epc` payload type system removed

**Status: Fixed**

**Problem**

Examples used `http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc` — a custom system placed on the HL7 domain. The standard FHIR R4 system is `http://terminology.hl7.org/CodeSystem/endpoint-payload-type`. Mixing them causes token searches to miss matches, and putting a custom code on HL7's domain is misleading.

**Before**

```json
"system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc"
```

**After**

```json
"system": "http://terminology.hl7.org/CodeSystem/endpoint-payload-type"
```

**Scope:** 10 occurrences across examples, param descriptions, and token schemas — all standardised.

---

### #4 — Schema named `OperationalOutcome`

**Status: Pending (team)**

**Problem**

The schema component is named `OperationalOutcome`. The FHIR resource is `OperationOutcome`. The `resourceType` value inside is correctly `"OperationOutcome"` — it's just the schema key that's off. This may be a typo or a deliberate NHS convention.

**Finding from NHS e-RS API:**

The NHSDigital e-Referral Service API (the closest comparable NHS FHIR API) names its R4 OperationOutcome schema `NHSDigital-OperationOutcome` — prefixed, but still uses `OperationOutcome` not `OperationalOutcome`. Neither the FHIR standard nor the NHS pattern uses `OperationalOutcome`.

**Options:**

- `OperationOutcome` — plain FHIR name, simplest.
- `NHSDigital-OperationOutcome` — aligns with e-RS naming convention.

**Decision needed:** Team to confirm which naming convention to follow.

---

### #5 — HealthcareService identifier systems: `http://` vs `https://`

**Status: Fixed**

**Problem**

FHIR treats the `system` URI as an opaque string. `http://fhir.nhs.uk/Id/dos-service-id` and `https://fhir.nhs.uk/Id/dos-service-id` are considered two completely different systems. A token search against one will never match the other. The HealthcareService examples used `http://` while everything else in the spec used `https://`.

**Before**

```json
{ "system": "http://fhir.nhs.uk/Id/dos-service-id", "value": "111111111" }
```

**After**

```json
{ "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "111111111" }
```

**Scope:** All `http://fhir.nhs.uk` occurrences (HealthcareService examples + `_has` param examples) converted to `https://`.

---

### #6 — `product-id` system uses lowercase `/id/`

**Status: Fixed**

**Problem**

The NHS canonical form for identifier systems is `https://fhir.nhs.uk/Id/...` (capital `Id`). The `product-id` entries uniquely used lowercase `/id/`. Since URIs are opaque strings, `/id/product-id` and `/Id/product-id` are different systems — causing silent search failures.

**Before**

```json
"system": "https://fhir.nhs.uk/id/product-id"
```

**After**

```json
"system": "https://fhir.nhs.uk/Id/product-id"
```

**Scope:** 8 occurrences in Endpoint/EndpointTemplate examples.

---

### #7 — Search param names disagree

**Status: Fixed**

**Problem**

The same search parameters were named differently depending on where you looked in the spec:


| Source                                    | Name used (before)                |
| ------------------------------------------- | ----------------------------------- |
| Query parameter definition (`name` field) | `ConnectionType`, `PayloadType`   |
| CapabilityStatement`searchParam`          | `connection-type`, `payload-type` |
| `GET /Endpoint` description table         | `connection-type`, `payload-type` |

FHIR search parameters use lowercase kebab-case (`connection-type`, `payload-type`). A client reading the CapabilityStatement sends `?connection-type=...`, but a client generated from the OpenAPI parameter would send `?ConnectionType=...`. On a case-sensitive server, one of them silently fails.

**Before**

```json
"ConnectionType_QParam": { "name": "ConnectionType", ... }
"PayloadType_QParam":    { "name": "PayloadType", ... }
```

**After**

```json
"ConnectionType_QParam": { "name": "connection-type", ... }
"PayloadType_QParam":    { "name": "payload-type", ... }
```

The two query-parameter `name` fields now match the CapabilityStatement and the `GET /Endpoint` description table. The internal component keys (`ConnectionType_QParam`, `PayloadType_QParam`) and their `$ref`s are unchanged — they never appear on the wire.

**Note:** The `identifier` parameter naming (#8) is a separate, still-pending item.

---

### #8 — Identifier param names use element paths

**Status: Fixed**

**Problem**

The `identifier` query parameters were named `Endpoint.identifier` and `HealthcareService.identifier` (element paths), but the CapabilityStatement advertises them as plain `identifier`. FHIR's search parameter for a resource's own business identifier is `identifier` — the dotted form is element-path syntax, not a search-parameter name, and would not match on the wire.

**Before**

```json
"HealthcareServiceIdentifier_QParam": { "name": "HealthcareService.identifier", ... }
"EndpointIdentifier_QParam":          { "name": "Endpoint.identifier", ... }
```

**After**

```json
"HealthcareServiceIdentifier_QParam": { "name": "identifier", ... }
"EndpointIdentifier_QParam":          { "name": "identifier", ... }
```

Both query parameters now use the plain FHIR search-parameter name `identifier`, matching the CapabilityStatement. The component keys and `$ref`s are unchanged (off-wire), as are the descriptions and `schema` refs (`Service` / `EndpointIdentifier`). `GET /HealthcareService` and `GET /Endpoint` each expose `?identifier=...` — no collision, as they are separate operations with their own schemas.

**Effect on #32:** This closes the `identifier` half of the search-parameter drift between the OAS and the CapabilityStatement. The CapabilityStatement side already had the correct names, so no CapabilityStatement edit was needed.

---

### #9 — `providedBy` param vs CapabilityStatement's `organization`

**Status: Fixed** (with follow-up correction: `organization` restored to `reference`)

**Problem**

The query parameter for filtering HealthcareServices by managing organisation was named `HealthcareService.providedBy` (the element path). But:

- The CapabilityStatement advertised it as `organization`.
- The value shape (`system|value` ODS code) is a token, not a reference.
- A client reading the CapabilityStatement would send `?organization=...`, not `?HealthcareService.providedBy=...`.

**Before**

```json
"HCProvidedBy_QParam": {
  "name": "HealthcareService.providedBy",
  "description": "The identifier of the organization..."
}
```

```json
// CapabilityStatement
{ "name": "organization", "type": "reference" }
```

**After (initial fix — later corrected)**

```json
"HCProvidedBy_QParam": {
  "name": "organization.identifier",
  "description": "Filter by the ODS code of the managing organisation (maps to HealthcareService.providedBy). Pass as a token in system|value form..."
}
```

**Correction (from later review — now applied):** Changing the CapabilityStatement `organization` parameter to `type: token` was **incorrect**. In FHIR R4, `organization` is a **`reference`** search parameter over `HealthcareService.providedBy`. A chained query `organization.identifier={system}|{value}` uses a token value only for the final `identifier` segment of the chain — this does not change the underlying `organization` parameter type.

**Final applied state:**

```json
// CapabilityStatement
{ "name": "organization", "type": "reference" }
```

```json
// Query parameter (reference chaining by ODS code)
"HCProvidedBy_QParam": { "name": "organization.identifier", "schema": { "$ref": "#/components/schemas/ODS" } }
```

---

### #10 — `Accept` header example trailing semicolon

**Status: Fixed**

**Problem**

The `AcceptEPC_HParam` example was `application/fhir+json;` with a dangling semicolon and nothing after it. This is an invalid media type string.

**Before**

```json
"example": "application/fhir+json;"
```

**After**

```json
"example": "application/fhir+json"
```

> Related: the `Accept` header is also incorrectly marked `required` — see #30.

---

### #11 — Stale version numbers and publisher in `Capability` schema

**Status: Fixed**

**Problem**

The `Capability` schema (the reusable schema for the CapabilityStatement resource) had example values from an older version of the spec: `1.2.0` / `1.1.0` for versions, `NHS Digital` for publisher, and `2022-09-16` for date. The authoritative values in the `info` block and `/metadata` example had moved on to `1.0.0-alpha` / `NHS England` / `2026-06-08`. This is a brand-new, first-release pre-release API.

**Before**

```json
"version":   { "example": "1.2.0" },
"date":      { "example": "2022-09-16T15:00:00+01:00" },
"publisher": { "example": "NHS Digital" },
"software.version": { "example": "1.1.0" }
```

**After**

```json
"version":   { "example": "1.0.0-alpha" },
"date":      { "example": "2026-06-08T00:00:00+01:00" },
"publisher": { "example": "NHS England" },
"software.version": { "example": "1.0.0-alpha" }
```

All version fields across the spec (`info.version`, `/metadata` example, `Capability` schema examples) now consistently say `1.0.0-alpha`.

---

### #12 — Capability schema `format` example

**Status: Fixed**

**Problem**

The `Capability` schema's `format` array example was `"xml"`, but the API only serves JSON and the `/metadata` example advertises `["json"]`.

**Before**

```json
"format": { "items": { "example": "xml" } }
```

**After**

```json
"format": { "items": { "example": "json" } }
```

---

### #13 — `managingOrganization` modelled as array

**Status: Fixed**

**Problem**

FHIR R4 defines `Endpoint.managingOrganization` as `Reference(Organization) [0..1]` — a single optional reference, not a collection. The spec modelled it as an array, implying an Endpoint can have multiple managing organisations. This is:

- Non-conformant: FHIR validators reject an array where the element is max 1.
- Misleading: it teaches implementers the wrong data shape.
- Potentially data-losing: lenient parsers might silently take only the first element.

**Before — schema**

```json
"managingOrganization": {
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "identifier": { ... }
    }
  }
}
```

**Before — examples**

```json
"managingOrganization": [
  { "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "X254" } }
]
```

**After — schema**

```json
"managingOrganization": {
  "type": "object",
  "properties": {
    "identifier": { ... }
  }
}
```

**After — examples**

```json
"managingOrganization": {
  "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "X254" }
}
```

**Scope:** 3 schema definitions + 8 example blocks.

---

### #14 — `Endpoint.header` misused as visibility flag

**Status: Fixed** (schema/examples aligned to FHIR; visibility moved out of descriptions)

**Problem**

The spec used `Endpoint.header` as a `public`/`private` flag controlling whether the `address` field is visible to non-owners. This is not what `header` means in FHIR R4.

FHIR R4 defines `Endpoint.header` as `string [0..*]` — a list of connection headers (like HTTP headers) that a connecting system must send when contacting the endpoint's address. Examples: `Authorization: Bearer ...`, `Content-Type: application/fhir+json`.

Setting `header: "public"` would mean a client should send `public` as a request header — which is meaningless.

**Before — schema**

```json
"header": {
  "type": "string",
  "example": "public"
}
```

**Before — examples**

```json
"header": "public"
```

**Before — descriptions**

> "The Endpoint.address will be omitted from results where Endpoint.header is set to private..."

**After — schema**

```json
"header": {
  "type": "array",
  "description": "Additional headers to send when connecting to the endpoint's address (e.g. HTTP headers). FHIR R4 Endpoint.header (string 0..*).",
  "items": { "type": "string" },
  "example": ["Content-Type: application/fhir+json"]
}
```

**After — examples**

```json
"header": ["Content-Type: application/fhir+json"]
```

**After — descriptions**

> "The Endpoint.address is omitted for non-owning organisations. Ownership is determined by matching NHSD-End-User-Organisation-ODS against managingOrganization."

The address-visibility mechanism itself is being handled separately (see #27) — the descriptions now state the behaviour without attributing it to a specific resource field.

A detailed note was also added to `endpoint-header.md` explaining the misuse and FHIR's true intent.

---

### #15 — `connectionType` modelled as CodeableConcept instead of Coding

**Status: Fixed**

**Problem**

In FHIR R4, `Endpoint.connectionType` is a `Coding (1..1)` — a flat object with `system`, `code`, `display`, etc. It is **not** a `CodeableConcept` (which wraps codings in a `coding[]` array). The spec had it wrapped in `coding[]` everywhere — schemas and examples.

(`payloadType` genuinely is `CodeableConcept [0..*]`, so its `coding[]` is correct and was left untouched.)

**Before — schema**

```json
"connectionType": {
  "type": "object",
  "properties": {
    "coding": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "system":  { ... },
          "code":    { ... },
          "display": { ... }
        }
      }
    }
  }
}
```

**After — schema**

```json
"connectionType": {
  "type": "object",
  "description": "FHIR R4 Endpoint.connectionType is a Coding (1..1), not a CodeableConcept.",
  "properties": {
    "system":       { "type": "string", "format": "url", "example": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type" },
    "version":      { "type": "string" },
    "code":         { "type": "string", "example": "hl7-fhir-msg" },
    "display":      { "type": "string", "example": "HL7 FHIR Messaging" },
    "userSelected": { "type": "boolean" }
  }
}
```

**Before — examples**

```json
"connectionType": {
  "coding": [
    { "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "hl7-fhir-msg", "display": "HL7 FHIR Messaging" }
  ]
}
```

**After — examples**

```json
"connectionType": {
  "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
  "code": "hl7-fhir-msg",
  "display": "HL7 FHIR Messaging"
}
```

**Description update:**

> Matches against Endpoint.connectionType.coding.code → Matches against Endpoint.connectionType.code

**Scope:** 3 schema definitions, 8 examples, 3 description references.

---

### #16 — Request body media type carried BaRS version parameter

**Status: Fixed**

**Problem**

The request body `content` keys used `application/fhir+json;version=1.4.0` — a version parameter carried over from the BaRS API. This API is new (`1.0.0-alpha`) and doesn't version its media types. Response content types already used plain `application/fhir+json`. The inconsistency between request and response media types would confuse implementers and could trip up code generators.

**Before**

```json
"content": {
  "application/fhir+json;version=1.4.0": { ... }
}
```

**After**

```json
"content": {
  "application/fhir+json": { ... }
}
```

**Scope:** 4 request bodies (Endpoint, EndpointTemplate, HealthcareService, FhirList).

---

### #17 — PUT operations missing `requestBody`

**Status: Fixed**

**Problem**

The PUT (update) operations for HealthcareService, Endpoint, and EndpointTemplate had no `requestBody` defined, even though their descriptions explicitly mention "the request body" and validate its contents. Without a `requestBody` in the OpenAPI spec:

- Swagger UI shows no body input for these operations.
- Generated SDKs produce methods with no payload parameter.
- Implementers have to guess the expected body shape.

The corresponding POST operations all had their request bodies correctly referenced.

**Before (HealthcareService example)**

```json
"put": {
  "operationId": "replaceHealthcareService",
  "responses": { ... },
  "parameters": []
}
```

**After**

```json
"put": {
  "operationId": "replaceHealthcareService",
  "responses": { ... },
  "requestBody": {
    "$ref": "#/components/requestBodies/HealthcareService"
  },
  "parameters": []
}
```

**Scope:** 3 PUT operations fixed:

- `PUT /HealthcareService/{id}` → `#/components/requestBodies/HealthcareService`
- `PUT /Endpoint/{id}` → `#/components/requestBodies/Endpoint`
- `PUT /Endpoint/{id}/$template` → `#/components/requestBodies/EndpointTemplate`

(`PUT /List/{id}` already had its requestBody.)

---

### #18 — `HealthcareService.type` absent

**Status: Pending (team)** — awaiting confirmation of the value-set / system binding before applying.

**Problem**

In the **HL7 FHIR UK Core Implementation Guide**, `HealthcareService.type` is configured with a **MustSupport (S)** flag. This means that while the element has a `0..*` cardinality (technically optional in structural syntax), downstream conformant applications **must be capable of capturing, storing, processing, and transmitting** this service-type data when it is available.

The current `HealthcareService` schema omits `type` entirely. Because it is absent from the OAS, generated clients have no typed representation of the element, so they cannot reliably support it as UK Core requires — even though the underlying JSON would not reject an unknown property.

`type` is a `CodeableConcept [0..*]` carrying a coded classification of the service (UK Core binds this to a SNOMED-based value set).

**Changes required**

1. **`HealthcareService` schema** — add a `type` property after `active` (matching FHIR element ordering), as an array of `CodeableConcept`:

```json
"type": {
  "type": "array",
  "description": "UK Core MustSupport. A coded classification of the service. FHIR HealthcareService.type (CodeableConcept 0..*).",
  "items": {
    "type": "object",
    "properties": {
      "coding": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "system":  { "type": "string", "format": "url", "example": "<value-set system — TBC>" },
            "code":    { "type": "string", "example": "<code — TBC>" },
            "display": { "type": "string", "example": "<display — TBC>" }
          }
        }
      }
    }
  }
}
```

2. **`HealthcareServiceBundle` nested resource schema** — add the same `type` block in the same position (deeper indentation).
3. **Examples (optional but recommended)** — add a populated `type` to the HealthcareService examples (request body, `200`/`201` atomic, bundle) so implementers see a worked instance. Optional because `type` is `0..*` and a valid instance may omit it.
4. **Do NOT add `type` to `required`.** MustSupport does not make it mandatory — its minimum cardinality remains 0.

**Decision needed before applying:** Confirm the value-set / system URL and a representative example `code`/`display`. Candidates: the UK Core Care Setting Type value set (SNOMED-backed), or a programme-specific service-type binding. The structure above is unaffected by the choice — only the `system`/`code`/`display` example values change.

**Reference:** [UKCore-HealthcareService](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/ProfilesandExtensions/Profile-UKCore-HealthcareService?version=current)

---

### #19 — `HealthcareService.providedBy` modelled as array

**Status: Fixed**

**Problem**

FHIR R4 and the `UKCore-HealthcareService` profile define `providedBy` as `Reference(Organization) [0..1]` — a single optional reference. The spec modelled it as an array. This is the same class of error as #13 (`managingOrganization`), but on a different element and against a specific UK Core profile that also marks `providedBy` as MustSupport.

**Before — schema**

```json
"providedBy": {
  "type": "array",
  "items": {
    "type": "object",
    "properties": { "identifier": { ... } }
  }
}
```

**Before — examples**

```json
"providedBy": [
  { "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "R778" } }
]
```

**After — schema**

```json
"providedBy": {
  "type": "object",
  "properties": { "identifier": { ... } }
}
```

**After — examples**

```json
"providedBy": {
  "identifier": { "system": "https://fhir.nhs.uk/Id/ods-organization-code", "value": "R778" }
}
```

**Scope:** 2 schema definitions (`HealthcareService`, `HealthcareServiceBundle` nested resource) + 5 example blocks.

**Related items:**

- The non-FHIR `display` inside `providedBy`'s identifier has since been removed under #22 (Fixed).
- Documenting that the target Organization should conform to `UKCore-Organization` — a CapabilityStatement/profile item (see #24), still pending.

---

### #20 — Create interactions return `200` instead of `201 Created`

**Status: Pending**

**Problem**

A successful FHIR create must return `201 Created`, along with a `Location` header identifying the newly created resource (and, where versioning is supported, `ETag` and `Last-Modified`). Both `POST /HealthcareService` and `POST /Endpoint` declare only `200` and omit these headers. (`POST /List` correctly uses `201`, but its shared response should also document the FHIR headers.)

**Before**

```json
"post": {
  "operationId": "createHealthcareService",
  "responses": {
    "200": { "$ref": "#/components/responses/200-AtomicHealthcareService" },
    ...
  }
}
```

**Recommended after**

```json
"post": {
  "operationId": "createHealthcareService",
  "responses": {
    "201": { "$ref": "#/components/responses/201-AtomicHealthcareService" },
    ...
  }
}
```

…where the `201` response also declares a `Location` header (and `ETag` / `Last-Modified` when versioning is supported).

**Recommended fix:** Change create success responses to `201`, add the `Location` header to every create response, and keep body behaviour consistent with the FHIR `Prefer` header rules. Response-header detail is tracked in #18 of the original comments (see also the header note under #27/#29).

---

### #21 — `$template` routes do not follow FHIR operation rules

**Status: Pending (team)**

**Problem**

A `$`-prefixed path under a FHIR base URL is a FHIR *operation*. FHIR R4 operations are invoked with `POST` (or `GET` for a safe operation meeting the R4 input restrictions). `PUT` and `DELETE` are not valid operation-invocation verbs. The spec defines `PUT` and `DELETE` on `Endpoint/{id}/$template`, and custom operations must also be backed by an `OperationDefinition` and advertised in the CapabilityStatement — neither of which is present.

**Recommended fix (choose one):**

1. Define `$template` as a genuine custom FHIR operation: publish an `OperationDefinition`, use permitted GET/POST invocation, and advertise it in `CapabilityStatement.rest.resource.operation`; or
2. Move template management outside the FHIR base URL and describe it as a non-FHIR API operation.

**Decision needed:** Which approach — a formal FHIR operation, or relocate outside the FHIR base path.

---

### #22 — `Identifier.display` is not a valid FHIR element

**Status: Fixed**

**Problem**

FHIR R4 `Identifier` has no `display` element — its fields are `use`, `type`, `system`, `value`, `period`, and `assigner`. The spec's Endpoint and HealthcareService identifiers included a `display` property.

**Before**

```json
"identifier": [
  {
    "value": "1000099999",
    "system": "https://fhir.nhs.uk/Id/dos-service-id",
    "display": "Directory of Services (DOS)"
  }
]
```

**After**

```json
"identifier": [
  {
    "value": "1000099999",
    "system": "https://fhir.nhs.uk/Id/dos-service-id"
  }
]
```

The `display` key was removed from every `Identifier` — 12 occurrences (5 example instances + 7 schema `properties` blocks, including the one nested inside `providedBy`).

**Care taken:** Removal was scoped to Identifiers only — objects with `value` + `display` and no `code`/`reference`. The valid `Coding.display` fields (connectionType, payloadType, List codes, orderedBy, entry displays — 35 in total) and `Reference.display` fields (18) were left untouched.

**If human-readable text is needed** for an identifier in future, use `Identifier.type.text`, `Identifier.assigner.display`, or a published extension — not a bare `display`.

---

### #23 — Resource schemas conflict with FHIR required fields and create semantics

**Status: Pending**

**Problem**

The reusable `Endpoint` schema requires `id`, but a FHIR create request may omit `id` because the server assigns it. At the same time, the schema does not require the base R4 mandatory Endpoint elements (`status`, `connectionType`, `payloadType`, `address`). The `resourceType` fields are examples rather than fixed values, so a schema could accept a resource of the wrong type.

**Recommended fix:**

- Prefer schemas generated from the applicable FHIR `StructureDefinition` snapshots, or maintain separate create / update / response schemas.
- Lock `resourceType` with a single-value `enum`.
- Enforce base and profile cardinalities (the mandatory Endpoint elements).
- Do not require server-assigned `id` or `meta.lastUpdated` in create requests.

---

### #24 — CapabilityStatement does not advertise supported profiles

**Status: Pending**

**Problem**

The embedded `/metadata` CapabilityStatement lists resource types, interactions, and search parameters, but does not declare the UK Core or EPC profiles the endpoints support. It also describes PUT as able to create resources without setting `updateCreate: true`, and omits the `$template` operation declarations.

**Recommended fix:**

- Declare `profile` / `supportedProfile` canonical URLs per resource (e.g. `UKCore-HealthcareService`, the EPC List profile).
- Set `updateCreate: true` wherever PUT may create a missing resource (see #29).
- Advertise each custom operation with its `OperationDefinition` canonical (see #21).
- Keep the CapabilityStatement synchronised with the OAS and the deployed implementation (see #32).

---

### #25 — `EPC-EndpointList` List profile conformance unverifiable

**Status: Pending**

**Problem**

The List examples claim conformance to `https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList`, but that profile (and its dependencies) was not supplied. It is therefore impossible to verify whether it derives correctly from `UKCore-List` or properly constrains `subject`, `orderedBy`, `entry`, terminology, and extensions.

**Recommended fix:** Publish and supply the EPC profile and its dependencies, state its `baseDefinition` and package version, and advertise it in the CapabilityStatement. This is primarily an informational / dependency item rather than an OAS edit.

---

### #26 — `OperationOutcome` schema does not enforce constraints

**Status: Pending**

**Problem**

Separate from the naming issue in #4, the `OperationOutcome` schema does not enforce the base or UK Core constraints. It does not require `resourceType = OperationOutcome`, at least one `issue`, `issue.severity`, or `issue.code`, and it does not represent elements such as `issue.expression` and `issue.location` or encode the relevant terminology bindings.

**Recommended fix:** Replace the hand-written schema with one derived from the pinned `UKCore-OperationOutcome` snapshot. Require `resourceType = OperationOutcome`, `issue` (1..*), `issue.severity` (1..1), and `issue.code` (1..1); constrain `severity` and `code` to their required FHIR value sets; and support `issue.details`, `issue.diagnostics`, `issue.location`, and `issue.expression`. Best handled together with the #4 naming decision.

---

### #27 — `Endpoint.address` redaction produces an incomplete FHIR resource

**Status: Pending (team)**

**Problem**

FHIR R4 defines `Endpoint.address` with a minimum cardinality of `1` — a complete Endpoint must contain an address. The OAS states the address is omitted for requesters who do not own the Endpoint. Returning the resource without its address therefore produces an incomplete (technically invalid) FHIR representation.

There is also a possible **business-rule regression**: the current wording hides the address for *every* non-owned Endpoint, whereas the earlier requirements only hid addresses explicitly marked `private`.

**Recommended fix:**

- If redaction is required, either exclude the Endpoint entirely from the response, or return an explicitly subsetted resource tagged with the standard `SUBSETTED` code in `Resource.meta.tag`, and document this behaviour clearly.
- Confirm the intended business rule: redact all non-owned addresses, or only those marked private.

**Note:** This is the visibility mechanism deferred under #14. It is being handled separately, but the FHIR-cardinality implication (`address 1..1`) and the SUBSETTED-tag approach are recorded here.

---

### #28 — Create examples include server-managed fields

**Status: Pending**

**Problem**

Endpoint and HealthcareService create examples include fields the server manages: `id`, `meta.lastUpdated` (and `meta.versionId`). Under FHIR create semantics, a client may omit `id` (the server assigns the logical id), and any supplied `id` is ignored; the server also controls `meta.versionId` and `meta.lastUpdated`. Including these in create examples encourages clients to treat them as client-assigned data.

**Recommended fix:** Use create-specific examples that omit `id`, `meta.versionId`, and `meta.lastUpdated`. `meta.profile` declarations may still be included. This pairs with #23 (separate create/response schemas).

---

### #29 — Update-as-create not supported (`updateCreate`)

**Status: Fixed**

**Problem**

The PUT operation descriptions stated that an update may create an initial resource when the logical id does not already exist — FHIR update-as-create behaviour — and the operations declared a `201 Created` response for it. FHIR advertises this capability via `CapabilityStatement.rest.resource.updateCreate`, which was not set, so the behaviour was also unadvertised.

**Decision:** The team confirmed the API will **not** support update-as-create. The fix therefore removes the update-as-create claims (rather than adding `updateCreate: true`).

**Change 1 — Removed the `201` response from all 4 PUT operations.**

Before (each PUT):

```json
"responses": {
  "200": { "$ref": ".../200-Atomic..." },
  "201": { "$ref": ".../201-Atomic..." },
  "4XX": { ... },
  "5XX": { ... }
}
```

After:

```json
"responses": {
  "200": { "$ref": ".../200-Atomic..." },
  "4XX": { ... },
  "5XX": { ... }
}
```

Applied to `PUT /HealthcareService/{id}`, `PUT /Endpoint/{id}`, `PUT /Endpoint/{id}/$template`, `PUT /List/{id}`.

**Change 2 — Removed the upsert wording from all 4 PUT descriptions.**

Before (representative):

> "Creates a new current version for an existing resource [(update)] **or creates an initial version if no resource already exists for the given id [(new)](…#upsert)**. …"

After (representative):

> "Updates an existing resource, creating a new current version [(update)]. **The resource must already exist; a request for an id that does not exist returns `404 Not Found` (update-as-create is not supported).** …"

**Change 3 — `CapabilityStatement.updateCreate`:** left absent. FHIR's default for `updateCreate` is `false`, so an absent flag correctly signals that update-as-create is not supported. (Per instruction, not set explicitly.)

**Observation — orphaned `201` response components:**
Removing the `201` from the PUTs leaves three response components with no remaining references:

- `201-AtomicHealthcareService`
- `201-AtomicEndpoint`
- `201-EndpointTemplate`

These were **intentionally left in place** (not deleted). They are expected to be reused when #20 switches the **POST** create operations from `200` to `201`. (`201-AtomicList` is still referenced by `POST /List`.) If #20 is closed a different way, these three components should be revisited for removal.

---

### #30 — `Accept` header marked required

**Status: Pending**

**Problem**

The reusable `Accept` header parameter is marked `required: true`. In FHIR, `Accept` is optional — clients may supply it to select a response format, but a FHIR server should have a default when it is absent.

**Before**

```json
"AcceptEPC_HParam": {
  "name": "Accept",
  "in": "header",
  "required": true,
  ...
}
```

**Recommended after**

```json
"AcceptEPC_HParam": {
  "name": "Accept",
  "in": "header",
  "required": false,
  ...
}
```

**Recommended fix:** Set `required: false`. The server may still reject explicitly unsupported media types with `406 Not Acceptable`.

---

### #31 — Logical ids constrained to UUID only

**Status: Pending**

**Problem**

The OAS constrains resource logical ids and path parameters with `format: uuid`. FHIR logical ids are not limited to UUIDs — the FHIR `id` datatype permits letters, digits, hyphens, and periods, up to the length limit. Constraining to UUID may reject valid FHIR ids.

**Recommended fix:**

- If UUID-only is an intentional EPC profile restriction, publish it in the relevant `StructureDefinition` and document it as an EPC rule.
- Otherwise, replace the UUID-only schemas with a FHIR `id`-compatible pattern.

---

### #32 — CapabilityStatement and OAS not kept in sync

**Status: Pending**

**Problem**

The embedded CapabilityStatement must stay synchronised with the paths and behaviour the OAS describes. Current drift points include: search parameter names and types, update-as-create behaviour, supported profiles, custom template operations, and response formats/interaction behaviour. Several of the other findings (#7, #8, #9, #21, #24, #29) are symptoms of this drift.

**Recommended fix:** Generate both artefacts from a shared source, or add automated tests comparing the OAS operations with the CapabilityStatement. A small, accurate CapabilityStatement is preferable to a detailed one that drifts from the implementation.

---

## Recommended remediation sequence (from the original conformance review)

For the larger conformance effort (beyond the fixes already applied), the suggested order is:

1. Select and pin the exact FHIR and UK Core package versions.
2. Publish or supply all EPC profiles, extensions, search parameters, operations, and terminology dependencies.
3. Replace hand-written resource schemas with schemas derived from profile snapshots, or maintain profile-aligned create/update/response schemas.
4. Correct remaining cardinalities and remove non-FHIR properties (both now done for `managingOrganization`, `providedBy`, and `Identifier.display`).
5. Correct create/update HTTP interactions and response headers (`201`, `Location`).
6. Normalise all search parameter names and synchronise them with the CapabilityStatement.
7. Redesign or relocate the `$template` operations.
8. Update the CapabilityStatement to advertise profiles, custom operations, and update-as-create behaviour.
9. Validate every request and response example using the official FHIR validator with pinned UK Core and EPC packages.
10. Add conformance validation to CI so future OAS changes cannot introduce profile drift.

**Suggested validation command (conceptual):**

```text
java -jar validator_cli.jar example.json \
  -version 4.0.1 \
  -ig fhir.r4.ukcore.stu2#<pinned-version> \
  -ig <epc-implementation-guide-package>#<pinned-version> \
  -profile <applicable-profile-canonical>
```

**Standards references:**

- [HL7 FHIR R4 RESTful API](https://hl7.org/fhir/R4/http.html)
- [HL7 FHIR R4 Operations](https://hl7.org/fhir/R4/operations.html)
- [HL7 FHIR R4 Search](https://hl7.org/fhir/R4/search.html)
- [HL7 FHIR R4 Endpoint](https://hl7.org/fhir/R4/endpoint-definitions.html)
- [HL7 FHIR R4 HealthcareService](https://hl7.org/fhir/R4/healthcareservice-definitions.html)
- [HL7 FHIR R4 Identifier](https://hl7.org/fhir/R4/datatypes-definitions.html)
- [UK Core R4 STU2 Implementation Guide](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2?version=2.0.2)
- [UKCore-HealthcareService](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/ProfilesandExtensions/Profile-UKCore-HealthcareService?version=current)
- [UK Core MustSupport guidance](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/Guidance/MustSupport-Guidance?version=current)

---

## Process Notes

- After multiple targeted text edits, a full JSON reformat (2-space indent) was applied to eliminate indentation inconsistencies introduced by structural changes (de-arraying `managingOrganization` and `providedBy`, converting `connectionType` to Coding). The file is semantically identical but the diff from the original is whole-file.
- A detailed note about the `Endpoint.header` misuse was separately added to `epc-work/Documents/endpoint-header.md`.
- The `info.version` and all version references were standardised to `1.0.0-alpha` reflecting this is the first release of the API, in pre-release state.
