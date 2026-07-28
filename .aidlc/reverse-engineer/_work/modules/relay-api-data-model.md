## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Data Model

This module owns no persistent schema. Everything below is either a wire DTO, a
cryptographic token structure, or a projection of a `buzz-db` row into JSON.

---

#### 1. Request DTOs (`serde::Deserialize`)

| Type | Fields | Notes | `file:line` |
|---|---|---|---|
| `WebhookQuery` | `secret: Option<String>` | `?secret=` fallback for `/hooks/{id}`; header preferred | `bridge.rs:1752-1757` |
| `ModerationReadQuery` | `status: Option<String>`, `limit: Option<i64>` | `#[derive(Default)]`; `limit` clamped by `clamp_limit` to `1..=500` | `bridge.rs:2062-2072` |
| `MintInviteRequest` | `ttl_secs: Option<u64>` | `#[serde(default)]`; empty body accepted (`invites.rs:249-258`) | `invites.rs:44-50` |
| `ClaimInviteRequest` | `code: String`, `policy_receipt: Option<String>` | `code` is required | `invites.rs:53-59` |
| `AcceptPolicyRequest` | `code: String`, `policy_version: String`, `age_confirmed: bool` (`default`) | `age_confirmed` only consulted when `age_attestation_required` | `invites.rs:62-71` |
| `ListCommunitiesQuery` | `owner_pubkey: String` | required query param | `operator.rs:28-31` |
| `CommunityAvailabilityQuery` | `host: String` | required query param | `operator.rs:34-37` |
| `TransferCommunityRequest` | `community_id: String`, `new_owner_pubkey: String`, `expected_owner_pubkey: String` | private; all three required — `expected_owner_pubkey` is the CAS token | `operator.rs:39-44` |
| `ArchiveCommunityRequest` | `host: String`, `owner_pubkey: String` | shared by archive and unarchive | `operator.rs:196-200` |
| `ProvisionCommunityRequest` | `host: String`, `initial_owner_pubkey: Option<String>`, `create_only: bool` | defined in `handlers/community_provisioning.rs:43-54`, deserialized here | `operator.rs:161` |
| `ReportQuery` | `community_id: Option<Uuid>`, `status`, `report_type`, `target_kind`, `before`/`after: Option<DateTime<Utc>>`, `limit: Option<i64>` | `rename_all="camelCase"` + **`deny_unknown_fields`** — the only strict-schema DTO in the module | `admin/mod.rs:63-73` |
| `Nip05Query` | `name: Option<String>` | lowercased before lookup (`nip05.rs:48`) | `nip05.rs:17-21` |
| `DemoEchoRequest` | `community_id: Uuid`, `session_id: Uuid`, `payload: String` | **`community_id` is client-supplied**, not host-derived | `mesh_demo.rs:48-56` |

`nostr::Filter` (plus the raw `serde_json::Value` shadow) is the de facto request DTO for
`/query` and `/count`; the bridge-only extension fields are enumerated in the api-surface aspect.

#### 2. Response DTOs

| Type | Shape | `file:line` |
|---|---|---|
| `TransferCommunityResponse` | `{community_id, new_owner_pubkey, status: &'static str, previous_owner?}` (`skip_serializing_if`) | `operator.rs:46-53` |
| `ProvisionCommunityResponse` | `{community_id, host, status: "created"\|"existed", owner_pubkey?}` | `handlers/community_provisioning.rs:58-69` |
| `FeedbackSummary` | camelCase `{id, communityId, communityHost, submitterPubkey, category?, bodySummary, receivedAt}` | `admin/mod.rs:139-149` |
| `ErrorEnvelope` / `ErrorBody` | `{"error":{"code","message","requestId"}}`; `requestId` is a **freshly generated** `Uuid::new_v4()` per response, not a correlated trace id | `admin/error.rs:16-28`, `:60-77` |
| `BlobDescriptor` | `{url, sha256, size, mime_type, uploaded, dim?, blurhash?, thumb?, duration?}` — owned by `buzz-media`, mutated here by `rewrite_descriptor_urls_for_tenant` | `media.rs:458-472` |
| `AdminReport`, `AdminFeedback` | `buzz_db::admin_moderation::*`, serialized verbatim (no field filtering) | `admin/mod.rs:93-98`, `:177-182` |

Everything else is ad-hoc `serde_json::json!` — see §5.

#### 3. Invite-token structure and signing (`invite_token.rs`)

```text
code    = base64url_nopad(payload_json) "." base64url_nopad(hmac_sha256(key, payload_json))
key     = sha256(relay_secret_key_bytes ‖ b"buzz-invite-v1")
```

| Element | Detail | `file:line` |
|---|---|---|
| `InvitePayload` | `{c: String (community UUID), r: String (role), e: u64 (unix expiry), n: String (b64 of 16 random bytes)}` | `invite_token.rs:64-74` |
| Key derivation | `derive_invite_key` = `sha256(secret_key ‖ KEY_DERIVATION_LABEL)`; label `b"buzz-invite-v1"` at `:58` | `invite_token.rs:112-117` |
| Signing | `Hmac<Sha256>` over the exact serialized payload bytes | `invite_token.rs:119-123` |
| Mint | `ttl.clamp(60, MAX_INVITE_TTL_SECS)`; `MAX_INVITE_TTL_SECS = 30 d` (`:55`), `DEFAULT_INVITE_TTL_SECS = 72 h` (`:52`); role hard-coded `"member"` | `invite_token.rs:128-149` |
| Verify order | length ≤ `MAX_CODE_LEN`(1024) → split on `.` → b64 decode both → **constant-time `mac.verify_slice`** → deserialize → expiry → community → `r == "member"` | `invite_token.rs:156-192` |
| `InviteError` | `Malformed`, `BadSignature`, `Expired`, `WrongCommunity`, `InvalidRole` | `invite_token.rs:79-92` |
| Nonce | `rand::random::<[u8;16]>()` | `invite_token.rs:133` |

Properties the code asserts and tests: multi-use until expiry (no server-side "used" bit,
`invite_token.rs:32-34`), community-scoped (`:36-38`, test `:227-235`), role-capped even for a
correctly-signed elevated payload (`:39-42`, test `:311-331`), revocation only by rotating the
relay keypair (`:43-46`).

##### Policy-acceptance receipt (same key, second payload type)

| Element | Detail | `file:line` |
|---|---|---|
| `PolicyAcceptancePayload` | `{c: hex(sha256(invite_code)), v: policy version, e: unix expiry}` | `invite_token.rs:335-343` |
| Mint | expiry = `now + 10 min`; same `sign_payload` key | `invite_token.rs:346-359` |
| Verify | length ≤ 2048 → MAC → expiry → `c` must equal `hex(sha256(code))` **and** `v == version`, else `Malformed` | `invite_token.rs:362-390` |

**Domain-separation gap:** both payload types are HMAC'd with the *same* derived key and carry no
purpose label inside the signed bytes. Cross-type confusion is currently blocked only by serde's
missing-field strictness (`InvitePayload` needs `r`+`n`; `PolicyAcceptancePayload` needs `v`), not by
an explicit tag. Adding an optional field to either struct would open the confusion.

#### 4. Webhook-secret model (`webhook_secret.rs`)

| Element | Detail | `file:line` |
|---|---|---|
| Storage location | inside the workflow definition JSON under key `"_webhook_secret"` — no separate column | `webhook_secret.rs:3-5`, `:34-41` |
| Value | `Uuid::new_v4().to_string()` — 122 bits, hyphenated, always 36 chars | `webhook_secret.rs:22-28` |
| Hash-ordering contract | `inject_secret` **must** run before `definition_hash` is computed, else the stored hash never matches | `webhook_secret.rs:7-22` |
| Read | `extract_secret` — `None` when absent or non-string | `webhook_secret.rs:44-50` |
| Redaction helper | `strip_secret` — **zero production callers** | `webhook_secret.rs:52-68` |
| Compare | `verify_secret`: length check (non-constant-time, justified at `:74-79`) then XOR-fold over all bytes | `webhook_secret.rs:70-90` |
| Consumers | mint/inject at `handlers/command_executor.rs:713-718`; verify at `bridge.rs:1805-1826` | — |

#### 5. Admin auth model

There is **no admin principal, token, or session**. The model is:

| Element | Detail | `file:line` |
|---|---|---|
| `AdminConfig` | `{host: String, web_dir: Option<PathBuf>}` | `config.rs:27-34` |
| Enablement | router exists iff `config.admin.is_some()` (i.e. `BUZZ_ADMIN_HOST` non-empty) | `router.rs:57-59`, `config.rs:813-841` |
| `is_admin_host` | exact string equality between the `Host` header and `config.admin.host` | `admin/auth.rs:6-14` |
| `authorize` | `admin` config present (else 404) → `is_admin_host` (else 403) → **if `Origin` present**, `origin_matches_host` (else 403) | `admin/auth.rs:16-33` |
| `origin_matches_host` | strips `https://` then `http://`, compares remainder | `admin/auth.rs:35-40` |
| Tenancy | none — `admin_list_reports`/`admin_list_feedback`/`admin_get_*` are deployment-wide; `communityId` is an *optional filter*, not a boundary | `admin/mod.rs:100-110`, `:151-175` |

The trust boundary is documented as the private ingress, not the application
(`docs/admin/README.md:7-9`, `:64-70`).

#### 6. Media sidecar / read model

The serve path treats **storage as non-authoritative**; the sidecar is the source of truth.

| Element | Detail | `file:line` |
|---|---|---|
| Sidecar reads | `read_sidecar_mime(tenant, hash)` for content-type; `get_sidecar(tenant, hash)` for `.ext` | `media.rs:635-660`, `:810-836` |
| Tenancy | sidecars are tenant-scoped; blobs are shared content-addressed objects, so the sidecar lookup is the per-community gate | `media.rs:632-634` |
| Path grammar | 1–3 dot segments: `{64-hex}` \| `{64-hex}.{ext}` \| `{64-hex}.thumb.jpg`; `ext` must satisfy `is_safe_ext` (1–8 chars, `[a-z0-9]`) | `media.rs:527-583` |
| Ext agreement | for `{hash}.{ext}`, `requested_ext` must equal `sidecar.ext` or 404 | `media.rs:646-658`, `:820-834` |
| Key resolution | bare hash → `{hash}.{sidecar.ext}`, re-validated through `is_safe_ext` | `media.rs:864-882` |
| Thumbnails | always `.thumb.jpg`, content-type forced to `image/jpeg`, parent sidecar must exist | `media.rs:636-643`, `:811-818` |
| Response headers | `Content-Disposition` `inline` iff `buzz_media::serve_inline(mime)` else `attachment`; always `CSP: default-src 'none'`, `X-Content-Type-Options: nosniff`, `Accept-Ranges: bytes` | `media.rs:663-668`, `:678-687`, `:736-744` |
| Cache-Control | `private, max-age=31536000, immutable` when `require_media_get_auth`, else `public, …` | `media.rs:517-523` |
| `UploadAttribution` | `{uploader_name: Option<String>, net: UploadNetworkInfo{ip, port}}`; only built when `upload_records_enabled`; IP read from `config.media.upload_ip_header` and validated public, fail-empty | `media.rs:238-283` |
| URL rewrite | descriptor `url`/`thumb` rewritten to `{scheme}://{tenant_host}/media/{sha256}.{ext}`; ext falls back to `bin` | `media.rs:447-476` |
| `AuthenticatedUpload` | private extractor: `{auth_event, tenant, route_mode, _upload_permit}` | `media.rs:33-41` |
| `UploadPermit` | RAII: global `OwnedSemaphorePermit` + a DashMap in-flight counter decremented on `Drop` | `media.rs:68-86` |
| `ScopedPubkeyKey` | `(CommunityId, [u8; 32])` — the key type for every per-pubkey limiter in this module | `state.rs:37` |

#### 7. Row → JSON projections (moderation reads)

Hand-written, not `Serialize` impls — all byte fields hex-encoded.

| Function | Emits | `file:line` |
|---|---|---|
| `report_json` | `{id, report_event_id, reporter_pubkey, target_kind: "event"\|"pubkey"\|"blob", target, channel_id, report_type, note, status, resolved_by?, resolved_at, action_id, created_at}` | `bridge.rs:2132-2153` |
| `action_json` | `{id, actor_pubkey, action, target_pubkey?, target_event_id?, channel_id, reason_code, public_reason, **private_reason**, matched_principal, created_at}` | `bridge.rs:2155-2170` |
| `ban_json` | `{pubkey, banned, ban_expires_at, ban_reason, muted_until, mute_reason, actor_pubkey, updated_at}` | `bridge.rs:2172-2184` |

`private_reason` is exposed to every ViewQueue-authorized reader (owner/admin) — by design, but note
it is the only field whose name declares it non-public.

#### 8. Standard error envelopes

Two incompatible shapes coexist:

| Shape | Used by | `file:line` |
|---|---|---|
| `{"error": "<message string>"}` | bridge, media (via `MediaError: IntoResponse`), invites, operator, mesh demo | `api/mod.rs:19-21`; `buzz-media/src/error.rs:107-162` |
| `{"error":{"code","message","requestId"}}` | admin only | `admin/error.rs:16-28` |

`internal_error` (`api/mod.rs:23-26`) logs the detail via `tracing::error!` and returns the fixed
string `"internal server error"` — the one place error text is deliberately not reflected.
