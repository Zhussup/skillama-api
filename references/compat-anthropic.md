# Anthropic-compatible API (`/v1/messages`)

Base URL: cloud `https://ollama.com` (SDK appends `/v1/messages`; direct path `https://ollama.com/v1/messages`)
· local `http://localhost:11434/v1/messages`.

## Auth — the big gotcha

- **Cloud requires `Authorization: Bearer $OLLAMA_API_KEY` — `x-api-key` alone is NOT accepted**
  on the cloud endpoint.
- Anthropic SDKs: pass the key as `auth_token` (Bearer), not `api_key` (which sends `x-api-key`).
- Local: keys unvalidated; for SDK tooling `ANTHROPIC_AUTH_TOKEN=ollama`, `ANTHROPIC_BASE_URL=http://localhost:11434`.

Hosted endpoint rejects `anthropic-version` values older than `2023-06-01`.

## SDK examples (cloud)

```python
import os, anthropic
client = anthropic.Anthropic(base_url="https://ollama.com",
                             auth_token=os.environ["OLLAMA_API_KEY"])
msg = client.messages.create(
    model="gemma4:31b", max_tokens=1024,
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
)
print(msg.content[0].text)
```

```javascript
import Anthropic from "@anthropic-ai/sdk";
const anthropic = new Anthropic({ baseURL: "https://ollama.com",
                                  authToken: process.env.OLLAMA_API_KEY });
const msg = await anthropic.messages.create({
  model: "gemma4:31b", max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
});
```

## curl

```bash
curl https://ollama.com/v1/messages \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{ "model": "gemma4:31b", "max_tokens": 1024,
        "messages": [{ "role": "user", "content": "Say hello in one sentence." }] }'
```

## Feature support

**Supported:** messages, streaming, system prompts (string or array), multi-turn, vision
(base64 images only — no URL images), tools/function calling (`name`, `description`,
`input_schema`), tool results, extended thinking, `max_tokens`, `temperature`, `top_p`,
`top_k`, `stop_sequences`, `thinking`, `output_config.effort` (model-defined names).

**NOT supported:** `tool_choice`, `metadata`, `POST /v1/messages/count_tokens`, prompt caching
(`cache_control`), Batches API, citations, PDF documents, server-sent error events during
streaming (errors come back as HTTP status codes). `budget_tokens` is accepted but not enforced.

**Response:** `id`, `type`, `role`, `model`, `content` (text / tool_use / thinking blocks),
`stop_reason` (`end_turn`, `max_tokens`, `tool_use`), `usage` (`input_tokens`, `output_tokens` —
approximations using the underlying model's tokenizer).

Tool-use responses are `tool_use` blocks with `name` and `input`.

## Streaming events (SSE)

`message_start`, `content_block_start`, `content_block_delta` (types `text_delta`,
`input_json_delta`, `thinking_delta`), `content_block_stop`, `message_delta`, `message_stop`,
`ping`, `error`. Use `client.messages.stream(...)`, iterating `content_block_delta`.

## Naming tricks

- For tools/clients expecting default Anthropic names (local): `ollama cp qwen3-coder claude-3-5-sonnet`.
- Cloud models used directly take names from `https://ollama.com/api/tags`.
