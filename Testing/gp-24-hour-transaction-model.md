# GP service 24 hour transaction test model

## Purpose

Define a repeatable 24-hour performance test for a GP healthcare service, with an average arrival rate of **1.4 transactions per second (TPS)** and a maximum arrival rate of **14 TPS**. The test evaluates performance through quiet periods, a morning peak, normal daytime activity, and evening demand.

The required daily volume is **120,960 transactions**. The model below achieves this exactly and includes a 15-minute period at the maximum rate.

## Requirements and assumptions

| Parameter | Value | Basis |
|---|---|---|
| Measured duration | 24 hours / 86,400 seconds | Required |
| Average arrival rate | 1.4 TPS over the entire measured period | Required |
| Maximum arrival rate | 14 TPS | Required |
| Daily transaction starts | 120,960 | 1.4 × 86,400 |
| Peak to average ratio | 10 to 1 | 14 ÷ 1.4 |
| Peak duration | 15 minutes | Modelling assumption |
| Peak start | 08:00 | Modelling assumption |
| Service availability | Transactions can be submitted throughout the day | Modelling assumption |
| Service scope | Aggregate traffic across the service under test | Modelling assumption; not per GP or per practice |

The traffic profile and transaction mix are illustrative test inputs, not measured GP usage. Replace these assumptions with production evidence when available. Average and maximum TPS alone do not determine a unique daily profile.

Use a fixed 86,400-second test clock. The times below represent service-local time on a normal day; avoid a daylight-saving transition date.

## Transaction definition

A transaction is one business operation, such as searching appointment availability or submitting a repeat prescription request. It starts when the test submits that operation and ends when the expected result is received and validated. A transaction may contain several HTTP or API requests, but counts once in this model.

TPS means **business transaction starts per second**. Record successful completions, failed completions, and outstanding transactions separately. This distinction prevents slow responses from silently reducing the offered workload.

If the supplied TPS figures instead describe individual API requests, use an API request model before executing this schedule. For example, an average of four requests per business transaction would imply approximately 5.6 requests per second at the daily average; actual request demand depends on each operation's implementation.

## Daily workload profile

Each interval includes its start time and excludes its end time. Rates are constant within each interval, with no additional ramps or bursts layered on top.

| Time interval | Duration in seconds | Target TPS | Transaction starts | Scenario |
|---|---:|---:|---:|---|
| 00:00–06:00 | 21,600 | 0.150 | 3,240 | Overnight activity |
| 06:00–08:00 | 7,200 | 0.600 | 4,320 | Early morning activity |
| 08:00–08:15 | 900 | 14.000 | 12,600 | Appointment opening peak |
| 08:15–10:00 | 6,300 | 3.000 | 18,900 | Post-peak activity |
| 10:00–12:00 | 7,200 | 3.400 | 24,480 | Busy late morning |
| 12:00–14:00 | 7,200 | 1.200 | 8,640 | Midday activity |
| 14:00–17:00 | 10,800 | 2.400 | 25,920 | Afternoon activity |
| 17:00–20:00 | 10,800 | 1.600 | 17,280 | Early evening activity |
| 20:00–22:00 | 7,200 | 0.600 | 4,320 | Late evening activity |
| 22:00–24:00 | 7,200 | 0.175 | 1,260 | Overnight transition |
| **Total** | **86,400** | **1.400 weighted average** | **120,960** | |

Validation:

- Interval volume = interval duration in seconds × interval TPS.
- Daily average = 120,960 ÷ 86,400 = **1.4 TPS**.
- Highest scheduled rate = **14 TPS**.
- Peak volume = 900 × 14 = **12,600 transactions**, approximately 10.42% of the daily total.

The rise from 0.6 to 14 TPS at 08:00 is an intentional abrupt peak. The 15-minute duration is a test design choice, not a duration implied by the maximum TPS requirement. Changing it requires rebalancing other intervals to preserve the daily total.

## Example transaction mix

Use the same mix throughout the day for the initial baseline. Peak TPS in this table is a proportional allocation, not an instruction to start fractional transactions in each second.

| Business operation | Share | Daily starts | Average TPS | TPS at peak |
|---|---:|---:|---:|---:|
| Search appointment availability | 30% | 36,288 | 0.420 | 4.200 |
| View appointment details | 20% | 24,192 | 0.280 | 2.800 |
| Book an appointment | 15% | 18,144 | 0.210 | 2.100 |
| View permitted patient record information | 15% | 18,144 | 0.210 | 2.100 |
| Submit a repeat prescription request | 10% | 12,096 | 0.140 | 1.400 |
| Submit an online consultation request | 5% | 6,048 | 0.070 | 0.700 |
| Cancel an appointment | 5% | 6,048 | 0.070 | 0.700 |
| **Total** | **100%** | **120,960** | **1.400** | **14.000** |

Prepare valid prerequisites for each operation: available appointment slots for bookings, existing appointments for cancellations, and appropriately authorised test accounts for record access. An online consultation transaction ends at successful submission; it does not include subsequent clinical review.

Authentication and session refresh requests support these operations and are measured separately rather than counted as additional business transactions. If login is itself in scope as a business transaction, revise the mix so the shares still total 100%.

## Load generation rules

1. **Use an open arrival model.** Schedule transaction starts independently of response completion. A fixed number of looping users alone does not guarantee the specified TPS.
2. **Pace starts evenly.** Use approximately one start every 71.43 milliseconds during the 14 TPS peak. Define the maximum as no more than 14 business starts in any half-open one-second window. Avoid random arrival bursts in this baseline because they can exceed that cap.
3. **Use an exact schedule.** Generate the specified number of starts for every interval. Allocate operation types using daily quotas and a reproducible shuffled order, preserving approximately the same mix within each interval. Integer rounding in individual intervals must reconcile to the exact daily mix totals.
4. **Provide sufficient workers.** Slow responses must not prevent scheduled starts. Estimate in-flight transactions as arrival rate × mean transaction duration, then provide additional worker capacity. At 14 TPS and a two-second mean duration, approximately 28 transactions are in flight; this is not the number of registered or logged-in users.
5. **Do not catch up with bursts.** Record late or missed starts as load-generator deviations. Do not exceed 14 TPS to recover a missed volume. A run with missed starts cannot claim to have delivered the exact specified workload.
6. **Keep retries explicit.** Disable automatic retries for the baseline where possible. Where essential, record their requests separately, retain the original logical transaction identifier, and include retry time in transaction latency. Do not count a retry as a new scheduled business operation.
7. **Keep preparation outside the measurement window.** Seed data, validate scripts, establish sessions, and warm up before 00:00. Stop new starts at 24:00, then allow outstanding transactions to finish under a declared timeout. Report this drain period separately.

## Test environment and data

Use an isolated performance environment with synthetic patient data and test identities. Route notifications and downstream clinical integrations to controlled test endpoints. Record the application version, infrastructure capacity, database size, cache state, dependency configuration, and test script version so results can be reproduced.

Seed enough appointments and eligible records to support the daily write volumes. Prevent unrelated scripts from consuming the same booking slots unless contention is deliberately being tested. Validate business outcomes, including that a successful booking creates one appointment and a cancellation updates the intended appointment.

Represent realistic data sizes and variation within synthetic records. Any simulated dependency latency, unavailable integration, or omitted background job must be documented as a limitation on the result.

## Measurements and proposed acceptance criteria

The workload totals and TPS limits are model requirements. Response-time and reliability thresholds below are initial proposals for agreement with the service owner; they are not supplied service-level commitments.

| Measure | Requirement or proposed threshold |
|---|---|
| Offered workload | Exactly 120,960 scheduled starts over 86,400 seconds, matching each interval's volume |
| Peak delivery | 12,600 starts during 08:00–08:15; no one-second window above 14 starts |
| Read operation latency | Proposed: 95th percentile at or below 2 seconds |
| Write operation latency | Proposed: 95th percentile at or below 3 seconds |
| Unexpected failure rate | Proposed: below 0.1% of starts, assessed for the full run and peak separately |
| Data integrity | No duplicate or incorrect writes attributable to the test execution |
| Peak recovery | Proposed: queue depth and latency return to the agreed normal range within 15 minutes after the peak |
| Sustained operation | No crashes, resource exhaustion, or sustained growth in outstanding work |

Measure latency from operation submission to validated result, including all requests within that operation. Publish percentile results per operation for the full day and peak separately; include the 99th percentile even if no threshold has yet been agreed. Do not allow a daily aggregate to conceal peak failures.

Record starts, successes, failures, timeouts, and outstanding work alongside CPU, memory, database connections, dependency latency, and queue depth. Collect start timestamps at sufficient resolution to verify the one-second cap. Report infrastructure and latency trends at one-minute resolution where practical.

Classify expected business rejections separately from technical errors. In this positive-path baseline, excessive rejections due to exhausted test data make the run unrepresentative even if the service returns technically valid responses.

## Execution and reporting

1. Confirm the transaction boundary, 15-minute peak assumption, service scope, transaction mix, and acceptance thresholds.
2. Validate each operation at low load and verify its resulting data changes.
3. Prepare the environment and test data, complete warm-up, and reset measurement counters.
4. Execute the complete 24-hour schedule with monitoring enabled.
5. Drain outstanding transactions and reconcile every start to a success, failure, timeout, or explicitly unresolved outcome.
6. Report actual versus target volume by interval, latency by operation, peak behaviour, errors, integrity checks, resource trends, and any load-generator deviations.

A valid workload run proves the planned traffic was delivered. A service pass additionally requires the agreed performance and integrity criteria to be met. Stress tests above 14 TPS and alternative peak durations should be separate scenarios so their results do not change this baseline.
