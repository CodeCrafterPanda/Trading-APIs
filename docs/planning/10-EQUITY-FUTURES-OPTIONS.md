# Product-specific intelligence and trading

Product support is complete at the contract level for all three brokers and enabled only through account/segment certification. Discovery, data availability, analytical readiness and trading permission remain separate state dimensions. Broker manifests are in [01](01-API-RESEARCH.md); every submitted order follows [11](11-ORDER-EXECUTION-ENGINE.md).

## Instrument identity and eligibility

Canonical equity identity includes exchange, segment, ISIN and series. NSE and BSE listings sharing an ISIN remain distinct executable instruments. A future is identified by venue, underlying, expiry and contract type; an option adds exact strike and CE/PE. Never derive tradable IDs from display strings or reuse expired exchange tokens (Zerodha reuses `instrument_token` after expiry; Upstox reuses exchange tokens). Version every mapping and invalidate candidates when lot/tick/expiry metadata changes.

| Master | Identity key | Present | Absent (platform supplies) |
|---|---|---|---|
| Dhan detailed CSV | segment + SECURITY_ID | ISIN, underlying ID, lot, tick, expiry, strike, CE/PE, freeze qty, bands, ASM/GSM, buy/sell indicator | — |
| Upstox BOD JSON | instrument_key | ISIN, lot, tick, freeze qty, expiry, underlying key, strike, CE/PE/FUT, weekly flag, cas_eligible, suspended/MIS lists | — |
| Zerodha daily CSV | exchange:tradingsymbol (+instrument_token) | name, expiry, strike, tick, lot, instrument_type, segment | ISIN (from holdings or Dhan/Upstox master via symbol+exchange), freeze quantity (NSE contract file), underlying link (derived from `name`) |

Eligibility evaluates enabled segment/product, instrument active date, suspension, surveillance, trade-to-trade status, F&O ban, buy/sell restrictions, price bands, lot/freeze, account category, session and broker square-off deadline. Missing restrictions are UNKNOWN, not permissive defaults. Index instruments supply context and underlying references; execution uses an eligible derivative or equity instrument. [Dhan instruments](https://dhanhq.co/docs/v2/instruments/), [Upstox instruments](https://upstox.com/developer/api-documentation/instruments/), [Kite instruments](https://kite.trade/docs/connect/v3/market-quotes/)

## Broker product and order constraints

| Constraint | Dhan | Upstox | Zerodha |
|---|---|---|---|
| Products | INTRADAY, CNC, MARGIN (F&O carryforward), MTF | I, D, MTF | MIS, CNC, NRML, MTF |
| Order types | LIMIT, MARKET, STOP_LOSS, STOP_LOSS_MARKET | LIMIT, MARKET, SL, SL-M; `market_protection` −1 or 1–25 % mandatory on MARKET/SL-M | LIMIT, MARKET, SL, SL-M; `market_protection` −1 or percent mandatory on MARKET/SL-M (0 rejected) |
| Validity | DAY, IOC | DAY, IOC | DAY, IOC, TTL (`validity_ttl` minutes; range uncertified) |
| Varieties | AMO/BO fields on the same endpoint | AMO inferred automatically | regular, amo, co, iceberg (2–50 legs), auction |
| Above freeze quantity | `/orders/slicing` | `slice=true`; multi ≤10 resulting orders, HTTP 207 | `autoslice=true` ≤10 children, each with its own order_id; or iceberg legs |
| Delivery sale authorisation | DDPI/POA status from `/profile` | CDSL TPIN/OTP without DDPI/POA | `POST /portfolio/holdings/authorise` → user completes CDSL flow; unauthorised sell returns HTTP 428 |

Automated intents default to LIMIT and permitted SL-LIMIT (ADR-08). Protected market/stop-market, IOC, TTL, delivery and MTF are represented but require account/session certification. An available holding is not proof it can be sold through the API. [Dhan orders](https://dhanhq.co/docs/v2/orders/), [Upstox orders](https://upstox.com/developer/api-documentation/orders/), [Kite orders](https://kite.trade/docs/connect/v3/orders/), [Kite portfolio](https://kite.trade/docs/connect/v3/portfolio/)

## Equity intraday long and short

Long entries reserve cash/margin and use the permitted intraday product. Shorts require explicit broker/instrument/account permission and a planned same-session exit before broker deadlines; naked short delivery is not permitted under the referenced framework, and institutional account restrictions differ from retail. [SEBI short-selling framework](https://www.sebi.gov.in/sebi_data/commondocs/dec-2024/RE_Chapter%201%20-%20Trading%20-%20NEW_p.pdf)

Quantity sizing uses min(risk_budget / per_unit_stop_risk, affordable_units, liquidity_participation_limit), rounded down to valid lot units; cash-equity lot is metadata-driven. per_unit_stop_risk includes adverse distance and allocated cost/slippage stress. Rounding to zero generates no order.

Exit plans distinguish price stop, profit target, time exit, regime invalidation and emergency reduction. All converge on one account-owned ExitController so two triggers cannot each sell the full position. Partial fills resize protection to confirmed net exposure.

## Futures contracts and rollover

Contract discovery loads actual expiry, underlying, lot size, tick size, freeze and settlement metadata from the dated masters above; Zerodha freeze quantities come from the NSE contract file. Never calculate expiries from weekday rules. [NSE contract information](https://www.nseindia.com/static/products-services/equity-derivatives-contract-information)

Futures quantity is units and must be a lot multiple. Notional = price × units × contract_multiplier with an explicit certified multiplier; never multiply by lot size twice. Margin comes from the execution broker (Dhan `/margincalculator`, Upstox `/v2/charges/margin`, Zerodha `/margins/orders`) including existing positions where supported, plus a configurable buffer.

Long and short futures share one lifecycle. Monitor basis, OI freshness, price bands, ban state and expiry proximity. Published F&O ban state gates new exposure; reduction-only behavior is validated separately. [NSE position limits](https://www.nseindia.com/static/products-services/equity-derivatives-position-limits)

Rollover is an explicit two-contract workflow: reduce the expiring contract, verify fills, then create a separately risk-checked next-contract intent. Parallel legs are permitted only under a certified basket policy with worst-case unhedged exposure reserved; default is sequential reduction-first. Stock derivatives create physical settlement obligations; default intraday policy stops new stock-derivative entries before an account-configured expiry cutoff and mandates earlier verified exit. [NSE physical settlement](https://nsearchives.nseindia.com/content/circulars/INSP47293.pdf)

## Option-chain pipeline

~~~mermaid
flowchart LR
  M[Dated masters] --> R[Contract registry: underlying, expiry, strike, right]
  R --> P{Provider chain?}
  P -- Dhan /optionchain, Upstox /v2/option/chain --> S[Provider snapshot: LTP, book, volume, OI, IV, Greeks]
  P -- Zerodha: none --> B[Platform ChainBuilder: /quote batches ≤500 @1/s + full-mode socket for selected strikes]
  S --> G[Greeks normalizer: PROVIDER units certified, LOCAL_MODEL cross-check]
  B --> L[LOCAL_MODEL Black-76 IV/Greeks]
  G --> C[ChainSnapshot]
  L --> C
  C --> F[Derived features: ATM, PCR, OI change, skew, liquidity scores]
~~~

Provider chains: Dhan `POST /optionchain` with expiry list (D-CHAIN one unique request per 3 s per underlying/expiry, concurrent pairs allowed); Upstox `GET /v2/option/chain` and `/v2/option/contract`, plus `GET /v3/market-quote/option-greek` (≤50 keys) for selected contracts and the bucketed `/v2/market/pcr`, OI and change-in-OI analytics as reference inputs only (not streaming). Provider PCR/POP are retained as provider fields, never adopted as platform confidence.

Zerodha ChainBuilder: join the daily CSV on `name`, `expiry`, `strike`, `instrument_type` (CE/PE) within `NFO-OPT`/`BFO-OPT`, batch `GET /quote` ≤500 keys at 1 req/s for the selected strike window, and stream selected strikes in full mode (OI, depth) within Z-LIVE. IV and Greeks are LOCAL_MODEL: Black-76 on the same-expiry futures price (or a synthetic forward from spot and the configured risk-free rate when no future exists), bounded bisection then Newton for IV with a convergence flag and residual; theta per day, vega per volatility point, IV as a fraction. Non-converged or stale-pair values carry UNIT_UNVERIFIED and disable Greek-dependent gates. The same LOCAL_MODEL runs as a cross-check on Dhan/Upstox chains; disagreement beyond tolerance lowers confidence rather than picking a side. ChainSnapshot records underlying time, expiry, per-field ages, strike coverage, source and model_source. [Dhan chain](https://dhanhq.co/docs/v2/option-chain/), [Upstox chain](https://upstox.com/developer/api-documentation/get-pc-option-chain/), [Upstox Greeks](https://upstox.com/developer/api-documentation/option-greek/), [Kite quotes](https://kite.trade/docs/connect/v3/market-quotes/)

ATM minimizes abs(strike − underlying reference) among valid strikes; ties take the lower strike and record the rule. Call ITM when S > K, put ITM when S < K. Moneyness and delta buckets are both available; a delta filter requires certified or converged Greeks. Keep all discovered contracts in the catalog; subscription capacity determines live coverage.

## Strike selection and options strategies

Bullish expressions: long call and defined-risk bullish spreads; bearish: long put and defined-risk bearish spreads. Short-option exposure requires explicit account/risk enablement, margin certification and worst-case stress limits; an order endpoint accepting SELL does not enable it.

Strike hard gates: active expiry, sufficient time to risk cutoff, fresh underlying/premium, two-sided quote, minimum volume/OI, maximum spread, lot affordability, converged/certified Greeks and acceptable impact. Rank remaining strikes by liquidity 0.35, spread/cost efficiency 0.25, delta suitability 0.20, expiry suitability 0.10, premium affordability 0.10; weights and target delta range are configurable.

A selected contract stores underlying opportunity ID, chain version, strike-selection explanation, entry premium region, premium stop, underlying invalidation, time exit and Greek stress assumptions. Revalidate before submission because chains lag streamed quotes.

Underlying-driven exits fire on structure invalidation; premium-driven exits respond to option price/risk; both feed the ExitController. If one source is stale, apply the certified fallback and broker-native protection. IV collapse, gamma acceleration near expiry and widening spread are explicit risk reasons.

Multi-leg orders are non-atomic on every broker. Default defined-risk opening submits the protective purchased leg first, verifies fill, then sizes the short leg to protected quantity; closing reduces short risk before disposing of protection. Zerodha `POST /margins/basket?consider_positions=true` and Dhan `/margincalculator/multi` give pre-trade basket margin with spread benefit; Upstox has no basket estimate, so legs are margin-checked individually with the worst-case unhedged reservation. Each leg has independent ACK/fill/rejection/unknown state; partial success retains residual exposure and escalates; no compensation order is sent for an unknown leg. Upstox multi-order BUY-first processing and Zerodha autoslice children do not establish atomic hedging. [Kite margins](https://kite.trade/docs/connect/v3/margins/), [Upstox multi](https://upstox.com/developer/api-documentation/place-multi-order/)

## Costs, market constraints and acceptance

CostModel is effective-dated by broker/exchange/segment/product/side: brokerage, exchange charges, STT, GST, stamp duty, SEBI turnover and other levies. The 2026 NSE schedule records option-sale STT 0.15 % of premium and futures-sale 0.05 %. Reconciliation inputs: Zerodha `POST /charges/orders` (virtual contract note), Upstox brokerage and trade-charges endpoints, Dhan margin calculator brokerage field and contract notes. [NSE STT](https://www.nseindia.com/static/products-services/equity-derivatives-securities-transaction-tax), [Kite charges](https://kite.trade/docs/connect/v3/margins/), [Upstox brokerage](https://upstox.com/developer/api-documentation/get-brokerage/)

Acceptance covers all three brokers and each enabled NSE/BSE asset class: mapping uniqueness including Zerodha ISIN enrichment and freeze-quantity sourcing, lot/tick/freeze checks, equity short restrictions, delivery-authorisation gates (HTTP 428 path), futures expiry/margin, CE/PE moneyness, stale chain rejection, provider vs LOCAL_MODEL Greek agreement on fixtures and solver non-convergence handling, partial multi-leg failure, residual exposure and account-owned exit routing. Unsupported account features remain visible as external gates.
