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

Para la implementacion FUSE usamos el crate [fuser](https://crates.io/crates/fuser), el binding FUSE mas maduro y activo en el ecosistema Rust.

**Estado Actual de FUSE en Rust:**
- [fuser](https://github.com/cberner/fuser) — Fork activo de rust-fuse, bien mantenido
- Soporte completo para libfuse3
- API sincrona con integracion tokio disponible

**Implementacion: `lnxdrive-fuse`**

```rust
use fuser::{Filesystem, Request, ReplyAttr, ReplyData, ReplyDirectory};
use std::ffi::OsStr;
use std::time::Duration;

/// Filesystem virtual para Files-on-Demand
pub struct LnxDriveFs {
    state_repo: Arc<dyn IStateRepository>,
    hydration_manager: Arc<HydrationManager>,
}

impl Filesystem for LnxDriveFs {
    fn getattr(&mut self, _req: &Request, ino: u64, reply: ReplyAttr) {
        // Retorna metadata sin descargar contenido
        match self.state_repo.get_item_by_inode(ino) {
            Some(item) => reply.attr(&Duration::from_secs(1), &item.to_file_attr()),
            None => reply.error(libc::ENOENT),
        }
    }

    fn read(&mut self, _req: &Request, ino: u64, _fh: u64,
            offset: i64, size: u32, _flags: i32, _lock: Option<u64>, reply: ReplyData) {
        // Hidrata on-demand si es necesario
        let data = self.hydration_manager.read_with_hydration(ino, offset, size);
        match data {
            Ok(bytes) => reply.data(&bytes),
            Err(e) => reply.error(e.to_errno()),
        }
    }

    fn readdir(&mut self, _req: &Request, ino: u64, _fh: u64,
               offset: i64, mut reply: ReplyDirectory) {
        // Lista desde cache de metadata (sin descargas)
        for (i, entry) in self.state_repo.list_children(ino).skip(offset as usize).enumerate() {
            if reply.add(entry.ino, (offset + i as i64 + 1), entry.kind, &entry.name) {
                break;
            }
        }
        reply.ok();
    }
}

// Caracteristicas:
// • Sin GC: latencia predecible <1ms para getattr
// • Zero-copy con slices para operaciones de lectura
// • Integracion con tokio para I/O asincrono
// • Extended attributes via xattr para estado de sync
```

---

## Ver tambien

- [Arquitectura Hexagonal](../03-Arquitectura/01-arquitectura-hexagonal.md) - Visión general del sistema
- [Adaptador GNOME](02-ui-gnome.md) - Integracion con GNOME
- [Adaptador KDE Plasma](03-ui-kde-plasma.md) - Integracion con KDE
- [CLI Universal](06-cli.md) - Interfaz de linea de comandos
