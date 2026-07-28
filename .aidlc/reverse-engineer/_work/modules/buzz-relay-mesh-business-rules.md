## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Business Rules

Numbered BR-MESH-nn. Every rule cites its enforcement site. Where a rule is
*documented but not implemented*, that is stated explicitly.

---

#### A. The governing law

**BR-MESH-01 — Membership is a hint; the Redis fenced generation is the arbiter.**
Stated at `lib.rs:33-35` and `wire.rs:19-27`. Structurally enforced by omission:
nothing in this crate reads, compares, or mutates `FencedHeader.generation`.
Verified — the four fence-rejection variants (`MeshError::StaleGeneration`,
`NoActiveLease`, `OwnerMismatch`, `FutureGeneration`, `lib.rs:96-130`) have **zero
constructors inside `crates/buzz-relay-mesh`**; all construction sites are in
`buzz-relay` (`tunnel/directory.rs:378,395,413,430,824,842,870,914`;
`audio/join.rs:1081,1201,1888,2854`). The crate defines the taxonomy and never
raises it.

**BR-MESH-02 — The mesh may say "don't dial"; it may never say "take over."**
`wire.rs:25-27`. Enforced by the seam split: `RelayMeshMembership` (`lib.rs:144-151`)
exposes only `peers()`, `local_runtime_id()`, `begin_drain()` — no ownership verb.
`RelayPeerTransport` (`lib.rs:158-175`) is explicitly documented as *not* performing
generation fencing (`lib.rs:161-163`). The consumer even keeps a compile-time
reminder: `fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
(`tunnel/reliable.rs:949`).

**BR-MESH-03 — Fencing is checked at every hop, by every consumer, not by the pipe.**
`wire.rs:21-24`, `lib.rs:161-163`. The crate provides the header slot; each
consumer validates. In practice the validators are
`SessionDirectory::validate_fenced_header` (`tunnel/directory.rs`) and the huddle
`GenerationFloor` (`audio/mesh.rs`).

---

#### B. Membership join

**BR-MESH-04 — The Redis ready registry is the only way *into* the mesh.**
`registry.rs:3-6`. A fresh runtime learns dialable peers exclusively from
`ReadyRegistry::scan_ready` (`registry.rs:211-257`), invoked at
`runtime.rs:311` (reconcile) and `runtime.rs:318` (inbound-admission rescan).
Gossip then propagates records peer-to-peer (BR-MESH-19).

**BR-MESH-05 — Ready records are only published while the relay is ready.**
`ReadyRegistry::publish_ready` deliberately has no internal readiness probe
(`registry.rs:177-181`); the caller owns the predicate. The relay's predicate is
`!shutting_down` (`mesh_boot.rs:469`), evaluated every `refresh_interval`
(`runtime.rs:602`).

**BR-MESH-06 — The first ready-registry publish is part of boot; failure is fatal.**
`mesh_boot.rs:456-463` — `.map_err(|e| anyhow!("mesh ready-registry publish failed: {e}"))?`
propagated through `main.rs:442`'s `?`. Rationale documented at
`mesh_boot.rs:456-457` and `mesh_boot.rs:406-411`: "an operator who sets
`BUZZ_MESH=on` wants the mesh or wants to know why not."

**BR-MESH-07 — A record cannot be published unless its own attestation verifies.**
`publish_ready` calls `record.verify_attestation()?` before touching Redis
(`registry.rs:183`). Self-check, so it only catches local construction bugs.

**BR-MESH-08 — Ready-record TTL is exactly 3× the refresh interval, floored at 1 s.**
`REGISTRY_EXPIRY_MULTIPLIER = 3` (`registry.rs:21`), `expiry_for`
(`registry.rs:154-156`), `ttl_secs = self.expiry().as_secs().max(1)`
(`registry.rs:187`). With the relay's hardcoded 15 s refresh (`config.rs:511`) the
TTL is 45 s. A crashed pod therefore disappears from the registry within 45 s
(`registry.rs:198` — "A crash is handled by TTL expiry"). Test:
`expiry_is_three_refreshes` (`registry.rs:322-325`).

**BR-MESH-09 — A scanned record whose Redis key does not match its own
`runtime_id` is discarded.** `scan_ready` requires `record.key() == key`
(`registry.rs:233`); mismatch logs and skips (`registry.rs:240-244`). Blocks
key-substitution.

**BR-MESH-10 — One malformed registry entry must not block bootstrap from healthy
peers.** `scan_ready` warns-and-skips on decode failure, key mismatch, and
attestation failure (`registry.rs:233-247`), documented at `registry.rs:206-208`.
Only Redis transport errors abort the scan (`registry.rs:225`, `?`).

---

#### C. Admission and identity

**BR-MESH-11 — Ready records must be attested by *this deployment's* relay key,
checked before signature verification.** `apply_ready_records`
(`membership.rs:85-118`): first `ready.relay_pubkey == expected` (`:90-92`), then
`ready.verify_attestation()` (`:103`). Ordering is deliberate and documented at
`membership.rs:80-84`: "a record signed by a key we don't recognize is foreign no
matter how valid its signature is." Rejections bump `foreign_relay_rejections`
(`:93-94`) and log with `anchored` (`:99`).
Tests: `ready_records_from_foreign_relay_identity_are_rejected` (`:451-462`),
`ready_records_must_have_valid_attestation` (`:437-448`).

**BR-MESH-12 — An unanchored membership table admits nothing (fail-closed).**
`expected_relay_pubkey: None` falls into the reject arm (`membership.rs:93`, the
`anchor =>` catch-all). Documented `membership.rs:36-40`; test
`unanchored_membership_rejects_all_ready_records` (`:465-471`). The relay always
anchors (`mesh_boot.rs:444-445`).

**BR-MESH-13 — Attestation binds the boot-unique endpoint pubkey to the deployment
identity, over a versioned textual preimage.** Preimage
`buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}`
(`registry.rs:85-91`), SHA-256'd (`registry.rs:93-96`), schnorr-signed
(`registry.rs:41`) / verified (`registry.rs:78-80`). Textual-and-versioned so
"integration can reproduce it exactly without depending on JSON key order"
(`registry.rs:82-84`).

**BR-MESH-14 — `runtime_pubkey` must equal `runtime_id.to_hex()`.**
`verify_attestation` rejects the mismatch before signature checking
(`registry.rs:140-145`). Tests
`ready_record_attestation_verifies_and_binds_runtime_pubkey` (`:341-349`) and
`attestation_rejects_signature_for_other_runtime` (`:352-357`).

**BR-MESH-15 — The mesh identity must NOT be the deployment Nostr key.**
`wire.rs:52-60`: all pods share one `BUZZ_RELAY_PRIVATE_KEY` Secret, so using it
would give every pod the same runtime id and "collapse the ownership plane."
Enforced by construction — the only RuntimeId source is
`SecretKey::generate()`/`public()` (`endpoint.rs:20`, `:38`).

**BR-MESH-16 — RuntimeId is boot-unique.** Fresh keypair every
`MeshEndpoint::bind` (`endpoint.rs:19-21`), documented `endpoint.rs:8-9`,
`wire.rs:47-50`. `bind_with_secret_key` exists only for stable test identities
(`endpoint.rs:23-25`).

**BR-MESH-17 — Inbound connections are admitted only from runtime ids present in the
membership table; unknown ids get exactly one registry rescan first.**
`accept_loop` → `is_known_peer` (`runtime.rs:262-270`, `:305-323`): `has_peer` →
if false and a registry exists, one `scan_ready` + `apply_ready_records`, then
re-check. Rejection logs `"rejected inbound connection from unattested runtime id"`
(`runtime.rs:265-269`) and `continue`s. Documented as gating *dialability*, never
ownership (`runtime.rs:11-14`, `membership.rs:184-186`).

**BR-MESH-18 — ALPN mismatch is a hard reject at connection construction.**
`MeshPeer::from_connection` returns `MeshError::Transport("unexpected mesh ALPN …")`
when `conn.alpn() != ALPN` (`peer.rs:50-55`). ALPN is `buzz/mesh/1`
(`wire.rs:37`) and is documented to change on every wire-version bump so rolling
deploys never half-speak (`wire.rs:34-36`).

---

#### D. Gossip, anti-entropy, record versioning

**BR-MESH-19 — Scuttlebutt push-pull: a received `Digest` is answered with a
`Delta`; a received `Delta` is applied.** `control_stream_exchange`
(`runtime.rs:531-551`): `Digest{entries}` → `membership.delta_for(&entries)` →
send back as `MeshStreamFrame::Gossip`; `Delta{records}` → `apply_gossip_record`
per record.

**BR-MESH-20 — Only the owning runtime may increment its own record version.**
`gossip.rs:23-24`. Enforced locally: `update_local` is the only version bumper
(`membership.rs:166-178`, `gossip.rs:100-109`) and it only ever touches
`self.local_record`. Enforced remotely by BR-MESH-22 (self-records rejected).
**Not enforced cryptographically** — see `-security.md` S-01.

**BR-MESH-21 — Conflict resolution is strict last-version-wins; equal versions
lose.** `membership.rs:128` (`record.version <= peer.record.version => false`) and
`gossip.rs:154-157` (`record.version > existing.version`). No vector clocks, no
timestamp tie-break. Test `stale_gossip_record_is_ignored` (`membership.rs:474-479`)
and `apply_delta_ignores_stale_versions` (`gossip.rs:253-266`).

**BR-MESH-22 — A runtime never accepts a gossiped record about itself.**
`membership.rs:121-123` (returns `false`), `membership.rs:87-89` (`continue`).
Prevents a peer from rewriting our own advertised addrs/draining flag.
Test `ready_records_seed_peers_but_skip_self` (`membership.rs:425-434`).

**BR-MESH-23 — A version bump always refreshes `heartbeat_millis`.**
`membership.rs:174-176`, `gossip.rs:105-107`. Version is `saturating_add(1)`, so it
never wraps (it saturates at `u64::MAX`).

**BR-MESH-24 — Ready-registry seeds enter as version-1 hints; existing newer gossip
records win.** `apply_ready_records` builds a fresh `GossipRecord::new(...)`
(version 1, `gossip.rs:38`) and routes it through `apply_gossip_record`
(`membership.rs:110-116`), which rejects it if the table already holds version ≥ 1.
Documented `membership.rs:76-78`. Consequence: after the first gossip tick a peer's
registry-published `endpoint_addrs` can never again correct a stale gossiped
address, because the seed is permanently version 1.

**BR-MESH-25 — Digest and Delta contents are deterministically ordered by
`runtime_id.to_hex()`.** `membership.rs:216`, `:241`; `gossip.rs:118`, `:144`.

**BR-MESH-26 — A digest/delta covers *all* records the runtime knows, local
included.** `records()` appends the local record (`membership.rs:204`), and both
`digest()` (`:210`) and `delta_for()` (`:232`) build from it. This is how a new pod's
own record reaches peers that have not yet rescanned Redis.

**BR-MESH-27 — Gossip payload version must equal `GOSSIP_PAYLOAD_VERSION`.**
`decode_message` rejects any other value with
`MeshError::Transport("unknown gossip payload version {v}")` (`gossip.rs:68-76`).
Note this is a `Transport` string, not a typed variant — unlike the outer frame's
`UnknownWireVersion` (`wire.rs:185`).

**BR-MESH-28 — Gossip frames carry no fence and no attestation.**
`MeshStreamFrame::Gossip { payload }` (`wire.rs:144-146`) has no `FencedHeader`;
`apply_gossip_record` (`membership.rs:120-153`) performs **no** signature or
relay-anchor check — unlike `apply_ready_records`. This is the single largest
asymmetry in the trust model; see `-security.md` S-01.

**BR-MESH-29 — Gossip is periodic and idempotent, so dropping a frame under
backpressure is preferred over blocking.** `send_control_frame` uses `try_send` and
discards the error (`runtime.rs:553-558`, comment at `:554-555`). Queue depth 64
(`runtime.rs:46`).

**BR-MESH-30 — Every gossip tick bumps the local record even when nothing changed.**
`gossip_tick_loop` calls `update_local(|_| {})` (`runtime.rs:566`) purely so peers'
phi accrual sees life, then broadcasts a digest to every peer with a live control
queue (`runtime.rs:567-585`). Interval `DEFAULT_GOSSIP_INTERVAL = 2s`
(`runtime.rs:44`).

---

#### E. Failure detection

**BR-MESH-31 — Heartbeat cadence is 2 s (gossip tick); registry refresh is 15 s.**
`runtime.rs:44`; `config.rs:511` / `registry.rs:20`. The gossip tick sleeps *before*
its first bump (`runtime.rs:564`), so the first heartbeat lands one interval after
`MeshRuntime::start`.

**BR-MESH-32 — Phi is computed only from observed heartbeat *intervals*, and only
advances on a strictly newer record.** `PhiAccrual::observe` is called only from
`apply_gossip_record` (`membership.rs:132`, `:139`) — i.e. inside the branches that
accepted a newer version. A peer whose gossip stalls but whose QUIC connection is
alive will still accrue suspicion.

**BR-MESH-33 — Phi formula: `(elapsed / mean_interval) / ln(10)`.**
`gossip.rs:203-215`, comment `:213`. Returns `None` when no heartbeat seen, when
fewer than 2 heartbeats have been observed (`samples.is_empty()`, `:207`), or when
mean ≈ 0 (`:211`). **Sample variance is unused** (`mean_secs`, `:217-220`, is the
only statistic) — this is an exponential approximation, not the
distribution-based phi-accrual detector the type name implies.

**BR-MESH-34 — Suspicion threshold is 8.0, hardcoded in production.**
`DEFAULT_PHI_SUSPECT_THRESHOLD = 8.0` (`membership.rs:17`). `with_phi_suspect_threshold`
(`membership.rs:66-69`) exists but has **zero callers** — the relay never overrides
it (`mesh_boot.rs:444-445` only anchors the relay pubkey). With a ~2 s mean, phi ≥ 8
requires `8 × ln(10) × 2s ≈ 36.8 s` of gossip silence.

**BR-MESH-35 — A suspect peer is filtered out of `peers()` entirely and reported as
`connection_state: "suspect"` in `/_mesh`.** `RelayMeshMembership::peers`
(`membership.rs:359-379`) returns `None` for `phi >= threshold` (`:366-369`);
`peer_statuses` overrides the stored state to `Suspect` (`membership.rs:340-344`).
Suspicion is recomputed per call — never persisted.

**BR-MESH-36 — Suspicion does not evict, disconnect, or stop dialing.** Verified:
`membership.rs` has no `remove`/`retain`/`clear`; `reconcile_once`
(`runtime.rs:296-326`) filters only on `runtime_id != local && !record.draining`,
so a suspect peer is still dialed every 5 s and still passes `has_peer` admission
(BR-MESH-17) forever. Suspicion is purely advisory — and since `peers()` has zero
production readers (see `-api-surface.md` §6), it currently has **no effect at all**.

**BR-MESH-37 — A peer whose recv loop errors is removed from the *connection* table
but not the membership table.** `datagram_recv_loop` / `stream_accept_loop` call
`remove_peer` on any error (`runtime.rs:359-363`, `:379-383`); `remove_peer`
(`runtime.rs:267-281`) aborts the entry's tasks and sets
`ConnectionState::Disconnected`. The `MeshMembership.peers` entry survives.

---

#### F. Peer selection, dialing, reconnect

**BR-MESH-38 — The mesh is a warm full mesh: every known non-draining peer is dialed
proactively.** `reconcile_once` (`runtime.rs:296-326`) dials all candidates not
already connected; rationale at `runtime.rs:15-19` — "failover is 'next frame goes
elsewhere,' not 'wait for a handshake.'" There is **no peer-selection or fan-out
sampling** — the "gossip fan-out" is 1:N to every connected peer
(`runtime.rs:575-585`), not the log-N sampling classical scuttlebutt uses.

**BR-MESH-39 — Draining peers are never dialed.** `reconcile_once` filter
`!record.draining` (`runtime.rs:302`).

**BR-MESH-40 — Reconcile cadence 5 s, preceded by one immediate pass at boot.**
`DEFAULT_RECONCILE_INTERVAL` (`runtime.rs:42`); `reconcile_loop` runs
`reconcile_once` *then* sleeps (`runtime.rs:288-293`); the relay additionally forces
one pass before returning the handle (`mesh_boot.rs:478`, rationale
`runtime.rs:148-149`).

**BR-MESH-41 — Dial tries a peer's advertised addresses in order and stops at the
first success.** `dial_peer` (`runtime.rs:328-355`): unparseable addr → warn +
next; bad runtime id → warn + `return` (abandons the whole peer); connect success →
`install_peer` + `return`; connect failure → warn + next addr. All addrs exhausted →
mark `Disconnected` (`runtime.rs:351-354`).

**BR-MESH-42 — There is no reconnect backoff.** Verified: no jitter, no exponential
delay, no failure counter anywhere in `runtime.rs`. A permanently unreachable peer
record is re-dialed on every 5 s tick, and each dial `await`s inside the sequential
loop (`runtime.rs:326`), so N dead peers serialize their connect timeouts into one
reconcile pass. Combined with BR-MESH-36 (no eviction) this is unbounded churn.

**BR-MESH-43 — Simultaneous-dial tie-break: the connection dialed by the numerically
smaller runtime id wins.** `new_connection_wins(local, remote, new_dialed_by_us)`
(`runtime.rs:204-213`): if `local.0 < remote.0` the winner is our outbound
(`new_dialed_by_us`), else the peer's inbound (`!new_dialed_by_us`). Byte-wise
lexicographic comparison of the 32-byte arrays. Deterministic and symmetric on both
ends (`runtime.rs:20-22`); tests `tie_break_is_symmetric` (`runtime.rs:846-856`) and
`simultaneous_dial_converges_to_one_connection` (`runtime.rs:858-...`).

**BR-MESH-44 — The losing connection's tasks are aborted and its entry replaced.**
`install_peer` (`runtime.rs:216-260`): if an entry exists and the new connection
loses, log + `return false` (keep existing, `:224-227`); if it wins,
`existing.abort()` (`:228`) then insert. `PeerEntry::abort` aborts every
`JoinHandle` (`runtime.rs:56-62`).

**BR-MESH-45 — Exactly one control stream per connection, opened by the dialer.**
`install_peer` spawns `open_control_stream` only when `dialed_by_us`
(`runtime.rs:236-247`); the acceptor learns its writer queue when
`Hello{Control}` arrives (`runtime.rs:441-448`). Documented `wire.rs:149-151`,
`runtime.rs:20-21`.

**BR-MESH-46 — Only peers with a live control queue receive gossip digests.**
`gossip_tick_loop` filters `control_tx.is_some()` (`runtime.rs:576`). An acceptor
that never receives a `Hello{Control}` is gossip-silent but still connected.

---

#### G. Wire discipline / framing rules

**BR-MESH-47 — Every encoded frame is prefixed with a one-byte wire version;
unknown versions are rejected loudly, never guessed.** `encode`
(`wire.rs:176-179`), `decode` (`wire.rs:182-188`) → `UnknownWireVersion` /
`EmptyFrame`. Documented `wire.rs:38-41`. Test `unknown_version_rejected`
(`wire.rs:246-257`).

**BR-MESH-48 — The first frame on any stream, in both directions, MUST be `Hello`.**
`wire.rs:29-31`, `wire.rs:128`. Enforced on the accept side: a non-`Hello` first
frame logs `"stream without Hello — dropped"` and `continue`s
(`runtime.rs:400-411`). **Delta vs doc:** `wire.rs:31` says the stream "is reset";
the implementation drops the `MeshStream` value and moves on — no explicit QUIC
`reset()` call exists in the crate. Also, the contract says "in both directions,"
but nothing validates the *peer's* Hello on a stream we opened
(`runtime.rs:190-191` sends ours and returns the stream unread).

**BR-MESH-49 — Stream frames are u32-LE length-delimited, capped at 16 MiB.**
Send: `write_all(len.to_le_bytes())` then body (`peer.rs:148-158`), with
`FrameTooLarge` pre-check (`peer.rs:142-147`). Recv: `read_exact(4)` → length →
`FrameTooLarge` if over cap → `read_exact(len)` (`peer.rs:169-191`).
`FinishedEarly(0)` maps to a clean `Ok(None)` end-of-stream (`peer.rs:172`).

**BR-MESH-50 — Datagram boundary is the frame boundary; senders must check size and
fail loud, never truncate.** `wire.rs:19-22`; `encode_datagram_checked`
(`lib.rs:213-226`) and `MeshPeer::send_datagram` (`peer.rs:105-116`) →
`DatagramTooLarge`. A peer with no datagram support yields
`Transport("peer does not support QUIC datagrams")` (`peer.rs:109`).
Test `oversized_datagram_is_rejected_before_send` (`endpoint.rs:239-254`).

**BR-MESH-51 — Datagram receivers tolerate gaps and reordering and never wait.**
`wire.rs:114-116`; `seq` is observability-only. `datagram_recv_loop`
(`runtime.rs:358-372`) never buffers or reorders.

**BR-MESH-52 — `RealtimeMedia` is datagram-only; `HuddleControl` is stream-only.**
`wire.rs:99-108` (HuddleControl "rides a reliable stream … never datagrams,
[because] a dropped roster delta is an unrecoverable peer-index desync"). Enforced
in the consumer's dispatcher: a stream claiming `RealtimeMedia` is rejected
(`mesh_boot.rs:118-126`), test `dispatcher_routes_session_streams_by_profile`
(`mesh_boot.rs:601-635`).

**BR-MESH-53 — Non-gossip frames on the control stream are logged and ignored, not
fatal.** `runtime.rs:545-547`.

**BR-MESH-54 — Control streams never reach the inbound handler; session streams
without a registered handler are dropped with a warning.** `runtime.rs:437-459`
(Control handled in-runtime); `runtime.rs:465-470` (no handler → warn + drop).
Consumer-side equivalent at `mesh_boot.rs:99-104`, `:128-134`. Documented as a
bounded boot-window race made safe by the peer's fenced retry
(`mesh_boot.rs:53-55`).

**BR-MESH-55 — Changes to `wire.rs` require a post in the mesh thread before the
edit.** `wire.rs:10-13`. Process rule, unenforced by tooling — no CODEOWNERS entry
or CI check references `wire.rs` (verified: `.github/CODEOWNERS` has no mesh entry).

---

#### H. Drain / lifecycle

**BR-MESH-56 — Drain sets a local flag and gossips `draining=true` (one version
bump).** `begin_drain` (`membership.rs:385-388`): `draining.store(true)` +
`update_local(|r| r.draining = true)`. Test `begin_drain_updates_local_record`
asserts version goes 1→2 (`membership.rs:482-489`).

**BR-MESH-57 — Drain is triggered by the relay's `shutting_down` flag, polled every
500 ms, one-shot.** `mesh_boot.rs:481-497` — the watcher `return`s after firing, so
drain cannot be un-set.

**BR-MESH-58 — `GoodbyeReason::Draining` tells the peer to re-establish elsewhere;
`SessionEnded` is normal; `StaleGeneration` is the sender fencing itself out.**
`wire.rs:166-174`. All three are produced by `buzz-relay` consumers, not by this
crate.

**BR-MESH-59 — A clean `Goodbye` is semantically distinct from a QUIC reset;
receivers treat reset as abnormal.** `wire.rs:135-137`.

**BR-MESH-60 — The heartbeat clears the registry record on the ready→not-ready
edge.** `ReadyHeartbeat::tick` (`registry.rs:295-304`): publish while ready; `DEL`
once on the falling edge (guarded by `published`), then stop.
`ReadyHeartbeat::shutdown()` (`:306-312`) does the same explicitly but has **zero
callers**. Clean-shutdown removal therefore depends entirely on the heartbeat task
getting at least one post-SIGTERM tick within its 15 s interval before the process
exits — the relay's graceful window is 30 s (`main.rs:1108`), so this usually holds
but is not guaranteed.

**BR-MESH-61 — `MeshRuntime` loops outlive all handle clones; only `shutdown()`
stops them.** `runtime.rs:75-76`, `shutdown` (`runtime.rs:155-164`) aborts the 3
loops and every peer entry. **Zero production callers** — the relay never calls it.
`spawn_registry_heartbeat`'s task (`runtime.rs:598-607`) has no shutdown path at all.

---

#### I. Version negotiation (exhaustive)

There are **four independent version channels**, and only two of them actually
negotiate anything:

| # | Channel | Value | Advertised | Checked | Effect of mismatch |
|---|---|---|---|---|---|
| V1 | **ALPN** `buzz/mesh/1` | `wire.rs:37` | offered at bind (`endpoint.rs:34`) and dial (`endpoint.rs:88`) | by QUIC/TLS, plus a belt-and-braces check `conn.alpn() != ALPN` (`peer.rs:50-55`) | connection never establishes; no half-speaking (`wire.rs:34-36`) |
| V2 | **frame version byte** `WIRE_VERSION = 1` | `wire.rs:42` | first byte of every datagram and stream frame (`wire.rs:177`) | `decode` (`wire.rs:183-186`) | `MeshError::UnknownWireVersion(v)`; frame dropped, loop continues |
| V3 | **gossip payload version** `GOSSIP_PAYLOAD_VERSION = 1` | `gossip.rs:13` | in-struct field on both `Digest` and `Delta` (`gossip.rs:52-59`) | `decode_message` (`gossip.rs:68-76`) | `MeshError::Transport("unknown gossip payload version …")`; frame logged + dropped (`runtime.rs:548-550`) |
| V4 | **`proto_version: u16`** | `WIRE_VERSION as u16` (`mesh_boot.rs:367`) | in `ReadyRecord.proto_version` (`registry.rs:110`) and `GossipRecord.proto_version` (`gossip.rs:19`) | **never** | none — see below |

**BR-MESH-62 — ALPN is the real negotiation boundary; V1 is redundant with it but
harmless.** Because ALPN embeds the version, two pods with different wire versions
cannot connect at all, which makes V2's `UnknownWireVersion` unreachable across a
rolling deploy and reachable only from a same-ALPN peer sending a corrupt byte.

**BR-MESH-63 — `proto_version` is advertised and never compared.** Verified: grep
for `proto_version` in `crates/**` shows only assignment sites
(`gossip.rs:19,33,34`, `registry.rs:110,126`, `membership.rs:112`, `status.rs:21`,
`membership.rs:337`, `mesh_boot.rs:367,439,451`) and the `/_mesh` echo — no
comparison, no gate, no rejection. The field is pure observability.

**BR-MESH-64 — `capabilities` is advertised and never consulted.**
`mesh_boot.rs:371-377` ships a static `["reliable-stream", "realtime-media",
"huddle-control"]`; `apply_ready_records` copies it onto the gossip record
(`membership.rs:113`); nothing reads it. There is **no capability negotiation** —
`open_session_stream` will happily open a `HuddleControl` stream to a peer that
never advertised it, and the only backstop is the receiver's dispatcher
(`mesh_boot.rs:112-134`).

**BR-MESH-65 — Wire enums are exhaustive and postcard-strict, so forward
compatibility within one ALPN is zero.** No `#[non_exhaustive]` anywhere
(verified). A future 5th `MeshStreamFrame` variant decodes to `MeshError::Decode`
on an older peer at the same ALPN — the design intent (bump ALPN with the wire
version) is the only safe path, and it is unenforced.

---

#### J. Ownership / lease rules (what this crate does *not* do)

**BR-MESH-66 — This crate grants no ownership and holds no lease.** `lib.rs:33-35`,
`registry.rs:3-6`, `gossip.rs:3-5`, `membership.rs:3-6`. Verified by absence: no
Redis `SET NX`, no `INCR`, no `EVAL`/Lua, no CAS anywhere in the crate. The only
Redis commands are `SET … EX`, `DEL`, `SCAN`, `GET` (`registry.rs:188-228`), all
against `mesh:ready:*`.

**BR-MESH-67 — `owner_runtime_id` in the fenced header is advisory; the generation
is what fences.** `wire.rs:90-92`. Consistent with the consumer, where
`OwnerMismatch` is raised by the Redis-backed directory
(`tunnel/directory.rs:430`), not by the transport.

**BR-MESH-68 — Generation monotonicity across owner death is guaranteed elsewhere.**
Stated by the consumer at `audio/mesh.rs:44-46` ("the directory's companion INCR
counter … this module trusts that"). Nothing in `buzz-relay-mesh` maintains or
validates that counter.

---

#### K. Rules documented but not implemented (delta list)

| Claim | Where claimed | Reality |
|---|---|---|
| Counter `mesh_fence_rejections_total{reason=…}` | `lib.rs:102-109` | does not exist — grep finds no such metric in the repo |
| `MeshCounters.stale_generation_rejections` reflects fence rejects | `status.rs:43` | always 0; the only writer `record_stale_generation_rejection` (`membership.rs:285`) has no caller but the test at `:486` |
| `BUZZ_MESH` is "`on` (default when replicas can exist)" | `lib.rs:55-56` | default is **off** (`config.rs:498-500`); tested `mesh_boot.rs:546-555` |
| `PeerInfo.load` is "gossiped by the peer (0.0..)" | `lib.rs:137-138` | structurally always `0.0` — nothing ever writes a non-zero `load` |
| Non-Hello first frame ⇒ "the stream is reset" | `wire.rs:31` | dropped, not reset (`runtime.rs:404-406`) |
| `MeshStream` halves are "placeholder … pre-transport" | `lib.rs:186-188` | they are the real iroh halves (`peer.rs:132-192`) |
| Relay consumes the crate "exclusively through two seams" | `lib.rs:11-19` | also uses `InboundHandler`, `MeshStream`, both half traits, `MeshEndpoint`, `MeshPeer`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshRuntime`, `spawn_registry_heartbeat` |
