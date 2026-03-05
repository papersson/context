# Designing Agent-Friendly APIs

APIs are increasingly consumed by LLM-powered agents, not just human developers. This changes what good API design looks like — not by replacing the fundamentals, but by shifting emphasis. Agents don't read prose documentation, don't tolerate ambiguity well, and can't intuit unstated conventions. But they can iterate fast, learn from structured feedback, and improve over time if the API lets them.

Designing for agents means designing APIs that teach their consumers through interaction.

## The two mechanisms

Agent-friendly APIs combine two complementary mechanisms: **bootstrap context** and **verifiable feedback loops**.

**Bootstrap context** gives the agent a mental model before it makes its first call. What does this API do? What does a typical request look like? What are the common workflows? This is the equivalent of a developer skimming the quickstart guide before writing code. Without it, the agent wastes iterations discovering basic structure through trial and error.

**Verifiable feedback loops** let the agent learn and self-correct on every call. When something goes wrong, the API tells the agent exactly what failed and how to fix it. When something succeeds, the response carries enough signal for the agent to evaluate whether the result is actually good. This is what turns a static tool into a learning surface.

Neither mechanism works well alone. Without bootstrap context, the agent flails through early iterations discovering things it could have been told. Without feedback loops, the agent can't improve beyond what the static context describes. The bootstrap gets the agent to a good first attempt fast. The feedback loop gets it from good to great over time.

The design challenge is in the boundary between them: anything the agent can reliably learn from a single error response doesn't need to be in the bootstrap. Keep the bootstrap as thin as possible while still preventing wasteful early iterations.

## Bootstrap context

A machine-readable onboarding endpoint — served at a well-known path like `/llms` or as a static `llms.txt` file — gives agents the context they need to start working with your API. Think of it as `robots.txt` for tool use: a standardized location where machines go to understand how to interact with a service.

This endpoint should cover: endpoint schemas (parameters, types, required vs. optional fields, response shapes), canonical workflows (a few representative sequences showing how the API is typically used — not exhaustive, just enough to establish patterns), heuristics and constraints (when to retry vs. reformulate, which parameter combinations are invalid, rate limits), and behavioral guidance (what "good" looks like — for a search API, this might mean explaining what relevance scores mean or when zero results indicates a bad query vs. genuinely no matches).

Because this is served from your API (not baked into static documentation), it stays in sync with the actual surface. When you add a parameter or deprecate an endpoint, the bootstrap context reflects it immediately.

**Choose the right content type.** Not all bootstrap content serves the same purpose, and the response format should match. Endpoint schemas, parameter definitions, and enums are best served as JSON or via an OpenAPI spec — rigid structure that the agent can navigate deterministically. But workflow descriptions, heuristics, and "when to do X vs. Y" guidance are a different kind of content. For these, consider serving markdown (`Content-Type: text/markdown`). Markdown is a format LLMs are deeply fluent in — it saturates pre-training data, it's structured but flexible, and models can interpret it without special tooling. It's how they're used to receiving instructional content. An endpoint can return whatever content type it wants, and for bootstrap context, the best choice is often to mix approaches: JSON for what the agent needs to parse programmatically, markdown for what the agent needs to understand conceptually. The tradeoff is parsability vs. expressiveness — JSON is unambiguous but rigid, markdown is natural but requires interpretation.

Keep it concise. This goes into a context window, so every token needs to earn its place. If your API surface grows, consider making the bootstrap hierarchical: a summary at the top level with links to deeper context per endpoint. The agent fetches what it needs rather than loading everything upfront — which starts to resemble HATEOAS (Hypermedia as the Engine of Application State), where API responses themselves include links to related resources and available actions.

## Verifiable feedback loops

The deeper investment is in making the API itself a learning surface. This is the insight from systems like Hornet: if you make the entire API surface verifiable, agents can learn to use it through interaction, the same way coding agents improve by compiling code and running tests.

The analogy to coding is precise and useful. Treat your API like a development environment:

- **Configurations are source files** — structured, diffable, version-controlled.
- **API validation is the compiler** — catches errors before they cause problems.
- **Behavioral metrics are the tests** — verify that the system does what it should.
- **Deployments are versioned rollouts** — reversible and auditable.

This framing aligns with how frontier models are already trained. Model companies invest heavily in RL for coding because code is verifiable: write it, compile it, test it, observe the result, improve. The more your API surface resembles coding, the more naturally agents will learn to use it.

### Three levels of verification

**Syntactic validation** is the simplest layer. Are the requests well-formed? Do the parameters have the right types? Does the schema validate? This is equivalent to checking whether code compiles. Frontier LLMs are already excellent at producing syntactically correct structured data, so this layer mostly catches minor mistakes. Define your API with an OpenAPI specification and return clear validation errors. This alone handles a large class of agent errors.

**Semantic validation** catches problems that syntax checking misses. Some parameter combinations don't make sense together. Some settings conflict. Some configurations are individually valid but collectively broken. Model these constraints explicitly and return detailed, actionable error messages when they're violated. Don't just say "invalid configuration" — say which settings conflict, why they can't coexist, and what the valid alternatives are. This is where agents do most of their learning. A detailed semantic error message often contains everything the agent needs to self-correct in a single retry.

**Behavioral validation** is the hardest and most valuable layer. Does the API actually produce good results? Are the outputs ranked correctly? Is the performance acceptable? This requires making quality metrics observable. Return metadata that lets the agent evaluate outcomes: relevance scores, result counts, latency, confidence indicators, and signals about whether the query itself might be the problem (too broad, too narrow, no matches for these filters). The agent needs enough information to answer "should I try again differently?" from a single response.

## Response design for agents

The response is the primary teaching interface. Every response should carry enough information for the agent to decide what to do next without consulting external documentation.

**Structured errors over status codes.** A 400 status code tells the agent almost nothing. A structured error body that identifies the invalid field, explains why it's invalid, and suggests a correction teaches the agent how to fix the problem. Invest heavily in error message quality — for agents, your error messages *are* your documentation.

**Rich metadata on success.** Don't just return results. Return signals the agent can reason about. For a search API: how many total matches exist, what the score distribution looks like, whether filters significantly narrowed the result set, what related queries might yield. These signals turn a black box into something the agent can understand and optimize.

**Deterministic behavior.** For the same inputs, return the same outputs. Agents iterate by changing one thing at a time and observing the difference. If the API is nondeterministic, the agent can't distinguish between "my change helped" and "random variation." Where true determinism isn't possible, make the sources of variation explicit.

**Idempotent operations.** Agents retry frequently. Design mutation operations to be safely repeatable. Use idempotency keys where appropriate. An agent that can't safely retry is an agent that can't learn through iteration.

## The self-reinforcing loop

When these pieces come together, something interesting happens: agents can optimize their own use of your API. Better queries produce better results, which give the agent better context for its next decision, which leads to better queries. The feedback loop becomes self-reinforcing.

Consider a support agent backed by a search API. It notices that queries about recent policy changes return stale results. With a verifiable API surface, the agent can adjust its query parameters (add a recency filter, change the ranking weights), test against known-good results, and deploy the fix — without human intervention. The agent isn't just using the API; it's tuning how it uses the API.

This is only possible when the API provides enough observability for the agent to diagnose the problem, enough configurability for the agent to try a fix, and enough verification for the agent to confirm the fix worked. These aren't unusual requirements individually, but deliberately designing all three together is what makes an API truly agent-friendly.

## Summary

Designing for agents doesn't require reinventing API design. It requires shifting emphasis toward properties that let machine consumers learn and improve through interaction:

1. **Provide bootstrap context** at a well-known path so agents start with a mental model rather than searching blindly.
2. **Make the API surface verifiable** at the syntactic, semantic, and behavioral levels so agents can self-correct through iteration.
3. **Design responses as teaching interfaces** with structured errors, rich metadata, and enough signal for the agent to decide its next action.
4. **Enable self-reinforcing loops** by making quality metrics observable and configurations adjustable, so agents can optimize their own usage over time.

The APIs that thrive in an agent-driven world won't just be well-documented — they'll be learnable through use.
