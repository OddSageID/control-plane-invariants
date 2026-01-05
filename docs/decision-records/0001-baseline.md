# ADR-0001: Baseline Architecture Decisions

Status: **Accepted**
Date: 2026-01-05

---

## Context

The Invariant Enforcement Kit (IEK) is a reference architecture for ledger-based state correction with a separate governance/control plane. Before implementation begins, we must lock foundational decisions to prevent scope drift and ensure implementations remain interoperable.

This ADR establishes what is decided (normative) versus what is deferred (to be defined by instantiating systems).

---

## Decided (Normative)

These decisions are locked. Changes require a new ADR with explicit rationale.

### D1: Append-Only Ledger

The Ledger is append-only. Events cannot be modified or deleted after sequencing. State correction is achieved through Revocation events, not mutation.

**Rationale**: Enables deterministic replay, audit, and eliminates hidden state.

**Invariants**: INV-L001, INV-L002, INV-L003.

---

### D2: Idempotency Keys

- Events carry `event_id` (client-assigned, globally unique).
- Duplicate appends with same `event_id` return existing `sequence_id`.
- Resolution plans carry `(evaluation_id, plan_id)` for execution idempotency.

**Rationale**: Enables at-least-once delivery; prevents duplicate state effects.

**Invariants**: INV-I001, INV-I002.

---

### D3: Replay Modes

Three modes: LIVE, AUDIT, SIMULATION.

- LIVE: Authoritative governance; retroactivity guard applies.
- AUDIT: Historical verification; non-authoritative results.
- SIMULATION: What-if analysis; non-authoritative, no production writes.

**Rationale**: Separates operational governance from verification and experimentation.

**Specification**: semantics.md §4.

---

### D4: Non-Retroactivity in LIVE Mode

Evaluations in LIVE mode MUST NOT reference ruleset versions created after the `occurred_at` of evaluated events.

**Rationale**: Prevents post-hoc rule application; preserves fairness and auditability.

**Specification**: architecture.md §4.3.

---

### D5: Plan Safety Rails

Every ResolutionPlan MUST declare:
- `blast_radius`: affected scopes, subjects, capabilities.
- `rollback_strategy`: COMPENSATE, IRREVERSIBLE, or MANUAL.
- `cascade_limits`: max_depth, max_fan_out, cycle_detected.

Irreversible plans require elevated governance mode. Cycle-detected plans require manual override.

**Rationale**: Prevents unbounded correction cascades; makes impact explicit.

**Specification**: schemas.md ResolutionPlan.

---

### D6: Capability Model

Revocation targets Capabilities, not Subjects directly.

- Capabilities are derived from CAPABILITY_GRANTED events.
- Revocation nullifies granting events via CAPABILITY_REVOKED events.
- Effective capabilities = granted - revoked.

**Rationale**: Provides concrete target for subtractive correction; enables fine-grained revocation.

**Specification**: schemas.md Capability, events.md capability lifecycle.

---

### D7: Trigger Rate Limiting

Control Plane governance actions are bounded by TriggerBudget per scope per window.

**Rationale**: Prevents cascade explosions and resource exhaustion (T7, T9).

**Invariant**: INV-T001.

---

### D8: Normative Layer Separation

Three layers, distinct and non-conflatable:
1. Invariants (structural; timeless)
2. Policies/Rulesets (governance; versioned, scoped)
3. Resolvers (mechanisms; pluggable)

**Rationale**: Prevents policy from being embedded in invariant logic, and resolution logic from being embedded in policy.

**Specification**: architecture.md §3.

---

### D9: Deterministic State Derivation

`derive_state(scope, seq)` is deterministic given Ledger + derivation_version.

**Rationale**: Enables audit replay and cross-instance verification.

**Invariants**: INV-E001, INV-I003, INV-I004.

---

### D10: Event Envelope Contract

Every event MUST carry:
- `event_id`: client-assigned, globally unique
- `sequence_id`: ledger-assigned, monotonic
- `event_type`: discriminator
- `occurred_at`: event-time timestamp
- `observed_at`: processing-time timestamp
- `derivation_version`: algorithm version for state computation

Evaluations and resolutions MUST additionally carry `ruleset_ref`.

**Rationale**: Eliminates ambiguity across implementations; enables interoperability.

**Specification**: events.md.

---

## Deferred (Implementation-Defined)

These are explicitly out of scope for the core specification. Instantiating systems MUST define them.

### Storage Engine

How the Ledger is persisted. Options include:
- Single-node append log
- Distributed log (Kafka, Pulsar)
- Database with append-only table
- Content-addressed store

**Constraint**: MUST satisfy INV-L001, INV-L002, INV-L003.

---

### Transport / Wire Format

How events are transmitted between components. Options include:
- gRPC / Protocol Buffers
- HTTP / JSON
- Message queue
- In-process calls

**Constraint**: MUST preserve event envelope fields; MUST support idempotency key transmission.

---

### Authentication / Authorization

How writers are authenticated and authorized to append events.

**Constraint**: MUST support attribution (every event traceable to authenticated source).

---

### Consistency Level Selection

Whether the system provides:
- Linearizability (strong consistency)
- Causal consistency
- Eventual consistency with conflict resolution

**Constraint**: MUST document chosen consistency model; MUST satisfy ordering guarantees in semantics.md §3.

---

### Policy Language

How invariants and rulesets are expressed. Options include:
- Datalog / Rego
- Custom DSL
- Embedded code (with sandboxing)
- JSON/YAML declarative rules

**Constraint**: MUST produce deterministic evaluation results (INV-E001).

---

### Deployment Topology

How components are deployed. Options include:
- Monolith
- Microservices
- Serverless functions
- Hybrid

**Constraint**: MUST respect component boundaries (Evaluator read-only, Resolver write-only via Revocations).

---

### Domain Schema

Subject types, Action types, and Capability types for specific use cases.

**Constraint**: MUST NOT include built-in human-targeting primitives (per project exclusions).

---

## Consequences

### Positive

- Implementations can diverge on storage, transport, and deployment while remaining interoperable at the semantic level.
- Core invariants are testable without implementation details.
- Future ADRs can promote deferred items to decided status with explicit rationale.

### Negative

- Instantiating systems must make more decisions before becoming operational.
- Interoperability testing requires conformance suite, not just shared code.

### Neutral

- This ADR will require revision if fundamental assumptions (e.g., append-only) prove unworkable in practice.

---

## References

- architecture.md
- invariants.md
- semantics.md
- schemas.md
- events.md
- threat-model.md
