# Prior art: open-source and proprietary RDP / VNC implementations

Status: research survey, snapshot as of **2026-11-07**. Versions, dates and licenses were checked against primary sources (GitHub API, LICENSE/COPYING files, source headers, crates.io API, project release notes). Facts I could not check are marked **Unverified:**.

## Summary

- **The Rust RDP ecosystem has consolidated around Devolutions' IronRDP** ([MIT OR Apache-2.0](https://github.com/Devolutions/IronRDP/blob/master/Cargo.toml)). It has about 80 crates covering client and server: an acceptor state machine, a server skeleton, EGFX with H.264/AVC420, RDP-UDP, TS Gateway, clipboard, audio, device redirection and a WASM web client. Several third-party RDP servers already build on it. With [sspi-rs](https://github.com/Devolutions/sspi-rs) (NTLM, Kerberos, Negotiate, CredSSP, client and server side) and [picky-rs](https://github.com/Devolutions/picky-rs) (ASN.1, X.509, Kerberos structures), it covers most of what universal-rdp-oss needs for RDP. All three are license-compatible with Apache-2.0.
- **FreeRDP and xrdp are Apache-2.0, not GPL.** They can be read, ported from, or linked, with attribution. FreeRDP 3.33.0 (2026-11-02) is the most complete open RDP implementation. It covers client, `libfreerdp-server`, shadow server, proxy, H.264, AV1, gateway and experimental UDP. Most Linux RDP servers (GNOME Remote Desktop, KRDP, Weston) are built on `freerdp-server3`.
- **The VNC world is mostly GPL.** TigerVNC, libvncserver/libvncclient, x11vnc, UltraVNC and TightVNC 2.x are GPL-2.0 or GPL-3.0 family. TightVNC 4.x is now closed source. Use them only as interop test peers. The permissive exceptions are **neatvnc/wayvnc (ISC)**, **noVNC (MPL-2.0)**, and the Rust crates **vnc-rs (MIT OR Apache-2.0, client)** and **rustvncserver (Apache-2.0, server)**.
- **Recommendation in one line:** depend on IronRDP, sspi-rs and picky for RDP; write our own RFB engine, optionally seeded from vnc-rs and rustvncserver; use FreeRDP, xrdp, mstsc/Windows App, TigerVNC, libvncserver and noVNC as interop peers in CI. Details are in [Reuse strategy](#reuse-strategy).

## Comparison table

C = client, S = server. "Latest" means the latest release or tag found on 2026-11-07.

| Project | Lang | License (SPDX) | Latest (date) | Role | Protocols / notable features | Platforms | Activity | Reuse for us |
|---|---|---|---|---|---|---|---|---|
| [FreeRDP](https://github.com/FreeRDP/FreeRDP) | C | Apache-2.0 | 3.33.0 (2026-11-02) | C + S + proxy | RDP: NLA, GFX, H.264, AV1, RemoteFX, progressive, RDG/TSG gateway, experimental UDP, RAIL, all main channels | Linux, Windows, macOS, iOS, Android, WASM | Very active | Reference + port source; test peer; optional FFI |
| [IronRDP](https://github.com/Devolutions/IronRDP) | Rust | MIT OR Apache-2.0 | `ironrdp` 0.17.0 (2026-07-10); core crates 2026-10-27 | C + S + web | RDP: NLA (CredSSP/NTLM/Kerberos), RFX, EGFX/AVC420, RDP-UDP, MS-TSGU, CLIPRDR, RDPSND, RDPDR, DVC, RAIL | Native, WASM, .NET | Very active | **Primary dependency** |
| [sspi-rs](https://github.com/Devolutions/sspi-rs) | Rust | MIT OR Apache-2.0 | `sspi` 0.23.1 (2026-10-27) | Auth lib | NTLM, Kerberos (client and server), Negotiate, PKU2U, CredSSP, smart card | Cross-platform | Very active | **Dependency** |
| [picky-rs](https://github.com/Devolutions/picky-rs) | Rust | MIT OR Apache-2.0 | `picky` 7.0.0-rc.29, `picky-krb` 0.13.1 (2026-10-26) | Crypto/PKI lib | ASN.1 DER, X.509, Kerberos ASN.1, JOSE | Cross-platform | Active | Transitive dependency |
| [xrdp](https://github.com/neutrinolabs/xrdp) | C | Apache-2.0 | v0.10.6.1 (2026-07-07) | S (+ RDP/VNC proxy) | GFX (0.10.0+), H.264 (0.10.2+), RFX, clipboard, audio, mic, drive; **no NLA** except VMConnect mode | Linux, BSD | Active (many CVE fixes) | Test peer; code reference |
| [xorgxrdp](https://github.com/neutrinolabs/xorgxrdp) | C | X11-style (GitHub: NOASSERTION) | v0.10.5 (2026-01-28) | Xorg driver for xrdp | Framebuffer capture | Linux | Active | Reference only |
| [rdesktop](https://github.com/rdesktop/rdesktop) | C | GPL-3.0 | v1.9.0 (2019-10-11) | C | RDP 4/5 | Unix | **Unmaintained** | None |
| mstsc / [Windows App](https://learn.microsoft.com/en-us/windows-app/overview) | n/a | Proprietary | n/a | C (and Windows RDS = S) | Reference implementation | Windows, macOS, iOS, Android, web | Vendor | **Primary interop peer** |
| [GNOME Remote Desktop](https://gitlab.gnome.org/GNOME/gnome-remote-desktop) | C | GPL-2.0-or-later | 51.0 (2026-09-14) | S (RDP + VNC) | FreeRDP 3 server, AVC via VA-API/Vulkan/NVENC, audio in/out, camera, smart card, headless/GDM, Kerberos login | Linux (GNOME) | Active | Test peer; design reference |
| [KRDP](https://invent.kde.org/plasma/krdp) | C++ | LGPL-2.1-only OR LGPL-3.0-only (lib); some GPL-2.0-or-later | v6.8.2 (2026-10-27) | S | FreeRDP 3 server, H.264 over GFX, portal/PipeWire | Linux (Plasma) | Active | Test peer |
| [Weston RDP backend](https://gitlab.freedesktop.org/wayland/weston) | C | MIT | 16.0.1 (2026-10-19), 15.0.2 (2026-11-02) | S | FreeRDP 3 (or 2) server backend | Linux | Active | Reference (MIT) |
| [TigerVNC](https://github.com/TigerVNC/tigervnc) | C++ | GPL-2.0-or-later | v1.16.2 (2026-03-26) | C + S | RFB 3.8, Tight/ZRLE, VeNCrypt/TLS, Xvnc, w0vncserver (Wayland) | Linux, Windows, macOS | Active | **Test peer only** |
| [TightVNC](https://www.tightvnc.com/) | C++ | 2.x: GPL-2.0; 4.x: proprietary | 4.0.2 (2026-10-28) | C + S | Tight encoding, file transfer | Windows (4.x/2.x) | Vendor | Test peer only |
| [UltraVNC](https://github.com/ultravnc/UltraVNC) | C++ | GPL-3.0-or-later | 1.8.3.2 (Oct/Nov 2026) | C + S + repeater | File transfer, MS-Logon, encryption plugins, repeater | Windows | Active | Test peer only |
| RealVNC Connect | n/a | Proprietary | **Unverified** | C + S | RA2 (RSA-AES) | Multi | Vendor | Test peer (licensed) |
| [noVNC](https://github.com/novnc/noVNC) | JS | MPL-2.0 (core), BSD-2-Clause (HTML/CSS) | v1.7.0 (2026-04-28) | C (browser) | WebSocket RFB; RA2, VeNCrypt, ARD, MSLogonII; Tight, ZRLE, H.264 | Browser | Active | Test peer; optional bundled web client (MPL) |
| [LibVNCServer/Client](https://github.com/LibVNC/libvncserver) | C | GPL-2.0-or-later | 0.9.15 (2024-12-22) | C + S libs | Many encodings and security types, WebSockets | Cross-platform | Active commits, slow releases | Test peer only |
| [x11vnc](https://github.com/LibVNC/x11vnc) | C | GPL-2.0 | 0.9.17 (2025-05-01) | S | X11 scraping | Unix | Dormant | Test peer |
| [wayvnc](https://github.com/any1/wayvnc) + [neatvnc](https://github.com/any1/neatvnc) | C | ISC | wayvnc v0.10.2 (2026-09-25); neatvnc v1.0.4 (2026-11-01) | S | Raw/ZRLE/Tight/H.264 (FFmpeg), VeNCrypt (GnuTLS), RSA-AES (nettle), audio | Linux (wlroots) | Active | Reference (ISC, copyable); test peer |
| [vnc-rs](https://github.com/HsuJv/vnc-rs) | Rust | MIT OR Apache-2.0 | 0.6.0 (2026-09-21) | C | Raw/Tight/ZRLE/TRLE; None and VncAuth only; WASM | Native, WASM | Maintained | Seed or reference for client |
| [rustvncserver](https://crates.io/crates/rustvncserver) | Rust | Apache-2.0 | 2.2.1 (2026-02-09) | S | 11 encodings, VncAuth, reverse/repeater | Cross-platform | Young, single author | Seed or reference for server |
| [Apache Guacamole](https://github.com/apache/guacamole-server) | C / Java | Apache-2.0 | 1.6.0 (2025-06-22) | Gateway (C) | RDP via FreeRDP 2/3, VNC via libvncclient, SSH, telnet, Kubernetes | Linux server, browser | Active | Architecture reference |
| [RustDesk](https://github.com/rustdesk/rustdesk) | Rust | AGPL-3.0 | 1.5.1 (2026-11-01) | C + S | Own protocol (not RDP/VNC) | Multi | Very active | None (AGPL) |
| [Remmina](https://gitlab.com/Remmina/Remmina) | C | GPL-2.0-or-later + OpenSSL exception | v1.4.43 (tag 2026-02-20) | C frontend | FreeRDP (2/3), libvncclient, SSH, SPICE | Linux | Low | Test peer (client) |
| [MeshCentral](https://github.com/Ylianst/MeshCentral) | JS | Apache-2.0 (but `rdp/` is GPL-3.0) | 1.3.1 (2026-10-23) | Web RMM | Own agent protocol; web RDP via node-rdpjs fork | Node.js | Very active | None |
| [rdp-rs](https://github.com/citronneur/rdp-rs) | Rust | MIT | v0.1.0 (2020-04-11) | C | Early RDP, NLA | Native | Dormant | Historical reference |
| [Myrtille](https://github.com/myrtille-rdp/myrtille) | C# | Apache-2.0 | v2.9.2 (year **Unverified**) | Web gateway | HTML5 RDP/SSH via FreeRDP fork | Windows Server/IIS | Low | None |
| [aardwolf](https://github.com/skelsec/aardwolf) | Python | MIT | 0.2.17 (2026-11-07) | C (headless) | RDP (CredSSP NTLM/Kerberos) + VNC; scanner | Cross-platform | Active | Scripted test client |
| [PyRDP](https://github.com/GoSecure/pyrdp) | Python | GPL-3.0 | v2.1.0 (2024-01-23) | MITM | RDP MITM, replay, credential capture | Cross-platform | Low | Security testing tool |

## RDP projects

### FreeRDP

- **License:** Apache-2.0 ([LICENSE](https://github.com/FreeRDP/FreeRDP/blob/master/LICENSE); README says "released under the Apache license").
- **Latest release:** 3.33.0 on 2026-11-02 ([release](https://github.com/FreeRDP/FreeRDP/releases/tag/3.33.0), [ChangeLog](https://github.com/FreeRDP/FreeRDP/blob/master/ChangeLog)). Its changelog adds WebAssembly support, "experimental UDP support for direct (without gateway) RDP connections", and the return of an "experimental windows shadow server", along with 13 GHSA advisories. The 3.x line ships every few weeks: 3.29.0 (2026-07-14) through 3.33.0.
- **FreeRDP 2:** still exists on the `stable-2.0` branch. The last tag is 2.11.8, dated 2026-02-24. **Unverified:** I found no formal end-of-life statement for 2.x. All the downstream servers checked here (GNOME Remote Desktop, KRDP, Weston) now require or prefer FreeRDP 3.
- **Client side:** front-ends for X11, Wayland, SDL, Windows, Mac, iOS and Android are in [`client/`](https://github.com/FreeRDP/FreeRDP/tree/master/client).
  - Codecs in [`libfreerdp/codec`](https://github.com/FreeRDP/FreeRDP/tree/master/libfreerdp/codec): H.264 (FFmpeg, OpenH264, MediaFoundation, MediaCodec, VideoToolbox, WebCodecs), AV1, RemoteFX, progressive, ClearCodec, NSCodec, planar, interleaved.
  - Bulk compression: MPPC, NCRUSH, XCRUSH.
  - Gateways in [`libfreerdp/core/gateway`](https://github.com/FreeRDP/FreeRDP/tree/master/libfreerdp/core/gateway): RDG over HTTP and WebSocket, legacy TSG over RPC-over-HTTP, Azure ARM.
- **Server side:** [`server/`](https://github.com/FreeRDP/FreeRDP/tree/master/server) contains the shadow server, the proxy, sample servers, and Mac and Windows variants.
  - Server-side channel implementations are under [`channels/*/server`](https://github.com/FreeRDP/FreeRDP/tree/master/channels): rdpgfx, cliprdr, rdpsnd, audin, disp, rdpdr, rdpecam and others.
  - The `freerdp-server3` library is the engine behind GNOME Remote Desktop, KRDP and Weston (see below).
- **For us:** FreeRDP is the best open behavioural reference and the most important open interop peer for both our client and our server. Because it is Apache-2.0, we may port algorithms and code. Examples are codec details where IronRDP lacks a piece, such as AVC444 or AV1, and gateway corner cases. The obligations are to keep copyright notices and add a NOTICE entry.
  - **Unverified:** I did not audit every FreeRDP source file for third-party code under other licenses. Check per-file headers before copying.
  - FFI-linking FreeRDP from Rust is possible: the [`freerdp-sys`](https://crates.io/crates/freerdp-sys) crate exists, but it is brand new (0.1.0, 42 downloads). It would bring C memory-safety exposure, and FreeRDP 3.33.0 alone fixed 13 advisories. Not recommended as a core dependency.

### IronRDP (Devolutions)

- **License:** `MIT OR Apache-2.0` ([workspace Cargo.toml](https://github.com/Devolutions/IronRDP/blob/master/Cargo.toml), [README](https://github.com/Devolutions/IronRDP#license)). Cargo authors are "Devolutions Inc." and "Teleport".
- **Releases:** the [`ironrdp`](https://crates.io/crates/ironrdp) meta crate is at 0.17.0 (2026-07-10). The latest GitHub release is `ironrdp-viewer-v0.1.0`, also 2026-07-10. Core crates were republished on 2026-10-27/28:
  - [`ironrdp-connector`](https://crates.io/crates/ironrdp-connector) 0.12.0
  - [`ironrdp-acceptor`](https://crates.io/crates/ironrdp-acceptor) 0.12.0
  - [`ironrdp-egfx`](https://crates.io/crates/ironrdp-egfx) 0.5.0
  - [`ironrdp-cliprdr`](https://crates.io/crates/ironrdp-cliprdr) 0.9.0
  - [`ironrdp-rdpsnd`](https://crates.io/crates/ironrdp-rdpsnd) 0.11.0
  - [`ironrdp-rdpdr`](https://crates.io/crates/ironrdp-rdpdr) 0.9.0
  - [`ironrdp-mstsgu`](https://crates.io/crates/ironrdp-mstsgu) 0.1.0
  - [`ironrdp-rdpeudp`](https://crates.io/crates/ironrdp-rdpeudp) 0.1.0

  [`ironrdp-server`](https://crates.io/crates/ironrdp-server) is 0.13.0 on crates.io and 0.14.0 on `master`. All crates are pre-1.0, so expect API churn.
- **Crates:** the [`crates/`](https://github.com/Devolutions/IronRDP/tree/master/crates) directory holds about 80 crates, including:
  - PDU and core: `ironrdp-pdu`, `-core`, `-connector`, `-acceptor`, `-session`, `-server`.
  - Runtime glue: `-tokio`, `-blocking`, `-futures`, `-async`, `-tls`.
  - Codecs and graphics: `-graphics`, `-bulk`, `-nscodec`, `-egfx`.
  - Channels: `-cliprdr`, `-rdpsnd`, `-rdpdr`, `-dvc`, `-displaycontrol`, `-rail`, `-ainput`, `-rdpei`, `-rdpecam`, `-rdpeusb`, `-echo`.
  - Transports: `-rdpeudp`, `-mstsgu`, `-rdcleanpath`.
  - Other: `-vmconnect` (Hyper-V console), `-w365` (Windows 365 DVCs), `-rdpfile`, `-web`, `iron-remote-desktop`, plus fuzzing and test suites.
- **Features per the [README](https://github.com/Devolutions/IronRDP/blob/master/README.md):**
  - The core is sans-I/O and `no_std`-compatible, and it is fuzzed.
  - Security: TLS 1.2/1.3 and NLA via CredSSP (NTLM, Kerberos). KDC proxy and RDCleanPath are supported.
  - Client decoding: RemoteFX and RDP 6.0 bitmaps. Server encoding: RDP 6.0, RemoteFX, NSCodec and QOI.
  - Codec primitives: ClearCodec, RFX Progressive, ZGFX.
  - Channels: CLIPRDR, RDPSND, RDPDR (including smart card), display control, USB.
  - [`ironrdp-egfx`](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-egfx) implements MS-RDPEGFX "including H.264/AVC420 and AVC444". OpenH264 is optional and can be bundled or loaded at runtime via libloading, which preserves Cisco's patent coverage.
- **Server:** [`ironrdp-server`](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-server) is an "extendable skeleton" on Tokio. You implement the `RdpServerInputHandler` and `RdpServerDisplay` traits.
  - The code supports `with_hybrid` (CredSSP/NLA), RemoteApp window orders, an `egfx` feature that bridges `GraphicsPipelineServer` for H.264 streaming ([gfx.rs](https://github.com/Devolutions/IronRDP/blob/master/crates/ironrdp-server/src/gfx.rs)), and USB.
  - The crate README's feature list is shorter than what the code does.
  - [`ironrdp-acceptor`](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-acceptor) authenticates CredSSP against configured credentials by default. On Windows it can use Windows Negotiate instead.
- **Web:** [`web-client/`](https://github.com/Devolutions/IronRDP/tree/master/web-client) contains `iron-remote-desktop`, a protocol-agnostic web component published as `@devolutions/iron-remote-desktop` (tag `npm-iron-remote-desktop-v0.11.0`). It also contains the RDP WASM module and a Svelte demo client.
- **Adoption:** listed users are Devolutions Gateway, Cloudflare browser RDP, Teleport, Lamco RDP Server, MacRDP, qemu-rdp and wayshare ([README "Who uses IronRDP"](https://github.com/Devolutions/IronRDP#who-uses-ironrdp)).
- **For us:** IronRDP is the obvious foundation for the RDP backend on both client and server. Its sans-I/O design fits a protocol-agnostic core. Its web component is explicitly protocol-agnostic and could in principle host a VNC backend too.

### sspi-rs and picky-rs (Devolutions)

- **[sspi](https://crates.io/crates/sspi) 0.23.1** (2026-10-27), `MIT OR Apache-2.0`.
  - Modules: NTLM, Kerberos (with a [server module](https://github.com/Devolutions/sspi-rs/tree/master/src/kerberos/server)), Negotiate, PKU2U, CredSSP (including Restricted Admin and credential-less modes), smart card, KDC proxy and channel bindings ([src/](https://github.com/Devolutions/sspi-rs/tree/master/src)).
  - It can call native SSPI on Windows.
  - The GitHub release `v2026.11.05.0` (2026-11-05) is the native/FFI packaging.
  - The workspace also ships DPAPI, `kdc` and `winscard` crates.
- **[picky](https://crates.io/crates/picky) 7.0.0-rc.29**, [picky-krb](https://crates.io/crates/picky-krb) 0.13.1 and [picky-asn1](https://crates.io/crates/picky-asn1) 0.10.2 (all 2026-10-26), `MIT OR Apache-2.0`. Picky covers X.509, PKI, JOSE and Kerberos ASN.1. It has no GitHub releases and has stayed on 7.0.0 release candidates for a long time.
- **For us:** take these as transitive dependencies through IronRDP. Use sspi directly for server-side NLA and for gateway authentication.

### xrdp and xorgxrdp

- **License:** xrdp is **Apache-2.0** ([COPYING](https://github.com/neutrinolabs/xrdp/blob/devel/COPYING)). xorgxrdp uses an X11-style permissive license ([COPYING](https://github.com/neutrinolabs/xorgxrdp/blob/devel/COPYING)), which GitHub reports as NOASSERTION.
- **Releases:** xrdp v0.10.6.1 is a security release fixing 10 CVEs, published on GitHub on 2026-07-07; its notes are dated 2026/07/06 ([NEWS](https://github.com/neutrinolabs/xrdp/blob/v0.10.6.1/NEWS.md)). xorgxrdp v0.10.5 was released 2026-01-28.
- **Graphics:** GFX arrived in v0.10.0 (2024-05-10) without AVC444. H.264 encoding arrived in v0.10.2 (2024-12-24) via x264 or OpenH264.
- **Other features ([README](https://github.com/neutrinolabs/xrdp/blob/devel/README.md)):** clipboard (text, bitmap, file), audio and microphone (through extra PulseAudio modules), drive redirection, session reconnection, resizing, and RDP/VNC proxying to Xvnc.
- **No NLA:** in [`xrdp_iso.c`](https://github.com/neutrinolabs/xrdp/blob/devel/libxrdp/xrdp_iso.c) the code says "At present we only support SSL and RDP security". It offers HYBRID only in VMConnect mode.
- **For us:** xrdp is an essential server interop peer because many real Linux deployments run it. Its session-manager design (sesman, chansrv) is a useful reference for our multi-session Linux server. Apache-2.0 allows reuse, but the C code is not a strong porting source compared with IronRDP. Note that its optional x264 encoder is GPL-licensed.

### rdesktop

The [README](https://github.com/rdesktop/rdesktop/blob/master/README.md) opens with "IMPORTANT: rdesktop is no longer maintained", warns of unreviewed vulnerabilities, and lists RDP 4/5 only. The license is GPL-3.0 (GitHub), and the last release is v1.9.0 (2019-10-11). It is historical only and not worth using even as a test peer, except perhaps for testing the legacy "Standard RDP Security" path.

### Microsoft clients and servers (closed, testing only)

- **[Windows App](https://learn.microsoft.com/en-us/windows-app/overview)** runs on Windows, macOS, iOS/iPadOS, Android/ChromeOS, web browsers and Meta Quest.
  - It connects to Azure Virtual Desktop, Windows 365, Dev Box, Remote Desktop Services and PCs.
  - On Windows, the overview says to keep using the Remote Desktop app for RDS connections.
  - Remote PC connections in the Windows build became GA in 2.0.1482.0 (October 2026) ([what's new](https://learn.microsoft.com/en-us/windows-app/whats-new)).
- **Retirements:** the Microsoft Store "Remote Desktop" app lost support and availability as of 2025-05-27 ([client what's new](https://learn.microsoft.com/en-us/previous-versions/remote-desktop-client/whats-new-windows)). The MSI client page gives conflicting end-of-support dates (2026-03-27 for public cloud and "September 28, 2026").
- **mstsc.exe:** **Unverified:** I found no current Microsoft page that confirms mstsc ships unchanged in Windows 11 25H2/26H1. It is widely assumed but not checked here.
- **For us:** mstsc, Windows App on macOS/iOS/Android, and Windows Server/Windows 11 RDS hosts are the conformance targets. They need licensed Windows VMs in CI or a lab. Microsoft's [Open Specifications](https://learn.microsoft.com/en-us/openspecs/windows_protocols/) (MS-RDPBCGR, MS-RDPEGFX, MS-RDPEUDP and others) are the normative source.

### GNOME Remote Desktop

- **License:** GPL-2.0-or-later (COPYING and source headers in a clone of [gnome-remote-desktop](https://gitlab.gnome.org/GNOME/gnome-remote-desktop)).
- **Release:** 51.0, tagged 2026-09-14 ([tags](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/tags)).
- **Dependencies:** [`meson.build`](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/blob/main/meson.build) requires `freerdp3`, `freerdp-server3` and `winpr3` >= 3.28.0 for RDP, and `libvncserver` for VNC. It also uses PipeWire, libei, FUSE (clipboard files), Vulkan, VA-API and NVENC/CUDA (AVC encoding).
- **Features:** source files cover GFX/AVC, audio playback and input, camera redirection, display control and smart card.
  - 51.rc added "Kerberos multi user remote login".
  - 51.beta added smart card redirection.
  - GDM headless remote login is supported.
- **For us:** an important server interop peer on GNOME, and a design reference for Wayland capture through portals and PipeWire, libei input, and hardware-encoded AVC. Do not copy its code (GPL).

### KRDP (KDE)

- **Release:** v6.8.2 (2026-10-27) per the [repo](https://invent.kde.org/plasma/krdp) tags.
- **License:** the library files are `LGPL-2.1-only OR LGPL-3.0-only OR LicenseRef-KDE-Accepted-LGPL`. Several files, such as `RdpConnection.cpp` and `VideoStream.cpp`, are `GPL-2.0-or-later` (SPDX headers).
- **Dependencies:** `CMakeLists.txt` requires FreeRDP 3.1, FreeRDP-Server and KPipeWire. It streams H.264 over the GFX pipeline (`VideoStream.h`, `sendFrameH264`).
- **For us:** a test peer, and a small, readable example of a FreeRDP-server-based desktop-sharing server.

### Weston RDP backend

- **License and releases:** Weston is MIT ([COPYING](https://gitlab.freedesktop.org/wayland/weston/-/blob/main/COPYING)). Recent releases are 16.0.0 (2026-07-14), 16.0.1 (2026-10-19) and 15.0.2 (2026-11-02) ([releases](https://wayland.freedesktop.org/releases.html)).
- **RDP backend:** [`libweston/backend-rdp/meson.build`](https://gitlab.freedesktop.org/wayland/weston/-/blob/main/libweston/backend-rdp/meson.build) prefers `freerdp3`/`freerdp-server3` >= 3.0.0 and falls back to `freerdp2` >= 2.3.0.
- **WSLg:** **Unverified:** Microsoft's WSLg is reported to use Weston's RDP backend, but I did not check that against WSLg sources.
- **For us:** an MIT-licensed reference for a compositor acting as an RDP server, including RAIL/RemoteApp-style window remoting.

## VNC / RFB projects

### TigerVNC

- **License:** GPL-2.0-or-later (source headers, [LICENCE.TXT](https://github.com/TigerVNC/tigervnc/blob/master/LICENCE.TXT)).
- **Release:** v1.16.2 (2026-03-26).
- **Components ([README](https://github.com/TigerVNC/tigervnc/blob/master/README.rst)):** a cross-platform FLTK `vncviewer`, `Xvnc`, `x0vncserver`, the new `w0vncserver` for Wayland, and a `vncserver@.service` systemd unit. The Windows `winvnc` server is "currently unmaintained".
- **For us:** the de facto Linux VNC server and viewer, and therefore the primary RFB interop peer. GPL means no code reuse.

### TightVNC

The [licensing](https://www.tightvnc.com/licensing.php) and [download](https://www.tightvnc.com/download.php) pages say TightVNC 4.x (4.0.2, 2026-10-28) is **closed source**, free for personal and commercial use, Windows only, with paid TLS 1.3 ("Pro"). TightVNC 2.8.90 and legacy 1.3.10 remain GPL-2.0 with source downloads, and commercial source licenses are offered. **Unverified:** the release dates of 2.8.90 and 1.3.10. For us it is a Windows test peer; Tight encoding itself is documented in the RFB community spec.

### UltraVNC

- **License and release:** GPL-3.0-or-later per the [1.8.3.2 release page](https://uvnc.com/downloads/ultravnc/175-ultravnc-1-8-3-2.html); GitHub reports GPL-3.0. That page says "October 2026" while [uvnc.com](https://uvnc.com/) says "November 2026". A 1.8.4.4 beta also exists.
- **Platform and features:** Windows only (Windows 7+). Features include file transfer, encryption plugins and a separate repeater. 1.8.3.2 "retire[s] MS-Logon II".
- **For us:** test peer for the repeater, MSLogon and UltraVNC extensions.

### RealVNC

RealVNC Connect is proprietary. Per the [rfbproto README](https://github.com/rfbproto/rfbproto), RealVNC maintains the RFB specification and allocates extension numbers; [rfbproto.rst](https://github.com/rfbproto/rfbproto/blob/master/rfbproto.rst) is the community version. **Unverified:** RealVNC pages returned HTTP 403, so I could not confirm its current third-party-viewer compatibility modes. The rfbproto repo also has no LICENSE file.

### noVNC

- **License:** the core JS is MPL-2.0; HTML/CSS are BSD-2-Clause; some images are CC-BY-SA-3.0 ([LICENSE.txt](https://github.com/novnc/noVNC/blob/master/LICENSE.txt)).
- **Release:** v1.7.0 (2026-04-28).
- **Features ([README](https://github.com/novnc/noVNC#features)):**
  - Auth: none, VNC, RealVNC RSA-AES, Tight, VeNCrypt Plain, XVP, Apple DH and MSLogonII.
  - Encodings: raw, copyrect, rre, hextile, tight, tightPNG, ZRLE, JPEG, Zlib and H.264.
- **For us:** a browser test client for our VNC server's WebSocket endpoint. We could also ship it unmodified as an optional web client: MPL-2.0 is file-level copyleft, so this is fine alongside Apache-2.0 code, but any changes to its files must stay MPL-2.0.

### LibVNCServer / LibVNCClient and x11vnc

- **libvncserver:** COPYING is GPLv2 and source headers say "see GPL (latest version)", so in practice GPL-2.0-or-later. The last release is LibVNCServer-0.9.15 (2024-12-22), and the repo is still pushed (2026-11-03).
- **Coverage ([README](https://github.com/LibVNC/libvncserver#rfb-protocol-support-status)):** the client supports TLS, VeNCrypt, SASL, ARD and MSLogonII. The server supports WebSockets. TRLE is client-side only.
- **Licensing consequence:** this is **not LGPL**, so anything linking libvncclient is a GPL combined work when distributed. That includes Guacamole's guacd, Remmina and GNOME Remote Desktop's VNC side.
- **x11vnc:** GPL-2.0 (GitHub). 0.9.17 (2025-05-01) is the last release and the last push, so it is dormant.
- **For us:** test peers only. The Rust `libvnc`, `libvnc-sys` and `vncserver` binding crates inherit GPL obligations, so avoid them.

### wayvnc and neatvnc

- **License:** both ISC.
- **Releases:** [neatvnc](https://github.com/any1/neatvnc) v1.0.4 (2026-11-01) and [wayvnc](https://github.com/any1/wayvnc) v0.10.2 (2026-09-25).
- **neatvnc** describes itself as a "liberally licensed VNC server library". Its [`meson.build`](https://github.com/any1/neatvnc/blob/master/meson.build) builds Raw, ZRLE and Tight encoders, optional H.264 via FFmpeg/libavcodec plus libdrm/GBM, VeNCrypt via GnuTLS, nettle-based crypto (RSA-AES, RFB DES) and audio.
- **wayvnc** targets wlroots compositors only; GNOME, KDE and Weston are "not supported".
- **For us:** the only modern, actively maintained, permissively licensed C VNC server. ISC allows porting encoder logic, such as damage refinement and parallel deflate, with attribution. It is also a good test peer for H.264-over-RFB.

### Rust VNC crates

Verified via the crates.io API:

| Crate | Version (date) | License | Scope | Notes |
|---|---|---|---|---|
| [vnc-rs](https://crates.io/crates/vnc-rs) | 0.6.0 (2026-09-21) | MIT OR Apache-2.0 | Async client engine (Tokio), WASM-capable | ~2.4M downloads. Raw, Tight, ZRLE, TRLE. The source enumerates many security types but implements **only None and VncAuth** ([auth.rs/connector.rs](https://github.com/HsuJv/vnc-rs/tree/main/src/client)). Has decoder size limits. |
| [rustvncserver](https://crates.io/crates/rustvncserver) | 2.2.1 (2026-02-09) | Apache-2.0 | Server | Encodings via [`rfb-encodings`](https://crates.io/crates/rfb-encodings) (Raw, CopyRect, RRE, CoRRE, Hextile, Zlib, Tight, ZlibHex, ZRLE, ZYWRLE, TightPng). VncAuth only. Reverse connections and UltraVNC repeater. NOTICE claims a spec-based implementation. Single author. **Unverified:** the GitHub README URL returned 404, so the repo may have moved; I read the crate tarball instead. |
| [rfb](https://crates.io/crates/rfb) (Oxide) | 0.1.0 (2022-05-02) | MPL-2.0 | Server | Minimal; stale. |
| [vnc](https://crates.io/crates/vnc) (whitequark) | 0.4.0 (2016-08-29) | MIT/Apache-2.0 | Client + proxy | Abandoned since 2016. **Unverified:** GitHub archive status (API rate-limited). |
| `libvnc`, `libvnc-sys`, `vncserver` | various | MIT bindings | FFI to libvncserver | Underlying library is GPL. Avoid. |
| `gvnc`, `gvnc-sys` | 0.7.0 (2026-03-26) | n/a | FFI to gtk-vnc | **Unverified:** gtk-vnc license (believed LGPL); not evaluated. |

None of these covers RA2/RSA-AES, VeNCrypt/TLS, ExtendedDesktopSize-heavy multi-monitor or H.264 at the quality we need on both sides. We should write our own sans-I/O RFB crate, mirroring IronRDP's design, and borrow from vnc-rs and rustvncserver where useful. Both licenses allow this.

## Gateways, frontends and others

- **Apache Guacamole:**
  - Apache-2.0. 1.6.0 was released 2025-06-22 ([releases](https://guacamole.apache.org/releases/)); `main` is already 1.6.1 in [`configure.ac`](https://github.com/apache/guacamole-server/blob/main/configure.ac).
  - 1.6.0 added FreeRDP 3 support ([notes](https://guacamole.apache.org/releases/1.6.0/)), and configure probes both `freerdp2` and `freerdp3`. VNC uses `libvncclient`.
  - It translates RDP/VNC/SSH/telnet/Kubernetes into its own browser protocol.
  - For us: a reference for a protocol-agnostic gateway architecture. Distributed guacd binaries link GPL libvncclient.
- **RustDesk:** AGPL-3.0, Rust, 1.5.1 (2026-11-01). It uses its own rendezvous/relay protocol, not RDP or VNC. Its changelog mentions an mstsc-related feature (`feat(rdp)` #15781), probably TCP tunnelling to RDP. **Unverified:** what that feature does. Do not reuse (AGPL).
- **Remmina:**
  - GPL-2.0-or-later with an OpenSSL exception ([COPYING](https://gitlab.com/Remmina/Remmina/-/blob/master/COPYING)). The latest tag is v1.4.43 (2026-02-20); the GitHub mirror lags at v1.4.41.
  - The RDP plugin builds against FreeRDP 2 or 3 (`WITH_FREERDP3`), and the VNC plugin links libvncclient.
  - The project [asked for new maintainers](https://remmina.org/) in 2022.
  - For us: a Linux client test peer.
- **MeshCentral:** Apache-2.0, 1.3.1 (2026-10-23). Its [`rdp/`](https://github.com/Ylianst/MeshCentral/tree/master/rdp) directory is a **GPL-3.0** fork of node-rdpjs, with NLA ported from rdp-rs. The project says deleting that folder removes RDP support. This is a good example of the license hygiene we need. No reuse.
- **rdp-rs** (citronneur): MIT, v0.1.0 (2020-04-11), dormant. A historical pure-Rust client; superseded by IronRDP.
- **Myrtille:** Apache-2.0, C#/.NET HTML5 RDP/SSH gateway on IIS, using a FreeRDP fork ([README](https://github.com/myrtille-rdp/myrtille)). It bundles some AGPL/GPL components for printing. The latest is v2.9.2. **Unverified:** the year of that release. No reuse.
- **aardwolf** (skelsec): MIT, Python, 0.2.17 (2026-11-07). A headless async RDP and VNC client with CredSSP NTLM/Kerberos, pass-the-hash and Restricted Admin, and a scanner ([README](https://github.com/skelsec/aardwolf)). Useful as a scriptable negative-path test client for our server.
- **PyRDP** (GoSecure): GPL-3.0, Python, v2.1.0 (2024-01-23). An RDP MITM and player ([README](https://github.com/GoSecure/pyrdp)). Use it in security tests, for example to check that our client refuses downgrade or MITM without cert pinning. Run it as an external tool only.
- **Other IronRDP-based servers** worth watching: [Lamco RDP Server](https://lamco.ai/products/lamco-rdp-server/), [qemu-rdp](https://gitlab.com/marcandre.lureau/qemu-display) and [MacRDP](https://github.com/clintcan/macrdp). They show `ironrdp-server` is usable in production-like settings.

## Licensing compatibility cheat sheet (for an Apache-2.0 project)

| Upstream license | Depend on / link (distributed binary stays Apache-2.0) | Copy code into our repo | Use as separate process / test peer | Examples |
|---|---|---|---|---|
| MIT, BSD-2/3-Clause, ISC, X11-style, Apache-2.0 | Yes (keep notices; Apache needs NOTICE propagation) | Yes, with attribution | Yes | IronRDP, sspi-rs, picky, FreeRDP, xrdp, neatvnc, wayvnc, Weston, vnc-rs, rustvncserver |
| MIT OR Apache-2.0 (dual) | Yes, choose Apache-2.0 | Yes | Yes | IronRDP, sspi-rs, vnc-rs |
| MPL-2.0 | Yes (file-level copyleft: modified MPL files stay MPL and their source must be offered) | Only as whole files kept under MPL; never relicensed | Yes | noVNC core, Oxide `rfb` |
| LGPL-2.1 / LGPL-3.0 | Dynamic linking OK if users can replace the library. Static linking (typical in Rust) is awkward: you must enable relinking | No | Yes | KRDP library parts, possibly gtk-vnc |
| GPL-2.0-only | **No.** Incompatible with Apache-2.0 per the FSF | No | Yes (network peer, not linked or shipped together) | x11vnc (if -only), TightVNC 2.x |
| GPL-2.0-or-later | Only if the whole product becomes GPL-3.0 (combined work), which defeats Apache-2.0 | No | Yes | TigerVNC, libvncserver/client, GNOME Remote Desktop, Remmina |
| GPL-3.0(-or-later) | One-way: Apache-2.0 code can go *into* GPL-3.0 works, not the reverse. Linking makes our distribution GPL-3.0 | No | Yes | UltraVNC, rdesktop, PyRDP, MeshCentral `rdp/` |
| AGPL-3.0 | No (plus network-use clause) | No | Yes, but avoid even hosting it in our service | RustDesk |
| Proprietary | No | No | Yes, if licensed (Windows VMs, RealVNC) | mstsc, Windows App, RealVNC, TightVNC 4 |

Two notes. Reading GPL code for understanding is legal, but transliterating it is a derivative work. For GPL peers, work from the specs (MS-* Open Specifications, RFC 6143, rfbproto) and black-box testing. H.264 also carries patent concerns separate from copyright: prefer runtime-loaded Cisco OpenH264 or OS/hardware encoders, as IronRDP's `openh264-libloading` and GNOME's VA-API/NVENC do.

## Reuse strategy

1. **RDP: depend on IronRDP; don't fork, don't rewrite.**
   - Use `ironrdp-pdu`, `-connector`, `-session`, `-acceptor`, `-server`, `-egfx`, `-cliprdr`, `-rdpsnd`, `-rdpdr`, `-dvc`, `-displaycontrol`, `-graphics`, `-mstsgu`, `-rdpeudp` and `-tokio` behind our backend trait.
   - Its sans-I/O state machines map directly onto a protocol-agnostic core.
   - Pin exact versions, because the crates are pre-1.0 with frequent minor bumps.
   - Upstream fixes and missing features instead of forking. Candidate gaps to verify: AVC444 encode, server-side progressive/ClearCodec, RDP-UDP integration in connector/acceptor, and multi-monitor GFX on the server.
   - Keep a thin adapter layer so a temporary vendored patch (`[patch.crates-io]`) is possible if upstream is slow.
2. **Authentication: sspi-rs and picky** (transitively via IronRDP, directly for server-side NLA/Kerberos and gateway auth).
3. **FreeRDP: reference, port source and interop peer.** Because it is Apache-2.0, we may port specific algorithms, such as AV1/AVC444 handling, gateway quirks and shadow-server capture tricks. Record provenance in NOTICE and check per-file headers first. Do not link it in the core product; this keeps us memory-safe and free of a large C dependency. An optional `freerdp-ffi` test harness is acceptable.
4. **VNC: write our own `rfb` crates.** Design them sans-I/O like IronRDP, with separate client and server state machines and a shared encodings crate.
   - Seed from vnc-rs (client decoders, WASM friendliness) and rustvncserver/rfb-encodings (server encoders), keeping their notices.
   - Implement RA2/RSA-AES, VeNCrypt (rustls), ExtendedDesktopSize, ExtendedClipboard and H.264 ourselves from the [rfbproto spec](https://github.com/rfbproto/rfbproto/blob/master/rfbproto.rst).
   - neatvnc (ISC) is the best architectural and encoder reference.
   - Never link or copy from libvncserver, TigerVNC, UltraVNC or TightVNC.
5. **Web client:** evaluate IronRDP's protocol-agnostic `iron-remote-desktop` web component as the shell for both RDP and VNC WASM backends. noVNC can serve as an optional, unmodified, MPL-2.0 VNC fallback.
6. **Interop test matrix (CI and lab):**
   - Our RDP client against: Windows Server/11 RDS (mstsc behaviour as oracle), xrdp, FreeRDP shadow/sample server, GNOME Remote Desktop, KRDP, and Weston `--backend=rdp`.
   - Our RDP server tested with: mstsc, Windows App (macOS/iOS/Android), FreeRDP `xfreerdp`/`sdl-freerdp`, Remmina, IronRDP viewer, aardwolf, and Guacamole.
   - Our VNC client against: TigerVNC Xvnc, x11vnc, libvncserver examples, wayvnc/neatvnc, UltraVNC (Windows), TightVNC, the macOS Screen Sharing server, and RealVNC (licensed).
   - Our VNC server tested with: TigerVNC viewer, noVNC, Remmina, UltraVNC viewer, and the macOS Screen Sharing client.
   - Security: PyRDP MITM scenarios, plus fuzzing harnesses modelled on IronRDP's `ironrdp-fuzzing`.

   All GPL and proprietary peers run as separate containers or VMs. They are never vendored into our repo or release artifacts.

## Recommendations for universal-rdp-oss

1. Adopt **IronRDP + sspi-rs + picky** (MIT OR Apache-2.0) as the RDP backend for both client and server. Contribute upstream instead of forking.
2. Treat **FreeRDP (Apache-2.0) as our main open reference and interop peer**. Code porting is legally fine with attribution, but avoid linking it into the shipping product.
3. Build a **native Rust sans-I/O RFB implementation**. Optionally seed it from vnc-rs (client) and rustvncserver (server), both permissive, and use neatvnc (ISC) as the encoder reference. Exclude all GPL VNC code.
4. Keep a **license gate in CI** (e.g. `cargo deny` with an allowlist of MIT/BSD/ISC/Apache-2.0/Zlib/Unicode-3.0, MPL-2.0 only for unmodified web assets). This prevents an accident like MeshCentral's GPL `rdp/` folder or Guacamole's GPL libvncclient linkage.
5. Ship H.264 via **runtime-loaded OpenH264 or OS/hardware encoders** to separate patent licensing from our distribution.
6. Budget for **Windows lab infrastructure**: Windows Server RDS and Windows 11 hosts, plus Windows App on macOS/iOS/Android. Microsoft's implementations remain the conformance oracle for the RDP features that matter most: NLA, GFX/AVC, UDP and gateway.

## Sources

Primary sources consulted (2026-11-07):

- GitHub REST API repo metadata and `releases/latest` for all GitHub-hosted projects above (`https://api.github.com/repos/<owner>/<repo>`).
- FreeRDP: <https://github.com/FreeRDP/FreeRDP>, [ChangeLog](https://github.com/FreeRDP/FreeRDP/blob/master/ChangeLog), [LICENSE](https://github.com/FreeRDP/FreeRDP/blob/master/LICENSE), tag `2.11.8` on branch `stable-2.0`.
- IronRDP: <https://github.com/Devolutions/IronRDP>, [README](https://github.com/Devolutions/IronRDP/blob/master/README.md), [Cargo.toml](https://github.com/Devolutions/IronRDP/blob/master/Cargo.toml), [ironrdp-server](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-server), [ironrdp-egfx](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-egfx), [ironrdp-rdpeudp](https://github.com/Devolutions/IronRDP/tree/master/crates/ironrdp-rdpeudp), <https://crates.io/crates/ironrdp>.
- sspi-rs: <https://github.com/Devolutions/sspi-rs>, <https://crates.io/crates/sspi>. picky-rs: <https://github.com/Devolutions/picky-rs>, <https://crates.io/crates/picky>.
- xrdp: <https://github.com/neutrinolabs/xrdp>, [NEWS v0.10.6.1](https://github.com/neutrinolabs/xrdp/blob/v0.10.6.1/NEWS.md), [xrdp_iso.c](https://github.com/neutrinolabs/xrdp/blob/devel/libxrdp/xrdp_iso.c). xorgxrdp: <https://github.com/neutrinolabs/xorgxrdp>.
- rdesktop: <https://github.com/rdesktop/rdesktop>.
- Microsoft: <https://learn.microsoft.com/en-us/windows-app/overview>, <https://learn.microsoft.com/en-us/windows-app/whats-new>, <https://learn.microsoft.com/en-us/previous-versions/remote-desktop-client/whats-new-windows>, <https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/connect-remote-desktop-services>, <https://learn.microsoft.com/en-us/openspecs/windows_protocols/>.
- GNOME Remote Desktop: <https://gitlab.gnome.org/GNOME/gnome-remote-desktop> (clone: COPYING, meson.build, NEWS).
- KRDP: <https://invent.kde.org/plasma/krdp> (clone: LICENSES/, CMakeLists.txt, SPDX headers).
- Weston: <https://gitlab.freedesktop.org/wayland/weston> (clone: COPYING, libweston/backend-rdp/meson.build), <https://wayland.freedesktop.org/releases.html>.
- TigerVNC: <https://github.com/TigerVNC/tigervnc> (README.rst, LICENCE.TXT, source headers).
- TightVNC: <https://www.tightvnc.com/download.php>, <https://www.tightvnc.com/licensing.php>, <https://www.tightvnc.com/whatsnew.php>.
- UltraVNC: <https://github.com/ultravnc/UltraVNC>, <https://uvnc.com/>, <https://uvnc.com/downloads/ultravnc/175-ultravnc-1-8-3-2.html>.
- RFB spec: <https://github.com/rfbproto/rfbproto>; RFC 6143 <https://www.rfc-editor.org/rfc/rfc6143.html>.
- noVNC: <https://github.com/novnc/noVNC> (README, LICENSE.txt).
- LibVNC: <https://github.com/LibVNC/libvncserver>, <https://github.com/LibVNC/x11vnc>.
- neatvnc / wayvnc: <https://github.com/any1/neatvnc>, <https://github.com/any1/wayvnc>.
- Rust VNC crates (crates.io API and crate tarballs): <https://crates.io/crates/vnc-rs>, <https://crates.io/crates/rustvncserver>, <https://crates.io/crates/rfb-encodings>, <https://crates.io/crates/rfb>, <https://crates.io/crates/vnc>, <https://crates.io/crates/libvnc>, <https://crates.io/crates/vncserver>, <https://crates.io/crates/gvnc>, <https://crates.io/crates/freerdp-sys>, <https://crates.io/crates/freerdp2>, <https://crates.io/crates/rdp-rs>.
- Guacamole: <https://guacamole.apache.org/releases/>, <https://guacamole.apache.org/releases/1.6.0/>, <https://github.com/apache/guacamole-server> (configure.ac).
- RustDesk: <https://github.com/rustdesk/rustdesk>, <https://github.com/rustdesk/rustdesk/releases>.
- Remmina: <https://gitlab.com/Remmina/Remmina> (clone: COPYING, plugins/rdp and plugins/vnc CMakeLists), <https://remmina.org/>.
- MeshCentral: <https://github.com/Ylianst/MeshCentral> (rdp/README.md, rdp/LICENSE).
- rdp-rs: <https://github.com/citronneur/rdp-rs>. Myrtille: <https://github.com/myrtille-rdp/myrtille>. aardwolf: <https://github.com/skelsec/aardwolf>. PyRDP: <https://github.com/GoSecure/pyrdp>.
