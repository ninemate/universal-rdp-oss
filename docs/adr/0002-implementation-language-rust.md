# 0002. Implement in Rust

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner ("follow the research recommendation unless a flaw is found")
- **Related:** [language-and-architecture.md](../research/language-and-architecture.md), [0003](0003-protocol-agnostic-core-sans-io-static-backends.md), [0004](0004-server-first-linux-and-web-first.md), [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [0011](0011-security-baseline.md)

## Context

The product parses attacker-controlled binary protocols (RDP PDUs, RFB messages, bitmap and video codecs, compressed streams), often before authentication, on both the client and the server side. FreeRDP, the most complete open implementation, fixed security advisories in most of its 2026 releases ([language-and-architecture.md §1.1](../research/language-and-architecture.md#11-memory-safety-the-policy-and-data-context)).

The research compared Rust, Go, C/C++ and Zig on weighted criteria ([§1.3](../research/language-and-architecture.md#13-scored-comparison)): Rust 95.8, C/C++ 72.2, Go 71.2, Zig 57.6. The deciding facts:

- Rust is memory safe without a garbage collector.
- It has permissively licensed, maintained RDP (IronRDP), CredSSP/NTLM/Kerberos (sspi-rs, picky) and TLS (rustls) libraries. Go's RDP libraries are GPL or unfinished and it has no CredSSP; C means FreeRDP's attack surface; Zig is pre-1.0.
- It has a working `wasm32` target, and IronRDP's web client already runs in browsers. This matters because Web is a first platform ([0004](0004-server-first-linux-and-web-first.md)).

### Review for flaws under the owner's decisions

The research assumed client-first. The owner chose server-first on Linux plus a Web client ([0004](0004-server-first-linux-and-web-first.md)). We checked whether that changes the outcome:

- **Server-first raises the weight of FFI to OS APIs** (PipeWire, xdg-desktop-portal over D-Bus, libei, X11, VA-API). Rust bindings exist for all of them (pipewire-rs, ashpd, reis, x11rb, cros-libva) but several are young, and some are thin wrappers over C libraries, so `unsafe` code concentrates in the server's platform crates. A rough sensitivity check that doubles the FFI weight (10 → 20, renormalised) gives Rust ≈ 94, C/C++ ≈ 75, Go ≈ 70. This is our arithmetic on the research's scores, not a new evaluation. The ranking does not change.
- **Server-first raises the dependency on IronRDP's server and acceptor crates**, which IronRDP itself classes as "community tier" with a single maintainer ([language-and-architecture.md, risks table](../research/language-and-architecture.md#risks-of-the-rust-choice-and-mitigations)). This is a risk of the reuse strategy ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)), not of the language: no other language has a permissive RDP server library at all.
- **Web first** favours Rust further.

**No flaw found that changes the recommendation.** The two points above are recorded as consequences.

## Decision

We will implement universal-rdp-oss in **Rust**.

## Proposed implementation details (not accepted by this ADR)

These are recommendations to carry into the workspace and architecture review; they are not owner decisions yet:

- Use one Cargo workspace and stable Rust pinned with `rust-toolchain.toml`; declare an MSRV and check it in CI. A candidate policy is the stable release that is six months old at each minor release; decide the exact policy in M0.
- Use **tokio** for native async I/O and `wasm-bindgen-futures` in the browser, as proposed in [0003](0003-protocol-agnostic-core-sans-io-static-backends.md).
- Confine `unsafe` to platform, codec and `*-sys` crates under the proposed rules in [0011](0011-security-baseline.md).

## Consequences

- Protocol code gets compile-time protection against use-after-free, out-of-bounds access and data races. Panics, unbounded allocation and logic bugs remain, and are handled by the security baseline ([0011](0011-security-baseline.md)) and fuzzing ([0012](0012-testing-and-conformance.md)).
- The browser client can share protocol crates with native builds if the proposed `wasm32` and sans-IO constraints in [0003](0003-protocol-agnostic-core-sans-io-static-backends.md) hold.
- Contributor pool is smaller than for Go or C++, and compile times need active management (no proc-macros in core crates, `sccache`, `mold` in CI).
- We will probably have to write and own some `-sys` or binding crates for hardware encoders and for any portal or PipeWire features the existing crates lack.
- Revisit if IronRDP is abandoned *and* no Rust fork is viable, or if a security audit finds that most real defects sit in our `unsafe` FFI rather than in safe code.

## Alternatives considered

- **C/C++ on FreeRDP.** Most complete today; rejected for memory safety and because it would make the project a FreeRDP distribution. FreeRDP stays a reference and interop peer ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)).
- **Go.** Memory safe with excellent networking; rejected because CredSSP would have to be written from scratch, its RDP libraries are GPL-3.0 or incomplete, and GC plus cgo per-frame costs are a poor fit for the server's frame pipeline.
- **Zig.** Rejected: pre-1.0 (0.17.1) with documented miscompilations and no protection against use-after-free.
- **C#/.NET, Swift, Java.** Rejected in the research without a scored column: runtime weight, weak cross-platform capture, or no RDP stack.
