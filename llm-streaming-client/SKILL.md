---
name: llm-streaming-client
description: Debug OpenAI-compatible LLM streaming clients (SSE parsing, timeouts, tool calls, local routers and agent proxies). Use when streamed replies stall, deltas are empty, auth 401s, or when switching between a raw router endpoint and an agent-proxy endpoint.
---

# OpenAI-Compatible Streaming Client Debugging

## SSE essentials

- Parse only lines starting `data:`; strip exactly one leading space; stop at `[DONE]`.
- Deltas arrive in `choices[0].delta.content`; tool calls arrive as `choices[0].delta.tool_calls[]` fragments with index-based accumulation (`id`/`function.name` on first fragment, `function.arguments` concatenated across fragments).
- Always accumulate the full reply server-side even while yielding deltas — history must store the complete message.

## httpx specifics (Python)

- `client.stream("POST", ...)` needs the timeout set on the CLIENT constructor for connect+read phases; a 120 s read timeout is normal for slow local models, but connect timeout should be ~5 s to fail fast when the gateway is down.
- On non-200 inside a stream, read the body for the error JSON before raising; log status + truncated body.
- One client per request is fine locally; reuse AsyncClient for throughput later.

## Raw router vs agent proxy

- A raw model router returns bare completions; an agent proxy (e.g. Hermes-style) exposes the actual AGENT (tools, memory, persona) as an OpenAI-compatible endpoint — different port/URL, and it may silently fall back to cloud providers on internal errors.
- Verify which one you are hitting by asking something that requires tools or memory.
- Fallback chain pattern: try agent proxy → on connect error/timeouts fall back to raw router; always surface which path served each turn in events/logs.
- Local servers restarted with different env can silently change behavior (keep-alive, context size, thinking enabled); after any restart re-measure rather than trust old numbers.

## Auth pitfalls

- Keys may live only in an agent config dir or `.env` file, not exported in the service process: implement env-var-first then parse that file as fallback (never copy secrets into repo files).
- Empty Bearer header vs missing header behave differently across gateways; send `Authorization: Bearer <key>` whenever a key exists.

## Health probes

- `GET {base}/models` is a cheap liveness check at boot; record the result as an event so dashboards show brain reachability.
- Distinguish: connection refused (down) / 401 (key) / 404 model (wrong id) / 200-but-empty-deltas (router quirk — retry once without stream).
