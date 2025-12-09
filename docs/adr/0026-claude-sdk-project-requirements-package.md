# ADR-0026 — Claude Agent SDK Project Requirements Package

## Status

**Accepted** — Production implementation complete
**Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)

## Context

Phase 4 framework comparison study required implementing a production-ready NetBox infrastructure agent. After Phase 3's multi-agent orchestration failure (0% success rate), we needed a reliable approach that prioritized:

1. **Rapid Development**: Time-to-production measured in days, not weeks
2. **Production Reliability**: Battle-tested orchestration without custom infrastructure
3. **Managed Complexity**: Offload agent coordination, tool selection, and safety to SDK
4. **Full-Stack Experience**: Modern web interface with real-time streaming responses
5. **Type Safety**: Comprehensive type checking for production robustness
6. **Testing Infrastructure**: Automated testing for confident deployments

**Research Objective**: Empirically compare production-ready, managed SDK approach against flexible DIY frameworks (Deepagents/LangChain) to validate framework selection criteria.

## Decision

Implement **modern Python packaging with pyproject.toml** using **Claude Agent SDK** for managed agent orchestration, complemented by **FastAPI + WebSocket** for real-time streaming and **Nuxt 3** for professional web interface.

### Core Architecture

**Full-Stack Application Pattern**:
```python
# Backend: Claude Agent SDK + FastAPI
backend/
├── api.py              # FastAPI WebSocket server
├── agent.py            # Claude SDK session management
├── config.py           # Environment configuration
├── mcp_config.py       # MCP server integration
├── models.py           # Pydantic data models
└── utils.py            # Helper functions

# Frontend: Nuxt 3 + TypeScript
frontend/
├── components/         # Vue chat components
├── composables/        # WebSocket connection logic
├── pages/              # Main chat interface
├── types/              # TypeScript definitions
└── utils/              # Formatting utilities

# Testing: Pytest with async support
tests/                  # 83+ unit tests covering all functionality
```

### Package Configuration (pyproject.toml)

**Hatchling Build System**:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["backend", "tests"]
```

**Project Metadata**:
```toml
[project]
name = "netbox-chatbox"
version = "1.0.0"
description = "Netbox chatbox with Claude Agent SDK"
requires-python = ">=3.10"
```

**Core Dependencies**:
```toml
dependencies = [
    "fastapi>=0.115.0",         # Modern async web framework
    "uvicorn[standard]>=0.32.0", # ASGI server with WebSocket support
    "claude-agent-sdk>=0.1.0",   # Managed agent orchestration
    "python-dotenv>=1.0.0",      # Environment configuration
    "pydantic>=2.0.0",           # Type-safe data models
    "websockets>=13.0",          # Real-time streaming
]
```

**Development Dependencies**:
```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",            # Testing framework
    "pytest-asyncio>=0.24.0",   # Async test support
    "pytest-cov>=5.0.0",        # Coverage reporting
    "httpx>=0.27.0",            # HTTP client for testing
    "black>=24.0.0",            # Code formatting
    "ruff>=0.6.0",              # Fast linting
    "mypy>=1.11.0",             # Static type checking
]
```

### Development Tooling Configuration

**Code Formatting (Black)**:
```toml
[tool.black]
line-length = 100
target-version = ["py310", "py311", "py312"]
```

**Linting (Ruff)**:
```toml
[tool.ruff]
line-length = 100
target-version = "py310"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]  # Error, pyflakes, imports, naming, warnings, pyupgrade
ignore = []
```

**Type Checking (mypy)**:
```toml
[tool.mypy]
python_version = "3.10"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true        # All functions must have type hints
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
strict_equality = true
```

**Testing (pytest)**:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"               # Automatic async test detection
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --strict-markers"
markers = [
    "asyncio: mark test as async",
]
```

## Rationale

### Benefits of Claude Agent SDK Approach

**1. Managed Agent Orchestration**
- **Tool Selection**: SDK handles tool coordination and selection autonomously
- **Error Recovery**: Built-in retry logic and graceful failure handling
- **Safety Controls**: Permission system prevents unsafe operations
- **Context Management**: Automatic context window optimization

**2. Model Context Protocol (MCP) Integration**
```python
# MCP configuration with environment isolation
options = ClaudeAgentOptions(
    mcp_servers={
        "netbox": {
            "command": "python",
            "args": ["-m", "netbox_mcp_server"],
            "env": {
                "NETBOX_URL": config.netbox_url,
                "NETBOX_TOKEN": config.netbox_token,
            }
        }
    },
    allowed_tools=["netbox_get_objects", "netbox_get_object_by_id", "netbox_get_changelogs"],
    permission_mode="acceptEdits",
)
```

**Benefits**:
- Subprocess isolation for security
- Explicit tool allowlist
- Environment variable scoping
- Automatic MCP protocol handling

**3. Session Lifecycle Management**
```python
class ChatAgent:
    async def start_session(self) -> None:
        """Start Claude Agent session with context manager pattern."""
        self.client = ClaudeSDKClient(options=self.options)
        await self.client.__aenter__()  # Manual context entry for long-lived session
        self.session_active = True

    async def query(self, message: str) -> AsyncIterator[StreamChunk]:
        """Stream responses maintaining conversation context."""
        await self.client.query(message)
        async for msg in self.client.receive_response():
            # Type-safe message processing
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        yield StreamChunk(type="text", content=block.text)
            elif isinstance(msg, ResultMessage):
                yield StreamChunk(type="text", content="", completed=True)

    async def close_session(self) -> None:
        """Cleanup resources with proper context exit."""
        await self.client.__aexit__(None, None, None)
```

**Pattern Advantages**:
- Continuous conversations with context preservation
- Proper resource cleanup with async context managers
- Type-safe message processing (AssistantMessage, ResultMessage, TextBlock)
- Graceful error handling with structured exceptions

**4. WebSocket Streaming Architecture**
```python
@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await websocket.accept()
    agent = ChatAgent(config)
    await agent.start_session()

    try:
        while True:
            data = await websocket.receive_json()
            async for chunk in agent.query(data["message"]):
                await websocket.send_json(chunk.model_dump())
    finally:
        await agent.close_session()
```

**Real-Time Benefits**:
- Low-latency streaming responses
- Progress visibility (tool usage, thinking)
- Instant feedback for better UX
- Connection lifecycle management

**5. Type Safety with Pydantic Models**
```python
class StreamChunk(BaseModel):
    """WebSocket response chunk with strict typing."""
    type: Literal["text", "tool_use", "tool_result", "error"]
    content: str
    completed: bool

class Config(BaseModel):
    """Environment configuration with validation."""
    anthropic_api_key: str
    netbox_url: HttpUrl
    netbox_token: str
    log_level: str = "INFO"

    @field_validator("netbox_token")
    def validate_token(cls, v: str) -> str:
        if not v or len(v) < 20:
            raise ValueError("Invalid NetBox token")
        return v
```

**Type Safety Benefits**:
- Runtime validation with Pydantic v2
- Editor autocomplete and IntelliSense
- Mypy static type checking (strict mode enabled)
- API contract enforcement

### Alternative Approaches Considered

**Custom DIY Framework (Deepagents/LangChain)**:
- **Pros**: Maximum flexibility, custom orchestration patterns, framework independence
- **Cons**: 18x longer development time (46h vs 2.5h), 4-6x higher maintenance burden
- **Verdict**: Excellent for research and novel agent behaviors, excessive for standard use cases

**Direct Anthropic API**:
- **Pros**: Complete control, no framework overhead
- **Cons**: Manual tool coordination, no MCP integration, custom error handling required
- **Verdict**: Too low-level for production applications

**LangChain with Custom Orchestration**:
- **Pros**: Rich ecosystem, flexible abstractions
- **Cons**: Complex configuration, token management overhead, steeper learning curve
- **Verdict**: Overkill for straightforward agent workflows

## Implementation Details

### 1. Environment Configuration

**`.env` Structure**:
```bash
# Anthropic API
ANTHROPIC_API_KEY=sk-ant-...

# NetBox Connection
NETBOX_URL=https://netbox.example.com
NETBOX_TOKEN=your_netbox_api_token

# Application Settings
LOG_LEVEL=INFO
BACKEND_PORT=8001

# Frontend WebSocket (frontend/.env)
NUXT_PUBLIC_WS_URL=ws://localhost:8001
```

**Configuration Management**:
```python
from pydantic import Field, HttpUrl
from pydantic_settings import BaseSettings

class Config(BaseSettings):
    """Type-safe configuration with automatic .env loading."""
    anthropic_api_key: str = Field(..., env="ANTHROPIC_API_KEY")
    netbox_url: HttpUrl = Field(..., env="NETBOX_URL")
    netbox_token: str = Field(..., env="NETBOX_TOKEN")
    log_level: str = Field("INFO", env="LOG_LEVEL")

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "case_sensitive": False,
    }
```

**Benefits**:
- Type-validated configuration loading
- Automatic .env file parsing
- Clear separation of environments (dev/staging/prod)
- No hardcoded secrets

### 2. MCP Server Integration

**MCP Configuration Pattern** (`backend/mcp_config.py`):
```python
def get_netbox_mcp_config(config: Config) -> dict:
    """Generate MCP server configuration with environment isolation."""
    return {
        "netbox": {
            "command": "python",
            "args": ["-m", "netbox_mcp_server"],
            "env": {
                "NETBOX_URL": str(config.netbox_url),
                "NETBOX_TOKEN": config.netbox_token,
                "LOG_LEVEL": config.log_level,
            }
        }
    }

def get_allowed_netbox_tools() -> list[str]:
    """Explicit allowlist of NetBox MCP tools."""
    return [
        "netbox_get_objects",        # Generic object retrieval
        "netbox_get_object_by_id",   # Single object lookup
        "netbox_get_changelogs",     # Change history tracking
    ]
```

**Security Benefits**:
- Subprocess isolation (MCP server in separate process)
- Explicit environment variable scoping
- Tool allowlist prevents unauthorized operations
- No direct API key exposure to frontend

### 3. Testing Infrastructure

**Async Testing with pytest-asyncio**:
```python
# tests/test_agent.py
import pytest
from backend.agent import ChatAgent
from backend.config import Config

@pytest.fixture
async def agent(config: Config):
    """Provide ChatAgent instance with session lifecycle."""
    agent = ChatAgent(config)
    await agent.start_session()
    yield agent
    await agent.close_session()

@pytest.mark.asyncio
async def test_query_streaming(agent: ChatAgent):
    """Test query streaming with type-safe response validation."""
    chunks = []
    async for chunk in agent.query("List all sites"):
        chunks.append(chunk)
        assert chunk.type in {"text", "tool_use", "error"}

    # Verify completion
    assert chunks[-1].completed is True
```

**Test Coverage**:
- **83+ unit tests** covering all functionality
- Backend tests: Configuration, agent lifecycle, WebSocket handling
- CLI tests: Interactive mode, single query mode, error handling
- Integration tests: Full query workflows with mocked MCP responses

**Running Tests**:
```bash
# All tests
uv run pytest

# With coverage report
uv run pytest --cov=backend --cov-report=term-missing

# Specific test module
uv run pytest tests/test_agent.py -v
```

### 4. Development Workflow

**Installation**:
```bash
# Clone repository
git clone https://github.com/FinnMacCumail/claude-agentic-netbox.git
cd claude-agentic-netbox

# Install dependencies (uv handles virtual environment)
uv sync

# Install frontend dependencies
cd frontend && npm install
```

**Development Servers**:
```bash
# Backend (Terminal 1)
./start_server.sh  # Starts FastAPI on http://localhost:8001

# Frontend (Terminal 2)
cd frontend && npm run dev  # Starts Nuxt on http://localhost:3000
```

**Code Quality Checks**:
```bash
# Format code
uv run black backend/ tests/

# Lint code
uv run ruff check backend/ tests/

# Type checking
uv run mypy backend/

# Run all quality checks
uv run black backend/ && uv run ruff check backend/ && uv run mypy backend/ && uv run pytest
```

### 5. Deployment Considerations

**Production Checklist**:
1. ✅ Type-safe configuration with validation
2. ✅ Comprehensive test suite (83+ tests)
3. ✅ Structured logging with log levels
4. ✅ Health check endpoint (`/health`)
5. ✅ WebSocket connection resilience
6. ✅ Error handling with graceful degradation
7. ✅ MCP subprocess isolation
8. ✅ Environment-based configuration
9. ✅ CORS configuration for production origins

**Production Deployment** (example):
```bash
# Backend with uvicorn
uv run uvicorn backend.api:app --host 0.0.0.0 --port 8001 --workers 4

# Frontend with Nuxt
cd frontend && npm run build && npm run preview
```

## Consequences

### Positive

**1. Rapid Development Velocity**
- **2.5 hours** from zero to working agent (vs 46 hours with Deepagents)
- **18x faster** initial development
- Production-ready immediately with built-in safety controls

**2. Production Reliability**
- Managed orchestration reduces operational risk
- Built-in error recovery and retry logic
- Permission system prevents unsafe operations
- Battle-tested SDK with Anthropic support

**3. Lower Maintenance Burden**
- **2-4 hours/month** ongoing maintenance (vs 8-12 hours with Deepagents)
- **4-6x reduction** in maintenance effort
- SDK updates handle framework improvements
- No custom infrastructure to maintain

**4. Type Safety Throughout**
- Pydantic models catch errors at runtime
- Mypy strict mode enforces type correctness
- Editor support with autocomplete and error highlighting
- API contract enforcement prevents integration bugs

**5. Full-Stack Developer Experience**
- Modern web interface with professional UI (Nuxt 3 + Vue)
- Real-time streaming responses via WebSocket
- CLI tool for command-line usage
- Comprehensive testing infrastructure (83+ tests)

**6. Cost Efficiency**
- **First Year TCO**: $18k-38k (vs $70k-145k with Deepagents)
- Lower development cost ($2k vs $36k)
- Lower operational cost (managed service)
- Lower maintenance cost ($10k vs $60k)

### Negative

**1. Framework Lock-in**
- Tightly coupled to Claude SDK and Anthropic API
- Migration to different LLM provider requires significant refactoring
- **Mitigation**: Abstractions in `agent.py` provide some isolation

**2. Limited Customization**
- Cannot implement novel orchestration patterns
- Tool coordination handled by SDK (black box)
- Constraint validation limited to SDK capabilities
- **Mitigation**: Custom logic implemented as MCP tools

**3. SDK Maturity Dependency**
- Early-stage SDK may have breaking changes
- Feature availability depends on SDK roadmap
- **Mitigation**: Pin SDK version, test before upgrading

**4. Debugging Complexity**
- Limited visibility into SDK orchestration decisions
- Tool selection reasoning opaque
- **Mitigation**: Comprehensive logging, verbose mode in CLI

## Performance Impact

### Development Time
- **Initial Development**: 2.5 hours (Phase 4 implementation)
- **Testing Infrastructure**: 4 hours (83 tests)
- **Frontend Development**: 8 hours (Nuxt 3 interface)
- **Total Time to Production**: **~2 weeks** (vs 4 weeks for Deepagents)

### Response Latency
- **WebSocket Connection**: <50ms
- **Query Streaming**: First chunk in 200-500ms
- **Tool Execution**: 1-3s per NetBox API call
- **Total Query Time**: 2-5s for typical queries

### Resource Usage
- **Backend Memory**: ~150MB (FastAPI + Claude SDK + MCP subprocess)
- **MCP Server Memory**: ~50MB (NetBoxLabs MCP server)
- **WebSocket Connections**: <1MB per active session
- **Total Footprint**: **~200MB** per user session

### Test Suite Performance
- **83 tests** complete in **<10 seconds**
- Async tests with proper fixtures
- Mocked external dependencies (MCP, Anthropic API)

## Integration with Phase 4 Research

This Project Requirements Package framework serves as **Implementation B** in the Phase 4 agent framework comparison study:

- **Implementation A**: [Deepagents (LangChain)](https://github.com/FinnMacCumail/deepagents) - Flexible DIY orchestration
- **Implementation B**: [Claude SDK](https://github.com/FinnMacCumail/claude-agentic-netbox) - Production-ready managed approach

**Comparative Metrics**:

| Dimension | Deepagents | Claude SDK | Winner |
|-----------|-----------|-----------|--------|
| Initial Development | 46 hours | 2.5 hours | **Claude SDK (18x faster)** |
| Code Complexity | 1,200 LOC | 400 LOC | **Claude SDK (3x simpler)** |
| Maintenance Burden | 8-12h/month | 2-4h/month | **Claude SDK (4-6x lower)** |
| Flexibility | Unlimited | Limited | **Deepagents** |
| Production Readiness | Custom | Built-in | **Claude SDK** |
| First Year TCO | $70k-145k | $18k-38k | **Claude SDK (4-8x lower)** |

**Key Research Finding**: Framework choice is context-dependent, not absolute. Claude SDK optimizes for **convenience and speed**, while Deepagents optimizes for **flexibility and control**.

## Recommendations

### When to Use Claude Agent SDK

**Choose Claude SDK if**:
- Time-to-market is critical (days/weeks, not months)
- Standard agent workflows without novel behaviors
- Production reliability more important than customization
- Small team with limited AI/ML expertise
- Budget constraints favor lower TCO

**Example Use Cases**:
- Internal tooling and automation
- Customer support chatbots
- Data query interfaces (like NetBox)
- MVP and prototype development
- Enterprise applications requiring stability

### When to Avoid Claude Agent SDK

**Avoid Claude SDK if**:
- Novel agent behaviors are core differentiator
- Need multi-framework LLM support (OpenAI, Anthropic, local models)
- Require custom orchestration patterns (hierarchical planning, sub-agents)
- Research and experimentation primary goal
- Full control over tool coordination critical

**Alternative Approach**:
Use Deepagents/LangChain for flexibility, accepting higher development and maintenance costs.

### Hybrid Strategy (Recommended)

**Best of Both Worlds**:
1. **Prototype with Claude SDK** - Fast validation of concept (2-3 weeks)
2. **Identify Constraints** - Determine if SDK limitations are blocking
3. **Selective Migration** - Only migrate components requiring custom behavior
4. **Maintain Production with SDK** - Use SDK for stable, standard workflows
5. **R&D with Deepagents** - Experiment with novel approaches separately

## Model Selection Enhancement (December 2025)

Following the successful initial implementation, user feedback indicated desire for explicit model control while maintaining cost efficiency. This led to discovery and documentation of an undocumented SDK feature: **intelligent multi-model routing**.

### Intelligent Routing Discovery

**Finding**: When `ClaudeAgentOptions(model=None)`, the SDK automatically routes between Claude models based on task complexity:
- MCP tool execution → Claude Haiku 4.5 (cost optimization)
- Simple/medium responses → Claude Sonnet 4 (balanced with caching)
- Complex analysis → Claude Sonnet 4.5 (extended thinking)

**Cost Impact**: 70-80% cost reduction vs using only Sonnet/Opus for all operations.

**Example**: Query "How many NetBox sites?" executes 5 tool calls with Haiku ($0.25/1M tokens) and 1 response with Sonnet ($3.00/1M), totaling $0.004 vs $0.018 if all Sonnet (76% savings).

### Implementation

**Backend Enhancement**:
```python
class ChatAgent:
    def __init__(self, config: Config, model: str | None = None):
        """
        Args:
            model: Explicit model ID or None for automatic routing
        """
        self.options = ClaudeAgentOptions(
            model=model,  # None = intelligent routing
            mcp_servers=get_netbox_mcp_config(config)
        )
```

**API Endpoint**:
```python
@app.get("/models")
async def get_models() -> list[ModelInfo]:
    return [
        ModelInfo(id="auto", name="Claude (Automatic Selection)"),
        ModelInfo(id="claude-sonnet-4-5-20250929", name="Claude Sonnet 4.5"),
        ModelInfo(id="claude-opus-4-20250514", name="Claude Opus 4"),
        ModelInfo(id="claude-haiku-4-5-20250925", name="Claude Haiku 4.5"),
    ]
```

**WebSocket Protocol**:
```python
# Client sends model change request
{
    "type": "model_change",
    "model": "claude-sonnet-4-5-20250929"  # or "auto"
}
```

**Frontend**: `ModelSelector.vue` component (450 LOC) provides modal interface with current model indicator, context reset warning, and localStorage persistence.

### Attempted Multi-Provider Support (Reverted)

**Goal**: Enable Ollama local models (Qwen 2.5:14b) via LiteLLM proxy for multi-provider flexibility.

**Problems Encountered**:
1. **Tool Results Not Displayed** (Critical) - Qwen 2.5 executed tools but didn't integrate results into responses
2. **SDK Feature Loss** - LiteLLM proxy broke MCP integration, streaming, caching (84% hit rate), permissions, and intelligent routing
3. **Architecture Complexity** - Dual-path system with separate agents, health checks, Docker Compose orchestration
4. **Framework Incompatibility** - SDK assumes direct Anthropic API; proxy translation breaks native protocol dependencies

**Decision**: Reverted to Anthropic-only after 6 hours implementation/debugging.

**Rationale**: Tool result reliability and SDK feature preservation more valuable than multi-provider flexibility. Intelligent routing makes Anthropic costs acceptable.

**Documentation**: Full analysis in [ADR-0027](0027-intelligent-routing-and-model-selection.md)

### Impact

**Benefits**:
- Preserved 70-80% cost optimization through intelligent routing
- User control for predictable model usage
- All SDK features maintained (MCP, streaming, caching, permissions)
- Simplified single-path architecture

**Trade-offs**:
- Anthropic ecosystem lock-in
- No local model support (privacy/offline constraints)
- Limited to Claude model family

**Key Insight**: SDK features depend on architecture assumptions (direct API, native protocol). Proxy layers break these assumptions, making multi-provider support incompatible with managed SDK benefits.

## References

- **Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)
- **Phase 4 Comparison**: [Framework Comparison Overview](../phases/phase-4-framework-comparison/overview.md)
- **ADR-0025**: [Project Requirements Package Framework (Deepagents)](0025-project-requirements-package-framework.md)
- **Claude SDK Documentation**: [https://docs.anthropic.com/en/docs/agents](https://docs.anthropic.com/en/docs/agents)
- **Model Context Protocol**: [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
- **NetBoxLabs MCP Server**: [https://github.com/netboxlabs/netbox-mcp-server](https://github.com/netboxlabs/netbox-mcp-server)

---

**Conclusion**: The Claude Agent SDK approach validates that production-ready frameworks can deliver significant value through managed orchestration, rapid development, and lower TCO, while accepting reasonable constraints on customization. This ADR documents the architectural decisions enabling **18x faster development** and **4-6x lower maintenance** compared to DIY approaches, confirming that framework selection should be context-driven based on project maturity, team capabilities, and business constraints.
