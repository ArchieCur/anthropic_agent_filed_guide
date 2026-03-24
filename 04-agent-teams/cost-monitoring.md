# Cost Monitoring Patterns

Agent teams can burn through budget faster than any other usage pattern on the platform. A runaway agent loop — one that fails to terminate, hits retry loops, or accumulates context without bound — will exhaust your spend limit before you notice. These patterns give you the early warning system.

---

## Pattern 1 — Per-Agent Cost Tracker

Identifies which agents are driving cost in a team. The first question when a run comes back expensive is always "which agent?"

```python
class AgentCostTracker:
    """
    Track token consumption and estimated cost per agent.
    Pricing sourced from https://anthropic.com/pricing — verify current rates before use.
    Rates change; hardcoded values go stale.
    """

    # Approximate costs per million tokens — CHECK https://anthropic.com/pricing FOR CURRENT RATES
    COSTS_PER_M = {
        "claude-opus-4-6":           {"input": 15.0,  "output": 75.0,  "cache_read": 1.50},
        "claude-sonnet-4-6":         {"input": 3.0,   "output": 15.0,  "cache_read": 0.30},
        "claude-haiku-4-5-20251001": {"input": 0.80,  "output": 4.0,   "cache_read": 0.08},
    }
    # Cache creation costs 25% more than standard input (first write only)
    CACHE_CREATION_MULTIPLIER = 1.25

    def __init__(self, model: str, agent_id: str):
        self.model = model
        self.agent_id = agent_id
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_cache_read_tokens = 0
        self.total_cache_creation_tokens = 0
        self.turn_count = 0

    def record(self, response) -> None:
        usage = response.usage
        self.total_input_tokens          += getattr(usage, 'input_tokens', 0) or 0
        self.total_output_tokens         += getattr(usage, 'output_tokens', 0) or 0
        self.total_cache_read_tokens     += getattr(usage, 'cache_read_input_tokens', 0) or 0
        self.total_cache_creation_tokens += getattr(usage, 'cache_creation_input_tokens', 0) or 0
        self.turn_count += 1

    def estimated_cost(self) -> float:
        rates = self.COSTS_PER_M.get(self.model, self.COSTS_PER_M["claude-sonnet-4-6"])
        input_cost    = (self.total_input_tokens / 1_000_000) * rates["input"]
        output_cost   = (self.total_output_tokens / 1_000_000) * rates["output"]
        cache_read    = (self.total_cache_read_tokens / 1_000_000) * rates["cache_read"]
        cache_create  = (self.total_cache_creation_tokens / 1_000_000) * rates["input"] * self.CACHE_CREATION_MULTIPLIER
        return input_cost + output_cost + cache_read + cache_create

    def summary(self) -> dict:
        return {
            "agent_id":            self.agent_id,
            "model":               self.model,
            "turns":               self.turn_count,
            "input_tokens":        self.total_input_tokens,
            "output_tokens":       self.total_output_tokens,
            "cache_read_tokens":   self.total_cache_read_tokens,
            "cache_create_tokens": self.total_cache_creation_tokens,
            "estimated_cost_usd":  round(self.estimated_cost(), 4),
        }
```

**Note on pricing:** Haiku 4.5 input rate is $0.80/M (corrected from $1.00 in some references). Always verify at [anthropic.com/pricing](https://anthropic.com/pricing) before deploying cost-sensitive guardrails — rates change and stale hardcoded values will miscalculate your guardrails.

---

## Pattern 2 — Team-Level Spend Guardrail with Graceful Shutdown

Hard `RuntimeError` mid-turn is the wrong behavior for production. A hard stop leaves tool calls in flight, corrupts agent state, and produces partial outputs that downstream agents may attempt to use.

The correct behavior: **signal agents to stop after their current turn completes.** No new turns start; in-progress turns finish cleanly.

```python
import threading

class TeamSpendGuardrail:
    """
    Graceful spend limit for agent team runs.
    Sets a halt flag checked before each new turn — does NOT interrupt mid-turn.
    In-progress turns complete. New turns do not start.
    """

    def __init__(self, max_spend_usd: float, warn_at: float = 0.80):
        self.max_spend_usd = max_spend_usd
        self.warn_at = warn_at
        self.trackers: dict[str, AgentCostTracker] = {}
        self._halt = threading.Event()  # thread-safe halt signal

    def add_agent(self, agent_id: str, model: str) -> None:
        self.trackers[agent_id] = AgentCostTracker(model, agent_id)

    def record(self, agent_id: str, response) -> None:
        if agent_id in self.trackers:
            self.trackers[agent_id].record(response)
        self._check_limits()

    def should_halt(self) -> bool:
        """Check before starting each new agent turn."""
        return self._halt.is_set()

    def _check_limits(self) -> None:
        total = sum(t.estimated_cost() for t in self.trackers.values())
        ratio = total / self.max_spend_usd

        if ratio >= 1.0:
            self._halt.set()
            print(
                f"HALT: Team spend limit reached — "
                f"${total:.4f} of ${self.max_spend_usd:.4f}. "
                f"In-progress turns will complete. No new turns will start."
            )
        elif ratio >= self.warn_at:
            print(
                f"WARNING: Team spend at {ratio:.0%} of limit "
                f"(${total:.4f} / ${self.max_spend_usd:.4f})"
            )

    def report(self) -> list[dict]:
        """Cost breakdown by agent — use to identify which agents drove cost."""
        return sorted(
            [t.summary() for t in self.trackers.values()],
            key=lambda x: x["estimated_cost_usd"],
            reverse=True
        )

    def total_cost(self) -> float:
        return sum(t.estimated_cost() for t in self.trackers.values())
```

**Integration — check before each turn:**
```python
# In your agent loop:
while True:
    if guardrail.should_halt():
        print(f"Agent {agent_id} halting at turn {turn} — team spend limit reached.")
        break

    response = call_with_retry(client, model=model, messages=messages, ...)
    guardrail.record(agent_id, response)
    # ... handle stop reason
```

---

## Pattern 3 — Runaway Loop Detection

A loop that reaches `max_turns` without `end_turn` is almost always a bug — a missing termination condition, a tool that always returns "retry," or a task that was never scoped to terminate. Catch it before it empties the token budget.

```python
class LoopGuardrail:
    """
    Detect and signal agent loops that fail to terminate.
    Raises on hard limits — these are bugs, not normal conditions.
    """

    def __init__(self, max_turns: int = 20, max_tool_calls: int = 50):
        self.max_turns = max_turns
        self.max_tool_calls = max_tool_calls
        self.turn_count = 0
        self.tool_call_count = 0

    def check(self, response) -> None:
        self.turn_count += 1
        tool_calls = sum(
            1 for block in response.content
            if hasattr(block, 'type') and block.type == 'tool_use'
        )
        self.tool_call_count += tool_calls

        if self.turn_count >= self.max_turns:
            raise RuntimeError(
                f"Agent loop exceeded max_turns={self.max_turns}. "
                f"Termination condition bug — loop never reached end_turn."
            )
        if self.tool_call_count >= self.max_tool_calls:
            raise RuntimeError(
                f"Agent exceeded max_tool_calls={self.max_tool_calls}. "
                f"Possible tool call loop — tool always returning retry signal."
            )
```

**Recommended limits by agent role:**

| Role | max_turns | max_tool_calls | Rationale |
|------|-----------|----------------|-----------|
| Orchestrator | 30 | 100 | Coordinates multiple subtasks across multiple agents |
| Worker | 10 | 20 | Well-scoped task — should terminate quickly |
| Validator | 5 | 10 | Assessment task — fast, or something is wrong |
| Monitor | 3 | 5 | Single-purpose check — should complete in 1–2 turns |

If a worker regularly hits turn 8–9, the task scope is wrong — it is not a worker task.

---

## Putting It Together — Instrumented Agent Run

```python
def run_instrumented_agent(
    client, agent_id: str, model: str, messages: list, tools: list,
    guardrail: TeamSpendGuardrail, max_tokens: int = 4096
):
    """
    Single agent run with cost tracking, loop detection, and graceful shutdown.
    """
    loop = LoopGuardrail(
        max_turns={"claude-opus-4-6": 30}.get(model, 10),
        max_tool_calls={"claude-opus-4-6": 100}.get(model, 20)
    )
    turn = 0

    while True:
        # Check spend limit before starting new turn
        if guardrail.should_halt():
            print(f"{agent_id}: Halting at turn {turn} — team spend limit signaled.")
            return None

        response = call_with_retry(
            client, model=model, messages=messages,
            tools=tools, max_tokens=max_tokens
        )

        # Record for cost and observability
        guardrail.record(agent_id, response)
        print(log_agent_turn(agent_id, turn, response, messages))

        # Check for runaway loop
        loop.check(response)
        turn += 1

        # Handle stop reason
        if response.stop_reason == "end_turn":
            return response

        elif response.stop_reason == "tool_use":
            tool_results = execute_tools(response.content)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})

        elif response.stop_reason == "pause_turn":
            messages.append({"role": "assistant", "content": response.content})

        elif response.stop_reason == "refusal":
            print(f"{agent_id}: Refusal at turn {turn}. Logging.")
            return response

        else:
            print(f"{agent_id}: Unhandled stop reason '{response.stop_reason}'.")
            return response
```

---

## Pre-Run Checklist

Before running an agent team:

- [ ] `TeamSpendGuardrail` initialized with a per-run budget
- [ ] `LoopGuardrail` limits set per agent role (not globally)
- [ ] `AgentCostTracker` registered for each agent
- [ ] `log_agent_turn` wired to a persistent log (not just stdout)
- [ ] `NeighborhoodMonitor` configured for each agent group
- [ ] Pricing in `AgentCostTracker.COSTS_PER_M` verified against [anthropic.com/pricing](https://anthropic.com/pricing)
- [ ] Compaction threshold set (75% recommended — see [cost-model.md](cost-model.md))

---

*Back: [Observability](observability.md) | [Model Assignment](model-assignment.md) | [Cost Model](cost-model.md)*
