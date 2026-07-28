## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. `buzz-db` — every call from this group

| Call | Caller | Purpose | Failure handling |
|---|---|---|---|
| `is_community_active(community)` | `connection.rs:133` (closure passed to `run_registered_community_connection`) | durable community revalidation after socket registration | anything other than `Ok(true)` cancels the socket (`state.rs:149-152`) — fail closed |
| `moderation_restriction_state(community, pubkey)` | `auth.rs:119-130` | ban seam on the authenticated pubkey | `Err` → `BanOutcome::DbError` → deny with `error: internal …` |
| `moderation_restriction_state(community, owner)` | `auth.rs:136-156` | NIP-OA owner ban cascade | same |
| `is_pubkey_allowed(community, pubkey)` | `auth.rs:189-212` | allowlist gate (only when enabled **and** `Nip42`) | `Err` → `false` → deny (fail closed, `auth.rs:195-201`) |
| `is_agent_owner(community, agent, owner)` | `event.rs:998-1002` | observer-frame publish authorisation | `Err` → `OK false "error: internal server error"` |
| `is_member(community, channel, pubkey)` | `req.rs:145-149` (REQ up-front confirm), `count.rs:126-138` (per-filter confirm) | uncached membership confirmation to repair a stale cache-negative | `Err` → `CLOSED "error: database error"` |
| `query_events(&EventQuery)` | `req.rs:307` (one per filter, concurrency 4), `count.rs:175`, `:245` (fallback) | historical/candidate rows | REQ → `EOSE` + return (**not** `CLOSED`, `req.rs:320-325`); COUNT → `CLOSED "error: {e}"` |
| `count_events(&EventQuery)` | `count.rs:164`, `:235` | exact fast-path count | `CLOSED "error: {e}"` |
| `get_events_by_ids(community, ids)` | `req.rs:641-645` | batch fetch of FTS hit ids | `warn!` + `break` out of the page loop (partial results, then EOSE) |
| `communities_of_channels(&[Uuid])` | `req.rs:355`, `:648` | conformance row-community projection **only** | `Err` → `warn!` + empty map; the emit's own guard-rail handles it |
| `db.clone()` | `req.rs:295` | cloned into the `buffered` stream | — |

Indirect (through `AppState` helpers, `state.rs`):

| Helper | Underlying | Caller | Cache |
|---|---|---|---|
| `get_accessible_channel_ids_cached` | `db.get_accessible_channel_ids` | `req.rs:94-98`, `count.rs:79-83` | moka, 10 s TTL, cap 10 000 (`state.rs:747-753`) |
| `is_member_cached` | `db.is_member` | `event.rs:186-189` (fan-out gate) | moka, 10 s TTL, cap 10 000 (`state.rs:740-746`) |
| `channel_visibility_cached` | `db.get_channel` | `event.rs:174-177` | moka, 10 s TTL — **caches only `"private"`** (`state.rs:1124-1150`) |

The read path is **not** replica-routed at this layer; `AppState.db` is a single `Db` and the optional `READ_DATABASE_URL` (`config.rs:57-59`) is handled inside `buzz-db`.

---

#### 2. `buzz-auth`

| Item | Used at |
|---|---|
| `generate_challenge()` | `connection.rs:157` |
| `AuthService::verify_auth_event(event, challenge, relay_url)` | `auth.rs:86-90` — the whole of NIP-42 verification; pure crypto |
| `AuthService::config().rate_limits` | `connection.rs:612` |
| `AuthContext` (`pubkey`, `scopes`, `channel_ids`, `auth_method`, `agent_owner_pubkey`) | stored in `AuthState::Authenticated` (`connection.rs:45`), read at `event.rs:611-631`, `req.rs:50-87`, `count.rs:38-50`, `connection.rs:604-609` |
| `AuthMethod::Nip42` | `auth.rs:191` (allowlist scoping) |
| `Scope::{MessagesRead, MessagesWrite}` | `req.rs:54`; `event.rs:658`, `:676` |
| `LimitType::{WsEvents, Messages}` | `connection.rs:619`, `:642` |
| `RateLimiter` trait (via `crate::admission::check_principal`) | `admission.rs:18-42` |
| `Nip98ReplayGuard` | not used by this group (HTTP only) |

`AuthService` is **not** consulted for authorization — only for verification and for reading rate-limit config. Every authz decision on the WS path is made in `handlers/*` against `buzz-db` and `buzz-core::kind` sets.

---

#### 3. `buzz-pubsub`

##### 3.1 Publish

| Topic | Caller | Preceded by `mark_local_event` |
|---|---|---|
| `EventTopic::Channel(ch)` | `event.rs:399` (persistent, when `stored_event.channel_id` is `Some`), `event.rs:828` (channel-scoped ephemeral) | yes (`:394`, `:824`) |
| `EventTopic::Global` | `event.rs:399` (persistent, channel-less), `event.rs:856` (channel-less ephemeral), `event.rs:1050` (observer frame) | yes (`:394`, `:852`, `:1046`) |

Every publish failure invalidates the local-echo mark before logging (`event.rs:400-405`, `:830-836`, `:858-864`, `:1052-1058`) — otherwise a failed publish would suppress a later legitimate delivery of the same id.

##### 3.2 `retain_topic` / `release_topic` lifecycle

Refcounting lives in `buzz-pubsub`: `desired_topics: HashMap<EventTopicKey, usize>`; `retain_topic` PSUBSCRIBEs only on the 0→1 transition (`buzz-pubsub/src/lib.rs:192-208`), `release_topic` schedules a debounced unsubscribe on the 1→0 transition and warns on an unmatched release (`:215-232`).

| Operation | Site | Topic |
|---|---|---|
| **retain** — after successful non-search REQ registration | `req.rs:254-257` | `topic_for_subscription(channel_id)` (`req.rs:1225-1230`) |
| **release** — the subscription that this REQ replaced | `req.rs:248-253` | `topic_for_subscription(replaced.channel_id)` |
| **release** — explicit CLOSE | `close.rs:20-25` | `topic_for_subscription(removed.channel_id)` (`close.rs:30-35`) |
| **release** — one per subscription at disconnect | `connection.rs:265-270` | `topic_for_subscription(removed.channel_id)` (`connection.rs:681-686`) |
| retain (test-only) | `event.rs:1683`, `:1687` | `EventTopic::Global`, inside `#[cfg(test)]` |

Balance audit:
- Every `register_scoped` is followed by exactly one `retain_topic`, and its `Option<RemovedSubscription>` return drives exactly one compensating `release_topic` — so a same-`sub_id` replacement is net-neutral when the scope is unchanged, and a correct swap when it changes (`req.rs:241-257`).
- `remove_connection` returns **one `RemovedSubscription` per subscription** (`subscription.rs:181-196`), and `connection.rs:265-270` releases once per element — so N subscriptions on the same topic produce N retains and N releases. Correct.
- **Three identical private copies** of `topic_for_subscription` exist (`req.rs:1225-1230`, `close.rs:30-35`, `connection.rs:681-686`). See the debt aspect.
- Search REQs never retain (they return at `req.rs:233`), so the counts stay balanced.

**Unbalanced-release risk found:** `close.rs:16` removes the entry from `conn.subscriptions` **before** `sub_registry.remove_subscription` at `:20`. Two concurrent CLOSE frames for the same `sub_id` cannot double-release, because `remove_subscription` is the one that returns `Some` and it is guarded by the DashMap entry removal (`subscription.rs:164-172`) — the second call returns `None`. Verified safe. The `conn.subscriptions` removal is not the guard.

##### 3.3 Subscribe (fan-in)

`main.rs:818-845` holds the only `subscribe_local()` consumer in production; it calls `fan_out_pubsub_event` (`event.rs:259`). Lag → `buzz_multinode_fanout_lag_total` and a warning (`main.rs:833-836`); a closed broadcast channel logs an error and ends the loop (`:840-842`) — **the loop is not restarted**, so a closed channel silently ends cross-node delivery for the process lifetime.

##### 3.4 Other pub/sub channels this group depends on

| Channel | Consumer | Effect on this group |
|---|---|---|
| cache invalidation | `main.rs:846-877` → `state.apply_cache_invalidation` | drops the membership / accessible-channels / visibility moka entries this group's gates read |
| conn control (`DisconnectPubkey`, `DisconnectCommunity`) | dispatched in `main.rs` → `conn_manager.disconnect_pubkey` / `community_connections.disconnect_community` | closes sockets owned by this group |
| presence (`set_presence` / `clear_presence`) | `event.rs:791-799`, `connection.rs:274-284` | Redis-side presence state |

---

#### 4. `buzz-search`

| Item | Used at |
|---|---|
| `SearchService::search(&SearchQuery)` | `req.rs:602-610` |
| `SearchQuery { community, q, channel_scope, kinds, authors, since, until, page, per_page, mode }` | built `req.rs:596-608` |
| `SearchMode::FullText` | `req.rs:607` — the only mode used |
| `ChannelScope::{Channels, ChannelsOrChannelLess, ChannelLessOnly}` | `req.rs:483-501`, `:580` |

Contract: search returns **event ids only** (`req.rs:637`); the full events are then fetched from Postgres (`req.rs:641-645`) and re-post-filtered (`req.rs:685-712`). This is why the sensitive-kind gates must run *before* the search branch (`req.rs:175-205`) — search hits skip the per-filter historical post-check chain by construction. A search error is non-fatal: `warn!` + `break` out of the page loop, then EOSE (`req.rs:611-616`).

---

#### 5. `buzz-conformance` (via `crate::conformance`)

| Emit | Site | Guard |
|---|---|---|
| `state_for_request(tenant, pubkey)` — builds the `AbstractState` once per REQ | `req.rs:110-116` | `None` only on malformed pubkey bytes |
| `record_req_authcheck(tracer, state, ch_id, member)` | `req.rs:141-148` | only on the DB-confirmation branch |
| `record_read_message_rows(tracer, state, per_filter_channel, &row_channels, &channel_communities)` | `req.rs:337-372` | non-search lane, per filter |
| `record_read_by_id_rows(tracer, state, None, &row_channels, &channel_communities)` | `req.rs:626-668` | search lane, per page |

Production binds `NoopTracer` (`state.rs:798`), so these are zero-cost — **except** the two `db.communities_of_channels` round-trips at `req.rs:355` and `:648`, which are issued **unconditionally** whenever `trace_state.is_some()` (i.e. always, in practice) regardless of whether the tracer is a no-op. That is a per-filter and per-search-page extra query on the hot read path with no production benefit. See the debt aspect.

No conformance emit exists on the write/fan-out side of this group; the comment at `event.rs:374-380` explains that acceptance is recorded at the ingest seam and fan-out surfaces as `ReadMessageRows` on the subscriber side.

---

#### 6. `buzz-core`

| Item | Used at |
|---|---|
| `filter::filters_match` | `subscription.rs:377`, `req.rs:334`, `:687`, `count.rs:187`, `:257` |
| `filter::reader_authorized_for_event` | `req.rs:311`, `:697`, `count.rs:195`, `:265` |
| `verification::verify_event` | `event.rs:749` (ephemeral), `:927` (observer) — both on `spawn_blocking` |
| `event::StoredEvent::{new, with_received_at}` | `event.rs:269`, `:841`, `:869`, `:1060` |
| `kind::{event_kind_u32, is_ephemeral, is_workflow_execution_kind, is_command_kind, is_parameterized_replaceable}` | `event.rs:588`, `:675`, `:509-510`; `req.rs:781-786`, `:944-950` |
| `kind::{AUTHOR_ONLY_KINDS, P_GATED_KINDS, RESULT_GATED_KINDS}` | `event.rs:137`; `req.rs:1046`, `:1156`, `:1139`, `:1188`, `:1208` |
| `kind::{KIND_GIFT_WRAP, KIND_AUTH, KIND_AGENT_OBSERVER_FRAME, KIND_PRESENCE_UPDATE, KIND_DM_VISIBILITY, KIND_AGENT_TURN_METRIC, KIND_AGENT_ENGRAM, KIND_NIP43_MEMBERSHIP_LIST}` | `event.rs:636`, `:647`, `:657`, `:772`, `:438-439`; `req.rs:836`, `:1065`, `:1114` |
| `observer::{content_looks_like_nip44, OBSERVER_AGENT_TAG, OBSERVER_FRAME_TAG, OBSERVER_FRAME_TELEMETRY, OBSERVER_FRAME_CONTROL}` | `event.rs:1072-1090` |
| `tenant::{TenantContext, CommunityId}` | throughout |

---

#### 7. `buzz-audit`

Only via the bounded channel: `state.audit_tx` (capacity 1000, `state.rs:654`), written at `event.rs:574` with `send().await`. The worker (`state.rs:658-696`) does the DB write. Channel-closed → `error!` + `buzz_audit_send_errors_total` (`event.rs:574-577`); the event is **not** rejected. When `audit_enabled == false`, `audit_tx` is `None` and the enqueue short-circuits (`event.rs:548-550`).

---

#### 8. `buzz-workflow`

`workflow_engine.on_event(community, &stored_event)` is spawned from `dispatch_persistent_event_inner` (`event.rs:512-533`). The community is passed **explicitly** because `StoredEvent` does not carry it and the same channel UUID can exist in another tenant (rationale `event.rs:505-509`). Skipped for workflow-execution kinds, command kinds, relay-signed `buzz:workflow`-tagged events, and kind 1059 (`event.rs:503-511`).

---

#### 9. Slow-client / backpressure handling

Two sender surfaces, one shared counter:

| Surface | Method | Site |
|---|---|---|
| direct (handler → its own socket) | `ConnectionState::send(String)` → `send_tx.try_send(Text)` | `connection.rs:88-113` |
| fan-out (any task → any socket) | `ConnectionManager::send_to(String)` / `send_to_text_bytes(Arc<Bytes>)` → `try_send_ws_message` | `state.rs:436-438`, `:443-447`, `:449-474` |

Shared state: `backpressure_count: Arc<AtomicU8>` created at `connection.rs:164`, handed to both `ConnectionState` (`:178`) and `ConnEntry` (`:210`). Semantics (identical in both):

1. `Ok` → `store(0)` — a single successful send fully forgives accumulated strikes (`connection.rs:92`, `state.rs:456`).
2. `Full` → `fetch_add(1)+1`; `>= grace_limit` (15) → `warn!` + `buzz_ws_backpressure_disconnects_total` + `cancel()`; otherwise a graded warning (`connection.rs:95-107`, `state.rs:458-472`).
3. `Closed` → `debug!`, return `false`, **no** strike (`connection.rs:108-111`, `state.rs:473-476`).

Callers that react to a `false` return: `req.rs:398-400` and `:720-722` abandon delivery (and skip EOSE); `event.rs:81-91` counts drops into `drop_count` and logs an aggregate warning (`:249-255`, `:299-305`, `:474-479`).

Control-plane backpressure is treated as **terminal**, not graded: a full 8-slot `ctrl_tx` closes the connection in the heartbeat loop (`connection.rs:396-400`) and on a client Ping (`connection.rs:464-472`). Best-effort ctrl sends that do *not* close: ban reason (`auth.rs:177-179`), `disconnect_pubkey` reason (`state.rs:325-328`), restart close (`state.rs:359`, `:229`).

Read-side backpressure: there is none. `recv_loop` awaits `ws_recv.next()` and, for EVENT/REQ/COUNT, spawns a task under `handler_semaphore` and immediately loops (`connection.rs:530-536`, `:552-558`, `:573-579`). `AUTH` and `CLOSE` are awaited inline (`:506-508`, `:582`), which is the only inbound self-throttle.

---

#### 10. Integration failure-mode summary

| Dependency | On failure | Posture |
|---|---|---|
| Postgres — community active check | socket cancelled | fail closed |
| Postgres — ban state | deny with `error: internal …` | fail closed, cause-preserving |
| Postgres — allowlist | deny | fail closed |
| Postgres — accessible channels | `CLOSED "error: database error"` | fail closed |
| Postgres — membership confirm | `CLOSED "error: database error"` | fail closed |
| Postgres — visibility (fan-out) | recipient list emptied | fail closed |
| Postgres — membership (fan-out) | that recipient dropped | fail closed |
| Postgres — historical query | `EOSE`, subscription stays live | **fail open (silent)** |
| Postgres — COUNT | `CLOSED "error: {raw}"` | fail closed, **leaks error text** |
| Postgres — `get_events_by_ids` (search) | partial page, then EOSE | **fail open (silent)** |
| Postgres — `communities_of_channels` | empty map, trace-only | fail soft |
| Redis — admission limiter | frame rejected | fail closed |
| Redis — `publish_event` | echo mark invalidated, `warn!`, local fan-out still happens | fail open (single-node delivery only) |
| Redis — presence | ignored (`let _ =`) | fail open |
| Redis — cache-invalidation publish | spawned, warn-only | fail open, ≤10 s TTL backstop |
| Redis — broadcast lag | counted, events lost | fail open |
| Redis — broadcast closed | error logged, **loop exits permanently** | fail open |
| Search (FTS) | `warn!` + break, then EOSE | fail open (silent) |
| Audit channel closed | `error!` + counter, event still accepted | fail open |
| Workflow engine | `error!` in spawned task | fail open |
