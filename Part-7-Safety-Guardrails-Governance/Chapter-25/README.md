# Chapter 25: Guardrails Systems

This chapter discusses the runtime controls and deterministic boundaries that ensure AI systems remain safe, accurate, and compliant.

## Key Takeaways:
- Guardrails as a separate **Software Layer** between the model and the user.
- The 3 phases: **Input Rails (Intent)**, **Dialog Rails (Flow)**, and **Output Rails (Verification)**.
- Deep technical dive into **Hallucination Containment**, **NLI-based Factuality**, and **Canonical Flows**.
- 8 real-world Python examples illustrating:
  - Implementing an **Intent-Based Input Rail**.
  - **Self-Correction** for policy-violating outputs.
  - RAG **Faithfulness Checks** using Natural Language Inference.
  - Enforcing data structures with **Guardrails AI**.
  - **Canary Probes** for detecting injection attempts.
  - Defining **Canonical User Flows** (NeMo syntax).
  - **Sensitive Data Masking** for safe observability (Presidio).
  - **Multi-Guardrail Consensus** to reduce false positives.
