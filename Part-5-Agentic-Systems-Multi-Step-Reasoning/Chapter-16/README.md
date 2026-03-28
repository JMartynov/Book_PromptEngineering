# Chapter 16: From Prompts to Agents

This chapter discusses the architectural shift from "Prompt-Response" models to **Goal-Oriented AI Agents**.

## Key Takeaways:
- AI Agents as autonomous systems that **Plan**, **Act**, and **Self-Correct**.
- The 4 pillars: **Planner**, **Tool Interface**, **Persistent Memory**, and the **Executor/Critic Loop**.
- Deep technical dive into **Plan-and-Execute** patterns, **Skill-based Tooling**, and **Reflection-driven Memory**.
- 8 real-world Python examples illustrating:
  - Defining structured **Mission Plans** with Pydantic.
  - Using **Typed Tools** for safe function calling.
  - The **ReAct Loop** for transparent reasoning.
  - Adding a **Critic Node** for automated verification.
  - **Episodic Memory** management across user sessions.
  - **Decoupled Architecture** (Planner vs. Executor).
  - **Human-in-the-Loop (HITL)** safety guardrails.
  - Basic **Manager-Worker** delegation patterns.
