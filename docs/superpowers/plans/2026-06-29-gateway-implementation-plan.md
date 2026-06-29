# Gateway AI Agent Launcher — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Python CLI (`gateway`) and FastAPI gateway that launches open-source AI agent harnesses against local LLM providers (Ollama, LM Studio, vLLM, OpenAI-compatible, Anthropic-compatible), exposing per-agent custom HTTP endpoints.

**Architecture:** A Typer/argparse CLI dispatches to a process manager that spawns harness subprocesses. Each harness is pointed at a per-agent FastAPI route (`/agents/{id}/v1/*`). Provider adapters translate between OpenAI/Anthropic request shapes and the native backend APIs. Configuration lives in `~/.gateway/agents.yaml` and is validated with Pydantic.

**Tech Stack:** Python 3.11, Pydantic 2, FastAPI, Uvicorn, Typer, httpx, PyYAML, pytest, pytest-asyncio.

---

## File Structure

```
/root/gateway/
├── pyproject.toml
├── README.md
├── chat.log
├── gateway/
│   ├── __init__.py
│   ├── cli.py
│   ├── config.py
│   ├── models.py
│   ├── exceptions.py
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── registry.py
│   │   ├── ollama.py
│   │   ├── lmstudio.py
│   │   ├── vllm.py
│   │   ├── openai_compatible.py
│   │   └── anthropic.py
│   ├── harnesses/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── registry.py
│   │   ├── codex.py
│   │   └── generic.py
│   ├── process/
│   │   ├── __init__.py
│   │   └── manager.py
│   └── server/
│       ├── __init__.py
│       ├── app.py
│       └── routes.py
└── tests/
    ├── conftest.py
    ├── test_config.py
    ├── test_ollama.py
    ├── test_lmstudio.py
    ├── test_vllm.py
    ├── test_openai_compatible.py
    ├── test_anthropic.py
    ├── test_process_manager.py
    └── test_server.py
```

---

### Task 1: Project Scaffolding

**Files:**
- Create: `/root/gateway/pyproject.toml`
- Create: `/root/gateway/README.md`
- Create: `/root/gateway/gateway/__init__.py`
- Create: `/root/gateway/tests/conftest.py`
- Create: `/root/gateway/.gitignore`

- [ ] **Step 1: Create `pyproject.toml`**

```toml
[project]
name = "gateway"
version = "0.1.0"
description = "Launch and route open-source AI agents across local LLM providers"
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
dev = ["pytest>=8.0", "pytest-asyncio>=0.23"]

[project.scripts]
gateway = "gateway.cli:main"

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["."]
include = ["gateway*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

- [ ] **Step 2: Create `gateway/__init__.py`**

```python
__version__ = "0.1.0"
```

- [ ] **Step 3: Create `tests/conftest.py`**

```python
import pytest
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent))
```

- [ ] **Step 4: Create `.gitignore`**

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

- [ ] **Step 5: Create `README.md` with install and usage**

```markdown
# Gateway

Launch open-source AI agents against local LLM providers with per-agent endpoints.

## Install

```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

## Usage

```bash
# Start the gateway HTTP server
gateway server

# Launch an agent against a provider
gateway ollama launch codex
gateway lmstudio launch codex
gateway vllm launch opencode

# Stop / restart
gateway ollama stop codex
gateway ollama restart codex

# List running agents and show logs
gateway list
gateway logs codex
```

## Configuration

Edit `~/.gateway/agents.yaml`.

```yaml
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    harness: codex
    env:
      CUSTOM_VAR: value
    endpoints:
      - path: /health
        type: static
        response: {"status":"ok"}

providers:
  ollama:
    base_url: http://localhost:11434
  lmstudio:
    base_url: http://localhost:1234/v1
    kind: openai
  vllm:
    base_url: http://localhost:8000/v1
    kind: openai
```
```

- [ ] **Step 6: Install the package and run a smoke test**

Run:
```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
gateway --help
```

Expected: CLI prints help and exits 0.

- [ ] **Step 7: Commit**

```bash
cd /root/gateway
git init
git add pyproject.toml README.md .gitignore gateway/__init__.py tests/conftest.py
git commit -m "chore: scaffold gateway project"
```

---

### Task 2: Configuration Engine

**Files:**
- Create: `/root/gateway/gateway/models.py`
- Create: `/root/gateway/gateway/exceptions.py`
- Create: `/root/gateway/gateway/config.py`
- Create: `/root/gateway/tests/test_config.py`

- [ ] **Step 1: Write failing config test**

Create `/root/gateway/tests/test_config.py`:

```python
import os
from pathlib import Path
from gateway.config import load_config, get_agent, get_provider
from gateway.models import AgentProfile, ProviderProfile

def test_load_config_from_path(tmp_path):
    cfg_path = tmp_path / "agents.yaml"
    cfg_path.write_text("""
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    harness: codex
providers:
  ollama:
    base_url: http://localhost:11434
""")
    cfg = load_config(cfg_path)
    assert "codex" in cfg.agents
    assert cfg.agents["codex"].model == "qwen2.5-coder:32b"
    assert "ollama" in cfg.providers

def test_get_agent_and_provider(tmp_path):
    cfg_path = tmp_path / "agents.yaml"
    cfg_path.write_text("""
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    harness: codex
providers:
  ollama:
    base_url: http://localhost:11434
""")
    cfg = load_config(cfg_path)
    agent = get_agent(cfg, "codex")
    provider = get_provider(cfg, "ollama")
    assert isinstance(agent, AgentProfile)
    assert isinstance(provider, ProviderProfile)
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /root/gateway
pytest tests/test_config.py -v
```

Expected: two failures because `load_config`, `get_agent`, `get_provider`, and the models do not exist.

- [ ] **Step 3: Implement Pydantic models and config loader**

Create `/root/gateway/gateway/models.py`:

```python
from __future__ import annotations
from typing import Any, Literal
from pydantic import BaseModel, Field, field_validator

class CustomEndpoint(BaseModel):
    path: str
    type: Literal["openai", "anthropic", "static", "proxy"]
    response: dict[str, Any] | None = None
    target_url: str | None = None
    method: Literal["GET", "POST"] = "GET"

class AgentProfile(BaseModel):
    provider: str
    model: str
    harness: str
    env: dict[str, str] = Field(default_factory=dict)
    endpoints: list[CustomEndpoint] = Field(default_factory=list)
    port: int | None = None
    restart_policy: Literal["never", "always", "on_failure"] = "on_failure"
    max_restarts: int = 3

class ProviderProfile(BaseModel):
    base_url: str
    kind: Literal["ollama", "openai", "anthropic"] = "openai"
    api_key: str | None = None
    timeout: float = 60.0

    @field_validator("base_url")
    @classmethod
    def strip_trailing_slash(cls, v: str) -> str:
        return v.rstrip("/")

class Config(BaseModel):
    agents: dict[str, AgentProfile]
    providers: dict[str, ProviderProfile]

class AgentState(BaseModel):
    agent_id: str
    pid: int | None = None
    port: int | None = None
    status: Literal["running", "stopped", "crashed", "restarting"] = "stopped"
    log_file: str | None = None
```

Create `/root/gateway/gateway/exceptions.py`:

```python
class GatewayError(Exception):
    pass

class ConfigError(GatewayError):
    pass

class AgentNotFoundError(GatewayError):
    pass

class ProviderNotFoundError(GatewayError):
    pass
```

Create `/root/gateway/gateway/config.py`:

```python
from __future__ import annotations
import os
from pathlib import Path
from typing import Any
import yaml
from gateway.models import AgentProfile, Config, ProviderProfile
from gateway.exceptions import AgentNotFoundError, ConfigError, ProviderNotFoundError

def default_config_path() -> Path:
    return Path(os.environ.get("GATEWAY_CONFIG", Path.home() / ".gateway" / "agents.yaml"))

def load_config(path: Path | str | None = None) -> Config:
    p = Path(path) if path else default_config_path()
    if not p.exists():
        raise ConfigError(f"Config file not found: {p}")
    try:
        raw = yaml.safe_load(p.read_text()) or {}
    except yaml.YAMLError as e:
        raise ConfigError(f"Invalid YAML in {p}: {e}") from e
    try:
        return Config.model_validate(raw)
    except Exception as e:
        raise ConfigError(f"Invalid config: {e}") from e

def get_agent(config: Config, agent_id: str) -> AgentProfile:
    if agent_id not in config.agents:
        raise AgentNotFoundError(f"Agent '{agent_id}' not found in config")
    return config.agents[agent_id]

def get_provider(config: Config, provider_id: str) -> ProviderProfile:
    if provider_id not in config.providers:
        raise ProviderNotFoundError(f"Provider '{provider_id}' not found in config")
    return config.providers[provider_id]
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd /root/gateway
pytest tests/test_config.py -v
```

Expected: both tests pass.

- [ ] **Step 5: Commit**

```bash
cd /root/gateway
git add gateway/models.py gateway/exceptions.py gateway/config.py tests/test_config.py
git commit -m "feat: add Pydantic config engine with YAML loader"
```

---

### Task 3: Provider Base + Ollama Adapter

**Files:**
- Create: `/root/gateway/gateway/providers/base.py`
- Create: `/root/gateway/gateway/providers/ollama.py`
- Create: `/root/gateway/gateway/providers/registry.py`
- Create: `/root/gateway/tests/test_ollama.py`

- [ ] **Step 1: Write the provider base and registry**

Create `/root/gateway/gateway/providers/base.py`:

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Any, AsyncIterator

class BaseProvider(ABC):
    name: str

    def __init__(self, profile: Any) -> None:
        self.profile = profile

    @abstractmethod
    async def chat(self, messages: list[dict[str, Any]], model: str, stream: bool = False, **kwargs: Any) -> AsyncIterator[dict[str, Any]] | dict[str, Any]:
        raise NotImplementedError

    @abstractmethod
    async def models(self) -> list[str]:
        raise NotImplementedError

    @abstractmethod
    async def health(self) -> bool:
        raise NotImplementedError
```

Create `/root/gateway/gateway/providers/registry.py`:

```python
from __future__ import annotations
from gateway.models import ProviderProfile
from gateway.providers.base import BaseProvider
from gateway.providers.ollama import OllamaProvider

_REGISTRY: dict[str, type[BaseProvider]] = {
    "ollama": OllamaProvider,
}

def get_provider(provider_id: str, profile: ProviderProfile) -> BaseProvider:
    if provider_id not in _REGISTRY:
        raise ValueError(f"Unknown provider '{provider_id}'")
    return _REGISTRY[provider_id](profile)

def register_provider(name: str, cls: type[BaseProvider]) -> None:
    _REGISTRY[name] = cls
```

- [ ] **Step 2: Write the Ollama adapter**

Create `/root/gateway/gateway/providers/ollama.py`:

```python
from __future__ import annotations
from typing import Any, AsyncIterator
import httpx
from gateway.providers.base import BaseProvider

class OllamaProvider(BaseProvider):
    name = "ollama"

    def __init__(self, profile):
        super().__init__(profile)
        self.client = httpx.AsyncClient(base_url=profile.base_url, timeout=profile.timeout)

    async def chat(self, messages, model, stream=False, **kwargs):
        url = "/api/chat"
        payload = {
            "model": model,
            "messages": messages,
            "stream": stream,
            "options": kwargs.get("options", {}),
        }
        if stream:
            async def _stream():
                async with self.client.stream("POST", url, json=payload) as response:
                    response.raise_for_status()
                    async for line in response.aiter_lines():
                        if line:
                            import json
                            yield json.loads(line)
            return _stream()
        response = await self.client.post(url, json=payload)
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

- [ ] **Step 3: Write the Ollama tests**

Create `/root/gateway/tests/test_ollama.py`:

```python
import pytest
import httpx
from gateway.models import ProviderProfile
from gateway.providers.ollama import OllamaProvider

@pytest.fixture
def provider():
    return OllamaProvider(ProviderProfile(base_url="http://localhost:11434", kind="ollama"))

@pytest.mark.asyncio
async def test_ollama_models(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def json(self):
            return {"models": [{"name": "qwen2.5-coder:32b"}]}
        def raise_for_status(self):
            pass

    async def fake_get(url):
        return FakeResponse()

    monkeypatch.setattr(provider.client, "get", fake_get)
    models = await provider.models()
    assert models == ["qwen2.5-coder:32b"]

@pytest.mark.asyncio
async def test_ollama_health_ok(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def raise_for_status(self):
            pass
    async def fake_get(url):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.health() is True
```

- [ ] **Step 4: Run the tests**

```bash
cd /root/gateway
pytest tests/test_ollama.py -v
```

Expected: tests pass.

- [ ] **Step 5: Commit**

```bash
cd /root/gateway
git add gateway/providers/ tests/test_ollama.py
git commit -m "feat: add provider base and Ollama adapter"
```

---

### Task 4: OpenAI-Compatible Providers (LM Studio, vLLM, Generic)

**Files:**
- Create: `/root/gateway/gateway/providers/openai_compatible.py`
- Create: `/root/gateway/gateway/providers/lmstudio.py`
- Create: `/root/gateway/gateway/providers/vllm.py`
- Modify: `/root/gateway/gateway/providers/registry.py`
- Create: `/root/gateway/tests/test_lmstudio.py`
- Create: `/root/gateway/tests/test_vllm.py`
- Create: `/root/gateway/tests/test_openai_compatible.py`

- [ ] **Step 1: Implement the shared OpenAI-compatible adapter**

Create `/root/gateway/gateway/providers/openai_compatible.py`:

```python
from __future__ import annotations
from typing import Any, AsyncIterator
import json
import httpx
from gateway.providers.base import BaseProvider

class OpenAICompatibleProvider(BaseProvider):
    name = "openai_compatible"

    def __init__(self, profile):
        super().__init__(profile)
        headers = {}
        if profile.api_key:
            headers["Authorization"] = f"Bearer {profile.api_key}"
        self.client = httpx.AsyncClient(base_url=profile.base_url, timeout=profile.timeout, headers=headers)

    async def chat(self, messages, model, stream=False, **kwargs):
        payload = {
            "model": model,
            "messages": messages,
            "stream": stream,
        }
        for key in ("temperature", "top_p", "max_tokens", "stop"):
            if key in kwargs:
                payload[key] = kwargs[key]
        if stream:
            async def _stream():
                async with self.client.stream("POST", "/v1/chat/completions", json=payload) as response:
                    response.raise_for_status()
                    async for line in response.aiter_lines():
                        if line.startswith("data: "):
                            data = line[6:]
                            if data == "[DONE]":
                                break
                            yield json.loads(data)
            return _stream()
        response = await self.client.post("/v1/chat/completions", json=payload)
        response.raise_for_status()
        return response.json()

    async def models(self):
        response = await self.client.get("/v1/models")
        response.raise_for_status()
        data = response.json()
        return [m["id"] for m in data.get("data", [])]

    async def health(self):
        try:
            response = await self.client.get("/v1/models")
            return response.status_code == 200
        except Exception:
            return False
```

- [ ] **Step 2: Create LM Studio and vLLM subclasses**

Create `/root/gateway/gateway/providers/lmstudio.py`:

```python
from gateway.providers.openai_compatible import OpenAICompatibleProvider

class LMStudioProvider(OpenAICompatibleProvider):
    name = "lmstudio"
```

Create `/root/gateway/gateway/providers/vllm.py`:

```python
from gateway.providers.openai_compatible import OpenAICompatibleProvider

class VLLMProvider(OpenAICompatibleProvider):
    name = "vllm"
```

- [ ] **Step 3: Register the new providers**

Modify `/root/gateway/gateway/providers/registry.py` to:

```python
from __future__ import annotations
from gateway.models import ProviderProfile
from gateway.providers.base import BaseProvider
from gateway.providers.ollama import OllamaProvider
from gateway.providers.lmstudio import LMStudioProvider
from gateway.providers.vllm import VLLMProvider
from gateway.providers.openai_compatible import OpenAICompatibleProvider

_REGISTRY: dict[str, type[BaseProvider]] = {
    "ollama": OllamaProvider,
    "lmstudio": LMStudioProvider,
    "vllm": VLLMProvider,
    "openai": OpenAICompatibleProvider,
}

def get_provider(provider_id: str, profile: ProviderProfile) -> BaseProvider:
    if provider_id not in _REGISTRY:
        raise ValueError(f"Unknown provider '{provider_id}'")
    return _REGISTRY[provider_id](profile)

def register_provider(name: str, cls: type[BaseProvider]) -> None:
    _REGISTRY[name] = cls
```

- [ ] **Step 4: Write tests for LM Studio / vLLM / generic**

Create `/root/gateway/tests/test_lmstudio.py`:

```python
import pytest
from gateway.models import ProviderProfile
from gateway.providers.lmstudio import LMStudioProvider

@pytest.fixture
def provider():
    return LMStudioProvider(ProviderProfile(base_url="http://localhost:1234/v1", kind="openai"))

@pytest.mark.asyncio
async def test_lmstudio_models(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def json(self):
            return {"data": [{"id": "qwen2.5-coder"}]}
        def raise_for_status(self):
            pass
    async def fake_get(url):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.models() == ["qwen2.5-coder"]
```

Create `/root/gateway/tests/test_vllm.py`:

```python
import pytest
from gateway.models import ProviderProfile
from gateway.providers.vllm import VLLMProvider

@pytest.fixture
def provider():
    return VLLMProvider(ProviderProfile(base_url="http://localhost:8000/v1", kind="openai"))

@pytest.mark.asyncio
async def test_vllm_health(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def raise_for_status(self):
            pass
    async def fake_get(url):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "get", fake_get)
    assert await provider.health() is True
```

Create `/root/gateway/tests/test_openai_compatible.py`:

```python
import pytest
from gateway.models import ProviderProfile
from gateway.providers.openai_compatible import OpenAICompatibleProvider

@pytest.fixture
def provider():
    return OpenAICompatibleProvider(ProviderProfile(base_url="http://localhost:1111/v1", kind="openai"))

@pytest.mark.asyncio
async def test_openai_chat_non_stream(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def json(self):
            return {"choices": [{"message": {"content": "hi"}}]}
        def raise_for_status(self):
            pass
    async def fake_post(url, json):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "post", fake_post)
    result = await provider.chat([{"role": "user", "content": "hello"}], "model-x")
    assert result["choices"][0]["message"]["content"] == "hi"
```

- [ ] **Step 5: Run the tests**

```bash
cd /root/gateway
pytest tests/test_lmstudio.py tests/test_vllm.py tests/test_openai_compatible.py -v
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
cd /root/gateway
git add gateway/providers/ tests/test_lmstudio.py tests/test_vllm.py tests/test_openai_compatible.py
git commit -m "feat: add LM Studio, vLLM and generic OpenAI-compatible providers"
```

---

### Task 5: Anthropic-Compatible Provider

**Files:**
- Create: `/root/gateway/gateway/providers/anthropic.py`
- Modify: `/root/gateway/gateway/providers/registry.py`
- Create: `/root/gateway/tests/test_anthropic.py`

- [ ] **Step 1: Implement the Anthropic adapter**

Create `/root/gateway/gateway/providers/anthropic.py`:

```python
from __future__ import annotations
from typing import Any, AsyncIterator
import json
import httpx
from gateway.providers.base import BaseProvider

class AnthropicProvider(BaseProvider):
    name = "anthropic"

    def __init__(self, profile):
        super().__init__(profile)
        headers = {"content-type": "application/json"}
        if profile.api_key:
            headers["x-api-key"] = profile.api_key
            headers["anthropic-version"] = "2023-06-01"
        self.client = httpx.AsyncClient(base_url=profile.base_url, timeout=profile.timeout, headers=headers)

    async def chat(self, messages, model, stream=False, **kwargs):
        system = None
        msgs = []
        for m in messages:
            if m.get("role") == "system":
                system = m.get("content")
            else:
                msgs.append({"role": m.get("role"), "content": m.get("content")})
        payload: dict[str, Any] = {
            "model": model,
            "messages": msgs,
            "max_tokens": kwargs.get("max_tokens", 1024),
        }
        if system:
            payload["system"] = system
        if stream:
            payload["stream"] = True
            async def _stream():
                async with self.client.stream("POST", "/v1/messages", json=payload) as response:
                    response.raise_for_status()
                    async for line in response.aiter_lines():
                        if line.startswith("data: "):
                            data = line[6:]
                            if data == "[DONE]":
                                break
                            yield json.loads(data)
            return _stream()
        response = await self.client.post("/v1/messages", json=payload)
        response.raise_for_status()
        return response.json()

    async def models(self):
        try:
            response = await self.client.get("/v1/models")
            response.raise_for_status()
            data = response.json()
            return [m["id"] for m in data.get("data", [])]
        except Exception:
            return []

    async def health(self):
        try:
            response = await self.client.get("/v1/models")
            return response.status_code == 200
        except Exception:
            return False
```

- [ ] **Step 2: Register Anthropic provider**

Modify `/root/gateway/gateway/providers/registry.py` to import and register:

```python
from gateway.providers.anthropic import AnthropicProvider

_REGISTRY: dict[str, type[BaseProvider]] = {
    "ollama": OllamaProvider,
    "lmstudio": LMStudioProvider,
    "vllm": VLLMProvider,
    "openai": OpenAICompatibleProvider,
    "anthropic": AnthropicProvider,
}
```

- [ ] **Step 3: Write Anthropic tests**

Create `/root/gateway/tests/test_anthropic.py`:

```python
import pytest
from gateway.models import ProviderProfile
from gateway.providers.anthropic import AnthropicProvider

@pytest.fixture
def provider():
    return AnthropicProvider(ProviderProfile(base_url="http://localhost:1111", kind="anthropic"))

@pytest.mark.asyncio
async def test_anthropic_chat(provider, monkeypatch):
    class FakeResponse:
        status_code = 200
        def json(self):
            return {"content": [{"type": "text", "text": "hello"}]}
        def raise_for_status(self):
            pass
    async def fake_post(url, json):
        return FakeResponse()
    monkeypatch.setattr(provider.client, "post", fake_post)
    result = await provider.chat([{"role": "user", "content": "hi"}], "claude-3")
    assert result["content"][0]["text"] == "hello"
```

- [ ] **Step 4: Run the tests**

```bash
cd /root/gateway
pytest tests/test_anthropic.py -v
```

Expected: test passes.

- [ ] **Step 5: Commit**

```bash
cd /root/gateway
git add gateway/providers/anthropic.py gateway/providers/registry.py tests/test_anthropic.py
git commit -m "feat: add Anthropic-compatible provider adapter"
```

---

### Task 6: Harness Abstraction

**Files:**
- Create: `/root/gateway/gateway/harnesses/base.py`
- Create: `/root/gateway/gateway/harnesses/codex.py`
- Create: `/root/gateway/gateway/harnesses/generic.py`
- Create: `/root/gateway/gateway/harnesses/registry.py`

- [ ] **Step 1: Implement base harness interface**

Create `/root/gateway/gateway/harnesses/base.py`:

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Any

class BaseHarness(ABC):
    name: str

    @abstractmethod
    def build_command(self, agent_id: str, agent: Any, gateway_base_url: str) -> list[str]:
        raise NotImplementedError

    @abstractmethod
    def build_env(self, agent_id: str, agent: Any, gateway_base_url: str) -> dict[str, str]:
        raise NotImplementedError

    def check_install(self) -> str | None:
        return None
```

- [ ] **Step 2: Implement Codex harness**

Create `/root/gateway/gateway/harnesses/codex.py`:

```python
from __future__ import annotations
import shutil
from gateway.harnesses.base import BaseHarness

class CodexHarness(BaseHarness):
    name = "codex"

    def build_command(self, agent_id, agent, gateway_base_url):
        return ["codex"]

    def build_env(self, agent_id, agent, gateway_base_url):
        return {
            "OPENAI_BASE_URL": f"{gateway_base_url}/agents/{agent_id}/v1",
            "OPENAI_API_KEY": agent.env.get("OPENAI_API_KEY", "gateway-local"),
        }

    def check_install(self):
        if not shutil.which("codex"):
            return "Codex CLI not found. Install with: npm install -g @openai/codex"
        return None
```

- [ ] **Step 3: Implement generic executable harness**

Create `/root/gateway/gateway/harnesses/generic.py`:

```python
from __future__ import annotations
import shutil
from gateway.harnesses.base import BaseHarness

class GenericHarness(BaseHarness):
    name = "generic"

    def build_command(self, agent_id, agent, gateway_base_url):
        import shlex
        raw = agent.env.get("COMMAND", "echo")
        return shlex.split(raw)

    def build_env(self, agent_id, agent, gateway_base_url):
        env = dict(agent.env)
        env.setdefault("GATEWAY_BASE_URL", gateway_base_url)
        env.setdefault("AGENT_ID", agent_id)
        return env

    def check_install(self):
        cmd = self.build_command("check", type("A", (), {"env": {}})(), "")
        if cmd and not shutil.which(cmd[0]):
            return f"Command '{cmd[0]}' not found in PATH"
        return None
```

- [ ] **Step 4: Implement harness registry**

Create `/root/gateway/gateway/harnesses/registry.py`:

```python
from __future__ import annotations
from gateway.models import AgentProfile
from gateway.harnesses.base import BaseHarness
from gateway.harnesses.codex import CodexHarness
from gateway.harnesses.generic import GenericHarness

_REGISTRY: dict[str, BaseHarness] = {
    "codex": CodexHarness(),
    "generic": GenericHarness(),
}

def get_harness(name: str) -> BaseHarness:
    if name not in _REGISTRY:
        raise ValueError(f"Unknown harness '{name}'")
    return _REGISTRY[name]

def register_harness(name: str, harness: BaseHarness) -> None:
    _REGISTRY[name] = harness
```

- [ ] **Step 5: Add a harness unit test**

Create `/root/gateway/tests/test_harnesses.py`:

```python
from gateway.harnesses.registry import get_harness
from gateway.models import AgentProfile

def test_codex_harness_env():
    harness = get_harness("codex")
    agent = AgentProfile(provider="ollama", model="qwen", harness="codex", env={"OPENAI_API_KEY": "abc"})
    env = harness.build_env("myagent", agent, "http://localhost:9000")
    assert env["OPENAI_BASE_URL"] == "http://localhost:9000/agents/myagent/v1"
    assert env["OPENAI_API_KEY"] == "abc"

def test_generic_harness_command():
    harness = get_harness("generic")
    agent = AgentProfile(provider="ollama", model="qwen", harness="generic", env={"COMMAND": "python -m agent"})
    cmd = harness.build_command("myagent", agent, "http://localhost:9000")
    assert cmd == ["python", "-m", "agent"]
```

- [ ] **Step 6: Run tests**

```bash
cd /root/gateway
pytest tests/test_harnesses.py -v
```

Expected: pass.

- [ ] **Step 7: Commit**

```bash
cd /root/gateway
git add gateway/harnesses/ tests/test_harnesses.py
git commit -m "feat: add harness abstraction with codex and generic adapters"
```

---

### Task 7: Process Manager

**Files:**
- Create: `/root/gateway/gateway/process/manager.py`
- Create: `/root/gateway/tests/test_process_manager.py`

- [ ] **Step 1: Write the process manager**

Create `/root/gateway/gateway/process/manager.py`:

```python
from __future__ import annotations
import asyncio
import os
import shutil
import socket
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any
from gateway.models import AgentProfile, AgentState
from gateway.config import load_config, get_agent, get_provider
from gateway.harnesses.registry import get_harness
from gateway.providers.registry import get_provider as get_provider_instance

@dataclass
class AgentRuntime:
    state: AgentState
    process: asyncio.subprocess.Process | None = None

class ProcessManager:
    def __init__(self, config_path: str | None = None, gateway_base_url: str = "http://localhost:8000"):
        self.config_path = config_path
        self.gateway_base_url = gateway_base_url
        self._agents: dict[str, AgentRuntime] = {}
        self._log_dir = Path.home() / ".gateway" / "logs"
        self._log_dir.mkdir(parents=True, exist_ok=True)

    def _load(self):
        return load_config(self.config_path)

    def _free_port(self) -> int:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.bind(("127.0.0.1", 0))
            return s.getsockname()[1]

    async def start(self, provider_id: str, agent_id: str) -> AgentState:
        if agent_id in self._agents and self._agents[agent_id].process and self._agents[agent_id].process.returncode is None:
            return self._agents[agent_id].state
        config = self._load()
        agent = get_agent(config, agent_id)
        if agent.provider != provider_id:
            raise ValueError(f"Agent '{agent_id}' is configured for provider '{agent.provider}', not '{provider_id}'")
        harness = get_harness(agent.harness)
        warning = harness.check_install()
        if warning:
            print(f"Warning: {warning}")
        port = agent.port or self._free_port()
        log_file = self._log_dir / f"{agent_id}.log"
        env = os.environ.copy()
        env.update(harness.build_env(agent_id, agent, self.gateway_base_url))
        env.update(agent.env)
        command = harness.build_command(agent_id, agent, self.gateway_base_url)
        with open(log_file, "a") as lf:
            process = await asyncio.create_subprocess_exec(
                *command,
                env=env,
                stdout=lf,
                stderr=asyncio.subprocess.STDOUT,
            )
        state = AgentState(agent_id=agent_id, pid=process.pid, port=port, status="running", log_file=str(log_file))
        self._agents[agent_id] = AgentRuntime(state=state, process=process)
        return state

    async def stop(self, agent_id: str) -> AgentState:
        runtime = self._agents.get(agent_id)
        if not runtime or not runtime.process:
            raise ValueError(f"Agent '{agent_id}' is not running")
        runtime.process.terminate()
        try:
            await asyncio.wait_for(runtime.process.wait(), timeout=5.0)
        except asyncio.TimeoutError:
            runtime.process.kill()
            await runtime.process.wait()
        runtime.state.status = "stopped"
        return runtime.state

    def list(self) -> list[AgentState]:
        for runtime in self._agents.values():
            if runtime.process and runtime.process.returncode is not None and runtime.state.status == "running":
                runtime.state.status = "crashed"
        return [r.state for r in self._agents.values()]

    async def restart(self, provider_id: str, agent_id: str) -> AgentState:
        if agent_id in self._agents:
            await self.stop(agent_id)
        return await self.start(provider_id, agent_id)

    def logs(self, agent_id: str, tail: int = 50) -> str:
        runtime = self._agents.get(agent_id)
        if not runtime or not runtime.state.log_file:
            raise ValueError(f"No log file for agent '{agent_id}'")
        log_file = Path(runtime.state.log_file)
        if not log_file.exists():
            return ""
        lines = log_file.read_text().splitlines()
        return "\n".join(lines[-tail:])
```

- [ ] **Step 2: Write process manager test**

Create `/root/gateway/tests/test_process_manager.py`:

```python
import pytest
from pathlib import Path
from gateway.process.manager import ProcessManager
from gateway.models import AgentProfile

def make_config(tmp_path):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  sleeper:
    provider: ollama
    model: qwen
    harness: generic
    env:
      COMMAND: "sleep 10"
providers:
  ollama:
    base_url: http://localhost:11434
""")
    return cfg

@pytest.mark.asyncio
async def test_start_and_stop(tmp_path, monkeypatch):
    cfg = make_config(tmp_path)
    pm = ProcessManager(config_path=str(cfg), gateway_base_url="http://localhost:8000")
    state = await pm.start("ollama", "sleeper")
    assert state.status == "running"
    assert state.pid is not None
    await pm.stop("sleeper")
    stopped = pm.list()[0]
    assert stopped.status == "stopped"
```

- [ ] **Step 3: Run the test**

```bash
cd /root/gateway
pytest tests/test_process_manager.py -v
```

Expected: test passes (sleep process spawns and terminates).

- [ ] **Step 4: Commit**

```bash
cd /root/gateway
git add gateway/process/manager.py tests/test_process_manager.py
git commit -m "feat: add process manager for spawning and monitoring agents"
```

---

### Task 8: CLI with Provider-Prefixed Commands

**Files:**
- Create: `/root/gateway/gateway/cli.py`

- [ ] **Step 1: Implement the CLI**

Create `/root/gateway/gateway/cli.py`:

```python
from __future__ import annotations
import argparse
import asyncio
import sys
from gateway.process.manager import ProcessManager

KNOWN_PROVIDERS = {"ollama", "lmstudio", "vllm", "openai", "anthropic"}
GLOBAL_COMMANDS = {"list", "logs", "server", "help"}

def _print_state(state):
    print(f"{state.agent_id:20} {state.status:12} pid={state.pid} port={state.port}")

async def _run_provider_command(args):
    pm = ProcessManager()
    if args.action == "launch":
        state = await pm.start(args.command, args.agent_id)
        _print_state(state)
    elif args.action == "stop":
        state = await pm.stop(args.agent_id)
        _print_state(state)
    elif args.action == "restart":
        state = await pm.restart(args.command, args.agent_id)
        _print_state(state)
    else:
        print(f"Unknown action '{args.action}'")
        sys.exit(1)

async def _run_global_command(args):
    pm = ProcessManager()
    if args.command == "list":
        states = pm.list()
        print(f"{'AGENT':20} {'STATUS':12} PID/PORT")
        for s in states:
            _print_state(s)
    elif args.command == "logs":
        if not args.agent_id:
            print("Usage: gateway logs <agent-id>")
            sys.exit(1)
        print(pm.logs(args.agent_id, tail=args.tail))
    elif args.command == "server":
        from gateway.server.app import create_app
        import uvicorn
        app = create_app(config_path=pm.config_path)
        uvicorn.run(app, host=args.host, port=args.port)

def main(argv=None):
    parser = argparse.ArgumentParser(prog="gateway")
    parser.add_argument("command", nargs="?", help="Provider name or global command")
    parser.add_argument("action", nargs="?", help="Action when provider is given")
    parser.add_argument("agent_id", nargs="?", help="Agent identifier")
    parser.add_argument("--config", dest="config_path", default=None)
    parser.add_argument("--tail", type=int, default=50)
    parser.add_argument("--host", default="127.0.0.1")
    parser.add_argument("--port", type=int, default=8000)
    args = parser.parse_args(argv)

    if args.command == "server":
        from gateway.server.app import create_app
        import uvicorn
        app = create_app(config_path=args.config_path)
        uvicorn.run(app, host=args.host, port=args.port)
        return

    if args.command in GLOBAL_COMMANDS:
        asyncio.run(_run_global_command(args))
    elif args.command in KNOWN_PROVIDERS:
        if args.action not in {"launch", "stop", "restart"}:
            parser.error(f"Action must be one of: launch, stop, restart")
        if not args.agent_id:
            parser.error("Agent ID is required")
        asyncio.run(_run_provider_command(args))
    else:
        parser.error(f"Unknown command/provider '{args.command}'. Known providers: {', '.join(sorted(KNOWN_PROVIDERS))}")

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Manually verify CLI help and smoke commands**

Run:
```bash
cd /root/gateway
gateway --help
gateway list
```

Expected: help prints; `gateway list` shows empty table.

- [ ] **Step 3: Commit**

```bash
cd /root/gateway
git add gateway/cli.py
git commit -m "feat: add provider-prefixed CLI"
```

---

### Task 9: FastAPI Endpoint Router

**Files:**
- Create: `/root/gateway/gateway/server/routes.py`
- Create: `/root/gateway/gateway/server/app.py`
- Create: `/root/gateway/tests/test_server.py`

- [ ] **Step 1: Implement per-agent routes**

Create `/root/gateway/gateway/server/routes.py`:

```python
from __future__ import annotations
from typing import Any
from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import JSONResponse, StreamingResponse
from gateway.config import load_config, get_agent, get_provider
from gateway.providers.registry import get_provider as get_provider_instance

def _resolve_provider(config_path: str | None, agent_id: str):
    config = load_config(config_path)
    agent = get_agent(config, agent_id)
    provider_profile = get_provider(config, agent.provider)
    return agent, provider_profile

def build_agent_router(config_path: str | None, agent_id: str) -> APIRouter:
    router = APIRouter()

    @router.post("/v1/chat/completions")
    async def chat_completions(request: Request):
        try:
            agent, profile = _resolve_provider(config_path, agent_id)
        except Exception as e:
            raise HTTPException(status_code=404, detail=str(e))
        provider = get_provider_instance(agent.provider, profile)
        body = await request.json()
        messages = body.get("messages", [])
        model = body.get("model", agent.model)
        stream = body.get("stream", False)
        if stream:
            async def event_stream():
                gen = await provider.chat(messages, model, stream=True, **body)
                async for chunk in gen:
                    yield f"data: {__import__('json').dumps(chunk)}\n\n"
                yield "data: [DONE]\n\n"
            return StreamingResponse(event_stream(), media_type="text/event-stream")
        result = await provider.chat(messages, model, stream=False, **body)
        return JSONResponse(content=result)

    @router.post("/v1/messages")
    async def anthropic_messages(request: Request):
        try:
            agent, profile = _resolve_provider(config_path, agent_id)
        except Exception as e:
            raise HTTPException(status_code=404, detail=str(e))
        provider = get_provider_instance(agent.provider, profile)
        body = await request.json()
        messages = body.get("messages", [])
        model = body.get("model", agent.model)
        stream = body.get("stream", False)
        if stream:
            async def event_stream():
                gen = await provider.chat(messages, model, stream=True, **body)
                async for chunk in gen:
                    yield f"data: {__import__('json').dumps(chunk)}\n\n"
                yield "data: [DONE]\n\n"
            return StreamingResponse(event_stream(), media_type="text/event-stream")
        result = await provider.chat(messages, model, stream=False, **body)
        return JSONResponse(content=result)

    # custom endpoints
    try:
        agent, _ = _resolve_provider(config_path, agent_id)
        for ep in agent.endpoints:
            if ep.type == "static":
                @router.get(ep.path)
                @router.post(ep.path)
                async def _static(response_body=ep.response):
                    return JSONResponse(content=response_body or {})
    except Exception:
        pass

    return router
```

- [ ] **Step 2: Implement the FastAPI app factory**

Create `/root/gateway/gateway/server/app.py`:

```python
from __future__ import annotations
from fastapi import FastAPI
from gateway.config import load_config
from gateway.server.routes import build_agent_router

def create_app(config_path: str | None = None) -> FastAPI:
    app = FastAPI(title="Gateway", version="0.1.0")

    try:
        config = load_config(config_path)
    except Exception:
        config = None

    if config:
        for agent_id in config.agents:
            router = build_agent_router(config_path, agent_id)
            app.include_router(router, prefix=f"/agents/{agent_id}")

    @app.get("/health")
    async def health():
        return {"status": "ok"}

    return app
```

- [ ] **Step 3: Write server test**

Create `/root/gateway/tests/test_server.py`:

```python
import pytest
from fastapi.testclient import TestClient
from gateway.server.app import create_app
from pathlib import Path

def test_server_health():
    app = create_app()
    client = TestClient(app)
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}

def test_agent_static_endpoint(tmp_path, monkeypatch):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  testagent:
    provider: ollama
    model: qwen
    harness: generic
    env:
      COMMAND: "echo"
    endpoints:
      - path: /status
        type: static
        response: {"ok": true}
providers:
  ollama:
    base_url: http://localhost:11434
""")
    app = create_app(config_path=str(cfg))
    client = TestClient(app)
    response = client.get("/agents/testagent/status")
    assert response.status_code == 200
    assert response.json() == {"ok": True}
```

- [ ] **Step 4: Run tests**

```bash
cd /root/gateway
pytest tests/test_server.py -v
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
cd /root/gateway
git add gateway/server/ tests/test_server.py
git commit -m "feat: add FastAPI per-agent endpoint router"
```

---

### Task 10: End-to-End Integration Test

**Files:**
- Create: `/root/gateway/tests/test_integration.py`

- [ ] **Step 1: Write integration test that launches a dummy agent and hits its endpoint**

Create `/root/gateway/tests/test_integration.py`:

```python
import pytest
from pathlib import Path
from fastapi.testclient import TestClient
from gateway.process.manager import ProcessManager
from gateway.server.app import create_app

@pytest.mark.asyncio
async def test_launch_dummy_agent_and_hit_endpoint(tmp_path, monkeypatch):
    cfg = tmp_path / "agents.yaml"
    cfg.write_text("""
agents:
  echoer:
    provider: ollama
    model: qwen
    harness: generic
    env:
      COMMAND: "python -c 'import time; time.sleep(1)'"
    endpoints:
      - path: /ping
        type: static
        response: {"ping": "pong"}
providers:
  ollama:
    base_url: http://localhost:11434
""")
    pm = ProcessManager(config_path=str(cfg), gateway_base_url="http://localhost:8000")
    state = await pm.start("ollama", "echoer")
    assert state.status == "running"

    app = create_app(config_path=str(cfg))
    client = TestClient(app)
    response = client.get("/agents/echoer/ping")
    assert response.status_code == 200
    assert response.json() == {"ping": "pong"}

    await pm.stop("echoer")
    assert pm.list()[0].status == "stopped"
```

- [ ] **Step 2: Run the integration test**

```bash
cd /root/gateway
pytest tests/test_integration.py -v
```

Expected: passes.

- [ ] **Step 3: Commit**

```bash
cd /root/gateway
git add tests/test_integration.py
git commit -m "test: add end-to-end launch + endpoint integration test"
```

---

### Task 11: Full Test Suite, Lint, and Final Documentation

**Files:**
- Modify: `/root/gateway/README.md`

- [ ] **Step 1: Run the full test suite**

```bash
cd /root/gateway
pytest -v
```

Expected: all tests pass.

- [ ] **Step 2: Add sample config to README**

Append to `/root/gateway/README.md`:

```markdown
## Sample `~/.gateway/agents.yaml`

```yaml
agents:
  codex:
    provider: ollama
    model: qwen2.5-coder:32b
    harness: codex
    endpoints:
      - path: /health
        type: static
        response: {"status":"ok"}
  claude-code-lm:
    provider: lmstudio
    model: claude-3-5-sonnet
    harness: generic
    env:
      COMMAND: "claude"
      ANTHROPIC_BASE_URL: http://localhost:8000/agents/claude-code-lm/v1
      ANTHROPIC_API_KEY: gateway-local

providers:
  ollama:
    base_url: http://localhost:11434
  lmstudio:
    base_url: http://localhost:1234/v1
    kind: openai
  vllm:
    base_url: http://localhost:8001/v1
    kind: openai
  anthropic:
    base_url: http://localhost:8002
    kind: anthropic
```
```

- [ ] **Step 3: Final commit**

```bash
cd /root/gateway
git add README.md
git commit -m "docs: complete README with sample config"
```

---

## Spec Coverage Check

| Requirement | Task |
|-------------|------|
| Launch multiple AI agents from terminal | Task 8 CLI |
| Provider-prefixed commands like `gateway ollama launch codex` | Task 8 CLI |
| Custom endpoints per agent | Task 9 routes + Task 2 endpoints config |
| Ollama support | Task 3 |
| LM Studio support | Task 4 |
| vLLM support | Task 4 |
| Other OpenAI-compatible providers | Task 4 generic `openai` provider |
| Anthropic API support | Task 5 |
| Open-source agent harnesses (Codex + generic) | Task 6 |
| Process management and logs | Task 7 |
| Config-driven | Task 2 |

## Placeholder Scan

No TBD, TODO, or vague steps remain. Every task includes exact file paths, complete code blocks, and exact verification commands.

## Execution Handoff

**Plan complete and saved to `/root/gateway/docs/superpowers/plans/2026-06-29-gateway-implementation-plan.md`.**

Two execution options:

1. **Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

**Which approach?**
