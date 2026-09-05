# 📋 Error Catalog — All Handled Error Types

> Complete reference for every error type the self-healing workflow detects and handles.

---

## Error Type Reference

| Error Type | HTTP Code / Signal | Risk Level | Recovery Strategy | Auto-Recover? | Max Attempts |
|---|---|---|---|---|---|
| `RATE_LIMIT` | 429 / "rate limit" | LOW | Exponential backoff + jitter | ✅ Yes | 3 |
| `SERVER_ERROR` | 5xx | LOW | Wait + retry with jitter | ✅ Yes | 3 |
| `TIMEOUT` | ETIMEDOUT / 408 | LOW | Retry with 2x timeout | ✅ Yes | 3 |
| `CONNECTION_REFUSED` | ECONNREFUSED | LOW | Wait 30s + retry | ✅ Yes | 2 |
| `DNS_ERROR` | ENOTFOUND | LOW | Wait 5s + retry | ✅ Yes | 2 |
| `INVALID_DATA` | null/undefined error | LOW | Sanitize + use fallbacks | ✅ Yes | 1 |
| `NOT_FOUND` | 404 | LOW | Log and skip item | ✅ Yes | 1 |
| `CLIENT_ERROR` | Other 4xx | MEDIUM | Human escalation | ❌ No | - |
| `AUTH_ERROR` | 401 / 403 | HIGH | Human escalation | ❌ No | - |
| `UNKNOWN` | Catch-all | LOW | Log + human escalation | ❌ No | - |

---

## Detailed Strategies

### RATE_LIMIT — Exponential Backoff
**Trigger:** HTTP 429, message contains "rate limit" or "too many requests"

**Why it happens:** The API has a request quota (e.g. 100 req/min) and your workflow exceeded it.

**Recovery:**
1. Calculate delay: `BASE_DELAY * 2^retryCount + random_jitter`
2. Wait for the calculated duration
3. Retry the original request
4. If still failing after 3 attempts → escalate

**Delay schedule (with 1000ms base):**
- Attempt 1: ~1000–1500ms
- Attempt 2: ~2000–2500ms
- Attempt 3: ~4000–4500ms

---

### SERVER_ERROR — Retry with Delay
**Trigger:** HTTP 500, 502, 503, 504

**Why it happens:** The upstream server had a transient failure — overload, restart, deployment.

**Recovery:** Wait (exponential backoff) and retry. Most 5xx errors resolve within seconds.

---

### TIMEOUT — Retry with Increased Timeout
**Trigger:** ETIMEDOUT, ECONNABORTED, HTTP 408, message contains "timeout"

**Why it happens:** Network latency or a slow server response exceeded the timeout window.

**Recovery:** Retry with the timeout doubled (10s → 20s). If the server is just slow, this gives it more time to respond.

---

### CONNECTION_REFUSED — Long Wait + Retry
**Trigger:** ECONNREFUSED, "connection refused" in message

**Why it happens:** The target server process is down, restarting, or the port is not listening.

**Recovery:** Wait 30 seconds (longer than most service restarts) then retry. Max 2 attempts before escalating.

---

### DNS_ERROR — Short Wait + Retry
**Trigger:** ENOTFOUND, "dns" in message

**Why it happens:** The hostname cannot be resolved — DNS propagation lag or temporary DNS server issue.

**Recovery:** Wait 5 seconds then retry. If DNS still fails, escalate.

---

### INVALID_DATA — Sanitize + Fallback
**Trigger:** Error message contains "null", "undefined", "cannot read", "invalid"

**Why it happens:** The API returned unexpected data — missing fields, wrong types, empty response.

**Recovery:** Apply data sanitization (trim, type-cast) and inject fallback values for null/missing fields. No retry needed — the data issue is handled in-place.

---

### NOT_FOUND — Log and Skip
**Trigger:** HTTP 404

**Why it happens:** The specific resource no longer exists.

**Recovery:** Log the missing resource ID and skip this item. Continue processing the rest of the batch. Useful for bulk operations where one missing record shouldn't stop everything.

---

### AUTH_ERROR — Human Escalation (HIGH RISK)
**Trigger:** HTTP 401, 403

**Why it happens:** API key expired, wrong credentials, insufficient permissions.

**Recovery:** Automatic retry is not safe — re-sending with bad credentials could trigger lockouts. Always escalate to human to refresh credentials.

---

### CLIENT_ERROR — Human Escalation
**Trigger:** Other 4xx (400, 405, 406, 409, 410, 422, etc.)

**Why it happens:** Request format is wrong, required parameters missing, or a business logic conflict.

**Recovery:** Requires understanding the API contract. Escalate for human review.

---

## Adding New Error Types

To add support for a new error type:

1. **Open `workflow/self_healing_workflow.json`**
2. **Root Cause Analyzer node** — add classification logic:
```javascript
} else if (code === 503 || msg.includes('service unavailable')) {
  errorType = 'SERVICE_UNAVAILABLE';
  rootCause = 'Service temporarily unavailable';
  riskLevel = 'LOW';
}
```
3. **Strategy Generator node** — add recovery case:
```javascript
case 'SERVICE_UNAVAILABLE':
  strategy = {
    type: 'RETRY_WITH_DELAY',
    can_auto_recover: retryCount < 3,
    retry_after_ms: 15000,
    max_retries: 3,
    current_attempt: retryCount + 1,
    action: 'Wait 15s then retry — service may be restarting',
    parameters: { delay_ms: 15000 }
  };
  break;
```
4. **Add a row to this table** above
5. **Open a PR** with a clear description of the error type and why your strategy is the right one

---

*Built by [Rana Ahmad](https://github.com/Ranahmad1)*
