# Product-specific intelligence and trading

Product support is complete at the contract level for both brokers and enabled only through account/segment certification. Discovery, data availability, analytical readiness and trading permission remain separate. The broker manifests are [01](01-API-RESEARCH.md); every submitted order follows [11](11-ORDER-EXECUTION-ENGINE.md).

## Instrument identity and eligibility

Canonical equity identity includes exchange, segment, ISIN and series. NSE and BSE listings sharing an ISIN remain distinct executable instruments. A future is identified by venue, underlying, expiry and contract type; an option adds exact strike and CE/PE. Never derive tradable IDs from display strings or reuse expired exchange tokens. Version every mapping and invalidate candidates when lot/tick/expiry metadata changes. [Dhan instruments](https://dhanhq.co/docs/v2/instruments/), [Upstox instruments](https://upstox.com/developer/api-documentation/instruments/).

Eligibility evaluates enabled segment/product, instrument active date, suspension, surveillance, trade-to-trade status, F&O ban, buy/sell restrictions, price bands, lot/freeze, account category, session and broker square-off deadline. Missing restrictions are UNKNOWN, not permissive defaults. Index instruments supply context and underlying references; execution uses an eligible derivative or equity instrument, not the index quote itself.

## Equity intraday long and short

Long entries reserve cash/margin and use permitted INTRADAY product. Shorts require explicit broker/instrument/account permission and a planned same-session exit before broker deadlines. Naked short delivery is not permitted under the referenced framework; institutional account restrictions differ from retail. Do not carry an intraday short into settlement by assuming the broker will handle it. [SEBI short-selling framework](https://www.sebi.gov.in/sebi_data/commondocs/dec-2024/RE_Chapter%201%20-%20Trading%20-%20NEW_p.pdf).

Quantity sizing uses min(risk_budget / per_unit_stop_risk, affordable_units, liquidity_participation_limit), rounded down to valid lot units. For cash equities lot is normally metadata-driven one unit, not assumed universally. per_unit_stop_risk includes adverse price distance and allocated cost/slippage stress. If rounding produces zero, no order is generated.

Default entries are LIMIT and protective STOP_LIMIT where allowed. Protected market/stop-market, IOC, delivery and MTF are represented but require account/session certification. Delivery sale authorization such as DDPI/POA or broker TPIN/OTP is a separate gate; an available holding is not proof it can be sold through API. [Upstox orders](https://upstox.com/developer/api-documentation/orders/).

Exit plans distinguish price stop, profit target, time exit, regime invalidation and emergency reduction. Multiple exit reasons converge on one account-owned exit controller so two triggers cannot each sell the full position. Partial fills resize protection to confirmed net exposure.

## Futures contracts and rollover

Contract discovery loads actual expiry, underlying, lot size, tick size, freeze and settlement metadata. Do not calculate all expiries from weekday rules. Use current exchange/broker contract files, including holiday adjustments. [NSE contract information](https://www.nseindia.com/static/products-services/equity-derivatives-contract-information).

Futures quantity is units and must be a multiple of lot_size_units. Notional = price×units×contract_multiplier, where multiplier is explicit and certified; for a contract whose quoted price already uses per-underlying-unit units, multiplier is one. Never multiply by lot size twice. Margin comes from the selected account broker and includes existing positions/orders where supported, plus a configurable safety buffer.

Long and short futures share the same lifecycle. Monitor underlying/futures basis, OI freshness, price bands, ban state and expiry proximity. Published F&O ban state gates new exposure; reduction-only behavior is separately validated. [NSE position limits](https://www.nseindia.com/static/products-services/equity-derivatives-position-limits).

Rollover is an explicit two-contract workflow: close/reduce the expiring contract, verify fills, then create a separately risk-checked next-contract intent. Parallel legs are permitted only under a certified basket policy with worst-case unhedged exposure reserved. No atomicity is assumed, and the default is sequential reduction-first. The platform does not silently carry an intraday strategy overnight.

Stock derivatives can create physical settlement obligations. Default intraday policy stops new stock-derivative entries before an account-configured expiry risk cutoff and mandates earlier verified exit. The cutoff is broker/product specific, not invented from generic market close. [NSE physical settlement](https://nsearchives.nseindia.com/content/circulars/INSP47293.pdf).

## Option-chain pipeline

Discover active expiries/contracts, fetch chain snapshots within broker cadence, map every CE/PE to canonical IDs, validate per-endpoint IV/Greek units, compute derived chain features, then stream selected actionable strikes. Keep all discovered contracts in the catalog; finite live subscription capacity determines coverage, not an arbitrary strike-universe cap.

ChainSnapshot records underlying time, expiry, per-field ages, strike coverage and source. OI change compares equivalent source/expiry/universe windows. PCR specifies OI or volume basis and selected moneyness range; missing sides and zero denominators produce unavailable results. Provider POP is never adopted as platform confidence. [Dhan chain](https://dhanhq.co/docs/v2/option-chain/), [Upstox chain](https://upstox.com/developer/api-documentation/get-pc-option-chain/).

ATM selection minimizes abs(strike−underlying reference price) among valid strikes; exact ties use the lower strike and record the tie rule. ITM/OTM depend on right: call ITM when S>K, put ITM when S<K. Moneyness and delta buckets are both available, but a delta filter requires certified Greek units.

## Strike selection and options strategies

For a bullish underlying opportunity, eligible expressions include long call and defined-risk bullish spreads; bearish includes long put and defined-risk bearish spreads. Short-option exposure is supported only under explicit account/risk enablement, margin certification and worst-case stress limits. It is never enabled merely because an order endpoint accepts SELL.

Strike candidate hard gates: active expiry, sufficient time to risk cutoff, fresh underlying/premium, two-sided quote, minimum volume/OI, maximum spread, lot affordability, confirmed unit conventions and acceptable impact. Rank remaining strikes by liquidity 0.35, spread/cost efficiency 0.25, delta suitability 0.20, expiry suitability 0.10 and premium affordability 0.10. Profile weights and target delta range are configurable; defaults are design choices, not claims of optimal returns.

A selected contract stores underlying opportunity ID, chain version, strike-selection explanation, entry premium region, premium stop, underlying invalidation, time exit and Greek stress assumptions. Revalidate before submission because the chain can lag streamed quotes.

Underlying-driven exits trigger when the underlying invalidates structure; premium-driven exits respond to actual option price/risk. Both feed the same exit controller. If one source is stale, use the certified fallback policy and broker-native protection; do not pretend a stale underlying stop protects a live premium position. IV collapse, gamma acceleration near expiry and widening spread are explicit risk reasons.

Multi-leg orders are non-atomic. Default defined-risk opening submits the protective purchased leg first, verifies fill, then sizes the short leg to protected quantity. Closing generally reduces short risk before disposing of protection. Each leg has independent ACK/fill/rejection/unknown state. Partial success retains residual exposure and escalates; no automatic “compensation” order is sent for a leg whose outcome is unknown. Upstox multi-order BUY-first behavior and 10-result limit do not establish atomic hedging. [Upstox multi](https://upstox.com/developer/api-documentation/place-multi-order/).

## Costs, market constraints and acceptance

CostModel is effective-dated by broker/exchange/segment/product/side and includes brokerage, exchange charges, STT, GST, stamp duty and other applicable levies. The 2026 NSE source records option-sale STT 0.15% of premium and futures-sale 0.05%; exercised-option treatment is separate. This is not a complete fee schedule. [NSE STT](https://www.nseindia.com/static/products-services/equity-derivatives-securities-transaction-tax).

Acceptance covers both brokers and each enabled NSE/BSE asset class: mapping uniqueness, lot/tick/freeze checks, equity short restrictions, futures expiry/margin, CE/PE moneyness, stale chain rejection, Greek units, partial multi-leg failure, residual exposure and account-owned exit routing. Unsupported account features remain visible as external gates rather than silently removed from the plan.
