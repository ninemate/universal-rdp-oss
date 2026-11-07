# 0011. Security baseline: secure defaults, TLS, NLA/CredSSP, certificate handling, fuzzing and `unsafe` policy

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [risks-and-open-questions.md §3](../research/risks-and-open-questions.md#3-security), [language-and-architecture.md §4.5](../research/language-and-architecture.md#45-supply-chain-and-code-security-practices), [vnc-protocol.md §3](../research/vnc-protocol.md#3-security-types), [0003](0003-protocol-agnostic-core-sans-io-static-backends.md), [0009](0009-transports-and-web-client.md), [0012](0012-testing-and-conformance.md)

## Context

RDP and VNC have a long record of pre-auth server bugs (BlueKeep, DejaBlue), client-side "reverse RDP" bugs, and weak legacy security modes (Standard RDP Security with RC4; VNC DES auth with 8-character passwords; RFB "None"). Thousands of VNC servers sit on the internet without authentication. FreeRDP has about 325 GitHub advisories, concentrated in codecs and dynamic channels ([risks §3.2](../research/risks-and-open-questions.md#32-lessons-from-past-rdp-vulnerabilities)). Server-first ([0004](0004-server-first-linux-and-web-first.md)) means our first public binary is a listening service.

Two research questions remain open for the owner and are answered here as proposals: support for legacy security modes (Q6) and NTLM (Q7).

## Decision

### Defaults (server)

1. **No unauthenticated or unencrypted listener off loopback.** The server refuses to start on a non-loopback address unless TLS and authentication are configured.
2. **RDP server:** offers only TLS + NLA (CredSSP, `HYBRID`/`HYBRID_EX`). Never offers Standard RDP Security (`PROTOCOL_RDP`) or TLS without NLA by default; there is no option to enable Standard RDP Security on our server at all.
3. **VNC server:** offers VeNCrypt X509Plain (system/PAM credentials over a certificate) and RA2_256 by default. VncAuth only when an admin explicitly sets a legacy password (needed for macOS Screen Sharing and some viewers), with a warning in logs. "None" only on loopback or a UNIX socket. Never Plain without TLS, TLSNone without warning, RA2ne or MSLogonII ([vnc-protocol.md recommendations](../research/vnc-protocol.md#recommendations-for-universal-rdp-oss)).
4. **WSS endpoint:** TLS always; RFB "None" never; Origin header checked against a configured allowlist; RA2_256 inside by default ([0009](0009-transports-and-web-client.md)). The separate noVNC CI fixture uses VeNCrypt Plain only over WSS, binds to loopback, uses disposable test credentials, and is not included in release builds.
5. **Rate limiting and lockout** for failed authentication per source address and per account.
6. **Parse as little as possible before authentication.** Pre-auth code paths are listed in the threat model and fuzzed first.

### Defaults (client, for M5)

7. Standard RDP Security and VNC "None"/VncAuth to remote hosts are **client-side opt-ins** behind explicit flags (e.g. `--insecure-legacy-rdp-security`) with a persistent warning. TLS 1.0/1.1 to legacy RDP hosts only via an opt-in `legacy-tls` feature.
8. **Kerberos first, NTLMv2 as fallback** for workgroup hosts; NTLMv1 never.
9. Device, drive, printer, smartcard and USB redirection off by default, per connection. Clipboard file names sanitised (reject `..`, absolute paths, alternate data streams, device names).

### TLS and certificates

10. rustls, TLS 1.2 and 1.3 only; a curated cipher-suite list (rustls defaults).
11. **Server certificates:** the server generates a self-signed certificate on first start (stored with 0600 permissions) and prints its SPKI SHA-256 fingerprint; an admin can supply a CA-issued certificate and key instead (needed for browsers). Certificate reload without restart.
12. **Client verification:** WebPKI/OS trust store when the certificate chains to a trusted CA; otherwise trust-on-first-use with the SPKI fingerprint pinned per host:port, a loud warning on change, and no silent acceptance ever.

### NLA / CredSSP

13. CredSSP via sspi-rs: TSRequest **v6** with `clientNonce` and the pubKeyAuth hash binding, bound to the TLS server public key; refuse versions below 5 by default (CVE-2018-0886).
14. **Server-side NLA on Linux:** NTLM acceptor against a pluggable credential backend: a local password file (hashed) first, PAM later. **Unverified:** how PAM can back an NTLM acceptor, since NTLM needs the NT hash, not the cleartext password; a Kerberos keytab path may be the realistic PAM-integrated option. The M4 spike resolves this.

### Memory safety and `unsafe`

15. `#![forbid(unsafe_code)]` in every core-tier crate (types, session, protocol, pure-Rust codecs) and in the runtime and transport crates.
16. `unsafe` is allowed only in `urdp-platform-*`, `urdp-codec-*` (FFI) and `*-sys` crates. Each block has a `// SAFETY:` comment (enforced by `clippy::undocumented_unsafe_blocks`), each crate's `unsafe` surface is listed in its README, and new `unsafe` needs a second reviewer.
17. Miri on core tests; ASan/UBSan nightly on FFI crates; `cargo-geiger` report in CI.
18. **Panics, assertion failures and unbounded allocation from network input are security bugs.** Decoders take explicit size limits; `#![deny(clippy::unwrap_used, clippy::expect_used, clippy::indexing_slicing)]` in protocol crates (with reviewed exceptions); `overflow-checks = true` in release profiles for protocol crates is evaluated in M1.
19. Cryptography only from reviewed crates (rustls/ring or aws-lc-rs, RustCrypto). RSA for RA2 must avoid known timing-side-channel issues; **Unverified:** the current status of the RustCrypto `rsa` crate's Marvin-attack advisory (RUSTSEC-2023-0071), to be checked before RA2 is implemented.

### Fuzzing and supply chain

20. Every parser and state machine gets a cargo-fuzz target **in the same PR that adds it**; CI runs a short smoke fuzz per target, nightly runs keep a corpus; apply to OSS-Fuzz once the first parsers stabilise ([0012](0012-testing-and-conformance.md)).
21. `cargo-deny` (licences, advisories, sources, bans) on every PR; `cargo-audit` nightly; `cargo-vet` before the first release.
22. Secrets are wrapped in redacting newtypes; TLS key logging only behind an explicit debug flag.
23. `SECURITY.md` with a private disclosure address and embargo process before the first release; GitHub private vulnerability reporting enabled; signed releases and an SBOM per release.
24. A written threat model (pre-auth surface, server privilege separation, clipboard and file exfiltration, WSS/HTTP surface) before the first release; an external audit before 1.0.

## Consequences

- Some old viewers and hosts will not connect without explicit opt-ins; that is intended.
- Browsers will show certificate errors with the self-signed default; docs must explain using a real certificate.
- The lint and `unsafe` rules add friction for contributors; CI enforces them so reviewers do not have to.
- Revisit Kerberos/NTLM defaults when Microsoft disables network NTLM by default in the next major Windows release ([risks §3.1](../research/risks-and-open-questions.md#31-credssp--nla)).

## Alternatives considered

- **Offer legacy modes on the server for compatibility.** Rejected: the old clients that need them are a tiny minority and the modes are broken cryptographically.
- **Drop VncAuth entirely.** Rejected: macOS Screen Sharing and some viewers need it; kept as an explicit admin opt-in.
- **NTLM-only server NLA.** Rejected as the end state; acceptable as the first step for local accounts.
