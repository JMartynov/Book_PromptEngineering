# Chapter 13: Prompt Optimization Algorithms

## Introduction: The "Search" for the Perfect Prompt

In traditional prompt engineering, finding a better prompt is a manual, intuitive process. You "try things" and see what happens. In 2026, we view prompt engineering as a **Search Problem**.

The "Prompt Space" is the set of all possible ways to word an instruction and choose few-shot examples. Instead of a human wandering this space, we use **Prompt Optimization Algorithms** (called **Teleprompters** in DSPy) to systematically search for the "Global Maximum"—the prompt configuration that yields the highest possible score on our Golden Dataset.

---

## Deep Technical Analysis: The Optimizer Landscape

The DSPy framework provides a hierarchy of optimizers, each suited for different data sizes and compute budgets:

### 1. BootstrapFewShot (The "Greedy" Inductive Learner)
**How it works:** It takes a few examples and attempts to "Bootstrap" intermediate labels (like reasoning chains) for them. It then selects the subset of these examples that, when used as few-shot demonstrations, maximize the program's accuracy.
**Technical Insight:** This is an **Inductive** process. It doesn't rewrite the instructions; it optimizes the *demonstrations*.

### 2. MIPROv2 (Multi-objective Instruction-Proposal Optimizer)
**How it works:** This is the flagship 2026 optimizer. It uses a Bayesian optimization loop to:
1.  **Propose** 10-20 different instruction variations using a "Teacher" LLM.
2.  **Select** the best combination of instructions and few-shot examples.
3.  **Optimize** across multiple objectives (e.g., accuracy AND token cost).
**Technical Insight:** It uses a surrogate model (often a Random Forest or Gaussian Process) to predict which prompt variations will perform best without having to run every single one.

### 3. COPRO (Chain-of-Thought PRompt Optimizer)
**How it works:** Specifically designed for reasoning tasks. It iteratively refines the "Thinking Steps" in a Chain-of-Thought prompt by analyzing model failures and "proposing" fixes to the reasoning logic.

---

## Why Algorithms Solve Real-World Problems

In practice, Prompt Optimization Algorithms solve several critical production issues:
-   **Eliminating Human Bias:** Humans tend to use "adjectives" (be concise, be smart). Optimizers use "data-driven patterns" that might be counter-intuitive to humans but highly effective for LLMs.
-   **Handling Interaction Effects:** A prompt change that fixes "Edge Case A" might break "General Case B." Optimizers evaluate the *entire dataset* on every iteration, ensuring that improvements are global, not local.
-   **Automatic Adapting to Models:** Llama-3-8B needs different instructions than GPT-4o. Optimizers allow you to "Compile" the same logic for two different models, finding the unique "Global Max" for each.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to use DSPy's optimizers (teleprompters) to automatically refine your AI programs.

### Example 1: Basic BootstrapFewShot Setup
**Problem:** Your model is struggling with a complex classification task.
**Solution:** Use the `BootstrapFewShot` optimizer to find the best examples.

```python
import dspy
from dspy.teleprompters import BootstrapFewShot

# 1. Define the task logic (Signature)
class SupportTriage(dspy.Signature):
    """Classify support requests into URGENT, NORMAL, or LOW."""
    request_text = dspy.InputField()
    priority = dspy.OutputField(desc="URGENT, NORMAL, or LOW")

# 2. Define the Metric (Success Criteria)
def triage_metric(example, prediction, trace=None):
    """Simple exact-match metric for classification."""
    return example.priority.upper() == prediction.priority.upper()

def compile_simple_optimizer(trainset: list):
    """Demonstrates basic few-shot optimization."""

    # 3. Initialize the Optimizer
    # max_bootstrapped_demos: how many examples to 'teach' the model
    optimizer = BootstrapFewShot(
        metric=triage_metric,
        max_bootstrapped_demos=4,
        max_labeled_demos=4
    )

    # 4. Compile (The Search phase)
    # student = dspy.Predict(SupportTriage)
    # compiled_program = optimizer.compile(student, trainset=trainset)
    # return compiled_program
    pass

# Note: In 2026, 'Compiling' a prompt is the equivalent of 'Training' a model.
```
**Why this is preferred:** It automatically creates a "Few-Shot Prompt" that is **mathematically proven** to work well on your training data, replacing manual example selection.

---

### Example 2: Optimizing with "LLM-as-a-Judge" Metric
**Problem:** You can't use simple "Exact Match" for a creative task like summarization.
**Solution:** Use a more powerful model inside the metric function to "Grade" the optimizer's candidate prompts.

```python
import dspy

def judge_metric(example, prediction, trace=None):
    """Uses a secondary LLM to grade the output of the optimizer's candidate."""

    # The 'Judge' prompt defines the desired qualitative properties
    judge_prompt = f"""
    ### RUBRIC
    - Score 1.0: Accurate, concise, and professional.
    - Score 0.0: Wordy, incorrect, or rude.

    REFERENCE: {example.summary}
    PREDICTION: {prediction.summary}
    """

    # score_str = call_gpt4o(judge_prompt)
    # return float(score_str) > 0.8
    return True

# MIPROv2 or COPRO can then use this 'Subjective' metric
# to find prompts that 'feel' better to human users.
```
**Why this is preferred:** It allows the optimizer to find prompts that improve **Qualitative** aspects like "Tone" and "Flow," which deterministic code cannot measure.

---

### Example 3: Using MIPROv2 for Multi-Objective Search
**Problem:** You need a prompt that is accurate but also stays under 500 tokens to save money.
**Solution:** Use `MIPROv2` to optimize for both accuracy and length.

```python
from dspy.teleprompters import MIPROv2

def run_advanced_optimization(trainset: list):
    """Uses Bayesian Optimization to find the best Instruction + Few-Shot combo."""

    # MIPROv2 uses a 'Teacher' model to propose new instructions
    # and a 'Student' model to evaluate them.
    optimizer = MIPROv2(
        metric=triage_metric,
        num_candidates=10, # Number of instruction variations to try
        init_temperature=1.0
    )

    # The 'Compile' step here is a heavy search over instructions AND examples
    # compiled_bot = optimizer.compile(
    #     dspy.Predict(SupportTriage),
    #     trainset=trainset,
    #     num_trials=30 # Total iterations of search
    # )
```
**Why this is preferred:** It is the most **advanced search strategy** available in 2026. It uses Bayesian Optimization to find the "Pareto Frontier" of performance vs. cost.

---

### Example 4: Automatic Instruction Proposal (Zero-Shot)
**Problem:** You don't even know how to write the initial "System Prompt."
**Solution:** Use an optimizer to "Propose" instructions based on a description of the task.

```python
# Conceptual Workflow of Automatic Proposal:
# 1. Signature: input(text) -> output(summary)
# 2. Optimizer: Proposes "You are a Chief of Staff. Distill the following..."
# 3. Optimizer: Proposes "You are a Technical Lead. Extract only the action items..."
# 4. Search: Finds that "Chief of Staff" instruction yields 12% higher factual recall.
# 5. Final Result: The "Chief of Staff" prompt is compiled into the program.
```
**Why this is preferred:** it addresses the **"Blank Page"** problem. The system often generates instructions that use specific model-trigger words you wouldn't know.

---

### Example 5: Handling "Negative Constraints" via Optimization
**Problem:** You want the model to STOP saying "As an AI language model..."
**Solution:** Include a negative penalty in your metric so the optimizer avoids any prompt that triggers that phrase.

```python
def anti_disclaimer_metric(example, prediction, trace=None):
    """A metric that punishes 'Helpful Assistant' fluff."""

    forbidden_phrases = ["as an ai", "i hope this helps", "certainly!"]

    # 1. Check for negative constraints
    if any(phrase in prediction.text.lower() for phrase in forbidden_phrases):
        return 0.0 # Hard failure for the optimizer

    # 2. Check for task accuracy
    return 1.0 if prediction.is_correct else 0.0

# The optimizer will now discard any prompt variations that lead to disclaimers.
```
**Why this is preferred:** The optimizer will "learn" to avoid certain wordings (like "Be polite") that often trigger LLM disclaimers.

---

### Example 6: "BootstrapFewShotWithRandomSearch"
**Problem:** You have enough compute budget and want the absolute highest accuracy.
**Solution:** Use random search to explore dozens of different "Bootstrap" combinations.

```python
from dspy.teleprompters import BootstrapFewShotWithRandomSearch

# num_candidate_programs: The number of different 'Prompt Sets' to evaluate
optimizer = BootstrapFewShotWithRandomSearch(
    metric=triage_metric,
    max_bootstrapped_demos=3,
    num_candidate_programs=50 # Brute-force search for the win
)

# compiled_program = optimizer.compile(MyModule(), trainset=trainset)
```
**Why this is preferred:** It prevents getting stuck in a **Local Maximum**. By exploring more of the search space, you find the "hidden gems" of prompt engineering.

---

### Example 7: Cross-Model Compilation
**Problem:** A prompt optimized for GPT-4 might not be best for Llama-3.
**Solution:** Run the same optimizer twice—once for each model.

```python
# Compilation 1: Target Llama-3 (Requires more detailed instructions)
# with dspy.context(lm=llama3):
#    llama_optimized = optimizer.compile(MyModule(), trainset=data)

# Compilation 2: Target GPT-4o (Requires more concise instructions)
# with dspy.context(lm=gpt4o):
#    gpt_optimized = optimizer.compile(MyModule(), trainset=data)
```
**Why this is preferred:** It acknowledges that LLMs have **"Dialects."** A prompt that is "too wordy" for GPT-4 might be "just right" for a smaller model that needs more guidance.

---

### Example 8: Multi-Stage Pipeline Optimization
**Problem:** You have a 5-step agentic pipeline. If you optimize everything at once, the search space is too big.
**Solution:** Optimize the first module, then "Freeze" its prompt and optimize the second.

```python
# Step 1: Optimize the 'Retriever' to find better facts.
# Step 2: Use those facts to optimize the 'Synthesizer'.
# Step 3: Use the synthesized output to optimize the 'Editor'.

# In 2026, we call this 'End-to-End Programmatic Training'.
```
**Why this is preferred:** It follows the **Layered Optimization** principle, ensuring that each part of the system is a stable foundation for the next.

---

## Conclusion: The Algorithm is the Engineer

In 2026, the best "Prompt Engineer" on your team is an **Optimizer**. By defining clear metrics and using search-based algorithms, we can find prompts that are significantly more accurate, cheaper, and more robust than anything a human could write by hand.

In the next chapter, we will look at **GEPA**, the 2025 breakthrough that made this optimization even faster and more efficient.

---

## References & Further Reading
- **Khattab et al. (2023)**: *DSPy: Compiling Declarative Language Model Programs*.
- **Medium (Buket Fildisi)**: *Prompt Optimisation with DSPy's MIPROv2*.
- **Stanford NLP**: *MIPROv2: Multi-objective Instruction-Proposal Optimizer*.
- **Emergent Mind**: *Dynamic Prompt Optimization with DSPy*.
