## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: API Surface

Scope: the 13 assigned files (9,626 LOC). Route registrations cross-checked against
`crates/buzz-relay/src/router.rs` (530 lines) line by line.

---

#### 1. Complete endpoint inventory

Auth-mechanism legend:
- **NIP-98\*** — `Authorization: Nostr <b64(kind:27235)>` **or** the unsigned `X-Pubkey: <hex>`
  header when `config.require_auth_token == false` (**default false**, `config.rs:475-477`).
  The `*` marks endpoints that are impersonatable by default. See `bridge.rs:118-127`.
- **NIP-98 (strict)** — `verify_bridge_auth_with_options(..., require_auth_token = true, ...)`;
  no `X-Pubkey` path.
- **Blossom** — kind:24242 event in `Authorization: Nostr <b64>` (`media.rs:887-908`).
- **Operator allowlist** — NIP-98 (strict) + pubkey ∈ `config.relay_operator_pubkeys`.
- **Admin host** — `Host` header equals `config.admin.host` exactly, plus an `Origin`
  match *if* `Origin` is present. **No credential of any kind.** (`admin/auth.rs:16-35`)
- **Webhook secret** — `X-Webhook-Secret` header or `?secret=`.
- **none** — no authentication.

##### 1a. Nostr HTTP bridge (`api/bridge.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/events` | `submit_event` `bridge.rs:613` | `router.rs:71` | NIP-98\* + relay membership + Redis rate limit | raw signed Nostr event JSON (≤1 MiB) | `{event_id, accepted, message}` | 400 parse/`Rejected`, 401 auth+replay, 403 `AuthFailed`/membership, 404 unmapped host, 429 quota, 500 `Internal`, 503 limiter down |
| POST | `/query` | `query_events` `bridge.rs:880` | `router.rs:72` | NIP-98\* + membership + rate limit | JSON array of NIP-01 filters + bridge extensions | JSON array of events | 400 bad filters / bad cursor / mixed search, 401, 403 p-gate/engram/author-only/membership, 404, 429, 500, 503 |
| POST | `/count` | `count_events` `bridge.rs:1314` | `router.rs:73` | NIP-98\* + membership + rate limit | JSON array of NIP-45 filters | `{count: N}` | 400 bad filters / "count filter requires narrower constraints", 401, 403, 404, 429, 500, 503 |
| GET | `/moderation/reports` | `moderation_reports` `bridge.rs:2074` | `router.rs:113` | NIP-98\* + `ModerationAction::ViewQueue` | `?status=&limit=` (clamped ≤500) | JSON array of report rows | 401, 403 `restricted: moderator access required`, 404, 500 |
| GET | `/moderation/audit` | `moderation_audit` `bridge.rs:2100` | `router.rs:114` | NIP-98\* + ViewQueue | `?limit=` (≤500) | JSON array of action rows | 401, 403, 404, 500 |
| GET | `/moderation/restricted` | `moderation_restricted` `bridge.rs:2118` | `router.rs:115-118` | NIP-98\* + ViewQueue | none | JSON array of ban rows | 401, 403, 404, 500 |
| POST | `/hooks/{id}` | `workflow_webhook` `bridge.rs:1762` | `router.rs:120` | **webhook secret only** | UUID path + optional JSON body | 202 `{run_id, workflow_id, status:"pending"}` | 400 bad UUID / not a webhook trigger / bad JSON, 401 bad-or-unconfigured secret, 404 unmapped host or unknown workflow, 500 corrupt definition / DB |

Notes on the bridge:
- The three POST bodies are capped at **1 MiB** by `RequestBodyLimitLayer` (`router.rs:130`).
- Tenant is bound from `Host` before any auth (`bridge.rs:626`, `:894`, `:1327`, `:1783`, `:2018`);
  unmapped host → generic 404 with a fixed string, never echoing the host.
- `bridge.rs:635`, `:903`, `:1336`, `:2031` build the NIP-98 `u`-tag expectation from
  `tenant.host()`, not `config.relay_url`'s host (`nip98_expected_url`, `bridge.rs:195-206`).
- `authorize_moderation_read` (`bridge.rs:2008`) appends the **verbatim** raw query string to the
  expected URL (`:2027-2030`), so `?limit=…&status=…` reads verify.

##### 1b. Blossom media (`api/media.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| PUT | `/upload` | `upload_blob` `media.rs:305` (extractor `AuthenticatedUpload::from_request_parts` `media.rs:142`) | `router.rs:39` | Blossom + `X-SHA-256` + membership + per-pod rate/concurrency | binary body ≤ `max(max_image, max_video)` = **500 MB** default; headers `Authorization`, `X-SHA-256`, optional `X-Auth-Tag`, `Content-Length` | `BlobDescriptor` JSON (url/sha256/size/mime/uploaded/dim/blurhash/thumb/duration) | 401 (all auth failures collapse to `authentication failed`), 403 membership/scope, 404 unmapped host or unknown route mode, 413, 415, 422, 429 rate/concurrency, 500 |
| PUT | `/media/upload` | same handler, `UploadRouteMode::LegacyMedia` (`media.rs:57-63`) | `router.rs:40` | same | same | same | same, **plus 415 for any non-image** (`media.rs:386-388`) |
| GET | `/media/{sha256_ext}` | `get_blob` `media.rs:604` → `serve_blob_for_tenant` `media.rs:619` | `router.rs:41-44` | **none by default**; Blossom `get` + membership when `require_media_get_auth=true` | path `{hash}` / `{hash}.{ext}` / `{hash}.thumb.jpg`; optional `Range` | 200 streamed body or 206 slice (≤16 MiB, `media.rs:587`) | 401/403 when auth on, 404 bad path or missing sidecar, 416 unsatisfiable range, 500 |
| HEAD | `/media/{sha256_ext}` | `head_blob` `media.rs:798` | `router.rs:41-44` | same as GET | same path | 200 with `content-type`/`content-length`/`accept-ranges`/`cache-control` | 401/403, 404, 500 |

##### 1c. Invites and join policy (`api/invites.rs`)

| Method | Path | Handler (`file:line`) | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/api/invites` | `mint_invite` `invites.rs:230` | `router.rs:95` | NIP-98 (strict) + payload tag + role ∈ {owner, admin} | `{ttl_secs?}` (empty body allowed) | `{code, expires_at, url}` | 400 bad JSON, 401 auth/payload-tag/replay, 403 non-owner/admin, 404 unmapped host, 500 role lookup |
| POST | `/api/invites/claim` | `claim_invite` `invites.rs:291` | `router.rs:111` | NIP-98 (strict) + payload tag; **membership gate deliberately skipped** | `{code, policy_receipt?}` | `{status:"joined"\|"already_member", community_id, host, role}` | 400 bad JSON, 401, 403 `invite_invalid`/`invite_expired`/`join_policy_required`, 404, 429 claim limit, 500 |
| POST | `/api/invites/accept-policy` | `accept_policy` `invites.rs:162` | `router.rs:107-110` | **none** | `{code, policy_version, age_confirmed?}` | `{receipt}` | 400 bad JSON / `join_policy_not_accepted`, 404 `join_policy_not_configured` |
| GET | `/api/join-policy` | `join_policy` `invites.rs:75` | `router.rs:96` | **none** | none | `{policy:{terms_markdown, privacy_markdown, age_attestation_required, version}}` or `{}` | 200 only |
| GET | `/api/join-policy/terms` | `join_policy_terms` `invites.rs:95` | `router.rs:99-102` | **none** | none | `text/html` page | 404 `join_policy_not_configured` |
| GET | `/api/join-policy/privacy` | `join_policy_privacy` `invites.rs:104` | `router.rs:103-106` | **none** | none | `text/html` page | 404 |

##### 1d. Deployment-operator control plane (`api/operator.rs`)

All six authenticate via `authorize_operator_request` (`operator.rs:60`): NIP-98 (strict),
expected URL = `config.relay_operator_api_origin` + path + verbatim query (`:73-77`), payload tag
required iff a body is present (`:84`), replay scope `"operator-management"` (`:55`, `:104`), then
the `relay_operator_pubkeys` allowlist (`:88-98`). **The request `Host` is never bound to a tenant.**

| Method | Path | Handler (`file:line`) | Route reg. | Request | Response | Error codes |
|---|---|---|---|---|---|---|
| POST | `/operator/communities` | `provision_community` `operator.rs:149` | `router.rs:74-77` | `{host, initial_owner_pubkey?, create_only?}` | `{community_id, host, status:"created"\|"existed", owner_pubkey?}` | 400 validation, 401, 403 not-operator, 409 `community already exists`/`limit_reached:`, 500 origin unset or persistence |
| GET | `/operator/communities` | `list_owned_communities` `operator.rs:302` | `router.rs:74-77` | `?owner_pubkey=<hex>` | `{owner_pubkey, communities:[{community_id, host, created_at, archived_at}]}` | 400 bad pubkey, 401, 403, 500 |
| POST | `/operator/communities/archive` | `archive_community` `operator.rs:203` | `router.rs:78-81` | `{host, owner_pubkey}` | `{community_id, host, archived_at, status:"archived"}` | 400, 401, 403, 404 not found, 409 deployment community, **503** `propagation:"pending"` (body carries the archived state), 500 |
| POST | `/operator/communities/unarchive` | `unarchive_community` `operator.rs:265` | `router.rs:82-85` | `{host, owner_pubkey}` | `{community_id, host, archived_at:null, status:"active"}` | 400, 401, 403, 404, 500 |
| GET | `/operator/communities/availability` | `community_availability` `operator.rs:468` | `router.rs:86-89` | `?host=<authority>` | `{host, normalized_host, available, community_id}` | 400 bad host, 401, 403, 500 |
| POST | `/operator/communities/transfer` | `transfer_community` `operator.rs:354` | `router.rs:90-93` | `{community_id, new_owner_pubkey, expected_owner_pubkey}` | `{community_id, new_owner_pubkey, status:"transferred"\|"already_owner", previous_owner?}` | 400 bad UUID/pubkey, 401, 403, 404 `NoOwner`, 409 `owner_conflict:`/`limit_reached:`, 500 |

##### 1e. Deployment-admin read API (`api/admin/mod.rs`)

Sub-router built only when `config.admin.is_some()` (i.e. `BUZZ_ADMIN_HOST` set), nested at
`/api/admin/v1` (`router.rs:57-59`, `admin/mod.rs:28-41`). Body limit **1024 bytes**
(`admin/mod.rs:39`). Only this sub-router carries `security_headers` (`admin/mod.rs:38`, `:43-61`).

| Method | Path | Handler (`file:line`) | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|
| GET | `/api/admin/v1/reports` | `reports` `admin/mod.rs:93` | Admin host | `?communityId&status&reportType&targetKind&before&after&limit` (camelCase, `deny_unknown_fields`) | `Vec<AdminReport>` | 400 `invalid_limit`/`invalid_status`/`invalid_target_kind`, 403 `forbidden`, 404 (admin unconfigured), 500 |
| GET | `/api/admin/v1/reports/{id}` | `report_detail` `admin/mod.rs:125` | Admin host | UUID path | `AdminReport` | 403, 404 `not_found`, 500 |
| GET | `/api/admin/v1/feedback` | `feedback` `admin/mod.rs:151` | Admin host | none (**hard-coded limit 100**, `:155`) | `Vec<FeedbackSummary>` | 403, 404, 500 |
| GET | `/api/admin/v1/feedback/{id}` | `feedback_detail` `admin/mod.rs:177` | Admin host | UUID path | `AdminFeedback` | 403, 404, 500 |
| GET | `/api/admin/v1/feedback/{id}/attachments/{sha256}` | `feedback_attachment` `admin/mod.rs:191` | Admin host | UUID + 64-hex path | raw blob (streamed, via `media::serve_blob_for_tenant`) | 403, 404 (all provenance failures collapse here), 405 for write verbs, 500 |

Error envelope differs from the rest of the surface:
`{"error":{"code","message","requestId"}}` (`admin/error.rs:16-28`, `:60-77`).

##### 1f. NIP-05 (`api/nip05.rs`)

| Method | Path | Handler | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| GET | `/.well-known/nostr.json` | `nostr_nip05` `nip05.rs:26` | `router.rs:65` | **none** | `?name=<local>` | `{names:{name:hex}, relays:{hex:[ws url]}}` or `{names:{},relays:{}}` + `Access-Control-Allow-Origin: *` (`nip05.rs:65-69`) | **always 200** — misses, unmapped host, and absent `name` all return the empty shape (`nip05.rs:59-62`) |

##### 1g. Mesh demo (`api/mesh_demo.rs`)

| Method | Path | Handler | Route reg. | Auth | Request | Response | Error codes |
|---|---|---|---|---|---|---|---|
| POST | `/_mesh/demo/echo` | `demo_echo` `mesh_demo.rs:58` | `router.rs:123` | **none** (gated on `BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on`) | `{community_id, session_id, payload}` — **community from the body, not the Host** | `{outcome:"owned", generation, owner_runtime_id}` or `{outcome:"forwarded", …, echoed_payload}` | 404 when either flag is off, 502 send/recv/join/goodbye/closed, 504 echo timeout (10 s, `:44`), **400/422 from the `Json` extractor before the 404 gate** |

##### 1h. Adjacent routes in the same app router (out of this module group, listed for completeness)

| Method | Path | Route reg. | Auth |
|---|---|---|---|
| GET | `/` (NIP-11 or WS upgrade) | `router.rs:63` | none for NIP-11; tenant bound before upgrade (`:280-296`) |
| GET | `/info` | `router.rs:64` | none |
| GET | `/health`, `/_liveness`, `/_readiness` | `router.rs:67-69` | none |
| GET | `/huddle/{channel_id}/audio` | `router.rs:125-128` | NIP-42-style, `audio/handler.rs:219` uses `bridge::nip42_expected_relay_url` |
| GET | `/git/{owner}/{repo}/info/refs`; POST `…/git-upload-pack`, `…/git-receive-pack` | `git/transport.rs:1760-1762` | git credential / membership (`git/transport.rs:211`) |
| POST | `/internal/git/policy` | `git/mod.rs:62` | HMAC + `require_localhost` middleware (`git/mod.rs:38-50`) |
| — | SPA fallback: `/assets/*`, `/invite/{code}`, `/`, `/repos*` | `router.rs:158-186` | none |
| GET | `/_liveness`, `/_readiness`, `/_status`, `/_mesh` on the **health port** (default 8080) | `router.rs:225-231` | **none** |

**Total app-router endpoints (method × path): 40.** Of those, **26 belong to this module group**
(7 bridge + 4 media + 6 invites/policy + 6 operator + 5 admin + 1 NIP-05 + 1 mesh demo = 30 rows;
minus the 4 non-group rows counted above). Counting only the assigned files: **30 method×path pairs**.

---

#### 2. Handlers never routed / dead entry points

| Item | `file:line` | Finding |
|---|---|---|
| `api/events.rs` (whole file, 5 lines) | `api/events.rs:1-5` | **100% dead.** Re-exports `submit_event`/`query_events`/`count_events` "for backward compatibility with router.rs" — but `router.rs:71-73` calls `api::bridge::*` directly. Repo-wide grep for `api::events`, `events::submit_event`, `events::query_events`, `events::count_events` returns **zero** hits outside the file itself. |
| `webhook_secret::strip_secret` | `webhook_secret.rs:57` | **Zero production callers** (only `webhook_secret.rs:128`, `:136` in its own tests). Its doc says "Use this before returning a definition to API callers — the secret must never be embedded in a response body." Nothing in the HTTP surface calls it. |
| `HttpAuthMethod::DevPubkey` | `handlers/ingest.rs:58` | **Zero constructors repo-wide.** `bridge.rs:830` hardcodes `HttpAuthMethod::Nip98` even when the request authenticated via the unsigned `X-Pubkey` header. |
| `IngestAuth::Http { auth_method }` | `bridge.rs:830` | Write-only: no code reads the field (only 3 construction sites — `bridge.rs:830` plus two in `ingest.rs` tests). |
| `not_found`'s `#[allow(dead_code)]` | `api/mod.rs:28` | Stale attribute — `not_found` **is** used at `bridge.rs:1785` and `:1792`. |
| `relay_members::check_relay_membership` / `MembershipDecision` | `api/mod.rs:61`, `:46` | Only caller is `enforce_relay_membership` in the same module (`api/mod.rs:130`). The transport-neutral enum has one consumer, so the abstraction currently buys nothing. |
| `relay_url_for_tenant_host` | `nip05.rs:105` | Single caller, same file (`nip05.rs:55`). Fine, but `pub(crate)` is wider than needed. |
| `POST /api/messages/{id}/reactions` | — | **Confirmed absent** from `router.rs:32-190` (and from `git`/`admin`/`media` sub-routers). `buzz-workflow`'s `add_reaction_impl` POSTs there (`buzz-workflow/src/executor.rs:889`), so that workflow action cannot succeed against this relay. |
| `/api/presence` | — | Referenced by `ARCHITECTURE.md:824` as an existing endpoint. **No such route exists.** `mobile/test/features/profile/presence_cache_provider_test.dart:13` records that it "has been removed". Presence is now synthesized inside `POST /query` (`bridge.rs:1920-1985`). |

---

#### 3. Routes whose auth differs from expectation

| Route | Expected (docs/name) | Actual | `file:line` |
|---|---|---|---|
| `POST /events`, `/query`, `/count` | "NIP-98 auth" per the module doc header | NIP-98 **or** unsigned `X-Pubkey` when `require_auth_token=false` (default) | `bridge.rs:1-4` vs `:118-127`, `config.rs:475-477` |
| `GET /moderation/*` | "NIP-98 auth + mod-authz gate" (`router.rs:112`) | Same `X-Pubkey` fallback applies — `authorize_moderation_read` passes `state.config.require_auth_token` | `bridge.rs:2033` |
| `GET /api/admin/v1/*` | "deployment-admin API" | **No credential.** Host equality + optional-`Origin` check only; a missing `Origin` passes | `admin/auth.rs:16-35` |
| `POST /api/invites/accept-policy` | sits beside two NIP-98 invite routes | **No auth at all**, and no rate limit — an open HMAC-minting oracle | `invites.rs:162-190` |
| `POST /_mesh/demo/echo` | "404 (not 403) so a non-demo deployment is indistinguishable from one without the route" | `Json<DemoEchoRequest>` is a `FromRequest` extractor evaluated **before** the flag gate, so a malformed body returns 400/422 on a mesh-off relay and reveals the route | `mesh_demo.rs:71-73` vs `:60-62` |
| `GET/HEAD /media/{sha256_ext}` | Blossom read auth implied by `verify_blossom_get_auth` existing | Unauthenticated by default (`require_media_get_auth` default **false**) | `media.rs:489-514`, `config.rs:682-689` |
| `GET /operator/communities` etc. | tenant-scoped like every other route | Deliberately **not** host-bound; authority is `relay_operator_api_origin` + allowlist | `operator.rs:57-60`, `:69-77` |
| `POST /operator/*` when unconfigured | 403/404 | **500** `internal_error("operator API origin is not configured")` | `operator.rs:69-72` |

---

#### 4. Delta vs `AGENTS.md`'s "deliberately narrow" HTTP surface

`AGENTS.md` lists the sanctioned set as: NIP-11/NIP-05, `POST /events|/query|/count`,
workflow webhooks `/hooks/{id}`, Blossom media, git smart HTTP, git policy hooks, health probes.

**14 routed endpoints in this module group fall outside that list:**

| # | Endpoint | `file:line` |
|---|---|---|
| 1 | `GET /operator/communities` | `router.rs:74-77` |
| 2 | `POST /operator/communities` | `router.rs:74-77` |
| 3 | `POST /operator/communities/archive` | `router.rs:78-81` |
| 4 | `POST /operator/communities/unarchive` | `router.rs:82-85` |
| 5 | `GET /operator/communities/availability` | `router.rs:86-89` |
| 6 | `POST /operator/communities/transfer` | `router.rs:90-93` |
| 7 | `POST /api/invites` | `router.rs:95` |
| 8 | `POST /api/invites/claim` | `router.rs:111` |
| 9 | `POST /api/invites/accept-policy` | `router.rs:107-110` |
| 10 | `GET /api/join-policy` (+ `/terms`, `/privacy` = 3 routes) | `router.rs:96`, `:99-106` |
| 11 | `GET /moderation/reports` | `router.rs:113` |
| 12 | `GET /moderation/audit` | `router.rs:114` |
| 13 | `GET /moderation/restricted` | `router.rs:115-118` |
| 14 | `POST /_mesh/demo/echo` | `router.rs:123` |

Plus, outside this group but also outside the documented set: `GET /huddle/{channel_id}/audio`
(`router.rs:125-128`), the five `/api/admin/v1/*` routes (`router.rs:57-59`), and `/_status` +
`/_mesh` on the health port (`router.rs:229-230`).

`bridge.rs:1994-2000` self-documents the moderation exception ("Mod-only structured rows … are not
nostr events, so they are served over dedicated NIP-98-authed GET endpoints"), and `operator.rs:3-6`
and `invites.rs:3-6` likewise argue their cases — but `AGENTS.md` was never updated.

**Delta vs `ARCHITECTURE.md:610-628`** (its own endpoint table): omits `PUT /upload`, all 6
operator routes, all 6 invite/policy routes, all 3 moderation routes, `/_mesh/demo/echo`,
`/huddle/{channel_id}/audio`, all 5 admin routes, and `/_status` + `/_mesh`. It also states
`PUT /media/upload` has a "50 MB limit" — the actual layer is
`max(max_image_bytes, max_video_bytes)` = **500 MB** by default (`router.rs:33-36`,
`config.rs:657-672`).

---

#### 5. Bridge `/query` extension grammar (undocumented outside code)

`nostr::Filter` silently drops unknown fields, so the bridge does a two-pass parse
(`bridge.rs:969-976`) and reads extensions off the raw JSON.

| Field (aliases) | Type | Effect | `file:line` |
|---|---|---|---|
| `before_id` | 64-hex | keyset cursor; requires `until`; malformed ⇒ **400** | `bridge.rs:277-291`, `:1198-1216` |
| `top_level` | literal `true` only | channel-window read model; requires exactly one `#h` | `bridge.rs:295-297`, `:404-581` |
| `include_aux` | bool | 2-hop reaction/deletion/edit closure (kinds at `:379-390`) | `bridge.rs:483-521` |
| `include_summaries` | bool | relay-signed kind:39005 overlays | `bridge.rs:534-556` |
| `depth_limit` | u32 | thread-reply path, needs exactly one `#e` | `bridge.rs:299-303`, `:1112-1183` |
| `thread_cursor` / `threadCursor` + `thread_cursor_id` / `threadCursorId` | i64 + hex | composite thread cursor, encoded BE-i64 ‖ id bytes | `bridge.rs:305-345` |
| `feed_types` | `["mentions","needs_action","activity"\|"agent_activity"]` | server-side feed queries, limit ≤100 | `bridge.rs:347-358`, `:1029-1109` |
| `search_mode` / `searchMode` | `"prefix"` else full-text | NIP-50 mode | `bridge.rs:368-378` |
| `page` / `search_page` / `searchPage` | u32 ≥1 | FTS page **and** (via `extract_page_offset`) offset paging on non-search queries | `bridge.rs:380-388`, `:390-410`, `:1218-1229` |

Response-side synthetic kinds emitted by the bridge, relay-signed with `state.relay_keypair`:
kind:39005 thread summary and exactly one kind:39006 window bounds per window response
(`bridge.rs:523-576`), and kind:20001 presence (`bridge.rs:1966-1977`).

Caps: `BRIDGE_FEED_MAX_LIMIT=100` (`:270`), `BRIDGE_THREAD_MAX_LIMIT=500` (`:271`),
`BRIDGE_WINDOW_DEFAULT_LIMIT=50` / `BRIDGE_WINDOW_MAX_LIMIT=200` (`:374-375`),
FTS `limit.min(500)` (`:1665`), aux hop `limit=1000` (`:492`), `MODERATION_READ_LIMIT=500` (`:2059`).
