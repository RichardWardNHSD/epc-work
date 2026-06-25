# EPC MVP Deferral — Role-Based Access Control (RBAC)

**Series:** EPC MVP Scope Deferrals
**Document:** 1 of N

---

## Purpose

This document analyses the deferral of Role-Based Access Control (RBAC) from the EPC MVP.
RBAC remains in scope for the final solution — this is about delivery sequencing, not
permanent removal.

---

## MVP context

The EPC MVP must deliver:

- Consumer lookup of Endpoints by HealthcareService (GET operations)
- Template-based Endpoint management (create, update, soft delete)
- HealthcareService management (create, update, associate Endpoints)
- Supplier switch workflow (re-association of Endpoints)
- Application-restricted authentication (signed JWT → bearer token)
- ODS-based ownership enforcement on write operations
- Basic audit trail (who changed what, when)

RBAC is not required to deliver any of the above.

---

## Deferral: Role-Based Access Control (RBAC)

### What is being deferred

The full CIS2 user-restricted RBAC model as defined in the authorisation document:

| Removed capability | Detail |
|---|---|
| CIS2 user-restricted authentication | NHS Care Identity Service login (smartcard, Windows Hello, NHS Identity app) for admin operations |
| Named roles | Consumer, Endpoint Admin, Healthcare Service Admin, Activation Admin, Programme Admin, System Admin |
| Role-to-operation enforcement | Checking the user's CIS2 role before allowing specific operations (e.g. only Endpoint Admin can create Templates) |
| Per-user audit identity | Audit trail attributing changes to a specific named individual |
| Admin UI gated by CIS2 | Any user interface that requires CIS2 login to access |
| Role assignment infrastructure | Integration with CIS2 role catalogue for EPC-specific roles |

### What is retained

| Retained capability | Detail |
|---|---|
| Application-restricted authentication | Signed JWT → bearer token via NHS England API Platform. All API calls are authenticated. |
| ODS ownership enforcement | `NHSD-End-User-Organisation-ODS` header validated against bearer token claims. Organisation cannot modify another organisation's resources. |
| Product ID ownership check | Token Product ID matched against resource Product ID. One supplier cannot modify another supplier's Templates/Endpoints. |
| ODS spoofing protection | Token claim cross-check ensures the ODS header cannot be forged. |
| Application-level audit | Audit records attribute changes to the registered application (client_id) and ODS code — but not to an individual user. |

### Implications of deferral

#### 1. No individual accountability on write operations

**Impact: Medium**

Without CIS2, the audit trail captures *which application* made the change and *which
organisation* it belongs to, but not *which person* within that organisation initiated it.

| With RBAC | Without RBAC (MVP) |
|---|---|
| "User john.smith@sonar.nhs.uk (Endpoint Admin) updated Template X at 10:05" | "Application sonar-bars-tool (client_id: abc-123, ODS: SONAR1) updated Template X at 10:05" |

**Mitigation:** For the MVP, this is acceptable because:
- The R&M team processes most changes via a batch pipeline anyway (no individual user)
- Supplier automation is inherently application-level, not user-level
- If individual accountability is needed, suppliers can implement their own internal
  access controls and audit on top of their registered application

**Risk accepted:** Cannot identify which individual within an organisation made a specific
change. Only the application and organisation are recorded.

---

#### 2. No operation-level access restriction within an organisation

**Impact: Medium**

Without roles, any authenticated application belonging to an organisation can perform
*any* write operation on resources that organisation owns. There is no distinction between
"this application can only read" and "this application can create and delete."

| With RBAC | Without RBAC (MVP) |
|---|---|
| Endpoint Admin can manage Templates and Endpoints. HS Admin can manage HealthcareServices. Consumer can only read. | Any application with write scope for the organisation can do all write operations on all resource types that organisation owns. |

**Mitigation:**
- Scope-based restrictions on application registration (read-only vs read-write) provide
  a coarse-grained separation at the application level
- The BaRS Proxy and other read-only consumers would be registered with `read` scope only
  and physically cannot perform write operations
- Write access is limited to applications explicitly registered with `write` scope

**Risk accepted:** A supplier's write-enabled application can modify both Templates and
HealthcareServices belonging to that organisation. No fine-grained separation between
"endpoint management" and "service management" within the same organisation.

---

#### 3. No Programme Admin or System Admin capabilities

**Impact: Low (for MVP)**

Without the Programme Admin and System Admin roles:
- No ability to perform cross-organisation operations (e.g. a programme team bulk-updating
  all services of a specific type)
- No admin override for exceptional circumstances
- Hard deletes remain admin-only by convention but are not technically enforced by role

| With RBAC | Without RBAC (MVP) |
|---|---|
| Programme Admin can manage all Endpoints of a specific connection type across organisations | Not possible — each organisation manages only its own resources |
| System Admin can perform any operation on any resource | Not possible — ownership rules are absolute |

**Mitigation:**
- For the MVP, the R&M team's batch pipeline acts as the de facto system admin — it
  operates under a privileged registered application with broad write scope
- Cross-organisation operations are handled by the R&M team operationally, not
  self-service
- Hard delete access is controlled by restricting which applications are registered with
  delete scope — not by user role

**Risk accepted:** No self-service cross-organisation management. R&M team is the
operational backstop for anything outside a single organisation's scope.

---

#### 4. CIS2 availability question becomes irrelevant

**Impact: Positive**

The open question "Is CIS2 available to non-NHS users (e.g. private supplier staff)?" is
eliminated entirely. All access is application-restricted — suppliers authenticate their
*application*, not their *staff*. No CIS2 dependency means:
- No dependency on CIS2 role catalogue
- No need to define or register EPC-specific roles
- No integration with smartcard infrastructure
- No blocker for private-sector supplier access

---

#### 5. Admin UI cannot be built until RBAC is in place

**Impact: Informational**

An admin UI (browser-based interface for managing Endpoints, Templates, and
HealthcareServices) is not in scope for the EPC. However, if an admin UI is required
in future, it is **dependent on RBAC being delivered first** — a user-facing interface
requires CIS2 authentication to identify and authorise the individual user. Until RBAC
is in place, there is no mechanism to authenticate a human user or enforce what operations
they are permitted to perform through a UI.

All management in the current model is performed via:
- The R&M team's CSV-based batch pipeline
- Supplier automation calling the API directly with application-restricted tokens

---

#### 6. Delegated authority model simplified

**Impact: Low**

Without RBAC, the delegated authority model (supplier acting on behalf of a provider) is
simplified to a single rule:

> A caller may write to a resource if their token's ODS code OR Product ID matches the
> ownership fields on the resource.

The more nuanced CIS2-based model (where delegation is explicitly registered and revocable
per-user) is deferred. The Product ID mechanism still provides delegated authority at the
*application* level, which is sufficient for the MVP.

---

### API changes for MVP (deferred items)

The following are deferred from the MVP API implementation and will be added when RBAC
is delivered:

#### Deferred from the API

| Item | Detail |
|---|---|
| **CIS2 token validation path** | The API does not need to handle user-restricted bearer tokens, extract user identity claims, or validate CIS2 role assertions in MVP. Only application-restricted token validation is implemented initially. |
| **Role check middleware** | No middleware or Lambda logic to extract the user's role from the token and compare it against the operation's required role. The MVP authorisation check is: valid token → ODS match → Product ID match → permitted. |
| **`DELETE` restriction by role** | Without System Admin role enforcement in MVP, hard delete is controlled by application registration scope rather than per-request role check. The API still supports `DELETE` but does not verify a System Admin role claim until RBAC is delivered. |
| **User identity in audit records** | Audit records capture `client_id`, `ods_code`, and `product_id` from the application token only. Individual user identity fields are added when CIS2 is integrated. |
| **Role-specific error responses** | No `403 Forbidden` responses based on "you have the wrong role for this operation" in MVP. The only 403 scenarios are ODS mismatch and Product ID mismatch. Role-based 403s are added with RBAC. |

#### Simplified for MVP (expanded to full model later)

| Item | Detail |
|---|---|
| **Token validation** | Single path only — application-restricted signed JWT. Branching logic for CIS2 tokens is added when RBAC is delivered. |
| **Authorisation decision tree** | Reduced from 5 steps (token valid → ODS spoofing check → role check → ODS ownership → Product ID ownership) to 3 steps (token valid → ODS spoofing check → ODS + Product ID ownership) for MVP. Full decision tree restored with RBAC. |
| **Operation permissions table** | All write operations are permitted for any authenticated application with `write` scope and matching ownership. Per-operation role matrix is added with RBAC. |
| **Security headers** | No need to forward or process CIS2-specific claims (e.g. `selected_roleid`, `nhsd_session_urid`) in MVP. Only `client_id`, `ods_code`, and `product_id` are extracted from the token. |

#### Unchanged between MVP and final solution

| Aspect | Detail |
|---|---|
| **ODS ownership check** | Enforced on all write operations in MVP and final. |
| **Product ID ownership check** | Enforced in MVP and final. |
| **ODS spoofing protection** | Enforced in MVP and final. |
| **`header: private` redaction** | Applies in MVP and final. |
| **All CRUD operations** | `GET`, `POST`, `PUT`, `DELETE` on all resource types exist in both MVP and final. The operations are the same — only the authorisation gate is expanded later. |
| **Error response format** | FHIR OperationOutcome throughout. Error codes unchanged. |
| **Request/response payloads** | No change to any FHIR resource payloads, search parameters, or response structures between MVP and final. |

#### OAS (OpenAPI Specification) changes for MVP

| Item | MVP | Final solution |
|---|---|---|
| **Security schemes** | `OAuth_AppRestricted` only | `OAuth_AppRestricted` + `OAuth_CIS2` |
| **Security requirements per operation** | Single scheme for all operations | Per-operation (some require CIS2, some allow app-restricted) |
| **Scopes** | `read` and `write` | `read`, `write`, plus role-based scopes |
| **Error documentation** | ODS/Product ID mismatch 403 only | ODS/Product ID mismatch 403 + role-based 403 |

---

### Summary: What the MVP auth model looks like (before RBAC is delivered)

| Aspect | MVP (RBAC deferred) | Final solution (RBAC delivered) |
|---|---|---|
| Authentication | Application-restricted (signed JWT) | Application-restricted + CIS2 user-restricted |
| Identity granularity | Application (client_id + ODS + Product ID) | Application + individual user |
| Write authorisation | ODS ownership + Product ID ownership | ODS + Product ID + role check |
| Read authorisation | All resources visible (except private address redaction) | Same |
| Audit attribution | Application + ODS code | Application + ODS code + individual user identity |
| Admin override | R&M batch pipeline (privileged application) | System Admin role |
| Cross-org operations | R&M team only | Programme Admin role |
| Hard delete | Restricted by application registration (scope) | System Admin role |
| Admin UI | None — API only (not in scope; requires RBAC as prerequisite) | CIS2-gated admin interface (if required) |

---

### Decision

| Decision | Rationale |
|---|---|
| **Defer RBAC from MVP** | CIS2 integration adds significant delivery complexity (role catalogue, smartcard infrastructure, UI). The ODS + Product ID ownership model provides sufficient access control for the MVP's operational model (R&M team + supplier automation). Individual accountability and fine-grained roles are added in a subsequent release. |

---

### When to deliver

RBAC should be added when any of the following become true:

1. **Self-service admin UI is required** — users need to log in and perform operations
   through a browser-based interface
2. **Regulatory requirement** — an audit or compliance review requires individual-level
   attribution of changes
3. **Multi-user organisations need internal separation** — a single organisation has
   multiple teams that should not be able to modify each other's resources
4. **Programme-level operations** — a programme team needs to manage resources across
   multiple organisations without using the R&M batch pipeline

---

## Related documents

| Document | Description |
|----------|-------------|
| [Authentication and Authorisation](./authorisation.md) | Full RBAC model (target state) |
| [Interim Support Process Access](./interim-support-process-access.md) | R&M team operational access model |
| [Managing Endpoint Templates](./manage-endpoint-template.md) | Template lifecycle operations |
| [Managing Endpoints](./manage-endpoint.md) | Endpoint lifecycle and supplier switches |
| [Managing HealthcareServices](./manage-healthcare-service.md) | Service lifecycle operations |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | This document |
| 2 | Observability (Odin) | [EPC MVP Deferral — Observability](./mvp-deferral-observability.md) |
