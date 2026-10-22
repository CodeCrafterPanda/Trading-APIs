# Historical and live candle engine

The engine maintains separate broker-history, locally observed and locally aggregated series. No broker snapshot feed is a documented exhaustive trade tape, so local OHLC/volume can differ from official candles; provenance and quality are part of every bar.

## Series identity and intervals

SeriesKey = instrument_id + interval + session_calendar_version + phase_policy + origin + provider + adjustment_basis. Required platform intervals are 1, 2, 3, 5, 10, 15, 30, 60 minutes and daily; additional provider-native intervals remain available.

| Provider | Native intervals | Current-day source | Window / rate (see [01](01-API-RESEARCH.md)) |
|---|---|---|---|
| Dhan | 1, 5, 15, 25, 60 min; daily | `POST /charts/intraday` with today's range | ≤90 days/request; D-DATA budget; daily `toDate` exclusive |
| Upstox | minutes 1–300, hours 1–5, day, week, month | `GET /v3/historical-candle/intraday/…` | 1–15 min: one month; >15 min and hours: one quarter; day: one decade; U-STANDARD |
| Zerodha | minute, 3minute, 5minute, 10minute, 15minute, 30minute, 60minute, day | `GET /instruments/historical/…` with `to` = now | Staff-published caps: minute 60 d, 3/5/10 min 100 d, 15/30 min 200 d, 60 min 400 d, day 2,000 d; `oi=1` adds OI column; `continuous=1` returns day candles only for expired NFO/MCX futures; Z-HIST 3 req/s; paid Connect tier |

One-minute bars are the canonical local aggregation base. 2/3/10/30-minute bars are locally aggregated for Dhan; Dhan's 25-minute interval is never relabelled 30; Zerodha's native 3/10/30-minute and Upstox's native intervals are stored as separate BROKER_HISTORY series and used as cross-checks, not substituted for the local aggregate. If a native interval and a local aggregate both exist, store both and expose the selected basis; never silently switch a chart or scanner between sources. [Dhan history](https://dhanhq.co/docs/v2/historical-data/), [Upstox V3 history](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data/), [Kite historical](https://kite.trade/docs/connect/v3/historical/)

## Historical retrieval scheduler

Requests are keyed by series, range and normalization version. Maintain a coverage manifest of complete, provisional, missing and unverified intervals. At startup fetch only missing warmup ranges plus a configurable overlap around the latest boundary. Cache completed immutable chunks with raw payload hashes.

Chunking: Dhan ≤90 days per intraday request; Upstox by calendar month/quarter/decade; Zerodha by the staff-published caps treated as mutable — if a response is shorter than the requested window without a documented reason, halve the window, record TRUNCATION_SUSPECTED and retry. Each fetch consumes its broker's rate scheduler (Zerodha 3 req/s shared across instruments). Priorities: open-position context, selected opportunities, active scanner warmup, background universe preparation. Identical concurrent requests coalesce. A provider returning no rows cannot mark a range complete.

Validate parallel arrays (Dhan) or tuple shapes (Upstox arrays, Zerodha `[ts, o, h, l, c, v(, oi)]`), timestamp order and zone (Dhan epoch seconds, Upstox ISO with offset, Zerodha `+0530`), OHLC inequalities, nonnegative volume/OI, session membership and duplicate keys. Reject invalid rows into quarantine; never "repair" extrema without evidence.

Optional IV-baseline sources: Upstox expired-instrument contracts and expired historical candles, Dhan expired options data. They are BROKER_HISTORY series on retired instrument IDs and never join live continuity windows. [Upstox expired](https://upstox.com/developer/api-documentation/expired-instruments/)

## Session alignment and phase boundaries

Persist UTC and apply Asia/Kolkata session rules. For a continuous phase beginning at s with width w, bar_start = s + floor((event_time − s)/w) × w, bounded by phase end; the last bar may be shorter. Never assume midnight alignment, a universal 375-minute session or one close time for all instruments.

The platform calendar ([16](16-SECURITY-AND-COMPLIANCE.md)) is authoritative and effective-dated. Upstox `GET /v2/market/holidays[/{date}]`, `/v2/market/timings/{date}` and `/v2/market/status/{exchange}` (with `cas_eligible_status`) are daily validation inputs that raise a discrepancy alert when they disagree with the calendar; they do not silently overwrite it. Dhan and Zerodha publish no equivalent endpoint. [Upstox holidays](https://upstox.com/developer/api-documentation/get-market-holidays/), [timings](https://upstox.com/developer/api-documentation/get-market-timings/), [status](https://upstox.com/developer/api-documentation/get-market-status/)

Pre-open, continuous, CAS and post-close observations are distinct phases. Default intraday scanners consume continuous-session bars only. Auction/final-close data is stored separately for official daily reconciliation. An authoritative random-close event ends the phase; the system does not predict it.

## Incremental local bar construction

For each active one-minute bar retain first/last valid price, extrema, cumulative-volume baseline/end, turnover baseline/end, observation count, source epoch and quality flags. OHLC uses valid price observations within the bar. Without a verified exchange timestamp, use receipt time with RECEIPT_TIME basis. The active bar updates on every valid observation inside the shard apply; nothing waits for interval end. The update is O(1) per interval and runs inside the L3 budget of [15](15-PERFORMANCE-ENGINEERING.md); no allocation, no clock syscall and no database access occur in it.

Observation ordering within a bar:

| Case | Rule |
|---|---|
| In-order tick | Update extrema/close/volume-delta of the active bar; bump revision |
| Exchange time earlier than the active bar start but inside the 2 s lateness window | Apply to the previous bar as a late revision (PROVISIONAL_CLOSE path); parents recompute from children |
| Older than the lateness window | CORRECTED revision with `correction_reason=LATE_OBSERVATION`; dependents retract per revision rules |
| Duplicate (same exchange time, price, last quantity and cumulative volume as the last applied observation from the same source) | Dropped; `duplicates_dropped` counter; no revision |
| Repeated LTP without a cumulative-volume change | Price fields only; volume unchanged (never sum last quantity) |
| Missing ticks / quiet interval | No synthetic bar; the bar simply has fewer observations; a missing *minute* becomes INCOMPLETE only when a gap record exists |
| Reconnect or source-epoch change mid-bar | Bar flagged INCOMPLETE, volume PARTIAL/UNKNOWN; repaired later from broker overlap |
| Observation outside the session phase of the series | Routed to the matching phase series (pre-open, CAS, post-close); never merged into continuous bars |

### Derived candle geometry

Computed in fixed-point from OHLC on every revision and attached to candle events and the read API; stored values are not persisted (recomputed on read from OHLC so they can never disagree with it):

| Field | Definition | Undefined when |
|---|---|---|
| body | abs(close − open) | — |
| upper_wick | high − max(open, close) | — |
| lower_wick | min(open, close) − low | — |
| range | high − low | — |
| body_ratio | body / range | range = 0 (NULL, not 0 or 1) |
| upper_wick_ratio, lower_wick_ratio | wick / range | range = 0 |
| body_to_wick | body / (upper_wick + lower_wick) | wick sum = 0 |
| direction | UP if close > open; DOWN if close < open; DOJI if body ≤ doji_ticks × tick (default 1 tick) | — |
| pct_from_open | (close − open) / open | open = 0 or missing |
| pct_from_prev_close | (close − prev_close) / prev_close, prev_close = prior bar of the same series or the session's comparable official close for the first bar | prev_close unavailable |
| range_atr | range / ATR14 of the same series | ATR warming |

Geometry on an OPEN bar is provisional and changes with every observation; detectors that need completed geometry (SC21–SC31) consume closed bars only.

Volume is exact only relative to a certified cumulative counter (Dhan quote/full volume, Upstox `vtt`, Zerodha day volume) with uninterrupted comparable boundaries. At subscription midway through a bar, prior volume is unknown. A jump spanning several bars cannot be allocated; mark affected bars UNKNOWN/PARTIAL. A negative delta implies reset/correction/source change and invalidates the baseline.

Exact VWAP uses certified cumulative turnover ÷ cumulative volume or the broker session ATP (all three feeds supply ATP in quote/full modes). Without turnover, sum(price × volume delta) is OBSERVED_VWAP_PROXY; no scanner requiring exact VWAP may consume it silently.

## Completion, lateness and revisions

At scheduled bar end emit PROVISIONAL_CLOSE. A configurable 2-second allowed-lateness window updates OHLC with new revisions; afterwards mark LOCALLY_FINAL with correction capability retained. "Final" means local processing finality, not exchange-authoritative completeness.

Older late data or historical repair writes a new immutable revision with supersedes_revision and correction_reason. Recalculate only affected higher bars and dependencies. Retract or revise scanner candidates with input-revision references; never execute a corrected historical signal as a live opportunity. Executed orders retain original decision evidence.

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

Every enabled interval (1, 2, 3, 5, 10, 15, 30, 60 minutes, daily) has its own **active accumulator** per instrument, updated from the same observation in the same shard apply as the one-minute bar: open from the first valid price in the parent window, high/low as running extrema, close as last, volume as cumulative-counter delta from the parent's own baseline, observation count and quality. All intervals therefore advance together on every tick and share one state version; a 5-minute chart and a 1-minute chart can never show different "current" prices. Cost is O(T) for T enabled intervals, bounded by the L3 budget.

At a child close the parent is additionally recomputed from its contiguous eligible children: open from first, close from last, high = max, low = min, volume = sum only when every required child volume is valid, OI = last available with age. The child-derived value is authoritative for the LOCALLY_FINAL parent; a mismatch against the running accumulator sets `ACCUMULATOR_MISMATCH` and the child-derived bar wins. Store child revision digest and completeness ratio. Missing children produce incomplete parents, never zero-volume synthetic bars. A late child revision propagates only to parents that contain it and to detectors subscribed to those parents.

A NO_TRADE record exists only when provider/session evidence establishes it. Chart continuity may draw DISPLAY_SYNTHETIC flat bars, excluded from indicators. Closed-market minutes are not missing bars.

Daily local aggregates use the chosen phase policy and stay separate from broker daily candles. Official settlement/closing prices can differ from last trade and daily close; store settlement price as a distinct field, especially for derivatives.

## Corporate actions and reconciliation

Store RAW, ADJUSTED and UNKNOWN bases independently. No broker documents a universal corporate-action convention; Zerodha `continuous` stitches expired futures without a documented adjustment. On split/bonus/dividend or symbol/lot change, attach the action version, verify continuity and suspend affected baselines until resolved. Never apply an assumed adjustment factor across series.

After disconnect, fetch overlap around the gap and compare same interval, phase, provider and adjustment basis. Tolerances are price tick, volume units and certified timestamp alignment. Broker bars replace the analytical basis only through an explicit revision policy; raw local observations remain for audit. Cross-provider differences remain visible, never averaged away.

Regeneration and gap repair, in order of preference: (1) rebuild local aggregates from stored one-minute children; (2) rebuild one-minute bars from retained raw observations (tick retention window in [13](13-DATA-ARCHITECTURE.md)); (3) fill from BROKER_HISTORY for the exact series key with origin recorded; (4) leave the gap flagged. Interpolation, carry-forward of the previous close as OHLC, or averaging providers is never a repair. Each regeneration writes new revisions with `supersedes_revision` and the source used; charts receive candle.corrected per affected bar.

## Storage, APIs and acceptance

Read API accepts instrument, interval, range, origin preference and adjustment basis; result includes candles (with geometry), gaps, revisions and provenance, paged backwards from a cursor for incremental chart history loading. Stream emits candle.opened/updated/closed/corrected with series key, state version and monotonically increasing revision. candle.updated for OPEN bars is coalesced per series at the view cadence (latest state wins); candle.opened/closed/corrected are published on the priority lane and never coalesced ([14](14-DASHBOARD-UX.md)). UI upserts by series/time/revision and never replaces the whole series on a live update.

Acceptance covers all requested intervals on all three providers, Dhan 25-minute distinctness, Zerodha native 3/10/30-minute vs local aggregate agreement within tolerance, Zerodha window-cap truncation detection, Upstox calendar chunking, session-shortened final bars, midnight/date boundaries, holidays (including Upstox holiday-endpoint disagreement alerts), CAS transition, duplicate and out-of-order observations, cumulative reset, reconnect spanning bars, late correction, regeneration from children/raw/broker sources and corporate-action uncertainty. Property tests verify aggregate OHLC invariants, geometry identities (body + wicks = range; ratios within [0,1]), accumulator-vs-children equality for complete parents, and child revision propagation. A synthetic burst test verifies O(T) update across all enabled intervals within the L3 budget of [15](15-PERFORMANCE-ENGINEERING.md), zero allocation, and no database operation inside the tick update.
