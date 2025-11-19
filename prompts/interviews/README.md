# Interview Prompt Library

This directory contains specialized system prompts for extracting information and clarifying intent.

## Selector: Which prompt do I need?

| If you are trying to... | Use this prompt |
| :--- | :--- |
| **Clarify your own thoughts** regarding a fuzzy idea, decision, or architectural problem. | `self-clarification.md` |
| **Extract "Tacit Knowledge"** from an expert. You want to know *how they think*, their mental models, and definitions. | `knowledge-extraction.md` |
| **Map a Workflow.** You want to know the sequence of steps, edge cases, inputs, and outputs of a specific task. | `process-clarification.md` |
| **Define a Problem.** You want to know *what to build* and *why*, focusing on success criteria and constraints. | `requirements-gathering.md` |

## Common Usage Patterns

### 1. The Direct Interview (Standard)
Load the prompt. You are the subject. The LLM interviews you.
*Use case: Brainstorming, rubber-ducking, creating a spec for an agent.*

### 2. The Copilot (Live Mode)
Load the prompt. Tell the LLM: *"I am interviewing a stakeholder about [Topic]. Here are my notes so far."*
The LLM will analyze your notes and suggest the next questions to ask.
*Use case: You are on a call and need help finding the missing requirements.*

### 3. The Transcript Analyzer (Post-Hoc)
Load the prompt. Paste a raw transcript from a meeting.
Instruction: *"Based on this transcript, fill out the 'Live Context' state and tell me what questions I failed to ask."*
