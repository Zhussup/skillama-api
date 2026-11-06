---
name: skillama-api
description: Reference for the Ollama Cloud API (ollama.com direct inference, OpenAI- and Anthropic-compatible endpoints, auth, account endpoints, capabilities and gotchas). Use when writing or debugging code that calls Ollama — especially Ollama Cloud — because most models' training data predates Ollama Cloud.
---

# Ollama Cloud API

Most LLMs were trained when Ollama was local-only. Ollama now runs **cloud models** served from
`https://ollama.com` — no local install needed. This skill is the corrected reference.

## Core facts (get these right)

**Three base URLs, one API key:**

| API | Direct cloud (no Ollama install) | Local server |
|---|---|---|
| Ollama native | `https://ollama.com/api` | `http://localhost:11434/api` |
| OpenAI-compatible | `https://ollama.com/v1` | `http://localhost:11434/v1` |
| Anthropic-compatible | `https://ollama.com` (SDK appends `/v1/messages`) | `http://localhost:11434` |

- API key: create at `https://ollama.com/settings/keys`, header `Authorization: Bearer $OLLAMA_API_KEY`.
  Keys don't expire. Keep out of browser code and git.
- **Cloud direct calls always require the key.** Local calls don't (`ollama signin` proxies to cloud instead).
- Anthropic-compatible `/v1/messages`: `x-api-key` alone is **NOT accepted** on cloud — Bearer required.
- Model names differ by access path: direct API → `gemma4:31b` (from `/api/tags`);
  CLI/app via a signed-in local server → `gemma4:cloud`. No pull needed for cloud models.
- Native endpoints stream by default (NDJSON, `application/x-ndjson`). Disable with `"stream": false`.
- **Cloud does NOT support structured outputs** (the `format` / JSON-schema feature of local Ollama).
  For cloud, prompt for JSON or validate after the fact.
- Errors: JSON `{"error": "..."}`. Statuses: 400 bad request, 404 unknown model, 429 rate limit
  (has `Retry-After` header), 502 cloud model unreachable. Mid-stream errors arrive as an NDJSON
  line `{"error": ...}` without changing the status code.

## Quickstart

```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "model": "gemma4:31b",
        "messages": [{"role": "user", "content": "Say hello in one sentence."}],
        "stream": false }'
# answer: resp["message"]["content"]
```

```python
# OpenAI SDK against cloud
from openai import OpenAI
client = OpenAI(base_url="https://ollama.com/v1", api_key=os.environ["OLLAMA_API_KEY"])
resp = client.chat.completions.create(model="gemma4:31b",
                                      messages=[{"role": "user", "content": "Hi"}])
```

```python
# Anthropic SDK against cloud
import anthropic, os
client = anthropic.Anthropic(base_url="https://ollama.com",
                             auth_token=os.environ["OLLAMA_API_KEY"])  # not api_key
msg = client.messages.create(model="gemma4:31b", max_tokens=1024,
                             messages=[{"role": "user", "content": "Hi"}])
```

## Cloud-only account endpoints

- `GET /api/balance` — remaining credits (`included` plan allowance + `purchased`); 10 req/min, `Retry-After` on 429.
- `GET /api/usage?range=24h|7d|30d&scope=self|team&group_by=model` — usage stats; 10 req/min.
- `POST /api/web_search` — `{ "query": ..., "max_results": 1-10 (default 5) }`; works with any tool-calling model.
- `POST /api/web_fetch` — `{ "url": ... }` → `{ title, content, links }`.
- Cloud models run at full context length; local models need `num_ctx` raised for web search results.

## Detailed references

Read the matching file for exact fields, schemas and examples:

- [references/native-api.md](references/native-api.md) — `/api/chat`, `/api/generate`, `/api/embed`,
  streaming shape, thinking (`think` param, `message.thinking`), tool calling, model options.
- [references/compat-openai.md](references/compat-openai.md) — OpenAI-compatible endpoints,
  supported/unsupported fields, reasoning effort mapping, `/v1/responses` limits.
- [references/compat-anthropic.md](references/compat-anthropic.md) — `/v1/messages`, Bearer auth,
  supported/unsupported features, streaming events, thinking.
- [references/account-and-tools.md](references/account-and-tools.md) — balance, usage, web search/fetch,
  error codes, rate limits.
- [references/models.md](references/models.md) — cloud model catalog: names, capabilities,
  and what each family is typically used for (coding, cheap/fast, long-context, vision, audio,
  decision). Model list drifts — verify via `GET https://ollama.com/api/tags`.

Docs change: verify anything uncertain against https://docs.ollama.com (index: https://docs.ollama.com/llms.txt).
The live cloud model list is `GET https://ollama.com/api/tags`.
