## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Debt

#### Dead code kept alive by `#[allow(dead_code)]`

| Item | Site | Callers |
|---|---|---|
| `BuzzClient::relay_url()` | `client.rs:567-570` | none (`grep -rn '\.relay_url()' commands/` → zero) |
| `BuzzClient::count()` — the whole `POST /count` integration | `client.rs:802-834` | none (`grep -rn 'client.count(' commands/` → zero) |

Both use `#[allow(dead_code)]` rather than a `TODO` explaining the intent or a
deletion. `POST /count` is a documented relay capability (`AGENTS.md:145`) with a
finished, retry-wrapped client method that no subcommand exposes — either wire it
to a `buzz count` subcommand or delete it. These are the only two `#[allow]`
attributes in the group (`grep -rn '#\[allow' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`).

There are **zero** `TODO`/`FIXME`/`HACK`/`XXX` markers in the group
(`grep -rn 'TODO\|FIXME\|HACK\|XXX' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns nothing) — so all debt below is undeclared.

#### Test-only functions living in production files

`parse_retry_in_secs` (`client.rs:172-186`) is `#[cfg(test)]`-gated and has six
tests devoted to it (`client.rs:1444-1477`). Production never calls it: the real
code path calls `parse_retry_hint_text` directly on the (already field-extracted)
body (`client.rs:663`, `client.rs:948`). So six of the group's 95 tests exercise a
JSON-extraction wrapper that does not exist at runtime, while the production
composition — `handle_response` extracts the field, *then*
`parse_retry_hint_text` runs — is covered only indirectly by the two integration
tests that assert elapsed time (`client.rs:1688`, `client.rs:1909`).

`percent_encode` (`validate.rs:75-99`) is the same pattern: `#[cfg(test)]`-gated,
five tests (`validate.rs:277-306`), zero production callers
(`grep -rn 'percent_encode' commands/` → zero). `crates/buzz-cli/README.md:216`
still advertises `validate.rs (UUID, hex, content size, percent-encode)` as if it
were live behavior.

#### Unreachable branch plus stale comment

`run_from_args` handles clap's non-stderr errors with the comment
"`--help` and `--version`: print normally (intentional human output)"
(`lib.rs:48-52`). `--version` is not declared — `#[command(...)]`
(`lib.rs:62-78`) has no `version` attribute — so that half of the comment
describes an impossible input. Verified: `buzz --version` exits **1** with
`unexpected argument '--version' found`.

#### Duplication

- The relay-error-body extraction rule exists three times: the named helper
  `extract_relay_message_field` (`client.rs:190-198`), an inline copy in
  `submit_moderation_event` under the comment "Map the body through
  `handle_response`'s error logic inline" (`client.rs:985-996`), and a third copy
  inside `handle_response` itself (`client.rs:1261-1270`) — which does not call
  the helper either.
- The 403 `BUZZ_AUTH_TAG` hint string is duplicated verbatim
  (`client.rs:991-997`, `client.rs:1271-1279`).
- `resp_was_success(u16)` (`client.rs:217-219`) re-implements
  `StatusCode::is_success` because `submit_moderation_event` consumes the
  response before it checks the status.
- `upload_file` repeats its entire request-building block for the legacy endpoint
  (`client.rs:1152-1178` vs `client.rs:1195-1226`) — ~30 lines differing only in
  the URL.
- `validate_hex64` (`validate.rs:28-36`) and `is_lower_hex_sha256`
  (`client.rs:260-262`) are two hex validators in one crate with different
  case-sensitivity rules.

#### Oversized files and functions

`client.rs` 2,477 lines (production ends at 1,433; 42% is tests) and `lib.rs`
2,035 lines — both above the 1,000-line ceiling the repo enforces for Dart via
`justfile:617`, with no Rust equivalent (`grep -rn 'check-file-sizes' justfile`
matches only the mobile recipe). `lib.rs` is one flat clap declaration: 22
subcommand enums plus the dispatch match, with no submodule split (a
`cli/` module tree mirroring `commands/` would be the obvious cut).
`submit_moderation_event` (`client.rs:873-1022`, ~150 lines, 5 retry branches,
one `unreachable!`) and `upload_file` (`client.rs:1100-1227`) are the two
functions most in need of decomposition.

#### Untested critical paths

| Behavior | Status |
|---|---|
| `exit_code` — the contract `AGENTS.md:189-190` publishes | **no test**: `grep -c 'exit_code' <(awk 'NR>=137' error.rs)` → 0. I verified 6 of the 12 mappings by running the built binary; `Conflict`(5), `NotFound`(1), `DeliveryUnknown`(2), `Other`(4) and both `Relay` branches remain unexercised anywhere |
| `print_error` | never invoked by a test; the two JSON-shape tests rebuild its object inline (`error.rs:197-210`, `:213-221`), so the production category strings (`error.rs:109-126`) are untested |
| `normalize_relay_url` / `to_ws_url` | no tests (`grep -c 'normalize_relay_url' <(awk 'NR>=1434' client.rs)` → 0). The ws↔http mapping every command depends on, and the `str::replace` non-prefix-anchored behavior, are unverified |
| `normalize_events` | no tests — the sig-stripping read envelope `AGENTS.md:188-189` documents |
| `normalize_write_response` | no tests, despite 47 call sites in `commands/` |
| `build_imeta_tag` | no tests |
| `sign_nip98` | no direct test; only observed indirectly through integration tests that assert an `Authorization` header exists (`client.rs:2193-2198`) |
| `extract_d_tag` / `extract_tag_value` / `extract_p_tags` | no tests; `extract_p_tags` contains the group's one bare `unwrap()` on relay data (`client.rs:1379`) |
| `mem` and `notes` command inventories | absent from `subcommand_names_are_stable` — grepping lines 1855-2034 of `lib.rs` for `"mem"` returns zero matches — and from `subcommand_counts_are_stable`, whose list covers 18 of 21 groups (`lib.rs:1997-2016`); the two largest command modules by size have no drift guard |

#### Panic-capable production paths

`advance_query_cursor`'s `.expect("a full query page always has a last event")`
(`client.rs:504-506`) and `extract_p_tags`'s `t.as_array().unwrap()`
(`client.rs:1379`) both violate the AGENTS.md rule against `unwrap`/`expect` in
production paths and both trigger on relay-shaped data rather than programmer
error. `jitter_delay` (`client.rs:133-135`) indexes a fixed 2-element array by
attempt number and is safe only because `retry_constants_are_sensible`
(`client.rs:1547-1552`) pins `RETRY_BASE_SECS.len() == RETRY_MAX_ATTEMPTS - 1`;
raising `RETRY_MAX_ATTEMPTS` alone would panic at runtime.

#### Silent no-op configuration

`--format` reaches only 5 of 21 dispatchers (`lib.rs:1772-1790`), so
`buzz --format compact notes ls` accepts the flag and ignores it. Nothing warns.
For an agent-first CLI whose calling convention is documented around
`--format compact` (`AGENTS.md:179-193`), a silently-ignored output mode is a
correctness trap rather than a cosmetic gap.

#### Documentation drift

| Claim | Reality |
|---|---|
| `AGENTS.md:181` example `buzz messages thread … --format compact` | Cannot parse — `--format` is not `global`; verified exit 1. Directly contradicts `AGENTS.md:192-193`, which states the correct position |
| `AGENTS.md` gotcha 3: "`messages search` must include `--kinds` … Pass at least `--kinds 9,45001,45003`" | `messages search` has no `--kinds` flag (`lib.rs:472-489`); verified `--kinds 9` → exit 1 `unexpected argument`. Kinds are hardcoded downstream (`commands/messages.rs:361`) |
| `AGENTS.md:188-189` "All reads return sig-stripped JSON arrays" | Only `normalize_events` strips `sig` (`client.rs:1307-1323`) and just two command modules call it (`commands/feed.rs`, `commands/messages.rs`); other reads print the relay body verbatim (`commands/social.rs:78`, `commands/issues.rs:32`). Whether the relay's serializer omits `sig` is outside this group and unverified |
| `AGENTS.md:189-190` exit-code table | Matches the six named classes, but omits that `NotFound` also maps to 1 (`error.rs:104`) and `DeliveryUnknown` to 2 (`error.rs:105`), so exit code alone cannot distinguish "may have been written" from "network failed" |
| `AGENTS.md:145-147` documented HTTP surface | The CLI also GETs `/moderation/reports`, `/moderation/restricted`, `/moderation/audit` (`commands/moderation.rs:114-128` via `client.rs:836-856`); `grep -c '/moderation/' AGENTS.md` → 0 |
| `README.md:216-221` architecture diagram (`main.rs ──▶ commands/*.rs`) | `main.rs` is 4 lines; the clap tree and dispatch live in `lib.rs:62-1792`. The diagram also labels the target "Buzz Relay REST API" when the surface is the Nostr HTTP bridge plus Blossom |
| `README.md` command table | Missing 8 of 21 groups: `agents`, `canvas` is present but `emoji`, `notes`, `patches`, `pr`, `issues`, `media`, `moderation` are absent, and `pack`/`mem` are listed. Compare `lib.rs:1808-1829` |
| `README.md:14-20` auth table | Lists only `BUZZ_PRIVATE_KEY`; omits `BUZZ_AUTH_TAG`, which this layer verifies and transmits (`lib.rs:1752-1767`) |
| `TESTING.md:37-73` token/scope model | Contradicted by `lib.rs:1745-1746` ("no tokens, no other auth"); also `TESTING.md:26` points at a stale `REPOS/buzz-nostr` path, and `TESTING.md:82` says "see cargo test for current count" instead of a number |
| `Cargo.toml:19` "(BUZZ_API_TOKEN auto-wired)" | Never read here (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → 0) |
| `Cargo.toml:44` "used in `buzz auth`, auto-mint" | No `auth` subcommand exists (`lib.rs:1808-1829`) |
| `validate.rs:28` "64-character lowercase hex" | Accepts uppercase (`validate.rs:30`) |
| `BUZZ_TIMEOUT_SECS`, `BUZZ_CONNECT_TIMEOUT_SECS` | Documented only in a doc comment (`client.rs:535-540`); absent from `.env.example`, `AGENTS.md` and README |

In-code `file:line` cross-references were checked and are **accurate**:
`client.rs:1077-1080` cites `buzz_ws_client::{AUTH_CHALLENGE_TIMEOUT_SECS,
AUTH_OK_TIMEOUT_SECS, PUBLISH_OK_TIMEOUT_SECS}`, which are 20/20/30 s at
`crates/buzz-ws-client/src/connection.rs:17-23`, summing to the 70 s the comment
claims and justifying the 75 s budget. `Cargo.toml:78-83` correctly points at
`crates/buzz-acp/Cargo.toml` for the matching rustls dependency.

#### Structural debt worth naming

- **Untyped filters.** Every relay query is a hand-built `serde_json::Value`
  (`client.rs:683-801`), so filter-shape errors (a missing `kinds`, a misspelled
  tag key) are relay-side 4xx rather than compile errors. This is the root cause
  of the AGENTS.md p-gate gotchas.
- **Stringly-typed responses.** `submit_event` and friends return `String`
  (`client.rs:863`), pushing JSON re-parsing into 47 `normalize_write_response`
  call sites instead of returning a typed `WriteResult`.
- **`sign_event_unchecked` has no guard rail.** Its "NIP-IA kinds 9035/9036 only"
  restriction is a doc comment (`client.rs:729-742`); a kind assertion would make
  the invariant enforceable rather than reviewable.
- **WS errors lose their class.** Every `buzz_ws_client` failure — timeout, NIP-42
  rejection, connect refusal — becomes `CliError::Other` → exit 4
  (`client.rs:1084`), while the HTTP equivalents are 2/3. Agents branching on
  exit codes see ephemeral publishes as a different failure category from every
  other write.
