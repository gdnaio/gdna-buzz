## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Debt

---

#### 0. Hard counts

| Metric | Value |
|---|---|
| Files / LOC in scope | 8 / 7,590 |
| Files over 1000 lines | 3 — `handlers/event.rs` 2461, `handlers/req.rs` 1946, `subscription.rs` 1562 |
| Inbound wire message types handled | **5** (`protocol.rs:23-42`) |
| Outbound formatters | **7** (`protocol.rs:181-208`) |
| Tests | **106** — `connection.rs` 5, `subscription.rs` 29, `event.rs` 24, `req.rs` 45, `auth.rs` 3, `count.rs` **0**, `close.rs` **0**, `mod.rs` **0** |
| `#[ignore]`d tests | **0** |
| `unsafe` | **0** |
| `unwrap()` / `expect()` outside `#[cfg(test)]` | **1** — `event.rs:88` |
| `TODO` / `FIXME` / `XXX` / `HACK` | **0** |
| Distinct subscription-related limits | 13 (see the data-model aspect); **11 of 13 are hard-coded** |
| Items with zero production callers | **3** (§3) |
| Duplicated helper implementations | 4 sets (§4) |

---

#### 1. CRITICAL

##### D-01 — The WS p-gate does not run for channel-scoped REQ
`req.rs:182` gates the p-gate, engram gate, and author-only gate behind `if channel_id.is_none()`. The in-code justification (`req.rs:179-181`) covers only *live fan-out*, but **historical delivery runs unconditionally** at `req.rs:261-406`.

Verified chain:
- `KIND_GIFT_WRAP` (1059) is p-gated (`buzz-core/src/kind.rs:150`) but is **not** in `is_global_only_kind` (`ingest.rs:379-436`), so a gift wrap with an `h` tag stores with `channel_id = Some(ch)`.
- `reader_authorized_for_event` gates only 30622 / 44200 (`buzz-core/src/filter.rs:25-27`) — no backstop for 1059, 24200, or member notifications.
- `is_author_only_event` covers only reminders + push leases (`buzz-core/src/kind.rs:120`) — no backstop.

So `["REQ","s",{"#h":["<a channel I'm in>"],"kinds":[1059]}]` reaches historical delivery with no `#p` predicate, returning every member's channel-scoped gift wraps. The HTTP bridge applies the gate unconditionally (`api/bridge.rs:981`), so WS is the weaker surface.

**Fix:** move `req.rs:182-205` out of the `channel_id.is_none()` branch. `count.rs:52-75` already does exactly this and is the reference implementation.
*Not verified:* whether shipped clients attach `h` tags to gift wraps — that determines exploitability today, not whether the path is open.

---

#### 2. HIGH

##### D-02 — Result-gated kind list is hardcoded in three more places
`RESULT_GATED_KINDS` (`buzz-core/src/kind.rs:129`) is honoured at `req.rs:1154-1159` and open-coded at:
- `buzz-core/src/filter.rs:25-27` — the actual result-level read gate
- `event.rs:438-439` — the live fan-out owner fence
- `req.rs:1063-1066` — the ids-exemption carve-out

The relay therefore **compounds** the known `filter.rs:25` drift. Adding a third result-gated kind updates only COUNT's pushdown decision and leaves the enforcement points open — a **fail-open** divergence. Fix: make all four sites read `RESULT_GATED_KINDS`, and add a test asserting each site's coverage equals the constant.

##### D-03 — DM-visibility owner fence is missing on the cross-node fan-out path
`event.rs:436-472` implements the owner fence for kinds 30622 / 44200, but only inside `dispatch_persistent_event_inner`. `fan_out_pubsub_event` (`event.rs:259-309`) and `fan_out_event_to_local_subscribers` (`event.rs:218-256`) apply only the four `filter_fanout_by_access` fences, and neither kind is in `AUTHOR_ONLY_KINDS` so `F2` does not substitute. On a multi-pod deployment a kindless `ids:[…]` subscription on pod B receives another user's 30622/44200 event — exactly the case the fence's own comment (`event.rs:433-436`) says it exists to close.

Aggravating: the doc comment at `event.rs:212-217` asserts both exception paths are "equivalent to this helper plus their own extra step". For `fan_out_pubsub_event` that is false — it has one fewer step. Fix: move the owner fence into `filter_fanout_by_access` (where `AUTHOR_ONLY_KINDS` already lives) so all three paths inherit it.

##### D-04 — No per-IP connection limit; the machinery exists and is unwired
`LimitType::IpConnections` (`buzz-auth/src/rate_limit.rs:66`) and `RateLimiter::check_ip_connection` (trait `:188`, Redis impl `buzz-pubsub/src/rate_limiter.rs:112`) have **zero** callers in the relay. Combined with:
- `max_connections` default 10 000 (`config.rs:449-452`),
- admission bypassed pre-auth (`connection.rs:604-609`),
- `remote_addr` captured but used only for logging (`connection.rs:183`, `:289`),

one host can hold every connection slot with unauthenticated sockets, each costing a 1000-slot mpsc plus three tasks, recycled every 5 s (`connection.rs:27`).

Note a prerequisite: `router.rs:238-243` reads `ConnectInfo` with a `0.0.0.0:0` fallback and does **no** `X-Forwarded-For` handling, so behind a proxy every connection shares one apparent address. A per-IP cap needs trusted proxy-header config first.

##### D-05 — No bound on filter internal cardinality; index amplification
`MAX_SUBSCRIPTIONS` = 1024 (`req.rs:26`) and `MAX_FILTERS_PER_REQ` = 10 (`protocol.rs:14`) bound the *counts*, but nothing bounds a single filter's `ids`, `authors`, `kinds`, or generic-tag arrays.

Two amplification vectors, both reachable by one ordinary authenticated member inside every advertised limit:
1. **Match cost** — `filter_match_one` (`buzz-core/src/filter.rs:41-46`) linear-scans `authors` / `ids` on **every** candidate event, for every one of up to 10 240 registered filters.
2. **Index cost** — `register_scoped` inserts one index entry **per kind in the union** (`subscription.rs:104-112`), each carrying a cloned `sub_id` of up to 256 B (`protocol.rs:11`). 10 filters × 1000 kinds × 1024 subs ≈ 10 M entries ≈ 2.6 GB of `String` on one connection. Removal is `Vec::retain` per bucket (`subscription.rs:407`, `:429`), so disconnect cleanup is O(that).

No test covers filter cardinality. Fix: cap per-filter array lengths at parse time in `protocol.rs`, and cap the kind-union size in `register_scoped`.

##### D-06 — `ARCHITECTURE.md` §9 #2 is factually wrong about rate limiting
`ARCHITECTURE.md:823` states "**No rate limiting implementation** … `RateLimitConfig` defines 4 tiers … but none are enforced." Three limits **are** enforced on this path: `human_ws_events_per_sec`, `human_messages_per_min`, `agent_standard_messages_per_min` (`connection.rs:612-650`) against a Redis-backed limiter (`state.rs:584`, ctor `:712`), failing closed (`admission.rs:34-40`).

This is the highest-impact doc defect in the group: an operator reading it would conclude the relay has no throttling and add an external one, or would not investigate a `rate-limited:` NOTICE. Accurate replacement: *"IP-connection limiting is not wired (`check_ip_connection` has no caller); the elevated and platform agent tiers are unreferenced."*

---

#### 3. MEDIUM

##### D-07 — Dead code: three `SubscriptionRegistry` methods with zero production callers
| Item | Site | Only referenced by |
|---|---|---|
| `get_filters` | `subscription.rs:338-343` | `subscription.rs:739`, `:743` (tests) |
| `total_subscriptions` | `subscription.rs:345-348` | `subscription.rs:651`, `:655` (tests) |
| `total_connections` | `subscription.rs:350-353` | `subscription.rs:656` (test) |

Verified by workspace-wide grep. `main.rs:1328-1331` computes its own totals from `per_community_subscriptions()` and `per_community_ws_connections()` — the local variables there are coincidentally named `total_subscriptions` / `total_connections`, which is why a shallow grep looks like a hit. All three are `pub` on a `pub` type, so the compiler does not flag them.

##### D-08 — COUNT skips the read-scope check
`handle_req` refuses a session lacking `MessagesRead` (`req.rs:54-61`); `handle_count` has no equivalent (`count.rs:38-50`). A scoped session with only write scopes can use COUNT as an existence oracle over every channel it belongs to.

##### D-09 — COUNT leaks raw database error text to clients
`count.rs:167`, `:203`, `:238`, `:273` all emit `CLOSED "error: {e}"` with the `buzz_db` error rendered verbatim. `handle_event` sanitises the identical case with an explicit comment (`event.rs:726-727`). Three different postures exist for a DB failure across the group: sanitised (`event.rs`), verbatim (`count.rs`), and silently swallowed into an `EOSE` (`req.rs:320-325`).

##### D-10 — A failed historical query is indistinguishable from an empty result
`req.rs:318-326` sends `EOSE` on a `query_events` error and returns, leaving the subscription registered. The client cannot tell "no matches" from "DB down", and there is no metric for the case. `handle_search_req` behaves the same way for FTS and batch-fetch errors (`req.rs:611-616`, `:646-650`) — both `break` out of pagination and fall through to `EOSE`. Fix: emit `CLOSED "error: …"` (or at minimum a counter) and consider deregistering.

##### D-11 — Delivery abort skips EOSE
`req.rs:398-400` and `:720-722` `return` on a failed `conn.send`, so the client never receives `EOSE` for a subscription that is registered and live. Combined with the 15-strike grace (`connection.rs:100`) the connection survives, so the client waits indefinitely on a promise the relay silently abandoned.

##### D-12 — Search REQ does not replace a same-`sub_id` live subscription (NIP-01 violation)
The search branch returns at `req.rs:233`, **before** `subs.insert` (`:238`) and `register_scoped` (`:241`). NIP-01 requires a REQ with an existing `sub_id` to overwrite it. Here the previous subscription keeps fanning out under an id the client believes was replaced by a one-shot query. Same-shaped consequence: a search REQ never counts against `MAX_SUBSCRIPTIONS` (`req.rs:66` reads `conn.subscriptions`, which search never writes).

##### D-13 — Rate-limited EVENT/COUNT get no protocol-level acknowledgement
`enforce_ws_admission` derives `sub_id` only from `ClientMessage::Req` (`connection.rs:624-627`), and the EVENT `Messages` check passes `None` explicitly (`:647`). So a throttled EVENT gets a `NOTICE` with **no `OK`** — a NIP-01 client keyed on `OK` hangs on that event id. Same for COUNT: no `COUNT` response, only a `NOTICE`. Fix: thread the event id into the admission result so an EVENT rejection is `OK false "rate-limited: …"`.

##### D-14 — `observer_rate_limiter` is unbounded and never evicted
`state.rs:589` is a plain `DashMap`; `event.rs:897-901` only ever `entry().or_insert()`. One permanent entry per distinct `(community, agent pubkey)` for the process lifetime. Every sibling structure on `AppState` is a moka cache with an explicit cap and TTL (`state.rs:734-787`). Fix: convert to `moka` with a TTL matching the 1 s window.

##### D-15 — Conformance side-queries run in production
`req.rs:355` and `:648` issue `db.communities_of_channels(...)` whenever `trace_state.is_some()` — i.e. on every REQ filter and every search page — even though production binds `NoopTracer` (`state.rs:798`) and discards the result. That is one extra Postgres round-trip per filter and per search page on the hot read path, purchasing nothing outside conformance runs. Fix: gate on a tracer-enabled predicate.

##### D-16 — Cross-node fan-out loop is not restartable
`main.rs:837-840`: a `RecvError::Closed` logs an error and `break`s out of the loop, permanently ending cross-node delivery for the process while readiness continues to report ready (`router.rs:353-390` checks only Postgres + Redis reachability, not the broadcast channel). Lag is counted (`main.rs:833-836`) but never repaired — lagged events are silently lost.

##### D-17 — Three identical `topic_for_subscription` copies
Byte-identical private functions at `req.rs:1225-1230`, `close.rs:30-35`, `connection.rs:681-686`. All three must move in lockstep with the `EventTopic` enum. The `retain`/`release` refcount correctness depends on all three agreeing.

##### D-18 — Duplicate `ConnectionManager` pubkey accessors
`pubkey_for_conn` (`state.rs:286-290`) and `pubkey_for` (`state.rs:425-429`) are line-for-line identical. `pubkey_for_conn` is used by `event.rs:146`, `:184` and `side_effects.rs:108`; `pubkey_for` only by `event.rs:460` — inside `dispatch_persistent_event_inner`, i.e. the *same file* uses both names for the same operation.

##### D-19 — Filters are stored twice per subscription
`conn.subscriptions` (`connection.rs:65`, written `req.rs:236-239`) duplicates the filter vectors already in `sub_registry.subs` (`subscription.rs:79-82`). The duplicate's only read is `len()`/`contains_key` for the cap check (`req.rs:66`); `subscriptions_for` (`state.rs:383`) hands the `Arc` to `side_effects.rs:71`, which also does not read the filters. Two `Vec<Filter>` clones per REQ (`req.rs:238`, `:245`) plus one inside `register_scoped` (`subscription.rs:81`) for a value never used as filters. Fix: store a count, or read the cap from the registry.

##### D-20 — `count.rs` and `close.rs` have zero tests
`handlers/count.rs` is 281 LOC of authorization-and-pushdown logic — the p-gate, the token narrowing, the pushdown safety predicate, the 5000-row budget, and four error paths — with **no** in-file test. Its helpers are tested in `req.rs`, but the composition (which is where D-08 and D-09 live) is not. `handlers/close.rs` (35 LOC) has none either, including the release-topic ordering it exists to guarantee.

Related: **no test drives `handle_req`, `handle_count`, or `handle_close` end-to-end.** The only message handler with an end-to-end unit test is `handle_agent_observer_event` (`event.rs:1318-1404`). Everything else in this group is tested at the helper level or in `crates/buzz-test-client`.

---

#### 4. LOW

##### D-21 — `ARCHITECTURE.md` numeric and structural drift
| Line | Claim | Actual |
|---|---|---|
| `:161` | frame size 65 536 B | 524 288 (`config.rs:14`) |
| `:161` | max historical results/filter 500 | 2 000 (`req.rs:25`) |
| `:208` | `SLOW_CLIENT_GRACE_LIMIT (3)` | config field, default 15 (`config.rs:470-473`) |
| `:150-159` | 4 inbound / 6 outbound messages | 5 / 7 — COUNT missing both ways |
| `:161` | no mention of 10-filter or 256-B sub_id caps | `protocol.rs:11`, `:14` (both NIP-11-advertised) |
| `:181-187` | auth described with no deadline | 5 s (`connection.rs:27`) |
| `:211-217` | cleanup = 5 steps | 7 — adds per-subscription `release_topic` (`connection.rs:265-270`) and last-connection presence clear (`:274-285`) |
| `:235` | step 10 "SEARCH INDEX — `search_index_tx.send` (bounded worker queue)" | removed; the code documents the removal at `event.rs:479-485` |
| `:225-237` | pipeline omits the 24200 observer branch and the pubkey-match gate ordering | `event.rs:636-669` |

Downstream: `.aidlc/reverse-engineer/configuration.md:89` and `:91` inherit the frame-size and grace-limit errors.

##### D-22 — Stale in-code comments naming line numbers and struct shapes
- `event.rs:2345` — "`filter_fanout_by_access` (this file, line 62)"; it is at `:115`.
- `event.rs:2338` — cites `state.rs:30-44` for `ConnEntry`; it is `:41-58`.
- `event.rs:2338-2343` — asserts `ConnEntry` "records `authenticated_pubkey` but NO community/tenant binding" and that `filter_fanout_by_access` "never compares" the receiver's tenant. Both were true at the cited revision `fb0d6a4ea` and are **false now**: `ConnEntry.community_id` exists (`state.rs:51`) and the comparison is the first fence (`event.rs:126-133`). The header also references a companion "documents the current (broken) behavior" test that it says MUST be deleted with the fix — that test is **absent**, so the deletion happened but the header did not.
- `event.rs:845-851` — describes a `Uuid::nil()` "global channel" Redis routing sentinel and a `is_nil()` check in the `main.rs` subscriber loop. Neither exists: `EventTopic::Global` is an enum variant (`event.rs:857`) and `main.rs:818-845` has no nil handling.
- `buzz-auth/src/lib.rs:69` — `channel_ids` documented as "reserved for future per-channel access control"; it is load-bearing at `req.rs:87-88`, `:137-139`, `count.rs:94-96`, `:145-149`.

##### D-23 — Production `expect()` on the fan-out hot path
`event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")`. The invariant holds at all three call sites (the cache is built from the same iterator that drives the send), but this is the group's only non-test `expect()`, it sits in delivery, a panic poisons a spawned dispatch task, and `AGENTS.md` prohibits new `unwrap()`/`expect()` in production paths. Fix: `if let Some(frame) = …` and count the miss.

##### D-24 — Unvalidated numeric config can panic or self-disable
| Var | Loader | Failure mode |
|---|---|---|
| `BUZZ_SEND_BUFFER=0` | `config.rs:459-462`, no `>0` filter | `mpsc::channel(0)` **panics** at `connection.rs:159` on the first connection |
| `BUZZ_MAX_CONNECTIONS=0` | `config.rs:449-452` | `Semaphore::new(0)` → every connection silently rejected with no frame |
| `BUZZ_MAX_CONCURRENT_HANDLERS=0` | `config.rs:454-457` | every EVENT/REQ/COUNT rejected as rate-limited |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT=0` | `config.rs:470-473` | disconnect on the **first** full buffer (`connection.rs:100`) |

`BUZZ_MAX_FRAME_BYTES` has a `>0` filter (`config.rs:467`) and the `BUZZ_RATE_LIMIT_*` family hard-errors on zero (`config.rs:270-283`), so the pattern exists — it just was not applied here. The `defaults_are_valid` test (`config.rs:938`) asserts exactly the invariants the loader fails to enforce, and only for the default path.

##### D-25 — `AuthState::Failed` is permanently sticky, including for transient causes
`auth.rs:161-165` writes `Failed` for a ban-state **DB error**, which is terminal (`auth.rs:58-66`). The surrounding comment (`auth.rs:98-112`) goes to visible lengths to avoid mislabelling a blip as a ban, but still pins the socket. Fail-closed is right; the residual cost is a forced reconnect on any Postgres hiccup, with no retry and no metric distinguishing it from a real auth failure (both increment `buzz_auth_failures_total`, differing only by the `reason` label — `auth.rs:169-171`).

##### D-26 — Silent drops with no signal
| Case | Site |
|---|---|
| Binary frame with invalid UTF-8 | `connection.rs:457-459` — no `else`, no NOTICE, no counter |
| Observer frame with an unrecognised `frame` tag value | `event.rs:1099-1102` → `OK true` (deliberate, documented) |
| REQ with two different `#h` values silently downgraded to a global subscription | `req.rs:1023-1027` — and a global subscription receives **no** channel events (`subscription.rs:320-327`), so the client gets a permanently silent subscription |
| Presence-set / presence-clear Redis errors | `event.rs:791-799` — `let _ =` |
| `#h` values all inaccessible in a search filter → filter skipped | `req.rs:580-582` |
| COUNT filter on an inaccessible channel → contributes 0 | `count.rs:150-151` |

##### D-27 — Case-sensitivity inconsistency across the filter gates
The p-gate compares `#p` with `==` (`req.rs:1070-1073`) and `result_gated_count_safe_for_pushdown` likewise (`:1177-1180`), while the engram gate (`:1123-1126`) and author-only gate (`:1216-1219`) use `eq_ignore_ascii_case`. Uppercase-hex `#p` therefore fails the p-gate but would clear an equivalent engram check. Fails closed, so it is a usability wart — but four sibling predicates should not disagree.

##### D-28 — `remote_addr` stored but unused for anything but logging
`connection.rs:61`, read only at `:183` and `:289`. No abuse correlation, no per-IP accounting, no IP in any rejection metric. The field's existence implies a capability the relay does not have (see D-04).

##### D-29 — File sizes
`event.rs` at 2461 lines mixes six concerns: metric-label bounding, frame serialisation, the access-filter chokepoint, the Redis fan-in path, post-commit dispatch, and three message handlers. `req.rs` at 1946 mixes the REQ handler, the search handler, filter→SQL translation, and seven authorization predicates that `count.rs` and `api/bridge.rs` import. There is no line-count guard for Rust (the `check-file-sizes.mjs` 1000-line ceiling in `AGENTS.md` applies to `mobile/`), so these will keep growing.

---

#### 5. Untested paths (production code with no test coverage in this group)

| Path | Site | Nearest coverage |
|---|---|---|
| `handle_req` end-to-end | `req.rs:44-418` | helper-level only |
| `handle_count` end-to-end | `count.rs:29-281` | **none** in-file |
| `handle_close` | `close.rs:12-30` | **none** |
| `handle_auth` (all four gates: ban cascade, allowlist, membership, NIP-OA materialisation) | `auth.rs:44-296` | only `extract_auth_tag_json` (3 tests, `auth.rs:297-349`) |
| `handle_event` dispatch gates (auth, pubkey match, kind 22242, scope) | `event.rs:611-696` | **none** |
| `handle_ephemeral_event` incl. presence parsing + 128-B truncation | `event.rs:739-874` | **none** |
| `handle_connection` / `handle_active_connection` lifecycle | `connection.rs:118-291` | send-loop only (4 tests) |
| `recv_loop` frame handling (oversize, binary, Ping/Pong) | `connection.rs:407-487` | **none** |
| `heartbeat_loop` 3-strike logic | `connection.rs:378-405` | **none** |
| `enforce_ws_admission` / `send_admission_result` | `connection.rs:594-679` | only `request_rejection_message` (`connection.rs:776-785`) and `admission.rs:46-157` |
| Auth-timeout task | `connection.rs:228-252` | **none** |
| `handle_search_req` | `req.rs:504-732` | only `build_search_channel_scope_filter` (2 tests) |
| `filter_to_query_params` beyond the `#d` matrix | `req.rs:860-991` | `#d` pushdown only (`req.rs:1594-1648`) |
| `filter_fully_pushable` | `req.rs:777-827` | **none** |
| Filter-cardinality limits | — | **no such limits exist** |
| Per-IP admission | — | **no such control exists** |

---

#### 6. Prioritised remediation order

1. **D-01** — move the sensitive-kind gates out of the `channel_id.is_none()` branch (`req.rs:182`). One-line structural change; `count.rs:52-75` is the model.
2. **D-03** — hoist the 30622/44200 owner fence into `filter_fanout_by_access` so all three fan-out paths inherit it.
3. **D-02** — collapse the four result-gated-kind lists onto `RESULT_GATED_KINDS`, with a coverage test.
4. **D-06 / D-21** — correct `ARCHITECTURE.md` §9 #2, the §2 limits line, §3 step 3/step 5, and §4 step 10.
5. **D-05** — cap per-filter array cardinality in `protocol.rs` and the kind-union size in `register_scoped`.
6. **D-08 / D-09** — add the `MessagesRead` check and sanitise DB errors in `count.rs`.
7. **D-13 / D-11 / D-10 / D-12** — protocol-correctness cluster: rate-limited `OK`, missing `EOSE`, `EOSE`-on-error, search-REQ sub_id replacement.
8. **D-20** — tests for `count.rs` and `close.rs`, then end-to-end tests for `handle_req` / `handle_count`.
9. **D-04** — wire `check_ip_connection` (after deciding proxy-header trust).
10. **D-14 / D-15 / D-16** — resource-hygiene cluster.
11. **D-07 / D-17 / D-18 / D-19 / D-22 / D-23 / D-24** — cleanup: remove dead methods, dedupe helpers and accessors, drop the duplicate filter store, fix stale comments, replace the production `expect()`, validate numeric config.
