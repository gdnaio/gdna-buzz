## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Integrations

### Declared dependencies (`crates/buzz-audit/Cargo.toml:11-22`)

| Crate | Declared line | Workspace version (`Cargo.toml`) | Actually used in `src`? | Where |
|---|---|---|---|---|
| `buzz-core` | `Cargo.toml:11` | path `crates/buzz-core` (`Cargo.toml:130`) | Yes — `CommunityId` | `entry.rs:1`, `service.rs:7` |
| `sqlx` | `:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (`Cargo.toml:52-54`) | Yes | `service.rs:3` (`Acquire, PgPool, Row`), `:84`, `:238` |
| `tokio` | `:13` | `1` (`Cargo.toml:43`) | **Tests only** | `service.rs:265` (`tokio::sync::Mutex`), `#[tokio::test]` at `:318,338,376,437,475,512` |
| `serde` | `:14` | `1` + derive (`Cargo.toml:64`) | Yes — derives | `action.rs:1,6-7`, `entry.rs:3,13` |
| `serde_json` | `:15` | `1` (`Cargo.toml:69`) | Yes | `entry.rs:34`, `hash.rs:80-116`, `error.rs:40` |
| `uuid` | `:16` | `1` + v4, serde (`Cargo.toml:89`) | Yes | `entry.rs:4,16`, `service.rs:5,246`, `hash.rs:45` (`Uuid::as_bytes`) |
| `chrono` | `:17` | `0.4` + serde (`Cargo.toml:90`) | Yes | `entry.rs:2,36`, `hash.rs:1,22-24`, `service.rs:1,21` |
| `tracing` | `:18` | `0.1` (`Cargo.toml:74`) | Yes | `service.rs:4` (`debug, instrument, warn`), `:52`, `:128`, `:159`, `:211`, `:241` |
| `thiserror` | `:19` | `2` (`Cargo.toml:85`) | Yes | `error.rs:1,11` |
| `sha2` | `:20` | `0.11` (`Cargo.toml:96`) | Yes | `hash.rs:2` (`Digest, Sha256`), `:43`, `:72` |
| `hex` | `:21` | `0.4` (`Cargo.toml:97`) | **No** — grep for `hex::` in `crates/buzz-audit/src` returns nothing | unused declaration |
| `futures-util` | `:22` | `0.3` (`Cargo.toml:110`) | Yes — `FutureExt::catch_unwind` | `service.rs:2`, `:68` |

No `[features]`, `[dev-dependencies]`, or `[build-dependencies]` sections exist in
`crates/buzz-audit/Cargo.toml`.

### Postgres / sqlx usage

- Connection handle is a `PgPool` held by value in `AuditService` (`service.rs:37-45`).
  The crate never creates the pool; the relay passes one in
  (`crates/buzz-relay/src/main.rs:321-330`, a dedicated pool with
  `max_connections(5)`, `min_connections(1)`).
- Write path pins a single pooled connection: `self.pool.acquire()` (`service.rs:54`),
  used for the lock (`:59-62`), the transaction (`:87`), and the unlock (`:71-74`) —
  necessary because advisory locks are session-scoped (`service.rs:49-51`).
- Transaction via `Acquire::begin` (`service.rs:87`) and `commit` (`:149`). No explicit
  rollback call; a dropped `Transaction` rolls back implicitly on the error paths
  (`service.rs:101`, `:126`, `:147`).
- Reads run directly on `&self.pool` with `fetch_all` (`service.rs:178`, `:231`).
- All statements are untyped `sqlx::query` with bind parameters (no `query!`/`query_as!`
  macros), so there is **no compile-time schema verification** and column access is
  runtime `Row::get` (`service.rs:105-106`, `:246-254`).
- Postgres-specific SQL used: `pg_advisory_lock`, `pg_advisory_unlock`,
  `hashtextextended($1, 0)` (`service.rs:59`, `:71`). `hashtextextended` is a Postgres
  internal hash function — this crate is not portable to other engines.
- Types crossing the boundary: `UUID`↔`Uuid`, `BIGINT`↔`i64`, `BYTEA`↔`Vec<u8>`,
  `VARCHAR`↔`String`/`&str`, `JSONB`↔`serde_json::Value`, `TIMESTAMPTZ`↔`DateTime<Utc>`
  (binds at `service.rs:137-145`; decodes at `:246-254`).

### Cryptography

`sha2::Sha256` only, used incrementally (`Sha256::new`, `update`, `finalize`) in
`compute_hash` (`hash.rs:43-72`). No HMAC, no signatures, no randomness, no key
material anywhere in the crate.

### Non-Postgres I/O

None. The crate performs no filesystem, network, HTTP, S3, Redis, or process I/O.
The only environment read is `DATABASE_URL` inside a test helper
(`service.rs:275-279`). Logging goes through `tracing` macros only.

### How the relay integrates it (fire-and-forget semantics)

| Step | Location | Behaviour |
|---|---|---|
| Construction gate | `crates/buzz-relay/src/main.rs:321-334` | `AuditService` built only when `config.audit_enabled`; otherwise `None` and an info log |
| Enabled gauge | `crates/buzz-relay/src/main.rs:139` | `buzz_audit_enabled` set to 1.0/0.0 |
| Queue | `crates/buzz-relay/src/state.rs:654` | bounded `mpsc::channel::<NewAuditEntry>(1000)`; `audit_tx: Option<Sender<...>>` (`state.rs:555`) |
| Producer (events) | `crates/buzz-relay/src/handlers/event.rs:540-577` | uses `send().await` (backpressure, not drop); on closed channel logs `error!` and increments `buzz_audit_send_errors_total` (`:575-577`) |
| Producer (media) | `crates/buzz-relay/src/api/media.rs:422-442` | same pattern; upload still returns `Ok` even if the audit send fails (`media.rs:443`) |
| Worker | `crates/buzz-relay/src/state.rs:657-690` | single task; select over `recv()` and a `CancellationToken`; on cancel closes the receiver and drains buffered entries |
| Failure handling | `crates/buzz-relay/src/state.rs:1199-1207` | `audit.log(entry)` error → `buzz_audit_log_errors_total` + `tracing::error!`; **no retry, no dead-letter**; success → `buzz_audit_log_seconds` histogram |
| Shutdown | `crates/buzz-relay/src/state.rs:632-636`, `:680-689` | `AuditShutdownHandle::drain()` flushes queued entries; a timeout path logs "Audit worker did not drain in time — exiting anyway" (`state.rs:1190-1191`) |

Consequences visible in code: a DB outage causes queued entries to be lost after one
failed attempt (`state.rs:1201-1203`), and because `log` is the only path that assigns
`seq`, a lost entry leaves **no gap** in the chain — the next successful append simply
takes the next `seq`. The chain therefore stays verifiable while being incomplete; the
crate offers no way to detect that an entry was dropped.

### Other repo touch points

- `crates/buzz-admin/Cargo.toml:20` declares the dependency, but grep for `audit` in
  `crates/buzz-admin/src` returns nothing — no operator CLI surface consumes
  `verify_chain`/`get_entries` today.
- `migrations/0023_push_match_gate.sql:21` references the `'buzz_audit:'` advisory-lock
  namespace in a comment about lock families.
- `crates/buzz-test-client/tests/conformance_multitenant.rs:2665-2710` documents that
  audit is deliberately *not* black-box testable over the wire and defers to this
  crate's own tests.
- `crates/buzz-conformance/Cargo.toml:19` explicitly excludes `buzz-audit` from the
  conformance checker's dependency set.
