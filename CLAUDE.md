# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run the bot
uv run python -m d_brain

# Lint
uv run ruff check src/
uv run ruff format src/

# Type check
uv run mypy src/

# Tests
uv run pytest

# Run a single test file
uv run pytest tests/path/to/test_file.py

# Install dependencies
uv sync

# Manual daily processing (cron alternative)
./scripts/process.sh

# Weekly digest
uv run python scripts/weekly.py

# Vault graph analysis
cd vault && uv run .claude/skills/graph-builder/scripts/analyze.py
```

## Architecture

This is a Telegram bot (`aiogram` 3.x) that acts as a voice-first personal second brain. The overall data flow is:

```
Telegram message → Bot handlers → VaultStorage (daily/*.md)
                                → SessionStore (vault/.sessions/*.jsonl)
/process or cron → ClaudeProcessor → claude CLI subprocess → MCP (Todoist)
                                   → VaultStorage (thoughts/, goals/)
                                   → VaultGit (git commit + push)
                → HTML report → Telegram message
```

### Key Architectural Decisions

**ClaudeProcessor uses `claude --print --dangerously-skip-permissions` as a subprocess.** This is intentional — the bot itself is the outer shell, and all "smart" work (classifying entries, creating Todoist tasks, building Obsidian links) is delegated to a Claude Code subprocess that runs with full vault access. The processor loads skill content from `vault/.claude/skills/dbrain-processor/SKILL.md` and injects it directly into the prompt because `@vault/` references don't work in `--print` mode.

**Vault is the single source of truth.** All bot interactions land in `vault/daily/YYYY-MM-DD.md` as `## HH:MM [type]` entries before any processing happens. Processing is always a separate step.

**Router registration order matters** (`bot/main.py`). The `do.py` router must come before `voice.py` and `text.py` so FSM state (`DoCommandState.waiting_for_input`) catches the follow-up message first. `text.py` is a catch-all and must be last.

**MCP config** lives at `mcp-config.json` in the project root. Claude subprocesses are started from the project root with `--mcp-config` pointing there, giving them access to `mcp__todoist__*` tools.

**Vault has its own Claude context** at `vault/.claude/CLAUDE.md`. When Claude Code is invoked from `vault/` as cwd (which `ClaudeProcessor` and `scripts/process.sh` do), it reads that file for vault-specific rules, agent definitions, and skills.

### Module Layout

```
src/d_brain/
  config.py          # Pydantic Settings — loads .env
  bot/
    main.py          # Bot/Dispatcher factory + auth middleware
    states.py        # FSM states (DoCommandState)
    handlers/        # One router per message type
      commands.py    # /start /help /status
      process.py     # /process → ClaudeProcessor.process_daily()
      do.py          # /do → ClaudeProcessor.execute_prompt() with FSM
      weekly.py      # /weekly → ClaudeProcessor.generate_weekly()
      voice.py       # Deepgram transcription → VaultStorage
      text.py        # Catch-all text → VaultStorage
      photo.py       # Download + VaultStorage (attachments/)
      forward.py     # Forwarded messages → VaultStorage
      buttons.py     # Reply keyboard shortcuts
    formatters.py    # HTML sanitizer/truncator for Telegram (4096 char limit)
    keyboards.py     # Reply keyboard layout
  services/
    processor.py     # ClaudeProcessor — runs claude CLI subprocess
    storage.py       # VaultStorage — reads/writes vault/daily/ and attachments/
    session.py       # SessionStore — JSONL append-only at vault/.sessions/
    transcription.py # DeepgramTranscriber — Nova-3, Russian, async
    git.py           # VaultGit — commit + push vault after processing
```

### Vault Layout (runtime data)

```
vault/
  daily/YYYY-MM-DD.md      # Raw entries, append-only during the day
  goals/                   # Goal cascade (3y → yearly → monthly → weekly)
  thoughts/                # Processed notes (ideas/, reflections/, projects/, learnings/)
  summaries/               # Weekly digests (YYYY-WXX-summary.md)
  attachments/YYYY-MM-DD/  # Photos
  MOC/                     # Maps of Content
  MEMORY.md                # Long-term curated memory
  .sessions/               # JSONL session logs per user_id
  .claude/                 # Claude Code context for vault operations
    CLAUDE.md              # Vault-specific Claude instructions
    skills/dbrain-processor/  # Main processing skill + references
    agents/                # Specialized agent definitions
    rules/                 # Format rules (daily, thoughts, goals, telegram)
```

### Environment Variables (`.env`)

| Variable | Required | Description |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Yes | Bot token from @BotFather |
| `DEEPGRAM_API_KEY` | Yes | Deepgram Nova-3 transcription |
| `TODOIST_API_KEY` | No | Passed to claude subprocess env |
| `VAULT_PATH` | No | Default `./vault` |
| `ALLOWED_USER_IDS` | No | JSON list e.g. `[123456789]` |
| `ALLOW_ALL_USERS` | No | Override user whitelist (security risk) |

### Telegram Report Format

All Claude subprocess output is expected to be raw Telegram HTML (not markdown). `formatters.py` sanitizes it to the allowed tag set: `<b>`, `<i>`, `<code>`, `<pre>`, `<a>`, `<s>`, `<u>`. The 4096-character limit is enforced with tag-balanced truncation.

### Deployment

Runs as a `systemd` service (`d-brain-bot.service`). Daily processing fires via `d-brain-process.timer` at 21:00 using `scripts/process.sh` (bash version) which handles PATH setup for systemd and sends results directly via Telegram Bot API curl. `scripts/weekly.py` is the Python equivalent for weekly digests.
