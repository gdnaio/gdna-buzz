## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Configuration

### Environment variables read by this crate

| Variable | Read at | Scope | Default |
|---|---|---|---|
| `DATABASE_URL` | `crates/buzz-audit/src/service.rs:276` | **Test-only** helper `test_pool()` inside `#[cfg(test)] mod tests` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`service.rs:277`) |

That is the **only** `std::env` access in the crate (grep for `env::var`/`std::env`
across `crates/buzz-audit/src` returns exactly `service.rs:276`).

`BUZZ_AUDIT_ENABLED` is **not** consumed here. It is read by the relay:

| Aspect | Location |
|---|---|
| Field declaration + doc | `crates/buzz-relay/src/config.rs:199-202` |
| Parse, default `true` | `crates/buzz-relay/src/config.rs:793` (`parse_bool("BUZZ_AUDIT_ENABLED", true)`) |
| Gate on service construction | `crates/buzz-relay/src/main.rs:321-334` — `Some(AuditService::new(pool))` or `None` + info log |
| Exposed as a gauge | `crates/buzz-relay/src/main.rs:139` (`buzz_audit_enabled` = 1.0/0.0) |
| Test | `crates/buzz-relay/src/config.rs:1041-1047` |

The relay's doc note states it does not control the separate `moderation_actions`
audit trail (`crates/buzz-relay/src/config.rs:200`).

### Runtime configuration surface of the crate

All configuration is by constructor argument, not environment:

| Input | Where | Notes |
|---|---|---|
| `PgPool` | `AuditService::new(pool)` (`service.rs:43-45`) | the crate never builds a pool; the relay supplies one sized `max_connections(5)`, `min_connections(1)` from `config.database_url` (`crates/buzz-relay/src/main.rs:322-328`) |
| `community: CommunityId` | `verify_chain` (`service.rs:162`), `get_entries` (`service.rs:214`) | per-call tenant scope |
| `from_seq` / `to_seq` / `limit` | `service.rs:163-164`, `:215-216` | caller-supplied; no defaults or caps in the crate |

### Cargo features

`crates/buzz-audit/Cargo.toml` declares **no `[features]` section**, no
`default-features` toggles on its own dependencies, and no `[dev-dependencies]`.
Every dependency is inherited via `workspace = true`
(`crates/buzz-audit/Cargo.toml:11-22`), so feature selection is entirely the
workspace's (e.g. sqlx features `runtime-tokio, tls-rustls, postgres, uuid, chrono,
json` at `Cargo.toml:52-54`).

Package metadata is all workspace-inherited except `description`
(`crates/buzz-audit/Cargo.toml:1-9`): `description = "Hash-chain audit log for Buzz"`
(`:8`).

### Compile-time constants

| Constant | Value | Visibility | Line |
|---|---|---|---|
| `GENESIS_HASH` | `[0u8; 32]` — 32 zero bytes | `pub` (re-exported at `lib.rs:34`) | `crates/buzz-audit/src/hash.rs:9` |
| `AUDIT_LOCK_NAMESPACE` | `"buzz_audit:"` | private to `service` module | `crates/buzz-audit/src/service.rs:29` |
| `AuditAction::ALL` | slice of all 11 variants | private (`const ALL: &'static [Self]`) | `crates/buzz-audit/src/action.rs:51-63` |

Other hard-coded values that behave as configuration:

| Value | Meaning | Line |
|---|---|---|
| `6` in `trunc_subsecs(6)` | timestamp precision = microseconds, matched to `TIMESTAMPTZ` | `hash.rs:23` |
| `0` in `hashtextextended($1, 0)` | Postgres hash seed for the advisory-lock key | `service.rs:59`, `:71` |
| `1` starting `seq` | derived from `prev_seq = 0` when a community has no rows | `service.rs:108-110` |
| presence tags `1u8` / `0u8` | optional-field framing in the pre-image | `hash.rs:55`, `:58`, `:62`, `:65` |

### Crate-level lint configuration

`#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`crates/buzz-audit/src/lib.rs:1-2`).
No `#[allow(...)]` attributes appear anywhere in the crate.

### Schema/DDL configuration

The crate ships no migrations — the `audit_log` table is owned by
`migrations/0001_initial_schema.sql:606-619`, stated explicitly at
`crates/buzz-audit/src/lib.rs:17-18`. The advisory-lock namespace `'buzz_audit:'` is
registered in migration commentary alongside other lock families
(`migrations/0023_push_match_gate.sql:21`).

### Test-environment configuration

- `test_pool()` returns `Option<PgPool>` so ignored tests skip silently when Postgres
  is unreachable (`service.rs:275-280`).
- `#[ignore = "requires Postgres"]` on all 6 async tests (`service.rs:319`, `:339`,
  `:377`, `:438`, `:476`, `:513`), so `cargo test` without `--ignored` needs no infra.
- DB tests require a `communities` row to satisfy the FK; `make_community()` inserts one
  with a unique host (`service.rs:296-305`).
- A `static OnceLock<Mutex<()>>` serializes DB tests that share the table
  (`service.rs:263-267`).
