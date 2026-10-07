# Heard Windows Port — Phase 1 Plan (headless core)

## Context
Heard is a macOS-only voice companion for Claude Code / Codex. The user cloned the repo on Windows and asked what needs porting. The plan implements Phase 1: a headless core where `heard say "hello"`, `heard run <cmd>`, and the Claude Code hook work on Windows with no UI.

The pasted analysis in the conversation was written from a macOS read of the repo and I verified/corrected it against the actual code (two key corrections: `pynput` is already used for hotkey parsing and `AppKit` imports are lazy in `hotkey.py`; the repo has **no** `heard/platform/` layer yet).

## Key findings driving the design

### What already works cross-platform (no change needed)
- `config.py` uses `platformdirs` — paths resolve to `%APPDATA%\heard` on Windows automatically.
- TTS backends (`kokoro.py`, `elevenlabs.py`, `speechify.py`) are pure HTTP/ONNX — cross-platform. Kokoro writes WAV files via `soundfile`.
- `hook.py`, `cli.py`, `adapters/claude_code.py`, `adapters/codex.py` are platform-neutral.
- `harness.py`, `persona.py`, `multi_agent.py`, `templates.py`, `verbosity.py`, `agent_state.py`, `working_memory.py` are all pure Python.
- `tests/conftest.py` isolates config dirs; hotkey tests mock `AppKit` via `sys.modules` — the pattern is already proven.
- `audio_monitor.py` already returns `None` on non-macOS (`_load_coreaudio` catches `OSError`).
- `accessibility.py` already returns `True` / no-op on non-Darwin.

### What actually breaks on Windows
1. **IPC / sockets**: `socket.AF_UNIX` in `daemon.py` (server, `_socket_accepts_ping`, `_prepare_runtime_for_bind`, `_terminate_pid`), `client.py` (send/request/is_alive), `push_to_talk.py`, `voice_service.py`, `home_window.py`, `ui.py`, `accessibility.py` (4 duplications of the Power voice-service poke).
2. **File locks**: `fcntl.flock` in `client.py` (spawn lock), `history.py` (truncate rewrite), `spoken.py` (session state lock). `wrapper.py` also uses `fcntl.ioctl(TIOCSWINSZ)`.
3. **Process lifecycle**: `client.py` uses `pgrep`, `SIGTERM`/`SIGKILL`, `os.kill(pid, 0)`, `start_new_session=True`. `updater.py` same.
4. **Audio playback**: `daemon.py` shells out to `/usr/bin/afplay` with optional `-r <rate>`. Kokoro WAVs need a player; ElevenLabs/Speechify MP3s need a player with speed support.
5. **Top-level imports**: `ui.py` imports `rumps` at top level; `settings_widgets.py` imports `AppKit`, `Foundation` at top level. On Windows these crash at import.
6. **Dependencies**: `pyproject.toml` lists macOS-only deps without markers.

## Architecture: `heard/platform/` abstraction layer

New package `heard/platform/` with one module per seam — all callers in `daemon.py`, `client.py`, `history.py`, `spoken.py` import from here instead of stdlib/OS primitives.

```
heard/platform/
  __init__.py          # re-exports; also a single place for `is_windows` / `is_darwin`
  sockets.py           # AF_UNIX (macOS) vs TCP localhost+token (Windows) server/client/poke
  locks.py             # fcntl.flock (macOS) vs portalocker (cross-platform)
  process.py           # os.kill/SIGTERM/SIGKILL/pgrep vs psutil-based equivalents
  playback.py          # afplay vs sounddevice/soundfile decode (Kokoro WAV) vs ffplay fallback
```

### Socket seam (highest priority)
- `create_server(path)` → on macOS, `AF_UNIX` + bind + chmod 0o600 + listen; on Windows, `socket(AF_INET, SOCK_STREAM)` bound to `127.0.0.1:<port>` where port is derived from a stable hash of `path`, with a token file at `path + ".token"` so callers authenticate.
- `connect_send(path, msg)`, `ping(path)` → same address-family selection.
- **Power-voice pokes**: 4 duplications (`push_to_talk.py:36`, `voice_service.py:190`, `home_window.py:1246`, `ui.py:1038`) collapse to a single `platform.sockets.poke(path, action)`.
- Daemon's `_prepare_runtime_for_bind` / `_socket_accepts_ping` / `_unlink_if_present` stay in `daemon.py` but call `platform.sockets`.

### Lock seam
- `FileLock(path)` context manager. macOS: `fcntl.flock(LOCK_EX|LOCK_NB)`. Windows: `portalocker.Lock` (exclusive, non-blocking). `history.py:commit_checkpoint_and_prune` and `spoken.py:_SessionLock` become `with platform.locks.FileLock(path):`.
- `client.py`'s `_SPAWN_LOCK_PATH` lock uses the same abstraction.
- Note: `history.append` does NOT flock (OS atomic appends ≤ PIPE_BUF) — preserve that. `wrapper.py`'s `fcntl.ioctl(TIOCSWINSZ)` is PTY-only; V1 stubs the wrapper.

### Process seam
- `pid_is_running(pid)`, `terminate_pid(pid)`, `spawn_daemon(argv)`, `other_daemon_pids()`. macOS: `os.kill(pid,0)`, `os.kill(pid,SIGTERM)`, `subprocess.Popen(..., start_new_session=True)`, `pgrep -f`. Windows: `psutil.pid_exists(pid)`, `psutil.Process(pid).terminate()` + wait/kill, `subprocess.Popen(..., creationflags=CREATE_NEW_PROCESS_GROUP)`, `psutil.process_iter(["name","cmdline"])` filter.
- `client.py:start_headless_daemon` and `daemon.py:_prepare_runtime_for_bind` both use these.

### Playback seam
- `play(path, speed, cancel)` + `interrupt()`. macOS: `/usr/bin/afplay` subprocess (current behavior). Windows: `sounddevice` playing decoded float samples from WAV/MP3, with `tempfile` cleanup. Kokoro WAV decoded via `soundfile` (already a dep). ElevenLabs/Speechify MP3 → `ffplay` subprocess with `-af atempo` (ffmpeg ships with many Windows installs). V1 default: `sounddevice` for Kokoro WAVs; `ffplay` as the universal fallback. Fallback chain mirrors daemon's `_speak` path.

## Dependency changes (`pyproject.toml`)

```toml
dependencies = [
    # ... existing ...
    "portalocker>=3.0",         # cross-platform file locks
    "psutil>=5.9",              # cross-platform process mgmt
]

[project.optional-dependencies]
macos = [
    "rumps>=0.4; sys_platform == 'darwin'",
    "pyobjc-framework-Cocoa>=10.0; sys_platform == 'darwin'",
    "pyobjc-framework-WebKit>=10.0; sys_platform == 'darwin'",
    "pyobjc-framework-Quartz>=10.0; sys_platform == 'darwin'",
    "pyobjc-framework-ApplicationServices>=10.0; sys_platform == 'darwin'",
]
```

Keep macOS base deps as-is for existing macOS users; add `portalocker`/`psutil` to base; guard macOS-only packages with markers.

## Files to change (Phase 1)

1. **`heard/platform/__init__.py`** — new; `is_windows`, `is_darwin`, `SOCKET_KIND`.
2. **`heard/platform/sockets.py`** — new; `create_server`, `connect_send`, `ping`, `poke`.
3. **`heard/platform/locks.py`** — new; `FileLock`.
4. **`heard/platform/process.py`** — new; `pid_is_running`, `terminate_pid`, `spawn_daemon`, `other_daemon_pids`.
5. **`heard/platform/playback.py`** — new; `play`, `interrupt`.
6. **`heard/daemon.py`** — replace `AF_UNIX`/`os.chmod`/`SIGTERM`-reaping/`afplay` with `platform.*` calls. Preserve structured `_log`, queue drain, cancel semantics, mute/spool logic, `_voice_suppress`.
7. **`heard/client.py`** — replace `AF_UNIX` client, `fcntl.flock` spawn lock, `pgrep`/`SIGTERM` orphan reaping, `start_new_session=True` with `platform.*`. Preserve memory-pressure guard, spawn lock semantics, `_wait_for_daemon`.
8. **`heard/history.py`** — replace `fcntl.flock` in `commit_checkpoint_and_prune` with `platform.locks.FileLock`.
9. **`heard/spoken.py`** — replace `fcntl.flock` in `_SessionLock` with `platform.locks.FileLock`.
10. **`heard/ui.py`** — lazy-import `rumps` inside `run()` guarded by `sys.platform == "darwin"`; non-Darwin `run()` falls back to a no-op or CLI help (full pystray tray = Phase 2).
11. **`heard/settings_widgets.py`** — lazy-import guards for `AppKit`/`Foundation` (same pattern as `prompt_window.py`).
12. **`heard/service.py`** — stub `install`/`uninstall`/`is_installed` no-ops on non-Darwin (Task Scheduler = Phase 2).
13. **`heard/wrapper.py`** — V1: degenerate `os.execvp` passthrough only; PTY/ANSI/tee loop = Phase 2.
14. **`heard/cli.py`** — guard `service` subcommands on platform; add `heard diagnose` that prints which platform pieces are available.
15. **`tests/conftest.py`** — no change needed (already isolates config dirs; hotkey tests already mock `AppKit`).
16. **`.github/workflows/ci.yml`** — add `windows-latest` runner to the test matrix with the same `ruff`/`pytest` gates.
17. **`pyproject.toml`** — add `portalocker`, `psutil` base deps; add `[project.optional-dependencies] macos` markers for `rumps`/`pyobjc-*`.

## Explicitly out of scope for Phase 1
- System tray (`pystray`), global hotkeys, notifications (`winotify`), Task Scheduler autostart, Mic-capture auto-silence (Windows WASAPI via `pycaw`), onboarding WKWebView (`pywebview`), installer/packaging (PyInstaller + Inno Setup), Codex Desktop log tailing, update pipeline (notarization removed; helper logic ports trivially).

## Verification
1. `ruff check heard/ tests/` passes (Linux dev box).
2. `pytest -q` passes — update tests asserting `AF_UNIX`/`fcntl` internals; `test_client_orphan_reap.py` must assert new process/lock abstraction semantics.
3. Headless smoke on Windows: `python -m heard daemon --help` starts without importing `rumps`/`AppKit`/`Quartz`.
4. `heard config set voice george` round-trips under `%APPDATA%\heard\config.yaml`.
5. `heard say "hello"` starts daemon, synth plays (Kokoro WAV via `sounddevice`, ElevenLabs MP3 via `ffplay`).
6. Claude Code hook fires (install on Windows path with `shlex.quote`-safe command, no `.app` PYTHONHOME wrap).
7. macOS CI gate still passes (macOS extra keeps existing macOS users unaffected; base deps unchanged for them).
