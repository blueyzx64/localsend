# LocalSend discovery and transfer performance (reference for HyperShare)

Scope: what a wire-compatible reimplementation must do to be **discovered by** and to **discover** real
LocalSend peers, plus an evidence-based list of why LocalSend transfers are slow and which speed-ups are
safe without breaking compatibility. All paths are relative to the LocalSend repo root
(upstream main `cfd6991`). Line numbers refer to that revision.

Core crate: `packages/core` (crate `localsend`). CLI: `cli/`. Flutter app: `app/` + `packages/localsend_isolates/`.
Both the app and the CLI use the **same Rust core** for multicast, discovery, HTTP client and HTTP server,
so the Rust code is the behavior of current real clients.

---

## 1. Constants

| Thing | Value | Source |
|---|---|---|
| IPv4 multicast group | `224.0.0.167` | `packages/core/src/multicast/mod.rs:28`, `packages/localsend_isolates/lib/constants.dart:24` (user-configurable in app settings) |
| IPv6 multicast group (LocalSend extension) | `ff12::fd3a:e420` | `multicast/mod.rs:35-36` |
| UDP multicast port = default HTTP(S) port | `53317` | `multicast/mod.rs:39`, `constants.dart:14` |
| Protocol version string | `"2.2"` | `packages/core/src/model/discovery.rs:4` |
| Announcement burst delays | 100 ms, 500 ms, 2000 ms (cumulative t = 0.1 s, 0.6 s, 2.6 s) | `multicast/mod.rs:45-49` |
| UDP receive buffer | 65536 B | `multicast/mod.rs:53` |
| Receive error tolerance | 10 consecutive errors, then socket abandoned | `multicast/mod.rs:58`, `:356` |
| Multicast TTL / hops | 1 | `multicast/socket.rs:118`, `:150` |
| Register (discovery) timeout | 500 ms total request timeout | `discovery/mod.rs:26`, app default `constants.dart:19` (`defaultDiscoveryTimeout = 500`) |
| Subnet scan concurrency | 50 in flight | `discovery/mod.rs:33` |
| Staged-discovery grace before subnet scan | 1 s (app and CLI) | `app/lib/provider/network/scan_facade.dart:31`, `cli/src/app/mod.rs:202` |
| Max interfaces scanned by app | 3 | `scan_facade.dart:15` |
| Server connection cap | 64 total, **8 per peer IP** (loopback exempt from per-IP) | `http/server/connection_limit.rs:9,14,46` |
| TCP keepalive (server) | 60 s idle, 10 s interval | `http/server/mod.rs:341-343` |
| Max upload attempts per file (only for 422) | 3 | `http/server/v2.rs:646` (`MAX_UPLOAD_ATTEMPTS`), Dart `upload_isolate.dart:23` |

API base path: `{http|https}://{host}:{port}/api/localsend/v2/{endpoint}` (`http/client/url.rs:22-23`).
IPv6 hosts are bracketed; link-local hosts carry the scope as `fe80::1%3` (`discovery/store.rs:153-166`).
Server routes: `http/server/mod.rs:631-676` (`register`, `info` (v1 and v2), `prepare-upload`, `upload`,
`cancel`, `prepare-download`, `download`, internal `show`). v1 endpoints other than `info` are not served.

---

## 2. Multicast announcement

### 2.1 Message JSON

Struct `MulticastMessageV2` (`model/discovery.rs:105-141`), `camelCase`, plus a flattened `announce`
flag on send (`multicast/mod.rs:174-180`, `:186-189`):

```json
{
  "alias": "Nice Orange",
  "version": "2.2",
  "deviceModel": "Samsung",        // optional, omitted when None
  "deviceType": "desktop",         // optional; mobile|desktop|web|headless|server; omitted when None
  "fingerprint": "<uppercase hex SHA-256 of cert DER>",   // HTTP mode: random string
  "port": 53317,                   // the HTTP server port (may differ from the UDP port)
  "protocol": "https",             // "http" | "https" (lowercase)
  "download": false,               // download API (web share) active; defaults false when absent
  "announce": true
}
```

Parsing rules (receiver side):

- Required: `alias`, `version`, `fingerprint`, `port`, `protocol`. `deviceModel`/`deviceType`/`download`
  optional (`model/discovery.rs:115-140`, test `:193-207`).
- Unknown `deviceType` values map to `desktop` (`model/discovery.rs:86-98`); matching is case-insensitive.
- Unknown fields are ignored (no `deny_unknown_fields` anywhere in core) — so extra fields are safe to send.
- The receiver **does not read `announce` at all**: every parsed datagram is treated as an announcement and
  answered (`multicast/mod.rs:367-387`, `discovery/mod.rs:477-489`).
- Legacy flag: protocol v1 used `announcement`; this repo neither sends nor parses it, and the clone is shallow
  so history could not be checked. Sending `"announcement": true` in addition to `"announce": true` is
  harmless for current peers (unknown field) and is the safe choice for very old apps — **unverified here**.

### 2.2 Burst timing

`MulticastHandle::announce()` sleeps 100 ms → send, 500 ms → send, 2000 ms → send (`multicast/mod.rs:224-234`).
Each send goes to **every** bound socket (one per interface address, see 2.4) (`:198-203`). Nothing is
sent periodically: the app announces once at discovery start (`packages/localsend_isolates/lib/src/task/discovery/discovery.dart:100`)
and again on each user-triggered scan (`StartSmartScan` → `discover_staged`, which joins `announce()`,
`discovery/mod.rs:332`). The CLI announces only at start and on refresh (`cli/src/app/mod.rs:184-215`).

### 2.3 Answering: HTTP register, not UDP

UDP is announce-only (`multicast/mod.rs:1-6`). On receiving an announcement from another fingerprint the
peer POSTs `/api/localsend/v2/register` to `{source_ip}:{message.port}` with its own `RegisterDtoV2`
(`discovery/mod.rs:506-565`):

- Request body `RegisterDtoV2` (`http/dto_v2.rs:15-50`): same fields as the multicast message minus `announce`.
- For `https`, the client **pins** the announced fingerprint in the TLS handshake (`discovery/mod.rs:519-528`)
  and presents its own client certificate (mandatory, see 3.2).
- Answers are spawned concurrently, one task per datagram (`discovery/mod.rs:486-488`), no dedup / rate limit
  — a 3-datagram burst x N interfaces produces several register POSTs to the announcer.
- Only after a 200 response is the device put into the store (`:543-555`); the store key is the fingerprint
  from the announcement (pinned), not from the body.
- Answering is disabled while the local HTTP server is not running (`discovery/mod.rs:381-389`,
  `discovery.dart:51-57`, `:93-95`).

Response body `RegisterResponseDtoV2` (`http/dto_v2.rs:55-84`): `alias, version, deviceModel?, deviceType?,
fingerprint (default ""), download (default false)` — note **no `port`/`protocol`** in the response.
Server side (`http/server/v2.rs:154-207`): on TLS, the register is only *emitted to the app* when the body
fingerprint (uppercased) equals the client-cert SHA-256; the HTTP 200 response is returned either way.
The event is `try_send` (dropped under load, by design: peers re-announce).

**So the announcer learns peers through its HTTP server's `Register` events**, and the answerer learns
the announcer through the register *response*. A HyperShare port must therefore run the HTTP(S) server
before announcing, and must feed server `Register` events into its device list (the app does this in
`app/lib/provider/network/server/controller/receive_controller.dart:55-65`, the CLI in
`cli/src/app/discovery.rs:101-127`).

### 2.4 Own-message filtering and sockets

- Loopback is **on** (several instances per host must see each other); own datagrams are dropped by
  `message.fingerprint == own fingerprint` (`multicast/mod.rs:375-378`, `socket.rs:113-115`). Register answers
  whose responder has our fingerprint are dropped too (`discovery/mod.rs:177-179`), and the app ignores
  `/register` from itself (`receive_controller.dart:56-59`).
- Interfaces: `if_addrs`, loopback skipped, optional whitelist/blacklist glob on any address of the interface
  (`*` = 1+ non-dot chars) (`util/interface.rs:5-66`, `:101-174`).
- **IPv4: one socket per interface address** (`socket.rs:36-56`): `SO_REUSEADDR` + `SO_REUSEPORT` (unix),
  bind `0.0.0.0:53317` (wildcard, required for multicast receive), `IP_ADD_MEMBERSHIP(group, iface_addr)`,
  `IP_MULTICAST_IF(iface_addr)` (otherwise all sockets send on the default-route interface),
  `IP_MULTICAST_LOOP=1`, TTL 1 (`socket.rs:88-121`).
- **IPv6** (only when `group_v6` set; the app and CLI always set it: `packages/localsend_isolates/rust/src/api/discovery.rs:199`,
  `cli/src/app/mod.rs:154`): one socket per interface that has any IPv6 address, `IPV6_V6ONLY`, bind
  `[::]:53317`, join by **interface index**, `IPV6_MULTICAST_IF(index)`, loop on, hops 1 (`socket.rs:57-82`, `:125-153`).
  Target `[ff12::fd3a:e420%index]:53317`.
- Failing interfaces are skipped; zero sockets → multicast error, but discovery continues via HTTP
  (`multicast/mod.rs:263-268`, `discovery/mod.rs:436-441`). All receive loops failing → `SocketsFailed`
  event; app restarts discovery after 1 s (`discovery.dart:110-120`).
- Linux note for HyperShare: with `SO_REUSEPORT` on N wildcard-bound sockets in the same group, Linux
  delivers each multicast datagram to **every** socket, so one datagram yields up to N `Discovered` events
  (all deduped by fingerprint in the store, but each triggers a register POST).

Received `Discovered` carries `scope_id` for IPv6 link-local sources; host becomes `fe80::x%scope`
(`multicast/mod.rs:380-386`, `discovery/mod.rs:512-515`).

---

## 3. HTTP discovery fallback (subnet scan)

### 3.1 Staged discovery (what the app/CLI actually run)

`DiscoveryHandle::discover_staged` (`discovery/mod.rs:308-334`), invoked by app `StartSmartScan`
(`scan_facade.dart:12-35`) when the send tab opens with no devices (`app/lib/pages/tabs/send_tab_vm.dart:229-234`)
or on manual refresh (`send_tab.dart:340-341`, which first clears the list):

1. In parallel: announcement burst (2.6 s) **and**
2. register-probe all favorites / known channels (concurrency 50, `:272-294`), then sleep `grace` (1 s);
3. if the global confirmation counter did not change during 2., scan the `/24` of each interface IPv4
   (up to 3 in the app) concurrently.

`StartLegacySubnetScan` (`scan_facade.dart:38-57`) forces scans on user-selected subnets + an announcement.

### 3.2 Scan algorithm

`scan_subnet` (`discovery/mod.rs:342-379`):

- Hosts: `a.b.c.0` … `a.b.c.255` (all 256, incl. .0 and .255) minus own IP — always `/24` regardless of
  real netmask.
- Each host: `POST /api/localsend/v2/register` with own `RegisterDtoV2`, on the configured port/protocol,
  using an **unpinned** client (any valid cert) with client cert (`:145-152`) and a **500 ms total timeout**.
- `buffer_unordered(50)` → worst case ≈ ceil(255/50) × 0.5 s ≈ 3 s per interface.
- HTTPS identity = SHA-256 of the server cert from the handshake, not the body (`:170-176`).
- One scan per interface at a time (`ScanGuard`, `:348-354`, `:421-434`).
- Discovery events are `try_send` (never block a scan) (`:193-207`).

Note: the HTTP client always presents the client cert; the LocalSend server **requires** a client
certificate on TLS unless web pages are being served (AGENTS.md "TLS"; `http/server/mod.rs:388-391`
`mandatory_client_auth`). HyperShare's client must therefore send a self-signed client cert, and its
server should request one (to get the peer fingerprint) but tolerate none for browsers.

### 3.3 Device store / expiry

- Store = in-memory `Vec<StatefulDevice>` keyed by fingerprint; channels merged by host
  (`discovery/store.rs:196-245`). Max 100 log entries per device (`store.rs:13`).
- **There is no time-based expiry** in core or app. Devices disappear only when the user refreshes
  (`ClearFoundDevicesAction`, `app/lib/provider/network/nearby_devices_provider.dart:63-75`, dispatched at
  `send_tab.dart:340,352`) or discovery restarts. Channels go `NotReachable` only if the application marks
  them (`store.rs:47-49`).
- Best channel ranking: available first, then **IPv6 preferred**, then most recently confirmed
  (`store.rs:97-107`).

---

## 4. Transfer path (upload, protocol v2)

1. `POST /prepare-upload[?pin=]` body `{info: RegisterDtoV2, files: {id: FileDto}}` (`dto_v2.rs:89-97`);
   `FileDto` = `id, fileName, size, fileType, sha256?, preview?, metadata{modified?,accessed?}?`
   (`model/transfer.rs:135-155`). Responses: 200 `{sessionId, files:{id: token}}`, 204 (nothing to transfer,
   e.g. text message), 401 pin, 403 rejected, 409 busy, 429 (`client/v2.rs:141-148`).
2. `POST /upload?sessionId=&fileId=&token=` raw body per file (`client/v2.rs:233-273`). 200 ok, 403 invalid
   token/IP, 409, 422 checksum mismatch, 500.
3. `POST /cancel?sessionId=` (`client/v2.rs:284-304`).

Receiver rules (`http/server/v2.rs`):

- **Single session slot**: a second `prepare-upload` while a session is pending/active → 409
  (`v2.rs:254-268`).
- Upload request is validated under the session lock, then the lock is released before streaming
  (`v2.rs:381-399`): checks sessionId, **sender IP equals the prepare-upload IP**, fileId, token, and
  `status == Pending`; then marks `InProgress`.
- **Parallel uploads of different fileIds in one session are permitted**: nothing serializes the body
  streams; each request has its own `FileUpload` event, target and writer task. The same fileId twice
  concurrently → second gets 403 (status no longer `Pending`).
- A file that fails for any reason other than 422 becomes `Failed` permanently (`v2.rs:663-669`); the
  sender **cannot retry** it within the session. 422 resets to `Pending` up to 3 attempts.
- Session ends (`SessionEnd::Finished`) when every accepted file is `Finished` or `Failed` (`v2.rs:672-690`,
  `common/session.rs:42-46`). There is **no idle timeout** on an active session (grep finds none); an
  abandoned sender keeps the receiver busy (409 for others) until the app closes the session.
- Body must be exactly `FileDto.size` bytes or the write fails (`common/save.rs:296-319`). Chunked transfer
  encoding is fine (hyper reads frames; no Content-Length requirement in the handler).
- If `verify_checksums` (app default **true**, `app/lib/provider/persistence_provider.dart:408`; CLI true,
  `cli/src/app/mod.rs:137`) and the sender provided `sha256`, the receiver hashes inline and replies 422 on
  mismatch (`v2.rs:405-408`, `common/save.rs:164-234`).
- File timestamps from `metadata` are applied after write (`common/save.rs:333-351`).

Real senders:

| Sender | Parallelism | Checksums sent | Source |
|---|---|---|---|
| Flutter app | **2 files in parallel** per session (`Pool(2)`), one pinned client (connection reuse) | yes by default (`ls_create_checksums` default true): **full SHA-256 pre-pass over every file, sequentially, before `prepare-upload`** | `packages/localsend_isolates/lib/src/isolate/child/upload_isolate.dart:17,141,146`; `app/lib/provider/network/send_provider.dart:97,164-213`; `persistence_provider.dart:400` |
| CLI | **sequential**, sorted by file name | no (`sha256: None`) | `cli/src/send_task.rs:201-285`; `cli/src/app/sending.rs:111,329` |
| `examples/stress_send.rs` | sequential, 1000 x 1 MiB over plain HTTP | no | `packages/core/examples/stress_send.rs:91-121` |

The core client doc explicitly says `upload` "can be called in parallel for multiple files"
(`client/v2.rs:215`), and the server's per-IP limit comment says "a sender uploads two files in parallel"
(`connection_limit.rs:11-13`).

---

## 5. Why LocalSend is slow

Evidence-based bottlenecks, most significant first.

| # | Bottleneck | Evidence | Effect |
|---|---|---|---|
| 1 | **Sender SHA-256 pre-pass** before the request, one file after another | `send_provider.dart:164-213`, default on (`persistence_provider.dart:400`) | Every byte is read twice; for large sets the receiver sees nothing for the whole hashing time. With software SHA-256 (~0.5–2 GB/s per core) this alone can exceed the network time on fast links. |
| 2 | **Low upload parallelism**: 2 (app) or 1 (CLI) | `upload_isolate.dart:17`, `send_task.rs:201` | Many small files are dominated by per-request latency (request → `FileUpload` event → app decides target → file create → 200), not bandwidth. One TCP stream also rarely saturates Wi-Fi 5/6 with TLS. |
| 3 | **Per-file round-trips through the app** on the receiver: each upload emits `FileUpload` and waits for the application to answer `target_tx` (path decision, SAF fd on Android) | `v2.rs:416-431` | Adds an IPC hop (Rust → FRB → Dart isolate → back) per file; small files are latency-bound. |
| 4 | **TLS everywhere**, RSA-2048 certificates, rustls/ring, HTTP/1.1 only | `crypto/cert.rs:7-36`; ALPN forced `http/1.1` (`client/mod.rs:229-230`); server `http1::Builder` (`server/mod.rs:534-553`) | Handshake cost per new connection (pool mitigates); symmetric crypto on every byte; HTTP/2 deliberately disabled (flow-control window). |
| 5 | **Small effective frames on receive**: body frames are ~TLS-record sized (≤16 KiB), passed through an `mpsc` of capacity 16 to a writer task with a 512 KiB `BufWriter` | `common/save.rs:10-15`, `:258`, `:293` | One channel send + await per 16 KiB; hashing (if enabled) runs inline in the request task (`save.rs:177-179`). Throughput OK on desktop, but per-chunk overhead is non-trivial on phones. |
| 6 | **Sender read-ahead 4 x 512 KiB** per file | `model/transfer.rs:9-13` | Fine for one stream; caps prefetch to 2 MiB per stream. |
| 7 | **Single session** at a time, no idle timeout | `v2.rs:254-268` | A stalled/aborted sender blocks other senders (409). Not a throughput issue per se, but a UX/latency one. |
| 8 | **Discovery latency**: no periodic announcements, 1 s grace + up to ~3 s per /24 scan, no expiry | §2.2, §3 | Devices appear late on multicast-hostile networks; stale devices linger. |
| 9 | Receiver `flush` only at end of file, then `set_len` + `set_times` | `save.rs:310-351` | Good (no per-chunk fsync). Not a bottleneck — keep this pattern. |

What is **not** a bottleneck: the receiver writes with a 512 KiB `BufWriter` and flushes once per file; there
is no fsync; TCP_NODELAY is set (`server/mod.rs:412`).

---

## 6. Faster, still wire-compatible: what HyperShare may change

Safe (verified against the receiving server's rules above):

1. **Upload several fileIds of one session in parallel** — allowed by the server (§4). Keep **≤ 7 concurrent
   connections per receiver IP**: the server silently closes connection #9+ at accept (`server/mod.rs:450-468`,
   per-IP limit 8), which a client sees as a connection reset, and that file's request never reached the
   handler (file stays `Pending`, so it can be retried). Leave one slot for `cancel`. Recommended default 4–6;
   use 1 connection per file request (HTTP/1.1, no pipelining).
2. **Skip the sender SHA-256 pre-pass** (`sha256` is optional; CLI never sends it). Alternatively hash while
   uploading — impossible to put in `prepare-upload` then, so on fast trusted links simply omit it. As a
   HyperShare receiver, verify only if the sender supplied it.
3. **Plain HTTP** on trusted links: protocol `"http"` is part of v2; the peer must also run in HTTP mode
   (the LocalSend "encryption" setting). A LocalSend HTTPS peer will *not* accept plain HTTP; HyperShare must
   follow the peer's announced `protocol`. HyperShare↔HyperShare can negotiate HTTP freely.
4. **Large I/O buffers**: sender chunks of 1–4 MiB, receiver writer buffers ≥ 1 MiB, socket buffers
   (`SO_SNDBUF/SO_RCVBUF`) raised; none of this is visible on the wire.
5. **Order small files first / batch** within the parallel pool to amortize per-request latency.
6. **Receiver side** (HyperShare as receiver): accept parallel uploads, decide save targets synchronously in
   Rust (no UI round-trip per file), add an idle timeout for abandoned sessions (LocalSend senders tolerate a
   409/403 later), and allow up to ≥ 8 connections per peer.
7. **Discovery**: announce periodically (e.g. every 30–60 s plus on network change) and expire devices after
   N missed announcements — both invisible to peers. Dedup answers per (fingerprint, host) for a few seconds to
   avoid N interfaces × 3 datagrams register storms. Use the real netmask instead of a fixed /24 when scanning
   (but /24 covers what LocalSend itself scans).

Not safe / breaks compatibility:

- Changing the multicast message to require new fields, changing `fingerprint` semantics (must be uppercase
  hex SHA-256 of the cert DER in HTTPS mode), or answering announcements over UDP only.
- Using HTTP/2 to a LocalSend receiver (server is HTTP/1.1-only; client must offer ALPN `http/1.1`).
- Splitting one file into several ranged uploads: the server has no range/offset API; each fileId accepts
  exactly one body of exactly `size` bytes. Multi-stream-per-file is only possible HyperShare↔HyperShare
  (e.g. a v2-compatible extension endpoint gated on a capability announced in an extra JSON field — unknown
  fields are ignored by LocalSend).
- Retrying a non-422 failed file in the same session (server marks it `Failed`, `v2.rs:663-669`); a resend
  needs a new `prepare-upload` session.
- Sending from a different source IP than the `prepare-upload` (e.g. a second interface): server checks
  `sender_ip` per upload (`v2.rs:387`). Bind all upload connections to the same local address.

---

## 7. Porting checklist

- [ ] UDP: bind per IPv4 interface address, join 224.0.0.167:53317, `IP_MULTICAST_IF`, loop on, TTL 1;
      optional IPv6 `ff12::fd3a:e420` per interface index.
- [ ] Send burst at +100/+600/+2600 ms with `announce: true` (and optionally `announcement: true`).
- [ ] On datagram: drop own fingerprint; POST `/api/localsend/v2/register` to `src_ip:message.port` with
      pinned fingerprint, 500 ms timeout; on 200 add device.
- [ ] HTTP(S) server on 53317 before announcing; `/register` returns `{alias, version:"2.2", deviceModel,
      deviceType, fingerprint, download}`; feed incoming registers into the device list (verify cert
      fingerprint on TLS).
- [ ] Fallback: if nothing confirmed 1 s after favorites probe, scan /24 per interface, 50 concurrent, 500 ms.
- [ ] Sender: parallel uploads (≤ 7/peer), optional sha256, follow peer protocol, single local source IP.
- [ ] Receiver: single session semantics compatible (409), parallel fileIds, exact-size check, 422 + 3 attempts
      if verifying checksums, timestamps from metadata.
