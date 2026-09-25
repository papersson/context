---
owner: Patrik Persson
last_updated: 2026-09-25
type: meta-prompt
usage: "Turn a subject into a build prompt for a short self-contained explainer lesson or animation (HTML + Canvas, optionally SVG + DOM text)"
---

# Meta-Prompt: Generate Explainer Animation Prompt

You write build prompts for explainer lessons and animations: single self-contained HTML files, vanilla JS + Canvas 2D (+ SVG and DOM text in hybrid mode, + WebAudio), no external assets except a Google Fonts stylesheet. You do not build the lesson yourself unless I ask; your output is the prompt a builder agent will follow. If you can run agents and a browser, you also run the builder and check its work.

SUBJECT: {{what I want explained}}

## Step 1: Gather sources, then understand the mechanism
Ground the lesson in primary material before writing anything. Lessons written from memory are the weakest ones.
- Collect sources: clone the relevant repos (check their licences), fetch the paper or book, save transcripts. When sources are large, have agents read them and write notes with file:line or section/table references for every claim.
- Then work out what actually happens, causally: 4-8 steps in order, each with a cause and a visible effect; the part the viewer can't normally see (too small, too fast, invisible), which needs a zoom inset or slow motion; and 3-5 real details only this subject has (units, magnitudes, terms of art, real field and function names).
- Write a FACTS block: every number, name and claim the lesson will show, each with its source. Mark anything invented as "illustrative" or "constructed from the code". Say so when you are unsure instead of inventing.
- Read the numbers critically. Headline claims often carry fine print (what "35x fewer" actually counts, which tasks an average covers); the lesson should show the fine print.

## Step 2: Ask me only what changes the spec
At most 4 questions, each with a recommended default, only where my answer changes the result. Typical:
- Audience and depth (curious adult / engineer who knows the theory / expert).
- Mode: stepped lesson (default: chapters of steps advanced with Next/Back, each ending on a frozen, annotated frame) or ambient loop.
- Render mode: hybrid (default for text-, code- or math-heavy subjects: a pixel canvas for the picture, SVG for geometry, real DOM text for notes, code and readouts) or pure pixel (for pictorial or physical subjects: every pixel from a fixed palette, bitmap fonts).
- Level of abstraction: concrete (real code and data) or first principles (my own clean abstractions, labelled "teaching model").
If the defaults fit, skip the questions and say which you picked.

## Step 3: Storyboard, then prompt
Show a storyboard first: chapters, and within each chapter a table of steps with what moves, the key frame it freezes on, what is spotlit, the note (at most 2 lines of 38 characters) and any question asked before a reveal. Revise it with me until I approve. Then write the build prompt with these sections:

GOAL: what the viewer can explain at the end, as 3-6 numbered points.

FACTS: the block from step 1, with sources. Every number on screen must come from here or be computed by the page.

RENDERING
- Hybrid mode: one 480x270 stage div scaled by an integer with transform: scale(k) and a 32 px gutter; inside it a canvas for the picture (image-rendering: pixelated), an SVG for geometry and leader lines, and a DOM layer for notes, code panels, readouts and real buttons, in a pixel font (e.g. Silkscreen) plus a readable monospace font for code. Palette as CSS custom properties mirrored into the canvas. Code panels can show fn.toString() of the live functions, so the code on screen is the code that ran.
- Pixel mode: a fixed logical frame (384x216, or 480x270 when there is code), palette indices in a Uint8Array converted through a lookup table, a DIM table for the spotlight, integer scaling with a 32 px gutter. A 5x7 bitmap font over full printable ASCII with descenders (g, j, p, q, y), plus a small mono font for code. No fillText.
- Anything that rotates is drawn per pixel from a precomputed (radius, angle) table.

LAYOUT: regions with coordinates so nothing overlaps; notes next to their target with a leader line, never covering it.

STEPS: the approved storyboard, with note texts verbatim.

PEDAGOGY
- The viewer sets the pace: steps freeze on key frames; Right/Space/Next advances, Left/Back goes back, A toggles autoplay.
- Spotlight the 2-3 parts that matter; dim the rest.
- Slow motion through the cause and effect; faster through routine motion.
- One variable at a time: compare variants on an identical scene.
- Guess before reveal: before a predictable decision, show the candidates and a question; reveal nothing until Next.
- Show the case where it fails (the counterexample that makes the rule necessary).
- Build up from simple; name each part as it appears.
- End with a recap frame.
- Keep it minimal. Clean, sparse scenes teach better than busy ones: one idea per frame, few elements on screen, generous empty space.

COLOUR ROLES: one colour per meaning, used the same way everywhere.

AUDIO: sounds tied to events, nodes built once, gain envelopes, start on first click, M mutes.

ENGINEERING
- Fixed 60 Hz update, rAF render, no allocation in the loop.
- Every frame is a pure function of (step, t, selection). ?step=n&t=s&paused=1 renders that moment locally, but the page must start at step 1 with no query string (viewers strip it).
- Test hooks: __goto(n, s), __steps, __state(), __hash() (a frame hash; in hybrid mode it includes DOM text and SVG attributes), __offPalette() (pixel mode), and __check(), which must return [] and covers: notes inside the stage, not covering their target, leaders not crossing text; no overlapping text boxes; every text fits its box; every element inside the stage; readouts equal a fresh recomputation; code panels equal their source.
- Pure ASCII file (\u escapes). No DOCTYPE/html/head/body tags when the host adds its own skeleton.

VERIFICATION (when the builder can run a browser)
- Screenshot every step's key frame and a few mid-motion frames (paused) at two viewport sizes; look at every one.
- __check() is [] on every step; no console errors; __offPalette() is 0 in pixel mode.
- Back from three steps matches a direct __goto by __hash.
- Question steps reveal nothing.
- Buttons and clicks work with real mouse events. Known tool gotchas: agent-browser mouse down/up fires at (0,0), so dispatch MouseEvents with clientX/clientY; `press` can open browser pages, so dispatch KeyboardEvents; serve the folder over a local HTTP server.
- Code excerpts are checked against their source files by a script (shortening with "..." is fine, changed identifiers are not).
- A fresh-eyes pass: a reviewer who sees only the screenshots, cold, reports anything unclear; fix it without changing the note texts.

LIKELY MISTAKES: end with the 2-4 ways a builder is most likely to get this subject wrong.

## Rules for the prompt you write
- Concrete over adjectival: pixel sizes, durations, colours, counts.
- Every step checkable from a single screenshot.
- Budget text against the font (6 px per character at 5x7) and list every character the subject needs.
- Originality: original characters and scenes only; no real people, teams or brands unless the subject requires them. Quote code only from permissively licensed sources, with credit; for other sources use your own summaries and short quotes.
- Reuse the previous lesson's engine when one exists instead of rebuilding it.
- Keep it under about 1500 words.
