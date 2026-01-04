# Invariant Enforcement Kit (IEK)

## Definition

The Invariant Enforcement Kit is a reference architecture for systems that maintain state correctness through ledger-based accounting and governance-triggered correction. It separates the **truth layer** (what happened) from the **control plane** (what responses are permitted), enabling testable, auditable state management without embedding policy in the data model.

---

## Core Principles

1. **Separation of Record and Response**: The Ledger records facts. The Control Plane determines permissible responses. These MUST NOT be conflated.
2. **Append-Only Truth**: State is derived from an immutable event log. Corrections are new events, not mutations.
3. **Mechanical Evaluation**: Invariant checking is deterministic and context-free. Evaluators MUST NOT interpret intent, motive, or ontological status.
4. **Subtractive Correction**: Resolvers correct imbalances by revocation or subtraction, not by addition of obligations or penalties.
5. **Jurisdiction as Signal**: The Control Plane signals which rules apply to which scopes. It does not define truth.
6. **No Implicit State**: All state MUST be derivable from the Ledger. Hidden state is a defect.
7. **Testability Over Completeness**: Primitives MUST be testable in isolation before composition.
8. **Minimal Authority**: Components operate with least privilege. The Evaluator cannot write; the Resolver cannot read outside its scope.

---

## Explicit Exclusions

1. **No Moral Adjudication**: The system does not determine right/wrong, good/bad, or deserving/undeserving.
2. **No Rehabilitation or Persuasion**: Correction is mechanical state adjustment, not behavioral modification.
3. **No Intent Modeling**: Evaluators do not infer or consider why an action occurred.
4. **No Human Targeting Primitives**: No built-in concepts for identifying, profiling, or tracking natural persons.
5. **No Coercive Enforcement Mechanisms**: Revocation removes state; it does not compel behavior.
6. **No Survivability Guarantees**: The optional Constraint Engine provides no guarantees that constraints are satisfiable.
7. **No Implicit Hierarchies**: Authority flows from explicit configuration, not embedded assumptions.
8. **No Real-Time Intervention**: This is an accounting system, not a prevention system.

---

## Glossary

| Term | Definition |
|------|------------|
| **Ledger** | Append-only event log constituting the system's truth layer. All state is derived from the Ledger. |
| **Evaluator** | Stateless component that checks invariants against derived state. Returns boolean or imbalance descriptor. |
| **Resolver** | Component that generates correction events (revocations) to restore invariant compliance. |
| **Control Plane** | Governance layer that defines active rulesets, jurisdiction boundaries, and trigger conditions. |
| **Subject** | Abstract entity whose state is tracked. Domain-specific; no default semantics. |
| **Action** | An event representing a state transition. Immutable once appended. |
| **Imbalance** | A detected invariant violation. Describes what is out of spec, not why. |
| **Revocation** | A correction event that subtracts or nullifies prior state contributions. |

---

## What Success Looks Like

1. All system state is reproducible from the Ledger alone.
2. Invariant violations are detectable by replaying the event log through Evaluators.
3. Corrections are auditable: every revocation references the imbalance it addresses.
4. Control Plane configuration is versioned and its effects are deterministic.
5. The architecture can be instantiated for any domain without modifying core primitives.

---

## What Failure Looks Like

1. State exists that cannot be derived from the Ledger (hidden state).
2. Evaluators produce different results for the same input (non-determinism).
3. Corrections create new obligations rather than removing state (scope creep).
4. Control Plane rules are embedded in Evaluator logic (conflation).
5. The system is instantiated for human behavioral control (misuse).

---

## Repository Structure

```
/docs
  architecture.md    # System design specification
  invariants.md      # Core invariant definitions (MUST statements)
  threat-model.md    # Abuse, misconfiguration, capture risks

/spec
  events.md          # Event type definitions
  schemas.md         # Conceptual data schemas

/reference
  pseudocode.md      # Reference implementation sketches

/sim                 # Placeholder: simulation harness
/tests               # Placeholder: test suites
```

---

## Status

**Phase**: Foundation
**Maturity**: Specification draft; no reference implementation.

---

## License

This specification is provided for architectural reference. See LICENSE for terms.
