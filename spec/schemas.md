# Schema Specification

Version: 0.1.0-draft

This document defines conceptual schemas for IEK core entities. Serialization format is deferred to instantiating systems.

---

## Notation

- Types use PascalCase.
- Fields use snake_case.
- `[]T` denotes list of T.
- `Map<K, V>` denotes key-value mapping.
- `?T` denotes optional field of type T.

---

## Core Schemas

### Subject

Abstract entity tracked by the system. No default schema; MUST be defined by instantiating system.

```
Subject {
  subject_id    : str             # Unique within scope
  subject_type  : str             # Domain-specific type discriminator
  attributes    : Map<str, any>   # Type-specific properties
}
```

**Constraints**:
- `subject_id` MUST be stable across the subject's lifetime.
- Subjects are not stored directly; derived from events.

---

### Invariant

Named predicate over derived state.

```
Invariant {
  invariant_id  : str             # Stable identifier (e.g., "INV-L001")
  description   : str             # Human-readable summary
  parameters    : []Parameter     # Inputs to evaluation
  predicate     : PredicateSpec   # Formal specification
}

Parameter {
  name          : str
  type          : TypeRef
  required      : bool
}

PredicateSpec {
  language      : str             # e.g., "pseudocode", "datalog", "z3"
  expression    : str             # Predicate definition
}
```

**Constraints**:
- `invariant_id` MUST be unique within a system.
- `predicate` MUST be deterministic given identical inputs.

---

### Ruleset

Collection of invariants active in a scope.

```
Ruleset {
  ruleset_id    : str             # Unique identifier
  version       : str             # Immutable version string
  invariants    : []str           # invariant_ids included
  mode          : Mode            # Default operational mode
  created_at    : Timestamp
}

Mode = ACCOUNTING | GOVERNANCE
```

**Constraints**:
- Each `version` MUST be immutable once created.
- Changes produce new versions, not modifications (INV-C001).

---

### Scope

Jurisdiction boundary.

```
Scope {
  scope_id      : str             # Unique identifier
  selector      : ScopeSelector   # What is included
  parent        : ?str            # Parent scope_id for hierarchy
  active_rulesets : []RulesetRef  # Currently active rulesets
}

RulesetRef {
  ruleset_id    : str
  version       : str
  effective_seq : uint64          # From which sequence_id
}
```

---

### DerivedState

Computed projection of events up to a sequence point.

```
DerivedState {
  scope         : str             # scope_id
  as_of_seq     : uint64          # Sequence point
  subjects      : Map<str, SubjectState>
  aggregates    : Map<str, any>   # Domain-specific rollups
}

SubjectState {
  subject_id    : str
  attributes    : Map<str, any>
  active_events : []uint64        # Contributing sequence_ids
  revoked_events: []uint64        # Revoked sequence_ids
}
```

**Constraints**:
- DerivedState MUST be reproducible from Ledger replay.
- DerivedState is ephemeral; not persisted as source of truth.

---

### EvaluationResult

Output of an Evaluator run.

```
EvaluationResult {
  scope         : str             # scope_id evaluated
  ruleset       : RulesetRef      # Ruleset applied
  as_of_seq     : uint64          # State snapshot point
  compliant     : bool            # All invariants satisfied?
  imbalances    : []ImbalanceDescriptor
  evaluated_at  : Timestamp       # When evaluation ran
}
```

---

### ResolutionPlan

Output of Resolver planning phase.

```
ResolutionPlan {
  plan_id           : str             # Unique identifier for idempotency
  evaluation_id     : str             # Source evaluation
  imbalance_id      : str             # Target imbalance
  strategy          : str             # MINIMAL | FIFO | CUSTOM
  revocations       : []PlannedRevocation
  estimated_effect  : ?DerivedState   # Projected state after resolution
  safety            : PlanSafetyInfo  # Required safety metadata
}

PlannedRevocation {
  targets       : []uint64
  reason_code   : str
}

PlanSafetyInfo {
  blast_radius      : BlastRadius     # Scope of impact
  rollback_strategy : RollbackStrategy
  cascade_limits    : CascadeLimits
}

BlastRadius {
  affected_scopes   : []str           # scope_ids impacted
  affected_subjects : uint32          # Count of subjects affected
  affected_capabilities : uint32      # Count of capabilities revoked
}

RollbackStrategy {
  strategy_type     : COMPENSATE | IRREVERSIBLE | MANUAL
  compensation_plan : ?str            # Plan ID for reversal (if COMPENSATE)
  requires_elevated : bool            # If true, requires elevated governance mode
}

CascadeLimits {
  max_depth         : uint32          # Maximum resolution chain depth
  max_fan_out       : uint32          # Maximum revocations per imbalance
  cycle_detected    : bool            # True if potential cycle identified
}
```

**Constraints**:
- Plans MUST declare `blast_radius` before execution.
- Plans with `rollback_strategy.strategy_type = IRREVERSIBLE` MUST set `requires_elevated = true`.
- Plans with `cascade_limits.cycle_detected = true` MUST NOT execute without manual override.
- `max_depth` and `max_fan_out` MUST be finite; defaults defined by Control Plane.

---

### Capability

Abstract permission or resource token. Revocation targets capabilities, not subjects directly.

```
Capability {
  capability_id     : str             # Unique identifier
  capability_type   : str             # Domain-specific type
  subject_id        : str             # Holder of the capability
  scope             : str             # Jurisdiction where valid
  granted_by        : uint64          # sequence_id of granting event
  attributes        : Map<str, any>   # Type-specific properties
}
```

**Properties**:
- Capabilities are derived from ACTION events, not stored directly.
- Revocation nullifies the granting event, removing the capability from derived state.
- A subject's effective capabilities = granted - revoked.

**Common Capability Types** (domain-specific):

| Type | Description |
|------|-------------|
| `permission` | Authorization to perform an action |
| `quota` | Numeric allocation that can be partially consumed |
| `token` | Fungible unit (balance-style accounting) |
| `credential` | Attestation of status or qualification |

**Constraints**:
- `capability_id` MUST be derivable from `(subject_id, capability_type, granted_by)`.
- Capabilities MUST NOT exist without a corresponding granting event.
- Resolver subtracts capabilities; it does not impose obligations.

---

## Type References

### Primitive Types

| Type | Description |
|------|-------------|
| str | UTF-8 string |
| uint64 | Unsigned 64-bit integer |
| int64 | Signed 64-bit integer |
| float64 | 64-bit floating point |
| bool | Boolean |
| bytes | Arbitrary byte sequence |
| any | Dynamic type (discouraged; use sparingly) |

### Complex Types

| Type | Description |
|------|-------------|
| Timestamp | seconds + nanos (see events.md) |
| ScopeSelector | selector_type + value (see events.md) |
| Map<K, V> | Key-value mapping |
| []T | Ordered list |

---

## Schema Evolution

1. New optional fields MAY be added to existing schemas.
2. Required fields MUST NOT be added to existing schemas (breaking change).
3. Field types MUST NOT change (create new schema version instead).
4. Removed fields SHOULD be marked deprecated, not deleted.
5. Schema version MUST be incremented for any structural change.

---

## Domain Extension Pattern

Instantiating systems define domain schemas by:

1. Subtyping Subject with domain-specific types.
2. Defining ActionPayload subtypes for domain actions.
3. Adding domain Invariants to rulesets.
4. Extending DerivedState.aggregates for domain rollups.

Example (not normative):

```
# Domain: Resource Allocation

ResourceSubject : Subject {
  subject_type  : "resource"
  attributes    : {
    capacity    : uint64
    allocated   : uint64
  }
}

AllocationAction : ActionPayload {
  action_type   : "allocate"
  subject_id    : str           # Resource being allocated
  parameters    : {
    amount      : uint64
    requester   : str
  }
}
```
