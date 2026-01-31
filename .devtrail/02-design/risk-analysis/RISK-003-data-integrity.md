# RISK-003: Data Integrity Risks Analysis

**Document ID:** RISK-003
**Created:** 2026-01-31
**Status:** Draft
**Author:** claude-code-v1.0
**Risk Level:** Critical

---

## Executive Summary

This document identifies data integrity risks in the LNXDrive architecture. Data integrity failures can result in file corruption, data loss, or synchronization inconsistencies that may be undetectable until significant damage has occurred.

---

## 1. Race Condition Risks

### 1.1 RACE-001: FUSE Write During Hydration

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **Probability** | Medium |
| **Impact** | File Corruption |
| **Component** | FUSE + HydrationManager |

**Description:**
When a file is being hydrated (downloaded from cloud), concurrent write operations can corrupt the file by mixing remote content with local modifications.

**Race Condition Timeline:**
```
Thread A (Hydration)           Thread B (Application Write)
        │                              │
   t1   │ open() triggers hydration    │
        │                              │
   t2   │ Download chunk 1 (0-1MB)     │
        │                              │
   t3   │ Download chunk 2 (1-2MB)     │ write() at offset 1.5MB
        │                              │      ↓
   t4   │ Download chunk 3 (2-3MB)     │ RACE! Write to partial file
        │                              │
   t5   │ Hydration complete           │
        │                              │
        │  FILE CORRUPTED              │
        │  (chunk 3 overwrote write)   │
```

**Impact:**
- Silent data corruption
- User modifications lost
- File becomes unusable
- No recovery without backup

**Current State:** Not handled in documentation

**Mitigation Strategy:**

**Option A: Exclusive Lock During Hydration**
```rust
impl FuseHandler {
    fn write(&self, path: &Path, data: &[u8], offset: u64) -> Result<usize> {
        let state = self.state_manager.get(path)?;

        match state {
            ItemState::Hydrating => {
                // Block write until hydration complete
                return Err(libc::EBUSY);
            }
            ItemState::Online => {
                // Trigger hydration first, then allow write
                self.hydration_manager.hydrate_sync(path)?;
            }
            _ => {}
        }

        self.do_write(path, data, offset)
    }
}
```

**Option B: Copy-on-Write with Conflict Detection**
```rust
impl FuseHandler {
    fn write(&self, path: &Path, data: &[u8], offset: u64) -> Result<usize> {
        let state = self.state_manager.get(path)?;

        if state == ItemState::Hydrating {
            // Create shadow copy for local modifications
            let shadow_path = self.create_shadow_copy(path)?;
            self.mark_as_conflict(path, ConflictType::WriteWhileHydrating)?;
            return self.do_write(&shadow_path, data, offset);
        }

        self.do_write(path, data, offset)
    }
}
```

**Recommended:** Option A (simpler, prevents corruption)

**Simulation Scenario:** SIM-L2-002

---

### 1.2 RACE-002: Dehydration During Read

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **Probability** | Low |
| **Impact** | Read Failure / Partial Data |
| **Component** | FUSE + DehydrationManager |

**Description:**
When the system automatically dehydrates files to reclaim disk space, a concurrent read operation may receive partial or no data.

**Race Condition Timeline:**
```
Thread A (Dehydration)         Thread B (Application Read)
        │                              │
   t1   │ Select file for dehydration  │
        │                              │
   t2   │ Start truncating content     │ open()
        │                              │      ↓
   t3   │ Remove bytes 0-50%           │ read() at offset 0
        │                              │      ↓
   t4   │                              │ Returns partial/empty!
        │                              │
   t5   │ Replace with placeholder     │
```

**Impact:**
- Application receives incomplete data
- Application may crash or corrupt its own state
- User sees truncated content

**Mitigation Strategy:**
```rust
impl DehydrationManager {
    fn dehydrate(&self, path: &Path) -> Result<()> {
        // Acquire exclusive lock
        let lock = self.locks.acquire_exclusive(path)?;

        // Check for active readers
        let readers = self.fuse_handler.active_readers(path);
        if readers > 0 {
            // Postpone dehydration
            self.schedule_retry(path, Duration::from_secs(60));
            return Ok(());
        }

        // Mark as dehydrating (blocks new readers)
        self.state_manager.set(path, ItemState::Dehydrating)?;

        // Perform dehydration
        self.do_dehydrate(path)?;

        // Mark as online
        self.state_manager.set(path, ItemState::Online)?;

        Ok(())
    }
}
```

**Simulation Scenario:** SIM-L2-003

---

### 1.3 RACE-003: Conflict Resolution During Sync

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **Probability** | Medium |
| **Impact** | Wrong Version Applied |
| **Component** | ConflictResolver + SyncEngine |

**Description:**
When a user resolves a conflict while synchronization is actively running, the resolution may be applied to the wrong version if a new remote change arrives during resolution.

**Race Condition Timeline:**
```
User (Conflict Dialog)        SyncEngine              Remote
        │                         │                      │
   t1   │ View conflict v1↔v2     │                      │
        │                         │                      │
   t2   │ User selects "keep v2"  │     Delta arrives    │
        │                         │←─────(file v3)───────│
        │                         │                      │
   t3   │ Confirm resolution      │ Process v3           │
        │         ↓               │                      │
   t4   │ Apply "keep v2"         │                      │
        │                         │                      │
        │  RESULT: v3 is lost!    │                      │
        │  User thought v2 was    │                      │
        │  latest, but v3 existed │                      │
```

**Impact:**
- Latest version silently discarded
- User unaware of lost changes
- Data loss

**Mitigation Strategy:**
```rust
impl ConflictResolver {
    fn resolve(&self, conflict_id: &str, resolution: Resolution) -> Result<()> {
        // Load conflict with version lock
        let conflict = self.conflicts.get_with_lock(conflict_id)?;

        // Check if conflict is still valid (no new versions)
        let current_remote = self.cloud.get_metadata(&conflict.remote_id)?;

        if current_remote.version != conflict.remote_version {
            // New version arrived during resolution
            self.conflicts.update(conflict_id, |c| {
                c.remote_version = current_remote.version;
                c.remote_hash = current_remote.hash;
                c.status = ConflictStatus::Updated;
            })?;

            // Notify user
            return Err(ConflictError::VersionChanged {
                old: conflict.remote_version,
                new: current_remote.version,
            });
        }

        // Safe to apply resolution
        self.apply_resolution(conflict, resolution)?;

        Ok(())
    }
}
```

**Simulation Scenario:** SIM-L2-004

---

### 1.4 RACE-004: inotify Eviction During Event Processing

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **Probability** | Medium |
| **Impact** | Missed File Changes |
| **Component** | FileWatcher (inotify) |

**Description:**
When the watch manager evicts watches due to inotify limits, pending events for those paths may be lost if the event is in the kernel queue but not yet processed.

**Race Condition Timeline:**
```
WatchManager                   Kernel inotify          Application
      │                             │                       │
 t1   │ evict_lru(path_A)           │                       │
      │         ↓                   │                       │
 t2   │ inotify_rm_watch()          │  Event for A pending  │
      │                             │                       │
 t3   │                             │  Deliver event A      │
      │                             │  (no handler!)        │
      │                             │         ↓             │ modify(path_A)
 t4   │                             │  Event lost           │
      │                             │                       │
 t5   │ Add path_A to polling       │                       │
      │                             │                       │
      │  RESULT: Modification at t4 │                       │
      │  was never detected         │                       │
```

**Impact:**
- File modifications not synced
- User believes file is synced when it's not
- Silent data divergence

**Mitigation Strategy:**
```rust
impl WatchManager {
    fn evict_watch(&mut self, path: &Path) -> Result<()> {
        // Drain pending events first
        self.drain_events_for(path)?;

        // Record eviction timestamp
        let eviction_time = SystemTime::now();

        // Remove from inotify
        self.inotify.rm_watch(path)?;

        // Add to polling with eviction context
        self.polling_paths.insert(path.to_owned(), PollingEntry {
            last_check: eviction_time,
            needs_immediate_check: true,  // Check immediately after eviction
        });

        Ok(())
    }

    fn drain_events_for(&mut self, path: &Path) -> Result<()> {
        // Non-blocking read of pending events
        let events = self.inotify.read_events_nonblocking()?;

        for event in events {
            if event.path == path {
                // Process event before eviction
                self.process_event(event)?;
            } else {
                // Re-queue for later processing
                self.event_queue.push(event);
            }
        }

        Ok(())
    }
}
```

**Simulation Scenario:** SIM-L3-004

---

### 1.5 RACE-005: Rate Limiter Token Allocation

| Attribute | Value |
|-----------|-------|
| **Severity** | LOW |
| **Probability** | Medium |
| **Impact** | API Throttling Errors |
| **Component** | AdaptiveRateLimiter |

**Description:**
When multiple threads request tokens while a refill occurs, the bucket may over-allocate tokens, causing more requests than allowed and triggering API throttling.

**Race Condition:**
```rust
// Problematic implementation
impl TokenBucket {
    fn try_acquire(&mut self) -> bool {
        self.refill();  // Thread A refills

        // Thread A and B both check tokens > 0
        if self.tokens > 0 {
            self.tokens -= 1;  // Both decrement
            true
        } else {
            false
        }
    }
}
// Result: tokens can go negative or over-allocate
```

**Mitigation Strategy:**
```rust
impl TokenBucket {
    fn try_acquire(&self) -> bool {
        loop {
            let current = self.tokens.load(Ordering::Acquire);

            if current == 0 {
                return false;
            }

            // Atomic compare-and-swap
            if self.tokens.compare_exchange(
                current,
                current - 1,
                Ordering::Release,
                Ordering::Relaxed
            ).is_ok() {
                return true;
            }

            // Retry on contention
        }
    }

    fn refill(&self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill.load());

        let new_tokens = (elapsed.as_secs_f64() * self.rate) as u32;

        // Atomic max with capacity
        self.tokens.fetch_update(Ordering::SeqCst, Ordering::SeqCst, |current| {
            Some(std::cmp::min(current + new_tokens, self.capacity))
        });

        self.last_refill.store(now);
    }
}
```

**Simulation Scenario:** SIM-L3-006

---

## 2. State Consistency Risks

### 2.1 STATE-001: SQLite ↔ FUSE State Divergence

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **Impact** | Sync Corruption |
| **Detection** | Difficult |

**Description:**
SQLite stores the authoritative sync state, while FUSE maintains its own view via extended attributes. If these diverge, the system enters an inconsistent state where operations may fail silently.

**Divergence Scenarios:**
1. SQLite updated but xattr write fails
2. xattr updated but SQLite write fails
3. External tool modifies xattrs directly
4. Power loss between SQLite commit and xattr update

**Detection Strategy:**
```rust
impl IntegrityChecker {
    fn check_consistency(&self) -> Result<Vec<Inconsistency>> {
        let mut issues = Vec::new();

        for item in self.db.query_all_items()? {
            let xattr_state = self.fuse.get_xattr_state(&item.path)?;

            if xattr_state != item.state {
                issues.push(Inconsistency {
                    path: item.path.clone(),
                    db_state: item.state,
                    xattr_state,
                    recommendation: self.recommend_fix(&item, &xattr_state),
                });
            }
        }

        Ok(issues)
    }
}
```

**Prevention Strategy:**
```rust
impl StateManager {
    fn update_state(&self, path: &Path, new_state: ItemState) -> Result<()> {
        // Use transaction for atomicity
        let tx = self.db.begin_transaction()?;

        // Update SQLite first (can rollback)
        self.db.update_state_in_tx(&tx, path, new_state)?;

        // Update xattr
        match self.fuse.set_xattr_state(path, new_state) {
            Ok(_) => {
                tx.commit()?;
                Ok(())
            }
            Err(e) => {
                tx.rollback()?;
                Err(e)
            }
        }
    }
}
```

---

### 2.2 STATE-002: State Machine Deadlock (Error State)

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **Impact** | Stuck Files |
| **Detection** | Easy (files never sync) |

**Description:**
The current state machine design allows files to enter the `Error` state but does not define recovery transitions. Files stuck in Error remain unsynchronized indefinitely.

**Current State Machine (Incomplete):**
```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
┌────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐     │
│ Online │───▶│Hydrating │───▶│ Hydrated │───▶│Modified│─────┘
└────────┘    └──────────┘    └──────────┘    └────────┘
     │              │               │              │
     │              │               │              │
     ▼              ▼               ▼              ▼
┌─────────────────────────────────────────────────────────┐
│                        ERROR                             │
│                    (NO EXIT!)                            │
└─────────────────────────────────────────────────────────┘
```

**Required Recovery Transitions:**
```
┌─────────┐
│  Error  │
└────┬────┘
     │
     ├──── RetryHydration ────▶ Hydrating (if was hydrating)
     │
     ├──── RetrySync ─────────▶ Modified (if was syncing)
     │
     ├──── ResetToOnline ─────▶ Online (discard local)
     │
     └──── ForceConflict ─────▶ Conflicted (manual resolution)
```

**Implementation:**
```rust
impl StateMachine {
    fn recover_from_error(&self, item: &SyncItem, strategy: RecoveryStrategy) -> Result<ItemState> {
        match strategy {
            RecoveryStrategy::Retry => {
                // Retry the operation that failed
                match item.error_context {
                    ErrorContext::Hydration => Ok(ItemState::Hydrating),
                    ErrorContext::Upload => Ok(ItemState::Modified),
                    ErrorContext::Download => Ok(ItemState::Online),
                }
            }
            RecoveryStrategy::Reset => {
                // Discard local state
                self.clear_local_content(item)?;
                Ok(ItemState::Online)
            }
            RecoveryStrategy::Conflict => {
                // Escalate to user
                Ok(ItemState::Conflicted)
            }
        }
    }
}
```

**Simulation Scenario:** SIM-L2-001

---

## 3. Data Loss Scenarios

### 3.1 LOSS-001: inotify Queue Overflow

| Attribute | Value |
|-----------|-------|
| **Severity** | HIGH |
| **Probability** | Medium (bulk operations) |
| **Impact** | Unsynced Changes |

**Description:**
When the inotify kernel queue overflows (IN_Q_OVERFLOW), events are silently dropped. The current design does not handle this event, leading to missed file changes.

**Trigger Conditions:**
- Bulk file operations (unzip, git checkout)
- Many directories monitored
- Slow event processing

**Detection and Recovery:**
```rust
impl FileWatcher {
    fn handle_event(&mut self, event: inotify::Event) -> Result<()> {
        if event.mask.contains(EventMask::Q_OVERFLOW) {
            log::warn!("inotify queue overflow detected, initiating full scan");

            // Record overflow for metrics
            self.metrics.overflow_count.inc();

            // Mark all watches as potentially stale
            self.mark_all_stale();

            // Trigger full filesystem scan
            self.scheduler.schedule(Job::FullScan {
                priority: Priority::High,
                reason: ScanReason::QueueOverflow,
            });

            return Ok(());
        }

        // Normal event processing
        self.process_event(event)
    }

    fn mark_all_stale(&mut self) {
        for path in self.watched_paths.keys() {
            self.stale_paths.insert(path.clone());
        }
    }
}
```

**Simulation Scenario:** SIM-L3-004

---

### 3.2 LOSS-002: WAL Checkpoint Corruption

| Attribute | Value |
|-----------|-------|
| **Severity** | CRITICAL |
| **Probability** | Low |
| **Impact** | Database Corruption |

**Description:**
SQLite WAL mode improves concurrency but introduces corruption risk if checkpoints are interrupted (power loss, crash, forced termination).

**Vulnerable Window:**
```
Normal Operation:
writes → WAL file → checkpoint → main DB

Crash during checkpoint:
writes → WAL file → checkpoint (CRASH) → main DB inconsistent
```

**Mitigation Strategy:**
```rust
impl DatabaseManager {
    fn safe_checkpoint(&self) -> Result<()> {
        // Pause writes during checkpoint
        let _write_lock = self.write_lock.lock();

        // Use TRUNCATE mode for safer checkpoint
        self.conn.execute("PRAGMA wal_checkpoint(TRUNCATE)")?;

        // Verify integrity after checkpoint
        let integrity = self.conn.query_row(
            "PRAGMA integrity_check",
            [],
            |row| row.get::<_, String>(0)
        )?;

        if integrity != "ok" {
            log::error!("Database integrity check failed: {}", integrity);
            return Err(DbError::IntegrityFailure);
        }

        Ok(())
    }

    fn startup_recovery(&self) -> Result<()> {
        // Check for incomplete checkpoint
        let wal_size = self.wal_file_size()?;

        if wal_size > 0 {
            log::info!("WAL file present, attempting recovery");

            // Force checkpoint to recover
            match self.safe_checkpoint() {
                Ok(_) => log::info!("WAL recovery successful"),
                Err(e) => {
                    log::error!("WAL recovery failed: {}", e);
                    return self.restore_from_backup();
                }
            }
        }

        Ok(())
    }
}
```

**Simulation Scenario:** SIM-L3-005

---

### 3.3 LOSS-003: Conflict Naming Collision

| Attribute | Value |
|-----------|-------|
| **Severity** | MEDIUM |
| **Probability** | Low |
| **Impact** | File Overwrite |

**Description:**
When resolving conflicts with "keep-both" strategy, the system creates a renamed copy. If the naming scheme is predictable and a file with that name already exists, it may be overwritten.

**Vulnerable Flow:**
```
Original: document.docx
Conflict: document.docx_conflict

If document.docx_conflict already exists:
  → OVERWRITTEN!
```

**Safe Naming Strategy:**
```rust
impl ConflictResolver {
    fn generate_conflict_name(&self, original: &Path) -> PathBuf {
        let stem = original.file_stem().unwrap_or_default();
        let ext = original.extension().unwrap_or_default();
        let parent = original.parent().unwrap_or(Path::new("."));

        // Include timestamp and short UUID for uniqueness
        let timestamp = chrono::Utc::now().format("%Y%m%d_%H%M%S");
        let uuid_short = &uuid::Uuid::new_v4().to_string()[..8];

        let conflict_name = format!(
            "{}_conflict_{}_{}.{}",
            stem.to_string_lossy(),
            timestamp,
            uuid_short,
            ext.to_string_lossy()
        );

        let mut path = parent.join(&conflict_name);

        // Extra safety: ensure uniqueness
        let mut counter = 1;
        while path.exists() {
            path = parent.join(format!(
                "{}_conflict_{}_{}_{}.{}",
                stem.to_string_lossy(),
                timestamp,
                uuid_short,
                counter,
                ext.to_string_lossy()
            ));
            counter += 1;
        }

        path
    }
}
```

**Example Output:**
```
document_conflict_20260131_143022_a1b2c3d4.docx
```

**Simulation Scenario:** SIM-L4-005

---

## 4. Verification Checklist

### 4.1 Pre-Release Data Integrity Tests

- [ ] FUSE write during hydration is blocked or handled
- [ ] Dehydration respects active readers
- [ ] Conflict resolution checks for version changes
- [ ] inotify IN_Q_OVERFLOW triggers full scan
- [ ] WAL checkpoint recovery works after crash
- [ ] State machine has recovery paths from Error
- [ ] Conflict file names are unique
- [ ] SQLite ↔ xattr consistency check runs periodically

### 4.2 Monitoring Requirements

| Metric | Alert Threshold |
|--------|-----------------|
| Files in Error state > 1 hour | Warning |
| inotify overflow events | Critical |
| WAL file size > 100MB | Warning |
| SQLite/xattr inconsistencies | Critical |
| Conflict resolution failures | Warning |

---

## 5. Related Documents

- RISK-001: Critical Paths (SPOF analysis)
- RISK-002: Security Vulnerabilities
- SIM-L2-001 to SIM-L3-006: Simulation Scenarios

---

*Document generated by architectural simulation analysis*
*DevTrail Framework | https://enigmora.com*
