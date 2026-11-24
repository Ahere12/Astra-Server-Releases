# Discord bot

Astra Server can run a Discord bot while the app is open (also when it's minimised to the tray). The bot answers
slash commands and posts notifications. It uses the same engine as the app's buttons — there is no second
server-management system — and it never runs anything from Discord as a Windows command. Nothing is installed as a
Windows service: when you quit Astra Server, the bot goes offline.

## 1. Create the bot (once)
1. Open https://discord.com/developers/applications → **New Application** → name it (e.g. "Astra Server").
2. **Bot** → **Reset Token** → copy the token.
3. Leave all **Privileged Gateway Intents** off — the bot doesn't need them.
4. In Astra Server: bell button at the bottom left (**Discord bot**) → paste the token → **Save & connect**.
   The token is checked with Discord, then stored encrypted with Windows' per-user protection
   (`%APPDATA%\AstraServer\discord-token.json`). It is never shown again, never sent to the app's window,
   never written to logs, and never part of the project files.

## 2. Invite it
Press **Invite the bot to your Discord server** (appears after the token is saved) and pick your server.
It asks only for these permissions (permission number 84992):
- View Channels, Send Messages, Embed Links, Read Message History
- and the `applications.commands` scope for slash commands.
It does not ask for Administrator, Manage Server, or any moderation permission.

## 3. Configure
- **Discord server & channel**: pick the server, and the channel where notifications go. The slash commands are
  registered in that server right away (Register again if you ever need to).
- **Who can use the bot**:
  - Read-only commands (`status`, `list`, `players`, `health`): everyone in the Discord server, or only authorized people.
  - **Authorized people / roles**: only they can `start`, `stop`, `restart`, `backup` and see `logs`.
    Add people by their Discord user ID (Discord → Settings → Advanced → Developer Mode, then right-click a
    person → Copy User ID), or tick roles. **With nobody authorized, nobody can control any server.**
- **Minecraft servers**: "Discord can control it" is **off for every server** until you turn it on.
  "Notifications" can be turned off per server.
- **Notifications**: pick events and the CPU / PC-memory thresholds. Press **Send a test notification**.

## Commands
| Command | Who | What |
|---|---|---|
| `/server status <server>` | read access | status, version, uptime, players, TPS, memory |
| `/server list` | read access | all servers with status and players |
| `/server players <server>` | read access | who is online |
| `/server health <server>` | read access | the Health check's must-fix / should-fix problems |
| `/server start <server>` | authorized + server allowed | starts it |
| `/server stop <server>` | authorized + server allowed | asks for a **Confirm** button (only the same person, 60 s), then stops safely (world saved) |
| `/server restart <server>` | authorized + server allowed | same confirmation, then restarts |
| `/server backup <server>` | authorized + server allowed | makes a backup |
| `/server logs <server>` | authorized | last 20 console lines, only visible to you; IPs, addresses, passwords/keys and your user folder removed |

There is no command to delete backups, read or send files, or run console commands.

## Safety rules (built in)
- Only the configured Discord server is answered; DMs and other servers are refused.
- Missing member information → refused (fails closed). Permissions are checked again when a confirmation is pressed.
- Rate limits: 5 commands per person per 30 seconds; a server can be controlled once per 15 seconds.
- Messages never ping anyone (`@everyone` in a player name can't ping).
- Discord's own rate limits are respected (waits and retries on HTTP 429).

## Notifications
Server started, stopped, crashed, **restarted** (one message instead of "stopped" + "started"), player joined/left,
backup completed/failed, high CPU (this server above the threshold for 2 minutes), high memory (the whole PC above
the threshold for 2 minutes), scheduled restart warnings (when players are warned 1 minute before), and health
problems (new must-fix problems, checked every 15 minutes and after a crash).

Against spam: joins/leaves within 20 seconds are one message; CPU and memory at most every 30 minutes; health once
until the problems change; repeats within 5 seconds are dropped; messages are sent one at a time with a short pause.
Joins/leaves and "backup completed" are off by default.

The old per-server webhook (server → Discord tab) still works and is independent of the bot.

## How it was tested
- `test/discord.test.js` (part of `npm test`) runs the bot against `tools/fake-discord.js`, a local stand-in that
  speaks Discord's REST API and Gateway protocol: invalid tokens, connecting, command registration, authorized and
  unauthorized users and roles, other servers and DMs, confirmations (wrong person, expired, cancel, forged), rate
  limits, cleaned logs, grouped notifications and cooldowns, a 429 from Discord, a network drop (reconnect + resume),
  Astra restart (reconnects by itself), disconnect, and a token revoked while connected (stops, no retry loop).
- `tools/smoke-discord.js` drives a **real Paper server** through the fake Discord: start, status, player
  notifications, backup, confirmed restart ("restarted"), a crash, stop, logs, and Astra restarting.
- **Not yet tested against the real Discord** — that needs your bot token. When you connect it, try
  `/server list`, then `/server status`, then a start/stop on a test server.
