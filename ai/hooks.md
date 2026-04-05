# Claude Code Hooks: Complete Reference & Practical Guide

> **What hooks are:** User-defined shell commands, HTTP endpoints, or LLM prompts that execute automatically at specific points in Claude Code's lifecycle. They give you **deterministic control** over what the AI agent does — unlike CLAUDE.md instructions (which are suggestions the model can ignore), hooks are **guaranteed to execute** every time their conditions are met.

---

## Mental Model

Think of hooks as the difference between asking a colleague to "please run the linter" versus wiring the linter into a CI pipeline. One is a request; the other is infrastructure.

Claude Code's agentic loop has specific lifecycle moments — before a tool runs, after a file is written, when the agent decides to stop. Hooks let you inject your own code at these moments. The agent doesn't choose whether your hook runs. It just runs.

```
User prompt → Claude reasons → [PreToolUse hook] → Tool executes → [PostToolUse hook] → ... → [Stop hook] → Done
```

---

## Hook Events (Lifecycle Points)

| Event | When It Fires | Fires Once or Repeatedly? |
|---|---|---|
| **SessionStart** | Session starts, resumes, or clears | Once per session |
| **UserPromptSubmit** | User submits a prompt, before Claude processes it | Each prompt |
| **PreToolUse** | After Claude creates tool parameters, before execution | Each tool call |
| **PermissionRequest** | When the user is shown a permission dialog | Each permission |
| **PostToolUse** | Immediately after a tool completes successfully | Each tool call |
| **Notification** | Claude sends a notification (permission, idle, auth) | Each notification |
| **Stop** | Main agent finishes responding | Each completion |
| **SubagentStop** | A subagent (Task tool) finishes responding | Each subagent completion |
| **PreCompact** | Before a compact operation (manual or auto) | Each compact |
| **SessionEnd** | Session terminates | Once per session |

---

## Configuration

Hooks live in JSON settings files. Three levels of scope:

- `~/.claude/settings.json` — User-wide (applies to all projects)
- `.claude/settings.json` — Project-level (committed, shared with team)
- `.claude/settings.local.json` — Local project (not committed)

### Basic Structure

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here"
          }
        ]
      }
    ]
  }
}
```

**Matcher rules:**
- Simple string match: `"Write"` matches only the Write tool
- Regex: `"Edit|Write"` or `"Notebook.*"`
- `"*"` or `""` matches everything
- Only applies to PreToolUse, PermissionRequest, and PostToolUse
- For events like Stop, UserPromptSubmit — omit the matcher

### Three Handler Types

| Type | When to Use | Example |
|---|---|---|
| `"command"` | Deterministic rules, formatting, blocking | Run Prettier, block `rm -rf` |
| `"prompt"` | Context-aware decisions needing LLM judgment | "Should Claude stop? Are all tasks complete?" |
| `"agent"` | When codebase state verification is needed | "Check whether tests exist for changed files" |

**Command handler:**
```json
{ "type": "command", "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\"" }
```

**Prompt handler** (sends context to a fast LLM for evaluation):
```json
{ "type": "prompt", "prompt": "Evaluate if this Bash command could affect production: $ARGUMENTS" }
```

**Agent handler:**
```json
{ "type": "agent", "prompt": "Check whether tests exist for the changed files" }
```

---

## Exit Codes (How Hooks Communicate Back)

| Exit Code | Meaning | Behavior |
|---|---|---|
| **0** | Success | stdout parsed for JSON. Shown in verbose mode (Ctrl+O). For UserPromptSubmit/SessionStart, stdout is added as context Claude can see. |
| **2** | Blocking error | stderr is fed back to Claude as an error. Blocks the action (PreToolUse blocks tool call, Stop prevents stopping, etc.) |
| **Other** | Non-blocking error | stderr shown in verbose mode. Execution continues. |

**Exit code 2 behavior per event:**

| Event | What Exit Code 2 Does |
|---|---|
| PreToolUse | Blocks the tool call, shows stderr to Claude |
| PermissionRequest | Denies permission, shows stderr to Claude |
| PostToolUse | Shows stderr to Claude (tool already ran) |
| UserPromptSubmit | Blocks prompt, erases it, shows stderr to user |
| Stop | Blocks stoppage, shows stderr to Claude (keeps it working) |
| SubagentStop | Blocks subagent stoppage |
| SessionStart/SessionEnd | Shows stderr to user only |

---

## Practical Patterns

### 1. Auto-Format After Every File Edit

The highest-ROI hook for most projects. Claude writes code, it gets formatted instantly, no tokens wasted.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

### 2. Block Dangerous Commands

Prevent `rm -rf`, force pushes, or anything targeting production. This is critical for autonomous/headless mode.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous.sh"
          }
        ]
      }
    ]
  }
}
```

Example `block-dangerous.sh`:
```bash
#!/bin/bash
json_input=$(cat)
command=$(echo "$json_input" | jq -r '.tool_input.command // empty')

if [ -z "$command" ]; then exit 0; fi

# Block destructive patterns
if echo "$command" | grep -qE '(rm\s+-rf\s+/|git\s+push\s+--force|DROP\s+TABLE)'; then
  echo "Blocked: destructive command detected" >&2
  exit 2
fi

exit 0
```

### 3. Stop Verification (TDD Enforcement)

When Claude thinks it's done, check that tests actually pass. If they fail, Claude gets told to keep going.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/verify-tests.sh"
          }
        ]
      }
    ]
  }
}
```

### 4. Intelligent Stop Hook (Prompt-Based)

Use an LLM to evaluate whether Claude should actually stop, based on whether the task is truly complete:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "You are evaluating whether Claude should stop working. Context: $ARGUMENTS\n\nAnalyze the conversation and determine if:\n1. All user-requested tasks are complete\n2. Any errors need to be addressed\n3. Follow-up work is needed\n\nRespond with JSON: {\"decision\": \"approve\" or \"block\", \"reason\": \"your explanation\"}",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### 5. Desktop Notifications

Get notified when Claude needs your attention so you can context-switch away during long tasks.

**macOS:**
```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code needs your attention\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

**Linux (notify-send):**
```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'Needs your attention'"
          }
        ]
      }
    ]
  }
}
```

### 6. Context Injection on Session Start

Inject project state, current branch, recent changes, or environment variables at the start of every session:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/inject-context.sh"
          }
        ]
      }
    ]
  }
}
```

Example `inject-context.sh`:
```bash
#!/bin/bash
branch=$(git branch --show-current 2>/dev/null || echo "unknown")
recent=$(git log --oneline -5 2>/dev/null || echo "no git history")

cat <<EOF
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Current branch: $branch\nRecent commits:\n$recent"
  }
}
EOF
exit 0
```

### 7. Secret Detection

Block writes that contain API keys, tokens, or passwords before they hit disk:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/detect-secrets.py"
          }
        ]
      }
    ]
  }
}
```

### 8. Auto-Approve Safe Operations

Skip permission dialogs for read-only operations on documentation files:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/auto-approve-docs.py"
          }
        ]
      }
    ]
  }
}
```

### 9. Persist Environment Variables

SessionStart hooks have access to `CLAUDE_ENV_FILE` for persisting environment variables across bash commands in the session:

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
  echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

### 10. GitButler Integration (Multi-Session Branch Isolation)

When running multiple Claude Code sessions, automatically isolate changes per session:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [{ "type": "command", "command": "but claude pre-tool" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [{ "type": "command", "command": "but claude post-tool" }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "but claude stop" }]
      }
    ]
  }
}
```

---

## Advanced: JSON Output Control

When exiting with code 0, hooks can output structured JSON to stdout for fine-grained control.

### PreToolUse Decision Control

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Documentation file auto-approved",
    "updatedInput": {
      "command": "npm run lint --fix"
    }
  }
}
```

- `"allow"` — bypasses permission system
- `"deny"` — prevents tool call, reason shown to Claude
- `"ask"` — prompts user to confirm
- `updatedInput` — modify tool parameters before execution (invisible to Claude)

### Stop/SubagentStop Decision Control

```json
{
  "decision": "block",
  "reason": "Tests have not been run yet. Please run the test suite before finishing."
}
```

### UserPromptSubmit Decision Control

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Current time: 2026-03-01T14:30:00Z. User is on branch: feature/auth-refactor."
  }
}
```

### Input Rewriting (PreToolUse)

Hooks can transparently modify tool inputs. The modifications are invisible to Claude. Use cases: auto-add `--dry-run` flags, redact secrets from commands, enforce commit message formatting.

---

## Working with MCP Tools

MCP tools follow the naming pattern `mcp__<server>__<tool>`. You can match on specific tools or entire servers:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__memory__.*",
        "hooks": [
          { "type": "command", "command": "echo 'Memory operation' >> ~/mcp-ops.log" }
        ]
      }
    ]
  }
}
```

---

## Environment Variables Available to Hooks

| Variable | Description |
|---|---|
| `CLAUDE_PROJECT_DIR` | Absolute path to project root |
| `CLAUDE_ENV_FILE` | File path for persisting env vars (SessionStart only) |
| `CLAUDE_CODE_REMOTE` | `"true"` if running in web/remote environment |
| `CLAUDE_TOOL_INPUT_FILE_PATH` | File path from tool input (PostToolUse with file tools) |

---

## Async Hooks

Hooks can run in the background without blocking:

```json
{
  "type": "command",
  "command": "node backup-script.js",
  "async": true,
  "timeout": 30
}
```

Useful for logging, notifications, backups — anything that shouldn't slow down Claude's agentic loop.

---

## Key Design Principles

**1. Start with three hooks.** Auto-format (PostToolUse), dangerous command blocking (PreToolUse), and desktop notifications (Notification). These cover the most ground for the least effort.

**2. Scope your matchers.** A hook on `"Bash"` fires on *every* bash command including `ls` and `pwd`. Use a smart dispatcher script that inspects the actual command before doing expensive work.

**3. Use prompt hooks for judgment, command hooks for rules.** If the decision is "does this command contain `rm -rf`?" — use a command. If the decision is "is this task actually complete?" — use a prompt.

**4. Keep hooks fast.** Default timeout is 60 seconds, but hooks block the agentic loop (unless async). Format-on-save should be sub-second. Move heavy validation behind smart dispatchers.

**5. Hooks don't replace CLAUDE.md.** CLAUDE.md tells Claude *how* to approach work (conventions, architecture decisions, coding style). Hooks enforce *what must happen* regardless of Claude's reasoning. They're complementary.

**6. Test hooks manually first.** Run your hook command with sample JSON piped to stdin before wiring it up. Use `claude --debug` for full execution details.

**7. Watch for infinite loops with Stop hooks.** The `stop_hook_active` field in Stop input is `true` when Claude is already continuing because of a previous stop hook. Check this to prevent Claude from running indefinitely.

---

## Security Considerations

Hooks run with your full user permissions. There is no sandbox.

- Validate and sanitize inputs — never trust input data blindly
- Always quote shell variables (`"$VAR"` not `$VAR`)
- Block path traversal — check for `..` in file paths
- Use absolute paths for scripts
- Skip sensitive files (`.env`, `.git/`, keys)
- For teams, use `allowManagedHooksOnly` to restrict to organization-approved hooks

Hook configuration is snapshotted at startup. Mid-session changes to settings files don't take effect until reviewed in the `/hooks` menu.

---

## Debugging

1. Run `/hooks` in Claude Code to verify hook registration
2. Toggle verbose mode with `Ctrl+O` to see hook output in the transcript
3. Run `claude --debug` for full execution details
4. Test hook commands manually: `echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./your-hook.sh`

**Common gotcha:** If your shell profile (`~/.zshrc`, `~/.bashrc`) has unconditional `echo` statements, their output gets prepended to your hook's JSON, causing parse errors. Wrap them in an interactive-shell guard:
```bash
if [[ $- == *i* ]]; then
  echo "Welcome back!"
fi
```

---

## Source Documentation

- **Official hooks reference:** https://code.claude.com/docs/en/hooks
- **Getting started guide:** https://code.claude.com/docs/en/hooks-guide
- **How Claude Code works:** https://code.claude.com/docs/en/how-claude-code-works
- **Extending Claude Code (hooks in context):** https://code.claude.com/docs/en/features-overview
- **Best practices:** https://code.claude.com/docs/en/best-practices
- **Plugins (distributing hooks):** https://code.claude.com/docs/en/plugins
- **Plugins reference (hook component spec):** https://code.claude.com/docs/en/plugins-reference
- **Settings files reference:** https://code.claude.com/docs/en/settings
- **GitButler hook integration:** https://docs.gitbutler.com/features/ai-integration/claude-code-hooks
