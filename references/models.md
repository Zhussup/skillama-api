# Cloud models on Ollama — what to use for what

**Live list is the source of truth** (names/dates drift): `GET https://ollama.com/api/tags`
or browse https://ollama.com/search?c=cloud. Use names exactly as returned there
(`gemma4:31b`, `deepseek-v4.1-flash:1017`). Dated tags (e.g. `:1017`, `:0813`) pin a version;
the bare name (e.g. `deepseek-v4.1-flash`) tracks the current one.

Captured 2026-11-07. Every model below has `thinking` + `tools` unless noted; "vision" = multimodal input.

## Typical use by task

| Task | Good first pick | Alternatives |
|---|---|---|
| Coding agents / software engineering | `glm-5.3` | `kimi-k3.1-pro`, `kimi-k2.7-code`, `minimax-m3`, `muse-spark-open` |
| Cheap / fast / high-volume | `glm-5.3-flash` | `minimax-m3.1-flash`, `deepseek-v4.1-flash`, `mimo-v2.6-flash`, `gpt-oss:20b` |
| General chat / assistant | `gemma4:31b` | `mistral-large-4`, `glm-5.3` |
| Long-context / long-horizon agents (1M ctx) | `minimax-m3` | `glm-5.2`, `muse-spark-open` |
| Vision (image input) | `glm-5.3-flash` | any model with `vision`: `kimi-k3*`, `mimo-*`, `minimax-m3*`, `gemma4:31b`, `mistral-large-4`, `deepseek-v4.1-flash` |
| Audio input | `gemma4` (only cloud model tagged audio) | — |
| Yes/no, choice, scoring decisions | `clef:27b` (via `/v1/systemone`) | — |

## Model details

| Model | Origin | Capabilities | For what |
|---|---|---|---|
| `glm-5.3` | Z.ai | thinking, tools | Flagship; strongest open-weights for coding and agentic tasks |
| `glm-5.3-flash` | Z.ai | + vision | Fast/cheap multimodal tier, near-flagship coding quality — good default for high-volume apps |
| `glm-5.2` | Z.ai | thinking, tools | Previous flagship, long-horizon tasks |
| `kimi-k3`, `kimi-k3.1-pro` | Moonshot AI | + vision | Most capable multimodal agentic models; coding, research, long documents |
| `kimi-k2.7-code` | Moonshot AI | + vision | Coding-agent specialist, reduced thinking-token use |
| `kimi-k2.6` | Moonshot AI | + vision | Predecessor of k2.7 |
| `deepseek-v4.1-flash` (+ `:1017`) | DeepSeek | + vision | Fast, high-throughput general reasoning with vision |
| `deepseek-v4-pro:0813` | DeepSeek | — | Pro (heavier) tier of the DeepSeek v4 line |
| `minimax-m3` | MiniMax | + vision | Coding/agentic frontier model, 1M context, native multimodality |
| `minimax-m3.1-flash` | MiniMax | + vision | Fast/cheap tier; strong on agentic tasks, Computer Use, cybersecurity |
| `minimax-m2.7` | MiniMax | — | Coding, agentic workflows, professional productivity |
| `muse-spark-open` | — | + vision | Large (816B/42B active) multimodal model for long-horizon software engineering |
| `beam` | Reflection AI | — | Agentic tool use, coding, reasoning; token-efficient |
| `mimo-v2.6-pro` | Xiaomi | + vision | Flagship, omnimodal input |
| `mimo-v2.6-flash` | Xiaomi | + vision | Efficiency-focused tier, omnimodal input |
| `nemotron-3-ultra` | NVIDIA | — | High-throughput reasoning, long-running agent workflows |
| `nemotron-3-super` | NVIDIA | — | Efficient complex multi-agent applications |
| `nemotron-3-nano:30b` | NVIDIA | — | Small/cheap efficient agentic model |
| `mistral-large-4` | Mistral | + vision | General-purpose multimodal MoE |
| `mistral-large-3:675b` | Mistral | + vision, no thinking | Previous large tier |
| `gpt-oss:120b` / `gpt-oss:20b` | OpenAI | — | Open-weight reasoning models; 20b = small/cheap, 120b = stronger |
| `gemma4:31b` | Google | + vision, audio | General purpose: reasoning, agents, coding, multimodal; only audio-capable cloud model |
| `clef:27b` | Cloudflare | decision, vision, no tools | Decision model — used via `POST /v1/systemone` (yes/no, choice, scoring questions), not chat |

## Notes

- `clef:27b` is the only `decision`-capability model — for the System One endpoint, not `/api/chat`.
- No cloud embedding models were listed at capture time — for embeddings prefer local
  (`embeddinggemma`, `nomic-embed-text` via `POST /api/embed` on `localhost:11434`) or the
  OpenAI-compatible `/v1/embeddings` against a local server; check `/api/tags` for changes.
- Model metadata (context, thinking levels, pricing) per model: `POST https://ollama.com/api/show`
  and the model's page at https://ollama.com/<name>; pricing/tiers at https://ollama.com/pricing.
