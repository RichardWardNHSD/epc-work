# GP service 24 hour transaction test model 2

## Purpose

Define a 24-hour diurnal workload with **120,960 business transaction starts**, an overall average of **1.4 TPS**, and short bursts at a maximum of **14 TPS**. This model follows the supplied hourly distribution and complements Model 1, which holds 14 TPS for 15 minutes.

The hourly curve represents the proposed operational pattern for testing. It is not a verified distribution of healthcare demand. Traffic contexts describe scenario assumptions, not additional transaction types or measured clinical behaviour.

## Relationship to Model 1

| Characteristic | Model 1 sustained peak | Model 2 diurnal profile with short bursts |
|---|---|---|
| Daily duration | 24 hours | 24 hours |
| Daily transaction starts | 120,960 | 120,960 |
| Daily average | 1.4 TPS | 1.4 TPS |
| Maximum arrival rate | 14 TPS | 14 TPS |
| Peak timing | 08:00–08:15 | Four short bursts between 09:00 and 11:00 |
| Time at 14 TPS | 900 seconds | 40 seconds |
| Starts during maximum-rate periods | 12,600 | 560 |
| Main test purpose | Sustained peak handling and recovery | Diurnal endurance and response to short bursts |

Run each model as a separate 24-hour test with equivalent starting conditions. Do not combine their schedules or daily totals into a single 24-hour run.

## Requirements and assumptions

- The scope is aggregate business traffic across the service under test, not traffic per practice or per GP.
- The measurement window is exactly 86,400 seconds; use service-local time on a day without a daylight-saving transition.
- All intervals include their start and exclude their end.
- The supplied hourly counts sum to 120,962. Reduce 10:00–11:00 from 12,685 to **12,683** to achieve exactly 120,960; preserve every other hourly count.
- Hourly transaction counts are authoritative. Calculate each exact hourly average as count ÷ 3,600. Displayed TPS values are rounded and must not be used to regenerate counts.
- The timing, number, and ten-second duration of bursts are explicit modelling assumptions. They are not determined by the average and maximum TPS requirements.
- Maintain the same business transaction mix throughout the day for comparability with Model 1. Background jobs, if needed, are separately declared environmental load and do not add business starts to these quotas.

A 10-to-1 peak-to-average ratio is a workload characteristic. It does not establish architecture safety or prove available capacity. TPS is an arrival rate, not concurrency: approximate in-flight work depends on arrival rate multiplied by transaction duration. At 14 TPS and a two-second mean duration, roughly 28 transactions may be in flight. This model defines its maximum over a one-second window; it does not claim to reproduce simultaneous sub-second clicks.

## Transaction definition

A transaction is one business operation, such as searching appointment availability or submitting a repeat prescription request. It starts when the test submits that operation and ends when the expected result is received and validated. A transaction may contain several HTTP or API requests, but counts once in this model.

TPS means **business transaction starts per second**. Record successful completions, failed completions, and outstanding transactions separately. This distinction prevents slow responses from silently reducing the offered workload.

If the supplied TPS figures instead describe individual API requests, use an API request model before executing this schedule. For example, an average of four requests per business transaction would imply approximately 5.6 requests per second at the daily average; actual request demand depends on each operation's implementation.

## Hourly workload profile

The hourly averages include the bursts specified below. Other hours use evenly paced starts.

| Time window | Average TPS rounded | Transaction starts | Assumed traffic context |
|---|---:|---:|---|
| 00:00–01:00 | 0.1889 | 680 | Late-night activity |
| 01:00–02:00 | 0.1258 | 453 | Core overnight quiet period |
| 02:00–03:00 | 0.1006 | 362 | Core overnight quiet period |
| 03:00–04:00 | 0.0631 | 227 | Lowest daily baseline |
| 04:00–05:00 | 0.0631 | 227 | Lowest daily baseline |
| 05:00–06:00 | 0.1511 | 544 | Early morning activity |
| 06:00–07:00 | 0.3775 | 1,359 | Shift handovers begin |
| 07:00–08:00 | 1.0067 | 3,624 | Pre-opening preparation |
| 08:00–09:00 | 2.2653 | 8,155 | Practices open and initial triage |
| 09:00–10:00 | 3.1461 | 11,326 | Morning peak with short bursts |
| 10:00–11:00 | 3.5231 | 12,683 | Busiest hour with short bursts |
| 11:00–12:00 | 3.2719 | 11,779 | Mid-morning steady volume |
| 12:00–13:00 | 2.7686 | 9,967 | Lunchtime reduction |
| 13:00–14:00 | 2.5169 | 9,061 | Early afternoon activity |
| 14:00–15:00 | 2.6428 | 9,514 | Steady booking traffic |
| 15:00–16:00 | 2.8944 | 10,420 | Secondary afternoon rise |
| 16:00–17:00 | 2.3911 | 8,608 | End-of-day administration |
| 17:00–18:00 | 1.7617 | 6,342 | Out-of-hours handover |
| 18:00–19:00 | 1.3842 | 4,983 | Evening triage |
| 19:00–20:00 | 1.0067 | 3,624 | Evening slowdown |
| 20:00–21:00 | 0.7550 | 2,718 | Evening slowdown |
| 21:00–22:00 | 0.5664 | 2,039 | Nighttime activity |
| 22:00–23:00 | 0.3775 | 1,359 | Nighttime activity |
| 23:00–24:00 | 0.2517 | 906 | Closing out the day |
| **Total** | **1.4000** | **120,960** | **24 hours** |

## Short burst schedule

| Burst interval | Duration | TPS | Starts |
|---|---:|---:|---:|
| 09:15:00–09:15:10 | 10 seconds | 14 | 140 |
| 09:45:00–09:45:10 | 10 seconds | 14 | 140 |
| 10:15:00–10:15:10 | 10 seconds | 14 | 140 |
| 10:45:00–10:45:10 | 10 seconds | 14 | 140 |
| **Total** | **40 seconds** | **14 during bursts** | **560** |

These bursts replace background traffic during their intervals. They are not added to an unchanged hourly baseline.

| Hour | Hourly quota | Burst starts | Non-burst seconds | Non-burst starts | Non-burst TPS rounded |
|---|---:|---:|---:|---:|---:|
| 09:00–10:00 | 11,326 | 280 | 3,580 | 11,046 | 3.085475 |
| 10:00–11:00 | 12,683 | 280 | 3,580 | 12,403 | 3.464525 |

For each burst-bearing hour:

`Hourly starts = 14 × 20 + non-burst rate × 3,580`

Use the exact non-burst rate `(hourly quota − 280) ÷ 3,580`, or schedule its integer quota directly. Do not use the rounded display value as the authoritative rate.

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

1. Use an open arrival model so scheduled starts are independent of response completion. Provide enough load-generator workers to maintain the schedule during slow responses.
2. Generate each hour's exact integer quota. For hours 09:00 and 10:00, reserve the four specified burst intervals first, then distribute each hour's remaining starts evenly across its non-burst periods.
3. Enforce a maximum of 14 starts in every half-open rolling one-second window, including burst boundaries. One deterministic implementation uses a global grid of 14 evenly spaced launch slots per second, fills every slot during bursts, and distributes non-burst quotas across the remaining eligible slots using an integer accumulator. At most one transaction may occupy a slot. Verify the resulting schedule before execution and actual timestamps afterwards.
4. Allocate operation types using the exact daily mix quotas and a reproducible shuffled order. Approximate that mix within each hour and burst; reconcile integer rounding across the day rather than demanding fractional operation counts per second.
5. Keep retries visible, with the same logical transaction identifier. Disable automatic retries where possible; otherwise record their request load separately and include retry time in business transaction latency.
6. Record missed or late starts as deviations. Do not use catch-up bursts that exceed the maximum rate. A run with missing starts does not meet the workload requirement.
7. Prepare data, establish sessions, and warm up before the measurement window. At 24:00 stop new starts and drain outstanding operations under a declared timeout; report drain time separately.

## Test environment and data

Use an isolated performance environment with synthetic patient data and test identities. Route notifications and downstream clinical integrations to controlled test endpoints. Record the application version, infrastructure capacity, database size, cache state, dependency configuration, and test script version so results can be reproduced.

Seed enough appointments and eligible records to support the daily write volumes. Prevent unrelated scripts from consuming the same booking slots unless contention is deliberately being tested. Validate business outcomes, including that a successful booking creates one appointment and a cancellation updates the intended appointment.

Represent realistic data sizes and variation within synthetic records. Any simulated dependency latency, unavailable integration, or omitted background job must be documented as a limitation on the result.

## Measurements and proposed acceptance criteria

The workload criteria are fixed by this model. The performance thresholds remain proposals for agreement with the service owner, using the same starting thresholds as Model 1.

| Measure | Requirement or proposed threshold |
|---|---|
| Workload delivery | Exactly 120,960 starts in 86,400 seconds; each hourly quota matches the table |
| Burst delivery | Exactly 140 starts in each of the four ten-second bursts |
| Maximum rate | No more than 14 starts in any half-open rolling one-second window |
| Read latency | Proposed: 95th percentile at or below 2 seconds |
| Write latency | Proposed: 95th percentile at or below 3 seconds |
| Unexpected failures | Proposed: below 0.1% for the full day and the combined burst cohort, reported separately |
| Data integrity | No duplicate or incorrect writes attributable to test execution |
| Burst recovery | Proposed: latency and outstanding work return to the agreed pre-burst range within 60 seconds of each burst ending |
| Endurance | No crashes, resource exhaustion, or sustained growth in outstanding work |

Measure latency from submission to validated completion. Attribute an operation to a burst by its start timestamp, including when it finishes after the burst. Report latency and errors by operation for the full day, each hour, each burst, and the combined burst cohort. Small per-operation burst samples make tail percentiles unstable; include sample counts, raw failure counts, and maximum latency. With only 560 burst starts, a strict below-0.1% failure criterion permits no failures in that cohort.

Capture start timestamps with sub-second resolution. Use one-second throughput and queue observations around bursts where available; one-minute averages alone can hide a ten-second spike. Track CPU, memory, database connections, dependency latency, and outstanding work throughout the run.

Define the normal recovery range from a stable pre-burst window, for example the preceding five minutes, before running the test. Use the same observation window and tolerances for every burst. Document these tolerances with the result.

## Execution and reporting

1. Confirm the service scope, transaction boundaries, assumed burst schedule, transaction mix, and proposed performance thresholds.
2. Validate scripts and seed sufficient synthetic data for all business operations.
3. Verify that the generated schedule has the correct hourly and daily counts and respects the rolling one-second cap.
4. Run the full 24-hour profile, then drain and reconcile all outcomes.
5. Report target versus actual starts by hour, burst counts, latency and failures by operation, recovery time after each burst, data integrity, and infrastructure trends.
6. Compare with Model 1 using the same environment configuration, starting data conditions, and business operation definitions. Explain that the models expose the service to different peak durations despite identical daily volume and maximum TPS.

A valid workload run confirms that the required traffic was delivered. A service pass additionally requires the agreed performance and integrity thresholds. Neither model alone proves clinical safety or capacity beyond the tested workload and environment.
