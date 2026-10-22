# Data ownership, schemas and retention

PostgreSQL is authoritative for execution/configuration; in-memory state is authoritative for current computed market views within a running source epoch; ClickHouse is analytical history; object storage is archive. No query to analytical storage is required to process a tick or pass a normal risk check.

## Entity relationships

~~~mermaid
erDiagram
  INSTRUMENT ||--o{ BROKER_MAPPING : maps
  INSTRUMENT ||--o{ SECTOR_MEMBERSHIP : belongs
  SECTOR ||--o{ SECTOR_MEMBERSHIP : contains
  ACCOUNT ||--o{ ORDER_INTENT : owns
  ORDER_INTENT ||--o{ ORDER_COMMAND : attempts
  ORDER_INTENT ||--o{ BROKER_ORDER : creates
  BROKER_ORDER ||--o{ EXECUTION : fills
  ACCOUNT ||--o{ POSITION : holds
  INSTRUMENT ||--o{ POSITION : identifies
  ORDER_INTENT ||--o{ RESERVATION : reserves
  ACCOUNT ||--o{ ACCOUNT_ALLOCATION : receives
  ACCOUNT ||--o{ RECONCILIATION_RUN : verifies
  CONFIG_VERSION ||--o{ ORDER_INTENT : governs
  ORDER_INTENT ||--o{ AUDIT_EVENT : explains
~~~

## PostgreSQL schema contract

| Entity/key | Principal columns | Constraints/indexes |
|---|---|---|
| instrument / instrument_id | canonical identity, asset class, underlying, expiry, strike atoms/scale, right, ISIN/series | Unique economic key within validity interval; venue preserved |
| broker_mapping / broker+segment+raw_id+master_version | instrument_id, raw/normalized units, valid dates, evidence | No overlapping active ambiguous mapping; lookup by broker/raw ID |
| master_version / version_id | source URL/hash, fetched_at, schema/normalizer versions, counts, quarantine | Immutable artifact reference |
| sector / sector_id | namespace, benchmark, methodology/license evidence | Unique namespace/name version |
| sector_membership / sector+instrument+effective_from | weight, effective_to, source version | No overlapping duplicate membership |
| account / account_id | broker, external identity hash, segment/profile refs | Broker account identity unique; no secret value |
| credential_metadata / account+generation | secret_ref, origin/class, expiry, scopes, status | Generation monotonic; one active execution generation |
| config_version / config_id+version | validated JSON document, hash, author/reason, effective time | Immutable after activation |
| account_lease / account_id | owner_epoch, gateway_id, lease_until, fencing evidence | Single active owner; epoch strictly increases |
| account_allocation / account+dimension+version | amount, encumbered, transfer state | Sum allocations≤global limit; serialized transfers |
| order_intent / intent_id | client key, payload hash, account/instrument/product, units/prices, UTC expiry, versions | Unique(account,client_key); immutable economic payload revision |
| order_command / command_id | intent, type, attempt, owner epoch, dispatch/outcome, correlation, payload hash | One active conflicting command/child; unique correlation scope |
| broker_order / account+broker_order_id | parent intent, total/cumulative/remaining, terms, status/raw status | Nonnegative quantities; cumulative≤total unless quarantined correction |
| execution / account+broker_trade_id | order, units, price, exchange/received time, evidence | Dedup unique trade identity; fallback fingerprint index |
| reservation / reservation_id | account/intent/child, dimension amounts, status, generation | Conservation checks; unknown outcome cannot release |
| position / account+instrument+product | net units/cost, realized P&L, reconciliation version | Account/product segregation |
| protection / protection_id | position scope, native IDs, desired/confirmed units, stop/target/trail, state | One exit controller ownership token |
| reconciliation_run / run_id | account, snapshots, watermarks, diff, status | Unresolved diff indexed by account/instrument |
| audit_event / event_id | actor, action, entity, before/after hashes, reason, trace, UTC time | Append-only; transactional outbox linkage |

Prices/money use checked fixed-point integers or NUMERIC with explicit scale; sums use wider accumulators. Foreign keys and uniqueness enforce identity/idempotency. Raw broker payloads are encrypted diagnostic objects with redaction, not unbounded JSON in hot rows.

## Event and analytical tables

Event envelope includes schema_version, event_id, kind, entity_id, source, source_epoch, local_sequence, UTC event/receipt time, clock_domain, host_boot_id, monotonic diagnostic times, quality, config versions and payload. Monotonic values are comparable only within the same clock domain/boot; durable deadlines use expires_at_utc.

| ClickHouse dataset | Sort key / partition | Data |
|---|---|---|
| market_observations | instrument_id, received_at, source_epoch / trading date | Selected normalized ticks, field masks and quality |
| depth_snapshots | instrument_id, received_at / trading date | Mode/depth levels and age; sampled retention |
| candles | instrument_id, interval, origin, open_time, revision / month | OHLCV/OI, phase, quality, child digest |
| features | instrument_id, feature_id, observed_at / date | Value, basis, input version, readiness |
| scanner_events | scanner_id, instrument_id, observed_at / date | Candidate revisions and evidence |
| opportunity_versions | opportunity_id, revision / date | Components, feasibility, expiry/invalidation |
| telemetry_rollups | service, metric, bucket / date | Histogram/count/health summaries |

Do not rely on background merge deduplication for immediate account correctness. Analytical queries select latest revision explicitly. Partition by day for high-volume data and month for candles; avoid a partition per instrument. Batch inserts by size/time, initially 10,000 rows or 100 ms, whichever first, then tune from measurement.

## In-memory state and checkpoints

Instrument shards store dense arrays/maps of prices, field ages, feature states, bar accumulators and bounded rolling windows. Sector workers retain constituent contributions and sufficient statistics. Account actors cache reconciled ledger state, funds snapshots, allocations and reservations. Immutable snapshots are versioned; raw mutable pointers never cross ownership boundaries.

Market checkpoints accelerate restart but do not establish fresh data. On restore, mark stale, validate master/config versions and warm from live/history. Account recovery uses durable PostgreSQL and broker reconciliation. Persisting a runtime monotonic deadline cannot extend an intent after reboot; reconstruct from UTC under clock-health policy or expire it.

## Retention policy and sizing

These are proposed operational defaults, subject to data rights and applicable obligations:

| Dataset | Hot retention | Archive/default action |
|---|---|---|
| Instruments/rules/mappings | All active + versions referenced by audit | Retain referenced versions with audit lifetime |
| One-minute/daily candles | Five years where licensed/available | Compressed Parquet; longer only with policy |
| Raw/normalized ticks | 7 days | Up to 30 days compressed if entitled; then delete |
| Full depth | 24 hours | Selected incident samples 30 days; otherwise delete |
| Features/scanner/rank revisions | 30 days | Daily/incident summaries 1 year |
| Orders/commands/fills/risk/audit | Five-year proposed minimum | Encrypted immutable exports, legal hold support |
| Detailed telemetry | 14 days | Minute/hour rollups 90 days/1 year |

Five-year broker audit obligations in exchange standards do not automatically determine every private application's legal duty; confirm the applicable policy. ClickHouse TTL executes through background merges, so monitor actual oldest retained data and storage. [NSE standards](https://nsearchives.nseindia.com/content/circulars/INVG67858.pdf), [ClickHouse TTL](https://clickhouse.com/docs/reference/engines/table-engines/mergetree-family/mergetree).

Sizing formula: daily_bytes = mean_events_per_second × active_seconds × average_encoded_bytes × compression_ratio + indexes/metadata. Illustrative only: 20,000 events/s × 24,000s × 96 bytes = 46.08 GB raw/day; at 0.30 compressed ratio ≈13.82 GB/day before overhead. Depth, replicas and backups are additional. This rate is not measured broker traffic or a platform cap. Establish real rates during read-only commissioning and set capacity alarms at 60/75/85% with retention actions.

## Durability, backup and restore

Use synchronous commit for intents, commands, reservations, executions and audit linkage. Keep asynchronous analytics outbox separate from ledger commit capacity. Backups include encrypted base backups, WAL continuity, restore tests and secret-independent configuration exports. State explicitly whether deployment durability covers process, host or availability-zone failure; local fsync alone is not zero-loss protection against destroyed storage.

Audit export includes source hashes, decisions, risk results, commands, responses, fills and reconciliation evidence. Redact tokens/PINs and minimize personal data. Object retention/legal hold settings follow policy and are tested for both recovery and authorized deletion.

Acceptance verifies constraints, atomic reservation/intent writes, duplicate fills, schema migration rollback, retention partitions, bounded spool behavior and full restore followed by broker reconciliation. Analytical storage failure must not corrupt execution truth; ledger loss must never be concealed by reconstructing a guessed position from market history.
