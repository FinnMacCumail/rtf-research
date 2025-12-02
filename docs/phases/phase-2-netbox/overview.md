# Phase 2 – NetBox MCP Server (Shared Infrastructure)

## Overview

Phase 2 establishes the foundational infrastructure layer for AI agents to interact with NetBox through the official NetBoxLabs MCP server. This read-only interface provides 3 generic tools that both Phase 4 agent frameworks (Deepagents and Claude SDK) build upon.

**Repository**: [https://github.com/netboxlabs/netbox-mcp-server](https://github.com/netboxlabs/netbox-mcp-server)

## Goals

- ✅ Integrate official NetBoxLabs MCP server for NetBox data access
- ✅ Implement field filtering to minimize token usage
- ✅ Handle pagination/limits to avoid token overflows
- ✅ Provide read-only, safe interface for agent exploration
- ✅ Establish shared infrastructure foundation for Phase 4 frameworks

## Architecture

### Generic Tool Design

The NetBoxLabs MCP server provides **3 generic tools** instead of specialized tools per object type:

1. **`get_objects`** - Generic retrieval with filters, pagination, and field selection
2. **`get_object_by_id`** - Detailed single object lookup with field filtering
3. **`get_changelogs`** - Change history tracking for audit trails

### Key Features

- **Field Filtering**: Specify exact fields needed (90% token reduction typical)
- **Read-Only Safety**: No write/update/delete operations
- **Generic Pattern**: One tool handles all NetBox object types
- **Shared Infrastructure**: Both Phase 4 frameworks use the same 3 tools

## Status

✅ **Complete**: NetBoxLabs MCP server integration

- Official NetBoxLabs MCP server deployed
- Field filtering capabilities validated
- Token optimization strategies documented
- Shared infrastructure foundation established
- Phase 4 agent frameworks successfully building upon this foundation

## Integration with Phase 4

Both Phase 4 implementations use this shared MCP infrastructure:

- **Deepagents (Phase 4A)**: Wraps MCP tools with LangGraph orchestration
- **Claude SDK (Phase 4B)**: Uses MCP protocol directly with managed orchestration

**Key Insight**: The framework choice (Deepagents vs Claude SDK) is independent of the MCP infrastructure layer.

## Demo Scenarios

The following demo queries work with the 3 generic tools:

- **"List VLANs"** - Uses `get_objects(object_type="ipam.vlans")` with pagination
- **"Show interfaces for device X"** - Two-step query using `get_objects` with filters
- **"Explain cable relationships for device Y"** - Multi-object retrieval with `get_object_by_id`
- **"Show all devices at site DM-Scranton"** - Filtered query with specific fields only

See [Demo Scenarios](demos.md) for detailed examples.

## Performance Highlights

### Token Efficiency
- Full device object: ~1500 tokens
- Filtered device (4 fields): ~150 tokens
- **90% token reduction** through field filtering

### Tool Simplicity
- 3 tool schemas: ~500-1,000 tokens
- vs 142+ specialized tools: ~8,000-10,000 tokens
- **90% schema overhead reduction**

## Documentation

- **[MCP Integration](mcp-integration.md)** - Detailed architecture and usage patterns
- **[Demo Scenarios](demos.md)** - Example queries and installation
- **[Performance Analysis](performance-analysis.md)** - Token optimization strategies

## Next Steps

Phase 2 provides the foundation for:

- **Phase 3** (Failed): OpenAI orchestration attempt (0% success rate)
- **Phase 4** (Complete): Agent framework comparison study validating context-dependent framework selection
- **Phase 5-7** (Future): Neo4j integration, RAG intelligence, analytics platform

All future phases build upon this shared MCP infrastructure layer.
