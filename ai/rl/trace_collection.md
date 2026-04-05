# Trace Collection

Capturing execution traces from Claude Code sessions.

---

## The Options

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| Claude Code Hooks | Shell commands on events | No code changes | Limited data, fragile |
| API Proxy | Intercept Anthropic calls | Works with unmodified CC | SSL complexity |
| Fork Claude Code | Add logging to source | Full functionality | Maintenance burden |
| Custom Executor | Build with Anthropic SDK | Full control, clean | Implement tools yourself |

Custom executor is the recommendation. You get exactly what you need in about 80 lines.

---

## Why Custom Executor

Full control over what data you capture. Simple to implement (tools are trivial). No dependencies beyond Anthropic SDK. Structured JSON from the start. Headless, easy to run in automated loops.

Claude Code's tools are straightforward:
- `read_file` is `Path.read_text()`
- `write_file` is `Path.write_text()`
- `bash` is `subprocess.run()`
- `search` is `grep`/`ripgrep` subprocess
- `glob` is `pathlib.glob()`

---

## The Code

```python
import anthropic
import subprocess
from pathlib import Path
from dataclasses import dataclass, asdict
from datetime import datetime
import json
import uuid

@dataclass
class ToolCall:
    tool: str
    args: dict
    result: str
    timestamp: str

@dataclass
class Trace:
    id: str
    task: str
    system_prompt: str
    steps: list[ToolCall]
    final_response: str

TOOLS = [
    {
        "name": "read_file",
        "description": "Read a file from disk",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "bash",
        "description": "Run a bash command",
        "input_schema": {
            "type": "object",
            "properties": {"command": {"type": "string"}},
            "required": ["command"]
        }
    },
    {
        "name": "list_dir",
        "description": "List directory contents",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    },
]

def execute_tool(name: str, args: dict) -> str:
    try:
        if name == "read_file":
            return Path(args["path"]).read_text()
        elif name == "write_file":
            Path(args["path"]).parent.mkdir(parents=True, exist_ok=True)
            Path(args["path"]).write_text(args["content"])
            return "ok"
        elif name == "bash":
            r = subprocess.run(
                args["command"],
                shell=True,
                capture_output=True,
                timeout=30
            )
            return r.stdout.decode() + r.stderr.decode()
        elif name == "list_dir":
            return "\n".join(str(p) for p in Path(args["path"]).iterdir())
        else:
            return f"Unknown tool: {name}"
    except Exception as e:
        return f"Error: {e}"

def run_with_trace(task: str, system_prompt: str) -> Trace:
    client = anthropic.Anthropic()
    messages = [{"role": "user", "content": task}]
    steps = []

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=system_prompt,
            tools=TOOLS,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            final = next(
                (b.text for b in response.content if hasattr(b, "text")),
                ""
            )
            return Trace(
                id=str(uuid.uuid4()),
                task=task,
                system_prompt=system_prompt,
                steps=steps,
                final_response=final,
            )

        # Collect tool results
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                steps.append(ToolCall(
                    tool=block.name,
                    args=block.input,
                    result=result[:2000],  # Truncate for storage
                    timestamp=datetime.now().isoformat(),
                ))
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

def save_trace(trace: Trace, trace_dir: str = "traces") -> Path:
    Path(trace_dir).mkdir(exist_ok=True)
    path = Path(trace_dir) / f"{trace.id}.json"
    path.write_text(json.dumps(asdict(trace), indent=2, default=str))
    return path
```

---

## Usage

```python
# Load your markdown prompts
system_prompt = Path("CLAUDE.md").read_text()

# Run workflow with tracing
trace = run_with_trace(
    task="Fix the bug in auth.py where login fails for emails with + symbols",
    system_prompt=system_prompt,
)

# Save trace
path = save_trace(trace)
print(f"Saved: {path}")
```

---

## Trace Format

```json
{
  "id": "a1b2c3d4-...",
  "task": "Fix the bug in auth.py...",
  "system_prompt": "You are an expert...",
  "steps": [
    {
      "tool": "read_file",
      "args": {"path": "auth.py"},
      "result": "def login(email, password):...",
      "timestamp": "2025-01-06T10:30:00"
    },
    {
      "tool": "write_file",
      "args": {"path": "auth.py", "content": "..."},
      "result": "ok",
      "timestamp": "2025-01-06T10:30:05"
    },
    {
      "tool": "bash",
      "args": {"command": "pytest test_auth.py"},
      "result": "...... OK",
      "timestamp": "2025-01-06T10:30:10"
    }
  ],
  "final_response": "Fixed the bug by..."
}
```

---

## Adding More Tools

```python
# Search with ripgrep
{
    "name": "search",
    "description": "Search for pattern in files",
    "input_schema": {
        "type": "object",
        "properties": {
            "pattern": {"type": "string"},
            "path": {"type": "string", "default": "."}
        },
        "required": ["pattern"]
    }
}

def execute_tool(name: str, args: dict) -> str:
    # ... existing tools ...
    elif name == "search":
        r = subprocess.run(
            ["rg", "--json", args["pattern"], args.get("path", ".")],
            capture_output=True,
            timeout=30
        )
        return r.stdout.decode()
```

---

## Integration with Optimization Loop

```python
def optimization_step(task: str, current_markdown: str) -> tuple[Trace, float]:
    trace = run_with_trace(task, current_markdown)
    score = evaluate(trace)  # Your evaluator
    save_trace(trace)
    return trace, score
```
