# The Optimization Spectrum

How prompt optimization, SFT, and RL relate.

---

## The Spectrum

```
OPTIMIZATION METHODS
        │
        ├─────────────────────────────────────────────────────────────────────┐
        │                                                                     │
   Black-Box                    Gradient-Free                        Gradient-Based
   (no structure)               (structured search)                  (differentiable)
        │                             │                                    │
   Random search               Evolutionary                          Policy gradients
   Grid search                 CMA-ES                                PPO, GRPO
                               GEPA (Pareto)                         Actor-Critic
                               TextGrad (LLM feedback)               DPO (implicit)
                                     │                                    │
                                     │                                    │
                               ┌─────┴─────┐                        ┌─────┴─────┐
                               │           │                        │           │
                          Prompt Opt    DSPy                       RL         SFT
                          (your loop)   (prompts)                  (weights)  (weights)
```

---

## Prompt Optimization Is Not RL

| Aspect | Prompt Optimization | RL |
|--------|--------------------|----|
| What's optimized | Text (markdown, prompts) | Model weights |
| Optimization signal | Score comparison | Policy gradients |
| Credit assignment | None (whole-trajectory) | Per-token or per-step |
| Exploration | Mutation | Entropy bonus, noise |
| Can exceed demonstrations? | No | Yes |

Prompt optimization is black-box search over text: mutation and selection with no gradients. RL uses gradients to update a parameterized policy.

---

## Your Workflow Is an Environment

If you can execute it in Claude Code, you have:

| Component | Your Workflow |
|-----------|---------------|
| Initial state | Task / ticket / spec |
| Actions | Tool calls (edit, run, search, bash) |
| Transitions | Tool outputs, file changes |
| Terminal | Verification passes or fails |
| Reward | Your evaluator (verifier + LLM judge) |

This is an RL environment. What you do with it determines the optimization method.

---

## The Progression

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PROMPT OPTIMIZATION                                                    │
│                                                                         │
│  Optimize: Markdown files                                               │
│  Method: Reflect + Mutate (LLM-guided search)                          │
│  Output: Better prompts + successful rollouts                          │
│                                                                         │
│  Stop here if: Claude API is fine, you want interpretable artifacts    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Accumulated rollouts
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  SFT DISTILLATION                                                       │
│                                                                         │
│  Optimize: Model weights                                                │
│  Method: Supervised learning on successful trajectories                 │
│  Output: Model that doesn't need prompts                               │
│                                                                         │
│  Stop here if: You want to own the model, run locally, reduce latency  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ SFT model + environment
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  RL REFINEMENT                                                          │
│                                                                         │
│  Optimize: Model weights via policy gradients                          │
│  Method: GRPO/PPO on environment rewards                               │
│  Output: Model that discovers strategies beyond demonstrations          │
│                                                                         │
│  Go here if: You want to exceed human/prompt performance               │
└─────────────────────────────────────────────────────────────────────────┘
```

Each stage is optional. Prompt optimization alone is complete and valuable.

---

## The Automated Loop

```python
while not converged:
    task = sample_from_task_distribution()
    rollout = execute(claude_code, task, current_markdown)
    score = evaluate(rollout)  # verifier + LLM judge

    # Always reflect, even on success (success is a spectrum)
    reflection = reflect(rollout, score)
    proposed_edit = mutate(current_markdown, reflection)

    # Test the mutation
    new_score = evaluate(execute(claude_code, task, proposed_edit))

    if new_score > score:
        current_markdown = proposed_edit

    store_rollout(rollout, score)  # For future SFT
```

---

## Convergence

The loop stops when:

```python
def converged(history, threshold=0.95, window=50):
    recent = history[-window:]
    return (
        mean(recent) > threshold and
        std(recent) < 0.05 and
        no_new_failure_modes(recent)
    )
```

---

## What Stays Constant

| Component | Same Across All Stages |
|-----------|------------------------|
| Environment | Your workflow in Claude Code |
| Task distribution | Your real/synthetic tasks |
| Evaluator | Your verifier + judge |
| Success criterion | Your definition of "done well" |

The workflow is the foundation. Everything else is how you use it.

---

## When to Use Each Stage

| Goal | Stage |
|------|-------|
| Claude API does my workflow better | Prompt Optimization |
| Own a model, no API dependency | + SFT |
| Model discovers new strategies | + RL |
| Fastest iteration, interpretable | Stay at Prompt Optimization |
| Production deployment at scale | Go to SFT minimum |
