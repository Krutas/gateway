# Configuration Reference

The gateway reads `~/.gateway/agents.yaml` unless overridden by `--config` or `GATEWAY_CONFIG`.

## Top-Level Keys

```yaml
agents:
  {agent_id}: AgentProfile
providers:
  {provider_id}: ProviderProfile
```

## AgentProfile

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | yes | — | Provider id matching a key under `providers:` |
| `model` | string | yes | — | Model identifier to send to the provider |
| `harness` | string | yes | — | Harness id (`codex`, `generic`, etc.) |
| `env` | map[str,str] | no | `{}` | Extra environment variables for the agent subprocess |
| `endpoints` | list[CustomEndpoint] | no | `[]` | Additional per-agent HTTP routes |
| `port` | int | no | auto | Port allocated to the agent harness if needed |
| `restart_policy` | `never`, `always`, `on_failure` | no | `on_failure` | When to restart the agent |
| `max_restarts` | int | no | `3` | Max restart attempts before giving up |

## CustomEndpoint

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | yes | — | Route under `/agents/{agent_id}` |
| `type` | `openai`, `anthropic`, `static`, `proxy` | yes | — | Handler type |
| `response` | map | no | — | For `static`: JSON body to return |
| `target_url` | string | no | — | For `proxy`: upstream URL |
| `method` | `GET`, `POST` | no | `GET` | Allowed HTTP method |

## ProviderProfile

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `base_url` | string | yes | — | Provider base URL (trailing slash stripped) |
| `kind` | `ollama`, `openai`, `anthropic` | no | `openai` | Adapter kind to use |
| `api_key` | string | no | — | API key or token. Prefer env var interpolation. |
| `timeout` | float | no | `60` | Request timeout in seconds |

## Example

```yaml
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    harness: codex
    endpoints:
      - path: /health
        type: static
        response: {"status": "ok"}
  claude-code-local:
    provider: lmstudio
    model: claude-3-5-sonnet
    harness: generic
    env:
      COMMAND: "claude"
      ANTHROPIC_BASE_URL: http://localhost:8000/agents/claude-code-local/v1
      ANTHROPIC_API_KEY: gateway-local

providers:
  ollama:
    base_url: http://localhost:11434
  lmstudio:
    base_url: http://localhost:1234/v1
    kind: openai
  vllm:
    base_url: http://localhost:8000/v1
    kind: openai
  anthropic:
    base_url: https://api.anthropic.com
    kind: anthropic
    api_key: ${ANTHROPIC_API_KEY}
```

## Env Var Interpolation

After MVP, the config loader will support `${VAR}` substitution for `api_key` and `env` values. Do not commit literal secrets.
