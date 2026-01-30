# Adaptadores de Salida (Infraestructura)

> **Ubicación:** `03-Arquitectura/04-adaptadores-salida.md`
> **Relacionado:** [Capas y Puertos](02-capas-y-puertos.md), [Microsoft Graph](../04-Componentes/08-microsoft-graph.md)

---

## Resumen de Adaptadores

| Adaptador | Tecnología | Propósito |
|-----------|-----------|-----------|
| `msgraph-adapter` | Microsoft Graph API | Comunicación con OneDrive |
| `fuse-adapter` | libfuse3 / FUSE wrapper | Files-on-Demand (placeholders) |
| `sqlite-adapter` | SQLite | Persistencia de estado |
| `dbus-notify-adapter` | DBus org.freedesktop.Notifications | Notificaciones desktop |
| `prometheus-adapter` | Prometheus format | Métricas exportables |
| `gvfs-adapter` | GIO/GVfs | Integración GNOME Files |

## Detalle por Adaptador

### Microsoft Graph Adapter

Implementa el puerto `ICloudProvider` para OneDrive:

**Responsabilidades:**
- Autenticación OAuth2 + PKCE
- Delta sync (obtener cambios incrementales)
- Upload/download de archivos
- Manejo de rate limiting (429)
- Upload en chunks para archivos grandes

**Tecnologías:**
- `reqwest` para HTTP async
- `oauth2-rs` para flujos OAuth2
- Token bucket para rate limiting proactivo

### FUSE Adapter

Implementa el puerto `ILocalFileSystem` usando FUSE:

**Responsabilidades:**
- Crear y mantener placeholders
- Hidratación on-demand cuando se accede a archivos
- Extended attributes para metadata
- Comunicación con el core sobre accesos

**Tecnologías:**
- `fuser` (bindings libfuse3 para Rust)
- Extended attributes (xattr)

### SQLite Adapter

Implementa el puerto `IStateRepository`:

**Responsabilidades:**
- Persistir estado de cada SyncItem
- Almacenar audit trail
- Guardar delta tokens
- Cache de metadata

**Tecnologías:**
- `sqlx` para acceso async a SQLite
- Migraciones embebidas

### DBus Notify Adapter

Implementa el puerto `INotificationService`:

**Responsabilidades:**
- Mostrar notificaciones de escritorio
- Progress notifications para operaciones largas
- Acciones en notificaciones (abrir conflicto, etc.)

**Tecnologías:**
- `zbus` para DBus async
- `org.freedesktop.Notifications` interface

### Prometheus Adapter

Implementa el puerto `IMetricsExporter`:

**Responsabilidades:**
- Exponer métricas en formato Prometheus
- Endpoint HTTP para scraping
- Métricas de sync, conflictos, rate limiting

**Métricas exportadas:**
```prometheus
lnxdrive_files_total{state="online"} 1234
lnxdrive_sync_operations_total{operation="upload",status="success"} 456
lnxdrive_api_requests_total{endpoint="delta",status="200"} 100
lnxdrive_conflicts_total{resolution="pending"} 3
```

### GVfs Adapter

Integración con GNOME Files:

**Responsabilidades:**
- Proveer overlay icons via GIO
- Columnas personalizadas en Nautilus
- Integración con GNOME Online Accounts

---

## Ver también

- [Capas y Puertos](02-capas-y-puertos.md) - Interfaces que implementan
- [Microsoft Graph](../04-Componentes/08-microsoft-graph.md) - Detalle del adaptador Graph
- [Files-on-Demand FUSE](../04-Componentes/01-files-on-demand-fuse.md) - Detalle del adaptador FUSE
- [Puerto ICloudProvider](../07-Extensibilidad/02-puerto-icloudprovider.md) - Interfaz para proveedores cloud
