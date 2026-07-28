## Module: buzz-media (`crates/buzz-media`)

### Aspect: Data Model

Scope: all 11 `src/*.rs` files + `tests/static_creds_minio.rs` read in full. The crate has **no database access** — `buzz-db` is not a dependency (`crates/buzz-media/Cargo.toml:11-35`). All persistence is S3 objects. Tenancy types (`CommunityId`, `TenantContext`) come from `buzz-core` (`crates/buzz-media/src/storage.rs:6`).

---

### 1. Public types

#### `BlobDescriptor` — Blossom BUD-02 upload response (`crates/buzz-media/src/types.rs:7-30`)

Derives `Debug, Clone, Serialize, Deserialize` (`crates/buzz-media/src/types.rs:6`).

| Field | Type | Serde | Source |
|---|---|---|---|
| `url` | `String` | — | `crates/buzz-media/src/types.rs:9` |
| `sha256` | `String` (64 hex) | — | `crates/buzz-media/src/types.rs:11` |
| `size` | `u64` | — | `crates/buzz-media/src/types.rs:13` |
| `mime_type` | `String` | `rename = "type"` | `crates/buzz-media/src/types.rs:15-16` |
| `uploaded` | `i64` (unix secs) | — | `crates/buzz-media/src/types.rs:18` |
| `dim` | `Option<String>` (`"WxH"`) | `skip_serializing_if = "Option::is_none"` | `crates/buzz-media/src/types.rs:20-21` |
| `blurhash` | `Option<String>` | skip-if-none | `crates/buzz-media/src/types.rs:23-24` |
| `thumb` | `Option<String>` (URL) | skip-if-none | `crates/buzz-media/src/types.rs:26-27` |
| `duration` | `Option<f64>` (secs, video only) | skip-if-none | `crates/buzz-media/src/types.rs:29-30` |

Empty-string `BlobMeta` fields are converted to `None` when building the descriptor, so they are omitted from JSON rather than serialized as `""` (`crates/buzz-media/src/upload.rs:539-560`).

#### `BlobMeta` — persisted sidecar JSON (`crates/buzz-media/src/storage.rs:385-403`)

Derives `Debug, Clone, Default, Serialize, Deserialize` (`crates/buzz-media/src/storage.rs:384`). Stored at `_meta/{community}/{sha256}.json` (`crates/buzz-media/src/storage.rs:183-185`).

| Field | Type | Serde | Notes |
|---|---|---|---|
| `dim` | `String` | — | `"WxH"`; empty for generic files (`crates/buzz-media/src/upload.rs:265`) |
| `blurhash` | `String` | — | empty for video/files |
| `thumb_url` | `String` | — | empty for video/files |
| `ext` | `String` | — | canonical extension, e.g. `jpg` |
| `mime_type` | `String` | — | sniffed MIME |
| `size` | `u64` | — | byte length |
| `uploaded_at` | `i64` | `#[serde(default)]` | `crates/buzz-media/src/storage.rs:399-400` |
| `duration_secs` | `Option<f64>` | `default`, skip-if-none | `crates/buzz-media/src/storage.rs:401-403` |

#### `BlobHeadMeta` — HEAD result (`crates/buzz-media/src/storage.rs:379-381`)

| Field | Type | Source |
|---|---|---|
| `size` | `u64` (from S3 `content_length`, `unwrap_or(0)`) | `crates/buzz-media/src/storage.rs:167-172` |

Note: no derives on this struct (`crates/buzz-media/src/storage.rs:378-381`).

#### `MediaStorage` — the single storage backend type (`crates/buzz-media/src/storage.rs:19-21`)

| Field | Type | Visibility |
|---|---|---|
| `bucket` | `Box<s3::Bucket>` | private |

There is **no storage trait** in this crate — `MediaStorage` is a concrete struct over `rust-s3`'s `Bucket`; the only pluggable seam is the page-fetching closure in `fold_bucket_listing` (`crates/buzz-media/src/bucket_index.rs:377-383`).

#### `ByteStream` — streaming read alias (`crates/buzz-media/src/storage.rs:16`)

`pub type ByteStream = Pin<Box<dyn futures_core::Stream<Item = Result<Bytes, MediaError>> + Send>>`

#### `MediaConfig` (`crates/buzz-media/src/config.rs:17-62`)

Derives `Debug, Clone, serde::Deserialize` (`crates/buzz-media/src/config.rs:16`). Full field list in the Configuration aspect; fields: `s3_endpoint`, `s3_access_key`, `s3_secret_key`, `s3_bucket`, `s3_region` (`String`); `max_image_bytes`, `max_gif_bytes`, `max_video_bytes`, `max_file_bytes` (`u64`); `public_base_url` (`String`); `upload_records_enabled` (`bool`); `upload_ip_header`, `upload_port_header` (`Option<String>`).

#### `VideoMeta` — MP4 parse result (`crates/buzz-media/src/validation.rs:222-231`)

| Field | Type | Source |
|---|---|---|
| `duration_secs` | `f64` (from `mvhd`/track timescale, not edit lists) | `crates/buzz-media/src/validation.rs:224` |
| `width` | `u32` | `crates/buzz-media/src/validation.rs:226` |
| `height` | `u32` | `crates/buzz-media/src/validation.rs:228` |
| `has_audio` | `bool` | `crates/buzz-media/src/validation.rs:230` |

#### `BlossomVerb` — auth verb enum (`crates/buzz-media/src/auth.rs:6-10`)

`Upload` → `"upload"`, `Get` → `"get"` (`crates/buzz-media/src/auth.rs:13-18`). Derives `Debug, Clone, Copy, PartialEq, Eq`.

#### `MediaError` — 35 variants (`crates/buzz-media/src/error.rs:8-86`)

`thiserror::Error`; full variant table in the Conventions aspect.

#### Upload-record types (`crates/buzz-media/src/upload_record.rs`)

`UploadRecord` (`crates/buzz-media/src/upload_record.rs:56-97`), persisted at `_uploads/{community}/{sha256}/{event_id}.json`:

| Field | Type | Serde |
|---|---|---|
| `version` | `u32` (= `UPLOAD_RECORD_VERSION` = 1, `crates/buzz-media/src/upload_record.rs:52`) | — |
| `event_id` | `String` (ULID) | — |
| `sha256` | `String` | — |
| `ext` | `String` | — |
| `mime_type` | `String` | — |
| `size` | `u64` | — |
| `uploaded_at` | `i64` | — |
| `community_id` | `String` (UUID) | — |
| `community_host` | `String` | — |
| `uploader_id` | `String` (hex pubkey) | — |
| `uploader_npub` | `String` (bech32) | — |
| `uploader_name` | `Option<String>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:85-86`) |
| `ip` | `Option<String>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:89-90`) |
| `port` | `Option<u16>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:94-95`) |

`UploadNetworkInfo` (`crates/buzz-media/src/upload_record.rs:100-105`): `ip: Option<IpAddr>`, `port: Option<u16>`; derives `Debug, Clone, Copy, Default, PartialEq, Eq`.

`UploadAttribution` (`crates/buzz-media/src/upload_record.rs:110-115`): `uploader_name: Option<String>`, `net: UploadNetworkInfo`; derives `Debug, Clone, Default`.

`UploadEventFacts<'a>` (`crates/buzz-media/src/upload_record.rs:119-130`): `sha256: &'a str`, `ext: &'a str`, `mime: &'a str`, `size: u64`, `uploaded_at: i64`; derives `Debug, Clone, Copy`.

#### Bucket-sweep types (`crates/buzz-media/src/bucket_index.rs`)

`KeyClass` enum (`crates/buzz-media/src/bucket_index.rs:34-49`), derives `Debug, Clone, PartialEq, Eq`:

| Variant | Payload | Key shape |
|---|---|---|
| `Thumb` | `sha256: String` | `{sha256}.thumb.jpg` |
| `Blob` | `sha256: String, ext: String` | `{sha256}.{ext}` |
| `Sidecar` | `community: Uuid, sha256: String` | `_meta/{community}/{sha256}.json` |
| `Auxiliary` | `community: Uuid, sha256: String, event_id: String` | `_uploads/{community}/{sha256}/{ulid}.json` |
| `Unknown` | — | everything else |

`CommunityStorage` (`crates/buzz-media/src/bucket_index.rs:197-200`): `bytes: u64`, `objects: u64`.

`BucketSnapshot` (`crates/buzz-media/src/bucket_index.rs:206-229`): `physical_bytes`, `physical_objects`, `logical_bytes`, `logical_objects` (`u64`), `per_community: HashMap<Uuid, CommunityStorage>`, `orphan_blob_bytes`, `orphan_blob_count`, `orphan_sidecar_count`, `multi_variant_shas`, `multi_variant_bytes`, `unknown_key_bytes`, `unknown_key_objects` (all `u64`).

`BucketAggregate` (`crates/buzz-media/src/bucket_index.rs:232-246`) — private fields: `blob_variant_bytes: HashMap<String, Vec<u64>>`, `thumb_bytes: HashMap<String, u64>`, `sidecar_bindings: HashMap<(Uuid, String), u64>`, plus `physical_bytes`, `physical_objects`, `unknown_bytes`, `unknown_objects`.

`Page` (`crates/buzz-media/src/bucket_index.rs:364-368`): `objects: Vec<(String, u64)>`, `next_continuation_token: Option<String>`, `is_truncated: bool`.

`SweepError` (`crates/buzz-media/src/bucket_index.rs:341-362`): `CapExceeded { seen, cap }`, `Storage(#[from] MediaError)`, `Timeout(Duration)`, `MalformedPage`.

#### Private pipeline types (`crates/buzz-media/src/upload.rs`)

`BufferedUploadInput<'a>` (`crates/buzz-media/src/upload.rs:45-52`): `storage`, `config`, `ctx`, `auth_event`, `body: Bytes`, `attribution: Option<UploadAttribution>`.
`MetadataInput` (`crates/buzz-media/src/upload.rs:195-201`): `sha256`, `ext`, `mime` (`String`), `body: Bytes`, `uploaded_at: i64`.

---

### 2. Content-addressing scheme

| Element | Value | file:line |
|---|---|---|
| Hash algorithm | SHA-256, lowercase hex | `crates/buzz-media/src/upload.rs:84` (buffered), `crates/buzz-media/src/upload.rs:397` (streamed) |
| Hash crate | `sha2::Sha256` + `hex::encode` | `crates/buzz-media/src/upload.rs:4`, `crates/buzz-media/Cargo.toml:19-20` |
| Blob key | `{sha256}.{ext}` | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/upload.rs:421` |
| Thumbnail key | `{sha256}.thumb.jpg` | `crates/buzz-media/src/upload.rs:531` |
| Sidecar key | `_meta/{community_uuid}/{sha256}.json` | `crates/buzz-media/src/storage.rs:183-185` |
| Upload-record key | `_uploads/{community_uuid}/{sha256}/{ulid}.json` | `crates/buzz-media/src/upload_record.rs:181-183` |
| Public blob URL | `{public_base_url}/{sha256}.{ext}` | `crates/buzz-media/src/upload.rs:548` |
| Public thumb URL | `{public_base_url}/{sha256}.thumb.jpg` | `crates/buzz-media/src/thumbnail.rs:44` |
| ULID source | `ulid::Ulid::new().to_string()` (uppercase Crockford base32) | `crates/buzz-media/src/upload_record.rs:150` |

Raw blob bytes are **globally shared CAS** (no community segment); the *sidecar* carries the tenant scoping and is described in code as "the tenant read gate" (`crates/buzz-media/src/storage.rs:177-182`). Two communities uploading identical bytes share one blob object but get distinct sidecars (`crates/buzz-media/src/storage.rs:352-377`).

Extension is derived from the sniffed MIME, never from a client filename: images via `mime_to_ext` (`crates/buzz-media/src/validation.rs:930-939`), generic files via `file_mime_to_ext` with fallback to `infer`'s extension then `"bin"` (`crates/buzz-media/src/validation.rs:99-156`, `crates/buzz-media/src/validation.rs:203-206`), video hardcoded `"mp4"` (`crates/buzz-media/src/upload.rs:420`).

---

### 3. Persisted metadata inventory (S3 objects only)

| Object | Payload type | Content-Type | Writer |
|---|---|---|---|
| `{sha}.{ext}` | raw uploaded bytes | sniffed MIME | `crates/buzz-media/src/upload.rs:129`, `crates/buzz-media/src/upload.rs:434` |
| `{sha}.thumb.jpg` | JPEG thumbnail (≤320px) | `image/jpeg` | `crates/buzz-media/src/upload.rs:531-532` |
| `_meta/{community}/{sha}.json` | `BlobMeta` JSON | `application/json` | `crates/buzz-media/src/storage.rs:210-221` |
| `_uploads/{community}/{sha}/{ulid}.json` | `UploadRecord` JSON | `application/json` | `crates/buzz-media/src/upload_record.rs:174-176` |

No DB rows, no Redis keys, no local filesystem persistence (temp files are deleted after the S3 put — `crates/buzz-media/src/upload.rs:435`).
