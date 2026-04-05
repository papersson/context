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

Properties guaranteed at compile time. No tests needed.

| Property | Mechanism |
|----------|-----------|
| [e.g., Amount is positive] | [PositiveInt branded type] |
| [e.g., All states handled] | [Exhaustive match on discriminated union] |

### 10.2 Unit Test Scenarios

Concrete test cases for isolated components. Each row is a test.

| ID | Component | Scenario | Given | When | Then |
|----|-----------|----------|-------|------|------|
| U1 | [Module] | [What's being tested] | [Setup/preconditions] | [Action] | [Expected outcome] |
| U2 | [Module] | [What's being tested] | [Setup/preconditions] | [Action] | [Expected outcome] |

### 10.3 Mock Definitions

How to isolate components for testing.

```typescript
// Example: Mock for external LLM client
const mockLLMClient: LLMClient = {
  complete: vi.fn().mockResolvedValue({
    response: 'mocked response',
    tokensUsed: 100,
  }),
};

// Example: Mock for database
const mockDB: Database = {
  query: vi.fn().mockResolvedValue([{ id: 1, name: 'test' }]),
};
```

| Dependency | Mock Type | Behavior |
|------------|-----------|----------|
| [External API] | [Mock/Stub/Fake] | [What it returns, when] |
| [Database] | [In-memory fake] | [How it behaves] |

### 10.4 Property-Based Test Scenarios

Invariants tested with random inputs.

| ID | Property | Generator | Assertion |
|----|----------|-----------|-----------|
| P1 | [Round-trip] | [Random valid inputs] | `parse(serialize(x)) === x` |
| P2 | [Conservation] | [Random operation sequences] | `sum(before) === sum(after)` |

### 10.5 Integration Test Scenarios

Tests crossing component boundaries. Still automated, may use mocks for external services.

| ID | Components | Scenario | Setup | Action | Expected |
|----|------------|----------|-------|--------|----------|
| I1 | [A → B → C] | [Flow being tested] | [State setup] | [Trigger] | [Outcome + trace] |

### 10.6 Manual Verification Checklist

**CRITICAL: The system is NOT complete until all manual verification items pass.**

These require real credentials, real external services, or human judgment. They cannot be automated or mocked.

| ID | Verification | Command/Steps | Expected Outcome | Status |
|----|--------------|---------------|------------------|--------|
| M1 | End-to-end happy path | `[actual command to run]` | [What success looks like] | ⬜ |
| M2 | End-to-end with real [external service] | `[command with real API key]` | [Expected behavior] | ⬜ |
| M3 | Error handling with real failures | [How to trigger real failure] | [Expected recovery] | ⬜ |
| M4 | [Edge case requiring manual check] | [Steps] | [Expected] | ⬜ |

**Completion criteria:** All M* items must be checked ✅ before declaring the implementation complete.

### 10.7 Test Coverage Requirements

| Category | Minimum | Notes |
|----------|---------|-------|
| Unit tests | [N tests or % coverage] | [Which modules must be covered] |
| Integration tests | [N scenarios] | [Which flows must be covered] |
| Property tests | [N properties] | [Which invariants] |
| Manual verification | **100%** | All M* items must pass |

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

---

# EXTENSIONS

Include relevant extensions based on your system type. Delete sections that don't apply.

---

## EXT-A: PROJECT SETUP

Use when: Implementing new software from scratch. Skip for modifications to existing codebases.

### A.1 Directory Structure

```
project-root/
├── src/
│   ├── [module]/
│   └── index.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── property/
├── docs/
├── package.json
├── tsconfig.json
└── README.md
```

### A.2 Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| [package] | [^x.y.z] | [Why needed] |

### A.3 Build Configuration

```jsonc
// tsconfig.json key settings
{
  "compilerOptions": {
    "strict": true,
    // [other requirements]
  }
}
```

### A.4 Dev Tooling

| Tool | Configuration | Purpose |
|------|---------------|---------|
| [eslint/prettier/etc] | [config file or key settings] | [Why] |

### A.5 Scripts

| Script | Command | Purpose |
|--------|---------|---------|
| `build` | [command] | [What it does] |
| `test` | [command] | [What it does] |
| `dev` | [command] | [What it does] |

---

## EXT-B: MODULE ARCHITECTURE

Use when: System has multiple components that need clear boundaries. Skip for single-file utilities.

### B.1 Module Boundaries

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Module A   │────▶│  Module B   │────▶│  Module C   │
│             │     │             │     │             │
│ [purpose]   │     │ [purpose]   │     │ [purpose]   │
└─────────────┘     └─────────────┘     └─────────────┘
```

### B.2 Interface Contracts

```typescript
// Module A exports this interface
interface ModuleAInterface {
  // [methods with full signatures]
}

// Module B exports this interface
interface ModuleBInterface {
  // [methods with full signatures]
}
```

### B.3 Dependency Rules

| Rule | Description |
|------|-------------|
| [e.g., "No circular deps"] | [Enforcement mechanism] |
| [e.g., "Domain doesn't import infra"] | [Why] |

### B.4 Testability Seams

| Module | How to Test in Isolation |
|--------|--------------------------|
| [Module A] | [Mock these dependencies] |
| [Module B] | [Inject these test doubles] |

---

## EXT-C: EXTERNAL DEPENDENCIES

Use when: System calls external services (APIs, databases, file systems). Skip for pure computation.

### C.1 External Interfaces

```typescript
// Interface that external dependency must satisfy
interface ExternalServiceClient {
  // [methods]
}
```

### C.2 Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Network timeout | [How detected] | [Retry? Fail? Degrade?] |
| Rate limited (429) | [How detected] | [Backoff strategy] |
| Invalid response | [Validation] | [Error handling] |
| Service unavailable | [How detected] | [Fallback behavior] |

### C.3 Retry Strategy

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Max retries | [N] | [Why] |
| Initial backoff | [Xms] | [Why] |
| Backoff multiplier | [Y] | [Why] |
| Max backoff | [Zms] | [Why] |
| Retryable errors | [List] | [Which errors trigger retry] |

### C.4 Mock/Stub Strategy

| External Dependency | Test Double | Behavior |
|---------------------|-------------|----------|
| [Service A] | [Mock/Stub/Fake] | [What it returns/does] |

---

## EXT-D: CONCURRENCY MODEL

Use when: System has async operations, parallelism, or shared state. Skip for synchronous single-threaded code.

### D.1 Threading Model

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Execution model | [Single-threaded / Multi-threaded / Event loop] | [Why] |
| Shared state | [None / Mutex-protected / Lock-free] | [Why] |

### D.2 Async Boundaries

| Operation | Sync/Async | Blocking? |
|-----------|------------|-----------|
| [Operation A] | [Async] | [No] |
| [Operation B] | [Sync] | [Yes, for Xms max] |

### D.3 Parallelism

| Operation | Parallel? | Concurrency Limit |
|-----------|-----------|-------------------|
| [e.g., Sub-LM calls] | [Yes/No] | [N concurrent max] |

### D.4 Cancellation

| Trigger | What Gets Cancelled | Cleanup |
|---------|---------------------|---------|
| [User request] | [In-flight operations] | [Resources released] |
| [Timeout] | [Current operation] | [Partial state handling] |

### D.5 Race Condition Mitigations

| Potential Race | Mitigation |
|----------------|------------|
| [Describe race] | [How prevented] |

---

## EXT-E: ERROR TAXONOMY

Use when: System has multiple error types requiring different handling. Skip for simple pass/fail systems.

### E.1 Error Classification

```typescript
type AppError =
  | { type: 'validation'; field: string; message: string }
  | { type: 'external_service'; service: string; cause: Error; retryable: boolean }
  | { type: 'internal'; message: string; stack: string }
  | { type: 'timeout'; operation: string; durationMs: number };
```

### E.2 Error Handling Matrix

| Error Type | Logged? | Retried? | User Message | Recovery |
|------------|---------|----------|--------------|----------|
| [validation] | [No] | [No] | [Specific field error] | [None] |
| [external_service] | [Yes] | [If retryable] | [Generic "try again"] | [Retry with backoff] |

### E.3 Error Propagation

| Layer | Catches | Transforms To | Passes Up |
|-------|---------|---------------|-----------|
| [Data layer] | [DB errors] | [AppError] | [Yes] |
| [Service layer] | [AppError] | [—] | [Yes] |
| [API layer] | [All] | [HTTP response] | [No] |

---

## EXT-F: SECURITY MODEL

Use when: System handles untrusted input, sensitive data, or runs untrusted code. Skip for internal tools with trusted users.

### F.1 Trust Boundaries

```
┌─────────────────────────────────────────┐
│            TRUSTED ZONE                 │
│  ┌─────────┐         ┌─────────┐       │
│  │ Config  │         │  Core   │       │
│  └─────────┘         │  Logic  │       │
│                      └────┬────┘       │
│                           │            │
├───────────────────────────┼────────────┤ ◄── Trust Boundary
│                           ▼            │
│            UNTRUSTED ZONE              │
│  ┌─────────┐         ┌─────────┐       │
│  │  User   │         │External │       │
│  │  Input  │         │   API   │       │
│  └─────────┘         └─────────┘       │
└─────────────────────────────────────────┘
```

### F.2 Threat Enumeration

| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| [e.g., Prompt injection] | [High/Med/Low] | [High/Med/Low] | [How addressed] |
| [e.g., Code injection] | [High/Med/Low] | [High/Med/Low] | [How addressed] |

### F.3 Input Sanitization

| Input Source | Sanitization | Validation |
|--------------|--------------|------------|
| [User input] | [Escaping/encoding] | [Schema validation] |

### F.4 Secrets Management

| Secret | Storage | Access |
|--------|---------|--------|
| [API keys] | [Env vars / Secret manager] | [Which components] |

---

## EXT-G: API CONTRACT

Use when: Building a service with external API. Skip for libraries or CLI tools.

### G.1 Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| [POST] | [/api/v1/resource] | [What it does] |

### G.2 Request/Response Schemas

```typescript
// POST /api/v1/resource
interface CreateResourceRequest {
  // [fields]
}

interface CreateResourceResponse {
  // [fields]
}
```

### G.3 Error Responses

| Status | Condition | Response Body |
|--------|-----------|---------------|
| 400 | [Invalid input] | `{ error: string, field?: string }` |
| 401 | [Unauthorized] | `{ error: string }` |
| 429 | [Rate limited] | `{ error: string, retryAfter: number }` |
| 500 | [Internal error] | `{ error: string, requestId: string }` |

### G.4 Authentication

| Mechanism | Details |
|-----------|---------|
| [Bearer token / API key / etc] | [How provided, how validated] |

### G.5 Rate Limiting

| Limit | Scope | Response |
|-------|-------|----------|
| [N requests/minute] | [Per API key / Per IP] | [429 with Retry-After] |

---

## EXT-H: DISTRIBUTED SYSTEMS

Use when: System spans multiple nodes, requires coordination, or has network partitions as a failure mode. Skip for single-process applications.

### H.1 Topology

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node A  │────▶│ Node B  │────▶│ Node C  │
│         │◀────│         │◀────│         │
└─────────┘     └─────────┘     └─────────┘
```

### H.2 Consistency Model

| Guarantee | Scope | Mechanism |
|-----------|-------|-----------|
| [Linearizable / Sequential / Eventual] | [Which operations] | [How achieved] |

### H.3 Failure Modes

| Failure | Detection | Response | Data Impact |
|---------|-----------|----------|-------------|
| Node crash | [Heartbeat timeout] | [Failover to replica] | [No data loss if...] |
| Network partition | [Timeout] | [Partition tolerance strategy] | [Consistency implications] |
| Split brain | [Quorum check] | [Fencing] | [Prevention mechanism] |

### H.4 Ordering Guarantees

| Guarantee | Scope | Mechanism |
|-----------|-------|-----------|
| [Causal / Total / FIFO / None] | [Which operations] | [How enforced] |

### H.5 Consensus (if applicable)

| Aspect | Choice |
|--------|--------|
| Algorithm | [Raft / Paxos / etc] |
| Quorum size | [N/2 + 1] |
| Leader election | [Mechanism] |

### H.6 Distributed Verification Strategy

| Technique | What It Tests | Tool |
|-----------|---------------|------|
| Deterministic simulation | [All interleavings] | [Framework] |
| Fault injection | [Failure recovery] | [Jepsen / Custom] |
| Property-based (distributed) | [Invariants under concurrency] | [Framework] |

---

## CHANGELOG

| Version | Date | Changes |
|---------|------|---------|
| [X.Y] | [Date] | [What changed] |
