# OpenAI-compatible API (`/v1/...`)

Base URL: cloud `https://ollama.com/v1` (real API key required) · local `http://localhost:11434/v1`
(key required but ignored). Works with the official OpenAI SDKs (`pip install openai`,
`npm install openai`) without any Ollama install.

```python
import os
from openai import OpenAI
client = OpenAI(base_url="https://ollama.com/v1", api_key=os.environ["OLLAMA_API_KEY"])
resp = client.chat.completions.create(
    model="gemma4:31b",           # names from https://ollama.com/api/tags
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
    stream=False,
)
print(resp.choices[0].message.content)
```

## Endpoint matrix

| Endpoint | Supported | Not supported |
|---|---|---|
| `POST /v1/chat/completions` | streaming, JSON mode, seed, vision, tools, reasoning | logprobs, `tool_choice`, `logit_bias`, `user`, `n`, image URLs (base64 only) |
| `POST /v1/completions` | streaming, JSON mode, `suffix` | `best_of`, `echo`, `logit_bias`, `user`, `n`; `prompt` is string-only |
| `GET /v1/models`, `/v1/models/{model}` | listing (`created`=last modified, `owned_by`=owner or `"library"`) | — |
| `POST /v1/embeddings` | `input` string/array of strings, `encoding_format`, `dimensions` | token-array inputs, `user` |
| `POST /v1/responses` (v0.13.3+) | streaming, tools, reasoning summaries | **stateless**: no `previous_response_id`, `conversation`, `truncation`; cloud lacks built-in web search and custom tool-call replay. Ollama extension field `think` (bool/model string/null) |
| `POST /v1/audio/speech` | `response_format` `wav`(default)/`pcm` only; output 16-bit mono 24 kHz | `instructions`, other formats (→ 400); `speed` must be `1`; `input` ≤ 4096 chars |
| `POST /v1/audio/transcriptions` | `file` (WAV ≤384 kHz, ≤512 MB, ≤2 h), `response_format` json/text/verbose_json, `temperature`, `stream` | `srt`, `vtt`, `timestamp_granularities` |
| `POST /v1/audio/translations` | plus Ollama extensions `to_language` (default `en`), `language` | — |

## chat/completions supported request fields

`model`, `messages` (text, base64 image, or content-part arrays:
`{"type":"text",...}` + `{"type":"image_url","image_url":"data:image/png;base64,..."}`),
`frequency_penalty`, `presence_penalty`, `response_format`, `seed`, `stop`, `stream`,
`stream_options.include_usage`, `temperature`, `top_p`, `max_tokens`, `tools`,
`reasoning_effort` / `reasoning.effort`.

### Reasoning effort mapping

Values resolve via model metadata (check with `POST /api/show`). Legacy aliases:
`"minimal"` → `"low"`; `"xhigh"` / `"ultra"` → `"max"`. GPT-OSS special case: `"minimal"` uses
`"low"`, `"xhigh"/"ultra"` use `"high"`, `"none"` disables thinking. Unsupported names fall back
to model defaults.

### Structured outputs

Works via `response_format` on local; **cloud does not support structured outputs** — use JSON
mode/prompting and validate afterwards.

## Vision (chat)

```json
{ "model": "qwen3-vl:8b", "max_tokens": 300,
  "messages": [{ "role": "user", "content": [
    { "type": "text", "text": "What's in this image?" },
    { "type": "image_url", "image_url": { "url": "data:image/png;base64,<...>" } }
  ]}] }
```

## Tips / gotchas

- Cloud model IDs come from `https://ollama.com/api/tags`. No pull step. Via a signed-in local
  server use `name:cloud` suffix.
- Context size is not settable through the OpenAI API (no `num_ctx`) — for local use a Modelfile
  with `PARAMETER num_ctx <n>` then `ollama create`; cloud models already run at full context.
- For clients requesting a hardcoded default model name (e.g. `gpt-3.5-turbo`), local trick:
  `ollama cp llama3.2 gpt-3.5-turbo`.
