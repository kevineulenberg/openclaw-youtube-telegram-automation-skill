# Prompt And Output Contracts

Use this reference when writing OpenClaw prompts, Telegram validation rules, or architecture diagram prompts.

## OpenClaw Agent Contract

The daily per-video routine should not generate a summary file. It should generate exactly one Telegram-ready post file from the transcript.

Prompt requirements:

```text
Create exactly one Telegram-ready post for the configured Telegram channel.

Input:
- video title
- channel display name
- video URL
- transcript path
- target Telegram post path

Rules:
- Write only the Telegram post file.
- Do not create a summary file.
- Keep the post under 4096 Telegram characters.
- End the final line with the exact YouTube URL.
- Use the transcript as source material.
- Do not claim to be the real creator unless the user explicitly wants that style.
- Do not include hidden metadata, implementation notes, or file paths.
- Do not include unrelated disclaimers unless requested.
```

Validation after the agent:

```text
file exists
file is non-empty
length <= 4096
final line == video URL
```

## Telegram Send Contract

Send plain text by default:

```text
POST https://api.telegram.org/bot<TOKEN>/sendMessage
chat_id=<TELEGRAM_CHAT_ID>
text=<VALIDATED_POST>
disable_web_page_preview=false
```

Use `disable_web_page_preview=true` only for compact link-list posts, not necessarily for YouTube posts where preview may be useful.

## Flowchart Prompt

Use this prompt to create a diagram from the architecture:

```text
Create a clean architecture flowchart for a generic OpenClaw VPS YouTube-to-Telegram automation.

Style:
- Professional technical diagram
- Clear left-to-right flow
- Use grouped sections or swimlanes
- Minimal but complete labels
- Show server-side automation as the main path
- Show Mac/Codex fallback as a separate recovery layer
- Use arrows with short labels
- Do not include real tokens, real Telegram IDs, or real user-specific channels

Title:
OpenClaw YouTube Telegram Automation

Main server flow:
1. systemd --user timer on OpenClaw VPS
   Runs twice daily, e.g. 08:30 and 20:30 Australia/Brisbane

2. Python orchestrator
   Responsibilities:
   - checks configured YouTube channels
   - detects newest unknown video per channel
   - prevents old backlog posts
   - controls transcript, generation and Telegram send flow

3. YouTube discovery
   Sources:
   - @ExampleFinanceChannel via YouTube RSS
   - @ExampleMacroChannel via yt-dlp standard_uploads
   - @ExampleCryptoChannel via yt-dlp standard_uploads

4. State and deduplication
   Files:
   - data/seen-videos.json
   - data/telegram-post-state.json
   Notes:
   - key format: TELEGRAM_CHAT_ID:videoId
   - only newest unknown video per channel is eligible
   - sent, pending, send_in_flight and uncertain states prevent duplicates

5. Transcript fetch layer
   Ordered fallback chain:
   - youtube-transcript-api
   - RapidAPI YT get_transcript with key 1
   - RapidAPI YT get_transcript with key 2
   - if all fail: mark transcriptStatus=blocked
   Note: RapidAPI is only used for transcripts, not discovery.

6. Transcript storage
   Paths:
   - data/youtube-blogger/...
   - data/vault-mirror/...
   Note:
   Transcript Markdown files older than 14 days are cleaned up; pending transcript files are protected.

7. OpenClaw generation
   Input:
   ready transcript
   Output:
   one Telegram-ready post file only
   Notes:
   - no daily summary file
   - strict prompt
   - agent does not receive Telegram, Finnhub or RapidAPI secrets

8. Validation
   Checks:
   - file exists
   - non-empty
   - <= 4096 characters
   - final line equals YouTube URL

9. Telegram delivery
   Telegram Bot API sendMessage
   Target:
   example Telegram channel id from TELEGRAM_CHAT_ID

10. Send-state update
   Only after Telegram success:
   - store Telegram message_id
   - mark video as sent
   - update monitor state

Side recovery lane:
Mac / Codex Transcript Recovery Layer

Start:
Blocked transcriptStatus=blocked in telegram-post-state.json

Then:
- Codex scheduled automation on Mac
- local fallback helper
- SSH into VPS
- read pending blocked items
- fetch local captions with yt-dlp
- render transcript markdown
- upload transcript and metadata to VPS
- trigger normal server poster

Important recovery note:
Mac never sends Telegram messages.
Mac never marks items as sent.
Server remains source of truth.
```

## Weekly Digest Optional Extension

This is separate from daily per-video posts.

Schedule example:

```text
Monday 14:00 Australia/Brisbane
```

Inputs:

- videos already successfully posted in the previous calendar week,
- their transcripts and metadata.

Output:

- one compact weekly Telegram recap.

Rules:

- no channel names,
- no handles,
- no YouTube link list,
- no daily per-video post duplication,
- separate state file keyed by `TELEGRAM_CHAT_ID:weekStart`.
