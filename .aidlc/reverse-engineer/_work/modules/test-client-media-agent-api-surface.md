## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: API Surface

#### Harness (`lib.rs`) functions called, per file

| File | `BuzzTestClient` methods called | Direct HTTP (bypassing the harness) |
|---|---|---|
| `e2e_media.rs` | none — no `buzz_test_client` import anywhere in the file | yes, exclusively: raw `reqwest::Client` to `PUT /upload`, `PUT /media/upload`, `GET`/`HEAD /media/{sha}.jpg`, `GET /media/{sha}.thumb.jpg` |
| `e2e_media_extended.rs` | `BuzzTestClient::connect` (`:562`, `:625`, `:692`), `.send_event` (`:582`, `:644`, `:705`), `.disconnect` (`:590`, `:656`, `:709`) — only inside the three `test_ws_*` functions | yes, for all upload/auth tests: raw `reqwest::Client` PUT to `/upload` and `/media/upload`, plus a raw channel-creation `POST /events` inside the WS tests (e.g. `:541-556`, `:604-619`, `:669-684`) |
| `e2e_media_video.rs` | `BuzzTestClient::connect` (`:531`, `:616`), `.send_event` (`:555`, `:638`), `.disconnect` (`:562`, `:644`) — only inside the two poster-imeta tests | yes, for all upload/range tests: raw `reqwest::Client` PUT/GET to `/upload`, `/media/upload`, plus range-header GETs |
| `e2e_managed_agent.rs` | `BuzzTestClient::connect` (`:120`, `:165`, `:192`, `:270`, `:319`), `.send_event` (called on every connected `client`, e.g. `:123`, `:167`, `:195`), `.subscribe`/`.collect_until_eose` (e.g. `:137-141`, `:339-345`), `.disconnect` (e.g. `:151`, `:180`) | none — every request goes through the harness |
| `e2e_mesh_llm.rs` | `BuzzTestClient::connect` (`:98`, `:185`), `.subscribe` (`:104`, `:193`), `.collect_until_eose` (`:108`, `:197`), `.disconnect` (`:170`, `:209`) — only in the two `trust_*` tests; `live_agent_completes_chat_over_mesh` and `live_split_model_completes` never touch the harness at all | yes, for the two `#[ignore]`d live/manual tests only: `reqwest::Client` `GET {base}/models` and `POST {base}/chat/completions` (`:236-260`), used solely by `live_agent_completes_chat_over_mesh` — no harness involvement in the mesh chat-completion path |
| `e2e_persona.rs` | `BuzzTestClient::connect` (18 call sites, e.g. `:168`, `:216`, `:260`, `:294`), `.send_event` (18+ sites), `.subscribe` (16+ sites), `.collect_until_eose` (16+ sites), `.disconnect` (18+ sites), `.send_raw` (`:816`, for a raw `["COUNT", ...]` frame), `.recv_event` (`:820`, `:896`, `:937`, `:979` — the latter three polling a live subscription for absence-of-event, not just the COUNT reply) | yes, via three **locally-defined** wrapper fns that call the NIP-98 HTTP bridge directly: `submit_event_http` (`:45-64`), `query_events_http` (`:68-85`), `count_events_http` (`:88-109`) — these do not go through `BuzzTestClient` at all, they build their own `reqwest` calls to `POST /events`, `POST /query`, `POST /count` |

Methods from `lib.rs` **never** called by any file in this group: `connect_unauthenticated`, `authenticate` (called internally by `connect`, never invoked standalone here), `authenticate_with_nip_oa`, `send_text_message`, `close_subscription`. All six are declared `pub` on `BuzzTestClient` (`crates/buzz-test-client/src/lib.rs:97-186`) but this group only exercises `connect`, `send_event`, `subscribe`, `collect_until_eose`, `disconnect`, `send_raw`, and `recv_event` — 7 of the crate's 12 public methods.

#### Reusable helpers exposed by these files

None. Every one of the six files is a `tests/*.rs` integration-test binary, each compiled as an independent crate by Cargo — nothing declared in one is importable by another (no `tests/common/` shared module exists in this crate; `grep -rn 'mod common' crates/buzz-test-client/tests/` returns zero matches). Each file re-derives its own private helpers:

| Helper | Duplicated in | Signature |
|---|---|---|
| `relay_http_url()` / `relay_url()` | all six files, under one of two names | `() -> String`, reads `RELAY_HTTP_URL` or `RELAY_URL` |
| `http_client()` | `e2e_media.rs:28-33`, `e2e_media_extended.rs:22-27`, `e2e_media_video.rs:24-29` | `() -> reqwest::Client`, timeout 15s/15s/30s respectively (not identical) |
| `sign_blossom_auth()` | `e2e_media.rs:36-48`, `e2e_media_extended.rs:35-44` (near-identical), `e2e_media_video.rs:38-49` | `(&Keys, &str) -> nostr::Event` |
| `blossom_auth_header()` | `e2e_media.rs:51-57`, `e2e_media_extended.rs:47`, `e2e_media_video.rs:52-57` | `(&nostr::Event) -> String` |
| `tiny_jpeg()` | `e2e_media.rs:60-84`, `e2e_media_extended.rs:63-87`, `e2e_media_video.rs:294-318` — byte-for-byte identical 339-byte literal in all three | `() -> Vec<u8>` |
| `relay_ws_url()` | `e2e_media_extended.rs:19-23`, `e2e_media_video.rs:290-293` | `() -> String`, http→ws scheme rewrite |
| `sub_id(name)` | `e2e_managed_agent.rs:14-16`, `e2e_mesh_llm.rs:76-78`, `e2e_persona.rs:122-124` — same pattern, different prefix | `(&str) -> String` |

No file in this group calls a helper defined in another file in this group or in a sibling group's file — confirmed by the absence of any `include!`, `#[path]`, or cross-`tests/*.rs` `use` statement (Rust integration tests cannot `use` a sibling `tests/*.rs` file's items without one of those, and none is present).

#### External-crate surfaces driven directly (not via the harness)

| Surface | Files | Notes |
|---|---|---|
| `POST /upload`, `PUT /upload` (Blossom BUD-02) | `e2e_media.rs`, `e2e_media_extended.rs`, `e2e_media_video.rs` | canonical route |
| `PUT /media/upload` (legacy alias) | all three media files | `e2e_media_extended.rs` specifically tests this route's *narrower* content-type policy vs. `/upload` (`:420-433`, `:443-451`) |
| `GET`/`HEAD /media/{sha256}.{ext}`, `.../{sha256}.thumb.{ext}` | all three media files | thumbnail and range-request variants only in `e2e_media.rs` and `e2e_media_video.rs` respectively |
| `POST /events` (NIP-98 HTTP bridge) | `e2e_media_extended.rs` (channel creation for WS tests), `e2e_media_video.rs` (same), `e2e_persona.rs` (`submit_event_http`, and `create_test_channel`) | dev-mode `X-Pubkey` header fallback used throughout (no NIP-98 signature actually computed — see Security) |
| `POST /query`, `POST /count` (NIP-98 HTTP bridge) | `e2e_persona.rs` only | `query_events_http`, `count_events_http` |
| Raw WebSocket `["COUNT", sub_id, filter]` frame | `e2e_persona.rs:801` | the only place in this group that bypasses `BuzzTestClient::subscribe`'s REQ-only framing and hand-builds a different message type via `send_raw` |
| `GET {mesh}/models`, `POST {mesh}/chat/completions` | `e2e_mesh_llm.rs` | OpenAI-compatible surface exposed by a mesh client node, not by `buzz-relay` itself |
