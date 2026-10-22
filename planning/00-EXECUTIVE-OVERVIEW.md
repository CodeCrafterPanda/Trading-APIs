# Indian live trading platform blueprint

Issued **22 October 2026**. Official documentation researched **21 October 2026**. Timezone: Asia/Kolkata. Status: **implementation specification; no application implemented or live account certified**.

Build a new standalone platform with independent, complete DhanHQ and Upstox adapters. Use Rust for ingestion, market state, candles, sector intelligence, scanners, opportunity ranking, risk and order management; TypeScript/React for the dashboard; PostgreSQL for transactional truth; ClickHouse and Parquet for asynchronous analytics storage. Treat reliable live data and explicit execution uncertainty as foundational requirements. Millisecond internal processing is an engineering target, not a promise of millisecond broker acknowledgement or exchange execution.

## Scope and operating assumption

The initial deployment serves one investor operating their own enabled DhanHQ and Upstox accounts. This is an architectural assumption, not a determination of account eligibility. A commercial, multi-client, advisory or algo-provider offering requires a separate broker and regulatory onboarding route. Each broker remains usable independently for market data, history, instruments, options, orders, order updates, portfolio and funds. Configuring Dhan data with Upstox execution, or the reverse, is supported after canonical mapping and permissions checks.

The platform covers NSE and BSE cash equities, indices as intelligence inputs, and enabled equity futures/options. An index quote does not imply the index itself can be ordered. Instrument discovery is broad; data coverage, scanner readiness and trading eligibility are distinct state dimensions. Every discovered instrument remains visible in the catalog. Actual live subscriptions and execution depend on entitlement, quotas, market liquidity, freshness, risk and capital.

Historical candles support live indicator warmup, relative-volume baselines and gap repair. There is no paper-trading subsystem, backtesting product, historical trading replay or integration with an existing application. The planning deliverables contain representative schemas and algorithms only.

## Final decisions

| Concern | Selected design | Reason |
|---|---|---|
| Core language | Rust, pinned stable toolchain at implementation kickoff | One typed model, controlled allocations and explicit ownership |
| Broker boundary | Independent Rust adapter processes | Separate credentials, decoder failures, reconnects and quotas |
| State ownership | Stable instrument shards; one writer per shard | Ordered incremental updates without global locks |
| Execution ownership | One active account actor and one fenced gateway per broker account | Prevent competing reservations and submissions |
| Durability | PostgreSQL synchronous intent/reservation commit before network submission | Recover intent after crashes and ambiguous sends |
| Analytics | ClickHouse batches and compressed Parquet | Keep scans and bulk persistence outside decision path |
| UI | React, TypeScript, Vite, selector stores, virtualized tables and canvas charts | Incremental rendering with bounded subscription views |
| Transport | Typed local IPC, bounded queues; browser WebSocket deltas | No Kafka/Redis hop for every tick |
| Deployment | Linux, dedicated resources, approved static egress, warm passive standby | Measured resource isolation and controlled takeover |
| Failover | Validated market-data source switch; account execution remains pinned | A different broker cannot close the original account's position |

Detailed reasoning is in [03](03-TECH-STACK-DECISIONS.md), component placement in [02](02-SYSTEM-ARCHITECTURE.md), and the execution invariant in [11](11-ORDER-EXECUTION-ENGINE.md).

## Required outcomes

Reliable streaming produces normalized, provenance-bearing observations and explicit data-quality states. Candles retain whether they came from broker history or local sampled observations. Sector and stock rankings update incrementally. Scanners activate only when their dependencies change. Opportunities expire, downgrade or invalidate as conditions change; producing no actionable opportunity is a valid result.

Equities support enabled intraday long/short routes with quantity, tick, band and product checks. Futures use actual dated contracts, lots, freeze quantities, margins and expiry controls. Options use active chain/contract discovery, strike liquidity, IV/Greeks provenance, underlying and premium exits, and non-atomic multi-leg controls. Each route supports manual and automated intent creation through the same risk and OMS gates.

The dashboard must expose market coverage, stale fields, sector/stock intelligence, scanner evidence, options, orders, positions, account risk, credentials status and performance. A green connectivity indicator alone never means data is complete or trading is safe.

## Non-negotiable invariants

1. No order leaves a gateway before its durable intent and reservation exist.
2. API acceptance is not exchange execution; cancel acceptance is not proof that no late fill exists.
3. A potentially transmitted command with no definitive response becomes **SUBMISSION_UNKNOWN**. It is reconciled, never blindly retried.
4. Tags/correlation IDs support matching; they are not a documented exactly-once guarantee.
5. An exit always targets the broker/account/product owning the position.
6. No credential, PIN or broker token reaches the browser or ordinary logs.
7. Unknown account state, unsupported session rules or insufficient critical data blocks new risk.
8. Risk checks are never removed to improve a latency number.
9. No hardcoded stock, opportunity or trade quota exists. Broker limits and explicitly configured economic risk limits still apply.
10. An application emergency stop first blocks new exposure and manages outstanding risk. Broker kill switches require flat positions and are not liquidation primitives. [Dhan controls](https://dhanhq.co/docs/v2/traders-control/), [Upstox kill switch](https://upstox.com/developer/api-documentation/update-kill-switch/)

## Evidence classification

**Verified broker fact** means supported by a linked official source reviewed on the research date. It does not mean verified against this user's account. **Proposed design** means a decision this blueprint makes for implementation. **External gate** means information requiring account access, licensing, current circulars, broker clarification or measured deployment evidence. The canonical limits and endpoints live in [01](01-API-RESEARCH.md); other documents reference them instead of inventing alternate constants.

Material limits include Dhan's native intraday intervals 1/5/15/**25**/60 minutes and Upstox's mode-dependent per-user subscription ceilings. Requested extra intervals are locally aggregated. Broker snapshots are not a documented complete exchange trade tape, so live candles and derived volume measures have explicit precision flags. [Dhan history](https://dhanhq.co/docs/v2/historical-data/), [Upstox feed](https://upstox.com/developer/api-documentation/v3/get-market-data-feed/)

Session configuration is effective-dated. Current CAS implementation and the prospective December 2026/February 2027 changes must remain separate. The research baseline does not establish BSE-specific circular conformance or account-specific square-off deadlines. [16](16-SECURITY-AND-COMPLIANCE.md) provides the source chain and launch gates.

## Acceptance and success measures

The initial compute target is local frame receive to candidate decision at p50 1 ms, p95 3 ms, p99 8 ms under a declared load envelope. Decision-to-request-write including durable commit targets p50 3 ms, p95 8 ms, p99 15 ms. These are unmeasured targets. Record sample count, observed maximum, queue age, event loss, CPU, memory and clock uncertainty; do not add unrelated percentiles. Broker acknowledgement and fill distributions are measured separately. See [15](15-PERFORMANCE-ENGINEERING.md).

Functional acceptance requires every eight-adapter interface implemented for both brokers, independent conformance evidence, instrument mapping without ambiguity, explainable scanner outputs, no submission on unknown state, correct partial-fill accounting and successful crash/reconnect reconciliation. Read-only commissioning precedes separately authorized live commissioning. Offline protocol and fault tests are verification tools, not a simulated trading product.

## Delivery and next action

Follow [17](17-IMPLEMENTATION-ROADMAP.md), implement the ordered task IDs in [18](18-GRANULAR-DEVELOPMENT-TASKS.md), and use [20](20-DEVELOPER-HANDOFF.md) as the execution checklist. External gates remain visible rather than being replaced by optimistic assumptions. The first milestone is a versioned contract and instrument foundation with two read-only live adapters; execution is enabled only after account, compliance, durability and reconciliation evidence is complete.
