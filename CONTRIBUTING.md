# Contributing

Thanks for your interest in universal-rdp-oss. The project is pre-alpha, so the most useful contributions right now are feedback on the design and research.

- Read [README.md](README.md), [docs/architecture.md](docs/architecture.md), [docs/ROADMAP.md](docs/ROADMAP.md) and the [ADRs](docs/adr/README.md) first. Structural changes need an ADR (see [docs/adr/README.md](docs/adr/README.md)).
- Open an issue before starting large changes, so we can agree on direction.
- Keep pull requests small and focused, with a clear description of the why.
- Everything (code, docs, issues, commit messages) is in English.
- Follow `.editorconfig`. The implementation language is Rust ([ADR 0002](docs/adr/0002-implementation-language-rust.md)); Rust style and lint rules arrive with the workspace in milestone M0.
- By contributing, you agree your contribution is licensed under the [Apache License 2.0](LICENSE).
- Be kind: see the [Code of Conduct](CODE_OF_CONDUCT.md).
- Do not paste proprietary protocol specifications or code with incompatible licenses into the repository. Never copy code from GPL VNC/RDP implementations (TigerVNC, libvncserver, x11vnc, UltraVNC, TightVNC, GNOME Remote Desktop, …); they are test peers only ([ADR 0005](docs/adr/0005-rdp-on-ironrdp-vnc-from-scratch.md)).
- A Developer Certificate of Origin sign-off on every commit is proposed in [ADR 0007](docs/adr/0007-contribution-policy-dco.md).
