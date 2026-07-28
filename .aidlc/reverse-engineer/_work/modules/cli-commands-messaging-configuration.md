## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Configuration
These ten files are almost entirely *flag*-configured. They read exactly **one**
environment variable directly (`channels.rs:1021`); everything else arrives as a
clap argument, a compile-time constant, or one local JSON file. Proof of the
absence: `grep -nE 'env::|env!|var_os|vars\(\)|option_env' channels.rs notes.rs
messages.rs emoji.rs social.rs channel_templates.rs reactions.rs dms.rs feed.rs
mod.rs` returns three lines, all in `channels.rs` — the production read at
`:1021` and `set_var`/`remove_var` inside the test module at `:1363`/`:1366`.
`grep -nE 'dirs::|home_dir|XDG|std::env::current_dir'` over the same ten files
returns one production line, `channel_templates.rs:73`.
#### Environment variables read inside the ten files
| Var | Read site | Default when unset | Parse rules | `.env.example` | `AGENTS.md` | `buzz-cli/README.md` |
|---|---|---|---|---|---|---|
| `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` | `channels.rs:1021` (`cmd_set_add_policy`) | absent ⇒ **no restriction**; every policy allowed | `split(',')` → `str::trim` → drop empty segments (`channels.rs:1022-1026`). An empty/whitespace-only value yields an empty list, which is also treated as "no restriction" (`channels.rs:1027`). No case folding, no validation that the listed names are real policies. | **No** | **No** | **No** |
Verifying the lead: the site, default, and parse rules above are all as
described. On the documentation half, `grep -rn 'BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES'`
across the repo returns only `channels.rs` (lines 1021, 1030, 1294, 1305, 1349,
1363, 1366) plus this reverse-engineering corpus. `.env.example` never mentions
it (its BUZZ_ACP block is around `.env.example:121-130`, listing only
`BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL`), and neither does `AGENTS.md` or
`crates/buzz-cli/README.md`. It is a **completely undocumented deployment
control** whose absence silently means "permit everything", and
`channels.rs:1014-1019` documents in-code that it is a client-side courtesy
gate only — a direct kind:10100 submission bypasses it.
#### Environment variables that reach these files indirectly
Set on `Cli` in `lib.rs` and threaded in as an already-constructed `BuzzClient`,
so none of the ten files parse them:
| Var | Declaration | Default | Consumed in scope via |
|---|---|---|---|
| `BUZZ_RELAY_URL` | `lib.rs:79` (`#[arg(long, env = …, default_value = "http://localhost:3000")]`) | `http://localhost:3000` | every `client.query` / `client.submit_event` call |
| `BUZZ_PRIVATE_KEY` | `lib.rs:82` | none — hard error at `lib.rs:1744-1746` | `client.keys().public_key()` (`channels.rs:33`, `messages.rs`, `dms.rs:9`, `feed.rs:15`, `emoji.rs:101`, `notes.rs:169`, `reactions.rs:43`) |
| `BUZZ_AUTH_TAG` | `lib.rs:85` | unset | `client.auth_tag_owner_hex()` — the template owner invariant at `channels.rs:646-648` |
Defaults-vs-docs disagreement: `AGENTS.md § Agent CLI` and `.env.example:121`
both show `BUZZ_RELAY_URL=ws://localhost:3000`, but the CLI's own default and
its `long_about` (`lib.rs:70`) are `http://localhost:3000`. The `ws://` form is
the relay/harness convention; the CLI speaks HTTP to `POST /query` and
`POST /events`. A reader following `AGENTS.md` verbatim gets a scheme the CLI
then has to normalize (`client::normalize_relay_url`, `lib.rs:1733`). Not a bug,
but the docs and the code disagree on the literal default string.
#### `--format`: declared non-global, honored by 4 of 26 read commands
`--format` is declared on the top-level `Cli` struct at `lib.rs:92-94` with
`#[arg(long, value_enum, default_value = "json")]` — **no `global = true`**.
Confirmed empirically against the prebuilt `target/debug/buzz`:
`buzz channels list --format compact` → `{"error":"user_error","message":"error:
unexpected argument '--format' found …"}`, while `buzz --format compact channels
list` parses and fails later on the missing key. So `AGENTS.md`'s instruction
("`--format compact` is a **global** flag — it goes before the subcommand") is
right about *position* and loose about *mechanism*: it is a top-level-only
argument, not a clap global.
Forwarding, from `lib.rs:1771-1791`:
| Dispatcher | Receives `&cli.format`? | Site |
|---|---|---|
| `messages::dispatch` | yes | `lib.rs:1772` |
| `channels::dispatch` | yes | `lib.rs:1773` |
| `feed::dispatch` | yes | `lib.rs:1780` |
| `channels::dispatch_canvas` | **no** | `lib.rs:1774` |
| `reactions::dispatch` | **no** | `lib.rs:1775` |
| `emoji::dispatch` | **no** | `lib.rs:1776` |
| `dms::dispatch` | **no** | `lib.rs:1777` |
| `social::dispatch` | **no** | `lib.rs:1781` |
| `notes::dispatch` | **no** | `lib.rs:1782` |
Within the three dispatchers that do receive it, forwarding is again partial:
| Command | Honors `--format compact`? | Evidence |
|---|---|---|
| `messages get` | yes | `messages.rs:300` → `format_events` (`messages.rs:242-262`) |
| `messages thread` | yes | `messages.rs:336` |
| `messages search` | yes | `messages.rs:383` |
| `feed get` | yes | `feed.rs:45-65` (inline projection) |
| `channels list` | yes | `channels.rs:95-109` — but a *different* compact shape (`channel_id`, `name`) |
| `channels get` | **no** | `cmd_get_channel` takes no format param (`channels.rs:224`) |
| `channels search` | **no** | `cmd_search_channels` takes no format param (`channels.rs:119-125`); `dispatch` drops it at `channels.rs:1083-1088` |
| `channels members` | **no** | `channels.rs:244-247` |
| `canvas get` | **no** | `channels.rs:262` |
| `reactions get`, `emoji list`/`export`, `dms list`, `social event`/`notes`/`contacts`/`list`, `notes get`/`ls` | **no** | none of these functions has an `OutputFormat` parameter |
Net effect: `buzz --format compact <anything except messages get/thread/search,
feed get, channels list>` is accepted, exits 0, and silently emits full JSON.
There is no warning and no test — `grep -n 'format_events' messages.rs` returns
only the definition (`:242`) and three production call sites (`:300`, `:336`,
`:383`), all above the `#[cfg(test)]` boundary at `messages.rs:877`.
#### Local filesystem configuration
Only `channels create --template` touches the filesystem for config.
| Item | Value | Site |
|---|---|---|
| Override flag | `--templates-file <PATH>`, taken verbatim, no existence check at resolve time | `lib.rs:568-571`, `channel_templates.rs:70-72` |
| Default path | `dirs::data_dir()` / `xyz.block.buzz.app` / `templates` / `channel-templates.json` | `channel_templates.rs:73-79` |
| Bundle identifier constant | `PROD_BUNDLE_IDENTIFIER = "xyz.block.buzz.app"` | `channel_templates.rs:17` |
| Missing-file behavior | `CliError::NotFound` naming the path and suggesting `--templates-file` | `channel_templates.rs:83-89` |
| Parse failure | `CliError::Other` (exit 4), not `Usage` | `channel_templates.rs:94-95` |
| Template lookup | exact match after `to_ascii_lowercase()` on both sides | `channel_templates.rs:101-110` |
| Store re-read | `load_templates` runs **twice** on the not-found path — once in `find_template`, once in `available_names` | `channel_templates.rs:102`, `channel_templates.rs:114` |
Configuration gap worth flagging: the constant covers only the **production**
bundle id. `desktop/src-tauri/tauri.dev.conf.json:2` sets
`xyz.block.buzz.app.dev`, so templates authored in a dev desktop build are
invisible to `buzz channels create --template` unless `--templates-file` is
passed. The doc comment at `channel_templates.rs:29-31` acknowledges the
override is "useful for the dev store or tests" but the dev path itself is
nowhere derived, and neither `AGENTS.md` nor `crates/buzz-cli/README.md`
mentions channel templates at all (`grep -n 'templates-file\|channel-templates'
crates/buzz-cli/README.md` → zero matches).
Consumed fields of `channel-templates.json`, with defaults supplied by serde:
| Field | Default | Site |
|---|---|---|
| `name` | required | `channel_templates.rs:21` |
| `description` | `None` | `channel_templates.rs:22-23` |
| `channel_type` | `"stream"` via `default_channel_type` | `channel_templates.rs:24-25`, `:57-59` |
| `visibility` | `"open"` via `default_visibility` | `channel_templates.rs:26-27`, `:61-63` |
| `canvas_template` | `None` | `channel_templates.rs:28-29` |
| `agents.personas[].personaId` | `[]` | `channel_templates.rs:30-31`, `:38-41` |
| `agents.teams[].teamId` | `[]` | `channel_templates.rs:43-47` |
The CLI's record is a deliberate narrowing of
`desktop/src-tauri/src/templates/types.rs:5-21`, documented at
`channel_templates.rs:1-8`. Dropped fields: `id`, `is_builtin`, `created_at`,
`updated_at`, and per-entry `runtime` / `model` / `role` / `backend`
(`types.rs:35-43`, `:49-55`). The dropped `role` is harmless in practice —
`channels.rs:748-751` hardcodes `MemberRole::Bot`, and desktop's
`useApplyTemplate.ts:101` and `:126` also hardcode `role: "bot"`, so both
surfaces ignore the field identically. Placeholder substitution in
`canvas_template` supports exactly two tokens, `{channel.name}` and
`{template.name}` (`channels.rs:726-728`); any other `{…}` is emitted literally.
#### CLI flags with defaults, clamps, or silently-ignored values
| Flag | Declared default | Effective default / clamp | Sites |
|---|---|---|---|
| `channels list --limit` | none (`Option<u32>`), doc comment says `[default: 500]` | `unwrap_or(500)`, no cap | `lib.rs:514-516`, `channels.rs:32` |
| `channels search --limit` | `default_value_t = 1000` | passed straight to `query_paginated`, **no clamp** | `lib.rs:538-539`, `channels.rs:131` |
| `channels create --ttl` | none | must be `> 0` and fit `i32` | `lib.rs:559-561`, `channels.rs:820-830` |
| `messages get --limit` | none | `unwrap_or(50).min(200)` | `lib.rs:443-445`, `messages.rs:271` |
| `messages get --kinds` | none | defaults to `[9, 40002, 40008, 45001, 45003]`; override only if ≥1 segment parses | `lib.rs:451-453`, `messages.rs:274-285` |
| `messages thread --limit` | none | `unwrap_or(100).min(500)` | `messages.rs:312` |
| `messages thread --depth-limit` | none | passed through as the relay extension field `depth_limit` | `messages.rs:322-324` |
| `messages search --limit` | none | `unwrap_or(20).min(100)` | `messages.rs:352` |
| `feed get --limit` | none | `unwrap_or(20).min(50)` | `feed.rs:17` |
| `feed get --types` | none | validated against `VALID_FEED_TYPES`; sent as relay extension `feed_types` | `feed.rs:6`, `:29-40` |
| `dms list --limit` | none | `unwrap_or(50).min(200)` | `dms.rs:10` |
| `social notes --limit` | none | `unwrap_or(50).min(100)` | `social.rs:94` |
| `social list --limit` | not exposed | hardcoded `10` | `social.rs:201` |
| `notes ls --limit` | none | `unwrap_or(50).min(200)` | `notes.rs:677` |
| `notes ls --author` | `default_value = "me"` (clap) | `unwrap_or("me")` again in the handler — unreachable | `lib.rs:1069-1071`, `notes.rs:678` |
| `notes get --name` fan-out | not exposed | hardcoded `limit: 50` cross-author | `notes.rs:191` |
| `emoji export --scope` | `default_value = "own"` | `own` \| `workspace` | `lib.rs:757-759` |
| `emoji import --replace` / `--dry-run` | `default_value_t = false` | merge, publish | `lib.rs:765-770` |
| `notes set --clear-tags` / `--allow-empty` | `default_value_t = false` | carry-forward, refuse-empty | `lib.rs:1035-1049` |
Two silent-ignore behaviors in this table are worth calling out explicitly:
- `messages get --kinds abc` → `filter_map(…parse().ok())` yields an empty list,
  the `if !kind_list.is_empty()` guard at `messages.rs:282` fails, and the
  request goes out with the **default** kinds. Exit 0, no warning. Mixed input
  like `--kinds 9,abc` silently degrades to `[9]`.
- `notes ls --author` can never be `None` at the handler (clap always supplies
  `"me"`), so the `unwrap_or("me")` at `notes.rs:678` is dead and the flag's
  `Option<String>` type is misleading.
#### Compile-time constants behaving like configuration
| Constant | Value | Site | What it configures |
|---|---|---|---|
| `KIND_LONG_FORM` | `30023` | `notes.rs:38` | every `notes` filter and the `set`/`rm` builders. **Redeclared** — `buzz_core::kind::KIND_LONG_FORM` already exists at `kind.rs:66` |
| `SLUG_MAX_LEN` | `80` | `notes.rs:42` | `parse_slug` upper bound (`notes.rs:55-60`) |
| `SET_STDIN_MAX_BYTES` | `1_048_576` (1 MiB) | `notes.rs:485` | `notes set --content -` stdin cap (`notes.rs:502-508`) |
| `STDIN_MAX_BYTES` | `10_000_000` | `emoji.rs:159` | `emoji import` stdin cap (`emoji.rs:171`) |
| `MAX_CONTENT_BYTES` | `65_536` | `validate.rs:4` | `messages send` / `messages edit` (`messages.rs:480`, `:705`) |
| `MAX_DIFF_BYTES` | `61_440` (60 KiB) | `validate.rs:7` | `messages send-diff` truncation at a hunk boundary (`messages.rs:606`) |
| `MENTION_CAP` | `50` | `buzz-sdk/src/mentions.rs:38` | mention-tag ceiling for `messages send` (`messages.rs:536`) |
| `CUSTOM_EMOJI_SET_D_TAG` | re-export of `buzz_sdk::CUSTOM_EMOJI_SET_D_TAG` | `emoji.rs:9` | `#d` on every kind:30030 filter (`emoji.rs:80`, `:103`, `:221`) |
| `VALID_FEED_TYPES` | `["mentions","needs_action","activity","agent_activity"]` | `feed.rs:6` | `feed get --types` allowlist |
| `PROD_BUNDLE_IDENTIFIER` | `"xyz.block.buzz.app"` | `channel_templates.rs:17` | template store path |
| `QUERY_PAGE_SIZE` | `500` | `client.rs:498` | page size for `query_paginated` / `query_all`, used by `channels.rs:41`, `:57`, `:62`, `:131`, `:452` |
Hardcoded kind-integer literals are the module's largest de-facto config
surface. `channels.rs` embeds `39002` (`:38`, `:250`), `39000` (`:54`, `:62`,
`:131`, `:227`) and `40100` (`:265`); `messages.rs` embeds `39002` (`:139`),
`0` (`:150`, `:407`) and the read sets at `:276`, `:320`, `:361`;
`reactions.rs` embeds `7` (`:45`, `:83`); `dms.rs` embeds `41001` (`:12`) and
`41010` (`:69`); `social.rs` embeds `1` (`:97`) and `3` (`:118`). This
contradicts `AGENTS.md § Common Gotchas #1` ("All kinds defined in
`buzz-core/src/kind.rs`") — the constants exist (`kind.rs:447-453` for the DM
kinds alone) and are used correctly by `dms.rs:104`, `emoji.rs:80`,
`channels.rs:3`, `social.rs:1-5`, so the module knows the pattern and applies
it inconsistently.
Three different stdin caps coexist for the same "read a body from `-`" job:
unbounded for `messages send` / `canvas set` (`validate.rs:168-181` has no cap;
`messages send` merely rejects the result afterwards at `messages.rs:480`, and
`canvas set` at `channels.rs:1050` never checks size at all), 1 MiB for
`notes set`, 10 MB for `emoji import`. None of the three is documented.
#### Stated uncertainties
- `channels search --limit 1000` (`lib.rs:539`) is a *fetch* budget, not a
  result count — the name-matching happens client-side at `channels.rs:134-138`.
  Whether the relay itself clamps that limit on `POST /query` is out of scope
  here; I did not trace it.
- `messages thread --depth-limit` and `social notes --before-id` are relay
  *extension* fields injected into the filter JSON (`messages.rs:323`,
  `social.rs:106`). `crates/buzz-relay/src/api/bridge.rs:283` and `:263` do read
  them, so they are real, but I did not verify their end-to-end semantics.
- The `--format` empirical checks above ran against `target/debug/buzz` built
  2025-07-26, which predates the module docs in this corpus. Both facts also
  follow directly from the source I read (`lib.rs:92-94` has no `global`;
  `lib.rs:476-489` declares no `kinds` on `messages search`), so I am confident
  in them, but they are not from a fresh build.
