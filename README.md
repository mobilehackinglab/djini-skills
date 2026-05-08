# Djini Skills

[Djini](https://djini.ai) is a mobile security platform with AI-powered scanning, sandbox consoles, virtual devices, and exploit research labs. These skills let external agents (Claude Code, OpenCode, etc.) interact with Djini via A2A streaming endpoints or direct REST APIs.

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
| AppSec findings | yes | yes | yes |
| Cline console | yes ($100) | yes ($150) | yes (unlimited) |
| Corellium devices | yes | yes | yes |
| Device lab | yes | yes | yes |
| BYOD | - | yes | yes |
| Native code analysis | - | yes | yes |
| Deep scan (0-day) | - | - | yes |
| Infra scanning (DAST) | - | - | yes |
| Project sharing | - | - | yes |

## Setup

1. Get an API key from your Djini instance: **Settings > API Key > Generate**

2. Set environment variables:
   ```bash
   export DJINI_CONSOLE_URL="https://your-instance.djini.ai"
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

## Skill Structure

```
djini-appsec/SKILL.md      - Standard mobile app security scanning
djini-researcher/SKILL.md   - + Native code analysis, BYOD
djini-enterprise/SKILL.md   - + Deep scan, infra scanning, project sharing
```

## Links

- [Djini Platform](https://djini.ai)
- [API Documentation](https://app.djini.ai/openapi/openapi.json)
