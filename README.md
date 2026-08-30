# Personal Assistant

Automation (no app, no server code) that watches Gmail and Google Calendar, summarizes
with OpenAI, and notifies you on Telegram — plus an optional flow that answers questions
on demand. Built on [Activepieces](https://www.activepieces.com), either as a hosted
account (no install, nothing to run) or self-hosted (MIT-licensed, single Docker
container).

## Architecture

![Two flows converge on one Telegram send step: Gmail trigger to OpenAI summarize to Telegram, and Schedule to Get Calendar events to Loop-and-dedup to Telegram](docs/images/two-flows-one-hub.png)

| Flow | Trigger | Steps |
|---|---|---|
| **Email** | New unread Gmail | Gmail trigger → OpenAI (classify) → Code (format + build request) → Custom API Call (Telegram `sendMessage`, with buttons) |
| **Meetings** | Every 5 min | Schedule → Get Calendar events (next 20 min) → Loop + Storage dedup → Telegram send |

## Ask on demand (optional)

![A Telegram message passes through a chat ID gate; on match it fetches the next 7 days of calendar events, has OpenAI answer using them, and replies on Telegram; on mismatch it's ignored](docs/images/ask-on-demand.png)

Message the bot a question ("what meetings do I have this week?") and it replies.
**The chat-ID gate is not optional** — without it, anyone who finds your bot can query
your calendar.

## Stack

| Piece | Used for |
|---|---|
| [Activepieces](https://www.activepieces.com) | Runs all flows — cloud account or self-hosted single container |
| Gmail + Google Calendar (OAuth2, read-only) | Trigger + data source |
| Telegram Bot API | Notifications + the ask-on-demand channel |
| OpenAI (`gpt-4o-mini`) | Email summaries and Q&A answers |

## Setup

Track progress with [`SETUP.md`](SETUP.md) — the same steps below, as a checklist.

1. **Start it** — pick one:
   - **Cloud (no install, no commands)**: sign up at [cloud.activepieces.com](https://cloud.activepieces.com). Free tier covers several active flows, which is all this needs.
   - **Self-hosted (needs a machine where you can run Docker commands)**: `cp .env.example .env && docker compose up -d` → open `localhost:8080`, create your admin account.
2. **Telegram bot**: [@BotFather](https://t.me/BotFather) → `/newbot` → copy the token.
   Message your bot once, then open `https://api.telegram.org/bot<TOKEN>/getUpdates` to
   read your chat id from the JSON.
3. **Google account access**:
   - **Cloud**: skip straight to step 4 — Activepieces Cloud has its own pre-registered
     Google OAuth app, so adding the Gmail/Calendar connections just pops up a normal
     Google sign-in window. No Google Cloud Console needed.
   - **Self-hosted**: you bring your own OAuth client — enable the Gmail API + Calendar
     API in a Google Cloud project, then create an OAuth client using the redirect URI
     Activepieces shows you when you add the connection.
4. **Connections** (Activepieces UI → Connections): add Gmail, Google Calendar, Telegram
   Bot, and OpenAI.
5. **Build the flows** in the visual editor, matching the diagrams above:

   **Email** — 4 steps, in this exact order (order matters — a step can only use data
   from steps *before* it):

   1. **Gmail trigger**: New Email, unread only.
   2. **OpenAI action** (model `gpt-4o-mini`) — classifies the email and extracts
      structured fields instead of writing a generic sentence. Single "Question" field:
      ```
      You triage an email for a push notification. Read it and respond with ONLY a JSON object — no markdown, no code fences, no explanation — matching exactly this shape:
      {"category":"action_required|calendar|fyi|digest_low_priority","title":"...","org":"...","summary":"...","action":"..."}

      Rules:
      - category: "action_required" if the sender wants you to do something (upload, pay, respond, approve, sign, complete). "calendar" if it's a meeting/event invite. "digest_low_priority" if it's promotional/marketing/newsletter content. "fyi" for anything else purely informational.
      - title: a short, specific, human title — do not just copy the subject line.
      - org: the sender's organization or person name, cleaned up, no email address.
      - summary: one concrete, specific sentence stating the actual content or ask.
      - action: if category is action_required, a short imperative next step. Otherwise an empty string.

      Email:
      Subject: {{subject}}
      From: {{from}}

      {{body}}
      ```
   3. **Code action** — parses that JSON (falling back to plain text if the model ever
      returns something unparseable), renders a category-coded message (emoji + caps
      label + bold title + org + summary + action), and builds the **entire Telegram API
      request body** as one JSON string — including a 3-button inline keyboard (Open
      Email / Remind Me / Mark Done). Inputs: `from`, `subject`, `messageId` (Gmail
      trigger fields), `summary` (the OpenAI step's raw JSON output):
      ```javascript
      export const code = async (inputs) => {
        const esc = (s) => String(s ?? '')
          .replace(/&/g, '&amp;')
          .replace(/</g, '&lt;')
          .replace(/>/g, '&gt;');

        const rawId = String(inputs.messageId ?? '').replace(/[<>]/g, '');
        const gmailLink = rawId
          ? `https://mail.google.com/mail/u/0/#search/rfc822msgid%3A${encodeURIComponent(rawId)}`
          : null;

        let data;
        try {
          const cleaned = String(inputs.summary ?? '')
            .replace(/```json/gi, '')
            .replace(/```/g, '')
            .trim();
          data = JSON.parse(cleaned);
        } catch (e) {
          data = null;
        }

        if (!data || !data.summary) {
          data = {
            category: 'fyi',
            title: inputs.subject,
            org: inputs.from,
            summary: String(inputs.summary ?? '').trim() || '(no summary available)',
            action: ''
          };
        }

        const headers = {
          action_required: ['⚠️', 'ACTION REQUIRED'],
          calendar: ['📅', 'CALENDAR'],
          digest_low_priority: ['📥', 'LOW PRIORITY'],
          fyi: ['✉️', 'NEW EMAIL'],
        };
        const [icon, label] = headers[data.category] || headers.fyi;

        let message = `${icon} <b>${label}</b>\n`;
        message += `<b>${esc(data.title || inputs.subject)}</b>\n`;
        message += `${esc(data.org || inputs.from)}\n\n`;
        message += esc(data.summary);
        if (data.action) message += `\n\n⏰ <b>Next:</b> ${esc(data.action)}`;
        if (gmailLink) message += `\n\n<a href="${gmailLink}">Open in Gmail</a>`;

        const shortId = rawId.slice(0, 50);
        const buttons = [[{ text: '📧 Open Email', url: gmailLink || 'https://mail.google.com' }]];
        if (shortId) {
          buttons[0].push({ text: '⏰ Remind Me', callback_data: `remind:${shortId}` });
          buttons[0].push({ text: '✅ Mark Done', callback_data: `done:${shortId}` });
        }

        const requestBody = JSON.stringify({
          chat_id: '<YOUR_CHAT_ID>',
          text: message,
          parse_mode: 'HTML',
          reply_markup: { inline_keyboard: buttons }
        });

        return { message, requestBody };
      };
      ```
      Replace `<YOUR_CHAT_ID>` with your own chat id from Setup step 2.
   4. **Custom API Call** (Telegram Bot piece) — not the packaged "Send Text Message"
      action. Method `POST`, URL `/sendMessage`, Header `Content-Type: application/json`,
      Body Type `JSON`, and the JSON Body field = *only* the Code step's `requestBody`
      output (a single inserted variable, nothing else typed around it).

   **Why the detour through Custom API Call?** The packaged "Send Text Message" action's
   Reply Markup field doesn't actually forward inline keyboards to Telegram in this piece
   version — buttons silently never appear, with no error. Hitting Telegram's
   `sendMessage` endpoint directly via Custom API Call (auth still handled automatically
   by the existing Connection) sends the exact same request Telegram's docs describe, and
   the buttons work.

   **Why not `MarkdownV2` for the text formatting?** It requires escaping ~20 reserved
   characters (`. - ( ) ! ...`) in *any* dynamic text — email content can contain any of
   them, so it breaks constantly. `HTML` mode only reserves `& < >`, which the Code step
   escapes safely.

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
  Connections store — never in `.env` or git. On the cloud plan that store is hosted by
  Activepieces; self-hosted, it's in your own `activepieces_data` Docker volume.
- Meeting source is Google Calendar, not the Zoom API directly — covers most Zoom
  invites since they carry a join link as a calendar event. Swap in a direct Zoom API
  call if some of yours don't land on your calendar.
- The Ask flow answers from a fixed 7-day window, not a real search — fine for "what's
  coming up," not for searching further back.
- The email flow's **"Remind Me" and "Mark Done" buttons currently do nothing when
  tapped** — they render correctly, but nothing is listening for the button press yet.
  Making them functional needs a second flow (Telegram `callback_query` trigger →
  branch on the button pressed → act) — not built yet.
