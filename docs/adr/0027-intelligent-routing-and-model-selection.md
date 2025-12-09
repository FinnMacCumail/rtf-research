# ADR-0027 — Intelligent Routing and Model Selection Strategy

## Status

**Accepted** — Production implementation complete (December 2025)
**Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)

## Context

Following Phase 4's successful Claude Agent SDK implementation, user feedback indicated desire for explicit model control while maintaining cost efficiency. Investigation revealed the SDK's intelligent multi-model routing feature—an undocumented behavior where the SDK automatically selects between Claude models (Haiku, Sonnet, Opus) based on task complexity, even when `model=None`.

### Discovery: Intelligent Routing Behavior

**Observation from API Logs** (December 2025):
When `ClaudeAgentOptions(model=None)` is specified, the SDK makes intelligent routing decisions:

| Query Type | SDK Selects | Reason |
|-----------|-------------|---------|
| Simple list operations | Claude Haiku 4.5 | Cost optimization ($0.25/1M tokens) |
| Tool execution | Claude Haiku 4.5 | Fast, cheap operations |
| Medium complexity | Claude Sonnet 4 | Balanced capability with caching |
| Complex analysis | Claude Sonnet 4.5 | Extended thinking, reasoning |

**Cost Impact**: 70-80% cost reduction vs using only Sonnet or Opus for all operations.

**Example**: Query "How many NetBox sites are there?"
```
API Calls Observed:
1. claude-haiku-4-5-20251001    - MCP tool execution (netbox_get_objects)
2. claude-haiku-4-5-20251001    - Additional tool calls
3. claude-sonnet-4-20241120     - Final response synthesis (with caching)

Cost: ~$0.004 vs $0.018 if all calls used Sonnet
Savings: 76%
```

### User Requirements

1. **Explicit Model Selection**: Users want predictable model usage for specific use cases
2. **Cost Transparency**: Visibility into which models are being used
3. **Preserve Optimization**: Don't lose intelligent routing when explicit selection not needed
4. **Multi-Provider Support** (attempted): Explore Ollama for local models

## Decision

Implement **Anthropic-only model selection** with support for both automatic intelligent routing and explicit model specification.

### Architecture

**Model Selection Options**:
1. **Auto (Recommended Default)**: `model=None` - SDK chooses optimal model per task
2. **Claude Haiku 4.5**: `model="claude-haiku-4-5-20250925"` - Explicit fast/cheap model
3. **Claude Sonnet 4.5**: `model="claude-sonnet-4-5-20250929"` - Explicit balanced model
4. **Claude Opus 4**: `model="claude-opus-4-20250514"` - Explicit maximum capability

**Backend Implementation**:
```python
class ChatAgent:
    def __init__(self, config: Config, model: str | None = None) -> None:
        """
        Initialize agent with optional model specification.

        Args:
            config: Application configuration
            model: Explicit model ID or None for automatic routing
        """
        self.model = model
        self.options = ClaudeAgentOptions(
            model=model,  # None = automatic, or explicit model ID
            mcp_servers=get_netbox_mcp_config(config),
            # ... other options
        )
```

**API Endpoint**:
```python
@app.get("/models")
async def get_models() -> list[ModelInfo]:
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

**Frontend Implementation**:
- `ModelSelector.vue` - Modal component for model selection
- `useModelSelection.ts` - Composable for state management
- WebSocket protocol: `{type: "model_change", model: "claude-sonnet-4-5-20250929"}`

## Alternative Considered: Ollama/LiteLLM Integration

### Attempted Approach

**Goal**: Enable local Ollama models (Qwen 2.5:14b-instruct-8k) alongside Anthropic models via LiteLLM proxy.

**Dual-Path Architecture Planned**:
```
Path A (Anthropic):
  Frontend → Backend → Claude Agent SDK → Anthropic API

Path B (Ollama):
  Frontend → Backend → LiteLLM Proxy → Ollama API
```

**Implementation Completed**:
- LiteLLM proxy configuration with Docker Compose
- Dual-path routing in backend
- Agent factory with model-based routing decisions
- Enhanced ModelSelector UI supporting both providers
- Health check monitoring for LiteLLM and Ollama services

### Problems Encountered

#### Problem 1: Tool Results Not Displayed (Critical)

**Symptom**: Qwen 2.5:14b model executes MCP tools successfully but doesn't display results in responses.

**Evidence**:
- Backend logs show successful MCP tool calls with data returned
- Qwen model generates acknowledgments ("I'll check that") but omits actual data
- Same query with Claude models displays full tool results correctly

**Example**:
```
Query: "How many NetBox sites are there?"

Qwen 2.5 Response: "I'll check the number of sites for you."
[No actual count provided despite tool returning count=24]

Claude Response: "There are 24 sites in your NetBox instance."
[Correctly synthesizes tool result into response]
```

**Root Cause**: Qwen 2.5:14b's instruction-following behavior differs from Claude's tool result integration patterns. The model doesn't naturally synthesize tool results into responses.

#### Problem 2: Loss of SDK Features

**When Routing Anthropic Through LiteLLM** (dual-path option considered):
- ❌ MCP protocol integration broken (subprocess isolation incompatible)
- ❌ Thinking blocks not streamed
- ❌ Prompt caching ineffective
- ❌ Permission system bypassed
- ❌ SDK's intelligent routing disabled

**Impact**: Anthropic models via LiteLLM would lose 70-80% cost optimization and critical safety features.

#### Problem 3: Architecture Complexity

**Dual-Path System Complexity**:
- Model registry with routing path decisions
- Separate agent implementations (ClaudeAgent, LiteLLMAgent)
- Health check orchestration for multiple services
- Session management across different agent types
- Telemetry tracking for routing decisions

**Maintenance Burden**:
- Docker Compose for LiteLLM proxy
- Ollama service management
- LiteLLM configuration updates
- Model compatibility testing
- Proxy authentication and security

#### Problem 4: Framework Incompatibility

**Claude Agent SDK Design Assumption**: Direct Anthropic API access with native protocol.

**LiteLLM Proxy Pattern**: OpenAI-compatible API adapter.

**Mismatch**: SDK features (MCP, streaming, hooks) depend on Anthropic-native protocol, which LiteLLM translates/adapts, breaking assumptions.

### Decision to Revert

**Factors**:
1. **Critical UX Failure**: Tool results not displayed (unacceptable for infrastructure queries)
2. **Feature Degradation**: Loss of SDK capabilities outweighs local model benefits
3. **Complexity Cost**: Dual-path architecture maintenance not justified by value
4. **Framework Mismatch**: LiteLLM proxy fundamentally incompatible with SDK architecture

**Reversion Actions**:
- Removed all LiteLLM-specific code (agent_factory.py, litellm_agent.py, litellm_config.py)
- Deleted docker-compose.litellm.yml and LiteLLM configuration files
- Simplified model selection to Anthropic-only
- Restored direct SDK integration exclusively
- Updated documentation to reflect Anthropic-only design

## Rationale: Anthropic-Only Model Selection

### Benefits

**1. Preserved Intelligent Routing**
- Auto mode defaults to `model=None`
- 70-80% cost savings through automatic Haiku usage for tools
- No user configuration required for optimization
- Transparent and automatic

**2. User Control When Needed**
- Explicit model selection for predictability
- Fixed model for entire session when selected
- Clear model identification in UI
- Session-scoped model binding

**3. SDK Feature Preservation**
- MCP integration fully functional
- Thinking blocks streamed
- Prompt caching active (84% hit rate observed)
- Permission system enforced
- Safety controls maintained

**4. Simplified Architecture**
- Single SDK integration path
- No proxy layer complexity
- Direct Anthropic API access
- Reduced maintenance burden
- Fewer failure modes

**5. Production Reliability**
- Proven SDK patterns
- Official Anthropic support
- Consistent behavior
- Predictable tool result integration

### Trade-offs

**Framework Lock-in**: Tightly coupled to Anthropic ecosystem

**Mitigation**:
- Abstractions in `agent.py` provide isolation layer
- Custom logic implemented as MCP tools (transferable)
- Clear separation of concerns for potential migration

**No Local Models**: Cannot use Ollama or other local LLMs

**Assessment**:
- Local model benefits (privacy, cost) not worth UX degradation
- Anthropic's pricing competitive for production use
- Intelligent routing makes costs acceptable
- Tool result reliability more valuable than local execution

## Implementation Details

### Model Switching Protocol

**Session Management**:
```python
# WebSocket protocol for model switching
{
  "type": "model_change",
  "model": "claude-sonnet-4-5-20250929"  # or "auto"
}

# Backend handling
async def switch_model(new_model_id: str):
    # Map "auto" to None
    model_param = None if new_model_id == "auto" else new_model_id

    # Close current session
    await agent.close_session()

    # Create new agent with new model
    agent = ChatAgent(config, model=model_param)
    await agent.start_session()

    # Context is reset - intentional
```

**Important**: Model switching always resets conversation context. This is intentional to prevent context confusion between different models.

### Frontend Model Selection

**UI Pattern**:
```vue
<ModelSelector :on-model-switch="handleModelSwitch" />
```

**Features**:
- Modal interface for model selection
- Current model indicator in header
- Warning about context reset when switching
- Persistence in localStorage
- Server-side validation of model availability

### Intelligent Routing Preservation

**Default Behavior** (Auto mode):
```python
# Frontend selects "auto"
agent = ChatAgent(config, model=None)

# SDK automatically routes:
# - Haiku for MCP tool calls (cheap)
# - Sonnet/Opus for responses (quality)
```

**Explicit Selection**:
```python
# User selects Sonnet 4.5
agent = ChatAgent(config, model="claude-sonnet-4-5-20250929")

# SDK uses:
# - Haiku for MCP tool calls (still optimized!)
# - Sonnet 4.5 for responses (user's choice)
```

**Key Insight**: Even with explicit model selection, SDK still uses intelligent routing for tool execution. The specified model controls response quality, not tool execution.

## Consequences

### Positive

**1. Cost Optimization Maintained**
- 70-80% cost reduction through intelligent routing
- Automatic Haiku usage for tools
- No user intervention required
- Transparent optimization

**2. User Empowerment**
- Choice between automatic and explicit models
- Predictable behavior when needed
- Clear model identification
- Informed decision-making

**3. Reliability**
- Tool results consistently displayed
- SDK features fully functional
- Proven architecture patterns
- Official framework support

**4. Simplified Operations**
- Single integration path
- Reduced configuration
- Lower maintenance burden
- Fewer failure modes

**5. Documentation Value**
- Discovered intelligent routing behavior
- Documented cost optimization patterns
- Shared learnings about SDK capabilities
- Valuable for framework comparison research

### Negative

**1. Anthropic Lock-in**
- Cannot easily migrate to other providers
- Dependent on Anthropic API availability
- Subject to Anthropic pricing changes

**Mitigation**: Acceptable trade-off for reliability and features. MCP tools are transferable.

**2. No Local Model Support**
- Cannot use privacy-sensitive data with local models
- Cannot reduce costs to zero with self-hosted LLMs
- Limited experimentation with alternative models

**Assessment**: Local model benefits don't justify UX and complexity costs for this use case.

**3. Limited Model Diversity**
- Anthropic-only model selection
- Cannot compare behaviors across providers
- Research constrained to single ecosystem

**Perspective**: Phase 4 already compares frameworks (Deepagents vs Claude SDK). Provider diversity is secondary research question.

## Performance Impact

### Development Time
- **Initial Model Selection**: 2 hours
- **Ollama/LiteLLM Attempt**: 6 hours (implementation + debugging)
- **Reversion to Anthropic-Only**: 1 hour
- **Total**: 9 hours for complete feature with learning

### Cost Optimization Evidence

**Query**: "List all NetBox sites"

**Without Intelligent Routing** (Sonnet-only):
```
API Calls: 6 × claude-sonnet-4-5
Tokens: ~6,000 × $3.00/1M = $0.018
```

**With Intelligent Routing** (Auto mode):
```
API Calls: 5 × claude-haiku-4-5 + 1 × claude-sonnet-4
Tokens: ~5,000 × $0.25/1M + ~1,000 × $3.00/1M = $0.00425
Savings: 76%
```

### Response Latency
- **Auto Mode**: 2-4s (Haiku for tools, Sonnet for response)
- **Explicit Sonnet**: 2-4s (Haiku for tools, Sonnet for response)
- **Explicit Haiku**: 1-3s (Haiku for everything)
- **Explicit Opus**: 3-6s (Haiku for tools, Opus for response)

**Key Finding**: Model selection controls response quality/latency but not tool execution (always Haiku).

## Integration with Phase 4 Research

This ADR complements Phase 4's framework comparison study:

**Original Research Question**: How do flexible frameworks (Deepagents) compare to managed SDKs (Claude SDK)?

**This Enhancement Addresses**: How to provide user control while preserving SDK optimizations?

**Key Insights**:
1. SDKs can optimize transparently (intelligent routing)
2. User control and automatic optimization are complementary
3. Framework-native patterns superior to proxy workarounds
4. Tool result integration quality varies significantly across models

**Research Contribution**: Documents that SDK features depend on architecture assumptions (direct API access, native protocol). Proxy layers break these assumptions.

## Recommendations

### When to Use Each Model Option

**Choose Auto (Default)**:
- General usage with cost consciousness
- Trust SDK's intelligent routing
- Optimal cost/performance balance
- 70-80% cost savings automatic

**Choose Explicit Haiku**:
- High-volume automated queries
- Simple list/count operations
- Maximum cost minimization
- Latency-sensitive applications

**Choose Explicit Sonnet**:
- Complex analysis requirements
- Predictable model for SLAs
- Balanced capability needs
- Default for important queries

**Choose Explicit Opus**:
- Maximum reasoning capability
- Strategic decision support
- Complex troubleshooting
- Critical analysis tasks

### Future Considerations

**If Multi-Provider Support Revisited**:
1. **Requirement**: Tool results must be reliably integrated into responses
2. **Validation**: Test with extensive tool-based workflows before production
3. **Architecture**: Separate code paths, don't compromise SDK features for Anthropic
4. **Documentation**: Clear feature comparison matrix (what's lost with each path)
5. **Monitoring**: Telemetry to track tool result integration success rates

**Alternative for Local Models**:
- Consider separate application for local model experimentation
- Use Deepagents framework for local model integration (more flexible)
- Don't compromise production Claude SDK application for research use cases

## References

- **Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)
- **Phase 4 Overview**: [Framework Comparison Study](../phases/phase-4-framework-comparison/overview.md)
- **ADR-0026**: [Claude SDK Project Requirements Package](0026-claude-sdk-project-requirements-package.md)
- **Model Selection Guide**: Repository `docs/MODEL_SELECTION.md` (comprehensive user guide)
- **Intelligent Routing Documentation**: Repository `examples/claude-agent-sdk-python.md`
- **PRP with Reversion Details**: Repository `PRPs/litellm-integration.md`

---

**Conclusion**: The Claude Agent SDK's intelligent routing feature provides transparent cost optimization (70-80% savings) without user intervention. Adding explicit model selection empowers users while preserving this optimization. The attempted Ollama/LiteLLM integration revealed critical incompatibilities (tool results not displayed, SDK features lost), validating the decision to maintain Anthropic-only architecture. This ADR documents both the successful model selection implementation and the valuable learnings from the unsuccessful multi-provider attempt.
