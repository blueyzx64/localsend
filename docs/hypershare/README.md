# LocalSend ↔ HyperShare

These pages are for developers porting LocalSend features to [HyperShare](https://github.com/blueyzx64/hypershare)
(a Tauri app for Linux that already shares files with Xiaomi/MTA phones over Bluetooth and Wi-Fi Direct) and for
anyone who needs an exact description of how LocalSend behaves on the wire. They describe this repository; no LocalSend
code was changed for the port.

| Page | What it covers |
|---|---|
| [localsend-protocol.md](localsend-protocol.md) | Protocol v2.2 as implemented in `packages/core`: endpoints, DTOs, status-code order, PIN, TLS and fingerprints, text messages, pitfalls |
| [localsend-discovery-and-performance.md](localsend-discovery-and-performance.md) | Multicast and subnet-scan discovery, the upload path, why transfers are slow and what can change without breaking compatibility |
| [localsend-app-features.md](localsend-app-features.md) | Feature inventory of the Flutter app, marked MUST / NICE for a port |
| [comparison.md](comparison.md) | LocalSend and HyperShare side by side, with the port's target feature matrix |

HyperShare reimplements protocol v2 in Rust and tests it against `packages/core` (its `interop/` crate). All facts
here were read from upstream commit `cfd6991`; re-check them after protocol changes.
