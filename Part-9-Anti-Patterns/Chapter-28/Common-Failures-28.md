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
# BAD: Monolithic Mega-Prompt
bad_prompt = "Summarize this, then translate to French, then output JSON..."

# GOOD: Modular Pipeline
def good_pipeline(text):
    summary = call_llm(f"Summarize: {text}")
    french = call_llm(f"Translate to French: {summary}")
    return call_llm(f"Extract JSON from: {french}")
```
**Why this is preferred:** It prevents **Attention Collapse**. Each model call has a 100% focus on a single, simple task.

---

### Example 2: "Ignore Prior" vs. Narrative Lock-in
**Problem:** The model gives a "General Knowledge" answer instead of using your specific data.
**Solution:** Use a "Grounding Anchor" at the end of the prompt.

```python
# GOOD: Explicitly countering narrative lock-in
prompt = f"""
CONTEXT: {data}
TASK: Based ONLY on the context above, answer the question.
CRITICAL: If the context contradicts your training data, prioritize the CONTEXT.
If the info isn't in the context, say 'I don't know'.
"""
```
**Why this is preferred:** It forces the model's attention back to the **Knowledge Layer** (the context) and away from its pre-trained "biases."

---

### Example 3: Missing Verification (Silent Regression)
**Problem:** You change a prompt and don't realize it broke the output format.
**Solution:** Use a Pydantic guardrail to catch format failures instantly.

```python
from pydantic import ValidationError

def safe_run(prompt):
    res = call_llm(prompt)
    try:
        return MySchema.model_validate_json(res)
    except ValidationError:
        return call_llm(f"Your previous output was invalid. Fix it: {res}")
```
**Why this is preferred:** It prevents **Error Propagation**. The system catches the mistake before it reaches the end user or the next pipeline step.

---

### Example 4: Context "Dumping" vs. Reranking
**Problem:** Dumping 10 documents into a prompt makes the model miss the relevant one.
**Solution:** Use a reranker to only send the "top 3" documents.

```python
# BAD: context = "\n".join(all_10_docs)
# GOOD:
context = rerank(query, all_docs)[:3]
```
**Why this is preferred:** It stays within the **Reasoning Peak** of the model. Giving the model less "Noise" allows it to focus more "Signal" on the answer.

---

### Example 5: Unstructured History vs. Summary Memory
**Problem:** A long chat log makes the model slow and confused.
**Solution:** Periodically summarize the "old" history.

```python
def get_memory(history):
    if len(history) > 10:
        summary = summarize_old_chats(history[:-2])
        return f"Past Summary: {summary}\nLatest: {history[-2:]}"
    return history
```
**Why this is preferred:** It prevents **Memory Overwrite**. The original goal and the latest context stay visible to the model.

---

### Example 6: "Magic Adjectives" vs. Success Criteria
**Problem:** Telling the model to be "Very smart and concise" doesn't work.
**Solution:** Give the model a "Checklist" of things to do.

```python
# BAD: "Write a high-quality summary."
# GOOD:
instructions = """
1. List 3 key points.
2. Use bullet points.
3. Keep total words under 50.
"""
```
**Why this is preferred:** "High-quality" is subjective. Numbered instructions are **Deterministic**.

---

### Example 7: Model Sensitivity (Hardcoded Logic)
**Problem:** A prompt written for GPT-4 fails on Llama 3.
**Solution:** Use a model-agnostic DSPy Signature.

```python
import dspy
class MyLogic(dspy.Signature):
    """(Signature logic here...)"""

# compiled_model = optimizer.compile(MyLogic(), lm=llama3)
```
**Why this is preferred:** It avoids **Model Lock-in**. DSPy handles the translation of logic into model-specific "best practices."

---

### Example 8: No Evaluation vs. Golden Dataset
**Problem:** "Testing" the prompt by running it 3 times manually.
**Solution:** Run a 50-example eval script on every change.

```python
def run_tests():
    dataset = load_golden_set()
    score = run_eval(new_prompt, dataset)
    if score < 0.9: raise Exception("Regression detected!")
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
