## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: API Surface

Three independent listeners are bound at startup (`main.rs:1113-1224`), plus one Prometheus listener bound inside `metrics::install` (`metrics.rs:73-74`):

| Listener | Bind | Router | Bound at | Middleware |
|----------|------|--------|----------|------------|
| App TCP | `config.bind_addr` (default `0.0.0.0:3000`) | `build_router` (`router.rs:32`) | `main.rs:1157` | metrics → trace → CORS |
| App UDS (optional, unix only) | `config.uds_path` | same router, `.into_make_service()` (**no `ConnectInfo`**) | `main.rs:1178-1187` | same |
| Health TCP | hard-coded `0.0.0.0:config.health_port` (default `8080`) | `build_health_router` (`router.rs:225`) | `main.rs:1116` | **none** |
| Metrics TCP | hard-coded `0.0.0.0:config.metrics_port` (default `9102`) | `PrometheusBuilder` internal | `metrics.rs:73-74`, spawned `metrics.rs:146` | none |

`router.rs` contains **34** `.route(...)` registrations: 33 production + 1 in `#[cfg(test)]` (`router.rs:476`).

---

#### 1. Complete route inventory

##### 1a. App router — `api_router` sub-router (`router.rs:56-131`, body limit 1 MB at `router.rs:129`)

| Method | Path | Handler | Reg. line | Auth enforced |
|--------|------|---------|-----------|---------------|
| GET | `/` | `nip11_or_ws_handler` (`router.rs:235`) | `router.rs:63` | **none** for NIP-11 branch; WS branch requires host→community bind (`router.rs:262`) then NIP-42 inside the socket |
| GET | `/info` | `nip11::relay_info_handler` (`nip11.rs:172`) | `router.rs:64` | **none** |
| GET | `/.well-known/nostr.json` | `api::nip05::nostr_nip05` | `router.rs:65` | **none** (NIP-05 is public by spec) |
| GET | `/health` | `health_handler` (`router.rs:295`) | `router.rs:67` | none |
| GET | `/_liveness` | `liveness_handler` (`router.rs:299`) | `router.rs:68` | none |
| GET | `/_readiness` | `readiness_handler` (`router.rs:304`) | `router.rs:69` | none |
| POST | `/events` | `api::bridge::submit_event` | `router.rs:71` | NIP-98 (`api/bridge.rs:111`) |
| POST | `/query` | `api::bridge::query_events` | `router.rs:72` | NIP-98 |
| POST | `/count` | `api::bridge::count_events` | `router.rs:73` | NIP-98 |
| GET | `/operator/communities` | `api::operator::list_owned_communities` | `router.rs:74-77` | NIP-98 + `relay_operator_pubkeys` allowlist (`api/operator.rs:70,91`) |
| POST | `/operator/communities` | `api::operator::provision_community` | `router.rs:74-77` | same |
| POST | `/operator/communities/archive` | `api::operator::archive_community` | `router.rs:78-81` | same |
| POST | `/operator/communities/unarchive` | `api::operator::unarchive_community` | `router.rs:82-85` | same |
| GET | `/operator/communities/availability` | `api::operator::community_availability` | `router.rs:86-89` | same |
| POST | `/operator/communities/transfer` | `api::operator::transfer_community` | `router.rs:90-93` | same |
| POST | `/api/invites` | `api::invites::mint_invite` | `router.rs:95` | NIP-98 + owner/admin |
| GET | `/api/join-policy` | `api::invites::join_policy` | `router.rs:96` | **none** |
| GET | `/api/join-policy/terms` | `api::invites::join_policy_terms` | `router.rs:99-102` | **none** |
| GET | `/api/join-policy/privacy` | `api::invites::join_policy_privacy` | `router.rs:103-106` | **none** |
| POST | `/api/invites/accept-policy` | `api::invites::accept_policy` | `router.rs:107-110` | NIP-98 (`api/invites.rs:193`) |
| POST | `/api/invites/claim` | `api::invites::claim_invite` | `router.rs:111` | NIP-98, **membership-gate exempt** (`router.rs:94` comment) + per-pubkey claim limiter (`api/invites.rs:380`) |
| GET | `/moderation/reports` | `api::bridge::moderation_reports` | `router.rs:113` | NIP-98 + mod-authz |
| GET | `/moderation/audit` | `api::bridge::moderation_audit` | `router.rs:114` | NIP-98 + mod-authz |
| GET | `/moderation/restricted` | `api::bridge::moderation_restricted` | `router.rs:115-118` | NIP-98 + mod-authz |
| POST | `/hooks/{id}` | `api::bridge::workflow_webhook` | `router.rs:120` | webhook secret only, **no NIP-98** (`router.rs:119` comment) |
| POST | `/_mesh/demo/echo` | `api::mesh_demo::demo_echo` | `router.rs:123` | 404 unless `BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on` |
| GET | `/huddle/{channel_id}/audio` | `audio::handler::ws_audio_handler` | `router.rs:125-128` | in-handler; `huddle_audio_available` gate at `audio/handler.rs:357` |

##### 1b. App router — `media_router` (`router.rs:37-45`, body limit `max(max_image_bytes, max_video_bytes)` at `router.rs:33-36,44`)

| Method | Path | Handler | Reg. line | Auth |
|--------|------|---------|-----------|------|
| PUT | `/upload` | `api::media::upload_blob` | `router.rs:39` | Blossom kind:24242 |
| PUT | `/media/upload` | `api::media::upload_blob` (same fn) | `router.rs:40` | same |
| GET | `/media/{sha256_ext}` | `api::media::get_blob` | `router.rs:41-44` | **off by default** — gated by `require_media_get_auth` (`config.rs:197`, default `false`) |
| HEAD | `/media/{sha256_ext}` | `api::media::head_blob` | `router.rs:43` | same |

##### 1c. App router — `git_router` (`api/git/transport.rs:1756-1765`, body limit `git_max_pack_bytes`)

| Method | Path | Handler | Line |
|--------|------|---------|------|
| GET | `/git/{owner}/{repo}/info/refs` | `info_refs` | `api/git/transport.rs:1760` |
| POST | `/git/{owner}/{repo}/git-upload-pack` | `upload_pack` | `api/git/transport.rs:1761` |
| POST | `/git/{owner}/{repo}/git-receive-pack` | `receive_pack` | `api/git/transport.rs:1762` |

##### 1d. App router — `git_policy_router` (`api/git/mod.rs:60-66`, body limit 1 MB)

| Method | Path | Handler | Line | Auth |
|--------|------|---------|------|------|
| POST | `/internal/git/policy` | `policy::hook_policy_check` | `api/git/mod.rs:62` | HMAC + `require_localhost` middleware (`api/git/mod.rs:38-52`, applied `:64`) |

##### 1e. App router — `admin_router`, conditional (`router.rs:47-54`)

Present only when `config.admin.is_some()` (`router.rs:47`), i.e. `BUZZ_ADMIN_HOST` is set. Nested under `/api/admin/v1` (`router.rs:53`). Body limit 1 KB, `security_headers` middleware (`api/admin/mod.rs:38-39`).

| Method | Path | Handler | Line |
|--------|------|---------|------|
| GET | `/api/admin/v1/reports` | `reports` | `api/admin/mod.rs:30` |
| GET | `/api/admin/v1/reports/{id}` | `report_detail` | `api/admin/mod.rs:31` |
| GET | `/api/admin/v1/feedback` | `feedback` | `api/admin/mod.rs:32` |
| GET | `/api/admin/v1/feedback/{id}` | `feedback_detail` | `api/admin/mod.rs:33` |
| GET | `/api/admin/v1/feedback/{id}/attachments/{sha256}` | `feedback_attachment` | `api/admin/mod.rs:34-37` |

##### 1f. App router — SPA fallback service (`router.rs:145-183`)

Installed only when `admin_web_dir.is_some() || web_dir.is_some()` (`router.rs:147`). Dispatch order (`router.rs:153-180`):
1. Admin host check first (`router.rs:158` → `api::admin::is_admin_host`). If admin host: serve `/assets/*` from admin `ServeDir` (`router.rs:160-162`), serve index for `is_admin_spa_path` (`router.rs:163-165` / predicate `router.rs:194-200`: `/`, `/reports`, `/reports/*`, `/feedback`, `/feedback/*`), else **404** (`router.rs:167`) — admin host can never fall through to the public bundle.
2. Public bundle: `/assets/*` from `ServeDir` (`router.rs:171-173`), index for `should_serve_spa` (`router.rs:174-176` / `router.rs:206-208`) = `is_invite_landing_path` (`/invite/{code}` with exactly one non-empty segment, `router.rs:202-204`) **or** `serve_git_web_gui && is_git_web_gui_path` (`/`, `/repos`, `/repos/*`, `router.rs:210-212`).
3. Otherwise 404 (`router.rs:178`).

##### 1g. Health router (`build_health_router`, `router.rs:225-232`)

| Method | Path | Handler | Line | Auth |
|--------|------|---------|------|------|
| GET | `/_liveness` | `liveness_handler` | `router.rs:227` | **none** |
| GET | `/_readiness` | `readiness_handler` | `router.rs:228` | **none** |
| GET | `/_status` | `status_handler` (`router.rs:387`) | `router.rs:229` | **none** — leaks `CARGO_PKG_VERSION` + uptime |
| GET | `/_mesh` | `mesh_status_handler` (`router.rs:380`) | `router.rs:230` | **none** — leaks the full mesh peer table incl. `endpoint_addrs` (`crates/buzz-relay-mesh/src/status.rs:20`) |

##### 1h. Metrics listener

`GET /metrics` (Prometheus text format) on `0.0.0.0:metrics_port`, served by `metrics_exporter_prometheus` (`metrics.rs:71-146`). Not registered in any axum router; not authenticated.

##### 1i. Unregistered route that production code calls

`crates/buzz-workflow/src/executor.rs:889` builds `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions` for the `add_reaction` workflow action. **No such route exists** in `router.rs:37-131`, `api/git/*`, or `api/admin/mod.rs`. The request lands on the SPA fallback (`router.rs:178` → 404) or a bare axum 404. Verified: the only two mentions of `api/messages` in the whole workspace are `crates/buzz-workflow/src/executor.rs:883` (doc) and `:889` (URL construction). Confirmed defect.

---

#### 2. Router-level middleware stack

Applied over the **merged** router in this order (`router.rs:185-191`, outermost last):

| Order | Layer | Line | Scope |
|-------|-------|------|-------|
| 1 (innermost) | per-sub-router `RequestBodyLimitLayer` | `router.rs:44` (media), `router.rs:129` (api, 1 MB), `api/git/transport.rs:1763`, `api/git/mod.rs:63`, `api/admin/mod.rs:39` | that sub-router only |
| 2 | `middleware::from_fn(track_metrics)` | `router.rs:188` | whole app router |
| 3 | `TraceLayer::new_for_http()` | `router.rs:189` | whole app router |
| 4 (outermost) | `build_cors_layer(&state.config.cors_origins)` | `router.rs:190`, impl `router.rs:409-432` | whole app router |

There is **no authentication middleware at any layer**. Every route's auth is enforced inside its handler. The two route-scoped middlewares are `require_localhost` (git policy, `api/git/mod.rs:64`) and `security_headers` (admin, `api/admin/mod.rs:38`).

`track_metrics` (`metrics.rs:164-206`) emits `http_requests_total{code,caller,action}` and `http_request_latency_ms{code,caller,action}`, skipping `MatchedPath` starting `/_`, `/health`, `/metrics`, and all unmatched paths (`metrics.rs:169-179`). `caller` comes from `x-envoy-downstream-service-cluster` with a `len<=64` + `[A-Za-z0-9_-]` filter (`metrics.rs:187-196`). The `p == "/metrics"` arm (`metrics.rs:170`) is unreachable — `/metrics` is never an app-router `MatchedPath`.

---

#### 3. `nip11_or_ws_handler` content negotiation (`router.rs:235-292`)

Decision order:

1. Read `ConnectInfo<SocketAddr>` or fall back to `0.0.0.0:0` (`router.rs:236-240`) — the UDS listener always hits the fallback because `main.rs:1179` uses `.into_make_service()`.
2. Read `accept` (`router.rs:242-245`) and raw `Host` (`router.rs:247-250`).
3. **Admin host short-circuit** (`router.rs:255-273`): if `is_admin_host`, require `text/html` else 404; serve admin `index.html` or 404. Admin host never reaches NIP-11 or WS.
4. `accept` contains `application/nostr+json` → `Json(nip11_document(...))` (`router.rs:275-277`). **Served before any tenant binding**, deliberately fail-open (`router.rs:279-286`).
5. Bind community from host (`router.rs:288-300`). Failure → `404 "relay: no community is configured for this host"` — a single generic message for both unmapped and lookup error (`router.rs:292-297`).
6. `WebSocketUpgrade::from_request` (`router.rs:303`):
   - Ok: if `shutting_down` → `503 "relay restarting"` (`router.rs:311-313`); else apply `limit_relay_websocket` (`router.rs:315` → `router.rs:324-332`, sets both `max_message_size` and `max_frame_size` to `config.max_frame_bytes`) and `on_upgrade(handle_connection)`.
   - Err: if `serve_git_web_gui && web_dir` and `accept` has `text/html`, serve `index.html` (`router.rs:320-330`); else fall back to the NIP-11 JSON document (`router.rs:333`).

Note step 4/6-Err: NIP-11 is reachable on `GET /` with **any** `Accept`, including none.

---

#### 4. NIP-11 document fields

`RelayInfo` (`nip11.rs:25-59`) as produced by `RelayInfo::build` (`nip11.rs:136-170`):

| Field | Line | Value | Conditional |
|-------|------|-------|-------------|
| `name` | `nip11.rs:27,152` | `"Buzz Relay"` | always (hard-coded) |
| `description` | `nip11.rs:29,153` | `"Buzz — private team communication relay"` | always (hard-coded) |
| `icon` | `nip11.rs:33,154` | `communities.icon` for the request host | omitted when `None`/empty (`skip_serializing_if`, `nip11.rs:32`); sourced `nip11.rs:277-286` |
| `pubkey` | `nip11.rs:35,155` | **always `None`** | hard-coded `None` |
| `contact` | `nip11.rs:37,156` | **always `None`** | hard-coded `None` |
| `supported_nips` | `nip11.rs:39,157` | `[1,2,10,11,16,17,23,25,29,33,38,42,50,56]` (`nip11.rs:15`) `+ [43]` when `advertise_nip43` (`nip11.rs:148-151`) | |
| `supported_extensions` | `nip11.rs:42,158` | `["nip-er"]`, `+ "nip-pl"` when push configured (`nip11.rs:254-256`) | omitted when `None` |
| `push` | `nip11.rs:45,159` | `push_descriptor(...)` object (`nip11.rs:183-233`) | omitted unless `push_gateway_delivery_url.is_some()` **and** the host binds to a community (`nip11.rs:187-188`) |
| `software` | `nip11.rs:47,160` | `"https://github.com/block/buzz"` | always |
| `version` | `nip11.rs:49,161` | `env!("CARGO_PKG_VERSION")` = `0.2.0` (`Cargo.toml:7`) | always |
| `limitation` | `nip11.rs:51,162` | `relay_limitation(config.max_frame_bytes)` | always `Some` |
| `pairing_relay_url` | `nip11.rs:54,163` | `config.pairing_relay_url` | omitted when `None` |
| `relay_self` (JSON key `self`) | `nip11.rs:58,164` | `relay_keypair.public_key().to_hex()` | only when `relay_private_key.is_some()` (`nip11.rs:302-303`) |

`RelayLimitation` (`nip11.rs:62-92`) from `relay_limitation` (`nip11.rs:96-119`):

| Field | Line | Value | Enforced? |
|-------|------|-------|-----------|
| `max_message_length` | `nip11.rs:103` | `config.max_frame_bytes` (default 524,288) | yes — `router.rs:330-331` (parser) + `recv_loop` |
| `max_subscriptions` | `nip11.rs:104` | `1024` hard-coded | yes — but via a **separate** hard-coded `MAX_SUBSCRIPTIONS = 1024` at `handlers/req.rs:26`, checked `handlers/req.rs:66` |
| `max_filters` | `nip11.rs:105` | `10` hard-coded | yes — separate const `protocol.rs:14`, checked `protocol.rs:93/131` |
| `max_limit` | `nip11.rs:106` | `10_000` hard-coded | **NO** — `buzz-db/src/event.rs:331` clamps to `max_limit.unwrap_or(1000)`; ordinary REQ does not raise it. 10× over-advertised |
| `max_subid_length` | `nip11.rs:107` | `256` | yes — separate const `protocol.rs:11`, checked `protocol.rs:86/125` |
| `min_pow_difficulty` | `nip11.rs:108` | `None` | n/a |
| `auth_required` | `nip11.rs:109` | `true` hard-coded | yes for REQ/EVENT/COUNT (asserted `nip11.rs:441-447`) |
| `payment_required` | `nip11.rs:110` | `false` | n/a |
| `restricted_writes` | `nip11.rs:111` | `true` hard-coded | advertised even on fully open relays (`require_relay_membership=false`, `pubkey_allowlist_enabled=false`) |
| `due_delivery_mode` | `nip11.rs:112` | `Some("push")` hard-coded | advertised **unconditionally**, even when `push_gateway_delivery_url` is `None` and no push worker was spawned (`main.rs:686-691`) |
| `max_not_before_delta` | `nip11.rs:113` | `SPROUT_MAX_NOT_BEFORE_DELTA` env, default `31_536_000` (`nip11.rs:97-100`) | read *per request* from env, not via `Config` |

##### NIP-11 advertisement gaps (verified)
- **NIP-45 (COUNT) is implemented but not advertised.** `ClientMessage::Count` (`protocol.rs:36`), `RelayMessage::count` (`protocol.rs:211`), `handlers/count.rs:280`, and `POST /count` (`router.rs:73`) all exist; `45` is absent from `SUPPORTED_NIPS` (`nip11.rs:15`).
- **NIP-98 (HTTP auth) is required on 12 routes but not advertised** (`api/bridge.rs:111`, `api/invites.rs:193`, `api/operator.rs`); `98` absent from `nip11.rs:15`.
- `due_delivery_mode: "push"` is advertised even with push disabled (see table above).

##### Per-request DB cost of the unauthenticated NIP-11 path
`nip11_document` (`nip11.rs:235-263`) performs, per request, with no caching and no rate limit:
1. `workspace_icon_for_host` → `bind_community` (`nip11.rs:278`) = 1 host lookup.
2. `get_community_icon` (`nip11.rs:283`) = 1 query.
3. A **second** `bind_community` (`nip11.rs:246`) when `push_gateway_delivery_url.is_some()` — which is the **default** (`config.rs:752-758` falls back to `DEFAULT_PUSH_GATEWAY_DELIVERY_URL`, `config.rs:332`).

So `GET /` and `GET /info` each cost up to 3 Postgres round-trips for an unauthenticated caller.

---

#### 5. Public items by file

| File | Public items | Notes |
|------|--------------|-------|
| `lib.rs` | 21 `pub mod` + 3 `pub use` (`lib.rs:53-55`) | all 3 re-exports unused (no external lib consumer; `main.rs:17-24` uses module paths) |
| `config.rs` | `DEFAULT_MAX_FRAME_BYTES:15`, `ConfigError:19`, `AdminConfig:29`, `JoinPolicyConfig:37`, `Config:51`, `Config::from_env:405` | all consumed |
| `error.rs` | `RelayError:8` (10 variants), `Result:49` | 9 of 10 variants dead; `Result` used only by `protocol.rs:6` |
| `protocol.rs` | `ClientMessage:23` (5 variants), `ClientMessage::parse:40`, `RelayMessage:176` + 7 fns | all consumed |
| `tenant.rs` | `HostResolver:31`, `BindError:53`, `bind_community:79`, `bind_deployment_community:130`, `relay_url_authority:139` (re-export), `impl HostResolver for Db:141` | `bind_deployment_community` ← `main.rs:562`; `relay_url_authority` ← `main.rs:239` |
| `telemetry.rs` | `service_resource:47`, `TracerInit:64`, `try_init_tracer:79` | all ← `main.rs:99-100`, `main.rs:1053` |
| `metrics.rs` | `install:71`, `track_metrics:164` | ← `main.rs:138`, `router.rs:188` |
| `nip11.rs` | `RelayInfo:25`, `RelayLimitation:62`, `RelayInfo::build:136`, `relay_info_handler:172`; `pub(crate)`: `SUPPORTED_NIPS:15`, `NIP_RELAY_MEMBERSHIP:21`, `nip11_document:235`, `nip11_facts:301` | `_RELAY_INFO_BUILD_STATIC_INPUT_FENCE` (`nip11.rs:329-335`) pins `build`'s exact signature as a compile-time conformance guard |
| `router.rs` | `build_router:32`, `build_health_router:225` | both ← `main.rs:939-940` |
| `state.rs` | see data-model §1/§5 | `bound_communities:111` has no external caller |
| `admission.rs` | `pub(crate)` only: `AdmissionError:12`, `check_principal:18`, `ws_admission_budget:39` | `check_principal` ← `api/bridge.rs:30`, `connection.rs:615/638`; `ws_admission_budget` ← `connection.rs:614` |

##### Name collision
Two distinct `AdmissionError` types coexist: `crate::admission::AdmissionError` (`admission.rs:12`, variants `Exceeded`/`Unavailable`) and `crate::audio::room::AdmissionError` (`audio/room.rs:83`, variants `Full`/`Ended`/`VersionMismatch`). Both are referenced in `handlers`/`audio` code paths (`connection.rs:657`, `audio/handler.rs:515`).
