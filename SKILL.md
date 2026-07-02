---
name: skill-yt-telegram
description: Build, adapt, review, troubleshoot, or document a generic OpenClaw VPS YouTube-to-Telegram automation. Use when setting up YouTube channel discovery, transcript fetching with youtube-transcript-api/RapidAPI/local Mac fallbacks, OpenClaw-generated Telegram posts, Telegram Bot API delivery, systemd user timers, duplicate-safe send state, transcript retention, Codex scheduled fallback automations, or architecture flowcharts for this workflow.
---

# YouTube Telegram Automation

Use this skill to help a user recreate the Financial-Advice-style YouTube-to-Telegram automation on their own OpenClaw server with generic example data.

## Non-Negotiables

- Never reuse a real production Telegram chat id, bot token, RapidAPI key, Finnhub key, OpenClaw credential, or user's channel-specific IDs in generated examples.
- Use placeholders such as `REPLACE_ME`, `-1001234567890`, `@ExampleFinanceChannel`, and `https://www.youtube.com/watch?v=VIDEO_ID`.
- Treat the VPS as source of truth: discovery, state, OpenClaw generation, Telegram sending, and sent-state updates happen on the server.
- Treat the local Mac/Codex automation as recovery only: it may fetch captions and upload transcript files, but must not send Telegram messages or mark videos as sent.
- Preserve duplicate safety: write send-state only after Telegram success, and use `send_in_flight`/`uncertain` states for ambiguous sends.
- Keep RapidAPI as a transcript fallback only, not a discovery mechanism.
- Keep workspace file writes independent from `systemd` `WorkingDirectory`: store state paths relative to `YOUTUBE_AGENT_DIR`, but resolve reads and writes with `agent_dir / relative_path`.

## Reference Routing

Read only the reference files needed for the user's task:

- For explaining the full flow, creating diagrams, or reviewing architecture, read `references/architecture.md`.
- For implementing the system on a fresh OpenClaw VPS, read `references/setup-blueprint.md`.
- For testing, deployment, troubleshooting, fallback recovery, and operational checks, read `references/operations.md`.
- For OpenClaw agent prompts, Telegram output rules, and flowchart prompts, read `references/prompt-contracts.md`.

## Default Architecture

The default workflow is:

```text
systemd --user timer
-> Python orchestrator on VPS
-> YouTube discovery
-> state/deduplication
-> transcript fallback chain
-> transcript storage
-> OpenClaw agent writes one Telegram post
-> Python validation
-> Telegram Bot API sendMessage
-> send-state update after Telegram success
```

Transcript fallback chain:

```text
1. youtube-transcript-api
2. RapidAPI YT transcript API key 1
3. RapidAPI YT transcript API key 2
4. local Mac/Codex fallback with yt-dlp
```

## Implementation Stance

When asked to implement, prefer:

- `systemd --user` timers over classic cron.
- One deterministic Python orchestrator script per routine.
- JSON state files for idempotency and manual recovery.
- Agent-workspace-relative storage paths, never process-CWD-relative storage writes.
- Strict env files for secrets.
- `yt-dlp` for `standard_uploads` discovery where RSS/handles are insufficient.
- `youtube-transcript-api` first, RapidAPI second, and local Mac `yt-dlp` captions last.
- Plain text Telegram messages unless the user explicitly asks for Markdown/HTML.

When asked to review, prioritize:

- duplicate Telegram sends,
- secret leakage to agent processes/logs,
- blocked transcript recovery,
- stale backlog posts,
- fragile parsing of external API responses,
- missing systemd verification,
- state writes before Telegram success.
