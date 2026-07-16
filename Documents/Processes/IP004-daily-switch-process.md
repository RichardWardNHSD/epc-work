# Daily Switch Process (BaRS)

## Overview

> **Scope:** This document describes the daily pharmacy supplier switch process for
> **BaRS Endpoints only**. It covers both the current (legacy) EPC Tool + GitHub workflow
> and the new EPC API-driven process. The new process uses the same CSV-to-S3-to-Lambda
> pipeline pattern as the other IP documents in this series.

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


### New daily workflow — Step by step

---

### Step 1 — Prepare the Switch CSV

Extract today's switches from the Master Switch Log (or receive them via an automated
feed). Create a CSV file with the following structure.

#### CSV structure

| Column | Required | Description | Provided by | Example |
|--------|----------|-------------|-------------|---------|
| `ODSCode` | **Mandatory** | ODS code of the pharmacy | Master Switch Log | `FQ024` |
| `ServiceId` | **Mandatory** | DoS Service ID (the `Pharm+ DoS Service ID` column) | Master Switch Log | `2000110743` |
| `OldProductId` | **Mandatory** | Product ID of the outgoing supplier's Endpoint Template | Master Switch Log / EPC | `PROD-CEGEDIM-001` |
| `NewProductId` | **Mandatory** | Product ID of the incoming supplier's Endpoint Template | Master Switch Log / EPC | `PROD-PHARMOUTCOMES-001` |
| `SwitchDate` | **Mandatory** | Effective date of the switch | Master Switch Log | `2026-07-07` |

```csv
ODSCode,ServiceId,OldProductId,NewProductId,SwitchDate
FQ024,2000110743,PROD-CEGEDIM-001,PROD-PHARMOUTCOMES-001,2026-07-07
FKV30,2000017778,PROD-POSITIVE-001,PROD-PHARMOUTCOMES-001,2026-07-07
FH123,2000099999,PROD-PHARMOUTCOMES-001,PROD-CEGEDIM-001,2026-07-07
```

> **Naming convention:** `epc-switches-YYYY-MM-DDTHHmmss.csv` (e.g., `epc-switches-2026-07-07T093000.csv`)

The CSV may contain multiple rows — one per pharmacy being switched. Each row is processed
independently.

> **Validation is performed by the Lambda after upload:** The R&M team does not need to
> pre-check that each `ServiceId` or `NewProductId` exists in the EPC. The Lambda validates
> every row during processing — if a `ServiceId` has no matching HealthcareService, or the
> `NewProductId` has no active Endpoint Template, the row is marked as `FAILED` in the
> processing report with a descriptive error. The R&M team then corrects and re-submits
> only the failed rows.

---

### Step 2 — Upload the CSV to S3

Upload the CSV to the designated S3 bucket. This triggers the Lambda processing pipeline
automatically.

#### Using the AWS CLI

```bash
aws s3 cp epc-switches-2026-07-07T093000.csv \
  s3://epc-switch-processing-prod/incoming/epc-switches-2026-07-07T093000.csv
```

#### Using the AWS Console

1. Navigate to S3 → `epc-switch-processing-prod` → `incoming/`
2. Click **Upload**
3. Select the CSV file
4. Click **Upload**

> **What happens on upload:** The S3 `PutObject` event triggers the
> `epc-switch-processor` Lambda function. The Lambda reads the CSV and processes each
> row independently.
>
> **After processing:** The CSV file is moved from the `incoming/` folder to an
> `archive/` folder in the same S3 bucket and retained for 30 days before automatic
> deletion.

---

### Pipeline processing (automated)

> **Note:** Lambda names used in this document (e.g., `epc-switch-processor`)
> are illustrative examples and may not reflect the actual deployed Lambda function names.

The `epc-switch-processor` Lambda function executes the following for **each row** in the
CSV:

---

#### Step 2a — Locate the HealthcareService

The Lambda calls the EPC API to find the pharmacy's HealthcareService using the DoS
Service ID:

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000110743 HTTP/1.1
Host: api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer {lambda-service-token}
X-Request-Id: {auto-generated-uuid}
X-Correlation-Id: {batch-correlation-id}
NHSD-End-User-Organisation-ODS: X26
```

The Lambda extracts:
- The HealthcareService `id` (e.g., `9f2c6f12-1a6d-4d9c-a111-123456789abc`)
- The current `endpoint[]` array (to confirm the old Endpoint is referenced)

#### Step 2b — Locate the new supplier's Endpoint

The Lambda looks up the new supplier's Endpoint using the `NewProductId`:

```http
GET /Endpoint?identifier=https://fhir.nhs.uk/id/product-id|PROD-PHARMOUTCOMES-001&ConnectionType=http://terminology.hl7.org/CodeSystem/endpoint-connection-type|hl7-fhir-rest&PayloadType=http://terminology.hl7.org/CodeSystem/endpoint-payload-type-epc|bars HTTP/1.1
Host: api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer {lambda-service-token}
X-Request-Id: {auto-generated-uuid}
X-Correlation-Id: {batch-correlation-id}
NHSD-End-User-Organisation-ODS: X26
```

The Lambda extracts the new Endpoint `id` (e.g., `ep-new-supplier-001`).

> **If the Endpoint does not exist:** The row is marked as `FAILED — Endpoint not found`
> in the processing report. No changes are made.

#### Step 3 — Update the HealthcareService endpoint reference

The Lambda issues a `PUT /HealthcareService/{id}` to replace the old Endpoint reference
with the new one:

```http
PUT /HealthcareService/9f2c6f12-1a6d-4d9c-a111-123456789abc HTTP/1.1
Host: api.service.nhs.uk
Content-Type: application/fhir+json;version=1.4.0
Accept: application/fhir+json
Authorization: Bearer {lambda-service-token}
X-Request-Id: {auto-generated-uuid}
X-Correlation-Id: {batch-correlation-id}
NHSD-End-User-Organisation-ODS: X26
```

```json
{
  "resourceType": "HealthcareService",
  "id": "9f2c6f12-1a6d-4d9c-a111-123456789abc",
  "meta": {
    "lastUpdated": "2026-07-07T09:00:00+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-HealthcareService"
    ]
  },
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/dos-service-id",
      "value": "2000110743"
    },
    {
      "system": "https://fhir.nhs.uk/id/product-id",
      "value": "PROD-PHARMOUTCOMES-001"
    }
  ],
  "active": true,
  "name": "Shirley Pharmacy",
  "providedBy": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "FQ024"
    }
  },
  "endpoint": [
    {
      "reference": "Endpoint/ep-new-supplier-001"
    }
  ]
}
```

> **Note:** The `identifier[]` array includes both the DoS Service ID and the **new**
> Product ID. This transfers delegated authority to the new supplier in the same atomic
> operation.

#### Pipeline behaviour — Error responses

| API Response | Pipeline Action | Processing Report Entry |
|--------------|-----------------|-------------------------|
| `200 OK` | Switch successful | `SUCCESS` |
| `404 Not Found` (HealthcareService) | Do not proceed | `FAILED` — "Service not found" |
| `404 Not Found` (Endpoint) | Do not proceed | `FAILED` — "Endpoint not found" |
| `409 Conflict` | Do not proceed | `FAILED` — "Conflict (concurrent update)" |
| `4XX` other | Do not proceed | `FAILED` — "{error detail from OperationOutcome}" |
| `5XX Server Error` | Retry up to 3 times with exponential backoff; if still failing, do not proceed | `FAILED` — "Server error after 3 retries" |
| Network timeout | Retry up to 3 times; if still failing, do not proceed | `FAILED` — "Timeout after 3 retries" |

---

### Processing report

After all rows are processed, the Lambda writes a report to S3:

```
s3://epc-switch-processing-prod/reports/epc-switches-2026-07-07T093000-report.csv
```

#### Report CSV structure

```csv
ODSCode,ServiceId,OldProductId,NewProductId,SwitchDate,Status,Detail
FQ024,2000110743,PROD-CEGEDIM-001,PROD-PHARMOUTCOMES-001,2026-07-07,SUCCESS,
FKV30,2000017778,PROD-POSITIVE-001,PROD-PHARMOUTCOMES-001,2026-07-07,SUCCESS,
FH123,2000099999,PROD-PHARMOUTCOMES-001,PROD-CEGEDIM-001,2026-07-07,FAILED,Endpoint not found for PROD-CEGEDIM-001
```

---

#### Status values

| Status | Meaning | Action |
|--------|---------|--------|
| `SUCCESS` | Switch completed successfully — live immediately | No action needed |
| `FAILED` | Error during processing | Investigate, correct, and re-submit |

#### Retrieving the report

```bash
aws s3 cp \
  s3://epc-switch-processing-prod/reports/epc-switches-2026-07-07T093000-report.csv \
  ./epc-switches-2026-07-07T093000-report.csv
```

Or via the AWS Console: S3 → `epc-switch-processing-prod` → `reports/`

#### Action on failed results

| Failure reason | Action |
|----------------|--------|
| `Service not found` | Confirm DoS Service ID is correct; check if HealthcareService needs to be created |
| `Endpoint not found` | Confirm the new supplier's Product ID is correct and their Endpoint Template + Endpoint exist |
| `Conflict` | Investigate concurrent modification; re-submit the row |
| `Server error after 3 retries` | Escalate to development team — likely an infrastructure issue |

#### Re-processing failed rows

Extract the failed rows from the report, correct the data, and upload a new CSV containing
only the corrected rows:

```bash
aws s3 cp epc-switches-2026-07-07T093000-fixes.csv \
  s3://epc-switch-processing-prod/incoming/epc-switches-2026-07-07T093000-fixes.csv
```

---

### Step 3 — Verify (Optional)

For high-confidence verification, the R&M team can query the EPC API directly to confirm
a switch has taken effect:

```http
GET /HealthcareService?identifier=https://fhir.nhs.uk/Id/dos-service-id|2000110743&_include=HealthcareService:endpoint HTTP/1.1
Host: api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer {token}
```

The response should show the HealthcareService with the new supplier's Endpoint in the
`endpoint[]` array. The BaRS Proxy will use this Endpoint immediately for routing.

---

#### Summary: New process at a glance

```
┌─────────────────────────────────────────────────────────────────┐
│  NEW DAILY SWITCH PROCESS (EPC API)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  R&M Team                        AWS Infrastructure             │
│  ─────────                        ──────────────────            │
│                                                                 │
│  1. Create CSV from Master       ─── upload ──►  S3 Bucket      │
│     Switch Log                                   (incoming/)    │
│                                                      │          │
│                                                      ▼          │
│                                              Lambda triggered    │
│                                              (epc-switch-       │
│                                               processor)        │
│                                                      │          │
│                                          For each row:          │
│                                          ┌───────────────────┐  │
│                                          │ GET /HC Service   │  │
│                                          │ GET /Endpoint     │  │
│                                          │ PUT /HC Service   │  │
│                                          └───────────────────┘  │
│                                                      │          │
│                                                      ▼          │
│  4. Download and review          ◄── writes ──  S3 Bucket       │
│     processing report                            (reports/)     │
│                                                                 │
│  5. Re-submit any failures                                      │
│     (corrected CSV)                                             │
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

## Error responses

The EPC API returns FHIR `OperationOutcome` resources for errors:

| HTTP Status | Meaning |
|-------------|---------|
| 200 | Success — resource returned or Bundle with results |
| 400 | Bad request — invalid parameter value or format |
| 401 | Unauthorised — missing or invalid Bearer token |
| 403 | Forbidden — valid token but insufficient permissions |
| 404 | Not found — no HealthcareService or Endpoint with the specified identifier |
| 409 | Conflict — concurrent modification |
| 4XX | Other client error — see OperationOutcome for details |
| 5XX | Server error — see OperationOutcome for details |

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/Codesystem/http-error-codes",
            "code": "RESOURCE_NOT_FOUND"
          }
        ]
      },
      "diagnostics": "No HealthcareService found for identifier https://fhir.nhs.uk/Id/dos-service-id|2000110743"
    }
  ]
}
```

---

## Related documents

| Document | Description |
|----------|-------------|
| [Managing Endpoints (BaRS)](./manage-endpoint.md) | Full Endpoint lifecycle — create, update, delete |
| [Managing Endpoint Templates (BaRS)](./manage-endpoint-template.md) | Creating and managing parent BaRS Templates |
| [Managing HealthcareServices](./manage-healthcare-service.md) | HealthcareService creation, update, and Endpoint association |
| [Interim Support Process Access](./interim-support-process-access.md) | How the R&M team accesses the EPC during the transition |
