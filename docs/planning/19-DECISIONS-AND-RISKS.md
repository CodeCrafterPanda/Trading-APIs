# Architectural decisions, limitations, risks and external gates

Record of closed decisions (ADRs), documented limitations, the risk register and the external requirements that cannot be verified without account access. A decision is reopened only through a new ADR with measured evidence ([03](03-TECH-STACK-DECISIONS.md)).

## Architecture decision records

| ID | Decision | Context | Alternatives rejected | Consequences | Change trigger |
|---|---|---|---|---|---|
| ADR-00 | Three first-class adapters: DhanHQ, Upstox, Zerodha | The master prompt's title and API section name three brokers (Kite Connect docs included) while §2–3 still say "both DhanHQ and Upstox"; the later edit added Zerodha | Dual-broker only; Zerodha as data-only | Every interface implemented three times; conformance suite runs per broker; Zerodha gaps (chain, Greeks, kill switch) filled by platform substitutes | Owner removes a broker in writing |
| ADR-01 | Rust core for ingestion, state, candles, scanners, risk, OMS; TypeScript/React UI | Sub-ms compute contract, binary decoding, no GC pauses | Go core; Python core; Rust+Go | No official Rust SDKs; adapters implement documented wire contracts directly; higher discipline | Measured bottleneck Rust cannot fix, with ADR |
| ADR-02 | PostgreSQL synchronous-commit ledger on the execution path | External action must be known after crash | Async commit; in-memory ledger with periodic snapshot; analytics store as truth | 1–5 ms commit inside decision→send budget; slow commit delays, never bypasses | Never for durability; topology change requires remeasurement |
| ADR-03 | ClickHouse + Parquet for analytics/archive | Tick/depth scans must not compete with ledger latency | PostgreSQL-only; files only | Separate failure domain; background TTL monitoring | Analytics volume too small to justify, measured |
| ADR-04 | No Redis/Kafka initially; Unix-socket framed IPC and bounded in-process queues | Single active host; every tick through a broker hop adds latency and operations | Kafka backbone; Redis cache | Multi-host fan-out deferred; IPC cost measured | Measured multi-host requirement |
| ADR-05 | One writer per instrument shard and per broker account; fenced execution gateway with owner epoch | Ordered updates without locks; stale process may hold valid broker token | Shared mutable state with locks; database lease alone | Controlled shard migration; standby promotion needs egress fencing | None expected |
| ADR-06 | LOCAL_MODEL Greeks (Black-76 on futures/forward, bounded IV solver) as a first-class source | Zerodha has no chain/IV/Greeks endpoint; provider units differ between Dhan and Upstox | Provider-only Greeks; third-party data vendor | `model_source` on every Greek; convergence flag gates IV detectors; cross-check against providers | Licensed analytics feed adopted |
| ADR-07 | One-minute canonical local aggregation base; native coarser intervals retained separately | Dhan 25-minute is not 30; Zerodha lacks 2-minute; Upstox supports 1–300 | Per-provider native intervals as base | 2/3/10/30-minute bars are local aggregates with provenance | None |
| ADR-08 | LIMIT and SL-LIMIT as initial automated order policy; MARKET/SL-M only with certified protection | Unprotected market orders prohibited; Zerodha requires market_protection; phase-specific exceptions | MARKET default | Protected-market variants gated per broker/segment/phase | Certified protection evidence per route |
| ADR-09 | Market-data failover automatic after validation; execution routing explicit and account-pinned | A different broker cannot close the original account's position | Cross-broker execution failover | Source epochs and baseline resets; exits always route to owning account | None |
| ADR-10 | Sub-millisecond compute contract ([15](15-PERFORMANCE-ENGINEERING.md)) | Prompt priority on millisecond-scale decisions | Millisecond-class targets | Zero-allocation hot paths, CPU pinning, no syscalls on tick path | Measured impossibility on reference hardware |
| ADR-11 | Webhooks/postbacks untrusted unless cryptographically signed; only Zerodha postbacks (SHA-256 checksum) count as authenticated | Upstox webhook policy is a URL filter; Dhan postback signing not documented | Trust all broker callbacks | Dhan/Upstox callbacks are redundant deduplicated inputs corroborated by authenticated streams/REST | Broker publishes a signing scheme |
| ADR-12 | Versioned length-delimited frames over Unix domain sockets between adapter processes and core | Process isolation per broker; binary typed IPC without JSON | Shared memory rings; gRPC | Schema negotiation at connect; bounded buffers; disconnect semantics | Profiling shows IPC dominates the frame→state budget |
| ADR-13 | Browser transport: one WebSocket per client, JSON field-level deltas on four lanes (P0 account, P1 event, P2 display, P3 background), per-subscription sequence, snapshot/cursor resync, server-enforced throttling ([14](14-DASHBOARD-UX.md)) | Browser WebSocket has no backpressure; order/risk events must never be coalesced while display values may be; token-efficient and debuggable encoding wanted first | Row-level full upserts; binary (MessagePack/Protobuf) from day one; SSE + REST polling; GraphQL subscriptions | Encode budget per row in [15](15-PERFORMANCE-ENGINEERING.md); publisher isolated from shards; client store applies patches per animation frame | Measured encode/parse time dominating L7 or L9 → binary encoding ADR |
| ADR-14 | TradingView Lightweight Charts 5.x behind a platform `ChartAdapter`; live bars via `update()` only, history via paged `setData`, corrections via `update(bar, true)`, annotations via price lines, `createSeriesMarkers` and series primitives | Official, Apache-2.0, canvas-rendered, incremental API verified against 5.2.1; annotation needs (zones, order blocks) exceed built-ins | Full TradingView Charting Library (licensing); custom WebGL charts; SVG/D3 | Attribution required; exact release pinned; IST shift at adapter boundary; adapter tests fail on API drift | Required capability missing or license change |
| ADR-15 | Tick-driven active bars for every enabled interval in one shard apply, reconciled against child aggregation at child close | All timeframes must advance on each tick and stay mutually consistent; waiting for 1-minute close adds up to 60 s latency to higher intervals | Parents updated only on child close; browser-side aggregation | O(T) per observation within L3; `ACCUMULATOR_MISMATCH` flag; child-derived bar authoritative when final | None expected |

## Known limitations

| Area | Limitation | Handling |
|---|---|---|
| Feeds | All three retail feeds are sampled snapshots; no exchange sequence, trade tape, replay cursor or delivery guarantee is documented | Suspected-gap detection, cumulative-volume deltas, bar precision flags |
| Execution | No broker documents an idempotency key or exactly-once placement (Zerodha `guid` is not one) | Durable intent before send; SUBMISSION_UNKNOWN reconciliation |
| Zerodha | No option chain, IV, Greeks, PCR, kill switch, exit-all, bracket, trailing stop; instruments CSV lacks ISIN/freeze quantity; history caps are staff-published; data needs Connect tier; token expires at "6 AM" without timezone label | ChainBuilder + LOCAL_MODEL; ENTRY_HALTED + reductions; freeze from NSE contract file; ISIN from holdings/external master; truncation detection; G13; renew before 06:00 IST |
| Upstox | Webhook unsigned; Plus D30 table ambiguity (50 vs 1500); exit-all limited to 50 positions during market hours; ordinary token expires at 03:30 without labeled timezone; analytics token GET-only | ADR-11; enforce 50/connection; per-item 207 handling; confirm expiry metadata; separate read credential |
| Dhan | Forever/OCO documented for CNC/MTF only; deep-feed connection sharing undocumented; bid/ask prose inconsistencies in depth docs; 24-hour token with Web-origin-only renewal; conditional triggers equities/indices only | Protection certification per product; fixture certification; credential coordinator; optional use |
| Regulation | BSE operational circulars not independently fetched; SEBI December 2026/February 2027 changes pending; algo message counting semantics per broker unconfirmed | Effective-dated rule sets with activation gates; conservative counting |
| Data rights | Broker feed access does not establish redistribution, index constituent/weight licensing or perpetual storage | G09; single-owner deployment assumption |
| Performance | No benchmark or live latency measured; sub-ms budgets and the reference workload (20 k/s mean, 80 k/s burst) are design assumptions | M6 certification; capacity report on failure; workload replaced by measured peaks at M4 |
| Frontend | Browser paint, WebSocket delivery and JSON parsing are not sub-ms; tick→screen is bounded (L8/L8c), not sub-ms | Measured and reported separately; never blocks CALC/EXEC |
| Charts | Lightweight Charts formats UTC only, has no built-in zone/rectangle drawing, and its plugin API evolves across minor releases | IST shift at adapter; series primitives for zones; pinned release with drift tests |

## Risk register

| ID | Risk | L/I | Mitigation | Owner | Detection signal |
|---|---|---|---|---|---|
| R-01 | Duplicate order after ambiguous send | M/H | SUBMISSION_UNKNOWN hold, correlation/tag lookup, no blind retry, worst-case reservation | OMS | unknown_submissions > 0 |
| R-02 | Stale process keeps submitting after failover | L/H | Owner epoch, fail-closed gateway, egress fencing before promotion | Ops | Two heartbeats for one account |
| R-03 | Decoder drift (Upstox proto change, Dhan packet change, Zerodha divisor) | M/H | Pinned descriptor hash, golden/malformed fixtures, SCHEMA_DRIFT quarantine, weekly doc review | Adapter owners | decode_quarantine_rate |
| R-04 | Feed quota exhaustion hides coverage loss | M/M | Four coverage dimensions, priority classes, visible deferred demand | Market engine | DATA_COVERED/DISCOVERED ratio drop |
| R-05 | Token expiry mid-session (Dhan 24 h, Upstox 03:30, Zerodha 06:00, login-elsewhere invalidation) | H/M | Credential coordinator, pre-expiry renewal, entry halt on REAUTH_REQUIRED | Security | auth_expired events |
| R-06 | Rate-limit breach (10 OPS exchange threshold, 400/min, 25 modifications) | M/H | Smooth limiter per window, reserved protective capacity, modification cap 20 | OMS | HTTP 429 / DH-904 counts |
| R-07 | Local Greeks diverge from provider | M/M | Fixture agreement tests, convergence flags, Q=0 on non-convergence | Derivatives | greeks_residual histogram |
| R-08 | Candle volume misallocation across gaps | H/L | UNKNOWN/PARTIAL flags, no even distribution, overlap repair | Candle engine | bars_with_partial_volume |
| R-09 | Unsigned webhook spoofing | L/H | Controlled ingress, schema/account validation, REST corroboration, never create intents | Security | webhook_rejected |
| R-10 | Protection cannot be established after fill | M/H | Block further entry fills, urgent alert, certified fallback reduction | Risk | unprotected_units > 0 |
| R-11 | Session rule misapplied (CAS, random close, pre-open market orders) | M/H | Effective-dated rules, broker phase/status feeds, unknown → block entries | Compliance | session_rule_conflicts |
| R-12 | Ledger commit latency exceeds budget | M/M | Local NVMe sync commit, dedicated pool, intent expiry | Ops | commit_p99 > 5 ms |
| R-13 | Overload degrades risk/order updates | L/H | Reserved channels, overload ladder, bounded durable spool | Performance | execution_spool_depth |
| R-14 | Cross-broker exposure confusion (position on A, exit sent to B) | L/H | Account-pinned exits, position identity includes broker | OMS | reconciliation diff position_mismatch |
| R-15 | Daily-loss or concentration limit breached by gap move | M/H | Reduction plan, global halt, Greeks/stress limits | Risk | daily_loss_utilization |
| R-16 | Regulatory change invalidates order-type policy or hours | M/M | Daily rule check, weekly doc review, activation gates | Compliance | rule_review_expired |
| R-17 | Data-rights violation through multi-user exposure | L/H | Single-owner assumption, G09 before any sharing | Owner | External review |
| R-18 | Sub-ms targets unattainable on chosen hardware | M/M | CPU pinning, allocation audit, capacity report, tighter admission | Performance | p99 budgets at M6 |
| R-19 | Zerodha Personal app lacks data/order-update channels | M/M | G13/G14 before enabling; Connect tier | Owner | profile/entitlement check |
| R-20 | Instrument token reuse after expiry (Zerodha, Upstox exchange tokens) | M/M | Daily master snapshots, master_version in every mapping, quarantine reused tokens | Instruments | mapping_reused_token |
| R-21 | Browser saturation at large visible universes (many charts/tiles) degrades L9 and hides fresh data | M/M | Virtualization, dirty-tile canvas, rAF batching, Web Worker decode, server cadence reduction, STALE banner | UI | render_frame_p95 > 16 ms; superseded_updates rising |
| R-22 | Multi-timeframe accumulator diverges from child aggregation (ordering, late ticks, epoch resets) | M/M | Child-derived bar authoritative at close; `ACCUMULATOR_MISMATCH` counter; property tests | Candle engine | accumulator_mismatch > 0 |
| R-23 | Chart library API drift or license change breaks the adapter | L/M | Exact release pin, adapter contract tests, ADR before upgrade | UI | adapter test failure on dependency bump |
| R-24 | Memory budget exceeded as covered universe or enabled intervals grow | M/M | Per-instrument/process budgets, bounded lookbacks, RSS alarms 75/85/95%, coverage halt | Performance | rss_utilization |

## Unresolved external requirements

| Gate | Evidence required | Blocks |
|---|---|---|
| G01 Account route | Owner/broker confirmation of self-developed (tech-savvy client) vs provider category, per broker | All execution |
| G02 Auth and IP | Account identity, token flow per broker, registered static egress (Dhan all order APIs; Upstox user-level IPs; Zerodha order endpoints, weekly change cap), daily logout behavior | Account mutations |
| G03 Algo controls | Generic tag/algo ID, registration status if >10 OPS, broker message-counting semantics (slices, modifications) | Automated/API execution |
| G04 Segment/products | Enabled NSE/BSE segments, intraday shorts, MTF, delivery sale authorization (DDPI/TPIN/holdings authorise) | Affected routes |
| G05 Data entitlement | Dhan Data API status/expiry, deep-feed connections; Upstox Plus; Zerodha Connect tier | Affected feeds/scanners |
| G06 Units/schema | Tick/price/volume/Greek/timestamp fixtures from real responses per broker | Affected calculations/orders |
| G07 Session/BSE | Operative NSE/BSE circulars, holiday calendar, broker square-off deadlines | Affected venue/session |
| G08 Protections | Dhan Super/Forever, Upstox GTT/trailing, Zerodha GTT OCO partial-fill and modify behavior | Strategies requiring native protection |
| G09 Data rights | Storage/display/derived-data and index constituent licensing evidence | Disallowed use/distribution |
| G10 Durability/recovery | Fencing drill, synchronous ledger, restore + reconciliation test | Live mutations |
| G11 Live conformance | Separately authorized read-only then live commissioning evidence per broker/product | Production enablement |
| G12 Risk values | Owner-approved account/global limits and emergency policy | All execution |
| G13 Zerodha app tier | Personal vs Connect, data entitlement, billing/waiver status | Zerodha market data and history |
| G14 Zerodha order-update channel | Whether the app type receives socket order frames and HTTP postbacks | Zerodha live order tracking; without it, snapshot reconciliation only and entries blocked |

Each gate carries status OPEN|VERIFIED|EXPIRED|REVOKED with evidence hash, reviewer and review dates ([16](16-SECURITY-AND-COMPLIANCE.md)). This pack marks none as verified.
