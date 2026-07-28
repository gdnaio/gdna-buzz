## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Data Model

This group holds the CLI's *shape*: the clap command tree, the process-global
options struct, the relay client's session state, and the JSON envelopes that
every read/write funnels through. Files in scope: `lib.rs` (2,035 lines),
`client.rs` (2,477), `validate.rs` (506), `error.rs` (236),
`agent_management.rs` (277), `main.rs` (4).

#### Global options struct (`Cli`)

`struct Cli` is **private** (`lib.rs:79`) — only `run_from_args` can build one
(`lib.rs:23-60`). Five fields:

| Field | Flag | Env | Type | Default | Site |
|---|---|---|---|---|---|
| `relay` | `--relay` | `BUZZ_RELAY_URL` | `String` | `http://localhost:3000` | `lib.rs:81-82` |
| `private_key` | `--private-key` | `BUZZ_PRIVATE_KEY` | `Option<String>` | none (required for all non-`pack` paths) | `lib.rs:84-85` |
| `auth_tag` | `--auth-tag` | `BUZZ_AUTH_TAG` | `Option<String>` | none | `lib.rs:87-89` |
| `format` | `--format` | — | `OutputFormat` | `json` | `lib.rs:92-93` |
| `command` | positional subcommand | — | `Cmd` | required | `lib.rs:95-96` |

`OutputFormat` is a two-variant `ValueEnum` deriving `Default` with `Json` as
the default variant (`lib.rs:163-172`). None of the five args are declared
`global = true` — `grep -n 'global' lib.rs` returns exactly one hit, a doc
comment at `lib.rs:1640`, so no arg is promoted into subcommand scope.

#### The command tree as data

`enum Cmd` (`lib.rs:175-240`) has 21 variants, each `#[command(subcommand)]`
wrapping a per-group enum. The dispatch table (`lib.rs:1772-1792`) shows which
groups receive the `format` value and which discard it:

| Group | Variant enum | Dispatch target | Gets `&cli.format`? |
|---|---|---|---|
| `agents` | `AgentsCmd` (`lib.rs:260-345`) | `commands::agents::dispatch` (`lib.rs:1771`) | no |
| `messages` | `MessagesCmd` (`lib.rs:348-499`) | `commands::messages::dispatch` (`lib.rs:1772`) | yes |
| `channels` | `ChannelsCmd` (`lib.rs:502-676`) | `commands::channels::dispatch` (`lib.rs:1773`) | yes |
| `canvas` | `CanvasCmd` (`lib.rs:679-695`) | `commands::channels::dispatch_canvas` (`lib.rs:1774`) | no |
| `reactions` | `ReactionsCmd` (`lib.rs:698-726`) | `commands::reactions::dispatch` (`lib.rs:1775`) | no |
| `emoji` | `EmojiCmd` (`lib.rs:729-768`) | `commands::emoji::dispatch` (`lib.rs:1776`) | no |
| `dms` | `DmsCmd` (`lib.rs:771-799`) | `commands::dms::dispatch` (`lib.rs:1777`) | no |
| `users` | `UsersCmd` (`lib.rs:802-841`) | `commands::users::dispatch` (`lib.rs:1778`) | yes |
| `workflows` | `WorkflowsCmd` (`lib.rs:844-920`) | `commands::workflows::dispatch` (`lib.rs:1779`) | no |
| `feed` | `FeedCmd` (`lib.rs:923-936`) | `commands::feed::dispatch` (`lib.rs:1780`) | yes |
| `social` | `SocialCmd` (`lib.rs:939-1013`) | `commands::social::dispatch` (`lib.rs:1781`) | no |
| `notes` | `NotesCmd` (`lib.rs:1016-1092`) | `commands::notes::dispatch` (`lib.rs:1782`) | no |
| `repos` | `ReposCmd` (`lib.rs:1095-1139`) + `ReposProtectCmd` (`lib.rs:1142-1181`) | `commands::repos::dispatch` (`lib.rs:1783`) | no |
| `patches` | `PatchesCmd` (`lib.rs:1193-1296`) | `commands::patches::dispatch` (`lib.rs:1784`) | no |
| `issues` | `IssuesCmd` (`lib.rs:1443-1515`) | `commands::issues::dispatch` (`lib.rs:1785`) | no |
| `pr` | `PrCmd` (`lib.rs:1299-1440`) | `commands::pr::dispatch` (`lib.rs:1786`) | no |
| `media` | `MediaCmd` (`lib.rs:1528-1538`) | `commands::upload::dispatch_media` (`lib.rs:1787`) | no |
| `upload` | `UploadCmd` (`lib.rs:1518-1525`) | `commands::upload::dispatch` (`lib.rs:1788`) | no |
| `mem` | `MemCmd` (`lib.rs:1541-1621`) | `commands::mem::dispatch` (`lib.rs:1789`) | no |
| `moderation` | `ModerationCmd` (`lib.rs:1644-1731`) | `commands::moderation::dispatch` (`lib.rs:1790`) | yes |
| `pack` | `PackCmd` (`lib.rs:1624-1641`) | short-circuited before client construction (`lib.rs:1737-1743`) | no |

Supporting value enums, all `clap::ValueEnum`: `ChannelType` (`lib.rs:101-106`,
`Display` at `108-116`), `ChannelVisibility` (`lib.rs:118-123`, `Display` at
`125-133`), `PresenceStatus` (`lib.rs:135-143`, `Display` at `152-160`),
`EmojiScope` (`lib.rs:145-150`, **no `Display` impl** unlike its three
siblings), `RespondToArg` (`lib.rs:242-257`, with a `to_wire()` string mapping
at `249-256`), `RepoPushRole` (`lib.rs:1183-1190`).

#### Relay client session state (`BuzzClient`)

`client.rs:521-529` — five fields, no `Debug`/`Clone` derive, no connection
pooling state of its own beyond `reqwest`'s:

| Field | Type | Meaning |
|---|---|---|
| `http` | `reqwest::Client` | built once in `new` with env-driven timeouts (`client.rs:547-551`) |
| `relay_url` | `String` | normalized base, no trailing slash (`client.rs:523`) |
| `keys` | `nostr::Keys` | the identity; the secret lives here for process lifetime (`client.rs:524`) |
| `auth_tag` | `Option<Tag>` | NIP-OA tag injected into every signed event (`client.rs:526`) |
| `auth_tag_json` | `Option<String>` | raw JSON for the `x-auth-tag` header (`client.rs:528`) |

There is no persistent session: no cookie jar, no token cache, no WebSocket
kept open. Every HTTP call re-signs a fresh NIP-98 event
(`client.rs:84-110`), and the only WebSocket is opened per-publish and closed
(`client.rs:1073-1098`). No local config or state file is read or written by
this layer — `grep -rn 'dirs::\|home_dir' lib.rs client.rs validate.rs error.rs agent_management.rs`
returns zero matches (the `dirs` dependency is used only by the sibling-owned
`commands/channel_templates.rs`).

#### Request / filter representation

Filters are untyped `serde_json::Value` objects, never a typed struct
(`client.rs:683-713`, `query`/`query_multi` at `767-801`). Pagination state is
written *into* the filter as a composite cursor:

| Key | Written by | Value |
|---|---|---|
| `limit` | `query_pages` (`client.rs:690`) | `min(remaining, QUERY_PAGE_SIZE)`; `QUERY_PAGE_SIZE = 500` (`client.rs:498`) |
| `until` | `advance_query_cursor` (`client.rs:515`) | `created_at` of the page's last event |
| `before_id` | `advance_query_cursor` (`client.rs:516`) | 64-hex `id` of the page's last event |

`advance_query_cursor` (`client.rs:500-519`) validates the id as 64 ASCII hex
chars before using it and errors `CliError::Other` when `created_at`/`id` are
missing or malformed. Tests: `query_cursor_uses_last_events_composite_sort_key`
(`client.rs:2305-2317`) and `query_cursor_rejects_missing_or_malformed_sort_key`
(`client.rs:2319-2331`).

Retry policy constants are data too: `RETRY_MAX_ATTEMPTS = 3`
(`client.rs:122`), `RETRY_BASE_SECS = [0.5, 1.5]` (`client.rs:126`),
`RETRY_IN_MAX_SECS = 30` (`client.rs:130`). The invariant
`RETRY_BASE_SECS.len() == RETRY_MAX_ATTEMPTS - 1` is asserted by
`retry_constants_are_sensible` (`client.rs:1547-1552`) — without it,
`jitter_delay` (`client.rs:133-135`) would index out of bounds.

#### Output envelopes

Three shapes leave this layer, all as single-line JSON on stdout:

| Envelope | Producer | Shape |
|---|---|---|
| Read array | `normalize_events` (`client.rs:1307-1323`) | `[{id, pubkey, kind, content, created_at, tags}]` — `sig` is dropped, missing fields defaulted to `""`/`0`/`[]` |
| Write result | `normalize_write_response` (`client.rs:1420-1432`) | `{event_id, accepted, message}`; falls back to the raw relay text when neither `event_id` nor `accepted` is present |
| Create result | `create_response_with_id` (`client.rs:1391-1399`) / `print_create_response` (`client.rs:1401-1403`) | write result plus an injected `<id_key>: <id_val>`, and `accepted: true` when the relay omitted it |
| Error | `print_error` (`error.rs:127-136`) | `{error: <category>, message, retryable}` on **stderr** |

The `AGENTS.md` claim that "all reads return sig-stripped JSON arrays" is only
structurally guaranteed for readers that call `normalize_events`, and
`grep -rln 'normalize_events' commands/` lists just two files
(`commands/feed.rs`, `commands/messages.rs`). Other read commands print the
relay body verbatim (e.g. `commands/social.rs:78`, `commands/issues.rs:32`,
`commands/moderation.rs:115`). Whether the relay's own `/query` serializer
omits `sig` is outside this group and I did not verify it, so treat
"sig-stripped everywhere" as unproven rather than false.

`BuzzClient::submit_event` and friends return `String` (raw relay body), not a
typed response — normalization is the caller's job, which is why
`normalize_write_response` has 47 call sites in `commands/`
(`grep -rn 'normalize_write_response' commands/ | wc -l` → 47).

Relay write responses sometimes smuggle a nested payload:
`extract_relay_response_field` (`client.rs:1407-1418`) parses
`message: "response:{...}"` and pulls a named field out — the shape used by
workflow creates (test at `client.rs:2333-2340`).

#### Media types

`BlobDescriptor` (`client.rs:12-38`) is the only serde-typed relay response in
the group: `url`, `sha256`, `size`, `type` (renamed from `mime_type`),
`uploaded`, plus optional `dim`, `blurhash`, `thumb`, `duration` (all
`skip_serializing_if = Option::is_none`). `build_imeta_tag`
(`client.rs:40-62`) flattens it into a NIP-92 `imeta` tag array of
`"key value"` strings.

Upload constraints are constants: `ALLOWED_MIMES` = jpeg/png/gif/webp/mp4
(`client.rs:64-71`), `MAX_IMAGE_BYTES = 50 MiB` (`client.rs:73`),
`MAX_VIDEO_BYTES = 500 MiB` (`client.rs:76`).

#### Error model

`enum CliError` (`error.rs:4-45`) — nine variants, `thiserror`-derived:

| Variant | Payload | Display |
|---|---|---|
| `Usage` | `String` | `{0}` (`error.rs:6-8`) |
| `Relay` | `{status: u16, body: String}` | `relay error {status}: {body}` (`error.rs:10-12`) |
| `Network` | `#[from] reqwest::Error` | `network error: <full source chain>` (`error.rs:14-16`, chain walker `error.rs:49-61`) |
| `Auth` | `String` | `auth error: {0}` (`error.rs:18-20`) |
| `Key` | `String` | `key error: {0}` (`error.rs:22-24`) |
| `Conflict` | `String` | `conflict: {0}` — NIP-33 LWW (`error.rs:26-29`) |
| `NotFound` | `String` | `{0}` (`error.rs:31-34`) |
| `DeliveryUnknown` | `String` | `delivery unknown: {0}` (`error.rs:36-42`) |
| `Other` | `String` | `{0}` (`error.rs:44-45`) |

#### Agent-draft payload model (`agent_management.rs`)

A three-layer envelope encrypted to the owner and wrapped in an observer frame:

| Type | Fields | Site |
|---|---|---|
| `CreateAgentDraft` | `channel_id`, `display_name`, `system_prompt` (camelCase on the wire) | `agent_management.rs:15-20` |
| `UpdateAgentDraft` | `channel_id`, `agent_name` + optional `display_name`, `system_prompt`, `runtime`, `provider`, `model`, `respond_to` | `agent_management.rs:23-40` |
| `ManagementRequest<T>` | `type` (always `agent_management_request`, `agent_management.rs:9`), `action`, `request_id` (UUIDv4), `request` | `agent_management.rs:42-49` |
| `ObserverEvent<T>` | `seq` (hardcoded `0`, `agent_management.rs:96`), `timestamp` (RFC3339), `kind`, `agent_index`, `channel_id`, `session_id`, `turn_id`, `payload` | `agent_management.rs:51-61` |
| `BuiltDraftRequest` | `event`, `request_id`, `action` | `agent_management.rs:63-68` |

Size caps are data: `MAX_NAME_CHARS = 120`, `MAX_PROMPT_CHARS = 20_000`
(`agent_management.rs:10-11`), generic optional-field cap 300 chars
(`agent_management.rs:83-85`). The resulting event is kind 24200 with `p`,
observer-agent and observer-frame tags and **no `h` tag** — asserted by
`create_is_owner_encrypted_and_matches_desktop_contract`
(`agent_management.rs:188-241`, kind assertion at `:206`, absent-`h` assertion
at `:225-227`).

#### Validation-layer constants

`MAX_CONTENT_BYTES = 65_536` (`validate.rs:4`) enforced by
`validate_content_size` (`validate.rs:64-72`); `MAX_DIFF_BYTES = 61_440`
(`validate.rs:7`) is *not* enforced here — `truncate_diff`
(`validate.rs:103-121`) takes `max_bytes` as a parameter and the constant is
referenced only from the sibling-owned `commands/messages.rs`
(2 hits from `grep -rn 'MAX_DIFF_BYTES' commands/`).

#### Test coverage for this aspect

Command-tree shape is the best-tested data in the group: `cli_definition_is_valid`
(`lib.rs:1800-1803`, clap `debug_assert`), `command_inventory_is_stable`
(`lib.rs:1806-1853`, pins all 21 group names), `subcommand_names_are_stable`
(`lib.rs:1855-1994`), `subcommand_counts_are_stable` (`lib.rs:1996-2034`).
Gaps: the names test omits per-group assertions for `mem` and `notes`
(`awk 'NR>=1855 && NR<=2034' lib.rs | grep '"mem"'` → zero matches), and the
counts test covers 18 of 21 groups (`mem`, `moderation`, `notes` absent from
the list at `lib.rs:1997-2016`). Envelope producers `normalize_events`,
`normalize_write_response` and `build_imeta_tag` have **no tests at all** —
`grep -c 'normalize_events' <(awk 'NR>=1434' client.rs)` returns 0, same for
the other two.
