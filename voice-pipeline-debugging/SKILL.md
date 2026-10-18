---
name: voice-pipeline-debugging
description: Diagnose live voice-agent pipelines (VAD endpointing, speaker-ID gating, barge-in, STT/TTS latency, silent cloud fallback). Use when a voice layer ignores the owner, rejects their voice, dies silently, replies with wrong/long latency, or the live call loop breaks.
---

# Live Voice Pipeline Debugging

## Reference architecture

```
MicStream(16k mono) -> VAD -> collect_utterance(endpointing)
  -> [SpeakerVerifier gate (optional)]
  -> STT (Whisper-class, GPU with CPU fallback + domain lexicon prompt)
  -> LLM stream (SSE to an OpenAI-compatible endpoint / agent proxy)
  -> split_sentences -> streaming TTS (synthesize while the LLM is still writing)
  -> speak_and_watch(barge-in: short voiced-frames confirm + owner check)
State changes are broadcast (events/WS) so the UI mirrors reality.
```

## Symptom -> root cause table

| Symptom | Likely root cause | Evidence to check |
|---|---|---|
| Owner ignored ~half the time | speaker gate threshold too tight for single embedding; scores hover just below | score lines in logs; listen to rejected wavs |
| Service silent after some minutes | mic frame generator ended (async loop exited) -> process exits quietly | lifecycle log `run() returned normally`; fault log |
| No reply at all | brain unreachable / auth 401 | `error brain:` events; key not exported when started outside its launcher |
| Reply text appears but no sound | Windows mixer per-app volume stuck | pycaw session volume for the python exe |
| Latency silently 2x-5x the local baseline | agent gateway fell back to a CLOUD provider after an internal error (e.g. invalid request field rejected by the local provider) | agent log shows provider error + fallback; responses are much shorter/longer than local; measure with fallback disabled to see the truth |
| Echo/self-trigger | mic hears TTS playback | barge-in must require owner check + confirmation frames |
| Wake word never fires (Arabic) | English-trained wake model can't match Arabic pronunciation | prefer always-on + speaker gate for Arabic |

## Speaker-gate repair playbook

1. Mine rejected + accepted turns: build score distribution for true-owner audio.
2. Re-enroll with 3-5 varied utterances; store multiple centroids; decision = max cosine over centroids, not one average embedding.
3. Set threshold from data (`p10(accepted) - margin`), never a magic constant; log every decision with its score.
4. Gray zone policy: still transcribe but prefix a confirmation ("?" ) instead of ignoring — an ignored owner feels like a dead product.
5. Keep >= 0.5 s minimum audio and VAD-trim before embedding (silence dilutes similarity).

## Latency budget (target < 2.5 s speech-to-speech)

- end_turn_silence_ms ~800 is a tradeoff; adaptive endpointing helps (shorter after long monologues, longer when the user pauses mid-sentence).
- STT GPU ~0.3-0.8 s per short turn; first SSE token typically < 1 s locally; TTS synth per sentence ~0.3-0.6 s — sentence-split streaming TTS hides most of it.
- Kill cold starts: pin the model in memory (keep-alive), and trim the system prompt a gateway injects per turn (it is often 10x the user message).

## Rules

- Never let one bad turn kill the loop: wrap per-frame handling, keep the mic generator infinite, persist exceptions to lifecycle/fault logs.
- Single-instance lock file with PID liveness probe (on Windows, SO_REUSEADDR lets two servers share one port silently).
- Every state change broadcasts over events/WS (`state/transcript/reply_delta/barge_in/error`) so the dashboard mirrors reality.
- When measuring latency, ALWAYS verify which backend actually served (local vs cloud) before trusting the number.
