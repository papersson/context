# SPEC: [Name]

**Version:** [X.Y]  
**Last Updated:** [YYYY-MM-DD]  
**Author:** [Name]  
**Status:** [Draft | Review | Approved]

---

## 1. PURPOSE

[One paragraph maximum. What is this system/feature/task? What problem does it solve? Who or what consumes the output?]

---

## 2. SCOPE

### IN SCOPE

- [What this specification covers]
- [Specific capabilities included]
- [Boundaries of responsibility]

### OUT OF SCOPE

- [What this explicitly does NOT cover]
- [Adjacent concerns deliberately excluded]
- [Future work not addressed here]

---

## 3. DEFINITIONS

| Term | Definition |
|------|------------|
| [Term A] | [Precise, unambiguous definition. No circular references.] |
| [Term B] | [Definition. If referencing another term, that term MUST be defined above.] |

---

## 4. INPUTS

### 4.1 Input Description

[What the system/agent receives. Overall structure and context.]

### 4.2 Input Schema

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| [field_name] | [type] | [Yes/No] | [Valid values, ranges, patterns] | [What this field represents] |

### 4.3 Input Validation

- **Valid input:** [What constitutes valid input]
- **Invalid input:** [What constitutes invalid input]
- **On invalid input:** [Exact behavior when input is invalid]

---

## 5. OUTPUTS

### 5.1 Output Description

[What the system/agent produces. Overall structure and purpose.]

### 5.2 Output Schema

| Field | Type | Guaranteed | Description |
|-------|------|------------|-------------|
| [field_name] | [type] | [Always/Conditional] | [What this field represents] |

### 5.3 Completeness Criteria

[When is the output considered complete? What properties MUST the output have to be acceptable?]

---

## 6. CONSTRAINTS

### 6.0 PRIORITY ORDER

When constraints conflict, resolve in this order (highest priority first):

1. **[Safety/Security]** — [Constraints in this class]
2. **[Correctness]** — [Constraints in this class]
3. **[Completeness]** — [Constraints in this class]
4. **[Performance/Style]** — [Constraints in this class]

---

### 6.1 POSITIVE CONSTRAINTS (MUST)

#### [P1] [Short name]

**Statement:** [The system/output MUST [precise requirement].]

**Rationale:** [Why this constraint exists. What goes wrong without it.]

**Verification:** [How to determine if this constraint is satisfied. Must be testable.]

---

#### [P2] [Short name]

**Statement:** [The system/output MUST [precise requirement].]

**Rationale:** [Why this constraint exists.]

**Verification:** [How to test.]

---

### 6.2 NEGATIVE CONSTRAINTS (MUST NOT)

#### [N1] [Short name]

**Statement:** [The system/output MUST NOT [precise prohibition].]

**Rationale:** [What failure mode this prevents. Why this is dangerous/wrong.]

**Verification:** [How to determine if this constraint is violated.]

---

#### [N2] [Short name]

**Statement:** [The system/output MUST NOT [precise prohibition].]

**Rationale:** [What failure mode this prevents.]

**Verification:** [How to test for violation.]

---

### 6.3 PREFERENCES (SHOULD / SHOULD NOT)

#### [S1] [Short name]

**Statement:** [The system/output SHOULD (NOT) [preference].]

**Rationale:** [Why this is preferred.]

**Override condition:** [Circumstances under which violating this preference is acceptable.]

---

## 7. EDGE CASES

| Case | Condition | Required Behavior |
|------|-----------|-------------------|
| Empty input | [When input is empty/null/zero-length] | [Exact behavior] |
| Boundary value | [At limits of valid ranges] | [Exact behavior] |
| Malformed input | [Input that is structurally invalid] | [Exact behavior] |
| [Other edge case] | [Condition] | [Behavior] |

---

## 8. EXAMPLES

### 8.1 Valid Examples (meets spec)

#### Example V1: [Name]

**Input:**
```
[Concrete input]
```

**Output:**
```
[Concrete expected output]
```

**Why correct:** [Explanation of which constraints are satisfied and how]

---

#### Example V2: [Name]

**Input:**
```
[Concrete input]
```

**Output:**
```
[Concrete expected output]
```

**Why correct:** [Explanation]

---

### 8.2 Invalid Examples (violates spec)

#### Example I1: [Name]

**Input:**
```
[Concrete input]
```

**Incorrect output:**
```
[Output that would be WRONG]
```

**Violation:** FAILS **[N1]** — [Explanation of how this violates the constraint]

---

#### Example I2: [Name]

**Input:**
```
[Concrete input]
```

**Incorrect output:**
```
[Output that would be WRONG]
```

**Violation:** FAILS **[P2]** — [Explanation of how this fails to satisfy the constraint]

---

### 8.3 Edge Case Examples

#### Example E1: [Name]

**Input:**
```
[Edge case input]
```

**Output:**
```
[Expected output for edge case]
```

**Why:** [Explanation of edge case handling]

---

## 9. ASSUMPTIONS

### 9.1 Environmental Assumptions

- [What this spec assumes about the execution environment]
- [External dependencies assumed to be available]
- [Preconditions assumed to be met]

### 9.2 Input Assumptions

- [What this spec assumes about the nature/quality of inputs]
- [Guarantees expected from upstream systems]

### 9.3 Assumption Violations

| Assumption | If Violated |
|------------|-------------|
| [Assumption A] | [What should happen if this assumption is false] |
| [Assumption B] | [Behavior when assumption fails] |

---

## 10. OPEN QUESTIONS

> [If any ambiguities remain unresolved, list them here for human decision. Remove this section if all questions are resolved.]

- [ ] [Question 1: Description of ambiguity and options]
- [ ] [Question 2: Description of ambiguity and options]

---

## CHANGELOG

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| [X.Y] | [YYYY-MM-DD] | [Name] | [Description of changes] |

---

## APPENDIX A: Extended Examples (Optional)

[Additional examples for complex cases, if needed]

## APPENDIX B: Related Specifications (Optional)

[References to other specs this depends on or relates to]
