# Line Guardian: questions and answers

This report covers the required Part 2, Stages A and B. Stage C is optional and was skipped. The original Python source files were not changed.

## What did we need to do?

Run the current PostgreSQL-based detector with a small plant and then the full plant. Compare the detector's readings with the database, explain missing readings and growing delay, and choose an architecture against requirements R1–R4. Estimate the number of detector instances needed.

## What did we do?

We ran the supplied baseline detector before starting each plant:

- Small experiment: 20 machines at 10 Hz for 90 seconds, followed by time for the detector to drain.
- Full experiment: 60 machines at 10 Hz for 60 seconds, stopping the detector shortly after the plant.

We saved the plant output, detector output and database counts in this folder. The architecture decision is in [ExSolutionAB.md](../ExSolutionAB.md), with a [PDF summary](../Line-Guardian-summary.pdf).

## Stage A: today's system

### 1. How many readings did the small plant store, and how many did the detector read?

PostgreSQL stored **18,000 readings**, but the detector read only **15,405**. Therefore, **2,595 stored readings were never read**, even though the final reported backlog was zero.

Evidence: [small-table-count.txt](small-table-count.txt), [small-detector.log](small-detector.log), and [small-plant.log](small-plant.log).

### 2. Why does the missing counter grow even though the detector keeps up?

The detector remembers the highest row ID it has read. However, PostgreSQL assigns sequence IDs when rows are inserted, before their transactions commit. Different gateways can commit in a different order from ID allocation.

For example:

1. Gateway 1 inserts IDs 100–114 but has not committed.
2. Gateway 2 inserts IDs 115–129 and commits first.
3. The detector polls between the two commits. It sees 115–129 and remembers **129**.
4. Gateway 1 commits. Its rows now exist, but the detector asks only for `id > 129`, so it never reads 100–114.

The readings are in the table. The polling cursor skips them permanently. Adding more pollers alone does not fix this problem.

### 3. Why is the model's missing counter 2,585 rather than 2,595?

The model detects sequence gaps between readings it receives for a machine. It cannot detect missing readings before the first sequence it sees or after the last. Consequently, the counter does not necessarily equal the database count minus the detector's count.

### 4. Does a backlog of zero prove that every reading was processed?

No. The displayed backlog is the maximum database ID minus the detector's last ID. It does not count all stored readings that were never processed. The small experiment demonstrates this: backlog reached zero while 2,595 rows had been skipped.

### 5. What happened with the full plant?

The plant stored **36,000 readings** at an intended rate of **600 readings/s**. The detector processed **16,801 readings** before stopping.

Across its last five periodic reports:

- Backlog rose from **9,690 to 18,490**.
- Reported delay rose from **16.3 to 30.9 seconds**.
- The missing counter remained at **50**.

Evidence: [full-table-count.txt](full-table-count.txt), [full-detector.log](full-detector.log), and [full-plant.log](full-plant.log).

The last periodic backlog is a snapshot before shutdown, not an exact final unread-row count.

### 6. Why do backlog and delay grow?

Each model call costs 3 ms. A sequential detector can therefore process at most about `1 / 0.003 = 333` readings/s before other overhead. The full plant generates 600 readings/s, so one detector cannot keep up. Readings accumulate and are scored increasingly late.

### 7. Why did the missing counter stop growing in the full run?

The growing backlog gives older transactions more time to commit before the detector reaches their IDs. This explains why the cursor bug is less visible in the full run. It does not make the cursor correct.

### 8. Why must we observe the detector while the plant runs?

Once the plant stops, no new readings arrive and the detector can work through its backlog. Waiting until it drains hides the live system's inability to sustain the incoming rate.

### 9. Did we measure the required p95 latency?

No. Periodic delay reports are not a p95 latency measurement. In addition, the detector prints zero delay when a poll returns no rows, so the small run's final zero is not proof of a latency guarantee. The full run's rising delay demonstrates a capacity problem, but we did not calculate a latency distribution.

## Stage B: architecture decision

### 10. What are requirements R1–R4?

| Requirement | Meaning |
| --- | --- |
| R1: scale | Handle 600 readings/s now and support a design growing to 100,000 readings/s. |
| R2: complete and ordered | Deliver every reading of each machine to its stateful model in sequence order. |
| R3: freshness | At full rate, p95 time from reading to anomaly event must be at most 5 seconds. |
| R4: independent consumers | Archive and dashboard consumers must not disturb the detector or add raw-reading load to PostgreSQL. |

### 11. Could we keep PostgreSQL and improve polling?

Yes, but it requires more than extra pollers. We would need a correct durable processing mechanism instead of the unsafe global ID cursor, machine ownership that preserves order, and enough workers for inference. Database ingestion, polling and additional readers still share database resources. Directly adding archive and dashboard pollers also violates R4's no-extra-PostgreSQL-load condition.

### 12. What does Kafka offer against R1–R4?

| Requirement | Kafka design and remaining risks |
| --- | --- |
| R1 | Partition the readings topic and distribute partitions across detector instances. Broker, network, storage and inference capacity still need sizing and benchmarking. |
| R2 | Use `machine_id` as the key so each machine reaches one partition. Process partitions sequentially, preserve producer order, and recover model state consistently with offsets. |
| R3 | Provision processing headroom and monitor event age and lag. Kafka alone does not guarantee the 5-second p95 target. |
| R4 | Use separate consumer groups for detection, archive and dashboard. Each group has its own offsets; shared broker resources still need sufficient capacity. |

### 13. Which option did we choose, and why?

We recommend **Kafka for raw readings**, with PostgreSQL remaining the system of record for the machine registry. This supports partitioned processing, replay and independent consumers without extra raw-reading queries against PostgreSQL.

The design must use reliable delivery, retries and producer idempotence. On detector failure or partition reassignment, restore model state consistently with committed offsets, or replay enough history to rebuild the model before emitting alerts. Deduplicate alerts using `anomaly_id`. Changing partition counts requires care because it can change the partition chosen for a machine.

### 14. How many detector instances are needed?

The inference-only calculation is:

```text
minimum instances = ceil(readings per second × 0.003 seconds)
```

| Input rate | Theoretical minimum | Estimate at 70% target utilisation |
| --- | --- | --- |
| 600 readings/s | 2 | 3 |
| 100,000 readings/s | 300 | 429 |

The headroom estimate is `ceil(rate × 0.003 / 0.70)`. Provide at least as many partitions as the desired active consumer count. These are capacity estimates, not benchmarked guarantees; overhead, uneven key distribution and recovery can require more resources.

### 15. Is PostgreSQL polling sufficient for one office temperature reading per minute?

Yes. That workload is tiny, so the operational complexity of Kafka may be unnecessary. The processing mechanism must still be correct; a low rate does not fix an unsafe ID cursor.

### 16. Where is the tipping point?

For one detector, inference alone reaches its ideal limit around **333 readings/s**, or about **233 readings/s at 70% utilisation**. Actual capacity is lower with overhead. There is no universal PostgreSQL-to-Kafka threshold: correctness, independent readers, database contention and latency requirements also influence the choice.

## What is finished, and what remains?

- Required Stage A experiments and Stage B decision record are completed locally.
- Optional Stage C was skipped: `KafkaSink` and `detector_service.py` TODOs remain unchanged.
- The proposed Kafka architecture has not been implemented or tested against the 5-second p95 target.
- GitHub push and Moodle submission have not been performed.
