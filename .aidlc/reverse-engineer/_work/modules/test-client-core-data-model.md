## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Data Model

This group covers the shared test harness (`src/lib.rs`, `src/main.rs`), two standalone bin targets (`src/bin/wamp_bench.rs`, `src/bin/mention.rs`), and three/four E2E suites exercising core NIP-01/NIP-42 relay protocol, NIP-42 host-binding, NIP-38 user status, and NIP-AP team events. None of these files define new domain types of their own — the harness re-exports wire-level types from `buzz-ws-client`, and every test builds `nostr::Event`s with `EventBuilder` and asserts on the JSON/tag shapes the relay returns.

#### The harness's own types (`src/lib.rs`)

`BuzzTestClient` (`crates/buzz-test-client/src/lib.rs:87-89`) is a thin wrapper around a single field, `inner: NostrWsConnection` (from `buzz-ws-client`) — it holds no state of its own (no buffer, no subscription table; `ARCHITECTURE.md`'s claim that `BuzzTestClient` itself wraps "a `VecDeque<RelayMessage>` buffer for message interleaving" is incorrect — that buffer lives one layer down, inside `NostrWsConnection` — see Debt for the full drift writeup, and Integrations for the delegation chain).

`TestClientError` (`lib.rs:18-64`) is a 10-variant `thiserror` enum that mirrors `buzz_ws_client::WsClientError` field-for-field, with a `From<WsClientError>` impl (`lib.rs:66-79`) that maps each variant across 1:1. `lib.rs` also re-exports three types verbatim from `buzz-ws-client`: `pub use buzz_ws_client::{parse_relay_message, OkResponse, RelayMessage}` (`lib.rs:16`). Consumers importing `buzz_test_client::RelayMessage` are using the exact same enum as `crates/buzz-ws-client/src/message.rs:6-42` — no shape difference, no wrapping.

`RelayMessage` (defined upstream at `buzz-ws-client/src/message.rs:6-42`) is a 7-variant enum: `Event { subscription_id, event: Box<Event> }`, `Ok(OkResponse)`, `Eose { subscription_id }`, `Closed { subscription_id, message }`, `Notice { message }`, `Auth { challenge }`, `Count { subscription_id, count }`. `OkResponse` (`message.rs:47-54`) is `{ event_id: String, accepted: bool, message: String }` — the canonical shape every test in this group asserts on via `ok.accepted` / `ok.message`.

#### Wire-level event/tag shapes constructed by this group's tests

None of the eight files in this group import `buzz_core::kind` — every kind number is a hardcoded `u16` literal or `Kind::Custom(N)` (confirmed by grep: zero matches for `buzz_core::kind` or `use buzz_core` across all eight files). This matches the same cross-cutting pattern the sibling `test-client-content`/`test-client-conformance` groups already documented for their files — a future renumbering in `buzz-core/src/kind.rs` would not be caught by a compile error anywhere in this crate. Every kind literal in this group was cross-checked against `crates/buzz-core/src/kind.rs` at analysis time and all matched:

| Kind | Constant (`buzz-core/src/kind.rs`) | Used in |
|---|---|---|
| 7 | `KIND_REACTION` (implicit, standard NIP-25) | `e2e_relay.rs` (reaction on a thread message) |
| 9 | `KIND_STREAM_MESSAGE` (implicit) | harness `send_text_message` default; most of `e2e_relay.rs` |
| 5 | `KIND_DELETION` = 5 (`kind.rs:56`, standard NIP-09) | `e2e_relay.rs` (message delete, reply delete), `e2e_team.rs` (tombstone) |
| 20001 | `KIND_PRESENCE_UPDATE` = 20001 (`kind.rs:403`) | `e2e_relay.rs::test_nip29_put_user_default_policy_allows`-adjacent tests use 20001 as a generic "ephemeral" example kind |
| 22242 | `KIND_AUTH` = 22242 (`kind.rs:77`) | `e2e_relay.rs::test_auth_event_kind_rejected`; `nip42_host_binding_live.rs` (via `EventBuilder::auth`) |
| 27235 | `KIND_HTTP_AUTH` = 27235 (`kind.rs:83`) | `e2e_relay.rs::nip98_post_header` (NIP-98 signed HTTP auth header) |
| 9000/9001/9002/9005/9007/9008 | `KIND_NIP29_PUT_USER`..`KIND_NIP29_DELETE_GROUP` = 9000..9008 (`kind.rs:275-285`) | `e2e_relay.rs` (channel creation, membership, metadata edit, archive) |
| 9007 (channel creation) content tags | — | `create_test_channel`/`create_private_channel_ws` build `["h", uuid], ["name", ...], ["channel_type","stream"], ["visibility","open"\|"private"]` (`e2e_relay.rs:170-178`, `:2179-2186`) |
| 10100 | `KIND_AGENT_PROFILE` = 10100 (`kind.rs:87`) | `e2e_relay.rs` (`channel_add_policy` settings event) |
| 13534 | `KIND_NIP43_MEMBERSHIP_LIST` = 13534 (`kind.rs:338`) | `e2e_relay.rs::test_client_submitted_nip43_membership_snapshots_are_rejected` (forged relay-only snapshot) |
| 39000/39001/39002 | `KIND_NIP29_GROUP_METADATA`/`_ADMINS`/`_MEMBERS` = 39000-39002 (`kind.rs:362-366`) | `e2e_relay.rs::test_nip29_standard_client_flow` (group discovery) |
| 39005 | `KIND_THREAD_SUMMARY` = 39005 (`kind.rs:375`) | `e2e_relay.rs::test_reply_ingest_pushes_live_thread_summary` |
| 40002 | `KIND_STREAM_MESSAGE_V2` (implicit) | used as a "non-matching kind" fixture in `test_subscription_filters_by_kind` |
| 44100/44101 | `KIND_MEMBER_ADDED_NOTIFICATION` = 44100, `KIND_MEMBER_REMOVED_NOTIFICATION` = 44101 (`kind.rs:472-476`) | `e2e_relay.rs` (six dedicated membership-notification tests) |
| 30176 | `KIND_TEAM` = 30176 (`kind.rs:250`) | `e2e_team.rs` |
| 30315 | `KIND_USER_STATUS` = 30315 (`kind.rs:70`) | `e2e_user_status.rs` |
| 49999 | not a registered constant (deliberately unused range) | `e2e_relay.rs::test_subscription_limit_enforced` uses this as a kind no other test writes, to avoid stale-event cross-talk in the overflow-subscription probe |

#### User status (`e2e_user_status.rs`) — kind 30315

`build_user_status_event(keys, d_tag, content, extra_tags)` (`e2e_user_status.rs:34-42`) always emits `["d", d_tag]` plus caller-supplied `extra_tags`, and sets `.content` to the literal status string (e.g. `"Working on NIP-38 support"`). Unlike `e2e_event_reminder.rs` (event reminders) in the sibling content group, there is no encryption here — content is always plaintext, matching `KIND_USER_STATUS`'s doc comment in `buzz-core/src/kind.rs:65-69`: "Stored globally (channel_id = NULL); user-owned personal data, not channel-scoped." No `KIND_USER_STATUS` test in this file ever attaches an `h` tag — the suite treats the kind as inherently global, consistent with the registry comment.

The NIP-33 replacement key is the standard `(pubkey, kind, d_tag)` triple. `test_user_status_multiple_d_tags_coexist` (`:174-212`) proves two independent `d`-tag "slots" (`general`, `music`) coexist under the same pubkey+kind — i.e. the d-tag genuinely partitions the replaceable-event space per NIP-33/NIP-38, not just per-pubkey. `test_user_status_stale_write_rejected` (`:214-272`) builds explicit-timestamp events via `.custom_created_at(Timestamp::from(...))` (`:224-230`, `:238-244`) rather than relying on wall-clock ordering, publishing a "newer" event *first* (created_at = now+100) then an "older" one (now-100) to prove the write order doesn't matter — only `created_at` does.

#### Team events (`e2e_team.rs`) — kind 30176

`team_event(keys, d_tag, content)` (`e2e_team.rs:33-38`) emits only `["d", d_tag]` — no other tags — and `.content` is always a hand-built JSON string of the shape `{"name": ..., "description": ..., "persona_ids": [...]}` (e.g. `:78-82`), matching the doc comment on `KIND_TEAM` in `buzz-core/src/kind.rs:245-249`: "Content is a JSON body projecting public team fields (name, description, persona_ids)." The file's own header comment (`:1-13`) is explicit that the team `d`-tag is "a slug-like string, not a 64-hex pubkey" and that this "exercises the generic parameterized-replaceable path that enforces only `D_TAG_MAX_LEN`, distinct from the persona slug envelope" — verified directly against the relay: `validate_persona_envelope` (`crates/buzz-relay/src/handlers/ingest.rs:1034-1080`) is gated specifically on `kind_u32 == KIND_PERSONA` (`ingest.rs:2049-2052`), so a kind:30176 team event never goes through the 64-char slug-grammar check (`^[a-z0-9][a-z0-9_-]{0,63}$`) that personas do — it only hits the generic NIP-33 `d_tag.len() > D_TAG_MAX_LEN` guard (`ingest.rs:2402-2410`, `D_TAG_MAX_LEN = 1024` per `crates/buzz-db/src/event.rs:140`). This is a real, confirmed structural difference in validation strictness between the two kinds, not just a comment in the test file.

The tombstone shape (`team_delete_event`, `:41-46`) is a kind:5 event carrying **only** an `a`-tag coordinate (`format!("{TEAM_KIND}:{}:{d_tag}", pubkey_hex)`) and no `e`-tag — the file's comment explicitly calls out that omitting the `e`-tag is what routes the relay down the coordinate-delete path rather than the event-id path (see Business Rules for the routing predicate this exercises, `has_e_tag` at `crates/buzz-relay/src/handlers/side_effects.rs:2300`).

#### NIP-42 host-binding (`nip42_host_binding_live.rs`)

This file constructs exactly one signed event shape: a kind:22242 NIP-42 AUTH event via `EventBuilder::auth(&challenge, parsed_relay_url)` (`nip42_host_binding_live.rs:70-72`), where `parsed_relay_url` is deliberately either the *matching* or a *mismatched* host relative to the WebSocket connection's actual origin — the entire test file's data model is "one event shape, two independently-chosen inputs (`connect_url`, `relay_tag_url`)" fed through `do_auth_with_relay_tag(connect_url, relay_tag_url)` (`:23-92`). There is no application-level content here; the object under test is purely the `relay` tag on the AUTH event and how it compares against the relay's own per-tenant host resolution (`crate::api::bridge::nip42_expected_relay_url`, see Business Rules/Integrations).

#### Core relay protocol (`e2e_relay.rs`) — the largest and most heterogeneous data surface in this group

Because this file spans NIP-01 wire protocol, NIP-11 relay metadata, NIP-29 group admin, NIP-05 profile sync, and relay invite tokens, its data model is the union of many small, single-purpose shapes rather than one coherent schema:

- **Channel creation** (`create_test_channel`, `:167-197`; `create_private_channel_ws`, `:2170-2192`): kind:9007 with `h`/`name`/`channel_type`/`visibility` tags, submitted over `POST /events` with an `X-Pubkey` dev-auth header (HTTP path) or over WS (private-channel helper).
- **NIP-98 HTTP auth header** (`nip98_post_header`, `:60-72`): builds a kind:27235 event with `u`/`method`/`payload`/`nonce` tags, base64-encodes it, and formats `Nostr <b64>` — the real production auth header shape, used only for the `/api/invites*` tests (contrasted with the `X-Pubkey` dev shortcut used everywhere else in this file).
- **Invite mint/claim bodies** (plain JSON, not Nostr events): `POST /api/invites` takes `{}` or `{"ttl_secs": N}`; `POST /api/invites/claim` takes `{"code": "..."}`; responses are `{"code", "url"}` (mint) and `{"status", "community_id", "host", "role"}` (claim) — these are REST JSON bodies, matching `crates/buzz-relay/src/api/invites.rs:45-67` (`MintInviteRequest`, `ClaimInviteRequest`) and the response literal at `invites.rs:355-360`.
- **kind:0 profile content** (`test_kind0_nip05_sync`, `:463-560`): a JSON `content` string `{"display_name": ..., "nip05": "<local>@<domain>"}` — the test constructs both a same-domain and an off-domain `nip05` value to probe the relay's sync-vs-clear side effect.
- **kind:9000/9001 with `role` tag** (`add_member_with_role_ws`, `:2211-2227`): extends the base PUT_USER shape with an optional third tag `["role", "admin"|"member"|...]`, exercising `buzz_db::channel::MemberRole` parsing on the relay side.
- **kind:39005 thread-summary content** (`test_reply_ingest_pushes_live_thread_summary`, `:2382-2477`): the relay-authored push carries a JSON `content` of shape `{"reply_count": N, ...}` and an `e`-tag pointing at the thread root — the test's `recv_summary` helper (`:2413-2427`) filters incoming events by kind and asserts on `content["reply_count"]` and the root `e`-tag value.
- **kind:44100/44101 notification shape**: relay-signed, carrying exactly a `p`-tag (target pubkey) and an `h`-tag (channel UUID) plus a JSON `content` of `{"type": "member_added"|"member_removed", "channel_id": ...}` (asserted at `:1858-1868` for the unarchive case and `:1229-1250`/`:1290-1310` for the add/remove case) — this is the concrete wire shape behind `AGENTS.md`'s "Thread counters" convention note and the relay's own comment at `side_effects.rs:1512-1520` describing 44100 as being "reused ... purely as a resubscribe trigger" on unarchive.

No test in this group constructs a NIP-44-encrypted payload, a Blossom media reference, or a git/manifest object — those data models belong to sibling groups (`test-client-media-agent`, `test-client-content`).
