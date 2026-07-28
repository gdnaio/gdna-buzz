## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Business Rules

#### `e2e_media.rs` — base upload/download rules

| Rule asserted | Site |
|---|---|
| A valid Blossom-auth'd upload of a JPEG returns 200 with a `BlobDescriptor` whose `sha256` matches the request digest, `url` contains that sha256, `size > 0`, and `type` is present | `e2e_media.rs:112-140` |
| `GET /media/{sha256}.jpg` returns the exact original bytes (byte-for-byte round-trip) | `e2e_media.rs:150-160` |
| `HEAD /media/{sha256}.jpg` returns 200 with a `content-type` header present | `e2e_media.rs:163-171` |
| `GET /media/{sha256}.thumb.jpg` (thumbnail route) returns 200 even for a 1×1 source image | `e2e_media.rs:174-182` |
| Uploading the same bytes under two different signing keys yields an identical `sha256` and `url` (content-addressed dedup, key-independent) | `e2e_media.rs:186-224` |
| Missing `Authorization` header → 401 | `e2e_media.rs:230-247` |
| Missing `X-SHA-256` header → 401 (asserted as BUD-11 mandatory in the doc comment) | `e2e_media.rs:249-271` |
| A Blossom-auth `x`-tag whose sha256 does not match the actual body → 401 (the test's own assertion message says "must be 400" but the coded expectation is `401`, see Debt) | `e2e_media.rs:273-295` |
| `GET` of a sha256 that was never uploaded → 404 | `e2e_media.rs:297-309` |
| A **real** image (path from `TEST_IMAGE_PATH`, test no-ops if unset) additionally populates `dim` and `blurhash` as non-empty strings, unlike the synthetic 1×1 fixture | `e2e_media.rs:311-372` |

No test in this file asserts a size-limit rejection — there is no oversized-upload case anywhere in `e2e_media.rs` (confirmed by reading the full file; the only size assertion is `size > 0` or `size == bytes.len()`, never an upper bound).

#### `e2e_media_extended.rs` — auth edge cases, format matrix, content-policy routing

This file goes well beyond "uploads a file" — it pins the Blossom auth-event **grammar** and a **two-tier content-type policy** (canonical media vs. everything else):

| Rule | Outcome asserted | Site |
|---|---|---|
| Auth event kind must be exactly 24242 | wrong kind (27235) → 401 | `e2e_media_extended.rs:220-243` |
| Auth event must carry a `t` tag | missing → 401 | `:245-265` |
| Auth event must carry an `expiration` tag | missing → 401 | `:267-286` |
| `expiration` must be in the future | `now-60` → 401 | `:288-309` |
| Auth event content must be non-empty | empty string → 401 | `:311-332` |
| An optional `server` tag, if present, must match the relay's own host | mismatch (`evil.example.com`) → 401; match (`localhost:3000`) → 200 | `:334-380` |
| PNG/GIF/WebP each upload, are typed correctly, and round-trip byte-for-byte | 200 + `type`/`url` extension check + GET-back equality | `:158-221` |
| SVG with an XML declaration is sniffed by `infer` as `text/xml` (not `image/svg+xml`), which is **not blocklisted**, so it is accepted through the generic "attachment" path | 200, `type == "text/xml"` | `:382-399` |
| PDF is detected and accepted the same way | 200, `type == "application/pdf"` | `:401-418` |
| The **legacy** `/media/upload` route enforces a **narrower** policy than `/upload`: it rejects a non-media file (PDF) with 415 | `reqwest::StatusCode::UNSUPPORTED_MEDIA_TYPE` | `:420-433` |
| The same legacy route still accepts canonical media (JPEG) | 200 | `:434-441` |
| The **standard** `/upload` route rejects a *recognized* audio file (MP3 magic bytes) — audio is explicitly disallowed even though PDF/SVG/octet-stream are allowed | 415 | `:443-455` |
| Zero-byte body has no sniffable magic and is accepted as `application/octet-stream` with `size == 0` | 200 | `:457-475` |
| Non-zero random bytes with no magic signature are likewise accepted as `application/octet-stream` | 200 | `:476-494` |
| Two concurrent uploads (different keys, same bytes, via `tokio::join!`) both succeed and converge on the same `sha256`/`url` | 200 + 200, equal descriptors | `:495-521` |
| A kind:9 message with a **valid** `imeta` tag (same-origin URL, correct `m`/`x`/`size`) is accepted over the WebSocket | `ok.accepted == true` | `:522-591` |
| A kind:9 message whose `imeta` `url` points to an **external** host (`evil.com`) is rejected, with a message containing `"invalid"` | `ok.accepted == false` | `:593-658` |
| A kind:9 message whose `imeta` tag is missing `m`/`x`/`size` (only `url` present) is rejected | `ok.accepted == false` | `:659-712` |

So the content-policy rule this file establishes is: **the canonical `/upload` route accepts any file type except recognized audio** (generic attachments — PDF, SVG-as-XML, octet-stream — flow through a "file" path), while **the legacy `/media/upload` alias only accepts canonical image/video media** and 415s everything else including PDF. That asymmetry (`:420-455`) is the load-bearing rule of this file, not merely "more formats work."

#### `e2e_media_video.rs` — video-specific rules

| Rule | Site |
|---|---|
| A structurally valid MP4 (hand-built ftyp/moov/mdat boxes, H.264 `avc1` track) uploads successfully and its `BlobDescriptor` includes a `duration` field (presence only, value unchecked) | `:230-278` |
| The relay **ignores the `Content-Type` header** and sniffs magic bytes: an MP4 body sent with `Content-Type: image/jpeg` is still accepted and typed as `video/mp4`, including through the legacy `/media/upload` route | `:280-321` (doc comment `:224-229` explicitly states this "including on the legacy compatibility route") |
| `Range: bytes=0-99` on a video blob returns 206 with `content-range` and `accept-ranges: bytes` headers, and the body is exactly the requested 100 bytes matching the source | `:322-364` |
| A range beyond the file's length returns 416 (`RANGE_NOT_SATISFIABLE`) | `:365-408` |
| Upload without auth → 401 | `:409-428` |
| A message referencing both a video (`imeta` `url`/`m`/`x`/`size`) and a separate image poster (`imeta` `image` field) is accepted | `:469-565` |
| A message whose `imeta` `image` field points at the **video's own URL** (`.mp4`) instead of an actual image is rejected — the poster must be an image blob, not the video reused | `:567-645` |

No duration **value** is ever asserted (only `.is_some()` at `:271-274`), and no video-specific size cap is tested anywhere in this file — `grep -n 'too large\|max.*size\|MAX_.*BYTES' crates/buzz-test-client/tests/e2e_media_video.rs` returns zero matches. Format validation is exercised only positively (one well-formed MP4 fixture); there is no test that uploads a corrupt/truncated MP4 and asserts rejection.

#### `e2e_managed_agent.rs` — managed-agent (kind:30177) lifecycle rules

This file tests **relay-side event handling**, not an actual agent process lifecycle (spawn/ready/turn/teardown are desktop-side concepts this crate cannot reach — see module doc `e2e_managed_agent.rs:8-16`). The "lifecycle" here is entirely about the NIP-33/NIP-09 event-store semantics of the coordinate:

| Rule | Site |
|---|---|
| A well-formed managed-agent event (opt-in projected content, `d`-tag = agent's 64-hex pubkey) is accepted and is queryable by `{kind:30177, author, #d}` | `:114-157` |
| The relay still accepts the **legacy "fat"** content shape (adds `persona_id`, `persona_source_version`) — a backward-compat guarantee during a schema transition | `:159-184` |
| The relay round-trips published content **verbatim** — it neither strips nor injects fields; verified by asserting the published string does not contain any of 8 secret-shaped substrings or 6 runtime-shaped substrings, and does contain the identity fields that were sent | `:186-263` |
| NIP-33: republishing with the same `d`-tag and a strictly newer `created_at` makes only the newer event queryable | `:265-311` |
| A NIP-09 `a`-tag-only delete (no `e`-tag) at the agent's coordinate removes it from subsequent queries — before-delete the agent is visible, after-delete a fresh query returns zero events | `:313-368` |

The file explicitly disclaims testing the secret-exclusion property itself (`:26-34`, `:186-193`) — that guard is a `buzz-agent`-desktop unit test (`content_excludes_secrets_and_runtime_fields`, confirmed present at `desktop/src-tauri/src/managed_agents/agent_events.rs:249`), not this e2e file. What this file actually proves is narrower: **given** a secret-free projected body, the relay does not add secrets and does not drop the identity fields.

#### `e2e_mesh_llm.rs` — mesh routing rules: precisely what is and isn't driven

The file's own acceptance matrix (module doc `:29-38`) is the authoritative scope statement. Breaking down what each numbered assertion actually requires:

| # | What the test drives | Requires a real mesh router/desktop? | Site |
|---|---|---|---|
| 1 | `trust_member_reads_mesh_status` — connects with a real nsec from `MEMBER_NSEC`, subscribes to `{kind:30003, #k:"buzz-mesh-status"}`, asserts owner-binding fields present and no secret-shaped substrings | **Yes** — needs `MEMBER_NSEC` env var pointing at an identity that is a relay member **and** has a live desktop that already published a status event; the test **skips** (prints and returns) if the env var is absent | `:92-171` |
| 2 | `trust_nonmember_read_denied` — connects with `STRANGER_NSEC`, asserts no kind:30003 event leaks | **Yes**, same infra requirement; also skips silently if the env var is absent or if auth itself is refused | `:174-217` |
| 3 | (not in this file — routed to `mesh_admission_smoke`, a separate `buzz-relay` example binary per the doc's own table) | n/a | doc `:33` |
| 4 | `live_agent_completes_chat_over_mesh` — only runs its body if `MESH_OPENAI_BASE` is set; then does a real `GET /models` + `POST /chat/completions` against that base and asserts non-empty content | **Yes, and the heaviest requirement**: two live desktop mesh nodes (one serving a model, one running a mesh client) plus a small downloaded model — explicitly "never in default CI" per the doc comment | `:219-268` |
| 5 | (not in this file — routed to a runbook + a `buzz-agent` unit test) | n/a | doc `:37` |
| 6 | `live_split_model_completes` — **prints a skip message and returns immediately**; contains no assertions and drives nothing | No — this is a documented no-op placeholder | `:281-291` |

So of `e2e_mesh_llm.rs`'s four `#[tokio::test]` functions, **none exercises a fake/mock mesh router** — every test either (a) requires real external identities/infra and self-skips when absent, or (b) is an unconditional no-op skip. There is no in-process fake mesh router or stub HTTP server anywhere in this file (contrast `crates/buzz-agent/src/llm.rs`'s own test module, which the earlier `buzz-agent` analysis found uses a hand-rolled `spawn_sequence_stub` HTTP server — that stub lives in the *unit* test suite, not here). This file is what its own doc calls "the opt-in full-stack acceptance layer" (`:9-10`) — it is real-infra-or-skip, never mocked. Consequently the mesh "Auto" collective-routing state machine (`resolve_openai_model`/`observe_mesh_virtual_model` in `buzz-agent/src/llm.rs`, per the prior `agent-llm` analysis) is **not exercised by this file at all** — this file tests the relay's kind:30003 read-visibility gate and, conditionally, a live chat-completions endpoint's mere reachability, not `buzz-agent`'s model-substitution logic, its TTL/cooldown state, or its catalog-probe classifier. That logic remains covered only by `buzz-agent`'s own unit tests (per the prior analysis, which found even those never exercise a live relay/router) — meaning the mesh-routing state machine has **no test at any level that exercises it against a live mesh router**, unit or e2e.

#### `e2e_persona.rs` — persona (kind:30175) rules: the largest and most detailed file in scope

Organized by the sub-clusters the file itself uses (comment-delimited sections):

**Core publish/query/replace (`:156-360`)**

| Rule | Site |
|---|---|
| A well-formed persona event is accepted and retrievable by its own event id | `:156-208` |
| A prompt-less persona (`{"display_name": "..."}` only, no `system_prompt`) ingests through the real relay path and round-trips byte-for-byte — the relay does not require or inject a system prompt | `:209-245` |
| A persona carrying the "reserved behavioral fields" (`system_prompt`, `respond_to`, `respond_to_allowlist`, `parallelism`) ingests and round-trips byte-for-byte as opaque content — the relay does not interpret or validate these fields' semantics, only stores them | `:246-287` |
| NIP-33: same `d`-tag, strictly newer `created_at` wins; only the newer content is returned | `:288-333` |
| NIP-33: publishing an **older** event after a newer one does not replace the head (relay may accept or reject the older event, but the query result is unaffected either way) | `:334-380` |

**`d`-tag slug grammar (`:381-561`)** — matches the NIP-AP spec grammar `^[a-z0-9][a-z0-9_-]{0,63}$` documented at `docs/nips/NIP-AP.md:24-29`:

| Rejected input | Site |
|---|---|
| empty `d`-tag | `:381-406` |
| missing `d`-tag entirely | `:408-429` |
| 65-character slug (1 over the 64 max) | `:430-458` |
| uppercase characters (`My-Persona`) | `:459-481` |
| special characters (`my.persona!`) | `:482-503` |
| leading underscore (`_invalid`) | `:504-526` |

| Accepted input | Site |
|---|---|
| single character, hyphens, underscores, leading digit, and exactly 64 `a` characters (boundary) | `:527-561` |

| Multiplicity rule | Site |
|---|---|
| One author can hold multiple live personas simultaneously (different `d`-tags); an `{authors:[self]}` query returns ≥2 | `:562-620` |

**Shared-read gate (`:622-1234`)** — the largest cluster, testing the author-only-unless-`["shared","true"]` visibility rule end-to-end across WS REQ, WS live fan-out, WS COUNT, and the HTTP `/query`+`/count` bridge:

| Rule | Surface | Site |
|---|---|---|
| A foreign reader querying `{kinds:[30175], authors:[victim]}` sees only the shared persona, never the unshared one; the author sees both via self-query | WS REQ | `:658-734` |
| `{ids:[unshared-id]}` returns nothing to a foreign reader (kindless lookup does not bypass the gate) | WS REQ | `:735-780` |
| Foreign `COUNT` over `{kinds:[30175], authors:[victim]}` returns 1 (the shared count), not the true total | WS COUNT | `:781-846` |
| Live fan-out: an unshared publish is **not** delivered to a foreign live subscription; republishing the same `d`-tag **with** `["shared","true"]` (and a strictly later timestamp) **is** delivered; republishing again **without** the tag makes it invisible again, both to a fresh REQ and to the live subscription | WS REQ + live fan-out | `:847-1035` (this single test, `test_persona_live_fanout_shared_gate`, drives three transitions: unshared→shared→unshared) |
| Ingest validates the `shared` tag's value grammar: `["shared","true"]` and tag-absent are accepted; `["shared","false"]`, `["shared","x"]`, `["shared"]` (no value), and duplicate `["shared","true"]` tags are all rejected | ingest | `:1036-1150` |
| A mixed-kind filter `{kinds:[30175,9], authors:[victim]}` excludes the foreign unshared persona **but still passes through** the author's kind:9 event — proving the gate is per-event, not a wholesale filter drop | WS REQ | `:1151-1231` |
| The same cross-author gate holds over the NIP-98 HTTP `/query` bridge, for both an `authors`-scoped query and kindless `{ids:[...]}` lookups (unshared → empty, shared → present) | HTTP `/query` | `:1232-1329` |
| The same gate holds over HTTP `/count`, including a wildcard (`authors`-unscoped) count and a second author-scoped count taken *after* the wildcard query (guards against session/caching leakage) | HTTP `/count` | `:1330-1409` |

**Visibility-before-LIMIT regression (`:1410-1553`)** — a specific ordering bug class: if N unshared (newer) events are published before 1 shared (older) event, a `limit < N+1` query must still surface the shared event, because the SQL-level visibility filter must run **before** `ORDER BY … LIMIT`, not after:

| Rule | Surface | Site |
|---|---|---|
| With `limit=2` and 3 newer-unshared + 1 older-shared, the shared event still appears and no unshared event leaks | WS REQ | `:1410-1486` |
| Same scenario over HTTP `/query` | HTTP | `:1487-1554` |

Both tests carry a regression citation to a specific commit (`312014d5e`) where the bug reproduced (`:1403-1408`), i.e. this is a pinned regression test, not a speculative property.

**Wire-level tag-shape rejection (`:1554-1591`)**

| Rule | Site |
|---|---|
| A three-element `["shared","true","extra"]` tag is rejected at ingest over the real WebSocket path (not just in a unit-level validator) | `:1554-1591` |

Taken as a whole, `e2e_persona.rs` is overwhelmingly about **read-visibility enforcement** (the `shared` tag gate) rather than persona *content* semantics — of its 24 tests, roughly half (12) exist solely to pin the shared/unshared boundary across every read surface the relay exposes (WS REQ, WS COUNT, WS live fan-out, HTTP query, HTTP count), and none inspects `mcp_servers`, `hooks`, `skills`, or any other `buzz-persona` pack-level field (see Data Model).
