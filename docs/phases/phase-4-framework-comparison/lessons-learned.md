# Lessons Learned: Framework Selection for LLM Agents

## Key Lessons

### 1. No Universal "Best" Framework
Both Deepagents and Claude SDK successfully solve the same problem. Framework choice is context-dependent, not absolute.

### 2. Flexibility Has a Cost
Deepagents provides unlimited customization but requires 18x more initial development time and 4-6x more monthly maintenance.

### 3. Start Simple, Evolve as Needed
Recommendation: Start with Claude SDK for faster time-to-value, migrate to Deepagents only if SDK constraints become blocking.

### 4. Hybrid Strategies Work
Organizations can use both frameworks: Claude SDK for production stability, Deepagents for R&D experiments.

### 5. Context Engineering Transcends Frameworks
Token optimization strategies (Offload, Reduce, Retrieve, Isolate, Cache) apply to both frameworks equally.

### 6. Intelligent Routing Provides Transparent Cost Optimization (December 2025)
Discovery: Claude Agent SDK automatically routes between models (Haiku for tools, Sonnet/Opus for responses) achieving 70-80% cost reduction without user intervention.

**Key Insight**: When `model=None`, the SDK intelligently selects the cheapest capable model for each operation. This "invisible optimization" works even with explicit model selection—tool execution always uses Haiku regardless of user's response model choice.

**Implication**: SDKs can provide significant cost benefits through managed optimizations that would require custom implementation in flexible frameworks.

### 7. Framework Architecture Matters for Multi-Provider Support (December 2025)
Attempted integration of Ollama (local models) via LiteLLM proxy revealed critical incompatibilities with Claude SDK architecture.

**Problems Encountered**:
- Tool results not displayed in responses (Qwen 2.5:14b model)
- Loss of SDK features (MCP integration, streaming, caching, permissions)
- Architecture complexity not justified by benefits

**Key Lesson**: SDK features depend on architecture assumptions (direct API access, native protocol). Proxy layers break these assumptions. Multi-provider flexibility conflicts with managed SDK benefits.

**Implication**: For multi-provider requirements, flexible frameworks (Deepagents) are more suitable than managed SDKs. Framework lock-in is the trade-off for SDK optimizations.

### 8. Tool Result Integration Quality Varies Across Models (December 2025)
Not all LLMs integrate tool results equally well into responses.

**Observation**: Claude models naturally synthesize tool outputs into coherent responses. Qwen 2.5:14b (and likely other non-Anthropic models) executed tools successfully but didn't incorporate results into generated text.

**Implication**: When evaluating LLMs for agentic workflows, test tool result integration explicitly. Model capability specifications don't guarantee this behavior.

## Practical Recommendations

**For New Projects**: Start with Claude SDK
- 18x faster initial development
- Production-ready immediately
- Lower risk of architectural mistakes

**For Research Projects**: Consider Deepagents
- Novel agent behaviors
- Custom orchestration experiments
- Framework independence valued

**For Organizations**: Hybrid approach
- Default: Claude SDK for production systems
- R&D: Deepagents for experiments
- Bridge: Successful experiments � MCP tools � Production

## Selection Guide

For detailed framework selection criteria, see [Selection Guide](../../framework-comparison/selection-guide.md).

## Conclusion

Framework choice depends on:
- Project maturity (prototype vs production)
- Team capabilities (expertise, capacity)
- Business constraints (time, budget, compliance)
- Technical requirements (customization, performance)

Both frameworks are viablechoose based on your specific context, not universal recommendations.
