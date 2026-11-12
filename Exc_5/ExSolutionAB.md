# Line Guardian — decision record

## Stage A: why the baseline fails

Measurements are saved in `results/` (20 machines for 90 seconds, then 60 machines for 60 seconds). The small run lets the detector drain; the full run stops the detector shortly after the plant to observe its live backlog.

| Observation | Small plant (200 readings/s) | Full plant (600 readings/s) |
| --- | --- | --- |
| Plant sent / rows stored for this run | 18,000 / 18,000 | 36,000 / 36,000 |
| Detector read when stopped | 15,405 | 16,801 |
| Model's missing counter | 2,585 | 50 |
| Last periodic backlog / delay | 0 / 0.0 s after draining | 18,490 / 30.9 s |
| Anomalies printed/counted | 350 | 0 before stopping |

**Small-run question: did the detector read everything, and why not?** No: 2,595 stored rows were never read despite the final backlog of zero. The model reports only 2,585 missing because it cannot count a gap before the first sequence it sees for a machine (or after the last one). While the plant ran, recent reports showed roughly 0.2–0.3 s delay. The idle report's 0.0 s is hard-coded when no rows are returned; it is not a measured latency percentile. The 350 anomalies are threshold-crossing readings, not 350 separate bearing faults.

**Full-run question: what happens to delay, backlog and missing, and why?** Backlog rose from 9,690 to 18,490 and delay from 16.3 to 30.9 s across the last five periodic reports. One detector's 3 ms inference cost cannot sustain 600/s; actual throughput here was lower still. Missing reached 50 and then stayed flat in those reports: a growing backlog gives earlier transactions more time to commit before the detector reaches their IDs, masking much of the cursor bug without fixing it. No anomalies had been reported before stopping; the delayed detector had not caught up to the fault period. The last periodic backlog is a snapshot before shutdown, not an exact final unread count, and periodic delay samples do not establish p95.

PostgreSQL sequence IDs follow allocation order, not commit order. If gateway 1 inserts IDs 100–114 but commits late, gateway 2 can commit 115–129 first. A poll sees 115–129 and remembers 129; when 100–114 commit, `WHERE id > 129` excludes them forever. Those readings remain in the table. Thus an empty backlog does not prove completeness, and adding pollers does not fix the cursor error. The displayed backlog is an ID difference, not an exact count of unread committed rows.

## Stage B: options and decision

| Requirement | Improve PostgreSQL polling | Send raw readings to Kafka |
| --- | --- | --- |
| R1: 600/s, later 100,000/s | Partition work by machine and add workers, but ingestion, polling and other readers share database capacity. Large scale requires measured database capacity. | Partition the log and scale a detector consumer group; broker, storage and network capacity must also be sized and measured. |
| R2: every reading, in machine order | Replace the unsafe global ID cursor with durable processing/acknowledgement tracking and ordered ownership per machine; simply adding pollers is insufficient. | Key by `machine_id`, with one active detector consumer per partition; preserve producer order and recover model state before resuming. |
| R3: p95 delay at most 5 s | One worker cannot keep up at 600/s. More workers and a correct cursor may help, with database contention still a risk. | Provision inference headroom, monitor event age and lag, and validate p95 at full load. Kafka alone cannot guarantee the target. |
| R4: independent archive/dashboard | Each additional polling consumer adds PostgreSQL load, violating the stated requirement. | Separate consumer groups have independent offsets and can replay without changing detector progress; they still share broker resources. |

**Decision:** use Kafka for raw readings, retaining PostgreSQL as the machine registry. Archive and dashboard use separate groups. Use durable delivery, producer idempotence and retries, and enough retention for outages/replay. Keep a machine on one partition and process that partition sequentially. On failure or reassignment, restore a checkpoint consistent with committed offsets, or replay sufficient history to rebuild the baseline and window before emitting alerts. Deduplicate alerts by `anomaly_id`. Plan partition-count changes carefully because changing key mapping can split a machine's history.

At 3 ms per reading, one sequential detector has an ideal ceiling of about 333 readings/s. The inference-only minimum is `ceil(rate × 0.003)`: **2 instances at 600/s**, **300 at 100,000/s**, with no allowance for overhead or imbalance. At a target 70% utilisation, budget **3 and 429 instances**, respectively, and at least as many partitions for that concurrency. These are capacity estimates, not measured guarantees; benchmark and allow for uneven machine distribution and recovery.

PostgreSQL polling is reasonable for a small office with one temperature reading per minute. The inference-only tipping point for one worker is around 333 readings/s (about 233/s at 70% utilisation); contention, polling latency and reliability requirements can force a change earlier. Here the measured baseline correctness issue and R4 also justify Kafka, independently of throughput.

Optional Stage C is intentionally skipped; no Kafka sink or detector implementation is included, and the 5-second p95 target has not been validated for the proposed design.
