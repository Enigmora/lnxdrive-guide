# RISK-002: Security Vulnerabilities Analysis

**Document ID:** RISK-002
**Created:** 2026-01-31
**Status:** Draft
**Author:** claude-code-v1.0
**Risk Level:** Critical

---

## Executive Summary

This document identifies security vulnerabilities in the LNXDrive architecture design. Each vulnerability is classified using CVSS-like severity and includes detailed remediation strategies.

---

## 1. Vulnerability Inventory

### 1.1 VULN-001: OAuth Token Exposure via DBus

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL (9.1) |
| **Component** | DBus Service API |
| **Attack Vector** | Local |
| **Privileges Required** | Low (same session) |
| **Impact** | Account Takeover |

**Description:**
OAuth access tokens and refresh tokens may be transmitted in cleartext over DBus messages. Any process in the same DBus session can intercept these tokens using `dbus-monitor` or similar tools, enabling account takeover.

**Vulnerable Flow:**
```
UI Adapter                      Daemon
    │                              │
    │──── GetTokens() ────────────→│
    │                              │
    │←─── {access_token: "eyJ..."} │  ← CLEARTEXT!
    │      refresh_token: "..."    │
    │                              │
```

**Proof of Concept:**
```bash
dbus-monitor --session "interface='org.enigmora.LNXDrive.Auth'"
# Tokens visible in output
```

**Impact:**
- Full access to user's cloud storage
- Ability to read/modify/delete all files
- Persistent access via refresh token
- Potential lateral movement to other services

**Remediation:**

**Short-term (P0):**
1. Never transmit tokens over DBus
2. Use opaque session handles instead of tokens
3. Implement token vault with secure IPC

**Long-term:**
1. Integrate with system keyring (GNOME Keyring, KWallet)
2. Implement credential isolation per UI adapter
3. Add token usage auditing

**Secure Design:**
```
UI Adapter                      Daemon
    │                              │
    │──── CreateSession() ────────→│
    │                              │
    │←─── {session_id: "abc123"}   │  ← Opaque handle
    │                              │
    │──── CallAPI(session_id) ────→│
    │                              │  Daemon uses internal
    │←─── {result: ...}            │  token vault
```

**Simulation Scenario:** SIM-L1-003

---

### 1.2 VULN-002: DBus API Without Authentication

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH (7.5) |
| **Component** | DBus Service |
| **Attack Vector** | Local |
| **Privileges Required** | Low |
| **Impact** | Unauthorized Operations |

**Description:**
The DBus API does not implement explicit authentication. Any process in the same user session can call any method, including sensitive operations like deleting sync configuration or forcing re-authentication.

**Vulnerable Endpoints:**
```xml
<interface name="org.enigmora.LNXDrive.Manager">
  <!-- No auth required -->
  <method name="RemoveAccount">
    <arg type="s" name="namespace"/>
  </method>
  <method name="ResetSync"/>
  <method name="ClearCache"/>
</interface>
```

**Attack Scenario:**
```bash
# Malicious script removes all accounts
dbus-send --session --dest=org.enigmora.LNXDrive \
  /org/enigmora/LNXDrive/Manager \
  org.enigmora.LNXDrive.Manager.ResetSync
```

**Impact:**
- Account configuration deletion
- Forced re-authentication
- Data loss from cache clearing
- Denial of service

**Remediation:**

**Short-term (P0):**
1. Implement PolicyKit integration for sensitive operations
2. Require user confirmation for destructive operations
3. Add caller identification logging

**Implementation:**
```xml
<interface name="org.enigmora.LNXDrive.Manager">
  <method name="RemoveAccount">
    <arg type="s" name="namespace"/>
    <annotation name="org.freedesktop.DBus.Deprecated" value="true"/>
  </method>

  <!-- New secure version -->
  <method name="RemoveAccountSecure">
    <arg type="s" name="namespace"/>
    <arg type="s" name="confirmation_token"/>
    <annotation name="org.freedesktop.policykit1.action"
                value="org.enigmora.lnxdrive.remove-account"/>
  </method>
</interface>
```

**Simulation Scenario:** SIM-L1-002

---

### 1.3 VULN-003: DBus Denial of Service

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM (5.3) |
| **Component** | DBus Service |
| **Attack Vector** | Local |
| **Privileges Required** | Low |
| **Impact** | Service Degradation |

**Description:**
The DBus service does not implement rate limiting. A malicious process can flood the service with requests, causing CPU exhaustion and making the daemon unresponsive to legitimate clients.

**Attack Vector:**
```bash
# Flood the service with 10,000 requests/second
while true; do
  dbus-send --session --dest=org.enigmora.LNXDrive \
    /org/enigmora/LNXDrive/Sync \
    org.enigmora.LNXDrive.Sync.GetStatus &
done
```

**Impact:**
- Daemon CPU at 100%
- Legitimate UI clients timeout
- Sync operations delayed
- System-wide responsiveness degradation

**Remediation:**

**Short-term (P1):**
1. Implement per-caller rate limiting
2. Add request prioritization (UI clients > unknown callers)
3. Implement circuit breaker for abusive callers

**Implementation:**
```rust
struct RateLimiter {
    limits: HashMap<String, TokenBucket>,  // per sender
    default_limit: u32,  // 100 req/s
}

impl RateLimiter {
    fn check(&mut self, sender: &str) -> Result<(), RateLimitError> {
        let bucket = self.limits.entry(sender.to_string())
            .or_insert_with(|| TokenBucket::new(self.default_limit));

        if !bucket.try_acquire() {
            return Err(RateLimitError::Exceeded);
        }
        Ok(())
    }
}
```

**Simulation Scenario:** SIM-L1-002

---

### 1.4 VULN-004: Plugin Code Execution Without Sandboxing

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL (9.8) |
| **Component** | Provider Plugin System |
| **Attack Vector** | Local |
| **Privileges Required** | None (if plugin is installed) |
| **Impact** | Arbitrary Code Execution |

**Description:**
The plugin system for cloud providers loads code without sandboxing. A malicious or compromised plugin can execute arbitrary code with full daemon privileges, access all credentials, and read/modify any synced files.

**Attack Surface:**
```rust
// Current: Plugins run with full privileges
trait ICloudProvider {
    // Plugin can do ANYTHING here
    async fn authenticate(&self, ...) -> Result<Tokens>;
}

// Malicious plugin example
impl ICloudProvider for MaliciousProvider {
    async fn authenticate(&self, ...) -> Result<Tokens> {
        // Steal other provider's tokens
        let other_tokens = std::fs::read("/path/to/token/vault")?;
        send_to_attacker(&other_tokens);

        // Exfiltrate user files
        exfiltrate_directory("/home/user/Documents")?;

        Ok(fake_tokens())
    }
}
```

**Impact:**
- Full system compromise
- Credential theft for all accounts
- Data exfiltration
- Ransomware deployment
- Lateral movement

**Remediation:**

**Short-term (P0):**
1. Only allow built-in providers (compile-time, no dynamic loading)
2. Require code signing for any external plugins
3. Implement strict permission model

**Long-term (P1):**
1. Run plugins in isolated processes
2. Implement capability-based security
3. Use seccomp/landlock for syscall filtering
4. Network namespace isolation per plugin

**Sandboxing Architecture:**
```
┌─────────────────────────────────────────────┐
│              lnxdrive-daemon                │
├─────────────────────────────────────────────┤
│              Plugin Manager                  │
└─────────────────┬───────────────────────────┘
                  │ IPC (Unix socket)
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼───┐    ┌───▼───┐    ┌───▼───┐
│sandbox│    │sandbox│    │sandbox│
│OneDrv │    │GDrive │    │Dropbox│
│seccomp│    │seccomp│    │seccomp│
│netns  │    │netns  │    │netns  │
└───────┘    └───────┘    └───────┘
```

**Simulation Scenario:** SIM-L4-002

---

### 1.5 VULN-005: YAML Configuration Injection

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH (7.8) |
| **Component** | Configuration Parser |
| **Attack Vector** | Local |
| **Privileges Required** | Low (write to config) |
| **Impact** | Arbitrary File Access, DoS |

**Description:**
The YAML configuration parser does not validate input thoroughly. Attackers with write access to the config file can exploit YAML features or inject malicious values.

**Attack Vectors:**

**A) Billion Laughs (DoS):**
```yaml
lnxdrive:
  a: &a ["lol","lol","lol","lol","lol"]
  b: &b [*a,*a,*a,*a,*a]
  c: &c [*b,*b,*b,*b,*b]
  d: &d [*c,*c,*c,*c,*c]
  e: &e [*d,*d,*d,*d,*d]
  # Exponential expansion → memory exhaustion
```

**B) Path Traversal:**
```yaml
lnxdrive:
  sync:
    root: "../../../etc/passwd"
    cache_dir: "/etc/shadow"
```

**C) Extreme Values:**
```yaml
lnxdrive:
  sync:
    cache_limit: 999999999TB
    max_file_size: -1
    threads: 999999
```

**Impact:**
- Memory exhaustion (DoS)
- Reading sensitive system files
- Writing to arbitrary locations
- Resource exhaustion

**Remediation:**

**Short-term (P0):**
1. Disable YAML anchors/aliases (use safe_load equivalent)
2. Validate paths are within allowed directories
3. Validate numeric ranges
4. Implement JSON Schema validation

**Implementation:**
```rust
fn load_config(path: &Path) -> Result<Config> {
    let raw = std::fs::read_to_string(path)?;

    // Limit YAML parsing to prevent DoS
    let yaml = serde_yaml::from_str_with_limit(&raw, MAX_YAML_SIZE)?;

    // Validate against JSON Schema
    validate_schema(&yaml, SCHEMA)?;

    // Validate paths
    let config: Config = serde_yaml::from_value(yaml)?;
    validate_paths(&config)?;
    validate_numeric_ranges(&config)?;

    Ok(config)
}

fn validate_paths(config: &Config) -> Result<()> {
    let sync_root = config.sync.root.canonicalize()?;

    // Must be under home directory
    if !sync_root.starts_with(dirs::home_dir().unwrap()) {
        return Err(ConfigError::InvalidPath("sync root must be under home"));
    }

    // No path traversal
    if sync_root.to_string_lossy().contains("..") {
        return Err(ConfigError::PathTraversal);
    }

    Ok(())
}
```

**Simulation Scenario:** SIM-L4-003

---

## 2. Security Architecture Gaps

### 2.1 Missing Security Controls

| Control | Status | Priority |
|---------|--------|----------|
| Input validation | Partial | P0 |
| Authentication (DBus) | Missing | P0 |
| Authorization (RBAC) | Missing | P1 |
| Credential storage | Undocumented | P0 |
| Audit logging | Partial | P1 |
| Encryption at rest | Not specified | P2 |
| Secure IPC | Missing | P0 |

### 2.2 Credential Storage Recommendations

```
┌─────────────────────────────────────────────┐
│           Credential Storage                │
├─────────────────────────────────────────────┤
│                                             │
│  Option 1: System Keyring                   │
│  ├── GNOME Keyring (libsecret)              │
│  ├── KWallet (kwallet5)                     │
│  └── SecretService D-Bus API                │
│                                             │
│  Option 2: Encrypted File                   │
│  ├── Key derived from user password         │
│  ├── AES-256-GCM encryption                 │
│  └── Memory-protected operations            │
│                                             │
│  NEVER: Plaintext, Weak encoding            │
└─────────────────────────────────────────────┘
```

---

## 3. Attack Surface Summary

```
                         ATTACK SURFACE
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼────┐          ┌─────▼─────┐         ┌─────▼─────┐
   │ Network │          │  Local    │         │  Config   │
   │         │          │  Process  │         │   Files   │
   └────┬────┘          └─────┬─────┘         └─────┬─────┘
        │                     │                     │
   ┌────▼────┐          ┌─────▼─────┐         ┌─────▼─────┐
   │MS Graph │          │   DBus    │         │   YAML    │
   │ API     │          │  Service  │         │  Parser   │
   │         │          │           │         │           │
   │• OAuth  │          │• No auth  │         │• Billion  │
   │  tokens │          │• No rate  │         │  laughs   │
   │• Delta  │          │  limiting │         │• Path     │
   │  token  │          │• Token    │         │  traversal│
   │         │          │  exposure │         │           │
   └─────────┘          └───────────┘         └───────────┘
```

---

## 4. Remediation Roadmap

### Phase 1: Critical (Sprint 1-2)
1. [ ] Remove token transmission over DBus (VULN-001)
2. [ ] Implement YAML safe parsing (VULN-005)
3. [ ] Disable dynamic plugin loading (VULN-004)
4. [ ] Add path validation for config (VULN-005)

### Phase 2: High (Sprint 3-4)
1. [ ] Implement PolicyKit for DBus (VULN-002)
2. [ ] Add DBus rate limiting (VULN-003)
3. [ ] Integrate with system keyring
4. [ ] Implement audit logging

### Phase 3: Hardening (Sprint 5+)
1. [ ] Plugin sandboxing architecture
2. [ ] Seccomp profiles
3. [ ] Security scanning in CI/CD
4. [ ] Penetration testing

---

## 5. Security Testing Requirements

### 5.1 Automated Tests
```rust
#[test]
fn test_no_tokens_in_dbus_messages() {
    let monitor = DbusCaptureMonitor::new();
    trigger_auth_flow();

    for message in monitor.captured() {
        assert!(!message.contains("eyJ"));  // JWT prefix
        assert!(!message.contains("refresh_token"));
    }
}

#[test]
fn test_yaml_billion_laughs_rejected() {
    let malicious = include_str!("fixtures/billion_laughs.yaml");
    let result = load_config_from_str(malicious);
    assert!(matches!(result, Err(ConfigError::YamlExpansionLimit)));
}

#[test]
fn test_path_traversal_rejected() {
    let config = r#"
        lnxdrive:
          sync:
            root: "../../../etc/passwd"
    "#;
    let result = load_config_from_str(config);
    assert!(matches!(result, Err(ConfigError::PathTraversal)));
}
```

### 5.2 Manual Testing Checklist
- [ ] dbus-monitor token capture attempt
- [ ] PolicyKit bypass attempts
- [ ] Config injection fuzzing
- [ ] Plugin isolation verification

---

## 6. Related Documents

- RISK-001: Critical Paths (SPOF analysis)
- RISK-003: Data Integrity Risks
- SIM-L1-002, SIM-L1-003, SIM-L4-002, SIM-L4-003: Security Simulations

---

*Document generated by architectural simulation analysis*
*DevTrail Framework | https://enigmora.com*
