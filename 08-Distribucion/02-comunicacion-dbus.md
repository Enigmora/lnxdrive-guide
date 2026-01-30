# Comunicacion D-Bus

> **Ubicación:** `08-Distribucion/02-comunicacion-dbus.md`
> **Relacionado:** [Adaptadores de Entrada](../03-Arquitectura/03-adaptadores-entrada.md), [Capas y Puertos](../03-Arquitectura/02-capas-y-puertos.md)

---

## Arquitectura de Comunicacion

Todos los clientes UI se comunican con el daemon a traves de D-Bus:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ARQUITECTURA DE COMUNICACION                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         lnxdrive-daemon                                     │
│                    (servicio systemd --user)                                │
│                              │                                              │
│              ┌───────────────┴───────────────┐                             │
│              │                               │                             │
│              │   D-Bus Session Bus           │                             │
│              │   org.enigmora.LNXDrive       │                             │
│              │                               │                             │
│              │   Interfaces:                 │                             │
│              │   ├── .Manager                │                             │
│              │   ├── .Sync                   │                             │
│              │   ├── .Files                  │                             │
│              │   ├── .Status                 │                             │
│              │   └── .Settings               │                             │
│              │                               │                             │
│              └───────────────┬───────────────┘                             │
│                              │                                              │
│    ┌─────────┬───────────┬───┴───┬───────────┬─────────┐                   │
│    │         │           │       │           │         │                   │
│    ▼         ▼           ▼       ▼           ▼         ▼                   │
│ ┌──────┐ ┌──────┐   ┌────────┐ ┌──────┐ ┌──────┐ ┌─────────┐              │
│ │GNOME │ │ GTK3 │   │ Plasma │ │Cosmic│ │ CLI  │ │ Otros   │              │
│ │ UI   │ │  UI  │   │   UI   │ │  UI  │ │      │ │(Thunar, │              │
│ │      │ │      │   │        │ │      │ │      │ │Nautilus)│              │
│ │ Rust │ │ Rust │   │  C++   │ │ Rust │ │ Rust │ │         │              │
│ └──────┘ └──────┘   └────────┘ └──────┘ └──────┘ └─────────┘              │
│                                                                             │
│  Cada UI usa lnxdrive-ipc (Rust) o implementacion D-Bus nativa             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Interfaz D-Bus

```xml
<!-- org.enigmora.LNXDrive.xml -->
<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
  "http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node name="/org/enigmora/LNXDrive">

  <interface name="org.enigmora.LNXDrive.Manager">
    <!-- Control del daemon -->
    <method name="Start"/>
    <method name="Stop"/>
    <method name="Restart"/>
    <method name="GetStatus">
      <arg type="s" direction="out" name="status"/>
    </method>
    <property name="Version" type="s" access="read"/>
    <property name="IsRunning" type="b" access="read"/>
  </interface>

  <interface name="org.enigmora.LNXDrive.Sync">
    <!-- Control de sincronizacion -->
    <method name="SyncNow"/>
    <method name="Pause"/>
    <method name="Resume"/>
    <method name="SyncPath">
      <arg type="s" direction="in" name="path"/>
    </method>
    <property name="SyncStatus" type="s" access="read"/>
    <property name="LastSyncTime" type="x" access="read"/>
    <property name="PendingChanges" type="u" access="read"/>

    <!-- Senales -->
    <signal name="SyncStarted"/>
    <signal name="SyncCompleted">
      <arg type="u" name="files_synced"/>
      <arg type="u" name="errors"/>
    </signal>
    <signal name="SyncProgress">
      <arg type="s" name="file"/>
      <arg type="u" name="current"/>
      <arg type="u" name="total"/>
    </signal>
    <signal name="ConflictDetected">
      <arg type="s" name="path"/>
      <arg type="s" name="conflict_type"/>
    </signal>
  </interface>

  <interface name="org.enigmora.LNXDrive.Files">
    <!-- Operaciones de archivos -->
    <method name="GetFileStatus">
      <arg type="s" direction="in" name="path"/>
      <arg type="s" direction="out" name="status"/>
    </method>
    <method name="PinFile">
      <arg type="s" direction="in" name="path"/>
    </method>
    <method name="UnpinFile">
      <arg type="s" direction="in" name="path"/>
    </method>
    <method name="FreeSpace">
      <arg type="s" direction="in" name="path"/>
    </method>
    <method name="GetConflicts">
      <arg type="as" direction="out" name="paths"/>
    </method>
  </interface>

  <interface name="org.enigmora.LNXDrive.Status">
    <!-- Informacion de estado -->
    <method name="GetQuota">
      <arg type="t" direction="out" name="used"/>
      <arg type="t" direction="out" name="total"/>
    </method>
    <method name="GetAccountInfo">
      <arg type="a{sv}" direction="out" name="info"/>
    </method>
    <property name="ConnectionStatus" type="s" access="read"/>
    <property name="WatchStats" type="a{sv}" access="read"/>

    <!-- Senales -->
    <signal name="QuotaChanged">
      <arg type="t" name="used"/>
      <arg type="t" name="total"/>
    </signal>
    <signal name="ConnectionChanged">
      <arg type="s" name="status"/>
    </signal>
  </interface>

</node>
```

## Libreria IPC Compartida

El crate `lnxdrive-ipc` proporciona un cliente D-Bus tipado para uso en clientes Rust:

```rust
// lnxdrive-ipc/src/lib.rs

use zbus::{Connection, proxy};

/// Cliente D-Bus para comunicacion con lnxdrive-daemon
#[derive(Clone)]
pub struct LnxDriveClient {
    manager: ManagerProxy<'static>,
    sync: SyncProxy<'static>,
    files: FilesProxy<'static>,
    status: StatusProxy<'static>,
}

impl LnxDriveClient {
    /// Conecta al daemon via D-Bus session bus
    pub async fn connect() -> Result<Self, LnxDriveError> {
        let connection = Connection::session().await?;

        Ok(Self {
            manager: ManagerProxy::new(&connection).await?,
            sync: SyncProxy::new(&connection).await?,
            files: FilesProxy::new(&connection).await?,
            status: StatusProxy::new(&connection).await?,
        })
    }

    /// Verifica si el daemon esta corriendo
    pub async fn is_running(&self) -> Result<bool, LnxDriveError> {
        Ok(self.manager.is_running().await?)
    }

    /// Inicia sincronizacion
    pub async fn sync_now(&self) -> Result<(), LnxDriveError> {
        self.sync.sync_now().await?;
        Ok(())
    }

    /// Obtiene estado de un archivo
    pub async fn get_file_status(&self, path: &str) -> Result<FileStatus, LnxDriveError> {
        let status_str = self.files.get_file_status(path).await?;
        FileStatus::from_str(&status_str)
    }

    /// Suscripcion a eventos de sincronizacion
    pub async fn subscribe_sync_events(&self) -> Result<SyncEventStream, LnxDriveError> {
        let progress = self.sync.receive_sync_progress().await?;
        let completed = self.sync.receive_sync_completed().await?;
        let conflicts = self.sync.receive_conflict_detected().await?;

        Ok(SyncEventStream::new(progress, completed, conflicts))
    }

    // ... mas metodos
}

/// Estados posibles de un archivo
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum FileStatus {
    /// Sincronizado y disponible localmente
    Synced,
    /// Solo en la nube (placeholder)
    CloudOnly,
    /// Sincronizando actualmente
    Syncing { progress: u8 },
    /// Pendiente de sincronizacion
    Pending,
    /// Error de sincronizacion
    Error { message: String },
    /// En conflicto
    Conflict,
    /// Excluido de sincronizacion
    Excluded,
}
```

## DBus API Multi-Cuenta

```xml
<!-- org.enigmora.LNXDrive.Accounts -->
<interface name="org.enigmora.LNXDrive.Accounts">
  <!-- Gestion de cuentas -->
  <method name="ListAccounts">
    <arg type="as" direction="out" name="account_ids"/>
  </method>
  <method name="AddAccount">
    <arg type="s" direction="in" name="provider"/>
    <arg type="s" direction="out" name="account_id"/>
  </method>
  <method name="RemoveAccount">
    <arg type="s" direction="in" name="account_id"/>
  </method>
  <method name="GetAccountInfo">
    <arg type="s" direction="in" name="account_id"/>
    <arg type="a{sv}" direction="out" name="info"/>
  </method>

  <!-- Senales -->
  <signal name="AccountAdded">
    <arg type="s" name="account_id"/>
  </signal>
  <signal name="AccountRemoved">
    <arg type="s" name="account_id"/>
  </signal>
</interface>
```

## Interfaces Disponibles

| Interfaz | Descripcion | Metodos Principales |
|----------|-------------|---------------------|
| `org.enigmora.LNXDrive.Manager` | Control del daemon | Start, Stop, Restart, GetStatus |
| `org.enigmora.LNXDrive.Sync` | Control de sincronizacion | SyncNow, Pause, Resume, SyncPath |
| `org.enigmora.LNXDrive.Files` | Operaciones de archivos | GetFileStatus, PinFile, UnpinFile, FreeSpace |
| `org.enigmora.LNXDrive.Status` | Informacion de estado | GetQuota, GetAccountInfo |
| `org.enigmora.LNXDrive.Accounts` | Gestion multi-cuenta | ListAccounts, AddAccount, RemoveAccount |

## Senales D-Bus

| Senal | Interfaz | Descripcion |
|-------|----------|-------------|
| `SyncStarted` | Sync | Inicio de sincronizacion |
| `SyncCompleted` | Sync | Fin de sincronizacion con estadisticas |
| `SyncProgress` | Sync | Progreso de archivo actual |
| `ConflictDetected` | Sync | Conflicto detectado |
| `QuotaChanged` | Status | Cambio en cuota de almacenamiento |
| `ConnectionChanged` | Status | Cambio en estado de conexion |
| `AccountAdded` | Accounts | Nueva cuenta agregada |
| `AccountRemoved` | Accounts | Cuenta eliminada |

---

## Ver tambien

- [01-estructura-repositorios.md](01-estructura-repositorios.md) - Estructura de repositorios
- [03-gobernanza-proyecto.md](03-gobernanza-proyecto.md) - Gobernanza y observabilidad
- [Capas y Puertos](../03-Arquitectura/02-capas-y-puertos.md) - Diagrama de componentes
