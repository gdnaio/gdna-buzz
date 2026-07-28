## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Conventions

#### Error handling and `CliError` mapping style

`CliError` (`error.rs:4-49`) has nine variants; exit codes come from `error::exit_code`
(`error.rs:89-107`). Three distinct mapping styles coexist in this group:

| Style | Sites | Effect |
|---|---|---|
| `map_err(sdk_err)` — the shared mapper (`validate.rs:151-156`) | `pr.rs:58`, `:100`, `:209`; `patches.rs:50`, `:169`, `:183`; `issues.rs:29`, `:140`; `workflows.rs:108`, `:130`, `:145`, `:185`, `:207`; `users.rs:299` | `SdkError::InvalidInput` → `Usage` (exit 1); **everything else** → `Other` (exit 4) |
| inline `map_err(\|e\| CliError::Usage(...))` | `moderation.rs:43`, `:54`, `:72`, `:82`, `:98`; `agents.rs:104`, `:134`; `repos.rs:82`, `:105`, `:333` | forces exit 1 regardless of the underlying `SdkError` variant |
| inline `map_err(\|e\| CliError::Other(...))` | `repos.rs:57`, `:124-126`, `:137`, `:220`; `users.rs:206`; `mem.rs:95`, `:356`; `agents.rs:36`, `:78`, `:273`, `:275` | exit 4 — note `repos.rs:124-126` classifies *pre-existing malformed rules on the fetched event* as internal, which is arguably right (not the caller's input) but reads oddly next to `repos.rs:82` |

The consequence of the first style is worth stating plainly: a user handing in a 100 KiB PR
body gets `SdkError::ContentTooLarge` (`builders.rs:35-41`, cap applied at
`builders.rs:1335`), which `sdk_err` routes to `CliError::Other` → **exit 4 ("other")**, not
exit 1 ("input error"). The same over-size input via `moderation resolve` would be exit 1,
because `moderation.rs:98` hardcodes `Usage`. So the same class of user mistake produces two
different exit codes depending on which command it hits. `AGENTS.md` documents exit 1 as
"input error" and 4 as "other".

`?`-propagation is the norm; no `.ok()`-swallowing of relay errors. Deliberate error
suppression is confined to three places and each carries a comment explaining it:
`mem.rs:120-128` (a corrupt event must not deny-of-service a listing),
`mem.rs:161-170` (undecryptable engrams are skipped, not fatal), and
`agents.rs:204-206` (a structurally malformed `auth` tag means "bare", not error).

Fallible-to-default conversions are pervasive on the read path — `serde_json::from_str(&resp)
.unwrap_or_default()` at `users.rs:47`, `users.rs:259`, `workflows.rs:20`, `workflows.rs:45`,
`workflows.rs:78`. These silently turn a malformed relay response into an empty array, so
`workflows list` on a broken relay prints `[]` and exits 0. The stricter modules parse and
error: `mem.rs:115-117`, `repos.rs:12-13`, `agents.rs:280-281`.

#### Output discipline: stdout, stderr, and which helper prints

The group's stated contract (`crates/buzz-cli/README.md:3`) is one-line JSON on stdout,
diagnostics on stderr. Four distinct printing conventions are in use:

| Convention | Commands | Sites |
|---|---|---|
| `println!("{}", normalize_write_response(&resp))` — the shared normalizer (`client.rs:1420-1434`) | all five `moderation` mutations, `users set-profile`, `users set-presence`, `workflows update/delete/trigger/approve` | `moderation.rs:47`, `:57`, `:75`, `:85`, `:101`; `users.rs:212`, `:303`; `workflows.rs:134`, `:148`, `:182`, `:187`, `:210` |
| `print_create_response` (`client.rs:1401-1403`) | `workflows create` only | `workflows.rs:112` |
| bare `println!("{resp}")` — raw relay body, unnormalized | `repos create/get/list`, all four `pr` reads/writes that aren't `protect`, all four `patches`, all four `issues`, all three `moderation` reads | `repos.rs:228`, `:252`, `:280`; `pr.rs:61`, `:103`, `:114`, `:147`, `:212`; `patches.rs:53`, `:80`, `:109`, `:186`; `issues.rs:32`, `:43`, `:76`, `:143`; `moderation.rs:115`, `:121`, `:129` |
| hand-built JSON via `serde_json::json!` | `agents archive/unarchive/archived`, `agents draft-*` (relay body reparsed and augmented) | `agents.rs:108-115`, `:138-145`, `:312`; `agents.rs:35-42`, `:77-84` |

So within one command family the discipline splits: `repos protect set/remove` normalize
(`repos.rs:198` → `validate_write_response` → `normalize_write_response`, `repos.rs:192`)
while `repos create` in the same file does not (`repos.rs:228`). `normalize_events`
(`client.rs:1307-1323`) exists and is used by other command families, but **no file in this
group calls it** — `grep -n 'normalize_events' ` across the eleven returns zero matches;
`workflows.rs` and `users.rs` hand-roll equivalent projections instead
(`workflows.rs:22-31`, `workflows.rs:79-89`, `users.rs:50-62`, `users.rs:260-269`).

Non-JSON stdout — three deliberate exceptions, each documented:

- `mem get` writes the raw value with no trailing newline (`mem.rs:296`, `mem.rs:300`),
  explained at `mem.rs:295` so it round-trips with `mem set <slug> -`.
- `mem ls` without `--json` prints TSV (`mem.rs:268`) and puts the empty-case notice
  `(no memories besides core)` on **stderr** (`mem.rs:265`) leaving stdout empty.
- `pack validate` / `pack inspect` print human-readable text only (`pack.rs:40-42`,
  `pack.rs:66-147`).
- `upload file` is the group's only pretty-printer (`upload.rs:8-11`).

stderr is used for two things: `error::print_error`'s JSON error object
(`error.rs:112-137`) and `mem`'s progress/confirmation lines — `mem.rs:370`, `:671-674`,
`:696`, `:733` — plus `pack`'s diagnostics (`pack.rs:29`, `pack.rs:32`). `mem set`/`rm`
therefore print **nothing to stdout on success**, which makes them the only writes in the
group whose result cannot be piped into `jq`.

#### Naming patterns

| Pattern | Convention | Deviations |
|---|---|---|
| Public handler | `cmd_<verb>[_<noun>]`, `async`, takes `&BuzzClient` first | `agents.rs` has **no** `cmd_*` handlers — all five subcommands are inline arms of `dispatch` (`agents.rs:14-157`); `pack.rs`'s two are sync and take no client (`pack.rs:15`, `pack.rs:52`) |
| Verb/noun order | inconsistent: `cmd_create_repo`/`cmd_get_repo`/`cmd_list_repos` (`repos.rs:202`, `:232`, `:256`) put the verb first, but `cmd_send_patch`/`cmd_patch_status` (`patches.rs:9`, `:114`) and `cmd_open_pr`/`cmd_pr_status` (`pr.rs:20`, `:152`) mix both orders in one file | — |
| Dispatch entry | `pub async fn dispatch(cmd, client)` | `upload.rs` needs two (`dispatch`, `dispatch_media`, `upload.rs:4`, `:17`); `users.rs:307` and `moderation.rs:133` add a third `&OutputFormat` parameter |
| Pure helper | private, snake_case, no `cmd_` prefix — `resolve_owner` (`mem.rs:33`), `sha256_hex` (`mem.rs:377`), `extract_owner_auth_tag` (`agents.rs:191`), `protection_pattern` (`repos.rs:48`), `presence_subject` (`users.rs:279`), `resolve_expiry` (`moderation.rs:26`), `parse_committer` (`patches.rs:61`) | consistent |
| Cross-module helper | `pub(crate)` — `patches::parse_status` (`patches.rs:194`), `agents::fetch_archived_snapshot` (`agents.rs:270`) | consistent |

#### `unsafe` and lint attributes

`unsafe`: **zero occurrences.** `grep -rn 'unsafe'` across the eleven files returns no
matches, matching `AGENTS.md:114`.

Lint attributes: seven, all the same one —
`#[allow(clippy::too_many_arguments)]` at `mem.rs:537`, `pr.rs:19`, `pr.rs:65`, `pr.rs:151`,
`patches.rs:8`, `patches.rs:113`, `issues.rs:80`. No `#[deny]`, no `#[warn]`, no crate-level
`#![...]` in `lib.rs` or `main.rs` (`grep -n '^#!\[' ` on both returns zero). The suppressed
functions have 8-15 parameters (`cmd_open_pr` has 15, `pr.rs:20-36`) — the flag-per-parameter
shape is the group's accepted cost of avoiding a params struct.

#### `unwrap()` / `expect()` / `panic!` / `unreachable!` on production paths

`AGENTS.md:115`: "Do not introduce new `unwrap()` or `expect()` in production paths — use
`?` and proper error types."

Scanning only the region above each file's `#[cfg(test)]` (or the whole file where there is
no test module — `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`), there
is **exactly one violation**:

- `agents.rs:299` — `let raw_event = events.into_iter().next().unwrap();`

It is guarded: `agents.rs:294-296` returns early when `events.is_empty()`, so it cannot panic
today. It is also unnecessary — the surrounding function already returns
`Result<Vec<String>, CliError>` and the two lines below it (`agents.rs:300-301`) use `?`. A
`let Some(raw_event) = … else` would cost one line. Zero `expect(`, zero `panic!`, zero
`unreachable!` on any production path in the group. (For contrast, the shared client carries
two: `unreachable!("loop exhausts all RETRY_MAX_ATTEMPTS")` at `client.rs:680` and
`client.rs:1007` — outside this group's files.)

Nothing enforces the rule mechanically. `just clippy` runs
`cargo clippy --workspace --all-targets -- -D warnings` (`justfile:107`), but
`clippy::unwrap_used` and `clippy::expect_used` are allow-by-default and are not enabled
anywhere: there is no `[lints]` table in the root `Cargo.toml`, no `clippy.toml`
(`ls clippy.toml` → "No such file or directory"), and no crate-level `#![deny]`. So
`AGENTS.md:115` is convention, reviewed by humans, not by CI.

`unwrap_or*` is used freely and is not a violation — it is the group's idiom for
best-effort output projection (`users.rs:47`, `workflows.rs:20`, `mem.rs:263`) and for
clock reads (`now_secs`'s `.unwrap_or(0)`, `mem.rs:85`, which would silently date a write to
the epoch if the system clock preceded 1970; `engram::monotonic_created_at` at `mem.rs:363`,
`:689`, `:726` covers the consequence).

#### Doc-comment discipline

`AGENTS.md:116`: "New public API must have doc comments."

| File | `//!` module doc | `pub fn` count | Undocumented `pub fn` |
|---|---|---|---|
| `mem.rs` | 15 lines, `mem.rs:1-15` | 7 | 1 — `dispatch` (`mem.rs:737`) |
| `moderation.rs` | 15 lines, `moderation.rs:1-15` — the best in the group; explains *why* `/moderation/*` is REST | 1 | 1 — `dispatch` (`moderation.rs:133`) |
| `pack.rs` | 3 lines, `pack.rs:1-3` | 2 | 0 |
| `agents.rs` | none | 1 | 1 — `dispatch` (`agents.rs:12`) |
| `repos.rs` | none | 4 | 4 — `cmd_create_repo` (`:202`), `cmd_get_repo` (`:232`), `cmd_list_repos` (`:256`), `dispatch` (`:349`) |
| `users.rs` | none | 5 | 2 — `cmd_set_profile` (`:150`), `dispatch` (`:307`) |
| `pr.rs` | none | 6 | 6 — including `cmd_open_pr` (`:20`) and `cmd_update_pr` (`:66`), whose preceding line is the `#[allow]` attribute, not a doc |
| `patches.rs` | none | 5 | 5 |
| `issues.rs` | none | 5 | 5 |
| `workflows.rs` | none | 9 | 1 — `dispatch` (`:214`); the other 8 are documented |
| `upload.rs` | none | 2 | 2 |

Eight of eleven files have no module-level doc. The three that do (`mem.rs`,
`moderation.rs`, `pack.rs`) are also the three whose *private* helpers are documented most
thoroughly — `verify_hunks_at_declared_position` carries a 17-line rationale
(`mem.rs:383-399`) plus a named known limitation and a PR reference inline at
`mem.rs:422-428` ("see PR #627 review"). `agents.rs` has no module doc but its private
verification helpers are well documented (`agents.rs:163-171`, `:198-205`, `:246-249`,
`:255-269`, `:319-322`). `pr.rs`,
`patches.rs`, `issues.rs` and `upload.rs` are the thin spots: mechanical arg-shuffling code
with almost no prose, and where comments exist they restate the NIP rather than the code
(`patches.rs:152-155`, `issues.rs:118-121`).

#### File-size discipline

`AGENTS.md:533` documents a **1000-line-per-file** ceiling, but scoped to the mobile app and
enforced by `mobile/scripts/check-file-sizes.mjs`. Equivalent guards exist for desktop
(`desktop/scripts/check-file-sizes.mjs`, wired at `desktop/package.json:11` and `:15`) and
web (`web/scripts/check-file-sizes.mjs`, `web/package.json:10`). **No Rust gate exists** —
`just check` (`justfile:95`) composes `fmt-check clippy desktop-check desktop-tauri-fmt-check
desktop-tauri-clippy web-check mobile-check`; none of those runs a line count over
`crates/`. `ls` of `crates/*/scripts` finds no such script.

Current sizes: `mem.rs` 1045, `agents.rs` 718, `repos.rs` 644, `users.rs` 359, `pr.rs` 342,
`patches.rs` 323, `workflows.rs` 243, `issues.rs` 198, `moderation.rs` 165, `pack.rs` 151,
`upload.rs` 36.

So `mem.rs` at 1,045 lines is the one file in the group that would trip the ceiling the
other three platforms enforce. Its production region is 779 lines (`#[cfg(test)]` at
`mem.rs:780`) with 266 lines of tests. The same is true across the group: the largest files
are large because of their test modules — `agents.rs` is 383 production + 335 test
(`#[cfg(test)]` at `agents.rs:383`), `repos.rs` 401 + 243 (`repos.rs:401`).

#### Test organization across the eleven files

All tests are inline `#[cfg(test)] mod tests` at the bottom of the file. Six of eleven files
have no tests at all. There is no `crates/buzz-cli/tests/` directory, and no async test
anywhere — `grep -rn 'tokio::test' crates/buzz-cli/src/commands/` returns zero matches, so
**no `dispatch` arm and no relay interaction in this group is exercised**.

| File | Tests | Focus |
|---|---|---|
| `mem.rs` | 15 (`mem.rs:793`-`:1044`) | `resolve_reader` matrix, `sha256_hex` vectors, diffy behavior, strict-position checker |
| `agents.rs` | 26 (`agents.rs:401`-`:717`) | `extract_owner_auth_tag` (14 cases), `normalize_relay_self_hex` (3), `verify_archived_event` tri-state (9) |
| `repos.rs` | 9 (`repos.rs:423`-`:643`) | protection tag build/update/remove, list JSON, write-response conflict split |
| `users.rs` | 3 (`users.rs:343`, `:349`, `:355`) | `presence_subject` fallback only |
| `pr.rs` | 2 (`pr.rs:334`, `:339`) | `read_optional_body` |
| `patches.rs` | 4 (`patches.rs:284`, `:298`, `:304`, `:319`) | `parse_committer`, `parse_status` |
| `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs` | **0** | — |

Naming is descriptive-assertion style (`protection_update_preserves_metadata_and_replaces_only_matching_pattern`,
`repos.rs:423`; `archived_state3_lone_malformed_nip70_tag_errors`, `agents.rs:650`) and
several carry provenance comments tying them to a review finding — `F5` at `agents.rs:665-669`,
`F6` at `agents.rs:500-503`, `F7` at `agents.rs:544-547`, "Max's offset-search case" at
`mem.rs:923-927`. Shared fixtures are per-file, not shared: `hex64`/`hex128`
(`agents.rs:390`, `:394`), `signed_repo`/`tag` (`repos.rs:410`, `:418`),
`test_client` (`mem.rs:788`), `build_archived_event` (`agents.rs:559`).

Tests that assert against a **copy** of a production rule rather than calling it:

- `multi_file_header_count` (`mem.rs:1037-1044`) re-implements the multi-file guard inline —
  `single.lines().filter(|l| l.starts_with("--- ")).count()` — instead of exercising the
  production check at `mem.rs:618`. If `cmd_patch`'s predicate changed to `"--- a/"` or moved
  to a helper, this test would still pass. Its comment (`mem.rs:1031-1036`) explains *why the
  rule is what it is* but not why it is duplicated rather than called.
- `diffy_apply_refuses_mismatched_context` (`mem.rs:878`),
  `diffy_apply_succeeds_on_exact_context` (`mem.rs:895`) and
  `diffy_roundtrip_preserves_content` (`mem.rs:915`) test the third-party `diffy` crate, not
  `buzz-cli` code. That is deliberate and stated (`mem.rs:874-876`: "if a future diffy
  upgrade loosens it, this test catches it") — a dependency-pinning test, correctly labelled,
  but it means three of `mem.rs`'s fifteen tests cover no first-party line.
- `protection_update_enforces_repository_rule_limit` (`repos.rs:551`) asserts
  `error.to_string().contains("exceeds max 50")` — the "50" is `buzz-core`'s rule, reached
  through `parse_protection_tags` (`repos.rs:126`). It does call production code, but its
  assertion is coupled to another crate's message text.

By contrast the strongest tests in the group do call the real function and assert on named
failure modes: `strict_position_rejects_offset_slide` (`mem.rs:928`) first demonstrates that
`diffy::apply` *would* slide the hunk, then asserts
`verify_hunks_at_declared_position` refuses — a genuinely discriminating test.

Notable coverage gaps beyond the six untested files: `resolve_expiry` (`moderation.rs:26-32`,
including the unchecked `Timestamp::now().as_secs() + secs` at `moderation.rs:28`),
`cmd_get_workflow`'s `null` branch (`workflows.rs:55`), the `--inputs` hand-rolled trigger
builder (`workflows.rs:170-181`), the `RepoPushRole` → string mapping (`repos.rs:310-314`),
and the `/moderation/*` query-string construction (`moderation.rs:110-112`).

#### Where I am uncertain

- The "undocumented `pub fn`" counts come from checking the line immediately preceding each
  `^pub (async )?fn`. A doc comment separated from its item by a blank line or an attribute
  block would be miscounted; I hand-checked `pr.rs` and `patches.rs` and found the attribute
  case, but did not hand-verify all 51 `pub fn` sites.
- I did not run `just ci`, so the claim that no gate rejects `agents.rs:299` is from reading
  `justfile:95-107` and the absence of lint configuration, not from an observed clean run.
