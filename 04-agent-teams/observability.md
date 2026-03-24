# Observability — Making Agent Teams Debuggable

A single agent that fails produces an error. An agent team that fails produces a plausible-looking wrong answer — and by the time you see it, three other agents have built on top of it. Observability in agent teams is not optional instrumentation. It is the difference between catching drift at one agent and catching it after it has propagated through the pipeline.

---

## What to Log Per Agent Turn

Log everything. Storage is cheap. Debugging blind is not.

```python
import time

def log_agent_turn(agent_id: str, turn_number: int, response, messages: list) -> dict:
    """
    Minimum viable logging for agent team observability.
    Emit this on every turn for every agent.
    """
    return {
        "agent_id":           agent_id,
        "turn":               turn_number,
        "stop_reason":        response.stop_reason,
        "input_tokens":       response.usage.input_tokens,
        "output_tokens":      response.usage.output_tokens,
        "cache_hit_tokens":   getattr(response.usage, 'cache_read_input_tokens', 0) or 0,
        "context_utilization": response.usage.input_tokens / 200_000,
        "output_preview":     str(response.content[0])[:200] if response.content else "",
        "timestamp":          time.time(),
    }
```

**What each field tells you at 2am:**

| Field | What it surfaces |
|-------|-----------------|
| `stop_reason` | Unexpected `max_tokens` or `refusal` on a specific agent |
| `context_utilization` | Which agent is approaching the compaction threshold |
| `cache_hit_tokens` | Whether caching is working per agent |
| `output_preview` | First signal of vocabulary drift without reading the full output |
| `turn` count per agent | Which agents are running long (Driver 2 cost alarm) |

---

## The Four Drift Signals

Drift in agent systems is positional and contagious — it compounds across turns and across agents. These four signals appear before semantic errors do. Catch them early.

**Signal 1 — Vocabulary migration**
Agent output registers shift from neutral/technical toward urgency, recovery, or opportunity framing. The determination is technically still present; the framing has changed.

> "This invoice has compliance findings" becomes "this invoice is a strong recovery candidate."

**Signal 2 — Hedged absolutes**
MUST constraints acquire softening language. "Generally," "in most cases," "typically" appearing adjacent to non-negotiable requirements. The constraint is still cited; it is no longer treated as absolute.

**Signal 3 — Selective omission**
Agent stops mentioning findings that were present in earlier turns. Not a retraction — an absence. Only detectable by comparing outputs across turns.

**Signal 4 — Self-reinforcing structure**
Output structure begins encoding drifted assumptions before body text does. Section headers and organization reflect the drifted framing first.

> "Immediate Pipeline Candidates" as a section header in an output that should be reporting compliance findings — the structure drifted before the content.

---

## Two-Stage Neighborhood Monitoring

Monitor neighborhoods collectively. Only drill down on individual agents when the aggregate signal warrants it. This keeps monitoring cost sub-linear with team size.

```python
from dataclasses import dataclass, field
from typing import Literal

DriftLevel = Literal["LOW", "MEDIUM", "HIGH"]

@dataclass
class NeighborhoodMonitor:
    """
    Two-stage monitoring for a group of agents with similar roles.
    Stage 1 always runs (aggregate). Stage 2 conditional (individual).
    """
    neighborhood_id: str
    agent_ids: list[str]
    baseline_terms: list[str]   # vocabulary expected in healthy output
    drift_terms: list[str]      # vocabulary indicating drift
    stage2_threshold: DriftLevel = "MEDIUM"

    _outputs: dict = field(default_factory=dict)  # agent_id -> latest output

    def record_output(self, agent_id: str, output: str) -> None:
        self._outputs[agent_id] = output

    def stage1_assessment(self) -> tuple[DriftLevel, dict]:
        """
        Aggregate drift assessment across the neighborhood.
        Fast first pass — runs on every cycle.
        """
        scores = {}
        for agent_id, output in self._outputs.items():
            scores[agent_id] = _vocabulary_drift_score(
                output, self.baseline_terms, self.drift_terms
            )

        if not scores:
            return "LOW", scores

        avg_drift = sum(s["drift_ratio"] for s in scores.values()) / len(scores)
        max_drift = max(s["drift_ratio"] for s in scores.values())

        if max_drift > 0.5 or avg_drift > 0.35:
            level = "HIGH"
        elif max_drift > 0.3 or avg_drift > 0.20:
            level = "MEDIUM"
        else:
            level = "LOW"

        return level, scores

    def stage2_drill_down(self, scores: dict) -> list[dict]:
        """
        Individual agent assessment. Only call when Stage 1 returns MEDIUM/HIGH.
        Classify each flagged agent by drift tier.
        """
        flagged = []
        for agent_id, score in scores.items():
            if score["drift_ratio"] > 0.3:
                tier = _classify_drift_tier(score, self._outputs.get(agent_id, ""))
                flagged.append({
                    "agent_id": agent_id,
                    "tier": tier,
                    "score": score,
                    "recommended_action": _escalation_action(tier, score),
                })
        return flagged

    def run(self) -> dict:
        """Full monitoring cycle."""
        level, scores = self.stage1_assessment()
        result = {"neighborhood": self.neighborhood_id, "aggregate_level": level}

        if level in ("MEDIUM", "HIGH"):
            result["flagged_agents"] = self.stage2_drill_down(scores)
            print(
                f"Neighborhood {self.neighborhood_id}: {level} drift signal. "
                f"{len(result['flagged_agents'])} agent(s) flagged."
            )
        return result


def _vocabulary_drift_score(output: str, baseline_terms: list, drift_terms: list) -> dict:
    lower = output.lower()
    baseline_hits = sum(1 for t in baseline_terms if t in lower)
    drift_hits    = sum(1 for t in drift_terms    if t in lower)
    baseline_ratio = baseline_hits / len(baseline_terms) if baseline_terms else 0
    drift_ratio    = drift_hits    / len(drift_terms)    if drift_terms    else 0
    return {
        "drift_ratio":       drift_ratio,
        "baseline_ratio":    baseline_ratio,
        "drift_terms_found": [t for t in drift_terms if t in lower],
    }


def _classify_drift_tier(score: dict, output: str) -> int:
    """
    Tier 1 — Contextual drift: vocabulary/framing shifted, recoverable via re-grounding.
    Tier 2 — Structural drift: output structure encoding drifted assumptions.
              Not recoverable via re-grounding alone.
    """
    structural_signals = [
        "pipeline candidate", "recovery candidate", "expedited",
        "immediate action", "strong candidate",
    ]
    lower = output.lower()
    has_structural = any(sig in lower for sig in structural_signals)

    if has_structural or score["drift_ratio"] > 0.6:
        return 2
    return 1


def _escalation_action(tier: int, score: dict) -> str:
    drift = score["drift_ratio"]
    if tier == 1 and drift < 0.5:
        return "Level 2: Re-ground agent — remind of constraints"
    elif tier == 1 and drift >= 0.5:
        return "Level 3A: Prune agent prior outputs, re-run"
    elif tier == 2 and drift < 0.6:
        return "Level 3B: Prune upstream compliance language, re-run"
    else:
        return "Level 4: HALT — downstream agents must not run"
```

---

## The Escalation Ladder

```
Level 1 — Monitor and log. No intervention. Continue.
Level 2 — Re-ground the drifting agent. Inject constraint reminder. Continue.
Level 3A — Prune agent's prior drifted outputs. Re-run the agent from earlier state.
Level 3B — Prune upstream compliance language that the agent has been building on.
            Re-run the agent.
Level 4 — HALT. Downstream agents must not run on contaminated output.
```

**Why Level 3 has two variants:** Drift source matters for recovery. If the drift originated within the agent's own outputs (3A), prune its history. If the drift is being inherited from upstream agents (3B), pruning the agent's own history doesn't help — you need to clean what it's reading.

---

## Production Gotchas — What Isn't in the Docs

**Cascade failures compound cost.** When one agent fails and the orchestrator retries it, the retry starts from the already-accumulated context of the failed turn. The failed turn's tokens are paid twice. On a long-running agent that failed at turn 15, retrying from turn 1 to reproduce the failure means paying for turns 1–15 again. Design retry logic to restart from the earliest recoverable checkpoint, not turn 1.

**Parallel agents writing to shared state create silent race conditions.** Two agents updating the same external resource simultaneously — a document, a database record, a shared file — will produce inconsistent state. The API doesn't coordinate this. Your application must. Agents that write to shared state need explicit coordination (locks, queues, or sequential execution for that resource).

**Orchestrator message routing errors are silent.** If the orchestrator routes Agent B's results back to Agent A, Agent A receives them, produces output based on incorrect input, and the pipeline continues. No error. No stop. Just a subtly wrong downstream result. Log routing decisions explicitly — which agent received which results — and make them auditable.

**Context quality degrades before the 413 arrives.** As an agent approaches its context limit, instruction-following quality drops before the hard limit is hit. The agent is still running; it is producing lower-quality outputs. This is invisible without context utilization monitoring. The `context_utilization` field in `log_agent_turn()` is your early warning — watch for agents consistently running above 70% utilization.

**Tool call deduplication is your responsibility.** Two agents in a team making the same external API call (same database query, same search, same file read) both get billed and may get inconsistent results if the external state changed between calls. The platform does not deduplicate. For shared external lookups, route through a shared cache at the application layer.

---

*Next: [Cost Monitoring](cost-monitoring.md) | Back: [Model Assignment](model-assignment.md)*
