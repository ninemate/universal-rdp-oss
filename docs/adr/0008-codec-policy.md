# 0008. Use OS-provided codecs by default; OpenH264 only as an opt-in download; bundle no patent-encumbered codecs

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner
- **Related:** [risks-and-open-questions.md §2](../research/risks-and-open-questions.md#2-codec-licensing), [vnc-protocol.md §2](../research/vnc-protocol.md#2-encodings), [rdp-protocol.md §3.1](../research/rdp-protocol.md#31-ms-rdpegfx-graphics-pipeline), [0006](0006-license-the-project-under-apache-2-0.md), [0014](0014-dependency-license-compatibility-policy.md)

## Context

- **H.264/AVC** (RDP AVC420/AVC444, RFB Open H.264 encoding 50) is covered by the Via LA patent pool plus non-pool holders. Distributors of compiled binaries with a built-in codec count units for royalties. US pool patents run until about 2027, with one (US 9356620) to 2030-11-26 ([risks §2.2–2.3](../research/risks-and-open-questions.md#22-h264avc-pool-terms-via-licensing-alliance)). **Unverified:** which late US patents are essential to the profiles RDP uses.
- **OS and hardware codecs** (VA-API, NVENC, Media Foundation, VideoToolbox) are shipped and licensed by the OS or GPU vendor. Caveat: Fedora and openSUSE disable H.264/HEVC in Mesa's VA-API, and Windows "N" editions lack Media Foundation codecs.
- **Cisco OpenH264**, as Cisco's own downloaded binary, is royalty-covered by Cisco, but must be separately downloaded, user-disableable, attributed, is Constrained Baseline only, has had a heap overflow (CVE-2025-27091), and a 2.6.0 ABI break.
- **Royalty-free paths exist for both protocols**: RFB Raw, CopyRect, Hextile, ZRLE, Tight (zlib + JPEG); RDP planar, interleaved RLE, RemoteFX/progressive, ClearCodec. The RDP ones are Microsoft-specified and fall under the RDP patent question ([0013](0013-legal-review-gate-for-rdp.md)) but no third-party pool.

## Decision

1. **Default builds and official binaries contain no H.264 (or HEVC/AV1) encoder or decoder** of our own or bundled from a third party.
2. **OS-provided codecs are the default H.264 path**, reached through a `VideoEncoder` / `VideoDecoder` trait: on Linux VA-API first (and NVENC where present), on Windows Media Foundation, on macOS VideoToolbox, in the browser WebCodecs (the browser's own codecs; WebCodecs was not covered by the research, and its H.264 availability per browser and OS is **Unverified**). Each is detected at runtime; absence is normal and must not be an error.
3. **Cisco OpenH264 is an opt-in runtime download.**
   - Off by default; enabled only after an explicit user or admin action that explains what is downloaded and why.
   - Downloaded from Cisco's binary distribution, with the version and SHA-256 pinned in our release and verified before loading; loaded with `libloading`, never linked.
   - Can be disabled and removed by the user at any time.
   - Shows "OpenH264 Video Codec provided by Cisco Systems, Inc." where the codec is listed.
   - Supported version range is pinned and tracked against Cisco security releases.
4. **Every non-H.264 path stays fully functional.** A server with no H.264 available still serves VNC with ZRLE/Tight and RDP with planar/RemoteFX/progressive. H.264 is an optimisation, never a requirement.
5. The supported software-codec policy is limited to the OS-provided path and the opt-in Cisco OpenH264 download above. **No other source-built H.264 codec is part of the accepted policy.** A future codec proposal needs a new review of patent exposure and must also follow the dependency policy if the owner accepts [0014](0014-dependency-license-compatibility-policy.md).
6. **Codecs that parse untrusted input from the network** (decoders on the client, and any decoder on the server) are candidates for running in a separate process, per [0003](0003-protocol-agnostic-core-sans-io-static-backends.md) point 7.

## Consequences

- No royalty tracking for official binaries; the project can publish releases without a Via LA agreement.
- On distributions without VA-API H.264 and without the OpenH264 opt-in, video-heavy content uses more bandwidth and CPU (ZRLE/Tight/RemoteFX). That is accepted.
- Encode quality on the OpenH264 path is limited to Constrained Baseline.
- CI must test both "H.264 available" and "no H.264" paths. Golden-image tests use PSNR/SSIM thresholds for lossy codecs ([0012](0012-testing-and-conformance.md)).
- Revisit around end-2030 when the late US H.264 patents expire, or earlier if counsel advises.

## Alternatives considered

- **Bundle OpenH264 built from source.** Simple, but we become a counted distributor.
- **Bundle FFmpeg/libavcodec.** LGPL obligations, a large C attack surface, and no patent licence.
- **No H.264 at all.** Rules out hardware-accelerated low-bandwidth video, which modern RDP clients expect; OS codecs avoid the licensing problem at no cost to us.
