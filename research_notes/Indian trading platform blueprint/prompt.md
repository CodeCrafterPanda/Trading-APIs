# Master Planning Prompt — High-Performance Indian Stock Market Scanner & Automated Trading Platform

Act as a **Principal Trading Systems Architect, Quantitative Systems Engineer, Low-Latency Backend Architect, and Indian Stock Market API Specialist**.

I want to build a completely new, standalone, production-grade intraday trading platform for the Indian stock market using **DhanHQ, Upstox, Zerodha as three independent, fully functional adapters**.

Your task is to conduct detailed technical research and produce a comprehensive, developer-ready architecture and implementation plan.

**Do not write or implement application code yet.** Focus exclusively on research, architecture, technical decisions, detailed specifications, development phases, granular tasks, API contracts, performance engineering, and implementation handoff documents.

## 1. Official API Documentation

### DhanHQ
- https://docs.dhanhq.co/api/v2/trading-apis
- https://docs.dhanhq.co/api/v2/data-apis

### Upstox
- https://upstox.com/developer/api-documentation/webhook
- https://upstox.com/developer/api-documentation/installing-sdk

### Zerodha
- https://kite.trade/docs/connect/v3/websocket/
- https://kite.trade/docs/connect/v3/sdks/

Explore all relevant linked official documentation, including WebSocket market feeds, instrument masters, historical candles, options, order placement, order updates, authentication, rate limits, and broker-specific capabilities.

Verify current API versions, restrictions, subscription limits, market-data entitlements, and supported functionality.

Do not assume an API supports something without documentation.

## 2. Primary Objectives

Design a live-market trading platform supporting:

1. NSE and BSE equities, indices, futures, and options.
2. Both DhanHQ and Upstox market-data and execution adapters.
3. Real-time WebSocket market feeds.
4. Historical candle retrieval and real-time candle construction.
5. Sector-wise live market heatmaps.
6. Dynamic bullish and bearish sector rankings.
7. Stock-wise rankings within each sector.
8. Real-time market scanners across the eligible trading universe.
9. Bullish, bearish, sideways, trending, volatile, and reversal markets.
10. Options-chain analysis, strike selection, and derivatives intelligence.
11. Automated opportunity identification and ranking.
12. Automated and manual live order placement and execution.
13. Real-time order, position, stop-loss, target, and risk management.
14. A responsive, real-time monitoring and trading dashboard.
15. Millisecond-scale internal processing and decision-making.

There must be **no hardcoded limit on the number of stocks analyzed or trades generated**. Market opportunities, data quality, liquidity, broker limits, risk, and available capital should determine actual activity.

## 3. Dual-Broker Adapter Architecture

Design common interfaces for:

- LiveMarketDataAdapter
- HistoricalDataAdapter
- InstrumentAdapter
- OptionsDataAdapter
- TradingAdapter
- OrderUpdateAdapter
- PortfolioAdapter
- FundsAndMarginAdapter

Implementations must be planned for both DhanHQ and Upstox.

Research and define:
- Authentication and token management.
- WebSocket connection management.
- Subscription and unsubscription.
- Quote formats and normalization.
- Broker-specific instrument ID mapping.
- Historical candle retrieval.
- Options and futures metadata.
- Order placement, modification, and cancellation.
- Order updates through WebSockets, webhooks, or postbacks.
- Position, holdings, and funds synchronization.
- Rate limits and request scheduling.
- Retry, reconnect, and failover handling.

Allow independent configuration of the market-data provider and order-execution broker.

Design reliable data-source failover without creating duplicate orders or mixing incompatible broker account states.

## 4. High-Performance Live Market Data Engine

Plan an event-driven, WebSocket-first architecture.

Cover:
- Large-universe instrument subscriptions.
- Adaptive subscription management.
- Binary/Protobuf decoding.
- Normalized tick and quote events.
- In-memory market state.
- Bid/ask and market depth.
- Volume and turnover.
- Open interest.
- Exchange and provider timestamps.
- Stale-data detection.
- WebSocket reconnect and recovery.
- Data gap detection.
- Efficient event queues and dedicated processing workers.

Compare **Rust, Go, Python, and hybrid architectures** and recommend the most suitable technology stack for genuinely low-latency operation.

Explain how to avoid unnecessary REST polling, database calls, serialization, and full-universe recalculation on every tick.

## 5. Historical Candles and Real-Time Candle Engine

Design historical retrieval and live candle generation for all supported intervals, including:

- 1-minute.
- 2-minute.
- 3-minute.
- 5-minute.
- 10-minute.
- 15-minute.
- 30-minute.
- Hourly.
- Daily.
- Other provider-supported intervals.

Include:
- Historical API request limits.
- Candle caching.
- Incremental retrieval.
- Missing-candle detection.
- Exchange-time alignment.
- Candle completion.
- Late or missing ticks.
- Aggregation into higher timeframes.
- Corporate-action handling where applicable.
- Data reconciliation after WebSocket disconnections.

Clearly distinguish broker-provided historical candles from locally constructed candles.

## 6. Advanced Sector-Wise Live Heatmap

Design a dynamic sector intelligence engine.

Include official NSE sector indices and configurable sector classifications.

Calculate and display:
- Sector percentage gains and losses.
- Strongest and weakest sectors.
- Sector momentum.
- Sector relative strength.
- Advancing and declining constituents.
- Sector breadth.
- Volume-based confirmation.
- Sector volatility.
- Intraday leadership changes.
- Sector rotation.
- Bullish and bearish rankings.

Create stock-level heatmaps within every sector.

Rank stocks using:
- Percentage change.
- Volume and relative volume.
- Price momentum.
- Relative strength against their sector.
- VWAP.
- Breakout/breakdown quality.
- Liquidity.
- Volatility.
- Spread.
- Market structure.

Design configurable rankings rather than one fixed scoring formula.

Explain exactly which data is obtained from brokers and which metrics must be calculated internally.

## 7. Comprehensive Market Scanner

Research and design an extensible, real-time scanning engine.

Cover at least these scanner families:

**Price and momentum:** Top gainers/losers, acceleration, momentum continuation, exhaustion, gap-ups, gap-downs, gap fills, intraday trends, and reversals.

**Volume and liquidity:** Unusual volume, relative volume, volume spikes, turnover acceleration, spread anomalies, and liquidity changes.

**Technical structure:** Support/resistance, swing structure, consolidation boxes, breakouts, breakdowns, retests, false breakouts, order blocks, supply/demand zones, liquidity sweeps, and multi-timeframe alignment.

**Sector intelligence:** Strong stocks in strong sectors, weak stocks in weak sectors, sector divergence, rotation, and emerging leaders or laggards.

**Volatility:** Expansion, compression, abnormal movement, and regime transitions.

**Derivatives:** Open-interest changes, put-call ratios, implied volatility, Greeks, strike-wise liquidity, futures buildup, and options activity.

For each scanner, document:
- Purpose.
- Required live/historical data.
- Detection algorithm.
- Parameters.
- Update frequency or event trigger.
- Data dependencies.
- Computation complexity.
- Candidate output.
- Confidence and ranking methodology.
- Performance considerations.

Avoid computing every indicator for every instrument unnecessarily.

## 8. Market Regime and Opportunity Ranking

Plan engines to identify:

- Bullish markets.
- Bearish markets.
- Sideways markets.
- High and low volatility.
- Breakout and breakdown conditions.
- Trend continuation.
- Reversals.
- Uncertain or mixed conditions.

Define a configurable opportunity-ranking framework that considers sector strength, instrument momentum, market structure, volume, liquidity, volatility, spread, estimated costs, and risk/reward.

Opportunities should be continuously reevaluated as the market changes.

No arbitrary requirement to generate trades.

## 9. Equity, Futures and Options Trading

Plan complete support for:

**Equities**
- Intraday long and short positions.
- Valid order types.
- Entry and exit management.
- Quantity calculation.
- Stop-loss and target orders.
- Product restrictions.

**Futures**
- Contract discovery.
- Expiry and lot sizes.
- Margin.
- Long and short trading.
- Rollover considerations.

**Options**
- Option-chain retrieval.
- CE/PE discovery.
- ATM/ITM/OTM strikes.
- Strike ranking.
- Open interest.
- IV and Greeks.
- Premium, spread, volume, and liquidity.
- Contract selection for bullish/bearish opportunities.
- Expiry-related risk.
- Underlying-driven and premium-driven exits.

Include accurate instrument mapping and broker-specific trading constraints.

## 10. Live Order Execution and Risk Management

Design a low-latency, reliable order management system.

Include:
- Pre-trade validation.
- Capital and margin checks.
- Order preparation.
- Broker routing.
- Submission.
- Order acknowledgements.
- Execution tracking.
- Partial fills.
- Modification/cancellation.
- Rejections.
- Order timeouts.
- Duplicate-order prevention.
- Position reconciliation.
- Stop-loss and target management.
- Configurable trailing stops.
- Emergency risk controls.
- Safe behavior during market-data or broker outages.

Do not interpret API acceptance as exchange execution.

Make risk checks efficient but never bypass them to achieve lower latency.

## 11. Millisecond-Level Architecture and Performance

This is a major priority.

Design dedicated processing components or workers for:

1. WebSocket ingestion.
2. Message decoding and normalization.
3. Instrument state.
4. Candle generation.
5. Sector calculations.
6. Stock rankings.
7. Market scanners.
8. Market-regime classification.
9. Opportunity scoring.
10. Risk evaluation.
11. Order routing/execution.
12. Order and position updates.
13. Persistent storage.
14. Dashboard event publishing.
15. System health and telemetry.

Evaluate worker boundaries, process isolation, concurrency, memory sharing, queues, backpressure, batching, and deployment topology.

Specify measurable latency objectives with p50, p95, p99, and worst-case measurements.

**Aim for millisecond-scale internal event-to-decision latency without making unsupported guarantees about broker or exchange execution speed.**

Identify which functions must operate synchronously on the critical path and which can execute asynchronously.

## 12. Data Storage and Retention

Recommend an efficient design for:
- Instrument masters.
- Sector constituents.
- Historical candles.
- Recent live ticks.
- Market depth snapshots.
- Calculated features.
- Scanner events.
- Opportunity rankings.
- Orders.
- Executions.
- Positions.
- Audit logs.
- Performance telemetry.

Compare appropriate databases and storage technologies.

Keep hot-path operations primarily in memory.

Define retention, compression, partitioning, recovery, and storage-growth policies.

## 13. Real-Time UI/UX Plan

Design screens for:
- Main trading dashboard.
- Sector heatmap.
- Sector-to-stock drilldown.
- Live stock scanner.
- Stock candlestick chart.
- Futures/options scanner.
- Option chain.
- Trade opportunities.
- Live order placement.
- Open orders and positions.
- Risk and capital overview.
- Broker connection settings.
- Scanner and strategy configuration.
- Feed health and performance monitoring.

Recommend an appropriate frontend stack and real-time transport.

Design efficient incremental UI updates without overwhelming the browser.

## 14. Security, Reliability and Indian Market Requirements

Include:
- Secure broker credentials.
- Encrypted token handling.
- Authentication renewal.
- Least-privilege configuration.
- Order auditability.
- Exchange calendars.
- Trading-session handling.
- Broker outages.
- Rate-limit compliance.
- Applicable SEBI/exchange/broker algo-trading requirements.
- Instrument-specific trading restrictions.
- Safe recovery after process crashes.
- System and order-state reconciliation.

Research current Indian regulatory and broker requirements using authoritative sources.

## 15. Required Planning Deliverables

Produce the following detailed Markdown documents:

1. **00-EXECUTIVE-OVERVIEW.md** — Scope, goals, assumptions, and final recommendations.
2. **01-API-RESEARCH.md** — DhanHQ/Upstox API capability and restriction matrix with official sources.
3. **02-SYSTEM-ARCHITECTURE.md** — Complete architecture, components, data flows, and diagrams.
4. **03-TECH-STACK-DECISIONS.md** — Languages, frameworks, databases, queues, and justified choices.
5. **04-BROKER-ADAPTER-CONTRACTS.md** — Every adapter, interface, method, payload, event, and error.
6. **05-MARKET-DATA-ENGINE.md** — WebSocket ingestion, subscriptions, normalization, and recovery.
7. **06-CANDLE-ENGINE.md** — Historical retrieval, real-time aggregation, and data quality.
8. **07-SECTOR-HEATMAP.md** — Sector mapping, ranking, formulas, and frontend behavior.
9. **08-MARKET-SCANNER-CATALOG.md** — Detailed individual scanner specifications.
10. **09-REGIME-AND-RANKING-ENGINE.md** — Market conditions, candidate generation, and scoring.
11. **10-EQUITY-FUTURES-OPTIONS.md** — Product-specific trading and derivatives intelligence.
12. **11-ORDER-EXECUTION-ENGINE.md** — OMS, broker routing, order lifecycle, and reconciliation.
13. **12-RISK-MANAGEMENT.md** — Risk checks, stops, targets, and emergency behavior.
14. **13-DATA-ARCHITECTURE.md** — Schemas, in-memory state, databases, and retention.
15. **14-DASHBOARD-UX.md** — Screens, components, interactions, and live updates.
16. **15-PERFORMANCE-ENGINEERING.md** — Critical paths, low-latency design, benchmarks, and targets.
17. **16-SECURITY-AND-COMPLIANCE.md** — Credentials, exchange rules, and operational requirements.
18. **17-IMPLEMENTATION-ROADMAP.md** — Ordered development phases, dependencies, and milestones.
19. **18-GRANULAR-DEVELOPMENT-TASKS.md** — Small, actionable tasks with implementation details and acceptance criteria.
20. **19-DECISIONS-AND-RISKS.md** — Architectural decisions, limitations, risks, mitigations, and unresolved external requirements.
21. **20-DEVELOPER-HANDOFF.md** — Complete implementation-ready instructions and recommended starting sequence.

## 16. Granular Development Planning Requirements

Do not produce only high-level feature descriptions.

Break every subsystem into detailed:
- Epics.
- Features.
- User stories or engineering stories.
- Development tasks.
- Technical subtasks.
- API contracts.
- Data models.
- Dependencies.
- Failure cases.
- Acceptance criteria.
- Performance criteria.
- Implementation order.

Each task must explain what to build, which components it affects, how it should function, what it depends on, and how completion will be verified.

Include architecture diagrams, event-flow diagrams, sequence diagrams, entity relationships, state machines, and representative API/event contracts where useful.

Make all key architectural decisions now instead of leaving choices to the implementation engineer.

Clearly identify external requirements that cannot be verified without broker credentials or account access.

## 17. Strict Scope and Execution Instructions

- This is a **brand-new greenfield platform**.
- Do not audit or reference an existing repository or application.
- Do not assume integration with another trading system.
- Do not include paper trading.
- Do not include backtesting.
- Do not include historical trading replay.
- Historical candle retrieval is required for live-market intelligence, not for constructing a backtesting subsystem.
- Do not impose a fixed number of stocks, opportunities, or trades.
- Do not implement application code in this planning task.
- Do not silently remove requested features.
- Do not recommend switching to a single broker; both adapters are required.
- Do not substitute polling for streaming when supported live feeds are available.
- Do not promise guaranteed daily profits or guaranteed millisecond exchange fills.
- Do not introduce unrelated product features.
- Make reasonable, well-supported technical decisions without repeatedly asking for confirmation.
- Clearly distinguish verified broker capabilities from internally calculated capabilities.
- Ensure the architecture supports actual live order execution once the necessary credentials, permissions, and regulatory requirements are satisfied.

## FINAL EXPECTATION

Deliver a complete, technically researched, implementation-ready development blueprint.

The resulting documentation must allow an implementation agent to develop the platform systematically, module by module, without needing to rediscover the architecture, decide fundamental technical choices, or reinterpret vague requirements.

**Priorities, in order:**

1. Reliable live market data through both adapters.
2. Extremely fast market processing and real-time sector/stock intelligence.
3. Comprehensive, configurable market scanning.
4. Accurate and timely opportunity ranking.
5. Reliable equity, futures, and options order execution.
6. Effective live position and risk management.
7. Clear and responsive user interfaces.
8. Maintainability, extensibility, operational reliability, and observability.

Begin with official API research, finalize the architecture, and then create all detailed planning and developer handoff documents.

**Do not begin implementation. The only output expected from this task is the complete plan and specification pack. But it should be fastest at calculations and output results. Under ms operations should happen**