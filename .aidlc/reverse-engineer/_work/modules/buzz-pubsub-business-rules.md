## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Business Rules

Rules extracted from all 10 `.rs` files. Each row is an enforced behaviour with its
enforcement site; "Tested" cites the asserting test.

---

### 1. Tenant scoping rules

| # | Rule | Enforcement | Tested |
|---|---|---|---|
| BR-PS-01 | Every event topic key embeds the server-resolved community id; the same channel UUID in two communities is two distinct Redis channels | `topic.rs:45-50` | `topic.rs:138-148` |
| BR-PS-02 | The community id comes only from `TenantContext::community()`, never from a caller argument | `EventTopicKey::from_context` `topic.rs:35-40` | — |
| BR-PS-03 | Presence keys are community-scoped by the same rule | `presence.rs:19-25` | `presence.rs:124-134` |
| BR-PS-04 | Cache-invalidation channels are community-scoped | `cache_invalidation.rs:30-35` | `cache_invalidation.rs:186-199` |
| BR-PS-05 | Conn-control channels are community-scoped | `conn_control.rs:33-35` | `conn_control.rs:172-181` |
| BR-PS-06 | NIP-98 replay seen-sets are per-community: the same event id is a *first* claim in a second community | `nip98_replay.rs:81` (scope-keyed) | `nip98_replay.rs:144-162` |
| BR-PS-07 | IP-keyed rate limits are deliberately **not** community-scoped (operator-global) | `rate_limiter.rs:118` → `buzz-auth/src/rate_limit.rs:213`; rationale `buzz-auth/src/rate_limit.rs:158` | `buzz-auth/src/rate_limit.rs:309-313` |
| BR-PS-08 | Releasing a topic in community A must not tear down the identical channel id in community B | refcount keyed by `EventTopicKey` not channel id — `lib.rs:193`, `:216`, `subscriber.rs:21` | `lib.rs:510-590` (explicitly written to catch the channel-id-only bug, comment `lib.rs:556-557`) |

### 2. Topic-key parsing rules (`topic.rs:53-99`)

| # | Rule | Line |
|---|---|---|
| BR-PS-09 | Prefix must be exactly `buzz` | `:58-63` |
| BR-PS-10 | Segment 2 must parse as a UUID | `:68-71` |
| BR-PS-11 | `global` topics must have exactly 3 segments — a 4th is rejected | `:78-82` |
| BR-PS-12 | `channel` topics must have exactly 4 segments; the 4th must parse as a UUID; a 5th is rejected | `:84-94` |
| BR-PS-13 | Any other scope word (including `presence`) is rejected | `:95` |

All 13 negative cases asserted at `topic.rs:179-195`, including
`buzz:{uuid}:presence:abc` (`:187`).

### 3. Subscription lifecycle rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-14 | The local desired-refcount map is the single source of truth for what should be subscribed; the Redis connection's `active_topics` set is derived | doc `lib.rs:108`, `subscriber.rs:36-38`; rebuild `subscriber.rs:90-101` |
| BR-PS-15 | Only the **first** retain of a topic emits `SUBSCRIBE`; later retains just increment | `lib.rs:194-201` (`was_zero` gate) |
| BR-PS-16 | Only the **last** release schedules an unsubscribe; the entry is removed from the map at zero | `lib.rs:217-231` |
| BR-PS-17 | Unsubscribe is debounced by `unsubscribe_debounce` (default 500 ms) and re-checks the refcount at execution time, so a re-retain during the window cancels it | schedule `lib.rs:233-244`; re-check `subscriber.rs:123-130` |
| BR-PS-18 | Releasing a topic that was never retained is a logged no-op, not an underflow | `lib.rs:219-222` (`warn` + early return) — prevents `usize` underflow panic |
| BR-PS-19 | On reconnect, subscriptions are re-established from the refcount snapshot **before** any message is processed | `subscriber.rs:90-101` precedes the `loop`/`select!` at `:109` |
| BR-PS-20 | Duplicate `SUBSCRIBE` for an already-active topic is suppressed | `active_topics.insert(...)` guard `subscriber.rs:118-120` |
| BR-PS-21 | The fan-out loop can only be started once; a second `run_subscriber` logs an error and returns | `subscription_rx` take-once `lib.rs:149-152` |

### 4. Event fan-out rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-22 | Published payload is the complete signed Nostr event JSON, not a delta | `publisher.rs:31` (`event.as_json()`) |
| BR-PS-23 | A message whose channel name fails to parse is dropped with a warning, never fanned out | `subscriber.rs:141-148` |
| BR-PS-24 | A payload that fails `Event::from_json` is dropped with a warning; the loop continues | `subscriber.rs:150-157` |
| BR-PS-25 | A payload-retrieval failure is dropped with a warning; the loop continues | `subscriber.rs:132-139` |
| BR-PS-26 | Zero local receivers is not an error — the message is dropped at `trace` level | `subscriber.rs:165-167` |
| BR-PS-27 | The community id delivered to local receivers is taken from the **parsed Redis channel**, not from the event body | `subscriber.rs:159-163` |
| BR-PS-28 | The topic key is a routing label, **not** an authorization boundary; per-recipient access is re-checked by the relay's `filter_fanout_by_access` on both the in-process and Redis paths | doc `topic.rs:3-6`, `lib.rs:305-320`; relay side `buzz-relay/src/handlers/event.rs:135`, `main.rs:696` |
| BR-PS-29 | Author-private reminders (kind 30300) are deliberately *not* isolated by Redis routing; author-only delivery is enforced downstream | `lib.rs:307-319` |

### 5. Presence rules

| # | Rule | Value / enforcement |
|---|---|---|
| BR-PS-30 | Presence TTL is 90 s — 3× the 30 s heartbeat so one missed beat doesn't flap | `presence.rs:16`, rationale `presence.rs:4-6`, `:15` |
| BR-PS-31 | Presence is written with `SET key status EX 90` (TTL refreshed on every heartbeat) | `presence.rs:36-43` |
| BR-PS-32 | Clean disconnect deletes the key immediately rather than waiting for TTL | `presence.rs:52-56`; callers `buzz-relay/src/connection.rs:280`, `handlers/event.rs:793` |
| BR-PS-33 | Bulk lookup short-circuits on an empty input list without touching Redis | `presence.rs:79-81` |
| BR-PS-34 | Bulk lookup returns only keys that exist — absent pubkeys are omitted rather than mapped to a default | `filter_map` `presence.rs:89-93` | 
| BR-PS-35 | Status is an arbitrary caller-supplied string; the crate defines no enum and validates nothing | `presence.rs:28-33` (`status: &str` written verbatim) |

BR-PS-34 tested at `presence.rs:159-184` (`pk3` asserted absent, `:180`).
BR-PS-30 tested indirectly by TTL-range assertions at `presence.rs:186-207` and
`lib.rs:490-499`.

### 6. Rate-limit rules (`rate_limiter.rs`)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-36 | Counter increment and TTL creation are atomic via a single Lua script — no crash window where a key exists without expiry | `RATE_LIMIT_SCRIPT` `rate_limiter.rs:24-31`; `EXPIRE` only when `count == 1` (`:26-28`) |
| BR-PS-37 | The window is **fixed**, accepting up to 2× burst at boundaries | documented `rate_limiter.rs:9-10`; also flagged upstream `buzz-relay/src/admission.rs:4-7` |
| BR-PS-38 | A key found with no TTL (`ttl < 0`) is repaired with a fresh `EXPIRE` and a warning; the reported reset becomes the full window | `rate_limiter.rs:57-70` |
| BR-PS-39 | Allow boundary is inclusive: `count <= limit` allows | `rate_limiter.rs:74-78` |
| BR-PS-40 | Redis failure surfaces as `AuthError::Internal`, and the relay treats that as **deny** (fail-closed) | mapping `rate_limiter.rs:44-46`, `:52`, `:66`; relay decision `buzz-relay/src/admission.rs:29-33` (`AdmissionError::Unavailable`) |
| BR-PS-41 | WS admission converts a per-second limit into a 5 s fixed window (`limit × 5`) to tolerate desktop startup bursts | `buzz-relay/src/admission.rs:8`, `:35-40`; asserted `ws_admission_budget(10) == (5, 50)` (`admission.rs:101`) |
| BR-PS-42 | Rate limiting applies only to `EVENT`, `REQ`, and `COUNT`; all other WS frames bypass it | `buzz-relay/src/connection.rs:598-601` |
| BR-PS-43 | Unauthenticated connections bypass rate limiting entirely (the limiter is pubkey-keyed) | `buzz-relay/src/connection.rs:602-609` (`_ => return true`) |
| BR-PS-44 | Agents get `agent_standard_messages_per_min`, humans `human_messages_per_min`, on a fixed 60 s window | `buzz-relay/src/connection.rs:628-646` |

### 7. NIP-98 replay rules (`nip98_replay.rs`)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-45 | Freshness is proven by `SET … NX EX` returning `OK` only on the first claim | `nip98_replay.rs:68-78`, `:86-88` |
| BR-PS-46 | An existing key (`nil` reply) yields `Ok(false)` so the caller rejects as replay | `nip98_replay.rs:87` |
| BR-PS-47 | Requested TTL is clamped into `[DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS]` — sub-floor is lifted, above-ceiling is pushed down so a buggy caller cannot send a Redis-invalid `EX` or pin a slot indefinitely | `nip98_replay.rs:47`, rationale `:42-46` |
| BR-PS-48 | Any reply other than `OK`/`nil` is an internal error, logged at `error` — never treated as a successful claim | `nip98_replay.rs:88-98` |
| BR-PS-49 | Redis unavailability returns `Err`, and the contract requires callers to fail **closed** | `nip98_replay.rs:49-58`, `:74-77` (log text states "caller MUST fail closed") |

BR-PS-45/46 tested `nip98_replay.rs:127-142`; BR-PS-47 both directions tested at
`:164-177` (sub-floor) and `:179-200` (`u64::MAX` clamp).

### 8. Cross-pod control-message rules

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-50 | Cache-invalidation messages are pure idempotent key drops — never "evict these subscriptions" payloads | stated invariant `cache_invalidation.rs:10-14`; enum carries only ids (`:58-80`) |
| BR-PS-51 | Each `CacheInvalidation` variant mirrors exactly one relay-local `invalidate_*` operation | `cache_invalidation.rs:53-80` |
| BR-PS-52 | Imperative, non-idempotent actions (disconnects) must live on a **separate** channel from cache drops, precisely to preserve BR-PS-50 | rationale `conn_control.rs:12-18`; distinct suffix `conn_control.rs:26` |
| BR-PS-53 | Both control publishes are fire-and-forget: the DB row (ban) or the next DB read (cache) is the durable backstop, so a dropped publish degrades gracefully | `lib.rs:266-271`, `:285-291`, `conn_control.rs:18-21` |
| BR-PS-54 | An unparseable/unknown control payload is skipped without breaking the subscriber loop | `cache_invalidation.rs:159-165`, `conn_control.rs:150-156`; regression test `conn_control.rs:209-217` |

### 9. Reconnect rules (all three subscribers)

| # | Rule | Enforcement |
|---|---|---|
| BR-PS-55 | Reconnect backoff starts at 1 s and doubles to a 30 s ceiling | `subscriber.rs:16-19`, `:46`, `:71`; `cache_invalidation.rs:91-94`, `:120-122`; `conn_control.rs:81-84`, `:110-112` |
| BR-PS-56 | A *clean* stream end resets backoff to 1 s; only errors let it grow | `subscriber.rs:57-63`, `cache_invalidation.rs:105-111`, `conn_control.rs:95-101` |
| BR-PS-57 | All three subscriber loops are infinite — they never return to the caller | `subscriber.rs:41-76`, `cache_invalidation.rs:100-127`, `conn_control.rs:90-117` |

**Rule count: 57.** Zero rules are enforced by type-state or the compiler; all are
runtime checks or conventions. No rule in this crate performs authorization — the
crate consistently defers that to `buzz-auth`/`buzz-relay` (BR-PS-28).
