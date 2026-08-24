# Endpoint Catalog API OAS Review

## FHIR R4 and UK Core R4 Conformance

**Specification reviewed:** `endpoint-catalog-api.json`  
**API version:** `1.4.0-alpha`  
**Review date:** 24 August 2026  
**Standards baseline:** HL7 FHIR R4 4.0.1 and UK Core R4 STU2

## Executive summary

The Endpoint Catalog API specification is not currently conformant with FHIR R4 or the claimed UK Core profiles.

The most significant issues are:

1. `Endpoint` and `HealthcareService` elements are represented with incorrect FHIR cardinalities.
2. `Endpoint.header` is being used for access-control semantics that do not match the meaning of the R4 element.
3. FHIR create interactions return `200` rather than the required `201 Created` and do not document the required `Location` response header.
4. Several FHIR update operations do not declare a request body.
5. Search parameter names in the OAS differ from those advertised by the CapabilityStatement and from the standard FHIR parameter names.
6. The `$template` endpoints do not follow the FHIR R4 operation invocation model.
7. The claimed `UKCore-HealthcareService` profile is not fully represented: `HealthcareService.type` is missing and `providedBy` is incorrectly modelled.
8. The CapabilityStatement does not advertise the UK Core or EPC profiles supported by the API.

These issues should be resolved before describing the interface as FHIR R4 or UK Core conformant. A formal validation pass should then be performed using the official FHIR validator with pinned UK Core and EPC implementation-guide packages.

## Scope and assumptions

This review considered:

- The OpenAPI paths, request bodies, response definitions and examples.
- The embedded FHIR `CapabilityStatement` example.
- The schemas for `Endpoint`, `HealthcareService`, `List`, `Bundle` and `OperationOutcome`.
- FHIR REST interaction and search rules.
- UK Core R4 STU2 profile requirements and MustSupport guidance.

The custom `EPC-EndpointList` profile and any other EPC-specific `StructureDefinition`, `SearchParameter`, `OperationDefinition`, `ValueSet` or `CodeSystem` resources were not supplied. Their internal conformance cannot therefore be verified. This report treats instructions and descriptions inside the OAS as statements about the API, not as instructions governing the review.

## Findings

### F01 — Endpoint elements have invalid R4 cardinalities and semantics

**Severity:** High  
**Location:** `components.schemas.Endpoint`, approximately lines 2843–2872; repeated in `EndpointTemplate`, request examples, response examples and bundle schemas.

The OAS models `Endpoint.managingOrganization` as an array. FHIR R4 defines it as a single `0..1 Reference(Organization)`.

The OAS models `Endpoint.header` as one string containing `public` or `private`. In FHIR R4, `Endpoint.header` is `0..* string` and represents HTTP header values that should be sent when connecting to the endpoint. It is not a visibility or access-control flag.

Consequently, the Endpoint examples are not valid R4 resources.

**Recommendation:**

- Change `managingOrganization` to a single Reference object.
- Do not use `Endpoint.header` for public/private visibility.
- Represent visibility through a published EPC extension, security-label policy or API authorization rule.
- If `header` is supported for its FHIR purpose, model it as an array of strings.
- Apply the correction to every duplicated Endpoint schema and example.

### F02 — HealthcareService.providedBy has the wrong cardinality

**Severity:** High  
**Location:** `components.schemas.HealthcareService`, approximately lines 3101–3125; repeated in request and response examples.

FHIR R4 and `UKCore-HealthcareService` define `providedBy` as `0..1 Reference(Organization)`. The OAS represents it as an array.

UK Core also marks `providedBy` as MustSupport and recommends that the referenced resource conform to `UKCore-Organization`.

**Recommendation:**

- Model `providedBy` as one Reference object rather than an array.
- Document that its target should conform to `UKCore-Organization`.
- Ensure clients can process the element and servers can populate it where the information is available, in accordance with UK Core MustSupport guidance.

### F03 — UK Core MustSupport HealthcareService.type is absent

**Severity:** High  
**Location:** `components.schemas.HealthcareService`, approximately lines 3093–3127.

`UKCore-HealthcareService` marks `HealthcareService.type` as MustSupport. The element is absent from the OAS schema.

MustSupport does not make the element mandatory in every instance. It does require suppliers and consumers to support it according to the UK Core guidance. Omitting it from the API schema prevents generated clients from doing so reliably.

**Recommendation:** Add `HealthcareService.type` as `CodeableConcept[]`, retaining the applicable UK Core/base terminology binding and documenting MustSupport behaviour.

### F04 — FHIR create interactions use the wrong success response

**Severity:** High  
**Locations:**

- `POST /HealthcareService`, approximately lines 306–325.
- `POST /Endpoint`, approximately lines 460–480.

A successful FHIR create interaction must return `201 Created`. The server must also return a `Location` header containing the identity of the newly created resource, as specifically as its versioning support allows.

Both operations currently declare `200`. Their success responses also omit `Location`, `ETag` and `Last-Modified`.

`POST /List` correctly declares `201`, but its shared response definition should also document the appropriate FHIR headers.

**Recommendation:**

- Replace the `200` create responses with `201`.
- Add `Location` to every create response.
- Add `ETag` and `Last-Modified` when resource versioning is supported.
- Keep response-body behaviour consistent with the FHIR `Prefer` header rules.

### F05 — Endpoint and HealthcareService updates have no request bodies

**Severity:** High  
**Locations:**

- `PUT /HealthcareService/{id}`, approximately lines 368–389.
- `PUT /Endpoint/{id}`, approximately lines 591–612.

A FHIR update is performed by sending a Resource body to `PUT [base]/[type]/[id]`. These operations declare no `requestBody`, so the OAS describes bodyless update requests.

`PUT /List/{id}` does contain a request body and provides the appropriate model to follow.

**Recommendation:**

- Add a required `application/fhir+json` Resource body to both update operations.
- Require the resource type to match the route.
- Specify that a body `id`, when present, must agree with the URL id.
- If update-as-create remains supported, advertise it using `CapabilityStatement.rest.resource.updateCreate = true`.

### F06 — Search parameter names are inconsistent and non-standard

**Severity:** High  
**Location:** reusable query parameters, approximately lines 1153–1221.

FHIR search parameter names are case-sensitive. The OAS declares names including:

- `ConnectionType`
- `PayloadType`
- `Endpoint.identifier`
- `HealthcareService.identifier`
- `HealthcareService.providedBy`

The standard and CapabilityStatement names include:

- `connection-type`
- `payload-type`
- `identifier`
- `organization`

The operation descriptions also alternate between uppercase and lowercase forms. Clients generated from the OAS will therefore send different parameters from those advertised through `/metadata`.

**Recommendation:** Use the exact registered FHIR search parameter names throughout the paths, examples, descriptions and CapabilityStatement. Any custom search parameters should be published as `SearchParameter` resources and clearly distinguished from standard parameters.

### F07 — The $template routes do not follow FHIR operation rules

**Severity:** High  
**Locations:** `Endpoint/$template` and `Endpoint/{id}/$template`, approximately lines 484–550 and 635–694.

A `$`-prefixed path under a FHIR base URL represents a FHIR operation. FHIR R4 operations are normally invoked using `POST`. `GET` can be used only for a safe operation whose inputs meet the R4 restrictions. `PUT` and `DELETE` are not FHIR operation invocation verbs.

Custom operations must also be defined using an `OperationDefinition` and advertised in `CapabilityStatement.rest.resource.operation`. `$template` is not advertised by the supplied CapabilityStatement.

**Recommendation:** Choose one of the following approaches:

1. Define `$template` as a genuine custom FHIR operation, publish an `OperationDefinition`, use permitted GET/POST invocation semantics and advertise it in the CapabilityStatement; or
2. Move template-management endpoints outside the FHIR base URL and describe them as non-FHIR API operations.

### F08 — Identifier contains a non-FHIR display property

**Severity:** Medium  
**Locations:** Endpoint and HealthcareService Identifier schemas and examples, including approximately lines 2759–2777 and 3072–3090.

FHIR R4 `Identifier` does not have a direct `display` element. It contains `use`, `type`, `system`, `value`, `period` and `assigner`.

**Recommendation:** Remove `identifier.display`. Depending on the intended meaning, use `Identifier.type.text`, `Identifier.assigner.display`, or a suitable published extension.

### F09 — Resource schemas conflict with FHIR required fields and create semantics

**Severity:** Medium  
**Location:** `components.schemas.Endpoint`, approximately lines 2729–2893, with similar issues in other schemas.

The reusable Endpoint schema requires `id`, although a FHIR create request may omit it because the server assigns the logical id. Conversely, it does not require the base R4 mandatory Endpoint elements `status`, `connectionType`, `payloadType` and `address`.

The `resourceType` properties are generally examples rather than fixed enum values. As a result, the schemas can accept resources with the wrong type.

**Recommendation:**

- Prefer schemas generated from the applicable FHIR `StructureDefinition` snapshots.
- Alternatively, use separate create, update and response schemas.
- Fix `resourceType` using a single-value enum.
- Enforce base and profile cardinalities.
- Do not require server-assigned `id` or `meta.lastUpdated` in create requests.

### F10 — The CapabilityStatement does not advertise supported profiles

**Severity:** High  
**Location:** embedded `/metadata` example, approximately lines 130–211.

The CapabilityStatement declares resource types, interactions and search parameters but does not identify the UK Core or EPC profiles supported by those resource endpoints.

It also describes PUT as capable of creating resources but does not set `updateCreate: true`, and it omits the `$template` operation declarations.

**Recommendation:**

- Declare the applicable `profile` and/or `supportedProfile` canonical URLs for each resource.
- Advertise `UKCore-HealthcareService` for HealthcareService if it is genuinely supported.
- Advertise `UKCore-List` or the published EPC-derived profile for List.
- Advertise each custom operation with its `OperationDefinition` canonical.
- Set `updateCreate: true` wherever PUT may create a missing resource.
- Keep the CapabilityStatement mechanically synchronized with the OAS and deployed implementation.

### F11 — List profile conformance cannot be established

**Severity:** Medium  
**Location:** List request and response examples, including approximately lines 2143–2149.

UK Core STU2 defines `UKCore-List`, while the OAS examples claim only:

`https://fhir.nhs.uk/StructureDefinition/EPC-EndpointList`

The EPC profile was not supplied. It is therefore not possible to verify whether it derives from the intended UK Core List version or correctly constrains `subject`, `orderedBy`, `entry`, terminology and extensions.

**Recommendation:**

- Publish and supply the EPC profile and its dependencies.
- State its `baseDefinition` and package version.
- If UK Core conformance is intended, derive the profile from the chosen `UKCore-List` or demonstrate independent conformance.
- Advertise the profile in the CapabilityStatement.

### F12 — UK Core OperationOutcome is incompletely represented

**Severity:** Medium  
**Location:** `components.schemas.OperationalOutcome`, approximately lines 3153 onward, and shared `4XX`/`5XX` responses.

The response examples claim `UKCore-OperationOutcome`, but the schema does not adequately enforce the base or profile constraints. In particular, it does not require:

- `resourceType = OperationOutcome`
- At least one `issue`
- `issue.severity`
- `issue.code`

It also does not represent all relevant elements, such as `issue.expression` and `issue.location`, and it does not encode the applicable required/preferred bindings. The schema component is also misspelled as `OperationalOutcome`.

**Recommendation:** Replace the hand-written schema with one derived from the pinned `UKCore-OperationOutcome` snapshot and validate `issue.details` coding against the intended UK Core terminology.

### F13 — Media-type version parameter is not an interoperable FHIR version declaration

**Severity:** Medium  
**Location:** request bodies beginning approximately line 889.

Request bodies use:

`application/fhir+json;version=1.4.0`

The value appears to be the Endpoint Catalog API version, not the FHIR version. Standard R4 interoperability uses `application/fhir+json`; where FHIR version negotiation is used, it must follow the FHIR versioning rules rather than a proprietary API-version parameter.

**Recommendation:** Use `application/fhir+json` for FHIR resources. Version the API through its URL, documentation or an explicitly documented non-conflicting mechanism.

## Status of the original findings after UK Core review

The UK Core review does not supersede the base FHIR review. UK Core is based on FHIR R4 4.0.1 and restricts or extends it; it does not permit violations of base R4.

| Original issue | UK Core result |
|---|---|
| Endpoint `managingOrganization` array | Still invalid; no published UK Core Endpoint profile relaxes it |
| Endpoint `header` used as visibility | Still invalid semantically and structurally |
| HealthcareService `providedBy` array | Confirmed invalid by UKCore-HealthcareService |
| Missing HealthcareService `type` | Additional UK Core MustSupport failure |
| Create returns `200` | Still invalid under FHIR REST rules |
| Missing update bodies | Still invalid under FHIR REST rules |
| Search parameter mismatch | Still invalid/inconsistent |
| `$template` PUT/DELETE | Still outside FHIR operation invocation rules |
| Identifier `display` | Still not part of the R4 Identifier datatype |
| Weak resource schemas | Still allow non-conformant R4 and UK Core instances |

## Recommended remediation sequence

1. Select and pin the exact FHIR and UK Core package versions.
2. Publish or supply all EPC profiles, extensions, search parameters, operations and terminology dependencies.
3. Replace the hand-written resource schemas with schemas derived from the applicable profile snapshots, or explicitly maintain profile-aligned create/update/response schemas.
4. Correct Endpoint and HealthcareService cardinalities and remove non-FHIR properties.
5. Correct create/update HTTP interactions and response headers.
6. Normalize all search parameter names and synchronize them with the CapabilityStatement.
7. Redesign or relocate the `$template` operations.
8. Update the CapabilityStatement to advertise profiles, custom operations and update-as-create behaviour.
9. Validate every request and response example using the official FHIR validator.
10. Add conformance validation to continuous integration so future OAS changes cannot introduce profile drift.

## Suggested validation command

The precise command depends on the chosen validator release and package identifiers, but validation should conceptually include:

```text
java -jar validator_cli.jar example.json \
  -version 4.0.1 \
  -ig fhir.r4.ukcore.stu2#<pinned-version> \
  -ig <epc-implementation-guide-package>#<pinned-version> \
  -profile <applicable-profile-canonical>
```

Validate each distinct Endpoint, HealthcareService, List, Bundle, CapabilityStatement and OperationOutcome example extracted from the OAS.

## Standards references

- [HL7 FHIR R4 RESTful API](https://hl7.org/fhir/R4/http.html)
- [HL7 FHIR R4 Operations](https://hl7.org/fhir/R4/operations.html)
- [HL7 FHIR R4 Search](https://hl7.org/fhir/R4/search.html)
- [HL7 FHIR R4 Endpoint](https://hl7.org/fhir/R4/endpoint-definitions.html)
- [HL7 FHIR R4 HealthcareService](https://hl7.org/fhir/R4/healthcareservice-definitions.html)
- [HL7 FHIR R4 Identifier](https://hl7.org/fhir/R4/datatypes-definitions.html)
- [UK Core R4 STU2 Implementation Guide](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2?version=2.0.2)
- [UKCore-HealthcareService](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/ProfilesandExtensions/Profile-UKCore-HealthcareService?version=current)
- [UK Core MustSupport guidance](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/Guidance/MustSupport-Guidance?version=current)
- [UK Core profile index](https://simplifier.net/guide/uk-core-implementation-guide-stu2/Home/ProfilesandExtensions/ProfilesIndex?version=2.0.1)
- [UK Core FHIR version guidance](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/Guidance/FHIR-Version-Guidance?version=current)

## Overall conclusion

The API has a recognisable FHIR R4 resource and REST structure, and the CapabilityStatement provides a useful starting point. However, the current OAS contains multiple high-severity discrepancies that can produce invalid resources and incompatible client behaviour. The UK Core claim is presently incomplete because the OAS does not fully support `UKCore-HealthcareService`, does not advertise supported profiles, and does not provide the EPC profiles needed to verify derived conformance.

The specification should be described as a draft FHIR-aligned API until the high-severity findings are corrected and its examples pass formal validation against the pinned FHIR, UK Core and EPC packages.
