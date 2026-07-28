## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Features

#### Supported

| Feature | Location | Notes |
|---|---|---|
| WebSocket connect + NIP-42 AUTH | `relay.rs:3825-3862` | 30 s connect timeout, 20 s per auth step |
| NIP-OA `auth_tag` in the AUTH event | `relay.rs:3444-3456` | hand-built tag vec; `EventBuilder::auth` used when absent |
| Bounded initial-connect retry | `relay.rs:3786-3822` | 6 attempts, terminal errors short-circuit |
| Autonomous reconnect on socket loss | `relay.rs:2893-3018` | 5 attempts + separate flat DNS retries |
| Caller-driven reconnect with persisted backoff | `relay.rs:3022-3151` | unbounded loop, 60 s tail |
| Channel REQ subscribe / CLOSE | `relay.rs:3160-3222`, `:1419-1432` | sub id `ch-<uuid>` |
| Membership-notification subscribe (44100/44101) | `relay.rs:3227-3270` | global, `#p`-scoped |
| Observer-control subscribe (24200) | `relay.rs:3273-3305` | `#p`-scoped, `since: now` |
| Publish signed events over WS | `relay.rs:2611-2624` | via `RelayCommand::PublishEvent` |
| Typing-indicator construction (kind 20002) | `relay.rs:843-867` | `h` tag + NIP-10 `root`/`reply` `e` tags |
| Non-blocking fire-and-forget publish | `relay.rs:834-840` | `try_send`, drops on full command channel |
| Cloneable publisher handle | `relay.rs:556-591` | lets spawned tasks publish without socket access |
| Client-initiated ping/pong liveness | `relay.rs:1937-1999` | `PING_INTERVAL` 30 s, `PONG_TIMEOUT` 10 s (`relay.rs:48-52`) |
| Server-initiated Ping → Pong | `relay.rs:2370-2376`, `:3903-3907`, `:3963-3967` | all three intake loops |
| Two-generation bounded dedup | `relay.rs:935-972` | 6k–12k ids |
| `since`-based replay with skew | `relay.rs:3185-3194` | −5 s on reconnect |
| Backpressure-driven proactive resubscribe | `relay.rs:2118-2181`, `:1584-1596` | runs on the existing socket |
| Rate-limit gate with jitter + max-extend | `relay.rs:1182-1207` | armed from NOTICE and CLOSED |
| Paced REQ drain (≈8 frames/s) | `relay.rs:1706-1779`, `:2667-2775` | 1 frame per select tick |
| Durable observer-frame parking + in-flight tracking | `relay.rs:1213-1263`, `:2629-2665` | bounded 256, drop-oldest with counter |
| Per-channel access-denial handling without reconnect | `relay.rs:3498-3529` | exact string match only |
| Graceful shutdown with close frame | `relay.rs:900-912`, `:1782-1791` | 5 s wait then abort |
| Channel discovery over HTTP bridge | `relay.rs:657-714` | two `POST /query` calls |
| Archived-channel exclusion | `relay.rs:171-232` | skips `archived=true` |
| NIP-98-authenticated HTTP `POST /query` | `relay.rs:399-406` | retried on 429/5xx |
| NIP-98-authenticated HTTP `POST /events` | `relay.rs:411-423` | used by metrics, reactions, notices |
| NIP-AE core-engram fetch + prompt rendering | `engram_fetch.rs:39-165` | fail-closed on undecryptable candidates |
| In-process observer bus with replay buffer | `observer.rs:83-136` | 1,000-event ring, monotonic `seq` |

#### Stubbed, absent, or partial

| Gap | Evidence |
|---|---|
| **No `COUNT` support** | no `"COUNT"` arm in `parse_relay_message` (`relay.rs:3535-3620`); nothing builds a COUNT frame anywhere in the module |
| **No NIP-50 `search`** | no `search` key in any filter; `RestClient::query` takes raw `nostr::Filter` and no call site sets it (`relay.rs:663`, `:698`, `engram_fetch.rs:79`) |
| **EOSE is inert** | logged only (`relay.rs:2190-2192`); no "initial replay complete" signal to the harness |
| **AUTH OK is not matched to the AUTH event id** | `wait_for_any_ok` accepts the first OK of any id; the comment concedes it (`relay.rs:3846-3854`) |
| **Mid-session AUTH from the handshake buffer is discarded** | `relay.rs:2432-2433` returns `None` for `RelayMessage::Auth` on replay |
| **No `limit` on any WS REQ** | `send_subscribe` / `send_membership_subscribe` / `send_observer_control_subscribe` build no `limit` key (`relay.rs:3170-3194`, `:3232-3250`, `:3274-3285`); only the HTTP engram query sets one (`engram_fetch.rs:88`) |
| **Wildcard REQ possible** | `kinds` omitted when `ChannelFilter.kinds` is `None` (`relay.rs:3172-3175`); reachable via `--subscribe-mode all` without `--kinds` (`config.rs:1180`, `:1272`) and via an empty-`kinds` config rule (`config.rs:1195-1196`) |
| **No `authors` filter on the observer-control REQ** | `relay.rs:3274-3285` — any pubkey's kind:24200 addressed to this agent is forwarded; owner enforcement lives in `lib.rs` |
| **No signature verification of forwarded events** | `relay.rs:2069-2076` and `:2154-2181` forward relay-supplied events verbatim; `engram_fetch.rs:126` is the only place this module calls `event.verify()` |
| **`publish_event` on `HarnessRelay` is unused** | `#[allow(dead_code)]` at `relay.rs:820`; callers use `RelayEventPublisher::publish_event` (`lib.rs:89`, `:829`, `setup_mode.rs:641`) or `try_publish_event` (`lib.rs:2338`) |
| **`_state` parameter unused in `send_subscribe`** | `relay.rs:3162` — takes `&BgState` and never reads it |
| **No `x-auth-tag` on the WS path** | the NIP-OA tag goes into the AUTH event only; there is no per-EVENT delegation tag |
| **Observer bus has no relay transport of its own** | `observer.rs:1-6` states this explicitly; publication is entirely in `lib.rs` |
| **`ObserverHandle::snapshot` silently returns empty on a poisoned mutex** | `observer.rs:96-100` |

#### Typing indicators: kind 20002 published from this crate

`build_typing_event` constructs a kind:20002 (`KIND_TYPING_INDICATOR`,
`buzz-core/src/kind.rs:331`) event with an `h` tag and, when threaded, NIP-10
`["e", root, "", "root"]` + `["e", parent, "", "reply"]` tags
(`relay.rs:849-867`). The publisher is the 3-second `typing_refresh` tick in the
main loop: `lib.rs:2333-2341` calls `relay.build_typing_event(...)` then
`relay.try_publish_event(event)` for every channel in `typing_channels`. The
interval is set at `lib.rs:1593-1596`, gated on `config.typing_enabled`.

This is a different mechanism from the one `ARCHITECTURE.md:452-458` documents.
That section describes a Redis sorted-set protocol
(`ZADD buzz:typing:{channel_id}` / `ZREMRANGEBYSCORE` / `EXPIRE`, 5 s activity
window, 60 s key TTL). No `KIND_TYPING_INDICATOR` reference exists anywhere in
`crates/buzz-relay/src` or `crates/buzz-pubsub/src`; kind 20002 falls in the
generic ephemeral range (`EPHEMERAL_KIND_MIN = 20000`,
`buzz-core/src/kind.rs:321`, `is_ephemeral` at `:621`) and is fanned out via
plain Redis pub/sub without being stored. The documented sorted-set design is
not what this module's typing events exercise.

#### Kind-registry compliance

Every event-kind integer used in production code in this module resolves through
`buzz_core::kind`: `KIND_AGENT_OBSERVER_FRAME`, `KIND_MEMBER_ADDED_NOTIFICATION`,
`KIND_MEMBER_REMOVED_NOTIFICATION`, `KIND_TYPING_INDICATOR` (imported at
`relay.rs:118-122`), `KIND_NIP29_GROUP_MEMBERS` (`relay.rs:665`),
`KIND_NIP29_GROUP_METADATA` (`relay.rs:700`), `KIND_AGENT_ENGRAM`
(`engram_fetch.rs:17`, used `:81`). NIP-42 and NIP-98 use the `nostr` crate's
own `Kind::Authentication` (`relay.rs:3452`) and `Kind::HttpAuth`
(`relay.rs:292`). No bare kind literal appears in the production half of
`relay.rs` (lines 1–3,994) — the only numeric kinds in the file are `Kind::Custom(9)`
inside tests. Kind numbers do appear in doc comments (`relay.rs:162`, `:262`,
`:653-654`, `:842`, `:1036`, `:1469`, `:3225`), which is drift risk but not a
registry bypass.
