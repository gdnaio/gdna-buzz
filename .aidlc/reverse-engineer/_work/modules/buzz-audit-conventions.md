## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Conventions

### Module organization

| File | Lines | Role | Declared in `lib.rs` |
|---|---|---|---|
| `src/lib.rs` | 35 | crate docs, lints, module declarations, re-exports | — |
| `src/action.rs` | 100 | `AuditAction` enum + string mapping + `FromStr`/`Display` | `lib.rs:21` |
| `src/entry.rs` | 72 | `AuditEntry` (stored) and `NewAuditEntry` (input) | `lib.rs:23` |
| `src/error.rs` | 108 | `AuditError` | `lib.rs:25` |
| `src/hash.rs` | 272 | `GENESIS_HASH`, `to_storage_precision`, `compute_hash`, `canonical_json` | `lib.rs:27` |
| `src/service.rs` | 527 | `AuditService` (append/verify/read) + row decode | `lib.rs:29` |

One concept per file; no `mod.rs` nesting, no `tests/` directory. Every module gets a
doc comment at its declaration site (`lib.rs:20-29`) in addition to its own file-level
items.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]`
(`lib.rs:1-2`). Public items are documented throughout (e.g. `action.rs:5`, `:9-30`,
`:34`; `entry.rs:8-12`, `:15-36`; `service.rs:31-36`, `:42`, `:47-51`).

Selective re-export at the root rather than a blanket `pub use`: `AuditAction`,
`AuditEntry`, `NewAuditEntry`, `AuditError`, `compute_hash`, `GENESIS_HASH`,
`AuditService` (`lib.rs:31-35`). `to_storage_precision` stays module-qualified
(`hash.rs:22`).

### Naming

- Types: `UpperCamelCase` — `AuditService`, `AuditEntry`, `NewAuditEntry`,
  `AuditAction`, `AuditError`. The `New*` prefix marks the pre-persistence input
  (`entry.rs:52`).
- Functions: `snake_case` verbs — `log`, `verify_chain`, `get_entries`, `compute_hash`,
  `canonical_json`, `to_storage_precision`, `log_timestamp`, `row_to_audit_entry`.
  `*_inner` marks the private continuation of a public wrapper (`service.rs:82`).
- Constants: `SCREAMING_SNAKE_CASE` — `GENESIS_HASH` (`hash.rs:9`),
  `AUDIT_LOCK_NAMESPACE` (`service.rs:29`), private `ALL` (`action.rs:51`).
- DB action strings are snake_case of the variant name (`action.rs:37-47`).
- Row-decode helper is a free function, not a `From` impl, because it must return
  `Result` (`service.rs:238`).
- Aliased import to avoid trait-name pollution: `use futures_util::FutureExt as _`
  (`service.rs:2`).

### Error handling

Single crate error enum `AuditError`, `thiserror`-derived (`error.rs:11-41`):

| Variant | Payload | `#[error]` message | Line | Constructed at |
|---|---|---|---|---|
| `Database` | `#[from] sqlx::Error` | `database error: {0}` | `error.rs:14-15` | implicitly by `?` on every sqlx call (`service.rs:54`, `:62`, `:101`, `:147`, `:149`, `:179`, `:232`) |
| `ChainViolation` | `{ seq: i64 }` | `hash chain integrity violation at seq {seq}: prev_hash does not match preceding entry` | `error.rs:19-25` | `service.rs:193` |
| `HashMismatch` | `{ seq: i64 }` | `hash mismatch at seq {seq}: stored hash does not match recomputed hash` | `error.rs:28-32` | `service.rs:199` |
| `UnknownAction` | none | `unknown audit action in database` | `error.rs:35-36` | `service.rs:242` |
| `Serialization` | `#[from] serde_json::Error` | `serialization error: {0}` | `error.rs:39-40` | via `?` on `canonical_json` (`hash.rs:67`) |

Conventions observed:
- `?` propagation everywhere in production paths; **no `unwrap()`/`expect()`/`panic!`/
  `unimplemented!()` outside `#[cfg(test)]`** (grep confirms all 38 hits are inside test
  modules, e.g. `service.rs:304`, `hash.rs:155`).
- `#[from]` used only for foreign error types (`error.rs:15`, `:40`); domain variants
  are constructed explicitly with named fields.
- Deliberate error-sanitization rule: no variant carries a `community_id` or constraint
  name, documented at `error.rs:3-10` and pinned by a test that asserts the rendered
  text contains neither the community UUID (both hyphenated and simple forms) nor the
  strings `community_id`, `audit_log_pkey`, `constraint`, `communities`
  (`error.rs:58-107`).
- One intentional error suppression: the advisory unlock result is discarded with
  `let _ = ...` (`service.rs:71`).
- `FromStr for AuditAction` uses `Err = String` (`action.rs:73`, message at `:80`); the
  service discards that string and substitutes `UnknownAction` after a `warn!`
  (`service.rs:240-243`).

### Tracing / observability conventions

- `#[instrument(skip(self, entry), fields(action = %entry.action))]` on `log`
  (`service.rs:52`) — the entry itself is skipped so `detail` never lands in a span.
- `#[instrument(skip(self))]` on `verify_chain` (`service.rs:159`) and `get_entries`
  (`service.rs:211`), so `community`/`seq` args *are* recorded.
- `debug!(seq, "writing audit entry")` (`service.rs:128`); `warn!("unknown action in
  audit log")` without the offending value (`service.rs:241`).
- No metrics emitted from this crate; counters/histograms live in the relay
  (`crates/buzz-relay/src/state.rs:1201-1206`).

### Documentation conventions

Doc comments carry rationale, not just description — e.g. why the pre-image order is
frozen (`hash.rs:28-30`), why timestamps are truncated (`hash.rs:11-21`), why
`NewAuditEntry` is not `Deserialize` (`entry.rs:46-51`), why the lock is per-community
(`service.rs:25-28`), why `detail` must not hold tokens (`entry.rs:64-71`). Intra-doc
links are used (`[`crate::hash::GENESIS_HASH`]` at `entry.rs:22`,
`[`AuditService::log_inner`]` at `service.rs:19`).

### Testing patterns

Counts across `crates/buzz-audit/src`:

| Metric | Count | Locations |
|---|---|---|
| `#[cfg(test)] mod tests` blocks | 4 | `action.rs:84`, `error.rs:43`, `hash.rs:118`, `service.rs:258` |
| `#[test]` (sync) | 13 | `action.rs:88,96`; `error.rs:58`; `hash.rs:152,159,167,184,201,216,226,256,266`; `service.rs:283` |
| `#[tokio::test]` (async) | 6 | `service.rs:318,338,376,437,475,512` |
| **Total tests** | **19** | — |
| `#[ignore = "requires Postgres"]` | 6 | `service.rs:319,339,377,438,476,513` — exactly the 6 async tests |
| Tests runnable without infra | 13 | all `#[test]` |

Patterns:
- Fixture builders instead of literals: `sample_entry()` (`hash.rs:125-141`),
  `new_entry()` (`service.rs:307-315`), `make_community()` which inserts an FK-satisfying
  `communities` row with a unique host (`service.rs:296-305`).
- Intent-naming helpers that document the scenario: `nanosecond_instant()`
  (`hash.rs:145-147`), `after_database_round_trip()` (`hash.rs:150-152`).
- Infra tests degrade gracefully rather than fail when Postgres is absent:
  `PgPool::connect(...).await.ok()` then `let Some(pool) = ... else { return; }`
  (`service.rs:275-280`, used at `:321-323` etc.).
- Shared-table serialization via a `static OnceLock<tokio::sync::Mutex<()>>` guard taken
  at the top of each DB test (`service.rs:263-267`, `_g = db_lock().lock().await` at
  `:320`, `:340`, `:378`, `:439`, `:477`, `:514`).
- Tampering is simulated with raw SQL `UPDATE`/`INSERT` against the table
  (`service.rs:459-465`, `:492-505`) — testing detection, not the API.
- `matches!` assertions on error variants (`service.rs:472`, `:509`).
- Test doc comments state the property under test (`service.rs:281-282`, `:369-372`,
  `:475-477`).
- One unit test exists specifically so a regression is caught by `just test-unit`
  rather than only by the ignored tests (`service.rs:281-292`).

### Naming/typing convention at the tenant boundary

Input uses the newtype (`NewAuditEntry.community_id: CommunityId`, `entry.rs:57`);
storage and read-back use the raw `Uuid` (`entry.rs:16`, `service.rs:246`); the
conversion happens once, explicitly, at the DB boundary with a comment marking it
(`service.rs:89-91`). Query methods take `CommunityId` and bind `community.as_uuid()`
(`service.rs:162`, `:175`, `:214`, `:228`).
