---
name: djini-enterprise
description: >
  Use Djini.ai for full mobile security scanning including deep scan (0-day),
  native analysis, infrastructure scanning, and project sharing. Enterprise plan:
  all scan features, unlimited console. A2A streaming or direct REST. Bearer token auth.
---

# Djini Enterprise Skill

Djini.AI is a mobile security platform. This skill covers the **Enterprise plan** capabilities.

**Included:** Everything in Researcher, plus deep scan (0-day research), infrastructure scanning (DAST), project sharing, unlimited console budget.
**Not included:** Research labs (requires Admin/MHL role).

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
# Upload, scan with all features
RESULT=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@./app.apk")
PROJECT_NAME=$(echo "$RESULT" | jq -r .projectName)

# Full scan: deep + native + infra
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Start a full scan on $PROJECT_NAME with deep scan and native scan enabled, whitelist api.example.com for infra scanning, use corellium\"}"

# Check progress
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Check scan status for $PROJECT_NAME, include all component statuses\"}"

# Get all findings (appsec + native + infra)
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Get all findings for $PROJECT_NAME, group by severity\"}"

# Deep analysis in sandbox
curl -N "$DJINI_CONSOLE_URL/api/a2a/$PROJECT_NAME/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Develop a PoC for the most critical vulnerability found", "mode": "task"}'
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
| `start_scan` | Start scan with `deep_scan`, `native_scan`, `whitelisted_domains` |
| `scan_status` | Check scan progress and component statuses |
| `get_findings` | Get consolidated security findings |
| `tavily_search` | Web search (if configured) |

> `start_scan` supports all options: `deep_scan=true`, `native_scan=true`, `preferred_device_provider`, `whitelisted_domains` (for infra/DAST scanning).

**Project endpoint** (`/api/a2a/<project>/ask`) adds:

| Tool | Description |
|---|---|
| `sandbox_ssh` | Execute commands on sandbox container |
| `read_project_file` | Read files or list directories from the project |

---

## AI SAST — Source-Only Scan (fast, no binary)

A fast, **white-box AI SAST** that scans source code directly — **no APK/IPA, no
decompile, no device, no dynamic phase**. It covers all **8 OWASP MASVS categories**
(STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, PRIVACY) plus a
cross-cutting attack-chain pass, mapping every finding to a MASVS category and MASWE id
with severity, `file:line` locations, evidence, and remediation. Typically finishes within
5 minutes.

**Availability:** every plan, **including Free** (first 10 scans free).

**Provide the source two ways** — a public git URL *or* a source zip — then trigger the
scan and poll. No device or Corellium/device-lab is needed.

> **Required first — bring your own model (BYOK).** The AI source scan is BYOK-only: it
> runs on **your** OpenAI-compatible model, never a djini-hosted one. Configure it once
> (step 0); if it's missing, the trigger in step 2 returns `400 {"code": "byok_required"}`.

### 0. Configure your OpenAI-compatible model (once)

```bash
curl -s -X PUT "$DJINI_CONSOLE_URL/api/user-settings/byok" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"providers":[{"provider":"openai_compatible",
        "baseUrl":"https://openrouter.ai/api/v1",
        "key":"sk-or-...",
        "model":"qwen/qwen3.8-flash"}]}'
```

- `provider` must be `openai_compatible`; `baseUrl`, `key` and `model` are all required
  (for a keyless local/self-hosted endpoint send `"key":"not-needed"`).
- Stored per user — do it once. Verify with `GET /api/user-settings` and look for the
  `openai_compatible` entry under `byokProviders`. The user can also set this in the UI
  under **Settings → BYOK**, or inline in the **Scan Source Code** dialog.

### 1a. From a public git repository

```bash
RESULT=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/clone-source" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"gitRepositoryUrl": "https://github.com/bitwarden/android.git"}')
PROJECT_NAME=$(echo "$RESULT" | jq -r .projectName)
```

- `gitRepositoryUrl` (**required**) — a **public** HTTPS git URL. Private repos aren't
  supported yet and return `400` with a clear message. Blobs > 10 MB are skipped on clone.

### 1b. From a source zip

```bash
RESULT=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/upload-source" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@./my-app-source.zip")
PROJECT_NAME=$(echo "$RESULT" | jq -r .projectName)
```

- `file` (**required**) — a `.zip` of the source tree. Files > 10 MB inside the zip are
  skipped. Much faster than `/upload` (no decompile that can time out the gateway).

Both return the same body — platform (Android/iOS) and package are auto-detected from the
source:

```json
{ "success": true, "projectName": "...", "platform": "Android",
  "packageName": "com.example.app", "appName": "...", "appVersion": "..." }
```

### 2. Trigger the scan

```bash
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/source-scan" \
  -H "Authorization: Bearer $DJINI_API_KEY"
# -> {"success": true, "message": "AI source scan started."}
# 409 if a scan is already in progress for this project.
```

### 3. Poll status

```bash
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/status" \
  -H "Authorization: Bearer $DJINI_API_KEY" | jq '{status, componentStatuses}'
```

Watch `componentStatuses["AI Source Scan"]`:
`In Progress` -> `Scanning <done>/<total> (<n> files)` -> `Completed` (or `Error`).
Top-level `status` moves `PENDING -> IN_PROGRESS -> COMPLETED`.

### 4. Get findings

```bash
# JSON (severity counts, MASVS/MASWE mapping, locations, remediation)
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/findings" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# SARIF 2.1.0 — GitHub Code Scanning compatible (upload-sarif)
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/sarif" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

Stop a running scan with `POST /api/dashboard/scans/$PROJECT_NAME/stop`.

**Errors:** `400 {"code":"byok_required"}` means no OpenAI-compatible model is configured — do step 0 first. `409` means a scan is already running for that project.

---

## Direct REST API Reference

### Upload & Scan

```bash
# Upload APK/IPA
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@/path/to/app.apk"

# Start full scan
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/process" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"appContext": "Banking app", "deepScan": true, "nativeScan": true, "whitelistedDomains": ["api.example.com"], "preferredDeviceProvider": "corellium"}'

# Trigger deep scan on already-completed scan
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/deep-scan" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Poll status / get findings / list / stop
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/status" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/findings" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans?page=1&per_page=20" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/stop" -H "Authorization: Bearer $DJINI_API_KEY"
```

**Scan parameters:** `appContext` (string, 280 chars), `deepScan` (bool — costs 1 extra credit), `nativeScan` (bool), `testCredentials` (array of `{username, password}`), `whitelistedDomains` (array — enables infra/DAST scanning), `preferredDeviceProvider` (`"corellium"` | `"device_lab"`), `llmModel` (string, `provider:model`).

**Status progression:** `VERIFYING -> PENDING -> QUEUED -> IN_PROGRESS -> COMPLETED`

### Findings

```bash
# AppSec findings
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/static-tool-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Native code findings
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/ai-powered-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Infrastructure / DAST findings (Enterprise+)
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/infra/findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Triage / notes
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings/status" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"findingId": "<id>", "status": "true"}'
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/findings/<finding_type>/<finding_id>/notes" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "Confirmed exploitable"}'
```

**Finding types:** `appsec`, `native`, `infra`. **Query params:** `triage=true|false|not_set|all`, `name_query=...`, `description_query=...`

### Native Code Analysis

```bash
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/libraries-and-functions" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/llm-analysis" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"libraryName": "libnative.so", "functionName": "Java_com_app_decrypt"}'
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/reverse-engineer" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"libraryName": "libnative.so", "functionName": "Java_com_app_decrypt"}'
```

### Chat

```bash
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/chat/$PROJECT_NAME" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Compare the infra findings with the appsec findings for this app"}'
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/chat/finding" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"findingId": "<id>", "message": "Show the vulnerable code path"}'
```

### Cline Sandbox

```bash
curl -s -X POST "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/start" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" -d '{}'
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/status" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/stop" -H "Authorization: Bearer $DJINI_API_KEY"

# File operations
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/list?path=/data/project/results" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/content?path=/data/project/results/findings.sarif" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X PUT "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/content" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"path": "/data/project/poc/exploit.py", "content": "#!/usr/bin/env python3\n..."}'
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/search?q=password&path=/data/project" -H "Authorization: Bearer $DJINI_API_KEY"
```

### Device Management

```bash
DEVICE_TYPE="corellium"  # or "device_lab" or "byod"

curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/start" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/status" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/install-app" -H "Authorization: Bearer $DJINI_API_KEY"

# Dynamic testing
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/instrumentation/start" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/fuzzing/start" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"libraryName": "libnative.so", "functionName": "Java_com_app_decrypt"}'
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/poc/start" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"findingId": 123}'
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/stop" -H "Authorization: Bearer $DJINI_API_KEY"
```

### Project Management

```bash
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans?page=1&per_page=20" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X DELETE "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/projects/clone" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"projectName": "existing-project"}'
curl -s "$DJINI_CONSOLE_URL/api/dashboard/$PROJECT_NAME/configuration" -H "Authorization: Bearer $DJINI_API_KEY"
```

### User Settings & BYOK

```bash
curl -s "$DJINI_CONSOLE_URL/api/user-settings" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/user-settings/user-api-key" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X PUT "$DJINI_CONSOLE_URL/api/user-settings/byok" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
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
