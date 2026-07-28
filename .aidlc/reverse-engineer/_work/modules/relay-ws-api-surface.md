## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: API Surface

The WS surface is NIP-01 + NIP-42 + NIP-45, with NIP-29 scoping expressed through the `#h` tag rather than distinct message types. Entry point: `router.rs:301-315` (`WebSocketUpgrade::from_request` → `handle_connection`), reached only through `GET /` with a WebSocket upgrade after the host resolves to a community (`router.rs:286-300`).

---

#### 1. Inbound wire surface — exactly 5 `ClientMessage` variants

Parser: `protocol.rs:57-170`. Dispatcher: `connection.rs:489-585`.

| Wire form | `ClientMessage` | Parse | Dispatch | Executed | Metered by admission |
|---|---|---|---|---|---|
| `["EVENT", <event>]` | `Event(Event)` | `protocol.rs:66-76` | `connection.rs:510-537` | spawned task under `handler_semaphore` | **yes** — `WsEvents` + `Messages` |
| `["REQ", <sub_id>, <filter>…]` | `Req { sub_id, filters }` | `protocol.rs:77-110` | `connection.rs:538-559` | spawned task under `handler_semaphore` | **yes** — `WsEvents` |
| `["COUNT", <sub_id>, <filter>…]` | `Count { sub_id, filters }` | `protocol.rs:111-151` | `connection.rs:560-580` | spawned task under `handler_semaphore` | **yes** — `WsEvents` |
| `["CLOSE", <sub_id>]` | `Close(String)` | `protocol.rs:152-162` | `connection.rs:581-583` | **inline, awaited** in the recv loop | no (`connection.rs:599-602`) |
| `["AUTH", <event>]` | `Auth(Event)` | `protocol.rs:163-172` | `connection.rs:503-509` | **inline, awaited** in the recv loop | no (`connection.rs:599-602`) |

Any other first element → `RelayError::InvalidMessage("unknown message type: …")` (`protocol.rs:173-175`) → `["NOTICE","invalid message: …"]` (`connection.rs:493`).

Parse-time rejections (all become one `NOTICE`):

| Condition | Message fragment | Site |
|---|---|---|
| not valid JSON | `JSON parse error: …` | `protocol.rs:59-60` |
| not a JSON array | `expected JSON array` | `:62-64` |
| empty array | `empty array` | `:66-68` |
| element 0 not a string | `first element must be a string` | `:70-72` |
| `EVENT`/`AUTH` with `len < 2` | `EVENT requires event object` / `AUTH requires event object` | `:67-71`, `:164-168` |
| event body not deserializable | `invalid event: …` / `invalid auth event: …` | `:73-74`, `:169-170` |
| `REQ`/`COUNT`/`CLOSE` with `len < 2` | `REQ requires sub_id` etc. | `:78-82`, `:112-116`, `:153-157` |
| sub_id not a string | `REQ sub_id must be a string` etc. | `:84-88`, `:118-122`, `:159-163` |
| empty sub_id (REQ/COUNT only — **CLOSE allows empty**) | `must not be empty` | `:89-93`, `:123-127` |
| sub_id > 256 B | `… sub_id exceeds maximum length of 256 bytes` | `:94-98`, `:128-133` |
| > 10 filters | `… contains N filters, maximum is 10` | `:100-105`, `:135-140` |
| a filter not deserializable | `invalid filter: …` | `:106-112`, `:141-147` |

Frame-level handling in `recv_loop` (`connection.rs:407-487`):

| Frame | Behaviour |
|---|---|
| `Text` > `max_frame_bytes` | `["NOTICE","error: frame too large (N bytes, limit M)"]` then **disconnect** (`:421-434`) — note this is a hand-built JSON string, not `RelayMessage::notice` |
| `Binary` > limit | `["NOTICE","error: binary frame too large …"]` then disconnect (`:440-452`) |
| `Binary` valid UTF-8 | treated as text (`:457-459`) — documented as a non-NIP-01 extension (`:454-456`) |
| `Binary` invalid UTF-8 | **silently dropped**, no NOTICE (`:457`, no `else`) |
| `Ping` | Pong via `ctrl_tx.try_send`; a full ctrl channel is terminal (`:464-472`) |
| `Pong` | resets `missed_pongs` (`:461-463`) |
| `Close` / stream end / stream error | break (`:474-481`) |

---

#### 2. Outbound wire surface — 7 `RelayMessage` formatters

`protocol.rs:178-217`. All are `String`-returning helpers on a unit struct; there is no outbound enum.

| Formatter | Wire form | Site |
|---|---|---|
| `auth_challenge` | `["AUTH", <challenge>]` | `protocol.rs:181` |
| `event` | `["EVENT", <sub_id>, <event>]` | `:185-189` |
| `notice` | `["NOTICE", <msg>]` | `:192` |
| `eose` | `["EOSE", <sub_id>]` | `:196` |
| `ok` | `["OK", <event_id>, <bool>, <msg>]` | `:200` |
| `closed` | `["CLOSED", <sub_id>, <msg>]` | `:204` |
| `count` | `["COUNT", <sub_id>, {"count": N}]` | `:208` |

Fan-out bypasses these helpers for performance: `event_frame_for_sub` hand-formats `["EVENT","<sub>",<json>]` (`event.rs:55-57`), byte-for-byte equality with the legacy form pinned by `event.rs:1155-1166`.

---

#### 3. Complete outbound message table with exact conditions and reason strings

##### 3.1 `AUTH` challenge

| Condition | Site |
|---|---|
| Immediately after a slot is granted, before any frame is read; on send failure the connection is abandoned without registering | `connection.rs:182-192` |

##### 3.2 `OK` (EVENT / AUTH acknowledgements)

| `accepted` | Reason string | Condition | Site |
|---|---|---|---|
| `true` | `""` | AUTH verified and all gates passed | `auth.rs:282` |
| `false` | `auth-required: already authenticated` | AUTH while `Authenticated` | `auth.rs:49-57` |
| `false` | `auth-required: authentication already failed` | AUTH while `Failed` | `auth.rs:58-66` |
| `false` | `auth-required: verification failed` | allowlist denial (`pubkey_allowlist_enabled` + `Nip42`) | `auth.rs:207-211` |
| `false` | `auth-required: verification failed` | `verify_auth_event` error | `auth.rs:284-292` |
| `false` | `restricted: not a relay member` | `enforce_relay_membership` error | `auth.rs:231-235` |
| `false` | `blocked: you are banned from this community` | self or NIP-OA owner banned — **queued on `ctrl_tx`, then `cancel()`** | `auth.rs:167-183` |
| `false` | `error: internal error checking restriction state` | ban-state DB error (fail closed) | `auth.rs:126-129`, delivered `:174-177` |
| `false` | `auth-required: not authenticated` | EVENT before auth | `event.rs:621-628` |
| `false` | `invalid: event pubkey does not match authenticated identity` | `event.pubkey != auth.pubkey` and kind ≠ 1059 | `event.rs:637-645` |
| `false` | `invalid: AUTH events cannot be submitted via EVENT` | kind 22242 over EVENT | `event.rs:647-655` |
| `false` | `restricted: insufficient scope for agent observer frames` | kind 24200 without `MessagesWrite` | `event.rs:658-666` |
| `false` | `restricted: insufficient scope for ephemeral events` | kind 20000–29999 without `MessagesWrite` | `event.rs:676-684` |
| `false` | `invalid: <verify error>` | signature/id verification failure (ephemeral) | `event.rs:753-759`; (observer) `:929-935` |
| `false` | `error: internal error` | `spawn_blocking` join failure | `event.rs:760-767`, `:936-943` |
| `false` | `<check_channel_membership msg>` | ephemeral event `#h` channel not permitted | `event.rs:809-820` |
| `true` | `""` | ephemeral accepted (after publish + local fan-out) | `event.rs:873` |
| `false` | `invalid: observer frame timestamp outside ±5 minute freshness window` | \|ts − now\| > 300 s | `event.rs:952-958` |
| `false` | `invalid: observer content must be NIP-44 encrypted` | plaintext content | `event.rs:1072-1074` → `:970` |
| `false` | `invalid: observer frame missing/multiple <tag> tags` | `p` / `agent` / `frame` tag arity | `event.rs:1117-1130` → `:970` |
| `false` | `invalid: observer <tag> tag must be a hex pubkey` | bad hex | `event.rs:1111-1115` → `:970` |
| `false` | `invalid: observer frame must be agent-to-owner telemetry or owner-to-agent control` | direction cannot be derived | `event.rs:1093-1097` |
| `true` | `""` | **unknown `frame` value → silently accepted and dropped** | `event.rs:1099-1102` → `:963-967` |
| `false` | `restricted: observer frame is not authorized for this agent owner` | not the agent's owner | `event.rs:1021-1029` |
| `false` | `error: internal server error` | `is_agent_owner` DB error | `event.rs:1006-1015` |
| `false` | `rate-limited: observer frame rate exceeded (100/sec per agent)` | telemetry burst | `event.rs:1036-1043` |
| `true` | `""` | observer frame accepted | `event.rs:1068` |
| pass-through | `result.message` | persistent event → `ingest_event` outcome | `event.rs:719-723` |
| `false` | `<IngestError::Rejected msg>` / `<AuthFailed msg>` / `error: internal server error` | ingest failure, internal errors sanitised | `event.rs:725-734` |

##### 3.3 `CLOSED`

| Reason string | Condition | Site |
|---|---|---|
| `""` | successful CLOSE ack — sent **even if the sub_id was never registered** | `close.rs:27` |
| `rate-limited: too many concurrent requests` | `handler_semaphore` exhausted on REQ | `connection.rs:544-547` |
| `rate-limited: quota exceeded; retry in Ns` | admission quota on REQ | `connection.rs:662-668` |
| `rate-limited: shared admission unavailable` | Redis limiter error on REQ | `connection.rs:670-676` |
| `auth-required: not authenticated` | REQ before auth (preceded by a NOTICE) | `req.rs:80-83` |
| `auth-required: not authenticated` | COUNT before auth (**no** preceding NOTICE) | `count.rs:43-47` |
| `restricted: insufficient scope` | REQ without `MessagesRead` (preceded by a NOTICE) | `req.rs:56-59` |
| `error: too many subscriptions` | ≥1024 live subs on this connection | `req.rs:67-70` |
| `error: database error` | accessible-channel lookup failure | `req.rs:101-104`; membership confirm `:158-161`; COUNT `:85`, `:130-132` |
| `restricted: not a channel member` | `#h` channel not accessible | `req.rs:167-171` |
| `restricted: p-gated events require #p matching your pubkey` | REQ p-gate — **global subs only** | `req.rs:184-189` |
| `restricted: p-gated kinds require #p tag matching your pubkey` | COUNT p-gate — **all** COUNTs | `count.rs:55-60` |
| `restricted: agent-engram reads require authors=[self] or #p=[self]` | kind 30174 gate | `req.rs:191-196`, `count.rs:62-67` |
| `restricted: author-only kinds require authors=[self]` | kinds 30300 / push-lease gate | `req.rs:198-203`, `count.rs:69-74` |
| `error: mixed search and non-search filters not supported` | some filters have `search`, some don't | `req.rs:213-218` |
| `restricted: count filter requires narrower constraints` | COUNT fallback candidate set > 5000 | `count.rs:177-183`, `:247-253` |
| `error: <db error>` | COUNT DB error — **raw error text is forwarded to the client** | `count.rs:167`, `:203`, `:238`, `:273` |

##### 3.4 `NOTICE`

| Message | Condition | Site |
|---|---|---|
| `invalid message: <parse error>` | any `ClientMessage::parse` failure | `connection.rs:493` |
| `error: frame too large (N bytes, limit M)` | oversized text frame (then close) | `connection.rs:428-432` |
| `error: binary frame too large (…)` | oversized binary frame (then close) | `connection.rs:447-451` |
| `rate-limited: too many concurrent requests` | `handler_semaphore` exhausted on EVENT or COUNT | `connection.rs:516-518`, `:566-568` |
| `rate-limited: quota exceeded; retry in Ns` | admission quota on EVENT or COUNT | `connection.rs:662-668` with `sub_id = None` (`:624-627`, `:647`) |
| `rate-limited: shared admission unavailable` | limiter error on EVENT or COUNT | `connection.rs:670-676`, same `None` |
| `auth-required: authenticate before subscribing` | REQ before auth, paired with a CLOSED | `req.rs:77-79` |
| `restricted: insufficient scope` | REQ without `MessagesRead`, paired with a CLOSED | `req.rs:55` |

##### 3.5 `EVENT`

| Path | Site |
|---|---|
| historical delivery (non-search REQ) | `req.rs:396-400` via `RelayMessage::event` |
| historical delivery (NIP-50 search) | `req.rs:720-722` |
| live fan-out — relay-local | `event.rs:239-252` (`fan_out_event_to_local_subscribers`) |
| live fan-out — persistent ingest (+ DM-visibility owner fence) | `event.rs:453-472` |
| live fan-out — Redis cross-node | `event.rs:293-306` |

##### 3.6 `EOSE`

| Condition | Site |
|---|---|
| after historical delivery completes | `req.rs:408` |
| **on historical query error** (in place of CLOSED) | `req.rs:320-325` |
| search: no accessible channels and no global access | `req.rs:520-525` |
| search: after all filters/pages | `req.rs:730` |

Never sent when `conn.send` returns `false` mid-delivery: `req.rs:398-400` and `:720-722` both `return` without EOSE.

##### 3.7 `COUNT` response

| Condition | Site |
|---|---|
| all filters processed without an early return | `count.rs:280` |

##### 3.8 Control frames (not NIP-01 payloads)

| Frame | Condition | Site |
|---|---|---|
| `Ping` | every 30 s | `connection.rs:383`, `:394` |
| `Pong` | client Ping | `connection.rs:467` |
| `Close(None)` | any `cancel()` — sent after draining queued ctrl frames | `connection.rs:334-345` |
| `Close(1012 "relay restarting")` | graceful shutdown / late registration | `state.rs:368-373`, queued `:359`, `:236` |

---

#### 4. NIP-29 expression on the wire

There is **no** NIP-29-specific message type. Channel scope is carried entirely by the `#h` generic tag:

- Subscription-level scope: `extract_channel_id_from_filters` (`req.rs:1013-1039`) — returns `Some(id)` only if **every** filter carries an `h` tag and they all agree; any kindless-h filter or divergent id → `None` (global).
- Per-filter scope for query construction: `req.rs:265-283` prefers a filter's own single `#h`, falling back to the subscription scope.
- COUNT scope: `extract_channel_from_filter` (`count.rs:16-26`) — single-valued `#h` only.

Consequences visible on the wire: a REQ mixing two different `#h` values is silently treated as a **global** subscription (`req.rs:1023-1027`) and therefore receives no channel-scoped events at all (`subscription.rs:320-327`). No NOTICE is emitted for this.

---

#### 5. Rust API exported by this group

| Item | Visibility | Consumers outside the group |
|---|---|---|
| `connection::handle_connection` | `pub` (`connection.rs:118`) | `router.rs:313` |
| `connection::{AuthState, ConnectionState}` | `pub` (`:36`, `:53`) | `handlers/*`, `api/*`, `audio/*` |
| `connection::ConnectionSubscriptions` | `pub(crate)` (`:30`) | `state.rs:34` |
| `subscription::SubscriptionRegistry` + `{ConnId, SubId, SubEntry, IndexKey, RemovedSubscription}` | `pub` (`:12`–`:45`) | `state.rs`, `handlers/side_effects.rs` |
| `handlers::event::filter_fanout_by_access` | `pub` (`event.rs:115`) | tests only outside the file; production callers all in-file (`:224`, `:284`, `:409`) |
| `handlers::event::fan_out_pubsub_event` | `pub` (`event.rs:259`) | `main.rs:827` |
| `handlers::event::fan_out_event_to_local_subscribers` | `pub(crate)` (`event.rs:218`) | `audio/handler.rs:1318`, `api/git/transport.rs:1703`, `side_effects.rs:803`/`:889`/`:2759`/`:2911` |
| `handlers::event::dispatch_persistent_event` | `pub(crate)` (`event.rs:326`) | `handlers/ingest.rs` |
| `handlers::event::bounded_kind_label` | `pub(crate)` (`event.rs:35`) | `handlers/ingest.rs` |
| `handlers::req::{p_gated_filters_authorized, engram_filters_authorized, author_only_filters_authorized, filter_can_match_*, result_gated_count_safe_for_pushdown, is_author_only_event, resolve_request_local_access, apply_access_scope_to_query, build_search_channel_scope_filter, apply_count_fallback_limit, count_fallback_exceeded, FILTER_QUERY_CONCURRENCY, COUNT_FALLBACK_CANDIDATE_LIMIT}` | `pub(crate)` | `count.rs`, `api/bridge.rs` |
| `handlers::req::{build_event_query_from_filter, filter_fully_pushable}` | `pub` (`:737`, `:777`) | `count.rs`, `api/bridge.rs` |
| `handlers::auth::extract_auth_tag_json` | `pub` (`auth.rs:28`) | in-file tests; git-auth path documented at `auth.rs:3-13` |
| `handlers::resolve_ttl` | `pub` (`mod.rs:42`) | `ingest.rs:2099`, `side_effects.rs:1681` |

---

#### 6. Deltas against `ARCHITECTURE.md` §2

| `ARCHITECTURE.md` claim | Code | Verdict |
|---|---|---|
| Wire table lists 4 inbound / 6 outbound (`:150-159`) | 5 inbound (`protocol.rs:23-42`), 7 outbound (`protocol.rs:181-208`) | **incomplete** — `COUNT` is missing in both directions |
| "Max frame size: 65,536 bytes" (`:161`) | `DEFAULT_MAX_FRAME_BYTES = 512 * 1024` (`config.rs:14`) | **wrong by 8×** |
| "Max subscriptions per connection: 1024" (`:161`) | `MAX_SUBSCRIPTIONS = 1024` (`req.rs:26`) | correct |
| "Max historical results per filter: 500" (`:161`) | `MAX_HISTORICAL_LIMIT = 2_000` (`req.rs:25`) | **wrong by 4×** |
| Does not mention the 10-filter or 256-byte sub_id caps | `protocol.rs:11`, `:14` | **omission** (both are NIP-11-advertised) |
| §3 Step 3 describes auth without any deadline | `AUTH_TIMEOUT = 5 s`, `connection.rs:27`, `:228-252` | **omission** |
