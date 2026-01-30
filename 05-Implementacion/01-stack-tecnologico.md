# Stack Tecnológico Propuesto

> **Ubicacion:** `05-Implementacion/01-stack-tecnologico.md`
> **Relacionado:** [Arquitectura Hexagonal](../03-Arquitectura/01-arquitectura-hexagonal.md), [Justificacion Rust](02-justificacion-rust.md)

---

## 7.1 Decision Principal: Lenguaje del Core

Despues de analizar las opciones, proponemos **dos caminos viables**:

### Opcion A: Core en Rust (Recomendado para Maximo Rendimiento)

```
┌──────────────────────────────────────────────────────────────────┐
│  STACK RUST                                                      │
│  ─────────────────────────────────────────────────────────────── │
│                                                                  │
│  Core:                                                           │
│  • Rust (memory safety, performance, async nativo)               │
│  • tokio (runtime async)                                         │
│  • serde (serialización)                                         │
│  • sqlx (SQLite async)                                           │
│                                                                  │
│  FUSE:                                                           │
│  • fuser (bindings libfuse3 modernos)                            │
│                                                                  │
│  IPC:                                                            │
│  • zbus (DBus async nativo Rust)                                 │
│                                                                  │
│  UI GNOME:                                                       │
│  • gtk4-rs + libadwaita-rs                                       │
│                                                                  │
│  UI KDE:                                                         │
│  • cxx-qt (bindings Qt6 desde Rust)                              │
│                                                                  │
│  UI Cosmic:                                                      │
│  • iced (nativo, mismo lenguaje!)                                │
│                                                                  │
│  HTTP:                                                           │
│  • reqwest (cliente HTTP async)                                  │
│  • oauth2-rs (flujos OAuth2)                                     │
│                                                                  │
│  Ventajas:                                                       │
│  ✓ Un solo lenguaje para daemon + UI Cosmic                      │
│  ✓ Memory safety sin GC                                          │
│  ✓ Excelente soporte async                                       │
│  ✓ Creciente adopcion en Linux desktop                           │
│                                                                  │
│  Desventajas:                                                    │
│  ✗ Curva de aprendizaje empinada                                 │
│  ✗ Bindings GTK/Qt menos maduros que nativos                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Opcion B: Core en .NET (Recomendado para Productividad)

```
┌─────────────────────────────────────────────────────────────────┐
│  STACK .NET                                                     │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Core:                                                          │
│  • C# 13 / .NET 10 (LTS, cross-platform maduro)                 │
│  • async/await nativo                                           │
│  • Entity Framework Core + SQLite                               │
│                                                                 │
│  FUSE:                                                          │
│  • Tmds.Fuse o LNXDrive.Fuse (wrapper propio moderno)           │
│  • Source generators para bindings                              │
│                                                                 │
│  IPC:                                                           │
│  • Tmds.DBus (DBus desde .NET)                                  │
│                                                                 │
│  UI Universal:                                                  │
│  • Avalonia UI (cross-platform, Linux-first)                    │
│  • .NET MAUI via Avalonia (2026, experimental)                  │
│                                                                 │
│  UI GNOME:                                                      │
│  • GtkSharp 4 (bindings GTK4)                                   │
│  • O usar Avalonia con theming GNOME                            │
│                                                                 │
│  HTTP:                                                          │
│  • Microsoft.Graph SDK (oficial!)                               │
│  • HttpClient nativo                                            │
│                                                                 │
│  Ventajas:                                                      │
│  ✓ SDK oficial de Microsoft Graph                               │
│  ✓ Ecosistema maduro y productivo                               │
│  ✓ Avalonia para UI consistente en todos los DE                 │
│  ✓ Native AOT para binarios pequenos y rapidos                  │
│                                                                 │
│  Desventajas:                                                   │
│  ✗ Runtime mas pesado (aunque AOT lo mitiga)                    │
│  ✗ Integracion desktop menos "nativa" que toolkits especificos  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 7.2 Recomendacion: Rust Core + UI Nativas por DE

```
┌─────────────────────────────────────────────────────────────────┐
│  ARQUITECTURA: RUST CORE + ADAPTADORES NATIVOS                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  lnxdrive-daemon (Rust):                                        │
│  • Core de sincronizacion                                       │
│  • Motor FUSE                                                   │
│  • Servicio DBus                                                │
│  • Alto rendimiento, bajo consumo                               │
│                                                                 │
│  lnxdrive-gnome (Python + GTK4):                                │
│  • Maxima integracion GNOME                                     │
│  • Usa PyGObject (maduro, bien documentado)                     │
│  • Extension Nautilus en Python                                 │
│                                                                 │
│  lnxdrive-kde (C++ + Qt6):                                      │
│  • Maxima integracion KDE                                       │
│  • KDE Frameworks nativo                                        │
│                                                                 │
│  lnxdrive-cosmic (Rust + iced):                                 │
│  • Comparte codigo con daemon                                   │
│  • 100% Rust, integracion perfecta                              │
│                                                                 │
│  lnxdrive-cli (Rust):                                           │
│  • Comparte codigo con daemon                                   │
│  • Binario standalone                                           │
│                                                                 │
│  Comunicacion: DBus (org.enigmora.LNXDrive)                     │
│  • Protocolo estandar freedesktop                               │
│  • Cualquier lenguaje puede implementar cliente                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ver tambien

- [Justificacion de Rust](02-justificacion-rust.md) - Analisis de rendimiento detallado
- [Arquitectura Hexagonal](../03-Arquitectura/01-arquitectura-hexagonal.md) - Filosofia arquitectonica
- [Adaptadores de Entrada](../03-Arquitectura/03-adaptadores-entrada.md) - UI intercambiables
- [Patrones Rust](04-patrones-rust.md) - Patrones de diseno en Rust
