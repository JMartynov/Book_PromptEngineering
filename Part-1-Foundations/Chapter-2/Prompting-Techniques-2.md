# Chapter 2: Prompting Techniques (Evolution Ladder)

## Introduction: The Ladder of Complexity

Prompting has evolved from simple "one-shot" questions to complex, multi-step reasoning architectures. In 2026, we view these techniques as an **Evolution Ladder**. As the difficulty of the task increases, we climb the ladder, adding more structure, reasoning, and feedback loops to ensure high-quality outputs.

Moving up the ladder isn't just about adding more words; it's about providing the model with a **computational framework** to process information more effectively.

---

## Deep Technical Analysis: The Evolution Ladder

### 🟢 Level 1: Foundations (Probabilistic Guidance)
At the base of the ladder, we focus on providing clear examples and personas. These techniques work by narrowing the model's output distribution.
-   **Zero-Shot / Few-Shot:** Providing 1–5 examples of the desired input-output mapping. Research shows few-shot prompting is the single most effective way to improve model reliability for formatting and style, often increasing accuracy from ~20% to over ~70% on complex classification.
-   **Role Prompting:** Assigning a persona to shift the model's vocabulary and decision-making logic.

### 🟡 Level 2: Reasoning (System 2 Thinking)
Level 2 techniques are designed to simulate "System 2" (deliberative) thinking in LLMs.
-   **Chain-of-Thought (CoT):** Asking the model to "think step-by-step." This forces the model to allocate more "compute-per-token" to the reasoning phase, drastically reducing hallucinations.
-   **Self-Consistency:** Running the same CoT prompt 5 times and taking the "Majority Vote" of the answers. Research has shown this can improve accuracy by an additional 12–18%.
-   **Decomposition:** Breaking a complex prompt into a sequence of smaller sub-prompts. Models have a limited "reasoning window"; solving five small problems is easier for an LLM than solving one massive problem.

### 🔴 Level 3: Agentic Thinking (Dynamic Loops)
The top of the ladder involves techniques where the model interacts with itself or external tools in a loop.
-   **ReAct (Reason + Act):** A framework where the model interleaves "Thoughts" (reasoning about what to do) and "Actions" (executing a tool call).
-   **Tree-of-Thoughts (ToT):** Exploring multiple "reasoning branches" simultaneously and evaluating which one is most likely to lead to the correct solution.
-   **Self-Reflection:** Asking the model to review its own output for errors and then regenerate a better version. This simulates the "first draft vs. final draft" process.

---

## Why This Ladder Solves Real-World Problems

In practice, the Evolution Ladder solves several critical engineering issues:
-   **Hallucination in Logic:** CoT forces the model to externalize its reasoning, making it easier to spot where a logical error occurred.
-   **Formatting Drift:** Few-shot examples anchor the model's output format, preventing it from deviating into conversational text.
-   **Handling Tool Errors:** ReAct loops allow the model to "retry" a tool call if the first one fails, making the overall system more resilient.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to implement the Evolution Ladder using modern Python patterns, focusing on robust reasoning and agentic behaviors.

### Example 1: Dynamic Few-Shot Selection with Embeddings
**Problem:** Hardcoding examples in a prompt is inefficient if you have hundreds of possible examples. Static few-shotting can also lead to "Example Fatigue" where the model ignores the examples that aren't relevant to the current input.
**Solution:** Use a simple similarity-based approach to select the most relevant examples from a library for the current task.

```python
import numpy as np
from typing import List, Dict, Any
from pydantic import BaseModel

# Mock embedding and LLM calls for demonstration
def get_embedding(text: str) -> List[float]:
    """Simulates a call to an embedding model like text-embedding-3-small."""
    return [0.1] * 1536 # Placeholder vector

class Example(BaseModel):
    """Represents a validated demonstration for a prompt."""
    query: str
    response: str
    embedding: Optional[List[float]] = None

class DynamicFewShotManager:
    """
    Manages a library of examples and retrieves them semantically.

    Benefit: Optimizes context window by only providing relevant demonstrations.
    """

    def __init__(self, example_library: List[Example]):
        self.library = example_library
        # Pre-compute embeddings for efficiency in production
        for ex in self.library:
            ex.embedding = get_embedding(ex.query)

    def get_top_k(self, current_query: str, k: int = 2) -> str:
        """Finds semantically similar examples using cosine similarity."""
        query_vec = np.array(get_embedding(current_query))

        # Calculate scores (In production, use a Vector DB like Pinecone/Qdrant)
        scored = []
        for ex in self.library:
            sim = np.dot(query_vec, np.array(ex.embedding)) # Simple dot product
            scored.append((sim, ex))

        # Sort and return top K
        scored.sort(key=lambda x: x[0], reverse=True)
        top_examples = [s[1] for s in scored[:k]]

        return "\n\n".join([f"Input: {e.query}\nOutput: {e.response}" for e in top_examples])

# Execution Example
if __name__ == "__main__":
    library = [
        Example(query="My battery is dead.", response="Category: Hardware"),
        Example(query="How do I change my password?", response="Category: Security")
    ]
    manager = DynamicFewShotManager(library)

    # Prompt would use manager.get_top_k("The phone won't turn on.")
    # print(manager.get_top_k("The phone won't turn on."))
```
**Why this is preferred:** It ensures the model sees examples that are contextually relevant to the current query, which is far more effective than static few-shotting.

---

### Example 2: The "Self-Consistency" Majority Vote
**Problem:** A single LLM call might produce a "fluke" error in logic or calculation.
**Solution:** Run the reasoning prompt multiple times and use a Python function to pick the most common answer.

```python
import re
from collections import Counter
from typing import List, Optional

def call_llm(prompt: str) -> str:
    """Mock LLM call returning a step-by-step solution."""
    return "Thinking: 1. A=1, B=1. Result: 2. FINAL: 2"

def extract_answer(text: str) -> Optional[str]:
    """Extracts the final result from a CoT response block."""
    match = re.search(r"FINAL: (\d+)", text)
    return match.group(1) if match else None

def solve_with_consensus(problem: str, n_samples: int = 5) -> str:
    """
    Runs the same reasoning prompt multiple times and picks the most common result.

    Problem solved: Reduces reasoning 'flukes' in complex math or logic.
    """
    results = []
    for _ in range(n_samples):
        # We increase 'temperature' slightly to ensure diversity of reasoning paths
        raw_output = call_llm(problem)
        ans = extract_answer(raw_output)
        if ans:
            results.append(ans)

    if not results:
        return "Error: No valid results produced."

    # Majority Vote Logic
    counts = Counter(results)
    most_common_ans, vote_count = counts.most_common(1)[0]

    print(f"Consensus reached: {most_common_ans} ({vote_count}/{n_samples} votes)")
    return most_common_ans

# Execution Example
if __name__ == "__main__":
    # ans = solve_with_consensus("If X=2 and Y=3, what is X+Y?")
    pass
```
**Why this is preferred:** It is the standard "Safety Pattern" for high-stakes arithmetic or logic. Research has proven that multiple independent "thoughts" are significantly more accurate than a single one.

---

### Example 3: Task Decomposition (Prompt Chaining)
**Problem:** Asking an LLM to "write a full blog post from a raw transcript" often results in poor structure and missed key points.
**Solution:** Chain two prompts—one to extract a structured outline, and a second to write the post section-by-section.

```python
def pipeline_stage_1_outline(transcript: str) -> List[str]:
    """Stage 1: Structural Extraction."""
    # prompt = f"Extract a 3-point outline from: {transcript}"
    return ["Introduction", "Product Features", "Conclusion"]

def pipeline_stage_2_content(section: str, global_outline: List[str]) -> str:
    """Stage 2: Detailed Drafting."""
    # prompt = f"Write the content for '{section}' based on this outline: {global_outline}"
    return f"Content for {section}..."

def execute_chained_pipeline(data: str):
    """Coordinates the multi-stage generation process."""
    outline = pipeline_stage_1_outline(data)

    full_document = []
    for section_name in outline:
        content = pipeline_stage_2_content(section_name, outline)
        full_document.append(f"## {section_name}\n{content}")

    return "\n\n".join(full_document)

# Execution Example
if __name__ == "__main__":
    # blog_post = execute_chained_pipeline("Raw meeting transcript...")
    pass
```
**Why this is preferred:** Each prompt has a much simpler task, leading to significantly higher overall quality and fewer hallucinations in long-form content.

---

### Example 4: The ReAct Agent Loop (Reason + Act)
**Problem:** LLMs can't access real-time data like stock prices or weather.
**Solution:** Use a prompt that encourages the model to "stop and ask" for information from a tool in a loop.

```python
import json

def get_current_stock_price(symbol: str) -> float:
    """Mock tool call."""
    return 190.20 if symbol == "AAPL" else 0.0

def react_agent_executor(goal: str):
    """
    Implements the Reason + Act loop.

    Logic:
    1. Model thinks about what tool it needs.
    2. Model calls the tool.
    3. Python executes tool and returns observation.
    4. Model reasons about the new data and provides final answer.
    """

    # SYSTEM PROMPT would define the THOUGHT, ACTION, OBSERVATION format.
    # We simulate a 2-turn interaction.

    # Turn 1: Model realizes it needs price data
    thought_1 = "I need to find the current price of AAPL to answer the user."
    action_1 = '{"tool": "get_price", "args": {"symbol": "AAPL"}}'

    # Execution: Python calls the tool
    obs_1 = get_current_stock_price("AAPL")

    # Turn 2: Model provides final answer based on observation
    final_answer = f"The current price of AAPL is ${obs_1}."
    return final_answer

# Execution Example
if __name__ == "__main__":
    # print(react_agent_executor("Price of Apple?"))
    pass
```
**Why this is preferred:** This is the foundation of "Agentic" systems. It allows the model to interact with the world instead of just guessing.

---

### Example 5: "System 2" Reflection (The Critique Loop)
**Problem:** LLMs often make subtle errors that they can catch themselves if given a second chance to review their work.
**Solution:** Run a second "Critique" prompt to find errors in the first response and then a third "Update" prompt to fix them.

```python
def generate_draft(task: str) -> str:
    return "def query(id): return db.execute(f'SELECT * FROM users WHERE id={id}')" # Vulnerable code

def run_critique(draft: str) -> str:
    """Stage 2: Critical analysis from a different semantic perspective."""
    # prompt = f"Critically review this code for SQL injection. Draft: {draft}"
    return "Vulnerability: Line 1 uses string interpolation, making it prone to SQL injection."

def apply_fixes(draft: str, critique: str) -> str:
    """Stage 3: Verified implementation."""
    return "def query(id): return db.execute('SELECT * FROM users WHERE id=?', (id,))" # Fixed code

# Execution Example
if __name__ == "__main__":
    # initial = generate_draft("database query function")
    # feedback = run_critique(initial)
    # final_code = apply_fixes(initial, feedback)
    pass
```
**Why this is preferred:** It mimics the peer-review process, leading to safer and more robust code generation in production.

---

### Example 6: Handling Ambiguity with Clarification Loops
**Problem:** Users often provide vague prompts (e.g., "Generate a report").
**Solution:** Instruct the model to ask for more info if the request is underspecified, rather than hallucinating a guess.

```python
def process_user_intent(user_msg: str):
    """
    Ensures intent quality before execution.

    Rule: If info is missing, ask. DO NOT hallucinate.
    """
    # Logic (Simulated):
    # Intent: SUMMARY
    # Missing: SOURCE_TEXT

    if "missing" == "missing": # Pseudo logic
        return "I'd be happy to summarize that. Could you please provide the text or a link?"

    return "Proceeding to summary..."

# Execution Example
if __name__ == "__main__":
    # print(process_user_intent("Summarize for me."))
    pass
```
**Why this is preferred:** It prevents "Wasteful Hallucination" and ensures the AI actually does what the user intended, improving user satisfaction.

---

### Example 7: Tree-of-Thought (ToT) Approach Selection
**Problem:** For creative or strategic tasks, the first path the model takes might not be the best.
**Solution:** Prompt the model to generate three distinct approaches and then "Judge" which one is most likely to succeed.

```python
def tot_strategy_selector(goal: str):
    """
    Explores the 'Solution Tree' before committing.

    Benefit: Maximizes creativity and strategic depth.
    """
    # 1. Generate 3 Paths (A, B, C)
    # 2. Score Paths
    # 3. Select Best
    return "Strategy B (Social-First) was selected as it has the highest reach-per-dollar."

# Execution Example:
# final_plan = tot_strategy_selector("Launch a new coffee brand.")
```
**Why this is preferred:** It encourages the model to explore the "Solution Space" more broadly before committing to a single answer, which research shows results in higher creativity.

---

### Example 8: Zero-Shot Chain-of-Thought (The "Take a Breath" Pattern)
**Problem:** You need a quick accuracy boost but don't want to write a complex multi-step prompt.
**Solution:** Append a "Reasoning Trigger" to the end of your prompt.

```python
def fast_accuracy_boost(query: str):
    """
    Lowest effort, highest ROI technique.
    """
    # Adding 'Let's think step by step' is a research-proven
    # trigger for System 2 thinking in LLMs.
    prompt = f"{query}\n\nLet's think step by step before providing the answer."

    # return call_llm(prompt)
    pass

# Execution Example:
# ans = fast_accuracy_boost("A bat and a ball cost $1.10. The bat costs $1.00 more...")
```
**Why this is preferred:** It is the "Lowest Effort, Highest ROI" technique. Research indicates that this simple phrase triggers a different "Mode" in transformer-based models that improves math and logic scores by 10-20%.

---

## Conclusion: Climbing the Ladder

The choice of technique depends entirely on the complexity of your task. For simple data extraction, **Few-Shot** is enough. For complex financial analysis, you might need a combination of **Decomposition**, **Self-Consistency**, and **Reflection**.

By understanding the Evolution Ladder, you can design AI systems that are as simple as possible, but as powerful as necessary.

---

## References & Further Reading
- **Wei et al. (2022)**: *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*.
- **Yao et al. (2022)**: *ReAct: Synergizing Reasoning and Acting in Language Models*.
- **Wang et al. (2022)**: *Self-Consistency Improves Chain of Thought Reasoning in Language Models*.
- **Meta-Intelligence Tech (2026)**: *Prompt Engineering Guide: Advanced Techniques*.
