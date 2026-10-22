# Broker capabilities and restriction manifest

Issued 22 October 2026. Dhan/Upstox sources accessed 21 October 2026 and re-checked 22 October; Zerodha Kite Connect sources accessed 22 October 2026. This is the **canonical broker fact and limit manifest** for the pack: every numeric ceiling, endpoint and version lives here as a configuration input with an evidence date. Other documents cite budget IDs and never restate values. No credentials or live account validation were used (C4 in [00](00-EXECUTIVE-OVERVIEW.md)).

## Evidence and version policy

| Broker | Version/transport facts | Source |
|---|---|---|
| DhanHQ | Route namespace `/v2` at `https://api.dhan.co/v2`; feature release 2.5 (9 Feb 2026) added conditional triggers, P&L exit, Exit All and TOTP token generation; 2.4 (Sep 2025) cut tokens to 24 h and mandated static IP for order APIs; supplied docs.dhanhq.co URLs return a landing page, substantive reference is dhanhq.co/docs/v2. Headers `access-token`, `client-id`; JSON bodies; errors `errorType/errorCode/errorMessage` | [Releases](https://dhanhq.co/docs/v2/releases/), [Introduction](https://dhanhq.co/docs/v2/) |
| Upstox | Mixed manifest: V3 feed/history/primary execution/funds/GTT/market-quote; V2 identity, order lookup, portfolio, margin, market information, bulk operations. Never select a version by replacing a URL prefix globally. `https://api.upstox.com`; primary writes on `https://api-hft.upstox.com`; `Authorization: Bearer` | [API documentation](https://upstox.com/developer/api-documentation/), [llms index](https://upstox.com/developer/api-documentation/llms.txt) |
| Zerodha | Single version v3 at `https://api.kite.trade`; header `X-Kite-Version: 3`; `Authorization: token api_key:access_token`; POST/PUT bodies `application/x-www-form-urlencoded` (margin endpoints JSON); timestamps `yyyy-mm-dd hh:mm:ss` IST; errors `status=error`, `message`, `error_type`; not CORS-enabled | [Introduction](https://kite.trade/docs/connect/v3/), [Response structure](https://kite.trade/docs/connect/v3/response-structure/) |

The broker endpoint page and current release/operational announcement outrank an older marketing example, forum post or SDK sample; conflicting prose is retained as a gate. Zerodha facts drawn from support articles, staff forum posts or official SDK code are labelled by source class in the research note. Documented capabilities are normalized separately from account-certified capabilities; API enums do not establish present-day segment/session permission.

## Capability matrix

| Capability | DhanHQ | Upstox | Zerodha | Platform decision |
|---|---|---|---|---|
| Identity | Client ID + access token; consent/API-key login; documented TOTP token endpoint | OAuth authorization-code exchange; detailed page documents TOTP endpoint | Redirect login → `request_token` (minutes) → `POST /session/token` with SHA-256(api_key+request_token+api_secret) | Server credential coordinator; approved interactive flow default |
| Ordinary expiry | Web token 24 h; `/RenewToken` only for active Web-origin tokens | Next 03:30 boundary (timezone unlabelled) | "Expires at 6 AM" next day (timezone unlabelled); login elsewhere invalidates; no refresh endpoint | Persist origin, generation, class, expiry; renew before boundary; block expired execution |
| Read-only credential | None documented | One-year analytics token, GET-only, cannot trade | Personal app tier: execution/portfolio/margins free, no quote/historical/socket | Separate read scope; mutations rejected locally; tier recorded in capability set |
| Streaming | JSON requests, little-endian binary responses; 8-byte header | Binary requests, Protobuf V3 | Big-endian binary; ltp 8 B / quote 44 B / full 184 B; index 28/32 B; text JSON frames for order updates | One decoder per wire format, fixture-certified |
| Ordinary depth | Full mode, 5 levels | Full mode, 5 levels | Full mode, 5 levels (10×12-byte entries) | Optional fields; mode-aware scanner readiness |
| Deep data | Separate 20/200-level NSE sockets | Plus tier full_d30 (30 levels) | None | Entitlement-gated, never assumed |
| REST snapshots | `/marketfeed/ltp,ohlc,quote` ≤1,000 instruments | V3 `/market-quote/quotes` ≤500, `/ohlc` and `/ltp`, option-greek ≤50 | `/quote` ≤500, `/quote/ohlc` ≤1,000, `/quote/ltp` ≤1,000 | Initialization/recovery only, never a stream substitute |
| Instruments | CSV compact/detailed; segment REST; ISIN, freeze, flags | BOD JSON; ISIN, freeze, `cas_eligible`; suspended/MIS lists; search API | Daily gzipped CSV; **no ISIN, no freeze quantity, no underlying**; tokens reused after expiry | Versioned daily imports, canonical mapping, cross-broker ISIN from Dhan/Upstox/holdings |
| History | Daily + intraday v2 (1/5/15/25/60 m, ≤90 d) | V3 minutes 1–300, hours 1–5, day/week/month | 8 intervals (minute…60minute, day); `oi=1`; `continuous=1` daily-only for expired futures; paid Connect tier | Provider series and local aggregate series separate ([06](06-CANDLE-ENGINE.md)) |
| Options data | Expiry list + chain with OI, IV, Greeks | Contracts + chain with OI, IV, Greeks, PCR/POP; option-greek REST; bucketed OI/PCR/max-pain analytics | **None** (BROKER_FEATURE_ABSENT): no chain/IV/Greeks endpoint | Platform ChainBuilder from masters + quotes/stream; `model_source=PROVIDER\|LOCAL_MODEL` ([04](04-BROKER-ADAPTER-CONTRACTS.md)) |
| Orders | Place/modify/cancel/slicing; correlation lookup | V3 place/modify/cancel/slice; V2 lookup/multi | `/orders/{variety}` regular/amo/co/iceberg/auction; iceberg 2–50 legs; autoslice ≤10 children; TTL validity; tag ≤20 | Durable intent + ambiguity reconciliation ([11](11-ORDER-EXECUTION-ENGINE.md)) |
| Live updates | Account-wide order WebSocket; token-scoped postback (unsigned) | Portfolio WebSocket (order/gtt/position/holding); app webhook (unsigned, URL-filtered) | Order text frames on market socket (covers Kite web/app orders); HTTP postback **checksum-signed** SHA-256(order_id+order_timestamp+api_secret) | Stream primary; webhook redundant; only Zerodha postback counts as authenticated |
| Portfolio | Holdings, positions, convert, Exit All | Holdings, positions, MTF positions, convert, Exit All ≤50 | Positions net/day, holdings with ISIN, convert; CDSL holdings authorise (HTTP 428 unauthorised sell); no exit-all | Account/product ledgers separate |
| Funds/margin | `/fundlimit`; margin calculator single/multi | Funds V3; margin V2; brokerage and trade-charges APIs | `/user/margins[/segment]`; `/margins/orders`, `/margins/basket` (spread benefit), `/charges/orders` virtual contract note | Cached conservative reservations; fresh preflight as needed |
| Protection | Super order (entry/target/stop/trailing jump); Forever/OCO (CNC/MTF); P&L exit; conditional triggers (equity/index) | GTT multi-rule ENTRY/TARGET/STOPLOSS, beta trailing_gap | GTT single / two-leg OCO (LIMIT only); alerts `ato` basket (500 active); CO variety | Exact product/partial-fill behavior certified per broker (gate G08) |
| Bulk operations | `DELETE /positions` Exit All | Cancel-multi ≤50, Exit All ≤50, HTTP 207 | None | Item-level handling inside the emergency sequence ([12](12-RISK-MANAGEMENT.md)) |
| Emergency disable | `/killswitch`: requires flat and no pending orders | Segment kill switch: requires flat, cancels orders, 12 h cooldown | **None via API** (BROKER_FEATURE_ABSENT); Kite web kill switch is manual | Platform ENTRY_HALTED and controlled exits precede any broker disable |
| Market information | None documented | `/v2/market/status/{exchange}` with CAS status; `/holidays[/date]`; `/timings/{date}` | None | Session engine input ([16](16-SECURITY-AND-COMPLIANCE.md)); never the only calendar source |

Authentication sources: [Dhan](https://dhanhq.co/docs/v2/authentication/), [Upstox OAuth](https://upstox.com/developer/api-documentation/authentication/), [Upstox token](https://upstox.com/developer/api-documentation/get-token/), [analytics token](https://upstox.com/developer/api-documentation/analytics-token/), [Kite user/session](https://kite.trade/docs/connect/v3/user/), [Kite pricing](https://zerodha.com/products/api). Row-specific sources follow each manifest.

## Subscription and request budgets

| ID | Scope | Documented ceiling | Enforcement interpretation |
|---|---|---|---|
| D-LIVE | Normal feed | 5 connections/user; 5,000 instruments/connection; 100 instruments/subscribe message | Track all local sockets; other account consumers must be known |
| D-DEPTH20 | 20-level socket | 50 instruments/connection and 50/request, NSE only | Shared-quota semantics unknown; do not multiply by five |
| D-DEPTH200 | 200-level socket | 1 instrument/connection, NSE only | Account connection/entitlement gate |
| D-ORDER | Order APIs | 10/s, 250/min, 1,000/h, 7,000/day; 25 modifications/order | Conservative shared account budget |
| D-DATA | Data APIs | 5/s; 100,000/day | Endpoint-specific rules additionally apply |
| D-QUOTE | Quote REST | 1/s; ≤1,000 instruments/request | Initialization/recovery only |
| D-NONTRADE | Non-trading REST | 20/s | Conservative account grouping pending clarification |
| D-CHAIN | Option chain | One unique (underlying, expiry) request per 3 s; different pairs concurrently | Pair cadence plus D-DATA ceiling |
| U-LIVE | Normal user sockets | 2 | Instrument caps are per user, not per socket |
| U-LTPC | Mode alone / mixed | 5,000 / 2,000 keys | Mixed-mode constraints applied together |
| U-GREEKS | Mode alone / mixed | 3,000 / 2,000 keys | Same per-user ownership |
| U-FULL | Mode alone / mixed | 2,000 / 1,500 keys | Full supplies volume/depth; LTPC does not |
| U-PLUS | Plus tier | 5 sockets; full_d30 50 instruments/connection (announcement); feed table prints 1,500 combined | Enforce 50 per connection until clarified |
| U-ORDER | Place/modify/cancel/multi/GTT combined | Regular 10/s; registered 50/s; both 500/min and 2,000/30 min | Longer windows always apply |
| U-STANDARD | Other standard APIs, per API/user | 50/s, 500/min, 2,000/30 min | Conservative aggregate limiter until grouping confirmed |
| U-QUOTE | Market Quote V3 | ≤500 keys/request (full); option-greek ≤50 keys | Counted inside U-STANDARD |
| U-BULK | Cancel-multi, Exit All | ≤50 orders/positions per request; HTTP 207; Exit All market hours only; separate undocumented rate limit | One in flight per account; item-level outcomes |
| U-TOTP | Token endpoint | 1/s, 10/min, 60/30 min | Single credential coordinator |
| Z-LIVE | Market socket | 3 connections/api_key; 3,000 instruments/connection | Hard slot budget 9,000; order-update frames share the socket |
| Z-ORDER | Order placement | 10/s (also exchange per-client-ID OPS threshold → 429); 400/min; 5,000/day; 25 modifications/order | Local modification cap 20; static IP mandatory |
| Z-QUOTE | Quote REST | 1/s; `/quote` ≤500 keys, `/quote/ohlc` and `/quote/ltp` ≤1,000 | Snapshot/recovery only |
| Z-HIST | Historical candles | 3/s; staff-published per-request caps (below) | Detect truncation; halve window |
| Z-OTHER | All other endpoints | 10/s | Shared across profile/margins/portfolio/GTT/alerts |

[Dhan feed](https://dhanhq.co/docs/v2/live-market-feed/), [Dhan depth](https://dhanhq.co/docs/v2/full-market-depth/), [Dhan limits](https://dhanhq.co/docs/v2/), [Dhan quotes](https://dhanhq.co/docs/v2/market-quote/), [Dhan chain](https://dhanhq.co/docs/v2/option-chain/), [Upstox V3 feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/), [Upstox Plus](https://upstox.com/developer/api-documentation/announcements/websocket-plus), [Upstox rates](https://upstox.com/developer/api-documentation/rate-limiting/), [Upstox quote V3](https://upstox.com/developer/api-documentation/get-full-market-quote-v3), [Upstox option-greek](https://upstox.com/developer/api-documentation/option-greek), [Upstox cancel-multi](https://upstox.com/developer/api-documentation/cancel-multi-order), [Upstox exit all](https://upstox.com/developer/api-documentation/exit-all-positions), [Kite exceptions/rates](https://kite.trade/docs/connect/v3/exceptions/), [Kite WebSocket](https://kite.trade/docs/connect/v3/websocket/), [Kite quotes](https://kite.trade/docs/connect/v3/market-quotes/).

The exchange algo threshold (10 orders/s per client, [16](16-SECURITY-AND-COMPLIANCE.md)) and broker REST ceilings are different budgets. Count children, slices, replacements and modifications conservatively until each broker's counting semantics are certified. Reserve protective capacity **inside** the hard envelope. Queue admission cannot guarantee that a cancel is accepted during broker outage or exhausted quota.

## Selected endpoint manifests

### DhanHQ (base `https://api.dhan.co/v2`)

| Interface | Endpoints | Wire detail |
|---|---|---|
| Identity | `GET /profile`; `GET /RenewToken`; `POST https://auth.dhan.co/app/generateAccessToken` (TOTP) | Renewal limited to active Web-origin token; IP endpoints `/ip/setIP,modifyIP,getIP`, 7-day change lock |
| Instruments | `GET /instrument/{exchangeSegment}`; detailed CSV `images.dhan.co/api-data/api-scrip-master-detailed.csv` | `SECURITY_ID`; raw scale validation; daily master hash |
| Snapshots | `POST /marketfeed/ltp`, `/ohlc`, `/quote` | D-QUOTE |
| History | `POST /charts/historical`; `POST /charts/intraday`; expired-options data page | Parallel OHLCV/timestamp arrays, epoch seconds; daily `toDate` exclusive |
| Options | `POST /optionchain`; `POST /optionchain/expirylist` | `UnderlyingScrip`, `UnderlyingSeg`, `Expiry`; includes `security_id`, `average_price`, IV, Greeks |
| Trading | `POST /orders`; `PUT/DELETE /orders/{id}`; `POST /orders/slicing` | `correlationId` ≤30 chars; modify quantity = total placed quantity |
| Lookup | `GET /orders`; `/orders/{id}`; `/orders/external/{correlationId}`; `/trades`; `/trades/{orderId}` | Correlation lookup is not placement idempotency |
| Portfolio | `GET /positions`; `/holdings`; `POST /positions/convert`; `DELETE /positions` (Exit All) | Gated mutations |
| Funds | `GET /fundlimit`; `POST /margincalculator`; `/margincalculator/multi` | Raw misspelling `availabelBalance` preserved in parser only |
| Protection | `/super/orders`; `/forever/orders`; `/pnlExit`; `/alerts/orders` conditional triggers (NSE_EQ/BSE_EQ/IDX_I conditions, price/indicator, default expiry 1 y, postback on trigger) | Forever documented CNC/MTF; triggers equities/indices only |
| Updates | `wss://api-order-update.dhan.co` (account-wide); HTTP postback (token-scoped) | JSON; field casing differs from REST |
| Disable | `/killswitch` | Flat/no pending prerequisite |

[Instruments](https://dhanhq.co/docs/v2/instruments/), [history](https://dhanhq.co/docs/v2/historical-data/), [orders](https://dhanhq.co/docs/v2/orders/), [portfolio](https://dhanhq.co/docs/v2/portfolio/), [funds](https://dhanhq.co/docs/v2/funds/), [Super](https://dhanhq.co/docs/v2/super-order/), [Forever](https://dhanhq.co/docs/v2/forever/), [conditional trigger](https://dhanhq.co/docs/v2/conditional-trigger/), [controls](https://dhanhq.co/docs/v2/traders-control/), [order stream](https://dhanhq.co/docs/v2/order-update/), [postback](https://dhanhq.co/docs/v2/postback/).

### Upstox (base `https://api.upstox.com`; primary writes `https://api-hft.upstox.com`)

| Interface | Endpoints | Wire detail |
|---|---|---|
| Identity | `POST /v2/login/authorization/token`; `POST /v2/totp-login/authorization/token`; `GET /v2/user/profile`; `GET /v2/user/ip` | One-use OAuth code; no refresh grant; user-level static IPs |
| Feed authorization | `GET /v3/feed/market-data-feed/authorize` | One-use authorized redirect URI |
| Snapshots | `GET /v3/market-quote/quotes` (≤500; depth, OI, circuits, CAS fields); `/v3/market-quote/ohlc`; `/v3/market-quote/ltp`; `/v3/market-quote/option-greek` (≤50) | U-QUOTE |
| Market information | `GET /v2/market/status/{exchange}` (+`cas_eligible_status`); `/v2/market/holidays[/{date}]`; `/v2/market/timings/{date}` | Timestamps 13-digit (ms, inferred) |
| History | `GET /v3/historical-candle/{key}/{unit}/{interval}/{to}/{from}`; `/v3/historical-candle/intraday/{key}/{unit}/{interval}`; expired instruments: `/v2/expired-instruments/...` contracts, expiries, candles | Date ordering to/from; calendar windows |
| Options | `GET /v2/option/contract`; `/v2/option/chain`; bucketed `/v2/market/pcr`, OI, change-in-OI, max-pain | Chain includes Greeks, IV, PCR, POP; analytics are not streaming |
| Placement | `POST /v3/order/place` (api-hft) | `order_ids[]`, `slice`, `market_protection` (-1 or 1–25; 0 rejected), optional `X-Algo-Name` |
| Modify/cancel | `PUT /v3/order/modify`; `DELETE /v3/order/cancel?order_id=` (api-hft) | Required existing fields serialized explicitly |
| Lookup | `GET /v2/order/history`; `/details`; `/retrieve-all`; `/trades`; `/v2/charges/historical-trades` | History by tag/order |
| Bulk | `POST /v2/order/multi/place` (≤10 results, 207); `DELETE /v2/order/multi/cancel` (≤50, 207); `POST /v2/order/positions/exit` (≤50, market hours, BUY first, 207) | U-BULK |
| Protection | `POST/PUT /v3/order/gtt/place,modify`; `DELETE /v3/order/gtt/cancel` (JSON id); `GET /v3/order/gtt` | Beta `trailing_gap` |
| Portfolio | `GET /v2/portfolio/short-term-positions`; `/long-term-holdings`; MTF positions; `PUT /v2/portfolio/convert-position` | Plus portfolio stream `wss://api.upstox.com/v2/feed/portfolio-stream-feed` |
| Funds/charges | `GET /v3/user/get-funds-and-margin`; `POST /v2/charges/margin`; `GET /v2/charges/brokerage`; trade-charges and P&L report | Analytics token does not imply POST margin |
| Disable | `POST /v2/user/kill-switch`; `GET` status | Segment-level; 12 h cooldown |

[Authorize](https://upstox.com/developer/api-documentation/get-market-data-feed-authorize-v3/), [history](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data/), [intraday](https://upstox.com/developer/api-documentation/v3/get-intra-day-candle-data/), [expired instruments](https://upstox.com/developer/api-documentation/expired-instruments), [contracts](https://upstox.com/developer/api-documentation/get-option-contracts/), [chain](https://upstox.com/developer/api-documentation/get-pc-option-chain/), [PCR](https://upstox.com/developer/api-documentation/get-pcr), [place V3](https://upstox.com/developer/api-documentation/v3/place-order/), [modify](https://upstox.com/developer/api-documentation/v3/modify-order/), [cancel](https://upstox.com/developer/api-documentation/v3/cancel-order/), [multi](https://upstox.com/developer/api-documentation/place-multi-order/), [history lookup](https://upstox.com/developer/api-documentation/get-order-history/), [GTT](https://upstox.com/developer/api-documentation/place-gtt-order/), [funds](https://upstox.com/developer/api-documentation/get-funds-and-margin-v3/), [margin](https://upstox.com/developer/api-documentation/margin/), [brokerage](https://upstox.com/developer/api-documentation/get-brokerage), [market status](https://upstox.com/developer/api-documentation/get-market-status), [holidays](https://upstox.com/developer/api-documentation/get-market-holidays), [timings](https://upstox.com/developer/api-documentation/get-market-timings), [portfolio stream](https://upstox.com/developer/api-documentation/get-portfolio-stream-feed/), [webhook](https://upstox.com/developer/api-documentation/webhook/), [market protection](https://upstox.com/developer/api-documentation/announcements/market-protection/).

### Zerodha Kite Connect (base `https://api.kite.trade`)

| Interface | Endpoints | Wire detail |
|---|---|---|
| Identity | Login `https://kite.zerodha.com/connect/login?v=3&api_key=`; `POST /session/token`; `DELETE /session/token`; `GET /user/profile` (`exchanges`, `products`, `order_types`) | Checksum exchange; profile is the only documented entitlement signal |
| Instruments | `GET /instruments`; `GET /instruments/{exchange}` | Gzipped CSV, daily (~08:30); key `exchange:tradingsymbol`; cache every day's file |
| Snapshots | `GET /quote?i=`; `/quote/ohlc`; `/quote/ltp` | Z-QUOTE; keys `EXCHANGE:TRADINGSYMBOL`, spaces as `+`; missing keys silently omitted |
| Streaming | `wss://ws.kite.trade?api_key=&access_token=` | JSON control `{"a":"subscribe\|unsubscribe\|mode","v":[...]}` |
| History | `GET /instruments/historical/{token}/{interval}?from&to&continuous&oi` | `[ts,o,h,l,c,v(,oi)]`, `+0530` timestamps; Connect tier only |
| Options | None | ChainBuilder from CSV (`segment` `*-OPT`, `name`, `expiry`, `strike`, `instrument_type`) + quotes/socket |
| Trading | `POST /orders/{variety}`; `PUT /orders/{variety}/{id}`; `DELETE /orders/{variety}/{id}` | Varieties regular/amo/co/iceberg/auction; `iceberg_legs` 2–50; `autoslice` ≤10 children; `validity_ttl`; `market_protection` on MARKET/SL-M; `tag` ≤20; `order_id` "does not guarantee receipt at the exchange" |
| Lookup | `GET /orders`; `/orders/{id}` (state history); `/trades`; `/orders/{id}/trades` | `guid` is not a client idempotency key; `exchange_order_id` null if never at exchange |
| Protection | `POST/GET/PUT/DELETE /gtt/triggers[/{id}]` (single, two-leg OCO; LIMIT only); `/alerts` (simple, `ato`; 500 active) | GTT is a trigger that places an order; no trailing |
| Updates | Socket text frames `{"type":"order"}`; HTTP postback to registered `postback_url` with `checksum` | Postback only for this api_key's orders; socket frames include Kite web/app orders |
| Portfolio | `GET /portfolio/positions` (net/day); `PUT /portfolio/positions` (convert); `GET /portfolio/holdings` (ISIN); `/holdings/auctions`; `POST /portfolio/holdings/authorise` | Unauthorised sell → HTTP 428 |
| Funds/margin | `GET /user/margins[/{segment}]`; `POST /margins/orders`; `POST /margins/basket?consider_positions=`; `POST /charges/orders` | JSON bodies; basket returns `initial`/`final` with spread benefit |
| Disable/bulk | None | BROKER_FEATURE_ABSENT |

[User](https://kite.trade/docs/connect/v3/user/), [Orders](https://kite.trade/docs/connect/v3/orders/), [GTT](https://kite.trade/docs/connect/v3/gtt/), [Alerts](https://kite.trade/docs/connect/v3/alerts/), [Portfolio](https://kite.trade/docs/connect/v3/portfolio/), [Margins](https://kite.trade/docs/connect/v3/margins/), [Quotes/instruments](https://kite.trade/docs/connect/v3/market-quotes/), [Historical](https://kite.trade/docs/connect/v3/historical/), [WebSocket](https://kite.trade/docs/connect/v3/websocket/), [Postbacks](https://kite.trade/docs/connect/v3/postbacks/), [Exceptions](https://kite.trade/docs/connect/v3/exceptions/), [Changelog](https://kite.trade/docs/connect/v3/changelog/).

## Historical interval and window manifest

| Provider | Native intervals | Published availability/window | Consequence |
|---|---|---|---|
| Dhan daily | 1 day | Since inception; `toDate` exclusive | Chunk and validate boundary |
| Dhan intraday | 1, 5, 15, 25, 60 min | Five years for active instruments; ≤90 days/request | Build 2/3/10/30 locally; never rename 25 as 30 |
| Upstox minutes | 1–300 | Since Jan 2022; 1–15: one month/request; >15: one quarter | Calendar-aware chunks |
| Upstox hours | 1–5 | Since Jan 2022; one quarter/request | Local hourly series retained |
| Upstox day / week / month | 1 | Since Jan 2000; decade/request (day); no stated window (week/month) | Bounded client requests |
| Zerodha | minute, 3minute, 5minute, 10minute, 15minute, 30minute, 60minute, day | Staff-published caps only: minute 60 d; 3/5/10 min 100 d; 15/30 min 200 d; 60 min 400 d; day 2,000 d; 3 req/s; `continuous=1` daily only for expired NFO/MCX futures | Treat caps as mutable; detect truncation and halve; 3/10/30 min native cross-check of local aggregates |

Neither published start date proves every contract has full history. Adjustment policy, missing-bar semantics, endpoint timezone/scale, incomplete-bar behavior and Zerodha index coverage remain certification gates ([06](06-CANDLE-ENGINE.md)).

## Streaming and update semantics

**Dhan.** Ordinary feed header is eight bytes; ticker lacks volume fields, quote adds cumulative volume/ATP/OHLC, full adds OI and five-level depth; separate packets for previous close and OI. Dedicated depth sockets use a 12-byte header and float64 prices; the 20-level sequence field is explicitly to be ignored. Server ping every 10 s; no pong for 40 s disconnects; a sixth socket displaces the first (805). Order stream is account-wide; postback scope is the originating token; neither is signed. [Feed](https://dhanhq.co/docs/v2/live-market-feed/), [depth](https://dhanhq.co/docs/v2/full-market-depth/), [order stream](https://dhanhq.co/docs/v2/order-update/).

**Upstox.** V3 Protobuf feed with distinct `ltt`/`currentTs`, market status, initial snapshot and auction fields; field validity depends on mode. Since 10 Oct 2024 `bidAskQuote.aq/bq` → `askQ/bidQ` and `marketOHLC.ohlc[].volume` → `vol` (int64); `ano`, `bno`, `fp`, `fv`, `dhoi`, `dloi` removed. Portfolio stream subscribes to `order,gtt_order,position,holding`. Webhook accepts unauthenticated POST; the April 2026 URL policy is a hosting filter, not a signing scheme. [Feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/), [proto changes](https://upstox.com/developer/api-documentation/announcements/market-proto-changes-notice), [webhook policy](https://upstox.com/developer/api-documentation/announcements/webhook-url-security-policy).

**Zerodha.** Frame = int16 packet count, then per packet int16 length + packet, big-endian. Tradeable packets: ltp 8 B, quote 44 B, full 184 B (token, LTP, last qty, ATP, day volume, buy/sell qty, OHLC, last-trade ts, OI, OI day high/low, exchange ts, 10 depth entries of 12 B). Index packets 28 B (quote) / 32 B (full) with no volume/depth. Prices in paise ÷100; CDS ÷10⁷; BCD ÷10⁴ and `segment = token & 0xff` are SDK-derived. One-byte heartbeat every few seconds; order updates/errors arrive as JSON text frames. No reconnect policy, sequence or replay is documented. [WebSocket](https://kite.trade/docs/connect/v3/websocket/), [pykiteconnect ticker](https://raw.githubusercontent.com/zerodha/pykiteconnect/master/kiteconnect/ticker.py).

No broker documents a universal exchange sequence, delivery guarantee, resumable cursor or complete trade tape; gap detection is suspected-gap detection ([05](05-MARKET-DATA-ENGINE.md)).

## Restrictions and external gates

Static approved egress is required for every execution route: Dhan all order APIs (7-day change lock), Upstox user-level primary/secondary, Zerodha order endpoints only from 1 April 2026 (1–2 IPs, one change per calendar week; data/read endpoints from any IP). Upstox protected-market accepts -1 or 1–25 % and rejects 0; Zerodha requires `market_protection` (-1 or >0) on MARKET/SL-M and rejects 0; Dhan MARKET permission is account/session gated. LIMIT and permitted SL-LIMIT are the initial automated policy (ADR-08). [Dhan auth](https://dhanhq.co/docs/v2/authentication/), [Upstox deployed requirements](https://community.upstox.com/t/important-update-regulatory-changes-for-api-and-algo-trading-are-now-live/14874), [Kite static IP](https://support.zerodha.com/category/trading-and-markets/general-kite/kite-api/articles/static-ip), [Kite algo rules](https://kite.trade/forum/discussion/comment/52171).

Zerodha-specific gates: Personal tier (free, no data) versus Connect tier (₹500/month incl. socket + historical; waived from Oct 2026 at ≥₹2,000 monthly brokerage); whether a Personal app receives socket order frames/postbacks; the 06:00 token boundary timezone; TTL range; GTT expiry/count; autoslice response shape; no idempotency key on any broker. [Kite pricing](https://zerodha.com/products/api), [fee revision](https://kite.trade/forum/discussion/comment/49590).

Before enabling a capability, obtain account identity/segment entitlement, subscription tier, price/tick/volume/Greek units, subscription scope, exact order policy, timestamp behavior, product protections and data-use rights. Open named gates: Dhan deep-connection ceilings; Upstox D30 table ambiguity and rate-window accounting; Zerodha historical caps and BCD divisor evidence class; corporate-action adjustment on all three; BSE operational circulars. A missing gate disables only its capability, except identity, execution ownership and fundamental compliance gates, which disable all mutation routes for that account.

Store source URL, accessed date, content hash, source class, effective dates, account evidence and reviewer in each capability record ([13](13-DATA-ARCHITECTURE.md) `broker_capability`). Daily startup checks token/segment/rule changes; weekly review checks documentation changes; changes affecting an active position trigger an immediate protective review. Full provenance: [Dhan notes](../research_notes/Indian%20trading%20platform%20blueprint/dhanhq.md), [Upstox notes](../research_notes/Indian%20trading%20platform%20blueprint/upstox.md), [Zerodha notes](../research_notes/Indian%20trading%20platform%20blueprint/zerodha.md), [market requirements](../research_notes/Indian%20trading%20platform%20blueprint/market-regulation-and-architecture.md).
