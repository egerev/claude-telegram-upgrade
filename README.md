# Telegram Zombie Fix for Claude Code

Fixes the **zombie process bug** in Claude Code's Telegram MCP plugin.

## The Problem

When Claude Code restarts or crashes, the old Telegram bot process can stay alive as a zombie. The new instance then gets `409 Conflict` errors from Telegram's `getUpdates` API because only one consumer is allowed per bot token. The built-in retry logic helps, but the zombie may linger for minutes.

## What the Patch Does

1. **PID file management** — on startup, writes `{pid, tokenHash, startedAt}` to `.server.pid` in the plugin's state directory
2. **Stale instance killer** — before starting, checks for a running process with the same bot token hash and kills it (SIGTERM → 2s grace → SIGKILL)
3. **Clean shutdown** — removes the PID file on graceful exit
4. **Awaited notification** — adds missing `await` to `mcp.notification()` call that could silently drop messages

Each bot token gets its own PID file (identified by a hash of the token), so multiple bots on the same machine don't interfere with each other.

## Installation

> Tested on macOS and Linux. Requires `git` and a working Claude Code installation.

### One-liner

```bash
cd "$(find ~/.claude / -path '*/external_plugins/telegram' -type d 2>/dev/null | head -1)/.." && git apply --check ~/Downloads/telegram-zombie-fix.patch && git apply ~/Downloads/telegram-zombie-fix.patch
```

### Step by step

**1. Find the plugin directory**

```bash
# The Telegram plugin typically lives here:
ls ~/.claude/external_plugins/telegram/server.ts

# If not found, search for it:
find ~ -path '*/external_plugins/telegram/server.ts' 2>/dev/null
```

**2. Download the patch**

```bash
curl -LO https://raw.githubusercontent.com/egerev/telegram-zombie-fix/main/telegram-zombie-fix.patch
```

**3. Preview changes (dry run)**

```bash
cd /path/to/external_plugins/..   # parent of external_plugins/
git apply --check telegram-zombie-fix.patch
```

**4. Apply**

```bash
git apply telegram-zombie-fix.patch
```

**5. Restart Claude Code** — the fix takes effect on next Telegram plugin startup.

### Reverting

```bash
cd /path/to/external_plugins/..
git apply --reverse telegram-zombie-fix.patch
```

## How It Works

```
Startup
  │
  ├─ Read .server.pid
  │   └─ Same token hash & process alive?
  │       ├─ Yes → SIGTERM → wait 2s → SIGKILL
  │       └─ No  → continue
  │
  ├─ Write new .server.pid (pid + tokenHash)
  │
  └─ Start bot polling (no more 409 conflicts)

Shutdown
  │
  └─ Remove .server.pid
```

## Details

- **Token isolation**: uses SHA-256 hash of the bot token to identify instances. Different bots never kill each other.
- **Graceful first**: sends SIGTERM and waits 2 seconds before SIGKILL.
- **PID file format**: JSON with `pid`, `tokenHash`, and `startedAt` fields, stored with `0600` permissions.

## License

MIT
