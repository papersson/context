# RL Taxonomy (2025)

A reference card for the modern RL landscape.

---

## The Landscape

```
                              RL Algorithms
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
    Model-Free                 Model-Based                 Offline RL
         │                          │                          │
   ┌─────┴─────┐             ┌──────┴──────┐            ┌──────┴──────┐
   │           │             │             │            │             │
 Policy    Q-Learning    Known Model  Learned Model  Sequence    Conservative
   │           │             │             │          Modeling    Q-Learning
   │           │             │             │             │             │
  PPO        DQN        AlphaZero      MuZero          DT           CQL
  GRPO      SAC                        Dreamer         TT           IQL
  TRPO      TD3                        TD-MPC2                     AWAC
  A3C     Rainbow
```

---

## Algorithm Families

| Family | What It Optimizes | Key Algorithms | When to Use |
|--------|-------------------|----------------|-------------|
| Policy Gradient | Policy directly | PPO, TRPO, REINFORCE | Continuous actions, stable training |
| Q-Learning | Value function | DQN, SAC, TD3 | Discrete actions, sample efficiency |
| Model-Based | World model + planning | MuZero, Dreamer, TD-MPC2 | When you can simulate or learn dynamics |
| Offline | Fixed dataset | CQL, IQL, DT | No environment access, only logs |
| Preference | Human/AI preferences | DPO, KTO, IPO | Alignment, no explicit reward |

---

## MuZero Family

| Algorithm | Innovation | Paper |
|-----------|------------|-------|
| MuZero | Learns dynamics in latent space | Schrittwieser 2020 |
| EfficientZero | Sample-efficient (100x less data) | Ye 2021 |
| Gumbel MuZero | Better exploration, fewer simulations | Danihelka 2022 |
| Sampled MuZero | Large/continuous action spaces | Hubert 2021 |
| Stochastic MuZero | Stochastic environments | Antonoglou 2022 |

---

## LLM-Specific RL

| Method | What It Does | Needs Critic? | Key Feature |
|--------|--------------|---------------|-------------|
| PPO | Policy gradient with clipping | Yes | Stable, standard |
| GRPO | Group-normalized advantages | No | No value network, simpler |
| REINFORCE++ | Batch-normalized baseline | No | Simple baseline |
| DPO | Direct preference optimization | No | No RL loop, offline |
| KTO | Kahneman-Tversky optimization | No | Works with binary feedback |
| IPO | Identity preference optimization | No | Theoretical improvements to DPO |

---

## Prompt Optimization Methods

| Method | Approach | Key Idea |
|--------|----------|----------|
| TextGrad | LLM feedback as "gradients" | Backprop error messages through computation graph |
| GEPA | Pareto evolutionary | Maintain diverse frontier of prompt variants |
| DSPy/MIPROv2 | Bayesian optimization | Surrogate model + acquisition function |
| OPRO | LLM-as-optimizer | LLM proposes candidate prompts |
| EvoPrompt | Genetic algorithms | Crossover + mutation on prompt population |
| RLPrompt | Actual RL on tokens | PPO on prompt token generation |

---

## The Three Levels

| Level | What's Optimized | Method | Output |
|-------|------------------|--------|--------|
| Weight Optimization | Model parameters θ | RL (PPO, GRPO, DPO) | Fine-tuned model |
| Prompt Optimization | Text artifacts | Black-box search | Better prompts |
| In-Context Learning | Nothing persistent | Few-shot examples | Single-session behavior |

---

## Key Papers

| Topic | Paper | Year |
|-------|-------|------|
| MuZero | "Mastering Atari, Go, Chess and Shogi by Planning with a Learned Model" | 2020 |
| GRPO | DeepSeekMath | 2024 |
| DPO | "Direct Preference Optimization" | 2023 |
| TextGrad | "TextGrad: Automatic Differentiation via Text" | 2024 |
| GEPA | "Reflective Prompt Evolution Can Outperform RL" | 2025 |
