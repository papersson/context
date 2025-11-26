
![[Pasted image 20251123200256.png]]
# Introduction to Reinforcement Learning for LLM Agents

Reinforcement Learning (RL) in the context of LLM agents is often mystified, but it is fundamentally **Evaluation on a Loop**.

If you have a robust evaluation harness—a set of test cases and a scoring rubric—you have already built 90% of an RL environment. The shift from "Evaluation" to "Optimization" is simple:
*   **Evaluation**: Measures current performance ($Score$).
*   **RL**: Uses that $Score$, the **Execution Trace**, and **Negative Feedback** to systematically update the system.

To understand how this works, we map the components of a standard agent system onto the rigorous formalism of a **Markov Decision Process (MDP)**.

---

### Running Example: The SQL Detective
To ground these definitions, we will reference **Project 3: The SQL Detective**.
*   **Task**: Input a Natural Language Question + Database Schema.
*   **Goal**: Output a SQL Query that executes to return the correct Row Set.

---

## 1. The Core Abstraction: The MDP

An agent task is a trajectory through a state space. We define the tuple $(S, A, P, R, \tau)$.

### State ($S$)
The full context available to the model at step $t$.
*   **Definition**: The System Prompt, the Current Code, and the **History of Errors**.
*   **In Example**: The State includes the untyped code and the `mypy` error log from the previous attempt (e.g., "Circular import detected").

### Action ($A$)
The model's output. In agent systems, actions are **generative** (writing code) or **operative** (calling a tool).
*   **In Example**: The agent generates a revised version of the code with new `TYPE_CHECKING` blocks.

### Transition ($P$)
The "World Model." When the agent takes action $a$, how does the state change?
*   **Definition**: In software tasks, this is the **Compiler** or **Interpreter**. It is deterministic but complex.
*   **In Example**: We run `mypy`. The state transitions from "Code with potential bugs" to "Code with specific Error List X".

### Reward ($R$)
The scalar signal indicating utility.
*   **In Example**: A score derived from the error count.
    *   +1.0 if `Success: no issues found`.
    *   -0.1 for every remaining error.
    *   -1.0 if the code crashes (SyntaxError).

### The Trajectory ($\tau$)
The complete record of the episode.
*   **Definition**: The sequence of `(Prompt -> Code -> Error -> Reflection -> New Prompt)`.
*   **Importance**: This trajectory contains the **Negative Constraints** (what *didn't* work) that are invisible in a static dataset.

## 2. The Environment (The Scaffolded Sandbox)

In classical RL, the environment is a game engine. In LLM-RL, the Environment is an **Execution Sandbox** responsible for **Isolation** and **Verification**.

*   **The Scaffold**: The Environment does not just "load text." It creates a **fresh, isolated context** (e.g., a temporary directory with the full library source or a clean SQLite database) for every episode.
*   **The Abstraction**: The Environment acts as the interface between the generic Optimizer and the specific Task.
    *   *User Responsibility:* Define `reset()` (load scenario) and `step()` (execute & score).
    *   *Library Responsibility:* The Optimizer calls `step()`, ignorant of the underlying logic.
*   **The Mechanism**:
    1.  `reset()`: Create temp dir, copy assets (Schema/Codebase), return Observation.
    2.  `step(action)`: Execute SQL/Code in sandbox, compute Reward, return `(Trace, Reward, Done)`.

## 3. The Policy ($\pi$)

The Policy is the function mapping State to Action: $a = \pi(s)$. For commercial LLMs, the policy has two parts:

1.  **Fixed Weights ($\theta$)**: The pre-trained model (e.g., Claude 3.5, Gemini Pro). We cannot update these.
2.  **Mutable Context ($\phi$)**: The **System Prompt**, Few-Shot Examples, and Tool Definitions.

**Optimization Definition**: "RL for LLMs" means optimizing $\phi$ (The Prompt) to maximize expected Reward, using the fixed $\theta$ as the engine.

## 4. Optimization Mechanisms

How do we use the Reward to update the Policy? There are two competing paradigms.

### A. One-Shot Meta-Prompting ("Compilation")
*   **Method**: Feed a batch of `{Input, GroundTruth, Error}` examples into a massive context window and ask the LLM to "Compile" a perfect system prompt.
*   **Limit**: It relies on **Pattern Recognition**. It can only learn explicit constraints present in the positive examples.
*   **Failure Mode**: It cannot learn **Implicit Negative Constraints**. For `mypy`, it sees the correct code but doesn't know *why* it is correct (e.g., it doesn't know that adding `X` would have caused a circular import).

### B. Iterative Refinement ("RL Loop")
*   **Method**: The Agent tries to solve a problem, fails, receives a **Compiler Gradient** (Error Message), reflects, and updates the prompt.
*   **Advantage**: It performs **Causal Reasoning**. By hitting the wall (Error), it learns where the wall is.

### C. The Search Distinction (Architecture-Task Fit)
We must distinguish between two types of "Optimization":

1.  **Training-Time Search (Prompt Optimization)**:
    *   *Target*: The System Prompt (The Policy).
    *   *Scope*: Happens *Offline*.
    *   *Suitability*: **Translation Tasks** (e.g., Text-to-SQL with Schema). Tasks where the mapping from Input $\to$ Output is direct, provided the instructions are clear.
2.  **Inference-Time Search (Agent Loop)**:
    *   *Target*: The Trajectory (The Action Sequence).
    *   *Scope*: Happens *Online* (Runtime).
    *   *Suitability*: **Search Tasks** (e.g., Typing with Dependencies). Tasks with **Hidden State** (like circular imports) that cannot be known without taking an exploratory action (running the compiler).

**Verdict**: You cannot Prompt-Optimize your way out of a problem that requires Inference-Time Search.

## 5. The Data Strategy: Curriculum Learning

Optimization fails if the reward signal is binary (0% or 100%). This creates a "Wall" with no gradient.
To succeed, we must manufacture a **Slope** using **Synthetic Data Mutation**.

*   **Near-Miss Mutations**: We algorithmically mutate valid examples to create "almost correct" problems (e.g., swapping `>` for `>=` in SQL).
*   **The Validation Protocol**: We only accept synthetic examples where a baseline model has a **40-80% Pass Rate**.
    *   *<40%*: Too hard (The Wall).
    *   *>80%*: Too easy (No signal).

## 6. The Optimization Loop (Trace-Aware)

The Optimizer is **Data-Agnostic** but **Trace-Aware**. It does not see the raw dataset; it sees the **Feedback** emitted by the Environment.

1.  **Generate**: Agent produces `Action` (SQL/Code).
2.  **Evaluate**: Environment executes action in Sandbox.
    *   *Returns*: `Execution Feedback` (Binary Score) + `Linguistic Feedback` (Stderr/Diff).
3.  **Reflect**: "Reflector" LLM reads the `Linguistic Feedback`.
    *   *Diagnosis*: "The SQL failed with 'Ambiguous Column'. The model guessed column names."
4.  **Mutate**: Optimizer updates the System Prompt based on Diagnosis.
5.  **Select**: Keep changes that improve the score on the "Near-Miss" Validation Set.

---

### Key References

**Primary Papers**
*   **GEPA**: Agrawal et al. (2025). *Reflective Prompt Evolution Can Outperform Reinforcement Learning.* (Establishes the Reflect/Mutate loop).
*   **TextGrad**: Yuksekgonul et al. (2024). *TextGrad: Automatic Differentiation via Text.* (Treats error messages as gradients for backpropagation).
*   **DeepSeek-R1**: DeepSeek-AI (2025). (Demonstrates that rigorous enforcement of logic rewards leads to reasoning emergence).
*   **Reflexion**: Shinn et al. (2023). (Foundational paper on using verbal feedback loops).

**Frameworks & Standards**
*   **Verifiers**: A library for managing RL environments and rubrics for LLMs.
*   **DSPy (MIPROv2)**: A framework for optimizing LM calls and prompts as differentiable programs.
*   **OpenTelemetry / LangSmith**: Industry standards for capturing the hierarchical execution traces essential for attribution.

**Concepts**
*   **RLVR (Reinforcement Learning from Verifiable Rewards)**: Using deterministic checks (Compilers) rather than human labels.
*   **Constitutional Data Engineering**: The process of harvesting and synthesizing high-quality evaluation datasets.
