## Module: buzz-media (`crates/buzz-media`)

### Aspect: Configuration

### 1. Environment variables read *inside* this crate

**None in `src/`.** A scan of all 11 source files finds zero `std::env`/`env::var` uses. `MediaConfig` is a plain `serde::Deserialize` struct (`crates/buzz-media/src/config.rs:16-17`); the caller populates it. Env reading happens only in the ignored integration test:

| Var | Default in test | file:line |
|---|---|---|
| `BUZZ_S3_ENDPOINT` | `http://localhost:9000` | `crates/buzz-media/tests/static_creds_minio.rs:23-24` |
| `BUZZ_S3_ACCESS_KEY` | `buzz_dev` | `crates/buzz-media/tests/static_creds_minio.rs:25-26` |
| `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` | `crates/buzz-media/tests/static_creds_minio.rs:27-28` |
| `BUZZ_S3_BUCKET` | `buzz-media` | `crates/buzz-media/tests/static_creds_minio.rs:29` |

Indirect env consumption: `Credentials::default()` triggers the AWS credential chain, which reads `AWS_*` variables (env, profile, web-identity/IRSA, container — including the Pod Identity `AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` pair the patched `aws-creds` fork adds) inside the dependency, not in this crate (`crates/buzz-media/src/storage.rs:25-33`, `:51-56`; fork rationale `Cargo.toml:163-171`).

---

### 2. `MediaConfig` fields and the env vars that populate them

Env-var names and default values come from the relay (`crates/buzz-relay/src/config.rs:614-657`); the type and any serde default come from this crate.

| Field | Type | Serde default (this crate) | Env var (relay) | Relay default |
|---|---|---|---|---|
| `s3_endpoint` | `String` | required | `BUZZ_S3_ENDPOINT` | `http://localhost:9000` |
| `s3_access_key` | `String` | required | `BUZZ_S3_ACCESS_KEY` | `buzz_dev` |
| `s3_secret_key` | `String` | required | `BUZZ_S3_SECRET_KEY` | `buzz_dev_secret` |
| `s3_bucket` | `String` | required | `BUZZ_S3_BUCKET` | `buzz-media` |
| `s3_region` | `String` | `default_s3_region()` = `"us-east-1"` (`crates/buzz-media/src/config.rs:11-13`, `:24-27`) | `BUZZ_S3_REGION`, falling back to `AWS_REGION` | `us-east-1` |
| `max_image_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:39-41`) | `BUZZ_MAX_IMAGE_BYTES` | `50 * 1024 * 1024` = 52_428_800 |
| `max_gif_bytes` | `u64` | **none — required** (`crates/buzz-media/src/config.rs:42-44`) | `BUZZ_MAX_GIF_BYTES` | `10 * 1024 * 1024` = 10_485_760 |
| `max_video_bytes` | `u64` | `default_max_video_bytes()` = `524_288_000` (500 MB) (`crates/buzz-media/src/config.rs:3-5`, `:45-47`) | `BUZZ_MAX_VIDEO_BYTES` | `500 * 1024 * 1024` = 524_288_000 |
| `max_file_bytes` | `u64` | `default_max_file_bytes()` = `104_857_600` (100 MB) (`crates/buzz-media/src/config.rs:7-9`, `:48-50`) | `BUZZ_MAX_FILE_BYTES` | `100 * 1024 * 1024` = 104_857_600 |
| `public_base_url` | `String` | required | `BUZZ_MEDIA_BASE_URL` | `http://localhost:3000/media` |
| `upload_records_enabled` | `bool` | `#[serde(default)]` → `false` (`crates/buzz-media/src/config.rs:49-50`) | `BUZZ_MEDIA_UPLOAD_RECORDS` (`"true"`/`"1"`) | `false` |
| `upload_ip_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:55-56`) | `BUZZ_MEDIA_UPLOAD_IP_HEADER` (trimmed, lowercased) | unset |
| `upload_port_header` | `Option<String>` | `#[serde(default)]` → `None` (`crates/buzz-media/src/config.rs:60-61`) | `BUZZ_MEDIA_UPLOAD_PORT_HEADER` (trimmed, lowercased) | unset |

Related relay-side knobs that shape this crate's behaviour but are not `MediaConfig` fields: `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (default 8), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (default 2, clamped to the global), `BUZZ_MEDIA_UPLOADS_PER_MINUTE` (default 30) (`crates/buzz-relay/src/config.rs:659-676`), and `BUZZ_REQUIRE_MEDIA_GET_AUTH` (default `false`, `crates/buzz-relay/src/config.rs:677-685`). `.env.example` documents the first three at `.env.example:85-87` and the read-auth flags at `.env.example:90-92`.

---

### 3. Startup validation rules (`MediaConfig::validate`, `crates/buzz-media/src/config.rs:66-122`)

| Check | Failure message | file:line |
|---|---|---|
| `public_base_url` must end with `/media` | `public_base_url must end with /media: got '…'` | `crates/buzz-media/src/config.rs:67-72` |
| `public_base_url` must not end with `/` | `public_base_url must not end with /: got '…'` | `crates/buzz-media/src/config.rs:73-78` |
| `max_image_bytes > 0` | `max_image_bytes must be > 0` | `crates/buzz-media/src/config.rs:79-81` |
| `max_gif_bytes > 0` and `<= max_image_bytes` | `max_gif_bytes must be > 0 and <= max_image_bytes` | `crates/buzz-media/src/config.rs:82-84` |
| `max_video_bytes > 0` | `max_video_bytes must be > 0` | `crates/buzz-media/src/config.rs:85-87` |
| `max_file_bytes > 0` | `max_file_bytes must be > 0` | `crates/buzz-media/src/config.rs:88-90` |
| IP header set without records enabled | long message ending "Enable upload records or unset the header." | `crates/buzz-media/src/config.rs:91-102` |
| Port header set without IP header | long message ending "Set the IP header or unset the port header." | `crates/buzz-media/src/config.rs:103-110` |
| Both header names must be valid HTTP header names (`axum::http::HeaderName::from_bytes`) | `{VAR} is not a valid header name: {h:?}` | `crates/buzz-media/src/config.rs:111-124` |

The two coherence checks are justified in-code as fail-loud choices: "an operator who set an IP header believes they are meeting a reporting obligation" (`crates/buzz-media/src/config.rs:91-95`).

---

### 4. Cargo features

The crate declares **no `[features]` section** (`crates/buzz-media/Cargo.toml`), and no `#[cfg(feature = …)]` appears in the source. Feature selection is only on dependencies:

| Dependency | Features |
|---|---|
| `s3` (`rust-s3` 0.37) | `default-features = false`, `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`) |
| `image` 0.25 | `default-features = false`, `jpeg`, `png`, `gif`, `webp` (`crates/buzz-media/Cargo.toml:26`) |
| `tokio-util` 0.7 | `io` (`crates/buzz-media/Cargo.toml:31`) |
| `tokio` (dev) | `test-util` (`crates/buzz-media/Cargo.toml:33-34`) |

Version/edition/license/rust-version are all inherited via `workspace = true` (`crates/buzz-media/Cargo.toml:3-8`).

---

### 5. Compile-time constants (all values)

| Constant | Value | Applies to | file:line |
|---|---|---|---|
| `ALLOWED_MIME_TYPES` | `["image/jpeg", "image/png", "image/gif", "image/webp"]` | image upload allowlist | `crates/buzz-media/src/validation.rs:15` |
| `MP4_BRANDS` | `isom iso2 iso3 iso4 iso5 iso6 iso7 iso8 iso9 mp41 mp42 avc1 dash "M4V "` (14 brands) | ISO-BMFF brand acceptance | `crates/buzz-media/src/validation.rs:17-20` |
| `BLOCKED_FILE_MIME_TYPES` | 14 entries: `text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, `application/x-msdownload`, `application/x-executable`, `application/vnd.microsoft.portable-executable`, `application/x-mach-binary`, `application/x-sharedlib`, `application/x-elf`, `application/x-msi`, `application/vnd.android.package-archive`, `application/x-apple-diskimage` | generic-file deny-list | `crates/buzz-media/src/validation.rs:75-95` |
| `MAX_PIXELS` | `25_000_000` (25 MP; comment: "100MB max RGBA decode") | image bomb guard | `crates/buzz-media/src/validation.rs:269` |
| duration limit | `600.0` seconds (literal, not a named const) | video length | `crates/buzz-media/src/validation.rs:357` |
| resolution limit | `3840` × `2160` (literals) | video resolution | `crates/buzz-media/src/validation.rs:364` |
| `MAX_ATOMS` | `1024` | top-level MP4 atoms scanned in the moov-order check | `crates/buzz-media/src/validation.rs:413` |
| `MAX_BOXES` | `100_000` | total boxes walked in the MP4 metadata check | `crates/buzz-media/src/validation.rs:832` |
| `MAX_BOX_DEPTH` | `32` | MP4 box nesting depth | `crates/buzz-media/src/validation.rs:833` |
| `EMPTY_FFMPEG_UDTA` | 53-byte literal sequence (`meta`/`hdlr`/`mdirappl`/`ilst`) | the only permitted `udta` payload | `crates/buzz-media/src/validation.rs:834-839` |
| `FORBIDDEN` (MP4 boxes) | `meta ilst keys data uuid "xml " bxml loci \xa9xyz name chap` (11) | metadata box denial | `crates/buzz-media/src/validation.rs:840-852` |
| `CONTAINERS` (MP4) | `moov trak mdia minf stbl edts dinf sinf schi` (9) | boxes walked recursively | `crates/buzz-media/src/validation.rs:853-855` |
| `ALLOWED` (MP4 boxes) | 35 box types (`ftyp moov mdat free skip wide trak mdia minf stbl edts dinf sinf schi udta mvhd tkhd mdhd hdlr vmhd smhd dref "url " "urn " stsd stts stss ctts stsc stsz stco co64 sgpd sbgp elst`) | box allowlist | `crates/buzz-media/src/validation.rs:856-864` |
| `PNG_SNAPSHOT_KEYWORDS` | `["buzz_agent_snapshot", "buzz_team_snapshot"]` | permitted `tEXt` keywords | `crates/buzz-media/src/validation.rs:579` |
| thumbnail max dimension | `320` × `320` (literals, aspect preserved) | thumbnail size | `crates/buzz-media/src/thumbnail.rs:30` |
| blurhash components | `4`, `3` (x, y) | blurhash resolution | `crates/buzz-media/src/thumbnail.rs:37` |
| thumbnail format | `ImageFormat::Jpeg` | thumbnail encoding | `crates/buzz-media/src/thumbnail.rs:31` |
| `MIN_SNIFF_BYTES` | `4096` | leading bytes retained for MP4 magic detection during streaming | `crates/buzz-media/src/upload.rs:351` |
| video read buffer | `64 * 1024` (64 KiB) | streaming chunk size | `crates/buzz-media/src/upload.rs:353` |
| buffered auth window | `600` seconds | image/file Blossom `created_at` age | `crates/buzz-media/src/upload.rs:85` |
| video auth window | `3600` seconds | video Blossom `created_at` age | `crates/buzz-media/src/upload.rs:412` |
| clock-skew tolerance | `5` seconds (future `created_at`) | Blossom auth | `crates/buzz-media/src/auth.rs:117` |
| Blossom auth kind | `24242` | auth event kind check | `crates/buzz-media/src/auth.rs:42` |
| `BUF` | `8 * 1024 * 1024` (8 MiB) | `put_file` read buffer | `crates/buzz-media/src/storage.rs:91` |
| `UPLOAD_RECORD_VERSION` | `1` | upload-record schema version | `crates/buzz-media/src/upload_record.rs:52` |
| sidecar key prefix | `_meta/` | tenant metadata namespace | `crates/buzz-media/src/storage.rs:184` |
| record key prefix | `_uploads/` | moderation namespace | `crates/buzz-media/src/upload_record.rs:182` |
| thumb key suffix | `.thumb.jpg` | derived artifact naming | `crates/buzz-media/src/upload.rs:531` |
| sha256 hex length | `64`, charset `[0-9a-f]` | key classification | `crates/buzz-media/src/bucket_index.rs:75-79` |
| blob ext bounds | 1–8 ASCII alphanumeric (uppercase allowed) | key classification | `crates/buzz-media/src/bucket_index.rs:84-87` |
| ULID length | `26`, uppercase Crockford base32 | record key classification | `crates/buzz-media/src/bucket_index.rs:93-105` |
| UUID form | exactly 36 chars, lowercase hex + hyphens at 8/13/18/23 | community segment parsing | `crates/buzz-media/src/bucket_index.rs:112-127` |

No concurrency-limit constants exist in this crate (semaphore sizes live in the relay); the only caller-supplied bounds are the sweep `cap` and `max_keys` arguments (`crates/buzz-media/src/bucket_index.rs:377-379`, `crates/buzz-media/src/storage.rs:242-246`).

---

### 6. Verification: where does "50 MB" actually live?

ARCHITECTURE.md states "PUT `/media/upload` — Upload media blob (Blossom, 50 MB limit)" (`ARCHITECTURE.md:624`). In this crate, 50 MB is **not** a production constant:

| Occurrence | Context |
|---|---|
| `crates/buzz-media/src/config.rs:34` | doc comment only: "Maximum upload size for images (bytes). Default: 50 MB." — the field itself has no serde default (`crates/buzz-media/src/config.rs:41`) |
| `crates/buzz-media/src/upload.rs:573` | test fixture |
| `crates/buzz-media/src/validation.rs:952`, `:1590` | test fixtures |
| `crates/buzz-media/src/storage.rs:288` | test fixture |
| `crates/buzz-media/tests/static_creds_minio.rs:33` | ignored integration test fixture |

The only place 50 MB is a real runtime default is the relay: `BUZZ_MAX_IMAGE_BYTES` → `unwrap_or(50 * 1024 * 1024)` (`crates/buzz-relay/src/config.rs:626-629`). It also applies to **images only** — video (500 MB) and generic files (100 MB) have larger caps, and the relay's request-body limit layer uses `max(max_image_bytes, max_video_bytes)` (`crates/buzz-relay/src/router.rs:33-45`).
