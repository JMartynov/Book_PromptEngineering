# Chapter 23: Prompt Injection Defense

This chapter discusses the architectural and programmatic strategies for securing AI systems against malicious instructions.

## Key Takeaways:
- Moving from "Clever Wording" to **Architectural Defense**.
- The 3-layer defense hierarchy: **Instruction Hierarchy**, **Context Isolation**, and **Real-time Guardrails**.
- Deep technical dive into **Semantic Boundaries**, **Defense in Depth**, and **Indirect Injection** patterns.
- 8 real-world Python examples illustrating:
  - Enforcing **XML Isolation** with strict processing rules.
  - Using a **Secondary Safety Model** (Llama-Guard) as a firewall.
  - **Instruction Priority Tagging** to guide model attention.
  - Deterministic **Output Sanitization** using regex fail-safes.
  - **Indirect Injection** defense for RAG systems.
  - **Tool Isolation** via parameterized API calls.
  - Implementing **Honeypot Tokens** for threat intelligence.
  - **Multi-Token Verification** using unique request-id delimiters.
