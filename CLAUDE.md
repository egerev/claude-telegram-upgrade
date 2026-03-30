# Claude Telegram Upgrade — Setup Guide

You are helping the user set up and configure upgrades for Claude Code's Telegram plugin.

## First Interaction

When the user opens this repo or asks to set up / install / configure, **start by explaining what you're going to do in plain language**. The user should understand the full picture before anything happens.

Say something like:

---

This is an upgrade pack for Claude Code's Telegram integration. Here's what I can set up for you:

1. **Zombie Process Fix** — the official Telegram plugin has a known bug where old bot processes don't get killed on restart, causing 409 errors and 100% CPU usage. I'll patch the plugin to automatically detect and kill stale processes.

2. **Voice Transcription** *(coming soon)* — right now, when you send a voice message to your bot, Claude just sees "(voice message)" and can't listen to the audio. This patch will auto-transcribe voice messages to text using Whisper running locally on your machine — no API keys, no cloud, everything stays private.

3. **Multi-Bot Setup** *(optional)* — if you work on multiple projects, you can have a separate Telegram bot for each one. Messages from different projects won't mix, and each bot has its own access list. I'll create the directories, save tokens, and wire up the per-project config.

All changes are patches on top of the official plugin. Nothing gets forked or replaced — you keep using the official plugin, just with fixes applied. If the official plugin updates, patches can be re-applied.

Shall I proceed? I'll go step by step and ask before making any changes.

---

Wait for the user's confirmation before proceeding.

## What This Repo Contains

- `patches/zombie-fix.patch` — kills zombie bot processes (409 Conflict / 100% CPU fix)
- `patches/voice-transcription.patch` — auto-transcribes voice messages via local Whisper - Multi-bot setup instructions (below)

## Setup Task

After the user confirms, follow these steps in order. Explain what each step does before doing it. Ask for confirmation before making changes.

### Step 1: Check the Official Telegram Plugin

```bash
find ~/.claude/plugins -path '*/telegram/server.ts' -o -path '*/telegram/*/server.ts' 2>/dev/null
```

Claude Code stores plugins in two locations: `cache/` (downloaded versions) and `marketplaces/` (source copies). The plugin may exist in one or both. **You must patch ALL copies** — Claude Code may run from either location.

If files are found — the plugin is installed. Store ALL parent directory paths.

If nothing is found, the official Telegram plugin is not installed. Help the user install it:

1. Explain: "The official Telegram plugin from Anthropic needs to be installed first. It's what these patches improve."
2. Check if `telegram@claude-plugins-official` is in `~/.claude/settings.json` under `enabledPlugins`:
   ```bash
   cat ~/.claude/settings.json
   ```
3. If not present, tell the user to add it:
   - They can run `/telegram:configure` in Claude Code
   - Or manually add `"telegram@claude-plugins-official": true` to the `enabledPlugins` section in `~/.claude/settings.json`
4. After enabling, they need to **restart Claude Code** so it downloads the plugin
5. Then re-run the search to find the plugin directory

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
