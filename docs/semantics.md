# Semantics Specification

Version: 0.1.0-draft

This document defines operational semantics for IEK systems. It prevents subtle contradictions across events, schemas, and pseudocode by establishing a single source of truth for behavioral expectations.

---

## 1. Determinism Requirements

### 1.1 Determinism Scope

The following operations MUST be deterministic:

| Operation | Inputs | Determinism Guarantee |
|-----------|--------|----------------------|
| State Derivation | Ledger + derivation_version | Identical DerivedState |
| Invariant Evaluation | DerivedState + Ruleset | Identical ImbalanceDescriptors |
| Resolution Planning | Imbalance + Strategy + Config | Identical ResolutionPlan |

### 1.2 Non-Deterministic Elements

The following MAY be non-deterministic and MUST NOT affect evaluation outcomes:

- Wall-clock timestamps (advisory only)
- Execution duration
- Memory allocation patterns
- Log output ordering

### 1.3 Determinism Versioning

When the derivation algorithm changes:
1. A new `derivation_version` identifier MUST be assigned.
2. Prior evaluations remain valid under their recorded version.
3. Re-derivation under a new version produces a distinct result set.
4. Systems MUST support derivation under historical versions for audit.

---

## 2. Time Semantics

### 2.1 Event Time vs Processing Time

| Concept | Definition | Usage |
|---------|------------|-------|
| **Event Time** | `timestamp` field in Event envelope | Advisory; display, debugging, human review |
| **Processing Time** | Wall-clock when event is processed | Logging only; MUST NOT affect evaluation |
| **Sequence Time** | Ordering implied by `sequence_id` | Authoritative for state derivation |

### 2.2 Ordering Authority

- `sequence_id` is the sole authoritative ordering for events.
- `timestamp` MAY be out of order with respect to `sequence_id`.
- State derivation uses `sequence_id` ordering exclusively.
- Evaluators MUST NOT use `timestamp` for invariant logic unless the invariant explicitly parameterizes time.

### 2.3 Clock Assumptions

- No global clock is assumed.
- `timestamp` is provided by the event source; it may be skewed or untrusted.
- Systems requiring trusted time MUST use a time oracle pattern with explicit trust model.

---

## 3. Ordering and Consistency

### 3.1 Ledger Ordering Guarantees

- **Total Order**: All events have a unique position in the sequence.
- **Append Linearizability**: Once an event is assigned a sequence_id, it is durably ordered.
- **No Gaps**: sequence_ids form a contiguous sequence (1, 2, 3, ...).

### 3.2 Read Consistency

| Query Type | Consistency | Notes |
|------------|-------------|-------|
| `derive_state(scope, seq)` | Point-in-time | Always consistent as of `seq` |
| `evaluate(state, ruleset)` | Snapshot | Consistent within single evaluation |
| Head queries | Eventual | May lag behind most recent append |

### 3.3 Write Consistency

- Append is atomic: an event either has a sequence_id or does not exist.
- Concurrent appends are serialized by the Ledger.
- Writers receive sequence_id upon successful append.

### 3.4 Happens-Before Relationships

```
append(E1) → append(E2)  implies  E1.sequence_id < E2.sequence_id

evaluate(S1) → resolve(I1) → append(R1)
  where S1 detected I1, and R1 addresses I1
```

---

## 4. Replay Modes

### 4.1 Live Mode

**Purpose**: Normal operation with authoritative governance.

**Properties**:
- Evaluations produce authoritative ImbalanceDescriptors.
- Resolutions may trigger Revocation appends.
- Mode is scoped per jurisdiction.

**Constraints**:
- Retroactivity guard applies (see architecture.md §4.3).
- Only current ruleset version may be used for new evaluations.

### 4.2 Audit Mode

**Purpose**: Reproduce historical evaluations for verification.

**Properties**:
- Evaluations use historical ruleset versions.
- Results are marked `non_authoritative: true`.
- No Revocations are generated.

**Constraints**:
- MUST specify exact `(ruleset_id, ruleset_version, as_of_seq)`.
- Results MUST match historical EvaluationResult if determinism holds.

### 4.3 Simulation Mode

**Purpose**: What-if analysis without affecting live state.

**Properties**:
- May use hypothetical rulesets not yet activated.
- May project forward with synthetic events.
- Results are marked `non_authoritative: true`, `simulated: true`.

**Constraints**:
- MUST NOT append to production Ledger.
- MAY use isolated simulation Ledger.

### 4.4 Mode Indicators

Every EvaluationResult includes:

```
replay_context {
  mode            : LIVE | AUDIT | SIMULATION
  authoritative   : bool
  simulated       : bool
  replay_reason   : ?str           # Why replay was invoked
}
```

---

## 5. Idempotency Keys

### 5.1 Event Idempotency

Events carry an `event_id` (client-assigned, globally unique):

```
Event {
  event_id      : str             # Idempotency key
  sequence_id   : uint64          # Ledger-assigned
  ...
}
```

**Behavior**:
- First append with `event_id = X` succeeds, assigns sequence_id.
- Subsequent appends with `event_id = X` return existing sequence_id.
- Idempotency window: indefinite (event_ids are never forgotten).

### 5.2 Resolution Idempotency

ResolutionPlans carry `(evaluation_id, plan_id)`:

```
ResolutionPlan {
  evaluation_id : str             # Source evaluation
  plan_id       : str             # Unique within evaluation
  ...
}
```

**Behavior**:
- First execution of `(evaluation_id, plan_id)` appends Revocations.
- Subsequent executions return existing Revocation sequence_ids.
- Resolver checks execution registry before appending.

### 5.3 Idempotency Failure Modes

| Scenario | Behavior |
|----------|----------|
| Duplicate event_id, same payload | Return existing sequence_id |
| Duplicate event_id, different payload | Reject with IDEMPOTENCY_CONFLICT |
| Duplicate plan execution | Return existing Revocation sequence_ids |
| Plan execution after imbalance superseded | No-op; imbalance no longer exists |

---

## 6. Consistency Boundaries

### 6.1 Scope Isolation

- Evaluations are scoped; cross-scope dependencies are explicit.
- A Revocation in Scope A does not automatically affect Scope B.
- Cross-scope effects require explicit GOVERNANCE events.

### 6.2 Version Boundaries

- A ruleset version defines a consistent set of invariants.
- Mixing invariants from different ruleset versions in one evaluation is forbidden.
- Ruleset transitions are atomic within a scope.

### 6.3 Derivation Boundaries

- State derivation is scoped and versioned.
- Changing derivation logic creates a new version; old derivations remain valid under old version.
- Migration between derivation versions is explicit, not automatic.

---

## 7. Error Semantics

### 7.1 Transient vs Permanent Failures

| Failure Type | Examples | Retry Behavior |
|--------------|----------|----------------|
| Transient | Network timeout, resource exhaustion | Retry with backoff |
| Permanent | Invalid event schema, referential violation | Reject; do not retry |
| Idempotent | Duplicate event_id | Return existing; no error |

### 7.2 Partial Failures

- Ledger append is atomic; no partial appends.
- Resolution execution may fail mid-plan; executed revocations persist.
- Failed plans should be re-planned from current state, not resumed.

### 7.3 Error Attribution

Errors are logged with:
- `error_code`: machine-readable identifier
- `error_scope`: component that detected error
- `correlation_id`: links to triggering operation
- `recoverable`: whether retry is meaningful

---

## 8. Cross-Reference

| Concept | Primary Definition |
|---------|-------------------|
| Append-only semantics | invariants.md INV-L001 |
| Monotonic sequencing | invariants.md INV-L002 |
| Determinism | invariants.md INV-E001 |
| Idempotency (Ledger) | invariants.md INV-I001 |
| Idempotency (Resolver) | invariants.md INV-I002 |
| Reproducibility | invariants.md INV-I003, INV-I004 |
| Retroactivity guard | architecture.md §4.3 |
| Trigger budget | architecture.md §5.2 |
| Cascade limits | schemas.md ResolutionPlan.safety.cascade_limits |
