---
name: python-windows-audio
description: Fix Python real-time audio on Windows (sounddevice/WASAPI, miniaudio, pycaw mixer volume, CUDA DLLs for faster-whisper, cp1252 console crashes). Use when playback is silent, mic stream dies, TTS volume is tiny, or transcription falls back to CPU.
---

# Python Windows Audio Field Manual

## sounddevice / PortAudio

- Device selection: list with `sd.query_devices()`; pick by name substring and pin its index in config (`null` = default is fragile across replugs).
- Callbacks run on the audio thread: never allocate heavily, never log per-frame there; push bytes into a `queue.Queue`/deque.
- An InputStream whose callback raised once can go permanently silent — probe devices at boot, and treat zero frames over N seconds as pipeline death (restart stream, then process).
- Sample rate: capture at 16000 Hz mono int16 for VAD/whisper; let PortAudio resample only if the device lacks 16 kHz (check `query_devices` default_samplerate).

## Playback (edge-tts mp3 → speakers)

- Decode mp3 with `miniaudio.decode(...)` to int16 at the TTS native 24 kHz; chunked writes of 30 ms into `sd.OutputStream` allow cooperative interruption between chunks.
- Windows Mixer remembers per-app volume per exe path: python.exe can be stuck at 1% with NO error and no sound. Fix via pycaw `AudioUtilities.GetAllSessions()` setting `SimpleAudioVolume.SetMasterVolume(1.0)` — do it lazily on first audible session, not at import.
- Beeps/tones: synthesize numpy sine, no network needed; keep a `beep()` path working even when TTS API is down so wake/sleep cues always function.

## faster-whisper / ctranslate2 CUDA

- A CUDA build that constructs fine can still explode on first encode (missing cublas/cudnn DLLs). Probe: run one real transcription of 0.5 s silence inside init; on ANY exception rebuild on CPU int8.
- Expose pip nvidia wheels via `os.add_dll_directory(<site-packages>/nvidia/*/bin)` BEFORE constructing the model; also prepend those dirs to PATH.
- GPU large-v3 float16 gives ~0.3 s for short Arabic turns; CPU int8 small is the fallback, not the target.

## Console encoding (cp1252 killer)

- Printing Arabic to stdout/stderr defaults to cp1252 → UnicodeEncodeError can kill a broadcast mid-stream. At startup: `sys.stdout.reconfigure(encoding="utf-8", errors="replace")` (same for stderr); file logs use `logging.FileHandler(..., encoding="utf-8")`; JSON dumps use `ensure_ascii=False`.
- Verify actual bytes with Python (`Path.read_bytes`) — PowerShell console display may show `?` even when the file is correct UTF-8.

## Process supervision on Windows

- Autostart: shortcut in `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup` running a .bat that loads secrets from env file then execs venv python.
- Watchdog pattern: scheduled task every minute checks lock-file PID liveness (`ctypes OpenProcess`), relaunches minimized if dead; count restarts to detect crash loops.
