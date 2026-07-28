## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Capabilities delivered over WebSocket

| Capability | NIP | Implemented at | Notes |
|---|---|---|---|
| Relay-initiated challenge/response auth | NIP-42 | `connection.rs:182-186` + `handlers/auth.rs:44-296` | mandatory within 5 s (`connection.rs:27`) |
| Owner-attestation agent auth (agent→owner delegation) | NIP-OA | `auth.rs:28-42`, `:224-281` | tag extracted from the signed AUTH event, so it is signature-protected |
| Signed event submission | NIP-01 | `event.rs:585-736` | delegates persistent kinds to the shared ingest seam |
| Ephemeral event relay (20000–29999), never stored | NIP-16 range | `event.rs:739-874` | WS-only; HTTP explicitly rejects 1059 and 20001 (`ingest.rs:1448-1452`) |
| Presence set/clear | Buzz kind 20001 | `event.rs:772-800` | Redis-backed; falls through to global ephemeral fan-out |
| Encrypted agent observer frames (bidirectional) | Buzz kind 24200 | `event.rs:920-1069` | NIP-44 content, structural direction inference, own rate limiter |
| Filter subscriptions with historical backfill + EOSE | NIP-01 | `req.rs:44-418` | one DB query per filter, bounded concurrency 4 |
| Full-text search subscriptions | NIP-50 | `req.rs:504-732` | one-shot (no fan-out registration), paginated |
| Aggregate counts | NIP-45 | `count.rs:29-281` | exact-or-refuse (5000-candidate budget) |
| Subscription cancellation | NIP-01 | `close.rs:12-30` | idempotent |
| Channel (group) scoping via `#h` | NIP-29 | `req.rs:1013-1039`, `count.rs:16-26`, `subscription.rs:265-330` | no dedicated message types |
| Gift-wrapped DMs | NIP-17/59 | pubkey-mismatch exemption `event.rs:636-645`; read gate `req.rs:1042-1074` | WS-only ingest |
| Live cross-node delivery | — | `event.rs:259-309` ← `main.rs:818-845` | Redis pub/sub fan-in |
| Heartbeat / dead-peer detection | RFC 6455 | `connection.rs:378-405` | 30 s ping, 3 misses |
| Slow-client shedding | — | `connection.rs:88-113`, `state.rs:436-475` | 15-strike grace, shared counter |
| Live ban enforcement on open sockets | — | `state.rs:290-325` (called from moderation) + `auth.rs:105-183` | frame-then-close idiom |
| Graceful-restart signalling | RFC 6455 1012 | `state.rs:352-374` | sticky drain flag |
| Community archival closing live sockets | — | `state.rs:99-176`, `connection.rs:132-140` | plus a periodic durable revalidation backstop |
| Runtime conformance tracing on the read seam | — | `req.rs:110-116`, `:141-148`, `:337-372`, `:626-668` | `NoopTracer` in production (`state.rs:798`) |

---

#### 2. Realtime delivery path, end to end

##### 2.1 Ingest → fan-out (persistent event, same pod)

```
client ──["EVENT",e]──▶ recv_loop                     connection.rs:419-436
                        ├─ frame size check           connection.rs:421-434
                        ├─ ClientMessage::parse       protocol.rs:66-76
                        ├─ enforce_ws_admission       connection.rs:498-500 → :594-653
                        │    · WsEvents  (5 s window)
                        │    · Messages  (60 s window)
                        ├─ handler_semaphore permit   connection.rs:513-521
                        └─ tokio::spawn(handle_event) connection.rs:530-536
handle_event                                          event.rs:585
  ├─ auth read                                        event.rs:611-631
  ├─ pubkey match (1059 exempt)                       event.rs:636-645
  ├─ kind 22242 reject                                event.rs:647-655
  ├─ kind 24200 → observer branch                     event.rs:657-669
  ├─ ephemeral → ephemeral branch                     event.rs:675-696
  └─ ingest_event(state, tenant, event, IngestAuth)   event.rs:705
       └─ (verify, membership, DB insert, …)          handlers/ingest.rs:1367+
       └─ dispatch_persistent_event                   event.rs:326
            ├─ AWAITED: enqueue_event_created_audit   event.rs:328-336  (bounded chan, cap 1000)
            └─ spawn dispatch_persistent_event_inner  event.rs:352-370
                 ├─ mark_local_event                  event.rs:394
                 ├─ pubsub.publish_event(topic)       event.rs:395-406
                 ├─ sub_registry.fan_out_scoped       event.rs:407-410
                 ├─ filter_fanout_by_access(threaded) event.rs:409-417
                 ├─ DM-visibility owner fence         event.rs:436-472
                 ├─ frame cache + send_to_text_bytes  event.rs:470-479
                 └─ spawn workflow_engine.on_event    event.rs:503-534
◀──["OK",id,true,""]── conn.send                      event.rs:719-723
◀──["EVENT",sub,e]──── per matching subscription      event.rs:76-98
```

The `OK` and the `EVENT` frames are on **independent** timelines: the `OK` is emitted as soon as `dispatch_persistent_event` returns (which is after the audit enqueue only), while fan-out happens in a spawned task (`event.rs:352`). A client can therefore observe its own event arriving on a subscription **before or after** its `OK`.

##### 2.2 Ingest → fan-out (ephemeral / observer, same pod)

Shorter: verify on `spawn_blocking` → optional membership check → `mark_local_event` → `publish_event` → `fan_out_event_to_local_subscribers` → `OK true`.

- channel-scoped ephemeral: `event.rs:808-842` (topic `Channel(ch)`)
- channel-less ephemeral: `event.rs:843-871` (topic `Global`)
- observer frame: `event.rs:1046-1066` (topic `Global`, always)

`fan_out_event_to_local_subscribers` (`event.rs:218-256`) is the canonical guarded local send: `fan_out_scoped` → `filter_fanout_by_access(…, None)` → serialise once → frame cache → `send_to_text_bytes`. It has six other production callers outside this group (`audio/handler.rs:1318`, `api/git/transport.rs:1703`, `side_effects.rs:803`/`:889`/`:2759`/`:2911`) — i.e. huddle audio events, git push notifications, and NIP-29/NIP-25 side effects all reach WS subscribers through this same gate.

##### 2.3 Where cross-node Redis events re-enter

Single re-entry point:

```
another pod ──▶ Redis PSUBSCRIBE buzz:{community}:{topic}
                  └─ buzz-pubsub subscriber task
                       └─ broadcast → pubsub.subscribe_local()
main.rs:818-845   loop { rx.recv() → fan_out_pubsub_event(&state, ev) }
event.rs:259-309  fan_out_pubsub_event
                    ├─ topic → Option<Uuid>              event.rs:264-267
                    ├─ StoredEvent::new(event, ch)       event.rs:269
                    ├─ local-echo dedup (community,id)   event.rs:272-281   ← consume-on-read
                    ├─ sub_registry.fan_out_scoped       event.rs:283
                    ├─ filter_fanout_by_access(…, None)  event.rs:284
                    ├─ buzz_multinode_fanout_total       event.rs:285
                    └─ frame cache + send_to_text_bytes  event.rs:288-307
```

Notes verified against the code:
- The **community label comes from the parsed Redis channel** (`channel_event.community_id`, `event.rs:268`), not from the event body — so a forged event body cannot relabel a delivery.
- The **`threaded` visibility argument is `None`** on this path (`event.rs:284`), so a cross-node private-channel delivery always pays a fresh visibility read (or a cached `private`).
- The **DM-visibility owner fence (BR-WS-110) does not exist on this path.** It lives only in `dispatch_persistent_event_inner` (`event.rs:436-472`). A kind 30622 / 44200 event that reaches a second pod via Redis is fanned out with only the four `filter_fanout_by_access` fences applied — neither of those kinds is in `AUTHOR_ONLY_KINDS` (`buzz-core/src/kind.rs:120`), so the owner check is absent. See the security and debt aspects.
- Broadcast lag is counted (`buzz_multinode_fanout_lag_total`, `main.rs:834`) but **not** repaired — lagged events are lost, not re-fetched.

##### 2.4 Redis subscription demand is driven only by REQ

`retain_topic` is called from exactly **one** production site: `req.rs:256`, after a successful non-search registration. The three `release_topic` sites are `req.rs:251` (replaced subscription), `close.rs:21` (explicit CLOSE), `connection.rs:268` (per removed subscription at disconnect).

Consequence: a pod that only *publishes* never PSUBSCRIBEs. `handlers/event.rs:1683` and `:1687` do call `retain_topic(&tenant, EventTopic::Global)` — but both are inside `#[cfg(test)]` (the Redis round-trip test, module begins `event.rs:1135`), precisely because the test needed to force the demand-driven subscription that production only creates via REQ (see the comment at `event.rs:1669-1676`). This is correct behaviour, not a gap, but it means **local echo suppression on the publishing pod only matters when that pod also has a subscriber for the topic**.

---

#### 3. Fan-out index selection (the performance feature)

`fan_out_scoped` (`subscription.rs:265-330`) is sub-linear in total subscriptions:

| Event shape | Indexes consulted | Site |
|---|---|---|
| `channel_id = Some(ch)` | `channel_kind_index[(community,{ch,kind})]`, then `channel_wildcard_index[(community,ch)]` | `:271-289` |
| `channel_id = None` | `global_p_kind_index[{community,kind,p}]` for **each** distinct `p` tag, then `global_kind_index[(community,kind)]`, then `global_wildcard_index[community]` | `:290-318` |

Every candidate still runs the full `filters_match` predicate (`:369-387`), with a `seen` set suppressing duplicates when a subscription appears in two indexes. The `(kind, #p)` index exists specifically so an observer-frame or membership-notification broadcast does not scan every subscription of that kind — pinned by `subscription.rs:1218-1250`.

---

#### 4. Frame-serialisation optimisation

One `serde_json::to_string` per event per fan-out cycle (`event.rs:229-236`, `:288-295`, `:420-431`), then one `Arc<Bytes>` per distinct `sub_id` (`fanout_frame_cache`, `event.rs:63-74`), cloned per recipient without copying the body (`state.rs:444-448`). Byte-for-byte compatibility with the legacy `format!` output is pinned at `event.rs:1155-1166`, and cross-cycle `Arc` reuse is explicitly forbidden and tested at `event.rs:1168-1188`.

Outbound writes are batched: up to 64 data frames per `flush()` (`connection.rs:347-369`), with control frames always drained first (`connection.rs:322-325`).

---

#### 5. Observability emitted by this group

| Metric | Type | Site |
|---|---|---|
| `buzz_ws_connections_total{community}` | counter | `connection.rs:184-188` |
| `buzz_ws_connections_active` | gauge | `connection.rs:196`, `:286` |
| `buzz_ws_auth_timeouts_total` | counter | `connection.rs:243` |
| `buzz_ws_backpressure_disconnects_total` | counter | `connection.rs:101`, `state.rs:466` |
| `buzz_ws_send_batch_size` | histogram | `connection.rs:368` |
| `buzz_admission_rejections_total{transport,reason}` | counter | `connection.rs:663`, `:671` |
| `buzz_auth_attempts_total{method}` | counter | `auth.rs:84` |
| `buzz_auth_failures_total{reason}` | counter | `auth.rs:169-171`, `:210`, `:235`, `:288` — reasons `banned`, `ban_check_error`, `allowlist_denied`, `not_relay_member`, `nip42_invalid` |
| `buzz_subscriptions_active` | gauge | `subscription.rs:83`, `:170`, `:192` |
| `buzz_events_received_total{kind}` | counter | `event.rs:601` (kind label bounded by `bounded_kind_label`, `:35-53`) |
| `buzz_community_events_received_total{community}` | counter | `event.rs:605-609` |
| `buzz_event_processing_seconds` | histogram | `event.rs:717` |
| `buzz_fanout_recipients` | histogram | `event.rs:225`, `:418` |
| `buzz_multinode_fanout_total` | counter | `event.rs:285` |
| `buzz_post_commit_dispatch_scheduled_total` | counter | `event.rs:351` |
| `buzz_post_commit_dispatch_errors_total{stage}` | counter | `event.rs:426` |
| `buzz_audit_send_errors_total` | counter | `event.rs:576` |
| `buzz_req_global_access_resolution_skips_total{kind}` | counter | `req.rs:90` |
| `buzz_count_fallback_rejections_total` | counter | `count.rs:178`, `:248` |
| `buzz_workflow_runs_total{trigger,community}` | counter | `event.rs:522-527` |

Cardinality is deliberately managed: kind is fleet-wide only and community is kind-free, because `bounded_kind_label` passes through all 10 000 client-controlled values in 20000–29999 (rationale `event.rs:597-600`).

Tracing spans: `ws.auth` (`connection.rs:505`), `ws.event` (`:524-529`), `ws.req` (`:551`), `ws.count` (`:572`) — each captured *before* the `tokio::spawn` so context propagates (comment `:522-523`).

---

#### 6. Capabilities notably **absent** from the WS surface

| Missing | Evidence |
|---|---|
| NIP-01 `["CLOSED", sub, "…"]` on relay-side subscription eviction by the reaper | `side_effects.rs:62` removes registry entries; no `CLOSED` is emitted from this group's code for that path |
| Any per-IP connection cap | `check_ip_connection` never called (`buzz-pubsub/src/rate_limiter.rs:112` has no relay caller) |
| Subscription-level result limits beyond per-filter `limit` | only `MAX_HISTORICAL_LIMIT` per filter (`req.rs:885`) |
| NIP-45 `HLL` / approximate counts | `RelayMessage::count` emits only `{"count": N}` (`protocol.rs:208`) |
| Compression (permessage-deflate) | no negotiation anywhere in `router.rs:334-342` |
| Backfill resumption / cursors | EOSE is terminal; no `since`-cursor handshake |
