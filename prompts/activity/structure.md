# MISSION
Act as a **Dataset Architect**. Transform the "Chronological Activity Log" above into a rigorous **Evaluation Test Case**.

We need to teach a future AI agent how to solve this problem *without* making the mistakes we just made.

# ANALYSIS INSTRUCTIONS
1.  **Extract the Hidden Context:** What knowledge did we gain in Step 2 or 3 that we *wish* we had in Step 1? (e.g., "Column 'amt' is text, not float"). This goes into the **Context**.
2.  **Weaponize the Failures:** Look at the "Failures" in the timeline. Turn these into **Negative Constraints** (Traps).
3.  **Define Success:** Look at the "Final Result". How do we prove programmatically that an agent reached this state?

# OUTPUT FORMAT (Markdown)

## 1. Scenario Config (The Prompt)
*   **Instruction:** [The initial request given to the agent]
*   **Necessary Context:**
    *   [Fact A: Schema details]
    *   [Fact B: Business logic constraints revealed during the session]

## 2. The Minefield (Negative Constraints)
*   *Trap 1:* [Description of a failed approach from the timeline]
    *   *Why it fails:* [The technical reason]
*   *Trap 2:* [Another failed approach]

## 3. The Golden Record (Ground Truth)
```[language]
[The final working solution]
```

## 4. The Verifier (Reward Function)
*   **Logic:** [Pseudocode for a script that grades the result]
    *   *Check 1 (Deterministic):* [e.g., "Output must be a valid JSON"]
    *   *Check 2 (Semantic):* [e.g., "Must include user 'Bob'"]
    *   *Check 3 (Negative):* [e.g., "Must NOT use deprecated API X"]
