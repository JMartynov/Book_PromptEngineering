# Chapter 16: From Prompts to Agents

## Introduction: The "Goal-Oriented" Shift

In the first half of this book, we've focused on "Input -> Output" systems. You give the LLM a prompt, and it gives you a response. This is a "Chatbot" model. But in 2026, the real value is in **AI Agents**.

An AI Agent is a system that isn't just a smarter chatbot; it is a **Goal-Oriented System**. You give it an objective (e.g., "Research the competitors of our new product and email a 3-bullet summary to the CEO"), and it plans the steps, selects the right tools (web search, email API), evaluates its own progress, and executes multiple sub-tasks autonomously to reach that goal.

---

## Deep Technical Analysis: The Agentic Architecture

The shift from "Static Prompts" to "Dynamic Agents" is built on four architectural layers:

### 1. The Planner (The Pre-frontal Cortex)
Instead of executing a prompt immediately, the agent uses a **Planning Module** to decompose the high-level goal into a directed graph of sub-tasks. Research into **Plan-and-Execute** patterns has shown that separate "Planning" and "Execution" steps reduce hallucinations by 30% because the model isn't trying to do two things (reasoning and acting) at once.

### 2. The Tool Interface (The Effectors)
Agents use **Function Calling** to interact with the world. In 2026, we treat tools as "Skills." Each tool is defined with a strict JSON schema (Pydantic model) that describes what it does and what arguments it needs. The LLM acts as the "Dispatcher," deciding which skill to invoke based on the current state of the plan.

### 3. Persistent Memory (Episodic & Long-Term)
A standard prompt is "Stateless." An agent is **Stateful**. It maintains:
-   **Short-Term Memory:** The history of the current task (thoughts, tool results).
-   **Long-Term Memory:** Learnings from previous tasks, stored in a Vector DB.
Modern agents use **Reflection** to summarize their short-term memory into long-term insights, preventing the context window from becoming overloaded.

### 4. The Executor/Critic Loop (Self-Correction)
This is the "Engine" of autonomy. The agent follows the **ReAct** (Reason + Act) pattern. After each action, it "Observes" the result and "Critiques" whether it is closer to the goal. If a tool fails (e.g., a 404 error from a website), the Critic triggers a "Re-planning" step to find an alternative path.

---

## Why Agents Solve Real-World Problems

In practice, AI Agents solve several critical production issues:
-   **Handling Open-Ended Tasks:** You can't write a single prompt for "Fix this bug in the codebase." An agent can browse the files, run tests, identify the error, and write the fix.
-   **Tool Interoperability:** Agents act as the "Natural Language API" for your entire software stack, connecting your CRM, database, and email server without needing 100 separate manual integrations.
-   **Cost-Efficient Autonomy:** Instead of a human spending 2 hours on a task, an agent can do it in 2 minutes for $0.50.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to build agentic systems using modern patterns and frameworks like LangChain and PydanticAI.

### Example 1: The "Hierarchical Planner"
**Problem:** A user goal is too complex for a single step.
**Solution:** A prompt that forces the LLM to output a structured, multi-step "Mission Plan" as its first action.

```python
from pydantic import BaseModel
from typing import List

class Task(BaseModel):
    id: int
    action: str
    tool_required: str

class MissionPlan(BaseModel):
    goal: str
    steps: List[Task]

# Prompt: "Decompose this goal: 'Audit our S3 buckets for public access' into a JSON plan."
```
**Why this is preferred:** It provides **Structural Guidance**. By committing to a plan before acting, the agent is less likely to wander off-topic or get stuck in a loop.

---

### Example 2: Typed Tool Definitions (The "Skill" Pattern)
**Problem:** The LLM often calls tools with the wrong arguments (e.g., passing a string where an int is needed).
**Solution:** Define tools using Pydantic models to ensure the LLM follows a strict schema.

```python
from pydantic import Field

def get_user_data(user_id: int = Field(..., description="The numeric ID of the user")):
    """Fetches user profile from the SQL database."""
    # (Database logic...)
    return {"id": user_id, "status": "active"}

# In 2026, we pass the 'get_user_data' schema directly to the agent.
```
**Why this is preferred:** It provides **Type Safety** at the model boundary. If the LLM tries to pass `user_id="abc"`, the system catches the error before the database call is even made.

---

### Example 3: The ReAct Loop (Reasoning in Public)
**Problem:** "Hidden Reasoning" makes agents hard to debug. You don't know why they chose Tool A over Tool B.
**Solution:** Force the agent to output a "Thought" block before every "Action" block.

```python
react_template = """
GOAL: {goal}
THOUGHT: <explain why you are taking the next step>
ACTION: <tool_name>(<args>)
OBSERVATION: <result from tool>
...
"""
```
**Why this is preferred:** It creates an **Audit Trail**. If the agent makes a mistake, you can read the "Thought" to see where its logic diverged from reality.

---

### Example 4: Agentic Self-Correction (The "Critic")
**Problem:** An agent might finish a task but provide a low-quality or incorrect result.
**Solution:** Add a "Critic Node" that reviews the final output against the original goal.

```python
def critic_node(goal, final_output):
    # Prompt: "Does this output satisfy the goal: '{goal}'? YES or NO. If NO, list why."
    # If the critic says NO, the agent is sent back to the 'Planner' node.
    pass
```
**Why this is preferred:** It increases the **Accuracy Floor** of the system. It's the difference between "I'm done" and "I've verified that I'm done correctly."

---

### Example 5: Episodic Memory with "Thread IDs"
**Problem:** The agent "forgets" what it did in a previous session, forcing the user to repeat themselves.
**Solution:** Store task trajectories in a database indexed by `thread_id` and inject them into the next session's context.

```python
def load_agent_memory(thread_id: str):
    # Fetch from Vector DB or Postgres
    # past_actions = db.query(f"SELECT * FROM memory WHERE thread_id={thread_id}")
    pass
```
**Why this is preferred:** It enables **Long-Horizon Support**. The agent can say "As we discussed yesterday, I've already checked the logs for server-01."

---

### Example 6: "Plan-and-Execute" (Decoupled Architecture)
**Problem:** In a ReAct loop, the model often forgets the original goal after 5 tool calls.
**Solution:** Use a "Master Planner" that stays fixed and an "Executor" that handles the current step.

```python
# Node 1 (Planner): "Current status: Step 2 of 5 complete."
# Node 2 (Executor): "Executing Step 3: Fetching API data."
# Node 3 (Planner): "Step 3 complete. Updating plan for Step 4."
```
**Why this is preferred:** It maintains **Global Context**. The Planner acts as the "Manager" ensuring the Executor stays on track to the ultimate goal.

---

### Example 7: Handling "Human Interrupts" (Governance)
**Problem:** You don't want an autonomous agent to "Delete all files" without a human double-checking.
**Solution:** Implement a "Human-in-the-Loop" (HITL) state in your agentic graph.

```python
def delete_files_tool(path):
    # This tool has an 'approval_required' flag
    # The framework pauses and waits for user.approve()
    pass
```
**Why this is preferred:** It provides **Safety Guardrails**. It allows for the efficiency of automation while retaining human control over high-risk actions.

---

### Example 8: Basic "Manager-Worker" Delegation
**Problem:** A single agent trying to be an expert in everything (SQL, Coding, Writing) becomes mediocre at all of them.
**Solution:** Use a "Manager Agent" to delegate tasks to specialized "Worker Agents."

```python
def manager_agent(task):
    if "code" in task:
        return call_worker("CODER_AGENT", task)
    elif "sql" in task:
        return call_worker("DATA_AGENT", task)
```
**Why this is preferred:** It enables **Domain Specialization**. You can use a smaller, faster model for the "Data Worker" and a larger, more capable model for the "Manager."

---

## Conclusion: The Agentic Future

The shift from prompts to agents is a move from "Passive" to "Active" AI. By mastering the 4 pillars of agentic architecture—Planning, Tools, Memory, and Critique—you can build systems that don't just "Talk," but actually **"Work."**

In the next chapter, we will dive deeper into **Multi-Agent Systems**, where teams of specialized AI work together to solve massive problems.

---

## References & Further Reading
- **Mortex Solutions (2026)**: *AI Agents in 2026: A Guide to Autonomous Systems*.
- **LangChain Docs**: *AgentExecutor and Tool Use Patterns*.
- **PydanticAI Docs**: *Building Typed and Verified Agents*.
- **CrewAI**: *Multi-Agent Orchestration Framework*.
- **DeepLearning.AI**: *AI Agents Specialization*.
