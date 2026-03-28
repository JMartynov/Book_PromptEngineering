# Chapter 7: Prompt Versioning & Testing (PromptOps)

This chapter discusses the practices and tools for managing prompts as versioned code artifacts.

## Key Takeaways:
- The concept of **PromptOps** and why it's necessary for scaling AI teams.
- Moving from hardcoded strings to **Git-versioned YAML/Markdown** files.
- The distinction between **Unit Tests (Fast)** and **Regression Tests (Comprehensive)**.
- Deep technical dive into the **Model-Prompt-Setting Triad** and **Canary Deployments**.
- 8 real-world Python examples illustrating:
  - Externalizing prompts to YAML with metadata.
  - The "Prompt Loader" utility for specific versioning.
  - Unit testing prompt output with Pytest for edge cases.
  - Creating a fast "Smoke Test" suite for developer feedback.
  - **Canary Releases** using user-id based feature flags.
  - Environment-based (Dev/Staging/Prod) prompt management.
  - Automated **Markdown Regression Reporting**.
  - Versioning model hyper-parameters (temperature, top_p) for reproducibility.
