# Operations And Troubleshooting

Use this reference for verification, recovery, and production safety.

## Standard Verification

Run on the VPS:

```bash
cd /home/openclaw/.openclaw/workspace/youtube-api-agent
.venv/bin/python src/transcripts.py --self-test
.venv/bin/python -m py_compile src/transcripts.py
.venv/bin/python /home/openclaw/.openclaw/workspace/scripts/youtube_telegram_post.py --self-test
.venv/bin/python /home/openclaw/.openclaw/workspace/scripts/youtube_telegram_post.py --dry-run
systemctl --user list-timers youtube-telegram-post.timer --no-pager
journalctl --user -u youtube-telegram-post.service -n 100 --no-pager
```

Expected no-op dry-run:

```json
{
  "dryRun": true,
  "pending": 0,
  "newCandidates": [],
  "messages": []
}
```

## RapidAPI Key Test

Test keys individually without printing secrets:

```bash
cd /home/openclaw/.openclaw/workspace/youtube-api-agent
set -a
. /home/openclaw/.config/youtube-telegram.env
set +a
.venv/bin/python - <<'PY'
import os
from src.transcripts import fetch_rapidapi_transcript_payload, parse_rapidapi_keys, VideoInfo

keys = parse_rapidapi_keys(os.environ.get("RAPIDAPI_YT_TRANSCRIPT_KEYS"))
print({"configuredKeys": len(keys)})
for index, key in enumerate(keys, start=1):
    payload = fetch_rapidapi_transcript_payload(
        "VIDEO_ID_HERE",
        "https://www.youtube.com/watch?v=VIDEO_ID_HERE",
        VideoInfo("Example Video", "@ExampleChannel"),
        keys=[key],
    )
    print({
        "key": index,
        "works": True,
        "source": payload.transcript_source,
        "language": payload.language,
        "chars": len(payload.transcript_text),
    })
PY
```

If a key returns `403` with `You are not subscribed to this API`, the RapidAPI account exists but must subscribe to the specific `YT Data + Download API` product before it can act as a fallback.

## Local Mac/Codex Fallback Automation

Use local Codex automations only after server timers, for example:

```text
08:45 Australia/Brisbane
20:45 Australia/Brisbane
```

Generic automation prompt:

```text
Use the local YouTube transcript fallback workflow for the Telegram automation.

First check the OpenClaw server state with:
ssh openclaw 'python3 /home/openclaw/.openclaw/workspace/scripts/youtube_telegram_post.py --dry-run'

If there are no pending items with transcriptStatus=blocked, finish with a short no-op status.

If blocked transcript items exist, run the local recovery helper that:
- reads the server pending state over SSH,
- fetches captions locally with yt-dlp,
- uploads transcript and metadata files to the expected server paths,
- triggers the normal server poster.

Then verify with another server dry-run and a short journal tail.

Report recovered video IDs, remaining blocked items, and whether the server poster completed.
Do not send Telegram messages locally.
Do not mark items as sent locally.
Respect the newest-only rule per YouTube channel.
```

## Manual Reconciliation

If send-state contains `send_in_flight` or `uncertain`:

1. Check the Telegram channel manually.
2. If the post is visible, move the item to `sent` and record `telegramMessageId` if known.
3. If the post is not visible, reset or remove the pending state carefully and rerun.
4. Do not blindly retry an uncertain item.

## Common Failure Modes

### No Post Today

Check:

- `--dry-run` output,
- `journalctl`,
- whether channels published a new video after baseline,
- whether newest video is already in `sent`,
- whether transcript is blocked pending recovery.

No post is correct if no newest unknown video exists.

### YouTube Blocks VPS Transcripts

Expected behavior:

- `youtube-transcript-api` fails,
- RapidAPI keys are tried,
- if all fail, item becomes `transcriptStatus=blocked`,
- Mac automation may recover captions later.

### Older Video Would Be Posted

Reject that behavior. The watcher must only process newest unknown video per channel. Older backlog behind a known newer video must be skipped.

### Duplicate Telegram Post Risk

Check:

- state key uses `TELEGRAM_CHAT_ID:videoId`,
- `sent` is written only after Telegram success,
- ambiguous sends become `uncertain`,
- monitor state is updated after successful send,
- no local fallback process writes `sent`.

### Secrets In Agent Environment

Before spawning OpenClaw agent, remove at least:

```text
TELEGRAM_BOT_TOKEN
RAPIDAPI_YT_TRANSCRIPT_KEYS
FINNHUB_API_KEY
```

Prefer an allowlist if the environment contains many unrelated secrets.

## Deployment Checklist

After edits:

```bash
python3 -m py_compile scripts/youtube_telegram_post.py
python3 -m py_compile youtube-api-agent/src/transcripts.py
git diff --check
```

Deploy generic examples:

```bash
scp scripts/youtube_telegram_post.py openclaw:/home/openclaw/.openclaw/workspace/scripts/youtube_telegram_post.py
scp youtube-api-agent/src/transcripts.py openclaw:/home/openclaw/.openclaw/workspace/youtube-api-agent/src/transcripts.py
```

Then run server verification again.
