---
id: AIDEC-2026-02-02-001
title: SQLite Bundled vs System Library
status: accepted
created: 2026-02-02
agent: gemini-cli-v1.0
confidence: high
review_required: false
risk_level: low
tags: [sqlite, dependencies, distribution, rust]
related: [05-Implementacion/01-stack-tecnologico.md, 03-Arquitectura/04-adaptadores-salida.md]
---

# AIDEC: SQLite Bundled vs System Library

## Context

LNXDrive requiere persistencia local de estado de sincronización usando SQLite a través del crate `sqlx`. Al configurar la dependencia, existen dos modos de vinculación con el motor SQLite.

## Problem

¿Debe el binario de LNXDrive incluir SQLite compilado internamente (`bundled`) o depender de la librería `libsqlite3` instalada en el sistema (`system`)?

## Alternatives Considered

### Alternative 1: System Library

**Description**: Vincular dinámicamente contra `libsqlite3.so` del sistema operativo.

```toml
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite"] }
```

**Pros**:
- Binario más pequeño (~1-2 MB menos)
- Actualizaciones de seguridad automáticas vía gestor de paquetes del sistema
- Menos tiempo de compilación

**Cons**:
- Requiere `libsqlite3-dev` o `sqlite-devel` instalado
- Versión de SQLite varía entre distribuciones (potencial incompatibilidad)
- Complica empaquetado Flatpak/AppImage
- Usuario debe resolver dependencia manualmente en algunas distros

### Alternative 2: Bundled (Incluido en el binario)

**Description**: Compilar SQLite estáticamente dentro del binario de LNXDrive.

```toml
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "bundled"] }
```

**Pros**:
- Cero dependencias externas en runtime
- Versión consistente de SQLite en todas las instalaciones
- Empaquetado simplificado (.deb, .rpm, Flatpak, AppImage)
- Funciona en cualquier distribución sin configuración adicional

**Cons**:
- Binario ~1-2 MB más grande
- El desarrollador debe actualizar a nuevas versiones de `sqlx` para parches de seguridad de SQLite
- Tiempo de compilación ligeramente mayor

## Decision

**Chosen**: Alternative 2 - Bundled

**Justification**: LNXDrive tiene como objetivo ser una aplicación portable para múltiples distribuciones Linux (Fedora, Ubuntu, Arch, openSUSE, etc.) y formatos de empaquetado (nativo, Flatpak, AppImage). El modo `bundled`:

1. **Elimina fricciones de instalación** — El usuario no necesita instalar dependencias adicionales.
2. **Garantiza comportamiento consistente** — Todas las instalaciones usan la misma versión de SQLite.
3. **Simplifica CI/CD** — Un solo binario funciona en todos los targets.
4. **Alineado con filosofía Rust** — Binarios autocontenidos son el estándar esperado.

## Consequences

### Positive
- Instalación sin dependencias externas
- Comportamiento predecible en todas las distribuciones
- Empaquetado Flatpak/AppImage trivial
- Testing más confiable (misma versión SQLite en dev y prod)

### Negative
- Tamaño del binario incrementa ~1-2 MB
- Responsabilidad de actualizar dependencias para parches de seguridad

### Risks
- **Vulnerabilidades en SQLite embebido**: Mitigación → Mantener `sqlx` actualizado y monitorear CVEs de SQLite.

## Implementation

Configurar `Cargo.toml` del crate `lnxdrive-daemon`:

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio",
    "sqlite",
    "bundled",      # ← Incluir motor SQLite en el binario
    "migrate",
] }
```

## References

- [sqlx - Documentación oficial](https://docs.rs/sqlx/latest/sqlx/)
- [SQLite Bundled Feature](https://github.com/launchbadge/sqlx/blob/main/sqlx-sqlite/README.md)
- [Stack Tecnológico LNXDrive](../../../05-Implementacion/01-stack-tecnologico.md)
- [Adaptadores de Salida](../../../03-Arquitectura/04-adaptadores-salida.md)

---

<!-- Template: DevTrail | https://enigmora.com -->
