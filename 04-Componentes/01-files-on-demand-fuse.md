# Files-on-Demand para Linux

> **Ubicación:** `04-Componentes/01-files-on-demand-fuse.md`
> **Relacionado:** [Arquitectura Hexagonal](../03-Arquitectura/01-arquitectura-hexagonal.md)

---

## Parte III: Files-on-Demand para Linux

### 3.1 El Desafio

Windows tiene [Cloud Files API (cfapi)](https://learn.microsoft.com/en-us/windows/win32/cfapi/build-a-cloud-file-sync-engine) integrada en el kernel con `cldflt.sys`. Esta API **no existe en Linux** porque depende de caracteristicas especificas de NTFS.

> "Cldflt.sys currently only supports NTFS volumes because it depends on some features unique to NTFS." — Microsoft Documentation

### 3.2 Nuestra Solucion: FUSE + Overlay + GIO

Implementaremos Files-on-Demand usando una combinacion de tecnologias:

```
┌───────────────────────────────────────────────────────────────────┐
│                    CAPA DE PRESENTACION                           │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  Administrador de Archivos (Nautilus/Dolphin/Thunar)       │   │
│  │  ────────────────────────────────────────────────────────  │   │
│  │  • Overlay icons via GIO/KIO extension                     │   │
│  │  • Menu contextual: "Make available offline"               │   │
│  │  • Indicador de estado: ☁️ online | ✓ local | ⟳ syncing   │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              │ GIO/KIO API                        │
│                              ▼                                    │
├───────────────────────────────────────────────────────────────────┤
│                    CAPA FUSE (Userspace)                          │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │  lnxdrive-fuse daemon                                       │    │
│  │  ───────────────────────────────────────────────────────  │    │
│  │  Implementa operaciones FUSE:                             │    │
│  │  • getattr() → Retorna metadata sin descargar contenido   │    │
│  │  • open() → Trigger de hidratacion si es placeholder      │    │
│  │  • read() → Streaming desde cache o desde nube            │    │
│  │  • readdir() → Lista desde cache de metadata              │    │
│  │  • setxattr() → "user.lnxdrive.state" para marcar estado    │    │
│  └───────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              │ Callbacks al Core                  │
│                              ▼                                    │
├───────────────────────────────────────────────────────────────────┤
│                    NUCLEO DE DOMINIO                              │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  HydrationManager                                           │  │
│  │  ─────────────────────────────────────────────────────────  │  │
│  │  • Gestiona cola de hidratacion con prioridades             │  │
│  │  • Streaming parcial (range requests) para archivos grandes │  │
│  │  • Cache LRU para archivos hidratados recientemente         │  │
│  │  • Dehydration automatica cuando espacio es bajo            │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

### 3.3 Estados de Archivo

```
┌─────────────────────────────────────────────────────────────────┐
│  PLACEHOLDER (Online-only)                                      │
│  ────────────────────────────────────────────────────────────── │
│  • Archivo sparse de 0 bytes en disco                           │
│  • Metadata completa en extended attributes                     │
│  • xattr: user.lnxdrive.state = "online"                          │
│  • xattr: user.lnxdrive.size = "1234567" (tamano real)            │
│  • xattr: user.lnxdrive.remote_id = "abc123"                      │
│  • Icono: ☁️ nube                                               │
├─────────────────────────────────────────────────────────────────┤
│  HYDRATING (Descargando)                                        │
│  ────────────────────────────────────────────────────────────── │
│  • Archivo parcialmente descargado                              │
│  • xattr: user.lnxdrive.state = "hydrating"                       │
│  • xattr: user.lnxdrive.progress = "45"                           │
│  • Icono: ⟳ sync spinner                                        │
├─────────────────────────────────────────────────────────────────┤
│  HYDRATED (Disponible offline)                                  │
│  ────────────────────────────────────────────────────────────── │
│  • Contenido completo en disco                                  │
│  • xattr: user.lnxdrive.state = "hydrated"                        │
│  • Puede ser dehidratado si espacio es necesario                │
│  • Icono: ✓ check verde                                         │
├─────────────────────────────────────────────────────────────────┤
│  PINNED (Siempre offline)                                       │
│  ────────────────────────────────────────────────────────────── │
│  • Usuario marco explicitamente "Keep on device"                │
│  • xattr: user.lnxdrive.state = "pinned"                          │
│  • Nunca se dehidrata automaticamente                           │
│  • Icono: 📌 pin                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Implementacion FUSE Moderna

Para la implementacion FUSE, proponemos crear un **wrapper moderno para .NET 10+** que supere las limitaciones de proyectos existentes:

**Estado Actual de FUSE en .NET:**
- [Tmds.Fuse](https://github.com/tmds/Tmds.Fuse) — Prometedor pero limitado
- [Mono.Fuse.NETStandard](https://www.nuget.org/packages/Mono.Fuse.NETStandard) — Port del viejo Mono.Fuse
- fusedotnet — Anticuado, solo wrapper basico

**Propuesta: `LNXDrive.Fuse` — Wrapper FUSE Moderno para .NET**

```csharp
// API propuesta para LNXDrive.Fuse
public interface IFuseFileSystem
{
    ValueTask<Stat> GetAttributesAsync(ReadOnlySpan<char> path, CancellationToken ct);
    ValueTask<int> ReadAsync(ReadOnlySpan<char> path, Memory<byte> buffer,
                              long offset, CancellationToken ct);
    ValueTask<IEnumerable<DirectoryEntry>> ReadDirectoryAsync(ReadOnlySpan<char> path,
                                                               CancellationToken ct);
    ValueTask<int> OpenAsync(ReadOnlySpan<char> path, OpenFlags flags, CancellationToken ct);
    // ... mas operaciones
}

// Caracteristicas modernas:
// • Async/await nativo (no callbacks bloqueantes)
// • Memory<T> y Span<T> para zero-copy
// • Source generators para bindings libfuse3
// • Compatible con .NET 10+ y Native AOT
```

---

## Ver tambien

- [Arquitectura Hexagonal](../03-Arquitectura/01-arquitectura-hexagonal.md) - Visión general del sistema
- [Adaptador GNOME](02-ui-gnome.md) - Integracion con GNOME
- [Adaptador KDE Plasma](03-ui-kde-plasma.md) - Integracion con KDE
- [CLI Universal](06-cli.md) - Interfaz de linea de comandos
