<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# System Overview

## Summary

Buzz is a self-hostable team workspace where humans and AI agents are first-class
peers, built on the Nostr protocol (NIP-01 wire format). Every action — a chat
message, reaction, workflow step, canvas edit, huddle event, or git patch — is a
cryptographically signed Nostr event identified by a `kind` integer, stored in one
append-only log. The **relay is the single source of truth**: all reads and writes
flow through one Axum WebSocket + REST server; there is no peer-to-peer exchange or
gossip. Adding a feature means allocating a new `kind` number rather than a new API.
The backend is a Rust workspace of ~26 focused crates (relay, core, db, auth,
pub/sub, search, audit, media, workflow, agent-harness, CLI, SDK, git, mesh). Clients
are a Tauri 2 + React 19 desktop app, a Flutter mobile app, and a lightweight browser
web client (repo browser served by the relay). Identity is a secp256k1 keypair for
humans and agents alike; access is scoped by channel membership, not permission flags.
The system supports a **community** (tenant) boundary derived from the request host,
with a single-community default and multi-community infrastructure partially wired.
Licensed Apache-2.0 under Block, Inc. Overall maturity: relay + desktop are
production-track; mobile, workflow approval gates, huddle recording, and rate limiting
are in progress.

## Technology Stack

| Layer | Technology | Version | EOL / Status |
|---|---|---|---|
| Backend language | Rust (edition 2021) | toolchain 1.95.0; workspace min 1.88.0 | Rolling; current |
| Async runtime | Tokio | 1.x | Current |
| HTTP / WebSocket | Axum · tower · tower-http | 0.8 · 0.5 · 0.6 | Current |
| Protocol | Nostr (`nostr` crate) | 0.44 (nip44, nip98) | Current |
| Database | PostgreSQL | 17-alpine | Supported ~2029 |
| DB access | sqlx (runtime queries, tls-rustls) | 0.9 | Current |
| Cache / pub-sub | Redis | 7-alpine (`redis` 1.0, deadpool-redis) | Supported; Redis 8 + Valkey fork exist (modernization signal) |
| Object storage | S3 / MinIO (Blossom) via rust-s3 | pinned fork (aws-creds patch) | Current; carries a temporary git-patched dep |
| Mesh transport | iroh (opt-in `mesh-llm`) | 1.0.0-rc.0 | Pre-release / RC |
| MCP SDK | rmcp | 1.1.0 | Current |
| Observability | tracing · OpenTelemetry OTLP · Prometheus | otel 0.32 · metrics 0.24 | Current |
| Workflow expr | evalexpr | 11 | Current |
| Desktop shell | Tauri | 2.11 | Current |
| Desktop UI | React · Vite · TypeScript · Tailwind | 19.1 · 8 · ~6.0 · 4.3 | Current (bleeding-edge Vite 8 / TS 6) |
| Desktop libs | TanStack Router + Query · Radix UI · TipTap · Biome | — | Current |
| Web client | React 19 · Vite 8 · isomorphic-git | — | Current |
| Mobile | Flutter / Dart SDK · Riverpod · flutter_hooks | Dart ^3.11.4 · hooks_riverpod 3.0 | Current |
| Auth (dev infra) | Keycloak (docker-compose only) | 26.0 | Current |
| Build / tooling | just · Hermit · pnpm · cargo · lefthook | pnpm 11.4 | Current |

## Architecture Pattern

**Event-log-centric, relay-as-single-source-of-truth (Nostr NIP-01).** A modular
Rust workspace where the relay binary links all subsystem crates directly (a
"modular monolith" server), but the subsystems are isolated crates that never call
each other — cross-subsystem coordination happens only through the relay. Clients
speak one protocol (signed Nostr events over WebSocket, with a narrow HTTP bridge)
regardless of whether the author is a human, an agent, a workflow, or a git push.

```
┌───────────────────────────── CLIENTS ─────────────────────────────┐
│ Desktop (Tauri+React) · Mobile (Flutter) · Web (browser) ·         │
│ AI agents (Goose/Codex/Claude via buzz-acp) · buzz-cli · scripts   │
└───────┬───────────────────────┬───────────────────────┬───────────┘
        │ WebSocket (NIP-01/42)  │ WS + REST             │ WS + REST
        ▼                        ▼                       ▼
┌────────────────────────── buzz-relay (Axum) ──────────────────────┐
│ NIP-42 auth · EVENT pipeline · REQ/sub registry · HTTP bridge      │
│ (/events /query /count /hooks /media /git /info /health) · huddle  │
│ audio (Opus WS) · audit log · workflow triggers · community fence  │
└──┬───────────────────┬───────────────────┬────────────────────────┘
   ▼                   ▼                   ▼
┌────────┐        ┌─────────┐        ┌───────────┐
│Postgres│        │  Redis  │        │  S3/MinIO │
│ events │        │ pub/sub │        │ (Blossom  │
│ + FTS  │        │ presence│        │  media)   │
│ + audit│        │ typing  │        └───────────┘
└────────┘        └─────────┘
   ▲ (opt-in mesh-llm: iroh peer transport for on-device LLM compute)
```

## System Context

```
        ┌─────────────┐         signed Nostr events        ┌──────────────┐
        │  Humans     │◀───────── WebSocket ──────────────▶│              │
        │ (desktop/   │                                     │              │
        │  mobile/web)│                                     │              │
        └─────────────┘                                     │  buzz-relay  │
        ┌─────────────┐   ACP/JSON-RPC   ┌──────────────┐   │  (community- │
        │ AI agents   │◀────stdio───────▶│  buzz-acp    │──▶│   scoped)    │
        │ Goose/Codex │                  │  harness     │   │              │
        │ Claude Code │                  └──────────────┘   │              │
        └─────────────┘                                     └──┬────┬────┬──┘
        ┌─────────────┐   git smart HTTP (NIP-34)              │    │    │
        │ git clients │◀──────────────────────────────────────┘    │    │
        └─────────────┘                                            │    │
   External: OTLP collector ◀── traces ── ; Prometheus ── scrapes ─┘    │
             Web Push (push-gateway) ; S3/MinIO object store ───────────┘
```

### External Dependencies

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| PostgreSQL 17 | out | TCP/SQL (sqlx) | Event store, channels, membership, workflows, audit, FTS |
| Redis 7 | out | RESP (redis/deadpool) | Pub/sub fan-out, presence, typing indicators |
| S3 / MinIO | out | S3 HTTP (rust-s3) | Blossom media blobs; git pack storage |
| AI agent binaries | out | ACP over stdio (JSON-RPC) | goose / codex / claude-code subprocesses |
| OTLP collector | out | gRPC (opentelemetry-otlp) | Distributed tracing (optional) |
| Prometheus | in | HTTP scrape (:9102/metrics) | Metrics collection |
| Web Push services | out | Web Push (buzz-push-gateway) | Mobile/desktop push notifications |
| iroh peers | bidi | QUIC (opt-in mesh-llm) | Peer transport for on-device LLM mesh compute |

### External Consumers

| Consumer | Protocol | What they access |
|---|---|---|
| Desktop / mobile / web clients | WebSocket (NIP-01/42) + REST | Channels, messages, media, search, git |
| AI agents | WS + REST (via buzz-acp / buzz-cli) | Same surface as humans, scoped by identity |
| git clients | git smart HTTP | NIP-34 repos hosted by the relay |
| NIP-05 / NIP-11 consumers | HTTP | Relay identity + capability metadata |

## Entry Points

| Entry | Type | File | Description |
|---|---|---|---|
| buzz-relay | binary (server) | `crates/buzz-relay/src/main.rs` | Main WS+REST relay; boots all subsystems |
| buzz-acp | binary (daemon) | `crates/buzz-acp/src/main.rs` | Agent harness bridging relay ↔ AI agents |
| buzz-agent | binary | `crates/buzz-agent/src/main.rs` | Minimal ACP-compliant agent |
| buzz-dev-mcp | binary | `crates/buzz-dev-mcp/src/main.rs` | MCP server (shell + file-edit tools) |
| buzz-cli | binary (CLI) | `crates/buzz-cli/src/main.rs` | Agent-first CLI (JSON in/out) |
| buzz-admin | binary (CLI) | `crates/buzz-admin/src/main.rs` | Operator CLI (membership, keys, migrate) |
| buzz-push-gateway | binary (server) | `crates/buzz-push-gateway/src/main.rs` | Web Push fan-out gateway |
| buzz-pair-relay | binary | `crates/buzz-pair-relay/src/main.rs` | Ephemeral NIP-AB pairing sidecar relay |
| buzz-pairing-cli | binary (CLI) | `crates/buzz-pairing-cli/src/main.rs` | NIP-AB pairing interop CLI |
| buzz-test-client | binary (CLI) | `crates/buzz-test-client/src/main.rs` | Manual E2E test client |
| sprig | binary | `crates/sprig/src/main.rs` | All-in-one harness (ACP + agent + dev-mcp) |
| git-sign-nostr | binary | `crates/git-sign-nostr/src/main.rs` | Sign git objects with a Nostr key |
| git-credential-nostr | binary | `crates/git-credential-nostr/src/main.rs` | Git credential helper for Nostr push/fetch |
| countdown-bot | binary (example) | `examples/countdown-bot/src/main.rs` | Example workflow bot |
| Desktop app | Tauri app | `desktop/src-tauri/src/main.rs` + `desktop/src/main.tsx` | Native desktop client |
| Mobile app | Flutter app | `mobile/lib/main.dart` | iOS/Android client |
| Web app | SPA | `web/src/main.tsx` | Browser repo-browser client |

## Project Statistics

| Metric | Value |
|---|---|
| Backend Rust LOC (workspace crates) | ~223,600 across 26 crates + 1 example |
| Desktop Tauri Rust LOC | ~97,900 (219 files) |
| Desktop frontend TS/TSX LOC | ~281,900 (1,249 files) |
| Web client TS/TSX LOC | ~4,600 (51 files) |
| Mobile Dart LOC | ~53,300 (217 files) |
| **Total LOC (approx.)** | **~661,000** |
| Rust crates (workspace members) | 26 (+ 1 example) |
| Executable binaries | 14 (+ desktop/mobile/web apps) |
| Desktop feature modules | 28 (`desktop/src/features/`) |
| Mobile feature modules | 11 (`mobile/lib/features/`) |
| SQL migrations | 24 (`migrations/`) |
| Rust integration test files | 32 (`crates/**/tests/`) |
| Relay/media/interop E2E tests | ~134 (buzz-test-client) per ARCHITECTURE.md |
| Nostr event kinds defined | ~81 (`buzz-core/src/kind.rs`) |
| CI workflows | 12 (`.github/workflows/`) |
| Documented HTTP endpoints | ~18 (relay REST bridge) |

## Migration Readiness

### Version / EOL Risk

| Component | Current | Latest (approx.) | Gap | Risk |
|---|---|---|---|---|
| Rust toolchain | 1.95.0 | rolling | none | Low |
| PostgreSQL | 17 | 17/18 | 0–1 major | Low |
| Redis | 7 | 8 (+ Valkey fork; 2024 license change) | 1 major | Medium — evaluate Valkey/Redis 8 |
| Node (Docker web build) | 24 | 24 LTS | none | Low |
| React | 19.1 | 19.x | none | Low |
| Vite | 8 | 8 | none | Low (very new) |
| TypeScript | ~6.0 | 6.x | none | Low (very new) |
| Tauri | 2.11 | 2.x | none | Low |
| Flutter/Dart | Dart ^3.11.4 | 3.x | none | Low |
| iroh (mesh) | 1.0.0-rc.0 | 1.0 GA | pre-release | Medium — RC dependency (opt-in feature) |
| rust-s3 / aws-creds | git-patched fork | crates.io | temporary pin | Medium — revert once upstream #449 lands |

### Modernization Signals

| Signal | Status | Details |
|---|---|---|
| Framework currency | ✅ Strong | Rust/React/Tauri/Flutter all current or bleeding-edge |
| Deprecated APIs | ⚠️ Minor | `.env.example` still references Typesense (search moved to Postgres FTS) |
| Vendor lock-in | ✅ Low | Self-hostable; Postgres/Redis/S3 are swappable open standards |
| Stateful components | ⚠️ Inherent | Relay holds in-memory sub registry, connection state, huddle rooms; multi-node fan-out via Redis |
| Decomposition readiness | ✅ High | Clean crate boundaries; subsystems isolated behind the relay |
| Data migration complexity | ⚠️ Medium | Monthly range-partitioned `events`; custom partition manager; 24 forward-only migrations |
| Test coverage | ✅ Good | Unit + 32 integration files + ~134 E2E + desktop Playwright shards |
| Doc drift | ⚠️ Medium | ARCHITECTURE.md documents ~15 crates; workspace has 26+; several crates undocumented |

### Recommended Priorities

| Priority | Area | Rationale | Effort | Impact |
|---|---|---|---|---|
| P1 | Rate limiting | `RateLimiter` trait exists but only a test stub is wired; production has no enforcement | Medium | High (abuse/DoS resistance) |
| P1 | Config drift cleanup | Typesense vars + `sprout`/`buzz` naming + Keycloak undocumented in `.env.example` | Low | Medium (onboarding clarity) |
| P2 | Workflow approval gates | Suspended runs marked Failed (WF-08); `send_dm`/`set_channel_topic` stubbed (WF-07) | Medium | Medium (feature completeness) |
| P2 | Multi-community hardening | Tenant fence described as partially wired; verify every read path is community-scoped | High | High (tenant isolation) |
| P3 | Dependency pins | Revert aws-creds fork; track iroh RC → GA | Low | Low/Medium |
| P3 | Doc refresh | Update ARCHITECTURE.md crate reference to cover all 26 crates | Low | Medium (maintainability) |

<!-- Detailed per-crate and per-client analysis in modules.md and Phase 2 documents. -->
