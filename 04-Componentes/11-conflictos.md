# Resolucion de Conflictos Visual

> **Ubicacion:** `04-Componentes/11-conflictos.md`
> **Relacionado:** [[04-Componentes/07-motor-sincronizacion|Motor de Sincronizacion]], [[04-Componentes/12-auditoria|Auditoria]]

---

## Parte VI: Resolucion de Conflictos Visual

### 6.1 Flujo de Resolucion

```
┌───────────────────────────────────────────────────────────────────┐
│  DETECCION DE CONFLICTO                                           │
│  ───────────────────────────────────────────────────────────────  │
│                                                                   │
│  Condicion: local_hash != remote_hash && ambos modificados        │
│             despues de ultima sincronizacion exitosa              │
│                                                                   │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  EVALUACION DE POLITICA                                          │
│  ─────────────────────────────────────────────────────────────   │
│                                                                  │
│  1. Buscar regla especifica en config.yaml                       │
│  2. Si no hay regla -> usar default_strategy                     │
│  3. Si strategy = "prompt" -> notificar al usuario               │
│                                                                  │
└───────────────────────────────┬──────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       ┌───────────┐     ┌───────────┐     ┌───────────┐
       │keep-local │     │keep-remote│     │ keep-both │
       └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
             │                 │                 │
             ▼                 ▼                 ▼
       ┌───────────┐     ┌───────────┐     ┌───────────┐
       │  Subir    │     │ Descargar │     │ Renombrar │
       │  local    │     │  remoto   │     │ + ambos   │
       └───────────┘     └───────────┘     └───────────┘
```

### 6.2 UI de Resolucion (GTK4 Example)

```
┌─────────────────────────────────────────────────────────────────┐
│  [!] Conflicto Detectado                                    [X] │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Archivo: documento.docx                                        │
│  Ruta: ~/OneDrive/Trabajo/documento.docx                        │
│                                                                 │
│  ┌─────────────────────────┬─────────────────────────────────┐  │
│  │     VERSION LOCAL       │       VERSION REMOTA            │  │
│  ├─────────────────────────┼─────────────────────────────────┤  │
│  │  Modificado: Hoy 15:30  │  Modificado: Hoy 15:45          │  │
│  │  Tamano: 245 KB         │  Tamano: 248 KB                 │  │
│  │  Por: (este dispositivo)│  Por: PC-Trabajo                │  │
│  │                         │                                 │  │
│  │  [Vista previa]         │  [Vista previa]                 │  │
│  └─────────────────────────┴─────────────────────────────────┘  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  [Comparar diferencias (abre Meld/diff)]                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌─ Resolucion ──────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  ( ) Mantener version local (sobrescribe remoto)          │  │
│  │  ( ) Mantener version remota (sobrescribe local)          │  │
│  │  (*) Mantener ambas (renombra local como _conflicto)      │  │
│  │                                                           │  │
│  │  [ ] Recordar esta decision para archivos .docx           │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    Cancelar     │  │   Resolver      │  │  Resolver Todos │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Configuracion de Politicas de Conflictos

```yaml
# ~/.config/lnxdrive/config.yaml
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
```

### 6.4 Estrategias de Resolucion

| Estrategia | Descripcion | Uso recomendado |
|------------|-------------|-----------------|
| `prompt` | Pregunta al usuario | Archivos importantes, codigo fuente |
| `keep-local` | Sobrescribe remoto con local | Desarrollo local prioritario |
| `keep-remote` | Sobrescribe local con remoto | Configuraciones compartidas |
| `keep-both` | Renombra local y conserva ambos | Documentos, archivos criticos |

### 6.5 Integracion con Herramientas de Diff

LNXDrive puede integrarse con herramientas externas de comparacion:

- **Meld** - Herramienta visual de diff para GNOME
- **KDiff3** - Herramienta de merge para KDE
- **diff** - Herramienta de linea de comandos
- **vimdiff** - Diff integrado en Vim

---

## Ver tambien

- [[04-Componentes/07-motor-sincronizacion|Motor de Sincronizacion]] - Estado de conflictos en el motor
- [[04-Componentes/12-auditoria|Auditoria]] - Registro de resoluciones de conflictos
- [[03-Arquitectura/03-adaptadores-entrada|Adaptadores de Entrada]] - UI intercambiables
