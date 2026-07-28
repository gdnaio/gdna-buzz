## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Configuration

Summary: this crate has **no runtime configuration mechanism of its own**. Everything
configurable is a function parameter or a compile-time constant.

---

### 1. Environment variables

**None.** Verified by search across `crates/buzz-ws-client/`: no `env::var`, no
`std::env`, no `dotenv`, no `option_env!`. The crate never reads process
environment. Environment-driven values such as `BUZZ_RELAY_URL`,
`BUZZ_PRIVATE_KEY`, and `BUZZ_AUTH_TAG` are resolved by the consuming crates and
passed in as arguments — e.g. `buzz-cli` supplies `ws_url`, `self.keys`, and
`self.auth_tag` at `crates/buzz-cli/src/client.rs:1080`.

---

### 2. Cargo features

| Feature | Declared? | Notes |
|---|---|---|
| (none) | No `[features]` section exists in `crates/buzz-ws-client/Cargo.toml` (file is 17 lines: `[package]` `:1`–`7`, `[dependencies]` `:9`–`17`) | No `#[cfg(feature = …)]` anywhere in the source (verified) |

All dependency features are inherited from the workspace root via
`{ workspace = true }` (`crates/buzz-ws-client/Cargo.toml:10`–`17`). The two that
change this crate's behaviour:

| Inherited feature set | Declared at | Effect on this crate |
|---|---|---|
| `tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }` | root `Cargo.toml:113` | Determines the TLS backend and trust anchors used by `connect_async` (`connection.rs:53`) |
| `tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "time", "sync", "io-util", "signal", "process"] }` | root `Cargo.toml:43` | `time` powers `timeout`/`Instant` (`connection.rs:7`, `:134`, `:176`); `net` provides `TcpStream` in the stream alias (`connection.rs:14`) |
| `nostr = { version = "0.44", features = ["nip44", "nip98"] }` | root `Cargo.toml:61` | Event/keys/builder types (`message.rs:1`); neither listed feature is referenced by this crate's code |

The only `#[cfg]` in the crate is the test gate `#[cfg(test)]`
(`connection.rs:296`).

---

### 3. Compile-time constants (the crate's tunables)

| Constant | Value | Visibility | Where applied | file:line |
|---|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | `20` | `pub` | `authenticate` challenge wait | decl `connection.rs:17`; use `:76` |
| `AUTH_OK_TIMEOUT_SECS` | `20` | `pub` | `authenticate` OK wait | decl `connection.rs:20`; use `:85` |
| `PUBLISH_OK_TIMEOUT_SECS` | `30` | `pub` | `send_event` OK wait | decl `connection.rs:23`; use `:99` |
| Challenge byte cap | `1024` (inline literal, not a named constant) | n/a | `wait_for_auth_challenge` socket-read path | `connection.rs:198` |

These are `const`, so they are not overridable at runtime; changing a timeout requires
a code edit. Lower bounds are guarded by compile-time assertions in the test module
(`connection.rs:300`–`313`), so an edit that reduces them below 20/20/30 fails to
build.

---

### 4. Caller-supplied configuration (the actual configuration surface)

| Parameter | Type | Function | file:line | Notes |
|---|---|---|---|---|
| `url` / `relay_url` | `&str` | `connect`, `connect_authenticated`, `publish_event` | `connection.rs:38`, `:48`, `:278` | No default; no scheme restriction |
| `keys` | `&Keys` | `connect_authenticated`, `authenticate`, `publish_event`, `build_auth_event` | `connection.rs:39`, `:72`, `:280`; `message.rs:154` | Signing identity |
| `auth_tag` | `Option<&Tag>` | same four | `connection.rs:40`, `:73`, `:281`; `message.rs:155` | Optional single authorization tag added to the AUTH event (`message.rs:159`–`163`) |
| `timeout_dur` | `Duration` | `next_event` | `connection.rs:106` | Per-call read budget; no default provided by the crate |
| `timeout_secs` | `u64` | `publish_event` | `connection.rs:282` | Whole-operation budget; `buzz-cli` passes `75` (`crates/buzz-cli/src/client.rs:1080`) |

Note the composition consequence: `publish_event`'s outer budget must exceed the sum
of the inner fixed waits (20 s challenge + 20 s auth OK + 30 s publish OK = 70 s) for
the inner errors to surface rather than the outer `Timeout`; `buzz-cli`'s `75`
(`crates/buzz-cli/src/client.rs:1080`) sits just above that sum, with a comment
referencing the three constants at `crates/buzz-cli/src/client.rs:1077`.

---

### 5. Logging configuration

`tracing::debug!` is used at three sites (`connection.rs:57`, `:91`, `:123`). The
crate installs no subscriber and reads no log-level configuration — verbosity is
controlled entirely by the host binary's `tracing` setup.

---

### 6. Package metadata

All inherited from the workspace: `version.workspace`, `edition.workspace`,
`rust-version.workspace`, `license.workspace`, `repository.workspace`
(`crates/buzz-ws-client/Cargo.toml:3`–`7`). No `build.rs`, no `[lib]` section
(default lib name `buzz_ws_client`), no `[[bin]]`, no `[dev-dependencies]`.
Registered in the workspace at root `Cargo.toml:16` and exposed as a workspace
dependency at root `Cargo.toml:134`.
