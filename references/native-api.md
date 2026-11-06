# Native Ollama API (`/api/...`)

Base URL: cloud `https://ollama.com/api` (Bearer key required) · local `http://localhost:11434/api` (no auth).

All inference endpoints stream NDJSON (`application/x-ndjson`) by default; `"stream": false` returns one JSON object.

## POST /api/chat

Generate the next chat message.

Request fields:
- `model` (required), `messages` (required)
- `tools` — array of `{ "type": "function", "function": { "name", "description", "parameters" (JSON Schema) } }`
- `format` — `"json"` or a JSON Schema object. **Structured outputs are NOT supported on cloud.**
- `think` — `true`/`false`/`null`, or a named level string (e.g. `"low"`). `null` = model default.
- `options` — `{ seed, temperature, top_k, top_p, min_p, stop, num_ctx, num_predict }`
- `stream` (default `true`), `keep_alive` (`"5m"`, `0` = unload), `logprobs`, `top_logprobs`

Message: `{ "role": "system|user|assistant|tool", "content": "...", "images": ["<base64>"], "tool_calls": [...] }`.
`tool_calls` items: `{ "function": { "name", "arguments": {object} } }`.

Non-streaming response:
```json
{
  "model": "gemma4",
  "created_at": "2025-10-17T23:14:07.414671Z",
  "message": { "role": "assistant", "content": "Hello!", "thinking": "...", "tool_calls": [...] },
  "done": true,
  "done_reason": "stop",
  "total_duration": 174560334,
  "prompt_eval_count": 11,
  "eval_count": 18
}
```
`message.thinking` — separate reasoning-trace field (thinking-capable models). Timing is in nanoseconds.

Example — thinking:
```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" -H "Content-Type: application/json" \
  -d '{ "model": "gpt-oss", "messages": [{"role":"user","content":"How many r in strawberry?"}],
        "think": true, "stream": false }'
```

Example — tool calling:
```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" -H "Content-Type: application/json" \
  -d '{ "model": "qwen3:4b", "stream": false,
        "messages": [{"role":"user","content":"What is the weather in Paris?"}],
        "tools": [{"type":"function","function":{"name":"get_current_weather",
          "description":"Get current weather",
          "parameters":{"type":"object","properties":{"location":{"type":"string"}},"required":["location"]}}}] }'
```

## POST /api/generate

Single-prompt completion. Request: `model` (required), `prompt` (required), plus the same
`think`/`options`/`format`/`stream`/`images` fields. Response: `response` (text) and `thinking`
(reasoning) instead of `message`.

## POST /api/embed

```bash
curl https://ollama.com/api/embed \
  -H "Authorization: Bearer $OLLAMA_API_KEY" -H "Content-Type: application/json" \
  -d '{ "model": "embeddinggemma", "input": ["Why is the sky blue?", "Why is the grass green?"],
        "dimensions": 128 }'
```
Response: `{ "model", "embeddings": [[...]], "total_duration", "prompt_eval_count" }`.
`truncate` (default `true`): `false` errors instead of truncating over-context input. Note: the legacy
singular endpoint is `/api/embeddings` — prefer `/api/embed`.

## Streaming shape (NDJSON)

Per line, one object; final line has `done: true` + `done_reason`:
```
{"model":"gemma4","created_at":"...","response":"That","done":false}
{"model":"gemma4","created_at":"...","response":"!","done":true,"done_reason":"stop"}
```
Chat streams carry partial `message.content` / `message.thinking` / `message.tool_calls`. Thinking
tokens stream before answer tokens — detect the first `thinking` chunk, then switch to `content`.
A mid-stream error is a line `{"error": "..."}` — the HTTP status no longer reflects it.

## Model info / management

- `GET /api/tags` — list models (the source of truth for cloud model names). Cloud: use names as-is
  (`gemma4:31b`); local server signed in: use `name:cloud`.
- `POST /api/show` — `{"model": "..."}` → model capabilities, incl. thinking metadata:
  ```json
  { "thinking": { "values": ["low", "medium", "high"], "default": "medium" } }
  ```
  `values: [false]` = thinking unsupported. Unsupported `think` strings fall back to the default.
- Also: `/api/ps`, `/api/pull`, `/api/push`, `/api/create`, `/api/delete`, `/api/copy` —
  mostly local-model management; not needed for cloud inference.
