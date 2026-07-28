## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: API Surface

#### Rust library surface

`buzz-cli` ships both a lib (`buzz_cli`, `Cargo.toml:11-13`) and a bin (`buzz`,
`Cargo.toml:15-17`). `main.rs` is a 4-line shim:
`std::process::exit(buzz_cli::run_from_args(std::env::args()).await)`
(`main.rs:1-4`).

Module visibility (`lib.rs:1-5`): only `agent_management` is `pub`; `client`,
`commands`, `error`, `validate` are private. Everything `pub` inside
`client.rs`/`validate.rs`/`error.rs` is therefore crate-internal despite the
keyword.

| Public item | Kind | Site |
|---|---|---|
| `run_from_args<I, S>(args) -> i32` | async fn — the only real entry point | `lib.rs:23-60` |
| `ChannelType`, `ChannelVisibility`, `PresenceStatus`, `EmojiScope`, `OutputFormat`, `RespondToArg`, `RepoPushRole` | value enums | `lib.rs:101,118,135,145,164,242,1183` |
| `AgentsCmd`, `MessagesCmd`, `ChannelsCmd`, `CanvasCmd`, `ReactionsCmd`, `EmojiCmd`, `DmsCmd`, `UsersCmd`, `WorkflowsCmd`, `FeedCmd`, `SocialCmd`, `NotesCmd`, `ReposCmd`, `ReposProtectCmd`, `PatchesCmd`, `PrCmd`, `IssuesCmd`, `UploadCmd`, `MediaCmd`, `MemCmd`, `PackCmd`, `ModerationCmd` | subcommand enums | `lib.rs:260`–`lib.rs:1644` |
| `agent_management::{CreateAgentDraft, UpdateAgentDraft, BuiltDraftRequest, build_create, build_update}` | structs + fns | `agent_management.rs:15,23,64,128,144` |

**Exported but unreachable:** the 22 `*Cmd` enums are `pub`, yet `Cli`
(`lib.rs:79`) and `Cmd` (`lib.rs:175`) are private and no public function takes
or returns a `*Cmd`. An external crate can name `buzz_cli::MessagesCmd` but has
no way to feed one into the CLI. The single downstream consumer uses only the
entry point: `crates/buzz-dev-mcp/src/lib.rs:170` calls
`buzz_cli::run_from_args(std::env::args())` when invoked as `buzz`
(`grep -rn 'buzz_cli::' crates/ | grep -v '^crates/buzz-cli/'` → that one hit).
`crates/git-sign-nostr/Cargo.toml:23` mentions buzz-cli only in a comment.

Undocumented public items (AGENTS.md: "New public API must have doc comments"):
`ChannelType` (`lib.rs:101`), `ChannelVisibility` (`lib.rs:118`),
`PresenceStatus` (`lib.rs:135`), `EmojiScope` (`lib.rs:145`), and all of
`AgentsCmd`/`MessagesCmd`/`ChannelsCmd`/`CanvasCmd`/`ReactionsCmd`/`EmojiCmd`/
`DmsCmd`/`UsersCmd`/`WorkflowsCmd`/`FeedCmd`/`SocialCmd` carry no `///`
comment; `NotesCmd` (`lib.rs:1016`), `MemCmd` (`lib.rs:1540`), `PackCmd`
(`lib.rs:1623`), `ModerationCmd` (`lib.rs:1636-1642`), `ReposProtectCmd`
(`lib.rs:1141`) and `RepoPushRole` (`lib.rs:1182`) do. In
`agent_management.rs` the module has a `//!` header (`:1`) but none of its five
public items are documented.

#### Crate-internal client surface (`client.rs`)

| Item | Signature | Site |
|---|---|---|
| `BuzzClient::new` | `(String, Keys, Option<Tag>, Option<String>) -> Result<Self, CliError>` | `client.rs:541-560` |
| `keys` | `-> &Keys` | `client.rs:562-564` |
| `relay_url` | `-> &str` — `#[allow(dead_code)]`, zero callers in `commands/` | `client.rs:567-570` |
| `auth_tag_owner_hex` | `-> Option<String>` (index 1 of the auth tag) | `client.rs:576-580` |
| `sign_event` | injects NIP-OA tag, then *rejects* events with an unexpected `auth` tag count | `client.rs:588-614` |
| `sign_event_unchecked` | verbatim signing, NIP-IA only | `client.rs:743-747` |
| `get_public` | unauthenticated GET, `Accept: application/nostr+json` | `client.rs:753-765` |
| `get_authed` | NIP-98 GET of an arbitrary root-relative path | `client.rs:836-856` |
| `query` / `query_multi` | `POST /query` with one/many filters | `client.rs:767-801` |
| `query_paginated` / `query_all` | cursor-following wrappers over `query_pages` | `client.rs:715-727` |
| `count` | `POST /count` — `#[allow(dead_code)]`, zero callers in `commands/` | `client.rs:802-834` |
| `submit_event` | kind-aware dispatch to moderation vs stored policy | `client.rs:863-870` |
| `publish_ephemeral_event` | WS publish via `buzz_ws_client` | `client.rs:1073-1098` |
| `upload_file` | Blossom PUT with legacy fallback | `client.rs:1100-1227` |
| `download_media` | Blossom GET with origin pinning | `client.rs:1229-1256` |
| free fns | `normalize_relay_url` (`:1291`), `normalize_events` (`:1307`), `extract_d_tag` (`:1325`), `extract_tag_value` (`:1346`), `extract_p_tags` (`:1366`), `create_response_with_id` (`:1391`), `print_create_response` (`:1401`), `extract_relay_response_field` (`:1407`), `normalize_write_response` (`:1420`), `build_imeta_tag` (`:40`) | |

`validate.rs` exposes `parse_event_id` (`:11`), `parse_uuid` (`:19`),
`validate_uuid` (`:23`), `validate_hex64` (`:29`), `validate_repo_id` (`:39`),
`validate_content_size` (`:64`), `truncate_diff` (`:103`), `infer_language`
(`:124`), `sdk_err` (`:155`), `read_or_stdin` (`:162`), `read_file_or_stdin`
(`:180`) — plus `percent_encode` (`:76`), which is gated `#[cfg(test)]`
(`validate.rs:75`) and so is not part of the production surface at all.

`error.rs` exposes `CliError` (`:4`), `is_retryable_error` (`:74`), `exit_code`
(`:92`), `print_error` (`:127`).

#### CLI surface

Global flags (must precede the subcommand — see below):

| Flag | Value | Default |
|---|---|---|
| `--relay <RELAY>` | URL | `http://localhost:3000` (`lib.rs:81`) |
| `--private-key <PRIVATE_KEY>` | hex or nsec | none (`lib.rs:85`) |
| `--auth-tag <AUTH_TAG>` | NIP-OA JSON | none (`lib.rs:89`) |
| `--format <json\|compact>` | enum | `json` (`lib.rs:93`) |
| `-h`, `--help` | — | exits 0 (`lib.rs:48-52`) |

`--version` is **not** implemented: no `version` attribute on
`#[command(...)]` (`lib.rs:62-78`). Verified by running the built binary —
`buzz --version` prints
`{"error":"user_error","message":"error: unexpected argument '--version' found…}`
and exits **1**, even though `run_from_args` has a branch commented
"`--help` and `--version`: print normally" (`lib.rs:49`).

Subcommand inventory, as pinned by the crate's own tests
(`lib.rs:1806-1853`, `1855-1994`, `1996-2034`):

| Group | Subcommands | Count |
|---|---|---|
| `agents` | archive, archived, draft-create, draft-update, unarchive | 5 |
| `messages` | delete, edit, get, search, send, send-diff, thread, vote | 8 |
| `channels` | add-member, archive, create, delete, get, join, leave, list, members, purpose, remove-member, search, set-add-policy, topic, unarchive, update | 16 |
| `canvas` | get, set | 2 |
| `reactions` | add, get, remove | 3 |
| `emoji` | export, import, list, rm, set | 5 |
| `dms` | add-member, hide, list, open | 4 |
| `users` | get, presence, set-presence, set-profile | 4 |
| `workflows` | approve, create, delete, get, list, runs, trigger, update | 8 |
| `feed` | get | 1 |
| `social` | contacts, event, list, notes, publish, set-contacts, set-list | 7 |
| `notes` | get, ls, rm, set | 4 (untested inventory) |
| `repos` | create, get, list, protect{list,remove,set} | 4 (+3) |
| `patches` | get, list, send, status | 4 |
| `issues` | create, get, list, status | 4 |
| `pr` | get, list, open, status, update | 5 |
| `media` | get | 1 |
| `upload` | file | 1 |
| `mem` | get, hash, ls, patch, rm, set | 6 (untested inventory) |
| `moderation` | audit, ban, reports, resolve, restricted, timeout, unban, untimeout | 8 |
| `pack` | inspect, validate | 2 |

`--format` position is a hard contract, not a convention: because the arg lives
on the top-level `Cli` without `global = true`, clap rejects it after the
subcommand. Verified against the built binary:
`buzz messages get --channel <uuid> --format compact` →
`error: unexpected argument '--format' found`, exit 1; while
`buzz --format compact channels list` parses and proceeds to the relay call.
This confirms `AGENTS.md:192-193` and **contradicts the example at
`AGENTS.md:181`** (`buzz messages thread --channel … --event … --format compact`),
which cannot parse.

`messages search` has **no `--kinds` flag** (`lib.rs:472-489` defines only
`--query`, `--author`, `--since`, `--limit`). Verified:
`buzz messages search --query x --kinds 9` → `error: unexpected argument
'--kinds' found`, exit 1. `AGENTS.md` gotcha 3 ("`messages search` must include
`--kinds` … Pass at least `--kinds 9,45001,45003`") is therefore unfollowable;
the kinds are hardcoded in the sibling-owned handler
(`commands/messages.rs:361`). `--kinds` exists only on `messages get`
(`lib.rs:453-455`), where it is optional and defaults are supplied downstream
(`commands/messages.rs:276`).

#### Exit-code contract

`exit_code` (`error.rs:92-107`) plus the clap-parse path (`lib.rs:44-47`):

| Class | Code | Site | Verified |
|---|---|---|---|
| success | 0 | `lib.rs:55` | — |
| clap usage error | 1 | `lib.rs:44-47` | yes: `--version`, stray `--format`, stray `--kinds` all exit 1 |
| `Usage` | 1 | `error.rs:94` | yes: `channels get --channel not-a-uuid` → 1, `{"error":"user_error"}` |
| `NotFound` | 1 | `error.rs:104` | not exercised |
| `Relay{401,403}` | 3 | `error.rs:96-100` | test `query_403_is_not_retried` (`client.rs:1949-1977`) covers the error value, not the code |
| `Relay{other}` | 2 | `error.rs:99` | — |
| `Network` | 2 | `error.rs:102` | yes: dead port → 2, `{"error":"network_error","retryable":true}` |
| `Auth` | 3 | `error.rs:103` | yes: unset key → 3; malformed `BUZZ_AUTH_TAG` → 3 |
| `Key` | 3 | `error.rs:103` | yes: `BUZZ_PRIVATE_KEY=zzz` → 3, `{"error":"key_error"}` |
| `Conflict` | 5 | `error.rs:103` | not exercised |
| `DeliveryUnknown` | 2 | `error.rs:105` | not exercised at process level |
| `Other` | 4 | `error.rs:106` | not exercised |
| `--help` / `-h` | 0 | `lib.rs:48-52` | yes |

`exit_code` itself has **no unit test**:
`grep -c 'exit_code' <(awk 'NR>=137' error.rs)` returns 0. The documented
contract in `AGENTS.md:189-190` and `lib.rs:76` matches the implementation for
the six classes it names, but silently omits that `NotFound` shares code 1 with
input errors and `DeliveryUnknown` shares code 2 with network errors — so an
agent cannot distinguish "the write may have landed" from "the connection
failed" by exit code alone, only by the `error` field on stderr
(`delivery_unknown` vs `network_error`, `error.rs:117-125`).

A schemeless `--relay` is mis-classified: `buzz --relay notaurl channels list`
exits **2** with `{"error":"network_error","message":"network error: builder
error: relative URL without a base","retryable":false}` — a pure input error
reported as a network error, because nothing validates the scheme
(`normalize_relay_url`, `client.rs:1291-1297`, only does substring replacement).

#### Outbound relay surface

| Method + path | Auth | Caller |
|---|---|---|
| `POST /query` | NIP-98 kind 27235 + optional `x-auth-tag` | `query_multi` (`client.rs:773-801`) |
| `POST /count` | same | `count` (`client.rs:803-834`) — dead |
| `POST /events` | same | `submit_moderation_event` (`client.rs:873`), `submit_stored_event` (`client.rs:1024`) |
| `PUT /upload` | Blossom kind 24242 `t=upload` | `upload_file` (`client.rs:1152-1178`) |
| `PUT /media/upload` | same | legacy fallback on 404/405 (`client.rs:1195-1226`) |
| `GET /media/<sha256[.ext]>` | Blossom kind 24242 `t=get` | `download_media` (`client.rs:1229-1256`) |
| `GET <arbitrary path>` (authed) | NIP-98 GET | `get_authed` (`client.rs:836-856`); used for `/moderation/reports?…`, `/moderation/restricted`, `/moderation/audit?limit=…` (`commands/moderation.rs:114,120,127`) |
| `GET /` (NIP-11) | none | `get_public` (`client.rs:753-765`); called with `"/"` from `commands/agents.rs:273` |
| WebSocket `EVENT` after NIP-42 `AUTH` | Nostr AUTH | `publish_ephemeral_event` → `buzz_ws_client::publish_event` (`client.rs:1084`) |

The `/moderation/*` GET endpoints are outside the HTTP surface AGENTS.md
documents: `grep -c '/moderation/' AGENTS.md` → 0. Likewise the `x-auth-tag`
request header (`client.rs:616-621`) appears in no markdown doc
(`grep -rln 'x-auth-tag' --include='*.md' .` outside `.aidlc/` → no hits).

WebSocket message flow is fully delegated, not reimplemented: connect →
NIP-42 challenge wait → AUTH → EVENT → OK → close, all inside
`buzz_ws_client::publish_event` (`crates/buzz-ws-client/src/connection.rs:277-293`),
with a 75 s outer budget chosen to exceed the 20+20+30 s inner ceilings
(`client.rs:1075-1085`).
