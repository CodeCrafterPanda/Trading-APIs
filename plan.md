# Implementation plan and progress tracker

Single source of truth for **what is done and what is next**. Agents update this file after every finished task (see [Update protocol](#update-protocol)). It tracks state only; it does not restate specs.

| Need | Read |
|---|---|
| Conventions C1-C6, invariants | [00 Executive overview](docs/planning/00-EXECUTIVE-OVERVIEW.md) |
| Phases, milestones, order | [17 Roadmap](docs/planning/17-IMPLEMENTATION-ROADMAP.md) |
| Task details and acceptance criteria | [18 Granular tasks](docs/planning/18-GRANULAR-DEVELOPMENT-TASKS.md) |
| ADRs, risks, open gates | [19 Decisions and risks](docs/planning/19-DECISIONS-AND-RISKS.md) |
| Repo layout, coding rules, do-not list, runbooks | [20 Developer handoff](docs/planning/20-DEVELOPER-HANDOFF.md) |
| Broker facts (limits, endpoints) | [01 API research](docs/planning/01-API-RESEARCH.md); never re-derive from memory |
| Gates G01-G14 | [16 Security and compliance](docs/planning/16-SECURITY-AND-COMPLIANCE.md) |

If a task row and doc 18 disagree, doc 18 wins; fix the row.

## Hard rules

- **No unit tests**, Rust or React (owner instruction, 22 Oct 2026). Evidence comes from property, fuzz, contract-fixture, fault-injection, performance and UI checks as listed per task in doc 18.
- Routine CI never contacts a broker (no-network guard). No brokerage write is sent before M5 (C5: no paper trading, backtesting or trading replay).
- Hot-path rules, browser rules and the full "must NOT do" list are in [doc 20](docs/planning/20-DEVELOPER-HANDOFF.md); read them before touching `core-*`, adapters or `ui/`.
- Never mark a gate G01-G14 verified without evidence hash, reviewer and review date.
- Never log or expose tokens, PINs, TOTP or `api_secret`.

## Environment (WSL2)

- Develop and build in **WSL2 Ubuntu**. The platform needs Linux: Unix domain socket IPC, vDSO monotonic clock, strace syscall checks, CPU pinning.
- Code lives in `platform/` at the repo root (layout in doc 20). Building through `/mnt/d` is slow: either work from a WSL-native clone, or keep the repo here and set `CARGO_TARGET_DIR` to a WSL path (for example `~/.cache/trading-target`).
- Rust toolchain comes from `platform/rust-toolchain.toml` (T-E01-01). Node, PostgreSQL and ClickHouse run inside WSL (containers or native), pinned by digest.
- Run every command in the verification column of doc 18 from WSL, not PowerShell.

## Status legend

| Status | Meaning |
|---|---|
| `TODO` | Not started |
| `WIP` | In progress (at most a few at once; note who/when in Notes) |
| `PARTIAL(scope)` | Part delivered; say what is left in Notes |
| `DONE` | All acceptance checks in doc 18 met with evidence |
| `BLOCKED(T-Exx-nn)` | Waiting on another task (use only when a dependency is not DONE and cannot be stubbed) |
| `GATED(Gxx)` | Live or external part needs credentials or gate evidence. The offline part (fixtures, serializers, models) proceeds normally and is recorded as `DONE-OFFLINE`; the row becomes `DONE` only with gate evidence |

## Next up

Pick work by dependencies, not milestone number. A task is ready when every ID in its **Depends on** column is `DONE` (or `DONE-OFFLINE` for gated ones).

**Ready now:** T-E01-01 (workspace, crate graph, toolchain pinning).

**Critical path (doc 18):** T-E01-01..07 -> T-E02-01..05 -> adapters E04/E05/E06 in parallel (each needs its credential task T-E03-03/04/05 and T-E07-07) -> T-E07-03 shards + T-E08-05/06/09 bars -> T-E16-02 publisher -> T-E16-11 sync -> T-E16-12 chart. Execution chain: T-E14-02 -> T-E14-03 -> T-E14-04 -> T-E14-06 -> T-E14-10/11.

**Starting sequence (doc 20):** T-E01-01 -> rest of E01 -> E15 skeleton -> E02 -> E03 -> E04/E05/E06 in parallel -> E12 -> E07..E11 on fixtures -> E16 read-only -> E13/E14 -> E18 -> E17.

## Milestone board

| Milestone | Definition | Status | Done / total |
|---|---|---|---|
| M0 | Contracts frozen | OPEN | 0/14 |
| M1 | Three read-only adapters certified offline | OPEN | 0/54 |
| M2 | Market intelligence live read-only | OPEN | 0/59 |
| M3 | Execution core passes fault suite | OPEN | 0/31 |
| M4 | Read-only live commissioning complete per broker | OPEN | 0/1 |
| M5 | Live execution enabled per broker/product | OPEN | 0/1 |
| M6 | Performance certified | OPEN | 0/6 |

Keep the counts current. Mark a milestone `DONE` only when every row in its table is `DONE` and the acceptance evidence listed for it in [doc 17](docs/planning/17-IMPLEMENTATION-ROADMAP.md) exists.

## Task tables

Columns: **Notes** starts with phase and effort (S <= 2 days, M <= 1 week, L > 1 week; per doc 18). Tables are in suggested execution order within each milestone. Titles are short labels; the spec is the doc 18 row with the same ID.

### M0 - Contracts frozen (14 tasks)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E01-01 | Workspace, crate skeleton, toolchain pin, CI config | - | TODO |  | P0 S |
| T-E01-02 | Fixed-point Price/Quantity/Money | T-E01-01 | TODO |  | P0 S |
| T-E01-03 | Canonical IDs, keys, enums (quarantine unknown) | T-E01-01 | TODO |  | P0 S |
| T-E01-04 | Event envelope, MarketObservation, Candle, Quality types | T-E01-02, T-E01-03 | TODO |  | P0 M |
| T-E01-05 | Execution contract types (OrderIntent, SubmissionResult...) | T-E01-03 | TODO |  | P0 S |
| T-E01-06 | Error taxonomy and redaction | T-E01-05 | TODO |  | P0 S |
| T-E01-07 | `ipc` crate: framing, negotiation, bounded buffers | T-E01-04 | TODO |  | P0 M |
| T-E01-08 | Fixture harness (three wire formats, no network) | T-E01-04 | TODO |  | P0 S |
| T-E01-10 | Telemetry skeleton (counters, histograms, log ring) | T-E01-01 | TODO |  | P0 S |
| T-E15-01 | PostgreSQL migrations for all tables | T-E01-01 | TODO |  | P0 M |
| T-E01-09 | Versioned configuration model | T-E01-01, T-E15-01 | TODO |  | P0 M |
| T-E15-02 | Ledger access layer, outbox | T-E15-01 | TODO |  | P0 M |
| T-E03-01 | Secret store, envelope encryption, credential metadata | T-E15-01 | TODO |  | P0 M |
| T-E03-09 | SBOM, signed artifacts, dependency/license gate | T-E01-01 | TODO |  | P0 S |

### M1 - Three read-only adapters certified offline (54 tasks)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E02-01 | Instrument master importer framework | T-E01-03, T-E15-01 | TODO |  | P1 M |
| T-E02-02 | Dhan detailed CSV normalizer | T-E02-01 | TODO |  | P1 S |
| T-E02-03 | Upstox BOD JSON normalizer | T-E02-01 | TODO |  | P1 S |
| T-E02-04 | Zerodha CSV normalizer + daily snapshot cache | T-E02-01 | TODO |  | P1 M |
| T-E02-05 | Canonical mapping resolver | T-E02-02..04 | TODO |  | P1 M |
| T-E02-06 | Freeze qty / permitted-lot ingestion, restrictions tri-state | T-E02-05 | TODO |  | P1 S |
| T-E02-07 | Option contract registry, futures contract list | T-E02-05 | TODO |  | P1 S |
| T-E02-08 | Sector catalog importer | T-E02-05 | GATED(G09) |  | P1 M. Without licensed weights only EQUAL_WEIGHT ships. |
| T-E02-09 | Session rules engine (effective-dated) | T-E01-09 | GATED(G07) |  | P1 M. PENDING rule sets stay inactive until operative-notice evidence. |
| T-E02-10 | Calendar inputs (holidays, timings, exchange status) | T-E02-09 | TODO |  | P1 S |
| T-E03-02 | CredentialCoordinator service | T-E03-01 | TODO |  | P1 M |
| T-E03-03 | Dhan auth flows | T-E03-02 | TODO |  | P2 M |
| T-E03-04 | Upstox auth flows | T-E03-02 | TODO |  | P2 M |
| T-E03-05 | Zerodha auth flow (06:00 expiry) | T-E03-02 | TODO |  | P2 M |
| T-E03-07 | Redaction and logging policy | T-E01-10 | TODO |  | P2 S |
| T-E03-08 | Egress policy and static IP tooling | T-E03-03..05 | GATED(G02) |  | P2 S. Needs accounts and approved egress. Tooling/detector offline-OK. |
| T-E04-01 | Dhan HTTP client and rate schedulers | T-E01-06, T-E03-03 | TODO |  | P2 M |
| T-E04-02 | Dhan ordinary feed decoder (LE) | T-E01-04 | TODO |  | P2 M |
| T-E04-03 | Dhan feed session manager | T-E04-02 | TODO |  | P2 M |
| T-E04-04 | Dhan 20/200-depth decoders | T-E04-02 | TODO |  | P2 M |
| T-E04-05 | Dhan REST snapshot client | T-E04-01 | TODO |  | P2 S |
| T-E04-06 | Dhan historical client | T-E04-01 | TODO |  | P2 S |
| T-E04-07 | Dhan options client | T-E04-01, T-E02-07 | TODO |  | P2 S |
| T-E04-08 | Dhan trading client | T-E04-01, T-E01-05 | TODO |  | P2 M |
| T-E04-09 | Dhan protection and controls (Super/Forever/kill switch) | T-E04-08 | TODO |  | P2 M |
| T-E04-10 | Dhan order-update stream and postback ingress | T-E04-01 | TODO |  | P2 M |
| T-E04-11 | Dhan portfolio and funds | T-E04-01 | TODO |  | P2 S |
| T-E04-12 | Dhan conformance suite + read-only checklist | T-E04-02..11 | GATED(G05,G06) |  | P2 M. Needs shared suite T-E07-07 first (not in Depends column). Live items: unit certification, tier. |
| T-E05-01 | Upstox HTTP client and limiters | T-E01-06, T-E03-04 | TODO |  | P2 M |
| T-E05-02 | Upstox market feed V3 decoder (Protobuf) | T-E01-04 | TODO |  | P2 M |
| T-E05-03 | Upstox feed session manager | T-E05-02 | TODO |  | P2 M |
| T-E05-04 | Upstox REST snapshots and market info | T-E05-01 | TODO |  | P2 S |
| T-E05-05 | Upstox historical V3 client | T-E05-01 | TODO |  | P2 S |
| T-E05-06 | Upstox options client | T-E05-01, T-E02-07 | TODO |  | P2 S |
| T-E05-07 | Upstox trading client V3 | T-E05-01, T-E01-05 | TODO |  | P2 M |
| T-E05-08 | Upstox protection and bulk (GTT, exit-all, kill switch) | T-E05-07 | TODO |  | P2 M |
| T-E05-09 | Upstox portfolio stream and webhook ingress | T-E05-01 | TODO |  | P2 M |
| T-E05-10 | Upstox portfolio and funds | T-E05-01 | TODO |  | P2 S |
| T-E05-11 | Upstox conformance suite + read-only checklist | T-E05-02..10 | GATED(G05,G06) |  | P2 M. Needs shared suite T-E07-07 first (not in Depends column). Live items: Plus/D30, BSE_FO, IV units. |
| T-E06-01 | Kite HTTP client and limiters | T-E01-06, T-E03-05 | TODO |  | P2 M |
| T-E06-02 | Zerodha WebSocket decoder (BE binary + text) | T-E01-04 | TODO |  | P2 M |
| T-E06-03 | Zerodha feed session manager | T-E06-02 | TODO |  | P2 M |
| T-E06-04 | Zerodha REST quote client | T-E06-01 | TODO |  | P2 S |
| T-E06-05 | Zerodha historical client (chunking, truncation) | T-E06-01 | TODO |  | P2 S |
| T-E06-06 | Zerodha options data (no chain endpoint) | T-E06-04, T-E02-07 | TODO |  | P2 S |
| T-E06-07 | Zerodha trading client | T-E06-01, T-E01-05 | TODO |  | P2 M |
| T-E06-08 | Zerodha protection (GTT, alerts) | T-E06-07 | TODO |  | P2 M |
| T-E06-09 | Zerodha order updates + checksum postback | T-E06-01, T-E03-05 | TODO |  | P2 M |
| T-E06-10 | Zerodha portfolio and funds | T-E06-01 | TODO |  | P2 S |
| T-E06-11 | Zerodha conformance suite + read-only checklist | T-E06-02..10 | GATED(G06,G13,G14) |  | P2 M. Needs shared suite T-E07-07 first (not in Depends column). Live items: Personal vs Connect tier, order-update channel. |
| T-E07-07 | Shared adapter conformance suite | T-E01-08 | TODO |  | P2 M |
| T-E12-01 | `greeks` crate (Black-76, IV solver) | T-E01-02 | TODO |  | P2 M |
| T-E18-01 | ComplianceProfile model | T-E01-09 | GATED(G01) |  | P2 M. Account category evidence is external. Model offline-OK. |
| T-E18-02 | Gate register G01-G14 | T-E18-01 | TODO |  | P2 M |

### M2 - Market intelligence live read-only (59 tasks)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E07-01 | Coverage dimensions and universe registry | T-E02-05 | TODO |  | P2 S |
| T-E07-02 | Subscription planner | T-E07-01, T-E04-03, T-E05-03, T-E06-03 | TODO |  | P3 L |
| T-E07-03 | Shard workers | T-E01-07 | TODO |  | P3 L |
| T-E07-04 | Validation and units | T-E07-03 | TODO |  | P3 S |
| T-E07-05 | Suspected-gap detection, source epochs | T-E07-03 | TODO |  | P3 M |
| T-E07-06 | Data-source failover | T-E07-05 | TODO |  | P3 M |
| T-E07-08 | Backpressure and overload ladder | T-E07-03 | TODO |  | P3 M |
| T-E08-01 | Series identity and storage | T-E07-03 | TODO |  | P3 S |
| T-E08-02 | Historical retrieval scheduler | T-E04-06, T-E05-05, T-E06-05 | TODO |  | P3 M |
| T-E08-03 | Row validation and quarantine | T-E08-02 | TODO |  | P3 S |
| T-E08-04 | Session alignment | T-E02-09 | TODO |  | P3 M |
| T-E08-05 | Incremental 1-minute bars | T-E07-04 | TODO |  | P3 M |
| T-E08-06 | Completion/lateness/revision state machine | T-E08-05 | TODO |  | P3 M |
| T-E08-07 | Higher-timeframe aggregation, daily bars | T-E08-06 | TODO |  | P3 S |
| T-E08-08 | Corporate actions, post-disconnect reconciliation | T-E08-02, T-E08-06 | TODO |  | P3 M |
| T-E08-09 | Tick-synchronized multi-timeframe bars (ADR-15) | T-E08-05, T-E08-06 | TODO |  | P3 M |
| T-E08-10 | Candle geometry and regeneration | T-E08-07, T-E08-09 | TODO |  | P3 S |
| T-E09-01 | Sector worker | T-E02-08, T-E07-03 | TODO |  | P3 M |
| T-E09-02 | Sector/stock metric definitions | T-E09-01 | TODO |  | P3 M |
| T-E09-03 | Ranking profiles | T-E09-02, T-E01-09 | TODO |  | P3 M |
| T-E09-04 | Stock-in-sector rank index, heatmap publisher | T-E09-03 | TODO |  | P3 S |
| T-E09-05 | Sector/stock acceptance fixtures | T-E09-02 | TODO |  | P3 S |
| T-E10-01 | Shared feature library | T-E08-06 | TODO |  | P3 M |
| T-E10-02 | Scanner framework | T-E10-01 | TODO |  | P3 L |
| T-E10-03 | Price/momentum scanners SC01-SC12 | T-E10-02 | TODO |  | P3 M |
| T-E10-04 | Volume/liquidity scanners SC13-SC18 | T-E10-02 | TODO |  | P3 M |
| T-E10-05 | Technical structure scanners SC19-SC31 | T-E10-02 | TODO |  | P3 L |
| T-E10-06 | Sector intelligence scanners SC32-SC37 | T-E10-02, T-E09-04 | TODO |  | P3 S |
| T-E10-07 | Volatility scanners SC38-SC41 | T-E10-02 | TODO |  | P3 S |
| T-E11-01 | Regime state vector | T-E09-02, T-E10-07 | TODO |  | P3 M |
| T-E11-02 | Candidate lifecycle | T-E10-02 | TODO |  | P3 M |
| T-E11-03 | Hard gates and cost-adjusted economics | T-E11-02, T-E02-09 | TODO |  | P3 M |
| T-E11-04 | Scoring model | T-E11-03 | TODO |  | P3 M |
| T-E11-05 | Strategy definitions and automated selection | T-E11-04, T-E13-01 | TODO |  | P3 M. Also depends on T-E13-01 (M3). See Mapping gaps. |
| T-E12-02 | Platform ChainBuilder | T-E12-01, T-E06-06, T-E04-07, T-E05-06 | TODO |  | P3 L |
| T-E12-03 | Derived chain features | T-E12-02 | TODO |  | P3 S |
| T-E10-08 | Derivatives scanners SC42-SC56 | T-E10-02, T-E12-02, T-E12-03 | TODO |  | P3 M |
| T-E10-09 | Scanner runtime acceptance report | T-E10-03..08 | TODO |  | P3 S |
| T-E12-04 | Strike selection | T-E12-02 | TODO |  | P3 M |
| T-E15-03 | ClickHouse datasets and batch inserter | T-E01-04 | TODO |  | P3 M |
| T-E15-04 | Market checkpoints and restore | T-E07-03 | TODO |  | P3 S |
| T-E15-05 | Retention and archive jobs | T-E15-03 | TODO |  | P3 S |
| T-E03-06 | Platform authentication and roles | T-E01-09 | TODO |  | P3 M. Not in doc 18 milestone map; placed in M2 (needed by T-E16-01). |
| T-E16-01 | Control API (Axum, OpenAPI) | T-E03-06, T-E14-02 | TODO |  | P3 L. Also depends on T-E14-02 (M3). See Mapping gaps. |
| T-E16-02 | Server publisher and view protocol | T-E16-01 | TODO |  | P3 L |
| T-E16-03 | UI foundation (stores, grids, canvas heatmap) | T-E16-02 | TODO |  | P3 L |
| T-E16-04 | Screens: main dashboard + health | T-E16-03 | TODO |  | P3 M |
| T-E16-05 | Screens: sector heatmap + drilldown | T-E16-03, T-E09-04 | TODO |  | P3 M |
| T-E16-06 | Screens: stock scanner + chart | T-E16-03 | TODO |  | P3 M |
| T-E16-07 | Screens: F&O scanner + option chain | T-E16-03, T-E12-02 | TODO |  | P3 M |
| T-E16-11 | Client sync manager | T-E16-03 | TODO |  | P3 M |
| T-E16-12 | Chart adapter (Lightweight Charts 5.x) | T-E16-03, T-E08-10 | TODO |  | P3 M |
| T-E16-13 | Chart annotations and crosshair sync | T-E16-12, T-E11-02 | TODO |  | P3 M |
| T-E17-01 | Hot-path instrumentation, clock-health gate | T-E01-10 | TODO |  | P3 M |
| T-E17-02 | Allocation and syscall audit | T-E07-03 | TODO |  | P3 S |
| T-E17-04 | Benchmark harness (reference workload) | T-E01-08, T-E07-03 | TODO |  | P3 M |
| T-E17-07 | Overload ladder automation, health model | T-E07-08 | TODO |  | P3 S. Listed under both M2 and M6 ranges in doc 18; tracked in M2. |
| T-E17-09 | End-to-end path telemetry L1-L9 | T-E17-01, T-E16-02 | TODO |  | P3 S |
| T-E17-10 | Memory and staleness budget enforcement | T-E07-03, T-E07-08 | TODO |  | P3 S |

### M3 - Execution core passes fault suite (31 tasks)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E12-05 | Product constraints matrix per broker | T-E02-06, T-E03-03..05 | TODO |  | P3 M |
| T-E13-01 | RiskPolicy model, global coordinator, allocations | T-E01-09, T-E15-01 | TODO |  | P4 M |
| T-E13-02 | Pre-trade gate pipeline | T-E13-01, T-E02-09, T-E12-05 | TODO |  | P4 L |
| T-E13-03 | Quantity and margin sizing | T-E13-02, T-E04-11, T-E05-10, T-E06-10 | TODO |  | P4 M. Live margin estimates are account-specific; fixtures offline-OK. |
| T-E12-06 | Futures lifecycle and rollover | T-E12-05, T-E13-03 | TODO |  | P3 M |
| T-E13-04 | Reservation lifecycle, daily-loss accounting | T-E13-02 | TODO |  | P4 M |
| T-E13-05 | Protection state machine | T-E13-04, T-E14-08 | TODO |  | P4 M. Circular with T-E14-08 in doc 18. Build state machine and ExitController as one unit. See Mapping gaps. |
| T-E13-07 | Risk telemetry and alerts | T-E13-04 | TODO |  | P4 S |
| T-E14-01 | Ledger entities and idempotency | T-E15-01, T-E01-05 | TODO |  | P4 M |
| T-E14-02 | Account actor | T-E14-01, T-E13-02 | TODO |  | P4 L |
| T-E14-03 | Fenced execution gateway, AccountLease | T-E14-02, T-E03-08 | TODO |  | P4 L |
| T-E14-04 | SUBMISSION_UNKNOWN reconciliation | T-E14-03, T-E04-08, T-E05-07, T-E06-07 | TODO |  | P4 L |
| T-E14-05 | Child orders and slicing | T-E14-02 | TODO |  | P4 M |
| T-E14-06 | Order-update ingestion and merge | T-E04-10, T-E05-09, T-E06-09 | TODO |  | P4 L |
| T-E14-07 | Modify and cancel semantics | T-E14-05 | TODO |  | P4 M |
| T-E14-08 | ExitController | T-E14-07, T-E13-05 | TODO |  | P4 M. Circular with T-E13-05 in doc 18. Build with it as one unit. See Mapping gaps. |
| T-E12-07 | Multi-leg execution policy | T-E12-04, T-E14-08 | TODO |  | P3 M. Basket margins are account-specific; fixtures offline-OK. |
| T-E14-09 | Protection placement adapters | T-E14-08, T-E04-09, T-E05-08, T-E06-08 | GATED(G08) |  | P4 M. Native partial-fill semantics need live evidence. Synthetic path offline-OK. |
| T-E14-10 | Snapshot reconciliation | T-E14-06 | TODO |  | P4 L |
| T-E13-06 | Emergency modes | T-E13-05, T-E14-10 | TODO |  | P4 M |
| T-E14-11 | Crash recovery procedure | T-E14-10, T-E14-03 | TODO |  | P4 L |
| T-E14-12 | OMS fault-injection suite | T-E14-01..11 | TODO |  | P4 M |
| T-E15-06 | Backup/restore and drill | T-E15-02 | TODO |  | P4 M |
| T-E15-07 | Audit export package | T-E14-01 | TODO |  | P4 S |
| T-E16-08 | Screens: opportunities, ticket, orders, positions | T-E16-01 | TODO |  | P4 L |
| T-E16-09 | Screens: risk, broker settings, config | T-E16-01, T-E03-03..05 | TODO |  | P4 M |
| T-E16-10 | Emergency interaction flow | T-E13-06, T-E16-08 | TODO |  | P4 M |
| T-E18-03 | Algo tagging and message counting | T-E18-01, T-E04-01, T-E05-01, T-E06-01 | GATED(G03) |  | P4 M. Algo registration above threshold is external. Limiter offline-OK. |
| T-E18-04 | Daily logout/expiry handling | T-E03-02..05 | TODO |  | P4 S |
| T-E18-05 | Webhook/postback ingress hardening | T-E06-09, T-E04-10, T-E05-09 | TODO |  | P4 S |
| T-E18-06 | Security test suite | T-E03-06, T-E18-05 | TODO |  | P4 M |

### M4 - Read-only live commissioning complete per broker (1 task)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E18-07 | Read-only live commissioning (per broker) | T-E04-12, T-E05-11, T-E06-11, G02, G05 | GATED(G02,G05,G06,G07,G13,G14) |  | P5 L. Run once per broker; needs credentials. |

### M5 - Live execution enabled per broker/product (1 task)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E18-08 | Live execution commissioning (per broker/product) | T-E18-07, T-E14-12, T-E13-06 | GATED(G01-G04,G08,G10-G12) |  | P5 L. Run once per broker and product; separately authorized, operator-attended. |

### M6 - Performance certified (6 tasks)

| ID | Task | Depends on | Status | Evidence / commit | Notes |
|---|---|---|---|---|---|
| T-E17-03 | CPU isolation deployment | T-E07-03 | TODO |  | P6 S |
| T-E17-05 | Scenario suite | T-E17-04, T-E07-06 | TODO |  | P6 M |
| T-E17-06 | Performance certification at 2x peak | T-E17-05, T-E18-07 | GATED(G05) |  | P6 M. Needs measured live feed peaks from M4. |
| T-E17-08 | Runbooks | T-E14-11, T-E15-06 | TODO |  | P6 M |
| T-E18-09 | Standby promotion and fencing drill | T-E14-03, T-E14-11 | TODO |  | P6 M |
| T-E18-10 | Operational review cadence | T-E18-02, T-E02-09 | TODO |  | P5 S |

## Update protocol

After finishing a task:

1. Check the Verification column of the task in doc 18. Every listed check and failure case has evidence (fixture, property/fuzz run, bench output, CI run).
2. Set the row: `Status = DONE`, `Evidence / commit` = commit hash plus path to fixtures or bench report. Definition of done (doc 20): code merged, acceptance met, fixtures committed, telemetry counters emitted, no new external gate silently assumed verified.
3. If a contract or schema changed: bump `schema_version` and update the owning planning doc (ownership rules in [docs/README.md](docs/README.md)).
4. Recompute **Next up**: list tasks whose dependencies are all `DONE`; prefer the critical path.
5. Update the milestone board counts. Mark a milestone `DONE` only when all its rows are `DONE`.
6. Add one line at the top of the [Log](#log).

Rules:

- Never mark a `GATED` row `DONE` without gate evidence (hash, reviewer, date). Use `DONE-OFFLINE` in the Notes and leave the status `GATED(...)`.
- Never edit doc 18 acceptance criteria to fit the code. If a criterion is wrong, record it under [Decisions and deviations](#decisions-and-deviations) and raise it with the owner.
- If a task is split, set `PARTIAL(scope)` and say what remains.
- If new work appears that is not in doc 18, add a row at the end of the right milestone table with an ID like `X-01` and a note why.

## Mapping gaps in the planning pack

Found while building this tracker; handle them as described.

1. **T-E03-06** (platform authentication and roles) is missing from doc 18's milestone map. It is tracked in M2 because T-E16-01 needs it.
2. **T-E17-07** appears in doc 18's M2 list and in the M6 range `T-E17-05..08`. It is tracked once, in M2.
3. **M2 tasks that depend on M3 tasks:**
   - T-E16-01 depends on T-E14-02.
   - T-E11-05 depends on T-E13-01.
   - T-E16-02, T-E16-03 and the rest of the UI chain depend on T-E16-01, so taken literally the whole read-only dashboard waits for the account actor.
   - Resolution (see D1): build the read-only subset of T-E16-01 first and mark it `PARTIAL(read-only)`. Finish the mutation endpoints after T-E14-02. T-E11-05 can start with strategy definitions and finish the execution wiring after T-E13-01.
4. **Circular dependency:** T-E13-05 depends on T-E14-08 and T-E14-08 depends on T-E13-05. Resolution (D2): build them as one unit. Deliver the protection state machine and trailing logic (T-E13-05) first against a stub exit interface, then T-E14-08 on top, then close both together with the combined verification from doc 18. Mark both `PARTIAL(...)` until then.
5. M4 and M5 rows (T-E18-07, T-E18-08) are executed per broker (and per product for M5). Track each broker/product as a sub-line in Notes, for example `Dhan: done 2026-..; Upstox: open; Zerodha: open`.

## Decisions and deviations

Record anything that departs from the pack: what, why, ADR link if one is needed in [doc 19](docs/planning/19-DECISIONS-AND-RISKS.md).

| # | Date | Decision | Reason | ADR |
|---|---|---|---|---|
| D1 | 2026-10-22 | Split T-E16-01: read-only endpoints (market snapshot, instruments, candles, options chains, opportunities, health, configurations, auth) first; mutation endpoints (order-previews, order-intents, modify/cancel, risk actions) after T-E14-02 | Doc 18 dependency chain would otherwise block the whole dashboard on the execution core. Proposed; confirm with owner | none yet |
| D2 | 2026-10-22 | Treat T-E13-05 and T-E14-08 as one unit (state machine first behind a stub exit interface) | Doc 18 lists each as the other's dependency. Proposed; confirm with owner | none yet |

## Log

Newest first. Format: `date | task | result | commit | evidence path`.

| Date | Task | Result | Commit | Evidence |
|---|---|---|---|---|
| 2026-10-22 | - | Tracker created from docs 17/18/20 (166 tasks) | - | - |
