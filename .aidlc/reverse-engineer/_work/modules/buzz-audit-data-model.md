## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Data Model

Source of truth read: `crates/buzz-audit/Cargo.toml`, `src/lib.rs`, `src/action.rs`,
`src/entry.rs`, `src/error.rs`, `src/hash.rs`, `src/service.rs` (all 6 `.rs` files,
1,114 non-blank / 1,136 total lines). The crate ships **no DDL** — the `audit_log`
table is owned by migration `0001` (`crates/buzz-audit/src/lib.rs:17-18`).

---

### 1. `AuditEntry` — materialised stored row (`crates/buzz-audit/src/entry.rs:14-37`)

Derives `Debug, Clone, PartialEq, Eq, Serialize, Deserialize` (`entry.rs:13`).

| Field | Rust type | Nullability | Notes (file:line) |
|---|---|---|---|
| `community_id` | `uuid::Uuid` (raw, **not** `CommunityId`) | required | "Leads the primary key" — `entry.rs:16` |
| `seq` | `i64` | required | monotonic **within** `community_id`, starts at 1 — `entry.rs:18` |
| `hash` | `Vec<u8>` | required | SHA-256 of this entry incl. `community_id` + `prev_hash` — `entry.rs:20` |
| `prev_hash` | `Option<Vec<u8>>` | nullable | `None` for the community's first entry; hashed as `GENESIS_HASH` — `entry.rs:21-23` |
| `action` | `AuditAction` | required | `entry.rs:25` |
| `actor_pubkey` | `Option<Vec<u8>>` | nullable | raw pubkey bytes, not hex — `entry.rs:27` |
| `object_id` | `Option<String>` | nullable | generic object identifier (event id hex, channel UUID, media sha256) — `entry.rs:28-31` |
| `detail` | `serde_json::Value` | required (type has no `Option`) | included in the hash via canonical JSON — `entry.rs:32-34` |
| `created_at` | `chrono::DateTime<Utc>` | required | `entry.rs:36` |

There is **no** `event_id`, `event_kind`, or `channel_id` field on this struct.
The relay puts those two values inside `detail` instead
(`crates/buzz-relay/src/handlers/event.rs:566-569`).

### 2. `NewAuditEntry` — append input (`crates/buzz-audit/src/entry.rs:52-72`)

Derives `Debug, Clone, PartialEq, Eq` only — deliberately **not**
`Serialize`/`Deserialize`, documented as a provenance fence so no client blob can
become a `NewAuditEntry` (`entry.rs:46-51`).

| Field | Rust type | Notes (file:line) |
|---|---|---|
| `community_id` | `buzz_core::CommunityId` (newtype over `Uuid`) | server-resolved tenant only — `entry.rs:53-57`; newtype defined at `crates/buzz-core/src/tenant.rs:37` |
| `action` | `AuditAction` | `entry.rs:59` |
| `actor_pubkey` | `Option<Vec<u8>>` | `entry.rs:61` |
| `object_id` | `Option<String>` | `entry.rs:63` |
| `detail` | `serde_json::Value` | doc-comment forbids token/secret material; opaque and persisted verbatim — `entry.rs:64-71` |

`seq`, `prev_hash`, `hash`, `created_at` are assigned by
`AuditService::log` (`entry.rs:39-40`; assignment at `service.rs:110-126`).

### 3. `AuditAction` enum (`crates/buzz-audit/src/action.rs:8-31`)

`#[serde(rename_all = "snake_case")]` (`action.rs:7`); `as_str()` supplies the
stable string used in both hashing and DB storage (`action.rs:34-49`).
**11 variants** (not 10):

| # | Variant | `as_str()` string | Declared (file:line) | String mapping (file:line) |
|---|---|---|---|---|
| 1 | `EventCreated` | `event_created` | `action.rs:10` | `action.rs:37` |
| 2 | `EventDeleted` | `event_deleted` | `action.rs:12` | `action.rs:38` |
| 3 | `ChannelCreated` | `channel_created` | `action.rs:14` | `action.rs:39` |
| 4 | `ChannelUpdated` | `channel_updated` | `action.rs:16` | `action.rs:40` |
| 5 | `ChannelDeleted` | `channel_deleted` | `action.rs:18` | `action.rs:41` |
| 6 | `MemberAdded` | `member_added` | `action.rs:20` | `action.rs:42` |
| 7 | `MemberRemoved` | `member_removed` | `action.rs:22` | `action.rs:43` |
| 8 | `AuthSuccess` | `auth_success` | `action.rs:24` | `action.rs:44` |
| 9 | `AuthFailure` | `auth_failure` | `action.rs:26` | `action.rs:45` |
| 10 | `RateLimitExceeded` | `rate_limit_exceeded` | `action.rs:28` | `action.rs:46` |
| 11 | `MediaUploaded` | `media_uploaded` | `action.rs:30` | `action.rs:47` |

Private `const ALL: &'static [Self]` mirrors all 11 variants (`action.rs:51-63`) and
backs both `FromStr` (`action.rs:72-82`) and the round-trip test (`action.rs:89-94`).
`Display` delegates to `as_str()` (`action.rs:66-70`). `FromStr::Err = String`
(`action.rs:73`), message `unknown audit action: {s:?}` (`action.rs:80`).

Note the `serde` representation and `as_str()` happen to agree (snake_case), but
they are two independent mappings — `serde` is derived (`action.rs:7`), storage/hash
uses `as_str()` (`action.rs:35`).

### 4. Database row shape — `audit_log` (`migrations/0001_initial_schema.sql:606-619`)

| Column | SQL type | Constraint | Line |
|---|---|---|---|
| `community_id` | `UUID NOT NULL` | `REFERENCES communities(id)`; part of PK | `0001_initial_schema.sql:607` |
| `seq` | `BIGINT NOT NULL` | part of PK | `:608` |
| `hash` | `BYTEA NOT NULL` | unique per community (index below) | `:609` |
| `prev_hash` | `BYTEA` | nullable | `:610` |
| `action` | `VARCHAR(64) NOT NULL` | free-text; no CHECK/enum | `:611` |
| `actor_pubkey` | `BYTEA` | nullable | `:612` |
| `object_id` | `TEXT` | nullable | `:613` |
| `detail` | `JSONB` | **nullable in SQL** while the Rust field is non-`Option` | `:614` |
| `created_at` | `TIMESTAMPTZ NOT NULL DEFAULT NOW()` | value always supplied by the crate | `:615` |

- `PRIMARY KEY (community_id, seq)` — `0001_initial_schema.sql:616`
- `CREATE UNIQUE INDEX idx_audit_log_hash ON audit_log (community_id, hash)` — `:619`
- No `DELETE`/`UPDATE` statement exists anywhere in the crate; the only write is
  the `INSERT` at `service.rs:130-147`.

Column ↔ field correspondence is 1:1 for both the insert bind order
(`service.rs:137-145`) and the row-decode path (`service.rs:245-255`). `community_id`
is decoded as `Uuid` (`service.rs:246`), i.e. the typed `CommunityId` fence exists
only on the input struct, not on rows read back.

### 5. `GENESIS_HASH` (`crates/buzz-audit/src/hash.rs:9`)

```rust
pub const GENESIS_HASH: [u8; 32] = [0u8; 32];
```

**32 zero bytes** (= 64 zero hex characters when rendered). It is a hashing-time
sentinel only: the first entry of a community stores `prev_hash = NULL`
(`hash.rs:7-8`, `service.rs:108`) and `compute_hash` substitutes `GENESIS_HASH` when
`prev_hash` is `None` (`hash.rs:68-71`). `GENESIS_HASH` is re-exported at the crate
root (`lib.rs:34`).

### 6. Error type shape (`crates/buzz-audit/src/error.rs:12-41`)

`AuditError` is the only error type; see the Conventions aspect for the full variant
table. Data-carrying variants hold only `seq: i64` (`error.rs:22-25`, `29-32`) plus
wrapped `sqlx::Error` / `serde_json::Error` (`error.rs:15`, `:40`). No variant carries
a `community_id` (`error.rs:7-10`).

### 7. Public type exports (`crates/buzz-audit/src/lib.rs:31-35`)

`AuditAction`, `AuditEntry`, `NewAuditEntry`, `AuditError`, `compute_hash`,
`GENESIS_HASH`, `AuditService`. `hash::to_storage_precision` is public but not
re-exported at the root (`hash.rs:22`, absent from `lib.rs:34`).
