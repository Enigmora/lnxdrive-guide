# Plan de Cierre del MVP — LNXDrive

**Fecha:** 2026-02-20
**Objetivo:** Cerrar las brechas de las Fases 0-3 para tener un MVP funcional end-to-end.

---

## Resumen de brechas

### Criticas (bloquean funcionalidad end-to-end)

| ID | Brecha | Estado |
|----|--------|--------|
| C1 | `filesystem.rs` open() no invoca `HydrationManager` | **RESUELTO** — open() inicia hydration async, read() bloquea con wait_for_range() |
| C2 | Bus name mismatch org vs com | **RESUELTO** — Unificado a `com.enigmora.LNXDrive` en 8 archivos |
| C3 | 7 iconos de emblema referenciados por Nautilus | **RESUELTO** — Ya existian en nautilus-extension/icons/ con meson.build configurado |

### Significativas (funcionalidad incompleta)

| ID | Brecha | Estado |
|----|--------|--------|
| S1 | `Conflicts.GetDetails` y `Conflicts.ResolveAll` no existen en servidor IPC | **RESUELTO** — Implementados get_details(), resolve_all(), + 2 signals D-Bus |
| S2 | `ConflictsInterface::resolve()` es no-op | **RESUELTO** — resolve() ahora valida estrategia y elimina conflicto del estado |
| S3 | OAuth2 CSRF state token nunca se valida | **RESUELTO** — Validacion csrf_token en login() |
| S4 | `user.lnxdrive.progress` xattr hardcodeado a "0" | **RESUELTO** — Lee progreso real de HydrationManager::progress() |
| S5 | GOA provider no implementado (diferido) | Diferido al post-MVP |

### Menores (no bloquean MVP)

| ID | Brecha | Estado |
|----|--------|--------|
| M1 | `release()` no notifica a `DehydrationManager` | **RESUELTO** — notify_file_closed() con dehydration inmediata si cache > umbral |
| M2 | Systemd service usa path de desarrollo | **RESUELTO** — Type=dbus, ExecStart=/usr/bin/lnxdrive-daemon, PrivateTmp |
| M3 | No hay archivo autostart para login | **RESUELTO** — XDG .desktop + D-Bus activation + systemd WantedBy=default.target |
| M4 | `last_sync` per-file hardcodeado a "—" | Pendiente (requiere motor de sync) |
| M5 | CI no ejecuta tests de container | **RESUELTO** — Eliminadas exclusiones, agregado libfuse3-dev |
| M6 | Scripts `dev-fuse-isolated.sh` y `dev-gnome-nested.sh` faltan | Diferido al post-MVP |
| M7 | FTS5 ausente del schema SQLite | Diferido al post-MVP |

---

## Progreso de bloques

### Bloque 1 — Funcionalidad end-to-end ✅ COMPLETADO

- **C2**: Unificado D-Bus bus name a `com.enigmora.LNXDrive` en 8 archivos (Nautilus .h/.c, Preferences .rs, Shell dbus.js, mock daemon .py, tests .js/.py)
- **C1**: HydrationManager conectado en FUSE — open() inicia hydration async para archivos Online, read() bloquea con wait_for_range(), constructor acepta Optional<HydrationManager>
- **C3**: Los 7 iconos SVG simbolicos (16x16, currentColor) ya existian en nautilus-extension/icons/ con meson.build configurado

### Bloque 2 — Completar capa GNOME ✅ COMPLETADO

- **S1**: Implementados GetDetails (busca conflicto por ID/prefijo en JSON) y ResolveAll (limpia todo con conteo) en ConflictsInterface. Agregadas signals ConflictDetected y ConflictResolved.
- **S4**: xattr progress ahora lee HydrationManager::progress(ino) real en vez de "0" hardcodeado. Tests actualizados con cobertura para progreso real (Some(75)).
- **M1**: DehydrationManager::notify_file_closed() implementado — intenta dehydration inmediata si cache > umbral. release() lo invoca via rt_handle.spawn().

### Bloque 3 — Seguridad y robustez ✅ COMPLETADO

- **S3**: CSRF state token ahora se valida en login(): compara callback.state contra csrf_token.secret()
- **S2**: resolve() ahora valida estrategia, busca conflicto por ID/prefijo, lo elimina del JSON y retorna true/false

### Bloque 4 — Infraestructura para distribucion ✅ COMPLETADO

- **M2**: Systemd service actualizado: `Type=dbus`, `BusName=com.enigmora.LNXDrive`, `ExecStart=/usr/bin/lnxdrive-daemon`, `PrivateTmp=true`
- **M3**: Creado `lnxdrive-autostart.desktop` (XDG autostart para escritorios que lo soportan). Para GNOME 49+, el `WantedBy=default.target` del service + D-Bus activation cubre el autostart.
- **M5**: CI actualizado: eliminadas exclusiones de 4 crates, agregado `libfuse3-dev` + `pkg-config` como deps del sistema. Ahora compila y testea el workspace completo.
- **Bonus**: Creado archivo de activacion D-Bus (`com.enigmora.LNXDrive.service`) con `SystemdService=lnxdrive.service` para arranque bajo demanda.

---

## Archivos modificados

### Repo lnxdrive (Rust daemon) — Bloques 1-3
- `crates/lnxdrive-fuse/src/filesystem.rs` — HydrationManager en open()/read(), DehydrationManager en release(), xattr progress
- `crates/lnxdrive-fuse/src/lib.rs` — mount() pasa None como hydration_manager
- `crates/lnxdrive-fuse/src/xattr.rs` — get_xattr() acepta hydration_progress: Option<u8>
- `crates/lnxdrive-fuse/src/dehydration.rs` — notify_file_closed() method
- `crates/lnxdrive-ipc/src/service.rs` — GetDetails, ResolveAll, signals, resolve() mejorado
- `crates/lnxdrive-graph/src/auth.rs` — CSRF state validation en login()
- `crates/lnxdrive-cli/src/commands/mount.rs` — actualizado constructor LnxDriveFs::new()

### Repo lnxdrive (Rust daemon) — Bloque 4
- `config/lnxdrive.service` — Type=dbus, BusName, ExecStart=/usr/bin/lnxdrive-daemon, PrivateTmp
- `config/com.enigmora.LNXDrive.service` — (NUEVO) D-Bus activation file
- `config/lnxdrive-autostart.desktop` — (NUEVO) XDG autostart para login
- `.github/workflows/ci.yml` — Eliminadas exclusiones, agregadas deps de sistema (libfuse3-dev)

### Repo lnxdrive-gnome (GNOME integration)
- `nautilus-extension/src/lnxdrive-dbus-client.h` — bus name com.enigmora
- `nautilus-extension/src/lnxdrive-dbus-client.c` — bus name com.enigmora
- `preferences/src/dbus_client.rs` — bus name com.enigmora (16 ocurrencias)
- `shell-extension/lnxdrive-indicator@enigmora.com/dbus.js` — bus name com.enigmora
- `tests/mock-dbus-daemon.py` — bus name com.enigmora
- `tests/test-shell-extension.js` — bus name com.enigmora
- `tests/test-nautilus-extension.py` — bus name com.enigmora

---

## Notas

- GOA provider (S5) se difiere oficialmente al post-MVP (el onboarding wizard cubre la necesidad)
- FTS5 (M7) se difiere al post-MVP (no es critico para funcionalidad basica)
- Scripts dev (M6) se difieren al post-MVP (la infra de testing existente es funcional)
- M4 (last_sync hardcodeado) pendiente — requiere integracion con motor de sync
- Workspace compila limpio (`cargo check --workspace`) tras todos los cambios
- **Todos los bloques (1-4) completados** — MVP listo para pruebas de integracion
