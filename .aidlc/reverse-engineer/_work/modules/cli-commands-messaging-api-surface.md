## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: API Surface

Eight command groups live in these files: `messages`, `channels`, `canvas`,
`reactions`, `emoji`, `dms`, `feed`, `social`, `notes` (nine enums; `channels.rs`
serves both `channels` and `canvas`). Each group has a module-level `dispatch`
called from `lib.rs:1772-1782`. `mod.rs` is a bare re-export list of 20 command
modules (`mod.rs:1-20`) with no logic and no doc comment.

#### Dispatch entry points

| Group | Dispatcher | Called from | Receives `&OutputFormat`? |
|-------|-----------|-------------|---------------------------|
| `messages` | `messages.rs:754` | `lib.rs:1772` | yes |
| `channels` | `channels.rs:1066` | `lib.rs:1773` | yes |
| `canvas` | `channels.rs:1168` `dispatch_canvas` | `lib.rs:1774` | no |
| `reactions` | `reactions.rs:127` | `lib.rs:1775` | no |
| `emoji` | `emoji.rs:311` | `lib.rs:1776` | no |
| `dms` | `dms.rs:128` | `lib.rs:1777` | no |
| `feed` | `feed.rs:67` | `lib.rs:1780` | yes |
| `social` | `social.rs:211` | `lib.rs:1781` | no |
| `notes` | `notes.rs:752` | `lib.rs:1782` | no |

`--format` is a global flag (`lib.rs:93-94`, default `json`). Only three of the
nine dispatchers accept it, and inside those only the multi-event read paths
honour it: `messages get`/`thread`/`search` via `format_events`
(`messages.rs:242-261`), `channels list` via its own inline match
(`channels.rs:96-109`), and `feed get` via a third copy of the same logic
(`feed.rs:47-60`). `--format compact` is therefore silently ignored by
`channels search`, `channels get`, `channels members`, `canvas get`,
`reactions get`, `emoji list`/`export`, `dms list`, all `social` reads, and all
`notes` reads.

#### `messages` (8 subcommands)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `send` | `cmd_send_message` `messages.rs:483` | 0..N `upload_file` (`:509`), 1 `/query` per reply (`resolve_thread_ref` `:64`), 2 `/query` for mentions (`:145`,`:157`), 1 `POST /events` (`:576`) | `{event_id,accepted,message}` |
| `send-diff` | `cmd_send_diff_message` `messages.rs:596` | 1 `/query` if `--reply-to` (`:637`), 1 `POST /events` (`:664`) | same |
| `edit` | `cmd_edit_message` `messages.rs:701` | 1 `/query` (`resolve_channel_id` `:93`), 1 `POST /events` (`:718`) | same |
| `delete` | `cmd_delete_message` `messages.rs:669` | 1 `/query`, 1 `POST /events` (`:695`) | same |
| `get` | `cmd_get_messages` `messages.rs:263` | 1 `/query` (`:296`) | event array |
| `thread` | `cmd_get_thread` `messages.rs:304` | 1 `/query` with **two** ORed filters via `query_multi` (`:332`) | event array |
| `search` | `cmd_search` `messages.rs:340` | 0..1 `/query` for author resolution (`:411`), 1 `/query` (`:373`) | event array |
| `vote` | `cmd_vote_on_post` `messages.rs:724` | 1 `/query`, 1 `POST /events` (`:749`) | same |

Every write in this group is preceded by at least one read when it needs the
channel: `resolve_channel_id` (`messages.rs:89-115`) re-derives the `h` tag from
the target event rather than taking `--channel`, so `edit`, `delete` and `vote`
cost one extra round trip and fail with `CliError::Other` (exit 4, not 1) when
the event is missing (`messages.rs:98`) or has no `h` tag (`messages.rs:112-114`).

#### `channels` (16 subcommands) and `canvas` (2)

| Subcommand | Handler | Relay calls |
|-----------|---------|-------------|
| `list` | `cmd_list_channels` `channels.rs:25` | 1 paginated `/query` (kind 39000) or 2 when `--member` (`:41`,`:65`) |
| `get` | `cmd_get_channel` `channels.rs:224` | 1 `/query` |
| `search` | `cmd_search_channels` `channels.rs:119` | 1 paginated `/query` (`:133`) |
| `create` | `cmd_create_channel` `channels.rs:282` | 1 `POST /events` (`:326`) |
| `create --template` | `cmd_create_channel_from_template` `channels.rs:655` | 1 `/query` per team (`:410`), 1 `query_all` for 30177 (`:449`), 1 `GET /` + 1 `/query` for the archive snapshot (`agents.rs:273`,`:287`), then 1 `POST /events` for the channel (`:721`), 1 for canvas (`:732`), 1 per resolved agent (`:754`) |
| `update` | `cmd_update_channel` `channels.rs:832` | 1 `POST /events` |
| `topic` / `purpose` | `channels.rs:864` / `:880` | 1 `POST /events` each |
| `join` / `leave` | `channels.rs:896` / `:908` | 1 each |
| `archive` / `unarchive` / `delete` | `channels.rs:920` / `:932` / `:944` | 1 each |
| `members` | `cmd_list_channel_members` `channels.rs:244` | 1 `/query` |
| `add-member` / `remove-member` | `channels.rs:956` / `:987` | 1 each |
| `set-add-policy` | `cmd_set_add_policy` `channels.rs:1005` | 1 `POST /events` (kind 10100) |
| `canvas get` | `cmd_get_canvas` `channels.rs:262` | 1 `/query` |
| `canvas set` | `cmd_set_canvas` `channels.rs:1049` | 1 `POST /events` |

`channels create --template` is the only handler in the module with a multi-write
transaction shape, and it is explicitly non-atomic: roster resolution runs first
so an ambiguous roster aborts with zero writes (`channels.rs:659-661` doc,
`:715`), but the channel-create response is **discarded** (`channels.rs:721`
`client.submit_event(event).await?;`), so a relay `accepted:false` is invisible
and the printed report still claims `status:"ok"` plus a `channel_id`. Canvas and
per-member failures are collected, not fatal (`channels.rs:737`, `:763-767`).

#### `reactions` (3), `emoji` (5), `dms` (4), `feed` (1)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `reactions add` | `reactions.rs:9` | 1 `POST /events` | write response |
| `reactions remove` | `reactions.rs:34` | 1 `/query` (own kind 7 on the target `:49`), 1 `POST /events` (`:75`) | write response |
| `reactions get` | `reactions.rs:80` | 1 `/query` (`:86`) | `{"reactions":[…]}` |
| `emoji list` | `emoji.rs:77` | 1 `/query` (`:82`) | `{"emojis":[…]}` |
| `emoji set` | `emoji.rs:128` | 1 `/query` (read-own), 1 `POST /events` | write response |
| `emoji rm` | `emoji.rs:141` | 1 `/query`, then 1 `POST /events` **or** short-circuit print (`:148-153`) | write response, or `{"accepted":true,"message":"not present"}` |
| `emoji export` | `emoji.rs:197` | 1 `/query` (own or workspace) | `{"emojis":[…]}` to stdout or `--file` |
| `emoji import` | `emoji.rs:234` | 0..1 `/query` (merge mode `:290`), 1 `POST /events` unless `--dry-run` | write response, or dry-run set + stderr note |
| `dms list` | `dms.rs:8` | 1 `/query` | `[{dm_id,participants,created_at}]` |
| `dms open` | `dms.rs:51` | 1 `POST /events` | write response + `dm_id` |
| `dms add-member` | `dms.rs:112` | 1 `POST /events` | write response |
| `dms hide` | `dms.rs:96` | 1 `POST /events` | write response |
| `feed get` | `feed.rs:9` | 1 `/query` | event array |

#### `social` (7)

| Subcommand | Handler | Relay call | Output |
|-----------|---------|-----------|--------|
| `publish` | `cmd_publish_note` `social.rs:22` | `POST /events` (kind 1) | normalized write response |
| `set-contacts` | `cmd_set_contact_list` `social.rs:43` | `POST /events` (kind 3) | normalized write response |
| `event` | `cmd_get_event` `social.rs:72` | `/query` `{ids:[…]}` | **raw** relay JSON |
| `notes` | `cmd_get_user_notes` `social.rs:83` | `/query` kind 1 | **raw** relay JSON |
| `contacts` | `cmd_get_contact_list` `social.rs:115` | `/query` kind 3 | **raw** relay JSON |
| `set-list` | `cmd_set_list` `social.rs:162` | `POST /events` | **raw** relay JSON (not normalized — `social.rs:180`) |
| `list` | `cmd_get_list` `social.rs:184` | `/query` | **raw** relay JSON |

#### `notes` (4)

| Subcommand | Handler | Relay calls | Output |
|-----------|---------|-------------|--------|
| `set` | `cmd_set` `notes.rs:487` | 1 `/query` read-before-write (`:530`), 1 `POST /events` (`:545`) | 5 plain-text lines (`:571-580`) |
| `get` | `cmd_get` `notes.rs:612` | 1 `/query` by coordinate or slug; +1 when `--author` is a name (`:216`) | pretty JSON, or raw markdown with `--content-only` |
| `ls` | `cmd_ls` `notes.rs:671` | 0..1 `/query` for author resolution, 1 `/query` (`:697`) | pretty JSON array |
| `rm` | `cmd_rm` `notes.rs:717` | 1 `/query` read-before-write (`:721`), 1 `POST /events` (`:733`) | 2 plain-text lines (`:747-748`) |

`notes set` and `notes rm` are the only writes in the module that inspect the
relay's `accepted` flag and message (`notes.rs:546-567`, `notes.rs:734-745`).

#### Underlying transport

Everything goes through three `BuzzClient` methods, all NIP-98-signed with the
`x-auth-tag` header when configured (`client.rs:767`, `:773`, `:863`,
`:120-127`):

| Client method | HTTP | Used by |
|---------------|------|---------|
| `query` (`client.rs:767`) | `POST /query`, single filter | all reads except the four below |
| `query_multi` (`client.rs:773`) | `POST /query`, ORed filters | `messages thread` only (`messages.rs:332`) |
| `query_paginated` (`client.rs:715`) | repeated `POST /query` with `(until,before_id)` cursor | `channels list` (`:42`,`:65`), `channels search` (`:133`) |
| `query_all` (`client.rs:724`) | same, unbounded | template roster scan (`channels.rs:449`) |
| `submit_event` (`client.rs:863`) | `POST /events` | every write |
| `upload_file` (`client.rs:1100`) | Blossom upload | `messages send --file` (`messages.rs:510`) |

No file in scope opens a WebSocket, and none calls `count`, `get_authed` or
`get_public` directly (`channels.rs` reaches `get_public` transitively through
`agents::fetch_archived_snapshot`).

#### Exit-code classes produced

Mapping is `error.rs:96-111`. Which classes each group can produce:

| Class | Variant | Raised in scope by |
|-------|---------|-------------------|
| 1 | `Usage` | flag validation everywhere, e.g. `channels.rs:288`, `:296`, `:824`, `:844`, `:975`, `:1015`, `:1028`; `messages.rs:345`, `:547`, `:568`, `:603`, `:733`; `dms.rs:53`; `feed.rs:34`; `social.rs:135`, `:147`, `:170`, `:189`; `notes.rs:52`, `:58`, `:64`, `:258`, `:262`, `:435`, `:496`, `:501`, `:590`, `:648`, `:687`, `:800`; `emoji.rs:178`, `:249`, `:256`, `:262`, `:266` |
| 1 | `NotFound` | `channel_templates.rs:90`, `:113`; `channels.rs:415`; `notes.rs:618`, `:637`, `:643`, `:722` |
| 2 / 3 | `Relay`, `Network` | propagated from `client.rs`, never constructed here |
| 4 | `Other` | build/parse failures, e.g. `messages.rs:66`, `:98`, `:112`, `:543`, `:571`; `channels.rs:311`, `:412`; `reactions.rs:21`, `:52`, `:58`, `:63`; `emoji.rs:84`, `:120`, `:131`, `:167`, `:171` |
| 5 | `Conflict` | **only** `notes.rs:562` (NIP-33 LWW `duplicate:`) |
| 3 | `Auth`, `Key` | not produced in scope (all in `lib.rs:1745-1766`) |

Because only `notes` inspects `accepted`, a relay rejection on any other write
prints `{"event_id":"…","accepted":false,"message":"…"}` and exits **0**
(`client.rs:1425-1431` then `Ok(())` at e.g. `messages.rs:577`,
`channels.rs:860`, `emoji.rs:123`, `dms.rs:107`, `reactions.rs:30`). Scripts that
branch on exit status alone will read a rejected write as success.

#### Public items and doc discipline

`pub` items with no doc comment (undocumented public API, against the AGENTS.md
rule "New public API must have doc comments"):

| Item | Site |
|------|------|
| `cmd_list_channels` | `channels.rs:25` |
| `cmd_get_channel` | `channels.rs:224` |
| `cmd_list_channel_members` | `channels.rs:244` |
| `cmd_create_channel` | `channels.rs:282` |
| `cmd_update_channel` | `channels.rs:832` |
| `cmd_set_channel_topic` / `cmd_set_channel_purpose` | `channels.rs:864` / `:880` |
| `cmd_join_channel` / `cmd_leave_channel` / `cmd_archive_channel` / `cmd_unarchive_channel` / `cmd_delete_channel` | `channels.rs:896`,`:908`,`:920`,`:932`,`:944` |
| `cmd_add_channel_member` / `cmd_remove_channel_member` | `channels.rs:956` / `:987` |
| `cmd_set_canvas` | `channels.rs:1049` |
| `dispatch` (all nine) | `channels.rs:1066`, `:1168`; `messages.rs:754`; `notes.rs:752`; `emoji.rs:311`; `social.rs:211`; `dms.rs:128`; `reactions.rs:127`; `feed.rs:67` |
| `SendMessageParams` / `SendDiffParams` and all their fields | `messages.rs:474-482`, `:581-595` |
| `cmd_get_messages` / `cmd_get_thread` / `cmd_search` | `messages.rs:263`, `:304`, `:340` |
| `cmd_add_reaction` / `cmd_remove_reaction` / `cmd_get_reactions` | `reactions.rs:9`, `:34`, `:80` |
| `cmd_publish_note` / `cmd_set_contact_list` / `cmd_set_list` | `social.rs:22`, `:43`, `:162` |
| `ContactEntry` fields | `social.rs:16-20` |
| `ChannelTemplateRecord` and nested structs' fields | `channel_templates.rs:22-52` |

`notes.rs` is the outlier in the other direction: a 27-line module doc
(`notes.rs:1-31`) plus doc comments on every public function, including a
ratified-semantics block for `build_set_event` (`notes.rs:390-417`).

Also note that `cmd_get` (`notes.rs:612`) and `cmd_set`/`cmd_rm` are `pub` but
depend on validation performed by the dispatcher: `cmd_get` calls
`name.expect("dispatch enforces --name xor --naddr")` (`notes.rs:632`), so any
caller that bypasses `dispatch` panics instead of getting an error. The
mutual-exclusion rules themselves live in `validate_get_args`
(`notes.rs:588-610`), not in clap — `grep -n 'conflicts_with' lib.rs` returns no
match inside the `NotesCmd` block (`lib.rs:1016-1093`), so `buzz notes get
--help` does not advertise them.

#### Test coverage — API surface

Command inventory is pinned in `lib.rs` unit tests: `command_inventory_is_stable`
(`lib.rs:1805`) lists all 21 groups; `subcommand_names_are_stable`
(`lib.rs:1849`) asserts exact subcommand name sets for `messages`, `channels`,
`canvas`, `reactions`, `emoji`, `dms`, `feed`, `social`;
`subcommand_counts_are_stable` (`lib.rs:1978`) pins counts. `notes` appears in
the group list (`lib.rs:1816`) but has **no** name assertion and **no** count
entry — `grep -n '"notes"' lib.rs` shows it only in the group inventory and as
the `social notes` alias, so `NotesCmd`'s four subcommand names are the one
uncovered inventory in scope.

No handler in scope has an end-to-end test: there is no `crates/buzz-cli/tests/`
directory (`ls crates/buzz-cli` shows only `Cargo.toml`, `README.md`,
`TESTING.md`, `src`), and the single `#[tokio::test]` in scope
(`channels.rs:1362`) exercises `cmd_set_add_policy`'s env gate only, which
returns before any network call.
