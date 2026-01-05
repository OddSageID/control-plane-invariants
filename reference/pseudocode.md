# Reference Pseudocode

Version: 0.1.0-draft

This document provides reference pseudocode for core IEK operations. Not executable; intended to clarify semantics.

---

## Notation

- `:=` assignment
- `->` function return type
- `for x in xs` iteration
- `if/else` conditional
- `assert` precondition check (throws on failure)
- `return` function result
- `//` comment

---

## 1. Ledger Operations

### Append Event

```
function append(ledger: Ledger, event: Event) -> uint64:
    // Validate references
    for ref in event.references:
        assert ledger.exists(ref), "INV-L003: reference must exist"

    // Assign sequence
    seq := ledger.next_sequence_id()
    assert seq > ledger.max_sequence_id(), "INV-L002: must be monotonic"

    event.sequence_id := seq

    // Append (immutable after this point - INV-L001)
    ledger.store(event)

    return seq
```

### Derive State

```
function derive_state(ledger: Ledger, scope: Scope, as_of: uint64) -> DerivedState:
    state := empty_state(scope)

    for event in ledger.events_up_to(as_of):
        if not in_scope(event, scope):
            continue

        if event.event_type == REVOCATION:
            state := apply_revocation(state, event)
        else if event.event_type == ACTION:
            state := apply_action(state, event)
        // GOVERNANCE events don't affect derived state directly

    return state
```

### Apply Revocation

```
function apply_revocation(state: DerivedState, revocation: Event) -> DerivedState:
    for target_seq in revocation.payload.targets:
        // Move event from active to revoked
        for subject in state.subjects.values():
            if target_seq in subject.active_events:
                subject.active_events.remove(target_seq)
                subject.revoked_events.append(target_seq)
                // Recompute subject attributes without target event
                subject.attributes := recompute_attributes(subject)

    return state
```

---

## 2. Evaluator Operations

### Evaluate Invariants

```
function evaluate(
    state: DerivedState,
    ruleset: Ruleset
) -> EvaluationResult:
    // INV-E002: This function MUST NOT write anything

    imbalances := []

    for invariant_id in ruleset.invariants:
        invariant := get_invariant(invariant_id)
        result := check_invariant(invariant, state)

        if not result.satisfied:
            imbalances.append(ImbalanceDescriptor{
                id: generate_id(),
                invariant_id: invariant_id,
                scope: state.scope,
                magnitude: result.deviation,
                contributing: result.contributing_events,
                detected_at: state.as_of_seq
            })

    // INV-E001: Same inputs MUST produce same imbalances
    return EvaluationResult{
        scope: state.scope,
        ruleset: ruleset.ref(),
        as_of_seq: state.as_of_seq,
        compliant: len(imbalances) == 0,
        imbalances: imbalances
    }
```

### Check Single Invariant

```
function check_invariant(
    invariant: Invariant,
    state: DerivedState
) -> InvariantCheckResult:
    // Extract parameters from state
    params := extract_parameters(invariant.parameters, state)

    // Evaluate predicate (deterministic)
    satisfied := evaluate_predicate(invariant.predicate, params)

    if satisfied:
        return InvariantCheckResult{satisfied: true}
    else:
        deviation := compute_deviation(invariant, params)
        contributors := identify_contributors(invariant, state)
        return InvariantCheckResult{
            satisfied: false,
            deviation: deviation,
            contributing_events: contributors
        }
```

---

## 3. Resolver Operations

### Plan Resolution

```
function plan_resolution(
    imbalance: ImbalanceDescriptor,
    state: DerivedState,
    strategy: Strategy
) -> ResolutionPlan:
    candidates := imbalance.contributing

    // Select targets based on strategy
    if strategy == MINIMAL:
        targets := find_minimal_revocation_set(candidates, imbalance, state)
    else if strategy == FIFO:
        targets := sort_by_sequence(candidates)  // Oldest first
    else:
        targets := custom_selection(candidates, strategy)

    // Build plan
    revocations := []
    for target_group in partition_targets(targets):
        revocations.append(PlannedRevocation{
            targets: target_group,
            reason_code: "IMBALANCE_CORRECTION"
        })

    return ResolutionPlan{
        imbalance_id: imbalance.id,
        strategy: strategy.name,
        revocations: revocations,
        estimated_effect: project_effect(state, revocations)
    }
```

### Execute Resolution

```
function execute_resolution(
    ledger: Ledger,
    plan: ResolutionPlan
) -> []uint64:
    // INV-R001: Only generate revocations, nothing additive
    // INV-R002: Must reference imbalance

    result_seqs := []

    for planned in plan.revocations:
        event := Event{
            event_type: REVOCATION,
            payload: RevocationPayload{
                targets: planned.targets,
                imbalance_ref: plan.imbalance_id,
                reason_code: planned.reason_code
            },
            references: planned.targets
        }

        seq := append(ledger, event)
        result_seqs.append(seq)

    return result_seqs
```

---

## 4. Control Plane Operations

### Activate Ruleset

```
function activate_ruleset(
    ledger: Ledger,
    scope_id: str,
    ruleset_id: str,
    version: str
) -> uint64:
    // Record as governance event
    event := Event{
        event_type: GOVERNANCE,
        payload: GovernancePayload{
            governance_type: RULESET_ACTIVATION,
            scope: ScopeSelector{type: SUBJECT_SET, value: scope_id},
            config: RulesetActivationConfig{
                ruleset_id: ruleset_id,
                version: version,
                effective_seq: ledger.head_sequence() + 1
            }
        }
    }

    return append(ledger, event)
```

### Change Mode

```
function change_mode(
    ledger: Ledger,
    scope_id: str,
    new_mode: Mode
) -> uint64:
    event := Event{
        event_type: GOVERNANCE,
        payload: GovernancePayload{
            governance_type: MODE_CHANGE,
            scope: ScopeSelector{type: SUBJECT_SET, value: scope_id},
            config: ModeChangeConfig{
                new_mode: new_mode,
                effective_seq: ledger.head_sequence() + 1
            }
        }
    }

    return append(ledger, event)
```

---

## 5. Operational Loop

### Accounting Mode Loop

```
function accounting_loop(system: System):
    while running:
        // Periodic evaluation
        for scope in system.active_scopes():
            state := derive_state(system.ledger, scope, HEAD)
            ruleset := get_active_ruleset(scope)

            result := evaluate(state, ruleset)

            // Report only; no automatic resolution
            if not result.compliant:
                log_imbalances(result.imbalances)
                emit_metrics(result)

        sleep(evaluation_interval)
```

### Governance Mode Loop

```
function governance_loop(system: System):
    while running:
        for scope in system.active_scopes():
            if get_mode(scope) != GOVERNANCE:
                continue

            state := derive_state(system.ledger, scope, HEAD)
            ruleset := get_active_ruleset(scope)

            result := evaluate(state, ruleset)

            if not result.compliant:
                for imbalance in result.imbalances:
                    // Plan and execute resolution
                    plan := plan_resolution(imbalance, state, default_strategy)
                    execute_resolution(system.ledger, plan)

                    log_resolution(imbalance, plan)

        sleep(evaluation_interval)
```

---

## 6. Utility Functions

### Idempotent Resolution Check

```
function is_already_resolved(
    ledger: Ledger,
    imbalance: ImbalanceDescriptor
) -> bool:
    // Check if revocations already address this imbalance
    for event in ledger.events_after(imbalance.detected_at):
        if event.event_type == REVOCATION:
            if event.payload.imbalance_ref == imbalance.id:
                return true
            // Also check if contributing events already revoked
            if all_revoked(imbalance.contributing, ledger):
                return true

    return false
```

### Cycle Detection

```
function would_cause_cascade(
    state: DerivedState,
    planned_revocations: []PlannedRevocation,
    max_depth: int
) -> bool:
    projected := project_effect(state, planned_revocations)

    // Re-evaluate with projected state
    result := evaluate(projected, current_ruleset())

    if result.compliant:
        return false

    if max_depth <= 0:
        return true  // Cascade detected

    // Recurse
    for imbalance in result.imbalances:
        plan := plan_resolution(imbalance, projected, MINIMAL)
        if would_cause_cascade(projected, plan.revocations, max_depth - 1):
            return true

    return false
```
