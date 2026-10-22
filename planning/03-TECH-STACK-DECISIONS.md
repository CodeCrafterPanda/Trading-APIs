# Technology decisions and alternatives

The selected production stack is **Rust + TypeScript**, PostgreSQL, ClickHouse and object storage. Versions are pinned to supported stable releases at implementation kickoff and captured in deployment manifests. This document makes the architectural choice; implementation does not reopen the language decision without an explicit ADR and evidence.

## Language comparison

| Option | Strengths for this platform | Costs and limits | Decision |
|---|---|---|---|
| Rust core | Ownership, memory/thread safety, no tracing GC, fixed-point types, efficient binary parsing | Higher implementation discipline, async/CPU separation, smaller official broker SDK coverage | Selected |
| Go core | Clear concurrency model, strong networking, deployment simplicity | Allocation, scheduling, GC assists and pauses must be measured at tails | Credible alternative, not selected |
| Python core | Fast formula iteration, broker SDK availability | Interpreter/extension overhead and process boundaries complicate predictable tails | Excluded from live decision/OMS path |
| Rust + Go services | Specialized control plane possible | Extra runtime, schemas, deployment and cross-language debugging | Unnecessary initially |
| Rust + Python tooling | Useful independent formula verification | Tooling must not become mandatory trading dependency | Optional only |

Rust's ownership model supports these design goals but proves no application latency number. Go's collector is largely concurrent and is not inherently unsuitable for milliseconds. Modern CPython has optional free-threaded builds; the choice is based on total runtime predictability and system complexity, not the inaccurate claim that every Python build has a GIL. [Rust ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html), [Go GC guide](https://go.dev/doc/gc-guide), [Python free-threading](https://docs.python.org/3/howto/free-threading-python.html).

## Concrete implementation selections

| Layer | Selection | Constraint |
|---|---|---|
| Rust networking | Tokio asynchronous runtime; separate CPU/state worker threads | No heavy scanner work on socket tasks |
| HTTP/control | Axum typed Rust HTTP service, OpenAPI 3.1 contract | Library release pinned and license/security reviewed at kickoff |
| HTTP client/TLS | reqwest with rustls; tokio-tungstenite for WebSocket | Timeouts explicit per endpoint; mutation retries disabled by default |
| Upstox binary | prost-generated bindings from pinned official descriptor | Preserve unknown field compatibility policy; descriptor hash in evidence |
| Dhan binary | Checked explicit endian/length decoding | No unchecked casts to native structs |
| Internal event types | Shared Rust contract crate; fixed-scale prices and typed IDs | Broker wire fields never leak into strategy interfaces |
| Interprocess transport | Versioned length-delimited local socket frames | Bounded buffers, negotiated schema, disconnect semantics |
| Serialization | Binary typed IPC; JSON control/UI boundaries | No JSON encode/decode between every in-process worker |
| Database access | sqlx for PostgreSQL with checked queries and bounded pools | No query inside tick/feature processing; dedicated ledger pool |
| Backend metrics | Prometheus counters/histograms and OpenTelemetry Rust traces | Sample detailed spans; preserve safety counters |
| UI | TypeScript/React/Vite, external store selectors, virtualized grids | Stable row IDs, immutable snapshots, bounded subscriptions |
| Charts | TradingView Lightweight Charts behind a platform chart adapter | Verify current license/attribution and supported release at kickoff |
| Testing | Rust unit/property/fuzz tests, contract fixtures, fault injection; browser interaction/performance tests | No brokerage writes in routine CI |

These are selected packages, not an open library shortlist. Exact releases are a dependency-lock task. The chart adapter must support streaming candle revisions, gaps, multiple timeframes, source badges and an accessible companion data table. Heatmaps use a separate canvas renderer. If licensing or a required capability fails verification, record an ADR before replacing the selected library; do not silently switch architecture.

## Storage comparison and decision

PostgreSQL stores instruments, rule versions, credentials metadata, account ownership, intents, commands, reservations, executions and audit outbox. It provides transactional constraints and synchronous durable commit. The intent commit is deliberately on the execution path because the external action must remain known after a crash. Asynchronous commit can acknowledge data before permanent storage and is unsuitable for this invariant. [PostgreSQL commit durability](https://www.postgresql.org/docs/current/wal-async-commit.html).

ClickHouse stores large analytical observations, candles, features, scanner evidence and ranking histories asynchronously. Sorting and partitioning support instrument/time queries. TTL is background-merge driven, so retention jobs monitor actual bytes/age rather than assuming instantaneous deletion. It is not the authority for account positions. [MergeTree](https://clickhouse.com/docs/reference/engines/table-engines/mergetree-family/mergetree).

Parquet plus object storage holds compressed analytical exports and immutable audit packages. Object immutability and retention settings are selected after applicable rights and obligations are established. Object Lock is a technical mechanism, not legal certification. [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html).

Redis is unnecessary for the initial state path: an extra network cache does not replace single-owner in-memory state or PostgreSQL durability. Kafka is deferred because the initial topology does not need every tick serialized through a distributed log. Introducing either later requires measured multi-host fanout/recovery needs and a design that keeps execution truth in the ledger. PostgreSQL-only analytics would simplify operations but large tick/depth scans compete with transactional latency; a separate analytical store is selected.

## Queue and concurrency policy

Use bounded queues with explicit capacities and ownership. Tokio offers bounded MPSC backpressure; unbounded queues can consume all available memory. Capacity planning derives from measured peak event rates, tolerated bursts and event size. [Tokio channels](https://docs.rs/tokio/latest/tokio/sync/mpsc/).

Use single-producer/single-consumer rings only where profiling demonstrates a meaningful benefit and the producer/consumer invariant is enforceable. Do not introduce lock-free structures everywhere. Instrument state updates are serial per shard; independent shards parallelize. Sector/candidate messages carry versions. A small fixed worker set owns CPU resources; spawning a task per tick is prohibited.

Account risk and OMS use one writer per account. A global exposure coordinator preallocates bounded account risk budgets; accounts reserve locally within their allocations. Budget rebalancing is serialized and durable, with the old allocation reduced and acknowledged before the new allocation is increased. Cash balances never merge. Database locks help coordinate actors, but do not fence an old process holding broker credentials. Gateway egress fencing remains mandatory. [PostgreSQL locking](https://www.postgresql.org/docs/current/explicit-locking.html).

## Browser and delivery decisions

Browser WebSocket has no built-in application backpressure. The server must coalesce market display updates, bound buffers, disconnect persistently slow consumers and offer snapshot resync. Orders/risk updates use priority delivery and durable read models; their correctness cannot depend on browser delivery. [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket).

Use selector-based external stores with stable immutable snapshots so unrelated rows do not rerender. React's external-store API provides the subscription mechanism; it does not automatically solve high-frequency rendering. [React external stores](https://react.dev/reference/react/useSyncExternalStore).

Initial deployment uses system-managed/containerized processes on Linux, resource limits and an automated immutable release. Kubernetes is not required for one active host plus standby. Database backup/restore, egress configuration and secret rotation are separate runbooks. Releases stop new entries, drain or preserve durable pending state, confirm native protection and reconcile before resuming.

## Acceptance and future change control

Pin compiler, schema descriptor, dependencies and container base by digest. Produce a dependency/license inventory. Measure decoding, allocation rate, queue age, commit latency, UI frame time and total decision distributions under the same declared workload. A language, queue or database change must preserve behavioral contracts and improve a measured bottleneck; isolated microbenchmarks are insufficient.

No technology selection creates a promise of exchange speed or profitability. The design favors understandable ownership and failure behavior over an unmeasured claim of maximum throughput. [15](15-PERFORMANCE-ENGINEERING.md) defines the benchmark contract; [19](19-DECISIONS-AND-RISKS.md) records ADRs and change triggers.
