## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Business Rules

This doc enumerates, per the task brief, precisely which behavior of each NIP is asserted by
`e2e_nostr_interop.rs`, and precisely which multi-tenant isolation property each
`conformance_multitenant.rs` test asserts — not just "tests NIP-17" or "tests isolation."

---

#### 1. `e2e_nostr_interop.rs` — NIP-by-NIP behavior table

| NIP | Test function | Line | Exact behavior asserted |
|---|---|---|---|
| NIP-50 (search) | `test_nip50_search_returns_results_and_eose` | `:262-338` | (a) a `search`-filter subscription returns the matching message pre-EOSE; (b) EOSE terminates the historical batch; (c) search is **one-shot** — a message sent *after* EOSE on the same subscription is never delivered live (asserts `TestClientError::Timeout` or a non-EVENT message, `:315-330`) |
| NIP-50 | `test_nip50_search_mixed_filters_rejected` | `:342-399` | a `REQ` combining one filter with `search` and one without is rejected with `CLOSED` whose message contains `"mixed"` (case-insensitive) — asserted, not just "rejected" |
| NIP-50 | `test_nip50_search_empty_results` | `:403-435` | a search matching nothing returns EOSE with **zero** events (not an error, not a hang) |
| NIP-50 | `test_nip50_search_relevance_order` | `:1055-1113` | of three messages (oldest = exact phrase match, middle = no match, newest = partial match), the **first returned result is the exact-match message**, not the chronologically-newest partial match — explicitly proving relevance ranking, not recency ordering (`:1103-1112`) |
| NIP-50 | `test_nip17_gift_wrap_not_searchable` | `:971-1051` | a kind:1059 gift wrap and a kind:9 control message share a unique search token; the control **is** returned by search, the gift wrap **is not** — asserts storage-level exclusion (the `events.search_tsv` generated column), citing the mutate-bite test in `buzz-search` directly (`:990-994`) |
| NIP-10 (threads) | `test_nip10_thread_reply_creates_metadata` | `:439-502` | a WS reply carrying `["e", root_id, "", "reply"]` is (a) accepted, (b) surfaced by the relay's thread-query surface (`POST /query` with `depth_limit`) under that root, (c) the recorded copy still carries the same e-tag pointing at the root, and (d) the **root itself is excluded** from its own thread-reply list |
| NIP-10 | `test_nip10_unknown_parent_rejected` | `:506-540` | a reply e-tagging a nonexistent 32-byte id is rejected with `OK false` and a message containing `"parent not found"` or `"not found"` |
| NIP-10 | `test_nip10_root_mismatch_rejected` | `:544-586` | a reply carrying **both** an explicit `root` marker (pointing at a fabricated id) and a `reply` marker (pointing at the real parent) — i.e. a self-contradictory two-marker claim — is rejected, message containing `"root tag does not match"` or `"root"` |
| NIP-10 | `test_nip10_thread_reply_not_in_top_level` | `:878-967` | precisely pins the relay's `depth`/`broadcast` top-level predicate from both directions in one test: a depth-1 reply with **no** `broadcast` tag is excluded from top-level view; an otherwise-identical depth-1 reply **with** `["broadcast","1"]` **is** surfaced; both remain visible via the thread-query surface regardless; the root remains top-level. This is reconstruction-order-adjacent but is really a *visibility* rule, not an ordering rule — no assertion in this file checks the ordering of multiple replies within a thread |
| NIP-17 (gift wrap) | `test_nip17_gift_wrap_accepted` | `:589-618` | a kind:1059 signed by an **ephemeral** key different from the connection's authenticated key is still accepted — i.e. gift wraps are exempt from the pubkey-must-match-auth rule other kinds presumably enforce |
| NIP-17 | `test_nip17_gift_wrap_requires_p_filter` | `:622-673` | subscribing to `{kinds:[1059]}` with **no** `#p` filter is rejected via `CLOSED`, message containing `"p-gated"`, `"#p"`, or `"restricted"` |
| NIP-17 | `test_nip17_gift_wrap_recipient_receives` | `:677-756` | end-to-end delivery: B subscribes `{kinds:[1059], #p:[B]}` *before* A sends; B receives the exact event (kind, content match) on the correct subscription id — proves routing, not just acceptance |
| DM discovery (not a numbered NIP; Buzz-custom kinds 39000/44100/41010) | `test_dm_discovery_events_emitted` | `:760-874` | opening a DM (kind:41010) causes the relay to emit, and make queryable, both a kind:44100 membership notification (carrying `#p` and `#h` of the DM channel) and a kind:39000 discovery event carrying **both** `"hidden"` and `"private"` tags |

**What this file does *not* assert about NIP-17**, despite exercising kind:1059 nine times across
three tests: it never checks that gift-wrap **content is actually encrypted** (the content is a
plain-string placeholder, never NIP-44 ciphertext), and it never unwraps a seal/rumor pair. Its
NIP-17 coverage is entirely about the relay's *transport* treatment of the outer gift-wrap
envelope (acceptance despite key mismatch, `#p`-gating, delivery, non-searchability) — see the
Security doc for why this specific scope is nonetheless the security-relevant scope.

##### Non-NIP business rules also asserted in this file (NIP-DV, dedup, channel window)

| Test | Line | Behavior |
|---|---|---|
| `test_historical_req_dedup_preserves_or_semantics` | `:1116-1166` | a REQ with two filters (one that excludes the target message by author, one that doesn't) still returns the message exactly once — OR semantics across filters survive post-match dedup |
| `test_empty_kinds_returns_zero_events` | `:1170-1206` | `kinds: []` returns **zero** events (the empty-kinds sentinel is honored as "match nothing," not "match everything") even when matching data exists |
| `test_nipdv_hide_then_reopen_updates_snapshot` | `:1256-1320` | hiding a DM (kind:41012) adds its channel id to the viewer's kind:30622 snapshot's hidden set; re-opening (kind:41010) removes it — full round trip |
| `test_nipdv_same_second_reopen_supersedes_hide` | `:1324-1379` | hide-then-reopen issued back-to-back with **no sleep** (same wall-clock second) still resolves with re-open winning, via a `created_at = max(now, prior+1)` monotonic-advance rule on the publisher side |
| `test_nipdv_snapshot_is_private_to_owner` | `:1383-1426` | a third party's REST `#p=<owner>` query for kind:30622 is rejected with HTTP 403, not merely filtered |
| `test_nipdv_two_viewers_independent_snapshots` | `:1430-1478` | two viewers hiding different DMs with a shared third party do not clobber each other's snapshot — proves the replacement key is per-viewer (`d`-tag), not global |
| `test_nipdv_ws_req_rejects_third_party` | `:1482-1540` | the same third-party-snapshot-read rule holds over WS `REQ`, closing with `CLOSED` rather than ever emitting `EVENT` |
| `test_nipdv_ids_query_rejects_third_party` | `:1544-1597` | a **kindless** `ids:[snapshot_id]` query (which is gate-exempt at the filter level) still returns zero results — the *result-level* owner check is the actual backstop for this bypass shape |
| `test_nipdv_explicit_kind_query_forbidden_for_third_party` | `:1601-1647` | the same query **with** an explicit `kinds:[30622]` is rejected earlier, at the filter gate, with HTTP 403 — a different failure mode than the previous test, deliberately distinguished |
| `test_nipdv_search_rejects_third_party` | `:1651-1699` | a kindless NIP-50 `search` filter carrying the target snapshot's id as an `ids` bypass attempt also returns zero results |
| `test_channel_window_rows_overlays_and_exact_multiple_exhaustion` | `:1776-1916` | the `top_level:true` window: replies never appear as rows; a replied-to root carries a relay-signed (not requester-signed) kind:39005 summary with correct `reply_count`; a reaction appears in the aux closure; pagination across an **exact-multiple** boundary (4 rows, page size 2) correctly reports `has_more:false` on the final page via the server-supplied `39006` overlay rather than an inferred "rows < limit" heuristic; no row is lost or duplicated across pages |
| `test_channel_window_rejects_half_cursor_and_client_overlay_kinds` | `:1920-1986` | `until` without `before_id` (or a malformed `before_id`) is a deterministic HTTP 400, never a silent fallback to a head request; client-submitted kind:39005/39006 events are rejected at ingest (overlay kinds are relay-only) |

---

#### 2. `conformance_multitenant.rs` — isolation-property table

Full inventory of the file's 27 test-shaped items (18 `#[tokio::test]` + 1 `#[test]` + 8
`pending_lane`-stubbed `#[tokio::test]`s, per the direct count run against the file), grouped by
module, with the precise isolation property each asserts:

| Module | Test | Line | Status | Precise property asserted |
|---|---|---|---|---|
| `row_zero_host_binding` | `unmapped_host_fails_closed_generically` | `:115-232` | live | an unmapped host (a) returns HTTP 404 on the non-`nostr+json` door while a mapped host does not; (b) the 404 body echoes neither the host authority string nor the host label; (c) a raw WS handshake to the unmapped host fails at the upgrade, never completing then binding to a default tenant |
| `row_zero_host_binding` | `client_supplied_community_cannot_override_host` | `:258-338` | live | a channel that exists **only** in community B is not postable via a kind:9 `#h`-tag from a connection on host A, even though B's own channel is genuinely open and postable (positive control at `:270-276`); the rejection is specifically pinned to the string `"restricted: not a channel member"` (not just any-rejection) to rule out an earlier gate (auth/parse/relay-membership) accidentally producing the same red; the rejection body does not echo B's channel UUID |
| `nip11_relay_info` | `nip11_is_not_a_cross_community_enumeration_oracle` | `:469-551` | live | NIP-11 documents from host A and host B are byte-identical **except** for each community's own `icon` field; an unmapped host receives the *same* document (200, not 404) with **no** `icon` at all — so neither a status-code difference nor a field difference can be used to enumerate which hosts are configured |
| `api_tokens_nip98_replay` | `token_minted_in_a_does_not_authorize_in_b` | `:636-679` | live, but **doc-only** (empty body, no assertion) | not wire-tested at all — the doc comment states the mint-token HTTP surface doesn't exist in `buzz-relay` (no `/tokens` route), so the "mint in A, present to B" precondition has no entry point; the real proof is cited as living in `buzz-db`'s unit tests (`crates/buzz-db/src/api_token.rs:425`, `:488`) |
| `api_tokens_nip98_replay` | `nip98_replay_seenset_is_shared_and_community_scoped` | `:729-893` | live | (1) **load-bearing bite**: reposting the byte-identical NIP-98-authenticated event to the *same* community (A) a second time is rejected with HTTP 401 and a body containing `"replay"` — proves the shared seen-set is actually consulted; (2) **tripwire, not a bite** (per the file's own doc comment, `:772-799`): an independent event signed for host B succeeds even though A's event was already spent — but the doc comment concedes this arm is "structurally redundant for natural wire traffic" because differing `u`-tag URLs already force differing event ids, so it only catches a *future* regression that globalizes/collapses the seen-set key |
| `membership_allowlist` | `archive_in_a_does_not_affect_b` | `:906-911` | **stub** (`pending_lane`) | not implemented; obligation text only |
| `users_profiles_nip05` | `same_pubkey_distinct_profiles_in_two_communities` | `:1039-1149` | live | the **same pubkey** publishing distinct kind:0 content on host A and host B: each host's `POST /query` for that pubkey's kind:0 returns exactly 1 event, and its content is that host's own — never the other's |
| `users_profiles_nip05` | `same_nip05_local_part_on_two_hosts_is_independent` | `:1151-1276` | live | the same NIP-05 local-part (`alice`) registered by **distinct** pubkeys on hosts A and B resolves, via `GET /.well-known/nostr.json?name=alice`, to each host's own pubkey — plus a secondary check that the `relays` map advertises each host's own domain, not a global `RELAY_URL` |
| `channelless_global_events_dms` | `same_event_id_and_dtag_coexist_across_communities` | `:1291-1298` | **stub** | not implemented |
| `channelless_global_events_dms` | `dm_does_not_cross_deliver_between_communities` | `:1301-1307` | **stub** | not implemented |
| `feed_read_side_isolation` | `feed_mentions_do_not_cross_communities_over_the_wire` | `:1326-1336` | **stub** | not implemented; doc comment notes DB-level adversarial coverage already exists in `buzz-db/src/feed.rs`, this row is specifically the wire-level gap |
| `channels_membership` | `same_channel_uuid_in_two_communities_is_isolated` | `:1552-1641` | live | a **shared channel UUID** created independently in A and B: the same pubkey posts distinct-content kind:9 messages to "the same" UUID on each side; `POST /query #h:<UUID>` on A returns exactly A's message (count 1, correct content), and the mirror holds for B — proving the read-side community fence (`query_events`'s `WHERE community_id`), not just channel-name non-collision |
| `workflows` | `workflow_trigger_is_community_confined` | `:1873-1940` | live | a workflow defined under community A (server-generated id) cannot be triggered under host B — even by a caller who **is** a legitimate member of the same channel UUID in B (membership is deliberately not the confounding variable); firing under A succeeds as a positive control; rejection message is the generic `"workflow not found"` |
| `workflows` | `approval_token_is_community_confined` | `:1941-1948` | **stub** | not implemented; doc comment states the executor approval gate is an explicit TODO (WF-08) and no wire path mints a pending approval yet |
| `search_fts` | `search_results_and_deletes_are_community_scoped` | `:2136-2284` | live | shared channel UUID + shared search token, community-distinct content: (a) each host's NIP-50 search returns exactly 1 hit with that host's own content; (b) a NIP-09 delete issued against A's event removes it from A's search results; (c) B's hit is **unaffected** by A's delete |
| `pubsub_presence_typing` | `fanout_and_presence_do_not_cross_communities` | `:2516-2606` | live | two separate fences in one test: (1) the same pubkey publishing distinct kind:20001 presence on A and B — each host's synthesized-presence query (Redis-backed) returns only its own status, never overwritten by the other; (2) shared channel UUID + shared kind:20002 typing subscription on both hosts — each side's live-delivery drain receives **exactly its own** typing event, never the other community's content |
| `media_blossom` | `media_metadata_boundary_holds_while_blob_bytes_shared` | `:2622-2627` | **stub** | not implemented |
| `git_hosting` | `same_owner_repo_isolated_push_does_not_cross` | `:2640-2645` | **stub** | not implemented |
| `mesh_agents_cli` | `one_key_two_communities_no_bleed` | `:2658-2663` | **stub** | not implemented |
| `audit_log` | (no test fn — doc-only module, `:2669-2723`) | n/a | **doc-only, no test item at all** | not merely unimplemented — the module doc comment argues no client-reachable wire surface for audit exists, so no wire test is *possible*; cites `buzz-audit`'s own unit tests as the actual proof location |
| `n1_parity` | (no test fn — doc-only module, `:2728-2739`) | n/a | **doc-only** | states N=1 parity is asserted by the *existing* e2e suites (including this repo's `e2e_nostr_interop.rs`) run against a single-host deployment; no new assertion lives here |

**Tally**: of 19 modules, 8 have at least one live (non-stub) `#[ignore]`d test; 8 test items are
`pending_lane` stubs that always panic if run with `--ignored`; 2 modules (`audit_log`, `n1_parity`)
contain no test function at all, only doc comments; 1 test (`token_minted_in_a_does_not_authorize_in_b`)
is a compiling `#[test]` with an empty body that asserts nothing.

**Which scenarios specifically exercise the tenant-boundary-crossing shapes `buzz-conformance`'s
own fixtures are designed to catch** (per the task brief's specific question): the
`row_zero_host_binding`, `channels_membership`, `workflows`, `search_fts`, and
`pubsub_presence_typing` live tests are the closest wire-level analogues to `buzz-conformance`'s
`Inv_NonInterference` (foreign-row-leak) and `AuthCheck`-override invariants — they construct
exactly the "same channel/community-adjacent id in two tenants, prove no cross-read" shape the
`buzz-conformance` data-model doc describes as `check_row_labels`'s target property. But as
established in Data Model/API Surface, none of them do so *through* `buzz_conformance` types —
they are an independent, parallel proof of a similar property using wire assertions instead of
abstract-trace replay.
