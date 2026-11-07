# 0004. Build the server role first, on Linux, with a Web client early

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner
- **Related:** [research README, "Decisions needed"](../research/README.md#decisions-needed-from-the-project-owner), [risks-and-open-questions.md §4](../research/risks-and-open-questions.md#4-platform-capture-and-input-server-side), [implementation plan](../plans/implementation-plan.md), [0010](0010-linux-server-capture-and-input.md), [0013](0013-legal-review-gate-for-rdp.md)

## Context

The research asked "client first or server first?" and assumed client first, because the client is better specified, can be tested against real Windows hosts today, and carries less platform risk ([risks-and-open-questions.md §6, Q1](../research/risks-and-open-questions.md#6-open-questions-for-the-project-owner)). Its prioritised task list put the RDP and VNC clients before any server work.

The owner has decided differently:

- **Server role first**, client after.
- **Platforms first: Linux and Web** (a browser client over WebSocket). Windows and macOS later.
- VNC work is not blocked by the RDP legal review ([0013](0013-legal-review-gate-for-rdp.md)); RDP work is.

## Decision

We will deliver in this order (details in the [implementation plan](../plans/implementation-plan.md)):

1. Workspace, CI and core abstractions.
2. **VNC/RFB server on Linux.**
3. **Web client** (WASM in the browser) talking to our server over WebSocket, starting with VNC.
4. **RDP server on Linux** through IronRDP, once the legal gate in [0013](0013-legal-review-gate-for-rdp.md) is passed.
5. Native clients (VNC, then RDP), and RDP in the Web client.
6. Windows and macOS (server and client).

VNC comes before RDP inside the server track because it is not gated by the legal review and the protocol is much smaller, so it validates the core, capture and transport layers sooner.

## Consequences

### Added risk compared with the research's client-first order

These are the risks the owner accepts with this ordering, and how the plan mitigates them:

| Risk | Why server-first makes it worse | Mitigation in the plan |
|---|---|---|
| **Unattended capture and input are blocked by platform security** (R5). On Wayland, portals need a user dialog and restore tokens are single-use and may be ignored by the compositor; login-screen access belongs to GNOME/KDE's own servers. On Windows, the secure desktop needs a SYSTEM service plus a per-session agent. On macOS, TCC re-prompts and the persistent-capture entitlement needs an application to Apple. ([risks §4](../research/risks-and-open-questions.md#4-platform-capture-and-input-server-side)) | The first deliverable depends on the hardest platform work instead of deferring it. | Start on Linux where X11 has no such restrictions; ship the portal path as attended-only and say so; prototype capture in a spike before committing to the server MVP ([0010](0010-linux-server-capture-and-input.md)). Windows and macOS stay in the last milestone. |
| **Larger pre-auth attack surface ships first** (R3). A listening server is reachable by anyone on the network; a client only talks to servers the user picks. | Our first public binary is the riskier role. | Secure defaults from day one ([0011](0011-security-baseline.md)): loopback-only or TLS + auth, no "None" auth off loopback, fuzzing before the first release, size caps on every allocation. |
| **Dependence on IronRDP's server crates** (`ironrdp-server`, `ironrdp-acceptor`), which IronRDP marks "community tier" with a single maintainer. | The RDP server is our first RDP deliverable instead of the better-maintained client crates. | Adapter crate, pinned versions, upstream contributions, readiness to carry a `[patch]` or fork ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)). |
| **Encoder work comes before decoder work.** Servers must encode (damage tracking, H.264 or RemoteFX), which is the expensive part ([rdp-protocol.md §6](../research/rdp-protocol.md#6-server-vs-client-responsibilities)). | CPU cost and codec licensing hit early. | VNC encodings (ZRLE, Tight/JPEG) need no patented codec; RDP uses RemoteFX/planar first; H.264 only via OS/hardware or opt-in OpenH264 ([0008](0008-codec-policy.md)). |
| **No first-party client to test against.** | Early testing relies on third-party clients. | Plenty exist: TigerVNC viewer, noVNC, Remmina, FreeRDP, mstsc, Windows App. GPL ones run only as external test peers ([0005](0005-rdp-on-ironrdp-vnc-from-scratch.md)). |
| **CI realism.** Wayland portal capture needs a running compositor and portal backend; this is hard in containers. | Server tests need a display. | X11/Xvfb backend first for CI; headless compositor (e.g. sway headless) for Wayland tests, accepted as best effort. **Unverified:** that xdg-desktop-portal-wlr works non-interactively in CI. |

### Other consequences

- The Web client is the first client we write. It forces the WebSocket transport and the `wasm32` build of the core early, which is good for the architecture.
- The RDP *client* (which the research expected to be the first deliverable) moves to M5. Kerberos/IAKerb work and broad Windows interop move with it.
- Revisit the order if the Linux capture spike shows that neither X11 nor portals give a usable server on the target distributions.

## Alternatives considered

- **Client first** (research default). Lower platform risk and a clear oracle (Windows hosts). Not chosen by the owner.
- **RDP server before VNC server.** Rejected: RDP is gated on the legal review and is a much larger protocol; VNC proves the same core and capture path sooner.
- **Windows server first.** Rejected by the owner's platform choice; Windows also needs the service-plus-agent architecture and driver signing for virtual displays.
