# LocalSend protocol v2 (2.2): implementer's spec

Audience: developers re-implementing LocalSend protocol v2 wire-compatibly (HyperShare port).
Reference: Rust crate `localsend` in `packages/core` at upstream `cfd6991`. Every statement below was read from that code
(paths are relative to `packages/core/` unless they start with `app/` or `packages/localsend_isolates/`). Where the
Flutter app (the real peer behaviour) adds rules on top of the core, it says so.

Scope: HTTP API v2 (server + client), DTOs, TLS identity, PIN, sessions, text messages, upload/download, limits. UDP
multicast and the subnet scan are covered only as far as they touch the HTTP API (section 11). v3 (`/api/localsend/v3/*`,
nonce exchange, `crypto/token.rs`) is experimental and out of scope.

---

## 1. Transport

| Aspect | Value | Source |
|---|---|---|
| Default port | `53317` (TCP for HTTP, same number for UDP multicast) | `src/multicast/mod.rs:39`, `packages/localsend_isolates/lib/constants.dart:14` |
| Listeners | `0.0.0.0:<port>` plus `[::]:<port>` with `IPV6_V6ONLY`; IPv6 failure is non-fatal | `src/http/server/mod.rs:230-245`, `:305-318` |
| HTTP version | **HTTP/1.1 only** (hyper `http1::Builder`); HTTP/2 preface is refused | `src/http/server/mod.rs:533-554`, `tests/connections.rs:58` |
| Client ALPN | `http/1.1` only | `src/http/client/mod.rs:228-230` |
| Schemes | `https` (default in the app) or `http` ("encryption off") | `src/model/discovery.rs:17-31` |
| Base path | `/api/localsend/v2` | `src/http/client/url.rs:22-40` |
| IPv6 URLs | `[addr]`; scoped link-local `fe80::1%3` becomes host `fe80--1s3.scoped.localsend.internal` resolved by a custom resolver (client-internal trick, not on the wire) | `src/http/client/url.rs:27-33`, `src/http/client/scoped_host.rs` |
| Query encoding | `application/x-www-form-urlencoded` (form_urlencoded) on both sides; `+` decodes to space | `src/http/server/common/query.rs`, `src/http/client/url.rs:41-48` |
| Proxies/redirects | client never uses a proxy and never follows redirects | `src/http/client/mod.rs:235-242` |

## 2. Endpoint table (server routing)

Routing is an exact `(method, path)` match in `src/http/server/mod.rs:628-692`. Anything else returns **404 with an
empty body** (`:687-691`). Routes guarded by "v2 enabled" return 404 **with a JSON error body** when the v2 config is absent.

| Method | Path | Query | Body | Purpose |
|---|---|---|---|---|
| POST | `/api/localsend/v2/register` | – | `RegisterDtoV2` | discovery handshake (answer to multicast / scan) |
| GET | `/api/localsend/v2/info` **and** `/api/localsend/v1/info` | – | – | device info (v1 path kept for app <= 1.17 probing) |
| POST | `/api/localsend/v2/prepare-upload` | `pin?` | `PrepareUploadRequestDtoV2` | ask receiver to accept files |
| POST | `/api/localsend/v2/upload` | `sessionId`, `fileId`, `token` | raw file bytes | upload one file |
| POST | `/api/localsend/v2/cancel` | `sessionId?` | – | sender (or receiver) cancels |
| POST | `/api/localsend/v2/prepare-download` | `sessionId?`, `pin?` | – | "web share"/download API: list files |
| GET | `/api/localsend/v2/download` | `sessionId`, `fileId` | – | download one shared file |
| POST | `/api/localsend/v2/show` | `token` | `{"args":[..]}` or empty | app-internal: foreground the running instance |
| GET | `/` | – | – | browser page (download page, upload page, or 403 page) |
| GET | `/i18n.json` | – | – | strings for the browser pages |
| POST | `/api/localsend/v3/nonce`, `/api/localsend/v3/register` | | | v3, ignore |

v1 transfer routes (`/api/localsend/v1/send-request`, `/v1/send`) are **not served**. The Dart route table still knows
them (`packages/localsend_isolates/lib/api_route_builder.dart:9-21`) and a client uses v1 paths only when the peer reports
`version == "1.0"`.

## 3. DTOs (exact JSON)

All DTOs are `camelCase`. Unknown fields are ignored (serde default, no `deny_unknown_fields`). "skip if None" means the
key is omitted, not sent as `null`. Parsing failure of any JSON body => **400 `{"message":"Invalid JSON body"}`**
(`src/http/server/common/collect_to_json.rs:12-16`).

### 3.1 Enums

| Field | Wire values | Parse rule | Source |
|---|---|---|---|
| `protocol` | `"http"` \| `"https"` | strict, lowercase only; anything else => 400 | `src/model/discovery.rs:36-60` |
| `deviceType` | `"mobile"`, `"desktop"`, `"web"`, `"headless"`, `"server"` | case-insensitive; **unknown => `desktop`**; `null`/absent => none | `src/model/discovery.rs:66-99` |

### 3.2 `RegisterDtoV2` (register body; `info` of prepare-upload) — `src/http/dto_v2.rs:15-50`

| Key | Type | Required | Notes |
|---|---|---|---|
| `alias` | string | yes | display name |
| `version` | string | yes | `"2.2"` today; never validated by the server |
| `deviceModel` | string | no (skip if None) | e.g. `"Samsung"`, `"Windows"` |
| `deviceType` | enum | no (skip if None) | see 3.1 |
| `fingerprint` | string | **yes** | HTTPS: uppercase hex SHA-256 of own cert DER. HTTP: any id (the app still sends its cert hash) |
| `port` | u16 | **yes** | missing => 400. Browser upload page sends `0` |
| `protocol` | enum | **yes** | how to reach the sender |
| `download` | bool | no (default `false`), always serialized | download API (web share) active |

### 3.3 `RegisterResponseDtoV2` (register response) — `src/http/dto_v2.rs:57-84`

`alias`, `version`, `deviceModel?`, `deviceType?`, `fingerprint` (string, `#[serde(default)]` => may be missing when
parsing), `download` (bool, default false). No `port`/`protocol` (the caller already knows them). The server always
answers `version: "2.2"` (`PROTOCOL_VERSION_V2`, `src/model/discovery.rs:4`; `src/http/server/v2.rs:196-206`).

### 3.4 `InfoResponseDtoV2` (GET info; `info` of prepare-download) — `src/http/dto_v2.rs:141-166`

Same keys as 3.3, but `fingerprint` is required when parsing.

### 3.5 `FileDto` — `src/model/transfer.rs:134-155`

| Key | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | sender-chosen (UUIDv4 in the app). **The server keys everything by the map key, not by this field** (`src/http/server/v2.rs:312-338`) — keep them equal |
| `fileName` | string | yes | may contain `/` for folder transfers (relative path) |
| `size` | u64 | yes | exact byte count; enforced on upload |
| `fileType` | string | yes | v2: a MIME type (`image/png`, `text/plain`, fallback `application/octet-stream`). v1 used bare enum names (`image`,`video`,`pdf`,`text`,`apk`,`other`); the Dart decoder still accepts those (`packages/localsend_isolates/lib/model/dto/file_dto.dart:80-89`) |
| `sha256` | string | no (skip if None) | lowercase hex SHA-256 of content; compared case-insensitively. Key is **`sha256`** on the wire (the Dart model calls it `hash` internally — do not copy that key) |
| `preview` | string | no (skip if None) | text-message body (section 8) |
| `metadata` | object | no (skip if None) | `{ "modified"?: RFC3339, "accessed"?: RFC3339 }`, each skipped if None (`:149-155`) |

Timestamps: RFC 3339 with offset required (`2000-01-01T00:00:00` without zone is rejected and ignored), fractional
seconds allowed, nanoseconds emitted (`src/model/transfer.rs:191-211`). Unparsable timestamps are ignored, not an error.

### 3.6 Prepare-upload request/response — `src/http/dto_v2.rs:91-111`

```json
// request
{ "info": RegisterDtoV2, "files": { "<fileId>": FileDto, ... } }
// response 200
{ "sessionId": "<uuid v4>", "files": { "<fileId>": "<token uuid v4>", ... } }   // only accepted files
```

### 3.7 Prepare-download response — `src/http/dto_v2.rs:124-134`

```json
{ "info": InfoResponseDtoV2 /* download always true */, "sessionId": "<string>", "files": { "<fileId>": FileDto } }
```

### 3.8 Error body — `src/http/dto.rs:23-26`, `src/http/server/common/error.rs:21-47`

Every handler error is `Content-Type: application/json` with `{"message": "<text>"}`. For bare status errors the text is
`"Status code: <code> <reason>"` (e.g. `"Status code: 500 Internal Server Error"`). Clients parse `message` if it is JSON,
otherwise use the raw body (`src/http/client/mod.rs:333-350`).

Successful JSON responses: `Content-Type: application/json` (`src/http/server/common/response.rs:28-40`). Empty-body
successes (upload, cancel, show) send 200 with no content type.

## 4. Endpoint semantics and status codes

### 4.1 `POST /register` — `src/http/server/v2.rs:154-207`

1. Parse body (400 on failure).
2. If TLS and a client cert is present: valid iff `payload.fingerprint.to_ascii_uppercase() == sha256_upper_hex(client_cert_der)`.
   Without TLS (or no cert in web mode) it is always "valid".
3. Valid => emit `Register` event via `try_send` (dropped if the app's channel is full). Invalid => logged, no event.
4. **Always 200** with own `RegisterResponseDtoV2` (also on fingerprint mismatch).

### 4.2 `GET /info` — `src/http/server/v2.rs:209-224`

200 `InfoResponseDtoV2`. Old clients appended `?fingerprint=` to detect themselves; it is ignored.

### 4.3 `POST /prepare-upload` — `src/http/server/v2.rs:226-360`

Order of checks (matters for which error a client sees):

| Step | Failure | Status | `message` |
|---|---|---|---|
| 1. PIN (before body is read) | blocked IP (>= 3 wrong) | 429 | `Too many requests` |
| | no `pin` param | 401 | `PIN required` |
| | wrong `pin` | 401 | `Invalid PIN` (counts an attempt) |
| 2. Body | invalid JSON | 400 | `Invalid JSON body` |
| | `files` empty | 400 | `No files provided` |
| 3. Single slot | another session pending or active | 409 | `Blocked by another session` |
| 4. App decision (no timeout) | sender sent session-less `/cancel` meanwhile | 403 | `Cancelled by sender` |
| | declined | 403 | `Rejected` |
| | app dropped decision channel | 500 | `Status code: 500 Internal Server Error` |
| 5. Accepted subset empty | – | **204**, empty body | – |
| 6. Success | – | 200 | `PrepareUploadResponseDtoV2` |

`sessionId` and every file `token` are fresh UUIDv4 strings (`:251`, `:319`). If the sender drops the connection while
the app is deciding, the slot is freed and `PrepareUploadAborted` is emitted (`PendingSessionGuard`, `:549-600`).
No check that `info.fingerprint` matches the client cert here; the event carries `cert_fingerprint` separately and the
app uses `certFingerprint ?? info.fingerprint` as identity.

### 4.4 `POST /upload?sessionId&fileId&token` — `src/http/server/v2.rs:362-453`

Body: raw bytes of the file. No content type needed. The Rust client streams it with **chunked transfer encoding**
(no `Content-Length`, `src/http/client/mod.rs:181-193`); the browser page sends a `Blob` with `Content-Length`. A server
must accept both.

| Condition | Status | `message` |
|---|---|---|
| any of the three params missing | 400 | `Missing parameters` |
| no active session / sessionId differs / **sender IP differs from prepare-upload IP** / unknown fileId / wrong token / file not `Pending` (already uploading, finished, failed) | 403 | `Invalid token or IP address` |
| body ended with fewer or more bytes than `size`, write error, stream error, app reported error | 500 | `Status code: 500 Internal Server Error` |
| `sha256` given, verification on, digest mismatch | 422 | `Checksum mismatch` |
| ok | 200 | empty |

Per-file state machine (`src/http/server/common/session.rs:22-77`): `Pending -> InProgress -> Finished | Failed`.
A 422 resets the file to `Pending` while `attempts < 3` (`MAX_UPLOAD_ATTEMPTS`, `v2.rs:643-671`) so the sender may retry
**with the same token**. When every accepted file is `Finished` or `Failed`, the session slot is freed and
`SessionEnd{Finished}` is emitted (`v2.rs:673-691`). Failed (non-422) files cannot be retried. A connection dropped
mid-body marks the file `Failed` (`UploadGuard`, `:606-641`). Server-side writes: 16-chunk channel, 512 KiB write buffer,
exact-size check, truncate, then apply `metadata` timestamps (`src/http/server/common/save.rs:11-15`, `:284-355`).

### 4.5 `POST /cancel?sessionId` — `src/http/server/v2.rs:455-528`

Always **200 empty** (when v2 is enabled). Effects, in order:
- Pending slot and request comes from the pending sender's IP and (`sessionId` absent or equal): the waiting
  prepare-upload returns 403 `Cancelled by sender`. This is how a 2.x sender cancels before it knows the session id.
- Active session with equal id and same IP: slot freed, `SessionEnd{Cancelled}`. In-flight uploads keep running.
- Otherwise with a `sessionId`: `CancelReceived{ip, sessionId}` — the *receiver* is cancelling a transfer that this device
  is *sending*; the app must check that `ip` is the send target. The receiving LocalSend calls `/cancel` on the sender's
  advertised port with the session id the receiver issued.

### 4.6 `POST /prepare-download?sessionId&pin` (download API / "share via link") — `src/http/server/web.rs:308-393`

Only active in web-download mode; otherwise **403 `Web download not initialized.`** (not 404, `:470-475`).
- `sessionId` of an accepted session from the same IP => 200 file list immediately (page reload).
- PIN check (separate attempt counter from upload) => 401/429 as in 4.3.
- `sessionId` := the client's IP rendered as string (`fe80::1%3` style). One session per IP; a new request replaces it.
- App decision: decline => 403 `File transfer rejected.`; channel dropped => 500. Accept => 200 `PrepareDownloadResponseDtoV2`.
No request body. `info.download` is always `true`.

### 4.7 `GET /download?sessionId&fileId` — `src/http/server/web.rs:395-468`

400 `Missing sessionId.` -> 403 `Invalid sessionId.` (not accepted or different IP) -> 400 `Missing fileId.` -> 403
`Invalid fileId.` -> 200 streamed body with headers `Content-Type: application/octet-stream`,
`Content-Disposition: attachment; filename="<name with / replaced by -, percent-encoded except A-Za-z0-9-_.!~*'()>"`,
`Content-Length: <size>`.

### 4.8 `POST /show?token` — `src/http/server/internal.rs:50-84`

404 if not configured, 403 `Invalid token`, 400 on bad JSON, else 200. Body `{"args":["..."]}` optional. Purely
local single-instance IPC; not needed for interop.

### 4.9 `GET /`

Download mode: download page; upload mode ("receive via link"): upload page; disabled: **403 HTML page**
(`src/http/server/web.rs:283-298`). Browser pages are `packages/core/assets/web/*.html`.

## 5. PIN

`src/http/server/common/pin.rs:9-48`
- Only `prepare-upload` and `prepare-download` check a PIN; `register`, `info`, `upload` never do.
- Transport: query parameter `pin` (URL-encoded). Comparison is exact string equality.
- Per-IP failure counter in an LRU of 200 IPs. `count >= 3` => 429 **even with the correct PIN**. A missing PIN does not
  count; a correct PIN resets the counter. There is **no time-based reset**: only LRU eviction or a server restart (the
  app restarts the server when the PIN changes) unblocks the IP.
- Sender behaviour (`app/lib/provider/network/send_provider.dart`, prepare loop): 401 => prompt and retry, 403 =>
  declined, 409 => recipient busy, 429 => too many attempts, other => error.

## 6. TLS identity

### 6.1 Certificate — `src/crypto/cert.rs:21-56`

- RSA-2048 key, PKCS#8 PEM; self-signed X.509, `CN=LocalSend User`, no SANs; validity 1975-01-01..4096-01-01 (rcgen default);
  serial derived from the public key hash. Older Dart-generated certs (e.g. the test vector in `cert.rs:137-155`) carry
  empty O/OU/L/ST/C and a 10-year validity — peers must accept any self-signed RSA cert.
- **Fingerprint = uppercase hex SHA-256 of the certificate DER**, 64 chars (`cert.rs:101-106`). Test vector:
  `4BADDE53A7F7CDEEED93189FD898E02BF6B4806CA4C05DE0ACE08319B86552FA` (`cert.rs:276-284`).
- Persist the identity: changing it changes the fingerprint and breaks favorites/pinning on peers.

### 6.2 Server side — `src/http/server/mod.rs:384-395`, `:556-572`, `src/http/server/common/client_cert_verifier.rs`

- rustls (ring provider), single cert. **Client certificates are requested always and mandatory unless a web share
  (download or upload page) is active**, where they become optional so browsers can connect.
- A presented client cert is checked only for: parseable X.509, time validity, and **self-signature** with its own key
  (`cert.rs:75-97`, `verify_signature(None)`). No CA, no name checks. CA-issued certs fail.
- Missing mandatory client cert => TLS handshake failure (no HTTP response; `tests/v2_tls_pinning.rs:351`).
- The peer identity used by the server is the fingerprint of the client cert (`RequestClientInfo::cert_fingerprint`,
  `mod.rs:584-589`).

### 6.3 Client side — `src/http/client/mod.rs:202-251`, `src/http/client/server_cert_verifier.rs:91-116`

- Always presents its own cert as client cert (same identity as the server).
- Server-cert verification: ignore host name; check validity + self-signature; if an expected fingerprint is configured
  (case-insensitive) it must match **during the handshake**, so nothing is sent to an impostor. Discovery uses no pin
  (trust on first use) and reads the fingerprint from the handshake (`cert_fingerprint_from_res`, `mod.rs:298-307`).
- In HTTPS mode the cert fingerprint overrides any fingerprint claimed in a response body (`src/discovery/mod.rs:169-176`).

### 6.4 Encryption off (`protocol: "http"`)

No TLS at all; `cert_fingerprint` is `None`, every register is trusted, identity = body `fingerprint`
(`src/http/server/mod.rs:519-525`, `v2.rs:163-166`). Spec intent is a random string; the app reuses its cert hash.
Mixed networks: a device scans and answers with **its own** protocol choice for the scan (`scan_subnet(.., protocol)`),
but answers multicast announcements with the announcer's `protocol` (`src/discovery/mod.rs:540`).

## 7. Session rules and concurrency

- **Exactly one upload session per receiver** (single `Mutex<Option<SessionStateV2>>`, `src/http/server/mod.rs:64-65`).
  A pending (undecided) request also occupies the slot => concurrent senders get 409.
- Upload requests are bound to the **IP** that made prepare-upload (`PeerIp` incl. IPv6 scope). A sender with several
  source addresses (IPv4/IPv6, multi-homed) must use the same one.
- No session timeout anywhere: a sender that vanishes after a 200 leaves the session `Active` until the receiving app
  cancels it (`ServerHandle::cancel_v2_session`, `mod.rs:201-213`). Re-implementations should add an idle timeout.
- Different files of one session may upload in parallel; the same file only once at a time. The LocalSend sender uploads
  **2 files in parallel** and retries a file up to **3 times on 422** (`packages/localsend_isolates/lib/src/isolate/child/upload_isolate.dart:16-23`, `:163-200`).
- Connection limits: 64 total, 8 per peer IP (loopback exempt from per-IP), excess connections are closed without a
  response (`src/http/server/connection_limit.rs:9-14`, `src/http/server/mod.rs:450-469`).
- Web download sessions: many concurrently, keyed by client IP (`web.rs:342-353`).

## 8. Text messages

Sender (`app/lib/provider/selection/selected_sending_files_provider.dart:45-58`, `app/lib/provider/network/send_provider.dart`):
- One file: `fileName = "<uuidv4>.txt"`, `size = UTF-8 byte length`, `fileType = "text/plain"` (MIME looked up from the
  `.txt` extension), `preview = full text`. `sha256` may also be present (checksums are on by default).
- Receiver detects a message iff `files.length == 1 && fileType is text (mime starts with "text/" or legacy "text") && preview != null`
  (`app/lib/model/state/server/receive_session_state.dart:59-63`). It shows the text and answers by **accepting the empty
  set => 204**; never auto-saves it as a file (`receive_controller.dart`, `acceptFileRequest({})`).
- Sender treats 204 (or an empty `files` map) as "finished, nothing to upload".
- Compatibility: if a receiver accepts the file instead (200 with a token), the sender uploads the bytes, so a
  HyperShare receiver may choose either; a HyperShare sender must handle both.

## 9. Version negotiation and history

| Protocol | LocalSend app | Notes (from `packages/localsend_isolates/lib/constants.dart:1-10`) |
|---|---|---|
| 1.0 | 1.0.0 - 1.8.0 | v1 paths (`send-request`, `send`), legacy enum `fileType` |
| 1.0 + 2.0 | 1.9.0 - 1.14.0 | v2 with v1 fallback |
| 1.0 + 2.1 | 1.15.0 - 1.17.0 | 2.1 (exact field delta vs 2.0 not visible in this repo; `download` flag is optional) |
| 2.2 | 1.18.0+ (Rust core) | mandatory mTLS, checksums (`sha256`, 422 retry), metadata timestamps, IPv6 multicast extension, session-less cancel of pending requests |

- There is no explicit negotiation: peers exchange `version` strings and clients pick v1 paths only for `"1.0"`.
- The 2.2 server answers `/api/localsend/v1/info` for <= 1.17 probing (`mod.rs:646-653`), but serves no other v1 route.
- 2.2-specific fields are all optional, so 2.0/2.1 peers interoperate: they simply never send `sha256`/`metadata`, and never
  get 422.
- The browser upload page identifies itself as `version: "2.1"`, `deviceType: "web"`, `fingerprint: "web-<random hex>"`,
  `port: 0`, `alias: "Web Browser"` (`assets/web/upload.html:224-235`).

## 10. Timeouts and buffers

| What | Value | Source |
|---|---|---|
| Discovery register request | 500 ms total | `src/discovery/mod.rs:23-26` |
| Subnet scan | `/24` per interface, 50 concurrent probes | `src/discovery/mod.rs:31-33`, `:342-385` |
| prepare-upload, upload, download | **no timeout** (decision waits for the user) | `src/http/client/v2.rs` (no timeout unless configured) |
| Server request timeouts | none; TCP keepalive 60 s idle / 10 s interval | `src/http/server/mod.rs:338-343` |
| Client read chunk | 512 KiB, 4-chunk read-ahead | `src/model/transfer.rs:9-13` |
| Server write buffer | 512 KiB, 16-chunk channel | `src/http/server/common/save.rs:11-15` |

## 11. Discovery touch points (HTTP side)

- Multicast group `224.0.0.167:53317` (IPv6 extension `ff12::fd3a:e420`), TTL 1, JSON = `MulticastMessageV2`
  (`src/model/discovery.rs:107-141`: RegisterDtoV2 fields) plus `"announce": true`; burst at 100 ms, 500 ms, 2000 ms
  (`src/multicast/mod.rs:28-49`, `:174-195`). Receivers ignore the `announce` flag value and answer **every** message
  via `POST /register` to the announcer (UDP is announce-only; 2.2 never sends UDP replies — older apps did, with
  `announce:false`, and those are treated as announcements).
- Own messages are dropped by fingerprint equality (`src/multicast/mod.rs:375-378`).
- Answering an HTTPS announcement pins the announced fingerprint (`src/discovery/mod.rs:518-530`).

## 12. Quirks and pitfalls (checklist)

1. Map key vs `FileDto.id`: server uses the **map key**; make them identical.
2. Wire key is `sha256`, lowercase hex; compare case-insensitively.
3. `port` and `protocol` are required in `info` of prepare-upload; omitting them => 400.
4. `deviceType` unknown => `desktop`; `protocol` must be lowercase.
5. PIN is checked before the body and before the busy check: a wrong PIN yields 401 even when busy.
6. 429 lockout has no expiry.
7. Upload IP binding — keep one source address per session.
8. `/cancel` always returns 200; it is also the receiver->sender "stop" signal (`CancelReceived`).
9. 204 on prepare-upload means "accepted, nothing to send" (text messages).
10. Register always 200; a fingerprint/cert mismatch is silent.
11. mTLS: present a **self-signed** client cert on every HTTPS request; servers without web share drop cert-less handshakes.
12. HTTP/1.1 only; do not negotiate h2. Max 8 connections per peer.
13. Uploads may be chunked; enforce exact `size` yourself.
14. `/prepare-download` without web share returns 403 (not 404); its `sessionId` is the client IP.
15. No session idle timeout in the reference: add one, but keep it long (prepare-upload waits for a human).
16. `AGENTS.md` lists `ServerConfigV2 { pin, event_tx, web_send }`; the code has `{ pin, verify_checksums, event_tx }`
    and web share is configured via `WebConfig` (`src/http/server/mod.rs:40-51`, `web.rs:87-150`).

## 13. Extension points for a re-implementation

- Unknown JSON fields are ignored by LocalSend, so extra capability fields (e.g. a HyperShare feature flag in
  `RegisterDtoV2`/`MulticastMessageV2`) are safe to add; never change the meaning of existing ones.
- Additional routes under a separate prefix are safe: LocalSend returns empty-body 404 for unknown paths.
- Parallel uploads beyond 2 per session are allowed by the server (bounded by 8 connections per IP).
- Plain-HTTP mode is a first-class protocol value and can be used on trusted links for speed, but the peer must also be
  configured for HTTP (a 2.2 HTTPS server does not accept plain HTTP on the same port).
