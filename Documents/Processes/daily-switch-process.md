# Daily Switch Process

## Overview

The daily switch process is the primary operational activity performed by the Run & Maintain
(R&M) team. It handles the movement of pharmacies (and other services) between BaRS-capable
system suppliers — for example, a pharmacy moving from Sonar Informatics to Pinnacle
(PharmOutcomes), or vice versa.

Today this process is manual, spreadsheet-driven, and reliant on the R&M team executing
changes against the Spine / MAIT tooling. With the new Endpoint Catalogue (EPC), the
process shifts from manual record manipulation to a structured, auditable, and
partially-automatable API-driven workflow.

---

## Current Process (Pre-EPC)

### Trigger

The R&M team receives switch requests via the **Master Switch Log** — a shared spreadsheet
maintained by the BaRS programme. Entries arrive from suppliers, commissioners, or NHS
England regional teams and typically include:

- Pharmacy ODS code
- Pharmacy name
- DoS Service ID
- Current (outgoing) supplier
- New (incoming) supplier
- Requested switch date

### Daily workflow

The R&M team performs the following steps **every working day**:

```
┌─────────────────────────────────────────────────────────────────┐
│  CURRENT DAILY SWITCH PROCESS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Check the Master Switch Log for today's switches            │
│     └── Filter rows where SwitchDate = today                    │
│                                                                 │
│  2. For each switch:                                            │
│     a. Log into the Spine / MAIT admin tooling                  │
│     b. Look up the pharmacy's existing endpoint record          │
│     c. Identify the current supplier's endpoint configuration   │
│     d. Manually update the endpoint record to point to the      │
│        new supplier's address / configuration                   │
│     e. Verify the change has taken effect                       │
│     f. Mark the row as complete in the Master Switch Log        │
│                                                                 │
│  3. Handle failures:                                            │
│     └── If a switch cannot be completed (missing data, system   │
│         error, supplier not ready), escalate or defer to the    │
│         next working day                                        │
│                                                                 │
│  4. Report:                                                     │
│     └── Update the Master Switch Log with outcomes              │
│         (completed / deferred / failed)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pain points with the current process

| Issue | Impact |
|-------|--------|
| **Manual, repetitive work** | Each switch requires the same sequence of clicks/edits in admin tooling. No batch capability. |
| **Spreadsheet as source of truth** | The Master Switch Log is a shared Excel/Google Sheet with no validation, versioning, or audit trail beyond cell-level edit history. |
| **No scheduling** | Switches cannot be pre-staged. They must be manually executed on the day. If the team is unavailable (bank holidays, sickness), switches are delayed. |
| **No audit trail in the system** | The change is made in Spine tooling but the only record of *why* it happened and *who requested it* lives in the spreadsheet. |
| **Error-prone** | Manual lookup and update means typos, wrong ODS codes, or wrong supplier configurations can be applied. Verification is also manual. |
| **Single point of failure** | Only the R&M team can execute switches. No self-service for suppliers. No API for automation. |
| **No rollback** | If a switch is applied incorrectly, reversing it requires another manual intervention — there is no "undo". |
| **Weekend/out-of-hours gaps** | Switches requested for weekends or bank holidays must either wait or require out-of-hours support. |

---

## New Process (With EPC)

### What changes

The EPC replaces the manual Spine/MAIT record manipulation with a FHIR R4 API. The
"switch" becomes a structured operation: update the `endpoint[]` reference on the
HealthcareService from the old supplier's Endpoint to the new supplier's Endpoint.

See [Managing Endpoints — Pharmacy Endpoint Switching](./manage-endpoint.md#pharmacy-endpoint-switching-supplier-switch)
for the full technical process.

### New daily workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  NEW DAILY SWITCH PROCESS (EPC)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Prepare the switch CSV                                      │
│     └── Extract today's switches from the Master Switch Log     │
│         (or receive them via an automated feed)                 │
│     └── CSV columns: ODSCode, ServiceId, OldProductId,          │
│         NewProductId, SwitchDate                                │
│                                                                 │
│  2. Upload the CSV to the processing S3 bucket                  │
│     └── The upload triggers the automated processing pipeline   │
│                                                                 │
│  3. Pipeline executes each switch:                              │
│     a. Locates the HealthcareService by DoS Service ID          │
│     b. Locates the new supplier's Endpoint by Product ID        │
│     c. Updates the HealthcareService endpoint[] reference        │
│     d. Updates the Product ID identifier (delegated authority)  │
│     e. Logs the result (success / failure / conflict)           │
│                                                                 │
│  4. Review the processing report                                │
│     └── Successful switches: no further action                  │
│     └── Failed switches: investigate and re-submit or escalate  │
│                                                                 │
│  5. (Future) Update the Master Switch Log                       │
│     └── Or: eliminate the spreadsheet entirely — the EPC        │
│         becomes the system of record for switch state           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What's better

| Aspect | Before (Manual) | After (EPC) |
|--------|-----------------|-------------|
| **Execution** | Manual clicks in admin tooling, one at a time | Batch CSV upload, pipeline processes all rows |
| **Scheduling** | Must be done on the day | Can use `period.start` / `period.end` to pre-stage switches days or weeks ahead |
| **Audit** | Spreadsheet edit history | Full FHIR resource versioning; every change is a new version with timestamp, actor, and reason |
| **Validation** | None — human must check ODS codes and supplier names visually | API validates Product IDs, Service IDs, and Endpoint existence before applying |
| **Rollback** | Another manual intervention | Revert the HealthcareService `endpoint[]` to the previous reference (old Endpoint still exists, unchanged) |
| **Out-of-hours** | Requires human presence | Pre-staged switches execute automatically when `period.start` arrives |
| **Supplier self-service** | Not possible | Suppliers can (with delegated authority) manage their own Endpoint status — reduces R&M team involvement |
| **Error handling** | Discovered by users reporting broken routing | Pipeline reports failures immediately; conflicts detected before changes are applied |
| **Bulk operations** | One at a time | Entire CSV processed in a single batch; 50 switches take the same effort as 1 |

---

## Pre-Staged Switches (Eliminating the Daily Manual Step)

The EPC's `period` mechanism allows switches to be scheduled in advance, removing the need
for daily manual execution entirely.

### How it works

1. When a switch request is received (e.g., "switch pharmacy FH123 to PharmOutcomes on 1 July"):
   - Set `period.end` = `2026-06-30T23:59:59+00:00` on the current Endpoint
   - Create a new Endpoint (child of the new supplier's Template) with `period.start` = `2026-07-01T00:00:00+00:00`
   - Associate both Endpoints with the HealthcareService

2. At any time before the switch date, consumers querying the service see the old supplier's Endpoint (its period is still valid).

3. On the switch date, the EPC's period-based visibility filtering automatically shows the new Endpoint and hides the old one. **No human intervention is needed on the day.**

### Impact on the R&M team

| Current | Future |
|---------|--------|
| Must process switches every working day | Process switch requests when they arrive (any time) — execution is automatic |
| Bank holidays and weekends are a gap | Switches execute automatically regardless of calendar |
| Team capacity is a bottleneck | Switch volume is decoupled from team availability |
| Reactive (process today's batch) | Proactive (stage next week's switches today) |

---

## Transition Period

During the transition from the current process to EPC, both systems will operate in
parallel:

| Phase | Master Switch Log | Spine/MAIT | EPC |
|-------|-------------------|------------|-----|
| **Phase 1 — Shadow** | Primary source | Switches executed here | Switches mirrored here for validation |
| **Phase 2 — Dual-run** | Primary source | Fallback only | Switches executed here; Spine updated as secondary |
| **Phase 3 — EPC primary** | Archived / read-only | Decommissioned | All switches via EPC; no manual Spine changes |

### What the R&M team needs

- Training on the CSV format and S3 upload process
- Access to the processing report (success/failure per row)
- Escalation path for pipeline failures
- Confidence that pre-staged switches work correctly (validated in INT/sandbox first)
- A clear cutover date after which the Master Switch Log is no longer the source of truth

---

## Supplier Self-Service (Future State)

Once delegated authority is fully operational, suppliers can manage their own Endpoint
lifecycle without R&M intervention:

| Action | Who does it today | Who does it with EPC |
|--------|-------------------|----------------------|
| Create a new Endpoint (new product version) | R&M team | Supplier (via API with their Product ID credentials) |
| Update Endpoint status (suspend/restore) | R&M team | Supplier |
| Report readiness for a switch | Supplier tells R&M via email/spreadsheet | Supplier activates their Endpoint; R&M (or automation) updates the HealthcareService reference |

This reduces the R&M team's daily workload from "execute every switch" to "monitor and
handle exceptions". The routine work is automated or delegated; the team focuses on
governance, escalations, and onboarding new suppliers/services.

---

## Summary

| | Current | EPC (Day 1) | EPC (Future) |
|-|---------|-------------|--------------|
| **Source of truth** | Master Switch Log (spreadsheet) | EPC (FHIR resources) | EPC |
| **Execution** | Manual, one-at-a-time | Batch CSV → pipeline | Pre-staged + supplier self-service |
| **Scheduling** | Same-day only | Same-day or pre-staged | Fully pre-staged |
| **Audit** | Spreadsheet comments | FHIR resource versions | FHIR resource versions + event log |
| **Rollback** | Manual reverse switch | Revert endpoint[] reference | Automatic (period-based expiry) |
| **R&M effort per switch** | ~5–10 minutes | ~30 seconds (prepare CSV row) | Zero (automated) |
| **Weekend coverage** | Gap or out-of-hours support | Pre-staged switches execute automatically | Fully automated |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Managing Endpoints](./manage-endpoint.md) | Full Endpoint lifecycle including the supplier switch process |
| [Managing HealthcareServices](./manage-healthcare-service.md) | HealthcareService creation, update, and Endpoint association |
| [Interim Support Process Access](./interim-support-process-access.md) | How the R&M team accesses the EPC during the transition |
