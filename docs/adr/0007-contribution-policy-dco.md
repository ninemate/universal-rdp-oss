# 0007. Accept contributions under the Developer Certificate of Origin (no CLA)

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [CONTRIBUTING.md](../../CONTRIBUTING.md), [0006](0006-license-the-project-under-apache-2-0.md), [0014](0014-dependency-license-compatibility-policy.md), [risks §6, Q15](../research/risks-and-open-questions.md#6-open-questions-for-the-project-owner)

## Context

[CONTRIBUTING.md](../../CONTRIBUTING.md) currently says that by contributing you agree your contribution is licensed under Apache-2.0 ("inbound = outbound"), with no mechanism to record that agreement. The research left "a CLA, or DCO only?" open, noting that it matters for patent-grant clarity and future relicensing.

The project implements protocols with third-party patent exposure and must not take in code copied from GPL implementations or from Microsoft ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [risks §1.3](../research/risks-and-open-questions.md#13-reverse-engineering-and-dmca)). A per-commit statement of origin helps with both.

## Decision

1. Every commit carries a `Signed-off-by:` trailer certifying the [Developer Certificate of Origin 1.1](https://developercertificate.org/). A DCO check runs in CI and blocks merging without it.
2. No Contributor Licence Agreement. Inbound licence = outbound licence (Apache-2.0, §5 of the licence).
3. CONTRIBUTING.md gains a **provenance rule**: contributors must not copy code from GPL/AGPL/LGPL projects, from Microsoft binaries or non-specification material, or from proprietary clients and servers. Ported permissive code (e.g. FreeRDP, neatvnc) must say where it came from in the PR and in `NOTICE`.
4. Behaviour learned from observing traffic rather than from a published spec is documented in the PR (what was observed, against which peer and version), following the policy in [risks §1.3](../research/risks-and-open-questions.md#13-reverse-engineering-and-dmca).
5. AI-assisted contributions follow the same rules; the contributor signing off is responsible for provenance.

## Consequences

- Low friction for contributors; no legal entity is needed to hold CLAs.
- Relicensing later would need consent from all contributors, so any change of outbound licence (e.g. to `MIT OR Apache-2.0`, see [0006](0006-license-the-project-under-apache-2-0.md)) should be decided before outside contributions accumulate.
- Maintainers must check sign-offs and provenance notes during review.

## Alternatives considered

- **CLA (individual and corporate).** Gives the project relicensing freedom and an explicit patent grant, but deters contributors and needs an entity to hold the agreements.
- **No mechanism (status quo).** Weakest record of origin, which matters more than usual for this protocol space.
