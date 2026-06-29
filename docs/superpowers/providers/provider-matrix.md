# Provider Quirks Matrix

| Provider | Chat API | Streaming | Models API | Auth | Default Port | Notes |
|----------|----------|-----------|------------|------|--------------|-------|
| Ollama | `/api/chat` | ✅ SSE lines | `/api/tags` | None | 11434 | `stream` is a JSON field; responses are NDJSON |
| LM Studio | `/v1/chat/completions` | ✅ SSE `data:` | `/v1/models` | Optional Bearer | 1234/v1 | OpenAI-compatible; set `base_url` to `/v1` |
| vLLM | `/v1/chat/completions` | ✅ SSE `data:` | `/v1/models` | Optional Bearer | 8000/v1 | OpenAI-compatible |
| Generic OpenAI | `/v1/chat/completions` | ✅ SSE `data:` | `/v1/models` | Bearer | varies | Use for any `/v1` shim |
| Anthropic | `/v1/messages` | ✅ SSE `data:` | `/v1/models` | `x-api-key` | 443 | Messages API, not chat.completions |

## Streaming Differences

- **Ollama:** `POST /api/chat {stream: true}` returns newline-delimited JSON.
- **OpenAI-compatible:** Server-Sent Events with `data: {...}` lines.
- **Anthropic:** Server-Sent Events with `data: {...}` lines and `content_block_delta` events.

## Authentication

- **Bearer:** `Authorization: Bearer <token>` for OpenAI-compatible and LM Studio/vLLM.
- **API key header:** `x-api-key: <token>` + `anthropic-version: 2023-06-01` for Anthropic.
- **None:** Ollama by default.

## Rate Limits

- **Ollama:** local hardware bound, no API rate limit.
- **Cloud providers:** follow upstream limits; the gateway will add token-bucket rate limiting in Phase 4.

## Common Issues

1. LM Studio `base_url` missing `/v1` suffix → 404.
2. Anthropic request using `chat.completions` shape → 400; must use messages.
3. Ollama model not yet pulled → 404 model not found.
4. vLLM `max_tokens` missing in Anthropic mode → 400.
