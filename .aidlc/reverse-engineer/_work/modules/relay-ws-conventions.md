## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signatures

Every WS message handler follows one shape — `(payload…, Arc<ConnectionState>, Arc<AppState>)`, returning `()`, replying through `conn.send`:

| Handler | Signature | Site |
|---|---|---|
| AUTH | `async fn handle_auth(event: nostr::Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `auth.rs:44-49` |
| EVENT | `async fn handle_event(event: Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `event.rs:585` |
| REQ | `async fn handle_req(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `req.rs:44-49` |
| COUNT | `async fn handle_count(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `count.rs:29-34` |
| CLOSE | `async fn handle_close(sub_id: String, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `close.rs:12` |

Conventions that hold across all five:

1. **No `Result`.** Errors are terminal side effects (a `conn.send` + `return`), never propagated. There is not one `?` in a handler's top-level body.
2. **`Arc` by value, not `&`.** Uniform even for the two handlers that are awaited inline (`auth`, `close`) and never need `'static` — so a handler can be moved into a spawn without a signature change.
3. **Auth is extracted into locals inside a scoped block**, so the `RwLock` guard drops before any `.await` on I/O. `event.rs:611-631`, `req.rs:50-87`, `count.rs:38-50`, `auth.rs:45-74` all use the same `let (…) = { let auth = conn.auth_state.read().await; match &*auth { … } };` shape.
4. **Internal helper naming**: `handle_*` for message entry points, `handle_*_event` for sub-branches (`handle_ephemeral_event` `event.rs:739`, `handle_agent_observer_event` `event.rs:920`, `handle_search_req` `req.rs:504`), `*_authorized` for boolean filter gates, `filter_can_match_*` for capability predicates, `extract_*` for tag/field readers.
5. **Tracing**: `#[tracing::instrument(skip_all, fields(...))]` with `tracing::field::Empty` placeholders recorded once the values exist (`event.rs:584`, `:591-594`; `auth.rs:45`, `:77-80`). The dispatcher captures the span *before* `tokio::spawn` and attaches it with `.instrument(span)`, with an explicit comment on why (`connection.rs:522-523`).
6. **`#[allow(clippy::too_many_arguments)]`** is used rather than introducing param structs, with a stated rationale where non-obvious (`req.rs:503`, `state.rs:201-204`).

Free-function helper conventions:
- `pub(crate)` for cross-module helpers inside the crate (`req.rs:448`, `:483`, `:993`, `:1042`, `:1099`, `:1137`, `:1154`, `:1172`, `:1186`, `:1204`; `event.rs:35`, `:218`, `:326`).
- `pub` only where the HTTP bridge needs it (`req.rs:737` `build_event_query_from_filter`, `req.rs:777` `filter_fully_pushable`, `event.rs:115` `filter_fanout_by_access`, `event.rs:259` `fan_out_pubsub_event`).
- Private for file-local (`req.rs:829`, `:842`, `:860`, `:1013`, `:1225`; `event.rs:55`, `:59`, `:63`, `:76`, `:894`, `:1071`, `:1111`, `:1117`).

---

#### 2. Error-to-wire mapping

The rule is **message-kind determines frame kind**:

| Inbound | Rejection frame | Rationale / site |
|---|---|---|
| `EVENT` | `["OK", <event_id>, false, "<prefix>: <detail>"]` | NIP-20 acknowledgement — `event.rs:623`, `:639`, `:649`, `:660`, `:678`, `:733` |
| `REQ` | `["CLOSED", <sub_id>, "<prefix>: <detail>"]` | `req.rs:56`, `:67`, `:80`, `:101`, `:167`, `:185`, `:192`, `:199`, `:214` |
| `COUNT` | `["CLOSED", <sub_id>, …]` | `count.rs:43`, `:56`, `:63`, `:70`, `:85`, `:167` |
| parse failure (no sub_id yet) | `["NOTICE", "invalid message: …"]` | `connection.rs:493` |
| pre-dispatch throttle | `CLOSED` if a sub_id exists, else `NOTICE` | `request_rejection_message`, `connection.rs:587-592` |

Reason strings use the NIP-01 machine-readable prefix set, consistently:

| Prefix | Meaning as used here | Examples |
|---|---|---|
| `auth-required:` | not authenticated, or an auth-state error | `event.rs:626`, `req.rs:78`/`:82`, `count.rs:46`, `auth.rs:54`/`:63`/`:210`/`:291` |
| `restricted:` | authenticated but not permitted | `event.rs:663`, `:681`, `:1027`; `req.rs:170`, `:187`, `:194`, `:201`; `auth.rs:234` |
| `invalid:` | malformed / semantically rejected | `event.rs:642`, `:652`, `:757`, `:956`, `:1073`, `:1096` |
| `blocked:` | moderation ban | `auth.rs:160` |
| `rate-limited:` | throttled | `connection.rs:517`, `:546`, `:567`, `:666`, `:674`; `event.rs:1040` |
| `error:` | server-side fault or a protocol-level cap | `event.rs:765`, `:1012`; `req.rs:69`, `:103`, `:216`; `count.rs:85`, `:167` |

Two deliberate escalations beyond the frame:
- **Ban** — frame queued on `ctrl_tx`, then `cancel()` so the socket closes immediately (`auth.rs:173-182`). The "queue on ctrl, then cancel" idiom is named as a reusable convention at `connection.rs:328-336` and pinned by `connection.rs:856-882`.
- **Oversized frame** — `NOTICE` then `break` out of `recv_loop` (`connection.rs:428-433`, `:447-452`).

##### Sanitisation convention (not uniformly applied)

`handle_event` sanitises `IngestError::Internal` to `error: internal server error` with an explicit comment (`event.rs:726-731`). `handle_count` does the opposite: four sites forward the raw error with `format!("error: {e}")` (`count.rs:167`, `:203`, `:238`, `:273`). The `req.rs` historical path forwards nothing (it emits a bare `EOSE`, `:323`). Three different postures for the same class of failure.

##### Metrics-on-rejection convention

`event.rs` funnels every rejection through `reject(reason)` (`event.rs:30-32`) → `reject_with_transport("ws", reason)` (`ingest.rs:156`) with a **bounded** reason label — the four values used are `"auth"`, `"invalid"`, `"scope"`, `"error"` (`event.rs:622`, `:638`, `:648`, `:659`, `:677`, `:732`, `:969`, `:1022`). `req.rs`, `count.rs`, and `close.rs` do **not** emit a rejection counter at all, so REQ/COUNT denials are invisible in metrics.

---

#### 3. Locking conventions

| Convention | Evidence |
|---|---|
| `auth_state` is a `RwLock` because it is read-heavy after auth; `subscriptions` is a `Mutex` because it is write-heavy during REQ/CLOSE. Stated in the struct doc. | `connection.rs:50-52` |
| Guards are scoped so they drop before I/O `await`s. | `event.rs:611-631`, `req.rs:50-87`, `count.rs:38-50` |
| Only one nested acquisition exists, always in the same order: `auth_state` (read) → `subscriptions` (lock). No reverse ordering exists, so no deadlock. | `req.rs:51` → `:65` |
| DashMap guards are explicitly `drop`ped before a `remove` on the same map, to avoid self-deadlock. | `subscription.rs:408-410`, `:430-432`, `:447-449`, `:456-457`, `:465-466` |
| `authenticated_pubkey` uses `std::sync::RwLock` (not tokio) because reads are non-async and lock-poisoning is handled with `.ok()?`. | `state.rs:56`, `:246-256`, `:286-290` |

---

#### 4. Concurrency conventions

| Convention | Evidence |
|---|---|
| `try_send` / `try_acquire_owned` everywhere on the hot path — never `send().await` or `acquire().await`, so the recv loop cannot be blocked by a slow peer or a saturated handler pool. | `connection.rs:89`, `:149`, `:513`, `:541`, `:563`; `state.rs:453` |
| One documented exception: the audit enqueue uses `send().await` **on purpose**, with a written rationale. | `event.rs:574` (rationale `:551-557`) |
| Semaphore permits are `Owned` and dropped explicitly at the end of the spawned body. | `connection.rs:533`, `:555`, `:576` |
| CPU-bound signature verification always goes to `spawn_blocking`. | `event.rs:749`, `:927`; `ingest.rs:1462` |
| Long delivery loops yield cooperatively every 100 items. | `req.rs:401-404` |
| Ordered concurrency uses `buffered`, never `buffer_unordered`, when downstream ordering is semantically load-bearing — with the reason spelled out. | `req.rs:314` (doc `:299-303`, and the constant's own doc `:28-34`) |
| Concurrency bounds are compile-time asserted where a wrong value would be silently harmful. | `req.rs:37-41` (`const _: () = assert!(…)`) |
| `biased;` in `select!` where starvation would break a safety property. | `connection.rs:326-327` |

---

#### 5. Test conventions

Counts (all in-file `#[cfg(test)] mod tests`):

| File | Tests | `#[ignore]` | Module starts |
|---|---|---|---|
| `connection.rs` | 5 | 0 | `:689` |
| `subscription.rs` | 29 | 0 | `:574` |
| `handlers/event.rs` | 24 | 0 | `:1135` |
| `handlers/req.rs` | 45 | 0 | `:1233` |
| `handlers/auth.rs` | 3 | 0 | `:298` |
| `handlers/count.rs` | **0** | — | — |
| `handlers/close.rs` | **0** | — | — |
| `handlers/mod.rs` | **0** | — | — |
| **total** | **106** | **0** | |

Conventions observed:

1. **Infra-optional tests skip, not `#[ignore]`.** Instead of the `#[ignore = "requires Postgres"]` style used elsewhere in the crate (e.g. `api/operator.rs:706`), this group probes availability at runtime and returns early with an `eprintln!`: `event.rs:1653-1656` (Redis), `:1636-1639` and `:1735-1738` (Postgres+Redis via `audit_state()`), `:1642-1649` (`redis_url_if_available`). Net effect: `just test-unit` never fails on a missing dependency, but it also never reports that a test was skipped as a test outcome.
2. **Lazy-pool test state.** `fanout_access::test_state()` (`event.rs:2018-2020`) builds a full `AppState` with `PgPool::connect_lazy` and an intentionally dead Redis (`redis://127.0.0.1:1`, `event.rs:1974-1979`), so pure fan-out logic is testable with no infrastructure. Cache pre-seeding substitutes for DB reads (`event.rs:2100-2103`, `:2143-2152`).
3. **Fail-closed proved by omission.** `threaded_visibility_open_passes_through` (`event.rs:2298-2321`) relies on the lazy pool erroring a fresh lookup: pass-through therefore *proves* the threaded value was consulted. The reasoning is written into the test doc comment (`:2291-2294`).
4. **Test-only single-tenant wrappers** keep pre-multi-tenant tests readable: `register`, `remove_channel_subscriptions`, `channel_subscriber_conns`, `fan_out`, all `#[cfg(test)]` and delegating to the `*_scoped` form with `test_community()` = nil UUID (`subscription.rs:154-160`, `:228-233`, `:257-260`, `:333-336`, `:568-571`).
5. **Security regressions are named as such** and carry the invariant they pin in the doc comment: `test_global_sub_does_not_receive_channel_events` (`subscription.rs:996-1032`), `channel_less_event_must_drop_recipient_in_different_community` (`event.rs:2435-2457`), `local_echo_suppression_is_scoped_to_its_community` (`event.rs:1553-1600`).
6. **Red-team module convention.** `event.rs:2327-2458` is a `mod redteam` whose 130-line header cites the TLA+ spec (`docs/spec/MultiTenantRelay.tla`), the invariant, the mutation class, the exact code sites, the required structural fix, and the ownership routing. It also documents a self-deleting pattern ("MUST be deleted in the same change that fixes the leak") — though the companion "documents the current broken behavior" test it refers to at `:2358-2360` **is not present**, so the header is now partly stale (see the debt aspect).
7. **Byte-compatibility pinning** for anything performance-refactored: `fanout_event_frame_matches_legacy_format_byte_for_byte` (`event.rs:1154-1166`) and the `Arc`-must-not-escape-a-cycle assertion (`:1168-1188`).
8. **Truth-table tests** for boolean gates, one test per row: `resolve_request_local_access` gets all four rows (`req.rs:1299-1360`); `d_tag` pushdown gets five (`req.rs:1594-1648`); `p_gated_filters_authorized` for 44200 gets four numbered cases in one test with case comments (`req.rs:1490-1546`).
9. **Assertion messages carry the invariant**, not the mechanics: `"kinds:[] sub must NOT be in the wildcard index"` (`subscription.rs:962`), `"Inv_NonInterference: a connection bound to community A must not receive a community-B event"` (`event.rs:2452-2455`).
10. **Mock sink over a real socket** for send-loop tests: `MockSink` implements `Sink<WsMessage>` with a scripted `fail_after_flushes` so `send_loop_inner` terminates deterministically (`connection.rs:692-757`). This is why `send_loop` is a thin wrapper over the generic `send_loop_inner` (`connection.rs:296-306`).

Notable coverage gaps in convention terms: `count.rs` (281 LOC) and `close.rs` (35 LOC) have **no** in-file tests at all, and no test in this group drives `handle_req`, `handle_count`, or `handle_close` end-to-end — only their extracted helpers. `handle_agent_observer_event` is the single message handler with an end-to-end unit test (`event.rs:1318-1404`).

---

#### 6. Documentation conventions

| Convention | Evidence |
|---|---|
| Module-level `//!` stating the pipeline in one line | `connection.rs:1`, `subscription.rs:1`, `event.rs:1`, `req.rs:1`, `count.rs:1`, `auth.rs:1-13` |
| Every `pub` item has a doc comment (crate enforces `missing_docs`; the one opt-out is `#[allow(dead_code, missing_docs)] pub mod push_lease` at `handlers/mod.rs:23`) | throughout |
| Invariants are written as prose next to the code that enforces them, and cross-reference the plan/spec section | `event.rs:196-217`, `:100-114`; `req.rs:423-447`; `auth.rs:92-112` |
| Rejected alternatives are recorded inline rather than dropped | `event.rs:551-557` (why `send().await`), `req.rs:28-34` (why per-filter queries), `state.rs:1107-1116` (why only `private` is cached) |
| Numbered fences referencing an external design doc | `event.rs:161-167` ("Fence 3 (§4.8 phase-2)", "Fence 1"), `auth.rs:95-97` ("COMMUNITY_MODERATION_PLAN.md §0 decision 4", "MOD-7/M20") |
| Stale-comment risk is high because comments name line numbers and revisions | `event.rs:2345` ("this file, line 62" — `filter_fanout_by_access` is now at `:115`), `event.rs:2338` (cites `state.rs:30-44` for `ConnEntry`, which is now `:41-58`) |

---

#### 7. Metrics naming convention

`buzz_<subsystem>_<noun>_<unit>` with `_total` for counters, `_seconds` for duration histograms, bare noun for gauges:

- counters: `buzz_ws_connections_total`, `buzz_ws_auth_timeouts_total`, `buzz_ws_backpressure_disconnects_total`, `buzz_admission_rejections_total`, `buzz_auth_attempts_total`, `buzz_auth_failures_total`, `buzz_events_received_total`, `buzz_community_events_received_total`, `buzz_multinode_fanout_total`, `buzz_post_commit_dispatch_scheduled_total`, `buzz_post_commit_dispatch_errors_total`, `buzz_audit_send_errors_total`, `buzz_req_global_access_resolution_skips_total`, `buzz_count_fallback_rejections_total`
- gauges: `buzz_ws_connections_active`, `buzz_subscriptions_active`
- histograms: `buzz_event_processing_seconds`, `buzz_fanout_recipients`, `buzz_ws_send_batch_size`

Label cardinality is explicitly bounded: `bounded_kind_label` (`event.rs:35-53`) collapses unknown kinds to `"other"`, and kind × community is deliberately never crossed (rationale `event.rs:597-604`). Rejection reasons are `&'static str` by type (`event.rs:30`), so the label set cannot grow at runtime.

---

#### 8. Code-style facts (quality-gate relevant)

| Check | Result |
|---|---|
| `unsafe` blocks in the 8 files | **0** |
| `unwrap()` / `expect()` outside `#[cfg(test)]` | **1** — `event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")` |
| `TODO` / `FIXME` / `XXX` / `HACK` markers | **0** |
| `#[ignore]`d tests | **0** |
| `#[allow(dead_code)]` | 1, on the out-of-group `push_lease` module (`handlers/mod.rs:23`) |
| Files over 1000 lines | 3 of 8 — `event.rs` 2461, `req.rs` 1946, `subscription.rs` 1562 |
| Production-code share | `connection.rs` 688/893 (77%), `subscription.rs` 573/1562 (37%), `event.rs` 1134/2461 (46%), `req.rs` 1232/1946 (63%), `auth.rs` 297/350 (85%) |
