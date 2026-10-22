# Performance targets and measurement contract

This document owns every latency number in the pack; other documents quote these values and never define alternates. All numbers are proposed acceptance targets under a declared load envelope (C1 in [00](00-EXECUTIVE-OVERVIEW.md)). The design intent is **sub-millisecond compute**: every in-memory stage on the market critical path carries a sub-millisecond p50 *and* p99 budget. Only durable commit and network I/O are allowed to exceed 1 ms.

## Latency budgets

| Path | p50 | p95 | p99 | Overrun policy |
|---|---:|---:|---:|---|
| Frame receive → normalized observation (decode, validate, map) | 0.05 ms | 0.20 ms | 0.50 ms | Alert when ingress queue age > 2 ms; reconnect on irrecoverable overflow |
| Normalized → instrument state + active bars + dirty feature nodes applied | 0.10 ms | 0.30 ms | 0.70 ms | Per-shard budget; shed optional feature nodes first |
| One sector contribution update + ordered rank-index change | 0.02 ms | 0.10 ms | 0.30 ms | Coalesce publication only, never the sufficient statistics |
| One scanner detector service time, per event | 0.01 ms | 0.05 ms | 0.10 ms | Degrade detector visibly; cannot delay risk path |
| Opportunity rescoring per affected candidate | 0.02 ms | 0.05 ms | 0.10 ms | Expire stale candidates rather than queue |
| **Frame receive → candidate decision (end-to-end compute)** | **0.50 ms** | **2 ms** | **5 ms** | Processing-age TTL 25 ms; older candidates are not auto-executed |
| Account risk evaluation, excluding durability | 0.05 ms | 0.20 ms | 0.50 ms | Never skip a check |
| Durable intent commit round trip (local NVMe, synchronous commit) | 1 ms | 3 ms | 5 ms | Slow commit delays or expires the intent; never bypassed |
| Eligible decision → broker request write, including commit | 2 ms | 5 ms | 10 ms | 50 ms pre-send TTL where the strategy permits |
| Order/execution update → ledger applied + risk revised | 0.20 ms | 0.50 ms | 1 ms | Reserved channel; never dropped |
| Server state → visible UI | 100 ms | 250 ms | 500 ms | Reduce cadence, then disconnect slow consumers with resync |

Processing-age TTL is not market-data freshness: a chain refreshed every few seconds or a completed 15-minute candle remains an older input even when processed in 0.5 ms. Strategies declare feature-specific freshness; telemetry reports input age and compute latency separately.

Broker ACK latency starts at request write and ends at the received broker response; fill latency ends at the verified fill observation. Report both per broker (DHAN, UPSTOX, ZERODHA) with sample size, reject/timeout rate and market phase. Upstox `metadata.latency` and any broker-supplied timing are broker processing measurements, not end-to-end exchange latency. Never add p99 values of independent stages to claim an end-to-end p99.

## Critical path and asynchronous work

Synchronous market path: decode/normalize → canonical mapping lookup (immutable master-version map) → shard state update → changed feature nodes → affected detectors → context snapshot validation → score. Synchronous execution path: account checks and reservation → durable intent transaction → ownership/freshness/session recheck → rate admission → request write. No historical fetch, full-universe sort, analytical query, JSON conversion, task spawn or browser publish occurs on either path.

Asynchronous: raw/analytical persistence, audit export, dashboard fan-out, historical warmup/repair, instrument/sector catalog imports, option-chain REST refresh, local Greeks recomputation for non-selected strikes, telemetry export. Account fill durability is operationally critical and has dedicated resources; it is never batched with analytics.

## Sub-millisecond engineering rules

1. **Zero allocation per tick on the hot path.** Reusable decode buffers, fixed-capacity typed events, preallocated rings/windows, dense per-shard arrays indexed by shard-local instrument index (not hash lookups per tick), bounded maps sized at startup from master counts.
2. **Fixed-point integers** for prices/quantities/turnover on the hot path; floats only inside feature math that tolerates them, never for accounting.
3. **One writer per shard; no locks on the tick path.** Cross-shard communication uses bounded SPSC/MPSC queues carrying immutable snapshots. Sector/regime/rank indexes are updated by their single owner from contribution deltas.
4. **Dependency DAG with refcounts.** Only dirty nodes evaluate; unused indicators are never computed; O(K) chain computations run once per snapshot and fan out cached results.
5. **CPU isolation.** Pin shard workers, adapter decode workers and the account actor to dedicated cores (isolcpus/cpuset); keep Tokio I/O threads on separate cores; disable frequency scaling on trading hosts; huge pages for state arrays after measurement.
6. **No syscalls inside the tick update.** Monotonic timestamps via vDSO; logging through a lock-free ring drained by a separate thread; metrics via per-core counters aggregated off-path.
7. **Batch only where latency is not on the path.** Analytics batches wait up to 100 ms; UI coalesces within view cadence; candle-relevant observations and execution events are never coalesced.

Optimization order after correctness: eliminate work (DAG, incremental sums, no DB/JSON per tick) → remove allocation → isolate CPU/I/O → tune batching → only then custom lock-free structures, kernel-bypass or transport changes, each justified by a profile.

## Measurement instrumentation

Capture monotonic stamps at receipt, decode start/end, queue enqueue/dequeue, state apply, feature/scanner start/end, decision, risk start/end, commit start/end, send, ACK and fill receipt. Local durations use monotonic clocks with `clock_domain` and `host_boot_id`; UTC correlates systems and cross-host comparisons carry the measured synchronization uncertainty. Monotonic stamps are never compared across reboot or standby.

Persist intent `expires_at_utc` and reconstruct runtime deadlines after restart only under healthy clock bounds. Clock-health gate: warn when the wall-clock offset estimate exceeds 5 ms; block short-lived automated entries when uncertainty exceeds their remaining TTL or 20 ms. Use PTP or disciplined NTP on trading hosts; record the source.

Histograms use sub-millisecond buckets (10 µs resolution below 1 ms) and record sample count, warmup exclusion, observed maximum, p50/p95/p99, queue age, coalesced/lost events, CPU per core, RSS, allocation rate, disk fsync latency and network errors. Detailed traces are sampled; safety counters are never sampled.

## Benchmark envelope

Each run records host CPU model/cores, memory, OS/kernel, runtime/compiler/dependency hashes, CPU affinity, storage/replication topology, instrument count, per-broker mode/depth mix, event sizes and rate distribution, chain cadence, active detectors, UI clients and persistence load. Test instrument counts are workload sizes, not product limits.

Begin with generated protocol fixtures for all three wire formats (Dhan little-endian binary, Upstox Protobuf, Zerodha big-endian binary) and synthetic bursts. After authorized read-only commissioning, measure each broker's entitled live-feed peak and run a sustained workload at twice the combined peak with a declared burst pattern. Report the sustainable envelope and saturation point.

Required scenarios: 30-minute steady load after warmup; 5-minute 2× burst; reconnect storm on one and on all adapters; malformed frames; high-cardinality instrument changes; maximum certified depth; slow analytical storage; slow ledger; CPU contention; token rotation; stalled browsers; data-source failover between brokers mid-session.

## Overload ladder and acceptance

At rising queue age: lower UI cadence → shed optional depth and expensive detectors → reduce discretionary subscriptions with explicit coverage loss → halt affected automated entries. Account updates and protection capacity are always preserved. If execution-update memory and durable spool exhaust, mark an audit gap, stop new risk and reconcile.

Pass criteria: every budget above met within the envelope, bounded memory, no unreported analytical gaps, no lost ledger truth, no bypassed risk, no blind write retry, usable UI. A failure produces a capacity report and a tighter admission envelope, not a rewritten threshold. Re-run after changes to decoders, queue ownership, features, database topology or dependency versions.
