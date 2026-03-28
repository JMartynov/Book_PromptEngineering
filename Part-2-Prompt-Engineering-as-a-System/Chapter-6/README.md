# Chapter 6: Evaluation-Driven Development (EDD)

This chapter focuses on the transition from "vibe-based" review to "metrics-based" engineering.

## Key Takeaways:
- The **Golden Dataset** as the ground truth of a reliable AI system.
- Tiered evaluation metrics: **Deterministic**, **Semantic**, and **Model-based (Judge)**.
- Deep technical dive into **Semantic Distance**, **BERTScore**, and **LLM-as-a-Judge** reliability.
- 8 real-world Python examples illustrating:
  - Creating a structured Golden Dataset using Pydantic.
  - The fast "Exact Match" evaluator for classification.
  - Using **JSON Schema** validation for structural integrity.
  - Semantic similarity using local SentenceTransformer models.
  - The **"Judge LLM"** pattern for rubric-based grading.
  - Building a full "Regression Test" suite.
  - Cost and latency benchmarking for production systems.
  - Multi-model ROI analysis (Quality vs. Cost).
