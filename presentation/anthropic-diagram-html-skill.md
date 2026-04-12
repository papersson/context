---
name: anthropic-diagram-html
description: Generate editorial-style diagrams in the Anthropic blog visual style as self-contained HTML+SVG files. Use whenever the user wants a diagram, flowchart, architecture diagram, comparison chart, or any visual that should look like Anthropic's blog illustrations — rendered in a browser, not drawio. Trigger on prompts like "draw a diagram", "create a flowchart", "visualize this process", even when the user doesn't say "Anthropic style" explicitly.
---

# Anthropic-Style HTML Diagram Skill

Generate a single `.html` file that renders an editorial-style diagram using inline SVG. Output should be self-contained (no external CSS/JS), look calm and publication-quality, and open directly in a browser.

## Workflow

```
User text → DiagramSpec (planning) → HTML+SVG → .html file
```

1. **Analyze the request**: Identify the main claim, pick a pattern (linear workflow, feedback loop, split comparison, grouped architecture, layered stack, swimlane, hub-and-spoke, parallel fan-out, Venn, editorial chart). Reading direction is left-to-right for workflows, top-to-bottom for stacks.
2. **Write the DiagramSpec** as text first — nodes, semantic types, groups, connections — so structural mistakes surface before committing to markup.
3. **Generate the HTML+SVG** using the scaffolding below.
4. **Write to file** with a descriptive name (`kebab-case.html`) in the working directory and `open` it.

Language consistency: node labels, titles, and edge labels must match the user's input language.

## Scaffold

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>{{Diagram Title}}</title>
<style>
  html, body { margin: 0; padding: 0; background: #F2EFE8;
               font-family: -apple-system, "Helvetica Neue", Helvetica, Arial, sans-serif; }
  .wrap { display: flex; justify-content: center; padding: 40px 20px; }
  svg   { width: 100%; max-width: 1200px; height: auto; }

  .title        { font-size: 32px; fill: #1F1F1C; font-weight: 700; text-anchor: middle; }
  .panel-label  { font-size: 15px; fill: #5F5A54; font-weight: 500; }
  .node-title   { font-size: 16px; fill: #2D2B28; font-weight: 500; text-anchor: middle; }
  .node-sub     { font-size: 13px; fill: #5F5A54; text-anchor: middle; }
  .edge-label   { font-size: 12px; fill: #5F5A54; text-anchor: middle; }
  .edge-label-bg{ fill: #F2EFE8; stroke: none; }
</style>
</head>
<body>
<div class="wrap">
<svg viewBox="0 0 1200 700" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="10" markerHeight="10" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10" fill="none" stroke="#7A756E" stroke-width="1.6"/>
    </marker>
    <marker id="arrow-dashed" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="10" markerHeight="10" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10" fill="none" stroke="#9A948C" stroke-width="1.6"/>
    </marker>
  </defs>
  <!-- panels, nodes, edges here -->
</svg>
</div>
</body>
</html>
```

Canvas background is always `#F2EFE8` (warm) except in very sparse diagrams where `#FFFFFF` is acceptable.

## Semantic node colors

Color encodes meaning, not decoration. Pick the fill/stroke by role.

| Semantic type        | Fill      | Stroke    | Use for                                                     |
|----------------------|-----------|-----------|-------------------------------------------------------------|
| Primary / Neutral    | `#E6E2DA` | `#8C867F` | Default blocks, generic system components                   |
| Secondary / Context  | `#EAF4FB` | `#6FA8D6` | Files, tools, docs, retrieved context, storage              |
| Tertiary / Control   | `#EEEAF9` | `#9A90D6` | Routers, memory, orchestration, events, loops               |
| Start / Trigger      | `#F8E9E1` | `#D88966` | User input, prompt, external trigger (peach)                |
| End / Success        | `#CFE8D7` | `#71AE88` | Verified result, accepted answer                            |
| AI / LLM / Active    | `#D7E6DC` | `#7FB08F` | Model calls, agent workers, active execution stages         |
| Warning / Reset      | `#F3E4DA` | `#C88E6A` | Retry, caution, manual steering                             |
| Decision (diamond)   | `#E6D7B4` | `#BFA777` | Branch gates, yes/no                                        |
| Error / Output file  | `#F8DFDA` | `#D96B63` | Failures — or coral-style outputs like generated artifacts  |
| Inactive             | `#EFECE6` | `#B4AEA6` | De-emphasized, passive-read components                      |

`stroke-width: 1.8` for node borders. `rx="10"` for standard nodes. Use `rx="999"` (or height/2) for pills.

## Panels (containers)

Group boundaries are **dashed, not filled** — Anthropic editorial diagrams use empty dashed rectangles with a small label in the top-left. This is the single strongest "Anthropic" cue.

```html
<rect x="40" y="40" width="640" height="620" rx="14" ry="14"
      fill="none" stroke="#B9B3AB" stroke-width="1.5" stroke-dasharray="6 6"/>
<text x="70" y="70" class="panel-label">Panel Title</text>
```

Keep nesting flat — max 2 levels.

## Connectors

All arrows use **open chevron arrowheads** via the `<marker>` defs in the scaffold. Never filled triangles. This is the second-strongest "Anthropic" cue.

| Connector       | Stroke    | Dash        | Meaning                                   |
|-----------------|-----------|-------------|-------------------------------------------|
| Primary flow    | `#7A756E` | solid       | Main path                                 |
| Optional / weak | `#9A948C` | `6 6`       | Inferred, optional                        |
| Feedback loop   | `#9A948C` | `6 6`       | Return path, retry, "records outcomes"    |
| Human override  | `#D88966` | `6 6`       | Manual intervention                       |
| Error path      | `#D96B63` | solid       | Failure flow                              |

`stroke-width: 1.8` for solid, `1.6` for dashed. Use `<path d="M x y L x y L x y">` for orthogonal routing — always right-angle bends, never diagonals.

## Layout hygiene (the rules that matter)

These are the rules a blind SVG author gets wrong. Follow them mechanically.

1. **One edge, one channel.** Never run two arrows on the same centerline. If A→B goes down at x=500 and B→A goes up, put the return arrow in a separate lane (e.g., x=440). Parallel shared-centerline arrows read as noise.
2. **Cross-panel arrows route in the gap between panels.** Compute the midpoint of the empty space between two panels. The vertical leg of any cross-panel edge belongs there, not inside either panel.
3. **Minimum 80px horizontal and 60px vertical gap** between adjacent node edges. Minimum 160px gap between panels when cross-panel routing is needed.
4. **Generous canvas.** Default viewBox is 1200×700+. Better to have empty space at the bottom than crammed nodes.
5. **Snap coordinates to multiples of 10.** Makes positions predictable and alignment automatic.
6. **Nodes: 180–240 px wide, 70–90 px tall.** Keep sizes consistent within a diagram.

## Label placement (the hard part)

Edge labels regularly collide with lines. Two rules prevent this:

1. **Always wrap edge labels in a canvas-colored backing rect.** The rect visually breaks the line under the label so the arrow appears to terminate at the label edge. Without this, lines graze glyphs and the diagram looks sloppy.

   ```html
   <rect class="edge-label-bg" x="..." y="..." width="..." height="18" rx="3"/>
   <text x="..." y="..." class="edge-label">records outcomes</text>
   ```

   Estimate rect width as `char_count × 6.6 + 16` at font-size 12. Give ~18px height for single-line, ~36px for two-line.

2. **Center the label on the edge, not above it.** For a horizontal edge at `y=450`:
   - rect: `y=441, height=18` (straddles the line)
   - text: `y=454` (baseline; visual center lands on y=450)

   For a vertical edge, rotate the same logic around x.

Labels floating *next to* an edge look like afterthoughts. Labels *centered on* an edge with a backdrop look intentional.

## Typography

| Level              | Size  | Color      |
|--------------------|-------|------------|
| Diagram title      | 28–36 | `#1F1F1C`  |
| Panel label        | 14–16 | `#5F5A54`  |
| Node title         | 16    | `#2D2B28`  |
| Node subtitle      | 13    | `#5F5A54`  |
| Edge label         | 12    | `#5F5A54`  |

Prefer short noun/verb-phrase labels. Never full sentences inside nodes.

## Core style principles

1. Colors encode meaning, not decoration.
2. Most of the canvas stays neutral; accent colors are sparse.
3. Strokes are always darker than their fill.
4. No shadows, no gradients, no glossy UI treatments.
5. Spacing carries more hierarchy than color.
6. Never more than 4 semantic accent colors per diagram.

## Quality checklist

Before writing the file, verify mentally:

- [ ] Canvas background `#F2EFE8` set on body
- [ ] All panel borders dashed, unfilled
- [ ] All arrows use open chevron markers
- [ ] No two arrows share the same centerline
- [ ] Cross-panel arrows route in the inter-panel gap
- [ ] Every edge label has a canvas-colored backing rect
- [ ] Edge labels are centered *on* their edge, not above
- [ ] Node colors reflect semantic role (not aesthetic choice)
- [ ] ≤ 4 semantic accent colors total
- [ ] Coordinates snapped to multiples of 10

## Known limitation

Static SVG authored without a rendering loop can miss fine collisions (label bounding boxes grazing lines, arrow endpoints clipping through node borders). If the diagram is non-trivial, expect one or two iterations after the user sees it rendered — treat the first output as a draft.
