## Module: buzz-media (`crates/buzz-media`)

### Aspect: Integrations

### 1. Dependency inventory (`crates/buzz-media/Cargo.toml:10-35`)

| Crate | Version / spec | Purpose in this crate |
|---|---|---|
| `buzz-core` | workspace | `tenant::{CommunityId, TenantContext, normalize_host}` (`crates/buzz-media/src/storage.rs:6`, `crates/buzz-media/src/auth.rs:170`) |
| `nostr` | workspace | `Event`, `PublicKey`, `Timestamp`, `ToBech32` for Blossom auth + records (`crates/buzz-media/src/auth.rs:33`, `crates/buzz-media/src/upload_record.rs:145`) |
| `s3` = `rust-s3` | **0.37**, `default-features = false`, features `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`); resolved **0.37.2** (`Cargo.lock:7432-7434`) | all object storage |
| `infer` | **0.19** | magic-byte MIME sniffing (`crates/buzz-media/src/validation.rs:239`, `:176`) |
| `image` | **0.25**, `default-features = false`, features `jpeg`, `png`, `gif`, `webp` | full decode + thumbnail JPEG encode (`crates/buzz-media/src/thumbnail.rs:26-32`) |
| `imagesize` | **0.14** | header-only dimension parse for the pixel-bomb guard (`crates/buzz-media/src/validation.rs:270`) |
| `blurhash` | **0.2** | 4×3 blurhash from the thumbnail (`crates/buzz-media/src/thumbnail.rs:36-37`) |
| `mp4` | **0.14** | MP4 header/track parsing (`crates/buzz-media/src/validation.rs:307-386`) |
| `sha2` + `hex` | workspace | SHA-256 content addressing (`crates/buzz-media/src/upload.rs:4`, `:84`, `:397`) |
| `tempfile` | **3** | `NamedTempFile` staging for streamed video (`crates/buzz-media/src/upload.rs:307`) |
| `tokio` | workspace | async fs/IO, `spawn_blocking` (`crates/buzz-media/src/upload.rs:79`, `:410`, `:418`) |
| `tokio-util` | **0.7**, feature `io` | `StreamReader` to adapt the axum body stream (`crates/buzz-media/src/upload.rs:325`) |
| `futures-util` / `futures-core` | **0.3** | `StreamExt::map`, stream trait bounds (`crates/buzz-media/src/storage.rs:141`, `crates/buzz-media/src/upload.rs:298`) |
| `bytes` | **1** | `Bytes` bodies / chunks (`crates/buzz-media/src/upload.rs:3`) |
| `axum` | workspace | only `StatusCode`/`IntoResponse` (`crates/buzz-media/src/error.rs:3-4`) and `http::HeaderName` validation (`crates/buzz-media/src/config.rs:118`) |
| `ulid` | **1** | upload-record ids (`crates/buzz-media/src/upload_record.rs:150`) |
| `uuid` | workspace | community UUID parsing in key classification (`crates/buzz-media/src/bucket_index.rs:22`, `:112-127`) |
| `chrono` | workspace | `Utc::now().timestamp()` for `uploaded_at` (`crates/buzz-media/src/upload.rs:113`, `:132`) |
| `serde` / `serde_json` | workspace | sidecar + record JSON (`crates/buzz-media/src/storage.rs:199`, `:218`) |
| `thiserror` | workspace | `MediaError`, `SweepError` (`crates/buzz-media/src/error.rs:7`, `crates/buzz-media/src/bucket_index.rs:340`) |
| `tracing` | workspace | 3 log sites (`crates/buzz-media/src/upload.rs:135`, `crates/buzz-media/src/error.rs:135`, `:155`) |

Dev-dependencies: `tokio` with `test-util` only (`crates/buzz-media/Cargo.toml:33-34`). No `mockall`, no `wiremock`, no `testcontainers`.

---

### 2. S3 client configuration

| Aspect | Behaviour | file:line |
|---|---|---|
| Region/endpoint | `Region::Custom { region: config.s3_region, endpoint: config.s3_endpoint }` — always a custom region, even for real AWS | `crates/buzz-media/src/storage.rs:35-38` |
| Path style | **Always** `.with_path_style()` — unconditional, not gated by a flag | `crates/buzz-media/src/storage.rs:66-68` |
| Bucket | `Bucket::new(&config.s3_bucket, region, creds)` | `crates/buzz-media/src/storage.rs:66` |
| TLS | `tokio-rustls-tls` feature (no native-tls) | `crates/buzz-media/Cargo.toml:24` |
| HTTP error strictness | `fail-on-err` feature — non-2xx responses surface as `S3Error` rather than being silently returned | `crates/buzz-media/Cargo.toml:24` |
| Signing region correctness | Documented requirement: `s3_region` must match the endpoint's region for real AWS, else SigV4 credential scope is wrong; default `us-east-1` preserves MinIO behaviour | `crates/buzz-media/src/config.rs:29-37`, `crates/buzz-media/src/config.rs:11-13` |

**MinIO compatibility** is achieved by (a) unconditional path-style addressing, (b) a custom `Region` with an explicit endpoint, and (c) tolerating a non-meaningful region value ("Defaults to 'us-east-1' to preserve MinIO/local behavior, where the value is not meaningfully checked" — `crates/buzz-media/src/config.rs:33-37`). The live round-trip test targets `http://localhost:9000` with creds `buzz_dev`/`buzz_dev_secret`, bucket `buzz-media` (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 3. Credential sources

Two mutually exclusive modes, chosen by whether both static keys are non-empty (`crates/buzz-media/src/storage.rs:39-65`):

| Case | Behaviour | file:line |
|---|---|---|
| both keys non-empty | `Credentials::new(Some(access), Some(secret), None, None, None)` — static keys, no environment/metadata access | `crates/buzz-media/src/storage.rs:44-50` |
| both keys empty | `Credentials::default()` — AWS default chain: environment, shared profile, **web-identity token (IRSA on EKS, `AssumeRoleWithWebIdentity`)**, container, instance metadata | `crates/buzz-media/src/storage.rs:51-56`, documented `crates/buzz-media/src/storage.rs:25-33` |
| exactly one key set | hard error `StorageError("s3_access_key and s3_secret_key must be configured together, or both empty to use the AWS credential chain")` — never silently falls back | `crates/buzz-media/src/storage.rs:57-62`; test `crates/buzz-media/src/storage.rs:312-331` |

**Patched `aws-creds` fork** (workspace-level, applies transitively to this crate): `Cargo.toml:170-171` pins `aws-creds` to `git+https://github.com/tlongwell-block/rust-s3` rev `c9fce3620dd434c1f810101d672cf384268dbb0f` (`Cargo.lock:422-424`). The reason recorded at `Cargo.toml:163-169`: aws-creds 0.39.1 cannot read **EKS Pod Identity** credentials (`AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE`), which the relay pod requires for S3 media + git storage; the fork adopts the aws-creds portion of `durch/rust-s3#449` (FULL_URI + token-file + Authorization header, refresh-safe, with a loopback allowlist for the auth token). Marked temporary — revert when #449 lands upstream.

No credential values are logged anywhere in the crate; the only `tracing` calls are the orphan-blob warning (`crates/buzz-media/src/upload.rs:135`) and the two error-mapping logs (`crates/buzz-media/src/error.rs:135`, `:155`).

---

### 4. Media-parsing integrations (in-process)

| Library | Where it runs | Input bound before it runs |
|---|---|---|
| `infer::get` | image path (`crates/buzz-media/src/validation.rs:239`), generic path (`crates/buzz-media/src/validation.rs:176`) | full buffered body |
| `imagesize::blob_size` | image path, before full decode (`crates/buzz-media/src/validation.rs:270`) | size cap already applied (`crates/buzz-media/src/validation.rs:259-268`) |
| `image::load_from_memory` (full pixel decode) | `generate_image_metadata_sync`, inside `spawn_blocking` (`crates/buzz-media/src/thumbnail.rs:26`, `crates/buzz-media/src/upload.rs:518-522`) | byte cap + 25 MP pixel cap |
| `image` thumbnail + JPEG encode | `crates/buzz-media/src/thumbnail.rs:30-32` | as above |
| `blurhash::encode` | `crates/buzz-media/src/thumbnail.rs:36-37` — failure swallowed via `unwrap_or_default()` | operates on the ≤320px thumbnail |
| `mp4::Mp4Reader::read_header` | `validate_video_file`, inside `spawn_blocking` (`crates/buzz-media/src/validation.rs:307`, `crates/buzz-media/src/upload.rs:416-419`) | byte cap, ISO-BMFF check, moov-order scan, box allowlist walk |

Custom hand-rolled parsers (no external crate): JPEG marker walk, PNG chunk walk, WebP RIFF walk, GIF block walk, MP4 box walk, top-level atom scan — `crates/buzz-media/src/validation.rs:502-928`.

---

### 5. Retry / timeout behaviour

| Aspect | Finding | file:line |
|---|---|---|
| Retries on S3 failure | **None** in this crate — every storage call maps the first error straight to `MediaError` | `crates/buzz-media/src/storage.rs:73-265` |
| Timeouts on S3 calls | **None set** in this crate; whatever `rust-s3`/`reqwest` defaults apply | `crates/buzz-media/src/storage.rs:34-70` (no timeout config on the client) |
| Sweep timeout | Only a *variant* to represent it — `SweepError::Timeout(Duration)`, constructed by the relay's sweep task which wraps the fold in `tokio::time::timeout` | `crates/buzz-media/src/bucket_index.rs:349-357` |
| Sweep object cap | Enforced by the caller-supplied `cap`, checked before folding each page | `crates/buzz-media/src/bucket_index.rs:394-398` |
| Listing page size | Caller-supplied `max_keys`, bounds one HTTP response only | `crates/buzz-media/src/storage.rs:236-241` |

---

### 6. Error handling on storage failures

| Situation | Result | file:line |
|---|---|---|
| Any `S3Error` | `MediaError::StorageError(e.to_string())` via `From` | `crates/buzz-media/src/error.rs:94-98` |
| 404 on `get`/`get_range` | `MediaError::NotFound` (special-cased on `HttpFailWithBody(404, _)`) | `crates/buzz-media/src/storage.rs:106-110`, `:119-123` |
| 404 on `head`/`head_with_metadata` | `Ok(false)` / `Ok(None)` — not an error | `crates/buzz-media/src/storage.rs:150-154`, `:168-174` |
| 404 on `get_stream` | checked via `response.status_code == 404` (different mechanism than the others) | `crates/buzz-media/src/storage.rs:136-139` |
| Sidecar read failure vs absence | Deliberately collapsed to `None` in `read_sidecar_mime` so a cross-community request cannot distinguish a foreign blob from a missing one | `crates/buzz-media/src/storage.rs:222-233` |
| Sidecar JSON parse failure | `MediaError::StorageError` (via `From<serde_json::Error>`) | `crates/buzz-media/src/error.rs:100-104` |
| `StorageError`/`Io`/`Internal` → HTTP | logged at `error` level, response body flattened to `"internal error"` | `crates/buzz-media/src/error.rs:154-158` |
| Blob PUT succeeded but metadata failed | Error propagates; blob intentionally left orphaned (no compensating delete) | `crates/buzz-media/src/upload.rs:131-141` |
| Upload-record PUT failed | Error propagates and the upload fails **before** the sidecar publish, so media is never served unscanned | `crates/buzz-media/src/upload.rs:154-172`, `crates/buzz-media/src/upload_record.rs:132-138` |
| `spawn_blocking` join failure | `MediaError::Internal` | `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:418-419`, `:523-524` |
| Axum body-limit error inside the video stream | Detected by matching three Display substrings (`length limit`, `body limit`, `LengthLimitError`), converted to `ErrorKind::WriteZero` → `FileTooLarge` (413 instead of 500) | `crates/buzz-media/src/upload.rs:328-341`, `crates/buzz-media/src/upload.rs:358-366` |

---

### 7. Consumers / integration boundary

| Consumer | How it integrates | file:line |
|---|---|---|
| `buzz-relay` HTTP handlers | Owns routes `PUT /upload`, `PUT /media/upload`, `GET|HEAD /media/{sha256_ext}` and the `RequestBodyLimitLayer` | `crates/buzz-relay/src/router.rs:33-45` |
| `buzz-relay` upload dispatch | Sniffs 4096 bytes, uses `buzz_media::looks_like_iso_bmff` to route to the video pipeline, else buffers and picks image vs generic-file path | `crates/buzz-relay/src/api/media.rs:47-51`, `:317-399` |
| `buzz-relay` read path | `read_sidecar_mime`, `get_stream`, `get_range`, and `buzz_media::serve_inline` for `Content-Disposition` | `crates/buzz-relay/src/api/media.rs:633-740` |
| `buzz-relay` config | Constructs `MediaConfig` from env; also depends on `rust-s3` 0.37 directly | `crates/buzz-relay/src/config.rs:614-657`, `crates/buzz-relay/Cargo.toml:65` |
| `buzz-relay` storage sweep | Supplies the `fetch_page` closure over `MediaStorage::list_page` | `crates/buzz-media/src/bucket_index.rs:11-14` (documented), `crates/buzz-media/src/storage.rs:235-241` |
| `buzz-moderation` (external consumer) | Triggers on S3 `ObjectCreated` under `_uploads/` and parses `UploadRecord` instead of HEADing blobs | `crates/buzz-media/src/upload_record.rs:29-48` |
| `buzz-test-client` | Also depends on `rust-s3` 0.37 for E2E media tests | `crates/buzz-test-client/Cargo.toml:38` |
