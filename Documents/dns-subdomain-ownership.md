# BaRS EPC — DNS Subdomain Ownership & mTLS Certificates

## Overview

This document records the DNS subdomains, Route 53 hosted zones, and mTLS client
certificates used by the **BaRS Endpoint Catalogue (EPC)** backend. These subdomains host
the AWS API Gateway custom domains for EPC; **mTLS** is configured on each endpoint to
authenticate incoming requests from the APIM BaRS Proxy (see
[bars-proxy-to-epc-proxy-chaining.md](./bars-proxy-to-epc-proxy-chaining.md)).

Values below reflect the current AWS state (dev account `826164438582`, region
`eu-west-2`) verified against Route 53 and the Terraform configuration in
`bars-endpoint-catalogue-infra`.

> **Naming convention (target):** `endpoint-catalogue-<env>.national.nhs.uk` — the legacy
> `barsepc-*` prefix is being retired. Currently applied to **dev** only; **int** and
> **staging** still use the legacy `barsepc-*` names in AWS and require a rename +
> re-delegation. This change is **backend-facing only** — the consumer-facing APIM URL is
> unaffected and remains `https://<env>.api.service.nhs.uk/endpoint-catalogue/FHIR/R4`.

## Environment mapping (APIM ↔ EPC backend)

The Apigee (APIM) environment names do not match the EPC backend environment names one to
one. In particular, the Apigee **`ref`** environment maps to the EPC **`staging`**
environment.

| EPC backend | Apigee / APIM | mTLS cert name |
|-------------|---------------|----------------|
| dev | dev | `epc.dev.api.service.nhs.uk` (CN) |
| int | int | `epc.int.api.service.nhs.uk` (SAN) |
| staging | **ref** | `epc.ref.api.service.nhs.uk` (SAN) |
| prod | prod | *(separate production certificate)* |

## Registered subdomains

| Subdomain | Environment | AWS Account ID | Hosted zone ID | Status |
|-----------|-------------|----------------|----------------|--------|
| `endpoint-catalogue-dev.national.nhs.uk` | Development | 826164438582 | `Z03119491BA7P8BDBEFZK` | Delegated / live |
| `barsepc-int.national.nhs.uk` *(rename to `endpoint-catalogue-int` pending)* | Integration | 826164438582 | `Z0418864395K44ZPUCKP1` | Live (delegated) |
| `barsepc-staging.national.nhs.uk` *(rename to `endpoint-catalogue-staging` pending)* | Staging | 826164438582 | `Z0950167191DK6OQRYAOY` | Live (delegated) |
| `endpoint-catalogue-prod.national.nhs.uk` | Production | *[prod account ID]* | *[hosted zone ID]* | Pending (zone not yet created; separate prod account) |

## NS records

Send these to `dnsteam@nhs.net` for delegation. Renaming a Route 53 zone is a
destroy + recreate, which produces **new** NS records that must be re-delegated — this
delegation lead time is the critical path, not the engineering.

**Development — `endpoint-catalogue-dev.national.nhs.uk`** (zone recreated; NS updated)

```
ns-1140.awsdns-14.org
ns-966.awsdns-56.net
ns-472.awsdns-59.com
ns-2025.awsdns-61.co.uk
```

**Integration — `barsepc-int.national.nhs.uk`** (unchanged)

```
ns-1731.awsdns-24.co.uk
ns-1308.awsdns-35.org
ns-960.awsdns-56.net
ns-18.awsdns-02.com
```

**Staging — `barsepc-staging.national.nhs.uk`**

```
ns-407.awsdns-50.com
ns-727.awsdns-26.net
ns-1645.awsdns-13.co.uk
ns-1066.awsdns-05.org
```

**Production** — to be captured once the zone is created in the production account.

## Purpose

These subdomains host the AWS API Gateway custom domains for the BaRS Endpoint Catalogue
(EPC) service. mTLS is configured on each endpoint to authenticate incoming requests from
the APIM BaRS Proxy. On the EPC side, mTLS is Terraform-managed via the
`api_custom_domain_enabled` toggle in `bars-endpoint-catalogue-infra`.

## Certificate information

mTLS client certificates are **one certificate per proxy**, not one per environment: the CN
is the `dev` name and the other environments are **Subject Alternative Names (SANs)**.

### EPC Proxy (`.service.nhs.uk`)

- **CN:** `epc.dev.api.service.nhs.uk`
- **SAN:** `epc.int.api.service.nhs.uk`, `epc.ref.api.service.nhs.uk` (`ref` = EPC staging)
- **CA / path to live:** DEV CA. The producer generates the key + CSR; **NHS DIR signs the
  CSR and supplies the Root + Sub CA chain** (both via ResolveIT); APIM only deploys the
  client certificate onto Apigee.

### BaRS Proxy (`.platform.nhs.uk`) — for reference

- **CN:** `bars.dev.api.platform.nhs.uk`
- **SAN:** `bars.int.api.platform.nhs.uk`, `bars.ref.api.platform.nhs.uk`
- **Chain:** NHS PTL G2 — `NHS INT Authentication G2` (sub CA) → `NHS PTL Root Authority G2`
  (root).

| Environment | Path to live | Status |
|-------------|--------------|--------|
| Development | DEV CA | Issued / active |
| Integration | INT CA | Pending |
| Staging (Apigee `ref`) | REF/INT CA | Pending |
| Production | NHS PKI Production CA | Pending |

## Open items

- **int / staging rename:** AWS zones are still `barsepc-int` / `barsepc-staging`; rename to
  `endpoint-catalogue-*` requires new zones → DNS re-delegation via `dnsteam@nhs.net` → new
  certificates → cutover.
- **Production:** hosted zone not yet created; account ID, zone ID, and NS records to be
  captured once provisioned.

## Change log

| Date | Change | By |
|------|--------|-----|
| 2026-09-24 | Recorded real zone IDs + NS records (dev/int/staging); documented APIM `ref` ↔ EPC `staging` mapping and mTLS CN/SAN structure; noted int/staging pending rename and prod pending | Gabriele Manna |
| 2026-05-06 | Page created, subdomains requested | Mukhtar Ata |
