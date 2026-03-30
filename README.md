# Claude Telegram Upgrade

Community upgrade pack for Claude Code's [Telegram plugin](https://github.com/anthropics/claude-plugins-official) (v0.0.4). Patches apply on top of the official plugin — no fork, no divergence, easy to update.

## Patches

| Patch | What it fixes |
|-------|---------------|
| [zombie-fix](patches/zombie-fix.patch) | Kills stale bot processes that cause 409 Conflict errors and 100% CPU |
| [voice-transcription](patches/voice-transcription.patch) | Transcribes voice messages to text via local Whisper (mlx-whisper / whisper.cpp) |
| [all](patches/all.patch) | Both patches combined — apply this one if you want everything |

> **Note:** Apply `zombie-fix` first, then `voice-transcription`. Or just use `all.patch` for both at once.

## Prerequisites

These patches require the **official Telegram plugin** for Claude Code. If you don't have it yet:

```bash
# Open Claude Code and enable the Telegram plugin
claude
# Then run:
/telegram:configure
```

Or add it manually to `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "telegram@claude-plugins-official": true
  }
}
```

Then restart Claude Code so it downloads the plugin. You'll also need a Telegram bot token from [@BotFather](https://t.me/BotFather).

## Quick Start

The easiest way: clone and let Claude Code do everything for you.

```bash
git clone https://github.com/egerev/claude-telegram-upgrade.git
cd claude-telegram-upgrade
claude
```

Then just say **"set it up"**. Claude Code will read the included `CLAUDE.md`, explain what it's going to do, and walk you through the entire setup:

- Check that the official Telegram plugin is installed (and help you install it if not)
- Apply all patches to the official plugin
- Configure separate Telegram bots for different projects (optional)
- Install voice transcription dependencies — `ffmpeg`, `mlx-whisper` (macOS) or `whisper-cpp` (Linux), download the Whisper model (optional)
- Verify everything works

All interactive, step by step. No need to read docs or run commands manually — Claude Code handles everything.

### Manual install

If you prefer to do it yourself:

```bash
# 1. Clone this repo
git clone https://github.com/egerev/claude-telegram-upgrade.git
cd claude-telegram-upgrade

# 2. Find ALL plugin locations (Claude Code may use cache/ or marketplaces/)
PLUGIN_DIRS=$(find ~/.claude/plugins -path '*/telegram/server.ts' -o -path '*/telegram/*/server.ts' 2>/dev/null | xargs -I{} dirname {} | sort -u)
echo "Found plugin at: $PLUGIN_DIRS"

# 3. Apply patches to every copy
# Note: list patches explicitly to control order. all.patch is an alternative
# that applies both at once (see patches/all.patch).
for dir in $PLUGIN_DIRS; do
  echo "Patching: $dir"
  for p in patches/zombie-fix.patch patches/voice-transcription.patch; do
    git -C "$dir/../.." apply --check "$(pwd)/$p" 2>/dev/null && \
    git -C "$dir/../.." apply "$(pwd)/$p" && \
    echo "  Applied: $p"
  done
done

# 4. Restart Claude Code
```

### Reverting

```bash
for p in patches/voice-transcription.patch patches/zombie-fix.patch; do
  git -C "$PLUGIN_DIR" apply --reverse "$(pwd)/$p" 2>/dev/null && echo "Reverted: $p"
done
```

## Patches in Detail

### Zombie Fix

The official plugin retries on 409 errors and does graceful shutdown ([PR #825](https://github.com/anthropics/claude-plugins-official/pull/825)), but **does not kill zombie processes**. The old `bun` process stays alive, burns CPU in a retry loop, and blocks the new instance. Multiple community PRs ([#903](https://github.com/anthropics/claude-plugins-official/pull/903), [#1070](https://github.com/anthropics/claude-plugins-official/pull/1070), [#1136](https://github.com/anthropics/claude-plugins-official/pull/1136)) were closed — the repo only accepts Anthropic contributions.

Open issues: [#947](https://github.com/anthropics/claude-plugins-official/issues/947), [#934](https://github.com/anthropics/claude-plugins-official/issues/934), [#1049](https://github.com/anthropics/claude-plugins-official/issues/1049), [#1146](https://github.com/anthropics/claude-plugins-official/issues/1146).

**This patch adds:**
- PID file with token hash — on startup, finds and kills the old process (SIGTERM → 2s → SIGKILL)
- Clean removal on shutdown
- Missing `await` on `mcp.notification()` that could silently drop messages
- Token isolation — different bots never interfere with each other

```
Startup → read .server.pid → same token & alive? → kill it → write new PID → start polling
Shutdown → remove .server.pid
```

### Voice Transcription

Without this patch, voice messages arrive as `(voice message)` with a `.oga` file that Claude can't listen to. With the patch, voice messages are **automatically transcribed to text** before reaching Claude.

**How it works:**
1. Downloads the `.oga` voice file from Telegram
2. Converts to `.wav` via `ffmpeg` (16kHz mono)
3. Transcribes using locally installed Whisper
4. Sends `[voice]: <transcribed text>` to Claude instead of `(voice message)`
5. Falls back to `(voice message)` if transcription fails

**Supported backends:**
- **macOS (Apple Silicon):** [mlx-whisper](https://github.com/ml-explore/mlx-examples/tree/main/whisper) — runs on GPU, fast
- **Linux / Intel Mac:** [whisper.cpp](https://github.com/ggerganov/whisper.cpp) — CPU-based fallback

**Configuration** (env vars in `.env` or project settings):
- `TELEGRAM_WHISPER_MODEL` — model name or path (default: `mlx-community/whisper-small-mlx`)
- `TELEGRAM_WHISPER_LANGUAGE` — language hint, e.g. `ru` (optional, auto-detect if omitted)

No API keys needed — everything runs locally.

## Multi-Bot Setup

Run different Telegram bots for different projects — each Claude Code session gets its own bot.

### How it works

The plugin reads `TELEGRAM_STATE_DIR` env var to determine where to store state (tokens, access, inbox). By default it's `~/.claude/channels/telegram`. Override it per-project to isolate bots.

### Setup

**1. Create a second bot** via [@BotFather](https://t.me/BotFather) on Telegram.

**2. Create a state directory** for the new bot:

```bash
mkdir -p ~/.claude/channels/telegram-myproject
```

**3. Save the token:**

```bash
echo "TELEGRAM_BOT_TOKEN=<your-token>" > ~/.claude/channels/telegram-myproject/.env
chmod 600 ~/.claude/channels/telegram-myproject/.env
```

**4. Configure the project.** In your project's `.claude/settings.local.json`:

```json
{
  "env": {
    "TELEGRAM_STATE_DIR": "/Users/<you>/.claude/channels/telegram-myproject"
  }
}
```

**5. Start Claude Code** in the project directory — it picks up the project-level setting and connects to the right bot.

**6. Pair your Telegram account** — send any message to the new bot, then run `/telegram:access pair <code>` in Claude Code.

### Result

```
~/.claude/channels/
├── telegram/              ← default bot (personal projects)
│   ├── .env               ← TELEGRAM_BOT_TOKEN=...
│   ├── access.json
│   └── inbox/
├── telegram-myproject/    ← project-specific bot
│   ├── .env
│   ├── access.json
│   └── inbox/
└── telegram-work/         ← another project
    ├── .env
    ├── access.json
    └── inbox/
```

Each bot is fully isolated — own token, own access list, own message inbox, own PID file (zombie fix handles this correctly).

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with the official Telegram plugin enabled
- `git` (to apply patches)
- `ffmpeg` (for voice transcription patch)
- `mlx-whisper` or `whisper-cpp` (for voice transcription patch)

## After Plugin Updates

Claude Code may auto-update the Telegram plugin, which overwrites patches. If voice transcription or zombie fix stops working after an update, re-apply:

```bash
cd ~/claude-telegram-upgrade
PLUGIN_DIRS=$(find ~/.claude/plugins -path '*/telegram/server.ts' -o -path '*/telegram/*/server.ts' 2>/dev/null | xargs -I{} dirname {} | sort -u)
for dir in $PLUGIN_DIRS; do
  echo "Patching: $dir"
  for p in patches/zombie-fix.patch patches/voice-transcription.patch; do
    git -C "$dir/../.." apply --check "$(pwd)/$p" 2>/dev/null && \
    git -C "$dir/../.." apply "$(pwd)/$p" && \
    echo "  Applied: $p"
  done
done
```

## Security Considerations

- **`--dangerously-skip-permissions`** disables all permission prompts in Claude Code. Claude can read, write, and execute anything on the server without asking. Only use this on a server you control and trust.
- **Telegram access = code execution.** Anyone who can send messages to your paired bot can instruct Claude to run arbitrary commands. Use a private bot and keep the bot token secret. Do not share bot tokens or add untrusted users to the allowlist.
- **Personal servers only.** This setup is designed for single-user personal servers. Do not run on shared infrastructure or expose the bot publicly.
- **Token storage.** The OAuth token is stored in `~/.claude/.env` with `chmod 600` permissions (owner read/write only). The bot token is stored in `~/.claude/channels/telegram-<project>/.env`, also `chmod 600`.

## License

Patches: MIT. Original plugin: Apache 2.0 (Anthropic).
