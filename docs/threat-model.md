# Threat Model

Version: 0.1.0-draft

This document identifies risks to IEK-based systems. Focus is on abuse, misconfiguration, and governance capture—not only external attackers.

---

## Scope

This threat model addresses:
1. **Misuse**: System used for purposes outside design intent.
2. **Misconfiguration**: Correct system, incorrect setup.
3. **Governance Capture**: Control Plane subverted by partial interests.
4. **Operational Failures**: Bugs, race conditions, resource exhaustion.

Explicitly out of scope:
- Physical security
- Network-level attacks (deferred to deployment specification)
- Cryptographic implementation details

---

## Threat Categories

### T1: Misuse for Human Targeting

**Description**: System instantiated with domain schemas designed to track, profile, or control natural persons.

**Risk Level**: Critical

**Mitigations**:
- Architectural exclusion: no built-in person/identity primitives.
- Review requirement: domain schemas should be audited for targeting potential.
- Documentation: explicit non-goal in charter.

**Control**: README.md exclusions; ADR-0001 D6 (Capability model defines abstract tokens, not identity).

**Conformance**: N/A (normative exclusion; not testable via conformance suite).

**Residual Risk**: Cannot prevent all misuse by determined instantiators. Mitigation is normative, not technical.

---

### T2: Scope Creep via Resolver

**Description**: Resolver generates events that exceed subtractive correction—creating obligations, penalties, or new tracking state.

**Risk Level**: High

**Mitigations**:
- INV-R001 enforcement: Resolver output schema restricted to Revocation type.
- Audit: all Resolver outputs logged and reviewable.
- Testing: invariant test suite includes scope creep detection.

**Control**: INV-R001, INV-R002; events.md EventType enum (only REVOCATION, CAPABILITY_REVOKED for resolution).

**Conformance**: RC-01 (subtractive only), RC-02 (imbalance reference required).

**Detection**: Schema validation; semantic analysis of Revocation effects.

---

### T3: Governance Capture

**Description**: Control Plane configuration modified to serve partial interests rather than stated system goals.

**Risk Level**: High

**Mitigations**:
- Version immutability (INV-C001): changes produce new versions, not mutations.
- Audit trail: all configuration changes are Ledger events.
- Separation: Control Plane defines rules but cannot directly modify Ledger truth.

**Control**: INV-C001; architecture.md §4 (versioning); events.md GOVERNANCE event type with attribution.

**Conformance**: L-01 (append-only preserves audit trail), I-01/I-02 (idempotency prevents silent overwrites).

**Detection**: Configuration diff analysis; stakeholder review of ruleset changes.

---

### T4: Retroactive Rule Application

**Description**: New invariants applied to historical state, creating imbalances for actions that were compliant when taken.

**Risk Level**: Medium

**Mitigations**:
- Ruleset versioning with effective-date semantics.
- Evaluator queries specify ruleset version explicitly.
- Policy decision: retroactivity should require explicit governance approval.

**Control**: architecture.md §4.3 (retroactivity guard); events.md `occurred_at` + `ruleset_ref` fields.

**Conformance**: R-01 (retroactive blocked in LIVE), R-02 (allowed in AUDIT with non_authoritative).

**Detection**: Timestamp comparison between event and ruleset effective date.

---

### T5: Non-Deterministic Evaluation

**Description**: Evaluator produces different results for same inputs due to hidden state, randomness, or external dependencies.

**Risk Level**: Medium

**Mitigations**:
- INV-E001 enforcement: determinism is a core invariant.
- Testing: property-based tests for Evaluator determinism.
- Sandboxing: Evaluators run in isolated environments without external access.

**Control**: INV-E001, INV-E002, INV-I003, INV-I004; events.md `derivation_version` field.

**Conformance**: D-01 (state derivation reproducibility), D-02 (evaluation reproducibility).

**Detection**: Repeated evaluation; replay comparison.

---

### T6: Ledger Integrity Compromise

**Description**: Events modified or deleted after sequencing, breaking derived state consistency.

**Risk Level**: Critical

**Mitigations**:
- INV-L001, INV-L002 enforcement.
- Hash chaining or Merkle tree for tamper evidence.
- Replication: multiple independent Ledger copies.

**Control**: INV-L001, INV-L002, INV-L003; events.md `sequence_id` (ledger-assigned, immutable).

**Conformance**: L-01 (append-only), L-02 (monotonic), L-03 (referential integrity), BR-01 (backup/restore determinism).

**Detection**: Hash verification; cross-replica comparison.

---

### T7: Revocation Cascade

**Description**: A single revocation triggers additional imbalances, causing unbounded cascade of further revocations.

**Risk Level**: Medium

**Mitigations**:
- Resolver strategies should include cycle detection.
- Rate limiting on revocation generation.
- Governance mode can be scoped to prevent global cascades.

**Control**: INV-T001, INV-T002; schemas.md `CascadeLimits` (max_depth, max_fan_out, cycle_detected).

**Conformance**: S-02 (cycle detection blocking), S-03 (cascade depth limit), S-04 (trigger budget exhaustion).

**Detection**: Revocation rate monitoring; dependency graph analysis.

---

### T8: Constraint Engine Manipulation

**Description**: Constraint Engine returns biased or incomplete solutions that favor certain outcomes.

**Risk Level**: Low (Engine is advisory only)

**Mitigations**:
- Engine output is advisory; Resolver makes final decision.
- Multiple solution enumeration rather than single "best" answer.
- Engine logic should be auditable.

**Control**: architecture.md §2.5 (Engine is advisory, read-only); INV-T003 (plan safety validation).

**Conformance**: S-01 (irreversible plan blocking validates Resolver override capability).

**Detection**: Solution comparison across runs; alternative engine validation.

---

### T9: Resource Exhaustion

**Description**: Unbounded event streams, expensive invariant checks, or large revocation batches exhaust system resources.

**Risk Level**: Medium

**Mitigations**:
- Rate limiting on event append.
- Timeout enforcement on Evaluator execution.
- Batch size limits on Resolver operations.

**Control**: INV-T001; architecture.md §5.2 `TriggerBudget`; schemas.md `CascadeLimits.max_fan_out`.

**Conformance**: S-04 (trigger budget exhaustion).

**Detection**: Resource monitoring; circuit breakers.

---

### T10: Jurisdiction Ambiguity

**Description**: Overlapping or undefined scopes cause events to be evaluated under multiple conflicting rulesets, or not at all.

**Risk Level**: Medium

**Mitigations**:
- Explicit scope resolution rules in Control Plane.
- Default-deny: events without clear jurisdiction are flagged, not silently processed.
- Testing: scope coverage analysis.

**Control**: schemas.md `Scope` with explicit `selector` and `parent`; events.md `ScopeSelector` type.

**Conformance**: R-01, R-02 (scope-bound evaluation); D-01 (scoped state derivation).

**Detection**: Jurisdiction gap/overlap reports.

---

## Summary Matrix

| ID | Threat | Risk | Control | Conformance |
|----|--------|------|---------|-------------|
| T1 | Human Targeting | Critical | README exclusions; ADR-0001 D6 | N/A (normative) |
| T2 | Resolver Scope Creep | High | INV-R001, INV-R002 | RC-01, RC-02 |
| T3 | Governance Capture | High | INV-C001; §4 versioning | L-01, I-01, I-02 |
| T4 | Retroactive Rules | Medium | §4.3 retroactivity guard | R-01, R-02 |
| T5 | Non-Determinism | Medium | INV-E001, INV-I003, INV-I004 | D-01, D-02 |
| T6 | Ledger Tampering | Critical | INV-L001, INV-L002, INV-L003 | L-01, L-02, L-03, BR-01 |
| T7 | Revocation Cascade | Medium | INV-T001, INV-T002; CascadeLimits | S-02, S-03, S-04 |
| T8 | Constraint Bias | Low | §2.5 advisory; INV-T003 | S-01 |
| T9 | Resource Exhaustion | Medium | INV-T001; TriggerBudget | S-04 |
| T10 | Jurisdiction Ambiguity | Medium | Scope schema; ScopeSelector | R-01, R-02, D-01 |

---

## Review Schedule

This threat model SHOULD be reviewed:
- When new components are added to the architecture.
- When domain-specific instantiation introduces new Subject/Action types.
- Annually, or after any security incident.
