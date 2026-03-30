# Claude Telegram Upgrade

Community upgrade pack for Claude Code's [Telegram plugin](https://github.com/anthropics/claude-plugins-official) (v0.0.4). Patches apply on top of the official plugin — no fork, no divergence, easy to update.

## Patches

| Patch | What it fixes |
|-------|---------------|
| [zombie-fix](patches/zombie-fix.patch) | Kills stale bot processes that cause 409 Conflict errors and 100% CPU |
| voice-transcription *(coming soon)* | Transcribes voice messages to text via local Whisper (mlx-whisper / whisper.cpp) |

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/egerev/claude-telegram-upgrade.git
cd claude-telegram-upgrade

# 2. Find your plugin directory
PLUGIN_DIR=$(find ~/.claude/plugins/cache -path '*/telegram/*/server.ts' -type f 2>/dev/null | head -1 | xargs dirname)
echo "Plugin found at: $PLUGIN_DIR"

# 3. Apply all patches
for p in patches/*.patch; do
  git -C "$PLUGIN_DIR" apply --check "$(pwd)/$p" && \
  git -C "$PLUGIN_DIR" apply "$(pwd)/$p" && \
  echo "Applied: $p"
done

# 4. Restart Claude Code
```

### Reverting

```bash
for p in patches/*.patch; do
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

### Voice Transcription *(coming soon)*

Currently, voice messages arrive as `(voice message)` with a `.oga` file that Claude can't listen to. This patch will auto-transcribe them using local Whisper:

- **macOS (Apple Silicon):** [mlx-whisper](https://github.com/ml-explore/mlx-examples/tree/main/whisper) — runs on GPU, fast
- **Linux / Intel Mac:** [whisper.cpp](https://github.com/ggerganov/whisper.cpp) — CPU-based fallback
- Converts `.oga` → `.wav` via `ffmpeg`, transcribes, sends text to Claude
- No API keys needed — everything runs locally

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

- Claude Code with Telegram plugin v0.0.4+
- `git` (to apply patches)
- `ffmpeg` (for voice transcription patch)
- `mlx-whisper` or `whisper-cpp` (for voice transcription patch)

## License

Patches: MIT. Original plugin: Apache 2.0 (Anthropic).
