Read the project spec at SPEC.md and the backlog template at BACKLOG_TEMPLATE.md. Explore the current codebase to understand what already exists from scaffolding. Then generate an initial BACKLOG.md for this project.

Follow the instructions below carefully.

### Step 1: Audit what exists

Before creating any items, inventory what the scaffolding session already produced:
- What files and directories exist?
- Which domain types from the spec are already defined in code?
- Is there a working build/run/test pipeline?
- Can the project be started or invoked, even if it does nothing useful yet?

Mark these as done in Phase 0. Don't create items for work that's already complete.

### Step 2: Decompose into phases

Phases are ordered so that each one builds on the previous. Use this general progression, but adapt to the project:

- **Phase 0 — Scaffolding:** Project structure, types, build tooling, dev scripts. (Likely mostly done already.)
- **Phase 1 — Core happy path:** The minimal end-to-end flow. One input goes in, one correct output comes out. No error handling, no edge cases. This is the first moment the human can see the system *do the thing.*
- **Phase 2 — Remaining happy paths:** Cover the other input types, modes, or features described in the spec. Each item adds a new capability the human can observe.
- **Phase 3 — Error handling and edge cases:** Invalid inputs, failure modes, boundary conditions. Reference the spec's edge cases and negative constraints.
- **Phase 4 — Integration and polish:** External service integration (if any), performance, logging, configuration, documentation. Reference the spec's manual verification checklist for final acceptance.

Not every project needs all phases. A small CLI tool might be 2 phases. A system with external dependencies might need 5. Use judgment, but keep the total number of phases between 2 and 5.

Each phase gets a one-sentence goal statement. The goal should describe what's *newly true* about the system, not what was coded. Good: "The system can ingest any valid CSV and produce a correctly formatted report." Bad: "Implement the report generator module."

### Step 3: Create items within phases

Each item should be a vertical slice where possible — meaning it has an observable behavior the human can verify without reading code. The test is: "Could the human confirm this works by running a command, looking at output, or interacting with the system?"

**Good items (observable):**
- "CLI accepts a config file path and starts with those settings"
- "API returns a 400 with a descriptive error when given malformed JSON"
- "System retries on 429 from the external service and succeeds on the second attempt"

**Bad items (internal, not observable):**
- "Implement the config parser"
- "Add retry logic to the HTTP client"
- "Write unit tests for validation"

The exception is Phase 0 / infrastructure work, where verification is structural: "Show me the type definitions and explain how they map to the spec."

**Sizing heuristic:** A single item should represent roughly 30-90 minutes of focused agent work. If you find yourself writing an item with multiple "and"s — "parse the input AND validate it AND transform it AND write the output" — split it. If the item is just "add a single field to a type," it's probably too small — group it with related work.

### Step 4: Write verification steps

Every item (except Phase 0 infrastructure) needs:

1. **Verify** — The primary way the human confirms this works. Written as a concrete action with an expected result.
2. **Fallback** — A quicker, simpler check if the primary verification is too involved.
3. **Spec ref** — Which spec section, constraint, or example this item implements.

**Rules for verification steps:**

- **They must be performable by the human without reading code.** Running a command, curling an endpoint, invoking the CLI, opening a browser, looking at a file — all fine. "Check that the function returns X" is not — that's a unit test.

- **They must reference concrete inputs and expected outputs.** Use values from the spec's examples wherever possible. "Run with the input from spec example V2, confirm output matches" is ideal.

- **They should catch the most likely misinterpretation.** Think about how the agent could implement this in a way that's technically correct but wrong in intent. The verification should catch that. For example, if the spec says "amounts are displayed in the user's locale," the verification should use a non-US locale to confirm it's not hardcoded.

- **Fallback verification is a quick sanity check, not a thorough test.** Good fallback: "Check that `output.json` exists and contains the expected top-level keys." Doesn't prove correctness but catches total failure.

**Good verification:**
```
Verify: Run `./cli process --input fixtures/sample.csv` and confirm the output
        table matches spec example V2 (3 rows, columns: name, total, status).
Fallback: Run `./cli process --input fixtures/sample.csv | head -5` and check
          that the header row has the right column names.
Spec ref: Outputs section, Example V2
```

**Bad verification:**
```
Verify: Run the test suite and confirm all tests pass.
Fallback: Check the code for obvious errors.
Spec ref: Outputs
```

### Step 5: Add dependencies where they exist

If an item cannot be started until another item is complete, note it with `depends: [item number]` in the note field. Only note true blockers — "this literally cannot work without that" — not soft preferences about ordering.

### Step 6: Review your own output

Before writing the file, check:
- Does every phase have a clear goal that describes system behavior, not code written?
- Is every item (outside Phase 0) a vertical slice the human can observe?
- Could the human actually perform every verification step right now, given the current state of the project?
- Are spec references specific (a named section, a specific example) rather than vague?
- Are there items covering the spec's negative constraints and edge cases?
- Is anything too big? Would any item take more than ~90 minutes of focused work?

Write the result to BACKLOG.md. I'll review and restructure before we start implementing.
