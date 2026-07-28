## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Technical Debt

Baseline health is good: 2,118 lines of Rust, zero `unsafe`, zero `TODO`/`FIXME`/
`HACK` markers, zero `unwrap`/`expect` outside `#[cfg(test)]`, 34 tests. The debt is
concentrated in (a) a capability the crate advertises but never implemented, (b) an
untested security control, and (c) unreachable knobs.

---

### Summary

| # | Item | Severity |
|---|---|---|
| D-PS-1 | Typing indicators advertised in the manifest and ARCHITECTURE.md; **no implementation exists** | High (doc integrity) |
| D-PS-2 | `rate_limiter.rs` has **zero tests** despite being a security control | High |
| D-PS-3 | `retain_topic` can block forever / silently lose subscriptions if the subscriber task isn't running | High |
| D-PS-4 | Control messages are unversioned and unauthenticated; a rolling deploy can silently skip a ban | Medium-High |
| D-PS-5 | ARCHITECTURE.md presence key omits community scope | Medium |
| D-PS-6 | `lib.rs:331` says 60 s presence TTL; actual is 90 s | Medium |
| D-PS-7 | `check_ip_connection` is dead code — and pre-auth traffic is unmetered | Medium |
| D-PS-8 | Fixed-window limiter permits 2× burst (known, self-documented) | Medium |
| D-PS-9 | Clean-disconnect backoff reset enables a 1 s hot reconnect loop | Medium |
| D-PS-10 | Redis is a hard availability dependency for authenticated reads, with no degraded mode | Medium |
| D-PS-11 | No metrics — reconnects, drops, lag, and refcounts are invisible | Medium |
| D-PS-12 | Hardcoded capacities/TTLs; unsubscribe-debounce knob unreachable in production | Low-Medium |
| D-PS-13 | Triplicated reconnect loops and backoff constants (~50 duplicated lines) | Low-Medium |
| D-PS-14 | Broken intra-doc link `[crate::ConnectionManager]` | Low |
| D-PS-15 | No presence index — "who is online" is unanswerable from this crate | Low-Medium |
| D-PS-16 | `pub mod subscriber` is an empty public namespace | Low |
| D-PS-17 | `channel_key`/`global_key` reachable by three paths | Low |
| D-PS-18 | Unused `chrono` dependency | Low |
| D-PS-19 | `topic_refcount` documented for metrics but unwired | Low |
| D-PS-20 | Test-fixture duplication and inconsistent test Redis endpoint | Low |
| D-PS-21 | 11 of 34 tests `#[ignore]`d behind live Redis | Low |
| D-PS-22 | `missing_docs` is `warn`, not `deny` | Low |

---

### D-PS-1 — Typing indicators: advertised everywhere, implemented nowhere (High)

Four independent places claim this crate implements typing indicators:

| Claim | Location |
|---|---|
| `description = "Redis pub/sub fan-out, presence, and typing indicators for Buzz"` | `crates/buzz-pubsub/Cargo.toml:8` |
| Doc comment `/// Typing indicator tracking in Redis.` | `crates/buzz-pubsub/src/lib.rs:43` |
| `buzz-pubsub (Redis pub/sub, presence, typing indicators)` in the crate map; section heading `### buzz-pubsub — Redis Pub/Sub, Presence, Typing`; "Manages … typing indicators" | `ARCHITECTURE.md:82`, `:432`, `:434` |
| `buzz-pubsub  # Redis pub/sub fan-out, presence, typing indicators` | `AGENTS.md` repo-structure table |

ARCHITECTURE.md goes further and specifies a concrete Redis design:

```
ZADD buzz:typing:{channel_id} {now_unix} {pubkey_hex}
ZREMRANGEBYSCORE buzz:typing:{channel_id} -inf {now - 5.0}
EXPIRE buzz:typing:{channel_id} 60
```
(`ARCHITECTURE.md:452-456`), lists `buzz:typing:{channel_uuid}` as a live 60 s
sorted-set key in the Redis keyspace table (`ARCHITECTURE.md:801`), and describes
Redis as providing "typing (sorted sets)" in the infrastructure table
(`ARCHITECTURE.md:777`).

**None of it exists.** Evidence of absence:
- No `typing` module — the module list ends at `pub mod topic` (`lib.rs:42`). The doc
  comment at `lib.rs:43` is **mis-attached to `pub use error::PubSubError;`**
  (`lib.rs:44`), which is how the claim survived: rustdoc renders it as the
  re-export's description, so `missing_docs` never fired.
- `ZADD`/`ZREMRANGEBYSCORE` appear **once** in all of `crates/**/*.rs`, and only as
  words inside the crate-header comment listing example commands (`lib.rs:10`). There
  is no sorted-set call anywhere in the workspace.
- The string `buzz:typing` appears **zero** times in the repository.

What actually happens: typing is a plain ephemeral Nostr event,
`KIND_TYPING_INDICATOR = 20002` (`crates/buzz-core/src/kind.rs:331`, registered in
`ALL_KINDS` at `:549`), produced by the agent harness
(`crates/buzz-acp/src/config.rs:380-382`, `:517`, refresh loop
`crates/buzz-acp/src/lib.rs:1593-1602`) and fanned out through the generic
`publish_event` path — consistent with `ARCHITECTURE.md:258-263`, which contradicts
`:452-456`. There is no server-side typing *state*, which is also why
`ARCHITECTURE.md:824` correctly notes there is no way to query current typers.

Fix is a choice, not a code change: either implement the ZSET design or delete the
claim from `Cargo.toml:8`, `lib.rs:43`, `AGENTS.md`, and `ARCHITECTURE.md:82`,
`:432`, `:434`, `:452-456`, `:777`, `:801`. Deleting is the honest option — the
event-based approach works and needs no Redis state.

### D-PS-2 — The Redis rate limiter has no tests (High)

`rate_limiter.rs` contains **zero test functions** — the only file in the crate with
public logic and no tests (alongside `error.rs` and `publisher.rs`, which are
trivial). Untested behaviours that matter:

- The Lua script itself (`rate_limiter.rs:24-31`) — that `EXPIRE` fires only when
  `count == 1`, and that the returned `{count, ttl}` tuple decodes as `(u64, i64)`.
- The allow boundary `count <= limit` (`rate_limiter.rs:74-78`) — an off-by-one here
  changes every limit in the product by one request.
- The TTL-repair branch (`rate_limiter.rs:57-70`), including that repair resets the
  window to full duration (i.e. a key in the broken state gets a fresh allowance).
- Key construction for both pubkey and IP paths (`rate_limiter.rs:110`, `:118`).

`buzz-relay`'s admission tests do not close this gap: they substitute a
`#[cfg(test)] StubLimiter` (`crates/buzz-relay/src/admission.rs:65-90`) that returns
canned `denied`/`Err` values and never touches Redis. So the enforcement *decision*
is tested while the *counting* is not, anywhere in the repo.

### D-PS-3 — `retain_topic` can block forever, or silently lose a subscription (High)

`retain_topic` increments the refcount, then awaits
`subscription_tx.send(Subscribe(topic))` and **discards the result**
(`lib.rs:203-207`). Two failure modes:

1. **Silent loss.** If the send fails (receiver dropped), the refcount already says
   "retained" but no Redis `SUBSCRIBE` will ever be issued. The connection believes
   it is subscribed and receives nothing — a silent correctness failure with no log.
2. **Unbounded await.** The channel is `mpsc::channel(4096)` (`lib.rs:129`), so
   `send` awaits when full. If `run_subscriber` was never spawned, nothing drains the
   receiver; after 4096 retains, every subsequent `retain_topic` awaits indefinitely,
   stalling whatever relay task called it.

The take-once guard (`lib.rs:149-152`) protects against a *double* start but does
nothing about a *missing* start. The same discard pattern is at `lib.rs:238-243`
(release path), where the consequence is milder — a stale Redis subscription.

Minimal fix: log on send failure, and make the "subscriber not started" state
observable rather than silent.

### D-PS-4 — Unversioned, unauthenticated control messages (Medium-High)

`CacheInvalidation` (`cache_invalidation.rs:57-80`) and `ConnControl`
(`conn_control.rs:55-73`) are internally-tagged enums with no version field, no
origin-pod id, no timestamp, no signature, and no `#[non_exhaustive]`.

Rolling-deploy consequence: a pod running an older binary receives a new variant,
fails deserialization, and skips it with a `warn`
(`cache_invalidation.rs:161-165`, `conn_control.rs:152-156`). For
`ConnControl::DisconnectPubkey` that means a ban is **not live-enforced** on old pods
until they restart. The DB row is the durable backstop
(`conn_control.rs:18-21`) so the member cannot re-authenticate, but an
already-connected socket survives. A forward-compat test pins the skip behaviour
(`conn_control.rs:209-217`) — the behaviour is intentional; the deploy-ordering
consequence is undocumented.

Also: `pubkey` is documented as "32 raw bytes" (`conn_control.rs:60`) but typed
`Vec<u8>` and length-validated on neither publish nor receive.

Injection risk from Redis write access is covered in the security aspect
(attacker-controlled `reason` echoed to the victim, arbitrary disconnects).

### D-PS-5 / D-PS-6 — Presence documentation drift (Medium)

| Claim | Reality |
|---|---|
| `ARCHITECTURE.md:450`: `SET buzz:presence:{pubkey_hex} {status} EX 90` | Actual key is `buzz:{community}:presence:{pubkey_hex}` (`presence.rs:19-25`) — the community segment is unconditional |
| `ARCHITECTURE.md:800`: lists `buzz:presence:{pubkey_hex}`, hedged as "single-community form; shared multi-community Redis **must** scope by community" | The code has no unscoped form; the hedge describes a requirement that is already met, which reads as though it isn't |
| `lib.rs:331`: "Set presence with **60s** TTL" | `PRESENCE_TTL_SECS = 90` (`presence.rs:16`); the module doc correctly says `EX 90` (`presence.rs:5`) and explains the 3× rationale (`presence.rs:4-6`, `:15`) |

D-PS-6 is an internal contradiction inside one crate, and it is the doc a caller sees
first (it is on the `PubSubManager` method, not the private helper). ARCHITECTURE.md
happens to be right about the 90 s value while the code comment is wrong.

### D-PS-7 — Dead IP limiter, live pre-auth gap (Medium)

`check_ip_connection` (`rate_limiter.rs:112-120`) has no production caller. Across
`crates/**/*.rs`, `LimitType::IpConnections` and `ip_rate_limit_key` appear only in
the `buzz-auth` definitions (`crates/buzz-auth/src/rate_limit.rs:66`, `:76`, `:213`),
this implementation, and a `#[cfg(test)]` stub
(`crates/buzz-relay/src/admission.rs:85`).

That matters because it is precisely the limiter that would cover the gap it leaves:
`enforce_ws_admission` returns `true` for any non-authenticated connection
(`crates/buzz-relay/src/connection.rs:602-609`) because the live limiter is
pubkey-keyed. Pre-AUTH WebSocket traffic is therefore unmetered by this subsystem —
functioning code exists to fix it and is simply not wired up.

### D-PS-8 — Fixed-window 2× burst (Medium, accepted)

Self-documented with a `⚠️` and a stated upgrade path: "Fixed windows allow up to 2×
burst at boundaries. Upgrade to sliding window or token bucket for strict limiting"
(`rate_limiter.rs:9-10`). Independently flagged at the call site:
"this is still a fixed-window limiter, so a Redis-backed token bucket would be a
better long-term fit for smoother refill behavior"
(`crates/buzz-relay/src/admission.rs:4-7`). Well-managed debt; recorded for
completeness.

### D-PS-9 — Clean-disconnect backoff reset permits a 1 s hot loop (Medium)

All three subscriber loops reset the backoff to `BACKOFF_INITIAL_SECS` whenever the
stream ends *cleanly* (`subscriber.rs:57-63`, `cache_invalidation.rs:105-111`,
`conn_control.rs:95-101`). The intent is stated and reasonable — "a brief Redis
restart should reconnect quickly" (`subscriber.rs:58-60`).

Failure mode: an endpoint that accepts a connection and immediately closes the stream
(misconfigured TCP proxy, `maxclients` eviction, TLS terminator that drops pub/sub)
returns `Ok(())` every cycle, so the backoff never escalates past 1 s. Three loops ×
one reconnect per second, indefinitely, each opening a fresh `redis::Client`
(`subscriber.rs:85`, `cache_invalidation.rs:135`, `conn_control.rs:126`). A
consecutive-clean-end counter, or a minimum-uptime requirement before resetting,
would bound this.

### D-PS-10 — Redis on the critical read path with no degraded mode (Medium)

`AuthError::Internal` from the limiter (`rate_limiter.rs:44-46`, `:52`, `:66`) becomes
`AdmissionError::Unavailable` (`crates/buzz-relay/src/admission.rs:29-33`), which the
relay treats as denial (`crates/buzz-relay/src/connection.rs:612-621`). A Redis
outage therefore rejects every authenticated `EVENT`, `REQ`, and `COUNT` — reads
included. Fail-closed is the right default for a limiter, but there is no circuit
breaker, no in-process fallback limiter, and no documented operator override. Worth an
explicit decision record rather than an emergent property.

The replay guard has the same shape by contract (`nip98_replay.rs:53-57`, `:74-79`),
which is correct for a replay control but compounds the blast radius of one Redis.

Related unstated operational requirement: the replay seen-set assumes keys survive
their TTL. Under `maxmemory` with an eviction policy, Redis may evict
`buzz:{community}:nip98:*` markers early, silently shortening the replay window.
Nothing in the crate or the deployment docs asserts `noeviction`.

### D-PS-11 — No metrics (Medium)

Observability is `tracing` logs only. Nothing counts events published or received,
messages dropped for parse failure (`subscriber.rs:143`, `:154`), broadcast lag
(`error.rs:22`), reconnect attempts, or active subscription counts. Prometheus is
part of the deployment, so the collection path exists. `topic_refcount`
(`lib.rs:248`) was written for exactly this and is unwired (D-PS-19).

### D-PS-12 — Hardcoded tunables; one knob unreachable (Low-Medium)

Four channel capacities at 4096 (`lib.rs:126-129`), `PRESENCE_TTL_SECS = 90`
(`presence.rs:16`), and the backoff envelope are all compile-time constants with no
operator override.

Worse, a knob that *was* built is unreachable: `PubSubConfig::with_unsubscribe_debounce`
(`lib.rs:93`) has zero callers outside this crate's tests, because every production
construction goes through `PubSubManager::new` (`lib.rs:117-119`), which hardcodes
`PubSubConfig::new`. All 11 relay call sites use `new`; the production one is
`crates/buzz-relay/src/main.rs:344` and the other ten are test fixtures. The 500 ms
default (`lib.rs:82`) is effectively a constant.

### D-PS-13 — Triplicated reconnect machinery (Low-Medium)

`BACKOFF_INITIAL_SECS` / `BACKOFF_MAX_SECS` are declared three times with identical
values (`subscriber.rs:16-19`, `cache_invalidation.rs:91-94`, `conn_control.rs:81-84`).
The three `run_*_subscriber` loops (`subscriber.rs:41-76`,
`cache_invalidation.rs:100-127`, `conn_control.rs:90-117`) are structurally identical,
and the two pattern-subscribe bodies (`cache_invalidation.rs:129-179` vs
`conn_control.rs:120-168`) differ only in payload type, pattern constant, and log
strings — roughly 50 duplicated lines. A generic
`run_pattern_subscriber<T: DeserializeOwned>(pattern, parse_scope, tx)` would collapse
both, and the shared backoff would then have one definition. `conn_control.rs:91-92`
already documents itself as "mirrors" the cache-invalidation loop.

### D-PS-14 — Broken intra-doc link (Low)

`conn_control.rs:7` links `[crate::ConnectionManager]`. No `ConnectionManager` exists
in `buzz-pubsub` — it is a `buzz-relay` type. The crate sets
`#![warn(missing_docs)]` (`lib.rs:2`) but not `#![deny(rustdoc::broken_intra_doc_links)]`,
so this produces a rustdoc warning and renders as literal text.

### D-PS-15 — No presence index (Low-Medium)

Presence is individual keys with no per-community roster set, so `get_presence_bulk`
(`presence.rs:74-95`) requires the caller to already know which pubkeys to ask about.
"List everyone online in this community" cannot be answered from Redis; the relay must
source candidates from the DB first (`crates/buzz-relay/src/api/bridge.rs:1954`).
Acceptable for member-list hydration, but it rules out cheap online counts and makes
presence cost scale with roster size rather than with online population.

### D-PS-16 to D-PS-19 — API-surface tidiness (Low)

- **D-PS-16:** `pub mod subscriber` (`lib.rs:40`) exports nothing — `DesiredTopics`
  (`subscriber.rs:21`), `SubscriptionCommand` (`:26`), and `run_subscriber` (`:41`) are
  all `pub(crate)`. An empty public namespace in the docs.
- **D-PS-17:** `channel_key`/`global_key` are reachable three ways —
  `buzz_pubsub::` (re-export `lib.rs:58`), `buzz_pubsub::topic::` (`topic.rs:103`,
  `:108`), and `buzz_pubsub::publisher::` (`publisher.rs:12`, `:17`, pure delegates).
  `publisher.rs` is 37 lines, two of which are redundant wrappers.
- **D-PS-18:** `chrono` is declared (`Cargo.toml:18`) but no `chrono::` path appears in
  any source file — unused dependency carrying build time and audit surface.
- **D-PS-19:** `topic_refcount` (`lib.rs:248`) is documented "for tests and metrics"
  and has zero non-test callers; no metric consumes it.

### D-PS-20 to D-PS-22 — Test hygiene (Low)

- **D-PS-20:** `test_presence_set_and_get` is defined twice with overlapping
  assertions (`lib.rs:477` and `presence.rs:138`). The `ctx(id, host)` fixture is
  copied six times (`lib.rs:388`, `topic.rs:119`, `presence.rs:119`,
  `cache_invalidation.rs:185`, `conn_control.rs:171`, `subscriber.rs:191`). Redis
  endpoint resolution is inconsistent: `nip98_replay.rs:110` honours `REDIS_URL`
  while `test_util::make_test_pool` (`lib.rs:371-376`) hardcodes `127.0.0.1:6379` —
  and the relay's own default is spelled `localhost:6379`
  (`crates/buzz-relay/src/config.rs:419`), which resolves differently on
  IPv6-preferring hosts.
- **D-PS-21:** 11 of 34 tests are `#[ignore = "requires Redis"]`, so the default
  unit-test gate runs 23 and covers none of the presence round-trip, replay-guard, or
  pub/sub round-trip behaviour. The two most valuable tests in the crate — the
  cross-community topic-isolation regression (`lib.rs:510-590`) and the replay-guard
  clamp tests (`nip98_replay.rs:164-200`) — only run when Redis is present.
- **D-PS-22:** `missing_docs` is `warn` (`lib.rs:2`), so doc coverage can regress
  without failing CI. D-PS-1's orphaned doc comment is a concrete example of what
  slips through.

### Inherited debt

`Cargo.toml:7` inherits `repository.workspace = true`, which resolves to the stale
`github.com/block/sprout` URL flagged in the Phase 2a findings — the same legacy
naming affects this crate's published metadata.
