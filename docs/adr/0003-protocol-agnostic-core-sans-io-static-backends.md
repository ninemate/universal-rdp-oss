# 0003. Protocol-agnostic sans-IO core with statically linked protocol backends

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [architecture.md](../architecture.md), [language-and-architecture.md §3](../research/language-and-architecture.md#3-workspace-layout-and-core-abstractions), [0002](0002-implementation-language-rust.md), [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)

## Context

The project's stated vision is one protocol-agnostic core (sessions, input, display, clipboard, audio, file transfer) with pluggable protocol backends ([README](../../README.md)). The scaffold left three things open: how backends plug in (static or dynamic), the threading/async model, and how mid-connection security upgrades (RDP's TLS and CredSSP, RFB's VeNCrypt) cross the transport boundary.

The research recommends ([language-and-architecture.md §3.1](../research/language-and-architecture.md#31-design-principles-sans-io-vs-async-integrated)) following IronRDP's design: protocol logic as **sans-IO** state machines that consume and produce bytes, with time injected by the caller, and a thin async driver that owns sockets and timers. IronRDP already works this way (`Sequence::step`, `should_perform_security_upgrade()`), as do quinn and str0m. This makes deterministic tests, fuzzing and a WASM build straightforward.

## Decision

1. **Protocol state machines are sans-IO where the underlying library allows it.** `urdp-core` (types), `urdp-session` (the `SessionMachine` contract) and the VNC/RFB protocol crates do no I/O, read no clocks and spawn no threads. They are `no_std + alloc` where dependencies allow it, `#![forbid(unsafe_code)]`, and compile for `wasm32-unknown-unknown`. RDP adapters preserve that boundary where IronRDP's APIs permit it; if the server API requires async I/O, isolate that adapter in the extra tier with the runtime.
2. **One driver.** `urdp-runtime` pumps session machines over transports on **tokio**. The browser build uses `wasm-bindgen-futures` instead. Async code exists only in the runtime, transport, platform and frontend crates.
3. **Security upgrades are explicit needs.** A session machine reports a `Need` (`TlsUpgrade`, `TlsServerKey`, `Credentials`, `Auth`) and the driver resolves it, so the transport layer supports in-place TLS upgrade without the protocol crate touching sockets. This answers the transport question left open in the scaffold.
4. **Backends are statically linked Cargo crates behind feature flags**, registered through a `ProtocolBackend` trait (id, role, default port, capabilities, `new_session`). There is no dynamic plugin loading.
5. **Capabilities are per backend.** The core exposes a `Capabilities` set and an opaque per-backend `Channel` escape hatch; frontends degrade when a backend lacks a capability.
6. **Server-side platform access is behind traits** (`FrameSource`, `InputSink`, `ClipboardProvider`, `AudioIo`) in `urdp-platform`, implemented per OS ([0010](0010-linux-server-capture-and-input.md)).
7. **Privilege separation is designed in from M1.** The server can run capture, input and codecs in a helper process separate from the network-facing process; the first implementation may run them in-process, but the trait boundaries must not assume shared memory with the network process.

The trait sketches in [language-and-architecture.md §3.4](../research/language-and-architecture.md#34-core-trait-sketch) are the starting point; final names and signatures are settled in M1 and documented in [architecture.md](../architecture.md).

## Consequences

- Whole handshakes can run in memory in tests (client machine ↔ server machine), with scripted faults and an injected clock.
- Every parser can be fuzzed by calling `handle_input` directly.
- Adding a protocol means adding crates and a feature flag, not changing the core, but also means rebuilding: third parties cannot ship binary plugins.
- More boilerplate than linear async code: explicit states and timers as data.
- The IronRDP adapters (`urdp-rdp`, `urdp-rdp-server`) have to map IronRDP's APIs to ours. The client adapter may fit the sans-IO core; where `ironrdp-server` requires async I/O, the server adapter lives in the extra tier rather than the core tier. Confirm the boundary in the M1/M4 API review; see [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md).
- Revisit if a real need for out-of-tree binary plugins appears, or if mapping IronRDP's server crates onto `SessionMachine` turns out to cost more than using them directly behind a narrower server trait.

## Alternatives considered

- **Async-integrated protocol code** (protocol functions `await` sockets). Reads linearly, but ties protocol crates to a runtime, makes fuzzing and WASM harder, and hides ordering bugs across channels.
- **Dynamic plugins** (`dlopen`, a C ABI, or WASM components). Rejected for now: Rust has no stable ABI, the security review surface grows, and there is no demand yet. A WASM-component plugin model could be reconsidered later.
- **No shared core** (separate RDP and VNC products). Rejected: contradicts the project's goal and duplicates transport, auth, platform and frontend code.
