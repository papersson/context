---
owner: Patrik Persson
last_updated: 2026-09-25
type: meta-prompt
usage: "Turn a lesson I want to learn into a build prompt for a narrated explainer video that an agent with a shell plans, voices, animates, renders and publishes end to end"
---

# Meta-Prompt: Generate Video Lesson Prompt

You write build prompts for short narrated explainer videos: a voice-over read by a text-to-speech model, over animated diagrams rendered with Manim, cut to the narration's timing and published as an MP4 on a review page. You do not make the video yourself unless I ask. Your output is one self-contained prompt that a coding agent with a shell (Claude Code or similar) runs from start to finish in a fresh session.

LESSON: {{what I want to learn, and what I already know}}

## Step 1: Work out the lesson

Before asking me anything, work out what the video has to teach and how it will show it.

- **The one idea.** One sentence on what the viewer should understand at the end, plus the payoff that makes it worth knowing. For out-of-core sorting: count block transfers, not operations, and a merge that takes as many runs as memory can hold sorts almost any file in two passes.
- **The canonical example.** Pick the worked example that textbooks and practitioners use to teach this idea, say why, and name one or two alternatives you rejected and why. Prefer an example that extends something the audience already knows (external merge sort extends merge sort).
- **An analogy that becomes a diagram.** Find one analogy that can be drawn as a fixed picture and reused in every scene. "Memory is your desk, disk is a warehouse across town, and a trip costs the same whether you carry one item or a crate" became a memory tray at the top, a disk strip at the bottom and a trip counter in the corner. The picture's geography never changes.
- **Four running examples.** A hook at real scale (a 100 GB file on a laptop with 16 GB of memory). A toy sized so every step is visible and the arithmetic comes out exact (48 cards, 4 per block, memory for 16: three runs, and three input buffers plus one output buffer fill memory exactly). A simulation or measurement behind every comparative claim. The real-scale math (16 GiB of memory with 1 MiB blocks merges 16,383 runs at once).
- **Real details only this subject has.** Units, magnitudes, formulas, defaults, tool names and what the tools print. For each one, note whether the agent can measure it in its environment, simulate it, or has to cite it.
- **Claims that need care.** Popular claims that are wrong or only partly true, and numbers that depend on a parameter (the gap between heapsort and merge sort was 7× with 64-item blocks and 26.5× with 256-item blocks). Decide the wording that will survive checking.

If you're unsure of a fact, say so instead of inventing a number.

## Step 2: Ask only what changes the spec

Ask at most 3 questions, each with a default, and skip them when the defaults fit:
- Audience and prerequisites (default: undergraduates who know the standard coursework underneath the lesson).
- Length (default: 5 minutes, about 750 words of narration).
- Checkpoints (default: none, the agent runs end to end and I review the published page; alternatives are to stop after the plan, after the narration, or after a few concept frames).

The voice (Kokoro, af_heart), the tools and the visual language stay fixed unless I say otherwise.

## Step 3: Write the build prompt

Above the prompt, give me five lines: the one idea, the canonical example, the analogy, the segment list with times, and the hero shots (the 3-4 frames that carry the lesson). Then write the prompt with the sections below. Copy the sections marked "include as written" unchanged; they come from failures in earlier builds.

GOAL: the one idea and its payoff, in two sentences.

AUDIENCE AND LENGTH: who the viewers are, what they already know, the target length, and the narration budget at about 150 words per minute of finished video. Kokoro af_heart at 0.92× speed speaks about 167 words per minute including pauses, and silent holds on the hero shots fill the rest.

LESSON CONTENT: the canonical example and why it was chosen, the analogy and the fixed diagram it becomes, the four running examples with exact numbers, the subject's real details, and the claims that need care with the wording each must take.

SEGMENTS: 6-9 segments with start times. For each one, say what it teaches, draft the narration, describe what is on screen, name where its data comes from, and mark which moment gets a silent hold. Write narration for the ear: parameter names spoken as words ("M over B"), formulas described rather than read aloud, one idea per line. Open on a concrete hook the viewer can watch fail and succeed, and end on a recap that reuses earlier visuals.

VISUAL LANGUAGE:
- Dark background, IBM Plex Sans and IBM Plex Mono, 1920×1080 at 30 fps.
- The analogy's fixed geography in every scene. Whatever unit the lesson counts moves only as a whole (a block never travels as loose cards).
- Colour roles: one accent reserved for cost (amber), one for the current selection (ice blue), grey for idle. Values use a perceptually uniform colormap (seaborn "mako") and, in toy scenes, are also printed as numbers so colour is never the only cue.
- A persistent counter for whatever the lesson says to count. It ticks only when that thing happens and stays visibly frozen during free work.
- At most one formula on screen at a time, for about 4 seconds, and the narration says it in words.

EVIDENCE (include as written):
- Every number spoken or shown comes from a run in this environment, a simulation written for this video, or a cited source. Typical published values are labelled as typical.
- Before finalizing the narration, run a quick check of every comparative or quantitative claim and word the line to match what the check found.
- Real-world captures (terminal sessions, tool output, logs) are run scaled down when needed, labelled on screen with the scale, and replayed as clean animated UI rather than screen recordings.
- Step-by-step animations of an algorithm replay an event log from a small instrumented implementation, so every move and every counter tick is correct by construction.
- PLAN.md keeps a fact-check table: claim, how it was checked, result.

PIPELINE (include as written, filling in the subject's file names):
1. In a new folder of the working repository, write PLAN.md: learning objectives, the canonical example and rejected alternatives, the analogy, the running examples, a segment table with times, the narration draft, a visual build list, style rules, the fact-check table and risks. Commit it.
2. narration.py renders each line separately with Kokoro (voice af_heart, speed 0.92), trims Kokoro's own leading and trailing silence, and writes narration.wav, narration.mp3 and timings.json with each line's id, spoken text, caption text, start and end. Silence: 0.8 s lead-in, 0.45 s between lines, 1.2 s between segments, plus per-line holds after hero shots. A segment's video runs from its first line to the next segment's first line. Keep spoken spellings ("Postgress", "Rocks D B", years in words) separate from caption spellings. Print every line's phonemes and respell anything mispronounced; the agent cannot listen, so this is the audio review.
3. make_data.py writes every dataset the visuals use (simulation traces, event logs, images) to data/.
4. One Manim Community scene per segment. A shared module holds the palette, fonts, the fixed diagram, the counter and a cue helper: each scene loads its segment from timings.json, waits for a line with at(line_id, offset), and ends with finish(), which waits until the segment's exact length. Consecutive scenes either end and start on the same frame or fade through the background colour.
5. Render every scene at 480p15, cut a contact sheet of frames at the cue times, and check it as described under VERIFICATION. Fix and re-render only the scenes that changed.
6. build.py renders all scenes at 1920×1080 30 fps in parallel, checks each scene's length against its segment, concatenates them with ffmpeg, muxes the narration as AAC, writes WebVTT captions from timings.json, and encodes a web copy under 15 MB (libx264, CRF about 27, -tune animation).
7. Build a review page: the video, a captions toggle, a chapter strip, the script with the current line highlighted and click-to-seek, and a list of where every number comes from. Publish it as a web page if the host offers one. Claude artifacts do not serve .vtt files, so embed the caption cues in the page and attach them with video.addTextTrack. Commit the sources, data, captures, narration mp3, web video and captions; keep the wav, the 1080p master and Manim's media folder out of git.
8. Finish with the page link, the video length, and anything that could not be verified.

ENVIRONMENT (include as written):
- System packages: ffmpeg, espeak-ng, libcairo2-dev, libpango1.0-dev, pkg-config, fonts-ibm-plex. Python venv: manim, kokoro, torch, numpy, scipy, matplotlib, seaborn, soundfile.
- Kokoro downloads its weights from huggingface.co, and the CPU build of torch comes from download.pytorch.org. If the network policy blocks a host, report the host and keep working on everything that does not need it: scenes can be written against timings estimated from word counts and re-rendered once the audio exists. Do not work around a policy block.
- There is no LaTeX. Use Text and MarkupText; Pango markup handles subscripts.

MANIM PITFALLS (include as written):
- Pango lays out small text with broken kerning (letters spread apart, spaces lost between words). Render every text at 4× its font size and scale it by 1/4.
- Manim renders Text into a canvas as wide as the video and MarkupText into a 600 px canvas with wrapping, so 4× text wraps mid-line. Patch manimpango.text2svg and MarkupUtils.text2svg to use a wide canvas with no wrap width, and clear Manim's text cache after changing either.
- Every animation lasts a whole number of frames. When many short events run back to back, compute each duration from the time left before the next cue instead of using a fixed value, or the sequence overruns the narration.
- Text drops leading spaces. Indent code and terminal lines by measured character width.
- For counters and other text that changes every frame, cache one Text object per value.
- A mobject that is reachable only through a group that was removed stops rendering. When cards move between containers, remove and add the groups explicitly, and rebuild a scene's end state from scratch instead of fading back in something that was faded out.

VERIFICATION (include as written):
- For every segment, cut a contact sheet of frames at its cue times, first from the 480p15 preview and again from the final 1080p render.
- In each frame, check that nothing overlaps or runs off the frame, every label is legible at 1080p, every number on screen matches the data and the narration, the thing the viewer must watch is visible and not covered, and each animation finishes before the line that follows it.
- Every scene's length matches its segment within 0.1 s, and the video's length matches the narration's.
- Report what failed and what changed.

## Rules for the prompt you write

- Concrete over adjectival: sizes, durations, counts, colours, file names. No "engaging" or "polished".
- Self-contained: the agent starts a fresh session with none of this conversation.
- The prompt runs end to end unless I asked for checkpoints.
- Keep it under about 1800 words, most of them in the fixed sections.
- End by naming the 2-3 places where this subject's build is most likely to go wrong. For external merge sort they were: the heapsort vs. merge sort ratio depended on the block size, so the narration had to say "dozens of times fewer" and show both ratios; the fast tail of the 3-way merge overran into the next line until durations were computed from the time left; and small text lost its spacing until it was rendered at 4×.

## Example invocations

```
LESSON: external memory algorithms and out-of-core processing. I know Big-O, merge sort and heaps, but not the memory hierarchy.
```

```
LESSON: how UMAP turns high-dimensional data into a 2D map, for an undergrad ML course. Students know PCA and k-nearest neighbours.
```
