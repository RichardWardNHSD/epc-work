# BaRS EPC — mTLS Certificates

## Overview

This document records the **mTLS client certificates** used to authenticate the APIM BaRS
Proxy against the BaRS Endpoint Catalogue (EPC) backend, and the corresponding BaRS Proxy
certificate for reference. The DNS subdomains and Route 53 hosted zones these certificates
are bound to are documented separately in [dns-subdomains.md](./dns-subdomains.md); the
proxy-to-proxy connection model is in
[bars-proxy-to-epc-proxy-chaining.md](./bars-proxy-to-epc-proxy-chaining.md).

mTLS is the *transport-layer* mutual authentication; it is complementary to the OAuth
bearer-token authentication that applications use, documented in
[api-token-authentication.md](./api-token-authentication.md). On the EPC side, mTLS is
Terraform-managed via the `api_custom_domain_enabled` toggle in
`bars-endpoint-catalogue-infra`.

## Certificate structure

mTLS client certificates are **one certificate per proxy**, not one per environment: the CN
is the `dev` name and the other environments are **Subject Alternative Names (SANs)**.

## Environment mapping (APIM ↔ EPC backend)

The Apigee (APIM) environment names do not match the EPC backend environment names one to
one. In particular, the Apigee **`ref`** environment maps to the EPC **`staging`**
environment.

| EPC backend | Apigee / APIM | EPC certificate name |
|-------------|---------------|----------------------|
| dev | dev | `epc.dev.api.service.nhs.uk` (CN) |
| int | int | `epc.int.api.service.nhs.uk` (SAN) |
| staging | **ref** | `epc.ref.api.service.nhs.uk` (SAN) |
| prod | prod | *(separate production certificate)* |

## EPC Proxy certificate (`.service.nhs.uk`)

- **CN:** `epc.dev.api.service.nhs.uk`
- **SAN:** `epc.int.api.service.nhs.uk`, `epc.ref.api.service.nhs.uk` (`ref` = EPC staging)
- **CA / path to live:** DEV CA. The producer generates the key + CSR; **NHS DIR signs the
  CSR and supplies the Root + Sub CA chain** (both via ResolveIT); APIM only deploys the
  client certificate onto Apigee.

## BaRS Proxy certificate (`.platform.nhs.uk`) — for reference

- **CN:** `bars.dev.api.platform.nhs.uk`
- **SAN:** `bars.int.api.platform.nhs.uk`, `bars.ref.api.platform.nhs.uk`
- **Chain:** NHS PTL G2 — `NHS INT Authentication G2` (sub CA) → `NHS PTL Root Authority G2`
  (root).

## Path to live

| Environment | Path to live | Status |
|-------------|--------------|--------|
| Development | DEV CA | Issued / active |
| Integration | INT CA | Pending |
| Staging (Apigee `ref`) | REF / INT CA | Pending |
| Production | NHS PKI Production CA | Pending |

## Open items

- **int / staging:** certificates pending issuance; a rename of the DNS zones to
  `endpoint-catalogue-*` (see [dns-subdomains.md](./dns-subdomains.md)) will require new
  certificates.
- **Production:** production certificate to be issued via the NHS PKI Production CA once the
  prod environment is provisioned.
