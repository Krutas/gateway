# Phase 4 Authentication & Rate Limiting Design

## Goals

1. Prevent unauthorized use of the gateway.
2. Allow per-agent access control.
3. Limit abuse via rate limiting.

## Authentication Modes

### Option A: Global API Key

Single bearer token for the entire gateway. Simplest, suitable for single-user local use.

### Option B: Per-Agent API Keys

Each agent has its own token. The harness uses its own token, and other callers cannot use another agent's endpoints. Recommended for shared servers.

### Recommended: Option B

## Request Authentication

```yaml
agents:
  codex:
    api_key: ${CODEX_GATEWAY_KEY}
```

Valid request:
```bash
curl -H "Authorization: Bearer ${CODEX_GATEWAY_KEY}" \
  http://localhost:8000/agents/codex/v1/chat/completions
```

## Middleware

Add `gateway.server.auth`:

```python
async def agent_api_key_dependency(request: Request, api_key: str = Header(...)):
    agent_id = request.path_params.get("agent_id")
    config = load_config()
    agent = get_agent(config, agent_id)
    expected = agent.env.get("GATEWAY_API_KEY") or config.global_api_key
    if not expected or api_key != expected:
        raise HTTPException(status_code=401, detail="Invalid API key")
```

## Rate Limiting

Use a token-bucket per `agent_id`:

```python
from slowapi import Limiter
limiter = Limiter(key_func=lambda req: req.path_params.get("agent_id", "global"))
```

Or implement a simple in-memory bucket:

```python
class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        self.capacity = capacity
        self.tokens = capacity
        self.rate = rate
        self.last = time.monotonic()

    def allow(self) -> bool:
        now = time.monotonic()
        self.tokens = min(self.capacity, self.tokens + self.rate * (now - self.last))
        self.last = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

Config:

```yaml
gateway:
  rate_limit:
    requests_per_minute: 60
    burst: 10
```

## Out of Scope for MVP

Authentication and rate limiting are Phase 4. The MVP gateway runs unauthenticated on localhost.
