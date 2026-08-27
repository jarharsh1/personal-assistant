# Personal Assistant (Activepieces)

A self-hosted, fully autonomous personal assistant built on [Activepieces](https://www.activepieces.com)
(MIT-licensed, free to self-host with no usage limits) — no web app, no server code to
maintain. It runs in the background and pings you on Telegram when something needs your
attention:

- **New email arrives** in your Gmail inbox → Telegram notification with the sender,
  subject, and a concise one-sentence AI summary of what the email actually says.
- **A meeting is starting soon** on your Google Calendar (including Zoom meetings, since
  most Zoom invites land as calendar events) → Telegram notification with the title, time,
  and join link, so you can add/confirm it on your calendar yourself.
- **You ask it something** — message the bot "what meetings do I have this week?" and
  it replies on the spot, instead of only pushing at you (optional third flow, step 7
  below).

All three flows live inside one Activepieces container, built with its visual flow
editor — once enabled, nothing needs to run on your machine except that one container.

## Why Activepieces (not n8n)

n8n's self-hosted core is also free for personal use, but Activepieces' engine is fully
**MIT-licensed** (no fair-code caveats at all) and ships a "hobbyist" single-container
mode — no separate Postgres/Redis to run — which is the lightest self-hosted option for
a 2-flow personal tool like this one. Cost either way is whatever machine you run the
container on, not the software: even a free-tier cloud VM (e.g. Oracle Cloud's Always
Free tier) is enough for this workload.

## How it works

```
Gmail (new email) ──▶ OpenAI (summarize) ──▶ Telegram (send)

Schedule (5 min) ──▶ Google Calendar (list events) ──▶ Loop each event
                                                          ├─ Storage: already notified? skip
                                                          └─ Telegram (send) + Storage: mark notified

Telegram (your question) ──▶ chat ID == you? ──▶ Google Calendar (next 7 days) ──▶ OpenAI (answer) ──▶ Telegram (reply)
                                    │ no
                                    └─▶ ignored, no reply
```

- **Flow 1 — Email Notifier**: a Gmail trigger fires on new unread mail; an OpenAI step
  turns the subject/sender/body into one concrete sentence; a Telegram step sends it.
- **Flow 2 — Meeting Notifier**: a Schedule trigger runs every 5 minutes, pulls calendar
  events starting in the next 20 minutes, and for each one checks Activepieces' built-in
  **Storage** piece (a small persistent key/value store) to see if it's already been
  notified — so you get exactly one Telegram ping per meeting, roughly 10-15 minutes
  before it starts.
- **Flow 3 — Ask the Assistant** (optional): a Telegram trigger fires on any message to
  the bot, a condition checks the sender's chat ID matches yours (everyone else is
  silently ignored), then it pulls the next 7 days of calendar events and has OpenAI
  answer your question using them.

## Prerequisites

- Docker + Docker Compose, on any machine that stays on (your own box, a home server, or
  a small/free-tier VPS)
- A Google account (Gmail + Calendar) and a Google Cloud project with the **Gmail API**
  and **Google Calendar API** enabled
- A Telegram account
- An [OpenAI API key](https://platform.openai.com/api-keys) (used only for the
  one-sentence email summary — a small model like `gpt-4o-mini` costs a fraction of a
  cent per email)

## Setup

### 1. Start Activepieces

```bash
cp .env.example .env
docker compose up -d
```

Open http://localhost:8080 — the first account you create becomes the admin account
(this is local to your own instance, not a cloud sign-up).

### 2. Telegram bot setup

1. Message [@BotFather](https://t.me/BotFather) on Telegram, run `/newbot`, and copy the
   bot token.
2. Get your chat id: message your new bot anything, then open
   `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in a browser and read
   `message.chat.id` from the JSON response. Keep both handy for step 4.

### 3. Google Cloud project

Create a Google Cloud project, enable the **Gmail API** and **Google Calendar API**, and
create an OAuth 2.0 client (Activepieces' Gmail/Calendar connection screens show you the
exact redirect URI to add). Read-only scopes are enough (`gmail.readonly`,
`calendar.readonly`).

### 4. Add Connections in Activepieces

In the Activepieces UI: **Connections → New Connection**, and create one each for:

- **Gmail** (OAuth2 — sign in with your Google account)
- **Google Calendar** (OAuth2 — same Google account)
- **Telegram Bot** (paste the bot token from step 2)
- **OpenAI** (paste your API key)

### 5. Build the Email Notifier flow

**New Flow** →

1. **Trigger**: Gmail piece → *New Email* trigger. Set it to poll your inbox, unread
   only.
2. **Action**: OpenAI piece → chat/completion action. Model: `gpt-4o-mini` (cheap and
   plenty for this). Prompt:
   - System: `You write a single, concrete one-sentence summary of an email for a push notification. State the key fact or ask directly - no greeting, no preamble, no quotation marks, no "this email is about".`
   - User: the trigger's subject + sender + body, e.g. `Subject: {{subject}}\nFrom: {{from}}\n\n{{body}}` (use the step's variable picker to insert the actual Gmail trigger fields).
3. **Action**: Telegram piece → *Send Message* action. Chat ID: your chat id from step
   2. Text: combine sender, subject, and the OpenAI step's output text, e.g.:
   ```
   📧 New email
   From: {{from}}
   Subject: {{subject}}

   {{openai_summary}}
   ```
4. Turn the flow **On**.

### 6. Build the Meeting Notifier flow

**New Flow** →

1. **Trigger**: Schedule piece → every 5 minutes.
2. **Action**: Google Calendar piece → *List Events* action, time window = now to
   now + 20 minutes.
3. **Action**: Loop on Items piece → loop over the events list.
4. Inside the loop:
   - **Action**: Storage piece → *Get* (key = the event's ID) → check if it returns
     empty (not yet notified).
   - **Condition/Branch**: only continue if that value was empty AND the event starts
     within ~15 minutes.
   - **Action**: Telegram piece → *Send Message*, with the event title, start time, and
     a join link (Calendar's `hangoutLink`/`location`, or a Zoom URL pulled from the
     description).
   - **Action**: Storage piece → *Put* (key = event ID, value = `true`) so it's never
     sent twice.
5. Turn the flow **On**.

### 7. Build the Ask-the-Assistant flow (optional)

A third flow that answers questions on demand instead of only pushing notifications —
message the bot "what meetings do I have this week?" and it replies. **The security
gate in step 2 below is not optional** — skip it and anyone who finds your bot's
username can ask it your calendar.

**New Flow** →

1. **Trigger**: Telegram piece → *New Message* trigger.
2. **Condition/Branch**: only continue if the incoming message's chat ID equals your
   own chat id (from step 2 of Setup). Everything else dead-ends here with no reply —
   this is the whole access-control model for this flow, so don't skip it.
3. **Action**: Google Calendar piece → *List Events*, time window = now to now + 7 days.
4. **Action**: OpenAI piece → chat/completion action.
   - System: `You are a personal assistant. Answer the user's question using only the calendar events provided below. Be concise - a sentence or two. If the answer isn't in the data provided, say you don't have that information.`
   - User: the incoming Telegram message text, plus the calendar events from step 3
     (insert via the variable picker).
5. **Action**: Telegram piece → *Send Message*, replying to the same chat ID, with the
   OpenAI step's answer as the text.
6. Turn the flow **On**.

This answers from a fixed 7-day calendar window, not a real search — it's good for
"what's coming up" questions. Asking about something further out, or about old emails,
needs a wider window or an added Gmail search step, not just a rephrase.

### 8. Test it

- Send yourself an email → you should get a Telegram message within a minute or two.
- Create a calendar event starting in ~12 minutes → you should get a Telegram message
  before it starts.

## Customizing

- **Summary model**: `gpt-4o-mini` is the cost-conscious default; swap to `gpt-4o` in
  the OpenAI step if you want higher-quality summaries and don't mind the extra cost.
- **Change how often it checks**: edit the Gmail trigger's poll interval or the Schedule
  trigger's interval (currently every 5 minutes).
- **Filter which emails notify you**: use Gmail search syntax in the trigger's filter,
  e.g. `is:unread in:inbox -category:promotions`.
- **Change the meeting lookahead window**: adjust the Calendar step's time window and
  the "starts within X minutes" condition together.
- **Auto-add events to your calendar instead of just notifying**: add a Google Calendar
  *Create Event* action after the Telegram step in the meeting flow.
- **Watch Zoom directly instead of via Calendar**: if some Zoom meetings never land on
  your Google Calendar, swap the Calendar step for an HTTP request to the
  [Zoom API](https://developers.zoom.us/docs/api/) `GET /users/me/meetings` endpoint.

## Notes

- Everything here is self-hosted; your data doesn't pass through any third-party service
  other than Google's, OpenAI's, and Telegram's own APIs — the email body is sent to
  OpenAI only to generate the one-sentence summary.
- Your Google, Telegram, and OpenAI credentials live only inside Activepieces' own
  encrypted connection store (in the `activepieces_data` Docker volume) — never in
  `.env` or git.
