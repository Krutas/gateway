# Agent Harness Guide

## Built-in Harnesses

| Harness | Executable | Endpoint Env Var | Notes |
|---------|------------|------------------|-------|
| codex | `codex` | `OPENAI_BASE_URL` | Points Codex at the per-agent OpenAI shim |
| generic | configurable | `GATEWAY_BASE_URL`, `AGENT_ID` | Any shell command |

## Adding a New Harness

1. Create `gateway/harnesses/{name}.py`.
2. Subclass `BaseHarness`.
3. Implement `build_command`, `build_env`, and optional `check_install`.
4. Register in `gateway/harnesses/registry.py`.

Example template:

```python
import shutil
from gateway.harnesses.base import BaseHarness

class MyHarness(BaseHarness):
    name = "myharness"

    def build_command(self, agent_id, agent, gateway_base_url):
        return ["my-agent", "--serve"]

    def build_env(self, agent_id, agent, gateway_base_url):
        return {
            "MY_AGENT_BASE_URL": f"{gateway_base_url}/agents/{agent_id}/v1",
            "MY_AGENT_MODEL": agent.model,
        }

    def check_install(self):
        if not shutil.which("my-agent"):
            return "Install my-agent: pip install my-agent"
        return None
```

## Target Harnesses (Phase 5+)

1. **Claude Code** — set `ANTHROPIC_BASE_URL` and `ANTHROPIC_API_KEY`.
2. **OpenCode** — configure its OpenAI-compatible endpoint.
3. **Aider** — pass `--openai-api-base` or `--anthropic-api-base`.
4. **Continue.dev** — local config file pointing at gateway.
5. **Custom MCP agents** — generic harness with `COMMAND` and env.

## Environment Variables Available to Agents

| Variable | Source | Description |
|----------|--------|-------------|
| `GATEWAY_BASE_URL` | Generic harness | Root URL of the gateway |
| `AGENT_ID` | Generic harness | Agent identifier |
| `OPENAI_BASE_URL` | Codex harness | Per-agent OpenAI endpoint |
| `OPENAI_API_KEY` | Config env | Token for OpenAI shim |
| `ANTHROPIC_BASE_URL` | User env | Per-agent Anthropic endpoint |
| `ANTHROPIC_API_KEY` | Config env | Token for Anthropic shim |
