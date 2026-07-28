## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Debt

Scope: the eleven files `mem.rs`, `agents.rs`, `repos.rs`, `users.rs`, `pr.rs`,
`patches.rs`, `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`.
`lib.rs`/`client.rs`/`validate.rs` are cited only to resolve a call or a rule these
files depend on.

#### Duplication

**`parse_events` exists three times in the crate, two byte-identical.**

| Site | Body | Behaviour |
|---|---|---|
| `repos.rs:11-14` | `serde_json::from_str(json).map_err(…"failed to parse relay response: {error}")` | strict — one bad event fails the whole response |
| `notes.rs:156-159` | byte-identical to the above (same message string, same shape) | strict |
| `mem.rs:114-136` | per-element `serde_json::from_value`, skipping failures | lenient, deliberately (comment at `mem.rs:118-120`, `:126-128`) |

The strict pair is copy-paste; the lenient one is a genuinely different policy that
happens to share a name. Nothing in `client.rs` offers either
(`grep -n 'fn parse_events' crates/buzz-cli/src/client.rs` → zero matches), so the
duplication has no shared home to move to.

**Duplicate-write detection is re-derived instead of shared.** See the LWW lead
below — `submit_engram` (`mem.rs:93-111`) and `repos::validate_write_response`
(`repos.rs:173-193`) implement the same three-rule ladder independently, and only
the latter finishes by calling `client::normalize_write_response`
(`client.rs:1420`).

**The NIP-34 `#a` coordinate is hand-formatted in three files.**
`format!("30617:{repo_owner}:{repo_id}")` at `patches.rs:94`, `pr.rs:129`,
`issues.rs:58`. `buzz-sdk` already models this as `GitRepoCoord`
(`patches.rs:8`, `pr.rs:6`, `issues.rs:4` all import it and use it for the *write*
side), so the read side is the only place the string form is rebuilt by hand — and it
embeds a bare kind integer while doing so.

**The `(repo_owner, repo_id)` pairing match is triplicated verbatim.**
`patches.rs:135-150`, `pr.rs:168-183`, `issues.rs:98-113` — same three arms, same
`"--repo-owner and --repo-id must be given together"` message. Redundant besides:
clap already declares `requires = "repo_id"` / `requires = "repo_owner"`
(`lib.rs:1269-1273`, `lib.rs:1423-1427`, `lib.rs:1501-1505`), so the runtime arm is
unreachable through the CLI.

**The recipient-dedup loop is triplicated verbatim.** `patches.rs:156-163`,
`pr.rs:187-194`, `issues.rs:118-125` — identical `Vec` + `validate_hex64` +
`contains` push. `O(n²)` in `--to` count, which is fine at these sizes but is
copy-paste all the same. Each copy carries its own comment explaining the same rule
(`patches.rs:152-155`, `pr.rs:185-186`, `issues.rs:115-117`).

**`users.rs`'s compact projection is written twice.** The
`OutputFormat::Compact => { … "display_name": p.get("display_name").or_else(|| p.get("name")) … }`
block at `users.rs:63-74` and `users.rs:134-145` are identical apart from the source
variable. Both then repeat the `serde_json::to_string(...).unwrap_or_default()` tail.

**`agents.rs` duplicates the draft-response augmentation.** `agents.rs:26-52`
(draft-create) and `agents.rs:62-88` (draft-update) differ only in the builder call;
the parse, the four `obj.insert` calls, and the identical
`"Draft sent to Buzz Desktop for owner review…"` literal are copied. That literal
appearing twice means a wording change needs two edits.

**64-hex / 128-hex validation is re-implemented rather than calling
`validate::validate_hex64`.** `validate_hex64` exists at `validate.rs:29-36` and is
imported by `agents.rs:9`, yet:

- `agents.rs:231-236` re-checks owner (64) and sig (128) shape inline.
- `agents.rs:251-256` re-checks 64-hex in `normalize_relay_self_hex`.
- `mem.rs:568-574` re-checks 64-hex for `--base-hash`.
- `agents.rs:696` (test) and `verify_archived_event`'s `p`-tag filter
  (`agents.rs:361-365`) each carry a fourth copy of the same predicate.

The `sig`-length-128 check has no shared helper at all, so that one is unavoidable
today; the 64-hex ones are not.

**Event-JSON projection duplicates `client::normalize_events`.**
`workflows.rs:82-89` builds `{event_id, kind, content, created_at, tags}`;
`client::normalize_events` (`client.rs:1307-1322`) builds
`{id, pubkey, kind, content, created_at, tags}`. The shapes differ only in the id key
name and `pubkey`, and no in-scope file calls the shared helper
(`grep -c 'normalize_events'` across the eleven → `0` for every file). `workflows.rs`
also repeats its own projection three times with small variations
(`workflows.rs:23-30`, `:47-53`, `:82-89`).

**`d`-tag extraction is inconsistent.** `workflows.rs` uses the shared
`client::extract_d_tag` (`workflows.rs:24`, `:49`), while `repos.rs:33-45` hand-rolls
`repo_id_from_event` and `mem.rs:212-220` hand-rolls a third variant using
`t.kind().to_string() == "d"` — a string comparison on a tag-kind enum, which is the
most fragile of the three.

#### Stale comments and cross-references

**`users.rs:5` is a stale TODO.**
`// TODO(phase-4): Replace raw nostr::EventBuilder usage in cmd_set_presence with buzz-sdk builder`
— but `cmd_set_presence` already uses `buzz_sdk::build_presence_update`
(`users.rs:299`). `grep -c 'EventBuilder' crates/buzz-cli/src/commands/users.rs`
returns `1`, and that one match is the TODO comment itself. The work is done; the
note is not.

**`workflows.rs:10` is a live TODO, correctly stated.** Same wording, and
`cmd_trigger_workflow` does still construct a raw `EventBuilder`
(`workflows.rs:172-177`) on the `--inputs`-provided branch only, while the
no-inputs branch uses `buzz_sdk::build_workflow_trigger` (`workflows.rs:181`). So
one command has two divergent build paths for the same kind.

**In-code cross-references — both resolve.**

| Reference | Resolves? |
|---|---|
| `agents.rs:263` → `channels::resolve_roster_with_archive_filter`'s doc comment | yes — `channels.rs:526`, and `channels.rs:11` does import `fetch_archived_snapshot` as claimed |
| `mem.rs:428` → "see PR #627 review" | not verifiable from the working tree (no in-repo artifact); the *technical* claim next to it — that a no-context insertion into a non-empty value is rejected rather than mis-applied — matches `mem.rs:421-437` |

`grep -n '\.rs:' ` across the eleven files returns zero matches, so there are no
in-code `file:line` cross-references to go stale. The only other prose pointer is
`workflows.rs:60-66`, whose claim that the relay does not emit kinds 46001-46003 and
that run history lives in the `workflow_runs` table is a statement about another
crate that I did not verify.

**`agents.rs:255-269`'s doc comment describes `verify_archived_event` via
`[`verify_archived_event`]` intra-doc link on a private function** — the link target
is `fn verify_archived_event` at `agents.rs:320`, which is private, so the link
resolves in-crate but not in published docs for the `pub(crate)` caller.

#### `unwrap()` / `expect()` on production paths

`AGENTS.md § Quality Gates` says "Do not introduce new `unwrap()` or `expect()` in
production paths." Filtering out `unwrap_or*` and `#[cfg(test)]` modules, the group
has exactly **one** violation:

- `agents.rs:299` — `let raw_event = events.into_iter().next().unwrap();`

It cannot panic today: `agents.rs:294-296` returns early on `events.is_empty()`
immediately above. It is still a latent panic if that guard is ever moved, and
`.next().ok_or_else(…)` would cost nothing. Every other match is inside
`mod tests` (`mem.rs:789`+, `agents.rs:407`+, `repos.rs:415`+, `patches.rs:285`+,
`pr.rs:340`).

No `unreachable!()` or `panic!()` in the eleven files
(`grep -n 'unreachable!\|panic!'` → zero matches). Note the group's one `unreachable!`
lives in its dispatcher rather than in the module: `lib.rs:1791`
(`Cmd::Pack(_) => unreachable!("handled above")`), guarded by the early return at
`lib.rs:1737-1742`.

**Silent-failure `unwrap_or_default()` on parse paths is the bigger risk than the
unwraps.** `serde_json::from_str(&resp).unwrap_or_default()` turns a malformed relay
response into an empty array with no error and exit code 0:

| Site | Command |
|---|---|
| `users.rs:47` | `users get` |
| `users.rs:263` | `users presence` |
| `workflows.rs:20` | `workflows list` |
| `workflows.rs:45` | `workflows get` (then prints `null`, `workflows.rs:55`) |
| `workflows.rs:79` | `workflows runs` |
| `users.rs:242-243` | `users set-profile`'s read-merge-write — a malformed current profile silently becomes `{}`, so **a merge can drop fields it was supposed to preserve** |

Contrast `mem.rs:95-96`, `agents.rs:36`, `repos.rs:174-175`, which all surface a
parse failure as `CliError::Other`. The inconsistency is the debt: same operation,
two opposite failure policies, no comment explaining either.

`users.rs:242-243` is the one with a correctness consequence rather than just a
misleading exit code, and it is untested — there is no `#[test]` for
`fetch_current_profile` or `cmd_set_profile`
(`grep -n '#\[test\]' crates/buzz-cli/src/commands/users.rs` returns three tests, all
for `presence_subject`: `users.rs:343`, `:349`, `:355`).

#### `#[allow(…)]` attributes

Seven `#[allow(clippy::too_many_arguments)]`, no `#[allow(dead_code)]`
(`grep -n '#\[allow' ` across the eleven returns exactly the seven below; `dead_code`
→ zero matches):

| Site | Function | Arity |
|---|---|---|
| `mem.rs:537` | `cmd_patch` | 8 |
| `pr.rs:19` | `cmd_open_pr` | 16 |
| `pr.rs:65` | `cmd_update_pr` | 12 |
| `pr.rs:151` | `cmd_pr_status` | 10 |
| `patches.rs:8` | `cmd_send_patch` | 13 |
| `patches.rs:113` | `cmd_patch_status` | 12 |
| `issues.rs:80` | `cmd_issue_status` | 8 |

These are suppressions standing in for a refactor that the codebase already knows how
to do: `buzz-sdk` provides exactly the right parameter-object types
(`GitPatchMeta`, `GitPullRequestMeta`, `GitStatusMeta`, `GitIssueMeta`,
`GitPrUpdateMeta`) and each of these functions *constructs one immediately*
(`patches.rs:31-40`, `pr.rs:43-56`, `pr.rs:95-102`, `patches.rs:167-176`,
`issues.rs:21-24`). Taking the meta struct as the parameter instead of 12 loose
arguments would delete the `allow` and the `dispatch` arm's positional-argument
hazard in one move. There is no `TODO` next to any of the seven.

The one clippy-adjacent gap with no `allow`: `dispatch` in `agents.rs:12-154` is a
143-line match arm block, and `mem.rs:737-778`'s dispatch re-lists `cmd_patch`'s
eight arguments positionally (`mem.rs:764-775`) — a silent-reorder hazard that no
test covers.

#### Dead code and unused capability

- No unused items in the eleven files: `cargo`'s own dead-code lint is on (no
  `#[allow(dead_code)]` anywhere here) and every function has at least the local
  `dispatch` as a caller.
- `validate::MAX_CONTENT_BYTES`, `MAX_DIFF_BYTES`, `validate_content_size`,
  `truncate_diff`, `infer_language`, `parse_event_id` are all unreferenced by this
  group (`grep -n 'validate_content_size\|MAX_CONTENT_BYTES\|MAX_DIFF_BYTES\|truncate_diff\|infer_language\|parse_event_id'`
  across the eleven → zero matches). They are live for `messages.rs`, so not dead
  crate-wide, but this group enforces no content ceiling of its own outside `mem`.
- `client::extract_tag_value` (`client.rs:1346`), `extract_p_tags`
  (`client.rs:1366`), `normalize_events` (`client.rs:1307`), `create_response_with_id`
  (`client.rs:1391`), `query_paginated`/`query_all` (`client.rs:715`, `:724`) are all
  unused here — the projection and pagination duplication above is the direct cost.
- `validate::percent_encode` is `#[cfg(test)]`-gated (`validate.rs:76-77`), so the
  crate's only URL-escaper is structurally unavailable to production code. That is
  why `moderation.rs:110-113` interpolates `--status` into a query string raw.

#### File and function sizes vs the documented ceiling

`AGENTS.md:533` documents a **1000-line hard ceiling per file**, enforced by
`mobile/scripts/check-file-sizes.mjs`, "mirroring desktop/web".

| In-scope file | Lines | Over 1000? |
|---|---|---|
| `mem.rs` | 1045 | **yes** |
| `agents.rs` | 718 | no |
| `repos.rs` | 644 | no |
| `users.rs` | 359 | no |
| `pr.rs` | 342 | no |
| `patches.rs` | 323 | no |
| `workflows.rs` | 243 | no |
| `issues.rs` | 198 | no |
| `moderation.rs` | 165 | no |
| `pack.rs` | 151 | no |
| `upload.rs` | 36 | no |

**No Rust gate enforces the ceiling.** The three checkers are
`desktop/scripts/check-file-sizes.mjs`, `web/scripts/check-file-sizes.mjs`,
`mobile/scripts/check-file-sizes.mjs`, wired only into the JS/Dart lanes
(`desktop/package.json:11`, `:15`; `web/package.json:10`, `:13`;
`justfile:617`). `just check` (`justfile:95`) is
`fmt-check clippy desktop-check desktop-tauri-fmt-check desktop-tauri-clippy web-check mobile-check`
— no Rust size step, and `grep -rn 'check-file-sizes' justfile` matches only line 617.
So `mem.rs` at 1045 lines breaches a documented ceiling that nothing checks, and the
sibling out-of-scope modules are further over (`channels.rs` 1713, `notes.rs` 1330,
`messages.rs` 1167 — measured with `wc -l crates/buzz-cli/src/commands/*.rs`). Whether
the ceiling was ever *intended* to bind Rust is genuinely ambiguous: `AGENTS.md:533`
states it inside the **Mobile App § Rules** section, not as a repo-wide rule. Either
the doc should scope it explicitly or a Rust gate should exist — right now it reads
repo-wide and behaves front-end-only.

Oversized functions in `mem.rs` (`grep -n '^pub async fn \|^async fn \|^fn '`):

| Function | Span | Lines |
|---|---|---|
| `cmd_patch` | `mem.rs:538-705` | 168 |
| `cmd_ls` | `mem.rs:189-276` | 88 |
| `verify_hunks_at_declared_position` | `mem.rs:400-483` | 84 |
| `fetch_head` | `mem.rs:136-188` | 53 |

`cmd_patch` does flag validation, base-hash gating, IO, multi-file rejection, parsing,
positional verification, application, size/empty checks, echo, and the write — nine
concerns in one function, and the only one of them with a unit test is the positional
verification (`mem.rs:930`+). `agents.rs`'s `dispatch` (`agents.rs:12-154`, 143 lines)
is the other outlier; the four other in-scope `dispatch` functions are pure routing.

#### Untested critical paths

There is no integration-test directory (`ls crates/buzz-cli/tests` → "No such file or
directory") and no async test anywhere in the group
(`grep -rn 'tokio::test' crates/buzz-cli/src/commands/` → zero matches). Files with
**zero** `#[test]`: `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`,
`upload.rs` (`grep -c '#\[test\]'` → `0` for each).

Untested and consequential:

| Path | Site | Why it matters |
|---|---|---|
| `submit_engram`'s duplicate ladder | `mem.rs:93-111` | the exit-5 producer for `mem set/patch/rm`; only the `repos.rs` twin is tested |
| `resolve_expiry` precedence | `moderation.rs:26-35` | decides whether a ban is permanent or timed |
| `--approved` default-true | `lib.rs:914`, `workflows.rs:193-211` | an omitted flag grants a workflow approval |
| approval `d`-tag = `hex(SHA256(token))` | `workflows.rs:204` | a wrong derivation silently fails to match any pending approval |
| `cmd_trigger_workflow`'s two build paths | `workflows.rs:156-189` | raw `EventBuilder` vs SDK builder for the same kind |
| `users set-profile` read-merge-write | `users.rs:150-214`, `:227-244` | field loss on a malformed current profile |
| `cmd_reports` URL construction | `moderation.rs:105-116` | unescaped `--status` in a NIP-98-signed URL |
| every `dispatch` arm | all eleven | no test constructs a `*Cmd` |
| `pack.rs` path branches | `pack.rs:16-22`, `:53-59`, `:62-63` | |

**Tests that assert against a copy of the rule rather than calling it.**
`mem.rs:1037-1046` (`multi_file_header_count`) re-implements the production predicate
in the test body —
`single.lines().filter(|l| l.starts_with("--- ")).count()` — instead of calling the
code under test at `mem.rs:618`. The production line could change to
`starts_with("---")` or `--- a/` and the test would still pass. It is the clearest
copy-of-the-rule test in the group; the comment above it (`mem.rs:1030-1036`) even
explains the rule it is re-deriving.

Two neighbouring tests assert third-party behaviour rather than this crate's:
`diffy_apply_refuses_mismatched_context` (`mem.rs:885`),
`diffy_apply_succeeds_on_exact_context` (`mem.rs:901`) and
`diffy_roundtrip_preserves_content` (`mem.rs:914`) pin `diffy`, not `mem`. That is
defensible as a dependency-drift canary and the comment at `mem.rs:880-882` says so
— worth noting only because three of the file's fifteen tests exercise no
first-party code.

#### Lead: kindless relay filters and the p-gate

`AGENTS.md § Common Gotchas #2` says a filter without `kinds` triggers a 403. The
actual predicate is `p_gated_filters_authorized`
(`crates/buzz-relay/src/handlers/req.rs:1038-1071`): a kindless filter is treated as
*possibly* p-gated (`is_none_or` at `req.rs:1041-1044`), then exempted if it carries
non-empty `ids` (`req.rs:1064-1066`) or a `#p` all of whose values equal the
authenticated pubkey (`req.rs:1068-1070`).

**Zero in-scope filters are kindless.** Enumerated exhaustively —
`grep -rn 'kinds' ` across the eight files that build filters returns 20 filter
literals plus one comment, and every filter literal carries `kinds`:

| Site | `kinds` |
|---|---|
| `mem.rs:151`, `mem.rs:198` | `[KIND_AGENT_ENGRAM]` |
| `agents.rs:180` | `[0]` |
| `agents.rs:285` | `[KIND_IA_ARCHIVED_LIST]` |
| `repos.rs:21`, `:240`, `:271` | repo announcement (constant, then two literals) |
| `users.rs:42`, `:91`, `:223` | `[0]` |
| `users.rs:258` | `[40902]` |
| `pr.rs:110`, `:131` | `[1618]` |
| `patches.rs:76`, `:96` | `[1617]` |
| `issues.rs:39`, `:60` | `[1621]` |
| `workflows.rs:16`, `:41` | `[30620]` |
| `workflows.rs:74` | `[46001, 46002, 46003]` |

`moderation.rs`, `pack.rs` and `upload.rs` build no filters at all. So the p-gate
exemption analysis is moot for this group — nothing here needs it. Separately, none
of these kinds is in `P_GATED_KINDS` (`crates/buzz-core/src/kind.rs:146-157`), so
the `p_gated_filters_authorized` path returns `true` early for all of them
(`req.rs:1045-1047`). `KIND_AGENT_ENGRAM` is gated by a *different* predicate
documented at `req.rs:1073-1081`, which the `mem` filters satisfy structurally by
always sending both `authors` and `#p` (`mem.rs:151-155`, `mem.rs:198-201`).

The debt here is documentation, not code: `AGENTS.md § Common Gotchas #2` states a
flat rule ("omitting `kinds` triggers the p-gate (403)") that the implementation
contradicts with two exemptions, and `#3` gives `--kinds 9,45001,45003` as the fix
for `messages search` without mentioning that an `ids` lookup needs no kinds at all.

#### Lead: NIP-33 LWW → exit 5, and the `normalize_write_response` split

`mem.rs` **hand-rolls** the detection. `submit_engram` (`mem.rs:93-111`) parses the
`POST /events` reply itself and applies:

1. `accepted` false/absent → `CliError::Other` (`mem.rs:102-104`)
2. `message.starts_with("duplicate:") || message == "duplicate"` →
   `CliError::Conflict` (`mem.rs:105-109`)
3. else `Ok(())`

It does **not** call `client::normalize_write_response` (`client.rs:1420-1435`) —
`grep -n 'normalize_write_response' crates/buzz-cli/src/commands/mem.rs` returns zero
matches, and `mem.rs`'s `use crate::client::BuzzClient` (`mem.rs:29`) imports only the
client type. Consistent with that, `mem set/patch/rm` print human-readable stderr
lines and emit **nothing on stdout** (`mem.rs:370`, `mem.rs:696`, `mem.rs:733`).

`repos::validate_write_response` (`repos.rs:173-193`) implements the same ladder with
the same two message forms, then *does* return
`normalize_write_response(raw)` (`repos.rs:192`) so `repos protect set/remove` emit
the canonical `{event_id, accepted, message}` (`repos.rs:195-199`).

Net: one rule, two implementations, two different output contracts, and the pair is
covered by tests only on the `repos` side (`duplicate_write_response_is_a_conflict`
`repos.rs:619`, `successful_write_response_is_normalized` `repos.rs:629`). Extracting
the ladder into `client.rs` next to `normalize_write_response` would fix the
duplication and let `mem` adopt the standard write shape at the same time. (The
*semantics* of what the relay's `duplicate:` means, including the false-conflict
retry interaction, are covered in Business Rules § NIP-33 LWW.)

#### Lead: drift-guard coverage for `mem` and `moderation`

Three guards in `lib.rs`'s test module:

| Guard | Site | Covers `mem`? | Covers `moderation`? |
|---|---|---|---|
| `command_inventory_is_stable` | `lib.rs:1807-1851` | yes — both appear in the 21-name group list (`lib.rs:1817`, `:1819`) | yes |
| `subcommand_names_are_stable` | `lib.rs:1855-1993` | **no** — `grep 'names(&cmd, "' ` over its body yields 19 group assertions; `mem` and `notes` are absent | yes (`lib.rs:1979-1990`, all 8 names) |
| `subcommand_counts_are_stable` | `lib.rs:1996-2033` | **no** | **no** |

`subcommand_counts_are_stable`'s `expected` list (`lib.rs:1998-2015`) enumerates 18
entries — `agents, canvas, channels, dms, emoji, feed, issues, media, messages, pack,
patches, pr, reactions, repos, social, upload, users, workflows` — omitting `mem`,
`moderation` and `notes`. So the reported "18 of 21" is confirmed, and the omissions
are exactly those three.

Consequence for this group: adding or removing a `mem` subcommand trips **no** guard
beyond the group-name list, because `mem` is missing from both the names guard and the
counts guard. `moderation` is half-covered — the names guard would catch a change, the
counts guard would not, so the two guards disagree about which groups are load-bearing.
And because the counts list is a hand-maintained literal with no cross-check against
`command_inventory_is_stable`'s 21-name list, a new group added tomorrow inherits the
same blind spot silently.

#### Lead: output-contract drift vs `AGENTS.md § Agent CLI`

The stated contract: "All reads return sig-stripped JSON arrays; all writes return
`{event_id, accepted, message}`; creates add the entity ID."

**Reads that are not sig-stripped.** Six commands print the relay's `POST /query`
body verbatim:

| Command | Site |
|---|---|
| `repos get` | `repos.rs:251-253` |
| `repos list` | `repos.rs:279-281` |
| `patches get` | `patches.rs:79-80` |
| `patches list` | `patches.rs:109-110` |
| `pr get` | `pr.rs:113-114` |
| `pr list` | `pr.rs:145-146` |
| `issues get` | `issues.rs:42-43` |
| `issues list` | `issues.rs:75-76` |

The relay serializes the whole event with `serde_json::to_value(&se.event)`
(`crates/buzz-relay/src/api/bridge.rs:1126`), and there is no stripping in that
handler (`grep -c '"sig"' crates/buzz-relay/src/api/bridge.rs` → `0`), nor in any of
the eleven files (`grep -c '"sig"'` → `0` for each). Since a NIP-01 event includes
`sig` by definition, these eight read paths emit `sig`. I did not read the `nostr`
crate's `Serialize` impl, so I am inferring the field's presence from NIP-01 rather
than observing it; the *absence of stripping* on both sides is directly verified.

Commands that comply by accident — they build their own projection, which drops
`sig`: `users get` (`users.rs:50-61`), `users presence` (`users.rs:264-273`),
`workflows list/get/runs` (`workflows.rs:21-31`, `:46-55`, `:80-91`),
`repos protect list` (`repos.rs:147-166`), `mem ls --json` (`mem.rs:263`),
`agents archived` (`agents.rs:312`).

**Reads that are not arrays.**

| Command | Emits | Site |
|---|---|---|
| `workflows get` | a bare object, or the literal `null` when absent | `workflows.rs:46-58` |
| `repos protect list` | a single object | `repos.rs:296-297` |
| `agents archived` | `{"archived": [...]}` — array wrapped in an object | `agents.rs:312` |
| `mem get` | raw value bytes, no newline, not JSON | `mem.rs:293-303` |
| `mem hash` | a bare hex line | `mem.rs:518` |
| `mem ls` (no `--json`) | TSV to stdout, `(no memories besides core)` to stderr | `mem.rs:263-271` |
| `pack validate` / `pack inspect` | human-readable text only | `pack.rs:24-46`, `pack.rs:65-149` |

**Writes that are not `{event_id, accepted, message}`.**

| Command | Actual shape | Site |
|---|---|---|
| `agents archive` | `{ok, event_id, action, target}` — adds `ok`, drops `accepted`/`message` | `agents.rs:111-119` |
| `agents unarchive` | same four-key shape | `agents.rs:140-148` |
| `agents draft-create` / `draft-update` | relay object **plus** `request_id`, `action`, `saved`, `message` | `agents.rs:34-51`, `agents.rs:70-87` |
| `mem set` / `mem patch` / `mem rm` | **nothing on stdout**; a prose line on stderr | `mem.rs:370`, `mem.rs:696`, `mem.rs:733` |
| `repos create` | raw relay body, unnormalized | `repos.rs:227-228` |
| `patches send`, `patches status`, `pr open`, `pr update`, `pr status`, `issues create`, `issues status` | raw relay body (happens to match, but is passthrough not normalization) | `patches.rs:56-57`, `patches.rs:183-185`, `pr.rs:59-60`, `pr.rs:105-106`, `pr.rs:216-218`, `issues.rs:32-33`, `issues.rs:140-142` |
| `upload file` | pretty-printed `BlobDescriptor`, the group's only multi-line output | `upload.rs:8-11` |
| `media get` | raw bytes to stdout or a file | `upload.rs:22-36` |

Compliant writes: `users set-profile` (`users.rs:212`), `users set-presence`
(`users.rs:303`), `workflows update/delete/trigger/approve` (`workflows.rs:134`,
`:148`, `:182`, `:187`, `:210`), all six `moderation` mutations
(`moderation.rs:47`, `:57`, `:75`, `:85`, `:101`), `repos protect set/remove`
(`repos.rs:198`).

**Creates that add the entity ID:** only `workflows create`
(`print_create_response(&resp, "workflow_id", …)`, `workflows.rs:114`). `repos create`
and `issues create` do not — `repos create` prints the raw body with no `repo_id`
(`repos.rs:227-228`) even though the caller supplied the `d`-tag, and `issues create`
returns no issue identifier beyond whatever the relay echoes (`issues.rs:32-33`).

Summary: of the ~54 leaf commands in this group, at least 8 reads leak `sig`, 7 reads are
not arrays, 12 writes deviate from the write shape, and 2 of 3 creates omit the entity
ID. The contract as written in `AGENTS.md § Agent CLI` describes roughly half of this
group's actual behaviour. Whether the fix is code or documentation is a product call I
can't make from the source; what is unambiguous is that agents told to rely on the
documented shapes will mis-parse `mem`, `pack`, `agents` and the eight NIP-34 read
commands.

#### Documentation drift, `crates/buzz-cli/README.md`

- **Six in-scope command groups are entirely missing from the command table**
  (`README.md:81-160`): `agents`, `patches`, `pr`, `issues`, `media`, `moderation`.
  Also missing crate-wide: `emoji`, `notes`. The table documents 13 of 21 groups.
- **`BUZZ_AUTH_TAG` is absent from the auth table** (`README.md:13-15`), which lists
  only `BUZZ_PRIVATE_KEY` — yet `agents draft-create`/`draft-update` hard-require it
  (`agents.rs:156-162`) and `mem` needs it or `--owner` (`mem.rs:37-42`).
- `README.md:3` promises "JSON in, JSON out" and `README.md:25` "All output is JSON on
  stdout"; see the output-contract table above for the seven commands where that is
  false.
- `README.md:1` names `main.rs` as the clap entry point in the architecture diagram
  (`README.md:164-168`). Accurate — `crates/buzz-cli/Cargo.toml` declares
  `[[bin]] path = "src/main.rs"` — but the `Cli` derive and dispatch actually live in
  `lib.rs:63-99` / `lib.rs:1733-1792`, so the diagram under-describes the split.
- `crates/buzz-cli/Cargo.toml:19`'s clap dependency comment reads
  `"derive macros + env var support (BUZZ_API_TOKEN auto-wired)"`. Nothing in this
  crate reads that variable — `grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/` matches
  only the comment itself, and the crate's three `env =` attributes are
  `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG` (`lib.rs:81`, `:85`, `:89`).
  The variable is still live in `buzz-acp` (`crates/buzz-acp/src/config.rs:718`),
  `buzz-workflow` (`crates/buzz-workflow/src/executor.rs:901`) and the desktop agent
  runtime (`desktop/src-tauri/src/managed_agents/env_vars.rs:63`), so the comment is
  stale for `buzz-cli` specifically, not repo-wide.
- `.env.example` documents `BUZZ_RELAY_URL` as `ws://localhost:3000`
  (`.env.example:130`) against the CLI's `http://localhost:3000` default
  (`lib.rs:81`), and never mentions `BUZZ_AUTH_TAG`, `BUZZ_TIMEOUT_SECS` or
  `BUZZ_CONNECT_TIMEOUT_SECS` — see Configuration for the full matrix.

#### Smaller items

- `agents.rs:180` queries kind:0 with `limit: 1` and takes the first event
  (`agents.rs:184-187`) with no `created_at` ordering, unlike `repos.rs:28-30` which
  explicitly sorts by `Reverse(created_at)` before taking one. Whether the relay
  orders `POST /query` results is not asserted at either call site; `repos.rs` defends
  itself, `agents.rs` does not.
- `patches send` accepts `--root` and `--root-revision` simultaneously with no check
  in either clap (`lib.rs:1218`, `:1221` — no `conflicts_with`) or `patches.rs:36-37`.
  Contrast `mem patch`, which enforces its flag pair by hand
  (`mem.rs:551-567`), and `pr open`, which uses `conflicts_with` (`lib.rs:1315-1319`).
  Three different strategies for the same class of problem inside one crate.
- `pr::read_optional_body`'s `(Some, Some)` arm (`pr.rs:11-13`) is dead through the
  CLI because clap already declares the conflict (`lib.rs:1315-1319`,
  `lib.rs:1369-1373`, `lib.rs:1417-1421`); its test (`pr.rs:333-336`) calls the
  function directly, so it passes without proving anything about CLI behaviour.
- `workflows.rs:60-66`'s note that `workflows runs` "will return an empty array"
  documents a command that ships knowingly non-functional, with no `#[test]` and no
  tracking marker beyond the prose.
- `moderation::dispatch` takes `_format` (`moderation.rs:133-137`) — an
  accepted-and-discarded parameter that reads as an unfinished feature; see
  Configuration § `--format` matrix.
