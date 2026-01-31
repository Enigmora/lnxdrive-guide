# RISK-001: Critical Paths and Single Points of Failure

**Document ID:** RISK-001
**Created:** 2026-01-31
**Status:** Draft
**Author:** claude-code-v1.0
**Risk Level:** Critical

---

## Executive Summary

This document identifies critical paths and Single Points of Failure (SPOF) in the LNXDrive architecture. Each SPOF represents a component whose failure would cause system-wide unavailability or data integrity issues.

---

## 1. Single Points of Failure Identified

### 1.1 SPOF-001: DBus Session Bus

| Attribute | Value |
|-----------|-------|
| **Component** | DBus Session Bus (org.enigmora.LNXDrive) |
| **Criticality** | CRITICAL |
| **Failure Impact** | All UI adapters lose communication with daemon |
| **MTTR Estimate** | 30-60 seconds (manual restart) |

**Description:**
All UI adapters (GNOME, KDE, Cosmic, XFCE, CLI) communicate with the daemon exclusively through DBus. If the DBus session crashes or becomes unresponsive, ALL user interfaces become non-functional.

**Failure Scenarios:**
1. DBus daemon crash
2. DBus message queue overflow
3. Session bus timeout under load
4. DBus service name collision

**Current Mitigations:** None documented

**Proposed Mitigations:**
1. Implement DBus connection health monitoring with automatic reconnection
2. Add fallback communication channel (Unix socket) for critical operations
3. Implement graceful degradation - daemon continues sync even without DBus
4. Add DBus watchdog with automatic service restart

**Simulation Scenario:** SIM-L1-001

---

### 1.2 SPOF-002: SQLite State Database

| Attribute | Value |
|-----------|-------|
| **Component** | SQLite + WAL (sync_state.db) |
| **Criticality** | CRITICAL |
| **Failure Impact** | Loss of sync state, potential data corruption |
| **MTTR Estimate** | 5-30 minutes (depends on recovery) |

**Description:**
SQLite is the single source of truth for synchronization state. All operations (FUSE, sync engine, conflict resolution) depend on this database. SQLite only supports single-writer, creating contention between FUSE operations and sync engine writes.

**Failure Scenarios:**
1. Database corruption during WAL checkpoint
2. Disk full preventing writes
3. Concurrent write contention causing SQLITE_BUSY
4. Power loss during checkpoint operation
5. File system corruption affecting DB file

**Current Mitigations:**
- WAL mode for read concurrency
- `PRAGMA synchronous = NORMAL`

**Proposed Mitigations:**
1. Implement periodic backup to secondary location
2. Add integrity checks on startup (`PRAGMA integrity_check`)
3. Implement WAL checkpoint policy (size-based + time-based)
4. Add write serialization layer to prevent SQLITE_BUSY
5. Implement corruption recovery from last known good state
6. Consider read replicas for FUSE queries

**Simulation Scenario:** SIM-L3-001, SIM-L3-005

---

### 1.3 SPOF-003: Microsoft Graph API

| Attribute | Value |
|-----------|-------|
| **Component** | Microsoft Graph API (graph.microsoft.com) |
| **Criticality** | HIGH |
| **Failure Impact** | No cloud synchronization for OneDrive |
| **MTTR Estimate** | External dependency (minutes to hours) |

**Description:**
Microsoft Graph is the sole communication channel with OneDrive. When Graph API is unavailable, no synchronization can occur. This affects all OneDrive accounts in multi-provider setup.

**Failure Scenarios:**
1. Graph API outage (Microsoft-side)
2. Rate limiting (429 responses)
3. OAuth token expiration without refresh
4. Network connectivity issues
5. DNS resolution failures
6. TLS certificate issues

**Current Mitigations:**
- Exponential backoff on errors
- Token bucket rate limiting

**Proposed Mitigations:**
1. Implement offline mode with change queuing
2. Add circuit breaker pattern (stop retries after N failures)
3. Implement proactive token refresh (before expiration)
4. Add health endpoint monitoring
5. Cache delta responses for resilience
6. Implement graceful degradation notifications

**Simulation Scenario:** SIM-L3-002, SIM-L3-003

---

### 1.4 SPOF-004: Delta Token

| Attribute | Value |
|-----------|-------|
| **Component** | Microsoft Graph Delta Token |
| **Criticality** | HIGH |
| **Failure Impact** | Requires full re-synchronization |
| **MTTR Estimate** | Minutes to hours (depends on file count) |

**Description:**
Delta sync relies on a token that tracks the last known state. This token expires after 3-4 months of inactivity or can be invalidated by Microsoft. Loss of delta token requires expensive full enumeration.

**Failure Scenarios:**
1. Token expiration (inactivity > 90 days)
2. Token invalidation (server-side)
3. Token corruption in local storage
4. API returns 410 Gone response

**Current Mitigations:** None documented

**Proposed Mitigations:**
1. Implement automatic detection of 410 Gone
2. Add graceful fallback to full sync with progress indication
3. Persist delta token with backup
4. Implement delta token refresh before expiration
5. Add user notification for long re-sync operations

**Simulation Scenario:** SIM-L3-002

---

### 1.5 SPOF-005: FUSE Mount Point

| Attribute | Value |
|-----------|-------|
| **Component** | FUSE Virtual Filesystem |
| **Criticality** | CRITICAL |
| **Failure Impact** | Files-on-Demand completely unavailable |
| **MTTR Estimate** | Seconds (unmount/remount) to minutes |

**Description:**
FUSE provides the Files-on-Demand functionality. If the FUSE daemon crashes or the mount becomes stale, all placeholder files become inaccessible, and applications may hang on I/O operations.

**Failure Scenarios:**
1. FUSE daemon crash
2. Kernel FUSE module failure
3. Mount point becomes stale (network filesystem-like behavior)
4. Out of memory in FUSE daemon
5. Deadlock between FUSE and SQLite operations

**Current Mitigations:** None documented

**Proposed Mitigations:**
1. Implement FUSE health monitoring
2. Add automatic remount on failure detection
3. Implement request timeout to prevent hangs
4. Add memory limits and monitoring
5. Implement graceful shutdown with pending I/O completion
6. Add stale mount detection and cleanup

**Simulation Scenario:** SIM-L2-002, SIM-L2-003

---

## 2. Critical Path Analysis

### 2.1 File Access Critical Path

```
User Application
      ↓
  FUSE (open/read)
      ↓
  Hydration Manager ←──→ SQLite (state check)
      ↓
  MS Graph API (download)
      ↓
  Local Cache (write)
      ↓
  FUSE (return data)
```

**Failure Points:**
1. FUSE daemon unresponsive → Application hangs
2. SQLite locked → Operation blocked
3. MS Graph timeout → Hydration fails
4. Disk full → Write fails

**Latency Budget:**
| Component | Max Latency | Current |
|-----------|-------------|---------|
| FUSE dispatch | 10ms | Unknown |
| SQLite query | 5ms | Unknown |
| Network (cached) | 0ms | N/A |
| Network (hydrate) | 5000ms | Unknown |

---

### 2.2 Sync Critical Path

```
inotify event
      ↓
  Event Router
      ↓
  Debounce/Coalesce
      ↓
  SQLite (state update)
      ↓
  MS Graph API (upload)
      ↓
  SQLite (confirm sync)
      ↓
  Observer notification
```

**Failure Points:**
1. inotify queue overflow → Events lost
2. Event router crash → Sync stops
3. SQLite write fails → State inconsistent
4. MS Graph fails → Upload queued
5. Observer timeout → Notification blocked

---

### 2.3 Conflict Resolution Critical Path

```
Conflict detected
      ↓
  SQLite (store conflict)
      ↓
  DBus signal (notify UI)
      ↓
  User interaction
      ↓
  Resolution applied
      ↓
  SQLite (update state)
      ↓
  MS Graph (sync resolution)
```

**Failure Points:**
1. DBus signal lost → User not notified
2. User timeout → Conflict stuck
3. Resolution race condition → Wrong version applied
4. Graph upload fails → Resolution pending

---

## 3. Dependency Graph

```
                    ┌─────────────────────┐
                    │   User Application  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
        ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
        │   GNOME   │    │    KDE    │    │    CLI    │
        │    UI     │    │    UI     │    │           │
        └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
              │                │                │
              └────────────────┼────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    DBus Service     │ ← SPOF-001
                    │  (org.enigmora...)  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   lnxdrive-daemon   │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
   ┌─────▼─────┐        ┌──────▼──────┐       ┌──────▼──────┐
   │   FUSE    │ ← SPOF │   SQLite    │ ← SPOF│   inotify   │
   │  Mount    │   -005 │   State     │   -002│   Watcher   │
   └─────┬─────┘        └──────┬──────┘       └──────┬──────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Sync Engine      │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Microsoft Graph    │ ← SPOF-003
                    │  (Delta Token)      │ ← SPOF-004
                    └─────────────────────┘
```

---

## 4. Risk Mitigation Priority

| Priority | SPOF | Mitigation | Effort | Impact |
|----------|------|------------|--------|--------|
| P0 | SPOF-002 | SQLite write serialization | M | Prevents corruption |
| P0 | SPOF-005 | FUSE request timeout | S | Prevents hangs |
| P1 | SPOF-001 | DBus reconnection | M | Improves availability |
| P1 | SPOF-003 | Offline mode | L | Enables local work |
| P2 | SPOF-004 | Delta token recovery | M | Graceful degradation |

---

## 5. Monitoring Requirements

### 5.1 Health Metrics

| Metric | Threshold | Alert |
|--------|-----------|-------|
| DBus latency p99 | > 100ms | Warning |
| SQLite write latency | > 50ms | Warning |
| FUSE operation latency p99 | > 500ms | Critical |
| Graph API errors/min | > 10 | Warning |
| inotify queue usage | > 80% | Critical |

### 5.2 Availability Targets

| Component | Target | Measurement |
|-----------|--------|-------------|
| FUSE availability | 99.9% | Mount responsive |
| Sync availability | 99% | Successful sync cycles |
| UI responsiveness | 99.5% | DBus response < 1s |

---

## 6. Related Documents

- RISK-002: Security Vulnerabilities
- RISK-003: Data Integrity Risks
- SIM-L1-001 to SIM-L3-006: Simulation Scenarios

---

*Document generated by architectural simulation analysis*
*DevTrail Framework | https://enigmora.com*
