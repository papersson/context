# SFT Mechanics

How supervised fine-tuning works.

---

## Pretraining vs. SFT

| Aspect | Pretraining | SFT |
|--------|-------------|-----|
| Task | Predict next token at every position | Predict assistant reply given context |
| Loss | Cross-entropy on all tokens | Cross-entropy on assistant tokens only |
| Data | Raw text corpus | (instruction, response) pairs |

---

## How the Loss Mask Works

```
Serialized input:
[USER] What is 2+2? [ASSISTANT] The answer is 4.

Token positions:
[USER] What is 2+2? [ASSISTANT] The   answer is   4   .
  0     1    2  3  4     5       6      7     8   9   10

Loss mask:
  0     0    0  0  0     0       1      1     1   1   1
  ↑ user tokens (ignored)       ↑ assistant tokens (loss computed)
```

User tokens are masked by setting labels to `-100`, which PyTorch's `CrossEntropyLoss` ignores.

---

## Training Data Format

### Simple (Single-Turn)

```json
{
  "messages": [
    {"role": "user", "content": "Summarize: 'The capital of France is Paris.'"},
    {"role": "assistant", "content": "Paris is the capital of France."}
  ]
}
```

### With Tool Use (Multi-Turn)

```json
{
  "messages": [
    {"role": "user", "content": "What's the weather in NYC?"},
    {"role": "assistant", "content": null, "tool_calls": [
      {"id": "call_1", "function": {"name": "get_weather", "arguments": "{\"location\": \"NYC\"}"}}
    ]},
    {"role": "tool", "tool_call_id": "call_1", "content": "72F and sunny"},
    {"role": "assistant", "content": "It's 72°F and sunny in New York City."}
  ]
}
```

Loss is computed on assistant messages only (including tool_calls JSON).

---

## Framework Defaults

| Framework | Default Behavior | Config |
|-----------|------------------|--------|
| TRL SFTTrainer | Loss on completions only | `completion_only_loss=True` |
| Axolotl | Masks user prompts | `train_on_inputs: false` |
| HuggingFace Trainer | Loss on all tokens | Needs explicit masking |

---

## Prompt Loss Weight

Research (EMNLP 2024) suggests small non-zero loss on prompts can help:

| Task Type | Optimal Prompt Loss Weight |
|-----------|---------------------------|
| Multiple choice, short answers | 0.01 - 0.5 |
| Long generation | ~0 (ignore prompts) |

OpenAI previously supported `prompt_loss_weight` (default 0.01) but removed it.

---

## Converting Traces to SFT Data

```python
def trace_to_sft(trace: Trace, min_score: float = 0.8) -> dict | None:
    """Convert a successful trace to SFT format."""
    if trace.score < min_score:
        return None

    messages = [{"role": "user", "content": trace.task}]

    # Build assistant response from tool calls
    for step in trace.steps:
        # Add tool call
        messages.append({
            "role": "assistant",
            "tool_calls": [{"function": {"name": step.tool, "arguments": json.dumps(step.args)}}]
        })
        # Add tool result
        messages.append({
            "role": "tool",
            "content": step.result
        })

    # Add final response
    messages.append({
        "role": "assistant",
        "content": trace.final_response
    })

    return {"messages": messages}
```

---

## The Limit

SFT teaches imitation. The model learns: given this task, produce this sequence of actions. Given this tool result, respond this way.

It cannot exceed the quality of the training data. For that, you need RL.
