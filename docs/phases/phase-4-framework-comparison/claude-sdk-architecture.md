# Claude Agent SDK Architecture - Implementation B

## Repository
[https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)

## Overview

This document provides detailed analysis of the Claude Agent SDK implementation of the NetBox infrastructure agent, including the intelligent routing and model selection features added in December 2025.

For framework comparison guidance, see [Framework Comparison Overview](../../framework-comparison/overview.md).

## Core Architecture

### Technology Stack

**Backend**:
- Python 3.13+ with async/await patterns
- FastAPI for WebSocket server
- Claude Agent SDK for managed orchestration
- Pydantic v2 for type-safe models
- pytest for comprehensive testing (83+ tests)

**Frontend**:
- Nuxt 3 with Vue Composition API
- TypeScript for type safety
- WebSocket for real-time streaming
- TailwindCSS for styling

**Integration**:
- Model Context Protocol (MCP) for NetBox tool access
- NetBoxLabs MCP server (stdio subprocess)
- Environment-based configuration

### Architecture Diagram

```
┌─────────────────┐         WebSocket          ┌─────────────────┐
│   Nuxt 3 UI     │◄────────────────────────────│  FastAPI Server │
│   (Frontend)    │                             │    (Backend)    │
│                 │   Model Selection           │                 │
│ ModelSelector   │──────────────────────────── │  /models        │
│ Component       │                             │  endpoint       │
└─────────────────┘                             └────────┬────────┘
                                                         │
                                                         │ Uses
                                                         ▼
                                                ┌────────────────┐
                                                │ ChatAgent      │
                                                │ (SDK Client)   │
                                                │                │
                                                │ model: None    │ ◄─ Intelligent Routing
                                                │   or explicit  │
                                                └────────┬───────┘
                                                         │
                                                         │ MCP Protocol
                                                         ▼
                                                ┌────────────────┐
                                                │ Netbox MCP     │
                                                │ Server (stdio) │
                                                └────────┬───────┘
                                                         │
                                                         │ REST API
                                                         ▼
                                                ┌────────────────┐
                                                │  NetBox API    │
                                                │ (Infrastructure)│
                                                └────────────────┘
```

## Model Context Protocol (MCP) Integration

### MCP Configuration

The SDK integrates with the NetBoxLabs MCP server via stdio subprocess:

```python
# backend/mcp_config.py
def get_netbox_mcp_config(config: Config) -> dict:
    """Generate MCP server configuration with environment isolation."""
    return {
        "netbox": {
            "command": "uv",
            "args": [
                "--directory",
                "/home/ola/dev/rnd/mcp/testmcp/netbox-mcp-server",
                "run",
                "server.py"
            ],
            "env": {
                "NETBOX_URL": str(config.netbox_url),
                "NETBOX_TOKEN": config.netbox_token,
                "LOG_LEVEL": config.log_level,
            }
        }
    }
```

**Security Benefits**:
- **Subprocess Isolation**: MCP server runs in separate process
- **Environment Scoping**: Credentials only in MCP subprocess
- **Explicit Tool Allowlist**: Only approved tools accessible
- **No Direct API Key Exposure**: Frontend never sees NetBox token

### Available MCP Tools

```python
def get_allowed_netbox_tools() -> list[str]:
    """Explicit allowlist of NetBox MCP tools."""
    return [
        "netbox_get_objects",        # Generic object retrieval with filters
        "netbox_get_object_by_id",   # Single object lookup
        "netbox_get_changelogs",     # Change history tracking
    ]
```

**Note**: Phase 2 research established these 3 generic tools + field filtering provide optimal context efficiency (90% schema overhead reduction).

## Intelligent Routing Discovery (December 2025)

### The Discovery

During API log analysis, we discovered the Claude Agent SDK performs **automatic multi-model routing** even when `model=None`:

**Observed Behavior**:
```python
# Backend configuration
options = ClaudeAgentOptions(model=None)  # "Automatic" mode

# SDK makes intelligent routing decisions:
# Query: "How many NetBox sites are there?"
API Calls Observed:
1. claude-haiku-4-5-20251001    - Tool execution (netbox_get_objects)
2. claude-haiku-4-5-20251001    - Additional tool calls
3. claude-sonnet-4-20241120     - Response synthesis (with caching)

Cost: ~$0.004 vs $0.018 if all Sonnet
Savings: 76%
```

**Pattern**:
| Operation | Model Selected | Reason |
|-----------|---------------|---------|
| MCP tool execution | Haiku 4.5 | Fast, cheap ($0.25/1M tokens) |
| Simple queries | Haiku 4.5 | Cost optimization |
| Medium complexity | Sonnet 4 | Balanced with caching |
| Complex analysis | Sonnet 4.5 | Extended reasoning |

### Cost Optimization Impact

**Real-World Example** (Query: "List all NetBox sites"):

Without Intelligent Routing (Sonnet-only):
```
6 API calls × claude-sonnet-4-5
~6,000 tokens × $3.00/1M = $0.018
```

With Intelligent Routing (Auto mode):
```
5 API calls × claude-haiku-4-5 = 5,000 tokens × $0.25/1M = $0.00125
1 API call × claude-sonnet-4 = 1,000 tokens × $3.00/1M = $0.003
Total: $0.00425
Savings: 76%
```

**Aggregate Impact**: 70-80% cost reduction across typical workload.

## Model Selection Implementation

### Architecture

**Four Model Options**:
1. **Auto (Default)** - `model=None` - SDK intelligent routing
2. **Haiku 4.5** - `model="claude-haiku-4-5-20250925"` - Fast/cheap
3. **Sonnet 4.5** - `model="claude-sonnet-4-5-20250929"` - Balanced
4. **Opus 4** - `model="claude-opus-4-20250514"` - Maximum capability

### Backend Implementation

```python
# backend/agent.py
class ChatAgent:
    def __init__(self, config: Config, model: str | None = None) -> None:
        """
        Initialize agent with optional model specification.

        Args:
            config: Application configuration
            model: Explicit model ID or None for automatic routing
        """
        self.model = model
        self.config = config

        self.options = ClaudeAgentOptions(
            model=model,  # None = automatic, or explicit model ID
            fallback_model=None,
            mcp_servers=get_netbox_mcp_config(config),
            allowed_tools=get_allowed_netbox_tools(),
            permission_mode="acceptEdits",
            system_prompt={
                "type": "preset",
                "preset": "claude_code",
                "append": (
                    "You are a NetBox infrastructure assistant..."
                )
            }
        )
```

### API Endpoints

```python
# backend/api.py
@app.get("/models", response_model=list[ModelInfo])
async def get_models() -> list[ModelInfo]:
    """Get available Claude models."""
    return [
        ModelInfo(id="auto", name="Claude (Automatic Selection)",
                 provider="anthropic", available=True),
        ModelInfo(id="claude-sonnet-4-5-20250929", name="Claude Sonnet 4.5",
                 provider="anthropic", available=True),
        ModelInfo(id="claude-opus-4-20250514", name="Claude Opus 4",
                 provider="anthropic", available=True),
        ModelInfo(id="claude-haiku-4-5-20250925", name="Claude Haiku 4.5",
                 provider="anthropic", available=True),
    ]
```

### WebSocket Protocol Enhancement

**Model Switching**:
```python
# Client → Server
{
  "type": "model_change",
  "model": "claude-sonnet-4-5-20250929"  # or "auto"
}

# Server → Client (confirmation)
{
  "type": "model_changed",
  "content": "Switched to Claude Sonnet 4.5. Context has been reset.",
  "completed": false,
  "metadata": {
    "model": {
      "model": "claude-sonnet-4-5-20250929",
      "model_display": "Claude Sonnet 4.5",
      "is_automatic": false
    }
  }
}
```

**Session Management**:
- Model switching always resets conversation context
- Old session closed, new session started with new model
- Frontend preserves message history for display
- Backend conversation context fully reset

### Frontend Implementation

**ModelSelector Component** (`frontend/components/ModelSelector.vue`):
```vue
<template>
  <div class="model-selector">
    <button @click="showModal = true" class="model-selector-button">
      <div class="model-info">
        <span class="model-label">Model:</span>
        <span class="model-name">{{ getModelDisplayName }}</span>
      </div>
      <svg class="chevron-icon"><!-- chevron down --></svg>
    </button>

    <Teleport to="body">
      <Transition name="modal">
        <div v-if="showModal" class="modal-overlay">
          <!-- Model selection modal -->
          <div class="modal-content">
            <div class="warning-message" v-if="selectedModel !== pendingModel">
              ⚠️ Switching models will reset your conversation context
            </div>

            <div class="models-list">
              <div v-for="model in availableModels"
                   :class="{ 'selected': model.id === selectedModel }"
                   @click="selectModelOption(model.id)">
                {{ model.name }}
              </div>
            </div>

            <button @click="confirmModelSwitch">Switch Model</button>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>
```

**State Management** (`frontend/composables/useModelSelection.ts`):
```typescript
export const useModelSelection = () => {
  const selectedModel = useState<string>('selected-model', () => 'auto')
  const availableModels = useState<ModelConfig[]>('available-models', () => [])

  const fetchModels = async () => {
    const response = await $fetch('/api/models')
    availableModels.value = response
  }

  const selectModel = (modelId: string) => {
    selectedModel.value = modelId
    localStorage.setItem('selected-model', modelId)
  }

  return {
    selectedModel: readonly(selectedModel),
    availableModels: readonly(availableModels),
    fetchModels,
    selectModel
  }
}
```

## WebSocket Streaming Architecture

### Session Lifecycle

```python
# backend/api.py
@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket) -> None:
    await websocket.accept()

    # Create agent with initial model (None = automatic)
    current_model = None
    agent = ChatAgent(config, model=current_model)

    try:
        # Start session
        await agent.start_session()
        logger.info(f"Agent session started (model: {current_model or 'auto'})")

        # Send connection confirmation
        await websocket.send_json({
            "type": "connected",
            "content": "Connected to Netbox chatbox.",
            "metadata": {"model": agent.get_model_info()}
        })

        # Message loop
        while True:
            data = await websocket.receive_text()
            request = json.loads(data)

            # Handle model change
            if request.get("type") == "model_change":
                await agent.close_session()
                current_model = None if request["model"] == "auto" else request["model"]
                agent = ChatAgent(config, model=current_model)
                await agent.start_session()
                # Send confirmation...
                continue

            # Handle regular message
            async for chunk in agent.query(request["message"]):
                await websocket.send_json(chunk.model_dump())

    finally:
        await agent.close_session()
```

### Message Types

```python
# backend/models.py
class StreamChunk(BaseModel):
    """Streaming response chunk."""
    type: Literal[
        "text",           # Response text
        "tool_use",       # Tool being executed
        "tool_result",    # Tool result (optional)
        "thinking",       # Thinking process (internal)
        "error",          # Error message
        "connected",      # Initial connection
        "reset_complete", # Session reset confirmation
        "model_changed"   # Model switch confirmation
    ]
    content: str
    completed: bool = False
    metadata: dict | None = None  # For model info, etc.
```

## Type Safety with Pydantic

### Model Definitions

```python
# backend/models.py
class Config(BaseModel):
    """Environment configuration with validation."""
    anthropic_api_key: str = Field(..., env="ANTHROPIC_API_KEY")
    netbox_url: HttpUrl = Field(..., env="NETBOX_URL")
    netbox_token: str = Field(..., env="NETBOX_TOKEN")
    log_level: str = Field("INFO", env="LOG_LEVEL")

    @field_validator("netbox_token")
    def validate_token(cls, v: str) -> str:
        if not v or len(v) < 20:
            raise ValueError("Invalid NetBox token")
        return v

    model_config = ConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )
```

**Benefits**:
- Runtime validation with clear error messages
- Editor autocomplete and IntelliSense
- Mypy static type checking (strict mode)
- API contract enforcement

## Production Deployment Considerations

### Testing Infrastructure

**83+ Unit Tests** covering:
- Configuration validation
- Agent lifecycle management
- WebSocket protocol handling
- Model selection and switching
- MCP integration (mocked)
- Error handling and recovery

**Running Tests**:
```bash
# All tests
uv run pytest

# With coverage
uv run pytest --cov=backend --cov-report=term-missing

# Specific module
uv run pytest tests/test_agent.py -v
```

### Performance Metrics

**Response Latency**:
- WebSocket Connection: <50ms
- First Chunk: 200-500ms
- Tool Execution: 1-3s per NetBox API call
- Total Query Time: 2-5s (typical)

**Resource Usage**:
- Backend Memory: ~150MB (FastAPI + SDK + MCP)
- MCP Server Memory: ~50MB (NetBoxLabs server)
- WebSocket: <1MB per session
- Total per User: ~200MB

**Cost Efficiency**:
- With Intelligent Routing: 70-80% cost reduction
- Auto mode (default): Optimal cost/performance
- Explicit Haiku: Maximum cost minimization
- Explicit Opus: Maximum capability (higher cost)

### Security Controls

**1. MCP Subprocess Isolation**:
- NetBox credentials never in main process
- Separate process lifecycle
- Environment variable scoping

**2. Tool Allowlist**:
- Explicit tool approval required
- Cannot execute unapproved operations
- SDK permission system enforced

**3. API Key Protection**:
- Never committed to version control
- Environment variable loading
- No frontend exposure

**4. CORS Configuration**:
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=config.cors_origins.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Monitoring and Observability

**Structured Logging**:
```python
logger.info(
    f"Query completed in {duration_ms}ms, {num_turns} turns"
)
logger.info(f"Model: {current_model or 'auto'}")
logger.debug(f"Tool use: {tool_name}")
```

**Health Check**:
```python
@app.get("/health")
async def health_check() -> HealthResponse:
    return HealthResponse(
        status="healthy",
        service="netbox-chatbox-api",
        version=__version__,
    )
```

## Attempted: Ollama/LiteLLM Integration (Reverted)

### Goal

Enable local Ollama models (Qwen 2.5:14b) alongside Anthropic models via LiteLLM proxy.

### Problems Encountered

**1. Tool Results Not Displayed** (Critical):
- Qwen 2.5 model executed MCP tools successfully
- Backend logs showed data returned
- Model generated acknowledgments but omitted actual data
- Example: "I'll check that" instead of "There are 24 sites"

**2. Loss of SDK Features**:
- Routing Anthropic through LiteLLM would break:
  - MCP protocol integration
  - Thinking block streaming
  - Prompt caching (84% hit rate lost)
  - Permission system
  - Intelligent routing (70-80% cost savings lost)

**3. Architecture Complexity**:
- Dual-path routing decisions
- Separate agent implementations
- Docker Compose for LiteLLM
- Health check orchestration
- Additional failure modes

### Decision to Revert

**Reasons**:
1. Tool result display failure unacceptable for infrastructure queries
2. SDK feature loss outweighed local model benefits
3. Architecture complexity not justified by value
4. Framework incompatibility (SDK designed for direct API access)

**Reversion Actions**:
- Removed all LiteLLM-specific code
- Deleted docker-compose.litellm.yml
- Simplified to Anthropic-only
- Restored direct SDK integration exclusively

**Learning**: Framework-native patterns (direct API access) superior to proxy workarounds. Tool result integration quality varies significantly across models.

See [ADR-0027](../../adr/0027-intelligent-routing-and-model-selection.md) for complete analysis.

## Key Architectural Patterns

### 1. Context Manager Pattern

```python
# Manual context management for long-lived sessions
async def start_session(self) -> None:
    self.client = ClaudeSDKClient(options=self.options)
    await self.client.__aenter__()
    self.session_active = True
```

**Benefits**:
- Proper resource cleanup
- Long-lived conversations
- Explicit lifecycle control

### 2. Streaming Response Pattern

```python
async def query(self, message: str) -> AsyncIterator[StreamChunk]:
    await self.client.query(message)
    async for msg in self.client.receive_response():
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    yield StreamChunk(type="text", content=block.text)
```

**Benefits**:
- Real-time feedback
- Progress visibility
- Type-safe message processing

### 3. Type-Safe Configuration

```python
class Config(BaseSettings):
    model_config = ConfigDict(
        env_file=".env",
        case_sensitive=False
    )
```

**Benefits**:
- Automatic .env loading
- Runtime validation
- Clear error messages

## Comparison: Deepagents vs Claude SDK

| Dimension | Deepagents | Claude SDK | Winner |
|-----------|-----------|-----------|--------|
| Initial Development | 46 hours | 2.5 hours | **Claude SDK (18x faster)** |
| Code Complexity | 1,200 LOC | 400 LOC | **Claude SDK (3x simpler)** |
| Maintenance | 8-12h/month | 2-4h/month | **Claude SDK (4-6x lower)** |
| Flexibility | Unlimited | Limited | **Deepagents** |
| Production Ready | Custom | Built-in | **Claude SDK** |
| Cost Optimization | Manual | Automatic (70-80%) | **Claude SDK** |
| First Year TCO | $70k-145k | $18k-38k | **Claude SDK (4-8x lower)** |

## References

- **Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)
- **ADR-0026**: [Claude SDK Project Requirements Package](../../adr/0026-claude-sdk-project-requirements-package.md)
- **ADR-0027**: [Intelligent Routing and Model Selection](../../adr/0027-intelligent-routing-and-model-selection.md)
- **Model Selection Guide**: Repository `docs/MODEL_SELECTION.md`
- **Claude Agent SDK Docs**: Repository `examples/claude-agent-sdk-python.md`
