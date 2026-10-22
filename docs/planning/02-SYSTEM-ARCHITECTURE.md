# System architecture and event ownership

Greenfield, event-driven design. Component boundaries separate failures and ownership; they do not imply fifteen network microservices. The market critical path stays in memory with incremental state and sub-millisecond budgets ([15](15-PERFORMANCE-ENGINEERING.md)); the execution path deliberately includes durable intent storage.

## Deployment topology

~~~mermaid
flowchart LR
  D[Dhan services] <--> DA[Dhan adapter process]
  U[Upstox services] <--> UA[Upstox adapter process]
  Z[Zerodha Kite services] <--> ZA[Zerodha adapter process]
  DA --> IPC[Typed bounded local IPC]
  UA --> IPC
  ZA --> IPC
  IPC --> CORE[Instrument shard workers]
  CORE --> C[Candles and features]
  C --> S[Sectors and scanners]
  S --> O[Regime and opportunity ranking]
  O --> A[Account actors and risk]
  A --> PG[(PostgreSQL durable ledger)]
  PG --> G[Fenced execution gateways]
  G --> DA
  G --> UA
  G --> ZA
  IPC --> A
  CORE --> P[Async persistence workers]
  S --> P
  P --> CH[(ClickHouse)]
  P --> OB[(Parquet object archive)]
  A --> PUB[UI publisher and control API]
  S --> PUB
  PUB <--> UI[React browser]
  H[Health and telemetry] -.-> CORE
  H -.-> A
  H -.-> PUB
~~~

One Linux compute host runs core, the three adapter processes and the control process with separate CPU/memory budgets ([15](15-PERFORMANCE-ENGINEERING.md) §rules). PostgreSQL runs on a dedicated durable service/host or local NVMe with synchronous commit; analytics storage never shares its latency-sensitive disk. A passive standby receives configuration and can restore durable state, but cannot submit until the old gateway is demonstrably fenced. All three adapters are deployed and connected even when one supplies the active data source for a given instrument group, so failover is a validated switch, not a cold start.

Choose an Indian region by measured RTT and packet loss to each broker's endpoints, not by a claim of exchange colocation. Approved primary/secondary static egress addresses are prerequisites for every execution route. Operator access is separate from application and database networks.

## Fifteen workers and their boundaries

| Worker | Placement/ownership | Input → output | Overload behavior |
|---|---|---|---|
| 1 Ingestion | Adapter I/O task per socket (Dhan ≤5, Upstox ≤2/5, Zerodha ≤3 sockets) | Frames → timestamped buffers | Reserved socket capacity; reconnect on irrecoverable overflow |
| 2 Decode/normalize | Adapter bounded decode workers; one decoder per wire format | Frames → typed observations | Quarantine malformed frames; quality event |
| 3 Instrument state | Dedicated core threads, stable hash shard | Observation → state version | Preserve ordering; age limit disables dependent execution |
| 4 Candles | Same shard ownership as instrument; one active accumulator per enabled interval updated in the same apply | State/trade observations → bar revisions for all intervals + geometry | Gap flag instead of fabricated completeness |
| 5 Sectors | Core aggregation owner per sector group | Changed contributions → sector snapshots | Coalesce deltas only with contribution accounting |
| 6 Stock ranking | Core ranking owner | Feature change → sorted index update | Bounded refresh queue with visible rank age |
| 7 Scanners | Dependency graph on shard/sector/chain workers | Feature changes → candidate evidence | Disable optional expensive nodes before essential risk |
| 8 Regime | Core scheduled aggregate owner | Breadth/trend/volatility → regime version | Stale regime prevents strategies requiring it |
| 9 Opportunity scoring | Core ranking owner | Candidate + context → opportunity revision | Expire stale candidates rather than queue indefinitely |
| 10 Risk | Single account actor per broker account; global exposure coordinator | Intent proposal → reservation/rejection | Always executes; no bypass |
| 11 Routing/execution | Fenced gateway per account inside its adapter process | Committed command → broker result | Priority budget; uncertain outcome held |
| 12 Order/position updates | Account actor | Stream/postback/snapshot/fill → ledger/risk revision | Never drop; new entries halt if persistence unavailable |
| 13 Persistence | Separate process/threads | Analytics batches and audit outbox → storage | Analytics spool bounded; ledger uses separate synchronous path |
| 14 Dashboard publishing | Control process; one publisher thread per client group reading immutable snapshots and per-view dirty sets | Latest state + account events → field-level view deltas on lanes P0–P3 with per-subscription sequences ([14](14-DASHBOARD-UX.md)) | Coalesce P2 display, preserve P0/P1, disconnect slow consumers with resync; never blocks workers 3–12 |
| 15 Health/telemetry | Dedicated lightweight collectors | Counters/spans → health snapshot | Sampling of diagnostics, not safety state |

All workers use bounded queues sized from peak rate × acceptable burst duration against the reference workload and memory budgets in [15](15-PERFORMANCE-ENGINEERING.md). Capacity is a resource parameter, not a universe limit. Every queue reports depth, oldest age, enqueue failures and shedding reason. Each domain has one owner; readers receive immutable snapshots or messages, never mutable shared objects. Nothing polls: every worker is driven by its input queue or a timer wheel (bar boundaries, lateness windows, chain cadence).

Option-chain assembly is a shared chain worker inside worker 7's dependency graph: for Dhan/Upstox it merges provider chain snapshots with streamed selected strikes; for Zerodha it builds the chain from the instrument master plus quote/stream fields and computes IV/Greeks locally ([04](04-BROKER-ADAPTER-CONTRACTS.md) §4, [10](10-EQUITY-FUTURES-OPTIONS.md)).

## Ordered event flow

~~~mermaid
flowchart TD
  F[Frame receipt monotonic time] --> N[Validate and normalize]
  N --> I[Resolve canonical instrument and source epoch]
  I --> Q{Data quality sufficient?}
  Q -- No --> X[Update health and invalidate dependents]
  Q -- Yes --> ST[Apply instrument state version]
  ST --> CF[Incremental candles and feature DAG]
  CF --> SC[Affected scanners only]
  CF --> SE[Sector contribution deltas]
  SE --> RG[Regime snapshot]
  SC --> OP[Candidate ranking and expiry]
  RG --> OP
  OP --> RK[Account risk and reservation]
  RK --> DI[Durable intent commit]
  DI --> LAST[Freshness ownership and session recheck]
  LAST --> SEND[Broker request write]
~~~

A decision records input versions: instrument mapping, feature state, bar revisions, sector, regime, ranking configuration, risk policy and compliance policy. Consistency means all required inputs satisfy the strategy's stated age/version rules, not that every instrument shares a global instantaneous snapshot.

## Process and state isolation

Adapter processes own broker credentials through narrowly scoped secret handles, sockets, raw decoding and broker-specific quotas. The core cannot issue arbitrary URLs; it sends typed commands. Execution gateway credentials cannot be used by analytical workers. Adapter crashes do not corrupt core memory. Separate market-data and execution channels within each adapter preserve order-update and protective-command capacity during feed bursts. Inbound HTTP postbacks/webhooks (Dhan postback, Upstox webhook, Zerodha postback) terminate in a small ingress process that validates, verifies the Zerodha checksum, hashes and forwards to the owning adapter; it never holds broker credentials.

Instrument state is assigned by a stable hash of canonical ID. Runtime shard migration occurs only in a controlled maintenance/rebalance transition: stop new strategy evaluation for affected instruments, drain the owner queue to a sequence watermark, transfer state/version, start the new owner and expose a new ownership epoch. Pending execution stays with the account actor. No uncontrolled work stealing mutates instrument state.

Sector aggregators store each constituent's latest contribution and subtract the prior contribution before adding the new one: O(1) sufficient-statistic updates and O(log N) ranked-index changes. Global regime inputs are periodically consolidated; no full-universe pass runs on any tick.

## Control plane and configuration

The control API handles authenticated dashboard queries, strategy/risk configuration, broker session status and manual intents. Every change creates a new immutable configuration version with actor/reason/time and validation results. Scanner/ranking settings activate at a defined event boundary; execution-sensitive changes activate only after account-actor acknowledgement. Unknown enum or unsupported capability fails validation.

Read endpoints never expose secrets. Manual order requests include a client idempotency key and expected account/instrument versions; server idempotency prevents duplicate local intent creation. It cannot promise broker idempotency after uncertain network transmission.

## Recovery and failover

~~~mermaid
sequenceDiagram
  participant H as Feed health
  participant C as Core
  participant B as Alternate data adapter
  participant A as Account OMS
  H->>C: Active source stale
  C->>C: Invalidate dependent opportunities
  C->>B: Request equivalent mapped coverage
  B-->>C: Snapshot and live observations
  C->>C: Verify mapping, phase, age and units
  C->>C: Increment source epoch; reset cumulative baselines
  C->>C: Warm features and publish coverage
  C-->>A: New source quality/version
  A->>A: Existing orders remain on original account
~~~

Source switching is per instrument or coherent dependency group among the three adapters, ranked by configured preference, measured freshness and field coverage (for example an Upstox LTPC-only subscription cannot replace a Dhan full-mode source for a volume scanner). Cross-provider price differences, missing OI/volume and stale chain snapshots are visible. A feature requiring a continuous window remains warming until repaired or rebuilt. Stop monitoring may use an alternate validated source under a separate conservative policy; entry readiness is not inferred from emergency monitoring readiness.

On core restart: restore configuration/instruments and the durable account ledger; connect order streams into a buffer; obtain account snapshots; merge buffered updates using execution IDs/cumulative quantities; reconcile unresolved commands; then warm market features. Trading remains blocked until account and feed readiness gates pass. PostgreSQL WAL recovery is operational state recovery (C5).

## Failure containment and acceptance

Analytics failure increases persistence lag and can shed raw tick retention under the declared policy; it never invalidates a known durable account ledger. Ledger failure blocks new mutations, preserves broker-native protection and escalates unresolved exposure. An unavailable UI does not stop account protection. Loss of authenticated order updates on one broker requires conservative snapshot reconciliation within that broker's quotas and blocks entries on that account if state age exceeds policy; the other accounts continue.

Acceptance requires process crash isolation per adapter, bounded memory during sustained overload, no lost execution event, correct source-epoch reset on failover between any two adapters, no cross-account state contamination, and explicit readiness after restart. Synthetic packet faults and state-machine tests prove these properties. Load envelope and recovery-time limits are in [15](15-PERFORMANCE-ENGINEERING.md); account reconciliation detail is in [11](11-ORDER-EXECUTION-ENGINE.md).
