# LNXDrive Guide

Technical design and development documentation for **LNXDrive**, a native cloud storage synchronization client for Linux supporting multiple providers (OneDrive, Google Drive, Dropbox, and more).

## About LNXDrive

LNXDrive is a cloud storage client featuring:

- **Files-on-Demand** via FUSE — access cloud files without downloading them
- **Hexagonal Architecture** — clean separation between core logic and adapters
- **Multi-Desktop Support** — native integration for GNOME, KDE Plasma, XFCE/MATE, and Cosmic
- **Multi-Provider Ready** — extensible architecture for future cloud providers
- **Built in Rust** — performance, safety, and low resource footprint

## Repository Structure

```
├── 01-Vision/          # Project concept and value proposition
├── 02-Analisis/        # Market analysis and opportunities
├── 03-Arquitectura/    # High-level system design
├── 04-Componentes/     # Detailed component specifications
├── 05-Implementacion/  # Technology stack and coding guidelines
├── 06-Testing/         # Testing strategies and CI/CD
├── 07-Extensibilidad/  # Multi-provider and multi-account design
├── 08-Distribucion/    # Packaging, D-Bus API, and i18n
├── 09-Referencia/      # Roadmap and external references
└── Anexos/             # Complete technical documents
```

## Getting Started

Start with the main guide: **[Guía-de-diseño-y-desarrollo.md](Guía-de-diseño-y-desarrollo.md)**

For AI agents and developers: each document is self-contained and can be consulted independently. See the navigation guide in the main document for task-specific recommendations.

## Related Repositories

This is part of the LNXDrive multi-repo project. Implementation repositories will be linked here as they become available.

## License

See [LICENSE](LICENSE) for details.

---

*[Enigmora](https://enigmora.com) — LNXDrive Project*
