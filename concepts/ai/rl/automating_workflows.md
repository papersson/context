# Automating Business Workflows with Trainable Agents

I want to automate business workflows so an agent performs them autonomously, at human-level quality or better. This document is my attempt to understand how to get there.

An agent is a system, not a model. It's an LLM plus prompts plus tools plus orchestration. The LLM reasons. Prompts encode domain knowledge. Tools provide capabilities (file access, code execution, search). Orchestration ties it together. When I improve an agent, I can change any of these components. The question is which ones I can change.

---

## Frozen vs Trainable Core

If I use Claude via API, the model is frozen. I can improve prompts, add tools, refine orchestration. I cannot update the model's weights. Knowledge stays in the prompts. Behavior changes come from better instructions, not better understanding.

If I own a model (Llama, Mistral, Qwen), I can update its weights. Knowledge can move from prompts into the model. Behavior changes come from the model itself getting better at the task.

This is the trainable core distinction. An API agent has a frozen core. An owned agent has a trainable core.

Prompt optimization is powerful. Better prompts can add domain knowledge, structure complex tasks, provide examples that guide behavior, handle edge cases. But prompts have limits. Context windows bound how much knowledge fits. Instructions are interpreted, not internalized. The model's base capabilities don't improve. Each session starts from scratch.

The deeper issue: prompts change behavior within fixed capability. Training can change capability itself. A model trained on examples may discover statistical patterns that in-context examples don't enable. The training process finds structure that explicit instructions can't articulate.

For some workflows, the prompt ceiling is high enough. For others, it's not.

---

## The Path

The progression from assisted workflow to autonomous agent has four stages.

**Stage 1: Human + API Agent.** This is where I am now. Claude Code with my prompts and tools. Human in the loop for course correction, quality gates, ambiguity resolution, recovery. The agent does work; the human ensures quality.

**Stage 2: Prompt-Optimized API Agent.** Same architecture, better prompts. I run the workflow repeatedly, observe failures, improve prompts. I collect traces of successful executions. Over time, the agent handles more cases autonomously. The human intervenes less often. This is the ceiling of API-based agents. Still requires human oversight because the core hasn't learned.

**Stage 3: Training an Owned Model.** I take successful traces from Stage 2 and use them to train an owned model. The model learns to produce similar outputs given similar inputs. Knowledge moves from prompts into weights. The model doesn't need detailed prompts anymore. Runs faster (less context). Works offline. I control the deployment.

**Stage 4: RL on Owned Model.** I run the trained model in the environment and provide reward signals. The model explores beyond the demonstrations. It may discover strategies I didn't demonstrate. Whether this stage adds value depends on whether I have a reward signal that can guide the model beyond what I showed it.

### Training Methods

Three methods for Stage 3, each with different trade-offs.

**Supervised fine-tuning (SFT)** learns from demonstrations. I show the model good outputs and it learns to produce similar ones. The model optimizes similarity to training data. It can only generalize from what it saw; it doesn't explore.

**Reinforcement learning (RL)** learns from rewards. The model generates outputs, receives reward signals, and updates to maximize expected reward. It explores through sampling. Whether RL exceeds human performance depends on the reward signal. If reward means "tests pass," the model can find valid solutions I wouldn't have thought of.

If reward means "human approves," the situation is more nuanced. RLHF can produce outputs better than any individual demonstration: novel strategies that humans approve but didn't demonstrate. But it cannot produce something objectively better that humans would reject. The approval function is the ceiling. I can exceed my demos; I cannot exceed my taste.

**On-policy distillation** is a third option I didn't initially consider. The student generates a response, I compute KL divergence to what a teacher model would produce, and update the student to minimize the gap. No reward function needed. The only supervision is "be more like the teacher."

On-policy distillation dramatically outperforms SFT in practice. On AIME'24 math problems, SFT on reasoning data reaches 55% accuracy after 3000 training steps. On-policy distillation reaches 65% accuracy after just 100 steps. The key difference: SFT trains on a fixed dataset. On-policy distillation trains on what the student actually produces, correcting it toward the teacher. The student learns from its own distribution.

### Prompt Distillation

A specific technique worth understanding: prompt distillation (also called context distillation). The goal is to make a model behave as if it has a complex prompt, without providing the prompt.

The process: a teacher generates with the full prompt, a student learns to produce the same output without the prompt. After training, the student behaves as if the prompt is baked into its weights.

My CLAUDE.md files, system prompts, and tool instructions are all prompts. Prompt distillation can internalize these into an owned model. Run workflows with full prompts (teacher), collect query-response pairs, train student to produce responses given only queries. The student now "knows" the prompt implicitly.

This is a concrete implementation of "knowledge moves from prompts to weights."

### When to Use What

SFT when I have good demonstrations and want a simple, predictable training process. On-policy distillation when I have a teacher model and want faster convergence without designing rewards. RL when I have verifiable reward signals (tests pass, metrics improve) and want the model to explore beyond my demonstrations.

The question isn't "SFT or RL?" It's "what signal can I give, and what does optimizing it buy me?"

---

## The Evaluation Problem

My workflows have no clear automated success signal. Quality requires human judgment. This creates tension. Prompt optimization needs a score to compare variants. SFT needs labeled examples. RL needs a reward signal for every rollout.

Several options exist, none perfect.

Human-in-the-loop evaluation is accurate but doesn't scale. I review outputs and provide scores. Works for prompt optimization and SFT data collection. For RL, this means RLHF (reinforcement learning from human feedback).

LLM-as-judge scales better. Another model evaluates outputs against criteria. But the judge's biases become my agent's biases. If the judge can't evaluate correctly, neither can the trained agent.

Proxy metrics (tests pass, linter clean, type-checks, code compiles) are verifiable but incomplete. They work as a floor: necessary but not sufficient for quality.

Decomposition helps. I break "good workflow execution" into evaluable sub-tasks. Some have clear signals (code compiles), others need judgment (solution is elegant). Evaluate what I can, sample-review the rest.

Rubric-based grading structures LLM-as-judge with explicit rubrics. Each rubric item specifies what to check, how to format the output, how to extract the score. A grader LLM scores each item. Sum of scores equals reward. This constrains the grader to specific, answerable questions rather than open-ended evaluation.

### Tacit Judgment

A harder problem: I know good output when I see it, but can't articulate what makes it good. Classic tacit knowledge problem.

Pairwise comparison bypasses explicit criteria. I don't define "good." I just repeatedly judge "A is better than B." The model learns my latent quality function from my choices, not from rules I write. This is how RLHF works: Bradley-Terry models over preference pairs.

Example curation also works. I carefully select the best examples for training, even if I can't explain why they're best. The quality function is implicit in the selection.

Both approaches transfer judgment without articulation. But both still bound the model by my approval. It learns to replicate my choices, not to exceed them.

### Self-Play as Alternative

When I can't design a reward function, self-play offers an alternative. Let agents compete. Use game outcomes as rewards.

Two agents play against each other (or one agent plays both sides). Reward equals win/loss from game rules. Both winning and losing trajectories train the model. No hand-crafted reward function needed.

In Twenty Questions, the agent asks yes/no questions to guess a hidden word. Another LLM answers. Reward is 1 if guessed correctly, 0 otherwise. The agent learns to ask better questions without anyone defining what makes a question "good."

In self-play tic-tac-toe, the agent plays against itself. A coordinator synchronizes two environment objects, passing moves between them. Both perspectives train the same model. Training improves reward from -1.0 (random play loses) to positive in about 40 steps.

Self-play applies when the task can be framed as a game with clear win/loss, when I can simulate the opponent, and when game outcome correlates with the quality I care about.

### My Current Approach

I don't have this solved, but three strategies seem to compose well for my situation (verifiable correctness plus tacit quality judgment).

First, decompose rewards. Use RL on verifiable metrics: tests pass, builds succeed, type-checks clean. Keep human judgment for taste and quality. Don't try to automate what I can't evaluate.

Second, expand coverage. RL explores the solution space I wouldn't manually search. The value is finding valid solutions I wouldn't have thought of, even if I still judge quality. The model can exceed my demos on correctness while I remain the arbiter of taste.

Third, refine judgment over time. Exposure to agent outputs may improve my own taste, gradually raising the ceiling. But this is risky. Approval drift can go toward better taste or toward accepting what the model easily produces. I need to monitor for the difference.

In practice: proxy metrics as baseline, LLM-judge with rubrics for style, human review on a sample for calibration, pairwise comparison where I can't articulate criteria.

---

## Getting Started: Trace Collection

The first actionable step is trace collection. Every time I run my workflow with Claude Code, I can capture the task, each tool call and its result, the final output, and whether it succeeded.

Why traces specifically? Three reasons.

First, traces are model-agnostic. The same traces work for SFT, RLHF, prompt analysis, or evaluation design. I defer the hard decisions (which model, which training method) while accumulating valuable data.

Second, traces capture tacit judgment implicitly. My successful traces are my quality function in action. Even if I can't articulate what makes output good, traces of good outputs encode it.

Third, traces are a maximally flexible starting point. I can collect now and decide later how to use them. The cost of collecting is low; the optionality is high.

### Implementation

Build a custom executor using the Anthropic SDK. Claude Code's tools are straightforward to replicate: `read_file` maps to `Path.read_text()`, `write_file` to `Path.write_text()`, `bash` to `subprocess.run()`, and so on.

The executor runs the agent loop and logs every step:

```python
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
    score: float | None

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
            return Trace(
                id=str(uuid.uuid4()),
                task=task,
                system_prompt=system_prompt,
                steps=steps,
                final_response=extract_text(response),
                score=None,
            )

        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                steps.append(ToolCall(
                    tool=block.name,
                    args=block.input,
                    result=result[:2000],
                    timestamp=datetime.now().isoformat(),
                ))

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
```

A saved trace looks like this:

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "task": "Fix the bug in auth.py where login fails for emails with + symbols",
  "system_prompt": "You are an expert Python developer...",
  "steps": [
    {
      "tool": "read_file",
      "args": {"path": "auth.py"},
      "result": "def login(email, password):\n    ...",
      "timestamp": "2025-01-06T10:30:00"
    },
    {
      "tool": "write_file",
      "args": {"path": "auth.py", "content": "...fixed code..."},
      "result": "ok",
      "timestamp": "2025-01-06T10:30:05"
    },
    {
      "tool": "bash",
      "args": {"command": "pytest test_auth.py"},
      "result": "...... 6 passed",
      "timestamp": "2025-01-06T10:30:10"
    }
  ],
  "final_response": "Fixed the bug by URL-encoding the email before validation.",
  "score": null
}
```

The score starts null. I fill it in after reviewing the output, or leave it for batch evaluation later.

### Converting Traces to Training Data

For SFT, filter to successful traces and convert to chat format. Each tool call becomes an assistant message with tool_calls, followed by a tool message with the result. The final response becomes the last assistant message.

SFT trains on assistant outputs only. User messages and tool results are context, not targets. Most frameworks (TRL, Axolotl) handle loss masking automatically. The model learns to produce assistant tokens given context, not to predict the context itself.

I don't need fine-tuning infrastructure yet. I need traces. Start collecting now; decide how to use them later.

---

## What to Expect

Concrete numbers from production systems (Tinker cookbook benchmarks).

Verifiable tasks with clear rewards converge fast. Arithmetic goes from 66% to 100% accuracy in about 10 steps. GSM8K math word problems reach 91% accuracy in 220 steps. LiveCodeBench coding improves from 33.8% to 42.7% pass@1 in 100 steps.

Tool use and multi-turn behaviors emerge quickly. Search-R1 (multi-hop QA with search tools) learns multi-turn behavior in 10-25 steps and reaches 51.8% accuracy on Natural Questions.

Self-play games show clear improvement. Guess the Number improves from 40% to over 50% success in 20 steps. Tic-tac-toe goes from -1.0 reward (random play loses) to positive in 40 steps.

Preference learning works. RLHF improves win rate from 40% to 70% in 100 steps. Training for shorter responses shows significant token count reduction in 40 steps.

Distillation is efficient. SFT on reasoning data reaches 55% on AIME'24 in 3000 steps. On-policy distillation reaches 65% in just 100 steps.

Key patterns: simple verifiable tasks converge in 10-20 steps, complex reasoning takes 100-200+ steps, multi-turn behaviors emerge in 10-25 steps, preference learning shows clear improvement in 40-100 steps, on-policy distillation is dramatically more efficient than SFT.

---

## Open Questions

What I haven't figured out yet.

**Evaluation design.** How do I approximate quality automatically? What's the right mix of proxy metrics, LLM-judge, and human review? Partial answer: decompose into verifiable versus judgment, use pairwise comparison for tacit parts.

**The end state.** Three possibilities: (1) Autonomous with spot-checks, where the agent runs without intervention and I review a sample for quality assurance. (2) Autonomous with guardrails, where the agent escalates when uncertain and quality is maintained because it knows what it doesn't know. (3) Continuously improving, where I provide occasional feedback and the model gets updated periodically. I haven't decided which I want. Probably some version of the first or second. My approval function is the ceiling regardless.

**When RL is worth it.** SFT on good traces might be enough. RL is more complex and needs reward engineering. When does the added complexity pay off? Partial answer: when I have verifiable rewards that let the model explore beyond my demos.

**Model selection timing.** Do I pick a model now and start building around it? Or stay model-agnostic until I have enough traces to run experiments? Answer: stay agnostic. Trace collection is model-agnostic; traces are the asset.

**Approval drift.** If my taste improves via exposure to agent outputs, that's good. If it drifts toward accepting what the model easily produces, that's bad. How do I distinguish these?

---

## References

### Libraries and Tools

**Tinker** (Thinking Machines Lab) is a distributed training SDK for fine-tuning LLMs. It separates the training plane from the sampling plane, enabling parallel rollout collection. The cookbook includes complete recipes for SFT, RL, RLHF, distillation, and self-play. https://thinkingmachines.ai/tinker/

**Verifiers / Environments Hub** (Prime Intellect) is a community repository of RL environments for LLMs. Install environments with `prime env install user/env-id`. https://app.primeintellect.ai/dashboard/environments

**Sandbox Fusion** (ByteDance) provides safe code execution for RL training via Docker-based sandboxing. https://github.com/bytedance/SandboxFusion

**TRL** (Hugging Face) is a transformer reinforcement learning library with SFTTrainer, PPOTrainer, and DPOTrainer. https://github.com/huggingface/trl

**Axolotl** is a fine-tuning framework with automatic loss masking, supporting LoRA, QLoRA, and full fine-tuning. https://github.com/OpenAccess-AI-Collective/axolotl

### Key Papers

Prompt and context distillation: Askell et al. (2021), "A General Language Assistant as a Laboratory for Alignment," arXiv:2112.00861. Snell et al. (2022), "Learning by Distilling Context," arXiv:2209.15189.

RLHF: Ouyang et al. (2022), "Training Language Models to Follow Instructions with Human Feedback," arXiv:2203.02155. Bai et al. (2022), "Training a Helpful and Harmless Assistant with RLHF," arXiv:2204.05862.

Direct Preference Optimization: Rafailov et al. (2023), "Direct Preference Optimization: Your Language Model is Secretly a Reward Model," arXiv:2305.18290.

GRPO (group relative policy optimization): DeepSeekMath (2024), arXiv:2402.03300. The key technique is centering advantages within groups, not globally.

Agentic RL: Search-R1 (2025), "Training LLMs to Reason and Leverage Search Engines with RL," arXiv:2503.09516.

Multi-agent and self-play: TextArena (2025), arXiv:2504.11442.

### Blog Posts

Thinking Machines Lab, "On-Policy Distillation," https://thinkingmachines.ai/blog/on-policy-distillation
