# Market-data ingestion, coverage and recovery

The engine consumes the supported WebSocket feeds of all three adapters continuously and updates only affected state. REST snapshots initialize or repair state; they never replace streaming. Broker limits and modes are governed by [01](01-API-RESEARCH.md); normalized events by [04](04-BROKER-ADAPTER-CONTRACTS.md); latency budgets by [15](15-PERFORMANCE-ENGINEERING.md).

## Universe and adaptive subscription planner

Maintain four independent dimensions per canonical instrument: DISCOVERED, DATA_COVERED, SCANNER_READY and TRADE_ELIGIBLE. A discovered contract need not fit current subscription capacity; an eligible trade requires fresh mapped data and certified account/product permissions. UI counts expose all four populations and exclusion reasons.

The planner owns a desired subscription set per adapter and the adapter reports the observed set. Capacity is a per-broker budget from [01](01-API-RESEARCH.md):

| Adapter | Budget | Modes and field coverage |
|---|---|---|
| Dhan | D-LIVE: 5 sockets × 5,000 instruments, 100 per subscribe message; D-DEPTH20/D-DEPTH200 separate sockets | ticker (LTP/LTT only), quote (+last qty, ATP, cumulative volume, buy/sell qty, OHLC), full (+OI, 5 depth) |
| Upstox | U-LTPC / U-GREEKS / U-FULL per-user key ceilings, shared across U-LIVE sockets; U-PLUS D30 | ltpc (no volume/depth), option_greeks, full (ATP, VTT, OI, IV/Greeks, 5 depth, candles), full_d30 |
| Zerodha | Z-LIVE: 3 sockets × 3,000 instruments per api_key | ltp (price only), quote (+last qty, ATP, day volume, buy/sell qty, OHLC), full (+LTT, OI, exchange time, 5 depth); index packets carry OHLC/change only, never volume or depth |

Priority classes: open positions and protective monitoring; pending orders; actionable candidates; enabled scanner dependencies; general universe coverage; optional deep analysis; viewport-only requests. Within a class use required fields, liquidity relevance, marginal scanner coverage and recency of denied coverage. Priority never bypasses a broker ceiling.

Choose the cheapest mode satisfying the dependency mask on the chosen adapter: price-only scanners can use Dhan ticker / Upstox ltpc / Zerodha ltp; volume, VWAP, OI and depth scanners require quote/full. Zerodha ltp and Upstox ltpc cannot feed a volume scanner; a Zerodha or Dhan index subscription cannot feed anything needing volume. A contract requested by several scanners is subscribed once with the union of fields. Promote/demote with a 5-second minimum residency except urgent protection. Capacity decisions are revisioned and deterministic for the same demand/configuration.

Under insufficient quota on the active adapter, the planner may place overflow coverage on another adapter as a separate source epoch (never merged counters), retain protective coverage, degrade optional depth, then time-slice lower-priority coverage only if its scanners permit discontinuity. Time-slicing cannot claim continuous momentum, cumulative-volume allocation or gap-free bars. Publish coverage age, missing fields and dropped/deferred demand. Never create extra accounts/apps to evade per-user constraints.

## Session state machine

~~~mermaid
stateDiagram-v2
  [*] --> DISCONNECTED
  DISCONNECTED --> AUTHORIZING
  AUTHORIZING --> CONNECTING
  CONNECTING --> SNAPSHOT
  SNAPSHOT --> WARMING
  WARMING --> LIVE
  LIVE --> DEGRADED: stale or queue gap
  DEGRADED --> RECOVERING
  LIVE --> RECOVERING: socket lost
  RECOVERING --> AUTHORIZING: token or URI renewal
  RECOVERING --> SNAPSHOT
  AUTHORIZING --> BLOCKED: permission unavailable
  BLOCKED --> AUTHORIZING: new approved credentials
~~~

Socket connectivity and instrument freshness are separate. Transport liveness timers are broker-specific: Dhan server ping every 10 s, disconnect after 40 s without pong, code 805 when a sixth socket displaces the first; Upstox idle WebSocket ping frames answered with pong; Zerodha one-byte heartbeat "every couple of seconds" when no data is pending (ignored as data, counted as liveness) and JSON text frames (`order`, `error`, `message`) on the same socket, routed to the order-update path. A quiet contract on a live socket has no current price until an observation arrives. [Dhan feed](https://dhanhq.co/docs/v2/live-market-feed/), [Upstox feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/), [Kite WebSocket](https://kite.trade/docs/connect/v3/websocket/).

## Framing, decoding and units

Timestamp receipt with local monotonic time before any work. Validate frame length, payload kind and declared packet boundaries; cap accepted frame size from the documented schema plus a conservative allowance. One decoder per wire format, each a separate conformance-fixture suite:

| Format | Framing | Packets | Units |
|---|---|---|---|
| Dhan ordinary | Little-endian 8-byte header (code, length, segment, security ID); concatenated packets iterated by length | ticker 2, quote 4, OI 5, prev-close 6, full 8 (5 depth × 20 B), disconnect 50 | float32 prices, int32 quantities, LTT epoch seconds |
| Dhan depth | Separate 12-byte header schema on dedicated sockets | 20-level (sequence field documented as ignored) and 200-level | float64 price, uint32 qty/orders |
| Upstox V3 | Protobuf from the pinned descriptor; mode-aware field validity | ltpc, full, option_greeks, index, market status, initial snapshot | Use `bidAskQuote.bidQ/askQ` and `marketOHLC.ohlc[].vol` (int64); `ano/bno/fp/fv/dhoi/dloi` are removed; `ltt` and `currentTs` distinct |
| Zerodha | Big-endian: int16 packet count, then per packet int16 length + body; many instruments per frame | ltp 8 B, quote 44 B, full 184 B (10 × 12 B depth: qty int32, price int32, orders int16, 2 pad); index quote 28 B, index full 32 B | Prices int32 in paise ÷100; currency ÷10,000,000; BCD ÷10,000 and segment = `instrument_token & 0xff` are SDK-derived facts, certified by captured frames; LTT and exchange timestamp are unsigned Unix seconds |

Map raw identifiers through an immutable master-version lookup (Dhan segment+securityId, Upstox instrument_key, Zerodha instrument_token with the daily CSV snapshot that issued it). Unresolved or reused tokens are quarantined; no symbol-string fallback for trading. Validate finite values, nonnegative volumes/depth quantities, bid/ask ordering and plausible timestamp ranges. A crossed/locked book is flagged, never "fixed". Missing fields retain missingness rather than overwriting prior values with zero.

Normalize prices to fixed scale and volume to certified units. Index volume/OI/depth are NOT_APPLICABLE. Preserve provider ATP, previous close, cumulative/day volume, OI and depth separately from derived metrics. Decoding obeys the zero-allocation and no-syscall rules in [15](15-PERFORMANCE-ENGINEERING.md) and its frame→normalized budget (p50 0.05 ms, p95 0.20 ms, p99 0.50 ms).

REST snapshot sources, used for initialization and repair only and scheduled by their budgets: Dhan `POST /marketfeed/ltp|ohlc|quote` ≤1,000 instruments at 1/s (D-QUOTE); Upstox `GET /v3/market-quote/quotes` ≤500 keys (full incl. depth, OI, circuits, CAS fields), `/ohlc`, `/ltp` (U-QUOTE); Zerodha `GET /quote` ≤500 keys at 1/s, `/quote/ohlc` and `/quote/ltp` ≤1,000 (Z-QUOTE). Missing keys in a Zerodha response are silently omitted and must be detected locally.

## In-memory state and event discipline

Each shard stores latest field value, field timestamp, validity bits, source epoch, local sequence, cumulative-volume baseline, depth snapshot, rolling feature state, active bar accumulators and scanner subscriptions, in dense arrays indexed by shard-local index. Updates apply in adapter output order. Exchange timestamps determine candle placement where verified (Zerodha exchange timestamp, Upstox ltt, Dhan LTT); receipt order alone cannot establish exchange ordering.

No universal exchange sequence, replay cursor or complete trade tape is documented for any of the three retail feeds. Gap detection is **suspected gap detection** from disconnects, queue overflow, timestamp discontinuity, cumulative resets and missing subscription coverage. Never expose "lossless" health from local_sequence continuity.

Do not sum last-traded quantity on repeated LTP; prefer cumulative/day-volume deltas where valid. OI change uses the latest comparable OI snapshot. Depth is a snapshot at its own age; it does not prove executable liquidity.

## Backpressure and isolation

Ingress/decode/state queues have fixed capacities and age budgets from [15](15-PERFORMANCE-ENGINEERING.md). UI, optional deep analytics and archival writes shed first. Observations needed by candles are not coalesced. If unavoidable loss occurs, emit a durable quality-gap record, increment source epoch where continuity is broken, invalidate dependent windows and recover.

Execution updates (including Zerodha socket text frames) use a distinct reserved channel with a bounded durable spool. If memory and spool exhaust, stop new entries, mark an audit gap, fence uncertain execution activity and reconcile authenticated broker snapshots before resuming.

## Reconnect and recovery procedure

1. Mark affected source/instruments stale immediately; invalidate executable candidates; alert protection logic.
2. Reconnect with exponential jittered delay, initially 250 ms, capped at 30 s, within provider constraints and socket counts.
3. Credentials: Dhan token renewal only for Web-origin tokens; Upstox obtain a fresh single-use authorized URI, never reuse a code; Zerodha access tokens expire at the 06:00 boundary and are invalidated by a login elsewhere, so a TokenException on any Zerodha call routes to the credential coordinator for interactive re-login and all Zerodha sockets reconnect after it. [Upstox authorize](https://upstox.com/developer/api-documentation/get-market-data-feed-authorize-v3/), [Kite user](https://kite.trade/docs/connect/v3/user/)
4. Reconcile desired/observed subscriptions in legal batches (Dhan 100/message; Zerodha JSON `subscribe` + `mode` messages; Upstox binary sub/change_mode), prioritizing positions and pending orders. No adapter documents subscription persistence across reconnect; resubscribe everything.
5. Receive initial snapshots (Upstox) or bounded REST snapshots; validate identity, time, session and price scale.
6. Increment source epoch; establish cumulative baselines; request missing bars with overlap ([06](06-CANDLE-ENGINE.md)).
7. Warm each dependency independently; restore entry eligibility only when its readiness mask passes.

Read retries use bounded exponential backoff and quotas. A reconnect storm cannot starve order updates or reconciliation. Circuit breakers are per endpoint/credential/scope; a data outage on one broker never disables a working cancel route on any broker.

## Data-source failover

Candidate alternate sources must share canonical venue/contract, certified units, relevant session, sufficient field coverage and acceptable age. Rank alternates by configured preference, then field coverage (a Zerodha quote-mode source cannot replace a Dhan full-mode source for an OI scanner; an Upstox ltpc source cannot replace anything needing volume), then freshness. Compare price discrepancy against a tolerance related to tick/spread/volatility; large discrepancy keeps both visible and blocks new risk.

Switch source epoch atomically per dependency group. Reset cumulative volume/turnover and OI baselines; invalidate continuity-dependent features. Existing orders retain their broker/account and native protection; a market-data switch never moves an intent to another broker ([11](11-ORDER-EXECUTION-ENGINE.md)).

## Acceptance

Golden and malformed fixtures for every packet mode of all three formats (including Zerodha index packets, segment divisors and text frames; Upstox deprecated-field absence; Dhan concatenated packets and depth header); mixed subscriptions under each quota; token rotation and Zerodha 06:00 expiry; heartbeat loss per broker; source switching between every adapter pair; repeated last quantities; missing OI; sequence absence; timestamp skew; silently omitted Zerodha quote keys. Load tests must show bounded RSS and queue age at twice the combined measured entitled-feed peak. All loss/degradation must be visible in health and scanner readiness. Live read-only certification compares representative NSE/BSE equity, index, future and option instruments across the three brokers without assuming enums prove account coverage.
