# 0014. Set a dependency license compatibility policy

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept or revise)
- **Related:** [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [0006](0006-license-the-project-under-apache-2-0.md), [0008](0008-codec-policy.md), [risks §1 and §2](../research/risks-and-open-questions.md)

## Context

The project's original code is licensed under Apache-2.0 ([0006](0006-license-the-project-under-apache-2-0.md)). The owner has selected IronRDP, sspi-rs and picky as RDP dependencies and has directed that GPL VNC servers and clients be used only as external test peers, never linked ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)). Those choices do not by themselves settle how to handle every transitive, optional or separately distributed component.

The research proposes a strict allowlist and a `cargo-deny` gate. The details below extend the accepted choices and need owner review; they are an engineering policy proposal, not a legal opinion about compatibility in every jurisdiction or distribution model.

## Decision

**Proposed:** adopt the following dependency policy, subject to owner review and revision before it becomes a merge-blocking CI rule.

1. Linked dependencies (compiled into or statically linked with project binaries or libraries) use a reviewed permissive licence compatible with Apache-2.0. The initial allowlist is Apache-2.0, MIT, BSD-2-Clause, BSD-3-Clause, ISC, Zlib, Unicode-3.0 / Unicode-DFS-2016, CC0-1.0, or a dual licence including one of these.
2. Do not link GPL, AGPL, LGPL, SSPL, BUSL or other source-available components into default builds. Reconsider LGPL or other exceptions only through an ADR that specifies the distribution and relinking obligations.
3. An optional, dynamically loaded component outside default builds (for example, an LGPL FFmpeg build or the Cisco OpenH264 binary permitted by [0008](0008-codec-policy.md)) requires an explicit opt-in and its own licence review.
4. Keep separately distributed MPL-2.0 components unmodified and identifiable; do not combine their source files into Apache-2.0 files.
5. Any test peer may run as a separate process in CI or a lab, but is never vendored or shipped. This general rule preserves the owner's specific GPL-peer restriction in [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md).
6. Maintain `NOTICE` attribution for any ported code and run `cargo-deny` against the accepted allowlist. A licence failure blocks merging only after the owner accepts this policy.

## Consequences

- The policy can prevent accidental incompatible links, but the allowlist and exceptions need maintenance as dependencies change.
- Some useful libraries and codec builds would not be linkable under the proposal; test peers can still run externally.
- The optional dynamic-loading exception adds security and packaging work and does not grant patent rights.

## Alternatives considered

- **Review each dependency only when it is added, without an allowlist.** Less initial policy work, but transitive dependency changes could enter without review.
- **Allow any licence compatible with Apache-2.0 on a case-by-case basis.** More flexible, but requires a documented legal review for every exception and gives weaker automated CI checks.
