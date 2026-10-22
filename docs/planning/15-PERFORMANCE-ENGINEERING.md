# Performance targets and measurement contract

This document owns every latency, throughput, memory and staleness number in the pack; other documents quote these values and never define alternates. All numbers are proposed acceptance targets under the declared load envelope (C1 in [00](00-EXECUTIVE-OVERVIEW.md)). Design intent: **sub-millisecond compute** for every in-memory stage on the market path (p50 *and* p99), **measured separately** from durable commit, broker network and browser rendering, which are bounded but not sub-ms.

## Measurement classes

Every stamp belongs to one class. A path is reported per class and end to end; p99 values of stages are never added to claim an end-to-end p99.

| Class | Boundary | Sub-ms eligible | Owner |
|---|---|---|---|
| INGEST | Socket read → normalized, mapped observation in the shard ring | Yes | Adapter process |
| CALC | Ring dequeue → instrument state, active bars (all intervals), dirty features, affected detectors, sector/rank deltas, candidate score | Yes | Core shard/sector/rank workers |
| EXEC | Candidate → risk → durable intent → request write; ACK and fill measured separately per broker | Risk yes; commit and network no | Account actor, gateway |
| PUBLISH | State version → view delta encoded → WebSocket frame written | Encode yes; socket write no | Control process publisher |
| RENDER | Browser frame received → store applied → painted | No (frame-budget bounded) | Browser |

## Named end-to-end paths

Each path is measured independently with its own histogram, sample count and market phase. A path with coalescing inside it reports the coalescing delay as a separate component.

| Path | From → to | p50 | p95 | p99 | Notes |
|---|---|---:|---:|---:|---|
| **L1 tick→calculation** | Frame receive → instrument state, all active bars, dirty features and detectors applied | 0.20 ms | 0.80 ms | 2 ms | INGEST + IPC + CALC without scoring |
| **L2 tick→decision** | Frame receive → candidate decision | 0.50 ms | 2 ms | 5 ms | Processing-age TTL 25 ms; older candidates are not auto-executed |
| **L3 candle update** | Observation applied → active 1-minute bar and every enabled parent interval (≤9) updated, geometry derived, revision stamped | 0.02 ms | 0.05 ms | 0.10 ms | Inside the state-apply budget; O(T) for T enabled intervals |
| **L4 heatmap update** | Constituent contribution → sector sufficient statistics and ordered rank index changed | 0.02 ms | 0.10 ms | 0.30 ms | Tile publication is coalesced separately (L7) |
| **L5 scanner evaluation** | One detector service time per event | 0.01 ms | 0.05 ms | 0.10 ms | All dirty detectors for one instrument per event p99 ≤ 0.50 ms |
| **L6 order submission** | Eligible decision → broker request write, including durable commit | 2 ms | 5 ms | 10 ms | 50 ms pre-send TTL where the strategy permits |
| **L7 tick→publish (priority lane)** | Frame receive → order/risk/candle-close/candidate-change delta written to socket | 1 ms | 3 ms | 10 ms | Never coalesced |
| **L7c state→publish (coalesced lane)** | State version → market display delta written | ≤ view cadence (100–200 ms) | +50 ms | +100 ms | Latest value wins; cadence 5–10 Hz |
| **L8 tick→screen (priority lane)** | Frame receive → painted in browser on the reference LAN host | 30 ms | 80 ms | 150 ms | Includes one animation frame |
| **L8c server state→visible UI (coalesced)** | State version → painted | 100 ms | 250 ms | 500 ms | Reduce cadence, then disconnect slow consumers with resync |
| **L9 render frame** | rAF batch: delta apply + React commit + chart/canvas paint at declared maximum visible rows | — | 16 ms | 33 ms | p95 under one 60 Hz frame |

## Stage budgets

| Stage | p50 | p95 | p99 | Overrun policy |
|---|---:|---:|---:|---|
| Frame receive → normalized observation (decode, validate, map) | 0.05 ms | 0.20 ms | 0.50 ms | Alert when ingress queue age > 2 ms; reconnect on irrecoverable overflow |
| IPC enqueue → shard dequeue (loopback Unix socket, SPSC ring) | 0.01 ms | 0.03 ms | 0.05 ms | Queue age drives the overload ladder |
| Normalized → instrument state + active bars + dirty feature nodes applied | 0.10 ms | 0.30 ms | 0.70 ms | Per-shard budget; shed optional feature nodes first |
| One feature node evaluation (EMA, ATR, RVOL, deque extrema, …) | — | — | 0.05 ms | Refcounted; never computed when unused |
| Opportunity rescoring per affected candidate | 0.02 ms | 0.05 ms | 0.10 ms | Expire stale candidates rather than queue |
| Account risk evaluation, excluding durability | 0.05 ms | 0.20 ms | 0.50 ms | Never skip a check |
| Durable intent commit round trip (local NVMe, synchronous commit) | 1 ms | 3 ms | 5 ms | Slow commit delays or expires the intent; never bypassed |
| Order/execution update → ledger applied + risk revised | 0.20 ms | 0.50 ms | 1 ms | Reserved channel; never dropped |
| View delta encode per changed row (field-level JSON) | 0.005 ms | 0.02 ms | 0.05 ms | Publisher thread; never on a shard |
| Browser decode + store apply per animation frame | — | 4 ms | 8 ms | Above p95 → move decode to a Web Worker ([14](14-DASHBOARD-UX.md)) |

Processing-age TTL is not market-data freshness: a chain refreshed every few seconds or a completed 15-minute candle remains an older input even when processed in 0.5 ms. Strategies declare feature-specific freshness; telemetry reports input age and compute latency separately.

Broker ACK latency starts at request write and ends at the received broker response; fill latency ends at the verified fill observation. Report both per broker (DHAN, UPSTOX, ZERODHA) with sample size, reject/timeout rate and market phase. Upstox `metadata.latency` and any broker-supplied timing are broker processing measurements, not end-to-end exchange latency.

## Critical path and asynchronous work

Synchronous market path: decode/normalize → canonical mapping lookup (immutable master-version map) → shard state update → active bars for every enabled interval → changed feature nodes → affected detectors → sector contribution delta → context snapshot validation → score. Synchronous execution path: account checks and reservation → durable intent transaction → ownership/freshness/session recheck → rate admission → request write. No historical fetch, full-universe sort, analytical query, JSON conversion, task spawn, browser publish or polling loop occurs on either path.

Asynchronous: raw/analytical persistence, audit export, dashboard fan-out, historical warmup/repair, instrument/sector catalog imports, option-chain REST refresh, local Greeks recomputation for non-selected strikes, telemetry export. Account fill durability is operationally critical and has dedicated resources; it is never batched with analytics. A slow or disconnected browser can only cause its own publisher queue to shed; it cannot delay any CALC or EXEC stage.

## Sub-millisecond engineering rules

1. **Zero allocation per tick on the hot path.** Reusable decode buffers, fixed-capacity typed events, preallocated rings/windows, dense per-shard arrays indexed by shard-local instrument index (not hash lookups per tick), bounded maps sized at startup from master counts.
2. **Fixed-point integers** for prices/quantities/turnover on the hot path; floats only inside feature math that tolerates them, never for accounting.
3. **One writer per shard; no locks on the tick path.** Cross-shard communication uses bounded SPSC/MPSC queues carrying immutable snapshots. Sector/regime/rank indexes are updated by their single owner from contribution deltas.
4. **Dependency DAG with refcounts.** Only dirty nodes evaluate; unused indicators are never computed; O(K) chain computations run once per snapshot and fan out cached results.
5. **Incremental everywhere; no full recalculation on an event.** Indicators use recurrences (EMA, Welford variance, Kahan/wide-integer sums), monotonic deques for rolling extrema, ring buffers for SMA/ATR, order-statistic trees for medians and percentiles, bounded level indexes for support/resistance, subtract-old/add-new sufficient statistics for sectors, O(log N) ordered indexes for ranks. Full rebuilds occur only on membership change, parameter change, source-epoch reset or scheduled audit, and run off the tick path.
6. **Precompute what is known before the session.** Session phase schedule, bar boundaries, same-phase elapsed-volume baselines, instrument→shard/sector maps, detector dependency masks and strike ladders are built before 09:00 IST and swapped in as immutable versions.
7. **CPU isolation.** Pin shard workers, adapter decode workers and the account actor to dedicated cores (isolcpus/cpuset); keep Tokio I/O and publisher threads on separate cores; disable frequency scaling on trading hosts; huge pages for state arrays after measurement.
8. **No syscalls inside the tick update.** Monotonic timestamps via vDSO; logging through a lock-free ring drained by a separate thread; metrics via per-core counters aggregated off-path.
9. **Batch only where latency is not on the path.** Analytics batches wait up to 100 ms; UI coalesces within view cadence; candle-relevant observations, candle close/correction events and execution events are never coalesced.
10. **Deterministic ordering where required.** Within a shard, observations apply in adapter output order; candle placement uses verified exchange time with the lateness window in [06](06-CANDLE-ENGINE.md); sector statistics are commutative so cross-shard arrival order cannot change results; every published stream carries a per-subscription sequence. Replaying a fixture produces byte-identical state, bar revisions and candidate envelopes.

Optimization order after correctness: eliminate work (DAG, incremental sums, no DB/JSON per tick) → remove allocation → isolate CPU/I/O → tune batching → only then custom lock-free structures, kernel-bypass or transport changes, each justified by a profile.

## Reference workload envelope (Indian market)

Design assumptions used for sizing, queue capacities and benchmarks. They are replaced by measured values at read-only commissioning (M4) and are not product limits (C1, C6).

| Dimension | Design value | Source / note |
|---|---|---|
| Session timeline (IST) | Pre-open 09:00–09:08; cash continuous 09:15–15:30 (CAS and post-close per effective-dated rules); normal derivatives to 15:40 | Phase rules owned by [16](16-SECURITY-AND-COMPLIANCE.md) |
| Burst windows | 09:15–09:20 open, 15:15–15:30 close, index-event minutes | Burst factor 4× mean for 5 minutes |
| Subscription slots available | Dhan D-LIVE 25,000; Upstox U-FULL 2,000 / U-LTPC 5,000 (mixed caps apply); Zerodha Z-LIVE 9,000 | Budget IDs in [01](01-API-RESEARCH.md); planner never exceeds them |
| Covered instruments (design) | 6,000 canonical instruments: 3,500 cash equities quote/full, 150 indices, 2,000 F&O contracts, 50 option chains at provider cadence | DATA_COVERED population; DISCOVERED is larger |
| Mean observation rate | 20,000 observations/s combined across active sources | Consistent with [13](13-DATA-ARCHITECTURE.md) sizing |
| Peak observation rate | 80,000 observations/s for 5 minutes; certification at 2× measured live peak | Scenario in benchmark envelope |
| Enabled intervals per instrument | Up to 9 (1, 2, 3, 5, 10, 15, 30, 60 min, daily) | [06](06-CANDLE-ENGINE.md) |
| Active detectors | All 56 enabled for the covered universe; derivatives family on 50 chains | [08](08-MARKET-SCANNER-CATALOG.md) |
| Browser clients | ≤ 8 concurrent; ≤ 32 view subscriptions per client; ≤ 2,000 visible rows per view | Single-owner deployment |
| Order traffic | ≤ 10 orders/s per broker (regulatory OPS threshold), reserved protective capacity | [01](01-API-RESEARCH.md) order budgets |

## Throughput, concurrency and memory budgets

| Resource | Budget | Policy |
|---|---|---|
| Shard worker throughput | ≥ 200,000 observation applies/s per pinned core at the stage budget | Default 8 shards → ≥ 1.6 M/s headroom vs 80 k/s peak |
| IPC | ≥ 1 M frames/s loopback, p99 enqueue→dequeue ≤ 50 µs | Frame ≤ 64 KiB; SPSC per direction |
| Publisher encode | ≥ 50,000 changed rows/s per publisher thread | One publisher thread per ≤ 4 clients |
| Worker counts | Fixed at startup: 1 decode worker per socket, 1 writer per shard, 1 sector owner per sector group, 1 account actor per broker account, 1 publisher per client group | No task per tick; no dynamic thread creation during the session |
| Per-instrument hot state | ≤ 8 KiB: fields, ages, validity bits, cumulative baselines, depth snapshot, 9 active bar accumulators, enabled feature scalars | Dense arrays; 6,000 instruments ≈ 48 MiB |
| Bounded lookback per instrument | 1-min 400 bars; 2/3/5-min 200; 10/15-min 100; 30/60-min 50; daily 300 ≈ 1,300 bars × 48 B ≈ 62 KiB | 6,000 instruments ≈ 370 MiB; older bars served from ClickHouse |
| Queue memory | capacity × event size, fixed at startup (e.g. 65,536 × 256 B = 16 MiB per shard ring) | Unbounded queues prohibited |
| Process RSS | core-market ≤ 4 GiB at the design universe; each adapter ≤ 512 MiB; control/publisher ≤ 1 GiB; ledger pool bounded | Alarm at 75%, shed at 85%, halt new coverage at 95% |
| Browser | ≤ 512 MiB heap per tab; ≤ 50,000 bars resident per chart (older pages released) | Conflation enabled on long histories |

## Backpressure and staleness policies

| Boundary | Queue | On full | Coalescing allowed |
|---|---|---|---|
| Socket → decoder | TCP receive buffer + reserved frame pool | Stop reading (TCP backpressure); reconnect on irrecoverable overflow with quality-gap record | No |
| Decoder → shard | SPSC ring | Record gap, increment source epoch, invalidate continuity windows | No (candle inputs) |
| Shard → sector/rank owner | Bounded MPSC of ContributionChanged | Latest contribution per instrument supersedes (old/new values carried) | Yes, lossless by construction |
| Shard/sector → publisher | Dirty-set per view (latest value) | Overwrite; publish at cadence | Yes (display only) |
| Shard → analytics outbox | Bounded spool | Shed raw ticks under declared retention policy; never shed candles or quality-gap records | Yes |
| Account updates → ledger | Reserved channel + durable spool | Never dropped; on exhaustion stop new entries, mark audit gap, reconcile | No |
| Publisher → browser | Per-client bounded buffer | Lower cadence → drop coalesced lanes → disconnect with resync reason; priority lane preserved | Lanes P2/P3 only |

Staleness defaults (configurable per strategy; UNKNOWN blocks new risk):

| Input | Stale when | Effect |
|---|---|---|
| Feed transport | No frame or heartbeat within the broker-specific liveness window ([05](05-MARKET-DATA-ENGINE.md)) | Source DEGRADED; failover evaluation |
| Instrument price (liquid class) | Age > 1 s for automated entry; > 3 s for protective-stop evaluation; a quiet contract with a live socket is QUIET, not STALE | Entry gate fails; UI badge at > 2 s |
| Candle | Provisional close not finalized within the 2 s lateness window; parent missing children | INCOMPLETE flag; detectors requiring complete bars idle |
| Sector aggregate | Benchmark or coverage age > 2 s, or valid coverage < 80% | PARTIAL; sector-confirmation strategies blocked |
| Regime | Snapshot age > 5 s | Strategies requiring regime idle |
| Chain snapshot | Age > 2 provider refresh periods (Dhan 3 s cadence; Upstox per U-STANDARD; Zerodha ChainBuilder cycle) | IV/Greek detectors Q=0 |
| Account state | Funds/positions snapshot older than policy, or order-update channel lost | Entries blocked on that account |
| Browser view | No delta or heartbeat for 3 s | Client shows STALE banner and requests resync |

## Measurement instrumentation

Capture monotonic stamps at receipt, decode start/end, IPC enqueue/dequeue, state apply, bar update, feature/scanner start/end, sector apply, decision, risk start/end, commit start/end, send, ACK and fill receipt, publisher dirty-mark, encode start/end and socket write. Local durations use monotonic clocks with `clock_domain` and `host_boot_id`; UTC correlates systems and cross-host comparisons carry the measured synchronization uncertainty. Monotonic stamps are never compared across reboot or standby.

Browser side: `performance.mark` at WebSocket message receive, store apply, rAF commit and (where available) paint; each delta carries `server_time` and the originating `state_version` so tick→screen is computed as browser-receive-minus-server-stamp plus local render, with the clock-offset estimate reported alongside. Chart adapter records `update()` call count per second and dropped (superseded) updates.

Persist intent `expires_at_utc` and reconstruct runtime deadlines after restart only under healthy clock bounds. Clock-health gate: warn when the wall-clock offset estimate exceeds 5 ms; block short-lived automated entries when uncertainty exceeds their remaining TTL or 20 ms. Use PTP or disciplined NTP on trading hosts; record the source.

Histograms use sub-millisecond buckets (10 µs resolution below 1 ms) and record sample count, warmup exclusion, observed maximum, p50/p95/p99, queue age, coalesced/lost events, CPU per core, RSS, allocation rate, disk fsync latency and network errors. Detailed traces are sampled; safety counters are never sampled.

## Benchmark envelope

Each run records host CPU model/cores, memory, OS/kernel, runtime/compiler/dependency hashes, CPU affinity, storage/replication topology, instrument count, per-broker mode/depth mix, event sizes and rate distribution, chain cadence, active detectors, enabled intervals, UI clients and visible rows, and persistence load. Test instrument counts are workload sizes, not product limits.

Begin with generated protocol fixtures for all three wire formats (Dhan little-endian binary, Upstox Protobuf, Zerodha big-endian binary) and synthetic bursts shaped by the reference workload. After authorized read-only commissioning, measure each broker's entitled live-feed peak and run a sustained workload at twice the combined peak with the declared burst pattern. Report the sustainable envelope and saturation point.

Required scenarios: 30-minute steady load after warmup; 5-minute 4× open burst; reconnect storm on one and on all adapters; malformed frames; high-cardinality instrument changes; maximum certified depth; slow analytical storage; slow ledger; CPU contention; token rotation; stalled browsers and 8 active browsers with maximum views; data-source failover between brokers mid-session; replay determinism (identical outputs on two runs of the same fixture).

## Overload ladder and acceptance

At rising queue age: lower UI cadence → shed optional depth and expensive detectors → reduce discretionary subscriptions with explicit coverage loss → halt affected automated entries. Account updates and protection capacity are always preserved. If execution-update memory and durable spool exhaust, mark an audit gap, stop new risk and reconcile.

Pass criteria: every named path and stage budget met within the envelope; bounded RSS within the memory budgets; zero allocations and zero syscalls per observation at steady state; replay determinism; no unreported analytical gaps; no lost ledger truth; no bypassed risk; no blind write retry; usable UI at L9 with 8 clients. A failure produces a capacity report and a tighter admission envelope, not a rewritten threshold. Re-run after changes to decoders, queue ownership, features, candle intervals, database topology, chart library or dependency versions.
