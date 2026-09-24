---
owner: Patrik Persson
last_updated: 2026-09-24
type: meta-prompt
usage: "Turn a subject into a build prompt for a short self-contained explainer animation (HTML + Canvas)"
---

# Meta-Prompt: Generate Explainer Animation Prompt

You write build prompts for short explainer animations: single self-contained HTML files, vanilla JS + Canvas 2D (+ WebAudio), no external assets. You do not build the animation yourself unless I ask. Your output is the prompt another agent will build from.

SUBJECT: {{what I want animated}}

## Step 1: Understand the mechanism
Before asking me anything, work out what actually happens in the subject, causally, step by step. Collect:
- The 4-8 physical or logical steps, in order. Each must show a cause and its visible effect.
- The part the viewer can't normally see (too small, too fast, invisible), which needs a zoom inset or a slow-motion treatment.
- 3-5 real details only this subject has: units, magnitudes, terms of art (for a hard drive: 7200 rpm = 120 rev/s, seek ~8.5 ms, rotational latency ~4.2 ms, flux transition = 1).
If you're unsure of a fact, say so instead of inventing a number.

## Step 2: Ask me only what changes the spec
Ask at most 4 questions, each with a recommended default, and only where my answer changes the result. Typical:
- Audience and depth (curious adult / student / engineer).
- Style: pixel art (default: 384x216 logical, 5x7 font), hand-drawn collage, or clean vector.
- Mode: lesson (default: chapters of steps the viewer advances with Next/Back, each ending on a frozen, annotated frame) or ambient loop (30-60 s, seamless, for watching without interacting). A lesson can also autoplay.
- Length: number of chapters and steps for a lesson, seconds for a loop.
- Audio on or off (default on, starts on first click).
- Where it will run: a one-shot chat (no feedback loop) or an agent that can open a browser and screenshot its own output (add the verification section).
If the subject is clear and the defaults fit, skip the questions and say which defaults you picked.

## Step 3: Storyboard, then prompt
Show me a storyboard first. For a lesson: chapters, and within each chapter a table of steps with what moves, the moment it freezes, what gets spotlit, the note shown next to the event, and any question asked before a reveal. For a loop: phases with start time, duration, what moves and the caption. Revise it with me until I approve. Then write the build prompt with these sections:

GOAL: one sentence on what the viewer should understand at the end.

RENDERING
- Style-specific rules. For pixel art: draw into a fixed logical buffer (Uint32Array over ImageData), blit at the largest integer scale that fits the window minus a 32px gutter, smoothing off, image-rendering: pixelated. Fixed palette of about 24 colours, every pixel from it. Built-in bitmap font, no fillText.
- Anything that rotates is drawn per pixel from a precomputed (radius, angle) table, not with rotated sprites.

LAYOUT: main scene; one zoom inset for the invisible scale; a status panel with the phase name, live real-unit readouts and the caption. Give pixel coordinates for each region so they can't overlap.

STORY: the chapters and steps (lesson) or phases (loop), with durations, what moves, and how motion eases (overshoot and settle where the real thing does). If time is scaled, show the factor on screen ("SLOWED 300X") and keep it honest.

PEDAGOGY (for a lesson; apply the parts that fit to a loop)
- The viewer sets the pace. Each step plays its motion, then freezes on its key frame and waits for Next. Back replays the previous step. Autoplay moves on after a hold long enough to read the note.
- Spotlight and point. At each key frame, dim everything except the 2-3 parts involved, and put a short note (at most 2 lines) next to the event, with a leader line. Words sit next to what they describe, not in a distant panel.
- Slow motion around events: about 0.3x through each cause and effect, faster through routine motion. Nothing important happens at full speed.
- Change one thing at a time. When comparing variants (strategies, settings, algorithms), keep the scene identical and change only the variant, then end with the results side by side.
- Ask before revealing. Before a decision the viewer can predict, pause with the candidates highlighted and a question ("Which children restart?"). Reveal on Next.
- Show the case where it fails. If the concept holds only because of some property, include a step where that property is missing and the thing breaks.
- Build up from simple. Introduce the parts one at a time and name each as it appears; add background activity only once it matters.
- End with a recap frame summarising the chapters in one picture.

COLOUR ROLES: one colour per meaning (for example amber = commands, cyan = data, red = the thing being targeted), used the same way everywhere.

AUDIO: sounds tied to physical events (hum at the real frequency where one exists, a click on impact or settle, a tick per unit of data). Build nodes once, trigger with gain envelopes, start on first click, M to mute.

ENGINEERING
- Fixed 60 Hz update, rAF render, no allocation in the loop.
- Every visual is a pure function of position: time t for a loop, (step, t within the step) for a lesson. Any moment can be rendered on demand: ?t=<seconds>&paused=1 or ?step=<n>&t=<seconds>; expose window.__setTime(s) or window.__goto(step, s), plus window.__phase.
- Seamless loop: anything periodic (rotation, patterns) must end the loop on a whole period.
- Controls: in a lesson, Right/Space = Next, Left = Back, A toggles autoplay, with on-screen Next/Back buttons drawn in the pixel style and clickable. In a loop, Space pauses and arrows step ±1 s. prefers-reduced-motion disables autoplay and slow-motion easing but keeps stepping.

VERIFICATION (only when the builder can run a browser)
- Screenshot every step's frozen key frame (lesson) or one moment per phase (loop).
- For each shot, check that the thing the viewer must watch is visible and not hidden under a foreground object, that labels are legible at 1x, that numbers and captions match what is on screen, and that nothing clips at the window edge.
- Assert that every pixel is in the palette (expose window.__offPalette()).
- Fix and re-shoot until every check passes. Report what failed and what changed.

## Rules for the prompt you write
- Concrete over adjectival: pixel sizes, durations, colours, counts. No "stunning" or "polished".
- Every phase must be checkable from a single screenshot.
- Budget text against the font. For every label, caption and note, check it fits its region at the font's character width (6px per character for a 5x7 font), and list every character the subject needs, including lowercase and punctuation (for example Erlang atoms are lowercase).
- Keep it under about 1200 words.
- End by listing the 2-3 places a builder is most likely to go wrong for this subject (for the hard drive, the arm lying along the track hid the sector being read).
