# The Art of the Pull Request

A practical guide to writing PRs that reviewers love and future-you will thank you for.

---

## Core Philosophy

**Your job is to deliver code you have proven to work.**

A pull request is not a code dump. It's a **narrative** — a persuasive argument that this change should exist, bundled with **evidence that it actually works**. Submitting untested code and expecting reviewers to validate it is a shifting of burden. It's rude, wastes others' time, and is honestly a dereliction of duty.

The best PRs answer three questions before they're asked:
1. **Why** does this change need to exist?
2. **What** does it actually do?
3. **How** have you proven it works?

---

## Before You Write Code

### 1. Scope ruthlessly

The single biggest predictor of PR quality is size. Studies consistently show:
- PRs under 200 lines get reviewed in minutes
- PRs over 400 lines get skimmed or delayed for days
- PRs over 1000 lines often get rubber-stamped (defeating the purpose)

**The rule:** If you can describe your PR with "and" ("fixes the bug *and* refactors the module *and* updates the tests"), split it.

**Techniques for breaking down work:**
```
Feature: Add user authentication

Split into:
1. Add password hashing utility (no dependencies)
2. Add user model with password field
3. Add login endpoint (depends on #1, #2)
4. Add session management
5. Add logout endpoint
6. Add integration tests
```

Each PR is independently reviewable, testable, and revertable.

### 2. Understand the codebase conventions

Before your first PR to a repo, study:
- Existing PR descriptions (what level of detail is expected?)
- Commit message conventions (Conventional Commits? Issue references?)
- Code style (is there a linter config? `.editorconfig`?)
- Test conventions (unit vs integration, naming patterns)

Match the existing style, even if you disagree. Style debates belong in separate PRs.

---

## Proving Your Code Works

This is non-negotiable. If you haven't proven your code works, you haven't finished the work — you've just shifted the burden to your reviewer.

### Manual Testing

If you haven't seen the code do the right thing yourself, that code doesn't work. If it does turn out to work, that's pure luck.

**Manual testing is a skill.** You need to:
1. Get the system into an initial state that demonstrates your change
2. Exercise the change
3. Verify and demonstrate it has the desired effect

**Include your evidence in the PR.** Options:

```markdown
## Testing

Tested locally by running:

$ curl -X POST localhost:8080/api/users -d '{"name": "test"}'
{"id": 42, "name": "test", "created_at": "2025-01-07T10:30:00Z"}

$ curl localhost:8080/api/users/42
{"id": 42, "name": "test", "created_at": "2025-01-07T10:30:00Z"}
```

For UI changes, **record a screen capture** or include before/after screenshots:

```markdown
## Before
![Old dialog](screenshots/before.png)

## After  
![New dialog with validation](screenshots/after.png)
```

**Test the edge cases.** Once the happy path works, try to break it:
- Empty inputs
- Extremely large inputs
- Special characters, unicode, emoji
- Concurrent access
- Network failures
- Permission boundaries

Finding the things that break is what separates senior engineers from juniors.

### Automated Testing

Your change should include automated tests that:
1. **Pass** with your implementation
2. **Fail** if you revert the implementation

If the test passes whether or not your code exists, it's not testing your change.

```markdown
## Testing

Added `test_user_creation_with_special_characters` which verifies:
- Names with unicode characters are stored correctly
- Names with HTML entities are escaped
- Empty names are rejected with 400

Run: `pytest tests/test_users.py -k special_characters`
```

**Don't skip manual testing because you have automated tests.** Almost every time you do this, you'll regret it. Automated tests check what you thought to check. Manual testing catches what you didn't think of.

### When Using AI/LLM Tools

If you're using coding agents, **make them prove their changes work too**:
- Teach them to run the code and verify output
- Have them take screenshots for UI changes
- Ensure they write tests that actually exercise the change

A computer can generate a thousand-line patch. That's not valuable. What's valuable is proven, working code. **The human provides the accountability.**

---

## Writing Style

PRs are technical writing. The goal is clarity and efficiency, not flair.

### General Principles

**Be direct.** Say what you mean in as few words as possible.

```
❌ "I decided to go ahead and implement a solution that basically 
    addresses the issue by making some changes to how we handle..."

✅ "URL-encode user input before validation."
```

**Use active voice.** Passive voice obscures who did what.

```
❌ "The bug was fixed and tests were added."
✅ "Fixed the null pointer bug. Added regression test."
```

**Assume intelligence, not context.** Your reader is smart but doesn't have your mental model. Explain the situation, not the basics.

```
❌ "As you know, HTTP 500 means server error..."  (condescending)
❌ "Fixed the thing."  (no context)
✅ "Users hitting /api/submit with special characters get 500s 
    because the validation service doesn't URL-decode input."
```

**Cut filler words.** These add nothing:

- "Basically", "essentially", "actually"
- "I think", "I believe", "In my opinion" (just state it)
- "Just", "simply", "obviously"
- "In order to" → "to"
- "Due to the fact that" → "because"

### Commit Messages

**Use imperative mood.** Write as if commanding the codebase:

```
❌ "Added validation for email"
❌ "Adding validation for email"  
❌ "Adds validation for email"
✅ "Add validation for email"
```

This matches Git's own style ("Merge branch...", "Revert...") and reads naturally after "This commit will...".

**First line: what. Body: why.**

```
Add rate limiting to /api/submit

Users were able to hammer this endpoint, causing downstream
timeouts. This adds a 10 req/min limit per IP.

Considered per-user limiting but anonymous submissions
make that ineffective. IP-based is imperfect but handles
the immediate abuse vector.

Fixes #892
```

### PR Descriptions

**Lead with the problem, not the solution.** Reviewers need to understand *why* before *what*.

```
❌ "This PR adds a cache invalidation check..."

✅ "Users occasionally see stale data after updates. 
    This happens because cache invalidation races with writes.
    
    This PR adds a version check before serving cached results."
```

**Be specific.** Vague descriptions waste reviewer time.

```
❌ "Fixes bug with user input"
✅ "Fixes 500 error when usernames contain apostrophes"

❌ "Improves performance"
✅ "Reduces p99 latency from 800ms to 120ms by batching DB queries"
```

**Quantify when possible.** Numbers are more convincing than adjectives.

```
❌ "This affects many users"
✅ "This affects ~2% of submissions (~500/day)"

❌ "Much faster"
✅ "3x faster (40ms → 12ms)"
```

### Review Comments

**Be specific and actionable.**

```
❌ "This is wrong"
✅ "This will throw if `user` is null. Consider adding a null check 
    or using Optional."

❌ "Can you clean this up?"
✅ "Nit: Extract lines 42-50 to a `validateInput()` method?"
```

**Suggest, don't demand** (unless it's a blocker).

```
❌ "Change this to use a map."
✅ "Consider using a Map here — it would make the lookup O(1) 
    instead of O(n). Wdyt?"
```

**Ask questions to understand, not to challenge.**

```
❌ "Why would you do it this way?"
✅ "What led to this approach? I'm curious if you considered X."
```

### Tone

- **Professional, not formal.** Write like you're explaining to a colleague, not writing a legal document.
- **Confident, not arrogant.** State your reasoning without hedging excessively, but stay open to feedback.
- **Helpful, not condescending.** Assume good intent. People are trying their best.

---

## The PR Description

This is the most underinvested part of most PRs. A great description is worth 10x the time it takes to write.

### Template

```markdown
## Problem

[What's broken, missing, or suboptimal? Link to issue if exists.]

## Solution

[What does this PR do? High-level summary, not a code walkthrough.]

## Approach

[Why this solution over alternatives? What tradeoffs were made?]

## Proof It Works

[This is not optional. Show your evidence.]

**Manual testing:**
- Steps you took to verify
- Terminal output, screenshots, or video link

**Automated testing:**
- What tests were added/modified
- How to run them

## Notes for Reviewers

[Optional: suggested review order, areas of uncertainty, questions]

## Checklist

- [ ] I have manually tested this change
- [ ] Tests added/updated
- [ ] Documentation updated (if applicable)
- [ ] No unrelated changes
- [ ] Self-reviewed the diff
```

### Description Anti-Patterns

**❌ The Empty Description**
```
Title: Fix bug
Description: (empty)
```

**❌ The Code Walkthrough**
```
In file X, I changed line 42 to use Y instead of Z.
Then in file A, I added method B which calls C.
```
(The diff already shows this. Explain *why*.)

**❌ The Novel**
```
(2000 words explaining the entire history of the codebase)
```

**✅ The Goldilocks Description**
```
## Problem

Users see a 500 error when submitting forms with special characters
in the "name" field. This affects ~2% of submissions (see #1234).

## Solution

URL-encode user input before passing to the validation API. Added
input sanitization at the form boundary rather than deeper in the
stack to catch all entry points.

## Why this approach

Considered fixing in the validation API itself, but that would require
coordinating a deploy with the platform team. This solution is local
to our service and handles the immediate issue. Filed #1235 to track
the API-level fix.

## Testing

- Added unit test for special character handling
- Manually tested with: é, ñ, <script>, emoji 🎉
- Ran against staging with production traffic sample
```

---

## Commits

Your commits are the chapters of your narrative. They should tell a story.

### Commit Hygiene

**Atomic commits:** Each commit should compile and pass tests independently (when possible). This enables:
- `git bisect` for debugging
- Cherry-picking specific changes
- Easier reverts

**Logical grouping:**
```
❌ Bad commit history:
- "WIP"
- "fix"
- "actually fix"
- "add tests"
- "fix tests"
- "review feedback"

✅ Good commit history:
- "Add UserService with CRUD operations"
- "Add validation for email uniqueness"
- "Add unit tests for UserService"
```

**Commit message format:**
```
<type>: <short summary in imperative mood>

[Optional body explaining WHY, not WHAT]

[Optional footer with issue references]
```

Example:
```
fix: prevent race condition in cache invalidation

The previous implementation could serve stale data when a write
and invalidation happened concurrently. This adds a version check
before serving cached results.

Measured 0.3ms latency increase, acceptable for consistency guarantee.

Fixes #892
```

### When to Squash

**Squash when:**
- Your commits are implementation noise ("WIP", "fix typo", "oops")
- The individual commits aren't meaningful to future readers

**Don't squash when:**
- Each commit represents a logical step reviewers should see separately
- You want to preserve the ability to revert specific parts
- Your team's convention is to preserve history

---

## The Diff

### Make the Diff Reviewable

**No drive-by changes.** If you notice unrelated issues while working:
- Fix them in a separate PR
- Or add a `// TODO` with an issue reference
- Never mix "while I'm here" cleanups with feature work

**Separate refactors from behavior changes.** If you need to refactor to implement a feature:
```
PR 1: Refactor X to prepare for feature (no behavior change)
PR 2: Implement feature using refactored X
```

This lets reviewers verify the refactor is behavior-preserving, then review the feature separately.

**Keep formatting changes separate.** If you're reformatting a file:
- Do it in a dedicated commit or PR
- Use your formatter's "whole file" mode so the blame history is clean
- Never mix formatting with logic changes

### Self-Review Checklist

Before requesting review, review your own diff as if you're the reviewer:

```
□ No debug code (console.log, print statements, commented code)
□ No hardcoded values that should be config
□ No secrets or credentials
□ No unrelated changes
□ Error handling is complete
□ Edge cases are handled
□ Variable/function names are clear
□ Comments explain "why", not "what"
□ Tests cover the happy path AND failure modes
□ Documentation is updated if needed
```

---

## Code Review Interaction

### The Standard of Code Review

The purpose of code review is to ensure the **overall code health improves over time** — not to achieve perfection.

There's no such thing as "perfect" code. There is only *better* code. A CL that improves maintainability, readability, and understandability shouldn't be delayed for days because it isn't "perfect."

**The principle:** Approve a PR once it definitely improves the overall code health of the system, even if it isn't perfect.

This cuts both ways:
- **Authors:** Don't expect perfection from yourself. Ship improvements.
- **Reviewers:** Don't block improvements waiting for perfection. Make forward progress.

### Requesting Review

**Choose reviewers intentionally:**
- Someone who knows this area of the codebase
- Someone who will be affected by the change
- For learning PRs, someone who can mentor

**Set expectations:**
- Is this urgent or can it wait?
- Are there specific areas you want scrutinized?
- Are there parts you're uncertain about?

```markdown
@alice — You know the auth system best, would appreciate your review
@bob — FYI since this touches the API you maintain

Particularly interested in feedback on the caching strategy in
UserCache.java — not sure if the TTL is appropriate.
```

### Responding to Feedback

**The golden rule:** Assume good intent. Comments that feel critical are usually trying to help.

**How to respond:**

| Comment Type | Response |
|-------------|----------|
| Blocking concern | Prioritize resolving before other feedback |
| Suggestion you agree with | Make the change, reply "Done" or "Good catch!" |
| Suggestion you disagree with | Explain your reasoning, propose alternatives |
| Question | Answer it (and consider if the code should be clearer) |
| `Nit:` comment | Author's choice — these are polish, not blockers |

### The "Nit:" Convention

Reviewers should prefix minor suggestions with `Nit:` to signal:
- This is a point of polish, not a blocker
- The author can choose to address it or not
- It shouldn't delay approval

```
Nit: Could rename `data` to `userData` for clarity.

Nit: Consider extracting this to a constant.
```

As a reviewer: Use `Nit:` liberally. It lets you share knowledge without blocking progress.

As an author: Address nits if they're quick wins. It's fine to skip them with a brief explanation ("Leaving as-is for consistency with adjacent code").

### Resolving Conflicts

When you disagree with a reviewer:

1. **Start with the technical facts.** Data and engineering principles trump opinions and preferences.

2. **On style matters:** The style guide is authoritative. If the guide is silent, match existing code. If there's no precedent, author's preference wins.

3. **On design matters:** These are rarely pure preference. If the author can demonstrate (with data or principles) that approaches are equally valid, reviewer should defer. Otherwise, standard design principles apply.

4. **If consensus is hard:** Move to synchronous communication — a call is worth a hundred comment threads. **Record the decision in the PR** for future readers.

5. **If still stuck:** Escalate to team discussion, tech lead, or code maintainer. Don't let PRs rot because two people can't agree.

**If a thread is getting long:** Move to synchronous communication (call, DM), then summarize the resolution in the PR.

**Never:**
- Take feedback personally
- Get defensive
- Argue about style (defer to team conventions)
- Ignore comments without responding

### Updating Your PR

**After addressing feedback:**
1. Push new commits (don't force-push if the discussion references specific commits)
2. Respond to each comment thread
3. Re-request review when ready

**If the PR scope has changed significantly:** Consider closing and opening a new PR with a fresh description.

---

## Special Cases

### Large PRs (When Unavoidable)

Sometimes you can't avoid a large PR (major refactor, generated code, etc.):

1. **Add a review guide:**
   ```markdown
   ## Review Guide

   Suggested order:
   1. Start with `schema.sql` — this defines the data model
   2. Then `UserRepository.java` — the data access layer
   3. Then `UserService.java` — business logic that uses the repository
   4. Finally `UserController.java` — HTTP layer (mostly boilerplate)

   The interesting decisions are in steps 1-3.
   ```

2. **Mark "ignore" files:** If there's generated or mechanical code:
   ```markdown
   Files that can be skimmed:
   - `migrations/*.sql` — generated from schema
   - `**/mocks/*.go` — generated mocks
   ```

3. **Use stacked PRs:** Many teams use stacked/dependent PRs:
   ```
   PR 1: Add base types (mergeable independently)
   PR 2: Add service layer (depends on PR 1)
   PR 3: Add HTTP handlers (depends on PR 2)
   ```

### Urgent/Hotfix PRs

When something is on fire:

1. **Lead with urgency:**
   ```markdown
   ## 🔥 URGENT: Production users seeing 500 errors

   This is a targeted fix for #incident-1234. Full remediation
   will follow in #1235.
   ```

2. **Keep scope minimal:** Fix the immediate problem only
3. **Get synchronous review:** Ping reviewers directly
4. **Follow up:** Create issues for proper fixes, monitoring, postmortem

### Documentation PRs

Docs deserve the same rigor as code:

- Explain what changed and why
- For new docs: who is the audience?
- For updates: what was wrong or out of date?
- Preview links if your system generates them

### Dependency Updates

```markdown
## Dependency Update: lodash 4.17.19 → 4.17.21

### Why
- Security fix for CVE-2021-23337 (prototype pollution)
- No breaking changes per changelog

### Verification
- All tests pass
- Manually tested affected code paths
- No deprecated API usage in our codebase

### Changelog
https://github.com/lodash/lodash/releases/tag/4.17.21
```

---

## PR Etiquette

### As an Author

- Respond to reviews within 24 hours (or set expectations if you can't)
- Don't let PRs go stale — if it's been open a week, something is wrong
- Be grateful for thorough reviews, even harsh ones
- After merging, delete the branch
- **Include your proof.** Don't make reviewers guess if it works.

### As a Reviewer

- Review within 24 hours or decline/reassign
- Be specific ("consider using X here" not "this is wrong")
- **Distinguish blocking issues from suggestions** — use `Nit:` for non-blockers
- **Approve when it improves code health**, even if imperfect
- Praise good work, not just critique problems

### Mentoring Through Reviews

Code review is an opportunity to teach. It's fine to leave educational comments about:
- Language features the author might not know
- Framework best practices
- General software design principles

Prefix these with `Nit:` or note that they're not blocking:

```
Nit: FYI, you could also write this using pattern matching 
(new in Java 21). Not required for this PR, just sharing!
```

Sharing knowledge improves code health over time. Just don't make learning mandatory for approval.

---

## Checklist Summary

### Before Creating the PR
- [ ] Scope is minimal and focused
- [ ] Commits are logical and atomic  
- [ ] **Manually tested the change**
- [ ] **Automated tests included**
- [ ] Self-reviewed the diff
- [ ] No unrelated changes

### The Description
- [ ] Problem is clearly stated
- [ ] Solution is explained (not just what, but why)
- [ ] **Evidence included** (terminal output, screenshots, test commands)
- [ ] Related issues/PRs are linked

### The Diff
- [ ] No debug code
- [ ] No formatting-only changes mixed with logic
- [ ] Code is documented where non-obvious
- [ ] Error handling is complete

### After Review
- [ ] All comments are addressed (or explicitly deferred)
- [ ] CI is passing
- [ ] Approvals are obtained
- [ ] Branch is deleted after merge

---

## Further Reading

- [The Standard of Code Review](https://google.github.io/eng-practices/review/reviewer/standard.html) — Google Engineering Practices
- [Your Job Is to Deliver Code You Have Proven to Work](https://simonwillison.net/2025/Jun/26/your-job-is-to-deliver-code-you-have-proven-to-work/) — Simon Willison
- [How to Write a Git Commit Message](https://cbea.ms/git-commit/) — Chris Beams
- [Stacked Diffs vs Pull Requests](https://jg.gg/2018/09/29/stacked-diffs-versus-pull-requests/) — Jackson Gabbard
- [Ship Small Diffs](https://blog.skyliner.io/ship-small-diffs-741308bec0d1) — Skyliner Blog

---

*"A pull request should be a gift to your reviewer, not a burden — wrapped with proof that it works."*
