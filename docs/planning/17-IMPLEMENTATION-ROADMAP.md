# Implementation roadmap

Ordered phases, epics and milestones for building the platform module by module. Task-level detail is in [18](18-GRANULAR-DEVELOPMENT-TASKS.md); decisions and gates in [19](19-DECISIONS-AND-RISKS.md); execution checklist in [20](20-DEVELOPER-HANDOFF.md). No calendar dates: ordering, dependencies and effort classes (S ≤ 2 engineer-days, M ≤ 2 weeks, L > 2 weeks) only.

## Epics

| Epic | Scope | Effort |
|---|---|---|
| E01 Platform foundation & contracts | Workspace, contract crate, IPC, config versioning, CI, fixture harness, telemetry skeleton | M |
| E02 Instruments, mapping, sectors, sessions | Three masters, canonical IDs, restrictions, sector catalog, effective-dated session rules/calendar | M |
| E03 Credential coordinator & security | Per-broker auth flows, secret storage, token generations, roles, postback ingress | M |
| E04 Dhan adapter | Eight interfaces: LE binary feed, depth feeds, REST, history, chain, orders, order stream/postback, portfolio, funds | L |
| E05 Upstox adapter | Eight interfaces: Protobuf V3 feed, Market Quote V3, history V3, chain/Greeks, V3 orders, portfolio stream/webhook, funds | L |
| E06 Zerodha adapter | Eight interfaces: BE binary feed + text order frames, quotes, history, ChainBuilder input, orders/GTT/alerts, signed postbacks, portfolio, margins | L |
| E07 Market data engine | Subscription planner, shards, state, quality, failover | L |
| E08 Candle engine | History scheduler, local bars, aggregation, revisions, reconciliation | M |
| E09 Sector & ranking engine | Contributions, sufficient statistics, rank indexes, profiles | M |
| E10 Scanner catalog | Feature DAG, 56 detectors, candidate envelopes | L |
| E11 Regime & opportunity engine | Regime axes, candidate lifecycle, scoring, reevaluation | M |
| E12 Options/derivatives intelligence | ChainBuilder, LOCAL_MODEL Greeks, strike ranking, multi-leg planning | M |
| E13 Risk engine | Policies, allocations, gates, reservations, protection state, emergency modes | L |
| E14 OMS, gateways & reconciliation | Intent/command ledger, fenced gateways, uncertainty handling, exit controller, reconciliation | L |
| E15 Data architecture & persistence | PostgreSQL schema/migrations, ClickHouse datasets, outbox, retention, backup/restore | M |
| E16 Dashboard & control API | Control endpoints, view streams, fourteen screens, accessibility | L |
| E17 Performance, telemetry & ops | Instrumentation, benchmark harness, overload ladder, CPU pinning, standby/fencing, runbooks | M |
| E18 Compliance gates & commissioning | Gate register G01–G14, evidence capture, read-only and live commissioning procedures | M |

## Phase table

| Phase | Goal | Epics | Entry criteria | Exit criteria / milestone | Gates that may block |
|---|---|---|---|---|---|
| P0 Foundations | Buildable workspace with frozen contracts and fixture harness | E01, E15 (schema skeleton), E17 (telemetry skeleton) | Pack approved; toolchain pinned | **M0 contracts frozen**: contract crate v1 tagged, IPC schema negotiated, CI green, three wire-format fixture loaders exist | None |
| P1 Instruments & sessions | Canonical universe and calendar | E02, E03 (secret storage, roles) | M0 | Masters for all three brokers import and diff; restrictions tri-state; sector catalog versioned; session rules CURRENT_2026_08_03_CAS and PENDING sets loaded with activation gates | G07, G09 (licensing of constituents) |
| P2 Read-only adapters | Three adapters passing the shared conformance suite offline | E04, E05, E06, E03 (auth flows), E12 (ChainBuilder, Greeks) | P1 | **M1 three read-only adapters certified offline**: identical semantic suite passes per broker on fixtures; mutation methods implemented but disabled by capability flags | G02 (auth), G05, G06, G13 |
| P3 Market core | Live intelligence from fixtures, then from read-only feeds | E07–E11, E15 (analytics), E16 (read-only screens) | M0 + fixture replay; M1 for live | **M2 market intelligence live read-only**: all 56 detectors, sector/regime/opportunity engines, candles and dashboard run against entitled live feeds with coverage/quality visible; sub-ms budgets met on fixtures | G05, G09 |
| P4 Execution core | Durable, fenced, reconciled OMS and risk, proven offline | E13, E14, E15 (ledger), E16 (ticket/orders/positions/risk screens) | M1 | **M3 execution core passes fault suite**: all fault tests in [11](11-ORDER-EXECUTION-ENGINE.md) and [12](12-RISK-MANAGEMENT.md) pass; no path from strategy to broker bypasses ledger+risk | G10 (durability/fencing) |
| P5 Commissioning | Account-backed certification, per broker and per product, read-only first | E18, E04–E06 (live fixes) | M2, M3, owner authorization | **M4 read-only live commissioning complete per broker**, then **M5 live execution enabled per broker/product** after bounded live checks | G01–G14 |
| P6 Hardening | Measured capacity, failover drills, operations | E17, E18 | M4 | **M6 performance certified** at 2× measured peak; standby fencing drill passed; runbooks executed once | G10, G11 |

M5 is reached independently per broker and per product (equity intraday, futures, options) — a broker/product remains fail-closed until its own evidence exists.

## Dependency graph

~~~mermaid
flowchart TD
  E01 --> E02
  E01 --> E03
  E01 --> E15
  E01 --> E17
  E02 --> E04
  E02 --> E05
  E02 --> E06
  E03 --> E04
  E03 --> E05
  E03 --> E06
  E02 --> E07
  E01 --> E07
  E07 --> E08
  E08 --> E09
  E08 --> E10
  E09 --> E10
  E02 --> E12
  E06 --> E12
  E10 --> E11
  E12 --> E11
  E09 --> E11
  E02 --> E13
  E15 --> E13
  E13 --> E14
  E04 --> E14
  E05 --> E14
  E06 --> E14
  E11 --> E16
  E14 --> E16
  E07 --> E16
  E14 --> E18
  E16 --> E18
  E17 --> E18
~~~

Fixture-driven development breaks the apparent E04–E06 → E07 dependency: E07–E11 consume the normalized observation contract from E01 and run on generated fixtures until M1.

## Ordered sequence inside each phase

**P0.** E01 workspace and crate graph → contract crate (types, enums, quality, error taxonomy from [04](04-BROKER-ADAPTER-CONTRACTS.md)) → IPC framing and negotiation → versioned config loader with ComplianceProfile/RiskPolicy/broker_capability schemas → fixture harness for Dhan LE, Upstox Protobuf (pinned descriptor), Zerodha BE formats → telemetry skeleton with sub-ms histogram buckets → CI (fmt, clippy, tests, license inventory, no-network guard) → E15 PostgreSQL skeleton and migration tool.

**P1.** E02 Dhan detailed CSV import → Upstox BOD JSON import → Zerodha gzipped CSV import with daily snapshot retention → canonical instrument key/ID assignment and ambiguity quarantine → option_contract derivation → restriction tri-state model → sector catalog and membership import → session rule engine with effective dating and phase states → E03 secret storage, credential metadata, role scopes.

**P2.** Per adapter (E04 ∥ E05 ∥ E06): describe_capabilities/health/start → credential flow → live feed decoder with golden and malformed fixtures → subscription reconciliation → REST snapshot → historical with chunking → options data (provider chain, or ChainBuilder for Zerodha) → order-update stream + ingress (Zerodha checksum) → portfolio/funds reads → trading methods implemented behind disabled capability flags → shared conformance suite. E12 ChainBuilder and LOCAL_MODEL Greeks run alongside E06.

**P3.** E07 planner and shards → E08 bars/aggregation/revisions → E09 contributions/ranks → E10 shared features then detector families in order: price/momentum, volume/liquidity, structure, sector, volatility, derivatives → E11 regime, candidates, scoring → E15 analytics outbox → E16 read-only screens (dashboard, heatmap, drilldown, scanner, chart, F&O scanner, chain, opportunities, broker settings, config, health).

**P4.** E15 ledger tables and constraints → E13 policy model, allocations, gate chain, reservations → E14 intent/command lifecycle, fenced gateway per account, submission uncertainty, update merge, exit controller, protection, reconciliation, account lease → E16 ticket/orders/positions/risk screens and emergency controls → fault suite.

**P5.** E18 gate register populated with evidence → per broker: read-only live certification (identity, entitlements, units, representative instruments, live decode, history, chain, streams) → per broker and product: bounded live commissioning (LIMIT, SL-LIMIT, modify, cancel, child orders, protection, reconciliation) → enable route.

**P6.** E17 benchmark at measured 2× peak, overload ladder, CPU pinning validation → standby promotion with fencing drill → runbook rehearsal → retention and restore drill.

## Parallelization guidance

- After M0, E02/E03/E15 run in parallel; E04, E05 and E06 start as soon as E02 provides mappings and E03 provides secret handles (three engineers, one per adapter, sharing the conformance suite).
- E07–E11 start on fixtures right after M0; do not wait for M1.
- E13 and E14 start after E15 ledger skeleton; they need only the contract crate and one adapter's mutation methods to begin fault testing.
- E16 follows the view-stream protocol from E01 and can build screens against recorded snapshots.
- E17 instrumentation is embedded in every epic from P0; the benchmark harness is built in P0 and rerun at each milestone.

## Milestone acceptance evidence

| Milestone | Evidence (link) |
|---|---|
| M0 | Contract crate conformance and error taxonomy — [04](04-BROKER-ADAPTER-CONTRACTS.md) acceptance; stack pinning — [03](03-TECH-STACK-DECISIONS.md) acceptance |
| M1 | Same semantic suite per adapter — [04](04-BROKER-ADAPTER-CONTRACTS.md) conformance; feed fixtures — [05](05-MARKET-DATA-ENGINE.md) acceptance; interval boundaries — [06](06-CANDLE-ENGINE.md) acceptance |
| M2 | [07](07-SECTOR-HEATMAP.md), [08](08-MARKET-SCANNER-CATALOG.md), [09](09-REGIME-AND-RANKING-ENGINE.md), [10](10-EQUITY-FUTURES-OPTIONS.md), [14](14-DASHBOARD-UX.md) acceptance; crash isolation and failover — [02](02-SYSTEM-ARCHITECTURE.md) acceptance |
| M3 | Fault tests — [11](11-ORDER-EXECUTION-ENGINE.md) and [12](12-RISK-MANAGEMENT.md) acceptance; ledger constraints and restore — [13](13-DATA-ARCHITECTURE.md) acceptance |
| M4/M5 | Gate register G01–G14 verified per broker/product — [16](16-SECURITY-AND-COMPLIANCE.md) launch gate register; security tests — [16](16-SECURITY-AND-COMPLIANCE.md) acceptance |
| M6 | Benchmark envelope and pass criteria — [15](15-PERFORMANCE-ENGINEERING.md) |

## Work without broker credentials vs work requiring them

| Without credentials (P0–P4) | Requires account access (P5–P6) |
|---|---|
| Everything in E01, E02 (public masters), E07–E17 on fixtures | Entitlement matrix, actual tiers (Dhan Data API, Upstox Plus, Zerodha Connect vs Personal) — G05, G13 |
| Adapter decoders, chunkers, serializers, error mapping, conformance suite | Price/tick/volume/Greek units from real responses — G06 |
| OMS/risk fault suite with synthetic broker responses | Order-update channel behavior per app type (Zerodha Personal postbacks) — G14 |
| ChainBuilder and LOCAL_MODEL Greeks validated against provider fixtures | Static IP registration, algo tagging, daily logout behavior — G02, G03 |
| Session rule engine with published circulars | BSE operational circulars, broker square-off deadlines — G07 |
| Benchmarks on synthetic bursts | Live-feed peak measurement and 2× certification — M6 |

Live commissioning is per broker and per product, read-only first. No task in P0–P4 submits a brokerage write; routine CI never contacts a broker.
