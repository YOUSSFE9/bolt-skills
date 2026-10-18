---
name: jev-dashboard-design
description: Design system for the Jev control dashboard (dark glassmorphism, live voice states, Arabic RTL). Use when building or polishing dashboard UI, live conversation panels, mic/voice state views, or any visual component in the Jev app.
---

# Jev Dashboard Design System

## Visual language (established)

- Theme: near-black background (#0a0e17 family), glass panels `rgba(255,255,255,.04-.08)` + `backdrop-filter: blur()`, 1px borders at ~8% white, radius 14-20px.
- Accent: single cyan/teal for "alive" states; red only for errors; amber for warnings. State colors must match voice service states: listening=calm pulse, thinking=shimmer/ellipsis, speaking=waveform animation, muted/asleep=dimmed.
- Typography: system stack; Arabic text needs `direction: rtl` on content nodes while keeping numerals/timers LTR (`unicode-bidi: plaintext` on mixed spans).

## Live voice panel rules

- Mic level meter driven by level events (~12 Hz): animate transform scaleX/scaleY or opacity, never layout properties; smooth with CSS transition ~80ms.
- Conversation log appends from live transcript (user, right-aligned) and streamed reply deltas (assistant, left); auto-scroll only when already at bottom.
- Streaming states: a thinking bubble shows the live partial text while the model writes; it is replaced by the final assistant message when the turn ends.
- Barge-in gets a visible momentary flash + inline marker in the transcript; errors surface as toast + persistent badge, never alert().
- Connection state is always visible as a small dot; reconnect with backoff and resync a hello snapshot.

## Layout

- Single-screen cockpit: header (system status: service state, brain endpoint health, model id), main split = conversation (fluid) + side rail (tasks/agents), footer for input controls.
- Responsive min width 360px; touch targets >= 44px; keyboard shortcut hints shown but optional.

## Motion & polish

- Animations <= 200ms ease-out; respect `prefers-reduced-motion`.
- Empty states are designed moments (short Arabic guidance line + icon), not blank areas.
- Numbers (latency ms, tokens) render tabular-nums.

## Implementation notes

- The Jev app frontend is vanilla JS + HTML/CSS (no framework) talking to a Rust (Tauri) backend via `invoke` + Tauri events (`voice-delta`, `live-partial`, `voice-barge`, ...) — not a WebSocket.
- Extract tokens to CSS variables first (`--bg, --panel, --accent, --ok, --err`); every new component consumes variables, no raw hex.
- Accessibility: aria-live="polite" on the conversation log; visible focus rings; contrast >= 4.5 for body text on glass.
