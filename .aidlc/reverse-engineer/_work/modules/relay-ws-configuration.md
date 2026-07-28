## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Configuration

Every value this group reads comes from `AppState.config: Arc<Config>` (`state.rs:490`). Struct definition: `config.rs:51` onward; loader: `Config::from_env` at `config.rs:405`, called exactly once at `main.rs:122`.

> The WS path reads config through `state.config.*` on every frame (e.g. `connection.rs:420`, `:439`) rather than snapshotting it per connection — but there is no reload path, so the values are effectively immutable for the process lifetime.

---

#### 1. Env vars consumed directly by this group

| Env var | Config field | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_MAX_CONNECTIONS` | `max_connections` | **10 000** (`config.rs:449-452`) | `state.rs:727` (`Semaphore::new`), acquired `connection.rs:149` | hard cap on concurrent WS connections; exhaustion rejects with **no frame sent** (`connection.rs:151-154`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `max_concurrent_handlers` | **1024** (`config.rs:454-457`) | `state.rs:728`, acquired `connection.rs:513`/`:541`/`:563` | cap on in-flight EVENT/REQ/COUNT handler tasks; exhaustion → `rate-limited: too many concurrent requests` |
| `BUZZ_SEND_BUFFER` | `send_buffer_size` | **1 000** (`config.rs:459-462`) | `connection.rs:159` | per-connection outbound **message** queue depth (not bytes). Depth is what the backpressure strike counter measures |
| `BUZZ_MAX_FRAME_BYTES` | `max_frame_bytes` | **524 288** (512 KiB) — `DEFAULT_MAX_FRAME_BYTES` `config.rs:14`, parsed `:464-468` with a `>0` filter | parser `router.rs:334-342`; app re-check `connection.rs:420`, `:439` | max inbound frame; oversize → one `NOTICE` then disconnect. Doc'd as needing headroom over the 256 KiB content cap after JSON+NIP-44 overhead (`config.rs:11-13`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `slow_client_grace_limit` (`u8`) | **15** (`config.rs:470-473`) | copied to `ConnectionState.grace_limit` `connection.rs:179` and `ConnEntry.grace_limit` `:212`; compared `connection.rs:100`, `state.rs:464` | consecutive buffer-full events tolerated before cancelling. **No `>0` filter** — `0` disconnects on the first full buffer; a value >255 fails to parse and silently falls back to 15 |
| `RELAY_URL` | `relay_url` | `ws://localhost:3000` (`config.rs:428`) | `auth.rs:80-81` via `nip42_expected_relay_url` | the URL a NIP-42 AUTH event's `relay` tag must match; tenant-adjusted per connection |
| `BUZZ_PUBKEY_ALLOWLIST` | `pubkey_allowlist_enabled` | **false** (`config.rs:479-481`) | `auth.rs:189` | when true, NIP-42 pubkey-only auth additionally requires a `pubkey_allowlist` row; DB error → deny |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **false** (`config.rs:483-485`) | `auth.rs:245` | closed vs open relay. On an open relay the NIP-OA owner is still extracted for agent→owner backfill |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **false** (`config.rs:520`) | indirectly, inside `enforce_relay_membership` (`auth.rs:217-223`) | permits NIP-OA owner attestation to *grant* membership on a closed relay |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `auth.rate_limits.human_ws_events_per_sec` | **10** (`buzz-auth/src/rate_limit.rs:116-118`) | `connection.rs:614` → `admission.rs:44-49` | multiplied by the 5 s burst window → **50 frames per 5 s** for EVENT+REQ+COUNT combined |
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `auth.rate_limits.human_messages_per_min` | **60** (`rate_limit.rs:110-112`) | `connection.rs:636` | EVENT-only, 60 s window, human tier |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `auth.rate_limits.agent_standard_messages_per_min` | **120** (`rate_limit.rs:119-121`) | `connection.rs:634` | EVENT-only, 60 s window, agent tier (selected by `agent_owner_pubkey.is_some()`) |

All `BUZZ_RATE_LIMIT_*` values go through `positive_u64_from_env` (`config.rs:270-283`), so `0` or a non-integer is a **hard startup error**, not a silent fallback — verified by `rate_limit_overrides_reject_zero`, `config.rs:1109`.

---

#### 2. Env vars consumed indirectly (services this group calls)

| Env var | Default | Reaches this group via |
|---|---|---|
| `REDIS_URL` | `redis://localhost:6379` (`config.rs:419`) | `admission_rate_limiter` (`state.rs:712`), `pubsub` publish/retain/release, presence |
| `BUZZ_REDIS_POOL_SIZE` | **16** (`config.rs:421-425`) — deadpool's own default (`CPU×2`) is called out as too small for a 2-vCPU pod (`config.rs:61-64`) | shared pool behind admission, presence, pub/sub |
| `DATABASE_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` (`config.rs:410-411`) | every `state.db.*` call listed in the integrations aspect |
| `READ_DATABASE_URL` | unset → all reads on the writer (`config.rs:413-417`) | handled inside `buzz-db`; not visible to this group |
| `BUZZ_AUDIT_ENABLED` | **true** (`config.rs:793`) | when false, `audit_tx` is `None` and `enqueue_event_created_audit` short-circuits (`event.rs:548-550`) — removing the only awaited step in `dispatch_persistent_event` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | unset (`config.rs:691-695`) | `handlers::resolve_ttl` (`mod.rs:42-62`) — consumed by `ingest.rs:2099` / `side_effects.rs:1681`, not by the WS handlers themselves |
| `BUZZ_RELAY_PRIVATE_KEY` | unset → fresh keypair each start (`config.rs:602`) | `state.relay_keypair`, compared at `event.rs:494` to skip workflow triggering for relay-signed events |
| `BUZZ_CORS_ORIGINS` | empty → permissive (`config.rs:595-600`) | applied to the merged router (`router.rs:191`); does not gate the WS upgrade |
| `BUZZ_ADMIN_HOST` | unset (`config.rs:814-836`) | short-circuits `/` before NIP-11 or the WS upgrade (`router.rs:259-277`) — an admin-host request can never open a relay socket |

---

#### 3. Hard-coded values that are *not* configurable

These are the tuning knobs an operator cannot reach:

| Value | Constant | Site | Consequence of being fixed |
|---|---|---|---|
| **5 s** NIP-42 auth deadline | `AUTH_TIMEOUT` | `connection.rs:27` | a client on a high-latency link that cannot sign+round-trip in 5 s cannot connect |
| **30 s** heartbeat interval | — | `connection.rs:383` | not tunable for mobile radio-sleep profiles |
| **3** missed pongs | — | `connection.rs:389-393` | fixed 90 s dead-peer detection |
| **8** control-channel slots | — | `connection.rs:162` | a stalled writer becomes terminal after 8 queued control frames |
| **64** frames per flush | `MAX_WS_SEND_BATCH` | `connection.rs:33` | fan-out write batching |
| **1024** subscriptions/connection | `MAX_SUBSCRIPTIONS` | `req.rs:26` | matches the NIP-11 advertisement |
| **10** filters/REQ | `MAX_FILTERS_PER_REQ` | `protocol.rs:14` | NIP-11 advertised |
| **256 B** sub_id | `MAX_SUB_ID_LENGTH` | `protocol.rs:11` | NIP-11 advertised |
| **2 000** historical rows/filter | `MAX_HISTORICAL_LIMIT` | `req.rs:25` | also clamps the search per-filter limit (`req.rs:538`) |
| **4** concurrent filter queries | `FILTER_QUERY_CONCURRENCY` | `req.rs:35`; compile-time asserted 2..=8 at `:41` | intentionally locked; raising it requires re-running the relay bench (doc `req.rs:37-40`) |
| **10** search pages, **100** hits/page | `MAX_SEARCH_PAGES`, `per_page` | `req.rs:421`, `:589` | search result ceiling is 1000 hits/filter before post-filtering |
| **5 000** COUNT candidate rows | `COUNT_FALLBACK_CANDIDATE_LIMIT` | `req.rs:753` | a non-pushable COUNT over a large set is refused, not approximated |
| **100/s** observer telemetry per agent | — | `event.rs:912` | shared by every deployment size |
| **±300 s** observer freshness | — | `event.rs:952` | tighter than the ingest ±900 s drift window (`ingest.rs:1480`) |
| **±900 s** ingest timestamp drift | `MAX_TIMESTAMP_DRIFT_SECS` | `ingest.rs:1480` | — |
| **256 KiB** event content | `MAX_EVENT_CONTENT_BYTES` | `ingest.rs:1489` | the reason `max_frame_bytes` defaults to 512 KiB |
| **128 B** presence status truncation | — | `event.rs:780-785` | UTF-8-boundary-safe |
| **60 s / 10 000** local-echo cache | — | `state.rs:734-739` | cross-node dedup window |
| **10 s** membership / accessible-channels / visibility TTL | — | `state.rs:740-760` | upper bound on stale-access exposure when a cross-pod invalidation is dropped |
| **300 s / 1 000** observer owner cache | — | `state.rs:782-787` | safe because `agent_owner_pubkey` is immutable within a community (`state.rs:601-606`) |
| **1 000** audit channel capacity | — | `state.rs:654` | the back-pressure point for `dispatch_persistent_event` |
| **5 s** window / **×5** burst multiplier for `WsEvents` | `WS_BURST_WINDOW_SECS` | `admission.rs:10`, `:44-49` | fixed-window, not a token bucket — the code notes a Redis token bucket would be a better long-term fit (`admission.rs:5-8`) |
| **60 s** window for `Messages` | — | `connection.rs:643` | — |

---

#### 4. Startup-time validation relevant to this group

| Check | Behaviour | Site |
|---|---|---|
| `BUZZ_MAX_FRAME_BYTES` must be > 0 | invalid/zero → silent fallback to 512 KiB | `config.rs:464-468` |
| `BUZZ_RATE_LIMIT_*` must be a positive integer | invalid/zero → **`ConfigError::InvalidValue`, startup fails** | `config.rs:270-283` |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | **no validation** — unparsable → 15; `0` accepted verbatim | `config.rs:470-473` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / `BUZZ_SEND_BUFFER` | **no `>0` filter** — `0` is accepted and would make `Semaphore::new(0)` / `mpsc::channel(0)`; `mpsc::channel(0)` **panics** at `connection.rs:159` | `config.rs:449-462` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` warns that REST bypasses token auth and states WS is unaffected | `warn!` only | `config.rs:588-593` |
| Defaults are asserted valid by a test | `config.rs:938-1000` — asserts `max_connections > 0`, `send_buffer_size > 0`, `slow_client_grace_limit > 0`, `max_frame_bytes == DEFAULT_MAX_FRAME_BYTES` | — |

The `defaults_are_valid` test (`config.rs:938`, 17 assertions through `:1000`) asserts the invariants that the **loader does not enforce**. It only covers the default path, so an operator setting `BUZZ_SEND_BUFFER=0` gets a runtime panic on the first connection rather than a config error.

---

#### 5. Configuration deltas against docs

| Claim | Source | Actual | Verdict |
|---|---|---|---|
| "Max frame size: 65,536 bytes" | `ARCHITECTURE.md:161` | `512 * 1024` (`config.rs:14`) | **wrong by 8×** |
| "Max historical results per filter: 500" | `ARCHITECTURE.md:161` | `2_000` (`req.rs:25`) | **wrong by 4×** |
| "After `SLOW_CLIENT_GRACE_LIMIT` (3) consecutive full-buffer events" | `ARCHITECTURE.md:208` | default **15** (`config.rs:473`); the value is a config field, not a constant | **wrong by 5×**, and misnames the mechanism |
| "`BUZZ_MAX_FRAME_BYTES` … default 65536" | `.aidlc/reverse-engineer/configuration.md:89` | 524 288 | **wrong** (inherits the ARCHITECTURE error) |
| "`BUZZ_SLOW_CLIENT_GRACE_LIMIT` … default 3" | `.aidlc/reverse-engineer/configuration.md:91` | 15 | **wrong** |
| "`BUZZ_SEND_BUFFER` … default (code default)" | `.aidlc/reverse-engineer/configuration.md:90` | 1 000 (`config.rs:462`) | resolvable — should be stated |
| "No rate limiting implementation … none are enforced" | `ARCHITECTURE.md:823` | `human_ws_events_per_sec`, `human_messages_per_min`, `agent_standard_messages_per_min` all enforced (`connection.rs:612-650`) | **wrong**; the accurate residual gap is `IpConnections` + the elevated/platform tiers |
| §3 Step 3 describes auth with no deadline | `ARCHITECTURE.md:181-187` | `AUTH_TIMEOUT` 5 s (`connection.rs:27`) | **omission** |
| §3 Step 5 cleanup lists 5 steps | `ARCHITECTURE.md:211-217` | 7 — adds per-subscription `release_topic` (`connection.rs:265-270`) and last-connection presence clear (`:274-285`) | **incomplete** |
| §4 pipeline step 10 "SEARCH INDEX — search_index_tx.send (bounded worker queue)" | `ARCHITECTURE.md:235` | removed; Postgres FTS makes the persisted row the searchable row, and both the worker and the mpsc are gone | **stale** — the code itself documents the removal at `event.rs:479-485` |
| `BUZZ_MAX_CONNECTIONS` / `BUZZ_MAX_CONCURRENT_HANDLERS` / rate-limit env vars | not in `.env.example` per `grep` of the documented table | present in the loader (`config.rs:449`, `:454`, `:288-320`) | **undocumented knobs** |

---

#### 6. Operational tuning notes derived from the code

- **Buffer sizing is coupled to the strike limit.** A 1000-message buffer with a 15-strike grace means a client must fail 15 *consecutive* sends; since any success resets to 0 (`connection.rs:92`), a client that drains slowly-but-steadily is never disconnected. Lowering `BUZZ_SLOW_CLIENT_GRACE_LIMIT` is the only lever, and there is no timeout-based eviction to complement it.
- **`max_frame_bytes` must exceed 256 KiB** or large legitimate events become unsendable: the ingest content cap is 256 KiB (`ingest.rs:1489`) and Nostr JSON + NIP-44 base64 roughly doubles it. The 512 KiB default is the documented reason (`config.rs:11-13`); a 65 536 setting (as `ARCHITECTURE.md` claims is the default) would silently break large messages.
- **The WS burst budget is not the advertised rate.** `human_ws_events_per_sec=10` yields a 5 s fixed window of 50, so a client can legitimately emit 50 frames in the first 10 ms of a window and then be starved for 4.99 s. Desktop startup is the stated reason (`admission.rs:5-7`).
- **Horizontal scaling requires `BUZZ_HUDDLE_AUDIO_AVAILABLE=false`** (`config.rs:114-129`, loader `config.rs:489-492`) — not a WS-protocol knob, but it shares the socket lifecycle registry this group registers into (`connection.rs:132`).
- **`BUZZ_MESH` defaults off** (loader `config.rs:498-500`) and is not consulted anywhere in this group; mesh-on and mesh-off WS behaviour is identical.
