## Module: buzz-pair-relay & buzz-pairing-cli (`crates/buzz-pair-relay`, `crates/buzz-pairing-cli`)
### Aspect: Configuration

#### Environment variables

| Variable | Read site | Default | Documented? |
|---|---|---|---|
| `BUZZ_PAIR_RELAY_BIND_ADDR` | `crates/buzz-pair-relay/src/main.rs:9-10` — parsed as `SocketAddr`; the process exits with a `fatal:` message and code 1 on parse failure (`main.rs:12-15`) | `"127.0.0.1:5000"` | Yes, operationally — set explicitly in the Helm chart's Deployment env (`deploy/charts/buzz/templates/pairing-relay.yaml:37-38`, overridden there to `0.0.0.0:<port>`). **Not** mentioned in `AGENTS.md`, root `.env.example`, `deploy/compose/.env.example`, or `NOSTR.md` — confirmed by grepping all four for `PAIR`, which returns zero matches in the two `.env.example` files. |

That is the **entire** set of environment variables read by either crate in this scope — confirmed by grepping both crates' source for `std::env::var`/`env::var`: `buzz-pair-relay/src/main.rs` has exactly one call site (`main.rs:9`); `buzz-pairing-cli/src/main.rs` has **zero** — it reads no environment variables at all, despite its `clap` dependency being pulled in with the `env` feature enabled (`crates/buzz-pairing-cli/Cargo.toml:19`, `clap = { version = "4", features = ["derive", "env"] }`). This is a parsed-but-unused capability: the `env` Cargo feature is compiled in, but no `#[arg(env = "...")]` attribute appears anywhere on any `Cli`/`Cmd` field (`main.rs:40-72`), so the feature adds dependency weight and compile time with no behavioral effect.

Contrast with `AGENTS.md`'s own claim: "Auth env vars (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`) are auto-injected by the ACP harness into managed agent subprocesses" — that description is about `buzz-cli` (the agent-first CLI), not `buzz-pairing-cli`. None of those three variables appear anywhere in `crates/buzz-pairing-cli/src/main.rs`, so `buzz-pairing-cli` is not part of that auto-injection mechanism — all of its configuration comes from explicit CLI flags instead.

#### CLI flags

| Flag | Subcommand | Type | Default | Read site |
|---|---|---|---|---|
| `--relay` | `source` | `String` | `"wss://relay.damus.io"` | `main.rs:49-50` |
| `--nsec` | `source` | `Option<String>` | `None` (generates a throwaway test key) | `main.rs:53-54` |
| `--relay` | `target` | `Option<String>` | `None` (falls back to the relay URL embedded in the scanned QR) | `main.rs:60-61` |
| `--show-secret` | `target` | `bool` | `false` | `main.rs:64-65` |

No flag on either subcommand overlaps with the other; there is no shared/global flag (no `--format`, no `-v`/`--verbose`). `test-vectors` takes no flags at all (`main.rs:70`, `enum Cmd::TestVectors`).

#### Configuration surface of `buzz-pair-relay`'s protocol limits — compile-time constants, not runtime-configurable

Every operational limit in `buzz-pair-relay` is a Rust `const`, not read from any environment variable, config file, or CLI flag — there is no config struct, no `clap`/`serde` config type, nothing. The full list:

| Constant | Value | Purpose | Citation |
|---|---|---|---|
| `CONN_TIMEOUT` | 120s | Hard per-connection lifetime | `lib.rs:59` |
| `MAX_CONNS` | 128 | Global concurrent WebSocket connection cap | `lib.rs:61` |
| `CHANNEL_CAP` | 4 | Bounded mpsc channel capacity per connection's writer task | `lib.rs:62` |
| `KIND_PAIR` | 24134 | The only Nostr event kind accepted | `lib.rs:63` |
| `MAX_FRAME` | 4096 bytes | Max WebSocket frame/message size | `lib.rs:66` |
| `RATE_WINDOW` | 10s | Rate-limit window duration | `lib.rs:67` |
| `RATE_MSG_MAX` | 20 | Max inbound frames per window per connection | `lib.rs:68` |
| `RATE_EVENT_MAX` | 10 | Max EVENT messages per window per connection | `lib.rs:69` |
| `SUB_ID_MAX` | 64 bytes | Max subscription ID length | `lib.rs:70` |
| `MAX_EVENTS_PER_CONN` | 6 | Hard lifetime cap on accepted EVENTs per connection | `lib.rs:73` |
| `MAX_DELIVERED_PER_P` | 12 | Per-`#p` delivery budget | `lib.rs:76` |
| `DEDUP_CAP` | 1024 | Global dedup vec fail-closed capacity | `lib.rs:79` |
| `DELIVERED_MAP_CAP` | 4096 | Delivered-map fail-closed capacity | `lib.rs:82` |
| `ENTRY_TTL` | 300s | Eviction TTL for dedup/delivered entries | `lib.rs:86` |
| `FRESHNESS_SECS` | 120 (±) | `created_at` acceptance window | `lib.rs:89` |

None of these 14 constants is documented outside its own inline doc comment and the module-level security-model summary (`lib.rs:19-27`) — there is no operator-facing doc (Helm chart README, `NOSTR.md`, `AGENTS.md`) that lists these tunables or explains they are fixed at compile time. An operator wanting, say, a higher `MAX_CONNS` for a busier deployment has no lever to pull short of editing the source and rebuilding — a legitimate design choice for a minimal sidecar (fewer knobs, fewer misconfiguration risks), but undocumented as a *constraint*: someone provisioning the Helm chart's `pairingRelay.resources` (`deploy/charts/buzz/values.yaml:200-211`) might expect to scale connection capacity via replicas/resources without realizing 128 is a hardcoded per-process ceiling.

#### buzz-core::pairing configuration (not in scope, but read by buzz-pairing-cli)

`DEFAULT_TIMEOUT = 120s` (`crates/buzz-core/src/pairing/session.rs:43`) is likewise a hardcoded constant, not configurable by `buzz-pairing-cli` — the CLI has no flag to change the session lifetime, and `PairingSession::new_source`/`new_target` never take a timeout parameter. The only way to override it is the `#[cfg(test)]`-gated `set_timeout` method (`session.rs:541-543`), which is test-only and inaccessible to the CLI binary.

#### Helm chart configuration surface (deployment-level, not code-level)

| Key | Default | Purpose | Citation |
|---|---|---|---|
| `pairingRelay.enabled` | `false` | Whether to deploy the sidecar at all | `deploy/charts/buzz/values.yaml:194` |
| `pairingRelay.url` | `""` | Advertised in the main relay's NIP-11 doc (consumed by `buzz-relay`, not by either crate in this scope) | `values.yaml:195` |
| `pairingRelay.replicaCount` | `1` | Deployment replica count | `values.yaml:196` |
| `pairingRelay.service.type` | `ClusterIP` | K8s Service type | `values.yaml:198` |
| `pairingRelay.service.port` | `5000` | Service + container port, also injected into `BUZZ_PAIR_RELAY_BIND_ADDR` | `values.yaml:199`, `pairing-relay.yaml:37-38` |
| `pairingRelay.resources.requests`/`limits` | `50m`/`32Mi` requests, `250m`/`128Mi` limits | Pod resource sizing | `values.yaml:205-211` |

These are documented in `deploy/charts/buzz/README.md` (§"Device pairing relay", lines 55-68) and cross-referenced in `values.yaml`'s own comment block (`values.yaml:189-192`). This is the best-documented configuration surface in the whole module group — better documented than the in-code constants above.

#### Undocumented / parsed-but-never-read config: none found beyond the `clap` `env` feature

Beyond the unused `clap` `env` feature already noted, no other parsed-but-unread configuration was found in either crate. Neither crate has a config-file loader (no `.toml`/`.yaml`/`.json` parsing) — everything is either a CLI flag, the single env var, or a compile-time constant.
