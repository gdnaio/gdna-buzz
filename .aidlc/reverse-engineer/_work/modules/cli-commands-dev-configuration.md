## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Configuration

Scope: `mem.rs`, `agents.rs`, `repos.rs`, `users.rs`, `pr.rs`, `patches.rs`,
`workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`. `lib.rs`,
`client.rs` and `validate.rs` are cited only where a value these files consume is
declared or read there.

#### Environment variables

None of the eleven files calls `std::env::var` directly:
`grep -n 'env::var' ` across all eleven returns zero matches. Every env var reaches
them either through the clap `env =` attribute on the root `Cli` struct or through
`BuzzClient`.

| Var | Default | Read site | Reaches these files via | `.env.example` | `AGENTS.md` | `buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_RELAY_URL` | `http://localhost:3000` (`lib.rs:81`) | clap `env =` (`lib.rs:81`), normalized by `client::normalize_relay_url` (`client.rs:1291-1296`) at `lib.rs:1734` | `client.relay_url()` | yes but **as `ws://localhost:3000`** (`.env.example:130`), and only inside the ACP-harness section | yes (`AGENTS.md:164`, `:166`) | yes (`README.md:29`) |
| `BUZZ_PRIVATE_KEY` | none — required (`lib.rs:85`, enforced `lib.rs:1746-1748`) | clap `env =` (`lib.rs:85`) | `client.keys()` — used in every file except `pack.rs`/`upload.rs` | yes, ACP section only (`.env.example:125`) | yes (`AGENTS.md:164`, `:166`) | yes (`README.md:15`, `:19`) |
| `BUZZ_AUTH_TAG` | none — optional (`lib.rs:89`) | clap `env =` (`lib.rs:89`); parsed + signature-verified at `lib.rs:1755-1765` | `client.auth_tag_owner_hex()` at `mem.rs:38`, `agents.rs:159` | **no** (`grep -c 'BUZZ_AUTH_TAG' .env.example` → `0`) | yes (`AGENTS.md:164`) | **no** — the auth table lists only `BUZZ_PRIVATE_KEY` (`README.md:13-15`) |
| `BUZZ_TIMEOUT_SECS` | `30` s (`client.rs:548`) | `env_duration_secs` (`client.rs:140-146`) | every relay call | **no** | **no** | **no** |
| `BUZZ_CONNECT_TIMEOUT_SECS` | `15` s (`client.rs:549`) | `env_duration_secs` (`client.rs:140-146`) | every relay call | **no** | **no** | **no** |

`grep -rn 'BUZZ_TIMEOUT_SECS\|BUZZ_CONNECT_TIMEOUT_SECS' .env.example AGENTS.md crates/buzz-cli/README.md CONTRIBUTING.md`
returns zero matches — both timeout knobs are documented only in the `BuzzClient::new`
doc comment (`client.rs:537-538`).

Two silent-ignore behaviours in that read path: `env_duration_secs` discards a value
that fails `parse::<u64>()` **and** discards `0`, falling back to the default rather
than erroring (`client.rs:140-146`; the doc comment at `client.rs:139` states `0` is
treated as invalid deliberately, to prevent disabling timeouts). So
`BUZZ_TIMEOUT_SECS=abc` and `BUZZ_TIMEOUT_SECS=0` both silently mean 30 s. All four
branches (valid / non-numeric / zero / unset) are pinned by the single test
`env_duration_secs_parsing` (`client.rs:1554-1569`).

`BUZZ_AUTH_TAG` is also read a second time, out of band, purely to improve a 403
message (`client.rs:991`, `client.rs:1271`) — `std::env::var("BUZZ_AUTH_TAG").is_ok()`.
That path bypasses `--auth-tag`, so a caller who passes the tag as a flag rather than an
env var gets the less helpful error text.

Documentation contradiction worth flagging: `.env.example:130` shows
`BUZZ_RELAY_URL=ws://localhost:3000` while the CLI flag default is
`http://localhost:3000` (`lib.rs:81`). Functionally both work —
`normalize_relay_url` rewrites `ws://`→`http://` and `wss://`→`https://`
(`client.rs:1292-1293`) — but the documented value never matches the documented
default, and `.env.example`'s comment scopes the var to the ACP harness
(`.env.example:113-121`), not to `buzz`.

#### Global flags and the `--format` matrix

`--format` is declared on the root parser **without `global = true`**
(`lib.rs:92-93`, `default_value = "json"`). It is therefore accepted only before the
subcommand, which matches the usage `AGENTS.md § Agent CLI` documents
(`buzz --format compact channels list`). `lib.rs:1770-1791` forwards `&cli.format` to
only five dispatchers; the rest never receive it.

| In-scope dispatcher | Receives `OutputFormat`? | Honors it? | Site |
|---|---|---|---|
| `users::dispatch` | yes | **yes** — `Compact` projects `{pubkey, display_name}` in both `cmd_get_users` (`users.rs:63-74`) and `search_by_name` (`users.rs:134-145`) | `lib.rs:1778`, `users.rs:307-311` |
| `moderation::dispatch` | yes | **no** — bound as `_format` and never read (`moderation.rs:133-137`) | `lib.rs:1790` |
| `agents::dispatch` | no | n/a | `lib.rs:1771` |
| `workflows::dispatch` | no | n/a | `lib.rs:1779` |
| `repos::dispatch` | no | n/a | `lib.rs:1783` |
| `patches::dispatch` | no | n/a | `lib.rs:1784` |
| `issues::dispatch` | no | n/a | `lib.rs:1785` |
| `pr::dispatch` | no | n/a | `lib.rs:1786` |
| `upload::dispatch_media` / `dispatch` | no | n/a | `lib.rs:1787-1788` |
| `mem::dispatch` | no | n/a — has its own per-subcommand `--json` bool instead (`lib.rs:1550-1552`, honored at `mem.rs:193`, `mem.rs:262-270`) | `lib.rs:1789` |
| `pack` | no — dispatched before the client is built (`lib.rs:1737-1742`) | n/a | `lib.rs:1737` |

Net effect: `--format compact` is **parsed and silently ignored** for 10 of the 11
in-scope groups. `moderation` is the sharpest case because it accepts the argument
and discards it at the function boundary; the others never plumb it. `mem ls --json`
is a redundant second output-format control that the global flag does not reach.

#### Flag defaults, clamps, and values with no client-side validation

| Flag | Type / default | Site | Notes |
|---|---|---|---|
| `mem ls --json` | `bool`, `false` | `lib.rs:1551-1552` | see matrix above |
| `mem set --allow-empty` | `bool`, `false` | `lib.rs:1581-1582` | gates the zero-byte-stdin refusal (`mem.rs:339-346`); a literal `""` positional is always allowed (`mem.rs:348-349`) |
| `mem patch --no-base-hash` | `bool`, `false` | `lib.rs:1603-1604` | mutually exclusive with `--base-hash` and one of the two is mandatory (`mem.rs:551-567`) — enforced in code, **not** by a clap `conflicts_with` |
| `mem patch --base-hash` | `Option<String>` | `lib.rs:1599` | must be exactly 64 ASCII hex (`mem.rs:568-574`), compared case-insensitively (`mem.rs:607`) |
| `mem patch --dry-run` / `--allow-empty` | `bool`, `false` | `lib.rs:1606-1607`, `lib.rs:1609-1610` | |
| `mem ls/get/hash --owner` / `--agent` | `Option<String>` | `lib.rs:1546`, `lib.rs:1549` (plus `:1558`/`:1561`, `:1567`/`:1570`) | mutually exclusive for reads (`mem.rs:60-63`); `--agent` must differ from the CLI identity (`mem.rs:67-71`) |
| `repos protect set --no-force-push` / `--no-delete` / `--require-patch` | `bool`, `default_value_t = false` | `lib.rs:1161`, `:1164`, `:1167` | at least one rule (incl. `--push`) is required — otherwise `build_protection_tag` errors via `parse_protection_tag` (`repos.rs:79-81`; test `protection_set_requires_at_least_one_rule`, `repos.rs:521`) |
| `patches send --root` / `--root-revision` | `bool`, `default_value_t = false` | `lib.rs:1218`, `lib.rs:1221` | no `conflicts_with`, and no runtime check in `patches.rs`; both are handed straight to `GitPatchMeta` (`patches.rs:36-37`) |
| `workflows approve --approved` | `bool`, `default_value_t = true`, `ArgAction::Set` | `lib.rs:914` | **defaults to approve** — omitting the flag grants |
| `workflows runs --limit` | `Option<u32>` → `limit.unwrap_or(20).min(100)` | `workflows.rs:72` | the only client-side clamp in the group |
| `moderation reports --limit`, `moderation audit --limit` | `i64`, `default_value_t = 50` | `lib.rs:1654-1655`, `lib.rs:1728-1729` | no client-side validation; a negative value is interpolated verbatim into the query string (`moderation.rs:110`, `moderation.rs:127`) |
| `moderation ban/timeout --expires-in` | `Option<u64>`, `conflicts_with = "expires_at"` | `lib.rs:1684`, `lib.rs:1708` | `--expires-in` wins when both are somehow set (`moderation.rs:27-32`); `timeout` requires one of them (`moderation.rs:69-70`), `ban` does not (permanent) |
| `moderation reports --status`, `resolve --status/--action` | free-form `String` | `lib.rs:1651-1652`, `lib.rs:1670-1676` | no `value_parser`; the accepted vocabularies exist only in help text and are validated relay-side. `--status` is spliced into a URL query with no escaping (`moderation.rs:110-113`) — the crate's only percent-encoder is `#[cfg(test)]`-gated (`validate.rs:76-77`) and so unavailable to production code |
| `patches status --status`, `issues status --status` | `String` with `value_parser = [...]` | `lib.rs:1263`, `lib.rs:1495` | clap restricts the words, then `patches::parse_status` re-validates (`patches.rs:194-206`). The two clap lists differ (`merged` vs `resolved`) while `parse_status` accepts both as synonyms for kind 1631 |
| `agents archive/unarchive --content` | `String`, `default_value = ""` | `lib.rs:319`, `lib.rs:332` | |
| `repos list --limit`, `patches list --limit`, `pr list --limit`, `issues list --limit` | `Option<u32>` | `lib.rs:1133`, `:1255`, `:1406`, `:1487` | injected into the filter unvalidated (`repos.rs:275-277`, `patches.rs:105-107`, `pr.rs:141-143`, `issues.rs:71-73`); server-side clamping is covered in Business Rules |
| `users get --pubkey` | `Vec<String>`, max 200 | `lib.rs:806-807`, enforced `users.rs:32-34` | the only explicitly-capped repeated flag in the group |
| `pr open/update --body` / `--body-file` | `Option<String>`, `conflicts_with` each other | `lib.rs:1315-1319`, `lib.rs:1369-1373`, `lib.rs:1417-1421` | clap already rejects the pair, so the runtime check in `read_optional_body` (`pr.rs:11-13`) is unreachable in practice — its test calls the function directly (`pr.rs:334`) |
| `media get --output` | `Option<String>`; `-` or absent ⇒ stdout | `lib.rs:1535`, `upload.rs:22-36` | |
| `upload file --file` | `String`, required | `lib.rs:1523` | |
| `pack validate/inspect <path>` | positional `String`, required | `lib.rs:1628`, `lib.rs:1633` | |

`--relay`, `--private-key` and `--auth-tag` are the only other root flags
(`lib.rs:80-90`); all three override their env var per clap precedence, as the
`long_about` states (`lib.rs:68-71`).

#### Local filesystem inputs

| Command | Path source | Resolution | Failure mode |
|---|---|---|---|
| `mem patch --patch-file <path>` | flag (`lib.rs:1593`) | `std::fs::read_to_string(path)` — relative to CWD, no canonicalization (`mem.rs:581-582`) | `Usage` (exit 1) |
| `mem set <slug> -`, `mem patch` (no `--patch-file`) | stdin | `std::io::stdin().take(NIP44_PLAINTEXT_MAX + 1)` (`mem.rs:327-332`, `mem.rs:579`, `mem.rs:583-588`) | oversize ⇒ `Usage`; empty ⇒ `Usage` unless `--allow-empty` (set) / always for patch (`mem.rs:589-595`) |
| `patches send --patch-file <path>` | flag (`lib.rs:1207`) | `validate::read_file_or_stdin` → `std::fs::read_to_string` or stdin on `-` (`validate.rs:178-192`), called at `patches.rs:26` | `Usage` naming the path (`validate.rs:190`) |
| `pr open/update --body-file <path>` | flag | `read_file_or_stdin` via `read_optional_body` (`pr.rs:10-18`) | `Usage`; `--body` + `--body-file` together is also `Usage` (`pr.rs:11-13`) |
| `issues create --content -`, `patches status --content -`, `issues status --content -`, `agents draft-* --system-prompt -`, `workflows create/update --yaml -` | positional/flag value `-` | `validate::read_or_stdin` (`validate.rs:163-175`) — treats a non-`-` value as **literal content**, never a path (`issues.rs:19`, `patches.rs:132`, `issues.rs:95`, `agents.rs:29`, `workflows.rs:106`) | unbounded stdin read (no `.take()`, unlike `mem`) |
| `pack validate/inspect <path>` | positional (`lib.rs:1628`, `lib.rs:1633`) | `Path::new(path)`, existence + is-dir pre-checks (`pack.rs:16-22`, `pack.rs:53-59`), then `buzz_persona::validate::validate_pack` / `resolve::resolve_pack` | `Usage` for missing/non-dir; `Other` for resolve failure (`pack.rs:62-63`) |
| `upload file --file <path>` | flag (`lib.rs:1523`) | delegated to `client.upload_file` (`upload.rs:7`) | handled in `client.rs` |
| `media get --output <path>` | flag | `std::fs::write(path, &bytes)` (`upload.rs:26-27`) | `Other` (exit 4) |

Pack directory layout consumed by `pack.rs` (resolved inside `buzz-persona`, not
here): manifest at `<pack>/.plugin/plugin.json` (`crates/buzz-persona/src/validate.rs:211`,
`:303`), persona files listed in the manifest's `personas` array
(`crates/buzz-persona/src/validate.rs:113`), skills at `<pack>/skills/<name>/SKILL.md`
(`crates/buzz-persona/src/validate.rs:369`, `:392`).

Note the asymmetry: `mem` bounds its stdin reads, every other stdin path in the group
uses the unbounded `read_or_stdin`. And `read_or_stdin` vs `read_file_or_stdin` is a
live footgun — `validate.rs:172-176`'s doc comment and the regression test
`read_file_or_stdin_does_not_treat_path_as_literal_content` (`validate.rs:499`) exist
because `--patch-file` once used the literal-content variant.

#### Compile-time constants that behave like configuration

| Constant | Value | Declared | Consumed here |
|---|---|---|---|
| `engram::NIP44_PLAINTEXT_MAX` | `65_535` | `crates/buzz-core/src/engram.rs:28` | stdin/patch bounds and result-size caps (`mem.rs:322`, `:326-329`, `:562`, `:630-636`) |
| `engram::CORE_SLUG` | `"core"` | `crates/buzz-core/src/engram.rs:20` | `Body::Core` selection (`mem.rs:352-359`, `:640-648`) and the `mem rm core` refusal (`mem.rs:713-716`) |
| `engram::D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | `crates/buzz-core/src/engram.rs:24` | indirectly, via `engram::d_tag(&k_c, slug)` (`mem.rs:148`) — the `d` tag is `HMAC`-derived from the agent↔owner conversation key, not the slug, so it is opaque and per-pair |
| `engram::SLUG_MAX_LEN` | `255` | `crates/buzz-core/src/engram.rs:31` | via `normalize_slug` (`mem.rs:284`, `:322`, `:515`, `:549`, `:712`) |
| `mem ls` filter limit | `5000` literal | `mem.rs:201` | server-clamped — see Business Rules |
| `mem` head-fetch limit | `16` literal | `mem.rs:155` | |
| `users get --name` search limit | `100` literal | `users.rs:93` | |
| `users get --pubkey` cap | `200` literal | `users.rs:32-34` | |
| `workflows runs` default/cap | `20` / `100` literals | `workflows.rs:72` | |
| repo protection-rule ceiling | `50` per repo | enforced in `buzz_core::git_perms::parse_protection_tags`, surfaced at `repos.rs:120-125`; pinned by `protection_update_enforces_repository_rule_limit` (`repos.rs:551`) | |
| `validate::MAX_CONTENT_BYTES` / `MAX_DIFF_BYTES` | `65_536` / `61_440` | `validate.rs:5`, `validate.rs:8` | **not used by any of the eleven** — `grep -n 'validate_content_size\|MAX_CONTENT_BYTES\|MAX_DIFF_BYTES'` across all eleven returns zero matches. Content-size enforcement in this group is either `mem`'s NIP-44 cap or relay-side |
| `client.rs` retry/upload constants (`RETRY_MAX_ATTEMPTS = 3`, `RETRY_BASE_SECS`, `RETRY_IN_MAX_SECS = 30`, `MAX_IMAGE_BYTES = 50 MiB`, `MAX_VIDEO_BYTES = 500 MiB`, `QUERY_PAGE_SIZE = 500`, upload timeouts 600 s video / 120 s other) | `client.rs:122-134`, `:73`, `:76`, `:498`, `:1139-1142` | inherited by every relay call these files make; `upload file` is the only in-scope consumer of the media caps and upload timeouts (`upload.rs:7`) |

`d`-tag conventions written by this group: repo announcements use the raw `repo_id`
(`repos.rs:36-44` reads it back; `buzz_sdk::build_repo_announcement*` writes it),
workflow definitions use the workflow UUID (`workflows.rs:41-44`,
`workflows.rs:169-171`), and workflow **approvals** use
`hex(SHA256(approval_token))` rather than the raw UUID — an easily-missed convention
documented in a one-line comment and implemented at `workflows.rs:204-206`.
Engram `d` tags are the derived opaque value above.

#### Hardcoded kind integers vs `crates/buzz-core/src/kind.rs`

`AGENTS.md § Common Gotchas #1` states all event-kind integers are defined in
`crates/buzz-core/src/kind.rs`. Three of the eleven files import and use those
constants; the rest inline bare integers even though a matching constant exists.

| Site | Bare literal | Existing constant it should use |
|---|---|---|
| `repos.rs:240` (`cmd_get_repo`) | `30617` | `KIND_GIT_REPO_ANNOUNCEMENT` (`kind.rs:545`) — already imported at `repos.rs:1-4` and used at `repos.rs:21` |
| `repos.rs:271` (`cmd_list_repos`) | `30617` | same |
| `pr.rs:129`, `patches.rs:94`, `issues.rs:58` | `"30617:{owner}:{id}"` in the `#a` coordinate | same |
| `pr.rs:110`, `pr.rs:131` | `1618` | `KIND_GIT_PULL_REQUEST` (`kind.rs:551`) |
| `patches.rs:76`, `patches.rs:96` | `1617` | `KIND_GIT_PATCH` (`kind.rs:549`) |
| `issues.rs:39`, `issues.rs:60` | `1621` | `KIND_GIT_ISSUE` (`kind.rs:555`) |
| `workflows.rs:16`, `workflows.rs:41` | `30620` | `KIND_WORKFLOW_DEF` (`kind.rs:382`) — the same file *does* use `buzz_sdk::kind::KIND_WORKFLOW_TRIGGER` at `workflows.rs:176` |
| `workflows.rs:74` | `46001, 46002, 46003` | `KIND_WORKFLOW_TRIGGERED` / `KIND_WORKFLOW_STEP_STARTED` / `KIND_WORKFLOW_STEP_COMPLETED` (`kind.rs:504-508`) |
| `users.rs:258` | `40902` | `KIND_PRESENCE_SNAPSHOT` (`kind.rs:443`) |
| `users.rs:42`, `users.rs:91`, `users.rs:223`, `agents.rs:180` | `0` | `KIND_PROFILE` (`kind.rs:9`) |

Compliant sites, for contrast: `mem.rs:24` imports `KIND_AGENT_ENGRAM`,
`agents.rs:1` imports `KIND_IA_ARCHIVED_LIST`, `repos.rs:3` imports
`KIND_GIT_REPO_ANNOUNCEMENT`. `repos.rs` and `workflows.rs` are internally
inconsistent — constant in one function, literal in the next.

`patches.rs`/`issues.rs` doc strings mention `kind:1630-1633` for statuses
(`lib.rs:1252`, `lib.rs:1489`) but no integer is written in those files; the
status kind is chosen inside `buzz_sdk::build_git_status` from the `GitStatus`
enum (`patches.rs:184`, `issues.rs:139`, `pr.rs:215`).

#### Configuration absent by design

- No config file. `grep -rn 'config\.toml\|\.buzzrc\|dirs::' ` across the eleven
  returns zero matches; `dirs` is a crate dependency
  (`crates/buzz-cli/Cargo.toml`, "locates the desktop app's channel-templates.json
  store for `channels create --template`") but is used only by `channels.rs`, out of
  scope here.
- No per-community or per-channel config: `moderation` derives its tenant purely
  from the relay host, documented at `moderation.rs:14-15` and `lib.rs:1636-1641`.
- `pack.rs` reads no environment or relay configuration at all — it is dispatched
  before `BuzzClient` construction (`lib.rs:1737-1742`), which is why `buzz pack`
  works with no `BUZZ_PRIVATE_KEY` (`lib.rs:1746-1748` would otherwise reject it).

#### Test coverage of configuration behaviour

Covered: `env_duration_secs` parse/zero/absent fallbacks
(`env_duration_secs_parsing`, `client.rs:1554`); the `--base-hash` digest convention
via `sha256_hex_*` vectors (`mem.rs:847`, `:855`, `:863`);
`--owner`/`--agent` resolution and both mutual-exclusion rules
(`resolve_reader_defaults_to_agent_identity` `mem.rs:793`,
`resolve_reader_agent_flag_uses_cli_identity_as_owner` `mem.rs:807`,
`resolve_reader_rejects_owner_with_agent_flag` `mem.rs:821`,
`resolve_reader_rejects_agent_flag_matching_cli_identity` `mem.rs:837`);
`parse_status`'s flag vocabulary (`patches.rs:304`, `:319`).

Not covered: `--format` propagation for any command
(`grep -n 'OutputFormat' ` in test modules across the eleven returns zero matches);
`resolve_expiry`'s `--expires-in`/`--expires-at` precedence (`moderation.rs:26-35`);
`workflows runs`'s `min(…, 100)` clamp (`workflows.rs:72`); the `--approved` default;
the `moderation` URL/query construction; every filesystem-path branch in `pack.rs`
(`grep -n '#\[test\]' crates/buzz-cli/src/commands/pack.rs` → zero matches, as for
`upload.rs`, `workflows.rs`, `issues.rs` and `moderation.rs`).

One caveat on the `mem` test client: `test_client` builds a `BuzzClient` with
`None` for both auth-tag slots (`mem.rs:787-789`), so the
`BUZZ_AUTH_TAG`-derived owner path (`mem.rs:38-41`) is never exercised — only the
explicit `--owner` and `--agent` branches are.
