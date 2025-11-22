---
owner: Patrik Persson
last_updated: 2025-11-22
type: meta-prompt
usage: "Inject into active chat to bridge knowledge gaps"
---

# Meta-Prompt: Generate Deep Research Specification

## Purpose
This prompt is an **orchestration tool**. It bridges the gap between a reasoning session (e.g., with Claude/GPT) and a research session (e.g., with Deep Research/Perplexity).

It solves the problem of "Lossy Handoffs" where the nuance of a long conversation is lost when the user manually tries to search for answers.

## When to Use
1.  **The "I Don't Know" Trigger:** When the current LLM expresses uncertainty or hallucination risks.
2.  **The Verification Trigger:** When a complex claim needs to be grounded in primary sources.

## The Prompt Snippet
*Copy and paste the block below into your current LLM session:*

---

*** SYSTEM INSTRUCTION: GENERATE DEEP RESEARCH SPECIFICATION ***

We have hit a knowledge gap. I need you to formulate a **Deep Research Prompt** that I can paste into an autonomous research agent.

**YOUR TASK:**
1.  Analyze our entire conversation history to identify exactly what is unknown, ambiguous, or needs verification.
2.  Construct a self-contained prompt that I can feed to the research agent.

**THE RESEARCH PROMPT STRUCTURE (Must be inside a Code Block):**
The prompt you generate must follow this strict schema:

*   **MISSION:** A single sentence defining the goal (e.g., "Find implementation details for X").
*   **CONTEXT:** A brief summary of *why* we need this (the "Commander's Intent").
*   **TARGET QUESTIONS:** 3-5 specific, atomic questions. (e.g., "What is the API endpoint for X?" not "How does X work?").
*   **SOURCE HIERARCHY:**
    *   *Priority:* Documentation, GitHub Issues, arXiv, Official RFCs.
    *   *Forbidden:* SEO blogs, marketing pages, listicles.
*   **NEGATIVE SPACE:** Explicitly tell the agent what *not* to research (e.g., "Do not explain what RL is").
*   **OUTPUT FORMAT:** Define the desired artifact (e.g., "Comparison Matrix," "Code Snippet").

**CONSTRAINT:** Do not be polite. Be imperative, dense, and self-contained.
