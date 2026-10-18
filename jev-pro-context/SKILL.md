---
name: jev-pro-context
description: Project context for Jev Pro - a local-first Arabic voice assistant desktop app (Tauri 2 + vanilla JS frontend + Rust backend; local AI pipeline VAD -> STT -> agent gateway -> streaming TTS). Use BEFORE making any change anywhere in this project (app/, servers/, tools/, jev-hub/, jev-brain/, docs/).
---

# Jev Pro - Project Context

## What this project is

A local-first Arabic voice assistant: desktop app (Tauri 2 + vanilla JS + Rust) with a fully local
AI pipeline for a live voice call: mic -> VAD endpointing -> STT -> agent gateway (local LLM) ->
sentence-split streaming TTS with barge-in. Everything must keep working without internet.

## Repo layout (read the files, do not guess)

- `app/src/` - frontend: `main.js` (all UI logic), `index.html`, `styles.css`. Arabic RTL.
- `app/src-tauri/src/` - Rust backend: `main.rs`, `audio.rs` (mic + VAD + barge-in), `bridges.rs`
  (Tauri commands like ask_voice; event emitters voice-delta / live-partial / voice-barge),
  `vad.rs` (Silero VAD via tract-onnx).
- `servers/` - python side servers: `stt_server.py` (port 8001), `tts_server.py` (port 8002),
  `vad.py` shared VAD gate.
- `tools/` - diagnostics & launchers: `diag_voice_speed.py` (voice latency + config health),
  launcher scripts.
- `jev-hub/` - MCP task hub (agents coordination). `jev-brain/` - agent skills. `docs/` - state + decisions (Arabic).
- `AGENTS.md` at root - shared project rules. Read it plus the latest docs before big changes.

## Contracts you must not break

- `ask_voice` command contract: `invoke("ask_voice", {prompt, persona, sessionId, turn})`;
  streaming via `voice-delta` events carrying FULL accumulated text + turn number.
- Frontend streaming state: `activeTurn / activeThinkId / ttsStream {sid, gen, text, spoken, chain}`;
  stale events are dropped by comparing turn (and sid/gen for TTS).
- Sentence cut points for streaming TTS: `.!?...` plus Arabic punctuation; min 6 cleaned chars.
- Barge-in = real cancel: increment the generation counter, cancel the request, drop the thinking bubble.
- localStorage keys: `jev_model_choice_v1`, `jev_model_vis_v1`, `jev_sessions_v1`, `jev_autospeak`.

## Rules of engagement

- Speed is a feature: the live voice loop must stay under ~2 s speech-to-speech; never add blocking
  calls (network, subprocess) to the per-turn path.
- Voice turns use a FRESH gateway session each turn (no long conversation history accumulation).
- After any change touching the voice path, verify with the project's own tools/tests and record
  measured numbers - never report success without evidence.
- UI text is Arabic, RTL; keep jargon-free labels; numbers render with tabular-nums.
