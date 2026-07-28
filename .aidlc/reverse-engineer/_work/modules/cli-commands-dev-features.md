## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Features

Eleven modules, 46 operator- or agent-invocable subcommands. Grouped by what they let you do,
with the limits stated alongside.

#### Agent persistent memory (NIP-AE) — `mem`, 6 subcommands

An agent can keep encrypted, owner-scoped notes on the relay and edit them safely under
concurrency.

| Capability | Command | Limits |
|---|---|---|
| List every live memory slug | `mem ls` (`mem.rs:189`) | excludes `core` and tombstones (`mem.rs:243-248`); asks for 5000 events but the relay caps around 1000 (`mem.rs:201` vs `crates/buzz-db/src/event.rs:346-347`) with no truncation signal |
| Read a value verbatim | `mem get <slug>` (`mem.rs:277`) | raw bytes, no trailing newline, so it round-trips with `mem set <slug> -` (`mem.rs:296-303`); absent or tombstoned → exit 1 |
| Capture a pre-edit digest | `mem hash <slug>` (`mem.rs:508`) | 64 hex chars, matches `printf '%s' "$v" \| sha256sum` (`mem.rs:373-381`) |
| Write a whole value | `mem set <slug> <value\|->` (`mem.rs:314`) | ≤ 65 535 bytes (`mem.rs:326-329`); an empty **stdin** read is refused unless `--allow-empty` (`mem.rs:331-338`) — a literal `""` argument is still accepted |
| Edit surgically with a unified diff | `mem patch <slug>` (`mem.rs:538`) | `--base-hash` required unless `--no-base-hash`; hunks must match verbatim **at their declared line numbers** — no fuzz, no sliding (`mem.rs:611-618`); single virtual file only (`mem.rs:600-608`); no-context insertion into a non-empty value refused (`mem.rs:435-443`); `--dry-run` echoes the input diff plus the would-be digest and writes nothing (`mem.rs:699-702`) |
| Tombstone a slug | `mem rm <slug>` (`mem.rs:706`) | `core` cannot be tombstoned (`mem.rs:711-716`) |
| Owner-side recovery | `--agent <pubkey>` on `ls`/`get`/`hash` (`mem.rs:63-74`) | read-only: `set`/`patch`/`rm` have no `--agent` flag at all (`lib.rs:1571-1622`) |

What you cannot do: enumerate slugs without decrypting (the `d` tag is an HMAC, `mem.rs:148`);
patch across multiple slugs in one call; recover from a lost write without re-reading the
head; get a paginated listing.

#### Git collaboration over Nostr (NIP-34) — `repos`, `patches`, `pr`, `issues`, 18 subcommands

| Capability | Command | Limits |
|---|---|---|
| Announce a repository | `repos create --id …` (`repos.rs:202`) | repo id must be 1-64 chars of `[A-Za-z0-9._-]`, no leading dot, no `..` (`validate.rs:40-61`) |
| Look up a repo announcement | `repos get --id [--owner]` (`repos.rs:232`) | without `--owner`, multiple same-named repos from different owners come back together — noted in the comment at `repos.rs:237-238` |
| List a pubkey's repos | `repos list [--owner] [--limit]` (`repos.rs:256`) | defaults to your own pubkey (`repos.rs:262-268`) |
| Inspect branch/tag protection | `repos protect list --id` (`repos.rs:295`) | shows unknown future rules and a `validation_error` string so a broken rule is repairable rather than invisible (`repos.rs:141-171`) |
| Set a protection rule | `repos protect set --id --ref … [--push owner\|admin\|member] [--no-force-push] [--no-delete] [--require-patch]` (`repos.rs:301`) | your own repos only (`repos.rs:22`); at least one constraint required (`repos.rs:521` test); **full replacement** for that exact ref pattern; refuses to write if the stored announcement already has a malformed rule (`repos.rs:118-131`); ≤ 50 rules per repo (`git_perms.rs:19`) |
| Remove a protection rule | `repos protect remove --id --ref` (`repos.rs:327`) | exact pattern match required; absent rule → exit 1 (`repos.rs:334-338`) |
| Send a patch series | `patches send --patch-file <path\|->` (`patches.rs:9`) | reads a `git format-patch` file or stdin; optional `--root`/`--root-revision`, `--reply-to`, `--commit`, `--parent-commit`, `--commit-pgp-sig`, `--committer 'name\|email\|ts\|tz'` |
| Read / list patches | `patches get --event`, `patches list --repo-owner --repo-id [--author] [--limit]` (`patches.rs:73`, `:84`) | raw relay array, unfiltered |
| Set patch status | `patches status --root --status open\|merged\|closed\|draft` (`patches.rs:114`) | `--revision`, `--q`, `--merge-commit`, `--applied-as-commit` are documented "merged only" but not enforced (`patches.rs:180-188`) |
| Open a pull request | `pr open --repo-owner --repo-id --subject --commit --clone …` (`pr.rs:20`) | `--clone` is required (`lib.rs:1327`); `--body` and `--body-file` are mutually exclusive (`pr.rs:9-11`); optional `--channel` binds it to a NIP-29 channel, `--revision-of` to a prior root |
| Update a PR tip | `pr update --pr --pr-author --commit --clone …` (`pr.rs:66`) | requires both PR event id and PR author pubkey |
| Read / list PRs | `pr get --event`, `pr list … [--author] [--label] [--limit]` (`pr.rs:107`, `:118`) | label filter is a `#t` tag match |
| Set PR status | `pr status --pr --status …` (`pr.rs:152`) | narrower than `patches status`: no `--revision`, no `--q`, no `--applied-as-commit` (all hardcoded empty at `pr.rs:199-207`) |
| File / read / list issues | `issues create`, `get`, `list` (`issues.rs:6`, `:36`, `:47`) | `--title` aliases `--subject` (`lib.rs:1451`) |
| Set issue status | `issues status --issue --status open\|resolved\|closed\|draft` (`issues.rs:81`) | narrowest of the three: no merge/revision affordances at all (`issues.rs:136-143`) |

Across all four modules, status events auto-`p`-tag the repo owner when `--repo-owner` is
supplied and let you add reviewers with repeated `--to` (`patches.rs:170-178`,
`pr.rs:186-194`, `issues.rs:127-135`).

What you cannot do: browse a repo's git tree, push, fetch, or clone — those are the relay's
git smart-HTTP surface, not this CLI; edit another owner's protection rules; list statuses
(there is no `patches statuses` / `pr statuses` read command anywhere in the group).

#### Agent identity lifecycle (NIP-IA / NIP-OA) — `agents`, 5 subcommands

| Capability | Command | Limits |
|---|---|---|
| Propose a new agent for owner review | `agents draft-create --channel --display-name --system-prompt` (`agents.rs:16`) | requires `BUZZ_AUTH_TAG` (`agents.rs:161-165`); nothing changes until the human saves it in Desktop (`"saved": false`, `agents.rs:43`); name ≤ 120 chars, prompt ≤ 20 000 chars (`agent_management.rs:11-12`) |
| Propose an edit to a personal agent | `agents draft-update --channel --agent-name [fields…]` (`agents.rs:50`) | at least one field must change (`agent_management.rs:170-179`); `--respond-to owner-only\|anyone` |
| Request an identity be archived | `agents archive <pubkey> [--reason] [--replaced-by]` (`agents.rs:93`) | the relay picks the consent path; an agent signing as itself can only ever satisfy the self path — stated in the help text at `lib.rs:275-279`; `--reason` ≤ 64 bytes and control-char free (`builders.rs:1706-1719`); `--replaced-by` must differ from the target (`builders.rs:1757-1761`) |
| Request an un-archive | `agents unarchive <pubkey> [--reason]` (`agents.rs:129`) | no `--replaced-by` — undefined on unarchive (`builders.rs:1806-1807`) |
| Verify the relay's archive snapshot | `agents archived` (`agents.rs:310`) | any trust failure is a nonzero exit, never a false-empty success (`agents.rs:306-309`); checks NIP-11 `self` authorship, exactly one NIP-70 `-` tag, and the signature before trusting a single pubkey (`agents.rs:320-372`) |

What you cannot do: create or update an agent directly (drafts are advisory by design); list
agents (no read command); archive a third party as an agent (the relay's consent model
blocks it).

#### Community moderation — `moderation`, 8 subcommands

| Capability | Command | Limits |
|---|---|---|
| Read the report queue | `moderation reports [--status] [--limit]` (`moderation.rs:105`) | mod-only relay endpoint; limit defaults to 50 and is clamped server-side to 500 (`crates/buzz-relay/src/api/bridge.rs:2084-2088`) |
| Resolve or dismiss a report | `moderation resolve --report --status --action [--reason]` (`moderation.rs:89`) | status ∈ `resolved\|dismissed`, action ∈ `delete\|kick\|ban\|timeout\|dismiss\|escalate` — validated in the SDK, not here (`builders.rs:1661-1676`); `--reason` is relayed to the reporter (`lib.rs:1685`) |
| Ban a member | `moderation ban --pubkey [--expires-in\|--expires-at] [--reason]` (`moderation.rs:34`) | omitting an expiry means permanent (`moderation.rs:39`); community-wide, no channel scope (`moderation.rs:16-18`) |
| Lift a ban | `moderation unban --pubkey` (`moderation.rs:51`) | — |
| Time out a member (write block) | `moderation timeout --pubkey --expires-in\|--expires-at [--reason]` (`moderation.rs:61`) | an expiry is **mandatory** (`moderation.rs:69-71`) — the one duration rule the CLI itself enforces |
| Clear a timeout early | `moderation untimeout --pubkey` (`moderation.rs:79`) | — |
| See who is restricted | `moderation restricted` (`moderation.rs:119`) | — |
| Read the audit trail | `moderation audit [--limit]` (`moderation.rs:125`) | newest first per help text (`lib.rs:1765`) |

The CLI performs **no local authorization check** — every mutation is a signed 9040-9044
command event the relay validates, authorizes, and executes without storing
(`moderation.rs:3-8`). Because those kinds execute before any dedup, `client.rs` gives them a
non-idempotent retry policy: an ambiguous outcome surfaces as `DeliveryUnknown` rather than
being re-sent (`client.rs:851-861`, `is_moderation_kind` at `client.rs:211-213`).

What you cannot do: read a single report by id; kick or delete content (those actions exist
only as `resolve --action` values the relay executes); scope any of it to one channel;
get `--format compact` output (the flag is accepted and discarded at `moderation.rs:136`).

#### Workflow authoring and operation — `workflows`, 8 subcommands

| Capability | Command | Limits |
|---|---|---|
| List / read definitions | `workflows list --channel`, `workflows get --workflow` (`workflows.rs:13`, `:38`) | `get` filters on `#d` with **no author constraint** and takes the first hit (`workflows.rs:40-47`), so a same-`d` definition from another author can win |
| Create from YAML | `workflows create --channel --yaml <text\|->` (`workflows.rs:98`) | YAML ≤ 64 KiB (`builders.rs:1468`); a client-side UUID is generated but a relay-supplied `workflow_id` wins (`workflows.rs:107-112`) |
| Update in place | `workflows update --channel --workflow --yaml` (`workflows.rs:119`) | `--channel` is required because the `h` tag is re-emitted (`builders.rs:1487-1490`) |
| Delete | `workflows delete --workflow` (`workflows.rs:139`) | kind:5 with an `a` coordinate bound to your own pubkey (`builders.rs:1502-1506`) |
| Trigger a run | `workflows trigger --workflow [--inputs '{…}']` (`workflows.rs:156`) | `--inputs` must be a JSON **object** (`workflows.rs:165-169`) and is **not** size-capped on that branch (`workflows.rs:170-181`) |
| Approve or deny a gated step | `workflows approve --token [--approved false] [--note]` (`workflows.rs:193`) | bare invocation grants (`lib.rs:909`); the relay is sent `hex(SHA256(token))`, never the token (`workflows.rs:205`) |
| Read run history | `workflows runs --workflow [--limit]` (`workflows.rs:66`) | **always returns `[]`** — the relay stores run history in the `workflow_runs` table and emits no 46001-46003 events; documented at `workflows.rs:62-65` and verified: no producer of those kinds exists anywhere in `crates/` |

`buzz-workflow`'s evalexpr conditions and the `/hooks/{id}` webhook trigger surface are not
reachable from this CLI at all — conditions live inside the YAML text this group only
transports, and no command in the group posts to `/hooks/`
(`grep -n 'hooks' ` across the eleven files returns zero matches).

#### Profiles and presence — `users`, 4 subcommands

| Capability | Command | Limits |
|---|---|---|
| Look up profiles | `users get [--pubkey …]` (`users.rs:12`) | ≤ 200 pubkeys (`users.rs:35-37`); no args = your own profile |
| Search by name | `users get --name <q>` (`users.rs:82`) | mutually exclusive with `--pubkey` (`users.rs:16-20`); NIP-50 server search narrowed by a client-side case-insensitive substring match (`users.rs:120-134`); returns `[]` on a relay without NIP-50 |
| Update your profile | `users set-profile [--name] [--avatar] [--about] [--nip05]` (`users.rs:150`) | at least one field (`users.rs:157-161`); read-merge-write preserves untouched fields (`users.rs:165-195`); the kind:0 `name` (username) field is not exposed and is written as `None` (`users.rs:200`) |
| Read presence | `users presence --pubkeys a,b,c` (`users.rs:247`) | **no count cap**, unlike `users get`; blank CSV yields `[]` |
| Set presence | `users set-presence --status online\|away\|offline` (`users.rs:298`) | kind 20001 is ephemeral, so this is the one `users` command that goes over WebSocket (`users.rs:292-301`) |

`users get` is one of only two commands in the group that honor `--format compact`
(`users.rs:62-74`, `users.rs:135-145`), reducing each profile to `{pubkey, display_name}`.

#### Media — `upload`, `media`, 2 subcommands

| Capability | Command | Limits |
|---|---|---|
| Upload a blob | `upload file --file <path>` (`upload.rs:6`) | MIME sniffed from magic bytes and checked against an allowlist (`client.rs:64-71`, `client.rs:1112-1119`); 50 MiB images, 500 MiB video (`client.rs:73-76`, `client.rs:1121-1132`); falls back from `PUT /upload` to `PUT /media/upload` on 404/405 (`client.rs:1180-1195`) |
| Download a blob | `media get <url\|sha256[.ext]> [-o path\|-]` (`upload.rs:19`) | input must resolve to a `/media/` path on the **same** relay origin (`client.rs:270-325`); `sha256`, `sha256.ext`, and `sha256.thumb.jpg` are the only accepted shapes (`client.rs:250-268`); writes to a file or raw to stdout (`upload.rs:21-33`) |

`upload file` is the only command in the group whose output is pretty-printed
(`upload.rs:9-10`). Note `AGENTS.md`'s PR-screenshot rule explicitly forbids using
`buzz upload` for PR images.

#### Local persona packs — `pack`, 2 subcommands

| Capability | Command | Limits |
|---|---|---|
| Validate a pack directory | `pack validate <path>` (`pack.rs:15`) | no relay connection — short-circuited before the client is built (`lib.rs:1735-1741`); warnings exit 0, errors exit 1; diagnostics to stderr |
| Show effective persona config | `pack inspect <path>` (`pack.rs:52`) | plain text only, no JSON mode; prints resolved model/provider, temperature, context limit, subscriptions, triggers, reply flags, MCP-server count, skills, avatar, a 77-char system-prompt preview, and derived `runtime_env_vars` (`pack.rs:66-149`) |

These are the only two commands in the entire CLI that work with no key and no relay —
`lib.rs:1744-1746` requires `BUZZ_PRIVATE_KEY` for everything else.

#### Cross-cutting limits worth knowing

- No command in this group paginates. Every read is one `POST /query`; result sets larger than
  the relay clamp (~1000 rows) are silently truncated. `grep -n 'query_paginated\|query_all'`
  across the eleven files returns zero matches.
- `--format compact` works on `users get` only. `moderation` accepts and drops it
  (`moderation.rs:136`); the other nine modules never receive it.
- `mem` and `pack` deliberately emit non-JSON output; everything else emits single-line JSON
  except `upload file`.
- Exit code 5 (write conflict) is reachable from exactly three commands in this group:
  `mem set`, `mem patch`, `mem rm` (`mem.rs:101`, `mem.rs:595`) and `repos protect
  set`/`remove` (`repos.rs:155`).
