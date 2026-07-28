## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Business Rules

#### Startup ordering rules

1. Install the `ring` rustls provider, ignoring failure (`lib.rs:39`,
   `let _ = rustls::crypto::ring::default_provider().install_default()`). The
   swallow is deliberate and documented (`lib.rs:30-38`): when `buzz-dev-mcp`
   delegates to `run_from_args`, it has already installed a provider
   (`crates/buzz-dev-mcp/src/lib.rs:165`), and the second install returns `Err`.
2. Parse argv. Usage errors print the JSON error envelope and return 1;
   help output prints verbatim and returns 0 (`lib.rs:41-53`).
3. Normalize the relay URL *before* any auth check (`lib.rs:1734`).
4. `pack` short-circuits: no key, no client, no relay (`lib.rs:1736-1743`).
   This makes `pack validate|inspect` the only commands runnable without
   `BUZZ_PRIVATE_KEY`.
5. Require the private key (`lib.rs:1746-1748`), parse it (`lib.rs:1749-1750`),
   then parse *and verify* the NIP-OA auth tag (`lib.rs:1752-1767`), then build
   the client (`lib.rs:1768`).

#### Auth rules

| Rule | Behavior | Site |
|---|---|---|
| Flag beats env | clap `env =` fallback, explicit arg wins | `lib.rs:81-89`; verified: `BUZZ_RELAY_URL=http://127.0.0.1:9 buzz --relay notaurl channels list` fails on `notaurl` |
| Key is mandatory | missing → `CliError::Auth`, exit 3 | `lib.rs:1746-1748`; verified exit 3 |
| Key format | `Keys::parse` accepts hex or nsec; failure → `CliError::Key`, exit 3 | `lib.rs:1749-1750`; verified with `BUZZ_PRIVATE_KEY=zzz` |
| Empty auth tag == unset | `Some(json) if !json.is_empty()` | `lib.rs:1755`; verified `BUZZ_AUTH_TAG=''` proceeds to the relay call |
| Auth tag must parse **and** verify against the signer's pubkey | `parse_auth_tag` then `verify_auth_tag(json, keys.public_key())`; either failure → `CliError::Auth`, exit 3 | `lib.rs:1756-1766`; verified with a bogus clause: `auth error: BUZZ_AUTH_TAG is malformed: … unsupported clause: "{}"` |
| The keypair *is* the identity | in-code comment "no tokens, no other auth" | `lib.rs:1745-1746` |

Per-request auth: every HTTP call signs a fresh NIP-98 kind-27235 event with
`u`, `method`, a UUIDv4 `nonce` and (for bodies) a `payload` SHA-256 tag
(`client.rs:84-110`); the nonce exists specifically so retries survive the
relay's replay guard (`client.rs:886-887`, `client.rs:1041-1042`).

#### Auth-tag injection rules

`sign_event` (`client.rs:588-614`) injects `self.auth_tag` and then **counts**
`auth` tags on the signed event, erroring if the count differs from the expected
0-or-1 — i.e. "callers must not add auth tags manually" is enforced at runtime,
not just by comment (`client.rs:606-613`).

`sign_event_unchecked` (`client.rs:743-747`) deliberately bypasses both
injection and the count check, and is reserved by *convention only* (a doc
comment, `client.rs:729-742`) for NIP-IA kinds 9035/9036. Nothing restricts it
to those kinds. Two tests pin the intent:
`sign_event_unchecked_does_not_inject_ambient_auth_tag` (`client.rs:2373-2400`)
and `sign_event_unchecked_preserves_callers_content_auth_tag`
(`client.rs:2401-2437`).

#### Output-format selection

`--format` is threaded to only 5 of the 21 dispatchers — `messages`,
`channels`, `users`, `feed`, `moderation` (`lib.rs:1772,1773,1778,1780,1790`).
The other 16 dispatch calls omit it, so `buzz --format compact social notes …`
parses successfully and is silently ignored. No warning is emitted; there is no
code path that rejects `compact` for an unsupported group
(`grep -n 'cli.format' lib.rs` → exactly those 5 lines).

#### Retry, timeout and idempotency rules

Two policies, chosen by event kind. `is_moderation_kind` = `9040..=9044`
(`client.rs:211-213`, test `client.rs:1511-1517` plus a negative test at
`1518-1527`).

| Failure | Stored events (all other kinds) | Moderation kinds 9040-9044 |
|---|---|---|
| TCP connect error | retry, and on exhaustion stay `Network` (retryable) | same (`client.rs:900-910`) |
| Timeout / request / body / decode | retry (`client.rs:653-661`), exhaustion → `DeliveryUnknown` (`client.rs:1060-1069`) | **no retry**, immediate `DeliveryUnknown` (`client.rs:911-923`) |
| 429 with body starting `rate-limited:` | retry honoring `retry in Ns` capped at 30 s (`client.rs:662-668`) | retry; exhaustion → `Relay{429}` (still retryable) (`client.rs:930-957`) |
| 429 otherwise (proxy) | retried as a plain 429 by `with_retry_body` | `DeliveryUnknown` immediately (`client.rs:958-963`) |
| 502-504 | retry, exhaustion → `DeliveryUnknown` (`client.rs:669-671`, `client.rs:1060-1069`) | `DeliveryUnknown` immediately (`client.rs:964-972`) |
| Other 4xx | no retry | no retry (`client.rs:983-1005`) |

Attempt budget is 3 (`client.rs:122`); inter-attempt delay is full jitter in
`[0, RETRY_BASE_SECS[attempt])` = `[0,0.5)` then `[0,1.5)`
(`client.rs:126-135`). The ambiguity classifier
`is_stored_event_exhaustion_ambiguous` (`client.rs:229-248`) is the rule that
decides retryability on exhaustion: connect → not ambiguous, canonical 429 →
not ambiguous, everything else → ambiguous.

Timeouts: per-request 30 s and connect 15 s, overridable by
`BUZZ_TIMEOUT_SECS` / `BUZZ_CONNECT_TIMEOUT_SECS`, with **zero treated as
invalid** so timeouts can never be disabled (`client.rs:140-150`,
`client.rs:547-551`; test `env_duration_secs_parsing`, `client.rs:1555-1580`).
Uploads override per-request: 120 s images, 600 s video
(`client.rs:1141-1145`); downloads use a dedicated 120 s client
(`client.rs:1231-1237`); WS publish is capped at 75 s (`client.rs:1084`).

Retry rules are well covered by behavioral integration tests over a local axum
server (`client.rs:1582-2295`): `moderation_kind_non_ingest_429_returns_delivery_unknown`
(`:1658`), `moderation_kind_ingest_429_is_retried_until_success` (`:1688`),
`exhausted_ingest_429_returns_relay_429_retryable` (`:1731`),
`moderation_kind_502_returns_delivery_unknown` (`:1764`),
`exhausted_connect_failures_return_network_retryable` (`:1787`),
`stored_event_502_is_retried_under_standard_policy` (`:1814`),
`query_403_is_not_retried` (`:1948`),
`with_retry_body_retries_on_body_transfer_failure` (`:1978`),
`stored_event_body_loss_is_retried_with_same_event_bytes` (`:2038`),
`upload_body_loss_is_retried_with_same_file_bytes` (`:2116`),
`stored_event_all_body_losses_return_delivery_unknown` (`:2207`),
`stored_event_all_502s_return_delivery_unknown` (`:2277`).

#### Exit-code mapping rules

See the API Surface aspect for the full table. The mapping rule of note:
`Relay{status}` splits on 401/403 → 3, else → 2 (`error.rs:96-100`), and the
same split drives the `error` category string (`error.rs:111-117`).
`is_retryable_error` (`error.rs:74-88`) publishes retryability separately from
the exit code, and explicitly refuses to mark `DeliveryUnknown` retryable
(`error.rs:85`).

#### Query, kinds and pagination rules

- `kinds` is **not** defaulted or required anywhere in this layer.
  `grep -c '"kinds"' client.rs` → 1, and that single hit is a test fixture
  (`client.rs:2306`). The relay p-gate documented in `AGENTS.md` gotcha 2 is
  satisfied only by the sibling command modules (e.g.
  `commands/messages.rs:276,320,361`).
- Page size is fixed at 500 (`client.rs:498`); each request asks for
  `min(remaining, 500)` (`client.rs:687-691`).
- Termination rule: a short page ends the loop; a full page advances the
  `(until, before_id)` cursor (`client.rs:692-700`).
- `query_all` (`client.rs:724-727`) has **no page or memory cap** — it loops
  until the relay returns a short page, accumulating every event in a `Vec`.
- `channels search --limit` defaults to 1000 metadata events (`lib.rs:527-528`);
  `moderation reports`/`audit --limit` default to 50 (`lib.rs:1656-1657`,
  `lib.rs:1729-1730`); most other `--limit` flags are `Option<u32>` with the
  default deferred to the relay or the sibling handler.

#### Channel scoping

No `h`-tag logic exists in this group: `grep -c '"h"' client.rs` → 0. NIP-29
scoping is entirely the command modules' concern. The one place this group
touches `h` is a *negative* assertion — agent draft frames must carry no `h`
tag (`agent_management.rs:225-227`).

#### NIP-33 last-write-wins

The `Conflict` variant and its exit code 5 exist here (`error.rs:26-29`,
`error.rs:103`) with a doc comment naming `buzz mem` set/rm as the producer, but
nothing in the six in-scope files constructs a `Conflict`:
`grep -n 'CliError::Conflict' lib.rs client.rs validate.rs agent_management.rs`
returns zero matches outside `error.rs` itself. The LWW rule is implemented in
the sibling-owned `commands/mem.rs`; this layer only carries the code.

#### Media rules

`media_url_from_input` (`client.rs:270-323`) enforces, in order:
absolute URLs must be `http(s)` (`:272`), path must start `/media/`
(`:275-284`), the segment must be `sha256`, `sha256.ext` or `sha256.thumb.jpg`
with lowercase hex and an `[a-z0-9]{1,8}` extension (`client.rs:250-268`), and
the origin (scheme + host + effective port) must equal the configured relay —
otherwise "refusing to sign media GET for a non-relay origin" (`:305-315`).
Bare inputs get `/media/` prefixed after the same segment check (`:317-322`).
Three tests cover this (`client.rs:391-451`).

Upload rules: regular file required (`client.rs:1102-1106`), MIME sniffed from
magic bytes rather than the extension (`client.rs:1109-1112`), MIME must be in
the 5-entry allowlist (`client.rs:1114-1116`), size ≤ 50 MiB image / 500 MiB
video (`client.rs:1118-1130`). Endpoint fallback is narrow by rule: only 404 and
405 switch to `/media/upload`, and the switch itself is not retried
(`client.rs:200-207`, `client.rs:1180-1193`; test
`legacy_upload_retry_statuses_are_narrow`, `client.rs:483-496`).

Blossom auth expiry rules: get = now+600 s (`client.rs:329-330`); upload =
now+3600 s for `video/*`, else now+600 s (`client.rs:359-363`).

#### Validation rules (`validate.rs`)

| Rule | Detail | Site |
|---|---|---|
| UUID | strict `Uuid::parse_str` | `:23-27` |
| 64-hex | length 64 **and** `is_ascii_hexdigit` — uppercase accepted despite the doc saying "lowercase" | `:28-36` |
| repo id | 1-64 chars, no leading `.`, no `..`, `[A-Za-z0-9._-]` only | `:39-61` |
| content size | ≤ 65,536 **bytes** (`str::len`, not chars) | `:64-72` |
| diff truncation | cut at the last `\n@@` inside the budget, else last newline, always UTF-8-boundary safe, notice appended | `:103-121` |
| language inference | extension → language, 26 mappings, `None` when unknown | `:124-152` |
| SDK error mapping | `SdkError::InvalidInput` → `Usage` (exit 1), all else → `Other` (exit 4) | `:155-160` |
| `-` means stdin | `read_or_stdin` treats any other value as literal content; `read_file_or_stdin` always treats it as a path | `:162-193` |

The `read_or_stdin` vs `read_file_or_stdin` split is itself a bug-fix rule with
a regression test — `read_file_or_stdin_does_not_treat_path_as_literal_content`
(`validate.rs:498-505`) documents the prior bug where `--patch-file` echoed the
path as content.

#### Agent-draft rules (`agent_management.rs`)

- Every string field is trimmed, must be non-empty, and is length-capped in
  characters (not bytes): 120 for names, 20,000 for prompts, 300 for other
  optional fields (`agent_management.rs:70-85`).
- `channel` must be a valid UUID *and* ≤128 chars (`:129-131`, `:145-147`).
- `respond_to` must be exactly `owner-only` or `anyone` (`:148-155`) — a second,
  string-level check on top of the `RespondToArg` value enum (`lib.rs:242-257`),
  because the public struct takes a free-form `Option<String>`.
- An update must change at least one field, else `Usage`
  (`agent_management.rs:169-179`; test `update_requires_a_change`, `:243-262`).
- Drafts are encrypted to the owner pubkey and published as observer telemetry
  frames (`:107-121`), so the rule "owner reviews before anything is created" is
  structural: the CLI never writes an agent record itself.

#### Rules enforced only by comment or convention

- `sign_event_unchecked` is "NIP-IA only" by doc comment (`client.rs:729-742`).
- "Callers MUST NOT add `auth` tags before calling `sign_event`" is documented
  (`client.rs:585-587`) *and* enforced (`client.rs:606-613`) — the good case.
- `AGENTS.md`'s no-`unwrap`/`expect` rule is not enforced by lint; three
  production panic sites remain (`client.rs:506`, `client.rs:1379`, plus
  `unreachable!` at `client.rs:679`, `client.rs:1018`, `lib.rs:1791`).
- The `retry in Ns` cap of 30 s is described as "defensive … real relay hints
  observed up to ~24 s" (`client.rs:128-130`) — a convention derived from
  observation, not a negotiated protocol value.
