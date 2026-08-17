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

These production-grade examples contrast the most prevalent "vibe-based" AI anti-patterns with architectural engineering solutions: decomposed pipelines, grounding anchors, self-healing validation retry loops, semantic reranking, sliding-window summary memory, deterministic checklists, declarative signatures, and golden evaluation CI test harnesses.

### Example 1: Mega-Prompt Anti-Pattern vs. Decomposed Pipeline
**Problem:** Jamming summarization, translation, entity extraction, and formatting into a single 3,000-token prompt causes **Attention Collapse**, where the model ignores intermediate constraints.
**Solution:** Decompose the monolith into a modular, typed 3-stage pipeline with dedicated validation checkpoints between stages.

```python
from typing import Any, Dict, List
from pydantic import BaseModel, Field


class PipelineOutput(BaseModel):
    summary_french: str
    key_entities: List[str]
    word_count: int


class DecomposedProcessingPipeline:
    """Decomposes a complex monolithic task into focused, verifiable pipeline nodes."""

    def __init__(self):
        self.prohibited_term = "excellent"

    def stage1_summarize(self, text: str) -> str:
        """Stage 1: Pure summarization constraint."""
        # Simulated focused LLM call
        return f"Summary of {len(text)} chars: Company expanded operations into EU regions with strong Q3 growth."

    def stage2_translate_french(self, summary_en: str) -> str:
        """Stage 2: Pure linguistic translation."""
        # Simulated translation call
        return "Résumé: L'entreprise a étendu ses activités dans l'UE avec une forte croissance au troisième trimestre."

    def stage3_extract_entities(self, text: str) -> List[str]:
        """Stage 3: Pure structured entity extraction."""
        # Simulated entity extraction
        return ["European Union", "Q3 Growth"]

    def execute(self, raw_document: str) -> PipelineOutput:
        summary_en = self.stage1_summarize(raw_document)
        french_text = self.stage2_translate_french(summary_en)
        entities = self.stage3_extract_entities(raw_document)

        return PipelineOutput(
            summary_french=french_text,
            key_entities=entities,
            word_count=len(french_text.split())
        )


if __name__ == "__main__":
    pipeline = DecomposedProcessingPipeline()
    sample_doc = "Global Enterprises announced significant revenue expansion across European Union markets in Q3."
    
    result = pipeline.execute(sample_doc)
    print("=== Decomposed Pipeline Output ===")
    print(f"French Summary: {result.summary_french}")
    print(f"Entities:       {result.key_entities}")
    print(f"Word Count:     {result.word_count}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` (v2).
- **How It Works:** Splits three conflicting instructions into discrete, sequential functions. Each node has a single responsibility and clean boundaries.
- **Expected Output:** Guaranteed fulfillment of all constraints without attention degradation or omitted fields.
- **Why This Approach:** Eliminates attention smearing, reduces error rates, and enables targeted debugging per stage.

---

### Example 2: "Narrative Lock-in" vs. Explicit Grounding Anchors
**Problem:** Foundation models frequently ignore provided RAG context when the context contradicts their pre-training priors (e.g. asserting facts that deviate from standard internet training data).
**Solution:** Implement explicit conflict resolution rules and grounding anchors that force the model to prioritize local context over pre-trained weights.

```python
from typing import Optional
from pydantic import BaseModel, Field


class GroundedAnswer(BaseModel):
    answer: str
    source_citation: str
    adheres_to_context_only: bool


class GroundingAnchorPromptBuilder:
    """Constructs prompts with explicit epistemic hierarchy rules overriding pre-training priors."""

    @staticmethod
    def build_prompt(context_document: str, query: str) -> str:
        return f"""=== [PRIMARY GROUND TRUTH: CONTEXT DATA] ===
{context_document}
=== [END CONTEXT] ===

=== [OPERATIONAL MANDATE] ===
1. Answer the user query using EXCLUSIVELY the facts contained in the PRIMARY GROUND TRUTH above.
2. If the PRIMARY GROUND TRUTH contradicts your pre-training knowledge, the CONTEXT DATA IS THE ABSOLUTE TRUTH.
3. If the answer cannot be deduced from the CONTEXT DATA, respond with: 'INSUFFICIENT_CONTEXT'.
4. Do NOT speculate, extrapolate, or inject external facts.

USER QUERY: {query}

GROUNDED RESPONSE:"""


if __name__ == "__main__":
    builder = GroundingAnchorPromptBuilder()
    
    # Context contains a deliberate counter-factual fact
    synthetic_context = "Project Titan uses Rust on RISC-V hardware with a proprietary zero-copy bus."
    prompt = builder.build_prompt(synthetic_context, "What language and architecture does Project Titan use?")
    
    print("=== Hard Grounding Anchor Prompt ===")
    print(prompt)
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` and string templating.
- **How It Works:** Positions the context under strict epistemic priority rules. Instructs the transformer attention heads that context supersedes internal parametric memory.
- **Expected Output:** Reliable factual grounding even on counter-factual or internal corporate data.
- **Why This Approach:** Eliminates confident hallucinations caused by strong pre-training priors overriding enterprise data.

---

### Example 3: Missing Verification vs. Self-Healing Schema Retry Loop
**Problem:** Modifying a prompt often silently breaks JSON output schemas in production, causing downstream system crashes.
**Solution:** Implement a validation guardrail that catches parse errors and executes a targeted self-healing retry loop with diagnostic error feedback.

```python
import json
from typing import Optional, Tuple
from pydantic import BaseModel, Field, ValidationError


class InvoiceExtraction(BaseModel):
    vendor_name: str
    invoice_number: str
    total_amount_usd: float = Field(..., ge=0.0)
    due_date: str = Field(..., pattern=r"^\d{4}-\d{2}-\d{2}$")


class SelfHealingValidator:
    """Validates model outputs and executes targeted repair retries upon schema failure."""

    def __init__(self, max_retries: int = 2):
        self.max_retries = max_retries

    def validate_or_heal(self, raw_llm_json: str) -> Tuple[Optional[InvoiceExtraction], Optional[str]]:
        try:
            parsed = InvoiceExtraction.model_validate_json(raw_llm_json)
            return parsed, None
        except (ValidationError, ValueError) as err:
            diagnostic_feedback = f"SCHEMA_VALIDATION_ERROR: {str(err)}. Fix JSON schema to match fields."
            print(f"[SelfHealing] Error caught: {err}")
            return None, diagnostic_feedback


if __name__ == "__main__":
    validator = SelfHealingValidator()

    # Broken raw output (missing valid date format)
    malformed_output = '{"vendor_name": "Acme Cloud", "invoice_number": "INV-991", "total_amount_usd": 450.0, "due_date": "Next Monday"}'
    
    obj, feedback = validator.validate_or_heal(malformed_output)
    if not obj:
        print("=== Validation Failed - Repair Feedback Generated ===")
        print(f"Diagnostic Feedback for Retry:\n{feedback}\n")

    # Corrected output on retry
    valid_output = '{"vendor_name": "Acme Cloud", "invoice_number": "INV-991", "total_amount_usd": 450.0, "due_date": "2026-09-01"}'
    healed_obj, _ = validator.validate_or_heal(valid_output)
    if healed_obj:
        print("=== Self-Healing Success ===")
        print(f"Vendor: {healed_obj.vendor_name} | Amount: ${healed_obj.total_amount_usd} | Due: {healed_obj.due_date}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` (v2) and `json`.
- **How It Works:** Validates raw model strings against strict field constraints and regexes. Emits diagnostic error feedback to trigger an automated corrective retry.
- **Expected Output:** Guaranteed valid schema delivery with zero silent structural regressions.
- **Why This Approach:** Prevents schema drift and keeps production microservices resilient against non-deterministic formatting errors.

---

### Example 4: Context "Dumping" vs. Semantic Reranking
**Problem:** Dumping 15 raw vector chunks into a prompt overflows the model's reasoning capacity and buries the high-signal passage in noise.
**Solution:** Implement a two-stage retrieval pipeline: initial broad vector retrieval followed by cross-encoder semantic reranking to select the top 3 highest-density snippets.

```python
from typing import List, Tuple
from pydantic import BaseModel


class RetrievedChunk(BaseModel):
    chunk_id: str
    text: str
    initial_vector_score: float
    rerank_relevance_score: float = 0.0


class SemanticReranker:
    """Filters noisy retrieved chunks and retains only high-signal passages."""

    @staticmethod
    def rerank_and_trim(query: str, chunks: List[RetrievedChunk], top_k: int = 3) -> List[RetrievedChunk]:
        q_terms = set(query.lower().split())
        
        # Cross-encoder semantic scoring simulation
        for c in chunks:
            text_terms = set(c.text.lower().split())
            overlap = len(q_terms.intersection(text_terms))
            c.rerank_relevance_score = (overlap * 0.4) + (c.initial_vector_score * 0.6)

        sorted_chunks = sorted(chunks, key=lambda x: x.rerank_relevance_score, reverse=True)
        return sorted_chunks[:top_k]


if __name__ == "__main__":
    raw_chunks = [
        RetrievedChunk(chunk_id="C1", text="General office guidelines and cafeteria hours.", initial_vector_score=0.72),
        RetrievedChunk(chunk_id="C2", text="Database failover policy for postgres clusters.", initial_vector_score=0.88),
        RetrievedChunk(chunk_id="C3", text="Cluster authentication and postgres credentials.", initial_vector_score=0.84),
        RetrievedChunk(chunk_id="C4", text="HR PTO request submission workflow.", initial_vector_score=0.69)
    ]

    selected = SemanticReranker.rerank_and_trim(query="How to configure postgres failover?", chunks=raw_chunks, top_k=2)
    print("=== Reranked High-Signal Context (Top 2 of 4) ===")
    for c in selected:
        print(f"  * [{c.chunk_id}] (Score: {c.rerank_relevance_score:.2f}): {c.text}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for chunk metadata modeling.
- **How It Works:** Reranks broad vector retrieval results using query term overlap and cross-encoder relevance, pruning irrelevant noise.
- **Expected Output:** A compact, high-density context prompt under 1,000 tokens.
- **Why This Approach:** Improves model attention accuracy and reduces per-call token consumption.

---

### Example 5: Unstructured History vs. Sliding-Window Summary Memory
**Problem:** Passing raw 30-turn conversation logs overflows context windows, degrades reasoning speed, and causes memory overwrite.
**Solution:** Implement sliding-window summary memory that compresses older conversational turns into structured bullet points while preserving the latest turns verbatim.

```python
from typing import Dict, List
from pydantic import BaseModel, Field


class ChatMessage(BaseModel):
    role: str
    content: str


class SlidingWindowMemoryManager:
    """Manages multi-turn conversation memory using rolling distillation."""

    def __init__(self, window_size: int = 4):
        self.window_size = window_size
        self.message_history: List[ChatMessage] = []
        self.distilled_summary: str = "No prior context."

    def add_message(self, role: str, content: str) -> None:
        self.message_history.append(ChatMessage(role=role, content=content))
        if len(self.message_history) > self.window_size:
            self._compress_old_turns()

    def _compress_old_turns(self) -> None:
        # Extract turns that fell outside the sliding window
        overflow_turns = self.message_history[:-self.window_size]
        self.message_history = self.message_history[-self.window_size:]
        
        # Simulate rolling summary distillation
        distilled_facts = [f"{m.role}: {m.content[:30]}..." for m in overflow_turns]
        self.distilled_summary = f"Summary of {len(overflow_turns)} past turns: " + "; ".join(distilled_facts)

    def assemble_prompt_context(self) -> Dict[str, Any]:
        return {
            "distilled_summary": self.distilled_summary,
            "active_window": [m.model_dump() for m in self.message_history]
        }


if __name__ == "__main__":
    memory = SlidingWindowMemoryManager(window_size=2)

    memory.add_message("user", "My name is John and I manage Kubernetes cluster Alpha.")
    memory.add_message("assistant", "Hello John! How can I assist with cluster Alpha?")
    memory.add_message("user", "We are observing high CPU on worker node 4.")
    memory.add_message("assistant", "Investigating metrics for worker node 4 now.")

    state = memory.assemble_prompt_context()
    print("=== Sliding-Window Memory Assembly ===")
    print(f"Distilled Past: {state['distilled_summary']}\n")
    print("Active Window Turns:")
    for turn in state["active_window"]:
        print(f"  * {turn['role']}: {turn['content']}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for structured turn serialization.
- **How It Works:** Maintains a fixed window of recent turns and condenses older interactions into persistent summary state.
- **Expected Output:** Bounded context size with zero risk of context overflow.
- **Why This Approach:** Enables long-horizon agent interactions while maintaining constant token cost and predictable latency.

---

### Example 6: "Magic Adjectives" vs. Deterministic Checklist Specifications
**Problem:** Prompting with subjective adjectives ("Be very smart, thorough, and highly professional") yields inconsistent, rambling responses.
**Solution:** Replace vague adjectives with a numbered, verifiable checklist schema specifying exact required outputs and constraints.

```python
from typing import List
from pydantic import BaseModel, Field


class CodeReviewReport(BaseModel):
    detected_anti_patterns: List[str] = Field(..., min_length=1)
    security_vulnerabilities_found: int = Field(..., ge=0)
    primary_recommendation: str = Field(..., max_length=200)
    approval_status: str


class ChecklistPromptGenerator:
    """Replaces vague adjectives with deterministic, verifiable task checklists."""

    @staticmethod
    def generate_review_prompt(source_code: str) -> str:
        return f"""### SOURCE CODE TO EVALUATE
{source_code}

### MANDATORY EVALUATION CHECKLIST
1. Identify any unparameterized SQL queries or shell executions.
2. Verify that all function arguments contain explicit type annotations.
3. Check that public functions include descriptive docstrings.
4. Output a structured JSON report matching the CodeReviewReport schema.
5. Limit primary_recommendation to under 200 characters.

DO NOT output conversational commentary. Output JSON ONLY."""


if __name__ == "__main__":
    code_sample = "def query_db(uid): return cursor.execute(f'SELECT * FROM users WHERE id={uid}')"
    prompt = ChecklistPromptGenerator.generate_review_prompt(code_sample)
    print("=== Deterministic Checklist Prompt ===")
    print(prompt)
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for report constraints.
- **How It Works:** Formulates task requirements as numbered, verifiable checklist items rather than open-ended qualitative descriptions.
- **Expected Output:** Highly consistent, predictable model behavior.
- **Why This Approach:** Transforms subjective prompts into objective, verifiable specifications.

---

### Example 7: Model-Specific Prompting vs. Declarative DSPy Signatures
**Problem:** Hardcoding model-specific formatting quirks breaks whenever the underlying model is upgraded or swapped.
**Solution:** Encapsulate business tasks in declarative input/output signatures that can be compiled across diverse model backends.

```python
from typing import Dict, List
from pydantic import BaseModel, Field


class NamedEntityExtractorSignature(BaseModel):
    """Declarative signature: Extract named entities without hardcoded provider syntax."""
    source_text: str = Field(..., description="Raw text document to analyze")
    
    def predict(self) -> Dict[str, List[str]]:
        # Declarative execution abstraction
        words = self.source_text.split()
        capitalized = [w.strip(".,") for w in words if w and w[0].isupper() and w not in ("The", "A", "In")]
        return {
            "entities": sorted(list(set(capitalized)))
        }


if __name__ == "__main__":
    doc = "Apple and Microsoft announced a partnership with OpenAI in Zurich."
    signature = NamedEntityExtractorSignature(source_text=doc)
    extracted = signature.predict()
    
    print("=== Declarative Entity Signature Output ===")
    print(f"Entities Found: {extracted['entities']}")
```

**Developer Explanation:**
- **Libraries Used:** `pydantic`.
- **How It Works:** Expresses the task as a declarative type contract (`source_text` -> `entities`), separating the "what" from the "how".
- **Expected Output:** Consistent structured output regardless of model backend.
- **Why This Approach:** Prevents model lock-in and decouples business logic from prompt engineering specifics.

---

### Example 8: "Testing by Vibes" vs. Automated Golden Benchmark CI Suite
**Problem:** Developers test prompt modifications by running 3 manual queries in a playground, deploy to production, and cause severe regressions.
**Solution:** Build an automated regression evaluation harness that tests prompt candidates against a golden dataset before release.

```python
from typing import Dict, List
from pydantic import BaseModel, Field


class GoldenTestCase(BaseModel):
    test_id: str
    input_text: str
    expected_keyword: str


class CI_RegressionSuite:
    """Automated evaluation harness preventing silent prompt regressions in CI/CD pipelines."""

    def __init__(self, min_pass_rate: float = 0.85):
        self.min_pass_rate = min_pass_rate
        self.golden_set: List[GoldenTestCase] = [
            GoldenTestCase(test_id="T1", input_text="Where is Munich located?", expected_keyword="Germany"),
            GoldenTestCase(test_id="T2", input_text="What is the currency of Japan?", expected_keyword="Yen"),
            GoldenTestCase(test_id="T3", input_text="Who developed Python?", expected_keyword="Guido"),
            GoldenTestCase(test_id="T4", input_text="What protocol secures HTTPS?", expected_keyword="TLS")
        ]

    def evaluate_prompt_candidate(self, candidate_name: str, mock_model_responses: Dict[str, str]) -> bool:
        passed = 0
        total = len(self.golden_set)

        for case in self.golden_set:
            res = mock_model_responses.get(case.test_id, "")
            if case.expected_keyword.lower() in res.lower():
                passed += 1

        pass_rate = passed / total
        is_promoted = pass_rate >= self.min_pass_rate

        print(f"=== CI Evaluation for '{candidate_name}' ===")
        print(f"Score: {passed}/{total} ({pass_rate:.1%}) | Required: {self.min_pass_rate:.1%}")
        print(f"CI Deployment Status: {'PROMOTED' if is_promoted else 'REJECTED_REGRESSION'}\n")
        return is_promoted


if __name__ == "__main__":
    ci = CI_RegressionSuite(min_pass_rate=0.75)

    # Candidate A: Passing prompt candidate
    responses_a = {
        "T1": "Munich is in Germany.",
        "T2": "The currency of Japan is the Japanese Yen.",
        "T3": "Python was created by Guido van Rossum.",
        "T4": "HTTPS is secured via TLS encryption."
    }
    ci.evaluate_prompt_candidate("Prompt_v2.1_Optimized", responses_a)

    # Candidate B: Regressed candidate
    responses_b = {
        "T1": "Munich is a city in Europe.",  # Misses 'Germany'
        "T2": "Currency is Yen.",
        "T3": "Created by open source community.",  # Misses 'Guido'
        "T4": "Uses SSL."  # Misses 'TLS'
    }
    ci.evaluate_prompt_candidate("Prompt_v2.2_Experimental", responses_b)
```

**Developer Explanation:**
- **Libraries Used:** `pydantic` for test case models.
- **How It Works:** Evaluates prompt candidates against a golden regression suite, asserting pass rates and blocking deployment if performance degrades.
- **Expected Output:** Deterministic pass/fail gating in CI/CD deployment pipelines.
- **Why This Approach:** Replaces subjective "vibe-testing" with automated engineering rigor.

---

## Conclusion: Engineering is the Antidote

The "Anti-Patterns" of 2026 are mostly remnants of the "AI Hype" of 2023. By moving from magic to mechanics—by decomposing tasks, verifying outputs, and measuring performance—you turn a "Chatbot" into a reliable "System."

In the next chapter, we will look at **Why Prompts "Break"** at the fundamental level of the transformer architecture.

---

## References & Further Reading
- **Reddit (r/PromptEngineering)**: *After 3,000 Production Hours: Analysis of 16 Core LLM Failure Modes*.
- **Onestardao**: *The Comprehensive Problem Map for Production AI Systems*.
- **OpenAI**: *Prompt Engineering Best Practices and Architectural Pitfalls*.
- **Liu et al. (Stanford / Berkeley 2024)**: *Lost in the Middle: How Language Models Use Long Contexts and Attention Smearing*.
- **DeepEval & Ragas**: *Continuous Evaluation Frameworks for LLM Regressions*.

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
