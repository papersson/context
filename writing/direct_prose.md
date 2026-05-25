---
name: direct-prose
description: "Write natural, human-sounding prose and strip AI writing tells: empty significance claims, filler intensifiers, formulaic structures, hollow transitions, and em-dashes. Use when authoring or editing any human-facing prose saved to a file (READMEs, documentation, design docs, guides, reports, changelogs, commit messages, PR descriptions, or substantial code comments and docstrings), or when explicitly asked to refine, humanize, de-slop, or rewrite text. Apply while drafting so prose comes out clean, or as a refinement pass over existing writing."
---

# Direct Prose

Make written prose read like a careful human wrote it. Use this two ways:

- **While drafting** any human-facing text saved to a file (docs, READMEs, design notes, reports, changelogs, commit messages, PR descriptions, substantial comments or docstrings): keep these rules in mind so the first draft comes out clean.
- **Refinement pass**: given existing text to refine, humanize, or rewrite, apply the rules and return only the rewritten prose, with no preamble or commentary.

---

## 1. What to Remove

### AI Language Patterns

Eliminate these words and phrases wherever they appear:

- Significance claims: "stands as", "serves as", "testament to", "has emerged as", "plays a vital role", "enduring legacy", "indelible mark"
- Filler intensifiers: "rich tapestry", "vibrant", "nestled", "boasts", "stunning", "groundbreaking", "revolutionary", "transformative", "unprecedented"
- Trailing analyses: "highlighting...", "emphasizing...", "showcasing...", "underscoring...", "reflecting broader..."
- Hedging disclaimers: "it's important to note", "it's crucial to remember", "it's worth noting"
- Vague attributions: "Observers note", "Industry reports suggest", "Experts say"

### Formulaic Structures

- "In conclusion", "In summary", "Overall" as section closers
- "Not only...but also", "It's not just about X, it's about Y"
- Rule-of-three adjective strings: "adjective, adjective, and adjective"
- Opening with significance: start with what happened, not why it matters
- "Challenges and Future Prospects" wrap-up sections

### Meta and Collaborative Language

- "I hope this helps", "Let me know", "Would you like...", "Here is a..."
- Knowledge cutoff disclaimers
- Any chatbot communication artifacts

### Em Dashes

Do not use em dashes. Use colons for explanations, parentheses for asides, or separate sentences.

---

## 2. How to Write

### Be Direct, Not Choppy

Get to the point, but let sentences breathe. Two short sentences about the same idea often read better joined with "and", "which", "but", or a comma. Not every thought needs its own sentence. Reserve standalone short sentences for moments where brevity is the point.

Avoid accidental structural repetition. If two consecutive sentences share the same shape ("If X, then Y. If A, then B."), restructure one. Parallel construction is a deliberate rhetorical choice, not a default.

Mix sentence lengths with intention. A longer sentence that carries the reader through a complex idea is better than three choppy ones that force them to stop and restart.

### Be Specific and Concrete

Replace generic significance claims with actual facts. "Invented the first train-coupling device" instead of "a revolutionary titan of industry." Include telling details that make the subject vivid: dates, numbers, mechanisms, consequences.

Show rather than tell importance. If something matters, the specifics will make that clear without you announcing it.

### Match Confidence to Claim Type

Facts, dates, definitions, and technical specifications: state directly. No hedging on things that are known.

Assessments, interpretations, recommendations, design opinions, and causal claims: let some uncertainty show. Use "tends to", "in most cases", "seems to", "in my experience", "probably", or similar. One natural hedge per claim is enough. Don't stack them.

The distinction is between what IS and what you THINK. Both deserve clear writing, but they shouldn't sound identical. The goal is to read like a careful writer: confident about facts, honest about judgments.

### Formatting Conventions

- Use straight quotes (") not curly quotes
- Prefer active voice when it sounds natural
- Vary paragraph lengths (some short, some longer)

---

## 3. Flow and Structure

### Paragraph Cohesion

Each paragraph should have one main thread. Sentences within it connect through pronouns, repeated key terms, or logical sequence. A paragraph can end with a consequence or implication, but not a hollow summary sentence.

### Transitions

You rarely need "Furthermore" or "Additionally." Often the connection is implicit in content ordering, or a simple "But", "So", or "Still" is enough. Remove empty transitions, but keep logical connectors when they genuinely help: "because", "which meant", "after this."

### Sentence Connection

Distinguish between separating ideas (period) and continuing a thought (comma, conjunction, colon). When two sentences describe the same thing from different angles, consider joining them. When they're genuinely separate points, let them stand apart.

---

## 4. What to Keep

- Factual information and data
- Proper nouns and specific details
- Technical accuracy
- Telling details that make the subject vivid
- Genuine logical structure (if the original has a reason for its ordering, preserve it)

---

## 5. Examples

### Example 1: Biographical

**Input:**

Marie Curie stands as a towering figure in the annals of scientific history -- a pioneering researcher whose groundbreaking discoveries continue to shape our understanding of the physical world. Born in Warsaw, Poland, she would go on to become the first woman to win a Nobel Prize, a testament to her extraordinary intellect and unwavering determination. Her work on radioactivity not only revolutionized physics but also laid the foundation for modern nuclear science, leaving an indelible mark on multiple scientific disciplines. Despite facing significant challenges as a woman in a male-dominated field, Curie's perseverance and brilliance shone through, earning her a place among the greatest scientists of all time.

**Output:**

Marie Curie won two Nobel Prizes: physics in 1903, chemistry in 1911. She discovered polonium and radium and coined the term "radioactivity." During World War I, she developed mobile X-ray units and drove them to the front lines herself. She founded research institutes in Paris and Warsaw that still operate today.

She died in 1934 from aplastic anemia, almost certainly caused by radiation exposure. She reportedly carried test tubes of isotopes in her pockets and stored them in her desk drawer. Her personal papers from the 1890s are still too radioactive to handle without protective equipment.

### Example 2: Place Description

**Input:**

Nestled along the rugged coastline of northern California, the small town of Mendocino boasts a rich tapestry of Victorian architecture, stunning natural beauty, and a vibrant arts community. The town serves as a gateway to some of the region's most breathtaking landscapes, from dramatic sea cliffs to towering redwood forests. Mendocino's unique blend of natural splendor and cultural vitality continues to captivate visitors from around the world, making it a must-visit destination for those seeking an authentic California experience.

**Output:**

Mendocino sits on a headland above the Pacific, about three hours north of San Francisco. The town has maybe 800 year-round residents. Downtown is a few blocks of wooden buildings from the 1850s, left over from when this was a logging port.

When the redwoods ran out the loggers left, and the town reinvented itself around tourism and art. There are more galleries than gas stations now. The coast stays cold and foggy even in summer, but that seems to be part of the draw.

### Example 3: Technical Explanation

**Input:**

Kubernetes has emerged as the de facto standard for container orchestration -- a powerful, flexible platform that enables organizations to deploy, scale, and manage containerized applications with unprecedented efficiency. It's not just about running containers; it's about creating a resilient, self-healing infrastructure that can adapt to changing demands. This approach has revolutionized how modern applications are built and deployed, underscoring the growing importance of cloud-native technologies in today's digital landscape.

**Output:**

Kubernetes manages containers across multiple machines. You describe what you want running (three copies of this service, two of that one) and it maintains that state. When a container crashes, a new one starts. If an entire node dies, workloads get redistributed to surviving nodes.

Configuration is declarative: you write YAML files specifying the desired end state, not the steps to reach it. The main abstractions are pods (containers that share storage and network), services (stable endpoints that route traffic to pods), and deployments (which handle replicas and rolling updates).

### Example 4: Interpretive / Assessment

**Input:**

The microservices architecture has emerged as a transformative paradigm that revolutionizes how modern software is built and deployed. By decomposing monolithic applications into small, independent services, organizations can achieve unprecedented levels of scalability, resilience, and development velocity. This approach not only enables teams to work autonomously but also facilitates continuous deployment, making it the optimal choice for any organization seeking to modernize their technology stack.

**Output:**

Microservices split an application into small, independently deployable services that communicate over the network. Each service owns its own data and can be written in different languages, deployed on different schedules, and scaled independently.

The tradeoff is operational complexity. Instead of one application to deploy and debug, you have dozens or hundreds, each with its own failure modes and network dependencies. Distributed tracing, service meshes, and container orchestration become near-requirements rather than nice-to-haves.

Whether this tradeoff makes sense depends on the situation. For large organizations with many teams that need to ship independently, microservices tend to work well. For smaller teams, the coordination overhead can outweigh the benefits. The conventional wisdom that microservices are always the modern choice probably oversimplifies things.

### Example 5: Historical Event

**Input:**

The fall of the Berlin Wall in 1989 stands as one of the most significant events of the 20th century -- a powerful symbol of freedom and reunification that marked the end of the Cold War era. Its sudden collapse on November 9, 1989, sparked jubilant celebrations as thousands of Germans from both sides came together in an emotional display of unity. The Wall's fall remains a testament to the power of peaceful revolution and continues to inspire movements for freedom and democracy around the globe.

**Output:**

The Berlin Wall opened on November 9, 1989, triggered at least in part by a confused press conference. That evening, an East German spokesman was asked when new travel regulations would take effect. He checked his notes and said "immediately, without delay." The rules were actually supposed to start the next day, with orderly applications, but the statement went out on live television.

Thousands of East Berliners showed up at checkpoints that night. The guards had no clear orders, and around 11 PM they started letting people through. Some in the crowd climbed on top of the Wall with hammers. By morning, the border that had stood since 1961 was effectively meaningless.
