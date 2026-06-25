# EPC MVP Deferral — Endpoint Ordering (List Resource)

**Series:** EPC MVP Scope Deferrals
**Document:** 3 of N

---

## Requirement references

| Requirement | Description |
|---|---|
| EPCFUNC-06 | Endpoint ordering — expressing communication preference for multi-protocol services |

---

## Purpose

This document analyses the deferral of Endpoint ordering via the FHIR List resource from
the EPC MVP. The full List-based ordering capability remains in scope for the final
solution — the MVP operates without explicit priority ordering.

---

## What is Endpoint ordering

The FHIR `List` resource provides a priority-ordered collection of Endpoints for a
HealthcareService. It allows a service owner to express a **communication preference** —
the order in which a sender should attempt to contact the Endpoints associated with a
service (e.g. try FHIR REST first, fall back to ITK3, then email).

This capability was designed primarily for DUEC (Directory of Urgent and Emergency Care)
services which support multiple connection protocols per service.

---

## What is being deferred

| Deferred capability | Detail |
|---|---|
| `POST /List` | Create a priority-ordered Endpoint list for a HealthcareService |
| `GET /List` | Search for lists by HealthcareService |
| `GET /List/{id}` | Read a specific list |
| `GET /List?_include=List:item` | Retrieve a list with all referenced Endpoints in one call |
| `PUT /List/{id}` | Reorder, add, or remove Endpoints from the priority list |
| `DELETE /List/{id}` | Remove a priority list |
| Auto-creation of List on `POST /HealthcareService` | Automatic List creation when a HealthcareService is created |
| Auto-synchronisation of List on `PUT /HealthcareService` | Keeping List entries in sync when HealthcareService endpoint references change |
| List-based consumer patterns | Two-step retrieval (get List → filter by type → apply order) |
| List ownership/authorisation | Ownership derived from referenced HealthcareService's `providedBy` |
| DynamoDB List table | Storage for List resources |
| Lambda List handler | CRUD operations on List resources |

---

## What is retained for MVP

| Retained capability | Detail |
|---|---|
| `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}` | Retrieve Endpoints for a service (unordered) |
| `GET /Endpoint` with `ConnectionType` and `PayloadType` filters | Consumers filter by the protocol/payload type they support |
| `HealthcareService.endpoint[]` | Unordered array of Endpoint references on the HealthcareService |
| Multiple Endpoints per HealthcareService | A service can still have multiple Endpoints — consumers select by type |

---

## Why this can be deferred

### MVP consumers don't need ordering

The primary MVP consumers are **BaRS senders**. A BaRS sender looks up a HealthcareService
and retrieves the Endpoint matching `connectionType=hl7-fhir-rest` and `payloadType=bars`.
There is typically **one Endpoint per type per service** — there is nothing to order.

The ordering requirement comes from **DUEC**, where a single service has multiple
connection methods (FHIR REST, ITK3, email) and the sender needs to know which to try
first. DUEC is not an MVP consumer.

### Single-endpoint-per-type is the BaRS norm

For BaRS pharmacy services (the MVP use case):
- Each pharmacy has one HealthcareService
- Each HealthcareService has one Endpoint (FHIR REST / BaRS)
- There is no priority decision to make — only one option exists

### Consumer behaviour without ordering

Without a List resource, consumers simply:
1. Query `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}&ConnectionType=hl7-fhir-rest&PayloadType=bars`
2. Receive one Endpoint (or zero if none exists)
3. Use it

No ambiguity, no priority logic required.

---

## Implications of deferral

#### 1. No multi-protocol failover for services with multiple Endpoints

**Impact: Low (for MVP)**

If a service has multiple Endpoints of different types, consumers cannot determine which
to try first. They must select based on `connectionType` and `payloadType` alone — there
is no server-expressed priority.

| With List | Without List (MVP) |
|---|---|
| Consumer retrieves priority List → tries Endpoint at index 0 → falls back to index 1 | Consumer queries by type → gets all matching Endpoints → no defined order for failover |

**Mitigation:** MVP services (BaRS pharmacies) have one Endpoint per type. Multi-protocol
services (DUEC) are not onboarded until ordering is delivered.

---

#### 2. No dynamic priority changes (e.g. promoting fallback during outage)

**Impact: Low (for MVP)**

Service owners cannot promote a secondary Endpoint to primary during an outage by
resequencing the List. The only mechanism to redirect traffic is to change `status` on
Endpoints (suspend the primary, so the secondary becomes the only active one).

| With List | Without List (MVP) |
|---|---|
| `PUT /List/{id}` — resequence entries to promote ITK3 over FHIR REST | Suspend the FHIR REST Endpoint → only ITK3 remains visible |

**Mitigation:** Status-based failover (suspend/unsuspend) provides a coarser but functional
alternative for the MVP. This is acceptable because BaRS services have a single Endpoint —
if it's down, the service is unavailable regardless of ordering.

---

#### 3. `POST /HealthcareService` does not auto-create a List

**Impact: None (for MVP)**

Without the List feature, creating a HealthcareService is simpler — no transactional
side-effect creating a companion List resource. The `endpoint[]` array on the
HealthcareService is just an unordered set of references.

This is a simplification, not a loss.

---

#### 4. `HealthcareService.endpoint[]` order has no defined semantics

**Impact: Low**

FHIR R4 does not guarantee array order on `HealthcareService.endpoint[]`. Without the
List resource to provide authoritative ordering, consumers must not assume the array
position means anything.

**Mitigation:** For MVP, there is typically one Endpoint per type — array order is
irrelevant when there's only one entry of each type.

---

## API changes for MVP

#### Removed from the API

| Item | Detail |
|---|---|
| `/List` endpoint (all operations) | `POST`, `GET`, `GET /{id}`, `PUT /{id}`, `DELETE /{id}` — entire resource type not exposed |
| `_include=List:item` | Not available (no List resource to include from) |
| Auto-creation logic on `POST /HealthcareService` | No companion List created when a service is created |
| Auto-sync logic on `PUT /HealthcareService` | No List synchronisation when endpoint references change |
| List ownership check | No authorisation logic for List writes (derived from HealthcareService `providedBy`) |

#### Simplified for MVP

| Item | Detail |
|---|---|
| **DynamoDB schema** | No List table required. Reduces infrastructure and IaC complexity. |
| **Lambda handlers** | No List handler Lambda. One fewer function to deploy, test, and monitor. |
| **Consumer contract** | Consumers query Endpoints directly — no two-step "get List then get Endpoints" pattern. |
| **HealthcareService write path** | No transactional side-effect (List creation/sync). Simpler write logic. |
| **OAS** | Smaller API surface — no `/List` paths, no List-related schemas or examples. |

#### Unchanged between MVP and final solution

| Aspect | Detail |
|---|---|
| `GET /Endpoint` with `_has` | Same query pattern — consumers look up Endpoints by HealthcareService ID. |
| `ConnectionType` / `PayloadType` filtering | Same filters. Consumers select Endpoints by type. |
| `HealthcareService.endpoint[]` | Same unordered reference array. |
| Template resolution | Endpoints still inherit from Templates at read time. |
| Status and period filtering | Visibility rules unchanged — only active, in-period Endpoints returned. |
| All Endpoint and HealthcareService CRUD | Unchanged. |

#### OAS impact

| Item | MVP | Final solution |
|---|---|---|
| `/List` paths | Not present | `POST`, `GET`, `GET /{id}`, `PUT /{id}`, `DELETE /{id}` |
| List resource schema | Not present | Full List schema with EPC-EndpointList profile |
| `_include=List:item` parameter | Not present | Supported on `GET /List` |
| Auto-creation documentation | Not present | Documented behaviour on `POST /HealthcareService` |

---

## Decision

| Decision | Rationale |
|---|---|
| **Defer endpoint ordering (List) from MVP** | The primary MVP consumer (BaRS) has one Endpoint per type per service — there is nothing to order. The List resource adds API surface, infrastructure (DynamoDB table, Lambda handler), and transactional complexity (auto-creation, auto-sync) that is not needed until DUEC or other multi-protocol consumers are onboarded. |

---

## When to deliver

Endpoint ordering should be delivered when:

1. **DUEC onboards** — DUEC services have multiple Endpoints per service with an explicit
   communication preference (FHIR REST → ITK3 → email). Ordering is a hard requirement
   for DUEC.
2. **Any consumer requires multi-protocol failover** — if a service has multiple Endpoints
   of the same `connectionType` and the consumer needs to know which to try first.
3. **Service owners need dynamic priority management** — the ability to promote/demote
   Endpoints during outages without changing their status.

---

## Summary: MVP vs final consumer experience

| Scenario | MVP (no List) | Final (with List) |
|---|---|---|
| BaRS sender looking up a pharmacy | Queries by type → gets one Endpoint → uses it | Same (List exists but contains one entry — no difference) |
| DUEC sender looking up a UTC | Not supported (DUEC not onboarded) | Queries List → gets priority order → tries in sequence |
| Service owner promoting fallback during outage | Suspend primary Endpoint → only secondary visible | Resequence List → secondary promoted to index 0 |
| Consumer needing all Endpoints in one call | `GET /Endpoint?_has:...` — unordered | `GET /List?_include=List:item` — ordered |

---

## Related documents

| Document | Description |
|----------|-------------|
| [Endpoint Ordering using the FHIR List Resource](../endpoint-ordering-with-list.md) | Full List design (target state) |
| [DUEC Endpoints](../duec-endpoints.md) | DUEC multi-protocol model requiring ordering |
| [Managing Endpoints](../manage-endpoint.md) | Endpoint lifecycle operations |
| [Managing HealthcareServices](../manage-healthcare-service.md) | HealthcareService lifecycle operations |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [mvp-deferral-rbac.md](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | [mvp-deferral-observability.md](./mvp-deferral-observability.md) |
| 3 | Endpoint Ordering (List) | This document |
