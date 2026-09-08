# Booking and Referral API OAS Review

## FHIR R4, UK Core R4 and Inherited-Error Assessment

**Specification reviewed:** `bars api OAS.json`  
**API title:** Booking and Referral API  
**API version:** `1.4.1-alpha`  
**Review date:** 24 August 2026  
**Standards baseline:** HL7 FHIR R4 4.0.1 and UK Core R4 STU2  
**Comparison baseline:** Endpoint Catalog API OAS review

## Executive summary

The BaRS OAS is not currently conformant with FHIR R4 or the UK Core profiles it claims. Several errors identified in the Endpoint Catalog OAS are demonstrably inherited or follow the same shared design pattern. The BaRS file also contains additional resource-specific defects.

The most serious findings are:

1. Appointment and ServiceRequest create interactions return `200` instead of `201 Created`.
2. PATCH operations send partial FHIR resources as `application/fhir+json`, which is not a valid FHIR PATCH format.
3. ServiceRequest instances use invalid element names and cardinalities, including `PatientInstruction` and scalar `basedOn`.
4. ServiceRequest search results are not structured as valid FHIR Bundle entries.
5. `DocumentReference.content.format` is represented using a non-FHIR nested array structure.
6. The DocumentReference create response schema and example contradict each other.
7. The CapabilityStatement materially disagrees with the paths implemented by the OAS.
8. The same incomplete `OperationalOutcome` schema found in the Endpoint Catalog OAS is present here.
9. UK Core MustSupport elements are omitted from the Appointment schema.

The inherited issues suggest that a shared OAS component library or source template should be corrected centrally rather than repairing each API independently.

## Scope and assumptions

The review covered paths, request bodies, response definitions, examples, resource schemas and the embedded CapabilityStatement. Descriptive text inside the OAS was treated as API documentation rather than as instructions for this review.

The custom BaRS profiles named under `https://fhir.nhs.uk/StructureDefinition/` were not supplied. Their additional constraints and their relationship to UK Core could not be verified. This review therefore checks the OAS against base FHIR R4 and published UK Core, and identifies where formal validation additionally requires the BaRS implementation-guide package.

## Inherited-error summary

| Issue previously found in Endpoint Catalog | Present in BaRS | Assessment |
|---|---:|---|
| Hand-written resource schemas do not enforce FHIR/profile constraints | Yes | Inherited design pattern |
| Incomplete, misspelled `OperationalOutcome` schema | Yes | Exact shared component appears to be inherited |
| Create operations return `200` instead of `201` | Yes, Appointment and ServiceRequest | Inherited REST interaction error |
| Proprietary `application/fhir+json;version=<API version>` media types | Yes | Inherited media-type pattern |
| CapabilityStatement does not advertise supported profiles | Yes | Inherited conformance-discovery gap |
| CapabilityStatement and OAS operations/searches disagree | Yes | Inherited synchronization problem, with BaRS-specific instances |
| `resourceType` represented as an example instead of a fixed value | Yes | Inherited schema weakness |
| Server-managed `id` and `meta.lastUpdated` encouraged in create bodies | Yes | Inherited create-schema weakness |
| FHIR response headers incompletely represented | Yes | Inherited interaction-documentation gap |
| Custom `$` operation missing from CapabilityStatement | No | BaRS advertises `$process-message`, though its declaration still needs correction |
| Endpoint-specific cardinality and `header` problems | No | Specific to Endpoint Catalog |

## Detailed findings

### B01 — Appointment and ServiceRequest create responses violate FHIR create rules

**Severity:** High  
**Locations:** `POST /Appointment` and `POST /ServiceRequest`, beginning approximately at lines 2230 and 2939.

A successful FHIR create interaction must return `201 Created` and a `Location` header identifying the new resource. These operations declare only `200` success responses.

The same defect appeared in the Endpoint Catalog create operations and is therefore assessed as inherited.

**Recommendation:** Replace `200` with `201`, add `Location`, and document `ETag` and `Last-Modified` when versioning is supported. Align response-body handling with the FHIR `Prefer` rules.

### B02 — PATCH requests are not valid FHIR PATCH representations

**Severity:** High  
**Locations:** `PATCH /Appointment/{id}` and `PATCH /ServiceRequest/{id}`; request bodies `AppointmentPatch` and `ServiceRequestPatch`, approximately lines 4191–4310.

The PATCH request bodies contain partial Appointment or ServiceRequest resources and use `application/fhir+json;version=1.4.0`. FHIR R4 does not define partial-resource merge semantics for this media type.

A FHIR PATCH request must use a supported patch format, such as JSON Patch with `application/json-patch+json`, XML Patch, or FHIRPath Patch represented by a `Parameters` resource where supported and properly defined.

The ServiceRequest patch example also incorrectly declares the `UKCore-Appointment` profile.

**Recommendation:** Select and document a standards-based PATCH format, define its schema and media type, advertise patch support in the CapabilityStatement, and use ServiceRequest-specific profile metadata in ServiceRequest examples.

### B03 — ServiceRequest uses invalid element names and cardinalities

**Severity:** High  
**Locations:** examples around lines 3015–3061 and schema around lines 5269–5412.

The ServiceRequest representation contains at least these structural errors:

- `basedOn` is modelled as a single object, but FHIR R4 defines it as `0..* Reference`.
- `PatientInstruction` is incorrectly capitalized. FHIR JSON property names are case-sensitive; the R4 element is `patientInstruction`.
- `performer.reference` is constrained as a UUID even though a FHIR Reference may be relative, absolute, contained or logical.
- Mandatory base elements are not consistently required by the schema.
- `resourceType` is not fixed to `ServiceRequest`.

**Recommendation:** Replace the hand-written ServiceRequest model with a schema derived from the applicable BaRS profile snapshot and its UK Core/base dependencies. Correct all examples at the same time.

### B04 — ServiceRequest search Bundle entries are malformed

**Severity:** High  
**Location:** `GET /ServiceRequest` example, approximately lines 2995–3017, and `SearchBundleRef` schema beginning around line 5415.

The example places `resourceType: ServiceRequest` directly inside `Bundle.entry[]`. A FHIR search Bundle entry must place the returned resource under `entry.resource` and normally provides `entry.fullUrl`. This differs from the Appointment search example, which uses the correct wrapper.

**Recommendation:** Structure each result as `entry: [{ fullUrl, resource: { resourceType: "ServiceRequest", ... }, search: { mode: "match" } }]` as appropriate, and correct the corresponding schema.

### B05 — DocumentReference.content.format has a non-FHIR structure

**Severity:** High  
**Location:** `components.schemas.DocumentReference`, within `content[].format`.

FHIR R4 defines `DocumentReference.content.format` as a single optional `Coding`. The OAS represents it as an array whose items contain another `coding` array, effectively modelling a list of CodeableConcept-like wrappers.

**Recommendation:** Model `format` directly as a Coding object containing fields such as `system`, `version`, `code`, `display` and `userSelected`, with cardinality `0..1` per content entry.

### B06 — DocumentReference create response schema contradicts its example

**Severity:** High  
**Location:** `POST /DocumentReference` response, approximately lines 3744–3759.

The `201` response schema references `OperationalOutcome`, but the example body is a `DocumentReference`. A generated client will deserialize the documented success response incorrectly.

The operation summary also says “Create new or update existing document pointer.” Plain FHIR `POST [base]/DocumentReference` is create; update uses PUT, while conditional create requires an `If-None-Exist` header and defined conditional-create behaviour.

**Recommendation:** Decide whether the success body is a DocumentReference or OperationOutcome and make the schema, examples and `Prefer` behaviour consistent. Document any conditional-create semantics explicitly; do not describe ordinary POST as update.

### B07 — Appointment schema does not support the claimed UK Core profile

**Severity:** High  
**Location:** `components.schemas.Appointment`, approximately lines 4950–5045.

The schema claims `UKCore-Appointment` in examples but omits several UK Core MustSupport elements, including:

- `specialty`
- `appointmentType`
- `reasonCode`
- `reasonReference`

It includes `status`, `start`, `basedOn` and `participant`, but does not enforce the base mandatory cardinalities. In particular, Appointment requires `status` and at least one `participant`, and every participant requires `status`.

UK Core also recommends that `basedOn` references conform to `UKCore-ServiceRequest` and slot references conform to `UKCore-Slot`.

**Recommendation:** Generate the schema from `UKCore-Appointment`, retain MustSupport elements, encode required base fields and bindings, and document UK Core target-profile expectations.

### B08 — CapabilityStatement disagrees with the implemented API

**Severity:** High  
**Location:** embedded `/metadata` example, approximately lines 150–520.

Material inconsistencies include:

- `ServiceRequest` advertises only read, vread and search, while the OAS implements create, update, patch and delete.
- Appointment and ServiceRequest PATCH are not advertised.
- Several resources advertise `vread`, but no `/{type}/{id}/_history/{vid}` paths exist.
- Resources declare `versioning: no-version` while also advertising vread, which requires versions.
- The CapabilityStatement advertises search parameters that differ from the OAS query parameters.
- No resource `profile` or `supportedProfile` declarations identify UK Core or BaRS profiles.
- The CapabilityStatement versions (`1.2.0` and software `1.1.0`) do not match the OAS version `1.4.1-alpha`.

This is the same synchronization class of defect found in Endpoint Catalog, with additional BaRS-specific contradictions.

**Recommendation:** Generate the CapabilityStatement from the same authoritative interaction/profile definition as the OAS and implementation, and test them for parity in CI.

### B09 — Some Slot include declarations are invalid

**Severity:** High  
**Location:** CapabilityStatement `Slot.searchInclude`, approximately lines 310–318 and 490–498.

`HealthcareService.providedBy` uses an element-style dotted name, not FHIR `_include` syntax. Include values use `SourceType:searchParameter(:TargetType)`, and chained inclusion normally requires separate `_include:iterate` processing. The OAS accepts only one `_include` value while the CapabilityStatement advertises a wider set.

**Recommendation:** Define each supported include using registered search parameter names and valid syntax. Document iterative includes explicitly, and make the OAS parameter enum match the CapabilityStatement.

### B10 — The $process-message declaration needs correction and tighter modelling

**Severity:** Medium  
**Locations:** `POST /$process-message`, approximately line 855; CapabilityStatement operation near lines 338 and 517.

Using POST at the system-level `$process-message` endpoint is consistent with FHIR operation routing. However:

- `CapabilityStatement.rest.operation.name` should identify the operation name rather than repeat routing punctuation; confirm and use the value defined by the referenced OperationDefinition.
- Request and response bodies should conform to the message-processing operation contract and the appropriate Message Bundle profiles.
- The JSON request uses `version=1.0.0`, while most other request bodies use `version=1.4.0` and the API itself is `1.4.1-alpha`.

**Recommendation:** Derive the operation documentation from the referenced `OperationDefinition`, pin the applicable BaRS message profiles, and remove proprietary or inconsistent media-type version parameters.

### B11 — The inherited OperationalOutcome schema is incomplete

**Severity:** Medium  
**Location:** `components.schemas.OperationalOutcome`, approximately lines 6798–6865.

This component is effectively the same incomplete schema found in Endpoint Catalog. It does not require `resourceType`, `issue`, `issue.severity` or `issue.code`; does not enforce their bindings; and omits relevant issue elements. Its component name is also misspelled.

**Recommendation:** Correct the shared source component once, rename it `OperationOutcome`, and derive it from the pinned `UKCore-OperationOutcome` snapshot and terminology dependencies.

### B12 — Proprietary and inconsistent FHIR media types are inherited

**Severity:** Medium  
**Locations:** throughout responses and request bodies, including lines 661, 893, 2279, 2991, 3690 and 4135 onward.

The OAS uses several values:

- `application/fhir+json;version=1.0.0`
- `application/fhir+json;version=1.4.0`
- `application/fhir+json`
- `application/fhir+xml`

The numeric parameters appear to be API versions, not FHIR versions, and conflict with the OAS version. This pattern is shared with Endpoint Catalog.

**Recommendation:** Use standard `application/fhir+json` and `application/fhir+xml` media types. Apply API versioning outside the FHIR media type unless a formally specified and interoperable negotiation mechanism is adopted.

### B13 — Resource schemas are partial and unsafe for conformance claims

**Severity:** Medium  
**Location:** all schemas under `components.schemas`.

As in Endpoint Catalog, schemas are hand-written subsets of FHIR resources. Common weaknesses include:

- `resourceType` is an example instead of a fixed enum.
- Required base elements and nested cardinalities are omitted.
- Required terminology bindings are not represented.
- FHIR references are reduced to incomplete shapes.
- Create, update and response needs are combined into one schema.
- `id` is UUID-only even though FHIR logical ids are not generally restricted to UUIDs.

**Recommendation:** Generate schemas from the exact FHIR/UK Core/BaRS profile snapshots or use a well-governed profile-to-OAS build process. Avoid treating the OAS schema alone as the authoritative FHIR validator.

### B14 — Search definitions do not consistently use registered FHIR parameters

**Severity:** Medium  
**Location:** Appointment, ServiceRequest, DocumentReference and Slot query parameters and CapabilityStatement declarations.

Patient name, date of birth and postcode searches appear as API-specific parameters on resource searches, while the CapabilityStatement mainly advertises `patient:identifier`. Custom parameters are permissible only when they are defined and published as SearchParameter resources and accurately advertised.

`next-page-token` is described as a token search parameter, although it functions as a paging cursor rather than a resource search criterion.

**Recommendation:** Use standard chaining where possible, publish every custom SearchParameter, and represent pagination using Bundle paging links or a clearly documented implementation-specific mechanism.

## Errors assessed as inherited

The following findings are sufficiently similar to the Endpoint Catalog OAS to treat them as inherited from a common source, template or design approach:

1. **OperationalOutcome:** same component name, shape and missing constraints.
2. **Create status handling:** resource creates documented with `200` rather than `201`.
3. **Media-type versioning:** API versions appended to `application/fhir+json`.
4. **Partial hand-written FHIR schemas:** examples used in place of fixed constraints and incomplete datatype/cardinality coverage.
5. **Profile discovery:** claimed profiles appear in examples but not CapabilityStatement `profile`/`supportedProfile` declarations.
6. **Capability/OAS drift:** the embedded CapabilityStatement does not accurately describe the paths.
7. **Server-managed fields in create models:** shared schemas encourage clients to send `id` and `meta.lastUpdated`.
8. **Incomplete FHIR response metadata:** inconsistent documentation of `Location`, `ETag` and `Last-Modified`.

## BaRS-specific errors

The following are not inherited from the Endpoint-specific review and arise from the BaRS resource and interaction modelling:

1. Invalid PATCH representation.
2. Scalar `ServiceRequest.basedOn`.
3. Incorrectly capitalized `PatientInstruction`.
4. Malformed ServiceRequest Bundle entries.
5. Invalid `DocumentReference.content.format` structure.
6. DocumentReference response schema/example mismatch.
7. Missing UKCore-Appointment MustSupport fields.
8. Invalid or unexplained Slot include declarations.
9. ServiceRequest interaction and vread/versioning contradictions in the CapabilityStatement.

## Recommended remediation sequence

1. Identify the shared OAS templates/components used by Endpoint Catalog and BaRS.
2. Correct shared OperationOutcome, media-type, response-header, profile-advertising and create-interaction components centrally.
3. Pin exact FHIR R4, UK Core and BaRS implementation-guide packages.
4. Regenerate Appointment, ServiceRequest, DocumentReference, Bundle and message schemas from their profiles.
5. Correct ServiceRequest and DocumentReference structures and every duplicated example.
6. Replace partial-resource PATCH with a supported FHIR patch format.
7. Rebuild the CapabilityStatement from the implemented interaction matrix.
8. Publish the custom SearchParameter and OperationDefinition resources used by the API.
9. Extract and validate every OAS example with the official FHIR validator.
10. Add automated parity checks between paths, schemas, examples and CapabilityStatement declarations.

## Formal validation requirements

Validation should include FHIR R4, the pinned UK Core package and the exact BaRS package containing the profiles used by each message or REST interaction.

```text
java -jar validator_cli.jar example.json \
  -version 4.0.1 \
  -ig fhir.r4.ukcore.stu2#<pinned-version> \
  -ig <bars-package>#<pinned-version> \
  -profile <applicable-profile-canonical>
```

Every Appointment, ServiceRequest, DocumentReference, Slot/search Bundle, message Bundle, OperationOutcome and CapabilityStatement example should be validated separately.

## Standards references

- [HL7 FHIR R4 RESTful API](https://hl7.org/fhir/R4/http.html)
- [HL7 FHIR R4 PATCH](https://hl7.org/fhir/R4/http.html#patch)
- [HL7 FHIR R4 Operations](https://hl7.org/fhir/R4/operations.html)
- [HL7 FHIR R4 Search](https://hl7.org/fhir/R4/search.html)
- [HL7 FHIR R4 Appointment](https://hl7.org/fhir/R4/appointment.html)
- [HL7 FHIR R4 ServiceRequest](https://hl7.org/fhir/R4/servicerequest.html)
- [HL7 FHIR R4 DocumentReference](https://hl7.org/fhir/R4/documentreference.html)
- [HL7 FHIR R4 Bundle](https://hl7.org/fhir/R4/bundle.html)
- [UK Core R4 STU2 Implementation Guide](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2?version=2.0.2)
- [UKCore-Appointment](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/ProfilesandExtensions/Profile-UKCore-Appointment?version=current)
- [UKCore-ServiceRequest](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/ProfilesandExtensions/Profile-UKCore-ServiceRequest?version=current)
- [UK Core MustSupport guidance](https://simplifier.net/guide/UK-Core-Implementation-Guide-STU2/Home/Guidance/MustSupport-Guidance?version=current)

## Overall conclusion

The BaRS OAS contains a mixture of inherited framework defects and BaRS-specific resource-modelling errors. The exact reuse of the incomplete OperationOutcome component and repetition of the create, media-type, schema and CapabilityStatement patterns indicate that remediation should begin in the shared specification source.

After shared fixes, BaRS still requires focused correction of its PATCH semantics, ServiceRequest representation, search Bundle structure, DocumentReference content format, UK Core Appointment support and CapabilityStatement interaction matrix. Until those changes are made and the examples pass formal validation against the relevant packages, the OAS should be treated as a draft FHIR-aligned contract rather than a conformant FHIR R4/UK Core interface.
