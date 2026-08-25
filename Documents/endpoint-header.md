# Endpoint Header Attribute

> ## ⚠️ Important: `header` has been misused by the requirements
>
> The requirements described in this document repurpose `Endpoint.header` as an
> address-visibility control that takes the values `public` or `private`. **This is not
> what `header` means in FHIR R4, and this usage is non-conformant.**
>
> **True FHIR R4 meaning of `Endpoint.header`**
> ([FHIR R4 Endpoint](https://hl7.org/fhir/R4/endpoint.html)):
>
> | Aspect | FHIR R4 definition | How this document uses it |
> |--------|--------------------|---------------------------|
> | Type | `string` | `string` |
> | Cardinality | `0..*` (a **list** of strings) | Single value |
> | Purpose | Connection headers a client must send when contacting the endpoint's `address` (e.g. HTTP headers like `Authorization` or `Content-Type`) | A privacy/visibility flag for the `address` field |
> | Valid values | Any header string, e.g. `"Authorization: Bearer <token>"`, `"Content-Type: application/fhir+json"` | `"public"` / `"private"` |
>
> In FHIR, `header` carries the *additional headers to include as part of the request* when
> a system connects to the endpoint. It is a transport/connection detail, **not** an access
> or visibility control. A conformant example looks like:
>
> ```json
> "header": [
>   "Content-Type: application/fhir+json",
>   "Authorization: Bearer <token>"
> ]
> ```
>
> **Implications of the misuse**
>
> - Instances that set `header: "public"` / `"private"` still *validate* (any string is legal),
>   but they carry a meaning no FHIR consumer will understand. A standard client reads `header`
>   as headers to send on the wire — it would attempt to send `public` / `private` as a request
>   header, which is meaningless.
> - The value is single, but FHIR `header` is `0..*`; tooling generated from the FHIR
>   `StructureDefinition` will type it as a list/array, not a scalar.
> - Address visibility is an authorisation concern. It should be enforced by the API layer
>   (owner check via `NHSD-End-User-Organisation-ODS` vs `managingOrganization`) and, if it
>   must be modelled on the resource, expressed through a **FHIR extension** or a profiled
>   element — not by overloading a core element with a non-standard meaning.
>
> **Recommendation:** rename this concept to something that reflects its purpose (e.g. an
> `address-visibility` extension or a dedicated field), and reserve `header` for its FHIR
> R4 meaning. The behaviour described below reflects the *current (non-conformant)*
> requirements and is retained for reference until that decision is made.

---

## Overview

The `header` field on an `Endpoint` or `Template` controls the visibility of the `address`
field — the URL that connecting parties use to communicate with the endpoint.

> **Note:** A Template is a specialised Endpoint. Both are FHIR `Endpoint` resources and
> both carry a `header` field. The visibility rule applies equally to both, but the
> operational impact differs depending on whether the field appears on an Endpoint (child,
> service-specific) or a Template (parent, shared protocol configuration).

---

## Values

The `header` field accepts two values:

| Value | Meaning |
|-------|---------|
| `public` | The `address` field is visible to all authenticated consumers. |
| `private` | The `address` field is only visible to the organisation that owns the Endpoint or Template. All other consumers receive the resource with `address` omitted. |

---

## The privacy rule

**If `header` is `private`, only the owning organisation can see the `address`.**

Ownership is determined by the `managingOrganization` field on the Template (which holds
the ODS code of the organisation that created and manages the endpoint configuration).
The requesting organisation is identified by the `NHSD-End-User-Organisation-ODS` header
on the API request.

If the requesting organisation's ODS code matches the `managingOrganization` ODS code on
the Template, the `address` is included in the response. Otherwise it is omitted.

---

## Behaviour by resource type

### On a Template

The `header` field on a Template controls visibility of the Template's own `address` — the
default or canonical address for that protocol configuration.

| header value | Requesting org is owner | Requesting org is not owner |
|-------------|------------------------|----------------------------|
| `public` | `address` included in response | `address` included in response |
| `private` | `address` included in response | `address` omitted from response |

When a consumer calls `GET /Endpoint/$template` or `GET /Endpoint/{id}/$template` and the
Template has `header: "private"`, non-owning consumers receive the Template with all fields
present except `address`.

**Example — Template with `header: "private"`, requested by non-owner:**

```json
{
  "resourceType": "Endpoint",
  "id": "t001-0000-0000-0000-000000000001",
  "status": "active",
  "name": "BaRS FHIR REST Endpoint",
  "connectionType": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
        "code": "hl7-fhir-rest",
        "display": "HL7 FHIR"
      }
    ]
  },
  "payloadType": [
    {
      "coding": [
        {
          "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type",
          "code": "bars",
          "display": "BaRS"
        }
      ]
    }
  ],
  "header": "private",
  "managingOrganization": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "A1001"
      }
    }
  ]
}
```

The `address` field is absent. The consumer can see the Template exists and what protocol
it supports, but cannot connect to it directly.

**Example — same Template, requested by the owner (ODS: A1001):**

```json
{
  "resourceType": "Endpoint",
  "id": "t001-0000-0000-0000-000000000001",
  "status": "active",
  "name": "BaRS FHIR REST Endpoint",
  "connectionType": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/endpoint-connection-type",
        "code": "hl7-fhir-rest",
        "display": "HL7 FHIR"
      }
    ]
  },
  "payloadType": [
    {
      "coding": [
        {
          "system": "https://fhir.nhs.uk/CodeSystem/EPC-endpoint-payload-type",
          "code": "bars",
          "display": "BaRS"
        }
      ]
    }
  ],
  "address": "https://bars.provider.nhs.uk/FHIR/R4",
  "header": "private",
  "managingOrganization": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "A1001"
      }
    }
  ]
}
```

The `address` is present because the requesting organisation matches `managingOrganization`.

---

### On an Endpoint

The `header` field on an Endpoint (child) controls visibility of the `address` in the
merged response returned by `GET /Endpoint`. Because the Lambda merges the child Endpoint
with its parent Template before returning the response, the `address` in the merged response
comes from the Template — but the visibility decision is governed by the Template's `header`
value, not the child's.

> The child Endpoint does not have its own `address` — it inherits it from the Template.
> The `header` field on the child Endpoint is therefore not the primary visibility control
> for the address. The Template's `header` is what the Lambda evaluates.

For completeness, the child Endpoint's `header` field is still present in the data model
and follows the same `public`/`private` semantics, but in practice the Template's `header`
is the authoritative visibility control for the merged response.

| Template header | Endpoint header | Address in merged response (non-owner) |
|-----------------|-----------------|----------------------------------------|
| `public` | `public` | Included |
| `public` | `private` | Included (Template is authoritative) |
| `private` | `public` | Omitted |
| `private` | `private` | Omitted |

---

## Behaviour in search responses

When Endpoints are returned via `GET /Endpoint` or included via
`GET /HealthcareService?_include=HealthcareService:endpoint`, the same rule applies:

- If the Template's `header` is `public` — `address` is present for all consumers.
- If the Template's `header` is `private` — `address` is omitted for non-owning consumers.

This means a Bundle may contain a mix of Endpoints where some have `address` populated and
others do not, depending on which Templates are public or private.

**Example Bundle — mixed visibility:**

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 2,
  "entry": [
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000001",
        "status": "active",
        "name": "BaRS Booking Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars", "display": "BaRS" }] }],
        "address": "https://bars.provider.nhs.uk/FHIR/R4",
        "header": "public"
      },
      "search": { "mode": "match" }
    },
    {
      "resource": {
        "resourceType": "Endpoint",
        "id": "e1a2b3c4-0000-0000-0000-000000000002",
        "status": "active",
        "name": "Internal Referral Endpoint",
        "connectionType": { "coding": [{ "code": "hl7-fhir-rest", "display": "HL7 FHIR" }] },
        "payloadType": [{ "coding": [{ "code": "bars-referral", "display": "BaRS Referral" }] }],
        "header": "private"
      },
      "search": { "mode": "match" }
    }
  ]
}
```

`e001` has `address` present — its Template is `public`.
`e002` has no `address` — its Template is `private` and the requesting organisation is not
the owner. The consumer can see the Endpoint exists and its protocol type, but cannot
connect to it.

---

## Summary

| Scenario | address visible? |
|----------|-----------------|
| Template `header: "public"`, any consumer | Yes |
| Template `header: "private"`, requesting org is owner | Yes |
| Template `header: "private"`, requesting org is not owner | No — field omitted |
| Endpoint (child) `header: "private"`, Template `header: "public"` | Yes — Template is authoritative |

---

## OAS schema note

The current `header` field on both `Endpoint` and `EndpointTemplate` schemas is defined as
a plain `string` with example `"public"`. It should be constrained to an enum to make the
valid values explicit:

```json
"header": {
  "type": "string",
  "enum": ["public", "private"],
  "example": "public"
}
```
