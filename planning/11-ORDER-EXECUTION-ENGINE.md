# Durable order execution and reconciliation

The OMS owns the complete lifecycle from intent to verified exposure. One account actor and one fenced execution gateway control each broker account. Market-data routing is independently configurable; order/position ownership never migrates automatically to the other broker.

## Intent, command, order and execution

An **intent** expresses desired economic action. A **command** is one durable place/modify/cancel attempt. A **broker order** is an external order or sliced child. An **execution** is a confirmed fill. These entities are not interchangeable. One intent can produce several children; each child has independent outcome and protection.

Local idempotency is unique(account_id, client_idempotency_key). Repeating the same key and payload returns the existing intent; repeating it with different payload returns IDEMPOTENCY_CONFLICT. This prevents duplicate platform creation but does not make broker placement idempotent. Broker correlation/tag mapping is persisted before send.

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

Risk checks happen before durable commit and time-sensitive guards again immediately before network write. The second check cannot silently change quantity/price; a changed economic payload requires a new committed command or revision with fresh risk validation. If the intent expires while waiting, persist EXPIRED_NOT_SENT and release only its unused reservation.

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

The diagram is a projection. Store command state, broker order state and cumulative fills independently so a late fill after cancellation can be represented without pretending history was impossible. CANCELLED with nonzero executed quantity is valid; the residual is cancelled, not the prior execution. A terminal state observation with greater confirmed cumulative fill updates exposure and raises reconciliation if inconsistent.

## Network uncertainty and retry rules

Before dispatch, record PREPARED. Immediately before network write, gateway records dispatch state in its durable command journal/ledger transaction; if a crash creates ambiguity between this record and actual send, recovery treats the command as possibly sent. Exactly-once execution over an external API is not promised.

Only a proven NOT_SENT transport failure can reuse the same command after freshness/risk checks. Any timeout, connection reset or 5xx after possible transmission enters SUBMISSION_UNKNOWN. Preserve full worst-case reservation. Query Dhan correlation lookup or Upstox tag/history, order book, trades and account snapshots; merge authenticated stream evidence. A temporarily absent lookup is not proof of rejection. [Dhan orders](https://dhanhq.co/docs/v2/orders/), [Upstox history](https://upstox.com/developer/api-documentation/get-order-history/).

Reconciliation retries are reads with backoff and priority budget. An unresolved command blocks overlapping new exposure for its account/instrument/product and any capital it could consume. Operator resolution requires documented broker evidence; it is not a “retry anyway” button. Do not send the same economic action through another broker.

## Partial fills, modifications and cancellations

For each child track ordered_total, cumulative_filled, confirmed_remaining, average_price and executions. Apply new broker trade IDs exactly once. If an update reports cumulative_filled C_new and local C_old, a nonnegative increase is provisional fill evidence until detailed trades reconcile; do not double count later trade-book records. Use a pending cumulative adjustment bucket tied to the order and replace it with matched executions.

Decreasing cumulative quantity or average-price inconsistency triggers authenticated lookup; it never creates a negative fill automatically. Corrections require explicit adjustment records and retained evidence. Fee/P&L calculations use actual fills and a separately reconciled charge ledger.

Modification references expected broker revision and total desired quantity, including fills. Desired total cannot be below confirmed executed units. Recheck lot/tick/bands and reserve incremental risk before increase. A modification rejection leaves the prior order potentially active. Do not overwrite local order terms until confirmation.

Cancel requests are serialized per child. After acceptance, continue monitoring fills and hold residual reservation until terminal status plus reconciliation establishes remaining quantity. Cancel-and-replace is default sequential: confirm cancellation/residual, then place a new intent for still-required quantity. Emergency urgency does not justify a second full-size order while cancellation is unknown.

## Protection and exit ownership

PositionProtection links confirmed exposure to native stops/targets and local trailing policy. Partial entry fill immediately creates/resizes protection for confirmed units within allowed broker semantics. If native protection cannot be established, block further entry fills where cancel is possible, raise an urgent alert and execute the certified fallback reduction policy.

One ExitController per account/instrument/product owns all stop, target, time, manual and emergency exit requests. It computes desired reduction = current confirmed exposure minus already outstanding reducing quantity, accounting conservatively for unknown fills. A reducing order cannot intentionally reverse position. Because brokers may not offer reduce-only, local serialization and reconciliation enforce this as far as observable state permits; external/manual orders can still race and must trigger reconciliation.

Targets and stops are OCO only when the broker's exact native product is certified. Two unrelated orders are not atomic OCO. When one fills, cancel/resize the sibling; model fill races and residual reversal risk. Local trailing modifies only on favorable movement, minimum tick improvement and rate-budget availability, with hard protection maintained.

## Account ownership and crash recovery

AccountLease stores owner_epoch, gateway_id, heartbeat and lease expiry. Database locks coordinate active actors; egress fencing or verified old-gateway shutdown is required before promotion because a stale process may retain a valid token. Gateways fail closed on lost ownership proof. No automatic dual-active failover.

Recovery procedure: disable new entries; acquire a new owner only after fencing; load durable intents/commands/reservations; start authenticated update stream into bounded spool; fetch orders/trades/positions/holdings/funds; merge buffered events; reconcile unresolved commands and externally initiated activity; recompute exposure/reservations; verify protection; warm data; enable route only after readiness evidence.

Persist expires_at_utc for intents and clock_domain/host_boot_id with diagnostic monotonic stamps. Monotonic values cannot cross restart/host boundaries. Reconstruct runtime deadlines from remaining UTC lifetime only when clock uncertainty is within policy; otherwise expire/block. Never revive an expired intent after recovery.

## Reconciliation schedule

Streaming is primary. Snapshot reconciliation runs at startup, reconnect, token rotation, uncertain command, unexpected update, manual/external activity and a configurable low-frequency health audit (initially every 60 seconds while active, adjusted within quotas). Funds/margin refresh is demand-driven and age-limited. This bounded account-state audit is not REST replacement of market streaming.

ReconciliationDiff categorizes missing local order, missing broker order, unmatched fill, position mismatch, funds drift, unknown product and stale protection. External orders are imported with EXTERNAL origin and consume risk. Never fabricate executions to force positions to match; persist explicit unexplained adjustments and require resolution.

## Failure behavior and acceptance

Ledger unavailability blocks new mutations. Existing broker-native protection remains; account updates spool within bounded durable capacity. Spool exhaustion marks an audit gap, halts entries and forces full reconciliation. Broker outage leaves unresolved exposure visible; the application cannot guarantee liquidation when the broker/exchange is unreachable.

Required fault tests cover crash before/after commit/send/ACK, duplicate local request, delayed ACK, partial sliced rejection, cancel-fill race, modify rejection, out-of-order updates, external manual orders, token rotation and standby fencing. In every case the test asserts actual durable state/reservations, no blind resubmit and no cross-account routing. Performance acceptance includes durable commit in the measured decision-to-send path. Authorized live commissioning is a separate gated phase, not performed by this planning task.
