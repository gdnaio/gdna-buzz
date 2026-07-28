## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Business Rules

Every rule is transcribed from code, not doc comments. Where the doc and the code
disagree, the delta is called out inline.

---

#### 1. Admission & capacity

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-01 | A huddle-audio connection is bound to a community from the **request `Host` header** before the WebSocket upgrade; an unmapped host fails closed with a generic 404, never a default tenant | `tenant::bind_community(&state.db, raw_host)` | `handler.rs:74-88` |
| BR-AU-02 | An audio connection consumes one permit from the **same** `conn_semaphore` as ordinary relay WebSockets; exhaustion → 503 | `try_acquire_owned()`; permit is moved into the connection task and dropped on exit | `handler.rs:90-99`, `:110-114`, `:167` |
| BR-AU-03 | A connection to a community whose `is_community_active` is false is dropped silently | `run_registered_community_connection` | `handler.rs:156-164` |
| BR-AU-04 | Every WS message is capped at 8192 bytes **at the parser**, before the handler reads it | `max_message_size` + `max_frame_size` | `handler.rs:52`, `:116-119` |
| BR-AU-05 | Auth must complete within 5 s of the challenge or the socket is closed silently | `tokio::time::timeout(AUTH_TIMEOUT, …)` | `handler.rs:61`, `:190` |
| BR-AU-06 | Room capacity is `min(MAX_PEERS_PER_ROOM=25, 255-index-space)`. The cap check is **inside** the admission mutex, so two concurrent joiners cannot both pass | `self.peers.len() >= MAX_PEERS_PER_ROOM` under `guard.lock()` | `room.rs:50`, `:233-238` |
| BR-AU-07 | `peer_index` allocation is 0..=254; `alloc()` returns `None` at `next_fresh == 255`, so **index 255 is never issued** and the effective hard cap is 255 peers | `AdmissionGuard::alloc` | `room.rs:143-153` |
| BR-AU-08 | Freed indices are recycled LIFO (`Vec::pop`) before fresh ones are minted | `AdmissionGuard::{alloc, release}` | `room.rs:144-146`, `:155-157`; pinned `room.rs:730-757` |
| BR-AU-09 | Admission error precedence is **`Ended` > `Full` > `VersionMismatch`** — a client that could not get a seat must not learn the room's pinned protocol version | ordered `if` chain under one lock | `room.rs:220-224` (rationale), `:230-243`; pinned `room.rs:759-782` |
| BR-AU-10 | A room is keyed by **(community, channel)**, so two tenants reusing one channel UUID never share peers, pins, or frames | `DashMap<(CommunityId, Uuid), …>` | `room.rs:490`, `:503-509`; pinned `room.rs:684-728` |

---

#### 2. Authentication & membership

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-11 | Authentication is NIP-42: the relay issues a challenge, the client returns a signed auth event, and `state.auth.verify_auth_event(event, challenge, relay_url)` must accept it | same `buzz_auth` verifier the main relay door uses | `handler.rs:176-238` |
| BR-AU-12 | The expected relay URL for the NIP-42 event is derived **per tenant**, not from a global constant | `api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)` | `handler.rs:219` |
| BR-AU-13 | Relay-level membership is checked via `enforce_relay_membership`, which is a **no-op when `require_relay_membership == false` — the default** | `api/mod.rs:67` early-returns `OpenRelay`; `api/mod.rs:130-131` maps that to `Ok` | `handler.rs:244-262` |
| BR-AU-14 | An **archived** channel is rejected before any membership check, so an auto-ended huddle can never be rejoined | `channel.archived_at.is_some()` first in `ensure_membership` | `handler.rs:1162-1170` |
| BR-AU-15 | For an ephemeral (TTL) channel, the lifecycle parent is **not** taken from the client-supplied `parent_channel_id` on trust: a creator-signed kind-48100 link must exist in the DB | `db.huddle_started_link_exists(community, parent, channel, channel.created_by)` | `handler.rs:1172-1191` |
| BR-AU-16 | A non-TTL channel's lifecycle parent is the channel itself | `else { channel_id }` | `handler.rs:1192-1194` |
| BR-AU-17 | Admission order is: existing member → `visibility == "open"` → auto-add. Anything else is `"not a member"` | sequential checks | `handler.rs:1196-1234` |
| BR-AU-18 | Auto-add applies **only** to a TTL channel whose caller is already a member of the resolved parent; the new row is `MemberRole::Member`, added_by = `channel.created_by` | `db.add_member(...)` then `invalidate_membership` | `handler.rs:1208-1231` |
| BR-AU-19 | After `get_or_create`, the archived check is **repeated** against the DB to close the cross-boundary race where a joiner passed `ensure_membership` before the last peer archived | second `db.get_channel` | `handler.rs:384-413` |
| BR-AU-20 | A DB error on that re-check **fails closed** (silent teardown), it does not fall through to admission | `Err(e) => { … return; }` | `handler.rs:404-410` |

---

#### 3. Protocol version negotiation

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-21 | `protocol_version` defaults to **1** when the auth message omits it, so pre-v2 clients keep working | `#[serde(default = "default_protocol_version")]` | `handler.rs:134-142` |
| BR-AU-22 | A requested version of `0` or `> CURRENT_PROTOCOL_VERSION (2)` is refused with `unsupported_version` **before** the room is touched, so an unknown version can never become a room's pin | range check after the archived re-check | `handler.rs:123-124`, `:415-441` |
| BR-AU-23 | The **first successful admission pins the room's version**; the pin is set *after* index allocation so a `Full` rejection cannot pin | `g.pinned_version.get_or_insert(requested_version)` following `g.alloc()?` | `room.rs:244-247`, `:311` |
| BR-AU-24 | A later joiner whose version differs from the pin is refused `upgrade_required` with both `pinned_version` and `requested_version` | `AdmissionError::VersionMismatch` → `handler.rs:534-548` | `room.rs:239-243` |
| BR-AU-25 | The pin **survives full peer churn** while the `Room` object lives, and is only cleared when `cleanup_if_empty` evicts the room | pin lives in `AdmissionGuard`, dropped with the `Room` | `room.rs:110-127`; pinned `room.rs:661-682`, `:730-757` |
| BR-AU-26 | Frames from a v≥2 room must carry the 8-byte header **plus** a non-empty Opus payload; otherwise they are dropped (frame only, not the connection) | `data.len() <= V2_HEADER_LEN` then `FrameHeader::parse` with `!payload.is_empty()` | `handler.rs:975-1017` |
| BR-AU-27 | The relay never strips, rewrites, or re-encodes a client frame. A malformed `level_dbov` is **clamped in the parsed view only** — the forwarded bytes are unchanged, and the frame is never dropped for bad telemetry | `parse` clamps to `-127`; `broadcast_frame`/`forward_media` take the original `data` | `wire.rs:73-77`, `:14-20`; `handler.rs:1017-1022` |
| BR-AU-28 | `level_dbov` MUST NOT feed any trust decision (admission, moderation, kicks) | stated invariant `wire.rs:16-20`; verified: the only consumer is `tracing::trace!` | `handler.rs:996-1003` |

---

#### 4. Frame fan-out

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-29 | Fan-out is "everyone except the sender", identified by peer UUID on the local path | `if *entry.key() == sender_id { continue; }` | `room.rs:406-408` |
| BR-AU-30 | On the mesh path the author is identified by **`peer_index`**, not peer UUID, so a participant whose own frame round-tripped owner→their-pod does not hear themselves | `deliver_prefixed` skips `entry.peer_index == author_index` | `room.rs:422-429`; `mesh.rs:246-247` |
| BR-AU-31 | Audio uses `try_send` **everywhere** — a full 8-slot per-peer queue drops the frame silently, with no counter and no log | 3 sites | `room.rs:409`, `:427`; `handler.rs:1115` |
| BR-AU-32 | Control JSON uses a separate 32-slot queue; a drop is logged as a warning because joined/left are state-bearing | `broadcast_control` warns on `try_send` failure | `room.rs:437-449` |
| BR-AU-33 | The outbound WS loop **drains all pending control frames before any data frame**, on every iteration and again in a `biased` select | `while let Ok(ctrl_msg) = ctrl_rx.try_recv()` then `biased` | `handler.rs:1060-1086` |
| BR-AU-34 | A frame from a sender not present in the peer map is discarded without error | `peers.get(&sender_id)` → `None => return` | `room.rs:394-397` |
| BR-AU-35 | On the ingress (non-owner) path there is **no local fan-out at all**: media goes only to the owner, which fans it back — including to co-located clients | `match remote_session { Some(s) => s.forward_media(&data), None => room.broadcast_frame(...) }` | `handler.rs:1017-1022`; rationale `join.rs:1453-1462` |
| BR-AU-36 | Media forwarding to the owner is drop-on-error; a dead link never blocks the receive loop | `if let Err(e) = self.transport.send_datagram(...) { debug!(...) }` | `join.rs:1760-1765` |

---

#### 5. Speaking / mute semantics

| ID | Rule | Status |
|---|---|---|
| BR-AU-37 | The relay has **no** notion of mute, speaking, active speaker, or dominant talker. There is no server-side mute state, no `mute` control message, and no speaker-election logic | Verified: grep for `mute\|speaking\|active_speaker\|dominant` across `crates/buzz-relay/src/audio/` finds only two doc-comment mentions (`mod.rs:5`, `wire.rs:14`) |
| BR-AU-38 | Active-speaker detection is entirely client-side, from `level_dbov` | `desktop/src-tauri/src/huddle/playout.rs:220` emits `huddle-active-speakers` |
| BR-AU-39 | Mute is client-side suppression of transmit; the relay cannot distinguish a muted peer from a silent one, except that a DTX-flagged frame is still forwarded | `FLAG_DTX` is parsed and traced only (`handler.rs:1001`) |
| BR-AU-40 | The `PeerCtrl::Json(...)` doc claims control messages include a `speakers` type (`room.rs:33`). **No `speakers` message is ever produced.** Documented delta | Verified: no `speakers` literal in `crates/buzz-relay/src` |

---

#### 6. Room ownership & ending (single-pod)

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-41 | A huddle has **no human owner or moderator**. Room lifetime is purely reference-counted on connected peers | no owner/creator field on `Room` (`room.rs:157-170`) |
| BR-AU-42 | When the **last** peer leaves, the room auto-ends: removal, index recycle, roster bump and `ended = true` all happen under one mutex acquisition | `remove_peer_and_check_ended` | `room.rs:358-388` |
| BR-AU-43 | Exactly one leaver wins the auto-end, so two simultaneous disconnects cannot both archive or both emit 48103 | `if !g.ended && self.peers.is_empty() { g.ended = true; true }` | `room.rs:378-384` |
| BR-AU-44 | Auto-end archives the channel in the DB **before** emitting 48103 | `db.archive_channel` then `emit_participant_event(48103)` | `handler.rs:835-860` |
| BR-AU-45 | If `archive_channel` fails, the end is **rolled back**: `clear_ended()` runs, `room_emptied = false`, no 48103 is emitted, and the huddle stays alive | `Err(e) => { room.clear_ended(); room_emptied = false; }` | `handler.rs:840-845` |
| BR-AU-46 | 48102 (participant left) is emitted on **every** disconnect path, including the ingress path and including when the archive fails | unconditional call | `handler.rs:822-831` |
| BR-AU-47 | An **ingress (non-owner) peer never archives** huddle state: it removes locally and leaves room lifetime to the owner | `if remote_session.is_some() { room.remove_peer(peer_id); false }` | `handler.rs:803-810` |
| BR-AU-48 | An ingress peer also does **not** broadcast `left` locally — the owner's roster delta carries it | `if remote_session.is_none() { room.broadcast_control(left_msg) }` | `handler.rs:818-820` |
| BR-AU-49 | A lifecycle event whose DB insert reports a **duplicate** skips fan-out entirely, to avoid double delivery | `Ok((_, false)) => return` | `handler.rs:1285-1295` |
| BR-AU-50 | A lifecycle event whose DB insert **errors** is still fanned out from an in-memory `StoredEvent`, accepting that late joiners will never see it | `StoredEvent::new(event.clone(), …)` | `handler.rs:1296-1307` |
| BR-AU-51 | Local fan-out precedes Redis publish and is preceded by `mark_local_event`, so the Redis echo is suppressed on this node; a publish failure invalidates that marker | `mark_local_event` → `fan_out_event_to_local_subscribers` → `publish_event` → invalidate on error | `handler.rs:1309-1331` |

---

#### 7. Reconnection / rejoin

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-52 | Heartbeat: a ping every 30 s; 3 consecutive missed pongs closes the connection. `missed_pongs` is incremented **before** the ping is sent and reset by any inbound Pong | `fetch_add(1) + 1 >= MAX_MISSED_PONGS` | `handler.rs:55-58`, `:1127-1151`, `:1038-1040` |
| BR-AU-53 | Effective idle detection window is 60–90 s (increment happens at tick, so failure is observed on the 3rd tick) | derived from `handler.rs:1132-1140` |
| BR-AU-54 | There is **no session resumption**. A reconnect is a brand-new NIP-42 handshake, a new peer UUID, and (usually) a different `peer_index` — clients must rebuild their index→pubkey map from `joined`/`roster` | `Uuid::new_v4()` per admission (`room.rs:255`) |
| BR-AU-55 | A rejoin is refused if the channel was archived by the previous session's auto-end | BR-AU-14 + BR-AU-19 | `handler.rs:1162-1170`, `:389-403` |
| BR-AU-56 | Owner-loss and owner-drain both **close local owner clients so they rejoin**; the relay never migrates a session | `owner_cancel.cancel(); fence.forget(channel_id)` | `handler.rs:727-772` |
| BR-AU-57 | On any teardown cause (`OwnerLost`, `OwnerDraining`, `SessionEnded`, `StreamClosed`) the ingress path takes the **same** action — cancel + `fence.forget`. The cause is observability only | `teardown_remote_huddle` ignores `cause` except for logging | `handler.rs:897-910`; doc `join.rs:1483-1495` |
| BR-AU-58 | `GenerationFloor::forget` clears **local stale-frame suppression only**; it never authorises ownership. Redis fenced CAS remains the arbiter | stated `handler.rs:889-895`, `mesh_boot.rs:154-163`; verified `forget` only does `seen.remove` | `mesh.rs:131-133` |

---

#### 8. Mesh owner / forwarding protocol

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-59 | A huddle's `session_id` **is** its `channel_id` | `resolve_join_owner_ready(…, channel_id, …)` | `handler.rs:324`; doc `join.rs:20` |
| BR-AU-60 | Exactly one pod owns a huddle: the holder of the Redis fenced CAS lease. Mesh membership is a routing hint and **never** grants ownership | doc `join.rs:22-38`; verified — every ownership decision goes through `HuddleDirectory` | `join.rs:317-379` |
| BR-AU-61 | `resolve_join` looks up the live lease **first** and only attempts `acquire` when unowned, so the steady state avoids the generation-INCR race window | `owner_of` → `None` → `acquire` | `join.rs:329-347` |
| BR-AU-62 | A lost CAS is not an error: the winner becomes the routing target (`RemoteOwner`) | `AcquireOutcome::Held(o) => o` | `join.rs:344-346`; pinned `join.rs:2035-2054` |
| BR-AU-63 | `ResolvedJoin.acquired` is `Some` **only** on the CAS-win arm. The steady-state owner reuses the registry's existing renewer instead of rebuilding authority from a generation snapshot | `acquired: None` on both non-acquire arms | `join.rs:293-303`, `:349-378` |
| BR-AU-64 | The remote-owner arm is fence-validated against Redis **before** a control stream is opened (origin-side hop) | `directory.validate(community, &fenced).await?` | `join.rs:365-372`; pinned `join.rs:2018-2033`, `:2056-2066` |
| BR-AU-65 | **The registry, not the `resolve_join` snapshot, gates reuse.** A `LocalOwner`+`acquired=None` result with no live registry entry is re-resolved in a bounded loop rather than admitting an owner peer with no loss watcher | `if owners.lost_for(session_id).is_some() { return }` else sleep+retry | `join.rs:426-441` |
| BR-AU-66 | That loop runs at most **25 attempts × 20 ms ≈ 500 ms** and then **fails closed** with a transient transport error, surfaced to the client as `join_rejected` | `OWNER_READY_MAX_ATTEMPTS`, `OWNER_READY_RETRY_INTERVAL` | `join.rs:387-388`, `:444-447`; three paths pinned `join.rs:2657-2745` |
| BR-AU-67 | Only the reuse arm may loop; CAS-win and remote-owner arms return immediately | `_ => return Ok(resolved)` | `join.rs:439` |
| BR-AU-68 | A `LocalOwner` reuse that finds `lost_for == None` after the loop is an **invariant violation**, logged at `error!` — not a benign race | `error!("huddle owner-ready invariant violated: …")` | `handler.rs:586-601` |
| BR-AU-69 | The owner-lease renewer is **per-room, not per-connection**: N local owner joiners must not each spawn a renewer | `HuddleOwnerRegistry` keyed by `session_id` | `join.rs:566-598`, `:665-724` |
| BR-AU-70 | If `attach_signals` finds a live entry, the existing renewer wins and the just-acquired extra lease is released by cancelling a throwaway renewer — **no lease leak, no double renewer** | pre-cancelled token handed to a discarded renewer | `join.rs:679-692`; pinned `join.rs:2470-2504` |
| BR-AU-71 | `release` and `drain` are **generation-fenced**: a stale room-empty cannot tear down a newer epoch installed by a re-acquire | `remove_if(|_, e| e.generation == generation)` | `join.rs:734-742`, `:750-761`; pinned `join.rs:2506-2552`, `:2626-2650` |
| BR-AU-72 | Exactly one `release` fires per room epoch, because only the last leaver sets `room_emptied` | `if room_emptied { mesh.owners.release(channel_id, generation) }` | `handler.rs:868-877` |
| BR-AU-73 | Renewal cadence is 10 s against a 30 s lease TTL → three attempts per lifetime. Missed ticks use `MissedTickBehavior::Delay` | `DEFAULT_HUDDLE_RENEW_INTERVAL`, mirrored from the reliable lane | `join.rs:452`, `:501-502`; `directory.rs:17`; `reliable.rs:34` |
| BR-AU-74 | Owner-loss is `Lost`, a renew error, a release `NotOwner`, **or** a release error on a non-caller-cancel exit. A caller-initiated cancel is normal teardown and stays silent | `if !caller_cancelled { lost_for_task.cancel() }` | `join.rs:504-560`; pinned `join.rs:2377-2422` |
| BR-AU-75 | Once `drain_all()` runs the registry is **permanently** draining: every later `attach_signals` immediately releases the new lease and returns a pre-cancelled `draining` token | sticky `AtomicBool` + early return | `join.rs:667-677`, `:765-773`; pinned `join.rs:2600-2624` |
| BR-AU-76 | `attach_signals` closes its own check→insert race with `drain_all` by re-checking after publication and retracting the exact epoch it installed | second `is_draining()` + generation-fenced `remove_if` | `join.rs:706-720` |
| BR-AU-77 | A new owner admission is refused up-front when the registry is draining (`huddle_relay_draining`) | `if mesh.owners.is_draining()` before `resolve_join_owner_ready` | `handler.rs:308-320` |
| BR-AU-78 | Owner-side: the community is **learned from the first `RegisterPeer`** and latched; a later frame naming a different community tears the stream down | `stream_community` latch | `join.rs:1200-1209` |
| BR-AU-79 | **Validate before admit.** The Redis fence runs on `RegisterPeer` before `room.add_peer`; a wrong community keys a lease that does not exist → `no_active_lease`, so no peer is admitted | `directory.validate(community, &fenced)` then `register_remote_peer` | `join.rs:1231-1245`; pinned `join.rs:2332-2392` |
| BR-AU-80 | A **non-fence** validate error (Redis unreachable, decode) is not a clean rejection — it tears the stream down | `None => break Err(e)` | `join.rs:1240-1244` |
| BR-AU-81 | `UnregisterPeer` needs no fence because it can only remove entries from **this stream's own** `registered` map | `registered.remove(&pubkey)` gate | `join.rs:1264-1288` |
| BR-AU-82 | Every control frame must carry the **same** fenced header the `Hello` did; a lease that moves mid-stream rejects subsequent frames | `if f != fenced { break Err(OwnerMismatch) }` | `join.rs:1191-1198` |
| BR-AU-83 | Control-stream teardown **always** drops every peer that stream registered, whatever the exit path, so no leaked remote peer holds an index | unconditional loop after the `result` binding | `join.rs:1345-1367`; pinned `join.rs:2253-2330` |
| BR-AU-84 | The owner is the **sole allocator** of `peer_index`, so indices cannot collide across pods | ingress calls `add_peer_at_index(owner_assigned)` | `handler.rs:507-509`; doc `mesh.rs:18-24` |
| BR-AU-85 | Remote registration happens **before** local ingress admission, so the owner-assigned index is the only index the client ever has — no frame or `joined` can escape with a placeholder | `dial_remote_owner` at `handler.rs:444-503` precedes `add_peer_at_index` at `:507` | `handler.rs:443-511` |
| BR-AU-86 | `add_peer_at_index` refuses an index already in use, removes it from the free list, and advances `next_fresh` past it so a later local allocation cannot collide | `peers.iter().any(...)`, `free.retain`, `next_fresh = idx+1` | `room.rs:288-306`; pinned `room.rs:559-575` |
| BR-AU-87 | Roster deltas are ordered and monotone; an ingress receiver that detects a gap requests `RosterResync` and **does not apply the gapped delta** | `next == revision.wrapping_add(1)` guard, else send `RosterResync` | `join.rs:1560-1597`; pinned `join.rs:2113-2183` |
| BR-AU-88 | A stale delta (`next <= revision`) is silently ignored | `Ok(RosterDelta{revision: next, ..}) if next <= revision => {}` | `join.rs:1583` |
| BR-AU-89 | Owner-side roster subscription happens **before** admission, and `PeerRegistered` carries a post-admission snapshot, so queued deltas at or below that revision are safely ignorable | `subscribe_roster()` at `:1229`, snapshot in the reply at `:1401` | `join.rs:1225-1263` |
| BR-AU-90 | A lagged owner-side `roster_rx` recovers by sending a full `RosterSnapshot`, never by skipping deltas | `RecvError::Lagged(_) => roster_snapshot_msg(&room)` | `join.rs:1174-1182` |
| BR-AU-91 | The media fence is **monotone-only**: accept `>= floor`, advance on `>`, reject `<`. A rejected frame does not move the floor | `GenerationFloor::check` | `mesh.rs:102-128`; pinned `mesh.rs:298-330` |
| BR-AU-92 | Dropping a stale-generation datagram is deliberately indistinguishable from packet loss for lossy audio | stated `mesh.rs:44-51`; verified — `RejectStale` returns before delivery | `mesh.rs:212-220` |
| BR-AU-93 | A media datagram whose channel UUID is **ambiguous** across two active communities is dropped (fail closed) rather than delivered to the wrong tenant | `get_unambiguous_by_channel` returns `None` on ≥2 matches | `room.rs:526-541`, `mesh.rs:221-227`; pinned `room.rs:684-728` |
| BR-AU-94 | An empty datagram payload is dropped with a warning | `payload.split_first()` → `None` | `mesh.rs:229-232` |
| BR-AU-95 | `RealtimeMedia` arriving as a **stream** (rather than a datagram) is a protocol violation and is rejected without routing | dispatcher match arm | `mesh_boot.rs:113-121`; pinned `mesh_boot.rs:611-616` |
| BR-AU-96 | Inbound mesh traffic arriving before its per-profile handler is registered is logged and **dropped**; fencing makes the peer's retry safe | `OnceLock::get()` → `None` | `mesh_boot.rs:52-55`, `:92-100`, `:122-129`; pinned `mesh_boot.rs:621-663` |
| BR-AU-97 | First handler registration wins; later registrations are logged and ignored | `OnceLock::set(...).is_err()` | `mesh_boot.rs:70-88`; pinned `mesh_boot.rs:645-647` |
| BR-AU-98 | On SIGTERM the mesh gossips `draining=true` **then** drains locally-owned huddles, each generation-fenced | drain watcher polls `shutting_down` every 500 ms | `mesh_boot.rs:481-496` |
| BR-AU-99 | `BUZZ_MESH` off is a hard no-op: no endpoint bind, no Redis write, no spawned task | early `return Ok(None)` | `mesh_boot.rs:418-421`; pinned `mesh_boot.rs:527-541` |
| BR-AU-100 | A **misconfigured but enabled** mesh is fatal at boot (bind failure or ready-registry publish failure), by explicit policy | `?` on `MeshEndpoint::bind` and `publish_ready` | `mesh_boot.rs:423-465`; rationale `:404-410` |

---

#### 9. Reliable-stream ordering & retransmission

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-101 | There is **no relay-level retransmission or ACK protocol.** Ordering and reliability come entirely from the underlying QUIC bi-stream; the relay only frames and validates | verified: no seq, no ack, no retry, no timer in `reliable.rs` |
| BR-AU-102 | First join acquires the lease and becomes owner; a join on the owner stays local; a join elsewhere opens a fenced bi-stream to the owner and sends the `Hello` first | `join` | `reliable.rs:78-126`; pinned `reliable.rs:759-841` |
| BR-AU-103 | A lease whose `profile != ReliableStream` is refused — profiles never share a session id | `ProfileMismatch` | `reliable.rs:99-105` |
| BR-AU-104 | Outbound payloads are chunked at **1 MiB** into ordered `Data` frames; QUIC preserves order across chunks | `bytes.chunks(MAX_RELIABLE_PAYLOAD_BYTES)` | `reliable.rs:274-294` |
| BR-AU-105 | Inbound acceptance validates `hello.sender == from`, `role == Session`, `profile == ReliableStream`, and `fenced.owner == local` — all structural, no Redis | `accept_inbound` | `reliable.rs:135-172` |
| BR-AU-106 | Every inbound `Data`/`Goodbye` is validated against **both** the stream's pinned fence and Redis. Unlike the media floor, a mismatch **fails the session** rather than dropping silently | `recv_validated` → `validate_frame_fence` | `reliable.rs:332-389`; rationale `:326-331` |
| BR-AU-107 | The community is latched from the first validated frame; any later frame naming a different community is `CommunityMismatch` | `validate_frame_fence` / `ensure_outbound_community` | `reliable.rs:370-388`, `:391-408` |
| BR-AU-108 | `MeshStreamFrame::{Goodbye, Hello, Gossip}` on a reliable session stream are all `UnexpectedFrame` errors — session close rides the **inner** `ReliableWireFrame::Goodbye`, not the outer mesh frame | three explicit arms | `reliable.rs:353-357` |
| BR-AU-109 | The inner frame format is validated strictly: `len >= 18`, `version == 1`, known kind, and `Goodbye` must be exactly 19 bytes with a known reason byte | `decode` | `reliable.rs:461-492`, `:509-518` |
| BR-AU-110 | Reliable-lease renewal follows the same loss contract as huddles (10 s / 30 s, caller-cancel silent) — but **no production site spawns it** | `spawn_lease_renewer_with_interval`; zero callers of `spawn_renewer`/`spawn_observable_renewer` | `reliable.rs:571-657`; stated `mesh_boot.rs:212-215` |
| BR-AU-111 | The demo echo consumer polls `shutting_down` every 100 ms and, on drain, sends `Goodbye(Draining)` if a community is latched, else bare `finish()` | `drain_tick` arm | `mesh_boot.rs:313-336` |

---

#### 10. Session-directory lease rules

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-112 | Acquire is a single Lua script: if a live lease exists, return it and **do not touch** the generation counter; otherwise `INCR` the counter and `SET` the lease with `PX ttl` | `ACQUIRE_SCRIPT` | `directory.rs:20-34`; pinned `directory.rs:650-711` |
| BR-AU-113 | The generation counter key **never expires**, so generation is monotone across lease expiry and pod death | no TTL on `…:generation` | `directory.rs:453-461`; pinned `directory.rs:712-777` |
| BR-AU-114 | Renew extends the TTL **only** if `(owner, generation)` match exactly; otherwise `Lost` | `RENEW_SCRIPT` string match | `directory.rs:36-52`, `:246-273` |
| BR-AU-115 | Release deletes **only** on an exact `(owner, generation)` match; otherwise `NotOwner` | `RELEASE_SCRIPT` | `directory.rs:54-70`, `:277-303` |
| BR-AU-116 | A malformed lease value (wrong part count, `generation == 0`, non-hex owner, unknown profile) is an error, never a silent default | `parse_lease` | `directory.rs:495-531`; pinned `directory.rs:639-648` |
| BR-AU-117 | Fence verdict order is fixed: `StaleGeneration` → `NoActiveLease` → `FutureGeneration` → `OwnerMismatch`, with `known = max(lease.generation, counter)` | `validate_fenced_header` | `directory.rs:348-439`; pinned `directory.rs:779-880` |
| BR-AU-118 | `StaleGeneration` is only raised when `known > 0`, so a never-used session does not reject generation 0 | `if known > 0 && …` | `directory.rs:382` |
| BR-AU-119 | Every fence rejection increments `mesh_fence_rejections_total{reason}` with the exact taxonomy string | `record_fence_rejection` | `directory.rs:481-483` |
| BR-AU-120 | A lease TTL that cannot be expressed as a positive `i64` ms is rejected before any Redis call | `ttl_ms` | `directory.rs:570-575` |
| BR-AU-121 | `takeover` is documented as a distinct operation but is a **verbatim delegate to `acquire`** — and has zero callers | `self.acquire(...)` | `directory.rs:233-242` |

---

#### 11. Conformance emit-coverage guard

| ID | Rule | Enforcement | Line |
|---|---|---|---|
| BR-AU-122 | Every entry to a critical seam must arm an `EmitGuard`; if the seam exits with zero emits, `Drop` records `TraceAction::ImplBug{kind}`, which the checker treats as a coverage breach | `Drop for EmitGuard` | `conformance/mod.rs:403-414`; contract `:26-30` |
| BR-AU-123 | Counting is done by a **wrapper tracer** returned from `arm`; production code calls `tracer.record` as before and must thread the wrapper downstream or the counter never moves | `CountingTracer` + `arm` returning a tuple | `conformance/mod.rs:357-372`, `:383-400` |
| BR-AU-124 | The synthetic `ImplBug` lands on the **inner** tracer, not the wrapper, so it cannot recursively bump the counter | `self.inner.record(step)` | `conformance/mod.rs:412` |
| BR-AU-125 | Exactly one seam is armed today: `ingest_event`, kind `"ingest_event_exited_without_trace"` | one production `EmitGuard::arm` call site | `ingest.rs:1382-1386` |
| BR-AU-126 | The ingest wrapper also maps `IngestError → SanitizedError` in one place, so individual `return Err(...)` sites need no emit. On `Ok` it emits nothing — the inner fn's dispatch points are responsible | `if let Err(err) = &result { emit(...) }` | `ingest.rs:1406-1418`; mapping `conformance/mod.rs:422-429` |
| BR-AU-127 | `SanitizedReason` is a closed 3-value alphabet asserted 1:1 with `IngestError`; a fourth variant makes the match non-exhaustive and CI catches it | exhaustive `match` with no `_` arm | `conformance/mod.rs:422-429` |
| BR-AU-128 | **Don't normalize away violations**: `claimed_community` (from the event `h` tag) is recorded separately from `resolved_community` (from `TenantContext`), because the checker's claim≠resolved bite depends on seeing both | `claimed_community_from_event` + `state_for_request` | `conformance/mod.rs:16-21`, `:94-117` |
| BR-AU-129 | On the REQ read path `claimed_community` is **unconditionally `None`** — the REQ wire carries no client-asserted community. Encoding `None` (not a copy of resolved) is load-bearing | hard-coded `None` | `conformance/mod.rs:128-146`, `:152` |
| BR-AU-130 | A channel-scoped row whose channel id is missing from the lookup map is a **coverage breach**, emitted as `ImplBug{"row_community_lookup_missing"}`, never silently substituted with the resolved label | `RowCommunityProjection::MissingLookup` → `ImplBug` | `conformance/mod.rs:234-255`, `:286-294`, `:321-329`; pinned `:660-696` |
| BR-AU-131 | A channel-**less** row legitimately projects as the resolved community; the distinction comes from the row's own `channel_id`, not from the query filter, so a channel-scoped row cannot masquerade as channel-less | `match row_channel_id { None => resolved, Some(ch) => lookup }` | `conformance/mod.rs:186-197`; rationale `:167-184` |
| BR-AU-132 | Production binds `NoopTracer`, so **all of the above is inert in production**: every emit and every `ImplBug` is discarded | `tracer: Arc::new(crate::conformance::NoopTracer)` | `state.rs:794-798`; `tracers.rs:20-24` |
| BR-AU-133 | A `JsonlTracer` write failure loses one step but must never fail the request; the coverage guard is the safety net for systemic loss | all-`let _ =` writes | `tracers.rs:63-71` |
| BR-AU-134 | This audio module emits **no trace steps at all** — no `EmitGuard`, no `Tracer` reference anywhere in `audio/`, `tunnel/`, or `mesh_boot.rs`. Huddle joins, cross-pod registrations, and the three lifecycle-event writes are entirely outside the conformance seam | verified by grep for `conformance`/`tracer` across the group |
