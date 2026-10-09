# SwarmAI: Multi-Agent Personal Assistant (n8n)

A no-code multi-agent personal assistant built on **n8n**, **OpenAI**, **Gemini** and **Docker**.

```
 Web UI   Telegram Bot   Voice Control   Mobile App (PWA)
    \          |              |              /
     ───────── n8n Workflow Automation Platform ─────────
                            │
              MASTER AGENT (Orchestrator, GPT-4o)
         route tasks · coordinate agents · memory
        ┌──────────┬──────────┼──────────┬──────────┐
   EMAIL AGENT  CALENDAR   RESEARCH AGENT   TRAVEL AGENT     ← sub-workflows
   (Gmail)      (Google)   (Gemini+SerpAPI) (SerpAPI flights/hotels)
```

## Project layout

| Path | What it is |
|---|---|
| `docker-compose.yml` | n8n + Postgres + nginx web UI (+ one-shot workflow importer) |
| `workflows/01_master_agent.json` | Entry points (Telegram text/voice, webhook), master agent, memory, routing of replies |
| `workflows/02_email_agent.json` | Sub-workflow: search / send / reply Gmail |
| `workflows/03_calendar_agent.json` | Sub-workflow: list / create / update / delete Google Calendar events |
| `workflows/04_research_agent.json` | Sub-workflow: Gemini + Google Search (SerpAPI) + Wikipedia |
| `workflows/05_travel_agent.json` | Sub-workflow: Google Flights + Google Hotels via SerpAPI |
| `web-ui/` | Chat UI with voice input (speech-to-text) and spoken replies, installable on phones |

### How it works
1. A message arrives from **Telegram** (text, or a voice note that is transcribed with OpenAI Whisper) or from the **web/mobile UI** (`POST /webhook/assistant`).
2. `Normalize Input` converts every channel to `{ text, sessionId, channel, chatId }`.
3. The **Master Agent** (GPT-4o + per-session window memory) decides which specialists to call. Each specialist is an n8n **sub-workflow** exposed as a tool (`Call n8n Workflow Tool` → `Execute Workflow Trigger`).
4. Specialists run their own AI agent with their own tools and return a text result.
5. The master combines the results and replies on the same channel.

## Setup

### 1. Prerequisites
- Docker with the Compose plugin. On Ubuntu, if `docker compose` says *unknown command*, run: `sudo apt install docker-compose-v2`
- API keys / accounts:
  - **OpenAI API key** (master, email, calendar and travel agents, plus voice transcription)
  - **Google Gemini API key** (research agent): https://aistudio.google.com/apikey
  - **SerpAPI key** (search, flights, hotels): https://serpapi.com
  - **Google Cloud OAuth client** with the Gmail API and Google Calendar API enabled
  - **Telegram bot token** from [@BotFather](https://t.me/BotFather)

### 2. Configure and start
```bash
cp .env.example .env          # set passwords + N8N_ENCRYPTION_KEY (openssl rand -hex 32)
docker compose up -d postgres
docker compose --profile setup run --rm n8n-import   # imports the 5 workflows (run ONCE)
docker compose up -d
```
- n8n editor: http://localhost:5678 (create the owner account on first visit)
- Assistant web UI: http://localhost:8080

> Running the importer again overwrites the workflows and clears the credentials you selected in them.

### 3. Create credentials in n8n (Settings → Credentials → Add)
| Credential type | Used by |
|---|---|
| **OpenAI** | Chat models in Master/Email/Calendar/Travel, *Transcribe Voice* |
| **Google Gemini (PaLM) API** | Research agent model |
| **SerpAPI** | Research agent *Google Search* |
| **Query Auth**: name `api_key`, value = your SerpAPI key | Travel agent *Search Flights* / *Search Hotels* |
| **Gmail OAuth2** | Email agent tools |
| **Google Calendar OAuth2** | Calendar agent tools |
| **Telegram** | Telegram Trigger, Download Voice, Reply on Telegram |

For Google OAuth, add `http://localhost:5678/rest/oauth2-credential/callback` (or your public URL) as an authorized redirect URI in Google Cloud Console.

Open each workflow, click every node with a warning icon, select the right credential, and save.

### 4. Activate
- Activate **only** `SwarmAI - Master Agent (Orchestrator)` (toggle at the top right). The sub-workflows run when the master calls them and do not need to be active.
- **Telegram** only accepts a public **HTTPS** webhook. For local dev, start a tunnel and set `WEBHOOK_URL` in `.env`:
  ```bash
  cloudflared tunnel --url http://localhost:5678     # or: ngrok http 5678
  # .env -> WEBHOOK_URL=https://<your-tunnel>.trycloudflare.com/
  docker compose up -d n8n                          # restart, then re-activate the master workflow
  ```
  The web UI works without a tunnel.

## Try it
- "Summarise my unread emails from today"
- "Reply to the latest email from Rahul saying I'll send the report by Friday"
- "What's on my calendar tomorrow? Book a 30 min call with the team at 4pm"
- "Research the latest developments in multi-agent AI frameworks"
- "Find one-way flights Hyderabad to Goa on 2026-11-14 and hotels near Calangute for 2 nights"
- "Find flights to Bangalore next Monday morning and block that time on my calendar" (chains travel → calendar)
- Send the Telegram bot a **voice note**, or tap 🎤 in the web UI.

Test the API directly:
```bash
curl -X POST http://localhost:5678/webhook/assistant \
  -H 'Content-Type: application/json' \
  -d '{"message":"What can you do?","sessionId":"test"}'
```

## Customising
- **Models:** change the model in any *Chat Model* node, e.g. switch the master to Gemini by replacing its OpenAI Chat Model with a Google Gemini Chat Model node.
- **Persistent memory:** the window buffer memory is kept in n8n process memory and is lost on restart. To keep it, swap it for *Postgres Chat Memory* (host `postgres`, credentials from `.env`).
- **New agent:** duplicate a sub-workflow, change its tools and prompt, add a *Call n8n Workflow Tool* node to the master pointing at it, and describe it in the master's system prompt.
- **Mobile app:** open the web UI on your phone (expose port 8080 through the tunnel) and choose *Add to Home Screen*.

## Troubleshooting
- **Web UI shows `HTTP 404`:** the Master Agent workflow is not active.
- **Telegram is silent:** `WEBHOOK_URL` must be public HTTPS; re-activate the workflow after changing it.
- **"Workflow does not exist" from a tool:** the sub-workflows must keep their imported IDs (`SwarmEmail000001`, …). Re-import them, or re-select the workflow in the tool node.
- **Executions:** n8n → *Executions* shows every agent and tool call step by step.
