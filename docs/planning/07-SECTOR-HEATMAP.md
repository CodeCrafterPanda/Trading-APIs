# Sector intelligence and stock heatmaps

Sector intelligence combines official index observations, dated constituent mappings and internally calculated breadth/momentum/liquidity measures. The UI must distinguish an official sector-index return from an equal-weight or custom-basket calculation. **From brokers:** index LTP/previous close (any adapter; Dhan `IDX_I`, Upstox `NSE_INDEX|…`, Zerodha index packets carry no volume/depth), constituent LTP, previous close, cumulative volume/turnover, ATP, bid/ask. **Calculated internally:** everything in the metric table below, all rankings, breadth, RVOL baselines, rotation and leadership. No broker feed establishes sector membership or official constituent weights.

## Catalog and provenance

Import the official NSE sector index catalog and licensed constituent/methodology data with source URL, retrieved_at, effective_from/to and content hash. Initial catalog includes Auto, Bank, Financial Services, FMCG, IT, Media, Metal, Pharma, Private Bank, PSU Bank, Realty, Consumer Durables, Oil & Gas, Healthcare and Chemicals; import updates dynamically rather than freezing this list. NSE industry taxonomy and user-defined sectors are separate namespaces. [Methodology catalog](https://www.niftyindices.com/resources/index-methodology), [industry classification](https://www.niftyindices.com/resources/industry-classification).

SectorDefinition stores sector_id, namespace=OFFICIAL_INDEX|INDUSTRY|CUSTOM, name, benchmark_instrument_id, membership_version, weighting_method and license_evidence. Membership contains instrument_id, effective dates, weight nullable, classification confidence and action version. Multi-sector membership is allowed; cross-sector aggregations must avoid double-counting when computing market-wide breadth.

Official weights require valid licensed inputs. If unavailable, provide a clearly labeled equal-weight constituent basket without presenting it as the official index. Public redistribution/storage rights are an external gate. [Index licensing](https://www.niftyindices.com/offerings/index-licensing).

## Metric definitions

Let P_i be current price, C_i comparable prior official close, r_i=P_i/C_i−1, v_i valid cumulative volume, b_i median cumulative volume at the same elapsed session point across the configured valid sessions, and w_i effective normalized basket weight. Missing/invalid observations are excluded and reduce coverage.

| Metric | Definition | Quality condition |
|---|---|---|
| Official sector return | IndexLTP / indexPreviousClose − 1 | Fresh broker index and comparable close |
| Derived basket return | sum(w_i r_i) / sum(valid w_i) | Show valid-weight coverage; not official return |
| Equal-weight breadth return | mean(valid r_i) | Explicit equal-weight label |
| Advances/declines | count(r_i > epsilon), count(r_i < −epsilon) | epsilon configurable, initially one comparable tick return |
| Breadth | (advances−declines)/(advances+declines) | Denominator zero → unavailable |
| Participation | valid_count / expected_active_constituents | Inactive members separated from stale members |
| Momentum | weighted return over 1/5/15-minute windows | Same origin/phase; warmup complete |
| Relative strength | sector window return − benchmark window return | Matched timestamps and basis |
| Relative volume | median or weighted mean(v_i/b_i) | Baseline aligned by elapsed phase, corporate actions handled |
| Volume confirmation | fraction of directional members with RVOL above threshold | No volume for index required; uses constituents |
| Realized volatility | sqrt(sum squared log returns over selected window) | Label nonannualized or explicit annualization convention |
| Leadership change | crossing of rank/score bands held for hysteresis duration | Avoid flicker from one update |
| Rotation | change in sector RS and momentum quadrants over window | Descriptive transition, not capital-flow proof |

No division by zero, missing-as-zero or mixing source epochs. Benchmark can be broad NSE index or configured market basket. A BSE listing uses its own canonical price and the configured cross-venue mapping policy; same ISIN does not make execution venue interchangeable.

## Incremental computation

Each constituent owner sends ContributionChanged containing sector IDs, old/new return, breadth class, weight, RVOL, volatility contribution and version. Sector worker subtracts old/adds new sufficient statistics. Rank updates use ordered indexes at O(log S) for sectors and O(log N_sector) for stocks. Only membership changes or scheduled audits require a full sector rebuild.

Maintain price-weight coverage, volume coverage, benchmark age and membership age separately. Default readiness threshold is 80% valid constituent weight/count for sector aggregate ranking, configurable per strategy. Below threshold show PARTIAL and prevent strategies requiring sector confirmation from executing.

## Configurable ranking profiles

RankingProfile contains id/version, scope, direction, feature transforms, weights, hard gates, missing-data policy and update cadence. Transform features to comparable bounded scores using rolling robust median/MAD or cross-sectional percentiles. Store the transformation universe and minimum valid sample count. Insufficient samples make the feature unavailable, not an invented rank.

Default sector profile weights: return 0.20, momentum 0.25, relative strength 0.25, breadth 0.20, volume confirmation 0.10. Bearish profiles invert directional features while preserving liquidity/quality gates. Do not invert volatility mechanically; its desirability depends on the strategy.

Stock profiles include return, RVOL, momentum, relative strength versus sector, distance/side of certified VWAP or labeled proxy, breakout quality, liquidity, volatility suitability, spread and confirmed structure. Hard spread/turnover/data gates run before score. Missing mandatory features invalidate ranking for execution; optional missing weights renormalize only when a profile explicitly permits it and reduce confidence.

Every displayed score exposes component contributions, input age, coverage and profile version. A stock can rank strongly within a weak sector; sector and stock contributions remain visible instead of hidden in one number.

## Heatmap and drilldown behavior

Main heatmap uses signed percentage return with a zero-centered accessible palette, numeric labels and an alternative table. Tile size defaults to equal area; optional licensed weight/turnover sizing is labeled. Color scale is stable within a view interval, with clipped extremes indicated rather than constantly rescaling all tiles.

Selecting a sector opens constituents, score components, advances/declines, coverage, momentum/RS traces and leadership changes. Selecting a stock opens its chart, scanner evidence and account-aware order panel. Keyboard navigation, focus retention and color-independent indicators are required. Stale tiles use a patterned/neutral state and last-update age, never the same vivid color as fresh values.

Server publishes only changed sector/stock rows and only changed fields on the P2 display lane at a proposed 5–10 Hz view cadence, not every raw tick; leadership changes and readiness transitions go on the P1 event lane ([14](14-DASHBOARD-UX.md)). The sector worker marks dirty tiles in the publisher's per-view set; it never serializes. The canvas heatmap repaints dirty tiles only. Sort order can be frozen while interacting; a “rank changed” marker avoids rows jumping under a pointer. Virtualization limits rendered rows without limiting analytical coverage. View subscriptions expire on disconnect.

## Acceptance

Use hand-calculated baskets to verify weighted/equal returns, breadth, missing coverage, zero denominators, membership changes and duplicated multi-sector members. Verify a single stock update changes only its sectors and affected ranks. Confirm official index return is never overwritten by a computed basket. At the declared load envelope, one constituent contribution update plus rank-index change meets path L4 (p50 0.02 ms, p95 0.10 ms, p99 0.30 ms) and tile publication meets L7c/L8c in [15](15-PERFORMANCE-ENGINEERING.md); UI rendering follows [14](14-DASHBOARD-UX.md). Licensing and current constituents remain production evidence requirements (gate G09 in [16](16-SECURITY-AND-COMPLIANCE.md)).
