# Chapter 29: Why Prompts “Break”

This chapter explores the fundamental technical reasons behind the fragility of natural language prompts.

## Key Takeaways:
- Prompts as the most fragile component of the AI stack.
- Deep technical dive into **Token Sensitivity**, **Distribution Mismatch (Model Drift)**, and **Analogical Overfitting**.
- Understanding the **U-Shaped Attention Curve** and **Sycophancy Bias**.
- 8 real-world Python examples illustrating:
  - **Semantic Variance Testing** for prompt robustness.
  - **Double-Anchoring** critical rules for attention stability.
  - **Anti-Sycophancy** instructions for truth-seeking agents.
  - **Diverse Few-Shotting** to improve generalization.
  - **Versioned Model Pinning** to prevent silent breaks.
  - Detecting **Instruction Bleed** using neutrality guardrails.
  - The **Zero-Shot Stability Test** for instruction quality.
  - Using hard **Stop Sequences** for deterministic formatting.
