# ⚙️ Setup Guide — Self-Healing n8n Workflow

> Step-by-step guide to get the self-healing workflow running in your n8n instance.
> Built by [Rana Ahmad](https://github.com/Ranahmad1)

---

## Prerequisites

- n8n version **0.214.0 or later** (self-hosted or n8n Cloud)
- Node.js **18+** (for self-hosted)
- Basic familiarity with n8n workflows

---

## Step 1 — Import the Workflow

1. Open your n8n instance in the browser
2. Click **Workflows** in the left sidebar
3. Click **Add Workflow** → **Import from file**
4. Select `workflow/self_healing_workflow.json` from this repo
5. Click **Import**

You should see all nodes appear on the canvas.

---

## Step 2 — Configure Environment Variables

### Self-Hosted n8n

Add to your `.env` file (or `docker-compose.yml` environment section):

```env
# Required for human escalation alerts
ALERT_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# Optional — override defaults
MAX_RETRY_ATTEMPTS=3
BASE_RETRY_DELAY_MS=1000
BACKOFF_MULTIPLIER=2
MAX_DELAY_MS=30000
```

### n8n Cloud

Go to **Settings** → **Environment Variables** and add the same keys.

---

## Step 3 — Connect Your Alert Webhook (For Human Escalation)

After the **Human Escalation** node, add an **HTTP Request** node:

```
Method:       POST
URL:          {{ $env.ALERT_WEBHOOK_URL }}
Content-Type: application/json
Body:         {{ JSON.stringify($json.escalation) }}
```

### Slack Payload Format

If using Slack, format the body like this:

```json
{
  "text": "{{ $json.escalation.message }}",
  "username": "n8n Self-Healer",
  "icon_emoji": ":warning:"
}
```

### PagerDuty

```json
{
  "routing_key": "YOUR_INTEGRATION_KEY",
  "event_action": "trigger",
  "payload": {
    "summary": "{{ $json.escalation.alert_type }}",
    "severity": "critical",
    "source": "{{ $json.escalation.workflow }}"
  }
}
```

---

## Step 4 — Protect Your Own Nodes

To use self-healing on any node in your workflows:

### Option A — continueOnFail (Recommended)

1. Click the node you want to protect
2. Go to **Settings** tab
3. Enable **Continue On Fail**
4. Connect the node's output → **Error Check** node

### Option B — Error Workflow

1. In Workflow Settings → **Error Workflow**
2. Set it to this self-healing workflow
3. n8n will automatically trigger it on any uncaught failure

---

## Step 5 — Test Each Error Type

Use [httpstat.us](https://httpstat.us) to simulate specific HTTP errors without touching real APIs:

| Test URL | Error Simulated |
|---|---|
| `https://httpstat.us/429` | Rate limit (429) |
| `https://httpstat.us/500` | Server error (500) |
| `https://httpstat.us/503` | Service unavailable |
| `https://httpstat.us/401` | Auth error (401) |
| `https://httpstat.us/403` | Forbidden (403) |
| `https://httpstat.us/404` | Not found (404) |
| `https://httpstat.us/408` | Timeout (408) |

**To test:**
1. Set the HTTP Request Node URL to one of the above
2. Click **Execute Workflow** (Manual Trigger)
3. Watch the execution path in the n8n canvas
4. Check the **Execution Logger** output for the log entry

---

## Step 6 — Enable Database Logging (Optional)

To persist recovery logs to a database, add a **Postgres** (or MySQL/SQLite) node after the Execution Logger:

```sql
INSERT INTO n8n_recovery_log (
  log_id, timestamp, workflow_name, failed_node,
  error_type, root_cause, risk_level,
  recovery_strategy, retry_attempt, delay_applied_ms,
  outcome, escalated_to_human, total_recovery_time_ms
) VALUES (
  '{{ $json.log_id }}',
  '{{ $json.timestamp }}',
  '{{ $json.workflow_name }}',
  '{{ $json.failed_node }}',
  '{{ $json.error_type }}',
  '{{ $json.root_cause }}',
  '{{ $json.risk_level }}',
  '{{ $json.recovery_strategy }}',
  {{ $json.retry_attempt }},
  {{ $json.delay_applied_ms }},
  '{{ $json.outcome }}',
  {{ $json.escalated_to_human }},
  {{ $json.total_recovery_time_ms }}
);
```

### Create the Table First

```sql
CREATE TABLE IF NOT EXISTS n8n_recovery_log (
  id SERIAL PRIMARY KEY,
  log_id VARCHAR(64) UNIQUE,
  timestamp TIMESTAMPTZ,
  workflow_name VARCHAR(255),
  failed_node VARCHAR(255),
  error_type VARCHAR(64),
  root_cause TEXT,
  risk_level VARCHAR(16),
  recovery_strategy VARCHAR(64),
  retry_attempt INT,
  delay_applied_ms INT,
  outcome VARCHAR(64),
  escalated_to_human BOOLEAN,
  total_recovery_time_ms INT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Step 7 — Production Checklist

Before going live:

- [ ] Imported `self_healing_workflow.json` successfully
- [ ] `ALERT_WEBHOOK_URL` set and tested (send a test message)
- [ ] All nodes you want protected have `continueOnFail: true`
- [ ] Tested with `httpstat.us/429` (rate limit path works)
- [ ] Tested with `httpstat.us/401` (human escalation fires)
- [ ] Execution Logger output verified in n8n UI
- [ ] (Optional) Database table created and logging works
- [ ] (Optional) Error Workflow set in production workflow settings

---

## Troubleshooting

**Q: The Error Check node always routes to success even on errors.**
A: Make sure `continueOnFail: true` is set on the HTTP Request Node. Without it, n8n stops execution before reaching Error Check.

**Q: Adaptive Wait node fails.**
A: The Wait node requires a Webhook URL to resume. In self-hosted n8n, ensure `WEBHOOK_URL` is set in your environment.

**Q: Human Escalation fires but no alert is received.**
A: Add an HTTP Request node after Human Escalation and point it at your `ALERT_WEBHOOK_URL`. The Escalation node only builds the payload — it doesn't send it by default.

**Q: I want to retry more than 3 times.**
A: In the Max Retry Check node, change `value2` from `3` to your desired max. Also update `MAX_RETRIES` constant in the Strategy Generator code.

---

*Built by [Rana Ahmad](https://github.com/Ranahmad1) — Full Stack Engineer @ MADigital.pk*
