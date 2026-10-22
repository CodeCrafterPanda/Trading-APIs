# Upstox live trading adapter research

Research date: **2026-10-21, Asia/Kolkata**. All links below were accessed on that date. This is documentation research, not account certification. No account credentials, live requests, or orders were used. Fact identifiers U01–U30 are stable references for the specification writer. Endpoint examples describe contracts, not application implementation.

## Which API versions, authentication methods, and account gates apply?

### Takeaway

Use an endpoint-version manifest: Upstox currently mixes V3 market data, candles, order submission, funds and GTT with V2 account/portfolio and order lookup APIs. Trading credentials and analytics credentials must remain distinct.

### Cited Findings

- **U01 — OAuth:** The normal flow redirects to Upstox, obtains a single-use authorization code, and exchanges it server-side at `POST https://api.upstox.com/v2/login/authorization/token`. The login dialog is `https://api.upstox.com/v2/login/authorization/dialog`; registered redirect URI must match. Semi-automated access-token requests still require user approval and deliver the result to a notifier URL. [Authentication](https://upstox.com/developer/api-documentation/authentication/)
- **U02 — Ordinary token lifetime and documented TOTP alternative:** The ordinary access token expires at the next 03:30 AM boundary, including tokens generated shortly before that boundary. The current detailed page also documents `POST /v2/totp-login/authorization/token`, with `x-api-key` header and six-digit `pin` and `totp` body fields, for individual apps with TOTP enabled. This issuance revokes an existing app access token. Its page conflicts with the older general authentication page's assertion that direct programmatic login is unavailable. Use the detailed endpoint as documented capability, subject to account validation. [Get Token](https://upstox.com/developer/api-documentation/get-token/)
- **U03 — Analytics token:** Current docs describe a free, one-year, read-only token generated in Developer Apps, one per account. It cannot place, modify or otherwise trade. The page restricts it to GET APIs. It lists market-data categories without static-IP requirements and account/order/portfolio read categories with static-IP requirements. Therefore a POST margin-calculation endpoint is not automatically enabled by the broad “Margins” category label. [Analytics Token](https://upstox.com/developer/api-documentation/analytics-token/)
- **U04 — Static IP and current algo enforcement:** Upstox's official March 31, 2026 announcement says the changes are deployed, effective April 1, 2026: registered static IP for API trading, registration for strategies exceeding 10 OPS, and unprotected market orders prohibited. This is a broker operational statement, not a substitute for the exchange circular. [Deployed regulatory changes](https://community.upstox.com/t/important-update-regulatory-changes-for-api-and-algo-trading-are-now-live/14874)
- **U05 — App/IP configuration:** The March 27 announcement introduced one active API app per user. Current static-IP endpoint docs describe **user-level**, rather than client-level, primary and optional secondary addresses: `GET https://api.upstox.com/v2/user/ip`; both IPv4 and IPv6 are represented. Account/app type can restrict this API. [Single-app announcement](https://community.upstox.com/t/important-new-sebi-exchange-mandates-for-api-trading-effective-1st-april-2026/14822), [Static-IP API](https://upstox.com/developer/api-documentation/get-app-static-ips/)
- **U06 — Individual vs business use:** Direct HTTP execution integration is described as intended for individual API users; businesses are directed to Uplink Business. Delivery sales without DDPI/POA require CDSL TPIN/OTP authorization. [Orders overview](https://upstox.com/developer/api-documentation/orders/)
- Official SDK installation pages list Python, Node.js, Java, PHP and .NET. They do not establish an official Rust or Go SDK. [SDK installation](https://upstox.com/developer/api-documentation/installing-sdk/)

### Inferences

- Default deployment assumption: one owner operating their own accounts, not a customer-facing brokerage or advisory business. Make this explicit in the blueprint; commercial/multi-client rollout requires a separate broker integration clearance.
- Use OAuth or broker-approved approval-based renewal by default. Support documented TOTP exchange behind a capability flag, but **do not store unattended TOTP seeds or PINs by default**. A secure operator-supplied TOTP/PIN flow may be specified. Token rotation needs a single elected owner; pause new submissions, publish token generation, reconnect streams, and reconcile orders after replacement. Never let competing workers regenerate tokens.
- Interpret ordinary expiry in India market time for scheduling, validate the account's returned expiry metadata before launch, and distinguish it from analytics expiry. Do not invent a refresh-token grant.
- Use the analytics token only in a separately scoped read connector after permitted endpoint checks. The execution adapter must reject it locally.
- Register both primary and backup egress addresses before any execution failover. Do not try to evade per-user restrictions by creating extra apps or distributing accounts.

### Gaps

- Actual active segments, app category, registered algo status/names, authorization grants, token issuance eligibility, approved egress IPs and account entitlements need authenticated verification.
- Ordinary token documentation spells out 03:30 AM but does not label the timezone in the inspected paragraph; Asia/Kolkata is the operational interpretation to confirm against returned metadata/broker onboarding.
- The general auth page and detailed token endpoint are inconsistent; record this explicitly instead of asserting that TOTP login is unsupported or universally available.

## What live streaming, instrument metadata, and data entitlements are documented?

### Takeaway

Market V3 is Protobuf over WebSocket with explicit user limits and mode-dependent capacity. Broad price coverage is possible within those limits; comprehensive volume/depth scanners need richer subscriptions and cannot infer rich fields from LTPC.

### Cited Findings

- **U07 — Feed protocol/capacity:** Market V3 uses supplied Protobuf schemas; outgoing subscription messages are binary. `sub`, `unsub`, and `change_mode` are supported. Normal: **2 connections per user**. Individual/mixed instrument-key limits: LTPC **5000/2000**, option Greeks **3000/2000**, full **2000/1500**. Plus: **5 connections**, full_d30 **50 individual / 1500 combined as printed**. The text describes mixed-mode limits per category for the same user. Do not multiply capacity by connection count. Full carries five depth levels; full_d30 carries thirty. LTPC lacks cumulative-volume/depth fields; full includes ATP, VTT, OI, IV, Greeks, depth and candles. Initial messages provide market status and snapshot, followed by live updates. `ltt` and `currentTs` are distinct. Current schema includes auction-specific fields. [Market Data Feed V3](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/)
- **U08 — Socket authorization:** `GET /v3/feed/market-data-feed/authorize` returns `data.authorized_redirect_uri`. Its embedded authorization code is single-use. Automatic redirect handling is an alternative. Idle connections receive WebSocket ping frames; normal clients respond with pong. [V3 authorized URL](https://upstox.com/developer/api-documentation/get-market-data-feed-authorize-v3/)
- **U09 — Instrument master:** BOD JSON includes equity, futures, options and indices, with NSE and BSE downloads. Prefer `instrument_key`; exchange tokens may be reused after expiry. Fields include segment, exchange, ISIN where relevant, trading symbol, lot size, tick size, freeze quantity, expiry, underlying key/type, strike, CE/PE/FUT classification and weekly flag. Separate suspended and MIS lists exist; current equity metadata also includes `cas_eligible`. [Instruments](https://upstox.com/developer/api-documentation/instruments/)
- **U10 — Portfolio stream:** `wss://api.upstox.com/v2/feed/portfolio-stream-feed` uses authenticated redirect flow. `update_types` accepts `order,gtt_order,position,holding`; default is order updates. Order payloads carry broker order ID, exchange order ID, status, cumulative filled and pending quantities, average price, timestamps, tag and account identity. [Portfolio stream](https://upstox.com/developer/api-documentation/get-portfolio-stream-feed/)
- **U11 — Webhook:** App webhook accepts POST without required authentication, returns 2XX, and receives order updates; GTT updates require explicit app configuration. Payload shape matches stream updates. No signed-payload protocol was documented on the inspected page. [Webhook](https://upstox.com/developer/api-documentation/webhook/)
- **U12 — Current session support:** Upstox publishes a dedicated CAS announcement and current stream/instrument fields for session transitions. Session behavior must be date- and instrument-aware; the separate regulatory researcher supplies authoritative effective-date reconciliation. [Closing Auction Session support](https://upstox.com/developer/api-documentation/announcements/closing-auction-session/)

### Inferences

- Define subscription constraints as a broker/user budget shared across all sockets. Keep the Plus full_d30 printed table as a documented ambiguity: the combined limit is unexpectedly higher than its single-mode limit. Until broker confirmation, enforce the smaller validated D30 ceiling and do not infer 1500 thirty-level subscriptions.
- Compute a coverage plan rather than a fixed universe count. Reserve feed capacity for open positions and pending orders, then selected execution candidates, then scanner coverage and UI requests. Surface unsupported/unsubscribed/stale feature states. Capacity limits are external constraints, not product-imposed stock limits.
- Maintain two independent provider snapshots and provenance epochs. Data failover switches a symbol's active source only after instrument identity, freshness, session and baseline checks. Never combine cumulative volume counters across brokers or assume their sample streams have identical trades.
- Equity/index canonical identity must include exchange and series/classification; a cross-exchange listing is not the same executable instrument. Derivative identity includes underlying, venue, contract type, expiry, strike and right. Store all mappings with daily master generation.
- Treat `tick_size` master values as raw provider values until scale is validated against quote/order APIs; e.g. examples showing 5.0 are not sufficient proof of rupee units. Preserve raw plus normalized values and unit conversion evidence.
- Do not derive interval volume by adding last-traded quantity on every quote: repeated LTPC values may describe the same trade. Prefer cumulative volume deltas where available, detect resets, and flag gaps.
- Portfolio stream is primary. Webhook is redundant, deduplicated input. Because the inspected webhook contract has no authenticated signature, put it behind a controlled ingress, validate schema/account/known order and monotonic quantities, and reconcile through authenticated broker APIs before treating a suspicious update as authoritative. Never let a webhook directly create orders.

### Gaps

- Confirm actual Plus activation, subscription entitlement/charges, whether other sessions consume the same quota, BSE derivative feed availability and the full_d30 table interpretation with the account.
- No inspected feed contract promises every exchange trade, sequence-based replay, guaranteed arrival order, or bounded delivery latency. Candle precision must reflect sampled broker feeds.
- IV scales differ in examples between chain and stream. Confirm per-endpoint units and Greeks conventions; do not silently interchange numerical values.
- Exact webhook delivery retries, authentication alternatives, source IP ranges and guaranteed retention were not documented in the inspected page.

## Which historical and options APIs support live intelligence?

### Takeaway

Use historical V3 for warmup and gap repair, and distinguish broker candles from locally constructed bars. Option-chain snapshots provide derivatives fields, while selected contracts can remain streamed.

### Cited Findings

- **U13 — Historical V3:** `GET https://api.upstox.com/v3/historical-candle/{instrument_key}/{unit}/{interval}/{to_date}/{from_date}`. Minutes **1–300**, available from January 2022; intervals 1–15 permit a **one-month** request, above 15 permit **one quarter**. Hours **1–5**, available January 2022, **one-quarter** window. Daily interval 1, available January 2000, **one-decade** request. Weekly/monthly interval 1 available January 2000 with no stated window limit. Responses are candle arrays with timestamp/OHLC/volume/OI. Availability is a published range, not proof that every instrument existed or has complete history. [Historical Candle Data V3](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data/)
- **U14 — Current-day candles:** `GET /v3/historical-candle/intraday/{instrument_key}/{unit}/{interval}` supports minutes **1–300**, hours **1–5**, daily **1**, for the current trading day. [Intraday Candle Data V3](https://upstox.com/developer/api-documentation/v3/get-intra-day-candle-data/)
- **U15 — Contracts:** `GET /v2/option/contract?instrument_key={underlying}[&expiry_date=YYYY-MM-DD]` discovers active option contracts, expiry, strike, side and instrument key. [Option Contracts](https://upstox.com/developer/api-documentation/get-option-contracts/)
- **U16 — Chain:** `GET /v2/option/chain?instrument_key={underlying}&expiry_date=YYYY-MM-DD` provides per-strike calls/puts and PCR, with LTP, close, volume, OI, previous OI, bid/ask prices/quantities, delta, gamma, theta, vega, IV and provider POP. MCX chain is excluded in current docs; it is outside this platform's NSE/BSE scope. [Put/Call Option Chain](https://upstox.com/developer/api-documentation/get-pc-option-chain/)

### Inferences

- All requested 1, 2, 3, 5, 10, 15, 30 and 60-minute intervals can be represented by V3 directly; maintain one-minute canonical local aggregation to keep internal calculations reproducible. Keep broker and local variants with origin, adjustment and quality flags.
- Chunk backfill by documented calendar windows, not by guessed maximum bar count. Cache immutable completed responses; re-fetch overlap at disconnection boundaries and reconcile only after checking timezone/session/adjustment basis.
- Retain raw and adjusted series separately. No verified corporate-action adjustment guarantee was found, so reject continuity-sensitive signals across unresolved actions.
- PCR, futures buildup labels, relative OI, strike-liquidity scores, sector rankings, volatility regimes and opportunities are **internal calculations**, even when the broker supplies some input fields or a PCR snapshot.
- Provider POP is a provider model output, not calibrated confidence in this platform's strategy or a guaranteed trade success probability.
- Schedule option-chain refresh within the standard API budget, driven by active underlying/expiry demand; stream selected strikes for live price/liquidity changes. Never claim whole-chain REST snapshots are continuous tick streams.

### Gaps

- Validate BSE underlyings and expiries with account responses; broad endpoint schemas alone do not prove all contracts have chain/Greeks coverage.
- Confirm corporate-action adjustment policy, exact candle session anchoring, current incomplete-bar semantics, volume unit conventions and missing-bar behavior using read-only account samples.

## Which execution, portfolio, risk and rate-limit contracts are documented?

### Takeaway

V3 execution can return multiple child orders; the OMS must track request acceptance, broker order state and fills separately. Correlation/tag lookup helps reconciliation but does not provide documented idempotent order placement.

### Cited Findings

- **U17 — V3 placement:** `POST https://api-hft.upstox.com/v3/order/place` accepts quantity, product (`I`,`D`,`MTF`), validity (`DAY`,`IOC`), price, instrument_token, side, order_type, disclosed quantity, trigger price, tag, slice and market_protection. Automatic AMO inference ignores supplied `is_amo`. Slicing splits around freeze quantity and response `data.order_ids` is an array. `metadata.latency` measures broker API processing. `X-Algo-Name` is only required for an exchange-approved configured strategy. Source text mistakenly describes noncommodity quantity as a multiple of “tick size”; margin docs correctly specify lot size. [Place Order V3](https://upstox.com/developer/api-documentation/v3/place-order/)
- **U18 — Modify/cancel:** `PUT https://api-hft.upstox.com/v3/order/modify` and `DELETE https://api-hft.upstox.com/v3/order/cancel?order_id=...`. Modification docs have a narrative claiming optional fields but mark several table fields required; serialize explicit existing values for required fields. Terminal-order modification/cancellation is rejected. [Modify V3](https://upstox.com/developer/api-documentation/v3/modify-order/), [Cancel V3](https://upstox.com/developer/api-documentation/v3/cancel-order/)
- **U19 — Protection:** Market/SL-M `market_protection=-1` requests automatic protection, **1–25** selects a percentage, and **0** is rejected under market-order restrictions. This feature is labelled beta, effective March 11, 2026. Legacy examples with zero must not be copied as permitted behavior. [Market Price Protection](https://upstox.com/developer/api-documentation/announcements/market-protection/)
- **U20 — Multi-order:** `POST https://api.upstox.com/v2/order/multi/place`, maximum **10 resultant orders per request**, including slices. BUY processing precedes SELL processing; partial success can return HTTP 207. Per-item correlation IDs must be unique within the request and 1–20 characters; tag limit is 40. No atomic multi-leg guarantee is supplied. [Place Multi Order](https://upstox.com/developer/api-documentation/place-multi-order/)
- **U21 — Lookup:** `GET /v2/order/history` accepts order_id, tag, or both; tag alone returns all matching orders. `GET /v2/order/details?order_id=...` reads current status; `GET /v2/order/retrieve-all` is the current-day order book; `GET /v2/order/trades?order_id=...` lists fills. All use `https://api.upstox.com`. [History](https://upstox.com/developer/api-documentation/get-order-history/), [Details](https://upstox.com/developer/api-documentation/get-order-details/), [Order Book](https://upstox.com/developer/api-documentation/get-order-book/), [Order Trades](https://upstox.com/developer/api-documentation/get-trades-by-order/)
- **U22 — V2 compatibility:** V2 placement remains documented at `POST https://api-hft.upstox.com/v2/order/place`; do not infer endpoint removal merely because V3 exists. [Place Order V2](https://upstox.com/developer/api-documentation/place-order/)
- **U23 — GTT:** `POST https://api.upstox.com/v3/order/gtt/place` supports single and multiple rules including ENTRY, TARGET and STOPLOSS; beta trailing stop uses `trailing_gap` on a multi-leg STOPLOSS rule. [Place GTT](https://upstox.com/developer/api-documentation/place-gtt-order/)
- GTT modification is `PUT /v3/order/gtt/modify`; cancellation is `DELETE /v3/order/gtt/cancel` with JSON `gtt_order_id`; details are `GET /v3/order/gtt[?gtt_order_id=...]`. [Modify GTT](https://upstox.com/developer/api-documentation/modify-gtt-order/), [Cancel GTT](https://upstox.com/developer/api-documentation/cancel-gtt-order/), [GTT Details](https://upstox.com/developer/api-documentation/get-gtt-order-details/)
- **U24 — Portfolio:** `GET /v2/portfolio/short-term-positions` and `GET /v2/portfolio/long-term-holdings`, on api.upstox.com, supply broker account state. [Positions](https://upstox.com/developer/api-documentation/get-positions/), [Holdings](https://upstox.com/developer/api-documentation/get-holdings/)
- **U25 — Funds:** V3 is `GET https://api.upstox.com/v3/user/get-funds-and-margin` with richer available-to-trade cash/collateral and margin breakdowns; sample includes `Api-Version: 3.0`. V2 remains `GET /v2/user/get-funds-and-margin`, optional `segment=SEC|COM`. [Funds V3](https://upstox.com/developer/api-documentation/get-funds-and-margin-v3/), [Funds V2](https://upstox.com/developer/api-documentation/get-user-fund-margin/)
- **U26 — Pretrade margin:** `POST https://api.upstox.com/v2/charges/margin` accepts instruments with instrument_key, quantity, side, product and optional price. Quantity is a multiple of lot size. [Margin Details](https://upstox.com/developer/api-documentation/margin/)
- **U27 — Kill switch:** `POST /v2/user/kill-switch` operates by segment. Positions must already be closed to disable a segment; open orders are cancelled, and re-enabling has a **12-hour cooling period**. It cannot be treated as a flatten-position endpoint. [Kill Switch](https://upstox.com/developer/api-documentation/update-kill-switch/)
- **U28 — Rate limits:** Docs say limits are per API/per user, with a combined placement class covering place/modify/cancel/multi/GTT. Regular class: **10/sec, 500/min, 2000/30 min**; registered class: **50/sec, 500/min, 2000/30 min**. Other standard APIs: **50/sec, 500/min, 2000/30 min**. TOTP login: **1/sec, 10/min, 60/30 min**. Exceeding limits may suspend access. [Rate Limits](https://upstox.com/developer/api-documentation/rate-limiting/)

### Inferences

- Use V3 place/modify/cancel, V2 lookup, V3 GTT, V3 funds and V2 margin as the chosen manifest. Preserve compatibility endpoints in capability metadata without automatic write retries across versions.
- Canonical quantities are units; enforce instrument lot multiples and freeze caps. Decimal/fixed-point prices must respect normalized tick size and price bands. Do not carry documentation's “quantity tick size” typo into contracts.
- Default execution type can be limit and stop-limit. Expose protected-market capability only when segment/account/session certified; never equate schema enum presence with current eligibility.
- Maintain durable intent ID and per-attempt submission record before the network call. Generate broker-compatible unique tag. On transport timeout, mark SUBMISSION_UNKNOWN, query history by tag and reconcile order book/trades before any further decision. A missing immediate lookup is not proof of nonacceptance. No blind retry, cross-broker resubmission or “exactly once” broker claim.
- Treat sliced children as independently acknowledged/rejected/partially filled/cancelled. Sum fills with deduplication by broker trade identity and preserve average price correctly. A batch 207 needs item-level handling; never compensate an uncertain child automatically.
- Use account-local cash and margin reservations. A Dhan position cannot be closed by placing a contrary Upstox order: that creates separate exposure. Only market data can fail over automatically after quality checks.
- Implement all rolling windows, not just a per-second token bucket. Reserve emergency-order capacity inside the same hard envelope; include cancels/modifies/GTT. The 10-OPS exchange threshold and REST “requests” may count slices/batches differently: track conservative child-order counts and confirm broker interpretation.
- Funds snapshots are cached with age and generation; use live order/fill events for conservative local reservations. Obtain fresh broker margin when contract/configuration changes, and never treat cached estimates as a broker guarantee.
- An application kill switch first blocks new risk and allows controlled exits. Broker segment disable is a later verified-flat step, not the first emergency action.

### Gaps

- No inspected API provides a documented idempotency key with guaranteed replay deduplication. Tags/correlation IDs are lookup/correlation tools.
- Confirm market/SL-M support per segment and session, GTT entry/exit interactions after partial fills, trailing behavior, expiry cancellation, child-order modification limits and freeze behavior through authorized account certification.
- Clarify standard API budget grouping and child-order/slicing accounting before high-rate launch. Until then, apply conservative per-user standard and order-wide aggregate ceilings.
- Check fees, margin haircuts, product eligibility, daily square-off schedule, BSE contract support and vendor data usage rights for the actual account; do not hardcode them from examples.

## How should implementation prove the adapter is usable without inventing capabilities?

### Takeaway

The documentation establishes a credible dual-adapter design, but production eligibility remains an explicit external gate. Keep concrete capabilities and unresolved gates in the final handoff.

### Cited Findings

- **U29 — Error mapping:** Relevant documented errors include static-IP rejection `UDAPI1154`, invalid configured algo name `UDAPI1156`, blocked unprotected market order `UDAPI1158`, business-app restrictions `UDAPI100049`, terminal order changes, invalid instrument, expired token and rate-limit violations. Preserve broker codes alongside normalized categories. [Place V3](https://upstox.com/developer/api-documentation/v3/place-order/), [Modify V3](https://upstox.com/developer/api-documentation/v3/modify-order/), [Cancel V3](https://upstox.com/developer/api-documentation/v3/cancel-order/)
- **U30 — Versioned session behavior:** CAS support is an announced addition rather than a timeless 09:15–15:30 session assumption. Current metadata and stream formats expose eligibility and phases. [CAS announcement](https://upstox.com/developer/api-documentation/announcements/closing-auction-session/)

### Inferences

Required acceptance evidence, as engineering design rather than broker promises:

1. Download BOD masters; validate all requested NSE/BSE asset classes, reject duplicate/expired/ambiguous contracts and establish price/quantity scales.
2. Authenticate using the selected approved flow; validate token class, account identity, active segments and registered egress. Verify read-only analytics cannot reach mutation methods.
3. Connect V3 with the current pinned Proto descriptor; decode LTPC/full/Greeks/index/auction messages; exercise binary subscribe/unsubscribe/change-mode, heartbeat and redirect renewal with synthetic protocol fixtures.
4. Compare instrument-mode coverage against shared user limits; expose degraded coverage rather than silently skipping scanners.
5. Fetch historical/current-day candles across all requested intervals and session boundaries; validate ordering, timestamp offsets, OI and volume units, empty intervals and adjustment uncertainty.
6. Establish order/position/holding streams; deduplicate redundant webhook updates and reconcile externally initiated account orders.
7. Verify durable OMS uncertainty handling with synthetic transport faults and contract fixtures, without a paper-trading or historical-replay subsystem.
8. Only after credentials, regulatory gates and operator authorization, conduct bounded live commissioning to verify each enabled product, order type, modify/cancel, fills, protection and recovery. The planning task itself performs none of these writes.
9. Store dated conformance evidence per broker/account/endpoint/segment. Automatically block only the affected uncertified feature when external capabilities change; keep other independently validated capabilities usable.

### Gaps

- No live conformance evidence, actual round-trip latency measurements, certified account feature matrix or guarantee of complete real-time universe coverage exists in this research. These must remain explicit implementation and go-live gates, not implied completed work.
