# Personal Assistant

Self-hosted automation (no app, no server code) that watches Gmail and Google Calendar,
summarizes with OpenAI, and notifies you on Telegram — plus an optional flow that
answers questions on demand. Built on [Activepieces](https://www.activepieces.com)
(MIT-licensed, free to self-host), single Docker container.

## Architecture

![Two flows converge on one Telegram send step: Gmail trigger to OpenAI summarize to Telegram, and Schedule to Get Calendar events to Loop-and-dedup to Telegram](docs/images/two-flows-one-hub.png)

| Flow | Trigger | Steps |
|---|---|---|
| **Email** | New unread Gmail | Gmail trigger → OpenAI (summarize) → Telegram send |
| **Meetings** | Every 5 min | Schedule → Get Calendar events (next 20 min) → Loop + Storage dedup → Telegram send |

## Ask on demand (optional)

![A Telegram message passes through a chat ID gate; on match it fetches the next 7 days of calendar events, has OpenAI answer using them, and replies on Telegram; on mismatch it's ignored](docs/images/ask-on-demand.png)

Message the bot a question ("what meetings do I have this week?") and it replies.
**The chat-ID gate is not optional** — without it, anyone who finds your bot can query
your calendar.

## Stack

| Piece | Used for |
|---|---|
| [Activepieces](https://www.activepieces.com), self-hosted | Runs all flows, single container, no Postgres/Redis needed |
| Gmail + Google Calendar (OAuth2, read-only) | Trigger + data source |
| Telegram Bot API | Notifications + the ask-on-demand channel |
| OpenAI (`gpt-4o-mini`) | Email summaries and Q&A answers |

## Setup

Track progress with [`SETUP.md`](SETUP.md) — the same steps below, as a checklist.

1. **Start it**: `cp .env.example .env && docker compose up -d` → open `localhost:8080`,
   create your admin account.
2. **Telegram bot**: [@BotFather](https://t.me/BotFather) → `/newbot` → copy the token.
   Message your bot once, then open `https://api.telegram.org/bot<TOKEN>/getUpdates` to
   read your chat id from the JSON.
3. **Google Cloud**: enable the Gmail API + Calendar API, create an OAuth client
   (Activepieces shows you the exact redirect URI when you add the connection).
4. **Connections** (Activepieces UI → Connections): add Gmail, Google Calendar, Telegram
   Bot, and OpenAI.
5. **Build the flows** in the visual editor, matching the diagrams above:

   **Email** — Gmail trigger (new email, unread) → OpenAI action → Telegram send:
   ```
   System: You write a single, concrete one-sentence summary of an email for a push
   notification. State the key fact or ask directly - no greeting, no preamble, no
   quotation marks, no "this email is about".
   User:   Subject: {{subject}}
           From: {{from}}

           {{body}}
   ```

   **Meetings** — Schedule (5 min) → Get Calendar events (next 20 min) → Loop over
   events → inside the loop: Storage *Get*(event id) → skip if already set → Telegram
   send → Storage *Put*(event id, true).

   **Ask** (optional) — Telegram trigger (new message) → condition: chat id == yours,
   else stop → Get Calendar events (next 7 days) → OpenAI answers the incoming message
   using those events → Telegram reply.

6. Turn the flows **On**. Test: email yourself, create an event ~12 min out, message
   the bot.

## Notes

- Secrets (OAuth tokens, bot token, API key) live only in Activepieces' encrypted
  Connections store — never in `.env` or git.
- Meeting source is Google Calendar, not the Zoom API directly — covers most Zoom
  invites since they carry a join link as a calendar event. Swap in a direct Zoom API
  call if some of yours don't land on your calendar.
- The Ask flow answers from a fixed 7-day window, not a real search — fine for "what's
  coming up," not for searching further back.
