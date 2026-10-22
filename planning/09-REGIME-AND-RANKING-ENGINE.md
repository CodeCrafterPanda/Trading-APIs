# Regime classification and opportunity ranking

The engine describes current conditions and ranks feasible setups. It is deterministic, configurable and explainable. It does not force trades, promise daily activity or treat a score as a probability of profit. Inputs come from [07](07-SECTOR-HEATMAP.md), [08](08-MARKET-SCANNER-CATALOG.md) and the quality contracts.

## Regime state vector

Represent regime as several axes rather than one mutually exclusive label: direction=BULLISH|BEARISH|SIDEWAYS|MIXED|UNKNOWN; volatility=LOW|NORMAL|HIGH|UNKNOWN; structure=BREAKOUT|BREAKDOWN|CONTINUATION|REVERSAL|RANGE|UNCERTAIN; liquidity=NORMAL|DEGRADED; coverage=READY|PARTIAL|STALE. A bullish high-volatility market is valid. Unknown mandatory inputs must not be silently converted to sideways.

Initial direction classifier uses broad-index 5/15-minute return, EMA9/21 alignment, market breadth and weighted sector participation. Normalize each to [-1,1]. D=0.35 trend +0.25 return +0.25 breadth +0.15 sector participation. Enter BULLISH at D≥0.4 for three 1-second evaluations; BEARISH at D≤−0.4. Exit directional states inside ±0.25 for three evaluations. SIDEWAYS requires abs(D)<0.2 plus low trend-efficiency and complete inputs; conflicts become MIXED.

Volatility uses current realized-volatility percentile against comparable valid live-context history: LOW below 20th, HIGH above 80th; exit at 30th/70th to reduce oscillation. Structure uses confirmed scanner events: active high-quality breaks, trend continuation or reversal. These thresholds are proposed defaults requiring live operational review, not empirical profitability claims.

RegimeSnapshot records id/version, scope, effective_at, axes, component values, data ages, confidence, coverage, configuration version and transition reason. Market-wide and sector/instrument regimes remain distinct. A stock-specific reversal can coexist with bullish market direction.

## Candidate lifecycle

~~~mermaid
stateDiagram-v2
  [*] --> OBSERVED
  OBSERVED --> QUALIFIED: evidence and data gates
  QUALIFIED --> RANKED: context and cost checks
  RANKED --> ACTIONABLE: account feasibility
  ACTIONABLE --> INTENT_CREATED: explicit policy or manual action
  OBSERVED --> INVALIDATED
  QUALIFIED --> INVALIDATED
  RANKED --> EXPIRED
  ACTIONABLE --> EXPIRED
  ACTIONABLE --> INVALIDATED
~~~

Candidate identity is scanner setup + instrument + direction + setup origin time/version. Opportunity identity groups compatible candidates into a strategy setup. New evidence revises the same opportunity rather than creating duplicate entries. Cooldowns suppress duplicate intent creation for an unchanged setup, not all future opportunities in that stock.

An opportunity records evidence IDs, feature-family contributions, entry region, stop hypothesis, target hypothesis, cost estimate, risk/reward, liquidity, source quality, selected execution account, applicable product, UTC expiry and invalidation conditions. It is executable only when an account is selected and its risk/compliance constraints pass.

## Hard gates precede scores

Reject execution eligibility when mapping/session/product is unknown, price/spread/position state is stale, required bars are incomplete, liquidity insufficient, maximum spread exceeded, tick/lot constraints fail, account budget unavailable, or estimated reward does not exceed cost-adjusted risk policy. Keep rejected observations visible with reason codes. A high momentum score cannot compensate for failed risk or data gates.

Use cost-adjusted reward = expected target distance×units − estimated round-trip costs − configured slippage allowance. Risk = stop distance×units + fees/slippage stress, adjusted for instrument multiplier and gap risk. Expected target distance is a scenario, not an expected-profit guarantee. If stop/target hypotheses lack valid structure, show analysis-only status.

## Scoring model

Each feature is transformed to [0,1] through explicit direction-aware percentile, robust z-score or bounded ratio. Store transform population/window and missingness. Initial profile weights: sector 0.15, momentum 0.15, structure 0.20, volume 0.10, liquidity 0.15, volatility suitability 0.05, cost efficiency 0.10, risk/reward 0.10. Spread is both a hard gate and a liquidity/cost component; prevent double weighting by naming its contribution.

S=100×sum(w_j f_j) over permitted components. Mandatory missing features invalidate. Optional missing features can renormalize weights only when profile allows it and maximum missing weight≤0.15; reduce confidence by retained weight. Directional features invert for bearish profiles; non-directional quality does not.

Group scanner evidence into price, structure, volume, sector, volatility and derivatives families. Cap each family's contribution and select strongest nonredundant evidence. SC01 top gainers, SC03 acceleration and SC09 trend are correlated price evidence, not three independent confirmations. Confidence Q is the minimum mandatory quality plus explicit confirmation/coverage factors, never a success probability.

## Continuous reevaluation

Maintain reverse dependency maps from instrument/sector/regime/account changes to opportunities. Recalculate only affected components, update an ordered score index at O(log M), and publish revisions at view cadence. Regime transitions can invalidate entire strategy classes through indexed membership, without recalculating every raw indicator.

Before intent creation and again before submission, revalidate quality, price region, session, risk, configuration and expiry. Durable expiry is UTC; runtime age uses monotonic time within its clock domain. Clock uncertainty beyond policy invalidates short-lived opportunities. A candle correction retracts analytical evidence but does not erase the original executed decision audit.

Strategies declare allowed regimes, scanner combinations, entry/exit hypotheses, ranking profile, permitted instruments/accounts and risk budget. Strategy enablement is a versioned control action. Automated policy can choose the highest feasible opportunity while capital/risk remains available, but no fixed trade count or obligation to trade is imposed.

## Acceptance

Hand-calculated fixtures prove component normalization, weight sum, missingness, family caps, bearish transforms and cost-adjusted feasibility. Transition tests prove hysteresis and UNKNOWN/MIXED handling. Event tests prove candidate deduplication, expiry, correction retraction and account changes invalidating actionable state. Complexity tests verify one instrument update only touches dependent candidates. Runtime decision latency follows [15](15-PERFORMANCE-ENGINEERING.md); a slow refresh publishes lag and disables affected automation.
