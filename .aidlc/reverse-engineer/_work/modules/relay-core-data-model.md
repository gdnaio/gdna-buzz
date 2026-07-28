## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `main.rs` (1940), `state.rs` (1932), `config.rs` (1346), `router.rs` (530), `nip11.rs` (523), `protocol.rs` (458), `tenant.rs` (333), `telemetry.rs` (214), `metrics.rs` (207), `admission.rs` (158), `lib.rs` (56), `error.rs` (50), `Cargo.toml`. Total 7,747 LOC. No `unsafe` (`lib.rs:1` `#![deny(unsafe_code)]`); `#![warn(missing_docs)]` at `lib.rs:2`.

---

#### 1. `AppState` — every field

`AppState` is declared `state.rs:488-628` (141 lines, 39 fields), `#[derive(Clone)]` at `state.rs:487` (cheap: every non-`Copy` field is `Arc`/`Db`/`Pool`). Constructed once in `AppState::new` (`state.rs:636-798`) and wrapped `Arc<AppState>` at `main.rs:435`.

| # | Field | Line | Type | Populated by | Read by (production) |
|---|-------|------|------|--------------|----------------------|
| 1 | `config` | `state.rs:490` | `Arc<Config>` | `state.rs:717` from `main.rs:425` | pervasive (101 `state.config.*` sites) |
| 2 | `db` | `state.rs:492` | `Db` | `state.rs:719` ← `main.rs:151` | pervasive; `main.rs:954`, `router.rs:352`, `nip11.rs:246/278` |
| 3 | `redis_pool` | `state.rs:494` | `deadpool_redis::Pool` | `state.rs:720` ← `main.rs:333` (`redis_health_pool` clone) | `router.rs:353` (readiness), `main.rs:979`, `mesh_boot` via `main.rs:445` |
| 4 | `audit` | `state.rs:496` | `Option<Arc<AuditService>>` | `state.rs:718` | **ZERO readers anywhere.** See §5 |
| 5 | `pubsub` | `state.rs:498` | `Arc<PubSubManager>` | `state.rs:721` ← `main.rs:343` | `main.rs:784/822/852/903`, `state.rs:966/1039`, `handlers/*`, `connection.rs:267` |
| 6 | `auth` | `state.rs:500` | `Arc<AuthService>` | `state.rs:722` ← `main.rs:368` | `api/bridge.rs:29`, `connection.rs:614/634/636`, `handlers/auth.rs` |
| 7 | `search` | `state.rs:502` | `Arc<SearchService>` | `state.rs:723` (`search_arc`, `state.rs:652`) | `handlers/req.rs`, `api/bridge.rs` |
| 8 | `sub_registry` | `state.rs:504` | `Arc<SubscriptionRegistry>` | `state.rs:724` | `main.rs:1327`, `handlers/req.rs`, `handlers/close.rs` |
| 9 | `conn_manager` | `state.rs:506` | `Arc<ConnectionManager>` | `state.rs:725` | 17 sites incl. `main.rs:918/1130/1145`, `handlers/event.rs:146/184/460` |
| 10 | `community_connections` | `state.rs:508` | `Arc<CommunityConnectionRegistry>` | `state.rs:726` | `connection.rs:127`, `audio/handler.rs:153`, `main.rs:910`, `state.rs:1077` |
| 11 | `community_revalidator_cancel` | `state.rs:510` | `CancellationToken` | `state.rs:727` | `main.rs:886` (task), `main.rs:1045` (cancel post-serve) |
| 12 | `community_disconnect_publish_attempts` | `state.rs:512` | `Arc<AtomicU64>` | `state.rs:728` | written `state.rs:1063`; read **only** at `api/operator.rs:1010,1020`, both inside `#[cfg(test)]` (`api/operator.rs:500`) → **test-only telemetry** |
| 13 | `conn_semaphore` | `state.rs:514` | `Arc<Semaphore>` | `state.rs:729` (`max_connections`, `state.rs:649`) | `connection.rs:149`, `audio/handler.rs:90/113` |
| 14 | `handler_semaphore` | `state.rs:516` | `Arc<Semaphore>` | `state.rs:730` (`max_concurrent_handlers`, `state.rs:650`) | `connection.rs:513/541/563` |
| 15 | `git_semaphore` | `state.rs:521` | `Arc<Semaphore>` | `state.rs:731` (`git_max_concurrent_ops`, `state.rs:692`) | `api/git/transport.rs:322` |
| 16 | `media_upload_semaphore` | `state.rs:523` | `Arc<Semaphore>` | `state.rs:732` (`media_max_concurrent_uploads`, `state.rs:693`) | `api/media.rs:119` |
| 17 | `workflow_engine` | `state.rs:526` | `Arc<WorkflowEngine>` | `state.rs:733` ← `main.rs:390` | `workflow_sink.rs`, `api/bridge.rs` |
| 18 | `relay_keypair` | `state.rs:528` | `nostr::Keys` | `state.rs:734` ← `main.rs:392-414` | 47 sites; `nip11.rs:242/303`, `mesh_boot` via `main.rs:444` |
| 19 | `local_event_ids` | `state.rs:541` | `Arc<moka::sync::Cache<(CommunityId,[u8;32]),()>>` | `state.rs:736-741` | `handlers/event.rs:278/279/401/832/860/1053`, `side_effects.rs:798/877`, `audio/handler.rs:1328` |
| 20 | `membership_cache` | `state.rs:545` | `Arc<Cache<(CommunityId,Uuid,Vec<u8>),bool>>` | `state.rs:742-748` | via `is_member_cached` (`state.rs:827`); direct at `handlers/event.rs:2148/2151/2267/2270` |
| 21 | `accessible_channels_cache` | `state.rs:549` | `Arc<Cache<(CommunityId,Vec<u8>),Vec<Uuid>>>` | `state.rs:749-755` | **only via `AppState` methods** (`state.rs:1089`, `868`, `882`, `937`); no external direct reader |
| 22 | `channel_visibility_cache` | `state.rs:552` | `Arc<Cache<(CommunityId,Uuid),String>>` | `state.rs:756-762` | `state.rs:1124`, `handlers/event.rs:2119/2142/2228` |
| 23 | `audit_tx` | `state.rs:555` | `Option<mpsc::Sender<NewAuditEntry>>` | `state.rs:763` — `audit_enabled.then_some(audit_tx)` (`state.rs:716`) | `handlers/event.rs:558+`, `api/media.rs` — **this, not `audit`, is the real audit path** |
| 24 | `media_storage` | `state.rs:557` | `Arc<MediaStorage>` | `state.rs:764` ← `main.rs:419` | `main.rs:1455`, `api/media.rs` (11 sites) |
| 25 | `storage_sweep` | `state.rs:561` | `Arc<tokio::sync::Mutex<StorageSweepState>>` | `state.rs:765-767` | `main.rs:1461`, `main.rs:1474` |
| 26 | `git_store` | `state.rs:565` | `GitStore` (not `Arc`) | `state.rs:768` ← built in-constructor `state.rs:694-701` | `main.rs:491`, `api/git/*` |
| 27 | `git_pack_cache` | `state.rs:569` | `Arc<GitPackCache>` | `state.rs:769` ← `state.rs:702-709` | `api/git/transport.rs`, `api/git/hydrate.rs` |
| 28 | `audio_rooms` | `state.rs:571` | `Arc<AudioRoomManager>` | `state.rs:770` | `main.rs:455`, `audio/handler.rs` (8 sites) |
| 29 | `shutting_down` | `state.rs:573` | `Arc<AtomicBool>` | `state.rs:771` | `router.rs:281` (WS refuse), `router.rs:337` (readiness 503), `main.rs:449/457` (mesh), written `main.rs:1133` |
| 30 | `started_at` | `state.rs:575` | `Instant` | `state.rs:772` | `router.rs:388` (`/_status` uptime) — single reader |
| 31 | `nip98_replay` | `state.rs:582` | `Arc<dyn Nip98ReplayGuard>` | `state.rs:710-711` (`RedisNip98ReplayGuard`) | `api/invites.rs`, `api/operator.rs`, `api/bridge.rs` |
| 32 | `admission_rate_limiter` | `state.rs:584` | `Arc<RedisRateLimiter>` | `state.rs:712` | `api/bridge.rs:31`, `connection.rs:616`, `connection.rs:639` |
| 33 | `observer_rate_limiter` | `state.rs:589` | `Arc<ScopedRateLimiter>` (`DashMap`) | `state.rs:773` | `handlers/event.rs:901` — single reader |
| 34 | `media_upload_rate_limiter` | `state.rs:592` | `Arc<ScopedRateLimiter>` (`DashMap`) | `state.rs:774` | `api/media.rs:97` — single reader |
| 35 | `invite_claim_rate_limiter` | `state.rs:597` | `Arc<Cache<ScopedPubkeyKey,Arc<AtomicU32>>>` | `state.rs:775-780` | `api/invites.rs:380` |
| 36 | `media_uploads_in_flight` | `state.rs:600` | `Arc<DashMap<ScopedPubkeyKey,u32>>` | `state.rs:781` | `api/media.rs:70-71` |
| 37 | `observer_owner_cache` | `state.rs:607` | `Arc<Cache<(CommunityId,Vec<u8>,Vec<u8>),bool>>` | `state.rs:782-787` | `handlers/event.rs` (6 sites) |
| 38 | `author_type_cache` | `state.rs:613` | `Arc<Cache<(CommunityId,Vec<u8>),bool>>` | `state.rs:788-793` | `handlers/ingest.rs:1334/1342`, `api/mod.rs:220` |
| 39 | `tracer` | `state.rs:620` | `Arc<dyn buzz_conformance::Tracer>` | `state.rs:797` — always `NoopTracer` in production | `handlers/*` emit sites |
| 40 | `mesh` | `state.rs:627` | `Arc<OnceLock<MeshHandle>>` | `state.rs:798` empty; set at `main.rs:459` | via `AppState::mesh()` (`state.rs:812`) → `router.rs:381` |

Count: 40 fields (the table numbering matches; the struct body spans `state.rs:488-628`).

##### Verified deltas
- The `Clone` on `AppState` (`state.rs:487`) is real but *unused as a `Clone`*: every consumer takes `&AppState` or `Arc<AppState>`. `git_store` (`state.rs:565`) is the only non-`Arc`, non-pool field and is `Clone`-by-value on each `AppState` clone.
- `AppState`'s `Debug` impl (`state.rs:1209-1215`) prints only `relay_url` and `max_connections` — no secret leakage, but also no diagnostic value beyond config echo.

#### 2. In-memory caches and their eviction

| Cache | Line | Key | Value | Max capacity | TTL | Invalidation closures |
|-------|------|-----|-------|--------------|-----|-----------------------|
| `local_event_ids` | `state.rs:736-741` | `(CommunityId,[u8;32])` | `()` | 10,000 | 60 s | no |
| `membership_cache` | `state.rs:742-748` | `(CommunityId,Uuid,Vec<u8>)` | `bool` | 10,000 | 10 s | **yes** (`state.rs:746`) |
| `accessible_channels_cache` | `state.rs:749-755` | `(CommunityId,Vec<u8>)` | `Vec<Uuid>` | 10,000 | 10 s | **yes** (`state.rs:753`) |
| `channel_visibility_cache` | `state.rs:756-762` | `(CommunityId,Uuid)` | `String` | 10,000 | 10 s | **yes** (`state.rs:760`) |
| `invite_claim_rate_limiter` | `state.rs:775-780` | `ScopedPubkeyKey` | `Arc<AtomicU32>` | `api::invites::CLAIM_RATE_CACHE_CAPACITY` | `api::invites::CLAIM_RATE_WINDOW` | no |
| `observer_owner_cache` | `state.rs:782-787` | `(CommunityId,Vec<u8>,Vec<u8>)` | `bool` | 1,000 | 300 s | no |
| `author_type_cache` | `state.rs:788-793` | `(CommunityId,Vec<u8>)` | `bool` | 10,000 | 300 s | no |

Unbounded structures (no capacity cap, no TTL, no reaper):

| Structure | Line | Key | Growth bound |
|-----------|------|-----|--------------|
| `observer_rate_limiter` | `state.rs:589`, alias `state.rs:39` | `(CommunityId,[u8;32])` → `(u32, Instant)` | **none** — `DashMap` entries are never removed in this file |
| `media_upload_rate_limiter` | `state.rs:592` | same | **none** |
| `media_uploads_in_flight` | `state.rs:600` | `ScopedPubkeyKey` → `u32` | **none** in this file |
| `ConnectionManager::connections` | `state.rs:183` | `Uuid` → `ConnEntry` | bounded by `conn_semaphore` (`state.rs:514`) |
| `CommunityConnectionRegistry::connections` | `state.rs:67` | `Uuid` → `(CommunityId, CancellationToken)` | bounded by the RAII guard `state.rs:120-130` |

The doc comment on `invite_claim_rate_limiter` (`state.rs:593-596`) explicitly justifies a hard capacity because "pre-membership callers can cheaply generate fresh Nostr keys". The same argument applies to `observer_rate_limiter` and `media_upload_rate_limiter` (both keyed on caller-chosen pubkey bytes) and is *not* applied there — see debt.

#### 3. Connection-tracking types

| Type | Line | Shape | Notes |
|------|------|-------|-------|
| `ScopedPubkeyKey` | `state.rs:37` | `(CommunityId, [u8;32])` | `pub(crate)`; re-used by `api/invites.rs:386`, `api/media.rs:70` |
| `SlidingWindowCounter` | `state.rs:38` | `(u32, Instant)` | private |
| `ScopedRateLimiter` | `state.rs:39` | `DashMap<ScopedPubkeyKey, SlidingWindowCounter>` | private alias, two `AppState` fields use it |
| `ConnEntry` | `state.rs:42-63` | `tx`, `ctrl_tx`, `cancel`, `community_id`, `backpressure_count: Arc<AtomicU8>`, `subscriptions`, `authenticated_pubkey: Arc<RwLock<Option<Vec<u8>>>>`, `grace_limit: u8` | private struct, 8 fields |
| `ConnectionManager` | `state.rs:182-188` | `connections: DashMap<Uuid,ConnEntry>` + `draining: AtomicBool` | sticky one-way drain flag (`state.rs:184-187`) |
| `CommunityConnectionRegistry` | `state.rs:65-68` | `Arc<DashMap<Uuid,(CommunityId,CancellationToken)>>` | shared by ordinary WS *and* huddle audio |
| `CommunityConnectionGuard` | `state.rs:120-123` | RAII deregister (`Drop` at `state.rs:125-130`) | |
| `ThreadedChannelVisibility` | `state.rs:1162-1170` | `community_id`, `channel_id`, `visibility: String` | consumed at `handlers/event.rs:120/332/380/2232/2273/2303`, `handlers/ingest.rs:1755` |
| `AuditShutdownHandle` | `state.rs:1177-1180` | `cancel: CancellationToken`, `handle: JoinHandle<()>` | returned from `AppState::new`, drained `main.rs:1049` |

Note the *authenticated pubkey* is `Vec<u8>` inside `Arc<RwLock<Option<..>>>` (`state.rs:60`) rather than a typed `PublicKey` — every comparison is a byte-slice compare (`state.rs:271-275`).

#### 4. `Config` struct shape

`Config` at `config.rs:51-263`: **51 public fields**, no nesting except three embedded sub-configs.

| Group | Fields (line) |
|-------|---------------|
| Listeners | `bind_addr:53`, `uds_path:95`, `health_port:98`, `metrics_port:100`, `relay_url:68`, `pairing_relay_url:70` |
| Datastores | `database_url:55`, `read_database_url:58`, `redis_url:60`, `redis_pool_size:66` |
| Connection limits | `max_connections:72`, `max_concurrent_handlers:74`, `send_buffer_size:76`, `max_frame_bytes:78`, `slow_client_grace_limit:80` |
| Auth / membership | `auth:82` (`buzz_auth::AuthConfig`), `require_auth_token:85`, `pubkey_allowlist_enabled:106`, `require_relay_membership:112`, `relay_private_key:92`, `relay_owner_pubkey:149`, `relay_operator_api_origin:157`, `relay_operator_pubkeys:170`, `allow_nip_oa_auth:184` |
| Web / CORS | `cors_origins:89`, `web_dir:259`, `serve_git_web_gui:262`, `admin:254` (`AdminConfig`), `join_policy:251` (`JoinPolicyConfig`) |
| Media | `media:187` (`buzz_media::MediaConfig`), `media_max_concurrent_uploads:189`, `media_max_concurrent_uploads_per_pubkey:191`, `media_uploads_per_minute:193`, `require_media_get_auth:197` |
| Git | `git_repo_path:218`, `git_pack_cache_path:220`, `git_max_pack_bytes:222`, `git_max_repo_bytes:227`, `git_pack_cache_max_bytes:230`, `git_pack_cache_max_concurrent_populations:232`, `git_max_repos_per_pubkey:234`, `git_max_concurrent_ops:236`, `git_hook_hmac_secret:239` |
| Push (NIP-PL) | `push_executor_key_id:242`, `push_gateway_delivery_url:245`, `push_gateway_timeout:247` |
| Mesh | `mesh:136` (`buzz_relay_mesh::MeshConfig`), `mesh_demo_echo:144` |
| Misc | `huddle_audio_available:129`, `audit_enabled:202`, `ephemeral_ttl_override:209` |

Sub-structs owned by this module:
- `AdminConfig` (`config.rs:29-34`): `host: String`, `web_dir: Option<PathBuf>`.
- `JoinPolicyConfig` (`config.rs:37-47`): `terms_markdown`, `privacy_markdown`, `age_attestation_required`, `version: String` (SHA-256 over the three, `config.rs:797-801`).

`Config` derives `Debug, Clone` (`config.rs:50`) — so `{:?}` on `Config` **prints `database_url`, `redis_url`, `relay_private_key`, `git_hook_hmac_secret`, `media.s3_secret_key` in cleartext**. No `Debug` redaction anywhere in `config.rs`. See security.

`ConfigError` (`config.rs:19-27`), 2 variants: `InvalidBindAddr(String)`, `InvalidValue(String)`. Both constructed; `InvalidBindAddr` only from `parse_bind_addr` (`config.rs:265-268`).

`DEFAULT_MAX_FRAME_BYTES = 512 * 1024` (`config.rs:15`), the only public const; used at `config.rs:468` and 8 test sites in `nip11.rs`.

#### 5. Dead / write-only data

| Item | Line | Evidence |
|------|------|----------|
| `AppState::audit` | `state.rs:496` | Written `state.rs:718`. Grep for `.audit` across `crates/**` + `desktop/src-tauri/**` returns **no read**. The `AuditService` stays alive only because the worker closure captured `audit_for_worker` (`state.rs:656`), and the field's `is_some()` is consumed via the separate local `audit_enabled` (`state.rs:716`). The field is pure retained state. |
| `AppState::community_disconnect_publish_attempts` | `state.rs:512` | Incremented `state.rs:1063` on every archive publish; read only at `api/operator.rs:1010,1020` — both after `#[cfg(test)]` (`api/operator.rs:500`). Test-only → counts as zero production readers. Its own doc says "Test/telemetry counter". |
| `RelayError` — 9 of 10 variants | `error.rs:8-48` | Only `InvalidMessage` (`error.rs:44`) is ever constructed, all in `protocol.rs:43-171`. `WebSocket:11`, `Json:15` (`#[from]`), `Database:19` (`#[from]`), `Auth:23` (`#[from]`), `PubSub:27` (`#[from]`), `ConnectionLimitReached:31`, `RateLimitExceeded:35`, `NotAuthenticated:39`, `Internal:47` have zero constructors in `crates/buzz-relay/**`. (Matches in `buzz-acp` are a *different* `RelayError` type.) The four `#[from]` impls are therefore unreachable. |
| `lib.rs` re-exports | `lib.rs:53-55` | `pub use config::Config; pub use error::{RelayError, Result}; pub use state::AppState;`. No crate depends on `buzz-relay` as a library (only `buzz-relay` itself; `buzz-admin`/`buzz-conformance`/`git-sign-nostr` only mention it in comments). `main.rs:17-24` imports via full module paths (`buzz_relay::config::Config`, `buzz_relay::state::AppState`), never the re-exports. All three re-exports are unused. |
| `CommunityConnectionRegistry::bound_communities` | `state.rs:111` | `pub`, but the only non-test caller is `revalidate_registered_communities` in the same file (`state.rs:174`). Should be `pub(crate)` or private. |
| `ConnectionManager::pubkey_for` vs `pubkey_for_conn` | `state.rs:425-430` vs `state.rs:286-291` | **Byte-identical bodies and signatures.** Both live: `pubkey_for` ← `handlers/event.rs:460`; `pubkey_for_conn` ← `handlers/event.rs:146/184`, `handlers/side_effects.rs:108`. Two names, one behaviour. |
| `ConnectionState::remote_addr` | `connection.rs:61` | Populated from the router's `ConnectInfo` (`router.rs:236-240` → `connection.rs:170`). No production read; the only reads are test fixtures (`state.rs:1358`, `handlers/event.rs:1365`). The client IP is captured and discarded. |

#### 6. Protocol message types (`protocol.rs`)

`ClientMessage` (`protocol.rs:23-42`), 5 variants, all parsed and all matched in production:

| Variant | Line | Payload | Parse guard |
|---------|------|---------|-------------|
| `Event(Event)` | `protocol.rs:25` | `nostr::Event` | `arr.len() >= 2` (`protocol.rs:59`) |
| `Req { sub_id, filters }` | `protocol.rs:27-32` | `String`, `Vec<Filter>` | non-empty sub_id (`:80`), `len <= 256` (`:86`), `filters <= 10` (`:93`) |
| `Close(String)` | `protocol.rs:34` | `String` | `arr.len() >= 2` (`:147`) — **no length or emptiness check** (unlike REQ/COUNT) |
| `Count { sub_id, filters }` | `protocol.rs:36-41` | `String`, `Vec<Filter>` | same guards as REQ (`:120`, `:125`, `:131`) |
| `Auth(Event)` | `protocol.rs:42` | `nostr::Event` | `arr.len() >= 2` (`:159`) |

Constants: `MAX_SUB_ID_LENGTH = 256` (`protocol.rs:11`), `MAX_FILTERS_PER_REQ = 10` (`protocol.rs:14`). Both are **hard-coded duplicates** of the NIP-11 advertised values at `nip11.rs:107` and `nip11.rs:105` — no shared const.

`RelayMessage` (`protocol.rs:176`) is a unit struct namespace with 7 associated fns, all returning `String`. All 7 have production callers: `auth_challenge:179`, `event:184`, `notice:191`, `eose:196`, `ok:201`, `closed:206`, `count:211` (→ `handlers/count.rs:280`, single caller).

#### 7. Tenancy types (`tenant.rs`)

| Item | Line | Shape |
|------|------|-------|
| `HostResolver` trait | `tenant.rs:31-49` | assoc `type Error`; native `async fn resolve_host(&self, normalized_host:&str) -> Result<Option<CommunityId>, Self::Error>` — no `async-trait`, no `dyn` |
| `BindError<E>` | `tenant.rs:52-62` | `UnmappedHost` \| `Lookup(E)` — 2 variants, both constructed (`tenant.rs:89`, `:93`, `:94`) |
| `impl HostResolver for buzz_db::Db` | `tenant.rs:141-152` | `Error = buzz_db::DbError`; adapts `lookup_community_by_host` → `CommunityId` |
| `relay_url_authority` | `tenant.rs:139` | `pub use buzz_core::tenant::relay_url_authority` — a re-export, not a local fn |

The canonical carrier is `buzz_core::tenant::TenantContext` (imported `tenant.rs:17`), constructed only via `TenantContext::resolved(community, host)` at `tenant.rs:91` (request path), `main.rs:641` (reaper, per-row), `main.rs:743` (reminder scheduler, per-row). No `TenantContext::default()` path exists in this module.

`BindError` intentionally has **no** distinct `EmptyHost` variant — the empty/whitespace host short-circuit at `tenant.rs:85-87` reuses `UnmappedHost` so the rejection is byte-identical (documented `tenant.rs:78-84`).

#### 8. Metrics-poller data types (`main.rs`)

| Type | Line | Shape | Note |
|------|------|-------|------|
| `EmissionScope` | `main.rs:50-53` | `All` \| `Off` | `allows(&self, _community_id: &Uuid)` (`main.rs:76-78`) **ignores its argument** — the documented `top:<k>` mode (`main.rs:44-47`) is unimplemented |
| `InMemoryMetricKey` | `main.rs:1279-1284` | `WsConnections(String)` \| `UsersOnline(String)` \| `Subscriptions(String)` — carries the *host label*, not the UUID | enables final-zero emission for removed communities (`main.rs:1371-1373`) |
| `USAGE_METRICS_LOCK_KEY` | `main.rs:80` | `i64 = 0x4255_5A5A_4D45_5452` (ASCII `BUZZMETR`) | pg advisory-lock key, `main.rs:1428` |
| `SWEEP_CONFIG` | `main.rs:1444-1445` | function-local `OnceLock<StorageSweepConfig>` | deliberately *not* on `Config`/`AppState` (`main.rs:1446-1449`) |

#### 9. NIP-11 document types (`nip11.rs`)

`RelayInfo` (`nip11.rs:25-59`), 13 fields; `RelayLimitation` (`nip11.rs:62-92`), 13 fields. See api-surface for the field-by-field values. `SUPPORTED_NIPS: &[u32] = &[1,2,10,11,16,17,23,25,29,33,38,42,50,56]` (`nip11.rs:15`, 14 entries) plus conditional `NIP_RELAY_MEMBERSHIP = 43` (`nip11.rs:21`).
