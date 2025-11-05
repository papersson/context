# Prompts

Single-purpose transformations for discrete input → output tasks.

## Purpose

Prompts define **what specific thing to do right now**. They are one-shot tasks or workflow building blocks that often chain together in sequences.

## Prompts vs. Instructions

**Use Prompts for:**
- Discrete transformations (input → output)
- "Do this specific thing"
- One-shot tasks
- Workflow building blocks that chain together
- Task-specific operations

**Use Instructions for:**
- Persistent behavioral guidance across conversations
- "How to think" or "how to approach problems"
- Agent personality and methodology
- See [instructions/](../instructions/) instead

## Decision Rule

**Ask: "Is this a discrete transformation or persistent behavior?"**

Examples:
- ✓ "Clean this transcript" → prompts/meetings/
- ✗ "When cleaning transcripts, always do X" → instructions/
- ✓ "Summarize this meeting" → prompts/meetings/
- ✗ "You are a meeting facilitator" → instructions/chat/
- ✓ "Rewrite in direct prose style" → prompts/writing-style/
- ✗ "Always write in direct prose" → instructions/writing-style/

## Current Categories

### [interviews/](interviews/)
Structured interview prompts for extracting information from stakeholders or yourself.
- `self-clarification.md` - Help clarify your own fuzzy, half-formed ideas
- `requirements-gathering.md` - Extract requirements from stakeholders
- `process-clarification.md` - Document how work actually gets done
- `knowledge-extraction.md` - Extract domain expertise and mental models

**Interview approach:**
- Ask 2-6 questions in chunks, iterate based on answers
- Heavy use of multiple choice to reduce friction
- Drill into vagueness, catch contradictions, push back strategically
- Output is transcript for later transformation

### [modeling/](modeling/)
Domain modeling transformations - from transcripts to executable types.
- `transcript-to-and-or-notation.md` - Transform interview transcripts into AND/OR notation
- `and-or-to-types.md` - Transform AND/OR notation into type definitions (TypeScript, F#, etc.)

**Modeling approach:**
- Interview → AND/OR notation → Type definitions
- Makes business rules explicit in type system
- "Make illegal states unrepresentable"
- Pairs with `interviews/knowledge-extraction.md`

### [meetings/](meetings/)
Meeting-related transformations.
- `clean_teams_json_transcript.md` - Clean messy Teams JSON transcripts
- `summarize_transcript.md` - Summarize meeting transcripts into key points

### [writing-style/](writing-style/)
Text style transformations.
- `direct_prose.md` - Rewrite text in direct, clear prose
- `conceptual_summary.md` - Create conceptual summaries

### [learning/](learning/)
Knowledge extraction and synthesis.
- `knowledge_map.md` - Extract and map key concepts from content

## Using Prompts

### One-Shot Usage
Apply a single prompt to transform input:

```bash
# Example: Clean a transcript
cat messy_transcript.json | llm -s "$(cat prompts/meetings/clean_teams_json_transcript.md)"
```

### Chaining Prompts
Combine prompts in workflows:

```bash
# Example workflow: Clean → Summarize
# Step 1: Clean the transcript
cat raw_transcript.json | llm -s "$(cat prompts/meetings/clean_teams_json_transcript.md)" > clean.txt

# Step 2: Summarize the cleaned transcript
cat clean.txt | llm -s "$(cat prompts/meetings/summarize_transcript.md)" > summary.txt
```

### In LLM Conversations
Copy prompt into a conversation when you need that transformation:

1. Start new conversation
2. Paste prompt content
3. Provide input to transform
4. Get output

## Prompt Workflows

Common patterns for chaining prompts:

**Interview-Based Workflows:**
1. `interviews/requirements-gathering.md` → requirements transcript
2. Transform transcript into structured requirements doc
3. Extract key decisions or constraints

Or:
1. `interviews/process-clarification.md` → process transcript
2. Transform into formal process documentation
3. Generate flowcharts or decision trees

**Domain Modeling Workflow:**
1. `interviews/knowledge-extraction.md` → domain expert transcript
2. `modeling/transcript-to-and-or-notation.md` → AND/OR notation
3. `modeling/and-or-to-types.md` → type definitions (TypeScript/F#/etc.)
4. Implement domain logic guided by types

**Meeting Processing:**
1. `clean_teams_json_transcript.md` → clean text
2. `summarize_transcript.md` → key points and action items

**Writing Refinement:**
1. Draft rough text
2. `direct_prose.md` → clear, direct version
3. `conceptual_summary.md` → executive summary

**Knowledge Synthesis:**
1. Gather source material
2. `knowledge_map.md` → structured concepts
3. Additional analysis or summarization

## Adding New Prompts

1. Verify it's a discrete transformation, not persistent behavior
2. Ask: "Is this a one-shot task or workflow building block?"
3. Create file in appropriate category: `prompts/category/name.md`
4. Include clear input/output examples
5. Document expected input format
6. Document expected output format
7. Consider how it might chain with other prompts
8. Update this README

## Prompt Standards

Each prompt should:
- Have clear input expectations
- Define output format
- Include examples where helpful
- Be self-contained (one file = one transformation)
- Work independently or as part of a chain

## Related Documentation

- [Instructions](../instructions/) - Multi-turn behavioral context
- [Concepts](../concepts/) - Universal reference knowledge
- [Falck Knowledge Base](../falck-knowledge-base/) - Falck-specific operational knowledge
