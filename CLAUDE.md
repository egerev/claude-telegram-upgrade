# Claude Telegram Upgrade — Setup Guide

You are helping the user set up and configure upgrades for Claude Code's Telegram plugin.

## What This Repo Contains

- `patches/zombie-fix.patch` — kills zombie bot processes (409 Conflict / 100% CPU fix)
- `patches/voice-transcription.patch` — auto-transcribes voice messages via local Whisper *(coming soon)*
- Multi-bot setup instructions (below)

## Setup Task

When the user asks to set up / install / configure, follow these steps in order. Ask for confirmation before making changes.

### Step 1: Find the Telegram Plugin

```bash
find ~/.claude/plugins/cache -path '*/telegram/*/server.ts' -type f 2>/dev/null
```

The plugin directory is the parent of `server.ts`. Store this path — you'll need it for applying patches.

### Step 2: Apply Patches

For each `.patch` file in the `patches/` directory of this repo:

```bash
# Dry run first
git -C "$PLUGIN_DIR" apply --check "/path/to/patches/zombie-fix.patch"

# Apply
git -C "$PLUGIN_DIR" apply "/path/to/patches/zombie-fix.patch"
```

If a patch fails with "already applied", skip it — it's already in place.

### Step 3: Multi-Bot Setup (optional)

Ask the user: "Do you want to set up separate Telegram bots for different projects?"

If yes:

1. Ask which project directories need their own bot
2. For each project:
   - Ask the user to create a bot via @BotFather and paste the token
   - Create the state directory:
     ```bash
     mkdir -p ~/.claude/channels/telegram-<project-name>
     ```
   - Save the token:
     ```bash
     echo "TELEGRAM_BOT_TOKEN=<token>" > ~/.claude/channels/telegram-<project-name>/.env
     chmod 600 ~/.claude/channels/telegram-<project-name>/.env
     ```
   - Create or update the project's `.claude/settings.local.json`:
     ```json
     {
       "env": {
         "TELEGRAM_STATE_DIR": "<full-path-to-state-dir>"
       }
     }
     ```
   - Remind the user to pair: send a message to the new bot, then run `/telegram:access pair <code>`

### Step 4: Voice Transcription Setup (optional)

Ask the user: "Do you want automatic voice message transcription?"

If yes, check what's available:

```bash
# Check for mlx-whisper (macOS Apple Silicon — best option)
which mlx_whisper 2>/dev/null && mlx_whisper --help 2>&1 | head -5

# Check for whisper-cpp (cross-platform fallback)
which whisper-cpp 2>/dev/null

# Check for ffmpeg (required for audio conversion)
which ffmpeg 2>/dev/null
```

**Install missing dependencies:**

- macOS Apple Silicon:
  ```bash
  pip3 install mlx-whisper   # GPU-accelerated Whisper
  brew install ffmpeg         # audio conversion
  ```
- macOS Intel / Linux:
  ```bash
  brew install whisper-cpp ffmpeg   # or: apt install whisper.cpp ffmpeg
  ```

**Download a Whisper model** (if not cached yet). Recommend `small` for multilingual (Russian + English), `base.en` for English-only:

- mlx-whisper downloads models automatically on first use (from `mlx-community/whisper-small`)
- whisper.cpp needs a manual download:
  ```bash
  mkdir -p ~/.local/share/whisper-models
  curl -L -o ~/.local/share/whisper-models/ggml-small.bin \
    "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin"
  ```

Then apply the voice transcription patch (same as Step 2).

### Step 5: Verify

1. Ask the user to restart Claude Code
2. Check that the plugin starts without errors:
   ```bash
   # Check for running telegram bot process
   ps aux | grep 'server.ts' | grep -v grep
   ```
3. Ask the user to send a test message from Telegram
4. If voice transcription is set up, ask them to send a voice message

## Troubleshooting

### "patch does not apply"
The official plugin may have updated. Check the version:
```bash
cat "$(find ~/.claude/plugins/cache -path '*/telegram/*/package.json' -type f 2>/dev/null | head -1)" | grep version
```
Patches are built for v0.0.4. If the version changed, the patches may need updating.

### Zombie processes still appearing
Check if the patch was applied:
```bash
grep -c "killOldInstance" "$(find ~/.claude/plugins/cache -path '*/telegram/*/server.ts' -type f 2>/dev/null | head -1)"
```
Should return `2` (function definition + call). If `0`, the patch wasn't applied.

### Multi-bot: wrong bot responding
Verify the project has the correct `TELEGRAM_STATE_DIR`:
```bash
cat .claude/settings.local.json
```
And that the `.env` in that directory has the right token.

## Important

- NEVER display or log bot tokens — they are secrets
- Always `chmod 600` on `.env` files
- Patches modify files inside `~/.claude/plugins/cache/` — plugin updates may overwrite them. Re-apply after updates.
- Multi-bot state dirs should follow the naming convention `~/.claude/channels/telegram-<name>`
