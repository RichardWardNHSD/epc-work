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
| `barsepc-staging.national.nhs.uk` *(rename to `endpoint-catalogue-staging` pending)* | Staging | 826164438582 | `Z0950167191DK6OQRYAOY` | Zone created, **not yet delegated** (does not resolve publicly) |
| `endpoint-catalogue.national.nhs.uk` | Production | *[prod account ID]* | *[hosted zone ID]* | Pending (zone not yet created; separate prod account) |

> Production drops the `-<env>` suffix (`endpoint-catalogue.national.nhs.uk`) per the wiki
> runbook; dev/int/staging keep the environment suffix.

## NS records

Send these to the NHS DNS team (`england.dnsteam@nhs.net`) for delegation. Renaming a Route 53 zone is a
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

## DNS delegation process

Delegation of an EPC subdomain under `national.nhs.uk` is requested by **email to the NHS
DNS team at `england.dnsteam@nhs.net`** — it is **not** a ServiceNow / Jira / ResolveIT
ticket. (ResolveIT is used only for the separate mTLS client-certificate signing — see
[mtls-certificates.md](./mtls-certificates.md) — do not raise DNS delegation through it.)
For dev the request was actioned by Martin Slater (Service Consultant, NHS England DNS team);
delegation completed the same day it was raised.

**Steps (repeat per environment):**

1. **Create or rename the Route 53 hosted zone** in Terraform
   (`bars-endpoint-catalogue-infra/terraform/envs/<env>/route53.tf`) to
   `endpoint-catalogue-<env>.national.nhs.uk`, then `terraform apply`. For int/staging this
   is a rename from the legacy `barsepc-*` name — a **destroy + recreate** that yields **new
   NS records**; use `create_before_destroy` / state import so a live domain is never dropped.
2. **Read the four NS records** of the new zone:
   ```bash
   aws route53 get-hosted-zone --id <ZONE_ID> \
     --query "DelegationSet.NameServers" --output text --region eu-west-2
   ```
   (or the Terraform output `<zone>_ns_records`).
3. **Email `england.dnsteam@nhs.net`** requesting delegation, using the template below (CC
   the owning team).
4. **Wait** for the DNS team to add the NS records (for dev this was actioned the **same
   day**), then allow time for public propagation. Treat delegation lead time as the
   **critical path**.
5. **Verify** it resolves publicly: `dig @8.8.8.8 NS <fqdn> +short` returns the four servers
   (dev verified resolving 2026-09-11).
6. **Enable** the custom-domain toggle (`api_custom_domain_enabled`) for the environment.

**Email template** (based on the actual dev request, 2026-09-03 — substitute `<env>` and the
zone-specific values):

```
To: england.dnsteam@nhs.net
Cc: <owning team>
Subject: DNS delegation for the EPC <env> environment

Hi,

I'm on the EPC team. We need a DNS delegation for the EPC <env> environment. We've created a
Route 53 hosted zone in our AWS account and its name servers need to be delegated from the
parent national.nhs.uk zone so the subdomain resolves publicly.

- Subdomain: endpoint-catalogue-<env>.national.nhs.uk
- AWS account: 826164438582 (eu-west-2)          # prod is a separate account
- Hosted zone ID: <zone-id>
- Action needed: create the NS delegation for endpoint-catalogue-<env>.national.nhs.uk in the
  parent national.nhs.uk zone, pointing at these name servers:
      <ns-1>
      <ns-2>
      <ns-3>
      <ns-4>

Could you please help with that?

Thank you.
Best regards,
<name>
```

**Per-environment status & specifics:**

- **int** — the legacy `barsepc-int` zone **is already delegated** and resolves. To move to
  `endpoint-catalogue-int` you must rename the zone (new NS) and **re-request delegation** for
  the new zone; keep `barsepc-int` live until cutover.
- **staging** — the `barsepc-staging` zone exists but is **not delegated**. Either request
  delegation for it as-is, or (preferred) rename to `endpoint-catalogue-staging` first and
  request delegation for the new zone.
- **prod** — separate production AWS account; the subdomain **drops the suffix**
  (`endpoint-catalogue.national.nhs.uk`); uses the NHS PKI Production CA; delegation is
  **out of scope until prod onboarding**. Prod also needs an API Gateway resource policy to
  block the default `execute-api` URL.

## Purpose

These subdomains host the AWS API Gateway custom domains for the BaRS Endpoint Catalogue
(EPC) service. Each custom domain carries a public **ACM server certificate**
(DigiCert-signed). mTLS is configured on each custom domain to authenticate incoming
requests from the APIM proxies (see [mtls-certificates.md](./mtls-certificates.md) and
[bars-proxy-to-epc-proxy-chaining.md](./bars-proxy-to-epc-proxy-chaining.md)). On the EPC
side, mTLS is Terraform-managed via the `api_custom_domain_enabled` toggle in
`bars-endpoint-catalogue-infra`.

mTLS is enforced **only on the custom domain**. The default AWS `execute-api` URL
(`https://<api-id>.execute-api.eu-west-2.amazonaws.com/`) bypasses mTLS and stays available
for Postman and automated e2e tests (except in prod, where direct `execute-api` access is
blocked).

## Open items

- **int / staging rename:** AWS zones are still `barsepc-int` / `barsepc-staging`; rename to
  `endpoint-catalogue-*` requires new zones → DNS re-delegation via `england.dnsteam@nhs.net` → new
  certificates → cutover.
- **Production:** hosted zone not yet created; account ID, zone ID, and NS records to be
  captured once provisioned.
