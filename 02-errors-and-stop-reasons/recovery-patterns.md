# Recovery Patterns

Copy-paste production patterns for the failure modes you will actually hit. Start with Pattern 1 for any retry scenario. Add the others as your system matures.

---

## Pattern 1 — Retry with Exponential Backoff

Handles 429 (both subtypes), 500, and 529. Non-retryable errors (400, 404, 413) re-raise immediately.

```python
import anthropic
import time
import random

def call_with_retry(client, max_retries=3, base_delay=1.0, **kwargs):
    """
    Retry on transient errors with exponential backoff.
    Distinguishes 429 subtypes by response body.
    Non-retryable errors (400, 404, 413) re-raise immediately.
    """
    for attempt in range(max_retries):
        try:
            return client.messages.create(**kwargs)

        except anthropic.RateLimitError as e:
            if attempt == max_retries - 1:
                raise

            # Distinguish 429 subtypes from response body
            error_message = str(e).lower()
            is_acceleration_limit = "acceleration" in error_message

            if is_acceleration_limit:
                # Acceleration limit: ramp gradually, don't just wait
                # Waiting and retrying at the same concurrency re-triggers immediately
                wait = base_delay * (3 ** attempt) + random.uniform(0, 2)
                print(f"Acceleration limit hit. Ramping down. Waiting {wait:.1f}s (attempt {attempt + 1}/{max_retries})")
            else:
                # Standard rate limit: respect retry-after header
                retry_after = getattr(e, 'retry_after', None)
                wait = float(retry_after) if retry_after else base_delay * (2 ** attempt)
                print(f"Rate limit hit. Waiting {wait:.1f}s (attempt {attempt + 1}/{max_retries})")

            time.sleep(wait)

        except anthropic.APIStatusError as e:
            if e.status_code in (500, 529):
                if attempt == max_retries - 1:
                    raise
                wait = base_delay * (2 ** attempt) + random.uniform(0, 1)
                print(f"Transient error {e.status_code}. Waiting {wait:.1f}s (attempt {attempt + 1}/{max_retries})")
                time.sleep(wait)
            else:
                # 400, 404, 413 — not retryable, fix the request
                raise

    raise Exception(f"Max retries ({max_retries}) exceeded")
```

**On 500s at high context:** Transient 500s are more frequent above ~100k tokens. For long-context agent runs, increase `max_retries` to 5 and `base_delay` to 2.0. A 500 that clears on retry 3 is normal at this context size.

---

## Pattern 2 — Proactive Context Window Monitoring

Prevents the 413 that arrives without warning. Call this before each request in a long-running agent loop.

```python
def check_context_headroom(
    client, model, messages, tools=None, threshold=0.80
) -> tuple[bool, float]:
    """
    Count tokens before sending. Returns (ok, utilization).
    Alert at threshold% of context limit — prune before proceeding.
    The 413 gives no warning. This is your early signal.
    """
    response = client.messages.count_tokens(
        model=model,
        messages=messages,
        tools=tools or []
    )

    context_limits = {
        "claude-opus-4-6": 200000,
        "claude-sonnet-4-6": 200000,
        "claude-haiku-4-5-20251001": 200000,
    }

    limit = context_limits.get(model, 200000)
    utilization = response.input_tokens / limit

    if utilization >= threshold:
        print(
            f"WARNING: Context at {utilization:.0%} capacity "
            f"({response.input_tokens}/{limit} tokens). Prune before proceeding."
        )
        return False, utilization

    return True, utilization


def prune_messages(messages, keep_system=True, keep_recent=10):
    """
    Minimal context pruning: keep system message and N most recent turns.
    Replace with summarization for production systems with long histories.
    """
    if not messages:
        return messages

    system_messages = [m for m in messages if m.get("role") == "system"]
    non_system = [m for m in messages if m.get("role") != "system"]

    pruned = non_system[-keep_recent:] if len(non_system) > keep_recent else non_system

    return (system_messages + pruned) if keep_system else pruned
```

---

## Pattern 3 — Complete Stop Reason Handler for Agent Loops

Handles all six stop reasons correctly. Drop this into any agent loop.

```python
def run_agent_loop(client, model, messages, tools, max_tokens=4096):
    """
    Agent loop with correct stop reason handling.
    Stop reasons are signals, not errors.
    """
    while True:
        # Check context headroom before each request
        ok, utilization = check_context_headroom(client, model, messages, tools)
        if not ok:
            messages = prune_messages(messages)

        response = call_with_retry(
            client,
            model=model,
            messages=messages,
            tools=tools,
            max_tokens=max_tokens
        )

        stop_reason = response.stop_reason

        if stop_reason == "end_turn":
            return response  # Complete

        elif stop_reason == "tool_use":
            # Execute client-side tools, append results, continue
            tool_results = execute_tools(response.content)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
            # Loop continues

        elif stop_reason == "pause_turn":
            # Server-side tool yielded — NOT an error, do NOT restart
            messages.append({"role": "assistant", "content": response.content})
            # Loop continues — model resumes the same turn

        elif stop_reason == "max_tokens":
            # Check for truncated tool call before continuing
            if check_for_truncated_tool_call(response):
                print("WARNING: Tool call truncated mid-JSON. Re-issuing with higher max_tokens.")
                max_tokens = int(max_tokens * 1.5)
                # Do not append truncated response — re-issue the same messages
            else:
                print(f"WARNING: Response truncated at max_tokens={max_tokens}")
                messages.append({"role": "assistant", "content": response.content})
            # Loop continues

        elif stop_reason == "refusal":
            print("Model refusal. Logging for review. Not retrying same input.")
            return response  # Handle gracefully — not a system failure

        elif stop_reason == "stop_sequence":
            return response  # Application-specific — treat as complete

        else:
            print(f"Unhandled stop reason: {stop_reason}. Treating as complete.")
            return response


def check_for_truncated_tool_call(response) -> bool:
    """Detect mid-JSON truncation in tool call arguments."""
    for block in response.content:
        if hasattr(block, 'type') and block.type == "tool_use":
            if not isinstance(block.input, dict):
                return True
    return False


def execute_tools(content_blocks) -> list:
    """
    Placeholder — replace with your tool execution logic.
    Returns tool result blocks in the format the API expects.
    """
    results = []
    for block in content_blocks:
        if hasattr(block, 'type') and block.type == "tool_use":
            result = dispatch_tool(block.name, block.input)
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result
            })
    return results
```

---

## Pattern 4 — Streaming with Interruption Recovery

Streaming and tool calls interact dangerously. If a stream is interrupted mid-tool-call, the partial JSON in the argument deltas is not recoverable. Do not attempt to reconstruct it — re-issue.

```python
import anthropic

def stream_with_recovery(client, max_retries=2, **kwargs):
    """
    Stream a response with fallback to non-streaming on interruption.
    If stream is interrupted mid-tool-call, re-issue without streaming
    to get a complete, parseable response.
    """
    for attempt in range(max_retries):
        collected_content = []
        stop_reason = None

        try:
            with client.messages.stream(**kwargs) as stream:
                for event in stream:
                    # Collect content as it arrives
                    pass

                # Get the final message after stream completes
                final_message = stream.get_final_message()
                return final_message

        except anthropic.APIConnectionError:
            if attempt == max_retries - 1:
                raise
            print(f"Stream interrupted (attempt {attempt + 1}). Falling back to non-streaming.")
            # Fall through to non-streaming retry

        except Exception as e:
            if attempt == max_retries - 1:
                raise
            print(f"Stream error: {e}. Retrying without streaming.")

    # Final attempt: non-streaming fallback
    # Removes the partial-JSON-in-deltas failure mode entirely
    kwargs_no_stream = {k: v for k, v in kwargs.items() if k != 'stream'}
    return client.messages.create(**kwargs_no_stream)
```

**The core rule for streaming in agent loops:** If your agent loop uses streaming, the stream completion handler must call `stream.get_final_message()` — not manually reconstruct content from deltas. Manual reconstruction of tool call arguments from streaming deltas is fragile. `get_final_message()` gives you a complete, validated response object.

**When not to stream in agent loops:** If your agent is executing tool calls in a tight loop and latency-to-first-token doesn't matter (tool execution will dominate), non-streaming is simpler and eliminates the interruption failure mode. Stream for user-facing output; don't stream for internal agent reasoning.

---

## Pattern 5 — Tool Result Truncation

Prevents context window creep from large tool outputs. Apply in every tool result handler.

```python
def truncate_tool_result(result: str, max_tokens: int = 1000) -> str:
    """
    Truncate tool results before returning to agent.
    Do this before appending to messages — not after.
    ~4 chars per token is a conservative estimate.
    """
    max_chars = max_tokens * 4
    if len(result) > max_chars:
        truncated = result[:max_chars]
        return (
            truncated
            + f"\n\n[Output truncated at ~{max_tokens} tokens. "
            f"Request a specific section if you need more.]"
        )
    return result


def wrap_tool_result(tool_use_id: str, result: str, max_tokens: int = 1000) -> dict:
    """Build a truncated tool result block ready to append to messages."""
    return {
        "type": "tool_result",
        "tool_use_id": tool_use_id,
        "content": truncate_tool_result(result, max_tokens)
    }
```

---

*Back: [Error Types](error-types.md) | [Stop Reasons](stop-reasons.md)*
