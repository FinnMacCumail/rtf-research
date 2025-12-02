# Architectural Patterns

## Overview

This document explores the core architectural patterns employed by deepagents and Claude Agent SDK, highlighting how each framework structures agent capabilities, manages state, and orchestrates tools.

## Deepagents Architecture

### Planning & Task Decomposition Pattern

**Philosophy**: Agents should plan before acting, breaking complex tasks into manageable subtasks.

**Implementation**:
```python
# Deepagents planning pattern
agent = await async_create_deep_agent(
    name="NetBox Infrastructure Analyst",
    tools=discover_netbox_tools(),
    planning_mode="hierarchical"  # Top-down task decomposition
)

# Agent internally maintains a to-do list
# Task: "Analyze production infrastructure"
# → Subtask 1: List all production sites
# → Subtask 2: Get devices per site
# → Subtask 3: Analyze connectivity patterns
# → Subtask 4: Generate summary report
```

**Key Characteristics**:
- Explicit planning phase before execution
- To-do list maintained in agent state
- Dynamic plan refinement based on intermediate results
- Provenance tracking for decisions

### Sub-Agent Delegation Pattern

**Philosophy**: Complex tasks benefit from specialized sub-agents working independently.

**Implementation**:
```python
# Main agent spawns specialized sub-agents
main_agent = create_coordinator_agent()

# Spawn sub-agent for specific domain
subnet_analyzer = await main_agent.spawn_subagent(
    name="Subnet Analyzer",
    tools=[analyze_ip_ranges, check_utilization],
    context="Analyze subnet 10.0.0.0/8 utilization"
)

# Sub-agent works in isolation
results = await subnet_analyzer.execute()

# Main agent aggregates results
await main_agent.integrate_subagent_results(results)
```

**Key Characteristics**:
- Context isolation between sub-agents
- Parallel execution capabilities
- Specialized toolsets per sub-agent
- Result aggregation at coordinator level

### Filesystem as Memory Pattern

**Philosophy**: Treat filesystem as unlimited, persistent memory for agent state.

**Implementation**:
```python
# Agents use filesystem for persistent state
await agent.write_file(
    path="/workspace/device_inventory.json",
    content=json.dumps(devices)
)

# Later retrieval across sessions
previous_data = await agent.read_file(
    path="/workspace/device_inventory.json"
)

# No context window constraints
# State persists between agent invocations
```

**Key Characteristics**:
- External state storage (not in LLM context)
- Unlimited memory capacity
- Structured data persistence
- Cross-session state continuity

### Custom Tool Integration Pattern

**Philosophy**: Maximum flexibility for domain-specific tool creation.

**Implementation**:
```python
# Custom tool definition with full control
@tool
def analyze_vlan_conflicts(site_id: str) -> dict:
    """Analyze VLAN ID conflicts across site devices."""
    # Custom orchestration logic
    devices = netbox_client.get_devices(site=site_id)
    vlans = [get_device_vlans(d) for d in devices]

    # Complex domain-specific analysis
    conflicts = detect_conflicts(vlans)

    return {
        "site": site_id,
        "conflicts": conflicts,
        "severity": calculate_severity(conflicts)
    }

# Register with agent
agent.add_tool(analyze_vlan_conflicts)
```

**Key Characteristics**:
- No constraints on tool complexity
- Direct control over tool behavior
- Custom error handling and retries
- Domain-specific optimizations

## Claude Agent SDK Architecture

### Model Context Protocol (MCP) Pattern

**Philosophy**: Standardized protocol for tool definition and discovery.

**Implementation**:
```python
# MCP server defines tools with schemas
mcp_servers = {
    "netbox": {
        "command": "python",
        "args": ["-m", "mcp_netbox"],
        "env": {"NETBOX_URL": url, "NETBOX_TOKEN": token}
    }
}

# SDK automatically discovers and wraps tools
agent = ChatAgent(config)
await agent.start_session()

# Tools available via MCP protocol
# SDK handles serialization, validation, permissions
response = await agent.query("List all sites")
```

**Key Characteristics**:
- Standardized tool interface
- Automatic tool discovery
- Schema-driven validation
- Managed lifecycle (connect/disconnect)

### Permission & Hook System Pattern

**Philosophy**: Safety and observability through declarative policies and lifecycle hooks.

**Implementation**:
```python
# Permission configuration
options = ClaudeAgentOptions(
    permission_mode="acceptEdits",  # Control tool execution
    allowed_tools=get_allowed_netbox_tools(),  # Whitelist
    hooks={
        "pre_tool_use": log_tool_invocation,
        "post_tool_use": validate_tool_results,
        "error": handle_tool_errors
    }
)

# Hooks provide observability
def log_tool_invocation(tool_name, args):
    logger.info(f"Executing: {tool_name} with {args}")
    # Can block execution here
    return True  # Allow

def validate_tool_results(tool_name, result):
    # Post-execution validation
    if not meets_criteria(result):
        raise ValidationError("Results failed validation")
    return result
```

**Key Characteristics**:
- Declarative permission model
- Pre/post execution hooks
- Audit trail generation
- Centralized policy enforcement

### Session Lifecycle Management Pattern

**Philosophy**: Managed agent sessions with proper resource cleanup.

**Implementation**:
```python
# Structured session lifecycle
class ChatAgent:
    async def start_session(self):
        """Initialize agent session"""
        self.client = ClaudeSDKClient(options=self.options)
        await self.client.__aenter__()  # Managed context
        self.session_active = True

    async def query(self, message: str):
        """Query within active session"""
        if not self.session_active:
            raise RuntimeError("Session not active")

        await self.client.query(message)
        async for response in self.client.receive_response():
            yield response

    async def close_session(self):
        """Cleanup resources"""
        await self.client.__aexit__(None, None, None)
        self.session_active = False
```

**Key Characteristics**:
- Explicit session boundaries
- Resource cleanup guarantees
- Error handling patterns
- Context manager integration

### Streaming Response Pattern

**Philosophy**: Real-time user feedback through incremental responses.

**Implementation**:
```python
# WebSocket-based streaming
async def query(self, message: str) -> AsyncIterator[StreamChunk]:
    await self.client.query(message)

    async for msg in self.client.receive_response():
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    # Stream text incrementally
                    yield StreamChunk(
                        type="text",
                        content=block.text,
                        completed=False
                    )
                elif isinstance(block, ToolUseBlock):
                    # Notify tool usage
                    yield StreamChunk(
                        type="tool_use",
                        content=f"Using tool: {block.name}",
                        completed=False
                    )
```

**Key Characteristics**:
- Incremental response delivery
- Type-safe message handling
- Real-time progress updates
- Low-latency user experience

## Pattern Comparison Matrix

| Pattern Category | Deepagents | Claude SDK |
|-----------------|------------|------------|
| **Task Planning** | Explicit to-do lists, hierarchical decomposition | Implicit planning within LLM reasoning |
| **Modularity** | Sub-agent spawning, context isolation | Single agent with tool composition |
| **State Management** | Filesystem as memory, unlimited state | Session-based state, managed lifecycle |
| **Tool Definition** | Custom Python functions, unlimited flexibility | MCP protocol, standardized interface |
| **Safety Controls** | Developer-implemented | Built-in permission model + hooks |
| **Observability** | Custom logging & monitoring | Hook-based lifecycle events |
| **Error Handling** | Custom retry logic | SDK-managed error recovery |
| **Resource Cleanup** | Developer responsibility | Managed context cleanup |

## When Each Pattern Excels

### Deepagents Patterns Excel When:

1. **Research & Experimentation**
   - Novel task decomposition strategies
   - Custom orchestration logic
   - Hypothesis testing with agent architectures

2. **Complex Domain Logic**
   - Multi-step workflows with branching
   - Domain-specific optimizations
   - Custom state management requirements

3. **Maximum Flexibility**
   - Unusual tool integration needs
   - Custom memory strategies
   - Experimental agent behaviors

### Claude SDK Patterns Excel When:

1. **Production Deployments**
   - Standardized tool interfaces
   - Audit trail requirements
   - Permission-based access control

2. **Team Collaboration**
   - Consistent patterns across codebase
   - Managed lifecycle reduces bugs
   - Clear extension points

3. **Rapid Development**
   - Batteries-included components
   - Less boilerplate code
   - Standard best practices

## Hybrid Approaches

Can you combine patterns from both frameworks?

**Yes, strategically**:

- **Prototype with Deepagents**, standardize with Claude SDK
- Use **Deepagents for R&D**, Claude SDK for production
- **Custom tools** in Deepagents, expose via MCP to Claude SDK
- **Sub-agent patterns** from Deepagents, permission model from Claude SDK

**Trade-offs**:
- Increased complexity maintaining two frameworks
- Need team expertise in both ecosystems
- Integration overhead between systems

**Best Practice**:
Choose one framework as primary, selectively adopt patterns from the other when compelling benefits exist.

## Conclusion

Both frameworks provide powerful architectural patterns for building LLM agents:

- **Deepagents**: Flexibility, control, experimentation
- **Claude SDK**: Standardization, safety, production-readiness

The choice depends on your project phase, team capabilities, and production requirements. See the [Selection Guide](selection-guide.md) for decision criteria.
