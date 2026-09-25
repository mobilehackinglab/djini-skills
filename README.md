# Djini Skills

[Djini.AI](https://djini.ai) is a mobile security platform with AI-powered scanning, sandbox consoles, virtual devices, and exploit research labs. These skills let external agents (Claude Code, OpenCode, etc.) interact with Djini via A2A streaming endpoints or direct REST APIs.

## ⚡ AI SAST — Source-Only Scan (fast, no binary)

A fast **white-box AI SAST** that scans source code directly — **no APK/IPA, no decompile, no device**. Point it at a **public git repo** or upload a **source zip**, and it audits all **8 OWASP MASVS categories** (STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, PRIVACY) plus a cross-cutting attack-chain pass, mapping findings to MASVS/MASWE with severity, `file:line`, evidence, and remediation. Results as JSON or SARIF (GitHub Code Scanning compatible), typically in ~1–3 minutes.

**Available on every plan — including Free (first 10 scans free).** See the `## AI SAST — Source-Only Scan` section in any `SKILL.md` for the full flow.

> **BYOK required:** the source scan runs on **your own OpenAI-compatible model**. Configure it once (step 0 below), or the trigger returns `400 {"code":"byok_required"}`.

```bash
# 0. Configure your model once (BYOK — OpenAI-compatible)
curl -s -X PUT "$DJINI_CONSOLE_URL/api/user-settings/byok" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"providers":[{"provider":"openai_compatible","baseUrl":"https://openrouter.ai/api/v1","key":"sk-or-...","model":"qwen/qwen3.8-flash"}]}'

# Scan a public repo end-to-end
P=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/clone-source" \
  -H "Authorization: Bearer $DJINI_API_KEY" -H "Content-Type: application/json" \
  -d '{"gitRepositoryUrl": "https://github.com/bitwarden/android.git"}' | jq -r .projectName)
curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/scans/$P/source-scan" \
  -H "Authorization: Bearer $DJINI_API_KEY"
curl -s "$DJINI_CONSOLE_URL/api/dashboard/scans/$P/findings" \
  -H "Authorization: Bearer $DJINI_API_KEY"
```

## Installation

```bash
npx skills add mobilehackinglab/djini-skills
```

Then keep only the skill matching your subscription plan and remove the others.

## Choose Your Plan

| | AppSec | Researcher | Enterprise |
|---|:---:|:---:|:---:|
| **Skill** | `djini-appsec` | `djini-researcher` | `djini-enterprise` |
| App upload & scan | yes | yes | yes |
| AppSec AI SAST | yes | yes | yes |
| AppSec AI DAST | yes | yes | yes |
| Infra scanning (DAST) | yes | yes | yes |
| Cline console | yes | yes | yes (unlimited) |
| Corellium devices | yes | yes | yes |
| Device lab | yes | yes | yes |
| BYOD | - | yes | yes |
| Native code analysis | - | yes | yes |
| Deep scan (0-day) | - | - | yes |
| Project sharing | - | - | yes |

## Setup

1. Get an API key from your Djini instance: **Settings > API Key > Generate**

2. Set environment variables:
   ```bash
   export DJINI_CONSOLE_URL="https://app.djini.ai"
   export DJINI_API_KEY="sk-your-api-key"
   ```

3. Verify access:
   ```bash
   curl -s -H "Authorization: Bearer $DJINI_API_KEY" "$DJINI_CONSOLE_URL/api/user-settings" | jq .
   ```

## Quick Example

```bash
# Upload an app
RESULT=$(curl -s -X POST "$DJINI_CONSOLE_URL/api/dashboard/upload" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -F "file=@./app.apk")
PROJECT_NAME=$(echo "$RESULT" | jq -r .projectName)

# Talk to the Djini agent via A2A
curl -N "$DJINI_CONSOLE_URL/api/a2a/ask" \
  -H "Authorization: Bearer $DJINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"Start a scan on $PROJECT_NAME and let me know when it's done\"}"
```

## How It Works

Each skill configures your agent with the endpoints, tools, and workflows available on your plan. The primary interface is **A2A (Agent-to-Agent)** — SSE streaming endpoints where you send a message and Djini's server-side agent handles tool orchestration:

- `POST /api/a2a/ask` — General queries, scan management, lab management
- `POST /api/a2a/<project>/ask` — Project-specific with sandbox SSH + file access
- Direct REST API is available for granular control when needed

### A2A Protocol Compatibility

Djini implements the [A2A protocol](https://a2a-protocol.org/) for agent discovery and communication:

- **Agent Card** at `/.well-known/agent.json` — standard discovery endpoint for capabilities, skills, and auth
- **Extended Agent Card** at `/api/a2a/agent-card` — authenticated, includes plan-specific skills
- **SSE streaming** with structured events (`token`, `tool_end`, `answer`, `error`, `end`)
- **Bearer token auth** as declared in the Agent Card security schemes

## Skill Structure

```
djini-appsec/SKILL.md      - Standard mobile app security scanning
djini-researcher/SKILL.md   - + Native code analysis, BYOD
djini-enterprise/SKILL.md   - + Deep scan, infra scanning, project sharing
```

## Links

- [Djini Platform](https://djini.ai)
- [API Documentation](https://app.djini.ai/openapi/openapi.json)
