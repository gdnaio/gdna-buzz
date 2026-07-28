## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Business Rules

#### Framing

- Transport is newline-delimited JSON over the child's stdio (`acp.rs:1-2`).
- Reads go through `FramedRead::new(stdout, LinesCodec::new_with_max_length(MAX_LINE_SIZE))` (`acp.rs:487`), `MAX_LINE_SIZE = 10_000_000` (10 MB, `acp.rs:21`).
- A line exceeding the cap surfaces as `LinesCodecError::MaxLineLengthExceeded` and is mapped to `AcpError::Protocol("agent stdout line exceeded 10MB limit")` (`acp.rs:1082-1086`, `acp.rs:1373-1381`) — the turn dies; the codec does not attempt resynchronisation.
- Writes are one `to_string` + `\n` + `flush`, wrapped in a fixed 30 s `WRITE_TIMEOUT` (`acp.rs:952-963`); expiry is `AcpError::WriteTimeout`.
- Empty/whitespace-only lines are skipped without resetting anything (`acp.rs:1097-1099`, `acp.rs:1391-1393`).
- Unparseable lines emit an `acp_parse_error` observer event, log a warning, and are skipped — they do **not** fail the turn (`acp.rs:1103-1118`, `acp.rs:1397-1412`).

#### Request/response correlation

- Ids are harness-generated `u64`s from a monotonic `next_id` (`acp.rs:149-150`, incremented at `acp.rs:981`, `acp.rs:690`, `acp.rs:1336`).
- A frame counts as *our* response only when `msg["id"] == json!(expected_id)` **and** `msg.get("method").is_none()` (`acp.rs:1123-1130`, `acp.rs:1444-1447`). The `method` guard exists so an agent-initiated request whose id happens to collide is not mistaken for a response (`acp.rs:1120-1122`).
- Comparison is done on `serde_json::Value`, so a string-typed id from the agent would not match a numeric expectation (`acp.rs:1072-1073`).
- **Non-matching ids are silently skipped** — the loop simply continues. Late responses from a previously timed-out request are therefore consumed and discarded on the next read (documented at `acp.rs:1010-1015`).
- Notifications are distinguished purely by absence of `id` on the outbound side (`acp.rs:1049-1057`) and by the presence of `method` on the inbound side (`acp.rs:1136`).
- Steer-response routing is checked **before** the prompt-response check, both under the same `no method` guard, disambiguating on id (`acp.rs:1439-1447`).

#### Timeouts

| Bound | Value | Site |
|---|---|---|
| Write | 30 s | `acp.rs:952` |
| Non-prompt request (read phase) | `REQUEST_TIMEOUT = 60 s` | `acp.rs:968`, applied `acp.rs:1000-1003` |
| Non-prompt request (write phase) | reuses the same 60 s wrapper | `acp.rs:996-999` — worst case ≈ 90 s, stated at `acp.rs:974-976` |
| Prompt idle | `Config::idle_timeout_secs`, reset on every valid JSON line | `acp.rs:1416-1418` |
| Prompt hard cap | `Config::max_turn_duration_secs`, absolute `Instant` | `acp.rs:681-682` |
| Cancel drain idle | fixed 30 s | `acp.rs:932-936` |
| Cancel-drain floor | 30 s grace when the inherited deadline is expired/near | `acp.rs:840-861` |
| Child reap after SIGKILL | 5 s | `acp.rs:394` |

The idle clock resets on **any** valid JSON frame (`acp.rs:1416-1418`) and is reset a second time on `session/update` variants that report a tool call starting (`acp.rs:1483-1490`) — `handle_session_update` returns `true` only for `tool_call` (`acp.rs:1550`).

Classification is decided *before* sleeping: `idle_fires_first = idle_deadline < hard_deadline` (`acp.rs:1247`), so scheduler jitter cannot mislabel which timeout fired (`acp.rs:1245-1246`).

A pre-select deadline check runs at the top of every loop iteration (`acp.rs:1256-1274`). The comment at `acp.rs:1257-1262` explains why: `tokio::select!` uses `biased` with the reader arm first (`acp.rs:1276-1278`), so an agent producing a continuous output stream would win every poll and `sleep_until` would never be reached, silently defeating the hard cap.

#### Cancellation

`cancel_with_cleanup_until` (`acp.rs:897-946`) is the single drain implementation:

1. **Precondition first** — `last_prompt_id.take()` must be `Some`, else `AcpError::Protocol("cancel_with_cleanup called with no in-flight prompt")` (`acp.rs:903-905`). Validated before any write so no stray frames reach an idle agent (`acp.rs:899-902`).
2. If a permission request is pending and `permission_responded == false`, write `outcome: "cancelled"` (`acp.rs:909-919`).
3. Send the `session/cancel` notification (`acp.rs:922`).
4. Drain with 30 s idle + the inherited hard deadline until the original prompt id responds (`acp.rs:937-945`).

Deadline inheritance rules (`acp.rs:840-861`): a stored deadline more than 30 s out is used as-is; otherwise a 30 s floor applies (with a debug log); a missing deadline logs a **warning** and uses the same 30 s fallback. `session_prompt_blocks_with_idle_timeout` deliberately leaves `last_prompt_id` and `current_hard_deadline` set on `IdleTimeout`/`HardTimeout` so the drain can inherit them, and clears them on every other outcome (`acp.rs:722-737`).

`cancel_with_cleanup_grace` (`acp.rs:881-895`) discards the stored deadline, uses a caller-supplied grace window, and remaps `HardTimeout` → `CancelDrainTimeout` (`acp.rs:891-892`) so a Stop-button drain expiry is distinguishable from a genuine configured hard-cap breach (`acp.rs:874-879`).

#### Permission auto-approval

`handle_permission_request` (`acp.rs:1680-1755`):

- Missing `id` or missing `params.options` → `AcpError::Protocol` (`acp.rs:1682-1685`, `acp.rs:1692-1694`).
- The option is found **by `kind`, never by hardcoded `optionId`** (`acp.rs:1676`, `acp.rs:1703-1705`). Preference order: `allow_once`, then `reject_once` (`acp.rs:1720-1722`).
- Neither present → `AcpError::Protocol("no suitable permission option found …")` (`acp.rs:1728-1731`).
- On the `reject_once` fallback, a missing `optionId` degrades to the literal `"reject"` (`acp.rs:1725`) rather than erroring, unlike the `allow_once` path which requires it (`acp.rs:1707-1709`).
- Ordering is write-then-flag (`acp.rs:1749-1751`). The comment at `acp.rs:1735-1748` records why: flag-before-write could leave `permission_responded == true` after a failed write, making the cancel path skip the cancelled outcome and deadlocking the agent forever. The residual double-response window is one memory store wide.

#### Capability probing and latching

- `initialize` requests `protocolVersion: 2` unconditionally — an intentional squat ahead of the upstream ACP RFD (`acp.rs:536-537`, `acp.rs:126`).
- The negotiated version is read by the caller as `init_result["protocolVersion"].as_u64().unwrap_or(1) as u32` (`lib.rs:3776`, `lib.rs:3864`). A missing or non-numeric field therefore silently means **legacy v1**, which changes prompt composition (`session_new_system_prompt` branches on it at `pool.rs:829-833`; base-prompt/heartbeat prefixing at `lib.rs:4197-4210`). `acp.rs` itself neither validates nor stores the version.
- Goose system-prompt support is latched negatively: `pool.rs:849` matches `Err(AcpError::AgentError { code: -32601, .. })` and records `goose_system_prompt_supported = Some(false)`, after which `pool.rs:838` skips the probe for the rest of the process. The code path that makes this possible is the `AgentError` code preservation in `agent_error_from_json` (`acp.rs:115-122`).
- `session/set_model` is the unstable path; `resolve_model_switch_method` prefers the stable `configOptions` route and only falls back to `SetModel` (`acp.rs:1873-1919`). The fresh `session/new` response is treated as authoritative (`acp.rs:1873-1874`).
- `model_in_catalog` (`acp.rs:1922-1951`) mirrors the same match against the two pre-extracted catalog halves for the idle-path pre-cancel guard.

#### Steer (goose-native, non-cancelling)

- At most one steer in flight: the select arm is gated on `pending_steer.is_none()` (`acp.rs:1288`).
- `active_run_id == None` at write time → nothing is written; the request is acked `SteerError::ExpectedRunIdMissing` and the caller falls back to cancel+merge (`acp.rs:1319-1333`).
- `expectedRunId` is read at write time, not snapshotted at dispatch, so it matches goose's current run (`acp.rs:1300-1310`).
- A successful steer **renews** the hard deadline to `now + max_duration` if that is later than the current one (`acp.rs:1465-1474`).
- Every early-return path drains `pending_steer` with `SteerAck::PromptCompletedNeutral` so the oneshot is never leaked: pre-select expiry (`acp.rs:1263-1269`), sleep arm (`acp.rs:1362-1364`), `AgentExited` (`acp.rs:1369-1371`), max-line (`acp.rs:1374-1376`), IO error (`acp.rs:1382-1384`), prompt error (`acp.rs:1449-1452`), prompt success (`acp.rs:1454-1457`).
- `install_steer_rx` **panics** if a receiver is already installed (`acp.rs:801-805`); `clear_steer_rx` is called on every exit path of `run_prompt_task` to keep that invariant (`acp.rs:811-814`, `pool.rs:1243`).

#### `session/new` and stop-reason mapping

- `session_new_full` requires `result.sessionId` to be a string, else `AcpError::Protocol("session/new response missing sessionId")` (`acp.rs:576-579`).
- `systemPrompt` is added to the params only when `Some`; the doc comment relies on JSON-RPC's ignore-unknown-fields behaviour for agents that don't support it (`acp.rs:559-561`, `acp.rs:572-574`).
- `parse_stop_reason` (`acp.rs:1758-1765`): missing `stopReason` → `Protocol("session/prompt response missing stopReason")`; unrecognised value → `Protocol("unknown stopReason: …")`. Parsing is case-insensitive (`acp.rs:62-64`).

#### `CODEX_CONFIG` merge contract

`build_codex_config_env` (`acp.rs:257-345`), gated entirely on `has_generated_codex_config`:

1. `false` → returns `Ok(None)` immediately; any persona `CODEX_CONFIG` falls through to plain operator-wins (`acp.rs:262-265`).
2. Collect all `CODEX_CONFIG` entries from `extra_env` in order (`acp.rs:268-273`); empty → `Ok(None)` rather than a panic (`acp.rs:275-279`).
3. Every entry must parse as a JSON **object**; otherwise `AcpError::Protocol`, with the message distinguishing `persona` (index 0) from `generated` (`acp.rs:284-302`).
4. First entry is the base; the rest are `deep_merge`d in (`acp.rs:305-309`).
5. `parent_codex_config` (the parent process's `CODEX_CONFIG`) is deep-merged last-but-one, so **parent wins on every colliding key at every nesting level** (`acp.rs:311-327`).
6. `sandbox_workspace_write.network_access` is **force-set to `true`** as the final step and always wins (`acp.rs:329-343`). The entry is created if absent (`acp.rs:330-332`); a non-object `sandbox_workspace_write` errors (`acp.rs:335-341`).

`deep_merge` (`acp.rs:208-227`) recurses only when both sides hold objects; scalars, arrays, and type mismatches let overlay win outright (`acp.rs:220-224`).

`codex_network_env` (`config.rs:646-677`) only fires for normalized commands `codex` / `codex-acp` (`config.rs:647-650`) and only when the relay URL parses **and** has a host — otherwise it logs a warning and skips injection rather than widening the sandbox (`config.rs:652-670`). The extracted `host` is logged (`config.rs:672`) but not used in the emitted value, which is the constant `{"sandbox_workspace_write":{"network_access":true}}` (`config.rs:674-677`).

#### Env injection precedence at spawn

`spawn` (`acp.rs:449-461`): for each `(key, value)` in `extra_env`, inject only `if std::env::var(key).is_err()` — **the parent environment wins** (`acp.rs:455-457`). `CODEX_CONFIG` is excluded from this loop when the merge path produced a value and is set unconditionally afterwards (`acp.rs:451-453`, `acp.rs:459-461`).

#### Agent command / args normalization

- `normalize_agent_command_identity` (`config.rs:600-615`): trim, `\` → `/`, strip trailing `/`, take basename, lowercase, strip `.exe`, then map space and `_` to `-`.
- `default_agent_args` (`config.rs:617-624`): `goose` → `["acp"]`; `codex`, `codex-acp`, `claude-agent-acp`, `claude-code-acp`, `claude-code`, `claudecode`, `buzz-agent` → `[]`; anything else → `None` (no normalization).
- `normalize_agent_args` (`config.rs:679-706`): trims and drops empty args; empty result takes the default; and a lone case-insensitive `"acp"` is treated as "no args" for zero-arg providers so legacy desktop launches behave the same as env-based launches (`config.rs:695-700`).

#### Config validation and precedence — `Config::from_args` (`config.rs:740-1009`)

Executed in this order:

| Rule | Behaviour | Line |
|---|---|---|
| Key parse | `Keys::parse(&args.private_key)`, error → `ConfigError::KeyParse` | `config.rs:741` |
| Key scrub | raw string overwritten with `0`s then cleared — best-effort, explicitly not a `zeroize` guarantee | `config.rs:742-748` |
| System prompt | `--system-prompt` wins over `--system-prompt-file`; clap also marks them mutually exclusive | `config.rs:750-756` |
| Heartbeat interval | `>0 && <10` → **error** | `config.rs:758-762` |
| Turn liveness | `>0 && <5` → **error** | `config.rs:764-768` |
| Heartbeat prompt | inline wins over file | `config.rs:770-776` |
| Base prompt file | read only when `!no_base_prompt`; `>1_048_576` bytes → **error** | `config.rs:778-791` |
| Config-mode warnings | `--kinds`, `--channels`, `--no-mention-filter` each log "ignored in config mode" | `config.rs:793-804` |
| Empty agent command | trimmed-empty → **error** | `config.rs:808-812` |
| Channel UUIDs | non-UUID entries **warn and are ignored**, never error | `config.rs:816-824` |
| Heartbeat cap | `>86400` → warn and clamp to `86400` | `config.rs:826-834` |
| Liveness cap | `>86400` → warn and clamp to `86400` | `config.rs:836-845` |
| Idle-timeout ladder | see below | `config.rs:849-874` |
| Max turn duration | `0` → warn + clamp to `60`; `>604_800` → **error** | `config.rs:876-890` |
| Ordering invariant | `idle_timeout_secs >= max_turn_duration_secs` → **error** | `config.rs:894-899` |
| Allowlist | `allowlist` mode with empty list → **error** (`:903-907`); entries validated (`:908`); list supplied in another mode → **warn and discard** (`:909-915`) | `config.rs:901-916` |
| `allowed_respond_to` | each entry must parse as a `RespondTo`, else **error**; non-empty list not containing the active mode → **error** | `config.rs:918-941` |
| Codex env | `codex_network_env` pushes into `persona_env_vars` and sets `has_generated_codex_config` | `config.rs:951-957` |
| Handling/dedup | `validate_multiple_event_handling` | `config.rs:959` |

The idle-timeout ladder (`config.rs:849-874`) resolves `--idle-timeout` > `--turn-timeout` (deprecated) > `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`):

- both set → `--idle-timeout` wins, deprecation warning logged (`config.rs:851-857`)
- only `--turn-timeout` → used, deprecation warning logged (`config.rs:859-864`)
- neither → `900`
- a resolved value of `0` → warn and clamp to `1` (`config.rs:868-873`)

The strict-less-than rule (`config.rs:894-899`) is justified in the comment above it: with `idle >= max`, the wall-clock cap always fires first and the idle timeout becomes a dead letter. With the shipped defaults (900 vs 7200) the invariant holds.

`validate_multiple_event_handling` (`config.rs:579-597`) rejects `Steer`/`Interrupt`/`OwnerInterrupt` combined with `DedupMode::Drop`, because `Drop` discards events during the cancel drain window and would yield incomplete merged prompts.

`validate_allowlist` (`config.rs:558-572`) trims + lowercases each entry, requires exactly 64 ASCII hex chars, errors otherwise, and dedupes via `HashSet`.

`allowed_respond_to` is validated at startup but is otherwise **display-only** — the stored `Vec<String>` (`config.rs:532`) is read nowhere except `summary()` (`config.rs:1019-1025`, emitted at `config.rs:1049`). No runtime enforcement re-checks it.

#### Legacy env-var propagation

`propagate_legacy_env_vars` (`config.rs:715-726`) maps `BUZZ_ACP_PRIVATE_KEY` → `BUZZ_PRIVATE_KEY` and `BUZZ_ACP_API_TOKEN` → `BUZZ_API_TOKEN`, and only when the canonical name is unset (`config.rs:720`). It uses `std::env::set_var` and must run before the tokio runtime starts because Rust 2024 requires a single-threaded process for that call (`config.rs:708-714`); it is invoked from the sync wrapper at `lib.rs:1234`. `Config::from_cli` explicitly does **not** call it (`config.rs:730-732`).

`BUZZ_API_TOKEN` is written by this function and then never read anywhere in the crate — the only other occurrence of the name in the repo's buzz-acp tree is the README table row (`crates/buzz-acp/README.md:107`).

#### Subscription filter construction

`resolve_channel_filters` (`config.rs:1134-1234`):

- Target set = `channels_override` ∩ `discovered_channels` when the override is set (non-UUID strings dropped by `filter_map`), else all discovered channels (`config.rs:1143-1153`).
- `Mentions`: `kinds = kinds_override` or the default triple `[KIND_STREAM_MESSAGE, KIND_WORKFLOW_APPROVAL_REQUESTED, KIND_STREAM_REMINDER]`; `require_mention = !no_mention_filter` (`config.rs:1158-1174`).
- `All`: `kinds = config.kinds_override.clone()` — **`None` when `--kinds` is absent** — and `require_mention = false` (`config.rs:1175-1185`).
- `Config`: per channel, union the `kinds` of every applicable rule; **any rule with an empty `kinds` list collapses the merge to `None` (wildcard)**; `require_mention` is the most permissive across matching rules; channels matched by no rule are omitted entirely (`config.rs:1186-1231`).

`resolve_dynamic_channel_filter` (`config.rs:1236-1314`) repeats the same three branches for a single newly-discovered channel, with `channels_override` enforced in Mentions/All and ignored in Config mode per the CLI contract (`config.rs:1246-1258`); `All` again yields `kinds: config.kinds_override.clone()` (`config.rs:1271-1274`).

`rule_applies_to_channel` (`config.rs:1316-1325`) matches `ChannelScope::All("all")` or a `List` containing the channel UUID; anything else is a non-match.

Consequence: `--subscribe all` without `--kinds` produces `ChannelFilter { kinds: None, .. }`, and Config mode does the same for any rule with an empty `kinds` array. Per `AGENTS.md § Common Gotchas #2`, a relay REQ with no `kinds` trips the p-gate and returns 403. Neither branch warns about the wildcard; the only `kinds`-related warning in the whole file is the "ignored in config mode" notice (`config.rs:794-796`).

#### TOML rule loading — `load_rules` (`config.rs:1060-1132`)

| Rule | Behaviour | Line |
|---|---|---|
| File read | missing file → `ConfigError::Io` | `config.rs:1064` |
| Parse | TOML error → `ConfigFile` | `config.rs:1065-1066` |
| Rule count | `>100` → **error** | `config.rs:1068-1073` |
| Zero rules | **warn** only: "agent will receive no events in Config mode" | `config.rs:1075-1080` |
| Empty name | **error** | `config.rs:1084-1088` |
| Duplicate name | **error** | `config.rs:1089-1094` |
| Filter length | `>4096` bytes → **error** | `config.rs:1096-1103` |
| Filter syntax | `evalexpr::build_operator_tree` at load time; failure → **error**; success cached in `compiled_filter` | `config.rs:1104-1116` |
| Channel scope | `ChannelScope::All(s)` where `s != "all"` → **error** (catches `"ALL"`, `"All"`) | `config.rs:1118-1125` |
| Counter reset | `consecutive_timeouts` explicitly reset to 0 after deserialization | `config.rs:1127-1128` |

#### Setup mode

`SetupPayload::from_raw_env_value` (`setup_mode.rs:226-234`): `None` or empty string → `Ok(None)` (normal startup); non-empty invalid JSON → `Err`. When it returns `Some`, `lib.rs:1290-1295` returns straight into `run_setup_listener` and the agent pool never starts.

Per-event nudge gates, in the order the loop applies them (`setup_mode.rs:388-478`):

1. Membership kinds are handled and `continue`d (`setup_mode.rs:405-411`).
2. Kinds other than `KIND_STREAM_MESSAGE` / `KIND_WORKFLOW_APPROVAL_REQUESTED` are dropped (`setup_mode.rs:414-416`).
3. `ignore_self` by pubkey equality (`setup_mode.rs:419-421`).
4. `event_mentions_agent` — an explicit @mention is **required even when `subscribe_mode = all`** (`setup_mode.rs:424-426`).
5. `author_allowed` with the real config's `respond_to` + allowlist, plus DM hardening (`setup_mode.rs:430-442`).
6. `filter::match_event` against the setup rules (`setup_mode.rs:444-451`).
7. `should_nudge_for_event` — author verdict, then filter verdict, then event-id dedup insert (`setup_mode.rs:495-509`).

`should_nudge_for_event` returns early on a blocked author **before** touching the dedup set (`setup_mode.rs:501-504`), so a blocked event leaves no phantom entry — asserted by `test_non_allowlisted_author_returns_no_nudge` (`setup_mode.rs:673-690`).

`build_setup_subscription_rules` (`setup_mode.rs:521-542`) uses a single hardcoded mentions rule for Mentions/All, and in Config mode loads the real rules but falls back to the mentions rule with a warning if `load_rules` fails (`setup_mode.rs:530-539`). Its default kinds are only `[KIND_STREAM_MESSAGE, KIND_WORKFLOW_APPROVAL_REQUESTED]` (`setup_mode.rs:526`) — no `KIND_STREAM_REMINDER`, unlike `resolve_channel_filters` (`config.rs:1161-1165`).

Nudge threading (`publish_setup_nudge`, `setup_mode.rs:595-646`): if the trigger carries a NIP-10 root, both `root_event_id` and `parent_event_id` point at that root (a flat reply); otherwise both point at the triggering event (`setup_mode.rs:608-624`). The asker is p-tagged (`setup_mode.rs:632`).

Nudge copy selection in `nudge_body` (`setup_mode.rs:243-303`): empty requirements → generic "needs configuration" prose (`setup_mode.rs:245-249`); footer is chosen by requirement mix — Git-Bash present, all-external (`CliConfigInvalid`), mixed, or all-Buzz-managed (`setup_mode.rs:271-289`). The structured payload is always appended as a fenced `buzz:config-nudge` block for the desktop card (`setup_mode.rs:296-302`); `serde_json::to_string` there uses `.expect(...)` guarded by a SAFETY comment (`setup_mode.rs:294-296`).
