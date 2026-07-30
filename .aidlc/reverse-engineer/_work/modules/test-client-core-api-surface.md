## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: API Surface

This group provides two kinds of API surface: the harness's own public Rust API (`src/lib.rs`, consumed by every test file in the crate, not just this group's), and the two standalone CLI binaries (`buzz-test-cli` via `src/main.rs`, `wamp-bench`, `mention`). The four test files are pure consumers of both the harness and the live relay's wire surface — they expose nothing themselves.

#### 1. `BuzzTestClient` public API (`src/lib.rs`)

| Method | Signature | Purpose |
|---|---|---|
| `connect` | `connect(url: &str, keys: &Keys) -> Result<Self, TestClientError>` (`lib.rs:94-98`) | Connect + full NIP-42 authenticate in one call — the overwhelming majority call site across this group |
| `connect_unauthenticated` | `connect_unauthenticated(url: &str) -> Result<Self, TestClientError>` (`lib.rs:101-105`) | Connect only, skip AUTH — used once in this group, by `test_unauthenticated_rejected` (`e2e_relay.rs:756-786`) |
| `authenticate` | `authenticate(&mut self, keys: &Keys) -> Result<(), TestClientError>` (`lib.rs:108-111`) | Standalone NIP-42 auth on an already-open connection; not called directly by any file in this group (only reached via `connect`) |
| `authenticate_with_nip_oa` | `authenticate_with_nip_oa(&mut self, agent_keys: &Keys, auth_tag: &Tag) -> Result<(), TestClientError>` (`lib.rs:114-121`) | NIP-OA owner-delegated auth — **not used by any file in this group** (used only by `e2e_human_edit_agent_content.rs`, outside this group's scope) |
| `send_event` | `send_event(&mut self, event: Event) -> Result<OkResponse, TestClientError>` (`lib.rs:124-126`) | Raw signed-event submission; used pervasively for every non-`h`-tagged or custom-tagged event in `e2e_relay.rs`, `e2e_team.rs` |
| `send_text_message` | `send_text_message(&mut self, keys: &Keys, channel_id: &str, content: &str, kind: u16) -> Result<OkResponse, TestClientError>` (`lib.rs:129-141`) | Convenience wrapper that builds the `["h", channel_id]`-tagged event; used by `e2e_relay.rs` for the majority of kind:9 sends, and by `wamp_bench.rs`'s load loop |
| `subscribe` | `subscribe(&mut self, sub_id: &str, filters: Vec<Filter>) -> Result<(), TestClientError>` (`lib.rs:144-157`) | Sends `["REQ", sub_id, filter, ...]`; used in every file except `nip42_host_binding_live.rs` and the bin targets |
| `close_subscription` | `close_subscription(&mut self, sub_id: &str) -> Result<(), TestClientError>` (`lib.rs:160-163`) | Sends `["CLOSE", sub_id]`; used exactly once in this group, `test_close_subscription_stops_delivery` (`e2e_relay.rs:788-822`) |
| `send_raw` | `send_raw(&mut self, value: &serde_json::Value) -> Result<(), TestClientError>` (`lib.rs:166-169`) | Escape hatch for hand-built frames; **not used by any file in this group** — `main.rs` doesn't use it either. `nip42_host_binding_live.rs` bypasses the harness entirely and builds raw frames directly over `tokio_tungstenite` instead (see Integrations) |
| `recv_event` | `recv_event(&mut self, timeout_dur: Duration) -> Result<RelayMessage, TestClientError>` (`lib.rs:172-177`) | Single-message receive with timeout; used heavily in `e2e_relay.rs` for live-fanout assertions |
| `collect_until_eose` | `collect_until_eose(&mut self, sub_id: &str, timeout_dur: Duration) -> Result<Vec<Event>, TestClientError>` (`lib.rs:180-202`) | Drains historical `EVENT`s until `EOSE`; used in every REQ-based test across `e2e_relay.rs`, `e2e_user_status.rs`, `e2e_team.rs` |
| `disconnect` | `disconnect(self) -> Result<(), TestClientError>` (`lib.rs:243-246`) | Graceful WS close; called at the end of virtually every test |

Of 11 public methods, this group's four test files exercise 9; `authenticate_with_nip_oa` and `send_raw` are dead from this group's perspective (see Debt for whether they're used anywhere in the crate).

#### 2. `buzz-test-cli` (`src/main.rs`) — manual testing CLI

A single flat binary (`[[bin]] name = "buzz-test-cli"`, `Cargo.toml:34-36`) with five flags parsed by a hand-rolled loop (`parse_args`, `main.rs:169-211`), not `clap`:

| Flag | Effect |
|---|---|
| `--url <URL>` | Relay WS URL, default `ws://localhost:3000` |
| `--send <MESSAGE>` | Send one text message and exit |
| `--channel <ID>` | Channel ID for send/subscribe, default `"default"` (a literal string, not validated as a UUID — see Debt) |
| `--subscribe` | Subscribe to a channel/kind and print events until Ctrl+C or a `CLOSED` |
| `--kind <KIND>` | Event kind for send/subscribe, default `9` |
| `--help` / `-h` | Print usage and exit 0 |

Identity comes from `BUZZ_PRIVATE_KEY` if set, else an ephemeral `Keys::generate()` (`main.rs:37-40`) — so running the tool without exporting a key silently authenticates as a throwaway identity every invocation, which is fine for smoke-testing but means two consecutive runs never see each other's channel membership unless the channel is `open`. Exit codes are `0` (help), `1` (bad args, connect failure, send rejection) — there's no distinct exit code for "subscribe loop broke" vs. "send failed," unlike the richer exit-code contract `AGENTS.md` documents for `buzz-cli` (0/1/2/3/4/5). This is a separate, much smaller tool and the two are not meant to share a contract, but the discrepancy is worth flagging for anyone scripting against `buzz-test-cli` expecting `buzz`-style codes.

#### 3. `wamp-bench` (`src/bin/wamp_bench.rs`) — write-amplification load generator

Positional-only CLI (no flag parsing at all): `wamp-bench <channel_uuid> <qps> <duration_secs> <conns> <latency_out>` (`wamp_bench.rs:15`, enforced by `args.len() != 6` at `:26-29`). Two env vars: `BUZZ_RELAY_URL` (default `ws://localhost:3000`) and `BENCH_PRIVATE_KEY` (hex, **required** — the process exits via `anyhow::bail!` if unset, `:37-39`). Output is a single JSON object on stdout (`sent`, `accepted`, `rejected`, `qps_target`, `duration_secs`, `conns`, `ok_latency_ms: {p50, p95, p99, max}`, `:106-121`) plus one float-per-line latency file at the path given by the final positional arg. There is no `--help`; malformed/missing args just print the one-line usage string and exit 1 — this tool is explicitly an internal benchmarking script, not a user-facing CLI, and its API surface is intentionally minimal.

#### 4. `mention` (`src/bin/mention.rs`) — one-shot @mention sender

Positional-only: `mention <channel_uuid> <target_pubkey_hex> <message...>` (`mention.rs:6`, `args.len() < 4` check at `:11-14`), where the message is `args[3..].join(" ")` — every remaining argument after the first two is rejoined with spaces, so an unquoted multi-word message on the shell works without explicit quoting. Uses a fresh `Keys::generate()` every run (no `BUZZ_PRIVATE_KEY` support at all, unlike `wamp_bench.rs` and `buzz-test-cli`) — printed to stdout so the operator can identify the sender post hoc, but there's no way to run this tool twice as the *same* identity, which limits its use for testing @mention-triggered ACP flows that depend on a stable sender.

#### 5. Relay HTTP/WS surface this group exercises

This is the group's real center of gravity — `e2e_relay.rs` alone drives nearly every documented relay door:

| Door | Method/path | Purpose | First call site |
|---|---|---|---|
| WS | connect + NIP-42 AUTH | universal | `BuzzTestClient::connect`, e.g. `e2e_relay.rs:242` |
| WS | `EVENT` | submit signed events of every kind in the Data Model table | throughout |
| WS | `REQ` / `CLOSE` | subscribe/unsubscribe, including deliberately malformed filters expected to `CLOSED` | throughout |
| WS | `COUNT` | **not exercised anywhere in this group** — no `Count` variant match, no COUNT frame sent by any of the four files (confirmed by grep: zero `"COUNT"` / `RelayMessage::Count` references in `e2e_relay.rs`, `e2e_user_status.rs`, `e2e_team.rs`, `nip42_host_binding_live.rs`) — a real coverage gap given the task brief's expectation that this group covers "EVENT/REQ/COUNT" (see Debt) |
| HTTP | `POST /events` | channel creation, NIP-29 admin commands, kind:0 profile, forged relay-only-kind probe, membership add/remove | `create_test_channel` (`:167-197`), `test_client_submitted_nip43_membership_snapshots_are_rejected` (`:216-244`), many more |
| HTTP | `POST /query` | kind:0 profile verification round-trip | `test_kind0_nip05_sync` (`:500-520`) |
| HTTP | `GET /info` | NIP-11 relay info | `test_nip11_relay_info` (`:522-556`) |
| HTTP | `GET /.well-known/nostr.json?name=...` | NIP-05 resolution | `test_kind0_nip05_sync` (`:540-560`) |
| HTTP | `POST /api/invites` | mint invite code (NIP-98 signed) | `test_invite_mint_and_claim_admits_new_pubkey`, `test_invite_mint_requires_owner_or_admin`, `test_invite_code_minted_for_one_host_fails_on_another` |
| HTTP | `POST /api/invites/claim` | claim invite code (NIP-98 signed) | same tests |

`nip42_host_binding_live.rs` uses a lower-level door than the rest of the group: it drives `tokio_tungstenite::connect_async` directly against two distinct hostnames (`ws://a.localhost:3100`, `ws://b.localhost:3100`) rather than going through `BuzzTestClient` at all, because the property under test is host-scoped connection routing itself — using the harness (which hardcodes no host-selection logic beyond the URL string) would work, but the file was written host-agnostic-harness-free, matching its "live two-host" framing in the module doc comment (`:1-20`).

#### 6. Non-`BuzzTestClient` harness imports

| Import | From | Used by |
|---|---|---|
| `RelayMessage` | `buzz_test_client::RelayMessage` | `e2e_relay.rs` (pattern-matching `Event`/`Eose`/`Closed`/`Notice`), `main.rs` |
| `TestClientError` | `buzz_test_client::TestClientError` | `e2e_relay.rs:26` (matches `TestClientError::Timeout` and `TestClientError::ConnectionClosed`), `main.rs` (matches `TestClientError::Timeout` in the subscribe loop) |

`e2e_user_status.rs` and `e2e_team.rs` import only `BuzzTestClient` itself — neither names `RelayMessage` or `TestClientError` directly, relying on `.expect()` to surface harness failures as panics (the same pattern the sibling `test-client-conformance` group's `conformance_multitenant.rs` uses).

#### 7. What this group does NOT expose or exercise

- No file in this group defines a `#[bin]` target beyond the three already covered (`buzz-test-cli`, `wamp-bench`, `mention`) — there is no library-level public API surface beyond `BuzzTestClient` and its error/message re-exports.
- No COUNT-door coverage (see above) — a real gap relative to the module's stated scope.
- No test in this group calls `authenticate_with_nip_oa` or `send_raw` — both are exercised elsewhere in the crate (outside this group) or, in `send_raw`'s case, effectively unused by any test file in the whole crate at the time of this analysis (see Debt).
