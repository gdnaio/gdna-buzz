## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Configuration

This crate has **no runtime configuration surface**. Findings are recorded so the
absence is explicit rather than assumed.

---

### 1. Environment variables

| Variable | Read where | Notes |
|---|---|---|
| — | — | A search for `env::var`, `env!`, `option_env!`, and `std::env` across `crates/buzz-sdk/src/` returns **no matches**. No SDK behavior is environment-dependent. |

The only `std::env` use in the crate is `std::env::args()` in the example binary,
which reads positional CLI arguments, not environment variables
(`crates/buzz-sdk/examples/compute_auth_tag.rs:12`).

Note for contrast: the agent-facing environment variables documented for Buzz
(`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`) are consumed by
`buzz-cli` / the ACP harness, **not** by this crate — `buzz-sdk` has no relay
URL, no key source, and no transport.

---

### 2. Cargo features

| Feature | Gates | Declared at |
|---|---|---|
| — | — | `crates/buzz-sdk/Cargo.toml` has **no `[features]` section** (`Cargo.toml:1-17`) |

No `#[cfg(feature = ...)]` attribute exists anywhere in `src/` or `examples/`.
The only conditional compilation is `#[cfg(test)]` on the three test modules
(`builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301`).

Feature selection the crate *inherits* (workspace-pinned, not switchable here):

| Dependency | Features enabled by the workspace | Declared at |
|---|---|---|
| `nostr` `0.44` | `nip44`, `nip98` | `Cargo.toml:61` (root) |
| `uuid` `1` | `v4`, `serde` | `Cargo.toml:89` (root) |
| `serde` `1` | `derive` | `Cargo.toml:64` (root) |

All six dependencies are declared as `{ workspace = true }`, so versions and
feature sets are centrally controlled (`crates/buzz-sdk/Cargo.toml:10-16`).

---

### 3. Package metadata

Everything except `name` and `description` is inherited from the workspace
(`crates/buzz-sdk/Cargo.toml:1-8`):

| Key | Value |
|---|---|
| `name` | `buzz-sdk` |
| `version` / `edition` / `rust-version` / `license` / `repository` | `.workspace = true` |
| `description` | `"Typed Nostr event builders for Buzz operations"` |

There is no `[[bin]]`, no `[lib]` override, no `build.rs`, and no
`[dev-dependencies]` section.

---

### 4. Compile-time behavior switches

| Switch | Effect | File:line |
|---|---|---|
| `#![deny(unsafe_code)]` | build fails on any `unsafe` | `crates/buzz-sdk/src/lib.rs:1` |
| `#![warn(missing_docs)]` | undocumented public items warn | `crates/buzz-sdk/src/lib.rs:2` |

---

### 5. Effective "configuration" — hardcoded limits

Because nothing is configurable at runtime, the crate's tunable behavior is
fixed in constants and inline literals. Consumers cannot override them.

| Limit | Value | File:line |
|---|---|---|
| `MENTION_CAP` | 50 mention p-tags | `mentions.rs:38` |
| `MAX_CONTACTS` | 10 000 NIP-02 contacts | `builders.rs:751` |
| `MAX_REASON_BYTES` | 64 UTF-8 bytes | `builders.rs:1704` |
| `CUSTOM_EMOJI_SET_D_TAG` | `"buzz:custom-emoji"` | `builders.rs:503` |
| Default content cap | 64 KiB (13 call sites) | `builders.rs:227`, `284`, `299`, `383`, `742`, `1087`, `1227`, `1335`, `1421`, `1468`, `1486`, `1795`, `1816` |
| Diff / patch content cap | 60 KiB | `builders.rs:314`, `1023` |
| Reaction emoji cap | 64 chars | `builders.rs:467` |
| Emoji shortcode cap | 64 bytes | `builders.rs:136` |
| Emoji / contact URL cap | 2048 bytes | `builders.rs:157`, `787` |
| Repo name / description / clone URL / relay caps | 128 / 1024 / 512 / 256 chars | `builders.rs:845`, `856`, `875`, `907` |
| Repo clone-URL and relay counts | 5 / 10 | `builders.rs:868`, `900` |
| Subject cap (issue, PR) | 256 chars | `builders.rs:1092`, `1340` |
| Petname cap | 256 bytes | `builders.rs:794` |
| DM participants | 1–8 | `builders.rs:1546` |
| NIP-OA `kind` clause range | 0–65535 | `nip_oa.rs:63` |
| NIP-OA `created_at` clause range | 0–4294967295 | `nip_oa.rs:65`, `67` |
