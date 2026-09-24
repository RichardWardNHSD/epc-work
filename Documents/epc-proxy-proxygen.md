# BaRS EPC — Building the EPC Proxy with Proxy Generator

## Overview

The **EPC Proxy** (the Apigee API product that fronts the Endpoint Catalogue) is built and
deployed using the NHS **Proxy Generator (Proxygen)**, rather than by hand-editing an Apigee
proxy bundle. This document is the entry point for that work; the concrete tooling and keys
live in the `proxygen/` folder of the workspace.

**Primary reference (NHS Confluence):**
[Building and deploying your API (alpha) with Proxy Generator](https://nhsd-confluence.digital.nhs.uk/spaces/APM/pages/678899024/Building+and+deploying+your+API+alpha+with+Proxy+Generator)

Related docs:

- [api-token-authentication.md](./api-token-authentication.md) — the OAuth bearer-token flow
  applications use against the proxy.
- [mtls-certificates.md](./mtls-certificates.md) — the mTLS client certificate the EPC Proxy
  presents to the backend (self-served via Proxygen's `mtls` secret).
- [bars-proxy-to-epc-proxy-chaining.md](./bars-proxy-to-epc-proxy-chaining.md) — the
  proxy-to-proxy connection model.

## How EPC uses Proxygen

- **Publishing model:** Direct-API ("Option 2") publishing via Proxygen. Helper tooling is
  in `proxygen/`.
- **Application / client:** `endpoint-catalogue-client` (Proxygen machine-user client used to
  authenticate to the Proxygen control-plane and deploy the proxy).
- **mTLS to the backend:** EPC **self-serves** its backend mTLS through Proxygen — the client
  certificate is uploaded as a Proxygen `mtls` secret and the OAS declares
  `target: external` / `security: mtls`. The private key never leaves us (contrast with the
  BaRS Proxy, whose cert is handed to APIM to load on Apigee). See
  [mtls-certificates.md](./mtls-certificates.md).

> Note: Proxygen authentication here is the **team's control-plane** authentication (deploy
> the proxy), which is distinct from the application-restricted token flow that *consumers*
> use to call the API — see [api-token-authentication.md](./api-token-authentication.md).
