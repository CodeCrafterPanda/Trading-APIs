# Technology decisions and alternatives

Selected production stack: **Rust + TypeScript**, PostgreSQL, ClickHouse and object storage. Versions are pinned to supported stable releases at implementation kickoff and captured in deployment manifests. The language decision is closed; reopening it requires an ADR with measured evidence ([19](19-DECISIONS-AND-RISKS.md)).

## Language comparison

| Option | Strengths for this platform | Costs and limits | Decision |
|---|---|---|---|
| Rust core | Ownership, memory/thread safety, no tracing GC, fixed-point types, efficient binary parsing, zero-allocation hot paths | Higher implementation discipline, async/CPU separation, no official broker SDKs (Dhan/Upstox/Zerodha ship Python/Node/Java/.NET; Zerodha also Go) | Selected |
| Go core | Clear concurrency model, strong networking, official Zerodha SDK, deployment simplicity | GC assists/pauses and allocation must be measured at sub-ms tails | Credible alternative, not selected |
| Python core | Fast formula iteration, all three brokers' official SDKs | Interpreter/extension overhead and process boundaries complicate sub-ms tails | Excluded from live decision/OMS path |
| Rust + Go services | Specialized control plane possible | Extra runtime, schemas, deployment and cross-language debugging | Unnecessary initially |
| Rust + Python tooling | Independent formula verification (Greeks, scanners) | Must not become a trading dependency | Optional only |

Rust's ownership model supports zero-allocation per-tick processing but proves no latency number. Go's collector is mostly concurrent and is not inherently unsuitable for milliseconds; it is harder to hold at the sub-millisecond p99 budgets in [15](15-PERFORMANCE-ENGINEERING.md). CPython offers optional free-threaded builds; the choice rests on total runtime predictability, not an inaccurate GIL claim. Official SDKs are reference material for wire behavior only; the adapters implement the documented HTTP/WebSocket contracts directly. [Rust ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html), [Go GC guide](https://go.dev/doc/gc-guide), [Python free-threading](https://docs.python.org/3/howto/free-threading-python.html), [Kite SDKs](https://kite.trade/docs/connect/v3/), [Upstox SDKs](https://upstox.com/developer/api-documentation/installing-sdk/), [Dhan SDK](https://github.com/dhan-oss/DhanHQ-py).

## Concrete implementation selections

| Layer | Selection | Constraint |
|---|---|---|
| Rust networking | Tokio asynchronous runtime; separate pinned CPU/state worker threads | No scanner work on socket tasks |
| HTTP/control | Axum typed HTTP service, OpenAPI 3.1 contract | Release pinned and license/security reviewed at kickoff |
| HTTP client/TLS | reqwest with rustls; tokio-tungstenite for WebSocket | Explicit per-endpoint timeouts; mutation retries disabled by default |
| Upstox binary | prost-generated bindings from the pinned official descriptor (post-Oct-2024 fields: `bidQ/askQ/vol` int64) | Unknown-field policy explicit; descriptor hash in evidence |
| Dhan binary | Checked little-endian length-prefixed decoding; separate ordinary and depth decoders | No unchecked casts to native structs |
| Zerodha binary | Checked big-endian decoding by packet length (ltp/quote/full/index layouts), text frames parsed as JSON order postbacks | Price divisor per segment from master; no assumption of ÷100 everywhere |
| Internal event types | Shared Rust contract crate; fixed-scale prices and typed IDs | Broker wire fields never leak into strategy interfaces |
| Interprocess transport | Versioned length-delimited local socket frames (Unix domain sockets) | Bounded buffers, negotiated schema, disconnect semantics |
| Serialization | Binary typed IPC; JSON only at control/UI/broker REST boundaries | No JSON between in-process workers |
| Options math | Platform `greeks` crate: Black-76 on futures/forward price, bounded IV solver (bisection then Newton), convergence flags | Used for Zerodha, cross-check for others; fixtures from provider values |
| Database access | sqlx for PostgreSQL with checked queries and bounded pools | No query inside tick/feature processing; dedicated ledger pool |
| Backend metrics | Prometheus counters/histograms (10 µs buckets below 1 ms) and OpenTelemetry traces | Sampled spans; unsampled safety counters |
| UI | TypeScript/React/Vite, external-store selectors, virtualized grids | Stable row IDs, immutable snapshots, bounded subscriptions |
| Charts | TradingView Lightweight Charts behind a platform chart adapter | Verify license/attribution and release at kickoff |
| Testing | Rust unit/property/fuzz tests, three wire-format fixture suites, fault injection; browser interaction/performance tests | No brokerage writes in routine CI |

These are selected packages, not an open shortlist. Exact releases are a dependency-lock task. The chart adapter must support streaming candle revisions, gaps, multiple timeframes, source badges and an accessible companion table; heatmaps use a separate canvas renderer. If licensing or a required capability fails verification, record an ADR before replacing a library.

## Storage comparison and decision

PostgreSQL stores instruments, rule versions, credential metadata, account ownership, intents, commands, reservations, executions and the audit outbox. The intent commit is deliberately on the execution path because the external action must remain known after a crash; asynchronous commit can acknowledge before permanent storage and is unsuitable. [PostgreSQL commit durability](https://www.postgresql.org/docs/current/wal-async-commit.html)

ClickHouse stores large analytical observations, candles, features, chain snapshots, scanner evidence and ranking histories asynchronously. TTL is background-merge driven, so retention jobs monitor actual bytes/age. It is never the authority for account positions. [MergeTree](https://clickhouse.com/docs/reference/engines/table-engines/mergetree-family/mergetree)

Parquet in object storage holds compressed analytical exports and immutable audit packages. Object Lock is a technical mechanism, not legal certification. [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)

Redis is unnecessary for the state path: a network cache does not replace single-owner in-memory state or PostgreSQL durability. Kafka is deferred: the initial topology does not serialize every tick through a distributed log. Introducing either later requires measured multi-host fan-out needs and a design that keeps execution truth in the ledger. PostgreSQL-only analytics is rejected because tick/depth scans would compete with transactional latency.

## Queue and concurrency policy

Bounded queues with explicit capacities and ownership everywhere; Tokio bounded MPSC provides backpressure, unbounded queues are prohibited. Capacity derives from measured peak rates, tolerated burst and event size. [Tokio channels](https://docs.rs/tokio/latest/tokio/sync/mpsc/)

SPSC rings are used on the adapter→shard and shard→sector paths where the single-producer invariant is enforceable; elsewhere bounded MPSC. Instrument state updates are serial per shard; independent shards parallelize. A small fixed worker set owns CPU resources; spawning a task per tick is prohibited ([15](15-PERFORMANCE-ENGINEERING.md) §rules).

Account risk and OMS use one writer per broker account. A global exposure coordinator preallocates bounded account budgets; accounts reserve locally. Budget rebalancing is serialized and durable. Cash balances never merge across brokers. Database locks coordinate actors but do not fence an old process holding a broker token; gateway egress fencing remains mandatory. [PostgreSQL locking](https://www.postgresql.org/docs/current/explicit-locking.html)

## Browser and delivery decisions

Browser WebSocket has no application backpressure; the server coalesces market display updates, bounds buffers, disconnects persistently slow consumers and offers snapshot resync. Order/risk updates use priority delivery and durable read models. [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

Selector-based external stores with stable immutable snapshots keep unrelated rows from rerendering. [React external stores](https://react.dev/reference/react/useSyncExternalStore)

Initial deployment uses system-managed/containerized processes on Linux with CPU pinning, resource limits and an automated immutable release. Kubernetes is not required for one active host plus standby. Releases stop new entries, drain or preserve durable pending state, confirm native protection and reconcile before resuming.

## Acceptance and change control

Pin compiler, Protobuf descriptor, dependencies and container base by digest; produce a dependency/license inventory. Measure decoding, allocation rate, queue age, commit latency, UI frame time and total decision distributions under the declared workload. A language, queue or database change must preserve behavioral contracts and improve a measured bottleneck; isolated microbenchmarks are insufficient. [15](15-PERFORMANCE-ENGINEERING.md) defines the benchmark contract; [19](19-DECISIONS-AND-RISKS.md) records ADRs and change triggers.
