# Risk policy, reservations and emergency behavior

Risk is a synchronous gate and an ongoing account-state owner for every broker account (Dhan, Upstox, Zerodha). It never disappears under load. Owner-configured economic limits are mandatory (C6); a missing account policy blocks execution rather than selecting unapproved limits.

## Policy model and budget allocation

RiskPolicy is versioned per owner/account/strategy/product and includes maximum loss per position, account daily loss, allocated capital, gross/net notional, sector/underlying concentration, option Greeks/stress loss, leverage, margin utilization, minimum cash buffer, spread/impact participation, stale-data tolerance, expiry cutoff and permitted short-option exposure. Values carry currency/unit and evidence.

The global coordinator **preallocates account risk budgets**. Sum(account loss/notional/sector allocations) ≤ global limit. Each account reserves locally, keeping the hot path free of cross-account round trips. Rebalancing is serialized and durable: reduce the donor allocation, wait for its actor acknowledgement and unencumbered capacity, then increase the receiver. Cash/collateral remains broker-local; it is never transferred or netted by this accounting.

Global exposure monitoring halts all new entries when aggregate drawdown, concentration or correlation stress breaches policy. Market moves can exceed a budget despite admission checks; the response is a controlled reduction plan.

## Pre-trade gate order

| Gate | Required evidence | Failure action |
|---|---|---|
| Identity/ownership | Account identity, active gateway epoch, credential scope and class (Upstox analytics token and any read-only credential are rejected here) | Fail closed |
| Compliance/session | Certified product/order type for broker+segment+phase, effective session rules, static IP registration state, algo tag | Reject new risk; permit only certified reductions |
| Instrument | Unique mapping, active contract, tick/lot/freeze/bands, restrictions ≠ UNKNOWN | Quarantine/reject |
| Data | Required price/book/context age and quality; source epoch stable | Invalidate opportunity |
| Account state | Reconciled orders/positions/funds, no conflicting unknown command | Block affected route |
| Capital/margin | Available-to-trade estimate minus reservations and buffers; broker margin estimate within age | Reject or reduce proposed size before commit |
| Exposure | Account allocation, concentration, gross/net/Greek/stress | Reject |
| Economics | Stop distance, costs, spread, impact, reward/risk policy | Reject |
| Protection | Native or synthetic plan certified for broker/product/session ([11](11-ORDER-EXECUTION-ENGINE.md)) | Reject entry if required protection infeasible |
| Submission | Current price/session/expiry/ownership recheck immediately before write | Expire or revalidate command |

Each check returns PASS/FAIL/UNKNOWN with reason code, measured value, limit, input version and age. UNKNOWN is not PASS. Protective reductions use a distinct policy that may bypass entry-quality requirements only where explicitly safe (for example a stale sector score), never account ownership, valid instrument, quantity or broker session constraints.

## Quantity and margin

loss_per_unit = adverse stop distance × contract_multiplier + per-unit cost/slippage stress. raw_units = floor(risk_available / loss_per_unit). Final units = floor-to-lot(min(raw_units, cash/margin-affordable units, concentration headroom, liquidity-participation units, broker freeze quantity per child)). Stop distance must be positive and above a configured tick floor. Quantity zero means no trade.

Options: premium paid is a maximum loss only for long positions. Long option stress includes premium, costs and exit liquidity; short options use the execution broker's margin estimate plus adverse underlying/IV scenarios and an explicit cap. Defined-risk spreads reserve temporary unhedged-leg risk during execution. Futures use actual multiplier/units and margin from the execution broker.

MarginEstimate sources: Dhan `POST /margincalculator` (+`/multi`), Upstox `POST /v2/charges/margin`, Zerodha `POST /margins/orders` and `POST /margins/basket` (basket margin for multi-leg) via `estimate_margin`/`estimate_basket`, and `estimate_charges` for fee estimates ([04](04-BROKER-ADAPTER-CONTRACTS.md), [01](01-API-RESEARCH.md)). Input hash includes instrument, quantity, product, side, price, account positions/orders and rule version. Cache only within configured age and unchanged material state. A broker estimate is not a guarantee; reserve an owner-configured buffer.

## Reservation lifecycle

Reservation stores account, intent/child, budget dimensions, cash/margin estimate, worst-case units, generation and status. RESERVED → COMMITTED before send → WORKING while the order remains possible. Partial fills move risk from pending order to position without releasing the same risk twice. Definitive rejection/cancel releases only confirmed unused quantity. SUBMISSION_UNKNOWN retains the worst-case reservation.

Position P&L uses confirmed fills and current marked price with mark quality. Realized and unrealized P&L are separate; costs are estimated from the effective-dated cost model until actual charges reconcile (Upstox trade-charges API, Zerodha `/charges/orders` virtual contract note, Dhan contract notes are reconciliation inputs). Daily-loss policy specifies whether unrealized loss and fees count (default yes), session reset timezone and treatment of carryforward positions. A process restart does not reset the daily-loss counter.

## Stops, targets and trailing

Protection state: UNPROTECTED_PENDING, NATIVE_PENDING, PROTECTED, MODIFY_PENDING, DEGRADED, EXIT_PENDING, FLAT. Entry is not operationally complete merely because it fills; required protection must be verified.

Native protection by broker (certification gate G08): Dhan Super Order (entry+target+stop+trailing jump), Forever/OCO (documented CNC/MTF); Upstox GTT multi-rule (ENTRY/TARGET/STOPLOSS, beta `trailing_gap`); Zerodha GTT single or two-leg OCO (no trailing; GTT is a trigger that places an order, not an exchange-resident stop) and Cover Order variety where the account permits. Where none is certified for the product/session, synthetic protection runs in the account actor with a certified fallback reduction policy, and the entry requires that dependency to be acceptable.

Initial stop derives from the strategy's structure/volatility hypothesis, rounded conservatively to a valid tick and checked against current price/session rules and the broker's trigger-price rules. Target is a scenario price with costs and liquidity considered. Stop-limit orders can fail to fill through a gap; protected-market availability is broker/session gated. The UI shows this limitation.

Trailing: favorable-only movement, long proposed_stop = max(old_stop, peak_since_entry − k×ATR); short mirrors with min. k is strategy configuration (reference default 2). Modify only after at least one tick improvement and a configured debounce, preserve a minimum distance from current price, and account for the broker's modification ceiling (Dhan 25/order; Zerodha exchange modification cap per [01](01-API-RESEARCH.md); Upstox counts modifications in U-ORDER). Never loosen a stop automatically after entry.

All stop, target, time, manual and emergency exits flow through the OMS ExitController ([11](11-ORDER-EXECUTION-ENGINE.md)).

## Emergency modes

| Mode | New entries | Existing exposure | Recovery |
|---|---|---|---|
| OBSERVE_ONLY | Blocked | Monitor/reconcile | Credentials/compliance/readiness evidence |
| ENTRY_HALTED | Blocked | Protection and reductions allowed | Owner/policy release after cause resolved |
| REDUCE_ONLY | Blocked | Controlled exposure reduction | Verified risk below limits |
| EMERGENCY_FLATTEN | Blocked | Cancel entries, reconcile, submit allowed reductions | Actual fills and flat snapshot verified |
| BROKER_DISABLED | Blocked by broker | Should already be flat | Broker cooldown/reauthorization rules |

Emergency sequence (durable, idempotent, per account): set entry halt → cancel unfilled entries → determine fills/unknowns → compute reducing quantities → submit permitted exits within reserved rate capacity → reconcile → verify flat/no pending risk → optionally invoke the broker disable control.

| Broker | `bulk_cancel` | `bulk_exit` | `disable_trading` |
|---|---|---|---|
| Dhan | Per order (D-ORDER budget) | `DELETE /positions` Exit All, non-atomic | `/killswitch`: requires flat and no pending orders |
| Upstox | `DELETE /v2/order/multi/cancel` ≤50 orders, HTTP 207 | `POST /v2/order/positions/exit` ≤50 positions, market hours only, BUY first, HTTP 207 | `POST /v2/user/kill-switch` per segment: requires flat, cancels open orders, 12 h re-enable cooldown |
| Zerodha | Per order (Z-ORDER budget) | None documented (BROKER_FEATURE_ABSENT): per-position reducing orders | None documented via API (BROKER_FEATURE_ABSENT); Kite web kill switch is a manual operator step |

Bulk endpoints are convenience paths inside the same sequence: their 207 partial results are processed item by item, and a bulk acceptance is never reported as "flat". [Dhan controls](https://dhanhq.co/docs/v2/traders-control/), [Upstox kill switch](https://upstox.com/developer/api-documentation/update-kill-switch/), [Upstox exit all](https://upstox.com/developer/api-documentation/exit-all-positions/), [Upstox cancel multi](https://upstox.com/developer/api-documentation/cancel-multi-order/).

If a broker is unreachable, preserve native protection, display unresolved exposure and retry safe reads/allowed reductions under policy. If market data fails, an alternate validated source (another adapter) can support protection monitoring; otherwise halt entries and follow the predeclared native-protection/manual escalation. If all prices are stale, unbounded market orders are not an automatic fallback.

## Acceptance and runtime objectives

Property tests prove no allocation oversubscription, no double release, lot rounding, reservation conservation across fills and conservative uncertainty. Race tests cover simultaneous strategies/manual orders, donor-budget rebalance and external fills. Scenario tests cover margin spike, price gap, stale data, expiry, short-option stress, stop-modification rejection, broker outage and bulk-exit partial success on each broker.

Risk service time excluding durability targets p50 0.05 ms, p95 0.20 ms, p99 0.50 ms ([15](15-PERFORMANCE-ENGINEERING.md)). A slow check delays/rejects the intent; it never skips validation. Metrics include utilization by dimension, reservation age, unprotected units, unknown submissions, margin age, daily-loss state and emergency progress per account.
