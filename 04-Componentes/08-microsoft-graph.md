# Integracion con Microsoft Graph

> **Ubicacion:** `04-Componentes/08-microsoft-graph.md`
> **Relacionado:** [[04-Componentes/07-motor-sincronizacion|Motor de Sincronizacion]], [[04-Componentes/09-rate-limiting|Rate Limiting]]

---

## Parte VIII: Integracion con Microsoft Graph

### 8.1 Modelo de Autenticacion

Siguiendo las recomendaciones de la investigacion, implementamos el **modelo hibrido**:

```yaml
# Flujo de autenticacion
authentication:
  # App ID por defecto (embebido en el codigo)
  default_app_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

  # Usuario puede especificar su propio App ID
  custom_app_id: null  # o UUID del usuario

  # Flujo: OAuth2 Authorization Code + PKCE
  flow: authorization_code_pkce

  # Sin client_secret (app publica)
  client_secret: null

  # Scopes minimos necesarios
  scopes:
    - "Files.ReadWrite"
    - "offline_access"
    - "User.Read"

  # Redirect para desktop app
  redirect_uri: "http://127.0.0.1:PORT/callback"

  # Token storage seguro
  token_storage:
    backend: keyring  # keyring | file | custom
    service: "lnxdrive"
```

### 8.2 Sincronizacion Eficiente con Delta API

```rust
// Pseudocodigo del flujo de delta sync
async fn sync_delta(&self) -> Result<()> {
    // 1. Obtener delta token del ultimo sync
    let delta_token = self.state_repo.get_delta_token().await?;

    // 2. Llamar a /drive/root/delta
    let response = self.graph_client
        .get("/me/drive/root/delta")
        .query(&[("token", delta_token.as_deref().unwrap_or("latest"))])
        .send()
        .await?;

    // 3. Procesar cada cambio
    for item in response.value {
        match item.deleted {
            Some(_) => self.handle_remote_delete(&item).await?,
            None => self.handle_remote_change(&item).await?,
        }
    }

    // 4. Guardar nuevo delta token
    if let Some(new_token) = response.delta_link {
        self.state_repo.save_delta_token(&new_token).await?;
    }

    // 5. Si hay mas paginas, continuar
    if let Some(next_link) = response.next_link {
        self.sync_delta_page(&next_link).await?;
    }

    Ok(())
}
```

---

## Ver tambien

- [[04-Componentes/07-motor-sincronizacion|Motor de Sincronizacion]] - Nucleo del sistema de sincronizacion
- [[04-Componentes/09-rate-limiting|Rate Limiting]] - Estrategias de throttling
- [[03-Arquitectura/04-adaptadores-salida|Adaptadores de Salida]] - Implementacion de MS Graph adapter
