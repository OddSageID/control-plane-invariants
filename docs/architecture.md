# Architecture Specification

Version: 0.1.0-draft
Status: Foundational

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Rulesets    │  │ Jurisdiction │  │  Governance Triggers │  │
│  │  (versioned) │  │   Scopes     │  │  (mode transitions)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │ signals
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       PROCESSING LAYER                          │
│                                                                 │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐                 │
│  │ Evaluator│───▶│ Imbalance │───▶│ Resolver │                 │
│  │ (read)   │    │ Descriptor│    │ (write)  │                 │
│  └────┬─────┘    └───────────┘    └────┬─────┘                 │
│       │ reads                          │ appends               │
│       ▼                                ▼                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                         LEDGER                              ││
│  │              (append-only event log)                        ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘

Optional:
┌─────────────────────────────────────────────────────────────────┐
│                    CONSTRAINT ENGINE                            │
│         (literal constraint satisfaction; advisory only)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Responsibilities

### 2.1 Ledger

**Purpose**: Single source of truth. Stores all events that constitute system history.

**Properties**:
- MUST be append-only. No event, once written, may be modified or deleted.
- MUST assign monotonic sequence numbers to events.
- MUST support deterministic replay from any sequence position.
- SHOULD support content-addressable event retrieval.
- MAY support multiple append streams with merge semantics.

**Non-responsibilities**:
- Does NOT interpret events.
- Does NOT enforce invariants at write time.
- Does NOT maintain derived state.

### 2.2 Evaluator

**Purpose**: Stateless invariant checker. Given derived state and a ruleset, returns compliance status.

**Properties**:
- MUST be a pure function: same inputs produce same outputs.
- MUST NOT write to the Ledger or any persistent store.
- MUST NOT access state outside its explicit inputs.
- MUST ignore event metadata unrelated to invariant parameters (e.g., timestamps unless time is an invariant parameter).
- SHOULD return structured imbalance descriptors, not boolean only.

**Non-responsibilities**:
- Does NOT determine corrections.
- Does NOT infer intent or causality.
- Does NOT prioritize among multiple imbalances.

### 2.3 Resolver

**Purpose**: Generates correction events to restore invariant compliance.

**Properties**:
- MUST only append Revocation events to the Ledger.
- MUST reference the Imbalance being addressed in each Revocation.
- MUST NOT create events that add obligations, penalties, or new state beyond nullification.
- SHOULD be idempotent: resolving an already-resolved imbalance produces no new events.
- MAY batch multiple revocations in a single operation.

**Non-responsibilities**:
- Does NOT detect imbalances (Evaluator responsibility).
- Does NOT define what constitutes an imbalance (Control Plane responsibility).

### 2.4 Control Plane

**Purpose**: Governance configuration. Defines which rules apply, under what conditions, to which scopes.

**Properties**:
- MUST version all ruleset configurations.
- MUST define jurisdiction boundaries as explicit scope selectors.
- MUST declare mode transitions (Accounting ↔ Governance) with clear triggers.
- SHOULD support rule composition without modification of base rules.
- MAY support delegation of authority to sub-scopes.

**Non-responsibilities**:
- Does NOT store truth (Ledger responsibility).
- Does NOT evaluate invariants (Evaluator responsibility).
- Does NOT execute corrections (Resolver responsibility).

### 2.5 Constraint Engine (Optional)

**Purpose**: Advisory constraint satisfaction. Answers "what revocations would restore invariant X?"

**Properties**:
- MUST be read-only with respect to the Ledger.
- MUST NOT guarantee that a satisfying assignment exists.
- MUST NOT guarantee termination for arbitrary constraint sets.
- SHOULD return candidate solutions ranked by some metric (e.g., minimal revocations).
- MAY timeout or return partial results.

**Non-responsibilities**:
- Does NOT execute revocations.
- Does NOT validate that candidates are permissible under current governance.

---

## 3. Interfaces

### 3.1 Events (Ledger ← All Writers)

```
Event {
  sequence_id    : monotonic identifier
  event_type     : ACTION | REVOCATION | GOVERNANCE
  payload        : domain-specific content
  references     : list of prior sequence_ids (optional)
  timestamp      : logical or wall-clock (advisory only)
}
```

### 3.2 Commands (Control Plane → Processing Layer)

```
EvaluateCommand {
  scope          : jurisdiction selector
  ruleset_version: version identifier
  as_of_sequence : sequence_id (optional; defaults to head)
}

ResolveCommand {
  imbalance_id   : reference to Evaluator output
  strategy       : MINIMAL | FIFO | CUSTOM
}
```

### 3.3 Queries (Evaluator → Ledger)

```
StateQuery {
  scope          : jurisdiction selector
  as_of_sequence : sequence_id
}

Returns: DerivedState (computed projection of events)
```

### 3.4 Outputs

```
ImbalanceDescriptor {
  invariant_id   : which invariant is violated
  scope          : affected jurisdiction
  magnitude      : quantified deviation (domain-specific)
  contributing_events : list of sequence_ids
}

RevocationEvent {
  event_type     : REVOCATION
  targets        : list of sequence_ids being nullified
  imbalance_ref  : ImbalanceDescriptor identifier
}
```

---

## 4. Data Model (Conceptual)

### 4.1 Core Entities

| Entity | Description |
|--------|-------------|
| Event | Immutable record in the Ledger |
| Subject | Abstract entity with state (no default schema) |
| Invariant | Named predicate over derived state |
| Scope | Jurisdiction boundary; set of subjects or event streams |
| Ruleset | Collection of invariants active in a scope |

### 4.2 State Derivation

Derived state is computed, not stored:

```
DerivedState(scope, seq) = fold(
  filter(events, scope),
  initial_state,
  apply_event
) where event.sequence_id <= seq
```

### 4.3 Revocation Semantics

Revocations do not delete events. They append a new event that marks prior events as nullified for state derivation:

```
apply_event(state, revocation) =
  state - effects_of(revocation.targets)
```

---

## 5. State Transitions

### 5.1 Event Lifecycle

1. **Proposed**: Event submitted but not yet appended.
2. **Appended**: Event written to Ledger with sequence_id.
3. **Active**: Event contributes to derived state.
4. **Revoked**: Event nullified by subsequent Revocation; no longer contributes.

### 5.2 Imbalance Lifecycle

1. **Detected**: Evaluator identifies invariant violation.
2. **Pending**: Imbalance awaits resolution.
3. **Resolved**: Revocation appended addressing imbalance.
4. **Superseded**: State changed such that imbalance no longer applies.

---

## 6. Operational Modes

### 6.1 Accounting Mode

**Purpose**: Passive observation and detection.

**Behavior**:
- Evaluators run and report imbalances.
- Resolvers do NOT automatically generate revocations.
- All events are appended normally.

**Use Case**: Monitoring, auditing, gradual rollout of new invariants.

### 6.2 Governance Mode

**Purpose**: Active correction.

**Behavior**:
- Evaluators run and report imbalances.
- Resolvers generate revocations for detected imbalances.
- Revocations are appended to Ledger.

**Use Case**: Enforced invariant compliance.

### 6.3 Mode Transitions

Transitions MUST be:
- Explicitly triggered (no automatic escalation).
- Recorded as GOVERNANCE events in the Ledger.
- Scoped to specific jurisdictions (not necessarily global).

---

## 7. Observability

### 7.1 Internally Auditable (Full Visibility)

- Complete event log.
- All Evaluator inputs and outputs.
- All Resolver decisions and generated revocations.
- Control Plane configuration history.
- Mode transition log.

### 7.2 Opaque to Subjects

- Internal Evaluator logic (subjects see invariant IDs, not implementation).
- Resolver strategy selection rationale.
- Other subjects' events outside shared scope.
- Control Plane governance deliberations (if any).

### 7.3 Audit Requirements

- MUST support deterministic replay: given Ledger + Control Plane config, reproduce all Evaluator outputs.
- MUST support point-in-time queries: derive state as of any sequence_id.
- SHOULD support diff queries: what changed between two sequence_ids.

---

## 8. Extension Points

The following are explicitly deferred:

- **Persistence**: How the Ledger is stored.
- **Distribution**: How components are deployed across nodes.
- **Authentication**: How writers are authorized.
- **Serialization**: Wire format for events.
- **Domain Schema**: Subject and Action types for specific use cases.

These MUST be defined by instantiating systems, not by this specification.
