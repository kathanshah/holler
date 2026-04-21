# holler

> A voice layer for AI agents — speak prompts, hear summaries.

Hold a hotkey to dictate prompts into any terminal or app. Get short, summarized voice alerts when your agent finishes a task, asks for permission, or gets stuck.

Built first for Claude Code on macOS. Designed to grow into Codex, Cursor, MCP clients, and surfaces beyond the terminal.

## Status

**Early — design phase.** See [`docs/design.md`](docs/design.md) for the full architecture.

## Why

Coding with an AI agent often means staring at a terminal waiting for it to finish. `holler` flips that:

- **Speak instead of type** — dictate long prompts hands-free with a push-to-talk hotkey, paste lands in whatever app is focused (Claude Code, Codex, browser, anywhere).
- **Get a heads-up when it matters** — when the agent finishes, asks a question, or stalls, you hear a short spoken summary. No more babysitting the terminal.

## Components (v1)

| Piece | What it does |
|---|---|
| `voice-dictate` | macOS hotkey daemon. Hold a key, speak, release. Transcribed text pastes at the cursor. |
| `voice-alert` | Claude Code hook handler. Reads agent events, summarizes via Claude Haiku, plays audio via ElevenLabs. |

Both share a small `lib/` for STT, TTS, and summarization — so adding new agents (Codex, Cursor) later is mostly wiring, not rebuilding.

## Stack

- **STT:** ElevenLabs Scribe API
- **TTS:** ElevenLabs Flash API (falls back to macOS `say`)
- **Summarizer:** Claude Haiku 4.5 via the `claude` CLI (reuses your existing Claude Code auth — no extra API key)
- **Hotkey + paste:** Python daemon, autostarted via `launchd`

## Install

Coming soon. See `docs/design.md` for the planned setup flow.

## License

MIT — see [LICENSE](LICENSE).
