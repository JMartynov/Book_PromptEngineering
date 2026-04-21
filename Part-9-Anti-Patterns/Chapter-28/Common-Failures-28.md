# Chapter 28: Common Failures

## Introduction: The "Problem Map" of AI

In the first eight parts of this book, we've focused on what to do right. But in 2026, many engineering teams still fail because they repeat the same "GenAI Anti-Patterns" from 2023. They treat the LLM as a "Magic Box" rather than a stochastic software component. This leads to brittle, expensive, and unreliable systems.

Research into thousands of production AI systems has shown that most failures are not caused by "dumb models," but by **poor system design**. This chapter maps out the most common failures and provides the engineering patterns to avoid them.

---

## Deep Technical Analysis: The 4 Major Failure Modes

Failure in AI System Engineering usually falls into one of four technical buckets:

### 1. The "Mega-Prompt" Bloat (Attention Collapse)
**The Failure:** Engineers keep adding "rules" to a single system prompt until it is 3,000 tokens long.
**Technical Why:** LLMs have a "Reasoning Window." As the prompt gets longer, the model's attention starts to "smear." It follows the first and last instructions but ignores the 50 rules in the middle. This is the **Attention Collapse** phenomenon.

### 2. Semantic Drift (Multi-step Decay)
**The Failure:** In a 5-step pipeline, Step 1 makes a tiny error, Step 2 amplifies it, and by Step 5, the output is complete nonsense.
**Technical Why:** Without "Checkpoints" (verification nodes), errors accumulate. This is similar to "rounding errors" in floating-point math, but for meaning.

### 3. Confident Hallucination (Narrative Lock-in)
**The Failure:** The model retrieves the correct data but ignores it because it has a "strong prior" training on the topic.
**Technical Why:** This is **Narrative Lock-in**. If a model was trained 1,000 times on "The sky is blue," and your RAG context says "The sky on Mars is red," the model might still say "blue" because its internal weights override the provided context.

### 4. Memory Overwrite (State Corruption)
**The Failure:** In an agentic loop, the agent "forgets" the original user goal because its "Action History" has filled up the context window.
**Technical Why:** Without **Context Engineering** (Chapter 4), the "Recent History" (last 2 tool calls) crowds out the "Original Goal" (the system prompt), causing the agent to wander aimlessly.

---

## Why Mapping Failures Solves Real-World Problems

In practice, identifying these anti-patterns allows teams to:
-   **Debug Faster:** Instead of saying "The AI is being weird," you can say "We are seeing Semantic Drift at Step 3."
-   **Reduce Costs:** Moving from one "Mega-Prompt" to three "Micro-Prompts" (Pipelines) often reduces token usage and improves accuracy simultaneously.
-   **Build Robust Systems:** By anticipating "Narrative Lock-in," you can design prompts that explicitly tell the model to "Ignore your prior knowledge and use ONLY the context."

---

## Practical Implementation: 8 Python Examples

These examples demonstrate the "Anti-Pattern" (Bad) and the "Engineering Solution" (Good).

### Example 1: Mega-Prompt vs. Decomposed Pipeline
**Problem:** A single prompt trying to summarize, translate, and format.
**Solution:** Break it into three distinct LLM calls.

```python
from typing import Dict, Any

# BAD: The 'Bloated' Prompt
bad_mega_prompt = """
Summarize this text, then translate it to French, and then
return it as a JSON object with the keys 'summary' and 'entities'.
Constraint: DO NOT use the word 'excellent'.
"""

# GOOD: The Decomposed Pipeline
def optimized_pipeline(text: str) -> Dict[str, Any]:
    """Decomposes a complex task into focused nodes to prevent Attention Collapse."""

    # 1. Focused Task: Summarization
    summary = call_llm(f"Summarize this text without using the word 'excellent': {text}")

    # 2. Focused Task: Translation
    french_text = call_llm(f"Translate this to French: {summary}")

    # 3. Focused Task: Structural Extraction
    # In practice, use instructor for 100% JSON reliability
    return {"summary_fr": french_text, "entities": ["..."]}

# Execution Example:
# res = optimized_pipeline("Long corporate report...")
```
**Why this is preferred:** It prevents **Attention Collapse**. Each model call has a 100% focus on a single, simple task.

---

### Example 2: "Ignore Prior" vs. Narrative Lock-in
**Problem:** The model gives a "General Knowledge" answer instead of using your specific data.
**Solution:** Use a "Grounding Anchor" at the end of the prompt.

```python
def build_grounded_prompt(data: str, query: str) -> str:
    """Uses explicit conflict rules to override model pre-training bias."""

    return f"""
    ### CONTEXT_DATA
    {data}

    ### MISSION
    Answer the user query based ONLY on the CONTEXT_DATA above.

    ### RESOLUTION_RULES
    1. If the CONTEXT_DATA contradicts your internal knowledge, the CONTEXT_DATA is the truth.
    2. If the info is not in the context, output: "I do not have enough information."

    USER_QUERY: {query}
    """

# Example: Context says "Mars has green water."
# AI will answer "Green" instead of "Frozen/Red".
```
**Why this is preferred:** It forces the model's attention back to the **Knowledge Layer** (the context) and away from its pre-trained "biases."

---

### Example 3: Missing Verification (Silent Regression)
**Problem:** You change a prompt and don't realize it broke the output format.
**Solution:** Use a Pydantic guardrail to catch format failures instantly.

```python
from pydantic import BaseModel, ValidationError

class OutputSchema(BaseModel):
    summary: str
    timestamp: str # Required field

def safe_execution_node(prompt: str) -> OutputSchema:
    """Prevents error propagation via deterministic schema validation."""

    raw_res = call_llm(prompt)
    try:
        # Validates against the contract
        return OutputSchema.model_validate_json(raw_res)
    except (ValidationError, ValueError):
        # Automated Retry with feedback
        print("Regression detected. Retrying with error trace...")
        # return call_llm(f"Your JSON was missing 'timestamp'. Fix it: {raw_res}")
        pass
```
**Why this is preferred:** It prevents **Error Propagation**. The system catches the mistake before it reaches the end user or the next pipeline step.

---

### Example 4: Context "Dumping" vs. Reranking
**Problem:** Dumping 10 documents into a prompt makes the model miss the relevant one.
**Solution:** Use a reranker to only send the "top 3" documents.

```python
from typing import List

def optimized_retrieval(query: str, all_retrieved_docs: List[str]) -> str:
    """Maintains the model's 'Reasoning Peak' by pruning irrelevant context."""

    # 1. Rerank 10 docs to find the most high-signal ones
    # ranked_docs = reranker.score(query, all_retrieved_docs)

    # 2. Only inject the Top 3 into the final prompt
    signal_docs = all_retrieved_docs[:3]

    return "\n---\n".join(signal_docs)

# Result: Prompt stays under 2000 tokens, accuracy increases.
```
**Why this is preferred:** It stays within the **Reasoning Peak** of the model. Giving the model less "Noise" allows it to focus more "Signal" on the answer.

---

### Example 5: Unstructured History vs. Summary Memory
**Problem:** A long chat log makes the model slow and confused.
**Solution:** Periodically summarize the "old" history.

```python
def build_compact_memory(history: list) -> str:
    """Prevents Memory Overwrite by distilling old turns into semantic facts."""

    if len(history) < 10:
        return str(history)

    # Summarize everything except the most recent turns
    summary_of_past = call_llm(f"Summarize key facts from: {history[:-2]}")

    return f"""
    PAST_CONTEXT_SUMMARY: {summary_of_past}
    LATEST_TURNS: {history[-2:]}
    """
```
**Why this is preferred:** It prevents **Memory Overwrite**. The original goal and the latest context stay visible to the model.

---

### Example 6: "Magic Adjectives" vs. Success Criteria
**Problem:** Telling the model to be "Very smart and concise" doesn't work.
**Solution:** Give the model a "Checklist" of things to do.

```python
# BAD: "Write a good summary of this code."

# GOOD:
structured_instructions = """
1. List all public functions.
2. Identify the primary design pattern used.
3. Keep the total output under 100 words.
4. Use valid Markdown headers.
"""

# result = call_llm(f"Analyze this code: {code}\nCHECKLIST:\n{structured_instructions}")
```
**Why this is preferred:** "High-quality" is subjective. Numbered instructions are **Deterministic**.

---

### Example 7: Model Sensitivity (Hardcoded Logic)
**Problem:** A prompt written for GPT-4 fails on Llama 3.
**Solution:** Use a model-agnostic DSPy Signature.

```python
import dspy

class EntityExtractor(dspy.Signature):
    """Extract names and organizations from a news article."""
    article = dspy.InputField()
    entities = dspy.OutputField(desc="JSON list of found entities")

# The compiler finds the optimal prompt for WHATEVER model you set:
# dspy.settings.configure(lm=llama3)
# compiled_bot = optimizer.compile(EntityExtractor(), trainset=data)
```
**Why this is preferred:** It avoids **Model Lock-in**. DSPy handles the translation of logic into model-specific "best practices."

---

### Example 8: No Evaluation vs. Golden Dataset
**Problem:** "Testing" the prompt by running it 3 times manually.
**Solution:** Run a 50-example eval script on every change.

```python
def execute_release_eval(new_prompt_candidate: str):
    """Replaces 'vibes' with engineering rigor before deployment."""

    dataset = load_golden_set("v1_stable")
    baseline_score = 0.88

    # current_score = run_eval_suite(new_prompt_candidate, dataset)

    # if current_score < baseline_score:
    #     raise Exception("PROMPT REJECTED: Regression detected in evaluation suite.")
    pass
```
**Why this is preferred:** It replaces "Vibes" with **Engineering Rigor**. It is the only way to scale a production AI system safely.

---

## Conclusion: Engineering is the Antidote

The "Anti-Patterns" of 2026 are mostly remnants of the "AI Hype" of 2023. By moving from magic to mechanics—by decomposing tasks, verifying outputs, and measuring performance—you turn a "Chatbot" into a reliable "System."

In the next chapter, we will look at **Why Prompts "Break"** at the fundamental level of the transformer architecture.

---

## References & Further Reading
- **Reddit (r/PromptEngineering)**: *After 3000 hours, everything is one of 16 failures*.
- **Onestardao**: *Problem Map for AI Systems*.
- **OpenAI**: *Prompt Engineering Best Practices - Common Pitfalls*.
- **Liu et al. (2024)**: *Attention Smearing and Context Window Limits*.
- **DeepEval**: *Identifying and Fixing AI Regressions*.
