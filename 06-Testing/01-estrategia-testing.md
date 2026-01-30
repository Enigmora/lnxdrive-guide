# Estrategia de Testing

> **Ubicación:** `06-Testing/01-estrategia-testing.md`
> **Relacionado:** [Anexo A: Metodologías de Testing](../Anexos/A-metodologias-testing-completo.md)

---

## 1. Introduccion y Problematica

LNXDrive involucra tres categorias de componentes que interactuan profundamente con el sistema operativo:

| Componente | Riesgo si falla | Impacto potencial |
|------------|-----------------|-------------------|
| **Servicios systemd** | Medio-Alto | Procesos zombie, recursos no liberados, conflictos de puertos |
| **FUSE filesystem** | Alto | Kernel panic (raro), procesos colgados en I/O, puntos de montaje huerfanos |
| **Extensiones desktop** | Alto | GNOME Shell crash, Nautilus inutilizable, sesion grafica corrupta |

**El desafio**: Desarrollar y depurar estos componentes sin:
- Bloquear la sesion de trabajo del desarrollador
- Corromper el entorno de escritorio
- Dejar el sistema en estado inconsistente
- Perder trabajo por crashes del entorno grafico

---

## 2. Estrategias de Aislamiento

### 2.1 Niveles de Aislamiento Disponibles

```
+---------------------------------------------------------------------------+
|  ESPECTRO DE AISLAMIENTO                                                  |
+---------------------------------------------------------------------------+
|                                                                           |
|  Menor aislamiento <------------------------------> Mayor aislamiento     |
|  Mayor velocidad                                      Menor velocidad     |
|                                                                           |
|  +---------+ +---------+ +---------+ +---------+ +---------+              |
|  | Proceso | |Namespace| |Container| |   VM    | | VM con  |              |
|  | normal  | | aislado | |(Podman) | | headless| | GUI     |              |
|  +---------+ +---------+ +---------+ +---------+ +---------+              |
|       |           |           |           |           |                   |
|       v           v           v           v           v                   |
|   Unit tests   FUSE dev   systemd    FUSE+systemd  Desktop                |
|   Mocks        aislado    testing    integration   extensions             |
|                                                                           |
+---------------------------------------------------------------------------+
```

### 2.2 Recomendacion por Componente

| Componente | Desarrollo | Testing Unitario | Testing Integracion | Testing E2E |
|------------|------------|------------------|---------------------|-------------|
| Core (logica) | Host directo | Host directo | Host directo | Container |
| FUSE | Namespace/Container | Container | VM headless | VM headless |
| systemd service | User systemd | Container con systemd | VM headless | VM headless |
| Nautilus extension | Container GUI | Container GUI | VM con GNOME | VM con GNOME |
| GNOME Shell ext | VM con GNOME | VM con GNOME | VM con GNOME | VM con GNOME |

---

## Ver tambien

- [Testing de Servicios systemd](02-testing-systemd.md) - Desarrollo y testing de servicios systemd
- [Testing de FUSE Filesystem](03-testing-fuse.md) - Testing del sistema de archivos FUSE
- [Testing de Extensiones Desktop](04-testing-desktop.md) - Testing de Nautilus y GNOME Shell
- [Mocking de APIs Externas](05-mocking-apis.md) - Mock de Microsoft Graph API
- [Pipeline CI/CD](06-ci-cd-pipeline.md) - Integracion continua y despliegue
