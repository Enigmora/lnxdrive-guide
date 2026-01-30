# Politicas Declarativas YAML

> **Ubicacion:** `05-Implementacion/05-configuracion-yaml.md`
> **Relacionado:** [Stack Tecnologico](01-stack-tecnologico.md), [Principios Rectores](../01-Vision/02-principios-rectores.md)

---

## Parte V: Gobernanza y Observabilidad

### 5.1 Politicas Declarativas (YAML)

```yaml
# ~/.config/lnxdrive/config.yaml
# Configuracion versionable, reproducible, auditable

lnxdrive:
  version: 1

  account:
    # Soporta App ID propio o usa el default
    # Referencia: modelo hibrido recomendado por la investigacion
    app_id: "default"  # o UUID personalizado

  sync:
    root: ~/OneDrive
    direction: bidirectional  # bidirectional | upload-only | download-only

    # Politicas de hidratacion
    hydration:
      strategy: on-demand      # on-demand | eager | manual
      cache_limit: 10GB        # Espacio maximo para archivos hidratados
      eviction_policy: lru     # lru | lfu | oldest

    # Archivos grandes
    large_files:
      threshold: 100MB
      strategy: chunked        # chunked | streaming | defer

    # Exclusiones con patrones glob
    exclude:
      - "**/.git"
      - "**/.git/**"
      - "**/node_modules/**"
      - "**/__pycache__/**"
      - "**/*.tmp"
      - "**/~$*"              # Archivos temporales de Office

    # Inclusiones explicitas (sobrescriben exclusiones)
    include:
      - "**/important.git"

  conflicts:
    # Estrategia por defecto
    default_strategy: prompt   # prompt | keep-local | keep-remote | keep-both

    # Reglas especificas por patron
    rules:
      - pattern: "**/*.docx"
        strategy: keep-both    # Documentos: siempre conservar ambos

      - pattern: "**/config/**"
        strategy: keep-remote  # Configs: preferir version del servidor

      - pattern: "**/src/**"
        strategy: prompt       # Codigo: siempre preguntar

  observability:
    # Nivel de logging
    log_level: info           # trace | debug | info | warn | error

    # Archivo de log estructurado
    log_file: ~/.local/share/lnxdrive/lnxdrive.log
    log_format: json          # json | text

    # Retencion de auditoria
    audit_retention: 30d

    # Metricas Prometheus
    metrics:
      enabled: true
      endpoint: "127.0.0.1:9100"

  notifications:
    # Cuando notificar
    on_conflict: always
    on_error: always
    on_sync_complete: summary  # always | summary | never

  systemd:
    # Integracion con systemd
    restart_on_failure: true
    watchdog_interval: 30s
```

### 5.2 Auditoria y Explicabilidad

Cada accion genera una entrada de auditoria explicable:

```json
{
  "timestamp": "2026-01-28T15:30:45.123Z",
  "event_id": "evt_abc123",
  "action": "sync_skip",
  "item": {
    "local_path": "/home/user/OneDrive/document.docx",
    "remote_id": "ABC123!456",
    "state": "modified"
  },
  "result": "skipped",
  "reason": {
    "code": "REMOTE_MODIFIED_DURING_UPLOAD",
    "explanation": "El archivo fue modificado en OneDrive mientras se subia la version local. Se detecto un conflicto.",
    "details": {
      "local_hash": "sha256:abc...",
      "remote_hash": "sha256:def...",
      "local_modified": "2026-01-28T15:30:00Z",
      "remote_modified": "2026-01-28T15:30:30Z"
    }
  },
  "resolution": {
    "action_taken": "conflict_created",
    "conflict_id": "conf_xyz789",
    "suggested_strategies": ["keep-local", "keep-remote", "keep-both"]
  }
}
```

### 5.3 Metricas Prometheus

```prometheus
# HELP lnxdrive_files_total Total de archivos gestionados
# TYPE lnxdrive_files_total gauge
lnxdrive_files_total{state="online"} 1234
lnxdrive_files_total{state="hydrated"} 567
lnxdrive_files_total{state="modified"} 12
lnxdrive_files_total{state="conflicted"} 3

# HELP lnxdrive_sync_operations_total Operaciones de sincronizacion
# TYPE lnxdrive_sync_operations_total counter
lnxdrive_sync_operations_total{operation="upload",status="success"} 456
lnxdrive_sync_operations_total{operation="upload",status="failed"} 2
lnxdrive_sync_operations_total{operation="download",status="success"} 789
lnxdrive_sync_operations_total{operation="download",status="failed"} 1

# HELP lnxdrive_sync_bytes_total Bytes transferidos
# TYPE lnxdrive_sync_bytes_total counter
lnxdrive_sync_bytes_total{direction="upload"} 1234567890
lnxdrive_sync_bytes_total{direction="download"} 9876543210

# HELP lnxdrive_api_requests_total Llamadas a Microsoft Graph
# TYPE lnxdrive_api_requests_total counter
lnxdrive_api_requests_total{endpoint="delta",status="200"} 100
lnxdrive_api_requests_total{endpoint="delta",status="429"} 2

# HELP lnxdrive_conflicts_total Conflictos detectados
# TYPE lnxdrive_conflicts_total counter
lnxdrive_conflicts_total{resolution="auto"} 50
lnxdrive_conflicts_total{resolution="manual"} 5
lnxdrive_conflicts_total{resolution="pending"} 3

# HELP lnxdrive_hydration_duration_seconds Tiempo de hidratacion
# TYPE lnxdrive_hydration_duration_seconds histogram
lnxdrive_hydration_duration_seconds_bucket{le="1"} 500
lnxdrive_hydration_duration_seconds_bucket{le="5"} 800
lnxdrive_hydration_duration_seconds_bucket{le="30"} 950
lnxdrive_hydration_duration_seconds_bucket{le="+Inf"} 1000
```

## Ventajas de la Configuracion Declarativa

| Aspecto | Configuracion Imperativa (flags) | Configuracion Declarativa (YAML) |
|---------|----------------------------------|----------------------------------|
| **Versionado** | Dificil de rastrear | Git-friendly, historial completo |
| **Reproducibilidad** | Requiere documentacion externa | Auto-documentado |
| **Auditoria** | Manual | Automatica con timestamps |
| **Validacion** | En tiempo de ejecucion | En tiempo de carga + esquema |
| **Compartir** | Copiar comandos | Copiar archivo |
| **Rollback** | Manual | `git checkout` |

---

## Ver tambien

- [Principios Rectores](../01-Vision/02-principios-rectores.md) - Filosofia del proyecto
- [Stack Tecnologico](01-stack-tecnologico.md) - Vision general del stack
- [Patrones Rust](04-patrones-rust.md) - Patrones de implementacion
