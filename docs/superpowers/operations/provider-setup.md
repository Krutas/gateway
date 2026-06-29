# Provider Setup Guides

## Ollama

Install:
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
ollama pull qwen2.5-coder:32b
```

Verify:
```bash
curl http://localhost:11434/api/tags
```

Gateway config:
```yaml
providers:
  ollama:
    base_url: http://localhost:11434
```

## LM Studio

1. Download from https://lmstudio.ai/
2. Load a model.
3. Start the local server: **Developer** → **Local Server** → **Start Server**.
4. Note the port (default `1234`).

Verify:
```bash
curl http://localhost:1234/v1/models
```

Gateway config:
```yaml
providers:
  lmstudio:
    base_url: http://localhost:1234/v1
    kind: openai
```

## vLLM

Install:
```bash
pip install vllm
```

Run:
```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-Coder-32B-Instruct \
  --port 8000
```

Verify:
```bash
curl http://localhost:8000/v1/models
```

Gateway config:
```yaml
providers:
  vllm:
    base_url: http://localhost:8000/v1
    kind: openai
```

## Generic OpenAI-Compatible Server

Any server exposing `/v1/chat/completions` and `/v1/models` works.

Gateway config:
```yaml
providers:
  myserver:
    base_url: http://localhost:9999/v1
    kind: openai
    api_key: ${MY_SERVER_API_KEY}
```

## Anthropic API

For the real Anthropic API:
```yaml
providers:
  anthropic:
    base_url: https://api.anthropic.com
    kind: anthropic
    api_key: ${ANTHROPIC_API_KEY}
```

For a local Anthropic-compatible shim, use the same `kind: anthropic` with a local base URL.
