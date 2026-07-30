# Configuration, Toolsets & Voice

Edit with `hermes config edit` or `hermes config set section.key value`.
Full reference: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

### Config Sections (most-used keys)

| Section | Key options |
|---------|-------------|
| `model` | `default`, `provider`, `base_url`, `api_key`, `context_length`, `aliases` |
| `agent` | `max_turns` (90), `tool_use_enforcement`, `service_tier`, `verify_on_stop` |
| `terminal` | `backend` (local/docker/ssh/modal/daytona/singularity), `cwd`, `timeout` (180) |
| `compression` | `enabled`, `threshold` (0.50), `target_ratio` (0.20) |
| `display` | `skin`, `interface` (cli/tui), `language`, `show_reasoning`, `show_cost`, `pet` |
| `approvals` | `mode` (smart/manual/off), `timeout`, `cron_mode` |
| `stt` | `enabled`, `provider` (local/groq/openai/mistral/elevenlabs/deepinfra) |
| `tts` | `provider` (edge/elevenlabs/openai/minimax/mistral/neutts/gemini/piper/kittentts/deepinfra/xai) |
| `memory` | `memory_enabled`, `user_profile_enabled`, `provider`, `write_approval` |
| `security` | `redact_secrets`, `tirith_enabled`, `website_blocklist` |
| `delegation` | `model`, `provider`, `max_concurrent_children`, `max_iterations` (50), `max_spawn_depth` |
| `checkpoints` | `enabled`, `max_snapshots` (50) |
| `curator` | `enabled`, `consolidate` (false, opt-in aux-model consolidation), `interval_hours`, `stale_after_days` |

`hermes config check` reports sections missing from an older config.

### Messaging final-result-only mode

Use this when a messaging surface (Discord, Telegram, Slack, etc.) is showing
gateway execution logs, tool progress, interim assistant commentary, reasoning
summaries, or repeated "still working" status messages and the user wants only
the final answer.

Prefer per-platform overrides when the user only complains about one surface;
use global settings when they want every gateway surface quiet. The gateway
resolves `display.platforms.<platform>.<key>` first, then global `display.<key>`,
then platform defaults. YAML may store bare `off` as boolean `false`; Hermes'
gateway display resolver normalizes `false` to tool-progress `off`, so the
warning from `hermes config set` can be ignored or avoided with `--force`.

Discord-only quiet/final mode:

```bash
hermes config set display.platforms.discord.tool_progress off --force
hermes config set display.platforms.discord.interim_assistant_messages false
hermes config set display.platforms.discord.long_running_notifications false
hermes config set display.platforms.discord.busy_ack_detail false
hermes config set display.platforms.discord.show_reasoning false
```

All messaging surfaces quiet/final mode:

```bash
hermes config set display.tool_progress off --force
hermes config set display.interim_assistant_messages false
hermes config set display.long_running_notifications false
hermes config set display.busy_ack_detail false --force
hermes config set display.show_reasoning false
```

Verification:

```bash
python - <<'PY'
from pathlib import Path
import os, yaml
home = Path(os.environ.get('HERMES_HOME', '~/.hermes')).expanduser()
cfg = yaml.safe_load((home / 'config.yaml').read_text()) or {}
display = cfg.get('display') or {}
print('global:', {k: display.get(k) for k in (
    'tool_progress', 'interim_assistant_messages',
    'long_running_notifications', 'busy_ack_detail', 'show_reasoning')})
print('discord:', (display.get('platforms') or {}).get('discord'))
PY
```

Notes:

- `display.interim_assistant_messages=false` suppresses assistant commentary
  emitted alongside tool calls before the final answer.
- `display.long_running_notifications=false` suppresses periodic heartbeat/status
  messages while a long turn is running.
- `display.busy_ack_detail=false` keeps busy-turn acknowledgements terse; for
  silent steering acknowledgements specifically, set
  `display.busy_steer_ack_enabled=false`.
- If a surface still shows chatter, inspect platform-specific overrides before
  changing global settings.

### Toolsets

Enable/disable via `hermes tools` (interactive) or `hermes tools enable/disable NAME`.
Full enumeration: `TOOLSETS` dict in `toolsets.py` (`_HERMES_CORE_TOOLS` is the default bundle most platforms inherit).

| Toolset | What it provides |
|---------|-----------------|
| `web` / `search` | Web search + extraction / search-only subset |
| `browser` | Browser automation (Browserbase, Camofox, or local Chromium) |
| `terminal` | Shell commands and process management |
| `file` | File read/write/search/patch |
| `code_execution` | Sandboxed Python execution |
| `coding` | Code-editing helpers (LSP-backed) |
| `computer_use` | Desktop GUI control (cua-driver) |
| `vision` | Image analysis |
| `image_gen` | Image generation and image-to-image editing |
| `video` / `video_gen` | Video analysis / video generation |
| `x_search` | X (Twitter) search (X OAuth or API key) |
| `tts` | Text-to-speech |
| `skills` | Skill browsing and management |
| `memory` | Persistent cross-session memory |
| `session_search` | Search past conversations |
| `context_engine` | Pluggable context-engine hooks |
| `project` | Named multi-folder workspace tools |
| `delegation` | Subagent task delegation |
| `cronjob` | Scheduled task management |
| `clarify` | Ask user clarifying questions |
| `todo` | In-session task planning |
| `kanban` | Multi-agent work-queue tools (gated to workers) |
| `debugging` | Extra introspection tools (off by default) |
| `safe` | Minimal low-risk toolset for locked-down sessions |
| `spotify`, `homeassistant`, `discord`, `discord_admin`, `feishu_doc`, `feishu_drive`, `yuanbao` | Service integrations (gated on their credentials) |

Tool changes take effect on `/reset` (new session) — never mid-conversation, to preserve prompt caching.

## Voice

### STT (Voice → Text)

Voice messages from messaging platforms are auto-transcribed.

```yaml
stt:
  enabled: true
  provider: local   # local (faster-whisper, free) | groq | openai | mistral | elevenlabs | deepinfra
  local:
    model: base     # tiny, base, small, medium, large-v3
```

Auto-detect priority: local faster-whisper (`pip install faster-whisper`) → Groq (`GROQ_API_KEY`, free tier) → OpenAI (`VOICE_TOOLS_OPENAI_KEY`) → Mistral Voxtral (`MISTRAL_API_KEY`).

### TTS (Text → Voice)

| Provider | Env var | Free? |
|----------|---------|-------|
| Edge TTS (default) | None | Yes |
| ElevenLabs | `ELEVENLABS_API_KEY` | Free tier |
| OpenAI | `VOICE_TOOLS_OPENAI_KEY` | Paid |
| MiniMax | `MINIMAX_API_KEY` | Paid |
| Mistral | `MISTRAL_API_KEY` | Paid |
| Gemini | `GOOGLE_API_KEY` | Free tier |
| NeuTTS / Piper / KittenTTS (local) | None | Free |

Voice commands: `/voice on` (voice-to-voice), `/voice tts` (always voice), `/voice off`.
