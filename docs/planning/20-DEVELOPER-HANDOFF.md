# Developer handoff

Execution checklist for the implementing engineer or agent. Read [00](00-EXECUTIVE-OVERVIEW.md) for conventions C1–C6, [17](17-IMPLEMENTATION-ROADMAP.md) for order, [18](18-GRANULAR-DEVELOPMENT-TASKS.md) for tasks, [19](19-DECISIONS-AND-RISKS.md) for closed decisions. Broker facts come only from [01](01-API-RESEARCH.md); never re-derive a limit from memory.

## Repository layout

~~~
platform/
  Cargo.toml                # workspace, pinned toolchain via rust-toolchain.toml
  crates/
    contract/               # shared types, enums, quality, errors, IPC message schema (04)
    greeks/                 # Black-76 pricing, bounded IV solver, convergence flags (ADR-06)
    ipc/                    # length-delimited framing, schema negotiation, bounded buffers (ADR-12)
    adapter-dhan/           # LE binary + depth decoders, REST, history, chain, orders, stream/postback
    adapter-upstox/         # prost descriptor, V3 feed, Market Quote V3, history V3, orders, portfolio stream
    adapter-zerodha/        # BE binary decoder + text frames, quotes, history, ChainBuilder inputs, orders/GTT, signed postbacks
    core-market/            # subscription planner, shards, candles, feature DAG, sectors, scanners, regime, opportunities
    core-execution/         # account actors, risk, reservations, OMS, exit controller, reconciliation, leases
    control-api/            # Axum control endpoints, view streams, postback ingress
    persistence/            # sqlx ledger access, ClickHouse outbox, Parquet export, retention jobs
    telemetry/              # histograms (10 µs buckets), counters, tracing, health snapshot
  ui/                       # Vite + React + TypeScript dashboard
  fixtures/
    dhan/  upstox/  zerodha/  # golden, malformed and boundary frames; REST samples (redacted)
  migrations/               # PostgreSQL migrations, ClickHouse DDL
  config/                   # schema for versioned config, ComplianceProfile, RiskPolicy, broker_capability
  runbooks/
~~~

~~~mermaid
flowchart LR
  contract --> ipc
  contract --> greeks
  contract --> adapter-dhan
  contract --> adapter-upstox
  contract --> adapter-zerodha
  ipc --> adapter-dhan
  ipc --> adapter-upstox
  ipc --> adapter-zerodha
  ipc --> core-market
  ipc --> core-execution
  greeks --> core-market
  contract --> core-market
  contract --> core-execution
  persistence --> core-execution
  persistence --> core-market
  core-market --> control-api
  core-execution --> control-api
  telemetry --> adapter-dhan
  telemetry --> core-market
  telemetry --> core-execution
  telemetry --> control-api
~~~

Adapters depend only on `contract`, `ipc`, `telemetry`; the core never depends on an adapter crate. Broker wire enums stay inside adapter serializers.

## Toolchain and pinning

Pinned stable Rust toolchain in `rust-toolchain.toml`; `Cargo.lock` committed; dependencies, Upstox Protobuf descriptor (hash recorded in evidence), Node toolchain, container base images pinned by digest. Dependency/license inventory generated in CI. Any upgrade of a decoder, queue, database driver or descriptor reruns the performance suite ([15](15-PERFORMANCE-ENGINEERING.md)).

## Coding rules for the hot path

1. Zero allocation per tick: reusable decode buffers, fixed-capacity events, dense per-shard arrays indexed by shard-local index, bounded maps sized at startup.
2. Fixed-point integers for prices/quantities/turnover; `Money` and `Price` from `contract` only; no `f64` in accounting.
3. One writer per shard and per account; cross-owner communication by bounded SPSC/MPSC queues carrying immutable snapshots; no `Mutex` on the tick path.
4. No syscalls inside the tick update: vDSO monotonic clock, lock-free log ring, per-core counters.
5. Bounded queues only; unbounded channels and task-per-tick spawning fail code review.
6. Dependency DAG: a feature computes only when dirty and referenced; detectors declare required feature masks.
7. Broker wire fields never cross the adapter boundary; unknown enums are quarantined, not mapped.
8. Every observation, bar, feature, candidate and decision carries versions, source epoch and quality.
9. Mutation methods are callable only through the fenced gateway with a CommittedOrderCommand; no other code path constructs a broker write.

## Configuration model

All configuration is a versioned immutable document (actor, reason, time, validation result). Schemas: `ComplianceProfile` ([16](16-SECURITY-AND-COMPLIANCE.md)), `RiskPolicy` ([12](12-RISK-MANAGEMENT.md)), `broker_capability` records with evidence URL/hash and expiry ([13](13-DATA-ARCHITECTURE.md)), `RankingProfile`, `ScannerDefinition`, `ExchangeSessionRule` with effective dates, subscription planner priorities, queue capacities. Unknown keys and unsupported capabilities fail validation. Execution-sensitive changes activate only after account-actor acknowledgement.

## Secrets

Server-side envelope encryption with a managed key service; token metadata separate from secret values; each adapter process receives only its account's secret handle; UI and analytical workers receive nothing. Never log URLs or bodies containing tokens, PINs, TOTP or `api_secret`. Zerodha `api_secret` lives only in the credential coordinator (checksum computation) and the postback ingress verifier. Unattended TOTP seeds/PINs disabled by default.

## Local development without credentials

Fixture replay drives protocol tests only (C5): recorded or generated frames for the three wire formats, redacted REST samples, synthetic order-update sequences and fault scripts. A `no-network` CI guard fails any test that opens a socket to a broker host. There is no simulated exchange, no paper account and no historical trading replay.

## Test taxonomy

| Kind | Scope | Required where |
|---|---|---|
| Unit | Pure functions, serializers, math | Every crate |
| Property | Candle aggregation invariants, reservation conservation, allocation sums, pivot determinism, Greeks round-trip | core-market, core-execution, greeks |
| Fuzz | All three frame decoders, IPC framing, postback parsers | Adapters, ipc |
| Contract fixture | Golden/malformed/boundary per broker; shared semantic conformance suite | Adapters |
| Fault injection | Crash before/after commit/send/ACK, delayed ACK, cancel-fill race, token rotation, standby fencing, 429, TokenException | core-execution |
| Performance | Budgets from [15](15-PERFORMANCE-ENGINEERING.md) on synthetic bursts | core-market, core-execution, ipc |
| UI | Interaction, resync, accessibility, stalled client | ui |

## CI gates

fmt, clippy (deny warnings), unit/property/contract tests, fuzz smoke (bounded iterations), no-network guard, migration up/down, license inventory, SBOM, fixture hash check, performance smoke (fails on allocation in hot-path functions and on p99 regression > 20 % against the stored baseline). Full performance certification runs on the reference host, not in routine CI.

## Definition of done per task

A task is done when: code and tests merged; acceptance criteria in [18](18-GRANULAR-DEVELOPMENT-TASKS.md) demonstrably met; fixtures/evidence committed; telemetry for the component emits the required counters; documentation in this pack updated if a contract changed (schema version bump); no new external gate silently assumed verified.

## Recommended starting sequence

1. T-E01-01 workspace, crate graph, toolchain pinning.
2. E01: contract crate types/enums/errors; IPC framing; versioned config loader; fixture harness (three formats); telemetry skeleton; CI gates.
3. E15: PostgreSQL skeleton, migration tool, ledger tables with constraints.
4. E02: three master imports, canonical IDs, option_contract derivation, restrictions, sector catalog, session rules.
5. E03: secret storage, credential metadata, role scopes, postback ingress with Zerodha checksum verification.
6. E04/E05/E06 in parallel: capabilities/health → auth → feed decoder → subscriptions → snapshots → history → options data → updates → portfolio/funds → trading behind flags → conformance suite.
7. E12: ChainBuilder and `greeks` crate alongside E06.
8. E07 → E08 → E09 → E10 → E11 on fixtures; E16 read-only screens.
9. E13 → E14 → fault suite; E16 execution screens.
10. E18 gate register and commissioning procedures; E17 benchmark and drills.

## Operational runbooks (write before P5)

Token renewal per broker (Dhan 24 h Web-origin renewal or re-login; Upstox before 03:30 via approved flow; Zerodha interactive login before 06:00 IST, login-elsewhere invalidation); static IP change (Dhan 7-day lock, Zerodha one change per calendar week, Upstox user-level update); standby promotion with egress fencing and reconciliation; emergency flatten per broker (Dhan Exit All, Upstox exit-all ≤50/cancel-multi with 207 handling, Zerodha per-position reducing orders) and broker disable where it exists; daily master import and diff review (~08:30 IST for Zerodha CSV); session calendar activation (CURRENT vs PENDING rule sets); ledger backup/restore drill; capacity alarms and retention actions; incident reconstruction from audit evidence.

## What you must NOT do

- Retry a write whose outcome is unknown; resolve SUBMISSION_UNKNOWN by reconciliation.
- Route an exit or a retried intent to a different broker than the one holding the position.
- Send MARKET/SL-M without certified protection (Zerodha `market_protection`, Upstox `market_protection`, phase rules); default to LIMIT/SL-LIMIT.
- Trust an unsigned webhook (Dhan postback, Upstox webhook) as authoritative; only checksum-verified Zerodha postbacks are authenticated transport.
- Compare monotonic timestamps across reboot, host or standby; durable deadlines are UTC.
- Treat API acceptance as a fill, cancel acceptance as proof of no late fill, or a bulk 207 as "flat".
- Default a missing restriction, entitlement or session rule to permissive; UNKNOWN blocks.
- Sum last-traded quantity as volume; use cumulative/day-volume deltas with reset detection.
- Mislabel Dhan 25-minute bars as 30-minute, or local aggregates as broker history.
- Put a database query, JSON conversion, allocation or task spawn on the tick path.
- Create extra apps/accounts to evade per-user quotas, or multiply deep-feed caps by connection counts.
- Mark a gate G01–G14 verified without evidence hash, reviewer and review date.
- Expose a broker token, PIN, TOTP or `api_secret` to the browser, logs, crash dumps or analytics storage.
- Build paper trading, backtesting or trading replay under any name.
