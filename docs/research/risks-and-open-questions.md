# Risks and open questions

Research survey for **universal-rdp-oss**: an Apache-2.0 RDP and VNC/RFB client and server. Compiled 2026-11-07.

> **Not legal advice.** This document summarises public sources so the project can spot issues early. Any licensing, patent or trademark decision should be checked with qualified counsel before release.

## Summary

- **The RDP specifications are not on Microsoft's Open Specification Promise (OSP) list.** The MS-RDP\* documents are covered by copyright permission to implement. Patent rights come through Microsoft's licensing programs, which include a non-assert pledge for open-source developers. That pledge does not cover *commercial distribution*, so downstream vendors who ship our code commercially may need their own licence.
- **H.264 (AVC420/AVC444) is the main licensing exposure.** The safest default is to use the decoders and encoders the OS provides. A pure-Rust or other source-built H.264 codec should be an opt-in build feature, not a default.
- **On the server side, every OS has an "unattended access" problem.** Windows needs a SYSTEM service plus a per-session agent. Wayland portals need a user dialog at least once. macOS has TCC prompts, and the persistent entitlement is restricted.
- **RDP parsers are a proven attack surface.** FreeRDP alone has roughly 325 published GitHub advisories. Memory safety (Rust), fuzzing from day one and secure defaults are the main mitigations.

## Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | Microsoft patent claims on RDP extensions. RDP is not under OSP, and the open-source pledge excludes commercial distribution. | Low (no known action against FreeRDP/xrdp in ~20 years) | High for commercial redistributors | Document the licensing position clearly. Tell commercial redistributors to contact Microsoft's IP licensing team (`iplg@microsoft.com` per the spec notices, `protocol@microsoft.com` per the MIP page). Implement only from published specs. |
| R2 | H.264 patent royalties (Via LA pool, plus non-pool holders) | Medium for distributors of binaries with a built-in encoder | Medium–High | Use OS codecs by default (Media Foundation, VideoToolbox, VA-API/V4L2 where the distro ships them). Offer Cisco OpenH264 only as a runtime download. Make any other software H.264 an opt-in Cargo feature. Keep RemoteFX, Progressive, ClearCodec and Planar as royalty-free fallbacks. |
| R3 | Memory-safety and logic bugs in parsers/codecs (client-side "reverse RDP" attacks, server RCE à la BlueKeep) | High (historically certain) | Critical | Rust with `#![forbid(unsafe_code)]` in protocol crates, continuous fuzzing (cargo-fuzz, OSS-Fuzz), size and allocation caps, `cargo-deny`/`cargo-audit`, and a published security policy. |
| R4 | Weak or legacy security modes enabled (Standard RDP Security/RC4, VNC DES auth, NTLM-only) | Medium | High | Secure defaults: TLS+NLA only. Legacy modes are client-side, opt-in, behind explicit flags with warnings, and never offered by our server by default. |
| R5 | Server capture/input blocked by platform security (secure desktop, Wayland portals, macOS TCC) | High | High (product unusable for unattended access) | Per-OS architecture work early: a Windows service plus session agent, portal restore tokens plus Mutter/KWin direct APIs, and a macOS entitlement application. Document limitations honestly. |
| R6 | Scope creep. There are dozens of MS-RDP\* specs, plus RD Gateway, UDP, AVD/Entra. | High | High (never ships) | Define an MVP profile (BCGR + CredSSP + EGFX/AVC420 + clipboard). Track the rest as tiered milestones. Keep the core protocol-agnostic. |
| R7 | AVD/Entra/Windows 365 features that are undocumented or service-gated | High | Medium | Treat AVD as a later milestone. Implement from public docs first, and use 17 U.S.C. §1201(f)-style clean interop analysis only where needed. |
| R8 | Interop testing needs many Windows versions, RDS roles, Gateway, AVD | High | Medium | Use evaluation VHDs, MSDN/Visual Studio subscriptions, Azure credits, and Microsoft plugfests. Keep a recorded-trace (pcap/PDU) regression corpus. |
| R9 | Maintainer bandwidth and bus factor | High | High | Narrow the MVP, write good contributor docs, and coordinate with IronRDP and FreeRDP instead of competing on every front. |
| R10 | Software H.264 encode latency/CPU on servers without a GPU | Medium | Medium | Hardware encoders first. Use RemoteFX/Progressive or planar for text-heavy content. Do damage-based encoding. Ship benchmarks in CI. |
| R11 | Trademark conflicts ("RDP", "Remote Desktop", "VNC") | Medium | Medium (forced rename) | Avoid marks in the project name. Use referential phrasing ("compatible with Microsoft Remote Desktop Protocol"). |
| R12 | Supply chain (crates, codec binaries, CI) | Medium | High | Lockfiles, `cargo-vet`, reproducible builds, signed releases, minimal C dependencies. Verify OpenH264 download hashes. |
| R13 | Accessibility gaps in client UI, and in remote screen-reader use | Medium | Medium | Choose a GUI toolkit with OS accessibility-API bridges. Plan audio redirection early, since remote screen readers depend on it. |

## 1. Legal and patent

### 1.1 Microsoft Open Specification Promise and the RDP specs

- The [Open Specification Promise](https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/1c24c7c8-28b0-4ce1-a47d-95fe1ff504bc) (revised 2023-02-24) is a non-assert covering only the specifications listed on that page. It applies to the "identified version", and new versions must be "separately considered". **No MS-RDP\*, MS-CSSP, MS-TSGU or MS-RDPEGFX document appears on that list.** The protocols it does list are things like TDS, SMB/CIFS drafts, Kerberos RFCs, SPNEGO and TLS/SSL.
- The [Microsoft Community Promise](https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/8b8d1b7a-a10a-4667-9558-6d9c43adf60d) covers several ITU standards that RDP's lower layers use:
  - T.123
  - T.124 (GCC)
  - T.128
  - "ITU X.224 (implemented in RDP)"

  It does not cover MS-RDPBCGR itself.
- Every MS-RDP\* page carries the same IP notice (e.g. [MS-RDPBCGR, rev. 62.0, 2026-03-09](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/5073f4ed-1e93-45e1-b039-6e30c385867c)):
  - **Copyright:** you may copy the docs to develop implementations, and distribute schemas, IDLs and samples.
  - **Trade secrets:** Microsoft claims no trade-secret rights.
  - **Patents:** "Microsoft has patents that might cover your implementations… Neither this notice nor Microsoft's delivery of this documentation grants any licenses under those patents." Licences are available via `iplg@microsoft.com`, and the [Patent and Program Map](https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/13571077-e344-4e6f-a477-369894979798) lists the program and patents per document.
- Windows protocols sit in the [Microsoft Interoperability Program (MIP)](https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/a83d8d17-0cde-46df-a496-2686bff58d5d) (which descends from the 2009 EU Interoperability Undertaking) and the [Interoperability Principles Program](https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/d84cac00-b312-44ee-9156-23bde6477c3d). Both include a **Patent Pledge for Open Source Developers**:
  - It is an irrevocable non-assert for "making, using, importing, or distributing" a Covered Implementation.
  - It covers anyone "participating in the creation of software code for an open source project".
  - It **excludes "commercial distribution"**, defined as deriving revenue "from subscriptions, updates, or user-based connection fees or from services that are contractually required".
  - Commercial users can take a MIP patent licence or covenant (e.g. the Windows Server / Windows Client Protocols agreements). Microsoft promises RAND, "low royalty rates".
- **What this means for us:**
  - Developing and distributing universal-rdp-oss non-commercially is inside the pledge.
  - Companies that sell products embedding it (appliances, paid SaaS, paid support contracts that bundle updates) are outside it and must assess MIP licensing themselves.
  - The README and NOTICE should say this plainly.
  - **Unverified:** which specific US patents the Patent and Program Map assigns to MS-RDPEGFX, MS-RDPEUDP, MS-RDPRFX, etc. The map is an interactive widget we could not query automatically. Someone should check it by hand and record the results.
- Apache-2.0 §3 grants a patent licence only from *contributors*, covering their own contributions. It gives no protection against Microsoft's or third-party patents. Its defensive-termination clause, plus the OSP/CP clauses that terminate if you sue Microsoft over the same spec, are worth noting in the governance docs.

### 1.2 Trademarks

- RealVNC states that ["RealVNC, VNC and RFB are trademarks of RealVNC Limited"](https://www.realvnc.com/en/?p=89077) with registrations in the EU, US and elsewhere. A third-party USPTO mirror lists ["VNC" as a live US registration (3641372)](https://markinton.com/trademark/vnc-77198888). **Unverified:** current USPTO/EUIPO status of "RFB", which one directory shows as cancelled. Many projects use "VNC" descriptively (TigerVNC, TurboVNC, wayvnc), but **don't put "VNC" in our product name**.
- Microsoft's [trademark guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks) forbid Microsoft marks in product or app names without a licence. They allow truthful compatibility statements ("works with…") and require that we not imply endorsement. "RemoteFX" is listed as a Microsoft trademark in [third-party trademark notices](https://support.kaspersky.co.uk/help/KTC/2.3/en-US/186467.htm). **Unverified:** whether "Remote Desktop" or "RDP" is a registered Microsoft mark. Neither appeared in what we could retrieve of the trademark list. Still, "Remote Desktop" is used as a Microsoft product name, so a name like "universal-rdp" carries some risk. A neutral project name with a "speaks RDP and VNC" tagline is lower risk.

### 1.3 Reverse engineering and DMCA

- For behaviour that isn't documented (AVD/Entra flows, the HEVC path in AVD, undocumented server quirks), the US DMCA [17 U.S.C. §1201(f)](https://www.law.cornell.edu/uscode/text/17/1201) allows circumvention and analysis "solely for the purpose of" interoperability of an independently created program. Results may be shared only for that purpose. The EU Software Directive (2009/24/EC, Art. 6) has a comparable decompilation exception.
- Policy proposal:
  - Prefer the published specs and observing traffic on the wire, using our own accounts and test tenants.
  - Never copy Microsoft binaries or code.
  - Keep notes on where each undocumented behaviour came from.
  - Be careful about service Terms of Use (AVD/Entra/Windows 365 ToS) even where DMCA allows the analysis.

## 2. Codec licensing

### 2.1 What RDP actually needs

[MS-RDPEGFX](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpegfx/fb919fce-cc97-4d2b-8cf5-a737a00ef1a6) codec IDs:

| Codec | ID |
|---|---|
| Uncompressed | 0x0 |
| CAVIDEO/RemoteFX | 0x3 |
| ClearCodec | 0x8 |
| Planar | 0xA |
| AVC420 | 0xB |
| Alpha | 0xC |
| AVC444 | 0xE |
| AVC444v2 | 0xF |

The Progressive codec is negotiated through capability flags. **The published spec has no HEVC or AV1 codec ID.** Yet Azure Virtual Desktop documents [HEVC/H.265 GPU frame encoding](https://learn.microsoft.com/en-us/azure/virtual-desktop/graphics-encoding), which requires Windows App ≥ 1.3.278 and the client HEVC extension. That is a proprietary or unpublished extension. **Unverified:** whether a preview MS-RDPEGFX revision documents it. No public source indicates AV1 in RDP.

RemoteFX, Progressive, ClearCodec and Planar are Microsoft-specified codecs under the same IP notice and MIP pledge as the rest of RDP (§1.1). They are *not* OSP-covered, but they need no third-party pool licence. They make a viable royalty-free fallback.

### 2.2 H.264/AVC pool terms (Via Licensing Alliance)

- MPEG LA's AVC pool now sits with Via LA. Section 1 terms for [end-product encoders/decoders](https://www.via-la.com/licensing-2/avc-h-264/):
  - **0–100,000 units/year: free.**
  - **$0.20/unit up to 5M, then $0.10/unit.**
  - **Enterprise cap $9.75M/yr (2017 onward).**
- The page was updated 2026-09-28 with a new active patent list.
- In 2026 Via LA replaced the flat $100k *streaming* cap with tiers up to $4.5M for new licensees ([Streaming Media](https://www.streamingmedia.com/Articles/News/Online-Video-News/H.264-Streaming-Fees-What-Changed-Whos-Affected-and-What-It-Means-173935.aspx)). That affects content services, not us.
- Open-source source code isn't a "unit". **Distributors of compiled binaries are the ones who count units**, and that includes us if we publish release binaries with a built-in encoder or decoder. A single entity's 100k free tier may cover early releases, but it needs tracking and a licence agreement. Non-pool holders (e.g. Nokia, which [sued HP and Amazon in 2023](https://meta.wikimedia.org/wiki/Have_the_patents_for_H.264_MPEG-4_AVC_expired_yet%3F)) are outside the pool entirely.

### 2.3 Patent expiry

[Wikimedia's tracker](https://meta.wikimedia.org/wiki/Have_the_patents_for_H.264_MPEG-4_AVC_expired_yet%3F) (edited 2026-09-16) reports that EP, JP, CN and CA pool patents have expired (EP Jan 2025, JP/CN Aug 2025). In the US, several pool patents remain until 2027, and US 9356620 runs to **2030-11-26**. Brazil runs to **2030-11-10**. It's unclear which late continuations are essential for Constrained Baseline/Main versus High profile.

**Practical horizon: H.264 is effectively unencumbered outside the US and Brazil now, and everywhere around end-2030.** **Unverified:** essentiality of the late US patents to the profiles RDP uses. RDP AVC420 commonly uses Main/High-like tools, but we did not verify which profiles Windows emits.

### 2.4 Implementation options

| Option | Copyright licence | Patent position | Notes |
|---|---|---|---|
| **OS codecs** (Media Foundation on Windows, VideoToolbox on macOS, VA-API/V4L2/NVENC/AMF on Linux) | No copyleft issue | The OS/GPU vendor ships the codec; we only call it | Best default. Caveat: Fedora/openSUSE Mesa [disable H.264/HEVC VA-API](https://itsfoss.com/news/fedora-drops-vaapi-codec/) for patent reasons, still true in [2025](https://github.com/LizardByte/Sunshine/issues/4464). Windows "N" editions lack Media Foundation codecs without the Media Feature Pack. |
| **Cisco OpenH264 binary** (runtime download) | BSD-2 | Cisco pays royalties only for [its own binary](https://www.openh264.org/faq.html). It must be ["separately downloaded… not integrated into or combined with third party software"](https://www.openh264.org/BINARY_LICENSE.txt), the user must be able to disable it, and the app must show "OpenH264 Video Codec provided by Cisco Systems, Inc." | Constrained Baseline only, so encode quality is limited. It also carries security and ABI risk: CVE-2025-27091, a heap overflow in ≤2.5.0 ([Red Hat](https://access.redhat.com/security/cve/CVE-2025-27091)), was fixed in 2.6.0, but that release broke the ABI. This led [Freedesktop SDK to retire OpenH264 (Mar 2025)](https://linuxiac.com/freedesktop-sdk-retires-cisco-openh264-codec-library/), and distros cannot patch the Cisco blob. Downloads are also [geoblocked in some countries](https://discussion.fedoraproject.org/t/ciscobinary-openh264-org-is-unreachable-in-some-countries-ru-ua-ir/161434?page=3). |
| **OpenH264 built from source** | BSD-2 | We or the distributor [pay the royalties](https://www.openh264.org/faq.html) | Same as any self-built codec. |
| **x264** | GPL-2.0+ (or [commercial licence](https://videolan.org/developers/x264.php)) | No patent licence | GPL is incompatible with distributing as Apache-2.0-only binaries, so it can only be an optional plugin. |
| **FFmpeg/libavcodec** | [LGPL-2.1+ by default; GPL if `--enable-gpl`/libx264](https://ffmpeg.org/legal.html) | FFmpeg grants no patent licence | Dynamically link an LGPL build. It's a big C attack surface for a decoder. |
| **Pure-Rust H.264 decoder** | Our choice | Same patent position as any implementation | Gives memory safety, but writing and verifying an AVC444 decoder is a large job. |

**Conclusion:**
- Default builds should use OS/hardware codecs and fall back to RemoteFX/Progressive/Planar.
- Offer Cisco OpenH264 as a user-consented runtime download for software H.264 where no OS codec exists. Pin and verify the hashes, and accept the Baseline-only limitation.
- Source-built software H.264 (OpenH264, FFmpeg, x264, Rust) should be a build-time feature, off in official binaries until the end-2030 horizon or legal review says otherwise.
- VNC needs no patented codec. Tight/ZRLE/JPEG are fine. Open H.264 in RFB (encoding 50, Tight/TigerVNC) inherits the same choices.

## 3. Security

### 3.1 CredSSP / NLA

- **CVE-2018-0886** (CredSSP encryption oracle) was fixed by adding a nonce/hash binding in TSRequest v5/v6. Microsoft's [Encryption Oracle Remediation](https://support.microsoft.com/en-us/help/4093492) policy controls fallback; "Force Updated Clients" refuses old versions, and the default became "Mitigated" in May 2018.
- **Our CredSSP must:**
  - implement TSRequest v6 with `clientNonce` and the pubKeyAuth hash binding ([MS-CSSP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/));
  - refuse to downgrade below v5 by default;
  - bind to the TLS server public key.
- **NTLM is being phased out:**
  - NTLMv1 was [removed in Windows 11 24H2 / Server 2025](https://support.microsoft.com/en-us/topic/upcoming-changes-to-ntlmv1-in-windows-11-version-24h2-and-windows-server-2025-c0554217-cdbc-420f-b47c-e02b2db49b2e). Starting with the October 13, 2026 update, `BlockNtlmv1SSO` defaults to Enforce.
  - Microsoft's [three-phase roadmap](https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526) (Jan 2026, updated Oct 2026): enhanced auditing now; IAKerb and a local KDC in H2 2026; **network NTLM disabled by default in the next major Windows Server and client release**. NTLM stays re-enableable via policy.
  - **Implication:** Kerberos via SPNEGO is mandatory for a credible client. That means IAKerb plus a KDC proxy ([MS-KKDCP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kkdcp/)) for clients that can't reach a DC, especially through RD Gateway. NTLMv2 remains a fallback for workgroup hosts, but must not be the only path.
- **Remote Credential Guard and Restricted Admin:** per [Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard), these keep credentials off the target. Remote Credential Guard is **Kerberos-only** (no NTLM fallback), AD-joined targets only, and not via Gateway or Broker. Supporting them ([MS-RDPEAR](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpear/)) is a security differentiator for admin users. Restricted Admin is the recommended mode for helpdesk.
- **Server identity:**
  - Windows hosts default to self-signed TLS certificates. The client needs trust-on-first-use with a pinned SPKI fingerprint stored per host:port.
  - It should give a clear, loud warning on change and honour CA validation when a real cert is present.
  - Gateway connections need HTTPS validation through the normal WebPKI.
  - Never silently accept certificates.

### 3.2 Lessons from past RDP vulnerabilities

- **Server-side wormable bugs:** [BlueKeep (CVE-2019-0708)](https://msrc.microsoft.com/blog/2019/11/rdpvulnerabilities) and [DejaBlue (CVE-2019-1181/1182)](https://msrc.microsoft.com/blog/2019/08/patch-new-wormable-vulnerabilities-in-remote-desktop-services-cve-2019-1181-1182). Both were pre-auth. NLA reduced exposure to authenticated attackers only ([Talos](https://blog.talosintelligence.com/the-latest-on-bluekeep-and-dejablue/)). **Lesson:** our server should parse as little as possible before CredSSP completes, and should require NLA by default.
- **Client-side attacks ("Reverse RDP"):** [Check Point (2019)](https://research.checkpoint.com/2019/reverse-rdp-the-hyper-v-connection/) found ~25 bugs across rdesktop and FreeRDP, plus a clipboard path-traversal in mstsc ([CVE-2019-0887](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2019-0887)). A malicious server could drop files anywhere via file-clipboard redirection.
  - **Lesson:** treat the server as hostile.
  - Sanitize clipboard file names (reject `..`, absolute paths, ADS, device names).
  - Make drive, printer, smartcard and USB redirection opt-in, per connection.
  - Cap every length and allocation.
- **FreeRDP's advisory history** shows the scale of the problem. Its [GitHub security advisories](https://github.com/FreeRDP/FreeRDP/security/advisories) number ~325. A new batch came on 2026-10-19/20 (DRDYNVC uncapped allocation, an RDPGFX CREATE_SURFACE integer wrap, RemoteFX assertion aborts, an incomplete certificate-identity fix). Earlier examples include [CVE-2024-32041](https://cvefeed.io/vuln/detail/CVE-2024-32041) (OOB read in `zgfx_decompress_segment`, fixed in 3.5.0/2.11.6).
  - **Lessons:** codecs and dynamic channels (GFX, RFX, ZGFX, DRDYNVC, RDPDR, audio) are the hot spots. Rust removes the OOB class but not panics or DoS. Treat `panic`, assertion aborts and unbounded allocation as security bugs, and fuzz every PDU decoder.
- **Codec libraries:** CVE-2025-27091 (OpenH264) shows that third-party decoders on the server→client path are an RCE surface too. Sandbox or isolate decoders where feasible.

### 3.3 Legacy modes

- **Standard RDP Security** ([PROTOCOL_RDP, RC4 40/56/128-bit](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/3e86b68d-3e2e-4433-b486-878875778f4b)) uses RC4, which [RFC 7465](https://www.rfc-editor.org/rfc/rfc7465) bans from TLS. It has no PKI-based server authentication. Proposal:
  - The **server never offers it**.
  - The client may support it **only behind an explicit `--insecure-legacy-rdp-security` opt-in**, for old embedded or XP-era targets, with a persistent UI warning.
- **VNC Authentication** ([RFC 6143 §7.2.2](https://www.rfc-editor.org/rfc/rfc6143.html)): DES challenge-response with the password "truncated to eight characters". The RFC calls it "cryptographically weak". RFB itself provides "no protection against observation of or tampering with the data stream" (§9). Cyble found [>8,000–9,000 VNC servers exposed without any authentication](https://www.csoonline.com/article/3670113/exposed-vnc-threatens-critical-infrastructure-as-attacks-spike.html) in 2022. Proposal:
  - Our VNC server listens on localhost or uses VeNCrypt/TLS only by default.
  - It refuses "None" auth on non-loopback interfaces.
  - It supports VeNCrypt X509 plus plain or system auth.
  - The client warns on unencrypted sessions.

### 3.4 Supply chain and secure defaults

- Deny-by-default redirection.
- No listening on 0.0.0.0 without TLS and auth.
- Rate limiting and lockout on the server.
- Signed releases.
- `cargo-deny`/`cargo-vet`.
- SBOM per release.
- Hash-pinned codec downloads.
- A SECURITY.md with an embargo process, and a CVE numbering arrangement (GitHub CNA).

## 4. Platform capture and input (server side)

### 4.1 Windows

- **Capture:**
  - [Desktop Duplication (DXGI)](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/desktop-dup-api) provides dirty and move rects plus pointer shape, which suits RDP well. But [`DuplicateOutput`](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_2/nf-dxgi1_2-idxgioutput1-duplicateoutput) returns `E_ACCESSDENIED` on the secure desktop: "only an application that runs at LOCAL_SYSTEM can access the secure desktop". It also fails on disconnected sessions and allows only 4 concurrent duplicators per session.
  - [Windows.Graphics.Capture](https://learn.microsoft.com/en-us/windows/apps/develop/media-authoring-processing/screen-capture) is designed around a user picker and a yellow capture border. It's better for app sharing than for headless servers.
- **Input:** [`SendInput`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput) is subject to UIPI. You can only inject into equal or lower integrity, and failures are silent. Ctrl+Alt+Del needs [`SendSAS`](https://learn.microsoft.com/en-us/windows/win32/api/sas/nf-sas-sendsas), which requires running as a service (or uiAccess, signed and in Program Files) plus the "software SAS" group policy.
- **Architecture this forces:**
  - a SYSTEM service in session 0;
  - which launches a per-session agent with a SYSTEM token *into the console session*;
  - and follows desktop switches (Default ↔ Winlogon/UAC).
  - This is how commercial tools do it.
- **Headless/virtual displays:** a [IddCx indirect display driver](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-overview), a user-mode UMDF driver that Microsoft explicitly mentions for "remote display", creates virtual monitors. It requires driver signing (attestation through the Microsoft Hardware Dev Center), which is a cost and process for an OSS project.
- **Coexistence with Windows' own RDP server:**
  - TermService owns 3389 on Pro/Enterprise/Server, so default to a different port or offer to take over.
  - Home editions have no RDP host, which is a selling point for us.
  - Multi-session hosting on client Windows conflicts with the [Windows licence terms](https://www.microsoft.com/en-us/useterms) (single licensed user; remote access from separately licensed devices; remote *assistance* of an active session allowed). Proposal: on Windows, ship console-session sharing only and leave multi-session to RDS.
- **Audio:** [WASAPI loopback](https://learn.microsoft.com/en-us/windows/win32/coreaudio/loopback-recording). By default it captures the mix from all sessions, which is a privacy issue on multi-user hosts. Per-process loopback (Win10 2004+) can scope it.

### 4.2 Linux X11

- XShm/XGetImage for frames.
- XDamage for dirty regions.
- XFixes for cursor image and changes.
- XTest for input injection.
- XRandR for layout.

Mature and unrestricted. Any X client with access to the display can do all of it. That makes it simple (x11vnc, xrdp) but a security non-boundary. For multi-session servers, spawn an Xvfb or Xorg-dummy session per user (the xrdp model).

### 4.3 Linux Wayland

- **Portable path:**
  - xdg-desktop-portal [ScreenCast](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html) delivers frames through PipeWire.
  - [RemoteDesktop](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html) v2 adds input, either through `Notify*` methods or, preferably, `ConnectToEIS` (libei), plus clipboard.
  - `Start()` shows a user dialog. `persist_mode=2` returns a **single-use `restore_token`**. Compositors may ignore it, in which case "the user will be prompted normally".
  - **Unattended access after reboot, or at the login screen, is therefore not guaranteed through portals.**
- **GNOME:**
  - gnome-remote-desktop uses Mutter's private [`org.gnome.Mutter.RemoteDesktop`](https://gitlab.gnome.org/GNOME/mutter/-/raw/main/data/dbus-interfaces/org.gnome.Mutter.RemoteDesktop.xml) D-Bus API, explicitly "private and not intended to be used outside of the integrated system". It includes `ConnectToEIS` and clipboard.
  - GNOME 46+ has a headless, multi-user ["remote login" via GDM](https://github.com/GNOME/gnome-remote-desktop) that uses RDP Server Redirection. g-r-d already *is* an RDP server built on FreeRDP.
  - A third-party server would be competing on GNOME's home turf, and would need either the private API or the portals.
- **KDE:** KWin exposes portals plus its own protocols. KRdp is KDE's RDP server.
- **wlroots/Sway/others:** [`ext-image-copy-capture-v1` + `ext-image-capture-source-v1`](https://phoronix.com/news/Wayland-Merges-Screen-Capture) were merged into wayland-protocols staging in Aug 2024, succeeding `wlr-screencopy-unstable-v1`. Input uses `virtual-keyboard`/`wlr-virtual-pointer` (wayvnc's approach). **Unverified:** a formal deprecation notice for wlr-screencopy. We saw only secondary sources.
- **Conclusion:** implement a portal backend (works everywhere, attended) plus compositor-specific backends (Mutter D-Bus, ext-image-copy-capture) for unattended or headless use. Accept that GNOME/KDE login-screen access belongs to their own integrated servers.
- **Audio:** PipeWire monitor streams.

### 4.4 macOS

- **Capture:**
  - ScreenCaptureKit is the supported API.
  - `CGDisplayCreateImage`/`CGWindowListCreateImage` are [unavailable in the macOS 15 SDK](https://groups.google.com/g/wx-dev/c/iReuuqxtGyk).
  - **Unverified:** the exact status of `CGDisplayStream`, which is reported deprecated but still functional, including at the login window.
- **Input:** CGEvent posting requires the **Accessibility** TCC permission.
- **TCC friction:**
  - Sequoia introduced a [monthly "continue to allow" screen-recording re-prompt](https://9to5mac.com/2024/08/14/macos-sequoia-screen-recording-prompt-monthly/) that applies to ScreenCaptureKit too. macOS 15.1 [reduced the frequency for regularly used apps](https://macrumors.com/2024/10/07/apple-screen-recording-popup-update) but didn't remove it.
  - The fix for VNC-style apps is the restricted [`com.apple.developer.persistent-content-capture`](https://developer.apple.com/forums/thread/770675) entitlement. It requires an application to Apple (with reported multi-month waits), and [MDM cannot pre-grant it (June 2026)](https://developer.apple.com/forums/thread/831795).
  - In macOS 27 betas a separate ["Remote Desktop" TCC category (`kTCCServiceRemoteDesktop`)](https://developer.apple.com/forums/thread/841530) appeared, with no public API to query its state.
  - **Implications:** we need an Apple Developer ID, notarization, and an entitlement application. An unsigned build from source will hit repeated prompts.
- **Login window:** access requires a LaunchDaemon plus a LaunchAgent in the `LoginWindow` session. **Unverified:** whether ScreenCaptureKit works at loginwindow on macOS 15–27. Forum reports suggest vendors fall back to CGDisplayStream there.
- **Audio:** ScreenCaptureKit audio capture (macOS 13+), or CoreAudio process taps (macOS 14.2+).

### 4.5 Client side

Client-side needs are lighter: rendering surfaces, keyboard grabs, IME, clipboard. The pitfalls:

- Wayland keyboard-shortcut inhibit (`zwp_keyboard_shortcuts_inhibit`), needed to pass Alt+Tab and similar shortcuts.
- macOS Secure Input blocking key capture.
- Windows low-level keyboard hooks, needed for Win-key passthrough.
- Smartcard/PCSC and USB redirection permissions.

## 5. Other risks

- **Scope.** Windows Protocols lists dozens of RDP-related documents, including BCGR, EGFX, RFX, EDYC, EUDP/EUDP2, EMT, RDPDR, CLIPRDR, EA, EAI, EI, EGT, ECAM, EUSB, ELE, EPS, TSGU, CSSP, NLMP, KKDCP, RDPEAR and RDSTLS. BCGR alone is at revision 62. An honest MVP is:
  - the BCGR core
  - CredSSP (NTLM+Kerberos)
  - EGFX with Planar/RFX/AVC420
  - CLIPRDR
  - RDPSND
  - DISPLAYCONTROL resize

  RD Gateway, UDP, AVC444, drive redirection and AVD should be later tiers.
- **AVD/Windows 365/Entra.** These need Entra ID (OAuth/PKU2U/RDSAAD "AAD" auth, seen in FreeRDP's advisory list), AVD broker and feed discovery, RDP Shortpath (STUN/TURN), and the undocumented HEVC path. Much of it is service-side and changes without spec revisions. It needs an Azure subscription and paid licences to test.
- **Interop test access.** Required: Windows Server with the RDS roles, the RD Gateway role, current and Insider client builds, plus AVD. Options: evaluation images (180-day), Visual Studio subscriptions, Azure credits for OSS, and Microsoft's [Open Specifications plugfests and test suites](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/5073f4ed-1e93-45e1-b039-6e30c385867c) (linked under "Development Resources").
- **Maintainer bandwidth.** FreeRDP's advisory volume (§3.2) shows the security-response load for a mature implementation. IronRDP (Rust, Devolutions) already covers much of the client and server protocol, and reusing or contributing to it may beat a rewrite. *(See the prior-art survey.)*
- **Performance.** Software H.264 at 1080p60 with low latency is CPU-heavy. RDP's AVC444 needs two AVC420 streams per frame, roughly doubling the work. Encoder latency is dominated by lookahead and B-frame settings, so we need zero-latency, no-B-frame, intra-refresh configurations. This is engineering judgement, not a cited benchmark; validate it with CI benchmarks.
- **Accessibility.**
  - The client UI must expose accessibility trees (UIA, AT-SPI, NSAccessibility). That affects the GUI-toolkit choice: egui/iced have limited or AccessKit-based support, while Qt/GTK/native are mature.
  - Remote sessions are pixels, so blind users depend on audio redirection to hear the remote screen reader.
  - Keyboard-only operation of the connection UI is essential.

## 6. Open questions for the project owner

1. **Server-first or client-first?** The client is better specified and testable against Windows today. The server has the harder platform work (§4) and a bigger security exposure.
2. **Which platforms first?** Suggested: client on Linux, Windows and macOS; server on Linux (X11 + portal/Mutter) and Windows; macOS server last, because of the entitlement wait.
3. **Build on IronRDP (Rust, MIT/Apache-2.0) or write a new stack?** This affects every estimate.
4. **GUI toolkit?** Native per OS, Qt, GTK, Slint, iced or egui. Accessibility and Wayland maturity should weigh heavily.
5. **Codec policy:** do we accept OS codecs as default, offer the OpenH264 runtime download, and keep LGPL FFmpeg or GPL x264 as optional plugins only? Do official binaries contain *any* bundled H.264?
6. **Legacy security:** do we support Standard RDP Security (RC4) at all, even client-only and opt-in? VNC "None"/DES auth on non-loopback?
7. **NTLM:** ship NTLMv2 at all given the deprecation, or Kerberos-first with NTLM opt-in?
8. **Commercial use:** will anyone (including a sponsor) distribute it commercially? If so, who evaluates Microsoft MIP licensing and Via LA AVC licensing?
9. **Project name:** pick a name without "RDP", "Remote Desktop" or "VNC"? Microsoft's guidelines forbid marks in product names, and RealVNC claims "VNC".
10. **Windows server scope:** console sharing only, or also multi-session, despite RDS licensing and Windows EULA constraints? Will we fund IddCx driver signing?
11. **AVD/Windows 365:** in scope for v1, a later tier, or out of scope? Who pays for the test tenant?
12. **Wayland strategy:** portal-only (attended), or also private or compositor-specific APIs (Mutter D-Bus, ext-image-copy-capture) for unattended or headless use?
13. **macOS distribution:** who holds the Apple Developer ID, notarizes releases, and applies for `persistent-content-capture`?
14. **Security process:** a bug bounty or OSS-Fuzz enrolment from the start? A CVE CNA via GitHub?
15. **Governance:** a CLA, or DCO only? (This matters for Apache-2.0 patent-grant clarity and future relicensing.)

## Sources

**Microsoft Open Specifications and legal**
- Open Specification Promise: <https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/1c24c7c8-28b0-4ce1-a47d-95fe1ff504bc>
- Microsoft Community Promise: <https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/8b8d1b7a-a10a-4667-9558-6d9c43adf60d>
- Patent Promises and Patents / Patent and Program Map: <https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/13571077-e344-4e6f-a477-369894979798>
- Microsoft Interoperability Program: <https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/a83d8d17-0cde-46df-a496-2686bff58d5d>
- Interoperability Principles Program: <https://learn.microsoft.com/en-us/openspecs/dev_center/ms-devcentlp/d84cac00-b312-44ee-9156-23bde6477c3d>
- MS-RDPBCGR landing page and IP notice: <https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/5073f4ed-1e93-45e1-b039-6e30c385867c>
- MS-RDPEGFX codec IDs: <https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpegfx/fb919fce-cc97-4d2b-8cf5-a737a00ef1a6>
- MS-RDPBCGR Server Security Data: <https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/3e86b68d-3e2e-4433-b486-878875778f4b>

**Trademarks and interop law**
- Microsoft trademark guidelines: <https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks>
- RealVNC legal: <https://www.realvnc.com/en/?p=89077>; VNC mark mirror: <https://markinton.com/trademark/vnc-77198888>
- 17 U.S.C. §1201: <https://www.law.cornell.edu/uscode/text/17/1201>
- Windows 11 OEM licence terms: <https://www.microsoft.com/en-us/useterms>

**Codecs**
- Via LA AVC/H.264: <https://www.via-la.com/licensing-2/avc-h-264/>
- Streaming Media on 2026 fees: <https://www.streamingmedia.com/Articles/News/Online-Video-News/H.264-Streaming-Fees-What-Changed-Whos-Affected-and-What-It-Means-173935.aspx>
- H.264 patent expiry tracker: <https://meta.wikimedia.org/wiki/Have_the_patents_for_H.264_MPEG-4_AVC_expired_yet%3F>
- OpenH264 FAQ: <https://www.openh264.org/faq.html>; binary licence: <https://www.openh264.org/BINARY_LICENSE.txt>
- CVE-2025-27091: <https://access.redhat.com/security/cve/CVE-2025-27091>; Freedesktop SDK retirement: <https://linuxiac.com/freedesktop-sdk-retires-cisco-openh264-codec-library/>; geoblocking: <https://discussion.fedoraproject.org/t/ciscobinary-openh264-org-is-unreachable-in-some-countries-ru-ua-ir/161434?page=3>
- FFmpeg legal: <https://ffmpeg.org/legal.html>; x264: <https://videolan.org/developers/x264.php>
- Fedora VA-API codec removal: <https://itsfoss.com/news/fedora-drops-vaapi-codec/>; <https://github.com/LizardByte/Sunshine/issues/4464>
- AVD graphics encoding (HEVC): <https://learn.microsoft.com/en-us/azure/virtual-desktop/graphics-encoding>

**Security**
- CredSSP CVE-2018-0886: <https://support.microsoft.com/en-us/help/4093492>
- NTLMv1 changes: <https://support.microsoft.com/en-us/topic/upcoming-changes-to-ntlmv1-in-windows-11-version-24h2-and-windows-server-2025-c0554217-cdbc-420f-b47c-e02b2db49b2e>
- NTLM roadmap: <https://techcommunity.microsoft.com/blog/windows-itpro-blog/advancing-windows-security-disabling-ntlm-by-default/4489526>
- Remote Credential Guard: <https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard>
- MSRC on BlueKeep and RDP vulnerabilities: <https://msrc.microsoft.com/blog/2019/11/rdpvulnerabilities>; DejaBlue: <https://msrc.microsoft.com/blog/2019/08/patch-new-wormable-vulnerabilities-in-remote-desktop-services-cve-2019-1181-1182>; Talos: <https://blog.talosintelligence.com/the-latest-on-bluekeep-and-dejablue/>
- Check Point Reverse RDP: <https://research.checkpoint.com/2019/reverse-rdp-the-hyper-v-connection/>; CVE-2019-0887: <https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2019-0887>
- FreeRDP advisories: <https://github.com/FreeRDP/FreeRDP/security/advisories>; CVE-2024-32041: <https://cvefeed.io/vuln/detail/CVE-2024-32041>
- RFC 6143 (RFB): <https://www.rfc-editor.org/rfc/rfc6143.html>; RFC 7465 (RC4 prohibition): <https://www.rfc-editor.org/rfc/rfc7465>
- Exposed VNC (Cyble via CSO): <https://www.csoonline.com/article/3670113/exposed-vnc-threatens-critical-infrastructure-as-attacks-spike.html>

**Platform APIs**
- Desktop Duplication: <https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/desktop-dup-api>; DuplicateOutput: <https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_2/nf-dxgi1_2-idxgioutput1-duplicateoutput>
- Windows.Graphics.Capture: <https://learn.microsoft.com/en-us/windows/apps/develop/media-authoring-processing/screen-capture>
- SendInput: <https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput>; SendSAS: <https://learn.microsoft.com/en-us/windows/win32/api/sas/nf-sas-sendsas>
- IddCx: <https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-overview>
- WASAPI loopback: <https://learn.microsoft.com/en-us/windows/win32/coreaudio/loopback-recording>
- Portals: ScreenCast <https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html>, RemoteDesktop <https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html>
- Mutter RemoteDesktop D-Bus: <https://gitlab.gnome.org/GNOME/mutter/-/raw/main/data/dbus-interfaces/org.gnome.Mutter.RemoteDesktop.xml>; gnome-remote-desktop: <https://github.com/GNOME/gnome-remote-desktop>
- ext-image-copy-capture: <https://phoronix.com/news/Wayland-Merges-Screen-Capture>
- macOS: Sequoia prompts <https://9to5mac.com/2024/08/14/macos-sequoia-screen-recording-prompt-monthly/>, 15.1 change <https://macrumors.com/2024/10/07/apple-screen-recording-popup-update>, entitlement threads <https://developer.apple.com/forums/thread/770675>, <https://developer.apple.com/forums/thread/831795>, Remote Desktop TCC <https://developer.apple.com/forums/thread/841530>, CG capture APIs obsolete <https://groups.google.com/g/wx-dev/c/iReuuqxtGyk>
