### Context Engineering

Every token loaded into an LLM's context window displaces potential space for something else. This constraint shapes how information enters, persists, and leaves the model's working memory. Context engineering emerges from this scarcity—the fundamental architecture that determines whether an agent succeeds or fails at complex tasks.

The context window functions as bounded working memory with attention that degrades as tokens accumulate. Quality degradation begins long before reaching the hard context limit. The attention mechanism forces every token to attend to every other token, creating quadratic computational scaling and attention dilution. As sequences lengthen, average attention per useful token falls, and weakly related text drags the model toward generic completions.

Models weight information at the beginning and end of prompts more heavily than content in the middle—a "lost in the middle" effect where crucial information vanishes despite being technically present. Primacy and recency bias mean late junk can override early gems, while early overload starves later specifics. Instruction interference emerges when multiple policies, examples, and conventions begin contradicting each other, forcing the model to hedge. Long copies of raw sources force on-the-fly compression, and the model does worse compression than a deliberate summarizer would.

This creates the central tension: agents need rich context to reason effectively, but excessive context degrades their ability to reason at all. The solution isn't finding a single optimal context size. It's architecting systems that maximize signal-to-token ratio by dynamically managing what information enters the context window, when it enters, and how long it persists.

**The Search-Then-Read Pattern**

Reading full files isn't the problem—reading the wrong files is. The key to effective context management is identifying the right information before loading it. This is why search strategy matters fundamentally: good search means reading the right files, which means using context efficiently. Ten wrong files waste more context than three right files consume.

Search establishes what exists and where. Iterative refinement—starting broad, then narrowing based on results—maps the landscape before committing tokens. Only after search identifies relevant files should those files load into context. Then reading them in full makes sense because you've already filtered for relevance. The pattern: search first to identify, read second to understand, act third with the right context loaded.

**Layers of Context Management**

Information has different temporal requirements. Core behavioral instructions need constant availability. Domain expertise should load on-demand when relevant. Exploratory work—reading dozens of files, running extensive searches, processing large datasets—should happen in isolation and return only distilled findings.

This stratification manifests in distinct primitives:

**Prompts** form the always-active foundation, consuming a fixed token budget on every invocation. This space holds a tiny kernel of invariants: principles, safety rules, IO contracts, conversational etiquette. Prompts shape behavior; they do not carry domain payloads. A small, stable core preserves headroom and reduces interference with task-specific details. An expansive system prompt explaining every domain detail consumes tokens that could instead load relevant skills when actually needed. Keep the kernel tiny; move payloads to skills and retrieval.

**Skills** add expertise progressively through triggered loading. When patterns appear—keywords, schemas, capabilities—metadata loads first describing what the skill does, when to use it, and its inputs/outputs. Then a compact SKILL.md provides core logic. Supporting files load only when explicitly referenced. This tiered disclosure prevents frontloading everything "just in case." Once mounted, a skill persists across conversational turns, enabling follow-up questions without reloading. Skills excel when expertise should span multiple exchanges and results stay concise. Enforce byte budgets and eviction policies—least recently used unless pinned.

**Subagents** isolate noisy exploration in separate contexts. When an agent needs to grep 100 files, read 20 PDFs, explore complex dependencies, or run tool fan-outs, spawning a subagent confines the mess to a sandbox. The subagent maintains its own working notes and returns distilled findings. The pattern: process is noisy, result is clean. Main agent context stays focused, receiving only conclusions. This enables parallel independent analyses—multiple subagents explore different aspects simultaneously without context interference.

For coding tasks, a code-intel subagent might search extensively across a codebase, read numerous files, trace dependencies, and build a mental model—then return a summary of key patterns with pointers to relevant files. The main agent can then read those specific files in full with confidence they're the right ones. The subagent's 50 searches and 20 file reads stay isolated; only the distilled navigation map enters the main context.

**Commands** execute predefined workflows with minimal context overhead. User-triggered runbooks like `/audit-baseline` or `/compare-versions` fetch just the needed inputs, execute a known pipeline, and emit outputs. Unlike skills that auto-invoke through pattern matching, commands require explicit user intent. This protects budgets and timing by giving control over when workflows run.

**Retrieval-augmented generation** loads external knowledge just-in-time based on query relevance. Vector databases enable semantic search across large knowledge bases, returning only relevant chunks. The retrieved context augments the prompt: QUESTION: <user query> CONTEXT: <retrieved information> Answer using the CONTEXT provided. This transforms the question from "recall from training data" to "synthesize from provided evidence." Modern implementations integrate retrieval as one step in broader reasoning loops where agents write, compress, isolate, and select context across tools and memory.

**Memory systems** accumulate learned context across conversations. Short-term memory holds recent turns. Long-term memory stores persistent facts and preferences retrieved based on relevance. Episodic memory captures specific past interactions as examples. Procedural memory encodes learned strategies. The challenge: ensuring relevant memories surface without overwhelming context with historical data. Effective memory systems use selective retrieval—querying for relevance rather than dumping everything into context.

**Compaction techniques** address long-horizon tasks where conversation history threatens to exceed limits. Summarization compresses old message history when the context buffer fills, discarding redundant data like old tool results while preserving critical decisions and dependencies. Sliding windows process overlapping text segments, maintaining some continuity across boundaries. The tradeoff: information loss against context efficiency. These techniques matter for managing accumulation over time, not for initial context loading where good search prevents loading the wrong things in the first place.

**Managing Context Quality**

Progressive disclosure prevents premature loading: metadata before core content before supporting details. When you need to work with specific files or information, prefer reading the right full files over reading partial snippets from many wrong files. The goal is loading relevant complete context, not fragmentary context from everywhere.

Cap token budgets per layer; compress or evict when caps hit. Prefer deltas (diffs, change logs) over full re-ingestion when proposing changes. Bind every large insertion to a purpose: "needed for step X?" If not, don't load it. Stop retrieval when uncertainty resolves; don't backfill "just in case."

Context budgeting manifests as explicit tradeoffs. Input tokens plus output tokens cannot exceed the context window. If input consumes most of the window, output confines to what remains. Beyond the hard limit lives the attention budget—the model's practical ability to track relationships across all loaded tokens.

Format matters for attention. Short, descriptive error messages work better than large JSON blobs. Structured sections (<instructions>, <tools>, ## Output format) improve readability and help models parse boundaries. Token-efficient tool design means clear input parameters, descriptive documentation that guides without overwhelming, minimal but sufficient context for each invocation.

**Coding Agents**

Coding agents magnify both the upside and failure modes of context engineering. The agent should never "read the repo" speculatively in the main context. Instead, use search to identify relevant files, then read those specific files in full. The shape of information matters: symbol tables, dependency graphs, and test surfaces help navigate before reading. Diffs communicate changes more efficiently than rewriting entire files.

A code-intel subagent (LSP/AST/index) can gather facts offline and return navigation aids: API surfaces with function signatures and types, caller/callee relationships and import graphs, open issues and failing tests, minimal context about where things are. Armed with this map, the main agent reads the specific files that matter, understanding them fully because search already filtered for relevance.

Workflow that holds up in practice:

1. **Plan**: state the target change and acceptance criteria concisely
2. **Index** (subagent): build or query the code graph; return only slices relevant to the plan
3. **Propose**: generate diffs when changing existing code, full files when creating new ones
4. **Validate**: run tests/linters; return failures concisely
5. **Iterate**: pull more context only when a specific missing fact is identified

This keeps the hot path under control while giving the agent accurate, up-to-date grounding. The pattern: search narrows scope, reading builds understanding, action operates on the right context.

**Operating Rules for Humans and Agents**

Both humans writing prompts and agents orchestrating workflows need to understand context engineering principles. The difference between a coding agent that thrives and one that flails often comes down to whether it manages its context budget deliberately or treats the context window as infinite until it hits a wall.

Humans writing prompts should write the kernel once and keep it short. When you need context, be specific about what you need so the agent can search effectively. Prefer commands for repeatable tasks; avoid free-form "do everything" prompts that make it hard to determine what context is actually relevant.

Agents orchestrating workflows should enforce layered budgets (core/history/skills/results) with caps. Route with metadata first; mount skills lazily; evict aggressively. Use subagents for retrieval, indexing, parsing, and evaluation; return distilled findings. Track budget versus quality per step; learn which skills earn their keep. Cache recent artifacts and decisions to avoid recomputation.

**Emerging Patterns**

Agentic context engineering treats contexts as evolving playbooks rather than static configurations. Instead of compressing knowledge into terse summaries, systems accumulate, refine, and organize strategies over time through structured processes: generators produce reasoning trajectories, reflectors distill insights from successes and errors, curators integrate insights into structured contexts. This prevents "brevity bias" where compression drops domain insights for concise summaries, and "context collapse" where iterative rewriting erodes details.

Hybrid approaches combine multiple techniques. Semantic search might provide initial retrieval, re-rankers assess relevance and reorder results, then selected chunks feed into the generation prompt. Memory systems filter recalled information through relevance scoring before adding to context. Subagents might use RAG internally while returning compressed summaries to the main agent.

**The Meta-Pattern**

A tiny always-on kernel sets behavior. Skills mount only when patterns call for them. Heavy work happens in subagents that return compact findings. Commands give precise control over when pipelines run. Quality stays high because the system protects its signal-to-token ratio long before the hard context limit becomes a factor.

Think in layers: always-on core, triggered expertise, heavy lifting in isolation, explicit user control. Each layer has different context implications. Choose the right primitive for context management needs, not just functional needs. The goal isn't maximizing context utilization—it's maximizing signal while minimizing noise. Search identifies what's relevant, then you can afford to read it fully. Load exactly what's needed when it's needed and nothing more.
