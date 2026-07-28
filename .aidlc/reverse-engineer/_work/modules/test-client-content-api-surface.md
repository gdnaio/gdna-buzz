## Module: buzz-test-client — content & git E2E (`crates/buzz-test-client/tests`)
### Aspect: API Surface

#### Shared harness functions consumed from `buzz_test_client` (`src/lib.rs`)

`BuzzTestClient` is the only public type these four files import from the harness crate (`e2e_git.rs` imports none of it — see below). Its full public surface is defined at `crates/buzz-test-client/src/lib.rs:87-206`. Usage per file:

| Harness function | Signature (from `lib.rs`) | `e2e_long_form.rs` | `e2e_event_reminder.rs` | `e2e_human_edit_agent_content.rs` |
|---|---|---|---|---|
| `BuzzTestClient::connect` | `async fn connect(url: &str, keys: &Keys) -> Result<Self, TestClientError>` (`:96-100`) | 8 call sites | 1 (`:594-596`) | 17 call sites |
| `BuzzTestClient::connect_unauthenticated` | `:103-106` | — | — | 1 (`:84-86`) |
| `authenticate_with_nip_oa` | `async fn authenticate_with_nip_oa(&mut self, agent_keys: &Keys, auth_tag: &Tag) -> Result<(), TestClientError>` (`:117-124`) | — | — | 1 (`:87-90`) |
| `send_event` | `async fn send_event(&mut self, event: Event) -> Result<OkResponse, TestClientError>` (`:127-129`) | 13 call sites | — (uses HTTP bridge instead) | 23 call sites |
| `send_text_message` | `async fn send_text_message(&mut self, keys: &Keys, channel_id: &str, content: &str, kind: u16) -> Result<OkResponse, TestClientError>` (`:132-141`) | — | 1 (`:743-746`, inside the mixed-kind-filter test) | 11 call sites |
| `subscribe` | `async fn subscribe(&mut self, sub_id: &str, filters: Vec<Filter>) -> Result<(), TestClientError>` (`:144-157`) | 8 call sites | 6 call sites | — |
| `collect_until_eose` | `async fn collect_until_eose(&mut self, sub_id: &str, timeout_dur: Duration) -> Result<Vec<Event>, TestClientError>` (`:174-198`) | 8 call sites | 5 call sites (plus 1 inside `await_scheduler_push`, `:1104-1106`) | — |
| `recv_event` | `async fn recv_event(&mut self, timeout_dur: Duration) -> Result<RelayMessage, TestClientError>` (`:167-172`) | — | 4 call sites (`:648`, `:915`, `:1035`, plus inside `await_scheduler_push` `:1120`) | — |
| `send_raw` | `async fn send_raw(&mut self, value: &serde_json::Value) -> Result<(), TestClientError>` (`:161-163`) | — | 1 (`:1015`, raw `["COUNT", ...]` frame) | — |
| `disconnect` | `async fn disconnect(self) -> Result<(), TestClientError>` (`:201-204`) | 8 call sites | 7 call sites | 36 call sites |
| `close_subscription` | `:159-163` | — | — | — |

`close_subscription` is defined by the harness but **not called by any of the four files in this group** — none of them explicitly closes a subscription before disconnecting; they rely on `disconnect()` tearing down the whole WebSocket.

`e2e_git.rs` imports none of `buzz_test_client` — it has no `use buzz_test_client` line at all (confirmed: `grep -n '^use ' crates/buzz-test-client/tests/e2e_git.rs` lists only `std::path`, `std::process::Command`, `std::time::Duration`, `nostr::{...}`, `s3::{...}`). It talks to the relay exclusively over raw HTTP (`reqwest::Client` posting to `/events`, `e2e_git.rs:47-59`) and drives `git` as a subprocess — it is the one file in this group that does **not** exercise the shared WebSocket test-client API at all.

`e2e_event_reminder.rs` also re-exports and uses `buzz_test_client::TestClientError` directly in a match arm (`Err(buzz_test_client::TestClientError::Timeout) => {}`, `:929`), and imports `RelayMessage` from the harness's re-export (`use buzz_test_client::{BuzzTestClient, RelayMessage}`, `:21`) to pattern-match `RelayMessage::Closed { .. }` (`:648-660`, `:1035-1049`) and `RelayMessage::Event { .. }` (`:934-938`, inside `await_scheduler_push` `:1121-1125`).

`e2e_human_edit_agent_content.rs` additionally imports `buzz_sdk::nip_oa` (`:21`) directly — this is the only file in the group that reaches into `buzz-sdk` rather than only `buzz_test_client` + `nostr`. It calls `nip_oa::compute_auth_tag` and `nip_oa::parse_auth_tag` (both documented in `crates/buzz-sdk/src/nip_oa.rs:139-160` and `:238-260` respectively).

#### Each file's own local helpers (not exported, not reusable by other test files)

None of these helpers are `pub` — each file's helpers are private free functions scoped to that test binary, so no other test file in the crate can import them. They are effectively single-file utility duplication (see Debt for the cross-file overlap this creates).

`e2e_long_form.rs`:
- `relay_url() -> String` (`:28-30`)
- `sub_id(name: &str) -> String` (`:32-34`)
- `build_long_form_event(keys, d_tag, title, content, extra_tags) -> nostr::Event` (`:37-48`)

`e2e_event_reminder.rs`:
- `relay_url() -> String` (`:28-30`), `relay_http_url() -> String` (`:32-38`), `sub_id(name: &str) -> String` (`:40-42`), `http_client() -> Client` (`:44-49`)
- `build_reminder(keys, d_tag, extra_tags) -> nostr::Event` (`:52-64`)
- `build_reminder_at(keys, d_tag, created_at, extra_tags) -> nostr::Event` (`:70-86`)
- `submit_event_http(client, keys, event) -> (bool, String)` (`:92-116`) — POSTs to `/events` with an `X-Pubkey` header, itself a from-scratch reimplementation of what `BuzzTestClient::send_event` does over WebSocket
- `query_events_http(client, pubkey_hex, filters) -> Vec<Value>` (`:118-136`) — POSTs to `/query`
- `count_events_http(client, pubkey_hex, filters) -> Result<u64, (u16, String)>` (`:138-160`) — POSTs to `/count`
- `has_d_tag(event, d_tag) -> bool` (`:1087-1093`)
- `await_scheduler_push(ws, sub_id, d_tag, timeout_dur) -> Result<(), String>` (`:1098-1140`) — the only helper in this group that composes multiple harness calls (`collect_until_eose` then a manual `recv_event` polling loop) into a higher-level assertion primitive

`e2e_human_edit_agent_content.rs`:
- `relay_url() -> String` (`:25-27`), `relay_http_url() -> String` (`:29-35`)
- `create_agent_owned_channel(agent_keys) -> String` (`:38-72`) — raw HTTP POST to `/events` for a kind:9007 open channel
- `create_private_agent_owned_channel(agent_keys) -> String` (`:591-624`) — same shape, `visibility: "private"`
- `make_nip_oa_auth_tag(owner_keys, agent_keys) -> Tag` (`:74-79`)
- `connect_agent_with_owner(agent_keys, owner_keys) -> BuzzTestClient` (`:82-91`) — the one helper that composes two harness calls (`connect_unauthenticated` + `authenticate_with_nip_oa`) into a named setup step

`e2e_git.rs`:
- `relay_http_url() -> String` (`:28-30`)
- `credential_helper() -> PathBuf` (`:34-42`) — resolves `GIT_CREDENTIAL_NOSTR_BIN` or derives `<workspace>/target/release/git-credential-nostr`
- `post_event(event: &nostr::Event)` (`:47-59`) — raw HTTP POST to `/events`, this file's equivalent of `submit_event_http`
- `git_status(args, cwd, owner_nsec) -> std::process::Output` (`:65-90`) and `git(args, cwd, owner_nsec) -> String` (`:95-105`) — subprocess wrappers around the real `git` binary, configured with `credential.helper=<path>`
- `GitS3Probe` struct with `from_env()` (`:117-136`), `pointer_key()` (`:141-148`), `pointer()` (`:152-169`), `require_pointer()` (`:172-185`), `assert_manifest_exists()` (`:186-193`) — a hand-rolled S3 read client using `rust-s3` directly, independent of the relay's own `buzz-media`/git-store S3 client
- `TempDir` struct + `tempdir()`/`tempdir_named(prefix)` (`:454-472`) — RAII temp-directory cleanup

#### External crate APIs each file leans on directly (beyond the harness)

- `nostr::{EventBuilder, Keys, Kind, Tag, ...}` — all four files build events by hand with `EventBuilder::new(...).tags(...).sign_with_keys(...)`. None of the four uses a higher-level builder from `buzz-sdk`'s typed event builders for the *content* kinds (long-form, reminder, edit/delete commands, repo announcement) — only `e2e_human_edit_agent_content.rs` touches `buzz-sdk`, and only for the NIP-OA `nip_oa` module, not an event builder.
- `reqwest::Client` — used directly (not through the harness) by `e2e_event_reminder.rs` (all write/query/count paths), `e2e_human_edit_agent_content.rs` (channel-creation helpers), and `e2e_git.rs` (`post_event`). `e2e_long_form.rs` is the only file in this group that never touches HTTP — it is pure WebSocket via the harness.
- `s3`/`rust-s3` (`Bucket`, `Region`, `Credentials`) — used only by `e2e_git.rs`, for the direct manifest-pointer probe described in Data Model.
- `uuid::Uuid` — used by all four for generating unique `d`-tags, channel UUIDs, and repo-name suffixes.
