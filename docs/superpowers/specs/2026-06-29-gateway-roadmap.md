# Gateway AI Agent Launcher — Roadmap & Future Phases

**Repo:** https://github.com/Krutas/gateway  
**Status:** MVP implementation plan complete, ready for execution.

## MVP Scope (Phase 1)

Build the core CLI + gateway in `/root/gateway`:

- Python 3.11 + Pydantic 2 + FastAPI project scaffold
- YAML config engine (`~/.gateway/agents.yaml`)
- Provider adapters: Ollama, LM Studio, vLLM, generic OpenAI-compatible, Anthropic-compatible
- Harness abstraction with Codex and generic executable adapters
- Process manager: spawn, stop, restart, logs, status
- CLI: `gateway ollama launch codex`, `gateway list`, `gateway logs`, `gateway server`
- FastAPI per-agent routes: `/agents/{id}/v1/chat/completions`, `/agents/{id}/v1/messages`, custom endpoints
- pytest test suite with mocked providers and one happy-path integration test

## Phase 2: Stability & DX

1. **Configuration reload** — watch `agents.yaml` and hot-reload without restarting the gateway server.
2. **Restart policies** — implement exponential backoff, max restart counters, and crash notifications.
3. **Log streaming endpoint** — `GET /agents/{id}/logs/stream` over SSE or WebSocket.
4. **CLI autocompletion** — shell completions for bash/zsh.
5. **Better error messages** — detect missing executables, provider health failures, and config typos.

## Phase 3: Multi-Provider Runtime Enhancements

1. **Provider fallback / load balancing** — route one agent across multiple backend URLs with round-robin or health-based failover.
2. **Provider-specific streaming** — full SSE/stream parity for Ollama, OpenAI, and Anthropic adapters.
3. **Embeddings endpoint** — `/v1/embeddings` for providers that support it (LM Studio, vLLM, OpenAI).
4. **Model discovery caching** — cache `models()` results with TTL to avoid hammering backends.

## Phase 4: Security & Multi-Tenancy

1. **API key authentication** — per-agent or global bearer tokens.
2. **Rate limiting** — token bucket per API key and per agent.
3. **Network isolation** — bind gateway to localhost by default, optional TLS/mTLS.
4. **Secret management** — pull API keys from env vars or a keyring, never commit them.
5. **Audit logging** — log all requests/responses metadata (not bodies) for compliance.

## Phase 5: Web Dashboard

1. **Agent status page** — running/stopped/crashed, CPU/memory, last error.
2. **Log viewer** — tail logs in browser.
3. **Launch/stop controls** — buttons per agent with confirmation.
4. **Provider health panel** — green/red per backend.
5. **Custom endpoint tester** — small HTTP client built into the UI.

## Phase 6: Packaging & Deployment

1. **Docker image** — `Dockerfile` with multi-stage build, expose 8000.
2. **PyPI package** — `pip install ai-agent-gateway`.
3. **systemd service** — auto-start gateway on boot.
4. **Homebrew formula** — optional for macOS users.
5. **Release CI** — GitHub Actions run tests, build, and publish on tag.

## Suggested Implementation Order

Execute the MVP plan first. Then tackle Phase 2 before Phase 4 because reliability is required before adding auth/rate limits. Phase 3 can be interleaved with Phase 2. Phase 5 (dashboard) is best after the API surface is stable. Phase 6 comes last.

## Open Questions to Resolve

1. Do you want the gateway to support **remote agents** (SSH/Slurm/Docker) or only local subprocesses?
2. Should the gateway expose a **single shared API key** or **per-agent keys** by default?
3. Do you want a **built-in web dashboard** in the MVP, or keep it CLI-only until Phase 5?
4. Which additional harnesses should be prioritized: Cline, OpenCode, Claude Code, Aider, Continue.dev?
