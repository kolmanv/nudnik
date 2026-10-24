# nudnik

A personal assistant that nags you about what is waiting for you. Every morning a background Claude run reads
WhatsApp (and later Gmail), writes a short briefing of who and what is waiting for you, keeps a list of open tasks
that grow more urgent as they age, and sends the briefing to you on Telegram.

**It is read-only by design.** It has no tool that can send, reply, delete or change anything anywhere. The only
thing that leaves the machine is the briefing, sent by the shell script (not by Claude) to one fixed Telegram chat.

## How it stays safe

An assistant that reads private messages is exposed to the "lethal trifecta": private data, untrusted input
(anyone can message you) and a way to send data out. nudnik removes the third leg, and does it in layers that
don't depend on the model behaving:

| Layer | What it does |
|---|---|
| Read-only where it reads strangers | The WhatsApp gateway has no send tools; the morning briefing gets read tools only |
| Google writes ask you | Only on your request in the chat. Sending mail shows a Telegram Allow/Deny prompt (`ask` in `chat/settings.json`); other Google changes run directly, and Docs and Drive keep version history; Gmail is authorised without settings or permanent delete |
| A fixed tool list | Scheduled runs get only file tools and the read tools listed in `bin/nudnik`; no shell, no web |
| No account connectors | `ENABLE_CLAUDEAI_MCP_SERVERS=false` and `--strict-mcp-config`: only the servers in your own `mcp.json` |
| Fixed recipient | Telegram sending is a `curl` in the script, to the chat id in your config; Claude can't choose a recipient |
| Chat limits | Tools allowed in `chat/settings.json` run; denied ones (shell, web, your config) never run; anything else asks you on Telegram. It can read this folder, edit tasks, read WhatsApp and Google, and reply to you |
| One Telegram user | The plugin's allowlist drops messages from anyone else, and replies only go to allowed chats |
| Its own machine | Run it in its own container or VM with no access to your home network |

Message and mail text is treated as data, never as instructions (see `CLAUDE.md`).

## Setup

1. A Linux user with `claude` logged in, plus `git jq curl`. Systemd with user lingering for the timer.
2. `git clone https://github.com/kolmanv/nudnik ~/nudnik && ~/nudnik/bin/nudnik init`
   creates `~/.config/nudnik/` from the examples. Then fill in, all mode 600 and never committed:
   - `owner.env`: your name, language, timezone, briefing time, Telegram chat id, what you care about
   - `telegram.env`: `TELEGRAM_BOT_TOKEN=...` from @BotFather
   - `mcp.json`: the read-only MCP servers (see `mcp.json.example`)
3. Run `bin/nudnik init` again to install the timer, then `NOTIFY=0 bin/nudnik briefing` to try it without sending.
4. To reply to nudnik in Telegram: `claude plugin install telegram@claude-plugins-official`, then
   `claude plugin disable telegram@claude-plugins-official` (it is enabled only for the chat session, by
   `chat/settings.json`), install [Bun](https://bun.sh), and create, mode 600:
   - `~/.claude/channels/telegram/.env`: the same `TELEGRAM_BOT_TOKEN=...`
   - `~/.claude/channels/telegram/access.json`:
     `{"dmPolicy": "allowlist", "allowFrom": ["<your Telegram user id>"], "groups": {}, "pending": {}}`
   Then `bin/nudnik init` once more starts the chat session (tmux `chat`).
5. Google (optional): [workspace-mcp](https://github.com/taylorwilsdon/google_workspace_mcp) in a venv at
   `~/.local/share/workspace-mcp` (`pip install workspace-mcp==2.1.2`), started by Claude over stdio with
   `--permissions gmail:send drive:full calendar:full sheets:full docs:full` (use `--read-only --tools gmail drive
   calendar sheets docs` for read-only). Your own OAuth client (Google Cloud project with the Gmail,
   Drive, Docs, Sheets and Calendar APIs; Web client, redirect `http://localhost:8000/oauth2callback`; app published so
   the sign-in lasts) in `~/.config/nudnik/google.env` (`GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`,
   `USER_GOOGLE_EMAIL`, `WORKSPACE_MCP_CREDENTIALS_DIR`). Sign in once by calling its `start_google_auth` tool and
   opening the callback address on the same machine. Then set `MAIL_TOOLS` in `owner.env`.

## Use

| Command | What |
|---|---|
| `bin/nudnik briefing` | collect, update tasks, write `briefing/<date>.md`, send it to Telegram (`NOTIFY=0` to skip sending) |
| `bin/nudnik chat` | start the Telegram chat session in tmux `chat` (normally started by systemd) |
| `bin/nudnik status` | regenerate and print `TASKS.md` (open) and `ARCHIVE.md` (closed) |
| `bin/nudnik send <file>` | send a text file to your Telegram chat |
| `journalctl --user -u nudnik` | logs of the scheduled runs |

If the run fails, a one-line "briefing failed" message is sent instead, so silence never means "all fine".

## Files

- `CLAUDE.md`: the rules every run follows (read-only, untrusted input, task format, priorities, aging)
- `prompts/briefing.md`: the morning prompt
- `owner.env.example`, `mcp.json.example`: templates for your private config
- Outputs (git-ignored): `briefing/`, `tasks/<area>/<task>/task.md`, `TASKS.md`, `ARCHIVE.md`, `state/`

Inspired by the structure of an org-work assistant: tasks as files, aging, close only with evidence, no side effects.

## Privacy

nudnik is a personal assistant that each owner runs on their own server for themselves. When the owner connects
Google (Gmail, Drive, Docs, Sheets, Calendar), access is used only to answer the owner, to write the owner's own
briefings and task lists, and to make the changes the owner asks for in their own chat. Data read from Google stays on the owner's server and in the owner's Claude
sessions; it is not sold, shared with anyone else or used for advertising. The owner can revoke access at any time at
https://myaccount.google.com/permissions.
