## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Business Rules

Numbered BR-API-nn. Every rule cites the enforcing line.

---

#### A. Cross-cutting request pipeline

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-01 | Every tenant-scoped HTTP door binds the community from the `Host` header **before** any auth or DB work; unmapped host or lookup failure fails closed with a generic 404 that never echoes the host. | `crate::tenant::bind_community` | `bridge.rs:626-633`, `:894-901`, `:1327-1334`, `:1783-1785`, `:2018-2025`; `invites.rs:200-207`; `media.rs:158-166`, `:480-487` |
| BR-API-02 | The NIP-98 `u`-tag expectation is built from `tenant.host()`, never `config.relay_url`'s host — closing both cross-host token reuse and false rejections on multi-tenant deployments. | `nip98_expected_url` | `bridge.rs:195-206`; tests `:2417-2449`, `:2636-2654` |
| BR-API-03 | For query-bearing NIP-98 GETs the expected URL appends the **verbatim** raw query string (not a re-serialized parse), so param order and encoding stay byte-exact with what the client signed. | `authorize_moderation_read`, `authorize_operator_request` | `bridge.rs:2027-2031`; `operator.rs:73-77`; tests `bridge.rs:2529-2630` |
| BR-API-04 | Every NIP-98 request is checked against a shared, community-scoped Redis seen-set (`SET NX EX`, TTL `DEFAULT_REPLAY_TTL_SECS`); a second presentation of the same event id is 401. | `check_nip98_replay` → `check_nip98_replay_with_guard` | `bridge.rs:136-176`; tests `:2262-2317`, `invites.rs:1103-1188` |
| BR-API-05 | A replay-guard **error** (Redis down) rejects the request — fail closed, never admit. | `Err(e) => 401 "replay check unavailable"` | `bridge.rs:167-176`; operator sibling `operator.rs:126-137`; no-infra test `bridge.rs:2340-2377` |
| BR-API-06 | Replay detection is **skipped** when the event id is the zero hash — the sentinel written by the `X-Pubkey` dev path. | early `return Ok(())` | `bridge.rs:150-153`, sentinel produced at `:122-125` |
| BR-API-07 | `X-Pubkey: <hex>` is accepted in place of a NIP-98 signature **only** when `require_auth_token == false`. | `if !require_auth_token` | `bridge.rs:118-127` |
| BR-API-08 | Bodies on the three bridge POSTs are capped at 1 MiB; media routes at `max(max_image, max_video)`; admin at 1024 bytes; git policy at 1 MiB. | `RequestBodyLimitLayer` | `router.rs:130`, `:33-46`; `admin/mod.rs:39`; `git/mod.rs:63` |

#### B. Rate limiting and admission

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-09 | `POST /events`, `/query`, `/count` consume one `LimitType::ApiCalls` token per request against a **Redis** fixed 60 s window sized by `human_api_calls_per_min` (default 300). | `enforce_http_admission` → `admission::check_principal` | `bridge.rs:24-56`, called `:760`, `:955`, `:1386`; limiter `state.rs:712`; default `buzz-auth/src/rate_limit.rs:113-115` |
| BR-API-10 | Quota exceeded ⇒ **429** with `retry in {reset_in_secs}s`; limiter unavailable ⇒ **503** — i.e. the limiter fails **closed**. | `AdmissionError` match | `bridge.rs:39-55`; mapping `admission.rs:24-33` |
| BR-API-11 | Admission and replay run **before** body parse, so a 429/replay reject on a malformed body is still attributed. | ordering in `submit_event_authed` | `bridge.rs:758-770` |
| BR-API-12 | The three `/moderation/*` GETs have **no** admission check — they are the only NIP-98 bridge routes without a rate limit. | absence in `authorize_moderation_read` | `bridge.rs:2008-2054` |
| BR-API-13 | Media upload enforces a **process-local** fixed 60 s window of `media_uploads_per_minute` (default 30) per `(community, pubkey)`. | `upload_rate_limited` | `media.rs:88-111`, `:66`; called `media.rs:215` |
| BR-API-14 | Media upload enforces two concurrency ceilings: a global `Semaphore(media_max_concurrent_uploads)` (default 8) and a per-`(community,pubkey)` counter ≤ `media_max_concurrent_uploads_per_pubkey` (default 2, clamped to the global). | `acquire_upload_permit` | `media.rs:113-136`; state `state.rs:730`, `:781`; clamp `config.rs:669-675` |
| BR-API-15 | Both media limits are **per pod** (DashMap / in-process Semaphore), not cluster-wide — unlike the bridge's Redis limiter. | types | `state.rs:38-39`, `:523`, `:592`, `:600` |
| BR-API-16 | `/api/invites/claim` allows ≤10 attempts per `(community, pubkey)` per 60 s, evaluated **before** code verification; over-limit ⇒ 429. | `claim_rate_limited` → `claim_key_rate_limited` | `invites.rs:37`, `:293-298`, `:374-390`; test `:1194-1252` |
| BR-API-17 | The claim limiter is bounded by capacity (10 000) **and** TTL (60 s) because a pre-membership caller can mint keypairs freely — an unbounded map would itself be the DoS. | moka builder | `invites.rs:36-43`; `state.rs:775-780`; tests `invites.rs:459-503` |
| BR-API-18 | `POST /api/invites/accept-policy`, `GET /api/join-policy*`, and `GET/HEAD /media/*` have **no** rate limit at all. | absence | `invites.rs:75`, `:95`, `:104`, `:162`; `media.rs:604`, `:798` |

#### C. `POST /events` (ingest bridge)

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-19 | Relay membership is enforced for the submitting pubkey, with a NIP-OA owner-delegation fallback read from `X-Auth-Tag`. | `enforce_relay_membership` | `bridge.rs:798-819`; gate `api/mod.rs:124-149` |
| BR-API-20 | On open relays (`require_relay_membership=false`) the membership check short-circuits, but a NIP-OA tag is still opportunistically parsed so the agent→owner edge can be materialized. The NIP-OA signature is self-proving, so no feature flag guards this. | `extract_nip_oa_owner` | `bridge.rs:806-813`; `api/mod.rs:151-170` |
| BR-API-21 | A verified agent→owner mapping is persisted first-write-wins; an existing mapping is accepted only if it names the same owner. Both principals are `ensure_user`'d first because the FK is community-scoped. | `materialize_nip_oa_owner` | `api/mod.rs:174-224` |
| BR-API-22 | HTTP ingest is granted `Scope::all_known()` — **all 16 scopes**, including `AdminChannels`/`AdminUsers`; channel authority comes from membership, not scopes. | `IngestAuth::Http` | `bridge.rs:827-831`; `buzz-auth/src/scope.rs:68-87` |
| BR-API-23 | `auth_method` is hardcoded `HttpAuthMethod::Nip98` regardless of whether the caller actually signed — the `X-Pubkey` path is indistinguishable downstream. | literal | `bridge.rs:830`; `DevPubkey` never constructed (`handlers/ingest.rs:58`) |
| BR-API-24 | `IngestError` maps: `Rejected`→400, `AuthFailed`→403, `Internal`→500; each increments `buzz_events_rejected_total{transport="http", reason=…}`. | match arms | `bridge.rs:842-871` |
| BR-API-25 | Rejection reasons can embed event-controlled text, so they are truncated to 256 bytes at a UTF-8 char boundary **before logging** — but the untruncated message is still returned in the HTTP body. | `truncate_reason` | `bridge.rs:595-611`, `:850-856`; tests `:3208-3241` |
| BR-API-26 | `serde_json` parse failures log only `category`/`line`/`column`, never the error's Display string (which embeds the offending input). | `ParseFail` variant | `bridge.rs:727-748` |
| BR-API-27 | Exactly one terminal `"HTTP bridge request"` log line is emitted per authenticated request, covering every outcome including early admission/replay/membership failures. | `SubmitOutcome` + thin wrapper | `bridge.rs:646-700`, `:704-747`; tests `:3586-3688` |

#### D. `POST /query` — read authorization

| ID | Rule | Enforcement | `file:line` |
|---|---|---|---|
| BR-API-28 | P-gated kinds (gift wraps, member notifications, observer frames) require the caller's own pubkey in `#p`; violation ⇒ **403**. | `p_gated_filters_authorized` | `bridge.rs:981-986`; `handlers/req.rs:1042` |
| BR-API-29 | Agent-engram reads require `authors=[self]` or `#p=[self]`; violation ⇒ 403. | `engram_filters_authorized` | `bridge.rs:987-992` |
| BR-API-30 | Author-only kinds require `authors=[self]`; violation ⇒ 403. | `author_only_filters_authorized` | `bridge.rs:993-998` |
| BR-API-31 | **The three gates above are applied unconditionally on HTTP.** The WS REQ path applies the identical three gates only when `channel_id.is_none()` (global subscriptions), because channel-scoped subs cannot receive globally-stored events. HTTP is therefore strictly stricter. | comparison | `bridge.rs:981-998`, `:1404-1421` vs `handlers/req.rs:183-205` |
| BR-API-32 | The caller's accessible-channel set is resolved once per request and every result is re-checked against it; channel-less (global) events are always allowed through. | `get_accessible_channel_ids_cached` + `event_in_accessible_channel` | `bridge.rs:1000-1005`, `:583-588` |
| BR-API-33 | Filters naming a single inaccessible `#h` channel are silently skipped (empty contribution), never an error. | `continue` | `bridge.rs:1141-1145`, `:1193-1196`, `:449-451` |
| BR-API-34 | Every delivered event passes `reader_authorized_for_event` — result-level read auth for viewer-private kinds (e.g. kind:30622 DM-visibility, kind:44200) even when reached via a kindless `ids` filter. | 4 call sites | `bridge.rs:1080-1083`, `:1170-1174`, `:1283-1288`, `:1601-1603`; test `:3125-3160` |
| BR-API-35 | Author-only events authored by someone else are dropped post-query. | `is_author_only_event` | `bridge.rs:1289-1291`, `:1723-1725` |
| BR-API-36 | Filter dispatch precedence is fixed: channel-window (`top_level`) → feed (`feed_types`) → thread (`depth_limit` + one `#e`) → catch-all. A filter handled by an earlier stage is never re-processed. | `handled: HashSet<usize>` | `bridge.rs:1017-1027`, `:1029`, `:1112`, `:1188` |
| BR-API-37 | Catch-all filters are built and validated in filter order (phase 1) **before** any DB read, so deterministic client errors surface ahead of transient DB errors; DB reads then run bounded-concurrent (`FILTER_QUERY_CONCURRENCY=4`) while post-processing preserves filter order. | 3-phase pipeline | `bridge.rs:1186-1264`; `handlers/req.rs:35` |
| BR-API-38 | `before_id` must be 64 hex chars — malformed ⇒ **400**, never demoted to "absent". It also requires `until` to be set ⇒ 400 otherwise. | `BeforeId::Malformed` | `bridge.rs:273-291`, `:1198-1216`; tests `:2940-2977` |
| BR-API-39 | Extension flags opt in only on a literal JSON `true`; `"true"`, `1`, and absent all read false, so a malformed filter degrades to a normal query rather than a wrong window. | `extension_flag` | `bridge.rs:295-297`; test `:2979-3000` |

##### Channel-window sub-rules (`top_level: true`)

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-40 | Requires exactly one `#h` channel ⇒ else 400. | `bridge.rs:414-419` |
| BR-API-41 | Cursor is composite: `until` **and** `before_id` together, or neither ⇒ else 400. No timestamp-only fallback (that ambiguity is the dense-second dup/loss bug). | `bridge.rs:436-458` |
| BR-API-42 | `until` out of `DateTime` range ⇒ 400. | `bridge.rs:441-444` |
| BR-API-43 | Row budget = `min(limit, 200)`, default 50, floor 1. Summary/bounds overlays and the aux closure do **not** consume it. | `bridge.rs:374-375`, `:460-465` |
| BR-API-44 | Aux closure walks exactly two hops: reactions/deletions/edits targeting retained rows, then deletions targeting those aux events. Each hop bounded at 1000 rows; duplicates deduped by event id. | `bridge.rs:379-390`, `:483-521` |
| BR-API-45 | Aux events are access-**checked** rather than channel-constrained, because deletions can be stored channel-less. | `bridge.rs:504-508` |
| BR-API-46 | Exactly one kind:39006 window-bounds overlay per response is the sole authority on exhaustion — `rows < limit` proves nothing on an exact-multiple final page. | `bridge.rs:558-576` |
| BR-API-47 | Overlays (39005, 39006) and synthesized presence (20001) are signed with `state.relay_keypair`. | `bridge.rs:523-527`, `:1972` |

##### Feed / thread / presence sub-rules

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-48 | Feed limit is `min(limit, 100)` default 20, shared across all requested feed types; `agent_activity` canonicalizes to `activity`; duplicate types deduped; unknown types silently ignored. | `bridge.rs:1035-1064` |
| BR-API-49 | Thread reads require exactly one `#e` value that decodes to 32 bytes; limit `min(limit, 500)` default 100. | `bridge.rs:1126-1140`, `:1147-1151` |
| BR-API-50 | Thread cursor decodes to BE-i64 seconds optionally followed by raw event-id bytes; a malformed id hex degrades to timestamp-only rather than erroring. | `bridge.rs:305-345`; test `:3072-3082` |
| BR-API-51 | If **every** filter targets only kind:20001 or kind:40902 **with** non-empty `authors`, the request is served entirely from Redis presence and never touches Postgres. Any other filter shape falls through. | `synthesize_presence` | `bridge.rs:1920-1985` |
| BR-API-52 | Presence lookup failure degrades to an empty result (`unwrap_or_default`), not an error. | `bridge.rs:1951-1957` |

##### NIP-50 search routing on `/query`

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-53 | Any filter with `search` routes the whole request to `buzz-search` (Postgres FTS); mixing search and non-search filters in one request ⇒ **400**. | `bridge.rs:1006-1021`, `:1578-1580` |
| BR-API-54 | The three read gates (BR-API-28/29/30) run **before** the search branch, so search cannot be used to harvest p-gated or engram kinds. | `bridge.rs:981-998` then `:1006` |
| BR-API-55 | Channel scope: intersect the filter's `#h` values with accessible channels — all-inaccessible ⇒ skip that filter; no `#h` ⇒ community-wide scope with `include_global = true`; no scope at all ⇒ return `[]`. | `bridge.rs:1622-1650` |
| BR-API-56 | Only kind/authors/since/until are pushed down to FTS; **every** other filter constraint (`#p`, `#h`, `#e`, `#d`, `ids`, …) is re-enforced against the full stored event by `search_hit_accepted`, plus channel scope and `reader_authorized_for_event`. | `bridge.rs:1590-1607`, `:1652-1671`, `:1717-1719`; tests `:2842-2934`, `:3125-3160` |
| BR-API-57 | Per-filter FTS page size is `min(limit, 500)`; `limit == 0` skips the filter; `page` defaults to 1 and must be > 0. | `bridge.rs:1664-1668`, `:380-388` |
| BR-API-58 | FTS relevance ordering is preserved by iterating hit ids and looking each up in a map built from the DB fetch; results are deduped across filters. | `bridge.rs:1700-1735` |
| BR-API-59 | On non-search queries, `page` (1-based) becomes SQL `OFFSET = (page-1)*limit`, but only when a positive `limit` is present; page ≤ 1 leaves the default offset untouched. | `bridge.rs:390-410`, `:1218-1229`; tests `:3002-3030` |

#### E. `POST /count` semantics

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-60 | The same three read gates apply, unconditionally, as on `/query`. | `bridge.rs:1404-1421` |
| BR-API-61 | Filters naming an inaccessible `#h` channel are skipped (contribute 0), not an error. | `bridge.rs:1454-1457` |
| BR-API-62 | With no `#h`, `query.channel_ids` is set to the accessible set so the scope is pushed into SQL, and `limit` is cleared for the pushdown path. | `bridge.rs:1524-1526`, `:1544` |
| BR-API-63 | SQL `COUNT(*)` pushdown is used only when **all three** hold: `filter_fully_pushable`, author-only filtering unnecessary or `authors == [self]`, and result-gated filtering unnecessary. | `bridge.rs:1467-1479`, `:1535-1543` |
| BR-API-64 | Result-gated kinds (44200 / 30622) force the per-event fallback unless `#p=[self]` is safely pushed down — otherwise the count itself leaks existence. | `filter_can_match_result_gated_kinds` + `result_gated_count_safe_for_pushdown` | `bridge.rs:1436-1443` |
| BR-API-65 | The fallback path bounds its candidate set (`apply_count_fallback_limit`); if the candidate set is exceeded, the request is **rejected 400** `"count filter requires narrower constraints"` and `buzz_count_fallback_rejections_total` is incremented — the count is never silently truncated. | `bridge.rs:1489-1497`, `:1551-1559` |
| BR-API-66 | Fallback counting applies `filters_match` + `is_author_only_event` + `reader_authorized_for_event` per candidate before incrementing. | `bridge.rs:1499-1517`, `:1561-1571` |
| BR-API-67 | Response is the summed total across all filters — **not** deduplicated across overlapping filters, so an event matching two filters is counted twice. | `bridge.rs:1432`, `:1575` |

#### F. Media upload / download

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-68 | Tenant binding runs **first** in the extractor, before Blossom verification, so the `server` tag is validated against the bound tenant host rather than a process-global domain — and it reads only the `Host` header, preserving the pre-body rejection guarantee. | `media.rs:145-167` |
| BR-API-69 | All upload authorization happens in `FromRequestParts`, i.e. before the body is buffered, so an unauthenticated caller cannot force the server to buffer up to the body limit. | `media.rs:29-38`, `:140-234` |
| BR-API-70 | Only `/upload` and `/media/upload` are valid upload paths; anything else ⇒ `MediaError::NotFound`. | `media.rs:57-63` |
| BR-API-71 | `X-SHA-256` is **mandatory** (BUD-11) and must be exactly 64 lowercase hex chars. | `media.rs:169-186` |
| BR-API-72 | `X-SHA-256` must match at least one `x` tag in the Blossom auth event, else `HashMismatch` (→401). | `media.rs:188-194` |
| BR-API-73 | Relay membership is the **only** upload authority — independent of bearer tokens and of `require_auth_token`. On open relays any valid Blossom signer may upload. | `media.rs:196-219` |
| BR-API-74 | The extractor verifies Blossom auth with the permissive 3600 s window because the content type is not yet known; `buzz-media` re-verifies with the per-type window (600 s images / 3600 s video) after the SHA-256 is computed. | `media.rs:171-177` |
| BR-API-75 | Content type is decided from **actual bytes** (a bounded 4096-byte sniff, replayed into the pipeline so the hashed body stays byte-identical), never from `Content-Type`. | `media.rs:319-336` |
| BR-API-76 | Video takes a streaming-to-disk path (never fully buffered); non-video buffers up to `max(max_image_bytes, max_file_bytes)` and then splits image vs generic-file by sniffed MIME. | `media.rs:338-400` |
| BR-API-77 | The legacy `/media/upload` alias accepts images only — any other sniffed type ⇒ **415** `DisallowedContentType`; it also increments `buzz_media_legacy_upload_route_total`. | `media.rs:313-317`, `:384-388` |
| BR-API-78 | Descriptor URLs are rewritten to the bound tenant host before returning, with the extension validated by `is_safe_ext` and defaulted to `bin`. | `media.rs:402-407`, `:458-476` |
| BR-API-79 | Upload MIME metric labels are normalized to a 6-value allowlist to bound cardinality. | `media.rs:410-424` |
| BR-API-80 | A `MediaUploaded` audit entry is sent over the bounded audit channel; a closed channel logs and increments `buzz_audit_send_errors_total` but does **not** fail the upload. | `media.rs:426-445` |
| BR-API-81 | Read auth (`GET`/`HEAD`) is applied only when `require_media_get_auth == true`: Blossom `get` verb + `x`-tag hash binding + `server` = tenant host + relay membership. Default is **off**. | `media.rs:489-514`; `config.rs:682-689` |
| BR-API-82 | Path validation runs before any auth or storage lookup and rejects anything that is not `{64-hex}[.ext][.thumb.jpg]`; traversal, uppercase hashes, `_uploads/`/`_meta/` keys, and >3 segments are all 404. | `media.rs:547-583`; tests `:1163-1272` |
| BR-API-83 | The sidecar is consulted **before** any blob I/O and is authoritative for content type and extension; storage metadata is never trusted. A missing sidecar is 404, not a metadata fallback. | `media.rs:629-660`, `:794-836` |
| BR-API-84 | For explicit `{hash}.{ext}` requests, `requested_ext` must equal `sidecar.ext` or the response is 404. | `media.rs:646-658`, `:820-834` |
| BR-API-85 | Non-inline content types are served `Content-Disposition: attachment` with `nosniff` and `CSP: default-src 'none'` — the primary defence against an uploaded file rendering as active content. | `media.rs:662-668` |
| BR-API-86 | Range handling: no `Range` ⇒ 200 streamed; single range ⇒ 206 capped at 16 MiB; `start >= total` or unparseable ⇒ **416** with `Content-Range: bytes */TOTAL`; multi-range (comma) is ignored and the full body served. | `media.rs:670-757` |
| BR-API-87 | Suffix ranges (`bytes=-N`) are supported and clamped; `bytes=-0` and any suffix on an empty file ⇒ `None` ⇒ 416; `start > end` ⇒ 416. | `media.rs:759-792`; tests `:1310-1370` |
| BR-API-88 | Blob content hash is verified **on upload** (`buzz-media/src/upload.rs:83`, `:392-393`, `auth.rs:195`) and **not** re-verified on read — reads trust the content-addressed key. | see cited lines |

#### G. Invite issuance, redemption, expiry

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-89 | Minting requires the caller to hold role `owner` or `admin` in the bound community — mirroring the kind:9030 authz; absent membership defaults to `""` and is rejected 403. | `invites.rs:233-247` |
| BR-API-90 | Invite and claim endpoints always require a real NIP-98 signature **and** a `payload` tag covering the body — no `X-Pubkey` fallback, regardless of `require_auth_token`. | `invites.rs:210-218`; payload check `bridge.rs:97-108` |
| BR-API-91 | `ttl_secs` is clamped to `[60, 30 days]`; default 72 h. | `invite_token.rs:129`, `:52`, `:55` |
| BR-API-92 | The invite role is fixed to `"member"` at mint **and re-checked on verify**, so a hand-crafted correctly-signed payload with an elevated role is still rejected. | `invite_token.rs:135`, `:189-191`; test `:311-331` |
| BR-API-93 | Verification order is MAC-first: nothing in the payload is trusted before `verify_slice` succeeds. | `invite_token.rs:171-176` |
| BR-API-94 | A code minted for community A fails on community B on the same deployment. | `invite_token.rs:186-188`; endpoint test `invites.rs:1004-1046` |
| BR-API-95 | `Expired` is the only distinguishable rejection (403 `invite_expired`); every other failure collapses to 403 `invite_invalid` so the endpoint is a poor forging oracle. | `invites.rs:303-312`; test `invites.rs:1071-1101` |
| BR-API-96 | Claim is idempotent: a repeat claim by an existing member returns 200 `already_member` and publishes nothing. | `invites.rs:340-366`; test `invites.rs:704-713` |
| BR-API-97 | On first insert, NIP-43 member-added and membership-list events are published best-effort — a publish failure is logged, never converted into an HTTP error. | `invites.rs:344-355` |
| BR-API-98 | The claim endpoint is **deliberately exempt** from the relay-membership gate; NIP-98 proves control of the joining key and the HMAC proves an admin authorized the join. | `invites.rs:8-13`, `:291-293` |
| BR-API-99 | Codes are multi-use until expiry — there is no server-side "used" bit; revocation is coarse (rotate the relay keypair or remove the member). | `invite_token.rs:32-34`, `:43-46` |
| BR-API-100 | Landing-page URL scheme follows the deployment's TLS posture (`wss://` ⇒ `https`, else `http`), same rule as `nip98_expected_url`. | `invites.rs:266-270` |

#### H. Join-policy acceptance

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-101 | The policy version is a SHA-256 over `terms ‖ 0x00 ‖ privacy ‖ 0x00 ‖ age_required`, so editing any document invalidates every outstanding receipt. | `config.rs:794-811` |
| BR-API-102 | `accept-policy` requires `policy_version == configured version` **and**, if `age_attestation_required`, `age_confirmed == true`; else 400 `join_policy_not_accepted`. | `invites.rs:176-184` |
| BR-API-103 | The receipt binds `hex(sha256(code))`, the policy version, and a 10-minute expiry, HMAC'd with the derived invite key. | `invite_token.rs:346-359` |
| BR-API-104 | When a join policy is configured, `claim` **requires** a receipt; absent ⇒ 403 `join_policy_required`; forged/cross-invite/stale-version ⇒ same 403. | `invites.rs:314-322`; tests `invites.rs:790-901` |
| BR-API-105 | The accepted policy version is persisted alongside the membership row on successful claim. | `invites.rs:324-338`; assertion `invites.rs:967-978` |
| BR-API-106 | `accept-policy` has **no caller authentication and no rate limit** — anyone who knows a code can mint a receipt, so the gate proves "someone clicked", not "this pubkey accepted". | `invites.rs:162-190` |
| BR-API-107 | Operator-supplied policy Markdown is rendered with raw HTML escaped to text, so the endpoint cannot serve arbitrary operator markup; the (literal) title is escaped too. | `invites.rs:126-155`; test `invites.rs:1254-1271` |
| BR-API-108 | Policy documents 404 until configured; the JSON endpoint returns `{}` rather than 404. | `invites.rs:112-124`, `:75-90` |
| BR-API-109 | The join policy is **deployment-global** config, not per-community — every tenant on a deployment shares one policy and version. | `config.rs:794-811`; read at `invites.rs:76`, `:171`, `:314` |

#### I. Moderation queue reads

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-110 | Authorization goes through the single capability helper with `ModerationAction::ViewQueue` and `ModerationTarget::None`, never an inline role check. | `bridge.rs:2036-2052` |
| BR-API-111 | `ViewQueue` is granted to community `owner` and `admin` only — channel-level roles do **not** confer it. | `handlers/moderation_authz.rs:83-96`; tests `:297-320` |
| BR-API-112 | Every authorization failure collapses to 403 `restricted: moderator access required` — the underlying reason is discarded. | `bridge.rs:2045-2052` |
| BR-API-113 | Row counts are clamped to `1..=500`; a non-positive or absent `limit` yields the maximum 500. | `bridge.rs:2059`, `:2066-2072` |
| BR-API-114 | Queue reads are community-wide with no channel context. | `bridge.rs:1998-2000` |

#### J. Workflow webhook

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-115 | The `Host` header — not the workflow row — determines the tenant, so a request for community A can only reach A's workflows even when the same UUID exists in B. | `bridge.rs:1773-1793` |
| BR-API-116 | The workflow must have `TriggerDef::Webhook`, else 400. | `bridge.rs:1798-1803` |
| BR-API-117 | Secret precedence is header (`X-Webhook-Secret`) then `?secret=`; comparison is XOR-fold constant-time after a length check. | `bridge.rs:1805-1813`; `webhook_secret.rs:70-90` |
| BR-API-118 | A workflow with **no** stored secret is rejected 401 with an explanatory message rather than allowed through. | `bridge.rs:1820-1826` |
| BR-API-119 | The response is **202 Accepted** with a `run_id`; execution is spawned detached and its outcome never affects the HTTP status. | `bridge.rs:1866-1913` |
| BR-API-120 | Every top-level key of the JSON body becomes a `webhook_fields` string (non-strings via `to_string()`); the workflow's own `channel_id` seeds the trigger context. | `bridge.rs:1828-1855` |
| BR-API-121 | A definition that fails to re-parse inside the spawned task marks the run `Failed` with the parse error, so the run row is never left `Pending`. | `bridge.rs:1873-1893` |

#### K. Operator control plane

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-122 | Operator requests authenticate against the configured `relay_operator_api_origin`, **not** the inbound `Host`, and never bind a tenant. | `operator.rs:57-60`, `:66-77` |
| BR-API-123 | A `payload` tag is required iff a body is present (`body.is_some()`), so GETs are exempt. | `operator.rs:84` |
| BR-API-124 | Operator replay uses a dedicated deployment-global scope `"operator-management"` rather than a community scope. | `operator.rs:55`, `:104-122` |
| BR-API-125 | The signer must appear in `relay_operator_pubkeys` (exact hex match) or 403. | `operator.rs:86-98` |
| BR-API-126 | If `relay_operator_api_origin` is unset the request fails with **500**, not 403/404 — and config refuses to start with pubkeys but no origin. | `operator.rs:69-72`; `config.rs:577-581` |
| BR-API-127 | Hosts are normalized and canonically validated (normalized-shape equality, no scheme/path/query/userinfo, valid domain labels, ≤253 bytes) before create; availability accepts normalizable-but-non-canonical input. | `handlers/community_provisioning.rs:83-165`, `:167+` |
| BR-API-128 | All pubkeys are validated as exactly 64 hex chars, lowercased. | `handlers/community_provisioning.rs:71-75`; call sites `operator.rs:227`, `:281`, `:317`, `:381`, `:389` |
| BR-API-129 | The deployment's own community host cannot be archived ⇒ 409. | `operator.rs:220-226` |
| BR-API-130 | Archive/unarchive require the asserted `owner_pubkey` to actually own the community; a wrong owner and a nonexistent host both return the same 404 `community not found`. | `operator.rs:234-240`, `:288-293`; test `operator.rs:900-919` |
| BR-API-131 | Archive must propagate a cluster-wide disconnect; a propagation failure returns **503** carrying the archived state and `propagation:"pending"` so the caller retries — the DB mutation is already committed and idempotent. | `operator.rs:241-264`; test `operator.rs:960-1029` |
| BR-API-132 | Transfer is compare-and-swap on `expected_owner_pubkey`; mismatch ⇒ 409 `owner_conflict`, no owner ⇒ 404, transferee at the per-owner cap ⇒ 409 `limit_reached`. | `operator.rs:398-431` |
| BR-API-133 | The previous owner is demoted to `member`, not `admin`. | `operator.rs:348`; test `operator.rs:1148-1157` |
| BR-API-134 | Post-transfer NIP-43 snapshot publication is best-effort and only attempted when `require_relay_membership`; failure logs a warning and still returns 200. | `operator.rs:433-457` |
| BR-API-135 | Provisioning error strings are mapped by prefix: `actor not authorized`→403, `community already exists`/`limit_reached:`→409, persistence prefixes→500 (generic body), everything else→400 (message passed through). | `operator.rs:180-199` |

#### L. Deployment-admin reads

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-136 | Every handler calls `authorize` as its first statement, before any DB access. | `admin/mod.rs:100`, `:130`, `:155`, `:182`, `:196`; test `:377-391` |
| BR-API-137 | `authorize` requires exact `Host` equality with `config.admin.host`, and — only when `Origin` is present — an origin whose host matches. A request with **no** `Origin` passes. | `admin/auth.rs:16-33` |
| BR-API-138 | `limit` must be `1..=200`, else 400 `invalid_limit`; `status` and `target_kind` are allowlisted. | `admin/mod.rs:75-91`, `:101-112` |
| BR-API-139 | Feedback listing is a hard-coded 100 rows with no client-controllable limit. | `admin/mod.rs:151-156` |
| BR-API-140 | Feedback summaries strip Markdown lines whose link target matches an `imeta` attachment URL, then truncate to 240 **chars** (Unicode-safe) with `…`. | `admin/mod.rs:295-322`; tests `:441-458` |
| BR-API-141 | Attachment reads require: 64-lowercase-hex path, an existing feedback row, an `imeta` tag whose `x` equals the requested hash **and** whose `url` resolves to the same community's `/media/{hash}.{safe-ext}` with no query/fragment/extra path. All failures collapse to 404. | `admin/mod.rs:196-240`, `:241-286`; tests `:460-506` |
| BR-API-142 | The tenant for an attachment read is derived from server-owned provenance (`feedback.community_host`), and the resolved community must still equal `feedback.community_id` — otherwise 404 plus a warning. | `admin/mod.rs:206-224` |
| BR-API-143 | Attachment reads go through `serve_blob_for_tenant`, which **bypasses** `authenticate_media_read` — `BUZZ_REQUIRE_MEDIA_GET_AUTH` is not applied on this route. | `admin/mod.rs:226`; `media.rs:619-631` |
| BR-API-144 | Only `GET` is routed for admin paths; write verbs get 405. | `admin/mod.rs:29-38`; test `:406-429` |
| BR-API-145 | Admin reads are **deployment-wide**: `community_id` is an optional filter on `reports`, and `report_detail`/`feedback_detail`/`feedback_attachment` take no community argument at all. | `admin/mod.rs:100-110`, `:125-138`, `:177-189` |
| BR-API-146 | Every `DbError` collapses to 500 `internal_error` with a fixed message. | `admin/error.rs:79-83` |

#### M. NIP-05 and mesh demo

| ID | Rule | `file:line` |
|---|---|---|
| BR-API-147 | The NIP-05 lookup is host-scoped: the community is bound from `Host` and the handle domain must match that same tenant host, not `config.relay_url`. | `nip05.rs:36-58`, `:79-102` |
| BR-API-148 | Absent `name`, unmapped host, and a miss are **indistinguishable** — all return `{names:{},relays:{}}` with 200. | `nip05.rs:41-62` |
| BR-API-149 | The advertised relay URL keeps the deployment scheme from config but uses the tenant host. | `nip05.rs:104-111` |
| BR-API-150 | `Access-Control-Allow-Origin: *` is set unconditionally on the NIP-05 response. | `nip05.rs:65-69` |
| BR-API-151 | `canonicalize_nip05` requires `local@domain` with `domain == tenant host`, lowercasing both parts. | `nip05.rs:79-102`; consumer `handlers/side_effects.rs:1145` |
| BR-API-152 | `/_mesh/demo/echo` returns 404 unless **both** mesh is enabled and `mesh_demo_echo` is on — 404 not 403 so a non-demo deployment looks route-less. | `mesh_demo.rs:64-70` |
| BR-API-153 | The `Owned` branch acquires a fenced Redis lease and **deliberately does not renew it** — it expires with its TTL (30 s default). | `mesh_demo.rs:29-33`, `:108-115` |
| BR-API-154 | The `Forwarded` branch waits at most 10 s for the owner's echo, then 504. | `mesh_demo.rs:44`, `:132-152` |
| BR-API-155 | The demo takes `community_id` from the **request body**, bypassing the host-derived tenant boundary every other route enforces. | `mesh_demo.rs:50-51`, `:100` |
