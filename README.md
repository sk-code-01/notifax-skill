# Notifax Skill

Send push notifications and full-screen alarms to your phone from any AI agent.

## Install

```bash
npx skills add siddhesh/notifax -g -y
```

## What it does

Once installed, your agent can:

- **Nudge** — send a standard push notification for routine updates (task done, deploy complete, results ready)
- **Alert** — trigger a full-screen alarm that wakes your phone, plays sound, and overrides Do Not Disturb — for urgent situations only

## Usage

Just tell your agent:

- "Notify me when the build finishes"
- "Send me a nudge when done"
- "Alert me if the tests fail"
- "Ping my phone with the results"

Or use the slash command:

```
/notifax
```

## How it works

1. Agent checks for `$NOTIFAX_API_KEY` in the environment
2. If not set, guides you through a one-time email verification to get your key
3. Sends the notification via the Notifax API → Firebase Cloud Messaging → your phone

## Authentication

Get an API key by verifying your email:

```bash
# Request a 6-digit code
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/auth/request-code \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'

# Verify the code you received
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/auth/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "code": "123456"}'
```

Then add to your shell profile:

```bash
export NOTIFAX_API_KEY="nfx_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

## Requirements

- `curl` (pre-installed on macOS and most Linux distributions)
- Notifax Android app installed with onboarding completed

## Plan Limits

| | Free |
|---|---|
| Nudges/day | 100 |
| Alerts/day | 20 |
