# Roadmap

Draft; phases are ordered but not dated. Scope and order may change as research lands.

1. **Research.** Survey RDP and VNC specifications, prior art, licensing and implementation language options (see `docs/research`). Decide language and reuse strategy; record the decisions.
2. **Core abstractions.** Define session, input, display/frame, clipboard, audio and file-transfer interfaces, the backend plugin contract and the transport layer (TCP, TLS, WebSocket).
3. **VNC client.** First backend: RFB client, as the simplest protocol, to validate the core design end to end.
4. **RDP client.** Legacy first, then modern: TLS, NLA/CredSSP, graphics pipeline (RDPEGFX), channels for clipboard, audio and files.
5. **Servers.** Server role for VNC and RDP, built on the same core abstractions.
6. **Hardening.** Interoperability testing against existing clients and servers, fuzzing, security review, performance work, documentation and release process.
