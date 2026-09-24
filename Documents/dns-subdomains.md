# BaRS EPC — DNS Subdomains

## Overview

This document records the DNS subdomains, Route 53 hosted zones, and NS records used by the
**BaRS Endpoint Catalogue (EPC)** backend. These subdomains host the AWS API Gateway custom
domains for EPC. The mTLS certificates that authenticate the APIM BaRS Proxy against these
domains are documented separately in [mtls-certificates.md](./mtls-certificates.md).

Values below reflect the current AWS state (dev account `826164438582`, region
`eu-west-2`) verified against Route 53 and the Terraform configuration in
`bars-endpoint-catalogue-infra`.

> **Naming convention (target):** `endpoint-catalogue-<env>.national.nhs.uk` — the legacy
> `barsepc-*` prefix is being retired. Currently applied to **dev** only; **int** and
> **staging** still use the legacy `barsepc-*` names in AWS and require a rename +
> re-delegation. This change is **backend-facing only** — the consumer-facing APIM URL is
> unaffected and remains `https://<env>.api.service.nhs.uk/endpoint-catalogue/FHIR/R4`.

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
the APIM BaRS Proxy (see [mtls-certificates.md](./mtls-certificates.md) and
[bars-proxy-to-epc-proxy-chaining.md](./bars-proxy-to-epc-proxy-chaining.md)). On the EPC
side, mTLS is Terraform-managed via the `api_custom_domain_enabled` toggle in
`bars-endpoint-catalogue-infra`.

## Open items

- **int / staging rename:** AWS zones are still `barsepc-int` / `barsepc-staging`; rename to
  `endpoint-catalogue-*` requires new zones → DNS re-delegation via `dnsteam@nhs.net` → new
  certificates → cutover.
- **Production:** hosted zone not yet created; account ID, zone ID, and NS records to be
  captured once provisioned.
