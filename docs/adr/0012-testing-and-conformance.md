# 0012. Testing and conformance strategy

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [language-and-architecture.md §4](../research/language-and-architecture.md#4-testing-strategy), [prior-art.md, interop matrix](../research/prior-art.md#reuse-strategy), [0003](0003-protocol-agnostic-core-sans-io-static-backends.md), [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [0011](0011-security-baseline.md)

## Context

There is no conformance test suite for RFB, and Microsoft's RDP implementations are the de-facto oracle for RDP. Sans-IO crates ([0003](0003-protocol-agnostic-core-sans-io-static-backends.md)) make deterministic in-memory testing cheap. Server-first ([0004](0004-server-first-linux-and-web-first.md)) means our early oracles are third-party *clients*. Several CI facts are unverified in the research: `/dev/kvm` on standard GitHub runners, in-runner loopback RDP on Windows runners, and the licence position of Windows evaluation images in public CI.

## Decision

Tests are organised in layers. Each layer has a CI home.

| Layer | What | Where it runs |
|---|---|---|
| **L1 Unit and byte vectors** | Every PDU/message: decode bytes → struct and struct → bytes, against captured bytes or spec examples (RFC 6143; Section 4 of each MS-RDP\* document). Vectors in `testdata/*.bin`, loaded with `include_bytes!`; snapshots with `insta` or `expect-test`. | Every PR, all targets including `wasm32` (build + core tests) |
| **L2 Property tests** | `proptest`: `decode(encode(x)) == x`, `encoded_len() == encode().len()`, state-machine invariants ("no framebuffer data before auth", "no input accepted before auth"). | Every PR |
| **L3 In-memory sessions** | Our client machine ↔ our server machine (and loopback backend) with injected clock and scripted faults (truncation, reorder, oversize lengths). | Every PR |
| **L4 Golden images** | Encoders/decoders against reference images: exact hash for lossless codecs; PSNR/SSIM thresholds for lossy ones; our encoder output decoded by an independent decoder (FFmpeg, FreeRDP, or a third-party client) where possible. | Every PR (small set), nightly (full set) |
| **L5 Fuzzing** | One cargo-fuzz target per parser and state machine, structure-aware with `arbitrary`; round-trip oracles. | Smoke on PRs; nightly corpus runs; OSS-Fuzz later ([0011](0011-security-baseline.md)) |
| **L6 Interop (containers)** | Linux peers in containers on hosted runners: for our **VNC server** — TigerVNC viewer (`vncviewer` in a scripted/headless mode or a test harness), noVNC in headless Chromium, and libvncclient-based tools; for our **RDP server** — FreeRDP (`xfreerdp /auth-only` and scripted sessions), IronRDP's own client. Server under test runs on Xvfb. | Nightly and on release branches; required check once stable |
| **L7 Interop (lab)** | mstsc and Windows App (Windows, macOS, iOS/Android), Remmina, RealVNC Viewer, macOS Screen Sharing; later Windows hosts for our RDP client. | Self-hosted runners or manual lab runs before each release; results recorded in `docs/interop/` |
| **L8 Performance** | Frame-pipeline benchmarks (capture → encode → send) with `criterion`; tracked over time. | Nightly; regressions flagged, not blocking at first |

Rules:

1. **GPL and proprietary peers run only as separate processes** (containers, VMs) and are never vendored ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)).
2. **Every interop run is recorded** (pcap + TLS key log under a debug flag) so failures become offline regression vectors (L1/L3).
3. **A bug fixed means a test added** at the lowest layer that reproduces it.
4. **CI matrix:** Linux x86_64 (primary), Linux aarch64 (build + L1–L3), `wasm32-unknown-unknown` (core crates + Web client build, headless browser test for the Web client from M3). Windows and macOS runners join in M6.
5. **Windows in CI** is experimental until verified; authoritative Windows interop is self-hosted. **Unverified:** `/dev/kvm` availability on standard runners and whether evaluation images in public CI are licence-compliant; both are checked before any Windows VM is added to CI (and the second goes to counsel with [0013](0013-legal-review-gate-for-rdp.md)).
6. **Conformance tracking:** each protocol crate keeps a feature matrix (`docs/conformance/rfb.md`, `docs/conformance/rdp-server.md`, …) listing message types, encodings, security types and capability sets as implemented / tested against which peer / not planned.

## Consequences

- Most protocol bugs are caught without a network or a display.
- The interop layers need maintained container images and, later, lab machines; that is ongoing cost.
- Early server testing relies on third-party clients, which is fine for VNC (many mature clients) and for RDP (FreeRDP in CI, mstsc/Windows App in the lab).
- Revisit when OSS-Fuzz accepts the project, or if container interop proves too flaky to be a required check.

## Alternatives considered

- **Interop-only testing against real peers.** Slow, flaky and hard to debug; kept as the top layers only.
- **Differential fuzzing against libvncserver/FreeRDP as a requirement.** Useful but optional; runs sandboxed if added.
