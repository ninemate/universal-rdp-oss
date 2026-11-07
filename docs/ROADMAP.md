# Roadmap

Phases are ordered but not dated. The detailed plan, with acceptance criteria and work items, is [`docs/plans/implementation-plan.md`](plans/implementation-plan.md); the reasons are in [`docs/adr/`](adr/README.md).

The order is **server first, on Linux, with a Web client early**; Windows and macOS come later ([ADR 0004](adr/0004-server-first-linux-and-web-first.md)). This differs from the research's client-first suggestion and puts unattended screen capture and input on the critical path early; the ADR lists the added risks and mitigations.

0. **Research** (done). Survey of RDP and VNC specs, prior art, licensing and language: [`docs/research/`](research/README.md). Decisions recorded as ADRs.
1. **M0 Workspace and CI.** Rust Cargo workspace, CI with lints, licence gate, `wasm32` build, `unsafe` policy, fuzz scaffolding, `SECURITY.md`.
2. **M1 Core abstractions.** Sans-IO session contract, protocol-neutral types, server-side platform traits, TCP/TLS/WebSocket transports, tokio driver, in-memory loopback backend.
3. **M2 VNC server on Linux.** Our own RFB server: X11/Xvfb capture first, then Wayland via desktop portals (attended); VeNCrypt and RA2_256 by default; ZRLE and Tight/JPEG encoders.
4. **M3 Web client.** Browser client (Rust compiled to WASM) over the server's built-in WSS endpoint, VNC first.
5. **M4 RDP server on Linux** via IronRDP, sspi-rs and picky. TLS + NLA only. **Blocked until a lawyer has reviewed RDP patent exposure** ([ADR 0013](adr/0013-legal-review-gate-for-rdp.md)); VNC work is not blocked.
6. **M5 Clients.** Native VNC and RDP client on Linux; RDP in the Web client.
7. **M6 Windows and macOS.** Server (Windows service + session agent; macOS ScreenCaptureKit with TCC) and clients; unattended wlroots backend on Linux.
8. **M7 Hardening and release.** Threat model, signed releases and SBOM, `cargo-vet`, OSS-Fuzz, packaging, external audit. Release needs the project name settled first.

Later, unscheduled: RD Gateway, RDP UDP transport, audio, drive redirection, AVC444, multi-monitor, RemoteApp, AVD/Windows 365.
