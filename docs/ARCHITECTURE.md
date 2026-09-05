# 🏗️ Architecture — Self-Healing n8n Workflow

> Deep dive into how every component works. Built by Rana Ahmad.

---

## Overview

The self-healing workflow is a **linear recovery pipeline** that activates whenever any upstream node fails. It runs in the same n8n execution context using Code nodes for logic and native n8n nodes for timing and routing.

The core principle: **detect → classify → decide → act → log**.

---

## Node-by-Node Breakdown

### 1. HTTP Request Node (protected node)
The primary node doing real work. `continueOnFail: true` ensures that when it fails, the error is passed as data to the next node rather than halting execution entirely.

### 2. Error Check (IF Node)
Examines output — if an error is present or status code >= 400, routes to the recovery pipeline. Otherwise routes to the success path.

### 3. Error Catcher (Code Node)
Activated by the error path. Normalizes the raw n8n error into a standard context object:

```json
{
  "timestamp": "ISO string",
  "execution_id": "n8n execution ID",
  "workflow_id": "workflow ID",
  "failed_node": "node that failed",
  "error_message": "human readable error",
  "status_code": 429,
  "retry_count": 0,
  "original_input": {}
}
```

### 4. Root Cause Analyzer (Code Node)
Pattern-matches against `error_message` and `status_code` to classify into one of these types:

| Type | Trigger |
|---|---|
| `RATE_LIMIT` | HTTP 429 / "rate limit" in message |
| `SERVER_ERROR` | HTTP 5xx |
| `AUTH_ERROR` | HTTP 401 / 403 |
| `NOT_FOUND` | HTTP 404 |
| `TIMEOUT` | ETIMEDOUT / timeout / 408 |
| `CONNECTION_REFUSED` | ECONNREFUSED |
| `DNS_ERROR` | ENOTFOUND |
| `INVALID_DATA` | null / undefined / cannot read |
| `CLIENT_ERROR` | Other 4xx |
| `UNKNOWN` | Everything else |

Also assigns `risk_level`: `LOW`, `MEDIUM`, or `HIGH`.

### 5. Strategy Generator (Code Node)
For each error type, generates a specific strategy object:

```json
{
  "type": "EXPONENTIAL_BACKOFF",
  "can_auto_recover": true,
  "retry_after_ms": 2500,
  "max_retries": 3,
  "current_attempt": 2,
  "action": "Wait 2500ms then retry (exponential backoff, attempt 2/3)",
  "parameters": { "delay_ms": 2500, "reduce_concurrency": true }
}
```

### 6. Safety Validator (IF Node)
Routes based on two conditions (both must be true to auto-recover):
- `risk_level !== "HIGH"`
- `can_auto_recover === true`

If either fails → Human Escalation. Otherwise → Retry Engine.

### 7. Retry Engine (Code Node)
Increments `retry_count`, builds `retry_config` (URL, timeout, delay), and passes forward. Also handles the case where `can_auto_recover` is false at this stage.

### 8. Max Retry Check (IF Node)
If `retry_count <= 3` → proceed to wait and retry. If over limit → Human Escalation.

### 9. Adaptive Wait (Wait Node)
Dynamic duration pulled from `recovery_strategy.retry_after_ms`. Pauses execution for the exact calculated delay, including jitter.

### 10. Retry HTTP Request (HTTP Request Node)
Re-fires the original HTTP call with parameters from `retry_config`. Has `continueOnFail: true` — if this also fails, it routes back through the logger.

### 11. Human Escalation (Code Node)
Builds a structured alert payload and logs it. In production, an HTTP Request node downstream sends this to `ALERT_WEBHOOK_URL` (Slack, PagerDuty, Teams, email).

### 12. Execution Logger (Code Node)
Writes a structured log entry for every event — whether it was a successful recovery, a failed retry, or an escalation. Includes full timing data.

---

## Data Flow Diagram

```
[Manual Trigger]
      |
[HTTP Request Node] --continueOnFail--> [Error Check]
                                              |
                              ┌───────────────┴──────────────┐
                           (error)                        (success)
                              |                               |
                      [Error Catcher]              [Success Handler]
                              |
                    [Root Cause Analyzer]
                              |
                     [Strategy Generator]
                              |
                     [Safety Validator]
                              |
              ┌───────────────┴───────────────┐
           (safe)                         (unsafe/HIGH risk)
              |                               |
       [Retry Engine]              [Human Escalation]
              |                               |
     [Max Retry Check]             [Execution Logger]
              |
    ┌─────────┴─────────┐
(within limit)     (exceeded)
    |                   |
[Adaptive Wait]  [Human Escalation]
    |
[Retry HTTP Request]
    |
[Execution Logger]
```

---

## Why `continueOnFail: true`?

Setting `continueOnFail: true` on the primary HTTP node makes n8n pass the error as structured data into the output instead of stopping execution. This is what lets the recovery pipeline see and act on the error.

Without this, n8n would halt the entire workflow on first error.

---

## Exponential Backoff Formula

```javascript
delay = Math.min(BASE_DELAY * Math.pow(MULTIPLIER, retryCount), MAX_DELAY) + jitter

// With defaults:
// Attempt 1: min(1000 * 2^0, 30000) + jitter = ~1000-1500ms
// Attempt 2: min(1000 * 2^1, 30000) + jitter = ~2000-2500ms
// Attempt 3: min(1000 * 2^2, 30000) + jitter = ~4000-4500ms
```

Jitter (0-500ms random) prevents thundering herd when multiple workflow instances fail simultaneously.

---

## Extending the Workflow

### Adding a New Error Type

1. Add classification in `Root Cause Analyzer`:
```javascript
} else if (msg.includes('your-pattern') || code === YOUR_CODE) {
  errorType = 'YOUR_ERROR_TYPE';
  rootCause = 'Human-readable description';
  riskLevel = 'LOW'; // or MEDIUM or HIGH
}
```

2. Add strategy in `Strategy Generator`:
```javascript
case 'YOUR_ERROR_TYPE':
  strategy = {
    type: 'YOUR_STRATEGY',
    can_auto_recover: true,
    retry_after_ms: 5000,
    max_retries: 3,
    current_attempt: retryCount + 1,
    action: 'What will happen',
    parameters: { your: 'params' }
  };
  break;
```

3. Add a row to `docs/ERROR_CATALOG.md`

---

*Built with ❤️ by [Rana Ahmad](https://github.com/Ranahmad1)*
