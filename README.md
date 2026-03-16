# Won-to-Onboard — Agentic Workflow Automation

> **From a closed deal in Pipedrive to a fully onboarded project — in seconds, not hours.**

[![n8n](https://img.shields.io/badge/orchestration-n8n-orange)](https://n8n.io)
[![OpenRouter](https://img.shields.io/badge/AI-OpenRouter%20%7C%20Gemini%202.0%20Flash-blue)](https://openrouter.ai)
[![Microsoft Graph](https://img.shields.io/badge/integrations-Microsoft%20Graph%20API-0078D4)](https://developer.microsoft.com/graph)
[![ClickUp](https://img.shields.io/badge/project%20management-ClickUp-7B68EE)](https://clickup.com)

The moment a deal is marked "won" in Pipedrive, a single AI agent analyses the deal, classifies the project, and generates all content needed to onboard the client: an AI-written kick-off email via Outlook, a SharePoint folder move, a ClickUp project with AI-suggested tasks, and a Teams channel with a welcome message and the full project team added automatically.

---

## Architecture

```
Pipedrive deal → "won"
        │
        ▼
IF: status === "won" AND previous !== "won"   ← idempotency guard
        │
        ▼
Code: build prompt → OpenRouter (Gemini 2.0 Flash) → parse JSON
        │
        ▼
IF: risk_level === "alto" → [future: human approval gate]
        │
        ├──→ Outlook        — AI-generated kick-off email
        ├──→ SharePoint     — folder moved to "Projetos Ativos"
        ├──→ ClickUp        — project + 5–8 AI-suggested tasks
        └──→ Teams          — channel + AI welcome message + 4 members
```

**One LLM call. Four actions. ~23 seconds end-to-end.**

---

## Tech Stack

### Orchestration: n8n (self-hosted)

n8n was chosen over Make, Zapier, and String.com for three reasons: deal data never leaves your infrastructure, real JavaScript execution in nodes, and workflows export as portable JSON — auditable and version-controlled.

### AI: OpenRouter + Gemini 2.0 Flash

OpenRouter provides an OpenAI-compatible abstraction over 200+ models. The model is a single `.env` variable — switching to GPT-4o or Claude in production requires zero code changes. Gemini 2.0 Flash was used for development (free tier); in production, cost per deal is approximately $0.0003 at ~800 tokens per inference.

### Integrations: Microsoft Graph API

One OAuth2 app (`n8n-m365-automation`) in Microsoft Entra ID covers Outlook, SharePoint, and Teams.

**Required permissions (Application, admin consent required):**
`Mail.Send` · `Sites.ReadWrite.All` · `Channel.Create` · `ChannelMessage.ReadWrite` · `TeamMember.ReadWrite.All` · `User.Read.All` · `Group.ReadWrite.All`

> **Note:** The Teams member addition node uses **Microsoft Outlook OAuth2** credentials. The `TeamMember.ReadWrite.All` scope must be added to the credential's Custom Scopes in n8n, followed by reconnection to generate a valid token.

---

## AI Agent

A single inference produces all downstream content:

```json
{
  "project_type": "implementacao",
  "risk_level": "medio",
  "suggested_template": "template_B",
  "kickoff_email": { "subject": "...", "html_body": "..." },
  "clickup_tasks": [{ "name": "...", "assignee_role": "...", "due_days_from_now": 5 }],
  "teams_intro_message": "...",
  "project_summary": "..."
}
```

The agent is prompted as a Get2C sustainability consultant — tone and content reflect the company's values, not generic corporate language.

**Template logic:** `template_A` < €20k · `template_B` €20k–€100k · `template_C` > €100k or high risk.

---

## Setup

### Prerequisites
- Docker Desktop · Git · OpenRouter account · ClickUp account · Microsoft 365 with Azure admin access

```bash
git clone https://github.com/VanessaJAvila/Won-to-Onboard-Agentic-Workflow-Automation.git
cd won-to-onboard
cp .env.example .env   # fill in your credentials
docker-compose up -d
# open http://localhost:5678
```

### Import workflow
n8n → **Workflows** → **Import from file** → select `n8n/workflows/pipedrive-won.json`

### Credentials (4 required)

| Credential | Type | Used by |
|---|---|---|
| Microsoft Outlook account | Outlook OAuth2 API | Send email · Add Teams members |
| Microsoft Teams account | Teams OAuth2 API | Create channel · Post message |
| ClickUp account | ClickUp API | Create tasks |
| OpenRouter | HTTP Header Auth (Bearer) | AI inference |

---

## Environment Variables

```env
OPENROUTER_API_KEY=sk-or-v1-...
OPENROUTER_MODEL=google/gemini-2.0-flash-001
CLICKUP_API_TOKEN=pk_...
CLICKUP_LIST_ID=...
SHAREPOINT_DRIVE_ID=...
SHAREPOINT_SOURCE_FOLDER_ID=...
SHAREPOINT_DEST_FOLDER_ID=...
TEAMS_TEAM_ID=...
```

---

## Running the Workflow

With n8n in "Listen for test event" mode, run in **PowerShell**:

```powershell
Invoke-WebRequest -Uri "http://localhost:5678/webhook-test/pipedrive-deal-won" `
  -Method POST -ContentType "application/json" `
  -Body '{"data":{"id":1,"title":"Cliente XYZ Lda","status":"won","value":45000,"currency":"EUR","owner_id":{"name":"Vanessa Avila","email":"vanessa@empresa.com"},"org_id":{"name":"Cliente XYZ Lda"},"close_time":"2026-03-16"},"previous":{"status":"open"}}'
```

> Use PowerShell, not WSL — `localhost` does not resolve to the Docker host from WSL.

**Production webhook:** Pipedrive → Settings → Webhooks → event `updated` / object `deal` → your public URL.

---

## AI Report

### Where AI enters the workflow

```
[1] Webhook received · deal data extracted
[2] Idempotency confirmed
[3] Prompt built from: title, client, value, currency, owner, close date

         ↓  SINGLE AI INFERENCE  ↓

[4] Risk assessed → routing decision
[5] Email generated → Outlook sends
[6] Tasks generated → ClickUp loop creates 5–8
[7] Welcome message generated → Teams posts
[8] Project classified → SharePoint moves folder
[9] 4 team members added to Teams channel
```

### Automated vs. human confirmation

| Decision | Automated | Human required | Rationale |
|---|---|---|---|
| Project type & risk classification | ✅ | — | Deterministic from deal data |
| Template selection | ✅ | — | Consequence of risk + value |
| Kick-off email | ✅ | — | Standard communication |
| ClickUp task list | ✅ | Optional review | AI suggests; team adjusts |
| Teams message + member addition | ✅ | — | Fixed team roles, internal comms |
| SharePoint folder move | ✅ | — | Deterministic, reversible |
| **High-risk deals** | — | ✅ | Financial exposure warrants oversight |

---

## Repository Structure

```
won-to-onboard/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── n8n/workflows/pipedrive-won.json
├── docs/workflow-diagram.png
└── README.md
```

---

*Built with n8n · OpenRouter · Gemini 2.0 Flash · Microsoft Graph API · ClickUp*