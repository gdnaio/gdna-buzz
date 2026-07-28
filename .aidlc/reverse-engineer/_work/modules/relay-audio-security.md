## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Authentication on the audio WebSocket

##### 1.1 The five gates, in order

| # | Gate | Mechanism | Line | Effect if it fails |
|---|---|---|---|---|
| 1 | Tenant binding | `tenant::bind_community(&state.db, Host)` — **before** the WS upgrade | `handler.rs:74-88` | 404 with a generic body; deliberately non-enumerable |
| 2 | Connection budget | shared `conn_semaphore` | `handler.rs:90-99` | 503 |
| 3 | Community liveness | `db.is_community_active` | `handler.rs:156-164` | silent close |
| 4 | NIP-42 | `state.auth.verify_auth_event(event, challenge, relay_url)` | `handler.rs:220-238` | `{"type":"error","message":"auth failed"}` + close |
| 5 | Relay membership | `api::relay_members::enforce_relay_membership` | `handler.rs:244-262` | `restricted: not a relay member` |
| 6 | Channel membership | `ensure_membership` | `handler.rs:265-286` | `not a member` |

##### 1.2 Huddle auth does reuse NIP-42, faithfully

The audio route runs the **same verifier object** as the main relay door
(`state.auth`, `handler.rs:222`), with:

- a fresh per-connection challenge from `buzz_auth::generate_challenge()`
  (`handler.rs:176`) — not reusable, not derived from anything client-supplied;
- a **per-tenant** expected relay URL via
  `api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)`
  (`handler.rs:219`), so an auth event minted for community A cannot be replayed
  against community B on a multi-tenant deployment;
- NIP-OA tag extraction *before* the event is consumed (`handler.rs:217`), so the
  delegated-owner path works identically to HTTP;
- a hard 5 s window (`handler.rs:61`, `:190`) after which the socket closes.

The identity used for everything downstream is `auth_ctx.pubkey`
(`handler.rs:240-242`) — never `auth_msg.event.pubkey` directly, and never a
client-supplied identifier.

##### 1.3 What the pre-auth surface exposes

Before authenticating, an anonymous caller reaches:

- The tenant lookup (a DB query per attempt) and the resulting 404/upgrade
  distinction. Because the 404 body is identical for "no such community" and
  "DB error" (`handler.rs:80-88`), community enumeration by host is not possible.
- A WebSocket upgrade + one `challenge` frame, holding a `conn_semaphore` permit and
  a spawned connection task for up to 5 s. **This is the primary unauthenticated
  resource cost** — see §6.1.
- Nothing about whether `{channel_id}` exists: the channel is not looked up until
  after auth (`handler.rs:1164`), so an unauthenticated caller cannot probe channel
  existence.

---

#### 2. Authorization / channel-membership enforcement

##### 2.1 Enforcement chain (`ensure_membership`, `handler.rs:1153-1235`)

1. `db.get_channel(tenant.community(), channel_id)` — community-scoped, so a
   cross-tenant channel UUID simply does not resolve (`handler.rs:1162-1167`).
2. Archived channels are rejected **first**, before any membership logic
   (`handler.rs:1168-1170`) — an auto-ended huddle cannot be rejoined by a member.
3. For a TTL channel, the lifecycle parent is verified against a **creator-signed
   kind-48100 link in the DB**, not the client's `parent_channel_id`
   (`handler.rs:1172-1191`). This is the notable hardening: the comment at
   `handler.rs:1172-1175` says explicitly "instead of trusting the UUID supplied by
   the client during audio auth". Without it, a member of any channel could inject
   48101/48102/48103 events into an arbitrary parent channel by claiming it.
4. Member fast path (`is_member_cached`, `handler.rs:1196-1203`).
5. `visibility == "open"` bypass (`handler.rs:1205-1207`).
6. Auto-add, restricted to TTL channels whose caller is a member of the **resolved**
   parent (`handler.rs:1208-1231`).
7. Otherwise `"not a member"`.

##### 2.2 What a non-member can reach

| Actor | Reach |
|---|---|
| Unauthenticated | challenge frame only; §1.3 |
| Authenticated, non-relay-member, `require_relay_membership=false` (**default**) | passes gate 5 unconditionally — `api/mod.rs:67` returns `OpenRelay`, `api/mod.rs:130-131` maps it to `Ok`. So on a default deployment gate 5 is decorative |
| Authenticated, non-channel-member, channel `visibility != "open"`, non-TTL | refused at gate 6 |
| Authenticated, channel `visibility == "open"` | **admitted with no membership row at all** (`handler.rs:1205-1207`). Any authenticated pubkey can join and both hear and speak in any open channel's huddle. This mirrors the relay's read model for open channels, but audio is a live bidirectional capability, not a read |
| Authenticated member of a parent channel, huddle is TTL+private | **auto-added as a member** (`handler.rs:1219-1227`) — a write to `channel_members` triggered by a WebSocket connect, attributed to `channel.created_by` rather than to the joiner |

##### 2.3 Residual authorization gaps

- **No per-channel huddle permission.** Channel membership is the only
  authorization; there is no "may join voice" role, no admin-only huddle, and no
  moderator capability. Any admitted peer can transmit to every other peer.
- **No revocation mid-call.** Membership is checked once at join. Removing a member
  from the channel does not disconnect them; there is no membership-change watcher
  on the audio path (contrast `fan_out_event_to_local_subscribers`, which re-gates
  per event — `handler.rs:1318`).
- **The auto-add write is a side effect of an unauthenticated-at-HTTP-level
  request.** It happens after NIP-42, so the actor is authenticated, but the
  resulting row records `added_by = channel.created_by` (`handler.rs:1224`) —
  attributing the membership to someone who did not perform the action. Audit
  attribution is therefore wrong for auto-added huddle members.

---

#### 3. Mesh trust model

##### 3.1 Who can become a mesh peer

Inbound QUIC connections are accepted only from a runtime already present in
membership, and membership is populated from the Redis **ready registry**
(`buzz-relay-mesh/src/runtime.rs:275-283`, `:309-320`). Records are attested
against the deployment's relay signing key
(`MeshMembership::with_expected_relay_pubkey(relay_keypair.public_key().to_hex())`,
`mesh_boot.rs:441-443`) — the comment cites the review rationale: "all pods share the
relay signing key, so a seed attested by any other key is foreign and rejected
(possession is not authorization)".

**Trust boundary:** anyone who can (a) write to the shared Redis and (b) sign with
the relay keypair can join the mesh as a peer. Both are pod-level secrets, so the
mesh trust domain is exactly "processes holding the relay identity". There is no
per-pod identity, no mTLS beyond iroh's own key exchange, and no authorization
distinction between peers — every peer is fully trusted once admitted.

##### 3.2 Who can claim ownership of a room

Ownership is **not** claimable over the mesh. It is granted only by the Redis
fenced CAS:

- `ACQUIRE_SCRIPT` (`directory.rs:20-34`) refuses when a live lease exists, and
  INCRs the non-expiring generation counter only when it creates one.
- `resolve_join` never treats a mesh hint as authority (`join.rs:317-379`); the
  owner side re-validates every registration's fence against Redis
  (`join.rs:1231-1245`).
- `accept_inbound` rejects a stream whose `fenced.owner_runtime_id` is not this pod
  (`join.rs:1079-1086`), so a peer cannot ask pod X to act as owner for a lease
  pod Y holds.
- `HuddleControlAcceptor` never calls `acquire` — it only ever `validate`s
  (verified: the acceptor's only directory call is `join.rs:1231`).

So a mesh peer can only ever *route* to the true owner. Ownership forgery requires
Redis write access, which is the same trust domain as §3.1.

##### 3.3 Can a peer forge a forwarded frame? Yes, within the mesh trust domain

This is the sharpest finding in this aspect.

**Control frames are Redis-fenced. Media datagrams are not.**

`MeshAudioRouter::on_media_datagram` (`mesh.rs:204-250`) performs exactly two checks
before delivering audio to local WS clients:

1. `GenerationFloor::check(session_id, generation)` — a **purely local, in-memory
   monotone counter** (`mesh.rs:102-128`). It accepts any generation `>= floor`, and
   a *higher* generation simply advances the floor (`mesh.rs:113-119`). It never
   consults Redis.
2. `rooms.get_unambiguous_by_channel(session_id)` — a **community-free** lookup
   (`room.rs:526-541`).

Consequences for a mesh peer (or anything that can reach the pod's QUIC endpoint and
pass §3.1 admission):

- It can inject audio into **any** huddle room on that pod by sending a datagram
  whose `fenced.session_id` is the target channel UUID and whose `generation` is
  large. `fenced.owner_runtime_id` is **never checked** on the datagram path — it is
  documented as "advisory for routing/diagnostics"
  (`buzz-relay-mesh/src/wire.rs:90-92`), and `mesh.rs` honours that literally.
- The injected `payload[0]` is an arbitrary `peer_index`, so the frame is attributed
  to whichever participant occupies that index — a **spoofing primitive**: the
  injected audio reaches every local peer except the impersonated one
  (`room.rs:422-429`), so the impersonated speaker cannot hear their own forgery.
- Because the check is monotone-advancing, a single high-generation forged datagram
  **poisons the floor** and causes the legitimate owner's subsequent frames to be
  rejected as stale — a targeted per-session audio DoS
  (`mesh.rs:106-112` returns `RejectStale` for anything below the poisoned floor).
- The room lookup is community-free, so the **tenant boundary does not hold on the
  media path**. It fails closed only when two *active* communities happen to share
  the channel UUID (`room.rs:534-539`). With a single active community per UUID —
  the normal case — a peer addresses that community's room without naming it.
  `mesh.rs:36-42` and `room.rs:519-525` both acknowledge that "the current mesh media
  envelope does not carry a community identifier".

The design intent is stated at `mesh.rs:44-51`: "Every datagram carries a
`FencedHeader`. Both ends reject frames whose generation is stale". The code
implements the *stale* half only; there is no owner check and no Redis check. Within
the stated trust model (all mesh peers are relay-key holders) this is defensible;
it is not defensible against a single compromised pod, which today can eavesdrop on
nothing but can inject into and silence any huddle on any peer pod.

##### 3.4 Cross-pod control-path forgery — well defended

By contrast the `HuddleControl` path is tight:

| Attack | Defence |
|---|---|
| Claim to be another pod | `hello.sender != from` rejected (`join.rs:1060-1065`) |
| Route a stream to a non-owner | `fenced.owner_runtime_id != local` rejected (`join.rs:1079-1086`) |
| Register into another tenant's room | fence keyed `(community, session)`; a wrong community keys a lease the owner never wrote → `no_active_lease` **before** `add_peer` (`join.rs:1231-1245`); pinned `join.rs:2332-2392` |
| Switch community mid-stream | latched and rejected (`join.rs:1200-1209`) |
| Change fence mid-stream | `f != fenced` → `OwnerMismatch` (`join.rs:1191-1198`) |
| Mint a `CommunityId` from wire input | impossible by type: `CommunityId` is non-`Deserialize`, so the wire field is a raw `Uuid` re-minted explicitly at `join.rs:1211` (rationale `join.rs:851-865`) |
| Send owner→non-owner replies to the owner | protocol violation, stream torn down (`join.rs:1302-1310`) |
| Leak peers by abandoning a stream | teardown always drops every registered peer (`join.rs:1345-1367`); pinned `join.rs:2253-2330` |

---

#### 4. Session-directory lease forgery

| Attack | Feasibility |
|---|---|
| Present a fabricated `FencedHeader` on a control frame | Fails — `validate_fenced_header` compares against the live Redis lease on generation **and** owner (`directory.rs:408-437`) |
| Replay an expired lease's fence | Fails — the generation counter never expires, so `known > 0` and `frame.generation < known` yields `StaleGeneration`; after expiry-before-takeover it yields `NoActiveLease` (`directory.rs:382-406`); pinned `directory.rs:882-919` |
| Claim a future generation to look "newer" | Fails on the control path — `FutureGeneration` when it does not match the live lease exactly (`directory.rs:408-422`). **Succeeds on the media datagram path** (§3.3), where a higher generation is *accepted and pinned* |
| Forge the lease value in Redis | Requires Redis write access — the same trust domain as the mesh. Nothing signs or MACs the lease value; it is a plain `owner_hex\|gen\|profile` string (`directory.rs:31`) |
| Corrupt the lease to a malformed value | Detected: wrong part count, `generation == 0`, non-hex owner, or unknown profile all error (`directory.rs:495-531`) rather than defaulting |
| Cross-tenant lease collision | Prevented by the key shape `buzz:{community}:tunnel:{session}` (`directory.rs:453-461`); pinned `directory.rs:625-637` |
| Cross-**profile** session hijack | Partially prevented: `reliable.rs:99-105` refuses a lease whose profile is not `ReliableStream`. The **huddle path does not check the profile** — `HuddleDirectory::owner_of` (`join.rs:107-127`) discards `lease.profile` entirely, so a `reliable-stream` lease on the same session id would be honoured as a huddle owner lease. Since huddle session ids are channel UUIDs and reliable session ids are caller-chosen, this becomes reachable through the demo route (§5) |

##### 4.1 Lease/fence blind window

Between lease expiry (30 s TTL) and the owner's next renew tick (10 s), the pod
still believes it owns the room. Local WS peers have **no per-frame fence** — the
comment at `handler.rs:568-575` says exactly this: "local WS peers have no per-frame
fence". The `resolve_join_owner_ready` loop exists to prevent an *ownerless* owner
peer at admission; it does not shorten the post-admission blind window. A network
partition longer than 30 s can therefore produce two pods each fanning out locally
for up to ~10 s, with cross-pod media fenced but same-pod media not.

---

#### 5. `POST /_mesh/demo/echo` — the tenant-boundary bypass

`api/mesh_demo.rs:59-79`:

- **No authentication of any kind.** No NIP-42, no NIP-98, no secret, no
  `enforce_relay_membership`. The only gates are `state.mesh().is_some()` and
  `config.mesh_demo_echo` — both 404 when off (`api/mesh_demo.rs:66-72`).
- **`community_id` is taken from the request body** (`api/mesh_demo.rs:50-52`,
  minted at `:92` via `CommunityId::from_uuid(req.community_id)`), not from the
  `Host` header. Every other route in the relay derives the tenant from the host;
  this one is the exception.
- `session_id` is also caller-chosen (`api/mesh_demo.rs:53-54`).

From the tunnel side, what that grants when both flags are on:

1. **Arbitrary lease creation in any community's key space.** `router.join` →
   `directory.acquire(community_from_body, session_from_body, …)` writes
   `buzz:{arbitrary}:tunnel:{arbitrary}:lease` and INCRs the matching non-expiring
   generation counter (`reliable.rs:87-95` → `directory.rs:200-225`).
2. **Denial of huddle ownership.** A huddle's `session_id` **is** its `channel_id`
   (`handler.rs:324`). An unauthenticated POST with
   `{community_id: <victim>, session_id: <channel uuid>}` takes the lease for that
   channel under `profile = reliable-stream`. A subsequent huddle join resolves
   `RemoteOwner` pointing at the attacker's chosen pod (or `LocalOwner` on a stale
   snapshot) and — because the huddle path never checks `lease.profile`
   (§4, last row) — proceeds against a lease minted by a non-huddle caller. The
   lease is deliberately never renewed (`api/mesh_demo.rs:11-15`), so it self-heals
   after 30 s, but it can be re-posted.
3. **Unbounded monotone generation inflation.** Repeated post/expire cycles INCR the
   never-expiring counter without limit, and `validate_fenced_header` computes
   `known = max(lease.generation, counter)` (`directory.rs:375-380`) — so a legitimate
   owner at a lower generation is rejected as `StaleGeneration`.
4. **Redis key-space growth** in arbitrary community namespaces, with no TTL on the
   generation keys.

Mitigating facts: both `BUZZ_MESH` and `BUZZ_MESH_DEMO_ECHO` default **off**
(`config.rs:498-500`, `:516-518`) and require an explicit `on`/`true`/`1`; the route
404s otherwise, indistinguishably from a build without it
(`api/mesh_demo.rs:67-72`). The module doc labels it "testbed-only" and "Not a
product flow" and says "the route stays demo-gated until it is deleted"
(`api/mesh_demo.rs:1-22`). It is nonetheless registered on the **public** router
(`router.rs:123`), not behind the admin host.

Also unauthenticated: `GET /_mesh` (`router.rs:230`, handler `:399-406`) on the
health router, which `router.rs:222-224` documents as having "no auth". It returns
the serialized `MeshStatus` — peer runtime ids, addresses, and drain state — to
anyone who can reach the health port.

---

#### 6. DoS surfaces

##### 6.1 Unauthenticated

| Surface | Bound | Gap |
|---|---|---|
| WS upgrade + 5 s auth window | one `conn_semaphore` permit + one spawned task + 1 DB query (`bind_community`) per attempt | The permit is held for the **whole** 5 s window (`handler.rs:90-99`, `:190`). Because the semaphore is **shared with ordinary relay WebSockets** (`handler.rs:105-107`, verified `state.rs:727`), an attacker can exhaust the relay's entire WebSocket budget through the audio route alone, at ~`max_connections/5s` request rate. There is no per-IP rate limit and no separate audio budget |
| Pre-auth text frames | 8192 bytes/message at the parser; unlimited *count* inside the 5 s window | The auth loop `continue`s on oversize and on non-`auth` JSON (`handler.rs:194-204`), so a client may send unlimited 8 KB frames for 5 s. Each one is `serde_json::from_str`-parsed into a full `nostr::Event` attempt |
| `POST /_mesh/demo/echo` (when enabled) | 1 MB body limit (`router.rs:130`) | §5 — unbounded Redis writes, no auth, no rate limit |
| `GET /_mesh` | none | information disclosure, and an unbounded-rate endpoint on the health port |

##### 6.2 Authenticated

| Surface | Bound | Gap |
|---|---|---|
| Peers per room | 25 soft (`room.rs:50`), 255 hard | Enforced inside the admission lock (`room.rs:233-238`) — sound |
| Rooms per pod | **none** | `AudioRoomManager.rooms` is an unbounded `DashMap` (`room.rs:490`). Any authenticated member can create a room per channel; `get_or_create` has no cap (`room.rs:503-509`). Bounded only indirectly by `conn_semaphore` and by channel count |
| Frames per second per peer | **none** | Only a 4096-byte size cap (`handler.rs:961-964`). No token bucket, no bitrate cap. A single peer can flood at line rate; each frame costs one allocation plus N−1 refcount clones (`room.rs:398-410`) |
| Fan-out amplification | 1 → N−1 within a pod | With 25 peers, one 4 KB frame produces 24 outbound 4 KB frames = **24×** amplification, capped only by the 8-slot drop-on-full queues. Cross-pod, the owner additionally re-emits to every remote peer's mesh sink (`mesh.rs:262-283`), so a non-owner pod's single inbound frame becomes 24 datagrams from the owner — the same 24× ratio, now on the network rather than in-process |
| Tasks per connection | 4–6 (`handler.rs:663`, `:667`, `:670`, `:704`, `:733`) | Multiplied by the shared connection budget |
| `GenerationFloor.seen` | **unbounded** `DashMap` (`mesh.rs:90`) | One entry per session ever observed. Removed only by explicit `forget` (`handler.rs:755`, `:763`, `:909`); no TTL sweep. A peer can create entries at will by sending datagrams for fresh random session ids (§3.3) — each one a permanent 24-byte-plus-overhead map entry. **Unauthenticated-within-the-mesh unbounded memory growth** |
| `HuddleOwnerRegistry.entries` | one per owned room | Removed by `release`/`drain` (`join.rs:734-761`). `release` only fires when the room empties (`handler.rs:868-877`); a room whose archive fails keeps its entry and its renewer |
| Inbound mesh streams | **unbounded** `tokio::spawn` per stream (`mesh_boot.rs:270-274`, `:290-298`) | No concurrency limit, no per-peer cap |
| Reliable-stream payload | 1 MiB per frame (`reliable.rs:31`), 16 MiB wire cap | No total-bytes, rate, or session-count bound; each `Data` frame allocates a fresh `Vec<u8>` (`reliable.rs:471-474`) |
| Redis calls per join | 1–3 (`owner_of`, possibly `acquire`, possibly `validate`) plus up to **25 retries** in the owner-ready loop (`join.rs:423-441`) | A contended room can drive 25 `owner_of` round trips in 500 ms per joining connection |

##### 6.3 Not a DoS surface (verified)

- Audio queues never grow: every audio send is `try_send` with drop-on-full
  (`room.rs:409`, `:427`, `handler.rs:1115`).
- `broadcast` roster channel is bounded at 64 with `Lagged` recovery, so a slow
  control-stream consumer cannot grow memory (`room.rs:179`, `join.rs:1174-1182`).
- The WS parser rejects oversize messages before the handler allocates
  (`handler.rs:116-119`), pinned by test (`handler.rs:1417-1427`).

---

#### 7. Data handling / confidentiality

| Concern | Finding |
|---|---|
| Audio content | Never decoded, never logged, never persisted. `broadcast_frame` copies bytes (`room.rs:398-410`); `forward_media` passes them through (`join.rs:1758-1766`). The relay has no plaintext of the Opus body beyond the transport |
| End-to-end encryption | **Not present for audio.** `buzz-relay-mesh/src/wire.rs:118-121` claims "the client frame's encrypted content is NIP-44 between client endpoints, so server-side plaintext of the media itself never exists". Nothing in this group or in `desktop/src-tauri/src/huddle/` encrypts the Opus payload — `relay_api.rs:303` builds a plain v2 frame. The relay does hold the audio in the clear (TLS-terminated), and so does any mesh peer it forwards to. The comment overstates the guarantee |
| Client telemetry | `level_dbov`, `seq`, `ts_48k` are logged at `trace` only (`handler.rs:996-1003`), and `wire.rs:16-20` forbids them from feeding trust decisions. Verified: no other consumer |
| Pubkeys in logs | Full hex, at `warn`/`info` (`handler.rs:255`, `:283`, `:419`, `:602`, `:880`). Consistent with the rest of the relay |
| Lifecycle events | Relay-signed (`handler.rs:1268`) with `p` = participant pubkey and content `{"ephemeral_channel_id": …}` (`handler.rs:1238-1256`). So huddle attendance is a **permanent, publicly queryable record** in the parent channel for anyone who can read it — the intended design, worth noting as a privacy property |
| Conformance trace | Projects community UUID, **full host string**, 16-hex-char pubkey prefix, 16-hex-char event-id prefix, raw channel UUID (`conformance/mod.rs:55-91`). No payloads, no signatures, no timestamps. In production nothing is written (`NoopTracer`, `state.rs:798`); `JsonlTracer` would write plaintext JSONL to a local file with no redaction and no permissions handling (`tracers.rs:37-46` uses default `OpenOptions`) |
| Redis lease values | Contain a mesh runtime id and a profile string — no secrets (`directory.rs:31`) |
| Information leak defence | Deliberate and documented: `Ended > Full > VersionMismatch` precedence exists so a rejected joiner cannot learn the room's pinned protocol version (`room.rs:216-227`, pinned `room.rs:759-782`) |

---

#### 8. Resource limits — consolidated table

| Limit | Value | Configurable? | Line |
|---|---|---|---|
| WS message / frame size | 8192 B | no | `handler.rs:52` |
| Binary audio frame | 4096 B | no | `handler.rs:44` |
| Text control frame | 8192 B | no | `handler.rs:47` |
| Auth timeout | 5 s | no | `handler.rs:61` |
| Heartbeat interval | 30 s | no | `handler.rs:55` |
| Missed pongs before close | 3 | no | `handler.rs:58` |
| Peers per room (soft) | 25 | no | `room.rs:50` |
| Peer index space | 0..=254 | no | `room.rs:146-152` |
| Per-peer audio queue | 8 | no | `room.rs:40` |
| Per-peer control queue | 32 | no | `room.rs:45` |
| WS data queue | 16 | no | `handler.rs:659` |
| WS control queue | 8 | no | `handler.rs:660` |
| Roster broadcast | 64 | no | `room.rs:179` |
| Protocol version ceiling | 2 | no | `handler.rs:124` |
| Owner-ready retries | 25 × 20 ms | no | `join.rs:387-388` |
| Huddle renew interval | 10 s | no | `join.rs:452` |
| Reliable renew interval | 10 s | no | `reliable.rs:34` |
| Lease TTL | 30 s | not via env (`with_lease_ttl` exists but production uses `new`) | `directory.rs:17`, `mesh_boot.rs:512` |
| Reliable payload chunk | 1 MiB | no | `reliable.rs:31` |
| Demo echo timeout | 10 s | no | `api/mesh_demo.rs:45` |
| Connection budget | `max_connections` | **yes** — shared with all relay WS | `state.rs:727` |
| Rooms per pod | unbounded | — | `room.rs:490` |
| `GenerationFloor` entries | unbounded | — | `mesh.rs:90` |
| Inbound mesh stream concurrency | unbounded | — | `mesh_boot.rs:270`, `:290` |
| Frames/sec per peer | unbounded | — | — |

Nineteen of the twenty-three bounded limits are compile-time constants. See
configuration for the tunability assessment.

---

#### 9. Ranked security findings

| # | Severity | Finding | Line |
|---|---|---|---|
| 1 | **High** | Media datagrams are not Redis-fenced and the `owner_runtime_id` is never checked; a mesh peer can inject audio into any room on any peer pod, spoof any `peer_index`, and poison the generation floor to silence the legitimate owner | `mesh.rs:204-250` |
| 2 | **High** | The media datagram envelope carries no community, and the room lookup is community-free, so the host-derived tenant boundary does not hold on the media path (fails closed only on an active UUID collision) | `room.rs:519-541`, `mesh.rs:221-227` |
| 3 | **High** (when both flags on) | `POST /_mesh/demo/echo` is unauthenticated **and** takes `community_id` from the body, allowing arbitrary lease creation and generation inflation in any community's Redis key space — including on a channel UUID that is a live huddle's `session_id` | `api/mesh_demo.rs:50-95`; `router.rs:123` |
| 4 | Medium | The huddle path ignores `lease.profile` (`HuddleDirectory::owner_of` discards it), so a `reliable-stream` lease is honoured as huddle ownership — the mechanism that makes finding 3 reach huddles | `join.rs:107-127` vs. `reliable.rs:99-105` |
| 5 | Medium | Audio connections share the global `conn_semaphore` and hold a permit for the whole 5 s pre-auth window, so an unauthenticated client can exhaust the relay's entire WebSocket budget via `/huddle/…/audio` | `handler.rs:90-99`, `:190` |
| 6 | Medium | `GenerationFloor.seen` is an unbounded `DashMap` with no TTL, growable at will from within the mesh | `mesh.rs:90`, `:131-133` |
| 7 | Medium | No frame-rate or bitrate limit; 24× in-pod and 24× cross-pod fan-out amplification from one authenticated peer | `room.rs:398-411`, `mesh.rs:262-283` |
| 8 | Medium | `require_relay_membership` defaults **false**, making the relay-membership gate on the audio route a no-op on default deployments | `config.rs:481-483`; `api/mod.rs:67`, `:130-131` |
| 9 | Medium | Membership is checked once at join with no revocation path; removing a member does not disconnect their audio | `handler.rs:1196-1234` |
| 10 | Low-Medium | `visibility == "open"` grants full bidirectional audio to any authenticated pubkey with no membership row | `handler.rs:1205-1207` |
| 11 | Low-Medium | Post-admission lease blind window: local WS peers have no per-frame fence, so a >30 s partition can produce two pods fanning out locally for up to ~10 s | `handler.rs:568-575`; `join.rs:452` |
| 12 | Low | Ephemeral auto-add writes a membership row attributed to `channel.created_by`, not to the joiner — wrong audit attribution | `handler.rs:1219-1227` |
| 13 | Low | `GET /_mesh` is unauthenticated and returns peer runtime ids, addresses, and drain state | `router.rs:230`, `:399-406` |
| 14 | Low | `buzz-relay-mesh/src/wire.rs:118-121` claims NIP-44 client-to-client encryption of media payloads; no such encryption exists — the relay and every mesh peer see plaintext Opus | verified across `audio/` and `desktop/src-tauri/src/huddle/` |
| 15 | Low | `mesh-llm` dev-dependencies are pinned by mutable git **tag**, not commit SHA, in the CI dependency graph | `Cargo.toml:87-88` |
| 16 | Informational | The conformance coverage guard is armed in production but writes into `NoopTracer`, so an `ImplBug` breach is silently discarded; `JsonlTracer` has zero callers | `state.rs:798`; `tracers.rs:20-24` |
