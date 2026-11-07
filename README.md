# universal-rdp-oss

An open-source remote access **client and server** harness and protocol layer that can speak existing remote desktop protocols, starting with Microsoft RDP and VNC/RFB.

> **Status: early / pre-alpha.** Nothing is implemented yet. This repository currently contains only documentation. Do not use it for anything real.

## Vision

One protocol-agnostic core (sessions, input, display, clipboard, audio, file transfer) with pluggable protocol backends, so the same client or server can work with different remote access protocols, and new protocols can be added without rewriting the core.

## Goals

- Support both roles: client (connect to a remote machine) and server (expose a machine).
- Cover RDP legacy and modern (including NLA/CredSSP and RDPEGFX) and VNC/RFB.
- Keep the core independent of any single protocol, with room for more backends.
- Support TCP, TLS and WebSocket transports.
- Stay fully open source under Apache-2.0.

## Non-goals

- Not a fork of an existing RDP or VNC implementation. The RDP backends build on IronRDP, sspi-rs and picky as dependencies; the VNC backend is our own code ([ADR 0005](docs/adr/0005-rdp-on-ironrdp-vnc-from-scratch.md)).
- Not a hosted service or a commercial product.
- Not inventing a new wire protocol as a first step.

## Planned protocol support

Everything below is **planned**; nothing is implemented.

| Protocol | Client | Server |
|---|---|---|
| VNC / RFB | Planned (browser first, then native) | Planned, **first** (Linux) |
| RDP modern (TLS, NLA/CredSSP, RDPEGFX) | Planned | Planned, after VNC (Linux) |
| RDP legacy (Standard RDP Security) | Proposed: opt-in only, with warnings | Proposed: never offered |
| Others | Possible later | Possible later |

- **Order:** server first, on Linux, plus a browser (WebAssembly) client over WebSocket; Windows and macOS later ([ADR 0004](docs/adr/0004-server-first-linux-and-web-first.md)).
- **Language:** Rust ([ADR 0002](docs/adr/0002-implementation-language-rust.md)).
- **Codecs:** operating-system codecs by default; Cisco OpenH264 only as an opt-in download; no bundled patent-encumbered codecs ([ADR 0008](docs/adr/0008-codec-policy.md)).
- **RDP and patents:** the RDP specifications are not covered by Microsoft's Open Specification Promise. RDP work starts only after a lawyer has reviewed the patent exposure ([ADR 0013](docs/adr/0013-legal-review-gate-for-rdp.md)). VNC work is not affected.
- **Name:** "universal-rdp-oss" is a working name. The research flags "RDP", "Remote Desktop" and "VNC" as naming/trademark concerns; the current registry status of "RDP" is unverified. The name may change before a first release ([open items](docs/adr/README.md#open-items-not-decisions)).

## Documentation

- [Architecture (draft)](docs/architecture.md)
- [Roadmap](docs/ROADMAP.md)
- [Implementation plan](docs/plans/implementation-plan.md) (milestones M0–M7 and work items)
- [Architecture Decision Records](docs/adr/README.md)
- [Research survey](docs/research/README.md)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

Apache License 2.0, copyright universal-rdp-oss contributors. See [LICENSE](LICENSE).
