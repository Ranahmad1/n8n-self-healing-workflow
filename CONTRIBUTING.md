# Contributing to n8n Self-Healing Workflow

Thank you for your interest in contributing! Every improvement — new error handlers, logging integrations, or documentation fixes — makes this tool more useful for the whole n8n community.

---

## How to Contribute

### 1. Fork & Clone

```bash
git clone https://github.com/YOUR_USERNAME/n8n-self-healing-workflow.git
cd n8n-self-healing-workflow
```

### 2. Create a Branch

```bash
git checkout -b feature/add-graphql-error-handler
# or
git checkout -b fix/rate-limit-jitter-bug
# or
git checkout -b docs/add-postgres-logging-example
```

### 3. Make Your Changes

See the sections below for what to change depending on the type of contribution.

### 4. Open a Pull Request

Push your branch and open a PR against `main`. Include:
- What error type / feature you added
- Why this recovery strategy is the right one
- How you tested it (which `httpstat.us` URL or real API)

---

## Adding a New Error Type

### Step 1 — Root Cause Analyzer

Open `workflow/self_healing_workflow.json`, find the `Root Cause Analyzer` node (id: `node-root-cause`), and add your classification in the `jsCode` field:

```javascript
} else if (code === 503 || msg.includes('service unavailable')) {
  errorType = 'SERVICE_UNAVAILABLE';
  rootCause = 'Service temporarily unavailable — likely overloaded or restarting';
  riskLevel = 'LOW';
}
```

**Risk Level Guide:**
- `LOW` — safe to auto-recover (rate limits, timeouts, server errors)
- `MEDIUM` — auto-recover with caution (bad request formats)
- `HIGH` — always escalate to human (auth failures, data deletion risks)

### Step 2 — Strategy Generator

Find the `Strategy Generator` node and add a matching `case`:

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

### Step 3 — Update Error Catalog

Add a row to `docs/ERROR_CATALOG.md`.

### Step 4 — Test It

Use `https://httpstat.us/503` (or equivalent) to verify your handler works end-to-end.

---

## What We Need

**New Error Handlers**
- Database connection errors (Postgres, MySQL, MongoDB)
- GraphQL errors (`errors` array in 200 response)
- OAuth token expiry (refresh and retry)
- File system errors
- Queue/message broker errors

**Logging Integrations**
- Datadog
- Splunk
- Grafana Loki
- Google Sheets

**Notification Channels**
- Microsoft Teams
- Discord
- PagerDuty
- Telegram

**Test Workflows**
- One workflow per error type that demonstrates the full recovery path

---

## Code Style

- All Code nodes must have comments explaining the logic
- Use `console.log()` for debug output (visible in n8n execution logs)
- Every new error type needs `errorType`, `rootCause`, and `riskLevel`
- Every new strategy needs `type`, `can_auto_recover`, `retry_after_ms`, `action`, and `parameters`
- Keep the strategy object shape consistent — the Retry Engine and Logger depend on it

---

## Questions?

Open an issue or reach out:

- GitHub: [@Ranahmad1](https://github.com/Ranahmad1)
- Email: ahmadaslam0904@gmail.com

---

*Built with ❤️ by [Rana Ahmad](https://github.com/Ranahmad1) — Full Stack Engineer @ MADigital.pk*
