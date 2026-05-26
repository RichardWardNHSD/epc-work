# Endpoint Ordering — Options Comparison (Condensed)

This document summarises three approaches to expressing endpoint priority/ordering in the
Endpoint Catalog. For full implementation details, see the
[complete options document](./endpoint-ordering-options-full.md).

---

## The Problem

Services in the Endpoint Catalog may have multiple Endpoints. Consumers need to know which
Endpoint to try first (primary), which to try second (failover), etc. Neither the FHIR
`Endpoint` nor `HealthcareService` resource provides a native mechanism for expressing this
priority.

Bundle entry order is **not** a reliable mechanism because:
- Pagination can split results across pages
- Intermediaries (caches, CDNs, API gateways) may reorder entries
- The FHIR specification does not mandate Bundle entry order preservation
- Future implementers may not realise order is semantically important

**Explicit metadata is required for resilient interoperability.**

---

## Option A: FHIR List Resource (Current Design)

### Approach

A separate `List` resource holds an ordered array of Endpoint references for each
HealthcareService. Array position = priority (index 0 = highest).

### Key Design Points

- `List.entry[0]` = highest priority, `entry[1]` = next, etc.
- `List.subject` references the HealthcareService
- `List.orderedBy` = `priority`
- One List per HealthcareService (auto-created on `POST /HealthcareService`)
- Auto-synchronised on `PUT`/`PATCH` of HealthcareService
- Consumers retrieve via `GET /List?subject=HealthcareService/{id}&_include=List:item`

### API Surface

| Operation | Purpose |
|-----------|---------|
| `POST /List` | Create a priority list |
| `GET /List?subject=...` | Retrieve the list (with optional `_include=List:item` for Endpoints) |
| `GET /List/{id}` | Read a specific list |
| `PUT /List/{id}` | Replace/reorder the list |
| `DELETE /List/{id}` | Remove the list |

### Consumer Pattern

```
Step 1: GET /List?subject=HealthcareService/{id}&status=current  → get priority order
Step 2: GET /Endpoint?...&ConnectionType=hl7-fhir-rest           → get filtered endpoints
Client: apply List order to filtered results
```

Or single-call: `GET /List?subject=...&_include=List:item` (returns List + all Endpoints)

### Example

```json
{
  "resourceType": "List",
  "status": "current",
  "mode": "working",
  "orderedBy": { "coding": [{ "code": "priority" }] },
  "subject": { "reference": "HealthcareService/9f2c6f12-..." },
  "entry": [
    { "item": { "reference": "Endpoint/...001", "display": "Primary" } },
    { "item": { "reference": "Endpoint/...002", "display": "Secondary" } },
    { "item": { "reference": "Endpoint/...003", "display": "Fallback" } }
  ]
}
```

---

## Option B: Implicit List — API Does the Heavy Lifting (Enhancement of Option A)

### Approach

Uses the **same List resource as Option A** for storage, but the API automatically applies
the List order when returning Endpoint results. The consumer never needs to know the List
exists — Endpoints arrive pre-sorted in every response.

### Key Design Points

- List resource still exists (admins manage priority via `PUT /List/{id}`)
- Lambda looks up the List and sorts results server-side before returning
- Consumer receives Endpoints in priority order without any client-side logic
- If no List exists, falls back to server-determined order (backward compatible)
- Template resolution (connectionType, payloadType, address) AND ordering happen in the same Lambda call

### API Surface

No new operations for consumers. Existing `GET /Endpoint` and `GET /HealthcareService`
calls return ordered results automatically. List CRUD remains for admin use.

### Consumer Pattern

```
GET /Endpoint?_has:HealthcareService:endpoint:_id={id}&ConnectionType=hl7-fhir-rest
→ results arrive in priority order — use top-to-bottom
```

Single call. No client-side sorting. No List awareness needed.

### Advantages

- Simplest consumer experience (one call, pre-ordered results)
- Backward compatible (no consumer code changes)
- Template-aware (Lambda resolves types + applies order in one operation)

### Disadvantages

- Non-standard FHIR behaviour (search results not normally ordered by spec)
- Implicit contract (consumers may not realise order is meaningful)
- Harder to debug (List not visible without separate admin call)

---

## Option C: Priority Extension on Endpoint Resource (Alternative)

### Approach

A custom FHIR extension (`endpoint-priority`) directly on each Endpoint resource declares
its numeric priority rank within the context of its parent HealthcareService.

### Key Design Points

- `valuePositiveInt: 1` = highest priority (try first)
- Priority is per connection type within a HealthcareService
- No separate resource to manage — priority is intrinsic to the Endpoint
- No synchronisation logic needed
- Consumers sort by extension value after filtering

### API Surface

No additional API operations — uses existing Endpoint CRUD (`POST`, `PUT`, `PATCH`, `DELETE`
on `/Endpoint`).

### Consumer Pattern

```
GET /Endpoint?_has:HealthcareService:endpoint:_id={id}&ConnectionType=hl7-fhir-rest
→ sort results by extension[endpoint-priority].valuePositiveInt ascending
```

Single call, client-side sort.

### Example

```json
{
  "resourceType": "Endpoint",
  "id": "ep-001",
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
      "valuePositiveInt": 1
    }
  ],
  "status": "active",
  "name": "Primary BaRS Endpoint",
  "connectionType": { "coding": [{ "code": "hl7-fhir-rest" }] },
  "payloadType": [{ "coding": [{ "code": "bars" }] }],
  "address": "https://provider.example.nhs.uk/FHIR/R4"
}
```

### DoS Example (Bradford Royal UTC)

| Endpoint | connectionType | payloadType | Priority | Role |
|----------|---------------|-------------|----------|------|
| ep-001 | ITK | CDA | 1 | Primary |
| ep-002 | ITK | CDA | 2 | Copy |
| ep-003 | HL7 FHIR | Scheduling | 1 | Primary |

Two ITK/CDA endpoints with explicit ranking — no ambiguity about primary vs copy.

---

## Comparison

| Aspect | Option A: Explicit List | Option B: Implicit List | Option C: Priority Extension |
|--------|------------------------|------------------------|------------------------------|
| **Ordering mechanism** | Separate `List`, array position | Same List, applied server-side | Extension on each Endpoint |
| **Explicit semantics** | Yes | Yes (but hidden from consumer) | Yes |
| **Survives Bundle reordering** | Yes | Yes (server controls order) | Yes |
| **Survives pagination** | Yes | Yes | Yes |
| **Additional API surface** | 5 new operations (CRUD on /List) | None for consumers; List CRUD for admin | None |
| **Synchronisation complexity** | High — auto-sync on HS create/update | Same as Option A (List still exists) | None |
| **Consumer complexity** | 1-2 calls + client-side ordering | 1 call, no sorting needed | 1 call + sort by extension |
| **FHIR conformance** | Strong — List is purpose-built | Custom behaviour (documented) | Acceptable — extensions are standard |
| **Multiple endpoints same type** | Supported | Supported | Supported |
| **Risk of stale ordering** | If List drifts from Endpoints | Same as A | None — priority is on the Endpoint |
| **Operational overhead** | List lifecycle management | Same as A (but transparent to consumers) | Minimal |
| **Template model impact** | None — type-agnostic | None — type-agnostic | Global priority works; per-type is awkward |
| **Consumer awareness needed** | Must know about List | None — just use results in order | Must know to sort by extension |

---

## Impact of the Template Model on Option C

In the EPC architecture, `connectionType`, `payloadType`, and `address` live on the parent
**Template**, not on the Endpoint itself. The Lambda resolves this join at query time and
projects those fields onto the response. The bare Endpoint record has no knowledge of its
own connection type.

Additionally, **Endpoints can be shared across multiple HealthcareServices**. This is the
critical flaw for Option C:

| Issue | Detail |
|-------|--------|
| **Multi-service sharing** | An Endpoint referenced by Service A and Service B cannot have different priority values — there's only one extension on the resource. **This breaks the model entirely.** |
| Extension can't self-describe its scope | `endpoint-priority: 1` means "priority 1" — but of what type? And for which service? |
| Per-type priority requires Template awareness | Consumer must group by Template (after Lambda resolves) before priority values are meaningful within a type |

**Option A/B avoids both problems** because the List is scoped to a specific HealthcareService
(`List.subject`). Each service has its own List with independent ordering, and the same
Endpoint can appear at different positions in different Lists.

---

## Why Bundle Order Alone Is Insufficient

| Risk | Impact |
|------|--------|
| Pagination splits results | Priority 1 and 2 may be on different pages |
| Intermediaries reorder | Caches, CDNs, API gateways don't preserve entry order |
| Indexing layers reorder | Databases and search engines have their own sort |
| Implicit contracts are fragile | Future implementers won't know order matters |
| FHIR spec doesn't mandate order | No guarantee of preservation |

---

## Open Questions (For IOPS / DoS Leads)

| # | Question |
|---|----------|
| 1 | Should priority be per connection type or global across all endpoints on a service? |
| 2 | What is the correct extension URL namespace? |
| 3 | Should the extension be mandatory or optional (unranked = lowest priority)? |
| 4 | Does DoS need anything beyond simple numeric ranking (weighted routing, percentage-based)? |
| 5 | Can we take this to IOPS for a second opinion? |

---

## Recommendation

| Context | Recommendation |
|---------|---------------|
| **PoC** | Implement Option A (List) — already designed and documented |
| **Production (consumer experience)** | Enhance with Option B (implicit List) — simplest consumer experience, backward compatible |
| **Option C** | **Not recommended** — Endpoints shared across multiple HealthcareServices cannot have per-service priority; Template model adds further ambiguity |

The options are not mutually exclusive:
- **Option A + B together** is the recommended production path: List for storage/admin, implicit ordering for consumers
- **Option C** has a fundamental architectural flaw (multi-service Endpoint sharing) that makes it unsuitable for the EPC data model
