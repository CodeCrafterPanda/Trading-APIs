# Trading dashboard and real-time interactions

The interface makes market opportunity, data quality and account risk visible together. React/TypeScript uses selector-based stores, virtualized lists, canvas heatmaps and TradingView Lightweight Charts behind a chart adapter; every panel updates through the push protocol below with no manual refresh. The server is the authority for risk/order state; the browser cannot submit directly to brokers or store broker credentials.

## Navigation and screen contracts

| Screen | Required content | Primary interaction and failure state |
|---|---|---|
| Main dashboard | Market regime, index/sector leaders, opportunities, account exposure, feed/route health | Drill into evidence; stale/incomplete badges remain prominent |
| Sector heatmap | Official vs calculated basis, return, breadth, RVOL, coverage | Select sector; accessible table and fixed color scale |
| Sector-to-stock | Constituents, ranks, component scores, membership date | Filter/sort; freeze row order while interacting |
| Live stock scanner | Detector, direction, evidence, age, quality, liquidity | Configure scanner filters; empty state explains no qualifying setup |
| Stock chart | Candles, volume, source/adjustment, gaps, levels, scanner markers | Interval/source selection; corrected bars marked |
| Futures/options scanner | Contract, expiry, lot, OI/IV, basis, liquidity, eligibility | Select contract; expired/unmapped rows disabled |
| Option chain | Strike, CE/PE price/book/OI/IV/Greeks with source badge (PROVIDER or LOCAL_MODEL), field age, underlying | Moneyness/delta filters; missing values shown as unavailable |
| Opportunities | Ranked score components, entry/stop/target hypotheses, cost/risk, expiry | Inspect or prepare order; expired setup cannot be reused silently |
| Order ticket | Explicit broker/account/product, units and lots, prices, protection, estimated margin | Preview then submit; server revalidation can reject stale preview |
| Open orders | Child orders, ACK/state/fills, remaining quantity, pending mutations | Modify/cancel; unknown state visibly distinct from rejected |
| Positions | Account/product units, P&L, mark age, protection and pending exits | Controlled close; exposure unresolved state prominent |
| Risk/capital | Account budgets, funds age, reservations, drawdown, Greeks, global allocations | Halt/reduce controls with exact scope |
| Broker settings (one tab per broker: Dhan, Upstox, Zerodha) | Token class/expiry and next-expiry countdown, segments, static IP status, entitlement/capability evidence, postback/webhook verification state, data-source role vs execution role | Broker-managed authorization (Dhan consent/TOTP, Upstox OAuth, Zerodha request-token login); never display secret values |
| Scanner/strategy configuration | Parameter validation, dependency cost, allowed regimes, risk allocation | Version preview/activate; warming state after change |
| Health/performance | Source coverage, queue ages, decoder errors, latency distributions, reconciliation | Drill into incident/runbook; no false green aggregate |

## Information hierarchy and accessibility

Persistent top strip shows active data source(s) per instrument group, selected execution account (broker + account), market phase from the session engine, and global trading mode. A source label and execution account label must remain distinct; with three brokers available, the ticket always names the broker that will receive the order. Red/green never carries meaning alone: signed values, arrows, text and patterns support color-vision differences. Keyboard navigation, visible focus, semantic labels and readable table alternatives are acceptance requirements.

Desktop uses main analytical area and optional account/risk side panel. Tablet collapses secondary panels. Mobile prioritizes positions, risk and order status; dense chain tables scroll horizontally with pinned strike/side columns. Large charts/heatmaps must not prevent access to emergency controls. Responsive behavior changes presentation, not risk checks.

## Real-time synchronization protocol

One authenticated WebSocket per client carries every stream; the control API handles mutations and snapshots. The publisher (worker 14 in [02](02-SYSTEM-ARCHITECTURE.md)) reads immutable state snapshots and per-view dirty sets; it never touches shard memory and a slow client can only shed its own queue. Latency targets are the L7/L8/L9 paths in [15](15-PERFORMANCE-ENGINEERING.md).

**Subscription management (client).** Mounted components declare the views they need (stream, filters, sort, viewport row range, columns, cadence); a subscription manager ref-counts identical declarations, diffs the desired set against the active set, and sends `subscribe`/`unsubscribe`/`modify` messages. Viewport changes are debounced 50 ms; scrolling never tears down the stream. Unmount removes the reference; the last reference unsubscribes. View limits (≤ 32 subscriptions, ≤ 2,000 visible rows) are presentation limits, not analysis universe caps.

**Lanes.** Every stream belongs to a lane that fixes its delivery policy:

| Lane | Streams | Policy |
|---|---|---|
| P0 account | orders, executions, positions, P&L, reservations, risk actions, emergency progress | Immediate; never coalesced; durable read model with cursor replay on reconnect |
| P1 event | candle.opened/closed/corrected, scanner candidate created/retracted, opportunity state changes, alerts, source failover, session phase change | Immediate; batched per 16 ms; never dropped while connected |
| P2 display | quotes, active-candle updates, heatmap tiles, rankings, chain values, Greeks | Coalesced at view cadence (5–10 Hz); latest value wins; droppable under overload |
| P3 background | health rollups, queue ages, latency histograms | 1 Hz |

**Delta encoding.** Each message is one subscription, one lane, with a per-subscription sequence. Upserts carry only changed fields (field-level deltas) plus the row revision and state version; removals carry a reason. Numbers are decimal strings for prices/money and integers for quantities; booleans and enums are short codes. Wide tables (heatmap, scanner) may use column arrays when a message exceeds 200 rows. JSON is the initial encoding at this boundary (ADR-13 in [19](19-DECISIONS-AND-RISKS.md)); a binary encoding is adopted only when measurement shows encode or parse time dominating L7/L9.

~~~json
{
  "v": 1, "sub": "hm-nifty50", "lane": "P2", "seq": 182, "snap": "s-91",
  "sv": 1734020, "st": "2026-10-22T04:00:00.100Z",
  "up": [{"id": "RELIANCE", "rev": 9, "f": {"ltp": "2951.40", "chg": "0.0123", "rvol": "1.84", "age": 120}}],
  "rm": [{"id": "EXPIRED-1", "r": "EXPIRED"}]
}
~~~

**Sequence tracking and recovery.** `seq` is contiguous per subscription within a server epoch; `snap` names the snapshot the deltas apply to. A missing sequence, unknown `snap` or epoch change triggers a resync of that subscription only: the client sends `resync{sub, last_seq}`; the server replays from its per-subscription ring (P1/P2, ≤ 2 s) or P0 cursor (durable), otherwise sends a fresh snapshot. A delta is never applied to a different snapshot. Heartbeats every 1 s carry the server time; 3 s without any frame marks all views STALE and starts reconnection.

**Reconnection.** Exponential backoff 250 ms → 10 s with jitter; on connect the client re-authenticates, resends its whole desired set with `last_seq` per subscription, and renders STALE badges until each subscription is resynced. Pending mutations are retrieved by client idempotency key, never re-sent blindly. Background tabs reduce P2 cadence to 1 Hz; server-side protection continues independently.

**Throttling and backpressure.** Per client: byte and message budgets per second, adaptive P2 cadence, bounded send buffer. Overload order: lower P2 cadence → drop P2 for non-visible rows → drop P2 entirely with a STALE banner → disconnect with `resync_reason=SLOW_CONSUMER`. P0 and P1 are preserved until disconnect. Browser WebSocket has no application backpressure, so these limits are server-enforced. [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

**Client rendering.** One external store per stream (`useSyncExternalStore`); rows are immutable objects replaced only when a field changes; selectors are per row/cell so unrelated rows do not rerender. Incoming messages are decoded and applied inside one `requestAnimationFrame` batch per frame; if decode + apply exceeds its stage budget in [15](15-PERFORMANCE-ENGINEERING.md), decoding moves to a Web Worker that posts applied patches. Tables virtualize rows; the heatmap repaints only dirty tiles on a canvas; charts receive `update()` calls through the chart adapter. Sort order is frozen while the pointer is over a table; a "rank changed" marker replaces row jumps. Everything that updates automatically (heatmaps, rankings, scanners, watchlists, charts, orders, positions, P&L, health) does so through this protocol; there is no manual refresh and no HTTP polling.

**Measurement.** The client stamps receive/apply/commit with `performance.mark`, computes tick→screen from `st` plus local render with the clock-offset estimate, and reports L8/L9 samples and superseded-update counts to `/api/v1/health` at 1 Hz.

## Charting with TradingView Lightweight Charts

Library: `lightweight-charts` 5.x (API verified against 5.2.1 on 22 October 2026; pinned exact release in the lockfile). Apache-2.0 with mandatory attribution: show the NOTICE text and a link to https://www.tradingview.com on a public page of the dashboard. All access goes through a platform `ChartAdapter` so the library can be replaced behind one interface (ADR-14 in [19](19-DECISIONS-AND-RISKS.md)). [Docs](https://tradingview.github.io/lightweight-charts/docs)

| Concern | Implementation | Rule |
|---|---|---|
| Chart instance | `createChart(container, {autoSize: true, localization: {priceFormatter}, crosshair: {mode: CrosshairMode.MagnetOHLC}, timeScale: {timeVisible: true, secondsVisible: false}})` once per mounted chart; `chart.remove()` on unmount | Never recreated on data, timeframe or symbol change |
| Price series | `chart.addSeries(CandlestickSeries, {priceFormat: {type: 'price', precision, minMove: tick}})`; `LineSeries`/`AreaSeries` for LTP-only or index views | `minMove` from the instrument tick size; precision from tick decimals |
| Volume | `chart.addSeries(HistogramSeries, {priceFormat: {type: 'volume'}}, 1)` in a second pane (`addPane`, stretch factor ≈ 0.25); bar color from candle direction; `vol` only when the bar's volume quality is valid, otherwise whitespace | Volume pane hidden for indices (NOT_APPLICABLE) |
| Indicators | EMA/VWAP/ATR bands as `LineSeries` in pane 0 or an indicator pane; values computed server-side and streamed as features, not recomputed in the browser | Server is the single source of indicator values |
| Historical load | Initial page via `GET /api/v1/candles` → `series.setData(bars)` once per series key; `timeScale().subscribeVisibleLogicalRangeChange` + `series.barsInLogicalRange(range).barsBefore < 50` → fetch the older page → prepend and `setData` (the only permitted `setData` after init; batched, ≤ 1/s) | Resident bars ≤ 50,000; `enableConflation: true` on time scale for long histories |
| Live updates | candle.updated/closed → `series.update(bar)` (replaces the last bar when `time` matches, appends when newer); histogram `update` in the same frame; candle.corrected on an older bar → `series.update(bar, true)` (`historicalUpdate`) | Never `setData` on a tick; updates batched per animation frame with latest-wins per series |
| Timeframe / symbol switch | Resolve the new series key, `setData` on the existing series objects, re-subscribe the candle stream; keep chart, panes, price lines and primitives | Interval, origin and adjustment basis shown in the chart header; a source switch is never silent |
| Gaps and quality | INCOMPLETE/PARTIAL bars rendered with a distinct border color via per-bar `color/borderColor/wickColor`; DISPLAY_SYNTHETIC bars as `WhitespaceData` (not drawn); corrected bars flagged in the companion table | No fabricated bar reaches the series |
| Time zone | Library has no time-zone support and formats UTC; bar `time` is shifted by the IST offset (+19,800 s, no DST) once at the adapter boundary and shifted back for lookups | One conversion site; never per render |
| Entry / exit / stop / target | `series.createPriceLine({price, title, color, lineStyle, axisLabelVisible: true})` per level; updated via `applyOptions`, removed via `removePriceLine` when the opportunity or order changes | Lines are bound to the opportunity/order revision |
| Support / resistance | Price lines for active SC19/SC20 bands (band edges as two lines or one primitive rectangle); evicted with the level | ≤ 16 visible lines; older levels in the table |
| Order blocks, supply/demand zones, consolidation boxes | Series primitive implementing `ISeriesPrimitive` (rectangle from `time_from`→`time_to`, `price_low`→`price_high`, using `priceToCoordinate` and `timeScale().timeToCoordinate`), attached with `series.attachPrimitive` and detached on invalidation | Primitives render in the chart's canvas layers; no DOM overlays |
| Signals, fills, scanner events | `createSeriesMarkers(series, markers)` plugin; `setMarkers` with the full current marker array for the visible setups; fills use `position: 'atPriceMiddle'` with `price` | Marker count bounded (≤ 200) to keep hover cost flat |
| Crosshair / scales | `subscribeCrosshairMove` drives the OHLC + geometry legend from `param.seriesData`; `priceScale().applyOptions({autoScale: true})`; right price scale for price, overlay scale for indicators where needed | Legend reads values from the store, not the DOM |
| Multiple charts | Shared crosshair via `setCrosshairPosition`/`clearCrosshairPosition`; visible-range sync via `subscribeVisibleLogicalRangeChange` | Only while both charts share the interval |
| Accessibility | Companion OHLC/geometry table (virtualized) for every chart; keyboard interval switching; color-independent direction glyphs | Chart is never the only representation |

Chart acceptance: a live 1-minute series at 10 updates/s per visible chart with four charts open keeps L9 within budget; history pagination never calls `setData` more than once per fetched page; correction of a non-last bar is visible without a full reload; switching intervals keeps price lines and zones; attribution link present; adapter tests run against the pinned release and fail on API drift (`addSeries`, `update(historicalUpdate)`, `createSeriesMarkers`, `attachPrimitive`, panes).

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

At the declared maximum visible rows and client count, meet the L7, L8, L8c and L9 targets in [15](15-PERFORMANCE-ENGINEERING.md) on the reference hardware. Test slow networks, disconnected browser, missing deltas, stale snapshots, epoch change, slow-consumer disconnect and resync, background-tab cadence, large chains, high-contrast mode, keyboard-only operation and mobile overflow. Verify that one stock update rerenders only its row/tile/chart, that no HTTP polling exists, that broker tokens never enter browser responses/storage and that every action displays the selected account. Empty/partial/stale/permission-denied states are designed screens, not blank panels.
