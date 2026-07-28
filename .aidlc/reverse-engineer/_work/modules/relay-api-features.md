## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Features

---

#### 1. Product capability → endpoint map

| Capability | Endpoints | Handler `file:line` | Consumers (verified by grep) |
|---|---|---|---|
| **Send a message / publish any event over HTTP** | `POST /events` | `bridge.rs:613` | `buzz-cli` (`client.rs:874`, `:1025`), desktop Tauri (`huddle/pipeline.rs:321`, `commands/personas/snapshot/import.rs:550`), desktop E2E bridge (`e2eBridge.ts:5189`) |
| **Read messages / channels / profiles over HTTP** | `POST /query` | `bridge.rs:880` | `buzz-cli` (`client.rs:774`), mobile (`relay_session.dart:136`), desktop timeline via the bridge (`channelWindowResponse.ts:84`, `e2eBridge.ts:4219`, `:4368`, `:4527`) |
| **Unread / badge counts** | `POST /count` | `bridge.rs:1314` | `buzz-cli` (`client.rs:804`) |
| **Server-assembled channel timeline (read model)** | `POST /query` with `top_level` / `include_aux` / `include_summaries` | `bridge.rs:404-581` | desktop (`channelWindowResponse.ts`, `e2eBridge.ts:4527`) |
| **Threaded replies with keyset pagination** | `POST /query` with `depth_limit` + `thread_cursor` | `bridge.rs:1112-1183`, `:305-345` | desktop `get_thread_replies` paging (doc at `bridge.rs:307-315`); `buzz messages thread` |
| **Activity / mentions / needs-action feeds** | `POST /query` with `feed_types` | `bridge.rs:1029-1109` | desktop feed views |
| **Full-text search (NIP-50), prefix + paged** | `POST /query` with `search` | `bridge.rs:1616-1749` | desktop search; `buzz messages search` |
| **Presence (online/away)** | `POST /query` for kind:20001/40902 | `bridge.rs:1920-1985` | desktop/mobile presence; replaced the removed `GET /api/presence` (`mobile/test/features/profile/presence_cache_provider_test.dart:13`) |
| **Moderation queue, audit log, active restrictions** | `GET /moderation/reports\|audit\|restricted` | `bridge.rs:2074`, `:2100`, `:2118` | desktop (`shared/api/moderation.ts:340-373`, `features/settings/lib/moderationQueue.ts`), `buzz-cli` (`commands/moderation.rs:110`, `:120`, `:127`) |
| **Workflow automation triggered by external systems** | `POST /hooks/{id}` | `bridge.rs:1762` | external callers only — no in-repo client |
| **Media upload (images, video, generic files)** | `PUT /upload`, `PUT /media/upload` | `media.rs:305` | desktop Tauri (`commands/media.rs:487`, `:498` with legacy fallback), mobile (`relay_client.dart:23`, `media_upload.dart:19`), `buzz-cli` (`client.rs:1144`, `:1197`) |
| **Media download, thumbnails, video seeking** | `GET/HEAD /media/{sha256_ext}` | `media.rs:604`, `:798` | desktop markdown/video (`mediaEntry.ts:75`, `MarkdownVideoPlayer.tsx:36`, `VideoPlayer.tsx:96`), `buzz-cli` (`client.rs` `/media/{hash}.jpg`) |
| **Invite a new member to a closed relay** | `POST /api/invites` | `invites.rs:230` | desktop (`shared/api/invites.ts:183`, `InviteLinkSection.tsx:44`) |
| **Redeem an invite (self-service join)** | `POST /api/invites/claim` | `invites.rs:291` | desktop (`invites.ts:203`, `useClaimInvite.ts`), web (`invite-api` via `InvitePage.tsx:110`), mobile (`invite_join_provider.dart:248`) |
| **Terms/privacy consent gate before join** | `GET /api/join-policy`, `/terms`, `/privacy`, `POST /api/invites/accept-policy` | `invites.rs:75`, `:95`, `:104`, `:162` | desktop (`invites.ts:106`, `:129`, `:160`, `JoinPolicyNotice.tsx`), web (`InvitePage.tsx:69`, `:80`) |
| **Invite landing page (browser)** | SPA fallback `/invite/{code}` | `router.rs:191-194` (`is_invite_landing_path`) | `web/src/features/invite/ui/InvitePage.tsx` |
| **NIP-05 identity resolution** | `GET /.well-known/nostr.json` | `nip05.rs:26` | **no product client** — only E2E/conformance tests (`e2e_relay.rs:1061`, `:1106`; `conformance_multitenant.rs:1201`, `:1242`); external Nostr clients are the real audience |
| **Community provisioning / lifecycle (control plane)** | 6 × `/operator/communities*` | `operator.rs:149`, `:203`, `:265`, `:302`, `:354`, `:468` | **no in-repo client at all** — see §2 |
| **Deployment-wide moderation + product-feedback dashboard** | 5 × `/api/admin/v1/*` | `admin/mod.rs:93`, `:125`, `:151`, `:177`, `:191` | `admin-web/` SPA (`admin-web/src/api.ts:1`, `App.tsx:520`, `tests/routes.spec.ts`) |
| **Mesh reliable-stream smoke test** | `POST /_mesh/demo/echo` | `mesh_demo.rs:58` | **none** — self-described as "Not a product flow" (`mesh_demo.rs:21-23`) |

#### 2. Endpoints with no in-repo client

Grepped `desktop/src`, `desktop/src-tauri`, `web/src`, `mobile/lib`, `crates/buzz-cli`,
`crates/buzz-ws-client`, `crates/buzz-sdk`, `crates/buzz-admin`, `crates/buzz-pairing-cli`,
plus a repo-wide sweep.

| Endpoint | Status | Notes |
|---|---|---|
| `GET /operator/communities` | zero clients | Only relay code + its own `#[ignore]`d tests reference the path (`operator.rs:312`, `:769`). Intended for an out-of-repo control plane (likely the buzz.xyz console). |
| `POST /operator/communities` | zero clients | same |
| `POST /operator/communities/archive` | zero clients | same |
| `POST /operator/communities/unarchive` | zero clients | same |
| `GET /operator/communities/availability` | zero clients | same |
| `POST /operator/communities/transfer` | zero clients | same |
| `POST /_mesh/demo/echo` | zero clients | testbed-only, double-flag-gated |
| `GET /.well-known/nostr.json` | zero product clients | tests only, in-repo |
| `POST /hooks/{id}` | zero in-repo clients | by design (external webhook senders) |
| `GET /api/admin/v1/*` | in-repo client is `admin-web/`, a separate SPA bundle | not shipped inside desktop/mobile |

**Consequence:** 6 of the 30 method×path pairs in this module group (20%) are an unexercised
control-plane surface from this repository's perspective; their only coverage is 11 `#[ignore]`d
Postgres-backed tests in `operator.rs`.

#### 3. Features that exist in code but are unreachable

| Item | `file:line` | Why |
|---|---|---|
| `api/events.rs` re-export shim | `api/events.rs:1-5` | Zero references repo-wide; `router.rs:71-73` binds `api::bridge::*` directly. Its stated reason ("backward compatibility with router.rs") is false. |
| `webhook_secret::strip_secret` | `webhook_secret.rs:57` | Zero production callers. The documented invariant "the secret must never be embedded in a response body" therefore has no enforcement point in the HTTP surface. |
| `POST /api/messages/{id}/reactions` | absent from `router.rs:32-190` | `buzz-workflow`'s `add_reaction` action POSTs here (`buzz-workflow/src/executor.rs:889`), so that workflow action always fails against this relay. |
| `GET /api/presence` | absent | `ARCHITECTURE.md:824` describes it as existing; it was removed (`mobile/test/features/profile/presence_cache_provider_test.dart:13`). |
| `HttpAuthMethod::DevPubkey` | `handlers/ingest.rs:58` | Never constructed; `bridge.rs:830` always reports `Nip98`. |
| `IngestAuth::Http.auth_method` | `bridge.rs:830` | Never read by any consumer. |
| `relay_members::check_relay_membership` / `MembershipDecision` | `api/mod.rs:61`, `:46` | Single caller (`api/mod.rs:130`) inside the same module; the transport-neutral indirection has no second consumer. |
| `MediaError::DisallowedContentType` on `/upload` | `media.rs:384-388` | Only reachable via the **legacy** `/media/upload` alias; the modern `/upload` route accepts generic files. |

#### 4. Feature flags / staged rollouts visible here

| Flag | Default | Gated feature | `file:line` |
|---|---|---|---|
| `require_auth_token` | **false** | real NIP-98 vs `X-Pubkey` header on `/events`, `/query`, `/count`, `/moderation/*` | `bridge.rs:118-127`; `config.rs:475-477` |
| `require_relay_membership` | **false** | closed-relay admission on `/events`, `/query`, `/count`, uploads, git | `api/mod.rs:67-69`; `config.rs:483-485` |
| `allow_nip_oa_auth` | **false** | NIP-OA owner delegation *for membership admission* (the owner **backfill** path is unflagged because the signature is self-proving) | `api/mod.rs:81`, `:151-156`; `config.rs:520-522` |
| `require_media_get_auth` | **false** | Blossom read auth on `GET/HEAD /media/*`; also flips `Cache-Control` public→private | `media.rs:491-514`, `:517-523`; `config.rs:682-689` |
| `join_policy` (derived from 3 env vars) | `None` | the consent gate on `claim` and the three policy endpoints | `config.rs:794-811`; `invites.rs:314-322` |
| `admin` (`BUZZ_ADMIN_HOST`) | `None` | the entire `/api/admin/v1` sub-router **and** its security-header middleware | `router.rs:56-59`; `config.rs:813-841` |
| `relay_operator_pubkeys` + `relay_operator_api_origin` | empty / `None` | effective usability of the 6 operator routes (routes always registered; requests 403 or 500 when unconfigured) | `operator.rs:69-98`; `config.rs:549-581` |
| `mesh` + `mesh_demo_echo` | false / false | `/_mesh/demo/echo` (404 unless both on) | `mesh_demo.rs:64-70`; `config.rs:509-518` |
| `upload_records_enabled` | **false** | per-upload `_uploads/` attribution records incl. uploader display name and edge IP | `media.rs:246-249`; `config.rs:651-653` |
| `serve_git_web_gui` | false | SPA fallback for `/`, `/repos*` (invite landing is always served) | `router.rs:196-198`; `config.rs:848-851` |

#### 5. Explicitly deferred / stubbed

| Item | `file:line` | Statement |
|---|---|---|
| Persistent per-pubkey storage quotas for media | `media.rs:302-304` | `TODO(v2)` — admission limits bound *active* work but do not cap durable bytes stored. The only TODO marker in all 13 files. |
| Per-invite-code revocation | `invite_token.rs:43-46` | "Revocation is coarse: rotate the relay keypair… Per-code revocation requires the future `relay_invites` table increment." |
| `/media/upload` legacy alias | `media.rs:5-7`, `:57-63`, `:313-317` | "temporary media-only legacy alias"; usage tracked via `buzz_media_legacy_upload_route_total`. Still called by desktop as a 404/405 fallback (`desktop/src-tauri/src/commands/media.rs:498`) and `buzz-cli` (`client.rs:1183-1197`). |
| `/_mesh/demo/echo` | `mesh_demo.rs:21-23` | "The real join-side consumer (goose/berd session wiring) replaces this; the route stays demo-gated until it is deleted." |
| Timestamp-only thread cursor | `bridge.rs:316-320` | "still accepted and paginates on time alone (unsafe across same-second ties)" |
