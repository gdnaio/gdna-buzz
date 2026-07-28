## Module: buzz-media (`crates/buzz-media`)

### Aspect: API Surface

Library crate only — "no Axum dependency for handlers. Axum handlers live in `buzz-relay`" (`crates/buzz-media/src/lib.rs:3`). Axum *is* a dependency, but only for `StatusCode`/`IntoResponse` (`crates/buzz-media/src/error.rs:3-4`) and `HeaderName` validation (`crates/buzz-media/src/config.rs:118`).

Modules, all `pub`: `auth`, `bucket_index`, `config`, `error`, `storage`, `thumbnail`, `types`, `upload`, `upload_record`, `validation` (`crates/buzz-media/src/lib.rs:5-15`). Re-exports at `crates/buzz-media/src/lib.rs:17-28`.

---

### 1. Upload pipeline (`upload.rs`)

| Fn | Operation | Storage backend calls | Return |
|---|---|---|---|
| `process_upload(storage, config, ctx, auth_event, body: Bytes, attribution)` (`crates/buzz-media/src/upload.rs:207-236`) | image path: sniff+validate → sha256 → Blossom auth (600 s window) → idempotency → put blob → thumbnail+blurhash → optional record → put sidecar | `head` ×2, `get_sidecar`, `put` (blob), `put` (thumb), `put` (record), `put_sidecar` | `Result<BlobDescriptor, MediaError>` |
| `process_file_upload(... body: Bytes, attribution)` (`crates/buzz-media/src/upload.rs:245-274`) | generic-file path: deny-list validate → sha256 → auth (600 s) → idempotency → put blob → minimal sidecar | same minus thumbnail | `Result<BlobDescriptor, MediaError>` |
| `process_video_upload(... body_stream, content_length, attribution)` (`crates/buzz-media/src/upload.rs:292-511`) | streaming path: stream→tempfile + incremental sha256 → ISO-BMFF check → auth (3600 s) → full MP4 validation → idempotency → `put_file` → sidecar with `duration_secs` | `head` ×2, `get_sidecar`, `put_file`, `put` (record), `put_sidecar` | `Result<BlobDescriptor, MediaError>` |
| `process_buffered_upload` (private, `crates/buzz-media/src/upload.rs:54-192`) | shared skeleton for the two buffered paths, parameterised by a `validate` closure and a `prepare_metadata` future | — | `Result<BlobDescriptor, MediaError>` |
| `prepare_image_metadata` (private, `crates/buzz-media/src/upload.rs:513-537`) | `spawn_blocking` thumbnail/blurhash then thumb PUT | `put` | `Result<BlobMeta, MediaError>` |
| `build_descriptor` (private, `crates/buzz-media/src/upload.rs:539-560`) | pure descriptor assembly; empty strings → `None` | none | `BlobDescriptor` |

`process_video_upload`'s stream parameter type: `impl futures_core::Stream<Item = Result<Bytes, axum::Error>> + Send + 'static` (`crates/buzz-media/src/upload.rs:298`).

---

### 2. Storage backend — `MediaStorage` (`storage.rs`)

Single backend: **S3-compatible object storage via `rust-s3`**, always path-style (`crates/buzz-media/src/storage.rs:66-68`). There is **no trait**, and no local-filesystem or in-memory backend implementation in this crate. Test doubles exist only for the sweep fold, via a closure (`crates/buzz-media/src/bucket_index.rs:377-383`).

| Method | Operation | `rust-s3` call | Return |
|---|---|---|---|
| `new(&MediaConfig)` (`crates/buzz-media/src/storage.rs:34-70`) | build client; static creds vs AWS chain | `Credentials::new` / `Credentials::default`, `Bucket::new(..).with_path_style()` | `Result<Self, MediaError>` |
| `put(key, bytes, content_type)` (`crates/buzz-media/src/storage.rs:73-79`) | write from slice | `put_object_with_content_type` | `Result<(), MediaError>` |
| `put_file(key, path, content_type)` (`crates/buzz-media/src/storage.rs:85-103`) | streamed write from disk, 8 MiB `BufReader` | `put_object_stream_with_content_type` | `Result<(), MediaError>` |
| `get(key)` (`crates/buzz-media/src/storage.rs:105-111`) | full read into `Vec<u8>` | `get_object`; 404 → `NotFound` | `Result<Vec<u8>, MediaError>` |
| `get_range(key, start, end)` (`crates/buzz-media/src/storage.rs:118-124`) | inclusive-range read (HTTP 206 support) | `get_object_range` | `Result<Vec<u8>, MediaError>` |
| `get_stream(key)` (`crates/buzz-media/src/storage.rs:131-146`) | chunked read, never buffered | `get_object_stream`; `status_code == 404` → `NotFound` | `Result<ByteStream, MediaError>` |
| `head(key)` (`crates/buzz-media/src/storage.rs:149-155`) | existence check | `head_object`; 404 → `Ok(false)` | `Result<bool, MediaError>` |
| `delete(key)` (`crates/buzz-media/src/storage.rs:158-164`) | delete object | `delete_object` | `Result<(), MediaError>` |
| `head_with_metadata(key)` (`crates/buzz-media/src/storage.rs:167-175`) | HEAD → size only | `head_object` | `Result<Option<BlobHeadMeta>, MediaError>` |
| `sidecar_key(community, sha256)` (`crates/buzz-media/src/storage.rs:183-185`) | key builder (assoc fn) | none | `String` |
| `ctx_sidecar_key(ctx, sha256)` (`crates/buzz-media/src/storage.rs:188-190`) | key builder from tenant | none | `String` |
| `get_sidecar(ctx, sha256)` (`crates/buzz-media/src/storage.rs:193-202`) | read + deserialize `BlobMeta` | `get_object` | `Result<BlobMeta, MediaError>` |
| `put_sidecar(ctx, sha256, meta)` (`crates/buzz-media/src/storage.rs:210-221`) | serialize + write sidecar | `put` → `put_object_with_content_type` | `Result<(), MediaError>` |
| `read_sidecar_mime(ctx, sha256_ext)` (`crates/buzz-media/src/storage.rs:226-233`) | MIME-only convenience; absence and failure both → `None` | `get_object` | `Option<String>` |
| `list_page(continuation_token, max_keys)` (`crates/buzz-media/src/storage.rs:242-265`) | one listing page, converted to storage-agnostic `Page` | `list_page` (manual pagination, not auto-paginating `list`) | `Result<Page, MediaError>` |

---

### 3. Validation (`validation.rs`)

| Fn | Operation | Return |
|---|---|---|
| `validate_content(bytes, config)` (`crates/buzz-media/src/validation.rs:238-274`) | image path: sniff → allowlist → size cap → metadata-free structural check → pixel-count cap | `Result<String, MediaError>` (MIME) |
| `validate_file_content(bytes, config)` (`crates/buzz-media/src/validation.rs:159-209`) | generic path: size cap → ISO-BMFF reject → sniff → reject image/video/audio → deny-list → ext mapping | `Result<(String, String), MediaError>` (mime, ext) |
| `validate_video_file(path, config)` (`crates/buzz-media/src/validation.rs:289-395`) | on-disk MP4: moov-before-mdat, box allowlist, brand, single H.264 track, ≤1 AAC track, duration, resolution | `Result<VideoMeta, MediaError>` |
| `looks_like_iso_bmff(bytes)` (`crates/buzz-media/src/validation.rs:48-50`) | structural `ftyp` box detection | `bool` |
| `looks_like_mp4_iso_bmff(bytes)` — `pub(crate)` (`crates/buzz-media/src/validation.rs:52-62`) | `ftyp` + MP4 brand match (major or compatible) | `bool` |
| `serve_inline(mime)` (`crates/buzz-media/src/validation.rs:216-218`) | inline-vs-attachment policy: `image/*` or `video/*` | `bool` |
| `mime_to_ext(mime)` (`crates/buzz-media/src/validation.rs:930-939`) | MIME → extension, fallback `"bin"` | `&'static str` |

Private validators: `iso_bmff_ftyp_payload` (`:22`), `file_mime_to_ext` (`:99`), `check_moov_before_mdat` (`:408`), `validate_image_metadata_free` (`:492`), `validate_jpeg_metadata_free` (`:502`), `is_snapshot_text_chunk` (`:584`), `validate_png_metadata_free` (`:592`), `validate_webp_metadata_free` (`:659`), `validate_gif_metadata_free` (`:734`), `validate_mp4_metadata_free` (`:831`).

---

### 4. Auth (`auth.rs`)

| Fn | Operation | Return |
|---|---|---|
| `verify_blossom_auth_event_for_verb(event, verb, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:31-145`) | signature, kind 24242, non-empty content, `t` verb, `expiration` future, `created_at` window, `server` tag match | `Result<(), MediaError>` |
| `verify_blossom_auth_event(event, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:147-152`) | upload-shaped wrapper over the above | `Result<(), MediaError>` |
| `verify_blossom_upload_auth(event, sha256, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:175-198`) | above + at least one `x` tag == `sha256` | `Result<(), MediaError>` |
| `verify_blossom_get_auth(event, sha256, server_domain, max_age_secs)` (`crates/buzz-media/src/auth.rs:207-236`) | verb `get` + (`x` match OR `server` match), else `InsufficientScope` | `Result<(), MediaError>` |
| `normalize_server_host` (private, `crates/buzz-media/src/auth.rs:163-170`) | strip scheme/path → `buzz_core::tenant::normalize_host` | `String` |

---

### 5. Thumbnails (`thumbnail.rs`)

| Fn | Operation | Return |
|---|---|---|
| `generate_image_metadata_sync(config, sha256, bytes, mime, ext)` (`crates/buzz-media/src/thumbnail.rs:15-50`) | sync/CPU-bound: full decode, 320px thumbnail, JPEG encode, 4×3 blurhash; non-`image/*` → `(BlobMeta::default(), None)` | `Result<(BlobMeta, Option<Vec<u8>>), MediaError>` |

---

### 6. Upload records (`upload_record.rs`)

| Fn | Operation | Storage call | Return |
|---|---|---|---|
| `record_upload_event(storage, ctx, uploader, attribution, facts)` (`crates/buzz-media/src/upload_record.rs:139-178`) | build `UploadRecord`, ULID id, write JSON | `put` | `Result<(), MediaError>` |
| `upload_record_key(ctx, sha256, event_id)` (`crates/buzz-media/src/upload_record.rs:181-183`) | key builder | none | `String` |
| `parse_public_ip(raw)` (`crates/buzz-media/src/upload_record.rs:191-194`) | parse + public-range filter (fail-empty) | none | `Option<IpAddr>` |
| `parse_port(raw)` (`crates/buzz-media/src/upload_record.rs:197-199`) | non-zero u16 parse | none | `Option<u16>` |
| `is_public_ip` (private, `crates/buzz-media/src/upload_record.rs:207-256`) | explicit reserved-range enumeration (v4 + v6) | none | `bool` |
| `UPLOAD_RECORD_VERSION` const = 1 (`crates/buzz-media/src/upload_record.rs:52`) | — | — | `u32` |

---

### 7. Bucket sweep (`bucket_index.rs`)

| Fn / method | Operation | Return |
|---|---|---|
| `classify_key(key)` (`crates/buzz-media/src/bucket_index.rs:54-72`) | thumb → blob → sidecar → auxiliary → unknown | `KeyClass` |
| `BucketAggregate::fold(&mut self, key, size)` (`crates/buzz-media/src/bucket_index.rs:250-274`) | incremental per-sha/per-binding accumulation | `()` |
| `BucketAggregate::finish(self)` (`crates/buzz-media/src/bucket_index.rs:277-337`) | orphan/multi-variant/per-community computation | `BucketSnapshot` |
| `fold_bucket_listing(cap, fetch_page)` (`crates/buzz-media/src/bucket_index.rs:377-411`) | paginate with cap check before folding each page | `Result<BucketSnapshot, SweepError>` |

`fetch_page: FnMut(Option<String>) -> Future<Output = Result<Page, MediaError>>` — the injection point production code fills with `MediaStorage::list_page` (`crates/buzz-media/src/bucket_index.rs:11-14`).

---

### 8. Config + error surface

| Item | Signature | file:line |
|---|---|---|
| `MediaConfig::validate(&self)` | `Result<(), String>` | `crates/buzz-media/src/config.rs:66-122` |
| `impl From<image::ImageError> for MediaError` | → `InvalidImage` | `crates/buzz-media/src/error.rs:88-92` |
| `impl From<s3::error::S3Error> for MediaError` | → `StorageError(String)` | `crates/buzz-media/src/error.rs:94-98` |
| `impl From<serde_json::Error> for MediaError` | → `StorageError(String)` | `crates/buzz-media/src/error.rs:100-104` |
| `impl IntoResponse for MediaError` | HTTP status mapping + JSON `{"error": …}` | `crates/buzz-media/src/error.rs:106-160` |

---

### 9. Consumers (boundary, outside this crate)

Relay routes that call into this crate: `PUT /upload`, `PUT /media/upload`, `GET|HEAD /media/{sha256_ext}` (`crates/buzz-relay/src/router.rs:38-45`). The relay selects the pipeline by sniffing the first 4096 bytes and calling `looks_like_iso_bmff` (`crates/buzz-relay/src/api/media.rs:47-51`, `crates/buzz-relay/src/api/media.rs:334-399`), and applies `serve_inline` for `Content-Disposition` (`crates/buzz-relay/src/api/media.rs:663`).
