## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. What an UNAUTHENTICATED socket can do

Reached: TCP connect + WS upgrade to a host that resolves to a community (`router.rs:286-315`). Budget: one `conn_semaphore` permit, held up to **5 s** (`connection.rs:27`, `:228-252`), and up to `max_frame_bytes` (512 KiB default) per frame with **no** frame-rate limit.

| Action | Allowed? | Site |
|---|---|---|
| Receive the AUTH challenge | yes | `connection.rs:182-186` |
| Send `AUTH` (any number of times until one succeeds or one fails) | yes | `connection.rs:503-509` |
| Send `EVENT` | rejected `OK false "auth-required: not authenticated"` | `event.rs:621-628` |
| Send `REQ` | rejected `NOTICE` + `CLOSED` | `req.rs:76-85` |
| Send `COUNT` | rejected `CLOSED` | `count.rs:41-47` |
| Send `CLOSE` | **fully executed** — `conn.subscriptions.remove`, `sub_registry.remove_subscription`, conditional `release_topic`, `CLOSED ""` reply | `connection.rs:581-583`; `close.rs:12-30` |
| Send arbitrary malformed JSON | parsed, rejected with a `NOTICE`, socket stays open, **unmetered** | `connection.rs:490-496` |
| Send oversized frames | one `NOTICE` then disconnect | `connection.rs:421-434`, `:440-452` |
| Send `Ping` | Pong'd through the priority control channel | `connection.rs:464-472` |
| Consume `handler_semaphore` permits | **no** — permits are taken after admission, and all three permit-taking branches require nothing, but the handlers themselves reject on auth *after* taking the permit | `connection.rs:513-521` then `event.rs:621` |
| Consume admission quota | **no** — admission short-circuits `true` for non-`Authenticated` | `connection.rs:604-609` |
| Read any event | **no** — every delivery path requires a registered subscription, which requires REQ, which requires auth | `req.rs:76-85` |
| Receive a fan-out event | **no** — `pubkey_for_conn` returns `None`, and both the author-only fence (`event.rs:150-153`) and the private-channel fence (`event.rs:184-186`) drop such recipients. Open/channel-less events would pass, but the connection holds no subscription to match | `event.rs:135-195` |

**Net exposure of an unauthenticated socket:** it can (a) hold a connection slot for 5 s, (b) burn CPU on `serde_json` parsing of up to 512 KiB frames at line rate, and (c) obtain `handler_semaphore` permits transiently. It cannot read data, write data, or affect other tenants' state.

Note that the `handler_semaphore` permit for an unauthenticated EVENT/REQ/COUNT **is** acquired before the auth check runs (`connection.rs:513` → `event.rs:611`), so an unauthenticated client can transiently occupy up to `max_concurrent_handlers` (1024) permits within its 5 s window. The window bounds it.

---

#### 2. Authorization decision points — write path

In order, all in `handle_event` unless noted:

| # | Decision | Site | Fails |
|---|---|---|---|
| W1 | `AuthState::Authenticated`? | `event.rs:611-631` | closed |
| W2 | `event.pubkey == auth.pubkey` (kind 1059 exempt) | `event.rs:636-645` | closed |
| W3 | kind ≠ 22242 | `event.rs:647-655` | closed |
| W4 | observer frames: `MessagesWrite` scope | `event.rs:658-666` | **permissive when `scopes == []`** |
| W5 | observer frames: NIP-44 content, tag arity, direction derivation, ±300 s freshness | `event.rs:949-958`, `:1071-1109` | closed |
| W6 | observer frames: caller is the agent's proven owner (session fast-path, else 5-min cache, else DB) | `event.rs:986-1030` | closed; DB error denies |
| W7 | observer frames: 100/s per `(community, agent)` telemetry cap | `event.rs:1032-1044` | closed |
| W8 | ephemeral kinds: `MessagesWrite` scope | `event.rs:675-684` | **permissive when `scopes == []`** |
| W9 | ephemeral: Schnorr signature + id hash on `spawn_blocking` | `event.rs:748-768` | closed |
| W10 | ephemeral with `#h`: channel membership | `event.rs:808-820` | closed |
| W11 | persistent: everything inside `ingest_event` (per-kind scope allowlist, verification, ±900 s drift, 256 KiB content cap, membership, relay-only kinds, ban) | `event.rs:705`; `ingest.rs:1367+` | closed |

Pre-dispatch fences that also gate writes: WS admission (`connection.rs:498-500`), `handler_semaphore` (`:513-521`), frame size (`:421`), and the connection-level ban seam at auth (`auth.rs:105-183`).

---

#### 3. Authorization decision points — read path

##### REQ (`req.rs:44-418`)

| # | Decision | Site |
|---|---|---|
| R1 | `AuthState::Authenticated` | `req.rs:50-87` |
| R2 | `MessagesRead` scope (permissive when `scopes == []`) | `req.rs:54-61` |
| R3 | ≤1024 subscriptions for a new sub_id | `req.rs:65-72` |
| R4 | accessible-channel set resolved (10 s cache) | `req.rs:89-104` |
| R5 | narrowed by scoped-token `channel_ids` | `req.rs:105-107` |
| R6 | subscription-level `#h` channel access, with a DB-confirmed repair bounded by R5 | `req.rs:130-174` |
| R7 | **p-gate** — only when `channel_id.is_none()` | `req.rs:182-190` |
| R8 | engram gate (kind 30174) — only when `channel_id.is_none()` | `req.rs:191-197` |
| R9 | author-only filter gate — only when `channel_id.is_none()` | `req.rs:198-204` |
| R10 | per-filter SQL scope: single `#h` → `channel_id`; global → `channel_ids = accessible` | `req.rs:265-283`, `:993-1002` |
| R11 | per-row: `filters_match` for *that* filter | `req.rs:333-336` |
| R12 | per-row: `stored.channel_id ∈ accessible_channels` | `req.rs:338-342` |
| R13 | per-row: `reader_authorized_for_event` (kinds 30622 / 44200 owner-only) | `req.rs:310-313` |
| R14 | per-row: `is_author_only_event` → skip if not the author | `req.rs:315-319` |

Search REQ adds the same R11–R14 chain per hit (`req.rs:685-712`) plus `#h`∩accessible intersection (`req.rs:568-583`) and `ChannelScope` construction (`req.rs:483-501`).

##### COUNT (`count.rs:29-281`)

R1 (`:38-50`), **no R2**, R4/R5 (`:77-96`), p-gate/engram/author-only **unconditionally** (`:52-75`), per-filter `#h` access with the same repair (`:114-152`), then either the exact SQL path or the bounded fallback with per-row R11/R13/R14 (`:186-201`, `:256-271`).

##### Fan-out (`event.rs:115-195`, `:436-472`)

| # | Fence | Site |
|---|---|---|
| F1 | receiver-side tenant label: `community_for_conn(conn) == Some(event_community)` | `event.rs:126-133` |
| F2 | author-only kinds → only the author's connection (unauthenticated fails closed) | `event.rs:135-157` |
| F3 | channel-less → pass through | `event.rs:159-160` |
| F4 | private channel → per-recipient `is_member_cached`; visibility error empties the list | `event.rs:161-195` |
| F5 | kinds 30622 / 44200 → recipient pubkey must equal the event's `#p` — **only in `dispatch_persistent_event_inner`** | `event.rs:436-472` |

---

#### 4. The p-gate, precisely

`p_gated_filters_authorized` (`req.rs:1042-1074`) returns `true` only if **every** filter satisfies:

```
can_match = filter.kinds.is_none_or(|ks| ks.iter().any(|k| P_GATED_KINDS.contains(k)))
if !can_match                                            → OK
if !explicitly_names(30622|44200) && ids non-empty       → OK          (ids exemption)
else require: #p non-empty AND every #p value == authed_pubkey_hex
```

`P_GATED_KINDS` = `{24200, member-added, member-removed, 1059, 30622, 44200}` (`buzz-core/src/kind.rs:146-156`).

Because `kinds: None` makes `can_match` true, **a filter that omits `kinds` is gated** — this is the mechanism behind AGENTS.md's "relay queries must specify `kinds`, or you get a p-gate 403". Two corrections to that framing:

1. The **403 is HTTP-only**. On HTTP `/query` and `/count` the failure is `StatusCode::FORBIDDEN` with `"restricted: p-gated kinds require #p tag matching your pubkey"` (`api/bridge.rs:981-985`, `:1404-1408`). On WS REQ it is `CLOSED "restricted: p-gated events require #p matching your pubkey"` (`req.rs:184-189`); on WS COUNT it is `CLOSED` with the HTTP wording (`count.rs:55-60`).
2. Omitting `kinds` is not itself fatal — supplying a non-empty `ids` also clears the gate (`req.rs:1066-1068`), and any filter whose named kinds contain no p-gated kind clears it outright.

Comparison of the `#p` value uses `==` on hex strings (`req.rs:1070-1073`) while the engram and author-only gates use `eq_ignore_ascii_case` (`req.rs:1123-1126`, `:1216-1219`). Since `PublicKey::to_hex()` is lowercase and `hex::encode` is lowercase, an uppercase client-supplied `#p` fails the p-gate but would pass an equivalent engram check. Inconsistent, and it fails **closed** for the p-gate, so it is a usability wart rather than a hole.

##### Finding S1 (HIGH) — the p-gate does not run for channel-scoped REQ

`req.rs:182` wraps all three sensitive-kind gates in `if channel_id.is_none()`. The in-code justification (`req.rs:179-181`) is: *"channel-scoped subs can never receive globally-stored events because of the `fan_out()` invariant in subscription.rs."*

That justification covers **live fan-out** correctly (`subscription.rs:265-330`). It does **not** cover **historical delivery**, which runs unconditionally at `req.rs:261-406`, nor does it hold for p-gated kinds that are genuinely channel-scoped. Verified:

- `KIND_GIFT_WRAP` (1059) is **not** in `is_global_only_kind` (`ingest.rs:379-436`), so a gift wrap carrying an `h` tag is stored with `channel_id = Some(ch)`.
- `reader_authorized_for_event` gates only 30622 and 44200 (`buzz-core/src/filter.rs:25-33`) — **not** 1059, not 24200, not member notifications. So R13 does not backstop it.
- `is_author_only_event` covers only reminders and push leases (`buzz-core/src/kind.rs:120`). Not a backstop either.

Result: an authenticated member of channel `X` can send `["REQ","s",{"#h":["X"],"kinds":[1059]}]` and the p-gate is skipped; the historical query is `channel_id = X` with no `#p` predicate, and R11–R14 do not exclude another member's gift wraps. Every other member's channel-scoped gift wraps in `X` become readable ciphertext plus full metadata envelope (sender, recipient `#p`, timestamps).

The HTTP bridge does **not** have this hole: `api/bridge.rs:981` applies the p-gate to all filters unconditionally. The WS path is the weaker of the two.

*What I verified:* the gate condition, the kind-set memberships, the absence of a result-level backstop for 1059, and that `KIND_GIFT_WRAP` is absent from `is_global_only_kind`. *What I did not verify:* whether the shipped clients actually attach `h` tags to gift wraps (that determines whether exploitable rows exist today, not whether the code path is open).

##### Finding S2 (MEDIUM) — `#p` gating relies on `Filter::generic_tags`, not on tag semantics

Both `p_gated_filters_authorized` (`req.rs:1043`) and `result_gated_count_safe_for_pushdown` (`req.rs:1177-1180`) read `filter.generic_tags[SingleLetterTag::lowercase(P)]`. `filter_match_one` matches the same tag key by `tag.kind().to_string() == "p"` (`buzz-core/src/filter.rs:70-78`). These agree today. But the *fan-out* index key is built from `event_p_tag_values`, which also compares `tag.kind().to_string() != "p"` (`subscription.rs:526`) with **no normalisation of the value**, so an event tagging `#p` with uppercase hex lands under a key that no lowercase-hex subscription will ever match — a silent non-delivery, not a leak.

---

#### 5. Result-gated kinds: three hardcoded copies of one list

`RESULT_GATED_KINDS = [KIND_DM_VISIBILITY, KIND_AGENT_TURN_METRIC]` (`buzz-core/src/kind.rs:129`).

| Site | Uses the constant? |
|---|---|
| `req.rs:1154-1159` `filter_can_match_result_gated_kinds` | **yes** — `RESULT_GATED_KINDS.contains(...)` |
| `buzz-core/src/filter.rs:25-27` `reader_authorized_for_event` | **no** — `if kind != KIND_DM_VISIBILITY && kind != KIND_AGENT_TURN_METRIC` |
| `event.rs:438-439` `owner_only_kind` in `dispatch_persistent_event_inner` | **no** — `kind_u32 == KIND_DM_VISIBILITY \|\| kind_u32 == KIND_AGENT_TURN_METRIC` |
| `req.rs:1063-1066` ids-exemption carve-out | **no** — same two-way `==` |

So the relay **compounds** the `buzz-core/src/filter.rs:25` drift with two more copies. Adding a third result-gated kind to `RESULT_GATED_KINDS` would silently gate COUNT's pushdown decision while leaving the actual result-level check (`filter.rs`), the live fan-out owner fence (`event.rs`), and the ids-exemption carve-out (`req.rs`) wide open — a fail-**open** divergence. Severity: MEDIUM now, HIGH the moment a kind is added.

##### Finding S3 (MEDIUM) — the owner fence is missing on the cross-node path

F5 (`event.rs:436-472`) is implemented **only** inside `dispatch_persistent_event_inner`. `fan_out_pubsub_event` (`event.rs:259-309`) and `fan_out_event_to_local_subscribers` (`event.rs:218-256`) apply F1–F4 only. Since neither 30622 nor 44200 is in `AUTHOR_ONLY_KINDS`, F2 does not substitute. A kind 30622 / 44200 event that reaches a *second* pod over Redis is fanned out to any subscription whose filters match — including a kindless `ids:[…]` subscription, exactly the case F5's own comment says it exists to close (`event.rs:433-436`). The doc comment at `event.rs:212-217` asserts the two exception paths are "equivalent to this helper plus their own extra step"; for `fan_out_pubsub_event` that is **not** true — it has one fewer step.

Mitigating: on the origin pod the event is echo-suppressed on arrival (`event.rs:277-281`), so single-pod deployments are unaffected. This bites only multi-pod.

---

#### 6. Rate-limit enforcement placement

| Element | Site |
|---|---|
| Invocation, before dispatch, for every frame | `connection.rs:498-500` |
| Implementation | `connection.rs:594-653` |
| Metered message set — `EVENT`, `REQ`, `COUNT` only | `connection.rs:599-602` |
| **Pre-auth bypass** — non-`Authenticated` returns `true` immediately | `connection.rs:604-609` |
| Limiter | `state.admission_rate_limiter: Arc<RedisRateLimiter>` (`state.rs:584`, constructed `:712`) |
| Fail-closed on limiter error | `admission.rs:34-40` → `connection.rs:670-677` |
| Community-scoped Redis key | `buzz-auth/src/rate_limit.rs:153-156` |

Correction to a prior batch's note: the pre-auth bypass is `connection.rs:604-609` (the `match &*auth { … _ => return true }`), and the metered-set filter is `:599-602`.

The rate limiter is **pubkey-keyed by design**, which is exactly why it cannot cover pre-auth traffic. The intended companion control for that window — `LimitType::IpConnections` / `RateLimiter::check_ip_connection` — is implemented (`buzz-pubsub/src/rate_limiter.rs:112`, trait `buzz-auth/src/rate_limit.rs:188`) and **never called from anywhere in the relay**. Confirmed by a workspace-wide grep: the only occurrences are the trait declaration, the two impls, and a test stub (`admission.rs:85`).

**Delta vs `ARCHITECTURE.md`.** §9 item 2 (`ARCHITECTURE.md:823`) claims "No rate limiting implementation … `RateLimitConfig` defines 4 tiers … but none are enforced." Two of the tiers (`human_messages_per_min`, `agent_standard_messages_per_min`) and `human_ws_events_per_sec` **are** enforced on this path (`connection.rs:612-650`). The elevated and platform tiers are not referenced by this group. The doc row should be rewritten as "IP-connection limiting not wired; elevated/platform agent tiers unused".

---

#### 7. DoS surfaces

| # | Surface | Bound | Gap |
|---|---|---|---|
| D1 | Connection count | `conn_semaphore` = `max_connections`, default **10 000** (`config.rs:449-452`; `connection.rs:149`) | **no per-IP cap** — one host can take all 10 000 slots; each costs a 1000-slot mpsc, an 8-slot ctrl channel, a `HashMap`, 3 tasks. At 5 s per unauthenticated slot the sustained cost is 2 000 slot-acquisitions/s to hold the relay full |
| D2 | Pre-auth frame flood | none | admission bypassed (`connection.rs:604-609`); each frame costs a `serde_json::from_str` over up to 512 KiB and, for EVENT/REQ/COUNT, a `handler_semaphore` permit + task spawn |
| D3 | `CLOSE` flood (pre- or post-auth) | none | never metered (`connection.rs:599-602`), always answered (`close.rs:27`), awaited inline — self-throttling per socket but free work at scale |
| D4 | Malformed-JSON flood | none | `NOTICE` per frame, socket stays open (`connection.rs:490-496`) |
| D5 | Subscription memory | 1024 subs × 10 filters × unbounded per-filter arrays | `MAX_SUBSCRIPTIONS` (`req.rs:26`) and `MAX_FILTERS_PER_REQ` (`protocol.rs:14`) bound the counts, but **nothing bounds a filter's internal cardinality** — no cap on `ids`, `authors`, `kinds`, or generic-tag value counts. A REQ with 10 filters × 100 000 `authors` each is accepted, stored (`subscription.rs:81`, and again at `req.rs:238`), and re-evaluated linearly on **every** matching event (`filter_match_one`, `buzz-core/src/filter.rs:41-46`) |
| D6 | Index fan-out amplification | one index entry per (kind × filter) | `register_scoped` inserts one entry per kind in the union (`subscription.rs:104-112`). 10 filters × 1000 kinds = 10 000 index entries **per subscription**, ×1024 subs = ~10 M entries on one connection. Each carries a cloned `sub_id` up to 256 B → ~2.6 GB of `String` per connection, from a single client, entirely within the advertised caps |
| D7 | Removal cost | `Vec::retain` per index bucket | O(bucket) per kind on removal (`subscription.rs:407`, `:429`); D6's shape makes disconnect cleanup O(10 M) |
| D8 | Historical read cost | 2000 rows × 10 filters, concurrency 4 | `req.rs:885`, `:314`. No global per-connection or per-pubkey read budget beyond the 5 s WsEvents window |
| D9 | Search cost | 10 pages × 100 hits × 10 filters, plus one `get_events_by_ids` per page | `req.rs:421`, `:589`, `:594` — up to 100 FTS queries and 100 batch fetches per REQ |
| D10 | COUNT cost | 5001-row candidate scan per non-pushable filter × 10 filters | `req.rs:753`; `count.rs:173`, `:243` — up to ~50 000 rows scanned per COUNT, and **no `MessagesRead` scope required** (S4) |
| D11 | Conformance side-queries | 1 extra `communities_of_channels` per filter + per search page | `req.rs:355`, `:648` — issued even with the production `NoopTracer` (`state.rs:798`) |
| D12 | `observer_rate_limiter` growth | none | plain `DashMap` with no capacity and no eviction (`state.rs:589`, ctor `:773`; only `entry().or_insert()` at `event.rs:897-901`). One permanent entry per distinct `(community, agent pubkey)` observed. Any authenticated agent-owner pair can seed entries at 100/s |
| D13 | Spawned-task fan-out | `handler_semaphore` (1024) for message handlers | but `dispatch_persistent_event` (`event.rs:352`) and the workflow trigger (`event.rs:512`) spawn **unbounded** detached tasks per accepted event, outside any semaphore |
| D14 | Slow-client memory | 1000-message send buffer × 15 strikes | `config.rs:459-462`, `:470-473`. At 512 KiB per frame the theoretical per-connection buffer is ~500 MB; in practice bounded by event size (256 KiB ingest cap, `ingest.rs:1484`) |

D5+D6 together are the sharpest: they are reachable by a single authenticated member with no elevated permissions, entirely inside every advertised limit, and there is no test covering filter-cardinality limits.

---

#### 8. Additional findings

##### S4 (MEDIUM) — COUNT skips the read-scope check

`handle_req` refuses a session without `MessagesRead` (`req.rs:54-61`). `handle_count` has no equivalent branch (`count.rs:38-50`). A scoped session holding, say, only `ChannelsWrite` can therefore issue `COUNT` and learn exact event counts for every channel it is a member of — the count is an existence oracle that the read-scope gate exists to deny. The `#h`/token narrowing (`count.rs:94-96`) limits *which* channels, not *whether* counting is permitted at all.

##### S5 (MEDIUM) — COUNT leaks raw database error text

`count.rs:167`, `:203`, `:238`, `:273` all send `CLOSED "error: {e}"` with the `buzz_db` error rendered verbatim. `handle_event` explicitly sanitises the equivalent case with a comment ("don't leak DB/system details over WS", `event.rs:726-727`). A crafted filter that provokes a Postgres error returns schema/constraint/type detail to any authenticated member.

##### S6 (LOW) — `Failed` is permanently sticky, including for transient causes

`auth.rs:172`, `:206`, `:230`, `:287` all set `AuthState::Failed`, which is terminal (`auth.rs:58-66`). The ban seam goes to visible lengths to avoid *mislabelling* a transient DB error as a ban (`auth.rs:98-117`) — but it still writes `Failed` for the DB-error case (`auth.rs:161-165`), so a Postgres blip permanently kills that socket's ability to authenticate. The client must reconnect. Correct as a fail-closed choice; worth noting because the surrounding comment implies the distinction preserves more than it does.

##### S7 (LOW) — `remote_addr` is captured but never used for security

`ConnectionState.remote_addr` (`connection.rs:61`) is read only by two `info!` calls (`connection.rs:183`, `:289`). No per-IP accounting, no abuse correlation, no ban-by-IP. Combined with D1, the relay has the data for a per-IP cap and does not use it. Also note the address comes from `ConnectInfo` with a `0.0.0.0:0` fallback (`router.rs:238-243`) and **no** `X-Forwarded-For` handling, so behind a proxy every connection would appear to share one address — a per-IP cap would need proxy-header trust configured first.

##### S8 (LOW) — binary frames with invalid UTF-8 are silently dropped

`connection.rs:457-459` decodes binary frames and, on failure, falls through with no `else` — no NOTICE, no counter, no disconnect. A client sending malformed binary gets total silence, which is indistinguishable from a hung relay.

##### S9 (INFO) — the production `expect()` on the fan-out hot path

`event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")`. The invariant does hold at all three call sites, because the cache is built from the same iterator that drives the send (`event.rs:245-252`, `:301-306`, `:470-472`). But it is the only `expect()` outside `#[cfg(test)]` in these 8 files, it sits on the delivery path, and a panic there would poison a spawned dispatch task. `AGENTS.md` prohibits new `unwrap()`/`expect()` in production paths.

---

#### 9. Multi-tenant isolation posture

Isolation is layered and each layer is independently verified in code:

| Layer | Mechanism | Site |
|---|---|---|
| Bind before read | community resolved from `Host` pre-upgrade; unmapped → generic 404 | `router.rs:286-300` |
| Subscription indexes | every index key is prefixed with `CommunityId` | `subscription.rs:49-58` |
| Fan-out receiver fence | `community_for_conn(conn) == Some(event_community)` | `event.rs:126-133` |
| Redis topic keys | `EventTopicKey::from_context(ctx, topic)` | `buzz-pubsub/src/lib.rs:193`, `:216` |
| Cross-node community label | taken from the **parsed Redis channel**, not the event body | `event.rs:268` |
| Local-echo dedup key | `(community_id, event_id)` | `event.rs:275`; `state.rs:530-540` |
| Every cache key | `(CommunityId, …)` | `state.rs:740-787` |
| Cache invalidation | predicate-scoped to one community, with a documented over-invalidate fallback | `state.rs:882-899`, `:753-773` |
| Rate-limit keys | `buzz:{community}:ratelimit:{pubkey}:{suffix}` | `buzz-auth/src/rate_limit.rs:153-156` |
| Observer limiter + owner cache | `(CommunityId, …)` keys | `state.rs:589`, `:607` |
| Ban / disconnect | tenant-fenced: a ban in A never closes B's socket | `state.rs:261-282`, `:296-325`; test `:1662-1704` |
| Presence clear | only when no other connection for that pubkey **in that community** | `connection.rs:274-287` |
| Workflow trigger | community passed explicitly because `StoredEvent` doesn't carry it | `event.rs:505-514` |
| Audit chain | `community_id` from the `TenantContext`, not the event | `event.rs:559`; test `:1714-1858` |

The one deliberate non-fence: `ConnectionManager::drain_all` cancels every connection regardless of tenant (`state.rs:352-366`), correct for process shutdown and asserted as such at `state.rs:1829`.

Cross-tenant isolation is the best-covered property in this group — the `redteam` module (`event.rs:2327-2458`) plus `subscription.rs:1379-1441` and `event.rs:1553-1600` pin it from three directions.
