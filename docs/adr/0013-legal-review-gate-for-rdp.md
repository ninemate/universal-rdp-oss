# 0013. Gate RDP work on a lawyer's review of RDP patent exposure; VNC is not gated

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner
- **Related:** [rdp-protocol.md §7](../research/rdp-protocol.md#7-licensing-and-patents), [risks-and-open-questions.md §1](../research/risks-and-open-questions.md#1-legal-and-patent), [0004](0004-server-first-linux-and-web-first.md), [0005](0005-rdp-on-ironrdp-vnc-from-scratch.md), [0008](0008-codec-policy.md)

## Context

From the research (not legal advice):

- **No MS-RDP\*, MS-CSSP, MS-TSGU or MS-RDPEGFX document is on Microsoft's Open Specification Promise or Community Promise lists.** The Community Promise covers only lower ITU layers (T.123, T.124, T.128, X.224).
- Every RDP spec's IP notice says Microsoft "has patents that might cover your implementations" and that the documentation grants no patent licence; licences are available via `iplg@microsoft.com`.
- Microsoft's **Patent Pledge for Open Source Developers** (MIP / Interoperability Principles) is a non-assert for open-source developers that **excludes commercial distribution**. Whether the RDP specs are in its scope depends on the Patent and Program Map, which the research could not read (**Unverified**). Whether the older MCPP pledge covers RDP is also **Unverified**.
- FreeRDP, xrdp, IronRDP and GNOME Remote Desktop have shipped for years without known action. That is a risk signal, not clearance.
- H.264 is a separate patent pool, handled by [0008](0008-codec-policy.md).

The RFB protocol is openly specified (RFC 6143 and the community spec) with no comparable patent notice ([vnc-protocol.md §7](../research/vnc-protocol.md#7-ipr-licensing-and-trademark-notes)).

## Decision

1. **Gate G1 (legal):** before any RDP implementation work starts (client or server, including the IronRDP adapter crates), a qualified lawyer reviews the RDP patent exposure and gives written advice. The review covers at least:
   - whether the MIP open-source pledge, the Interoperability Principles or the MCPP pledge cover the MS-RDP\* documents we plan to implement (MS-RDPBCGR, MS-CSSP, MS-RDPEGFX, MS-RDPEDYC, MS-RDPECLIP, MS-RDPEDISP, MS-RDPRFX, and later MS-RDPEA, MS-RDPEUDP/EMT, MS-TSGU), checked by hand against the Patent and Program Map;
   - the position of the project itself (non-commercial open-source distribution, publishing release binaries) and of downstream commercial redistributors;
   - whether to write to Microsoft (`iplg@microsoft.com` / `protocol@microsoft.com`);
   - the wording for README and NOTICE;
   - as a secondary item, whether Windows evaluation images in public CI are licence-compliant ([0012](0012-testing-and-conformance.md)).
2. Until G1 is passed, the RDP milestone (M4) and RDP parts of M5 do not start. Reading the specs, assessing IronRDP's API (desk research) and designing the core so that RDP fits are allowed.
3. **VNC/RFB work is not gated** and proceeds immediately ([0004](0004-server-first-linux-and-web-first.md)).
4. The outcome is recorded as a new ADR (or a supersession of this one) and a short `docs/legal/rdp-patents.md`, without privileged detail the lawyer advises against publishing.
5. Whatever the outcome, the project never claims to be licensed or endorsed by Microsoft, implements only from published specs and observed wire behaviour, and never copies Microsoft code or binaries ([risks §1.3](../research/risks-and-open-questions.md#13-reverse-engineering-and-dmca)).

## Consequences

- The RDP server's start date depends on finding and, if needed, paying for counsel. The plan schedules M2–M3 (VNC, Web) so they can run while G1 is pending.
- If counsel advises against RDP or attaches conditions (for example "no official binaries with RDP"), the RDP milestones change; the core stays protocol-agnostic so the VNC product stands alone.
- Who engages and pays for counsel is an open item (see the [ADR index](README.md#open-items-not-decisions)).

## Alternatives considered

- **Proceed like FreeRDP/IronRDP without review.** Rejected by the owner.
- **Gate VNC too.** Rejected: no patent notice on RFB comparable to Microsoft's, and the owner explicitly excluded it.
