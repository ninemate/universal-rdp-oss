# 0006. License the project under Apache-2.0

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner (Apache-2.0 is set at repository creation)
- **Related:** [LICENSE](../../LICENSE), [NOTICE](../../NOTICE), [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [0007](0007-contribution-policy-dco.md), [0014](0014-dependency-license-compatibility-policy.md)

## Context

The repository already contains an Apache-2.0 `LICENSE`, and the README states the goal to keep the project open source under that licence. Apache-2.0 §3 grants patent rights from contributors for their contributions; it does not clear Microsoft's RDP patent exposure or the H.264 patent pool ([risks §1.1](../research/risks-and-open-questions.md#11-microsoft-open-specification-promise-and-the-rdp-specs)).

This ADR records the selected licence for original project material. It does not decide which additional licences are acceptable for linked dependencies; that policy remains Proposed in [0014](0014-dependency-license-compatibility-policy.md). The chosen RDP dependencies and the external-test-peer restriction are recorded separately in [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md).

## Decision

All original code and documentation in this repository is licensed under **Apache License 2.0**. The copyright line is "universal-rdp-oss contributors".

## Consequences

- Downstream users receive the rights and obligations described by Apache-2.0. The licence itself does not provide clearance for Microsoft RDP patents or H.264 patent-pool claims.
- The licence choice is simple to identify in the repository and remains compatible with the selected IronRDP dependencies ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)).
- Inbound contribution records and any future licence change are separate governance questions; the proposed DCO policy is in [0007](0007-contribution-policy-dco.md).

## Alternatives considered

- **MIT OR Apache-2.0 dual licence.** Common in the Rust ecosystem and used by IronRDP, but not the licence chosen when this repository was created. The owner may reconsider before published crates accumulate outside contributions (open item 8 in the [ADR index](README.md#open-items-not-decisions)).
- **MIT only.** Would omit Apache-2.0's express patent grant for contributor contributions; not selected for this repository.
