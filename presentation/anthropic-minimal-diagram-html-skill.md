---
name: anthropic-minimal-diagram-html
description: Generate minimal, editorially clean diagrams as self-contained HTML+SVG files. Anthropic's warm palette, but stripped to outlines (no fills) and structured with strong dashed parent containers in the TUI fieldset/legend tradition. Use when the user wants a calm, publication-quality diagram that emphasizes hierarchy through grouping and spacing rather than color blocks. Trigger on prompts like "draw a clean diagram", "minimal flowchart", "Anthropic-style but cleaner", "with proper grouping".
---

# Anthropic-Minimal HTML Diagram Skill

Generate a single `.html` file that renders a quiet, editorial diagram using inline SVG. This style keeps Anthropic's warm canvas and accent palette but strips node fills, leans on dashed parent containers for hierarchy, and routes arrows orthogonally on a strict grid. Output is self-contained, calm, and publication-grade.

## When to pick this style over `anthropic-diagram-html`

Pick this minimal variant when:
- Hierarchy and grouping matter more than per-node semantic color
- The diagram has 3+ logical regions that benefit from visible containers
- A clean, "specification document" feel is wanted over a colorful editorial illustration
- The reader will scan for structure first, semantics second

Pick the original `anthropic-diagram-html` when nodes need to read as differentiated artifacts (input vs. tool vs. model vs. output) and color is doing real semantic work.

## Workflow

```
User text → DiagramSpec (planning) → HTML+SVG → .html file
```

1. **Identify the regions first.** Before placing nodes, decide which dashed parent containers will hold them. Groups are the load-bearing structure; nodes are tenants.
2. **Write the DiagramSpec** as text — regions, their children, and the cross-region edges.
3. **Generate the HTML+SVG** using the scaffolding below.
4. **Write to file** with a descriptive name (`kebab-case.html`) in the working directory and `open` it.

Language consistency: all labels match the user's input language.

## Scaffold

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{Diagram Title}}</title>
<style>
  html, body { margin: 0; padding: 0; background: #F2EFE8;
               font-family: -apple-system, "Helvetica Neue", Helvetica, Arial, sans-serif; }
  body  { min-height: 100vh; display: flex; align-items: center; justify-content: center; }
  .wrap { width: 100%; display: flex; justify-content: center;
          padding: clamp(12px, 3vw, 40px) clamp(8px, 2vw, 24px); box-sizing: border-box; }
  svg   { width: 100%; max-width: min(100%, 1600px); height: auto; }

  .title          { font-size: 28px; fill: #1F1F1C; font-weight: 600; text-anchor: middle;
                    letter-spacing: -0.005em; }
  .panel-label    { font-size: 14px; fill: #7A756E; font-weight: 500;
                    letter-spacing: 0.02em; text-transform: uppercase; }
  .panel-label-bg { fill: #F2EFE8; }
  .node-text      { font-size: 16px; fill: #2D2B28; font-weight: 500; text-anchor: middle;
                    dominant-baseline: middle; }
  .node-sub       { font-size: 12px; fill: #7A756E; text-anchor: middle;
                    dominant-baseline: middle; }
  .edge-label     { font-size: 13px; fill: #5F5A54; text-anchor: middle;
                    dominant-baseline: middle; }
  .edge-label-bg  { fill: #F2EFE8; stroke: none; }
</style>
</head>
<body>
<div class="wrap">
<svg viewBox="0 0 1400 800" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 Z" fill="#7A756E"/>
    </marker>
    <marker id="arrow-accent" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 Z" fill="#C88E6A"/>
    </marker>
  </defs>
  <!-- panels first (they live behind), then edges, then nodes -->
</svg>
</div>
</body>
</html>
```

Canvas background is always `#F2EFE8`.

## Color palette (warm, restrained)

| Token              | Hex        | Use                                                       |
|--------------------|------------|-----------------------------------------------------------|
| Canvas             | `#F2EFE8`  | Background; also fills label backings                     |
| Stroke / Default   | `#5F5A54`  | Node borders, primary arrows                              |
| Stroke / Muted     | `#A09A91`  | Inactive / de-emphasized                                  |
| Panel border       | `#B9B3AB`  | Dashed group containers                                   |
| Panel label        | `#7A756E`  | Top-left inset label                                      |
| Text / Primary     | `#2D2B28`  | Node titles                                               |
| Text / Muted       | `#7A756E`  | Subtitles, secondary annotations                          |
| Edge label text    | `#5F5A54`  | Connector annotations                                     |
| Accent / Warn      | `#C88E6A`  | Manual override, retry, exception path                    |
| Accent / Success   | `#71AE88`  | Verified result, accepted path                            |
| Accent / Error     | `#D96B63`  | Failure flow                                              |

Use **at most one accent color** per diagram beyond the default stroke. The minimalism breaks if more than one role is being signaled by hue.

## Nodes

**Default node:** outline only, no fill. The absence of fill is the entire point — color blocks would compete with the dashed parent containers for visual weight.

```html
<rect x="80" y="240" width="200" height="72" rx="8" ry="8"
      fill="none" stroke="#5F5A54" stroke-width="1.6"/>
<text x="180" y="276" class="node-text">Spec</text>
```

- `stroke-width: 1.6` for default; `1.8` only for a single emphasized "primary" node per diagram.
- `rx="8"` standard. Avoid pills (`rx="999"`) — they break the orthogonal feel and clash with the panel rectangles.
- Width 180–240 px, height 64–80 px. Keep widths uniform within a row for clean column alignment.
- Vertically center text via `dominant-baseline: middle`.

**Decision node:** notched-rectangle (hex) silhouette, outline only.

```html
<path d="M cx-90,cy-30
         L cx-78,cy-42 L cx+78,cy-42 L cx+90,cy-30
         L cx+90,cy+30 L cx+78,cy+42 L cx-78,cy+42 L cx-90,cy+30 Z"
      fill="none" stroke="#5F5A54" stroke-width="1.6"/>
<text x="cx" y="cy" class="node-text">Ready?</text>
```

**Storage / file node:** keep it a plain rectangle. Don't draw cylinders — they read as 90s clip-art against this restraint.

**Emphasized node** (single per diagram): `stroke-width: 2.2` and either default or accent stroke color. No fill change.

## Panels (containers) — load-bearing

This is where this style departs from the original Anthropic skill. Panels are not decorative; they carry hierarchy. Use them whenever the diagram has 2+ logical regions.

**Fieldset/legend pattern** — the panel label sits over the top border, with a canvas-colored rect hiding the dashed segment behind it:

```html
<!-- Panel rectangle -->
<rect x="320" y="180" width="780" height="440" rx="10" ry="10"
      fill="none" stroke="#B9B3AB" stroke-width="1.4" stroke-dasharray="6 6"/>

<!-- Label backing rect (hides the dashed segment) -->
<rect class="panel-label-bg" x="350" y="170" width="120" height="18"/>

<!-- Panel label -->
<text x="360" y="184" class="panel-label">Build Plan</text>
```

Sizing the label backing: `width = char_count × 7.6 + 16` at panel-label 14px (sans-serif, slightly tighter than monospace). Position so the text vertically straddles the top border line.

**Grouping rules:**
- Every node belongs to a group, even if some groups are implicit (a single trivial node sitting outside any panel is fine — but if you have ≥3 nodes that share a phase, give them a panel).
- Panels can sit side-by-side or stacked. Avoid overlapping panels.
- Nesting maxes at 2 levels. Beyond that, refactor the diagram.
- Inter-panel padding: minimum 160 px horizontal or 100 px vertical so cross-panel arrows have a clean channel.
- Intra-panel padding: 40 px from panel border to nearest node edge.

## Connectors

Strictly orthogonal. Diagonal lines and curves are forbidden in this style — the clean grid is the aesthetic.

| Connector       | Stroke    | Width | Dash      | Meaning                          |
|-----------------|-----------|-------|-----------|----------------------------------|
| Primary flow    | `#5F5A54` | 1.6   | solid     | Main path                        |
| Branch path     | `#5F5A54` | 1.6   | solid     | Decision outcomes (with label)   |
| Feedback / loop | `#A09A91` | 1.4   | `5 5`     | Return path, retry               |
| Manual / warn   | `#C88E6A` | 1.6   | solid     | Human override, exception        |
| Error path      | `#D96B63` | 1.6   | solid     | Failure flow                     |

`<path d="M x y L x y L x y" marker-end="url(#arrow)">` for orthogonal routing. Stop the path 8–10 px short of the target border so the marker tip lands cleanly without overlapping the node stroke.

## Edge labels

Two acceptable patterns. Pick one per diagram and stay consistent.

**Pattern A — centered on the line with a canvas-colored backing rect.** Best when arrows are short.

```html
<rect class="edge-label-bg" x="..." y="..." width="..." height="18" rx="2"/>
<text x="..." y="..." class="edge-label">approved</text>
```

Estimate rect width as `char_count × 6.8 + 16` at 13px sans-serif.

**Pattern B — label-then-arrow inline.** Best when several branches fan out to right (like the TUI screenshots).

```
─────  pass  ─────►  [Stage]
```

Render as: short line → label text (no rect needed if the label is between two line segments rather than over one) → short line ending in the arrow marker.

## Layout hygiene

1. **Plan panels first, then nodes, then edges.** This order prevents geometry surprises late in the layout.
2. **Snap coordinates to multiples of 10.**
3. **One edge, one channel.** Never two arrows on the same centerline.
4. **Cross-panel arrows route in the gap between panels.** Vertical legs belong in inter-panel space, never inside a panel they don't belong to.
5. **Minimum 80 px horizontal, 60 px vertical gap** between adjacent nodes within a panel.
6. **Generous canvas.** Default viewBox 1400×800. Better empty than cramped.
7. **Match viewBox to content aspect ratio** — no padded dead space.
8. **Align node baselines and edges.** Same row → same `y`. Same column → same `x`. The grid is what makes this style read as "designed."

## Typography

| Level         | Size  | Color      | Weight | Notes                          |
|---------------|-------|------------|--------|--------------------------------|
| Diagram title | 24–32 | `#1F1F1C`  | 600    | Tight letter-spacing           |
| Panel label   | 13–14 | `#7A756E`  | 500    | Uppercase, tracked +0.02em     |
| Node text     | 15–17 | `#2D2B28`  | 500    | Sentence case                  |
| Node subtitle | 12–13 | `#7A756E`  | 400    | Optional second line           |
| Edge label    | 12–13 | `#5F5A54`  | 400    | Lowercase typical              |

System sans-serif throughout. The uppercase, slightly-tracked panel labels are the one typographic flourish — they read as "section heading" and reinforce that panels are structural.

Short noun/verb-phrase labels. Never sentences inside nodes.

## Responsiveness

Same rules as the parent skill: viewport meta tag, `viewBox` + CSS width 100%, no fixed pixel `width`/`height` on `<svg>`, no media-query reflow. For phone-friendly diagrams pick a squarer viewBox at authoring time (e.g. 1000×900).

## Core style principles

1. Outline only — no fills. Color is for accent only, not for blocking.
2. Panels carry hierarchy. Plan them first.
3. Strict orthogonal routing.
4. One accent color maximum, and only when it carries meaning.
5. Grid alignment is mandatory. Off-grid nodes destroy the calm.
6. Spacing > color for hierarchy.
7. No shadows, no gradients, no UI chrome treatments.

## Quality checklist

Before writing the file, verify mentally:

- [ ] Canvas background `#F2EFE8` set on body
- [ ] `<meta name="viewport" content="width=device-width, initial-scale=1">` present
- [ ] `<svg>` has no fixed pixel `width`/`height` — uses `viewBox` + CSS
- [ ] `viewBox` aspect ratio hugs the actual content bounds
- [ ] Every node has `fill="none"` (outline only)
- [ ] Every panel border is dashed, unfilled, with a fieldset/legend label
- [ ] Panel labels uppercase + tracked
- [ ] All arrows are orthogonal — no diagonals, no curves
- [ ] All arrowheads are small filled triangles
- [ ] No two arrows share the same centerline
- [ ] Cross-panel arrows route in the inter-panel gap
- [ ] Every edge label is either centered with a backing rect *or* inline between two line segments — pick one and stay consistent
- [ ] At most one accent color used
- [ ] Coordinates snapped to multiples of 10
- [ ] Same-row nodes share `y`; same-column nodes share `x`

## Known limitation

Static SVG authored without a rendering loop can miss fine alignment issues (label backings off by a pixel, arrow endpoints clipping into a node border). For non-trivial diagrams, expect one iteration after the user sees it rendered — treat the first output as a draft.
