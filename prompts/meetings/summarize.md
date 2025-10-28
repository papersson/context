### **Prompt for Comprehensive Meeting Transcript Summarization**

**Your Task:**

Analyze the provided meeting transcript and generate a structured summary document in Markdown format. The goal is to distill the key information into a clear, concise, and actionable format.

The summary must be organized into the following three sections:

**1. Conceptual Summary:**
   - **Objective:** Provide a high-level overview of the meeting's purpose and the main topics discussed. This section should answer the question, "What was this meeting about?"
   - **Instructions:**
     - Begin with a concise, one-sentence statement summarizing the primary goal of the meeting.
     - Elaborate on the core concepts, key challenges, and main arguments or proposals presented by the speakers.
     - Group related topics together to create a logical flow, even if the conversation jumped between subjects.
     - Focus on the "why" and "what" of the discussion, rather than a chronological play-by-play.
     - The summary should be easy for someone who did not attend the meeting to understand the context and significance of the conversation.

**2. Decision Log:**
   - **Objective:** Create a clear log of all decisions made during the meeting, including agreements, disagreements, and items that were postponed.
   - **Instructions:**
     - Use a bulleted list to itemize each decision.
     - For each item, clearly state:
       - **The Topic:** What was being decided? (e.g., "Storage Access Model," "Use of Managed vs. External Tables").
       - **The Decision:** What was the final resolution? (e.g., "Agreed to proceed with a new storage account for new projects," "Decision postponed pending cost analysis").
       - **The Rationale (Optional but Recommended):** Briefly explain the key reasons behind the decision. (e.g., "To improve security and isolate blast radius").
     - If a decision was not reached, explicitly state that (e.g., "No consensus was reached," "Further discussion is required").

**3. Action Items:**
   - **Objective:** List all actionable tasks assigned during the meeting, including who is responsible and any relevant deadlines or context.
   - **Instructions:**
     - Use a bulleted list or a table to present the action items.
     - For each action item, clearly specify:
       - **The Task:** A clear and concise description of what needs to be done (e.g., "Perform a cost analysis of serverless vs. non-serverless SQL warehouses").
       - **The Owner(s):** Who is responsible for completing the task? (e.g., "Pedro Rodrigues"). If no one was explicitly assigned, note that (e.g., "Owner to be determined").
       - **Due Date (if mentioned):** When is the task expected to be completed? (e.g., "Next week," "Before the next meeting"). If not mentioned, state "Not specified."
       - **Relevant Context (Optional):** Any additional information needed to understand the task.

**Final Output Format:**

The final output should be a single, well-structured Markdown document with clear headings for each section (`# Conceptual Summary`, `# Decision Log`, `# Action Items`).

---

