# LocalSend app: user-facing feature inventory (for the HyperShare port)

Source: the Flutter app `app/lib` and the Dart isolate layer `packages/localsend_isolates/lib`, upstream main `cfd6991`.
Wire formats live in `localsend-protocol.md`, and discovery and throughput in `localsend-discovery-and-performance.md`.
This file covers **app behaviour**: what the user sees, which settings drive it, and which protocol calls it makes.
Paths are relative to the repository root. `L` means line numbers.

Priority legend: **MUST** means it is required for a credible, wire-compatible LocalSend peer on Linux. **NICE** means it is optional, a follow-up, or specific to another platform.

---

## 0. Architecture in one paragraph

The main isolate holds the UI state: Refena providers in `app/lib/provider`. Three child isolates do the networking: discovery, HTTP upload and HTTP server (`packages/localsend_isolates/lib/src/isolate/child/*`). The actual HTTP server and client are Rust (`packages/core`). The app never routes HTTP requests itself. It **reacts to server events** (`HttpServerEvent`, `server_isolate.dart` L166-347) and **answers decision oneshots** such as accept/decline, save targets and download sources. The state that children need is pushed through `SyncState` (`child/sync_provider.dart` L13-25): alias, port, protocol (http/https), whitelist, blacklist, multicast group, discovery timeout, serverRunning, download and the security context. HyperShare should mirror this split: a protocol engine that emits events, plus app-level policy (accept, quick save, destination, history).

---

## 1. Settings: complete list, defaults and persistence keys

`app/lib/model/state/settings_state.dart` L10-77 and `app/lib/provider/persistence_provider.dart` (keys L48-98, defaults L343-593).

| Setting | Key | Default | Effect | Needs server restart? | Port |
|---|---|---|---|---|---|
| alias | `ls_alias` | random "Adjective Fruit" (`util/alias_generator.dart`, localized) | name in `/info`, `/register`, multicast and prepare-upload `info` | yes (restart banner, `settings_tab.dart` L311-318) | MUST |
| port | `ls_port` | `53317` (`packages/localsend_isolates/lib/constants.dart` L14) | HTTP server port **and** multicast UDP port | yes | MUST |
| https (encryption) | `ls_https` | `true` (L496-498) | TLS with a self-signed cert and mandatory client cert. Off: plain HTTP | yes | MUST |
| multicastGroup | `ls_multicast_group` | `224.0.0.167` (constants L24) | multicast discovery group | discovery restart (synced through `IsolateSyncSettingsAction`) | MUST |
| discoveryTimeout | `ls_timeout` | `500` ms (constants L19) | HTTP subnet-scan per-host timeout | synced | MUST |
| networkWhitelist / networkBlacklist | `ls_network_whitelist` / `_blacklist` | `null` (disabled) | interface IP filter, `*` wildcard (see §14) | synced | MUST |
| destination | `ls_destination` | `null` means `~/Downloads` (Linux: `getDownloadsDirectory()`, fallback `$HOME/Downloads`, then `$HOME`; symlinks resolved; `util/native/directories.dart` L7-37) | receive folder | no | MUST |
| saveToGallery | `ls_save_to_gallery` | `true` | Android/iOS only | no | NICE (n/a on Linux) |
| saveToHistory | `ls_save_to_history` | `true` | record received files and messages | no | MUST |
| quickSave | `ls_quick_save` = `QuickSaveMode{off,on,paired}` | **`paired`** (L459-462) | auto-accept all requests (`on`) or only from favorites (`paired`) | no | MUST |
| receivePin | `ls_receive_pin` | `null` | PIN required on prepare-upload | yes (`settings_tab.dart` L212-214) | MUST |
| autoFinish | `ls_auto_finish` | `false` | progress page closes itself 3 s after all files are done (`progress_page.dart` L62, L99-112) | no | NICE |
| createChecksums | `ls_create_checksums` | `true` | sender computes the SHA-256 for each file before prepare-upload | no | MUST (for compatibility; the receiver may verify) |
| verifyChecksums | `ls_verify_checksums` | `true` | receiver verifies SHA-256 and answers 422 on mismatch | yes | MUST |
| sendMode | `ls_send_mode` | `single` (`model/send_mode.dart`: single / multiple / link) | see §8 | no | MUST (single) / NICE (multiple) |
| shareViaLinkAutoAccept | `ls_share_via_link_auto_accept` | `false` | web download: auto-accept browsers | no | NICE |
| receiveViaLinkAutoAccept | `ls_receive_via_link_auto_accept` | `false` | web upload page: auto-accept uploads | no | NICE |
| deviceType | `ls_device_type` | `null` means detected (desktop on Linux) | advertised `deviceType` | yes (synced) | MUST |
| deviceModel | `ls_device_model` | `null` means detected (`"Linux"`, `util/native/device_info_helper.dart`) | advertised `deviceModel` | yes | MUST |
| showToken | `ls_show_token` | UUID v4 generated on first start | single-instance `show` endpoint | at start | MUST (desktop) |
| securityContext | `ls_security_context` | generated on first start (`util/security_helper.dart` L5-13) | key, cert, `certificateHash` = **fingerprint** | reset means a new identity | MUST |
| favorites / history | `ls_favorites` / `ls_receive_history` | `[]` | JSON string lists | n/a | MUST |
| theme, color, locale, animations, tray, autostart, window placement, always-on-top | various | — | UI only | no | NICE |
| signaling / STUN servers | `ls_signaling_servers` / `ls_stun_servers` | — | WebRTC (internet) transfer | — | NICE (out of scope) |

Settings changes that the children need are pushed by `settingsProvider.onChanged` (`provider/settings_provider.dart` L22-41) and `serverProvider.onChanged` (`server_provider.dart` L35-62). **Ordering rule:** the server start publishes the sync state *before* it dispatches the start task (`server_provider.dart` L137-139), because children read it at start.

---

## 2. Identity: alias, fingerprint, device type and model

- **Fingerprint** is always `securityContext.certificateHash`: SHA-256 of the cert DER, upper-case hex. This holds in HTTP (encryption-off) mode too (`provider/device_info_provider.dart`, `deviceFullInfoProvider`). It is sent as `fingerprint`/`token` in register and prepare-upload (`send_provider.dart` L235-249: `token: originDevice.fingerprint`).
- Self-detection: register events that carry our own fingerprint are dropped (`receive_controller.dart` L56-59).
- The trusted sender identity is `event.certFingerprint ?? event.info.fingerprint` (`receive_controller.dart` L89). The payload fallback applies only when https is off.
- Device type enum: `mobile, desktop, web, headless, server`. Unknown values decode as `desktop` (`packages/localsend_isolates/lib/model/device.dart` L1-10).
- `deviceInfoProvider` = the settings override, or else the raw detected values. Changes are synced to the isolates (`device_info_provider.dart`).
- The UI shows the own IP "hashtag" `#<last octet>` per local IPv4 (`pages/tabs/receive_tab.dart` L97, `util/ip_helper.dart` L4).
- Port: **MUST**. Generate the identity once and persist it. Expose alias, type and model in settings.

---

## 3. Sending

### 3.1 What can be selected (`app/lib/provider/selection/selected_sending_files_provider.dart`, `app/lib/util/native/file_picker.dart`)

Desktop picker options are `file, folder, text, clipboard` (`file_picker.dart` L90-98).

| Source | How it works internally | Priority |
|---|---|---|
| Files | `openFiles()` then `AddFilesAction` with `CrossFileConverters.convertXFile`. Reads `modified`/`accessed` metadata (Rust `readFileMetadata`). Exact duplicates are skipped (`isSameFile`) | MUST |
| Folder | `AddDirectoryAction` (L150-204): recursive listing. Each file's `fileName` becomes **`<folderName>/<relative/path>`** with `/` separators, so the folder structure is carried in the name. `.sendignore` support is stubbed (disabled, `util/send_ignore.dart`) | MUST |
| Drag & drop | `pages/home_page.dart` L74-104: dropped dirs go to `AddDirectoryAction`, files to `AddFilesAction`, then the app switches to the Send tab | MUST (desktop) |
| Text message | `MessageInputDialog`, then `AddMessageAction` (L35-65): an in-memory file named `<uuid>.txt`, `FileType.text`, bytes = UTF-8 | MUST |
| Clipboard / paste (Ctrl+V or Cmd+V, `widget/watcher/shortcut_watcher.dart` L35-48) | `_pickClipboard` (`file_picker.dart` L292-357), in this order: 1) **text**, which becomes a message; 2) **image** bytes (`Pasteboard.image`, BMP converted to PNG), named `clipboard_YYYY-MM-DD_HH-MM.<ext>`, `AddBinaryAction`; 3) **file list** (`Pasteboard.files()`) added as files | MUST (text) / NICE (image, files) |
| Media picker / gallery, APK picker | mobile only | NICE (n/a) |
| CLI args / "open with" / context menu | `LoadSelectionFromArgsAction` (L288-355): paths become files or dirs, `--text <s>` or `-t` becomes a message, `--share <json>` becomes share-intent payload | MUST (desktop: also used by the single-instance handover, §13) |

Images and videos have no special path on send: they are files whose MIME type comes from the extension. `FileDto.encode` sends `fileType` as **MIME** (`lookupMime`, `packages/localsend_isolates/lib/model/dto/file_dto.dart` L105-115). On decode, both MIME and the legacy enum name are accepted (L80-90).

### 3.2 Send session flow (`app/lib/provider/network/send_provider.dart` `startSession`, L88-454)

1. A new local `sessionId` and a fresh UUID per file. Status `waiting`. The `SendPage` is pushed unless `background`.
2. **Message embedding:** if the selection is exactly **one text file with in-memory bytes**, `preview` = the full UTF-8 text (L121-123). This turns the request into a *message request* on the receiver.
3. File timestamps go into `metadata.{modified,accessed}` (raw RFC 3339 strings, kept at nanosecond precision, `file_dto.dart` L8-22).
4. If `createChecksums` is set, SHA-256 is computed per file **sequentially** in Rust with cancel support and progress (L164-216). A hash failure is ignored and the file is sent without `hash`.
5. `POST /api/localsend/v2/prepare-upload` uses a client **pinned to `target.fingerprint`** (TLS-level, `provider/http_provider.dart` L24-37; discovery uses an unpinned client). The body is `{info: RegisterDto{alias, version, deviceModel, deviceType, token=fingerprint, port, protocol, hasWebInterface}, files}`. The PIN goes in the `?pin=` query.
6. Response handling (L272-347):

| Status | App outcome |
|---|---|
| 200 | `{sessionId, files: {fileId: token}}`. Files without a token are `skipped` |
| 204 | the receiver accepted nothing ("read and close", used for messages), so the session is `finished` and closed (L354-399) |
| 401 | `PinDialog` (obscured, "invalid PIN" from the 2nd try), then retry with `?pin=`. Dialog cancelled means `canceledBySender` |
| 403 | `declined` |
| 409 | `recipientBusy` (the receiver already has a session) |
| 429 | `tooManyAttempts` (3 wrong PINs per IP, `packages/core/src/http/server/common/pin.rs` L9) |
| other / transport | `finishedWithErrors` + `errorMessage` |

7. Upload: one isolate task per batch (`IsolateHttpUploadFilesAction`). **2 files in parallel** (`upload_isolate.dart` L17 `_concurrency = 2`), and all share one pinned connection (L141). Each file goes `POST /v2/upload?sessionId&fileId&token` with the raw body, the content length = size, a path, an fd or a byte stream (`src/task/upload/http_upload.dart`). **HTTP 422 = checksum mismatch, so the file is retried up to 3 times** (L23, L195-201).
8. Per-file events: `Started`, `Progress (0..1)`, `Finished`, `Failed`. The final status is `finished` or `finishedWithErrors`. **Retry a failed file** with `sendFile(isRetry: true)` (L544-587), which reuses the same token.
9. In `single` mode, a successful session close clears the selection (L787-790).

### 3.3 Send by address, favorites and recent devices
- `AddressInputDialog` (`widget/dialogs/address_input_dialog.dart`) has two modes. **Hashtag** takes the last octet only and tries `<own /24 prefix>.<n>` for every local IP prefix. **IP** takes a full address. It probes with `POST /v2/register` (unpinned discovery client) using **our own https setting and port** (L64-86). The response becomes a `Device`, which is added to `lastDevicesProvider` (the last 5, kept in memory only).
- Favorites dialog: `register` to `favorite.ip:favorite.port` with our https setting, then send (`widget/dialogs/favorite_dialog.dart` L29-58).
- Edge: a peer that runs a different http/https mode or port than ours cannot be reached by manual address. HyperShare should try both schemes.

### 3.4 Cancel (sender side)
`cancelSession` (`send_provider.dart` L710-743) cancels the hash token, the prepare-upload token (aborting the request tells the receiver "aborted while deciding") and the upload tasks. If a remote session exists it then calls `POST /v2/cancel?sessionId=<remote>`, pinned. `cancelSessionByReceiver` runs when *our* server gets `/cancel` from the receiver's IP while we are `sending` (`receive_controller.dart` L477-497, which checks that the IP matches the target).

---

## 4. Receiving

### 4.1 Flow (`app/lib/provider/network/server/controller/receive_controller.dart`)
1. `PrepareUpload` event (L70-235). The Rust server has already checked the PIN and holds the **single-session slot** (409 to others). Any stale displayed session is closed first.
2. Session state: `destinationDirectory = settings.destination ?? default`. `senderAlias` = the favorite's alias if the sender fingerprint is a favorite, else the payload alias (L100). `saveToGallery` is never true on Linux.
3. **Message request** = `files.length == 1 && fileType == text` → `message = preview` (`model/state/server/receive_session_state.dart` L58-63).
4. **Auto-accept** (L129-158): `quickSave` (mode `on`), or `paired` and the sender fingerprint is in favorites, or the web upload page is active and `receiveViaLinkAutoAccept` is set. **Messages are never auto-accepted.** When auto-accepting, the progress page opens and all files are accepted with their original names.
5. Otherwise the window is raised from the tray (L160-162) and `ReceivePage` is shown (sender alias, device, file list, Accept / Decline). A message is written to **history immediately when shown** (L164-182, `isMessage: true`, size = UTF-8 bytes).
6. Accept (L526-605) sends the decision with `fileNameMap {fileId: desiredName}` to the isolate, which answers prepare-upload 200 with tokens for the accepted ids only. **An empty map makes the server answer 204 and create no session**; this is how a message is "accepted" (L532-538, L195-198). Decline answers **403** (`config: null`, L618-626).
7. Uploads: the server isolate resolves the save target and Rust writes the file, emitting progress, then a result event (`server_isolate.dart` L470-503, L619-735). Uploads of the same fileId are queued (checksum retry), different files run concurrently.
8. Each finished file goes to history (L353-368). When every file is finished or failed, the status becomes `finished` or `finishedWithErrors`. A quick-saved, fully successful session closes and returns Home, then `OpenFileDialog` opens for the last file (L412-444). Sessions with errors stay open so the sender can retry.

### 4.2 Receive options (before accepting) (`pages/receive_options_page.dart`, `selected_receiving_files_provider.dart`)
- Per-session **destination override** (`setSessionDestinationDir`, `\` normalized to `/`), and save-to-gallery (mobile).
- Per-file **select/unselect** (skip), **rename**, bulk **random names** (UUID, keep extension) and **counter names** (prefix, zero-pad, optional sort) (`selected_receiving_files_provider.dart` L24-70).
- Priority: MUST (destination, select all/none), NICE (rename, bulk rename).

### 4.3 Saving, subfolders and filename conflicts (`packages/localsend_isolates/lib/src/task/server/file_saver.dart`)
- `sanitizeRelativeName` (L170-194): split on separators. **`..` or an absolute root makes the upload fail with "Path traversal detected"** (it is not rewritten). `.` and empty components are dropped. Each component is sanitized for the platform: on Linux (POSIX rules) `/` and control chars become `_`, and names are capped at 255 bytes (`packages/core/src/util/filename.rs`).
- **Subfolders**: intermediate dirs are created with `createSync(recursive: true)`, plus a second `p.isWithin` traversal gate (L225-240).
- **Conflict handling** (L242-247): if the name exists (checked with `followLinks: false`), it becomes `name (2).ext`, `name (3).ext` and so on. An existing ` (n)` suffix is replaced, not stacked (`util/file_path_helper.dart` `withCount`, L28-45). It never overwrites, **except** a retry of the same fileId within a session, which reuses the same target and overwrites it (`server_isolate.dart` L651-662).
- A partial file is **kept** on failure (L699-700).
- Timestamps: Rust applies `metadata.modified`/`accessed` to the written file (`packages/core/src/http/server/v2.rs` ~L410, `common/save.rs` L81-90).
- Checksum: when `verifyChecksums` is set and the sender gave `hash`, a mismatch gives **HTTP 422** and the file is marked failed. The sender retries.

### 4.4 Text message UI (`pages/receive_page.dart`)
- Shows the message text. **Copy** writes it to the clipboard and accepts with the empty map (L229-240). If the text is a single absolute URI with no whitespace (`isLink`, L51), an **Open** button launches the URL (L242-255). The close (X) button also accepts with the empty map (L292-300), so the sender sees "finished". Edge: popping the page with system back calls `onDecline` (L91-95), which answers 403, so the sender sees "declined".
- Priority: MUST (show and copy). NICE (open link).

### 4.5 Cancel (receiver side)
`cancelSession` (L653-681) tells Rust to cancel the session (further uploads fail), then sends `POST /v2/cancel?sessionId=` to the sender, pinned to its fingerprint. Sender-side cancel arrives as `SessionEnd(cancelled)` or `PrepareUploadAborted` and becomes `canceledBySender` (L450-473, L700-722).

---

## 5. PIN
- Receive PIN (settings) is enforced by Rust on `prepare-upload` via `?pin=`. 401 means a PIN is required or wrong. **3 wrong attempts per IP mean 429** (LRU of 200 IPs, `pin.rs`). A correct PIN resets the counter.
- The PIN is fixed at server start, so changing it restarts the server.
- Web share has its own PIN (§9). In **upload-page mode the web PIN replaces the receive PIN for all v2 clients** (`server_provider.dart` L150-153).
- Sender side: 401 opens a PIN dialog and loops (§3.2).
- Priority: **MUST**.

## 6. Favorites (`provider/favorites_provider.dart`, `model/persistence/favorite_device.dart`)
- `{id(uuid v1), fingerprint, ip, port, alias, customAlias}`. Add, edit and remove (remove is by fingerprint).
- Used for: quick-save `paired` (fingerprint match), the alias override on receive, sending by stored IP:port (register probe), and auto-updating the alias on rediscovery unless `customAlias` (`provider/network/nearby_devices_provider.dart` L88-93).
- Edge: the stored IP goes stale (DHCP), so sending fails until rediscovery. Identity is the fingerprint, which is spoofable when https is off.
- Priority: **MUST** (it is the backbone of auto-accept).

## 7. Receive history (`provider/receive_history_provider.dart`, `pages/receive_history_page.dart`)
- Newest first, **max 30 entries** (L5). Recorded only if `saveToHistory` is set. Entry: `{id, fileName (or message text), fileType, path, savedToGallery, isMessage, fileSize, senderAlias, timestamp(UTC)}`.
- UI: open file, open folder (destination), show info, delete entry, clear all, copy message.
- Priority: **MUST** (basic), NICE (rich actions).

## 8. Send to multiple devices (`SendMode.multiple`, `pages/tabs/send_tab_vm.dart` L168-219)
- Each device tap starts a **background** session (`background: true`). Sessions are keyed in the send map. The existing session for the same `device.ip` is reopened (SendPage while waiting, ProgressPage while sending) or replaced.
- Background sessions remove themselves on success (`send_provider.dart` L518-521). Switching back to single mode clears all sessions (L140-143).
- Priority: NICE (single-target sending is MUST).

## 9. Web share: "share via link" (download) and "receive via link" (upload page) (`pages/web_share_page.dart`, `server/controller/send_controller.dart`)
- Entry points: send mode `link` (`send_tab_vm.dart` L122-137) gives download mode. The Receive tab button (`receive_tab.dart` L110) gives upload mode.
- The **server restarts with https = false by default** (`_init(encrypted: false)`, L50-55, toggleable). On leaving the page it restarts from settings (L106-108). While a web page is served, client certificates become optional so browsers can connect.
- Download mode: `WebMode.download{files, pin}`. In-memory files are written to cache files `web-download-<id>` (`send_controller.dart` L45-52). Each browser visit emits a `WebPrepareDownload{ip, sessionId, userAgent}` request, listed with "Browser (OS)" and accept/decline unless auto-accept is on. File fetches resolve to a path or fd.
- Upload mode: the server serves the drop page. Browser uploads arrive as normal `PrepareUpload` events and use the receive UI.
- The page shows one URL per local IP: `http(s)://<ip>:<port>`, copy button, and QR code with `?pin=` appended (L194-245). Toggles: encryption, auto-accept, PIN.
- Changing the PIN restarts the server and drops sessions (`server_provider.dart` L301-321).
- Edge: in web mode LocalSend peers see an HTTP-only server (the sync state advertises http), and the text `preview` is attached based on `files.first` for all files (`send_controller.dart` L63, an upstream quirk).
- Priority: NICE for v1 of the port. Valuable for the "no LocalSend app" case.

## 10. Progress and speed display (`pages/progress_page.dart`, `util/file_speed_helper.dart`)
- Per-file status `queue/skipped/sending/failed/finished` (`packages/localsend_isolates/lib/model/file_status.dart`) and progress 0..1 (`fileTransferProvider`).
- Total = the sum of accepted file sizes. **Speed = bytes · 1000 / elapsed ms since session `startTime`** (an average, not instantaneous), shown only after ≥ 500 KiB (L241-251). The remaining time is recomputed at most once per second. The taskbar progress bar is updated.
- autoFinish counts down 3 s. Cancel asks for confirmation. Open folder after receive. Retry per failed file (sender).
- Session statuses: `waiting, recipientBusy, declined, tooManyAttempts, sending, finished, finishedWithErrors, canceledBySender, canceledByReceiver` (`session_status.dart`).
- Priority: **MUST** (progress, speed). NICE (taskbar, autoFinish).

## 11. Encryption toggle
- `https` setting (default on). On: TLS with a per-device self-signed cert, **mandatory client cert**, and fingerprint pinning for all session traffic. Off: plain HTTP, and the identity is the self-reported fingerprint. The UI shows `EncryptionDisabledNotice` when turned off (`settings_tab.dart` L479-486).
- The app advertises `protocol: http|https` in register and multicast, so peers dial accordingly.
- Priority: **MUST** (both modes for compatibility). This also matches HyperShare's "unencrypted on trusted links" goal.

## 12. Port, multicast and discovery settings
- Port 53317 is used for both TCP HTTP and UDP multicast. The group defaults to `224.0.0.167`. Discovery timeout is 500 ms. The settings tab "restart server" also restarts discovery and refreshes local IPs (`settings_tab_controller.dart` L134-155).
- Start failure (port in use) is surfaced as a snackbar. Alias and port are "repaired" on restart: an empty alias becomes random, an invalid port becomes the default (`server_provider.dart` L126-133).
- Server self-heal: on `ListenerFailed`, or a failed loopback probe, the server restarts with the same config (L374-431).
- Details are in `localsend-discovery-and-performance.md`. Priority: **MUST**.

## 13. Single instance: the `show` endpoint (`packages/localsend_isolates/lib/util/show_instance.dart`, `app/lib/config/init.dart` L100-114)
- On desktop start, before anything else: `POST http(s)://127.0.0.1:<port>/api/localsend/v2/show?token=<showToken>` with the JSON body `{"args": [...]}`, presenting **our own client cert** (badCertificateCallback accepts self). Connection timeout 100 ms, response timeout 500 ms. A 200 response makes the new process `exit(0)`.
- The running instance (`receive_controller.dart` L501-523) shows the window from the tray and feeds the args into `LoadSelectionFromArgsAction`. If files were added it switches to the Send tab.
- Port: **MUST** semantics, but HyperShare may use its own IPC (D-Bus or a unix socket) instead. It must not open a LAN-visible unauthenticated endpoint. LocalSend relies on the random token.

## 14. Network interface filter (`pages/settings/network_interfaces_page.dart`, `packages/localsend_isolates/lib/util/network_interfaces.dart`)
- Whitelist **or** blacklist (mutually exclusive in the UI) of IP patterns. `*` matches one segment without dots: `123.123.124.*` becomes `^123\.123\.124\.[^.]+$` (L49-51).
- Whitelist: an interface is ignored if **none** of its addresses match. Blacklist: an interface is ignored if **any** address matches (L56-67).
- Applied to the shown local IPs (`local_ip_provider.dart` L95-101, **IPv4 only**, ranked so that `.1` addresses go last) and to discovery (passed to Rust `startDiscovery`, `src/task/discovery/discovery.dart` L64-78).
- Priority: **MUST** (Linux machines have docker, VPN and hotspot interfaces; the HyperShare hotspot interface must be selectable).

---

## 15. Porting summary

| Feature | Priority |
|---|---|
| Identity (alias, fingerprint from cert, type, model), persisted | MUST |
| Send files, folders (relative names), text message, clipboard text, drag & drop, CLI args | MUST |
| Clipboard image / file list | NICE |
| prepare-upload status handling 200/204/401/403/409/429/422 | MUST |
| Checksums create + verify, 422 retry | MUST |
| Receive accept dialog, decline, select subset, destination override | MUST |
| Quick save on / favorites-only (default `paired`) | MUST |
| Safe naming: traversal refusal, sanitizing, ` (n)` conflicts, subfolders, timestamps | MUST |
| Text message receive: show and copy, 204 answer, history | MUST |
| PIN (receive + sender prompt, 3 attempts per IP) | MUST |
| Favorites, history (30) | MUST |
| Cancel both directions, single-session busy (409) | MUST |
| Progress, average speed, ETA, retry failed file | MUST |
| http/https toggle | MUST |
| Port, multicast group, timeout, interface filter | MUST |
| Single instance / show | MUST (own IPC acceptable) |
| Rename / bulk rename on receive, open-link, autoFinish | NICE |
| Send to multiple, web share download/upload pages | NICE |
| Gallery, APK, SAF, WebRTC/signaling | NICE / not applicable on Linux |
