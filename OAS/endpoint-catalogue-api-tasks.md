# Endpoint Catalogue API — Remediation Checklist

Tasks to bring the new `endpoint-catalogue-api.json` back in line with the FHIR R4 / UK Core review (`endpoint-catalog-oas-review-changes.md`).

**Context:** The new `endpoint-catalogue-api.json` is a fresh baseline that regressed almost all the fixes previously applied to `endpoint-catalog-api.json` (now in `OAS/old/`). Most tasks below are re-applications of fixes already made once. Two items are genuinely new (the `_id` search additions and the version number). Item numbers in brackets refer to the review report.

**Priority key:** Must / Should / Could / Won't (this release).

---

## Open decisions (resolve first)

- [x] **Version number** — set to `1.0.0-alpha` (first release). Aligned all 5 occurrences: `info.version`, the `/metadata` CapabilityStatement `version` + `software.version`, and the two `Capability` schema examples. ✅ Done.
- [ ] **`_id` search additions** — the new file adds `_id` on `GET /HealthcareService` and `_has:HealthcareService:endpoint:_id` on `GET /Endpoint`. Confirm these are intended to stay (they resolve process conflicts #35/#36 by extending the API rather than editing the process docs).
- [ ] **Approach** — re-apply each fix by hand, or port the fixed content from `OAS/old/endpoint-catalog-api.json` via a careful merge that preserves the new file's additions (`_id`, `_has:_id`, version).

---

## Must (required for a correct/conformant alpha)

- [ ] **[#1] Period casing** — `Period`/`Start`/`End` → `period`/`start`/`end` in the Endpoint schema, EndpointBundle schema, and all examples.
- [ ] **[#2] connectionType/payloadType query params as tokens** — reference `system|value` token string schemas, not nested `coding` object schemas. Remove the `ConnectionType`/`PayloadType` object schemas; add token schemas. Fix the connectionType schema example system (currently shows `endpoint-payload-type`).
- [ ] **[#3] Remove `-epc` system** — replace `endpoint-payload-type-epc` with standard `endpoint-payload-type` everywhere (~10 occurrences).
- [ ] **[#5] Identifier system scheme** — `http://fhir.nhs.uk` → `https://fhir.nhs.uk` on HealthcareService identifiers.
- [ ] **[#6] product-id casing** — `https://fhir.nhs.uk/id/product-id` → `/Id/product-id` (8 occurrences).
- [ ] **[#7] Rename search params** — `ConnectionType` → `connection-type`, `PayloadType` → `payload-type` (param `name` field only).
- [ ] **[#8] Rename identifier params** — `Endpoint.identifier` / `HealthcareService.identifier` → `identifier`.
- [ ] **[#9] organization search param** — CapabilityStatement already has `organization` as `reference` (good). Align `HCProvidedBy_QParam` name `HealthcareService.providedBy` → `organization.identifier` (reference chaining).
- [ ] **[#13] managingOrganization cardinality** — model `Endpoint.managingOrganization` as a single object (`0..1`), not an array (3 schemas + 8 examples).
- [ ] **[#14] Endpoint.header** — model as FHIR `string 0..*` (array of connection headers); remove the `public`/`private` visibility usage; update the 2 operation descriptions to stop attributing address visibility to `header`.
- [ ] **[#15] connectionType as Coding** — model `Endpoint.connectionType` as a flat FHIR `Coding` (`system`/`code`/`display`/`version`/`userSelected`), not `CodeableConcept` `coding[]`. Update description refs to `connectionType.code`. Leave `payloadType` as `CodeableConcept[]`.
- [ ] **[#16] Media type version param** — `application/fhir+json;version=1.4.0` → `application/fhir+json` on the 4 request bodies.
- [ ] **[#17] PUT request bodies** — add `requestBody` to `PUT /HealthcareService/{id}`, `PUT /Endpoint/{id}`, `PUT /Endpoint/{id}/$template`.
- [ ] **[#19] providedBy cardinality** — model `HealthcareService.providedBy` as a single object (`0..1`), not an array (2 schemas + 5 examples).
- [ ] **[#22] Remove Identifier.display** — remove the invalid `display` from all 12 `Identifier` objects (examples + schemas). Keep valid `Coding.display` and `Reference.display`.
- [ ] **[#25] Remove custom List profile** — remove `EPC-EndpointList` profile and `EPC-list-code` from List; use base FHIR `List`; drop `List.code` (priority semantics live in `orderedBy = priority`).
- [ ] **[#29] Remove update-as-create** — remove the `201` response and the upsert wording from all 4 PUT operations; leave `updateCreate` absent (upsert not supported).
- [ ] **[#33] Parent-template extension** — re-add the parent-template `extension` (`valueReference` to the parent Template Endpoint) to the Endpoint schema, the EndpointBundle nested resource, and the 5 Endpoint examples. Not on templates. Confirm/replace the extension `url` canonical (currently `http://hl7.org`, which is not a valid extension canonical).
- [ ] **[#34/#37] Process alignment** — after the OAS param renames (#7/#8) and connectionType flattening (#15), update process docs `IP001`–`IP004`: `$template` `productId` → `identifier`, kebab-case params, flat `connectionType`.

---

## Should (important, but alpha can proceed short-term)

- [ ] **[#11] Stale Capability schema examples** — update `version`, `publisher`, `date` to match the info block / `/metadata` (depends on the version decision above).
- [ ] **[#18] HealthcareService.type** *(team)* — add `type` (`CodeableConcept 0..*`, UK Core MustSupport) to the HealthcareService schema + bundle. Do **not** add to `required`. Blocked on confirming the value-set / system binding.
- [ ] **[#23] resourceType enum (Part A)** — enum-lock `resourceType` on `Endpoint`, `EndpointTemplate`, `HealthcareService` (`FhirList` already done). Parts B & C (required-cardinality via create/response schema split) deferred with #28.
- [ ] **[#26] OperationOutcome constraints** — enforce base/UK Core structure (`resourceType` enum, `issue` 1..*, `severity`/`code` required + value sets). Tie to the #4 naming decision.
- [ ] **[#28] Server-managed fields in create examples** — remove `id`, `meta.lastUpdated`, `meta.versionId` from create examples. Tie to the #23 schema split.
- [ ] **[#31] UUID-only ids** — reconsider `format: uuid` on logical ids / path params (~41 occurrences) vs the broader FHIR `id` datatype. If UUID-only is an intentional EPC restriction, document it; otherwise relax the pattern.
- [ ] **[#38] Stale process-doc details** — update `IP001`–`IP004` for `;version=`, `-epc`, `/id/`, and premature `UKCore-HealthcareService` profile claims (once the OAS is re-fixed).

---

## Could (desirable polish, low urgency)

- [ ] **[#10] Accept header example** — remove the trailing semicolon: `application/fhir+json;` → `application/fhir+json`.
- [ ] **[#12] Capability format example** — `xml` → `json`.
- [ ] **[#24] CapabilityStatement advertising** *(deferred, alpha)* — profiles / `updateCreate` / `$template` advertising; parked with `/metadata` for alpha.
- [ ] **[#32] CapabilityStatement/OAS sync** *(deferred, alpha)* — parked with `/metadata` for alpha.

---

## Won't (this release) — confirm only, no change

- [ ] **[#4] OperationOutcome naming** *(team)* — `OperationalOutcome` → `OperationOutcome` or `NHSDigital-OperationOutcome`. Awaiting team decision. *(Note: this one is still an open decision, not strictly "won't" — kept here to flag it needs sign-off.)*
- [ ] **[#20] Create returns 200, no Location** — by-design to match BaRS. Confirm no change. Note `POST /List` currently returns `201`; decide whether to align it to `200` for consistency.
- [ ] **[#21] `$template` PUT/DELETE** *(team)* — non-conformant FHIR grammar but must stay (hidden template feature). Record as accepted deviation. Optional: consider dropping the `$` (`/Endpoint/{id}/template`) to regain conformance with no behaviour change.
- [ ] **[#27] Endpoint.address redaction** — deferred pending the requirements review (same visibility concern as #14).
- [ ] **[#30] Accept header required** — stays `required: true` by design (WAF/versioning/intent). Confirm no change.

---

## New reconciliation (from the new file's additions)

- [ ] **[#35/#36] `_id` search reconciliation** — the new file added `_id` on `GET /HealthcareService` (`ServiceId_QParam`) and `_has:HealthcareService:endpoint:_id` on `GET /Endpoint`. If kept, update report items #35 and #36 to "resolved by extending the OAS" and align the process docs to use these. Also note the new file still exposes `_include` on `GET /List` — decide if that stays.

---

*Generated from the review report `endpoint-catalog-oas-review-changes.md`. Tick items as they are completed; keep the report as the detailed reference for each fix (with before/after).*
