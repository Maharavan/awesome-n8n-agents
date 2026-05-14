# awesome-n8n-agents

A collection of production-ready n8n automation workflows, each with its own Docker-based local stack.

## Stack

| Service  | Image                  | Port  |
|----------|------------------------|-------|
| n8n      | `n8nio/n8n:latest`     | 5678  |
| Postgres | `postgres:16-alpine`   | —     |

n8n stores all workflow data and credentials in Postgres. No reverse proxy is needed for local development — n8n is accessible directly at `http://localhost:5678`.

## Quick start

### 1. Clone & configure

```bash
git clone https://github.com/maharavan/awesome-n8n-agents.git
cd awesome-n8n-agents
cp .env.example .env
```

Edit `.env` and fill in the required values:

| Variable            | Description                                              |
|---------------------|----------------------------------------------------------|
| `POSTGRES_PASSWORD` | Strong password for the Postgres `n8n` user              |
| `N8N_ENCRYPTION_KEY`| 32-byte hex key — generate with `openssl rand -hex 32`   |
| `TIMEZONE`          | Optional — timezone for the scheduler, e.g. `Asia/Kolkata` (default: `UTC`) |

### 2. Start

```bash
docker compose up -d
```

n8n will be available at **http://localhost:5678**.

### 3. Stop

```bash
docker compose down          # keep data volumes
docker compose down -v       # also remove volumes (wipes all data)
```

## Workflows

| Workflow | Description |
|----------|-------------|
| [LinkedIn Post Generator](linkedin-post-generator/) | Dual-trigger (Telegram + Google Sheets schedule) → AI agents → human approval → LinkedIn post, archived to Notion |

## Project layout

```
awesome-n8n-agents/
├── docker-compose.yml          # n8n + Postgres stack
├── .env.example                # environment variable template
├── linkedin-post-generator/
│   ├── Linkedin Post Generator.json          # importable n8n workflow
│   └── README.md
└── README.md
```
