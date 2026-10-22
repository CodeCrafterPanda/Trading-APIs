# Performance targets and measurement contract

All numbers below are **proposed acceptance targets**. No implementation benchmark or live broker latency measurement has been performed. General-purpose operating systems, internet paths, brokers and exchanges do not provide a finite worst-case execution guarantee.

## Latency paths

| Path | p50 target | p95 target | p99 target | Maximum/overrun policy |
|---|---:|---:|---:|---|
| Local frame receive → normalized observation | 0.10 ms | 0.30 ms | 0.75 ms | Record observed max; alert queue age >5 ms |
| Local frame receive → candidate decision | 1 ms | 3 ms | 8 ms | Record max; initial processing-age TTL 25 ms |
| Account risk evaluation, excluding durability | 0.10 ms | 0.40 ms | 1 ms | Never skip checks on overrun |
| Durable intent commit round trip | 1 ms | 3 ms | 5 ms | Report topology; slow commit delays or expires intent |
| Eligible decision → request write, including commit | 3 ms | 8 ms | 15 ms | Initial 50 ms pre-send TTL where strategy permits |
| Server state → visible UI | 100 ms | 250 ms | 500 ms | Record max and reconnect gap |

Processing-age TTL is not market-data freshness. A chain refreshed every seconds or a completed 15-minute candle remains an older input even if processed in 1 ms. Strategies specify feature-specific freshness; report both input age and compute latency.

Broker ACK latency starts at request write and ends at received broker response; exchange fill latency ends at verified fill observation. Report them separately with sample size, reject/timeout rates and market context. Broker-supplied processing latency is another measurement, not end-to-end exchange latency. Never add p99 values of independent stages to claim end-to-end p99.

## Critical path and asynchronous work

Synchronous market decision path: decode/normalize, mapping lookup, state update, changed feature nodes, relevant scanner, context snapshot validation and score. Synchronous execution path: account checks/reservation, durable intent transaction, ownership/freshness/session recheck, rate admission and request write. No historical fetch, full-universe sort, analytical query or browser publish blocks these paths.

Asynchronous: raw/analytic persistence, audit export, dashboard fanout, historical warmup/repair, sector catalog imports, telemetry export and reports. Account fill durability is operationally critical and receives dedicated resources; it is not lumped into disposable analytics batching.

## Measurement instrumentation

Capture receipt, decode start/end, queue enqueue/dequeue, state apply, feature/scanner start/end, decision, risk start/end, commit start/end, send, ACK and fill receipt. Use monotonic clocks for local durations with clock_domain and host_boot_id. UTC timestamps correlate systems; compare cross-host times only with measured synchronization uncertainty. Never compare monotonic stamps across reboot/standby.

Persist intent UTC expiry and reconstruct runtime deadlines after restart only under healthy clock bounds. Proposed clock-health gate: warn when wall-clock offset estimate exceeds 5 ms, block short-lived automated entries when uncertainty exceeds their remaining TTL or configured 20 ms maximum. These are engineering defaults requiring infrastructure validation; they do not assert exchange timestamp precision.

Use histograms with suitable sub-millisecond buckets and record sample count, warmup exclusion, observed max, p50/p95/p99, queue age, coalesced/lost events, CPU/core utilization, RSS, allocation rate, disk latency and network errors. Detailed traces are sampled; all safety counters remain unsampled.

## Benchmark envelope

Each run records host CPU/model/cores, memory, OS/kernel, runtime/compiler/dependency hashes, CPU affinity, storage/replication topology, instrument count, mode/depth mix, event sizes, event-rate distribution, chain cadence, active scanners, UI clients and persistence load. Test instrument counts are workload sizes, not product limits.

Begin with generated protocol fixtures and synthetic event bursts. These test software behavior and throughput without creating a paper-trading or historical trading-replay subsystem. After authorized read-only commissioning, measure entitled live-feed peaks and run a sustained workload at twice that peak with a declared burst pattern. Report sustainable envelope and saturation point rather than extrapolating unlimited capacity.

Test steady load for 30 minutes after warmup, a 5-minute 2× burst, reconnect storm, malformed frames, high-cardinality instrument changes, maximum certified depth, slow analytical storage, slow ledger, CPU contention, token rotation and stalled browsers. Exact durations can be extended for soak certification; no latency claim is accepted from a tiny median-only sample.

## Optimization order

First eliminate unnecessary work: dependency DAG, shared features, incremental sector sums/ranks, no per-tick DB call, no repeated JSON conversion and no task spawn per tick. Next reduce allocation: reusable decode buffers, compact typed events, preallocated rings/windows and bounded maps. Then isolate CPU/I/O and tune batching without violating freshness. Only after profiling consider custom lock-free structures, core pinning or transport changes.

Batching has a maximum delay as well as a size. Analytical batches can wait 100 ms initially; order commands cannot wait for an analytics batch. UI display coalesces freely within view cadence after analytical extraction; candle-relevant observations cannot silently disappear.

## Overload ladder and acceptance

At rising queue age, lower UI cadence, shed optional depth/expensive scanner work, reduce discretionary subscriptions with explicit coverage loss, then halt affected automated entries. Preserve account updates and protection capacity. If execution-update memory and durable spool exhaust, mark an audit gap, stop new risk and reconcile; no infinite buffer or silent loss claim.

Pass criteria: target distributions within the stated envelope, bounded memory, no unreported analytical gaps, no lost ledger truth, no bypassed risk, no blind write retry and usable UI. A failure produces a capacity report and tighter admission envelope, not a rewritten “success” threshold. Re-run after changes to decoders, queue ownership, features, database topology or dependency versions; cosmetic documentation changes do not require full performance certification.
