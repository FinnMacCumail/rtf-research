# Framework Comparison Overview

## Research Question

What are the trade-offs between flexible, open-source agent frameworks (deepagents/LangChain) versus opinionated, production-ready SDKs (Claude Agent SDK) when building LLM-powered infrastructure agents?

## Motivation

After the failure of Phase 3's multi-agent orchestration attempt (0% success rate), we needed a reliable approach to build intelligent NetBox infrastructure agents. Rather than commit to a single framework, we implemented the same system using two fundamentally different architectural approaches:

1. **Deepagents (LangChain)** - Flexible, DIY orchestration
2. **Claude Agent SDK** - Production-ready, managed solution

This comparative study provides empirical insights into framework selection for production LLM applications.

## Key Similarities

Both frameworks successfully solve the same problem and share core capabilities:

### Multi-Step Tool Orchestration
- Both support complex, multi-step workflows
- Both can plan actions, execute tools, and maintain context
- Both handle NetBox infrastructure queries effectively

### Context & State Management
- Built-in filesystem tools for persistent memory
- Session management across multiple interactions
- External state management (MCP servers, databases)

### Tool Integration & Customization
- Support for custom tool definitions
- Extensible architectures for domain-specific needs
- Integration with external APIs and services

### Production Focus
- Both designed for real-world, long-horizon tasks
- Support for complex domain applications (not just chat)
- Scalable beyond single-turn interactions

## Fundamental Differences

### Design Philosophy

| Aspect | Deepagents | Claude SDK |
|--------|-----------|------------|
| **Origin** | Open-source LangChain ecosystem | Official Anthropic SDK |
| **Philosophy** | "Harness" for building custom solutions | "Batteries included" managed platform |
| **Target** | Researchers, experimenters | Production deployments |

### Abstraction Level

**Deepagents**: Lower-level primitives
- Maximum flexibility and control
- Build orchestration logic from scratch
- Full responsibility for safety and permissions

**Claude SDK**: Higher-level components
- Opinionated best practices built-in
- Structured extension points
- Managed safety controls and permissions

### Deployment Model

**Deepagents**:
- Pure Python library
- Minimal external dependencies
- Self-contained in your codebase

**Claude SDK**:
- SDK + runtime requirements
- Model Context Protocol (MCP) integration
- Managed agent lifecycle

## Research Contributions

### 1. Empirical Framework Comparison
Real implementations of the same NetBox infrastructure agent using both frameworks, providing concrete data on:
- Development complexity
- Code maintainability
- Performance characteristics
- Safety and permission management

### 2. Decision Framework for Framework Selection
Clear guidance on when to choose each approach based on:
- Project maturity stage (prototype vs production)
- Team capabilities and resources
- Customization requirements
- Safety and compliance needs

### 3. Architectural Patterns Documentation
Documented patterns for:
- Tool orchestration strategies
- State management approaches
- Permission and safety controls
- Production deployment considerations

## Document Structure

This framework comparison section is organized as follows:

- **[Architectural Patterns](architectural-patterns.md)**: Core design patterns in each framework
- **[Flexibility vs Convenience](flexibility-vs-convenience.md)**: Trade-off analysis
- **[Tool Orchestration](tool-orchestration.md)**: How each framework handles tool coordination
- **[Production Considerations](production-considerations.md)**: Safety, permissions, deployment
- **[Selection Guide](selection-guide.md)**: Decision criteria for choosing a framework

For detailed implementation case studies, see [Phase 4 - Framework Comparison](../phases/phase-4-framework-comparison/overview.md).

## Key Findings Preview

### Both Frameworks Succeed
- 93.8% success rate for individual tools validated in both implementations
- Natural language NetBox queries work effectively
- Context management strategies apply to both frameworks

### Different Optimization Targets
- **Deepagents**: Optimizes for flexibility and innovation velocity
- **Claude SDK**: Optimizes for stability and deployment safety

### Complementary, Not Competing
- Can prototype in Deepagents, deploy in Claude SDK
- Can use Claude SDK for production, Deepagents for R&D
- Choice depends on project phase and requirements

## Next Steps

Continue to the following sections to understand the detailed comparison and make informed framework selection decisions:

1. Start with [Architectural Patterns](architectural-patterns.md) to understand core design differences
2. Review [Flexibility vs Convenience](flexibility-vs-convenience.md) for trade-off analysis
3. Check [Selection Guide](selection-guide.md) for decision criteria

For hands-on implementation details, see the [Phase 4 implementation case studies](../phases/phase-4-framework-comparison/overview.md).
