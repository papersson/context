# Designing Agent-Friendly APIs

APIs are increasingly consumed by LLM-powered agents, not just human developers. This changes what good API design looks like — not by replacing the fundamentals, but by shifting emphasis. Agents don't read prose documentation, don't tolerate ambiguity well, and can't intuit unstated conventions. But they can iterate fast, learn from feedback, and improve over time if the API lets them.

Designing for agents means designing APIs that teach their consumers through interaction. And "API" here means any programmatic interface — HTTP endpoints, CLIs, SDKs, anything an agent invokes and gets a response from.

## The two mechanisms

Agent-friendly APIs combine two complementary mechanisms: **bootstrap context** and **verifiable feedback loops**.

**Bootstrap context** gives the agent a mental model before it makes its first call. What does this API do? What does a typical request look like? What are the common workflows? This is the equivalent of a developer skimming the quickstart guide before writing code. Without it, the agent wastes iterations discovering basic structure through trial and error.

**Verifiable feedback loops** let the agent learn and self-correct on every call. When something goes wrong, the API tells the agent exactly what failed and how to fix it. When something succeeds, the response carries enough signal for the agent to evaluate whether the result is actually good. This is what turns a static tool into a learning surface.

Neither mechanism works well alone. Without bootstrap context, the agent flails through early iterations discovering things it could have been told. Without feedback loops, the agent can't improve beyond what the static context describes. The bootstrap gets the agent to a good first attempt fast. The feedback loop gets it from good to great over time.

The design challenge is in the boundary between them: anything the agent can reliably learn from a single error response doesn't need to be in the bootstrap. Keep the bootstrap as thin as possible while still preventing wasteful early iterations.

## Bootstrap context

An agent needs enough context to make a reasonable first attempt. How that context is delivered varies by interface — a well-known endpoint like `/llms.txt`, a `--help` flag, a discovery response — but the function is the same: teach the consumer the surface area before they start making real calls.

This should cover: what endpoints or commands exist and what they do, what the parameters are and which ones matter, a few canonical workflows showing typical usage, and behavioral guidance (what "good" looks like — for a search API, this might mean explaining what relevance scores mean or when zero results indicates a bad query vs. genuinely no matches).

The best bootstrap context is served from the API itself, not maintained in separate documentation. When you add a parameter or deprecate an endpoint, the bootstrap reflects it immediately. A CLI that derives its help text from its command definitions gets this for free. An HTTP API that serves its onboarding context from the same source as its routing logic gets the same benefit.

Keep it concise. If this goes into a context window, every token needs to earn its place. If your surface is large, consider making the bootstrap hierarchical: a summary at the top level with links to deeper context per area. The agent fetches what it needs rather than loading everything upfront — which is essentially HATEOAS (Hypermedia as the Engine of Application State), where responses themselves include links to related resources and available actions.

## Verifiable feedback loops

The deeper investment is in making the API itself a learning surface. This is the insight from systems like Hornet: if you make the entire API surface verifiable, agents can learn to use it through interaction, the same way coding agents improve by compiling code and running tests.

The analogy to coding is precise and useful. Think of your API as a development environment:

- **Configurations are source files** — what the agent writes and submits.
- **Validation is the compiler** — catches errors before they cause problems.
- **Behavioral metrics are the tests** — verify that the system does what it should.
- **Deployments are versioned rollouts** — reversible and auditable.

This framing matters because it aligns with how frontier models are already trained. Model companies invest heavily in RL for coding because code is verifiable: write it, compile it, test it, observe the result, improve. The more your API surface resembles this loop — submit, validate, observe, adjust — the more naturally agents will learn to use it.

### Three levels of verification

**Syntactic validation** is the compiler. Are the requests well-formed? Do the parameters have the right types? Does the schema validate? Frontier LLMs are already excellent at producing syntactically correct requests, so this layer mostly catches minor mistakes. But when it does catch something, the error message matters enormously — it's the difference between the agent self-correcting in one retry or flailing for five.

**Semantic validation** catches problems that syntax checking misses. Some parameter combinations don't make sense together. Some settings conflict. Some configurations are individually valid but collectively broken. Model these constraints explicitly. Don't just say "invalid configuration" — say which settings conflict, why they can't coexist, and what the valid alternatives are. This is where agents do most of their learning. A good semantic error often contains everything the agent needs to self-correct in a single retry.

**Behavioral validation** is the hardest and most valuable layer. Does the API actually produce good results? This requires making quality metrics observable. Return metadata that lets the agent evaluate outcomes: relevance scores, result counts, latency, confidence indicators, signals about whether the query itself might be the problem (too broad, too narrow, no matches for these filters). The agent needs enough information to answer "should I try again differently?" from a single response.

## Response design for agents

The response is the primary teaching interface. Every response should carry enough information for the agent to decide what to do next without consulting external documentation.

**Informative errors over codes.** A 400 status code tells the agent almost nothing. An exit code of 1 tells it even less. What matters is whether the error response carries enough information for the agent to understand the failure and correct it. Compare a CLI that says `Error: unknown command "subdomain"` and exits, to one that says the same thing and then prints every valid command grouped by category. Same error, completely different learning outcome. The format doesn't matter — prose, plain text tables, whatever — but the information content does. Invest heavily in error message quality. For agents, your error messages *are* your documentation.

Consider the Wrangler CLI: run a nonexistent subcommand and it tells you the command doesn't exist, then prints the full command tree organized by category. That single error response is both a correction ("this isn't valid") and a bootstrap ("here's everything that is"). An agent recovers in one retry. This is the pattern to aim for regardless of interface type.

**Rich metadata on success.** Don't just return results. Return signals the agent can reason about. For a search API: how many total matches exist, what the score distribution looks like, whether filters significantly narrowed the result set, what related queries might yield. For a CLI: exit codes that distinguish between "no results" and "error," output that shows what was actually done, not just that it succeeded. These signals turn a black box into something the agent can understand and optimize.

**Deterministic behavior.** Agents iterate by changing one thing at a time and observing the difference. If the API is nondeterministic, the agent can't distinguish between "my change helped" and "random variation." For the same inputs, return the same outputs. Where true determinism isn't possible, make the sources of variation explicit so the agent can account for them.

**Safe to retry.** Agents retry frequently — it's how they learn. Design mutation operations to be safely repeatable. Use idempotency keys where appropriate. An agent that can't safely retry is an agent that can't learn through iteration.

## The self-reinforcing loop

When these pieces come together, something interesting happens: agents can optimize their own use of your API. Better queries produce better results, which give the agent better context for its next decision, which leads to better queries. The feedback loop becomes self-reinforcing.

Consider a support agent backed by a search API. It notices that queries about recent policy changes return stale results. With a verifiable API surface, the agent can adjust its query parameters (add a recency filter, change the ranking weights), test against known-good results, and deploy the fix — without human intervention. The agent isn't just using the API; it's tuning how it uses the API.

This is only possible when the API provides enough observability for the agent to diagnose the problem, enough configurability for the agent to try a fix, and enough verification for the agent to confirm the fix worked. These aren't unusual requirements individually, but deliberately designing all three together is what makes an API truly agent-friendly.

## Summary

Designing for agents doesn't require reinventing API design. It requires shifting emphasis toward properties that let machine consumers learn and improve through interaction:

1. **Provide bootstrap context** so agents start with a mental model rather than discovering the surface through trial and error.
2. **Make the surface verifiable** at the syntactic, semantic, and behavioral levels so agents can self-correct through iteration.
3. **Design responses as teaching interfaces** with informative errors, rich metadata, and enough signal for the agent to decide its next action.
4. **Enable self-reinforcing loops** by making quality metrics observable and configurations adjustable, so agents can optimize their own usage over time.

The APIs that thrive in an agent-driven world won't just be well-documented — they'll be learnable through use.
