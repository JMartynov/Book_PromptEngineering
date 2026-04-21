# Chapter 4: Context Engineering (NEW CORE DISCIPLINE)

## Introduction: Context is the New Code

In the early days of LLM interaction, we focused almost entirely on the "Prompt"—the 50-word instruction. But as models have grown more powerful, with context windows expanding from 4K tokens to 1M+ tokens, the focus has shifted. In 2026, the real engineering challenge isn't just "writing the prompt," it's **Context Engineering**.

Context Engineering is the systematic design and management of the information provided to the LLM. It's the "Data Layer" of your AI application. If the Prompt is the "Function," the Context is the "Input Data." And in practice, **bad context is the #1 cause of AI failure**.

---

## Deep Technical Analysis: The Context Window Budget

The core insight of 2026 is that **LLMs have a finite "Attention Budget."** Every token in the context window competes for the model's attention. As the context grows, the model's precision drops, its reasoning weakens, and it starts "losing the needle in the haystack."

### 1. The "Lost in the Middle" Phenomenon
Research (Liu et al., 2024) has proven that LLMs are significantly better at using information at the very beginning or the very end of a prompt. Information placed in the middle is often ignored. Context Engineering solves this by **Ranking** and **Ordering** context so the most "High-Signal" tokens are placed in the model's "high-attention" zones.

### 2. Progressive Disclosure (Context Fetching)
Instead of dumping all 10,000 documents into the prompt at once, modern systems use **Progressive Disclosure**. The agent is given a small "Summary" of the available knowledge and must "Fetch" the full details only when it decides they are necessary. This keeps the prompt clean and the model's attention focused.

### 3. Semantic, Episodic, and Working Memory
We now architect AI memory using a 3-tier model:
-   **Semantic Memory:** The "Knowledge Base" (RAG). Static facts about the world or your company.
-   **Episodic Memory:** The "History" of previous interactions. What did the user ask 5 minutes ago?
-   **Working Memory:** The "Current Task." The immediate reasoning steps and tool outputs.

---

## Why Context Engineering Solves Real-World Problems

In practice, Context Engineering solves several critical production issues:
-   **Token Overload & Cost:** Sending 100K tokens for every query is expensive and slow. Context Filtering ensures you only send the 500 tokens that actually matter.
-   **Information Contradiction:** If your RAG system retrieves two documents that say different things, the AI will get confused. Context Resolution logic (e.g., "prefer the most recent date") is essential.
-   **Privacy & Permissions:** Context Filtering ensures that the AI only "sees" the data that the current user has permission to access, preventing data leaks.

---

## Practical Implementation: 8 Python Examples

These examples demonstrate how to build robust context management systems using modern Python patterns.

### Example 1: Reranking for "High-Signal" Retrieval
**Problem:** A vector search (RAG) might return 10 "similar" documents, but only 2 are actually useful. The other 8 are "noise" that distracts the model.
**Solution:** Use a "Reranker" (a smaller, faster model) to score the 10 results and only inject the top 2 into the final prompt.

```python
import numpy as np
from typing import List, Dict, Any

# Mock reranking model for demonstration
def call_reranker_api(query: str, documents: List[str]) -> List[float]:
    """
    Simulates a Cross-Encoder (e.g., Cohere Rerank or BGE-Reranker)
    that scores (Query, Document) pairs for exact relevance.
    """
    # In reality, this returns a relevance score from 0.0 to 1.0
    return [np.random.uniform(0.1, 0.9) for _ in documents]

def get_optimized_context(query: str, raw_retrieval_results: List[Dict[str, Any]], top_k: int = 2) -> str:
    """
    Stages context retrieval to maximize the Signal-to-Noise Ratio (SNR).

    Logic:
    1. Takes the top 10-20 results from a Vector DB.
    2. Uses a Reranker to find the 2 docs that actually answer the query.
    3. Prunes the rest to save model attention and tokens.
    """
    texts = [res['text'] for res in raw_retrieval_results]

    # Stage 2: Reranking (High precision, high cost)
    scores = call_reranker_api(query, texts)

    # Sort by the new relevance score
    scored_docs = sorted(zip(scores, raw_retrieval_results), key=lambda x: x[0], reverse=True)

    # Keep only the high-signal tokens
    signal_docs = [doc for score, doc in scored_docs[:top_k]]

    return "\n\n".join([f"[DOC {i}] {d['text']}" for i, d in enumerate(signal_docs)])

# Execution Example
if __name__ == "__main__":
    q = "How do I reset my password?"
    results = [
        {"text": "To reset password, click settings.", "id": 1},
        {"text": "Security is important.", "id": 2},
        {"text": "Office hours are 9-5.", "id": 3}
    ]
    # context = get_optimized_context(q, results)
    # print(f"Optimized Context:\n{context}")
```
**Why this is preferred:** It prevents "Information Dilution." By reducing the noise, you significantly increase the probability that the model will find the "needle" it needs to answer the question.

---

### Example 2: Context Compression (History Summarization)
**Problem:** A long chat history (Episodic Memory) can consume 80% of your token budget.
**Solution:** Use an LLM to "Compress" the old parts of the history into a concise summary, while keeping the last 2 messages in full.

```python
from typing import List, Dict

def call_llm_summarizer(history: str) -> str:
    """Simulates an LLM call to compress history."""
    return "User is debugging a Python script and wants to use Pydantic."

def manage_conversation_memory(chat_history: List[Dict[str, str]], limit: int = 5) -> str:
    """
    Maintains a 3-tier memory model:
    1. Working Memory: The last 2 messages (Full text).
    2. Episodic Memory: Older messages compressed into a summary.
    3. Semantic Memory: (External knowledge base, not handled here).
    """
    if len(chat_history) <= limit:
        return str(chat_history)

    # Tier 1: Preserve Working Memory (Latest 2 turns)
    working_memory = chat_history[-2:]

    # Tier 2: Compress everything else
    old_turns = chat_history[:-2]
    summary = call_llm_summarizer(str(old_turns))

    return f"SUMMARY OF PAST TURNS: {summary}\nLATEST TURNS: {working_memory}"

# Execution Example
if __name__ == "__main__":
    history = [{"role": "u", "content": "Hi"}, {"role": "a", "content": "Hello"}] * 5
    # context = manage_conversation_memory(history)
```
**Why this is preferred:** It allows for "Infinite Context" conversations without the linear cost and latency increase of a growing prompt.

---

### Example 3: The "Context-First" Order Pattern
**Problem:** Models suffer from "Recency Bias," often following the most recent instruction and ignoring the earlier context.
**Solution:** Place the **Context** at the very beginning and the **Query** at the very end.

```python
def build_grounded_prompt(context_data: str, user_query: str) -> str:
    """
    Applies the Context-First pattern to maximize instruction following.

    Structure:
    1. Context (The Ground Truth)
    2. Instructions (The Reasoning Rules)
    3. Query (The Action Trigger)
    """
    return f"""
<context_block>
{context_data}
</context_block>

### INSTRUCTIONS:
Answer the query based ONLY on the <context_block> above.
If the information is not present, do not hallucinate; say 'NOT_FOUND'.

### USER QUERY:
{user_query}
""".strip()

# Execution Example
if __name__ == "__main__":
    data = "Our office is located at 123 Main St."
    query = "Where is the office?"
    # prompt = build_grounded_prompt(data, query)
```
**Why this is preferred:** Research shows that putting the "Call to Action" (the query) at the end of the prompt improves "Instruction Following" scores by up to 15%.

---

### Example 4: Context Isolation with "Schema Headers"
**Problem:** When providing multiple data types (JSON, Markdown, Code), the model can get confused about where one ends and the next begins.
**Solution:** Use distinct "Type Headers" and delimiters for each part of the context.

```python
def format_multimodal_context(logs: str, docs: str) -> str:
    """Formats context with clear schema headers for multi-source disambiguation."""

    return f"### CONTEXT DATA\n[SOURCE: SYSTEM_ERROR_LOGS]\n{logs}\n\n[SOURCE: DOCS]\n{docs}"

# Execution Example
if __name__ == "__main__":
    l = '{"error": "timeout", "service": "payment-api"}'
    d = "# Payment API\nTimeouts usually occur if the DB latency exceeds 500ms."
    # print(format_multimodal_context(l, d))
```json
{ "error": "auth_failure", "user": "jd_99" }
```

[SOURCE: DOCUMENTATION | TYPE: MARKDOWN]
# Auth Failure
This occurs when the JWT token has expired.

Based on the LOGS and DOCUMENTATION above, what is the fix?
"""
```
**Why this is preferred:** It provides **Visual Hierarchy** for the model's attention mechanism, making it easier to distinguish between "Ground Truth" and "Reference Material."

---

### Example 5: Metadata Filtering for Privacy (Security)
**Problem:** An AI assistant might "hallucinate" an answer using data from a different user's context.
**Solution:** Filter the context at the **Database Level** using metadata before it ever reaches the LLM.

```python
from typing import List, Dict, Any

class SecureRetriever:
    """Ensures data privacy by filtering context at the retrieval layer."""

    def fetch_authorized_context(self, query: str, user_role: str, user_id: str) -> List[str]:
        # Simulation of a Vector DB search with metadata filtering
        # In practice: db.search(query, filter={"allowed_roles": user_role})
        raw_data = [
            {"text": "General Policy", "role": "employee"},
            {"text": "Manager Salaries", "role": "admin"}
        ]

        # Hard Filter in Python (The 'Security Gate')
        authorized_context = [
            d['text'] for d in raw_data
            if d['role'] == user_role or d.get('owner_id') == user_id
        ]

        return authorized_context

# Execution Example
if __name__ == "__main__":
    retriever = SecureRetriever()
    # A 'junior' user will never see 'admin' context in the prompt
    context = retriever.fetch_authorized_context("What are the salaries?", "junior", "user_123")
    # print(context) # ['General Policy']
```
**Why this is preferred:** It is the only way to ensure **Data Privacy**. You should never rely on the LLM's "Instructions" to keep data secret; you must engineer the context so it never sees the secret data in the first place.

---

### Example 6: Dynamic "Skill" Loading (Progressive Disclosure)
**Problem:** A "Mega-Prompt" with 50 different "Skills" (how to refund, how to upgrade, how to cancel) is too noisy.
**Solution:** Use a "Router" to identify the required skill and only load the relevant context for that skill.

```python
def classify_intent(query: str) -> str:
    """Mock router: Classifies query intent."""
    if "refund" in query.lower(): return "refund_policy"
    return "general_faq"

def build_dynamic_skill_prompt(query: str) -> str:
    """Demonstrates progressive disclosure by loading skill-specific context."""

    intent = classify_intent(query)

    # Load only the relevant "Skill" context from a dictionary or database
    skills_db = {
        "refund_policy": "Refunds are processed in 5 business days.",
        "general_faq": "Our office is open from 9 AM to 5 PM EST."
    }

    relevant_context = skills_db.get(intent, "General company information...")

    return f"RELEVANT_POLICY: {relevant_context}\n\nUSER_QUESTION: {query}"

# Execution Example
if __name__ == "__main__":
    # Prompt will only contain the refund policy, keeping it small and focused.
    prompt = build_dynamic_skill_prompt("How long do refunds take?")
```
**Why this is preferred:** It keeps the "Attention Budget" focused. The model is 100% focused on the refund policy rather than being distracted by the cancellation or upgrade rules.

---

### Example 7: Context Resolution (Handling Conflicts)
**Problem:** Two documents in the context provide conflicting information (e.g., an old price vs. a new price).
**Solution:** Inject "Recency Metadata" and instruct the model to prioritize the most recent information.

```python
from typing import List, Dict

def format_context_with_recency(docs: List[Dict[str, str]]) -> str:
    """Resolves conflicts by injecting recency metadata into the context."""

    formatted_docs = []
    for doc in docs:
        formatted_docs.append(f"[LAST UPDATED: {doc['date']}] Content: {doc['text']}")

    context_str = "\n".join(formatted_docs)

    return f"""
### DATA CONTEXT
{context_str}

### RESOLUTION RULE
If information in the context conflicts (e.g. different prices or dates),
always treat the document with the LATEST (most recent) 'LAST UPDATED' date as the truth.
"""

# Execution Example
if __name__ == "__main__":
    data = [
        {"date": "2023-01-01", "text": "Price is $50"},
        {"date": "2024-05-01", "text": "Price is $60"}
    ]
    # prompt = format_context_with_recency(data)
```
**Why this is preferred:** It provides a **Deterministic Resolution Rule** for the model's probabilistic reasoning, ensuring consistency in a world of changing data.

---

### Example 8: Self-Correction (The "Context Check")
**Problem:** The RAG system returns context that is totally irrelevant, but the model tries to "Force" an answer anyway.
**Solution:** Ask the model to first evaluate if the context is sufficient before answering.

```python
def build_self_checking_prompt(context: str, query: str) -> str:
    """Builds a prompt that empowers the model to reject insufficient context."""

    return f"""
### INSTRUCTIONS
1. Read the provided CONTEXT carefully.
2. Determine if the CONTEXT contains the specific information needed to answer the QUERY.
3. If the answer is NOT present, output ONLY the string: [INSUFFICIENT_CONTEXT].
4. If the answer IS present, provide a direct and concise response.

### CONTEXT
{context}

### QUERY
{query}
"""

# Execution Example
if __name__ == "__main__":
    c = "Our office is in New York."
    q = "What is the capital of France?"
    # response = call_llm(build_self_checking_prompt(c, q))
    # if "[INSUFFICIENT_CONTEXT]" in response:
    #     print("AI recognized it didn't have the data. Safe!")
```
**Why this is preferred:** It reduces **Hallucination by Force**. By giving the model an "Explicit Exit," you prevent it from making things up when the context layer fails.

---

## Conclusion: Engineering the Surface

Context Engineering is the transition from "writing instructions" to "managing a dynamic environment." By mastering retrieval, filtering, and compression, you ensure that your AI has the right information at the right time, formatted in the right way.

In the next part, we will move beyond single prompts and explore how to build **Prompt Pipelines**—the "System" that orchestrates these contexts and prompts into a cohesive application.

---

## References & Further Reading
- **Liu et al. (2024)**: *Lost in the Middle: How Language Models Use Long Contexts*.
- **Kushal Banda (2026)**: *State of Context Engineering in 2026*.
- **Anthropic Documentation**: *Context Engineering for Agents*.
- **Meta-Intelligence (2026)**: *Context Engineering Guide: Memory Systems for Production AI*.
