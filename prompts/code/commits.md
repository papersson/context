# Commit Guidelines

Organize changes into logical atomic commits for easier review.

---

## Principles

### 1. Each Commit Should Be Self-Contained

A commit should:
- Compile and pass tests on its own
- Represent one logical change
- Be understandable without external context (don't reference tickets, specs, or milestones)

### 2. Build Dependencies First

Order commits so each builds on the previous:
```
1. Add utility/helper code
2. Add core data structures
3. Add main implementation
4. Add tests/verification
5. Add documentation
```

### 3. Separate Concerns

Split by type of change:
- **New files** — one commit per logical unit (not per file if files are coupled)
- **Refactoring** — separate from behavioral changes
- **Bug fixes** — separate from features
- **Documentation** — separate from code changes

---

## Commit Message Format

```
<short summary in imperative mood>

<optional body explaining why, not what>
<what is evident from the diff, why is not>
```

### Good Examples

```
Add safetensors loader with BF16 conversion

Header-only parser using mmap for efficient loading.
Converts BF16 to FP32 on the fly (BF16 is truncated FP32,
so conversion is just a 16-bit left shift).
```

```
Fix off-by-one in token position calculation

Was using 0-indexed position for RoPE but the cache
expects 1-indexed. This caused incorrect embeddings
for all tokens after the first.
```

### Bad Examples

```
# Too vague
Fix bug

# References external context
Implement M1 from spec

# Describes what (obvious from diff), not why
Change hidden_size from 512 to 1024

# Multiple unrelated changes
Add parser and fix tests and update docs
```

---

## Process

When you have many uncommitted changes:

1. **List what changed** — `git status`, `git diff --stat`

2. **Identify logical units** — group related files/changes

3. **Determine order** — dependencies first, then dependents

4. **Stage and commit incrementally**:
   ```bash
   git add <files-for-first-logical-unit>
   git commit -m "..."
   git add <files-for-second-logical-unit>
   git commit -m "..."
   ```

4. **Verify each commit** — ideally each should build: `git stash && make && git stash pop`

---

## Splitting a Large Diff

If you have one big change that should be multiple commits:

**By file:**
```bash
git add file1.c file2.c
git commit -m "Add core data structures"
git add file3.c
git commit -m "Add implementation"
```

**By hunk (interactive):**
```bash
git add -p  # stage specific hunks within a file
```

**By rewriting history (if not yet pushed):**
```bash
git reset HEAD~1  # undo last commit, keep changes
# re-commit in smaller pieces
```

---

## Example Breakdown

A feature adding "user authentication" might become:

| # | Commit |
|---|--------|
| 1 | Add password hashing utilities |
| 2 | Add user model with email/password fields |
| 3 | Add authentication middleware |
| 4 | Add login/logout API endpoints |
| 5 | Add authentication tests |

Not one commit: "Add user authentication"
