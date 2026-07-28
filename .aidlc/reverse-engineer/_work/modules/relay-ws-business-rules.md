## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Business Rules

Authoritative rule set for the WebSocket path. Every rule cites its enforcement site. Rules marked **⚠** are behaviours that deviate from NIP-01, from an in-repo doc, or from the sibling HTTP path.

---

#### A. Connection admission and lifecycle

| ID | Rule | Site |
|---|---|---|
| BR-WS-01 | The community is bound from the HTTP `Host` **before** the WebSocket upgrade. An unmapped host or a lookup error is rejected with a generic `404 relay: no community is configured for this host` — never a default tenant, and the host is never echoed back. | `router.rs:286-300` |
| BR-WS-02 | A socket is refused with `503 relay restarting` if `shutting_down` is set, even though readiness already returns 503 (direct dials and in-flight upgrades still arrive). | `router.rs:302-312` |
| BR-WS-03 | The parser's `max_message_size`/`max_frame_size` are set from `config.max_frame_bytes` before the handler exists; the app-level size check in `recv_loop` is defence in depth. | `router.rs:334-342`; `connection.rs:421`, `:440` |
| BR-WS-04 | Every socket registers in the community lifecycle registry, then the community's `is_community_active` is revalidated **after** registration. Archive-before-check is observed by the check; archive-after-registration is observed by the cancellation token. Either ordering closes the socket. | `connection.rs:132-140`; `state.rs:135-163` |
| BR-WS-05 | A connection slot is a `conn_semaphore` permit taken with `try_acquire_owned` — no queueing. Exhaustion logs and returns with **no** frame sent to the client. The permit is held for the whole connection and dropped last. | `connection.rs:149-155`, `:287` |
| BR-WS-06 | The AUTH challenge is sent as the very first frame, before the connection is registered in `ConnectionManager` and before `buzz_ws_connections_active` is incremented — so an immediate client disconnect leaks neither a registry entry nor a gauge count. | `connection.rs:182-212` (ordering comments `:194-196`, `:198`) |
| BR-WS-07 | A socket that has not reached `Authenticated` within **5 s** is cancelled. The timer starts at task spawn, is independent of client activity, and is cancelled by the shared token on normal close. | `connection.rs:27`, `:228-252` |
| BR-WS-08 | Heartbeat: `Ping` every 30 s on the **control** channel. The counter is pre-incremented, so the 3rd unanswered tick cancels. Any `Pong` resets it to 0. A full control channel is itself terminal (writer stalled). | `connection.rs:383-403`; reset `:461-463` |
| BR-WS-09 | Cleanup order on any exit: `cancel()` → await send+heartbeat+auth-timeout tasks → `sub_registry.remove_connection` + one `release_topic` per removed subscription → `conn_manager.deregister` → presence clear → gauge decrement → drop permit. | `connection.rs:263-287` |
| BR-WS-10 | Presence is cleared in Redis **only** when this was the last connection for that pubkey **in that community**. A session held in another community does not keep presence alive, and a second session in the same community does. | `connection.rs:274-287`; `state.rs:261-282` |
| BR-WS-11 | Graceful shutdown sends WS close code **1012 "relay restarting"** to every live socket regardless of tenant, and the drain flag is sticky so a registration that races past the drain snapshot self-signals. | `state.rs:352-374`, `:227-233` |

---

#### B. AUTH handshake sequence (NIP-42)

| ID | Rule | Site |
|---|---|---|
| BR-WS-12 | The handshake is relay-initiated: `["AUTH", challenge]` is pushed unsolicited; the client replies `["AUTH", <kind:22242 event>]`. `generate_challenge()` produces the nonce. | `connection.rs:157`, `:182-186`; `buzz-auth/src/nip42.rs:38` |
| BR-WS-13 | AUTH is processed **inline and awaited** in the recv loop (not spawned), so no two AUTH frames on one socket can interleave. | `connection.rs:503-509` |
| BR-WS-14 | AUTH is accepted only from `Pending`. From `Authenticated` → `OK false "auth-required: already authenticated"`. From `Failed` → `OK false "auth-required: authentication already failed"`. **There is no way to recover from `Failed`** — the socket is authentication-dead. | `auth.rs:45-74` |
| BR-WS-15 | The NIP-OA `auth` tag is extracted from the signed event **before** verification and is trusted only because the Schnorr signature covers it. Exactly one `auth` tag is required: zero → `None`; two or more → `None` (fail closed per NIP-OA). | `auth.rs:28-42`, extraction `:88-90` |
| BR-WS-16 | NIP-42 verification is pure crypto (challenge + expected relay URL); no DB, no tokens, no JWT. The expected relay URL is derived per-tenant. | `auth.rs:80-90`; `api/bridge.rs::nip42_expected_relay_url` |
| BR-WS-17 | **Gate order after verification is fixed**: (1) ban seam, (2) pubkey allowlist, (3) relay membership, (4) NIP-OA owner backfill/materialisation. The ban gate is first by design (MOD-7/M20: a ban must block connection auth even for open channels). | `auth.rs:105-275` |
| BR-WS-18 | Ban check is a **cascade**: the authenticated pubkey is checked first; only if clear is the cryptographically-proven NIP-OA owner checked. Owner ban ⇒ agent blocked; agent ban is agent-only. | `auth.rs:132-156` |
| BR-WS-19 | Ban lookups **fail closed but distinguish the cause**: `banned` → `blocked: you are banned from this community`; DB error → `error: internal error checking restriction state`. Both set `Failed` and cancel. | `auth.rs:113-183` |
| BR-WS-20 | The ban denial frame is queued on `ctrl_tx` (not `send_tx`) and *then* the token is cancelled — the send loop's cancel branch drains ctrl before emitting `Close`, so the client learns why. This "queue on ctrl, then cancel" idiom is load-bearing and regression-tested. | `auth.rs:173-182`; drain `connection.rs:334-345`; test `connection.rs:856-882` |
| BR-WS-21 | The pubkey allowlist applies **only** when `pubkey_allowlist_enabled` **and** `auth_method == Nip42`. A DB error is treated as "not allowed" (fail closed). | `auth.rs:189-212` |
| BR-WS-22 | Relay membership is enforced through the shared helper with NIP-OA owner-delegation fallback; failure → `restricted: not a relay member` + `Failed`. | `auth.rs:216-237` |
| BR-WS-23 | On an **open** relay (`require_relay_membership == false`) the NIP-OA owner is still extracted, unconditionally, purely for the agent→owner mapping the observer path needs. No feature flag gates this because the attestation is self-proving. | `auth.rs:239-253` |
| BR-WS-24 | `agent_owner_pubkey` is set on the `AuthContext` **only after** `materialize_nip_oa_owner` confirms the first-write-wins relationship; a failed materialisation logs a warning and leaves the field `None` (auth still succeeds). | `auth.rs:255-275` |
| BR-WS-25 | On success the pubkey is written into `ConnEntry.authenticated_pubkey` — the copy every fan-out access decision reads. `ConnectionState` never holds it. | `auth.rs:279-281`; reads `event.rs:146`, `:184`, `:460` |

##### Auth requirement matrix

| Message | Auth required | Behaviour when unauthenticated |
|---|---|---|
| `AUTH` | n/a | the only productive pre-auth message |
| `EVENT` | **yes** | `OK false "auth-required: not authenticated"` (`event.rs:621-628`) |
| `REQ` | **yes** | `NOTICE` + `CLOSED "auth-required: not authenticated"` (`req.rs:76-85`) |
| `COUNT` | **yes** | `CLOSED "auth-required: not authenticated"`, no NOTICE (`count.rs:41-47`) |
| `CLOSE` | **no** | fully processed pre-auth: mutates `conn.subscriptions`, calls `sub_registry.remove_subscription`, may call `release_topic`, and returns `CLOSED ""` (`connection.rs:581-583`; `close.rs:12-30`) |

BR-WS-26 **⚠** `CLOSE` is the one message an unauthenticated socket can execute end-to-end. It is harmless in effect (an unauthenticated socket owns no subscriptions, so `remove_subscription` returns `None` and no `release_topic` fires — `subscription.rs:162-178`) but it is unmetered, unauthenticated, and always answered — an unbounded free-work channel. See the security aspect.

---

#### C. Per-message admission (rate limiting)

| ID | Rule | Site |
|---|---|---|
| BR-WS-27 | Admission runs **after** parse and **before** dispatch, for every frame. | `connection.rs:498-500` |
| BR-WS-28 | Only `EVENT`, `REQ`, `COUNT` are metered. `AUTH` and `CLOSE` return `true` immediately. | `connection.rs:599-602` |
| BR-WS-29 | **Unauthenticated frames bypass admission entirely** — the limiter is pubkey-keyed, so a non-`Authenticated` state short-circuits to `true`. | `connection.rs:604-609` (`_ => return true`) |
| BR-WS-30 | Every metered frame consumes one `LimitType::WsEvents` token. The budget is a **5 s fixed window** of `human_ws_events_per_sec × 5` (default 10 → **50 per 5 s**), deliberately allowing a bounded burst for desktop startup. | `connection.rs:612-623`; `admission.rs:10`, `:44-49`; default `buzz-auth/src/rate_limit.rs:116-118` |
| BR-WS-31 | `EVENT` additionally consumes one `LimitType::Messages` token in a **60 s** window: `agent_standard_messages_per_min` (**120**) if `agent_owner_pubkey.is_some()`, else `human_messages_per_min` (**60**). | `connection.rs:632-650`; defaults `buzz-auth/src/rate_limit.rs:110-121` |
| BR-WS-32 | The agent tier is selected by `agent_owner_pubkey.is_some()`. Since that field is only set when NIP-OA materialisation succeeded (BR-WS-24), an agent whose materialisation failed is silently metered as a **human**. | `connection.rs:607`, `:633-637` |
| BR-WS-33 | Admission **fails closed**: a Redis error becomes `AdmissionError::Unavailable` → the frame is rejected with `rate-limited: shared admission unavailable`. | `admission.rs:34-40`; `connection.rs:670-677` |
| BR-WS-34 | Rejections are subscription-scoped only for `REQ` (`CLOSED`); `EVENT` and `COUNT` get a `NOTICE`. | `connection.rs:587-592`, `:624-627`, `:647` |
| BR-WS-35 **⚠** | A rate-limited `EVENT` therefore receives **no `OK` frame at all** — only a `NOTICE`. NIP-01 clients that key delivery confirmation on `OK` will hang on that event id. Same for a `COUNT` (no `COUNT` response, only a `NOTICE`). | `connection.rs:647` (passes `None`), `:566-569` |
| BR-WS-36 | Rate limits are community-scoped: the Redis key is `buzz:{community}:ratelimit:{pubkey}:{suffix}`, so the same key in two communities holds two independent quotas. | `buzz-auth/src/rate_limit.rs:153-156` |
| BR-WS-37 | A second, independent bound is the `handler_semaphore` (`max_concurrent_handlers`, default **1024**), taken with `try_acquire_owned` per EVENT/REQ/COUNT. Exhaustion → `rate-limited: too many concurrent requests`. `CLOSE` and `AUTH` do not take a permit. | `connection.rs:513-521`, `:541-550`, `:563-571`; default `config.rs:454-457` |
| BR-WS-38 | Observer frames (kind 24200) carry their **own** in-process limiter: 100/s per `(community, agent pubkey)`, fixed 1 s window. Control frames (owner→agent) are exempt so bursty telemetry cannot starve them. | `event.rs:894-916`, `:1032-1044` |

**Delta vs `ARCHITECTURE.md`.** §9 item 2 (`ARCHITECTURE.md:823`) states "**No rate limiting implementation** … none are enforced". That is false: `enforce_ws_admission` runs on every EVENT/REQ/COUNT (`connection.rs:498-500` → `:594-653`) against `state.admission_rate_limiter`, a `RedisRateLimiter` (`state.rs:712`, impl `buzz-pubsub/src/rate_limiter.rs`), using `RateLimitConfig` values that are env-overridable and validated non-zero (`config.rs:284-317`). What *is* true is narrower: the `IpConnections` tier is unused (see BR-WS-39).

| ID | Rule | Site |
|---|---|---|
| BR-WS-39 **⚠** | `LimitType::IpConnections` / `RateLimiter::check_ip_connection` are implemented (`buzz-pubsub/src/rate_limiter.rs:112`) but **never called** anywhere in the relay. There is no per-IP connection cap on the WS accept path. | `grep check_ip_connection` → only trait (`buzz-auth/src/rate_limit.rs:188`, `:234`), impl (`buzz-pubsub/src/rate_limiter.rs:112`), test stub (`admission.rs:85`) |

---

#### D. EVENT write path

| ID | Rule | Site |
|---|---|---|
| BR-WS-40 | Order of gates before any branch: auth → pubkey match → kind-22242 reject → observer route → ephemeral route → persistent ingest. | `event.rs:611-705` |
| BR-WS-41 | `event.pubkey` must equal the authenticated pubkey, **except** kind 1059 (gift wrap), whose sealed-sender design requires a mismatched author. | `event.rs:636-645` |
| BR-WS-42 | Kind 22242 (`KIND_AUTH`) can never be submitted via `EVENT`. | `event.rs:647-655` |
| BR-WS-43 | Scope enforcement is **empty-scopes-permissive**: `if !scopes.is_empty() && !scopes.contains(&MessagesWrite)`. A NIP-42 pubkey-only session has `scopes == []` and therefore passes every WS scope gate. | `event.rs:658`, `:676`; `req.rs:54` |
| BR-WS-44 | Kind 24200 observer frames are routed **before** the ephemeral branch even though 24200 is inside 20000–29999, so they never take the generic ephemeral path. | `event.rs:657-669` |
| BR-WS-45 | Ephemeral kinds (20000–29999) bypass the ingest pipeline entirely: verify → optional presence side-effect → optional membership check → `mark_local_event` → Redis publish → local fan-out → `OK true`. Never stored. | `event.rs:739-874` |
| BR-WS-46 | Ephemeral verification runs on `spawn_blocking`; a join error is reported as `error: internal error`, never as a validation failure. | `event.rs:748-768` |
| BR-WS-47 | Presence (kind 20001) accepts both a bare status string and legacy `{"status":…}` JSON; a bare string longer than 128 bytes is truncated on a UTF-8 char boundary. `"offline"` clears Redis presence; anything else sets it. | `event.rs:772-800` |
| BR-WS-48 | Presence then **falls through** to the ordinary channel-less ephemeral publish path so other nodes see the delta. | `event.rs:802-806`, `:843-871` |
| BR-WS-49 | A channel-scoped ephemeral event requires channel membership *before* publish; failure returns the membership error verbatim as `OK false`. | `event.rs:808-820` |
| BR-WS-50 | `mark_local_event` is always called **before** `publish_event`; if the publish fails the mark is immediately invalidated, so a failed publish cannot suppress a later legitimate Redis delivery of the same id. | `event.rs:824-838`, `:852-865`, `:1046-1058`, `:394-406` |
| BR-WS-51 | Channel-less ephemeral events publish to `EventTopic::Global`. The in-code "nil UUID sentinel" narrative at `event.rs:845-851` is **stale**: `EventTopic` is a two-variant enum (`Channel(Uuid)` / `Global`) and `publish_event` is called with `EventTopic::Global` (`event.rs:857`), with no `Uuid::nil()` anywhere on the path. | `event.rs:845-857` vs `buzz-pubsub` `EventTopic` |
| BR-WS-52 | Persistent events delegate to `super::ingest::ingest_event` with `IngestAuth::Nip42 { pubkey, scopes, channel_ids, conn_id }`; the WS handler only maps the outcome to `OK`. | `event.rs:698-735` |
| BR-WS-53 | `IngestError::Internal` is **sanitised** to `error: internal server error` before it reaches the wire; `Rejected`/`AuthFailed` messages pass through. | `event.rs:726-733` |
| BR-WS-54 | `OK true` means *durably accepted*, not *delivered*: `dispatch_persistent_event` awaits only the bounded audit enqueue, then spawns Redis publish + fan-out + workflow triggering and returns `0`. | `event.rs:311-371` (doc `:313-322`) |
| BR-WS-55 | The audit enqueue stays on the awaited path deliberately, using `send().await` on a capacity-1000 channel, so a saturated audit DB back-pressures the event handler rather than growing memory. | `event.rs:540-578` (rationale `:551-557`) |
| BR-WS-56 | The audit actor is the **caller-resolved actor hex**, not `event.pubkey` — for relay-signed events the claimed author is the relay key and deriving from the event would erase the human. | `event.rs:561-567`; regression test `event.rs:1738-1822` |
| BR-WS-57 | Workflow triggering is skipped for workflow-execution kinds, command kinds, relay-signed events tagged `buzz:workflow`, and kind 1059. | `event.rs:503-534` |

##### Observer frames (kind 24200)

| ID | Rule | Site |
|---|---|---|
| BR-WS-58 | Content must look NIP-44 encrypted, else `invalid: observer content must be NIP-44 encrypted`. | `event.rs:1072-1074` |
| BR-WS-59 | Exactly one each of `p`, `agent`, `frame` tags; zero or duplicates → `invalid`. | `event.rs:1117-1130` |
| BR-WS-60 | Direction is derived structurally: `pubkey == agent && p != agent` ⇒ Telemetry (owner = `p`); `p == agent && pubkey != agent` ⇒ Control (owner = `pubkey`); anything else rejected. | `event.rs:1082-1097` |
| BR-WS-61 | The `frame` tag value must match the direction's expected constant; a mismatch is **silently accepted** (`OK true`) and dropped, deliberately not signalled to the publisher. | `event.rs:1099-1102`, `:963-967` |
| BR-WS-62 | Timestamp must be within ±300 s of server time. | `event.rs:949-958` |
| BR-WS-63 | The publisher must be authorised for the `(agent, owner)` pair. Fast path: the session's verified `agent_owner_pubkey` equals the frame's owner (no DB read). Otherwise a 5-min `observer_owner_cache` lookup, then `db.is_agent_owner`. A DB error → `error: internal server error` (deny). | `event.rs:986-1030` |
| BR-WS-64 | Observer frames are routed as **global** ephemeral events (`EventTopic::Global`, `StoredEvent` with `channel_id = None`), never stored. Subscription-side gating is the REQ `#p` p-gate. | `event.rs:1046-1066` (doc `:915-919`) |

---

#### E. REQ read path

| ID | Rule | Site |
|---|---|---|
| BR-WS-65 | REQ is refused without `MessagesRead` (when scopes are non-empty) with a `NOTICE` **and** a `CLOSED`. | `req.rs:54-61` |
| BR-WS-66 | The 1024-subscription cap is checked against `conn.subscriptions` and only for a **new** `sub_id`; replacing an existing id is always allowed. | `req.rs:65-72` |
| BR-WS-67 | The accessible-channel set is resolved once per REQ from a 10 s cache, then narrowed by any scoped-token `channel_ids`. Every downstream consumer (search scope, registration, historical delivery, per-filter access) reads that one vector. | `req.rs:89-104`, `:105-107` |
| BR-WS-68 | Optimisation: a REQ whose every filter targets **only** kind 13534 (NIP-43 membership list) skips accessible-channel resolution entirely, because that kind is globally stored. | `req.rs:89-93`, `:829-840` |
| BR-WS-69 | Channel access for the subscription-level `#h` is confirmed **up front**, before the search branch, so a `#h` search against a just-joined channel is not scoped against a stale cache. | `req.rs:130-174` (rationale `:118-129`) |
| BR-WS-70 | Request-local access repair truth table (`resolve_request_local_access`): token denies → deny, no DB, no repair; cached hit → allow, no DB; cache-miss + DB member → allow **and push into the vector**; cache-miss + DB non-member → deny, vector untouched. The scoped-token bound is an absolute ceiling — a DB positive can never push a channel past a narrower token. | `req.rs:448-465` (doc `:423-447`); tests `:1299-1360` |
| BR-WS-71 | Denial for an inaccessible `#h` is `CLOSED "restricted: not a channel member"`. | `req.rs:167-171` |

##### The p-gate (the "kinds required" rule)

| ID | Rule | Site |
|---|---|---|
| BR-WS-72 | **The p-gate.** For a filter that *can* match a p-gated kind, the filter must carry a non-empty `#p` whose **every** value equals the authenticated pubkey. "Can match" is `filter.kinds.is_none_or(|ks| ks.iter().any(|k| P_GATED_KINDS.contains(k)))` — so **a filter that omits `kinds` always can match** and is therefore gated. This is the exact mechanism behind AGENTS.md's "relay queries must specify `kinds`". | `req.rs:1042-1074`; kind set `buzz-core/src/kind.rs:146-156` (24200, member-added, member-removed, 1059, 30622, 44200) |
| BR-WS-73 | **`ids` exemption**, with two carve-outs. A filter carrying a non-empty `ids` is exempt ("knowing the id implies authorisation") — *unless* the filter explicitly names `KIND_DM_VISIBILITY` (30622) or `KIND_AGENT_TURN_METRIC` (44200), which are relay-signed / metadata-leaking and keep the `#p` requirement. A **kindless** `ids` filter keeps the exemption. | `req.rs:1052-1068`; tests `:1454-1487`, `:1493-1546` |
| BR-WS-74 **⚠** | **On the WS REQ path the p-gate only runs when the subscription is global** (`channel_id.is_none()`). A REQ carrying a resolvable single `#h` skips all three sensitive-kind gates. | `req.rs:182-205` |
| BR-WS-75 | The p-gate's transport-specific outcome differs: WS → `CLOSED "restricted: p-gated events require #p matching your pubkey"`; HTTP `/query` and `/count` → **HTTP 403** `"restricted: p-gated kinds require #p tag matching your pubkey"`; WS COUNT → `CLOSED` with the *HTTP* wording. Three surfaces, two wordings. | `req.rs:184-189`; `api/bridge.rs:981-985`, `:1404-1408`; `count.rs:55-60` |
| BR-WS-76 | The engram gate (kind 30174): a filter that can match it needs `authors` all-self **or** `#p` all-self; non-empty `ids` is exempt; a kindless filter is gated. | `req.rs:1099-1135` |
| BR-WS-77 | The author-only gate (kinds in `AUTHOR_ONLY_KINDS` = event reminder + push lease, `buzz-core/src/kind.rs:120`): a filter targeting **only** author-only kinds must set `authors` all-self. Mixed-kind filters pass and rely on the per-event filter. | `req.rs:1204-1223`, per-event `:1186-1190` |
| BR-WS-78 | All three gates run **before** the NIP-50 search branch, deliberately, because search hits are fetched by id and skip the per-filter post-check. | `req.rs:175-205` (rationale `:175-181`) |

##### Search REQ

| ID | Rule | Site |
|---|---|---|
| BR-WS-79 | Mixing `search` and non-`search` filters in one REQ → `CLOSED "error: mixed search and non-search filters not supported"`. | `req.rs:211-219` |
| BR-WS-80 | A search REQ is **one-shot**: no `conn.subscriptions` entry, no registry entry, no `retain_topic`. Results then EOSE. | `req.rs:220-234` |
| BR-WS-81 **⚠** | Because the search branch returns before `subs.insert` (`:238`) and `register_scoped` (`:241`), **a search REQ reusing a live `sub_id` does not replace the existing subscription.** NIP-01 requires a REQ with an existing sub_id to overwrite it; here the old subscription keeps live-fanning-out under an id the client believes was replaced by a one-shot query. | `req.rs:212-234` vs `:236-247` |
| BR-WS-82 | A search REQ also never counts against the 1024 cap (the cap reads `conn.subscriptions`, which search never writes). | `req.rs:66` vs `:236-239` |
| BR-WS-83 | Search scope resolution: no accessible channels **and** no global access → immediate EOSE. Otherwise `ChannelsOrChannelLess` / `Channels` / `ChannelLessOnly` per the 4-case table. `include_global` is `token_channel_ids.is_none()` — i.e. a scoped token never broadens to channel-less rows. | `req.rs:483-501`, `:224`, `:520-525`; test `:1650-1666` |
| BR-WS-84 | A filter's `#h` values are intersected with accessible channels; if all are invalid or inaccessible the filter is **skipped** (matches nothing) rather than broadened. | `req.rs:568-583` |
| BR-WS-85 | Search paginates up to 10 pages of 100 hits per filter, always fetching full pages regardless of the requested limit, because post-filtering discards hits. `limit == 0` skips the filter (NIP-01). A short page terminates the loop. | `req.rs:421`, `:536-545`, `:586-600`, `:604-606`, `:725-727` |
| BR-WS-86 | Search results are post-filtered against **that filter only** (not the OR of all filters), then channel-access checked, then `reader_authorized_for_event`, then author-only, then deduped. | `req.rs:685-712` |

##### Subscription registration and historical delivery

| ID | Rule | Site |
|---|---|---|
| BR-WS-87 | Registration replaces same-`sub_id` (NIP-01). `register_scoped` returns the replaced entry's scope so exactly one `release_topic` is issued for it, then `retain_topic` for the new scope. | `req.rs:236-257`; `subscription.rs:69-160` |
| BR-WS-88 | Subscription scope is `Some(id)` only if **every** filter carries an `h` tag and all resolvable ids agree; a kindless-`h` filter or two distinct ids ⇒ global. | `req.rs:1013-1039` |
| BR-WS-89 | Historical delivery is **one DB query per filter**, not one merged query, so per-filter `limit` and time windows are respected (NIP-01 OR semantics). | `req.rs:261-330` (rationale `:261-265`) |
| BR-WS-90 | Queries run with bounded concurrency 4 via `buffered` (not `buffer_unordered`), so post-processing observes filters in original order — preserving dedupe order, conformance-trace order, and first-error-wins. Pinned by a test. | `req.rs:299-314`; test `:1271-1297` |
| BR-WS-91 | Per-filter channel scope for query construction prefers the filter's own single `#h` over the subscription scope, so cross-channel rows do not consume the LIMIT. | `req.rs:267-283` |
| BR-WS-92 | A logically global query gets `channel_ids = accessible_channels` pushed into SQL so `LIMIT` counts only visible rows; an explicit single-channel filter keeps its narrower `channel_id`. | `req.rs:993-1002`; tests `:1236-1260` |
| BR-WS-93 | Per-row acceptance order in the delivery loop: `filters_match` (this filter only) → channel accessible → `reader_authorized_for_event` → not-author-only → dedupe. | `req.rs:332-406` |
| BR-WS-94 | **Dedupe happens after acceptance**, so an event rejected by filter A stays eligible for filter B. | `req.rs:389-392` |
| BR-WS-95 | Every 100 delivered events the task yields, so one large REQ cannot starve the runtime. | `req.rs:401-404` |
| BR-WS-96 **⚠** | A failed historical query sends `EOSE` and returns — **not** `CLOSED`. The subscription stays registered and live. The client sees a normal (empty) end-of-stored-events and cannot distinguish "no matches" from "DB error". | `req.rs:318-326` |
| BR-WS-97 **⚠** | If `conn.send` fails mid-delivery (buffer full), the handler returns **without EOSE**, leaving the client waiting on a subscription that is registered and live. | `req.rs:398-400`; search `:720-722` |
| BR-WS-98 | `limit` is clamped to `MAX_HISTORICAL_LIMIT` (2000) per filter; `kinds: []` produces `Some(vec![])`, which the DB layer treats as "no matching kinds". | `req.rs:883-886`, `:861-871` |
| BR-WS-99 | SQL pushdown set: kinds, single-author (`pubkey`) or multi-author (`authors`), `ids`, `since`, `until`, `#h`→`channel_id`, single `#p`, `#e` (any count), `#d` **only** when every kind is NIP-33 (30000–39999) because `d_tag` is NULL otherwise. Everything else post-filters. | `req.rs:860-991`; pushability predicate `:777-827`; test `:1594-1648` |

---

#### F. CLOSE

| ID | Rule | Site |
|---|---|---|
| BR-WS-100 | CLOSE removes from `conn.subscriptions` first, then deregisters from the fan-out index, then releases the Redis topic, and only then sends `CLOSED ""` — so no event is routed to the sub after the ack. | `close.rs:12-30` (ordering comment `:18-19`) |
| BR-WS-101 | An unknown `sub_id` still receives `CLOSED ""`; `remove_subscription` returning `None` simply skips `release_topic`. Idempotent, unauthenticated, and always answered. | `close.rs:20-27`; `subscription.rs:162-178` |
| BR-WS-102 | An empty-string `sub_id` is accepted for CLOSE (unlike REQ/COUNT, which reject it at parse). | `protocol.rs:152-162` vs `:89-93` |

---

#### G. Fan-out and delivery

| ID | Rule | Site |
|---|---|---|
| BR-WS-103 | **Registration is never sufficient for delivery.** Every fan-out path re-validates access on the sending pod, so a subscription surviving a membership change or an open→private flip cannot leak. | `event.rs:196-217` (doc), enforced `:224`, `:284`, `:409` |
| BR-WS-104 | `filter_fanout_by_access` applies four fences, in this order: (1) receiver-side tenant label — `community_for_conn(conn) == Some(event_community)`; (2) author-only kinds → author's connection only; (3) channel-less short-circuit (pass through); (4) private-channel membership. | `event.rs:115-195` |
| BR-WS-105 | Fence 1 is a hard cross-tenant fence at the send chokepoint, independent of the (already community-scoped) subscription indexes. | `event.rs:126-133`; regression test `:2436-2457` |
| BR-WS-106 | The author-only fence runs **before** the channel fence because author-only kinds are stored globally (`channel_id = None`) and would otherwise short-circuit past it. An unauthenticated recipient fails closed. | `event.rs:135-157`; test `:2211-2255` |
| BR-WS-107 | Channel visibility: only `"private"` triggers per-recipient membership filtering. A visibility lookup **error returns an empty recipient list** (fail closed). Unknown/unauthenticated recipients on a private channel are dropped; a membership lookup error drops that recipient only. | `event.rs:159-195` |
| BR-WS-108 | A threaded visibility value is honoured **only** when its `(community_id, channel_id)` exactly match this fan-out's; a mismatch or absence falls back to the fresh fail-closed lookup, never to "assume open". Membership checks always stay fresh. | `event.rs:161-180`; fence tests `:2259-2325` |
| BR-WS-109 | The visibility cache caches **only** `"private"`, never a non-private value — so the worst stale entry is over-restrictive, never a leak. | `state.rs:1124-1150` |
| BR-WS-110 | `dispatch_persistent_event_inner` layers a fifth fence for kinds 30622 / 44200: the recipient's authenticated pubkey must equal the event's `#p` value. | `event.rs:436-472` |
| BR-WS-111 | Frames are serialised once per event and cached per distinct `sub_id` within one fan-out cycle; the cache must not be reused across cycles (pinned by test). | `event.rs:63-98`; test `:1168-1188` |
| BR-WS-112 | Subscription scoping is **symmetric**: global subscriptions never receive channel-scoped events, and channel-scoped subscriptions never receive global events. Achieved structurally by index partitioning, not by a skip check. | `subscription.rs:265-330` (invariant comment `:320-327`); tests `:1005-1041`, `:1191-1227` |
| BR-WS-113 | Global fan-out consults the `(kind, #p)` index for **each** distinct `p` tag value on the event, then the generic kind index, then the global wildcard index; duplicates are suppressed by a `seen` set. | `subscription.rs:289-318`, `:369-387` |
| BR-WS-114 | A `kinds: []` subscription is never indexed and can never receive an event. | `subscription.rs:95-100`, `:415-418`; test `:952-1002` |

##### Local-echo dedup

| ID | Rule | Site |
|---|---|---|
| BR-WS-115 | The dedup key is `(community_id, event_id)`. A same-id event arriving for a **different** community is a distinct delivery and must not be suppressed. | `event.rs:272-281`; `state.rs:530-540`; test `:1560-1600` |
| BR-WS-116 | The check is **consume-on-read**: a hit invalidates the entry and returns, so only the first Redis echo per `(community, id)` is suppressed. | `event.rs:277-281` |
| BR-WS-117 | Entries expire after 60 s with a 10 000-entry cap, so the cache is bounded regardless of subscriber health. | `state.rs:734-739` |
| BR-WS-118 | Mark-before-publish and invalidate-on-publish-failure (BR-WS-50) is what keeps dedup from swallowing a real delivery. | `event.rs:394-406` |
| BR-WS-119 | The Redis consumer loop applies the same `filter_fanout_by_access` gate as the local path, then writes frames. Broadcast lag increments `buzz_multinode_fanout_lag_total`; a closed channel ends the loop. | `main.rs:818-845`; gate `event.rs:284` |

---

#### H. COUNT (NIP-45)

| ID | Rule | Site |
|---|---|---|
| BR-WS-120 | COUNT requires `Authenticated`; unauthenticated → `CLOSED "auth-required: not authenticated"`. | `count.rs:38-50` |
| BR-WS-121 **⚠** | COUNT does **not** check `MessagesRead` scope, unlike REQ (`req.rs:54`). A scoped session without read scope can still COUNT. | `count.rs:38-50` (no scope branch) |
| BR-WS-122 | COUNT applies the p-gate, engram gate, and author-only gate **unconditionally** — not only for global filters. It is therefore *stricter* than REQ (BR-WS-74). | `count.rs:52-75` |
| BR-WS-123 | Accessible channels are resolved and narrowed by the token exactly as in REQ; without the `retain` narrowing a scoped token could count outside its scope via the no-channel-filter pushdown. | `count.rs:77-96` (rationale `:88-93`) |
| BR-WS-124 | Per filter, the fast SQL `count_events` path is taken only when **all** hold: `filter_fully_pushable`, and (author-only cannot match **or** `authors` is all-self), and result-gated cannot match **or** `#p` is all-self. Otherwise a bounded candidate query + per-row post-filter. | `count.rs:100-113`, `:160-163`, `:230-233` |
| BR-WS-125 | The fallback fetches 5001 rows; > 5000 candidates → `CLOSED "restricted: count filter requires narrower constraints"` and `buzz_count_fallback_rejections_total`. COUNT must be exact, so a truncated count is never returned. | `req.rs:753-765`; `count.rs:173-184`, `:243-254` |
| BR-WS-126 | Fallback rows are counted only if they pass `filters_match`, are not another author's author-only event, and pass `reader_authorized_for_event`. | `count.rs:186-201`, `:256-271` |
| BR-WS-127 | A filter targeting an inaccessible channel is `continue`d (contributes 0), not an error — so a partially-inaccessible COUNT silently returns a smaller number. | `count.rs:145-152` |
| BR-WS-128 | A filter with no `#h` gets `channel_ids = accessible_channels` and `limit = None` on the fast path. | `count.rs:216-229`, `:234` |
| BR-WS-129 **⚠** | COUNT DB errors are forwarded verbatim: `CLOSED "error: {e}"`. Unlike the EVENT path (BR-WS-53), there is no sanitisation — raw `sqlx`/Postgres text can reach an authenticated client. | `count.rs:167`, `:203`, `:238`, `:273` |

---

#### I. Backpressure and slow clients

| ID | Rule | Site |
|---|---|---|
| BR-WS-130 | All data sends are `try_send`, never awaited. A full buffer increments a shared strike counter; a successful send resets it to 0. At `grace_limit` (default **15**) strikes the connection is cancelled. | `connection.rs:88-113`; `state.rs:436-475` |
| BR-WS-131 | The strike counter is shared (`Arc<AtomicU8>`) between direct `ConnectionState::send` and fan-out `ConnectionManager::send_to*`, so mixed-path pressure accumulates on one counter. | `connection.rs:77`; `state.rs:54`, `:212`; test `state.rs:1343-1405` |
| BR-WS-132 | The send loop drains **all** pending control frames before touching data on every iteration, and the `select!` is `biased` (cancel > control > data). | `connection.rs:308-372` (`biased;` `:326`) |
| BR-WS-133 | On cancel the loop drains queued control frames **again** before emitting `Close`, which is what makes the "queue frame on ctrl, then cancel" idiom safe. | `connection.rs:328-345` |
| BR-WS-134 | Data frames are batched up to 64 per `flush()` and the batch size is recorded as a histogram. | `connection.rs:347-369` |
| BR-WS-135 | A full **control** channel is always terminal (client Ping, heartbeat Ping): it means the socket writer is completely stalled. | `connection.rs:396-400`, `:464-472` |
| BR-WS-136 | Live ban enforcement is cluster-wide: pod-local `disconnect_pubkey` (tenant-fenced) plus a fire-and-forget conn-control publish; the DB ban row is the durable backstop. | `state.rs:290-325`, `:508-556` |
