## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Security

#### Private-key handling

The agent's Nostr secret key lives in `Config.keys` (`nostr::Keys`) for the whole process. `lib.rs` clones it into six places: `presence_keys` (`lib.rs:1367`), the observer publisher tuple (`lib.rs:1414`), `PromptContext.agent_keys` (`lib.rs:1567`), the observer-control decrypt path (`lib.rs:2199`), and `HarnessRelay::connect` / `RestClient` for signing.

Exactly one site materialises the raw `nsec1…`:

```
lib.rs:4159-4170   EnvVar { name: "BUZZ_PRIVATE_KEY",
                            value: config.keys.secret_key().to_bech32()
                                     .expect("secret key bech32 encoding should never fail") }
```

**Transport is the agent's stdin pipe, not argv and not the child's environment block.** `build_mcp_servers` (`lib.rs:4141-4184`) returns an `McpServer` whose `env` vector becomes the `mcpServers` field of the `session/new` JSON-RPC request (`pool.rs:832` → `acp.rs:566-571`). So the key is not visible in `ps`/`/proc/*/cmdline` for the ACP child, and it is the *agent* — not the harness — that materialises it into the MCP server's real process environment.

Two exposure paths do exist:

1. **Observer feed carries the key verbatim.** `acp.rs:963` emits `self.observe("acp_write", value.clone())` for *every* outgoing JSON-RPC frame, with no key filtering or redaction. `session/new` is such a frame. Consequences:
   - the in-process observer ring buffer (`observer.rs:18`, cap 1,000 events) holds the plaintext `nsec1…` in memory for the process lifetime;
   - when `--relay-observer` is on, `publish_relay_observer_event` (`lib.rs:790-833`) NIP-44-encrypts that frame to the owner pubkey and publishes it as kind 24200 to the relay. The relay therefore stores a ciphertext blob that decrypts to the agent's own secret key, retrievable by anyone who later holds the owner's key. Nothing in `lib.rs` strips or masks `mcpServers[].env` before this call, and `grep -n redact crates/buzz-acp/src/acp.rs` finds no redaction helper anywhere in the crate.
2. **Environment inheritance.** `AcpClient::spawn` inherits the parent environment and only injects `extra_env` keys not already present (`acp.rs:456-461`). The harness's own `BUZZ_PRIVATE_KEY` / `BUZZ_RELAY_URL` / `BUZZ_AUTH_TAG` are therefore inherited by every agent subprocess implicitly, regardless of the `mcpServers` injection. Any agent-executed shell tool can read them from `/proc/self/environ`.

The key is **not** logged: `config.summary()` (`config.rs:1012-1040`) prints `relay`, `pubkey` hex, commands, timeouts, and gate modes only, and no `tracing` call in `lib.rs` touches `secret_key()`.

#### Subprocess spawning and argument construction

- Command and args come from `config.agent_command` / `config.agent_args` (`lib.rs:1762-1763`, `3486-3487`, `3664-3665`, `3731-3732`), i.e. from `BUZZ_ACP_AGENT_COMMAND` / `BUZZ_ACP_AGENT_ARGS`. Args are a `Vec<String>` passed to `Command::args` — no shell, so no injection via arg values. `BUZZ_ACP_AGENT_ARGS` splits on commas (README), which means an arg containing a comma cannot be expressed.
- No agent-supplied or event-derived data ever reaches the command line. `spawn_and_init` takes `&str` / `&[String]` owned from config only (`lib.rs:3848-3855`).
- `cmd.process_group(0)` on Unix (`acp.rs:474-476`) isolates the child so a SIGKILL of the agent group cannot propagate to the harness.
- `kill_on_drop(true)` (`acp.rs:426`) with the comment that callers must still `shutdown().await`. The four-stage shutdown (`lib.rs:2605-2672`) exists to make `Drop` the fallback, not the primary reaper.
- `stderr` is `Stdio::inherit()` (`acp.rs:423`) — agent stderr goes straight to the harness's stderr, unfiltered. Any secret the agent prints to stderr lands in the harness's log stream.
- `MCP command` is used to derive the server name via `Path::file_stem()` (`lib.rs:4146-4150`) with an `unwrap_or("mcp")` fallback; no path validation, no existence check, no allowlist.

#### Author gate — the primary authorization boundary

`author_allowed` (`lib.rs:235-256`) is the only thing standing between an arbitrary relay event and a prompt to the agent. Design properties that hold:

| Property | Evidence |
|---|---|
| Default is `owner-only` | `config.rs:444` default; README documents it |
| No owner ⇒ `is_owner_or_sibling` returns `false` | `lib.rs:198-201` |
| DM channels ignore both the allowlist and `anyone` | `lib.rs:242-247`; rationale `lib.rs:220-233` |
| Unknown channel type ⇒ treated as DM | `lib.rs:280-285` |
| Sibling claims are cryptographically verified locally, not trusted from the relay | `lib.rs:322-323`, `lib.rs:344-357` |
| Sibling lookup timeout fails closed | `lib.rs:310-315` (2,000 ms) |
| `Nobody` drops even the owner | `lib.rs:243`, test `lib.rs:4551-4565` |

Weak points:

- **Owner is resolved once, never re-resolved.** `OwnerCache.pubkey` has no setter (`lib.rs:161-163`). A transient REST/verification failure at startup permanently degrades an `owner-only` agent to dropping all traffic (warned at `lib.rs:1379-1384`), and permanently disables `--relay-observer` (`lib.rs:1421-1425`).
- **`BUZZ_AUTH_TAG` verification failure silently downgrades to the unverified flag.** `resolve_agent_owner` (`lib.rs:138-141`) warns and falls through to `config.agent_owner`, so a tampered attestation does not fail the boot.
- **DM classification depends on an HTTP fetch.** `is_dm_channel` fails closed, so a bridge outage silently collapses `allowlist`/`anyone` to owner-only. Safe, but a silent availability failure mode rather than a loud one.
- **Sibling cache is cleared wholesale at 256 entries** (`lib.rs:183-186`), so an attacker who can generate profile queries can force repeated 2 s REST round-trips on the gate path — a cheap latency amplifier against the main loop's event handling, since `author_allowed` is `await`ed inline in the `select!` arm (`lib.rs:2151-2159`).
- **`--respond-to anyone` is a documented, supported mode** (README) that removes the gate entirely outside DMs.

#### Owner control commands — plaintext text protocol

`!shutdown`, `!cancel`, `!rotate` are matched on `event.content.trim()` of an unencrypted kind-9 message that `p`-tags the agent (`is_owner_control_command`, `lib.rs:2718-2727`). Authorization is `event.pubkey.to_hex() == *owner` (`lib.rs:2042`, `2072`, `2110`), i.e. it relies entirely on the relay having verified the signature — `lib.rs` does not call `verify_event` on these, unlike the observer control path.

`!shutdown` from the owner terminates the harness (`lib.rs:2042-2049`). It is a plaintext, replayable string in channel history: anyone who can get the owner's client to echo `!shutdown` in a channel the agent is in, or who can replay an old owner message, kills the agent. There is no freshness window on this path — the ±300 s guard exists only for encrypted observer control frames (`lib.rs:861-869`).

Non-owner senders of these strings fall through to normal prompt handling (`lib.rs:2056-2058`, `2090-2091`, `2130-2131`), so `!shutdown` from a stranger is delivered to the agent as content.

#### Observer control frames — properly hardened

`handle_relay_observer_control_event` (`lib.rs:837-893`) is the strongest gate in the file, with three defence-in-depth checks before decryption:

1. `buzz_core::verify_event(&event)` — signature re-verified despite the relay having checked (`lib.rs:844-848`);
2. sender must equal the resolved owner hex (`lib.rs:850-858`);
3. `created_at` within ±`OBSERVER_CONTROL_FRESHNESS_SECS` = 300 s (`lib.rs:860-869`).

Payload is then NIP-44-decrypted (`lib.rs:871-877`) and dispatched on `type`; unknown types are dropped (`lib.rs:888-891`). If no owner is resolved, control frames are dropped with a warning (`lib.rs:2197-2202`).

#### Prompt-injection surface

Every accepted relay event's content is rendered into the agent's prompt by `queue::format_prompt` / `format_event_block`, and the same rendering is reused for native steer bodies (`lib.rs:2826-2831`). `lib.rs` performs **no content sanitisation** — it only decides *whether* an event is admitted (author gate + rule match), never *what* it may say.

Consequences within `lib.rs`'s scope:

- Under `--respond-to anyone`, any relay participant controls arbitrary text inside the agent's prompt, including the `[Buzz event: …]` framing region.
- The steer body is assembled by `format!` from harness framing plus attacker-controllable event content (`lib.rs:2829-2831`), so a crafted message can emit text resembling the `header`/`closing` framing strings.
- The default `permission_mode` is `BypassPermissions` in both test configs (`lib.rs:4984`, `lib.rs:5145`) — combined with an MCP server exposing shell/file tools, an injected instruction is executed without a permission prompt.

`--respond-to owner-only` (the shipped default) reduces this to owner-and-sibling-controlled input.

#### Secret redaction — absent

No redaction anywhere in the crate. `grep -n 'redact' crates/buzz-acp/src/acp.rs` → no matches. The observer pipeline's only content transformation is size-driven elision (`fit_observer_event_to_budget`, `lib.rs:659-687`), which is length-based and keyword-blind — a 3,000-byte head retention (`OBSERVER_LEAF_RETAIN_BYTES`, `lib.rs:632`) preserves the beginning of any leaf, and an `nsec1…` is 63 characters, so it survives elision intact.

#### Other observations

- `#![deny(unsafe_code)]` at `lib.rs:1`; zero `unsafe` blocks.
- 👀 reaction cleanup after membership removal is expected to 403 on non-open channels because the relay revokes membership before emitting the notification — the comment at `lib.rs:2000-2006` accepts stale reactions and assigns the fix to the relay.
- The `expect` at `lib.rs:4167` panics the process on a malformed secret key. Deliberate (comment `lib.rs:4161-4164`) — fail loud rather than inject a bogus credential.
- `rustls` is pinned to `default-features = false` with `ring` + `std` only (`Cargo.toml:33`), and the provider is installed exactly once before any connection (`lib.rs:1241-1243`).
