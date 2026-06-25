# EPC MVP Deferral — Private Endpoint Address Redaction

**Series:** EPC MVP Scope Deferrals
**Document:** 5 of N

---

## Purpose

This document analyses the deferral of the `header: private` address redaction feature
from the EPC MVP. The feature remains in scope for the final solution — the MVP treats
all Endpoint addresses as publicly visible to any authenticated consumer.

---

## What is private endpoint address redaction

The `header` field on an Endpoint (or its parent Template) can be set to either `public`
or `private`. When set to `private`, the Endpoint's `address` field is **omitted from
API responses** unless the requesting organisation matches the `managingOrganization` on
the Endpoint's parent Template (i.e. the requester is the owner).

This allows suppliers to register Endpoints that are discoverable (consumers know the
Endpoint exists) but not directly addressable (consumers cannot see the URL unless they
are the owning organisation).

---

## What is being deferred

| Deferred capability | Detail |
|---|---|
| `header: private` field semantics | The field exists on the resource but has no effect in the MVP — all addresses are returned regardless of header value |
| Ownership check on GET responses | No per-Endpoint ownership lookup on read operations to determine whether to include or omit the address |
| Address omission logic | No conditional logic stripping the `address` field from responses for non-owner consumers |
| Private Endpoint consumer experience | Consumers do not see the "Endpoint exists but address is hidden" pattern in MVP |

---

## What is retained for MVP

| Retained capability | Detail |
|---|---|
| `header` field on Template/Endpoint | The field is stored and returned as-is — it's just not enforced. Suppliers can still set `header: private` in preparation for enforcement later. |
| All Endpoint addresses visible | Every authenticated consumer sees the full `address` on every Endpoint, regardless of ownership. |
| ODS ownership enforcement on writes | Write operations still require ownership. Only read-path redaction is deferred. |

---

## Why this can be deferred

### Feature is already under discussion

The authorisation document explicitly notes:

> "The `header: private` requirement is currently under discussion and may be struck from
> the specification."

Deferring a feature whose requirement is not yet confirmed avoids building logic that may
never be needed.

### All MVP endpoints are public

BaRS pharmacy Endpoints are public — senders need the address to route referrals. There
is no known MVP use case where a supplier would register an Endpoint and want its address
hidden from other authenticated consumers.

### Adds conditional logic to every read path

Address redaction requires:
1. For each Endpoint in a response, resolve the parent Template
2. Check `header` value
3. If `private`, compare `managingOrganization` ODS with the requester's ODS
4. If no match, strip the `address` field from the response

This check runs on every Endpoint in every GET response (search results can contain many
Endpoints). Deferring it simplifies the read path and avoids a per-Endpoint ownership
lookup that adds latency.

### No security risk from deferral

Endpoint addresses are not secrets — they are the URLs that senders connect to. Making
them visible to all authenticated consumers does not create a security vulnerability.
The `private` feature is a business preference (some suppliers don't want competitors
seeing their URLs), not a security control.

---

## Implications of deferral

#### 1. All Endpoint addresses are visible to all authenticated consumers

**Impact: Low**

Any authenticated application can see the `address` of any Endpoint, including those
marked `header: private`.

| With redaction | Without redaction (MVP) |
|---|---|
| Private Endpoints returned without `address` to non-owners | All Endpoints returned with full `address` to all consumers |

**Mitigation:** No MVP supplier has requested private endpoints. If a supplier registers
an Endpoint with `header: private` during MVP, they should be informed that enforcement
is not yet active. The field value is stored — enforcement can be enabled without data
migration.

---

#### 2. No "discoverable but not addressable" pattern

**Impact: None (for MVP)**

The pattern where a consumer sees an Endpoint exists (it appears in search results) but
cannot see its URL (address is omitted) is not available. Consumers either see the full
Endpoint or don't see it at all.

**Mitigation:** Not needed for MVP. BaRS senders need the address — hiding it would
break routing.

---

## API changes for MVP

#### Simplified for MVP

| Item | Detail |
|---|---|
| **GET response assembly** | No per-Endpoint ownership check. All fields returned unconditionally. Simpler, faster. |
| **Template resolution** | No need to resolve Template ownership on read operations (only needed for write auth). |
| **Response payload** | `address` always present. No conditional field omission. |

#### Unchanged between MVP and final solution

| Aspect | Detail |
|---|---|
| **`header` field on resources** | Stored and returned as-is. Suppliers can set it to `private` — it just has no effect yet. |
| **Write authorisation** | ODS ownership still enforced on all writes. Unaffected by this deferral. |
| **ODS spoofing protection** | Still enforced. Unrelated to read-path redaction. |
| **Resource schemas** | No schema change. `header` remains a valid field with values `public` or `private`. |

#### What changes when delivered

| Item | MVP | Final |
|---|---|---|
| **GET response for private Endpoints (non-owner)** | Full Endpoint with `address` | Endpoint returned without `address` |
| **GET response for private Endpoints (owner)** | Full Endpoint with `address` | Full Endpoint with `address` (same) |
| **Per-Endpoint ownership check on reads** | Not performed | Performed for Endpoints with `header: private` |
| **Response assembly latency** | No additional lookup | Marginal increase (Template ownership resolution per private Endpoint) |

---

## Decision

| Decision | Rationale |
|---|---|
| **Defer private endpoint address redaction from MVP** | The requirement is under discussion and may be struck. All MVP Endpoints are public (BaRS). The feature adds per-Endpoint conditional logic to every read path. No security risk from deferral — addresses are not secrets. |

---

## When to deliver

Private endpoint address redaction should be delivered when:

1. **The requirement is confirmed** — the specification review concludes that `header: private`
   is retained (not struck)
2. **A supplier requests it** — a supplier onboards with a genuine need to hide their
   Endpoint address from other consumers
3. **Competitive concerns arise** — multiple suppliers serving the same services want to
   prevent competitors seeing their infrastructure URLs

---

## Related documents

| Document | Description |
|----------|-------------|
| [Authentication and Authorisation](../authorisation.md) | Full ownership model including private endpoint redaction (target state) |
| [Endpoint Header](../endpoint-header.md) | Detailed design of the `header` field behaviour |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [mvp-deferral-rbac.md](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | [mvp-deferral-observability.md](./mvp-deferral-observability.md) |
| 3 | Endpoint Ordering (List) | [mvp-deferral-endpoint-ordering.md](./mvp-deferral-endpoint-ordering.md) |
| 4 | Disaster Recovery (Full DR Plan) | [mvp-deferral-disaster-recovery.md](./mvp-deferral-disaster-recovery.md) |
| 5 | Private Endpoint Address Redaction | This document |
