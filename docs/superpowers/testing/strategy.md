# Testing Strategy

## Philosophy

Every public function has a unit test. Every adapter is tested against mocked HTTP responses. Every major subsystem has at least one integration test. No external LLM provider is required to run the test suite.

## Test Layout

```
tests/
├── conftest.py                 # Shared fixtures and sys.path setup
├── test_config.py              # Config loading and validation
├── test_ollama.py              # Ollama adapter unit tests
├── test_lmstudio.py            # LM Studio adapter unit tests
├── test_vllm.py                # vLLM adapter unit tests
├── test_openai_compatible.py   # Generic OpenAI adapter unit tests
├── test_anthropic.py           # Anthropic adapter unit tests
├── test_harnesses.py           # Harness command/env generation
├── test_process_manager.py     # Subprocess lifecycle
├── test_server.py              # FastAPI routes
└── test_integration.py         # Happy path end-to-end
```

## Unit Testing Providers

Mock the `httpx.AsyncClient` methods (`get`, `post`, `stream`). Assert:
1. Correct URL and payload are built.
2. Correct auth header is set.
3. Streaming produces expected chunks.
4. Health returns `True/False` based on mocked status.

## Testing Subprocesses

Use short-lived shell commands (`sleep 1`, `echo done`) and verify:
1. Process starts and has a PID.
2. Log file is created.
3. Stop terminates cleanly.
4. Restart stops then starts a new process.

## Integration Test

1. Write a temporary `agents.yaml`.
2. Launch a dummy generic agent.
3. Start `TestClient(create_app(...))`.
4. Call a custom static endpoint.
5. Stop the agent and assert status.

## CI Test Command

```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
pytest -q
```

## Coverage Targets (Phase 2)

- `gateway/config.py` → 100%
- `gateway/providers/*` → >90%
- `gateway/harnesses/*` → >80%
- `gateway/process/manager.py` → >70%
- `gateway/server/*` → >70%
