# Event Specification

Version: 0.1.0-draft

This document defines the event types and structures for IEK-compliant systems.

---

## Event Envelope

All events share a common envelope structure. This is a **strict contract**; all required fields MUST be present.

```
Event {
  # === Identity (client-provided) ===
  event_id          : str             # REQUIRED. Globally unique; idempotency key
  event_type        : EventType       # REQUIRED. Discriminator

  # === Sequencing (ledger-assigned) ===
  sequence_id       : uint64          # REQUIRED. Assigned by Ledger; monotonic

  # === Time (dual timestamps) ===
  occurred_at       : Timestamp       # REQUIRED. When event happened (event time)
  observed_at       : Timestamp       # REQUIRED. When ledger received (processing time)

  # === Versioning ===
  derivation_version : str            # REQUIRED. Algorithm version for state computation

  # === Governance Context (conditional) ===
  ruleset_ref       : ?RulesetRef     # REQUIRED for EVALUATION, RESOLUTION events

  # === Content ===
  payload           : bytes           # REQUIRED. Type-specific content
  references        : []uint64        # Optional. Prior sequence_ids (for Revocations)

  # === Extensibility ===
  metadata          : Map<str, str>   # Optional. Tracing, debugging; ignored by Evaluators

  # === Authentication (optional) ===
  auth_context      : ?AuthContext    # Optional. Writer identity and authorization claims
  signature         : ?Signature      # Optional. Cryptographic signature over envelope
}

AuthContext {
  principal_id      : str             # Authenticated identity of writer
  claims            : Map<str, str>   # Authorization claims (roles, scopes, etc.)
  auth_method       : str             # How authentication was performed
  auth_time         : Timestamp       # When authentication occurred
}

Signature {
  algorithm         : str             # Signature algorithm (e.g., "ed25519", "ecdsa-p256")
  key_id            : str             # Identifier of signing key
  value             : bytes           # Signature bytes
  signed_fields     : []str           # Which envelope fields are covered
}

RulesetRef {
  ruleset_id        : str
  ruleset_version   : str
}
```

### Field Semantics

| Field | Required | Mutable | Assigned By | Notes |
|-------|----------|---------|-------------|-------|
| event_id | **Yes** | No | Client | Idempotency key; duplicate appends return existing sequence_id |
| event_type | **Yes** | No | Client | Determines payload schema |
| sequence_id | **Yes** | No | Ledger | Monotonic; authoritative ordering |
| occurred_at | **Yes** | No | Client | Event time; used for retroactivity checks |
| observed_at | **Yes** | No | Ledger | Processing time; audit trail only |
| derivation_version | **Yes** | No | Client | Enables reproducible state derivation |
| ruleset_ref | Conditional | No | Client | Required for EVALUATION, RESOLUTION; links to governance context |
| payload | **Yes** | No | Client | Interpreted per event_type |
| references | No | No | Client | For Revocations: target events |
| metadata | No | No | Client | Tracing, debugging; ignored by Evaluators |
| auth_context | No | No | Client | Writer identity; for attribution and access control |
| signature | No | No | Client | Cryptographic proof of authenticity |

### Idempotency Behavior

Per INV-I001:
- First append with `event_id = X` succeeds; assigns `sequence_id`.
- Subsequent appends with `event_id = X` and **identical payload**: return existing `sequence_id`.
- Subsequent appends with `event_id = X` and **different payload**: reject with `IDEMPOTENCY_CONFLICT`.

### Time Field Usage

| Field | Used For | Not Used For |
|-------|----------|--------------|
| `occurred_at` | Retroactivity guard (§4.3 architecture.md); display | Ordering; evaluation logic |
| `observed_at` | Audit; latency measurement | Ordering; evaluation logic |
| `sequence_id` | Authoritative ordering; state derivation | Time-based queries |

### Authentication Fields

The `auth_context` and `signature` fields are OPTIONAL. When present:

- `auth_context` provides attribution metadata for access control and audit.
- `signature` provides cryptographic proof that the event was created by a specific key holder.

**Derivation Exclusion Rule**: DerivedState MUST NOT depend on `auth_context` or `signature` contents.

- These fields are excluded from state derivation inputs.
- Evaluators MUST NOT read `auth_context` or `signature` for invariant evaluation.
- The only permitted derivation-adjacent use is a `signature_verified: bool` audit annotation, which:
  - MUST be treated as metadata, not derivation input.
  - MUST NOT affect ImbalanceDescriptor generation.
  - MAY be recorded in EvaluationResult for audit purposes.

**Rationale**: Preserves determinism (INV-E001, INV-I003). Authentication state is orthogonal to truth; mixing them breaks replay guarantees.

> **FUTURE**: LIVE mode MAY require signature verification via policy configuration. When signature enforcement is enabled:
> - Ledger SHOULD reject unsigned events or events with invalid signatures.
> - AUDIT mode records `signature_verified: bool` status but does not reject on failure.
> - Signature verification MUST NOT affect determinism: the same event with the same signature produces the same verification result.
> - See ADR-0001 "Auth/Signing Enforcement" for constraints on introducing this capability.

---

## Event Types

```
EventType = ACTION | REVOCATION | GOVERNANCE | CAPABILITY_GRANTED | CAPABILITY_REVOKED
```

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

**Subtypes (see below)**

---

### CAPABILITY_GRANTED

Issuance of a capability to a subject. Source-of-truth for capability existence.

```
CapabilityGrantedPayload {
  capability_id     : str             # Unique identifier
  capability_type   : str             # permission | quota | token | credential | domain-specific
  subject_id        : str             # Recipient of capability
  scope             : str             # Jurisdiction where valid
  attributes        : Map<str, any>   # Type-specific properties
  granted_by        : str             # Authority that issued (may be system or subject)
  expires_at        : ?Timestamp      # Optional expiration
}
```

**Constraints**:
- `capability_id` MUST be unique within scope.
- MUST specify `subject_id` and `capability_type`.
- Creates a capability in DerivedState upon processing.

**Lifecycle**: This event is the sole source-of-truth for capability existence. Capabilities do not exist until granted.

---

### CAPABILITY_REVOKED

Nullification of a previously granted capability. Resolution effect.

```
CapabilityRevokedPayload {
  capability_id     : str             # Capability being revoked
  granted_event_id  : str             # event_id of CAPABILITY_GRANTED being nullified
  imbalance_ref     : str             # ImbalanceDescriptor that triggered revocation
  reason_code       : str             # Machine-readable reason
  partial           : bool            # If true, partial revocation (quota reduction)
  reduction_amount  : ?float64        # For partial: amount to subtract
}
```

**Constraints**:
- `granted_event_id` MUST reference a valid CAPABILITY_GRANTED event.
- `imbalance_ref` MUST reference a valid ImbalanceDescriptor (INV-R002).
- If `partial = false`, capability is fully removed from DerivedState.
- If `partial = true`, `reduction_amount` MUST be specified; capability attributes are adjusted.

**Lifecycle**: Removes or reduces the capability from DerivedState. The granting event remains in Ledger (append-only), but is marked as revoked for state derivation.

---

## Capability Lifecycle

```
CAPABILITY_GRANTED (event_id: G1)
        │
        ▼
    Capability exists in DerivedState
        │
        │ (imbalance detected)
        ▼
CAPABILITY_REVOKED (granted_event_id: G1)
        │
        ▼
    Capability removed from DerivedState
    (G1 still in Ledger, marked revoked)
```

**Derived State Computation**:

```
capabilities(subject, scope, seq) =
  { c | CAPABILITY_GRANTED(c) in events(seq)
      AND c.subject_id = subject
      AND c.scope = scope
      AND NOT EXISTS CAPABILITY_REVOKED(c.capability_id, partial=false) in events(seq)
  }
  with partial revocations applied to attributes
```

---

## Governance Event Subtypes

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
