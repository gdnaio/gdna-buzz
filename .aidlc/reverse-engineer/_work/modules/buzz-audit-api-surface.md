## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: API Surface

Crate root declares 5 public modules — `action`, `entry`, `error`, `hash`, `service`
(`crates/buzz-audit/src/lib.rs:21-29`) — with re-exports at `lib.rs:31-35`.
Lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`lib.rs:1-2`).

### Public functions and methods

| Symbol | Signature (file:line) | Operation | Tables touched | Return type |
|---|---|---|---|---|
| `AuditService::new` | `pub fn new(pool: PgPool) -> Self` (`service.rs:43-45`) | stores the pool; no I/O | none | `AuditService` |
| `AuditService::log` | `pub async fn log(&self, entry: NewAuditEntry) -> Result<AuditEntry, AuditError>` (`service.rs:53`) | acquire pooled conn → `pg_advisory_lock` → `log_inner` under `catch_unwind` → `pg_advisory_unlock` → resume panic if any | `audit_log` (via `log_inner`); advisory-lock functions are not table reads | `Result<AuditEntry, AuditError>` |
| `AuditService::verify_chain` | `pub async fn verify_chain(&self, community: CommunityId, from_seq: i64, to_seq: i64) -> Result<bool, AuditError>` (`service.rs:160-165`) | one `SELECT` of the range, then per-row link check + hash recompute | `audit_log` (read) | `Result<bool, AuditError>` |
| `AuditService::get_entries` | `pub async fn get_entries(&self, community: CommunityId, from_seq: i64, limit: i64) -> Result<Vec<AuditEntry>, AuditError>` (`service.rs:212-217`) | one `SELECT` with `LIMIT`, decode each row | `audit_log` (read) | `Result<Vec<AuditEntry>, AuditError>` |
| `compute_hash` | `pub fn compute_hash(entry: &AuditEntry) -> Result<[u8; 32], AuditError>` (`hash.rs:42`) | pure SHA-256 over the fixed pre-image | none | `Result<[u8; 32], AuditError>` |
| `to_storage_precision` | `pub fn to_storage_precision(created_at: DateTime<Utc>) -> DateTime<Utc>` (`hash.rs:22-24`) | `trunc_subsecs(6)` | none | `DateTime<Utc>` |
| `GENESIS_HASH` | `pub const GENESIS_HASH: [u8; 32]` (`hash.rs:9`) | constant | none | `[u8; 32]` |
| `AuditAction::as_str` | `pub fn as_str(&self) -> &'static str` (`hash-independent`, `action.rs:35-49`) | variant → stable string | none | `&'static str` |
| `impl Display for AuditAction` | `fmt(&self, f) -> fmt::Result` (`action.rs:66-70`) | writes `as_str()` | none | `fmt::Result` |
| `impl FromStr for AuditAction` | `fn from_str(s: &str) -> Result<Self, String>` (`action.rs:75-81`) | linear search over `ALL` | none | `Result<AuditAction, String>` |

Public **types**: `AuditEntry`, `NewAuditEntry` (`entry.rs:14`, `:52`), `AuditAction`
(`action.rs:8`), `AuditError` (`error.rs:12`), `AuditService` (`service.rs:37`).
`AuditService` has no `Clone`/`Debug` derive (`service.rs:37-39`).

### Private helpers

| Symbol | Signature (file:line) | Purpose |
|---|---|---|
| `AuditService::log_inner` | `async fn log_inner(&self, conn: &mut PoolConnection<Postgres>, entry: NewAuditEntry) -> Result<AuditEntry, AuditError>` (`service.rs:82-86`) | the transactional head-read + insert |
| `row_to_audit_entry` | `fn row_to_audit_entry(row: &sqlx::postgres::PgRow) -> Result<AuditEntry, AuditError>` (`service.rs:238-256`) | row → `AuditEntry`, parsing `action` |
| `canonical_json` | `fn canonical_json(value: &serde_json::Value) -> Result<String, serde_json::Error>` (`hash.rs:80-116`) | recursive sorted-key JSON serialization |
| `log_timestamp` | `fn log_timestamp() -> DateTime<Utc>` (`service.rs:21-23`) | `to_storage_precision(Utc::now())` |
| `AUDIT_LOCK_NAMESPACE` | `const AUDIT_LOCK_NAMESPACE: &str = "buzz_audit:"` (`service.rs:29`) | advisory-lock key prefix (private) |

### Exact SQL issued

| Call site | SQL | Binds |
|---|---|---|
| `service.rs:59-62` | `SELECT pg_advisory_lock(hashtextextended($1, 0))` | `$1` = `"buzz_audit:{community_id}"` |
| `service.rs:71-74` | `SELECT pg_advisory_unlock(hashtextextended($1, 0))` | same key; result discarded via `let _ =` |
| `service.rs:94-101` | `SELECT seq, hash FROM audit_log WHERE community_id = $1 ORDER BY seq DESC LIMIT 1` | `$1` = raw `Uuid`; `fetch_optional` inside the tx |
| `service.rs:130-147` | `INSERT INTO audit_log (community_id, seq, hash, prev_hash, action, actor_pubkey, object_id, detail, created_at) VALUES ($1..$9)` | 9 binds in column order (`service.rs:137-145`) |
| `service.rs:166-179` | `SELECT community_id, seq, hash, prev_hash, action, actor_pubkey, object_id, detail, created_at FROM audit_log WHERE community_id = $1 AND seq BETWEEN $2 AND $3 ORDER BY seq ASC` | community, `from_seq`, `to_seq`; run on `&self.pool` (no tx) |
| `service.rs:218-232` | same projection `... WHERE community_id = $1 AND seq >= $2 ORDER BY seq ASC LIMIT $3` | community, `from_seq`, `limit`; run on `&self.pool` |

All queries are `sqlx::query(...)` with bind parameters (no string interpolation of
values; the only formatted string is the advisory-lock key, which is bound as `$1`
at `service.rs:58-60`). `sqlx::query` (untyped) is used throughout — no
compile-time-checked `query!` macros, so column names are resolved at runtime via
`Row::get` (`service.rs:105-106`, `246-254`).

### Transaction / connection semantics

- `log` takes one pooled connection (`service.rs:54`) and holds it for the lock,
  transaction, and unlock — required because Postgres advisory locks are
  session-scoped (`service.rs:49-51`).
- `log_inner` opens the transaction with `conn.begin()` (`service.rs:87`, via
  `sqlx::Acquire` imported at `service.rs:3`) and commits at `service.rs:149`.
  Head read and insert are both inside that transaction (`service.rs:100`, `:146`).
- `verify_chain` / `get_entries` are non-transactional single statements against the
  pool (`service.rs:178`, `:231`) and take **no** advisory lock.

### Return-value contracts

- `log` returns the fully materialised `AuditEntry` including assigned `seq`,
  `prev_hash`, computed `hash`, and `created_at` (`service.rs:151`).
- `verify_chain` returns `Ok(false)` when the range is empty (`service.rs:181-183`) —
  an empty range is *not* an error; `Ok(true)` when the whole segment is internally
  consistent (`service.rs:205`); `Err(ChainViolation)` / `Err(HashMismatch)` on the
  first offending `seq` (`service.rs:193`, `:199`).
- `get_entries` collects `Result` per row, so a single unparsable `action` fails the
  whole call (`service.rs:234`).

### Observed consumers (outside this crate)

| Consumer | Call | Notes |
|---|---|---|
| `crates/buzz-relay/src/main.rs:321-334` | `AuditService::new(audit_pool)` behind `config.audit_enabled` | dedicated pool, `max_connections(5)`, `min_connections(1)` |
| `crates/buzz-relay/src/state.rs:654`, `:1199-1207` | `audit.log(entry)` from a background worker fed by an `mpsc` channel (capacity 1000) | errors counted (`buzz_audit_log_errors_total`) and logged, never retried |
| `crates/buzz-relay/src/handlers/event.rs:540-577` | builds `NewAuditEntry { action: EventCreated, … }` | `object_id` = event id hex; `detail` = `{event_kind, channel_id}` |
| `crates/buzz-relay/src/api/media.rs:422-442` | builds `NewAuditEntry { action: MediaUploaded, … }` | `object_id` = media sha256 |
| `crates/buzz-admin/Cargo.toml:20` | declares `buzz-audit` dependency | no `audit`/`AuditService` reference found in `crates/buzz-admin/src` (grep for `audit` returned nothing) |

`verify_chain` and `get_entries` have **no production caller** in the repo — the only
call sites are this crate's `#[ignore]` tests (`service.rs:368`, `:417`, `:427`, `:468`,
`:508`, `:523`) and relay tests (`crates/buzz-relay/src/handlers/event.rs:1906-1952`).
