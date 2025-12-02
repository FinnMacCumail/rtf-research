# Tool Orchestration

## Overview

Tool orchestration—how agents discover, invoke, and coordinate multiple tools—is a critical architectural decision that fundamentally differs between deepagents and Claude SDK.

## Tool Orchestration Philosophy

### Deepagents: Custom Orchestration
**Philosophy**: "Build exactly the orchestration logic your domain needs"

**Key Principle**: Maximum flexibility to implement domain-specific coordination patterns

### Claude SDK: Protocol-Based Orchestration
**Philosophy**: "Standardized tool protocol enables managed orchestration"

**Key Principle**: Consistent, safe tool invocation through Model Context Protocol (MCP)

## Tool Definition & Discovery

### Deepagents Approach

#### Dynamic Tool Creation
```python
# Define tools as Python functions
@tool
def get_site_devices(site_name: str) -> List[dict]:
    """Get all devices at a specific site."""
    return netbox_client.dcim.devices.filter(site=site_name)

@tool
def analyze_vlan_usage(site_name: str) -> dict:
    """Analyze VLAN utilization across site."""
    devices = get_site_devices(site_name)
    vlans = [extract_vlans(d) for d in devices]
    return calculate_utilization(vlans)

# Tool composition
@tool
def site_health_check(site_name: str) -> dict:
    """Complete site health analysis."""
    devices = get_site_devices(site_name)
    vlans = analyze_vlan_usage(site_name)

    # Custom orchestration logic
    return {
        "device_count": len(devices),
        "vlan_utilization": vlans,
        "health_score": calculate_health(devices, vlans)
    }
```

**Characteristics**:
- Unlimited tool complexity
- Direct Python function calls
- Custom composition patterns
- No protocol constraints

#### Tool Registration
```python
# Register tools with agent
agent = await async_create_deep_agent(
    name="NetBox Analyst",
    tools=[
        get_site_devices,
        analyze_vlan_usage,
        site_health_check,
        # Can add 100+ custom tools
    ]
)

# Or dynamic discovery
tools = discover_domain_tools(module="netbox_tools")
agent = await async_create_deep_agent(name="NetBox Analyst", tools=tools)
```

**Benefits**:
- Add tools without external processes
- Tool composition at Python level
- Instant tool availability
- Full control over tool lifecycle

### Claude SDK Approach

#### Model Context Protocol (MCP)
```python
# Tools defined in separate MCP server process
# mcp_netbox/server.py
@mcp.tool()
async def netbox_get_objects(
    object_type: str,
    filters: Optional[dict] = None,
    fields: Optional[List[str]] = None,
    limit: int = 50
) -> dict:
    """Generic tool for querying NetBox objects."""
    # Tool schema automatically generated
    # Input validation handled by MCP
    # Output serialization managed by protocol

    return await netbox_client.get_objects(
        object_type=object_type,
        filters=filters or {},
        fields=fields,
        limit=limit
    )
```

#### Tool Discovery via MCP
```python
# SDK automatically discovers MCP tools
mcp_servers = {
    "netbox": {
        "command": "python",
        "args": ["-m", "mcp_netbox"],
        "env": {"NETBOX_URL": url, "NETBOX_TOKEN": token}
    }
}

options = ClaudeAgentOptions(
    mcp_servers=mcp_servers,
    allowed_tools=[
        "netbox_get_objects",
        "netbox_get_object_by_id",
        "netbox_search_objects"
    ]
)

# Tools discovered at runtime via MCP protocol
agent = ChatAgent(options)
await agent.start_session()  # Tool discovery happens here
```

**Characteristics**:
- Standardized tool interface
- Separate process for tools
- Schema-driven validation
- Managed lifecycle

## Tool Coordination Patterns

### Deepagents: Explicit Coordination

#### Sequential Coordination
```python
# Explicit multi-step orchestration
class NetBoxWorkflow:
    async def analyze_infrastructure(self, site: str):
        # Step 1: Get foundational data
        devices = await self.get_site_devices(site)

        # Step 2: Analyze based on step 1 results
        vlans = await self.analyze_vlans(devices)

        # Step 3: Cross-reference data
        circuits = await self.get_site_circuits(site)

        # Step 4: Correlation logic
        return self.correlate_infrastructure(devices, vlans, circuits)
```

#### Parallel Coordination
```python
# Custom parallel execution
class ParallelNetBoxAgent:
    async def multi_site_analysis(self, sites: List[str]):
        # Launch parallel tool calls
        tasks = [
            self.analyze_infrastructure(site)
            for site in sites
        ]

        # Gather results
        results = await asyncio.gather(*tasks)

        # Custom aggregation
        return self.aggregate_site_analyses(results)
```

#### Conditional Coordination
```python
# Dynamic tool selection based on conditions
class AdaptiveNetBoxAgent:
    async def smart_query(self, query: str):
        # Analyze query intent
        intent = await self.classify_intent(query)

        if intent == "single_object":
            return await self.get_object_by_id(extract_id(query))
        elif intent == "filtered_list":
            filters = await self.extract_filters(query)
            return await self.get_objects_filtered(filters)
        elif intent == "complex_analysis":
            # Multi-step workflow
            return await self.complex_analysis_workflow(query)
```

**Benefits**:
- Complete control over execution order
- Custom error handling per step
- Domain-specific optimization
- Transparent orchestration logic

### Claude SDK: LLM-Driven Coordination

#### Autonomous Tool Selection
```python
# LLM decides which tools to call and when
async def query(self, message: str) -> AsyncIterator[StreamChunk]:
    """
    SDK + LLM handle orchestration automatically.

    User: "Show all devices at site DM-Scranton with their IP addresses"

    LLM reasoning (internal):
    1. Need to query devices with site filter
    2. Tool: netbox_get_objects(object_type="devices", filters={"site": "DM-Scranton"})
    3. Results include device IDs
    4. Need IP addresses? Already in results
    5. Format and return
    """
    await self.client.query(message)

    async for response in self.client.receive_response():
        # SDK streams tool usage and results
        yield response
```

**Characteristics**:
- LLM chooses tools dynamically
- Automatic multi-step planning
- Error recovery by LLM
- No explicit orchestration code

#### Hook-Based Observation
```python
# Observe orchestration via hooks
options = ClaudeAgentOptions(
    hooks={
        "pre_tool_use": lambda tool, args: log_tool_call(tool, args),
        "post_tool_use": lambda tool, result: validate_result(tool, result)
    }
)

# Orchestration visible through hooks, but not controllable
```

**Benefits**:
- No orchestration code to maintain
- LLM adapts to novel queries
- Handles unexpected scenarios
- Simpler codebase

**Trade-offs**:
- Less control over execution order
- Can't optimize specific patterns
- LLM may make suboptimal choices
- Harder to guarantee behavior

## Tool Performance Optimization

### Deepagents: Custom Optimization

#### Batching Pattern
```python
# Custom batching for efficiency
class BatchedNetBoxAgent:
    async def get_multiple_objects(self, object_ids: List[int]):
        # Custom batching logic
        batch_size = 50

        batches = [
            object_ids[i:i + batch_size]
            for i in range(0, len(object_ids), batch_size)
        ]

        # Parallel batch requests
        results = []
        for batch in batches:
            batch_results = await self.netbox_batch_get(batch)
            results.extend(batch_results)

        return results
```

#### Caching Pattern
```python
# Custom caching strategy
class CachedNetBoxAgent:
    def __init__(self):
        self.cache = TTLCache(maxsize=1000, ttl=300)  # 5-minute TTL

    async def get_site_devices(self, site: str):
        cache_key = f"devices:{site}"

        if cache_key in self.cache:
            logger.info(f"Cache hit: {cache_key}")
            return self.cache[cache_key]

        # Cache miss
        devices = await self.netbox_client.get_devices(site=site)
        self.cache[cache_key] = devices
        return devices
```

**Benefits**:
- Optimize for specific access patterns
- Fine-grained cache control
- Custom invalidation strategies
- Domain-specific batching

### Claude SDK: MCP-Level Optimization

#### Server-Side Caching
```python
# Caching in MCP server (transparent to agent)
class NetBoxMCPServer:
    def __init__(self):
        self.cache = TTLCache(maxsize=1000, ttl=60)

    @mcp.tool()
    async def netbox_get_objects(self, object_type: str, filters: dict):
        cache_key = self.make_cache_key(object_type, filters)

        if cache_key in self.cache:
            return self.cache[cache_key]

        result = await self.fetch_from_netbox(object_type, filters)
        self.cache[cache_key] = result
        return result
```

**Characteristics**:
- Caching transparent to agent
- Server manages cache lifecycle
- Agent code remains simple
- Consistent across all agents using server

#### Prompt Caching
```python
# Anthropic's prompt caching (automatic)
options = ClaudeAgentOptions(
    system_prompt={...},  # Automatically cached
    mcp_servers={...}  # Tool schemas cached
)

# 70% cost reduction on repeated system prompts
# No code changes required
```

**Benefits**:
- Automatic optimization
- No agent-level complexity
- Provider-managed efficiency
- Cost reduction without code changes

## Tool Error Handling

### Deepagents: Custom Error Recovery

```python
# Complete control over error handling
class RobustNetBoxAgent:
    async def resilient_query(self, object_type: str, filters: dict):
        max_retries = 3

        for attempt in range(max_retries):
            try:
                return await self.netbox_get_objects(object_type, filters)

            except RateLimitError:
                # Custom backoff strategy
                wait_time = 2 ** attempt
                await asyncio.sleep(wait_time)

            except NetworkError as e:
                if attempt == max_retries - 1:
                    # Final attempt: try fallback
                    return await self.fallback_query(object_type, filters)
                else:
                    logger.warning(f"Network error, retrying: {e}")

            except ValidationError:
                # Fix validation issues programmatically
                filters = self.sanitize_filters(filters)
                return await self.netbox_get_objects(object_type, filters)

        raise MaxRetriesExceeded()
```

### Claude SDK: Hook-Based Error Handling

```python
# Error handling via hooks
def handle_tool_error(tool_name: str, error: Exception) -> dict:
    """Custom error handling logic."""
    if isinstance(error, RateLimitError):
        # Return error context to LLM
        return {
            "error": "rate_limit",
            "message": "NetBox API rate limit exceeded",
            "suggestion": "Reduce query scope or wait 60 seconds"
        }

    elif isinstance(error, NotFoundError):
        return {
            "error": "not_found",
            "message": f"Object not found in NetBox",
            "suggestion": "Check object type and ID"
        }

    # Default handling
    raise error

options = ClaudeAgentOptions(
    hooks={"error": handle_tool_error}
)
```

**Characteristics**:
- LLM receives error context
- LLM decides recovery strategy
- Less deterministic recovery
- Simpler error handling code

## Comparison Matrix

| Aspect | Deepagents | Claude SDK |
|--------|-----------|------------|
| **Tool Definition** | Python functions | MCP protocol |
| **Tool Discovery** | Python imports | Runtime MCP discovery |
| **Orchestration Control** | Explicit code | LLM-driven |
| **Execution Order** | Deterministic | LLM-determined |
| **Parallel Execution** | Custom async code | LLM may parallelize |
| **Caching** | Custom implementation | MCP server + prompt cache |
| **Error Handling** | Custom retry logic | Hook-based + LLM recovery |
| **Performance Tuning** | Fine-grained control | Server-level optimization |
| **Debugging** | Standard Python debugging | Hook-based observation |

## Recommendations

**Choose Deepagents orchestration when**:
- Need deterministic tool execution order
- Require custom coordination patterns
- Performance optimization critical
- Domain-specific batching needed

**Choose Claude SDK orchestration when**:
- Standard tool coordination sufficient
- Prefer LLM adaptability
- Want minimal orchestration code
- Value managed optimization

## Conclusion

Tool orchestration philosophies differ fundamentally:

- **Deepagents**: Explicit, controlled, optimized
- **Claude SDK**: Implicit, adaptive, managed

Both approaches successfully handle complex multi-tool workflows—the choice depends on your need for control vs convenience.

See [Selection Guide](selection-guide.md) for choosing the right approach for your use case.
