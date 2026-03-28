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
# In 2026, we use tools like Cohere Rerank or BGE-Reranker
def rerank_documents(query: str, docs: list) -> list:
    # (Mocking a reranking model call)
    # The model compares (query, doc) and returns a relevance score.
    scored_docs = sorted(docs, key=lambda d: d['relevance_score'], reverse=True)
    return scored_docs[:2] # Keep only the 'High-Signal' tokens

# query = "How do I reset my password?"
# raw_docs = get_vector_search_results(query)
# signal_docs = rerank_documents(query, raw_docs)
```
**Why this is preferred:** It prevents "Information Dilution." By reducing the noise, you significantly increase the probability that the model will find the "needle" it needs to answer the question.

---

### Example 2: Context Compression (History Summarization)
**Problem:** A long chat history (Episodic Memory) can consume 80% of your token budget.
**Solution:** Use an LLM to "Compress" the old parts of the history into a concise summary, while keeping the last 2 messages in full.

```python
def compress_memory(full_history: list) -> str:
    # Step 1: Keep the last 2 messages as 'Working Memory'
    working_memory = full_history[-2:]
    # Step 2: Summarize everything else into 'Summary Context'
    old_history = full_history[:-2]
    summary = call_llm(f"Summarize this conversation history: {old_history}")

    return f"SUMMARY OF PAST: {summary}\nLATEST MESSAGES: {working_memory}"
```
**Why this is preferred:** It allows for "Infinite Context" conversations without the linear cost and latency increase of a growing prompt.

---

### Example 3: The "Context-First" Order Pattern
**Problem:** Models suffer from "Recency Bias," often following the most recent instruction and ignoring the earlier context.
**Solution:** Place the **Context** at the very beginning and the **Query** at the very end.

```python
def build_optimized_order_prompt(context: str, query: str):
    return f"""
<context>
{context}
</context>

USER QUERY: {query}
(Remember: Answer using ONLY the context provided above.)
"""
```
**Why this is preferred:** Research shows that putting the "Call to Action" (the query) at the end of the prompt improves "Instruction Following" scores by up to 15%.

---

### Example 4: Context Isolation with "Schema Headers"
**Problem:** When providing multiple data types (JSON, Markdown, Code), the model can get confused about where one ends and the next begins.
**Solution:** Use distinct "Type Headers" and delimiters for each part of the context.

```python
advanced_isolation_prompt = """
[SOURCE: SYSTEM_LOGS | TYPE: JSON]
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
def secure_context_fetch(query: str, user_id: int):
    # This happens in the Vector DB (e.g. Pinecone/Qdrant)
    # It is a 'Hard Filter' that the AI cannot bypass.
    return db.search(query, filter={"owner_id": user_id, "is_private": False})
```
**Why this is preferred:** It is the only way to ensure **Data Privacy**. You should never rely on the LLM's "Instructions" to keep data secret; you must engineer the context so it never sees the secret data in the first place.

---

### Example 6: Dynamic "Skill" Loading (Progressive Disclosure)
**Problem:** A "Mega-Prompt" with 50 different "Skills" (how to refund, how to upgrade, how to cancel) is too noisy.
**Solution:** Use a "Router" to identify the required skill and only load the relevant context for that skill.

```python
def skill_router(user_query: str):
    # LLM identifies the intent: "User wants a refund"
    return "refund_policy"

def build_skill_aware_prompt(user_query: str):
    skill = skill_router(user_query)
    policy_context = fetch_policy_from_db(skill)
    return f"POLICY: {policy_context}\nUSER: {user_query}"
```
**Why this is preferred:** It keeps the "Attention Budget" focused. The model is 100% focused on the refund policy rather than being distracted by the cancellation or upgrade rules.

---

### Example 7: Context Resolution (Handling Conflicts)
**Problem:** Two documents in the context provide conflicting information (e.g., an old price vs. a new price).
**Solution:** Inject "Recency Metadata" and instruct the model to prioritize the most recent information.

```python
def format_with_recency(docs: list):
    formatted = ""
    for doc in docs:
        formatted += f"[Date: {doc['updated_at']}] {doc['content']}\n"

    return f"""
CONTEXT:
{formatted}

RULE: If information conflicts, the document with the LATEST Date is the truth.
"""
```
**Why this is preferred:** It provides a **Deterministic Resolution Rule** for the model's probabilistic reasoning, ensuring consistency in a world of changing data.

---

### Example 8: Self-Correction (The "Context Check")
**Problem:** The RAG system returns context that is totally irrelevant, but the model tries to "Force" an answer anyway.
**Solution:** Ask the model to first evaluate if the context is sufficient before answering.

```python
def build_self_checking_prompt(context: str, query: str):
    return f"""
STEP 1: Read the CONTEXT.
STEP 2: Determine if the CONTEXT contains the answer to the QUERY.
If NO, say 'I do not have enough info' and STOP.
If YES, provide the answer.

CONTEXT: {context}
QUERY: {query}
"""
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
