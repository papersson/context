---
name: anthropic-diagram
description: Generate editorial-style diagrams in the Anthropic blog visual style as .drawio files. Use this skill whenever the user wants to create a diagram, flowchart, architecture diagram, comparison chart, swimlane, or any visual that should look like Anthropic's blog article illustrations. Trigger on prompts like "draw a diagram", "create a flowchart", "visualize this process", "make an architecture diagram", "画流程图", "画架构图", "帮我画", or any request to turn text/process descriptions into a visual — even if the user doesn't say "Anthropic style" explicitly. This skill produces the calm, editorial, publication-quality look characteristic of Anthropic's technical blog.
---

# Anthropic-Style Diagram Skill

Generate draw.io diagrams that match the editorial, warm, minimalist visual style of Anthropic's blog article illustrations.

## Workflow

```
User text → DiagramSpec (written out as text) → Styled draw.io XML → .drawio file
```

---

## Step 1: Analyze the Request

Determine:
- **Main claim**: What is the one thing this diagram should make obvious?
- **Pattern**: Which visual pattern best serves that claim? (See the [Pattern Library](#pattern-library) below)
- **Reading direction**: left-to-right for workflows/comparisons; top-to-bottom for stacks/hierarchies

### Pattern Selection

| User Intent | Preferred Pattern |
|-------------|-------------------|
| Explain a sequence of steps | Linear Workflow |
| Show retries, iteration, refinement | Feedback Loop Workflow |
| Show decisions or branching logic | Decision Tree / Branch Workflow |
| Compare before/after or old/new | Split Comparison |
| Show components inside systems | Grouped Architecture |
| Show layered stack or hierarchy | Layered Stack |
| Show interactions across actors or systems | Swimlane Sequence |
| Show shared responsibility or overlap | Venn / Overlap |
| Show performance or metric comparison | Editorial Chart |
| Show one central concept with supporting ideas | Hub-and-Spoke |
| Show multiple workers operating in parallel | Parallel Fan-out / Fan-in |

When uncertain about pattern, default rules:
- Sequential steps → Linear Workflow
- System components/containment → Grouped Architecture
- Before/after or two approaches → Split Comparison
- Multiple actors over time → Swimlane
- Overlap/shared ownership → Venn
- Numerical contrast → Editorial Chart

### Pattern Priority Rule

When multiple patterns seem possible:
1. Prefer the pattern with the **clearest reading order**
2. Prefer the pattern with the **fewest crossing lines**
3. Prefer the pattern that supports the **main claim in one glance**
4. Prefer **comparison** over complicated topology when the core message is contrast
5. Prefer **grouped architecture** over workflow when containment matters more than time

---

## Step 2: Build the DiagramSpec

Before writing any XML, write out the diagram plan explicitly as text — this helps catch structural mistakes before committing to XML, and lets the user see your reasoning. Output the DiagramSpec in this format:

```
**DiagramSpec**

main_claim: [one sentence — what is the diagram making obvious?]
pattern: [primary pattern]
secondary_pattern: [optional, or none]
reading_direction: [left-to-right / top-to-bottom]
title: "Diagram Title"

nodes:
  - id: n1
    label: "Short label"
    semantic_type: [primary | secondary | tertiary | start | end | warning | decision | ai_llm | inactive | error]
    shape: [rect | pill | diamond | cylinder]
    group: [container_id if inside a container, else none]

connections:
  - from: n1
    to: n2
    label: ""          # keep short or empty
    style: [primary | optional | feedback | human | context | error]

groups:
  - id: g1
    label: "Panel title"
    type: [outer_panel | inner_panel | swimlane | soft_region]
    children: [n1, n2, ...]
```

Writing this out is an internal planning step — clarify the structure in your own reasoning before committing to XML. After writing the DiagramSpec, **proceed immediately to Step 3** without waiting for user confirmation. Read the [Pattern Library](#pattern-library) for layout rules per pattern type.

**Language consistency**: Write the DiagramSpec — and all node labels, titles, and edge labels in the final XML — in the same language the user used. If the user wrote in Chinese, the diagram text should be Chinese too. A diagram whose labels match the user's language feels natural and avoids the jarring effect of mixed-language visuals.

---

## Step 3: Generate draw.io XML

### Canvas setup

```xml
<mxGraphModel background="#F2EFE8" grid="0" tooltips="0" connect="0" arrows="0" fold="0" page="0" pageScale="1" pageWidth="1654" pageHeight="1169" math="0" shadow="0">
  <root>
    <mxCell id="0"/>
    <mxCell id="1" parent="0"/>
    <!-- all cells go here with parent="1" (or parent="container_id" if nested) -->
  </root>
</mxGraphModel>
```

Use `background="#F2EFE8"` (warm canvas) for most diagrams. Use `background="#FFFFFF"` only for very minimal, sparse diagrams with few elements.

### Title

```xml
<mxCell id="title" value="Your Diagram Title" style="text;html=1;strokeColor=none;fillColor=none;align=center;verticalAlign=middle;whiteSpace=wrap;overflow=hidden;fontStyle=1;fontSize=32;fontColor=#1F1F1C;" vertex="1" parent="1">
  <mxGeometry x="80" y="40" width="1200" height="50" as="geometry"/>
</mxCell>
```

### Node style strings by semantic type

Apply these styles based on the semantic role, not just aesthetics. The color encodes meaning.

| Semantic type | draw.io style |
|---|---|
| **Primary / Neutral** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#E6E2DA;strokeColor=#8C867F;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Secondary / Context** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#EAF4FB;strokeColor=#6FA8D6;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Tertiary / Control** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#EEEAF9;strokeColor=#9A90D6;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Start / Trigger** | `rounded=1;whiteSpace=wrap;arcSize=50;fillColor=#F8E9E1;strokeColor=#D88966;strokeWidth=1.8;fontColor=#D88966;fontSize=20;fontStyle=1;` |
| **End / Success** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#CFE8D7;strokeColor=#71AE88;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Warning / Reset** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#F3E4DA;strokeColor=#C88E6A;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Decision** | `rhombus;whiteSpace=wrap;fillColor=#E6D7B4;strokeColor=#BFA777;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **AI / LLM** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#D7E6DC;strokeColor=#7FB08F;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Inactive / Disabled** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#EFECE6;strokeColor=#B4AEA6;strokeWidth=1.8;fontColor=#7A756E;fontSize=20;` |
| **Error** | `rounded=1;whiteSpace=wrap;arcSize=10;fillColor=#F8DFDA;strokeColor=#D96B63;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Pill label** | `rounded=1;whiteSpace=wrap;arcSize=50;fillColor=#FAF8F4;strokeColor=#8C867F;strokeWidth=1.8;fontColor=#2D2B28;fontSize=20;` |
| **Code/evidence block** | `rounded=1;whiteSpace=wrap;arcSize=6;fillColor=#EEF3F7;strokeColor=#B7C9D8;strokeWidth=1.5;fontColor=#44515C;fontSize=20;align=left;` |

### Semantic interpretation

Use this mapping when converting user intent into styled nodes:

- **Primary/Neutral**: default blocks, generic system components, standard containers
- **Secondary / Context**: files, skills, tools, docs, retrieved context, storage-like resources
- **Tertiary / Control**: routers, memory, evaluators, aggregators, orchestration, policy layers
- **Start/Trigger**: user input, prompt input, external trigger, human intervention
- **End/Success**: verified result, completed output, accepted answer, successful state
- **Warning/Reset**: retry, reset, re-entry, interrupt, caution, manual steering
- **Decision**: branch points, filters, approval logic, yes/no gates
- **AI/LLM**: model calls, agent workers, active execution stages, reasoning stages
- **Inactive/Disabled**: optional items, de-emphasized elements, unavailable or future elements
- **Error**: failures, blocked execution, invalid state, rejected result, deny path

If semantic meaning is unclear:
1. Default to **Primary/Neutral**
2. Use **Secondary / Context** for passive information objects
3. Use **AI/LLM** for active computational steps
4. Use **Tertiary / Control** for routing or memory-like logic
5. Use **Start/Trigger** only when something truly initiates flow

If too many colors are present:
- Collapse low-priority semantics into **Primary/Neutral**
- Keep only the 2-3 most meaningful accent categories

### Container / panel styles

All container styles include `html=1;` so that the `value` attribute can contain HTML. Use a `<font>` tag in the `value` to control the label font size (typically 2-4px larger than the `fontSize` in the style string, for visual hierarchy).

**Outer panel** (large system boundary):
```
rounded=1;whiteSpace=wrap;arcSize=4;fillColor=#FAF8F4;strokeColor=#8C867F;strokeWidth=2;fontSize=18;fontStyle=1;fontColor=#5F5A54;swimlane;startSize=63;horizontal=1;html=1;
```
value: `<font style="font-size: 22px;">Panel Title</font>`

**Inner panel** (subsystem or grouping):
```
rounded=1;whiteSpace=wrap;arcSize=6;fillColor=#FAF8F4;strokeColor=#8C867F;strokeWidth=1.8;fontSize=16;fontStyle=1;fontColor=#5F5A54;swimlane;startSize=50;horizontal=1;html=1;
```
value: `<font style="font-size: 20px;">Panel Title</font>`

**Soft region** (dashed grouping, no strong boundary):
```
rounded=1;fillColor=#F6F4EE;strokeColor=#B9B3AB;strokeWidth=1.5;dashed=1;dashPattern=6 6;fontSize=16;fontColor=#7A756E;html=1;
```
value: `<font style="font-size: 18px;">Region Label</font>`

Example XML for an outer panel:
```xml
<mxCell id="panel_server" parent="1" style="rounded=1;whiteSpace=wrap;arcSize=4;fillColor=#FAF8F4;strokeColor=#8C867F;strokeWidth=2;fontSize=18;fontStyle=1;fontColor=#5F5A54;swimlane;startSize=63;horizontal=1;html=1;" value="&lt;font style=&quot;font-size: 22px;&quot;&gt;Panel Title&lt;/font&gt;" vertex="1">
  <mxGeometry x="580" y="110" width="480" height="920" as="geometry"/>
</mxCell>
```

Children of containers use `parent="container_id"` and coordinates relative to the container.

### Container rules

- Prefer rounded rectangles over sharp-cornered boxes.
- Use pills for small categorical labels or compact component tags.
- Use soft grouping regions to imply relationship without adding visual clutter.
- Avoid nested containers deeper than 3 levels.
- Most diagrams should have 1 outer grouping level and 0-2 inner grouping levels.

### Connector styles

**The single most important style rule**: all arrows use open chevron arrowheads.

```
endArrow=open;endSize=14;
```

This is what gives Anthropic diagrams their clean, editorial look. Never use filled/block arrowheads.

All connectors also use `edgeStyle=orthogonalEdgeStyle` — this makes lines route with right-angle bends and gives the diagram a clean, structured feel. Combined with `rounded=1`, the corners are softened into smooth curves rather than hard 90-degree turns.

| Connector type | Full style |
|---|---|
| **Primary flow** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#7A756E;strokeWidth=1.8;rounded=1;exitX=1;exitY=0.5;entryX=0;entryY=0.5;` |
| **Optional / inferred** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#9A948C;strokeWidth=1.6;rounded=1;dashed=1;dashPattern=6 6;` |
| **Feedback loop** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#8E8982;strokeWidth=1.8;rounded=1;curved=1;` |
| **Human override** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#D88966;strokeWidth=1.8;rounded=1;dashed=1;dashPattern=6 6;` |
| **Context / support** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#7FB08F;strokeWidth=1.8;rounded=1;` |
| **Error path** | `endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#D96B63;strokeWidth=1.8;rounded=1;` |

Every edge cell must have a child geometry element — never self-close:
```xml
<mxCell id="e1" edge="1" source="n1" target="n2" style="endArrow=open;endSize=14;edgeStyle=orthogonalEdgeStyle;strokeColor=#7A756E;strokeWidth=1.8;rounded=1;" parent="1">
  <mxGeometry relative="1" as="geometry"/>
</mxCell>
```

### Connector grammar

Use connectors consistently to preserve meaning.

| Connector Style | Meaning |
|-----------------|---------|
| Solid | primary path |
| Dashed | optional, inferred, soft relationship, human-overridable |
| Curved return | feedback loop |
| Orthogonal | architecture/system interaction |
| Straight | simple step progression |
| No arrowhead | grouping, alignment, weak association |

Rules:
- Do not let connectors do layout work — layout first, connect second
- Minimize crossings
- Label connectors only when the transformation is important
- Main flow should be visually obvious within 3 seconds of viewing
- Dashed lines indicate optionality, inferred paths, soft grouping, or human-overridable behavior
- Use dashed connectors sparingly; too many make the diagram feel uncertain

### Text colors (hierarchy)

Use color on free-floating text to create visual hierarchy without needing extra containers.

| Level | Color | Use For |
|-------|-------|---------|
| Title | #1F1F1C | Main diagram title |
| Subtitle | #5F5A54 | Section headers, panel titles |
| Body/Detail | #4F4A44 | Labels, annotations, helper copy |
| Muted/Support | #7A756E | Secondary annotations, non-primary labels |
| On light fills | #2D2B28 | Text inside most boxes and panels |
| On dark fills | #FAF8F4 | Rare use only; avoid dark fills by default |

Rules:
- Use **Title** for the single main headline only.
- Use **Subtitle** for section names, panel labels, and major group headings.
- Use **Body/Detail** for node labels, arrows, notes, and supporting explanation.
- Use **Muted/Support** for optional captions or low-priority labels.
- Prefer free-floating text over unnecessary containers.

### Evidence artifact styles

Used for code snippets, data examples, and other concrete evidence inside technical diagrams.

| Artifact | Background | Border | Text Color |
|----------|------------|--------|------------|
| Code snippet | #EEF3F7 | #B7C9D8 | #44515C |
| JSON/data example | #EEF4EE | #B9CCBE | #46574C |
| Terminal/CLI example | #F3F1EC | #C8C1B6 | #4C4943 |
| File tree / spec excerpt | #EDF2F8 | #B8C6D8 | #506070 |

Rules:
- Evidence blocks should look quieter than primary diagram nodes.
- Use evidence artifacts only when concrete examples improve understanding.
- Do not make evidence artifacts the most visually dominant element unless the diagram is explicitly evidence-driven.

### Stroke & line colors

| Element | Color |
|---------|-------|
| Structural lines (dividers, trees, timelines) | #8C867F |
| Secondary structural lines | #9A948C |
| Marker dots (fill) | #E6E2DA |
| Marker dots (stroke) | #8C867F |
| Grouping boundary (solid) | #8C867F |
| Grouping boundary (dashed) | #B9B3AB |

---

## Step 4: Layout rules

These rules keep diagrams feeling calm and well-composed:

- **Node spacing**: minimum 80px horizontal gap between adjacent nodes; 60px vertical gap
- **Recommended horizontal pitch**: 200px center-to-center for workflow steps
- **Recommended vertical pitch**: 120px center-to-center for parallel elements
- **Canvas padding**: 60px around the outermost content
- **Grid alignment**: snap all positions to multiples of 10
- **Node size**: standard nodes 140x60 to 180x70; wide containers 300-600+; title bar height 40
- **Keep layout flat**: max 3 nesting levels; prefer whitespace over extra containers

For pattern-specific layout rules (swimlane lane widths, comparison panel sizing, fan-out spacing, etc.), see the [Pattern Library](#pattern-library) section below.

### Geometry tokens

| Token | Value |
|-------|-------|
| Small radius | 10 |
| Medium radius | 14 |
| Large radius | 20 |
| Pill radius | 999 |
| Default stroke width | 1.8 |
| Panel stroke width | 2.0 |
| Dashed pattern | 6 6 |
| Small arrow size | 10 |
| Default arrow size | 12 |

Rules:
- Keep radii consistent across the diagram.
- Prefer a single radius family per diagram.
- Avoid mixing very thick and very thin strokes.
- Use panel strokes slightly heavier than internal strokes.

### Typography & spacing

| Token | Value |
|-------|-------|
| Title size | 30-42 |
| Section title size | 18-24 |
| Body label size | 12-16 |
| Caption size | 11-13 |
| Title weight | 700-800 |
| Section weight | 600-700 |
| Body weight | 400-500 |
| Outer canvas padding | 48 |
| Section gap | 28 |
| Internal box padding | 12-18 |
| Grid step | 8 |
| Default node gap | 20-28 |
| Large panel gap | 32-48 |

Rules:
- The title should be large and editorial, not UI-like.
- Do not overuse bold inside nodes.
- Prefer short labels over wrapped paragraphs.
- In diagrams, spacing should do more work than font variation.

### Outer border

Every diagram gets a single outer border — a thin rounded rectangle that frames the entire composition (title + all content). This gives the diagram a poster-like finish and makes the whitespace feel intentional rather than accidental.

- Place this cell **first** in the XML, before all other nodes, so it renders behind everything
- `x=20, y=20`; width = pageWidth - 40 (-> **1614** for the standard 1654px canvas)
- Height: extend ~40px below the lowest node — snug enough to feel cohesive, loose enough to breathe
- Style: subtle warm stroke, no fill (the canvas background shows through)

```xml
<mxCell id="border" value="" style="rounded=0;arcSize=3;fillColor=none;strokeColor=#B9B3AB;strokeWidth=1.5;pointerEvents=0;" vertex="1" parent="1">
  <mxGeometry x="20" y="20" width="1614" height="1000" as="geometry"/>
</mxCell>
```

Adjust `height` so it ends roughly 40px below the bottom-most element. `pointerEvents=0` keeps it non-interactive so users can click through it in draw.io.

---

## Step 5: Write and open

1. Write the complete XML to a descriptive `.drawio` file in the current working directory.
   - Filename: lowercase with hyphens, e.g., `agent-loop.drawio`, `context-engineering.drawio`
2. Open the file: `start <filename>.drawio` (Windows) or `open <filename>.drawio` (macOS)

If the user asks for PNG/SVG export:
```bash
"C:\Program Files\draw.io\draw.io.exe" -x -f png -e -b 20 -o output.drawio.png input.drawio
```

---

## Quality checklist

Before finalizing the XML, verify:

- [ ] Outer border present (`id="border"`, first cell in XML, x=20 y=20, width=1614, height covers all content + 40px)
- [ ] Title is large (fontSize>=28), bold, dark (#1F1F1C), horizontally centered
- [ ] Canvas background set (`background="#F2EFE8"` in mxGraphModel)
- [ ] Every arrow uses `endArrow=open;endSize=14`
- [ ] Node colors follow semantic meaning, not decoration
- [ ] Main flow path is visually dominant within 3 seconds
- [ ] No more than 4 semantic accent colors in one diagram
- [ ] Every edge has `<mxGeometry relative="1" as="geometry"/>`
- [ ] All coordinates are multiples of 10
- [ ] No `--` inside XML comments (invalid XML)
- [ ] Special characters escaped: `&amp;` `&lt;` `&gt;`

---

## Core Style Principles

1. **Colors encode meaning, not decoration.**
2. **Most of the canvas should remain neutral.**
3. **Accent colors should be sparse and semantic.**
4. **Strokes are always darker than fills.**
5. **Typography and spacing carry more hierarchy than color.**
6. **Prefer calm, editorial contrast over vivid UI contrast.**
7. **Default to no shadows and no gradients.**

### Non-goals

Avoid these unless the user explicitly requests them:
- saturated brand colors
- drop shadows
- glossy UI treatments
- gradients
- icon-heavy decoration
- dense legends
- more than 4 semantic accent colors in one diagram
- dashboard aesthetics

---

## Diagram-type defaults

### Workflow Diagram
- Background: canvas background
- Nodes: AI/LLM, Start/Trigger, End/Success, Decision as needed
- Connectors: primary flow + optional feedback loop
- Layout: clean linear or gently branching
- Preferred emphasis: flow clarity

### Architecture Diagram
- Background: canvas background with panel grouping
- Nodes: Primary/Neutral + Secondary / Context + Tertiary / Control
- Connectors: orthogonal
- Layout: grouped systems and interfaces
- Preferred emphasis: containment and interaction

### Comparison Diagram
- Background: two or more large neutral panels
- Nodes: soft semantic fills
- Connectors: minimal
- Layout: mirrored or side-by-side
- Preferred emphasis: contrast in structure, not color

### Chart Diagram
- Background: warm neutral
- Bars/series: use neutral + 1-2 restrained accents
- Gridlines: very light neutral stroke
- Preferred emphasis: direct value comparison, low clutter

### Editorial Poster Diagram
- Large title, sparse elements, oversized whitespace
- Minimal arrows
- Strong compositional balance
- One dominant idea only

---

## Pattern Library

A good diagram should not merely display information. It should make a claim obvious.

Choose a pattern based on the **argument** the diagram is making: Is this showing a sequence? A comparison? A system boundary? A loop? A branching decision? A layered architecture? A group overlap? A quantitative contrast?

The wrong pattern makes the content feel busy even when the styling is correct.

### Universal composition rules

These rules apply to all patterns.

1. **One dominant idea per diagram.** Secondary details should support that point, not compete with it.
2. **Reading order must be obvious.** The viewer should know where to start and where to look next within 3 seconds. Default: left-to-right for workflows/comparisons; top-to-bottom for stacks/hierarchies; outer-to-inner for containment.
3. **Keep hierarchy shallow.** Prefer at most: 1 title level, 1 section/group level, 1 node level, optional annotation level.
4. **Use whitespace as structure.** Spacing should separate groups before boxes do.
5. **Use containers sparingly.** Do not put every label in a box. Use free-floating text when the relationship is already clear.
6. **Reduce visual grammar before adding color.** First solve layout, grouping, and connector logic. Then add semantic color.

### Node content rules

- Keep labels short — prefer nouns or verb phrases
- Avoid full-sentence labels inside nodes
- Good: "Gather context", "Verify results", "Retrieved docs", "Policy layer", "Remote MCP server"
- Weak: "This stage is where the system gathers context from different sources"

Only include concrete evidence blocks (code snippet, file tree, API response, schema excerpt) when they meaningfully improve precision.

### Pattern 1: Linear Workflow

**Use when** the process is mostly sequential, each step leads to the next, and the key message is procedural flow.

Examples: prompt -> retrieve -> reason -> answer; gather context -> take action -> verify -> done

**Layout:**
- One row or one column of nodes, evenly spaced
- 3 to 7 major steps
- Keep the main path straight; avoid side branches unless critical
- Start node may use Start/Trigger; processing steps use AI/LLM or Primary/Neutral; end node may use End/Success
- Primary flow connectors only; use optional arrow labels only when they clarify transformation

**Anti-patterns:** too many side annotations; mixing architecture grouping into a simple flow; long paragraphs inside nodes

### Pattern 2: Feedback Loop Workflow

**Use when** the process involves retry, refinement, evaluation, or iteration — work loops until success criteria are met.

Examples: generate -> evaluate -> revise; agent loop with human steering

**Layout:**
- One main forward path with one loop-back connector
- Keep the main path straight; route the loop outside the main path
- Use curved or gently orthogonal return connectors
- Forward path = solid; retry loop = feedback connector; human interrupt = dashed warm-colored connector

**Anti-patterns:** multiple overlapping loopbacks; loop connector passing through nodes; showing every internal retry as a separate visible step

### Pattern 3: Branch Workflow / Decision Tree

**Use when** a choice meaningfully changes the path — approval, routing, or gating.

Examples: policy check -> allow / deny; task type -> choose specialized agent

**Layout:**
- One incoming path, one decision node, 2 to 4 outgoing branches
- Branches should fan out clearly; label branches directly near the divergence
- Rejoin later only if the paths truly converge
- Decision nodes should be visually distinct; do not use decision styling for generic steps
- Branch labels should be short: yes / no / low confidence / use tool

**Anti-patterns:** more than 4 branches from one decision; unlabeled branch semantics; arbitrary diamonds everywhere

### Pattern 4: Parallel Fan-out / Fan-in

**Use when** one task splits into multiple concurrent workers and results are aggregated.

Examples: planner -> multiple subagents -> synthesizer; dispatcher -> workers -> reducer

**Layout:**
- One source node, 2 to 5 parallel branches, one merge/synthesis node
- Distribute branches evenly; keep branch node sizes consistent
- Align parallel workers on one row or column
- Worker nodes often use AI/LLM or Secondary / Context; merge node often uses Tertiary / Control

**Anti-patterns:** uneven branch spacing; too many workers; converging lines crossing each other

### Pattern 5: Split Comparison

**Use when** the point is contrast between two states or approaches — before/after, without/with AI.

Examples: before AI vs with AI; prompt engineering vs context engineering

**Layout:**
- Two or three large side-by-side panels
- Mirrored or comparable composition; panels should have equal visual weight
- Use repeated labels where comparison benefits from symmetry
- Keep color differences restrained; use structure, not saturation, to show difference
- Minimal connectors across panels
- Title should state the comparison claim, not just the categories

**Anti-patterns:** unrelated layouts in each panel; overusing arrows between panels; too much text explaining what the eye should already see

### Pattern 6: Grouped Architecture

**Use when** the main point is system boundaries, modules, resources, or containment — components belong to subsystems more than they happen in time.

Examples: agent config + virtual machine; application + tools + storage + runtime

**Layout:**
- Large outer groups or panels with smaller nodes inside
- Place grouped systems first, internal nodes second, connect after layout is stable
- Connectors between components or between groups; use orthogonal connectors
- Use containers for ownership/environment; softer grouping for conceptual regions

**Anti-patterns:** excessive nesting; using workflow arrows when simple adjacency is enough; putting every sentence inside a container

### Pattern 7: Layered Stack

**Use when** the diagram explains hierarchy, dependency, or abstraction layers — the order is vertical, not procedural.

Examples: model layer / orchestration layer / tool layer; UI / services / storage

**Layout:**
- Stacked horizontal bands or cards
- Top-to-bottom hierarchy; keep layers horizontally aligned
- Optional small dependencies between layers
- Do not overconnect adjacent layers unless needed

**Anti-patterns:** turning a stack into a workflow; too many internal arrows; inconsistent widths without semantic reason

### Pattern 8: Swimlane Sequence

**Use when** multiple actors or systems interact over time and timing/order matters across participants.

Examples: user / agent / tool / verifier; browser / model / sandbox / file system

**Layout:**
- Vertical or horizontal lanes for actors; 2 to 5 lanes only
- Time direction is consistent; events inside lanes align to a timeline
- Lane titles should be visually quiet but clear
- Messages cross lanes; message lines should be short and readable

**When to avoid:** when the true message is system architecture, not interaction order; when a simple workflow would suffice.

**Anti-patterns:** too many lanes; too many tiny message steps; unaligned timing rows

### Pattern 9: Hub-and-Spoke

**Use when** one central entity connects to several supporting concepts — central coordination, not sequence.

Examples: a central model using tools; one orchestrator with multiple capabilities

**Layout:**
- One central dominant node with 3 to 6 surrounding nodes
- Surrounding nodes balanced around center; preserve symmetry
- Minimal connector complexity

**Anti-patterns:** using hub-and-spoke for actual sequential workflows; too many spokes; unequal visual weight making the composition drift

### Pattern 10: Venn / Overlap

**Use when** the message is shared responsibility, convergence, or blended ownership — overlap itself is the idea.

Examples: product / design / engineering overlap; overlapping capability zones

**Layout:**
- 2 or 3 overlapping soft shapes; keep shapes large and simple
- Labels placed in or near each region
- Center overlap may carry emphasis if meaningful
- Use transparency lightly; avoid dense annotations

**Anti-patterns:** more than 3 overlapping groups; forcing workflow semantics into overlap shapes; too much text in intersections

### Pattern 11: Editorial Chart

**Use when** the main claim is numerical comparison and precise values matter less than the contrast/trend.

Allowed chart types: bar chart, column chart, simple grouped bar chart, very restrained line chart.

**Rules:**
- No dashboard styling, no saturated palettes
- No heavy legends when direct labels are possible
- Faint gridlines; 2 to 4 series max
- Warm neutral background; bars in neutral + 1-2 semantic accents
- Large title; direct value labels where helpful

**Anti-patterns:** pie charts; rainbow series; dense axes and ticks; multi-metric chart junk

### Pattern 12: Callout Annotation

**Use when** one supporting fact or note clarifies a main diagram — helpful but not central to the structure.

Examples: "contents of skill directories live in the file system"; "policy layer enforces sandbox restrictions"

**Layout:**
- Small callout box outside the main flow
- One connector pointing to the referenced region; minimal text
- Keep callouts outside major flow regions; 1 to 3 callouts per diagram is usually enough

**Anti-patterns:** too many callouts; callouts becoming the dominant content; crossing callout connectors over main structure

### Pattern combination rules

Patterns can be combined, but only when one pattern is clearly primary.

**Good combinations:**
- Grouped Architecture + Callout Annotation
- Linear Workflow + Feedback Loop
- Split Comparison + Simple Workflow
- Parallel Fan-out / Fan-in + Grouped Architecture
- Swimlane Sequence + Callout Annotation

**Risky combinations:**
- Split Comparison + Swimlane + Architecture
- Venn + Workflow
- Layered Stack + Branch Tree + Fan-out all together

**Rule:** If combining patterns increases cognitive load more than it increases explanatory value, split into two diagrams instead.

### Density guidelines

| Level | Nodes | Patterns | Callouts |
|-------|-------|----------|----------|
| Simple | 3-5 | 1 primary | almost none |
| Medium | 6-12 | 1 primary + 1 secondary | 1-3 |
| Dense | 12-20 | only when genuinely required | strong grouping required |

A single editorial diagram should usually avoid:
- more than 20 visible nodes
- more than 4 semantic colors
- more than 3 nested group levels
- more than 2 visible loopbacks
- more than 5 parallel branches

### Pattern fallback rules

When the structure is ambiguous:
1. Default to **Linear Workflow** for process-like user input
2. Default to **Grouped Architecture** for system/component descriptions
3. Default to **Split Comparison** for before/after content
4. Default to **Callout Annotation** rather than adding extra nodes
5. Split into multiple diagrams rather than forcing a hybrid

---

## Visual review checklist

Before finalizing, verify:

**Argument:**
- Can the main point be understood in one sentence?
- Does the chosen pattern support that point?

**Reading order:**
- Is the start obvious?
- Is the main path visually dominant?
- Are groups clear before the viewer reads labels?

**Density:**
- Are there too many nodes?
- Are there too many accents?
- Can any labels become free text instead of boxes?

**Connectors:**
- Any crossings that can be removed?
- Any loopbacks that can be simplified?
- Are dashed lines used sparingly?

**Balance:**
- Does the composition feel centered and stable?
- Is whitespace distributed intentionally?
- Is there one focal point rather than many?
