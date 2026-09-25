---
name: djini-researcher
description: >
  Use Djini.ai to run mobile security scans with native code analysis, manage
  BYOD and Corellium devices, and interact with sandbox consoles. Researcher
  plan: adds Ghidra/JNI analysis and BYOD. A2A streaming or direct REST. Bearer token auth.
---

# Djini Researcher Skill

Djini.AI is a mobile security platform. This skill covers the **Researcher plan** capabilities.

**Included:** Everything in AppSec, plus native code analysis (Ghidra/JNI), BYOD devices, higher console budget.
**Not included:** Deep scan (0-day), infrastructure scanning, project sharing, research labs.

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

# Start scan with native analysis
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Start a scan on $PROJECT_NAME with native scan enabled, use corellium\"}"

# Analyze native code after scan
curl -N "$DJINI_CONSOLE_URL/api/a2a/$PROJECT_NAME/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "List native libraries and analyze any JNI functions that handle crypto", "mode": "query"}'

# Delegate complex task to Cline
curl -N "$DJINI_CONSOLE_URL/api/a2a/$PROJECT_NAME/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Reverse engineer the certificate pinning implementation", "mode": "task"}'
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
| `start_scan` | Start scan (supports `native_scan`) |
| `scan_status` | Check scan progress and component statuses |
| `get_findings` | Get consolidated security findings |
| `tavily_search` | Web search (if configured) |

> `start_scan` supports `native_scan=true`, `preferred_device_provider` (`"corellium"`, `"device_lab"`), and `whitelisted_domains`. Do **not** use `deep_scan` — requires Enterprise or higher.

**Project endpoint** (`/api/a2a/<project>/ask`) adds:

| Tool | Description |
|---|---|
| `sandbox_ssh` | Execute commands on sandbox container |
| `read_project_file` | Read files or list directories from the project |

---

## AI SAST — Source-Code-Only Scan (fast, no binary)

A fast, **white-box AI SAST** that scans source code directly — **no APK/IPA, no
decompile, no device, no dynamic phase**. It covers all **8 OWASP MASVS categories**
(STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, PRIVACY) plus a
cross-cutting attack-chain pass, mapping every finding to a MASVS category and MASWE id
with severity, `file:line` locations, evidence, and remediation. Typically finishes within
5 minutes.

**Availability:** every plan, **including Free** (first 10 scans free).

**Provide the source two ways** — a public git URL *or* a source zip — then trigger the
scan and poll. No device or Corellium/device-lab is needed.

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

---

## Direct REST API Reference

### Upload & Scan

```bash
# Upload APK/IPA
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@/path/to/app.apk"

# Start scan with native analysis
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/process" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"appContext": "Banking app", "nativeScan": true, "preferredDeviceProvider": "corellium"}'

# Poll status / get findings / list / stop — same as AppSec plan
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/status" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/findings" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans?page=1&per_page=20" -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$PROJECT_NAME/stop" -H "Authorization: Bearer $DJINI_API_KEY"
```

**Scan parameters:** `appContext` (string, 280 chars), `nativeScan` (bool), `testCredentials` (array of `{username, password}`), `whitelistedDomains` (array), `preferredDeviceProvider` (`"corellium"` | `"device_lab"`), `llmModel` (string, `provider:model`).

**Status progression:** `VERIFYING -> PENDING -> QUEUED -> IN_PROGRESS -> COMPLETED`

### Findings

```bash
# AppSec findings
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/static-tool-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Native code findings (Researcher+)
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/ai-powered-findings?page=1&per_page=50" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Triage / notes — same pattern as AppSec
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/appsec/ai-powered-findings/status" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"findingId": "<id>", "status": "true"}'
```

### Native Code Analysis

```bash
# List decompiled libraries/functions
curl -s "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/libraries-and-functions" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# LLM analysis of a function
curl -s -X POST "$DJINI_CONSOLE_URL/api/workspace/$PROJECT_NAME/native/llm-analysis" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"libraryName": "libnative.so", "functionName": "Java_com_app_decrypt"}'

# Reverse engineer via Cline
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
  -d '{"message": "Explain the JNI crypto implementation"}'

# Chat about specific finding
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
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/list?path=/data/project/results" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/cline/$PROJECT_NAME/files/content?path=/data/project/ghidra/exports/libnative.c" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

### Device Management

```bash
DEVICE_TYPE="corellium"  # or "device_lab" or "byod" (Researcher+)

curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/start" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/status" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/install-app" \
  -H "Authorization: Bearer $DJINI_API_KEY"

# Dynamic testing
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/instrumentation/start" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s -X POST "$DJINI_CONSOLE_URL/api/device/$PROJECT_NAME/$DEVICE_TYPE/fuzzing/start" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"libraryName": "libnative.so", "functionName": "Java_com_app_decrypt"}'
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
