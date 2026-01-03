# SPEC: [Name]

**Version:** [X.Y]  
**Last Updated:** [YYYY-MM-DD]  
**Status:** [Draft | Review | Approved]

---

## 1. PURPOSE

[One paragraph. What problem does this solve? What does success look like?]

---

## 2. SCOPE

### IN SCOPE
- [What this covers]

### OUT OF SCOPE
- [What this explicitly excludes]
- [Adjacent concerns that are someone else's problem]

---

## 3. DEFINITIONS

| Term | Definition |
|------|------------|
| [Term] | [Precise definition] |

---

## 4. DOMAIN TYPES

Define the core data structures. Use types to make invalid states unrepresentable.
Where possible, use refined/branded types to encode constraints at the type level.

```typescript
// Example: Instead of `amount: number`, use a refined type
type PositiveInt = number & { readonly brand: unique symbol };

// Example: Use discriminated unions to make states explicit
type TaskState = 
  | { status: "pending" }
  | { status: "running"; startedAt: Date }
  | { status: "completed"; result: Result }
  | { status: "failed"; error: Error; retriesRemaining: number };

// Example: Make invalid states unrepresentable
// A completed task MUST have a result. A failed task MUST have an error.
// The type system enforces this; no runtime check needed.
```

### 4.1 Core Types

```typescript
// [Define the primary data structures here]
// [Include type aliases, interfaces, discriminated unions]
// [Use branded types for refinements: NonEmptyString, PositiveInt, ValidEmail, etc.]
```

### 4.2 Type Constraints

| Type | Constraint | Enforcement |
|------|------------|-------------|
| [Type name] | [What makes it valid] | [Type-level or runtime validation at boundary] |

### 4.3 State Machine (if applicable)

If the system has states and transitions, define them explicitly:

```
[Initial] --> [State A] --> [State B] --> [Terminal]
                  |              ^
                  v              |
              [State C] --------+
```

| From State | Event | To State | Side Effects |
|------------|-------|----------|--------------|
| [State A] | [Event X] | [State B] | [What happens] |

---

## 5. INPUTS

### 5.1 Input Types

```typescript
// [Define input types using the domain types above]
interface Input {
  // ...
}
```

### 5.2 Validation

| Field | Valid | Invalid | On Invalid |
|-------|-------|---------|------------|
| [field] | [What's accepted] | [What's rejected] | [Behavior: error, default, etc.] |

**Validation boundary:** [Where validation occurs — at API boundary, before processing, etc.]

After validation, types guarantee validity. No defensive checks needed downstream.

---

## 6. OUTPUTS

### 6.1 Output Types

```typescript
// [Define output types]
interface Output {
  // ...
}
```

### 6.2 Completeness

[When is output considered complete? What properties must it have?]

---

## 7. INVARIANTS

Properties that must hold across ALL states and operations. These are candidates for property-based tests.

### 7.1 Conservation

[Things that are preserved across operations]

| Invariant | Statement | Example |
|-----------|-----------|---------|
| [INV-1] | [Total X before operation equals total X after] | [e.g., money in system is conserved across transfers] |

### 7.2 Monotonicity

[Things that only move in one direction]

| Invariant | Statement |
|-----------|-----------|
| [INV-2] | [X never decreases / Y never increases] |

### 7.3 Structural

[Relationships between data that must always hold]

| Invariant | Statement |
|-----------|-----------|
| [INV-3] | [If A then B / A implies B] |

### 7.4 Idempotence (if applicable)

[Operations that should produce the same result when applied multiple times]

| Operation | Idempotent? | Notes |
|-----------|-------------|-------|
| [Operation X] | [Yes/No] | [f(f(x)) = f(x)?] |

---

## 8. CONSTRAINTS

### 8.0 PRIORITY ORDER

When constraints conflict, resolve in this order (highest first):

1. **[Safety]** — [Which constraints]
2. **[Correctness]** — [Which constraints]  
3. **[Completeness]** — [Which constraints]
4. **[Performance/Style]** — [Which constraints]

---

### 8.1 POSITIVE CONSTRAINTS (MUST)

**[P1] [Name]**
- Statement: [MUST ...]
- Rationale: [Why]
- Proven by: [Type / Test / Review]

**[P2] [Name]**
- Statement: [MUST ...]
- Rationale: [Why]
- Proven by: [Type / Test / Review]

---

### 8.2 NEGATIVE CONSTRAINTS (MUST NOT)

**[N1] [Name]**
- Statement: [MUST NOT ...]
- Rationale: [What failure this prevents]
- Verified by: [Type / Test / Review]

**[N2] [Name]**
- Statement: [MUST NOT ...]
- Rationale: [Why]
- Verified by: [Type / Test / Review]

---

### 8.3 PREFERENCES (SHOULD)

**[S1] [Name]**
- Statement: [SHOULD ...]
- Override: [When acceptable to violate]

---

## 9. EDGE CASES

| Case | Condition | Behavior |
|------|-----------|----------|
| Empty input | [When] | [What happens] |
| Boundary value | [At limits] | [What happens] |
| Concurrent access | [If applicable] | [What happens] |
| [Other] | [Condition] | [Behavior] |

---

## 10. VERIFICATION STRATEGY

### 10.1 What Types Prove

These properties are guaranteed at compile time. No tests needed.

| Property | Proven By |
|----------|-----------|
| [e.g., Amount is positive] | [PositiveInt type] |
| [e.g., All states handled] | [Exhaustive pattern match on TaskState] |
| [e.g., Completed tasks have results] | [Discriminated union structure] |

### 10.2 Unit Tests

Specific input-output cases for core logic.

| Test | Input | Expected Output | Constraint Verified |
|------|-------|-----------------|---------------------|
| [Test name] | [Input] | [Output] | [P1, P2, ...] |

### 10.3 Property-Based Tests

Invariants that should hold for all (or many random) inputs.

| Property | Statement | Generator |
|----------|-----------|-----------|
| [Round-trip] | `parse(serialize(x)) === x` for all valid x | [Random valid objects] |
| [Conservation] | Total before === total after | [Random operation sequences] |
| [Idempotence] | `f(f(x)) === f(x)` | [Random inputs] |
| [INV-1] | [From invariants section] | [Appropriate generator] |

### 10.4 Integration Tests

Cross-component behavior that can't be tested in isolation.

| Test | Components | Scenario | Expected Outcome |
|------|------------|----------|------------------|
| [Name] | [A, B, C] | [Setup and action] | [What should happen] |

### 10.5 What Requires Human Review

Properties that can't be automatically verified.

| Property | Review Criteria |
|----------|-----------------|
| [e.g., Code clarity] | [Can a reader understand the intent?] |
| [e.g., Appropriate abstractions] | [Are the boundaries in the right places?] |

---

## 11. EXAMPLES

### 11.1 Valid

**[V1] [Name]**
```
Input: [concrete input]
Output: [concrete output]
```
Why correct: [Which constraints satisfied, which invariants preserved]

**[V2] [Name]**
```
Input: [concrete input]  
Output: [concrete output]
```
Why correct: [Explanation]

### 11.2 Invalid

**[I1] [Name]**
```
Input: [concrete input]
Wrong output: [what would be wrong]
```
Violation: **[N1]** — [How this violates the constraint]

**[I2] [Name]**
```
Input: [concrete input]
Wrong output: [what would be wrong]
```
Violation: **[P2]** — [How this fails to satisfy the constraint]

### 11.3 Edge Cases

**[E1] [Name]**
```
Input: [edge case input]
Output: [expected output]
```
Why: [Explanation of edge case handling]

---

## 12. ASSUMPTIONS

### 12.1 Environmental
- [What's assumed about execution environment]
- [Dependencies assumed available]

### 12.2 Input Quality
- [What's assumed about input correctness beyond validation]

### 12.3 If Assumptions Fail

| Assumption | If Violated |
|------------|-------------|
| [Assumption] | [Behavior] |

---

## 13. OPEN QUESTIONS

- [ ] [Unresolved ambiguity needing human decision]

---

## CHANGELOG

| Version | Date | Changes |
|---------|------|---------|
| [X.Y] | [Date] | [What changed] |
