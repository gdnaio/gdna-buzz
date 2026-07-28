## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Security

The crate performs **no authorization**. It states this explicitly and consistently:
pub/sub topics are "a routing/performance boundary, not an authorization boundary"
(`topic.rs:3-6`), and the relay re-checks access before local fan-out. Assessment
below covers what the crate *does* guarantee, and where the guarantees stop.

---

### 1. Tenant isolation — mechanism and limits

| Control | Implementation | Strength |
|---|---|---|
| Community id sourced only from server-resolved context | `EventTopicKey::from_context` reads `ctx.community()` (`topic.rs:35-40`); every key builder takes `&TenantContext` (`presence.rs:19`, `cache_invalidation.rs:30`, `conn_control.rs:33`, `topic.rs:103`, `:108`) | Strong — no public API accepts a caller-supplied community id |
| Delivered community comes from the parsed channel name, not the payload | `subscriber.rs:159-163`; control paths `cache_invalidation.rs:144-148`, `conn_control.rs:135-139` | Strong — a spoofed community field inside an event body cannot redirect routing |
| Strict channel-name grammar | `parse_redis_channel` `topic.rs:53-99`; rejects wrong prefix, non-UUID community, unknown scope word, and any extra segment | Strong — 13 negative cases asserted (`topic.rs:179-195`), including a `presence:` key (`:187`) |
| Refcount keyed by `(community, topic)` not by channel id | `subscriber.rs:21`, `lib.rs:193`, `:216` | Strong — regression test proves releasing community A does not silence community B (`lib.rs:510-590`) |
| Replay seen-sets isolated per community | `nip98_replay.rs:81` | Strong — asserted `nip98_replay.rs:144-162` |

**Limit:** isolation is key-prefix naming inside one shared Redis instance. There is
no per-tenant Redis logical db, no ACL, no separate credentials. Two subscribers
deliberately consume **all** tenants' control traffic via wildcard patterns —
`buzz:*:cache-invalidate` (`cache_invalidation.rs:27`, subscribed `:139`) and
`buzz:*:conn-control` (`conn_control.rs:30`, subscribed `:130`). Any process able to
reach Redis can read every community's event stream and inject forged control
messages. The trust model is stated: "Redis only ever carries events between nodes
inside the relay trust domain" (`lib.rs:317-318`).

### 2. Control-message integrity

Neither control payload is authenticated, versioned, timestamped, or
origin-attributed (`CacheInvalidation` `cache_invalidation.rs:57-80`; `ConnControl`
`conn_control.rs:55-73`). Consequences for an attacker with Redis write access:

- Publishing `ConnControl::DisconnectPubkey` to `buzz:{community}:conn-control`
  disconnects arbitrary members with an attacker-chosen `reason` string, which is
  echoed to the victim in the closing `OK` frame (`conn_control.rs:62-65`, dispatch
  `buzz-relay/src/main.rs:913`) — a denial-of-service and a message-injection
  surface into a security-relevant UI string. `reason` has no length or content
  validation on either side.
- Publishing `ConnControl::DisconnectCommunity` disconnects every socket for a
  community (`conn_control.rs:57-58`, dispatch `buzz-relay/src/main.rs:908`).
- Publishing a forged Nostr event to a topic cannot forge *content* — the relay
  applies `filter_fanout_by_access` on the Redis path
  (`buzz-relay/src/handlers/event.rs:135`, `main.rs:696`) and the event carries a
  Nostr signature — but it can cause unsolicited delivery attempts.
- `CacheInvalidation` forgery is comparatively benign by design: the payload is a
  pure idempotent key drop and the next read re-fetches authoritative state from
  the DB (`cache_invalidation.rs:11-14`). Worst case is added DB load.

`pubkey` in both `CacheInvalidation::Membership` and `ConnControl::DisconnectPubkey`
is typed `Vec<u8>` while documented as "32 raw bytes" (`conn_control.rs:60`); no
length check exists on publish or receive, so a malformed length propagates to the
relay's matching logic.

Mitigating factor for `ConnControl`: the DB ban row is the durable backstop, so a
*dropped* message still denies the next auth attempt (`conn_control.rs:18-21`). That
protects availability of the control, not its integrity against injection.

### 3. Rate limiting (`rate_limiter.rs`)

| Property | Finding |
|---|---|
| Atomicity | `INCR` + conditional `EXPIRE` + `TTL` in a single Lua script (`rate_limiter.rs:24-31`), eliminating the crash window that could leave a TTL-less key (`:19-22`) |
| Fixed-window weakness | Self-documented: up to **2× burst** at window boundaries; "upgrade to sliding window or token bucket for strict limiting" (`rate_limiter.rs:9-10`). Echoed at `buzz-relay/src/admission.rs:4-7` |
| Broken-state repair | `ttl < 0` triggers a fresh `EXPIRE` plus a `warn` (`rate_limiter.rs:57-70`). Note the repair resets the window to full duration, so a key in that state effectively gets a fresh allowance |
| Allow boundary | Inclusive — `count <= limit` allows (`rate_limiter.rs:74-78`), so the Nth request within a limit of N succeeds |
| Failure mode | Redis error → `AuthError::Internal` (`rate_limiter.rs:44-46`, `:52`, `:66`) → relay maps to `AdmissionError::Unavailable` (`buzz-relay/src/admission.rs:29-33`) → request **denied**. Fail-closed, which is correct, but makes Redis a hard dependency for all authenticated reads and writes (`buzz-relay/src/connection.rs:612-621`) |
| Coverage gap | **Unauthenticated frames are never rate limited.** `enforce_ws_admission` returns `true` for any connection not in `AuthState::Authenticated` (`buzz-relay/src/connection.rs:602-609`), because the limiter is pubkey-keyed. Pre-AUTH traffic is unmetered by this control |
| Scope gap | Only `EVENT`, `REQ`, and `COUNT` are metered; every other frame type short-circuits (`buzz-relay/src/connection.rs:598-601`) |
| Dead control | `check_ip_connection` (`rate_limiter.rs:112-120`) — the one limiter that could cover pre-auth traffic — has **no production caller**. Repo-wide, `LimitType::IpConnections` and `ip_rate_limit_key` appear only in the `buzz-auth` definitions (`buzz-auth/src/rate_limit.rs:66`, `:76`, `:213`), this impl, and a `#[cfg(test)]` `StubLimiter` (`buzz-relay/src/admission.rs:85`) |
| Test coverage | **Zero tests in this file.** The Lua script, the `count <= limit` boundary, and the TTL-repair branch are unverified anywhere; `buzz-relay`'s tests stub the limiter out entirely (`admission.rs:65-90`) |

### 4. NIP-98 replay protection (`nip98_replay.rs`)

| Property | Finding |
|---|---|
| Atomicity | `SET key 1 NX EX ttl` — single round-trip set-if-absent (`nip98_replay.rs:68-78`); freshness proven by Redis returning `OK` only on first claim (`:20-23`) |
| Replay outcome | `nil` → `Ok(false)` so the caller rejects (`:87`); tested `:127-142` |
| TTL clamping | `ttl_secs.clamp(DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS)` (`:47`). Both directions matter and both are tested: sub-floor is lifted (`:164-177`), and `u64::MAX` is pushed down so Redis cannot reject the `EX` arg and turn every request into an error (`:179-200`, rationale `:184-190`) |
| Unexpected reply | Logged at `error` and returned as `Err`, never treated as a claim (`:88-98`) — closes the "any non-nil means success" foot-gun |
| Failure mode | `Err` on pool or command failure with explicit "caller MUST fail closed" contract, restated in the log payloads (`:53-57`, `:74-79`) |
| Residual | Correctness depends on the caller honouring fail-closed; the crate cannot enforce it. Redis eviction under `maxmemory` pressure would silently shorten the seen-set window — no `noeviction` requirement is asserted or documented |

### 5. Secret and PII handling in logs

| Site | Content | Assessment |
|---|---|---|
| `rate_limiter.rs:58` | `warn!(key = %key, ...)` — the key embeds the full `pubkey_hex` | Pubkeys are public identifiers, not secrets; still a per-user identifier in logs |
| `nip98_replay.rs:53-56`, `:74-78`, `:91-96` | `scope` (contains community id) and Redis error strings | Redis errors can include connection strings; the crate does not scrub them before logging |
| `subscriber.rs:143`, `:154` | Channel name and deserialization error | Channel name contains community + channel UUIDs |
| Event payloads | Never logged | Good — no `warn!("{payload}")` anywhere; failures log only the error (`subscriber.rs:154`) |
| `redis_url` | Stored in `PubSubManager` (`lib.rs:105`) and passed to each reconnect | **Never logged.** No `Debug` derive on `PubSubManager` (`lib.rs:100`), so a credential-bearing URL cannot leak through a struct dump. `PubSubConfig` *does* derive `Debug` (`lib.rs:72`) and holds `redis_url`, so debug-printing a config would expose credentials — no such site exists today |

### 6. Denial-of-service considerations

| Vector | Status |
|---|---|
| Broadcast lag | `broadcast::channel(4096)` (`lib.rs:126-128`); a slow WS receiver gets `RecvError::Lagged` mapped to `PubSubError::BroadcastLagged` (`error.rs:22`, `:33-34`) rather than blocking producers. Capacity is hardcoded, not configurable |
| Unbounded topic growth | `desired_topics` (`subscriber.rs:21`) grows with distinct subscribed topics; entries are removed at refcount zero (`lib.rs:227`), so growth is bounded by concurrent live subscriptions. No explicit cap on topics per connection is enforced here |
| Subscription churn | Debounced unsubscribe (default 500 ms, `lib.rs:82`) with refcount re-check (`subscriber.rs:123-130`) damps thrash |
| Per-release task spawn | `release_topic` spawns a fresh `tokio::spawn` for every last-release (`lib.rs:236-243`); a client rapidly toggling subscriptions creates one short-lived task per toggle |
| Reconnect hot loop | A Redis that accepts connections then immediately closes the stream cleanly resets backoff to 1 s each cycle (`subscriber.rs:57-63`, `cache_invalidation.rs:105-111`, `conn_control.rs:95-101`), producing a sustained 1 s reconnect loop with no escalation |
| Event size | Whole Nostr events are relayed as JSON (`publisher.rs:31`); the crate imposes no size cap of its own |

### 7. Positive findings

- Zero `unsafe` (`#![deny(unsafe_code)]` `lib.rs:1`).
- Zero `unwrap`/`expect` outside `#[cfg(test)]`.
- No SQL, no shell, no filesystem, no HTTP — the attack surface is Redis-only.
- No path where a caller can supply a community id directly.
- Strict deny-by-default channel-name parsing with explicit negative tests.
- Both `buzz-auth` seams are fail-closed by contract, and the replay guard restates
  that obligation in its telemetry.
- Author-private reminder routing is explicitly reasoned about rather than assumed
  safe, with the real enforcement point named (`lib.rs:305-320`).
