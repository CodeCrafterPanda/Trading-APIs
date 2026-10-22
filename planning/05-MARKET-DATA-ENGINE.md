# Market-data ingestion, coverage and recovery

The engine consumes supported WebSocket feeds continuously and updates only affected state. REST snapshots initialize or repair state; they do not replace live streaming. Broker limits and exact modes are governed by [01](01-API-RESEARCH.md), and normalized events by [04](04-BROKER-ADAPTER-CONTRACTS.md).

## Universe and adaptive subscription planner

Maintain four independent dimensions per canonical instrument: DISCOVERED, DATA_COVERED, SCANNER_READY and TRADE_ELIGIBLE. A discovered contract need not fit simultaneously into current subscription capacity. An eligible trade requires fresh mapped data and certified account/product permissions. UI counts expose all four populations and reasons for exclusions.

The planner owns a desired subscription set and the adapter reports the observed set. Priority classes are: open positions and protective monitoring; pending orders; actionable candidates; enabled scanner dependencies; general universe coverage; optional deep analysis; viewport-only requests. Within a class use required fields, liquidity relevance, marginal scanner coverage and recency of denied coverage. Priority never bypasses broker ceilings.

Choose the cheapest mode satisfying the dependency mask. Price-only scanners can use LAST_PRICE; volume/VWAP/OI/depth scanners require richer modes. A contract requested by several scanners is subscribed once with the union of fields. Promote/demote with a 5-second proposed minimum residency to avoid churn, except urgent protection. Capacity decisions are revisioned and deterministic given the same demand/configuration.

Under insufficient quota, retain protective coverage, degrade optional depth, then time-slice lower-priority observational coverage only if its scanners permit discontinuity. Time-slicing cannot claim continuous momentum, cumulative-volume allocation or gap-free bars. Publish coverage age, missing fields and dropped/deferred demand. Never create extra accounts/apps to evade per-user constraints.

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

Socket connectivity and instrument freshness are separate. Heartbeats establish transport activity, not that a quiet contract has current price/depth. Dhan documents ping/pong behavior; Upstox supports idle ping frames. Keep the protocol timers broker-specific. [Dhan feed](https://dhanhq.co/docs/v2/live-market-feed/), [Upstox feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/).

## Framing, decoding and units

Timestamp receipt with local monotonic time before expensive work. Validate frame length, payload kind and declared packet boundaries; cap maximum accepted frame size from observed schema plus conservative allowance. Dhan ordinary and full-depth decoders are separate. Iterate concatenated packets by validated length; reject overrun/truncation without reading outside the buffer. Upstox uses the official pinned Protobuf descriptor and mode-aware field interpretation.

Map raw identifiers through an immutable master-version lookup. Unresolved or reused tokens are quarantined; no symbol-string fallback for trading. Validate finite numeric values, nonnegative volumes/depth quantities, bid/ask ordering and plausible timestamp ranges. A crossed/locked book is flagged for policy handling rather than silently “fixed.” Missing fields retain missingness instead of overwriting previous values with zero.

Normalize prices to fixed scale and volume to certified units. Index volume/OI can be NOT_APPLICABLE. Preserve provider ATP, previous close, cumulative volume, OI and depth separately from internal derived metrics. Broker float parsing and raw units are covered by conformance fixtures.

## In-memory state and event discipline

Each shard stores latest field value, field timestamp, validity bits, source epoch, local sequence, cumulative-volume baseline, depth snapshot, rolling feature state, active bar accumulators and scanner subscriptions. Updates apply in adapter output order. Exchange timestamps determine candle placement where verified; receipt order alone cannot establish true exchange ordering.

No universal exchange sequence or complete trade delivery guarantee is documented for these retail feeds. Gap detection is therefore **suspected gap detection** using disconnects, queue overflow, timestamp discontinuity, cumulative resets and missing subscription coverage. Never expose “lossless” health from local_sequence continuity.

Do not sum last-traded quantity every time LTP repeats. Prefer cumulative-volume differences where valid. OI change uses the latest comparable OI snapshot, not assumed per-trade updates. Depth is a snapshot at its own age; it does not prove executable liquidity remains available.

## Backpressure and isolation

Ingress/decode/state queues have fixed capacities and age budgets derived from [15](15-PERFORMANCE-ENGINEERING.md). UI, optional deep analytics and archival writes shed first. Market state updates needed by candles are not silently coalesced. If unavoidable loss occurs, emit a durable quality-gap record, increment source epoch where continuity is broken, invalidate dependent windows and recover.

Execution updates use a distinct reserved channel with a bounded durable spool. If both memory and spool capacity are exhausted, stop new entries, mark an audit gap, fence uncertain execution activity and reconcile authenticated broker snapshots/trades before resuming. “Never silently drop an execution event” is the requirement; infinite buffering is not promised.

## Reconnect and recovery procedure

1. Mark affected source/instruments stale immediately; invalidate executable candidates and alert protection logic.
2. Reconnect with exponential jittered delay, initially 250 ms and capped at 30 seconds, under provider constraints. Token expiry requires the credential coordinator.
3. Obtain a new Upstox authorized URI when required; never reuse its single-use code. [Authorization](https://upstox.com/developer/api-documentation/get-market-data-feed-authorize-v3/)
4. Reconcile desired/observed subscriptions in legal batches, prioritizing positions and pending orders.
5. Receive initial snapshots or bounded REST snapshots; validate identity, time, session and price scale.
6. Increment source epoch and establish cumulative baselines; request missing historical bars with overlap.
7. Warm each dependency independently. Restore entry eligibility only when its readiness mask passes.

Read retries use bounded exponential backoff and quotas. A reconnect storm cannot starve order updates or account reconciliation. Circuit breakers are per endpoint/credential/scope; a data endpoint outage must not automatically disable a working execution cancel route.

## Data-source failover

Candidate alternate sources require the same canonical venue/contract, certified units, relevant session, sufficient field coverage and acceptable age. Compare price discrepancy against a configured tolerance related to tick/spread/volatility. Large discrepancy keeps both observations visible and blocks new risk rather than selecting the more favorable price.

Switch source epoch atomically for a dependency group. Reset cumulative volume/turnover and OI baselines; invalidate continuity-dependent features. Existing orders retain original broker/account and native protective orders. Account execution outage never causes automatic routing of the same intent to another broker.

## Acceptance

Verify all required packet modes with golden and malformed fixtures, mixed subscriptions under quotas, token rotation, heartbeat loss, source switching, repeated last quantities, missing OI, sequence absence and timestamp skew. Load tests must show bounded RSS and queue age at twice the measured entitled-feed peak. All loss/degradation must be visible in health and scanner readiness. Live read-only certification must compare representative NSE/BSE equity, index, future and option instruments for each broker without assuming broad enums prove account coverage.
