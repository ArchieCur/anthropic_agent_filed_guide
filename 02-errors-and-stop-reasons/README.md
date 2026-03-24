# Section 02 — Errors, Stop Reasons & Recovery

*Triage: if you're here, something stopped or failed. Start with the stop reason or error code.*

---

## In This Section

- [Error Types — All Six with Recovery Strategies](error-types.md)
- [Stop Reasons Decoded for Agent Loops](stop-reasons.md)
- [Recovery Patterns (copy-paste)](recovery-patterns.md)

---

## Quick Reference

| Stop Reason | Means | Action |
|---|---|---|
| `end_turn` | Model finished | Normal completion |
| `pause_turn` | Yielding for tool use | Execute tools, continue turn |
| `max_tokens` | Token limit hit | Check headroom budget, handle partial output |
| `stop_sequence` | Hit a stop sequence | Normal or configured stop |

| Error | Means | Retryable? |
|---|---|---|
| 400 | Bad request | No — fix the request |
| 404 | Not found | No — check model ID |
| 413 | Context too large | No — reduce context |
| 429 | Rate limited | Yes — back off |
| 500 | Server error | Yes — transient |
| 529 | Overloaded | Yes — longer backoff |

---

