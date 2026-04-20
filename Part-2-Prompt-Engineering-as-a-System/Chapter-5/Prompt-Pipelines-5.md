# Chapter 5: Prompt Pipelines

## Introduction: Moving Beyond the Single Message

In the previous part, we focused on the individual components of a prompt—its architecture, techniques, and context. But in 2026, real AI value is built not by single messages, but by **Prompt Pipelines**.

A pipeline is a series of interconnected steps where the output of one LLM call (or a tool call) becomes the input for the next. Instead of a simple `User → Prompt → Output` model, we now use a more robust `Input → Decompose → Reason → Tool → Verify → Output` workflow. This modular approach is the foundation of **AI System Engineering**.

---

## Deep Technical Analysis: Pipeline Architectures

The shift from "Monolithic Prompts" to "Modular Pipelines" is driven by several key technical factors:

### 1. The Reasoning Token Budget
Research has shown that LLMs have a "Reasoning Peak"—they are most accurate when focused on a single, well-defined sub-task. As you add more tasks to a single prompt (e.g., "Summarize this, then translate it, then format as JSON"), the error rate grows exponentially. Pipelines solve this by allocating a fresh "Reasoning Budget" (a new LLM call) to each specific task.

### 2. State Management and Error Propagation
In a monolithic prompt, if the model fails at step 2 of 5, the entire output is usually unusable. In a pipeline, we can implement **Checkpointing**. We can verify the output of step 2; if it fails, we can "Retry" or "Re-route" before moving to step 3. This prevents "Error Propagation," where a small mistake early in the process ruins the final result.

### 3. Latency vs. Throughput (Parallelism)
Pipelines allow for **Parallel Execution**. If you need to analyze 5 different aspects of a document (e.g., Sentiment, Entity Extraction, Summary), you can run 5 LLM calls in parallel. This significantly reduces "User-Perceived Latency" compared to a single long prompt that has to process everything sequentially.

---

## Why Pipelines Solve Real-World Problems

In practice, Prompt Pipelines solve several critical production issues:
-   **Hallucination in Complex Tasks:** By breaking a task like "Write a 10-page report" into "Write an outline," then "Research each section," then "Write each section," you drastically reduce the model's tendency to drift or invent facts.
-   **Debugging "Black Boxes":** When a pipeline fails, you can see exactly which node in the graph was the culprit. Was it the "Retriever" failing to find data, or the "Summarizer" failing to process it?
-   **Tool and API Integration:** Pipelines act as the "Glue" between LLMs and traditional software. You can run a Python script, call a SQL database, or hit a third-party API between two LLM steps.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to build robust, multi-stage pipelines using modern Python patterns.

### Example 1: The "Sequential" Extraction-to-Summary Pipeline
**Problem:** Summarizing a 5,000-word document in one go often leads to the AI missing key details (the "Lost in the Middle" problem).
**Solution:** Use a 2-step pipeline. Step 1 extracts "Atomic Facts," and Step 2 synthesizes those facts into a summary.

```python
from typing import List, Dict

# Mock LLM call for demonstration
def call_llm_api(prompt: str) -> str:
    """Simulates a call to a language model."""
    if "FACTS" in prompt:
        return "1. Revenue grew 20%. 2. New office in Paris. 3. Costs cut by 5%."
    return "strong growth and international expansion."

def execute_summary_pipeline(document_text: str) -> str:
    """
    Demonstrates a linear sequential pipeline.

    What it solves: Prevents hallucination by grounding the final summary
    in intermediate extracted facts.
    """

    # Node 1: Fact Extraction (Focus: Precision)
    # By narrowing the task to 'bullets only', we maximize recall.
    extract_prompt = f"### TASK: Extract top 5 facts as bullets:\n{document_text}"
    atomic_facts = call_llm_api(extract_prompt)

    # Node 2: Summary Generation (Focus: Narrative)
    # The model no longer sees the noisy original text, only the clean facts.
    summary_prompt = f"""
    ### CONTEXT (FACTS ONLY):
    {atomic_facts}

    ### TASK:
    Using only the facts above, write a 1-sentence executive summary.
    """
    final_summary = call_llm_api(summary_prompt)

    return final_summary

# Execution Example
if __name__ == "__main__":
    doc = "Long corporate document text here..."
    # result = execute_summary_pipeline(doc)
    # print(f"Sequential Result: {result}")
```
**Why this is preferred:** It ensures the summary is **grounded in extracted facts**. By forcing the model to first "commit" to a list of facts, you prevent it from hallucinating external information during the summary phase.

---

### Example 2: The "Conditional Router" Pipeline
**Problem:** You have different "Expert" prompts for different topics (Billing vs. Tech Support), but the user doesn't know which one to use.
**Solution:** Use a "Router" LLM call to categorize the query and then route it to the appropriate specialized pipeline.

```python
from typing import Callable, Dict

def billing_specialist(query: str) -> str:
    return "Routing to billing secure server..."

def tech_specialist(query: str) -> str:
    return "Checking server logs for your ID..."

def router_pipeline(user_query: str) -> str:
    """
    Routes queries to specialized modules based on intent classification.
    """
    # 1. Classification Node (Low cost)
    # intent = call_cheap_model(f"Categorize as BILLING or TECH: {user_query}")
    intent = "BILLING" # Mock result

    # 2. Logic Dispatcher
    expert_map: Dict[str, Callable[[str], str]] = {
        "BILLING": billing_specialist,
        "TECH": tech_specialist
    }

    handler = expert_map.get(intent, lambda q: "General response...")
    return handler(user_query)

# Execution Example
if __name__ == "__main__":
    # print(router_pipeline("Why was I charged twice?"))
    pass
```
**Why this is preferred:** It enables **Specialization**. Specialized prompts with specialized few-shot examples are always more accurate than a single "Generalist" prompt.

---

### Example 3: Parallel Reasoning for Multi-Topic Queries
**Problem:** A user asks "What is the weather in London AND the price of Gold?". Sequential processing is slow.
**Solution:** Use Python's `asyncio` to trigger multiple independent LLM/Tool calls in parallel.

```python
import asyncio
from typing import List

async def fetch_tool_data(tool_name: str, query: str) -> str:
    """Simulates an asynchronous API/Tool call."""
    await asyncio.sleep(0.5) # Simulate network latency
    return f"[{tool_name} Result for {query}]"

async def parallel_query_pipeline(query: str) -> str:
    """Executes independent sub-tasks in parallel to minimize latency."""

    # In a real app, an LLM would first split the compound query into sub-tasks
    tasks = [
        fetch_tool_data("Weather", "London"),
        fetch_tool_data("Finance", "Gold Price")
    ]

    # Run tasks concurrently
    results = await asyncio.gather(*tasks)

    return " | ".join(results)

# Execution Example
if __name__ == "__main__":
    # asyncio.run(parallel_query_pipeline("London weather and Gold price"))
    pass
```
**Why this is preferred:** It optimizes for **Latency**. In production, reducing response time from 4 seconds to 2 seconds is often more valuable than a slight increase in accuracy.

---

### Example 4: The "Self-Correction" Verification Loop
**Problem:** LLMs often fail on negative constraints (e.g., "Do not use the word 'excellent'").
**Solution:** Add a "Verification Node" that checks the output of the "Generation Node" and triggers a retry if the constraint is violated.

```python
def generation_node(topic: str) -> str:
    return "This is an excellent summary of AI."

def verification_node(output: str) -> str:
    """Checks for violations of negative constraints."""
    if "excellent" in output.lower():
        return "FAIL: You used the forbidden word 'excellent'."
    return "PASS"

def polish_pipeline(topic: str):
    """
    Implements an automated quality assurance loop.
    """
    # 1. First Attempt
    draft = generation_node(topic)

    # 2. Automated QA
    feedback = verification_node(draft)

    if "FAIL" in feedback:
        # 3. Corrective pass using feedback as a 'hint'
        # draft = call_llm(f"Fix this: {draft}. Rule: {feedback}")
        return "This is a great summary of AI." # Fixed

    return draft
```
**Why this is preferred:** It builds **Quality Assurance (QA)** into the system itself. This "Critic" pattern is the most effective way to enforce hard constraints that a single prompt might ignore.

---

### Example 5: Task Decomposition (The "Outline-First" Pattern)
**Problem:** Generating a long document (e.g., a README or a Project Plan) all at once leads to loss of structure and coherence.
**Solution:** Decompose the task into an "Outline" phase and a "Section Generation" phase.

```python
from typing import List

def planner_node(topic: str) -> List[str]:
    """Stage 1: Logic planning."""
    # prompt = f"Create a 3-section outline for: {topic}"
    return ["Introduction", "Architecture", "Security"]

def executor_node(section: str, topic: str) -> str:
    """Stage 2: Focused generation."""
    # prompt = f"Write the content for '{section}' in the context of {topic}"
    return f"Details about {section}..."

def document_pipeline(topic: str) -> str:
    # 1. Generate plan
    sections = planner_node(topic)

    # 2. Iterate through plan
    full_doc = []
    for s in sections:
        content = executor_node(s, topic)
        full_doc.append(f"## {s}\n{content}")

    return "\n\n".join(full_doc)
```
**Why this is preferred:** It avoids **Model Exhaustion**. LLMs have a "Reasoning Window" that degrades as they generate more text. By resetting the prompt for each section, you maintain high quality throughout the document.

---

### Example 6: The "Tool-Assisted" Context Injection
**Problem:** The AI makes up user data because it doesn't have access to your live database.
**Solution:** Chain a "Database Lookup" (Python code) *before* the LLM reasoning step.

```python
import json

class Database:
    @staticmethod
    def get_user_balance(uid: str) -> float:
        return 150.50 # Mock data

def balance_inquiry_pipeline(user_id: str, query: str) -> str:
    """
    Combines deterministic code with stochastic reasoning.
    """
    # 1. Traditional Code (Deterministic Truth)
    balance = Database.get_user_balance(user_id)

    # 2. AI Reasoning (Grounded Context)
    prompt = f"""
    ### USER_DATA:
    Balance: ${balance}

    ### TASK:
    Answer the query based ONLY on the data above.
    Query: {query}
    """
    # return call_llm(prompt)
    pass
```
**Why this is preferred:** It ensures **Grounding**. In AI System Engineering, we always prefer to fetch "Ground Truth" using deterministic code (SQL/APIs) rather than asking the LLM to remember it.

---

### Example 7: The "Translation & Format" Split
**Problem:** Asking an LLM to translate text and output JSON at the same time often results in "Broken JSON" because the model focuses too much on the linguistic translation.
**Solution:** Separate the linguistic task from the structural task.

```python
# Step 1: Pure Linguistic Node
# prompt = "Translate this to Spanish: 'Meet Bob in London'"
translation = "Encuentro con Bob en Londres"

# Step 2: Pure Structural Node
# prompt = f"Extract entities from this text into JSON: {translation}"
# Result: { "person": "Bob", "location": "Londres" }
```
**Why this is preferred:** It follows the **Single Responsibility Principle**. By isolating the tasks, you reduce the "Cognitive Load" on the model, leading to 100% JSON validity and better translation quality.

---

### Example 8: Human-in-the-Loop (Staged Deployment)
**Problem:** You don't want an agent to automatically send an email to a client without a sanity check.
**Solution:** Create a pipeline that "Pauses" after generating a draft and waits for a human "Approval" signal.

```python
class PipelineState:
    def __init__(self, draft: str):
        self.draft = draft
        self.is_approved = False

def stage_1_generate_draft(user_input: str) -> PipelineState:
    """AI works autonomously to create a proposal."""
    # draft = call_llm(f"Draft email: {user_input}")
    return PipelineState("Mock Email Body")

def stage_2_finalize_action(state: PipelineState) -> str:
    """Only proceeds if a human has verified the work."""
    if not state.is_approved:
        return "WAITING: Manual approval required."

    # Send email logic...
    return "SUCCESS: Action executed."

# Execution Example:
# state = stage_1_generate_draft("Refund request")
# ... wait for human ...
# state.is_approved = True
# res = stage_2_finalize_action(state)
```
**Why this is preferred:** It provides the **Governance** necessary for enterprise AI. Human-in-the-loop is not a failure of AI; it is a design pattern for high-stakes environments.

---

## Conclusion: The Modular Mindset

A single prompt is a script; a pipeline is an application. By moving to a modular architecture, you create systems that are more reliable, easier to debug, and capable of solving far more complex problems.

In the next chapter, we will learn how to measure the success of these pipelines using **Evaluation-Driven Development**.

---

## References & Further Reading
- **LangChain Documentation**: *Chain of Thought and Prompt Pipelines*.
- **Reddit (r/salesengineers)**: *A Practical Guide to AI Upskilling in 2026*.
- **DeepLearning.AI**: *Building Systems with the ChatGPT API*.
- **Anthropic Guide**: *Chaining Prompts for Complex Tasks*.
