# Setup Checklist

Six segments, in order. Each is independently testable before moving to the next —
details and prompts for each are in the [README](./README.md). Each segment also has a
[GitHub issue](https://github.com/jarharsh1/personal-assistant/issues/1) with the same
checklist — closing it (see workflow below) ticks the tracking issue's progress bar
automatically, no manual bookkeeping needed.

## Workflow

For each segment:

1. `git checkout main && git pull origin main`
2. `git checkout -b segment/<n>-<slug>` (e.g. `segment/1-infra`)
3. Do the work, commit
4. `git push -u origin segment/<n>-<slug>`
5. Open a PR into `main`. In the PR description, include `Closes #<issue-number>` (see
   table below) so merging auto-closes that segment's issue.
6. Merge the PR, then `git pull origin main` before starting the next segment.

| Segment | Branch | Issue |
|---|---|---|
| 1. Infra | `segment/1-infra-cloud` | [#2](https://github.com/jarharsh1/personal-assistant/issues/2) |
| 2. Credentials | `segment/2-credentials` | [#3](https://github.com/jarharsh1/personal-assistant/issues/3) |
| 3. Email flow | `segment/3-email-flow` | [#4](https://github.com/jarharsh1/personal-assistant/issues/4) |
| 4. Meeting flow | `segment/4-meeting-flow` | [#5](https://github.com/jarharsh1/personal-assistant/issues/5) |
| 5. Ask-on-demand flow | `segment/5-ask-flow` | [#6](https://github.com/jarharsh1/personal-assistant/issues/6) |
| 6. Test & verify | `segment/6-verify` | [#7](https://github.com/jarharsh1/personal-assistant/issues/7) |

## 1. Infra

Pick one:

**Cloud** (no install, no commands — use this if you can't run Docker on your machine):
- [x] Signed up at [cloud.activepieces.com](https://cloud.activepieces.com)
- [x] Confirmed the dashboard loads, Connections and Flows are reachable

**Self-hosted** (needs a machine where you can run Docker commands):
- [ ] `cp .env.example .env`, `AP_WORKER_TOKEN` filled in
- [ ] `docker compose up -d`
- [ ] Open `localhost:8080`, create your admin account

## 2. Credentials

- [x] Telegram bot created via [@BotFather](https://t.me/BotFather), token saved
- [x] Your chat id read from `https://api.telegram.org/bot<TOKEN>/getUpdates`
- [ ] Google Cloud OAuth client created — **self-hosted only**; skip on Cloud (its own
      Google OAuth app handles this via a sign-in popup)
- [x] Gmail connection added in Activepieces
- [ ] Google Calendar connection added
- [ ] Telegram Bot connection added
- [ ] OpenAI connection added

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
