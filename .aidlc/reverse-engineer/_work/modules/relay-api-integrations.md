## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Integrations

---

#### 1. Dependency summary

| Collaborator | Reached via | Surface used |
|---|---|---|
| `buzz-db` | `state.db` | 27 distinct methods (§2) |
| `buzz-media` | `state.media_storage`, free fns | 5 storage methods + 6 pipeline/helper fns (§3) |
| `buzz-search` | `state.search` | `SearchService::search` (§4) |
| `buzz-pubsub` | `state.pubsub`, `state.nip98_replay`, `state.admission_rate_limiter` | presence bulk read, Redis replay seen-set, Redis rate limiter (§5) |
| `buzz-auth` | `state.auth`, free fns/traits | `verify_nip98_event`, `Scope::all_known`, `Nip98ReplayGuard`, `LimitType`, `DEFAULT_REPLAY_TTL_SECS`, `AuthConfig::rate_limits` (§6) |
| `buzz-core` | types | `TenantContext`, `CommunityId`, `StoredEvent`, `kind::*`, `filter::{filters_match, reader_authorized_for_event}`, `tenant::{normalize_host, relay_url_authority}` |
| `buzz-audit` | `state.audit_tx` | one `NewAuditEntry` for `MediaUploaded` |
| `buzz-workflow` | `state.workflow_engine` | `WorkflowDef`, `TriggerDef`, `TriggerContext`, `executor::execute_from_step`, `finalize_run` |
| `buzz-sdk` | free fn | `nip_oa::verify_auth_tag` |
| `buzz-relay-mesh` | `state.mesh()` | `RelayPeerTransport`, `ReliableStreamRouter`, `SessionDirectory` |
| `nostr` crate | ubiquitous | `Event`, `Filter`, `EventBuilder`, `Tag`, `PublicKey`, `EventId`, `SingleLetterTag` |
| `pulldown_cmark` | `invites.rs:132` | Markdown → HTML for policy pages |
| `url` crate | `admin/mod.rs:262-264` | parsing `imeta` attachment URLs |
| `infer` crate | `media.rs:50`, `:369-372`, `:380-382` | content sniffing |

**No outbound HTTP is issued from any file in this module group.** Every `reqwest` use in the
relay lives elsewhere (`buzz-workflow/src/executor.rs`). Grep for `reqwest` across the 13 assigned
files returns zero non-test hits.

---

#### 2. `buzz-db` call patterns

| Method | Called from | `file:line` | Notes |
|---|---|---|---|
| `is_relay_member(community, pubkey_hex)` | membership gate (agent, then owner) | `api/mod.rs:73-76`, `:92-96` | two round trips on the NIP-OA delegation path |
| `ensure_user(community, pubkey)` | NIP-OA backfill | `api/mod.rs:184-188` | called twice (agent + owner) before the FK-bearing write |
| `set_agent_owner` / `is_agent_owner` | NIP-OA backfill | `api/mod.rs:200-212` | first-write-wins, then confirm-same-owner |
| `query_events(&EventQuery)` | catch-all `/query`, aux closure | `bridge.rs:492-496`, `:1252-1255`, and count fallbacks `:1483`, `:1547` | `/query` catch-all runs **bounded-concurrent** with `futures_util::buffered(FILTER_QUERY_CONCURRENCY = 4)` and order-preserving post-processing |
| `count_events(&EventQuery)` | `/count` pushdown | `bridge.rs:1480`, `:1545` | only on the fully-pushable path |
| `get_channel_window(community, ch, limit, cursor, kinds)` | channel-window filter | `bridge.rs:466-476` | single call returning rows + `has_more` + `next_cursor` + per-row `thread_summary` |
| `get_thread_replies(community, root, depth, limit, cursor)` | thread filter | `bridge.rs:1153-1162` | keyset cursor encoded BE-i64 ‖ id bytes |
| `get_events_by_ids(community, &[&[u8]])` | FTS hydration | `bridge.rs:1690-1694` | batch fetch, then a `HashMap` to restore FTS relevance order |
| `query_feed_mentions` / `query_feed_needs_action` / `query_feed_activity` | feed filter | `bridge.rs:1066-1090` | one call per requested feed type, budget shared |
| `get_workflow(community, id)` | webhook | `bridge.rs:1791-1794` | any error ⇒ generic 404 |
| `create_workflow_run` / `update_workflow_run` | webhook | `bridge.rs:1861-1865`, `:1877-1886` | the update runs inside the detached task |
| `list_moderation_reports` / `list_moderation_actions` / `list_community_restrictions` | moderation reads | `bridge.rs:2084-2091`, `:2107-2111`, `:2124-2128` | limits clamped to ≤500 before the call |
| `get_user(community, pubkey)` | upload attribution | `media.rs:262-269` | best-effort; `.ok().flatten()` degrades to no display name |
| `get_user_by_nip05(community, name, domain)` | NIP-05 | `nip05.rs:50-54` | miss and error both fold into the empty response (`_ =>` arm at `:64`) |
| `get_relay_member(community, hex)` | invite mint authz | `invites.rs:234-239` | absent member ⇒ `role = ""` ⇒ 403 |
| `claim_relay_membership(community, hex, role, policy_version)` | invite claim | `invites.rs:325-338` | returns `was_inserted` for idempotency |
| `archive_community_owned_by(host, owner, deployment_host)` | operator | `operator.rs:234-239` | `None` ⇒ 404 |
| `unarchive_community_owned_by(host, owner)` | operator | `operator.rs:288-292` | `None` ⇒ 404 |
| `list_communities_owned_by(owner_hex)` | operator | `operator.rs:325-328` | **not** community-scoped (control plane) |
| `lookup_community_by_host_for_management(host)` | operator availability | `operator.rs:480-484` | separate from the admission lookup so archived rows are visible |
| `lookup_community_host(community)` | post-transfer publish | `operator.rs:437-441` | |
| `transfer_ownership(community, new, expected)` | operator | `operator.rs:392-396` | returns the `TransferResult` enum mapped to 200/404/409 |
| `admin_list_reports(...8 args)` / `admin_get_report(id)` / `admin_list_feedback(100)` / `admin_get_feedback(id)` | admin | `admin/mod.rs:101-111`, `:132-134`, `:155`, `:184-186` | **deployment-wide**, no `CommunityId` parameter on 3 of the 4 |
| `ensure_configured_community(host)` | tests only | `bridge.rs:3415`, `invites.rs:565`, `media.rs:1015` | |
| `ping()` | readiness (outside group) | `router.rs:249` | |

Cache reads/writes owned by `AppState` but driven from here:
`get_accessible_channel_ids_cached` (`bridge.rs:1002`, `:1425`),
`author_type_cache.insert` and `observer_owner_cache.insert` after a NIP-OA materialization
(`api/mod.rs:215-224`).

#### 3. `buzz-media` call patterns

| Call | From | `file:line` |
|---|---|---|
| `auth::verify_blossom_auth_event(event, Some(tenant.host()), 3600)` | upload extractor | `media.rs:177` |
| `auth::verify_blossom_get_auth(event, sha256, Some(tenant.host()), 3600)` | read gate | `media.rs:502` |
| `process_video_upload(storage, cfg, tenant, auth_event, stream, content_length, attribution)` | streaming video | `media.rs:344-353` |
| `process_upload(...)` (buffered image) | image path | `media.rs:375-383` |
| `process_file_upload(...)` (buffered generic) | generic path | `media.rs:390-398` |
| `MediaStorage::read_sidecar_mime(tenant, hash)` | serve + head | `media.rs:637-641`, `:648-652`, `:812-816`, `:822-826` |
| `MediaStorage::get_sidecar(tenant, hash)` | ext agreement + key resolution | `media.rs:653-657`, `:829-833`, `:868-871` |
| `MediaStorage::head_with_metadata(key)` | size for 200/206/HEAD | `media.rs:673-677`, `:705-710`, `:839` |
| `MediaStorage::get_stream(key)` | 200 body | `media.rs:678` |
| `MediaStorage::get_range(key, start, end)` | 206 body | `media.rs:730` |
| `looks_like_iso_bmff`, `serve_inline`, `parse_public_ip`, `parse_port` | helpers | `media.rs:51`, `:663`, `:279-280` |
| `MediaError` as the handler error type (its own `IntoResponse`) | all media handlers | `buzz-media/src/error.rs:107-162` |

**Key direction-of-dependency fact:** `verify_blossom_get_auth` is defined in `buzz-media`
(`auth.rs:207`) but its **only** call site in the whole repo is `media.rs:502` in this module —
i.e. `buzz-media` never gates reads itself; the gate lives here behind `require_media_get_auth`.

##### S3 access

All object access is indirect through `MediaStorage`; this module never constructs an S3 client and
never handles credentials. Object keys are content-addressed (`{sha256}.{ext}`) and derived from
either the request path (after `validate_media_path`) or the sidecar's `ext` (re-validated by
`is_safe_ext`, `media.rs:875-879`) — client input never reaches the key builder unvalidated.
Sidecar/`_uploads/` keys are structurally unreachable through the serve path
(`media.rs:547-583`; test `:1250-1264`).

#### 4. `buzz-search`

One integration point: `handle_bridge_search` (`bridge.rs:1616-1749`).

| Element | Detail | `file:line` |
|---|---|---|
| Entry | `state.search.search(&SearchQuery)` | `bridge.rs:1683-1687` |
| `SearchQuery` fields | `community`, `q`, `channel_scope`, `kinds`, `authors`, `since`, `until`, `page`, `per_page`, `mode` | `bridge.rs:1672-1682` |
| `ChannelScope` | `Channels(valid_uuids)` when `#h` present and intersects accessible, else `build_search_channel_scope_filter(accessible, include_global = true)`; `None` ⇒ early `Ok([])` | `bridge.rs:1618-1650` |
| `SearchMode` | `Prefix` on `search_mode`/`searchMode == "prefix"`, else `FullText` | `bridge.rs:368-378` |
| Post-filter | FTS returns only ids; full events are fetched from `buzz-db` and re-checked by `search_hit_accepted` (all non-pushed constraints + channel scope + reader auth) | `bridge.rs:1590-1607`, `:1717-1719` |
| Error mapping | `internal_error("search error: …")` and `internal_error("search fetch error: …")` → generic 500 | `bridge.rs:1686`, `:1694` |

#### 5. `buzz-pubsub` / Redis

| Integration | Detail | `file:line` |
|---|---|---|
| Presence bulk read | `state.pubsub.get_presence_bulk(tenant, &pubkeys)`; failure ⇒ `unwrap_or_default()` (empty) | `bridge.rs:1951-1957` |
| NIP-98 replay seen-set | `RedisNip98ReplayGuard` behind `Arc<dyn Nip98ReplayGuard>` (`state.rs:710-711`); community-scoped `try_mark` for bridge/invites, deployment-scoped `try_mark_in_scope("operator-management", …)` for operator | `bridge.rs:141`, `:158-161`; `operator.rs:108-122` |
| Rate limiter | `RedisRateLimiter` (`state.rs:712`) via `admission::check_principal` with `LimitType::ApiCalls`, 60 s window | `bridge.rs:31-38` |
| Cluster disconnect fan-out | `state.disconnect_community_clusterwide(&tenant)` after archive; failure ⇒ 503 retryable | `operator.rs:243-264` |
| NIP-43 side effects | `handlers::side_effects::publish_nip43_member_added` / `publish_nip43_membership_list` after invite claim and after ownership transfer — both best-effort | `invites.rs:344-355`; `operator.rs:445-455` |
| Mesh session directory | `SessionDirectory` (Redis fenced leases) via `ReliableStreamRouter::join` | `mesh_demo.rs:71-77`, `:100` |

Two limiters in this module are **not** Redis-backed and therefore not cluster-consistent:
`media_upload_rate_limiter` + `media_uploads_in_flight` (DashMap, `state.rs:592`, `:600`) and
`invite_claim_rate_limiter` (moka, `state.rs:597-598`).

#### 6. `buzz-auth`

| Item | Use | `file:line` |
|---|---|---|
| `verify_nip98_event(json, url, method, body)` | the actual signature/URL/method/payload check | `bridge.rs:110-111` |
| `Nip98ReplayGuard` trait | injected via `AppState`; four test doubles implement it (`AlwaysErrGuard`, `AlwaysFreshReplayGuard` ×3, `SeenOnceReplayGuard`) | `bridge.rs:2330-2338`, `:3268-3281`; `invites.rs:415-427`, `:1103-1128`; `operator.rs:551-563` |
| `DEFAULT_REPLAY_TTL_SECS` | TTL for both replay scopes | `bridge.rs:159`; `operator.rs:113` |
| `LimitType::ApiCalls` | rate-limit bucket | `bridge.rs:34` |
| `Scope::all_known()` | 16 scopes granted to every HTTP ingest | `bridge.rs:829` |
| `state.auth.config().rate_limits.human_api_calls_per_min` | the per-minute budget | `bridge.rs:29` |
| `AuthError` | surfaced only through guard failures, mapped to 401 | `bridge.rs:167-176` |
| `nip42::verify_nip42_event` | tests only in this module; production caller is `handlers/auth.rs:80-81` using `bridge::nip42_expected_relay_url` | `bridge.rs:2779-2786` |

#### 7. Reverse dependencies — who calls into this module

| Consumer | Symbol | `file:line` |
|---|---|---|
| `handlers/auth.rs` (WS NIP-42 door) | `relay_members::enforce_relay_membership`, `extract_nip_oa_owner`, `materialize_nip_oa_owner`, `bridge::nip42_expected_relay_url` | `handlers/auth.rs:217`, `:137`, `:246`, `:258`, `:81` |
| `audio/handler.rs` (huddle WS) | `relay_members::enforce_relay_membership`, `bridge::nip42_expected_relay_url` | `audio/handler.rs:244`, `:219` |
| `api/git/transport.rs` | `relay_members::enforce_relay_membership` | `git/transport.rs:211` |
| `handlers/ingest.rs` | `api::validate_imeta_tags`, `api::verify_imeta_blobs`, `api::media::media_base_url_for_tenant` | `ingest.rs:2213`, `:2215`, `:2212` |
| `handlers/product_feedback.rs` | same three | `product_feedback.rs:31-33` |
| `handlers/imeta.rs` | `api::media::is_safe_ext` | `imeta.rs:378`, `:406` |
| `handlers/side_effects.rs` | `api::nip05::canonicalize_nip05` | `side_effects.rs:1145` |
| `api/admin/mod.rs` | `api::media::serve_blob_for_tenant`, `api::media::is_safe_ext` | `admin/mod.rs:226`, `:283` |
| `handlers/command_executor.rs` | `webhook_secret::{generate_webhook_secret, inject_secret, extract_secret}` | `command_executor.rs:713-718` |
| `router.rs` | every routed handler + `api::admin::{router, is_admin_host}` | `router.rs:39-128`, `:59`, `:145`, `:264` |

**Doc delta:** `api/mod.rs:36-38` states the membership gate is "Called by `media.rs`, `bridge.rs`,
`git/transport.rs`, and `audio/handler.rs`." It omits `handlers/auth.rs:217`, which is the WebSocket
door — arguably the most important caller.

#### 8. Metrics emitted

| Metric | Labels | `file:line` |
|---|---|---|
| `buzz_admission_rejections_total` | `transport="http"`, `reason="quota"\|"unavailable"` | `bridge.rs:40`, `:49` |
| `buzz_events_rejected_total` | via `reject_with_transport("http", "invalid"\|"auth"\|"error")` | `bridge.rs:734`, `:857`, `:862`, `:868` |
| `buzz_count_fallback_rejections_total` | none | `bridge.rs:1491`, `:1553` |
| `buzz_media_upload_rejections_total` | `reason="rate_limit"\|"concurrency"` | `media.rs:216`, `:222` |
| `buzz_media_legacy_upload_route_total` | none | `media.rs:315` |
| `buzz_media_uploads_total` | `mime` (6-value allowlist), `community` (tenant host) | `media.rs:419-424` |
| `buzz_audit_send_errors_total` | none | `media.rs:443` |
| `buzz_users_created_total` | `community` | `api/mod.rs:189-193` |

Note `buzz_media_uploads_total` carries an unbounded-cardinality `community` label (one series per
tenant host) — acceptable at current tenant counts, a concern on a large multi-tenant deployment.
