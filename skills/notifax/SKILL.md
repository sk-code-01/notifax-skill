---
name: notifax
description: Send notifications directly to the user's phone via Notifax. Use it to keep the user informed about task completions, failures, or anything that needs their attention. Also triggers on "notify me when done", "send me a push notification", "alert me if", "ping my phone", "let me know when finished".
argument-hint: "[message]"
allowed-tools: Bash
---

# Notifax — Agent-to-Human Notifications

Send push notifications and full-screen alarms directly to the user's phone. Use it to keep them informed about task completions, failures, or anything that needs their attention.

Base URL: `https://workspace-bizz--notifax-backend-fastapi-app.modal.run`
Auth header: `Authorization: Bearer $NOTIFAX_API_KEY`

---

## Setup (one-time)

Check for the API key before sending anything:

```bash
echo $NOTIFAX_API_KEY
```

**If the variable is empty or unset**, walk the user through setup:

### Step 1 — Request a verification code

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/auth/request-code \
  -H "Content-Type: application/json" \
  -d '{"email": "<user_email>"}'
```

### Step 2 — Verify the code (user checks their email)

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/auth/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "<user_email>", "code": "<6_digit_code>"}'
```

Response:
```json
{ "apiKey": "nfx_xxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
```

### Step 3 — Save the key

```bash
export NOTIFAX_API_KEY="nfx_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Tell the user to add this line to their `~/.zshrc` or `~/.bashrc` so it persists across sessions.

**Setup is now complete. The user must have the Notifax Android app installed and have completed in-app onboarding for notifications to reach their phone.**

---

## Sending Notifications

### Send a Nudge (standard push notification)

Use for routine updates — task done, results ready, nothing urgent.

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "nudge",
    "title": "Deploy successful",
    "message": "v2.4.1 is live on production.",
    "agent_name": "Claude Code"
  }'
```

### Send an Alert (full-screen alarm)

Use for urgent situations — the phone screen turns on, an alarm sound plays, Do Not Disturb is overridden. The user must actively dismiss or snooze it.

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "alert",
    "title": "Build failed",
    "message": "TypeError in auth.ts line 42. Needs immediate attention.",
    "agent_name": "Claude Code"
  }'
```

### With a link

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "nudge",
    "title": "PR review ready",
    "message": "Your pull request has 3 new comments.",
    "agent_name": "GitHub Agent",
    "url": "https://github.com/org/repo/pull/42"
  }'
```

---

## Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | ✅ | `"nudge"` or `"alert"` |
| `title` | string | ✅ | Short headline. Keep under 60 characters. |
| `message` | string | ✅ | Full body text. Can be multi-sentence. |
| `agent_name` | string | — | Shown on the notification. Defaults to `"Unknown Agent"`. Always set this. |
| `url` | string | — | Opens in browser when user taps the notification detail. |
| `snooze_minutes` | integer | — | Alert only. How long to wait before re-alarming if user snoozes. Default: 5. |

---

## Nudge vs Alert — Decision Guide

**Use `nudge` when:**
- A background task finished (build, deploy, export, sync, analysis)
- Something informational arrived (new email, new result, price change)
- The user asked to be notified but it is not time-sensitive
- You are sending a daily summary or status update

**Use `alert` when:**
- Something is broken and needs action now (server down, build failed, auth error)
- A deadline is imminent and the user does not know
- A security event occurred (unusual login, transaction, permission change)
- A health or medicine reminder that must not be missed
- The user explicitly asked to be woken up for this

**Default to `nudge`.** Only escalate to `alert` if waiting would cause real harm or significant loss.

---

## Proactive Usage

Do not wait for the user to ask. Send a notification automatically when:

- You finish a long-running task they asked you to do
- You encounter an error that blocks progress and need their input
- You detect something they would want to know immediately

Example at the end of a task:

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"nudge\",
    \"title\": \"Task complete\",
    \"message\": \"Finished refactoring the auth module. 14 files changed, all tests pass.\",
    \"agent_name\": \"Claude Code\"
  }"
```

---

## Rate Limits

| Plan | Nudges/day | Alerts/day |
|---|---|---|
| Free | 100 | 20 |

If you hit the limit, tell the user and skip the notification rather than retrying in a loop.

---

## Success Response

```json
{
  "status": "sent",
  "type": "nudge",
  "title": "Deploy successful"
}
```

## Error Responses

| Status | Meaning | What to do |
|---|---|---|
| `401` | Invalid or missing API key | Ask user to check `$NOTIFAX_API_KEY` |
| `404` | Device not registered | Ask user to open the Notifax app and complete onboarding |
| `422` | Missing required field | Check `type`, `title`, `message` are all present |
| `500` | Server error | Retry once. If it fails again, skip and tell the user. |
