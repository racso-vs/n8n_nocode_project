# Trading Signal Radar - README

Oct 14, 2026 - @Oscar

## Overview

Trading Signal Radar watches five US stocks (AAPL, MSFT, NVDA, SPCX, AMZN) and sends a Telegram alert when their daily indicators trigger a signal. It replaces the routine of refreshing charts and logging price action by hand.

Each alert carries the key numbers and a two-sentence AI summary. A signal is sent once per ticker, signal type and day, so repeated runs stay quiet.

It runs on n8n in Docker on a laptop and reaches Level 3 of the project grid: multi-criteria alerts, state tracking and deduplication.

| Part | Tool | Role |
| --- | --- | --- |
| Backend | n8n in Docker Compose | Triggers, routing, indicator logic |
| Market data | Alpha Vantage | Daily prices and volume, free tier of 25 requests per day |
| Database | Supabase Postgres | Alerts table for deduplication and /status |
| AI | Groq, through a Basic LLM Chain | Two-sentence summary of each new alert |
| Messaging | Telegram bot | Alerts out, commands in |
| Public access | ngrok | HTTPS tunnel so Telegram can reach the local n8n |

## Architecture

The system has three parts: n8n with its three workflows, four external services, and an ngrok tunnel that lets Telegram reach the local instance.

&#91;embedded content: system architecture, 3 workflows, 4 services, 1 tunnel\]

Telegram posts each message to the ngrok tunnel, which hands it to the Telegram Trigger in Main; alerts and replies go straight back to Telegram. Schedule runs, manual runs and /check all call the same sub-workflow, which fetches prices from Alpha Vantage. Main then checks Supabase for duplicates and asks Groq for the summary. The Error Handler sits beside both workflows and messages you when one of them fails.

## The three workflows

The project is split into a shared sub-workflow, a main workflow that owns every trigger, and an error handler.

### Trading Radar - Fetch & Indicators (sub-workflow)

Returns one item per ticker with its indicators. It starts from "When Executed by Another Workflow" (called by Main) or from a Manual trigger used for testing.

1. `Tickers` (Code) returns the five tickers.
2. `HTTP Request` makes one daily-price call per ticker (about 100 days of candles). Batching is one item per request with a 1500 ms interval, Retry On Fail is on (3 tries, 2000 ms), and the key lives in an Alpha Vantage credential.
3. `Indicators` (Code) computes SMA20, SMA50, RSI14 (Wilder), the volume ratio against the 20-day average and the list of signals. A ticker whose request failed returns an `error` field instead.

### Trading Radar - Main

Three entry points: Schedule Trigger, Manual trigger and Telegram Trigger.

The scan path serves the schedule, manual runs and /check: `Call Fetch & Indicators`, then `If` (signal\_count above 0), then the `Supabase HTTP Request` (POST that ignores duplicates), then `Basic LLM Chain` with a Groq Chat Model, then `Edit Fields` (builds the message) and `Send a text message`. A duplicate returns no row, so the chain stops before Telegram.

The command path starts at the Telegram Trigger and goes through a `Switch` on the message text:

| Command | Branch | Result |
| --- | --- | --- |
| /status | `Get recent alerts` (GET, last 5 rows), `Format status` (Code), `Send a text message` | Latest alerts, with no Alpha Vantage request |
| /check | `Edit Fields2` and `Send a text message2` (acknowledgement), then the scan path | Acknowledgement first, then alerts if there are any |
| Anything else | `Edit Fields1`, `Send a text message` | Help text listing the two commands |

### Trading Radar - Error Handler

An Error Trigger feeds a Telegram node that sends the failed workflow's name, the last node run and the error message. It is attached through Settings, Error Workflow, in both Main and Fetch & Indicators, and it only runs for published, automatic executions.

## Requirements mapping

Every minimum technical requirement from the proposal is covered by a node in the build.

| Requirement | Where it is met |
| --- | --- |
| If or Set node | `If` checks signal\_count above 0; the `Edit Fields` nodes build the Telegram text |
| Code node | `Indicators` (SMA20, SMA50, RSI14, volume ratio, signals) and `Format status` |
| Router or Switch | `Switch` on the Telegram message text: /status, /check, fallback |
| API or MCP connection | HTTP Request nodes for Alpha Vantage and the Supabase REST API, plus the Groq Chat Model |
| Message trigger | `Telegram Trigger`, reached through the ngrok tunnel |
| Database for persistence | Supabase `alerts` table with a unique constraint on ticker, signal type and date |
| AI node | `Basic LLM Chain` with the Groq Chat Model |
| Sub-workflow | Fetch & Indicators, called from Main by an Execute Workflow node |
| Error handling and retries | Error Handler workflow; Retry On Fail on the Alpha Vantage request and the LLM chain |

## Setup

You need Docker Desktop and free accounts at Telegram (BotFather), Alpha Vantage, Supabase, Groq and ngrok.

### 1. Supabase table

Create a project, open the SQL Editor and run:

```sql
create table alerts (
  id bigint generated always as identity primary key,
  ticker text not null,
  signal_type text not null,
  signal_date date not null,
  price numeric,
  summary text,
  created_at timestamptz default now(),
  unique (ticker, signal_type, signal_date)
);
```

The unique constraint is the deduplication. n8n talks to the table through the Supabase REST API (`/rest/v1/alerts`) over HTTPS, with the secret key stored in a Header Auth credential named `apikey`.

### 2. n8n and the tunnel in Docker Compose

n8n runs from `compose.yml` with its task runners and sandbox services. Two changes make Telegram reachable. In the `n8n` service, add a `restart` policy and an `environment` block that holds the public URL, the time zone and the proxy setting. Then add an `ngrok` service that forwards your free ngrok domain to n8n:

```yaml
  n8n:
    restart: unless-stopped
    env_file: .env
    environment:
      WEBHOOK_URL: https://YOUR-DOMAIN.ngrok-free.dev/
      GENERIC_TIMEZONE: Europe/Paris
      TZ: Europe/Paris
      N8N_PROXY_HOPS: '1'

  ngrok:
    image: ngrok/ngrok:latest
    restart: unless-stopped
    command: http --url=YOUR-DOMAIN.ngrok-free.dev n8n:5678
    environment:
      NGROK_AUTHTOKEN: ${NGROK_AUTHTOKEN}
    ports:
      - '4040:4040'
    depends_on:
      - n8n
```

Put `NGROK_AUTHTOKEN=your_token` in the `.env` file next to `compose.yml`, then run `docker compose up -d`. Never run `docker compose down -v`: the `-v` flag deletes the volume that holds your workflows and credentials. The editor stays at http://localhost:5678, and ngrok's inspector is at http://localhost:4040.

### 3. Credentials in n8n

| Credential | Type | Used by |
| --- | --- | --- |
| Telegram account | Telegram API (bot token from BotFather) | Telegram Trigger and the send nodes |
| Alpha Vantage key | Query Auth, name `apikey` | HTTP Request in the sub-workflow |
| Supabase key | Header Auth, name `apikey` | Insert and `Get recent alerts` |
| Groq account | Groq API key | Groq Chat Model |

Keep every key in a credential, never in a node parameter, so the workflows can be exported safely.

### 4. Workflows

1. Build or import Fetch & Indicators, then publish it.
2. Build Error Handler (Error Trigger, then a Telegram node) and publish it.
3. In Main and in Fetch & Indicators, open Settings and select Error Handler under Error Workflow.
4. In Main, set the time zone to Europe/Paris, set the Schedule Trigger to the cron `0 17,23 * * 1-5`, and in the Telegram Trigger use Additional Fields, Restrict to Chat IDs, with your own chat ID so nobody else can send commands.
5. Publish Main.

## Usage

The scan runs on weekdays at 17:00 and 23:00 Paris time (cron `0 17,23 * * 1-5`). The 17:00 run happens mid-session in the US; the 23:00 run happens after the close, so it reads the finished daily candle. You can also trigger a scan at any time with /check.

### Commands

| Message to the bot | Reply |
| --- | --- |
| /status | The five latest alerts from the database; no Alpha Vantage request |
| /check | An acknowledgement, then a scan now (5 Alpha Vantage requests); alerts follow if there are any |
| Anything else | A help line listing the commands |

### Signals

| Signal | Condition |
| --- | --- |
| golden\_cross | SMA20 crosses above SMA50 on the latest candle |
| death\_cross | SMA20 crosses below SMA50 on the latest candle |
| rsi\_overbought | RSI14 above 70 |
| rsi\_oversold | RSI14 below 30 |
| volume\_spike | Latest volume above 1.5 times the 20-day average |

### Alert format

A real alert from the first live run looks like this:

```
🚨 MSFT (2026-10-13)
Close: 543.45
RSI14: 70.7
SMA20: 514.2 | SMA50: 503.09
Signals: rsi_overbought
Summary: The RSI is high at 70.7, showing overbought conditions. MSFT is trading above both its 20-day and 50-day moving averages with volume slightly above the 20-day average, indicating a short-term bullish trend.
```

### Deduplication

Before a message is built, the workflow inserts the signal into the `alerts` table. The insert ignores a row whose ticker, signal type and date already exist, returns nothing, and the chain stops. To replay an alert while testing, delete its row in the Supabase Table Editor.

## Limits and operations

The workflow only runs while the laptop is awake and Docker is up, and the free Alpha Vantage tier caps how often it can scan.

- **Alpha Vantage quota:** 25 requests per day on the free tier. A scan costs 5 (one per ticker), so two scheduled runs use 10 and each /check uses 5. That leaves room for about 3 manual checks a day. The quota resets daily.
- **Laptop and Docker:** the Mac must be awake at 17:00 and 23:00, or that run is skipped. Docker Desktop is set to start at login, and the containers use `restart: unless-stopped`, so n8n and ngrok come back after a reboot.
- **Tunnel domain:** `WEBHOOK_URL` in `compose.yml` must match the ngrok domain exactly, including the ending. Inspect Telegram's requests at http://localhost:4040.
- **Editor tests versus production:** clicking Execute step on the Telegram Trigger replaces the production webhook with a test one, so commands stop answering. Unpublish and publish Main again to restore it. Editor runs also use pinned data when a node is pinned.
- **Error Handler:** it only fires for published, automatic executions, never for a run started from the editor.
- **SPCX:** it listed in June 2026, so it has fewer than 100 days of history. That is enough for SMA20, SMA50 and RSI14, but not for a 200-day average.
- **Not advice:** the summary prompt forbids financial advice and price predictions, and the alerts are indicator readings, not recommendations.

## Troubleshooting

These are the problems met while building the project and how each was fixed.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Alpha Vantage returns "Invalid API call" | A required parameter (`symbol`, `apikey`) is missing or misspelled | Rebuild the node with Import cURL |
| `Indicators` returns ticker UNKNOWN with a rate-limit message | The 25 requests per day quota is used up | Wait for the daily reset; pin data or scan one ticker while testing |
| Postgres node: "Connection terminated unexpectedly" | The Supabase pooler drops the connection from a home network | Use the Supabase REST API with an HTTP Request node instead |
| `getUpdates` returns an empty result | The bot has received no message yet | Send /start and a message to your own bot first |
| Telegram Trigger: "Failed to resolve host" | `WEBHOOK_URL` has a wrong domain ending | Copy the domain exactly from ngrok, then run `docker compose up -d` |
| The summary is a number such as 0.0017 | A Prompt Guard model, which is a safety classifier, was selected | Choose a text model such as openai/gpt-oss-20b |
| The message shows rsioverbought | Markdown parse mode swallows underscores | Set Parse Mode to HTML on the Telegram node |
| Everything goes to the false branch of `If` | The RSI test threshold of 65 was set back to 70 | Expected; use 65 only for tests |
| Commands stop answering after an editor test | The test webhook replaced the production one | Unpublish and publish Main |
| The alert arrives before "Scan started" | The two /check branches ran in parallel | Chain them: acknowledgement, then the scan |
| No message on a rerun | Deduplication is working | Delete the row in Supabase to replay the alert |
