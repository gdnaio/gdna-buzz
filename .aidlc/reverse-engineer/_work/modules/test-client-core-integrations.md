## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Integrations

#### Upstream crate dependencies

Per `Cargo.toml:11-25` (`[dependencies]`), the library/bin surface (`src/lib.rs`, `src/main.rs`, both bin targets) depends on: `anyhow`, `buzz-core`, `buzz-ws-client`, `nostr`, `tokio`, `tokio-tungstenite`, `futures-util`, `serde`/`serde_json`, `tracing`/`tracing-subscriber`, `thiserror`, `uuid`, `url`, and a pinned `rustls = "0.23"`. Despite `buzz-core` being a listed dependency, no file in this group imports anything from it directly (confirmed by grep — zero `buzz_core::` references across all eight files); it's present in the manifest for the crate as a whole but this group doesn't exercise that edge.

`[dev-dependencies]` (`Cargo.toml:27-38`) add `reqwest`, `base64`, `hex`, `rand`, `sha2`, `sqlx`, `chrono`, `rust-s3` (aliased `s3`), and `buzz-sdk` — all of these are used by *other* test files in the crate (media, git, event-reminder groups), and this group's `e2e_relay.rs` uses exactly two of them: `reqwest` (every HTTP-door call) and `sha2`/`hex`/`base64` (building the NIP-98 auth header for invite tests) plus `sqlx` (direct Postgres access for test fixture seeding — see below). `e2e_user_status.rs`, `e2e_team.rs`, and `nip42_host_binding_live.rs` use none of the dev-dependencies beyond `uuid` and `nostr`/`buzz_test_client` themselves.

#### `buzz-ws-client` — the actual transport layer underneath `BuzzTestClient`

`BuzzTestClient` (`lib.rs:87-89`) does not implement any WebSocket logic itself — it holds one field, `inner: NostrWsConnection`, and every method on `BuzzTestClient` is a thin pass-through to the identically-named (or near-identically-named) method on `NostrWsConnection` (`crates/buzz-ws-client/src/connection.rs:26-30`):

| `BuzzTestClient` method | Delegates to |
|---|---|
| `connect_unauthenticated` | `NostrWsConnection::connect` (`connection.rs:47-63`) |
| `authenticate` / `authenticate_with_nip_oa` | `NostrWsConnection::authenticate` (`connection.rs:67-89`), with `auth_tag: Option<&Tag>` threaded through unchanged |
| `send_event` | `NostrWsConnection::send_event` (`connection.rs:92-97`) |
| `subscribe` / `close_subscription` / `send_raw` | all three route through `NostrWsConnection::send_raw` (`connection.rs:120-125`) |
| `recv_event` | `NostrWsConnection::next_event` (`connection.rs:100-108`) |
| `disconnect` | `NostrWsConnection::disconnect` (`connection.rs:111-114`) |

The `VecDeque<RelayMessage>` message-interleaving buffer that `ARCHITECTURE.md` attributes to `BuzzTestClient` actually lives inside `NostrWsConnection` (`connection.rs:28`, field `buffer`) — it exists to let `wait_for_auth_challenge` and `wait_for_ok` push back any out-of-order frame (e.g., a stray `NOTICE` or a second `AUTH` challenge arriving while waiting for an `OK`) so `next_event`/`recv_one` can drain it before reading a fresh frame off the socket. This buffering is what lets tests in this group call `recv_event`/`collect_until_eose` without themselves worrying about frame interleaving — a genuinely useful integration point, just implemented one layer below where the architecture doc says it is (see Debt).

`build_auth_event` (`buzz-ws-client/src/message.rs:163-183`) is the function that actually constructs the NIP-42 AUTH event, including the optional NIP-OA `auth` tag splice — `authenticate_with_nip_oa` (unused by this group, but part of the shared surface) exists purely to thread that optional tag through.

#### `nostr` crate — event construction and signing

Every test file in this group builds events via `nostr::EventBuilder`, `nostr::Keys`, `nostr::Kind`, `nostr::Tag`, and filters via `nostr::Filter`/`nostr::SingleLetterTag`/`nostr::Alphabet`. This group makes no use of `nostr`'s NIP-44 encryption helpers (unlike the sibling content/media groups' event-reminder and DM tests) — every event's `.content` in this group is either empty, plain text, or plaintext JSON.

`nip42_host_binding_live.rs` uses `nostr::EventBuilder::auth` and `nostr::RelayUrl` directly (bypassing `buzz-ws-client`'s `build_auth_event` wrapper) because the whole point of the file is to control the `relay` tag's value independently of the connection's actual host — it needs the low-level constructor, not the harness's opinionated one.

#### `tokio-tungstenite` / `futures-util` — raw WebSocket access in `nip42_host_binding_live.rs`

This is the one file in this group that does not go through `BuzzTestClient` or `NostrWsConnection` at all. `do_auth_with_relay_tag` (`:23-92`) calls `tokio_tungstenite::connect_async` directly and manually drives the AUTH handshake with `SinkExt::send`/`StreamExt::next`, parsing frames with raw `serde_json::from_str::<Value>`. This is a deliberate integration choice: the harness's `authenticate` method always signs the AUTH event for the connection's own URL, but this file needs to sign for an **independently chosen** URL that may deliberately mismatch the connection's actual host — a capability the harness doesn't (and shouldn't) expose as a first-class parameter.

#### `reqwest` — HTTP-door integration in `e2e_relay.rs`

Every HTTP-door test builds its own `reqwest::Client` inline (no shared client, no connection pooling across tests) and constructs requests by hand: `POST /events` with either an `X-Pubkey` header (dev-mode shortcut) or a hand-built `Authorization: Nostr <base64 NIP-98 event>` header (`nip98_post_header`, `:60-72`, real production-shaped auth), `POST /query`, `GET /info`, `GET /.well-known/nostr.json`, `POST /api/invites`, `POST /api/invites/claim`. There is no typed request/response client for these endpoints anywhere in this group — every response is deserialized ad hoc into `serde_json::Value` and indexed with `["field"]` / `.get("field")`, which is consistent with the crate's stated purpose as a black-box integration harness rather than a typed SDK (that role belongs to `buzz-sdk` and `buzz-cli`'s `client.rs`, neither of which this group touches).

#### `sqlx` — direct Postgres fixture seeding, bypassing the relay entirely

`e2e_relay.rs` is the only file in this group that talks to Postgres directly, via `e2e_db_pool()` (`:74-82`, connecting to `DATABASE_URL` or the dev default `postgres://buzz:buzz_dev@localhost:5432/buzz`) and two seeding helpers:

- `ensure_test_community(host)` (`:84-100`) — `INSERT INTO communities (id, host) ... ON CONFLICT (lower(host)) DO NOTHING`, then looks the row up by host.
- `seed_relay_member(host, keys, role)` / `seed_relay_owner(keys)` (`:102-125`) — `INSERT INTO relay_members (community_id, pubkey, role, added_by) ... ON CONFLICT ... DO UPDATE`.

This is a real, deliberate side-channel: the invite-mint tests need a pre-existing `owner`/`admin` relay member before the HTTP door will let them mint an invite, and there's no event-driven or REST path in this crate's harness to establish that role from a cold start — the test reaches around the relay's own API and writes the row directly. This makes those four invite tests (`test_invite_mint_and_claim_admits_new_pubkey`, `test_invite_mint_requires_owner_or_admin`, `test_invite_code_minted_for_one_host_fails_on_another`, and transitively `test_client_submitted_nip43_membership_snapshots_are_rejected`'s dependency on `create_test_channel`) load-bearing on Postgres schema knowledge (`communities`, `relay_members` table/column names) staying in sync with `buzz-db`'s actual migrations — a schema rename in `buzz-db` would silently break these tests at the SQL level rather than at the Rust type level, since `sqlx::query` (runtime, not `sqlx::query!` compile-time-checked) is used throughout, matching the crate-wide "no sqlx offline cache" convention `ARCHITECTURE.md` documents for `buzz-db` itself.

#### Relay-side integration surface (what this group's tests actually reach into)

Cross-referenced directly against `crates/buzz-relay/src` in this pass:

| Relay module | What this group's tests exercise there |
|---|---|
| `handlers/auth.rs` | NIP-42 verification, ban-cascade gate, allowlist gate, relay-membership gate (via `connect`/`connect_unauthenticated` and the host-binding file) |
| `api/bridge.rs` | `nip42_expected_relay_url` (host-binding), NIP-98 header verification (invite mint/claim), dev-mode `X-Pubkey` fallback (`create_test_channel`, most `POST /events` calls) |
| `handlers/event.rs` | full EVENT pipeline: auth check, pubkey-match, relay-only-kind gate, AUTH-kind gate, membership-notification-kind gate |
| `handlers/ingest.rs` | kind-specific validation (`is_relay_only_kind`, `KIND_AUTH` rejection, membership-notification rejection), channel creation (kind:9007) |
| `handlers/side_effects.rs` | NIP-29 admin command validation (`validate_admin_event`'s 9000/9002/9005/9007/9008 branches), `channel_add_policy` third-party-add gate, archive/unarchive resubscribe-notification emission |
| `handlers/req.rs` | `MAX_SUBSCRIPTIONS` enforcement, membership-notification `#p`-filter gating |
| `subscription.rs` | fan-out correctness (indirectly, via multi-client broadcast tests) |
| `nip11.rs` | `GET /info` payload shape and `limitation` values |
| `api/invites.rs` | mint/claim HTTP endpoints, rate limiting (not directly probed by this group — no test sends 10+ claims to trigger `CLAIM_RATE_LIMIT`) |

#### What this group does NOT integrate with

- No Redis/`buzz-pubsub` integration is exercised directly — multi-client fan-out tests (`test_multiple_concurrent_clients`) prove the *observable* behavior but stay within a single relay process' local fan-out path; nothing in this group starts two relay instances to prove the documented Redis-backed multi-node fan-out actually crosses processes.
- No `buzz-search`/FTS integration — no NIP-50 `search` filter appears anywhere in this group (that's the sibling `test-client-conformance` group's territory).
- No media/Blossom, no git/manifest, no workflow-engine integration — all out of scope by design (see AGENTS.md's crate boundaries; those live in `test-client-media-agent` and `test-client-content`).
- No `buzz-acp`/agent-harness process integration — `mention.rs` is clearly *designed* to trigger an ACP-observed `@mention`, but nothing in this group spawns or asserts against an actual `buzz-acp` process; the tool only proves the relay accepts the mention event, not that an agent picks it up.
