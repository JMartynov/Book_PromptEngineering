# Chapter 28: Common Failures

This chapter maps out the most common engineering failures in modern AI systems and provides patterns to avoid them.

## Key Takeaways:
- Moving from LLM as a "Magic Box" to a **Stochastic Software Component**.
- The 4 major failure modes: **Mega-Prompt Bloat**, **Semantic Drift**, **Narrative Lock-in**, and **Memory Overwrite**.
- Deep technical dive into **Attention Collapse**, **Error Propagation**, and **Stochastic Feedback Loops**.
- 8 real-world Python examples illustrating:
  - Replacing monolithic mega-prompts with **Modular Pipelines**.
  - Using **Grounding Anchors** to counter Narrative Lock-in.
  - Implementing **Structural Guardrails** to prevent silent regressions.
  - Optimization via **Reranking** vs. context dumping.
  - Using **Summary Memory** to prevent state corruption.
  - Replacing "Magic Adjectives" with **Deterministic Checklists**.
  - Model-agnostic logic with **DSPy Signatures**.
  - Replacing manual testing with **Golden Dataset Evaluations**.
