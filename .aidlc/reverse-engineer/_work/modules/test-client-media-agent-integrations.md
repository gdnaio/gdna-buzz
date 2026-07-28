## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Integrations

#### Real subprocesses/services required, beyond a bare relay

| File | Beyond `buzz-relay` itself | Evidence |
|---|---|---|
| `e2e_media.rs` | MinIO (relay's media backend) — stated as a hard requirement in the module doc | `e2e_media.rs:3` ("Requires: relay running at localhost:3000, MinIO running at localhost:9000") |
| `e2e_media_extended.rs` | MinIO, implied by the same upload/GET pattern; no explicit doc-comment restatement, but every test PUTs through `/upload` which the relay backs with the same media store | inferred from shared upload helpers, no independent statement |
| `e2e_media_video.rs` | MinIO, explicitly stated | `e2e_media_video.rs:3` |
| `e2e_managed_agent.rs` | none beyond the relay — no agent subprocess is spawned; "managed-agent" here means a relay-stored event describing an agent config, not a live process | module doc `:8-16` explicitly disclaims reaching a real agent process |
| `e2e_mesh_llm.rs` | Two live desktop mesh nodes (one serving a model, one running a mesh client) plus a downloaded model, for the `live_*` tests; a membership-gated relay plus real `MEMBER_NSEC`/`STRANGER_NSEC` identities for the `trust_*` tests | module doc `:1-26` |
| `e2e_persona.rs` | none beyond the relay | no MinIO/Postgres/Redis dependency stated or exercised — persona content is opaque JSON stored as an ordinary event |

None of these six files spins up MinIO, Postgres, or Redis itself — that is the operator's job (`just setup`/`docker compose`), not the test binary's. Confirmed by `grep -rn 'docker\|testcontainers\|Command::new' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs crates/buzz-test-client/tests/e2e_managed_agent.rs crates/buzz-test-client/tests/e2e_mesh_llm.rs crates/buzz-test-client/tests/e2e_persona.rs` → zero matches. Unlike the sibling `e2e_git.rs` (owned by a different analysis group), which the top-level `api-surface.md:3322-3323` notes drives a real `git` CLI and reads the S3 pointer directly, none of the six files in this group shells out to an external CLI or touches S3/MinIO's API directly — they only speak HTTP/WebSocket to the relay, which is itself responsible for the MinIO round-trip.

#### Crates each file actually depends on

All six files draw from the same core set declared in `crates/buzz-test-client/Cargo.toml` (`[dependencies]` + `[dev-dependencies]`):

| Crate | Used by | Purpose |
|---|---|---|
| `nostr` | all six | `EventBuilder`, `Keys`, `Kind`, `Tag`, `Filter`, `Timestamp`, `SingleLetterTag`, `Alphabet` — event construction/signing |
| `reqwest` | all six | HTTP client for every non-WS call (upload PUT/GET, `/events`, `/query`, `/count`) |
| `tokio` (`#[tokio::test]`) | all six | async test runtime |
| `buzz_test_client` (this crate's own `lib.rs`) | `e2e_media_extended.rs`, `e2e_media_video.rs` (WS-imeta / poster-imeta tests only), `e2e_managed_agent.rs`, `e2e_mesh_llm.rs`, `e2e_persona.rs` — **not** `e2e_media.rs` | `BuzzTestClient` harness — see API Surface |
| `sha2` | `e2e_media.rs`, `e2e_media_extended.rs`, `e2e_media_video.rs` | computing the sha256 digest that both the Blossom auth tag and the upload body must agree on |
| `base64` | same three media files | building the `Authorization: Nostr <base64url>` header |
| `hex` | `e2e_media.rs`, `e2e_media_extended.rs`, `e2e_media_video.rs`, indirectly via sha256 hex-encoding | digest formatting |
| `uuid` | `e2e_media_extended.rs`, `e2e_media_video.rs`, `e2e_managed_agent.rs`, `e2e_mesh_llm.rs`, `e2e_persona.rs` | unique channel names, `d`-tag uniqueness, subscription-id uniqueness |
| `serde_json` | all six | ad hoc `Value` construction/access — see Data Model |

Declared dev-dependencies **not used by any of these six files**: `s3`/`rust-s3` (`Cargo.toml`, the MinIO-direct-access crate — used by sibling `e2e_git.rs`, not this group), `sqlx` (used by sibling test files that need direct DB assertions, e.g. `conformance_multitenant.rs`, not this group), `chrono`, `rand`, `buzz-sdk`. Confirmed: `grep -rn 'use s3\|use sqlx\|use chrono\|use rand\|use buzz_sdk'` across all six files in this group returns zero matches. This group's tests are pure HTTP/WebSocket black-box clients — they never reach into the database, S3, or a typed event-builder SDK to construct or verify state.

#### `e2e_mesh_llm.rs`'s unique integration shape

This file is the only one in the group (and arguably in the whole crate) whose primary subject is a **second network service that is not `buzz-relay`**: an OpenAI-compatible HTTP endpoint exposed by a desktop mesh-client node (`MESH_OPENAI_BASE`, default assumption `http://127.0.0.1:9337/v1` per the module doc `:20-21`). The relay is only used for the `trust_*` half of the file (kind:30003 discovery-note visibility); the `live_*` half talks exclusively to this second service and never touches the relay at all — `live_agent_completes_chat_over_mesh` (`:219-268`) contains no `BuzzTestClient` or relay HTTP call of any kind, only two `reqwest` calls to `{base}/models` and `{base}/chat/completions`.

#### What each file does NOT integrate with (checked)

- No file in this group touches Postgres or Redis directly — `grep -rn 'DATABASE_URL\|REDIS_URL\|sqlx::' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs crates/buzz-test-client/tests/e2e_managed_agent.rs crates/buzz-test-client/tests/e2e_mesh_llm.rs crates/buzz-test-client/tests/e2e_persona.rs` returns zero matches (this group's tests only assert on relay responses, never on database rows).
- No file spawns `buzz-agent`, `buzz-acp`, or any other Buzz binary as a subprocess — `grep -rn 'Command::new\|std::process' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs crates/buzz-test-client/tests/e2e_managed_agent.rs crates/buzz-test-client/tests/e2e_mesh_llm.rs crates/buzz-test-client/tests/e2e_persona.rs` returns zero matches. "Managed agent" and "mesh" in this group's file names both refer to relay-observable *effects* of an agent process (an event, an HTTP endpoint some other process exposes), never a process this crate itself launches.
