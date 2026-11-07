# 0010. Linux server capture and input: X11 backend first, then xdg-desktop-portal (PipeWire + libei), compositor-specific later

- **Status:** Proposed
- **Date:** 2026-11-07
- **Deciders:** project owner (to accept)
- **Related:** [risks-and-open-questions.md §4.2–4.3](../research/risks-and-open-questions.md#42-linux-x11), [language-and-architecture.md §1.2](../research/language-and-architecture.md#12-per-language-findings) (binding crates), [0003](0003-protocol-agnostic-core-sans-io-static-backends.md), [0004](0004-server-first-linux-and-web-first.md)

## Context

The server is built first, on Linux ([0004](0004-server-first-linux-and-web-first.md)). It needs frames (with damage regions and cursor), input injection (keyboard, pointer), and later clipboard and audio. The options ([risks §4.2–4.3](../research/risks-and-open-questions.md#42-linux-x11)):

| Option | Capture | Input | Unattended? | Works on | Notes |
|---|---|---|---|---|---|
| **X11** | XShm / XGetImage, XDamage for dirty regions, XFixes for cursor, XRandR for layout | XTest | Yes | Xorg sessions, Xvfb / Xorg-dummy virtual sessions, XWayland *only for X clients* | Mature, unrestricted, a security non-boundary (any X client can do it). Easy to run headless in CI with Xvfb. |
| **xdg-desktop-portal ScreenCast + RemoteDesktop** | PipeWire stream | RemoteDesktop v2: `ConnectToEIS` (libei) preferred, or `Notify*` methods; clipboard in the portal | **No guarantee.** `Start()` shows a dialog; `persist_mode=2` gives a single-use restore token that compositors may ignore. No login-screen access. | GNOME, KDE, wlroots (via -wlr portal backend), others with a portal backend | The only portable Wayland path. |
| **Mutter private D-Bus** (`org.gnome.Mutter.RemoteDesktop`) | PipeWire | EIS | Yes, inside the GNOME session | GNOME | Explicitly "private and not intended to be used outside of the integrated system". gnome-remote-desktop already is an RDP server there. |
| **wlroots / ext protocols** (`ext-image-copy-capture-v1`, `virtual-keyboard`, `wlr-virtual-pointer`) | Wayland protocol | Wayland protocol | Yes (no dialog) | wlroots compositors (Sway, etc.) | wayvnc's approach. |
| **uinput** | – | Kernel virtual input device | Yes | Any | Needs root or `input` group; device-level, so keyboard layout handling is on us; injects into whatever has focus, including other sessions. |

Binding crates exist for all of these in Rust: x11rb, ashpd (portals), pipewire-rs, reis (libei), wayland-client ([language-and-architecture.md §1.2](../research/language-and-architecture.md#12-per-language-findings)). **Unverified:** that ashpd's RemoteDesktop and `ConnectToEIS` support and reis are complete enough for our needs; the M2 spike checks this.

Major desktop distributions now default to Wayland sessions. **Unverified:** the exact status of X11 session support in current GNOME and KDE releases (the research did not cover it); if GNOME no longer ships an X11 session, the X11 backend serves mostly virtual (Xvfb) sessions and non-GNOME desktops.

## Decision

Implement the `FrameSource` / `InputSink` / `ClipboardProvider` traits ([0003](0003-protocol-agnostic-core-sans-io-static-backends.md)) for Linux in this order:

1. **`urdp-platform-x11` first (M2).** XShm + XDamage + XFixes for capture, XTest for input, X selections for clipboard. Two modes: attach to an existing X display (`$DISPLAY`), or spawn a private **Xvfb** session per server instance (xrdp-style virtual desktop). This is the CI workhorse and the unattended/headless story on Linux.
2. **`urdp-platform-wayland` portal backend second (M2, after the X11 path works).** ScreenCast + RemoteDesktop through ashpd, frames via PipeWire, input via `ConnectToEIS` (libei through reis), falling back to `Notify*` methods where EIS is not offered. Restore tokens are stored and reused; when the compositor ignores them, the server reports "needs local approval" rather than failing silently. **This backend is documented as attended access.**
3. **Later (not before M6):** an `ext-image-copy-capture` + virtual-keyboard/pointer backend for wlroots compositors, which gives unattended access there.
4. **Not planned:** Mutter's private D-Bus API (private by its own description; GNOME has its own RDP server) and uinput as a default input path (privileges and layout problems). uinput may be revisited as an explicit opt-in for kiosk/headless boxes.
5. **Login screen and multi-user remote login on GNOME/KDE are out of scope**; they belong to gnome-remote-desktop and KRdp. Documented honestly in the README.
6. Backend selection is automatic (`$WAYLAND_DISPLAY` → portal; `$DISPLAY` → X11; `--virtual` → Xvfb) with an explicit override.
7. The capture and input code runs behind the privilege-separation boundary from [0003](0003-protocol-agnostic-core-sans-io-static-backends.md) point 7, as the logged-in user, never as root.

## Consequences

- We get a server that works unattended and in CI quickly (X11/Xvfb), and a Wayland path that works on every major desktop but needs a person to click "Allow" at least once per session, possibly every time.
- Unattended Wayland access is a known gap until the wlroots backend, and stays a gap on GNOME/KDE except via their own servers. The README must say so.
- Two capture backends to maintain from the start, with different damage models: XDamage gives regions; PipeWire gives buffers with optional damage metadata. The `FrameSource` trait must allow "full frame, no damage info" and the server must diff frames itself in that case.
- `unsafe` code concentrates in these crates (PipeWire buffers, SHM segments), under the rules in [0011](0011-security-baseline.md).
- Revisit if the M2 spike shows the portal + libei path is not workable through existing Rust crates, or if XWayland-only capture is needed (it is not useful for full-desktop sharing).

## Alternatives considered

- **Portal-only** (research Q12, option 1). Portable, but no unattended access and hard to test in CI. Rejected as the *only* backend; kept as the desktop path.
- **Mutter D-Bus for GNOME.** Gives unattended GNOME access but uses an API GNOME says is private; would break without notice.
- **uinput for input everywhere.** Needs privileges and bypasses the compositor's input model.
- **Wayland first, X11 later.** More future-proof, but slows the first working server and CI. Rejected for ordering only; both are in M2.
