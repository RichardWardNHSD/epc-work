# Endpoint Ordering — Option B: Priority Extension on Endpoint Resource

## Context

This document presents an alternative approach to endpoint ordering in the Endpoint Catalog.
It should be read alongside [Option A: FHIR List Resource](./endpoint-ordering-with-list.md)
which uses a separate `List` resource to express priority order.

This alternative was raised during review of Option A, based on the following concerns:

---

## Concerns with Option A (List-based ordering)

### 1. Implicit ordering via Bundle position is fragile

The current DoS (Directory of Services) model implies endpoint priority through the order of
Endpoint objects in a Bundle response. This is problematic because:

- Pagination can reorder results across pages
- Intermediaries (caches, CDNs, API gateways) may reorder Bundle entries
- Downstream indexing layers may not preserve insertion order
- Future implementers may not realise that array position carries semantic meaning
- The FHIR specification does not guarantee Bundle entry order is preserved

While Option A (List resource) addresses this by making the List the authoritative source of
order rather than Bundle position, it introduces a separate resource that must be managed,
synchronised, and queried alongside the Endpoint and HealthcareService.

### 2. "Primary" and "Copy" labels are ambiguous in the current DoS model

In the DoS example below, endpoints are labelled "Primary" and "Copy" based on their position
in the list and whether another endpoint of the same connection type exists further down:

| Order | Transport | Interaction | Format | Business Scenario |
|-------|-----------|-------------|--------|-------------------|
| 1 | itk | primaryOutOfHourRecipientNHS111CDADocument-v2-0 | CDA | Primary |
| 2 | itk | copyRecipientNHS111CDADocument-v2-0 | CDA | Copy |
| 3 | http | scheduling | FHIR | Primary |

This labelling is derived from position + connection type matching — a fragile inference that
leaves too much to chance. In EPCS, these could instead be declared as two distinct endpoints
with different connection types (even if their URL is the same), making the relationship
explicit rather than implied.

### 3. EPCS should support multiple endpoints of the same connection type with explicit ranking

The generic requirement is: **EPCS should cater for two or more endpoints of the same
connection type and give them explicit ranking/priority/ordering.**

This is not well served by a separate List resource when the priority is an intrinsic property
of the Endpoint itself within the context of its parent HealthcareService.

---

## Option B: `endpoint-priority` Extension on the Endpoint Resource

### Approach

Add a custom FHIR extension directly on the `Endpoint` resource that declares its priority
(ranking/failover order) within the context of its parent HealthcareService.

### Extension Definition

```json
{
  "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
  "valuePositiveInt": 1
}
```

| Field | Value | Description |
|-------|-------|-------------|
| `url` | `https://fhir.nhs.uk/StructureDefinition/endpoint-priority` | Extension URL (could also be named `endpoint-ranking` or `endpoint-failover-priority`) |
| `valuePositiveInt` | 1, 2, 3... | Priority rank. 1 = highest priority (try first). Lower numbers = higher priority. |

### Example: Endpoint with priority extension

```json
{
  "resourceType": "Endpoint",
  "id": "e1a2b3c4-0000-0000-0000-000000000001",
  "meta": {
    "profile": ["https://fhir.nhs.uk/StructureDefinition/EPC-Endpoint"]
  },
  "extension": [
    {
      "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority",
      "valuePositiveInt": 1
    }
  ],
  "status": "active",
  "name": "Primary BaRS Endpoint",
  "connectionType": {
    "coding": [{ "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type", "code": "hl7-fhir-rest", "display": "HL7 FHIR" }]
  },
  "payloadType": [
    { "coding": [{ "code": "bars", "display": "BaRS" }] }
  ],
  "address": "https://provider.example.nhs.uk/FHIR/R4"
}
```

### Example: Multiple endpoints of the same connection type with ranking

Replicating the DoS example (Bradford Royal UTC) using explicit priority:

```json
[
  {
    "resourceType": "Endpoint",
    "id": "ep-001",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 1 }
    ],
    "status": "active",
    "name": "Primary CDA Endpoint",
    "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
    "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
    "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
  },
  {
    "resourceType": "Endpoint",
    "id": "ep-002",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 2 }
    ],
    "status": "active",
    "name": "Copy CDA Endpoint",
    "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
    "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
    "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
  },
  {
    "resourceType": "Endpoint",
    "id": "ep-003",
    "extension": [
      { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 1 }
    ],
    "status": "active",
    "name": "Primary FHIR Scheduling Endpoint",
    "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
    "payloadType": [{ "coding": [{ "code": "scheduling", "display": "Scheduling" }] }],
    "address": "ASID:200000064489"
  }
]
```

In this model:
- `ep-001` and `ep-002` are both ITK/CDA endpoints. Priority 1 vs 2 makes the primary/copy
  relationship explicit without relying on Bundle order or naming conventions.
- `ep-003` is a different connection type (FHIR). Its priority of 1 means it's the primary
  endpoint for that type — there's no ambiguity.
- Priority is **per connection type within a HealthcareService** — two endpoints can both be
  priority 1 if they have different connection types.

---

## API Impact

### No new resource type required

Unlike Option A, this approach does not introduce a `List` resource. Priority is stored
directly on the Endpoint, which means:

- No separate `POST /List`, `PUT /List`, `DELETE /List` operations
- No List synchronisation logic when HealthcareService endpoints change
- No need for consumers to make a separate call to retrieve ordering

### Retrieving ordered endpoints

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
```

The response Bundle contains Endpoint resources with the `endpoint-priority` extension.
The consumer sorts by `extension[endpoint-priority].valuePositiveInt` ascending to determine
the priority order.

### Filtering by connection type + priority

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-...&ConnectionType=hl7-fhir-rest HTTP/1.1
```

Returns only FHIR endpoints. The consumer sorts by the priority extension to determine
which is primary, secondary, etc.

### Setting/changing priority

Priority is set when creating or updating an Endpoint:

```http
PUT /Endpoint/ep-001 HTTP/1.1
```

```json
{
  "resourceType": "Endpoint",
  "id": "ep-001",
  "extension": [
    { "url": "https://fhir.nhs.uk/StructureDefinition/endpoint-priority", "valuePositiveInt": 2 }
  ],
  "status": "active",
  "name": "Demoted CDA Endpoint",
  "connectionType": { "coding": [{ "code": "ihe-xcpd", "display": "ITK" }] },
  "payloadType": [{ "coding": [{ "code": "cda", "display": "CDA" }] }],
  "address": "https://yga.oneoneone.nhs.uk:1880/systmonemhs/nhs111cda"
}
```

No separate List update is needed — the priority moves with the Endpoint.

---

## Comparison: Option A (List) vs Option B (Extension)

| Aspect | Option A: List Resource | Option B: Priority Extension |
|--------|------------------------|------------------------------|
| **Ordering mechanism** | Separate `List` resource with `entry[]` array position | `endpoint-priority` extension on each Endpoint |
| **Explicit semantics** | Yes — List is purpose-built for ordered collections | Yes — extension value is unambiguous |
| **Survives Bundle reordering** | Yes — List.entry[] is authoritative, not Bundle position | Yes — priority is on the resource itself |
| **Survives pagination** | Yes | Yes |
| **Additional API surface** | POST/GET/PUT/DELETE /List | None — uses existing Endpoint CRUD |
| **Synchronisation complexity** | High — List must stay in sync with HealthcareService.endpoint[] | None — priority is intrinsic to the Endpoint |
| **Multiple endpoints same type** | Supported (by List position) | Supported (by priority value per type) |
| **Consumer complexity** | Must retrieve List + Endpoints (1–2 calls), apply order client-side | Single call, sort by extension value |
| **FHIR conformance** | Strong — List is a standard FHIR resource for ordered collections | Acceptable — extensions are a standard FHIR extensibility mechanism |
| **Precedent** | Used in some UK FHIR implementations | Recommended by multiple AI-assisted architecture reviews; common pattern in FHIR IGs |
| **Supports "primary/copy" semantics** | Implicitly via position | Explicitly via numeric rank |
| **Operational overhead** | List lifecycle management, auto-creation, auto-sync | Minimal — just an extension field |
| **Risk of stale ordering** | If List gets out of sync with actual Endpoints | None — priority is on the Endpoint itself |

---

## Why Bundle order alone is insufficient

Even without a List or extension, some implementations rely on Bundle entry order to imply
priority. This is problematic because:

1. **Pagination can split results** — priority 1 and priority 2 may end up on different pages
2. **Intermediaries may reorder** — caches, CDNs, and API gateways do not guarantee entry order
3. **Downstream indexing layers may reorder** — databases and search engines have their own sort
4. **Future implementers may not realise order is semantic** — implicit contracts are fragile
5. **The FHIR specification does not mandate Bundle entry order preservation**

For operational failover logic, **explicit metadata is always preferable to implicit ordering**.

---

## Alternative considered: `_include` of List in Endpoint search

One option discussed was to include the List object in the `GET /Endpoint` response Bundle
using the `_include` mechanism (Endpoint entries with `search.mode = 'match'`, List with
`search.mode = 'include'`).

**Why this was rejected:**
- `_include` is designed for following references already present in resources
- `Endpoint` does not reference a `List` — the relationship is the other way around
- While technically defensible, it goes against FHIR design principles
- A `GET /List` with `_include=List:item` (where List is the match and Endpoints are included)
  is more natural but contradicts the core EPCS API design pattern of `GET /Endpoint` as the
  primary query

---

## Impact of the Template Model

### The Problem

In the EPC architecture, `connectionType`, `payloadType`, and `address` (URL) are **not
stored on the Endpoint resource itself** — they live on the parent **Template**. An Endpoint
is essentially an instance of a Template, inheriting those fields at read time when the
Lambda resolves the join.

The bare Endpoint record in the data store looks like:

```json
{
  "resourceType": "Endpoint",
  "id": "ep-001",
  "status": "active",
  "name": "Primary BaRS Endpoint",
  "managingOrganization": [{ "identifier": { "value": "R778" } }]
}
```

The `connectionType`, `payloadType`, and `address` are projected onto the response by the
Lambda at query time — they don't exist on the Endpoint itself.

### Implication for "Priority Per Connection Type"

If priority should be **per connection type** (two ITK/CDA endpoints can both be priority 1
alongside a FHIR endpoint also at priority 1), the extension alone can't express this because
the Endpoint doesn't know its own connection type until the Template is resolved.

### Resolution: Global Priority Model

Option B works cleanly with a **global priority** model:
- The extension is a single rank across all Endpoints on the service
- Consumer retrieves filtered Endpoints (Lambda resolves Templates and applies
  `ConnectionType`/`PayloadType` filters)
- Consumer sorts the filtered results by `endpoint-priority` extension value
- Two endpoints of different types can share the same priority value — it only matters
  within a type group after filtering

This is functionally equivalent to Option A's List position but without the separate resource.

### Where It Gets Awkward

If priority must be explicitly scoped per connection type (e.g. "priority 1 within CDA" vs
"priority 1 within FHIR"), Option B would need either:
- A composite extension referencing the Template
- Or the consumer to group by Template before interpreting priority

Option A avoids this entirely because the List is type-agnostic — ordering and type filtering
are orthogonal concerns.

---

## Open Questions

| # | Question | For |
|---|----------|-----|
| 1 | Should priority be **per connection type** (two endpoints of different types can both be priority 1) or **global across all endpoints** on a service? | Architecture / IOPS |
| 2 | What is the correct extension URL namespace? (`https://fhir.nhs.uk/StructureDefinition/endpoint-priority` vs `https://fhir.nhs.uk/StructureDefinition/endpoint-failover-priority`) | IOPS / FHIR governance |
| 3 | Should the extension be mandatory on all Endpoints, or optional (with unranked endpoints treated as lowest priority)? | Product |
| 4 | Does DoS have requirements beyond simple numeric ranking (e.g. weighted routing, percentage-based distribution)? | DoS leads |
| 5 | Can we take this proposal to IOPS for a second opinion on the extension approach vs the List approach? | Architecture |

---

## Recommendation

**For the PoC:** Implement Option A (List resource) as it is already designed and documented.

**For production:** Schedule a review with IOPS and DoS leads to evaluate Option B (extension)
as a simpler, lower-overhead alternative. The key question is whether the additional API
surface and synchronisation complexity of the List approach is justified, or whether an
extension on the Endpoint resource provides sufficient expressiveness with less operational
risk.

Both options solve the core problem (explicit, resilient ordering that survives Bundle
reordering and pagination). The choice comes down to:
- **Option A** if you value FHIR's purpose-built ordered collection semantics and want a
  single authoritative ordering document per service
- **Option B** if you value simplicity, lower API surface, and want priority to be an
  intrinsic property of the Endpoint rather than managed externally
