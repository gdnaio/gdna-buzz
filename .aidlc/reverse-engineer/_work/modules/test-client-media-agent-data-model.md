## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Data Model

This group's six files construct requests and assert on responses from the client side — none define production types. All shapes below are inferred from literal JSON construction and `serde_json::Value` field access in the tests themselves, not from importing a shared schema crate (the only intra-workspace type import across all six files is `buzz_test_client::{BuzzTestClient, RelayMessage, TestClientError}`, e.g. `e2e_media_extended.rs:322`, `e2e_persona.rs:19`).

#### Blossom `BlobDescriptor` (media upload response)

Exercised as an untyped `serde_json::Value` in all three media files — there is no `BlobDescriptor` struct in this crate; the shape is reverse-engineered from field accesses:

| Field | Type observed | Asserted in | Site |
|---|---|---|---|
| `sha256` | string, hex | equals request digest | `e2e_media.rs:125-128`, `e2e_media_extended.rs:180` |
| `url` | string | contains the sha256; ends with the correct extension (`.png`, `.gif`, `.webp`) | `e2e_media.rs:130-132`; ext checks `e2e_media_extended.rs:181`, `:203`, `:221` |
| `size` | u64 | `>0` for the tiny fixture, `== bytes.len()` for a real image, `==0` for the empty-body case | `e2e_media.rs:133-136`; `e2e_media.rs:290`; `e2e_media_extended.rs:474` |
| `type` | string (MIME) | present (`e2e_media.rs:137-139`); exact value asserted per-format (`image/png`, `image/gif`, `image/webp`, `text/xml`, `application/pdf`, `application/octet-stream`, `video/mp4`) | `e2e_media_extended.rs:180,203,221,392,412,472,491`; `e2e_media_video.rs:265` |
| `dim` | string (best-effort) | not asserted for the 1x1 synthetic JPEG (`e2e_media.rs:143-146`); asserted **present** for a real image loaded from `TEST_IMAGE_PATH` | `e2e_media.rs:288` |
| `blurhash` | string (best-effort) | same pattern — skipped for synthetic, asserted present for a real image | `e2e_media.rs:143-146`, `e2e_media.rs:292` |
| `duration` | present (video only) | asserted `.is_some()`, value never checked | `e2e_media_video.rs:271-274` |

No test ever deserializes this into a typed Rust struct — every access is `descriptor["field"].as_str()/.as_u64()`. The video descriptor is not shown to carry `dim` for width/height in any assertion in this group (only `duration` is checked at `e2e_media_video.rs:271-274`); whether the relay's video path also populates `dim` is not verified here.

#### Blossom auth event (kind 24242, BUD-02)

Built inline by every media file's `sign_blossom_auth` helper (near-identical copies — see Conventions/Debt) as an unsigned-then-signed `nostr::Event`:

| Tag | Required by these tests | Site |
|---|---|---|
| `["t", "upload"]` | yes | `e2e_media.rs:46`, `e2e_media_extended.rs:59`, `e2e_media_video.rs:47` |
| `["x", "<sha256>"]` | yes | same sites |
| `["expiration", "<unix-secs>"]` | yes, `now + 300` | same sites |
| `["server", "<host>"]` | optional; tested both matching and mismatching | `e2e_media_extended.rs:325-351` (mismatch → 401), `:354-371` (match → 200) |

`e2e_media_extended.rs` additionally builds this event with deliberately malformed shapes via `sign_custom_auth` (`e2e_media_extended.rs:143-148`): wrong kind (27235 instead of 24242), missing `t`, missing `expiration`, expired `expiration` (`now - 60`), empty content. These are request fixtures, not response shapes, but they pin the auth event's required-field contract from the client side.

#### imeta tag (NIP-92-style, media attached to kind:9 messages)

Constructed as a single multi-value `Tag::parse(["imeta", "url ...", "m ...", "x ...", "size ...", "image ..."])`:

| Sub-field | Format | Site |
|---|---|---|
| `url <http-url>` | full media URL, must be same-origin (`http://localhost:3000/media/{sha}.{ext}`) | `e2e_media_extended.rs:534-536`; rejected with an external host at `:606` |
| `m <mime>` | e.g. `image/jpeg`, `video/mp4` | `e2e_media_extended.rs:537`; `e2e_media_video.rs:530` |
| `x <sha256>` | hex digest | `e2e_media_extended.rs:538` |
| `size <bytes>` | decimal string, hardcoded literal `"347"` in two media tests regardless of actual fixture size (`e2e_media_extended.rs:539`, `:611`) — see Debt | |
| `image <poster-url>` | video-only; must point at an actual image blob, not the video's own URL | `e2e_media_video.rs:533`; the rejection case deliberately reuses the video URL as the poster (`e2e_media_video.rs:610-614`) |

No response shape exists for imeta — the only observable outcome is the `OkResponse{accepted, message}` from `send_event` (see API Surface).

#### Managed-agent event content (kind 30177, NIP-AP)

`e2e_managed_agent.rs` hand-builds two content literals as `serde_json::json!` objects, explicitly *not* importing the desktop's `ManagedAgentRecord`/projection function (`e2e_managed_agent.rs:29-34` states the e2e crate cannot reach the desktop crate):

**Current (slimmed) projection**, `agent_projection_content` (`e2e_managed_agent.rs:36-46`):

| Field | Example value | Site |
|---|---|---|
| `name` | `"Test Agent"` | `e2e_managed_agent.rs:38` |
| `system_prompt` | `"You are a test agent."` | `:39` |
| `model` | `"claude-opus-4"` | `:40` |
| `provider` | `"anthropic"` | `:41` |
| `parallelism` | `24` | `:42` |
| `respond_to` | `"allowlist"` | `:43` |
| `respond_to_allowlist` | `["79be667e"]` | `:44` |

**Legacy "fat" projection**, `legacy_fat_agent_projection_content` (`e2e_managed_agent.rs:52-64`): the same seven fields plus `persona_id` and `persona_source_version`, asserted still-acceptable by the relay (`e2e_managed_agent.rs:159-176`).

This matches the "opt-in allowlist projection" described by the file's own module doc comment (`e2e_managed_agent.rs:26-34`) and independently confirmed by reading `desktop/src-tauri/src/managed_agents/agent_events.rs` (`ManagedAgentEventContent` struct at line 38, `build_agent_event` at line 113) — the e2e test's field set is a strict subset that omits secrets/runtime fields the desktop struct also excludes.

The `d`-tag for a managed-agent event is asserted to be the agent's 64-hex-char pubkey (`agent_d_tag()` builds a 64-char lowercase-hex string by repeating a UUID's simple form twice, `e2e_managed_agent.rs:67-69`), and the NIP-09 delete uses an `a`-tag-only coordinate `"{AGENT_KIND}:{pubkey}:{d_tag}"` with no `e`-tag (`e2e_managed_agent.rs:79-85`).

The round-trip test (`test_managed_agent_round_trips_only_projected_fields`, `e2e_managed_agent.rs:186-263`) asserts by **substring absence**, not schema validation: it checks the published content string does not contain any of `private_key_nsec`, `private_key`, `nsec1`, `auth_tag`, `env_vars`, `backend`, `backend_agent_id`, `provider_binary_path`, `runtime_pid`, `last_started_at`, `last_stopped_at`, `last_exit_code`, `last_error`, `relay_url` (`e2e_managed_agent.rs:229-255`). This is a relay round-trip-fidelity check, explicitly not the secret-exclusion guard itself (that lives in the desktop unit test `content_excludes_secrets_and_runtime_fields`, confirmed present at `desktop/src-tauri/src/managed_agents/agent_events.rs:249`).

#### Mesh-status content (kind 30003, client-owned NIP-51 bookmark-set repurposed as discovery note)

`e2e_mesh_llm.rs` never constructs this content (it is published by a desktop process outside the test's control) — it only reads and asserts shape on it:

| Field | Asserted | Site |
|---|---|---|
| `ownerId`, `ownerVerifyingKey`, `ownerBindingSig` | each present and non-empty-after-trim | `e2e_mesh_llm.rs:135-142` |
| `serveTargets` | array; each element optionally has `endpointAddr` | `e2e_mesh_llm.rs:144-151` |
| (whole content, lowercased) | must NOT contain `nsec`, `secret`, `/users/`, `/home/`, `runtime_dir`, `local_path` | `e2e_mesh_llm.rs:154-163` |

The `d`-tag is asserted to start with the constant prefix `buzz-mesh-member-status:` (`e2e_mesh_llm.rs:45`, checked `e2e_mesh_llm.rs:127-133`), and the event must carry `["k", "buzz-mesh-status"]` (filter built at `e2e_mesh_llm.rs:80-84`). Both the constant kind number (30003) and the `d`-tag prefix are independently verified elsewhere in the repo: `crates/buzz-db/src/lib.rs:3682-3683` and `migrations/0019_mesh_status_retention.sql:1-18` hard-delete superseded rows matching this exact kind/prefix/tag combination, and `desktop/src-tauri/src/mesh_llm/coordinator.rs:20-21` defines the same kind and prefix as `KIND_BUZZ_MESH_MEMBER_STATUS`/`STATUS_D_TAG_PREFIX`. The test file's hardcoded constants are not invented — they mirror a real, independently-defined wire contract.

#### Mesh chat-completion request/response (OpenAI-compatible surface, `live_agent_completes_chat_over_mesh`)

Only exercised when `MESH_OPENAI_BASE` is set (`e2e_mesh_llm.rs:226-231`) — otherwise the test returns early and asserts nothing. When it runs:

- `GET {base}/models` → expects `{"data": [{"id": "<model-id>"}, ...]}`, reads `data[0].id` (`e2e_mesh_llm.rs:236-243`).
- `POST {base}/chat/completions` with `{"model", "messages": [{"role":"user","content":"Reply with exactly one word: PONG"}], "max_tokens": 512, "temperature": 0.0}` (`e2e_mesh_llm.rs:246-253`).
- Response expected shape: `{"choices": [{"message": {"content": "<string>"}}]}`, only asserted non-empty after trim (`e2e_mesh_llm.rs:262-266`) — the literal "PONG" is requested but never checked in the assertion.

This is the standard OpenAI Chat Completions wire shape; nothing mesh-specific is validated in the JSON structure itself (the "mesh" property being tested is that the endpoint answers at all when pointed at a mesh-client's local port, not any field in the payload).

#### Persona event content (kind 30175, NIP-AP)

`e2e_persona.rs` constructs persona `content` as **raw JSON strings**, not typed structs, and treats content as an opaque byte string for round-trip purposes — the relay is asserted to store and return it unchanged (`assert_eq!(events[0].content, content, ...)`, e.g. `e2e_persona.rs:225`, `:270`). Content field combinations actually constructed across the file:

| Test | Fields in `content` | Site |
|---|---|---|
| `test_persona_publish_and_query` | `name`, `display_name`, `description` | `e2e_persona.rs:169-173` |
| `test_promptless_persona_ingests_and_round_trips` | `display_name` only (no `system_prompt`) | `e2e_persona.rs:214` |
| `test_behavioral_fields_persona_ingests_and_round_trips` | `display_name`, `system_prompt`, `respond_to`, `respond_to_allowlist` (empty array), `parallelism` | `e2e_persona.rs:250-256` |
| shared-gate tests (`persona_event_with_shared*`) | `display_name` only | `e2e_persona.rs:414-415`, `:432-433` |

This is a materially **smaller** field set than the `PersonaConfig` frontmatter schema documented for `buzz-persona` (`name`, `display_name`, `avatar`, `description`, `version`, `author`, `skills`, `mcp_servers`, `subscribe`, `triggers`, `model`, `runtime`, `temperature`, `max_context_tokens`, `thread_replies`, `broadcast_replies`, `hooks`, `prompt` — per the prior `buzz-persona-data-model.md` analysis). None of `avatar`, `version`, `author`, `skills`, `mcp_servers`, `model`, `runtime`, `temperature`, `max_context_tokens`, `thread_replies`, `broadcast_replies`, or `hooks` is ever constructed in this file. That is expected — this crate's relay layer treats persona `content` as opaque JSON (it validates only the `d`-tag and the `shared` tag, never the content schema, confirmed by the promptless and behavioral-fields tests explicitly exercising the relay's schema-agnosticism) — but it means **this file provides zero coverage of the `buzz-persona` crate's frontmatter/pack/MCP-server/hooks data model**. That coverage, if it exists, lives elsewhere (unit tests in `buzz-persona` itself, not in this e2e group).

The `["shared","true"]` tag is the one structural (non-content) field this file adds beyond a bare `d`-tag: `persona_event_with_shared`/`persona_event_with_shared_at` (`e2e_persona.rs:405-435`) conditionally append it, and malformed variants (`["shared","false"]`, `["shared","x"]`, `["shared"]` with no value, duplicate `["shared","true"]` twice, and a three-element `["shared","true","extra"]`) are constructed as negative fixtures (`e2e_persona.rs:1075-1136`, `:1567-1576`). The two-element-only grammar these tests assert against (`persona_event_is_shared` requiring exactly two elements) is corroborated by the top-level `business-rules.md` finding `BR-15c`, which cites `crates/buzz-core/src/kind.rs:226-241`.

#### What is absent (checked, zero matches)

- No `BlobDescriptor`, `ManagedAgentRecord`, `PersonaConfig`, or `MeshCatalogObservation` Rust type is imported or referenced by name anywhere in these six files — `grep -rn 'BlobDescriptor\|ManagedAgentRecord\|PersonaConfig\|MeshCatalogObservation' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs crates/buzz-test-client/tests/e2e_managed_agent.rs crates/buzz-test-client/tests/e2e_mesh_llm.rs crates/buzz-test-client/tests/e2e_persona.rs` returns zero matches. Every shape in this document is inferred from ad hoc JSON construction/access, not from a shared schema.
