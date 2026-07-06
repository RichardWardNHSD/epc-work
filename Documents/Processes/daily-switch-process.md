# Daily Switch Process

## Overview

The daily switch process is the primary operational activity performed by the Run & Maintain
(R&M) team. It handles the movement of pharmacies (and other services) between BaRS-capable
system suppliers — for example, a pharmacy moving from PharmOutcomes to Cegedim, or vice
versa.

Today this process is manual, spreadsheet-driven, and reliant on the R&M team executing
changes through the EPC Tool (a standalone web application) and committing the resulting
`targets.json` file to the `NHSDigital/booking-and-referral-targets` GitHub repository.
With the new Endpoint Catalogue (EPC API), the process shifts from this tool-and-GitHub
workflow to a structured, auditable, API-driven operation.

---

## Current Process (EPC Tool + GitHub)

### What is the EPC Tool?

The EPC Tool is a standalone web application (hosted on AWS App Runner) used to manage
Organisations, Healthcare Services, and Endpoints for BaRS. It facilitates and simplifies
the daily management of endpoint switches created by the Pharmacy First programme, along
with all other BaRS endpoints.

Key points:
- The EPC Tool is **not directly linked** to the Endpoint Catalogue — it is purely a
  management interface
- The actual Endpoint Catalogue is a `targets.json` file in the GitHub repository
  `NHSDigital/booking-and-referral-targets`
- The EPC Tool must be used as the source of all updates — direct edits to the GitHub repo
  risk desynchronisation and accidental endpoint deletion
- The BaRS Proxy (FHIR API) reads from the GitHub repo to route requests

### Pre-requisites

The R&M team needs:
- Access to the EPC Tool
- Access to the **MASTER - CPCS IT Supplier Submissions Log** (SharePoint spreadsheet)
- Access to a team member who is an approver for the `NHSDigital/booking-and-referral-targets`
  repository in GitHub

### Trigger

Switch requests arrive via the **MASTER - CPCS IT Supplier Submissions Log** — a shared
Excel spreadsheet on SharePoint maintained by the Pharmacy First programme. Entries come
from suppliers, commissioners, or NHS England regional teams.

### Why daily checks are needed

- There is continuous implementation of BaRS Application 5 in the pharmacy sector
- Pharmacies switch their system suppliers frequently
- Once a week a full comparison check is performed against Directory of Services (DoS) data
  to ensure the Endpoint Catalogue remains synchronised

---

### Daily Switches — Step by Step

```
┌─────────────────────────────────────────────────────────────────┐
│  CURRENT DAILY SWITCH PROCESS (EPC Tool)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Open the MASTER - CPCS IT Supplier Submissions Log          │
│     └── Filter: 'Completed Switch Date' = today                 │
│     └── Copy all populated rows (including header row)          │
│                                                                 │
│  2. Open the EPC Tool                                           │
│     └── Log in (https://vmnfkm94pj.eu-west-2.awsapprunner.com) │
│     └── Select 'Daily Switches' from the left navigation       │
│     └── Paste the copied Excel cells into the text box          │
│                                                                 │
│  3. Review and execute                                          │
│     └── Review the 'Status' column to confirm processing        │
│     └── Click 'Perform changes'                                 │
│     └── Green text confirms: "Switched N Endpoints"             │
│                                                                 │
│  4. Generate and commit targets.json                            │
│     └── Click 'Targets.json' in the EPC Tool navigation        │
│     └── Copy the rendered JSON                                  │
│     └── In GitHub: create a new branch on                       │
│         NHSDigital/booking-and-referral-targets                 │
│     └── Paste the JSON into                                     │
│         targets/prod/targets.json                               │
│     └── Commit the change                                       │
│     └── Create a Pull Request                                   │
│                                                                 │
│  5. Notify and get approval                                     │
│     └── Post to the EPC Changes Teams chat:                     │
│         "N switches, M new — PR: [url]"                         │
│     └── Approver reviews the PR (cross-references against       │
│         the Master CPCS spreadsheet)                            │
│     └── Approver merges into production                         │
│     └── Approver tests the BaRS Proxy remains functional        │
│         (Postman or Splunk — 200 responses)                     │
│                                                                 │
│  6. Update the spreadsheet                                      │
│     └── Add today's date to the "BaRS EPC Update" column        │
│         for each processed service                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Weekly Maintenance (DoS Reconciliation)

In addition to daily switches, a weekly reconciliation is performed (currently after
Thursday's daily process):

1. In the EPC Tool, select **Reconciliation** → expand "1. Create report for DOS team"
2. Click **Create**, then **Download CSV**
3. Email the CSV to `england.dos@nhs.net` and `england.dosmduec@nhs.net`
4. On Friday, the DoS team returns a reconciliation spreadsheet identifying discrepancies
5. Copy all rows (including header) from the returned spreadsheet
6. In the EPC Tool, navigate to "2. Process DOS report" → paste and click **Process**
7. The resulting changes are included in that day's `targets.json` commit

### Approval Process

Any changes to the Endpoint Catalogue require an approver to:
- Collate data sources (Master CPCS spreadsheet, DoS reconciliation, any additional emails)
- In GitHub, open the Pull Request and navigate to "Files changed"
- Cross-reference the number of changes against expected counts from the spreadsheet
- Spot-check individual changes (DoS Service ID → correct supplier endpoint)
- If correct: approve, merge to production, and verify the BaRS Proxy is still functional
- If incorrect: comment on the PR and liaise with the person who made the changes

### Error Handling

| Scenario | Action |
|----------|--------|
| Missing/incorrect data in the Master spreadsheet (identified before processing) | Exclude the record, email `england.dos@nhs.net`, don't mark as updated |
| Missing/incorrect data (identified during processing — EPC Tool shows error) | Email `england.dos@nhs.net`, don't mark as updated |
| Structural paste error | Clear the field, re-copy from the Master spreadsheet, re-paste, re-process |
| Record error but data is correct | Use Supplier Endpoint / Service Provider Endpoint / Audit to investigate; escalate to BaRS team |
| Unexpected EPC Tool error | Seek support from BaRS team members |

### Non-pharmacy endpoints

Endpoints for services other than pharmacy are updated less frequently and arrive through
a different data flow. For small numbers, use the Service Provider Endpoint or Supplier
Endpoint function in the EPC Tool manually. For larger volumes, structure the data using
the Master CPCS header format and process as daily switches.

---

## Pain Points with the Current Process

| Issue | Impact |
|-------|--------|
| **Multi-step manual workflow** | Copy from spreadsheet → paste into tool → click process → copy JSON → create branch → paste JSON → commit → create PR → notify → get approval → merge → test → update spreadsheet. ~15 discrete manual steps per batch. |
| **Spreadsheet as source of truth** | The MASTER - CPCS IT Supplier Submissions Log is a shared Excel file with no validation, no API access, and audit limited to cell-level edit history. |
| **EPC Tool is disconnected from the actual catalogue** | The tool is not linked to the GitHub repo — it generates JSON that a human must manually copy and commit. If they diverge, endpoints can be accidentally deleted. |
| **GitHub-as-a-database** | The production endpoint catalogue is a flat JSON file in a Git repo. Every change requires a branch, a commit, a PR, a review, a merge, and a post-merge smoke test. |
| **No scheduling** | Switches cannot be pre-staged. They must be manually executed on the day. If the team is unavailable (bank holidays, sickness), switches are delayed. |
| **No automation** | There is no API to trigger switches programmatically. Every step requires human interaction with a web UI or GitHub. |
| **Single point of failure** | Only the R&M team can execute switches. No self-service for suppliers. |
| **No rollback** | If a switch is applied incorrectly, reversing it requires re-running the entire process with corrected data. |
| **Weekend/out-of-hours gaps** | Switches requested for weekends or bank holidays must wait or require out-of-hours support. |
| **Manual approval is a bottleneck** | Every change — even a single pharmacy switch — requires a PR review, merge, and smoke test. |

---

## New Process (With EPC API)

### What changes

The new EPC replaces the entire EPC Tool + GitHub workflow with a FHIR R4 API. The
"switch" becomes a structured API operation: update the `endpoint[]` reference on the
HealthcareService from the old supplier's Endpoint to the new supplier's Endpoint.

There is no more:
- Manual JSON generation
- GitHub branches, commits, or Pull Requests
- Copy-paste between spreadsheets and web tools
- Post-merge smoke testing

The BaRS Proxy reads directly from the EPC API — changes take effect immediately on
successful API response.

See [Managing Endpoints — Pharmacy Endpoint Switching](./manage-endpoint.md#pharmacy-endpoint-switching-supplier-switch)
for the full technical process.

### New daily workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  NEW DAILY SWITCH PROCESS (EPC API)                             │
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
│  5. (Future) Eliminate the spreadsheet entirely                  │
│     └── The EPC API becomes the system of record for switch     │
│         state — no manual spreadsheet update needed             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What's better

| Aspect | Before (EPC Tool + GitHub) | After (EPC API) |
|--------|---------------------------|-----------------|
| **Steps per batch** | ~15 manual steps across 4 systems | Upload CSV → review report (2 steps) |
| **Execution** | Copy/paste → tool → copy JSON → GitHub branch → commit → PR → review → merge → test | Batch CSV upload → pipeline processes all rows via API |
| **Change propagation** | Only takes effect after PR merge and Proxy cache refresh | Immediate — API write is the change |
| **Scheduling** | Must be done on the day | Can use `period.start` / `period.end` to pre-stage switches days or weeks ahead |
| **Audit** | Spreadsheet edit history + Git commit log (disconnected) | Full FHIR resource versioning; every change has timestamp, actor, and reason |
| **Validation** | Human reviews a preview table in the EPC Tool | API validates Product IDs, Service IDs, and Endpoint existence before applying |
| **Rollback** | Re-run the entire process with corrected data | Revert the HealthcareService `endpoint[]` to the previous reference |
| **Out-of-hours** | Requires human presence | Pre-staged switches execute automatically when `period.start` arrives |
| **Supplier self-service** | Not possible | Suppliers can manage their own Endpoint status via delegated authority |
| **Error handling** | Errors discovered post-merge by users reporting broken routing | Pipeline reports failures immediately; conflicts detected before changes are applied |
| **Approval** | Every change requires a GitHub PR review | Built-in RBAC; routine switches are self-approving within policy |
| **Weekly reconciliation** | Manual CSV exchange with DoS team + re-process | EPC API is the single source of truth; reconciliation is automated comparison |

---

## Pre-Staged Switches (Eliminating the Daily Manual Step)

The EPC API's `period` mechanism allows switches to be scheduled in advance, removing the
need for daily manual execution entirely.

### How it works

1. When a switch request is received (e.g., "switch pharmacy FH123 to PharmOutcomes on 1 July"):
   - Set `period.end` = `2026-06-30T23:59:59+00:00` on the current Endpoint
   - Create a new Endpoint (child of the new supplier's Template) with `period.start` = `2026-07-01T00:00:00+00:00`
   - Associate both Endpoints with the HealthcareService

2. Before the switch date: consumers querying the service see the old supplier's Endpoint (its period is still valid).

3. On the switch date: the EPC API's period-based visibility filtering automatically shows the new Endpoint and hides the old one. **No human intervention is needed on the day.**

### Impact on the R&M team

| Current | Future |
|---------|--------|
| Must process switches every working day (Mon–Fri) | Process switch requests when they arrive — execution is automatic |
| Bank holidays and weekends are a gap | Switches execute automatically regardless of calendar |
| Team capacity is a bottleneck | Switch volume is decoupled from team availability |
| Reactive (process today's batch) | Proactive (stage next week's switches today) |

---

## Transition Period

During the transition from the current process to the EPC API, both systems will operate
in parallel:

| Phase | Master Spreadsheet | EPC Tool + GitHub | EPC API |
|-------|-------------------|-------------------|---------|
| **Phase 1 — Shadow** | Primary source | Switches executed here | Switches mirrored here for validation |
| **Phase 2 — Dual-run** | Primary source | Fallback only | Switches executed here; GitHub updated as secondary |
| **Phase 3 — EPC API primary** | Archived / read-only | Decommissioned | All switches via EPC API; no manual GitHub changes |

### What the R&M team needs for transition

- Training on the CSV format and S3 upload process
- Access to the processing report (success/failure per row)
- Escalation path for pipeline failures
- Confidence that pre-staged switches work correctly (validated in INT/sandbox first)
- A clear cutover date after which the EPC Tool and GitHub repo are no longer the source
  of truth

---

## Supplier Self-Service (Future State)

Once delegated authority is fully operational, suppliers can manage their own Endpoint
lifecycle without R&M intervention:

| Action | Who does it today | Who does it with EPC API |
|--------|-------------------|--------------------------|
| Create a new Endpoint (new product version) | R&M team (via EPC Tool) | Supplier (via API with their Product ID credentials) |
| Update Endpoint status (suspend/restore) | R&M team | Supplier |
| Report readiness for a switch | Supplier tells R&M via email/spreadsheet | Supplier activates their Endpoint; R&M (or automation) updates the HealthcareService reference |

This reduces the R&M team's daily workload from "execute every switch manually" to
"monitor and handle exceptions". The routine work is automated or delegated; the team
focuses on governance, escalations, and onboarding new suppliers/services.

---

## Summary

| | Current (EPC Tool + GitHub) | EPC API (Day 1) | EPC API (Future) |
|-|----------------------------|-----------------|------------------|
| **Source of truth** | Master spreadsheet + GitHub `targets.json` (disconnected) | EPC API (FHIR resources) | EPC API |
| **Execution** | Manual 15-step process across 4 systems | Batch CSV → pipeline | Pre-staged + supplier self-service |
| **Change propagation** | After PR merge + cache refresh | Immediate on API write | Immediate |
| **Scheduling** | Same-day only | Same-day or pre-staged | Fully pre-staged |
| **Audit** | Spreadsheet + Git commits (disconnected) | FHIR resource versions | FHIR resource versions + event log |
| **Rollback** | Re-run entire process | Revert endpoint[] reference | Automatic (period-based expiry) |
| **R&M effort per switch** | ~5–10 minutes (including PR cycle) | ~30 seconds (prepare CSV row) | Zero (automated) |
| **Weekend coverage** | Gap or out-of-hours support | Pre-staged switches execute automatically | Fully automated |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [Managing Endpoints](./manage-endpoint.md) | Full Endpoint lifecycle including the supplier switch process |
| [Managing HealthcareServices](./manage-healthcare-service.md) | HealthcareService creation, update, and Endpoint association |
| [Interim Support Process Access](./interim-support-process-access.md) | How the R&M team accesses the EPC during the transition |
