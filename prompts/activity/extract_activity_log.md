# MISSION
Act as a **Technical Biographer**. Transform the raw session dump into a **Structured Narrative** that preserves the *reasoning* and *constraints* of the work.

# INPUT CONTEXT
(Paste Claude Code logs, Terminal history, or messy notes here)

# EXTRACTION RULES
1.  **The "Why", not just the "What":** If we changed direction, explain the realization that caused it.
2.  **Preserve the Nouns:** Keep specific library names (`pydantic v2`), error types (`RecursionError`), and variable names if they are central to the logic.
3.  **The Struggle is the Asset:** Highlight *exactly* where the model/human got stuck. This is future training data for "Hard Cases."

# OUTPUT FORMAT (Markdown)

## 1. Executive Summary
*   **Intent:** [One sentence goal]
*   **Outcome:** [Success/Partial/Fail]
*   **Complexity Rating:** [Low/Med/High] - *Why?*

## 2. The Decision Log (Chronological)
*   **[Time/Step]**: **[Action Name]**
    *   *Hypothesis:* "We thought X would work..."
    *   *Reality:* "...but it failed because [Specific Technical Reason]."
    *   *Pivot:* "So we switched to method Y."

## 3. The "Technical DNA" (Critical Context)
*   **Key Libraries:** [e.g. Pandas, NumPy]
*   **Hidden Constraints:** [e.g. "Data must be sorted before grouping", "API rate limit is 5/sec"]
*   **Traps Encountered:** [Specific bugs or logic errors to watch out for]

## 4. Future Synthesis Hints
*   *If we wanted to teach an AI to do this, what is the one lesson from this session?*
