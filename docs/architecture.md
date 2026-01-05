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

## 3. Normative Layer Separation

The system distinguishes three normative layers. Conflating them is a design defect.

### 3.1 Invariants (Structural Constraints)

**Definition**: Predicates that MUST hold at all times, independent of policy or configuration.

**Properties**:
- Timeless: not versioned, not scoped, not changeable without system redesign.
- Structural: encode what the system cannot represent, not what it should prefer.
- Examples: referential integrity (INV-L003), append-only semantics (INV-L001), determinism (INV-E001).

**Enforcement**: Violated invariants indicate system defects, not policy violations.

### 3.2 Policies / Rulesets (Governance Constraints)

**Definition**: Versioned, scoped predicates that define what constitutes an imbalance within a jurisdiction.

**Properties**:
- Mutable: new versions can be created (old versions are immutable).
- Scoped: apply to specific jurisdictions, not globally.
- Temporal: have effective dates; do not apply retroactively unless explicit replay mode.
- Examples: "Subject X may not hold more than N of resource Y," "Allocations must balance to zero."

**Enforcement**: Violated policies produce ImbalanceDescriptors, triggering Resolver action in Governance Mode.

### 3.3 Resolver Mechanisms (Correction Strategies)

**Definition**: Strategies for generating revocations to restore policy compliance.

**Properties**:
- Pluggable: multiple strategies may exist (MINIMAL, FIFO, CUSTOM).
- Policy-bound: a Ruleset MAY specify which Resolver strategies are permitted.
- Stateless: strategies are pure functions from (Imbalance, DerivedState, Config) → ResolutionPlan.

**Enforcement**: Resolver selection is a governance decision, not hardcoded.

### 3.4 Layer Interactions

```
INVARIANTS (structural)
    │
    │ define what is representable
    ▼
POLICIES (governance)
    │
    │ define what is compliant
    ▼
RESOLVERS (mechanisms)
    │
    │ define how to restore compliance
    ▼
LEDGER (truth)
```

**Invariant → Policy**: Invariants constrain what policies can express. A policy cannot require a state that violates a structural invariant.

**Policy → Resolver**: Policies declare imbalances. Resolvers correct them. A policy MUST NOT embed resolution logic.

**Resolver → Ledger**: Resolvers only interact with the Ledger through Revocation events. They cannot bypass invariants.

---

## 4. Versioning and Temporal Semantics

### 4.1 Version Immutability

- `ruleset_version` is immutable once created.
- Changes to a ruleset produce a new version; old versions persist unchanged.
- Invariant IDs are stable; invariant definitions may only change via system redesign (not governance).

### 4.2 Evaluation Binding

Every EvaluationResult MUST record:
- `ruleset_id`: which ruleset was applied
- `ruleset_version`: exact version used
- `invariant_ids`: list of invariants checked (derived from ruleset)
- `as_of_seq`: the sequence point of derived state

This tuple uniquely identifies the evaluation context for audit purposes.

### 4.3 Retroactivity Guard

**Structural Constraint**: An evaluation MUST NOT reference a ruleset version whose `created_at` timestamp is after the `timestamp` of the events being evaluated, unless:
1. Explicit `replay_mode: AUDIT` or `replay_mode: SIMULATION` is declared, AND
2. The evaluation result is marked `non_authoritative: true`.

This prevents retroactive rule application in live governance.

### 4.4 Migration Semantics

When a new ruleset version is activated:
1. A GOVERNANCE event is appended with `effective_seq`.
2. Events with `sequence_id >= effective_seq` are evaluated under the new version.
3. Events with `sequence_id < effective_seq` remain evaluated under the prior version.
4. Re-evaluation of historical events requires explicit replay mode invocation.

---

## 5. Control Plane Trigger Semantics

### 5.1 Trigger Interface

Triggers define when the Control Plane initiates evaluation or mode transitions.

```
TriggerCondition {
  trigger_id     : str                # Stable identifier
  predicate      : MetricPredicate    # Condition over derived metrics
  scope          : ScopeSelector      # Jurisdiction
  action         : TriggerAction      # What happens when triggered
  cooldown       : Duration           # Minimum time between firings
}

MetricPredicate {
  metric_name    : str                # e.g., "imbalance_count", "event_rate"
  operator       : GT | LT | EQ | GTE | LTE
  threshold      : float64
}

TriggerAction = EVALUATE | ENTER_GOVERNANCE | EXIT_GOVERNANCE | ALERT
```

### 5.2 Trigger Budget

Triggers are rate-limited to prevent cascade failures (see T7, T9 in threat model).

```
TriggerBudget {
  scope          : ScopeSelector
  max_evaluations_per_window : uint32
  max_revocations_per_window : uint32
  window_duration : Duration
  overflow_action : QUEUE | DROP | ALERT
}
```

**Constraints**:
- When budget is exhausted, new triggers MUST follow `overflow_action`.
- Budget consumption MUST be logged for audit.
- Budget limits apply per-scope; global limits MAY also be defined.

### 5.3 Attribution

All governance actions are attributable:
- Every GOVERNANCE event records `triggered_by: trigger_id | MANUAL`.
- Every Revocation records `authorized_by: ruleset_id, ruleset_version`.
- Control Plane decisions are logged even if opaque to subjects.

---

## 6. Interfaces

### 6.1 Events (Ledger ← All Writers)

```
Event {
  sequence_id    : monotonic identifier
  event_type     : ACTION | REVOCATION | GOVERNANCE
  payload        : domain-specific content
  references     : list of prior sequence_ids (optional)
  timestamp      : logical or wall-clock (advisory only)
}
```

### 6.2 Commands (Control Plane → Processing Layer)

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

### 6.3 Queries (Evaluator → Ledger)

```
StateQuery {
  scope          : jurisdiction selector
  as_of_sequence : sequence_id
}

Returns: DerivedState (computed projection of events)
```

### 6.4 Outputs

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

## 7. Data Model (Conceptual)

### 7.1 Core Entities

| Entity | Description |
|--------|-------------|
| Event | Immutable record in the Ledger |
| Subject | Abstract entity with state (no default schema) |
| Invariant | Named predicate over derived state |
| Scope | Jurisdiction boundary; set of subjects or event streams |
| Ruleset | Collection of invariants active in a scope |

### 7.2 State Derivation

Derived state is computed, not stored:

```
DerivedState(scope, seq) = fold(
  filter(events, scope),
  initial_state,
  apply_event
) where event.sequence_id <= seq
```

### 7.3 Revocation Semantics

Revocations do not delete events. They append a new event that marks prior events as nullified for state derivation:

```
apply_event(state, revocation) =
  state - effects_of(revocation.targets)
```

---

## 8. State Transitions

### 8.1 Event Lifecycle

1. **Proposed**: Event submitted but not yet appended.
2. **Appended**: Event written to Ledger with sequence_id.
3. **Active**: Event contributes to derived state.
4. **Revoked**: Event nullified by subsequent Revocation; no longer contributes.

### 8.2 Imbalance Lifecycle

1. **Detected**: Evaluator identifies invariant violation.
2. **Pending**: Imbalance awaits resolution.
3. **Resolved**: Revocation appended addressing imbalance.
4. **Superseded**: State changed such that imbalance no longer applies.

---

## 9. Operational Modes

### 9.1 Accounting Mode

**Purpose**: Passive observation and detection.

**Behavior**:
- Evaluators run and report imbalances.
- Resolvers do NOT automatically generate revocations.
- All events are appended normally.

**Use Case**: Monitoring, auditing, gradual rollout of new invariants.

### 9.2 Governance Mode

**Purpose**: Active correction.

**Behavior**:
- Evaluators run and report imbalances.
- Resolvers generate revocations for detected imbalances.
- Revocations are appended to Ledger.

**Use Case**: Enforced invariant compliance.

### 9.3 Mode Transitions

Transitions MUST be:
- Explicitly triggered (no automatic escalation).
- Recorded as GOVERNANCE events in the Ledger.
- Scoped to specific jurisdictions (not necessarily global).

---

## 10. Observability

### 10.1 Internally Auditable (Full Visibility)

- Complete event log.
- All Evaluator inputs and outputs.
- All Resolver decisions and generated revocations.
- Control Plane configuration history.
- Mode transition log.

### 10.2 Opaque to Subjects

- Internal Evaluator logic (subjects see invariant IDs, not implementation).
- Resolver strategy selection rationale.
- Other subjects' events outside shared scope.
- Control Plane governance deliberations (if any).

### 10.3 Audit Requirements

- MUST support deterministic replay: given Ledger + Control Plane config, reproduce all Evaluator outputs.
- MUST support point-in-time queries: derive state as of any sequence_id.
- SHOULD support diff queries: what changed between two sequence_ids.

---

## 11. Extension Points

The following are explicitly deferred:

- **Persistence**: How the Ledger is stored.
- **Distribution**: How components are deployed across nodes.
- **Authentication**: How writers are authorized.
- **Serialization**: Wire format for events.
- **Domain Schema**: Subject and Action types for specific use cases.

These MUST be defined by instantiating systems, not by this specification.
