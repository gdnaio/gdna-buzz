## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Configuration

#### Every env var read, across all six files

Verified exhaustively via `grep -n 'std::env::var' <each file>` — this is the complete set, no others exist in this group.

| Env var | Default (if unset) | Read at | Files | Documented in `TESTING.md`? | Documented in `.env.example`? |
|---|---|---|---|---|---|
| `RELAY_HTTP_URL` | `http://localhost:3000` | `e2e_media.rs:26`, `e2e_media_extended.rs:13`, `e2e_media_video.rs:19` | 3 media files | **No** — `TESTING.md`'s configuration-reference table documents `RELAY_URL` (relay-side advertised URL) but never `RELAY_HTTP_URL` (this crate's client-side override); the module doc comments of these same files are the only place this variable is explained (`e2e_media.rs:13-16`) | No |
| `RELAY_URL` | `ws://localhost:3000` | `e2e_managed_agent.rs:33`, `e2e_mesh_llm.rs:49`, `e2e_persona.rs:26` | 3 non-media files | Yes — `TESTING.md`'s configuration table documents `RELAY_URL` as "Advertised in NIP-11 / NIP-42 challenges. **Note: no `BUZZ_` prefix.**" (relay-side meaning); these three tests instead read it client-side as *where to connect*. Same variable name, two different roles (relay config vs. test-client target) — see Debt | No |
| `TEST_IMAGE_PATH` | none — test prints "Skipping" and returns if unset | `e2e_media.rs:314` | `e2e_media.rs` only | No | No |
| `MEMBER_NSEC` | none — test skips (returns without asserting) if unset or empty; panics if set but unparseable | `e2e_mesh_llm.rs:57` (via `keys_from_env`), consumed `e2e_mesh_llm.rs:95` | `e2e_mesh_llm.rs` | No — not in `TESTING.md`'s config table, not in the top-level aggregated `configuration.md`'s Mesh section either | No |
| `STRANGER_NSEC` | same skip/panic semantics as `MEMBER_NSEC` | `e2e_mesh_llm.rs:57`, consumed `e2e_mesh_llm.rs:182` | `e2e_mesh_llm.rs` | No | No |
| `MESH_OPENAI_BASE` | none — test prints "SKIP: ..." and returns if unset | `e2e_mesh_llm.rs:229` | `e2e_mesh_llm.rs` | No (not in `TESTING.md`), but **yes** at the aggregated top-level `configuration.md`'s "Mesh (opt-in `mesh-llm` / relay mesh)" section, which lists it alongside `BUZZ_MESH`, `MESH_ROLE`, etc. — that list is a cross-crate summary from a prior analysis pass, not `TESTING.md`/`.env.example` itself | No |

No CLI flags are read by any file in this group — `grep -rn 'std::env::args\|clap::' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs crates/buzz-test-client/tests/e2e_managed_agent.rs crates/buzz-test-client/tests/e2e_mesh_llm.rs crates/buzz-test-client/tests/e2e_persona.rs` returns zero matches; test selection and filtering happen entirely through `cargo test`'s own argument parsing (test-name substring filters like `-- trust`, `-- live_agent_completes`), not anything these files parse themselves.

#### Parsed-but-never-read: none found

Every env var this group reads is read exactly once (a single `std::env::var` call per variable per file) and its result is used immediately in the same function — there is no case of a variable being parsed into a struct field and then never consulted. This is a meaningfully different pattern from, e.g., `buzz-agent`'s `Config` struct (per the prior `agent-llm` analysis, which found several `Config` fields parsed but exercised only defensively) — these test files have no persistent config object at all; each helper function re-reads the environment on every call.

#### Cross-file naming inconsistency: `RELAY_URL` vs `RELAY_HTTP_URL`

The three media files use `RELAY_HTTP_URL` while the three non-media files (`e2e_managed_agent.rs`, `e2e_mesh_llm.rs`, `e2e_persona.rs`) use `RELAY_URL` for what is conceptually the same purpose (where is the relay). This is not a distinction between HTTP and WebSocket base URLs in practice — `e2e_persona.rs` derives its HTTP URL from `RELAY_URL` via a scheme rewrite (`relay_http_url()`, `e2e_persona.rs:31-37`: `ws://`→`http://`, `wss://`→`https://`), and `e2e_media_extended.rs`/`e2e_media_video.rs` derive their WS URL from `RELAY_HTTP_URL` the same way in reverse (`e2e_media_extended.rs:16-20`, `e2e_media_video.rs`'s equivalent). So a caller who exports only `RELAY_URL=ws://localhost:3030` (per `TESTING.md`'s own port-override example, "In your working / CLI terminal... export BUZZ_RELAY_URL=http://localhost:3030") would have `e2e_persona.rs` correctly target port 3030, but `e2e_media.rs`/`e2e_media_extended.rs`/`e2e_media_video.rs` would silently fall back to the hardcoded default `http://localhost:3000` — a real footgun for anyone following `TESTING.md`'s override instructions and then running the media test files, with no error, just silent misdirection to the wrong port. Confirmed by reading both `relay_http_url()`/`relay_url()` implementations directly (`e2e_media.rs:25-27` vs. `e2e_persona.rs:22-24`).

#### Relay-side configuration this group's tests implicitly depend on (read by `buzz-relay`, not by these test files, but required for the tests to behave as documented)

These are not read by any of the six files — they are documented in `.env.example`/`TESTING.md` as relay-side settings whose *value* determines whether the assertions in this group hold:

| Relay var | Relevance to this group | Default |
|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | If `true`, the NIP-98 `X-Pubkey` dev fallback that `e2e_persona.rs`'s HTTP-bridge helpers rely on (`submit_event_http`, `query_events_http`, `count_events_http`) stops working — none of these tests sets this var, so they only prove behavior under the default (`false`) | `false` (`.env.example`, `TESTING.md` config table) |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `e2e_mesh_llm.rs`'s module doc explicitly requires "a membership-gated buzz-relay" (`:1-3`); no test in this group sets the var itself, it must be set on the relay process externally | `false` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` (`.env.example`) | If `true`, `GET`/`HEAD /media/*` would require Blossom `t=get` auth — none of the media files in this group send that auth on their `GET` calls (confirmed: no `sign_blossom_auth` call with `t=get`, only `t=upload` is ever constructed, e.g. `e2e_media.rs:46`), so every `GET`/`HEAD` test in this group (`e2e_media.rs:150-182`, `e2e_media_video.rs:322-408`) would fail under a relay with this flag turned on. Not mentioned by any of the three media files' module docs — see Debt | `false` |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS`, `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY`, `BUZZ_MEDIA_UPLOADS_PER_MINUTE` (`.env.example`) | Admission limits that could make `test_concurrent_upload_same_file` (`e2e_media_extended.rs:495-521`, which fires 2 simultaneous uploads) flaky under a tightly configured relay; the test hardcodes an expectation of "both succeed" with no accounting for these limits | 8 / 2 / 30 (commented defaults in `.env.example`) |

None of these relay-side variables is set by the six test files themselves (they only ever configure the *client* side — which URL to hit, which key to use) — they are the operator's responsibility per `TESTING.md`'s setup instructions, and this group's tests silently assume the relay's default posture unless a human configures otherwise.
