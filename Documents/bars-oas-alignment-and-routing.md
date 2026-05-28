# BaRS OAS Alignment, Authentication, and API Routing

## Overview

This document covers three related topics:

1. Changes needed to the BaRS OAS specification to align it with the Endpoint Catalog API
2. The introduction of user-based authentication (CIS2) for EPC administrative operations
3. How API calls are routed — EPC operations direct to the EPC vs BaRS operations routed
   via the BaRS Proxy

---

## 1. BaRS OAS Alignment with the Endpoint Catalog API

### Current state

The BaRS OAS currently includes both BaRS messaging operations AND
Endpoint Catalog operations in a single specification:

**BaRS messaging operations (routed via BaRS Proxy):**

| Path | Methods | Purpose |
|------|---------|---------|
| `/$process-message` | POST | Send a BaRS message |
| `/Appointment` | GET, POST | Manage appointments |
| `/Appointment/{id}` | GET, PUT, PATCH, DELETE | Manage a specific appointment |
| `/DocumentReference` | GET, POST | Manage documents |
| `/DocumentReference/{id}` | GET, PUT, DELETE | Manage a specific document |
| `/ServiceRequest` | GET, POST | Manage service requests |
| `/ServiceRequest/{id}` | GET, PUT, PATCH, DELETE | Manage a specific service request |
| `/Slot` | GET | Query available slots |
| `/MessageDefinition` | GET | Get message definitions |
| `/metadata` | GET | Capability statement |

**Endpoint Catalog operations (routed direct to EPC):**

| Path | Methods | Purpose |
|------|---------|---------|
| `/Endpoint` | GET, POST | Search/create endpoints |
| `/Endpoint/$template` | GET, POST | Search/create templates |
| `/Endpoint/{id}` | GET, PUT, DELETE | Manage a specific endpoint |
| `/Endpoint/{id}/$template` | PUT, DELETE | Manage a specific template |
| `/HealthcareService` | GET, POST | Search/create healthcare services |
| `/HealthcareService/{id}` | GET, PUT, DELETE | Manage a specific healthcare service |

### Changes needed to align with the EPC OAS

The BaRS OAS needs the following updates to align with the standalone EPC OAS
(`endpoint-catalog-api.json`):

#### Missing paths

| Path | Methods | Purpose |
|------|---------|---------|
| `/List` | POST, GET | Create and search endpoint priority ordering lists |
| `/List/{id}` | GET, PUT, DELETE | Read, update, and delete a specific ordering list |

These are entirely absent from the BaRS OAS and must be added.

#### Security scheme changes

| Aspect | BaRS OAS (current) | EPC OAS (target) | Action |
|--------|-------------------|------------------|--------|
| Schemes defined | `OAuth_Token` (single) | `OAuth_AppRestricted` + `OAuth_CIS2` (dual) | Replace single scheme with two schemes |
| Global security | `[{"OAuth_Token": []}]` | `[{"OAuth_AppRestricted": []}, {"OAuth_CIS2": []}]` | Update to show both modes |

#### Missing parameters (component-level)

These parameters exist in the EPC OAS but not in the BaRS OAS:

| Parameter | Type | Purpose |
|-----------|------|---------|
| `NHSD-End-User-Organisation-ODS` | Header (required) | ODS code of the requesting organisation — used for authorisation |
| `RequestingOrganisationODS_HParam` | Header | ODS code parameter (component reference) |
| `ServiceId_QParam` | Query | Service ID search parameter for HealthcareService |

> **Note:** `X-Request-Id` and `X-Correlation-Id` are already defined in the BaRS OAS as
> `RequestId_HParam` and `CorrelationId_HParam`. These do not need to be added — but the
> EPC operations in the BaRS OAS need to reference them (they currently don't).


#### Server configuration

| Aspect | BaRS OAS (current) | EPC OAS (target) | Action |
|--------|-------------------|------------------|--------|
| Sandbox | `https://sandbox.api.service.nhs.uk/booking-and-referral/FHIR/R4` | `https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4` | Different base path — EPC has its own |
| Integration | Labelled "Production" (wrong) | `https://int.api.service.nhs.uk/endpoint-catalog/FHIR/R4` | Fix label + different base path |
| Production | Labelled "Integration Server" (wrong) | `https://api.service.nhs.uk/endpoint-catalog/FHIR/R4` | Fix label + different base path |

Note: The BaRS OAS uses `/booking-and-referral/FHIR/R4` as the base path. The EPC OAS
uses `/endpoint-catalog/FHIR/R4`. These are different API products on the NHS England API
Platform with different base URLs. The BaRS OAS server entries for EPC operations should
either point to the EPC base URL or be removed (if EPC operations are separated out per
Option C).

#### Summary of all changes required

| # | Change | Impact |
|---|--------|--------|
| 1 | Add `/List` and `/List/{id}` paths with POST, GET, PUT, DELETE | New paths |
| 2 | Replace `OAuth_Token` with `OAuth_AppRestricted` + `OAuth_CIS2` | Security scheme |
| 3 | Add `NHSD-End-User-Organisation-ODS` parameter definition | New component |
| 4 | Add `RequestingOrganisationODS_HParam` parameter definition | New component |
| 5 | Add `ServiceId_QParam` parameter definition | New component |
| 6 | Add `NHSD-End-User-Organisation-ODS` to every EPC operation (15 operations) | Per-operation params |
| 7 | Add `RequestId_HParam` and `CorrelationId_HParam` refs to every EPC operation (already defined, just not referenced) | Per-operation params |
| 8 | Add `ConnectionType_QParam` and `PayloadType_QParam` to `GET /Endpoint` | Query params |
| 9 | Add `ServiceId_QParam` and `_include` to `GET /HealthcareService` | Query params |
| 10 | Fix server description labels (Production/Integration are swapped) | Server config |
| 11 | Consider separate server entries for EPC base URL vs BaRS base URL | Server config |

### Recommendation

Rather than maintaining EPC operations in both the BaRS OAS and the standalone EPC OAS,
the BaRS OAS should **reference** the EPC operations or the two specs should be clearly
separated:

- **Option A:** Remove EPC paths from the BaRS OAS entirely. The BaRS OAS covers only
  messaging operations. The EPC OAS is the authoritative spec for catalog operations.
  Consumers use two separate API specs.

- **Option B:** Keep EPC paths in the BaRS OAS but ensure they are identical to the
  standalone EPC OAS. Any change to the EPC OAS must be reflected in the BaRS OAS.
  Risk of drift.

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

The EPC OAS already defines both security schemes (`OAuth_AppRestricted` and `OAuth_CIS2`).
The BaRS OAS needs to be updated to reflect that EPC write operations require CIS2 — or
those operations should be removed from the BaRS OAS entirely (per Option C above).

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
| `/Endpoint*` | EPC backend (AWS) | App-restricted or CIS2 | Manage endpoints/templates |
| `/HealthcareService*` | EPC backend (AWS) | App-restricted or CIS2 | Manage healthcare services |
| `/List*` | EPC backend (AWS) | App-restricted or CIS2 | Manage endpoint ordering |
| `/metadata` | **⚠️ See below** | App-restricted | Capability statement |

### `/metadata` routing — open design question

The `/metadata` endpoint returns a FHIR `CapabilityStatement` describing what the API
supports. In the current architecture this is problematic because:

- The **BaRS Proxy** has its own capabilities (messaging, appointments, slots, documents)
- The **EPC backend** has its own capabilities (endpoints, templates, healthcare services, lists)
- A single `/metadata` response needs to describe **both** — consumers expect one
  CapabilityStatement from one base URL

With split routing (BaRS paths → Apigee, EPC paths → AWS), neither backend alone can
produce a complete CapabilityStatement. Three options:

| Option | How it works | Pros | Cons |
|--------|-------------|------|------|
| **A — API layer (Apigee) handles `/metadata`** | Apigee intercepts `/metadata` and returns a pre-configured or dynamically assembled CapabilityStatement covering both BaRS and EPC capabilities | Single source of truth; consumer sees one complete response; no routing ambiguity | Apigee must maintain the CapabilityStatement content (or assemble it from both backends); adds logic to the API layer |
| **B — EPC backend handles `/metadata`** | Route to EPC; EPC returns a combined statement | Simple routing rule | EPC would need to know about BaRS capabilities it doesn't own; tight coupling |
| **C — Separate CapabilityStatements per service** | BaRS Proxy returns its own at `/booking-and-referral/FHIR/R4/metadata`; EPC returns its own at `/endpoint-catalog/FHIR/R4/metadata` | Clean separation; each service owns its own capabilities | Consumers must call two endpoints; breaks the "single API" illusion |

**Recommendation:** Option A (API layer handles it) is the cleanest approach if the two
services share a single base URL. Apigee already acts as the routing layer — it can also
serve the combined CapabilityStatement. If the services are separated to different base
URLs (per the server URL decision), Option C becomes natural and no special handling is
needed.

This is recorded as open question #3 below and needs an architecture decision.

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

| Topic | Decision |
|-------|----------|
| BaRS OAS alignment | Remove EPC write operations from BaRS OAS; keep only `GET /Endpoint` for consumer lookup. EPC OAS is authoritative for catalog operations. |
| User authentication | CIS2 user-restricted access required for all EPC write operations. App-restricted sufficient for reads and automated supplier writes. |
| Routing | BaRS Proxy runs inside Apigee; EPC paths route to AWS backend. Proxy calls EPC internally for endpoint resolution. |
| Token validation | Apigee validates token and extracts claims; backend enforces ODS match, RBAC, and ownership. |

---

## Open Questions

| # | Question | For |
|---|----------|-----|
| 1 | Should the BaRS OAS retain `GET /Endpoint` (read-only consumer lookup) or should all EPC operations be removed? | Architecture |
| 2 | What Apigee header names are used to forward token claims to the backend? (e.g. `NHSD-Client-Id`, `NHSD-User-Id`) | Platform team |
| 3 | Should `/metadata` return a combined capability statement or separate ones per service? Depends on whether BaRS and EPC share a base URL. See [/metadata routing](#metadata-routing--open-design-question) for options. | Architecture |
| 4 | How does the BaRS Proxy's internal EPC call authenticate? Service account credentials or internal Apigee policy? | Platform team |
| 5 | Are the `X-Request-Id` and `X-Correlation-Id` headers (already defined as `RequestId_HParam` and `CorrelationId_HParam` in the BaRS OAS) sufficient, or do the EPC-specific definitions need to be added separately? | Architecture |
