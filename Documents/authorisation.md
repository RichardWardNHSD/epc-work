# Authentication and Authorisation

## Authentication

### Access modes

The Endpoint Catalog API has **two access modes**:

| Access Mode | Authentication | Use Case |
|---|---|---|
| **Application-restricted** | Signed JWT → bearer token | System-to-system operations (BaRS Proxy lookups, DoS queries, supplier automation) — no human user present |
| **User-restricted (CIS2)** | NHS CIS2 authentication → bearer token | Administrative operations where a human is making a decision (creating templates, activating endpoints, managing healthcare services) |

Both modes use OAuth 2.0 bearer tokens. The difference is how the token is obtained and
what identity it carries.

---

### Application-restricted access

This mode authenticates the **calling application** but not the end user. It is used for
automated, unattended operations where no human is in the loop.

The Endpoint Catalog API uses the NHS England
[Application-restricted RESTful APIs — signed JWT authentication](https://digital.nhs.uk/developer/guides-and-documentation/security-and-authorisation/application-restricted-restful-apis-signed-jwt-authentication)
pattern.

**The flow:**

1. **Registration** — the calling application is registered on the NHS England API Platform
   (developer portal) and issued a client ID and a public/private key pair
2. **Token request** — the application constructs a signed JWT using its private key and
   sends it to the NHS England OAuth 2.0 token endpoint
3. **Token issuance** — the token endpoint validates the JWT signature against the
   application's registered public key and returns a short-lived bearer token
4. **API call** — the application includes the bearer token in the `Authorization` header
5. **Validation** — the API validates the token (signature, expiry, audience) before
   processing the request

**Token contents:**
- Application identity (client ID)
- Organisation (ODS code)
- Product ID
- Scopes (read, write)

**When to use:** BaRS Proxy resolving endpoints at runtime, DoS pulling endpoint data,
GP Connect looking up a target, supplier automation creating/updating endpoints in bulk.

---

### User-restricted access (CIS2)

This mode authenticates both the **calling application** and the **end user**. The end user
must be present, authenticated via NHS Care Identity Service 2 (CIS2), and authorised for
the operation they are performing.

For full details, see:
[User-restricted RESTful APIs — NHS CIS2 separate authentication and authorisation](https://digital.nhs.uk/developer/guides-and-documentation/security-and-authorisation/user-restricted-restful-apis-nhs-cis2-separate-authentication-and-authorisation)

**The flow:**

1. **User authenticates** via CIS2 (smartcard, Windows Hello, or NHS Identity app) —
   establishes identity at IAL3/AAL2+
2. **Application requests token** — uses the CIS2-issued identity assertion to request an
   OAuth 2.0 bearer token from the NHS England token endpoint
3. **Token issuance** — the token endpoint returns a bearer token containing both the
   application identity and the user identity
4. **API call** — the application includes the bearer token in the `Authorization` header
5. **Validation** — the API validates the token and extracts the user's identity, role, and
   organisation to enforce RBAC

**Token contents:**
- Application identity (client ID)
- User identity (SDS User ID or UUID)
- User's selected role (e.g. Endpoint Admin, HS Admin)
- User's organisation (ODS code)
- Scopes

**When to use:** Human administrators managing endpoints via an admin UI — DoS leads
activating endpoints, suppliers configuring templates, programme admins performing bulk
operations.

---

### Token lifecycle (both modes)

| Aspect | Value |
|--------|-------|
| Token type | OAuth 2.0 bearer token |
| Issued by | NHS England OAuth 2.0 token endpoint |
| Lifetime | Short-lived (typically 5–10 minutes) |
| Refresh | Application requests a new token when the current one expires |
| Revocation | Tokens cannot be individually revoked; they expire naturally |

---

### Sequence diagrams

#### Application-restricted authentication flow

![Application-restricted authentication flow](./auth-app-restricted-flow.drawio)

#### User-restricted (CIS2) authentication flow

![User-restricted CIS2 authentication flow](./auth-cis2-flow.drawio)

#### Write operation authorisation check (both modes)

![Write operation authorisation decision tree](./auth-write-decision-tree.drawio)

---

## RBAC Model

### Roles

The following roles are defined for the Endpoint Catalog, aligned with the EPCSe
Requirements V1.8 Section 5.0:

| Role | Objects | Access | How enforced |
|------|---------|--------|-------------|
| **Consumer** | Endpoint retrieval by business identifier | Read only | App-restricted token with `read` scope; or CIS2 token with Consumer role |
| **Endpoint Admin** | Endpoint + its managing Organisation | CRUD | CIS2 token with Endpoint Admin role; ODS ownership check |
| **Healthcare Service Admin** | HealthcareService + its providing Organisation | CRUD | CIS2 token with HS Admin role; ODS ownership check |
| **Activation Admin** | Links between HealthcareServices and Endpoints | CRUD | CIS2 token with Activation Admin role; ODS ownership check |
| **Programme Admin** | All Endpoints of a specific connection type | CRUD | CIS2 token with Programme Admin role; connection type scope |
| **System Admin** | All data objects | CRUD | CIS2 token with System Admin role; no ownership restriction |

### Enforcement by access mode

| Operation | App-restricted | User-restricted (CIS2) |
|---|---|---|
| `GET /Endpoint` (consumer lookup) | ✅ | ✅ (any role) |
| `GET /HealthcareService` | ✅ | ✅ (any role) |
| `GET /List` | ✅ | ✅ (any role) |
| `POST /Endpoint/$template` (create template) | ❌ | ✅ Endpoint Admin, System Admin |
| `PUT /Endpoint/{id}/$template` (update template) | ❌ | ✅ Endpoint Admin, System Admin |
| `POST /Endpoint` (create endpoint) | ⚠️ Supplier automation only | ✅ Endpoint Admin, Activation Admin |
| `PUT /Endpoint/{id}` (update status/period) | ⚠️ Supplier automation only | ✅ Endpoint Admin |
| `POST /HealthcareService` | ❌ | ✅ HS Admin, System Admin |
| `PUT /HealthcareService/{id}` | ❌ | ✅ HS Admin, System Admin |
| `DELETE` (any) | ❌ | ✅ System Admin only |

⚠️ = permitted for registered supplier applications with `write` scope and matching ODS
ownership. This allows suppliers to automate endpoint management without requiring a
human to authenticate for every change.

### How roles are assigned

- **Application-restricted mode:** The role is effectively the application's registered
  scope (`read`, `read write`, `read write admin`). Assigned at application registration
  time on the NHS England API Platform.
- **User-restricted (CIS2) mode:** The role comes from the CIS2 token claims — the user's
  selected role at their organisation. EPC-specific roles would need to be registered in
  the CIS2 role catalogue or mapped from existing CIS2 roles.

### Product ID as an additional authorisation check

The requirements state: "The use of the identifier 'Product ID' will be used to facilitate
authorisation upon a given data object. Should the client application's assigned Product ID
not match that of one within the data object, Update and Delete operations will be
restricted."

This means:
- Each registered application has an assigned Product ID (from Digital Onboarding)
- Write operations check that the token's Product ID matches the resource's Product ID
- This prevents one supplier from modifying another supplier's endpoints even if they
  share the same ODS code

Product ID + ODS code together form the ownership key:
- ODS code = which organisation
- Product ID = which product within that organisation

### Implementation phases

| Phase | Access Mode | RBAC | Notes |
|-------|-------------|------|-------|
| **PoC** | Application-restricted only | Scopes + ODS ownership | Sufficient for BaRS Proxy reads and basic supplier writes |
| **Production** | Application-restricted + CIS2 | Full RBAC model + Product ID ownership | Admin UI uses CIS2; automated systems use app-restricted |

### Audit trail benefits

With CIS2 user-restricted access, the bearer token contains the individual user's identity.
The audit record (see audit requirements) can capture *which person* made the change, not
just which application. This satisfies EPCSe006 (Provision of Audit) more completely.

### Ownership model by resource type

HealthcareServices are provided by NHS organisations (pharmacies, trusts, GP practices) —
not by suppliers. A supplier owns the *Endpoint Template* (the technical connection), but
the *HealthcareService* belongs to the provider organisation. This means the ownership
model splits along resource type:

| Resource | Owned by | Ownership key | Product ID relevant? |
|---|---|---|---|
| **Template** | Supplier | `managingOrganization` ODS + Product ID | ✅ Yes |
| **Endpoint** | Supplier (inherited from Template) | `managingOrganization` ODS + Product ID | ✅ Yes |
| **HealthcareService** | Provider organisation (pharmacy, trust, GP) | `providedBy` ODS code | ❌ No — providers don't have Product IDs |
| **List** | Provider organisation (derived from HS) | `providedBy` ODS of referenced HS | ❌ No |

### Write authorisation rules by resource type

| Operation | Authorisation check |
|---|---|
| Write to Template | Token ODS matches `managingOrganization` AND token Product ID matches Template Product ID |
| Write to Endpoint | Token ODS matches `managingOrganization` AND token Product ID matches parent Template Product ID |
| Write to HealthcareService | Token ODS matches `providedBy` OR token Product ID matches a Product ID identifier on the HS (delegated authority — see below) |
| Write to List | Same rule as the referenced HealthcareService |
| Link HS to Endpoint (Activation) | Caller owns the HealthcareService (by either rule above); does NOT need to own the Template |

---

### Suppliers acting on behalf of provider organisations

#### The problem

In practice, suppliers frequently manage HealthcareServices on behalf of the provider
organisations they serve. For example:

- Sonar onboards 200 pharmacies to their platform and needs to create a HealthcareService
  for each one
- Pharmoutcomes rolls out a new blood pressure service across 500 pharmacies and needs
  to create HealthcareServices + link them to Endpoints
- A trust's IT supplier manages all endpoint configuration for the trust's Emergency
  Departments

The provider organisations (pharmacies, trusts) will not log in and do this themselves —
the supplier does it as part of their onboarding/deployment process.

#### How delegated authority works

The supplier's relationship with the provider organisation is established through the
**Product ID**. When a supplier is onboarded via the Digital Onboarding Service, their
Product ID is registered and linked to the organisations they serve.

The authorisation rule for HealthcareServices is:

> A caller may write to a HealthcareService if EITHER:
> 1. Their token ODS code matches `providedBy` (they ARE the provider), OR
> 2. Their token Product ID matches a Product ID identifier on the HealthcareService
>    (they are the supplier managing it on behalf of the provider)

This means:
- When a supplier creates a HealthcareService for a pharmacy, they set `providedBy` to
  the pharmacy's ODS code (the pharmacy owns it) and include their own Product ID as an
  identifier on the resource
- The supplier can subsequently update the HealthcareService because their Product ID
  matches
- The pharmacy can also update it because their ODS code matches `providedBy`
- If the pharmacy switches to a different supplier, the new supplier's Product ID replaces
  the old one — the old supplier loses write access

#### Example: Supplier onboarding a pharmacy

Sonar (ODS: `SONAR1`, Product ID: `PROD-SONAR-BARS-001`) creates a HealthcareService
for Swan Lake Pharmacy (ODS: `FH123`):

```json
{
  "resourceType": "HealthcareService",
  "identifier": [
    { "system": "https://fhir.nhs.uk/Id/dos-service-id", "value": "2000099999" },
    { "system": "https://fhir.nhs.uk/Id/product-id", "value": "PROD-SONAR-BARS-001" }
  ],
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FH123"
    }
  },
  "name": "Swan Lake Pharmacy — Blood Pressure Service",
  "active": true
}
```

**Who can modify this HealthcareService:**
- Sonar (`PROD-SONAR-BARS-001` matches the Product ID identifier) ✅
- Swan Lake Pharmacy (`FH123` matches `providedBy`) ✅
- Any other supplier or organisation ❌

**Who can modify Sonar's Template:**
- Only Sonar (`managingOrganization` ODS + Product ID match) ✅
- Swan Lake Pharmacy ❌ (they don't own the Template)

#### Supplier switching scenario

When Swan Lake Pharmacy switches from Sonar to MedRec:

1. MedRec creates a new Endpoint for the pharmacy using MedRec's Template
2. The HealthcareService's Product ID identifier is updated from `PROD-SONAR-BARS-001`
   to `PROD-MEDREC-001`
3. Sonar loses write access to the HealthcareService (their Product ID no longer matches)
4. MedRec gains write access (their Product ID now matches)
5. The pharmacy retains write access throughout (their ODS code always matches `providedBy`)

#### Audit trail

When a supplier acts on behalf of a provider, the audit record captures:
- **Who made the change:** the supplier's application identity (from the bearer token)
- **On whose behalf:** the provider organisation (from `providedBy` on the resource)
- **What was changed:** the resource and field-level diff

This provides full traceability even when the supplier is doing the work.

#### Future enhancement: explicit delegation registry

For production, a more robust approach would add an explicit delegation registry:
- The provider organisation formally authorises a supplier to act on their behalf
- This relationship is stored separately (e.g. in the Digital Onboarding Service)
- The API checks this registry before allowing delegated writes
- The provider can revoke the delegation independently of the Product ID

This is not required for the PoC but should be considered for production.

### Open questions

| # | Question | For |
|---|----------|-----|
| 1 | Is CIS2 available to non-NHS users (e.g. private supplier staff)? If not, what's the auth path for suppliers? | Security / CIS2 team |
| 2 | Should write operations be restricted to user-restricted mode only, or should some writes be permitted via app-restricted (for supplier automation)? | Product / Architecture |
| 3 | How are EPC-specific roles (Endpoint Admin, HS Admin, etc.) mapped to CIS2 role assertions? Custom role codes or existing CIS2 roles? | IOPS / CIS2 team |
| 4 | Is the admin UI a separate application or part of the Digital Onboarding Service? | Product |
| 5 | Should Product ID be the primary ownership key instead of (or in addition to) ODS code? | Architecture |

---

## Authorisation

### Overview

Authorisation — determining whether an authenticated caller is permitted to perform a
specific operation on a specific resource — is enforced by the API on all write operations.

The authorisation model is based on **resource ownership**. Each resource in the Endpoint
Catalog is owned by an organisation identified by an ODS code. A caller may only perform
destructive operations (`PUT`, `PATCH`, `DELETE`) on resources they own.

Read operations (`GET`) are also affected by ownership in one specific case: if an Endpoint
or Template has `header` set to `private`, the `address` field is **redacted from the
response** unless the requesting organisation matches the owning organisation. This allows
suppliers to register private Endpoints that are not publicly addressable — only the owner
can see the actual URL.

> **⚠️ Note:** The `header: private` requirement is currently under discussion and may be
> struck from the specification. If removed, all Endpoint addresses would be visible to
> all authenticated callers regardless of ownership, and the address-redaction logic
> described in this document would not apply.

---

## The ownership rule

The ODS code supplied in the `NHSD-End-User-Organisation-ODS` request header must match
the owning organisation of the resource being modified. If the codes do not match, the API
returns `403 Forbidden`.

| Header | Description |
|--------|-------------|
| `NHSD-End-User-Organisation-ODS` | The ODS code of the organisation making the request. Required on all calls. |

---

## ODS code spoofing (man-in-the-middle protection)

### The threat

The `NHSD-End-User-Organisation-ODS` header is a plain string supplied by the API consumer.
Without additional validation, a malicious or misconfigured client could supply any ODS code
in that header — including one belonging to a different organisation — and potentially gain
write access to resources it does not own, or read the `address` field of private Endpoints
it should not see.

This is a form of **man-in-the-middle / identity spoofing attack**: the request arrives
looking like it originates from organisation `R778`, but the bearer token was actually issued
to `A1001`.

### Countermeasure — token claim validation

To prevent this, the API **must cross-check the `NHSD-End-User-Organisation-ODS` header
value against the organisation claims carried inside the bearer token** issued by the NHS
England OAuth 2.0 token endpoint (for application-restricted access) or CIS2 (for
user-restricted access). The bearer token is cryptographically signed by the identity
provider and cannot be tampered with by the consumer.

The relevant claim in the CIS2 access token is `selected_roleid` / `organisation` (the
exact claim name depends on the token profile in use — for NHS CIS2 combined
authentication and authorisation the organisation ODS code is available in the
`organisation.identifier` claim within the `nhsd_session_urid` or equivalent role
assertion).

The validation rule is:

> The ODS code in `NHSD-End-User-Organisation-ODS` **must exactly match** the ODS code
> asserted for the authenticated user's selected role in the bearer token claims. If they
> do not match, the request must be rejected with `403 Forbidden` before any ownership
> check is performed.

This check is performed **before** the resource-level ownership check described above, so
a spoofed header is caught at the token-validation layer regardless of which resource is
being accessed.

### Validation sequence

```
1. Verify bearer token signature and expiry (→ 401 if invalid/expired)
2. Extract the ODS code from the token's organisation claim
3. Compare with NHSD-End-User-Organisation-ODS header value
   └─ Mismatch → 403 Forbidden (SEND_FORBIDDEN) — stop processing
4. Proceed to resource-level ownership check
   └─ Mismatch → 403 Forbidden (SEND_FORBIDDEN) — stop processing
5. Execute the requested operation
```

### Error response — token/header ODS mismatch

```http
HTTP/1.1 403 Forbidden
Content-Type: application/fhir+json
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "forbidden",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "SEND_FORBIDDEN"
          }
        ]
      },
      "diagnostics": "The organisation in NHSD-End-User-Organisation-ODS ('R778') does not match the organisation asserted in the bearer token ('A1001'). Request rejected."
    }
  ]
}
```

### Worked example — spoofed ODS code rejected

Organisation `A1001` holds a valid bearer token issued to their application. They craft a
request with `NHSD-End-User-Organisation-ODS: R778` in an attempt to modify a Template
owned by `R778`.

```http
PUT /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...   ← token issued to A1001
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
NHSD-End-User-Organisation-ODS: R778             ← spoofed
```

The API decodes the bearer token and finds the organisation claim is `A1001`. This does not
match the header value `R778`. The request is rejected immediately with `403 Forbidden` —
the resource-level ownership check is never reached.

---

## Ownership by resource type

Each resource type stores its owning organisation in a different field:

| Resource | Ownership field | Notes |
|----------|----------------|-------|
| Template (`Endpoint/$template`) | `managingOrganization[].identifier.value` | The supplier organisation that owns and manages the Template |
| Endpoint | `managingOrganization[].identifier.value` | Inherited from the parent Template at read time — the owning organisation is the same as the Template's |
| HealthcareService | `providedBy[].identifier.value` | The organisation that provides the service |
| List | Derived from `subject` HealthcareService | The List is owned by the same organisation as the HealthcareService it references — the `providedBy` ODS code of that HealthcareService is used |

---

## Enforcement by operation

| Operation | Authorisation check | Applies to |
|-----------|--------------------|-----------| 
| `GET` (any) | No ownership check — all resources are returned. **Exception:** if an Endpoint or Template has `header: private`, the `address` field is omitted from the response unless `NHSD-End-User-Organisation-ODS` matches `managingOrganization` | Endpoint, Template |
| `POST /Endpoint/$template` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` in the request payload | Template |
| `PUT /Endpoint/{id}/$template` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the existing Template | Template |
| `DELETE /Endpoint/{id}/$template` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the existing Template | Template |
| `POST /Endpoint` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the parent Template | Endpoint |
| `PUT /Endpoint/{id}` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the existing Endpoint's parent Template | Endpoint |
| `PATCH /Endpoint/{id}` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the existing Endpoint's parent Template | Endpoint |
| `DELETE /Endpoint/{id}` | `NHSD-End-User-Organisation-ODS` must match `managingOrganization` of the existing Endpoint's parent Template | Endpoint |
| `POST /HealthcareService` | `NHSD-End-User-Organisation-ODS` must match `providedBy` in the request payload | HealthcareService |
| `PUT /HealthcareService/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the existing HealthcareService | HealthcareService |
| `PATCH /HealthcareService/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the existing HealthcareService | HealthcareService |
| `DELETE /HealthcareService/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the existing HealthcareService | HealthcareService |
| `POST /List` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the HealthcareService referenced in `subject` | List |
| `PUT /List/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the HealthcareService referenced in the existing List's `subject` | List |
| `PATCH /List/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the HealthcareService referenced in the existing List's `subject` | List |
| `DELETE /List/{id}` | `NHSD-End-User-Organisation-ODS` must match `providedBy` of the HealthcareService referenced in the existing List's `subject` | List |

---

## Error responses

### 403 Forbidden — ownership mismatch

Returned when the `NHSD-End-User-Organisation-ODS` header does not match the owning
organisation of the resource being modified.

```http
HTTP/1.1 403 Forbidden
Content-Type: application/fhir+json
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "forbidden",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "SEND_FORBIDDEN"
          }
        ]
      },
      "diagnostics": "The requesting organisation 'A1001' is not authorised to modify this resource. The resource is owned by 'R778'."
    }
  ]
}
```

### 401 Unauthorized — missing or invalid token

Returned when the bearer token is absent, expired, or invalid. The caller must
re-authenticate before retrying.

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/fhir+json
```

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "login",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "SEND_UNAUTHORIZED"
          }
        ]
      },
      "diagnostics": "Missing or expired bearer token."
    }
  ]
}
```

---

## Worked examples

### Read — private Endpoint, non-owner request

Organisation `A1001` retrieves Endpoints for a HealthcareService. One of the Endpoints has
`header: private` and is owned by `R778`. Because `A1001` does not match `R778`, the
`address` field is omitted from that Endpoint in the response.

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: A1001
```

The private Endpoint is returned in the Bundle but with `address` omitted:

```json
{
  "resource": {
    "resourceType": "Endpoint",
    "id": "0cb21027-a246-43e6-9c7a-35b17163eab1",
    "status": "active",
    "name": "Private Endpoint",
    "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
    "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
    "header": "private"
  }
}
```

The `address` field is absent. The Endpoint is visible — the caller knows it exists — but
cannot use it.

---

### Read — private Endpoint, owner request

The same request made by `R778` (the owner) returns the full Endpoint including `address`:

```http
GET /Endpoint?_has:HealthcareService:endpoint:_id=9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
NHSD-End-User-Organisation-ODS: R778
```

```json
{
  "resource": {
    "resourceType": "Endpoint",
    "id": "0cb21027-a246-43e6-9c7a-35b17163eab1",
    "status": "active",
    "name": "Private Endpoint",
    "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
    "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
    "address": "https://private.anytown-utc.nhs.uk/FHIR/R4",
    "header": "private"
  }
}
```

---

### Permitted — organisation modifies its own Template

Organisation `R778` updates a Template it owns. The `NHSD-End-User-Organisation-ODS`
header matches the Template's `managingOrganization`.

```http
PUT /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: R778
```

The Template's `managingOrganization` is `R778` → **permitted**, returns `200 OK`.

---

### Rejected — organisation attempts to modify another organisation's Template

Organisation `A1001` attempts to update a Template owned by `R778`.

```http
PUT /Endpoint/5fce3e6a-ba37-4289-84d1-cc3ebdb992f5/$template HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: c1ab3fba-6bae-4ba4-b257-5a87c44d4a91
X-Correlation-Id: 9562466f-c982-4bd5-bb0e-255e9f5e6689
NHSD-End-User-Organisation-ODS: A1001
```

The Template's `managingOrganization` is `R778`, but the header is `A1001` → **rejected**,
returns `403 Forbidden`.

---

### Permitted — organisation modifies its own HealthcareService

Organisation `R778` patches a HealthcareService it provides.

```http
PATCH /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/json-patch+json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: d2bc4fca-7cbf-5cb5-c368-6b98d55e5b02
X-Correlation-Id: a6673577-d093-5ce6-bf1f-366f0g6f7790
NHSD-End-User-Organisation-ODS: R778
```

The HealthcareService's `providedBy` is `R778` → **permitted**, returns `200 OK`.

---

### Permitted — organisation modifies the List for its own HealthcareService

Organisation `R778` updates the priority List for a HealthcareService it provides. The
List's ownership is derived from the `providedBy` of the referenced HealthcareService.

```http
PUT /List/f7d3e2a1-0000-0000-0000-000000000099 HTTP/1.1
Host: sandbox.api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
X-Request-Id: e3cd5gdb-8dcg-6dc6-d479-7c09e66f6c13
X-Correlation-Id: b7784688-e194-6df7-cg2g-477g1h7h8901
NHSD-End-User-Organisation-ODS: R778
```

The List's `subject` references `HealthcareService/9f2c6f12-...`, whose `providedBy` is
`R778` → **permitted**, returns `200 OK`.

---

## Summary

| Rule | Detail |
|------|--------|
| Token/header ODS validation | ODS code in `NHSD-End-User-Organisation-ODS` must match the organisation claim in the bearer token — checked **before** resource ownership. Mismatch → `403 Forbidden` |
| Read operations | Generally permitted without ownership check. **Exception:** if `header: private`, the `address` field is omitted unless `NHSD-End-User-Organisation-ODS` matches `managingOrganization` |
| Write operations (`POST`, `PUT`, `PATCH`, `DELETE`) | `NHSD-End-User-Organisation-ODS` must match the resource's owning organisation |
| Template and Endpoint ownership | `managingOrganization[].identifier.value` |
| HealthcareService ownership | `providedBy[].identifier.value` |
| List ownership | Derived from `providedBy` of the referenced HealthcareService |
| Mismatch response | `403 Forbidden` with `SEND_FORBIDDEN` error code |
| Missing/invalid token | `401 Unauthorized` with `SEND_UNAUTHORIZED` error code |
