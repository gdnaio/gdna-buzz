## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: API Surface

Both files are consumers, not providers — there is no public API exposed *by* these test files
(no `#[bin]`, no library exports). This doc covers (a) which `buzz_test_client` harness functions
each file calls, (b) the relay-side HTTP/WS surface each file exercises, and (c) — the specific
question the task brief calls out — whether `conformance_multitenant.rs` calls into the
`buzz-conformance` crate directly.

---

#### 1. `buzz_test_client::BuzzTestClient` methods called

`BuzzTestClient` is defined in `crates/buzz-test-client/src/lib.rs:83-244` (read for context only,
per task scope — analyzed as primary subject by the sibling group that owns `lib.rs`). Its public
methods and each file's usage:

| Method | Signature (from `lib.rs`) | Used in `e2e_nostr_interop.rs` | Used in `conformance_multitenant.rs` |
|---|---|---|---|
| `connect` | `connect(url: &str, keys: &Keys) -> Result<Self, TestClientError>` (`lib.rs:94-98`) | Yes — every test (e.g. `e2e_nostr_interop.rs:275`) | Yes — `users_profiles_nip05`, `channels_membership`, `search_fts`, `pubsub_presence_typing` modules only (e.g. `conformance_multitenant.rs:1122`) |
| `connect_unauthenticated` | `lib.rs:101-105` | No | No |
| `authenticate` | `lib.rs:108-111` | No (only via `connect`) | No |
| `authenticate_with_nip_oa` | `lib.rs:114-121` | No | No |
| `send_event` | `send_event(event: Event) -> Result<OkResponse, TestClientError>` (`lib.rs:124-126`) | Yes — e.g. `e2e_nostr_interop.rs:467` (NIP-10 reply), `:606` (gift wrap) | Yes — e.g. `conformance_multitenant.rs:1436` (kind:9 post) |
| `send_text_message` | `send_text_message(keys, channel_id, content, kind) -> Result<OkResponse, ...>` (`lib.rs:129-141`) | Yes — `e2e_nostr_interop.rs:279` | No |
| `subscribe` | `subscribe(sub_id, filters: Vec<Filter>) -> Result<(), ...>` (`lib.rs:144-157`) | Yes — e.g. `e2e_nostr_interop.rs:293` | Yes — `search_fts::search_for` (`conformance_multitenant.rs:2044`), `pubsub_presence_typing::subscribe_typing` (`:2364`) |
| `close_subscription` | `lib.rs:160-163` | Yes — `e2e_nostr_interop.rs:1220` (`read_snapshot_event`) | No |
| `send_raw` | `lib.rs:166-169` | No | No |
| `recv_event` | `recv_event(timeout: Duration) -> Result<RelayMessage, ...>` (`lib.rs:172-177`) | Yes — e.g. `e2e_nostr_interop.rs:311` | Yes — `pubsub_presence_typing::drain_live_events` (`conformance_multitenant.rs:2394`) |
| `collect_until_eose` | `collect_until_eose(sub_id, timeout) -> Result<Vec<Event>, ...>` (`lib.rs:180-202`) | Yes — e.g. `e2e_nostr_interop.rs:297` | Yes — `search_fts::search_for` (`:2045`) |
| `disconnect` | `disconnect(self) -> Result<(), ...>` (`lib.rs:243-246`) | Yes — every test | Yes — every module that opens a `BuzzTestClient` |

Neither file calls `authenticate_with_nip_oa` — NIP-OA owner-attestation flows are out of scope for
both interop and multi-tenant conformance coverage as currently written.

`e2e_nostr_interop.rs` is the heavier consumer of the harness surface (uses 9 of 10 public
methods); `conformance_multitenant.rs` uses only 5, and 4 of its 8 fully-implemented test modules
(`row_zero_host_binding`, `nip11_relay_info`, `api_tokens_nip98_replay`, `workflows`) never
instantiate `BuzzTestClient` at all — they drive the relay purely over `reqwest` REST calls (see
§3 below), because their obligations are HTTP-door properties (host binding, NIP-11 content,
NIP-98 header replay, workflow trigger confinement) that don't need a live WS session.

---

#### 2. Non-`BuzzTestClient` harness imports

| Import | From | Used by |
|---|---|---|
| `RelayMessage` | `buzz_test_client::RelayMessage` (re-export of `buzz_ws_client::RelayMessage`, `lib.rs:11`) | Both files, for pattern-matching `Event`/`Eose`/`Closed`/`Notice` variants |
| `TestClientError` | `buzz_test_client::TestClientError` (`lib.rs:36-64`) | `e2e_nostr_interop.rs:22` only (matches `TestClientError::Timeout` at `:315`) — `conformance_multitenant.rs` never names this type, relying on `.expect()`/`.unwrap()` to surface harness errors as test panics instead |

---

#### 3. Relay HTTP/WS surface exercised

##### `e2e_nostr_interop.rs`

| Door | Method/path | Purpose | First call site |
|---|---|---|---|
| WS | connect + NIP-42 AUTH (via `BuzzTestClient::connect`) | all message send/subscribe traffic | `e2e_nostr_interop.rs:275` |
| REST | `POST /events` (dev-mode `X-Pubkey` header) | channel creation, root messages, DM open/hide/reopen commands | `create_test_channel` `:44-72`, `send_rest_message` `:84-107`, `create_dm` `:110-147`, `post_signed_event` `:150-176` |
| REST | `POST /query` | thread-reply lookup (`depth_limit` extension), plain channel query, channel-window (`top_level` extension), cross-viewer NIP-DV probes | `query_thread_replies` `:184-210`, `query_channel_messages` `:233-256`, `query_channel_window` `:1703-1727` |
| REST | `GET /query` cross-viewer forbidden-check | NIP-DV third-party snapshot read | `e2e_nostr_interop.rs:1401-1420` (`test_nipdv_snapshot_is_private_to_owner`) |

##### `conformance_multitenant.rs`

| Door | Method/path | Purpose | Module |
|---|---|---|---|
| HTTP | bare `GET <base_url>` (default Accept, non-`nostr+json`) | fail-closed host-binding proof (mapped vs. unmapped status) | `row_zero_host_binding::unmapped_host_fails_closed_generically` (`:130-146`) |
| WS handshake (no auth) | `tokio_tungstenite::connect_async` against the unknown host | WS-upgrade-door fail-closed proof | `row_zero_host_binding::unmapped_host_fails_closed_generically` (`:225-232`) |
| REST | `POST /events` | channel creation, cross-community override attempt, workflow define/trigger, kind:0/kind:9/kind:20001/kind:20002 posts | `create_open_channel` (`:346-372`), `post_kind9` (`:383-407`), and per-module equivalents |
| HTTP | `GET <base_url>` with `Accept: application/nostr+json` | NIP-11 relay-info enumeration-oracle check | `nip11_relay_info::fetch_nip11` (`:436-448`) |
| REST | `POST /events` with `Authorization: Nostr <base64>` (hand-built NIP-98 header, not via `buzz-auth`) | NIP-98 replay seen-set proof | `api_tokens_nip98_replay::nip98_replay_seenset_is_shared_and_community_scoped` (`:729-893`) |
| REST | `POST /query` | kind:0 profile isolation, NIP-05 well-known lookup, kind:9 channel-scoped isolation, FTS search isolation, synthesized-presence isolation | `query_kind0` (`:952-971`), `query_kind9_in_channel` (`:1461-1481`), `query_presence` (`:2366-2382`) |
| HTTP | `GET /.well-known/nostr.json?name=<local>` | NIP-05 cross-host resolution isolation | `users_profiles_nip05::same_nip05_local_part_on_two_hosts_is_independent` (`:1230-1238`, `:1264-1268`) |
| WS | connect + subscribe (via `BuzzTestClient`) | kind:0 publish, kind:9 post/search, typing fan-out | `users_profiles_nip05`, `channels_membership`, `search_fts`, `pubsub_presence_typing` modules |

Both files exclusively use the relay's **generic bridge surface** (`POST /events` / `POST /query`)
plus raw NIP-11/NIP-05/WS doors — neither uses any `buzz-cli`-style typed REST endpoint, consistent
with `AGENTS.md`'s stated preference for the generic Nostr bridge over endpoint-specific JSON APIs.

---

#### 4. Does `conformance_multitenant.rs` call into `buzz-conformance` directly? — No, verified exhaustively

The task brief requires an exact-imports listing if this file touches the `buzz-conformance` crate.
It does not. Full accounting:

- **Crate-level `use` statements**: there are none at file scope for `buzz_conformance` — the file
  opens with `use std::time::Duration;` (`conformance_multitenant.rs:43`) and every other `use` is
  declared inside a `mod { ... }` block, none of which names `buzz_conformance`.
- **Per-module imports actually present** (exhaustive list, one row per `use` line in the file):
  - `api_tokens_nip98_replay`: `base64::{engine::general_purpose::STANDARD as BASE64, Engine as
    _}`, `nostr::{EventBuilder, Keys, Kind, Tag}`, `sha2::{Digest, Sha256}` (`:562-565`)
  - `users_profiles_nip05`: `buzz_test_client::BuzzTestClient`, `nostr::{EventBuilder, Keys, Kind}`
    (`:919-920`)
  - `channels_membership`: `buzz_test_client::BuzzTestClient`, `nostr::{EventBuilder, Keys, Kind,
    Tag}` (`:1345-1346`)
  - `workflows`: `nostr::{EventBuilder, Keys, Kind, Tag}` (`:1650`)
  - `search_fts`: `buzz_test_client::{BuzzTestClient, RelayMessage}`, `nostr::{Alphabet,
    EventBuilder, Filter, Keys, Kind, SingleLetterTag, Tag}` (`:1957-1958`)
  - `pubsub_presence_typing`: `buzz_test_client::{BuzzTestClient, RelayMessage}`,
    `nostr::{Alphabet, EventBuilder, Filter, Keys, Kind, SingleLetterTag, Tag}` (`:2293-2294`)
  - `row_zero_host_binding`, `nip11_relay_info`, `membership_allowlist`,
    `channelless_global_events_dms`, `feed_read_side_isolation`, `media_blossom`, `git_hosting`,
    `mesh_agents_cli`, `audit_log`, `n1_parity`: no `use` statements beyond the file-level `super::*`
    glob (which itself only carries `Duration`, `url_a`, `url_b`, `url_unknown`, `pending_lane`).
- **Cargo dependency**: `buzz-conformance` is absent from `crates/buzz-test-client/Cargo.toml`'s
  `[dependencies]` and `[dev-dependencies]` sections entirely (full manifest reviewed —
  `Cargo.toml:9-32`). A file cannot import a crate its own package manifest doesn't declare, so
  this is independently conclusive without the grep.
- **Only reference to the crate name anywhere nearby**: `token_minted_in_a_does_not_authorize_in_b`'s
  doc comment says the row's black-box vantage point deliberately excludes `sqlx`/`buzz-db` reads
  and lists the file's actual dependency set as `buzz-ws-client`/`reqwest`/`tokio-tungstenite`/`s3`
  (`conformance_multitenant.rs:600-604`) — `buzz-conformance` is conspicuously not in that
  self-description either, confirming the omission is by design, not oversight.

**Conclusion for this aspect:** `conformance_multitenant.rs` is a pure black-box HTTP/WS conformance
suite. It proves multi-tenant isolation entirely through wire-observable request/response pairs
against a live (assumed two-host) relay. It has **no code path that constructs a `TraceStep`,
binds a `Tracer`, or calls `check_trace`**. The file cannot be, and is not, the place where the
`buzz-conformance` crate's own replay-checker pipeline gets test-time exercise — that pipeline's
only exercise is `buzz-conformance`'s own in-crate unit/fixture tests (per the earlier batch's
integrations doc). See the Security and Debt docs for what this means for the "does test-time
verification of the conformance gate happen anywhere" question.
