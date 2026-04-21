# holler — design spec

> A voice layer for AI agents — speak prompts, hear summaries.

**Status:** v1 design, pre-implementation.
**Date:** 2026-04-21
**Target platform:** macOS (Apple Silicon).

---

## 1. Why this exists

Working with AI coding agents (Claude Code, Codex, Cursor, MCP-driven tools) involves two repetitive frictions:

1. **Typing long, context-rich prompts.** Voice is faster, hands-free, and produces better prompts because you talk the way you think.
2. **Babysitting the terminal.** Agents pause for permission, finish silently, or stall mid-task. Without ambient awareness, you either watch the screen or miss the moment.

`holler` solves both with a single small toolset:

- **Push-to-talk dictation** that pastes into whatever window is focused — works with Claude Code, Codex, Cursor, browsers, anything.
- **Summarized voice alerts** triggered by agent events — short spoken updates ("done", "needs approval", "stuck on X").

## 2. Scope

### In scope (v1)

- macOS dev box (Apple Silicon)
- Single user, single machine
- Claude Code as the only agent for voice alerts (hooks already exist)
- Cloud APIs for audio (ElevenLabs for both STT and TTS) — no local Whisper/Piper
- Claude Haiku 4.5 for summarization (reuses existing `claude` CLI auth, no extra key)

### Out of scope (v1, designed-for-later)

- **Codex voice alerts** — Codex has a `notify_program` hook that takes a JSON payload; the architecture splits the alert handler so adding Codex is `+1 stdin parser, same TTS/summarizer pipeline`.
- **Mac Mini / headless agent nodes** — no listener, no value.
- **Local STT/TTS** — could be added as a fallback arm in `lib/stt.py` and `lib/tts.py` without restructuring.
- **Linux / Windows** — possible later, but the hotkey + paste layer is macOS-specific in v1.
- **A "read responses aloud" mode** — long markdown read-back is tedious; summarized alerts are the high-value play.

## 3. Architecture

Two small daemons + shared libs in one repo.

```
holler/
├── bin/
│   ├── voice-dictate            # global hotkey daemon (always-on)
│   └── voice-alert              # Claude Code hook handler (invoked on demand)
├── lib/
│   ├── stt.py                   # ElevenLabs Scribe wrapper + VAD trim
│   ├── tts.py                   # ElevenLabs Flash + macOS `say` fallback
│   └── summarize.py             # `claude -p --model claude-haiku-4-5` wrapper
├── hooks/
│   └── settings.snippet.json    # drop-in for ~/.claude/settings.json
├── launchd/
│   └── com.kathanshah.holler.voice-dictate.plist
├── install.sh                   # perms, launchd, hook wiring, key prompt
├── requirements.txt
├── docs/
│   └── design.md                # this file
└── README.md
```

### 3.1 Why one repo, not two

`voice-dictate` and `voice-alert` are independent at runtime but share three concerns:

- The TTS pipeline (`lib/tts.py`)
- The STT pipeline (`lib/stt.py`)
- API key + config management (`~/.config/holler/env`)

Splitting into two repos doubles the install surface and forks the shared libs. One repo, two entrypoints, common libs.

### 3.2 Why no MCP server in v1

MCP servers fit when an agent needs to *call* voice as a tool. `holler` is the inverse: voice wraps the agent, and the agent doesn't need to know it exists. Adding an MCP surface is a separate layer that could come later for "agent triggers TTS deliberately" use cases.

## 4. Input half — `voice-dictate`

### 4.1 Behavior

- Runs as a Python daemon, autostarted at login via `launchd`
- Listens for a global hotkey (`Right-Option` hold-to-talk by default)
- On hotkey-down: starts microphone capture
- While held: streams audio frames into a ring buffer
- On hotkey-up: stops capture, trims leading/trailing silence (webrtcvad), sends to ElevenLabs Scribe
- On STT response: writes transcript to clipboard, sends ⌘V to the focused app

### 4.2 Why hold-to-talk (not toggle)

- Tap-to-toggle has two failure modes: forgetting to start (lost first sentence) and forgetting to stop (transcribes the next phone call).
- Hold-to-talk has one failure mode: lifting the key too early. Easier to learn, harder to misuse.
- A future flag can switch to toggle for users who prefer it.

### 4.3 VAD pre-trim (cherry-picked from `local.ai/listen.sh`)

Sending the raw recording to Scribe is expensive and slow. Trimming silence with `webrtcvad` before the API call:

- Cuts API cost roughly in half on typical 5-15s utterances
- Reduces round-trip latency by 200-500ms
- Avoids the "transcribed your breath as 'um'" failure mode

### 4.4 Permissions required (one-time)

- **Accessibility** — `pynput` needs it for global hotkey capture and synthetic ⌘V. Prompt fired by `install.sh` via a no-op test.
- **Microphone** — triggered the first time `sounddevice` opens an `InputStream`.

### 4.5 Failure handling

- ElevenLabs API failure → audible beep + macOS notification ("STT failed: <reason>"). No silent failure. No retry storm — one attempt per hold.
- No speech detected (VAD trimmed everything) → silent no-op; daemon stays ready for next hold.
- Daemon crash → `launchd` `KeepAlive=true` restarts within seconds.

## 5. Output half — `voice-alert`

### 5.1 Behavior

Invoked by Claude Code hooks. Receives JSON on stdin, decides whether to speak, summarizes, plays audio, exits.

```
Claude fires Stop / SubagentStop / Notification hook
      ↓
  stdin: {"hook_event_name": "...", "transcript_path": "...", ...}
      ↓
  voice-alert reads stdin, extracts relevant text
      ↓
  claude -p --model claude-haiku-4-5
      "Summarize the agent's last turn in ≤12 words..."
      ↓
  strip markdown
      ↓
  ElevenLabs Flash TTS → sounddevice playback
      (fallback: macOS `say`)
```

### 5.2 Hooks wired

| Hook | Source field | Summarizer prompt | Example output |
|---|---|---|---|
| `Stop` | last assistant message in transcript | *"Summarize what the agent did in ≤12 words. Be terse, factual."* | *"Refactored tax service, 18 tests pass."* |
| `SubagentStop` | last subagent message | *"Summarize what the subagent produced in ≤10 words."* | *"Code reviewer flagged two security issues."* |
| `Notification` | `message` field | *"Restate the permission request in ≤10 words."* | *"Waiting for approval to run migrations."* |

### 5.3 Why summarize via `claude -p` and not OpenAI / OpenRouter

- **Reuses existing Claude Code auth** — zero extra keys to manage, zero extra billing accounts.
- **Haiku 4.5 is cheap and fast** — ~$0.25/M input, ~$1.25/M output. A typical Stop summary costs a fraction of a cent.
- **Single point of model control** — when Anthropic releases a faster Haiku, `holler` upgrades automatically.

### 5.4 Latency budget

| Step | Target |
|---|---|
| Hook fires → script starts | <50ms (subprocess spawn) |
| Read + parse stdin | <10ms |
| `claude -p` call (Haiku 4.5) | ~400-800ms |
| ElevenLabs Flash TTS | ~300-600ms |
| Audio starts playing | **~1.5-2s total** |

If total time exceeds 3s, the script bails and falls back to a canned string ("Claude is done") — late notifications feel broken and erode trust.

### 5.5 Failure handling (graceful degradation chain)

1. `claude -p` fails or times out (>3s) → use a canned string for the event type
2. ElevenLabs TTS fails → fall back to macOS `say`
3. `say` fails → log to `~/.config/holler/error.log`, exit silently (don't crash Claude Code)

The chain guarantees: if any single component is broken, you still hear *something* — the system never goes silent without warning.

## 6. Shared libs

### 6.1 `lib/tts.py`

Cherry-picked from `local.ai/speak.sh`, ported to Python.

- `speak(text: str)` — primary entrypoint
- Markdown stripping (code blocks, inline code, bold, italics, headers, list bullets, links)
- ElevenLabs Flash via the `elevenlabs` Python SDK, output `pcm_22050`, played via `sounddevice`
- Voice ID configurable via env, defaults to a single neutral voice
- Fallback to `subprocess.run(["say", text])` on any ElevenLabs error

### 6.2 `lib/stt.py`

- `transcribe(audio: np.ndarray) -> str` — primary entrypoint
- VAD trim (webrtcvad, aggressiveness 2) on input audio
- ElevenLabs Scribe API call
- Returns empty string on no-speech-detected (caller decides what to do)

### 6.3 `lib/summarize.py`

```python
def summarize(text: str, prompt: str, *, max_seconds: float = 3.0) -> str | None
```

- Wraps `subprocess.run(["claude", "-p", "--model", "claude-haiku-4-5", full_prompt], timeout=max_seconds)`
- Returns `None` on timeout or non-zero exit (caller falls back to canned string)
- No retries — fast or nothing

## 7. Setup flow

### 7.1 What `install.sh` does

1. Verify deps: `python3.11+`, `ffmpeg` (`brew install ffmpeg`), `osascript` (always present), `claude` on PATH
2. Create venv at `~/.holler/.venv`, `pip install -r requirements.txt`
3. Prompt for `ELEVENLABS_API_KEY` (or read from env), write to `~/.config/holler/env` (chmod 600)
4. Install launchd plist → daemon autostarts and starts now
5. Trigger Accessibility + Microphone permission prompts (run a no-op `pynput` listener once)
6. Merge `hooks/settings.snippet.json` into `~/.claude/settings.json` (preserve existing hooks via JSON merge, not overwrite)
7. Self-test: invoke `voice-alert` with a fake `Stop` payload, confirm audio plays
8. Print final instructions: hotkey, how to disable, where logs live

### 7.2 Uninstall

`./uninstall.sh`:

- Unload + remove launchd plist
- Remove `holler` hook entries from `~/.claude/settings.json` (additive merge in reverse)
- Leave the venv and config in place (user explicitly removes if desired)

## 8. Submodule story

`holler` is structured to be vendored into other repos that want voice on:

```bash
git submodule add https://github.com/kathanshah/holler .holler
ln -sf .holler/hooks/settings.snippet.json .claude/holler-hooks.json
# Reference in .claude/settings.json
```

The hotkey daemon is **machine-level** (one install per Mac), not per-repo. So the submodule only ships:

- The hooks fragment for `voice-alert`
- The setup script (idempotent — re-running after the daemon is installed is a no-op)
- README pointing at one-time per-Mac install steps

## 9. Success criteria

1. Hold Right-Option in any focused app → speak → release → text appears within ~1-1.5s
2. Claude Code finishes a turn → 8-12 word summary plays within ~2s
3. Claude Code asks for permission → I hear the reason in plain English
4. Subagent finishes → I hear what it produced
5. Any single component fails → fallback engages, system never goes silent
6. `./install.sh` on a fresh Mac → working voice in <5 minutes

## 10. Non-goals (explicit)

- **Not a Claude Desktop replacement** — that has its own voice mode.
- **Not a `vibecoding` tool** — `holler` is for engineers who type for a living and want voice as a complement, not a replacement.
- **Not a wake-word system** — push-to-talk only. Always-listening daemons are creepy and wrong for this use case.
- **Not an agent personality framework** — summaries are terse and factual. No Jarvis monologue.

## 11. Risks & open questions

| Risk | Mitigation |
|---|---|
| `claude -p --model claude-haiku-4-5` may behave differently across Claude.ai login vs. API key auth | Verify on Day 1 of implementation. If broken under a particular auth, swap to direct Anthropic API call (still no new key — reuses `ANTHROPIC_API_KEY` from Claude Code's env). |
| ElevenLabs Scribe quality on technical jargon (function names, file paths) | Validate with 50 sample utterances before declaring v1 done. If quality is poor, add OpenAI Whisper as a second arm in `lib/stt.py`. |
| Global hotkey conflict on user's Mac (Right-Option already used) | Make hotkey configurable in `~/.config/holler/env` from Day 1. |
| `pynput` synthetic ⌘V doesn't work in some apps (sandboxed Mac App Store apps with extra entitlements) | Document the limitation; fall back to clipboard-only (user pastes manually) if synthetic key fails. |
| First-run macOS Accessibility prompt requires the user to manually flip a switch in System Settings | `install.sh` opens the System Settings pane and prints clear instructions. |

## 12. Roadmap (post-v1, illustrative only)

- **v1.1** — Codex `notify_program` bridge (same `voice-alert` script, new stdin parser).
- **v1.2** — Configurable summarization prompts per event type.
- **v2.0** — Local-only mode (whisper.cpp + Piper) for offline / privacy-sensitive contexts.
- **v2.1** — Cross-platform (Linux at minimum; Windows if there's demand).
