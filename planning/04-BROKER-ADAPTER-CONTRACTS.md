# Broker adapter contracts

DhanHQ, Upstox and Zerodha each implement the eight interfaces below as an independent adapter process. This document defines the platform contract, not executable code. Wire routes, versions and ceilings are fixed by [01](01-API-RESEARCH.md). An unsupported optional capability returns a structured error (UNSUPPORTED, or BROKER_FEATURE_ABSENT when the broker documents no such feature); a common interface never fabricates broker functionality.

## Shared types and rules

Every request carries RequestContext: request_id, trace_id, broker, account_id where account-scoped, credential_generation, deadline_monotonic_ns, capability_version and cancellation_token. Mutation requests additionally require owner_epoch, intent_id, command_id and compliance_profile_version. Cancelling a local wait after bytes might have been sent does **not** cancel the broker order or establish nonacceptance.

| Type | Required fields and semantics |
|---|---|
| InstrumentId | Stable platform UUID; never broker token or display symbol |
| InstrumentKey | exchange, segment, asset_class, series, underlying_id, expiry, strike, option_right; inapplicable fields null |
| BrokerInstrumentRef | broker, raw_id (Dhan SECURITY_ID, Upstox instrument_key, Zerodha instrument_token + exchange:tradingsymbol), raw_segment, master_version, valid_from, valid_to |
| Price | Signed integer atoms plus decimal scale; INR; tick alignment checked with integer arithmetic; Zerodha wire divisor per segment applied in the decoder only |
| Quantity | Signed integer units; order quantities positive; position signed; lot_size_units separate |
| Money | Integer minor units or wider fixed-point accumulator, currency, scale; no binary-float accounting |
| TimestampSet | exchange_time nullable, provider_time nullable, received_utc, received_monotonic_ns, normalized_monotonic_ns, clock_domain, host_boot_id |
| Quality | status, missing_fields, stale_fields, suspected_gap, source_epoch, measurement_basis, last_valid_time |
| SnapshotMeta | snapshot_id, observed_at, source_epoch, broker, completeness, coverage, raw_schema_version |
| Page | items, next_cursor nullable, completeness, as_of; cursor local or broker-derived explicitly |
| CapabilitySet | broker/account/tier/segment/product/session scope, feature flags, limits_ref (budget IDs), evidence_ref, source_class, certified_at, expires_at |

Enums (owner of record for the pack): Broker = DHAN | UPSTOX | ZERODHA; Side = BUY | SELL; AssetClass = EQUITY | INDEX | FUTURE | OPTION; Product = INTRADAY | DELIVERY | CARRYFORWARD | MTF; OrderType = LIMIT | STOP_LIMIT | PROTECTED_MARKET | PROTECTED_STOP_MARKET; TimeInForce = DAY | IOC | TTL(minutes). Raw MARKET/SL-M are retained in broker observations; new automated intents use the gated canonical protection policy. Unknown enum values are retained raw and quarantined, never silently mapped.

| Canonical | Dhan | Upstox | Zerodha |
|---|---|---|---|
| INTRADAY / DELIVERY / CARRYFORWARD / MTF | INTRADAY / CNC / MARGIN / MTF | I / D / D (F&O carryforward) / MTF | MIS / CNC / NRML / MTF |
| LIMIT / STOP_LIMIT | LIMIT / STOP_LOSS | LIMIT / SL | LIMIT / SL |
| PROTECTED_MARKET / PROTECTED_STOP_MARKET | MARKET / STOP_LOSS_MARKET (account-gated) | MARKET / SL-M + market_protection (-1 or 1–25) | MARKET / SL-M + market_protection (-1 or >0; 0 rejected) |
| DAY / IOC / TTL | DAY / IOC / UNSUPPORTED | DAY / IOC / UNSUPPORTED | DAY / IOC / TTL via validity_ttl |

Prices sent to external APIs are decimal strings or checked decimal serialization with explicit scale; Zerodha form-encoded bodies serialize the same decimal text. Incoming floats are validated for finiteness/range and converted using broker-specific rules. Lot multiples, tick sizes and freeze quantities are certified from actual metadata fixtures; Zerodha supplies no freeze quantity, so freeze comes from the exchange file or the other brokers' masters mapped by canonical instrument.

## Authentication and common lifecycle

CredentialCoordinator is a shared support service. Operations: begin_authorization(redirect_uri, state, nonce), complete_authorization(code_or_request_token, state), inspect_session(), renew_if_supported(token_origin), rotate(operator_input), revoke_local_generation(). It returns CredentialMetadata with account identity, token class/origin/tier, expiry, generation, granted scopes, segment status and secret reference; tokens remain in secret storage and adapter memory.

| Broker | Flow | Expiry/renewal | Invalidation signal |
|---|---|---|---|
| Dhan | Consent/API-key login or TOTP endpoint | Web token 24 h; `/RenewToken` only for active Web-origin tokens | DH-901 / feed 807–811 |
| Upstox | OAuth code → server exchange; documented TOTP endpoint | Next 03:30 boundary; no refresh grant | UDAPI token errors |
| Zerodha | Redirect login → `request_token` (minutes) → `POST /session/token` with SHA-256(api_key+request_token+api_secret) | "6 AM" next day (scheduled as 06:00 IST, renewed before); no refresh endpoint | TokenException/403, also raised by a login elsewhere; treat as session-wide |

All adapters expose describe_capabilities(scope), health(), start(context), close(reason). Start validates identity/capability generation (Zerodha: `GET /user/profile` exchanges/products/order_types versus manifest). A token rotation serializes account use, blocks new submissions, reconnects affected streams and reconciles account state. Unsupported renewal returns REAUTH_REQUIRED. [Dhan authentication](https://dhanhq.co/docs/v2/authentication/), [Upstox token](https://upstox.com/developer/api-documentation/get-token/), [Kite user](https://kite.trade/docs/connect/v3/user/).

## 1. LiveMarketDataAdapter

| Method | Request | Result / event | Failure semantics |
|---|---|---|---|
| connect | credential_generation, desired_modes, connection_group | FeedSession(session_id, source_epoch, negotiated_schema) | AUTH_EXPIRED, ENTITLEMENT, CONNECTION_LIMIT |
| subscribe | session_id, subscription_revision, list of {instrument_id, broker_ref, mode} | SubscriptionResult accepted/rejected/pending per item | No all-or-nothing assumption; reconcile desired state |
| unsubscribe | session_id, revision, instrument/mode keys | Per-item result | Local removal is not broker acknowledgement |
| change_mode | session_id, keys, new_mode, revision | Per-item transition status | If native change absent, safe unsubscribe/subscribe |
| stream | session_id, consumer_cursor (local only) | MarketObservation, FeedStatus, SubscriptionStatus, BrokerObservation (Zerodha order text frames forwarded to interface 6) | No claimed broker resume cursor |
| snapshot | refs, fields | QuoteSnapshot with coverage/age | Rate-scheduled recovery only (D-QUOTE, U-QUOTE, Z-QUOTE) |
| disconnect | session_id, reason | Closed acknowledgement | Mark data unavailable immediately |

Modes: LAST_PRICE, QUOTE, FULL5, GREEKS, DEPTH20, DEPTH30, DEPTH200. Mapping: Dhan ticker/quote/full plus depth sockets; Upstox ltpc/full/option_greeks/full_d30; Zerodha ltp/quote/full (GREEKS and deep modes BROKER_FEATURE_ABSENT). Mode is a request for fields, not an assertion that every instrument supports them: an index on any broker lacks volume/depth (Zerodha index packets are 28/32 bytes), and LAST_PRICE/LTPC cannot satisfy a volume scanner.

MarketObservation includes event_id (local), instrument_id, broker_ref, source_epoch, local_sequence, kind, timestamps, fields_present, ltp, last_trade_quantity, cumulative_volume, cumulative_turnover, average_traded_price, previous_close, session_ohlc, oi, previous_oi, oi_day_high/low, best_bid/ask, depth arrays and circuit limits. Each field is optional with quality metadata. local_sequence orders this adapter's output only. Trade ID exists only when documented and present. A repeated last quantity is not a new trade.

~~~json
{
  "schema_version": 1,
  "kind": "market.observation",
  "event_id": "evt-example-1",
  "instrument_id": "instrument-example",
  "broker": "ZERODHA",
  "source_epoch": 7,
  "local_sequence": 415,
  "fields_present": ["ltp", "cumulative_volume", "best_bid", "best_ask"],
  "ltp": {"atoms": 123450, "scale": 2},
  "cumulative_volume": 210040,
  "exchange_time": "2026-10-22T04:00:00Z",
  "received_utc": "2026-10-22T04:00:00.001Z",
  "received_monotonic_ns": 9000001000,
  "quality": {"measurement_basis": "BROKER_SNAPSHOT", "suspected_gap": false}
}
~~~

## 2. HistoricalDataAdapter

| Method | Request | Result |
|---|---|---|
| supported_intervals | instrument scope | IntervalCapabilities with unit, interval, availability/window, boundary convention, evidence class (Zerodha caps are staff-published) |
| fetch_candles | instrument_ref, interval, from_inclusive, to_exclusive, adjustment_basis, include_oi | CandlePage and raw boundary metadata |
| fetch_current_session | instrument_ref, interval, session_id | Current-session CandlePage with provisional flags (Upstox intraday endpoint; Dhan/Zerodha same endpoint with today's range) |
| plan_chunks | range, interval, capabilities | Ordered non-overlapping request windows plus overlap policy; truncation handling |

Candle contains instrument_id, interval, session_id, phase, open_time, close_time_exclusive, OHLC, volume nullable, turnover nullable, OI nullable, origin = BROKER_HISTORY | LOCAL_STREAM | LOCAL_AGGREGATE, provider, adjustment_basis = RAW | ADJUSTED | UNKNOWN, completeness, revision and source_epoch. Requested unsupported native interval returns UNSUPPORTED_INTERVAL with a local aggregation suggestion. Zerodha `continuous=1` is exposed only as a daily stitched series flagged CONTINUOUS_UNADJUSTED. Validation errors: RANGE_INVALID, WINDOW_EXCEEDED, INSTRUMENT_INACTIVE, ADJUSTMENT_UNKNOWN, TRUNCATED (fewer rows than window implies). A zero-row response is not proof of no trades.

## 3. InstrumentAdapter

| Method | Request | Result |
|---|---|---|
| fetch_master | exchange/segments, conditional checksum | RawMasterArtifact(source_url, bytes_hash, downloaded_at, schema) |
| normalize_master | raw artifact, normalization_version | InstrumentSnapshot with accepted and quarantined rows |
| resolve | canonical key or exact broker_ref | Unique mapping or AMBIGUOUS/NOT_FOUND |
| list_contracts | underlying_id, asset_class, expiry range, active_on | Versioned contract list |
| get_restrictions | instrument/account/session | InstrumentRestrictions with evidence/age |
| diff_master | old/new versions | Added/changed/expired/reused-token events |

Instrument record stores canonical ID, exchange, segment, series, asset class, ISIN where available, display symbol, underlying ID, expiry, strike Price, right, lot units, tick Price, currency, freeze units (nullable with source), settlement type, active interval and raw metadata hash. Zerodha's CSV lacks ISIN, freeze and underlying: ISIN is joined from Dhan/Upstox masters or holdings; underlying is derived from `name`; derivatives are cached daily because tokens are reused after expiry. Restrictions (suspended, intraday_allowed, short_allowed, surveillance, F&O ban, buy/sell, price bands, CAS eligibility, permitted products) are TRUE | FALSE | UNKNOWN. [Dhan instruments](https://dhanhq.co/docs/v2/instruments/), [Upstox instruments](https://upstox.com/developer/api-documentation/instruments/), [Kite instruments](https://kite.trade/docs/connect/v3/market-quotes/).

## 4. OptionsDataAdapter

| Method | Request | Result |
|---|---|---|
| list_expiries | underlying broker_ref | ExpiryList with source time (Zerodha: derived from master) |
| list_option_contracts | underlying_id, expiry | Contracts with canonical mappings |
| fetch_chain | underlying_ref, expiry, requested_fields | OptionChainSnapshot; Zerodha returns BROKER_FEATURE_ABSENT |
| subscribe_selected_contracts | contract IDs, required mode | Delegated LiveMarketData subscription result |
| normalize_greeks | raw values, endpoint convention version | Greeks with scale/basis, model_source = PROVIDER \| LOCAL_MODEL, or UNIT_UNVERIFIED |

**ChainBuilder** is the platform's common chain assembler, used as the sole source for Zerodha and as the merge layer for Dhan/Upstox: join `option_contract` rows (from masters) for underlying+expiry, fill per-contract LTP/bid/ask/volume/OI from the selected feed mode or a quote snapshot (Z-QUOTE ≤500 keys at 1/s; U-QUOTE; D-QUOTE), fill underlying price from its own observation, then compute LOCAL_MODEL IV/Greeks with the `greeks` crate (Black-76 on the futures/forward price, bounded IV solver, convergence flag and residual). Provider values, when present, are kept beside local values; the strategy profile selects the basis and the UI badges it.

OptionChainSnapshot has chain_id, underlying_id, expiry, underlying_price/time, snapshot_time, strikes[{strike, call, put}], completeness, missing contracts and builder_version. Each side contains contract ID, LTP, bid/ask prices and sizes, volume, OI, previous OI, IV, delta/gamma/theta/vega, per-field ages, model_source and solver flags. Greeks record unit conventions (IV fraction vs percent, theta per day/year, vega per volatility point), model_source and calculation_time; LOCAL_MODEL records rate, forward basis and time-to-expiry convention. Unknown conventions or non-converged solves disable numerical risk aggregation while raw values remain displayable with a warning.

Provider PCR/POP remain provider fields, not strategy confidence. Platform PCR specifies universe, expiry, denominator validity and missing-strike coverage. Chain cadence: D-CHAIN per pair; U-STANDARD; Zerodha batches under Z-QUOTE or socket slots under Z-LIVE.

## 5. TradingAdapter

| Method | Request | Result |
|---|---|---|
| validate_order | OrderIntent, capability/rule versions | ValidationResult, normalized payload preview, estimated child count |
| place | CommittedOrderCommand | SubmissionResult with per-child outcomes and broker latency metadata |
| modify | committed command, broker_order_id, expected revision, desired total quantity/prices | MutationResult; later order update determines final state |
| cancel | committed command, broker_order_id, expected state | CancelRequestResult; not a fill/cancel guarantee |
| get_order | broker_order_id | BrokerOrderSnapshot |
| find_by_correlation | persisted correlation/tag, time/account scope | MatchSet with confidence and completeness (Dhan correlation endpoint; Upstox history by tag; Zerodha `GET /orders` filtered by tag locally) |
| list_orders | session/date scope | OrderPage |
| list_executions | order/session scope | ExecutionPage |
| place_protection | entry/position reference, protection plan | ProtectionResult with native child IDs or UNSUPPORTED |
| modify_protection / cancel_protection | committed command, native leg references | Per-leg results |
| bulk_cancel | account, segment/tag filter, committed command | Per-item results (Upstox ≤50, HTTP 207); Dhan/Zerodha UNSUPPORTED |
| bulk_exit | account, segment/tag filter, committed child commands | Per-position results (Dhan Exit All; Upstox ≤50, market hours, 207); Zerodha UNSUPPORTED |
| disable_trading | segment, verified-flat evidence, command | BrokerControlResult with cooldown; Zerodha BROKER_FEATURE_ABSENT |

OrderIntent includes intent_id, client_idempotency_key, broker/account, instrument/mapping version, side, product, quantity_units, order_type, limit/trigger prices, validity (+ttl_minutes), market_protection, strategy_id/version nullable, opportunity_id nullable, purpose = ENTRY | REDUCE | PROTECT | HEDGE, parent_intent_id, expires_at_utc, maximum_slippage, expected_account_version and risk/compliance versions. Source is MANUAL or AUTOMATED; both pass the same validation. Diagnostic monotonic timestamps carry clock_domain and host_boot_id and are never compared across reboot/hosts; expired or uncertain-age intents are never revived.

CommittedOrderCommand adds command_id, owner_epoch, durable_commit_reference, attempt_number, broker_correlation_token (Dhan correlationId ≤30, Upstox tag ≤40, Zerodha tag ≤20 alphanumeric) and payload_hash. Only the gateway constructs the broker request. Broker-specific placement: Dhan `/orders` or `/orders/slicing`; Upstox V3 place with `slice`; Zerodha `variety` selection (regular, amo, co, iceberg with iceberg_legs 2–50, auction), `autoslice` (≤10 children, each with its own order_id) and mandatory `market_protection` on MARKET/SL-M. SubmissionResult has outcome = ACKNOWLEDGED | DEFINITIVELY_REJECTED | NOT_SENT | SUBMISSION_UNKNOWN, broker_order_ids[], per_child[], raw_code, raw_response_ref, received_at and may_have_been_sent (C3).

Execution contains broker/account, broker_order_id, broker_trade_id when present, instrument/product, side, executed_units, price, exchange_time, received_time and source. Deduplication prefers broker trade identity; fallback fingerprints require collision handling and reconciliation. Order snapshots contain total, cumulative_filled, remaining, average_price, broker_status/raw_status and revision; Zerodha transient statuses map to PENDING_BROKER until a terminal or OPEN state is observed. Modification quantities are normalized to total order quantity including filled units; local modification caps are Dhan 25, Zerodha 20 of the documented 25, Upstox counted within U-ORDER.

Native protection mapping: Dhan Super Order (entry+target+stop+trailing jump) and Forever single/OCO; Upstox GTT single/multi-rule with beta trailing_gap; Zerodha GTT single or two-leg OCO (LIMIT orders only, no trailing) and CO variety where certified. Native multi-order is optional and non-atomic. The platform's child-order planner provides explicit per-child reservations and status, keeping every child under freeze limits. No adapter retries a write across API versions.

## 6. OrderUpdateAdapter

| Method | Request | Result / event |
|---|---|---|
| connect_updates | account, credential generation, types | UpdateSession (Dhan order socket; Upstox portfolio stream with update_types; Zerodha: shares the market socket, text frames) |
| stream_updates | session | OrderObserved, ExecutionObserved, PositionObserved, HoldingObserved, GttObserved, StreamHealth |
| accept_webhook | ingress request metadata + body | IngressReceipt and BrokerObservation; Zerodha checksum SHA-256(order_id+order_timestamp+api_secret) verified → authenticated_transport = true; Dhan/Upstox payloads remain untrusted |
| reconcile_snapshot | account, buffered event watermark | ReconciliationBundle and completeness |
| close_updates | reason | Closed |

BrokerObservation carries authenticated_transport, provenance = STREAM | WEBHOOK | REST, raw_payload_hash, account, broker IDs, status, cumulative quantities and timestamps. Webhook ingress never creates an order intent. Unknown account/order, decreasing cumulative quantity or impossible transition triggers quarantine and authenticated lookup. A raw status not mapped to the canonical lifecycle becomes BROKER_STATE_UNKNOWN and blocks dependent risk release. Zerodha socket frames also carry orders placed from Kite web/app and are imported with EXTERNAL origin; Zerodha postbacks cover only this api_key's orders. Dedupe Zerodha by order_id + exchange_update_timestamp + status. [Dhan order stream](https://dhanhq.co/docs/v2/order-update/), [Upstox portfolio stream](https://upstox.com/developer/api-documentation/get-portfolio-stream-feed/), [Kite postbacks](https://kite.trade/docs/connect/v3/postbacks/).

## 7. PortfolioAdapter

| Method | Request | Result |
|---|---|---|
| fetch_positions | account | PositionSnapshot with as_of/completeness (Zerodha net and day arrays) |
| fetch_holdings | account | HoldingSnapshot (Zerodha includes ISIN and authorised_quantity) |
| reconcile | local account version, broker snapshots, execution watermark | ReconciliationDiff and unresolved items |
| convert_product | committed authorized conversion command | MutationResult, gated per broker |
| authorise_holdings | account, holdings list, committed command | AuthorisationRequest (Zerodha CDSL flow: request_id, user completes at Kite URL, valid until 17:30; unauthorised sells fail HTTP 428); Dhan DDPI/Upstox TPIN are external gates, UNSUPPORTED here |
| request_exit_all | explicit emergency plan, account, committed child commands | Per-position outcomes; delegates to bulk_exit where it exists, otherwise per-position reducing commands |

Position identity is account + broker + instrument + product. It stores net units, bought/sold units and values, average cost basis, realized/unrealized P&L, external/manual contribution, snapshot generation and reconciliation status. Holdings separate settled, unsettled/T1, pledged/collateral and sell-authorized quantities. A position snapshot is not a synthetic fill; unexplained differences create adjustment records with evidence and block new exposure until resolved. [Dhan portfolio](https://dhanhq.co/docs/v2/portfolio/), [Upstox positions](https://upstox.com/developer/api-documentation/get-positions/), [Kite portfolio](https://kite.trade/docs/connect/v3/portfolio/).

## 8. FundsAndMarginAdapter

| Method | Request | Result |
|---|---|---|
| fetch_funds | account, segment scope | FundsSnapshot |
| estimate_margin | proposed order/portfolio, include_positions/orders flags | MarginEstimate and assumptions |
| estimate_basket | ordered legs, existing account generation | BasketMarginEstimate (Zerodha `/margins/basket` with consider_positions, spread benefit; Dhan `/margincalculator/multi`); Upstox UNSUPPORTED |
| estimate_charges | order or fill reference | ChargeEstimate (Zerodha `/charges/orders`; Upstox brokerage/trade-charges endpoints; Dhan margin-calculator brokerage field) |
| validate_funds_generation | cached generation, required age | FreshnessResult |

FundsSnapshot separates available cash, collateral, used/blocked margin, receivables, withdrawable amount and available-to-trade when supplied (Zerodha `available.live_balance`/`cash`/`collateral`, `utilised.span/exposure/option_premium`). MarginEstimate contains initial/total/SPAN/exposure/additional components, fees if supplied, source time, input hash and validity constraints. Missing components are null, never zero. Local reservations are never written back as broker balances. [Dhan funds](https://dhanhq.co/docs/v2/funds/), [Upstox margin](https://upstox.com/developer/api-documentation/margin/), [Kite margins](https://kite.trade/docs/connect/v3/margins/).

## Error taxonomy and retry contract

All errors contain category, broker_code, message_safe, request_id, operation, retry_after, retry_policy, may_have_been_sent, scope, raw_response_ref and capability_version.

| Category | Meaning | Permitted handling |
|---|---|---|
| AUTH_EXPIRED / REAUTH_REQUIRED | Credential invalid/expired or interactive approval needed | Pause route, renew approved flow, reconcile |
| ENTITLEMENT / SEGMENT_DISABLED | Data/product/tier unavailable | Disable affected capability, request evidence |
| RATE_LIMIT | Budget exceeded (HTTP 429 on all three) | Honor retry hint; schedule reads; writes require outcome certainty |
| VALIDATION / UNSUPPORTED | Invalid field or capability not implemented for this broker | Correct intent before any submission |
| BROKER_FEATURE_ABSENT | Broker documents no such feature | Use the named platform substitute; never retry |
| INSTRUMENT_UNKNOWN / AMBIGUOUS | Mapping absent/conflicting | Quarantine mapping |
| SESSION_RESTRICTED / COMPLIANCE_BLOCK | Current rule rejects action (static IP, market protection, phase) | No automatic alternative order type |
| INSUFFICIENT_FUNDS / MARGIN | Broker or local capital rejection | Release only confirmed-unused reservation |
| DELIVERY_AUTH_REQUIRED | Sale of holdings needs depository authorisation (Zerodha HTTP 428; Upstox CDSL TPIN; Dhan DDPI) | Block the sell route; run the broker's authorise flow; never retry blindly |
| BROKER_REJECTED | Definitive broker rejection | Persist reason; strategy reevaluates, no automatic repeat |
| TRANSPORT_NOT_SENT | Proven no request bytes dispatched | Same durable command may be scheduled if still valid |
| SUBMISSION_UNKNOWN | Request possibly accepted | Hold reservation; lookup/stream/trade reconciliation |
| DATA_STALE / DATA_GAP | Insufficient input quality | Invalidate affected features/opportunities |
| SCHEMA_DRIFT / UNIT_UNVERIFIED | Decoder or scale unsupported; solver non-convergence | Quarantine, retain safe raw diagnostic |
| OWNERSHIP_LOST | Gateway epoch/lease invalid | Fail closed and fence before takeover |

Raw codes stay visible beside categories: Dhan DH-901..DH-910 and feed 804..814; Upstox UDAPI1154 (static IP), UDAPI1156 (algo name), UDAPI1158 (unprotected market), UDAPI1108–1113 (bulk operations) and others; Zerodha TokenException (403, includes login elsewhere), UserException, OrderException, InputException (400), MarginException, HoldingException (428 unauthorised sell), NetworkException (OMS unreachable), DataException, GeneralException, with HTTP 400/403/404/405/410/429/500/502/503/504. Never infer retry safety solely from HTTP 5xx or NetworkException. [Dhan errors](https://dhanhq.co/docs/v2/annexure/), [Upstox placement errors](https://upstox.com/developer/api-documentation/v3/place-order/), [Kite exceptions](https://kite.trade/docs/connect/v3/exceptions/).

## Conformance acceptance

Run the same semantic suite against each of the three adapters independently: identity/units (including Zerodha divisors and index packets), interval boundaries and truncation, subscription partial success and slot caps, malformed binary/Protobuf, token generation change and login-elsewhere invalidation, unknown enums, rate windows, order uncertainty, duplicate/reordered updates (socket vs postback vs REST), partial fills, iceberg/autoslice children, GTT two-leg behavior, checksum verification and snapshot races. ChainBuilder fixtures compare LOCAL_MODEL Greeks with provider values on Dhan/Upstox chains within a stated tolerance. Account-dependent tests remain pending until credentials and authorization exist. A shared interface passes only when all three implementations preserve semantics; contract changes increment schema versions, supply migration rules and retain raw evidence.
