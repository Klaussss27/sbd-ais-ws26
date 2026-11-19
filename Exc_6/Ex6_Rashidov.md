# Exercise 6 — Fraud detection

## Requirements

| Requirement | What it means |
| --- | --- |
| R1 — Leave the payment system alone | Keep the payment program unchanged. It saves payments without waiting for fraud checks or taking on extra queries from the models. |
| R2 — Give both models every payment | Each model receives every committed payment. Adding another reader leaves both models able to read the full stream. |
| R3 — Catch fraud accurately | Each model catches at least 90% of known fraud cases with few false alerts, even when it runs with multiple workers. |
| R4 — Send alerts quickly | At full payment speed, at least 95% of alerts arrive within 5 seconds of the payment being committed. |

## Part 1 — Decision

I would use log-based change data capture (CDC) so Debezium can forward each committed payment to Kafka. It reads PostgreSQL’s existing transaction log, so the payment program can stay unchanged and the fraud teams do not have to query the payments table. Each model gets its own Kafka consumer group and can independently read the full payment stream. If Debezium stops reading, PostgreSQL’s replication slot retains the unread log, which can eventually fill the disk. I would monitor the amount of retained log data and free disk space and alert when either approaches an unsafe level. If we owned the payment program, I would consider a transactional outbox so the payment and a record to publish are saved together in one database transaction.

## Part 2 — Three runs

| Run | Prediction | Velocity R3 | Velocity R4 |
| --- | --- | --- | --- |
| 1 — Default | I expected one worker to get the only partition and fall behind. | Frauds caught 2/30 (7%) **FAIL**. False alerts 0 **PASS**. | Alert latency p50/p95 39.3 s / 45.1 s **FAIL**. |
| 2 — Six partitions | Three workers will keep up, but payments from the same card may reach different workers. I expect R3 to fail. | Frauds caught 1/15 (7%) **FAIL**. False alerts 0 **PASS**. | Alert latency p50/p95 0.8 s / 0.8 s **PASS**. |
| 3 — Cardholder key | I expected each worker to see all payments for its cardholders, so R3 and R4 would pass. | Frauds caught 63/63 (100%) **PASS**. False alerts 5 **PASS**. | Alert latency p50/p95 0.8 s / 1.1 s **PASS**. |

**Run 1.** Kafka had one partition and used the payment ID (`id`) as the key. Only `vel-1` handled payments, so the queue grew while the other two workers were idle. The p95 alert delay reached 45.1 seconds, above the five-second limit.

**Run 2.** With six partitions, all three velocity workers processed payments and the p95 alert delay dropped to 0.8 seconds. The key was still the payment ID, so payments from the same cardholder could go to different workers. Each worker counted only the payments it saw, and the model caught just 1 of 15 fraud cases.

**Run 3.** The cardholder key sent all payments for each cardholder to one partition, letting a single velocity worker count them together. The model caught all 63 card-testing frauds, with five false alerts. At about 1,000 payments per second, p95 alert delay was 1.1 seconds, and both R3 and R4 passed.

## What would we see in production?

One partition could leave most workers idle and cause alerts to arrive late. Partitioning by payment ID could cause the workers to miss card-testing fraud even when they keep up with incoming payments. I would monitor consumer lag and the time from payment commit to alert, then compare alerts against confirmed fraud to check for missed cases.
