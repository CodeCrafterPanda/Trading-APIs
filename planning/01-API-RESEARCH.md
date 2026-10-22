# Broker capabilities and restriction manifest

Issued 22 October 2026; sources accessed 21 October 2026. This is the **canonical broker fact and limit manifest** for the pack. All numeric ceilings are configuration inputs with evidence dates, not constants scattered through strategy code. No credentials or live account validation were used.

## Evidence and version policy

Dhan's substantive documentation is under [dhanhq.co/docs/v2](https://dhanhq.co/docs/v2/); the supplied docs.dhanhq.co URLs returned generic landing content. Its route namespace remains v2 although feature release 2.5 is dated 9 February 2026. Upstox uses a mixed version manifest: V3 feed/history/primary execution/funds/GTT and V2 identity, order lookup, portfolio and margin. Never select a version by replacing a URL prefix globally. [Dhan releases](https://dhanhq.co/docs/v2/releases/), [Upstox API documentation](https://upstox.com/developer/api-documentation/)

The broker endpoint page and current release/operational announcement outrank an older marketing example. Conflicting prose is retained as a gate. Normalize documented capabilities separately from account-certified capabilities. API enums do not establish present-day segment/session permission.

## Capability matrix

| Capability | DhanHQ verified fact | Upstox verified fact | Platform decision |
|---|---|---|---|
| Identity | Client ID plus access token; consent/API credential and documented TOTP flows | OAuth authorization-code flow; detailed token page also documents TOTP endpoint | Server credential coordinator; approved interactive flow default |
| Ordinary expiry | Web-generated token 24 hours; active Web-origin renewal invalidates old token | Next 03:30 boundary documented; timezone/account metadata confirmation required | Persist origin, generation, class, expiry; block expired execution |
| Read-only special token | Do not assume a separate analytics class | One-year analytics token, GET-only, cannot trade | Separate read credential scope; mutations reject locally |
| Streaming | JSON requests, little-endian binary responses | Binary subscription requests, Protobuf V3 | Independent decoders and fixture certification |
| Ordinary depth | Full mode five levels | Full mode five levels | Optional fields and mode-aware scanner readiness |
| Deep data | Separate 20/200-level NSE feeds | Plus full_d30 30 levels | Entitlement-gated, never assumed universal |
| Instruments | CSV compact/detailed; segment REST | BOD JSON, separate suspended/MIS lists | Versioned imports, canonical mapping, units verification |
| History | Daily and intraday v2 | Historical/intraday V3 | Provider series and local aggregate series separate |
| Options | Expiry list and chain including OI, IV, Greeks | Contracts and chain including OI, IV, Greeks | Derived PCR/ranking internally, endpoint units verified |
| Orders | Place/modify/cancel/slicing, account lookup | V3 place/modify/cancel/slicing; V2 lookup/multi | Durable intent and ambiguity reconciliation |
| Live updates | Account-wide order WebSocket; token-scoped postback | Portfolio WebSocket; app webhook | Stream primary, webhook redundant and deduplicated |
| Portfolio | Holdings, positions, conversion, Exit All | Holdings and positions endpoints | Account/product ledgers remain separate |
| Funds/margin | Fund limit and single/multi margin calculator | Funds V3, margin V2 | Cached conservative reservations, fresh preflight as needed |
| Protection | Super order, Forever/OCO, P&L exit controls | GTT/multi-rule and beta trailing | Exact product/partial-fill behavior certification required |
| Emergency disable | Kill switch requires already flat/no pending orders | Kill switch requires flat positions; cancels open orders; 12h re-enable cooldown | Local entry block and controlled exits precede disable |

Authentication sources: [Dhan](https://dhanhq.co/docs/v2/authentication/), [Upstox OAuth](https://upstox.com/developer/api-documentation/authentication/), [Upstox token](https://upstox.com/developer/api-documentation/get-token/), [analytics token](https://upstox.com/developer/api-documentation/analytics-token/). Other row-specific sources appear below.

## Subscription and request budgets

| ID | Scope | Documented ceiling | Enforcement interpretation |
|---|---|---|---|
| D-LIVE | Normal feed user/connections | 5 connections/user; 5,000 instruments/connection; 100 instruments/subscribe message | Track all local sockets; other account consumers must be known |
| D-DEPTH20 | Separate 20-level socket | 50 instruments/connection and 50/request, NSE only | Total deep connections/shared quota unknown; do not multiply by five |
| D-DEPTH200 | Separate 200-level socket | 1 instrument/connection, NSE only | Account connection/entitlement gate |
| D-ORDER | Order requests | 10/s, 250/min, 1,000/h, 7,000/day | Conservative shared account budget; 25 modifications/order |
| D-DATA | Data API | 5/s; 100,000/day | Endpoint-specific rules additionally apply |
| D-QUOTE | Quote REST | 1/s; up to 1,000 instruments/request | Initialization/recovery, not streaming substitute |
| D-NONTRADE | Non-trading REST | 20/s | Conservative category/account grouping pending clarification |
| D-CHAIN | Unique chain request | One per 3s for underlying/expiry; different pairs documented concurrently | Enforce pair cadence plus account data ceiling |
| U-LIVE | Normal user | 2 sockets | Instrument caps are per user, not multiplied by sockets |
| U-LTPC | Mode alone / mixed modes | 5,000 / 2,000 keys | Apply category mixed-mode constraints together |
| U-GREEKS | Mode alone / mixed modes | 3,000 / 2,000 keys | Same per-user budget ownership |
| U-FULL | Mode alone / mixed modes | 2,000 / 1,500 keys | Full supplies volume/depth; LTPC does not |
| U-PLUS | Plus user | 5 sockets; D30 table prints 50 alone / 1,500 combined | Ambiguous D30 table: enforce smaller certified cap until clarified |
| U-ORDER | Combined place/modify/cancel/multi/GTT | Regular 10/s; registered 50/s; both 500/min and 2,000/30min | Higher second limit does not remove longer windows |
| U-STANDARD | Other standard APIs, per API/user wording | 50/s, 500/min, 2,000/30min | Conservative aggregate user limiter until grouping confirmed |
| U-TOTP | Token endpoint | 1/s, 10/min, 60/30min | Single credential coordinator; never brute-force |

[Dhan feed](https://dhanhq.co/docs/v2/live-market-feed/), [Dhan depth](https://dhanhq.co/docs/v2/full-market-depth/), [Dhan limits](https://dhanhq.co/docs/v2/), [Dhan quotes](https://dhanhq.co/docs/v2/market-quote/), [Dhan chain](https://dhanhq.co/docs/v2/option-chain/), [Upstox V3 feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/), [Upstox rates](https://upstox.com/developer/api-documentation/rate-limiting/).

The exchange algo threshold and broker REST request ceilings are different budgets. Count children, replacements and modifications conservatively until broker counting semantics are certified. Reserve protective capacity **inside** the hard envelope. Queue admission cannot guarantee that a cancel is accepted during broker outage or exhausted quota.

## Selected endpoint manifest

Dhan base is https://api.dhan.co/v2 unless noted. Exact enum and wire casing live in broker serializers, not shared models.

| Interface | Selected Dhan endpoints | Important wire detail |
|---|---|---|
| Identity | GET /profile; GET /RenewToken | Renewal limited to active Web-origin token |
| Instruments | GET /instrument/{exchangeSegment}; official detailed CSV | SECURITY_ID; raw scale validation; daily master hash |
| History | POST /charts/historical; POST /charts/intraday | Parallel OHLCV/timestamp arrays; daily toDate exclusive |
| Options | POST /optionchain; POST /optionchain/expirylist | UnderlyingScrip, UnderlyingSeg, Expiry |
| Trading | POST /orders; PUT/DELETE /orders/{id}; POST /orders/slicing | correlationId ≤30 characters; modify quantity means total quantity |
| Lookup | GET /orders; /orders/{id}; /orders/external/{correlationId}; /trades; /trades/{orderId} | Correlation lookup does not guarantee placement idempotency |
| Portfolio | GET /positions; /holdings; POST /positions/convert; DELETE /positions | Product conversion and Exit All are gated mutations |
| Funds | GET /fundlimit; POST /margincalculator; /margincalculator/multi | Preserve raw misspelling availabelBalance in parser only |
| Protection | /super/orders; /forever/orders; /pnlExit | Forever documented CNC/MTF; no assumed intraday F&O coverage |
| Disable | /killswitch | Flat/no pending prerequisite |

[Dhan instruments](https://dhanhq.co/docs/v2/instruments/), [history](https://dhanhq.co/docs/v2/historical-data/), [orders](https://dhanhq.co/docs/v2/orders/), [portfolio](https://dhanhq.co/docs/v2/portfolio/), [funds](https://dhanhq.co/docs/v2/funds/), [Super](https://dhanhq.co/docs/v2/super-order/), [Forever](https://dhanhq.co/docs/v2/forever/), [controls](https://dhanhq.co/docs/v2/traders-control/).

Upstox uses https://api.upstox.com except primary writes on https://api-hft.upstox.com.

| Interface | Selected Upstox endpoint/version | Important wire detail |
|---|---|---|
| Identity | POST /v2/login/authorization/token | Server exchange of one-use OAuth code; no invented refresh grant |
| Feed authorization | GET /v3/feed/market-data-feed/authorize | One-use authorized redirect URI |
| History | GET /v3/historical-candle/{key}/{unit}/{interval}/{to}/{from} | Date ordering differs from local from/to convention |
| Intraday | GET /v3/historical-candle/intraday/{key}/{unit}/{interval} | Current trading day |
| Options | GET /v2/option/contract; /v2/option/chain | Underlying key plus expiry |
| Placement | POST /v3/order/place on api-hft | order_ids array, slice, market_protection, optional X-Algo-Name |
| Modification/cancel | PUT /v3/order/modify; DELETE /v3/order/cancel on api-hft | Explicit required existing fields; order_id query for cancel |
| Lookup | GET /v2/order/history; /details; /retrieve-all; /trades | History by tag/order; current-day book |
| Multi | POST /v2/order/multi/place | ≤10 resulting orders including slices; HTTP 207 partial success |
| Protection | POST/PUT /v3/order/gtt/place or /modify; DELETE /v3/order/gtt/cancel; GET /v3/order/gtt | GTT cancellation has JSON ID body; beta trailing_gap |
| Portfolio | GET /v2/portfolio/short-term-positions; /long-term-holdings | Snapshot plus authenticated portfolio stream |
| Funds/margin | GET /v3/user/get-funds-and-margin; POST /v2/charges/margin | Analytics GET token does not imply POST margin permission |
| Disable | POST /v2/user/kill-switch | Segment-level; 12h re-enable cooldown |

[Authorize](https://upstox.com/developer/api-documentation/get-market-data-feed-authorize-v3/), [history](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data/), [intraday](https://upstox.com/developer/api-documentation/v3/get-intra-day-candle-data/), [contracts](https://upstox.com/developer/api-documentation/get-option-contracts/), [chain](https://upstox.com/developer/api-documentation/get-pc-option-chain/), [place V3](https://upstox.com/developer/api-documentation/v3/place-order/), [modify](https://upstox.com/developer/api-documentation/v3/modify-order/), [cancel](https://upstox.com/developer/api-documentation/v3/cancel-order/), [multi](https://upstox.com/developer/api-documentation/place-multi-order/), [history lookup](https://upstox.com/developer/api-documentation/get-order-history/), [GTT](https://upstox.com/developer/api-documentation/place-gtt-order/), [funds](https://upstox.com/developer/api-documentation/get-funds-and-margin-v3/), [margin](https://upstox.com/developer/api-documentation/margin/).

## Historical interval and window manifest

| Provider | Native intervals | Published availability/window | Consequence |
|---|---|---|---|
| Dhan daily | 1 day | Since inception; toDate exclusive | Chunk and validate endpoint boundary |
| Dhan intraday | 1,5,15,25,60 minutes | Five years for active instruments; ≤90 days/request | Build 2/3/10/30 locally; do not rename 25 as 30 |
| Upstox minutes | 1–300 | Since Jan 2022; 1–15: one month/request; >15: one quarter | Calendar-aware chunks |
| Upstox hours | 1–5 | Since Jan 2022; one quarter/request | Local hourly series still retained |
| Upstox day | 1 | Since Jan 2000; one decade/request | Actual listing date and gaps still apply |
| Upstox week/month | 1 | Since Jan 2000; no stated request-window limit | Bounded client requests anyway |

Neither published start date proves every contract has full history. Adjustment policy, missing-bar semantics, endpoint timezone/scale and incomplete-bar behavior remain certification gates. See [06](06-CANDLE-ENGINE.md).

## Streaming and update semantics

Dhan ordinary feed header is eight bytes; ticker lacks rich volume fields, quote adds cumulative volume and ATP, full adds OI and five-depth. Dedicated depth uses a different header/schema. Twenty-level sequence is explicitly marked ignored; do not claim reliable gap detection from it. The documented bid/ask prose contains inconsistencies, so certify against packet fixtures and snapshots. Dhan order stream is account-wide; postback scope is the originating access token. [Feed](https://dhanhq.co/docs/v2/live-market-feed/), [depth](https://dhanhq.co/docs/v2/full-market-depth/), [order stream](https://dhanhq.co/docs/v2/order-update/), [postback](https://dhanhq.co/docs/v2/postback/).

Upstox V3 includes distinct ltt/currentTs, market status and initial snapshot; field validity depends on mode and instrument. Portfolio stream can subscribe to order, GTT, position and holding updates. The inspected webhook documentation requires no authentication and supplies no verified signing scheme. Treat it as untrusted redundant input and corroborate suspicious state via authenticated endpoints. [Portfolio stream](https://upstox.com/developer/api-documentation/get-portfolio-stream-feed/), [webhook](https://upstox.com/developer/api-documentation/webhook/).

## Restrictions and external gates

Static approved egress is required for current trading routes. Upstox's current protected-market feature accepts -1 automatic protection or 1–25 percent, while zero is rejected; its beta status and segment/session eligibility remain explicit. LIMIT and permitted SL-LIMIT are the initial automated policy. [Dhan auth](https://dhanhq.co/docs/v2/authentication/), [Upstox deployed requirements](https://community.upstox.com/t/important-update-regulatory-changes-for-api-and-algo-trading-are-now-live/14874), [market protection](https://upstox.com/developer/api-documentation/announcements/market-protection/).

Before enabling a capability, obtain account identity/segment entitlement, actual subscription tier, price/tick/volume/Greek units, subscription scope, exact order policy, timestamp behavior, product protections and applicable data-use rights. Dhan deep connection ceilings, Upstox D30 ambiguity, precise rate-window accounting, corporate-action adjustment and BSE-specific operational circulars remain named open gates. A missing gate disables only its affected capability, except account identity, execution ownership and fundamental compliance gates, which disable all mutation routes.

Store source URL, accessed date, content hash, effective dates, account evidence and reviewer in each capability record. Daily startup checks token/segment/rule changes; weekly operational review checks documentation changes; changes affecting an active position trigger an immediate protective review. Full research provenance is retained in [Dhan notes](../research_notes/Indian%20trading%20platform%20blueprint/dhanhq.md), [Upstox notes](../research_notes/Indian%20trading%20platform%20blueprint/upstox.md), and [market requirements](../research_notes/Indian%20trading%20platform%20blueprint/market-regulation-and-architecture.md).
