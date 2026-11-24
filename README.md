# AI Customer Support

An AI-assisted email support system for Harbor & Pine that monitors a Gmail inbox, classifies customer requests, proposes replies from an editable knowledge base, and escalates messages that require a human colleague.

The project contains:

- a FastAPI backend and Gmail polling service;
- a Pydantic AI classifier and support agent;
- PostgreSQL/Supabase storage for events, tickets, messages, and knowledge;
- a React dashboard for monitoring activity and editing support information.

> The agent is designed to draft and send support replies. Review the escalation rules and knowledge base carefully before running it against a real customer inbox.

## Table of contents

- [How it works](#how-it-works)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Google Gmail setup](#google-gmail-setup)
- [Database setup](#database-setup)
- [Knowledge base](#knowledge-base)
- [Running the application](#running-the-application)
- [Dashboard development](#dashboard-development)
- [Evaluation](#evaluation)
- [Operational behavior](#operational-behavior)
- [HTTP API](#http-api)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

## How it works

1. `app/poller.py` searches the Gmail inbox for unread incoming messages that do not have the `ai-processed` label.
2. `app/pipeline.py` creates or updates a ticket and stores the incoming message.
3. The classifier categorizes the thread and decides whether an escalation rule already requires a person.
4. The support agent reads relevant entries in the knowledge base and generates a plain-text reply.
5. Python policy rules in `app/ai/rules.py` apply deterministic safeguards for attachments, requested actions, missing product details, refunds, and other escalation cases.
6. The system either sends a reply or applies the Gmail `needs-human` label and records the proposed status.
7. Every Gmail message receives the `ai-processed` label after handling, and events are stored with status and retry information.

The dashboard shows event history, tickets and conversations, and allows knowledge-base entries to be edited.

## Features

- Gmail OAuth authentication and automatic label management
- Pydantic AI classification and structured replies
- Knowledge-base retrieval with database-backed content
- Deterministic escalation policies in addition to model decisions
- Automatic handling of sent mail, automated mail, bulk mail, and own-address replies
- Retry tracking for failed Gmail or AI operations
- Human escalation through the `needs-human` Gmail label
- Basic HTTP authentication for the dashboard and protected API endpoints
- React dashboard for events, tickets, and knowledge-base management
- Golden-email evaluations for classification and action decisions

## Tech stack

**Backend**

- Python 3.13+
- FastAPI and Uvicorn
- Pydantic AI
- Pydantic Settings
- Google Gmail API
- psycopg
- PostgreSQL or Supabase

**Dashboard**

- React 19
- TypeScript
- Vite
- Tailwind CSS
- shadcn/ui and Radix UI

**Development tools**

- uv
- Pydantic Evals
- Supabase CLI
- npm

## Repository layout

```text
app/
  ai/                 Pydantic AI agents, schemas, classifier, and policy rules
  config.py           Environment settings
  db.py               PostgreSQL queries
  gmail.py            Gmail API integration
  guards.py           Incoming-message filters
  main.py             FastAPI application and dashboard mounting
  pipeline.py         End-to-end email processing
  poller.py           Inbox polling loop

dashboard/
  src/pages/          Events, tickets, and knowledge-base UI
  src/lib/api.ts      Dashboard API and basic-auth client

evals/
  golden_emails.py    Golden test email cases
  run_evals.py        Pydantic Evals runner

knowledge_base/
  *.md                Human-editable support knowledge

scripts/
  get_refresh_token.py  Generate Gmail OAuth credentials
  gmail_check.py        Test Gmail connection and sending
  seed_knowledge_base.py Import Markdown into PostgreSQL

supabase/
  migrations/         Database schema migration
  config.toml         Local Supabase configuration
```

## Prerequisites

- Python 3.13 or newer
- uv
- Node.js 20 or newer
- npm
- PostgreSQL, or Docker with the Supabase CLI for local development
- a Google Cloud project with Gmail API enabled
- an AI model provider supported by Pydantic AI

## Quick start

### 1. Install Python dependencies

From the project root:

```powers
uv sync
```

### 2. Configure the environment

Create a local `.env` file from the example:

```powers
Copy-Item .env.example .env
```

Complete the required values described in [Configuration](#configuration). Never commit `.env`, OAuth credentials, refresh tokens, API keys, or database passwords.

### 3. Start PostgreSQL or Supabase

For local Supabase:

```powers
supabase start
```

Apply the migrations:

```powers
supabase db reset
```

Obtain the local connection string printed by Supabase or shown in Supabase Studio, and assign it to `DATABASE_URL`.

### 4. Create Gmail OAuth credentials

Place the OAuth desktop client JSON downloaded from Google in `credentials.json`, then run:

```powers
uv run python scripts/get_refresh_token.py
```

Add the printed client ID, client secret, and refresh token to `.env`.

### 5. Seed the knowledge base

```powers
uv run python scripts/seed_knowledge_base.py
```

The seed script reads every `knowledge_base/*.md` file and imports entries split on `##` headings.

### 6. Build the dashboard

```powers
cd dashboard
npm install
npm run build
cd ..
```

### 7. Start the API and dashboard

```powers
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Open `http://127.0.0.1:8000`. The FastAPI process mounts the built dashboard from `dashboard/dist`.

### 8. Start the inbox poller

In a separate terminal:

```powers
uv run python app/poller.py
```

The poller checks the inbox every `POLL_INTERVAL_SECONDS` seconds.

## Configuration

Create `.env` using `.env.example` as a template:

| Variable | Required | Description |
| --- | --- | --- |
| `GOOGLE_CLIENT_ID` | Yes | OAuth client ID from Google Cloud |
| `GOOGLE_CLIENT_SECRET` | Yes | OAuth client secret |
| `GOOGLE_REFRESH_TOKEN` | Yes | Refresh token generated for the Gmail account |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `GOOGLE_API_KEY` | When used | API key for an AI provider that requires it |
| `CLASSIFIER_MODEL` | Yes | Pydantic AI model identifier for classification |
| `AGENT_MODEL` | Yes | Pydantic AI model identifier for reply generation |
| `POLL_INTERVAL_SECONDS` | No | Polling delay; defaults to `60` |
| `DASHBOARD_USERNAME` | No | Dashboard/API username; defaults to `admin` |
| `DASHBOARD_PASSWORD` | Recommended | Password required for dashboard and protected API endpoints |
| `PYDANTIC_AI_NO_BANNER` | No | Set to `1` to suppress the Pydantic AI banner |

Example model configuration depends on the provider supported by Pydantic AI:

```dotenv
CLASSIFIER_MODEL=provider:model-name
AGENT_MODEL=provider:model-name
```

For Google AI Studio models, use the model identifier format supported by Pydantic AI and set `GOOGLE_API_KEY`. For Anthropic or another provider, configure the corresponding provider environment variable documented by Pydantic AI.

The dashboard login is disabled effectively if `DASHBOARD_PASSWORD` is empty: protected API requests are rejected. Set a strong password before exposing the service.

## Google Gmail setup

1. Create or select a Google Cloud project.
2. Enable the **Gmail API**.
3. Configure the OAuth consent screen.
4. Create an OAuth 2.0 **Desktop app** client.
5. Download the client JSON and save it as `credentials.json` in the repository root.
6. Run `uv run python scripts/get_refresh_token.py`.
7. Approve Gmail modification access in the browser flow.

The application requests the `gmail.modify` scope and can add labels, read threads, and send replies.

To confirm that Gmail credentials work:

```powers
uv run python scripts/gmail_check.py
```

> The Gmail check script reads one inbox message and sends a test reply. Do not run it until you are comfortable with the target account.

Gmail labels are created automatically:

- `ai-processed`: added after a message is handled or skipped
- `needs-human`: added when the message should be handled by a person

## Database setup

The migration at `supabase/migrations/20261124144243_create_tables.sql` creates:

- `events`: processing status, attempt count, error, and result
- `tickets`: one row per Gmail thread
- `messages`: incoming and outgoing message history
- `knowledge_base`: editable support entries

For a hosted PostgreSQL or Supabase database, set `DATABASE_URL` to a server-side PostgreSQL connection string that the Python process can use. Apply the migration in Supabase SQL Editor or through Supabase CLI/database tooling.

The migration enables row-level security on the public tables. This application connects directly with PostgreSQL credentials and is not currently using Supabase's browser/API role model.

## Knowledge base

Markdown files under `knowledge_base/` are organized by sections:

- `brand-and-tone.md`
- `escalation-rules.md`
- `paddle-boards.md`
- `shelving.md`
- `returns-and-warranty.md`
- `shipping.md`

Each `##` heading becomes one database entry:

```markdown
# Shipping

## Tracking a parcel
Tell the customer to check their tracking link. Do not promise a delivery date.

## Replacement parts
Ask for the order number and part number.
```

After editing Markdown files, run:

```powers
uv run python scripts/seed_knowledge_base.py
```

Entries can also be created or changed in the dashboard's **Knowledge base** tab. Dashboard changes are stored directly in PostgreSQL.

## Running the application

### Production-style local run

Build the dashboard, then run FastAPI:

```powers
cd dashboard
npm run build
cd ..
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Run the poller in a second process:

```powers
uv run python app/poller.py
```

Health check:

```powers
Invoke-RestMethod http://127.0.0.1:8000/health
```

For development, use separate frontend and backend processes:

```powers
# Terminal 1
uv run uvicorn app.main:app --reload --port 8000

# Terminal 2
cd dashboard
npm run dev
```

Open the Vite URL printed by the dev server, usually `http://127.0.0.1:5173`. Vite proxies `/api` requests to port `8000`.

## Dashboard development

Useful dashboard commands:

```powers
cd dashboard
npm run dev
npm run build
npm run lint
npm run typecheck
npm run format
```

The dashboard provides:

- **Events**: processing status, attempts, and errors, plus **Check inbox now**
- **Tickets**: customer threads, status, category, and proposed action
- **Knowledge base**: editing support entries

You can trigger one polling cycle from **Events > Check inbox now**. The protected API also exposes `POST /api/poll`.

## Evaluation

Run golden-email evaluations:

```powers
uv run python evals/run_evals.py
```

The evaluator checks:

- the section selected by the classifier;
- whether the final decision is `reply` or `escalate`.

The email cases are stored in `evals/golden_emails.py`. Add focused cases to this file when changing escalation rules or classifier instructions.

You can also run an example pipeline without Gmail:

```powers
uv run python playground/try_pipeline.py
```

## Operational behavior

- Polling is capped at 10 new Gmail messages per cycle.
- Failed messages are retried. The event attempt counter increments only when the database status is `failed`.
- A message is retried at most three times for label/result purposes.
- Bad messages do not stop the rest of a polling cycle.
- Already processed or currently processing Gmail message IDs are not handled again.
- Threads containing `needs-human` are left for a colleague and are not answered again.
- Emails with attachments, always-escalate topics, unsafe missing-information cases, or promised actions are escalated.
- The system is one process per account and does not include a distributed job queue.

For live operation, run the API and poller as supervised background services and ensure they share the same environment variables and database.

## HTTP API

The `/api` routes require HTTP Basic authentication, except that `GET /health` is public.

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Basic service health |
| `GET` | `/api/login` | Verify dashboard credentials |
| `GET` | `/api/events` | List recent processing events |
| `GET` | `/api/tickets` | List recent tickets |
| `GET` | `/api/tickets/{ticket_id}/messages` | List messages for a ticket |
| `GET` | `/api/knowledge-base` | List knowledge-base entries |
| `PUT` | `/api/knowledge-base/{entry_id}` | Update an entry's title and content |
| `POST` | `/api/poll` | Process one inbox polling cycle |

Interactive API documentation is available at:

- Swagger UI: `http://127.0.0.1:8000/docs`
- OpenAPI JSON: `http://127.0.0.1:8000/openapi.json`

## Troubleshooting

### `Settings` validation error at startup

Ensure `.env` exists and contains `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN`, `DATABASE_URL`, `CLASSIFIER_MODEL`, and `AGENT_MODEL`. Pydantic Settings loads `.env` relative to the process environment.

### Gmail authentication fails

- Confirm that the OAuth consent screen and Gmail API are configured.
- Re-run `scripts/get_refresh_token.py` after changing OAuth credentials or scopes.
- Verify that the refresh token belongs to the intended Gmail account.
- Check that the OAuth app is published or that the account is a test user.

### Database connection fails

- Verify `DATABASE_URL` is a direct PostgreSQL connection string.
- Confirm that all migrations have been applied.
- For local Supabase, run `supabase status` and copy the current database URL.

### Knowledge base entry is missing

Run `uv run python scripts/seed_knowledge_base.py`, then confirm the entry exists through the dashboard or PostgreSQL.

### Dashboard shows 401 responses

Set `DASHBOARD_PASSWORD`, restart FastAPI, and log in with `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD`. Basic-auth credentials are stored only in browser session storage for the current tab session.

### Dashboard is not served

Build it with `npm run build` inside `dashboard/`. FastAPI mounts `dashboard/dist` only when that directory exists.

## Security notes

- Keep `credentials.json`, `.env`, refresh tokens, API keys, and database credentials outside version control.
- Use a strong `DASHBOARD_PASSWORD` and do not expose FastAPI directly to an untrusted network without HTTPS and stronger authentication.
- The dashboard currently uses HTTP Basic authentication and no CSRF/session server. Add HTTPS and a production identity or session system before broader deployment.
- The Gmail account is an operational mailbox: test with dedicated data and avoid running sending scripts in production unintentionally.
- Human escalation should be reviewed from Gmail and/or the dashboard before claims or commitments are made.