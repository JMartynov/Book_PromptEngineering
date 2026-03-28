# Chapter 13: Prompt Optimization Algorithms

This chapter introduces the algorithms used to find the "Global Maximum" of prompt performance.

## Key Takeaways:
- Prompting as a **Search Problem**.
- The hierarchy of optimizers (Teleprompters): **BootstrapFewShot**, **MIPROv2**, and **COPRO**.
- Deep technical dive into **Bayesian Optimization**, **Surrogate Models**, and the **Pareto Frontier**.
- 8 real-world Python examples illustrating:
  - Basic **BootstrapFewShot** setup for complex classification.
  - Qualitative optimization using **LLM-as-a-Judge** metrics.
  - Multi-objective search with **MIPROv2** (Accuracy vs. Cost).
  - Automated **Zero-Shot** instruction proposal.
  - Eliminating common phrases using **Negative Penalty Metrics**.
  - Avoiding local maxima with **RandomSearch**.
  - **Cross-Model Compilation** for model-specific "dialects."
  - **Layered Optimization** for multi-stage agentic pipelines.
