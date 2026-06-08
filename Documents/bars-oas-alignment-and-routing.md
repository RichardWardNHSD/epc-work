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

### Outcome — Option A adopted

The recommendation from this document was **Option A**: remove EPC paths from the BaRS
OAS entirely. The EPC OAS is the authoritative specification for catalogue operations.
Consumers use two separate API specs.

This eliminates the drift risk from Option B (maintaining both) and makes ownership clear:
- BaRS team owns the BaRS OAS (messaging)
- EPC team owns the EPC OAS (catalogue)

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

The BaRS Proxy is hosted **inside Apigee** (as an Apigee proxy policy/target), not as a
separate AWS-hosted service. The Endpoint Catalog API is a separate backend service
(AWS API Gateway + Lambda + DynamoDB). Apigee routes requests to one or the other based
on the path.

**Simplified view:**

- Consumer sends request to Apigee
- Apigee validates the token and extracts claims
- If the path is a BaRS messaging operation → BaRS Proxy (inside Apigee) handles it
- If the path is an EPC catalog operation → Apigee routes to the EPC backend (AWS)

The BaRS Proxy resolves target endpoints by calling the EPC API internally (within Apigee)
before forwarding messages to backend receivers.

---

### What Apigee does vs what the EPC backend does

#### Apigee layer (API Platform)

| Responsibility | Detail |
|---|---|
| **Token validation** | Validates the bearer token (signature, expiry, audience). Rejects invalid/expired tokens with 401. |
| **Token claim extraction** | Extracts claims from the token and passes them to the backend as verified headers or context variables |
| **Routing** | Routes to BaRS Proxy (internal Apigee policy) or EPC backend (external AWS) based on path |
| **Rate limiting** | Applies throttling policies per application |
| **Audit logging** | Logs all requests to Splunk (gateway-level audit) |
| **BaRS Proxy execution** | For messaging paths, executes the proxy logic (endpoint resolution + forwarding) within Apigee |

#### EPC Backend (AWS Lambda)

| Responsibility | Detail |
|---|---|
| **ODS validation** | Cross-checks `NHSD-End-User-Organisation-ODS` header against the ODS claim forwarded by Apigee |
| **RBAC enforcement** | Checks the user's role (from forwarded token claims) permits the requested operation |
| **Ownership check** | Verifies the caller's ODS/Product ID matches the resource's owner |
| **Business logic** | CRUD operations on DynamoDB |
| **Audit record creation** | Writes application-level audit records (who changed what) |

---

### Token claims: what Apigee extracts and forwards to the backend

When Apigee validates the token, it extracts claims and forwards them to the EPC backend
as trusted headers. The backend trusts these because they come from Apigee (internal),
not from the external caller.

#### Application-restricted token claims

| Claim in token | Forwarded to backend as | Used by backend for |
|---|---|---|
| `client_id` | `NHSD-Client-Id` header | Identifying the calling application |
| `product_id` | `NHSD-Product-Id` header | Product ID ownership check on Templates/Endpoints |
| `ods_code` | Validated against `NHSD-End-User-Organisation-ODS` header | ODS ownership check |
| `scope` | `NHSD-Scope` header | Determining read/write permissions |

#### User-restricted (CIS2) token claims — additional

All of the above, plus:

| Claim in token | Forwarded to backend as | Used by backend for |
|---|---|---|
| `sub` (user UUID) | `NHSD-User-Id` header | Audit trail — who made the change |
| `selected_roleid` | `NHSD-User-Role` header | RBAC enforcement — what operations are permitted |
| `organisation.ods_code` | Validated against `NHSD-End-User-Organisation-ODS` | ODS ownership check + audit |

---

### The validation split: Apigee vs Backend

| Check | Performed by | Failure response |
|---|---|---|
| Token signature valid? | Apigee | 401 Unauthorized |
| Token expired? | Apigee | 401 Unauthorized |
| Token audience correct? | Apigee | 401 Unauthorized |
| Application registered and not revoked? | Apigee | 401 Unauthorized |
| Rate limit exceeded? | Apigee | 429 Too Many Requests |
| ODS header matches ODS in token claims? | EPC Backend | 403 Forbidden (spoofing) |
| User role permits this operation? (RBAC) | EPC Backend | 403 Forbidden (role) |
| Caller ODS/Product ID matches resource owner? | EPC Backend | 403 Forbidden (ownership) |
| Resource status allows this operation? | EPC Backend | 403 or 404 |

---

### Routing rules

| Path prefix | Handled by | Authentication | Purpose |
|---|---|---|---|
| `/$process-message` | BaRS Proxy (inside Apigee) | App-restricted | Send BaRS messages |
| `/Appointment*` | BaRS Proxy (inside Apigee) | App-restricted | Manage appointments |
| `/ServiceRequest*` | BaRS Proxy (inside Apigee) | App-restricted | Manage service requests |
| `/DocumentReference*` | BaRS Proxy (inside Apigee) | App-restricted | Manage documents |
| `/Slot` | BaRS Proxy (inside Apigee) | App-restricted | Query slots |
| `/MessageDefinition` | BaRS Proxy (inside Apigee) | App-restricted | Get message definitions |
| `/metadata` | BaRS Proxy (inside Apigee) | App-restricted | Capability statement (forwarded to receiving system) |
| `/Endpoint*` | EPC backend (AWS) | App-restricted or CIS2 | Manage endpoints/templates |
| `/HealthcareService*` | EPC backend (AWS) | App-restricted or CIS2 | Manage healthcare services |
| `/List*` | EPC backend (AWS) | App-restricted or CIS2 | Manage endpoint ordering |

### `/metadata` routing — resolved

There are now **two separate `/metadata` endpoints** on two separate API products:

| API | `/metadata` behaviour |
|---|---|
| **BaRS API** (`/booking-and-referral/FHIR/R4/metadata`) | BaRS Proxy forwards the request to the **receiving system** (identified by `NHSD-Target-Identifier` header) and returns that system's CapabilityStatement. |
| **EPC API** (`/endpoint-catalog/FHIR/R4/metadata`) | Returns the **EPC's own CapabilityStatement** describing the catalogue's supported resources, interactions, and search parameters. No `NHSD-Target-Identifier` needed. |

This is a clean separation — BaRS `/metadata` is about the receiver, EPC `/metadata` is
about the catalogue itself.

---

### The BaRS Proxy's internal EPC call

When the BaRS Proxy (inside Apigee) needs to resolve a target endpoint for routing:

1. BaRS Proxy receives `POST /$process-message` with `NHSD-Target-Identifier` header
2. Proxy decodes the target identifier to get the HealthcareService identifier
3. Proxy calls `GET /Endpoint?HealthcareService.identifier={system}|{value}` on the EPC
   (internal Apigee-to-AWS call using service credentials)
4. EPC returns the matching active Endpoint(s)
5. Proxy extracts the `address` from the first active Endpoint
6. Proxy forwards the original message to that backend address

This internal call uses Apigee's service-to-service credentials — it does not go through
the external token validation flow again. The external consumer never sees this EPC lookup.

---

### Key distinction

- **BaRS Proxy (inside Apigee)** — handles runtime messaging traffic. Receives BaRS
  messages, resolves the target endpoint via an internal EPC call, forwards to the
  backend receiver. Consumer-facing for messaging operations.

- **EPC API (AWS backend)** — handles catalog management. CRUD on endpoints, templates,
  healthcare services, and ordering lists. Called directly by admin tools, supplier
  systems, and DoS. Also called internally by the BaRS Proxy for endpoint resolution.

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
