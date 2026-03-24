# Error Types — All Six with Recovery Strategies

Something is broken. Find your error code below and go.

---

## 400 — invalid_request_error

**Plain language:** Your request is malformed. The API rejected it before processing.

**Common agent causes:**
- `temperature` and `top_p` both set — breaking change in Claude 4, hard 400
- Prefilling assistant messages on Opus 4.6 — not supported, returns 400
- Tool schema invalid or malformed JSON
- Message roles out of order or empty content blocks
- `max_tokens` not set (required field)

**Recovery:** Fix the request. A 400 will not resolve with a retry — the request is structurally wrong.

**Agent-specific:** In multi-agent pipelines, a 400 on one agent often means a message construction error in the **orchestrator**, not the failing agent itself. The orchestrator built the message; check upstream. If you only test with Sonnet and have an Opus path, the prefill gotcha will pass local testing and surface here in production.

---

## 404 — not_found_error

**Plain language:** The resource you requested does not exist.

**Common agent causes:**
- Deprecated model string — `claude-3-5-sonnet-20240620` retired October 2025
- Wrong model string format
- Files API reference that expired or was never created

**Recovery:** Verify model strings against the current model list. Check Files API references for expiry.

**Agent-specific:** If this surfaces after a deployment, check for hardcoded model strings. Model retirement is the most common cause and is entirely silent until the 404 arrives.

---

## 413 — request_too_large

**Plain language:** Your request exceeded 32MB. The API rejected it at the gateway — it never reached the model.

**Common agent causes:**
- Context window creep — conversation history accumulated without pruning
- Large tool results returned in full without truncation
- Multiple large file attachments in a single request

**Recovery:** Reduce request size. Implement proactive context monitoring. Truncate tool results before returning them to the agent.

**Agent-specific:** This error arrives without warning. There is no gradual degradation before the 413 hits. Your agent has been growing silently — the 413 is the first signal. The correct fix is upstream: monitor context utilization before each request and prune at 80% capacity. See [recovery-patterns.md](recovery-patterns.md) for the monitoring pattern.

---

## 429 — rate_limit_error

**Plain language:** You have exceeded a rate limit.

**CRITICAL: There are two distinct 429 subtypes with different recovery profiles.** The HTTP status code is the same. The response body distinguishes them. A single retry handler for all 429s will mishandle one of them.

### Subtype 1 — Standard Rate Limit

Triggered when you exceed RPM, ITPM, or OTPM limits over a sustained period.

**Response body signature:**
```json
{
  "type": "error",
  "error": {
    "type": "rate_limit_error",
    "message": "Rate limit exceeded: requests per minute..."
  }
}
```
The message will name which limit was hit: requests per minute, input tokens per minute, or output tokens per minute.

**`retry-after` header:** Present. Contains the number of seconds to wait.

**Recovery:** Wait the duration in `retry-after`. Then retry. The limit has replenished.

### Subtype 2 — Acceleration Limit

Triggered by a sharp usage spike — too much traffic in too short a window, even if sustained rate is within limits. The token bucket algorithm detects the burst rate, not just the total.

**Response body signature:**
```json
{
  "type": "error",
  "error": {
    "type": "rate_limit_error",
    "message": "Rate limit exceeded: acceleration limit..."
  }
}
```

**`retry-after` header:** Present, but waiting that duration and firing the same burst again will immediately re-trigger the limit.

**Recovery:** Don't just wait and retry at the same concurrency. **Ramp traffic gradually.** Reduce parallelism, add jitter between requests, and increase gradually. A 5-10 second wait followed by sequential or low-concurrency retry is more effective than the full `retry-after` wait followed by the same burst.

**Agent-specific:** Agent teams that fan out in parallel are specifically exposed to Subtype 2. Five agents firing simultaneously creates a burst that trips acceleration limiting even if your sustained rate is well within limits. The fix is staggered agent startup, not just a retry handler.

---

## 500 — api_error

**Plain language:** Unexpected error on Anthropic's infrastructure. Not caused by your request.

**Common agent causes:** Transient infrastructure issue. More frequent on long-context requests.

**Observed threshold:** Transient 500s become noticeably more common at context windows above ~100k tokens. This is not documented explicitly but is consistent across long-running agent sessions. At 150k+ tokens, treat 500 as a normal part of the retry envelope, not an exception.

**Recovery:** These are retryable. Implement exponential backoff and retry up to 3 attempts. A 500 that persists across 3 retries with backoff is no longer transient — escalate or fail the run.

**Agent-specific:** Practitioners who treat 500 as fatal will abandon recoverable long-context runs. The retry is the entire fix for the 90% case. Losing a 150k-token context window because you didn't retry a transient 500 is an expensive mistake.

---

## 529 — overloaded_error

**Plain language:** Anthropic's API is temporarily overloaded across all users. Not caused by your account or usage patterns.

**Common agent causes:** High traffic periods. Unrelated to your system.

**Recovery:** Same pattern as 500 — exponential backoff and retry. Combine 500 and 529 in your error handler; they require identical treatment.

**Agent-specific:** If 529s are frequent and business-critical, investigate Anthropic's Priority Tier (documented in Service Tiers). Standard tier requests are deprioritized during overload events.

---

## Error Handler Reference

| Code | Retryable | Fix location | First action |
|------|-----------|--------------|--------------|
| 400 | No | Your request | Read error message, fix structure |
| 404 | No | Your config | Verify model string or resource ID |
| 413 | No | Your context management | Reduce request size, add monitoring |
| 429 | Yes | Your retry logic | Read response body for subtype, apply correct backoff |
| 500 | Yes | Your retry logic | Exponential backoff, up to 3 retries |
| 529 | Yes | Your retry logic | Same as 500 |

---

*Next: [Stop Reasons](stop-reasons.md) | [Recovery Patterns](recovery-patterns.md)*
