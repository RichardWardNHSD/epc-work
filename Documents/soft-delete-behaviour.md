# Soft Delete Behaviour and Impact on Child Records

## Overview

The Endpoint Catalog uses **soft delete** as the preferred mechanism for withdrawing
resources from active use without permanently destroying them. Soft-deleted resources
are retained in the data store, can be reinstated, and continue to exist for audit and
traceability purposes.

This document describes how soft delete works for each resource type and, critically,
how soft-deleting a parent resource affects its child records.

---

## Soft delete mechanisms by resource type

| Resource | Soft delete mechanism | Field changed | Soft-deleted value | Reinstated value |
|---|---|---|---|---|
| **Endpoint Template** | `PUT /Endpoint/{id}/$template` | `status` | `entered-in-error` | `active` |
| **Endpoint** | `PUT /Endpoint/{id}` | `status` | `entered-in-error` | `active` |
| **HealthcareService** | `PUT /HealthcareService/{id}` | `active` | `false` | `true` |

> **Note:** The List resource does not have a soft delete mechanism. Lists are deleted
> outright (`DELETE /List/{id}`) or have their entries removed via `PUT /List/{id}`.

---

## Resource hierarchy

Understanding the parent-child relationships is essential to understanding cascade
behaviour:

```
Endpoint Template (parent)
    │
    ├── Endpoint (child) ── inherits connectionType, payloadType, address, name, header
    │       │
    │       └── referenced by HealthcareService.endpoint[]
    │
    └── Endpoint (child)
            │
            └── referenced by HealthcareService.endpoint[]

HealthcareService
    │
    └── endpoint[] ── references one or more Endpoints
```

Key relationships:
- A **Template** has zero or more child **Endpoints**
- An **Endpoint** inherits all protocol fields from its parent Template at read time
- A **HealthcareService** references one or more Endpoints via its `endpoint[]` array
- A **HealthcareService** is not a child of a Template — it sits alongside

---

## Cascade behaviour: Soft-deleting a Template

### What happens

When a Template is soft-deleted (`status` set to `entered-in-error`):

| Aspect | Behaviour |
|---|---|
| Template status | Set to `entered-in-error` |
| Child Endpoint status | **NOT automatically changed** — child Endpoints retain their current status |
| Child Endpoint visibility | **Immediately hidden from consumers** — the visibility rule requires *both* Endpoint status = `active` AND Template status = `active`. If the Template is not active, all children are invisible regardless of their own status. **Exception:** the managing organisation (owner) can always see their Endpoints regardless of status — the visibility filter does not apply to owners. |
| Child Endpoint data | **Unchanged** — no fields on child Endpoints are modified |
| HealthcareService references | **Unchanged** — HealthcareServices still reference the child Endpoints, but those Endpoints are now invisible to consumers |

### The key point

Soft-deleting a Template **does not cascade a status change to child Endpoints**, but it
**effectively hides all children** from consumer search results because the visibility
rules require the parent Template to be active.

This is an **implicit cascade** (visibility effect) not an **explicit cascade** (data
change).

### Constraint: Cannot soft-delete a Template with active children

The API **blocks** soft-deletion of a Template if it has active child Endpoints:

```
PUT /Endpoint/{id}/$template  (status: entered-in-error)
    │
    └── API checks: does this Template have child Endpoints with status = active?
            │
            ├── Yes → 409 Conflict (blocked)
            └── No  → 200 OK (permitted)
```

**Error response:**

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "conflict",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "PROXY_CONFLICT"
          }
        ]
      },
      "diagnostics": "Cannot delete Template — one or more active Endpoints are derived from this Template."
    }
  ]
}
```

### Required sequence to soft-delete a Template

1. Soft-delete (or deactivate) all child Endpoints first:
   - `PUT /Endpoint/{id}` with `status: entered-in-error` for each child
2. Then soft-delete the Template:
   - `PUT /Endpoint/{id}/$template` with `status: entered-in-error`

This ensures there is no window where active child Endpoints exist under a deleted
Template — which would create an inconsistent state (children are technically active but
invisible).

---

## Cascade behaviour: Soft-deleting an Endpoint

### What happens

When an Endpoint is soft-deleted (`status` set to `entered-in-error`):

| Aspect | Behaviour |
|---|---|
| Endpoint status | Set to `entered-in-error` |
| Parent Template | **Unchanged** — the Template remains active and continues to serve other child Endpoints |
| HealthcareService references | **Unchanged** — the HealthcareService still holds a reference to this Endpoint in its `endpoint[]` array |
| Consumer visibility | **Immediately hidden** — the Endpoint no longer appears in consumer search results (status ≠ active) |
| Managing organisation visibility | **Still visible** — the managing organisation can still see the Endpoint (owner bypass rule) |

### Orphaned references

After soft-deleting an Endpoint, the HealthcareService that references it still holds
the reference in its `endpoint[]` array. This is intentional:

- The reference is not automatically removed because `PUT` on HealthcareService is a
  full replacement — the API would need to autonomously modify the HealthcareService,
  which violates the principle that resources are only changed by explicit caller action
- The orphaned reference is harmless — when consumers query
  `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}`, the soft-deleted Endpoint
  is excluded by the visibility filter. It simply doesn't appear.
- The managing organisation can clean up the reference at their convenience by issuing
  a `PUT /HealthcareService/{id}` without the deleted Endpoint in `endpoint[]`

### Recommendation

When soft-deleting an Endpoint, also update the HealthcareService to remove the reference:

1. `PUT /Endpoint/{id}` — set `status` to `entered-in-error`
2. `PUT /HealthcareService/{id}` — remove the Endpoint reference from `endpoint[]`

This keeps the data clean, but step 2 is not enforced by the API — it's an operational
best practice.

---

## Cascade behaviour: Soft-deleting a HealthcareService

### What happens

When a HealthcareService is soft-deleted (`active` set to `false`):

| Aspect | Behaviour |
|---|---|
| HealthcareService active | Set to `false` |
| Referenced Endpoints | **NOT changed** — Endpoint status and period are unaffected |
| Endpoint visibility | **Immediately hidden from consumers** — the visibility rule requires `HealthcareService.active = true`. If the service is inactive, all its Endpoints are invisible to consumer queries via that service. **Exception:** the managing organisation (owner) can always see their Endpoints regardless of the HealthcareService's active status. |
| Endpoint availability via other services | **Unaffected** — if the same Endpoint is referenced by another active HealthcareService, it remains visible through that other service |
| Template | **Unchanged** |
| List (if exists) | **Unchanged** — the List still references the HealthcareService, but queries via the inactive service return nothing |

### The key point

Soft-deleting a HealthcareService is a **service-level visibility switch**. It doesn't
touch the Endpoints or their Templates — it simply makes the service (and therefore its
Endpoints) invisible to consumers querying via that service.

This is important for the supplier switch scenario: a HealthcareService can be temporarily
deactivated during a transition without affecting the Endpoints themselves, which may
still be serving other HealthcareServices.

### No constraint on child state

Unlike Template soft-deletion, there is **no constraint** preventing deactivation of a
HealthcareService with active Endpoints. This is intentional — deactivating a service is
a common operational action (e.g. temporary closure, migration) that should not require
first deactivating all Endpoints.

---

## Summary: Cascade behaviour matrix

| Action | Child Endpoints | HealthcareService references | Consumer visibility | Constraint |
|---|---|---|---|---|
| **Soft-delete Template** | Status unchanged | Unchanged | All children hidden (Template not active) | ❌ Blocked if active children exist |
| **Soft-delete Endpoint** | N/A (is the child) | Orphaned reference remains | Endpoint hidden (status not active) | None |
| **Soft-delete HealthcareService** | Status unchanged | N/A (is the parent) | All Endpoints hidden via this service | None |

---

## Reinstatement

All soft deletes are reversible:

| Resource | Reinstatement | Effect |
|---|---|---|
| Template | Set `status` back to `active` | Children become visible again (if their own status is also active and period is valid) |
| Endpoint | Set `status` back to `active` | Endpoint becomes visible again (if Template is active and period is valid) |
| HealthcareService | Set `active` back to `true` | Endpoints become visible via this service again (if their own status/period/Template are valid) |

Reinstatement is immediate — the resource becomes visible on the next consumer query
after the change is persisted.

---

## Visibility rules recap

An Endpoint is visible to consumers only when **all** of the following are true:

1. `HealthcareService.active` = `true`
2. Endpoint `status` = `active`
3. Parent Template `status` = `active`
4. Current time is within Endpoint `period` (if set)

Soft-deleting any one of the first three breaks the chain and hides the Endpoint.

> **Owner exception:** The managing organisation (the owner of the Template/Endpoint)
> can **always** see their resources regardless of status, period, or HealthcareService
> active state. Visibility filtering only applies to non-owner consumers. This ensures
> owners can manage the lifecycle of their resources — reactivate, troubleshoot, and
> review — even when those resources are hidden from consumers.

---

## Hard delete vs soft delete — when to use which

| Scenario | Use soft delete | Use hard delete |
|---|---|---|
| Temporary withdrawal (maintenance, review) | ✅ | |
| Supplier product retired but may return | ✅ | |
| Service temporarily closed (seasonal, migration) | ✅ | |
| Error — resource created by mistake, may need to undo | ✅ | |
| Permanent decommission — resource will never be needed | | ✅ |
| Data cleanup — test data removal from production | | ✅ |
| Regulatory requirement to purge data | | ✅ |

> **Reminder:** Hard delete requires admin access. Soft delete can be performed by any
> authorised write operation.

---

## Operational sequence examples

### Example 1: Decommission a supplier product (Template + all children)

```
1. For each child Endpoint:
     PUT /Endpoint/{id} → status: entered-in-error
     PUT /HealthcareService/{id} → remove Endpoint from endpoint[]

2. Once all children are deactivated:
     PUT /Endpoint/{id}/$template → status: entered-in-error
```

### Example 2: Temporarily withdraw a service

```
1. PUT /HealthcareService/{id} → active: false

   (All Endpoints immediately hidden from consumers.
    Endpoints and Template are unchanged — ready for reinstatement.)
```

### Example 3: Temporarily suspend a single Endpoint

```
1. PUT /Endpoint/{id} → status: suspended

   (Endpoint hidden from consumers.
    HealthcareService and Template unchanged.
    Other Endpoints on the same service remain visible.)
```

### Example 4: Reinstate a withdrawn service

```
1. PUT /HealthcareService/{id} → active: true

   (All referenced Endpoints immediately visible again,
    provided their own status is active and period is valid.)
```

---

## Related documents

| Document | Description |
|----------|-------------|
| [Endpoint Visibility: Status and Period](./endpoint-visibility-status-and-period.md) | Full visibility rules, status meanings, and decision flowchart |
| [Managing Endpoint Templates](./manage-endpoint-template.md) | Template soft/hard delete procedures |
| [Managing Endpoints](./manage-endpoint.md) | Endpoint status transitions and deletion |
| [Managing HealthcareServices](./manage-healthcare-service.md) | HealthcareService deactivation and deletion |
