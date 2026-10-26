# LocalSend vs HyperShare

This page compares the two projects side by side so the port has a shared starting point. The LocalSend facts come
from `/home/user/localsend` (README, `packages/core/src`, synced with upstream `cfd6991`) and the LocalSend
inventories in `/home/user/localsend/docs/hypershare/`. The HyperShare facts come from this repository
(`docs/architecture.md`, `crates/`, `app/`). For details, see `docs/port/hypershare-app.md` in the HyperShare repository and
`docs/port/hypershare-engine.md`.

## Summary

| | LocalSend | HyperShare (today) |
|---|---|---|
| Purpose | Share files and messages with any device that runs LocalSend, over a shared local network | Share files between Linux and Xiaomi/Redmi/POCO (and other MTA) phones using the phone's built-in Mi Share, with **no app on the phone** |
| Peer requirement | Both sides run LocalSend (or a browser for web share) | The phone needs only its stock share sheet |
| Network requirement | Same LAN or subnet, without AP isolation (README §Setup) | None: the devices find each other over BLE and bring up their own Wi-Fi link |
| Licence | Apache-2.0 (core crate) | MIT |

## Features

| Feature | LocalSend | HyperShare |
|---|---|---|
| Send/receive files | Yes, many files and folders (folder structure is carried in `fileName`) | Yes. Receive: the phone sends a ZIP stream. Send: legacy "Share to PC" 1.0 |
| Text messages | Yes: a single `<uuid>.txt` `text/plain` file with `preview`; the receiver may answer 204 | Receive only, partly: `Transfer.text` is filled from CatShare requests (`crates/engine/src/receive.rs:109`), but the UI doesn't show it |
| Accept prompt / auto-accept | Yes. Quick-save `off`/`on`/`paired` (favourites) | Yes, with an answer timeout and "auto-accept known phones" |
| PIN protection | Receive PIN (`?pin=`, 401, at most 3 tries per IP then 429; `http/server/common/pin.rs`) | No |
| Web share (browser download) | Yes (`http/server/web.rs`, assets in `packages/core/assets/web/`) | No |
| Favourites, history, send by address | Yes | Known-sender list only, plus Activity history (at most 20 transfers, 1 h, `crates/engine/src/store.rs:11-12`) |
| Internet transfer | WebRTC with a signalling server (`packages/core/src/webrtc/`, `server/`) | No |
| Thumbnails / previews | Optional `preview` field | Yes: the phone serves thumbnails, which show in the request sheet and in notifications |
| Desktop integration | Tray, autostart, `--hidden`, single instance through the `show` endpoint, CLI crate | Tray with per-transfer menu, freedesktop notifications with Accept/Decline, D-Bus API, GNOME Shell extension, Nautilus script, AppImage menu entry with quick actions |
| i18n | Yes, through Slang and Weblate | No (English literals) |

## Discovery

| | LocalSend | HyperShare |
|---|---|---|
| Primary | UDP multicast `224.0.0.167:53317` (`multicast/mod.rs:28,39`), announce-only, sent as a burst (`ANNOUNCE_DELAYS`). Replies go back as an HTTP `POST /api/localsend/v2/register` | BLE: a 31-byte connectable MTA advert plus a GATT service while visible. Phones are found by BLE scan (`PHONE_TTL` 20 s, `crates/engine/src/discovery.rs:14`) |
| Secondary | HTTP subnet scan of the `/24` of each interface, 50 hosts in parallel (`discovery/mod.rs:33,336`) | Scan bursts. The engine keeps scanning for 30 s after stop (`SCAN_GRACE`) |
| IPv6 | LocalSend extension: `ff12::fd3a:e420`, one socket per interface (`multicast/mod.rs:35`) | n/a |
| Identity | Alias plus fingerprint (SHA-256 of the cert DER, uppercase hex). Own messages are dropped by fingerprint | Phone name from the advert (can be cut to 15 bytes, with "…"), plus brand |
| Visibility control | The server runs or doesn't. There is no timed mode | Off / Everyone for 10 min / Everyone |

## Transports

| | LocalSend | HyperShare |
|---|---|---|
| Link | The existing LAN (router) | **Receive:** Wi-Fi Direct group created by the phone. Its credentials are written over GATT, encrypted with ECDH + AES-CTR. **Send:** a WPA2 hotspot on `192.168.137.1` (NetworkManager, firewalld `trusted` zone) that the phone joins (`docs/architecture.md` §Flows) |
| Session protocol | REST over HTTPS (HTTP/1.1, hyper): `register`, `prepare-upload`, `upload`, `cancel`, `info`, `prepare-download`/`download` (`http/server/mod.rs:628-692`) | Receive: a WSS handshake to the phone, then `GET /download` streaming a ZIP. Send: the phone pulls from our HTTPS API |
| Direction model | The sender pushes (`upload` per file); web share is a pull | Receive is a pull from the phone; send is a pull by the phone |
| Concurrency | One upload session at a time on the receiver (409 when busy). The Dart sender uploads 2 files in parallel. Connection caps: 64 in total, 8 per IP | One flow per transfer. The radio is a shared resource (`WifiLease`) |
| Fallback | HTTP scan when multicast fails. WebRTC over the internet | None. If BLE or Wi-Fi is missing, there's no transfer |

## Security

| | LocalSend | HyperShare |
|---|---|---|
| Transport encryption | TLS with a per-device self-signed RSA-2048 certificate (`crypto/cert.rs`). Encryption can be turned off (plain HTTP) | Receive: TLS on the WebSocket and download over a WPA2 Wi-Fi Direct link. Send: WPA2 hotspot plus HTTPS with a self-signed cert in `~/.config/hypershare/tls` |
| Peer authentication | **Mandatory client certificate** unless web share is on (`http/server/mod.rs:386`). The peer id is the client-cert fingerprint, and `Register` is dropped when the claimed fingerprint doesn't match | Key agreement (ECDH) during the GATT exchange. Phones are "known" by remembered identity for auto-accept |
| Access control | Accept prompt, PIN, favourites | Accept prompt, timeout, known senders, visibility modes |
| Integrity | Optional SHA-256 per file (`sha256` key). The receiver answers 422 on a mismatch and the sender retries up to 3 times | TLS only |

## Speed (qualitative; neither repo publishes benchmarks)

| Factor | LocalSend | HyperShare |
|---|---|---|
| Path | Through the router: client → AP → client, so roughly half the air time on a single AP | Direct peer-to-peer Wi-Fi (Wi-Fi Direct or our hotspot): one hop, so the band and channel width depend on the adapter |
| Parallelism | Sender with 2 concurrent file uploads. The server accepts parallel uploads of different `fileId`s | One stream |
| Overheads | Optional SHA-256 pre-pass on the sender (on by default), TLS, chunked bodies | Hotspot or Wi-Fi Direct setup time (`JOIN_TIMEOUT` 45 s cap), ZIP framing, TLS |
| Port target | Parallel streams, large buffers, optional plain HTTP on trusted links, and a path choice based on link speed (shared decision 4) | |

## Platform reach

| | LocalSend | HyperShare |
|---|---|---|
| OS | Android 7+, iOS 13+, macOS 11+, Windows 10+, Linux, Fire OS (README §Compatibility) | Linux only (BlueZ, NetworkManager, freedesktop D-Bus). Other OSes would need new `ports::Bluetooth`/`Wifi` adapters |
| Peers reachable | Any LocalSend app, plus browsers through web share | Xiaomi/Redmi/POCO (send and receive), other MTA brands (receive), Samsung/Hisense (some models). See `app/src/features/settings/brands.ts` |
| Packaging | Stores, Flathub, Snap, AUR, DEB, AppImage, … | AppImage |

## Architecture

| | LocalSend | HyperShare |
|---|---|---|
| UI | Flutter (`app/`) with Refena state | Tauri 2 + Svelte 5 (`app/`), miuix design system |
| Core | Rust crate `localsend` (`packages/core`), with features `crypto`/`http`/`multicast`/`discovery`/`webrtc`/`full` | Hexagonal: `hypershare-core` (domain, ts-rs types), `-protocol` (wire formats), `-engine` (use cases + ports), `-linux` (adapters), `-dbus` |
| Bridge | flutter_rust_bridge plus Dart isolates. Networking runs in child isolates | Tauri commands plus a `hs://state` snapshot event. The engine runs inside the app process |
| Server → app events | `ServerEventV2` channel with oneshot responders (`decision_tx`) | Engine `Store` → `watch<Snapshot>`, plus `Event::IncomingRequest` |
| External API | HTTP (protocol), `show` endpoint, CLI | D-Bus `org.hypershare.Daemon1` |
| Testing | `cargo test --features full`, Flutter tests | Golden protocol fixtures, engine fakes over loopback TLS, Vitest, and Playwright with screenshots |

## Port target feature matrix

Decided in the shared decisions: Linux only; LocalSend protocol v2 **reimplemented** inside `hypershare-protocol`
and `hypershare-engine`, wire-compatible, with LocalSend core used only as a test peer.
Legend: ✅ exists, ◐ partial, 🎯 port target, ➖ out of scope.

| Capability | MTA phones (existing) | LocalSend apps over LAN | LocalSend phone, no shared LAN | HyperShare ↔ HyperShare |
|---|---|---|---|---|
| Discovery | ✅ BLE advert/scan | 🎯 UDP multicast `224.0.0.167:53317` (IPv4, plus IPv6 group as an option) and HTTP `/24` scan, `register` answer | 🎯 the phone joins our hotspot through a QR, then multicast/scan on the hotspot subnet | 🎯 BLE auto-discovery (HyperShare-specific advert) |
| Link setup | ✅ Wi-Fi Direct (receive) / WPA2 hotspot (send) | existing LAN | 🎯 5 GHz hotspot plus a Wi-Fi QR `WIFI:T:WPA;S:..;P:..;;` | 🎯 automatic 5 GHz hotspot or Wi-Fi Direct join, with no manual steps |
| Receive files / images / video | ✅ | 🎯 v2 `prepare-upload` → `upload` | 🎯 | 🎯 |
| Send files / images / video | ✅ (Xiaomi family) | 🎯 v2 client | 🎯 | 🎯 |
| Text messages | ◐ receive data only, no UI | 🎯 send and receive (`<uuid>.txt` + `preview`), composer and viewer UI | 🎯 | 🎯 |
| TLS + client certs + fingerprint identity | n/a | 🎯 required for interop | 🎯 | 🎯 |
| Plain-HTTP fast mode on trusted links | ➖ | 🎯 optional (LocalSend `https=false` peers) | 🎯 optional | 🎯 |
| Parallel streams, large buffers | ➖ | 🎯 | 🎯 | 🎯 |
| Path choice based on link speed | ➖ | 🎯 picks LAN, hotspot or Wi-Fi Direct per peer | 🎯 | 🎯 |
| Receive PIN | ➖ | 🎯 (as receiver; sender PIN prompt) | 🎯 | 🎯 |
| Checksum (`sha256`) | ➖ | 🎯 accept and verify (422) | 🎯 | 🎯 |
| Per-device transport badge (MTA/LAN/BLE) | 🎯 | 🎯 | 🎯 | 🎯 |
| Web share (browser) | ➖ | ➖ | ➖ | ➖ |
| WebRTC / internet transfer | ➖ | ➖ | ➖ | ➖ |
| Protocol v1, Windows/macOS/mobile builds | ➖ | ➖ | ➖ | ➖ |
