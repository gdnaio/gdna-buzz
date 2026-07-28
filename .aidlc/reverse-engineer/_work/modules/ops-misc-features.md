## Module: buzz-admin, sprig & countdown-bot (`crates/buzz-admin`, `crates/sprig`, `examples/countdown-bot`)
### Aspect: Features

#### buzz-admin: what an operator can do end-to-end

`buzz-admin` is the shipped operator tool for managing a single deployment's relay membership and a handful of other break-glass/bootstrap tasks, run either via `./run.sh <cmd>` in Compose deployments (`deploy/compose/run.sh:89-96`) or directly with `docker compose exec relay buzz-admin ...` (`NOSTR.md:227-234`), or via `cargo run -p buzz-admin --` in local dev (`Justfile:190`, inside `_ensure-migrations`).

| Feature | End-to-end flow | Evidence |
|---|---|---|
| Bootstrap a fresh database | `buzz-admin migrate` runs pending SQLx migrations before the relay ever starts, or ahead of `BUZZ_AUTO_MIGRATE=true` | `main.rs:139-143`; documented as the alternative to `BUZZ_AUTO_MIGRATE` in `deploy/compose/README.md` ("Production notes" bullet on `BUZZ_AUTO_MIGRATE`) |
| Generate a bootstrap identity | `buzz-admin generate-key` prints a fresh keypair and a reminder to set `BUZZ_PRIVATE_KEY` | `main.rs:132-138` |
| Add a relay member | `buzz-admin add-member --pubkey <npub-or-hex> [--role admin\|member]`, then live clients see the updated kind:13534 roster via Redis without restarting the relay | `main.rs:158-192`; roster fan-out via `publish_membership_list_with_bump` → `pubsub.publish_event` (`main.rs:369-375`) |
| Remove a relay member (with optional safety check) | `buzz-admin remove-member --pubkey <npub-or-hex> [--role <role>]`; owner is always protected | `main.rs:194-251` |
| Audit the current roster | `buzz-admin list-members` prints a formatted table (pubkey, role, added_by, created_at) | `main.rs:260-286` |
| Backfill missing NIP-29 discovery events | `buzz-admin reconcile-channels [--relay-key <hex>]` — makes channels created via direct SQL/seed scripts visible to pure-Nostr clients that discover channels via kind:39000/39001/39002 | `main.rs:461-579`; motivating doc comment `main.rs:84-88` |
| Review deployment-wide product feedback from the terminal | `buzz-admin product-feedback list [--limit N]` prints pretty JSON | `main.rs:253-258` |

The `NOSTR.md` "CLI: Managing Members" section documents three of these (`add-member`, `remove-member`, `list-members`) with worked examples and an exit-code table (`NOSTR.md:208-251`) — the other four subcommands (`generate-key`, `migrate`, `reconcile-channels`, `product-feedback list`) are undocumented in `NOSTR.md` and only partially documented in `ARCHITECTURE.md`'s `buzz-admin` section (which lists `add-member`, `remove-member`, `list-members`, `generate-key`, `reconcile-channels` but omits `migrate` and `product-feedback`). See `ops-misc-debt.md`.

Distribution: the `buzz-admin` binary is built and stripped into the same multi-stage Docker image as the relay (`Dockerfile:67-72`, `:139`) and lands at `/usr/local/bin/buzz-admin` inside the container — an operator never installs it separately; it ships wherever the relay ships.

#### sprig: one binary, three (or more) agent-surface personalities

`sprig`'s feature is entirely about deployment ergonomics: instead of shipping three separate binaries (`buzz-acp`, `buzz-agent`, `buzz-dev-mcp`) to a fresh host, an operator ships **one** binary plus symlinks and gets all three, because the multicall binary's shared Rust runtime/TLS stack is stored once rather than three times (`scripts/build-sprig.sh` README heredoc, lines documenting "so shared Rust runtime/TLS code is stored only once"). This confirms the framing in the task: sprig is genuinely a single binary that behaves as three different agent-surface tools depending on invocation name, and its practical purpose is exactly what the CI workflow's own header comment states — "one deploy-anywhere Linux multicall binary" (`.github/workflows/sprig.yml:1-9`).

| Personality | What it enables |
|---|---|
| `buzz-acp` | ACP harness — bridges Buzz channel `@mentions` to an AI agent subprocess over stdio (per `ARCHITECTURE.md`'s `buzz-acp` section) |
| `buzz-agent` | The ACP-compliant agent itself (spawns MCP servers, calls LLMs) |
| `buzz-dev-mcp` | Developer MCP server (shell/file-edit tools for the agent), which *itself* multicalls further into `rg`, `tree`, `buzz` (the agent CLI), `git-credential-nostr`, `git-sign-nostr` (`crates/buzz-dev-mcp/src/lib.rs:149-152`) |

So a single `sprig` artifact, correctly symlinked, effectively stands in for **eight** command names (`sprig`, `buzz-acp`, `buzz-agent`, `buzz-dev-mcp`, `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` — the last five via `buzz-dev-mcp`'s own dispatch). This is exactly the "deploy-anywhere environments" use case the release workflow targets: static-musl builds for `x86_64`/`aarch64` Linux (`.github/workflows/sprig.yml:34-38`), released on a rolling `sprig-latest` tag on every push to `main` and on versioned `sprig-v*` tags (`sprig.yml:14-15,105-146`).

`build-sprig.sh` is the feature's actual packaging implementation: it builds `sprig` once, then creates the `buzz-acp`/`buzz-agent`/`buzz-dev-mcp` symlinks and a `sprig.json` manifest with per-binary SHA-256/size (`scripts/build-sprig.sh:104-131`), and bundles a self-contained `README.md` inside the tarball with install/configure instructions (`build-sprig.sh:135-166`).

#### countdown-bot: what it demonstrates

Per its own module doc comment and README, `countdown-bot` is a worked example proving that **Buzz participants are not required to be LLM agents** — any process that can hold a Nostr key, complete NIP-42 auth, publish a kind:0 profile, subscribe, and publish kind:9 messages can be a first-class Buzz participant (`examples/countdown-bot/src/main.rs:1-13`, README "It demonstrates that Buzz participants do not have to be LLM agents"). Concretely it demonstrates:

- The **raw NIP-42 handshake** end-to-end (challenge → signed AUTH event → OK), using plain `tokio-tungstenite` + the `nostr` crate rather than `buzz-ws-client` or `buzz-cli` — the README states this is deliberate, "so the protocol path is easy to inspect in one small file" (README, Notes section).
- Two independent **relay-membership auth strategies** for third-party/bot identities: `standalone` (bot key is its own relay member) and `owner-attested` (bot key rides in on a NIP-OA `auth` tag signed by an already-admitted owner/agent key) — both implemented and selectable via `BUZZ_BOT_AUTH_MODE` (`main.rs:90-116`).
- A **minimal command bot** pattern: prefix commands (`!countdown`, `!fib`) and @mention commands, both bounded to prevent spam (README "Notes" — "Commands are bounded ... so one message cannot make the bot spam the relay").
- Self-registration as a NIP-29 channel member (`kind:9000`, `role=bot`) so the bot appears in the members list and mention autocomplete, with graceful degradation on private channels where it isn't pre-authorized (`announce_channel_membership`, `main.rs:172-190`; README explains the fallback: "an owner/admin must add the bot pubkey to the channel membership").

It is explicitly a **reference/example** program, not a shipped product feature: `examples/countdown-bot/Cargo.toml:5` sets `publish = false`, and the crate lives under `examples/` rather than `crates/`. It is excluded from the module inventory's LOC/test columns (`.aidlc/reverse-engineer/modules.md`'s `countdown-bot (example)` row shows `LOC: —`, `Tests: —`) even though it does have 437 lines and one test module — see `ops-misc-debt.md` for that documentation gap.
