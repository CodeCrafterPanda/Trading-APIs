# Indian live trading platform blueprint

Issued **22 October 2026** (revision 2). Official documentation researched 21–22 October 2026 (Dhan, Upstox, NSE/SEBI) and 22 October 2026 (Zerodha Kite Connect v3). Timezone Asia/Kolkata. Status: **implementation specification; no application implemented or live account certified**.

Build a new standalone platform with **three independent, complete broker adapters — DhanHQ, Upstox and Zerodha Kite Connect** — each implementing all eight adapter interfaces. Use Rust for ingestion, market state, candles, sector intelligence, scanners, opportunity ranking, risk and order management; TypeScript/React for the dashboard; PostgreSQL for transactional truth; ClickHouse and Parquet for asynchronous analytics. The compute contract is **sub-millisecond**: every in-memory stage on the market path has a sub-ms p50 and p99 budget; only durable commit and network exceed 1 ms ([15](15-PERFORMANCE-ENGINEERING.md)).

## Pack-wide conventions

These apply to every document and are not restated elsewhere.

- **C1** Latency and capacity numbers are engineering targets under a declared load envelope, not broker or exchange guarantees.
- **C2** Weights, thresholds and parameters are tunable design defaults; scores, ranks and confidence are evidence quality, never probabilities of profit.
- **C3** Broker API acceptance is not exchange execution; ACK is not fill; cancel acceptance is not proof that no late fill exists.
- **C4** Evidence classes: *verified broker fact* (documented on the linked official page on the research date; not verified against this user's account), *proposed design* (this pack's decision), *external gate* (needs account access, licensing, circulars, broker clarification or measured deployment evidence). [01](01-API-RESEARCH.md) owns broker limits and endpoints; [15](15-PERFORMANCE-ENGINEERING.md) owns latency numbers; [16](16-SECURITY-AND-COMPLIANCE.md) owns gates G01–G14.
- **C5** No paper trading, backtesting or historical replay subsystem exists. Protocol fixtures, synthetic bursts and fault injection are verification tools only; historical candles serve live intelligence.
- **C6** No hardcoded stock, opportunity or trade quota. Broker quotas and owner-configured economic risk limits still apply; an empty opportunity set is a valid output.

## Scope and operating assumption

The initial deployment serves one investor operating their own enabled Dhan, Upstox and Zerodha accounts. This is an architectural assumption, not a determination of account eligibility; commercial, multi-client or algo-provider use requires a separate broker and regulatory route. Each broker is usable independently for market data, history, instruments, options, orders, order updates, portfolio and funds. The market-data provider and the execution account are configured independently per instrument group and per strategy; any pairing among the three brokers is permitted after canonical mapping and permission checks. An order is always exited on the broker/account that owns the position.

The platform covers NSE and BSE cash equities, indices as intelligence inputs, and enabled equity futures/options. Discovery, data coverage, scanner readiness and trading eligibility are distinct state dimensions; every discovered instrument stays visible in the catalog.

## Final decisions

| Concern | Selected design | Reason |
|---|---|---|
| Core language | Rust, pinned stable toolchain | One typed model, zero-allocation hot path, explicit ownership |
| Broker boundary | Three independent Rust adapter processes (Dhan, Upstox, Zerodha) | Separate credentials, decoders, reconnects and quotas; crash isolation |
| State ownership | Stable instrument shards; one writer per shard | Ordered incremental updates without locks |
| Execution ownership | One account actor and one fenced gateway per broker account | No competing reservations or submissions |
| Durability | PostgreSQL synchronous intent/reservation commit before network submission | Intent survives crashes and ambiguous sends |
| Options data | Provider chains (Dhan, Upstox) plus platform ChainBuilder with LOCAL_MODEL Black-76 IV/Greeks (required for Zerodha) | Zerodha documents no chain/Greeks endpoint; one chain contract for all brokers |
| Analytics | ClickHouse batches and compressed Parquet | Scans and bulk persistence stay off the decision path |
| UI | React, TypeScript, Vite, selector stores, virtualized tables, canvas charts | Incremental rendering with bounded views |
| Transport | Typed Unix-socket IPC, bounded queues; browser WebSocket deltas | No Kafka/Redis hop per tick |
| Deployment | Linux, pinned cores, approved static egress per broker, warm passive standby | Resource isolation and controlled takeover |
| Failover | Automatic validated market-data source switch among three adapters; execution pinned to the owning account | A different broker cannot close the original account's position |
| Order policy | LIMIT and SL-LIMIT automated initially; MARKET/SL-M only with broker market protection and certification | All three brokers now reject unprotected market orders |

Reasoning: [03](03-TECH-STACK-DECISIONS.md); placement: [02](02-SYSTEM-ARCHITECTURE.md); execution invariant: [11](11-ORDER-EXECUTION-ENGINE.md); ADR register: [19](19-DECISIONS-AND-RISKS.md).

## Broker capability summary

| Capability | Dhan | Upstox | Zerodha |
|---|---|---|---|
| Live feed | Binary LE, 5 sockets × 5,000, ticker/quote/full, 20/200-depth sockets | Protobuf V3, 2 sockets (Plus 5), mode-dependent per-user caps, D30 | Binary BE, 3 sockets × 3,000, ltp/quote/full, index packets |
| History | 1/5/15/25/60 min, daily; 90 days/request | 1–300 min, 1–5 h, day/week/month; calendar windows | minute…60minute, day; staff-published caps; paid Connect tier |
| Option chain + Greeks | Yes (3 s cadence per pair) | Yes (+ Greeks REST, OI/PCR analytics) | None → platform ChainBuilder + LOCAL_MODEL |
| Order updates | Account-wide socket + unsigned postback | Portfolio stream + unsigned webhook | Socket text frames + **checksum-signed** postback |
| Native protection | Super Order, Forever/OCO (CNC/MTF) | GTT multi-rule, beta trailing | GTT single/two-leg OCO (LIMIT only), alerts-to-order |
| Bulk/emergency | Exit All; kill switch (flat required) | Exit All ≤50, cancel-multi ≤50, kill switch (12 h cooldown) | None documented (per-order reductions) |
| Idempotency | correlationId lookup only | tag lookup only | tag (local filter) only |

Full matrix, budgets and endpoints: [01](01-API-RESEARCH.md).

## Required outcomes

Streaming from any adapter produces normalized, provenance-bearing observations with explicit data-quality states. Candles retain their origin (broker history, local stream, local aggregate). Sector and stock rankings update incrementally; scanners activate only when dependencies change; opportunities expire, downgrade or invalidate as conditions change.

Equities support intraday long/short routes with quantity, tick, band and product checks. Futures use dated contracts, lots, freeze quantities, margins and expiry controls. Options use chain/contract discovery, strike liquidity, IV/Greeks with source provenance, underlying and premium exits, and non-atomic multi-leg controls. Manual and automated intents pass the same risk and OMS gates.

The dashboard exposes market coverage, stale fields, sector/stock intelligence, scanner evidence, options, orders, positions, account risk, per-broker credential status and performance. A green connectivity indicator alone never means data is complete or trading is safe.

## Non-negotiable invariants

1. No order leaves a gateway before its durable intent and reservation exist.
2. A potentially transmitted command with no definitive response becomes **SUBMISSION_UNKNOWN**; it is reconciled, never blindly retried, and never re-routed to another broker.
3. Tags/correlation IDs support matching; no broker documents exactly-once placement.
4. An exit always targets the broker/account/product owning the position.
5. No credential, PIN, TOTP or broker token reaches the browser or ordinary logs.
6. Unknown account state, unsupported session rules or insufficient critical data blocks new risk.
7. Risk checks are never removed to improve a latency number.
8. Unsigned postbacks/webhooks never release risk on their own; only Zerodha's checksum-verified postback is authenticated input.
9. Application emergency stop first blocks new exposure and manages outstanding risk; broker kill switches (Dhan, Upstox) require flat positions and Zerodha has none.
10. MARKET/SL-M orders carry broker market protection or are not sent.

## Material limits and gates

Dhan native intraday intervals are 1/5/15/25/60 minutes; Zerodha lacks 2-minute bars; all requested intervals come from local one-minute aggregation with provenance ([06](06-CANDLE-ENGINE.md)). Subscription capacity is a per-broker budget ([01](01-API-RESEARCH.md)); the planner allocates it by priority, never by a fixed universe. Zerodha's streaming and history require the paid Connect tier (G13), its access token expires at 06:00, and its static IP applies to order endpoints with one change per week. Session configuration is effective-dated: current CAS rules and the December 2026/February 2027 changes stay separate ([16](16-SECURITY-AND-COMPLIANCE.md)).

## Acceptance and success measures

Compute: frame receive → candidate decision p50 0.5 ms, p95 2 ms, p99 5 ms; decision → request write including durable commit p50 2 ms, p95 5 ms, p99 10 ms; every stage budget in [15](15-PERFORMANCE-ENGINEERING.md). Record sample count, observed maximum, queue age, event loss, CPU, memory and clock uncertainty; measure broker ACK and fill distributions separately per broker.

Functional: all eight interfaces implemented for all three brokers with independent conformance evidence; instrument mapping without ambiguity; explainable scanner outputs; no submission on unknown state; correct partial-fill accounting; successful crash/reconnect reconciliation; data-source failover between any two adapters without baseline contamination. Read-only commissioning precedes separately authorized live commissioning per broker and product.

## Delivery and next action

Follow [17](17-IMPLEMENTATION-ROADMAP.md), implement the ordered tasks in [18](18-GRANULAR-DEVELOPMENT-TASKS.md), and use [20](20-DEVELOPER-HANDOFF.md) as the execution checklist. The first milestone is a versioned contract and instrument foundation with three read-only adapters certified against fixtures; execution is enabled only after account, compliance, durability and reconciliation evidence is complete ([19](19-DECISIONS-AND-RISKS.md) lists open gates).
