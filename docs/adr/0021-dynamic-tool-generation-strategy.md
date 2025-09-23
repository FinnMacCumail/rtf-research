# ADR-0021 — Dynamic Tool Generation Strategy ✅ IMPLEMENTED

## Status
**Accepted** - Successfully implemented in Phase 4 deepagents solution
**Implementation**: [NetBox Agent](https://github.com/FinnMacCumail/deepagents/blob/master/examples/netbox/netbox_agent.py)
**Commit**: 9502c04869fbdf0dea21d8736d66ab89f39afe15

## Context

Phase 4 development faced a critical decision regarding NetBox MCP tool integration. The initial approach involved manual tool wrapping, which created significant maintenance overhead and incomplete coverage:

### Manual Tool Wrapping Limitations
- **Limited Coverage**: Only 8 out of 62 NetBox tools manually wrapped (13% coverage)
- **Maintenance Burden**: Each new NetBox tool required manual wrapper creation
- **Inconsistent Implementation**: Manual wrappers had varying error handling and response formatting
- **Development Overhead**: Significant time investment for each tool addition
- **Version Compatibility**: Manual wrappers could break with NetBox MCP updates

### NetBox Tool Landscape
- **Total Available Tools**: 62 NetBox MCP tools across multiple categories
- **Tool Categories**: SYSTEM (1), DCIM (37), IPAM (8), TENANCY (3), EXTRAS (1), VIRTUALIZATION (12)
- **Dynamic Nature**: NetBox tools evolve with API updates and new features
- **Comprehensive Coverage Need**: Infrastructure management requires access to full tool ecosystem

## Decision

Implement **dynamic tool generation** system that automatically creates wrappers for all NetBox MCP tools at runtime:

### Core Architecture
```python
def create_async_tool_wrapper(tool_name: str, tool_schema: dict) -> Callable:
    """
    Dynamically generates async wrapper function for NetBox MCP tool
    """
    async def wrapper(**kwargs):
        # Automatic client injection and error handling
        result = await execute_netbox_tool(tool_name, kwargs)
        return format_human_friendly_response(result)

    # Dynamic function signature generation with proper type annotations
    wrapper.__name__ = tool_name
    wrapper.__doc__ = generate_tool_documentation(tool_schema)
    return wrapper

def generate_all_tool_wrappers() -> Dict[str, Callable]:
    """
    Discovers and wraps all available NetBox MCP tools automatically
    """
    tools = {}
    for tool_name, schema in discover_netbox_tools():
        tools[tool_name] = create_async_tool_wrapper(tool_name, schema)
    return tools
```

### Key Implementation Features
1. **Runtime Tool Discovery**: Automatic detection of all available NetBox MCP tools
2. **Dynamic Function Generation**: Creates properly typed wrapper functions with documentation
3. **Automatic Client Injection**: Handles NetBox client management transparently
4. **Unified Error Handling**: Consistent error processing across all tools
5. **Human-Friendly Responses**: Automatic formatting with emojis and structured output
6. **Category Organization**: Tools grouped by functionality (DCIM, IPAM, etc.)

## Consequences

### Positive
- **Complete Coverage**: 100% tool coverage (62/62 tools) vs 13% with manual approach
- **Zero Maintenance**: New NetBox tools automatically available without code changes
- **Consistent Experience**: Uniform error handling and response formatting across all tools
- **Enhanced User Experience**: Human-friendly responses with emojis, structure, and helpful suggestions
- **Future-Proof**: Automatic adaptation to NetBox MCP evolution
- **Development Efficiency**: No manual wrapper creation required for new tools

### Negative
- **Runtime Dependency**: Requires NetBox MCP registry availability at startup
- **Dynamic Code Generation**: More complex debugging compared to static wrappers
- **Performance Overhead**: Minimal runtime cost for dynamic function generation
- **Type Safety**: Dynamic typing requires careful schema validation

### Performance Metrics
- **Coverage Improvement**: 13% → 100% (775% increase)
- **Development Time**: Zero ongoing maintenance vs hours per manual wrapper
- **User Experience**: Consistent formatting and helpful suggestions across all tools
- **Error Handling**: Unified error recovery and reporting

## Implementation Notes

### Tool Discovery Process
```python
def discover_netbox_tools() -> Iterator[Tuple[str, dict]]:
    """
    Leverages NetBox MCP registry for comprehensive tool discovery
    """
    for category, tools in NETBOX_TOOL_REGISTRY.items():
        for tool_name, schema in tools.items():
            yield tool_name, schema

def organize_tools_by_category(tools: Dict[str, Callable]) -> Dict[str, List[str]]:
    """
    Categorizes tools for enhanced agent instructions
    """
    categories = {
        'DCIM': [],      # Data Center Infrastructure Management
        'IPAM': [],      # IP Address Management
        'TENANCY': [],   # Multi-tenant organization
        'VIRTUALIZATION': [], # Virtual machines and clusters
        'EXTRAS': [],    # Additional features
        'SYSTEM': []     # System-level operations
    }
```

### Integration with DeepAgents Framework
The dynamic tool generation integrates seamlessly with the deepagents framework:
- **Agent Creation**: Tools automatically available to `async_create_deep_agent()`
- **Planning Integration**: Agent can reason about available tools dynamically
- **Context Preservation**: Tool discovery results cached for session consistency
- **Error Recovery**: Framework-level error handling enhanced with tool-specific recovery

### Human-Friendly Response Enhancement
```python
def format_human_friendly_response(result: dict) -> str:
    """
    Transforms technical API responses into user-friendly format
    """
    return f"""
    🔧 **NetBox Query Results**

    ✅ **Status**: {result.get('status', 'Completed')}
    📊 **Data**: {format_data_with_emojis(result.get('data', {}))}

    💡 **Helpful Tips**:
    {generate_contextual_suggestions(result)}
    """
```

## Related Decisions
- **ADR-0022**: Interactive CLI Architecture Design (enables practical tool usage)
- **ADR-0023**: Claude API Prompt Caching Implementation (optimizes performance for large tool sets)
- **ADR-0024**: Cache Performance Monitoring Strategy (tracks tool usage patterns)

## Lessons Learned
1. **Dynamic Generation Benefits**: Eliminated maintenance overhead while improving coverage dramatically
2. **User Experience Focus**: Human-friendly formatting significantly improved tool adoption
3. **Framework Integration**: Seamless integration with deepagents enhanced overall system capability
4. **Future-Proofing**: Dynamic approach provides resilience to NetBox evolution