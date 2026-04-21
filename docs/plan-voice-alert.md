# Plan A — voice-alert + shared libs

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the first milestone of holler — summarized voice alerts on Claude Code hooks (Stop, SubagentStop, Notification). No dictation yet; that's Plan B.

**Architecture:** Python package `holler/` holding shared libs (`markdown`, `config`, `tts`, `summarize`) plus a CLI entrypoint `holler.cli.voice_alert`. A thin `bin/voice-alert` shell wrapper makes it directly invokable from Claude Code's hook config. TTS uses ElevenLabs Flash with macOS `say` fallback. Summarization uses `claude -p --model claude-haiku-4-5` (subprocess) to reuse the user's existing Claude Code auth — no extra API key.

**Tech Stack:** Python 3.11+, `elevenlabs` SDK, `sounddevice`, `numpy`, `pytest`, `pytest-mock`.

**Milestone 1 deliverables:**
- `holler/` package with `markdown.py`, `config.py`, `tts.py`, `summarize.py`, `cli/voice_alert.py`
- `bin/voice-alert` executable wrapper
- `hooks/settings.snippet.json` for Claude Code
- `install.sh` (Milestone-1 subset: ElevenLabs key, venv, hook wiring, self-test)
- Full pytest suite with ≥90% coverage on libs
- README install section for voice alerts

**Out of scope for this plan:** `stt.py`, `voice-dictate` daemon, launchd plist, Accessibility perms. All deferred to Plan B.

---

## File map

Files this plan creates:

| Path | Purpose |
|---|---|
| `pyproject.toml` | Project metadata, tool configs (pytest, ruff) |
| `requirements.txt` | Runtime deps |
| `requirements-dev.txt` | Dev deps (pytest, pytest-mock) |
| `holler/__init__.py` | Package marker + version |
| `holler/markdown.py` | Strip markdown for speech synthesis |
| `holler/config.py` | Load API keys + voice settings from env / `~/.config/holler/env` |
| `holler/tts.py` | ElevenLabs Flash + `say` fallback |
| `holler/summarize.py` | `claude -p` subprocess wrapper with timeout |
| `holler/cli/__init__.py` | CLI subpackage marker |
| `holler/cli/voice_alert.py` | Hook-event dispatcher |
| `bin/voice-alert` | Shell wrapper around `python -m holler.cli.voice_alert` |
| `hooks/settings.snippet.json` | Claude Code hooks fragment |
| `install.sh` | Mac install (Milestone-1 subset) |
| `tests/__init__.py` | Test package marker |
| `tests/test_markdown.py` | Markdown stripper tests |
| `tests/test_config.py` | Config loader tests |
| `tests/test_tts.py` | TTS module tests (mocked) |
| `tests/test_summarize.py` | Summarize module tests (mocked subprocess) |
| `tests/test_voice_alert.py` | Dispatcher tests (mocked deps) |

Files modified:

| Path | Change |
|---|---|
| `.gitignore` | Already comprehensive — no change needed |
| `README.md` | Replace "Install: coming soon" with Milestone-1 install steps |

---

## Task 1: Scaffold Python project

**Files:**
- Create: `pyproject.toml`
- Create: `requirements.txt`
- Create: `requirements-dev.txt`
- Create: `holler/__init__.py`
- Create: `holler/cli/__init__.py`
- Create: `tests/__init__.py`
- Create: `tests/conftest.py`

- [ ] **Step 1.1: Create `pyproject.toml`**

```toml
[project]
name = "holler"
version = "0.1.0"
description = "A voice layer for AI agents — speak prompts, hear summaries."
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
authors = [{ name = "Kathan Shah" }]

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
include = ["holler*"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
addopts = "-v --tb=short"

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "SIM"]
```

- [ ] **Step 1.2: Create `requirements.txt`**

```
elevenlabs>=1.10.0
sounddevice>=0.4.6
numpy>=1.26
```

- [ ] **Step 1.3: Create `requirements-dev.txt`**

```
-r requirements.txt
pytest>=8.0
pytest-mock>=3.12
ruff>=0.4
```

- [ ] **Step 1.4: Create `holler/__init__.py`**

```python
"""holler — a voice layer for AI agents."""

__version__ = "0.1.0"
```

- [ ] **Step 1.5: Create `holler/cli/__init__.py`**

```python
"""CLI entrypoints."""
```

- [ ] **Step 1.6: Create `tests/__init__.py`**

Empty file.

- [ ] **Step 1.7: Create `tests/conftest.py`**

```python
"""Shared pytest fixtures."""

import pytest


@pytest.fixture
def fake_elevenlabs_key(monkeypatch):
    """Set a fake ElevenLabs API key in the environment."""
    monkeypatch.setenv("ELEVENLABS_API_KEY", "test-key-fake")
    return "test-key-fake"
```

- [ ] **Step 1.8: Verify scaffolding works**

Run: `python -m venv .venv && .venv/bin/pip install -r requirements-dev.txt && .venv/bin/pytest`
Expected: `0 tests ran` (no tests yet) — exit code 0 or 5 (no tests collected). Confirms pytest can import the package structure.

- [ ] **Step 1.9: Commit**

```bash
git add pyproject.toml requirements.txt requirements-dev.txt holler/ tests/
git commit -m "Scaffold Python project structure"
```

---

## Task 2: Markdown stripper (`holler/markdown.py`)

Used to clean markdown out of text before sending to TTS so the voice doesn't say "star star bold star star".

**Files:**
- Create: `holler/markdown.py`
- Create: `tests/test_markdown.py`

- [ ] **Step 2.1: Write failing tests**

Create `tests/test_markdown.py`:

```python
"""Tests for markdown stripper."""

from holler.markdown import strip_markdown


def test_strips_fenced_code_blocks():
    text = "before\n```python\nfoo = 1\n```\nafter"
    assert strip_markdown(text) == "before\n\nafter"


def test_strips_inline_code():
    assert strip_markdown("hello `world` ok") == "hello world ok"


def test_strips_bold():
    assert strip_markdown("**bold** text") == "bold text"


def test_strips_italic():
    assert strip_markdown("*italic* text") == "italic text"


def test_strips_atx_headers():
    assert strip_markdown("# Heading\nBody") == "Heading\nBody"
    assert strip_markdown("### Sub\nBody") == "Sub\nBody"


def test_strips_list_bullets():
    assert strip_markdown("- first\n- second") == "first\nsecond"
    assert strip_markdown("* star\n* bullet") == "star\nbullet"


def test_strips_markdown_links_keeps_text():
    assert strip_markdown("see [docs](https://x.com) here") == "see docs here"


def test_preserves_plain_text():
    assert strip_markdown("hello world") == "hello world"


def test_empty_string():
    assert strip_markdown("") == ""


def test_only_markdown_returns_empty():
    assert strip_markdown("```\ncode\n```").strip() == ""


def test_combined_markdown():
    text = "**Refactored** `user.py` — see [PR](https://x.com/pr/1)"
    assert strip_markdown(text) == "Refactored user.py — see PR"
```

- [ ] **Step 2.2: Run tests — verify they fail**

Run: `.venv/bin/pytest tests/test_markdown.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'holler.markdown'`

- [ ] **Step 2.3: Implement `holler/markdown.py`**

```python
"""Strip markdown formatting for cleaner text-to-speech output."""

import re

_FENCED_CODE = re.compile(r"```[\s\S]*?```")
_INLINE_CODE = re.compile(r"`([^`]+)`")
_BOLD = re.compile(r"\*\*([^*]+)\*\*")
_ITALIC = re.compile(r"\*([^*]+)\*")
_HEADER = re.compile(r"^#{1,6}\s+", flags=re.MULTILINE)
_LIST_BULLET = re.compile(r"^\s*[-*]\s+", flags=re.MULTILINE)
_LINK = re.compile(r"\[([^\]]+)\]\([^)]+\)")


def strip_markdown(text: str) -> str:
    """Remove markdown formatting so TTS speaks the underlying text only.

    Fenced code blocks are removed entirely (code-as-speech is useless).
    Inline code, bold, italic, headers, list bullets, and links have their
    markup removed but their text content preserved.
    """
    text = _FENCED_CODE.sub("", text)
    text = _INLINE_CODE.sub(r"\1", text)
    text = _BOLD.sub(r"\1", text)
    text = _ITALIC.sub(r"\1", text)
    text = _HEADER.sub("", text)
    text = _LIST_BULLET.sub("", text)
    text = _LINK.sub(r"\1", text)
    return text
```

- [ ] **Step 2.4: Run tests — verify they pass**

Run: `.venv/bin/pytest tests/test_markdown.py -v`
Expected: All 11 tests PASS.

- [ ] **Step 2.5: Commit**

```bash
git add holler/markdown.py tests/test_markdown.py
git commit -m "Add markdown stripper for TTS preprocessing"
```

---

## Task 3: Config loader (`holler/config.py`)

Loads `ELEVENLABS_API_KEY` and voice settings from environment variables, with fallback to `~/.config/holler/env`. Environment variables win.

**Files:**
- Create: `holler/config.py`
- Create: `tests/test_config.py`

- [ ] **Step 3.1: Write failing tests**

Create `tests/test_config.py`:

```python
"""Tests for config loader."""

from pathlib import Path

import pytest

from holler.config import load_config


@pytest.fixture
def fake_home(tmp_path: Path, monkeypatch):
    """Redirect HOME to a tmp dir so we don't touch the real user config."""
    monkeypatch.setenv("HOME", str(tmp_path))
    monkeypatch.delenv("ELEVENLABS_API_KEY", raising=False)
    monkeypatch.delenv("HOLLER_VOICE_ID", raising=False)
    return tmp_path


def test_returns_none_key_when_nothing_set(fake_home):
    cfg = load_config()
    assert cfg.elevenlabs_api_key is None


def test_env_var_wins(fake_home, monkeypatch):
    monkeypatch.setenv("ELEVENLABS_API_KEY", "env-key")
    cfg = load_config()
    assert cfg.elevenlabs_api_key == "env-key"


def test_reads_from_config_file(fake_home):
    cfg_dir = fake_home / ".config" / "holler"
    cfg_dir.mkdir(parents=True)
    (cfg_dir / "env").write_text('ELEVENLABS_API_KEY="file-key"\n')
    cfg = load_config()
    assert cfg.elevenlabs_api_key == "file-key"


def test_env_var_overrides_file(fake_home, monkeypatch):
    cfg_dir = fake_home / ".config" / "holler"
    cfg_dir.mkdir(parents=True)
    (cfg_dir / "env").write_text("ELEVENLABS_API_KEY=file-key\n")
    monkeypatch.setenv("ELEVENLABS_API_KEY", "env-key")
    cfg = load_config()
    assert cfg.elevenlabs_api_key == "env-key"


def test_voice_id_defaults_to_neutral_voice(fake_home):
    cfg = load_config()
    assert cfg.voice_id == "JBFqnCBsd6RMkjVDRZzb"  # George (ElevenLabs default neutral male)


def test_voice_id_override_from_env(fake_home, monkeypatch):
    monkeypatch.setenv("HOLLER_VOICE_ID", "abc123")
    cfg = load_config()
    assert cfg.voice_id == "abc123"


def test_tts_model_default(fake_home):
    cfg = load_config()
    assert cfg.tts_model == "eleven_flash_v2_5"


def test_file_ignores_comments_and_blank_lines(fake_home):
    cfg_dir = fake_home / ".config" / "holler"
    cfg_dir.mkdir(parents=True)
    (cfg_dir / "env").write_text(
        "# comment\n\nELEVENLABS_API_KEY=ok\n#ELEVENLABS_API_KEY=wrong\n"
    )
    cfg = load_config()
    assert cfg.elevenlabs_api_key == "ok"


def test_handles_quoted_values(fake_home):
    cfg_dir = fake_home / ".config" / "holler"
    cfg_dir.mkdir(parents=True)
    (cfg_dir / "env").write_text('ELEVENLABS_API_KEY="quoted-key"\n')
    cfg = load_config()
    assert cfg.elevenlabs_api_key == "quoted-key"
```

- [ ] **Step 3.2: Run tests — verify they fail**

Run: `.venv/bin/pytest tests/test_config.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'holler.config'`

- [ ] **Step 3.3: Implement `holler/config.py`**

```python
"""Load runtime configuration from environment + ~/.config/holler/env."""

from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path

CONFIG_FILE = Path.home() / ".config" / "holler" / "env"
DEFAULT_VOICE_ID = "JBFqnCBsd6RMkjVDRZzb"  # George — neutral male
DEFAULT_TTS_MODEL = "eleven_flash_v2_5"


@dataclass(frozen=True)
class Config:
    elevenlabs_api_key: str | None
    voice_id: str
    tts_model: str


def _read_file_env(path: Path) -> dict[str, str]:
    """Parse a simple KEY=VALUE env file. Ignores blanks and `#` comments."""
    if not path.exists():
        return {}
    result: dict[str, str] = {}
    for raw in path.read_text().splitlines():
        line = raw.strip()
        if not line or line.startswith("#"):
            continue
        if "=" not in line:
            continue
        key, _, value = line.partition("=")
        value = value.strip().strip('"').strip("'")
        result[key.strip()] = value
    return result


def load_config() -> Config:
    """Load config. Env vars take precedence over the config file."""
    file_env = _read_file_env(CONFIG_FILE)
    return Config(
        elevenlabs_api_key=os.environ.get("ELEVENLABS_API_KEY")
        or file_env.get("ELEVENLABS_API_KEY")
        or None,
        voice_id=os.environ.get("HOLLER_VOICE_ID")
        or file_env.get("HOLLER_VOICE_ID")
        or DEFAULT_VOICE_ID,
        tts_model=os.environ.get("HOLLER_TTS_MODEL")
        or file_env.get("HOLLER_TTS_MODEL")
        or DEFAULT_TTS_MODEL,
    )
```

- [ ] **Step 3.4: Run tests — verify they pass**

Run: `.venv/bin/pytest tests/test_config.py -v`
Expected: All 9 tests PASS.

- [ ] **Step 3.5: Commit**

```bash
git add holler/config.py tests/test_config.py
git commit -m "Add config loader with env and file fallback"
```

---

## Task 4: Summarize module (`holler/summarize.py`)

Wraps `claude -p --model claude-haiku-4-5 "prompt"` as a subprocess call with a hard timeout. Returns `None` on failure/timeout so callers can fall back to a canned string.

**Files:**
- Create: `holler/summarize.py`
- Create: `tests/test_summarize.py`

- [ ] **Step 4.1: Write failing tests**

Create `tests/test_summarize.py`:

```python
"""Tests for the Claude CLI summarizer."""

import subprocess
from unittest.mock import MagicMock

from holler.summarize import summarize


def test_returns_stdout_on_success(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=0, stdout="  Refactored the service.  \n", stderr="")
    result = summarize("agent output", "summarize please")
    assert result == "Refactored the service."


def test_returns_none_on_nonzero_exit(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=1, stdout="", stderr="auth error")
    assert summarize("text", "prompt") is None


def test_returns_none_on_timeout(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.side_effect = subprocess.TimeoutExpired(cmd="claude", timeout=3.0)
    assert summarize("text", "prompt", max_seconds=3.0) is None


def test_returns_none_on_file_not_found(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.side_effect = FileNotFoundError("claude not on PATH")
    assert summarize("text", "prompt") is None


def test_passes_correct_model(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=0, stdout="ok", stderr="")
    summarize("text", "prompt")
    args, kwargs = run.call_args
    cmd = args[0]
    assert "claude" in cmd
    assert "-p" in cmd
    assert "--model" in cmd
    assert "claude-haiku-4-5" in cmd


def test_combines_prompt_and_text(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=0, stdout="ok", stderr="")
    summarize("AGENT_TEXT_HERE", "PROMPT_HERE")
    args, _ = run.call_args
    cmd = args[0]
    full_prompt = cmd[-1]
    assert "AGENT_TEXT_HERE" in full_prompt
    assert "PROMPT_HERE" in full_prompt


def test_respects_max_seconds(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=0, stdout="ok", stderr="")
    summarize("text", "prompt", max_seconds=1.5)
    _, kwargs = run.call_args
    assert kwargs["timeout"] == 1.5


def test_empty_stdout_returns_none(mocker):
    run = mocker.patch("holler.summarize.subprocess.run")
    run.return_value = MagicMock(returncode=0, stdout="   \n", stderr="")
    assert summarize("text", "prompt") is None
```

- [ ] **Step 4.2: Run tests — verify they fail**

Run: `.venv/bin/pytest tests/test_summarize.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'holler.summarize'`

- [ ] **Step 4.3: Implement `holler/summarize.py`**

```python
"""Summarize text via the local `claude` CLI using Haiku 4.5."""

from __future__ import annotations

import logging
import subprocess

log = logging.getLogger(__name__)

_MODEL = "claude-haiku-4-5"


def summarize(text: str, prompt: str, *, max_seconds: float = 3.0) -> str | None:
    """Return a short summary of `text` using the Haiku model, or None on failure.

    `prompt` is the instruction (e.g. "Summarize in ≤12 words"). `text` is the
    agent output being summarized. The two are combined before being sent.

    Returns None on timeout, non-zero exit, empty output, or missing `claude`
    binary — callers should fall back to a canned string in any of those cases.
    """
    full_prompt = f"{prompt}\n\n---\n{text}"
    try:
        proc = subprocess.run(
            ["claude", "-p", "--model", _MODEL, full_prompt],
            capture_output=True,
            text=True,
            timeout=max_seconds,
        )
    except subprocess.TimeoutExpired:
        log.warning("summarize: claude timed out after %.1fs", max_seconds)
        return None
    except FileNotFoundError:
        log.error("summarize: `claude` not found on PATH")
        return None

    if proc.returncode != 0:
        log.warning("summarize: claude exited %d: %s", proc.returncode, proc.stderr.strip())
        return None

    result = proc.stdout.strip()
    return result or None
```

- [ ] **Step 4.4: Run tests — verify they pass**

Run: `.venv/bin/pytest tests/test_summarize.py -v`
Expected: All 8 tests PASS.

- [ ] **Step 4.5: Commit**

```bash
git add holler/summarize.py tests/test_summarize.py
git commit -m "Add Claude CLI summarizer wrapper"
```

---

## Task 5: TTS module (`holler/tts.py`)

ElevenLabs Flash primary, `say` fallback. Markdown is stripped before synthesis. Empty text after stripping is a silent no-op. Both engines failing logs an error but does NOT raise — hook handlers must never crash Claude Code.

**Files:**
- Create: `holler/tts.py`
- Create: `tests/test_tts.py`

- [ ] **Step 5.1: Write failing tests**

Create `tests/test_tts.py`:

```python
"""Tests for TTS module."""

from unittest.mock import MagicMock

import numpy as np

from holler.tts import speak


def _fake_client_with_pcm(pcm_bytes: bytes):
    """Build a fake ElevenLabs client that returns the given PCM bytes."""
    client = MagicMock()
    client.text_to_speech.convert.return_value = iter([pcm_bytes])
    return client


def test_empty_text_is_noop(mocker):
    elevenlabs = mocker.patch("holler.tts.ElevenLabs")
    run = mocker.patch("holler.tts.subprocess.run")
    speak("")
    assert elevenlabs.call_count == 0
    assert run.call_count == 0


def test_markdown_only_is_noop(mocker):
    elevenlabs = mocker.patch("holler.tts.ElevenLabs")
    run = mocker.patch("holler.tts.subprocess.run")
    speak("```\njust code\n```")
    assert elevenlabs.call_count == 0
    assert run.call_count == 0


def test_uses_elevenlabs_when_key_present(mocker, fake_elevenlabs_key):
    fake_client = _fake_client_with_pcm(b"\x00\x00" * 1000)
    mocker.patch("holler.tts.ElevenLabs", return_value=fake_client)
    play = mocker.patch("holler.tts.sd.play")
    wait = mocker.patch("holler.tts.sd.wait")

    speak("hello world")

    assert fake_client.text_to_speech.convert.called
    args, kwargs = fake_client.text_to_speech.convert.call_args
    assert kwargs["text"] == "hello world"
    assert "voice_id" in kwargs
    assert "model_id" in kwargs
    play.assert_called_once()
    wait.assert_called_once()


def test_strips_markdown_before_elevenlabs(mocker, fake_elevenlabs_key):
    fake_client = _fake_client_with_pcm(b"\x00\x00" * 100)
    mocker.patch("holler.tts.ElevenLabs", return_value=fake_client)
    mocker.patch("holler.tts.sd.play")
    mocker.patch("holler.tts.sd.wait")

    speak("**Bold** and `code`")

    args, kwargs = fake_client.text_to_speech.convert.call_args
    assert "**" not in kwargs["text"]
    assert "`" not in kwargs["text"]
    assert "Bold and code" in kwargs["text"]


def test_falls_back_to_say_on_elevenlabs_error(mocker, fake_elevenlabs_key):
    fake_client = MagicMock()
    fake_client.text_to_speech.convert.side_effect = RuntimeError("api down")
    mocker.patch("holler.tts.ElevenLabs", return_value=fake_client)
    run = mocker.patch("holler.tts.subprocess.run")
    run.return_value = MagicMock(returncode=0)

    speak("fallback test")

    run.assert_called_once()
    args, _ = run.call_args
    assert args[0][0] == "say"
    assert "fallback test" in args[0]


def test_falls_back_to_say_when_no_key(mocker, monkeypatch, tmp_path):
    # Clear key + redirect HOME so config file can't supply one either
    monkeypatch.setenv("HOME", str(tmp_path))
    monkeypatch.delenv("ELEVENLABS_API_KEY", raising=False)
    elevenlabs = mocker.patch("holler.tts.ElevenLabs")
    run = mocker.patch("holler.tts.subprocess.run")
    run.return_value = MagicMock(returncode=0)

    speak("no key path")

    assert elevenlabs.call_count == 0
    run.assert_called_once()


def test_both_engines_fail_does_not_raise(mocker, fake_elevenlabs_key):
    fake_client = MagicMock()
    fake_client.text_to_speech.convert.side_effect = RuntimeError("api down")
    mocker.patch("holler.tts.ElevenLabs", return_value=fake_client)
    mocker.patch("holler.tts.subprocess.run", side_effect=OSError("say missing"))

    # Must not raise — hooks must never crash Claude Code
    speak("double fail")


def test_empty_pcm_from_elevenlabs_falls_back(mocker, fake_elevenlabs_key):
    fake_client = _fake_client_with_pcm(b"")
    mocker.patch("holler.tts.ElevenLabs", return_value=fake_client)
    run = mocker.patch("holler.tts.subprocess.run")
    run.return_value = MagicMock(returncode=0)

    speak("empty audio path")

    run.assert_called_once()
```

- [ ] **Step 5.2: Run tests — verify they fail**

Run: `.venv/bin/pytest tests/test_tts.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'holler.tts'`

- [ ] **Step 5.3: Implement `holler/tts.py`**

```python
"""Text-to-speech with ElevenLabs primary and macOS `say` fallback."""

from __future__ import annotations

import logging
import subprocess

import numpy as np
import sounddevice as sd
from elevenlabs.client import ElevenLabs

from holler.config import load_config
from holler.markdown import strip_markdown

log = logging.getLogger(__name__)

_SAMPLE_RATE = 22050  # matches pcm_22050 output format


def speak(text: str) -> None:
    """Speak `text` via ElevenLabs Flash, falling back to macOS `say`.

    Never raises — a hook handler must never crash Claude Code. On total
    failure the function logs an error and returns silently.
    """
    clean = strip_markdown(text).strip()
    if not clean:
        return

    cfg = load_config()

    if cfg.elevenlabs_api_key:
        try:
            _speak_elevenlabs(clean, cfg.elevenlabs_api_key, cfg.voice_id, cfg.tts_model)
            return
        except Exception as e:  # noqa: BLE001 — we want to catch everything
            log.warning("elevenlabs tts failed: %s; falling back to `say`", e)

    try:
        subprocess.run(["say", clean], check=False)
    except Exception as e:  # noqa: BLE001
        log.error("tts fallback `say` also failed: %s", e)


def _speak_elevenlabs(text: str, api_key: str, voice_id: str, model_id: str) -> None:
    client = ElevenLabs(api_key=api_key)
    audio_iter = client.text_to_speech.convert(
        text=text,
        voice_id=voice_id,
        model_id=model_id,
        output_format="pcm_22050",
    )
    pcm = b"".join(audio_iter)
    if not pcm:
        raise RuntimeError("elevenlabs returned empty pcm")
    audio = np.frombuffer(pcm, dtype=np.int16)
    sd.play(audio, samplerate=_SAMPLE_RATE)
    sd.wait()
```

- [ ] **Step 5.4: Run tests — verify they pass**

Run: `.venv/bin/pytest tests/test_tts.py -v`
Expected: All 8 tests PASS.

- [ ] **Step 5.5: Commit**

```bash
git add holler/tts.py tests/test_tts.py
git commit -m "Add TTS with ElevenLabs and `say` fallback"
```

---

## Task 6: voice-alert dispatcher (`holler/cli/voice_alert.py`)

Reads JSON from stdin, dispatches on `hook_event_name`, extracts the text to summarize, calls `summarize` + `speak`. Fallback chain: missing text → skip; summarize returns None → canned string; speak fails → log only.

**Transcript handling:** Claude Code writes `transcript_path` in the hook payload pointing at a JSONL file whose last line is the last assistant turn. We read the last line, parse it, and extract the message text.

**Files:**
- Create: `holler/cli/voice_alert.py`
- Create: `tests/test_voice_alert.py`

- [ ] **Step 6.1: Write failing tests**

Create `tests/test_voice_alert.py`:

```python
"""Tests for the voice-alert hook dispatcher."""

import io
import json
from pathlib import Path

import pytest

from holler.cli.voice_alert import handle_event, main


def _make_transcript(tmp_path: Path, messages: list[dict]) -> Path:
    """Write a JSONL transcript file and return its path."""
    path = tmp_path / "transcript.jsonl"
    path.write_text("\n".join(json.dumps(m) for m in messages) + "\n")
    return path


# ─── handle_event ──────────────────────────────────────────────────────────


def test_stop_event_speaks_summary(mocker, tmp_path):
    transcript = _make_transcript(
        tmp_path,
        [
            {"role": "user", "content": "refactor tax service"},
            {"role": "assistant", "content": "Refactored tax service and updated 18 tests."},
        ],
    )
    summarize = mocker.patch("holler.cli.voice_alert.summarize", return_value="Done. 18 tests pass.")
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "Stop", "transcript_path": str(transcript)})

    summarize.assert_called_once()
    args, _ = summarize.call_args
    assert "Refactored tax service" in args[0]  # text being summarized
    speak.assert_called_once_with("Done. 18 tests pass.")


def test_subagent_stop_speaks_summary(mocker, tmp_path):
    transcript = _make_transcript(
        tmp_path,
        [{"role": "assistant", "content": "Reviewer flagged two security issues."}],
    )
    mocker.patch(
        "holler.cli.voice_alert.summarize",
        return_value="Subagent flagged two security issues.",
    )
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "SubagentStop", "transcript_path": str(transcript)})

    speak.assert_called_once_with("Subagent flagged two security issues.")


def test_notification_uses_message_field(mocker):
    summarize = mocker.patch(
        "holler.cli.voice_alert.summarize",
        return_value="Waiting for approval to run migrations.",
    )
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event(
        {
            "hook_event_name": "Notification",
            "message": "Claude needs your permission to use Bash: python manage.py migrate",
        }
    )

    summarize.assert_called_once()
    args, _ = summarize.call_args
    assert "permission to use Bash" in args[0]
    speak.assert_called_once_with("Waiting for approval to run migrations.")


def test_unknown_event_is_noop(mocker):
    summarize = mocker.patch("holler.cli.voice_alert.summarize")
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "PreToolUse"})

    summarize.assert_not_called()
    speak.assert_not_called()


def test_summarize_failure_uses_canned_string(mocker, tmp_path):
    transcript = _make_transcript(tmp_path, [{"role": "assistant", "content": "something"}])
    mocker.patch("holler.cli.voice_alert.summarize", return_value=None)
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "Stop", "transcript_path": str(transcript)})

    speak.assert_called_once()
    spoken = speak.call_args.args[0]
    assert spoken  # non-empty
    assert "claude" in spoken.lower() or "done" in spoken.lower()


def test_missing_transcript_path_uses_canned_string(mocker):
    mocker.patch("holler.cli.voice_alert.summarize")
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "Stop"})

    speak.assert_called_once()


def test_nonexistent_transcript_file_uses_canned_string(mocker):
    speak = mocker.patch("holler.cli.voice_alert.speak")
    mocker.patch("holler.cli.voice_alert.summarize")

    handle_event({"hook_event_name": "Stop", "transcript_path": "/nope/does/not/exist.jsonl"})

    speak.assert_called_once()


def test_empty_transcript_uses_canned_string(mocker, tmp_path):
    transcript = tmp_path / "empty.jsonl"
    transcript.write_text("")
    mocker.patch("holler.cli.voice_alert.summarize")
    speak = mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "Stop", "transcript_path": str(transcript)})

    speak.assert_called_once()


def test_non_string_content_is_stringified(mocker, tmp_path):
    # Claude Code sometimes emits content as a list of blocks
    transcript = _make_transcript(
        tmp_path,
        [{"role": "assistant", "content": [{"type": "text", "text": "block content"}]}],
    )
    summarize = mocker.patch("holler.cli.voice_alert.summarize", return_value="summary")
    mocker.patch("holler.cli.voice_alert.speak")

    handle_event({"hook_event_name": "Stop", "transcript_path": str(transcript)})

    args, _ = summarize.call_args
    assert "block content" in args[0]


# ─── main() ────────────────────────────────────────────────────────────────


def test_main_reads_stdin(mocker):
    dispatch = mocker.patch("holler.cli.voice_alert.handle_event")
    mocker.patch("sys.stdin", io.StringIO('{"hook_event_name": "Stop"}'))

    main()

    dispatch.assert_called_once_with({"hook_event_name": "Stop"})


def test_main_malformed_json_is_noop(mocker):
    dispatch = mocker.patch("holler.cli.voice_alert.handle_event")
    mocker.patch("sys.stdin", io.StringIO("not json"))

    main()  # must not raise

    dispatch.assert_not_called()


def test_main_empty_stdin_is_noop(mocker):
    dispatch = mocker.patch("holler.cli.voice_alert.handle_event")
    mocker.patch("sys.stdin", io.StringIO(""))

    main()

    dispatch.assert_not_called()
```

- [ ] **Step 6.2: Run tests — verify they fail**

Run: `.venv/bin/pytest tests/test_voice_alert.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'holler.cli.voice_alert'`

- [ ] **Step 6.3: Implement `holler/cli/voice_alert.py`**

```python
"""Claude Code hook dispatcher — reads event JSON, summarizes, speaks."""

from __future__ import annotations

import json
import logging
import sys
from pathlib import Path

from holler.summarize import summarize
from holler.tts import speak

log = logging.getLogger(__name__)

_PROMPTS = {
    "Stop": "Summarize what the agent just did in 12 words or fewer. Be terse and factual. No preamble.",
    "SubagentStop": "Summarize what the subagent produced in 10 words or fewer. Be terse. No preamble.",
    "Notification": "Restate the permission request in 10 words or fewer. Be terse. No preamble.",
}

_CANNED = {
    "Stop": "Claude is done.",
    "SubagentStop": "Subagent finished.",
    "Notification": "Claude needs your attention.",
}


def _extract_last_assistant_text(transcript_path: str | None) -> str | None:
    """Read the JSONL transcript and return the last assistant message as text."""
    if not transcript_path:
        return None
    path = Path(transcript_path)
    if not path.exists():
        return None
    try:
        lines = path.read_text().splitlines()
    except OSError:
        return None

    # Walk backwards to find the last assistant entry
    for raw in reversed(lines):
        raw = raw.strip()
        if not raw:
            continue
        try:
            entry = json.loads(raw)
        except json.JSONDecodeError:
            continue
        if entry.get("role") != "assistant":
            continue
        return _content_to_text(entry.get("content"))

    return None


def _content_to_text(content: object) -> str:
    """Handle string or list-of-blocks content shapes."""
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts: list[str] = []
        for block in content:
            if isinstance(block, dict) and "text" in block:
                parts.append(str(block["text"]))
        return "\n".join(parts)
    return str(content or "")


def handle_event(event: dict) -> None:
    """Route a single Claude Code hook event to its speaker."""
    name = event.get("hook_event_name")
    if name not in _PROMPTS:
        return  # unknown / unsupported event: silent no-op

    prompt = _PROMPTS[name]
    canned = _CANNED[name]

    if name == "Notification":
        source_text = event.get("message", "") or ""
    else:
        source_text = _extract_last_assistant_text(event.get("transcript_path")) or ""

    if not source_text.strip():
        speak(canned)
        return

    summary = summarize(source_text, prompt)
    speak(summary or canned)


def main() -> None:
    """Entry point — read JSON from stdin and dispatch."""
    raw = sys.stdin.read().strip()
    if not raw:
        return
    try:
        event = json.loads(raw)
    except json.JSONDecodeError as e:
        log.warning("voice-alert: could not parse stdin as JSON: %s", e)
        return
    handle_event(event)


if __name__ == "__main__":
    main()
```

- [ ] **Step 6.4: Run tests — verify they pass**

Run: `.venv/bin/pytest tests/test_voice_alert.py -v`
Expected: All 12 tests PASS.

- [ ] **Step 6.5: Full suite check**

Run: `.venv/bin/pytest -v`
Expected: All tests from all previous tasks still PASS.

- [ ] **Step 6.6: Commit**

```bash
git add holler/cli/voice_alert.py tests/test_voice_alert.py
git commit -m "Add voice-alert hook dispatcher"
```

---

## Task 7: `bin/voice-alert` executable wrapper

Thin shell script Claude Code's hooks config invokes directly. Activates the holler venv and runs `python -m holler.cli.voice_alert`.

**Files:**
- Create: `bin/voice-alert`

- [ ] **Step 7.1: Create the wrapper**

```bash
#!/usr/bin/env bash
# holler voice-alert — Claude Code hook handler.
# Invoked by Claude Code with a JSON hook payload on stdin.
set -euo pipefail

HOLLER_HOME="${HOLLER_HOME:-$HOME/.holler}"
VENV="$HOLLER_HOME/.venv"

if [ ! -x "$VENV/bin/python" ]; then
    echo "holler: venv not found at $VENV — did you run install.sh?" >&2
    exit 1
fi

exec "$VENV/bin/python" -m holler.cli.voice_alert
```

Save as `bin/voice-alert`.

- [ ] **Step 7.2: Mark it executable**

Run: `chmod +x bin/voice-alert`

- [ ] **Step 7.3: Smoke-test the wrapper locally**

With the project venv already created and package installed:

```bash
.venv/bin/pip install -e .
HOLLER_HOME="$PWD" ln -sf .venv "$PWD/.venv" 2>/dev/null || true
echo '{"hook_event_name": "Stop"}' | HOLLER_HOME="$PWD" bin/voice-alert
```

Expected: Process exits 0. If ELEVENLABS_API_KEY is unset, you'll hear the macOS `say` fallback speak "Claude is done."

Note: set `HOLLER_HOME=$PWD` only for this smoke test — the real flow uses `~/.holler`.

- [ ] **Step 7.4: Commit**

```bash
git add bin/voice-alert
git commit -m "Add bin/voice-alert shell wrapper"
```

---

## Task 8: Hooks snippet (`hooks/settings.snippet.json`)

The JSON fragment that consumers merge into their `~/.claude/settings.json`.

**Files:**
- Create: `hooks/settings.snippet.json`

- [ ] **Step 8.1: Create the snippet**

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.holler/bin/voice-alert"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.holler/bin/voice-alert"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.holler/bin/voice-alert"
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 8.2: Verify JSON validity**

Run: `python -m json.tool hooks/settings.snippet.json > /dev/null && echo ok`
Expected: `ok`

- [ ] **Step 8.3: Commit**

```bash
git add hooks/settings.snippet.json
git commit -m "Add Claude Code hooks snippet for voice-alert"
```

---

## Task 9: `install.sh` (Milestone-1 subset)

Single-Mac install. No launchd / Accessibility / microphone handling in this milestone — those belong to Plan B.

**Files:**
- Create: `install.sh`

- [ ] **Step 9.1: Write the installer**

```bash
#!/usr/bin/env bash
# holler installer — Milestone 1 (voice-alert only; no dictation daemon yet).
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
HOLLER_HOME="$HOME/.holler"
CONFIG_DIR="$HOME/.config/holler"
CLAUDE_SETTINGS="$HOME/.claude/settings.json"

say() { printf "\033[1;34m[holler]\033[0m %s\n" "$*"; }
ok()  { printf "\033[0;32m  ✓\033[0m %s\n" "$*"; }
err() { printf "\033[0;31m  ✗ %s\033[0m\n" "$*" >&2; }

say "Checking dependencies..."

# Python 3.11+
if ! command -v python3 >/dev/null 2>&1; then
    err "python3 not found"; exit 1
fi
PY_VER=$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
if python3 -c 'import sys; sys.exit(0 if sys.version_info >= (3,11) else 1)'; then
    ok "python3 $PY_VER"
else
    err "python3 >= 3.11 required (found $PY_VER)"; exit 1
fi

# claude CLI
if command -v claude >/dev/null 2>&1; then
    ok "claude CLI present"
else
    err "claude CLI not on PATH — install Claude Code first"; exit 1
fi

# ffmpeg (elevenlabs SDK needs it for decoding)
if command -v ffmpeg >/dev/null 2>&1; then
    ok "ffmpeg present"
else
    err "ffmpeg missing — run: brew install ffmpeg"; exit 1
fi

# Create holler home + venv
say "Setting up venv at $HOLLER_HOME/.venv..."
mkdir -p "$HOLLER_HOME"
if [ ! -d "$HOLLER_HOME/.venv" ]; then
    python3 -m venv "$HOLLER_HOME/.venv"
    ok "venv created"
else
    ok "venv exists"
fi

"$HOLLER_HOME/.venv/bin/pip" install --quiet --upgrade pip
"$HOLLER_HOME/.venv/bin/pip" install --quiet -e "$SCRIPT_DIR"
ok "holler installed into venv"

# Link bin/voice-alert into HOLLER_HOME/bin
mkdir -p "$HOLLER_HOME/bin"
ln -sf "$SCRIPT_DIR/bin/voice-alert" "$HOLLER_HOME/bin/voice-alert"
ok "bin/voice-alert linked"

# API key
say "Configuring ElevenLabs..."
mkdir -p "$CONFIG_DIR"
ENV_FILE="$CONFIG_DIR/env"
if [ -z "${ELEVENLABS_API_KEY:-}" ] && [ ! -f "$ENV_FILE" ]; then
    printf "  Paste your ELEVENLABS_API_KEY (or press Enter to skip and use macOS \`say\` only): "
    read -r KEY || KEY=""
    if [ -n "$KEY" ]; then
        printf 'ELEVENLABS_API_KEY="%s"\n' "$KEY" > "$ENV_FILE"
        chmod 600 "$ENV_FILE"
        ok "key saved to $ENV_FILE"
    else
        ok "skipped — \`say\` fallback will be used"
    fi
elif [ -f "$ENV_FILE" ]; then
    ok "$ENV_FILE already exists"
else
    # env var is set but file isn't — persist it
    printf 'ELEVENLABS_API_KEY="%s"\n' "$ELEVENLABS_API_KEY" > "$ENV_FILE"
    chmod 600 "$ENV_FILE"
    ok "key persisted to $ENV_FILE"
fi

# Merge hooks into ~/.claude/settings.json (preserving existing keys)
say "Wiring Claude Code hooks..."
mkdir -p "$(dirname "$CLAUDE_SETTINGS")"
if [ ! -f "$CLAUDE_SETTINGS" ]; then
    echo "{}" > "$CLAUDE_SETTINGS"
fi

"$HOLLER_HOME/.venv/bin/python" - <<PYEOF
import json, os
from pathlib import Path

settings_path = Path(os.path.expanduser("$CLAUDE_SETTINGS"))
snippet_path = Path("$SCRIPT_DIR/hooks/settings.snippet.json")

settings = json.loads(settings_path.read_text() or "{}")
snippet = json.loads(snippet_path.read_text())

settings.setdefault("hooks", {})
for event, entries in snippet["hooks"].items():
    existing = settings["hooks"].setdefault(event, [])
    # Avoid duplicate holler entries on re-install
    non_holler = [e for e in existing if not any(
        h.get("command", "").endswith("/.holler/bin/voice-alert")
        for h in e.get("hooks", [])
    )]
    settings["hooks"][event] = non_holler + entries

settings_path.write_text(json.dumps(settings, indent=2) + "\n")
print(f"  hooks merged into {settings_path}")
PYEOF
ok "hooks configured"

# Self-test
say "Running self-test..."
if echo '{"hook_event_name":"Stop"}' | "$HOLLER_HOME/bin/voice-alert"; then
    ok "self-test passed — you should have heard audio"
else
    err "self-test failed"; exit 1
fi

say "Install complete. Restart Claude Code to activate hooks."
```

- [ ] **Step 9.2: Mark executable**

Run: `chmod +x install.sh`

- [ ] **Step 9.3: Dry-run sanity check**

Run: `bash -n install.sh && echo ok`
Expected: `ok` (syntax-valid bash).

- [ ] **Step 9.4: Commit**

```bash
git add install.sh
git commit -m "Add install.sh for Milestone 1"
```

---

## Task 10: README install section + manual E2E verification

**Files:**
- Modify: `README.md` (replace the "Coming soon" install section)

- [ ] **Step 10.1: Update `README.md`**

Replace the `## Install` section with:

```markdown
## Install (Milestone 1 — voice alerts only)

Prerequisites:

- macOS (Apple Silicon)
- Python 3.11+
- [Claude Code](https://claude.com/download) installed and authenticated
- [ffmpeg](https://ffmpeg.org/) — `brew install ffmpeg`
- (Optional) An [ElevenLabs](https://elevenlabs.io) API key. Without one, holler falls back to the macOS `say` voice.

Install:

```bash
git clone https://github.com/kathanshah/holler.git
cd holler
./install.sh
```

The installer:

- Creates a venv at `~/.holler/.venv`
- Prompts for your `ELEVENLABS_API_KEY` (skip for `say`-only mode)
- Writes config to `~/.config/holler/env`
- Merges Claude Code hooks into `~/.claude/settings.json` (preserving existing entries)
- Runs a self-test — you'll hear "Claude is done" if everything works

**Restart Claude Code** for the hooks to take effect.

## What happens next

- Claude Code finishes a turn → short summary plays via ElevenLabs Flash (or `say` fallback)
- Claude Code asks for permission → you hear a restatement of the request
- A subagent finishes → you hear what it produced

Dictation (`voice-dictate`) is planned for Milestone 2.

## Uninstall

```bash
./uninstall.sh          # coming soon
```

For now, manual: remove `~/.holler/`, `~/.config/holler/`, and the `hooks.*.holler` entries in `~/.claude/settings.json`.
```

- [ ] **Step 10.2: Run full test suite one last time**

Run: `.venv/bin/pytest -v`
Expected: all tests PASS.

- [ ] **Step 10.3: Manual E2E — real Claude Code integration**

1. Run: `./install.sh` (prompts for ElevenLabs key if set; else uses `say`)
2. Restart Claude Code (`Cmd+Q` then relaunch)
3. In any repo, send Claude Code any prompt (e.g. "list the files in this directory")
4. Wait for Claude's response to finish → you should hear a spoken summary within ~2s
5. Ask Claude something that requires permission (e.g. "run `ls` using Bash") → after it asks, you should hear the permission request spoken
6. If any step fails, check `~/.holler/.venv/bin/python -m holler.cli.voice_alert` logs or re-run the self-test: `echo '{"hook_event_name":"Stop"}' | ~/.holler/bin/voice-alert`

Do not commit until all 6 steps pass.

- [ ] **Step 10.4: Commit**

```bash
git add README.md
git commit -m "Document Milestone 1 install and usage"
git push
```

---

## Milestone 1 done criteria

- [ ] All pytest tests pass (`pytest -v` green)
- [ ] `./install.sh` runs cleanly on a fresh Mac
- [ ] Self-test prints "self-test passed" and plays audio
- [ ] Live Claude Code session: Stop, SubagentStop, Notification all trigger a voice summary within ~2s
- [ ] Both ElevenLabs success path AND `say` fallback path are observed working
- [ ] README install section reflects reality
- [ ] All commits are authored as `kathan <kathan@stockypro.com>` (no Claude attribution anywhere)

## Known risks to verify during execution

| Risk | How to verify | Fallback if broken |
|---|---|---|
| `claude -p --model claude-haiku-4-5` works under user's auth | Manual E2E Step 3 | If `--model` doesn't work, fall back to `claude -p` without the flag (uses whatever model is configured) |
| Claude Code transcript shape (`content` string vs. list of blocks) | Existing test `test_non_string_content_is_stringified` covers both; confirm live shape matches in Manual E2E | If shape differs, adjust `_content_to_text` and add a test |
| ElevenLabs Flash voice quality on short technical summaries | Manual E2E Steps 3-5 — listen carefully | If quality is poor, switch default to `eleven_turbo_v2_5` via `HOLLER_TTS_MODEL` |
| Hooks snippet merges cleanly into user's existing `~/.claude/settings.json` | Run installer on a machine with existing hooks, verify originals are preserved | Debug the JSON-merge block in `install.sh` |

---

## After Milestone 1

When this plan is done and voice alerts are working in real use, write Plan B (`docs/plan-voice-dictate.md`) for:

- `holler/stt.py` (ElevenLabs Scribe + VAD pre-trim)
- `bin/voice-dictate` (hotkey daemon)
- `launchd/com.kathanshah.holler.voice-dictate.plist`
- `install.sh` additions — Accessibility permission prompt, launchd load
- Tests covering STT + hotkey event handling
