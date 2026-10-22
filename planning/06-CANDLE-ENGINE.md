# Historical and live candle engine

The engine maintains separate broker-history, locally observed and locally aggregated series. A broker snapshot feed is not a documented exhaustive trade tape. Therefore local OHLC/volume can differ from official candles; provenance and quality are part of every bar, not optional diagnostics.

## Series identity and intervals

SeriesKey = instrument_id + interval + session_calendar_version + phase_policy + origin + provider + adjustment_basis. Required platform intervals are 1,2,3,5,10,15,30,60 minutes and daily; additional provider-native intervals remain available. Dhan also provides 25-minute bars; Upstox supports broader minute/hour/week/month units. The exact native interval/window manifest is [01](01-API-RESEARCH.md).

Use one-minute bars as the canonical local aggregation base. Two/three/ten/thirty-minute bars are not mislabeled native Dhan results. If a native broker interval and a local aggregate both exist, store both and expose the selected basis. Do not silently switch a chart or scanner between sources.

## Historical retrieval scheduler

Historical requests are keyed by series, requested range and normalization version. Maintain a coverage manifest of known complete, provisional, missing and unverified intervals. At startup fetch only missing warmup ranges plus a configurable overlap around the latest boundary. Cache completed immutable chunks; retain raw payload hashes for diagnosis.

Chunk Dhan intraday requests to at most 90 days, preserving its supported intervals and endpoint boundary conventions. Chunk Upstox by calendar month/quarter/decade rules rather than guessed bar count. Daily Dhan toDate is exclusive; normalize other endpoint boundaries only after conformance tests. [Dhan history](https://dhanhq.co/docs/v2/historical-data/), [Upstox V3 history](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data/).

Requests are prioritized: open-position context, selected opportunities, active scanner warmup, then background universe preparation. Every fetch consumes the broker's rate scheduler. Concurrent identical requests coalesce into one fetch. Read retries honor budget and backoff; a provider returning no rows cannot automatically mark the range complete.

Validate parallel-array lengths/tuple shapes, timestamp order, OHLC inequalities, nonnegative volume/OI, session membership and duplicate keys. Reject invalid rows into a quarantine artifact; do not “repair” high/low values without evidence.

## Session alignment and phase boundaries

Persist UTC timestamps and use Asia/Kolkata for exchange-session rules. For a continuous phase beginning at s, interval width w gives bar_start = s + floor((event_time-s)/w)×w, bounded by the phase end. The last bar may be shorter. Never assume midnight alignment, a universal 375-minute session or one close time for all instruments.

Pre-open, continuous, CAS and post-close observations are distinct phases. Default intraday scanners consume continuous-session bars only. Auction/final-close data is stored separately and used for official daily reconciliation. The effective-dated current/future session rules in [16](16-SECURITY-AND-COMPLIANCE.md) govern actual boundaries. An authoritative random-close event ends the phase; the system does not predict the random instant.

## Incremental local bar construction

For each active one-minute bar, retain first/last valid observed price, extrema, cumulative-volume baseline/end, valid turnover baseline/end, observation count, source epoch and quality flags. OHLC uses valid price observations within the bar. If no verified exchange timestamp exists, use receipt time with RECEIPT_TIME basis, exposing the lower precision.

Volume is exact only relative to a certified cumulative counter and uninterrupted comparable boundaries. At initial subscription halfway through a bar, volume before the first observation is unknown. A jump spanning multiple bars cannot be allocated accurately among them. Mark affected bar volume UNKNOWN/PARTIAL; do not put the entire gap into the reconnect minute or distribute it evenly. A negative counter delta implies reset/correction/source change and invalidates the baseline.

True traded-value VWAP can use a certified cumulative turnover divided by cumulative volume, or certified broker session ATP. If turnover is absent, sum(observed price × cumulative-volume delta) is an approximation because the delta may contain trades at other prices. Name it OBSERVED_VWAP_PROXY; no scanner requiring exact VWAP may silently consume it.

## Completion, lateness and revisions

At scheduled bar end, emit PROVISIONAL_CLOSE. A proposed 2-second allowed-lateness window handles ordinary delayed observations; this is configurable and measured, not a broker promise. Late observations within this window update OHLC and emit a new revision. After the window, mark LOCALLY_FINAL but retain correction capability. “Final” means local processing finality, not exchange-authoritative completeness.

For older late data or historical repair, write a new immutable revision with supersedes_revision and correction_reason. Recalculate only affected higher bars and feature dependencies. Retract or revise scanner candidates with input-revision references; do not execute a historical corrected signal as a new live opportunity. Existing executed orders retain the original decision evidence.

~~~mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> PROVISIONAL_CLOSE: interval end
  PROVISIONAL_CLOSE --> PROVISIONAL_CLOSE: allowed late observation
  PROVISIONAL_CLOSE --> LOCALLY_FINAL: watermark passes
  LOCALLY_FINAL --> CORRECTED: broker reconciliation or late evidence
  OPEN --> INCOMPLETE: gap or source change
  INCOMPLETE --> CORRECTED: verified repair
~~~

## Higher timeframes and daily bars

Aggregate contiguous eligible one-minute children: open from first, close from last, high=max, low=min, volume=sum only when every required child volume is valid, OI=last available observation with age. Store child revision digest and completeness ratio. Missing children produce incomplete parents, not zero-volume synthetic bars.

A no-trade interval can be represented as an explicit NO_TRADE record only when the provider/session evidence establishes it. Optional chart continuity may draw previous-close flat bars, but these are DISPLAY_SYNTHETIC and excluded from indicators by default. Closed-market minutes are not missing bars.

Daily local aggregates use the chosen phase policy and remain separate from broker daily candles. Official settlement/closing prices can differ from last traded price and daily bar close. Store settlement price as a distinct field, especially for derivatives.

## Corporate actions and reconciliation

Store RAW, ADJUSTED and UNKNOWN adjustment bases independently. Broker documentation did not establish a universal corporate-action convention. On split/bonus/dividend or symbol/lot change, attach the action version, verify continuity and suspend affected return/volume baselines until resolved. Never apply an assumed adjustment factor to all historic series.

After disconnect, fetch overlap around the gap and compare same interval, session phase, provider and adjustment basis. Tolerances are price tick, volume units and certified timestamp alignment. Broker bars can replace the selected analytical basis only through an explicit revision policy; raw local observations remain available for audit. Cross-provider differences remain visible rather than averaged away.

## Storage, APIs and acceptance

Read API accepts instrument, interval, range, origin preference and adjustment basis; result includes candles, gaps, revisions and provenance. Stream emits candle.opened, candle.updated, candle.closed and candle.corrected with series key and monotonically increasing revision. UI upserts by series/time/revision.

Acceptance covers all requested intervals, Dhan 25-minute distinctness, session-shortened final bars, midnight/date boundaries, holidays, CAS transition, duplicate observations, cumulative reset, reconnect spanning bars, late correction and corporate-action uncertainty. A property test verifies aggregate OHLC invariants and child revision propagation. A bounded synthetic burst test verifies O(1) update per active interval and no database operation inside the tick update. Historical retrieval supports live context only; no trading replay engine is introduced.
