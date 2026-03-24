# Section 04 — Agent Teams

*Triage: if you're here, you're dealing with runaway costs, coordination failures, or trying to design a multi-agent system that stays observable.*

---

## In This Section

- [The Cost Model — Why 7x and How to Design Around It](cost-model.md)
- [Model Assignment Strategy for Agent Teams](model-assignment.md)
- [Cost Monitoring Patterns (copy-paste)](cost-monitoring.md)

---

## Key Operational Facts

- Each agent teammate runs its own full context window. Token usage scales roughly linearly with team size.
- Agent teams use approximately **7x more tokens** than standard sessions. This is a floor, not a ceiling.
- Sonnet is the recommended model for teammates. Opus at the teammate level is rarely justified and dramatically changes the cost model.
- **Tool result tokens compound fast.** Long tool outputs fed into agent context accumulate across turns. Truncate and summarize before returning to the agent.

---


