# Real-time scanner catalog

This catalog specifies **56 independent detectors**. Each emits evidence, not an order. Strategies choose enabled detectors, required confirmation and ranking profiles; the OMS/risk gates remain mandatory. Every detector is implementable from the declared inputs, is broker-agnostic (it consumes normalized observations from any of the three adapters), and does no work when its dependencies have not changed. Numeric defaults follow convention C2 in [00](00-EXECUTIVE-OVERVIEW.md).

## Shared execution contract

ScannerDefinition contains scanner_id/version, eligible asset classes, required feature mask, interval, parameters, event trigger, maximum input age, output TTL, directional interpretation, complexity budget and confidence policy. ScannerState is owned by the instrument/sector/chain worker. Stateful detectors receive closed bars unless explicitly marked quote-driven. Missing mandatory data yields UNAVAILABLE with a reason, not a negative signal.

CandidateEnvelope contains candidate_id, scanner_id/version, instrument or sector/chain scope, direction, observed_at, expires_at, input_versions, reference levels, normalized_strength [0,1], confidence [0,1], evidence_values, invalidation_rule and quality flags. Stable candidate identity groups the same setup across revisions. Update or retract a candidate when its premise changes. No detector can submit directly.

Confidence Q is evidence quality. Base Q = minimum(required field freshness scores, coverage score, warmup score, unit-verification score). A binary mandatory gate failing sets Q=0. Structure confidence multiplies Q by confirmed-touch/hold evidence; directional strength is separate. Default freshness scores decay linearly to zero at the feature's configured age limit; candle features use completion age, not tick age. Score transforms clip to [0,1] using configured threshold and saturation points. IV/Greek inputs carry `model_source=PROVIDER|LOCAL_MODEL` ([04](04-BROKER-ADAPTER-CONTRACTS.md)); LOCAL_MODEL is the only source for Zerodha-fed chains, and its unit-verification score includes the solver's residual/convergence flag, so a non-converged IV yields Q=0 for IV-dependent detectors.

Shared features: R(w)=log(P_now/P_w_ago); ATR14 uses true range of completed bars; EMA(n) updates incrementally; RVOLcum=valid cumulative volume / median same-phase elapsed volume over 20 valid prior sessions; RVOLbar=bar volume / median comparable bar volume over those sessions; spread_bps=10,000×(ask−bid)/mid. Baselines exclude unresolved corporate actions, partial sessions and invalid bars. Window lengths are configurable. Observed VWAP proxies are explicitly permitted or rejected per scanner profile.

Complexity notation: O(1) incremental arithmetic; O(log N) ordered rank update; O(L) bounded active levels; O(K) requested strikes for one chain; O(W) bounded lookback. These are per event, not full-universe passes. Default output TTL is 2 seconds for quote/liquidity events, one interval for bar setups and two refresh periods for chain snapshots; each detector below can override it.

## Price and momentum

### SC01 — Top gainers

**Purpose/data:** Identify strongest positive session returns using fresh LTP and comparable prior official close. **Algorithm/parameters:** Compute r=P/C−1, require r>0 and liquidity gate; insert into an ordered positive-return index. Profile may require minimum +0.5% return. **Trigger/dependencies/complexity:** Changed price or close; PRICE, PREV_CLOSE, LIQUIDITY; O(log N), no full sort per tick. **Output/confidence/rank:** Bullish candidate with return, cross-sectional percentile and baseline date; Q times baseline comparability, strength=return percentile. **Performance/invalidation:** Coalesce ranking publication only, preserve current state; withdraw when return/gates fail.

### SC02 — Top losers

**Purpose/data:** Surface weakest negative session returns from LTP/prior close and liquidity. **Algorithm/parameters:** r=P/C−1, require r<0, optional −0.5% threshold; maintain ascending return index. **Trigger/dependencies/complexity:** Price/close change; PRICE, PREV_CLOSE, LIQUIDITY; O(log N). **Output/confidence/rank:** Bearish candidate with negative return and lower-tail percentile; Q reflects close/price quality, rank uses loss magnitude. **Performance/invalidation:** Reuse SC01 return feature, not duplicate division; remove after recovery above threshold. A high loser rank does not establish short-sale eligibility.

### SC03 — Momentum acceleration

**Purpose/data:** Detect increasing directional speed using complete one-minute prices and volatility. **Algorithm/parameters:** a=R(1m)−R(previous 1m); require same-sign R(3m), abs(a)>0.25×ATR14/P and two consecutive qualifying closes. **Trigger/dependencies/complexity:** One-minute close; RETURN_WINDOWS, ATR; O(1). **Output/confidence/rank:** Direction=sign(a), acceleration normalized by ATR, prior/current speed and confirmation count; Q×min(count/2,1), strength from normalized acceleration. **Performance/invalidation:** Ring-buffer returns shared across detectors; expire after one bar or sign reversal. No second derivative on every raw quote.

### SC04 — Momentum continuation

**Purpose/data:** Identify a pullback resuming an established trend. **Algorithm/parameters:** EMA9 above EMA21 for bullish case, both slopes positive; pullback remains above EMA21; close reclaims EMA9 with RVOLbar≥1.2. Mirror for bearish. **Trigger/dependencies/complexity:** Bar close; EMA, VOLUME_BASELINE, STRUCTURE; O(1). **Output/confidence/rank:** Trend direction, pullback extreme, reclaim level and RVOL; Q multiplied by three-condition completion, rank by trend strength and controlled pullback depth. **Performance/invalidation:** Subscribe only trending instruments; invalidate on EMA21 break or three bars without continuation.

### SC05 — Momentum exhaustion

**Purpose/data:** Flag weakening continuation after an extended move. **Algorithm/parameters:** abs(P−EMA21)>2×ATR14, last three directional returns decelerate, and final bar fails to extend the prior extreme by one tick. Optional RVOLbar>2 with weak close location strengthens evidence. **Trigger/dependencies/complexity:** Bar close; EMA, ATR, RETURN, OHLCV; O(1). **Output/confidence/rank:** Exhaustion watch opposite the existing trend, not automatic reversal; Q×confirmed condition fraction, strength from extension/deceleration. **Performance/invalidation:** Reuse shared features; clear on renewed extreme with accelerating return or after three bars.

### SC06 — Gap up

**Purpose/data:** Detect a meaningful opening gap above comparable previous close. **Algorithm/parameters:** First valid continuous-session open O, gap=O/C−1; require gap≥max(0.5%,0.5×dailyATR/C). **Trigger/dependencies/complexity:** First complete opening bar; OFFICIAL_CLOSE, SESSION_OPEN, DAILY_ATR; O(1) once/session. **Output/confidence/rank:** Bullish gap context with gap bounds and action-adjustment basis; Q requires comparable prior close, rank by normalized gap and liquidity. **Performance/invalidation:** Store one session event; subsequent quotes update fill status through SC08. Pre-open indicative price is not the continuous open.

### SC07 — Gap down

**Purpose/data:** Detect a negative opening discontinuity. **Algorithm/parameters:** gap=O/C−1≤−max(0.5%,0.5×dailyATR/C), using first continuous bar and comparable close. **Trigger/dependencies/complexity:** Session opening bar; same dependencies as SC06; O(1). **Output/confidence/rank:** Bearish gap bounds, ATR-normalized magnitude, source basis; Q rejects unresolved corporate actions, rank uses negative magnitude and executable liquidity. **Performance/invalidation:** One session setup reused by gap-fill and reversal detectors; do not repeatedly emit on every tick. A gap across a split is unavailable, not a signal.

### SC08 — Gap fill

**Purpose/data:** Track closure of a valid SC06/07 gap. **Algorithm/parameters:** progress=(O−P)/(O−C), direction-adjusted; emit approach at ≥0.8 and completed fill when a closed bar touches/crosses C within one tick. **Trigger/dependencies/complexity:** Price for progress, bar close for confirmation; GAP_CONTEXT, PRICE/OHLC; O(1). **Output/confidence/rank:** Gap direction, fill fraction, close level and confirmation state; Q inherited from gap, strength from progress plus hold. **Performance/invalidation:** One active gap state/session; invalidate at session end or basis change. A wick touch and confirmed close are distinct outputs.

### SC09 — Intraday uptrend

**Purpose/data:** Recognize sustained bullish structure. **Algorithm/parameters:** Last two confirmed swing highs and lows both rise, EMA9>EMA21 and positive 5-minute return; minimum two pivot pairs. **Trigger/dependencies/complexity:** Confirmed pivot/bar close; SC21, EMA, RETURNS; O(1). **Output/confidence/rank:** Bullish trend with pivot IDs and slope; Q×pivot confirmation coverage, strength combines normalized slope and breadth of alignment. **Performance/invalidation:** Pivot events drive updates; invalidate on a confirmed lower low or EMA/structure conflict. Confirmation delay is visible.

### SC10 — Intraday downtrend

**Purpose/data:** Recognize bearish swing structure. **Algorithm/parameters:** Two lower highs and lower lows, EMA9<EMA21 and negative 5-minute return. **Trigger/dependencies/complexity:** Confirmed pivot/bar close; SC21, EMA, RETURNS; O(1). **Output/confidence/rank:** Bearish trend with lower-high invalidation level; Q×confirmed-pivot coverage, strength uses negative slope and alignment. **Performance/invalidation:** Share SC09 state machinery with direction parameter but separate scanner ID; invalidate at confirmed higher high or conflicting structure. Product short permissions are checked later.

### SC11 — Bullish reversal

**Purpose/data:** Identify a bearish trend changing direction. **Algorithm/parameters:** Prior SC10 active, downside sweep or failed breakdown, close above last confirmed lower high, and RVOLbar≥1.3. **Trigger/dependencies/complexity:** Closed bar/structure event; SC10, SC21, SC26/30, VOLUME; O(1). **Output/confidence/rank:** Bullish reversal, broken level, low invalidation and evidence IDs; Q×number of confirmed independent conditions, strength from break distance/ATR. **Performance/invalidation:** Evaluate only prior bearish contexts; withdraw if price closes below reversal low or volume basis fails.

### SC12 — Bearish reversal

**Purpose/data:** Identify an uptrend losing structure. **Algorithm/parameters:** Prior SC09 active, upside sweep or false breakout, close below last confirmed higher low, RVOLbar≥1.3. **Trigger/dependencies/complexity:** Closed bar/structure event; trend, pivots, SC26/30, VOLUME; O(1). **Output/confidence/rank:** Bearish reversal with upper invalidation; Q times independent evidence completion, rank by confirmed break/ATR and liquidity. **Performance/invalidation:** Evaluate only prior bullish contexts; remove on close above reversal high. No future pivot is used before its confirmation time.

## Volume and liquidity

### SC13 — Unusual cumulative volume

**Purpose/data:** Find session participation materially above historical context. **Algorithm/parameters:** Cumulative volume exceeds the 95th percentile of same-elapsed-session volumes over 20 valid sessions, with at least ten valid baselines. **Trigger/dependencies/complexity:** Cumulative-volume change, published at 1 Hz; VOLUME, SESSION_BASELINE; O(log W) baseline lookup/O(1) comparison. **Output/confidence/rank:** Nondirectional volume anomaly plus price direction context; Q×baseline coverage, strength=exceedance ratio. **Performance/invalidation:** Precompute time-bucket baseline arrays; unavailable across unresolved counter gaps.

### SC14 — Relative-volume leadership

**Purpose/data:** Rank participation intensity comparably across stocks. **Algorithm/parameters:** RVOLcum≥1.5, denominator above minimum liquid baseline; rank within asset/sector cohorts. **Trigger/dependencies/complexity:** Valid volume update or baseline bucket change; VOLUME_BASELINE; O(log N). **Output/confidence/rank:** RVOL, baseline sample count, elapsed phase and price return; Q weighted by baseline quality, strength=cohort percentile. **Performance/invalidation:** Share baseline with SC13; expire when source epoch changes or volume becomes partial. Thin stocks cannot win merely from a near-zero denominator.

### SC15 — Bar-volume spike

**Purpose/data:** Detect sudden local participation bursts. **Algorithm/parameters:** Completed bar volume / median previous 20 valid bars≥3 and absolute turnover floor passes. **Trigger/dependencies/complexity:** Bar close; OHLCV, rolling median, turnover; O(log W). **Output/confidence/rank:** Spike ratio, close-location value and direction context; Q requires complete bar volume, strength from ratio with saturation at 6. **Performance/invalidation:** Maintain rolling order-statistic structure, not sort history each bar; expire after two bars unless renewed.

### SC16 — Turnover acceleration

**Purpose/data:** Detect increased monetary trading activity. **Algorithm/parameters:** Certified traded-value delta per minute exceeds twice the trailing 10-minute median and accelerates for two bars. If only proxy exists, profile must explicitly allow PROXY. **Trigger/dependencies/complexity:** Bar close; TURNOVER or labeled price-volume proxy; O(log W). **Output/confidence/rank:** Monetary rate, acceleration and basis; Q penalizes proxy only by configured policy, strength from ratio. **Performance/invalidation:** Shared rolling aggregates; unavailable when turnover counter resets or crosses a data gap.

### SC17 — Spread anomaly

**Purpose/data:** Detect abnormal execution cost or dislocation. **Algorithm/parameters:** Fresh bid/ask spread exceeds max(configured bps ceiling, median spread+3 MAD), or abruptly narrows below cohort threshold. **Trigger/dependencies/complexity:** Best-book change, debounce 100 ms; DEPTH1, rolling spread baseline; O(log W). **Output/confidence/rank:** WIDENING/NARROWING, spread bps and age; Q requires both sides fresh, severity uses robust deviation. **Performance/invalidation:** Bounded sampled baseline; stale/crossed books emit quality alert rather than favorable narrowing.

### SC18 — Liquidity change

**Purpose/data:** Identify deterioration/improvement in near-touch depth. **Algorithm/parameters:** Sum bid/ask notional within configured 10 bps; compare with five-minute median and flag <0.5× or >2×, with spread corroboration. **Trigger/dependencies/complexity:** Depth snapshot, max 5 Hz; DEPTH, PRICE; O(D) for supplied depth levels. **Output/confidence/rank:** Side-specific depth change and imbalance, Q from depth age/coverage, severity from ratio. **Performance/invalidation:** Bound D by subscribed mode; no claim that displayed depth will execute or that hidden liquidity is known.

## Technical structure

### SC19 — Support proximity and hold

**Purpose/data:** Find price near established downside levels. **Algorithm/parameters:** Cluster confirmed pivot lows within 0.2 ATR; require ≥2 touches separated by ≥3 bars; proximity ≤0.25 ATR and bullish close away for HOLD. **Trigger/dependencies/complexity:** Pivot/bar close; SC21, ATR, OHLC; O(L), active levels default 64 per instrument. **Output/confidence/rank:** Level band, touch count, age and hold state; Q×min(touches/3,1), rank by recency/response. **Performance/invalidation:** Evict obsolete levels by age/relevance; confirmed close below band invalidates.

### SC20 — Resistance proximity and rejection

**Purpose/data:** Find established overhead supply levels. **Algorithm/parameters:** Cluster confirmed highs within 0.2 ATR, ≥2 separated touches; proximity ≤0.25 ATR, bearish close away confirms rejection. **Trigger/dependencies/complexity:** Pivot/bar close; SC21, ATR; O(L). **Output/confidence/rank:** Resistance band, touch evidence, rejection distance; Q×touch confirmation, strength normalized by ATR. **Performance/invalidation:** Same bounded level index as support; close above band invalidates resistance and can seed a retest candidate.

### SC21 — Swing structure

**Purpose/data:** Produce deterministic confirmed pivots. **Algorithm/parameters:** A high exceeds/equal-tie-resolves two bars left and two right; lows mirror. Ties select earliest bar. Confirm only after both right bars close; classify HH/HL/LH/LL against prior confirmed pivot. **Trigger/dependencies/complexity:** Bar close; OHLC window; O(1) fixed-width deque. **Output/confidence/rank:** Pivot ID, pivot_time, confirmation_time, class and prominence/ATR; Q reflects complete bars. **Performance/invalidation:** Never repaint confirmed pivots except explicit candle correction revision, which retracts dependent setups.

### SC22 — Consolidation box

**Purpose/data:** Detect bounded low-range structure. **Algorithm/parameters:** Last 12 bars range≤2 ATR, absolute net move≤0.5 ATR, ≥2 touches near each boundary within 0.2 ATR. **Trigger/dependencies/complexity:** Bar close; rolling extrema, ATR, touch counters; O(1) amortized. **Output/confidence/rank:** Box high/low, duration, compression and touch evidence; Q×boundary confirmation, strength favors tighter valid boxes. **Performance/invalidation:** Monotonic deques avoid rescanning; breakout or missing bar invalidates/ends the box.

### SC23 — Breakout

**Purpose/data:** Detect upward escape from resistance/box. **Algorithm/parameters:** Closed price exceeds prior confirmed level by max(one tick,0.1 ATR), RVOLbar≥1.5, close in upper 25% of bar. **Trigger/dependencies/complexity:** Bar close; SC20/22, ATR, VOLUME; O(L). **Output/confidence/rank:** Bullish level break, distance, volume and invalidation band; Q×confirmed conditions, rank from distance/volume/level quality. **Performance/invalidation:** Evaluate nearest relevant levels only; retract on close back inside within configured three bars.

### SC24 — Breakdown

**Purpose/data:** Detect downward escape from support/box. **Algorithm/parameters:** Close below level by max(tick,0.1 ATR), RVOLbar≥1.5, close in lower 25%. **Trigger/dependencies/complexity:** Bar close; SC19/22, ATR, VOLUME; O(L). **Output/confidence/rank:** Bearish broken level, volume and stop reference; Q×condition completion, strength normalized by ATR/volume. **Performance/invalidation:** Shared level index; close back above within three bars transitions to false-break evidence.

### SC25 — Retest

**Purpose/data:** Confirm a prior breakout/breakdown level holds after return. **Algorithm/parameters:** Within 10 bars of SC23/24, price touches ±0.2 ATR band, then closes on breakout side without exceeding invalidation band. **Trigger/dependencies/complexity:** Bar close; active break setup, ATR/OHLC; O(A) bounded active setups. **Output/confidence/rank:** Original setup ID, touch/hold bars and direction; Q×original level confidence, rank rewards shallow valid retest and participation. **Performance/invalidation:** Expire after 10 bars or opposite close; do not continuously search all past breaks.

### SC26 — False breakout or breakdown

**Purpose/data:** Identify failed level escapes. **Algorithm/parameters:** After SC23/24 or an intrabar breach, close returns inside original range within three bars and next close fails to reclaim breakout side. **Trigger/dependencies/complexity:** Bar close; active levels/breaks, OHLC; O(A). **Output/confidence/rank:** Opposite-direction watch, breached level and failure duration; Q×two-close confirmation, strength from rejection/ATR. **Performance/invalidation:** Separate intrabar watch from confirmed failure; clear if original breakout resumes.

### SC27 — Order-block heuristic

**Purpose/data:** Mark a reproducible displacement-origin zone; it is not proof of institutional orders. **Algorithm/parameters:** A completed move of ≥1.5 ATR across at most three bars breaks a confirmed swing; select the last opposite-colored closed candle immediately before displacement, zone=its body-to-wick range. **Trigger/dependencies/complexity:** Confirmed displacement close; OHLC, ATR, SC21; O(3)+O(L). **Output/confidence/rank:** Zone, direction, origin/confirmation times, displacement evidence; Q×structure confirmation. **Performance/invalidation:** Keep bounded active zones; invalidate on close through far edge or expiry after 60 bars.

### SC28 — Supply zone

**Purpose/data:** Locate repeated bearish rejection regions. **Algorithm/parameters:** Base of 2–5 narrow bars (<0.7 ATR each) followed by downside displacement≥1.5 ATR; zone spans base highs/body lows, then track retests. **Trigger/dependencies/complexity:** Bar close; OHLC, ATR, pivots; O(5)+O(L). **Output/confidence/rank:** Supply band, freshness, retests and displacement; Q×confirmation, rank decreases after repeated penetrations. **Performance/invalidation:** Maximum active zones is a memory/work parameter; close above far edge invalidates. Label heuristic, not observed hidden orders.

### SC29 — Demand zone

**Purpose/data:** Locate bullish displacement origins. **Algorithm/parameters:** Base of 2–5 narrow bars followed by upside move≥1.5 ATR; zone spans base lows/body highs and records subsequent holds. **Trigger/dependencies/complexity:** Bar close; OHLC, ATR, pivots; O(5)+O(L). **Output/confidence/rank:** Demand band, displacement, age and touch count; Q×confirmed evidence, strength balances displacement and unbroken freshness. **Performance/invalidation:** Reuse zone engine; close below far edge or configured age invalidates. It does not establish actual resting demand.

### SC30 — Liquidity sweep heuristic

**Purpose/data:** Detect a brief excursion beyond a confirmed extreme followed by rejection. **Algorithm/parameters:** Bar exceeds prior high/low by ≥one tick and ≤0.5 ATR, then closes back inside; next bar holds inside for confirmation. **Trigger/dependencies/complexity:** Bar close; confirmed levels, ATR/OHLC; O(L). **Output/confidence/rank:** Swept level, excursion and reversal direction; Q×two-bar confirmation, rank by rejection proportion. **Performance/invalidation:** No claim that stops were actually executed; deep continuation beyond level invalidates.

### SC31 — Multi-timeframe alignment

**Purpose/data:** Require consistent context across time horizons. **Algorithm/parameters:** 1m and 5m directional EMA/structure states agree with 15m trend, using only completed bars; all three must be ready. **Trigger/dependencies/complexity:** Any relevant bar/structure change; SC09/10 plus interval features; O(number of configured intervals). **Output/confidence/rank:** Direction, interval votes and ages; confidence=min interval Q, strength weighted alignment. **Performance/invalidation:** Cached interval states, no historical rescans; one conflict moves to MIXED rather than silently dropping that interval.

## Sector intelligence

### SC32 — Strong stock in strong sector

**Purpose/data:** Find aligned bullish leadership. **Algorithm/parameters:** Sector bullish score≥70/100, stock sector-relative return percentile≥80, positive momentum and liquidity gates. **Trigger/dependencies/complexity:** Stock/sector score change; sector engine, returns, liquidity; O(1) plus rank index. **Output/confidence/rank:** Bullish alignment with component versions; confidence=min stock/sector coverage Q, strength combines normalized sector score and stock RS. **Performance/invalidation:** Reevaluate only affected members when sector state changes; expire on sector downgrade or stale benchmark.

### SC33 — Weak stock in weak sector

**Purpose/data:** Find aligned bearish laggards. **Algorithm/parameters:** Bearish sector score≥70, stock RS in bottom 20%, negative momentum and liquidity gates. **Trigger/dependencies/complexity:** Stock/sector changes; same dependencies as SC32; O(1). **Output/confidence/rank:** Bearish candidate with separate stock/sector contributions; Q=min qualities, strength from lower-tail RS and bearish breadth. **Performance/invalidation:** No automatic short eligibility assumption; withdraw when alignment or sector coverage fails.

### SC34 — Sector divergence

**Purpose/data:** Surface stocks moving against their sector. **Algorithm/parameters:** Opposite signs of stock and sector 5m returns, abs(stock return−sector return)>0.5 ATR/P, held two bars. **Trigger/dependencies/complexity:** 1m close or sector revision; stock/sector returns, ATR; O(1). **Output/confidence/rank:** POSITIVE/NEGATIVE_DIVERGENCE and relative move; Q×persistence, rank by normalized separation. **Performance/invalidation:** Compare matched timestamps/bases; clear on sign convergence. Divergence is context, not automatically reversal.

### SC35 — Sector rotation

**Purpose/data:** Detect changes in sector leadership. **Algorithm/parameters:** Sector RS/momentum moves from lagging to improving/leading quadrant or reverse; rank changes by configured percentile distance≥20 over 15m and holds 2m. **Trigger/dependencies/complexity:** Sector snapshot, 1 Hz; sector ranks/RS; O(log S). **Output/confidence/rank:** Transition, previous/current quadrant and dwell; Q from breadth coverage, strength by rank/RS change. **Performance/invalidation:** Sector-level only; no tick-wide constituent recalculation and no claim of actual fund flows.

### SC36 — Emerging leader

**Purpose/data:** Find stocks climbing within-sector ranks before becoming top-ranked. **Algorithm/parameters:** RS percentile increases≥25 points over 10m, RVOL≥1.3, positive 3m momentum; initial percentile below 80. **Trigger/dependencies/complexity:** Rank/bar change; sector stock ranks, RVOL/returns; O(1). **Output/confidence/rank:** Bullish rank trajectory and evidence; Q×persistence, strength from rank velocity and volume. **Performance/invalidation:** Store rolling rank checkpoints, not all tick ranks; invalidate on momentum reversal or coverage discontinuity.

### SC37 — Emerging laggard

**Purpose/data:** Find deteriorating within-sector positions. **Algorithm/parameters:** RS percentile falls≥25 points over 10m, RVOL≥1.3, negative 3m momentum; initial percentile above 20. **Trigger/dependencies/complexity:** Rank/bar event; same features as SC36; O(1). **Output/confidence/rank:** Bearish rank trajectory, relative return and volume; Q×persistence, strength from deterioration. **Performance/invalidation:** Shared rank checkpoint ring; clear after recovery above prior threshold or sector membership revision.

## Volatility

### SC38 — Volatility expansion

**Purpose/data:** Detect widening realized movement. **Algorithm/parameters:** 5m realized variance / trailing 60m median comparable variance≥1.8 and current true range≥1.3 ATR. **Trigger/dependencies/complexity:** Bar close; RETURNS, ATR, variance baseline; O(1) plus rolling quantile update. **Output/confidence/rank:** Nondirectional expansion plus price direction, Q from complete returns, strength by variance ratio. **Performance/invalidation:** Reuse rolling sums; no annualized interpretation unless explicitly configured; expire when ratio drops below hysteresis 1.4.

### SC39 — Volatility compression

**Purpose/data:** Identify unusually quiet conditions. **Algorithm/parameters:** Rolling 10-bar range/ATR and variance percentile both below configured 20th percentile, while liquidity remains sufficient. **Trigger/dependencies/complexity:** Bar close; extrema, ATR, variance baseline, liquidity; O(1)/O(log W). **Output/confidence/rank:** Compression watch, box/range and duration; Q requires adequate samples, strength increases with sustained compression. **Performance/invalidation:** Thin illiquid inactivity is excluded; expansion ends setup. No direction is inferred until separate break evidence.

### SC40 — Abnormal movement

**Purpose/data:** Flag moves inconsistent with recent volatility. **Algorithm/parameters:** abs(1m return−rolling median)>4×robust sigma, with absolute tick/return floor. **Trigger/dependencies/complexity:** Bar close; RETURN, median/MAD, quality; O(log W). **Output/confidence/rank:** Directional anomaly severity plus data-quality classification; Q falls for price jumps unsupported by bid/ask, strength from robust z-score. **Performance/invalidation:** Distinguish potential bad tick from genuine move; require corroboration before execution; expire after two bars.

### SC41 — Volatility regime transition

**Purpose/data:** Publish low/normal/high volatility state changes. **Algorithm/parameters:** Realized-vol percentile enters >80 or <20 bands for three closes; exit thresholds 70/30 provide hysteresis. **Trigger/dependencies/complexity:** Bar close; variance/cohort baseline; O(1). **Output/confidence/rank:** Previous/new state, dwell and percentile; Q includes baseline size, confidence rises with dwell. **Performance/invalidation:** Reuse regime engine state; gaps transition to UNKNOWN rather than keeping a stale state indefinitely.

## Derivatives

### SC42 — Open-interest change

**Purpose/data:** Detect unusual participation changes in a contract. **Algorithm/parameters:** deltaOI=current−comparable baseline, ratio=deltaOI/max(baseline,minimumOI); flag abs(ratio)≥5% over 5m and minimum absolute lot change. **Trigger/dependencies/complexity:** OI observation, not every price tick; OI, contract units; O(1). **Output/confidence/rank:** OI increase/decrease with paired price return, timestamps and baseline; Q from OI age/units, strength from normalized change. **Performance/invalidation:** Counter/contract/source change resets baseline; no interpretation of trader intent from OI alone.

### SC43 — Put-call ratio shift

**Purpose/data:** Track chain-wide positioning balance. **Algorithm/parameters:** PCR=sum put OI/sum call OI over specified expiry and moneyness range; flag change≥0.2 over three comparable snapshots. Denominator and coverage gates apply. **Trigger/dependencies/complexity:** Chain refresh; CHAIN_OI, expiry/strike membership; O(K), incremental if deltas available. **Output/confidence/rank:** PCR level/change, strike universe and coverage; Q=min chain coverage/age, strength by change magnitude. **Performance/invalidation:** Never compare changing strike universes without recomputation; no fixed bullish/bearish conclusion.

### SC44 — Implied-volatility anomaly

**Purpose/data:** Detect unusual option IV relative to comparable contracts. **Algorithm/parameters:** Unit-certified IV deviates >3 MAD from same-expiry/moneyness cohort or rises >20% relative over 5m; exclude invalid/crossed quotes. **Trigger/dependencies/complexity:** Chain/Greeks update; IV units, underlying price, expiry; O(K) cohort refresh. **Output/confidence/rank:** IV anomaly with cohort/model source; Q includes quote and convention quality, strength robust deviation. **Performance/invalidation:** Cache cohort statistics per chain; unavailable when IV units or stale underlying are unresolved.

### SC45 — Greek exposure alert

**Purpose/data:** Identify contracts/positions with high delta, gamma, theta or vega sensitivity. **Algorithm/parameters:** Compute unit-certified Greek×units, aggregate by underlying/expiry; compare configured risk thresholds and stress price/IV steps. **Trigger/dependencies/complexity:** Greeks, position or underlying change; GREEKS_UNITS, POSITION; O(changed positions). **Output/confidence/rank:** Exposure vector and breached threshold, Q=min Greek/position quality; severity=utilization. **Performance/invalidation:** No aggregation across unknown conventions; feeds risk alerts, not a directional trade guarantee.

### SC46 — Strike liquidity

**Purpose/data:** Rank executable option contracts. **Algorithm/parameters:** Hard gates on fresh two-sided quote, spread bps/premium, minimum traded value/OI and lot affordability; score depth, turnover, spread and quote age. **Trigger/dependencies/complexity:** Selected-strike book or chain update; DEPTH1, OI, VOLUME, contract metadata; O(log K) rank update. **Output/confidence/rank:** Strike ranking with each gate and component; Q from field ages, strength=liquidity score. **Performance/invalidation:** Stream shortlisted strikes within quota; a low premium alone never implies liquidity.

### SC47 — Futures long-buildup classification

**Purpose/data:** Label simultaneous positive price and OI change descriptively. **Algorithm/parameters:** 5m return>0.1 ATR/P and OI change>2%, both comparable and above absolute OI floor. **Trigger/dependencies/complexity:** OI observation; FUTURE_PRICE, OI, ATR; O(1). **Output/confidence/rank:** LONG_BUILDUP_LABEL with both deltas/times; Q=min paired-input quality, strength=min normalized price/OI moves. **Performance/invalidation:** This conventional label does not identify actual buyer intent; reset on source/contract change.

### SC48 — Futures short-buildup classification

**Purpose/data:** Label falling price with rising OI. **Algorithm/parameters:** Return<−0.1 ATR/P and OI change>2% over comparable 5m window. **Trigger/dependencies/complexity:** OI update; FUTURE_PRICE, OI, ATR; O(1). **Output/confidence/rank:** SHORT_BUILDUP_LABEL, paired evidence; Q and strength as SC47. **Performance/invalidation:** Reuse paired window state; prohibit conclusions from an OI timestamp much older than price; no automatic bearish order.

### SC49 — Futures short-covering classification

**Purpose/data:** Label rising price with falling OI. **Algorithm/parameters:** Return>0.1 ATR/P and OI change<−2%, absolute floor and synchronized window. **Trigger/dependencies/complexity:** OI event; price/OI/ATR; O(1). **Output/confidence/rank:** SHORT_COVERING_LABEL with deltas and confidence Q; strength from smaller normalized component. **Performance/invalidation:** Labels are conventional interpretations only; expire after one OI refresh interval if data stops and reset across rollover.

### SC50 — Futures long-unwinding classification

**Purpose/data:** Label falling price with falling OI. **Algorithm/parameters:** Return<−0.1 ATR/P and OI change<−2%, comparable 5m window. **Trigger/dependencies/complexity:** OI update; price/OI/ATR; O(1). **Output/confidence/rank:** LONG_UNWINDING_LABEL and evidence; Q from both fields, strength normalized jointly. **Performance/invalidation:** Same shared state as SC47–49; mutually exclusive quadrant labels for one window, UNKNOWN when either delta is below threshold.

### SC51 — Unusual options activity

**Purpose/data:** Find abnormal contract participation without inferring buyer/seller initiation. **Algorithm/parameters:** Valid bar-volume ratio≥3, premium turnover floor, OI change magnitude and spread gate; aggregate related strikes by expiry. **Trigger/dependencies/complexity:** Contract bar/OI or chain refresh; option OHLCV/OI/liquidity; O(1) per contract, O(K) aggregate. **Output/confidence/rank:** Activity cluster, call/put composition and evidence; Q from coverage, strength volume/turnover anomaly. **Performance/invalidation:** Repeated quote last quantity is not new volume; no “smart money” claim.

### SC52 — IV skew change

**Purpose/data:** Detect relative repricing across option strikes. **Algorithm/parameters:** Compare unit-certified IV at matched absolute delta buckets, e.g. 25-delta put minus call; flag change beyond configured robust deviation. **Trigger/dependencies/complexity:** Chain refresh; IV, delta, expiry, valid quotes; O(K) bucket selection. **Output/confidence/rank:** Skew value/change with selected contracts and interpolation policy; Q=min pair quality, strength deviation. **Performance/invalidation:** No interpolation across missing wide strike gaps; expiry and convention must match.

### SC53 — Futures basis anomaly

**Purpose/data:** Detect unusual futures-versus-underlying divergence. **Algorithm/parameters:** basis=(F−S)/S, compare same-contract rolling median/MAD; flag >3 MAD after synchronized quote-age and spread gates. **Trigger/dependencies/complexity:** Future or underlying quote, 2 Hz maximum; both prices, expiry; O(1). **Output/confidence/rank:** Basis deviation and days to expiry; Q from paired freshness, strength robust deviation. **Performance/invalidation:** Not a guaranteed arbitrage; funding/dividends/settlement/costs are separate context; expired contract invalidates.

### SC54 — Option premium acceleration

**Purpose/data:** Detect rapidly changing option premium with underlying confirmation. **Algorithm/parameters:** Premium 1m return accelerates above 0.5 premium-ATR/P, spread gate passes, and underlying move/IV change explains context. **Trigger/dependencies/complexity:** Option bar close; premium returns/ATR, underlying, IV optional; O(1). **Output/confidence/rank:** Premium direction, underlying/IV contributions and missingness; Q penalizes stale pair, strength normalized acceleration. **Performance/invalidation:** Exclude tiny-premium denominator artifacts and stale quotes; no assumption premium movement equals underlying direction.

### SC55 — IV contraction

**Purpose/data:** Flag falling implied volatility affecting option premium risk. **Algorithm/parameters:** IV decreases≥15% relative over 5m across ≥60% valid neighboring strikes, with matched expiry and unchanged units. **Trigger/dependencies/complexity:** Chain refresh; IV coverage, strike cohort; O(K). **Output/confidence/rank:** IV contraction, breadth and premium sensitivity; Q from cohort coverage, strength magnitude×breadth. **Performance/invalidation:** No unsupported event-cause attribution; incomplete chain or expiry/source transition invalidates comparison.

### SC56 — OI concentration

**Purpose/data:** Highlight strike clusters with concentrated outstanding interest. **Algorithm/parameters:** Compute each strike's OI share and local three-strike sum; flag top concentration percentile and changes≥5 percentage points. **Trigger/dependencies/complexity:** Chain OI snapshot; OI/strike/expiry; O(K). **Output/confidence/rank:** Concentration bands, calls/puts separately, universe coverage; Q reflects completeness, strength concentration/change. **Performance/invalidation:** Concentration is not proven support/resistance or pinning; no comparison across changing chain coverage without rebasing.

## Dependency scheduling and performance controls

Create one feature DAG per instrument/interval plus shared sector/chain DAGs. Topologically evaluate only dirty nodes; retain feature refcounts so unused indicators are not computed. Quote-driven outputs can update at their configured cadence while risk monitoring remains event-driven. Expensive O(K) chain calculations run once per snapshot and fan out cached results. Bound active levels/setups/windows by configuration and memory budget; these bounds do not restrict the number of eligible stocks or opportunities.

Deduplicate correlated evidence: SC01/03/09 all derive partly from price momentum, so ranking must not count them as independent confirmations. [09](09-REGIME-AND-RANKING-ENGINE.md) defines feature-family caps. Scanner version changes invalidate old state and warm new dependencies before execution eligibility.

Acceptance for each detector includes a minimal positive fixture, negative boundary, missing-data fixture, source-epoch reset, candle-correction/retraction and parameter-boundary test. Technical structure tests verify confirmation times prevent future-data leakage. Derivative tests verify units, zero denominators, missing strikes, stale paired inputs and PROVIDER-vs-LOCAL_MODEL Greek agreement on fixtures. Runtime acceptance records per-detector calls/event, service time, state bytes and queue lag against the per-detector budget in [15](15-PERFORMANCE-ENGINEERING.md) (p50 0.01 ms, p95 0.05 ms, p99 0.10 ms per event); a detector exceeding its budget is degraded visibly and cannot delay order-risk processing.
