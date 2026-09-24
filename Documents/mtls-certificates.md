# BaRS EPC — mTLS Certificates

## Overview

This document records the **mTLS client certificates** used to authenticate the APIM
proxies against the BaRS Endpoint Catalogue (EPC) backend, and the corresponding BaRS Proxy
certificate for reference. The DNS subdomains and Route 53 hosted zones these certificates
are bound to are documented separately in [dns-subdomains.md](./dns-subdomains.md).

mTLS is the *transport-layer* mutual authentication; it is complementary to the OAuth
bearer-token authentication that applications use, documented in
[api-token-authentication.md](./api-token-authentication.md). On the EPC side, mTLS is
Terraform-managed via the `api_custom_domain_enabled` toggle in
`bars-endpoint-catalogue-infra`.

> **Authoritative deep-dive:** the full backend-mTLS runbook (per-environment steps,
> keystore/target-server config, ResolveIT process, verification) lives in the wiki:
> **EPC API Gateway — Backend mTLS Configuration**
> (`bars-endpoint-catalogue-documentation/wiki/components/epc-gateway-mtls-configuration.md`).
> This page is a summary of the certificate identities and ownership; consult the wiki for
> the operational detail.

## Two names — the address vs the identity

Two similar-looking FQDNs are involved and must not be confused:

| Name | Role | DNS-resolvable? | Certificate |
|------|------|-----------------|-------------|
| `endpoint-catalogue-<env>.national.nhs.uk` | **Server domain** — the *address* Apigee calls to reach the EPC backend | Yes (Route 53 custom domain) | Public **ACM server certificate** (DigiCert-signed) |
| `epc.<env>.api.service.nhs.uk` | **Client-cert CN** — the *identity* the EPC proxy presents | **No — virtual FQDN, never navigated to** | NHS **client certificate** (validated by the gateway truststore) |

In one line: *Apigee connects to `endpoint-catalogue-<env>.national.nhs.uk` and, to prove who
it is, presents a client cert whose CN is `epc.<env>.api.service.nhs.uk`.* The client-cert CN
is only ever used inside the CSR, the Apigee keystore/target-server config, and truststore
validation.

## How mTLS works — two phases

1. **Phase 1 — server authentication (standard TLS, no custom trust).** Apigee connects to
   `endpoint-catalogue-<env>.national.nhs.uk` and validates the EPC server certificate
   (AWS ACM, DigiCert-signed) against public CA roots.
2. **Phase 2 — client authentication (the mTLS step).** The EPC API Gateway requests a
   certificate; Apigee presents the NHS-signed **client certificate**; the gateway validates
   it against the NHS CA chain held in an **S3 truststore**. On success the request is
   forwarded to the EPC Lambda.

## Certificate structure

mTLS client certificates are **one certificate per proxy**, not one per environment: the CN
is the `dev` name and the other environments are **Subject Alternative Names (SANs)**. The
gateway validates by **CA chain, not by CN**, so any certificate issued under the NHS PTL G2
chain is accepted (this is what allows a single multi-SAN cert per proxy across PTL envs).

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

- **CN (virtual FQDN):** `epc.dev.api.service.nhs.uk`
- **SAN:** `epc.int.api.service.nhs.uk`, `epc.ref.api.service.nhs.uk` (`ref` = EPC staging)
- **Key:** RSA-4096 minimum, SHA-256, `extendedKeyUsage = clientAuth`.
- **Deployment:** the EPC Proxy **self-serves** its client certificate via Proxygen (the
  private key never leaves us). See [epc-proxy-proxygen.md](./epc-proxy-proxygen.md).

## BaRS Proxy certificate (`.platform.nhs.uk`) — for reference

- **CN (virtual FQDN):** `bars.dev.api.platform.nhs.uk`
- **SAN:** `bars.int.api.platform.nhs.uk`, `bars.ref.api.platform.nhs.uk`
- **Chain:** NHS PTL G2 — `NHS INT Authentication G2` (sub CA) → `NHS PTL Root Authority G2`
  (root).
- **Deployment:** the BaRS Proxy certificate is **handed to the APIM team** to load on Apigee.

## Ownership and signing

The mTLS certificate steps are the **producer (EPC) team's** responsibility — not the APIM
team's:

- The **producer (EPC) team** generates the client key + CSR, stores the signed cert, and
  builds/uploads the truststore.
- The **NHS DIR team** is the signing authority (Spine PKI): it signs the CSR via a
  **ResolveIT** ticket **and supplies the Root + Sub CA chain** bundled with the signing.
- The **APIM team's** only role is deploying the signed BaRS client cert onto Apigee (EPC is
  self-served via Proxygen).

## Truststore

The gateway validates the presented client cert against the **NHS Root + Sub CA chain**,
concatenated into a single PEM and held in S3
(`s3://bars-terraform-state/mtls/truststore/<env>/truststore.pem`). AWS API Gateway performs
**no revocation check** at the gateway.

## Testing without mTLS

mTLS is enforced **only on the custom domain** (`endpoint-catalogue-<env>.national.nhs.uk`).
The default AWS `execute-api` URL (`https://<api-id>.execute-api.eu-west-2.amazonaws.com/`)
does **not** enforce mTLS and remains available for Postman and automated e2e tests. Keep
`BASE_URL` / test config pointed at the execute-api URL, not the custom domain. For **prod**,
direct `execute-api` access is blocked (resource policy) so all traffic must go through
Apigee with a valid client cert.

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

## References

- Wiki: `bars-endpoint-catalogue-documentation/wiki/components/epc-gateway-mtls-configuration.md`
  (authoritative backend-mTLS runbook).
- [Backend Mutual TLS and DNS](https://nhsd-confluence.digital.nhs.uk/spaces/APM/pages/558502608/Backend+Mutual+TLS+and+DNS)
  🔒 — NHS API Platform producer-side guide.
- [Configuring mutual TLS authentication for a REST API](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html)
  — AWS truststore requirements.
- [`NHSDigital/api-management-cert-generation`](https://github.com/NHSDigital/api-management-cert-generation)
  — "easy mode" client key + CSR generation.
