## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Security

---

#### 1. Auth requirement of every endpoint in this module group

| # | Endpoint | Auth actually required | Rate limited? | Tenant bound from `Host`? | `file:line` |
|---|---|---|---|---|---|
| 1 | `POST /events` | NIP-98 **or** unsigned `X-Pubkey` (default) + relay membership | ✅ Redis | ✅ | `bridge.rs:636-642`, `:798`, `:760` |
| 2 | `POST /query` | same | ✅ Redis | ✅ | `bridge.rs:904-910`, `:960`, `:955` |
| 3 | `POST /count` | same | ✅ Redis | ✅ | `bridge.rs:1337-1343`, `:1391`, `:1386` |
| 4 | `GET /moderation/reports` | same + `ViewQueue` (owner/admin) | ❌ **none** | ✅ | `bridge.rs:2033`, `:2036-2052` |
| 5 | `GET /moderation/audit` | same | ❌ **none** | ✅ | `bridge.rs:2100-2106` |
| 6 | `GET /moderation/restricted` | same | ❌ **none** | ✅ | `bridge.rs:2118-2123` |
| 7 | `POST /hooks/{id}` | **webhook secret only** — no user identity | ❌ | ✅ | `bridge.rs:1805-1826` |
| 8 | `PUT /upload` | Blossom kind:24242 + `X-SHA-256` binding + membership | ✅ per-pod | ✅ | `media.rs:142-234` |
| 9 | `PUT /media/upload` | same | ✅ per-pod | ✅ | same |
| 10 | `GET /media/{sha256_ext}` | **none by default**; Blossom `get` + membership only when `require_media_get_auth` | ❌ | ✅ | `media.rs:489-514`; `config.rs:682-689` |
| 11 | `HEAD /media/{sha256_ext}` | same as GET | ❌ | ✅ | `media.rs:798-806` |
| 12 | `POST /api/invites` | NIP-98 **strict** + payload tag + role owner/admin | ❌ | ✅ | `invites.rs:210-218`, `:233-247` |
| 13 | `POST /api/invites/claim` | NIP-98 **strict** + payload tag; membership gate intentionally skipped | ✅ per-pod moka | ✅ | `invites.rs:210-218`, `:293-298` |
| 14 | `POST /api/invites/accept-policy` | **NONE** | ❌ | ❌ (config-only) | `invites.rs:162-190` |
| 15 | `GET /api/join-policy` | **NONE** | ❌ | ❌ | `invites.rs:75-90` |
| 16 | `GET /api/join-policy/terms` | **NONE** | ❌ | ❌ | `invites.rs:95-102` |
| 17 | `GET /api/join-policy/privacy` | **NONE** | ❌ | ❌ | `invites.rs:104-110` |
| 18 | `POST /operator/communities` | NIP-98 strict + payload tag + operator allowlist | ❌ | ❌ (origin-based) | `operator.rs:66-98`, `:154-160` |
| 19 | `GET /operator/communities` | NIP-98 strict + operator allowlist | ❌ | ❌ | `operator.rs:307-315` |
| 20 | `POST /operator/communities/archive` | NIP-98 strict + payload + allowlist | ❌ | ❌ | `operator.rs:208-209` |
| 21 | `POST /operator/communities/unarchive` | same | ❌ | ❌ | `operator.rs:270-271` |
| 22 | `GET /operator/communities/availability` | NIP-98 strict + allowlist | ❌ | ❌ | `operator.rs:473-480` |
| 23 | `POST /operator/communities/transfer` | NIP-98 strict + payload + allowlist | ❌ | ❌ | `operator.rs:355-364` |
| 24 | `GET /api/admin/v1/reports` | **`Host` header equality only** (+ `Origin` iff present) | ❌ | ❌ (deployment-wide) | `admin/auth.rs:16-33`; `admin/mod.rs:100` |
| 25 | `GET /api/admin/v1/reports/{id}` | same | ❌ | ❌ | `admin/mod.rs:130` |
| 26 | `GET /api/admin/v1/feedback` | same | ❌ | ❌ | `admin/mod.rs:155` |
| 27 | `GET /api/admin/v1/feedback/{id}` | same | ❌ | ❌ | `admin/mod.rs:182` |
| 28 | `GET /api/admin/v1/feedback/{id}/attachments/{sha256}` | same + `imeta` provenance binding | ❌ | derived from the **row**, then cross-checked | `admin/mod.rs:196-240` |
| 29 | `GET /.well-known/nostr.json` | **NONE** | ❌ | ✅ (fails open to empty) | `nip05.rs:26-70` |
| 30 | `POST /_mesh/demo/echo` | **NONE** (double flag gate only) | ❌ | ❌ — **community from the body** | `mesh_demo.rs:58-77` |

##### Fully unauthenticated endpoints (explicit list)

1. `POST /api/invites/accept-policy` — `invites.rs:162`
2. `GET /api/join-policy` — `invites.rs:75`
3. `GET /api/join-policy/terms` — `invites.rs:95`
4. `GET /api/join-policy/privacy` — `invites.rs:104`
5. `GET /.well-known/nostr.json` — `nip05.rs:26`
6. `GET /media/{sha256_ext}` (default config) — `media.rs:604`
7. `HEAD /media/{sha256_ext}` (default config) — `media.rs:798`
8. `POST /_mesh/demo/echo` (when both flags on) — `mesh_demo.rs:58`
9. `POST /hooks/{id}` — no *user* auth; a shared bearer secret only — `bridge.rs:1762`
10. All 5 `GET /api/admin/v1/*` — no credential of any kind — `admin/auth.rs:16`

---

#### 2. Highest-severity findings

##### SEC-01 (CRITICAL, config-dependent) — `X-Pubkey` header grants unsigned impersonation by default

`verify_bridge_auth_with_options` accepts a bare `X-Pubkey: <hex>` header, with **no signature**,
whenever `require_auth_token == false` (`bridge.rs:118-127`). `BUZZ_REQUIRE_AUTH_TOKEN` defaults to
**false** (`config.rs:475-477`, asserted by `config.rs:954-956`'s sibling default tests). Four
routes pass that flag straight through: `/events` (`:640`), `/query` (`:908`), `/count` (`:1341`),
and all three `/moderation/*` reads (`:2033`).

Impact on a default-configured relay: any network-reachable caller can publish events as an
arbitrary pubkey, read any accessible channel as that pubkey, and — by naming a community
owner/admin's pubkey — read the entire moderation queue, audit log (including `private_reason`,
`bridge.rs:2165`), and ban list. Replay protection is additionally **skipped** for this path because
the zero event-id sentinel short-circuits the guard (`bridge.rs:122-125` → `:150-153`).

The startup warning understates it: "REST API requests bypass token auth" (`config.rs:588-593`) —
the actual effect is full pubkey impersonation. `.env.example` does not mention the variable at all.

##### SEC-02 (HIGH) — the admin API has no application-layer credential

`authorize` (`admin/auth.rs:16-33`) checks only:
(a) `config.admin.is_some()`, (b) `Host` header string-equals `config.admin.host`,
(c) **if** an `Origin` header is present, that it matches. A request with **no** `Origin` — i.e. any
non-browser client such as `curl -H 'Host: admin.example.com'` — passes (b) and skips (c) entirely.

The routes behind it are deployment-wide: `admin_list_reports`, `admin_get_report`,
`admin_list_feedback`, `admin_get_feedback` take no `CommunityId` (`admin/mod.rs:101-111`, `:132`,
`:155`, `:184`), and `feedback_attachment` streams raw blob bytes (`admin/mod.rs:226`).

This is a documented deliberate posture — "The human trust boundary remains the private admin
ingress" (`docs/admin/README.md:64-70`) — but it means a single ingress/`NetworkPolicy`
misconfiguration, or any in-cluster workload that can reach the relay port, yields full cross-tenant
read of moderation reports and user-submitted feedback plus attachments. There is no defence in
depth, no audit of who read what beyond a `tracing::info!` with no principal
(`admin/mod.rs:228-233`), and `docs/admin/README.md:66-70` explicitly disclaims per-operator
attribution.

##### SEC-03 (HIGH) — `POST /_mesh/demo/echo` takes the tenant from the request body

`DemoEchoRequest.community_id` is client-supplied (`mesh_demo.rs:50-51`) and is converted straight
into a `CommunityId` used to acquire a Redis fenced session lease (`mesh_demo.rs:99-101`). This is
the only route in the module that does not derive the tenant from `Host`, so it bypasses the row-zero
boundary every other door enforces (BR-API-01). With no authentication either, an unauthenticated
caller on a demo-enabled deployment can take or contend session leases in **any** community and
inject arbitrary payloads onto the mesh reliable stream.

Mitigation is the double flag gate (`BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on`, both default
off, `config.rs:509-518`). The file itself says "Not a product flow… the route stays demo-gated
until it is deleted" (`mesh_demo.rs:21-23`).

Secondary: the 404-indistinguishability property claimed at `mesh_demo.rs:71-73` does not hold —
`Json<DemoEchoRequest>` is a `FromRequest` extractor evaluated **before** the handler body, so a
malformed body returns 400/422 on a mesh-off relay while a genuinely absent route would return
404/405. The route's existence is therefore probeable.

##### SEC-04 (MEDIUM-HIGH) — `POST /api/invites/accept-policy` is an unauthenticated HMAC-minting oracle

`accept_policy` (`invites.rs:162-190`) has no NIP-98, no pubkey binding, and no rate limit. It
accepts an arbitrary `code` string, hashes it, and returns a relay-HMAC'd receipt
(`invite_token.rs:346-359`).

Consequences:
- The consent gate proves "someone POSTed the right version string", not "this pubkey accepted the
  policy". Any party holding an invite link can obtain a receipt on behalf of the eventual joiner,
  so the acceptance recorded at `invites.rs:324-338` is not attributable.
- It is an unbounded, unauthenticated MAC oracle over attacker-chosen input signed with
  `derive_invite_key(relay_keypair)` — the **same** key that signs invite codes
  (`invite_token.rs:112-117`, used at `invites.rs:186`, `:260`, `:301`).
- No domain-separation label distinguishes the two payload types inside the signed bytes.
  Cross-type confusion is prevented only by serde's missing-field strictness (`InvitePayload` needs
  `r` + `n`, `invite_token.rs:64-74`; `PolicyAcceptancePayload` needs `v`, `:335-343`). Adding an
  optional field to either struct would open it.

##### SEC-05 (MEDIUM) — `/moderation/*` reads have no rate limit

`authorize_moderation_read` (`bridge.rs:2008-2054`) performs bind → NIP-98 → replay → capability
check but never calls `enforce_http_admission`. These are the only NIP-98 bridge routes without a
limiter, and they return up to 500 rows each including `private_reason`. Combined with SEC-01 on a
default-config relay, this is an unmetered bulk export of the moderation record.

##### SEC-06 (MEDIUM) — unbounded per-pod media upload limiter map

`media_upload_rate_limiter` is a plain `DashMap<(CommunityId,[u8;32]), (u32, Instant)>`
(`state.rs:38-39`, `:592`, `:774`) with **no capacity bound, no TTL, and no sweep task** — grep for
`media_upload_rate_limiter` returns exactly three hits: the field, its `DashMap::new()`, and
`media.rs:97`.

`invites.rs:36-43` argues the exact reason this is wrong for the sibling limiter: "a pre-membership
caller can cheaply create fresh Nostr keypairs; retaining one immortal entry per key would make the
limiter itself an unbounded-memory denial-of-service vector." On an open relay
(`require_relay_membership = false`, the default) media upload authorization is a valid Blossom
signature from *any* key (`media.rs:196-206`), so the same argument applies verbatim — but the
media limiter got a bare DashMap while the invite limiter got moka with capacity 10 000 + 60 s TTL
(`state.rs:775-780`).

##### SEC-07 (MEDIUM) — media reads are unauthenticated by default and `verify_blossom_get_auth` has one conditional caller

`require_media_get_auth` defaults to **false** (`config.rs:682-689`, asserted by
`config.rs:973-976`). `authenticate_media_read` returns early with only the tenant when the flag is
off (`media.rs:491-494`), so `GET`/`HEAD /media/{sha256_ext}` serve blob bytes to anyone who knows a
SHA-256. `verify_blossom_get_auth` is defined in `buzz-media` (`auth.rs:207`) and its **only** call
site repo-wide is `media.rs:502` — behind that flag.

Mitigations that are real: unguessable 256-bit keys, sidecar-per-tenant lookup so a blob is only
readable under a community that has its own sidecar (`media.rs:632-660`), `attachment` disposition
for non-inline types plus `nosniff` and `CSP: default-src 'none'` (`media.rs:662-687`), and
`Cache-Control: private` once auth is on (`media.rs:517-523`).

Related: the admin attachment route calls `serve_blob_for_tenant` **directly**
(`admin/mod.rs:226`), bypassing `authenticate_media_read`, so `BUZZ_REQUIRE_MEDIA_GET_AUTH` never
applies there.

##### SEC-08 (LOW-MEDIUM) — `/hooks/{id}` is a workflow-state oracle

An unauthenticated caller can distinguish four states by status/message
(`bridge.rs:1787-1826`): unknown workflow in this community → 404 `workflow not found`; exists but
not webhook-triggered → 400 `workflow does not have a webhook trigger`; exists, webhook, no secret
configured → 401 with the descriptive `"webhook secret required but not configured — re-save the
workflow to generate one"`; exists, webhook, wrong secret → 401 `authentication failed`. UUID
entropy makes enumeration impractical, but a leaked-then-rotated id still leaks its trigger type and
secret-configuration state.

Also: the `?secret=` query fallback (`bridge.rs:1752-1757`, `:1809`) puts a bearer credential in
access logs and `Referer` headers; the doc acknowledges the header is preferred but the fallback is
unconditional and never warns.

##### SEC-09 (LOW) — reflected request content in 4xx bodies

`bridge.rs:735-742` explicitly reasons that `serde_json`'s `Display` "embeds the offending input
verbatim… would otherwise reflect attacker-controlled text into a log line at full size" and fixes
the **log** path — but the same string is still returned to the client at `bridge.rs:745`
(`"invalid event JSON: {e}"`), `:970` (`"invalid filters: {e}"`), `:1381`, and in
`invites.rs:172-176`, `:252-257`, `:302`, `operator.rs:163-167` etc. Similarly, `submit_event`
truncates `IngestError::Rejected` to 256 bytes for logs (`bridge.rs:850`) but returns the
**untruncated** message in the body (`:855`). Self-reflection only (the caller sees its own input),
so impact is bounded — but the asymmetry means the log-size defence has no response-side twin.

`internal_error` (`api/mod.rs:23-26`) is the correct pattern and is used consistently for DB and
serialization failures.

##### SEC-10 (LOW) — operator actions are not attributed

`transfer_community` binds the authenticated operator to `let _pubkey` and discards it
(`operator.rs:355`); `archive_community` (`:209`) and `unarchive_community` (`:271`) call
`authorize_operator_request(...).await?;` without binding at all. The resulting `tracing::info!`
lines carry community + host but no actor (`operator.rs:281-282`, `:299`, `:429`), and no audit-log
entry is written. `provision_community` is the only one that threads the operator pubkey through
(`operator.rs:157`, `:179`). Community archival and ownership transfer are the two highest-impact
operations on the surface.

---

#### 3. NIP-98 verification and replay protection

| Property | Status | `file:line` |
|---|---|---|
| Signature/URL/method/payload verification | delegated to `buzz_auth::verify_nip98_event` | `bridge.rs:110-111` |
| `u`-tag host bound to the **resolved tenant**, not `config.relay_url` | ✅ | `bridge.rs:195-206`; tests `:2417-2449`, `:2477-2504`, `:2636-2654` |
| Query string included verbatim in the expected URL for GETs | ✅ | `bridge.rs:2027-2031`; `operator.rs:73-77`; tests `:2529-2630` |
| `payload` tag required on body-bearing requests | ✅ for invites (`invites.rs:217`) and operator (`operator.rs:84`); ❌ **not** for `/events`, `/query`, `/count`, which call `verify_bridge_auth` (`require_payload = false`, `bridge.rs:69`) | `bridge.rs:97-108`, `:62-70` |
| Replay guard consulted on every NIP-98 request | ✅ all 8 NIP-98 routes: `bridge.rs:766` (`/events`), `:956` (`/query`), `:1387` (`/count`), `:2034` (moderation ×3), `invites.rs:219` (mint + claim), `operator.rs:85` (all 6 operator) | — |
| Guard error fails **CLOSED** | ✅ 401 `"NIP-98: replay check unavailable"` | `bridge.rs:167-176`; operator sibling `operator.rs:126-137`; infra-free test `bridge.rs:2340-2377` |
| Cross-pod correctness | ✅ shared Redis `SET NX EX`, community-scoped key | `state.rs:710-711`; tests `bridge.rs:2262-2292` (two guard instances, one pool) |
| Same-pod replay still rejected after the moka→Redis migration | ✅ | test `bridge.rs:2294-2317` |
| Community scoping of the seen-set | ✅ same event id in a different community succeeds | test `bridge.rs:2290-2292` |
| Operator scope isolation | ✅ dedicated `"operator-management"` scope, not a community | `operator.rs:55`, `:108-122` |
| **Gap** | replay is skipped for the zero event-id sentinel produced by the `X-Pubkey` path | `bridge.rs:122-125`, `:150-153` |

**Note on the payload-tag asymmetry:** `/events` does not require a `payload` tag, but
`verify_nip98_event` receives `Some(&body)` (`bridge.rs:639`), so if a payload tag *is* present it is
verified against the body. A NIP-98 event without a payload tag is accepted and can therefore be
replayed against a *different* body — except the replay guard blocks the second use of the same
event id, and the seen-set is community-scoped. Net effect: single-use, so not exploitable, but the
safety depends entirely on the replay guard rather than on payload binding. The invite and operator
routes chose the stronger posture (`require_payload = true`).

---

#### 4. Rate limiting posture

| Surface | Backend | Scope | Fail mode | `file:line` |
|---|---|---|---|---|
| `/events`, `/query`, `/count` | **Redis** `RedisRateLimiter` | `(tenant, pubkey)`, 60 s window, `human_api_calls_per_min` (default 300) | **closed** (503) | `bridge.rs:24-56`; `admission.rs:24-33`; `state.rs:712` |
| Media upload | in-process DashMap | `(community, pubkey)`, 60 s, `media_uploads_per_minute` (30) | n/a — pure local | `media.rs:88-111` |
| Media upload concurrency | in-process `Semaphore` + DashMap counter | global 8, per-pubkey 2 | reject 429 | `media.rs:113-136` |
| Invite claim | in-process moka (cap 10 000, TTL 60 s) | `(community, pubkey)`, 10/window | n/a | `invites.rs:374-390`; `state.rs:775-780` |
| Everything else in the module | **none** | — | — | see §1 |

**Documentation delta (confirmed):** `ARCHITECTURE.md:823` (§9 limitation #2) states "No rate
limiting implementation… none are enforced"; `ARCHITECTURE.md:390` states "No Redis-backed rate
limiter exists anywhere in the codebase — rate limiting is not currently enforced"; and
`ARCHITECTURE.md:459` says `buzz-pubsub` "Does NOT implement the rate limiter". All three are
**false**: `RedisRateLimiter` lives in `buzz_pubsub::rate_limiter` (`state.rs:26`), is constructed at
`state.rs:712`, and is consulted on all three bridge POSTs (`bridge.rs:760`, `:955`, `:1386`).

---

#### 5. Admin token handling / constant-time comparison

There is **no admin token** — see SEC-02. The only constant-time comparison in the module group is
the webhook secret:

```rust
if provided.len() != stored.len() { return false; }      // webhook_secret.rs:82-84
let mut result = 0u8;
for (a, b) in provided.bytes().zip(stored.bytes()) { result |= a ^ b; }  // :86-88
result == 0
```

Length is revealed non-constant-time, justified at `webhook_secret.rs:74-79` on the grounds that the
generator always emits a 36-char UUID. Sound.

The operator allowlist uses plain `==` string comparison (`operator.rs:91-95`) — acceptable, since
the compared value is a public pubkey, not a secret. The admin host comparison is likewise plain
equality on a non-secret (`admin/auth.rs:11-14`).

---

#### 6. Invite-token forgery resistance

| Property | Assessment | `file:line` |
|---|---|---|
| MAC | HMAC-SHA256 over the exact serialized payload bytes | `invite_token.rs:119-123` |
| Key | `sha256(relay_secret_key ‖ b"buzz-invite-v1")` — domain-separated from other uses of the keypair | `invite_token.rs:112-117`, `:58` |
| Verification order | MAC **before** trusting any payload claim | `invite_token.rs:171-176`; doc `:151-155` |
| Comparison | `mac.verify_slice` — constant-time by the `hmac` crate | `invite_token.rs:174` |
| Tamper resistance | re-encoding the payload with `r:"owner"` and the original MAC ⇒ `BadSignature` | test `invite_token.rs:238-262` |
| Wrong key | ⇒ `BadSignature` | test `invite_token.rs:264-272` |
| Role ceiling re-checked post-MAC | a correctly-signed `r:"admin"` payload is still rejected | `invite_token.rs:189-191`; test `:311-331` |
| Community binding | code minted for A rejected on B | `invite_token.rs:186-188`; endpoint test `invites.rs:1004-1046` |
| Input bounds before parsing | `MAX_CODE_LEN = 1024` (receipts 2048) | `invite_token.rs:57`, `:157-159`, `:369-371` |
| Oracle hardening at the HTTP layer | only `Expired` is distinguishable; all else → 403 `invite_invalid` | `invites.rs:303-312` |
| Nonce | 16 random bytes so identically-parameterised invites differ | `invite_token.rs:133` |
| **Non-properties (documented)** | multi-use until expiry (no used-bit); revocation only by rotating the relay keypair | `invite_token.rs:32-34`, `:43-46` |
| **Residual risk** | shared HMAC key with the policy-receipt payload and no in-payload purpose label (see SEC-04) | `invite_token.rs:346-359` |

Forgery resistance is sound. The weak links are operational, not cryptographic: a leaked link is
valid for up to 30 days for unlimited redemptions, and revocation is deployment-wide.

---

#### 7. Tenancy isolation per endpoint

| Endpoint group | Isolation mechanism | Verdict |
|---|---|---|
| Bridge (`/events`, `/query`, `/count`) | `bind_community(Host)` → every DB call takes `tenant.community()`; NIP-98 `u`-host must equal `tenant.host()`; accessible-channel set resolved per tenant | ✅ strong |
| Moderation reads | same bind; queries community-scoped; `ViewQueue` evaluated against `relay_members` in that community | ✅ |
| Webhook | tenant from `Host`, **not** from the workflow row — same UUID in another community is a 404 | ✅ documented at `bridge.rs:1773-1782` |
| Media upload | tenant bound before Blossom verification so the `server` tag validates against the bound host | ✅ `media.rs:145-156` |
| Media read | sidecar lookup is tenant-scoped; blobs are shared content-addressed objects, so a blob is only readable under a community holding its own sidecar | ✅ `media.rs:632-660` |
| Invites | community from `Host`; code carries the community UUID and is re-checked | ✅ double-bound |
| Join policy | **deployment-global config**, not per-community — all tenants share one policy and one version | ⚠️ by design (`config.rs:794-811`) |
| Operator | deliberately **not** tenant-bound; authority is `relay_operator_api_origin` + allowlist; `list_communities_owned_by` and `lookup_community_by_host_for_management` are cross-community by design | ⚠️ intended control plane (`operator.rs:57-60`) |
| Admin | deliberately deployment-wide; `communityId` is an optional filter, and 3 of 4 DB calls take no community at all | ⚠️ intended, but the boundary is the ingress (SEC-02) |
| Admin attachment | tenant derived from the row's own `community_host`, then asserted to still resolve to `feedback.community_id`, plus `imeta` `x`+`url` binding | ✅ the strongest provenance chain in the module (`admin/mod.rs:206-240`) |
| NIP-05 | bound from `Host`; handle domain must equal the tenant host | ✅ `nip05.rs:36-58`, `:79-102` |
| Mesh demo | **community from the request body** | ❌ SEC-03 |

Per-tenant limiter isolation is explicitly tested: media rate + concurrency
(`media.rs:1120-1161`), invite claim (`invites.rs:481-503`), replay seen-set
(`bridge.rs:2290-2292`).

---

#### 8. SSRF, path traversal, IDOR, enumeration

##### SSRF
No outbound HTTP is issued from any of the 13 files. The only URL parsing is
`attachment_url_matches` (`admin/mod.rs:260-286`), which **compares** a stored `imeta` URL and never
fetches it — and even then it requires scheme ∈ {http, https}, normalized-host equality with the
feedback row's community host, exact hash match, a safe extension, and **no query and no fragment**
(`admin/mod.rs:281-286`; test `:483-504` covers `?token=leak`, `.thumb.jpg` substitution, an extra
path segment, and `evil.example`).

##### Path traversal
`validate_media_path` (`media.rs:547-583`) is the gate: 1–3 dot segments, first segment exactly 64
lowercase hex, `ext` matching `is_safe_ext` (1–8 chars `[a-z0-9]`, `media.rs:533-535`), 3-segment
form allowed only as `.thumb.jpg`. Tests cover `../etc/passwd`, `../{hash}.jpg`, uppercase hashes,
`_uploads/…`, `_meta/…`, `{hash}.png.metadata`, `{hash}.tar.gz`, and overlong extensions
(`media.rs:1163-1272`). Sidecar-derived extensions are re-validated before key construction
(`media.rs:875-879`) — "never trust storage as authoritative for path construction". Axum's single
path segment cannot contain `/` anyway, and the validator is asserted to hold independently of
routing (`media.rs:1250-1253`).

##### IDOR
| Vector | Control | `file:line` |
|---|---|---|
| Read another user's private snapshot via kindless `ids` | `reader_authorized_for_event` at 4 result-level checkpoints | `bridge.rs:1080`, `:1170`, `:1283`, `:1601`; test `:3125-3160` |
| Read another owner's engram via NIP-50 text match | `search_hit_accepted` re-applies `#p`/authors/channel | `bridge.rs:1590-1607`; tests `:2842-2896` |
| Read a channel you are not in | accessible-channel intersection at query build + result time | `bridge.rs:1000-1005`, `:583-588` |
| Count-based existence leak on result-gated kinds | forced per-event fallback unless `#p=[self]` is pushed down | `bridge.rs:1436-1443` |
| Claim an invite as a different pubkey | NIP-98 proves control of the claiming key; role fixed to `member` | `invites.rs:210-218`; `invite_token.rs:189-191` |
| Archive/unarchive someone else's community | `*_owned_by(host, owner, …)` requires the asserted owner to actually own it; wrong owner and unknown host both 404 | `operator.rs:234-240`, `:288-293` |
| Transfer from a stale owner | CAS on `expected_owner_pubkey` ⇒ 409 | `operator.rs:392-431` |
| Read an attachment not belonging to the named feedback | `imeta` `x` + `url` binding + community cross-check | `admin/mod.rs:216-240` |
| **Weak spot** | `list_owned_communities` returns any pubkey's communities to any allowlisted operator | `operator.rs:305-331` |

##### Enumeration surfaces
| Surface | Status | `file:line` |
|---|---|---|
| Which communities exist on a deployment | closed — unmapped host always yields the same generic 404, host never echoed | `bridge.rs:626-633`; `router.rs:288-296` |
| NIP-05 handle existence | **open by design** — but a miss, an unmapped host, and a missing `name` are indistinguishable (all `{names:{},relays:{}}` with 200), so it leaks handle existence only, and that is the point of NIP-05 | `nip05.rs:41-62` |
| Blob existence via `HEAD /media/{hash}` | open by default, but requires a 256-bit hash | `media.rs:798` |
| Workflow existence / trigger type / secret-configured state | **open** — four distinguishable outcomes on `/hooks/{id}` (SEC-08) | `bridge.rs:1787-1826` |
| Community host availability + community UUID | operator-only | `operator.rs:480-491` |
| Mesh demo route existence | leaks via the `Json` extractor rejection (SEC-03) | `mesh_demo.rs:60-70` |
| Admin route existence | 404 when `admin` unconfigured vs 403 on a non-admin host — distinguishable, but both non-informative | `admin/auth.rs:17-24` |
| Media auth failures | all 15 variants collapse to one 401 `"authentication failed"` explicitly to avoid an oracle | `buzz-media/src/error.rs:118-127` |
| Invite validity | only `Expired` distinguishable; everything else `invite_invalid` | `invites.rs:303-312` |

---

#### 9. Security headers and CORS

| Control | Coverage | `file:line` |
|---|---|---|
| `security_headers` middleware (`no-store`, `nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `CSP: default-src 'none'; frame-ancestors 'none'`) | **only** the `/api/admin/v1` sub-router, and only when `BUZZ_ADMIN_HOST` is set | `admin/mod.rs:38`, `:43-61`; `router.rs:56-59` |
| `CSP: default-src 'none'` + `nosniff` on blob responses | media GET (200 and 206) only — **not** on HEAD | `media.rs:684-686`, `:741-743` vs `:841-851` |
| `Content-Disposition: attachment` for non-inline types | media GET | `media.rs:662-668` |
| `Access-Control-Allow-Origin: *` | unconditionally on NIP-05 | `nip05.rs:65-69` |
| CORS | `CorsLayer::permissive()` when `BUZZ_CORS_ORIGINS` is empty — the **default**; an explicit list that fails to parse falls back to `CorsLayer::new()` (deny) rather than permissive, with an error log | `router.rs:373-397` |
| Body limits | 1 MiB bridge/api, `max(image, video)` = 500 MB media, 1 KiB admin, 1 MiB git policy | `router.rs:130`, `:33-46`; `admin/mod.rs:39` |

**Gap:** every non-admin HTML-bearing response — notably the operator-authored policy pages at
`invites.rs:95-124`, which return `text/html` on an unauthenticated route — receives **no**
`X-Frame-Options`, `Referrer-Policy`, or CSP. The pages do escape raw HTML in the operator Markdown
(`invites.rs:126-155`, test `:1254-1271`), so stored XSS via Markdown is closed, but there is no
header-level defence in depth and the pages are framable.

---

#### 10. Positive security properties worth preserving

1. **Fail-closed replay guard** with an infrastructure-free regression test (`bridge.rs:2321-2377`).
2. **Fail-closed rate limiter** — `AdmissionError::Unavailable` ⇒ 503, not a bypass
   (`bridge.rs:47-55`).
3. **Fail-closed tenant binding** with a uniform, non-echoing 404 on every door.
4. **Auth before body buffering** on uploads via `FromRequestParts` (`media.rs:29-38`).
5. **Sidecar-before-I/O** ordering and storage-is-not-authoritative discipline
   (`media.rs:629-660`).
6. **Read gates applied before the search branch** so NIP-50 cannot be used to bypass `#p`/author
   rules (`bridge.rs:981-998` then `:1006`).
7. **Result-level read auth** as defence in depth on all four `/query` result paths, with an
   explicit "defense-in-depth" comment at each (`bridge.rs:1076-1079`, `:1166-1169`).
8. **COUNT never silently truncates** — an over-large fallback candidate set is a 400, not a wrong
   number (`bridge.rs:1489-1497`).
9. **Attacker-controlled text truncated at a UTF-8 boundary before logging**, with a 1 MiB-input
   regression test (`bridge.rs:595-611`, `:3208-3241`).
10. **Admin errors are `&'static str` by construction**, so no dynamic text can leak through that
    surface (`admin/error.rs:12-17`).
11. **Cross-host NIP-98/NIP-42 binding** with positive *and* negative controls so neither test can
    pass vacuously (`bridge.rs:2417-2504`, `:2709-2747`).
12. **Zero `unsafe`, zero `unwrap()` outside tests**; the 5 production `expect()`s are all
    infallible-by-construction HMAC/serialize calls in `invite_token.rs`.
