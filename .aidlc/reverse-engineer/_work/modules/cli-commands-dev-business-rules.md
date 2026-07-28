## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Business Rules

#### Identity-perspective rules (`mem`)

`resolve_reader` (`mem.rs:54-79`) produces the `(agent, owner, their_pubkey)` triple that
drives both the query filter and the NIP-44 decrypt key:

| Flags | agent | owner | NIP-44 counterparty | Site |
|---|---|---|---|---|
| neither | CLI identity | from `--owner` or `BUZZ_AUTH_TAG` | owner | `mem.rs:76-78` |
| `--agent <pk>` | the supplied pubkey | CLI identity | the supplied agent | `mem.rs:63-74` |
| both | **rejected** | — | — | `mem.rs:56-60` |
| `--agent == CLI identity` | **rejected** | — | — | `mem.rs:67-72` |

Rules enforced:

- R1. `--owner` and `--agent` are mutually exclusive on read commands (`mem.rs:56-60`).
  Tested: `resolve_reader_rejects_owner_with_agent_flag` (`mem.rs:821`).
- R2. `--agent` must differ from the CLI identity (`mem.rs:67-72`). Tested:
  `resolve_reader_rejects_agent_flag_matching_cli_identity` (`mem.rs:837`).
- R3. Owner resolution order is `--owner` flag first, then the `auth_tag` owner slot
  (index 1), else a `Usage` error (`resolve_owner`, `mem.rs:33-49`).
- R4. Write commands (`set`, `patch`, `rm`) accept `--owner` but **not** `--agent`
  (`lib.rs:1571-1622` defines no `agent` field on `Set`/`Patch`/`Rm`), so owner-side
  recovery is read-only by construction.

`fetch_head` picks the ECDH counterparty by comparing the CLI identity to the agent
(`mem.rs:143-147`) — the conversation key is symmetric per `engram.rs:134-138`, so both
perspectives derive the same `d` tag and the same decrypt key.

#### NIP-33 LWW conflict detection and exit code 5

Two independent mechanisms, both landing on `CliError::Conflict` → exit 5.

**Mechanism A — relay-reported domination.** `submit_engram` (`mem.rs:93-105`) parses the
`POST /events` reply and applies three rules in order:

1. `accepted` missing or false → `Other` (exit 4), message passed through (`mem.rs:97-104`).
2. `accepted == true` **and** `message` starts with `"duplicate:"` or equals `"duplicate"`
   → `Conflict` (`mem.rs:100-104`).
3. otherwise → success.

`repos.rs`'s `validate_write_response` (`repos.rs:173-193`) is a byte-for-byte re-derivation
of the same rule with the same two message forms and a repo-flavoured error string
(`repos.rs:154-158`). It is not shared code — see Integrations.

The relay side that produces the `"duplicate:"` message is
`crates/buzz-relay/src/handlers/ingest.rs:2451-2457` (`was_inserted == false`), and for
NIP-33 kinds `was_inserted` is false exactly when the incoming `(created_at, id)` tuple is
dominated by the stored head:
`created_at < accepted_ts || (created_at == accepted_ts && incoming_id >= accepted_id)`
(`crates/buzz-db/src/lib.rs:3719-3735`). So the CLI's interpretation is faithful for a
genuinely-stale write.

**Important caveat the code does not handle.** The identical `"duplicate:"` reply is also
returned for a byte-identical *re-submission* of an event that already landed (same event id
⇒ `incoming_id >= accepted_id` holds trivially). `client.rs`'s `submit_stored_event` retries
the **same serialized event bytes** on a transient failure (`client.rs:1030-1050`: `body` is
cloned per attempt, only the NIP-98 wrapper is re-signed). If attempt 1 stores the event but
its response body is lost, attempt 2 receives `"duplicate:"` and the CLI reports exit 5 —
a *false* conflict on a write that actually succeeded. There is no test for this path;
`repos.rs:628`'s test only pins the string-matching rule, not the retry interaction.

**Mechanism B — client-side base-hash gate.** `mem patch` compares `sha256_hex(current)`
against `--base-hash` (case-normalized) and raises `Conflict` on mismatch
(`mem.rs:592-599`). This is the only optimistic-concurrency check in the group that fires
before any write is attempted.

#### Monotonic timestamp rules

- Engram writes compute `created_at = max(now, prior_head + 1)` via
  `engram::monotonic_created_at` (`engram.rs:588-593`), called at `mem.rs:353`,
  `mem.rs:677-678`, `mem.rs:722`. Each write therefore performs a read of the current head
  first (`mem.rs:351`, `mem.rs:590`, `mem.rs:720`) — the write path is always read-then-write.
- Repo announcement updates use a **different** rule: `existing.created_at + 1` with
  `checked_add`, deliberately *not* wall-clock (`repos.rs:132-136`). The comment at
  `repos.rs:129-130` states the reason: wall-clock would let a delayed writer leapfrog an
  intervening update and erase metadata. Overflow is an `Other` error, not a panic.
  Tested: `protection_update_preserves_metadata_and_replaces_only_matching_pattern` asserts
  `created_at == 101` from an input of `100` (`repos.rs:445`).

#### Memory value and patch rules

| Rule | Enforcement | Site | Test |
|---|---|---|---|
| slug grammar: `core` or `mem/<seg>(/<seg>)*`, ≤ 255 bytes, segments ≤ 64 bytes of `[a-z0-9][a-z0-9_-]*` | `engram::normalize_slug` → `validate_slug` | called at `mem.rs:279`, `:315`, `:510`, `:539`, `:707`; rules at `engram.rs:67-112` | in `buzz-core` |
| a bare slug is auto-prefixed with `mem/` | `normalize_slug` | `engram.rs:123-131` | in `buzz-core` |
| stdin read is bounded to `NIP44_PLAINTEXT_MAX + 1` (65 536 B) | explicit `.take()` | `mem.rs:322-325`, `mem.rs:576` | none |
| value over 65 535 B rejected | length check after read | `mem.rs:326-329` (set), `mem.rs:630-636` (patch result) | none |
| empty **stdin** value rejected unless `--allow-empty` | `mem.rs:331-338` | — | none |
| a literal `""` positional value is **accepted** | the guard only runs on the `-` branch (`mem.rs:319`) | documented at `mem.rs:309-311` | none |
| empty patch from stdin rejected unconditionally (no `--allow-empty` escape) | `mem.rs:586-591` | — | none |
| `--base-hash` required unless `--no-base-hash`; the two are mutually exclusive | `mem.rs:545-561` | — | none |
| `--base-hash` must be exactly 64 ASCII hex chars | `mem.rs:562-568` | — | none |
| multi-file patch rejected | count of `"--- "` lines > 1 | `mem.rs:600-608` | `multi_file_header_count` (`mem.rs:1035`) — but the test only exercises the counting expression, not `cmd_patch` |
| hunks must apply at their declared line numbers, no fuzz, no slide | `verify_hunks_at_declared_position` | `mem.rs:400-482`, invoked `mem.rs:611-618` | 5 tests, `mem.rs:938`-`mem.rs:1033` |
| no-context insertion into a non-empty value refused | `mem.rs:435-443` | documented limitation, "failure mode is rejection, not corruption" | none (the accepted empty-file case is tested at `mem.rs:968`) |
| empty patch **result** rejected unless `--allow-empty` | `mem.rs:637-643` | — | none |
| `mem rm core` refused; operator told to `mem set core ''` | `mem.rs:711-716` | rationale at `mem.rs:701-704`: NIP-AE defines tombstones only for memory entries | none |
| `mem ls` excludes `core` and tombstones | `mem.rs:238-250` | per spec, comment at `mem.rs:238` | none |
| listings sorted by slug | `mem.rs:260` | — | none |
| a corrupt event must not deny-of-service the whole listing | `parse_events` skips undeserializable entries (`mem.rs:114-133`); signature/decrypt failures `continue` (`mem.rs:164-172`, `mem.rs:207-231`) | — | none |

`mem get` and `mem hash` funnel through `fetch_value` (`mem.rs:484-506`), which maps absent
head, absent body, and tombstone all to `NotFound` and unwraps `Body::Core`'s `profile` as
if it were a value (`mem.rs:503`) — so `mem hash core` hashes the profile text.

#### Repo id and protection-rule semantics

- `validate_repo_id` (`validate.rs:40-61`) requires 1-64 chars of `[A-Za-z0-9._-]`, forbids a
  leading `.` and any `..`. Applied at `repos.rs:207`, `repos.rs:236`, `repos.rs:288`, plus
  every `pr`/`patches`/`issues` command that names a repo. Nine tests in `validate.rs:410-449`.
- `repos protect *` operates **only on the caller's own announcement**: the head fetch pins
  `authors: [self]` (`repos.rs:22`) and a miss is `NotFound` (`repos.rs:287-293`). There is
  no `--owner` escape hatch, so the CLI cannot even attempt to edit someone else's repo.
- `protect set` requires **at least one** constraint: `build_protection_tag` assembles the
  rule words then round-trips them through `parse_protection_tag`, which rejects a
  pattern-only tag (`repos.rs:60-85`; `git_perms.rs:280`). Tested:
  `protection_set_requires_at_least_one_rule` (`repos.rs:521`).
- `protect set` is a **full replacement** for the exact ref pattern: every existing
  `buzz-protect` tag with the same pattern is dropped and the new one appended
  (`repos.rs:108-116`). Any constraint omitted from the command is therefore removed —
  stated in `crates/buzz-cli/README.md:80-82` and pinned by the `count == 1` assertion at
  `repos.rs:474-486`.
- Non-protection tags are **preserved verbatim** across the rewrite, including unknown
  future metadata; only `auth` and the matching `buzz-protect` are filtered
  (`repos.rs:103-106`). Tested for `buzz-channel` and a synthetic `future-metadata` tag at
  `repos.rs:456-465`.
- The update **fails closed** if the stored announcement already contains a malformed
  protection rule: `parse_protection_tags` over the full rebuilt tag set must succeed
  (`repos.rs:118-131`). Tested: `protection_update_rejects_malformed_existing_rules`
  (`repos.rs:526`).
- The 50-rules-per-repo ceiling (`git_perms.rs:19`, `MAX_PROTECTION_RULES`) is enforced
  transitively by the same `parse_protection_tags` call — the CLI never counts rules itself.
  Tested at the boundary: `protection_update_enforces_repository_rule_limit` builds 50 rules
  and asserts the 51st is rejected (`repos.rs:551-573`).
- `protect list` deliberately does **not** fail closed: it reports `unknown_rules` and a
  string `validation_error` so an owner can see and repair a broken rule
  (`repos.rs:141-171`). Two tests, `repos.rs:575` and `repos.rs:601`.
- `protect remove` validates the ref pattern (`RefPattern::parse`, `repos.rs:331-332`), then
  requires an existing rule for that exact pattern before writing (`repos.rs:334-338`) —
  removal is not idempotent by design.
- Ref-pattern grammar (must start with `refs/`, ≤ 256 chars, ≤ 3 wildcards, `**` only as the
  last segment, no partial globs like `v*`) lives in `git_perms.rs:80-125` and is reached
  from the CLI only through `parse_protection_tag`/`RefPattern::parse`.

#### PR / patch / issue state transitions

- Status words map to kinds through one shared function, `parse_status`
  (`patches.rs:194-205`): `open`→1630, `merged`|`resolved`→1631, `closed`→1632,
  `draft`→1633. `merged` and `resolved` are explicit synonyms for the same kind — the
  comment at `patches.rs:190-193` justifies this as NIP-34 using different words for patches
  vs issues over one status kind. Tested: `parse_status_accepts_known_words` (`patches.rs:304`),
  `parse_status_rejects_unknown_word` (`patches.rs:319`).
- clap narrows the accepted words **per command** before `parse_status` ever runs:
  `patches status` allows `open|merged|closed|draft` (`lib.rs:1263`), `pr status` the same
  (`lib.rs:1413`), `issues status` allows `open|resolved|closed|draft` (`lib.rs:1490`). So
  `patches status --status resolved` is rejected by clap even though `parse_status` accepts
  it. That divergence is a convention enforced only by the clap `value_parser` lists, not by
  any shared constant.
- `--repo-owner` and `--repo-id` must be given **together** on all three status commands.
  Enforced twice: clap `requires` (`lib.rs:1249-1254`, `lib.rs:1420-1425`, `lib.rs:1497-1502`)
  and a defensive `match` arm returning `Usage` (`patches.rs:154-158`, `pr.rs:192-196`,
  `issues.rs:121-125`). The `match` arm is unreachable through the CLI but protects the
  library entry points.
- Recipient assembly rule, replicated in all three: seed with the repo owner when known,
  then append each `--to` after `validate_hex64`, skipping duplicates
  (`patches.rs:170-178`, `pr.rs:186-194`, `issues.rs:127-135`). Note the seeded repo owner
  is **not** re-validated at that point (it was validated earlier at `patches.rs:150`),
  and the dedup is `Vec::contains` — O(n²) but n is tiny.
- `pr status` hardcodes `accepted_revision_root: None` and empty `applied_patches` /
  `applied_as_commits` (`pr.rs:199-207`), so the `--revision` and `--q` affordances that
  `patches status` exposes are simply unavailable for PRs. `issues status` additionally
  hardcodes `merge_commit: None` (`issues.rs:141`).
- `--revision`, `--q`, `--merge-commit`, `--applied-as-commit` are documented as
  "status=merged only" (`lib.rs:1256`, `:1273`, `:1276`, `:1279`) but **nothing enforces
  that**: `cmd_patch_status` passes them into `GitStatusMeta` regardless of the resolved
  status (`patches.rs:180-188`). A convention enforced only by help text.
- `--root` / `--root-revision` on `patches send` are two independent booleans
  (`patches.rs:37-38`) with no mutual-exclusion check in the CLI and no clap `conflicts_with`
  (`lib.rs:1215-1221`); whatever the SDK does with both set is not constrained here.
- `--committer` must be exactly four `|`-separated fields (`parse_committer`,
  `patches.rs:58-71`). Fields are **not** individually validated — the timestamp and tz
  offset are passed through as strings. Tested for arity only (`patches.rs:284`, `:298`).

#### Workflow trigger and approval rules

- `workflows create` generates a fresh v4 UUID client-side (`workflows.rs:107`) but prefers
  a relay-supplied `workflow_id` from a `response:{…}` message when present
  (`workflows.rs:111-112`, via `extract_relay_response_field`, `client.rs:1407-1418`).
- `workflows trigger --inputs` must parse as JSON **and** be an object
  (`workflows.rs:163-169`). When present, the SDK builder is bypassed and the event is
  hand-assembled with only a `d` tag (`workflows.rs:170-181`) — which means the 64 KiB
  `check_content` cap that `build_workflow_def` applies (`builders.rs:1468`) is **not**
  applied to `--inputs`. Uncapped input content on this branch.
- `workflows approve` sends `hex(SHA256(token))` as the `d` tag, never the raw token
  (`workflows.rs:205`). The comment at `workflows.rs:204` states the relay expects the hash.
  The raw token is validated as a UUID first (`workflows.rs:196`).
- `--approved` defaults to `true` with `ArgAction::Set` (`lib.rs:909`), so a bare
  `workflows approve --token X` grants. Grant vs deny selects 46030 vs 46031 inside the SDK
  (`builders.rs:1532-1536`).
- `workflows runs` clamps `--limit` to `min(limit.unwrap_or(20), 100)` (`workflows.rs:72`) —
  the only client-side limit clamp in the entire group.
- `workflows runs` is a **known-dead read path**: its own doc comment says the relay does not
  emit 46001-46003 (`workflows.rs:62-65`). Verified: `grep -rn '46001'` across `crates/`
  finds only kind-registry entries, exclusion comments
  (`crates/buzz-db/src/feed.rs:232`, `crates/buzz-workflow/src/lib.rs:266`), a name-mapping
  arm (`crates/buzz-relay/src/handlers/event.rs:48`), and this CLI query — no producer.
- `workflows get` filters on `#d` with **no `authors` constraint** (`workflows.rs:40-43`) and
  then takes `events.first()` (`workflows.rs:47`). Two different authors can publish kind
  30620 with the same `d` value; whichever the relay returns first wins. `workflows update`
  and `delete`, by contrast, are author-scoped implicitly — `build_workflow_delete` embeds
  the caller's own pubkey in the `a` coordinate (`workflows.rs:143`, `builders.rs:1502-1506`).
- `workflows update` requires `--channel` in addition to `--workflow` (`lib.rs:867-877`)
  because the `h` tag is re-emitted on every 30620 write (`builders.rs:1487-1490`).

#### Moderation action semantics

| Rule | Site |
|---|---|
| `--expires-in` is converted to an absolute unix second via `Timestamp::now() + secs`; `--expires-at` is used as-is; `--expires-in` wins if both were somehow set | `resolve_expiry`, `moderation.rs:26-32` |
| the two flags are mutually exclusive (clap only) | `lib.rs:1697`, `lib.rs:1723` — `conflicts_with = "expires_at"` on `expires_in` |
| `ban` with no expiry is permanent | `moderation.rs:39` passes `None` through |
| `timeout` **requires** an expiry — this is the one duration rule the CLI enforces itself | `moderation.rs:69-71` |
| `resolve --status` must be `resolved`\|`dismissed` and `--action` one of six words | **not** in `moderation.rs`; enforced in `buzz_sdk::build_moderation_resolve_report` (`builders.rs:1661-1676`). The CLI passes both strings through unvalidated (`moderation.rs:97`) |
| the community is the relay host; moderation commands carry no channel scope | module doc `moderation.rs:16-18`, mirrored in `lib.rs:1638-1643` |
| authorization (owner/admin) is entirely the relay's job | module doc `moderation.rs:6-8`; nothing in `moderation.rs` checks a local role |
| `reports --limit` / `audit --limit` default to 50 (`lib.rs:1663`, `lib.rs:1768`) and are `i64` with **no client-side validation** | `moderation.rs:105-117`, `moderation.rs:125-130`; the relay clamps to `1..=500` and maps non-positive to 500 (`crates/buzz-relay/src/api/bridge.rs:2084-2088`) |
| `--reason` on ban/timeout/resolve is uncapped and unchecked for control characters | `builders.rs:1607-1609` pushes it straight into a tag; contrast `check_reason`'s 64-byte + control-char rule applied to NIP-IA reasons (`builders.rs:1706-1719`) |

`resolve_expiry`'s `Timestamp::now().as_secs() + secs` (`moderation.rs:29`) is unchecked
`u64` addition. A large `--expires-in` overflows: panic in a debug build, silent wrap to a
tiny timestamp in release — which would produce an already-expired ban. Every other
timestamp arithmetic in the group uses `checked_add` (`repos.rs:134`) or
`saturating_add` (`engram.rs:590`). No test covers `resolve_expiry`.

#### Agent archive / draft rules

- Draft commands **require** `BUZZ_AUTH_TAG`: `require_owner` (`agents.rs:158-166`) returns
  `Auth` (exit 3) when absent. Archive/unarchive do not require it.
- Drafts are advisory only. The output injects `"saved": false` and a message stating nothing
  changes until the owner saves (`agents.rs:41-45`, `agents.rs:84-88`). Payload is NIP-44
  encrypted to the owner (`agent_management.rs:107-108`).
- Draft field caps: display name ≤ 120 chars, system prompt ≤ 20 000 chars, channel ≤ 128
  chars, other optional fields ≤ 300 chars — all measured in **chars, not bytes**, and all
  trimmed first (`agent_management.rs:11-12`, `:70-84`). Channel must parse as a UUID
  (`agent_management.rs:130-131`). `draft-update` requires at least one field to change
  (`agent_management.rs:170-179`); `--respond-to` must be `owner-only` or `anyone`
  (`agent_management.rs:147-153`). Tested: `update_requires_a_change`
  (`agent_management.rs:239`), `create_rejects_invalid_channel` (`agent_management.rs:261`).
- The NIP-OA `auth` tag on 9035/9036 is resolved by `resolve_auth` (`agents.rs:172-204`):
  self-path (`target == signer`, case-insensitive) → `None`; otherwise fetch the target's
  kind:0 and extract. Query or network failure is an **error**, not a silent downgrade
  (`agents.rs:181-183`) — the comment at `agents.rs:167-170` explains that degrading to bare
  would make the relay's rejection misleading.
- `extract_owner_auth_tag` (`agents.rs:206-248`) applies a **set-level** rule: there must be
  exactly one `auth`-labelled tag on the kind:0 (`agents.rs:216-219`). A structurally valid,
  owner-matching tag sitting next to a second `auth`-labelled tag — duplicate or malformed —
  yields `None` (bare). Owner match is case-insensitive; owner must be 64 hex, sig 128 hex.
  Fourteen tests, `agents.rs:387`-`agents.rs:503`, including the two discriminating
  set-level cases at `agents.rs:483` and `agents.rs:494`.
- Archive/unarchive use `sign_event_unchecked` (`agents.rs:113`, `agents.rs:142`), bypassing
  the "exactly one auth tag" invariant that `sign_event` enforces (`client.rs:597-609`).
  Rationale documented at `client.rs:735-742`: the 9035 `auth` tag is a content-level
  attestation about the *target*, not this client's membership delegation, so injecting
  `self.auth_tag` would either double up or drop the caller's attestation.
- The 13535 snapshot is a strict tri-state (`fetch_archived_snapshot`, `agents.rs:270-305`):
  no events → `Ok(vec![])`; a fully-verified event → `Ok(pubkeys)`; any trust failure →
  `Err`. Verification (`verify_archived_event`, `agents.rs:320-372`) requires, in order:
  kind == 13535; author == the NIP-11 `self` pubkey; **exactly one** NIP-70 `-` tag with
  arity exactly 1 (a valid `["-"]` alongside a malformed `["-","extra"]` poisons the
  snapshot — `agents.rs:335-352`, tested at `agents.rs:665`); then `event.verify()`.
  Non-hex or short `p` values are dropped silently rather than failing the whole snapshot
  (`agents.rs:353-366`, tested at `agents.rs:686` and `agents.rs:704`).
- The NIP-11 `self` field must be 64 hex and is lowercased before comparison
  (`normalize_relay_self_hex`, `agents.rs:250-258`) — without this an uppercase `self` would
  never match the always-lowercase event author. Tested at `agents.rs:527`, `:534`, `:539`,
  and end-to-end at `agents.rs:548`.
- `agents archived` treats a trust failure as fatal; the `--template` roster resolver in
  `channels.rs` fails *open* on the same failure. The split is documented at
  `agents.rs:255-269` and `agents.rs:306-309`, with the rationale living in
  `channels.rs:526`'s doc comment.

#### Users rules

- `--name` and `--pubkey` are mutually exclusive (`users.rs:16-20`).
- `--pubkey` accepts at most 200 values (`users.rs:35-37`). **`users presence` has no such
  cap** — it splits an arbitrary CSV and sends every entry as an author (`users.rs:248-256`).
- `--name` must be non-blank after trim (`users.rs:85-87`); the relay's NIP-50 result set is
  additionally narrowed client-side by a case-insensitive substring match on `display_name`
  or `name` (`users.rs:120-134`). The doc comment at `users.rs:78-79` states an empty array is
  returned when the relay does not implement NIP-50.
- `users set-profile` requires at least one field (`users.rs:157-161`) and is
  read-merge-write: caller values win, absent ones fall back to the current kind:0 content,
  with `display_name` falling back to `name` (`users.rs:167-179`). The `name` (username)
  field is hardcoded to `None` on write (`users.rs:200`), so a merge that promoted `name` to
  `display_name` silently drops the original `name`.
- `users presence` prefers a `p`-tag subject over the event author (`presence_subject`,
  `users.rs:279-289`), because the relay may synthesize relay-authored presence snapshots.
  Three tests at `users.rs:343`, `:349`, `:355`.
- Kind 20001 is ephemeral, so `set-presence` must go over WebSocket, not `POST /events`
  (`users.rs:292-296` doc, `users.rs:301`).

#### Pack rules

`pack validate` (`pack.rs:15-49`): path must exist and be a directory (`pack.rs:17-22`);
diagnostics print to stderr; errors → `Usage` (exit 1); warnings only → exit 0 with
`Valid (with warnings).`; clean → `Valid.`. The doc comment at `pack.rs:11-13` matches the
implementation. `pack inspect` (`pack.rs:52-150`) applies the same path checks then
`resolve_pack`, and truncates the system-prompt preview at 77 **chars** with an ellipsis
(`pack.rs:143-148`) while reporting the length in **bytes** (`pack.rs:150`) — a mixed unit
in one output line.

#### Pagination and limit defaults

The group sets almost no limits of its own; effective bounds come from the relay.

| Command | Client limit | Server-side outcome |
|---|---|---|
| `mem ls` | `5000` (`mem.rs:201`) | clamped to `MAX_HISTORICAL_LIMIT` 2000 (`crates/buzz-relay/src/handlers/req.rs:25`, applied `:879-882`), then to 1000 by `max_limit.unwrap_or(1000)` in `crates/buzz-db/src/event.rs:346-347`. Effective cap ≈ 1000 rows, silently |
| `mem get`/`hash`/`patch` | `16` (`mem.rs:155`) | fine |
| `repos get`, `workflows list`, `workflows get`, `pr get`/`list`, `patches get`/`list`, `issues get`/`list` (without `--limit`) | none | relay default = `MAX_HISTORICAL_LIMIT` (2000) then DB clamp 1000 |
| `repos list`, `pr list`, `patches list`, `issues list` with `--limit` | passed through, unvalidated | relay clamps |
| `workflows runs` | `min(limit.unwrap_or(20), 100)` (`workflows.rs:72`) | fine |
| `users get` | `authors.len()` (`users.rs:44`) | fine |
| `users get --name` | `100` (`users.rs:93`) | fine |
| `users presence` | `pubkeys.len()` (`users.rs:260`) — `0` when the CSV is blank, which yields an empty result rather than an unbounded scan | fine |
| `moderation reports`/`audit` | 50 default (`lib.rs:1663`, `:1768`) | relay clamps `1..=500`; a non-positive value becomes 500 |

No command in this group uses `BuzzClient::query_paginated` or `query_all`
(`client.rs:715-729`): `grep -n 'query_paginated\|query_all' ` across the eleven files
returns zero matches. Every read is a single unpaginated `POST /query`, so a result set
larger than the server clamp is silently truncated with no signal to the caller. `mem ls` is
the clearest exposure: it *asks* for 5000 and can receive at most about 1000.
