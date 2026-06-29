# Gateway API Contract

Base URL: `http://127.0.0.1:8000` (default).

## Global Endpoints

### `GET /health`

Returns gateway status.

**Response 200:**
```json
{"status": "ok"}
```

## Per-Agent Endpoints

All routes are mounted under `/agents/{agent_id}`.

### `POST /agents/{agent_id}/v1/chat/completions`

OpenAI-compatible chat completions.

**Request headers:**
- `Content-Type: application/json`
- `Authorization: Bearer {token}` *(Phase 4)*

**Request body:**
```json
{
  "model": "qwen2.5-coder:32b",
  "messages": [
    {"role": "system", "content": "You are a coding assistant."},
    {"role": "user", "content": "Write a Python function."}
  ],
  "stream": false,
  "temperature": 0.7,
  "max_tokens": 1024
}
```

**Response 200 (non-streaming):**
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "qwen2.5-coder:32b",
  "choices": [
    {
      "index": 0,
      "message": {"role": "assistant", "content": "def hello(): print('world')"},
      "finish_reason": "stop"
    }
  ]
}
```

**Response 200 (streaming):**

`text/event-stream` with lines:
```
data: {"choices":[{"delta":{"content":"def"}}]}

data: {"choices":[{"delta":{"content":" hello"}}]}

data: [DONE]

```

### `POST /agents/{agent_id}/v1/messages`

Anthropic-compatible messages endpoint.

**Request body:**
```json
{
  "model": "claude-3-5-sonnet",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "max_tokens": 1024,
  "stream": false
}
```

**Response 200:**
```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "text", "text": "Hi there."}
  ]
}
```

### `GET /agents/{agent_id}/v1/models`

Proxy provider model list. Optional in MVP.

**Response 200:**
```json
{
  "object": "list",
  "data": [
    {"id": "qwen2.5-coder:32b", "object": "model"}
  ]
}
```

### `GET /agents/{agent_id}/logs`

Return recent agent log lines.

**Query params:**
- `tail` (int, default 50)

**Response 200:**
```json
{"agent_id": "codex", "lines": ["..."]}
```

### Custom Endpoints

Defined in `agents.yaml`:

```yaml
agents:
  codex:
    endpoints:
      - path: /health
        type: static
        response: {"status": "ok"}
      - path: /proxy/special
        type: proxy
        target_url: http://localhost:3000/special
```

## Error Responses

All errors return JSON:

```json
{
  "detail": "Agent 'codex' not found in config"
}
```

Common status codes:
- `400` — bad request body
- `404` — agent/provider not found
- `422` — Pydantic validation error
- `500` — unexpected gateway error
- `503` — provider unavailable
