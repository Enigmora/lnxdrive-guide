# Backlog: Simulation-Derived Issues

**Document ID:** BACKLOG-001
**Created:** 2026-01-31
**Status:** Draft
**Author:** claude-code-v1.0
**Total Issues:** 27

---

## Issue Template

```markdown
### ISSUE-XXX: [Title]

**Severity:** Critical | High | Medium | Low
**Component:** [Component name]
**Simulation:** SIM-XX-XXX
**Sprint:** X

#### Description
[Problem description]

#### Steps to Reproduce
1. [Step 1]
2. [Step 2]

#### Expected Behavior
[What should happen]

#### Actual Behavior
[What currently happens or would happen]

#### Proposed Fix
[Solution approach]

#### Test Case
[Test to verify fix]

#### Labels
[bug, security, race-condition, etc.]
```

---

## P0 - Critical Issues (Sprint 1-2)

### ISSUE-001: OAuth Tokens Visible in DBus Traffic

**Severity:** Critical
**Component:** DBus Auth Service
**Simulation:** SIM-L1-003
**Sprint:** 1

#### Description
OAuth access tokens and refresh tokens are transmitted in cleartext over DBus session messages. Any process in the same session can intercept these tokens using `dbus-monitor`.

#### Steps to Reproduce
1. Start lnxdrive daemon
2. Run `dbus-monitor --session "interface='org.enigmora.LNXDrive.Auth'"`
3. Trigger authentication flow
4. Observe captured messages

#### Expected Behavior
No tokens or credentials should be visible in DBus messages.

#### Actual Behavior
Access tokens, refresh tokens, and possibly client secrets are visible in message arguments.

#### Proposed Fix
1. Remove all token transmission over DBus
2. Implement opaque session handles
3. Store tokens in system keyring (GNOME Keyring/KWallet)
4. UI requests operations via session ID, daemon uses internal vault

#### Test Case
```rust
#[test]
fn test_no_tokens_in_dbus_messages() {
    let monitor = DbusCaptureMonitor::new();
    trigger_full_auth_flow();

    for message in monitor.captured() {
        assert!(!message.contains("eyJ"), "JWT token found in DBus");
        assert!(!message.contains("refresh_token"));
        assert!(!message.contains("access_token"));
    }
}
```

#### Labels
`security`, `critical`, `dbus`, `oauth`

---

### ISSUE-002: YAML Configuration Billion Laughs DoS

**Severity:** Critical
**Component:** Configuration Parser
**Simulation:** SIM-L4-003
**Sprint:** 1

#### Description
YAML parser accepts recursive anchor references that can cause exponential memory expansion, leading to denial of service.

#### Steps to Reproduce
1. Create config with recursive anchors:
```yaml
lnxdrive:
  a: &a ["x","x","x","x","x"]
  b: &b [*a,*a,*a,*a,*a]
  c: &c [*b,*b,*b,*b,*b]
  d: &d [*c,*c,*c,*c,*c]
```
2. Start daemon with this config
3. Observe memory exhaustion

#### Expected Behavior
Config parsing should fail fast with clear error about expansion limit.

#### Actual Behavior
Parser attempts full expansion, consuming all available memory.

#### Proposed Fix
1. Use `serde_yaml` with anchor/alias limits
2. Implement maximum expansion size check
3. Add JSON Schema validation before parsing
4. Validate all string fields for path traversal

#### Test Case
```rust
#[test]
fn test_billion_laughs_rejected() {
    let malicious = include_str!("fixtures/billion_laughs.yaml");
    let result = load_config_from_str(malicious);
    assert!(matches!(result, Err(ConfigError::ExpansionLimitExceeded)));
}
```

#### Labels
`security`, `critical`, `config`, `dos`

---

### ISSUE-003: FUSE Write During Hydration Causes Corruption

**Severity:** Critical
**Component:** FUSE Handler + HydrationManager
**Simulation:** SIM-L2-002
**Sprint:** 1

#### Description
When a file is being hydrated (downloaded from cloud), concurrent write operations can proceed without blocking, causing file corruption.

#### Steps to Reproduce
1. Have a large file (>50MB) in Online (placeholder) state
2. Process A opens file for reading (triggers hydration)
3. While hydrating, Process B writes to the same file
4. Hydration completes
5. File content is corrupted (mix of remote + local data)

#### Expected Behavior
Write operations should be blocked with EBUSY while hydration is in progress, OR hydration should be cancelled and file marked as Modified.

#### Actual Behavior
Write proceeds concurrently with hydration, causing data corruption.

#### Proposed Fix
```rust
impl FuseHandler {
    fn write(&self, path: &Path, ...) -> Result<usize> {
        let state = self.state_manager.get(path)?;

        if state == ItemState::Hydrating {
            // Option 1: Block
            return Err(libc::EBUSY);

            // Option 2: Wait
            // self.hydration_manager.wait_for_completion(path)?;
        }

        self.do_write(path, data, offset)
    }
}
```

#### Test Case
```rust
#[test]
fn test_write_blocked_during_hydration() {
    let file = create_large_placeholder("test.bin", 100_MB);

    let hydration = spawn(|| open_and_read(&file));
    sleep(Duration::from_millis(100)); // Ensure hydration started

    let result = write_to_file(&file, b"data");
    assert_eq!(result.unwrap_err().raw_os_error(), Some(libc::EBUSY));
}
```

#### Labels
`bug`, `critical`, `race-condition`, `fuse`, `data-integrity`

---

### ISSUE-004: SQLite Concurrent Access Race Condition

**Severity:** Critical
**Component:** State Repository + FUSE
**Simulation:** SIM-L3-001
**Sprint:** 1

#### Description
SQLite only supports single writer. When FUSE operations and sync engine compete for writes, SQLITE_BUSY errors occur, potentially leaving state inconsistent.

#### Steps to Reproduce
1. Start daemon with active sync
2. Perform many FUSE operations (1000 getattr/s)
3. Sync engine writes state updates
4. Observe SQLITE_BUSY errors
5. Force crash during this state
6. On restart, state may be inconsistent

#### Expected Behavior
All writes should be serialized through a single writer with proper transaction handling.

#### Actual Behavior
Concurrent writes cause SQLITE_BUSY, and transaction rollbacks may leave xattrs inconsistent with database.

#### Proposed Fix
```rust
impl StateRepository {
    fn new() -> Self {
        // Single writer connection
        let writer = Connection::open("state.db")?;
        writer.execute("PRAGMA journal_mode=WAL")?;

        // Multiple reader connections
        let reader_pool = Pool::new(4, || Connection::open("state.db"))?;

        Self { writer: Mutex::new(writer), readers: reader_pool }
    }

    fn write(&self, f: impl FnOnce(&Connection)) -> Result<()> {
        let conn = self.writer.lock();
        let tx = conn.transaction()?;
        f(&tx)?;
        tx.commit()
    }
}
```

#### Test Case
```rust
#[test]
fn test_concurrent_fuse_and_sync_writes() {
    let state = StateRepository::new();

    let writers: Vec<_> = (0..10).map(|i| {
        let state = state.clone();
        spawn(move || {
            for j in 0..1000 {
                state.update_item(&format!("file_{i}_{j}"), ItemState::Modified)?;
            }
            Ok(())
        })
    }).collect();

    for w in writers {
        w.join().unwrap().unwrap(); // All should succeed
    }
}
```

#### Labels
`bug`, `critical`, `race-condition`, `sqlite`, `data-integrity`

---

### ISSUE-005: State Machine Has No Exit From Error State

**Severity:** Critical
**Component:** State Machine
**Simulation:** SIM-L2-001
**Sprint:** 2

#### Description
Files that enter the Error state have no defined recovery transitions. They remain stuck indefinitely and cannot be accessed or synced.

#### Steps to Reproduce
1. Start hydration of a file
2. Disconnect network during hydration
3. File enters Error(NetworkTimeout) state
4. Reconnect network
5. File remains in Error state forever

#### Expected Behavior
Error state should have recovery transitions:
- Error → Hydrating (retry)
- Error → Online (reset)
- Error → Conflicted (escalate to user)

#### Actual Behavior
Error is a terminal state with no exits.

#### Proposed Fix
```rust
impl StateMachine {
    fn get_valid_transitions(&self, from: ItemState) -> Vec<ItemState> {
        match from {
            ItemState::Error(ctx) => match ctx {
                ErrorContext::Hydration => vec![
                    ItemState::Hydrating, // retry
                    ItemState::Online,    // reset
                    ItemState::Conflicted // escalate
                ],
                ErrorContext::Upload => vec![
                    ItemState::Modified,  // retry
                    ItemState::Conflicted // escalate
                ],
                // ...
            },
            // other states...
        }
    }
}
```

#### Test Case
```rust
#[test]
fn test_error_recovery_transitions() {
    let sm = StateMachine::new();

    let item = create_item_in_state(ItemState::Error(ErrorContext::Hydration));

    // Should allow retry
    assert!(sm.can_transition(&item, ItemState::Hydrating));

    // Should allow reset
    assert!(sm.can_transition(&item, ItemState::Online));

    // Should allow escalate
    assert!(sm.can_transition(&item, ItemState::Conflicted));
}
```

#### Labels
`bug`, `critical`, `state-machine`, `deadlock`

---

## P1 - High Priority Issues (Sprint 3-4)

### ISSUE-006: DBus Single Point of Failure

**Severity:** High
**Component:** DBus Service
**Simulation:** SIM-L1-001
**Sprint:** 3

#### Description
All UI adapters depend exclusively on DBus. If DBus session crashes, all UIs lose visibility and control of synchronization.

#### Proposed Fix
1. Implement DBus health monitoring
2. Add automatic reconnection with exponential backoff
3. Continue sync operations even without DBus
4. Queue notifications for delivery after reconnection

#### Labels
`enhancement`, `high`, `dbus`, `availability`

---

### ISSUE-007: Delta Token Expiration Not Handled

**Severity:** High
**Component:** MS Graph Client
**Simulation:** SIM-L3-002
**Sprint:** 3

#### Description
When Microsoft Graph returns 410 Gone for expired delta token, the system has no recovery mechanism, causing sync to fail repeatedly.

#### Proposed Fix
1. Detect 410 Gone response
2. Clear expired token
3. Initiate full enumeration with progress indication
4. Notify user of resync operation
5. Implement proactive token refresh before expiration

#### Labels
`bug`, `high`, `graph`, `sync`

---

### ISSUE-008: No Offline Mode Support

**Severity:** High
**Component:** Sync Engine
**Simulation:** SIM-L3-003
**Sprint:** 3

#### Description
When network is unavailable, the system fails to provide graceful degradation. Hydrated files should remain accessible and changes should be queued.

#### Proposed Fix
1. Detect network unavailability
2. Allow access to Hydrated files
3. Queue local modifications
4. Automatically sync when connection restored
5. Notify user of offline status

#### Labels
`enhancement`, `high`, `sync`, `offline`

---

### ISSUE-009: inotify IN_Q_OVERFLOW Not Handled

**Severity:** High
**Component:** File Watcher
**Simulation:** SIM-L3-004
**Sprint:** 3

#### Description
When inotify queue overflows during bulk operations, events are silently lost, causing files to not sync.

#### Proposed Fix
1. Handle IN_Q_OVERFLOW event
2. Log warning with metrics
3. Mark all watches as stale
4. Trigger full filesystem scan
5. Emit user notification

#### Labels
`bug`, `high`, `inotify`, `data-integrity`

---

### ISSUE-010: Plugin Execution Without Sandboxing

**Severity:** High
**Component:** Plugin System
**Simulation:** SIM-L4-002
**Sprint:** 4

#### Description
Cloud provider plugins execute with full daemon privileges, allowing malicious plugins to access all credentials and files.

#### Proposed Fix
1. Only allow built-in providers (no dynamic loading) initially
2. Long-term: Process isolation with IPC
3. Seccomp/Landlock for syscall filtering
4. Network namespace isolation
5. Capability-based security model

#### Labels
`security`, `high`, `plugin`, `sandbox`

---

### ISSUE-011: Observer Callbacks Can Block Sync

**Severity:** High
**Component:** Observer Pattern
**Simulation:** SIM-L4-006
**Sprint:** 4

#### Description
If an observer callback blocks (e.g., frozen UI), the sync engine is blocked waiting for the callback to return.

#### Proposed Fix
1. Wrap callbacks with timeout
2. Execute callbacks in separate thread pool
3. Detect and log slow observers
4. Implement circuit breaker for problematic observers

#### Labels
`bug`, `high`, `observer`, `performance`

---

## P2 - Medium Priority Issues (Sprint 4-5)

### ISSUE-012: DBus API Rate Limiting Missing

**Severity:** Medium
**Component:** DBus Service
**Simulation:** SIM-L1-002
**Sprint:** 4

#### Description
No rate limiting on DBus calls allows denial of service by flooding the service with requests.

#### Proposed Fix
Per-caller rate limiting with token bucket algorithm.

#### Labels
`security`, `medium`, `dbus`, `dos`

---

### ISSUE-013: Dehydration Race With Concurrent Read

**Severity:** Medium
**Component:** FUSE Dehydration
**Simulation:** SIM-L2-003
**Sprint:** 4

#### Description
Automatic dehydration can conflict with concurrent read operations.

#### Proposed Fix
Track active readers and postpone dehydration until readers complete.

#### Labels
`bug`, `medium`, `race-condition`, `fuse`

---

### ISSUE-014: Conflict Resolution Version Race

**Severity:** Medium
**Component:** Conflict Resolver
**Simulation:** SIM-L2-004
**Sprint:** 4

#### Description
New remote versions arriving during conflict resolution may be lost.

#### Proposed Fix
Version check before applying resolution, with user notification if changed.

#### Labels
`bug`, `medium`, `race-condition`, `conflict`

---

### ISSUE-015: WAL Checkpoint Policy Undefined

**Severity:** Medium
**Component:** SQLite
**Simulation:** SIM-L3-005
**Sprint:** 5

#### Description
No defined policy for WAL checkpoints can lead to unbounded WAL growth or corruption on crash.

#### Proposed Fix
Implement size-based + time-based checkpoint triggers with integrity checks.

#### Labels
`enhancement`, `medium`, `sqlite`, `reliability`

---

### ISSUE-016: Rate Limiter Token Bucket Race

**Severity:** Medium
**Component:** Rate Limiter
**Simulation:** SIM-L3-006
**Sprint:** 5

#### Description
Concurrent token requests during refill may cause over-allocation.

#### Proposed Fix
Use atomic operations for token bucket management.

#### Labels
`bug`, `medium`, `race-condition`, `rate-limit`

---

## P3 - Low Priority Issues (Sprint 5+)

### ISSUE-017: Resource Leaks in Long-Running Sessions

**Severity:** Low
**Component:** Core
**Simulation:** SIM-L2-005
**Sprint:** 5

#### Description
WatchHandle and SyncSession lifecycle not properly managed may cause memory/FD leaks.

#### Proposed Fix
Implement reference counting and periodic leak detection.

#### Labels
`bug`, `low`, `memory-leak`, `reliability`

---

### ISSUE-018: ICloudProvider Missing Batch Operations

**Severity:** Low
**Component:** Provider Interface
**Simulation:** SIM-L4-001
**Sprint:** 5

#### Description
No batch operations in ICloudProvider makes bulk operations inefficient.

#### Proposed Fix
Add batch_delete, batch_move, batch_metadata methods.

#### Labels
`enhancement`, `low`, `provider`, `performance`

---

### ISSUE-019: DBus API No Versioning

**Severity:** Low
**Component:** DBus Service
**Simulation:** SIM-L4-004
**Sprint:** 6

#### Description
No version in DBus interface makes backward compatibility impossible.

#### Proposed Fix
Add versioned interfaces (org.enigmora.LNXDrive.Manager.1, etc.)

#### Labels
`enhancement`, `low`, `dbus`, `compatibility`

---

### ISSUE-020: Conflict File Naming Collision

**Severity:** Low
**Component:** Conflict Resolver
**Simulation:** SIM-L4-005
**Sprint:** 6

#### Description
Conflict file naming (_conflict suffix) can collide with existing files.

#### Proposed Fix
Use timestamp + UUID in conflict file names.

#### Labels
`bug`, `low`, `conflict`, `naming`

---

### ISSUE-021: No Configuration Migration Path

**Severity:** Low
**Component:** Configuration
**Simulation:** SIM-L4-007
**Sprint:** 6

#### Description
No migration system for configuration versions.

#### Proposed Fix
Implement versioned config with migration scripts.

#### Labels
`enhancement`, `low`, `config`, `upgrade`

---

## Summary

| Priority | Count | Sprint |
|----------|-------|--------|
| P0 Critical | 5 | 1-2 |
| P1 High | 6 | 3-4 |
| P2 Medium | 5 | 4-5 |
| P3 Low | 5 | 5+ |
| **Total** | **21** | - |

---

*Document generated by architectural simulation analysis*
*DevTrail Framework | https://enigmora.com*
