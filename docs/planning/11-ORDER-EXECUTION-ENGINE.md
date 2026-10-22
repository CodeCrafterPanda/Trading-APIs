# Durable order execution and reconciliation

The OMS owns the lifecycle from intent to verified exposure. One account actor and one fenced execution gateway control each broker account (Dhan, Upstox, Zerodha). Market-data routing is independently configurable; order/position ownership never migrates to another broker.

## Intent, command, order and execution

An **intent** expresses desired economic action. A **command** is one durable place/modify/cancel attempt. A **broker order** is an external order or child (Dhan slice, Upstox slice/multi item, Zerodha autoslice child or iceberg leg). An **execution** is a confirmed fill. One intent can produce several children, each with independent outcome and protection.

Local idempotency is unique(account_id, client_idempotency_key): same key and payload returns the existing intent; same key with a different payload returns IDEMPOTENCY_CONFLICT. No broker documents server-side idempotency (Zerodha `guid` is explicitly unusable for it). The broker correlation token is persisted before send:

| Broker | Correlation field | Lookup |
|---|---|---|
| Dhan | `correlationId` ≤30 chars | `GET /orders/external/{correlationId}`, `/orders/{id}`, `/trades/{orderId}` |
| Upstox | `tag` ≤40 chars (multi: per-item correlation 1–20) | `GET /v2/order/history?tag=`, `/details`, `/retrieve-all`, `/trades` |
| Zerodha | `tag` ≤20 alphanumeric | No tag endpoint: `GET /orders` (day book) filtered locally by `tag`, `/orders/{id}` history, `/orders/{id}/trades` |

Correlation lookup supports matching; it is not placement idempotency. [Dhan orders](https://dhanhq.co/docs/v2/orders/), [Upstox history](https://upstox.com/developer/api-documentation/get-order-history/), [Kite orders](https://kite.trade/docs/connect/v3/orders/)

## Submission sequence

~~~mermaid
sequenceDiagram
  participant S as Strategy or operator
  participant A as Account actor
  participant DB as PostgreSQL
  participant G as Fenced gateway
  participant B as Broker
  S->>A: Intent proposal with expected versions
  A->>A: Validate quality, session, product and risk
  A->>A: Reserve account allocation and funds
  A->>DB: Intent + command + reservation + audit transaction
  DB-->>A: Synchronous durable commit
  A->>A: Recheck age, session, ownership and price
  A->>G: Committed command and owner epoch
  G->>G: Verify ownership and rate budget
  G->>B: Submit once
  alt Broker response received
    B-->>G: ACK or definitive rejection
    G->>DB: Persist result and child mappings
  else Outcome uncertain
    G->>DB: SUBMISSION_UNKNOWN
    A->>B: Correlation/order/trade reconciliation
  end
  B-->>A: Order updates and fills
  A->>DB: Deduplicated execution and reservation transition
~~~

Risk checks run before durable commit and time-sensitive guards run again immediately before network write. The second check cannot change quantity/price; a changed payload requires a new committed command with fresh risk validation. If the intent expires while waiting, persist EXPIRED_NOT_SENT and release only its unused reservation. Decision→request-write budget including commit is p50 2 ms, p95 5 ms, p99 10 ms ([15](15-PERFORMANCE-ENGINEERING.md)).

Rate admission per gateway: Dhan D-ORDER (10/s, 250/min, 1,000/h, 7,000/day); Upstox U-ORDER (10/s regular, 500/min, 2,000/30 min, shared with modify/cancel/multi/GTT); Zerodha Z-ORDER (10/s, 400/min, 5,000/day; 429 at the per-client-ID 10 OPS threshold). Protective capacity is reserved inside each envelope; children and modifications are counted conservatively.

## Lifecycle states

~~~mermaid
stateDiagram-v2
  [*] --> PROPOSED
  PROPOSED --> REJECTED_LOCAL
  PROPOSED --> RESERVED
  RESERVED --> COMMITTED
  COMMITTED --> EXPIRED_NOT_SENT
  COMMITTED --> SUBMITTING
  SUBMITTING --> ACKNOWLEDGED
  SUBMITTING --> REJECTED_BROKER
  SUBMITTING --> SUBMISSION_UNKNOWN
  SUBMISSION_UNKNOWN --> ACKNOWLEDGED: matched order
  SUBMISSION_UNKNOWN --> REJECTED_BROKER: definitive evidence
  ACKNOWLEDGED --> OPEN
  OPEN --> PARTIALLY_FILLED
  PARTIALLY_FILLED --> FILLED
  OPEN --> FILLED
  OPEN --> CANCEL_PENDING
  PARTIALLY_FILLED --> CANCEL_PENDING
  CANCEL_PENDING --> CANCELLED
  CANCEL_PENDING --> PARTIALLY_FILLED: racing fill
  CANCEL_PENDING --> FILLED: racing completion
  OPEN --> EXPIRED
  PARTIALLY_FILLED --> EXPIRED
~~~

The diagram is a projection. Store command state, broker order state (raw status retained: Dhan TRANSIT/PENDING/PART_TRADED/TRADED; Upstox status appendix; Zerodha OPEN/COMPLETE/CANCELLED/REJECTED plus transient pending states) and cumulative fills independently, so a late fill after cancellation is representable. CANCELLED with nonzero executed quantity is valid. A terminal observation with greater confirmed cumulative fill updates exposure and raises reconciliation if inconsistent. Zerodha `exchange_order_id` null means the order never reached the exchange.

## Network uncertainty and retry rules

Before dispatch record PREPARED; immediately before network write the gateway records dispatch state in its durable journal; a crash between that record and actual send is treated as possibly sent. Only a proven NOT_SENT transport failure can reuse the same command after freshness/risk checks. Any timeout, reset or 5xx after possible transmission enters SUBMISSION_UNKNOWN with the full worst-case reservation preserved. Reconcile via the correlation table above, the order book, trades and account snapshots, merged with authenticated stream evidence. A temporarily absent lookup is not proof of rejection; a Zerodha 429 or NetworkException ("OMS unreachable") is likewise ambiguous once bytes were written.

Reconciliation retries are reads with backoff and priority budget. An unresolved command blocks overlapping new exposure for its account/instrument/product and any capital it could consume. Operator resolution requires documented broker evidence. The same economic action is never sent through another broker.

## Partial fills, modifications and cancellations

Per child track ordered_total, cumulative_filled, confirmed_remaining, average_price and executions. Apply broker trade IDs exactly once; a nonnegative cumulative increase is provisional fill evidence until trades reconcile, held in a pending adjustment bucket replaced by matched executions. Decreasing cumulative quantity or average-price inconsistency triggers authenticated lookup, never a negative fill.

Child orders: Dhan `/orders/slicing` and Upstox `slice`/multi (≤10 resulting orders, HTTP 207 per item) return explicit child IDs; Zerodha `autoslice` returns ≤10 children with their own `order_id` (success and `error` objects can coexist in one response), while `iceberg` legs are released server-side under one parent and tracked through order history. Each child is independently acknowledged, filled, rejected or unknown.

Modification references the expected broker revision and total desired quantity including fills (Dhan modify quantity = total placed quantity; Upstox and Zerodha require explicit existing values for mandatory fields). Caps: Dhan 25 modifications/order; Zerodha 25 (local cap 20, then cancel/replace); Upstox modifications consume U-ORDER. Recheck lot/tick/bands and reserve incremental risk before an increase. A modification rejection leaves the prior order potentially active; local terms update only on confirmation.

Cancels are serialized per child. After acceptance keep monitoring fills and hold residual reservation until terminal status plus reconciliation. Cancel-and-replace is sequential by default. Bulk paths (`bulk_cancel`/`bulk_exit` in [04](04-BROKER-ADAPTER-CONTRACTS.md)) are convenience inside the same rules: Upstox `DELETE /v2/order/multi/cancel` (≤50, HTTP 207) and `POST /v2/order/positions/exit` (≤50 positions, market hours only, BUY first); Dhan `DELETE /positions` Exit All; Zerodha has neither (BROKER_FEATURE_ABSENT) and uses per-order cancels and reducing orders. Every 207 item is processed individually. [Upstox cancel multi](https://upstox.com/developer/api-documentation/cancel-multi-order/), [Upstox exit all](https://upstox.com/developer/api-documentation/exit-all-positions/), [Dhan portfolio](https://dhanhq.co/docs/v2/portfolio/)

## Order-update sources and merge rule

| Broker | Primary stream | Secondary | Trust |
|---|---|---|---|
| Dhan | `wss://api-order-update.dhan.co` account-wide (includes manual orders) | HTTP postback, scoped to the originating access token, unsigned | Stream authenticated; postback untrusted |
| Upstox | Portfolio stream (`order,gtt_order,position,holding`) | App webhook, unsigned, URL-filtered only | Stream authenticated; webhook untrusted |
| Zerodha | Market-data socket JSON `order` text frames (includes Kite web/app orders) | HTTP postback, api_key-scoped, `checksum` = SHA-256(order_id + order_timestamp + api_secret) | Both authenticated after checksum verification |

Merge rule: key by broker order ID; order by cumulative filled quantity then broker update timestamp (Zerodha `exchange_update_timestamp`); duplicates across sources are discarded; an untrusted source can only raise a reconciliation lookup, never release risk; an unknown raw status becomes BROKER_STATE_UNKNOWN and blocks dependent release. Whether a Zerodha Personal-tier app receives order updates is gate G14. [Dhan order stream](https://dhanhq.co/docs/v2/order-update/), [Upstox portfolio stream](https://upstox.com/developer/api-documentation/get-portfolio-stream-feed/), [Kite postbacks](https://kite.trade/docs/connect/v3/postbacks/)

## Protection and exit ownership

PositionProtection links confirmed exposure to native stops/targets and local trailing policy. A partial entry fill immediately creates/resizes protection for confirmed units within broker semantics. If native protection cannot be established, block further entry fills where cancel is possible, alert, and run the certified fallback reduction.

Native protection by broker: Dhan Super Order (entry/target/stop/trailing jump) and Forever OCO (documented CNC/MTF); Upstox GTT multi-rule ENTRY/TARGET/STOPLOSS with beta `trailing_gap`; Zerodha GTT `two-leg` OCO with LIMIT orders only and no trailing, so Zerodha trailing is local cancel/replace of the GTT or working stop under the modification cap. GTT is a broker-side trigger that places an order, not an exchange-resident stop. OCO is assumed only where the exact native product is certified (gate G08); otherwise sibling cancel/resize is local with fill-race modelling. [Dhan Super](https://dhanhq.co/docs/v2/super-order/), [Upstox GTT](https://upstox.com/developer/api-documentation/place-gtt-order/), [Kite GTT](https://kite.trade/docs/connect/v3/gtt/)

One ExitController per account/instrument/product owns all stop, target, time, manual and emergency exits. Desired reduction = confirmed exposure − outstanding reducing quantity, conservative for unknown fills. A reducing order never intentionally reverses a position; no broker offers reduce-only, so local serialization and reconciliation enforce it, and external orders trigger reconciliation.

## Account ownership and crash recovery

AccountLease stores owner_epoch, gateway_id, heartbeat and lease expiry. Database locks coordinate actors; egress fencing or verified old-gateway shutdown is required before promotion because a stale process may hold a valid token (Zerodha tokens stay valid until 06:00 or explicit `DELETE /session/token`, which also invalidates the successor's session if shared — use distinct logins per gateway epoch). Gateways fail closed on lost ownership proof; no dual-active failover.

Recovery: disable new entries; acquire a new owner only after fencing; load durable intents/commands/reservations; start authenticated streams into a bounded spool; fetch orders/trades/positions/holdings/funds; merge buffered events; reconcile unresolved commands and external activity; recompute exposure/reservations; verify protection; warm data; enable the route only after readiness evidence.

Persist expires_at_utc for intents and clock_domain/host_boot_id with monotonic stamps; reconstruct runtime deadlines only under clock-health policy; never revive an expired intent.

## Reconciliation schedule

Streaming is primary. Snapshot reconciliation runs at startup, reconnect, token rotation, uncertain command, unexpected update, external activity and a low-frequency audit (initially every 60 s while active) within each broker's read budget: Dhan D-NONTRADE 20/s, Upstox U-STANDARD, Zerodha Z-OTHER 10/s. Funds/margin refresh is demand-driven and age-limited.

ReconciliationDiff categorizes missing local order, missing broker order, unmatched fill, position mismatch, funds drift, unknown product and stale protection. External orders are imported with EXTERNAL origin and consume risk. Never fabricate executions; persist explicit unexplained adjustments and require resolution.

## Failure behavior and acceptance

Ledger unavailability blocks new mutations; broker-native protection remains; updates spool within bounded durable capacity; spool exhaustion marks an audit gap, halts entries and forces full reconciliation. A broker outage leaves unresolved exposure visible; liquidation cannot be guaranteed when the broker/exchange is unreachable.

Required fault tests on every adapter: crash before/after commit/send/ACK, duplicate local request, delayed ACK, partial child rejection, cancel-fill race, modify rejection, out-of-order updates, external manual orders, token rotation and standby fencing. Broker-specific: Dhan sliced-child correlation and postback-vs-stream duplicate; Upstox 207 partial multi/cancel-multi/exit-all and unsigned webhook spoof; Zerodha TokenException from a login elsewhere mid-session, 429 at 10/s and 400/min, autoslice response with coexisting success and error children, postback checksum mismatch, tag collision in local filtering. Every test asserts durable state/reservations, no blind resubmit and no cross-account routing. Authorized live commissioning is a separate gated phase.
