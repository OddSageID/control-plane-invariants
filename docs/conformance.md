# Conformance Test Matrix

Version: 0.1.0-draft

This document defines conformance scenarios for IEK-compliant systems. Implementations MUST pass all scenarios to claim conformance.

---

## Purpose

Before code exists, this matrix establishes expected behavior. It serves as:
1. North star for implementation.
2. Acceptance criteria for testing.
3. Interoperability baseline for multiple implementations.

---

## Notation

- **Scenario**: Named test case.
- **Precondition**: State before test.
- **Action**: Operation performed.
- **Expected**: Required outcome.
- **Invariant**: Which invariant(s) this tests.

---

## Ledger Conformance

### L-01: Append-Only Integrity

| Aspect | Value |
|--------|-------|
| **Precondition** | Event E1 appended with sequence_id = 100 |
| **Action** | Attempt to modify E1 payload or delete E1 |
| **Expected** | Operation REJECTED; E1 unchanged |
| **Invariant** | INV-L001 |

---

### L-02: Monotonic Sequencing

| Aspect | Value |
|--------|-------|
| **Precondition** | Ledger head at sequence_id = 100 |
| **Action** | Append new event E2 |
| **Expected** | E2 receives sequence_id > 100 (e.g., 101) |
| **Invariant** | INV-L002 |

---

### L-03: Referential Integrity

| Aspect | Value |
|--------|-------|
| **Precondition** | Ledger contains events with sequence_ids [1, 2, 3] |
| **Action** | Append REVOCATION targeting sequence_id = 999 |
| **Expected** | Append REJECTED; "reference not found" error |
| **Invariant** | INV-L003 |

---

## Idempotency Conformance

### I-01: Duplicate Event Same Payload

| Aspect | Value |
|--------|-------|
| **Precondition** | Event with event_id = "EVT-001" appended; sequence_id = 50 |
| **Action** | Append event with event_id = "EVT-001", identical payload |
| **Expected** | Returns sequence_id = 50; no new ledger entry; derived state unchanged |
| **Invariant** | INV-I001 |

---

### I-02: Duplicate Event Different Payload

| Aspect | Value |
|--------|-------|
| **Precondition** | Event with event_id = "EVT-001" appended with payload P1 |
| **Action** | Append event with event_id = "EVT-001", payload P2 ≠ P1 |
| **Expected** | Append REJECTED with IDEMPOTENCY_CONFLICT error |
| **Invariant** | INV-I001 |

---

### I-03: Duplicate Resolution Execution

| Aspect | Value |
|--------|-------|
| **Precondition** | ResolutionPlan (evaluation_id = "EVAL-01", plan_id = "PLAN-01") executed; Revocations appended |
| **Action** | Execute same ResolutionPlan again |
| **Expected** | No new Revocation events; returns existing sequence_ids |
| **Invariant** | INV-I002 |

---

## Retroactivity Conformance

### R-01: Retroactive Evaluation Blocked in LIVE

| Aspect | Value |
|--------|-------|
| **Precondition** | Event E1 with occurred_at = T1; Ruleset V2 with created_at = T2 where T2 > T1 |
| **Action** | Evaluate E1 in LIVE mode using Ruleset V2 |
| **Expected** | Evaluation REJECTED; "retroactivity violation" error |
| **Invariant** | architecture.md §4.3 |

---

### R-02: Retroactive Evaluation Allowed in AUDIT

| Aspect | Value |
|--------|-------|
| **Precondition** | Event E1 with occurred_at = T1; Ruleset V2 with created_at = T2 where T2 > T1 |
| **Action** | Evaluate E1 in AUDIT mode using Ruleset V2 |
| **Expected** | Evaluation succeeds; result marked non_authoritative = true |
| **Invariant** | semantics.md §4.2 |

---

## Resolution Safety Conformance

### S-01: Irreversible Plan in Non-Elevated Mode

| Aspect | Value |
|--------|-------|
| **Precondition** | ResolutionPlan with rollback_strategy.strategy_type = IRREVERSIBLE; current mode = GOVERNANCE (not elevated) |
| **Action** | Attempt to execute plan |
| **Expected** | Execution BLOCKED; "elevated mode required" error |
| **Invariant** | INV-T003 |

---

### S-02: Cycle Detected Plan Without Override

| Aspect | Value |
|--------|-------|
| **Precondition** | ResolutionPlan with cascade_limits.cycle_detected = true; no manual override present |
| **Action** | Attempt to execute plan |
| **Expected** | Execution BLOCKED; "cycle detected, manual override required" error |
| **Invariant** | INV-T003 |

---

### S-03: Cascade Depth Exceeded

| Aspect | Value |
|--------|-------|
| **Precondition** | max_depth = 3; resolution chain at depth = 3 triggers new imbalance |
| **Action** | Resolver attempts to create resolution at depth = 4 |
| **Expected** | Resolution HALTED; ALERT generated; manual intervention required |
| **Invariant** | INV-T002 |

---

### S-04: Trigger Budget Exhausted

| Aspect | Value |
|--------|-------|
| **Precondition** | TriggerBudget for scope S1: max_evaluations_per_window = 10; 10 evaluations already triggered |
| **Action** | 11th trigger fires |
| **Expected** | Trigger follows overflow_action (QUEUE, DROP, or ALERT); budget exhaustion logged |
| **Invariant** | INV-T001 |

---

## Determinism Conformance

### D-01: State Derivation Reproducibility

| Aspect | Value |
|--------|-------|
| **Precondition** | Ledger L with events [E1, E2, ..., En]; derivation_version = V1 |
| **Action** | Derive state on Instance A and Instance B with same (L, V1, scope, seq) |
| **Expected** | DerivedState on A == DerivedState on B (byte-identical or semantically equivalent) |
| **Invariant** | INV-I003 |

---

### D-02: Evaluation Reproducibility

| Aspect | Value |
|--------|-------|
| **Precondition** | DerivedState S; Ruleset R (id, version); derivation_version V |
| **Action** | Evaluate on Instance A and Instance B with same (S, R, V) |
| **Expected** | ImbalanceDescriptors on A == ImbalanceDescriptors on B |
| **Invariant** | INV-I004 |

---

## Capability Lifecycle Conformance

### C-01: Capability Grant Creates State

| Aspect | Value |
|--------|-------|
| **Precondition** | Subject S1 has no capabilities in scope SC1 |
| **Action** | Append CAPABILITY_GRANTED for capability CAP-01 to S1 in SC1 |
| **Expected** | DerivedState includes CAP-01 for S1 in SC1 |
| **Invariant** | events.md capability lifecycle |

---

### C-02: Capability Revocation Removes State

| Aspect | Value |
|--------|-------|
| **Precondition** | Subject S1 has capability CAP-01 (granted by event G1) |
| **Action** | Append CAPABILITY_REVOKED targeting G1, partial = false |
| **Expected** | DerivedState no longer includes CAP-01 for S1; G1 remains in Ledger |
| **Invariant** | events.md capability lifecycle |

---

### C-03: Partial Revocation Reduces Quota

| Aspect | Value |
|--------|-------|
| **Precondition** | Subject S1 has quota capability CAP-01 with amount = 100 |
| **Action** | Append CAPABILITY_REVOKED targeting CAP-01, partial = true, reduction_amount = 30 |
| **Expected** | DerivedState shows CAP-01 with amount = 70 |
| **Invariant** | events.md capability lifecycle |

---

## Resolver Constraint Conformance

### RC-01: Subtractive Only

| Aspect | Value |
|--------|-------|
| **Precondition** | Imbalance detected |
| **Action** | Resolver generates resolution events |
| **Expected** | All generated events are REVOCATION or CAPABILITY_REVOKED; no additive events |
| **Invariant** | INV-R001 |

---

### RC-02: Imbalance Reference Required

| Aspect | Value |
|--------|-------|
| **Precondition** | Resolver attempts to generate Revocation |
| **Action** | Revocation created without imbalance_ref |
| **Expected** | Revocation REJECTED; "imbalance_ref required" error |
| **Invariant** | INV-R002 |

---

## Backup and Recovery Conformance

### BR-01: Backup/Restore Determinism

| Aspect | Value |
|--------|-------|
| **Precondition** | Ledger L with events [E1...En]; DerivedState S with state_hash H1 |
| **Action** | Export L to backup; restore to new instance; derive state; compute state_hash H2 |
| **Expected** | H1 == H2 (byte-identical or semantically equivalent) |
| **Invariant** | INV-I003, INV-L001 |

**Notes**: Validates that backup/restore preserves ledger integrity and derived state reproducibility. Critical for disaster recovery and system migration.

---

### EB-01: Evidence Bundle Integrity

| Aspect | Value |
|--------|-------|
| **Precondition** | Ledger L; DerivedState S at sequence N; exported evidence bundle B containing events + cryptographic hashes |
| **Action** | Independently replay events from B; compute state_hash; verify against bundle's claimed hash |
| **Expected** | Replayed state_hash matches bundle's claimed hash; hash chain verifies |
| **Invariant** | INV-I003, INV-L001, INV-L002 |

**Notes**: Validates that evidence bundles are self-verifying. Enables offline audit and cross-system verification without trusted third party.

---

## Scope Constraint Conformance

### T1-01: INDIVIDUAL Scope Rejection

| Aspect | Value |
|--------|-------|
| **Precondition** | System in ACCOUNTING or standard GOVERNANCE mode (not elevated) |
| **Action** | Attempt to create or evaluate scope with `scope_type = INDIVIDUAL` |
| **Expected** | Operation REJECTED; "INDIVIDUAL scope requires elevated mode + policy approval" error |
| **Invariant** | T1 structural control |

**Notes**: Validates architectural gate against human-targeting misuse. INDIVIDUAL scope is permitted only when: (1) elevated governance mode is active, AND (2) a GOVERNANCE event records explicit policy approval for the scope. This is a defense-in-depth measure; it raises the bar but does not eliminate misuse risk.

---

## Summary Matrix

| ID | Category | Scenario | Key Invariant |
|----|----------|----------|---------------|
| L-01 | Ledger | Append-only integrity | INV-L001 |
| L-02 | Ledger | Monotonic sequencing | INV-L002 |
| L-03 | Ledger | Referential integrity | INV-L003 |
| I-01 | Idempotency | Duplicate event same payload | INV-I001 |
| I-02 | Idempotency | Duplicate event different payload | INV-I001 |
| I-03 | Idempotency | Duplicate resolution execution | INV-I002 |
| R-01 | Retroactivity | Blocked in LIVE | §4.3 |
| R-02 | Retroactivity | Allowed in AUDIT | §4.2 |
| S-01 | Safety | Irreversible plan blocking | INV-T003 |
| S-02 | Safety | Cycle detection blocking | INV-T003 |
| S-03 | Safety | Cascade depth limit | INV-T002 |
| S-04 | Safety | Trigger budget exhaustion | INV-T001 |
| D-01 | Determinism | State derivation | INV-I003 |
| D-02 | Determinism | Evaluation | INV-I004 |
| C-01 | Capability | Grant creates state | lifecycle |
| C-02 | Capability | Revocation removes state | lifecycle |
| C-03 | Capability | Partial revocation | lifecycle |
| RC-01 | Resolver | Subtractive only | INV-R001 |
| RC-02 | Resolver | Imbalance reference | INV-R002 |
| BR-01 | Backup/Recovery | Restore determinism | INV-I003, INV-L001 |
| EB-01 | Backup/Recovery | Evidence bundle integrity | INV-I003, INV-L001, INV-L002 |
| T1-01 | Scope Constraint | INDIVIDUAL scope rejection | T1 control |

---

## Conformance Levels

### Level 1: Core

Implementations MUST pass: L-01, L-02, L-03, I-01, I-02, D-01, D-02.

### Level 2: Governance

Level 1 + MUST pass: R-01, R-02, S-01, S-02, S-03, S-04, I-03.

### Level 3: Full

Level 2 + MUST pass: C-01, C-02, C-03, RC-01, RC-02, BR-01, EB-01, T1-01.

---

## Test Execution Notes

1. Tests are black-box; implementation details are not validated.
2. "Byte-identical" for determinism may be relaxed to "semantically equivalent" if serialization differs.
3. Error messages are examples; implementations MAY use different wording with equivalent semantics.
4. Timing-sensitive tests (budget windows) require test harness with controllable clock.
5. Backup/recovery tests (BR-01, EB-01) require export/import capability; implementations without backup features MAY defer these tests.
