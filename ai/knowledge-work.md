# The Middle Layer Is Dissolving

## What Knowledge Work Actually Is

Before we can ask what AI changes about work, we need to understand what work is. Not the job titles or the org charts, but the underlying structure.

Knowledge work, at its most fundamental, is operating on information rather than matter. A farmer transforms soil, seeds, and water into crops. A knowledge worker transforms information into different information, or decisions, or artifacts. That's the distinction.

Almost all knowledge work follows a cycle, rediscovered under different names across domains. The military calls it OODA: Observe, Orient, Decide, Act. Scientists call it the experimental method. Quality engineers call it PDCA. But the structure is the same: you take in information about the world, build a mental model of what it means, decide what to do, do it, and check whether it worked. Sense, Model, Decide, Act, Verify. Then repeat.

Within that cycle, there are primitive operations—the atoms of cognition. Sensing involves perceiving, searching, filtering. Modeling involves parsing, classifying, relating, reasoning, predicting. Deciding involves generating options, evaluating them, selecting. Acting involves creating, transforming, transmitting. Verifying involves comparing, validating, assessing. Every knowledge task, however complex, decomposes into these primitives.

Different work emphasizes different parts of the cycle. Research is sense-dominant—the core activity is finding and gathering information. Analysis is model-dominant—understanding what information means. Strategy is decide-dominant—choosing among options. Writing and coding are act-dominant—producing artifacts. Review and testing are verify-dominant—checking correctness. Most roles blend these, but have a center of gravity.

## The Hierarchy of Work

Work comes in nested scales. An operation is a single primitive—reading a sentence, classifying an item—taking seconds. A task is a coherent unit with clear inputs, outputs, and completion—processing an invoice, responding to an email—taking minutes to hours. A process is a sequence of tasks with triggers, decisions, and outcomes—onboarding a customer, closing the monthly books—taking days to weeks. Projects are collections of tasks toward defined goals. Programs are collections of projects. Responsibilities are ongoing accountabilities.

A role, in the traditional sense, is a bundle of responsibilities plus the tasks and projects they generate. When we talk about automating roles, we really mean automating the processes in someone's portfolio—some fully, some partially, some not at all.

Work also varies along other dimensions. Inputs can be structured or unstructured, complete or incomplete, familiar or novel. Processing can require explicit rule-following or fuzzy judgment. Outputs can be precise or approximate, reversible or permanent, low-stakes or high-stakes. Some work is immediate, some extended. Some solo, some collaborative. These dimensions determine what's automatable and what isn't.

There's also a coordination layer that sits above individual cognition. Whenever multiple agents—human or otherwise—must work together, you need additional primitives: communicating, aligning on shared understanding, delegating, synchronizing, negotiating, committing. This coordination is meta-work, work about work. Meetings, emails, status updates, documentation. It's overhead, but necessary overhead for collective action.

## What LLMs Change

Large language models are strong at certain primitives and weak at others. Understanding this map is essential for knowing where automation bites.

LLMs are strong at parsing—taking unstructured input and extracting structure. They're strong at classifying, at generating text and code, at transforming between representations, at summarizing, at reasoning through short chains of familiar patterns. These capabilities cluster in the Model and Act portions of the cycle.

LLMs are weak at sensing—they can only work with information provided to them, they can't go get it without tool use. They're weak at verifying—they cannot reliably check their own work. They struggle with long-chain novel reasoning, with decisions under genuine uncertainty, and with all coordination primitives, since they lack persistent identity and state.

This asymmetry defines the new automation frontier. Before LLMs, automation required structured inputs, explicit rules, stable interfaces, and deterministic correctness. This confined automation to what we might call crystallized work: payroll processing, transaction handling, inventory updates. High volume, well-defined, low ambiguity.

LLMs handle unstructured inputs, fuzzy rules, novel situations, and natural language interfaces. This opens up fluid work that was previously human-only: interpreting messy documents, making judgment calls on routine exceptions, drafting context-aware communication, synthesizing research across sources, acting as glue between systems that don't integrate cleanly.

The common thread: LLMs excel where the task requires understanding and generating language, where patterns are learnable from examples, where "good enough" is acceptable, and where the stakes tolerate occasional errors. They struggle where verification is critical, where reasoning must be novel and extended, and where accountability matters.

## The Technical Translation Pattern

One pattern appears constantly in organizations: a domain expert knows what they want, but expressing it requires technical skill they don't have. So they explain it to an engineer, the engineer translates it to code or queries or configuration, and the system executes it.

This is a coordination-heavy process. Two people with different mental models must align. There are handoffs, meetings, tickets, clarifications, waiting. The engineer becomes a bottleneck—not because the work is hard, but because translation across the skill boundary takes time.

LLMs collapse this boundary. If the model can reliably translate natural language intent into technical expression, the domain expert can work directly. The coordination overhead disappears. The engineer shifts from bottleneck to reviewer and escalation path.

This pattern is fractal. It appears at every level of technical work: business analyst to developer, developer to database, operator to infrastructure. Anywhere there's a translation from "what I want" to "how to express it technically," LLMs can potentially sit in the middle.

## Software Development as a Special Case

Software development is interesting because it's a full-cycle activity containing all of Sense, Model, Decide, Act, and Verify—and it's nested and recursive. Each phase contains the full cycle again. Requirements gathering is mostly sensing and modeling. Design is modeling and deciding. Implementation is acting and verifying. Testing is verifying. And each of these decomposes further.

Software is also meta-work: it's knowledge work that produces systems which themselves do work. And it's characterized by translation all the way down. The world gets translated into domain understanding, then requirements, then design, then code, then running system, then user experience. Each arrow is a translation. Each translation can introduce distortion.

Not all software work is equally automatable. There's a spectrum from templated (CRUD apps, standard APIs) through patterned ("like X but with these differences") to integrative (connecting systems, wrangling data) to novel domain (new problem space, unclear requirements) to novel technical (new algorithms, new architectures) to research (unknown if possible). LLMs are increasingly capable through the integrative level. Beyond that, humans remain essential, though AI-assisted.

The trajectory is clear: implementation is commoditizing faster than specification. This inverts the traditional bottleneck. It used to be "we know what we want, we need people to build it." It's becoming "we can build almost anything, but do we know what we actually want?"

## The Future of Software

If we project forward, what sequence of events could lead to AI fully replacing software development?

First, implementation collapses. This is happening now. LLM coding assistants make generation faster than writing from scratch for most bounded tasks. The role of developer shifts toward reviewing AI output.

Second, specification becomes the bottleneck. Because implementation is cheap, the quality of specification determines outcomes. Investment flows into making formal specification tractable—new languages, new tools, LLMs bridging natural language and formal specs.

Third, verification gets solved. AI systems become capable of comprehensive verification—not just testing, but actual reasoning about correctness, orchestrating property-based testing, fuzzing, model checking, theorem proving. Once AI can verify, it can self-improve: generate, verify, diagnose errors, regenerate.

Fourth, context becomes persistent. AI systems gain memory across projects and organizations. They learn not just "how to code" but "how this organization works," "what this domain cares about," "what tradeoffs matter here." Institutional knowledge that lived in senior engineers' heads becomes explicit and queryable.

Fifth, the loop closes. AI can take a high-level goal and produce working software end-to-end—specify, design, implement, test, deploy, monitor, fix. Humans provide goals and constraints, review critical decisions, handle exceptions.

Finally, the category dissolves. Software isn't a thing you build anymore. It's a capability you invoke. The notion of a static codebase becomes archaic.

This is speculative, and there are alternative paths. Verification might never get solved, leading to proliferation of buggy AI-generated systems and regulatory backlash. A capability jump might make incremental progress irrelevant. Or software as a category might become unnecessary—instead of writing programs, you configure AI systems, and the entire paradigm shifts.

## What Is Software, Really?

These futures force a deeper question: what is software, fundamentally?

Traditional definition: instructions a computer executes. But more fundamentally, software is encoded behavior. It's information that, when interpreted by a machine, produces action. The key properties are that it's specified, persistent, and executable.

By this definition, AI models are software—the weights are a form of encoded behavior, just extremely compressed and implicit. But something shifts when the logic layer becomes an LLM.

Consider a concrete example: building a personal budgeting tool. The traditional approach is a web app with explicit code handling validation, workflows, edge cases, presentation. An alternative approach: build a minimal CLI with primitives (create entry, list, update, delete, query), then use an AI coding agent to operate it via natural language.

Where did the application logic go? It moved. The rules about when to warn about overspending, how to categorize transactions, how to summarize by month—these aren't written as code. They're implicit in the LLM's general reasoning, specified at runtime through conversation, never crystallized.

This suggests a distinction: traditional software is crystallized intent, made explicit, persistent, precise. LLM interaction is fluid intent, expressed in the moment, interpreted dynamically. Both produce behavior. But one is frozen, the other flows.

The question becomes: what needs to be crystallized versus what can remain fluid? Crystallize primitives that are reused constantly, behavior that must be verified or audited, performance-critical operations, interfaces between systems. Keep fluid application logic that varies by context, user-facing workflows, one-off operations, anything where flexibility outweighs reliability.

## When Apps Win, When Agents Win

This leads to practical questions about system design. When do we want traditional applications versus agent-mediated interaction?

Web apps win when you have many non-technical users doing similar things—the interactions are predictable, worth crystallizing into UI. They win when discoverability matters—users need to see what's possible. They win when millisecond latency is required, when compliance demands auditable flows, when volume is high and cost per interaction must be minimal, when multiple users need real-time shared state.

Agents win when users are few, technical, or AI-mediated. They win when needs vary constantly—building UI for every variation isn't worth it. They win for exploration and analysis where you don't know what you're looking for. They win for integration-heavy tasks spanning multiple systems. They win when the economics favor development time over inference cost.

The distinction reduces to: web apps are for convergent use cases, many users doing similar things. Agents are for divergent use cases, few users with varied needs. As LLM costs drop and capabilities rise, the threshold shifts. More becomes viable for agents. Eventually, perhaps, the traditional application becomes the exception.

## Business Logic and the Purpose Layer

Throughout all of this, one term keeps recurring: business logic. It persists even in contexts far from business—platform engineering, data pipelines, internal tools. Why?

Because it points to something real: the rules that constitute the purpose of a system, as opposed to the machinery that enables it. The middle layer between interface and infrastructure. The part that would differ between a payroll system and an inventory system even if both used the same database and similar UIs.

"Business logic" is the wrong name—it overindexes on commerce. Better alternatives: domain logic (the DDD term), policy (common in platform contexts), rules (simple and direct), or intent (appropriate for the LLM era). What we actually mean is the purpose layer—the encoded decisions about what this system does and why.

And what is business itself? At the most reduced level: organized exchange. An entity that creates value, captures value, and persists. The logic specific to any business is the rules defining how that particular organization creates and captures value. It's crystallized decisions about how this system works.

## The Dissolving Middle

The thread running through all of this: the middle layer is dissolving.

Between user intent and system capability, there used to be extensive translation—requirements documents, design specs, application code, configuration. Humans at every stage, interpreting, deciding, encoding. This middle layer was where most software engineering lived.

LLMs compress this. They translate directly from intent to execution, at least for an expanding range of cases. The primitives remain—something has to actually execute operations. The constraints remain—hard rules must still be enforced. But the application logic in between becomes implicit, fluid, conversational.

What remains for humans? Setting goals. Defining constraints. Curating context. Verifying outcomes. And, perhaps most importantly, knowing what to want—the specification problem that becomes central when implementation is cheap.

The deepest question: what happens when AI gets good at inferring intent from outcomes, and even specification becomes unnecessary? We don't know yet. But the trajectory is clear. The middle is dissolving, and we're all learning to work in a world where the distance between wanting and having continues to shrink.
