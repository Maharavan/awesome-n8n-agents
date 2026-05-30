# Telegram Atlas Agent

An n8n workflow that powers a Telegram chatbot named **Atlas** — a personal AI assistant that receives messages via Telegram, searches the web, sends and reads emails, and manages Google Calendar events.

## How it works

```
Telegram Trigger (inbound message)
        ↓
    AI Agent (Atlas / OpenAI)
        ├──→ Web Search (Tavily)       — news, facts, docs, tech info
        ├──→ Calculator                — math and expressions
        ├──→ Weather Tool              — current weather for any city
        ├──→ Gmail Send                — compose and send emails
        ├──→ Gmail Get                 — read and retrieve emails
        ├──→ Calendar Get              — check upcoming events
        └──→ Calendar Create           — schedule new events
        ↓
Telegram reply (back to user)
```

1. **Inbound trigger** — Telegram Trigger listens for incoming messages and passes the message body to the agent.
2. **AI Agent (Atlas)** — A LangChain agent powered by OpenAI. It reads the user's message, decides whether a tool is needed, and responds naturally.
   - Small talk / greetings → replies directly, no tools
   - Factual or current info → calls **Web Search** (Tavily)
   - Math expressions → calls **Calculator**
   - Weather queries → calls **Weather Tool** (OpenWeatherMap)
   - Send email request → calls **Gmail Send** (collects only missing details)
   - Read email request → calls **Gmail Get**
   - Check schedule → calls **Calendar Get**
   - Create event → calls **Calendar Create** (collects only missing details)
3. **Simple Memory** — Buffer-window memory scoped per sender, so each user gets their own conversation context.
4. **Telegram reply** — The agent's response is sent back to the user on Telegram.

## Nodes used

| Node | Purpose |
|------|---------|
| Telegram Trigger | Receives inbound Telegram messages |
| AI Agent (Atlas) | Decides which tool to use and generates replies |
| OpenAI Chat Model | LLM powering the AI Agent |
| Web Search (Tavily) | Searches the internet for current or factual info |
| Calculator | Evaluates math expressions |
| Weather Tool | Gets current weather for a city via OpenWeatherMap |
| Gmail Send | Composes and sends emails on behalf of the user |
| Gmail Get | Reads and retrieves emails from the user's inbox |
| Calendar Get | Fetches upcoming Google Calendar events |
| Calendar Create | Creates new Google Calendar events |
| Simple Memory | Per-sender buffer-window memory |

## Setup

### Prerequisites

- n8n running locally (see [root README](../README.md))
- Accounts and credentials for:
  - **Telegram** — Bot token from [@BotFather](https://t.me/BotFather)
  - **OpenAI** — API key
  - **Tavily** — API key for web search ([tavily.com](https://tavily.com))
  - **OpenWeatherMap** — API key ([openweathermap.org](https://openweathermap.org/api))
  - **Gmail** — OAuth2 credentials (Google Cloud Console)
  - **Google Calendar** — OAuth2 credentials (same Google Cloud project as Gmail)

### Telegram bot setup

1. Open Telegram and message [@BotFather](https://t.me/BotFather).
2. Send `/newbot` and follow the prompts to create a bot and get the token.
3. In n8n, add the bot token as a **Telegram credential**.
4. Activate the workflow — n8n will register the webhook automatically.

### Gmail & Google Calendar setup

1. Go to [Google Cloud Console](https://console.cloud.google.com) and create a project.
2. Enable the **Gmail API** and **Google Calendar API**.
3. Create **OAuth 2.0 credentials** (Desktop or Web app type).
4. In n8n, add the credentials under **Gmail OAuth2** and **Google Calendar OAuth2**.
5. Authorize the credentials by following the OAuth flow in n8n.

### Import the workflow

1. Open n8n at `http://localhost:5678`.
2. Go to **Workflows → Import from file**.
3. Select `Personal Agent.json`.
4. Open the imported workflow and connect each credential node.

### Required credentials

| Credential | Used by |
|------------|---------|
| Telegram API | Telegram Trigger, Telegram reply |
| OpenAI API key | OpenAI Chat Model → AI Agent |
| Tavily API | Web Search |
| OpenWeatherMap API | Weather Tool |
| Gmail OAuth2 | Gmail Send, Gmail Get |
| Google Calendar OAuth2 | Calendar Get, Calendar Create |

## System prompt

The agent's system prompt is embedded directly in the **AI Agent** node inside `Personal Agent.json`. Atlas uses web search for all factual questions, calls the weather tool for weather queries, and only triggers Gmail or Calendar tools when the user explicitly requests it — casual messages get a natural reply with no tool calls. The prompt also handles IST↔UTC conversion for calendar operations.
