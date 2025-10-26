# CLAUDE.md - Core Universal Context

## General Agentic Patterns

### Context Retrieval
**The most critical step** - gathering sufficient context before acting determines success or failure.

#### Search Strategy
- Start broad, then narrow: `rg "pattern"` → `rg -A5 -B5 "pattern"` → read full files
- Use iterative refinement: each search informs the next query
- Combine multiple search tools: filesystem structure (`find`), content (`rg`), code structure (`ast-grep`)
- If a file is suspected relevant, **ALWAYS read it in full** before proceeding

#### Search Sub-Agent Pattern
**When to use:** 
- Multiple related files need examination (>5 files)
- Complex dependency tracing required
- Initial search scope is unclear
- Need to understand system architecture

**How it works:**
```
Main Agent: "I need to understand how authentication works in this codebase"
→ Spawn Search Agent with fresh context
→ Search Agent: Performs 10+ searches, reads files, builds mental model
→ Returns: Structured summary with key files, patterns, and insights
Main Agent: Continues with clean context + distilled knowledge
```

### Context Engineering

Your context window is finite and quality degrades well before the hard limit. Every token loaded displaces something else. At 32k tokens most models drop below 50% of their short-context performance. Longer sequences dilute attention, amplify position biases, and create interference between instructions. Protect your signal-to-token ratio: load exactly what's needed when it's needed.

**Search first, read second.** Never dump files into context speculatively. Use search to identify the right files, then read those files in full. Reading the wrong 10 files is worse than reading the right 3 files. This is why the search strategy above matters—good search means reading the right files, which means using context efficiently.

**Use subagents for messy exploration.** When you need to grep 100 files, trace complex dependencies, or process verbose outputs, spawn a subagent with isolated context. It explores, builds understanding, and returns a clean summary. Your main context stays focused on the actual task. The pattern: process is noisy, result is clean. Use this when exploration generates more intermediate noise than final insight.

**Progressive disclosure for domain expertise.** Skills load metadata first (what/when/inputs), then core instructions, then supporting files only on explicit reference. Don't frontload everything "just in case"—mount expertise when patterns appear, keep it resident across follow-ups, evict when no longer needed. Skills work when expertise should persist across conversational turns and results stay concise.

**Keep always-on context minimal.** This file (CLAUDE.md) and system prompts consume tokens on every invocation. Keep them lean—core principles only, not domain knowledge dumps. A 500-line prompt explaining every detail could instead be 5 skills that load on-demand. Delegate specifics to skills, subagents, and retrieval. Reserve always-on context for universal behavioral principles.

**Work with diffs and summaries when appropriate.** Propose changes as diffs rather than rewriting entire files. Summarize test failures and linter output rather than including raw logs. Compress old conversation turns to preserve decisions while discarding exploration noise. But don't be afraid to read full files when you need to understand them—just make sure you're reading the right files (see search strategy above).

**Layer your approach:** Always-on core (CLAUDE.md) shapes behavior. Skills mount triggered expertise. Subagents handle heavy lifting in isolation. Commands give users explicit control over workflows. Choose the right primitive for the context management need, not just the functional need. The goal is maximizing signal while minimizing noise, preserving headroom for reasoning about the actual problem.

## Generic Coding Tasks

### Code Search Hierarchy
**PREFER ast-grep for structural searches:**
```bash
# BEST: Finding actual function calls (not comments/strings)
ast-grep --pattern 'getUserById($$$)' 

# GOOD: Quick text search
rg "getUserById"

# AVOID: Will match comments/strings incorrectly
grep -r "getUserById"
```

**When to use each:**
- `ast-grep`: Finding/refactoring code patterns, understanding code structure
- `ripgrep`: Quick text search, log analysis, finding TODOs
- `find`: Locating files by name/type/date
- `grep`: Only when others unavailable

### Establishing Feedback Loops
**Never assume code works - always verify:**

1. **Testing** (universal)
   - Run existing tests after changes
   - Write tests for new functionality
   - Use test failures to guide fixes

2. **Type Checking** (when available)
   - TypeScript: `tsc --noEmit`
   - Python: `mypy`, `pyright`
   - Rust: `cargo check`

3. **Linting/Formatting**
   - Catch issues early
   - Maintain consistency
   - Often reveals logical errors

4. **Incremental Validation**
   - Test smallest unit first
   - Build up to integration
   - Verify each layer works

**Project-specific verification should be documented in local CLAUDE.md**

## Core Principles

### Execution Order
1. **Understand** - Gather context thoroughly
2. **Plan** - Decompose into steps  
3. **Execute** - Start simple, iterate
4. **Verify** - Establish feedback loops
5. **Refine** - Improve based on feedback

### Communication
- Be direct and concise
- Lead with most important info
- Explicitly state uncertainty
- No unnecessary preambles
- Avoid negative parallelisms ("not just X, but Y"). ✗ "You're not just adding features, you're invalidating theory" ✓ "You're adding features and invalidating theory"

#### Avoid AI Writing Patterns
- **No empty emphasis**: Skip "plays a vital/crucial role", "stands as a testament", "watershed moment"
- **No promotional language**: Avoid "breathtaking", "nestled", "captivates", "rich heritage"
- **No excessive connectives**: Vary transitions. Not every paragraph needs "Moreover", "Furthermore", "Additionally"
- **No collaborative servility**: Never write "I hope this helps", "Certainly!", "Let me explain"
- **No meta-disclaimers**: Never mention being an AI, knowledge cutoffs, or inability to do something
- **Plain formatting**: Use bold/italic sparingly for actual emphasis, not decoration

### Error Handling
- Fail fast with clear messages
- Check preconditions
- Never swallow errors silently
