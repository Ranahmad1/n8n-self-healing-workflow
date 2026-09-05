# 🔧 n8n Self-Healing Workflow

> **Built by [Rana Ahmad](https://github.com/Ranahmad1) — Full Stack Engineer & AI Automation Enthusiast**

[![n8n](https://img.shields.io/badge/n8n-workflow-orange?style=flat-square&logo=n8n)](https://n8n.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Automation](https://img.shields.io/badge/AI-Self--Healing-green?style=flat-square)](#)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)
[![Stars](https://img.shields.io/github/stars/Ranahmad1/n8n-self-healing-workflow?style=flat-square)](https://github.com/Ranahmad1/n8n-self-healing-workflow/stargazers)

---

## 🚀 What Is This?

A production-grade **self-healing n8n workflow** that automatically:

1. **Captures** any node failure in real-time
2. **Identifies** the root cause (API error, rate limit, timeout, bad data, etc.)
3. **Generates** a targeted recovery strategy using AI logic
4. **Validates** the strategy for safety before applying it
5. **Retries** the failed operation with the fix applied
6. **Logs** every execution and recovery attempt with full detail
7. **Escalates** to a human if the proposed fix could be unsafe

No more silent failures. No more manual debugging at 2 AM. Your workflow heals itself.

---

## 🧠 Error Types Handled

| Error Type | Detection | Recovery Strategy |
|---|---|---|
| `HTTP 429` Rate Limit | Status code check | Exponential backoff + retry |
| `HTTP 5xx` Server Error | Status code check | Wait + retry with jitter |
| `HTTP 401/403` Auth Error | Status code check | Flag for human approval |
| `ETIMEDOUT` Timeout | Error message match | Increase timeout + retry |
| `ECONNREFUSED` | Error message match | Wait 30s + retry |
| Invalid/Null Data | Schema validation | Skip or use fallback value |
| `HTTP 404` Not Found | Status code check | Log and skip item |
| Unknown Error | Catch-all | Log + human escalation |

---

## 🗂️ Repository Structure

```
n8n-self-healing-workflow/
├── workflow/
│   └── self_healing_workflow.json        # Main n8n workflow (import this)
├── docs/
│   ├── ARCHITECTURE.md                   # How it works, deep dive
│   ├── ERROR_CATALOG.md                  # All error types & strategies
│   └── SETUP_GUIDE.md                    # Step-by-step setup
├── examples/
│   └── api_call_example.json             # Example: API call with healing
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## ⚡ Quick Start

### 1. Import the Workflow

```bash
# In n8n UI:
# Settings → Import Workflow → Upload workflow/self_healing_workflow.json
```

### 2. Configure Environment Variables

```env
ALERT_WEBHOOK_URL=https://hooks.slack.com/your-webhook-url
MAX_RETRY_ATTEMPTS=3
BASE_RETRY_DELAY_MS=1000
```

### 3. Wrap Any Node

Connect any node's **error output** to the `Error Catcher` node. The workflow handles everything else automatically.

---

## 🔄 Workflow Architecture

```
[Your Node] ──▶ [Success Path] ──▶ [...]
     │
     └─(error)─▶ [Error Catcher]
                      │
                      ▼
               [Root Cause Analyzer]
                      │
                      ▼
               [Strategy Generator]
                      │
                 ┌────┴────┐
                 ▼         ▼
           [Safe?]    [Unsafe → Human]
                 │
                 ▼
          [Apply & Retry]
                 │
            ┌───┴───┐
            ▼       ▼
         [Pass]  [Max Retries → Escalate]
```

---

## 📋 Recovery Log Schema

Every recovery event produces a structured log entry:

```json
{
  "timestamp": "2026-09-05T10:30:00Z",
  "workflow_id": "wf_abc123",
  "execution_id": "exec_xyz789",
  "failed_node": "HTTP Request - Fetch Users",
  "error_type": "RATE_LIMIT",
  "error_message": "429 Too Many Requests",
  "root_cause": "API rate limit exceeded (100 req/min)",
  "recovery_strategy": "exponential_backoff",
  "retry_attempt": 2,
  "delay_applied_ms": 4000,
  "outcome": "SUCCESS",
  "total_recovery_time_ms": 6200
}
```

---

## 🛡️ Safety First — Human Approval Gate

If the auto-generated fix involves:
- Changing authentication credentials
- Bypassing validation steps
- Any action flagged as `risk_level: HIGH`

The workflow **pauses** and sends an approval request (Slack/email/webhook) before proceeding. Nothing dangerous happens automatically.

---

## 🔧 Configuration Reference

| Variable | Default | Description |
|---|---|---|
| `MAX_RETRY_ATTEMPTS` | `3` | Max retries before escalation |
| `BASE_RETRY_DELAY_MS` | `1000` | Base delay for exponential backoff |
| `BACKOFF_MULTIPLIER` | `2` | Multiplier per retry attempt |
| `MAX_DELAY_MS` | `30000` | Cap for backoff delay |
| `ALERT_WEBHOOK_URL` | `""` | Webhook for human escalation |

---

## 🧪 Test with Simulated Errors

Use httpstat.us to test each error type without hitting real APIs:

```
https://httpstat.us/429  → Rate limit
https://httpstat.us/500  → Server error
https://httpstat.us/401  → Auth error
https://httpstat.us/404  → Not found
https://httpstat.us/408  → Timeout
```

---

## 📊 What Gets Logged

- ✅ Successful recoveries
- ❌ Failed recoveries (with full error chain)
- ⏱️ Time-to-recover per error type
- 🔁 Retry attempt history
- 👤 Human escalations and their outcomes

---

## 🤝 Contributing

Pull requests are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Want to add a new error handler? Open an issue with the error type and proposed recovery logic.

---

## 👤 Author

**Rana Ahmad**
- 🌐 [Portfolio](https://ranahmad1.github.io/rana-ahmad-portfolio/)
- 💼 [LinkedIn](https://www.linkedin.com/in/rana-ahmad-896004365/)
- 🐙 [GitHub](https://github.com/Ranahmad1)
- 📧 ahmadaslam0904@gmail.com

Full Stack Engineer @ MADigital.pk | Building FlexERP | BSCS @ University of Central Punjab

---

## 📄 License

MIT License — free to use, modify, and distribute. See [LICENSE](LICENSE) for details.

---

## ⭐ If This Helped You

Give it a ⭐ star — it helps others find this project and motivates me to build more open-source automation tools.

---

<!-- SEO Keywords -->
*n8n self-healing workflow · n8n error handling · n8n retry logic · n8n automation · n8n API error recovery · n8n rate limit handling · n8n timeout retry · exponential backoff n8n · n8n human approval gate · n8n execution log · no-code automation · low-code workflow · n8n tutorial · rana ahmad · pakistan developer*
