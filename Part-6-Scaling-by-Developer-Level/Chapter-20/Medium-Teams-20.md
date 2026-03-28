# Chapter 20: Medium Teams Stack

## Introduction: From Speed to Reliability

When your team grows from one developer to five or ten, "moving fast" is no longer enough. You need to ensure that when Developer A changes a prompt, it doesn't break Developer B's feature. In 2026, the **Medium Team Stack** is defined by **Consistency and Observability**.

The stack shifts from simple scripts to **Orchestration Frameworks**, **Persistent Vector Databases**, and **Automated Evaluation Pipelines**. The goal is to build a system that is robust, collaborative, and easy to debug.

---

## Deep Technical Analysis: The Collaborative AI Stack

The Medium Team Stack is built on four technical pillars:

### 1. Orchestration: LangGraph / LangChain
While indies use simple scripts, medium teams use **Stateful Orchestration**. Frameworks like LangGraph allow the team to define complex multi-step workflows as a "Graph." This provides a shared mental model of how the AI works, making it much easier for team members to collaborate on specific parts of the system (nodes).

### 2. Knowledge: Production Vector DBs (Qdrant / Weaviate)
Medium teams move away from "Vector-as-a-Service" and often deploy their own high-performance vector databases like **Qdrant** or **Weaviate**. This allows for more complex schemas, hybrid search (Vector + SQL), and better control over data privacy and latency.

### 3. Verification: Automated Eval Pipelines (CI/CD)
The #1 difference for a medium team is the **Eval Suite**. Every pull request triggers a "Regression Test" against a Golden Dataset. This ensures that the system's "Intelligence" is actually improving over time, rather than just drifting.

### 4. Visibility: Centralized Tracing (LangSmith / Langfuse)
Medium teams cannot debug via `print()` statements. They use **Centralized Tracing** to see every LLM call made by every developer and every production user in a single dashboard. This allows for rapid "Root Cause Analysis" when a user reports a bug.

---

## Why the Medium Stack Solves Real-World Problems

In practice, this stack solves several critical scaling issues:
-   **Prompt Drift:** Automated evals catch when a model update or a prompt tweak reduces accuracy across the board.
-   **Knowledge Silos:** Shared orchestration graphs and tracing allow any developer on the team to understand and debug any part of the AI system.
-   **Resource Contention:** Centralized observability helps the team identify which features are burning the most tokens, allowing for data-driven cost optimization.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to build a collaborative, production-ready AI system for a growing team.

### Example 1: Shared "State" in LangGraph
**Problem:** Multiple developers working on different parts of an agent need a way to share information.
**Solution:** Use a TypedDict to define a global "State" that all nodes in the graph can read and write to.

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class TeamState(TypedDict):
    # 'add_messages' ensures history is appended, not overwritten
    messages: Annotated[list, add_messages]
    research_notes: str
    is_ready_for_review: bool
```
**Why this is preferred:** It provides a **Single Source of Truth**. Any developer adding a new "Node" to the system knows exactly what data is available and how to update it.

---

### Example 2: Modular Node Functions
**Problem:** A 2,000-line Python file for an agent is impossible to maintain.
**Solution:** Break the agent's logic into small, independent "Node Functions" that can be tested in isolation.

```python
def research_node(state: TeamState):
    # Developer A focuses only on the research logic
    return {"research_notes": "Found 5 competitors..."}

def review_node(state: TeamState):
    # Developer B focuses only on the quality check logic
    return {"is_ready_for_review": True}
```
**Why this is preferred:** It enables **Parallel Development**. Two engineers can work on different parts of the same agent without stepping on each other's toes.

---

### Example 3: Production RAG with Metadata Filtering
**Problem:** A simple RAG system returns documents that aren't relevant to the user's specific project.
**Solution:** Use "Metadata Filters" in your production Vector DB to restrict the search space.

```python
def filtered_search(query, project_id):
    # This filter happens in the DB engine, not the LLM
    return vector_db.search(
        query,
        filter={"project_id": project_id, "status": "published"}
    )
```
**Why this is preferred:** It ensures **Data Isolation** between different projects or users, which is a hard requirement for B2B applications.

---

### Example 4: Automated CI/CD Regression Tests
**Problem:** A developer updates the "System Prompt" and accidentally breaks the "Billing" extractor.
**Solution:** Run a script in your CI/CD pipeline that checks the LLM's output against a "Golden Dataset."

```python
def test_billing_regression():
    # Load 50 'Golden' examples
    # Run new prompt version
    # Assert similarity > 0.95
    pass
```
**Why this is preferred:** It moves from **"Vibes-based deployment"** to **"Metrics-based deployment."** It gives the team the confidence to iterate fast.

---

### Example 5: Centralized Trace Logging
**Problem:** A user says "The AI gave a weird answer," but you can't see what actually happened.
**Solution:** Use a decorator or a context manager to send every step to a tracing platform (e.g. Langfuse).

```python
# In 2026, we use standard OpenTelemetry wrappers
@trace_span(name="AgentRun")
def run_agent(task):
    # All LLM calls inside this function are automatically correlated
    pass
```
**Why this is preferred:** It provides **Forensic Visibility**. You can "Replay" the exact sequence of events that led to a failure, even if it happened 3 days ago.

---

### Example 6: Multi-Model "Failover" Logic
**Problem:** Your primary LLM (e.g. GPT-4) hits a rate limit during peak hours.
**Solution:** Implement a "Fallback" mechanism in your orchestration layer.

```python
def call_llm_with_fallback(prompt):
    try:
        return gpt4.invoke(prompt)
    except RateLimitError:
        return claude3.invoke(prompt) # The 'Warm Standby'
```
**Why this is preferred:** It ensures **High Availability**. Your application remains functional even when your primary AI provider is struggling.

---

### Example 7: Standardized "Prompt Config" Files
**Problem:** Prompts are scattered throughout the code in different formats.
**Solution:** Use a dedicated `prompts/` directory with YAML files that include model settings and version numbers.

```yaml
# prompts/support_v2.yaml
model: gpt-4o
temperature: 0.2
text: "You are a support bot..."
```
**Why this is preferred:** It makes prompt changes **Reviewable**. A prompt update now looks like a normal code change in a Pull Request.

---

### Example 8: Collaborative "Human-in-the-Loop" UI
**Problem:** High-stakes AI outputs need a human "Expert" to verify them before they are saved.
**Solution:** Build a "Review Node" into your graph that pauses the state and sends a notification to a Slack channel or internal UI.

```python
def human_review_node(state: TeamState):
    if not state.get("human_approved"):
        return "wait_for_human"
    return "finalize"
```
**Why this is preferred:** It builds **Trust and Governance**. It allows the team to deploy AI for sensitive tasks while maintaining human accountability.

---

## Conclusion: Engineering for Scale

The Medium Team Stack is about removing the "Black Box" of AI and replacing it with a transparent, testable, and collaborative system. By moving to LangGraph, production Vector DBs, and automated evals, you ensure that your AI scales with your team and your user base.

In the next chapter, we will look at **Enterprise Systems**, where security, compliance, and multi-cloud reliability become the primary concerns.

---

## References & Further Reading
- **LangGraph**: *Building Stateful, Multi-Agent Applications*.
- **Qdrant**: *Vector Search Engine for Production AI*.
- **LangSmith**: *The Platform for LLM Debugging and Testing*.
- **DeepEval**: *Unit Testing Framework for LLMs*.
- **Klement Gunndu (2026)**: *The AI Engineering Stack: Layers for Teams*.
