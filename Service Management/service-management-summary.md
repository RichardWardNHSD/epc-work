# EPC Service Management — Summary & Gap Analysis

## Purpose

This document summarises the NHS England Service Management Requirements (Draft V0.2) and
Runbook (Draft V0.2) for the Endpoint Catalogue (EPC), and cross-references them against
the technical design documents already produced for the EPC build.

The goal is to identify:

- What is already addressed by the technical design
- What gaps remain between the requirements and the current build
- What actions are needed to reach operational readiness

> **Authoritative sources:** The `.docx` files in this folder are the formal requirements.
> This markdown summary is a working cross-reference for the delivery team.

---

## 1. Service Overview

The Endpoint Catalogue provides a central repository of endpoint and service metadata,
enabling interoperability across NHS services. Although not user-facing, EPC is
business-critical — failures directly impact service integration and data exchange
(e.g. BaRS Proxy cannot route messages without it).

---

## 2. Support Model

The requirements specify an identical model to BaRS Proxy API support:

| Tier | Role | Responsibility |
|------|------|----------------|
| **1** | National Service Desk (NSD) | Logging, triage, initial engagement |
| **2** | NHS England Service Management / Service Bridge | Coordination, escalation, oversight |
| **3** | Accenture (Supplier) | Technical investigation and resolution |
| **4** | NHS England BaRS Interoperability Team | Technical analysis / support to Tier 3 |

**Support hours:** Core business hours as per SoW; Major Incident support 24×7.

**Contact routes:** NSD (email/telephone) → ServiceNow portal → defined escalation.

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Support model defined | ✅ Interim support process documented (R&M team) |
| Target state (supplier self-service) | ✅ Documented in authorisation & admin UI designs |
| Escalation paths | ⚠️ On-call rota and ITOC handover not yet defined |

---

## 3. Incident Management

### Priority and response

All incidents must be logged in ServiceNow and prioritised by NHS England. Response and
resolution targets follow the BaRS Proxy OLAs.

### Alerting and incident creation rules

This is the most operationally significant section. KPI-based alerting uses three states:

| State | Meaning |
|-------|---------|
| **Normal** | Within acceptable parameters |
| **High** | Threshold breached — degradation detected |
| **Critical** | Severe degradation or failure |

**A ServiceNow Incident is created ONLY when severity increases:**

| Transition | Action |
|---|---|
| Normal → High | Create incident (Runbook Action 2) |
| Normal → Critical | Create incident (Runbook Action 2) |
| High → Critical | Create incident (Runbook Action 2) |

**No incident is created when alert level stays the same or decreases:**

| Transition | Action |
|---|---|
| Normal → Normal, High → High, Critical → Critical | Notification only (Runbook Action 1) |
| Critical → High, Critical → Normal, High → Normal | Notification only (Runbook Action 1) |

### KPI documentation requirements

Each KPI must be documented in Confluence with:

- Unique name (used in ServiceNow short description)
- Full description
- Frequency and timeframe analysed
- Thresholds for Normal / High / Critical
- Supported by a Key Operating Procedure (KOP) for ITOC

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Alerting thresholds defined | ⚠️ Technical alarms exist (5XX rate, latency, etc.) but not yet mapped to the Normal/High/Critical state model |
| KPI unique names defined | ❌ Not yet — need formal KPI naming scheme |
| Confluence documentation of each KPI | ❌ Not yet started |
| State transition logic (only escalate on worsening) | ❌ Not implemented — current alarms fire on threshold breach without state tracking |
| KOP for ITOC | ❌ Not yet written |

**Gap:** The technical design defines CloudWatch/ODIN alerts that fire on threshold
breaches, but the requirements need a **stateful alerting model** where incidents are
only raised on worsening transitions. This likely requires either ODIN alerting rules
with state tracking or custom logic in the alert pipeline.

---

## 4. Runbook Actions

The Runbook defines two actions triggered by KPI state transitions:

### Action 1 — Notification Only (no change / improving)

Triggered when alert state remains the same or improves. Actions:

1. **Teams message** to `[ BaRS ] KPI and ServiceNow alerts` channel containing:
   - Full date/time of alert
   - Alert level prior → alert level attained + KPI name
   - Threshold value / current value

2. **Email** to `England.sm.CellTwo@nhs.net` with:
   - Subject: `{prior level} - {current level}: KPI Episode Update: ITOC - APIM - EPC (BARS) -- {KPI name}`
   - Body: service details, KPI description, current value, links

### Action 2 — Create Incident (worsening)

Triggered when alert severity increases. Actions:

1. **ServiceNow Incident creation** (directly via ODIN if supported, otherwise email to
   `ssd.nationalservicedesk@nhs.net`, cc `England.sm.CellTwo@nhs.net`):
   - Service: BaRS - Booking and Referral Standard
   - Service Offering: BaRS EndPoint Catalogue (EPC)
   - Assignment Group: ITO BaRS Service Management
   - Impact and Urgency: Priority 5
   - Body: service, KPI name, description, current value

2. **Teams message** to `[ BaRS ] KPI and ServiceNow alerts` (same format as Action 1)

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| ODIN integration for ServiceNow creation | ❌ ODIN capabilities for this not yet confirmed |
| Email-based incident creation fallback | ❌ Not implemented |
| Teams channel notifications | ❌ Not implemented |
| Runbook procedures documented | ⚠️ DR runbook exists; operational runbook per these actions does not |

**Gap:** The runbook actions are entirely process/tooling-dependent on ODIN or an
integration layer. If ODIN cannot create ServiceNow incidents directly, an email-based
fallback is needed. Neither path is currently built.

---

## 5. Major Incident Management

Declared when EPC is unavailable or severely degraded, multiple consumers impacted, or
endpoint data issues affect integrations at scale.

- Service Bridge leads coordination
- Bridge calls established as required
- Accenture, BaRS Service Management, and Clinical representation required
- Post Incident Review (PIR) mandatory

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| DR plan supports rapid recovery | ✅ PITR, Lambda rollback, Terraform redeploy documented |
| Monitoring detects major incidents | ✅ Critical alarms (5XX rate, no traffic) defined |
| Bridge call attendance defined | ⚠️ Process — not a technical deliverable |

---

## 6. Change & Release Management

Standard RFC process — identical to BaRS Proxy. All changes require:

- Description, justification, implementation plan
- Risk/impact analysis, backout plan, test plan
- Comms plan (if required), planned start/end date
- CAB/ECAB approval where required

Emergency changes permitted with Service Owner or EDM approval.

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| CI/CD with manual promotion gate | ✅ Documented in DR and deployment design |
| Rollback capability | ✅ Lambda alias swap, API GW stage rollback |
| Infrastructure-as-code | ✅ Terraform |
| Pre-deployment backups | ✅ On-demand DynamoDB backup in pipeline |

---

## 7. Service Level Management

SLAs must cover:

- API availability
- Response times
- Error rates

Continuous monitoring required, with SLA breach analysis in monthly Service Review Packs.

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Availability monitoring | ✅ CloudWatch/ODIN alarms defined |
| Response time monitoring (P90/P95) | ✅ Latency metrics and alarms defined |
| Error rate monitoring | ✅ 4XX/5XX rate metrics defined |
| SLA breach reporting | ❌ No automated SLA breach report exists |
| Monthly reporting pack | ❌ Not a technical deliverable — process TBD |

---

## 8. Monitoring & Observability

### Requirements summary

| Area | Requirement |
|------|-------------|
| Coverage | Availability, error rates, latency, integration health, overall EPC health |
| Dashboards | Key metrics, detailed analysis capability, component-level visibility |
| Alerting | Automated on threshold breach, ServiceNow routing, noise reduction |
| Event handling | Logged, traceable, service-impacting events generate incidents |
| Thresholds | Agreed with NHS England, aligned to service impact, automatic and near real-time |

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Availability monitoring | ✅ |
| Error rate monitoring | ✅ |
| Latency monitoring | ✅ |
| Dashboards (Grafana/CloudWatch) | ✅ Designed (ODIN-first and CloudWatch approaches) |
| Component-level visibility | ✅ Per-Lambda, per-table metrics |
| Metric extraction for NHS England analysis | ⚠️ Depends on ODIN access — TBD |
| Alerting → ServiceNow | ❌ Integration not built (see Runbook gap above) |
| Traceability: alert → incident → root cause | ⚠️ Trace IDs in logs support this, but the ServiceNow linkage is not automated |

---

## 9. Service Continuity & Resilience (BCDR)

### Requirements summary

- Define and maintain RTO and RPO
- DR arrangements documented, tested, aligned to criticality
- Backup and restore in place
- Monitoring integrated with service management for rapid detection and recovery

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| RPO defined (< 5 min) | ✅ Achieved via DynamoDB PITR |
| RTO defined (< 30 min) | ✅ Achievable via PITR restore + redeploy |
| DR documented | ✅ Full disaster-recovery.md |
| DR tested | ❌ First test not yet scheduled |
| Multi-region (if Gold) | ✅ Designed but not enabled — awaiting tier decision |
| Backup & restore | ✅ PITR + on-demand backups |

---

## 10. Security & Audit

### Requirements summary

- Audit logging of endpoint create/update/delete, metadata changes, access/activity
- Logs include timestamp, identity, action, outcome
- Secure storage, access-controlled, protected from modification
- Encrypted at rest and in transit
- Retention per NHS Records Management Policy
- Authorised users can view, search, filter, extract, and correlate log data

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Audit table capturing who/what/when | ✅ `epc-audit` DynamoDB table designed |
| Identity captured (app/user) | ✅ App-restricted: client_id, Product ID, ODS. CIS2: user UUID, role, org |
| Secure storage | ✅ DynamoDB encrypted at rest (AWS KMS) |
| Retention (365 days min) | ✅ Audit log retention set to 365 days |
| Search/filter/extract capability | ⚠️ Available via ODIN/Loki or CloudWatch Insights — no dedicated audit UI |
| Protection from modification | ✅ Write-once pattern in DynamoDB (no update/delete on audit records) |

### Audit gap (interim)

The interim tactical support process (direct AWS access bypassing Apigee) captures only
IAM role identity, not application/user identity. The recommended CSV → Apigee path
resolves this.

---

## 11. Documentation Requirements

Accenture must provide and maintain:

| Area | Content required |
|------|-----------------|
| Observability | Tools, dashboards, metrics, thresholds, alerting behaviour |
| Audit | Logging approach, data structure, retention, access methods |
| Service behaviour | Endpoint lifecycle, filtering logic, expected behaviours |
| Limitations | Known constraints on supportability, integrations, performance |

### Alignment with technical design

| Requirement | Technical design status |
|---|---|
| Observability documentation | ✅ Two detailed observability docs (CloudWatch + ODIN) |
| Audit documentation | ✅ Audit architecture documented |
| Service behaviour | ✅ Endpoint lifecycle, visibility, filtering all documented |
| Limitations documented | ⚠️ Partially — some known constraints noted in open questions |

---

## 12. Key Gaps Summary

| # | Gap | Severity | Notes |
|---|-----|----------|-------|
| 1 | **Stateful KPI alerting model** — current alarms fire on threshold breach, requirements need state transition tracking (only create incidents on worsening) | High | May need ODIN alerting rules or custom state logic |
| 2 | **KPI naming and Confluence documentation** — each KPI needs a unique name, description, thresholds formally documented | Medium | Process task — can start now |
| 3 | **ServiceNow integration** — no automated path from alert to ServiceNow incident creation | High | Depends on ODIN capability or email fallback |
| 4 | **Teams notifications** — runbook requires messages to BaRS KPI channel | Medium | Integration needed (ODIN → Teams or custom webhook) |
| 5 | **ITOC KOP** — Key Operating Procedure for ITOC not yet written | Medium | Process document |
| 6 | **On-call rota and escalation path** — not defined | Medium | Delivery/operational decision |
| 7 | **SLA breach reporting** — no automated report for Service Review Packs | Low | Can be built from ODIN/CloudWatch data |
| 8 | **First DR test** — not yet scheduled | Medium | Schedule quarterly |
| 9 | **ODIN capabilities confirmation** — can ODIN create ServiceNow incidents directly? | High | Blocker for runbook automation |

---

## 13. Recommended Next Steps

1. **Confirm ODIN capabilities** — specifically whether it can:
   - Track alert state transitions (Normal/High/Critical)
   - Create ServiceNow incidents directly on state worsening
   - Send Teams channel notifications
   - If not, define the integration architecture needed

2. **Define KPIs formally** — create a KPI register with:
   - Unique name per KPI (e.g. `EPC-4XX-RECV`, `EPC-5XX-RATE`, `EPC-LATENCY-P90`)
   - Normal/High/Critical thresholds for each
   - Frequency and analysis window
   - Document in Confluence

3. **Write the ITOC KOP** — what ITOC should check, when to escalate, who to contact

4. **Define on-call rota** — who is on call, hours, escalation path

5. **Schedule first DR test** — quarterly cadence, first test before or shortly after go-live

6. **Build runbook automation** — once ODIN capabilities are confirmed, implement the
   Action 1 (notify) and Action 2 (incident + notify) workflows

7. **Agree service tier** — Gold vs Silver classification drives multi-region decision

---

## 14. Requirements vs Technical Design — Full Traceability

| Req Section | Requirement | Design Doc | Status |
|---|---|---|---|
| §4.1 Support Model | Tiered support (NSD → SM → Supplier → BaRS) | interim-support-process-access.md | ✅ |
| §6.3 Alerting Rules | KPI state transitions drive incidents | observability-odin.md | ⚠️ Alarms exist but no state model |
| §6.3 Alerting Rules | KPIs documented in Confluence | — | ❌ |
| §11.1 Monitoring | Availability, errors, latency, integration health | observability.md / observability-odin.md | ✅ |
| §11.2 Observability | Dashboards with analysis capability | observability-odin.md | ✅ |
| §11.3 Alerting | Automated alerts, ServiceNow routing | observability-odin.md | ⚠️ Alerts exist, ServiceNow routing TBD |
| §11.5 Thresholds | Agreed, documented, automatic, near real-time | observability-odin.md | ⚠️ Technical thresholds set, formal agreement TBD |
| §12 BCDR | RPO/RTO defined, DR tested | disaster-recovery.md | ✅ (test pending) |
| §14 Audit | Comprehensive audit logging | epc-audit architecture | ✅ |
| §14.2 Log Storage | Secure, encrypted, retention per policy | observability.md | ✅ |
| §14.3 Log Access | Search, filter, extract, correlate | ODIN/Loki or CloudWatch Insights | ⚠️ |
| §17 Documentation | Observability, audit, behaviour, limitations | Multiple design docs | ✅ |

---

## Document References

| Document | Version | Location |
|----------|---------|----------|
| Service Management Requirements | Draft V0.2 | This folder (.docx) |
| Runbook | Draft V0.2 | This folder (.docx) |
| Observability (CloudWatch) | Current | `../Documents/observability.md` |
| Observability (ODIN-first) | Current | `../Documents/observability-odin.md` |
| Disaster Recovery | Current | `../Documents/disaster-recovery.md` |
| Interim Support Process | Current | `../Documents/interim-support-process-access.md` |
| Audit Architecture | Current | `../Documents/epc-audit-architecture.drawio` |
