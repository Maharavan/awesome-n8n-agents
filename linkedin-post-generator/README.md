# LinkedIn Post Generator

An n8n workflow that generates and publishes LinkedIn posts from two sources — a scheduled Google Sheets content queue or a live Telegram message — with a Tavily web research step and a human-in-the-loop Telegram approval step before posting.

## How it works

```
Telegram Trigger (on-demand)
  → LinkedIn Agent (Leo)
  → Is LinkedIn post?
      │                  │
     YES                 NO
      │                  └──→ Conversational Reply (Telegram)
      ↓
  Normalize (Telegram)           ──┐
                                   ├──→ Merge Paths
Schedule Trigger (daily 9 AM)    ──┘
  → Get row(s) in sheet (NOT-TAKEN)
  → Normalize (Schedule)

           ↓
    Web Search (Tavily)
           ↓
Content Generation Agent
           ↓
      Extract Draft
           ↓
"LinkedIn post okay?" (Telegram sendAndWait — Approve / Decline)
           ↓
     Human in Loop
      │                  │
  Approved            Declined
      │                  └──→ Post declined (Telegram)
      ↓
  Get Profile (LinkedIn API)
  Post to LinkedIn (ugcPosts)
  Save Post to Notion
  Is Schedule source?
      │              │
     YES             NO
      ↓              └──→ POST live (Telegram)
  Mark Sheet as TAKEN (Google Sheets)
      ↓
  POST live (Telegram)
```

1. **Two trigger sources** — The Schedule Trigger fires daily at 9 AM and reads the first Google Sheet row where `Status = NOT-TAKEN`, `Date = today`, and `Flag = Yes`. The Telegram Trigger handles on-demand requests any time.
2. **Route with Leo** — The LinkedIn Agent (`gpt-4.1-mini`) runs immediately on Telegram messages and decides: LinkedIn post topic → `YES`, casual chat → short reply, off-topic → redirect message.
3. **Conversational reply** — Non-post Telegram messages get an immediate reply; the pipeline stops here.
4. **Normalize & merge** — Code nodes convert both sources into a common `{ userMessage, from, source, rowNumber }` shape before merging.
5. **Web research** — Tavily searches for "Latest trends and real world impact of <topic>" to gather fresh data, statistics, and examples.
6. **Generate content** — The Content Generation Agent (`gpt-5`) writes a 150–300 word LinkedIn post using the topic and Tavily research results. It adds a strong hook, short paragraphs, real insights, and 5–10 hashtags. Output format: `TOPIC: …` / `POST DRAFT: …`.
7. **Extract draft** — A Code node parses the agent output into `{ topic, postDraft }`.
8. **Approval via Telegram** — "LinkedIn post okay?" sends the draft with **Approve / Decline** buttons using `sendAndWait`.
9. **Human in Loop** — Branches on `data.approved`.
   - **Approved** → fetches LinkedIn profile (`/v2/userinfo`), posts via `ugcPosts`, saves to Notion, then checks source:
     - **Schedule** → marks the sheet row as `TAKEN`, then sends "✅ Post is now live" via Telegram.
     - **Telegram** → sends "✅ Post is now live" via Telegram directly.
   - **Declined** → sends "❌ LinkedIn post has been declined." via Telegram.

## Nodes used

| Node | Purpose |
|------|---------|
| Schedule Trigger | Fires daily at 09:00 (cron `0 9 * * *`) |
| Telegram Trigger | Receives on-demand messages |
| LinkedIn Agent (gpt-4.1-mini) | Routes Telegram message — post request vs. chat |
| OpenAI (Extractor) | LLM sub-node powering LinkedIn Agent |
| Postgres Chat Memory | Per-user conversation memory for LinkedIn Agent |
| Is LinkedIn post? | Branches on agent output starting with "YES" |
| Conversational Reply (Telegram) | Sends non-post replies back to user |
| Google Sheets — Get row(s) | Fetches the next queued row (NOT-TAKEN, today, Flag=Yes) |
| Normalize (Schedule) | Converts sheet row to common payload |
| Normalize (Telegram) | Converts Telegram message to common payload |
| Merge Paths | Joins both trigger paths |
| Web Search (Tavily) | Searches latest trends and real-world impact for the topic |
| Content Generation Agent (gpt-5) | Writes the LinkedIn post using topic + research data |
| OpenAI (Content) | LLM sub-node powering Content Generation Agent |
| Extract Draft | Parses TOPIC / POST DRAFT from agent output |
| LinkedIn post okay? (Telegram sendAndWait) | Sends draft with Approve / Decline buttons |
| Human in Loop | Branches on `data.approved` |
| Get Profile | `GET /v2/userinfo` — fetches LinkedIn member ID |
| Post to LinkedIn | `POST /v2/ugcPosts` — publishes the post publicly |
| Save Post to Notion | Creates a page in the Notion database (Status: Published) |
| Is Schedule source? | Checks if trigger was the scheduler |
| Google Sheets — Mark Sheet as TAKEN | Updates the sheet row Status to `TAKEN` |
| POST live (Telegram) | Sends "✅ Post is now live" confirmation |
| Post declined (Telegram) | Sends "❌ Post declined" message |

## Setup

### Prerequisites

- n8n running locally (see [root README](../README.md))
- Accounts and credentials for:
  - **Telegram** — Bot token from [@BotFather](https://t.me/BotFather)
  - **OpenAI** — API key (workflow uses `gpt-4.1-mini` and `gpt-5`)
  - **Tavily** — API key for web search ([tavily.com](https://tavily.com))
  - **Google Sheets** — OAuth2 or Service Account with read/write access to your content spreadsheet
  - **LinkedIn** — OAuth2 app with `r_liteprofile` and `w_member_social` scopes
  - **Notion** — Integration token + a database for archived posts
  - **PostgreSQL** — Database for agent conversation memory

### Google Sheets content queue

Create a spreadsheet with these columns (the workflow filters on all three):

| Column | Values |
|--------|--------|
| `Topic` | The post idea or brief |
| `Status ` | `NOT-TAKEN` for queued rows; the workflow writes `TAKEN` after posting |
| `Date` | Date to publish, formatted as `M/dd/yyyy` (e.g. `5/19/2026`) |
| `Flag` | `Yes` to enable the row for processing |

Set the spreadsheet ID and sheet name inside the **Get row(s) in sheet** and **Mark Sheet as TAKEN** nodes.

### Telegram chat ID

The **"LinkedIn post okay?"** and **"POST live"** nodes send to a hardcoded chat ID. Update that value to your own Telegram chat ID. You can get it by messaging [@userinfobot](https://t.me/userinfobot).

### Import the workflow

1. Open n8n at `http://localhost:5678`.
2. Go to **Workflows → Import from file**.
3. Select `Linkedin Post Generator.json`.
4. Open the imported workflow and connect each credential node.

### Required credentials

| Credential | Used by |
|------------|---------|
| Telegram API | Telegram Trigger, LinkedIn post okay?, POST live, Conversational Reply, Post declined |
| Postgres account | Postgres Chat Memory |
| OpenAI API key | OpenAI (Extractor) → LinkedIn Agent, OpenAI (Content) → Content Generation Agent |
| Tavily API | Web Search |
| Google Sheets (OAuth2 or Service Account) | Get row(s) in sheet, Mark Sheet as TAKEN |
| LinkedIn OAuth2 | Get Profile, Post to LinkedIn |
| Notion API | Save Post to Notion |

## Environment variables

No extra env vars beyond the root `.env` are needed. Credentials are stored inside n8n and referenced by the workflow JSON.
