---
name: tui-diagram-html
description: Generate terminal/TUI-style diagrams as self-contained HTML+SVG files. Dark navy canvas, monospace type, outline-only boxes in light cyan, dashed panel containers with top-left inset labels, small filled-triangle arrowheads. Use when the user wants a diagram, flowchart, architecture diagram, or visual that should look like a polished terminal UI sketch (Charm/Bubble Tea, Lazygit, k9s, ASCII-art-as-SVG aesthetic). Trigger on prompts like "draw a TUI diagram", "make it look like a terminal", "ASCII-style flowchart", or any diagram request where the user has cited a TUI/terminal reference.
---

# Terminal / TUI Style HTML Diagram Skill

Generate a single `.html` file that renders a diagram in a terminal/TUI aesthetic using inline SVG. The output should look like a polished sketch from a modern TUI (Charm, k9s, Lazygit) — dark canvas, monospace, outline-only geometry, no fills, no shadows, no gradients.

## Workflow

```
User text → DiagramSpec (planning) → HTML+SVG → .html file
```

1. **Analyze the request**: Identify the main claim, pick a pattern (linear workflow, decision branch, feedback loop, fan-out, vertical pipeline). Reading direction is left-to-right for workflows, top-to-bottom for pipelines.
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
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{Diagram Title}}</title>
<style>
  html, body { margin: 0; padding: 0; background: #0D1B2E;
               font-family: "JetBrains Mono", "SF Mono", "Fira Code",
                            ui-monospace, Menlo, Consolas, monospace; }
  body  { min-height: 100vh; display: flex; align-items: center; justify-content: center; }
  .wrap { width: 100%; display: flex; justify-content: center;
          padding: clamp(12px, 3vw, 40px) clamp(8px, 2vw, 24px); box-sizing: border-box; }
  svg   { width: 100%; max-width: min(100%, 1600px); height: auto; }

  .title        { font-size: 22px; fill: #E8F1F8; font-weight: 400; text-anchor: middle;
                  letter-spacing: 0.02em; }
  .panel-label  { font-size: 14px; fill: #6B7D8F; font-weight: 400; }
  .panel-label-bg { fill: #0D1B2E; }
  .node-text    { font-size: 16px; fill: #E8F1F8; font-weight: 400; text-anchor: middle;
                  dominant-baseline: middle; letter-spacing: 0.01em; }
  .node-sub     { font-size: 12px; fill: #8AA0B5; text-anchor: middle;
                  dominant-baseline: middle; }
  .edge-label   { font-size: 14px; fill: #A8D5E8; text-anchor: middle;
                  dominant-baseline: middle; }
  .edge-label-bg{ fill: #0D1B2E; stroke: none; }
</style>
</head>
<body>
<div class="wrap">
<svg viewBox="0 0 1400 800" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <!-- Filled triangle, small. Match arrow color via context-stroke or fill. -->
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 Z" fill="#A8D5E8"/>
    </marker>
    <marker id="arrow-emph" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 Z" fill="#C4E3F1"/>
    </marker>
  </defs>
  <!-- panels, nodes, edges here -->
</svg>
</div>
</body>
</html>
```

Canvas background is always `#0D1B2E` (deep navy). Never lighter — the TUI cue depends on the dark surface.

## Color palette (terminal-honest)

The whole diagram lives within a tiny, monochromatic-ish palette. Color encodes role; saturation is low.

| Token              | Hex        | Use                                                       |
|--------------------|------------|-----------------------------------------------------------|
| Canvas             | `#0D1B2E`  | Background only. Also fills label backings.               |
| Stroke / Default   | `#A8D5E8`  | Box borders, arrows, primary lines                        |
| Stroke / Emphasis  | `#C4E3F1`  | Brighter outline for highlighted node or critical path    |
| Stroke / Muted     | `#5A6E82`  | De-emphasized boxes, inactive paths                       |
| Panel border       | `#3F5468`  | Dashed group containers                                   |
| Panel label        | `#6B7D8F`  | Top-left inset label inside panel border                  |
| Text / Primary     | `#E8F1F8`  | Node titles                                               |
| Text / Muted       | `#8AA0B5`  | Subtitles, secondary annotations                          |
| Edge label text    | `#A8D5E8`  | "pass", "approved", "billing" — matches stroke            |
| Accent / Warn      | `#F0B870`  | Sparingly: retry loops, manual override, error path       |

Use no more than **two** stroke colors in a diagram (default + at most one accent). The TUI aesthetic breaks immediately if the canvas turns festive.

## Nodes

**Default node:** outline only, no fill, rounded corners.

```html
<rect x="80" y="240" width="180" height="70" rx="8" ry="8"
      fill="none" stroke="#A8D5E8" stroke-width="1.6"/>
<text x="170" y="275" class="node-text">Spec</text>
```

- `stroke-width: 1.6` — TUI lines are thin and crisp. Avoid heavier weights; they read as bold/pixel-doubled.
- `rx="8"` standard. Use `rx="0"` for an explicitly boxy/CLI feel. Avoid pill shapes — they break the orthogonal grid.
- Width 160–240 px, height 60–80 px. Keep heights uniform within a row so arrows align cleanly.
- Text vertically centered via `dominant-baseline: middle` and the `y` set to the rect's vertical midpoint.

**Decision node (the "Ready?" hex shape):** use a notched octagon/hex to mark a branch gate. This is a strong TUI affordance — recognizable from Mermaid's terminal renderer.

```html
<!-- Notched-rectangle decision shape, ~180×80, centered at (cx,cy) -->
<path d="M cx-80,cy-30
         L cx-70,cy-40 L cx+70,cy-40 L cx+80,cy-30
         L cx+80,cy+30 L cx+70,cy+40 L cx-70,cy+40 L cx-80,cy+30 Z"
      fill="none" stroke="#A8D5E8" stroke-width="1.6"/>
<text x="cx" y="cy" class="node-text">Ready?</text>
```

(Substitute real coordinates for `cx,cy`.)

**Database / store cylinder:** keep simple — a rectangle with the label is fine in TUI mode. Don't draw 3D cylinders; they don't fit the aesthetic.

## Panels (containers) — fieldset/legend pattern

The single strongest TUI cue is the **dashed panel with a label that breaks the top border**, exactly like an HTML `<fieldset>` with a `<legend>`. The label text sits *over* a small canvas-colored backing rect that hides the segment of dashed border behind it.

```html
<!-- Panel rectangle -->
<rect x="320" y="180" width="780" height="440" rx="6" ry="6"
      fill="none" stroke="#3F5468" stroke-width="1.4" stroke-dasharray="4 4"/>

<!-- Label backing (hides border segment) -->
<rect class="panel-label-bg" x="350" y="172" width="120" height="16"/>

<!-- Label text -->
<text x="360" y="185" class="panel-label">Build Plan</text>
```

Sizing the label backing: `width = char_count × 8.4 + 16` (panel-label is monospace 14px). Position it so the text baseline at `y = panel_y - (label_font_size / 2 + 2)` reads as straddling the top border.

Keep nesting flat — at most 2 levels.

## Connectors

All arrows are **thin, light-cyan, with small filled-triangle arrowheads** via the `<marker>` defs. Routing is strictly orthogonal (right-angle bends only).

| Connector       | Stroke    | Width | Dash      | Meaning                          |
|-----------------|-----------|-------|-----------|----------------------------------|
| Primary flow    | `#A8D5E8` | 1.6   | solid     | Main path                        |
| Branch path     | `#A8D5E8` | 1.6   | solid     | Decision outcomes (with label)   |
| Feedback / loop | `#A8D5E8` | 1.6   | solid     | Return path (route around)       |
| Inactive        | `#5A6E82` | 1.4   | `4 4`     | De-emphasized                    |
| Accent / retry  | `#F0B870` | 1.6   | solid     | Manual / warn — use sparingly    |

Dashed lines are rare in TUI diagrams — most paths are solid even when they represent feedback or retry. Differentiate via routing (loop arrows curve back through empty channels) instead of stroke style.

`<path d="M x y L x y L x y" marker-end="url(#arrow)">` for orthogonal routing — never diagonals, never curves. The terminal grid forbids both.

## Edge labels — straddle the line, no rect needed

Unlike the editorial style, TUI edge labels work best when:

- Placed **above and beside** the segment (not centered on it), in the same cyan as the line
- Followed by a short arrow segment, e.g. `pass →` reads as a label with its own tail directing into the next node
- Backed by a canvas-colored rect **only when the label sits directly on a line** (rare)

Two patterns:

**Pattern A — label-then-arrow (most common):**
```
─────  pass  ─────►  [Stage]
```
Implement as a horizontal line that visually breaks for the label, then resumes into the target node:
```html
<line x1="700" y1="320" x2="780" y2="320" stroke="#A8D5E8" stroke-width="1.6"/>
<text x="820" y="324" class="edge-label">pass</text>
<line x1="860" y1="320" x2="940" y2="320" stroke="#A8D5E8" stroke-width="1.6"
      marker-end="url(#arrow)"/>
```

**Pattern B — label on continuous line (use a backing rect):**
```html
<line ... marker-end="url(#arrow)"/>
<rect class="edge-label-bg" x="..." y="..." width="..." height="18"/>
<text class="edge-label" ...>approved</text>
```

## Layout hygiene

These rules matter more in TUI than in editorial style — the monospace grid amplifies misalignment.

1. **Snap coordinates to multiples of 10.** No exceptions.
2. **One edge, one channel.** Never run two arrows on the same centerline.
3. **Cross-panel arrows route in the gap between panels.** Compute the midpoint of the empty space; the vertical leg belongs there.
4. **Minimum 100px horizontal and 80px vertical gap** between adjacent nodes. TUI diagrams breathe — they are not dense.
5. **Generous canvas.** Default viewBox 1400×800. Better to have empty space at the bottom than crammed nodes.
6. **Match viewBox to content aspect ratio** — don't pad a 1400×500 diagram to 1400×800.
7. **Align node baselines.** All nodes in a row share the same `y`. All nodes in a column share the same `x`. Inspect by eye: any node off the grid breaks the TUI illusion.
8. **Arrowhead clearance.** Stop the line 6–10 px short of the target node's border so the chevron doesn't bite into the stroke.

## Typography

| Level         | Size  | Color      | Weight |
|---------------|-------|------------|--------|
| Diagram title | 20–24 | `#E8F1F8`  | 400    |
| Panel label   | 13–14 | `#6B7D8F`  | 400    |
| Node text     | 16    | `#E8F1F8`  | 400    |
| Node subtitle | 12    | `#8AA0B5`  | 400    |
| Edge label    | 13–14 | `#A8D5E8`  | 400    |

All text is monospace. Never bold. Never italic. Letter-spacing 0.01–0.02em adds the slight rendered-terminal feel; more than that looks affected.

Prefer short noun/verb-phrase labels. One or two words per node. Never sentences.

## Responsiveness

The scaffold is responsive: `<meta name="viewport">` enables mobile rendering, `viewBox` + default `preserveAspectRatio="xMidYMid meet"` scales uniformly, fluid `clamp()` padding tightens edge gutters. Fonts scale with the diagram automatically — do not set CSS font-size media queries on SVG text.

Do **not** reflow node positions with CSS media queries.
Do **not** set fixed pixel `width`/`height` on `<svg>`.
For phone-friendly diagrams, pick a squarer `viewBox` at authoring time (e.g. 1000×900).

## Core style principles

1. Outline only, never fill. Empty boxes are the TUI signature.
2. One stroke color does most of the work; reserve a second only for true accent.
3. Strict orthogonal routing — terminal grids have no diagonals.
4. Thin lines (1.4–1.6 px). Heavy strokes read as a different aesthetic.
5. Spacing carries hierarchy. Empty cells in the grid are part of the design.
6. Monospace everything. Including the title.
7. No shadows, no gradients, no glows, no rounded-pill chips, no badges.

## Quality checklist

Before writing the file, verify mentally:

- [ ] Canvas background `#0D1B2E` set on body
- [ ] `<meta name="viewport" content="width=device-width, initial-scale=1">` present
- [ ] `<svg>` has no fixed pixel `width`/`height` — uses `viewBox` + CSS
- [ ] `viewBox` aspect ratio hugs the actual content bounds
- [ ] All fonts are monospace (no fallback to sans-serif)
- [ ] All nodes are stroke-only (`fill="none"`)
- [ ] All panel borders are dashed and unfilled, with a label that breaks the top border via a canvas-colored backing rect
- [ ] All arrows use small filled-triangle markers, oriented `auto-start-reverse`
- [ ] All paths are orthogonal — no diagonals, no curves
- [ ] No two arrows share the same centerline
- [ ] Cross-panel arrows route in the inter-panel gap
- [ ] Coordinates snapped to multiples of 10
- [ ] Nodes in a row share `y`; nodes in a column share `x`
- [ ] At most 2 stroke colors used
- [ ] Decision shapes use the notched-rectangle (hex) silhouette

## Known limitation

Static SVG authored without a rendering loop can miss fine alignment issues (label backings off by a pixel, arrow endpoints clipping into a node border). For non-trivial diagrams, expect one iteration after the user sees it rendered — treat the first output as a draft.
