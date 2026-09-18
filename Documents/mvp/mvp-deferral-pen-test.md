# EPC MVP Deferral — Full Penetration Test

**Series:** EPC MVP Scope Deferrals
**Document:** 5 of N

---

## Requirement references

| Requirement | Description |
|---|---|
| Security assurance (path to live) | Independent security testing required before external exposure |
| NHS England security assurance / IT Health Check (ITHC) | Organisational requirement for penetration testing of internet-facing services before wider rollout |

---

## Purpose

This document analyses the deferral of a full, independent penetration test from the EPC
MVP. A complete penetration test — covering the API, backend, and infrastructure, with
findings remediated and formally signed off — remains a **mandatory prerequisite for full
rollout**. The MVP relies on secure-by-design controls, automated security scanning in the
build pipeline, and the fact that access is restricted to internal, trusted consumers.

> ⚠️ **Full external rollout must not proceed until a complete penetration test has been
> passed and its findings remediated.** This deferral applies only while the EPC is
> internal-only.

---

## What is a full penetration test

An independent, full-scope penetration test of the EPC would cover:

- **API layer** — authentication and authorisation bypass, token handling, ODS/Product ID
  ownership enforcement, ODS header spoofing, injection, FHIR-specific abuse (malformed
  resources, oversized payloads, parameter tampering on search)
- **Backend** — AWS API Gateway, Lambda business logic, DynamoDB access paths, IAM role
  scoping, secrets handling
- **Infrastructure** — network boundaries, mTLS configuration between Apigee and AWS,
  encryption in transit and at rest, environment isolation (INT pipeline vs PROD catalogue)
- **The CSV-to-API pipeline** — S3 bucket permissions, input validation, and the
  privileged app-restricted path the R&M pipeline uses into PROD
- **Reporting and remediation** — findings rated by severity, a remediation plan, retest of
  fixes, and a formal sign-off suitable for NHS England security assurance

---

## What is being deferred

| Deferred capability | Detail |
|---|---|
| Independent penetration test engagement | Booking an accredited (e.g. CHECK / CREST) testing team to test the full EPC |
| Full-scope test execution | API, backend, infrastructure, and CSV pipeline tested as a whole |
| Findings remediation and retest | Remediation of identified issues and a retest to confirm closure |
| Formal security sign-off | Documented assurance artefact accepted by the service owner / NHS England security |
| External-exposure threat coverage | Testing of the attack surface that only becomes relevant once external suppliers have direct access |

---

## What is retained for MVP

| Retained capability | Detail |
|---|---|
| Application-restricted authentication | All access requires a valid signed-JWT bearer token via the NHS England API Platform. No unauthenticated access. |
| ODS + Product ID ownership enforcement | Every write is checked against the owning organisation and product. |
| ODS spoofing protection | `NHSD-End-User-Organisation-ODS` header cross-checked against token claims. |
| mTLS | Between Apigee and AWS API Gateway, and between the BaRS Proxy and receivers. |
| Encryption | At rest (DynamoDB, S3) and in transit (TLS 1.2+). |
| Least-privilege IAM | Lambda execution roles scoped to required resources; Secrets Manager for credentials. |
| Automated security scanning | Build/test/security scan stage in the GitHub Actions CI/CD pipeline on every change. |
| Audit trail | Every write attributed to client_id + ODS + timestamp; gateway audit to Splunk. |
| Internal-only access | No external supplier or third-party access in MVP — writes go through the R&M team. |

---

## Why this can be deferred

### The MVP is internal-only

The single most important control is that the MVP is **not exposed to external consumers.**
All access is limited to trusted internal parties — the BaRS Proxy (runtime lookups) and
the R&M team (via the CSV pipeline and internal tooling). Endpoint suppliers do **not** have
direct API access in MVP; that is gated behind RBAC/CIS2, which is itself deferred. This
sharply limits the exploitable attack surface compared with an internet-facing,
supplier-facing service.

### Secure-by-design controls are in place from day one

The EPC is not relying on a pen test to *introduce* security. Authentication, ownership
enforcement, spoofing protection, mTLS, encryption, and least-privilege IAM are all built
in for MVP. A penetration test validates and hardens these controls — it is assurance, not
the primary control.

### Automated scanning provides continuous baseline coverage

The CI/CD pipeline runs a security scan on every change, catching common issues
(dependency vulnerabilities, misconfigurations) continuously rather than at a single
point in time. This is not a substitute for a full pen test, but it maintains a baseline
between now and the formal test.

### The data is configuration, not clinical

The EPC holds **service routing configuration** (endpoints, templates, healthcare
services) — not patient or clinical data. The confidentiality impact of the data itself is
low; endpoint addresses are the URLs senders already connect to, not secrets.

---

## Implications of deferral

#### 1. Unvalidated attack surface

**Impact: Medium**

Without an independent test, the security posture is assured by design and internal review
only — it has not been probed by an adversarial third party.

| With full pen test | Without (MVP) |
|---|---|
| Independently validated, findings remediated and signed off | Secure-by-design + automated scanning; not externally validated |

**Mitigation:** Internal-only access limits exposure. Automated scanning and code review
cover common issues. The full test is completed before any external exposure.

---

#### 2. No formal security sign-off for external exposure

**Impact: High (for external rollout only)**

An internet-facing, supplier-facing service requires a formal security assurance artefact.
Without it, the EPC **cannot** be opened to external consumers.

| With full pen test | Without (MVP) |
|---|---|
| Formal assurance supports external rollout | Rollout to external consumers is blocked until testing is complete |

**Mitigation:** This is precisely why the pen test is a hard gate for full rollout, not for
MVP. The internal MVP proceeds; external rollout does not until the test is passed.

---

#### 3. Late discovery risk

**Impact: Medium**

Deferring the test means any architectural security issue is found later, when it may be
more costly to remediate.

**Mitigation:** Secure-by-design and automated scanning reduce the likelihood of a
significant architectural finding. Scheduling the test early in the path-to-live (before
external onboarding begins) keeps remediation cost contained.

---

## What the MVP still guarantees

Despite deferring the full pen test, the MVP is not unprotected:

| Protection | How |
|---|---|
| No unauthenticated access | Signed-JWT bearer token required on every call |
| No cross-organisation tampering | ODS + Product ID ownership enforced on all writes |
| No ODS header spoofing | Header cross-checked against token claims |
| Encrypted everywhere | TLS 1.2+ in transit; DynamoDB and S3 encrypted at rest |
| Least-privilege access | Scoped IAM roles; Secrets Manager for credentials |
| Continuous scanning | Security scan stage in CI/CD on every change |
| Limited blast radius | Internal-only consumers; no external supplier access |
| Full traceability | Application-level audit + gateway audit to Splunk |

---

## Decision

| Decision | Rationale |
|---|---|
| **Defer the full penetration test from MVP; retain secure-by-design controls + automated scanning** | The MVP is internal-only, with no external supplier or third-party access, so the exploitable attack surface is limited. Authentication, ownership enforcement, spoofing protection, mTLS, and encryption are all in place from day one. A full penetration test is mandatory before full rollout and must be passed, with findings remediated, before any external consumer is granted access. |

> ⚠️ **Hard gate for full rollout:** External rollout **must not** proceed until an
> independent full-scope penetration test has been completed, findings remediated, a retest
> confirms closure, and a formal security sign-off is obtained.

---

## When to deliver

The full penetration test should be delivered when:

1. **Before external exposure** — ahead of granting any external supplier or third-party
   direct access to the EPC API (i.e. alongside RBAC/CIS2 delivery)
2. **Production readiness review** — as part of promoting the EPC from internal MVP to full
   production
3. **After significant architectural change** — any material change to the auth model,
   network boundaries, or externally reachable surface should trigger a (re)test
4. **On the cadence required by NHS England security assurance** — including periodic
   retesting once live

---

## Summary: MVP vs final security-testing posture

| Aspect | MVP | Final |
|---|---|---|
| Access scope | Internal-only (BaRS Proxy, R&M team) | External suppliers + internal, via RBAC/CIS2 |
| Security controls | Secure-by-design (authN, ownership, mTLS, encryption) | Same + validated by independent test |
| Automated scanning | CI/CD security scan on every change | Same |
| Independent penetration test | Deferred | Completed, findings remediated, retested |
| Formal security sign-off | Not required for internal MVP | Required before external rollout |
| Retest cadence | N/A | Periodic + on significant change |

---

## Related documents

| Document | Description |
|----------|-------------|
| [Authentication and Authorisation](../authorisation.md) | Auth model, ownership, and ODS spoofing protection |
| [EPC MVP — Scope and Architecture](./README.md) | MVP scope, internal-only access model, deferrals |
| [Role-Based Access Control (RBAC) deferral](./mvp-deferral-rbac.md) | External supplier access gate — related prerequisite for rollout |

---

## Other MVP deferrals in this series

| # | Deferral | Document |
|---|----------|----------|
| 1 | Role-Based Access Control (RBAC) | [mvp-deferral-rbac.md](./mvp-deferral-rbac.md) |
| 2 | Observability (ODIN) | [mvp-deferral-observability.md](./mvp-deferral-observability.md) |
| 3 | Endpoint Ordering (List) | [mvp-deferral-endpoint-ordering.md](./mvp-deferral-endpoint-ordering.md) |
| 4 | Disaster Recovery (Full DR Plan) | [mvp-deferral-disaster-recovery.md](./mvp-deferral-disaster-recovery.md) |
| 5 | Full Penetration Test | This document |
