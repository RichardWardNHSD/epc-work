# BaRS OAS Alignment, Authentication, and API Routing

## Overview

This document covers three related topics:

1. Changes needed to the BaRS OAS specification to align it with the Endpoint Catalog API
2. The introduction of user-based authentication (CIS2) for EPC administrative operations
3. How API calls are routed — EPC operations direct to the EPC vs BaRS operations routed
   via the BaRS Proxy

---

## 1. BaRS OAS Alignment with the Endpoint Catalog API

### Completed — OAS Split

The BaRS OAS and EPC OAS have been **separated into two independent specifications**.
This was completed in June 2026.

**Before (single combined spec):** The BaRS OAS contained both messaging operations
and Endpoint Catalog operations in one file, leading to confusion over routing,
authentication, and ownership.

**After (two separate specs):**

| Specification | File | Covers | Auth model |
|---|---|---|---|
| **BaRS API** | `bars api OAS.json` | Messaging operations only (`/$process-message`, `/Appointment`, `/ServiceRequest`, `/DocumentReference`, `/Slot`, `/MessageDefinition`, `/metadata`) | Application-restricted (OAuth signed JWT) |
| **Endpoint Catalog API** | `endpoint-catalog-api.json` | Catalogue management (`/Endpoint`, `/HealthcareService`, `/List`, `/metadata`) | Application-restricted + CIS2 user-restricted |

### What was done

| # | Change | Status |
|---|--------|--------|
| 1 | Removed all EPC paths from BaRS OAS (`/Endpoint*`, `/HealthcareService*`, `/List*`) | ✅ Done |
| 2 | Removed EPC-only tags (`Endpoint`, `HealthcareService`, `List`) | ✅ Done |
| 3 | Removed EPC-only parameters (11 parameters) | ✅ Done |
| 4 | Removed EPC-only responses (14 responses) | ✅ Done |
| 5 | Removed EPC-only schemas (15 schemas) | ✅ Done |
| 6 | Removed EPC-only request bodies (4 request bodies) | ✅ Done |
| 7 | Removed `OAuth_Token` (legacy) and `OAuth_CIS2` from BaRS OAS — BaRS is app-restricted only | ✅ Done |
| 8 | Fixed server label swap (INT/PROD descriptions were reversed) | ✅ Done |
| 9 | Rewrote EPC `/metadata` to return the catalogue's own CapabilityStatement | ✅ Done |
| 10 | Removed `_include=HealthcareService:endpoint` from both specs | ✅ Done |
| 11 | Validated both specs — no broken `$ref` links, both are self-contained | ✅ Done |

### Final state — BaRS OAS

| Aspect | Value |
|---|---|
| Paths | 10 (messaging only) |
| Tags | 7 |
| Parameters | 21 |
| Responses | 2 (`4XX-BARS`, `5XX-BARS`) |
| Schemas | 18 |
| Request bodies | 5 |
| Security schemes | 1 (`OAuth_AppRestricted`) |
| Servers | 3 (Sandbox, Integration, Production — correctly labelled) |
| `$ref` links | 221 (all resolve) |

### Final state — EPC OAS

| Aspect | Value |
|---|---|
| Paths | 9 (catalogue + metadata) |
| Tags | 4 |
| Security schemes | 2 (`OAuth_AppRestricted`, `OAuth_CIS2`) |
| Servers | 3 (standalone path — target state) |
| `$ref` links | 200 (all resolve) |
| Self-contained | Yes — no external dependencies |

### Decision

The BaRS API and Endpoint Catalogue API are maintained as **two separate specifications**
with clear ownership boundaries:

- **BaRS team** owns the BaRS OAS — messaging operations only
- **EPC team** owns the EPC OAS — catalogue management operations

Consumers use two separate API specs. There is no duplication between them.

---

## 2. Introduction of User-Based Authentication (CIS2)

### Why user authentication is needed

The EPC supports two access modes:

| Access Mode | Authentication | Use Case |
|---|---|---|
| **Application-restricted** | Signed JWT → bearer token | System-to-system (BaRS Proxy lookups, DoS queries, supplier automation) |
| **User-restricted (CIS2)** | NHS CIS2 → bearer token | Human admin managing endpoints (DoS leads, suppliers, programme admins) |

Application-restricted access is sufficient for **read operations** and **automated supplier
writes**. But for administrative operations where a human is making a decision, CIS2
user-restricted access provides:

- Individual user identity in the audit trail (who made the change)
- Role-based access control (what operations they're allowed to perform)
- Organisation-level authorisation (which resources they can modify)

### What CIS2 provides in the token

| Claim | Purpose |
|-------|---------|
| User UUID | Unique user identifier — for audit |
| ODS code | User's organisation — for ownership checks |
| Role | User's role within the organisation — for RBAC |

### Which operations require CIS2

| Operation | App-restricted | CIS2 required |
|---|---|---|
| `GET /Endpoint` (consumer lookup) | ✅ | Optional |
| `GET /HealthcareService` | ✅ | Optional |
| `GET /List` | ✅ | Optional |
| `POST /Endpoint/$template` | ❌ | ✅ |
| `PUT /Endpoint/{id}/$template` | ❌ | ✅ |
| `POST /Endpoint` | ⚠️ Supplier automation | ✅ for manual |
| `PUT /Endpoint/{id}` | ⚠️ Supplier automation | ✅ for manual |
| `POST /HealthcareService` | ❌ | ✅ |
| `PUT /HealthcareService/{id}` | ❌ | ✅ |
| `DELETE` (any) | ❌ | ✅ (System Admin only) |
| `POST /List`, `PUT /List/{id}` | ❌ | ✅ |

### Impact on the OAS

The EPC OAS defines both security schemes (`OAuth_AppRestricted` and `OAuth_CIS2`).
The BaRS OAS has been updated to contain only `OAuth_AppRestricted` — CIS2 is not
relevant to BaRS messaging operations.

This separation is now complete. CIS2 authentication is an EPC concern only.

---


## 3. API Routing — EPC Direct vs BaRS Proxy

### Architecture

The BaRS Proxy and the EPC are **completely separate API products** on the NHS API
Platform. The BaRS API has no role in hosting, routing, or managing EPC operations —
it is purely a **consumer** of the EPC for endpoint resolution.

See the architecture diagram: [epc-architecture-routing.drawio](./epc-architecture-routing.drawio)

**Key architectural points:**

1. The **BaRS Proxy** and the **EPC Proxy** are separate Apigee proxies, each with their
   own authentication, routing, and policy configuration.
2. The BaRS Proxy consumes the EPC (via a GET call) to resolve target endpoints when
   routing messages. This is a client call — the BaRS Proxy is a consumer of the EPC,
   not an owner or host.
3. The EPC has its own **dedicated Apigee proxy** (EPC Proxy) that handles authentication
   for external callers (suppliers, admin tools).
4. The EPC backend (AWS API Gateway + Lambda + DynamoDB) is the same regardless of
   whether the caller is the BaRS Proxy or an external Endpoint Supplier.

### Actors

| Actor | Role | Connects to |
|---|---|---|
| **Sender** | Submits BaRS messages (bookings, referrals) | BaRS Proxy |
| **Receiver** | Receives BaRS messages from the Proxy | BaRS Proxy (outbound) |
| **Endpoint Supplier** | Manages endpoints, templates, healthcare services | EPC Proxy |
| **BaRS Proxy** | Routes messages to receivers; resolves endpoints | EPC Gateway (as a consumer) |

### Routing — Two Separate API Products

| API Product | Base Path | Apigee Proxy | Backend | Purpose |
|---|---|---|---|---|
| **BaRS API** | `/booking-and-referral/FHIR/R4` | BaRS Proxy | Receiver systems (via mTLS) | Runtime messaging |
| **Endpoint Catalog API** | `/endpoint-catalog/FHIR/R4` (target state) | EPC Proxy | AWS API Gateway → Lambda → DynamoDB | Catalogue management |

> **Current state:** Both APIs currently share the `/booking-and-referral/FHIR/R4` base
> path with Apigee routing internally based on path prefix. The target state is separate
> base paths once the EPC is promoted to a standalone API product.

### How the BaRS Proxy consumes the EPC

The BaRS Proxy acts as a **consumer** of the EPC — it makes a GET request to resolve the
target endpoint address before forwarding a message to the receiver.

```
1. Sender → BaRS Proxy:    POST /$process-message (with NHSD-Target-Identifier header)
2. BaRS Proxy → EPC:       GET /Endpoint?_has:HealthcareService:endpoint:identifier={system}|{value}
3. EPC → BaRS Proxy:       Returns matching active Endpoint(s)
4. BaRS Proxy extracts the `address` from the first active Endpoint
5. BaRS Proxy → Receiver:  Forwards the original message to that address (via mTLS)
```

**Authentication for this internal call:**

The BaRS Proxy authenticates to the EPC using **mTLS and/or an API key** — not via the
external OAuth flow. The Apigee keystore holds the client certificate used for this
connection. This is an internal platform-to-platform call; the external consumer (Sender)
never sees or participates in it.

### How Endpoint Suppliers access the EPC

Endpoint Suppliers (and admin tools) access the EPC through the **EPC Proxy** in Apigee,
which handles authentication:

| Operation type | Auth path | Token type |
|---|---|---|
| `GET` (reads, lookups) | App-restricted | Signed JWT → bearer token |
| `POST`, `PUT`, `DELETE` (writes) | User-restricted (CIS2) | CIS2 login → bearer token |

The EPC Proxy validates the token and injects trusted headers before forwarding to the
EPC backend:

**From app-restricted tokens:**
- `NHSD-Client-Id` — identifies the calling application
- `NHSD-Product-Id` — product ownership key
- `NHSD-Scope` — read/write permissions

**From CIS2 tokens (additional):**
- `NHSD-User-Id` — individual user identity (for audit)
- `NHSD-User-Role` — role assertion (for RBAC)

### ODS header vs token validation

The `NHSD-End-User-Organisation-ODS` header supplied by the consumer must match the ODS
code in the bearer token claims. This prevents spoofing.

**Open question:** Whether this validation is performed by Apigee (EPC Proxy layer) or by
the EPC backend (Lambda). The diagram marks this as TBD. The recommendation is for Apigee
to perform it since the token claims are already available at that layer.

---

### The EPC Backend (AWS)

The EPC backend is a serverless stack:

| Component | Role |
|---|---|
| **EPC Keystore** | Holds the server certificate for mTLS termination |
| **EPC Gateway** (AWS API Gateway) | Receives requests from Apigee (both BaRS Proxy and EPC Proxy paths), routes to Lambda |
| **Lambda functions** | Business logic — CRUD, visibility filtering, template resolution, RBAC, ownership checks |
| **DynamoDB** | Data store for Endpoints, Templates, HealthcareServices, Lists |

### What the BaRS Proxy no longer does

With this architecture, the BaRS Proxy has **no involvement** in EPC operations beyond
consuming the GET endpoint for resolution:

| Responsibility | BaRS Proxy | EPC |
|---|---|---|
| Hosting EPC paths | ❌ No | ✅ Own API product |
| Authenticating EPC consumers | ❌ No | ✅ Via EPC Proxy |
| Authorising EPC writes | ❌ No | ✅ Lambda RBAC |
| Storing endpoint data | ❌ No | ✅ DynamoDB |
| Resolving endpoints for message routing | ✅ Yes (as consumer) | ✅ Serves the response |

### S3 routing configuration — deprecated

The previous design included an S3-based routing configuration database used by the BaRS
Proxy. This is now **deprecated** (marked with ✗ in the diagram). The BaRS Proxy resolves
endpoints by calling the EPC API directly, not by reading from a local configuration store.

---

### Key distinction

- **BaRS Proxy (inside Apigee)** — handles runtime messaging traffic. Receives BaRS
  messages, resolves the target endpoint via a GET call to the EPC, forwards to the
  backend receiver. The BaRS Proxy is a **consumer** of the EPC, nothing more.

- **EPC Proxy (inside Apigee)** — handles authentication and header injection for
  external EPC consumers (suppliers, admin tools). Routes to the EPC backend.

- **EPC Backend (AWS)** — handles catalog management. CRUD on endpoints, templates,
  healthcare services, and ordering lists. Serves both internal consumers (BaRS Proxy)
  and external consumers (via EPC Proxy).

---

## Summary

| Topic | Decision | Status |
|-------|----------|--------|
| BaRS OAS alignment | All EPC operations removed from BaRS OAS. EPC OAS (`endpoint-catalog-api.json`) is the authoritative spec for catalogue operations. | ✅ Complete |
| User authentication | CIS2 user-restricted access required for all EPC write operations. App-restricted sufficient for reads and automated supplier writes. BaRS OAS contains only app-restricted. | ✅ Complete (EPC OAS); CIS2 implementation pending |
| Routing | BaRS Proxy runs inside Apigee; EPC paths route to AWS backend. Proxy calls EPC internally for endpoint resolution. | ✅ Documented |
| Token validation | Apigee validates token and extracts claims; backend enforces ODS match, RBAC, and ownership. | ✅ Documented |
| `/metadata` | BaRS `/metadata` forwards to receiver. EPC `/metadata` returns its own CapabilityStatement. | ✅ Complete |

---

## Open Questions

| # | Question | For | Status |
|---|----------|-----|--------|
| 1 | ~~Should the BaRS OAS retain `GET /Endpoint` (read-only consumer lookup) or should all EPC operations be removed?~~ | Architecture | ✅ **Resolved** — All EPC operations removed. BaRS OAS is messaging-only. |
| 2 | What Apigee header names are used to forward token claims to the backend? (e.g. `NHSD-Client-Id`, `NHSD-User-Id`) | Platform team | Open |
| 3 | ~~Should `/metadata` return a combined capability statement or separate ones per service?~~ | Architecture | ✅ **Resolved** — Two separate `/metadata` endpoints on two separate APIs. |
| 4 | How does the BaRS Proxy's internal EPC call authenticate? Service account credentials or internal Apigee policy? | Platform team | Open |
| 5 | ~~Are the `X-Request-Id` and `X-Correlation-Id` headers sufficient, or do the EPC-specific definitions need to be added separately?~~ | Architecture | ✅ **Resolved** — Existing headers are sufficient. EPC OAS defines its own copies. |
