# Trading dashboard and real-time interactions

The interface makes market opportunity, data quality and account risk visible together. React/TypeScript uses selector-based stores, virtualized lists, canvas heatmaps and TradingView Lightweight Charts behind a chart adapter. The server is the authority for risk/order state; the browser cannot submit directly to brokers or store broker credentials.

## Navigation and screen contracts

| Screen | Required content | Primary interaction and failure state |
|---|---|---|
| Main dashboard | Market regime, index/sector leaders, opportunities, account exposure, feed/route health | Drill into evidence; stale/incomplete badges remain prominent |
| Sector heatmap | Official vs calculated basis, return, breadth, RVOL, coverage | Select sector; accessible table and fixed color scale |
| Sector-to-stock | Constituents, ranks, component scores, membership date | Filter/sort; freeze row order while interacting |
| Live stock scanner | Detector, direction, evidence, age, quality, liquidity | Configure scanner filters; empty state explains no qualifying setup |
| Stock chart | Candles, volume, source/adjustment, gaps, levels, scanner markers | Interval/source selection; corrected bars marked |
| Futures/options scanner | Contract, expiry, lot, OI/IV, basis, liquidity, eligibility | Select contract; expired/unmapped rows disabled |
| Option chain | Strike, CE/PE price/book/OI/IV/Greeks, field age, underlying | Moneyness/delta filters; missing values shown as unavailable |
| Opportunities | Ranked score components, entry/stop/target hypotheses, cost/risk, expiry | Inspect or prepare order; expired setup cannot be reused silently |
| Order ticket | Explicit broker/account/product, units and lots, prices, protection, estimated margin | Preview then submit; server revalidation can reject stale preview |
| Open orders | Child orders, ACK/state/fills, remaining quantity, pending mutations | Modify/cancel; unknown state visibly distinct from rejected |
| Positions | Account/product units, P&L, mark age, protection and pending exits | Controlled close; exposure unresolved state prominent |
| Risk/capital | Account budgets, funds age, reservations, drawdown, Greeks, global allocations | Halt/reduce controls with exact scope |
| Broker settings | Token class/expiry, segments, static IP status, entitlement/capability evidence | Broker-managed authorization; never display secret values |
| Scanner/strategy configuration | Parameter validation, dependency cost, allowed regimes, risk allocation | Version preview/activate; warming state after change |
| Health/performance | Source coverage, queue ages, decoder errors, latency distributions, reconciliation | Drill into incident/runbook; no false green aggregate |

## Information hierarchy and accessibility

Persistent top strip shows active data source(s), selected execution account, market phase and global trading mode. A source label and execution account label must remain distinct. Red/green never carries meaning alone: signed values, arrows, text and patterns support color-vision differences. Keyboard navigation, visible focus, semantic labels and readable table alternatives are acceptance requirements.

Desktop uses main analytical area and optional account/risk side panel. Tablet collapses secondary panels. Mobile prioritizes positions, risk and order status; dense chain tables scroll horizontally with pinned strike/side columns. Large charts/heatmaps must not prevent access to emergency controls. Responsive behavior changes presentation, not risk checks.

## Transport and state protocol

Client authenticates to the platform, requests a snapshot and subscribes to named views. View subscription includes filters, sort, viewport range, required columns and desired cadence. Server validates permissions and caps client resource use. View limits are presentation limits, not analysis universe caps.

~~~json
{
  "schema_version": 1,
  "stream": "opportunities",
  "subscription_id": "view-example",
  "snapshot_id": "snapshot-example",
  "sequence": 182,
  "server_time": "2026-10-22T04:00:00.100Z",
  "upserts": [{"id": "opportunity-example", "revision": 9, "score": 82.4}],
  "removals": [{"id": "expired-example", "reason": "EXPIRED"}]
}
~~~

Sequence is per subscription/server epoch. Missing sequence, changed epoch or expired snapshot triggers resync. Never apply a delta to the wrong snapshot. Latest market display values can coalesce; order/risk notifications have priority and durable read-model recovery. A slow browser gets reduced cadence, then disconnected with a resync reason before its buffer grows unbounded. [Browser WebSocket constraints](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket).

Initial cadence: quotes/heatmaps 5–10 Hz, candles on meaningful revision, rankings 5 Hz, order/risk events immediately scheduled. Client batches store updates per animation frame and renders only changed selectors. Background tabs reduce market cadence; server-side protection continues independently.

## Control API

All endpoints are platform endpoints, separate from broker APIs:

| Endpoint | Contract |
|---|---|
| GET /api/v1/market/snapshot | View/filter + snapshot/version/coverage |
| GET /api/v1/instruments/{id} | Canonical metadata, mappings and restrictions |
| GET /api/v1/candles | Series/range + bars/gaps/revisions/provenance |
| GET /api/v1/options/chains | Underlying/expiry + snapshot/ages |
| GET /api/v1/opportunities | Filter/cursor + score evidence and expiry |
| POST /api/v1/order-previews | Proposed intent → normalized terms, risk reasons, estimated margin, preview expiry |
| POST /api/v1/order-intents | Idempotency key + preview/input versions → durable intent or rejection |
| POST /api/v1/orders/{id}/modify | Expected revision + desired terms → command ID |
| POST /api/v1/orders/{id}/cancel | Expected revision/idempotency key → command ID |
| POST /api/v1/risk/actions | Explicit scope/mode/reason + expected version → durable action |
| POST /api/v1/configurations | Validated draft/version; activation separate |
| GET /api/v1/health | Scoped connectivity, quality, account and performance states |

Mutation errors use stable reason codes, safe messages and remediation; never show raw tokens or unfiltered broker responses. Submit button becomes pending by client idempotency key; reloading retrieves the same intent. “Submitted” means platform durable intent; “Broker acknowledged,” “Partially filled” and “Filled” are separate labels.

## Order and emergency interactions

Ticket defaults to no hidden broker switch. Show quantity units and equivalent lots, product, limit/trigger, expiry risk, costs, protection basis and account funds age. A manual action can use a non-opportunity instrument, but must pass identical server gates. A changed quote does not silently change user-entered limit price.

Emergency action names state scope: halt new entries, reduce exposure, flatten account, or broker disable after flat verification. Progress shows cancelled entries, remaining positions, unresolved commands and actual fills. An HTTP success toast cannot say “All positions closed” before reconciliation.

## Acceptance and performance

At declared maximum visible rows, maintain responsive interaction and target p95 frame work under 16 ms on reference hardware; server-to-visible update target p99 500 ms. Test slow networks, disconnected browser, missing deltas, stale snapshots, large chains, high-contrast mode, keyboard-only operation and mobile overflow. Verify broker tokens never enter browser responses/storage and every action displays the selected account. Empty/partial/stale/permission-denied states are designed screens, not blank panels.
