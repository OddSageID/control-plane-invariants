# Event Specification

Version: 0.1.0-draft

This document defines the event types and structures for IEK-compliant systems.

---

## Event Envelope

All events share a common envelope structure:

```
Event {
  sequence_id   : uint64          # Assigned by Ledger; monotonic
  event_type    : EventType       # Discriminator
  timestamp     : Timestamp       # Advisory; not used for ordering
  payload       : bytes           # Type-specific content
  references    : []uint64        # Prior sequence_ids (optional)
  metadata      : Map<str, str>   # Extensible; not used in invariant checks
}
```

### Field Semantics

| Field | Required | Mutable | Notes |
|-------|----------|---------|-------|
| sequence_id | Yes | No | Assigned at append time |
| event_type | Yes | No | Determines payload schema |
| timestamp | Yes | No | Wall-clock or logical; advisory |
| payload | Yes | No | Interpreted per event_type |
| references | No | No | For Revocations: target events |
| metadata | No | No | Tracing, debugging; ignored by Evaluators |

---

## Event Types

### ACTION

Domain-specific state change. Schema defined by instantiating system.

```
ActionPayload {
  action_type   : str             # Domain-specific discriminator
  subject_id    : str             # Affected subject identifier
  parameters    : Map<str, any>   # Action-specific data
}
```

**Constraints**:
- MUST specify subject_id.
- MUST NOT reference other events (use REVOCATION for corrections).

---

### REVOCATION

Nullifies effects of prior events.

```
RevocationPayload {
  targets       : []uint64        # sequence_ids being revoked
  imbalance_ref : str             # Identifier of triggering imbalance
  reason_code   : str             # Machine-readable reason
}
```

**Constraints**:
- `targets` MUST be non-empty.
- `targets` MUST reference existing sequence_ids (INV-L003).
- `imbalance_ref` MUST reference a valid ImbalanceDescriptor (INV-R002).
- MUST NOT revoke other REVOCATION events (prevents infinite regress).

---

### GOVERNANCE

Control Plane configuration change.

```
GovernancePayload {
  governance_type : GovernanceType  # Discriminator
  scope           : ScopeSelector   # Affected jurisdiction
  config          : bytes           # Type-specific configuration
}

GovernanceType = MODE_CHANGE | RULESET_ACTIVATION | RULESET_DEACTIVATION | SCOPE_DEFINITION
```

**Subtypes**:

#### MODE_CHANGE

```
ModeChangeConfig {
  new_mode      : Mode            # ACCOUNTING | GOVERNANCE
  effective_seq : uint64          # Sequence from which mode applies
}
```

#### RULESET_ACTIVATION

```
RulesetActivationConfig {
  ruleset_id    : str
  version       : str
  effective_seq : uint64
}
```

#### RULESET_DEACTIVATION

```
RulesetDeactivationConfig {
  ruleset_id    : str
  effective_seq : uint64
}
```

#### SCOPE_DEFINITION

```
ScopeDefinitionConfig {
  scope_id      : str
  selector      : ScopeSelector
  parent_scope  : str             # Optional; for hierarchy
}
```

---

## Supporting Types

### Timestamp

```
Timestamp {
  seconds       : int64           # Unix epoch seconds
  nanos         : uint32          # Sub-second precision
}
```

### ScopeSelector

```
ScopeSelector {
  selector_type : SUBJECT_SET | EVENT_STREAM | PREDICATE
  value         : str             # Interpretation depends on type
}
```

### ImbalanceDescriptor

Produced by Evaluators; referenced by Revocations.

```
ImbalanceDescriptor {
  id            : str             # Unique identifier
  invariant_id  : str             # Which invariant violated
  scope         : ScopeSelector   # Affected jurisdiction
  magnitude     : float64         # Quantified deviation
  contributing  : []uint64        # Events contributing to imbalance
  detected_at   : uint64          # Sequence_id when detected
}
```

---

## Event Ordering

1. Events are ordered by sequence_id, not timestamp.
2. Timestamp MAY be used for display or debugging; MUST NOT affect evaluation.
3. Concurrent appends (in distributed deployments) MUST be serialized before sequencing.

---

## Extension Guidelines

Instantiating systems:
- MUST define domain-specific ActionPayload schemas.
- MAY define additional GovernancePayload subtypes.
- MUST NOT modify the Event envelope structure.
- SHOULD document payload schemas in their own spec/schemas.md.
