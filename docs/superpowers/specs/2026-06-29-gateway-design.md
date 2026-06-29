# Gateway AI Agent Launcher — Design Spec

**Date:** 2026-06-29  
**Project root:** `/root/gateway`  
**Goal:** A single CLI/HTTP gateway that launches open-source AI agent harnesses (Codex, Claude Code, OpenCode, etc.) against local LLM providers (Ollama, LM Studio, vLLM, OpenAI-compatible, Anthropic-compatible) and exposes per-agent custom HTTP/WebSocket endpoints.

## User-Approved Decisions

1. **Architecture:** Python CLI + FastAPI gateway + subprocess agent harnesses.
2. **CLI style:** Provider-prefixed commands, e.g. `gateway ollama launch codex`.
3. **Project location:** All files live under `/root/gateway`.
4. **Chat log:** Plain-text chat output saved to `/root/gateway/chat.log`.

## Components

| Component | Responsibility |
|-----------|----------------|
| `gateway.cli` | Parses terminal commands, validates, dispatches to process manager. |
| `gateway.config` | Loads `~/.gateway/agents.yaml`, validates with Pydantic. |
| `gateway.providers.*` | Adapts each LLM backend to a common chat/models/health interface. |
| `gateway.harnesses.*` | Knows how to spawn/configure each agent harness executable. |
| `gateway.process` | Spawns, monitors, restarts, stops agents; manages ports and logs. |
| `gateway.server` | FastAPI app that mounts OpenAI/Anthropic shims and custom routes per agent. |

## Data Flow

1. User runs `gateway ollama launch codex`.
2. CLI looks up the `codex` agent profile in config, confirms its provider is `ollama`.
3. Process manager allocates a port, builds the harness command, and spawns it with environment variables pointing at the per-agent endpoint.
4. FastAPI server (already running or started) registers `/agents/codex/v1/chat/completions` and any custom routes.
5. The harness calls the endpoint instead of the raw LLM; the provider adapter forwards to Ollama.
6. Logs stream to `~/.gateway/logs/{agent_id}.log`.

## Error Handling

- Config validation errors print the offending field and exit non-zero.
- Provider health failures surface as 503 with JSON detail.
- Agent crashes trigger configurable restart (default: 3 attempts, exponential backoff).
- Missing harness executable prints install instructions from the harness adapter.

## Testing Strategy

- Unit tests for every provider adapter using mocked HTTP responses.
- Process manager tested with a no-op shell harness.
- End-to-end test launches a dummy harness and verifies its endpoint responds.

## Scope for This Plan

This spec/plan covers the full MVP: CLI, config, all providers, harness abstraction, process manager, HTTP gateway, and one happy-path integration test. Web UI, authentication, and rate-limiting are noted as future phases but excluded from the initial implementation.
