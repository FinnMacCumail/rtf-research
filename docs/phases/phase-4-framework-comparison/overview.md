# Phase 4: Agent Framework Comparison Study

## Research Overview

**Research Question**: What are the trade-offs between flexible, open-source agent frameworks (deepagents/LangChain) versus opinionated, production-ready SDKs (Claude Agent SDK) when building LLM-powered infrastructure agents?

**Methodology**: Implement the same NetBox infrastructure agent using both frameworks to empirically compare architectural approaches, development velocity, and production readiness.

## Motivation

After Phase 3's multi-agent orchestration failed completely (0% success rate), we needed a reliable approach to build intelligent NetBox infrastructure agents. Rather than commit to a single framework without understanding the trade-offs, we implemented production systems using two fundamentally different approaches:

1. **Deepagents (LangChain-based)** - Flexible, research-oriented framework
2. **Claude Agent SDK** - Production-ready, managed solution

This comparative study provides empirical evidence for framework selection decisions in production LLM applications.

## Research Hypothesis

**Hypothesis**: Different agent frameworks optimize for different dimensions of the flexibility-convenience spectrum, and the optimal choice depends on project maturity, team capabilities, and customization requirements rather than one framework being universally superior.

**Validation**: Both frameworks successfully solve the same problem, confirming that architectural choice is context-dependent rather than absolute.

## Implementation Approach

### Implementation A: Deepagents (LangChain)

**Repository**: [https://github.com/FinnMacCumail/deepagents](https://github.com/FinnMacCumail/deepagents)

**Framework Philosophy**:
- Open-source, community-driven
- Maximum flexibility and customization
- DIY orchestration and tool management
- Research and experimentation focus

**Key Features Implemented**:
- Planning & task decomposition
- Sub-agent delegation capabilities
- Filesystem as memory pattern
- Custom tool integration
- LangGraph-based orchestration
- Dynamic tool discovery

**Architecture Highlights**:
```python
# Deepagents implementation pattern
agent = await async_create_deep_agent(
    name="NetBox Infrastructure Analyst",
    tools=discover_netbox_tools(),  # Dynamic tool discovery
    enable_cache=True,
    cache_ttl=300,
    system_message=generate_netbox_system_message(),
    enable_conversation_cache=True
)

# Flexible, custom orchestration
response = await agent.execute(user_query)
```

**See**: [Deepagents Architecture Details](deepagents-architecture.md)

### Implementation B: Claude Agent SDK

**Repository**: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)

**Framework Philosophy**:
- Official Anthropic SDK
- Production-ready out-of-box
- Managed orchestration and safety
- Batteries-included approach

**Key Features Implemented**:
- Model Context Protocol (MCP) integration
- Built-in permission system
- WebSocket streaming responses
- Session lifecycle management
- Type-safe implementation (Pydantic models)
- Hook-based extensibility

**Architecture Highlights**:
```python
# Claude SDK implementation pattern
options = ClaudeAgentOptions(
    mcp_servers=get_netbox_mcp_config(config),
    allowed_tools=get_allowed_netbox_tools(),
    permission_mode="acceptEdits",
    system_prompt={"type": "preset", "preset": "claude_code"}
)

agent = ChatAgent(config)
await agent.start_session()

# Managed orchestration
async for chunk in agent.query(user_query):
    yield chunk
```

**See**: [Claude SDK Architecture Details](claude-sdk-architecture.md)

### Model Selection Enhancement (December 2025)

Following the successful Claude SDK implementation, user feedback indicated desire for explicit model control while maintaining cost efficiency. Investigation revealed an undocumented SDK feature: **intelligent multi-model routing**.

#### Discovery: Intelligent Routing

When `ClaudeAgentOptions(model=None)` is specified, the SDK automatically routes between Claude models based on task complexity:

| Query Type | SDK Selects | Optimization |
|-----------|-------------|--------------|
| MCP tool execution | Claude Haiku 4.5 | Cost ($0.25/1M tokens) |
| Simple responses | Claude Haiku 4.5 | Speed & cost |
| Medium complexity | Claude Sonnet 4 | Balanced with caching |
| Complex analysis | Claude Sonnet 4.5 | Extended thinking |

**Cost Impact**: 70-80% cost reduction vs using only Sonnet or Opus for all operations.

**Example Query**: "How many NetBox sites are there?"
```
API Calls:
1. claude-haiku-4-5 (tool: netbox_get_objects)
2. claude-haiku-4-5 (additional tool calls)
3. claude-sonnet-4 (response synthesis with caching)

Cost: ~$0.004 vs $0.018 if all Sonnet
Savings: 76%
```

#### Implementation

Enhanced the Claude SDK implementation with explicit model selection while preserving intelligent routing:

```python
# 4 model options available
class ChatAgent:
    def __init__(self, config: Config, model: str | None = None):
        """
        Args:
            model: "auto" (None), "claude-haiku-4-5",
                   "claude-sonnet-4-5", or "claude-opus-4"
        """
        self.options = ClaudeAgentOptions(
            model=model,  # None enables intelligent routing
            mcp_servers=get_netbox_mcp_config(config)
        )

# API endpoint for model discovery
@app.get("/models")
async def get_models() -> list[ModelInfo]:
    return [
        ModelInfo(id="auto", name="Claude (Automatic Selection)", ...),
        ModelInfo(id="claude-sonnet-4-5-20250929", ...),
        ModelInfo(id="claude-opus-4-20250514", ...),
        ModelInfo(id="claude-haiku-4-5-20250925", ...),
    ]

# WebSocket protocol for runtime model switching
{
    "type": "model_change",
    "model": "claude-sonnet-4-5-20250929"  # or "auto"
}
```

**Frontend**: `ModelSelector.vue` component provides modal interface with:
- Current model indicator in header
- Warning about context reset when switching
- Persistence in localStorage
- Server-side validation

**Key Insight**: Even with explicit model selection, SDK continues using intelligent routing for tool execution. The specified model controls response quality, not tool execution costs.

#### Attempted: Ollama/LiteLLM Integration (Reverted)

**Goal**: Enable local Ollama models (Qwen 2.5:14b) alongside Anthropic models via LiteLLM proxy.

**Dual-Path Architecture Planned**:
```
Path A (Anthropic): Frontend → Backend → Claude SDK → Anthropic API
Path B (Ollama):    Frontend → Backend → LiteLLM → Ollama API
```

**Problems Encountered**:

1. **Tool Results Not Displayed** (Critical):
   - Qwen 2.5 executed MCP tools successfully
   - Backend logs showed data returned correctly
   - Model generated acknowledgments but omitted actual data
   - Example: Query "How many sites?" → Response "I'll check that" [no count]
   - Claude models correctly integrated tool results into responses

2. **Loss of SDK Features**:
   - Routing Anthropic through LiteLLM would break:
     - MCP protocol integration (subprocess isolation incompatible)
     - Thinking block streaming
     - Prompt caching (84% hit rate lost)
     - Permission system
     - Intelligent routing (70-80% cost savings lost)

3. **Architecture Complexity**:
   - Dual-path system with separate agent implementations
   - Health check orchestration for multiple services
   - Docker Compose for LiteLLM proxy management
   - Model compatibility testing across providers

4. **Framework Incompatibility**:
   - Claude SDK assumes direct Anthropic API access
   - LiteLLM proxy adapts to OpenAI-compatible API
   - SDK features depend on native protocol, broken by proxy translation

**Decision**: Reverted to Anthropic-only model selection after 6 hours of implementation and debugging.

**Rationale**:
- Tool result reliability more valuable than local model support
- SDK feature preservation outweighs multi-provider flexibility
- Intelligent routing makes Anthropic costs acceptable
- Framework lock-in acceptable trade-off for reliability

**Documentation**: Full analysis in [ADR-0027](../../adr/0027-intelligent-routing-and-model-selection.md)

#### Model Selection Impact

**Benefits**:
- ✅ Preserved 70-80% cost optimization through intelligent routing
- ✅ User control when predictability needed
- ✅ All SDK features maintained (MCP, streaming, caching, permissions)
- ✅ Simplified architecture (single integration path)
- ✅ Discovered and documented undocumented SDK capability

**Trade-offs**:
- ⚠️ Anthropic ecosystem lock-in
- ⚠️ No local model support (privacy/offline use cases)
- ⚠️ Limited to Claude model family

**Recommendations**:
- **Default**: Use "Auto" mode for 70-80% cost savings
- **Explicit Haiku**: High-volume automated queries
- **Explicit Sonnet**: Complex analysis, predictable SLAs
- **Explicit Opus**: Maximum reasoning capability

**Research Contribution**: Documents that SDK features depend on architecture assumptions (direct API access, native protocol). Proxy layers break these assumptions, making multi-provider support incompatible with managed SDK benefits.

## Comparative Analysis Framework

### Evaluation Dimensions

We evaluated both implementations across multiple dimensions:

1. **Development Velocity**: Time from zero to working agent
2. **Code Complexity**: Lines of code, maintainability
3. **Flexibility**: Ability to customize behavior
4. **Production Readiness**: Safety, reliability, monitoring
5. **Maintenance Burden**: Ongoing effort required
6. **Performance**: Response times, resource usage
7. **Tool Integration**: Ease of adding new capabilities

### Key Findings Summary

#### Both Frameworks Succeed
- ✅ Natural language NetBox queries work effectively
- ✅ Multi-step tool coordination successful
- ✅ Context management strategies apply to both
- ✅ Production deployment viable for both

#### Different Optimization Targets

**Deepagents Optimizes For**:
- Flexibility and control
- Experimentation velocity
- Custom orchestration patterns
- Framework independence

**Claude SDK Optimizes For**:
- Development speed
- Production reliability
- Security and compliance
- Maintenance minimization

#### Complementary, Not Competing

The frameworks are not mutually exclusive:
- Can prototype with Deepagents, deploy with Claude SDK
- Can use Claude SDK for production, Deepagents for R&D
- Can implement custom tools in Deepagents, expose via MCP
- Choice depends on project phase and requirements

## Research Contributions

### 1. Empirical Framework Comparison

Real production implementations provide concrete data on:
- Actual development time differences (46 hours vs 2.5 hours initial development)
- Code complexity metrics (1,200 LOC vs 400 LOC)
- Maintenance burden (8-12 hours/month vs 2-4 hours/month)
- Feature parity despite different architectures

### 2. Decision Framework for Practitioners

Clear guidance on when to choose each approach based on:
- Project maturity (prototype vs MVP vs production)
- Team capabilities (expertise level, size, maintenance capacity)
- Business constraints (time-to-market, budget, compliance)
- Technical requirements (customization needs, performance, scale)

### 3. Architectural Pattern Documentation

Documented patterns applicable beyond NetBox domain:
- Planning & task decomposition (Deepagents)
- Model Context Protocol integration (Claude SDK)
- Custom vs managed orchestration trade-offs
- Permission and safety control strategies
- Tool coordination approaches

### 4. Context Engineering Insights

Both implementations benefit from shared context engineering strategies:
- **Offload**: MCP servers as external state
- **Reduce**: Field filtering (Phase 2's 3 generic tools reduce schema overhead by 90%)
- **Retrieve**: On-demand data fetching with specific field selection
- **Isolate**: Session-based boundaries
- **Cache**: Prompt caching (84% hit rate)

**Note**: Phase 2 NetBoxLabs MCP server already provides 3 generic tools; context optimization happens through field filtering, not tool count reduction. Both Phase 4 frameworks build upon the same 3-tool foundation.

While context engineering is important, the framework architecture choice is the primary research focus.

## Key Insights

### Insight 1: No Universal "Best" Framework

**Finding**: Both frameworks successfully solve the same problem with different trade-offs.

**Implication**: Framework selection should be context-driven, not based on universal recommendations.

**Evidence**: Our NetBox implementations achieve feature parity despite radically different architectures.

### Insight 2: Flexibility-Convenience Trade-off is Real

**Finding**: Deepagents requires 18x more initial development time (46h vs 2.5h) but provides unlimited customization.

**Implication**: Teams must explicitly value flexibility to justify the development cost.

**Evidence**: Claude SDK reached production-ready state in 1 week; Deepagents took 4 weeks including custom infrastructure.

### Insight 3: Maintenance is the Long-Term Cost

**Finding**: Ongoing maintenance differentiates frameworks more than initial development.

**Implication**: Total Cost of Ownership (TCO) heavily influenced by maintenance burden.

**Evidence**: Deepagents requires 4-6x more monthly maintenance (8-12h vs 2-4h) due to custom infrastructure.

### Insight 4: Hybrid Strategies Are Viable

**Finding**: Frameworks can coexist in the same organization.

**Implication**: Can optimize for different use cases with different frameworks.

**Evidence**: Our R&D uses Deepagents for experiments; production deployments could use Claude SDK for stability.

### Insight 5: Context Engineering Transcends Frameworks

**Finding**: Token optimization strategies apply to both frameworks.

**Implication**: Context engineering is orthogonal to framework choice.

**Evidence**: Both implementations achieve 70% cost reduction through prompt caching, 99.1% prompt token problem affects both.

## Document Structure

This Phase 4 section includes:

- **[Overview](overview.md)** (this document) - Research question and findings summary
- **[Deepagents Architecture](deepagents-architecture.md)** - Implementation A detailed analysis
- **[Claude SDK Architecture](claude-sdk-architecture.md)** - Implementation B detailed analysis
- **[Comparative Analysis](comparative-analysis.md)** - Side-by-side evaluation
- **[Lessons Learned](lessons-learned.md)** - Insights and recommendations

For framework-specific guidance, see the [Framework Comparison](../../framework-comparison/overview.md) section.

## Practical Recommendations

### For New Projects

**Recommendation**: Start with Claude SDK

**Reasoning**:
- 18x faster initial development
- Production-ready immediately
- Lower risk of architectural mistakes
- Can always migrate to Deepagents later if needed

**Exception**: Choose Deepagents if novel agent behaviors are core differentiator

### For Existing Projects

**If currently using Claude SDK**:
- Remain on SDK unless hitting specific limitations
- Implement custom logic as MCP tools first
- Only migrate to Deepagents if SDK constraints become blocking

**If currently using Deepagents**:
- Continue if customization provides value
- Consider extracting stable patterns as MCP tools
- Evaluate migration to Claude SDK for mature, stable workflows

### For Organizations

**Hybrid Strategy Recommended**:
1. **Default**: Claude SDK for production systems
2. **R&D**: Deepagents for experiments and novel approaches
3. **Bridge**: Successful experiments → MCP tools → Production

## Success Metrics

### Phase 4 Objectives - ✅ All Achieved

- ✅ Implement working NetBox agent with Deepagents
- ✅ Implement working NetBox agent with Claude SDK
- ✅ Document architectural patterns in both frameworks
- ✅ Compare development velocity and code complexity
- ✅ Evaluate production readiness and maintenance burden
- ✅ Provide clear framework selection guidance
- ✅ Validate that both approaches succeed

### Research Impact

**Academic Contribution**:
- Empirical comparison of agent frameworks
- Context-dependent framework selection criteria
- Replicable evaluation methodology

**Practical Impact**:
- Decision framework for practitioners
- Reduced trial-and-error in framework selection
- Clear cost-benefit analysis

## Timeline

- **Week 1-2**: Deepagents implementation
- **Week 3**: Claude SDK implementation
- **Week 4**: Comparative evaluation and documentation
- **Week 5**: Framework comparison guide creation

## Next Steps

### Immediate

1. Review detailed architecture documentation:
   - [Deepagents Architecture](deepagents-architecture.md)
   - [Claude SDK Architecture](claude-sdk-architecture.md)

2. Understand trade-offs:
   - [Comparative Analysis](comparative-analysis.md)
   - [Framework Comparison Guide](../../framework-comparison/overview.md)

3. Make informed decision:
   - [Selection Guide](../../framework-comparison/selection-guide.md)

### Future Research (Phase 5-7)

- **Phase 5**: Neo4j graph database integration for instant complex queries
- **Phase 6**: RAG-powered semantic intelligence with institutional memory
- **Phase 7**: Advanced analytics platform using graph algorithms

Phase 4's framework foundation enables these advanced capabilities with either Deepagents or Claude SDK as the orchestration layer.

## Conclusion

Phase 4 validates that framework choice is context-dependent rather than absolute. Both deepagents and Claude SDK successfully build production LLM agents, but optimize for different dimensions of the flexibility-convenience spectrum.

**Key Takeaway**: Choose your framework based on project maturity, team capabilities, and business constraints—not on universal "best practices."

For most projects, **start with Claude SDK** for faster time-to-value, and migrate to Deepagents only if SDK constraints become blocking.

---

**Repository Links**:
- Deepagents Implementation: [https://github.com/FinnMacCumail/deepagents](https://github.com/FinnMacCumail/deepagents)
- Claude SDK Implementation: [https://github.com/FinnMacCumail/claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox)

**Complete Documentation**: See [Framework Comparison](../../framework-comparison/overview.md) for comprehensive framework selection guidance.
