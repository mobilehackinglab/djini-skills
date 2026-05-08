---
name: djini-appsec
description: >
  Use Djini.ai to run mobile application security scans, triage findings, and
  interact with sandbox consoles. AppSec plan: standard scans on Corellium and
  device lab devices. A2A streaming or direct REST. Bearer token auth.
---

# Djini AppSec Skill

Djini is a mobile security platform. This skill covers the **AppSec plan** capabilities.

**Included:** App upload & scan, appsec findings, Cline console, Corellium & device lab devices, web search, BYOK LLM.
**Not included:** Native code analysis, deep scan (0-day), infrastructure scanning, BYOD, project sharing, research labs.

## Authentication

```bash
# Required environment variables:
# DJINI_CONSOLE_URL  - https://test.djini.ai (or your instance)
# DJINI_API_KEY      - User API key (sk-...)

curl -s -H "Authorization: Bearer $DJINI_API_KEY" "$DJINI_CONSOLE_URL/api/..."
```

OpenAPI spec: `$DJINI_CONSOLE_URL/openapi/openapi.json`

---

## A2A (Agent-to-Agent) — Primary Interface

SSE streaming endpoints. The Djini agent orchestrates tools internally — you send a message, it streams back results.

### Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/a2a/ask` | General — scans, general queries (no project needed) |
| `POST` | `/api/a2a/<project>/ask` | Project — sandbox SSH + file access |
| `GET` | `/api/a2a/<project>/status` | Check sandbox readiness |

### Quick Start

```bash
# Upload an app, then manage via A2A
RESULT=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@./app.apk")
PROJECT_NAME=$(echo "$RESULT" | jq -r .projectName)

# Start scan
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Start a scan on $PROJECT_NAME using corellium as device provider\"}"

# Check status
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Check scan status for $PROJECT_NAME\"}"

# Get findings
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Get findings for $PROJECT_NAME and summarize critical ones\"}"

# Project endpoint: query sandbox
curl -N "$DJINI_CONSOLE_URL/api/a2a/$PROJECT_NAME/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "What permissions does this app request?", "mode": "query"}'
```

### Request Body

| Field | Type | Default | Description |
|---|---|---|---|
| `message` | string | *(required)* | Your message or task |
| `mode` | string | `"query"` | `"query"` (agent) or `"task"` (Cline proxy) — project endpoint only |
| `session_id` | string | auto | For multi-turn conversations |

### SSE Response Events

| Event | Data | Description |
|---|---|---|
| `token` | `{"token": "..."}` | Incremental text |
| `tool_end` | `{"tool": "...", "output": "..."}` | Tool result |
| `answer` | `{"text": "full answer"}` | Complete answer |
| `error` | `{"error": "..."}` | Error |
| `end` | `{}` | Stream finished |

### Agent Tools

**General endpoint** (`/api/a2a/ask`):

| Tool | Description |
|---|---|
| `list_scans` | List scan projects with status, platform, severity counts |
| `upload_app` | Upload APK/IPA from server-side path |
| `start_scan` | Start scan on a project |
| `scan_status` | Check scan progress and component statuses |
| `get_findings` | Get consolidated security findings |
| `tavily_search` | Web search (if configured) |

> `start_scan` supports `preferred_device_provider` (`"corellium"` or `"device_lab"`) and `whitelisted_domains`. Do **not** use `deep_scan` or `native_scan` — these require higher plans.

**Project endpoint** (`/api/a2a/<project>/ask`) adds:

| Tool | Description |
|---|---|
| `sandbox_ssh` | Execute commands on sandbox container |
| `read_project_file` | Read files or list directories from the project |

---

## Direct REST API Reference

### Upload & Scan

```bash
# Upload APK/IPA
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@/path/to/app.apk"

# Start scan (no deepScan or nativeScan on this plan)
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/process" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"appContext": "Banking app", "preferredDeviceProvider": "corellium"}'

# Poll status
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/status" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Get findings
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/findings" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# List scans
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans?page=1&per_page=20" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Stop scan
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/stop" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

**Scan parameters:** `appContext` (string, 280 chars), `testCredentials` (array of `{username, password}`), `whitelistedDomains` (array), `preferredDeviceProvider` (`"corellium"` | `"device_lab"`), `llmModel` (string, `provider:model`).

**Status progression:** `VERIFYING -> PENDING -> QUEUED -> IN_PROGRESS -> COMPLETED`

### Findings

```bash
# AppSec findings
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/static-tool-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Triage
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings/status" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"findingId": "<id>", "status": "true"}'

# Add note
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/findings/appsec/<finding_id>/notes" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "Confirmed exploitable"}'
```

**Query params:** `triage=true|false|not_set|all`, `name_query=...`, `description_query=...`

### Chat

```bash
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/chat/$PROJECT_NAME" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain the impact of this hardcoded API key"}'
```

### Cline Sandbox

```bash
# Start / stop
curl -s -X POST "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/start" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" -d '{}'
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/status" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/stop" -H "Authorization: Bearer $DJINI_API_KEY"

# File operations
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/list?path=/data/project/results" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/content?path=/data/project/results/findings.sarif" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

### Device Management

```bash
DEVICE_TYPE="corellium"  # or "device_lab" (no BYOD on this plan)

curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/start" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/status" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/install-app" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/stop" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

### Project Management

```bash
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans?page=1&per_page=20" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X DELETE "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/$PROJECT_NAME/configuration" -H "Authorization: Bearer $DJINI_API_KEY"
```

### User Settings & BYOK

```bash
curl -s "$DJINI_CONSOLE_URL/api/user-settings" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/user-settings/user-api-key" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X PUT "$DJINI_CONSOLE_URL/api/user-settings/byok" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"providers": [{"provider": "anthropic", "key": "sk-ant-...", "model": "claude-sonnet-4-6"}]}'
```

**LLM providers:** `anthropic`, `openai`, `google`, `dashscope`, `moonshot`, `openrouter`, `openai_compatible`.

---

## Environment Variables

| Variable | Description |
|---|---|
| `DJINI_CONSOLE_URL` | Base URL (e.g. `https://test.djini.ai`) |
| `DJINI_API_KEY` | Bearer token (`sk-...`) |

## Error Reference

| Status | Meaning |
|---|---|
| `401` | Missing or invalid token |
| `403` | Insufficient role or feature not in plan |
| `404` | Resource not found |
| `409` | Conflict (container running, scan terminal) |
| `422` | Validation error |
| `500` | Internal error |
