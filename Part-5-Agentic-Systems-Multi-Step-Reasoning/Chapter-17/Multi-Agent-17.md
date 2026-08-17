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

These production-grade examples demonstrate how to build modular multi-agent systems using supervisor routing, encapsulated sub-agent tools, consensus debate loops, shared state reducers, automated QA critique cycles, parallel fan-out/fan-in execution, and infinite-loop circuit breakers.

### Example 1: The "Supervisor" Router Pattern
**Problem:** A monolithic agent given tools for both SQL querying and live Web browsing experiences high tool hallucination rates and prompt clutter.
**Solution:** Implement a Supervisor Agent that classifies the user goal and routes execution exclusively to a specialized worker with an isolated toolset.

```python
from typing import Callable, Dict, Literal
from pydantic import BaseModel, Field


class RoutingDecision(BaseModel):
    """Structured routing classification returned by the Supervisor."""
    next_specialist: Literal["SQL_EXPERT", "WEB_EXPERT", "FINISH"] = Field(
        ..., description="The chosen specialist or completion signal"
    )
    justification: str = Field(..., description="Engineering reason for the routing choice")


class SqlExpertWorker:
    """Specialist agent with tools limited strictly to database querying."""
    def run(self, query: str) -> str:
        # Simulated database query execution
        return f"[SqlExpert] Executed schema-validated query for: '{query}'. Returned 42 rows."


class WebExpertWorker:
    """Specialist agent with tools limited strictly to web retrieval."""
    def run(self, query: str) -> str:
        # Simulated live search & scraping
        return f"[WebExpert] Retrieved and synthesized latest market benchmarks for: '{query}'."


class MultiAgentSupervisor:
    """Supervisor orchestrator managing domain-specific worker execution."""

    def __init__(self):
        self.sql_worker = SqlExpertWorker()
        self.web_worker = WebExpertWorker()

    def route_query(self, query: str) -> RoutingDecision:
        """Determines the appropriate specialist based on input heuristics / LLM classification."""
        lowered = query.lower()
        if any(keyword in lowered for keyword in ["select", "table", "rows", "database", "orders", "revenue"]):
            return RoutingDecision(
                next_specialist="SQL_EXPERT",
                justification="Query involves internal tabular data and transactional records."
            )
        elif any(keyword in lowered for keyword in ["news", "market", "competitor", "trends", "search"]):
            return RoutingDecision(
                next_specialist="WEB_EXPERT",
                justification="Query requires external real-time web intelligence."
            )
        else:
            return RoutingDecision(
                next_specialist="FINISH",
                justification="Query is conversational or does not require external specialist tools."
            )

    def process(self, query: str) -> Dict[str, str]:
        decision = self.route_query(query)
        print(f"[Supervisor Decision] -> {decision.next_specialist} (Reason: {decision.justification})")

        if decision.next_specialist == "SQL_EXPERT":
            result = self.sql_worker.run(query)
        elif decision.next_specialist == "WEB_EXPERT":
            result = self.web_worker.run(query)
        else:
            result = "Supervisor answered directly: Request fulfilled without tool invocation."

        return {"query": query, "specialist": decision.next_specialist, "result": result}


if __name__ == "__main__":
    supervisor = MultiAgentSupervisor()

    # Query 1: Routed to SQL Specialist
    res1 = supervisor.process("Fetch monthly revenue by region from the orders table")
    print(f"Output: {res1['result']}\n")

    # Query 2: Routed to Web Specialist
    res2 = supervisor.process("Find latest 2026 enterprise cloud pricing trends")
    print(f"Output: {res2['result']}\n")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` (`BaseModel`, `Field`) for typed routing decisions and standard Python dispatching.
- **How It Works:** The supervisor analyzes the incoming request and generates a `RoutingDecision`. The request is dispatched exclusively to the selected worker, ensuring that the SQL worker never sees web-search tools and vice versa.
- **Expected Output:** Requests are dispatched to the appropriate specialized worker with clean context isolation.
- **Why This Approach:** Restricting tool access to 2–4 tools per specialist drastically improves tool-call accuracy and keeps prompt token overhead low.

---

### Example 2: Sub-Agents as Tools (Hierarchical Tool Pattern)
**Problem:** A top-level writer agent needs deep multi-step research, but handling browsing, HTML scraping, and summarization directly explodes its context window.
**Solution:** Encapsulate an entire multi-step research agent inside a single callable tool function presented to the primary writer agent.

```python
from typing import Any, Dict
from pydantic import BaseModel, Field


class ResearchResult(BaseModel):
    topic: str
    key_findings: list[str]
    citations: list[str]


class DeepResearchSubAgent:
    """An autonomous sub-agent that executes a multi-step research cycle."""

    def execute_research(self, topic: str) -> ResearchResult:
        # Step 1: Query planning
        # Step 2: Web fetching & scraping
        # Step 3: Synthesis & citation validation
        return ResearchResult(
            topic=topic,
            key_findings=[
                "Agentic architectures reduce human workflow intervention by up to 70%.",
                "Context isolation prevents tool-selection drift in agents with >10 tools.",
                "Pydantic v2 schemas improve parameter extraction reliability to 99.4%."
            ],
            citations=["https://arxiv.org/abs/2210.03629", "https://docs.pydantic.dev"]
        )


def deep_research_tool(topic: str) -> str:
    """
    Exposed tool interface for top-level orchestrators.
    Hides internal multi-step research complexity from the calling agent.
    """
    sub_agent = DeepResearchSubAgent()
    result = sub_agent.execute_research(topic)
    
    formatted_summary = (
        f"=== Deep Research Summary for: {result.topic} ===\n"
        + "\n".join(f"- {f}" for f in result.key_findings)
        + "\nCitations:\n"
        + "\n".join(f"  * {c}" for c in result.citations)
    )
    return formatted_summary


class ExecutiveWriterAgent:
    """High-level agent with access only to the consolidated research tool."""

    def __init__(self):
        self.available_tools = {"deep_research": deep_research_tool}

    def draft_executive_brief(self, subject: str) -> str:
        print(f"[ExecutiveWriter] Delegating deep research for '{subject}' to sub-agent tool...")
        research_brief = self.available_tools["deep_research"](subject)
        
        # High-level synthesis
        return (
            f"# Executive Briefing: {subject}\n\n"
            f"{research_brief}\n\n"
            f"**Strategic Takeaway:** Modernize enterprise workflows with multi-agent specialization."
        )


if __name__ == "__main__":
    writer = ExecutiveWriterAgent()
    brief = writer.draft_executive_brief("Autonomous AI Systems in 2026")
    print(brief)
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for structured data aggregation.
- **How It Works:** The sub-agent encapsulates internal complexity (search queries, scraping, parsing). The parent agent interacts with it as a simple function returning a synthesized summary.
- **Expected Output:** A cleanly formatted executive brief generated without cluttering the parent agent's reasoning loop with low-level scraping steps.
- **Why This Approach:** Encapsulation hides intermediate tokens and failure retries from the supervisor, preserving top-level context budget.

---

### Example 3: Multi-Agent "Debate" (Consensus Pattern)
**Problem:** A single model can have subtle blind spots, biased assumptions, or hallucinations on critical code security reviews.
**Solution:** Run a multi-agent debate where a Security Auditor and a Performance Auditor provide competing critiques, and an impartial Judge reconciles differences into a final verdict.

```python
from typing import Dict, List
from pydantic import BaseModel, Field


class AuditorReview(BaseModel):
    auditor_role: str
    identified_issues: List[str]
    risk_score: int = Field(..., ge=1, le=10, description="Risk score from 1 (lowest) to 10 (highest)")
    recommendation: str


class ConsensusVerdict(BaseModel):
    final_status: str
    approved_for_release: bool
    reconciled_findings: List[str]
    required_actions: List[str]


class SecurityAuditor:
    def audit(self, code: str) -> AuditorReview:
        has_eval = "eval(" in code
        return AuditorReview(
            auditor_role="Security Auditor",
            identified_issues=["Use of dangerous eval() function allows arbitrary code execution."] if has_eval else [],
            risk_score=9 if has_eval else 1,
            recommendation="Replace eval() with safe JSON deserialization." if has_eval else "Security check passed."
        )


class PerformanceAuditor:
    def audit(self, code: str) -> AuditorReview:
        has_nested_loops = "for " in code and code.count("for ") > 1
        return AuditorReview(
            auditor_role="Performance Auditor",
            identified_issues=["O(n^2) nested loop detected on unbounded dataset."] if has_nested_loops else [],
            risk_score=6 if has_nested_loops else 2,
            recommendation="Vectorize operations or use hash lookups." if has_nested_loops else "Performance check passed."
        )


class ConsensusJudge:
    def adjudicate(self, security: AuditorReview, performance: AuditorReview) -> ConsensusVerdict:
        critical_failure = security.risk_score >= 7 or performance.risk_score >= 8
        findings = security.identified_issues + performance.identified_issues
        actions = []
        if security.risk_score > 3:
            actions.append(f"Security: {security.recommendation}")
        if performance.risk_score > 3:
            actions.append(f"Performance: {performance.recommendation}")

        return ConsensusVerdict(
            final_status="REJECTED" if critical_failure else "APPROVED",
            approved_for_release=not critical_failure,
            reconciled_findings=findings if findings else ["All audits passed successfully."],
            required_actions=actions if actions else ["Proceed to staging deployment."]
        )


def run_code_governance_pipeline(code_snippet: str) -> ConsensusVerdict:
    sec_agent = SecurityAuditor()
    perf_agent = PerformanceAuditor()
    judge = ConsensusJudge()

    print("[Pipeline] Running independent specialized audits...")
    sec_report = sec_agent.audit(code_snippet)
    perf_report = perf_agent.audit(code_snippet)

    print(f"  - Security Risk: {sec_report.risk_score}/10 | Issues: {sec_report.identified_issues}")
    print(f"  - Performance Risk: {perf_report.risk_score}/10 | Issues: {perf_report.identified_issues}")

    verdict = judge.adjudicate(sec_report, perf_report)
    return verdict


if __name__ == "__main__":
    vulnerable_code = """
def parse_payload(user_input):
    for item in user_input:
        for sub in item['data']:
            result = eval(sub)
    return result
    """
    verdict = run_code_governance_pipeline(vulnerable_code)
    print(f"\n[Final Verdict] Status: {verdict.final_status} (Approved: {verdict.approved_for_release})")
    print(f"Reconciled Issues: {verdict.reconciled_findings}")
    print(f"Mandatory Actions: {verdict.required_actions}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for structured audit and verdict models.
- **How It Works:** Independent specialist agents inspect the payload through distinct domain lenses (security vs performance) without biasing each other. The Judge aggregates the evaluations using deterministic threshold rules.
- **Expected Output:** A comprehensive, reconciled governance verdict with itemized actions.
- **Why This Approach:** Prevents blind spots; a model tasked with checking performance will often overlook severe security flaws unless roles are explicitly separated.

---

### Example 4: Shared State Management (State Reducer & Checkpointing)
**Problem:** In complex multi-agent graphs, state mutations by different agents can overwrite data, introduce race conditions, or prevent debugging past steps.
**Solution:** Implement an immutable, appending state reducer with deterministic checkpointing and snapshot history.

```python
import copy
from datetime import datetime, timezone
from typing import Any, Dict, List
from pydantic import BaseModel, Field


class AgentMessage(BaseModel):
    sender: str
    content: str
    timestamp: str = Field(default_factory=lambda: datetime.now(timezone.utc).isoformat())


class TeamState(BaseModel):
    """Shared state container with immutable history and tracking flags."""
    session_id: str
    messages: List[AgentMessage] = Field(default_factory=list)
    shared_artifacts: Dict[str, Any] = Field(default_factory=dict)
    is_completed: bool = False
    current_step: int = 0


class StateManager:
    """State orchestrator supporting updates, snapshotting, and time-travel rollback."""

    def __init__(self, initial_session_id: str):
        self.state = TeamState(session_id=initial_session_id)
        self.history: List[TeamState] = []
        self._save_checkpoint()

    def _save_checkpoint(self):
        """Stores deep copy snapshot for time-travel debugging."""
        self.history.append(copy.deepcopy(self.state))

    def append_message(self, sender: str, content: str) -> None:
        """Appends a new message to the shared timeline and increments step."""
        msg = AgentMessage(sender=sender, content=content)
        self.state.messages.append(msg)
        self.state.current_step += 1
        self._save_checkpoint()

    def update_artifact(self, key: str, value: Any) -> None:
        """Adds or modifies a shared data artifact."""
        self.state.shared_artifacts[key] = value
        self.state.current_step += 1
        self._save_checkpoint()

    def rollback_to_step(self, step_number: int) -> TeamState:
        """Rolls back the system to an earlier snapshot."""
        if 0 <= step_number < len(self.history):
            self.state = copy.deepcopy(self.history[step_number])
            print(f"[StateManager] Rolled back state to Step {step_number}.")
            return self.state
        raise ValueError("Invalid step number for rollback.")


if __name__ == "__main__":
    manager = StateManager(initial_session_id="session-graph-101")

    # Agent 1 (Planner) updates state
    manager.append_message("PlannerAgent", "Decomposed mission into 3 sub-tasks.")
    manager.update_artifact("plan", ["task_1", "task_2", "task_3"])

    # Agent 2 (Worker) updates state
    manager.append_message("WorkerAgent", "Completed task_1: S3 scanning done.")
    manager.update_artifact("s3_results", {"public_buckets": 0})

    print(f"Current Step: {manager.state.current_step}")
    print(f"Artifacts: {list(manager.state.shared_artifacts.keys())}")
    print(f"Total Checkpoints: {len(manager.history)}")

    # Time-travel rollback demonstration
    manager.rollback_to_step(1)
    print(f"Artifacts after Rollback to Step 1: {list(manager.state.shared_artifacts.keys())}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` and `copy` for immutable state cloning and validation.
- **How It Works:** Every state modification increments a logical clock (`current_step`) and creates a checkpoint in `history`. Agents only interact via explicit reducer methods.
- **Expected Output:** Deterministic step-by-step state records with instant rollback capability.
- **Why This Approach:** Mimics enterprise state frameworks like LangGraph and Redux, enabling observability, auditability, and recovery from failed branches.

---

### Example 5: The "Critic" Loop Pattern (Coder-Reviewer Cycle)
**Problem:** AI code generation agents frequently produce subtle syntax errors, undefined variables, or unhandled exceptions in first drafts.
**Solution:** Build an automated QA critique loop where a Reviewer Agent parses and executes static checks against generated code, requesting repairs until the code compiles cleanly.

```python
import ast
from typing import Dict, List, Tuple
from pydantic import BaseModel, Field


class LintReport(BaseModel):
    is_valid_syntax: bool
    syntax_error: str = ""
    defined_functions: List[str] = Field(default_factory=list)


class CoderAgent:
    """Simulates a code generation worker."""
    def generate(self, prompt: str, feedback: str = "") -> str:
        if not feedback:
            # First draft with intentional syntax flaw (missing colon)
            return "def calculate_discount(price, rate)\n    return price * (1 - rate)"
        else:
            # Corrected version after receiving reviewer feedback
            return "def calculate_discount(price: float, rate: float) -> float:\n    return price * (1 - rate)"


class CodeReviewerAgent:
    """Performs AST validation and static checking on generated code."""
    def review(self, code_str: str) -> LintReport:
        try:
            parsed = ast.parse(code_str)
            functions = [node.name for node in ast.walk(parsed) if isinstance(node, ast.FunctionDef)]
            return LintReport(is_valid_syntax=True, defined_functions=functions)
        except SyntaxError as err:
            return LintReport(is_valid_syntax=False, syntax_error=f"SyntaxError at line {err.lineno}: {err.msg}")


def run_coder_reviewer_loop(prompt: str, max_retries: int = 3) -> str:
    coder = CoderAgent()
    reviewer = CodeReviewerAgent()

    feedback = ""
    for attempt in range(1, max_retries + 1):
        print(f"\n--- [Cycle {attempt}] Generating Code ---")
        code = coder.generate(prompt, feedback)
        print(f"Generated Code:\n{code}")

        report = reviewer.review(code)
        if report.is_valid_syntax:
            print(f"[Reviewer] APPROVED: Syntax is valid. Functions defined: {report.defined_functions}")
            return code

        print(f"[Reviewer] REJECTED: {report.syntax_error}")
        feedback = f"Fix the syntax error: {report.syntax_error}"

    raise RuntimeError("Failed to produce valid code within retry limits.")


if __name__ == "__main__":
    valid_code = run_coder_reviewer_loop("Write a discount calculator function in Python.")
    print(f"\nFinal Verified Output:\n{valid_code}")
```

**Developer Explanation:**
- **Libraries Used:** Python built-in `ast` module for syntax tree compilation and `pydantic` for structured lint reporting.
- **How It Works:** The Reviewer compiles the Coder's output using Python's Abstract Syntax Tree parser. If a `SyntaxError` is raised, line number and error message are passed back to the Coder for immediate repair.
- **Expected Output:** Broken drafts are automatically repaired before being exposed to end users.
- **Why This Approach:** Eliminates syntax bugs and broken scripts before production deployment.

---

### Example 6: Heterogeneous Model Orchestration
**Problem:** Sending simple data formatting or extraction requests to top-tier flagship LLMs is cost-inefficient and slow.
**Solution:** Implement multi-tier model orchestration, routing complex reasoning to a high-capability model and high-volume parsing to an ultra-fast, cost-effective model.

```python
from typing import Any, Dict
from pydantic import BaseModel, Field


class ModelProfile(BaseModel):
    name: str
    cost_per_1k_tokens: float
    tier: str


class ModelRegistry:
    TIER_1_REASONING = ModelProfile(name="gpt-4o / claude-3-5-sonnet", cost_per_1k_tokens=0.015, tier="Tier 1 High-Reasoning")
    TIER_2_FAST = ModelProfile(name="gpt-4o-mini / claude-3-haiku", cost_per_1k_tokens=0.0006, tier="Tier 2 Fast-Utility")


class HeterogeneousOrchestrator:
    """Dispatches tasks to the most cost-effective model tier based on task complexity."""

    def __init__(self):
        self.total_cost = 0.0

    def execute_task(self, task_type: str, payload: str) -> Dict[str, Any]:
        if task_type in ["system_architecture", "root_cause_analysis", "legal_contract_review"]:
            selected_model = ModelRegistry.TIER_1_REASONING
            tokens_used = 1200
            result = f"[{selected_model.name}] Completed deep cognitive synthesis for: '{payload[:30]}...'"
        else:
            selected_model = ModelRegistry.TIER_2_FAST
            tokens_used = 400
            result = f"[{selected_model.name}] Parsed and extracted structured entities for: '{payload[:30]}...'"

        cost = (tokens_used / 1000.0) * selected_model.cost_per_1k_tokens
        self.total_cost += cost

        return {
            "task_type": task_type,
            "model_used": selected_model.name,
            "tier": selected_model.tier,
            "tokens": tokens_used,
            "estimated_cost_usd": round(cost, 6),
            "result": result
        }


if __name__ == "__main__":
    orchestrator = HeterogeneousOrchestrator()

    # High-reasoning task
    job1 = orchestrator.execute_task("system_architecture", "Design zero-trust multi-region failover topology")
    print(f"Task 1 -> {job1['tier']} | Cost: ${job1['estimated_cost_usd']}")

    # Low-complexity data parsing task
    job2 = orchestrator.execute_task("json_formatting", "Format customer list to standardized schema")
    print(f"Task 2 -> {job2['tier']} | Cost: ${job2['estimated_cost_usd']}")

    print(f"\nTotal Pipeline Cost: ${round(orchestrator.total_cost, 6)}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for model configuration typing.
- **How It Works:** The orchestrator inspects the task classification. High-stakes architecture or root-cause tasks are assigned to Tier-1 models, while utility transformation tasks are routed to Tier-2 models (25x cheaper).
- **Expected Output:** Accurate cost calculation and optimal model tier selection.
- **Why This Approach:** Achieves top-tier intelligence where it matters while reducing overall enterprise inference budgets by 60–80%.

---

### Example 7: Parallel Multi-Agent Execution (Scatter-Gather)
**Problem:** Executing market research, compliance verification, and financial analysis sequentially creates unacceptable user-facing latency.
**Solution:** Use asynchronous parallel execution (`asyncio.gather`) to fan-out sub-agent tasks concurrently, gathering results in a consolidator node.

```python
import asyncio
from typing import Dict, List
from pydantic import BaseModel


class SpecialistReport(BaseModel):
    specialist: str
    latency_seconds: float
    summary: str


async def market_research_worker() -> SpecialistReport:
    # Simulates external API calls
    await asyncio.sleep(0.1)
    return SpecialistReport(
        specialist="Market Analyst",
        latency_seconds=0.1,
        summary="Demand in enterprise AI tooling projected to grow 45% YoY in 2026."
    )


async def compliance_worker() -> SpecialistReport:
    await asyncio.sleep(0.1)
    return SpecialistReport(
        specialist="Compliance Officer",
        latency_seconds=0.1,
        summary="EU AI Act Tier-2 risk assessment complete: logging and audit controls required."
    )


async def finance_worker() -> SpecialistReport:
    await asyncio.sleep(0.1)
    return SpecialistReport(
        specialist="Financial Modeler",
        latency_seconds=0.1,
        summary="Projected ROI: Break-even within 3.5 months with 4.2x operational yield."
    )


async def run_parallel_multi_agent_workflow(mission: str) -> Dict[str, Any]:
    print(f"=== Starting Scatter-Gather Workflow: '{mission}' ===")
    start_time = asyncio.get_event_loop().time()

    # Scatter: execute all 3 specialists concurrently
    reports: List[SpecialistReport] = await asyncio.gather(
        market_research_worker(),
        compliance_worker(),
        finance_worker()
    )

    elapsed = asyncio.get_event_loop().time() - start_time
    print(f"[Gather] All 3 agents completed in parallel ({round(elapsed, 3)}s elapsed)")

    # Gather & Consolidate
    consolidated_findings = {r.specialist: r.summary for r in reports}
    return {
        "mission": mission,
        "elapsed_time_seconds": round(elapsed, 3),
        "consolidated_report": consolidated_findings
    }


if __name__ == "__main__":
    result = asyncio.run(run_parallel_multi_agent_workflow("New Product Launch Feasibility"))
    print("\nConsolidated Report:")
    for role, summary in result["consolidated_report"].items():
        print(f"  * [{role}]: {summary}")
```

**Developer Explanation:**
- **Libraries Used:** `asyncio` for non-blocking asynchronous concurrency, `pydantic` for report schemas.
- **How It Works:** `asyncio.gather` fires off multiple independent specialist workers simultaneously. The parent process pauses only until the slowest specialist finishes, then aggregates all responses.
- **Expected Output:** Concurrent execution finishing in roughly the duration of a single agent call rather than the sum of all three.
- **Why This Approach:** Cuts user-perceived latency by 60–70% for multi-domain advisory pipelines.

---

### Example 8: Handling Agentic "Infinite Loops" (Circuit Breakers)
**Problem:** Multi-agent dialogue can devolve into infinite ping-pong loops (Agent A rejects B, B resubmits identical data, A rejects again) burning API tokens.
**Solution:** Implement a Circuit Breaker with repetition detection (hash tracking) and hard step recursion limits.

```python
import hashlib
from typing import List, Optional
from pydantic import BaseModel, Field


class AgentActionRecord(BaseModel):
    step: int
    agent_name: str
    payload_hash: str


class CircuitBreaker:
    """Monitors agent trajectories to detect and terminate infinite loops and ping-pong cycles."""

    def __init__(self, max_steps: int = 6, max_repeated_hashes: int = 2):
        self.max_steps = max_steps
        self.max_repeated_hashes = max_repeated_hashes
        self.records: List[AgentActionRecord] = []
        self.hash_counts: dict[str, int] = {}

    def _hash_content(self, content: str) -> str:
        return hashlib.sha256(content.strip().encode("utf-8")).hexdigest()[:12]

    def record_and_evaluate(self, agent_name: str, output: str) -> str:
        step_num = len(self.records) + 1
        content_hash = self._hash_content(output)

        # Update tracking
        self.records.append(AgentActionRecord(step=step_num, agent_name=agent_name, payload_hash=content_hash))
        self.hash_counts[content_hash] = self.hash_counts.get(content_hash, 0) + 1

        # Check 1: Max step limit
        if step_num > self.max_steps:
            return "CIRCUIT_TRIPPED: MAX_RECURSION_LIMIT_EXCEEDED"

        # Check 2: Repetition / Cycle detection
        if self.hash_counts[content_hash] >= self.max_repeated_hashes:
            return f"CIRCUIT_TRIPPED: REPETITIVE_CYCLE_DETECTED (Hash {content_hash} occurred {self.hash_counts[content_hash]} times)"

        return "OK"


if __name__ == "__main__":
    breaker = CircuitBreaker(max_steps=5, max_repeated_hashes=2)

    # Simulating a ping-pong loop between Coder and Reviewer
    conversation_stream = [
        ("CoderAgent", "def run(): return 42"),
        ("ReviewerAgent", "Please add type annotations."),
        ("CoderAgent", "def run(): return 42"),  # Repeated payload
        ("ReviewerAgent", "Please add type annotations."),  # Repeated payload
    ]

    for agent, text in conversation_stream:
        status = breaker.record_and_evaluate(agent, text)
        print(f"[Step {len(breaker.records)}] {agent} outputted: '{text}' -> Breaker Status: {status}")
        if status.startswith("CIRCUIT_TRIPPED"):
            print(f"\n[EMERGENCY ESCALATION] Terminating workflow and notifying human supervisor.")
            break
```

**Developer Explanation:**
- **Libraries Used:** `hashlib` for payload hashing, `pydantic` for tracking records.
- **How It Works:** Computes a cryptographic hash of every agent response. If duplicate responses occur above a threshold or if the step count exceeds bounds, the breaker trips and aborts execution.
- **Expected Output:** Immediate termination upon detecting cyclic message exchanges.
- **Why This Approach:** Prevents runaway cloud bills and deadlock states in production agent swarms.

---

## Conclusion: The Power of Teams

Multi-agent systems represent the move from "Chatting with an AI" to "Managing an AI Workforce." By specializing your agents, isolating their contexts, and coordinating them with a robust shared state, you can solve problems that are orders of magnitude more complex than what a single prompt could ever handle.

In the next chapter, we will look at **Long-Horizon Learning Systems**, where these agents learn and improve over days and weeks, rather than just seconds.

---

## References & Further Reading
- **Klement Gunndu (2026)**: *Build Your First Multi-Agent System in Python: 3 Patterns That Scale*.
- **LangGraph Documentation**: *Multi-Agent Workflows and Coordination Graphs*.
- **Wu et al. (2023)**: *AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation*. Microsoft Research. arXiv:2308.08155.
- **Li et al. (2023)**: *Camel: Communicative Agents for "Mind" Exploration of Large Language Model Society*. NeurIPS 2023.
- **CrewAI**: *Orchestrating Role-Based Autonomous AI Agents in Production*.
- **DeepLearning.AI**: *Multi-Agent Systems with LangGraph & CrewAI Specialization*.
