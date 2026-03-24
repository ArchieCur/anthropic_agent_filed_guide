# Model Assignment Strategy for Agent Teams

In a single agent you pick one model. In an agent team you are designing a cost and capability topology — every assignment decision affects every other one. A weak worker produces output the validator has to spend more tokens interpreting. A weak orchestrator produces malformed instructions every worker has to compensate for. Get the topology right and costs drop; get it wrong and costs compound.

---

## The Baseline Team Topology

```
ORCHESTRATOR  (Opus 4.6 or Sonnet 4.6)
      │
      ▼ routes tasks
WORKERS × N   (Haiku 4.5 or Sonnet 4.6)
      │
      ▼ produce outputs
VALIDATOR     (Sonnet 4.6)
      │
      ▼ checks outputs
ORCHESTRATOR  receives results, routes next task
```

Every agent team maps to some variation of this structure. Assign models to positions, not to agents by name.

---

## Assignment Principles

**1. Orchestrator intelligence scales with routing complexity.**
Simple, well-defined routing with deterministic task assignment: Sonnet. Ambiguous, multi-step, conditional routing where the orchestrator must reason through edge cases: Opus. Do not pay for Opus when routing is deterministic — Sonnet handles it and costs a third as much.

**2. Worker model matches actual task scope, not intended task scope.**
Well-scoped task with clear inputs and predictable outputs: Haiku. Task requiring judgment, interpretation, or handling of edge cases: Sonnet. Task requiring complex multi-step reasoning: Sonnet (or Opus — but if a worker needs Opus, task decomposition is probably wrong; that reasoning belongs in the orchestrator).

**3. Validators are consistency problems, not intelligence problems.**
Sonnet for all validators. Always. A validator needs to follow structured checking instructions reliably and consistently — not reason through novel problems. Sonnet's instruction-following consistency outperforms Opus in validator positions. Opus validators are over-engineered and more expensive for no quality gain.

**4. High-risk infrastructure agents warrant Sonnet regardless of task scope.**
Agents that run before any neighborhood forms, or agents whose output contaminates all downstream agents if it drifts, need stronger instruction-following than Haiku provides. The cost premium is justified by contagion risk.

**5. High-conflict positions are high-drift-risk positions.**
Agents operating where objectives conflict — business goals vs. compliance constraints, speed vs. accuracy — will drift under pressure more than agents with unambiguous objectives. Haiku at a high-conflict position produces plausible-but-wrong outputs at scale. Sonnet's stronger constraint-following reduces drift risk in these positions.

---

## Mixed-Model Team Example

```python
AGENT_MODELS = {
    # Orchestrator — complex routing, conditional task distribution
    "orchestrator":      "claude-opus-4-6",

    # High-risk infrastructure — runs first, output contaminates all downstream
    # Drift in data validation here = every downstream agent inherits bad evidence
    "intake":            "claude-sonnet-4-6",

    # Well-scoped compliance workers — clear inputs, deterministic checks
    "compliance_aria":   "claude-haiku-4-5-20251001",
    "compliance_petra":  "claude-haiku-4-5-20251001",
    "compliance_tax":    "claude-haiku-4-5-20251001",

    # High-conflict worker — business objectives vs. compliance constraints
    # Drift-prone by position; Sonnet's constraint-following reduces risk
    "business_vera":     "claude-sonnet-4-6",

    # Validator — consistency critical, not intelligence
    "validator":         "claude-sonnet-4-6",

    # Reporting — structured output, clear scope, fast
    "reporting":         "claude-haiku-4-5-20251001",
}
```

**Reading the topology:** 1 Opus, 3 Sonnet, 3 Haiku. The Opus budget is spent where ambiguous routing decisions happen. Sonnet covers the high-risk and high-conflict positions. Haiku handles the volume. This is not a compromise — it is correct assignment.

---

## Neighborhood Grouping — Reducing Monitoring and Cost

Agents with similar roles and similar data exposure can be grouped into neighborhoods and monitored collectively rather than individually.

```
COMPLIANCE NEIGHBORHOOD
├── compliance_aria   (Haiku)
├── compliance_petra  (Haiku)
└── compliance_tax    (Haiku)
    → Monitored as a unit: aggregate drift score first, individual drill-down only if needed
```

**Why this matters for cost:** Individual monitoring of every agent in a team scales linearly with team size. Neighborhood monitoring scales with the number of neighborhoods (typically 2–4), not the number of agents. See [observability.md](observability.md) for the two-stage monitoring pattern.

**Why INTAKE stays individually monitored:** INTAKE runs before any neighborhood forms. It is not in a neighborhood — it is a single infrastructure agent whose output all downstream agents inherit. There is no "aggregate" to compute. Individual monitoring is the only option.

---

## Failure Mode Reference

| Wrong assignment | Symptom | Cost impact |
|-----------------|---------|-------------|
| Opus on all workers | Cost exceeds budget ceiling | 3x cost vs. Sonnet workers |
| Haiku as orchestrator on complex routing | Silent incorrect task distribution, downstream failures | Multiplied recovery cost |
| Opus as validator | Over-engineered consistency check, no quality gain | ~3x validator cost for same output quality |
| Haiku at high-conflict position | Plausible-but-wrong outputs at scale, silent drift | Downstream propagation cost |
| Haiku at high-risk infrastructure position | Instruction-following failures contaminate all downstream agents | Full pipeline recovery cost |

---

*Next: [Observability](observability.md) | Back: [Cost Model](cost-model.md)*
