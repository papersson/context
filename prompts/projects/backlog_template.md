# BACKLOG: [Project Name]

**Spec:** `./SPEC.md`
**Status:** [Scaffolding | In Progress | Wrapping Up | Complete]
**Last Updated:** [YYYY-MM-DD]

---

## Agent Instructions

**At session start:**
1. Read the spec (`SPEC.md`)
2. Read this backlog
3. Check the latest commits for context
4. Plan your session — pick items to work on, starting from the highest-priority not-started item in the current phase

**During implementation:**
- Work through items in phase order; don't skip ahead to a later phase unless the current phase is complete or blocked
- If you discover that an item needs splitting, is blocked, or the spec has a gap — note it but keep working on what you can

**At session end:**
1. For each item you completed, prepare the verification so the human can run it
2. Present a session summary:
   - Which items were completed (with a quick demo or output for each)
   - Any proposed changes to this backlog (new items, reordering, splits, scope changes)
   - What you'd recommend tackling next session
3. **Do not update this file directly.** Propose your changes and let the human approve them.

---

## Phase 0: Scaffolding
*Verification style: "Show me the structure and explain why it's shaped this way."*

| # | Item | Status | Note |
|---|------|--------|------|
| 0.1 | [e.g., Project structure and build setup] | ⬜ not-started | |
| 0.2 | [e.g., Core type definitions from spec §4] | ⬜ not-started | |
| 0.3 | [e.g., Dev tooling — linter, test runner, scripts] | ⬜ not-started | |

---

## Phase 1: [Name — e.g., Core Happy Path]
*Goal: [One sentence describing what's true when this phase is done]*

| # | Item | Status | Note |
|---|------|--------|------|
| 1.1 | [Item title — a vertical slice] | ⬜ not-started | |
| | **Verify:** [What the human does to confirm this works] | | |
| | **Fallback:** [Quicker/simpler alternative verification] | | |
| | **Spec ref:** [§X, example VN] | | |
| 1.2 | [Item title] | ⬜ not-started | |
| | **Verify:** [...] | | |
| | **Fallback:** [...] | | |
| | **Spec ref:** [...] | | |

---

## Phase 2: [Name — e.g., Error Handling & Edge Cases]
*Goal: [One sentence]*

| # | Item | Status | Note |
|---|------|--------|------|
| 2.1 | [Item title] | ⬜ not-started | |
| | **Verify:** [...] | | |
| | **Fallback:** [...] | | |
| | **Spec ref:** [...] | | |

---

## Phase N: [Name — e.g., Polish & Integration]
*Goal: [One sentence]*

| # | Item | Status | Note |
|---|------|--------|------|
| N.1 | [Item title] | ⬜ not-started | |
| | **Verify:** [...] | | |
| | **Fallback:** [...] | | |
| | **Spec ref:** [...] | | |

---

## Status Key

| Status | Meaning |
|--------|---------|
| ⬜ not-started | Not yet attempted |
| 🔨 in-progress | Started, not complete |
| ✅ done | Complete — verification step is ready for the human |
| ⛔ blocked | Cannot proceed — see note for reason |
| ✂️ split | Replaced by smaller items |

---

## Guidance for Writing Good Items

**An item is a vertical slice when possible.** "System accepts a CSV and returns a summary" is verifiable. "CSV parser module" is not — the human can't see it work without reading code.

**Early phases are the exception.** Scaffolding and infrastructure items don't have user-visible behavior. Their verification is structural: "show me the project layout," "show me the type definitions match the spec."

**Verification steps are written for the human, not the test runner.** The human should be able to verify in under a few minutes by observing system behavior. Good: "Run `./cli --input sample.csv` and check that the output matches spec example V2." Bad: "Run `npm test` and confirm all tests pass."

**Verification should connect to intent, not implementation.** The human doesn't care that `parseConfig` returns the right object. They care that the system behaves correctly when given a config file.

**Reference the spec.** Verification steps should point to specific spec sections or examples. This keeps the backlog light and the spec as the source of truth.

**Fallback verification is the quick sanity check.** If the primary verification requires setup or running the full system, the fallback is something simpler — checking a file was created, eyeballing a log output, or running a smaller command.

**Items should be completable in a single session.** If an item can't reasonably be finished in one coding session, it's too big — split it.
