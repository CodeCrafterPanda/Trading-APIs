# Broker adapter contracts

Both DhanHQ and Upstox implement the eight interfaces below. This document defines the platform contract, not executable application code. Broker wire routes, versions and ceilings are fixed by [01](01-API-RESEARCH.md). Unsupported optional capabilities return a structured error; a common interface must not fabricate broker functionality.

## Shared types and rules

Every request carries RequestContext: request_id, trace_id, broker, account_id where account-scoped, credential_generation, deadline_monotonic_ns, capability_version and cancellation_token. Mutation requests additionally require owner_epoch, intent_id, command_id and compliance_profile_version. Cancellation of a local wait after bytes might have been sent does **not** cancel the broker order or establish nonacceptance.

| Type | Required fields and semantics |
|---|---|
| InstrumentId | Stable platform UUID; never broker token or display symbol |
| InstrumentKey | exchange, segment, asset_class, series, underlying_id, expiry, strike, option_right; inapplicable fields null |
| BrokerInstrumentRef | broker, raw_id, raw_segment, master_version, valid_from, valid_to; account capability reference where needed |
| Price | signed integer atoms plus decimal scale; normalized currency INR; tick alignment checked with integer arithmetic |
| Quantity | signed integer units; order quantities positive; position signed; lot_size_units separate |
| Money | integer minor units or wider fixed-point accumulator, currency, scale; no binary-float accounting |
| TimestampSet | exchange_time nullable, provider_time nullable, received_utc, received_monotonic_ns, normalized_monotonic_ns, clock_domain and host_boot_id |
| Quality | status, missing_fields, stale_fields, suspected_gap, source_epoch, measurement_basis, last_valid_time |
| SnapshotMeta | snapshot_id, observed_at, source_epoch, broker, completeness, coverage, raw_schema_version |
| Page | items, next_cursor nullable, completeness, as_of; cursor local or broker-derived explicitly |
| CapabilitySet | broker/account/segment/product/session scope, feature flags, limits_ref, evidence_ref, certified_at, expires_at |

Enum choices: Broker=DHAN|UPSTOX; Side=BUY|SELL; AssetClass=EQUITY|INDEX|FUTURE|OPTION; Product=INTRADAY|DELIVERY|CARRYFORWARD|MTF; OrderType=LIMIT|STOP_LIMIT|PROTECTED_MARKET|PROTECTED_STOP_MARKET; TimeInForce=DAY|IOC. Raw MARKET/SL-M can be retained in broker observations, but new automated intents use the gated canonical protection policy. Unknown enum values are retained as raw values and quarantined rather than silently mapped.

All prices sent to external JSON APIs are decimal strings or checked decimal serialization with explicit scale. Incoming floats are validated for finiteness/range and converted using broker-specific rules. Lot multiples, tick sizes and freeze quantities are certified from actual metadata fixtures; no assumption that all tick values are already rupees.

## Authentication and common lifecycle

CredentialCoordinator is a shared support service, not a ninth broker business adapter. Its operations are begin_authorization(redirect_uri,state,nonce), complete_authorization(code,state), inspect_session(), renew_if_supported(token_origin), rotate(operator_input), and revoke_local_generation(). It returns CredentialMetadata with account identity, token class/origin, expiry, generation, granted scopes, segment status and secret reference. Actual tokens remain in secret storage and adapter memory.

All adapters expose describe_capabilities(scope), health(), start(context), and close(reason). Start validates identity/capability generation. A token rotation serializes account use, blocks new submissions, reconnects affected streams and reconciles account state. Unsupported renewal returns REAUTH_REQUIRED. Dhan active Web-origin renewal and Upstox next-03:30 ordinary expiry must not be treated as a generic refresh-token grant. [Dhan authentication](https://dhanhq.co/docs/v2/authentication/), [Upstox token](https://upstox.com/developer/api-documentation/get-token/).

## 1. LiveMarketDataAdapter

| Method | Request | Result / event | Failure semantics |
|---|---|---|---|
| connect | credential_generation, desired_modes, connection_group | FeedSession(session_id, source_epoch, negotiated_schema) | AUTH_EXPIRED, ENTITLEMENT, CONNECTION_LIMIT |
| subscribe | session_id, subscription_revision, list of {instrument_id, broker_ref, mode} | SubscriptionResult accepted/rejected/pending per item | No all-or-nothing assumption; reconcile desired state |
| unsubscribe | session_id, revision, instrument/mode keys | Per-item result | Local removal is not broker acknowledgement |
| change_mode | session_id, keys, new_mode, revision | Per-item transition status | If native change absent, safe unsubscribe/subscribe sequence |
| stream | session_id, consumer_cursor(local only) | MarketObservation, FeedStatus, SubscriptionStatus | No claimed broker resume cursor |
| snapshot | refs, fields | QuoteSnapshot with coverage/age | Rate scheduled recovery only |
| disconnect | session_id, reason | Closed acknowledgement | Mark data unavailable immediately |

Modes are LAST_PRICE, QUOTE, FULL5, GREEKS, DEPTH20, DEPTH30, DEPTH200 with per-broker capability flags. Mode is a request for fields, not an assertion that every instrument supports them. A Dhan index or Upstox LTPC subscription cannot satisfy a volume scanner.

MarketObservation payload includes event_id (local), instrument_id, broker_ref, source_epoch, local_sequence, kind, timestamps, fields_present, ltp, last_trade_quantity, cumulative_volume, cumulative_turnover, average_traded_price, previous_close, session_ohlc, oi, previous_oi, best_bid/ask and depth arrays. Each field is optional with quality metadata. local_sequence orders this adapter's output and is not an exchange sequence. Trade ID exists only when documented and present. Repeated last quantity is not a new trade.

~~~json
{
  "schema_version": 1,
  "kind": "market.observation",
  "event_id": "evt-example-1",
  "instrument_id": "instrument-example",
  "broker": "UPSTOX",
  "source_epoch": 7,
  "local_sequence": 415,
  "fields_present": ["ltp", "cumulative_volume", "best_bid", "best_ask"],
  "ltp": {"atoms": 123450, "scale": 2},
  "cumulative_volume": 210040,
  "exchange_time": null,
  "received_utc": "2026-10-22T04:00:00.001Z",
  "received_monotonic_ns": 9000001000,
  "quality": {"measurement_basis": "BROKER_SNAPSHOT", "suspected_gap": false}
}
~~~

## 2. HistoricalDataAdapter

| Method | Request | Result |
|---|---|---|
| supported_intervals | instrument scope | IntervalCapabilities with unit, interval, availability/window, boundary convention |
| fetch_candles | instrument_ref, interval, from_inclusive, to_exclusive, adjustment_basis, include_oi | CandlePage and raw boundary metadata |
| fetch_current_session | instrument_ref, interval, session_id | Current-session CandlePage, provisional flags |
| plan_chunks | range, interval, capabilities | Ordered non-overlapping request windows plus overlap policy |

Candle contains instrument_id, interval, session_id, phase, open_time, close_time_exclusive, OHLC, volume nullable, turnover nullable, OI nullable, origin=BROKER_HISTORY|LOCAL_STREAM|LOCAL_AGGREGATE, provider, adjustment_basis=RAW|ADJUSTED|UNKNOWN, completeness, revision and source_epoch. fetch_candles normalizes order and bounds without hiding missing bars. Requested unsupported native interval returns UNSUPPORTED_INTERVAL with local aggregation suggestion; it does not silently return a different interval.

Validation errors include RANGE_INVALID, WINDOW_EXCEEDED, INSTRUMENT_INACTIVE and ADJUSTMENT_UNKNOWN. A zero-row response carries provider status and coverage; it is not automatically proof of no trades.

## 3. InstrumentAdapter

| Method | Request | Result |
|---|---|---|
| fetch_master | exchange/segments, conditional checksum | RawMasterArtifact(source_url, bytes_hash, downloaded_at, schema) |
| normalize_master | raw artifact, normalization_version | InstrumentSnapshot with accepted rows and quarantine rows |
| resolve | canonical instrument key or exact broker_ref | Unique mapping or AMBIGUOUS/NOT_FOUND |
| list_contracts | underlying_id, asset_class, expiry range, active_on | Versioned contract list |
| get_restrictions | instrument/account/session | InstrumentRestrictions with evidence/age |
| diff_master | old/new versions | Added/changed/expired/reused-token events |

Instrument record stores canonical ID, exchange, segment, series, asset class, ISIN where relevant, display symbol, underlying ID, expiry timestamp/date, strike Price, right, lot units, tick Price, currency, freeze units, settlement type, active interval and raw metadata hash. Corporate-action relationships are separate entities; a symbol rename does not create accidental economic equivalence.

Restrictions include suspended, intraday_allowed, short_allowed, surveillance, F&O ban, buy/sell restrictions, price bands, CAS eligibility and permitted products. Every restriction is TRUE|FALSE|UNKNOWN, not a missing boolean defaulting to true. Dhan/Upstox master evidence: [Dhan instruments](https://dhanhq.co/docs/v2/instruments/), [Upstox instruments](https://upstox.com/developer/api-documentation/instruments/).

## 4. OptionsDataAdapter

| Method | Request | Result |
|---|---|---|
| list_expiries | underlying broker_ref | ExpiryList with source time |
| list_option_contracts | underlying_id, expiry | Contracts with canonical mappings |
| fetch_chain | underlying_ref, expiry, requested_fields | OptionChainSnapshot |
| subscribe_selected_contracts | contract IDs, required mode | Delegated LiveMarketData subscription result |
| normalize_greeks | raw values, endpoint convention version | Greeks with scale/basis or UNIT_UNVERIFIED |

OptionChainSnapshot has chain_id, underlying_id, expiry, underlying_price/time, snapshot_time, strikes[{strike, call, put}], completeness and missing contracts. Each side contains contract ID, LTP, bid/ask prices and sizes, volume, OI, previous OI, IV, delta/gamma/theta/vega and separate per-field ages. Greeks record unit conventions (IV fraction vs percent, theta per day/year, vega per volatility point), model_source and calculation_time. Unknown conventions disable numerical risk aggregation; raw values remain displayable with a warning.

Provider PCR/POP are retained as provider fields, not strategy confidence. Platform aggregate PCR specifies universe, expiry, denominator validity and missing-strike coverage. Chain REST cadence and selected-strike streaming are separate. Dhan pair cadence is governed by D-CHAIN; Upstox uses U-STANDARD.

## 5. TradingAdapter

| Method | Request | Result |
|---|---|---|
| validate_order | OrderIntent, capability/rule versions | ValidationResult, normalized payload preview, estimated child count |
| place | CommittedOrderCommand | SubmissionResult with per-child outcomes and broker latency metadata |
| modify | committed command, broker_order_id, expected revision, desired total quantity/prices | MutationResult; later order update determines final state |
| cancel | committed command, broker_order_id, expected state | CancelRequestResult; not a fill/cancel guarantee |
| get_order | broker_order_id | BrokerOrderSnapshot |
| find_by_correlation | persisted correlation/tag, time/account scope | MatchSet with confidence and completeness |
| list_orders | session/date scope | OrderPage |
| list_executions | order/session scope | ExecutionPage |
| place_protection | entry/position reference, protection plan | ProtectionResult with native child IDs or UNSUPPORTED |
| modify_protection / cancel_protection | committed command, native leg references | Per-leg results |
| disable_trading | segment, verified-flat evidence, command | BrokerControlResult, cooldown if present |

OrderIntent includes intent_id, client_idempotency_key, broker/account, instrument/mapping version, side, product, quantity_units, order_type, limit/trigger prices, validity, market_protection, strategy_id/version nullable, opportunity_id nullable, purpose=ENTRY|REDUCE|PROTECT|HEDGE, parent_intent_id, expires_at_utc, maximum_slippage, expected_account_version and risk/compliance versions. Source can be MANUAL or AUTOMATED. Both pass the same validation. Diagnostic monotonic timestamps carry clock_domain and host_boot_id; they are never compared across reboot/hosts. A runtime deadline is reconstructed from UTC expiry only under healthy clock bounds, and expired or uncertain-age intents are never revived.

CommittedOrderCommand adds command_id, owner_epoch, durable_commit_reference, attempt_number, broker_correlation_token and payload_hash. Only the gateway can construct the broker request after checking this envelope. SubmissionResult has outcome=ACKNOWLEDGED|DEFINITIVELY_REJECTED|NOT_SENT|UNKNOWN, broker_order_ids[], per_child[], raw_code, raw_response_ref, received_at and may_have_been_sent. ACKNOWLEDGED is not FILLED.

Execution contains broker/account, broker_order_id, broker_trade_id when present, instrument/product, side, executed_units, price, exchange_time, received_time and source. Deduplication prefers broker trade identity; fallback fingerprints require collision handling and reconciliation. Order snapshots contain total, cumulative_filled, remaining, average_price, broker_status/raw_status and revision. Modification quantities are normalized to total order quantity, including already filled units.

Native multi-order is optional and non-atomic. The platform's default child-order planner provides explicit child risk reservations and status, keeping every child under freeze limits. Upstox's multi request limit and Dhan slicing behavior are enforced from [01](01-API-RESEARCH.md). Neither adapter automatically retries a write across API versions.

## 6. OrderUpdateAdapter

| Method | Request | Result / event |
|---|---|---|
| connect_updates | account, credential generation, types | UpdateSession |
| stream_updates | session | OrderObserved, ExecutionObserved, PositionObserved, HoldingObserved, StreamHealth |
| accept_webhook | ingress request metadata + body | IngressReceipt and untrusted BrokerObservation |
| reconcile_snapshot | account, buffered event watermark | ReconciliationBundle and completeness |
| close_updates | reason | Closed |

BrokerObservation carries authenticated_transport flag, provenance=STREAM|WEBHOOK|REST, raw_payload_hash, account, broker IDs, status, cumulative quantities and timestamps. Webhook ingress never creates an order intent. Unknown account/order, decreasing cumulative quantity or impossible transition triggers quarantine/authenticated lookup. A known raw status not mapped to the canonical lifecycle becomes BROKER_STATE_UNKNOWN and blocks dependent risk release.

## 7. PortfolioAdapter

| Method | Request | Result |
|---|---|---|
| fetch_positions | account | PositionSnapshot with as_of/completeness |
| fetch_holdings | account | HoldingSnapshot |
| reconcile | local account version, broker snapshots, execution watermark | ReconciliationDiff and unresolved items |
| convert_product | committed authorized conversion command | MutationResult, gated per broker |
| request_exit_all | explicit emergency plan, account, committed child commands | Per-position outcomes; never assumed atomic |

Position identity is account + broker + instrument + product. It stores net units, bought/sold units and values, average cost basis, realized/unrealized P&L, external/manual contribution, snapshot generation and reconciliation status. Holdings separate settled, unsettled/T1, pledged/collateral and sell-authorized quantities. A position snapshot is not a synthetic fill; unexplained differences create adjustment records with evidence and block new exposure until resolved.

## 8. FundsAndMarginAdapter

| Method | Request | Result |
|---|---|---|
| fetch_funds | account, segment scope | FundsSnapshot |
| estimate_margin | proposed order/portfolio, include_positions/orders flags | MarginEstimate and assumptions |
| estimate_basket | ordered legs, existing account generation | BasketMarginEstimate or UNSUPPORTED |
| validate_funds_generation | cached generation, required age | FreshnessResult |

FundsSnapshot separates available cash, collateral, used/blocked margin, receivables, withdrawable amount and available-to-trade when supplied. MarginEstimate contains initial/total/SPAN/exposure/additional components, fees if supplied, source time, input hash and validity constraints. Missing components are null, never zero. Local risk reservations are not written back as broker balances. An estimate does not guarantee acceptance or future margin sufficiency.

## Error taxonomy and retry contract

All errors contain category, broker_code, message_safe, request_id, operation, retry_after, retry_policy, may_have_been_sent, scope, raw_response_ref and capability_version.

| Category | Meaning | Permitted handling |
|---|---|---|
| AUTH_EXPIRED / REAUTH_REQUIRED | Credential invalid/expired or interactive approval needed | Pause route, renew approved flow, reconcile |
| ENTITLEMENT / SEGMENT_DISABLED | Data/product unavailable | Disable affected capability, request evidence |
| RATE_LIMIT | Budget exceeded | Honor retry hint; schedule reads; writes require outcome certainty |
| VALIDATION / UNSUPPORTED | Invalid field or capability | Correct intent before any submission |
| INSTRUMENT_UNKNOWN / AMBIGUOUS | Mapping absent/conflicting | Quarantine mapping |
| SESSION_RESTRICTED / COMPLIANCE_BLOCK | Current rule rejects action | No automatic alternative order type |
| INSUFFICIENT_FUNDS / MARGIN | Broker or local capital rejection | Release only confirmed-unused reservation |
| BROKER_REJECTED | Definitive broker rejection | Persist reason; strategy reevaluates, no automatic repeat |
| TRANSPORT_NOT_SENT | Proven no request bytes dispatched | Same durable command may be scheduled if still valid |
| SUBMISSION_UNKNOWN | Request possibly accepted | Hold reservation; lookup/stream/trade reconciliation |
| DATA_STALE / DATA_GAP | Insufficient input quality | Invalidate affected features/opportunities |
| SCHEMA_DRIFT / UNIT_UNVERIFIED | Decoder or scale unsupported | Quarantine, retain safe raw diagnostic |
| OWNERSHIP_LOST | Gateway epoch/lease invalid | Fail closed and fence before takeover |

Dhan raw DH-901..DH-910 and feed 804..814 remain alongside normalized categories; Upstox UDAPI1154/1156/1158 and other codes likewise remain visible. Never infer retry safety solely from HTTP 5xx. [Dhan errors](https://dhanhq.co/docs/v2/annexure/), [Upstox placement errors](https://upstox.com/developer/api-documentation/v3/place-order/).

## Conformance acceptance

Run the same semantic suite against each adapter independently: identity/units, interval boundaries, subscription partial success, malformed binary, token generation change, unknown enums, rate windows, order uncertainty, duplicate/reordered updates, partial fills and snapshot races. Account-dependent tests remain pending until credentials and authorization exist. A shared interface passes only when both implementations preserve semantics; matching method names alone is insufficient. Contract changes increment schema versions, supply migration rules and retain raw evidence for diagnosis.
