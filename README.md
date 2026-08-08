# WhatsApp Campaign & Inbox Platform

A commercial-ready rebuild of the original MedCross WhatsApp chatbot — the AI
chatbot has been removed entirely. What's left is a lean campaign + inbox
platform: import contacts, send templated WhatsApp campaigns, track delivery
and read receipts, and handle replies from a shared team inbox.

## What's here vs. what changed from the original repo

**Kept:** the WhatsApp Cloud API transport layer (`whatsapp.py`) — sending
text/template messages, parsing incoming webhook payloads, marking messages
read. That part of the original project was solid and needed no rework.

**Removed:** the AI chatbot / conversation engine, the disease-and-slot
booking flow, and the `leads` table that was shaped around it.

**New:** the whole data model and UI — contacts/groups/tags, campaigns with
per-recipient delivery tracking, a two-pane inbox, bulk replies, template
management, filterable message logs, and a settings page for API credentials.

## Stack

- **Backend:** FastAPI + SQLite (`platform.db`, created automatically on
  first run), APScheduler for scheduled campaigns.
- **Frontend:** Plain HTML/CSS/JS (no build step) served directly by
  FastAPI as static files, so there's a single process to run.

## Getting started

```bash
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# edit .env and fill in your WhatsApp Cloud API credentials
# (you can also leave placeholders and set real values later from the
# Settings page in the app — those override the .env file)

uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Open **http://localhost:8000** — the app (frontend + API) is served from the
same origin.

## Connecting to Meta / WhatsApp Cloud API

1. In [Meta for Developers](https://developers.facebook.com/), create/open
   a WhatsApp Business app and grab your **Phone Number ID**, **WhatsApp
   Business Account ID**, and a permanent **Access Token** (via a System
   User in Business Manager — temporary tokens expire in 24h).
2. Put those in `.env` (or the Settings page).
3. In the app's Webhook configuration, set the Callback URL to
   `https://your-domain.com/webhook` and the Verify Token to whatever you
   set as `WHATSAPP_VERIFY_TOKEN`. You'll need a public HTTPS URL — use a
   tunnel (e.g. ngrok) for local testing.
4. Subscribe to the `messages` webhook field so incoming replies and
   delivery/read status updates reach the Inbox and Campaign Analytics.
5. **Templates**: this app doesn't submit templates to Meta for approval —
   that still happens in Meta Business Manager. Once a template is
   approved there, add a matching record on the **Templates** page here
   (name must match exactly) so campaigns can use it.

## Module map

| Module | What it does |
|---|---|
| Dashboard | Total contacts, active campaigns, sent/delivered/read/replies/failed, recent activity feed |
| Contacts | CRUD, groups, tags, search/filter, CSV import (`name,phone,email,tags`) |
| Campaigns | Create (template + group + send-now-or-schedule), history, pause/resume, duplicate |
| Inbox | WhatsApp-Web-style contact list + thread view, send individual replies |
| Bulk Replies | Filter contacts (by campaign + reply text + group), select, send one message to all |
| Campaign Analytics | Per-campaign sent/delivered/read/failed/replied counts and rates, on the campaign detail page |
| Templates | Local records of Meta-approved templates (name, category, language, variable count) |
| Message Logs | Every inbound/outbound message, filterable by campaign/contact/phone/date/status/template |
| Settings | WhatsApp API credentials, webhook verify token, business name |

## Notes on scaling this up for real commercial use

This is a single-tenant MVP built for speed:

- **SQLite** is fine for one business's traffic; if you need multiple
  businesses/users or heavier concurrent write load, move to Postgres —
  the SQL is plain enough that the migration is mostly swapping the
  connection layer in `database.py`.
- **No authentication yet** — anyone who can reach the server can use the
  API and UI. Add a login layer (e.g. FastAPI + sessions or OAuth) before
  exposing this beyond your own machine/VPN.
- **Sending is synchronous-in-background** (FastAPI `BackgroundTasks`), which
  is fine for hundreds of recipients per campaign but not for a serious
  bulk-sender at scale — for tens of thousands of recipients, move campaign
  sends to a real task queue (Celery/RQ) so a server restart can't drop a
  send in progress.
- **Rate limits**: WhatsApp Cloud API has its own throughput limits per
  phone number/tier — the campaign loop currently sends as fast as it can;
  add pacing/backoff if you hit 429s at higher volume.
