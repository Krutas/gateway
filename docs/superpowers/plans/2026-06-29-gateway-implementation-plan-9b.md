# Gateway AI Agent Launcher — 9B-Friendly Implementation Plan

> **For agentic workers:** This plan is written to be executed by a 9B-class coding model. Each task is tiny, explicit, and self-contained. Follow tasks in order. Do not skip steps.

**Goal:** Build a small CLI + HTTP gateway that launches one agent harness against one LLM provider at a time, starting with Ollama and the `codex` harness.

**Tech Stack:** Python 3.11, FastAPI, Uvicorn, Pydantic 2, Typer, httpx, PyYAML, pytest.

---

## Project Layout (Final)

```
/root/gateway/
├── pyproject.toml
├── README.md
├── gateway/
│   ├── __init__.py
│   ├── cli.py
│   ├── config.py
│   ├── models.py
│   ├── server.py
│   ├── process.py
│   └── ollama_provider.py
└── tests/
    ├── test_config.py
    ├── test_ollama.py
    ├── test_server.py
    └── test_cli.py
```

**Rule:** Build exactly these files. Do not create extra folders until Phase 2.

---

## Task 1: Create Project Files

Create these 4 files with exact content.

### File 1: `pyproject.toml`

```toml
[project]
name = "gateway"
version = "0.1.0"
description = "Launch AI agents against local LLM providers"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "pyyaml>=6.0",
    "fastapi>=0.110",
    "uvicorn[standard]>=0.28",
    "typer>=0.12",
    "httpx>=0.27",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-asyncio>=0.23", "httpx>=0.27"]

[project.scripts]
gateway = "gateway.cli:app"

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["."]
include = ["gateway*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### File 2: `gateway/__init__.py`

```python
__version__ = "0.1.0"
```

### File 3: `README.md`

```markdown
# Gateway

Launch AI agents against local LLM providers.

## Install

```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

## Start server

```bash
gateway server
```

## Launch agent

```bash
gateway launch codex
```

## Config file

`~/.gateway/agents.yaml`:

```yaml
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    command: codex
providers:
  ollama:
    base_url: http://localhost:11434
```
```

### File 4: `.gitignore`

```gitignore
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/
.venv/
.env
*.log
```

### Verify

```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

Expected: installs without error.

### Commit

```bash
cd /root/gateway
git add pyproject.toml README.md .gitignore gateway/__init__.py
git commit -m "chore: scaffold 9b-friendly gateway project"
git push origin main
```

---

## Task 2: Config Engine

### File 1: `gateway/models.py`

```python
from pydantic import BaseModel

class AgentProfile(BaseModel):
    provider: str
    model: str
    command: str
    env: dict[str, str] = {}

class ProviderProfile(BaseModel):
    base_url: str
    api_key: str | None = None

class Config(BaseModel):
    agents: dict[str, AgentProfile]
    providers: dict[str, ProviderProfile]
```

### File 2: `gateway/config.py`

```python
from pathlib import Path
import os
import yaml
from gateway.models import Config, AgentProfile, ProviderProfile

CONFIG_PATH = Path(os.environ.get("GATEWAY_CONFIG", Path.home() / ".gateway" / "agents.yaml"))

def load_config(path: Path | str | None = None) -> Config:
    p = Path(path) if path else CONFIG_PATH
    if not p.exists():
        raise FileNotFoundError(f"Config not found: {p}")
    raw = yaml.safe_load(p.read_text())
    return Config.model_validate(raw)

def get_agent(config: Config, agent_id: str) -> AgentProfile:
    if agent_id not in config.agents:
        raise KeyError(f"Agent '{agent_id}' not found")
    return config.agents[agent_id]

def get_provider(config: Config, provider_id: str) -> ProviderProfile:
    if provider_id not in config.providers:
        raise KeyError(f"Provider '{provider_id}' not found")
    return config.providers[provider_id]
```

### File 3: `tests/test_config.py`

```python
from pathlib import Path
from gateway.config import load_config, get_agent, get_provider

def test_load_config(tmp_path):
    p = tmp_path / "agents.yaml"
    p.write_text("""
agents:
  codex:
    provider: ollama
    model: qwen
    command: codex
providers:
  ollama:
    base_url: http://localhost:11434
""")
    cfg = load_config(p)
    assert cfg.agents["codex"].model == "qwen"
    assert cfg.providers["ollama"].base_url == "http://localhost:11434"

def test_get_agent_and_provider(tmp_path):
    p = tmp_path / "agents.yaml"
    p.write_text("""
agents:
  codex:
    provider: ollama
    model: qwen
    command: codex
providers:
  ollama:
    base_url: http://localhost:11434
""")
    cfg = load_config(p)
    agent = get_agent(cfg, "codex")
    provider = get_provider(cfg, "ollama")
    assert agent.command == "codex"
    assert provider.base_url == "http://localhost:11434"
```

### Verify

```bash
cd /root/gateway
pytest tests/test_config.py -v
```

Expected: 2 tests pass.

### Commit

```bash
cd /root/gateway
git add gateway/models.py gateway/config.py tests/test_config.py
git commit -m "feat: add simple config engine"
git push origin main
```

---

## Task 3: Ollama Provider Adapter

### File 1: `gateway/ollama_provider.py`

```python
from __future__ import annotations
import json
import httpx
from gateway.models import ProviderProfile

class OllamaProvider:
    def __init__(self, profile: ProviderProfile):
        self.profile = profile
        self.client = httpx.AsyncClient(base_url=profile.base_url, timeout=60.0)

    async def chat(self, messages: list[dict], model: str, stream: bool = False):
        payload = {
            "model": model,
            "messages": messages,
            "stream": stream,
        }
        if stream:
            async def _stream():
                async with self.client.stream("POST", "/api/chat", json=payload) as response:
                    response.raise_for_status()
                    async for line in response.aiter_lines():
                        if line:
                            yield json.loads(line)
            return _stream()
        response = await self.client.post("/api/chat", json=payload)
        response.raise_for_status()
        return response.json()

    async def models(self):
        response = await self.client.get("/api/tags")
        response.raise_for_status()
        data = response.json()
        return [m["name"] for m in data.get("models", [])]

    async def health(self):
        try:
            response = await self.client.get("/api/tags")
            return response.status_code == 200
        except Exception:
            return False
```

### File 2: `tests/test_ollama.py`

```python
import pytest
from gateway.models import ProviderProfile
from gateway.ollama_provider import OllamaProvider

@pytest.fixture
def provider():
    return OllamaProvider(ProviderProfile(base_url="http://localhost:11434"))

@pytest.mark.asyncio
async def test_models(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def json(self):
            return {"models": [{"name": "qwen"}]}
        def raise_for_status(self):
            pass
    async def fake_get(url):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.models() == ["qwen"]

@pytest.mark.asyncio
async def test_health_ok(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
    async def fake_get(url):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.health() is True

@pytest.mark.asyncio
async def test_health_bad(provider, monkeypatch):
    async def fake_get(url):
        raise Exception("fail")
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.health() is False
```

### Verify

```bash
cd /root/gateway
pytest tests/test_ollama.py -v
```

Expected: 3 tests pass.

### Commit

```bash
cd /root/gateway
git add gateway/ollama_provider.py tests/test_ollama.py
git commit -m "feat: add Ollama provider adapter"
git push origin main
```

---

## Task 4: Process Manager

### File 1: `gateway/process.py`

```python
from __future__ import annotations
import asyncio
import os
from pathlib import Path
from dataclasses import dataclass
from gateway.config import load_config, get_agent

@dataclass
class AgentState:
    agent_id: str
    pid: int | None = None
    status: str = "stopped"
    log_file: str | None = None

class ProcessManager:
    def __init__(self, config_path: str | None = None, gateway_base_url: str = "http://localhost:8000"):
        self.config_path = config_path
        self.gateway_base_url = gateway_base_url
        self.agents: dict[str, AgentState] = {}
        self.processes: dict[str, asyncio.subprocess.Process] = {}
        self.log_dir = Path.home() / ".gateway" / "logs"
        self.log_dir.mkdir(parents=True, exist_ok=True)

    async def start(self, agent_id: str) -> AgentState:
        if agent_id in self.processes:
            proc = self.processes[agent_id]
            if proc.returncode is None:
                return self.agents[agent_id]
        config = load_config(self.config_path)
        agent = get_agent(config, agent_id)
        log_file = self.log_dir / f"{agent_id}.log"
        env = os.environ.copy()
        env["OPENAI_BASE_URL"] = f"{self.gateway_base_url}/agents/{agent_id}/v1"
        env["OPENAI_API_KEY"] = "gateway-local"
        env.update(agent.env)
        with open(log_file, "a") as lf:
            proc = await asyncio.create_subprocess_exec(
                agent.command,
                env=env,
                stdout=lf,
                stderr=asyncio.subprocess.STDOUT,
            )
        state = AgentState(agent_id=agent_id, pid=proc.pid, status="running", log_file=str(log_file))
        self.agents[agent_id] = state
        self.processes[agent_id] = proc
        return state

    async def stop(self, agent_id: str) -> AgentState:
        if agent_id not in self.processes:
            raise ValueError(f"Agent '{agent_id}' not running")
        proc = self.processes[agent_id]
        proc.terminate()
        try:
            await asyncio.wait_for(proc.wait(), timeout=5.0)
        except asyncio.TimeoutError:
            proc.kill()
            await proc.wait()
        self.agents[agent_id].status = "stopped"
        del self.processes[agent_id]
        return self.agents[agent_id]

    def list(self) -> list[AgentState]:
        for agent_id, proc in list(self.processes.items()):
            if proc.returncode is not None and self.agents[agent_id].status == "running":
                self.agents[agent_id].status = "crashed"
        return list(self.agents.values())

    def logs(self, agent_id: str, tail: int = 50) -> str:
        if agent_id not in self.agents:
            raise ValueError(f"Agent '{agent_id}' not found")
        log_file = Path(self.agents[agent_id].log_file or "")
        if not log_file.exists():
            return ""
        lines = log_file.read_text().splitlines()
        return "\n".join(lines[-tail:])
```

### File 2: `tests/test_process.py`

```python
import pytest
from gateway.process import ProcessManager

@pytest.mark.asyncio
async def test_start_stop(tmp_path):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  sleeper:
    provider: ollama
    model: qwen
    command: sleep
providers:
  ollama:
    base_url: http://localhost:11434
""")
    pm = ProcessManager(config_path=str(cfg))
    state = await pm.start("sleeper")
    assert state.status == "running"
    assert state.pid is not None
    stopped = await pm.stop("sleeper")
    assert stopped.status == "stopped"
```

### Verify

```bash
cd /root/gateway
pytest tests/test_process.py -v
```

Expected: test passes.

### Commit

```bash
cd /root/gateway
git add gateway/process.py tests/test_process.py
git commit -m "feat: add process manager"
git push origin main
```

---

## Task 5: FastAPI Server

### File 1: `gateway/server.py`

```python
from __future__ import annotations
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse, StreamingResponse
from gateway.config import load_config, get_agent, get_provider
from gateway.ollama_provider import OllamaProvider

def create_app(config_path: str | None = None) -> FastAPI:
    app = FastAPI(title="Gateway")

    try:
        config = load_config(config_path)
    except Exception:
        config = None

    if config:
        for agent_id in config.agents:
            agent = get_agent(config, agent_id)
            provider_profile = get_provider(config, agent.provider)

            @app.post(f"/agents/{agent_id}/v1/chat/completions")
            async def chat(request: Request, agent_id=agent_id, agent=agent, provider_profile=provider_profile):
                provider = OllamaProvider(provider_profile)
                body = await request.json()
                messages = body.get("messages", [])
                model = body.get("model", agent.model)
                stream = body.get("stream", False)
                if stream:
                    async def event_stream():
                        gen = await provider.chat(messages, model, stream=True)
                        async for chunk in gen:
                            yield f"data: {__import__('json').dumps(chunk)}\n\n"
                        yield "data: [DONE]\n\n"
                    return StreamingResponse(event_stream(), media_type="text/event-stream")
                result = await provider.chat(messages, model, stream=False)
                return JSONResponse(content=result)

    @app.get("/health")
    async def health():
        return {"status": "ok"}

    return app
```

### File 2: `tests/test_server.py`

```python
from fastapi.testclient import TestClient
from gateway.server import create_app
from pathlib import Path
import gateway.server as server

def test_health():
    app = create_app()
    client = TestClient(app)
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}

def test_chat_endpoint(tmp_path, monkeypatch):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  testbot:
    provider: ollama
    model: qwen
    command: echo
providers:
  ollama:
    base_url: http://localhost:11434
""")
    app = create_app(config_path=str(cfg))
    client = TestClient(app)

    class FakeProvider:
        async def chat(self, messages, model, stream=False):
            return {"message": {"content": "hi"}}

    monkeypatch.setattr(server, "OllamaProvider", lambda p: FakeProvider())

    r = client.post("/agents/testbot/v1/chat/completions", json={"messages": []})
    assert r.status_code == 200
    assert r.json()["message"]["content"] == "hi"
```

### Verify

```bash
cd /root/gateway
pytest tests/test_server.py -v
```

Expected: 2 tests pass.

### Commit

```bash
cd /root/gateway
git add gateway/server.py tests/test_server.py
git commit -m "feat: add FastAPI server with per-agent chat endpoint"
git push origin main
```

---

## Task 6: CLI

### File 1: `gateway/cli.py`

```python
from __future__ import annotations
import asyncio
import typer
from gateway.process import ProcessManager
from gateway.server import create_app
import uvicorn

app = typer.Typer()

@app.command()
def server(host: str = "127.0.0.1", port: int = 8000):
    uvicorn.run(create_app(), host=host, port=port)

@app.command()
def launch(agent_id: str, config: str | None = None):
    pm = ProcessManager(config_path=config)
    state = asyncio.run(pm.start(agent_id))
    print(f"Started {state.agent_id} pid={state.pid}")

@app.command()
def stop(agent_id: str, config: str | None = None):
    pm = ProcessManager(config_path=config)
    state = asyncio.run(pm.stop(agent_id))
    print(f"Stopped {state.agent_id}")

@app.command()
def list(config: str | None = None):
    pm = ProcessManager(config_path=config)
    states = pm.list()
    for s in states:
        print(f"{s.agent_id:20} {s.status:12} pid={s.pid}")

@app.command()
def logs(agent_id: str, tail: int = 50, config: str | None = None):
    pm = ProcessManager(config_path=config)
    print(pm.logs(agent_id, tail=tail))
```

### File 2: `tests/test_cli.py`

```python
from typer.testing import CliRunner
from gateway.cli import app

runner = CliRunner()

def test_help():
    r = runner.invoke(app, ["--help"])
    assert r.exit_code == 0
    assert "server" in r.output

def test_server_help():
    r = runner.invoke(app, ["server", "--help"])
    assert r.exit_code == 0
```

### Verify

```bash
cd /root/gateway
gateway --help
gateway server --help
pytest tests/test_cli.py -v
```

Expected: CLI help prints; 2 tests pass.

### Commit

```bash
cd /root/gateway
git add gateway/cli.py tests/test_cli.py
git commit -m "feat: add Typer CLI"
git push origin main
```

---

## Task 7: End-to-End Test

### File 1: `tests/test_e2e.py`

```python
import pytest
from fastapi.testclient import TestClient
from gateway.server import create_app
from gateway.process import ProcessManager
import gateway.server as server

@pytest.mark.asyncio
async def test_launch_and_hit_endpoint(tmp_path, monkeypatch):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  echoer:
    provider: ollama
    model: qwen
    command: sleep
providers:
  ollama:
    base_url: http://localhost:11434
""")
    pm = ProcessManager(config_path=str(cfg))
    state = await pm.start("echoer")
    assert state.status == "running"

    class FakeProvider:
        async def chat(self, messages, model, stream=False):
            return {"message": {"content": "pong"}}

    monkeypatch.setattr(server, "OllamaProvider", lambda p: FakeProvider())

    app = create_app(config_path=str(cfg))
    client = TestClient(app)
    r = client.post("/agents/echoer/v1/chat/completions", json={"messages": []})
    assert r.status_code == 200
    assert r.json()["message"]["content"] == "pong"

    await pm.stop("echoer")
    assert pm.list()[0].status == "stopped"
```

### Verify

```bash
cd /root/gateway
pytest tests/test_e2e.py -v
```

Expected: test passes.

### Commit

```bash
cd /root/gateway
git add tests/test_e2e.py
git commit -m "test: add end-to-end launch + endpoint test"
git push origin main
```

---

## Task 8: Full Test Run and README Update

### Step 1: Run all tests

```bash
cd /root/gateway
pytest -v
```

Expected: 8+ tests pass.

### Step 2: Update `README.md`

Append:

```markdown
## Run tests

```bash
pytest -v
```
```

### Step 3: Commit

```bash
cd /root/gateway
git add README.md
git commit -m "docs: update README with test command"
git push origin main
```

---

## What This Delivers

- A working CLI: `gateway server`, `gateway launch codex`, `gateway stop codex`, `gateway list`, `gateway logs codex`.
- A FastAPI server with `/agents/{id}/v1/chat/completions`.
- One provider: Ollama.
- Simple config via YAML.
- Subprocess lifecycle management.
- Full pytest coverage for the above.

## What Is NOT In This 9B Plan

- LM Studio, vLLM, Anthropic, generic OpenAI providers.
- Multiple harness types.
- Custom endpoints beyond chat.
- Auth, rate limiting, dashboard.

Add those in Phase 2 once Phase 1 is solid and tested.
