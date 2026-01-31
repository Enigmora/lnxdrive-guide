# Traceability Matrix: Risks → Simulations → Mitigations

**Document ID:** TRACE-001
**Created:** 2026-01-31
**Status:** Draft
**Author:** claude-code-v1.0

---

## 1. Risk to Simulation Mapping

| Risk ID | Risk Description | Simulation ID | Priority |
|---------|------------------|---------------|----------|
| A1 | DBus SPOF | SIM-L1-001, SIM-L1-002 | P0 |
| A2 | SQLite ↔ FUSE race | SIM-L3-001 | P0 |
| A3 | MS Graph SPOF | SIM-L3-003 | P1 |
| A4 | Delta token expiry | SIM-L3-002 | P1 |
| A5 | WAL checkpoint | SIM-L3-005 | P2 |
| A6 | Observer timeout | SIM-L4-006 | P1 |
| B1 | No Error recovery | SIM-L2-001 | P0 |
| B2 | Ambiguous errors | - | P2 |
| B3 | No SLAs | - | P3 |
| B4 | Lifecycle undefined | SIM-L2-005 | P3 |
| B5 | IN_Q_OVERFLOW | SIM-L3-004 | P1 |
| B6 | Naming collision | SIM-L4-005 | P3 |
| C1 | Write during hydration | SIM-L2-002 | P0 |
| C2 | Dehydration race | SIM-L2-003 | P2 |
| C3 | inotify eviction | SIM-L3-004 | P1 |
| C4 | Priority update race | - | P3 |
| C5 | Conflict resolution race | SIM-L2-004 | P2 |
| C6 | Rate limiter race | SIM-L3-006 | P2 |
| D1 | DBus no auth | SIM-L1-002 | P0 |
| D2 | Token exposure | SIM-L1-003 | P0 |
| D3 | DBus no rate limit | SIM-L1-002 | P1 |
| D4 | No plugin sandbox | SIM-L4-002 | P1 |
| D5 | YAML injection | SIM-L4-003 | P0 |
| E1 | No race tests | SIM-TEST-001 | P2 |
| E2 | No chaos eng | SIM-TEST-002 | P2 |
| E3 | No multi-provider | SIM-TEST-003 | P2 |
| E4 | No load testing | SIM-TEST-004 | P2 |
| F1 | ICloudProvider gaps | SIM-L4-001 | P3 |
| F2 | No dynamic plugins | SIM-L4-002 | P3 |
| F3 | No DBus versioning | SIM-L4-004 | P3 |
| F4 | No config migration | SIM-L4-007 | P3 |

---

## 2. Simulation to Mitigation Mapping

| Simulation ID | Component | Mitigation Strategy | Implementation Effort |
|---------------|-----------|---------------------|----------------------|
| SIM-L1-001 | DBus | Reconnection + fallback socket | L |
| SIM-L1-002 | DBus | PolicyKit + rate limiting | M |
| SIM-L1-003 | DBus | Session handles, keyring integration | M |
| SIM-L2-001 | State Machine | Add Error recovery transitions | M |
| SIM-L2-002 | FUSE | Exclusive lock during hydration | S |
| SIM-L2-003 | FUSE | Reader tracking before dehydration | M |
| SIM-L2-004 | Conflict | Version check before resolution | S |
| SIM-L2-005 | Core | Reference counting, leak detection | M |
| SIM-L3-001 | SQLite | Write serialization layer | M |
| SIM-L3-002 | Graph | 410 detection + full resync | M |
| SIM-L3-003 | Graph | Offline mode + change queue | L |
| SIM-L3-004 | inotify | IN_Q_OVERFLOW handler + full scan | M |
| SIM-L3-005 | SQLite | Checkpoint policy + integrity check | M |
| SIM-L3-006 | Rate | Atomic token bucket operations | S |
| SIM-L4-001 | Provider | Document limitations, add methods | S |
| SIM-L4-002 | Plugin | Process isolation + seccomp | L |
| SIM-L4-003 | Config | JSON Schema + safe YAML parsing | S |
| SIM-L4-004 | DBus | Versioned interfaces | M |
| SIM-L4-005 | Conflict | UUID-based unique naming | S |
| SIM-L4-006 | Observer | Timeout wrapper for callbacks | S |
| SIM-L4-007 | Config | Migration system | M |

---

## 3. Mitigation to Test Mapping

| Mitigation | Test Type | Test Cases |
|------------|-----------|------------|
| DBus reconnection | Integration | `test_dbus_reconnect_after_crash` |
| PolicyKit auth | Unit + E2E | `test_polkit_required_for_sensitive_ops` |
| Session handles | Security | `test_no_tokens_in_dbus_traffic` |
| Error recovery | Unit | `test_error_to_hydrating_transition` |
| Hydration lock | Concurrency | `test_write_blocked_during_hydration` |
| Version check | Unit | `test_resolution_fails_on_version_change` |
| Write serialization | Concurrency | `test_concurrent_fuse_sqlite_writes` |
| Offline mode | Integration | `test_sync_continues_offline` |
| IN_Q_OVERFLOW | Stress | `test_inotify_overflow_triggers_scan` |
| Checkpoint policy | Recovery | `test_wal_recovery_after_crash` |
| Atomic token bucket | Concurrency | `test_no_token_overallocation` |
| Plugin sandbox | Security | `test_plugin_cannot_access_other_accounts` |
| YAML validation | Security | `test_billion_laughs_rejected` |
| Unique naming | Unit | `test_conflict_names_never_collide` |
| Observer timeout | Unit | `test_slow_observer_does_not_block_sync` |

---

## 4. Priority Matrix

### P0 - Critical (Sprint 1-2)

| Risk | Simulation | Mitigation | Test |
|------|------------|------------|------|
| D2 | SIM-L1-003 | Session handles | Security scan |
| D5 | SIM-L4-003 | YAML validation | Fuzzing |
| C1 | SIM-L2-002 | Hydration lock | Concurrency |
| A2 | SIM-L3-001 | Write serialization | Stress |
| B1 | SIM-L2-001 | Error recovery | Unit |

### P1 - High (Sprint 3-4)

| Risk | Simulation | Mitigation | Test |
|------|------------|------------|------|
| A1 | SIM-L1-001 | DBus reconnection | Integration |
| A4 | SIM-L3-002 | Delta recovery | Integration |
| A3 | SIM-L3-003 | Offline mode | E2E |
| B5 | SIM-L3-004 | Overflow handler | Stress |
| D4 | SIM-L4-002 | Plugin sandbox | Security |
| A6 | SIM-L4-006 | Observer timeout | Unit |

### P2 - Medium (Sprint 4-5)

| Risk | Simulation | Mitigation | Test |
|------|------------|------------|------|
| C2 | SIM-L2-003 | Dehydration sync | Concurrency |
| C5 | SIM-L2-004 | Version check | Unit |
| A5 | SIM-L3-005 | Checkpoint policy | Recovery |
| C6 | SIM-L3-006 | Atomic bucket | Concurrency |
| D3 | SIM-L1-002 | Rate limiting | Stress |

### P3 - Low (Sprint 5+)

| Risk | Simulation | Mitigation | Test |
|------|------------|------------|------|
| B4 | SIM-L2-005 | Lifecycle tracking | Memory |
| F1 | SIM-L4-001 | API documentation | Review |
| F3 | SIM-L4-004 | Versioned API | Compat |
| B6 | SIM-L4-005 | Unique naming | Unit |
| F4 | SIM-L4-007 | Config migration | Upgrade |

---

## 5. Coverage Summary

### By Component

| Component | Risks | Simulations | Coverage |
|-----------|-------|-------------|----------|
| DBus | 4 | 3 | 75% |
| FUSE | 3 | 3 | 100% |
| SQLite | 2 | 2 | 100% |
| State Machine | 2 | 2 | 100% |
| MS Graph | 2 | 2 | 100% |
| inotify | 2 | 1 | 50% |
| Config | 2 | 2 | 100% |
| Plugin | 2 | 1 | 50% |
| Rate Limiter | 1 | 1 | 100% |

### By Severity

| Severity | Total Risks | Covered | Coverage |
|----------|-------------|---------|----------|
| Critical | 8 | 8 | 100% |
| High | 10 | 9 | 90% |
| Medium | 7 | 5 | 71% |
| Low | 5 | 3 | 60% |

---

## 6. Gap Analysis

### Uncovered Risks

| Risk ID | Description | Reason | Recommendation |
|---------|-------------|--------|----------------|
| B2 | Ambiguous errors | Documentation issue | ADR for error taxonomy |
| B3 | No SLAs in contracts | Design decision needed | AIDEC for SLA policy |
| C4 | Priority update race | Low probability | Monitor in production |

### Missing Simulations

| Area | Gap | Priority |
|------|-----|----------|
| Multi-account | Concurrent sync of 5+ accounts | P2 |
| Large files | Files > 10GB handling | P2 |
| Symbolic links | Symlink sync behavior | P3 |
| Hard links | Hard link handling | P3 |
| Sparse files | Sparse file support | P3 |

---

## 7. Dependency Chain

```
SIM-L2-001 (State Machine Error Recovery)
    │
    ├──→ SIM-L2-002 (FUSE Hydration Race)
    │        └──→ Requires defined state transitions
    │
    ├──→ SIM-L2-003 (Dehydration Race)
    │        └──→ Requires defined state transitions
    │
    └──→ SIM-L2-004 (Conflict Resolution Race)
             └──→ Requires Conflicted state handling

SIM-L3-001 (SQLite Race)
    │
    └──→ SIM-L3-005 (WAL Checkpoint)
             └──→ Requires stable concurrent access

SIM-L1-001 (DBus SPOF)
    │
    ├──→ SIM-L1-002 (DBus DoS)
    │        └──→ Requires reconnection logic first
    │
    └──→ SIM-L4-004 (DBus Versioning)
             └──→ Requires stable DBus infrastructure

SIM-L4-003 (YAML Validation)
    │
    └──→ SIM-L4-007 (Config Migration)
             └──→ Requires validation before migration
```

---

## 8. Verification Checklist

### Pre-Implementation
- [ ] All P0 risks have simulations defined
- [ ] All simulations have mitigations specified
- [ ] All mitigations have test cases identified
- [ ] Dependencies between simulations documented

### During Implementation
- [ ] Mitigations implemented with tests
- [ ] Simulations executed and verified
- [ ] Coverage metrics updated

### Post-Implementation
- [ ] All P0/P1 simulations pass
- [ ] No regression in existing tests
- [ ] Documentation updated

---

*Document generated by architectural simulation analysis*
*DevTrail Framework | https://enigmora.com*
