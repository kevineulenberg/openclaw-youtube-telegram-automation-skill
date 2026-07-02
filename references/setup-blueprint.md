# Setup Blueprint

Use this reference to implement the automation on a fresh OpenClaw VPS with generic example data.

## Workspace Layout

Recommended server paths:

```text
/home/openclaw/.openclaw/workspace/
  scripts/
    youtube_telegram_post.py
  youtube-api-agent/
    .venv/
    data/
      channels.json
      seen-videos.json
      telegram-post-state.json
      youtube-blogger/
      vault-mirror/
    src/
      transcripts.py
      paths.py
      youtube.py
```

Recommended env file:

```text
/home/openclaw/.config/youtube-telegram.env
```

## Env Template

Use placeholders only:

```bash
TELEGRAM_BOT_TOKEN=REPLACE_ME
TELEGRAM_CHAT_ID=-1001234567890
YOUTUBE_AGENT_DIR=/home/openclaw/.openclaw/workspace/youtube-api-agent
OPENCLAW_BIN=/home/openclaw/.nvm/versions/node/v22.22.1/bin/openclaw
YOUTUBE_RSS_LIMIT=15
YOUTUBE_STANDARD_UPLOAD_LIMIT=15
TRANSCRIPT_RETENTION_DAYS=14
RAPIDAPI_YT_TRANSCRIPT_KEYS=REPLACE_ME,REPLACE_ME
RAPIDAPI_YT_HOST=yt-api.p.rapidapi.com
RAPIDAPI_YT_BASE_URL=https://yt-api.p.rapidapi.com
```

Set strict permissions:

```bash
chmod 600 /home/openclaw/.config/youtube-telegram.env
```

## Channel Config

Example:

```json
[
  {
    "channel": "@ExampleFinanceChannel",
    "enabled": true,
    "discoveryMode": "rss"
  },
  {
    "channel": "@ExampleMacroChannel",
    "enabled": true,
    "discoveryMode": "standard_uploads"
  },
  {
    "channel": "@ExampleCryptoChannel",
    "enabled": true,
    "discoveryMode": "standard_uploads"
  }
]
```

## Python Dependencies

Install in the agent venv:

```bash
cd /home/openclaw/.openclaw/workspace/youtube-api-agent
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install youtube-transcript-api yt-dlp
```

The orchestrator may use only Python standard library plus the agent venv.

## systemd Units

Service:

```ini
[Unit]
Description=YouTube Telegram post automation

[Service]
Type=oneshot
EnvironmentFile=/home/openclaw/.config/youtube-telegram.env
WorkingDirectory=/home/openclaw/.openclaw/workspace/scripts
ExecStart=/home/openclaw/.openclaw/workspace/youtube-api-agent/.venv/bin/python /home/openclaw/.openclaw/workspace/scripts/youtube_telegram_post.py
TimeoutStartSec=30min
```

The service may run from `workspace/scripts`, so the orchestrator must not write
project files with process-CWD-relative paths. Keep paths stored in state as
relative values such as `data/youtube-blogger/...`, but write and read them as
`Path(YOUTUBE_AGENT_DIR) / relative_path`.

Timer:

```ini
[Unit]
Description=Run YouTube Telegram post automation twice daily

[Timer]
OnCalendar=*-*-* 08:30:00 Australia/Brisbane
OnCalendar=*-*-* 20:30:00 Australia/Brisbane
Persistent=true

[Install]
WantedBy=timers.target
```

Deploy to:

```text
/home/openclaw/.config/systemd/user/youtube-telegram-post.service
/home/openclaw/.config/systemd/user/youtube-telegram-post.timer
```

Enable:

```bash
systemctl --user daemon-reload
systemctl --user enable --now youtube-telegram-post.timer
systemctl --user list-timers youtube-telegram-post.timer --no-pager
```

## Orchestrator Implementation Checklist

The orchestrator should implement:

- argument parsing with `--dry-run`, `--self-test`, and `--env-file`,
- env loading before runtime work,
- process lock,
- channel config loading,
- discovery via RSS and/or `yt-dlp`,
- newest-only rule per channel,
- baseline initialization to avoid posting pre-go-live videos,
- transcript fetch via the fallback chain,
- blocked pending state when transcripts fail,
- project transcript/metadata/post paths resolved against `YOUTUBE_AGENT_DIR`,
  not `os.getcwd()` or the systemd working directory,
- target Telegram post parent directory created before invoking OpenClaw,
  including recovered `transcriptStatus=blocked` items,
- OpenClaw child process with secret-stripped environment,
- post validation before Telegram send,
- `send_in_flight` before Telegram request,
- `sent` only after Telegram success,
- `uncertain` on ambiguous send failures,
- monitor-state update after successful send,
- transcript retention cleanup that protects pending-referenced project and
  vault transcript paths.

## RapidAPI Transcript Normalization

Expected endpoint:

```text
GET /get_transcript?id=<VIDEO_ID>
```

Expected useful fields:

```json
{
  "selected": {
    "languageCode": "en"
  },
  "transcript": [
    {
      "text": "Segment text"
    }
  ]
}
```

Validation:

- response must be a JSON object,
- `transcript` must be a non-empty list,
- each segment must be an object,
- each segment must contain string `text`,
- at least one non-empty text segment is required,
- invalid data should fail the current key and try the next key.

## Telegram Channel Setup

Generic steps:

1. Create a Telegram channel.
2. Create a bot with BotFather.
3. Add the bot as channel admin.
4. Determine the channel id, usually a `-100...` id for channels.
5. Store the id in `TELEGRAM_CHAT_ID`.
6. Send only via Bot API from the server.

Do not hardcode the chat id in code.
