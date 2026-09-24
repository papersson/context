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
- Length and loop (default 30-60 s, seamless loop).
- Audio on or off (default on, starts on first click).
- Where it will run: a one-shot chat (no feedback loop) or an agent that can open a browser and screenshot its own output (add the verification section).
If the subject is clear and the defaults fit, skip the questions and say which defaults you picked.

## Step 3: Storyboard, then prompt
Show me a storyboard first: a table of phases with start time, duration, what moves, and the on-screen caption (at most 2 short lines). Revise it with me until I approve. Then write the build prompt with these sections:

GOAL: one sentence on what the viewer should understand at the end.

RENDERING
- Style-specific rules. For pixel art: draw into a fixed logical buffer (Uint32Array over ImageData), blit at the largest integer scale that fits the window minus a 32px gutter, smoothing off, image-rendering: pixelated. Fixed palette of about 24 colours, every pixel from it. Built-in bitmap font, no fillText.
- Anything that rotates is drawn per pixel from a precomputed (radius, angle) table, not with rotated sprites.

LAYOUT: main scene; one zoom inset for the invisible scale; a status panel with the phase name, live real-unit readouts and the caption. Give pixel coordinates for each region so they can't overlap.

STORY: the phases as a looping state machine, with durations, what moves, and how motion eases (overshoot and settle where the real thing does). If time is scaled, show the factor on screen ("SLOWED 300X") and keep it honest.

COLOUR ROLES: one colour per meaning (for example amber = commands, cyan = data, red = the thing being targeted), used the same way everywhere.

AUDIO: sounds tied to physical events (hum at the real frequency where one exists, a click on impact or settle, a tick per unit of data). Build nodes once, trigger with gain envelopes, start on first click, M to mute.

ENGINEERING
- Fixed 60 Hz update, rAF render, no allocation in the loop.
- Every visual is a pure function of time t, so any moment can be rendered on demand. ?t=<seconds>&paused=1 renders that moment; expose window.__setTime(s) and window.__phase.
- Seamless loop: anything periodic (rotation, patterns) must end the loop on a whole period.
- Space pauses, arrow keys step ±1 s, prefers-reduced-motion starts paused.

VERIFICATION (only when the builder can run a browser)
- Screenshot one moment per phase with ?t=.
- For each shot, check that the thing the viewer must watch is visible and not hidden under a foreground object, that labels are legible at 1x, that numbers and captions match what is on screen, and that nothing clips at the window edge.
- Assert that every pixel is in the palette (expose window.__offPalette()).
- Fix and re-shoot until every check passes. Report what failed and what changed.

## Rules for the prompt you write
- Concrete over adjectival: pixel sizes, durations, colours, counts. No "stunning" or "polished".
- Every phase must be checkable from a single screenshot.
- Keep it under about 900 words.
- End by listing the 2-3 places a builder is most likely to go wrong for this subject (for the hard drive, the arm lying along the track hid the sector being read).
