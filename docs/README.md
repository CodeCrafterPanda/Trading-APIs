# Indian live trading platform — planning pack (revision 3, 22 October 2026)

Implementation blueprint for a greenfield, sub-millisecond, incrementally computed intraday platform with three independent broker adapters (**DhanHQ, Upstox, Zerodha Kite Connect v3**) and a push-synchronized React dashboard on TradingView Lightweight Charts. No application code; no account certified. Read in order; [00](planning/00-EXECUTIVE-OVERVIEW.md) defines the pack-wide conventions C1–C6 that every other document relies on.

| Doc | Owns | Read when |
|---|---|---|
| [00 Executive overview](planning/00-EXECUTIVE-OVERVIEW.md) | Scope, conventions, final decisions, invariants | First |
| [01 API research](planning/01-API-RESEARCH.md) | Canonical broker facts, budgets (D-*/U-*/Z-*), endpoints | Any broker question |
| [02 System architecture](planning/02-SYSTEM-ARCHITECTURE.md) | Topology, 15 workers, event flow, failover | Before any component |
| [03 Tech stack](planning/03-TECH-STACK-DECISIONS.md) | Rust/TS/PostgreSQL/ClickHouse selections | Kickoff |
| [04 Adapter contracts](planning/04-BROKER-ADAPTER-CONTRACTS.md) | Eight interfaces, types, enums, errors | Adapter work |
| [05 Market data engine](planning/05-MARKET-DATA-ENGINE.md) | Subscription planner, decoders, recovery | E07 |
| [06 Candle engine](planning/06-CANDLE-ENGINE.md) | History retrieval, tick-driven bars for all intervals, geometry, revisions, regeneration | E08 |
| [07 Sector heatmap](planning/07-SECTOR-HEATMAP.md) | Sector/stock metrics and ranking profiles | E09 |
| [08 Scanner catalog](planning/08-MARKET-SCANNER-CATALOG.md) | 56 detectors SC01–SC56 | E10 |
| [09 Regime and ranking](planning/09-REGIME-AND-RANKING-ENGINE.md) | Regime axes, candidate lifecycle, scoring | E11 |
| [10 Equity/futures/options](planning/10-EQUITY-FUTURES-OPTIONS.md) | Product rules, chain pipeline, strike selection | E12 |
| [11 Order execution](planning/11-ORDER-EXECUTION-ENGINE.md) | OMS, uncertainty, reconciliation, protection | E14 |
| [12 Risk management](planning/12-RISK-MANAGEMENT.md) | Gates, reservations, emergency modes | E13 |
| [13 Data architecture](planning/13-DATA-ARCHITECTURE.md) | Schemas, analytics tables, retention | E15 |
| [14 Dashboard UX](planning/14-DASHBOARD-UX.md) | Screens, WebSocket lane/delta/resync protocol, Lightweight Charts contract, control API | E16 |
| [15 Performance](planning/15-PERFORMANCE-ENGINEERING.md) | Named paths L1–L9, stage budgets, Indian workload envelope, throughput/memory/backpressure/staleness policies, benchmarks | Every hot-path and UI task |
| [16 Security and compliance](planning/16-SECURITY-AND-COMPLIANCE.md) | Algo framework, sessions, credentials, gates G01–G14 | Before any live route |
| [17 Roadmap](planning/17-IMPLEMENTATION-ROADMAP.md) | Phases P0–P6, milestones M0–M6 | Planning |
| [18 Granular tasks](planning/18-GRANULAR-DEVELOPMENT-TASKS.md) | Epics E01–E18, tasks T-Exx-nn with acceptance | Daily |
| [19 Decisions and risks](planning/19-DECISIONS-AND-RISKS.md) | ADR-00..12, limitations, risk register, open gates | Change control |
| [20 Developer handoff](planning/20-DEVELOPER-HANDOFF.md) | Repo layout, rules, starting sequence, runbooks | Day one |

Research provenance: [Dhan](research_notes/Indian%20trading%20platform%20blueprint/dhanhq.md), [Upstox](research_notes/Indian%20trading%20platform%20blueprint/upstox.md), [Zerodha](research_notes/Indian%20trading%20platform%20blueprint/zerodha.md), [market regulation and architecture](research_notes/Indian%20trading%20platform%20blueprint/market-regulation-and-architecture.md), [master prompt](research_notes/Indian%20trading%20platform%20blueprint/prompt.md). Revision log: [AUDIT-2026-10-22.md](AUDIT-2026-10-22.md).

Ownership rules: broker numbers live only in 01; latency, throughput, memory and staleness numbers only in 15; enums only in 04; gates only in 16; task IDs only in 18; UI transport and chart API detail only in 14. Other documents cite, never redefine.
