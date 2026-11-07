# Architecture Decision Records

This directory records the project's significant architecture and policy decisions, one per file, in the MADR-style format defined by [template.md](template.md). Why and how we use ADRs is itself a decision: [0001](0001-record-architecture-decisions.md).

- **Accepted** means the project owner has decided. **Proposed** means a recommendation (usually from [`docs/research/`](../research/README.md)) that the owner has not yet accepted; work may proceed on it, but it can still change without a superseding ADR.
- [`docs/architecture.md`](../architecture.md) describes the current design and [`docs/plans/implementation-plan.md`](../plans/implementation-plan.md) the delivery order; both link back here for the reasons.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions as ADRs | Accepted |
| [0002](0002-implementation-language-rust.md) | Implement in Rust (reviewed for flaws under server-first; none found) | Accepted |
| [0003](0003-protocol-agnostic-core-sans-io-static-backends.md) | Protocol-agnostic sans-IO core with statically linked protocol backends | Proposed |
| [0004](0004-server-first-linux-and-web-first.md) | Build the server role first, on Linux, with a Web client early (and the risk this adds) | Accepted |
| [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md) | RDP on IronRDP, sspi-rs and picky; our own VNC; GPL software only as external test peers | Accepted |
| [0006](0006-license-the-project-under-apache-2-0.md) | License the project under Apache-2.0 | Accepted |
| [0007](0007-contribution-policy-dco.md) | Contributions under the DCO, no CLA, with a provenance rule | Proposed |
| [0008](0008-codec-policy.md) | OS codecs by default; OpenH264 only as opt-in download; no bundled patent-encumbered codecs | Accepted |
| [0009](0009-transports-and-web-client.md) | TCP/TLS/WebSocket transports; built-in WSS endpoint; WASM Web client, VNC first | Proposed |
| [0010](0010-linux-server-capture-and-input.md) | Linux capture/input: X11 first, then portals (PipeWire + libei), wlroots later | Proposed |
| [0011](0011-security-baseline.md) | Security baseline: secure defaults, TLS, NLA/CredSSP, certificates, fuzzing, `unsafe` policy | Proposed |
| [0012](0012-testing-and-conformance.md) | Layered testing and conformance strategy | Proposed |
| [0013](0013-legal-review-gate-for-rdp.md) | Gate RDP work on a lawyer's review of RDP patent exposure; VNC not gated | Accepted |
| [0014](0014-dependency-license-compatibility-policy.md) | Set a dependency license compatibility policy | Proposed |

## Gates

| Gate | Blocks | Defined in |
|---|---|---|
| **G1 Legal review of RDP patent exposure** | All RDP implementation work (M4, RDP parts of M5) | [0013](0013-legal-review-gate-for-rdp.md) |
| **G2 Project and crate name settled** | Publishing crates to crates.io, first tagged release, any public branding | Open item 1 below |

## Open items (not decisions)

These are known open questions. They are tracked here, not given ADR numbers, until someone decides them.

1. **Project name and trademarks.** The project stays `universal-rdp-oss` for now. The research notes that "VNC" is a RealVNC trademark and "Remote Desktop" is a Microsoft product name, that Microsoft's guidelines forbid its marks in product names, and that a name containing "rdp" carries some risk; a neutral name with a "speaks RDP and VNC" tagline is lower risk ([risks §1.2](../research/risks-and-open-questions.md#12-trademarks), R11). **Unverified:** current registry status of "RDP", "Remote Desktop", "RFB". Crate names use the placeholder prefix `urdp-`. Must be settled before G2.
2. **Who engages and pays for counsel** for G1, and for any H.264 or Microsoft licensing questions from commercial redistributors (research Q8).
3. **GUI toolkit for the native client** (research Q4). Accessibility (UIA, AT-SPI, NSAccessibility) and Wayland maturity should weigh heavily. Needed by M5.
4. **Windows server scope** (research Q10): console sharing only vs multi-session; funding IddCx driver signing. Needed by M6.
5. **macOS distribution** (research Q13): who holds the Apple Developer ID, notarises, and applies for the `persistent-content-capture` entitlement (reported multi-month wait; apply early). Needed well before M6.
6. **AVD / Windows 365 / Entra** (research Q11): later tier or out of scope; who pays for a test tenant.
7. **Security process extras** (research Q14): OSS-Fuzz enrolment timing, bug bounty, CVE CNA arrangement.
8. **Licence variant before first release:** stay Apache-2.0 only, or move to `MIT OR Apache-2.0` like IronRDP while relicensing is cheap ([0006](0006-license-the-project-under-apache-2-0.md)).
9. **API stability guarantees** for published crates (semver policy before 1.0).

## Adding an ADR

1. Copy [template.md](template.md) to `NNNN-short-title.md` with the next free number.
2. Set status **Proposed** unless the project owner has decided.
3. Add a row to the index above in the same PR.
4. If it changes an earlier decision, mark the old one "Superseded by NNNN" and link both ways.
