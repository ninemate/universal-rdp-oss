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

- Not a fork or replacement of an existing RDP or VNC implementation (reuse is under evaluation, see the architecture doc).
- Not a hosted service or a commercial product.
- Not inventing a new wire protocol as a first step.

## Planned protocol support

Everything below is **planned**; nothing is implemented.

| Protocol | Client | Server |
|---|---|---|
| RDP legacy (classic security, RDP security layer) | Planned | Planned |
| RDP modern (TLS, NLA/CredSSP, RDPEGFX) | Planned | Planned |
| VNC / RFB | Planned | Planned |
| Others | TBD | TBD |

Implementation language and reuse of existing projects are undecided (TBD, pending `docs/research`).

## Documentation

- [Architecture (draft)](docs/architecture.md)
- [Roadmap](docs/ROADMAP.md)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

Apache License 2.0, copyright universal-rdp-oss contributors. See [LICENSE](LICENSE).
