# 0009. Transports (TCP, TLS, WebSocket) and the Web client approach

- **Status:** Proposed (that Web is a first platform is Accepted in [0004](0004-server-first-linux-and-web-first.md); *how* is proposed here)
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [architecture.md](../architecture.md), [language-and-architecture.md §3](../research/language-and-architecture.md#3-workspace-layout-and-core-abstractions), [vnc-protocol.md §4](../research/vnc-protocol.md#4-extensions-beyond-encodings), [prior-art.md: IronRDP, noVNC](../research/prior-art.md#ironrdp-devolutions), [0003](0003-protocol-agnostic-core-sans-io-static-backends.md), [0011](0011-security-baseline.md)

## Context

- Browsers cannot open raw TCP sockets; a browser client needs WebSocket (or WebTransport) to reach a server.
- **RFB over WebSocket** is established practice: the RFB byte stream is carried unchanged in binary WebSocket frames. noVNC requires it; neatvnc, and reportedly QEMU and libvncserver, have built-in WebSocket listeners; others use the websockify proxy ([vnc-protocol.md §4](../research/vnc-protocol.md#4-extensions-beyond-encodings)).
- **noVNC does not support RA2_256 in its current upstream master.** Its security-type list includes VeNCrypt and the Plain subtype, but not RA2_256 ([upstream `rfb.js`](https://github.com/novnc/noVNC/blob/master/core/rfb.js#L54-L68), checked 2026-11-07). Re-check when selecting the test version.
- **RDP in a browser** is harder. CredSSP binds to the TLS server public key, which browser JavaScript cannot see on a `wss://` connection. IronRDP's web client solves this with **RDCleanPath**: the browser talks WebSocket to a proxy, the proxy opens the TCP/TLS connection to the RDP server and hands the server's certificate chain back in-band, so the client can complete CredSSP ([prior-art.md](../research/prior-art.md#ironrdp-devolutions); IronRDP's `ironrdp-rdcleanpath` crate). **Unverified:** the exact RDCleanPath message flow and where TLS terminates; read the IronRDP crate and Devolutions Gateway before implementing.
- IronRDP ships `iron-remote-desktop`, a protocol-agnostic web component (npm `@devolutions/iron-remote-desktop`), which could host both RDP and VNC WASM backends.
- rustls does not support TLS < 1.2. Some legacy RDP hosts only offer TLS 1.0; that concerns our *client* talking to old Windows hosts, not our server ([language-and-architecture.md §1.2](../research/language-and-architecture.md#12-per-language-findings)).

## Decision

### Transports (native)

1. `urdp-transport` provides **TCP**, **TLS** (rustls, TLS 1.2 and 1.3 only) and **WebSocket** (`ws` and `wss`, RFC 6455, binary frames) as byte-stream transports behind one trait, with in-place TLS upgrade driven by the session machine's `Need::TlsUpgrade` ([0003](0003-protocol-agnostic-core-sans-io-static-backends.md)).
2. The server can listen on several endpoints at once: e.g. RFB on TCP 5900, RDP on TCP 3389 (configurable; see the Windows coexistence note in [risks §4.1](../research/risks-and-open-questions.md#41-windows)), and an HTTPS/WSS endpoint for browsers.
3. **The server's WSS endpoint is built in**, so no websockify or separate gateway is needed for the common case. It serves the Web client's static files and upgrades `/vnc` (and later `/rdp`) to WebSocket.
4. UDP (MS-RDPEUDP/MT), RD Gateway (MS-TSGU) and an opt-in legacy-TLS client feature (`native-tls`) are **later** and out of scope for the first milestones.

### Web client

5. **Language and build:** the Web client's protocol logic is the same Rust crates compiled to `wasm32-unknown-unknown` with `wasm-bindgen`; the browser glue (canvas rendering, keyboard and pointer capture, clipboard API) is a thin TypeScript layer.
6. **First protocol: VNC over WSS to our own server.** The default WSS profile uses **RA2_256 (RSA-AES-256)** with username and password; RFB "None" is never accepted. Because current noVNC does not implement RA2_256, independent noVNC interop uses a separate loopback-only CI build with VeNCrypt Plain as the RFB authentication subtype inside WSS. The outer TLS protects that test exchange; the test-only profile is not a release build option. The product Web client continues to use RA2_256.
7. **Rendering:** 2D canvas (`putImageData`) first; WebGL/WebGPU only if profiling shows a need. Video decode, when H.264 is negotiated, through the browser's WebCodecs API ([0008](0008-codec-policy.md)).
8. **UI shell:** evaluate IronRDP's `iron-remote-desktop` component as the shell in M3 (licence and API fit); fall back to our own minimal component if it is too RDP-specific. The evaluation result is recorded in the M3 work item, not a new ADR, unless it changes this decision.
9. **RDP in the browser comes later (M5)**, via an RDCleanPath-compatible endpoint built into our server for connections to itself, and compatible with Devolutions Gateway or a future standalone proxy for connecting to other RDP hosts.
10. **noVNC** (unmodified) is used as an independent test client from M2 on against the loopback-only CI build and its test authentication profile ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)).

## Consequences

- The Web client works with any modern browser and needs no install, and the server needs no extra proxy process.
- The WSS endpoint makes the server an HTTP server too: it needs HTTP parsing limits, origin checks and rate limiting ([0011](0011-security-baseline.md)), and certificate management becomes user-facing (a browser shows an error on a self-signed cert, unlike a native client with TOFU).
- Our Web client only talks to servers that speak RFB over WebSocket (ours, neatvnc/wayvnc, QEMU, or anything behind websockify) until RDP support lands.
- Revisit if WebTransport becomes broadly available and materially better for latency, or if the RDCleanPath design turns out to require terminating TLS in a place we do not want.

## Alternatives considered

- **websockify as a required sidecar.** Extra process, extra install, extra attack surface; our server can do it natively.
- **Ship noVNC as the Web client.** Works today, but it is VNC-only, JavaScript, MPL-2.0, and would not share code with the RDP path. Kept as a test peer and possible optional fallback.
- **VeNCrypt over WSS.** TLS inside TLS in WASM; no benefit when the WSS layer already authenticates the server.
- **Apache Guacamole-style server-side rendering gateway.** A different architecture (protocol translation in a daemon, its own wire protocol); contradicts the "client speaks the protocol" design and Guacamole links GPL libvncclient.
