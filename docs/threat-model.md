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

**Residual Risk**: Cannot prevent all misuse by determined instantiators. Mitigation is normative, not technical.

---

### T2: Scope Creep via Resolver

**Description**: Resolver generates events that exceed subtractive correction—creating obligations, penalties, or new tracking state.

**Risk Level**: High

**Mitigations**:
- INV-R001 enforcement: Resolver output schema restricted to Revocation type.
- Audit: all Resolver outputs logged and reviewable.
- Testing: invariant test suite includes scope creep detection.

**Detection**: Schema validation; semantic analysis of Revocation effects.

---

### T3: Governance Capture

**Description**: Control Plane configuration modified to serve partial interests rather than stated system goals.

**Risk Level**: High

**Mitigations**:
- Version immutability (INV-C001): changes produce new versions, not mutations.
- Audit trail: all configuration changes are Ledger events.
- Separation: Control Plane defines rules but cannot directly modify Ledger truth.

**Detection**: Configuration diff analysis; stakeholder review of ruleset changes.

---

### T4: Retroactive Rule Application

**Description**: New invariants applied to historical state, creating imbalances for actions that were compliant when taken.

**Risk Level**: Medium

**Mitigations**:
- Ruleset versioning with effective-date semantics.
- Evaluator queries specify ruleset version explicitly.
- Policy decision: retroactivity should require explicit governance approval.

**Detection**: Timestamp comparison between event and ruleset effective date.

---

### T5: Non-Deterministic Evaluation

**Description**: Evaluator produces different results for same inputs due to hidden state, randomness, or external dependencies.

**Risk Level**: Medium

**Mitigations**:
- INV-E001 enforcement: determinism is a core invariant.
- Testing: property-based tests for Evaluator determinism.
- Sandboxing: Evaluators run in isolated environments without external access.

**Detection**: Repeated evaluation; replay comparison.

---

### T6: Ledger Integrity Compromise

**Description**: Events modified or deleted after sequencing, breaking derived state consistency.

**Risk Level**: Critical

**Mitigations**:
- INV-L001, INV-L002 enforcement.
- Hash chaining or Merkle tree for tamper evidence.
- Replication: multiple independent Ledger copies.

**Detection**: Hash verification; cross-replica comparison.

---

### T7: Revocation Cascade

**Description**: A single revocation triggers additional imbalances, causing unbounded cascade of further revocations.

**Risk Level**: Medium

**Mitigations**:
- Resolver strategies should include cycle detection.
- Rate limiting on revocation generation.
- Governance mode can be scoped to prevent global cascades.

**Detection**: Revocation rate monitoring; dependency graph analysis.

---

### T8: Constraint Engine Manipulation

**Description**: Constraint Engine returns biased or incomplete solutions that favor certain outcomes.

**Risk Level**: Low (Engine is advisory only)

**Mitigations**:
- Engine output is advisory; Resolver makes final decision.
- Multiple solution enumeration rather than single "best" answer.
- Engine logic should be auditable.

**Detection**: Solution comparison across runs; alternative engine validation.

---

### T9: Resource Exhaustion

**Description**: Unbounded event streams, expensive invariant checks, or large revocation batches exhaust system resources.

**Risk Level**: Medium

**Mitigations**:
- Rate limiting on event append.
- Timeout enforcement on Evaluator execution.
- Batch size limits on Resolver operations.

**Detection**: Resource monitoring; circuit breakers.

---

### T10: Jurisdiction Ambiguity

**Description**: Overlapping or undefined scopes cause events to be evaluated under multiple conflicting rulesets, or not at all.

**Risk Level**: Medium

**Mitigations**:
- Explicit scope resolution rules in Control Plane.
- Default-deny: events without clear jurisdiction are flagged, not silently processed.
- Testing: scope coverage analysis.

**Detection**: Jurisdiction gap/overlap reports.

---

## Summary Matrix

| ID | Threat | Risk | Primary Mitigation |
|----|--------|------|-------------------|
| T1 | Human Targeting | Critical | No person primitives; audit |
| T2 | Resolver Scope Creep | High | INV-R001; schema restriction |
| T3 | Governance Capture | High | Version immutability; audit trail |
| T4 | Retroactive Rules | Medium | Effective-date semantics |
| T5 | Non-Determinism | Medium | INV-E001; sandboxing |
| T6 | Ledger Tampering | Critical | Hash chain; replication |
| T7 | Revocation Cascade | Medium | Cycle detection; rate limits |
| T8 | Constraint Bias | Low | Advisory only; auditability |
| T9 | Resource Exhaustion | Medium | Rate/size limits; timeouts |
| T10 | Jurisdiction Ambiguity | Medium | Explicit resolution; default-deny |

---

## Review Schedule

This threat model SHOULD be reviewed:
- When new components are added to the architecture.
- When domain-specific instantiation introduces new Subject/Action types.
- Annually, or after any security incident.
