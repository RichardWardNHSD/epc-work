# Endpoint Catalogue API — Remediation Checklist

Tasks to bring the new `endpoint-catalogue-api.json` back in line with the FHIR R4 / UK Core review (`endpoint-catalog-oas-review-changes.md`).

**Context:** The new `endpoint-catalogue-api.json` is a fresh baseline that regressed almost all the fixes previously applied to `endpoint-catalog-api.json` (now in `OAS/old/`). Most tasks below are re-applications of fixes already made once. Two items are genuinely new (the `_id` search additions and the version number). Item numbers in brackets refer to the review report.

**Priority key:** Must / Should / Could / Won't (this release).

---

## Open decisions (resolve first)

- [x] **Version number** — set to `1.0.0-alpha` (first release). Aligned all occurrences in `info.version`. ✅ Done. (The `/metadata` and `Capability` version examples were subsequently removed — see below.)
- [x] **Remove `/metadata` + CapabilityStatement for alpha** — removed the `/metadata` path, the `Capability` schema, and the `Metadata` tag. Zero dangling references; valid JSON. This makes #11, #12, #24, #32 N/A for alpha. The `_id` reconciliation also removed the `_id` searchParam from the (now-deleted) CapabilityStatement. ✅ Done. Backup: `/tmp/cat-backup-pre-metadata.json`.
- [x] **[#39] "Catalog" → "Catalogue" spelling** — corrected the US spelling in `info.title` ("Endpoint Catalog API" → "Endpoint Catalogue API") and `info.description`. The server URLs / API slugs already used `endpoint-catalogue`/`api-catalogue`. No "Catalog" (non-Catalogue) spelling remains. ✅ Done.
- [ ] **`_id` search additions** — the new file adds `_id` on `GET /HealthcareService` and `_has:HealthcareService:endpoint:_id` on `GET /Endpoint`. Confirm these are intended to stay (they resolve process conflicts #35/#36 by extending the API rather than editing the process docs).
- [ ] **Approach** — re-apply each fix by hand, or port the fixed content from `OAS/old/endpoint-catalog-api.json` via a careful merge that preserves the new file's additions (`_id`, `_has:_id`, version).

---

## Must (required for a correct/conformant alpha)

- [x] **[#1] Period casing** — `Period`/`Start`/`End` → `period`/`start`/`end` in the Endpoint schema, EndpointBundle schema, and all examples. ✅
- [x] **[#2] connectionType/payloadType query params as tokens** — params already used inline `system|code` token string schemas in this file; removed the two orphaned `ConnectionType`/`PayloadType` object schemas (0 `$ref`s). connectionType schema example system already correct via #15. ✅
- [x] **[#3] Remove `-epc` system** — replaced `endpoint-payload-type-epc` with standard `endpoint-payload-type` (11 occurrences). ✅
- [x] **[#5] Identifier system scheme** — `http://fhir.nhs.uk` → `https://fhir.nhs.uk` (8 occurrences). ✅
- [x] **[#6] product-id casing** — `https://fhir.nhs.uk/id/product-id` → `/Id/product-id` (8 occurrences). ✅
- [x] **[#7] Rename search params** — `ConnectionType` → `connection-type`, `PayloadType` → `payload-type` (param `name` field). ✅
- [x] **[#8] Rename identifier params** — `Endpoint.identifier` / `HealthcareService.identifier` → `identifier`. ✅
- [x] **[#9] organization search param** — CapabilityStatement already has `organization` as `reference`. Aligned `HCProvidedBy_QParam` name `HealthcareService.providedBy` → `organization.identifier` (reference chaining). ✅
- [x] **[#13] managingOrganization cardinality** — modelled `Endpoint.managingOrganization` as a single object (`0..1`), not an array (11 occurrences). ✅
- [x] **[#14] Endpoint.header** — modelled as FHIR `string 0..*` (array); removed `public`/`private` usage from schema (3) + examples (8); rewrote the 2 operation descriptions to attribute address visibility to ownership, not `header`. ✅
- [x] **[#15] connectionType as Coding** — modelled `Endpoint.connectionType` as a flat FHIR `Coding` (`system`/`version`/`code`/`display`/`userSelected`) in 3 schemas + 8 examples; description ref → `connectionType.code`; `payloadType` left as `CodeableConcept[]`. ✅
- [x] **[#16] Media type version param** — `application/fhir+json;version=1.4.0` → `application/fhir+json` on the 4 request bodies. ✅
- [x] **[#17] PUT request bodies** — added `requestBody` to `PUT /HealthcareService/{id}`, `PUT /Endpoint/{id}`, `PUT /Endpoint/{id}/$template`. ✅
- [x] **[#19] providedBy cardinality** — modelled `HealthcareService.providedBy` as a single object (`0..1`), not an array (6 occurrences). ✅
- [x] **[#22] Remove Identifier.display** — removed the invalid `display` from all 12 `Identifier` objects (examples + schemas). Valid `Coding.display` and `Reference.display` preserved. ✅
- [x] **[#25] Remove custom List profile** — removed `EPC-EndpointList` profile and `EPC-list-code`; List now uses base FHIR `List`; `List.code` dropped (priority semantics remain in `orderedBy = priority`). ✅
- [x] **[#29] Remove update-as-create** — removed the `201` response from all 4 PUT operations and the upsert wording from all 4 descriptions; `updateCreate` left absent. ✅
- [x] **[#33] Parent-template extension** — re-added the parent-template `extension` (`valueReference` to the parent Template Endpoint) to the Endpoint schema, the EndpointBundle nested resource, and the 5 Endpoint examples (not on templates). ✅ ⚠️ **Still TODO:** the extension `url` is `http://hl7.org`, which is not a valid extension canonical — replace with a real `StructureDefinition` URL when available.
- [ ] **[#34/#37] Process alignment** — after the OAS param renames (#7/#8) and connectionType flattening (#15), update process docs `IP001`–`IP004`: `$template` `productId` → `identifier`, kebab-case params, flat `connectionType`.

---

## Should (important, but alpha can proceed short-term)

- [x] **[#11] Stale Capability schema examples** — **N/A: the `Capability` schema was removed with `/metadata`** (see below). No longer applicable. ✅
- [ ] **[#18] HealthcareService.type** *(team)* — add `type` (`CodeableConcept 0..*`, UK Core MustSupport) to the HealthcareService schema + bundle. Do **not** add to `required`. Blocked on confirming the value-set / system binding.
- [x] **[#23] resourceType enum (Part A)** — enum-locked `resourceType` on `Endpoint`, `EndpointTemplate`, `HealthcareService` (`FhirList` already done). ✅ Parts B & C (required-cardinality via create/response schema split) still pending, deferred with #28.
- [ ] **[#26] OperationOutcome constraints** — enforce base/UK Core structure (`resourceType` enum, `issue` 1..*, `severity`/`code` required + value sets). Tie to the #4 naming decision.
- [ ] **[#28] Server-managed fields in create examples** — remove `id`, `meta.lastUpdated`, `meta.versionId` from create examples. Tie to the #23 schema split.
- [ ] **[#31] UUID-only ids** — reconsider `format: uuid` on logical ids / path params (~41 occurrences) vs the broader FHIR `id` datatype. If UUID-only is an intentional EPC restriction, document it; otherwise relax the pattern.
- [ ] **[#38] Stale process-doc details** — update `IP001`–`IP004` for `;version=`, `-epc`, `/id/`, and premature `UKCore-HealthcareService` profile claims (once the OAS is re-fixed).

---

## Could (desirable polish, low urgency)

- [x] **[#10] Accept header example** — removed the trailing semicolon: `application/fhir+json;` → `application/fhir+json`. ✅
- [x] **[#12] Capability format example** — **N/A: the `Capability` schema was removed with `/metadata`** (see below). No longer applicable. ✅
- [x] **[#24] CapabilityStatement advertising** — **N/A for alpha: `/metadata` and the CapabilityStatement have been removed** (see below). No statement to advertise profiles/operations in. Revisit if/when a CapabilityStatement is reintroduced post-alpha. ✅
- [x] **[#32] CapabilityStatement/OAS sync** — **N/A for alpha: `/metadata` removed** (see below). No published statement to keep in sync. Revisit post-alpha. ✅

---

## Won't (this release) — confirm only, no change

- [ ] **[#4] OperationOutcome naming** *(team)* — `OperationalOutcome` → `OperationOutcome` or `NHSDigital-OperationOutcome`. Awaiting team decision. *(Note: this one is still an open decision, not strictly "won't" — kept here to flag it needs sign-off.)*
- [ ] **[#20] Create returns 200, no Location** — by-design to match BaRS. Confirm no change. Note `POST /List` currently returns `201`; decide whether to align it to `200` for consistency.
- [ ] **[#21] `$template` PUT/DELETE** *(team)* — non-conformant FHIR grammar but must stay (hidden template feature). Record as accepted deviation. Optional: consider dropping the `$` (`/Endpoint/{id}/template`) to regain conformance with no behaviour change.
- [ ] **[#27] Endpoint.address redaction** — deferred pending the requirements review (same visibility concern as #14).
- [ ] **[#30] Accept header required** — stays `required: true` by design (WAF/versioning/intent). Confirm no change.

---

## New reconciliation (from the new file's additions)

- [x] **[#35/#36] `_id` search reconciliation** — **Decision applied (both `_id` forms removed):**
  - **Removed** `GET /HealthcareService?_id={uuid}` (`ServiceId_QParam` component + its use) — redundant with `GET /HealthcareService/{id}` (direct read by logical id). → Report **#36: `_id` query not supported; use `GET /HealthcareService/{id}`**.
  - **Removed** `_has:HealthcareService:endpoint:_id` on `GET /Endpoint` (from the `_has` param description and 4 operation descriptions; also removed from the CapabilityStatement before `/metadata` itself was deleted). Only `_has:HealthcareService:endpoint:identifier` remains. → Report **#35: the `_id` reverse-chain is NOT supported**; process steps using `_has:...:_id` must switch to `_has:...:identifier`.
  - `GET /List` `_include=List:item` retained (legitimate on List). ✅
  - ⚠️ **Follow-up:** report #35/#36 wording and the process docs (IP001–IP004) still need updating to match this decision — see the process-doc tasks below.

---

*Generated from the review report `endpoint-catalog-oas-review-changes.md`. Tick items as they are completed; keep the report as the detailed reference for each fix (with before/after).*
