# Account endpoints, web tools, errors & rate limits (Ollama Cloud)

Auth for all of these: `Authorization: Bearer $OLLAMA_API_KEY` (cloud only, requires account at ollama.com).

## GET https://ollama.com/api/balance — credits

```bash
curl https://ollama.com/api/balance -H "Authorization: Bearer $OLLAMA_API_KEY"
```

```json
{
  "included": {
    "balance_usd": 72.5,
    "allowance_usd": 100,
    "period": { "from": "2026-09-15T09:30:00Z", "until": "2026-10-15T09:30:00Z" }
  },
  "purchased": { "balance_usd": 25 }
}
```

- `included.balance_usd` — remaining plan credits; `allowance_usd` — full-period allowance; `period`
  follows the plan's monthly reset (from inclusive, until exclusive, UTC).
- `purchased.balance_usd` — remaining unexpired purchased credits.
- Legacy plans instead return `included.session` / `included.weekly` with `remaining_percent`
  (0–100) and `resets_at`.
- Rate limit: **10 requests/min per user** — shared across API keys and devices; poll once per minute.
- Teams: all members share the same included and purchased balances.

## GET https://ollama.com/api/usage — usage stats

```bash
curl 'https://ollama.com/api/usage?range=24h&group_by=model' \
  -H "Authorization: Bearer $OLLAMA_API_KEY"
```

- `range`: `24h` (hourly buckets, 25) | `7d` (default, daily, 8) | `30d` (daily, 31); UTC;
  last bucket `partial: true`. Empty buckets included as zeroes.
- `scope`: `self` (default) | `team` (team admins only). `group_by=model` adds `groups`
  (top 10 by `usage_usd`, rest in `remainder`).
- Metrics: `request_count`, `usage_usd`, `input_tokens` (includes cached),
  `cached_input_tokens` (already inside input_tokens), `output_tokens`.
- Rate limit: 10 req/min. New usage may take time to appear.
- Response: `{ "range", "scope", "granularity", "from", "until", "totals", "buckets": [...] }`.

## POST https://ollama.com/api/web_search

```bash
curl https://ollama.com/api/web_search \
  --header "Authorization: Bearer $OLLAMA_API_KEY" -H "Content-Type: application/json" \
  -d '{ "query": "what is ollama?", "max_results": 5 }'
```

- `max_results`: default 5, **max 10** → `{ "results": [{ "title", "url", "content" }] }`.
- Requires a free account + API key (works independently of inference).
- SDK: Python `ollama.web_search(...)`, JS `client.webSearch(...)`.

## POST https://ollama.com/api/web_fetch

```bash
curl https://ollama.com/api/web_fetch \
  --header "Authorization: Bearer $OLLAMA_API_KEY" -H "Content-Type: application/json" \
  -d '{ "url": "https://ollama.com/" }'
```

→ `{ "title", "content", "links": [...] }`. SDK: `ollama.web_fetch(url)` / `client.webFetch(url)`.

### Search agent notes

- Give the model web_search/web_fetch as regular tools; execute tool calls and append results
  as `tool` role messages in the chat loop.
- Search results can be thousands of tokens — use ≥ ~32k context for local models
  (cloud models already run at full context length).

## Errors

Format: `{ "error": "Error message describing what went wrong" }` (application/json).

| Status | Meaning |
|---|---|
| 400 | Bad request — missing params, invalid JSON, invalid query params |
| 401 | Invalid credentials (bad/missing API key) |
| 403 | Account suspended, or team scope without team-admin access |
| 404 | Model doesn't exist |
| 429 | Rate limit exceeded — includes `Retry-After` (integer seconds) |
| 500 | Internal server error |
| 502 | Bad gateway — cloud model cannot be reached |
| 503 | Reporting temporarily unavailable (usage/balance endpoints) |

Mid-stream errors: NDJSON line `{"error": "..."}`; the HTTP status is NOT changed because the
response already started — clients must watch for error lines inside a 200 stream.

## Rate limits

- Documented limits: balance & usage = 10 req/min/user (shared across keys/devices), web search
  max_results ≤ 10. Other per-plan rate limits and quotas are not published — see
  https://ollama.com/pricing and usage dashboards at https://ollama.com/settings/usage.
- On 429, honor `Retry-After`.
