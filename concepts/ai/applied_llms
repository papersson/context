
# Applied LLMs: A Comprehensive Taxonomy

**Scope**: Everything about working with and applying LLMs (using them effectively + building systems with them), excluding LLM internals (training, model architecture, inference optimization)

---

## Introduction: Core LLM Properties

**Fundamental properties that apply universally, whether you're chatting with ChatGPT or architecting production systems**

### Context Windows: The Working Memory Space

- Definition: Finite token capacity where LLM "sees" information
- Constraints: Token limits (8K-200K+), quality degradation at high usage
- The architectural substrate where context engineering operates
- Implications for system design (see Part 2: Context Window Topology)

### Model Behavior

- Non-determinism: Same prompt ≠ same output
- Hallucination: Models generate plausible-sounding incorrect information
- Token-by-token generation: Sequential, not parallel reasoning
- Training cutoff dates: No knowledge of events after training

### Tool Usage Fundamentals

- Why tools exist: LLMs weak at arithmetic, current info, precise factual recall
- Basic mechanisms: Code execution, web search, function calling, RAG
- Same underlying patterns whether using or building

### Grounding

- Code execution for precise computation
- Web search for current information
- RAG for private/specialized knowledge
- **Verifiable Rewards**: Grounding mechanisms (unit tests, compiler output, schema validation) serve as objective signals for inference-time optimization (RLVR)
- Verification methods (tests, human review, LLM-as-judge)

---

## Part 1: Using LLMs Effectively

**How to work with LLMs as an end user - personal productivity, knowledge work, collaboration patterns**

### Usage Patterns (How YOU Work With Systems)

**Note**: These are NOT system properties - they're patterns for how you choose to work with systems. Well-designed systems (like Claude Code) may support multiple patterns; others deliberately constrain to one.

- **Conversational**: Turn-by-turn collaboration, you guide each step
- **Task-Oriented**: Set goal, system executes, you review results
- **Async**: Long-running background work, notification on completion

### Iterative Refinement (Clarifying Human Intent)

- The human constraint: We're bad at articulating intent upfront
- Generate → Evaluate → Refine loop
- LLM as translator: From vague goals to specific instructions
- Techniques: Multiple choice questioning, chunked questioning, progressive disclosure, example-based clarification
- **Automated Equivalent**: In system building, this loop is formalized as **Reflective Optimization** (GEPA), where an LLM acts as the user to refine the prompt
- Fast iteration is key (seconds to minutes per cycle)

### Knowledge Architecture

- **Primary use case**: Personal/organizational knowledge for LLM consumption
- **Three-layer system**: Sources (raw) → Drafts (WIP) → Public (stable)
- **Context repositories**: Instructions, prompts, concepts, domain knowledge
- **Note**: LLM systems can also consume these repositories for domain-specific tasks

### Personal Workflow Patterns

- Prompt libraries (reusable transformations)
- Context management strategies
- Integration with tools (Claude Code for notes, research, general file system work)

### Effective Prompting

- Few-shot learning (show examples)
- Role and persona prompting
- Structured output specifications
- LLM-assisted prompt creation

---

## Part 2: Building LLM Systems

**Architecture, primitives, patterns, and production considerations for developers building systems where LLMs are core components**

### Primitives (The Building Blocks)

**The fundamental execution mechanisms that compose into systems**

#### LLM Call

- Single prompt → response
- Stateless (no memory between calls)
- Fastest, cheapest, simplest

#### Agent (ToolCallingLoop)

- **Definition**: LLM drives sequential decisions using specialized toolset
- **Key insight**: This is the primitive that enables LLM-driven control flow
- **Core loop**: LLM decides → call tool → observe → LLM decides → repeat
- **Control flow**: LLM determines what happens next (vs deterministic code)
- **Specialization**: SearchAgent, CodeAgent, DataAgent, PlanningAgent, etc.
- **Why it matters**: Transforms LLMs from text generators to autonomous decision-makers

#### Deterministic Workflow

- Traditional program logic with embedded LLM calls
- Code controls flow, LLM used at specific steps
- Predictable, debuggable, lower cost

---

### Context Engineering

**Formalized patterns for managing context in LLM systems - especially critical for systems with heavy AI control flow (Agents, Deep Agents)**

#### The Four Core Patterns

- **Write**: Save information outside context window (scratchpads, long-term memory, filesystem)
- **Select**: Pull only relevant information in (memory selection, tool selection, knowledge retrieval)
- **Compress**: Retain only necessary tokens (summarization, trimming, pruning)
- **Isolate**: Split context across specialized agents or environments

#### Trace Engineering (Optimization Context)

- **Definition**: Formatting context not just for the Agent's next step, but for the Optimizer's retrospective analysis
- **Pattern**: Ensure intermediate reasoning and raw tool outputs are preserved in optimization logs, even if "Compressed" for the immediate agent context
- **Goal**: Enable the "Reflector" LLM to correctly attribute blame during optimization

#### Relationship to Context Windows

- Operates ON the context window substrate
- Pattern requirements vary by system architecture
- Essential for Deep Agents (managing hierarchical contexts)

#### Context Failure Modes

- **Context poisoning**: Hallucinations enter and get repeated
- **Context distraction**: Too much history, model fixates
- **Context confusion**: Superfluous information degrades quality
- **Context clash**: Contradictory information causes reasoning failures
- **Opaque Traces**: Over-aggressive summarization hides specific tool errors or logic flaws, causing the Optimizer to hallucinate the cause of failure

#### Planning Tools as Context Engineering

- Often no-ops (don't execute actions)
- Force explicit reasoning and planning
- Keep agents on track by externalizing goals
- Example: Claude Code's todo list tool

---

### System Architectures (How Primitives Compose)

**For each architecture: definition, primitives used, context window topology, context engineering requirements, when to use**

#### Traditional + LLM

- **Definition**: Deterministic workflow with LLM calls at specific steps
- **Primitives**: Deterministic Workflow + LLM Calls
- **Control flow**: Your code decides what happens next
- **Context topology**: Single window, cleared/reset each call, no accumulation
- **Example**: Contract classification pipeline (iterate contracts, classify each)
- **When to use**: Well-defined workflows, predictable steps, cost-sensitive

#### Agent-Driven (Single ToolCallingLoop)

- **Definition**: Single Agent (ToolCallingLoop) controls system flow
- **Primitives**: Agent (ToolCallingLoop) + tools
- **Control flow**: LLM decides next actions autonomously
- **Context topology**: Single accumulated window, history grows with each loop iteration
- **Context engineering**: Becomes critical as tasks lengthen (Write, Compress especially)
- **Examples**: SearchAgent, simple coding agents, The Oracle's search component
- **When to use**: Open-ended tasks, exploratory work, flexibility over predictability

#### Deep Agent (Hierarchical Orchestration)

- **Definition**: Orchestrator agent delegates to hierarchical subagents with planning tools, external memory, and isolated contexts
- **Architectural justification**: 15× token economics, 90% performance gain on breadth-first tasks, fundamentally different composition
- **Note**: NOT a new primitive - still uses Agent (ToolCallingLoop), but composed hierarchically

**The Four Core Components**:

1. **Planning tools**: Explicit planning (often no-ops for context engineering), todo lists, adaptive plans
2. **Subagents**: Hierarchical delegation with isolated contexts, specialized tools per subagent, parallel execution
3. **Filesystem/memory**: External persistent storage, prevents context overflow, shared workspace for collaboration
4. **Detailed prompts**: ~2000 lines, tool usage instructions, few-shot examples, error handling protocols

**Context topology**:

- **Hierarchical windows**: Main (orchestrator) + multiple auxiliary (subagents)
- **Main context**: Orchestrator maintains plan, coordinates subagents, synthesizes results
- **Subagent contexts**: Isolated execution, clean slate per delegation, only synthesized results returned
- **Context engineering**: Essential for managing multiple windows, preventing overflow, coordinating across contexts

**The Spectrum**:

```
Simple Agent → Deep Agent (Stateless) → Deep Agent (Stateful) → True Multi-Agent
```

Most production implementations fall in middle (some persistence but not full Actor model)

**When justified**:

- Breadth-first queries (multiple independent directions)
- Information exceeds single context window (>200K tokens)
- High task value (justifies 15× token cost)
- Parallelizable subtasks
- 10-40 minute runtime acceptable

**Examples**: ChatGPT Deep Research, Claude Code, Anthropic Multi-Agent Research, Perplexity Deep Research, Gemini Deep Research

**When NOT to use**: Sequential dependencies, context coherence critical, cost-sensitive, debugging priority

#### Multi-Agent System (Peer Coordination)

- **Definition**: Multiple autonomous agents with independent persistent state, peer coordination
- **Primitives**: Multiple Agents (ToolCallingLoop) with coordination layer
- **Context topology**: Distributed peer contexts, each agent maintains independent state
- **Coordination**: Via written artifacts, message passing (not shared context)
- **Note**: Theoretical ideal (Actor model) not fully achieved in production
- **Challenges**: Conflicting assumptions, coordination overhead, debugging complexity

#### Hybrid/Composed

- **Definition**: Mix of agent and deterministic components
- **Example**: The Oracle (SearchAgent → finds docs → deterministic synthesis → answer)
- **Context topology**: Mixed (depends on composition)
- **Flexibility**: Combine predictability (deterministic) with flexibility (agents)

---

### Tool Design Patterns

#### Design Principles (Anthropic Guidelines)

- Choose the right tools (consider agent affordances vs API parity)
- Namespace related tools (grouping by service or resource)
- Return meaningful context (human-readable over technical IDs)
- Optimize for token efficiency (pagination, filtering, truncation)
- Prompt-engineer tool descriptions (write for "new hire on team")

#### Design for Optimization (New)

- **Verbose Error Signals**: Error messages must be diagnostic, not just descriptive (e.g., "Invalid Date" → "Invalid Date: YYYY-MM-DD required")
- **Reflector Visibility**: Return values are the primary signal for the Optimizer; they must explain *why* something happened
- **Traceability**: Return metadata (confidence scores, sources) to allow specific attribution of quality

#### Key Patterns

- Fewer tools with parameters > many specialized tools
- Consolidate multiple API calls under single tool
- Provide response_format options (concise vs detailed)
- Agent-readable error messages

#### Tool Testing

- Tool-testing agents that rewrite descriptions (40% completion time improvement)
- Iterate tool design based on agent usage patterns

---

### Evaluation & Iteration

**Critical feedback loops for improving systems - applies during development AND production**

#### Evaluation Strategies

- **Unit tests**: For deterministic components
- **LLM-as-judge**: For evaluating outputs quality
- **Human review loops**: Essential for catching subtle issues
- **Benchmark datasets**: Start small (20 queries), grow systematically
- **Multiple metrics**: Accuracy, runtime, token consumption, tool errors, user satisfaction

#### Observability

- **Tracing**: What happened in each step (especially critical for Deep Agents)
- **Context window monitoring**: Usage patterns, overflow detection
- **Tool call patterns**: Which tools used, success/failure rates
- **Token usage tracking**: Cost monitoring, efficiency analysis
- **Decision patterns**: For debugging non-deterministic behavior

#### Feedback Loops

- Eval results → prompt refinement
- Observability → architecture changes
- User feedback → system improvements
- The cycle: Build → Eval → Observe → Iterate → Build

---

### Agent Optimization (Inference-Time Learning)

**The systematic improvement of agents using RL concepts without model weight training. Replaces manual "prompt engineering" with automated evolutionary loops.**

#### The Optimization Loop (The Primitive)

1. **Generate**: Run Agent trajectory (trace) on training set
2. **Evaluate**: Score trace using Rubrics (Metric-based rewards)
3. **Reflect**: "Reflector" LLM analyzes trace + scores to diagnose failure
4. **Mutate**: Reflector proposes updated System Prompt or Few-Shot examples
5.  **Select**: Keep candidates that improve the Pareto frontier

#### The Mechanism: Linguistic Gradients

- **Concept**: Instead of numeric gradients (backprop), we use **Natural Language Feedback**
- **Credit Assignment**: Implicitly handled by the Reflector reading the verbose trace (see *Trace Engineering*)
- **Turn-Level Rewards**: Used as **Metrics** for the Reflector (diagnostic data), NOT injected into the Agent's context (which disrupts inference)

#### Strategies by Architecture

- **Traditional+LLM**: Optimize instructions/few-shots (DSPy, MIPROv2, GEPA)
- **Agent-Driven**: Optimize for tool selection accuracy and error recovery via trace analysis
- **Deep Agent**: **Joint Optimization** (Co-evolving Orchestrator and Subagents simultaneously)
    - *Note*: Evidence shows joint optimization outperforms coordinate descent (freezing layers) for hierarchical prompts

#### The Cost/Benefit

- **Sample Efficiency**: Requires ~500-1000 rollouts (vs 20k+ for RL training)
- **Target**: High-value, complex prompts (e.g., Deep Agent Orchestrators) where manual tuning is intractable

---

### Production Considerations

#### Deployment Patterns

- Synchronous vs asynchronous execution
- Streaming for perceived latency reduction
- Human-in-the-loop checkpoints

#### Cost Management

- Token budgeting (especially for Deep Agents: 15× cost)
- KV-cache optimization (10× savings for cached tokens)
- Model selection (task-appropriate models)

#### Reliability & Error Handling

- Graceful degradation
- Retry strategies with backoff
- Error preservation for debugging
- Context recovery mechanisms

#### Security & Guardrails

- Input validation and sanitization
- Output filtering
- Tool access controls
- Sandboxed execution environments

#### Scaling Considerations

- Rate limiting
- Load balancing
- State management at scale
- Monitoring and alerting

---

## Part 3: Understanding "Agent" Overloading

**Explicit mapping of how "agent" is used across different contexts - resolving the terminology confusion**

### The Core Confusion

**Two orthogonal dimensions create terminology collision:**

#### Dimension 1: Control Flow (Technical/Mechanism)

- Who decides what happens next?
- Deterministic code vs LLM decides
- **Agent (ToolCallingLoop)** is the primitive for LLM-driven control flow

#### Dimension 2: Autonomy/Behavior (User-Facing/Outcome)

- How autonomous does it feel?
- Does it augment or replace human work?
- What business people mean by "agent"

### Four Definitions of "Agent"

#### 1. Agent (ToolCallingLoop) - Technical Primitive

- **Definition**: LLM drives sequential decisions using specialized toolset
- **Audience**: Developers building LLM systems
- **Key property**: LLM controls the flow
- **Examples**: SearchAgent, CodeAgent, DataAgent
- **Status**: Technical definition for this taxonomy

#### 2. Deep Agent - Architectural Pattern

- **Definition**: Hierarchical orchestration with planning, subagents, memory, detailed prompts
- **Audience**: System architects, framework designers
- **Key property**: Composition pattern, not new primitive
- **Examples**: ChatGPT Deep Research, Claude Code
- **Status**: System Architecture category

#### 3. Agent - Colloquial/Business Term

- **Definition**: "Something that autonomously performs tasks and achieves goals like a human"
- **Audience**: Business stakeholders, end users, marketing
- **Key property**: Behavioral (feels autonomous), not mechanistic
- **Examples**: "We need an agent for customer support"
- **Status**: Vague but pervasive usage

#### 4. Agent - Buzzword/Misuse

- **Definition**: Any AI thing that does stuff
- **Audience**: Non-technical users influenced by poor communication
- **Examples**: Calling specialized LLMs with system prompts "agents"
- **Status**: Terminology pollution to be aware of

### Mapping the Confusion

```
High autonomy + LLM control flow = "Agent" (all definitions align)
  Example: Claude Code, Deep Research

High autonomy + Deterministic flow = Feels "agent-like" but isn't technical Agent
  Example: Well-designed automation workflow

Low autonomy + LLM control flow = Technical Agent, but doesn't feel autonomous
  Example: Simple SearchAgent that just returns context

Low autonomy + Deterministic = Traditional software
  Example: Contract classification pipeline
```

### Why ToolCallingLoop Matters

**Key insight**: ToolCallingLoop is the primitive that enables **LLM-driven decision making** - the fundamental shift from LLMs as text generators to LLMs as autonomous controllers.

Before ToolCallingLoops: LLMs only generated text

With ToolCallingLoops: LLMs can:

- Decide which tool to use next
- React to tool results
- Plan multi-step sequences
- Adapt based on observations

This is why it's a foundational primitive, even though "agent" as a term is overloaded.

### Usage Guidance

**In technical discussions**: Use "Agent (ToolCallingLoop)" or just "ToolCallingLoop" for the primitive

**In architecture discussions**: Use "Deep Agent (Hierarchical Orchestration)" for the pattern

**With business stakeholders**: Use "autonomous system" or "AI-powered workflow" - avoid "agent" unless you clarify which definition

**When reading sources**: Determine context - is author referring to mechanism, architecture, or behavior?

---

## Appendix: Context Window Topology Reference

**Quick reference for understanding context architecture across system types**

|System Architecture|Context Topology|Main Context Location|Engineering Priority|
|---|---|---|---|
|Traditional + LLM|Single, reset each call|N/A (stateless)|Minimal - each call isolated|
|Agent-Driven|Single, accumulated|The agent's context|High - grows with task length|
|Deep Agent|Hierarchical (main + subagents)|Orchestrator context|Critical - managing multiple windows|
|Multi-Agent|Distributed peers|Depends (orchestrator or none)|Critical - coordination without sharing|
|Hybrid/Composed|Mixed|Varies by composition|Depends on agent components|

**Design consideration**: When architecting systems, identify where primary state/reasoning lives - this is your "main context window" that requires most careful engineering.
