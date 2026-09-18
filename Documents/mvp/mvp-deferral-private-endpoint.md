# EPC MVP — Private Endpoint Address Redaction (Withdrawn)

**Series:** EPC MVP Scope Deferrals
**Document:** 5 of N

> ## ⚠️ Status: REMOVED FROM SCOPE — not deferred
>
> Private Endpoint Address Redaction has been **withdrawn entirely**, not deferred to a
> later iteration. The feature was built on a **non-conformant misuse of the FHIR R4
> `Endpoint.header` element** and there is no MVP or known future use case that justifies
> it. This document is retained as the withdrawal record; it no longer describes a
> deferred capability.
>
> If an address-visibility capability is genuinely required in future, it must be
> **re-specified from scratch using a conformant mechanism** (a FHIR extension or a
> dedicated element) — not by reviving the `header: public/private` approach described
> historically below.

---

## Why it was removed

The feature controlled `address` visibility by setting `Endpoint.header` (or its parent
Template's `header`) to `public` or `private`, redacting `address` from non-owning callers
when `private`. That usage is not valid FHIR R4:

| Aspect | FHIR R4 `Endpoint.header` | How the withdrawn feature used it |
|--------|---------------------------|-----------------------------------|
| Type | `string` | `string` |
| Cardinality | `0..*` (a list) | Single value |
| Purpose | Connection headers to **send** when contacting the endpoint's `address` (e.g. `Authorization`, `Content-Type`) | A privacy/visibility flag for the `address` field |
| Valid values | Any header string, e.g. `"Authorization: Bearer <token>"` | `"public"` / `"private"` |

Repurposing `header` as a visibility flag means a standard FHIR client would read
`public` / `private` as a literal header to transmit on the wire — which is meaningless.
Address visibility is an authorisation concern and does not belong on this element. See
[Endpoint Header](../endpoint-header.md) for the full analysis.

**Additional reasons the removal is safe:**

- **All MVP endpoints are public.** BaRS pharmacy Endpoints must expose their `address` so
  senders can route referrals. There is no MVP use case for hiding an address.
- **Addresses are not secrets.** They are the URLs senders connect to. Making them visible
  to all authenticated consumers is not a security vulnerability — the `private` idea was a
  business preference, not a security control.
- **The requirement was already unconfirmed.** It was flagged as "under discussion and may
  be struck from the specification" — it has now been struck.

---

## What this means for the EPC

| Area | Behaviour |
|------|-----------|
| `address` visibility | Every authenticated consumer sees the full `address` on every Endpoint and Template. No redaction. |
| `header` field | Retains its correct FHIR R4 meaning — a `0..*` array of connection-header strings to send when contacting the address. The EPC OAS already defines it this way. |
| Read path (`GET`) | No per-Endpoint ownership lookup and no conditional field omission. Simpler and faster. |
| Write authorisation | Unaffected. ODS ownership and Product ID ownership are still enforced on all writes. |
| ODS spoofing protection | Unaffected. Still enforced on all calls. |

---

## Requirement reference

| Requirement | Description | Disposition |
|---|---|---|
| EPCSe001 AC4 | "Set the status to private" — private Endpoint address visibility control | **Withdrawn** — non-conformant use of `Endpoint.header`. To be re-specified conformantly only if a genuine need arises. |

---

## Decision

| Decision | Rationale |
|---|---|
| **Remove private endpoint address redaction from the EPC entirely** | Built on a non-conformant misuse of the FHIR R4 `Endpoint.header` element. No MVP use case (all BaRS endpoints are public). Addresses are not secrets, so no security value. Requirement was unconfirmed and has been struck. |
| **Do not revive `header: public/private`** | If address visibility is ever required, model it with a FHIR extension or a dedicated element — never by overloading `header`. |

---

## Related documents

| Document | Description |
|----------|-------------|
| [EPC MVP — Scope and Architecture](./README.md) | MVP scope; records this removal |
| [Endpoint Header](../endpoint-header.md) | Analysis of the `header` misuse and the conformant FHIR R4 meaning |
| [Authentication and Authorisation](../authorisation.md) | Ownership model for write operations |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [mvp-deferral-rbac.md](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | [mvp-deferral-observability.md](./mvp-deferral-observability.md) |
| 3 | Endpoint Ordering (List) | [mvp-deferral-endpoint-ordering.md](./mvp-deferral-endpoint-ordering.md) |
| 4 | Disaster Recovery (Full DR Plan) | [mvp-deferral-disaster-recovery.md](./mvp-deferral-disaster-recovery.md) |
| 5 | Private Endpoint Address Redaction | This document — **withdrawn, not deferred** |
