# Notifax — Agent Skill

You have access to Notifax: a service that sends notifications directly to the user's phone. Use it to keep the user informed about task completions, failures, or anything that needs their attention.

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

**Endpoint:** `POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify`
**Auth:** `Authorization: Bearer $NOTIFAX_API_KEY`

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
| `schedule_at` | string | — | ISO 8601 UTC datetime. If provided and in the future, delivery is deferred. |
| `snooze_minutes` | integer | — | Alert only. How long to wait before re-alarming if user snoozes. Default: 5. |

---

## Scheduling a Future Notification

Add `schedule_at` (ISO 8601 UTC) to defer delivery. The notification fires within 60 seconds of the target time. If `schedule_at` is absent or in the past, the notification fires immediately.

### Step 1 — Get the user's timezone and current time

**Always do this before scheduling.** You need the user's timezone to correctly convert relative expressions like "tomorrow at 9am" into an absolute UTC time.

```bash
curl -s https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/me \
  -H "Authorization: Bearer $NOTIFAX_API_KEY"
```

Response:
```json
{
  "email": "user@example.com",
  "timezone": "Asia/Kolkata",
  "server_time_utc": "2026-03-27T11:30:00Z",
  "device_count": 1
}
```

### Step 2 — Convert the user's time expression to UTC

Use `timezone` from the response above to interpret what the user said:

| User says | What to do |
|---|---|
| "in 30 minutes" | `server_time_utc + 30min` |
| "in 2 hours" | `server_time_utc + 2h` |
| "at 3pm" | today 15:00 in `timezone` → convert to UTC |
| "tomorrow at 9am" | next day 09:00 in `timezone` → convert to UTC |
| "tomorrow" | **Ask the user: "What time tomorrow?"** — do not guess |
| "next Monday" | **Ask the user: "What time on Monday?"** — do not guess |

**Never guess a time when the user gives only a date.** Always ask.

### Step 3 — Send with computed `schedule_at`

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "nudge",
    "title": "Standup in 5 minutes",
    "message": "Your daily standup starts at 10:00 AM.",
    "agent_name": "Claude Code",
    "schedule_at": "2026-03-27T04:25:00Z"
  }'
```

**Scheduled response:**
```json
{ "status": "scheduled", "id": "abc123", "scheduled_at": "2026-03-27T04:25:00Z" }
```

The user will see this in the Notifax app's **Upcoming** tab with a countdown until delivery.

### List pending scheduled notifications

```bash
curl -s https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/scheduled \
  -H "Authorization: Bearer $NOTIFAX_API_KEY"
```

### Cancel a scheduled notification

```bash
curl -s -X DELETE \
  https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/scheduled/<id> \
  -H "Authorization: Bearer $NOTIFAX_API_KEY"
```

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

## When to Send Notifications

**Only send a notification when the user has explicitly asked for one.** Do not send notifications automatically at the end of every task — this leads to spam and defeats the purpose.

Good reasons to send:
- The user said "notify me when X is done"
- The user said "alert me if this fails"
- The user asked you to schedule a reminder for a specific time

Do not send if the user just asked you to do a task and didn't mention notifications.

Example of explicit request: *"Run the tests and notify me when they finish."*

```bash
curl -s -X POST https://workspace-bizz--notifax-backend-fastapi-app.modal.run/api/v1/notify \
  -H "Authorization: Bearer $NOTIFAX_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"nudge\",
    \"title\": \"Tests passed\",
    \"message\": \"All 142 tests passed in 18s.\",
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
