# Gateway Architecture Diagrams

## 1. High-Level Component Diagram

```mermaid
flowchart TB
    subgraph CLI["CLI Shell"]
        A["gateway ollama launch codex"]
        B["gateway list"]
        C["gateway logs"]
    end

    subgraph Config["Config"]
        CFG["~/.gateway/agents.yaml"]
    end

    subgraph Gateway["Gateway Runtime"]
        PM["Process Manager"]
        HV["Harness Registry"]
        PR["Provider Registry"]
        API["FastAPI Router"]
    end

    subgraph Agents["Agent Subprocesses"]
        C1["codex"]
        C2["claude code"]
        C3["generic script"]
    end

    subgraph LLMs["LLM Backends"]
        O["Ollama :11434"]
        L["LM Studio :1234"]
        V["vLLM :8000"]
        OA["OpenAI-compatible"]
        AN["Anthropic-compatible"]
    end

    A -- parses --> CLI
    CLI -- loads --> CFG
    CLI -- dispatches --> PM
    PM -- spawns --> Agents
    Agents -- LLM calls --> API
    API -- forwards via --> PR
    PR -- native API --> LLMs
    PM -- uses --> HV
    HV -- configures --> Agents
```

## 2. Request Flow for `gateway ollama launch codex`

```mermaid
sequenceDiagram
    participant U as User
    participant CLI as gateway CLI
    participant CFG as Config
    participant PM as ProcessManager
    participant H as CodexHarness
    participant S as codex subprocess
    participant API as /agents/codex/v1/chat/completions
    participant OA as OllamaAdapter
    participant O as Ollama

    U->>CLI: gateway ollama launch codex
    CLI->>CFG: load agents.yaml
    CFG-->>CLI: AgentProfile codex -> provider ollama
    CLI->>PM: start("ollama", "codex")
    PM->>H: build_command + build_env
    H-->>PM: ["codex"], env OPENAI_BASE_URL=/agents/codex/v1
    PM->>S: create_subprocess_exec(...)
    S->>API: POST /v1/chat/completions
    API->>OA: provider.chat(messages, model, stream)
    OA->>O: POST /api/chat
    O-->>OA: response chunks
    OA-->>API: OpenAI-shaped response
    API-->>S: SSE / JSON
```

## 3. Provider Adapter Inheritance

```mermaid
classDiagram
    BaseProvider <|-- OllamaProvider
    BaseProvider <|-- OpenAICompatibleProvider
    BaseProvider <|-- AnthropicProvider
    OpenAICompatibleProvider <|-- LMStudioProvider
    OpenAICompatibleProvider <|-- VLLMProvider

    class BaseProvider {
        +profile
        +chat(messages, model, stream, **kwargs)
        +models()
        +health()
    }
    class OllamaProvider {
        +name = "ollama"
    }
    class OpenAICompatibleProvider {
        +name = "openai_compatible"
    }
    class AnthropicProvider {
        +name = "anthropic"
    }
    class LMStudioProvider {
        +name = "lmstudio"
    }
    class VLLMProvider {
        +name = "vllm"
    }
```

## 4. Endpoint Mapping

```mermaid
flowchart LR
    A["/health"] --> GW["Gateway health"]
    B["/agents/{id}/v1/chat/completions"] -- OpenAI shim
    C["/agents/{id}/v1/messages"] -- Anthropic shim
    D["/agents/{id}/v1/models"] -- Optional model list proxy
    E["/agents/{id}/{custom}"] -- User-defined static/proxy endpoint

    B --> P["Provider registry"]
    C --> P
    D --> P
    E --> H["Custom handler"]
```
