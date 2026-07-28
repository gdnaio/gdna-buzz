## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Debt

---

#### 1. Hard counts

| Metric | Value |
|---|---|
| Files in scope | 13 |
| Total LOC | 9,626 |
| Production LOC (outside `#[cfg(test)]`) | **5,168** (54%) |
| Test LOC | **4,309** (45%) |
| Routed endpoints owned by this group (method × path) | **30** |
| Endpoints with zero in-repo client | **6** (all `/operator/*`) + `/_mesh/demo/echo` + `/.well-known/nostr.json` (tests only) |
| Test functions | **159** |
| `#[ignore]`d tests | **28** (18%) |
| `unsafe` blocks | **0** (the word appears once, in a doc comment at `bridge.rs:303`) |
| `unwrap()` outside `#[cfg(test)]` | **0** |
| `expect()` outside `#[cfg(test)]` | **5** — all in `invite_token.rs` (`:119`, `:139`, `:172`, `:349`, `:374`), all infallible-by-construction HMAC/serialize calls |
| TODO / FIXME / XXX / HACK markers | **1** — `media.rs:303` `TODO(v2)` persistent storage quotas |
| Production functions in `bridge.rs` | 38 |
| Longest production function | `query_events_authed`, **362 lines** (`bridge.rs:947-1308`), 53 branch keywords |
| Dead code items | **6** (see D-01…D-06) |

##### `bridge.rs` complexity profile (production region = lines 1–2183)

| Function | Lines | Range | Branch keywords (`if`/`match`/`while`/`for`) |
|---|---|---|---|
| `query_events_authed` | **362** | `bridge.rs:947-1308` | **53** |
| `count_events_authed` | 199 | `bridge.rs:1378-1576` | 25 |
| `handle_channel_window_filter` | 178 | `bridge.rs:404-581` | 14 |
| `workflow_webhook` | 152 | `bridge.rs:1762-1913` | 11 |
| `handle_bridge_search` | 134 | `bridge.rs:1616-1749` | 17 |
| `submit_event_authed` | 126 | `bridge.rs:750-875` | 8 |
| `submit_event` | 90 | `bridge.rs:613-702` | 3 |
| `synthesize_presence` | 66 | `bridge.rs:1920-1985` | 8 |
| `verify_bridge_auth_with_options` | 57 | `bridge.rs:72-128` | 5 |
| `authorize_moderation_read` | 47 | `bridge.rs:2008-2054` | 3 |

`query_events_authed` alone contains **five** sequential dispatch stages over the same
`raw_filters.iter().zip(filters.iter()).enumerate()` (window `:1017`, feed `:1029`, thread `:1112`,
catch-all build `:1188`, catch-all post-process `:1266`), coordinated by a shared
`handled: HashSet<usize>`. Each stage is independently comprehensible; the composition is not.

---

#### 2. Findings, prioritized

##### CRITICAL

**D-C1 — `require_auth_token` defaults to `false`, enabling unsigned pubkey impersonation on 6 routes**
`bridge.rs:118-127` accepts a bare `X-Pubkey: <hex>` header in place of a NIP-98 signature when the
flag is off; `config.rs:475-477` defaults it off; `bridge.rs:640`, `:908`, `:1341`, `:2033` pass it
through. Replay protection is bypassed on that path too (`bridge.rs:122-125` → `:150-153`). The
variable is absent from `.env.example`, and the startup warning
("REST API requests bypass token auth", `config.rs:588-593`) understates the effect.
*Fix:* default to `true`; make the dev path require an explicit `BUZZ_DEV_UNSAFE_HEADER_AUTH=1`; add
the var to `.env.example` with the real consequence spelled out.

##### HIGH

**D-H1 — the admin API's only credential is a spoofable `Host` header**
`admin/auth.rs:16-33`: `admin` configured → `Host == config.admin.host` → `Origin` checked **only if
present**. A non-browser client sending no `Origin` passes. Behind it sit 5 cross-tenant read
endpoints, 3 of which take no `CommunityId` at all (`admin/mod.rs:132`, `:155`, `:184`), plus raw
blob streaming (`:226`). `docs/admin/README.md:64-70` accepts this ("the human trust boundary remains
the private admin ingress") and explicitly disclaims per-operator attribution, so this is a knowing
trade — but there is no defence in depth and the access log records no principal
(`admin/mod.rs:228-233`).
*Fix:* require a shared secret or mTLS-derived header in addition to `Host`; reject requests with no
`Origin` from a browser-shaped UA; log a principal.

**D-H2 — `POST /_mesh/demo/echo` derives the tenant from the request body**
`mesh_demo.rs:50-51` + `:99-101`: `community_id` is client-supplied and used to acquire a Redis fenced
session lease. This is the only route in the module that bypasses the row-zero `Host` boundary
(`bridge.rs:621` etc.), and it has no authentication. Double-flag-gated (both default off,
`config.rs:509-518`) and self-described as "Not a product flow… stays demo-gated until it is deleted"
(`mesh_demo.rs:21-23`).
*Secondary:* the claimed 404-indistinguishability (`mesh_demo.rs:71-73`) is false — `Json<T>` is a
`FromRequest` extractor evaluated **before** the flag gate, so a malformed body returns 400/422 and
reveals the route on a mesh-off relay.
*Fix:* delete it, or gate it behind the operator allowlist and bind the tenant from `Host`; move the
body parse inside the handler so the 404 fires first.

**D-H3 — `api/events.rs` is 100% dead, and its doc comment asserts a false reason**
`api/events.rs:1-5` re-exports three bridge handlers "for backward compatibility with router.rs", but
`router.rs:71-73` binds `api::bridge::*` directly. A repo-wide grep for `api::events`,
`events::submit_event`, `events::query_events`, `events::count_events` returns **zero** hits.
*Fix:* delete the file and the `pub mod events;` line (`api/mod.rs:6`).

**D-H4 — `webhook_secret::strip_secret` has zero production callers**
`webhook_secret.rs:52-68`. Its doc says "Use this before returning a definition to API callers — the
secret must never be embedded in a response body (it is returned once, at creation time, via a
dedicated `webhook_secret` field)." Nothing in the HTTP surface calls it. The secret lives inside the
workflow definition JSON (`webhook_secret.rs:3-5`, `:34-41`), so any code path that serializes a
definition back to a client leaks it, and there is no enforcement point.
*Fix:* either wire it into every definition-returning path, or delete it and replace with a
serialization-level redaction that cannot be forgotten.

##### MEDIUM

**D-M1 — `query_events_authed` is a 362-line, 53-branch function**
`bridge.rs:947-1308`. Five dispatch stages coordinated by a mutable `handled: HashSet<usize>` and a
mutable `events: Vec<Value>` accumulator. Adding a sixth filter extension means auditing every
earlier stage's `continue`/`handled.insert` bookkeeping to be sure the new one is reachable and
mutually exclusive.
*Fix:* extract each stage into a `handle_*_filters(…) -> Result<HandledSet, …>` and drive them from a
list, so precedence is data rather than control flow.

**D-M2 — `/moderation/*` reads have no rate limit**
`authorize_moderation_read` (`bridge.rs:2008-2054`) does bind → NIP-98 → replay → capability, but never
calls `enforce_http_admission`. They are the only NIP-98 bridge routes without a limiter and return up
to 500 rows including `private_reason` (`bridge.rs:2165`). Combined with D-C1, an unmetered bulk
export of the moderation record on a default-config relay.
*Fix:* add `enforce_http_admission` to the prelude.

**D-M3 — `media_upload_rate_limiter` is an unbounded, never-swept `DashMap`**
`state.rs:38-39`, `:592`, `:774`; the only read is `media.rs:97`. No capacity, no TTL, no sweep task.
`invites.rs:36-43` documents the exact hazard for its own limiter — "a pre-membership caller can
cheaply create fresh Nostr keypairs; retaining one immortal entry per key would make the limiter
itself an unbounded-memory denial-of-service vector" — and solves it with moka (capacity 10 000 + 60 s
TTL, `state.rs:775-780`). On an open relay (default) media upload authorization is a valid Blossom
signature from any key (`media.rs:196-206`), so the same argument applies verbatim and was not applied.
*Fix:* switch to the same moka pattern; the `(u32, Instant)` value already encodes the window.

**D-M4 — `POST /api/invites/accept-policy` is an unauthenticated MAC-minting oracle**
`invites.rs:162-190`: no NIP-98, no pubkey binding, no rate limit. It signs an attacker-chosen `code`
string with `derive_invite_key(relay_keypair)` — the same key that signs invite codes
(`invite_token.rs:112-117`). Consequences: (a) the consent gate proves "someone POSTed the right
version", not "this pubkey accepted", so the acceptance persisted at `invites.rs:324-338` is not
attributable; (b) no in-payload domain-separation label distinguishes `InvitePayload` from
`PolicyAcceptancePayload` — cross-type confusion is prevented only by serde's missing-field
strictness (`invite_token.rs:64-74` vs `:335-343`), so adding an optional field to either struct
opens it.
*Fix:* require NIP-98 on `accept-policy` and bind the receipt to the signer's pubkey; add an explicit
purpose byte/label to both signed payloads.

**D-M5 — two limiters in this module are per-pod while the bridge's is cluster-wide**
`media_upload_rate_limiter` + `media_uploads_in_flight` (DashMap/`Semaphore`, `state.rs:592`, `:600`,
`:730`) and `invite_claim_rate_limiter` (moka, `state.rs:597-598`) are process-local, whereas
`admission_rate_limiter` is Redis-backed (`state.rs:712`). On N pods the effective media-upload and
invite-claim budgets are N× the configured value, and the per-pubkey upload concurrency ceiling is
N×2. No comment acknowledges the multiplication.
*Fix:* document the per-pod semantics in the config field docs, or move both to Redis.

**D-M6 — operator archive / unarchive / transfer are unattributed**
`operator.rs:355` binds the authenticated operator to `let _pubkey` and discards it;
`operator.rs:209` and `:271` call `authorize_operator_request(...).await?;` with no binding. The
resulting log lines (`operator.rs:281-282`, `:299`, `:429`) carry community + host but no actor, and
no audit entry is written. These are the two highest-impact operations on the surface (community
archival and ownership transfer). `provision_community` is the only one that threads the pubkey
through (`operator.rs:157`, `:179`).
*Fix:* include the operator pubkey in every log line and write a `buzz-audit` entry.

**D-M7 — reflected request content in 4xx bodies contradicts the log-truncation reasoning**
`bridge.rs:735-742` reasons at length that `serde_json`'s `Display` "embeds the offending input
verbatim… would otherwise reflect attacker-controlled text into a log line at full size" and fixes the
log path — then returns that same string to the caller at `bridge.rs:745`. Same pattern at `:970`,
`:1381`; `invites.rs:172-176`, `:252-257`, `:302`; `operator.rs:163-167`. And `submit_event`
truncates `IngestError::Rejected` to 256 bytes for logs (`bridge.rs:850`) but returns it untruncated
(`:855`). Impact is bounded (self-reflection only), but the defence has no response-side twin.
*Fix:* return a bounded, structured 400 (`{error, category, line, column}`) instead of the raw
`Display`.

**D-M8 — `security_headers` covers only the admin sub-router**
`admin/mod.rs:38`, `:43-61`, mounted only when `BUZZ_ADMIN_HOST` is set (`router.rs:56-59`). Every
other route — including the two unauthenticated `text/html` policy pages at `invites.rs:95-124` —
gets no `X-Frame-Options`, `Referrer-Policy`, or CSP. The policy pages do escape raw HTML in operator
Markdown (`invites.rs:126-155`, test `:1254-1271`), so stored XSS is closed, but the pages are
framable and there is no header-level defence in depth.
*Fix:* move `security_headers` (minus `no-store`) onto the merged router in `router.rs:187-190`.

**D-M9 — six routed endpoints have no in-repo client and only `#[ignore]`d coverage**
All 6 `/operator/*` routes (`router.rs:74-93`). Repo-wide grep for `operator/communities` returns only
relay source and its own tests. All **11** tests in `operator.rs` are `#[ignore = "requires Postgres"]`,
so `just test-unit` exercises **zero** operator code. The control plane that consumes them lives
outside this repository.
*Fix:* add non-ignored unit coverage for `authorize_operator_request`'s allowlist and payload-tag
logic (both are pure given a `HeaderMap`), and record the external consumer in `AGENTS.md`.

**D-M10 — media read auth is dead code in the default configuration**
`verify_blossom_get_auth` is defined in `buzz-media` (`auth.rs:207`) and its **only** repo-wide call
site is `media.rs:502`, behind `require_media_get_auth` which defaults to `false`
(`config.rs:682-689`). `authenticate_media_read` returns early at `media.rs:491-494` when off.
Additionally, the admin attachment route calls `serve_blob_for_tenant` directly
(`admin/mod.rs:226`) and so never applies the flag, while `docs/admin/README.md:44-46` says
"Existing community `/media/*` authorization is unchanged, including `BUZZ_REQUIRE_MEDIA_GET_AUTH`" —
technically true of `/media/*` but easy to misread as covering the admin route.
*Fix:* flip the default once clients are rolled out (the "staged client rollout" justification at
`config.rs:973-976` implies a planned flip that has no tracking marker in the code).

##### LOW

**D-L1 — three incompatible handler error dialects and two error-envelope shapes**
Tuple `(StatusCode, Json<Value>)` (bridge/invites/operator), `MediaError` (media), `ApiError`
(admin). Envelopes: `{"error": "<string>"}` (`api/mod.rs:19-21`) vs
`{"error":{"code","message","requestId"}}` (`admin/error.rs:16-28`). Clients cannot share an error
parser across the surface.

**D-L2 — `admin/error.rs` mints a fresh `requestId` per response**
`admin/error.rs:26`, `:71`: `uuid::Uuid::new_v4()` at serialization time, not a correlated trace id.
The field's name promises correlation it cannot deliver.

**D-L3 — operator status codes are classified by error-string prefix**
`operator.rs:180-199` matches `msg.starts_with("actor not authorized")`, `== "community already exists"`,
`starts_with("limit_reached:")`, etc. A wording change in
`handlers/community_provisioning.rs` silently reclassifies HTTP statuses.
*Fix:* return a typed error enum from the provisioning module.

**D-L4 — `/hooks/{id}` is a four-state workflow oracle**
`bridge.rs:1787-1826` distinguishes unknown workflow (404), non-webhook trigger (400), webhook with no
secret (401 with a descriptive `"re-save the workflow to generate one"`), and wrong secret (401
generic). Also, the `?secret=` fallback (`:1752-1757`, `:1809`) puts a bearer credential into access
logs and `Referer` with no warning beyond the doc comment.

**D-L5 — `/count` sums across filters without dedup**
`bridge.rs:1432`, `:1575`: an event matching two filters in one request is counted twice. `/query`
deduplicates in the feed and search paths (`bridge.rs:1069`, `:1727`) but `/count` never does.

**D-L6 — tenant-binding preamble copy-pasted nine times**
`bridge.rs:621-633`, `:888-901`, `:1321-1334`, `:1777-1786`, `:2013-2025`; `invites.rs:198-207`;
`media.rs:154-166`; `nip05.rs:31-40`. Only `media.rs` factors it
(`bind_media_read_tenant`, `media.rs:478-488`). A future change to the fail-closed behaviour must be
applied nine times.

**D-L7 — stale `#[allow(dead_code)]` on a live function**
`api/mod.rs:28` on `not_found`, which is used at `bridge.rs:1785` and `:1792`.

**D-L8 — `check_relay_membership` / `MembershipDecision` abstraction has one consumer**
`api/mod.rs:46`, `:61`; only caller is `enforce_relay_membership` at `:130`. The transport-neutral
enum buys nothing today. Its doc at `api/mod.rs:35-38` also **omits `handlers/auth.rs:217`** — the
WebSocket door — from the list of callers.

**D-L9 — `HttpAuthMethod` is write-only dead data**
`bridge.rs:830` hardcodes `Nip98` even on the `X-Pubkey` path; `HttpAuthMethod::DevPubkey`
(`handlers/ingest.rs:58`) has zero constructors repo-wide; no code reads `IngestAuth::Http.auth_method`.
So downstream logic and logs cannot distinguish a signed request from an unsigned one — which makes
D-C1 harder to detect operationally.

**D-L10 — worst-case upload memory is ~800 MB per pod**
`media.rs:362-367` buffers `max(max_image_bytes, max_file_bytes)` = 100 MB by default for every
non-video upload, and `media_max_concurrent_uploads` defaults to 8 (`config.rs:663-668`,
`state.rs:730`). Video streams to disk, so the exposure is the generic-file path only.

**D-L11 — the `TODO(v2)` storage-quota gap is real and unbounded**
`media.rs:302-304`: "Admission limits below bound active parser/storage work, but they do not cap
durable bytes stored." With `media_uploads_per_minute = 30` and `max_file_bytes = 100 MB`, one key can
durably store ~180 GB/hour per pod. The only TODO marker in all 13 files.

**D-L12 — three duplicated NIP-98 test-header builders**
`invites.rs:505-519` (`nip98_auth_header`), `operator.rs:596-616` (`nip98_auth_header` +
`nip98_auth_header_without_payload`), `bridge.rs:2380-2404` (`build_nip98_event_json` +
`nip98_auth_headers`). Near-identical bodies; a change to NIP-98 tag shape needs three edits.

**D-L13 — `#[ignore]`d tests skip vacuously in two files but panic in others**
`invites.rs:664-666`, `:772-774`, `:930-932`, `:986-988` and `operator.rs:686-688`, `:713-715`, …
use `let Some(state) = … else { return; }`, so the test **passes** when Postgres is unreachable.
`bridge.rs:3405-3407` and `invites.rs:1074-1076` correctly `panic!`/`expect` with an actionable
message. The former hides infrastructure regressions in CI.

**D-L14 — boolean env parsing is inconsistent across four spellings**
`require_media_get_auth` accepts `true`/`1`/`yes`/`on` (`config.rs:682-689`);
`require_auth_token`/`require_relay_membership`/`allow_nip_oa_auth` accept only `true`/`1`
(`config.rs:475-477`, `:483-485`, `:520-522`); mesh vars accept `on`/`true`/`1`
(`config.rs:509-518`). `BUZZ_REQUIRE_AUTH_TOKEN=yes` silently means `false`.

**D-L15 — `serve_blob_for_tenant` re-validates a path its callers already validated**
`media.rs:604`/`:619`/`:630`: `get_blob` calls `validate_media_path` then `serve_blob_for_tenant`
calls it again. Harmless, but it signals uncertainty about which layer owns the invariant — and
`admin/mod.rs:226` relies on the inner call for its own safety.

**D-L16 — `#[allow(private_interfaces)]` splits a doc comment**
`media.rs:296-304`: the attribute sits between two `///` blocks on `upload_blob`, so rustdoc renders
only the second half. Cosmetic but confusing.

**D-L17 — `nostr_nip05` folds DB errors into a 200**
`nip05.rs:64` uses a catch-all `_ =>` arm, so a Postgres failure is indistinguishable from a miss.
Deliberate for privacy (see the doc at `:31-35`) but means NIP-05 outages are invisible to callers
and unmetered.

**D-L18 — `buzz_media_uploads_total` carries an unbounded `community` label**
`media.rs:419-424` labels by tenant host, so series count grows with tenant count. Fine today,
a Prometheus-cardinality problem on a large multi-tenant deployment. Contrast the deliberate
6-value `mime` allowlist immediately above it (`:410-417`).

---

#### 3. Documentation deltas (verified against code)

| Doc claim | Reality | Evidence |
|---|---|---|
| `ARCHITECTURE.md:823` (§9 #2): "No rate limiting implementation… none are enforced" | Redis rate limiting **is** enforced on `/events`, `/query`, `/count` | `bridge.rs:24-56`, called `:760`, `:955`, `:1386`; limiter `state.rs:712` |
| `ARCHITECTURE.md:390`: "No Redis-backed rate limiter exists anywhere in the codebase" | `RedisRateLimiter` is imported from `buzz_pubsub::rate_limiter` | `state.rs:26`, `:712` |
| `ARCHITECTURE.md:459`: `buzz-pubsub` "Does NOT implement the rate limiter" | it does | `state.rs:26` |
| `ARCHITECTURE.md:824` (§9 #3): "`/api/presence` returns online/away status only" | no such route exists; presence is synthesized inside `POST /query` | `router.rs:32-190`; `bridge.rs:1920-1985`; removal noted at `mobile/test/features/profile/presence_cache_provider_test.dart:13` |
| `ARCHITECTURE.md:623`: `PUT /media/upload` "50 MB limit" | route body limit is `max(max_image, max_video)` = **500 MB** default | `router.rs:33-36`; `config.rs:657-672` |
| `ARCHITECTURE.md:610-628` endpoint table | omits `PUT /upload`, 6 operator routes, 6 invite/policy routes, 3 moderation routes, `/_mesh/demo/echo`, `/huddle/{channel_id}/audio`, 5 admin routes, `/_status`, `/_mesh` | `router.rs:39`, `:74-128`, `:57-59`, `:229-230` |
| `AGENTS.md` "Nostr-first HTTP surface" / "deliberately narrow" list | **14 routed endpoints in this group** sit outside the documented set (all operator, all invite/policy, all moderation, mesh demo) | enumerated in the api-surface aspect §4 |
| `api/events.rs:3`: "re-exports bridge handlers for backward compatibility with router.rs" | `router.rs:71-73` uses `api::bridge::*`; zero references to `api::events` repo-wide | `api/events.rs:1-5` |
| `api/mod.rs:35-38`: membership gate "Called by `media.rs`, `bridge.rs`, `git/transport.rs`, and `audio/handler.rs`" | omits `handlers/auth.rs:217`, the WebSocket door | `handlers/auth.rs:217` |
| `webhook_secret.rs:52-56`: `strip_secret` "Use this before returning a definition to API callers" | zero production callers | `webhook_secret.rs:57` |
| `mesh_demo.rs:71-73`: "404 (not 403) so a non-demo deployment is indistinguishable from one without the route" | the `Json` extractor rejects malformed bodies with 400/422 before the gate | `mesh_demo.rs:60-62` |
| `bridge.rs:1-4`: "authenticated via NIP-98 signed events" | also accepts the unsigned `X-Pubkey` header by default | `bridge.rs:118-127`; `config.rs:475-477` |
| p-gate parity comment "same enforcement as WS REQ handler" (`bridge.rs:979-980`, `:1403`) | **HTTP is stricter**: unconditional, vs WS which gates only `channel_id.is_none()` | `bridge.rs:981-998`, `:1404-1421` vs `handlers/req.rs:183-205` |
| `handlers/ingest.rs:57-58`: `DevPubkey` "X-Pubkey dev-mode header (backward compat during transition)" | never constructed; `bridge.rs:830` always reports `Nip98` | `handlers/ingest.rs:58` |
| `docs/admin/README.md:44-46`: "Existing community `/media/*` authorization is unchanged, including `BUZZ_REQUIRE_MEDIA_GET_AUTH`" | true of `/media/*`, but the admin attachment route bypasses `authenticate_media_read` entirely | `admin/mod.rs:226`; `media.rs:619` |
| `.env.example` | omits 11 vars this module consumes, including `BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_ADMIN_HOST`, and both `RELAY_OPERATOR_*` | see the configuration aspect §4 |

---

#### 4. Suggested ordering

1. **D-C1** — flip `require_auth_token` to default `true` and document it. Single highest-value change.
2. **D-H1**, **D-H2** — add a real credential to the admin surface; delete or gate the mesh demo.
3. **D-H3**, **D-H4** — delete `api/events.rs`; resolve `strip_secret` (wire it or replace it).
4. **D-M2**, **D-M3**, **D-M4** — add the moderation rate limit; bound the media limiter map;
   authenticate `accept-policy` and add a domain-separation label.
5. **D-M1** — decompose `query_events_authed` before the next filter extension lands.
6. **D-M6**, **D-M8**, **D-M10** — attribute operator mutations; widen `security_headers`; plan the
   `require_media_get_auth` flip.
7. Correct the `ARCHITECTURE.md` / `AGENTS.md` / `.env.example` deltas in §3 — five of them
   (rate limiting ×3, `/api/presence`, the 50 MB claim) actively mislead anyone reasoning about
   the relay's security posture.
