# 0005. Build RDP on IronRDP, sspi-rs and picky; write our own VNC; GPL peers only as external test peers

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner
- **Related:** [prior-art.md, "Reuse strategy"](../research/prior-art.md#reuse-strategy), [0002](0002-implementation-language-rust.md), [0006](0006-license-the-project-under-apache-2-0.md), [0013](0013-legal-review-gate-for-rdp.md), [0014](0014-dependency-license-compatibility-policy.md)

## Context

From [prior-art.md](../research/prior-art.md):

- **IronRDP** (Rust, `MIT OR Apache-2.0`, about 80 crates, `ironrdp` 0.17.0; `ironrdp-acceptor` 0.12.0, `ironrdp-server` 0.13.0 on crates.io) covers client and server, TLS 1.2/1.3, NLA via CredSSP, the EGFX pipeline including AVC420/AVC444 PDUs, CLIPRDR, RDPSND, RDPDR, DVCs, display control, MS-TSGU and RDCleanPath for browsers. All crates are pre-1.0. Its server and acceptor crates are "community tier" with a single maintainer.
- **sspi-rs** (0.23.1) and **picky** (7.0.0-rc.29), same licence, provide NTLM, Kerberos (including a server module), Negotiate, CredSSP client and server, KDC proxy, X.509 and ASN.1.
- **FreeRDP and xrdp are Apache-2.0**: porting individual algorithms with attribution is legal, but linking FreeRDP brings back C's attack surface.
- **Mature VNC servers and libvncserver are GPL-2.0-or-later** and cannot be linked into, or copied into, an Apache-2.0 codebase we intend to keep permissive. **neatvnc/wayvnc (ISC)** are usable references; **noVNC (MPL-2.0)** can be used unmodified as a separate component. No mature permissive Rust VNC server exists.

## Decision

1. **RDP:** depend on IronRDP crates, **sspi-rs** and **picky** as normal crates.io dependencies.
   - Pin exact versions (`=x.y.z`) and bump deliberately; run `cargo-semver-checks`-style review on each bump.
   - Wrap them in our own adapter crates (`urdp-rdp` for the client, `urdp-rdp-server` for the server) so the rest of the workspace sees only our traits ([0003](0003-protocol-agnostic-core-sans-io-static-backends.md)).
   - Send fixes and missing features upstream. Use `[patch.crates-io]` with a public fork only as a stop-gap while an upstream PR is pending, and record each patch in `docs/` with a link to the upstream PR.
   - Do not fork IronRDP wholesale unless upstream becomes unmaintained; that would need a new ADR.
2. **VNC/RFB:** write our own sans-IO crates (`urdp-vnc-proto`, `urdp-vnc-server`, `urdp-vnc-client`) from [RFC 6143](https://www.rfc-editor.org/rfc/rfc6143) and the community [rfbproto](https://github.com/rfbproto/rfbproto/blob/master/rfbproto.rst) spec.
   - neatvnc (ISC) may be consulted and, with attribution in `NOTICE`, small pieces adapted.
   - Permissive Rust crates (vnc-rs, rustvncserver) may be consulted or seed code, keeping their notices. Whether to take any of their code is decided per pull request.
3. **GPL and proprietary software is used only as external test peers**: TigerVNC, x11vnc, libvncserver examples, UltraVNC, TightVNC, QEMU's VNC server, GNOME Remote Desktop, PyRDP, RealVNC and Microsoft's clients and servers. They run as separate processes in containers or VMs, are never linked, never vendored into the repository, and never shipped in release artifacts. Contributors must not copy code from them.
4. **FreeRDP** is a reference and interop peer. Porting a specific algorithm from it is allowed with provenance recorded in `NOTICE` and per-file headers checked first. It is not linked into shipped binaries. An optional test-only FFI harness is allowed.
5. **noVNC** may be used unmodified as a test client, and possibly later as an optional separately shipped web client. Its files stay MPL-2.0 and are never mixed into Apache-2.0 files.

## Consequences

- The RDP server and client get NLA, EGFX and channels without writing an RDP stack, at the cost of tracking a fast-moving pre-1.0 API.
- Server-first ([0004](0004-server-first-linux-and-web-first.md)) puts the weakest-maintained part of IronRDP (server, acceptor) on our critical path. We should expect to contribute there, and the plan includes a gap assessment of `ironrdp-server` before RDP server work starts. Candidate gaps the research lists to verify: AVC444 encode, server-side progressive/ClearCodec, RDP-UDP in the acceptor, multi-monitor GFX on the server.
- The VNC code is ours to maintain and secure, including fuzzing every decoder and the security-type handshakes.
- `cargo-deny` can enforce the proposed dependency policy in CI once the owner accepts it ([0014](0014-dependency-license-compatibility-policy.md)).
- Revisit if IronRDP's server crates stall for more than a release cycle on a fix we need, or if Devolutions changes the licence.

## Alternatives considered

- **Write our own RDP stack.** Rejected: years of work duplicating a permissive, fuzzed implementation.
- **Link FreeRDP.** Rejected: C attack surface; FreeRDP fixed 13 advisories in its 3.33.0 release alone.
- **Fork IronRDP.** Rejected for now: splits a small community and multiplies maintenance.
- **Link libvncserver or TigerVNC.** Rejected: GPL-2.0-or-later is incompatible with keeping the project permissively licensed.
- **Base the VNC server on neatvnc via FFI.** Rejected: C dependency in the network-facing process, and its encoders depend on FFmpeg/GnuTLS/nettle.
