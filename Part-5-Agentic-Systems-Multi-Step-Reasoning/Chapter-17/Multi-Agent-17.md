# Chapter 17: Multi-Agent Systems

## Introduction: The Single-Agent Ceiling

In the previous chapter, we learned how to build a single autonomous agent. However, as your system's complexity grows, you will inevitably hit the **Single-Agent Ceiling**. This occurs when one agent is given too many tools (e.g., more than 10-15), leading to a significant drop in "Tool Selection Accuracy." The model spends so many tokens reasoning about *which* tool to use that it has no "Reasoning Budget" left to actually use them.

In 2026, the solution is **Multi-Agent Systems (MAS)**. Instead of one "God Agent," we build teams of specialized agents that collaborate. This modular approach is the foundation of industrial-scale AI engineering.

---

## Deep Technical Analysis: Multi-Agent Coordination Patterns

The shift from "Single-Agent" to "Multi-Agent" is built on three technical pillars:

### 1. Context Isolation (Noise Reduction)
Every tool definition added to an agent consumes 200-500 tokens of the context window. A single agent with 20 tools wastes 4,000-10,000 tokens before the user even speaks. In a Multi-Agent system, each specialist only "sees" the 3 tools it needs. This keeps the prompt high-signal and the model's attention focused.

### 2. Hierarchical vs. Network Orchestration
-   **Hierarchical (Supervisor):** A "Manager" agent analyzes the goal and delegates tasks to "Worker" agents. Workers report back to the Manager. This is the most stable pattern for production.
-   **Network (Collaborative):** Agents talk to each other directly without a manager. While flexible, this pattern is prone to "Infinite Loops" and is harder to debug.

### 3. Shared State vs. Message Passing
How do agents share information?
-   **Shared State:** A central "State Object" (e.g., in LangGraph) that all agents can read and write to.
-   **Message Passing:** Agents send discrete messages to each other.
In 2026, **Shared State** is the preferred pattern for engineering because it allows for easy "Check-pointing" and "Time-Travel Debugging" (seeing exactly what the state was at step 4).

---

## Why Multi-Agent Systems Solve Real-World Problems

In practice, MAS solves several critical production issues:
-   **Domain Specialization:** You can use a large, expensive model (GPT-4o) for the "Supervisor" and small, fast models (Llama-3-8B) for "Workers," optimizing for both quality and cost.
-   **Parallel Processing:** While the "Researcher Agent" is browsing the web, the "Coder Agent" can be writing the boilerplate, and the "Writer Agent" can be drafting the introduction.
-   **Resilience:** If the "Researcher" fails to find data, the "Supervisor" can decide to try a different research specialist or ask the user for clarification, without the whole system crashing.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to build multi-agent systems using modern patterns and the **LangGraph** framework.

### Example 1: The "Supervisor" Router Pattern
**Problem:** A user query might require either a "SQL expert" or a "Web expert."
**Solution:** Use a Supervisor agent to route the query to the correct specialist.

```python
from typing import Literal
from pydantic import BaseModel

class Route(BaseModel):
    next_agent: Literal["SQL_EXPERT", "WEB_EXPERT", "FINISH"]

# Supervisor Prompt: "Based on the user query, who should act next?"
# If user says "What's in the DB?", route to SQL_EXPERT.
```
**Why this is preferred:** It prevents "Tool Confusion." The SQL expert never even sees the Web search tools, ensuring it stays focused on writing perfect SQL.

---

### Example 2: Sub-Agents as Tools
**Problem:** You want an agent to "Research and Write" a report.
**Solution:** Wrap the "Research Agent" as a Python function (a Tool) and give it to the "Writer Agent."

```python
def research_tool(query: str):
    # This function triggers a separate, internal agent loop
    # and returns a summarized string.
    return research_agent.run(query)

# The 'Writer Agent' just sees a single tool called 'get_research_data'
```
**Why this is preferred:** It is the **simplest way to scale**. It allows you to build complex nested logic while keeping the top-level agent's context window clean.

---

### Example 3: Multi-Agent "Debate" (Consensus Pattern)
**Problem:** A single LLM call for a high-stakes decision (e.g. medical diagnosis) might be biased or wrong.
**Solution:** Have two agents argue for different viewpoints and a third "Judge" agent decide the winner.

```python
# Agent A: "The code is secure because X."
# Agent B: "The code is insecure because Y."
# Judge: "Based on both arguments, the code needs a fix for Y."
```
**Why this is preferred:** It increases the **Accuracy Floor**. Research shows that "Multi-Agent Debate" significantly reduces hallucinations in logical reasoning tasks.

---

### Example 4: Shared State Management (LangGraph)
**Problem:** Agents need to build on each other's work without losing information.
**Solution:** Use a TypedDict to maintain a global "State" that all agents update.

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # 'add_messages' ensures history is appended, not overwritten
    messages: Annotated[list, add_messages]
    research_notes: str
    is_complete: bool
```
**Why this is preferred:** It provides **Auditability**. You can inspect the `AgentState` at any point in the process to see which agent added which piece of information.

---

### Example 5: The "Critic" Loop Pattern
**Problem:** A "Coder Agent" often writes code that has syntax errors.
**Solution:** Add a "Reviewer Agent" that runs the code and provides feedback to the Coder.

```python
def reviewer_node(state: AgentState):
    # 1. Extract code from state
    # 2. Run 'pytest' or 'pylint'
    # 3. If errors: add error msg to state and route back to 'CODER'
    # 4. If success: route to 'FINISH'
```
**Why this is preferred:** It automates **Quality Assurance**. The user never sees the broken "First Draft" of the code; they only see the "Final, Verified" version.

---

### Example 6: Heterogeneous Model Orchestration
**Problem:** Using GPT-4 for simple data cleaning is a waste of money.
**Solution:** Use GPT-4 for the "Supervisor" and GPT-4o-mini for the "Data Cleaning" workers.

```python
# supervisor_llm = ChatOpenAI(model="gpt-4o")
# worker_llm = ChatOpenAI(model="gpt-4o-mini")

# In the graph:
# Node 'Manager' uses supervisor_llm
# Node 'Cleaner' uses worker_llm
```
**Why this is preferred:** It provides **Production ROI**. It allows you to spend your "Intelligence Budget" exactly where it's needed most (high-level planning) while using cheaper compute for repetitive tasks.

---

### Example 7: Parallel Multi-Agent Execution
**Problem:** Running a "Market Research" agent and a "Legal Review" agent sequentially takes 30 seconds.
**Solution:** Trigger both nodes simultaneously in a LangGraph and "Join" them at a "Consolidator" node.

```python
# Graph:
# START -> [ResearchNode, LegalNode] (Parallel)
# [ResearchNode, LegalNode] -> ConsolidatorNode
```
**Why this is preferred:** It optimizes for **User-Perceived Latency**. The user gets a comprehensive report in 15 seconds instead of 30.

---

### Example 8: Handling Agentic "Infinite Loops"
**Problem:** Two agents keep passing a task back and forth without finishing (e.g. A: "Fix this", B: "I fixed it", A: "No you didn't").
**Solution:** Implement a "Recursion Limit" and a "Loop Monitor" in the orchestration layer.

```python
def check_recursion(state: AgentState):
    if len(state['messages']) > 20:
        return "human_intervention" # Halt and ask user
    return "continue"
```
**Why this is preferred:** It provides **Operational Stability**. It prevents a single "confused" request from burning through your entire API budget in a loop.

---

## Conclusion: The Power of Teams

Multi-agent systems represent the move from "Chatting with an AI" to "Managing an AI Workforce." By specializing your agents, isolating their contexts, and coordinating them with a robust shared state, you can solve problems that are orders of magnitude more complex than what a single prompt could ever handle.

In the next chapter, we will look at **Long-Horizon Learning Systems**, where these agents learn and improve over days and weeks, rather than just seconds.

---

## References & Further Reading
- **Klement Gunndu (2026)**: *Build Your First Multi-Agent System in Python: 3 Patterns That Scale*.
- **LangGraph Documentation**: *Multi-Agent Workflows and Coordination*.
- **Wu et al. (2023)**: *AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation*.
- **CrewAI**: *Orchestrating Role-Based Autonomous AI Agents*.
- **DeepLearning.AI**: *Multi-Agent Systems with LangGraph*.
