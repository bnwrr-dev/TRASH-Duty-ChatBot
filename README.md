# Trash Duty Bot — Automated Task Rotation for Shared Housing

A LINE chatbot that automates weekly trash-duty rotation for shared households.  
It assigns responsibilities, sends reminders, and maintains a fair rotation system directly within a group chat.

**Tech Stack**

Backend: Node.js 20+, Express, @line/bot-sdk  
Database: SQLite (better-sqlite3, WAL mode)  
Scheduling: node-cron (minute-level automation)  
Deployment: Railway (Docker-based cloud hosting)

## Problem

In shared housing environments, managing recurring chores often relies on manual reminders and informal coordination.  
This can lead to confusion, missed tasks, and uneven workload distribution.

This project was created to automate task scheduling and ensure fair responsibility sharing among household members.

## Key Features

- Automated weekly task assignment
- Reminder notifications for incomplete tasks
- Flexible rotation management
- Support for multiple group chats
- Persistent cloud deployment
- Command-based interaction system

## What it does

- Automatically assigns weekly responsibilities every Thursday at 09:00.
- If the task is not completed, the system sends a reminder on Monday at 19:00.
- Inline buttons on the assignment message: **Done**, **Not Free**, **Not at Home**.
- "Not Free" passes the week to the next active member; the skipper rotates to the back of the line so everyone still does one duty per full cycle.
- "Not at Home" marks the member inactive (skipped in future rotations). They return with `back`.
- "Done" is undoable for 24 hours (configurable) via a button on the completion message.
- Supports multiple LINE groups; state is isolated per group.
- Full audit history available via `history`.

## Real Usage

This bot is currently deployed in a live shared housing environment and manages weekly trash duty scheduling for household members.
It runs continuously in the cloud and automatically handles task assignment and reminders without manual coordination.

## Setup Guide (For Developers)

1. Log into [LINE Developers Console](https://developers.line.biz/).
2. Create a **Provider** (or reuse one), then create a **Messaging API** channel.
3. On the channel page:
   - Copy **Channel secret** → `LINE_CHANNEL_SECRET`.
   - In the **Messaging API** tab, issue a **Channel access token (long-lived)** → `LINE_CHANNEL_ACCESS_TOKEN`.
   - Disable **Auto-reply messages** and **Greeting messages** (they conflict with the bot).
   - Enable **Allow bot to join group chats**.
   - Set the **Webhook URL** to `https://<your-domain>/webhook` and enable **Use webhook**.
4. On your phone, scan the channel QR code to add the bot as a friend, then invite it into the group you want to manage.

## Development

Run the following commands to start the bot locally:

```bash
cp .env.example .env
# edit .env: paste LINE_CHANNEL_ACCESS_TOKEN and LINE_CHANNEL_SECRET

npm install
npm run migrate       # initializes ./data/trashbot.db
npm run dev           # starts on PORT (default 3000) with --watch
```

Expose the port with a tunnel during development so LINE can reach your webhook:

```bash
# pick one
cloudflared tunnel --url http://localhost:3000
ngrok http 3000
```

Set the LINE channel's Webhook URL to `<tunnel-host>/webhook` and click **Verify**.

## Deployment

The bot is deployed to Railway, a cloud hosting platform that provides automatic scaling and persistent storage.

1. Push this repo to GitHub.
2. In Railway: **New Project → Deploy from GitHub repo**, select this repo.
3. **Variables** tab: set `LINE_CHANNEL_ACCESS_TOKEN`, `LINE_CHANNEL_SECRET`, and any overrides (`DEFAULT_TZ`, `ASSIGN_HOUR`, etc).
4. **Settings → Volumes**: create a volume mounted at `/app/data` so the SQLite file survives deploys.
5. **Settings → Networking**: generate a public domain. Use `https://<domain>/webhook` as the LINE Webhook URL.
6. `healthcheckPath` is already set to `/healthz`.

### Render (alternative)

Use the same Dockerfile. Create a **Web Service** from the repo, add an env group with the LINE variables, and attach a **Disk** mounted at `/app/data`. Set `DATABASE_PATH=/app/data/trashbot.db`. Point the LINE webhook at `https://<service>.onrender.com/webhook`.

## In-group usage

The bot is controlled through simple text commands directly in the LINE group.

After adding the bot to a group:

```
setup                    # first caller becomes admin; optionally: setup Asia/Tokyo
join                     # every participating member types this
rotation Anna, Ben, Cara # admin sets order (or just run `assign` to use join-order)
assign                   # manually trigger this week (normally automatic on Thursday)
```

Reference: type `help` in the group for the full command list.

### Command reference

| Command | Who | What |
|---|---|---|
| `help` | Anyone | Show all commands |
| `setup [tz]` | First caller | Initialize group; caller becomes admin. Default tz from `DEFAULT_TZ`. |
| `join` | Anyone | Register yourself as a rotation member |
| `leave` | Anyone | Mark yourself inactive |
| `back` | Anyone | Return from inactive to active |
| `status` | Anyone | Show this week's assignment |
| `schedule` | Anyone | Show full rotation order and pointer |
| `members` | Anyone | List members and status |
| `history` | Anyone | Show last 12 tasks |
| `rotation A, B, C` | Admin | Set rotation order by member display names |
| `force <name>` | Admin | Force-reassign this week to a specific member |
| `remove <name>` | Admin | Remove a member |
| `reset` | Admin | Reset rotation pointer to start |
| `pause` / `resume` | Admin | Pause or resume scheduled assignments |
| `assign` | Admin | Run this week's assignment job now |

Commands also work with a leading `/`, e.g. `/status`.

## Configuration

These environment variables control scheduling behavior, database location, and runtime configuration.

| Env var | Default | Notes |
|---|---|---|
| `LINE_CHANNEL_ACCESS_TOKEN` | — | Required |
| `LINE_CHANNEL_SECRET` | — | Required |
| `PORT` | `3000` | |
| `DATABASE_PATH` | `./data/trashbot.db` | Use `/app/data/trashbot.db` in Docker |
| `DEFAULT_TZ` | `Asia/Taipei` | IANA timezone; override per-group via `setup <tz>` |
| `ASSIGN_HOUR` / `ASSIGN_MINUTE` | `9` / `0` | Thursday assignment time in group TZ |
| `REMIND_HOUR` / `REMIND_MINUTE` | `19` / `0` | Monday reminder time in group TZ |
| `UNDO_WINDOW_HOURS` | `24` | How long a Done click can be undone |
| `LOG_LEVEL` | `info` | pino log level |

## System Architecture

The system follows an event-driven architecture using LINE webhooks.

```
LINE Platform
     │ webhook
     ▼
 /webhook ──► webhook.handleEvent ──► commands.handle  (text → admin/user cmds)
                                 └──► postbacks.handle (Done / Not Free / Not at Home / Undo)

rotation.js   — core assignment + skip/reassign + overdue logic
weekUtil.js   — Thursday-anchored week key in group timezone
scheduler.js  — minutely tick that fires Thursday 09:00 and Monday 19:00 per group TZ
models/*.js   — thin SQL wrappers over better-sqlite3
```

## Rotation semantics (important)

This logic ensures fairness across all members in the rotation.

**Not Free (this-week-only pass)**

Example with rotation `[A, B, C, D]`:

- Week 1: assigned to A. A clicks Not Free → B assigned as replacement.
  - Rotation pointer moves to B.
- Week 2: C. Week 3: D. Week 4: A.
- Over a full cycle everyone still does exactly one duty — A's slot was effectively moved to the back of the line.

This is the fair interpretation; if your household wants different semantics (e.g. A does next week regardless), the rule lives in `services/rotation.js` in `skipAndReassign`.

**Not at Home**

Marks the user inactive. If they're the current assignee, it triggers the same reassignment flow. They stay in the rotation table (position preserved) and re-enter on `back`.

**Overdue**

If a week turns over with a task still `pending`, the Thursday assignment job marks the prior task `overdue` and creates a fresh assignment. Admin can also run `force <name>` at any time to override.

## Operations

These operational practices ensure system reliability and data safety in production environments.

- **Backups.** The SQLite file is at `DATABASE_PATH`. For Railway volumes, snapshot regularly or add a daily `sqlite3 backup` job. For disaster recovery, copy the `.db` file while the app is running — SQLite WAL mode makes this safe.
- **Logs.** Structured JSON via pino. In dev, pretty-printed. Ship to your log backend of choice.
- **Healthcheck.** `GET /healthz` returns `{ok: true}`.
- **Signature validation.** The LINE middleware rejects tampered webhooks with HTTP 401.

## Security & privacy

- Only LINE user IDs, display names, and rotation state are stored — no PII beyond what LINE provides.
- Admin is whoever first ran `setup` in a given group; other admin-only commands reject non-admins.
- The `/webhook` endpoint validates `X-Line-Signature` with HMAC-SHA256 against `LINE_CHANNEL_SECRET`.

## Known Limitations

- LINE groups cap at 500 members, but the PRD limits rotation to 20; no enforcement in code — add if needed.
- Reminder sends are per-group push messages, which count against your LINE channel's monthly push quota.
- Minute-resolution scheduler: assignments fire within 60s of the target minute.

## Future Improvements

- Support multiple household tasks
- Custom reminder scheduling
- Admin dashboard
- Usage analytics
- Web configuration interface
