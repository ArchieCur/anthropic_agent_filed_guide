# Stop Reasons Decoded for Agent Loops

Stop reasons are not errors. They are the model telling you what happened and what to do next. Treating them as errors is the bug — and it is one of the most common bugs in production agent loops.

---

## Quick Reference

| Stop Reason | Meaning | Action | Common mistake |
|-------------|---------|--------|----------------|
| `end_turn` | Model finished normally | Process output, continue loop if incomplete | None — this is the happy path |
| `tool_use` | Model requesting a client-side tool | Execute tool, return results, continue | Forgetting to append assistant message before tool results |
| `pause_turn` | Model yielded for server-side tool | Continue the turn — do NOT restart | Treating as error, restarting loop |
| `max_tokens` | Response truncated at token limit | Decide: use partial or re-issue | Discarding output, or not noticing mid-JSON truncation |
| `stop_sequence` | Hit a custom stop sequence | Application-specific | N/A unless you use custom stop sequences |
| `refusal` | Model declined the request | Handle gracefully, do not retry same input | Treating as system failure, crashing the loop |

---

## `end_turn`

**Meaning:** Model completed its response normally.

**Action:** Process output. If the agent task is incomplete, continue the loop with the next message. No special handling required.

**Notes:** The happy path. The only stop reason that confirms the model finished what it started.

---

## `tool_use`

**Meaning:** Model is requesting execution of a client-side tool defined in your `tools` array.

**Action:** Execute the requested tool. Append the assistant message (with the tool use block) to your messages array. Append the tool result. Continue the loop.

**The order matters:**
```python
# Correct
messages.append({"role": "assistant", "content": response.content})  # append first
messages.append({"role": "user", "content": tool_results})            # then results
# Continue loop
```

**Common mistake:** Appending tool results without first appending the assistant message that requested them. This produces an invalid message structure and a 400 on the next request.

**Note:** `tool_use` and `pause_turn` are related but distinct. `tool_use` is the model requesting a **client-side** tool you defined. `pause_turn` is the model yielding for a **server-side** tool (like web search or code execution running on Anthropic's infrastructure).

---

## `pause_turn`

**Meaning:** The model yielded mid-turn because a server-side tool is executing. It expects to continue the turn when results are available. **This is not an error.**

**Action:** Execute pending server-side tool calls. Append results to messages. Re-submit to continue — do not restart the turn from the beginning.

**The mishandling pattern:** Agent loops that treat any non-`end_turn` stop reason as terminal will throw or restart on `pause_turn`. Restarting creates **duplicate tool calls and corrupted state** — the model already started something, and restarting asks it to start over without that context.

```python
# Correct pause_turn handling
if response.stop_reason == "pause_turn":
    messages.append({"role": "assistant", "content": response.content})
    # Add server-side tool results to messages
    continuation = client.messages.create(
        model=model,
        messages=messages,
        tools=tools,
        max_tokens=max_tokens
    )
    # Continue — do NOT restart the loop
```

**Why this matters:** `pause_turn` is the most commonly mishandled stop reason in production. The symptom of incorrect handling is usually not an obvious error — it's duplicate work, unexpected state, or tools appearing to run twice.

---

## `max_tokens`

**Meaning:** The model's response was cut off because it reached the `max_tokens` limit. The output is incomplete.

**Action:** Do not automatically discard. Assess: is the partial response usable? Options:
1. Use the partial response as-is if it's sufficient for the task
2. Re-issue with a higher `max_tokens` value
3. Log and fail gracefully if the truncation makes the output unusable

**Agent-specific warning — mid-JSON truncation:** In agent loops, hitting `max_tokens` can cut off a tool call mid-argument. The result is a partial JSON object that will fail to parse. This is not recoverable from the response — the only fix is re-issuing the request.

```python
import json

def check_for_truncated_tool_call(response):
    """Detect mid-JSON truncation before it crashes the parser."""
    for block in response.content:
        if block.type == "tool_use":
            try:
                # Tool call arguments should be a complete dict
                if not isinstance(block.input, dict):
                    return True  # Truncated
            except Exception:
                return True
    return False
```

**Per-role `max_tokens` guidance — set these per agent role, not globally:**

| Role | Recommended `max_tokens` | Rationale |
|------|--------------------------|-----------|
| Orchestrator | 4096–8192 | Needs to reason, plan, and produce detailed instructions |
| Worker | 1024–4096 | Scoped task, clear output — higher values waste budget |
| Validator | 512–1024 | Pass/fail, brief feedback — rarely needs more |

A global `max_tokens` set for the orchestrator will silently truncate workers on longer tasks, or waste budget on validators.

---

## `stop_sequence`

**Meaning:** The model's output reached a custom stop sequence defined in your request's `stop_sequences` parameter.

**Action:** Process output up to the stop sequence. Handling is application-specific.

**Notes:** Only relevant if you are using custom stop sequences. If you are not, you will not see this stop reason.

---

## `refusal`

**Meaning:** The model declined to complete the request.

**Action:** Handle gracefully. Do not retry with the same input — the model will refuse again. Log for review. This is not a system failure.

**Breaking change — Claude 4 migration:** `refusal` is a new stop reason added in Claude 4. Systems migrating from Claude 3.x that do not explicitly handle `refusal` will encounter an unhandled stop reason and may crash or behave unexpectedly. Add `refusal` to your stop reason handler before migrating.

**Agent-specific:** Refusals in agent pipelines should route to a fallback handler, not crash the loop. A refusal on one agent should not take down the whole pipeline.

---

*Next: [Recovery Patterns](recovery-patterns.md) | Back: [Error Types](error-types.md)*
