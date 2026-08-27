# Setup Checklist

Six segments, in order. Each is independently testable before moving to the next —
details and prompts for each are in the [README](./README.md).

## 1. Infra

- [ ] `cp .env.example .env`
- [ ] `docker compose up -d`
- [ ] Open `localhost:8080`, create your admin account

## 2. Credentials

- [ ] Telegram bot created via [@BotFather](https://t.me/BotFather), token saved
- [ ] Your chat id read from `https://api.telegram.org/bot<TOKEN>/getUpdates`
- [ ] Gmail API + Calendar API enabled in Google Cloud, OAuth client created
- [ ] Connections added in Activepieces: Gmail, Google Calendar, Telegram Bot, OpenAI

## 3. Email flow

- [ ] Gmail trigger (new email, unread only)
- [ ] OpenAI action (summarize prompt from README)
- [ ] Telegram send action
- [ ] Flow turned **On**
- [ ] Tested: emailed myself → got a Telegram message

## 4. Meeting flow

- [ ] Schedule trigger (every 5 min)
- [ ] Get Calendar events action (next 20 min)
- [ ] Loop over events
- [ ] Inside loop: Storage *Get* (dedup check)
- [ ] Inside loop: Telegram send
- [ ] Inside loop: Storage *Put* (mark notified)
- [ ] Flow turned **On**
- [ ] Tested: created an event ~12 min out → got exactly one Telegram message

## 5. Ask-on-demand flow (optional)

- [ ] Telegram trigger (new message)
- [ ] Condition: chat id == yours, else stop
- [ ] Get Calendar events action (next 7 days)
- [ ] OpenAI answer action
- [ ] Telegram reply action
- [ ] Flow turned **On**
- [ ] Tested: asked a question from my own chat → got an answer
- [ ] **Security-tested**: a message from a different chat ID gets no reply

## 6. Test & verify

- [ ] Every flow built above produces the expected Telegram message
- [ ] Chat-ID gate confirmed blocking (segment 5, if built)
- [ ] Poll intervals / lookahead windows reviewed and tuned to taste
