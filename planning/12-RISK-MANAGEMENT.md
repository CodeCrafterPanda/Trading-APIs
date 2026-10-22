# Risk policy, reservations and emergency behavior

Risk is a synchronous gate and an ongoing account-state owner. It never disappears under load. Configurable economic limits are required even though the platform imposes no arbitrary stock, opportunity or trade-count cap. Missing account policy blocks execution rather than selecting unapproved financial limits.

## Policy model and budget allocation

RiskPolicy is versioned per owner/account/strategy/product and includes maximum loss per position, account daily loss, allocated capital, gross/net notional, sector/underlying concentration, option Greeks/stress loss, leverage, margin utilization, minimum cash buffer, spread/impact participation, stale-data tolerance, expiry cutoff and permitted short-option exposure. Values are owner-selected launch inputs with currency/unit and evidence, not profitability recommendations.

The global coordinator **preallocates account risk budgets**. Sum(account loss/notional/sector allocations) cannot exceed the global limit. Each account reserves locally, keeping the hot path independent of a cross-account round trip. Rebalancing is serialized and durable: reduce the donor allocation, wait for its actor acknowledgement and unencumbered capacity, then increase the receiver. Cash/collateral remains broker-local and is never transferred or netted by this accounting operation.

Global exposure monitoring can halt all new entries when aggregate drawdown, concentration or correlation stress breaches policy. Existing market moves can exceed a budget despite admission checks; the response is a controlled reduction plan, not a claim that limits prevent all losses.

## Pre-trade gate order

| Gate | Required evidence | Failure action |
|---|---|---|
| Identity/ownership | Account identity, active gateway epoch, credential scope | Fail closed |
| Compliance/session | Certified product/order type, effective rules, static IP, session phase | Reject new risk; permit only certified reductions |
| Instrument | Unique mapping, active contract, tick/lot/freeze/bands | Quarantine/reject |
| Data | Required price/book/context age and quality | Invalidate opportunity |
| Account state | Reconciled orders/positions/funds, no conflicting unknown command | Block affected route |
| Capital/margin | Available-to-trade estimate minus reservations and buffers | Reject or reduce proposed size before commit |
| Exposure | Account allocation, concentration, gross/net/Greek/stress | Reject |
| Economics | Stop distance, costs, spread, impact, reward/risk policy | Reject |
| Protection | Native/synthetic plan certified for product/session | Reject entry if required protection infeasible |
| Submission | Current price/session/expiry/ownership recheck | Expire or revalidate command |

Each check returns PASS/FAIL/UNKNOWN, reason code, measured value, limit, input version and age. UNKNOWN is not PASS. Protective reductions use a distinct policy that can bypass entry-quality requirements only where explicitly safe (for example stale sector score), never account ownership, valid instrument, quantity or broker session constraints.

## Quantity and margin

For a proposed position, loss_per_unit = adverse stop distance×contract_multiplier + per-unit cost/slippage stress. raw_units = floor(risk_available/loss_per_unit). Final units are floor-to-lot(min(raw_units, cash/margin affordable units, concentration headroom units, liquidity participation units)). Stop distance must be positive and above a configured tick floor. Quantity zero means no trade.

For options, premium paid is not a universal maximum loss for short positions. Long option stress includes premium, costs and exit liquidity; short options use certified margin plus adverse underlying/IV scenarios and explicit risk cap. Defined-risk spreads still reserve temporary unhedged leg risk during execution. Futures use actual multiplier/units and margin estimates from the execution broker.

MarginEstimate input hash includes instrument, quantity, product, side, price, account positions/orders and rule version. Cache only within configured age and unchanged material state; refresh when quantity/product/portfolio changes. Broker estimate is not a guarantee. Reserve an owner-configured margin buffer for price/margin changes.

## Reservation lifecycle

Reservation stores account, intent/child, budget dimensions, cash/margin estimate, worst-case units, generation and status. RESERVED becomes COMMITTED before send, then WORKING while order remains possible. Partial fills move risk from pending order to position without releasing the same risk twice. Definitive rejection/cancel releases only confirmed unused quantity. SUBMISSION_UNKNOWN retains worst-case reservation.

Position P&L uses confirmed fills and current marked price with mark quality. Realized and unrealized P&L are separate; costs are estimated until actual charges reconcile. Daily-loss policy specifies whether unrealized loss and fees count (default yes), session reset timezone and treatment of carryforward positions. A process restart does not reset the daily-loss counter.

## Stops, targets and trailing

Protection state is UNPROTECTED_PENDING, NATIVE_PENDING, PROTECTED, MODIFY_PENDING, DEGRADED, EXIT_PENDING or FLAT. Entry is not considered operationally complete merely because it fills; required protection must be verified.

Initial stop derives from the strategy's structure/volatility hypothesis, rounded conservatively to valid tick and checked against current price/session rules. Target is a scenario price with costs and available liquidity considered. Stop-limit orders can fail to fill through a gap; protected-market availability is broker/session gated. The UI explicitly shows this limitation.

Trailing defaults to structure/ATR-based favorable-only movement: for long, proposed_stop=max(old_stop, peak_since_entry−k×ATR); short mirrors with min. k is strategy configuration; initial reference value 2 is a design default, not recommended financial sizing. Modify only after at least one tick improvement and a configured debounce, preserve a minimum distance from current price and account for the broker's modification ceiling. Never loosen a stop automatically after entry.

Underlying and option-premium stops, target, time exit and emergency action all use the OMS ExitController. Native OCO/trailing semantics require independent broker/product certification. Local synthetic protection depends on functioning data, process and execution connectivity; where this dependency is unacceptable, the entry must require certified native protection.

## Emergency modes

| Mode | New entries | Existing exposure | Recovery |
|---|---|---|---|
| OBSERVE_ONLY | Blocked | Monitor/reconcile | Credentials/compliance/readiness evidence |
| ENTRY_HALTED | Blocked | Protection and reductions allowed | Owner/policy release after cause resolved |
| REDUCE_ONLY | Blocked | Controlled exposure reduction | Verified risk below limits |
| EMERGENCY_FLATTEN | Blocked | Cancel entries, reconcile, submit allowed reductions | Actual fills and flat snapshot verified |
| BROKER_DISABLED | Blocked by broker | Should already be flat | Broker cooldown/reauthorization rules |

Emergency sequence is durable and idempotent: set entry halt; cancel unfilled entries; determine fills/unknowns; compute reducing quantities; submit permitted exits within reserved rate capacity; reconcile; verify flat/no pending risk; optionally invoke broker kill switch. Dhan requires flat and no pending orders. Upstox requires flat positions and has a 12-hour re-enable cooldown. [Dhan controls](https://dhanhq.co/docs/v2/traders-control/), [Upstox control](https://upstox.com/developer/api-documentation/update-kill-switch/).

If broker unreachable, preserve native protection, display unresolved exposure and retry safe reads/allowed reductions under policy. Do not claim flatten success from an HTTP acceptance. If market data fails, alternate validated price can support protection; otherwise halt entries and follow predeclared native-protection/manual escalation. If all prices are stale, unbounded market orders are not an automatic fallback.

## Acceptance and runtime objectives

Property tests prove no allocation oversubscription, no double release, lot rounding, reservation conservation across fills and conservative uncertainty. Race tests cover simultaneous strategies/manual orders, donor-budget rebalance and external fills. Scenario tests cover margin spike, price gap, stale data, expiry, short option stress, stop modification rejection and broker outage.

Risk service time excluding durability targets p50 0.10 ms, p95 0.40 ms, p99 1 ms under the declared envelope. A slow check delays/rejects the intent; it never skips validation. Metrics include utilization by dimension, reservation age, unprotected units, unknown submissions, margin age, daily-loss state and emergency progress.
